---
layout: post
title: Kubernetes - AI/ML 모델을 Kubernetes에서 운영하기
date: 2025-06-14 22:20:23 +0900
category: Kubernetes
---
# Kubernetes에서 AI/ML 모델 운영하기: 확장, 보안, 관측성 구현

## 개요: 목표와 범위

이 실습 가이드는 Kubernetes 환경에서 AI/ML 모델을 안정적으로 운영하기 위한 종합적인 접근 방식을 제시합니다. 핵심 목표는 모델 서빙부터 확장, 보안, 관측성까지 전체 운영 라이프사이클을 다루는 것입니다.

**주요 구성 요소:**
- **모델 서빙**: Flask 또는 FastAPI 기반 REST API 구현
- **Kubernetes 리소스**: Deployment, Service, Ingress, ConfigMap, Secret, PVC
- **스케일링**: HPA(CPU 기반) 및 KEDA(이벤트 기반) 자동 확장
- **관측성**: Prometheus 메트릭, OpenTelemetry 트레이싱
- **보안**: Pod Security Admission, SecurityContext, NetworkPolicy
- **배포 전략**: 카나리/블루-그린 배포 (Argo Rollouts/Ingress 활용)
- **성능 최적화**: Gunicorn/Uvicorn 설정, 프리워밍, 멀티 프로세스 처리
- **GPU 가속**: nvidia.com/gpu 리소스, 노드 선택, 톨러레이션

---

## 전체 아키텍처

```
사용자/클라이언트
      │
      ▼
   Ingress (+TLS/카나리) ──> (옵션) WAF/속도 제한
      │
      ▼
 Service (ClusterIP)
      │
      ▼
 Deployment (Pod: Flask/FastAPI + 모델)
      │                   │
      │                   ├─ ConfigMap/Secret (환경/비밀 정보)
      │                   ├─ PVC (대형 모델 파일, 캐시)
      │                   └─ 메트릭/트레이싱 (Prometheus/OpenTelemetry)
      │
      └─ HPA/KEDA (자동 확장)
```

---

## 1. 모델 준비: 학습 및 직렬화

```python
# train_model.py
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_iris
import joblib

# 데이터 로드 및 모델 학습
X, y = load_iris(return_X_y=True)
clf = RandomForestClassifier(n_estimators=200, random_state=42)
clf.fit(X, y)

# 모델 직렬화
joblib.dump(clf, 'model.pkl')
print("모델 저장 완료: model.pkl")
```

실행:
```bash
python train_model.py
```

> **대형 모델 처리**: 모델 파일이 큰 경우 PVC에 저장하거나, 시작 시 외부 저장소(S3/GCS)에서 InitContainer를 통해 다운로드하는 패턴을 권장합니다.

---

## 2. 컨테이너 이미지 구축

### requirements.txt
```
flask
scikit-learn
joblib
prometheus_client
gunicorn
numpy
```

### app.py (Flask 애플리케이션)
```python
from flask import Flask, request, jsonify
import joblib, numpy as np, os, time
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

MODEL_PATH = os.getenv("MODEL_PATH", "model.pkl")
PORT = int(os.getenv("PORT", "5000"))

app = Flask(__name__)
model = joblib.load(MODEL_PATH)

# Prometheus 메트릭 정의
REQ = Counter("inference_requests_total", "추론 요청 총 수", ["endpoint"])
LAT = Histogram("inference_latency_seconds", "추론 지연 시간", 
                buckets=(0.005, 0.01, 0.02, 0.05, 0.1, 0.2, 0.5, 1, 2, 5))

@app.route("/healthz")
def healthz():
    return "ok", 200

@app.route("/readyz")
def readyz():
    try:
        _ = model.n_features_in_
        return "ready", 200
    except Exception:
        return "not-ready", 500

@app.route("/metrics")
def metrics():
    return generate_latest(), 200, {"Content-Type": CONTENT_TYPE_LATEST}

@app.route("/predict", methods=["POST"])
def predict():
    REQ.labels(endpoint="predict").inc()
    start = time.time()
    
    body = request.get_json(force=True)
    inputs = np.array(body["inputs"]).reshape(1, -1)
    pred = model.predict(inputs).tolist()
    
    LAT.observe(time.time() - start)
    return jsonify({"prediction": pred})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=PORT)
```

### Dockerfile (멀티스테이지 빌드)
```dockerfile
FROM python:3.9-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM base AS runtime
WORKDIR /app
COPY --from=base /usr/local /usr/local
COPY app.py model.pkl .
ENV PORT=5000

# 프로덕션 환경에서는 Gunicorn 사용 권장
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "--threads", "2", "app:app"]
```

