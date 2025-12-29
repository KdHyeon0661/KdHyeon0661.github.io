---
layout: post
title: Kubernetes - Django + PostgreSQL + Redis 클러스터 구성
date: 2025-06-14 20:20:23 +0900
category: Kubernetes
---
# 실습: Django + PostgreSQL + Redis 클러스터 구성

## 아키텍처 개요

```plaintext
[User] ──TLS──> [Ingress (nginx, cert-manager)]
                    │
                    ▼
              [Service: django-web]  ◀──── HPA
                    │
                    ▼
           [Deployment: django-web]
             │     │        │
     ConfigMap  Secret   (옵션)InitContainer: collectstatic
             │     │
             ▼     ▼
     [ENV/Settings] [DB/REDIS creds]

[StatefulSet: postgres] ─ PVC(RWO) ─ Headless Service
[StatefulSet: redis]    ─ (옵션: AOF/PVC) ─ Headless Service

[Job: migrate] → Django DB 마이그레이션
[Deployment: celery-worker] / [Deployment: celery-beat] (옵션)
```

---

## 사전 준비: 네임스페이스 및 레이블

애플리케이션을 위한 전용 네임스페이스를 생성합니다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webapp
  labels:
    pod-security.kubernetes.io/enforce: "baseline"
```

**도메인, 이미지 태그, 버전**과 같은 변수는 Helm의 values.yaml이나 Kustomize의 변환기(transformer)를 사용해 일관되게 관리하는 것을 권장합니다.

---

## PostgreSQL 구성 (StatefulSet + PVC + Secret)

> 이 구성은 단일 인스턴스를 기준으로 합니다. 프로덕션 환경에서 고가용성이 필요하다면 Patroni, CloudSQL, RDS와 같은 관리형 서비스를 고려하세요.

**비밀 정보를 저장할 Secret 리소스를 생성합니다. (postgres-secret.yaml)**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: webapp
type: Opaque
stringData:
  POSTGRES_DB: mydb
  POSTGRES_USER: myuser
  POSTGRES_PASSWORD: strong-password-change-me
```

**PostgreSQL StatefulSet과 Headless Service를 정의합니다. (postgres-statefulset.yaml)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: webapp
  labels: { app: postgres }
spec:
  clusterIP: None        # Headless Service (DNS 직접 반환)
  selector: { app: postgres }
  ports:
    - name: pg
      port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: webapp
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels: { app: postgres }
  template:
    metadata:
      labels: { app: postgres }
    spec:
      securityContext:
        fsGroup: 999 # PostgreSQL 컨테이너의 기본 GID와 일치시킵니다.
      containers:
        - name: postgres
          image: postgres:14
          ports: [{ name: pg, containerPort: 5432 }]
          envFrom:
            - secretRef: { name: postgres-secret } # Secret으로부터 환경 변수 주입
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec: { command: ["pg_isready","-U","myuser"] }
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            tcpSocket: { port: 5432 }
            initialDelaySeconds: 20
            periodSeconds: 20
          resources:
            requests:
              cpu: "200m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi
