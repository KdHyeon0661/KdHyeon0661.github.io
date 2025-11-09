---
layout: post
title: Kubernetes - AI/ML 모델을 Kubernetes에서 운영하기
date: 2025-06-14 22:20:23 +0900
category: Kubernetes
---
# 실습: AI/ML 모델을 Kubernetes에서 운영하기 — 확장·보안·관측까지

## ✅ 목표와 범위

- **모델 서빙**: Flask 기반 REST API(추가로 FastAPI 대안 제공)
- **K8s 리소스**: Deployment/Service/Ingress/TLS, Config/Secret/PVC
- **스케일링**: HPA(CPU) + KEDA(요청 수/큐 길이) 예제
- **관측**: Prometheus 지표, OpenTelemetry 트레이싱(선택)
- **보안/격리**: Pod Security, SecurityContext, NetworkPolicy
- **전략**: Canary/Blue-Green(Argo Rollouts/Ingress), 버전 라우팅
- **성능**: Gunicorn+Uvicorn, 프리워밍, 멀티 프로세스
- **GPU**: nvidia.com/gpu, nodeSelector, tolerations

---

## 🧱 전체 아키텍처

```plaintext
사용자/클라이언트
      │
      ▼
   Ingress (+TLS/Canary) ──> (옵션) WAF/RateLimit
      │
      ▼
 Service (ClusterIP)
      │
      ▼
 Deployment (Pod: Flask/FastAPI + Model)
      │                   │
      │                   ├─ ConfigMap/Secret (환경/민감정보)
      │                   ├─ PVC (대형 모델 파일, 캐시)
      │                   └─ Metrics/Tracing (Prometheus/OTel)
      │
      └─ HPA/KEDA (오토스케일)
```

---

## 📦 1. 모델 준비 — 학습 및 직렬화(Joblib)

```python
# train_model.py
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_iris
import joblib

X, y = load_iris(return_X_y=True)
clf = RandomForestClassifier(n_estimators=200, random_state=42)
clf.fit(X, y)

joblib.dump(clf, 'model.pkl')
print("Saved: model.pkl")
```

```bash
python train_model.py
```

> 대형 모델은 **PVC**에 올리거나, **시작 시 외부 저장소(S3/GCS)**에서 가져와 **InitContainer**로 마운트하는 패턴을 추천합니다(아래 예제 포함).

---

## 🐳 2. 컨테이너 이미지 — Flask + 프로메테우스 지표 + 예측 API

**requirements.txt**
```
flask
scikit-learn
joblib
prometheus_client
gunicorn
numpy
```

**app.py (Flask + /metrics + 예측 API + 헬스체크)**

```python
from flask import Flask, request, jsonify
import joblib, numpy as np, os, time
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

MODEL_PATH = os.getenv("MODEL_PATH", "model.pkl")
PORT = int(os.getenv("PORT", "5000"))

app = Flask(__name__)
model = joblib.load(MODEL_PATH)

REQ = Counter("inference_requests_total", "Total inference requests", ["endpoint"])
LAT = Histogram("inference_latency_seconds", "Latency per inference", buckets=(0.005,0.01,0.02,0.05,0.1,0.2,0.5,1,2,5))

@app.route("/healthz")
def healthz():
    return "ok", 200

@app.route("/readyz")
def readyz():
    # 간단한 모델 로딩 검증
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
    # 개발용: gunicorn을 쓰면 멀티프로세스/워커로 이득
    app.run(host="0.0.0.0", port=PORT)
```

**Dockerfile (멀티스테이지 빌드 권장)**
```Dockerfile
FROM python:3.9-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM base AS runtime
WORKDIR /app
COPY --from=base /usr/local /usr/local
COPY app.py model.pkl . 
ENV PORT=5000
# 프로덕션: Gunicorn + gevent/uvicorn workers (Flask→FastAPI 전환시 uvicorn 워커)
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "--threads", "2", "app:app"]
```

이미지 빌드·푸시:
```bash
docker build -t <docker_id>/ml-api:0.1.0 .
docker push <docker_id>/ml-api:0.1.0
```

> **팁**: 버전 태그를 Git SHA와 함께 관리하세요. `ml-api:0.1.0-<SHA>`.

---

## 🚀 3. Kubernetes 배포 — Deployment/Service/Config/Secret/Probe/Resource

### 3.1 ConfigMap/Secret (환경·민감정보 분리)

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

### 3.2 PVC(선택: 대형 모델/캐시)

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

> 오브젝트 스토리지에서 **InitContainer**로 모델을 내려받아 PVC에 저장하는 패턴도 유용합니다.

