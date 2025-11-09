---
layout: post
title: Docker - Kubernetes 클러스터에 Docker 이미지 배포
date: 2025-03-24 20:20:23 +0900
category: Docker
---
# Kubernetes 클러스터에 Docker 이미지 배포하기

- 이미지 버전 고정(다이제스트), 프라이빗 레지스트리 인증, 멀티 컨테이너/사이드카
- Readiness/Liveness/Startup Probe, 리소스 요청/제한, HPA/VPA, PDB
- ConfigMap/Secret, RBAC/ServiceAccount, NetworkPolicy
- Service 타입별(ClusterIP/NodePort/LoadBalancer), Ingress + TLS
- Blue-Green/Canary 롤아웃, 롤백/이력, 배포 전략 파라미터
- Helm/Kustomize, 네임스페이스 전략, 운영/장애 대응 체크리스트

---

## 0. 핵심 흐름 복습

1) Docker 이미지 빌드  
2) 레지스트리에 Push(Docker Hub/Harbor/ECR/GHCR 등)  
3) Kubernetes 리소스 정의(Deployment/Service/Ingress/ConfigMap/Secret/SA 등)  
4) `kubectl`로 적용 또는 Helm/Kustomize로 배포  
5) 노출/스케일/보안/관측/롤백 운영

---

## 1. Docker 이미지 빌드 & Push

```bash
# 1) 이미지 빌드
docker build -t yourname/myapp:1.0.0 .

# 2) 로그인(Docker Hub 예시)
docker login

# 3) 태그 추가(가독 태그 + 불변 태그)
docker tag yourname/myapp:1.0.0 yourname/myapp:latest

# 4) 푸시
docker push yourname/myapp:1.0.0
docker push yourname/myapp:latest
```

### 1.1 다이제스트 고정 이미지(권장)
K8s 매니페스트에는 태그 대신 **다이제스트**를 고정 사용하면 재현성이 좋아진다.

{% raw %}
```bash
# 다이제스트 조회
docker pull yourname/myapp:1.0.0
docker inspect --format='{{index .RepoDigests 0}}' yourname/myapp:1.0.0
# 출력 예: yourname/myapp@sha256:abcdef...
```
{% endraw %}

배포 시:
```yaml
image: yourname/myapp@sha256:abcdef...   # 태그 대신 다이제스트
```

---

## 2. 네임스페이스 & 기본 권장 설정

```bash
kubectl create namespace prod
kubectl config set-context --current --namespace=prod
```

권장 애드온/설정:
- Admission(예: OPA/Gatekeeper/Kyverno)로 **이미지 서명/다이제스트/리소스 제한** 정책 게이트
- Metrics Server(HPA), Ingress Controller(NGINX/Contour/Traefik), Cert-Manager(TLS 자동화)

---

## 3. 기초 Deployment 매니페스트

```yaml
# myapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels: { app: myapp }
spec:
  replicas: 2
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # 추가 생성 허용
      maxUnavailable: 0     # 가용성 유지
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp }
    spec:
      containers:
        - name: myapp
          image: yourname/myapp:1.0.0   # 또는 다이제스트 권장
          imagePullPolicy: IfNotPresent  # 또는 Always(태그 최신 갱신 시)
          ports:
            - containerPort: 80
          env:
            - name: RUNTIME_ENV
              value: "prod"
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet: { path: /healthz, port: 80 }
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:
            httpGet: { path: /livez, port: 80 }
            initialDelaySeconds: 10
            periodSeconds: 10
          startupProbe:
            httpGet: { path: /startupz, port: 80 }
            failureThreshold: 30
            periodSeconds: 3
      terminationGracePeriodSeconds: 30
```

> **Probes**: 레디니스는 트래픽 수신 가능 상태, 라이브니스는 프로세스 생존 확인, 스타트업은 초기 부팅 안정화.

---

## 4. 서비스(Service) 노출

```yaml
# myapp-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector: { app: myapp }
  ports:
    - name: http
      port: 80         # 클러스터 내부 포트
      targetPort: 80   # 컨테이너 포트
  type: ClusterIP      # 내부 통신(default)
```