```

적용 명령:
```bash
kubectl apply -f postgres-secret.yaml
kubectl apply -f postgres-statefulset.yaml
```

---

## Redis 구성 (StatefulSet + Service)

> 주로 세션 저장과 캐싱 용도로 사용합니다. 고가용성이 필요한 경우 Redis Sentinel 또는 Redis Operator를 고려하세요.

**redis.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: webapp
  labels: { app: redis }
spec:
  clusterIP: None # Headless Service
  selector: { app: redis }
  ports:
    - name: redis
      port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: webapp
spec:
  serviceName: redis
  replicas: 1
  selector:
    matchLabels: { app: redis }
  template:
    metadata:
      labels: { app: redis }
    spec:
      containers:
        - name: redis
          image: redis:7
          args: ["--save","","--appendonly","no"]  # 디스크 지속성 비활성화 (캐시/세션 전용)
          ports: [{ name: redis, containerPort: 6379 }]
          readinessProbe:
            tcpSocket: { port: 6379 }
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            tcpSocket: { port: 6379 }
            initialDelaySeconds: 15
            periodSeconds: 15
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

적용 명령:
```bash
kubectl apply -f redis.yaml
```

---

## Django 애플리케이션 컨테이너화

**의존성 패키지를 정의하는 requirements.txt 파일**

```
Django>=4.2
gunicorn
psycopg[binary]
django-redis
whitenoise
```

**Django 설정 파일의 핵심 부분 (mysite/settings.py)**

```python
import os
SECRET_KEY = os.getenv("DJANGO_SECRET_KEY", "change-me")
DEBUG = os.getenv("DJANGO_DEBUG", "false").lower() == "true"
ALLOWED_HOSTS = os.getenv("DJANGO_ALLOWED_HOSTS", "localhost,app.local").split(",")

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.getenv("DB_NAME", "mydb"),
        "USER": os.getenv("DB_USER", "myuser"),
        "PASSWORD": os.getenv("DB_PASSWORD", ""),
        "HOST": os.getenv("DB_HOST", "postgres.webapp.svc.cluster.local"),
        "PORT": os.getenv("DB_PORT", "5432"),
        "CONN_MAX_AGE": int(os.getenv("DB_CONN_MAX_AGE", "60")),
    }
}

CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": f"redis://{os.getenv('REDIS_HOST', 'redis.webapp.svc.cluster.local')}:6379/1",
        "OPTIONS": {"CLIENT_CLASS": "django_redis.client.DefaultClient"},
        "KEY_PREFIX": "webapp",
    }
}

SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "default"

STATIC_URL = "/static/"
STATIC_ROOT = os.getenv("STATIC_ROOT", "/var/www/static")
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware", # 정적 파일 서빙
    # ... 기타 미들웨어
]
```

**Dockerfile**

```Dockerfile
FROM python:3.12-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /code
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM base AS runtime
WORKDIR /code
COPY --from=base /usr/local /usr/local
COPY . .
ENV DJANGO_SETTINGS_MODULE=mysite.settings
ENV PORT=8000
# collectstatic 작업은 InitContainer에서 수행할 예정입니다.

CMD ["gunicorn", "mysite.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "3", "--threads", "2", "--timeout", "60"]
```

이미지 빌드 및 푸시:
```bash
docker build -t <username>/django-app:0.1.0 .
docker push <username>/django-app:0.1.0
```

---

## Django 애플리케이션 설정 (ConfigMap, Secret)

**django-config.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
  namespace: webapp
data:
  DJANGO_ALLOWED_HOSTS: "app.local,127.0.0.1"
  DJANGO_DEBUG: "false"
  DB_HOST: "postgres.webapp.svc.cluster.local"
  DB_NAME: "mydb"
  DB_USER: "myuser"
  DB_CONN_MAX_AGE: "60"
  REDIS_HOST: "redis.webapp.svc.cluster.local"
  STATIC_ROOT: "/var/www/static"
---
apiVersion: v1
kind: Secret
metadata:
  name: django-secret
  namespace: webapp
type: Opaque
stringData:
  DJANGO_SECRET_KEY: "please-change-me"
  DB_PASSWORD: "strong-password-change-me"
```

적용 명령:
```bash
kubectl apply -f django-config.yaml
```

---

## 정적 파일 수집 및 데이터베이스 마이그레이션 자동화

### InitContainer를 이용한 정적 파일 수집 (collectstatic)

Django Deployment 매니페스트에 정적 파일을 수집하는 InitContainer와 이를 공유할 Volume을 정의합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-web
  namespace: webapp
  labels: { app: django-web }
spec:
  replicas: 2
  selector:
    matchLabels: { app: django-web }
  template:
    metadata:
      labels: { app: django-web }
    spec:
      securityContext:
        fsGroup: 10001 # 애플리케이션 컨테이너의 파일 시스템 그룹
      initContainers:
        - name: collectstatic
          image: <username>/django-app:0.1.0
          command: ["sh","-c"]
          args: ["python manage.py collectstatic --noinput"]
          envFrom:
            - configMapRef: { name: django-config }
            - secretRef: { name: django-secret }
          volumeMounts:
            - name: static
              mountPath: /var/www/static
      containers:
        - name: web
          image: <username>/django-app:0.1.0
          ports: [{ containerPort: 8000, name: http }]
          envFrom:
            - configMapRef: { name: django-config }
            - secretRef: { name: django-secret }
          volumeMounts:
            - name: static
              mountPath: /var/www/static
          readinessProbe:
            httpGet: { path: /admin/login/, port: http }
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /healthz, port: http }
            initialDelaySeconds: 20
            periodSeconds: 10
          startupProbe:
            httpGet: { path: /healthz, port: http }
            failureThreshold: 30
            periodSeconds: 2
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
          securityContext:
            runAsNonRoot: true
            runAsUser: 10001
            allowPrivilegeEscalation: false
      volumes:
        - name: static
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: django-web
  namespace: webapp
