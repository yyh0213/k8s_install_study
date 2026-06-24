# Ubuntu 22.04 LTS 멀티 노드 K8s 클러스터 + GPU 세팅 가이드

> **테스트 환경 (Verified Version)**
> - **OS**: Ubuntu 22.04.5 LTS (Jammy Jellyfish)
> - **Kubernetes**: v1.32.x
> - **Container Runtime**: containerd v2.x
> - **GPU Driver**: NVIDIA Container Toolkit v1.15+
> - **NVIDIA Driver**: 550.x
---

## ✅ 1단계: 사전 준비 `[ALL]`

### 1-1. Swap 메모리 비활성화 (필수 ⭐)
Kubernetes는 파드의 메모리 리소스를 정확하게 관리하기 위해 Swap 메모리 사용을 원칙적으로 허용하지 않습니다. Swap이 활성화되어 있으면 Kubelet이 정상적으로 시작되지 않습니다.

```bash
# 1. 현재 세션에서 즉시 Swap 비활성화
sudo swapoff -a

# 2. 재부팅 시에도 Swap이 비활성화되도록 영구 설정
# /etc/fstab 파일에서 swap이 포함된 라인을 찾아 맨 앞에 주석(#)을 추가합니다.
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 3. systemd GPT Auto-Generator에 의한 자동 활성화 방지 (필수 ⭐)
# Ubuntu 등 systemd 기반 OS는 /etc/fstab 주석 처리와 무관하게 swap 파티션을 자동 감지하여 활성화합니다.
# 이를 방지하기 위해 생성된 systemd swap 유닛을 찾아 완전히 마스크(mask) 처리해야 합니다.
# 우선 활성화된 swap 유닛을 확인합니다:
systemctl list-units --type=swap --all
# 예: dev-sda3.swap 와 같은 유닛이 확인되면 아래 명령어로 마스크 처리합니다.
sudo systemctl mask <swap-unit-name>  # 예: sudo systemctl mask dev-sda3.swap

# 4. Swap 비활성화 정상 확인 (Swap 행의 값이 모두 0이면 성공)
free -h
```

### 1-2. 커널 모듈 및 네트워크 설정

```bash
# br_netfilter 모듈 영구 로드 설정
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl 파라미터 설정 (파드 간 통신을 위한 트래픽 포워딩)
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 설정 즉시 적용
sudo sysctl --system
```

### 1-3. 적용 확인

```bash
# 아래 값들이 모두 1이 나오면 정상
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward
```

---

## 🐋 2단계: Containerd 컨테이너 런타임 설치 `[ALL]`

### 2-1. 설치

```bash
sudo apt-get update
sudo apt-get install -y containerd
```

### 2-2. 기본 설정 파일 생성 및 CgroupDriver 설정

```bash
# 기본 설정 파일 생성
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# systemd cgroup driver 활성화 (쿠버네티스 권장 방식)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Containerd 재시작 및 부팅 시 자동 시작 등록
sudo systemctl restart containerd
sudo systemctl enable containerd

# 상태 확인
sudo systemctl status containerd
```

---

## 🎮 3단계: NVIDIA GPU 드라이버 및 Container Toolkit 설치 `[WORKER-GPU]`

> [!WARNING]
> 아래 단계는 **GPU가 장착된 Worker 노드에서만** 실행합니다.

### 3-1. NVIDIA 드라이버 설치

```bash
# 설치 가능한 드라이버 목록 확인
sudo ubuntu-drivers devices

# 권장 드라이버 자동 설치 (권장)
sudo ubuntu-drivers autoinstall

# 또는 특정 버전 지정 설치 (예: 550 버전)
# sudo apt-get install -y nvidia-driver-550
```

> [!IMPORTANT]
> 드라이버 설치 후 **반드시 서버를 재부팅** 해야 합니다.
> ```bash
> sudo reboot
> ```
> 재부팅 후 `nvidia-smi` 명령어로 GPU 인식 여부를 확인하세요.

