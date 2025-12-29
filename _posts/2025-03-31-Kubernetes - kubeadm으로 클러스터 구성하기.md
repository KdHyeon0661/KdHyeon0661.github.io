---
layout: post
title: Kubernetes - kubeadm으로 클러스터 구성하기
date: 2025-03-31 23:20:23 +0900
category: Kubernetes
---
# kubeadm으로 실제 서버에 쿠버네티스 클러스터 구성하기: 실전 가이드

## 소개: kubeadm을 활용한 쿠버네티스 클러스터 구축

kubeadm은 쿠버네티스 공식 프로젝트에서 제공하는 클러스터 부트스트랩 도구로, 최소한의 운영 노력으로 프로덕션 준비 클러스터를 구성할 수 있도록 설계되었습니다. 이 가이드는 실제 서버 환경에서 kubeadm을 사용하여 완전한 쿠버네티스 클러스터를 구축하는 모든 단계를 상세히 다룹니다.

기존 문서에서 다루는 기본 흐름을 확장하여 시스템 튜닝(커널/방화벽/cgroups), containerd 표준 설정, 공식 APT 패키지 관리, 구성 파일 기반 클러스터 설정, 운영 및 유지보수 절차, 일반적인 문제 해결 방법을 포함합니다.

---

## 목표 아키텍처와 환경 구성

이 가이드에서는 다음과 같은 기본 아키텍처를 구성합니다:

- **제어 평면(Control Plane) 노드 1대**: `kube-apiserver`, `etcd`, `kube-controller-manager`, `kube-scheduler` 실행
- **작업자(Worker) 노드 N대**: 애플리케이션 파드 실행

네트워크 구성 예시:
```
[ 제어 평면 (192.168.56.101) ]
        ↑ 6443/TCP (kube-apiserver)
        │
        ├── 작업자-1 (192.168.56.102)
        └── 작업자-2 (192.168.56.103)
```

프로덕션 환경에서는 고가용성을 위해 여러 제어 평면 노드와 외부 로드 밸런서(kube-vip 등)를 고려해야 합니다.

---

## 모든 노드에 적용할 공통 시스템 준비

### 호스트 이름 및 호스트 파일 설정

각 노드에서 적절한 호스트 이름을 설정하고, 모든 노드의 IP 주소와 호스트 이름을 `/etc/hosts` 파일에 추가합니다:

```bash
# 제어 평면 노드에서
sudo hostnamectl set-hostname control-plane

# 작업자 노드 1에서
sudo hostnamectl set-hostname worker-1

# 작업자 노드 2에서
sudo hostnamectl set-hostname worker-2
```

모든 노드의 `/etc/hosts` 파일에 다음 항목을 추가합니다:
```bash
echo "192.168.56.101 control-plane" | sudo tee -a /etc/hosts
echo "192.168.56.102 worker-1" | sudo tee -a /etc/hosts
echo "192.168.56.103 worker-2" | sudo tee -a /etc/hosts
```

### 스왑 메모리 비활성화

쿠버네티스 kubelet은 스왑 메모리가 활성화된 상태에서 정상적으로 작동하지 않습니다:

```bash
# 현재 스왑 비활성화
sudo swapoff -a

# 부팅 시 자동 활성화 방지
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 커널 모듈 및 네트워크 필터링 설정

컨테이너 네트워킹과 오버레이 파일시스템을 위한 커널 모듈을 로드하고 네트워크 설정을 구성합니다:

```bash
# 필요한 커널 모듈 로드 설정
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# 모듈 즉시 로드
sudo modprobe overlay
sudo modprobe br_netfilter

# 커널 파라미터 설정
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