이미지 빌드 및 푸시:
```bash
docker build -t <docker_id>/ml-api:0.1.0 .
docker push <docker_id>/ml-api:0.1.0
```

> **버전 관리 팁**: Git SHA와 함께 버전 태그를 관리하세요 (예: `ml-api:0.1.0-<SHA>`).

---

## 3. Kubernetes 배포 구성

### ConfigMap 및 Secret
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ml-api-config
data:
  PORT: "5000"
  MODEL_PATH: "/models/model.pkl"
---
apiVersion: v1
kind: Secret
metadata:
  name: ml-api-secrets
type: Opaque
stringData:
  API_KEY: "local-dev-only-change-me"
```

### PVC (대형 모델 파일용)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ml-model-pvc
spec:
  accessModes: ["ReadOnlyMany"]
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
```

### Deployment 및 Service
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-api
  labels: { app: ml-api }
spec:
  replicas: 2
  selector:
    matchLabels: { app: ml-api }
  template:
    metadata:
      labels: { app: ml-api }
    spec:
      initContainers:
      - name: fetch-model
        image: curlimages/curl:8.10.1
        command: ["sh", "-c"]
        args:
          - |
            set -e
            echo "모델 다운로드 중..."
            curl -fSL "$MODEL_URL" -o /models/model.pkl
        env:
        - name: MODEL_URL
          valueFrom:
            secretKeyRef:
              name: ml-api-secrets
              key: MODEL_URL
        volumeMounts:
        - name: model-vol
          mountPath: /models

      containers:
      - name: api
        image: <docker_id>/ml-api:0.1.0
        ports: [{ containerPort: 5000 }]
        envFrom:
        - configMapRef: { name: ml-api-config }
        - secretRef: { name: ml-api-secrets }
        volumeMounts:
        - name: model-vol
          mountPath: /models
          readOnly: true
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
        livenessProbe:
          httpGet: { path: /healthz, port: 5000 }
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet: { path: /readyz, port: 5000 }
          initialDelaySeconds: 5
          periodSeconds: 5
        startupProbe:
          httpGet: { path: /healthz, port: 5000 }
          failureThreshold: 30
          periodSeconds: 2
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          runAsUser: 10001
          capabilities:
            drop: ["ALL"]
      volumes:
      - name: model-vol
        persistentVolumeClaim:
          claimName: ml-model-pvc
      securityContext:
        fsGroup: 10001
---
apiVersion: v1
kind: Service
metadata:
  name: ml-api-svc
  labels: { app: ml-api }
spec:
  selector: { app: ml-api }
  ports:
  - name: http
    port: 80
    targetPort: 5000
  type: ClusterIP
```

> **프로브 설정 팁**: 초기 로딩 시간을 고려하여 `startupProbe`를 충분히 관대하게 설정하고, `livenessProbe`와 `readinessProbe`를 적절히 조정하세요.

---

## 4. Ingress 및 TLS 구성

### NGINX Ingress 기본 구성
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ml-api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  rules:
  - host: ml.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ml-api-svc
            port: { number: 80 }
```

### cert-manager를 활용한 TLS 자동화
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http01
spec:
  acme:
    email: you@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef: { name: letsencrypt-key }
    solvers:
    - http01:
        ingress:
          class: nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ml-api-ingress-tls
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-http01"
spec:
  ingressClassName: nginx
  tls:
  - hosts: ["ml.your-domain.com"]
    secretName: ml-api-tls
  rules:
  - host: ml.your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ml-api-svc
            port: { number: 80 }
```

---

## 5. API 테스트

```bash
curl -X POST http://ml.local/predict \
  -H "Content-Type: application/json" \
  -d '{"inputs":[5.1,3.5,1.4,0.2]}'
```

예상 응답:
```json
{"prediction":[0]}
```

---

## 6. 자동 확장 구성

### HPA (CPU 기반)
```bash
kubectl autoscale deployment ml-api \
  --cpu-percent=60 --min=2 --max=8
kubectl get hpa
```

### KEDA (이벤트 기반 확장)
KEDA 설치 후 다음과 같이 ScaledObject를 정의할 수 있습니다:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: ml-api-keda
spec:
  scaleTargetRef:
    name: ml-api
  pollingInterval: 10
  cooldownPeriod: 60
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-server.monitoring.svc.cluster.local
      metricName: inference_requests_total
      threshold: '50'            # 초당 50건 요청 기준
      query: |
        sum(rate(inference_requests_total{endpoint="predict"}[1m]))
```