spec:
  selector: { app: django-web }
  ports:
    - name: http
      port: 80
      targetPort: http
```

### 데이터베이스 마이그레이션 Job

배포 전에 데이터베이스 스키마를 최신 상태로 업데이트하기 위한 Job을 생성합니다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: django-migrate
  namespace: webapp
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: <username>/django-app:0.1.0
          command: ["sh","-c"]
          args: ["python manage.py migrate --noinput"]
          envFrom:
            - configMapRef: { name: django-config }
            - secretRef: { name: django-secret }
```

적용 순서:
1. 마이그레이션 Job을 먼저 실행하여 데이터베이스를 준비합니다.
2. Job이 완료된 후 웹 애플리케이션 Deployment를 배포합니다.

```bash
kubectl apply -f django-migrate-job.yaml
kubectl wait --for=condition=complete job/django-migrate -n webapp --timeout=120s
kubectl apply -f django-web-deployment.yaml
```

---

## 외부 접근 구성: Ingress 및 TLS

### 개발 환경용 Ingress (HTTP)

로컬 개발을 위한 간단한 Ingress 리소스를 구성합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: django-ingress
  namespace: webapp
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "20m"
spec:
  ingressClassName: nginx
  rules:
    - host: app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: django-web
                port: { number: 80 }
```

로컬 호스트 파일에 다음 항목을 추가합니다.
```
127.0.0.1 app.local
```

### 프로덕션 환경용 Ingress (TLS 자동화 with cert-manager)

Let's Encrypt를 이용해 TLS 인증서를 자동으로 발급 및 갱신하는 구성입니다. cert-manager가 클러스터에 설치되어 있어야 합니다.

```yaml
# 1. ClusterIssuer 정의 (인증서 발급자)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http01
spec:
  acme:
    email: you@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef: { name: letsencrypt-account-key }
    solvers:
      - http01:
          ingress:
            class: nginx
---
# 2. TLS가 활성화된 Ingress 정의
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: django-ingress-tls
  namespace: webapp
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-http01"
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["app.example.com"]
      secretName: django-tls # cert-manager가 생성할 Secret 이름
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: django-web
                port: { number: 80 }
```

---

## Django 관리 작업 수행

실행 중인 파드에서 직접 Django 관리 명령어를 실행할 수 있습니다.

```bash
# 데이터베이스 마이그레이션 (실시간)
kubectl exec -n webapp deploy/django-web -- python manage.py migrate

