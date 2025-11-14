---
layout: post
title: Kubernetes - Multi-Tenant
date: 2025-05-11 22:20:23 +0900
category: Kubernetes
---
# Multi-Tenant 환경에서 네임스페이스 분리 + 정책 구성

여러 팀(또는 고객)이 **하나의 Kubernetes 클러스터**를 공유하되 서로 **보안/안정성/운영 측면에서 완전 분리**되도록 만드는 것이 Multi-Tenancy의 목표다.

- 테넌트 경계: **네임스페이스** 중심 + **노드/스토리지/네트워크** 보조 격리
- 안전한 기본값: **기본 거부 네트워크 정책**, **PSA(베이스라인/리스트릭티드)**, **자원 할당**
- 권한과 거버넌스: **RBAC**, **감사/라벨링/과금 태깅**
- 자동화: **Helm/Kustomize**로 “테넌트 키트” 배포
- 검증/테스트 체크리스트와 트러블슈팅

---

## 설계 개요(빠른 로드맵)

| 범주 | 핵심 결정 |
|---|---|
| 격리 단위 | **Namespace**(필수) + 필요하면 **노드 풀/테인트**(하드 격리) |
| 기본 보안 | **Pod Security Admission(PSA)**: `restricted` 권장 |
| 리소스 | **ResourceQuota**, **LimitRange**, **PriorityClass**, **PDB** |
| 네트워크 | **Default Deny** Ingress/Egress + DNS/레지스트리 등 **허용 목록** |
| 권한 | 테넌트별 **Role/RoleBinding**, 그룹 매핑, 읽기/쓰기 분리 |
| 스토리지 | 테넌트 전용 **StorageClass**(선택) 또는 PVC 네임스페이스 경계 |
| 배포 | Helm/Kustomize로 **일관된 템플릿** 배포 |
| 가시성 | **라벨/어노테이션 표준화**, namespace-scoped 대시보드/로그 |
| 거버넌스 | **Kyverno/OPA**로 정책 강제(이미지 레지스트리/리소스 요청/보안 옵션 등) |

---

## 테넌트 네임스페이스 생성 + 표준 라벨/PSA

테넌트 생성 시 **라벨 표준**과 **PSA**를 함께 부여한다.

```yaml
# ns-tenant-a.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: team-dev
  labels:
    tenant/name: team-dev
    tenant/contact: dev-lead@example.com
    pod-security.kubernetes.io/enforce: "restricted"
    pod-security.kubernetes.io/audit:   "restricted"
    pod-security.kubernetes.io/warn:    "restricted"
```

```bash
kubectl apply -f ns-tenant-a.yaml
```

> PSA를 `restricted`로 두면 `privileged` 요구, `hostNetwork=true`, RunAsRoot 등 위험한 스펙이 기본 차단된다. 팀 특성상 완화가 필요하면 별도 샌드박스 네임스페이스를 따로 만든다.

---

## 리소스 격리: ResourceQuota + LimitRange + PriorityClass

### ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-team-dev
  namespace: team-dev
spec:
  hard:
    pods: "40"
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    persistentvolumeclaims: "20"
    requests.storage: 2Ti
```

### LimitRange(기본 요청/제한 강제)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: lr-team-dev
  namespace: team-dev
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    default:
      cpu: "1"
      memory: "1Gi"
    max:
      cpu: "2"
      memory: "2Gi"
```

### PriorityClass(선택)

공용 클러스터에서 운영/개발 우선순위를 구분한다.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: prio-team-dev
value: 1000
globalDefault: false
description: "team-dev workloads priority"
```

테넌트 Deployment에 붙인다:

```yaml
spec:
  template:
    spec:
      priorityClassName: prio-team-dev
```

---

## 네트워크 격리: Default Deny + 허용 목록

### Ingress/Egress 기본 거부

```yaml
# deny-by-default.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: team-dev
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
```

### 내부 통신 허용(동일 네임스페이스)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-ns
  namespace: team-dev
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector: {}   # 같은 NS 전체 허용(상황에 따라 app 라벨로 더 좁히기)
```

