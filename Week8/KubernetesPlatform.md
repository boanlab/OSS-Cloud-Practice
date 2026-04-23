# 08. 쿠버네티스 플랫폼 기초 및 설치 (kubeadm)

> (본 실습은 Ubuntu 24.04 VM 환경을 기준으로 하며, **Manager(Control Plane) 1대 + Worker 1대, 총 2노드 클러스터**를 권장합니다.)
>
> 최소 사양: 각 노드 2 vCPU / 2GB RAM 이상, 노드 간 사설 네트워크 통신 가능.
>
> 본 문서는 쿠버네티스 v1.30 / containerd 기반 설치를 기준으로 합니다.

---

## Step 1. 쿠버네티스 개요 (PDF 2~6쪽)
쿠버네티스(Kubernetes, K8s)는 컨테이너화된 애플리케이션의 배포·확장·운영을 자동화해 주는 오픈소스 **컨테이너 오케스트레이션 플랫폼**입니다. Docker Swarm과 달리 대규모 분산 환경을 전제로 설계되어 있으며, 선언적(Declarative) API를 통해 "원하는 상태(desired state)"를 명시하면 컨트롤 플레인이 이를 유지하도록 동작합니다.

### 주요 컴포넌트 요약
| 구분 | 컴포넌트 | 설명 |
| :--- | :--- | :--- |
| Control Plane | `kube-apiserver` | 모든 요청을 받는 클러스터의 관문(API 서버) |
| Control Plane | `etcd` | 클러스터의 모든 상태를 저장하는 분산 키-값 저장소 |
| Control Plane | `kube-scheduler` | 새 Pod를 어떤 노드에 배치할지 결정 |
| Control Plane | `kube-controller-manager` | 각종 컨트롤러(노드·레플리카·엔드포인트 등) 실행 |
| Worker Node | `kubelet` | 각 노드에서 Pod를 실행·관리하는 에이전트 |
| Worker Node | `kube-proxy` | 서비스 네트워크 규칙(iptables/IPVS) 관리 |
| Worker Node | `container runtime` | 실제 컨테이너 실행 엔진 (본 실습은 containerd 사용) |
| Add-on | `CNI 플러그인` | Pod 네트워크 제공 (본 실습은 Calico 사용) |

### 핵심 오브젝트
- **Pod**: 쿠버네티스에서 배포 가능한 가장 작은 단위. 1개 이상의 컨테이너 묶음.
- **Deployment**: Pod의 원하는 개수와 업데이트 전략을 선언적으로 관리.
- **Service**: Pod 집합에 고정된 접근점(IP/DNS)을 제공하는 네트워크 추상화.
- **Namespace**: 클러스터 리소스를 논리적으로 분리.

---

## Step 2. 사전 준비 (모든 노드에서 실행) (PDF 7~9쪽)
쿠버네티스는 Swap이 켜져 있으면 kubelet이 기동하지 않으며, 컨테이너 간 트래픽이 브리지를 거치도록 커널 파라미터를 조정해야 합니다.

```bash
# 1. 시스템 업데이트
sudo apt update && sudo apt -y upgrade
```
![figure1](./images/figure1.png)

```bash
# 2. Swap 비활성화 (런타임 + 영구)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 확인: Swap 값이 0이어야 정상
free -h
```
![figure2](./images/figure2.png)

```bash
# 3. 필요한 커널 모듈 로드 (overlay, br_netfilter)
cat << 'EOF' | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```
![figure3](./images/figure3.png)

```bash
# 4. sysctl 네트워크 파라미터 설정 (브리지 트래픽 iptables 경유 + IP 포워딩)
cat << 'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```
![figure4](./images/figure4.png)

```bash
# 5. (선택) 호스트명/hosts 설정 — 노드 식별이 쉬워집니다
# Master 노드에서
sudo hostnamectl set-hostname k8s-master
# Worker 노드에서
sudo hostnamectl set-hostname k8s-worker1

# 모든 노드에서 /etc/hosts 편집 — 각 노드의 IP 기입
sudo tee -a /etc/hosts << 'EOF'
192.168.56.10  k8s-master
192.168.56.11  k8s-worker1
EOF
```
![figure5](./images/figure5.png)

