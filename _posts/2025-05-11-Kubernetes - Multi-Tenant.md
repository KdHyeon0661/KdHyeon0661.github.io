---
layout: post
title: Kubernetes - Multi-Tenant
date: 2025-05-11 22:20:23 +0900
category: Kubernetes
---
# Multi-Tenant 환경에서 네임스페이스 분리와 정책 구성

하나의 Kubernetes 클러스터를 여러 팀이나 고객이 공유하면서도 보안, 안정성, 운영 측면에서 완전히 분리된 환경을 구성하는 것이 Multi-Tenancy의 핵심 목표입니다. 이를 위해 네임스페이스를 중심으로 한 격리와 다양한 보안 정책을 체계적으로 적용해야 합니다.

---

## 설계 개요

멀티 테넌트 환경 구축을 위한 핵심 구성 요소는 다음과 같습니다.

| 범주 | 핵심 구성 요소 |
|---|---|
| 격리 단위 | **네임스페이스**(기본) + 필요한 경우 **노드 풀/테인트**(강력한 격리) |
| 기본 보안 | **Pod Security Admission(PSA)**: `restricted` 프로필 권장 |
| 리소스 관리 | **ResourceQuota**, **LimitRange**, **PriorityClass**, **PodDisruptionBudget** |
| 네트워크 격리 | **기본 거부(Default Deny)** 정책 + 필수 통신만 **허용 목록** 방식 |
| 권한 관리 | 테넌트별 **Role/RoleBinding**, 그룹 기반 접근 제어, 읽기/쓰기 권한 분리 |
| 스토리지 관리 | 테넌트 전용 **StorageClass** 또는 네임스페이스 경계 내 PVC 관리 |
| 배포 자동화 | Helm/Kustomize를 통한 **일관된 템플릿** 배포 |
| 모니터링 | **라벨/어노테이션 표준화**, 네임스페이스 스코프 대시보드 및 로그 |
| 거버넌스 | **Kyverno/OPA**를 통한 정책 강제(이미지 레지스트리, 리소스 요청, 보안 옵션 등) |

---

## 테넌트 네임스페이스 생성과 표준 라벨링

테넌트 생성 시 네임스페이스에 표준 라벨과 Pod Security Admission(PSA) 라벨을 함께 적용합니다.

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

> PSA를 `restricted`로 설정하면 `privileged`, `hostNetwork=true`, `runAsRoot` 등 위험한 설정이 기본적으로 차단됩니다. 특별한 요구사항이 있는 경우 별도의 샌드박스 네임스페이스를 생성하여 완화된 정책을 적용할 수 있습니다.

---

## 리소스 격리: ResourceQuota + LimitRange + PriorityClass

### ResourceQuota (리소스 할당량 제한)

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

### LimitRange (기본 요청/제한 강제)

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

### PriorityClass (우선순위 분류)

공용 클러스터에서 운영과 개발 워크로드의 우선순위를 구분합니다.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: prio-team-dev
value: 1000
globalDefault: false
description: "team-dev workloads priority"
```

테넌트의 Deployment에 적용:

```yaml
spec:
  template:
    spec:
      priorityClassName: prio-team-dev
```

---

## 네트워크 격리: 기본 거부(Default Deny)와 허용 목록

### 모든 수신/발신 기본 거부

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

### 동일 네임스페이스 내 통신 허용

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
    - podSelector: {}   # 동일 네임스페이스 내 모든 Pod 허용
```

### 필수 아웃바운드 통신 허용 (DNS 및 HTTPS)

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

> 실제 운영 환경에서는 회사 프록시, 레지스트리, FQDN 기반 egress controller를 통해 더 세밀한 아웃바운드 통신 제어가 가능합니다. Calico, Cilium과 같은 CNI는 FQDN 기반 egress 정책을 지원합니다.

---

## 권한 격리: RBAC을 통한 역할 기반 접근 제어

### 팀 운영자 역할 (네임스페이스 범위 관리자)

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

팀 그룹에 역할 바인딩:

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

### 읽기 전용 역할

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

