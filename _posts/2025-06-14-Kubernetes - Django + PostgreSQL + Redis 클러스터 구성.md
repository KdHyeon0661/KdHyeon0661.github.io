---
layout: post
title: Kubernetes - Django + PostgreSQL + Redis 클러스터 구성
date: 2025-06-14 20:20:23 +0900
category: Kubernetes
---
# 🛠️ 실습: Django + PostgreSQL + Redis 클러스터 구성

이번 실습에서는 **Kubernetes 위에서 Django 웹 애플리케이션을 PostgreSQL, Redis와 함께 배포**해보겠습니다. 이 구성은 실무에서 많이 사용하는 **풀스택 웹 애플리케이션 구성**입니다.

---

## ✅ 실습 목표

- Django 애플리케이션을 Docker로 컨테이너화하고 Kubernetes에 배포
- PostgreSQL을 **PersistentVolume**과 함께 구성
- Redis를 **캐시 및 세션 저장소**로 사용
- Kubernetes의 핵심 리소스를 조합하여 **실무형 구성** 구현

---

## 📦 아키텍처 구성

```plaintext
[Ingress]
   ↓ (app.example.com)
[Service: django-service] → [Deployment: django]
        ↘                      ↙
    [Service: redis]      [Service: postgres]
            ↓                    ↓
        [Pod: redis]         [Pod: postgres]
                               ↕
                           [PVC: postgres-data]
```

---

## 1️⃣ PostgreSQL 구성

### `postgres.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
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
  name: postgres-secret
type: Opaque
data:
  POSTGRES_PASSWORD: cGFzc3dvcmQ= # 'password'
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:14
          env:
            - name: POSTGRES_DB
              value: mydb
            - name: POSTGRES_USER
              value: myuser
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
          ports:
            - containerPort: 5432
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-storage
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  ports:
    - port: 5432
  selector:
    app: postgres
```

```bash
kubectl apply -f postgres.yaml
```

---

## 2️⃣ Redis 구성

### `redis.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  ports:
    - port: 6379
  selector:
    app: redis
```

```bash
kubectl apply -f redis.yaml
```

---

## 3️⃣ Django 애플리케이션 Docker화

### 📁 프로젝트 구조 예시

```plaintext
my-django-app/
├── Dockerfile
├── requirements.txt
└── mysite/
    └── ...
```

### `Dockerfile`

```Dockerfile
FROM python:3.10-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /code

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["gunicorn", "mysite.wsgi:application", "--bind", "0.0.0.0:8000"]
```

**이미지를 빌드하고 DockerHub에 Push**

```bash
docker build -t <username>/django-app .
docker push <username>/django-app
```

---

## 4️⃣ Django 배포

### `django-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django
spec:
  replicas: 1
  selector:
    matchLabels:
      app: django
  template:
    metadata:
      labels:
        app: django
    spec:
      containers:
        - name: django
          image: <username>/django-app
          ports:
            - containerPort: 8000
          env:
            - name: DB_HOST
              value: postgres
            - name: DB_NAME
              value: mydb
            - name: DB_USER
              value: myuser
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            - name: REDIS_HOST
              value: redis
---
apiVersion: v1
kind: Service
metadata:
  name: django-service
spec:
  selector:
    app: django
  ports:
    - port: 80
      targetPort: 8000
```

```bash
kubectl apply -f django-deployment.yaml
```

---

## 5️⃣ Ingress 설정 (도메인 라우팅)

### `django-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: django-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: django-service
                port:
                  number: 80
```

```bash
kubectl apply -f django-ingress.yaml
```

🔧 `/etc/hosts`에 추가:

```
127.0.0.1 app.local
```

---

## 🧪 6️⃣ 마이그레이션 및 Admin 계정 생성

```bash
kubectl exec -it deploy/django -- python manage.py migrate
kubectl exec -it deploy/django -- python manage.py createsuperuser
```

---

## ✅ 결과 확인

- 브라우저 접속: http://app.local
- Django 프로젝트가 정상적으로 PostgreSQL 및 Redis와 연결되었는지 확인
- Redis는 세션 저장소나 캐시 용도로 설정 가능

---

## 📌 마무리 요약

| 구성 요소 | 역할 |
|-----------|------|
| **Django** | 웹 애플리케이션 |
| **PostgreSQL** | 관계형 데이터 저장소 |
| **Redis** | 캐시 / 세션 저장소 |
| **PVC, Secret, ConfigMap** | 안정적이고 보안성 있는 데이터 관리 |
| **Ingress** | 도메인 기반 트래픽 라우팅 |

---

## 🔄 다음 단계 제안

- TLS 인증서 적용 (cert-manager + Let's Encrypt)
- Redis 세션 설정 (Django 세션 엔진 연동)
- Django + Celery + Redis for async task
- Helm 차트로 배포 자동화
- GitHub Actions로 CI/CD 구축

---

## 📚 참고 자료

- [Django 공식 문서](https://docs.djangoproject.com/)
- [PostgreSQL 공식 Docker 이미지](https://hub.docker.com/_/postgres)
- [Redis Docker 문서](https://hub.docker.com/_/redis)
- [Gunicorn 공식 문서](https://docs.gunicorn.org/)