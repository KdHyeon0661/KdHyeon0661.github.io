---
layout: post
title: Kubernetes - 실습 : 블로그 플랫폼을 쿠버네티스로 배포하기
date: 2025-06-14 19:20:23 +0900
category: Kubernetes
---
# 🛠️ 실습: 블로그 플랫폼을 쿠버네티스로 배포하기

이번 실습에서는 **간단한 블로그 플랫폼을 Kubernetes 클러스터에 직접 배포**해보겠습니다.

- ✅ 대상 플랫폼: **[Ghost](https://ghost.org/)** (오픈소스 블로그 플랫폼)
- ✅ 구성 요소:
  - `Ghost` (Node.js 기반 블로그 CMS)
  - `MySQL` (데이터 저장)
  - `PersistentVolume`, `Service`, `Deployment`
  - `Ingress` (도메인 연결)
- ✅ 목표:
  - Kubernetes의 **핵심 오브젝트(Pod, PVC, Service, Ingress)** 활용
  - 실전 수준의 블로그 배포 구성 익히기

---

## 📦 1. 준비사항

- Minikube, Kind, 또는 실제 K8s 클러스터
- `kubectl` 설치
- Ingress Controller 설치 (예: NGINX)

```bash
minikube start --driver=docker
minikube addons enable ingress
```

---

## 📁 2. 전체 구조 개요

```plaintext
[Ingress]
   ↓ blog.example.com
[Service: ghost-service] → [Pod: ghost]
                      ↘ (connects to)
                   [Service: mysql] → [Pod: mysql]
                          ↕
               [PersistentVolumeClaim: mysql-pvc]
```

---

## 🛠️ 3. MySQL 구성

### `mysql-deployment.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  password: bXlzcWw=   # 'mysql' base64 인코딩
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:5.7
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
            - name: MYSQL_DATABASE
              value: ghost
          volumeMounts:
            - mountPath: /var/lib/mysql
              name: mysql-storage
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
    - port: 3306
  selector:
    app: mysql
```

```bash
kubectl apply -f mysql-deployment.yaml
```

---

## 👻 4. Ghost 블로그 배포

### `ghost-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ghost
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ghost
  template:
    metadata:
      labels:
        app: ghost
    spec:
      containers:
        - name: ghost
          image: ghost:5
          ports:
            - containerPort: 2368
          env:
            - name: database__client
              value: mysql
            - name: database__connection__host
              value: mysql
            - name: database__connection__user
              value: root
            - name: database__connection__password
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
            - name: database__connection__database
              value: ghost
---
apiVersion: v1
kind: Service
metadata:
  name: ghost-service
spec:
  selector:
    app: ghost
  ports:
    - port: 80
      targetPort: 2368
```

```bash
kubectl apply -f ghost-deployment.yaml
```

---

## 🌐 5. Ingress 설정

### `ghost-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ghost-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: blog.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ghost-service
                port:
                  number: 80
```

```bash
kubectl apply -f ghost-ingress.yaml
```

🔧 `hosts` 파일에 다음 라인 추가:

```plaintext
127.0.0.1 blog.local
```

---

## ✅ 6. 결과 확인

```bash
kubectl get all
kubectl describe ingress ghost-ingress
```

브라우저에서 `http://blog.local` 접속 → Ghost 초기 설정 화면 확인 가능

---

## 🔄 7. 고급 구성 (선택)

- **ConfigMap**으로 설정 분리
- **TLS 인증서** 자동 발급 (cert-manager + Let's Encrypt)
- **HorizontalPodAutoscaler**로 트래픽 기반 확장
- **Helm 차트화** → 배포 자동화

---

## 🧼 8. 정리 및 삭제

```bash
kubectl delete -f ghost-ingress.yaml
kubectl delete -f ghost-deployment.yaml
kubectl delete -f mysql-deployment.yaml
```

---

## 📚 참고 링크

- [Ghost 공식 Docker 문서](https://ghost.org/docs/docker/)
- [Kubernetes Ingress 소개](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Minikube + Ingress 설정](https://minikube.sigs.k8s.io/docs/handbook/ingress/)