### 3.3 Deployment + Service

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
      # (옵션) InitContainer: 외부에서 모델 가져오기
      initContainers:
      - name: fetch-model
        image: curlimages/curl:8.10.1
        command: ["sh","-c"]
        args:
          - |
            set -e
            echo "Downloading model..."
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

> Probe를 **관대하게** 잡고, `startupProbe`를 통해 **초기 로딩 지연**을 견딜 수 있게 하는 것이 안정화에 중요합니다.

---

## 🌐 4. Ingress + TLS — 실서비스 노출

### 4.1 NGINX Ingress (기본)

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

호스트 맵:
```plaintext
127.0.0.1 ml.local
```

### 4.2 cert-manager를 이용한 TLS(실서비스)

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

## 🧪 5. 호출 테스트

```bash
curl -X POST http://ml.local/predict \
  -H "Content-Type: application/json" \
  -d '{"inputs":[5.1,3.5,1.4,0.2]}'
```

예상:
```json
{"prediction":[0]}
```

---

## 📈 6. 오토스케일링 — HPA(기본) + KEDA(이벤트)

### 6.1 HPA(CPU 기반)
```bash
kubectl autoscale deployment ml-api \
  --cpu-percent=60 --min=2 --max=8
kubectl get hpa
```

> CPU/메모리만으로 예측부하를 정확히 반영하긴 어려울 수 있습니다. **KEDA**로 큐 길이/요청 수 기반 확장도 함께 고려하세요.

### 6.2 KEDA(요청 수·큐 기반, 예: Prometheus Scaler)

**KEDA 설치 후**, 아래처럼 `ScaledObject`를 정의하면 Prometheus 지표(초당 요청 수 등)에 맞춰 스케일합니다.

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
      threshold: '50'            # 초당 요청 50건 기준
      query: |
        sum(rate(inference_requests_total{endpoint="predict"}[1m]))
```

---

## 📊 7. 관측(Observability) — Prometheus, Grafana, OpenTelemetry

- **/metrics** 엔드포인트로 Prometheus 스크랩 (위 Flask 예제 포함)
- Grafana로 대시보드(Pod QPS, p95 지연, Error rate) 구성
- 분산 트레이싱이 필요하면 **OpenTelemetry**로 **Flask/FastAPI**에 tracer 설치

**OpenTelemetry(선택, 간단 예시)**
```bash
pip install opentelemetry-api opentelemetry-sdk opentelemetry-instrumentation-flask
```

```python
# app.py 상단
from opentelemetry.instrumentation.flask import FlaskInstrumentor
FlaskInstrumentor().instrument_app(app)
```

Collector/Jaeger/Tempo 연동은 환경에 맞춰 배치(Helm 차트 추천).

---

## 🔐 8. 보안·격리 — SecurityContext, Pod Security, NetworkPolicy

### 8.1 Pod Security(네임스페이스 레벨)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ml-prod
  labels:
    pod-security.kubernetes.io/enforce: "restricted"
```

### 8.2 NetworkPolicy(내부 통신만 허용)
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

### 8.3 이미지서명/검증(권장)
- Cosign으로 서명 → OPA/Kyverno로 검증 정책 적용

---

## 🧪 9. 카나리/블루그린 — 점진배포·롤백

### 9.1 NGINX Ingress 기반 간단 Canary(가중치)
```yaml
metadata:
  name: ml-api-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"  # 20% 트래픽
```

### 9.2 Argo Rollouts(권장)
- Rollout CR로 **Step(10%→30%→50%→100%)** 정의, 메트릭 게이팅(Prometheus), 자동 롤백

---

## ⚡ 10. 성능·신뢰성 튜닝

- **Gunicorn 워커/스레드** 조정(코어·메모리 대비): `--workers 2 --threads 2`
- **프리워밍(warm-up)**: 시작 후 준비단계에서 더미 입력으로 모델 로딩·JIT 캐시
- **동시성**: FastAPI(+Uvicorn/ASGI) 전환 시 고동시성 HTTP에 유리
- **대형 모델**: 서버 프로세스 **preload**, **shared memory**(읽기 전용) 전략
- **로깅**: 구조화(JSON) 로깅 + 수명주기 이벤트 기록
- **리소스 가드**: requests/limits를 확실히(메모리 누수 대비), OOMKilled 모니터

---

## 🧮 11. 간단 용량·지연 추정

요청율 \(\lambda\) (req/s), 평균 처리시간 \(S\) (s), Pod당 동시성 \(c\) 일 때 필요한 Pod 수 \(N\) 근사:
$$
N \approx \left\lceil \frac{\lambda \cdot S}{c} \right\rceil
$$