---

## Step 3. 컨테이너 런타임 설치 (containerd, 모든 노드) (PDF 10~12쪽)
쿠버네티스 v1.24 이후로 Dockershim이 제거되어 **containerd**를 런타임으로 직접 사용합니다.

```bash
# 1. containerd 설치
sudo apt update
sudo apt -y install containerd
```
![figure6](./images/figure6.png)

```bash
# 2. 기본 설정 파일 생성
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```
![figure7](./images/figure7.png)

```bash
# 3. systemd cgroup 드라이버 사용 (kubelet 기본값과 맞추기)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 4. containerd 재시작 및 상태 확인
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd --no-pager
```
![figure8](./images/figure8.png)

---

## Step 4. kubeadm · kubelet · kubectl 설치 (모든 노드) (PDF 13~15쪽)
쿠버네티스 공식 APT 저장소(`pkgs.k8s.io`)를 등록하고 세 패키지를 같은 버전으로 설치합니다.

```bash
# 1. 필수 패키지 설치
sudo apt -y install apt-transport-https ca-certificates curl gpg
```
![figure9](./images/figure9.png)

```bash
# 2. 쿠버네티스 저장소 키 등록
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 3. 저장소 추가 (v1.30 stable 채널)
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
![figure10](./images/figure10.png)

```bash
# 4. kubeadm / kubelet / kubectl 설치 및 버전 고정 (자동 업데이트 방지)
sudo apt update
sudo apt -y install kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
![figure11](./images/figure11.png)

```bash
# 5. kubelet 활성화 (init 전에는 CrashLoop 상태가 정상 — 걱정하지 않아도 됩니다)
sudo systemctl enable --now kubelet

# 설치 버전 확인
kubeadm version
kubectl version --client
```
![figure12](./images/figure12.png)

---

## Step 5. Control Plane 초기화 (Master 노드에서만) (PDF 16~18쪽)
`kubeadm init`으로 컨트롤 플레인을 부트스트랩합니다. `--pod-network-cidr`은 이후 CNI(Calico) 기본값과 일치시킵니다.

```bash
# 1. 컨트롤 플레인 초기화
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<MASTER-IP>
```
![figure13](./images/figure13.png)

> 출력 마지막에 표시되는 `kubeadm join ...` 명령어를 **반드시 복사**해 두세요. Worker 노드 조인에 사용됩니다.

```bash
# 2. 일반 사용자 kubectl 설정 (Master 노드의 현재 사용자 기준)
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
![figure14](./images/figure14.png)

```bash
# 3. 현재 상태 확인 — CNI 설치 전이므로 노드는 NotReady 상태가 정상입니다
kubectl get nodes
kubectl get pods -n kube-system
```
![figure15](./images/figure15.png)

### CNI 설치 (Calico)
Pod 간 네트워크를 제공하는 CNI 플러그인을 설치해야 노드가 `Ready`로 전환됩니다.

```bash
# 1. Calico Operator 설치
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

# 2. Calico 커스텀 리소스 적용
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```
![figure16](./images/figure16.png)

```bash
# 3. Calico Pod가 모두 Running 상태가 될 때까지 대기 (1~2분 소요)
watch kubectl get pods -n calico-system

