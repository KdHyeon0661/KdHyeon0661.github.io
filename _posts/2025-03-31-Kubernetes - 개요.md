---
layout: post
title: Kubernetes - 개요
date: 2025-03-31 19:20:23 +0900
category: Kubernetes
---
# 1. 들어가며

- 왜 필요한가? → **수작업 배포의 한계**, **대규모 컨테이너 운영의 필요**, **자동화와 신뢰성**
- 무엇을 제공하는가? → **배포/롤링 업데이트/롤백**, **부하분산과 트래픽 제어**, **헬스체크와 자가치유**, **자동 확장**, **설정/비밀 관리**
- 어떻게 쓰는가? → **YAML 선언(Declarative)**, `kubectl`/CI/CD/GitOps, Helm/Kustomize
- 운영 관점에서? → **관측(로그/메트릭/트레이스)**, **보안(RBAC/NetworkPolicy/PodSecurity)**, **리소스/스케줄링**, **업그레이드/백업/DR**

---

# 2. 쿠버네티스의 정의와 어원

- **Kubernetes**는 그리스어 **“조타수(helmsman)”** 에서 유래.
- Google 내부 Borg/Borgmon 경험을 토대로 공개, 현재는 **CNCF** 산하에서 커뮤니티 주도로 발전.
- 핵심은 **“선언형(Declarative)”** 으로 **원하는 상태(Desired State)** 를 기술하면, **컨트롤러 루프**가 실제 상태를 지속적으로 일치시키는 것.

> 요약: “_내가 원하는 서비스 상태를 선언하면, 쿠버네티스가 계속 맞춰준다._”

---

# 3. 왜 필요한가? — 전통/컨테이너/오케스트레이션의 맥락

## 3.1 수작업 배포의 한계 (기존 글 확장)

- **환경 편차**: 서버마다 OS 패키지/런타임 버전이 달라, 재현 불가 이슈 빈발
- **스케일링 난이도**: 서버 개수를 늘릴수록 배포/롤백/상태 점검 비용 급증
- **가용성 관리**: 장애 시 수동 재시작, 시그널 처리, 헬스체크 부재
- **운영 가시성 결핍**: 로그/메트릭/트레이스의 표준화/집중화 부족

## 3.2 컨테이너화의 대두 (기존 글 확장)

- **이미지 기반 불변 인프라**: Docker/OCI 이미지로 동형 배포(같은 이미지면 어디서나 동일)
- **격리/밀도**: 프로세스 격리, VM 대비 가벼움 → **리소스 효율** ↑
- **하지만** 컨테이너 수가 늘면 **스케줄링, 복구, 네트워킹, 스토리지, 배포 전략** 등이 필요 → **오케스트레이션**의 등장

## 3.3 쿠버네티스가 해결하는 것 (기존 글 강화)

- **자동 배포/업데이트/롤백**: 롤링/카나리/블루그린 전략
- **부하분산/서비스 디스커버리**: Service/Ingress/게이트웨이
- **자가치유(Self-healing)**: liveness/readiness/startup probe
- **자동 확장**: HPA/VPA/Cluster Autoscaler
- **설정/비밀 분리**: ConfigMap/Secret (KMS 통합 가능)
- **선언형/불변성/재현성**: GitOps와 궁합

---

# 4. 전통 vs Docker vs Kubernetes (기존 표 확장)

| 항목 | 전통 배포 | Docker 단독 | Kubernetes |
|---|---|---|---|
| 배포 | ssh/scp 스크립트 | `docker run` | Declarative YAML + 컨트롤러 |
| 스케일링 | 수동 서버 증설 | 컨테이너 복제 수동 | ReplicaSet/Deployment/HPA |
| 장애 대응 | 수동 재시작 | 컨테이너 재실행 수동 | Probe + 재스케줄링/재시작 |
| 서비스 디스커버리 | 별도 구성 | 제한적 | Service/ClusterDNS |
| L4/L7 트래픽 | 로드밸런서 장비 | 별도 | Service/Ingress/Gateway |
| 설정/비밀 | 파일/ENV | Dockerfile/ENV | ConfigMap/Secret |
| 관측 | 개별 수집 | 제한적 | 표준화된 Exporter/EFK/OTel |
| 배포 전략 | 위험/불가역 | 수동 교체 | 롤링/카나리/블루그린/즉시 롤백 |
| 멀티테넌시/보안 | 취약 | 취약 | RBAC/NetworkPolicy/PSA |

