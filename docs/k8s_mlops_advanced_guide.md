# 📘 [Advanced K8s MLOps] AI 분산 학습 및 파이프라인 구축 매뉴얼

이 문서는 K8s 클러스터 환경에서 다중 GPU 자원을 활용하여 AI 모델의 분산 학습 및 MLOps 파이프라인을 구축하기 위한 심화 기술 매뉴얼입니다. 데이터 덮어쓰기 문제, 이기종 GPU 혼용 시의 병목 현상, 그리고 파이프라인 자동화에 대한 구체적인 해결책과 검증된 YAML 구조를 제공합니다.

---

## 1장. 파드(Pod)별 독립적 저장소 동적 할당 (`subPathExpr` 활용)

단일 공유 저장소(NFS/NAS)를 여러 워커 파드가 동시에 마운트할 때 발생하는 파일 충돌(Race Condition) 및 덮어쓰기를 방지하기 위해서는 쿠버네티스의 **Downward API**와 **동적 경로 매핑** 기술이 필요합니다. 이를 통해 애플리케이션(Python 코드)의 수정 없이 인프라 레벨에서 데이터를 격리할 수 있습니다.

### 1.1. 기술 원리
* **Downward API (`fieldRef`)**: 파드가 생성되는 시점에 쿠버네티스 API 서버가 파드의 메타데이터(이름, IP, 네임스페이스 등)를 컨테이너 내부의 환경 변수로 주입합니다.
* **`subPathExpr`**: 일반적인 `subPath`와 달리, 주입된 환경 변수를 해석하여 볼륨 마운트 경로를 동적으로 생성합니다.

### 1.2. 구현 구조 (YAML)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: train-worker-1
spec:
  containers:
  - name: ai-model-container
    image: <비공개_레지스트리_경로>/model-image:latest
    
    # 1. Downward API를 통한 환경 변수 주입
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name

    volumeMounts:
    - name: nfs-storage
      mountPath: /app/outputs
      # 2. NAS 루트 디렉토리 하위에 파드 이름(train-worker-1)으로 폴더를 자동 생성 및 마운트
      subPathExpr: $(POD_NAME) 

  volumes:
  - name: nfs-storage
    persistentVolumeClaim:
      claimName: shared-nas-pvc
```
이 설정을 적용하면 코드가 `/app/outputs/weights.pth`를 저장할 때, 실제 물리적 NAS에는 `/nas_root/train-worker-1/weights.pth`로 저장되어 워커 간 데이터 오염이 원천 차단됩니다.

---

## 2장. Kubeflow Training Operator를 활용한 분산 학습 오케스트레이션 (PyTorchJob)

K8s에서 PyTorch의 DDP(Distributed Data Parallel)를 실행하려면, 클러스터에 분산된 워커들이 **서로의 IP를 발견(Discovery)**하고, **자신의 고유 역할 번호(Rank)를 인식**해야 합니다. 과거에는 이를 위해 `Indexed Job`과 `Headless Service`를 직접 설정해야 했지만, 이제는 사실상의 업계 표준인 **Kubeflow Training Operator**의 `PyTorchJob` Custom Resource를 활용하여 훨씬 안정적이고 단순하게 처리할 수 있습니다.

### 2.1. Kubeflow Training Operator의 역할
* **자동 네트워크 프로비저닝**: 각 레플리카(Master, Worker 등) 간의 통신을 위해 개별 Headless Service를 자동으로 생성해 줍니다.
* **환경 변수 자동 주입**: 분산 학습에 필수적인 `MASTER_ADDR`, `MASTER_PORT`, `WORLD_SIZE`, `RANK` 등을 컨테이너 내부에 자동으로 바인딩해 줍니다.
* **안정적인 부트스트랩**: 워커 노드가 실행될 때 마스터 노드의 DNS가 분석될 때까지 대기해 주는 init container(`init-pytorch`)를 기본 제공하여, 파드 시작 순서 불일치로 인한 조기 크래시를 원천 방지합니다.

### 2.2. 분산 학습 구현 구조 (`PyTorchJob` YAML)
아래는 총 4개의 GPU 워커(Master 1, Worker 3)를 구성하여 PyTorch DDP 학습을 실행하기 위한 `PyTorchJob` 표준 YAML 구조입니다.

```yaml
apiVersion: "kubeflow.org/v1"
kind: "PyTorchJob"
metadata:
  name: "pytorch-ddp-dist"
  namespace: "kubeflow"
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        metadata:
          labels:
            job-name: pytorch-ddp-dist
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:latest
            # PyTorch DDP 환경 변수들이 Operator에 의해 자동 주입되므로 별도의 env 설정이 생략됩니다.
            command: ["python", "-m", "torch.distributed.run"]
            args:
            - "--nnodes=$(WORLD_SIZE)"
            - "--node_rank=$(RANK)"
            - "--master_addr=$(MASTER_ADDR)"
            - "--master_port=$(MASTER_PORT)"
            - "custom_runner.py"
            volumeMounts:
            - name: shared-storage
              mountPath: /data
          volumes:
          - name: shared-storage
            persistentVolumeClaim:
              claimName: shared-nas-pvc
    Worker:
      replicas: 3
      restartPolicy: OnFailure
      template:
        metadata:
          labels:
            job-name: pytorch-ddp-dist
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:latest
            command: ["python", "-m", "torch.distributed.run"]
            args:
            - "--nnodes=$(WORLD_SIZE)"
            - "--node_rank=$(RANK)"
            - "--master_addr=$(MASTER_ADDR)"
            - "--master_port=$(MASTER_PORT)"
            - "custom_runner.py"
            volumeMounts:
            - name: shared-storage
              mountPath: /data
          volumes:
          - name: shared-storage
            persistentVolumeClaim:
              claimName: shared-nas-pvc
