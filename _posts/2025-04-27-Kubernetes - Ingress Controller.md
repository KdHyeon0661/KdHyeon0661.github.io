---
layout: post
title: Kubernetes - Ingress Controller
date: 2025-04-27 20:20:23 +0900
category: Kubernetes
---
# Kubernetes Ingress Controller로 외부 접근 설정하기

Kubernetes에서 외부 요청을 내부 서비스로 전달하려면 다양한 방식이 있습니다:

- `NodePort`  
- `LoadBalancer`  
- `Ingress`

이 중 `Ingress`는 가장 유연하고 확장성 높은 방식입니다.  
특히 **여러 서비스를 단일 도메인/포트로 관리**할 수 있어 **실제 운영 환경에서 필수**로 사용됩니다.

---

## ✅ Ingress란?

`Ingress`는 클러스터 외부의 HTTP(S) 요청을 내부 서비스로 **라우팅하는 규칙**입니다.

단, Ingress 객체만으로는 동작하지 않습니다.  
실제로 요청을 수신하고 처리하는 **Ingress Controller**가 필요합니다.

### 📌 구성 요소

| 구성 | 역할 |
|------|------|
| `Ingress` | 도메인/경로 기반 라우팅 규칙 정의 |
| `Ingress Controller` | Ingress 규칙을 실제로 처리하는 리버스 프록시 (ex: Nginx) |
| `Service` | 실제 트래픽 전달 대상 (백엔드) |
| `Pod` | 백엔드 앱 |

---

## ✅ Ingress Controller 설치 (Nginx 예시)

가장 널리 쓰이는 Ingress Controller는 `nginx`입니다.  
아래는 **Minikube 환경 기준**으로 설치하는 예시입니다.

### 🔧 설치

```bash
minikube addons enable ingress
```

### 📌 일반 클러스터에서는 Helm 사용

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx
```

→ 설치 후 `nginx-controller`라는 이름의 Pod와 Service가 생성됩니다.

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

---

## ✅ 서비스 배포 예제: 두 개의 앱

### 📄 1. app1.yaml (Hello App 1)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app1-service
spec:
  selector:
    app: app1
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: hashicorp/http-echo
        args:
          - "-text=Hello from App1"
        ports:
          - containerPort: 5678
```

### 📄 2. app2.yaml (Hello App 2)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app2-service
spec:
  selector:
    app: app2
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: hashicorp/http-echo
        args:
          - "-text=Hello from App2"
        ports:
          - containerPort: 5678
```

---

## ✅ Ingress 리소스 정의

이제 `/app1`, `/app2`로 라우팅되는 Ingress를 정의합니다.

### 📄 ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /  # 경로 재작성
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

---

## ✅ 적용 및 확인

```bash
kubectl apply -f app1.yaml
kubectl apply -f app2.yaml
kubectl apply -f ingress.yaml
```

```bash
kubectl get ingress
```

---

## ✅ 테스트 (Minikube 기준)

```bash
minikube ip
```

예를 들어 IP가 `192.168.49.2`이면 다음 주소로 테스트합니다:

- http://192.168.49.2/app1 → `Hello from App1`
- http://192.168.49.2/app2 → `Hello from App2`

### 또는 port-forward로 테스트

```bash
kubectl port-forward svc/ingress-nginx-controller -n ingress-nginx 8080:80
curl http://localhost:8080/app1
```

---

## ✅ HTTPS (TLS) 설정도 가능

TLS 인증서를 Secret으로 등록하고 아래처럼 Ingress에 연결합니다:

```yaml
tls:
- hosts:
  - example.com
  secretName: tls-secret
```

→ 무료 인증서 발급을 원하면 `cert-manager` 연동도 고려해보세요.

---

## ✅ 정리

| 리소스 | 설명 |
|--------|------|
| `Ingress` | HTTP/S 트래픽을 경로/호스트 기반으로 라우팅 |
| `Ingress Controller` | 실제로 요청을 수신하고 처리하는 프록시 |
| `Nginx Ingress` | 가장 많이 사용하는 오픈소스 컨트롤러 |
| `annotations` | 경로 재작성, 인증, CORS 등 고급 설정 가능 |
| `cert-manager` | TLS 인증서 자동 발급/갱신 (Let's Encrypt) |

---

## ✅ 결론

Ingress Controller를 이용하면 하나의 LoadBalancer 또는 IP를 통해  
**여러 서비스를 도메인/경로 기반으로 유연하게 라우팅**할 수 있습니다.

이는 운영환경에서 다음을 가능하게 합니다:

- `/api`, `/admin`, `/shop` 등 다양한 서비스 경로 제공
- TLS 및 인증 설정 일괄 관리
- DNS 기반 서비스 노출