외부 노출 옵션:
- **NodePort**: 디버그/온프레미스 간단 노출
- **LoadBalancer**: 클라우드 L4 LB 할당
- **Ingress**: L7 라우팅(도메인/경로/TLS)

```yaml
# NodePort 예시
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080   # 30000-32767
```

---

## 5. Ingress + TLS(권장)

Ingress Controller(NGINX 등) 설치 후:

```yaml
# myapp-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
    - hosts: [ myapp.example.com ]
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port: { number: 80 }
```

**Cert-Manager** 자동 TLS 예시(요약):
- ClusterIssuer(ACME/Let’s Encrypt) 생성
- Ingress의 `cert-manager.io/cluster-issuer: letsencrypt` 주석 추가

---

## 6. 프라이빗 레지스트리 이미지 Pull

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=DOCKER_USER \
  --docker-password=DOCKER_PASS \
  --namespace=prod
```

Deployment에:

```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

또는 ServiceAccount에 부여(모든 Pod에 상속):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
imagePullSecrets:
  - name: regcred
---
apiVersion: apps/v1
kind: Deployment
# ...
spec:
  template:
    spec:
      serviceAccountName: app-sa
```

---

## 7. 구성/비밀 — ConfigMap & Secret

```yaml
# config
apiVersion: v1
kind: ConfigMap
metadata: { name: myapp-config }
data:
  APP_MODE: production
  APP_REGION: ap-northeast-2
---
# secret (base64 인코딩 but k8s 저장소는 평문에 가까움 → KMS/SealedSecret/Vault 고려)
apiVersion: v1
kind: Secret
metadata: { name: myapp-secret }
type: Opaque
data:
  DB_PASSWORD: c2VjdXJlX3Bhc3M=   # echo -n 'secure_pass' | base64
```

컨테이너에 주입:

```yaml
envFrom:
  - configMapRef: { name: myapp-config }
  - secretRef: { name: myapp-secret }
```

또는 파일 마운트:

```yaml
volumeMounts:
  - name: app-config
    mountPath: /etc/myapp
volumes:
  - name: app-config
    configMap:
      name: myapp-config
      items:
        - key: APP_MODE
          path: app_mode
```

---

## 8. 리소스/스케일 — HPA/VPA/PDB

### 8.1 HPA(수평 오토스케일)
Metrics Server 필요.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: myapp-hpa }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
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

### 8.2 PDB(중단 예산)
노드 업그레이드 등 동안 최소 가용성 보장.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: myapp-pdb }
spec:
  minAvailable: 1
  selector:
    matchLabels: { app: myapp }
```

### 8.3 대략 용량 산정(간이)
요청 기준으로 노드 수 산정:
$$
\text{노드수} \approx \left\lceil \frac{\sum_{i=1}^{N} \text{pod}_i\_\text{cpu\_request}}{\text{노드당 vCPU}} \cdot \frac{1}{\alpha} \right\rceil
$$
- \(\alpha\): 여유율(예: 0.7). 메모리도 동일 방식으로 별도 산출하여 큰 값 채택.

---

## 9. 네트워크/보안 — RBAC/SA/NetworkPolicy/SecContext

### 9.1 RBAC + ServiceAccount
필요 최소 권한 부여.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: config-reader }
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: config-reader-bind }
subjects:
  - kind: ServiceAccount
    name: app-sa
roleRef:
  kind: Role
  name: config-reader
  apiGroup: rbac.authorization.k8s.io
```

### 9.2 NetworkPolicy(기본 거부 + 화이트리스트)

```yaml
# 같은 네임스페이스 내 myapp로 향하는 80/tcp만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: myapp-allow-frontend }
spec:
  podSelector: { matchLabels: { app: myapp } }
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector: { matchLabels: { role: frontend } }
      ports:
        - protocol: TCP
          port: 80
