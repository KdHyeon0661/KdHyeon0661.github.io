---
layout: post
title: Kubernetes - Kubernetes RBAC
date: 2025-05-22 19:20:23 +0900
category: Kubernetes
---
# Kubernetes RBAC (Role-Based Access Control)

## 0) 빅픽처: 인증→인가 파이프라인

```
[인증(Authentication)]  OIDC/SAML/x509/SA 토큰
        │ (user/group/serviceaccount, extras)
        ▼
[인가(Authorization)=RBAC]  (Role/ClusterRole) ⟷ (RoleBinding/ClusterRoleBinding)
        │
        ▼
[어드미션(Admission)]  (예: Gatekeeper/PSA/LimitRanger 등)
```

- **누가**(user/group/SA)가 **어떤 네임스페이스/전역 범위**에서 **무엇(resource/verb)**을 할 수 있는지 RBAC로 통제.
- RBAC는 **허용만 정의**합니다(Allow-list). 매칭이 없으면 **거부**됩니다.

---

## 1) 핵심 리소스 4종 요약 (정확한 의미)

| 리소스 | 범위 | 목적 | 자주 혼동되는 포인트 |
|---|---|---|---|
| **Role** | Namespace | 해당 네임스페이스 안 리소스 권한 정의 | `resources`/`verbs`는 **NS스코프 리소스** 중심 |
| **ClusterRole** | Cluster-wide | 클러스터 전역 또는 모든 NS에 적용 가능한 역할 | **비NS 리소스(nodes, namespaces)**, **CRD**, `nonResourceURLs` 포함 가능 |
| **RoleBinding** | Namespace | 한 NS에서 Role(또는 ClusterRole)을 **주체**(user/group/SA)에 바인딩 | ClusterRole을 **NS 한정**으로 바인딩할 때도 **RoleBinding** 사용 |
| **ClusterRoleBinding** | Cluster-wide | ClusterRole을 **전역 범위**로 바인딩 | 모든 NS/비NS에 영향 |

> **핵심 규칙**: **ClusterRole**은 전역 역할의 **정의**이고, 그걸 **어디에 적용하느냐**는 Binding이 결정합니다.  
> 전역으로 적용하고 싶으면 **ClusterRoleBinding**, 특정 NS로 제한하고 싶으면 **RoleBinding**을 쓰세요.

---

## 2) 권한 평가 모델(개념 수식)

RBAC의 허용 여부는 다음 **실현가능한 논리식**으로 이해할 수 있습니다.

사용자 \(u\)가 네임스페이스 \(n\)에서 리소스 \(r\)에 대해 동사 \(v\)를 수행하려면:

$$
\exists\ \text{Binding}\ B,\ \exists\ \text{Role}\ R:\
\bigl(B.\text{subject} \ni u \lor \text{Group}(u)\bigr)\ \land\
B.\text{roleRef}=R\ \land\
\bigl(R.\text{rules} \ni (r, v)\bigr)\ \land\
\text{scope\_ok}(B, n)
$$

- `scope_ok`는 RoleBinding/ClusterRoleBinding의 **범위 일치**(NS/전역)를 뜻합니다.

---

## 3) Role/Binding 최소 단위 예제(기본기)

### 3.1 읽기 전용 Role (NS 한정)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: readonly
rules:
- apiGroups: [""]
  resources: ["pods","services","endpoints","configmaps"]
  verbs: ["get","list","watch"]
```

### 3.2 사용자 alice를 dev NS에 바인딩

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: dev
  name: readonly-binding
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: readonly
  apiGroup: rbac.authorization.k8s.io
```

---

## 4) ClusterRole을 네임스페이스 한정으로 쓰기

> **패턴**: 표준 `view`/`edit`/`admin` ClusterRole을 **RoleBinding**으로 특정 NS에만 제한.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: team-a
  name: team-a-developers
subjects:
- kind: Group
  name: devs
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit           # ClusterRole이지만 NS 한정으로 효력
  apiGroup: rbac.authorization.k8s.io