### DNS/Egress 허용(필수 아웃바운드)

```yaml
# allow-egress-dns-and-web.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-core
  namespace: team-dev
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
    ports:
    - protocol: TCP
      port: 443
```

> 실제 운영에서는 **회사 프록시/레지스트리/FQDN egress controller**를 통해 더 세밀하게 허용한다. Calico/CEF 등 CNI 확장을 쓰면 FQDN 기반 egress 도 가능하다.

---

## 권한 격리: RBAC(역할/바인딩), 서비스어카운트 경계

### 팀 운영자 롤(팀 네임스페이스 한정 관리자)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-dev-operator
  namespace: team-dev
rules:
- apiGroups: ["", "apps", "batch", "autoscaling"]
  resources: ["pods","services","configmaps","secrets","deployments","jobs","cronjobs","hpa"]
  verbs: ["*"]
- apiGroups: ["", "storage.k8s.io"]
  resources: ["persistentvolumeclaims"]
  verbs: ["create","get","list","watch","update","delete"]
```

팀 사용자/그룹 바인딩:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-dev-operator-binding
  namespace: team-dev
subjects:
- kind: Group
  name: grp-team-dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: team-dev-operator
  apiGroup: rbac.authorization.k8s.io
```

### 읽기 전용 롤

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-dev-readonly
  namespace: team-dev
rules:
- apiGroups: ["*", ""]
  resources: ["*"]
  verbs: ["get","list","watch"]
```

---

## 스토리지 격리: StorageClass/Quota/백업 경계

- PVC는 네임스페이스 경계를 벗어나지 않는다.
- 필요한 경우 팀 전용 StorageClass를 제공하고 ResourceQuota로 `requests.storage`를 제한한다.
- 백업/복구 도구(예: Velero)는 네임스페이스 단위 스냅샷·복구 전략을 적용한다.

예: 팀 전용 StorageClass 이름만 허용하도록 Kyverno로 강제(7장 참조).

---

## 노드 격리(선택): 테인트/톨러레이션 + 노드어피니티

민감 고객이나 빌링 분리를 강하게 원하면 **전용 노드풀**을 만든다.

```yaml
# 노드에 테인트 추가(예: nodepool=team-dev 전용)

kubectl taint nodes <node-name> tenant=team-dev:NoSchedule
```

워크로드에 톨러레이션/어피니티 부여:

```yaml
spec:
  template:
    spec:
      tolerations:
      - key: "tenant"
        operator: "Equal"
        value: "team-dev"
        effect: "NoSchedule"
      nodeSelector:
        tenant: team-dev
```

> 실제 노드 라벨 `tenant=team-dev`도 미리 부여한다: `kubectl label node <node> tenant=team-dev`.

---

## 거버넌스/정책 강제: Kyverno 또는 Gatekeeper(OPA)

### 리소스 요청/제한 강제(없으면 거부)

```yaml
# kyverno: require-requests-limits.yaml

apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-requests-limits
spec:
  validationFailureAction: Enforce
  rules:
  - name: containers-require-rr
    match:
      resources:
        kinds: ["Pod"]
    validate:
      message: "containers must set requests/limits for cpu and memory"
      pattern:
        spec:
          containers:
          - resources:
              requests:
                cpu: "?*"
                memory: "?*"
              limits:
                cpu: "?*"
                memory: "?*"
```

### 허용된 이미지 레지스트리만 사용

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: allowed-registries
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-registry
    match:
      resources:
        kinds: ["Pod"]
        namespaces: ["team-dev"]
    validate:
      message: "use images from registry.example.com only"
      pattern:
        spec:
          containers:
          - image: "registry.example.com/*"
```