---

# 5. 빠른 비교 예시 (기존 예시 확장)

## 5.1 Docker로 단일 Nginx

```bash
docker run -d -p 80:80 --name web nginx
```

- 장점: 간단함
- 한계: 스케일/복구/업데이트/트래픽 제어/관측은 수동 구성 필요

## 5.2 Kubernetes로 Nginx (Deployment + Service)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 2
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

```bash
kubectl apply -f web.yaml
kubectl get deploy,rs,po,svc -l app=web
```

- `replicas: 3` → 자동 스케일/자가치유
- `readiness/liveness` → 배포 중 트래픽 안전성 확보
- `requests/limits` → 스케줄링/자원 격리 기반

---

# 6. 쿠버네티스 아키텍처 — Control Plane와 Worker

## 6.1 Control Plane

- **API Server**: 모든 요청의 단일 진입점. `kubectl`/컨트롤러/스케줄러가 이 경로로 상호작용
- **etcd**: 클러스터 상태 저장(강한 일관성 Key-Value Store)
- **Controller Manager**: Deployment/Job/Node 등 **실제 상태를 원하는 상태**로 맞추는 컨트롤 루프
- **Scheduler**: 새 Pod을 자원/제약 조건 만족 노드로 **할당**

## 6.2 Worker Node

- **kubelet**: 노드/Pod 라이프사이클 관리
- **Container Runtime**: containerd/CRI-O 등
- **CNI**: Pod 네트워킹 플러그인 (Calico, Cilium 등)
- **kube-proxy**(ipvs/iptables) 또는 CNI-LB: Service 가상 IP/로드밸런싱

---

# 7. 핵심 리소스와 실전 YAML

## 7.1 Pod, ReplicaSet, Deployment

- **Pod**: 배포 가능한 최소 단위(1+ 컨테이너)
- **ReplicaSet**: 동일 Pod의 목표 복제수 유지
- **Deployment**: ReplicaSet의 선언적 롤링 업데이트/롤백

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: api }
spec:
  replicas: 4
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
      - name: api
        image: ghcr.io/example/api:1.2.3
        envFrom:
        - configMapRef: { name: api-config }
        - secretRef: { name: api-secret }
        ports: [{ containerPort: 8080 }]
        resources:
          requests: { cpu: "200m", memory: "256Mi" }
          limits: { cpu: "1", memory: "512Mi" }
        readinessProbe:
          httpGet: { path: /healthz, port: 8080 }
          periodSeconds: 5
        livenessProbe:
          httpGet: { path: /livez, port: 8080 }
          initialDelaySeconds: 10
```

## 7.2 Service & Ingress

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-svc
spec:
  type: ClusterIP
  selector: { app: api }
  ports:
  - name: http
    port: 80
    targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
```

## 7.3 ConfigMap & Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: api-config }
data:
  LOG_LEVEL: "info"
  FEATURE_X_ENABLED: "true"
---
apiVersion: v1
kind: Secret
metadata: { name: api-secret }
type: Opaque
stringData:
  DB_PASSWORD: "s3cr3t"
```

> Secret은 Base64 인코딩일 뿐이므로, **암호화 제공자(KMS) 연계**/`EncryptionConfiguration`으로 **at-rest 암호화** 권장.

## 7.4 Volume, PVC, StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: fast }
provisioner: kubernetes.io/no-provisioner # 예: 로컬 프로비저너 교체 가능
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data-pvc }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast
  resources:
    requests: { storage: 10Gi }
```

---

# 8. 배포 전략 — 롤링/카나리/블루그린

## 8.1 롤링 업데이트 (기본)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

- 장점: 다운타임 최소화
- 주의: readinessProbe가 정확해야 트래픽 안정

## 8.2 카나리