```

---

## 5) ServiceAccount(워크로드 아이덴티티) 권한 부여

### 5.1 SA 생성 + RoleBinding

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: app
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-config-reader
  namespace: app
rules:
- apiGroups: [""]
  resources: ["configmaps","secrets"]
  verbs: ["get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-config-reader-binding
  namespace: app
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: app
roleRef:
  kind: Role
  name: app-config-reader
  apiGroup: rbac.authorization.k8s.io
```

> **주의**: Secret 읽기 권한은 **권한 에스컬레이션 위험**(크리덴셜 유출) — 정말 필요한 최소 범위로만.

---

## 6) 흔한 과제와 실전 패턴

### 6.1 “로그는 보되, 리소스 수정은 금지”
- `pods/log` 서브리소스에 대한 `get` 허용, 그 외는 read-only.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: logs-only
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### 6.2 “exec/port-forward 금지”
- 서브리소스 `pods/exec`, `pods/portforward`를 **포함하지 말 것**(기본적으로 허용되지 않음).
- 만약 `create`가 들어가면 `exec`을 우연히 열 수도 있습니다. (항상 **서브리소스 명시**)

### 6.3 “네임스페이스 관리자(팀장)”
- 네임스페이스 내 **대부분**의 권한 부여(권한 부여 자체는 제한).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: team-a
  name: team-a-admin
subjects:
- kind: Group
  name: team-a-leads
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin     # NS 관리용 표준 역할
  apiGroup: rbac.authorization.k8s.io
```

### 6.4 “CI/CD 배포 전용 계정(이미지 배포, 롤링)”
- `deployments`/`statefulsets`의 `get/list/watch/patch/update` + `rollout`에 필요한 동사.
- `secrets`/`configmaps`는 **읽기만** 또는 별도 SA로 분리.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: prod
  name: cicd-deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments","statefulsets","replicasets"]
  verbs: ["get","list","watch","patch","update"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","list","watch"]
- apiGroups: ["apps"]
  resources: ["deployments/scale"]
  verbs: ["get","update"]
```

### 6.5 “읽기는 전역으로, 쓰기는 자기 NS만”
- `ClusterRoleBinding(view)` + 각 NS에 `RoleBinding(edit)`의 **조합**.

---

## 7) Aggregated ClusterRole (확장형 역할)

CRD 설치 시, 운영자는 **라벨 기반 집계**로 역할을 자동 확장할 수 있습니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: custom-resources-view
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: ["example.com"]
  resources: ["widgets"]
  verbs: ["get","list","watch"]
```

> 이렇게 라벨을 붙이면 **표준 `view` 역할**에 자동 집계되어, `view`를 가진 대상이 `widgets`도 조회 가능.

---

## 8) CRD/리소스 세부 권한(서브리소스/리소스네임)

- **서브리소스**: `"deployments/scale"`, `"pods/log"`, `"pods/exec"` 등은 **별개 리소스**로 취급.
- **resourceNames**: 특정 **이름**으로 제한 가능.

```yaml
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config"]   # 특정 CM만
  verbs: ["get"]
```

---

## 9) 비자원 URL(nonResourceURLs) — 헬스/메트릭 엔드포인트

클러스터 전역의 API 경로(예: `/healthz`, `/metrics`):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nonres-metrics
rules:
- nonResourceURLs: ["/metrics","/healthz"]
  verbs: ["get"]
```

> **주의**: `/metrics`는 민감할 수 있어 **최소 주체**에게만.

---

## 10) 권한 에스컬레이션 방지 가이드

- `*`(와일드카드) **지양**: 특히 `verbs: ["*"]`, `resources: ["*"]`.
- `secrets` 읽기(특히 전역) 남발 금지.  
- `roles`/`rolebindings`/`clusterroles`/`clusterrolebindings` **수정 권한**은 곧 **권한 에스컬레이션** 가능성.
- `pods/exec`/`pods/portforward` 접근은 사실상 **노드/네트워크 경유 권한**. 운영계정에서 엄격 관리.
- 상위 시스템(SCM/CI)의 토큰/SA 키 로테이션과 짧은 TTL(투명 JWT/Bound SA Token) 적용.