# 설정 적용
sudo sysctl --system
```

### 방화벽 구성

테스트 환경에서는 방화벽을 일시적으로 비활성화할 수 있지만, 프로덕션 환경에서는 필요한 포트만 허용하는 것이 좋습니다:

```bash
# 테스트 목적으로 방화벽 비활성화 (프로덕션에서는 권장하지 않음)
sudo ufw disable
```

프로덕션 환경에서 허용해야 할 주요 포트:
- **6443**: kube-apiserver
- **10250**: kubelet API
- **2379-2380**: etcd 서버 및 클라이언트 통신
- **10257**: kube-controller-manager
- **10259**: kube-scheduler
- **30000-32767**: NodePort 서비스 범위
- **CNI 관련 포트**: 사용하는 CNI 플러그인에 따라 다름

---

## 컨테이너 런타임: containerd 구성

containerd는 쿠버네티스 권장 컨테이너 런타임으로, Docker보다 가볍고 표준화된 인터페이스를 제공합니다.

### containerd 설치 및 기본 설정

```bash
# 패키지 업데이트 및 containerd 설치
sudo apt update
sudo apt install -y containerd

# 설정 디렉토리 생성 및 기본 설정 생성
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

### Systemd Cgroup 드라이버 활성화

쿠버네티스 kubelet과 일관된 cgroup 드라이버를 사용하려면 containerd 설정을 수정해야 합니다:

```bash
# Systemd Cgroup 드라이버 활성화
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# containerd 서비스 활성화 및 시작
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

---

## 쿠버네티스 컴포넌트 설치

### 공식 APT 저장소 등록

공식 쿠버네티스 패키지 저장소를 시스템에 등록합니다:

```bash
# 필수 패키지 설치
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg

# 공식 GPG 키 등록 (apt-key 대신 권장 방식)
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# APT 저장소 목록에 쿠버네티스 저장소 추가
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```

**참고**: 버전 트랙은 사용할 쿠버네티스 버전에 따라 `v1.29`, `v1.31` 등으로 조정하세요.

### kubeadm, kubelet, kubectl 설치

```bash
# 패키지 목록 업데이트
sudo apt update

# 쿠버네티스 컴포넌트 설치
sudo apt install -y kubeadm kubelet kubectl

# 자동 업데이트 방지 (의도치 않은 업그레이드 방지)
sudo apt-mark hold kubeadm kubelet kubectl
```

---

## kubeadm 구성 파일을 활용한 클러스터 초기화 준비

구성 파일을 사용하면 재현성이 보장되고 설정 옵션을 체계적으로 관리할 수 있습니다.

### kubeadm 구성 파일 생성

`kubeadm-config.yaml` 파일을 생성합니다 (Calico CNI를 사용하는 경우):

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
  kubeletExtraArgs:
    cloud-provider: ""
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: "v1.30.0"
networking:
  podSubnet: "192.168.0.0/16"   # Calico 기본 CIDR
  serviceSubnet: "10.96.0.0/12"
controllerManager:
  extraArgs:
    bind-address: "0.0.0.0"
scheduler:
  extraArgs:
    bind-address: "0.0.0.0"
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
```

**중요**: 사용하는 CNI 플러그인에 따라 `podSubnet`을 적절히 조정해야 합니다:
- **Calico**: 일반적으로 `192.168.0.0/16`
- **Flannel**: 일반적으로 `10.244.0.0/16`

---

## 제어 평면 노드 초기화

### kubeadm init 실행

구성 파일을 사용하여 제어 평면 노드를 초기화합니다:

```bash
sudo kubeadm init --config kubeadm-config.yaml
```

성공적으로 실행되면 다음과 같은 정보가 출력됩니다:
1. 작업자 노드 조인을 위한 `kubeadm join` 명령어
2. kubectl 설정 방법

### kubectl 구성

제어 평면 노드에서 kubectl을 사용할 수 있도록 구성합니다:

```bash
# kubectl 구성 디렉토리 생성
mkdir -p $HOME/.kube

# 관리자 구성 파일 복사
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# 소유권 설정
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 노드 상태 확인
kubectl get nodes
```

이 시점에서 제어 평면 노드는 `NotReady` 상태로 표시됩니다. CNI 플러그인 설치가 필요합니다.

---

## 컨테이너 네트워크 인터페이스(CNI) 설치

### Calico CNI 설치 (권장)

Calico는 네트워크 정책을 지원하는 인기 있는 CNI 플러그인입니다:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

### Flannel CNI 설치 (대안)

Flannel은 단순한 오버레이 네트워크를 제공하는 경량 CNI 플러그인입니다:

```bash
# Flannel은 일반적으로 10.244.0.0/16 CIDR을 사용합니다
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

**참고**: CNI 플러그인 설치 후 몇 분 이내에 모든 노드가 `Ready` 상태로 전환되어야 합니다. 그렇지 않은 경우 네트워크 구성을 확인해야 합니다.

---

## 작업자 노드 클러스터 조인

### 조인 명령어 실행

각 작업자 노드에서 제어 평면 초기화 시 출력된 `kubeadm join` 명령어를 실행합니다:

```bash
sudo kubeadm join 192.168.56.101:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

### 클러스터 상태 확인

제어 평면 노드에서 모든 노드의 상태를 확인합니다:

```bash
kubectl get nodes -o wide
```

모든 노드가 `Ready` 상태이면 클러스터 구성이 성공적으로 완료된 것입니다.

---

## 기본 기능 검증

### 간단한 애플리케이션 배포

Nginx 웹 서버를 배포하여 클러스터가 정상적으로 작동하는지 확인합니다:

```bash
# Nginx 디플로이먼트 생성
kubectl create deployment nginx --image=nginx:1.27-alpine

# NodePort 서비스로 노출
kubectl expose deployment nginx --type=NodePort --port=80

# 서비스 정보 확인
kubectl get svc nginx
```

### 애플리케이션 접속 테스트

할당된 NodePort를 사용하여 애플리케이션에 접속합니다:

```bash
# NodePort 확인
NODE_PORT=$(kubectl get svc nginx -o jsonpath='{.spec.ports[0].nodePort}')

# 작업자 노드 IP로 접속 테스트
curl http://<작업자-노드-IP>:${NODE_PORT}
```

---

## 운영자를 위한 필수 관리 팁

### kubelet 서비스 관리

```bash
# kubelet 서비스 상태 확인
systemctl status kubelet

# 부팅 시 자동 시작 활성화
sudo systemctl enable kubelet
```

### 컨테이너 런타임 도구 활용

```bash
# 컨테이너 런타임 인터페이스 도구 설치
sudo apt install -y cri-tools

# 실행 중인 컨테이너 확인
sudo crictl ps

# 로컬 이미지 목록 확인
sudo crictl images

# 컨테이너 로그 확인
sudo crictl logs <컨테이너-ID>
```

### 토큰 및 인증서 해시 재생성

작업자 노드를 추가할 때 필요한 토큰과 CA 인증서 해시를 재생성하는 방법:

```bash
# 새 조인 토큰 생성
sudo kubeadm token create

# CA 인증서 해시 계산
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt \
  | openssl rsa -pubin -outform der 2>/dev/null \
  | openssl dgst -sha256 -hex | sed 's/^.* //'
```

### 클러스터 업그레이드 절차

단일 제어 평면 클러스터의 업그레이드 기본 절차:

```bash
# 1. 원하는 버전의 APT 저장소로 변경
# 2. kubeadm 업그레이드
sudo apt update
sudo apt install -y kubeadm=1.30.x-00
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.30.x

# 3. kubelet 및 kubectl 업그레이드
sudo apt install -y kubelet=1.30.x-00 kubectl=1.30.x-00
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### 클러스터 리셋 (테스트 환경 정리)

실습 환경을 정리하거나 클러스터를 처음부터 다시 구축해야 할 때:

```bash
# 클러스터 리셋
sudo kubeadm reset -f

# 컨테이너 런타임 재시작
sudo systemctl restart containerd

# kubectl 구성 제거
sudo rm -rf $HOME/.kube

# 선택적: iptables 및 네트워크 브리지 규칙 정리
sudo iptables -F && sudo iptables -t nat -F
sudo ip link delete cni0 2>/dev/null || true
sudo ip link delete flannel.1 2>/dev/null || true
```

---

## 선언적 구성 파일을 활용한 작업자 노드 조인

구성 파일을 사용하면 조인 과정도 선언적으로 관리할 수 있습니다:

### 작업자 노드 조인 구성 파일 생성

`worker-join.yaml` 파일을 생성합니다:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
discovery:
  bootstrapToken:
    token: "<TOKEN>"
    apiServerEndpoint: "192.168.56.101:6443"
    caCertHashes:
    - "sha256:<HASH>"
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
```