- 새 버전 ReplicaSet 일부만 노출 → 오류/성능 검증 후 점진적 확대
- Ingress/서비스 라우팅(헤더/가중치) or 서비스 셀렉터 분리로 구현

예시(가중치 라우팅은 Ingress 구현체/서비스메시에 의존):

```yaml
# canary-annotation 예시(nginx ingress 기반)
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
```

## 8.3 블루그린

- Blue(현재)와 Green(신규)를 병행 배치 → 스위치 순간에 트래픽 전환
- 롤백 빠름. 비용/자원 여유 필요

---

# 9. 오토스케일링 — HPA / VPA / Cluster Autoscaler

## 9.1 HPA (Horizontal Pod Autoscaler)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api-hpa }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

- 메트릭 소스: CPU/메모리/외부/커스텀(예: QPS, 대기열 길이)
- **readiness**가 부정확하면 HPA가 오류를 키울 수 있음

## 9.2 VPA (Vertical Pod Autoscaler)

- Pod의 `requests/limits`를 동적으로 조정 (재시작 수반 가능)
- 안정화 기간/관측 기간 고려

## 9.3 Cluster Autoscaler

- Node 그룹 규모 자동 증감(클라우드 종속)
- 스케줄 실패 → 노드 증설 트리거

---

# 10. 스케줄링 제어 — Affinity/Taints/TopologySpread

```yaml
spec:
  nodeSelector:
    disktype: ssd
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
    topologyKey: "topology.kubernetes.io/zone"
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels: { app: api }
```

- **AntiAffinity**로 동일 워크로드 분산
- **Taint/Toleration**으로 전용 노드 운영
- **TopologySpread**로 AZ/노드 전반에 고르게 배치

---

# 11. 리소스 모델과 QoS

- **Requests**: 최소 보장, 스케줄러 기준
- **Limits**: 상한. CPU 제한 시 cgroup throttle, 메모리 초과 시 OOMKill
- **QoS Class**
  - Guaranteed: req==limit 모두 지정
  - Burstable: 일부 지정
  - BestEffort: 미지정 → 가장 먼저 제거 대상

성능/안정성/비용을 동시에 고려해 **관측-조정 루프**로 지속 최적화한다.

수용량 산정의 예시 계산을 수식으로 표현하면:

$$
\text{Target Nodes} \approx
\frac{\sum_i \max(\text{req}_{\text{cpu}, i} / U_\text{cpu}, \text{req}_{\text{mem}, i} / U_\text{mem})}{\min(\text{node}_\text{cpu}, \text{node}_\text{mem})}
$$

여기서 \(U\)는 목표 자원 사용률(예: 0.6~0.7), `node_cpu/mem`은 노드 제공량.

---

# 12. 네트워킹과 서비스 메시 (선택)

- **CNI**: Pod ↔ Pod 기본 통신
- **Service**: ClusterIP/NodePort/LoadBalancer
- **Ingress/Gateway**: L7 라우팅/정책/보안
- **서비스 메시(Istio/Linkerd)**:
  - **장점**: 트래픽 분할/관측/정책/회로차단/리트라이/MTLS
  - **주의**: 복잡도/오버헤드, 팀 역량/운영 툴링 정비 필요

---

# 13. 관측(Observability) — 로그/메트릭/트레이스

## 13.1 로그

- **EFK/ELK**: Fluent Bit/Fluentd → Elasticsearch → Kibana
- 앱 표준 출력(stdout/stderr)로 남기고, 사이드카 혹은 데몬셋 수집

## 13.2 메트릭

- **Prometheus** + **exporter**(node, kube-state, app)
- **Alertmanager**로 경보 라우팅
- **Grafana** 대시보드 표준화

예: 간단한 `ServiceMonitor` (Prometheus Operator 사용 시)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: api-sm }
spec:
  selector:
    matchLabels: { app: api }
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

## 13.3 트레이스

- OpenTelemetry(OTel) SDK/Collector → Jaeger/Tempo
- 분산 트랜잭션 성능 병목 파악

---

# 14. 보안 — 계층별 체크리스트

## 14.1 접근제어

- **RBAC**: 최소 권한 원칙
- **네임스페이스 격리**, **리소스 쿼터**
- **Pod Security Admission(PSA)**: `baseline/restricted` 적용