---

## 7. 관측성 구성

### Prometheus 메트릭 수집
Flask 애플리케이션에 이미 구현된 `/metrics` 엔드포인트를 Prometheus가 스크랩하도록 구성합니다.

### OpenTelemetry 통합 (선택사항)
```bash
pip install opentelemetry-api opentelemetry-sdk opentelemetry-instrumentation-flask
```

```python
# app.py 상단에 추가
from opentelemetry.instrumentation.flask import FlaskInstrumentor
FlaskInstrumentor().instrument_app(app)
```

OpenTelemetry Collector와 Jaeger/Tempo를 연동하여 분산 트레이싱을 구현할 수 있습니다.

---

## 8. 보안 강화

### Pod Security Admission
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ml-prod
  labels:
    pod-security.kubernetes.io/enforce: "restricted"
```

### NetworkPolicy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ml-api-allow-ingress
  namespace: ml-prod
spec:
  podSelector:
    matchLabels: { app: ml-api }
  ingress:
  - from:
    - namespaceSelector:
        matchLabels: { kubernetes.io/metadata.name: ingress-nginx }
    ports: [{ port: 5000, protocol: TCP }]
```

### 이미지 서명 및 검증
Cosign을 사용하여 컨테이너 이미지에 서명하고, OPA/Kyverno를 통해 검증 정책을 적용하는 것을 권장합니다.

---

## 9. 카나리 및 블루-그린 배포

### NGINX Ingress 기반 카나리 배포
```yaml
metadata:
  name: ml-api-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"  # 20% 트래픽 전환
```

### Argo Rollouts (권장)
Argo Rollouts를 사용하면 단계적 배포, 메트릭 기반 게이팅, 자동 롤백과 같은 고급 배포 전략을 구현할 수 있습니다.

---

## 10. 성능 및 신뢰성 최적화

1. **Gunicorn 워커/스레드 튜닝**: `--workers 2 --threads 2`와 같이 CPU 코어 및 메모리에 맞게 조정
2. **프리워밍**: 시작 시 더미 입력으로 모델 로딩 및 JIT 캐시 워밍업
3. **동시성 처리**: FastAPI + Uvicorn 조합으로 고동시성 HTTP 요청 처리 향상
4. **대형 모델 처리**: 서버 프로세스 preload, 공유 메모리(읽기 전용) 활용
5. **구조화된 로깅**: JSON 형식 로깅 및 수명주기 이벤트 기록
6. **리소스 제한**: 메모리 누수 방지를 위한 requests/limits 명시적 설정

---

## 11. 용량 및 성능 추정

### 필요한 파드 수 계산
```
N ≈ ⌈(λ × S) / c⌉
```
여기서:
- λ: 초당 요청 수 (req/s)
- S: 평균 처리 시간 (초)
- c: 파드당 동시 처리 가능 수

### 예상 응답 시간
```
ERT ≈ p_cold × T_cold + (1 - p_cold) × T_warm
```
여기서:
- p_cold: 콜드 스타트 비율
- T_cold: 콜드 스타트 지연 시간
- T_warm: 웜 스타트 지연 시간

**최적화 전략**: 최소 레플리카 수 유지로 콜드 스타트 비율 감소, 런타임 최적화로 콜드 스타트 시간 단축

---

## 12. GPU 가속 지원

NVIDIA GPU Operator 설치 후:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-gpu-api
spec:
  replicas: 1
  selector: { matchLabels: { app: ml-gpu } }
  template:
    metadata: { labels: { app: ml-gpu } }
    spec:
      nodeSelector: { "nvidia.com/gpu.present": "true" }
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: api
        image: <docker_id>/ml-gpu-api:0.1.0
        resources:
          limits:
            nvidia.com/gpu: 1
            cpu: "2"
            memory: "8Gi"
        env:
        - name: TORCH_CUDA_ARCH_LIST
          value: "8.0+PTX"
```

> **GPU 활용 팁**: 단순한 파드 수 증가보다 배치 처리 및 세션 고정을 통한 GPU 활용률 향상에 중점을 두세요. Triton Inference Server, TorchServe, TensorFlow Serving과 같은 전용 모델 서빙 솔루션도 고려해보세요.

---

## 13. FastAPI 대안 (고성능)

### requirements.txt
```
fastapi
uvicorn[standard]
scikit-learn
joblib
prometheus_client
numpy
```

### main.py
```python
from fastapi import FastAPI
from pydantic import BaseModel
import joblib, numpy as np
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from starlette.responses import Response
import time

