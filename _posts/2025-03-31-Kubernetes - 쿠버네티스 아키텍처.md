---
layout: post
title: Kubernetes - 쿠버네티스 아키텍처
date: 2025-03-31 20:20:23 +0900
category: Kubernetes
---
# 쿠버네티스 아키텍처

## 들어가며 — 기존 글의 핵심 요지와 확장 방향

기존 글에서 잘 정리된 대로, 쿠버네티스는 **Control Plane(마스터 역할)** 과 **Worker Node(작업자 역할)** 로 나뉜다. 이번 재정리는 다음을 더 깊이 파고든다: 구성요소 상세 동작, 데이터/트래픽 경로, 운영(HA/업그레이드/DR), 보안/정책, 관측/진단, 실전 예시와 트러블슈팅.

## 상위 아키텍처 개요 (Master vs Node)

- **Control Plane**: *원하는 상태(Desired State)* 저장 및 컨트롤 루프로 수렴.
- **Worker Node**: Pod 실행, 네트워킹/스토리지 연결, 로컬 자원 관리.

```
              [ Control Plane ]
          ┌─────────────────────────┐
          │  kube-apiserver         │
          │  etcd                   │
          │  kube-scheduler         │
          │  kube-controller-manager│
          │  cloud-controller-mgr   │
          └─────────────────────────┘
                     ▲        ▲
                     │        │
          ┌──────────┘        └──────────┐
          ▼                               ▼
   [ Worker Node A ]                [ Worker Node B ]
 ┌──────────────────┐             ┌──────────────────┐
 │ kubelet          │             │ kubelet          │
 │ kube-proxy/CNI   │             │ kube-proxy/CNI   │
 │ containerd/CRI-O │             │ containerd/CRI-O │
 │ Pods             │             │ Pods             │
 └──────────────────┘             └──────────────────┘
```

## Control Plane 심층

### kube-apiserver — 단일 진입점과 선언형 API

- 인증(쿠키/토큰/MTLS/OIDC) → 인가(RBAC) → 어드미션(기본+Webhook) → 유효성 검증 → etcd 저장.
- 리소스 버전과 **Watch** 스트림으로 컨트롤러/클라이언트가 실시간 변경 수신.

```bash
kubectl get --raw /healthz
kubectl api-resources
kubectl explain deployment.spec.strategy
```

### etcd — 강한 일관성 KV 저장소

- 쿠버네티스의 **단일 진실 원천**. 멀티 노드(홀수) Raft 합의.
- 스냅샷/복구가 DR의 핵심.

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-$(date +%F).db
```

### kube-scheduler — 점수화 기반 배치

필터링(자원/테인트/어피니티/토폴로지) → 점수화(가중치) → 최적 노드 선택.

$$
\text{score}(n)=\sum_k w_k \cdot f_k(n,\text{pod},\text{cluster})
$$

**입력 제어 포인트**: `resources.requests`, `nodeSelector`, `affinity/antiAffinity`, `topologySpreadConstraints`, `tolerations`, `pdb` 등.

### kube-controller-manager — 원하는 상태로 수렴

- 수많은 컨트롤러(Deployment/ReplicaSet/Job/Node/EndpointSlice …)가 워치 이벤트를 구독, 차이를 줄이기 위해 **행동** 실행.

### cloud-controller-manager — 퍼블릭 클라우드 통합

- 노드/로드밸런서/볼륨/라우팅 등 클라우드 API 연계를 Control Plane에서 분리.

## Worker Node 심층

### kubelet — 노드의 관리자

- 할당된 PodSpec을 받아 컨테이너 생성/감시, Probe/라이프사이클 훅/termination 유예 준수.

```bash
kubectl describe node <node>
kubectl get pods -o wide -A --field-selector spec.nodeName=<node>
journalctl -u kubelet -e --no-pager
```

### kube-proxy vs eBPF 기반 L4

- kube-proxy(iptables/ipvs): Service → Pod L4 LB.
- Cilium 등 eBPF CNI: kube-proxy 없이 고성능 데이터패스/가시성.

### 컨테이너 런타임 — CRI 추상화

- **containerd/CRI-O** 권장(도커 shim 제거). 이미지 풀/컨테이너 라이프사이클 표준화.

## 리소스 선언과 API 경로 — 실전 YAML

### 최소 웹 앱(Probe/Resource/Service 포함)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels: { app: web }
spec:
  replicas: 3
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
      - name: web
        image: nginx:1.27-alpine
        ports: [{ containerPort: 80 }]
        readinessProbe:
          httpGet: { path: /, port: 80 }
          periodSeconds: 5
        livenessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 10
        resources:
          requests: { cpu: "100m", memory: "128Mi" }
          limits: { cpu: "500m", memory: "256Mi" }
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector: { app: web }
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### ConfigMap/Secret 분리

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config }
data:
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata: { name: app-secret }
type: Opaque
stringData:
  DB_PASSWORD: "s3cr3t"
```