### 위험 플래그 차단(hostNetwork/privileged 등)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-hostnetwork
spec:
  validationFailureAction: Enforce
  rules:
  - name: no-privileged
    match:
      resources: {kinds: ["Pod"]}
    validate:
      pattern:
        spec:
          containers:
          - securityContext:
              privileged: "false"
  - name: no-hostnetwork
    match:
      resources: {kinds: ["Pod"]}
    validate:
      pattern:
        spec:
          hostNetwork: "false"
```

> Gatekeeper(OPA)로도 동일한 제약을 Rego로 구현할 수 있다. 조직 표준/감사 요구에 따라 선택.

---

## 디플로이 기본 템플릿(테넌트 킷) — Helm/Kustomize

### Helm 값으로 테넌트 변수화 예시

`values.yaml`:

```yaml
tenant:
  name: team-dev
  contact: dev-lead@example.com

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1
    memory: 1Gi
```

`templates/namespace.yaml`:

{% raw %}
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.tenant.name }}
  labels:
    tenant/name: {{ .Values.tenant.name }}
    tenant/contact: {{ .Values.tenant.contact }}
    pod-security.kubernetes.io/enforce: "restricted"
```
{% endraw %}

`templates/quota.yaml`, `templates/limitrange.yaml`, `templates/networkpolicies/*.yaml`, `templates/rbac.yaml` 등을 같은 차트에 포함해서 **한 번에 테넌트 온보딩**이 가능하도록 설계한다.

설치:

```bash
helm install tenant-team-dev ./tenant-kit -n kube-system
```

### Kustomize 베이스/오버레이

```
tenant-kit/
  base/
    ns.yaml
    quota.yaml
    limitrange.yaml
    rbac.yaml
    np-default-deny.yaml
  overlays/
    team-dev/kustomization.yaml
    team-qa/kustomization.yaml
```

`kustomization.yaml`에서 네임스페이스/라벨/패치만 바꿔 각 팀 템플릿을 생성한다.

---

## 관측/감사/과금

### 라벨 표준화

모든 리소스에 공통 라벨을 강제:

```yaml
metadata:
  labels:
    tenant/name: team-dev
    app.kubernetes.io/part-of: web-suite
```

Kyverno `mutate`로 자동 주입하는 것도 좋다.

### 메트릭/로그/트레이싱

- Prometheus: `namespace` 별 리소스/애플리케이션 대시보드
- Grafana: 폴더/팀별 대시보드 분리
- Loki: `{namespace="team-dev"}` 쿼리로 로그 필터
- Tracing(Tempo/Jaeger): 팀 별 서비스맵

### 과금/쇼백(Showback/Chargeback)

- 자원 사용량 쿼리: `container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`
- 스토리지 사용: PVC capacity 및 CSI metrics
- 네임스페이스 라벨 단위로 **코스트 대시보드** 구축

---

## 신뢰성: PDB/HPA/BU

- **PodDisruptionBudget(PDB)**: 유지보수/노드 드레인에도 다운타임 최소화

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pdb-web
  namespace: team-dev
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

- **HPA**: 팀별 스케일 정책을 독립적으로 설정
- **백업/복구**: 네임스페이스 단위 스냅샷 계획 수립

---

## 예제 앱으로 격리 검증

### 배포

```bash
kubectl -n team-dev create deployment web --image=nginx
kubectl -n team-dev expose deployment web --port=80
kubectl -n team-qa create deployment web --image=nginx
kubectl -n team-qa expose deployment web --port=80
```

### 통신 테스트

기본 거부가 적용되었으므로, `team-qa`에서 `team-dev`의 `web` 접근은 차단되어야 한다.

```bash
kubectl -n team-qa run tester --image=busybox:1.36 -it --rm -- sh
wget -S -O- http://web.team-dev.svc.cluster.local   # 실패해야 정상
```

동일 네임스페이스 통신 허용을 설정했으면 `team-dev` 내부에서는 접근 가능하다.

---

## 운영 체크리스트