class PredictIn(BaseModel):
    inputs: list[float]

REQ = Counter("inference_requests_total", "추론 요청 총 수", ["endpoint"])
LAT = Histogram("inference_latency_seconds", "추론 지연 시간")

app = FastAPI()
model = joblib.load("model.pkl")

@app.get("/healthz")
def healthz(): return {"ok": True}

@app.get("/readyz")
def readyz(): return {"ready": hasattr(model, "n_features_in_")}

@app.get("/metrics")
def metrics(): return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)

@app.post("/predict")
def predict(body: PredictIn):
    REQ.labels(endpoint="predict").inc()
    start = time.time()
    pred = model.predict(np.array(body.inputs).reshape(1, -1)).tolist()
    LAT.observe(time.time() - start)
    return {"prediction": pred}
```

### Docker 명령어
```dockerfile
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "5000", "--workers", "2"]
```

---

## 14. 배포 관리 전략

- **Kustomize**: `overlays/dev`, `overlays/prod`를 통해 환경별 설정 차등 관리
- **Helm**: values 파일을 통해 HPA 임계값, Ingress 호스트, TLS 설정, 리소스 제한 등 파라미터화
- **ArgoCD**: Git 기반 선언적 배포, 자동 동기화, 드리프트 복구, 롤백 지원

---

## 15. 리소스 정리

```bash
kubectl delete -f deployment_service_ingress.yaml
kubectl delete hpa ml-api
kubectl delete pvc ml-model-pvc
kubectl delete cm ml-api-config
kubectl delete secret ml-api-secrets
```

---

## 예제 프로젝트 구조

```
.
├─ app/                 # 애플리케이션 소스 코드
│  ├─ app.py / main.py
│  ├─ requirements.txt
│  └─ Dockerfile
├─ model/
│  └─ model.pkl
├─ k8s/
│  ├─ config-secret.yaml
│  ├─ pvc.yaml
│  ├─ deployment.yaml
│  ├─ service.yaml
│  └─ ingress.yaml
├─ kustomize/
│  ├─ base/ ...
│  └─ overlays/ (dev/prod) ...
└─ .github/workflows/deploy.yml (CI/CD 파이프라인)
```

---

## 운영 모범 사례 요약

1. **가용성 보장**: 최소 2개 이상의 레플리카, 적절한 readiness/liveness/startup 프로브 설정
2. **스케일링 전략**: HPA(기본 메트릭), KEDA(이벤트 기반), 최소 레플리카로 콜드 스타트 감소
3. **관측성 강화**: Prometheus 메트릭, 구조화된 로깅, OpenTelemetry 트레이싱
4. **보안 강화**: Pod Security Admission, SecurityContext, NetworkPolicy, 이미지 서명
5. **성능 최적화**: 적절한 워커/스레드 구성, 프리워밍, 모델 로딩 최적화
6. **배포 전략**: 카나리/블루-그린 배포, GitOps 자동화
7. **스토리지 관리**: PVC 또는 외부 저장소를 통한 모델 배포
8. **GPU 활용**: 적절한 리소스 제한, 배치 처리 최적화, 전용 모델 서빙 솔루션 고려

---

## 결론

이 가이드는 AI/ML 모델의 학습부터 Kubernetes 환경에서의 운영까지 전체 라이프사이클을 종합적으로 다루고 있습니다. 제공된 템플릿과 패턴을 활용하면 소규모 모델도 몇 시간 안에 프로덕션 수준의 환경에 배포할 수 있습니다.

성공적인 AI/ML 모델 운영을 위해 다음 단계를 고려해보세요:

1. **기본 인프라 구축**: 이 가이드의 템플릿을 기반으로 기본적인 서빙 환경 구성
2. **GitOps 도입**: 배포 프로세스의 일관성과 재현성 확보
3. **카나리 배포 구현**: 점진적 롤아웃으로 배포 위험 최소화
4. **SLO 기반 운영**: 관측 메트릭을 기반으로 서비스 수준 목표 설정 및 모니터링
5. **팀 표준화**: 조직 전체가 일관된 방식으로 모델을 운영할 수 있는 표준 프로세스 수립

이러한 단계적 접근 방식을 통해 AI/ML 모델의 운영을 체계적으로 성숙시킬 수 있으며, 결국 더 안정적이고 효율적인 모델 서빙 인프라를 구축할 수 있습니다.