## 스토리지 격리

PersistentVolumeClaim(PVC)은 기본적으로 네임스페이스 경계를 벗어나지 않습니다. 필요에 따라 다음과 같은 추가 격리 수단을 고려할 수 있습니다:

- 팀 전용 StorageClass 제공
- ResourceQuota를 통한 스토리지 할당량 제한
- 백업/복구 도구(Velero 등)를 통한 네임스페이스 단위 스냅샷 관리
- Kyverno 정책을 통한 특정 StorageClass만 사용하도록 제한

---

## 노드 격리 (선택적): 테인트/톨러레이션과 노드 어피니티

민감한 워크로드나 비용 분리가 필요한 경우 전용 노드 풀을 구성할 수 있습니다.

```bash
# 노드에 테인트 추가
kubectl taint nodes <node-name> tenant=team-dev:NoSchedule
# 노드에 라벨 추가
kubectl label node <node-name> tenant=team-dev
```

워크로드에 톨러레이션과 노드 셀렉터 적용:

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

---

## 거버넌스 및 정책 강제: Kyverno 또는 Gatekeeper(OPA)

### 리소스 요청 및 제한 강제

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

### 허용된 이미지 레지스트리 제한

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

### 위험한 설정 차단

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

> Gatekeeper(OPA)를 사용하면 Rego 언어로 동일한 제약 조건을 구현할 수 있습니다. 조직의 표준과 감사 요구사항에 따라 적절한 정책 엔진을 선택하세요.

---

## 배포 자동화: Helm과 Kustomize 활용

### Helm을 통한 테넌트 변수화

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

설치 명령:

```bash
helm install tenant-team-dev ./tenant-kit -n kube-system
```

### Kustomize를 통한 베이스와 오버레이 구조

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

각 팀별 오버레이에서 네임스페이스, 라벨, 리소스 할당량 등을 개별적으로 설정할 수 있습니다.

---

## 관측, 감사 및 비용 관리

### 라벨 표준화

모든 리소스에 공통 라벨을 적용하여 분류 체계를 구축합니다.

```yaml
metadata:
  labels:
    tenant/name: team-dev
    app.kubernetes.io/part-of: web-suite
```

Kyverno의 mutate 정책을 사용하여 라벨을 자동으로 주입할 수도 있습니다.

### 모니터링 스택 통합

- **Prometheus**: 네임스페이스별 리소스 사용량 및 애플리케이션 메트릭 수집
- **Grafana**: 팀별 대시보드 폴더 구분으로 가시성 확보
- **Loki**: `{namespace="team-dev"}`와 같은 쿼리로 로그 필터링
- **분산 추적**: Jaeger나 Tempo를 통한 팀별 서비스맵 제공

### 비용 관리 및 쇼백(Showback)

- 리소스 사용량 메트릭: `container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`
- 스토리지 사용량: PVC 용량 및 CSI 메트릭
- 네임스페이스 라벨 기반 비용 대시보드 구성

---

## 신뢰성 확보: PDB, HPA 및 백업 전략

### PodDisruptionBudget (PDB)

노드 드레인이나 유지보수 시에도 가용성을 유지합니다.

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

### Horizontal Pod Autoscaler (HPA)

팀별로 독립적인 스케일링 정책을 설정합니다.

### 백업 및 복구 전략

네임스페이스 단위로 스냅샷을 생성하고 복구하는 계획을 수립합니다.

---

## 예제 애플리케이션으로 격리 검증

### 테스트 배포

```bash
# team-dev 네임스페이스에 배포
kubectl -n team-dev create deployment web --image=nginx
kubectl -n team-dev expose deployment web --port=80

# team-qa 네임스페이스에 배포
kubectl -n team-qa create deployment web --image=nginx
kubectl -n team-qa expose deployment web --port=80
```

### 통신 테스트

기본 거부 정책이 적용되었으므로, 다른 네임스페이스 간 통신은 차단되어야 합니다.