```

### 2.3. 즉시 검증 가능한 실습용 매니페스트 (CPU/gloo 백엔드 활용)
별도의 데이터 볼륨(PVC)이나 외부 Python 스크립트 파일 없이, 공식 PyTorch 이미지와 인라인 코드를 사용하여 실제 PyTorch DDP 분산 네트워크 초기화가 원활히 작동하는지 즉시 테스트해 볼 수 있는 매니페스트입니다. CPU 기반의 `gloo` 통신 백엔드를 사용하여 GPU 장치가 없는 일반 노드에서도 작동합니다.

```yaml
apiVersion: "kubeflow.org/v1"
kind: "PyTorchJob"
metadata:
  name: "pytorch-ddp-test"
  namespace: "kubeflow"
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:2.1.2-cuda12.1-cudnn8-runtime
            command:
            - "python"
            - "-c"
            - |
              import os
              import torch.distributed as dist
              print("Master: PyTorch DDP 연결 초기화 시작...")
              dist.init_process_group(
                  backend="gloo",
                  init_method="env://",
                  world_size=int(os.environ["WORLD_SIZE"]),
                  rank=int(os.environ["RANK"])
              )
              print(f"Master: 성공적으로 Rank {os.environ['RANK']}로 연결되었습니다!")
              dist.destroy_process_group()
    Worker:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:2.1.2-cuda12.1-cudnn8-runtime
            command:
            - "python"
            - "-c"
            - |
              import os
              import torch.distributed as dist
              print("Worker: PyTorch DDP 연결 초기화 시작...")
              dist.init_process_group(
                  backend="gloo",
                  init_method="env://",
                  world_size=int(os.environ["WORLD_SIZE"]),
                  rank=int(os.environ["RANK"])
              )
              print(f"Worker: 성공적으로 Rank {os.environ['RANK']}로 연결되었습니다!")
              dist.destroy_process_group()