콜드 스타트 비율 \(p_{cold}\), 콜드/웜 지연 \(T_{cold}, T_{warm}\) 일 때 기대 응답시간:
$$
ERT \approx p_{cold} \cdot T_{cold} + (1-p_{cold}) \cdot T_{warm}
$$

> **Warming/MinReplicas** 로 \(p_{cold}\) 낮추고, **런타임 최적화**로 \(T_{cold}\) 를 줄입니다.

---

## 🧲 12. GPU 서빙(선택)

NVIDIA 플러그인 설치 후:

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

> GPU는 **Pod 수보다 세션 고정/배치 처리**가 중요할 수 있습니다.  
> Triton Inference Server, TorchServe, TF Serving로 전환 검토.

---

## 🧱 13. FastAPI 대안 (高동시성/스키마/문서화)

**requirements.txt (대체)**
```
fastapi
uvicorn[standard]
scikit-learn
joblib
prometheus_client
numpy
```

**main.py**
```python
from fastapi import FastAPI
from pydantic import BaseModel
import joblib, numpy as np
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from starlette.responses import Response

class PredictIn(BaseModel):
    inputs: list[float]

REQ = Counter("inference_requests_total", "Total inference requests", ["endpoint"])
LAT = Histogram("inference_latency_seconds", "Latency per inference")

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
    import time
    REQ.labels(endpoint="predict").inc()
    st = time.time()
    pred = model.predict(np.array(body.inputs).reshape(1,-1)).tolist()
    LAT.observe(time.time()-st)
    return {"prediction": pred}
```

**Docker CMD(예시)**:
```Dockerfile
CMD ["uvicorn","main:app","--host","0.0.0.0","--port","5000","--workers","2"]
```

---

## 🧰 14. Kustomize/Helm, GitOps(ArgoCD)로 배포 관리

- **Kustomize**: `overlays/dev`, `overlays/prod` 로 이미지 태그·Replica·리소스 차등 관리
- **Helm**: values로 HPA 임계치, Ingress host, TLS, 리소스 등 파라미터화
- **ArgoCD**: Git push → 자동 동기화, 드리프트 자동 복구, 롤백 이력

---

## 🧼 15. 정리·삭제

```bash
kubectl delete -f deployment_service_ingress.yaml
kubectl delete hpa ml-api
kubectl delete pvc ml-model-pvc
kubectl delete cm ml-api-config
kubectl delete secret ml-api-secrets
```

---

## 📁 예제 리포지토리 구조(권장)

```plaintext
.
├─ app/                 # 앱 소스(Flask/FastAPI)
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
└─ .github/workflows/deploy.yml (선택: CI/CD)
```

---

## ✅ 실전 체크리스트 (현장 요약)

- **가용성**: replicas≥2, readiness/liveness/startup probe 설정
- **스케일링**: HPA(기본), KEDA(이벤트/지표), minReplicas로 콜드 감소
- **관측**: /metrics(프로메테우스), 로깅(JSON), 트레이싱(OTel)
- **보안**: Pod Security, SecurityContext(runAsNonRoot), NetworkPolicy, 이미지서명
- **성능**: Gunicorn/uvicorn 워커, Pre-warm, 대형모델 로딩 최적화
- **배포전략**: Canary/Blue-Green(Argo Rollouts), Helm/Kustomize/ArgoCD
- **스토리지**: PVC/InitContainer로 모델 공급(or 외부 저장소)
- **GPU**: nvidia.com/gpu limits, 스케줄링 정책, 배치/세션 설계

---

## 📚 참고 링크

- Kubernetes — Workloads/Autoscaling/Ingress
- Prometheus & Grafana — Metrics & Dashboard
- KEDA — Event-driven Autoscaling
- cert-manager — TLS 자동화
- ArgoCD & Argo Rollouts — GitOps & Progressive Delivery
- NVIDIA GPU Operator / Triton Inference Server
- TorchServe / TensorFlow Serving

---

## 🔚 마무리

이 가이드는 **학습→서빙→운영**으로 이어지는 실전 플로우를 “**단일 문서**”로 엮었습니다.  
여기 나온 템플릿을 **복사-수정**만 해도, **작은 모델은 수 시간 내** 안정적으로 프로덕션 느낌의 환경에 올릴 수 있을 것입니다.  
그다음 단계로는 **GitOps·Canary·관측 지표 기반 SLO**를 도입해, 팀 전체가 **일관된 방식**으로 모델을 운영하도록 확장해보세요.