```bash
# team-qa 네임스페이스에서 team-dev 서비스 접근 시도
kubectl -n team-qa run tester --image=busybox:1.36 -it --rm -- sh
wget -S -O- http://web.team-dev.svc.cluster.local   # 실패해야 정상

# 동일 네임스페이스 내 통신 테스트
kubectl -n team-dev run tester --image=busybox:1.36 -it --rm -- sh
wget -S -O- http://web.team-dev.svc.cluster.local   # 성공해야 정상
```

---

## 운영 모범 사례

1. **네임스페이스 생성 시 표준화**: PSA 라벨과 표준 라벨을 반드시 포함합니다.
2. **리소스 제한 필수**: ResourceQuota와 LimitRange 없이는 배포를 허용하지 않습니다.
3. **네트워크 기본 거부**: 모든 네트워크 정책은 기본 거부부터 시작하여 필요한 통신만 허용합니다.
4. **최소 권한 원칙**: Role/RoleBinding을 사용하며, ClusterRole 사용은 신중하게 검토합니다.
5. **이미지 보안**: 이미지 출처 검증과 서명 확인을 위한 정책을 구현합니다.
6. **비밀 정보 관리**: KMS, Sealed-Secrets, External-Secrets 등을 통한 안전한 비밀 정보 관리
7. **접근 제어**: 각 팀에 전용 ServiceAccount와 네임스페이스 스코프 kubeconfig 제공
8. **변경 관리**: 모든 변경사항은 스테이징 환경에서 먼저 검증 후 프로덕션 적용
9. **감사 및 모니터링**: 감사 로그와 정책 이벤트 대시보드를 운영하여 이상 징후 탐지

---

## 트러블슈팅 가이드

| 증상 | 가능한 원인 | 해결 방안 |
|---|---|---|
| 다른 테넌트 네임스페이스에 접근 가능 | Default Deny 정책 누락, namespaceSelector 라벨 불일치 | 기본 거부 정책 추가, 네임스페이스 라벨 정합성 재확인 |
| 배포가 거부됨 | PSA restricted 정책 위반, Kyverno/Gatekeeper 정책 위반 | 이벤트 로그와 어드미션 거부 메시지 확인, Pod 스펙 수정 |
| CPU/메모리 요청이 적용되지 않음 | LimitRange 또는 Quota 미설정 | 네임스페이스 정책 확인, 템플릿 재적용 |
| 외부 통신 실패 | Egress 기본 거부 정책만 적용, DNS 허용 미구성 | DNS 및 필수 포트 화이트리스트 정책 추가 |
| 비용 대시보드에서 팀 구분 불가 | 라벨 표준 미적용 또는 불일치 | Kyverno mutate 정책으로 라벨 자동 주입 구현 |

---

## 통합 테넌트 키트 샘플

다음 매니페스트를 하나의 번들로 적용하여 테넌트 환경을 구성할 수 있습니다.

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

적용 명령:

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

Kubernetes에서 효과적인 멀티 테넌시 환경을 구축하기 위해서는 다음과 같은 원칙을 준수해야 합니다:

1. **네임스페이스 중심 설계**: 모든 테넌트 격리의 기본 단위는 네임스페이스입니다.
2. **방어적 보안 정책**: PSA, Quota, LimitRange, NetworkPolicy, RBAC을 조합하여 기본적인 보안 장치를 구축합니다.
3. **격리 수준 조정**: 필요에 따라 노드 풀 격리, 전용 스토리지 클래스, 정책 엔진을 활용하여 격리 수준을 강화합니다.
4. **자동화와 표준화**: Helm/Kustomize를 통한 일관된 배포, 라벨 표준화, 통합 모니터링으로 운영 효율성을 높입니다.

이 문서에서 제시한 매니페스트와 정책 템플릿을 조합하면, 단일 Kubernetes 클러스터에서 안전하고 예측 가능한 멀티 테넌시 환경을 체계적으로 구축하고 검증할 수 있습니다. 각 조직의 요구사항에 맞게 이러한 구성 요소를 조정하고 확장하여 최적의 멀티 테넌시 아키텍처를 구현하시기 바랍니다.