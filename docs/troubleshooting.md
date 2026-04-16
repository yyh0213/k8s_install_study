# 🛠️ Kubernetes 설치 트러블슈팅 가이드

이 문서는 쿠버네티스 클러스터 구축 중 발생할 수 있는 주요 장애 상황과 해결 방법을 정리합니다.

---

## 🚨 Case 1: Kubelet Unhealthy (Wait-control-plane Timeout)

### **현상**
`kubeadm init` 실행 중 아래 메시지에서 멈춘 뒤 실패함:
> `[kubelet-check] The kubelet is not healthy after 4m0s`  
> `error: Get "http://127.0.0.1:10248/healthz": dial tcp 127.0.0.1:10248: connect: connection refused`

### **원인**
1. **Containerd 설정 미반영**: `config.toml` 수정 후 서비스를 재시작하지 않음.
2. **Cgroup Driver 불일치**: K8s는 `systemd`를 원하는데 Containerd는 `cgroupfs`를 사용 중일 때 발생.
3. **Swap 활성화**: Swap이 켜져 있으면 Kubelet이 시작되지 않음.

### **해결 방법**

**1. 실패한 상태 초기화**
```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes/
sudo rm -rf $HOME/.kube/
```

**2. Cgroup Driver 설정 확인 및 강제 적용**
`containerd`의 설정 파일에서 `SystemdCgroup` 값을 `true`로 확실하게 바꿔야 합니다.
```bash
# 설정 파일 내 SystemdCgroup 위치 확인
grep -n "SystemdCgroup" /etc/containerd/config.toml

# 만약 false로 되어있거나 검색이 안 된다면 아래 명령어로 강제 수정 (v2/v3 공통)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 서비스 재시작 (필수!)
sudo systemctl restart containerd
```

**3. Swap 비활성화 재확인**
```bash
sudo swapoff -a
# free -h 명령어로 Swap이 0B인지 확인
```

---

## 🚨 Case 2: Kubeadm 초기화 시 "Port 6443 is already in use"

### **현상**
`kubeadm init` 재시도 시 아래 에러 발생:
> `[preflight] Some fatal errors occurred: [ERROR Port-6443]: Port 6443 is already in use`

### **원인**
이전에 실행했던 `kubeadm init`의 프로세스나 설정이 남아있음.

### **해결 방법**
```bash
sudo kubeadm reset -f
```

---

## 🚨 Case 3: GPU Pod는 Running인데 kubectl describe에 GPU가 안 보임

### **현상**
NVIDIA Device Plugin 파드는 정상 작동 중이나, `kubectl describe node` 시 `nvidia.com/gpu` 항목이 아예 없거나 0임. 

### **원인**
`containerd`가 NVIDIA 라이브러리(`libnvidia-ml.so.1`)를 컨테이너 내부로 바인딩하지 못함. 보통 `containerd`의 `config.toml` 설정이 꼬였을 때 발생.

### **확인 방법 (플러그인 로그)**
```bash
kubectl logs -n kube-system <nvidia-device-plugin-pod-name>
# 로그에 "could not load NVML library" 또는 "Incompatible platform" 메시지 확인
```

### **해결 방법**
`containerd` 설정을 완전히 초기화하고 다시 잡는 것이 가장 빠릅니다.
```bash
# 1. 설정 초기화
sudo rm /etc/containerd/config.toml
containerd config default | sudo tee /etc/containerd/config.toml

# 2. NVIDIA 런타임 기본값 설정 및 systemd 적용
sudo nvidia-ctk runtime configure --runtime=containerd --set-as-default
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 3. 서비스 재시작 및 관련 파드 재로그인
sudo systemctl restart containerd
kubectl delete pod -n kube-system -l app.kubernetes.io/name=nvidia-device-plugin # 라벨에 맞춰 삭제
```

문제가 해결되지 않을 때는 에러 메시지보다는 **시스템 로그**를 직접 확인해야 합니다.

1. **Kubelet 로그 실시간 확인** (가장 중요)
   ```bash
   journalctl -xeu kubelet -f
   ```
2. **Containerd 상태 확인**
   ```bash
   systemctl status containerd
   ```
3. **실행 중인 컨테이너 확인 (Crictl)**
   ```bash
   sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps -a
   ```

---

## 💡 팁: 마스터 노드에 Pod 배포 허용 (Single Node)
서버 1대로 워커 노드 역할까지 수행하게 하려면 설치 완료 후 아래 명령어를 입력해야 합니다.
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-
```
