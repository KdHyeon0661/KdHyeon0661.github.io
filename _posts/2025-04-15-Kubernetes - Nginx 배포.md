---
layout: post
title: Kubernetes - Nginx 배포
date: 2025-04-15 20:20:23 +0900
category: Kubernetes
---
# Kubernetes 예제: 간단한 Nginx 배포

이 문서는 Kubernetes 클러스터에 Nginx 웹 서버를 배포하는 가장 기본적인 예제를 설명합니다.  
실습 목적에 적합하며, 실전에서도 유용한 구성입니다.

---

## ✅ 목표

- Nginx를 **Deployment**로 배포하여 **Pod를 관리**  
- Nginx를 **Service(NodePort)**로 외부에 노출  
- `kubectl` 명령어로 상태 확인 및 테스트

---

## ✅ 1. Nginx Deployment 정의

### 📄 `nginx-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
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
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 📌 설명

| 항목 | 설명 |
|------|------|
| `replicas: 2` | Pod 2개 실행 |
| `selector.matchLabels` | 관리할 Pod 지정 |
| `template.spec.containers.image` | 최신 nginx 이미지 사용 |
| `containerPort: 80` | Nginx 기본 포트 노출 |

---

## ✅ 2. Service 정의 (NodePort)

### 📄 `nginx-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

### 📌 설명

| 항목 | 설명 |
|------|------|
| `type: NodePort` | 클러스터 외부에서 접근 가능 |
| `selector.app=nginx` | 해당 라벨을 가진 Pod에 연결 |
| `port: 80` | Service 접근 포트 |
| `targetPort: 80` | 컨테이너 내부 포트 |
| `nodePort: 30080` | 노드 외부 노출 포트 (30000~32767) |

---

## ✅ 3. 리소스 배포

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
```

---

## ✅ 4. 리소스 확인

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```

### 예시 출력

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           1m

NAME             TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-service    NodePort   10.96.22.121   <none>        80:30080/TCP   1m
```

---

## ✅ 5. 외부에서 접근하기

### 방법 1: Minikube 사용자

```bash
minikube service nginx-service
```

→ 브라우저에서 Nginx Welcome 페이지 열림

### 방법 2: 노드 IP 직접 접근 (NodePort)

```bash
curl http://<노드 IP>:30080
```

→ Nginx HTML 페이지가 출력되어야 함

---

## ✅ 6. YAML 하나로 묶기 (선택)

```yaml
# nginx-full.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
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
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

```bash
kubectl apply -f nginx-full.yaml
```

---

## ✅ 7. 리소스 정리

```bash
kubectl delete -f nginx-full.yaml
# 또는 개별 삭제
kubectl delete deployment nginx-deployment
kubectl delete service nginx-service
```

---

## ✅ 결론

이 간단한 예제를 통해 다음 내용을 실습했습니다:

- Deployment로 Pod 2개 생성
- NodePort Service로 외부에 노출
- `kubectl` 명령어로 확인 및 관리
