---
layout: post
title: Kubernetes - kubeadm으로 클러스터 구성하기
date: 2025-03-31 23:20:23 +0900
category: Kubernetes
---
# kubeadm으로 실제 서버에 쿠버네티스 클러스터 구성하기

## 들어가며 — 기존 글의 핵심을 뽑아 확장하기
기존 글은 **사전 준비 → kubeadm init → 네트워크 플러그인 → 워커 조인** 흐름이 잘 잡혀 있다. 이 글에서는 여기에 **시스템 튜닝(커널/방화벽/cgroup)**, **containerd 표준 설정**, **공식 APT 서명 방식(apt-key 미사용)**, **kubeadm 구성 파일 방식(init/join)**, **운영/업그레이드/리셋/토큰 재발급**, **자주 겪는 문제 해결**을 보강한다.

---

## 1. 목표 아키텍처

- **Control Plane(마스터) 1대**: `kube-apiserver`, `etcd`, `kube-controller-manager`, `kube-scheduler`
- **Worker N대**: 애플리케이션 Pod 실행

```
[ control-plane (192.168.56.101) ]
        ↑ 6443/TCP (apiserver)
        │
        ├── Worker-1 (192.168.56.102)
        └── Worker-2 (192.168.56.103)
```

실습은 단일 Control Plane로 시작하되, 운영 이행 시 **HA(kube-vip/외부 LB + 다중 etcd)** 를 고려한다.

---

## 2. 모든 노드 공통 — 시스템 준비

### 2.1 호스트네임/hosts
```bash
# 각 노드에서 (예시)
sudo hostnamectl set-hostname control-plane   # 또는 worker-1, worker-2 ...
echo "192.168.56.101 control-plane" | sudo tee -a /etc/hosts
echo "192.168.56.102 worker-1"       | sudo tee -a /etc/hosts
echo "192.168.56.103 worker-2"       | sudo tee -a /etc/hosts
```

### 2.2 스왑 비활성 (kubelet 요구 사항)
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 2.3 커널 모듈/네트 필터
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sudo sysctl --system
```

### 2.4 방화벽(테스트 편의)
실무에선 포트 단일 허용을 권장한다. 학습 목적이라면 임시 비활성:
```bash
sudo ufw disable
```
**주요 포트**: 6443(apiserver), 10250(kubelet), 2379-2380(etcd CP 내부), 10257/10259(컨트롤러/스케줄러), NodePort(30000-32767), CNI 포트 등.

---

## 3. 컨테이너 런타임 — containerd 권장 설정

### 3.1 containerd 설치
```bash
sudo apt update
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

### 3.2 SystemdCgroup 활성화 (kubelet과 cgroup 드라이버 일치)
`/etc/containerd/config.toml` 에서 `SystemdCgroup = true` 로 변경:
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl enable --now containerd
```

---

## 4. kubeadm / kubelet / kubectl 설치 (공식 APT 서명 방식)

### 4.1 패키지 저장소 키/리스트
```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```
> 버전 트랙은 환경에 맞춰 `v1.29`, `v1.31` 등으로 조정.

### 4.2 설치 및 홀드
```bash
sudo apt update
sudo apt install -y kubeadm kubelet kubectl
sudo apt-mark hold kubeadm kubelet kubectl
```

---

## 5. kubeadm 구성 파일로 초기화 준비(권장)

**구성 파일을 써야 재현성과 옵션 관리가 쉽다.**

`kubeadm-config.yaml` (Calico를 가정해 Pod CIDR = 192.168.0.0/16 예시):
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
  podSubnet: "192.168.0.0/16"   # Calico 기본과 맞춤
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

> **Flannel**을 쓸 경우 일반적으로 `10.244.0.0/16`. CNI가 요구하는 `podSubnet`과 반드시 일치시킨다.

---

## 6. Control Plane 초기화

### 6.1 init
```bash
sudo kubeadm init --config kubeadm-config.yaml
```
성공 시 `kubeadm join ...` 명령이 출력된다(워커에서 사용).

### 6.2 kubectl 사용 준비 (Control Plane 노드)
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
```

---

## 7. 네트워크 플러그인(CNI) 설치

### 7.1 Calico (예)
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

### 7.2 Flannel (대안 — Pod CIDR 10.244.0.0/16 필요)
```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

> CNI 설치 전에는 노드가 `NotReady`. 설치 후 수십 초 내 `Ready`로 전환되어야 정상.

---

## 8. 워커 노드 조인

### 8.1 join 실행 (각 워커 노드)
```bash
sudo kubeadm join 192.168.56.101:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

### 8.2 Control Plane에서 확인
```bash
kubectl get nodes -o wide
```
모든 노드가 `Ready` 이면 성공.

---

## 9. 간단 기능 검증

### 9.1 Nginx 배포/노출
```bash
kubectl create deployment nginx --image=nginx:1.27-alpine
kubectl expose deployment nginx --type=NodePort --port=80
kubectl get svc
```

### 9.2 서비스 접속
```bash
# 노드 IP + 할당된 NodePort 로 접속
curl http://<worker-node-ip>:<nodeport>
```

---

## 10. 운영자를 위한 필수 팁

### 10.1 kubelet 서비스/부팅 연동
```bash
systemctl status kubelet
sudo systemctl enable --now kubelet
```

### 10.2 CRI 도구(crictl) 유용 명령
```bash
sudo apt install -y cri-tools
sudo crictl ps
sudo crictl images
sudo crictl logs <container-id>
```

