---
layout: post
title: Kubernetes - K8s 이후의 기술 트렌드
date: 2025-06-13 21:20:23 +0900
category: Kubernetes
---
# Kubernetes 이후의 기술 트렌드: K3s, MicroK8s, Edge Kubernetes

표준 Kubernetes는 대규모 클라우드 네이티브 환경에서 확실한 솔루션으로 자리잡았지만, 저사양 디바이스, 엣지 컴퓨팅, 로컬 개발 환경, 소규모 서버에서는 복잡성과 리소스 소비가 주요 장애물로 작용합니다. 이러한 환경을 위해 등장한 경량화된 Kubernetes 배포판들은 새로운 가능성을 제시하고 있습니다.

---

## 환경별 Kubernetes 배포판 선택 가이드

| 사용 환경 | 권장 솔루션 | 선택 근거 | 빠른 시작 명령어 |
|---|---|---|---|
| 라즈베리파이·저사양 싱글/소규모 클러스터 | **K3s** | 단일 바이너리·경량·ARM 최적화 | `curl -sfL https://get.k3s.io | sh -` |
| 우분투 로컬 개발·AI/ML 개념 검증 | **MicroK8s** | Snap 설치·통합 애드온·GPU 지원 | `sudo snap install microk8s --classic` |
| 대규모 엣지 노드, 간헐적 네트워크 환경 | **KubeEdge/OpenYurt** | 클라우드-엣지 분산 제어/데이터 처리 | (아래 Edge 섹션 참조) |
| 오프라인/에어갭 환경 | **K3s** | 이미지 번들링·내장 런타임 | k3s 에어갭 설치 |
| 통합 개발 데스크톱 환경 | **MicroK8s** | `enable` 명령으로 Ingress/레지스트리/관측성 활성화 | `microk8s enable ...` |

---

## 경량 Kubernetes의 필요성

### 리소스 효율성과 운영 간소화의 중요성

- **컨트롤 플레인 구성 요소**: API 서버, 컨트롤러, 스케줄러, etcd, CNI, CSI, Ingress 등이 지속적으로 RAM과 CPU를 소비합니다.
- **설치 및 구성 복잡성**: kubeadm 기반 설치에는 다양한 구성 요소, 인증서, CNI, 스토리지 설정이 필요하며 학습 곡선이 가파릅니다.
- **엣지 컴퓨팅 환경 특수성**: 간헐적 연결, 낮은 사양, 물리적 제약으로 인해 로컬 자율 운영과 중앙 정책 동기화가 모두 필요합니다.

**리소스 할당 개념적 공식**:
```
사용 가능한 노드 메모리 ≧ 운영체제 + K8s 오버헤드 + Σ(파드 요청 리소스)
```

경량 배포판은 **K8s 오버헤드**를 최소화하여 **애플리케이션 실행에 사용 가능한 용량**을 극대화합니다.

---

## K3s: Rancher가 개발한 초경량 CNCF 호환 Kubernetes

### 주요 특징

- **단일 바이너리**: `k3s` 하나에 핵심 구성 요소가 모두 포함되어 있으며, ARM(aarch64) 아키텍처에 최적화되었습니다.
- **경량 런타임**: 기본적으로 containerd를 사용하며, SQLite(기본) 또는 etcd(선택)를 백엔드 저장소로 지원합니다.
- **통합 구성 요소**: Flannel CNI, Traefik Ingress, 유틸리티가 기본 포함되어 있으며, 자동 인증서 순환 기능을 제공합니다.
- **유연한 운영**: systemd에 의존하지 않고 동작 가능하며, 에어갭 환경 설치를 지원합니다.

### 단일 노드 설치 (가장 빠른 시작 방법)

```bash
curl -sfL https://get.k3s.io | sh -
sudo kubectl get nodes -o wide
# kubeconfig 위치: /etc/rancher/k3s/k3s.yaml (root 소유)
```

### 멀티 노드 클러스터 구성 (서버 + 에이전트)