# Ctrl+C 로 watch 종료 후 노드 상태 재확인 — Master가 Ready로 바뀝니다
kubectl get nodes
```
![figure17](./images/figure17.png)

---

## Step 6. Worker 노드 조인 (Worker 노드에서 실행) (PDF 19쪽)
Step 5에서 복사해 둔 `kubeadm join` 명령어를 그대로 실행합니다. 토큰을 분실했다면 Master 노드에서 새로 발급할 수 있습니다.

```bash
# [Master 노드에서] 토큰 분실 시 join 명령어 재생성
kubeadm token create --print-join-command
```
![figure18](./images/figure18.png)

```bash
# [Worker 노드에서] 위 명령 출력값 그대로 sudo를 붙여 실행
sudo kubeadm join <MASTER-IP>:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```
![figure19](./images/figure19.png)

```bash
# [Master 노드에서] 조인 결과 확인 — Worker가 Ready 상태로 보여야 정상
kubectl get nodes -o wide
```
![figure20](./images/figure20.png)

---

## Step 7. 클러스터 상태 점검 (PDF 20쪽)
배포 실습 전에 컨트롤 플레인과 시스템 Pod가 정상인지 확인합니다.

```bash
# 1. 노드 상태
kubectl get nodes
```
![figure21](./images/figure21.png)

```bash
# 2. 시스템 네임스페이스 Pod 상태 (모두 Running 이어야 함)
kubectl get pods -A
```
![figure22](./images/figure22.png)

```bash
# 3. 컨트롤 플레인 핵심 컴포넌트 상태
kubectl get componentstatuses 2>/dev/null || kubectl get pods -n kube-system
```
![figure23](./images/figure23.png)

```bash
# 4. 클러스터 정보
kubectl cluster-info
```
![figure24](./images/figure24.png)

---

## Step 8. [실습] 첫 애플리케이션 배포 (PDF 21~23쪽)
클러스터 동작 확인을 위해 nginx Deployment를 배포하고 Service로 외부에 노출해 봅니다.

### 1. Deployment 생성
```bash
# 레플리카 2개짜리 nginx Deployment
kubectl create deployment nginx-demo --image=nginx:alpine --replicas=2
```
![figure25](./images/figure25.png)

```bash
# Pod가 Worker 노드에 배치되는지 확인
kubectl get pods -o wide
```
![figure26](./images/figure26.png)

### 2. Service로 노출 (NodePort)
```bash
# 80 포트를 NodePort 타입으로 노출
kubectl expose deployment nginx-demo --type=NodePort --port=80
```
![figure27](./images/figure27.png)

```bash
# 할당된 NodePort 확인 (30000~32767 범위)
kubectl get svc nginx-demo
```
![figure28](./images/figure28.png)

```bash
# 접속 테스트 — 어느 노드 IP로 접속해도 라우팅됩니다
curl http://<ANY-NODE-IP>:<NODEPORT>
```
![figure29](./images/figure29.png)

### 3. 스케일링
```bash
# 레플리카 5개로 확장
kubectl scale deployment nginx-demo --replicas=5
kubectl get pods -o wide
```
![figure30](./images/figure30.png)

### 4. Pod 자가 복구 확인
```bash
# Pod 하나를 강제로 삭제
kubectl delete pod <POD-NAME>

# Deployment가 자동으로 새 Pod를 생성해 원하는 레플리카 수를 복구
kubectl get pods -o wide
```
![figure31](./images/figure31.png)

---

## Step 9. 리소스 정리
```bash
# 실습용 리소스 삭제
kubectl delete svc nginx-demo
kubectl delete deployment nginx-demo
```
![figure32](./images/figure32.png)

### (선택) 클러스터 자체를 초기화하려면
> 다음 명령은 노드에서 쿠버네티스 설정을 **전부 제거**합니다. 실습 환경을 재사용하지 않을 때만 실행하세요.

```bash
# [Worker 노드] 클러스터 탈퇴
sudo kubeadm reset -f

# [Master 노드] 컨트롤 플레인 초기화 해제
sudo kubeadm reset -f
sudo rm -rf $HOME/.kube /etc/cni/net.d
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
```

---

## 트러블슈팅 체크리스트
| 증상 | 주요 원인 / 조치 |
| :--- | :--- |
| `kubelet`이 CrashLoop | Swap이 켜져 있음 → `sudo swapoff -a` 후 `/etc/fstab` 수정 |
| 노드가 계속 `NotReady` | CNI 미설치 또는 `calico-system` Pod 비정상 → `kubectl get pods -n calico-system` 로그 확인 |
| `kubeadm init` 실패 | `sudo kubeadm reset -f` 후 재실행, 커널 모듈/sysctl 재확인 |
| `kubeadm join` 실패 | 토큰 만료(24시간) → Master에서 `kubeadm token create --print-join-command` 재발급 |
| Pod가 `ImagePullBackOff` | 이미지 이름/태그 오타, 사설 레지스트리 인증 누락 |