### 배치/스케줄 작업(Job/CronJob)

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: data-migration }
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrator
        image: ghcr.io/example/migrator:1.0.0
        args: ["--run-once"]
```

## 네트워킹 데이터 경로와 정책

### Service/Endpoints/EndpointSlice

- Service(가상 IP) → EndpointSlice(실제 Pod IP 목록) → kube-proxy/eBPF가 DNAT/분산.

### Ingress vs Gateway API

- Ingress: L7 HTTP(S) 라우팅 규칙.
- Gateway API: Listener/Route로 역할 분리·표현력 향상.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  rules:
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: web-svc, port: { number: 80 } } }
```

### NetworkPolicy — 허용 리스트 기반

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-frontend, namespace: prod }
spec:
  podSelector: { matchLabels: { app: api } }
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - namespaceSelector: { matchLabels: { role: frontend } }
    ports: [{ protocol: TCP, port: 8080 }]
  egress:
  - to:
    - ipBlock: { cidr: 10.0.0.0/8 }
    ports: [{ protocol: TCP, port: 5432 }]
```

## 스케줄링 제어 — 어피니티/테인트/토폴로지 분산

```yaml
spec:
  nodeSelector:
    nodepool: general
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels: { app: api }
        topologyKey: "kubernetes.io/hostname"
  tolerations:
  - key: "workload"
    operator: "Equal"
    value: "batch"
    effect: "NoSchedule"
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: "topology.k8s.io/zone"
    whenUnsatisfiable: DoNotSchedule
    labelSelector: { matchLabels: { app: api } }
```

## 자원 모델, QoS, 축출(Eviction)

- **Requests**: 스케줄러 기준(최소 보장), **Limits**: 상한. 메모리 초과 시 OOMKill.
- **QoS**: Guaranteed(요청=한도), Burstable, BestEffort. 노드 압박 시 축출 우선순위: BestEffort → Burstable → Guaranteed.

## 보안 아키텍처 — RBAC/PSA/Admission

### RBAC 최소 권한

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: read-pods, namespace: dev }
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","list","watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata: { name: read-pods-binding, namespace: dev }
subjects:
- kind: User
  name: alice
roleRef:
  kind: Role
  name: read-pods
  apiGroup: rbac.authorization.k8s.io
```

### Pod Security Admission(PSA)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: restricted-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

### Admission Webhook (검증/변형)

OPA Gatekeeper/Kyverno로 이미지 서명/루트 권한 금지/호스트 네트 금지 등 정책 강제.

## — 이벤트/메트릭/로그/트레이스

- **이벤트**: `kubectl get events -A --sort-by=.lastTimestamp`
- **메트릭**: metrics-server/Prometheus(+ kube-state-metrics)/Alertmanager
- **로그**: stdout/stderr → Fluent Bit/Fluentd → Elasticsearch/Opensearch → Kibana
- **트레이스**: OpenTelemetry SDK/Collector → Jaeger/Tempo

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: api-sm }
spec:
  selector: { matchLabels: { app: api } }
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

## Control Plane

### 토폴로지

- **etcd** 3/5개(홀수), 서로 다른 AZ.
- **apiserver** 다중 인스턴스 + 앞단 LB.
- 컨트롤러/스케줄러 다중 인스턴스(리더 선출).

### kubeadm HA 개념

```bash
kubeadm init --control-plane-endpoint "cp-lb:6443" --upload-certs
```

