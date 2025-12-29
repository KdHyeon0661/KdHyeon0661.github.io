---
layout: post
title: Kubernetes - Kubernetes RBAC
date: 2025-05-22 19:20:23 +0900
category: Kubernetes
---
# Kubernetes RBAC (Role-Based Access Control) 완전 가이드

## RBAC 아키텍처와 개념적 이해

Kubernetes RBAC(Role-Based Access Control)는 클러스터 내 리소스에 대한 접근을 제어하는 표준 인가(Authorization) 시스템입니다. RBAC는 "누가 무엇을 할 수 있는지"를 명확하게 정의하여 보안과 거버넌스를 강화합니다.

```
[인증(Authentication)]  OIDC/SAML/x509/서비스어카운트 토큰
        │ (사용자/그룹/서비스어카운트, 추가 속성)
        ▼
[인가(Authorization)=RBAC]  (Role/ClusterRole) ⟷ (RoleBinding/ClusterRoleBinding)
        │
        ▼
[어드미션 컨트롤(Admission)]  (예: Gatekeeper/PSA/LimitRanger 등)
```

RBAC 시스템의 핵심은 세 가지 질문에 답하는 것입니다:
1. **누가** 접근하는가? (사용자, 그룹, 서비스어카운트)
2. **어디에서** 접근하는가? (특정 네임스페이스 또는 클러스터 전체)
3. **무엇을** 할 수 있는가? (리소스에 대한 특정 동작)

RBAC는 **허용 목록(Allow-list)** 방식으로 동작합니다. 명시적으로 허용된 권한만 부여되며, 매칭되는 규칙이 없는 모든 요청은 자동으로 거부됩니다.

---

## RBAC 구성 요소: 네 가지 핵심 리소스

### Role과 ClusterRole: 권한 정의
**Role**은 특정 네임스페이스 내에서의 권한을 정의합니다. 예를 들어 "development" 네임스페이스에서 Pod를 읽을 수 있는 권한을 정의할 수 있습니다.

**ClusterRole**은 클러스터 전체에 적용되는 권한을 정의하거나, 모든 네임스페이스에서 사용할 수 있는 역할 템플릿으로 작동합니다. ClusterRole은 네임스페이스에 속하지 않는 리소스(노드, PersistentVolume 등)에 대한 권한도 정의할 수 있습니다.

### RoleBinding과 ClusterRoleBinding: 권한 할당
**RoleBinding**은 Role이나 ClusterRole을 특정 네임스페이스의 사용자, 그룹, 또는 서비스어카운트에 연결합니다. 흥미롭게도 RoleBinding은 ClusterRole을 네임스페이스 범위로 제한하여 사용할 수 있습니다.

**ClusterRoleBinding**은 ClusterRole을 클러스터 전체의 주체에 연결합니다. 이는 모든 네임스페이스에 영향을 미칩니다.

### 주요 차이점 요약

| 리소스 | 적용 범위 | 주요 용도 |
|---|---|---|
| **Role** | 단일 네임스페이스 | 네임스페이스 스코프 리소스 권한 정의 |
| **ClusterRole** | 클러스터 전체 또는 템플릿 | 전역 리소스 권한 또는 재사용 가능한 역할 템플릿 |
| **RoleBinding** | 단일 네임스페이스 | 네임스페이스 내 권한 할당 |
| **ClusterRoleBinding** | 클러스터 전체 | 전역 권한 할당 |

**핵심 개념**: ClusterRole은 권한의 **정의**이고, Binding은 이를 **어디에 적용할지** 결정합니다. ClusterRole을 특정 네임스페이스로 제한하려면 RoleBinding을 사용하고, 클러스터 전체에 적용하려면 ClusterRoleBinding을 사용합니다.

---

## 기본 예제: 실습을 통한 이해

### 네임스페이스 읽기 전용 역할 생성
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: read-only-role
rules:
- apiGroups: [""]  # 코어 API 그룹
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
```

이 Role은 "development" 네임스페이스에서 Pod, 서비스, ConfigMap을 읽을 수 있는 권한을 정의합니다.

### 사용자에게 역할 할당
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: development
  name: developer-access
subjects:
- kind: User
  name: alice@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: read-only-role
  apiGroup: rbac.authorization.k8s.io
```

이 RoleBinding은 "alice@example.com" 사용자에게 위에서 정의한 읽기 전용 역할을 할당합니다.

