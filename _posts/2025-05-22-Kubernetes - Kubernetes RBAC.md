---
layout: post
title: Kubernetes - Kubernetes RBAC
date: 2025-05-22 19:20:23 +0900
category: Kubernetes
---
# Kubernetes RBAC (Role-Based Access Control) 완벽 가이드

Kubernetes는 기본적으로 **모든 사용자가 모든 리소스에 접근 가능한 구조**입니다.  
이를 보안상 제어하기 위해 **RBAC(Role-Based Access Control)**을 제공합니다.

RBAC는 사용자/서비스가 할 수 있는 행위(**권한**)를 정의하고,  
**정책에 따라 허용/거부**하는 방식입니다.

---

## ✅ RBAC 개념 요약

RBAC는 크게 네 가지 리소스로 구성됩니다:

| 리소스 | 설명 |
|--------|------|
| **Role** | 특정 네임스페이스 내 권한 정의 |
| **ClusterRole** | 클러스터 전역 또는 모든 네임스페이스에 대한 권한 정의 |
| **RoleBinding** | Role을 사용자/그룹/서비스어카운트에 연결 |
| **ClusterRoleBinding** | ClusterRole을 전역 대상에게 연결 |

---

## ✅ 동작 흐름

```text
[User/ServiceAccount] ──> [Role/ClusterRoleBinding] ──> [Role/ClusterRole] ──> [리소스 권한]
```

→ "누가", "어떤 역할을", "어떤 범위에서", "어떤 리소스에" 수행할 수 있는지를 설정

---

## ✅ Role 예제 (네임스페이스 내 권한 부여)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

→ `dev` 네임스페이스에서 `pods` 리소스의 조회 권한만 부여

---

## ✅ RoleBinding 예제 (Role을 사용자에게 바인딩)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-dev
  namespace: dev
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

→ `alice`는 `dev` 네임스페이스에서 `pod-reader` 역할 수행 가능

---

## ✅ ClusterRole 예제 (클러스터 전역 권한 부여)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

→ 클러스터 전역의 `nodes` 리소스를 읽을 수 있음

---

## ✅ ClusterRoleBinding 예제 (ClusterRole을 전역 사용자에게 바인딩)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
- kind: User
  name: bob
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

→ `bob`은 클러스터 전체에서 `nodes`를 조회 가능

---

## ✅ ServiceAccount에 권한 부여하기

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: app
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-sa-binding
  namespace: app
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: app
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

→ `my-app-sa`로 실행되는 Pod는 `app` 네임스페이스 내 `pods`를 조회할 수 있음

---

## ✅ 자주 사용하는 ClusterRole

Kubernetes는 기본적으로 여러 ClusterRole을 내장하고 있습니다:

| 이름 | 설명 |
|------|------|
| `cluster-admin` | 모든 권한 보유 |
| `admin` | 네임스페이스 수준의 전체 권한 |
| `edit` | 수정 가능, 권한 변경은 불가 |
| `view` | 읽기 전용 |
| `system:node` | 노드에서 실행되는 kubelet용 |

→ 커스텀 Role을 정의해도 되고, 이들을 재사용해도 됨

---

## ✅ 권한 확인 및 디버깅

### 1. 현재 유저가 어떤 권한을 가졌는지 확인

```bash
kubectl auth can-i get pods --as alice --namespace dev
```

→ `yes` 또는 `no`로 응답

### 2. 전체 권한 목록 확인

```bash
kubectl get role,clusterrole,rolebinding,clusterrolebinding --all-namespaces
```

---

## ✅ 실수 방지 체크리스트

- [ ] **Role vs ClusterRole 구분**  
  Role은 네임스페이스 한정, ClusterRole은 전역 가능
- [ ] **RoleBinding vs ClusterRoleBinding 구분**  
  ClusterRole을 특정 네임스페이스에 제한하려면 RoleBinding을 사용
- [ ] **ServiceAccount 대상 여부 확인**  
  Pod가 어떤 ServiceAccount로 실행되는지 반드시 명시
- [ ] **권한 최소화 원칙 적용**  
  `cluster-admin` 무분별한 사용 지양

---

## ✅ 예시: 읽기 전용 사용자를 위한 Role 생성

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: readonly
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
```

→ `readonly` Role을 이용해 감시용 사용자를 구성 가능

---

## ✅ 결론

| 개념 | 설명 |
|------|------|
| Role | 네임스페이스 내 권한 |
| ClusterRole | 클러스터 전체 권한 |
| RoleBinding | Role을 유저/SA에 연결 |
| ClusterRoleBinding | ClusterRole을 전역 사용자에 연결 |
| 최소 권한 원칙 | 보안상 필수 정책 |

> Kubernetes RBAC는 보안 및 운영 통제의 핵심 기능입니다.  
> 모든 권한 부여는 **명시적이고 최소한의 범위로 제한**하는 것이 좋습니다.

---

## ✅ 참고 링크

- [Kubernetes RBAC 공식 문서](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [kubectl auth can-i](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#auth)
- [Best Practices for Kubernetes RBAC](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-using-rbac-effectively)