### 3-2. NVIDIA Container Toolkit 설치

```bash
# NVIDIA 저장소 키 및 소스 추가
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

### 3-3. Containerd에 NVIDIA 런타임 강제 등록 (핵심 ⭐)

가장 에러가 많이 발생하는 구간입니다. 설정을 초기화한 후 확실하게 등록해야 합니다.
최신 nvidia-ctk는 메인 설정 파일(config.toml)을 직접 덮어쓰지 않고, conf.d 디렉토리에 분할 설정 파일(Drop-in)을 생성하는 방식을 사용합니다.
```bash
# 1. containerd 설정을 기본값으로 초기화 및 conf.d 디렉토리 활성화
sudo rm -f /etc/containerd/config.toml
containerd config default | sudo tee /etc/containerd/config.toml

# 2. config.toml 파일에 imports 구문이 있는지 확인 및 추가 (중요)
# 최신 containerd는 보통 기본 포함되어 있으나, 확실히 하기 위해 실행합니다.
if ! grep -q 'imports = \["/etc/containerd/conf.d/\*.toml"\]' /etc/containerd/config.toml; then
  sudo sed -i '1s/^/imports = ["\/etc\/containerd\/conf.d\/*.toml"]\n/' /etc/containerd/config.toml
fi

# 3. conf.d 디렉토리 생성
sudo mkdir -p /etc/containerd/conf.d

# 4. NVIDIA 런타임 설정 주입 (conf.d/99-nvidia.toml 파일로 생성됨)
sudo nvidia-ctk runtime configure --runtime=containerd

# 5. 쿠버네티스 Cgroup 드라이버(systemd) 활성화
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 6. 서비스 재시작
sudo systemctl restart containerd
```

### 3-4. 설정 검증 (필수 ⭐)

다음 단계로 넘어가기 전에 설정 파일(`config.toml`)이 정상적으로 수정되었는지 반드시 확인하세요.

```bash
# 1. SystemdCgroup 정상 변경 여부 확인
grep 'SystemdCgroup = true' /etc/containerd/config.toml
# 정상이라면 콘솔에 'SystemdCgroup = true' 가 출력되어야 합니다.

# 2. 외부 설정 파일 Import 구문 확인
head -n 5 /etc/containerd/config.toml | grep 'imports'
# 정상이라면 'imports = ["/etc/containerd/conf.d/*.toml"]' 이 출력되어야 합니다.

# 3. NVIDIA 런타임 분할 설정 파일 확인
cat /etc/containerd/conf.d/99-nvidia.toml
# 정상이라면 [plugins."io.containerd.cri.v1.runtime".containerd.runtimes.nvidia] 로 시작하는 설정 블록이 출력되어야 합니다.

# (만약 위 명령어 중 하나라도 예상된 출력이 나오지 않는다면, 3-3 단계를 다시 실행해 주세요.)
```

---

## ☸️ 4단계: Kubeadm / Kubelet / Kubectl 설치 `[ALL]`

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# 쿠버네티스 공식 저장소 추가 (v1.32 기준 - 2025년 최신 stable)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# 버전이 자동으로 업그레이드되지 않도록 고정
sudo apt-mark hold kubelet kubeadm kubectl

# kubelet 시작
sudo systemctl enable --now kubelet
```

---

## 🏗️ 5단계: 클러스터 초기화 `[MASTER]`

> [!IMPORTANT]
> Master 노드에서만 실행합니다. `--pod-network-cidr`은 Calico CNI 기본값입니다.