```


### 2.4. 실무 운영 환경 예시 (Detectron2 DDP)

실제 운영 환경에서 다중 GPU 자원을 활용해 Detectron2 분산 학습을 수행하며 겪었던 여러 실제 장애와 병목 문제를 해결한 최종 검증 매니페스트 및 관련 노하우 정리입니다. 전체 YAML 코드는 [5_detectron_ddp_job.yaml](file:///home/user/k8s_install_study/examples/5_detectron_ddp_job.yaml)에서 확인할 수 있습니다.

#### 2.4.1. 겪었던 문제와 최종 해결 방향 요약

> [!WARNING]
> **장애 1) 학습이 Validation 단계 즈음에서 멈추고, 30분 후 아래와 같은 에러로 죽는 현상**
> * **에러 메시지**: `Watchdog caught collective operation timeout... ran for 1800004 ms`
> * **원인**: Detectron2의 `PeriodicCheckpointer`와 `EvalHook`은 `CHECKPOINT_PERIOD`와 `EVAL_PERIOD`가 같은 값(예: 1000)으로 설정되어 있으면 같은 iteration에서 함께 트리거됩니다. 이때 체크포인트 저장(`torch.save`)은 오직 global rank 0 (Master)에서만 실행되는데, `torch.save`는 텐서/메타데이터를 아주 작은 단위로 수천 번 `write()`하는 pickle 직렬화 방식이라, 이 파일을 NAS(NFS) PVC에 직접 저장하면 매 write마다 네트워크 왕복 지연(RTT)이 누적되어 초당 수십 KB 수준으로 극단적으로 느려집니다. (반면 initContainer의 rsync성 대량 파일 복사는 큰 청크 단위 순차 read/write라 이 문제에 노출되지 않고 정상 속도가 나옴). Master가 체크포인트 저장에 발이 묶여 있는 동안, Worker는 이미 evaluation을 마치고 allreduce 동기화 지점에서 Master를 기다리게 되고, NCCL watchdog의 기본 타임아웃(1,800,000ms = 30분)이 지나면 전체 프로세스 그룹이 강제로 죽습니다.
> * **해결책**: 학습 중 저장 경로(save_path / OUTPUT_DIR)를 NAS가 아닌 로컬(emptyDir) SSD로 변경하여, NFS write 자체를 학습 루프에서 완전히 제거했습니다.

> [!TIP]
> **장애 2) 로컬에만 저장할 경우, 학습 도중 중간 가중치를 확인할 방법이 없는 문제**
> * **원인**: 학습 종료 후 한 번에 NAS로 복사하는 방식이면 학습 도중 진행상황 검증 및 모니터링이 불가능합니다.
> * **해결책**: 같은 Pod(Master) 안에 `sync-sidecar` 컨테이너를 추가하여, 로컬 emptyDir(`local-results`)을 학습 컨테이너와 공유시키고 주기적으로 NAS(`results-storage`)에 동기화합니다. 학습 컨테이너가 종료될 때 `TRAINING_DONE` 마커 파일을 남기면, sidecar는 마지막으로 한 번 더 동기화한 뒤 스스로 종료합니다.
>   - *주의*: 사용 중인 컨테이너 이미지에 rsync 바이너리가 없어서, 순수 파이썬으로 "파일 크기+수정시각이 같으면 skip" 방식의 얕은 동기화 로직을 직접 구현하여 연동했습니다.
>   - *주의*: 학습 컨테이너의 command를 bash 스크립트로 감쌀 때 `set -e`를 쓰면, 학습이 비정상 종료(0이 아닌 exit code)되는 순간 스크립트 전체가 그대로 죽어버려 마커 파일을 못 남길 수 있습니다. 반드시 `... || TRAIN_EXIT=$?` 형태로 실패를 붙잡아 다음 줄(마커 생성)까지 반드시 도달하도록 만들어야 sidecar가 무한 대기에 빠지지 않습니다.

> [!CAUTION]
> **장애 3) Pod를 강제 종료(kill)하려 하면 10분 가까이 안 죽는 현상**
> * **에러 메시지**: `failed to KillContainer ... rpc error: code = DeadlineExceeded`
> * **원인**: 컨테이너 내부 프로세스가 NFS(NAS) I/O 대기 중 커널 레벨의 Uninterruptible Sleep (D state)에 빠지면, SIGTERM은 물론 SIGKILL조차 커널이 해당 syscall을 리턴하기 전까지 전달되지 않습니다. NFS 서버 응답 지연이나 네트워크 문제가 있으면 이 상태가 수 분~수십 분 지속될 수 있고, kubelet-containerd 간 CRI 호출도 자체 타임아웃에 걸려 위 에러 메시지가 반복 출력됩니다.
> * **해결책**: 근본적으로는 장애 1과 동일하게 "학습 중 NFS I/O 자체를 없애는 것"이 해결책입니다. 로컬 저장 방식으로 전환한 이후로는 이 현상도 함께 사라졌습니다. (클러스터 관리자 측에서 해당 PVC의 NFS 마운트 옵션을 hard -> soft + timeo/retrans 설정으로 바꾸면 이런 hang 자체를 줄일 수 있으나, 이는 애플리케이션 코드에서 I/O 에러 처리/재시도 로직을 추가로 요구하므로 트레이드오프가 있습니다.)

#### 2.4.2. 새 학습 Job을 만들 때 반드시 지킬 것 (체크리스트)
1. **데이터셋 로컬 복사**: 데이터셋은 `initContainer`로 NAS -> 로컬(emptyDir)에 미리 복사해서 사용할 것. 학습 중 NAS를 직접 읽게 하지 마십시오 (특히 DataLoader의 랜덤 접근 패턴은 NFS에 매우 불리함).
2. **출력 경로 로컬 지정**: `cfg.OUTPUT_DIR` (또는 `--save_path`)은 반드시 로컬(emptyDir) 경로로 지정할 것. NAS 경로를 직접 지정하면 장애 1이 재발합니다.
3. **결과 백업 연동**: 학습 도중 결과를 NAS에서 확인하고 싶다면 `sync-sidecar` 패턴을 사용하십시오.
4. **NCCL 환경변수 통일**: Master/Worker의 NCCL 관련 환경변수는 반드시 동일하게 통일하십시오. 값이 서로 다르면 우연히 동작하더라도 나중에 원인 추적이 매우 어려워집니다.
5. **정리 태스크 보장**: bash로 command를 감쌀 경우 `set -e`와 실패 후처리(`|| VAR=$?`)를 함께 고려해서, 실패 상황에서도 후속 정리(sidecar 신호 등)가 반드시 실행되게 만드십시오.
6. **주기 주의**: `CHECKPOINT_PERIOD`와 `EVAL_PERIOD`를 동일하게 맞출 경우, 장애 1이 재현될 수 있다는 점을 인지하십시오.

#### 2.4.3. 최종 검증된 PyTorchJob 설정 (YAML)
```yaml
# =============================================================================
# Detectron2 Multi-Node DDP 분산학습 - Kubeflow PyTorchJob 참조 구성
# =============================================================================
#
# 이 문서는 단순 설정 예시가 아니라, 실제 운영 중 발생했던 장애들을 해결하며
# 최종적으로 검증 완료된 구성입니다. 각 섹션의 주석은 "왜 이렇게 구성했는지"를
# 반드시 함께 읽고, 새로운 학습 Job을 만들 때 동일한 실수를 반복하지 않도록
# 참고하시기 바랍니다.
#
# =============================================================================

apiVersion: "kubeflow.org/v1"   # Kubeflow Training Operator(KTO) v1 규칙을 사용함
kind: "PyTorchJob"              # 일반 Pod가 아닌, 분산 학습을 전담하는 커스텀 리소스 선언
metadata:
  name: "detectron-ddp-job"     # 리소스 고유 이름 (쿠버네티스 표준에 따라 언더바(_) 금지, 대시(-) 사용)
  namespace: "default"          # 분산 학습 작업이 실행될 가상 네임스페이스 그룹