**서버 노드 설치**
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --cluster-init" sh -
sudo cat /var/lib/rancher/k3s/server/node-token
```

**에이전트 노드 조인**
```bash
export K3S_URL=https://<서버_IP>:6443
export K3S_TOKEN=<서버_노드_토큰>
curl -sfL https://get.k3s.io | K3S_URL=$K3S_URL K3S_TOKEN=$K3S_TOKEN sh -
```

### 고급 구성 옵션

- 외부 etcd 3노드 구성: `--datastore-endpoint="https://etcd1:2379,..."`
- 사용자 정의 인증서: `--datastore-cafile`, `--datastore-certfile`, `--datastore-keyfile` 옵션 사용

### K3s 프로덕션 환경 강화 체크리스트

1. **트래픽 관리**: `--disable traefik` 옵션으로 기본 Ingress 비활성화 후 Nginx Ingress로 교체
2. **네트워킹 강화**: Cilium CNI(eBPF 기반)로 교체하여 네트워킹 및 네트워크 정책 기능 향상
3. **보안 강화**: `--kube-apiserver-arg`를 통한 인증, 감사 로깅, RBAC 세분화 설정
4. **데이터 저장소**: etcd 외부화 및 정기적인 스냅샷/백업 전략 수립
5. **저장소 모니터링**: `/var/lib/rancher/k3s/agent/containerd/` 디렉토리 디스크 사용량 모니터링
6. **파드 보안**: PodSecurity Admission `restricted` 네임스페이스 라벨 적용

---

## MicroK8s: Canonical의 개발자 친화적 경량 Kubernetes

### 5분 안에 설치부터 애드온까지

```bash
sudo snap install microk8s --classic
microk8s status --wait-ready
microk8s enable dns ingress dashboard registry
microk8s kubectl get all -A
```

**추가 애드온 활성화**:
```bash
microk8s enable prometheus jaeger gpu hostpath-storage
```

**kubeconfig 추출**:
```bash
microk8s config
```

### 멀티 노드 클러스터 구성

```bash
# 새 노드 추가 토큰 생성
microk8s add-node

# 워커 노드에서 클러스터 조인
microk8s join <IP>:<PORT>/<TOKEN>
```

### 개발자 생산성 기능

- **내장 컨테이너 레지스트리**: 로컬 이미지 빌드 및 배포 지원
- **Kubeflow 통합**: `microk8s enable kubeflow` 명령으로 머신러닝 플랫폼 설치
- **GPU 가속 지원**: `microk8s enable gpu`로 NVIDIA GPU 드라이버 연동

### 운영 시 고려사항

- Snap 채널 시스템을 통한 안정적인 버전 관리 및 롤백 기능
- 통합 관측성(Observability) 애드온으로 Prometheus/Grafana 즉시 활용
- Snap 업데이트 정책과 호스트 커널 의존성 확인 필요

---

## 엣지 Kubernetes: 클라우드-엣지 하이브리드 아키텍처

### 엣지 컴퓨팅 환경의 특수 요구사항

- **네트워크 불안정성**: 오프라인 상태에서도 로컬 자율 운영 가능, 연결 복구 시 상태 동기화 필요
- **하드웨어 제약**: 저사양 하드웨어에 적합한 경량 런타임, 네트워킹, 스토리지 솔루션
- **보안 강화**: mTLS, 제로 트러스트, 디바이스 신원 관리(SPIFFE/SPIRE)
- **데이터 처리 전략**: 비디오/센서 데이터 등 대용량 데이터의 로컬 처리 및 요약된 결과만 업로드

### 주요 엣지 Kubernetes 플랫폼 비교

| 플랫폼 | 포지셔닝 | 강점 | 고려사항 |
|---|---|---|---|
| **KubeEdge** | 엣지 디바이스-Kubernetes 클라우드 통합 | MQTT/EdgeCore/디바이스 SDK, 오프라인 운영 | 초기 구성 복잡성 |
| **OpenYurt** | 클라우드 네이티브 엣지 확장 | YurtHub를 통한 엣지 캐싱, YurtAppSet | 알리바바 생태계 친화적 |
| **SuperEdge** | 대규모 엣지 노드 관리 | 다중 클러스터/하이브리드 제어 | 운영 도구 학습 필요 |

### KubeEdge 아키텍처 개요

```
클라우드 측:
  [Kubernetes Control Plane] ↔ [CloudCore/CloudHub]

엣지 측:
  [EdgeCore] → [App Pods]
            → [MQTT/DeviceTwin]
