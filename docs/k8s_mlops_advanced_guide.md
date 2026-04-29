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

## 2장. 다중 서버 분산 학습 네트워크 및 인덱스 구성 (PyTorch DDP)

K8s에서 PyTorch의 DDP(Distributed Data Parallel)를 실행하려면, 클러스터에 흩어진 컨테이너들이 **서로의 IP를 찾고(Discovery)**, **자신의 역할 번호(Rank)를 인식**해야 합니다. K8s 1.22+부터 정식 지원되는 **Indexed Job**과 **Headless Service**가 이 역할을 수행합니다.

### 2.1. 통신 계층 구축 (Headless Service)
마스터 파드의 유동적인 IP를 고정된 도메인 네임으로 덮어씌워 워커들이 항상 마스터를 찾을 수 있게 합니다.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: distributed-training-svc
spec:
  clusterIP: None # K8s 내부 로드밸런싱을 비활성화하고 DNS 레코드만 생성
  selector:
    job-name: ddp-job
```

### 2.2. 분산 학습 오케스트레이션 (Indexed Job)
`completionMode: Indexed`를 사용하여 생성되는 파드에 0번부터 순차적인 고유 인덱스를 부여합니다.
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ddp-job
spec:
  completions: 4 # 총 4개의 GPU 워커 파드 실행
  parallelism: 4 # 4개를 동시에 실행
  completionMode: Indexed 
  backoffLimit: 0 # 하나라도 실패하면 학습 전체가 오염되므로 재시도 방지
  template:
    metadata:
      labels:
        job-name: ddp-job
    spec:
      subdomain: distributed-training-svc
      restartPolicy: Never
      containers:
      - name: pytorch-worker
        image: pytorch/pytorch:latest
        
        env:
        # 분산 학습 필수 환경 변수
        - name: MASTER_ADDR
          value: "ddp-job-0.distributed-training-svc" # 0번 인덱스 파드를 마스터로 지정
        - name: MASTER_PORT
          value: "29500"
        - name: WORLD_SIZE
          value: "4"
        # 파드의 인덱스 번호를 컨테이너 내부의 RANK로 매핑
        - name: RANK
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']

        # 표준화된 MLOps 런너(Runner) 스크립트 실행
        command: ["python", "-m", "torch.distributed.run"]
        args:
        - "--nnodes=$(WORLD_SIZE)"
        - "--node_rank=$(RANK)"
        - "--master_addr=$(MASTER_ADDR)"
        - "--master_port=$(MASTER_PORT)"
        - "custom_runner.py"
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
동일한 연산 능력을 가진 GPU들끼리만 묶어서 DDP Job을 할당하도록 YAML을 구성합니다. 나아가 `podAffinity`를 설정하여 파드들이 최대한 동일한 물리적 서버(스위치를 거치지 않는 로컬 버스)에 배치되도록 강제하여 통신 지연을 최소화합니다.

```yaml
# spec 하위 설정 예시
    spec:
      # 1. 특정 GPU 모델(RTX 3090)이 장착된 노드만 선택
      nodeSelector:
        nvidia.com/gpu.product: "NVIDIA-GeForce-RTX-3090"
        
      # 2. 파드 몰아주기 (네트워크 통신 최적화)
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: job-name
                  operator: In
                  values:
                  - ddp-job
              topologyKey: "kubernetes.io/hostname" # 동일한 호스트명(서버)에 우선 배치
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
    # (2장에서 설명한 Indexed Job 로직이 Argo 형태로 통합됨)
    container:
      image: <레지스트리>/train:v1
      command: ["python", "-m", "torch.distributed.run", "train.py"]
      volumeMounts:
      - name: shared-storage
        mountPath: /data

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
2. **네트워크 안정성**: Headless Service와 Indexed Job의 조합은 PyTorch DDP를 위한 가장 완벽한 K8s 네이티브 솔루션입니다.
3. **자원 최적화**: 3090, 4060 등 서버별 하드웨어 스펙을 K8s가 명확히 인지하도록 `nodeSelector`를 구성하여 GPU 낭비를 차단하십시오.
4. **표준화**: 단일 스크립트를 수동으로 실행하는 단계를 벗어나, 모든 프로세스를 컨테이너 단위로 분할하고 Argo Workflows와 같은 도구를 통해 DAG로 연결하면 완성도 높은 MLOps 시스템이 구축됩니다. 이 과정에서 직접 개발한 런너 모듈을 활용하면 유지보수성이 극대화될 것입니다.