# 슈퍼유저 생성 (실시간)
kubectl exec -n webapp deploy/django-web -- python manage.py createsuperuser
```

> 프로덕션 환경에서는 `createsuperuser` 명령을 일회성 Job으로 실행하고, 생성된 자격 증명은 Secret Manager와 같은 외부 비밀 저장소로 안전하게 관리하는 방식을 권장합니다.

---

## Redis 세션 및 캐시 최적화

- 앞선 설정(`SESSION_ENGINE = cache`, `django-redis`)으로 세션을 Redis에 저장하도록 구성했습니다.
- 자주 조회되거나 비용이 큰 쿼리 결과에는 `cache_page` 데코레이터나 저수준 캐시 API를 활용해 성능을 향상시킬 수 있습니다.
- 현재 구성은 단일 Redis 인스턴스입니다. Redis 파드가 재시작되면 세션 데이터가 손실될 수 있습니다. 이를 대비하여 로그인 유지 정책, 재시도 로직, Graceful Shutdown을 애플리케이션에 구현하는 것이 좋습니다.

---

## Celery를 이용한 비동기 작업 (선택 사항)

**Celery 애플리케이션 설정 (mysite/celery.py)**

```python
import os
from celery import Celery
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "mysite.settings")
app = Celery("mysite")
app.conf.broker_url = f"redis://{os.getenv('REDIS_HOST','redis.webapp.svc.cluster.local')}:6379/0"
app.conf.result_backend = app.conf.broker_url
app.autodiscover_tasks()
```

**Celery Worker Deployment (celery-worker.yaml)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: celery-worker
  namespace: webapp
spec:
  replicas: 1
  selector: { matchLabels: { app: celery-worker } }
  template:
    metadata: { labels: { app: celery-worker } }
    spec:
      containers:
        - name: worker
          image: <username>/django-app:0.1.0
          command: ["sh","-c"]
          args: ["celery -A mysite.celery:app worker --loglevel=INFO -Q default -c 2"]
          envFrom:
            - configMapRef: { name: django-config }
            - secretRef: { name: django-secret }
          resources:
            requests: { cpu: "100m", memory: "256Mi" }
            limits: { cpu: "500m", memory: "512Mi" }
```

---

## 가용성 및 안정성 강화

### 수평 파드 오토스케일러 (HPA)

CPU 사용률을 기준으로 웹 애플리케이션 파드의 수를 자동으로 조정합니다.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: django-web
  namespace: webapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: django-web
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

### PodDisruptionBudget (PDB)

의도적인 노드 드레이닝(Drain)이나 업그레이드 중에도 최소한의 서비스 가용성을 보장합니다.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: django-web-pdb
  namespace: webapp
spec:
  minAvailable: 1
  selector:
    matchLabels: { app: django-web }
```

### NetworkPolicy

Ingress Controller를 통한 트래픽만 Django 웹 파드에 접근하도록 제한합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: django-allow-only-ingress
  namespace: webapp
spec:
  podSelector:
    matchLabels: { app: django-web }
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx # Ingress Controller 네임스페이스
      ports:
        - port: 8000
          protocol: TCP
```

---

## 보안 및 운영 모범 사례

- **SecurityContext**: 모든 컨테이너에 `runAsNonRoot: true`, `allowPrivilegeEscalation: false`를 설정하고 불필요한 Linux Capabilities를 제거하세요.
- **비밀 정보 관리**: Kubernetes Secret과 함께 SOPS, External Secrets Operator 등을 활용해 안전하게 관리하세요.
- **데이터 백업**: PostgreSQL 데이터는 정기적인 `pg_dump` 백업 또는 스토리지 스냅샷을 수행하고, 복구 절차를 정기적으로 리허설하세요.
- **관측 가능성**: Liveness/Readiness 프로브 이벤트, Nginx Ingress 로그, 애플리케이션의 구조화된 로그(JSON 형식), 요청 수/지연 시간/오류율 메트릭을 수집하세요.
- **배포 전략**: Ingress 어노테이션을 이용한 간단한 Canary 배포나 Argo Rollouts를 도입해 위험을 최소화하면서 점진적으로 롤아웃하세요.