---

## 11) 멀티테넌시 RBAC 패턴(요약)

| 요구 | 패턴 |
|---|---|
| 팀 경계(네임스페이스 단위) | 각 NS에 `edit`/`view`를 RoleBinding, 전역은 제한 |
| 공용 관측 권한 | ClusterRole(view-logs) + RoleBinding(각 NS) |
| 플랫폼 운영팀 | ClusterRole(노드/스토리지/네임스페이스 관리) + ClusterRoleBinding |
| 고객별 테넌트 | 네임스페이스 ISO + `NetworkPolicy` + `ResourceQuota` + RBAC |

---

## 12) 점검/검증/감사

### 12.1 능동 질의: `kubectl auth can-i`

```bash
kubectl auth can-i get pods --as alice -n dev
kubectl auth can-i create deployments --as system:serviceaccount:prod:ci-sa -n prod
kubectl auth can-i get --raw /api/v1/nodes --as bob
```

### 12.2 전체 맵 훑기

```bash
kubectl get role,rolebinding -A
kubectl get clusterrole,clusterrolebinding
```

### 12.3 특정 주체의 바인딩 찾기

```bash
# ClusterRoleBinding에서 Group 'devs'가 어디 묶였는지
kubectl get clusterrolebinding -o json | jq '.items[] | select(.subjects[]?.name=="devs") | .metadata.name'

# NS별 RoleBinding에서 SA app:app-sa가 묶인 곳
kubectl get rolebinding -A -o json | jq '.items[] | select(.subjects[]?.kind=="ServiceAccount" and .subjects[]?.name=="app-sa" and .subjects[]?.namespace=="app") | .metadata.namespace + "/" + .metadata.name'
```

### 12.4 감사 로깅(Audit) 포인트
- **승인된/거부된** 요청 기록(누가/언제/무엇/어디서).
- 민감 리소스(`secrets`, `configmaps`, RBAC 오브젝트) 접근은 **고레벨**로.

---

## 13) 트러블슈팅 테이블

| 증상 | 흔한 원인 | 해결 |
|---|---|---|
| `Forbidden` (User “X” cannot get resource Y) | Binding 없음/NS 불일치 | RoleBinding 존재/네임스페이스 확인 |
| 전역 읽기 의도인데 NS만 됨 | ClusterRoleBinding 누락 | ClusterRoleBinding 생성 |
| ClusterRole이 NS 제한 안 됨 | ClusterRoleBinding 사용 | **RoleBinding**으로 NS 제한 |
| exec 가능해버림 | `pods/exec` 허용 포함 | 서브리소스 권한 제거 |
| Secrets가 보임 | 과한 `get` 허용 | `secrets` 제외 또는 특정 `resourceNames`만 허용 |
| CRD 권한 안 됨 | apiGroups/리소스명 오탈자 | `kubectl api-resources`로 정확한 이름 확인 |

---

## 14) 역할 카탈로그(실무형 템플릿)

### 14.1 View-Logs Only (NS)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: ops
  name: view-logs-only
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### 14.2 Read-Only + Events + Describe (리소스 폭 넓게, 변경 금지)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: qa
  name: readonly-extended
rules:
- apiGroups: [""]
  resources: ["pods","services","endpoints","configmaps","secrets"]
  verbs: ["get","list","watch"]      # secrets 포함 여부는 환경별 정책 결정
- apiGroups: ["apps"]
  resources: ["deployments","replicasets","daemonsets","statefulsets"]
  verbs: ["get","list","watch"]