### 구성 파일을 사용한 조인

```bash
sudo kubeadm join --config worker-join.yaml
```

---

## 선택적 추가 구성 요소 설치

### NGINX Ingress Controller

L7 로드 밸런싱과 인그레스 기능을 제공합니다:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
```

### Metrics Server

Horizontal Pod Autoscaler(HPA)와 `kubectl top` 명령어에 필요한 메트릭을 수집합니다:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Horizontal Pod Autoscaler 예제

Metrics Server 설치 후 자동 스케일링을 구성할 수 있습니다:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

---

## 일반적인 문제 해결 가이드

| 증상/오류 메시지 | 가능한 원인 | 해결 방법 |
|---|---|---|
| kubelet 시작 실패, 스왑 관련 오류 | 스왑 메모리 활성화 상태 | `swapoff -a` 실행, `/etc/fstab`에서 스왑 항목 주석 처리 |
| 작업자 노드 조인 타임아웃 | 방화벽 차단, 포트 미개방 | 6443/TCP 포트 열기, 노드 간 네트워크 연결 확인 |
| 노드 `NotReady` 상태 지속 | CNI 미설치 또는 CIDR 불일치 | CNI 설치 확인, `podSubnet`과 CNI CIDR 일치 여부 확인 |
| cgroup 드라이버 불일치 경고 | containerd와 kubelet 드라이버 불일치 | containerd 설정에서 `SystemdCgroup=true` 확인, kubelet 재시작 |
| CoreDNS 파드 CrashLoop | 네트워크/DNS 전달 문제 | CNI 구성 확인, `/etc/resolv.conf` 점검, 노드 IP 충돌 확인 |
| 이미지 풀 실패 | 프록시 또는 사설 레지스트리 인증 문제 | `imagePullSecrets` 구성, 프록시 환경변수 설정 확인 |
| kubeadm init 재실행 실패 | 기존 클러스터 메타데이터 충돌 | `kubeadm reset -f`로 정리 후 재시도 |

### 기본 진단 명령어

문제 발생 시 다음 명령어로 시스템 상태를 확인하세요:

```bash
# 클러스터 전체 상태 확인
kubectl get nodes -o wide
kubectl get pods -A -o wide

# 이벤트 로그 확인 (시간순 정렬)
kubectl get events -A --sort-by=.lastTimestamp

# 상세 정보 확인
kubectl describe node <노드이름>
kubectl describe pod <파드이름> -n <네임스페이스>

# 컴포넌트 로그 확인
sudo journalctl -u kubelet -f
sudo journalctl -u containerd -f
```

---

## 네트워킹 및 성능 개념 이해

### 파드 수용량 추정 기초

클러스터의 파드 수용량을 추정하는 간단한 공식:

$$
\text{필요 파드 수} \approx
\left\lceil \frac{\text{QPS} \cdot t}{\text{파드당 처리량}} \right\rceil \cdot \text{버퍼 계수}
$$

여기서:
- **QPS**: 초당 쿼리 수
- **t**: 평균 요청 처리 시간(초)
- **버퍼 계수**: 트래픽 급증을 흡수하기 위한 여유 (일반적으로 1.2~1.5)

### 노드 자원 계획

노드의 효율적인 자원 활용을 위한 기본 원칙:
- CPU/메모리 `requests`의 합이 노드 용량의 목표 사용률(일반적으로 60~70%)을 초과하지 않도록 계획
- `limits`는 실제 물리적 한계를 초과하지 않도록 설정
- 시스템 리소스(약 10%)를 운영 체제와 쿠버네티스 시스템 컴포넌트용으로 예약

---

## 실전 예제: 완전한 애플리케이션 스택 배포

다음은 간단한 API 서버와 인그레스를 포함한 완전한 애플리케이션 배포 예제입니다:

### 애플리케이션 매니페스트 파일

`echo-api.yaml` 파일을 생성합니다:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-api
  labels:
    app: echo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo
        args: ["-text=hello from kubeadm cluster"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: echo-svc
spec:
  selector:
    app: echo
  ports:
  - port: 80
    targetPort: 5678
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ing
spec:
  ingressClassName: nginx
  rules:
  - host: echo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: echo-svc
            port:
              number: 80
```