## 업그레이드, 백업/DR, 감사(Audit)

### 업그레이드

- Control Plane → Worker 순. 릴리스 노트/Deprecated API 확인.

### 백업/DR

- etcd 스냅샷 + 리소스 매니페스트(Helm/Kustomize) + 이미지 레지스트리 가용성.
- Velero로 PVC 스냅샷/이관.

### 감사(Audit)

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  verbs: ["create","update","patch","delete"]
```

## 워크로드

- **CSI** 드라이버 표준, **StatefulSet**: 안정 ID/스토리지 바인딩.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: pg }
spec:
  serviceName: "pg"
  replicas: 3
  selector: { matchLabels: { app: pg } }
  template:
    metadata: { labels: { app: pg } }
    spec:
      containers:
      - name: postgres
        image: postgres:16
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata: { name: data }
    spec:
      accessModes: [ReadWriteOnce]
      resources: { requests: { storage: 100Gi } }
```

## 실전 운영 시나리오

### — 드레이닝/언코드론

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# 업데이트 후

kubectl uncordon <node>
```

### 부분 장애 대응 — 패턴별 트러블슈팅

| 증상 | 원인 후보 | 1차 확인 | 해결 가이드 |
|---|---|---|---|
| CrashLoopBackOff | 설정/포트/OOM | `logs -p`, `describe pod` | 설정 복원, 리소스 조정, probe 수정 |
| Pending 지속 | 자원 부족/테인트 | `describe pod` | requests 조정, 노드 증설, tolerations |
| Service 연결 불가 | 셀렉터/포트 mismatch | `get endpoints` | selector/targetPort 동기화 |
| Ingress 404/502 | Host/Path 불일치 | `describe ingress` | 규칙/백엔드 서비스 점검 |
| PVC 바인딩 실패 | SC/용량/모드 불일치 | `describe pvc` | SC/용량/zone 정책 맞춤 |

## 개발자 경험(DX)과 배포 전략

- **Helm** 패키징, **Kustomize** 오버레이, **GitOps(Argo CD/Flux)**.
- 기본 롤링 + 카나리/블루그린(서비스 메시/Ingress 가중치).

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
```

## 수용량/스케일링 간단 계산과 HPA

$$
\text{동시 처리량} \approx \text{QPS} \cdot t,\quad
\text{replicas}_{\min}=\left\lceil \frac{\text{동시 처리량}}{\text{cap}_{pod}} \right\rceil \cdot \text{버퍼}
$$

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 70 }
```

## 로컬/테스트 클러스터 빠른 시작

### kind(개발자 친화)

```bash
kind create cluster --name demo
kubectl cluster-info
```

### kubeadm(실전 학습)

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube && sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config
# CNI 설치(예: Calico/Cilium)

```

## Control Plane vs Worker 요약 테이블 (확장판)

| 항목 | Control Plane | Worker Node |
|---|---|---|
| 핵심 책임 | 상태 저장, 정책/스케줄, 컨트롤 루프 | Pod 실행, 네트워킹/스토리지 연결 |
| 주요 컴포넌트 | kube-apiserver, etcd, scheduler, controller-manager, cloud-controller-manager | kubelet, kube-proxy/eBPF, containerd/CRI-O |
| 확장성 포인트 | 다중 apiserver + etcd 합의 | 노드 풀 증감, HPA/VPA/CA, taints/tolerations |
| 장애 영향 | 클러스터 전반(쓰기 불가 등) | 특정 노드 파드(재스케줄로 완화) |
| 보안 포커스 | 인증/인가/어드미션/감사, etcd 암호화 | 런타임 최소 권한, NetworkPolicy/볼륨 격리 |

## 결론 — 선언형 오케스트레이션의 심장

쿠버네티스 아키텍처는 **중앙 집중 제어**와 **분산 실행**을 분리해, **원하는 상태를 선언**하고 컨트롤 플레인이 **지속 수렴**하도록 만든다. 다음 단계 제안:
1) Helm/Kustomize 템플릿화, 2) HPA+관측 대시보드, 3) RBAC/PSA/NetworkPolicy 기준선, 4) 업그레이드/백업/DR 리허설.