## 14.2 네트워크

- **NetworkPolicy**: Pod 간 트래픽 허용 리스트
- **Ingress 보안**: TLS/보안 헤더, WAF/라이트레이트 제한

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-web }
spec:
  podSelector: { matchLabels: { app: web } }
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector: { matchLabels: { role: frontend } }
    ports: [{ protocol: TCP, port: 80 }]
```

## 14.3 런타임

- Root-less 컨테이너, readOnlyRootFilesystem, drop capabilities
- 이미지 서명/취약점 스캔(Trivy/Grype), 정책 게이트(Kyverno/OPA)
- Secret 관리: KMS 통합, 외부 비밀관리(Vault/ASM/PS)

---

# 15. CI/CD와 GitOps

## 15.1 CI

- 이미지 빌드: Docker/BuildKit/kaniko/Buildpacks
- 스캔/테스트/서명(SBOM 포함)

## 15.2 CD

- **Helm**: 템플릿 기반 패키징
- **Kustomize**: 오버레이/패치 기반
- **GitOps(Argo CD/Flux)**: Git 선언 → 클러스터 동기화

간단한 Helm `values.yaml` 예:

```yaml
image:
  repository: ghcr.io/example/api
  tag: "1.2.3"
replicaCount: 3
resources:
  requests: { cpu: 200m, memory: 256Mi }
  limits: { cpu: 1, memory: 512Mi }
ingress:
  enabled: true
  className: nginx
  hosts:
  - host: api.example.com
    paths:
    - path: /
      pathType: Prefix
```

---

# 16. 환경별 전략 — 로컬/스테이징/프로덕션

- **로컬**: kind/minikube → 기능 검증, 개발 루프 단축
- **스테이징**: 실제와 유사한 인프라, 데이터 샘플링, 카나리 사전 검증
- **프로덕션**: 멀티 AZ, 백업/DR, 가용성 목표(SLO/SLA), 비용 모니터링

---

# 17. 스토리지와 상태풀(Stateful) 워크로드

- **StatefulSet**: 안정된 네트워크 ID/스토리지 바인딩
- DB/RabbitMQ/Kafka 등은 **운영 책임 경계**를 명확히:
  관리형 서비스(RDS/MSK 등) vs 클러스터 내 운영

예: StatefulSet 스니펫

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: db }
spec:
  serviceName: "db"
  replicas: 3
  selector: { matchLabels: { app: db } }
  template:
    metadata: { labels: { app: db } }
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
      resources:
        requests: { storage: 100Gi }
```

---

# 18. 운영 수명주기 — 업그레이드/백업/DR

- **업그레이드**: Control Plane → Worker 순, 호환성/디프리케이션 체크
- **백업**: etcd 스냅샷, 오브젝트 백업(Velero), PVC 스냅샷
- **DR**: 다른 리전에 재구성 가능한 매니페스트/이미지 레지스트리 가용성 확보

etcd 백업 예(개념):

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-$(date +%F).db
```

---

# 19. 비용/성능 최적화

- **리소스 요청/상한 정밀화**: 실제 사용량 기반 조정
- **노드 패킹/스팟 혼합**: 파드 우선순위/선점 고려
- **오토스케일 튜닝**: 쿨다운, 버퍼
- **이미지 최적화**: multi-stage build, slim base
- **네트워킹**: 큰 응답/대기열 조정, keepalive/HTTP2/TLS 재활용

---

# 20. 시나리오 기반 실습

## 20.1 “3주 내 MSA 전환 PoC” 시나리오

**목표**: 단일 API 서버를 3개 마이크로서비스로 분리, 카나리로 점진 이행

1) **리포 분리/공통 라이브러리 정리**
2) Dockerfile/빌드 파이프라인(테스트/스캔/서명)
3) Helm 차트 생성 → `values`로 환경별 오버라이드
4) 공용 Ingress 도메인 규칙 설계
5) HPA 도입(메트릭: CPU + 큐 길이)
6) 카나리 10% → 30% → 100% 증량
7) 메트릭/로그/트레이스에서 오류율/지연 체크
8) 레이트리밋/리트라이/서킷브레이커(서비스 메시) 도입

**검증 체크리스트**

- 배포 시간/롤백 시간
- P95/P99 레이턴시, 에러율
- 캐패시티 여유율, HPA 동작 로그
- 알람 민감도/오탐률

## 20.2 “고가용 웹 + 캐시 + 백엔드 API” 예제 매니페스트

```yaml
# web-frontend Deployment
apiVersion: apps/v1
kind: Deployment
metadata: { name: web }
spec:
  replicas: 4
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
      - name: web
        image: ghcr.io/example/web:2.0.1
        env:
        - name: API_BASE
          value: "http://api-svc"
        ports: [{ containerPort: 3000 }]
        readinessProbe:
          httpGet: { path: /healthz, port: 3000 }
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata: { name: web-svc }
spec:
  selector: { app: web }
  type: ClusterIP
  ports:
  - name: http
    port: 80
    targetPort: 3000
