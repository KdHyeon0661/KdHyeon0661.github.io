---
layout: post
title: Kubernetes - Django + PostgreSQL + Redis 클러스터 구성
date: 2025-06-14 20:20:23 +0900
category: Kubernetes
---
# 실습: Django + PostgreSQL + Redis 클러스터 구성

## 아키텍처

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

## 0. 사전 준비: 네임스페이스/레이블/주요 변수

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webapp
  labels:
    pod-security.kubernetes.io/enforce: "baseline"
```

**도메인/이미지 태그/버전**은 환경변수처럼 통일해서 관리하는 것을 권장합니다(Helm/Kustomize values로 치환).

---

## 1. PostgreSQL (StatefulSet + PVC + Secret)

> 단일 인스턴스 기준. 고가용성이 필요하면 Patroni/CloudSQL/RDS 사용을 권장.

**postgres-secret.yaml**

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

**postgres-statefulset.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: webapp
  labels: { app: postgres }
spec:
  clusterIP: None        # Headless Service
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
        fsGroup: 999
      containers:
        - name: postgres
          image: postgres:14
          ports: [{ name: pg, containerPort: 5432 }]
          envFrom:
            - secretRef: { name: postgres-secret }
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

적용:

```bash
kubectl apply -f postgres-secret.yaml
kubectl apply -f postgres-statefulset.yaml
```

---

## 2. Redis (StatefulSet + Service)

> 세션/캐시용. 고가용성은 Redis Sentinel/Operator 고려.

**redis.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: webapp
  labels: { app: redis }
spec:
  clusterIP: None
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
          args: ["--save","","--appendonly","no"]  # 단순 캐시/세션 용도
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

적용:

```bash
kubectl apply -f redis.yaml
```

---

## 3. Django 컨테이너화

**requirements.txt**

```
Django>=4.2
gunicorn
psycopg[binary]
django-redis
whitenoise
```

**mysite/settings.py** (핵심 부분 발췌)

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
    "whitenoise.middleware.WhiteNoiseMiddleware",
    # ...
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
# collectstatic는 InitContainer에서 수행(아래 참조)
CMD ["gunicorn", "mysite.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "3", "--threads", "2", "--timeout", "60"]
```

빌드/푸시:

```bash
docker build -t <username>/django-app:0.1.0 .
docker push <username>/django-app:0.1.0
```

---

## 4. Django 설정: Secret/ConfigMap/ServiceAccount

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

적용:

```bash
kubectl apply -f django-config.yaml
```

---

## 5. 정적파일 수집/DB 마이그레이션 자동화

### 5.1 정적파일: EmptyDir + InitContainer(collectstatic)

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
        fsGroup: 10001
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

### 5.2 마이그레이션 Job

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

```bash
kubectl apply -f django-migrate-job.yaml
kubectl apply -f django-web-deployment.yaml
```

---

## 6. Ingress + TLS (nginx-ingress + cert-manager)

**Ingress (로컬 개발용)**

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

호스트 파일:

```
127.0.0.1 app.local
```

**프로덕션**에서는 cert-manager로 TLS 자동화:

```yaml
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
      secretName: django-tls
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

## 7. Django 관리: 마이그레이션/슈퍼유저

```bash
kubectl exec -n webapp deploy/django-web -- python manage.py migrate
kubectl exec -n webapp deploy/django-web -- python manage.py createsuperuser
```

> 프로덕션에서는 createsuperuser는 일시적 Job으로 수행 후 Secret Manager로 관리하는 것을 추천.

---

## 8. Redis 세션/캐시 최적화

- `SESSION_ENGINE = cache` 및 `django-redis` 사용(이미 settings 반영).
- 관리자/빈번한 쿼리는 `cache_page`/저수준 캐시로 캐시 효율을 높입니다.
- Redis를 StatefulSet으로 구성했지만, **세션 캐시 손실**에 대비해 로그인 유지 정책/재시도/Graceful Shutdown을 고려하세요.

---

## 9. Celery(옵션): 비동기 작업

**celery.py**

```python
import os
from celery import Celery
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "mysite.settings")
app = Celery("mysite")
app.conf.broker_url = f"redis://{os.getenv('REDIS_HOST','redis.webapp.svc.cluster.local')}:6379/0"
app.conf.result_backend = app.conf.broker_url
app.autodiscover_tasks()
```

**celery-worker.yaml**

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

## 10. 가용성/안정성: HPA, PDB, NetworkPolicy

**HPA**

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

**PDB (업그레이드/장애 중 최소 1개 인스턴스 유지)**

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

**NetworkPolicy (Ingress Controller만 접근 허용 예시)**

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
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - port: 8000
          protocol: TCP
```