### ClusterRole을 네임스페이스에 제한하여 사용
Kubernetes에는 미리 정의된 여러 ClusterRole이 있습니다. 이를 특정 네임스페이스에 제한하여 사용할 수 있습니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: team-alpha
  name: team-developers
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit  # Kubernetes 기본 ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

이 구성은 "developers" 그룹의 멤버들에게 "team-alpha" 네임스페이스에서 "edit" ClusterRole의 권한을 부여합니다.

---

## 서비스어카운트와의 통합

서비스어카운트는 Pod 내에서 실행되는 애플리케이션에 권한을 부여하는 데 사용됩니다.

### 서비스어카운트 생성 및 권한 부여
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: application-sa
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-config-access
  namespace: production
subjects:
- kind: ServiceAccount
  name: application-sa
  namespace: production
roleRef:
  kind: Role
  name: config-reader
  apiGroup: rbac.authorization.k8s.io
```

이 구성은 "production" 네임스페이스의 "application-sa" 서비스어카운트가 ConfigMap과 Secret을 읽을 수 있는 권한을 부여합니다.

**보안 고려사항**: Secret 읽기 권한은 신중하게 관리해야 합니다. 이 권한은 자격 증명 유출과 같은 권한 상승 위험을 초래할 수 있으므로 실제로 필요한 최소한의 범위로만 제한해야 합니다.

---

## 실전 패턴과 활용 시나리오

### 로그 조회 전용 역할
운영팀이 로그를 확인할 수 있지만 리소스를 수정하지 못하도록 제한하는 경우:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: operations
  name: log-viewer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### CI/CD 배포 전용 계정
CI/CD 시스템이 애플리케이션을 배포하고 업데이트할 수 있도록 권한을 제한적으로 부여:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: cicd-deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "patch", "update"]
- apiGroups: ["apps"]
  resources: ["deployments/scale", "statefulsets/scale"]
  verbs: ["get", "update"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

이 역할은 배포 업데이트와 스케일링을 허용하지만, 다른 리소스 생성이나 삭제는 허용하지 않습니다.

### 네임스페이스 관리자 역할
팀 리더가 자신의 네임스페이스를 관리할 수 있도록 권한 부여:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: team-beta
  name: namespace-admin
subjects:
- kind: Group
  name: team-beta-leads
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin  # Kubernetes 기본 관리자 역할
  apiGroup: rbac.authorization.k8s.io
```

### 특정 리소스에 대한 세부 권한 제어
특정 ConfigMap만 접근할 수 있도록 제한:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: secure-app
  name: restricted-config-access
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-configuration", "feature-flags"]
  verbs: ["get"]
```

`resourceNames` 필드를 사용하면 특정 이름의 리소스에만 접근을 허용할 수 있습니다.

---

## 고급 RBAC 패턴

### Aggregated ClusterRoles
여러 ClusterRole을 하나로 통합하여 관리할 수 있습니다. 이는 CRD(Custom Resource Definitions)를 설치할 때 특히 유용합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: custom-resources-viewer
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: ["monitoring.example.com"]
  resources: ["alerts", "dashboards"]
  verbs: ["get", "list", "watch"]
```

`rbac.authorization.k8s.io/aggregate-to-view: "true"` 레이블이 있는 ClusterRole은 자동으로 기본 `view` ClusterRole에 통합됩니다. 따라서 `view` 역할을 가진 사용자는 자동으로 알림과 대시보드 리소스도 볼 수 있게 됩니다.

### 비리소스 URL 권한
Kubernetes API의 특정 엔드포인트에 대한 접근을 제어할 수 있습니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: health-metrics-viewer
rules:
- nonResourceURLs: ["/healthz", "/metrics", "/readyz"]
  verbs: ["get"]
