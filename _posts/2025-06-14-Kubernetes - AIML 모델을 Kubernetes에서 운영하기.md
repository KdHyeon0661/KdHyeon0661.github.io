---
layout: post
title: Kubernetes - 실습 : AI/ML 모델을 Kubernetes에서 운영하기
date: 2025-06-13 22:20:23 +0900
category: Kubernetes
---
# 🧠 실습: AI/ML 모델을 Kubernetes에서 운영하기

AI/ML 모델 개발이 완료된 후, 가장 큰 과제는 **안정적이고 유연한 운영 환경 구축**입니다. Kubernetes는 AI 모델을 **서버리스처럼 배포하고 확장하며, 자원을 효율적으로 관리**할 수 있는 강력한 플랫폼입니다.

이번 실습에서는 **사전 학습된 머신러닝 모델을 Kubernetes에 서빙**하고, 이를 외부에서 호출하는 구조를 구현합니다.

---

## ✅ 실습 목표

- 학습된 AI 모델을 Docker 이미지로 감싸고 배포
- Flask + Pickle 기반 API 구성
- Kubernetes에서 Deployment + Service + Ingress 구현
- Resource Limit 및 Auto-scaling 적용

---

## 🧱 전체 아키텍처

```plaintext
사용자 → Ingress → Service → Pod (Flask + ML 모델)
                          ↕
                     PersistentVolume (모델 저장)
```

---

## 📦 1. 모델 준비

### 모델 학습 및 저장 예 (scikit-learn)

```python
# train_model.py
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_iris
import joblib

X, y = load_iris(return_X_y=True)
clf = RandomForestClassifier()
clf.fit(X, y)

joblib.dump(clf, 'model.pkl')
```

```bash
python train_model.py
```

---

## 🐳 2. Dockerfile 작성

```Dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY model.pkl .
COPY app.py .

CMD ["python", "app.py"]
```

### `requirements.txt`

```
flask
scikit-learn
joblib
```

### `app.py` (Flask API)

```python
from flask import Flask, request, jsonify
import joblib
import numpy as np

app = Flask(__name__)
model = joblib.load("model.pkl")

@app.route("/predict", methods=["POST"])
def predict():
    data = request.get_json(force=True)
    inputs = np.array(data["inputs"]).reshape(1, -1)
    prediction = model.predict(inputs)
    return jsonify({"prediction": prediction.tolist()})
```

### 이미지 빌드 및 업로드

```bash
docker build -t <username>/ml-api:latest .
docker push <username>/ml-api:latest
```

---

## 🚀 3. Kubernetes 배포 파일 작성

### `ml-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ml-api
  template:
    metadata:
      labels:
        app: ml-api
    spec:
      containers:
        - name: ml-api
          image: <username>/ml-api:latest
          ports:
            - containerPort: 5000
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: ml-api-service
spec:
  selector:
    app: ml-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
```

```bash
kubectl apply -f ml-deployment.yaml
```

---

## 🌐 4. Ingress로 외부에 공개

### `ml-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ml-api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: ml.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ml-api-service
                port:
                  number: 80
```

```bash
kubectl apply -f ml-ingress.yaml
```

📌 `/etc/hosts` 에 아래 라인 추가:

```plaintext
127.0.0.1 ml.local
```

---

## 🧪 5. 테스트: 예측 호출

```bash
curl -X POST http://ml.local/predict \
  -H "Content-Type: application/json" \
  -d '{"inputs": [5.1, 3.5, 1.4, 0.2]}'
```

```json
{"prediction": [0]}
```

---

## 📈 6. Horizontal Pod Autoscaler 설정

```bash
kubectl autoscale deployment ml-api \
  --cpu-percent=50 \
  --min=1 \
  --max=5
```

확인:

```bash
kubectl get hpa
```

로드 테스트 후 자동 스케일링 되는지 확인할 수 있습니다.

---

## 📊 7. 모니터링 연동 (선택)

- `Prometheus + Grafana`로 Flask API 모니터링
- `Kubernetes Dashboard`나 `Lens`로 Pod 상태 확인
- `kube-metrics-server` 설치 필요

---

## 📁 전체 구성 정리

```plaintext
- Dockerfile
- model.pkl
- app.py
- ml-deployment.yaml
- ml-ingress.yaml
```

---

## 💡 실전 응용 아이디어

| 응용 시나리오 | 설명 |
|----------------|------|
| REST API 기반 모델 서빙 | Flask → FastAPI로 확장 |
| 모델 버전 관리 | `/v1`, `/v2` 등의 경로로 Ingress 라우팅 |
| A/B 테스트 | 두 모델을 각각 배포하고 트래픽 분할 |
| GPU 서빙 | nodeSelector + nvidia runtime 적용 |
| TensorFlow Serving / TorchServe | 전용 서빙 솔루션으로 교체 가능 |

---

## 🧼 삭제 정리

```bash
kubectl delete -f ml-deployment.yaml
kubectl delete -f ml-ingress.yaml
kubectl delete hpa ml-api
```

---

## 📚 참고 링크

- [Kubernetes Python 모델 서빙](https://kubernetes.io/docs/tutorials/)
- [Joblib + Flask 모델 예시](https://scikit-learn.org/)
- [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

---

## ✅ 마무리 요약

- 학습된 모델을 Flask API로 감싸 Docker 이미지로 만들고 배포
- Kubernetes에서 안정적이고 유연하게 운영 가능
- Ingress로 외부 서비스 제공, HPA로 부하 대응
- 실무 환경에 확장 적용 가능 (CI/CD, GPU, 버전 관리 등)