```

- **CloudCore**: Kubernetes와 엣지 간 메시지 브리지 역할
- **EdgeCore**: 엣지에서 파드 관리 및 장치 제어(오프라인 상태에서도 지속 운영)
- **DeviceTwin**: 물리적 장치 상태의 디지털 복제본 관리

### KubeEdge 설치 개요

1. **클라우드 측**: Kubernetes 클러스터에 CloudCore 설치 (Helm 차트 제공)
2. **엣지 측**: 바이너리 또는 컨테이너로 EdgeCore 설치 후 CloudCore에 등록
3. **보안 구성**: mTLS 인증서, 터널링 포트(웹소켓), 방화벽 규칙 설정

---

## 네트워킹, 스토리지, 보안, 관측성: 경량/엣지 특화 설계

### 네트워킹 (CNI)

- **K3s 기본**: Flannel - 단순하고 경량
- **고성능/정책 필요 시**: Cilium(eBPF) 또는 Calico로 교체
- **엣지 환경 특수성**: NAT/방화벽, 이중 NIC, 셀룰러 네트워크 고려, L4 로드 밸런서(MetalLB) 활용

**K3s에 Cilium 적용 예시**:
```bash
INSTALL_K3S_EXEC="--flannel-backend=none --disable-network-policy" sh -
# 이후 Cilium Helm 차트 설치
```

### 스토리지 (CSI)

- **로컬/단일 노드**: hostPath, OpenEBS LocalPV
- **엣지 분산 스토리지**: Longhorn (경량, UI 제공, 복구 용이)
- **오브젝트 스토리지**: MinIO를 통한 데이터 수집/버퍼링 후 업링크

**Longhorn 설치**:
```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml
```

### 보안 구성

- **PodSecurity Admission**: 네임스페이스 수준에서 제한적 보안 정책 적용
```bash
kubectl label ns prod pod-security.kubernetes.io/enforce=restricted
```
- **mTLS 구현**: Ingress부터 서비스까지 종단 간 암호화 (서비스 메시 또는 cert-manager + SPIFFE)
- **엣지 장치 인증**: SPIFFE/SPIRE를 통한 워크로드 신원 관리
- **런타임 보안**: Falco 런타임 공격 탐지, Trivy 컨테이너 이미지 취약점 스캔

### 관측성 (Observability)

- **MicroK8s**: `microk8s enable prometheus` 명령으로 간편 설치
- **K3s**: kube-prometheus-stack Helm 차트, Loki+Promtail 로깅, Tempo/Jaeger 분산 추적
- **엣지 특화 전략**: 로컬 저장 후 요약된 데이터만 업링크 (데이터 비용 및 불안정 연결 대비)

---

## GitOps 및 CI/CD: 경량 및 엣지 환경 적용

### ArgoCD를 통한 K3s 동기화

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: edge-web
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  source:
    repoURL: https://github.com/your/repo
    path: k8s/overlays/edge
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### GitHub Actions를 통한 이미지 빌드 및 K3s 배포

{% raw %}
```yaml
name: build-and-deploy
on: 
  push:
    branches: [main]
jobs:
  cd:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USER }}
        password: ${{ secrets.DOCKER_PASS }}
    - run: |
        docker build -t ${{ secrets.DOCKER_USER }}/edge-app:${{ github.sha }} .
        docker push ${{ secrets.DOCKER_USER }}/edge-app:${{ github.sha }}
    - run: |
        echo "${{ secrets.KUBECONFIG_B64 }}" | base64 -d > kubeconfig
        KUBECONFIG=$PWD/kubeconfig kubectl -n default set image deploy/edge-app app=${{ secrets.DOCKER_USER }}/edge-app:${{ github.sha }}
        KUBECONFIG=$PWD/kubeconfig kubectl rollout status deploy/edge-app -n default
```
{% endraw %}

---

## 실전 구성 예제

### K3s + Nginx Ingress + Longhorn + HPA 구성

**디플로이먼트 및 서비스**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: app
        image: ghcr.io/you/web:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "150m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

**인그레스 구성**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - "web.edge.local"
    secretName: web-tls
  rules:
  - host: web.edge.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

**수평 파드 오토스케일러**:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

**영구 볼륨 클레임 (Longhorn StorageClass)**:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-data
spec:
  accessModes:
  - "ReadWriteOnce"
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
```

### MicroK8s에서 Kubeflow 설치

```bash
microk8s enable kubeflow    # 설치에 시간이 소요될 수 있으며, 충분한 디스크 공간 권장
```

### KubeEdge 장치 관리 예시

```yaml
apiVersion: devices.kubeedge.io/v1alpha2
kind: DeviceModel
metadata:
  name: temp-sensor-model
spec:
  properties:
  - name: temperature
    type:
      int:
        accessMode: ReadOnly
---
apiVersion: devices.kubeedge.io/v1alpha2
kind: Device
metadata:
  name: temp-sensor-01
spec:
  deviceModelRef:
    name: temp-sensor-model
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
      - key: node-role
        operator: In
        values: ["edge"]
```

---

## 오프라인/에어갭 환경 운영 전략

- **사전 이미지 패키징**: 컨테이너 레지스트리 미러 또는 tar 아카이브 번들링
- **K3s 에어갭 설치**: `/var/lib/rancher/k3s/agent/images/` 디렉토리에 이미지 tar 배포 후 설치
- **OTA 업데이트 정책**: 카나리/안정 버전 링과 Helm history/Argo Rollout 기반 롤백 전략
- **관측 데이터 관리**: Loki boltdb-shipper와 간헐적 업로드를 통한 데이터 요약 및 버퍼링

---