```

이 ClusterRole은 클러스터 상태 체크와 메트릭 엔드포인트에 대한 읽기 접근을 허용합니다. `/metrics` 엔드포인트는 민감한 정보를 노출할 수 있으므로 신중하게 관리해야 합니다.

---

## 권한 에스컬레이션 방지

RBAC 설계 시 권한 상승 가능성을 최소화하는 것이 중요합니다. 다음 원칙을 준수하세요:

1. **최소 권한 원칙**: 필요한 최소한의 권한만 부여합니다.
2. **와일드카드 제한**: `verbs: ["*"]` 또는 `resources: ["*"]`와 같은 광범위한 권한은 지양합니다.
3. **민감한 리소스 관리**: Secret과 RBAC 오브젝트 자체에 대한 접근 권한을 엄격히 제한합니다.
4. **실행 권한 통제**: `pods/exec`와 `pods/portforward` 권한은 사실상 노드와 네트워크에 대한 접근을 제공하므로 신중하게 관리합니다.
5. **정기적인 권한 검토**: 권한 할당을 정기적으로 검토하고 불필요한 권한을 제거합니다.

---

## 다중 테넌시 환경에서의 RBAC 패턴

| 요구사항 | 권장 패턴 |
|---|---|
| 팀별 네임스페이스 격리 | 각 네임스페이스에 RoleBinding으로 `edit`/`view` 역할 할당 |
| 공통 모니터링 접근 | ClusterRole 생성 후 각 네임스페이스에 RoleBinding으로 할당 |
| 플랫폼 운영팀 | ClusterRole과 ClusterRoleBinding으로 전역 권한 부여 |
| 완전한 테넌트 격리 | 네임스페이스 + NetworkPolicy + ResourceQuota + RBAC 조합 |

---

## RBAC 검증 및 감사

### 실시간 권한 확인
```bash
# 특정 사용자가 Pod를 볼 수 있는지 확인
kubectl auth can-i get pods --as alice@example.com -n development

# 서비스어카운트가 Deployment를 생성할 수 있는지 확인
kubectl auth can-i create deployments --as system:serviceaccount:production:ci-sa -n production

# API 엔드포인트 접근 권한 확인
kubectl auth can-i get --raw /api/v1/nodes --as bob@example.com
```

### RBAC 구성 검사
```bash
# 클러스터 전체 RBAC 상태 확인
kubectl get clusterrole,clusterrolebinding

# 모든 네임스페이스의 Role과 RoleBinding 확인
kubectl get role,rolebinding -A