spec:
  pytorchReplicaSpecs:

    # ==================== [ 마스터 노드 정의 ] ====================
    Master:
      replicas: 1               # 분산 학습의 메인 컨트롤러 역할을 할 랭크 0(Rank 0) Pod 개수 (보통 1개)
      restartPolicy: Never      # 학습 에러 발생 시 자동 재기동하지 않고 즉시 작업을 실패(Failed) 처리하여 상태 보존
      template:
        metadata:
          labels:
            job-name: pytorch-ddp-dist # 분산 학습 파드들을 묶어주는 논리적 식별 라벨
        spec:
          # ------------------------------------------------------------------
          # 3. 초기화 컨테이너(initContainer): 메인 학습 컨테이너가 뜨기 전에
          #    NAS(NFS)에 있는 원본 데이터셋을 로컬 SSD(emptyDir)로 미리 복사해둠.
          #    -> 학습 도중 반복적으로 발생하는 NAS I/O(특히 DataLoader의 랜덤 접근)를
          #       제거해서 NCCL collective timeout 등 통신 hang 문제를 회피하기 위함.
          #    -> 'ori_json' 폴더는 학습에 불필요한 원본 라벨 백업이므로 복사 대상에서 제외.
          # ------------------------------------------------------------------
          initContainers:
          - name: copy-dataset
            image: private-registry.domain.com/ai-team/detectron2:latest
            command:
            - "python3"
            - "-c"
            - |
              import os, shutil
              src = "/nas/project/dataset"
              dst = "/local/project/dataset"

              print("[1/2] Scanning NAS file list (excluding 'ori_json')...", flush=True)
              files_to_copy = []
              for root, dirs, files in os.walk(src):
                  if 'ori_json' in root.split(os.sep):
                      continue
                  for f in files:
                      files_to_copy.append(os.path.join(root, f))

              total = len(files_to_copy)
              print(f"[2/2] Starting file copy to Local SSD (Total: {total} files)...", flush=True)

              for i, f in enumerate(files_to_copy, 1):
                  rel_path = os.path.relpath(f, src)
                  target_path = os.path.join(dst, rel_path)
                  os.makedirs(os.path.dirname(target_path), exist_ok=True)
                  shutil.copy2(f, target_path)
                  if i % 200 == 0 or i == total:
                      print(f" -> Progress: {i}/{total} files copied ({i/total*100:.1f}%)", flush=True)
              print("Dataset local copy completed successfully!", flush=True)
            volumeMounts:
            - name: datasets-storage  # 원본 NAS 마운트 (읽기 전용 소스)
              mountPath: /nas
            - name: local-storage     # 빈 로컬 마운트 (복사 대상)
              mountPath: /local

          containers:
          # 🚨 중요: KTO 웹훅 규칙에 따라 메인 컨테이너 이름은 반드시 'pytorch'여야 함
          - name: pytorch
            image: private-registry.domain.com/ai-team/detectron2:latest

            # ------------------------------------------------------------------
            # 4. [핵심 수정] 컨테이너 구동 방식을 bash 스크립트로 감싸서 사용.
            #    학습이 끝난 뒤 sync-sidecar 컨테이너에게 "학습 종료" 신호(마커 파일)를
            #    남겨주기 위함이며, 학습이 실패하더라도(0이 아닌 exit code) sidecar가
            #    무한정 대기하지 않도록 반드시 마커 파일을 남기고 종료함.
            # ------------------------------------------------------------------
            command: ["/bin/bash", "-c"]
            args:
            - |
              set -e
              /root/miniconda3/envs/detectron2/bin/python -u /app/detectron_train_multi_node.py \
                --input=/data/datasets/project/dataset \
                --save_path=/data/local_results/detectron2_results \
                --learning_rate=0.00025 \
                --num_classes=6 \
                --max_iter=40000 \
                --steps=25000,35000 \
                --batch_size=4 || TRAIN_EXIT=$?
              # 학습이 정상 종료됐으면 TRAIN_EXIT는 비어있으므로 0으로 기본값 처리
              TRAIN_EXIT=${TRAIN_EXIT:-0}
              # 학습 성공/실패 여부와 무관하게 sync-sidecar가 루프를 빠져나올 수 있도록
              # 반드시 마커 파일을 생성 (set -e에 의해 스크립트가 중간에 죽더라도
              # 위 || 처리 덕분에 이 지점까지는 항상 도달함)
              touch /data/local_results/TRAINING_DONE
              exit ${TRAIN_EXIT}

            # ------------------------------------------------------------------
            # 5. NCCL 통신 관련 환경변수
            #    Master/Worker 양쪽 모두 동일하게 통일하여 설정 (예전에 값이 서로
            #    달라 발생했던 원인불명 통신 이슈를 예방)
            # ------------------------------------------------------------------
            env:
            - name: NCCL_IB_DISABLE
              value: "1"                         # 인피니밴드가 없다면 강제로 끄고 일반 TCP 소켓 통신 사용
            - name: NCCL_P2P_DISABLE
              value: "1"                         # [필수] 동일 노드 내 GPU 간 P2P 무한 대기 버그 차단
            - name: NCCL_SOCKET_IFNAME
              value: "^lo,docker0"               # [필수] 경량 테스트에서 검증 완료된 안전한 네트워크 타겟팅
            - name: NCCL_DEBUG
              value: "INFO"                      # 통신 문제 재발 시 상세 로그를 남기도록 설정

            resources:
              limits:
                nvidia.com/gpu: "2"               # 노드당 2 GPU 설정 유지
              requests:
                nvidia.com/gpu: "2"

            volumeMounts:
            - name: local-storage
              mountPath: /data/datasets    # initContainer가 NAS에서 미리 복사해 둔 로컬 데이터셋 마운트
            - name: local-results          # [신규] 체크포인트/평가 결과를 저장할 로컬(emptyDir) 볼륨
              mountPath: /data/local_results
              # -> save_path를 NAS가 아닌 이 로컬 경로로 지정한 이유:
              #    torch.save()의 pickle 저장 방식은 작은 단위의 write가 매우 많이
              #    발생하는데, 이를 NFS(NAS)에 직접 쓰면 latency 누적으로 인해
              #    체크포인트 저장이 극도로 느려지고(심할 경우 초당 수십 KB),
              #    그 동안 학습 프로세스(rank 0)가 블로킹되어 NCCL allreduce
              #    타임아웃(30분) 및 전체 Job hang으로 이어짐. 로컬 SSD에 저장하면
              #    이 문제가 원천 차단됨.
            - name: dshm
              mountPath: /dev/shm         # PyTorch DataLoader 멀티프로세싱용 공유메모리 (부족 시 DDP 크래시 방지)

          # ------------------------------------------------------------------
          # 6. [신규 추가] sync-sidecar 컨테이너
          #    학습 컨테이너(pytorch)와 같은 Pod 안에서 나란히 실행되며,
          #    로컬(emptyDir)에 저장된 체크포인트/결과물을 주기적으로 NAS(NFS)에
          #    동기화함. rsync 바이너리가 이미지에 없으므로 순수 파이썬으로
          #    "변경된 파일만 재복사"하는 얕은 동기화 로직을 직접 구현.
          #    -> 학습 도중에도 NAS에서 중간 가중치를 확인할 수 있게 해주며,
          #       동기화 작업이 학습 루프(NCCL collective)와 완전히 분리되어 있어
          #       NAS I/O가 느려지더라도 학습 프로세스를 블로킹하지 않음.
          # ------------------------------------------------------------------
          - name: sync-sidecar
            image: private-registry.domain.com/ai-team/detectron2:latest
            command: ["/bin/bash", "-c"]
            args:
            - |
              python3 -u -c "
              import os, shutil, time, sys

              src = '/data/local_results/detectron2_results'   # 학습 컨테이너가 쓰는 로컬 결과 경로
              dst = '/data/results/detectron2_results'         # NAS 상의 최종 보관 경로
              marker = '/data/local_results/TRAINING_DONE'    # 학습 컨테이너가 남기는 종료 신호 파일

              def sync_once():
                  # src -> dst로 파일을 재귀적으로 복사하되,
                  # 크기+수정시각이 동일하면 이미 복사된 것으로 간주하고 건너뜀
                  # (rsync -a의 '변경분만 전송' 동작을 얕게 흉내낸 것)
                  copied, skipped, failed = 0, 0, 0
                  os.makedirs(dst, exist_ok=True)
                  for root, dirs, files in os.walk(src):
                      rel_dir = os.path.relpath(root, src)
                      target_dir = os.path.join(dst, rel_dir) if rel_dir != '.' else dst
                      os.makedirs(target_dir, exist_ok=True)
                      for f in files:
                          s = os.path.join(root, f)
                          d = os.path.join(target_dir, f)
                          try:
                              if os.path.exists(d):
                                  s_stat, d_stat = os.stat(s), os.stat(d)
                                  if s_stat.st_size == d_stat.st_size and int(s_stat.st_mtime) <= int(d_stat.st_mtime):
                                      skipped += 1
                                      continue
                              shutil.copy2(s, d)
                              copied += 1
                          except Exception as e:
                              print(f'[Sidecar] Failed to copy {s}: {e}', flush=True)
                              failed += 1
                  print(f'[Sidecar] Sync done. copied={copied}, skipped={skipped}, failed={failed}', flush=True)

              print('[Sidecar] Periodic sync started.', flush=True)
              # 학습 컨테이너가 TRAINING_DONE 마커를 남기기 전까지 2분 주기로 계속 동기화
              while not os.path.exists(marker):
                  try:
                      sync_once()
                  except Exception as e:
                      print(f'[Sidecar] sync_once failed: {e}', flush=True)
                  time.sleep(120)

              # 학습 종료 감지 -> 마지막 완전한 상태를 한 번 더 동기화 후 스스로 종료
              # (Pod가 Succeeded로 판정되려면 이 컨테이너도 반드시 종료되어야 함)
              print('[Sidecar] Training done detected. Performing final sync.', flush=True)
              sync_once()
              print('[Sidecar] Final sync completed. Exiting.', flush=True)
              "
            volumeMounts:
            - name: local-results
              mountPath: /data/local_results  # 학습 컨테이너와 동일한 emptyDir을 공유 (같은 Pod라서 가능)
            - name: results-storage
              mountPath: /data/results        # 최종적으로 결과물이 보관될 NAS(NFS) 마운트

          # ------------------------------------------------------------------
          # 7. Pod 레벨 스토리지 자원 정의
          # ------------------------------------------------------------------
          volumes:
          - name: datasets-storage
            persistentVolumeClaim:
              claimName: nas-models-pvc  # 원본 데이터셋용 NAS PVC (initContainer가 읽어감)
          - name: results-storage
            persistentVolumeClaim:
              claimName: nas-results-pvc     # 최종 결과 보관용 NAS PVC (sync-sidecar가 씀)
          - name: local-storage
            emptyDir: {}                     # 데이터셋 로컬 복사본 (Pod 삭제 시 함께 소멸)
          - name: local-results
            emptyDir: {}                     # 체크포인트/평가결과 로컬 임시 저장소 (pytorch, sync-sidecar가 공유)
          - name: dshm
            emptyDir:
              medium: Memory                 # /dev/shm을 메모리 기반으로 별도 확보 (기본 64MB 한도 회피)

    # ==================== [ 워커 노드 정의 ] ====================
    Worker:
      replicas: 1               # 분산 학습에 참여할 일꾼(Worker) Pod 개수
      restartPolicy: Never      # 워커 노드가 튕기면 분산 학습 정합성을 위해 전체 작업을 즉시 중단
      template:
        metadata:
          labels:
            job-name: pytorch-ddp-dist
        spec:
          # Worker도 Master와 동일하게 데이터셋을 NAS에서 로컬로 미리 복사
          # (같은 학습 데이터를 각 노드가 각자 로컬로 들고 있어야 하므로 동일 로직 필요)
          initContainers:
          - name: copy-dataset
            image: private-registry.domain.com/ai-team/detectron2:latest
            command:
            - "python3"
            - "-c"
            - |
              import os, shutil
              src = "/nas/project/dataset"
              dst = "/local/project/dataset"

              print("[1/2] Scanning NAS file list (excluding 'ori_json')...", flush=True)
              files_to_copy = []
              for root, dirs, files in os.walk(src):
                  if 'ori_json' in root.split(os.sep):
                      continue
                  for f in files:
                      files_to_copy.append(os.path.join(root, f))

              total = len(files_to_copy)
              print(f"[2/2] Starting file copy to Local SSD (Total: {total} files)...", flush=True)

              for i, f in enumerate(files_to_copy, 1):
                  rel_path = os.path.relpath(f, src)
                  target_path = os.path.join(dst, rel_path)
                  os.makedirs(os.path.dirname(target_path), exist_ok=True)
                  shutil.copy2(f, target_path)
                  if i % 200 == 0 or i == total:
                      print(f" -> Progress: {i}/{total} files copied ({i/total*100:.1f}%)", flush=True)
              print("Dataset local copy completed successfully!", flush=True)
            volumeMounts:
            - name: datasets-storage
              mountPath: /nas
            - name: local-storage
              mountPath: /local

          containers:
          # 🚨 중요: 워커 영역 역시 메인 컨테이너 이름은 무조건 'pytorch'로 일치시켜야 함
          - name: pytorch
            image: private-registry.domain.com/ai-team/detectron2:latest
            # Worker는 rank 0(main process)이 아니므로 체크포인트/eval 결과를
            # 직접 저장하지 않음. 따라서 Master처럼 bash 래핑이나 sidecar 없이
            # 학습 스크립트를 곧바로 실행해도 무방함.
            command: ["/root/miniconda3/envs/detectron2/bin/python", "-u", "/app/detectron_train_multi_node.py"]
            env:
            - name: NCCL_IB_DISABLE
              value: "1"
            - name: NCCL_P2P_DISABLE
              value: "1"                         # [필수] 동일 노드 내 GPU 간 P2P 무한 대기 버그 차단 (Master와 동일하게 통일)
            - name: NCCL_SOCKET_IFNAME
              value: "^lo,docker0"               # [필수] 검증 완료된 소켓 필터링 (Master와 동일하게 통일)
            - name: NCCL_DEBUG
              value: "INFO"
            args:
            - "--input=/data/datasets/project/dataset"
            - "--save_path=/data/local_results/detectron2_results"  # Worker도 로컬 경로 사용 (NAS 불필요한 쓰기 방지)
            - "--learning_rate=0.00025"       # 기본 학습률 제어 (world_size에 따라 내부적으로 자동 스케일링됨)
            - "--num_classes=6"              # 클래스 개수 지정
            - "--max_iter=40000"             # 최대 반복 횟수 제어
            - "--steps=25000,35000"          # 쉼표(,)를 통해 복수의 스텝 인자값을 깔끔하게 전달 가능
            - "--batch_size=4"               # 한 개 GPU당 가져갈 이미지 개수
            resources:
              limits:
                nvidia.com/gpu: "2"
              requests:
                nvidia.com/gpu: "2"
            volumeMounts:
            - name: local-storage
              mountPath: /data/datasets
            - name: results-storage
              mountPath: /data/results        # Worker는 실질적으로 여기에 쓰지 않지만 인자 호환을 위해 마운트 유지
            - name: local-results
              mountPath: /data/local_results
            - name: dshm
              mountPath: /dev/shm
          volumes:
          - name: datasets-storage
            persistentVolumeClaim:
              claimName: nas-models-pvc
          - name: results-storage
            persistentVolumeClaim:
              claimName: nas-results-pvc
          - name: local-storage
            emptyDir: {}
          - name: local-results
            emptyDir: {}
          - name: dshm
            emptyDir:
              medium: Memory
