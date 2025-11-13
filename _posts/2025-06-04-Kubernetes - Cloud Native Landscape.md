---
layout: post
title: Kubernetes - Cloud Native Landscape
date: 2025-06-04 22:20:23 +0900
category: Kubernetes
---
# Cloud Native Landscape 이해하기

클라우드 네이티브는 **컨테이너 + 선언형 구성 + 자동화 + 관측성 + 회복력**으로 대표되는 운영 패러다임입니다.
CNCF(Cloud Native Computing Foundation)는 방대한 생태계를 **Landscape**로 체계화했고, 우리는 이 지도를 통해 **문제→영역→도구**를 신속히 대응할 수 있습니다.

---

## 1. Cloud Native란 무엇인가 — 운영 관점 재정의

> **정의**: 불변(Immutable) 아티팩트와 선언형(Declarative) 스펙으로 시스템을 구성하고, 자동화된 파이프라인과 컨트롤루프(Reconciler)로 **원하는 상태(Desired State)** 를 일관되게 유지/복구하는 접근.

핵심 속성
- **컨테이너화**: 환경 의존성 제거, 일관된 배포 단위
- **마이크로서비스**: 작은 단위의 독립 배포/스케일링
- **GitOps/DevOps**: 변경 이력·검증·롤백 표준화
- **관측성**: 메트릭/로그/트레이스로 가시성 확보
- **보안 내재화**: 시프트레프트(코드·이미지·런타임 전 과정)

> **가용성 추정(연간 가용성)**
> 서비스 서브시스템 가용성 \(A_i\)가 독립이라고 가정하면 전체 가용성 \(A\)는
> $$ A = \prod_{i=1}^{n} A_i $$
> 예) 3구성 요소가 각각 99.95%라면 전체 가용성은 \(0.9995^3 \approx 99.85\%\). 병목을 찾고 상위 SLO를 역산하는 데 유용합니다.

---

## 2. CNCF Landscape 한 장으로 보기

주요 카테고리
1. **Provisioning & Infrastructure**: IaC, 클러스터 부트스트랩
2. **Orchestration & Management**: Kubernetes, 스케줄링/운영
3. **Runtime**: containerd/CRI-O, Wasm
4. **Networking**: CNI, API Gateway, Service Discovery
5. **Storage**: CSI, 분산 스토리지, 백업/복구
6. **Observability & Analysis**: 모니터링/로그/트레이스/프로파일링
7. **Security & Compliance**: 이미지/런타임/정책/비밀관리
8. **App Definition & DevOps**: Helm·Kustomize·GitOps·CI/CD
9. **Chaos/Testing**: Chaos Mesh, Litmus

**프로젝트 등급**
- **Graduated**: Kubernetes, Prometheus, Envoy, Linkerd, etc.
- **Incubating**: Argo, OpenTelemetry, Cilium, Flux 등
- **Sandbox**: 초기 실험/성장 단계 프로젝트

---

## 3. Provisioning & Infrastructure — IaC로 시작과 끝을 관리

### 3.1 Terraform로 EKS 예시(핵심만)
```hcl
provider "aws" { region = "ap-northeast-2" }

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "demo-eks"
  cluster_version = "1.29"
  vpc_id          = var.vpc_id
  subnet_ids      = var.subnet_ids
  eks_managed_node_groups = {
    default = {
      desired_size = 3
      instance_types = ["m6i.large"]
    }
  }
}
```
- **원칙**: 모든 인프라(클러스터/노드/보안그룹/Route53)를 코드화.
- **Pulumi**(TS/Python/Go)도 동일 기능, 언어 친화적인 IaC 선호 시 고려.

### 3.2 Kubeadm/Kubespray
- 온프렘/자체 VM에서 제어성↑. **프로덕션**은 HA control-plane 설계(다중 etcd) 필수.

---

## 4. Orchestration & Management — Kubernetes 운영 패턴

### 4.1 기본 배포 매니페스트(검증 지향)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels: {app: api}
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: {maxSurge: 1, maxUnavailable: 0}
  selector: {matchLabels: {app: api}}
  template:
    metadata: {labels: {app: api}}
    spec:
      containers:
      - name: api
        image: ghcr.io/acme/api:1.4.7
        resources:
          requests: {cpu: 200m, memory: 256Mi}
          limits:   {cpu: "1",  memory: 512Mi}
        ports: [{containerPort: 8080}]
        readinessProbe:
          httpGet: {path: /healthz/ready, port: 8080}
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet: {path: /healthz/live, port: 8080}
          initialDelaySeconds: 20
          periodSeconds: 10
      nodeSelector: {workload: general}