---
# cache (Redis)
apiVersion: apps/v1
kind: Deployment
metadata: { name: cache }
spec:
  replicas: 2
  selector: { matchLabels: { app: cache } }
  template:
    metadata: { labels: { app: cache } }
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports: [{ containerPort: 6379 }]
---
apiVersion: v1
kind: Service
metadata: { name: cache-svc }
spec:
  selector: { app: cache }
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
---
# api backend
apiVersion: apps/v1
kind: Deployment
metadata: { name: api }
spec:
  replicas: 3
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
      - name: api
        image: ghcr.io/example/api:1.5.0
        env:
        - name: CACHE_HOST
          value: "cache-svc:6379"
        ports: [{ containerPort: 8080 }]
        readinessProbe:
          httpGet: { path: /healthz, port: 8080 }
---
apiVersion: v1
kind: Service
metadata: { name: api-svc }
spec:
  selector: { app: api }
  ports:
  - name: http
    port: 80
    targetPort: 8080
---
# ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fe-be
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: web-svc, port: { number: 80 } } }
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: api-svc, port: { number: 80 } } }
```

**포인트**: 의존성은 DNS(Service 이름)로 느슨히 결합, 캐시 다운 시 API fallback/디그레이드 전략 추가 권장.

---

# 21. 트러블슈팅 — 자주 겪는 문제와 해결

| 증상 | 원인 후보 | 해결 접근 |
|---|---|---|
| 파드 CrashLoopBackOff | 설정/비밀 누락, 포트 충돌, 리소스 OOM | `kubectl logs -p`, `describe pod`, 리소스 재조정, Secret/ConfigMap 확인 |
| Readiness 미통과로 트래픽 없음 | 헬스엔드포인트 경로/포트 오기 | probe 정의/지연/타임아웃 재설계 |
| 이미지 Pull 실패 | 레지스트리 인증/네트워크 | ImagePullSecret, 네트워크/DNS 점검 |
| HPA가 반응 없음 | 메트릭 파이프라인 미정의 | metrics-server/Prometheus 어댑터 확인 |
| PVC 바인딩 실패 | StorageClass/용량/접근모드 불일치 | SC/용량/VolumeBindingMode 확인 |
| Ingress 404/502 | 경로/호스트 오기, 백엔드 포트 mismatch | Ingress rule/service targetPort 정합성 점검 |
| 노드 불안정 | 커널/런타임 버전 안맞음, 디스크 압박 | 커널/CRI 호환, 노드 헬스 알람/자원 모니터링 |

디버깅 기본 명령:

```bash
kubectl get events --sort-by=.lastTimestamp -A
kubectl describe pod <name>
kubectl logs <name> -c <container> --since=10m
kubectl top pod -A
kubectl get --raw /metrics | head
```

---

# 22. 보강 학습 포인트 (기존 글에서 부족할 수 있는 부분 채우기)

- **Pod lifecycle**: initContainers, preStop hook, terminationGracePeriod
- **잡/크론잡**: 배치/스케줄 작업
- **멀티-컨테이너 패턴**: 사이드카/어댑터/앰배서더
- **정책/검증**: Admission Controller, OPA/Kyverno 정책으로 사전 차단
- **운영 가드레일**: 리소스 쿼터, LimitRange, PDB(PodDisruptionBudget)
- **개발자 경험(DX)**: Telepresence/Bridge to Kubernetes로 로컬-클러스터 하이브리드 디버깅
- **API 안정성**: 버전/스키마 진화, CRD + 컨트롤러(오퍼레이터 패턴)

PDB 예시:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: api-pdb }
spec:
  minAvailable: 2
  selector:
    matchLabels: { app: api }
```

