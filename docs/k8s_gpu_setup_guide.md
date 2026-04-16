# Ubuntu 22.04 LTS 멀티 노드 K8s 클러스터 + GPU 세팅 가이드

> **환경**: Ubuntu 22.04.5 LTS (Jammy Jellyfish)  
> **구성**: Master Node 1대 + GPU Worker Node N대  
> **표기**: `[ALL]` = 모든 노드, `[MASTER]` = Master 노드만, `[WORKER-GPU]` = GPU 장착 Worker 노드만

---

## ✅ 1단계: 사전 준비 `[ALL]`

> [!IMPORTANT]
> **이미 완료한 항목**: `swapoff -a` 실행 ✅ / `/etc/fstab` swap 라인 주석 처리 ✅

### 1-1. 커널 모듈 및 네트워크 설정

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

### 1-2. 적용 확인

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

### 3-3. Containerd에 NVIDIA 런타임 등록

```bash
# Containerd가 NVIDIA GPU를 사용할 수 있도록 설정
sudo nvidia-ctk runtime configure --runtime=containerd

# /etc/containerd/config.toml 에서 default_runtime_name을 확인/수정
# nvidia-ctk 명령이 자동으로 설정하지만, 아래 명령으로 검증
grep -A2 'default_runtime_name' /etc/containerd/config.toml

# Containerd 재시작
sudo systemctl restart containerd
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
> Master 노드에서만 실행합니다. `--pod-network-cidr`은 Calico CNI 기본값입니다. 변경 시 이후 Calico 설정도 함께 변경해야 합니다.

```bash
# Master 노드 IP를 변수에 저장 (실제 Master 노드 IP로 변경)
MASTER_IP="<Master 노드의 내부 IP 주소>"

sudo kubeadm init \
  --apiserver-advertise-address=${MASTER_IP} \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

### 5-1. kubectl 설정 (Master 노드 일반 사용자 계정용)

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 5-2. 클러스터 상태 확인

```bash
kubectl get nodes
# STATUS가 NotReady인 것은 정상 - CNI 설치 전이기 때문
```

---

## 🌐 6단계: 네트워크 플러그인(CNI) 설치 - Calico `[MASTER]`

```bash
# Calico 설치
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml

# Pod가 Running 상태가 될 때까지 대기 (약 1-2분 소요)
watch kubectl get pods -n kube-system

# Node 상태가 Ready로 변경된 것 확인
kubectl get nodes
```

---

## 🔗 7단계: Worker 노드 클러스터 조인 `[WORKER-GPU]`

`kubeadm init` 완료 시 출력된 `kubeadm join` 명령어를 각 Worker 노드에서 실행합니다.

```bash
# 아래는 예시 형식 - 실제 init 완료 시 터미널에 출력된 명령어를 복사해서 사용
sudo kubeadm join <Master-IP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

> [!NOTE]
> join 토큰을 분실했을 경우, Master에서 아래 명령어로 새 토큰을 발급할 수 있습니다.
> ```bash
> kubeadm token create --print-join-command
> ```

### 조인 확인 (Master에서)

```bash
kubectl get nodes
# 모든 노드가 Ready 상태인지 확인
```

---

## 🏷️ 8단계: GPU 노드 라벨링 `[MASTER]`

```bash
# GPU가 있는 Worker 노드에 레이블 추가 (노드 이름은 kubectl get nodes로 확인)
kubectl label nodes <gpu-worker-node-name> accelerator=nvidia-gpu

# 확인
kubectl get nodes --show-labels
```

---

## 🎯 9단계: NVIDIA Device Plugin 배포 `[MASTER]`

이 단계에서 쿠버네티스 스케줄러가 GPU를 자원으로 인식하게 됩니다.

```bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.15.0/deployments/static/nvidia-device-plugin.yml

# DaemonSet 상태 확인 (GPU 노드 수만큼 Pod가 Running이어야 함)
kubectl get pods -n kube-system | grep nvidia
```

### GPU 자원 인식 확인

```bash
# GPU 노드 상세 정보 확인
kubectl describe node <gpu-worker-node-name> | grep -A5 "Capacity"

# 아래와 같이 출력되면 성공!
# Capacity:
#   ...
#   nvidia.com/gpu: 2   <-- GPU 갯수
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
| GPU Pod가 `Pending` 상태 | Device Plugin 미설치 또는 GPU 노드 자원 부족 | `kubectl describe pod <pod>` 로 이벤트 메시지 확인 |
| `nvidia-smi` 정상이나 컨테이너에서 GPU 미인식 | NVIDIA Container Toolkit 미설정 | `sudo nvidia-ctk runtime configure --runtime=containerd` 후 재시작 |
| `kubeadm init` 실패 | swap이 꺼지지 않음 / containerd CRI 소켓 문제 | `swapoff -a` 재확인, `--cri-socket` 옵션 명시 |

---

## 📦 선택: kube-proxy 없이 더 강력한 네트워크를 원한다면

Calico 대신 **Cilium** CNI를 사용하면 eBPF 기반의 더 빠른 네트워크와 강력한 네트워크 정책을 사용할 수 있습니다. GPU AI 워크로드에서 많이 사용합니다.

```bash
# Helm으로 Cilium 설치 (대안)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.15.5 --namespace kube-system
```