### 애플리케이션 배포 및 테스트

```bash
# 애플리케이션 배포
kubectl apply -f echo-api.yaml

# 인그레스 외부 IP 확인
kubectl get ingress echo-ing

# 로컬 호스트 파일에 도메인 매핑 추가
echo "<인그레스-IP> echo.local" | sudo tee -a /etc/hosts

# 애플리케이션 접속 테스트
curl http://echo.local/
```

---

## 결론: kubeadm을 통한 효과적인 쿠버네티스 클러스터 운영

kubeadm을 사용한 쿠버네티스 클러스터 구축은 단순히 클러스터를 실행하는 방법을 배우는 것 이상의 가치가 있습니다. 이 과정을 통해 쿠버네티스의 내부 작동 방식을 깊이 이해하고, 문제 해결 능력을 기르며, 프로덕션 환경에서의 운영 원칙을 체득할 수 있습니다.

### 핵심 성공 원칙:

1. **재현성 보장**: 구성 파일(`kubeadm-config.yaml`)을 사용하여 모든 설정을 버전 관리하고, 동일한 환경을 재현할 수 있어야 합니다. 이는 개발, 스테이징, 프로덕션 환경 간의 일관성을 보장합니다.

2. **표준화된 구성**: 모든 노드에서 동일한 컨테이너 런타임(containerd), 동일한 cgroup 드라이버(Systemd), 일관된 CNI 플러그인을 사용해야 합니다. 불일치는 미묘한 문제의 원인이 될 수 있습니다.

3. **체계적인 운영 절차**: 토큰 관리, 인증서 로테이션, 업그레이드 프로세스, 백업 및 복구 계획을 문서화하고 정기적으로 실행해야 합니다.

4. **관측성 기반 운영**: 클러스터가 운영되기 시작하면 메트릭 수집(Prometheus), 로깅(EFK/ELK 스택), 분산 추적 시스템을 구축하여 시스템 상태를 실시간으로 모니터링해야 합니다.

5. **점진적 개선 접근**: 처음부터 완벽한 구성을 목표로 하기보다 기본적인 작동 환경을 먼저 구축한 후, 보안 강화, 고가용성 구성, 성능 최적화를 단계적으로 추가하는 것이 효과적입니다.

### 프로덕션 환경으로의 발전:

이 가이드에서 구성한 단일 제어 평면 클러스터는 학습과 개발에 적합합니다. 프로덕션 환경으로 전환할 때는 다음 요소를 고려해야 합니다:

- **고가용성**: 다중 제어 평면 노드, etcd 클러스터, 외부 로드 밸런서 구성
- **보안 강화**: RBAC 정교화, Pod Security Admission, 네트워크 정책, etcd 암호화
- **재해 복구**: 정기적인 etcd 스냅샷, 리소스 매니페스트 버전 관리, 크로스 리전 백업
- **자동화**: 클러스터 생성, 업그레이드, 확장을 자동화하는 CI/CD 파이프라인 구축

kubeadm은 쿠버네티스 생태계의 중요한 진입점으로, 이를 통해 획득한 지식은 관리형 쿠버네티스 서비스(EKS, GKE, AKS)를 운영할 때도 매우 유용합니다. 클러스터의 내부 작동 방식을 이해하면 문제 해결, 비용 최적화, 성능 튜닝에 있어 큰 차이를 만들 수 있습니다.

이 가이드가 제공하는 단계별 접근 방식을 따라가면, 처음 쿠버네티스 클러스터를 구축하는 것에서 시작하여 점차적으로 프로덕션 수준의 운영 역량을 갖출 수 있을 것입니다. 각 단계에서 배운 내용을 바탕으로 조직의 특정 요구사항에 맞게 구성을 조정하고 발전시켜 나가세요.

기술은 빠르게 진화하지만, 재현성, 일관성, 관측성, 보안이라는 기본 원칙은 변하지 않습니다. 이러한 원칙에 충실한 클러스터 운영은 장기적인 성공의 기반이 될 것입니다.