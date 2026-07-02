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