- 네임스페이스 생성 시 **PSA 라벨**과 **라벨 표준**을 반드시 포함
- **ResourceQuota/LimitRange** 없이는 배포 금지(정책으로 Enforce)
- **NetworkPolicy: Default Deny**부터 시작, 필요한 것만 허용
- **RBAC 최소 권한**: Role/RoleBinding만, ClusterRole은 신중히
- 이미지 출처/서명 검증(COSIGN, policy-controller)
- 비밀정보: **Secret 관리 스토리**(KMS/Sealed-Secrets/External Secrets)
- 각 팀에 **전용 SA/네임스페이스 kubeconfig** 제공(권한 축소)
- 변경은 반드시 **스테이징에 먼저 적용** 후 프로덕션 롤아웃
- **감사 로그** 및 정책 이벤트 대시보드 운영

---

## 트러블슈팅 가이드

| 현상 | 가능 원인 | 해결 |
|---|---|---|
| 다른 팀 네임스페이스에 접근됨 | Default Deny 누락, namespaceSelector 라벨 불일치 | 기본 거부 정책 추가, 네임스페이스 라벨 정합성 재점검 |
| 배포가 거부됨(PSA/정책 에러) | PSA restricted 위반, Kyverno/GK 정책 위반 | 이벤트/어드미션 거부 메시지 확인, 스펙 수정 |
| CPU/메모리 요청이 적용 안됨 | LimitRange/Quota 미설치 | 네임스페이스 정책 확인, 템플릿 재적용 |
| 외부 접속 실패 | Egress 기본 거부, DNS 허용 미구성 | DNS/필요 포트 화이트리스트 정책 추가 |
| 과금 대시보드에 팀 구분 불가 | 라벨 표준 미적용 | Kyverno mutate로 라벨 자동 주입 |

---

## “테넌트 키트” 한 번에 적용(샘플 번들)

```yaml
# 01-namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: team-dev
  labels:
    tenant/name: team-dev
    pod-security.kubernetes.io/enforce: "restricted"
---
# 02-quota.yaml

apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-team-dev
  namespace: team-dev
spec:
  hard:
    pods: "40"
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
---
# 03-limitrange.yaml

apiVersion: v1
kind: LimitRange
metadata:
  name: lr-team-dev
  namespace: team-dev
spec:
  limits:
  - type: Container
    defaultRequest: {cpu: "250m", memory: "256Mi"}
    default:        {cpu: "1",    memory: "1Gi"}
---
# 04-netpol-default-deny.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: team-dev
spec:
  podSelector: {}
  policyTypes: ["Ingress","Egress"]
---
# 05-netpol-egress-core.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-core
  namespace: team-dev
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - {protocol: UDP, port: 53}
    - {protocol: TCP, port: 53}
---
# 06-rbac-operator.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-dev-operator
  namespace: team-dev
rules:
- apiGroups: ["","apps","batch","autoscaling"]
  resources: ["pods","services","configmaps","secrets","deployments","jobs","cronjobs","horizontalpodautoscalers"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-dev-operator-binding
  namespace: team-dev
subjects:
- kind: Group
  name: grp-team-dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: team-dev-operator
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f 01-namespace.yaml \
              -f 02-quota.yaml \
              -f 03-limitrange.yaml \
              -f 04-netpol-default-deny.yaml \
              -f 05-netpol-egress-core.yaml \
              -f 06-rbac-operator.yaml
```

---

## 결론

1) **네임스페이스**는 멀티 테넌시의 기초다.
2) **PSA/Quota/LimitRange/NetworkPolicy/RBAC**를 “세트”로 묶어 **기본 안전 장치**를 만든다.
3) 필요 시 **노드 풀 격리**(테인트/어피니티), **스토리지 클래스 분리**, **정책 엔진(Kyverno/OPA)** 로 거버넌스를 강화한다.
4) **Helm/Kustomize**로 테넌트 온보딩을 자동화하고, **라벨 표준화 + 관측/감사/코스트**를 통해 운영 가시성을 확보한다.

이 글의 매니페스트/정책 템플릿을 조합하면, 한 클러스터에서 **안전하고 예측 가능한 멀티 테넌시**를 즉시 구축·검증할 수 있다.