### 10.3 토큰/해시 재발급(워커 추가 시)
```bash
# 새 토큰 생성
sudo kubeadm token create

# CA cert hash 출력
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt \
  | openssl rsa -pubin -outform der 2>/dev/null \
  | openssl dgst -sha256 -hex | sed 's/^.* //'
```

### 10.4 업그레이드 개요(단일 CP)
```bash
# 원하는 버전 채널로 apt 소스 교체 후
sudo apt update
sudo apt install -y kubeadm=1.30.x-00
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.30.x

# 각 노드에서 kubelet/kubectl
sudo apt install -y kubelet=1.30.x-00 kubectl=1.30.x-00
sudo systemctl restart kubelet
```

### 10.5 리셋(연습 환경 정리)
```bash
sudo kubeadm reset -f
sudo systemctl restart containerd
sudo rm -rf $HOME/.kube
# (필요 시 iptables/브리지 규칙 정리)
```

---

## 11. 선언형으로 더 엄격하게 — JoinConfiguration 사용

```yaml
# worker-join.yaml
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

```bash
sudo kubeadm join --config worker-join.yaml
```

---

## 12. 선택: Ingress/Metric·HPA 빠른 구성

### 12.1 NGINX Ingress Controller(매니페스트 예시)
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
```

### 12.2 Metrics Server
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### 12.3 HPA 스니펫
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: nginx-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: nginx }
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 60 }
```

---

## 13. 자주 겪는 문제 해결(확장판)

| 증상/오류 | 원인 후보 | 해결 방법 |
|---|---|---|
| `kubelet` 시작 실패, 스왑 관련 로그 | 스왑 활성 | `swapoff -a`, `/etc/fstab` 주석 처리 |
| 워커 `join` 시 타임아웃 | 방화벽/포트 차단 | 6443/TCP 열기, 노드 간 통신 확인 |
| 노드 `NotReady` 지속 | CNI 미설치/Pod CIDR 불일치 | CNI 설치, `podSubnet`과 CNI CIDR 일치 확인 |
| `cgroup driver` 불일치 경고 | containerd와 kubelet 드라이버 상이 | containerd `SystemdCgroup=true`, kubelet 재시작 |
| CoreDNS CrashLoop | 네트워크/DNS 포워딩 문제 | CNI 재확인, `/etc/resolv.conf` 점검, node IP 충돌 여부 |
| 이미지 풀 실패 | 프록시/사설 레지스트리 인증 미설정 | `imagePullSecrets`, 프록시 환경변수/NO_PROXY 설정 |
| `kubeadm init` 재실행 실수 | 클러스터 메타/증분 충돌 | `kubeadm reset -f`로 정리 후 재시도 |

**기본 점검 명령**
```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe node <node>
kubectl describe pod <pod> -n <ns>
```

---

## 14. 네트워킹/성능/용량 개념식(학습용)

Pod 수용량 추정의 단순식:
$$
\text{필요 파드 수} \approx
\left\lceil \frac{\text{QPS}\cdot t}{\text{파드당 처리량}} \right\rceil \cdot \text{버퍼}
$$
여기서 \(t\)는 평균 처리시간(초), 버퍼는 스파이크 흡수(예: 1.2~1.5).
노드 패킹은 CPU/메모리 `requests` 합이 노드 용량의 목표 활용률 \(U\)(예: 0.6~0.7)를 넘지 않도록 한다.

---

## 15. 실전 예제 — 간단 API + Ingress 배포

```yaml
# api-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: echo-api, labels: { app: echo } }
spec:
  replicas: 2
  selector: { matchLabels: { app: echo } }
  template:
    metadata: { labels: { app: echo } }
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo
        args: ["-text=hello from kubeadm"]
        ports: [{ containerPort: 5678 }]
---
apiVersion: v1
kind: Service
metadata: { name: echo-svc }
spec:
  selector: { app: echo }
  ports: [{ port: 80, targetPort: 5678 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: echo-ing }
spec:
  ingressClassName: nginx
  rules:
  - host: echo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: echo-svc, port: { number: 80 } } }
```

적용/접속:
```bash
kubectl apply -f api-deploy.yaml
# /etc/hosts에 control-plane or ingress LB IP 매핑
echo "<INGRESS_IP> echo.local" | sudo tee -a /etc/hosts
curl http://echo.local/
```

---

## 16. 마무리 — 현업 이행 체크리스트
- **재현성**: kubeadm 구성 파일로 init/join, Git에 버전 관리
- **표준화**: containerd(SystemdCgroup), CNI 선택/Pod CIDR, kube-proxy 모드
- **보안**: RBAC 최소 권한, etcd 암호화(EncryptionConfiguration), 감사 로깅
- **가용성**: 다중 Control Plane + 외부 LB, etcd 3/5노드, 백업/DR 시나리오
- **관측/운영**: metrics-server/Prometheus/Grafana, 로깅 스택, 경보 기준선
- **수명주기**: 계획된 업그레이드 절차/리허설, 토큰/인증서 로테이션

> kubeadm은 “클러스터를 **어떻게 올라가는지**”를 몸으로 익히는 가장 좋은 경로다.
> 이 경험은 매니지드 K8s(EKS/GKE/AKS) 운영 시에도 장애 대응과 비용/성능 최적화에 큰 차이를 만든다.
