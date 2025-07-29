---
layout: post
title: Docker - Kubernetes 클러스터에 Docker 이미지 배포
date: 2025-03-24 20:20:23 +0900
category: Docker
---
# 🚢 Kubernetes 클러스터에 Docker 이미지 배포하기

---

## 📌 핵심 흐름 요약

1. ✅ Docker 이미지 빌드
2. 🚀 이미지 Push (Docker Hub, Harbor 등)
3. 📄 Kubernetes 리소스 정의 (Deployment, Service 등)
4. 🔁 `kubectl`로 클러스터에 배포
5. 🌐 외부 노출 (NodePort, Ingress 등)

---

## ✅ 1. Docker 이미지 빌드 & Push

```bash
# 이미지 빌드
docker build -t yourname/myapp:latest .

# Docker Hub 로그인
docker login

# 이미지 푸시
docker push yourname/myapp:latest
```

> 이미지가 클러스터에서 접근 가능한 퍼블릭 or 프라이빗 Registry에 있어야 함

---

## 📦 2. Deployment 작성 (기본 예시)

```yaml
# myapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: yourname/myapp:latest
          ports:
            - containerPort: 80
```

배포 실행:

```bash
kubectl apply -f myapp-deployment.yaml
```

---

## 🌐 3. 서비스(Service)로 외부 노출

```yaml
# myapp-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 80
  type: NodePort  # 또는 LoadBalancer / ClusterIP
```

적용:

```bash
kubectl apply -f myapp-service.yaml
```

확인:

```bash
kubectl get svc myapp-service
```

---

## 🔐 4. Private Registry 이미지 사용 시

### 방법 1: `kubectl create secret` 사용

```bash
kubectl create secret docker-registry regcred \
  --docker-username=DOCKER_USER \
  --docker-password=DOCKER_PASS \
  --docker-server=https://index.docker.io/v1/
```

Deployment에서 사용:

```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

---

## 💡 5. 실전 팁

| 항목 | 설명 |
|------|------|
| 이미지 버전 | `latest` 대신 `v1.0.0`처럼 명시적 버전 사용 권장 |
| Rolling Update | Deployment로 자동 롤링 업데이트 지원 |
| Pod 상태 확인 | `kubectl get pods`, `kubectl describe pod`, `kubectl logs` |
| YAML 통합 | 여러 리소스를 하나의 `k8s.yaml`로 구성 가능 (`---`로 구분) |

---

## 🧪 실습: Flask 앱 배포 예시

```Dockerfile
# Dockerfile
FROM python:3.11-alpine
WORKDIR /app
COPY app.py .
RUN pip install flask
CMD ["python", "app.py"]
```

```python
# app.py
from flask import Flask
app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from Kubernetes!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

### 이미지 빌드 & 푸시

```bash
docker build -t yourname/flask-app:latest .
docker push yourname/flask-app:latest
```

### Kubernetes 배포

```yaml
# flask-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask
          image: yourname/flask-app:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  selector:
    app: flask-app
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
```

```bash
kubectl apply -f flask-deploy.yaml
```

```bash
kubectl get svc flask-service
```

접속 주소: `http://<노드 IP>:<NodePort>`

---

## 🌍 6. 외부 도메인 연결 (Ingress)

Ingress Controller 설치 후 아래와 같이 정의:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

---

## 🔁 Rolling Update & 배포 전략

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

→ pod 하나씩 새 이미지로 교체하며 무중단 배포

---

## 🛡️ 보안 팁

| 항목 | 설정 |
|------|------|
| Read-Only Root FS | `securityContext.readOnlyRootFilesystem: true` |
| Non-root | `runAsUser: 1000` 등 설정 |
| Secret | 환경변수 대신 `Secret` 리소스 사용 |
| Resource 제한 | `resources.limits`, `requests` 지정 |

---

## 📦 Helm으로 배포하기 (선택 사항)

```bash
helm create mychart
# values.yaml 수정 후
helm install myapp ./mychart
```

---

## 📚 참고 자료

- [kubectl 공식 명령어](https://kubernetes.io/docs/reference/kubectl/)
- [Kubernetes Deployment 가이드](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Docker Hub Private Registry](https://docs.docker.com/docker-hub/)
- [Helm 공식 문서](https://helm.sh/docs/)