---

# 23. 현업 체크리스트 — 도입 전/후

## 23.1 도입 전 질문

- 왜 K8s인가? (관리형 PaaS/서버리스 대비)
- 팀의 **운영 역량**(SRE/플랫폼팀)과 **툴체인** 준비?
- 애플리케이션이 **12-Factor**에 근접하는가?
- 관측/보안/백업/DR **기준선**은 있는가?

## 23.2 도입 후 가드레일

- 네임스페이스/리소스쿼터/LimitRange 표준값
- 공용 모듈화(Helm/Kustomize), 릴리즈 네이밍 규약
- 오류 예산(SLO/SLA)과 알람 정책
- 비용 대시보드(네임스페이스/라벨 태깅)

---

# 24. 결론 — 쿠버네티스는 “신뢰 가능한 운영”의 기반

쿠버네티스는 **단순히 많은 컨테이너를 띄우는 도구가 아니라**,
_원하는 상태를 선언_ 하고, **계속 그 상태를 지키도록 자동화**하는 **운영 플랫폼**이다.
**배포/확장/트래픽 제어/관측/보안/복구**를 하나의 프레임으로 통합하면서,
팀은 **기능 개발에 더 집중**하고 운영은 **정책과 선언**으로 다룰 수 있게 된다.

- 소규모라면 **과도한 복잡성**을 경계하되, 장기 성장을 본다면 **표준화/자동화** 관점에서 유리
- 성공의 핵심: **관측/보안/가드레일**과 **개발자 경험**의 균형

> 다음 단계: 현재 서비스 1~2개를 **Helm/Kustomize**로 템플릿화하고,
> **HPA + 관측 대시보드**를 갖춘 **소규모 GitOps 파이프라인**부터 시작하자.

---

# 부록 A. 로컬 실습 빠른 스타트

```bash
# kind 설치 후
kind create cluster --name demo

# Ingress-NGINX 설치(예시)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# 샘플 배포
kubectl apply -f web.yaml
kubectl get all -A
```

---

# 부록 B. 간단한 Go API 예시 + Dockerfile

```go
// main.go
package main

import (
  "fmt"
  "log"
  "net/http"
  "os"
)

func healthz(w http.ResponseWriter, r *http.Request) { w.WriteHeader(200); w.Write([]byte("ok")) }

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/healthz", healthz)
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "hello from %s\n", os.Getenv("HOSTNAME"))
  })
  log.Println("listen :8080")
  log.Fatal(http.ListenAndServe(":8080", mux))
}
```

```dockerfile
# Dockerfile
FROM golang:1.23 as build
WORKDIR /src
COPY . .
RUN go build -o app

FROM gcr.io/distroless/base-debian12
COPY --from=build /src/app /app
EXPOSE 8080
USER 65532:65532
ENTRYPOINT ["/app"]
```

Helm 차트/쿠버 매니페스트는 본문 예제를 참고해 구성한다.

---

# 부록 C. 성능/용량 계획 간단 공식

요청/초(QPS), 평균 처리시간(초), 동시성, HPA 목표를 잇는 단순 지표:

$$
\text{동시 처리량} \approx \text{QPS} \times \text{평균 처리시간}
$$

여기에 Pod 1개가 안정적으로 처리 가능한 동시 처리량 \(\text{cap}_{pod}\) 이라면, 최소 파드 수:

$$
\text{replicas}_{min} = \left\lceil \frac{\text{동시 처리량}}{\text{cap}_{pod}} \right\rceil \times \text{버퍼 계수}
$$

버퍼 계수(예: 1.2~1.5)로 스파이크를 흡수하고, HPA는 관측 기반으로 미세 조정한다.