```

### 4.2 스케줄링 제약(요약)
- **Node Affinity/Anti-Affinity**, **topologySpreadConstraints**, **Taint/Toleration**로 **HA + 비용** 균형 설계.
- **HPA/VPA/Cluster Autoscaler**의 상호작용을 고려(요청치 기반).

---

## 5. Runtime — 컨테이너 런타임 & Wasm

- **containerd**(권장), **CRI-O**: Kubernetes 친화적인 경량 런타임.
- **Wasm**(WasmEdge/Wasmtime): 콜드스타트·보안 장점. **게이트웨이 플러그인/필터**나 **경량 함수형 워크로드**에서 파일럿 도입.

---

## 6. Networking — CNI, Ingress, Gateway API, Mesh

### 6.1 CNI 예시: Cilium (eBPF)
- **장점**: 고성능 데이터패스, L3-L7 정책, Hubble로 **흐름 관측**.
- **대안**: Calico(성숙/정책풍부), Flannel(간단·소규모).

### 6.2 Gateway API (차세대 Ingress)
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: {name: api-route}
spec:
  parentRefs: [{name: public-gw}]
  rules:
  - matches: [{path: {type: PathPrefix, value: "/api"}}]
    backendRefs: [{name: api, port: 8080}]
```
- Ingress의 한계를 보완한 **표준화된 확장 모델**.

### 6.3 Service Mesh(실전 기본)
- **Linkerd**(Graduated): 경량/단순 mTLS/헬스.
- **Istio**: 트래픽 관리/정책/멀티클러스터까지 포괄.

**Istio AuthorizationPolicy 예시**
```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata: {name: api-authz, namespace: default}
spec:
  selector: {matchLabels: {app: api}}
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/frontend/sa/web-sa"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/v1/*"]
```

---

## 7. Storage — CSI, 분산 스토리지, 백업/복구

### 7.1 PVC + StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {name: fast-gp3}
provisioner: ebs.csi.aws.com
parameters: {type: gp3, iops: "3000", throughput: "125"}
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```
- **WaitForFirstConsumer**: Pod 스케줄 후 **Zone 일치**로 올바른 볼륨 프로비저닝.

### 7.2 분산 스토리지: Rook/Ceph, Longhorn
- **Ceph**: 대규모/다양한 프로토콜.
- **Longhorn**: 경량·K8s 네이티브, 온프렘/엣지 친화.

### 7.3 백업/DR: Velero
```bash
velero backup create daily-$(date +%F) --include-namespaces prod --wait
```

---

## 8. Observability — Prometheus/Loki/Tempo/OTel/Grafana

### 8.1 Prometheus + Alertmanager
**간단 ServiceMonitor (prometheus-operator)**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: {name: api-sm, labels: {release: prom}}
spec:
  selector: {matchLabels: {app: api}}
  endpoints:
  - port: http
    interval: 15s
```

**PromQL 예시**
```promql
# CPU 사용률(%)
100 * rate(container_cpu_usage_seconds_total{container!="",pod=~"api-.*"}[5m])
  / on(pod) group_left
    kube_pod_container_resource_requests{resource="cpu", unit="core"}
```

### 8.2 Loki(로그) + Promtail
```yaml
scrape_configs:
- job_name: kubernetes-pods
  kubernetes_sd_configs: [{role: pod}]
  pipeline_stages:
  - docker: {}
```

### 8.3 OpenTelemetry(표준화된 SDK/Collector)
**OTel Collector to Prometheus + Tempo**
```yaml
receivers:
  otlp: {protocols: {http: {}, grpc: {}}}
exporters:
  prometheus: {endpoint: "0.0.0.0:9464"}
  otlp:
    endpoint: tempo:4317
service:
  pipelines:
    metrics: {receivers: [otlp], exporters: [prometheus]}
    traces:  {receivers: [otlp], exporters: [otlp]}
```

---

## 9. Security & Compliance — Shift Left + Runtime

### 9.1 이미지 스캔: Trivy
```bash
trivy image ghcr.io/acme/api:1.4.7 --severity CRITICAL,HIGH
```

### 9.2 정책: OPA Gatekeeper / Kyverno

