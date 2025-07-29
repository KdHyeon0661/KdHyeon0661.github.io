---
layout: post
title: Kubernetes - Multi-Tenant
date: 2025-05-11 22:20:23 +0900
category: Kubernetes
---
# Multi-Tenant 환경에서 네임스페이스 분리 + 정책 구성

Kubernetes는 기본적으로 **모든 자원이 공유된 환경**입니다.  
하지만, 조직 내 여러 팀(개발/QA/운영) 또는 여러 고객을 **분리된 환경에서 운영**하려면  
**Multi-Tenant(다중 테넌시)** 구성이 필요합니다.

---

## ✅ Multi-Tenancy란?

- **복수의 사용자(또는 팀, 서비스)**가 한 클러스터를 공유하면서
- **서로 격리된 논리적/보안적 공간에서 작업**하는 구조입니다.

Kubernetes에서는 보통 **네임스페이스(Namespace) 단위로 테넌시 분리**를 구현합니다.

---

## ✅ 주요 고려 요소

| 항목 | 설명 |
|------|------|
| 리소스 격리 | CPU, 메모리, 스토리지 제한 |
| 네트워크 격리 | Pod 간 통신 제한 |
| 권한 격리 | 사용자별 접근 제어(RBAC) |
| 이름 충돌 방지 | 동일한 리소스 이름 허용 (`ns-a/web`, `ns-b/web`) |
| 감사 로그 | 누가 무엇을 했는지 추적 가능해야 함 |

---

## ✅ 1. 네임스페이스 기반 분리

### 예시: dev팀, qa팀, 운영팀 분리

```bash
kubectl create ns team-dev
kubectl create ns team-qa
kubectl create ns team-prod
```

---

## ✅ 2. 리소스 제한: ResourceQuota & LimitRange

### team-dev에 리소스 제한 설정

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: team-dev
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 1Gi
    limits.cpu: "4"
    limits.memory: 2Gi
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: team-dev
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "200m"
      memory: "256Mi"
    type: Container
```

→ 팀마다 리소스 할당량을 제한하여 **오버유즈 방지**

---

## ✅ 3. 접근 권한 분리: RBAC

예시: dev팀은 자신의 네임스페이스만 접근 가능하게

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-dev
  name: dev-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "create", "delete", "update"]
```

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dev-binding
  namespace: team-dev
subjects:
- kind: User
  name: dev-user1
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
```

→ dev-user1은 **team-dev 네임스페이스에만 권한**

---

## ✅ 4. 네트워크 격리: NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-other-namespaces
  namespace: team-dev
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: team-dev
```

- `team-dev` 내부에서는 통신 가능
- 외부 네임스페이스에서 접근 차단

> `kubectl label namespace team-dev name=team-dev` 로 라벨 설정 필요

---

## ✅ 5. ConfigMap, Secret 분리

네임스페이스마다 설정 분리 가능:

- `team-dev/configmap-dev`
- `team-qa/configmap-qa`

```bash
kubectl create configmap app-config --from-literal=key=value -n team-dev
```

→ 서로 다른 팀이 **설정파일 충돌 없이 운영 가능**

---

## ✅ 6. Storage 분리 (선택)

- `PersistentVolumeClaim`도 네임스페이스 단위로 격리
- StorageClass 또는 PVC 이름 충돌 없음
- CSI 기반 동적 프로비저닝 지원 시 효율적

---

## ✅ 7. 오브젝트 명명 전략

팀별 prefix 규칙 도입

| 팀 | 리소스 명명 예시 |
|----|------------------|
| dev | `web-dev`, `api-dev` |
| qa | `web-qa`, `db-qa` |

---

## ✅ 8. Helm 차트 격리

Helm 설치 시 네임스페이스 지정 가능:

```bash
helm install myapp ./chart -n team-dev
```

→ 동일한 차트를 여러 팀이 독립적으로 설치 가능

---

## ✅ 9. 관측 및 감사

- `kubectl audit` 로그를 통해 **사용자 활동 추적**
- Prometheus / Grafana를 **namespace별로 분리 시각화**
- 로그도 Loki 등으로 **tenant별 필터링 가능**

---

## ✅ 결론

| 구성 요소 | 격리 수단 |
|-----------|------------|
| 네트워크 | NetworkPolicy |
| 리소스(CPU/Memory) | ResourceQuota, LimitRange |
| 접근 권한 | Role / RoleBinding (RBAC) |
| 설정 정보 | Namespace 단위 ConfigMap / Secret |
| 배포 도구 | Helm + 네임스페이스 |
| 스토리지 | PVC, StorageClass 격리 |

> Kubernetes에서 Multi-Tenancy는 **네임스페이스 단위 격리**가 기본이며,  
> 이를 기반으로 보안, 리소스, 트래픽 등을 점진적으로 통제할 수 있습니다.