- apiGroups: ["batch"]
  resources: ["jobs","cronjobs"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["list","watch"]
```

### 14.3 Deployer (변경 최소한, 보수적)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: staging
  name: deployer-min
rules:
- apiGroups: ["apps"]
  resources: ["deployments","statefulsets","replicasets"]
  verbs: ["get","list","watch","patch","update"]
- apiGroups: ["apps"]
  resources: ["deployments/scale"]
  verbs: ["get","update"]
```

### 14.4 Namespace Owner (팀 리더)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: team-x
  name: ns-owner
subjects:
- kind: Group
  name: team-x-leads
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

### 14.5 CRD 조회/수정(예: `widgets.example.com`)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: apps
  name: widgets-editor
rules:
- apiGroups: ["example.com"]
  resources: ["widgets"]
  verbs: ["get","list","watch","create","update","patch","delete"]
```

---

## 15) 자동화/리뷰 팁

- 모든 RBAC YAML은 **PR 게이트**(Code Review + 정책 스캔)로만 반영.
- **기본 세트(카탈로그)**에서 조합: `view-logs`, `readonly`, `deployer`…
- 바인딩 주체는 **그룹 중심**(IdP 그룹) → 사람 교체에도 YAML 불변.

---

## 16) 실전 점검 스크립트(샘플)

```bash
# 1) 전체 RBAC 오브젝트 요약
kubectl get clusterrole,clusterrolebinding
kubectl get role,rolebinding -A

# 2) 민감 권한 스팟체크(Secrets, RBAC 수정)
kubectl get clusterrole -o json | jq -r '
 .items[] | select(.rules[]? | (.resources? // []) | index("secrets")) |
 .metadata.name'
kubectl get clusterrole -o json | jq -r '
 .items[] | select(.rules[]? | (.resources? // []) | inside(["roles","rolebindings","clusterroles","clusterrolebindings"])) |
 .metadata.name'

# 3) 특정 사용자 권한 스모크
kubectl auth can-i get secrets --as alice -n prod
kubectl auth can-i create role --as alice -n prod
kubectl auth can-i get nodes --as alice            # 전역 리소스
```

---

## 17) 수학적 관점(선택)

RBAC의 규칙 집합을 \( \mathcal{R} \)라 하면, 특정 요청 \(q=(u,n,r,v)\)에 대한 허용 여부는:

$$
\text{allow}(q) = \bigvee_{(B,R) \in \mathcal{B}\times\mathcal{R}}
\Bigl( \underbrace{u \in \text{subjects}(B)}_{\text{사용자/그룹/SA 매치}}
\land \underbrace{\text{roleRef}(B)=R}_{\text{바인딩 연결}}
\land \underbrace{(r,v) \in \text{rules}(R)}_{\text{리소스/동사 매치}}
\land \underbrace{\text{scope\_ok}(B,n)}_{\text{범위 일치}} \Bigr)
$$

— **부합하는 한 쌍이라도 존재**하면 허용, 없으면 거부.

---

## 18) 베스트 프랙티스 요약 체크리스트

- [ ] **Least Privilege**: 필요한 리소스/동사만.
- [ ] **서브리소스 명시**: exec/portforward/log/scale는 별도.
- [ ] **ClusterRole은 정의, 범위는 Binding이 결정**.
- [ ] **ClusterRole을 NS 제한**하려면 **RoleBinding** 사용.
- [ ] `secrets`·RBAC 오브젝트 수정 권한은 **극소수**로.
- [ ] **그룹 기반 바인딩** + IdP 연동 → 사람 교체에도 코드 고정.
- [ ] **감사 로깅**/정기 점검(스캔·리포트).
- [ ] 템플릿 카탈로그 운영(표준 역할 재사용).

---

## 19) 결론

- RBAC는 쿠버네티스 **보안/거버넌스의 중심축**입니다.
- **정확한 스코프 모델(역할 vs 바인딩)**과 **서브리소스 인식**, **최소권한 설계**가 안정 운영의 관건.
- 본문 템플릿을 **역할 카탈로그**로 삼아 팀/환경별로 조합하세요.
- 점검 명령(`kubectl auth can-i`, JSON 스캔)과 감사 로깅으로 **지속 검증**하면 권한 스프로울을 예방할 수 있습니다.

---

## 참고
- Kubernetes RBAC 레퍼런스: <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>
- `kubectl auth can-i`: <https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#auth>
- Best Practices: <https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-using-rbac-effectively>