**Gatekeeper ConstraintTemplate + Constraint**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata: {name: k8srequiredlabels}
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels
      violation[{"msg": msg}] {
        required := {"owner", "team"}
        provided := {k | input.review.object.metadata.labels[k]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("missing required labels: %v", [missing])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata: {name: must-have-owner-team}
spec:
  match:
    kinds: [{apiGroups: [""], kinds: ["Namespace"]}]
```

**Kyverno(예: root 금지)**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: {name: disallow-root}
spec:
  validationFailureAction: enforce
  rules:
  - name: runAsNonRoot
    match:
      resources: {kinds: ["Pod"]}
    validate:
      message: "Containers must not run as root."
      pattern:
        spec:
          securityContext:
            runAsNonRoot: true
```

### 9.3 런타임 보안: Falco
- 커널 이벤트 기반 **행동 탐지**(eBPF). 무단 쉘/민감파일 접근 규칙으로 경보.

### 9.4 인증서·신뢰: cert-manager / SPIFFE/SPIRE
- **내부 mTLS**, 서비스 아이덴티티 자동화.

---

## 10. App Definition & DevOps — Helm/Kustomize/GitOps/CI

### 10.1 Helm 차트 베이스(템플릿 추상화)
```yaml
# values.yaml
image:
  repository: ghcr.io/acme/api
  tag: "1.4.7"
replicaCount: 3
resources:
  requests: {cpu: 200m, memory: 256Mi}
  limits:   {cpu: "1",  memory: 512Mi}
```

{% raw %}
```yaml
# templates/deployment.yaml (발췌)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "api.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: api
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        resources: {{- toYaml .Values.resources | nindent 10 }}
```
{% endraw %}

### 10.2 Kustomize 오버레이
```
base/ (deployment.yaml, service.yaml)
overlays/
  dev/kustomization.yaml      # namePrefix: dev-
  prod/kustomization.yaml     # replicas/리소스 상향, 주석 주입 등
```

### 10.3 GitOps: Argo CD(ApplicationSet 포함)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: {name: api, namespace: argocd}
spec:
  source:
    repoURL: https://github.com/acme/infra
    path: helm/api
    targetRevision: main
    helm:
      valueFiles: ["values-prod.yaml"]
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated: {prune: true, selfHeal: true}
    syncOptions: ["CreateNamespace=true"]
```

**ApplicationSet(다중 환경 생성)**
{% raw %}
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: {name: apps}
spec:
  generators:
  - list:
      elements:
      - {name: api-dev,  namespace: dev,  values: values-dev.yaml}
      - {name: api-prod, namespace: prod, values: values-prod.yaml}
  template:
    metadata:
      name: "{{name}}"
      namespace: argocd
    spec:
      source:
        repoURL: https://github.com/acme/infra
        path: helm/api
        targetRevision: main
        helm: {valueFiles: ["{{values}}"]}
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{namespace}}"
      syncPolicy: {automated: {prune: true, selfHeal: true}}
```
{% endraw %}

### 10.4 CI(예: GitHub Actions → 이미지 빌드/푸시)
{% raw %}
```yaml
name: ci
on: {push: {branches: [main]}}
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: docker/setup-buildx-action@v3
    - uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    - name: Build & Push
      run: |
        docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
        docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```
{% endraw %}

---

## 11. Chaos/Resilience — Litmus / Chaos Mesh

- 장애 주입으로 **SLO·오토스케일·헬스 체크** 유효성 검증.
```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata: {name: pod-kill, namespace: chaos}
spec:
  appinfo: {appns: prod, applabel: "app=api", appkind: deployment}
  experiments:
  - name: pod-delete
```

---

## 12. 선택 가이드 — 상황별 추천 조합

| 요구/규모 | 네트워킹 | 배포 | 관측 | 보안 | 스토리지 |
|---|---|---|---|---|---|
| 소규모/단일 AZ | Flannel/Calico | Helm + ArgoCD(단일) | Prometheus+Grafana+Loki | Trivy+Kyverno(basic) | EBS/PD + Velero |
| 중규모/멀티 AZ | Calico/Cilium | Helm + ArgoCD(ApplicationSet) | + Tempo/OTel | Falco+Gatekeeper | EBS/PD + Rook/Ceph |
| 대규모/멀티 리전 | Cilium + Mesh | GitOps(멀티클러스터) | Thanos/Cortex | SPIRE+mTLS+OPA/PSA | Regional DR + Velero |

**서비스 메시 선택**
- 단순 mTLS/헬스/레이트리밋: **Linkerd**
- 복합 라우팅/멀티클러스터/정책 풍부: **Istio**

---

## 13. 비용·성능·안정성 최적화 체크리스트

- **Requests/Limits 적정화**: HPA 기준·Throttling/OOM 방지
- **노드 풀 분리**: CPU/GPU/메모리 최적, 스팟 혼용(비핵심)
- **스토리지**: RWO/RWX·IOPS/Throughput 맞춤
- **eBPF(Cilium)**: L7 정책·가시성으로 문제 시간 단축
- **SLO→SLA 역산**: 상단의 가용성 곱셈식으로 약점 찾기
- **릴리즈 전략**: Canary/Blue-Green + 자동 롤백(에러·라틴시 기준)
- **보안**: 이미지 서명(COSIGN), Supply Chain(SLSA) 단계적 도입

---

## 14. 레퍼런스 아키텍처(요약 다이어그램 텍스트)

```
[Dev] --(Git Push)--> [CI: Actions/Tekton] --(Image)--> [Registry]
         \                                            /
          \--(Infra IaC)--> [Cloud/VPC/Cluster by Terraform]
[Argo CD] --(GitOps Sync)--> [Kubernetes Cluster]
   |-- Helm/Kustomize
   |-- AppSet (multi env/cluster)
[Networking] CNI(Cilium) + Gateway API + (optional) Istio/Linkerd
[Observability] Prom + Loki + Tempo + OTel + Grafana
[Security] Trivy + OPA/Kyverno + cert-manager + Falco
[Storage] CSI(EBS/PD) + (Rook/Ceph) + Velero
```

---

## 15. 실전 운영 런북(핵심 절차)

1) **문제 인입**(지연/오류율): SLO 대시보드 확인 → 영향 범위
2) **서비스 경로 추적**: Gateway/Route/Mesh → 대상 워크로드 식별
3) **Pod 상태**: `kubectl describe`/이벤트/Probe/HPA
4) **리소스**: CPU/메모리/ephemeral-storage/네트워크/디스크
5) **릴리즈/환경차**: GitOps Diff/Helm Diff
6) **정책/보안**: PSA/Gatekeeper/Kyverno 로그
7) **회복**: 라우팅 롤백/Canary 중단/Autoscaler 상향
8) **사후분석**: 트레이스(Tempo/Jaeger), 로그(Loki), 근본원인 문서화