---

## 배포 자동화: Helm 및 Kustomize 활용

- **Helm**: `values.yaml` 파일에 `image.tag`, `ingress.host`, `resources`, 환경 변수 등을 파라미터화하여 템플릿을 관리합니다.
- **Kustomize**: `base` 디렉터리에 공통 리소스를 정의하고, `overlays/dev`, `overlays/staging`, `overlays/prod` 디렉터리에서 환경별로 레플리카 수, HPA 설정, Ingress 호스트, TLS 여부 등을 오버레이합니다.

---

## 전체 배포 실행 절차

아래는 모든 리소스를 순서대로 배포하는 샘플 명령어입니다.

```bash
# 1. 네임스페이스 생성
kubectl apply -f ns.yaml

# 2. 데이터 계층 배포 (PostgreSQL, Redis)
kubectl apply -f postgres-secret.yaml
kubectl apply -f postgres-statefulset.yaml
kubectl apply -f redis.yaml

# 3. 애플리케이션 설정 배포
kubectl apply -f django-config.yaml

# 4. 데이터베이스 마이그레이션 실행 및 대기
kubectl apply -f django-migrate-job.yaml
kubectl wait --for=condition=complete job/django-migrate -n webapp --timeout=120s

# 5. 웹 애플리케이션 배포
kubectl apply -f django-web-deployment.yaml

# 6. 외부 접근 구성
kubectl apply -f django-ingress.yaml

# 7. 가용성 및 보안 구성 (선택)
kubectl apply -f hpa.yaml
kubectl apply -f pdb.yaml
kubectl apply -f networkpolicy.yaml
```

---

## 검증 및 운영 명령어

애플리케이션 상태를 확인하고 문제를 진단하는 데 유용한 명령어들입니다.

```bash
# 기본 상태 확인
kubectl -n webapp get pods,svc,ingress,cm,secret,pvc
kubectl -n webapp get events --sort-by=.lastTimestamp

# 상세 정보 및 로그 확인
kubectl -n webapp describe deploy django-web
kubectl -n webapp logs deploy/django-web -c web --tail=200

# 컨테이너 내부 셸 접근
kubectl -n webapp exec -it deploy/django-web -- /bin/sh

# 네트워크 연결 테스트 (netshoot 유틸리티 팟 사용)
kubectl -n webapp run -it netshoot --rm --image=nicolaka/netshoot -- /bin/sh
# netshoot 컨테이너 내에서:
curl -I django-web.webapp.svc.cluster.local
nc -zv postgres.webapp.svc.cluster.local 5432
nc -zv redis.webapp.svc.cluster.local 6379
```

---

## 리소스 정리

실습을 마치고 모든 리소스를 삭제하려면 네임스페이스를 삭제하면 됩니다.

```bash
kubectl delete ns webapp
```

---

## 결론

이 실습을 통해 Kubernetes 상에서 Django, PostgreSQL, Redis로 구성된 전형적인 웹 애플리케이션 스택을 배포하는 방법을 살펴보았습니다. 핵심은 상태를 유지하는 데이터베이스와 캐시 서버는 안정적인 StatefulSet과 영구 볼륨으로 구성하고, 무상태(stateless)인 Django 애플리케이션은 다중 레플리카의 Deployment로 배포하여 확장성과 가용성을 확보하는 것입니다. InitContainer와 Job을 활용한 빌드 타임/배포 타임 작업 자동화, Ingress와 cert-manager를 통한 안전한 외부 노출, HPA와 PDB를 통한 운영 견고성 강화는 프로덕션급 배포에 필수적인 요소입니다. 보안 컨텍스트 설정, 비밀 정보 관리, 모니터링, 백업 전략을 함께 고려하면 더욱 완성도 높은 운영 체계를 구축할 수 있습니다.