```

---


## 3장. 이기종 GPU 하드웨어 분류 및 스케줄링 전략

RTX 3090, RTX 4060, A5000 등 다양한 스펙의 GPU가 섞인 인프라에서 분산 학습을 무작위로 배포하면, 가장 연산 속도가 느린 GPU(예: 4060)가 다음 스텝으로 넘어갈 때까지 고성능 GPU(예: 3090)가 대기해야 하는 **Straggler(지연자) 병목 현상**이 발생합니다. K8s 스케줄러를 제어하여 이기종 GPU를 격리해야 합니다.

### 3.1. 자동화된 라벨링 (NVIDIA GPU Feature Discovery)
수동으로 라벨을 붙이는 대신, 노드에 설치된 NVIDIA 데몬셋이 하드웨어 스펙을 K8s API에 실시간으로 보고하게 만듭니다.
```bash
# GFD(GPU Feature Discovery) Helm 차트 배포
helm upgrade -i nvdp nvdp/nvidia-device-plugin \
  --namespace kube-system \
  --set gfd.enabled=true
```
배포 후 노드의 상세 정보를 조회하면 `nvidia.com/gpu.product=NVIDIA-GeForce-RTX-3090`과 같은 구체적인 하드웨어 라벨이 자동으로 부착된 것을 확인할 수 있습니다.

### 3.2. 스케줄링 제어 (`nodeSelector` 및 `Affinity`)
동일한 연산 능력을 가진 GPU들끼리만 묶어서 PyTorchJob을 실행하도록 YAML을 구성합니다. `PyTorchJob` 스펙 하위의 `Master` 및 `Worker` 템플릿(Pod Template Spec)의 `nodeSelector`와 `affinity` 필드에 이를 적용합니다.

나아가 `podAffinity`를 설정하여 파드들이 최대한 동일한 물리적 서버(스위치를 거치지 않는 로컬 버스)에 배치되도록 강제하여 통신 지연을 최소화합니다.

```yaml
# PyTorchJob spec 하위 replicaSpecs 예시
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template:
        spec:
          # 1. 마스터 파드를 특정 GPU 모델(RTX 3090)이 장착된 노드에 할당
          nodeSelector:
            nvidia.com/gpu.product: "NVIDIA-GeForce-RTX-3090"
          containers:
          - name: pytorch
            image: pytorch/pytorch:latest
            # 생략...
    Worker:
      replicas: 3
      template:
        spec:
          # 2. 워커 파드도 마스터와 동일한 GPU 모델(RTX 3090)이 장착된 노드에 할당 (병목 방지)
          nodeSelector:
            nvidia.com/gpu.product: "NVIDIA-GeForce-RTX-3090"
            
          # 3. 파드 몰아주기 (네트워크 통신 최적화)
          # Master 파드와 동일한 호스트명(서버)에 우선적으로 배치하도록 스케줄러 유도
          affinity:
            podAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchExpressions:
                    - key: training.kubeflow.org/job-name
                      operator: In
                      values:
                      - pytorch-ddp-dist
                  topologyKey: "kubernetes.io/hostname" # 동일 호스트에 우선 순위 부여
          containers:
          - name: pytorch
            image: pytorch/pytorch:latest
            # 생략...