---

## 11. 보안/운영 팁

- **SecurityContext**: 모든 컨테이너에서 `runAsNonRoot`, `allowPrivilegeEscalation: false`, capabilities drop.
- **Secret 관리**: K8s Secret + 외부 비밀관리(예: SOPS, External Secrets Operator).
- **데이터 백업**: Postgres `pg_dump`/스냅샷, 복구 리허설.
- **모니터링/로깅**: Liveness/Readiness 이벤트, Nginx Ingress 로그, 애플리케이션 구조화 로그(JSON), 메트릭(요청/지연/오류율).
- **배포전략**: Ingress Canary 혹은 Argo Rollouts로 점진적 롤아웃.

---

## 12. Helm/Kustomize (요약)

- Helm values에 `image.tag`, `ingress.host`, `resources`, `env`를 파라미터화.
- Kustomize `base` + `overlays/dev,staging,prod` 로 환경 별 설정(Replica/HPA/Ingress Host/TLS).

---

## 13. 전체 적용 순서(샘플)

```bash
# 네임스페이스
kubectl apply -f ns.yaml

# 데이터 계층
kubectl apply -f postgres-secret.yaml
kubectl apply -f postgres-statefulset.yaml
kubectl apply -f redis.yaml

# 앱 설정
kubectl apply -f django-config.yaml

# 마이그레이션
kubectl apply -f django-migrate-job.yaml
kubectl wait --for=condition=complete job/django-migrate -n webapp --timeout=120s

# 웹 배포
kubectl apply -f django-web-deployment.yaml
kubectl apply -f django-ingress.yaml

# 가용성/보안
kubectl apply -f hpa.yaml
kubectl apply -f pdb.yaml
kubectl apply -f networkpolicy.yaml
```

---

## 14. 검증/운영 명령어

```bash
# 상태
kubectl -n webapp get pods,svc,ingress,cm,secret,pvc
kubectl -n webapp get events --sort-by=.lastTimestamp

# 상세/로그
kubectl -n webapp describe deploy django-web
kubectl -n webapp logs deploy/django-web -c web --tail=200

# 셸 진입
kubectl -n webapp exec -it deploy/django-web -- /bin/sh

# 연결 확인
kubectl -n webapp run -it netshoot --rm --image=nicolaka/netshoot -- /bin/sh
curl -I django-web.webapp.svc.cluster.local
nc -zv postgres.webapp.svc.cluster.local 5432
nc -zv redis.webapp.svc.cluster.local 6379
```

---

## 15. 정리 및 삭제

```bash
kubectl delete ns webapp
```

---

## 결론 요약

| 주제 | 핵심 |
|------|------|
| 상태 리소스 | PostgreSQL/Redis는 StatefulSet+PVC로 안정성 확보 |
| 앱 배포 | Django는 Deployment+Service, 프로브/리소스/보안 컨텍스트 |
| 자동화 | 마이그레이션 Job, InitContainer로 collectstatic |
| 노출/보안 | Ingress+TLS, NetworkPolicy, Secret/ConfigMap |
| 운영 | HPA/PDB/모니터링/백업, 점진 배포로 신뢰도 향상 |

---

## 참고

- Django: https://docs.djangoproject.com/
- PostgreSQL 이미지: https://hub.docker.com/_/postgres
- Redis 이미지: https://hub.docker.com/_/redis
- Gunicorn: https://docs.gunicorn.org/
- Kubernetes Docs: Workloads/StatefulSet/Ingress/Autoscaling/NetworkPolicy
- cert-manager: https://cert-manager.io/
- Argo Rollouts: https://argo-rollouts.readthedocs.io/