```bash
# Master 노드 IP를 변수에 저장 (실제 Master 노드 IP로 변경)
MASTER_IP="192.168.1.153" # 이미 설정되어 있다면 패스

sudo kubeadm init \
  --apiserver-advertise-address=${MASTER_IP} \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

### 5-1. kubectl 설정 (필수 ⭐)

초기화 성공 후, 현재 로그인한 사용자 계정에서 `kubectl`을 사용할 수 있게 권한을 가져옵니다.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 🛣️ 6단계: 구성 시나리오 선택

서버 환경에 따라 아래 **A** 또는 **B** 중 하나를 선택하여 진행하세요.

### **[시나리오 A] 서버 1대만 사용하는 경우 (단일 노드)**
마스터 노드에서도 파드(Pod)가 실행될 수 있도록 스케줄링 제한을 해제합니다. (v1.24+ 기준)

```bash
# 마스터 노드에서도 워크로드 실행 허용 (Taint 제거)
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### **[시나리오 B] 서버 여러 대를 연결하는 경우 (다중 노드)**
다른 서버(Worker)들을 마스터에 연결합니다. 기존에 연결된 워커 노드는 재등록할 필요가 없으며, 신규 워커 노드를 추가할 때만 아래 과정을 진행합니다.

1. **Master 노드에서 연결 명령어 발급 (최초 1회 발급된 토큰은 24시간 후 만료됨)**
워커 노드를 붙이기 전, 마스터 노드에서 새로운 임시 토큰과 조인(join) 명령어를 발급받습니다.
   ```bash
   sudo kubeadm token create --print-join-command
   ```
   (위 명령어 실행 시 출력되는 kubeadm join ... 전체 줄을 복사합니다.)
2. **Worker 노드에서 실행**: 복사한 `join` 명령어를 Worker 노드에서 실행합니다.
   ```bash
   sudo kubeadm join <Master-IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```
3. **Master 노드에서 확인**:
   ```bash
   kubectl get nodes
   ```

---

## 🌐 7단계: 네트워크 플러그인(CNI) 설치 `[MASTER]`

단일/다중 노드 상관없이 반드시 설치해야 합니다.

```bash
# Calico 설치
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml

# 상태 확인 (모든 Pod가 Running이 될 때까지 대기)
watch kubectl get pods -n kube-system
```

---

## 🎮 8단계: Helm 및 NVIDIA GPU Operator 설치 `[MASTER]`

NVIDIA GPU Operator는 Kubernetes 클러스터의 GPU 노드를 자동으로 관리해 주는 솔루션입니다. 여기서는 Helm을 사용하여 GPU Operator를 배포합니다.

### 8-1. Helm 설치
Kubernetes 패키지 매니저인 Helm이 설치되어 있지 않다면 아래 명령어로 설치합니다.

```bash
# Helm 설치 스크립트 다운로드 및 실행
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 설치 확인
helm version
```

### 8-2. NVIDIA GPU Operator 설치
GPU Operator Helm 저장소를 추가하고, Host에 이미 GPU 드라이버와 Container Toolkit이 설치된 상태이므로 해당 기능의 자동 설치 옵션을 비활성화(`enabled=false`)하여 배포합니다.

```bash
# 1. NVIDIA Helm 저장소 추가 및 업데이트
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

# 2. GPU Operator 네임스페이스 및 권한 설정 (PSA 설정 포함)
kubectl create namespace gpu-operator
kubectl label --overwrite namespace gpu-operator pod-security.kubernetes.io/enforce=privileged

# 3. GPU Operator 설치
# (Host에 이미 드라이버와 툴킷이 설치된 상태이므로 driver.enabled=false, toolkit.enabled=false 설정)
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --set driver.enabled=false \
  --set toolkit.enabled=false

# 4. 배포 상태 확인 (모든 Pod가 Running이 될 때까지 대기)
kubectl get pods -n gpu-operator
```

> [!NOTE]
> 만약 Worker 노드에 NVIDIA Driver와 Container Toolkit이 설치되어 있지 않고, GPU Operator가 컨테이너 기반으로 이들을 자동 설치하도록 하려면 `--set driver.enabled=false --set toolkit.enabled=false` 옵션을 제외하고 설치해야 합니다.

### 8-3. GPU 인식 확인
GPU Operator가 정상 설치되면 쿠버네티스가 GPU를 자원으로 인식합니다.