# 특정 그룹이 할당된 ClusterRoleBinding 찾기
kubectl get clusterrolebinding -o json | jq '.items[] | select(.subjects[]?.name=="developers") | .metadata.name'
```

### 권한 분석 스크립트
```bash
# Secret 접근 권한이 있는 ClusterRole 찾기
kubectl get clusterrole -o json | jq -r '
  .items[] | select(.rules[]? | (.resources? // []) | index("secrets")) |
  .metadata.name'

# RBAC 오브젝트 수정 권한이 있는 역할 찾기
kubectl get clusterrole -o json | jq -r '
  .items[] | select(.rules[]? | (.resources? // []) | 
    inside(["roles", "rolebindings", "clusterroles", "clusterrolebindings"])) |
  .metadata.name'
```

---

## 일반적인 문제 해결

| 증상 | 일반적인 원인 | 해결 방법 |
|---|---|---|
| "Forbidden" 오류 | Binding이 없거나 네임스페이스 불일치 | RoleBinding 존재 여부 및 네임스페이스 확인 |
| 클러스터 전체 권한이 네임스페이스로 제한됨 | ClusterRoleBinding 대신 RoleBinding 사용 | 의도에 맞게 Binding 유형 선택 |
| exec/port-forward가 의도치 않게 허용됨 | 서브리소스 권한이 포함됨 | `pods/exec`, `pods/portforward` 권한 명시적 제거 |
| 불필요한 Secret 접근 가능 | 과도한 읽기 권한 | 필요한 Secret만 접근 허용 또는 접근 제한 |
| CRD 접근 불가 | apiGroups 또는 리소스 이름 오류 | `kubectl api-resources`로 정확한 이름 확인 |

---

## 실무용 역할 템플릿 카탈로그

### 로그 전용 뷰어
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: operations
  name: log-viewer-only
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### 확장 읽기 전용 역할
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: monitoring
  name: extended-read-only
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "events"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch"]
```

### 최소한의 배포 권한
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: minimal-deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "patch", "update"]
- apiGroups: ["apps"]
  resources: ["deployments/scale", "statefulsets/scale"]
  verbs: ["get", "update"]
```

### CRD 관리자 역할
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: custom-apps
  name: custom-resource-manager
rules:
- apiGroups: ["custom.example.com"]
  resources: ["widgets", "gadgets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

---

## 운영 모범 사례

### 정책 기반 접근 관리
- **그룹 기반 권한 할당**: 개별 사용자 대신 그룹에 권한을 할당하여 사용자 변경 시 구성을 유지합니다.
- **역할 카탈로그 운영**: 표준화된 역할 세트를 유지 관리하고 재사용합니다.
- **코드 리뷰 게이트**: 모든 RBAC 변경사항을 PR과 코드 리뷰를 통해 검토합니다.

### 보안 강화
- **정기적 권한 감사**: 사용되지 않는 권한을 식별하고 제거합니다.
- **감사 로깅**: 모든 권한 관련 이벤트를 로깅하고 모니터링합니다.
- **자격 증명 수명 주기 관리**: 서비스어카운트 토큰을 정기적으로 갱신합니다.

### 자동화 및 통합
- **GitOps 통합**: RBAC 구성을 Git에서 관리하고 자동으로 동기화합니다.
- **CI/CD 파이프라인 통합**: 배포 파이프라인에 필요한 최소 권한만 할당합니다.
- **ID 공급자 연동**: 기업 ID 시스템(OIDC, SAML)과 통합하여 중앙 집중식 사용자 관리.

---

## 종합 검증 스크립트 예제

```bash
#!/bin/bash
# RBAC 상태 검증 스크립트

echo "=== 클러스터 RBAC 상태 검증 ==="
echo

# 1. 기본 RBAC 오브젝트 상태
echo "1. ClusterRole 및 ClusterRoleBinding 상태:"
kubectl get clusterrole,clusterrolebinding

echo
echo "2. 네임스페이스별 Role 및 RoleBinding 상태:"
kubectl get role,rolebinding -A --no-headers | wc -l | xargs echo "총 개수:"

echo
echo "3. 민감한 권한이 있는 ClusterRole 확인:"
echo "   - Secret 접근 권한:"
kubectl get clusterrole -o json | jq -r '
  .items[] | select(.rules[]? | (.resources? // []) | index("secrets")) |
  "    * " + .metadata.name' | sort

echo
echo "   - RBAC 오브젝트 수정 권한:"
kubectl get clusterrole -o json | jq -r '
  .items[] | select(.rules[]? | (.resources? // []) | 
    inside(["roles", "rolebindings", "clusterroles", "clusterrolebindings"])) |
  "    * " + .metadata.name' | sort

echo
echo "4. 일반적인 권한 테스트:"
echo "   - 익명 사용자 Pod 읽기:"
kubectl auth can-i get pods --as system:anonymous 2>/dev/null || echo "    실패"

echo "   - 기본 서비스어카운트 Secret 생성:"
kubectl auth can-i create secrets --as system:serviceaccount:default:default -n default 2>/dev/null || echo "    실패"

echo
echo "=== 검증 완료 ==="
```

---

## 결론

Kubernetes RBAC는 클러스터 보안과 거버넌스의 핵심 구성 요소로서, 세분화된 접근 제어를 통해 안전한 다중 사용자 환경을 구축할 수 있게 해줍니다. 효과적인 RBAC 관리를 위해서는 다음과 같은 원칙을 준수해야 합니다:

1. **최소 권한 원칙의 엄격한 적용**: 각 사용자와 서비스에 필요한 최소한의 권한만 부여합니다.
2. **명확한 역할 정의와 범위 이해**: Role/ClusterRole과 Binding의 관계를 정확히 이해하고 적절히 활용합니다.
3. **서브리소스 권한의 신중한 관리**: 로그, 실행, 포트 포워딩과 같은 민감한 작업에 대한 접근을 엄격히 제어합니다.
4. **체계적인 권한 검증과 감사**: 정기적인 권한 검토와 감사 로깅을 통해 권한 남용을 방지합니다.
5. **자동화와 표준화**: 역할 템플릿 카탈로그와 자동화 도구를 활용하여 일관된 권한 관리를 구현합니다.

RBAC는 한 번 설정하면 방치되는 정책이 아니라, 지속적인 관리와 개선이 필요한 살아있는 시스템입니다. 조직의 변화와 보안 요구사항에 맞춰 RBAC 정책을 정기적으로 검토하고 조정하는 것이 장기적인 성공의 열쇠입니다.

이 가이드의 개념, 패턴, 모범 사례를 적용하여 안전하고 관리 가능한 Kubernetes 환경을 구축하시기 바랍니다. 적절하게 구성된 RBAC는 보안 강화뿐만 아니라 운영 효율성과 규정 준수 요구사항 충족에도 크게 기여할 것입니다.