```

### 9.3 SecurityContext(비루트/읽기전용 fs/드롭 Cap)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

---

## 10. 멀티 컨테이너/사이드카 패턴(예: Reverse Proxy, 로그 셰퍼)

```yaml
spec:
  containers:
    - name: app
      image: yourname/myapp:1.0.0
      ports: [{containerPort: 8080}]
    - name: envoy
      image: envoyproxy/envoy:v1.30-latest
      ports: [{containerPort: 80}]
      args: ["-c","/etc/envoy/envoy.yaml"]
      volumeMounts:
        - name: envoy-config
          mountPath: /etc/envoy
  volumes:
    - name: envoy-config
      configMap: { name: envoy-config }
```

---

## 11. Blue-Green/Canary 배포

### 11.1 Blue-Green(서비스 스위치)
- `myapp-blue`, `myapp-green` 두 Deployment
- `Service.selector`를 원하는 색으로 전환 → 무중단 릴리스/롤백

```yaml
# Service selector만 바꾸면 트래픽 전환
spec:
  selector: { app: myapp, color: blue }  # → green 으로 교체 시 전환
```

### 11.2 Canary(NGINX Ingress 가중치)
- canary 어노테이션으로 일부 비율만 신버전에 라우팅

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
```

---

## 12. 롤아웃/이력/롤백

```bash
# 진행 상황
kubectl rollout status deploy/myapp

# 이력
kubectl rollout history deploy/myapp

# 특정 리비전 상세
kubectl rollout history deploy/myapp --revision=3

# 롤백
kubectl rollout undo deploy/myapp --to-revision=3
```

문제 발생 시 즉시 이전 리비전으로 복귀 가능.

---

## 13. 관측성 — 로그/지표/트레이싱 핸드북

### 13.1 빠른 명령어
```bash
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs -f deploy/myapp      # 모든 파드 스트리밍
kubectl top pod                   # CPU/메모리 사용(메트릭 서버 필요)
```

### 13.2 애플리케이션 레벨
- /healthz, /livez, /metrics(Prometheus) 노출
- 로깅: JSON 구조화 → Loki/ELK
- 트레이싱: OpenTelemetry SDK → Collector → Jaeger/Tempo

---

## 14. Helm으로 배포(차트화)

```bash
helm create myapp
# values.yaml 수정(이미지/replica/probes/env/resources 등)
helm install myapp ./myapp -n prod
helm upgrade myapp ./myapp -f values-prod.yaml
helm rollback myapp 3
```

Helm 값만 바꿔 환경(dev/stage/prod) 별 커스터마이징.

---

## 15. Kustomize(오버레이로 환경 분리)

```
k8s/
├─ base/           # 공통 리소스
│  ├─ deployment.yaml
│  └─ kustomization.yaml
└─ overlays/
   ├─ dev/kustomization.yaml
   └─ prod/kustomization.yaml
```

```bash
kubectl apply -k k8s/overlays/prod
```

---

## 16. 실습: Flask 앱(간단) → K8s

### 16.1 Dockerfile
```dockerfile
FROM python:3.11-alpine
WORKDIR /app
COPY app.py .
RUN pip install --no-cache-dir flask
EXPOSE 80
CMD ["python","app.py"]
```

### 16.2 앱
```python
# app.py
from flask import Flask
app = Flask(__name__)
@app.get("/")
def root():
    return "Hello from Kubernetes!"
@app.get("/healthz")
def healthz():
    return "ok"
@app.get("/livez")
def livez():
    return "alive"
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

### 16.3 배포 리소스
```yaml
# flask.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: flask-app, labels: { app: flask-app } }
spec:
  replicas: 2
  selector: { matchLabels: { app: flask-app } }
  template:
    metadata: { labels: { app: flask-app } }
    spec:
      containers:
        - name: flask
          image: yourname/flask-app:1.0.0
          ports: [{containerPort: 80}]
          readinessProbe: { httpGet: { path: /healthz, port: 80 }, initialDelaySeconds: 5 }
          livenessProbe:  { httpGet: { path: /livez,   port: 80 }, initialDelaySeconds: 10 }