```bash
# GPU 인식 확인
kubectl describe node | grep -A20 "Capacity"
# nvidia.com/gpu: <갯수> 가 보이면 성공!
```

---

## 🖥️ 9단계: Lens를 이용한 GUI 관리 (추천 ⭐)

CLI(명령어)가 익숙지 않다면, 시각적으로 클러스터를 관리할 수 있는 **Lens** 사용을 강력히 추천합니다. 서버에 따로 무거운 프로그램을 깔지 않아도 됩니다.

### 1. 내 PC에 Lens 설치
1. [Lens 홈페이지](https://k8slens.dev/)에 접속하여 OS(Windows/Mac)에 맞는 버전을 다운로드하고 설치합니다. (개인용 Personal 버전은 무료입니다.)

### 2. 접속 설정 파일(kubeconfig) 가져오기
서버의 접속 정보를 내 PC로 가져와야 합니다.
1. 마스터 노드 터미널에서 아래 명령어로 설정 파일 내용을 출력합니다.
   ```bash
   cat ~/.kube/config
   ```
2. 출력된 텍스트 전체를 복사합니다.

### 3. Lens에 클러스터 등록
1. Lens 앱을 실행합니다.
2. 좌측 트리에서 KUBERNETES CLUSTERS - Local Kuberconfigs 우측에 마우스를 올리면 + 버튼이 보입니다. 클릭합니다.
3. add Kubeconfig를 선택합니다.
4. 아까 cat ~/.kube/config로 복사했던 정보를 붙여 넣습니다.
5. **Connect**를 누르면 서버의 CPU, 메모리, GPU 사용량과 Pod 리스트를 실시간 GUI로 볼 수 있습니다.

---

## 💡 팁: 유용한 명령어 모음

### 모든 Pod 상태 확인 (모든 네임스페이스)
```bash
kubectl get pods -A
```

### 서비스 로그 실시간 확인
```bash
kubectl logs -f <pod-name>
```

### Pod에 접속해서 Bash 띄우기
```bash
kubectl exec -it <pod-name> -- /bin/bash
```

---

## 🧪 10단계: GPU Pod 할당 동작 테스트

```bash
# test-gpu-pod.yaml 파일 생성
cat <<EOF > test-gpu-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  restartPolicy: OnFailure
  containers:
  - name: cuda-vector-add
    image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04
    resources:
      limits:
        nvidia.com/gpu: 1
EOF

# Pod 배포
kubectl apply -f test-gpu-pod.yaml

# 상태 확인 (Completed가 되면 성공)
kubectl get pod gpu-test

# 로그로 GPU 연산 결과 확인
kubectl logs gpu-test
```

정상 동작 시 아래와 같은 로그가 출력됩니다:
```
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

---

## 🔧 트러블슈팅 체크리스트

| 증상 | 원인 | 해결책 |
|------|------|--------|
| `kubectl get nodes`에서 노드가 `NotReady` | CNI 미설치 또는 containerd 문제 | Calico 설치 확인, `sudo systemctl status containerd` |
| GPU Pod가 `Pending` 상태 | GPU Operator/Device Plugin 미설치 또는 GPU 노드 자원 부족 | `kubectl describe pod <pod>` 로 이벤트 메시지 확인 |
| `nvidia-smi` 정상이나 컨테이너에서 GPU 미인식 | NVIDIA Container Toolkit 미설정 | `sudo nvidia-ctk runtime configure --runtime=containerd` 후 재시작 |
| `kubeadm init` 실패 | swap이 꺼지지 않음 / containerd CRI 소켓 문제 | `swapoff -a` 재확인, `--cri-socket` 옵션 명시 |

---

---

## 🔐 11단계: 사내 비공개 레지스트리(Harbor) 연동

Harbor는 비공개 서버이므로, 쿠버네티스가 이미지를 가져올 때 **"로그인 정보(열쇠)"**가 필요합니다. 이 설정이 없으면 `ImagePullBackOff` 에러가 발생합니다.

### 1. Harbor 로그인 정보 Secret 생성
쿠버네티스 클러스터에서 사용할 수 있는 로그인 열쇠를 생성합니다.

```bash
kubectl create secret docker-registry harbor-registry-secret \
  --docker-server=<HARBOR_URL> \
  --docker-username=<USER> \
  --docker-password=<PASSWORD>
```
*   `harbor-registry-secret`: 내가 정한 Secret의 이름입니다.
*   `<HARBOR_URL>`: 사내 Harbor 주소 (예: harbor.company.com)

### 2. YAML 파일에 열쇠 명시하기
파드 정의 시 아래와 같이 `imagePullSecrets`를 명시해야 해당 열쇠를 써서 이미지를 가져옵니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-ai-app
spec:
  # 생성한 Secret 이름을 여기에 명시합니다 ⭐
  imagePullSecrets:
  - name: harbor-registry-secret
  containers:
  - name: ai-container
    # Harbor 주소가 포함된 이미지 경로를 사용합니다
    image: <HARBOR_URL>/project-name/image-name:tag
    resources:
      limits:
        nvidia.com/gpu: 1
```

---

## 📦 선택: kube-proxy 없이 더 강력한 네트워크를 원한다면

Calico 대신 **Cilium** CNI를 사용하면 eBPF 기반의 더 빠른 네트워크와 강력한 네트워크 정책을 사용할 수 있습니다. GPU AI 워크로드에서 많이 사용합니다.

```bash
# Helm으로 Cilium 설치 (대안)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.15.5 --namespace kube-system
```

---

## 📚 부록: 실전 Pod 작성 치트 시트 (Cheat Sheet)

실무에서 자주 쓰이는 파드 설정 패턴들을 정리합니다.

### 1. 컨테이너를 죽지 않게 유지하기 (Keep-alive)
AI 모델 학습이나 전처리용 파드는 작업이 끝나면 바로 종료됩니다. 내부 접속 후 디버깅을 하거나 코드를 수정하려면 컨테이너를 계속 켜둬야 합니다.
*   **패턴**: `command`와 `args`에 무한 대기 명령어를 넣습니다.
```yaml
containers:
- name: preprocessing
  image: <HARBOR_URL>/ai-team/preprocessing:latest
  command: ["/bin/sh", "-c"]
  args: ["tail -f /dev/null"] # 이 명령어가 파드를 무한 대기 상태로 유지합니다.
```

### 2. 호스트 폴더 연결하기 (Volume Mount)
서버의 특정 폴더(예: NAS 마운트 지점, 데이터셋 폴더)를 파드 내부로 연결할 때 사용합니다.
```yaml
spec:
  volumes:
  - name: dataset-storage
    hostPath:
      path: /mnt/nas/data # 실제 서버(Host)의 경로
  containers:
  - name: worker
    volumeMounts:
    - name: dataset-storage
      mountPath: /data # 파드 내부에서 접근할 경로
```

### 3. 외부 브라우저에서 접속 가능하게 만들기 (Service)
파드 안에서 띄운 웹 서버(예: Jupyter Notebook, Flask)를 내 PC 브라우저에서 접속하고 싶을 때 사용합니다.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-web-service
spec:
  type: NodePort
  ports:
  - port: 80         # 파드 내부용 포트
    targetPort: 8888 # 컨테이너 앱 포트 (예: Jupyter)
    nodePort: 30005  # 사용자 접속 포트 (서버IP:30005 로 접속)
  selector:
    name: preprocessing # 파드의 metadata.name과 일치해야 함
```

### 4. 환경 변수 동적 설정 (Environment Variables)
코드 수정 없이 학습 파라미터 등을 바꿀 때 유용합니다.
```yaml
containers:
- name: training-pod
  env:
  - name: LEARNING_RATE
    value: "0.001"
  - name: BATCH_SIZE
    value: "64"
```