## 미래 기술 트렌드: eBPF, WASM, AI@Edge

- **eBPF(Cilium)**: 커널 수준의 고성능 네트워킹, 정책 적용, 가시성 확보
- **WASM(WasmEdge/Spin+k8s)**: 경량 함수 실행 환경 (콜드 스타트 감소, 격리성 향상)
- **엣지 AI/ML**: ONNX/TensorRT/TF Lite를 활용한 엣지 추론, 결과만 클라우드 업로드
- **SPIFFE/SPIRE**: 엣지-클라우드 간 워크로드 신원 기반 mTLS (제로 트러스트 네트워크 접근)

---

## 실무 환경별 용량 계획 가이드라인

- **싱글 보드 컴퓨터 (4GB RAM)**: K3s + 1~3개 경량 워크로드
- **로컬 개발 데스크톱 (16GB RAM)**: MicroK8s + Ingress + Prometheus/Grafana + 소규모 ML 워크로드
- **엣지 게이트웨이 (8~32GB RAM)**: K3s + Longhorn + Loki + 1~3개 추론 워크로드

**파드 리소스 계획 개념**:
```
사용 가능한 노드 메모리 ≒ 총 메모리 - (운영체제 + K8s 오버헤드 + 버퍼)
```

버퍼는 페이지 캐시 및 트래픽 급증에 대비하여 10~25% 정도로 설정하는 것이 좋습니다.

---

## 문제 해결 및 운영 모니터링

**초기 설치 검증**:
```bash
kubectl get nodes -o wide
kubectl -n kube-system get pods
```

**네트워크 문제 진단**:
```bash
kubectl -n kube-system logs ds/cilium -c cilium-agent --tail=100
```

**디스크 압력 확인**:
```bash
kubectl describe node <노드_이름> | egrep -i 'Pressure|Allocatable|Capacity'
```

**컨테이너 런타임 상태 확인**:
```bash
sudo journalctl -u containerd -f
sudo crictl ps -a
```

**업그레이드 전 준비사항**:
- Helm 릴리스 백업, etcd/SQLite 스냅샷 생성
- PodDisruptionBudget 완화
- 순차적 또는 블루-그린 업데이트 전략 수립

---

## 보안 운영 핵심 원칙

1. **PodSecurity Admission**: 네임스페이스 수준에서 `restricted` 정책 강제 적용
2. **종단 간 암호화**: Ingress부터 서비스까지 mTLS 구현 (서비스 메시 또는 cert-manager+SPIFFE 조합)
3. **이미지 무결성**: COSIGN 서명 및 Kyverno 정책 기반 검증
4. **런타임 보안**: Falco 런타임 공격 탐지 및 Trivy 취약점 스캔 정기 실행
5. **엣지 장치 관리**: 원격 키 순환, 시간 동기화(chrony/ntp) 구성

---

## 결론

Kubernetes 생태계는 표준 Kubernetes를 넘어 다양한 환경과 요구사항에 맞춘 특화된 배포판들로 확장되고 있습니다:

1. **경량 Kubernetes의 가치**: 리소스 효율성과 운영 간소화를 통해 더 넓은 환경에 Kubernetes를 적용할 수 있습니다.

2. **K3s의 적합 환경**: 단일 바이너리, 에어갭 설치, ARM 지원, 현장 배치 환경에 최적화되어 있습니다.

3. **MicroK8s의 장점**: Snap 패키지 관리와 통합 애드온 시스템으로 개발 및 AI/ML 개념 검증 환경에 적합합니다.

4. **엣지 Kubernetes의 필요성**: KubeEdge와 OpenYurt는 오프라인 자율 운영과 중앙 정책 관리의 균형을 제공합니다.

5. **운영 표준화**: CNI/CSI 선택, PSA/mTLS 적용, 통합 관측성, GitOps 자동화, 체계적인 업그레이드 및 롤백 전략이 필수적입니다.

이러한 다양한 옵션을 이해하고 조직의 특정 요구사항에 맞는 솔루션을 선택하는 것이 성공적인 클라우드 네이티브 전략 수립의 핵심입니다.

---

## 부록: 라즈베리파이 K3s + MetalLB + Nginx 빠른 설치 스크립트

```bash
# K3s 설치
curl -sfL https://get.k3s.io | sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# MetalLB 설치
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

# MetalLB IP 풀 구성
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: pool
  namespace: metallb-system
spec:
  addresses:
  - "192.168.0.240-192.168.0.250"
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2
  namespace: metallb-system
spec: {}
EOF

# Nginx 배포 및 LoadBalancer 서비스 생성
kubectl create deploy web --image=nginx
kubectl expose deploy web --port=80 --type=LoadBalancer
kubectl get svc web -w
```
