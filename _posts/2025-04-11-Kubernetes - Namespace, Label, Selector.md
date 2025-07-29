---
layout: post
title: Kubernetes - Namespace, Label, Selector
date: 2025-04-11 19:20:23 +0900
category: Kubernetes
---
# Kubernetes 핵심 개념: Namespace, Label, Selector

쿠버네티스 클러스터가 커지고 리소스가 많아질수록 이를 **논리적으로 분리하고 효율적으로 관리하는 기술**이 중요합니다.  
이때 핵심 개념이 바로 다음 세 가지입니다:

- `Namespace`: 클러스터 리소스의 논리적 분리
- `Label`: 리소스에 부착하는 key-value 메타데이터
- `Selector`: Label 기반 리소스 필터링

---

## ✅ 1. Namespace란?

Namespace는 **Kubernetes 리소스를 격리된 공간으로 분리**할 수 있는 논리적인 단위입니다.

### 🎯 목적
- 멀티팀/멀티프로젝트 환경에서 리소스를 분리
- 리소스 할당량 제한 (ResourceQuota)
- 네임 충돌 방지 (같은 이름의 Pod를 다른 namespace에서 생성 가능)

### 📌 기본 네임스페이스

| 이름 | 설명 |
|------|------|
| `default` | 기본 네임스페이스 |
| `kube-system` | Kubernetes 시스템 컴포넌트 (kube-dns 등) |
| `kube-public` | 공개 리소스 (클러스터 정보 등) |
| `kube-node-lease` | 노드 상태 heartbeat용 |

### 🛠 예제: Namespace 생성

```bash
kubectl create namespace dev
kubectl get namespaces
```

### 🔧 리소스에 네임스페이스 지정

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  namespace: dev
```

### 💡 명령어에 네임스페이스 지정

```bash
kubectl get pods -n dev
kubectl delete svc my-service -n test
kubectl config set-context --current --namespace=dev
```

---

## ✅ 2. Label이란?

Label은 **Kubernetes 리소스에 부착하는 key-value 메타데이터**입니다.  
리소스를 그룹화하거나 선택할 때 사용합니다.

### 🎯 목적
- 리소스를 논리적으로 그룹핑
- Service, ReplicaSet, Deployment 등에서 **Selector**로 사용
- Helm, ArgoCD, GitOps 등에도 필수 구성요소

### 📌 예제

```yaml
metadata:
  name: mypod
  labels:
    app: frontend
    tier: web
    env: dev
```

### 🛠 명령어로 Label 붙이기

```bash
kubectl label pod mypod team=backend
```

---

## ✅ 3. Selector란?

Selector는 **Label을 기준으로 리소스를 선택(filtering)** 하기 위한 방식입니다.

### 📌 사용되는 곳

| 위치 | 설명 |
|------|------|
| `Service` | 어떤 Pod로 트래픽을 라우팅할지 결정 |
| `ReplicaSet` | 어떤 Pod를 관리할지 선택 |
| `kubectl get` 명령어 | 리소스 검색 시 필터링 |

### 🎯 Selector 종류

| 유형 | 예시 | 설명 |
|------|------|------|
| Equality | `app=frontend` | 정확히 일치하는 경우 |
| Set-based | `env in (dev,qa)` | 집합에 포함되면 선택 |
| 복수 조건 | `app=web,tier=backend` | AND 조건 |

---

## ✅ 4. 실습 예제: Label + Selector

### 🛠 1) 라벨이 붙은 Pod 정의

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: labeled-pod
  labels:
    app: myapp
    tier: backend
spec:
  containers:
  - name: nginx
    image: nginx
```

### 🛠 2) 라벨을 선택하는 서비스 정의

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
    tier: backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

→ 이 Service는 `app=myapp` AND `tier=backend`인 Pod만 선택

---

## ✅ 5. Label/Selector 명령어 예시

### 🛠 Label로 리소스 조회

```bash
kubectl get pods -l app=myapp
kubectl get svc -l 'env in (dev,stage)'
```

### 🛠 Label 제거

```bash
kubectl label pod mypod team-  # 'team' label 삭제
```

---

## ✅ 6. 요약 비교

| 항목 | 설명 |
|------|------|
| **Namespace** | 리소스의 물리적/논리적 구분 단위 |
| **Label** | 리소스에 붙이는 메타데이터 (Key-Value) |
| **Selector** | Label을 기준으로 리소스를 선택 |

---

## ✅ 결론

Kubernetes를 안정적으로 운영하기 위해서는 다음 세 가지가 핵심입니다:

- 네임스페이스로 **프로젝트나 팀 간 리소스 격리**
- 라벨로 **리소스를 논리적으로 그룹핑**
- 셀렉터로 **정확한 리소스 타겟팅**