```

### 3.3. 다중 GPU 모델 선택 및 특정 VRAM 크기 기준 스케줄링 (`nodeAffinity`)

`nodeSelector`는 1대1 매핑(예: GPU 제품명은 반드시 RTX 3090이어야 함)만 가능합니다. 하지만 **Node Affinity (`nodeAffinity`)**를 활용하면 복수의 GPU 모델을 선택하거나, 특정 VRAM 크기(GB) 이상을 가진 GPU만 필터링하여 스케줄링할 수 있습니다.

#### 1) 여러 특정 GPU 모델 중 하나를 임의로 선택할 경우 (OR 연산)
`RTX 3090` 또는 `RTX A5000` 등 연산 성능과 메모리 크기가 유사한 복수의 이기종 GPU를 함께 사용하려 할 때는 `In` 연산자를 사용합니다.
```yaml
spec:
  pytorchReplicaSpecs:
    Worker:
      template:
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: nvidia.com/gpu.product
                    operator: In
                    values:
                    - "NVIDIA-GeForce-RTX-3090"
                    - "NVIDIA-RTX-A5000"
```

#### 2) 특정 VRAM 크기(예: 24GB) 이상을 가진 GPU 노드만 선택할 경우
NVIDIA GPU Feature Discovery(GFD)는 GPU의 VRAM 메모리 용량 정보를 MiB 단위의 라벨(`nvidia.com/gpu.memory`)로 제공합니다. (예: 24GB VRAM = 24576MiB)

* **방법 A: `Gt`(Greater Than) 연산자 활용**
  쿠버네티스의 노드 어피니티는 숫자형 비교 연산자인 `Gt`를 제공하므로, 특정 메모리 용량 초과인 노드만 필터링할 수 있습니다.
  ```yaml
  spec:
    pytorchReplicaSpecs:
      Worker:
        template:
          spec:
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                  - matchExpressions:
                    - key: nvidia.com/gpu.memory
                      operator: Gt
                      values:
                      - "20000" # 20000MiB(약 20GB) 초과의 VRAM 노드만 매핑
  ```

* **방법 B: 특정 대용량 VRAM 규격 명시 (`In` 연산자)**
  정확한 용량 매칭을 원할 때는 24GB 이상의 주요 VRAM 규격(24GB, 40GB, 48GB, 80GB 등)을 명시적으로 나열하여 매칭합니다.
  ```yaml
  spec:
    pytorchReplicaSpecs:
      Worker:
        template:
          spec:
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                  - matchExpressions:
                    - key: nvidia.com/gpu.memory
                      operator: In
                      values:
                      - "24576" # 24GB
                      - "40960" # 40GB
                      - "49152" # 48GB
                      - "81920" # 80GB
  ```

---

## 4장. MLOps 파이프라인 (DAG) 구성 및 오케스트레이션

개별적인 K8s 리소스 구성을 마쳤다면, 이를 '전처리 -> 모델 학습 -> 가중치 평가'로 이어지는 자동화된 비순환 방향 그래프(DAG, Directed Acyclic Graph)로 엮어야 합니다. 쿠버네티스 네이티브 환경에서는 **Argo Workflows**가 산업 표준으로 사용됩니다.

파이프라인의 핵심은 **"작업은 컨테이너가 수행하고, 데이터의 상태와 전환은 공유 저장소(PVC)가 담당한다"**는 철학입니다.

### 4.1. 파이프라인 아키텍처 설계
당신이 구축한 커스텀 Python 라이브러리(`AI_utility` 등)를 각 컨테이너의 진입점(Entrypoint)으로 사용하여 입출력을 표준화합니다.

1. **데이터 전처리 단계 (Data Preparation)**
   * **사용 자원**: CPU, 대용량 Memory
   * **행동**: 원본 데이터를 정제하여 `.npy` 또는 `.tfrecord` 형태로 가공한 뒤 `shared-nas-pvc`의 `/processed_data` 경로에 저장합니다.
2. **분산 학습 단계 (Distributed Training)**
   * **사용 자원**: 동일 스펙의 GPU 다수 (예: RTX 3090 4장)
   * **트리거 조건**: 전처리 단계의 파드가 `Succeeded` 상태로 완전 종료될 것.
   * **행동**: `/processed_data`에서 데이터를 읽어와 학습을 진행하고, 결과 가중치를 `/models/version_x/` 경로에 저장합니다.
3. **결과 평가 단계 (Evaluation & Export)**
   * **사용 자원**: GPU 1장 또는 CPU
   * **트리거 조건**: 분산 학습 Job이 완료될 것.
   * **행동**: 생성된 모델 가중치를 로드하여 테스트 데이터셋으로 검증하고, 검증 스코어를 DB나 로그로 전송합니다.

### 4.2. Argo Workflows를 이용한 파이프라인 YAML 예시
아래는 세 가지 단계가 의존성(`dependencies`)을 가지고 순차적으로 실행되도록 정의한 파이프라인 오케스트레이션의 뼈대입니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: mlops-pipeline-
spec:
  entrypoint: main-dag
  
  # 전체 워크플로에서 공통으로 사용할 볼륨 정의
  volumes:
  - name: shared-storage
    persistentVolumeClaim:
      claimName: shared-nas-pvc

  templates:
  # 파이프라인의 구조(DAG) 정의
  - name: main-dag
    dag:
      tasks:
      - name: step-1-preprocess
        template: preprocessing-job
        
      - name: step-2-train
        dependencies: [step-1-preprocess] # 전처리가 성공해야만 실행됨 ⭐
        template: distributed-training-job
        
      - name: step-3-evaluate
        dependencies: [step-2-train] # 학습이 완료되어야만 실행됨 ⭐
        template: evaluation-job

  # 각 단계별 실제 실행 컨테이너 정의 (간략화)
  - name: preprocessing-job
    container:
      image: <레지스트리>/data-prep:v1
      command: ["python", "preprocess.py"]
      volumeMounts:
      - name: shared-storage
        mountPath: /data

  - name: distributed-training-job
    # 2장에서 설명한 Kubeflow Training Operator의 PyTorchJob을 생성하는 리소스 템플릿
    resource:
      action: create
      successCondition: status.conditions[*].status == True && status.conditions[*].type == Succeeded
      failureCondition: status.conditions[*].status == True && status.conditions[*].type == Failed
      manifest: |
        apiVersion: kubeflow.org/v1
        kind: PyTorchJob
        metadata:
          generateName: pytorch-ddp-dist-
          namespace: kubeflow
        spec:
          pytorchReplicaSpecs:
            Master:
              replicas: 1
              restartPolicy: OnFailure
              template:
                spec:
                  containers:
                  - name: pytorch
                    image: <레지스트리>/train:v1
                    command: ["python", "-m", "torch.distributed.run", "train.py"]
                    volumeMounts:
                    - name: shared-storage
                      mountPath: /data
                  volumes:
                  - name: shared-storage
                    persistentVolumeClaim:
                      claimName: shared-nas-pvc
            Worker:
              replicas: 3
              restartPolicy: OnFailure
              template:
                spec:
                  containers:
                  - name: pytorch
                    image: <레지스트리>/train:v1
                    command: ["python", "-m", "torch.distributed.run", "train.py"]
                    volumeMounts:
                    - name: shared-storage
                      mountPath: /data
                  volumes:
                  - name: shared-storage
                    persistentVolumeClaim:
                      claimName: shared-nas-pvc

  - name: evaluation-job
    container:
      image: <레지스트리>/eval:v1
      command: ["python", "eval.py"]
      volumeMounts:
      - name: shared-storage
        mountPath: /data
```

### 💡 매뉴얼 요약 및 엔지니어링 체크포인트
1. **격리성 확보**: 파드별 동적 볼륨 마운트(`subPathExpr`)를 통해 데이터 덮어쓰기 로직을 인프라 단에서 완전히 분리하십시오.
2. **분산 오케스트레이션 자동화**: Kubeflow Training Operator(PyTorchJob)를 활용하면 통신 인프라 구축, 네트워킹 설정 및 파드 인덱스 매핑이 자동화되어 오류를 대폭 경감합니다.
3. **자원 최적화**: 3090, 4060 등 서버별 하드웨어 스펙을 K8s가 명확히 인지하도록 `nodeSelector`를 구성하여 GPU 낭비를 차단하십시오.
4. **표준화**: 단일 스크립트를 수동으로 실행하는 단계를 벗어나, 모든 프로세스를 컨테이너 단위로 분할하고 Argo Workflows와 같은 도구를 통해 DAG로 연결하면 완성도 높은 MLOps 시스템이 구축됩니다. 이 과정에서 직접 개발한 런너 모듈을 활용하면 유지보수성이 극대화될 것입니다.