---

## 16. 안티패턴 경계

- `latest` 태그 남용(재현 불가/롤백 불가)
- Requests 미설정(스케줄 편향/HPA 오작동)
- 단일 AZ/노드 집중(장애 도메인 과밀)
- 무분별한 ClusterRoleBinding(cluster-admin)
- 중앙관측 부재(사건 추적 불가)
- Git 밖 수동 kubectl 패치(드리프트·재발)

---

## 결론

- **Cloud Native**는 도구 나열이 아니라 **운영 철학**입니다.
- CNCF Landscape의 각 퍼즐(네트워킹/스토리지/관측/보안/배포)을 **문제 지향**으로 조립하세요.
- 본문 예제(Helm/Kustomize/ArgoCD/OPA/PromQL/Velocity)로 **즉시 적용 가능한 기준선**을 잡고,
  조직 상황(규모/규제/비용/경험)에 맞춰 **단계적 심화**(Mesh/OTel/Policy/DR)를 진행하면 됩니다.

---

## 부록: 빠른 실습 번들(리스트)

- IaC: Terraform EKS 모듈
- 배포: Helm 차트 + ArgoCD Application
- 관측: kube-prometheus-stack + Loki + Tempo + OTel Collector
- 보안: Trivy 스캔, Kyverno/Gatekeeper 한 가지 정책
- 네트워크: Cilium + Gateway API(또는 NGINX Ingress)
- 백업: Velero **하루 1회 스케줄** 추가

```bash
# Velero 스케줄 예시
velero schedule create daily --schedule="0 2 * * *" --include-namespaces prod
```

---

## 참고 링크

- CNCF: https://www.cncf.io/
- CNCF Landscape: https://landscape.cncf.io/
- Kubernetes Docs: https://kubernetes.io/
- Prometheus/Alerting: https://prometheus.io/
- OpenTelemetry: https://opentelemetry.io/
- Argo CD: https://argo-cd.readthedocs.io/
- Cilium: https://cilium.io/
- Kyverno: https://kyverno.io/
- Gatekeeper/OPA: https://open-policy-agent.github.io/gatekeeper/
- Velero: https://velero.io/