---
apiVersion: v1
kind: Service
metadata: { name: flask-svc }
spec:
  selector: { app: flask-app }
  ports: [{ port: 80, targetPort: 80 }]
  type: NodePort
```

```bash
kubectl apply -f flask.yaml
kubectl get svc flask-svc -o wide
# http://<노드IP>:<NodePort> 로 접속
```

---

## 17. 운영 팁/트러블슈팅

| 증상 | 원인/대응 |
|---|---|
| `ImagePullBackOff` | 레지스트리 인증 누락(imagePullSecrets), 이미지 경로/태그 오타 |
| `CrashLoopBackOff` | 프로세스 종료(환경변수/포트/DB연결), liveness probe가 너무 공격적 |
| `Readiness probe failed` | 앱 부팅 지연 → startupProbe/initialDelaySeconds 조정 |
| 외부 접속 불가 | Ingress 컨트롤러 미설치/잘못된 host/TLS, Service selector 오타 |
| HPA 작동 X | Metrics Server 미설치/권한 문제 |
| 롤링 무중단 실패 | readiness 설정 누락, maxUnavailable>0, DB migrations로 장기 lock |

---

## 18. 보안/거버넌스 요약

- **이미지**: 최소 베이스(슬림/알파인/디스트로리스), Trivy 스캔, Cosign 서명, 다이제스트 배포
- **Pod 보안**: 비루트, readOnlyRootFilesystem, drop ALL caps, AppArmor/SELinux, seccomp
- **네트워크**: NetworkPolicy로 제어면/데이터면 분리, egress 제한
- **비밀**: Secret은 KMS/SealedSecret/Vault로 암호화 관리
- **정책**: Gatekeeper/Kyverno로 강제(리소스 제한/프로브/SA/서명검증)
- **RBAC**: 최소 권한 원칙(서비스 계정 별도 발급, Role/Binding 세분화)

---

## 19. 간단 용량/비용 감 잡기
- 평균 요청당 CPU 시간 \(c\), 초당 요청 \(r\), 포드당 CPU 할당 \(C\)일 때 필요한 포드 수 \(k\)의 근사:
$$
k \approx \left\lceil \frac{c \cdot r}{C \cdot \beta} \right\rceil
$$
- \(\beta\): 안전률(예: 0.6–0.7). 메모리 제약도 별도로 체크해 큰 값을 채택.

---

## 20. 배포 자동화(선택) — GitOps/Helm/Kustomize

- **GitOps(Argo CD/Flux)**: 매니페스트/차트를 Git에 선언 → 클러스터가 Pull로 동기화  
- **Helm**: 값 파일 프로모션(dev→staging→prod), 릴리스 이력/롤백 쉬움  
- **Kustomize**: 오버레이로 환경 차이 최소 diff

---

## 21. 최종 체크리스트

- [ ] 이미지: 보안 스캔/서명, 다이제스트 고정  
- [ ] 리소스: requests/limits 정의, HPA/PDB 설정  
- [ ] 안정성: readiness/liveness/startup probe 설정  
- [ ] 보안: 비루트/RO FS/Capabilities drop/NetworkPolicy/RBAC  
- [ ] 노출: Ingress + TLS, 올바른 Service selector  
- [ ] 운영: 로그/지표/트레이싱, 경보, 롤아웃 이력/롤백  
- [ ] 문서화: 런북/장애 시나리오/백업·복구 절차

---

## 참고 자료
- Kubernetes Concepts & API Reference: https://kubernetes.io/docs/home/  
- kubectl 명령어: https://kubernetes.io/docs/reference/kubectl/  
- Ingress-NGINX: https://kubernetes.github.io/ingress-nginx/  
- Cert-Manager: https://cert-manager.io/  
- HPA: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/  
- NetworkPolicy: https://kubernetes.io/docs/concepts/services-networking/network-policies/  
- Helm: https://helm.sh/docs/  
- Kustomize: https://kubectl.docs.kubernetes.io/installation/kustomize/