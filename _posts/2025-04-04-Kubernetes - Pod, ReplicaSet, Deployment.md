---
layout: post
title: Kubernetes - Pod, ReplicaSet, Deployment
date: 2025-04-04 21:20:23 +0900
category: Kubernetes
---
# K8s 핵심 오브젝트 이해하기: Pod, ReplicaSet, Deployment

쿠버네티스(Kubernetes)는 다양한 리소스 오브젝트(object)를 통해 컨테이너화된 애플리케이션을 관리합니다. 그중 가장 핵심이 되는 오브젝트는 다음 세 가지입니다:

- `Pod` : 컨테이너의 기본 단위  
- `ReplicaSet` : Pod의 수량 보장  
- `Deployment` : 애플리케이션의 선언적 업데이트 관리  

이 글에서는 각각의 역할과 차이점, 사용 예제를 살펴봅니다.

---

## ✅ 1. Pod: 쿠버네티스의 가장 작은 단위

### 🌟 개념
- Pod는 **하나 이상의 컨테이너를 묶는 논리적 단위**
- 보통 **하나의 Pod에는 하나의 컨테이너**만 존재하지만, sidecar 패턴 등으로 2개 이상도 가능
- **같은 Pod 내 컨테이너는 네트워크와 볼륨을 공유**함

### 🔧 YAML 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
```

### ✅ 주요 특징
- Pod는 ephemeral(휘발성) → 삭제되면 복구되지 않음
- 직접 사용하는 경우는 드물며, 일반적으로 ReplicaSet이나 Deployment를 통해 생성됨

---

## ✅ 2. ReplicaSet: Pod의 개수를 보장

### 🌟 개념
- ReplicaSet은 **특정 수의 동일한 Pod가 항상 존재하도록 보장**
- Pod가 꺼지면 자동으로 새로 생성하여 개수를 유지함

### 🔧 YAML 예시

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

### ✅ 주요 특징
- Pod가 삭제되면 자동 복구됨
- 하지만 업데이트(버전 변경 등)는 수동으로 해야 함
- 이를 해결하기 위해 **Deployment** 사용이 일반적

---

## ✅ 3. Deployment: 선언적 업데이트를 위한 최상위 오브젝트

### 🌟 개념
- Deployment는 **ReplicaSet을 관리**하고, **롤링 업데이트**, **롤백** 등 애플리케이션의 **배포 전략을 정의**함
- 실무에서 가장 많이 사용되는 오브젝트

### 🔧 YAML 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### ✅ 주요 기능
- `kubectl apply`만으로 **버전 업데이트 가능**
- 롤링 업데이트, Canary 배포, 롤백 지원
- ReplicaSet을 자동으로 생성 및 교체

---

## ✅ 4. 세 오브젝트의 관계

```
[ Deployment ]
     ↓
[ ReplicaSet ]
     ↓
[ Pod ] → Container(s)
```

- Deployment는 ReplicaSet을 생성 및 관리
- ReplicaSet은 Pod를 생성 및 개수 유지
- Pod는 실제로 컨테이너를 실행

---

## ✅ 5. 실습 예시: nginx 배포

```bash
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --type=NodePort --port=80
kubectl get all
```

→ 위 명령은 Deployment를 생성하고, 그 아래 ReplicaSet과 Pod가 자동 생성됨

---

## ✅ 6. 자주 사용하는 명령어

```bash
kubectl get pods
kubectl get rs
kubectl get deployments

kubectl describe pod [이름]
kubectl describe rs [이름]
kubectl describe deployment [이름]

kubectl delete pod [이름]
kubectl delete deployment [이름]
```

---

## ✅ 7. 언제 무엇을 쓸까?

| 목적 | 오브젝트 | 설명 |
|------|----------|------|
| 실습, 디버깅용 임시 컨테이너 | `Pod` | 수동으로 생성 가능하나 비권장 |
| 동일한 Pod를 여러 개 유지 | `ReplicaSet` | 직접 사용보단 Deployment를 통해 간접 사용 |
| 앱 배포, 롤백, 업데이트 | **`Deployment`** | ✅ 실무 사용의 표준 방식 |

---

## ✅ 결론

Kubernetes에서 Pod, ReplicaSet, Deployment는 **앱 배포와 확장성을 위한 핵심 구성요소**입니다.  
아래와 같이 요약할 수 있습니다:

- **Pod**: 컨테이너 실행 단위  
- **ReplicaSet**: Pod 수 유지  
- **Deployment**: 앱 배포, 롤링 업데이트, 자동 관리