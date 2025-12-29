---
layout: post
title: Docker - Kubernetes 클러스터에 Docker 이미지 배포
date: 2025-03-24 20:20:23 +0900
category: Docker
---
# Kubernetes 클러스터에 Docker 이미지 배포: 실전 가이드

Docker 이미지를 Kubernetes 클러스터에 배포하는 과정은 단순한 컨테이너 실행을 넘어서는 여러 고려사항을 포함합니다. 이 가이드는 이미지 버전 관리, 프라이빗 레지스트리 인증, 멀티 컨테이너 구성, 헬스 체크, 리소스 관리, 네트워킹, 보안, 배포 전략 등 실무에서 필요한 모든 요소를 다룹니다.

---

## 배포 흐름 개요

Kubernetes에 애플리케이션을 배포하는 일반적인 흐름은 다음과 같습니다:

1. **Docker 이미지 빌드**: 애플리케이션을 컨테이너 이미지로 패키징
2. **레지스트리에 푸시**: Docker Hub, Harbor, ECR, GHCR 등의 레지스트리에 이미지 업로드
3. **Kubernetes 리소스 정의**: Deployment, Service, ConfigMap, Secret 등의 매니페스트 작성
4. **클러스터에 적용**: kubectl 명령어나 Helm/Kustomize를 통해 배포
5. **운영 관리**: 노출, 스케일링, 모니터링, 롤백 등의 운영 작업 수행

---

## Docker 이미지 준비 및 레지스트리 업로드

### 기본 이미지 빌드 및 푸시 과정

```bash
# 이미지 빌드
docker build -t yourname/myapp:1.0.0 .

# Docker Hub 로그인
docker login

# 추가 태그 생성
docker tag yourname/myapp:1.0.0 yourname/myapp:latest

# 레지스트리에 푸시
docker push yourname/myapp:1.0.0
docker push yourname/myapp:latest
```

### 재현성 보장을 위한 다이제스트 사용

운영 환경에서는 태그 대신 다이제스트를 사용하여 특정 이미지 버전을 고정하는 것이 좋습니다. 이렇게 하면 동일한 코드가 항상 동일한 이미지를 참조하도록 보장할 수 있습니다.

{% raw %}
```bash
# 이미지 다이제스트 확인
docker pull yourname/myapp:1.0.0
docker inspect --format='{{index .RepoDigests 0}}' yourname/myapp:1.0.0
# 출력 예: yourname/myapp@sha256:abcdef...
```
{% endraw %}

Kubernetes 매니페스트에서 다이제스트 사용:
```yaml
image: yourname/myapp@sha256:abcdef...   # 태그 대신 다이제스트 사용
```

---

## Kubernetes 환경 설정

### 네임스페이스 생성 및 컨텍스트 설정

```bash
# 프로덕션 네임스페이스 생성
kubectl create namespace prod

# 현재 컨텍스트의 네임스페이스 설정
kubectl config set-context --current --namespace=prod
```

### 권장 애드온 설치

효율적인 클러스터 운영을 위해 다음 애드온을 고려하세요:
- **Admission 컨트롤러**: OPA/Gatekeeper/Kyverno를 사용하여 이미지 서명, 다이제스트 필수 사용, 리소스 제한 등의 정책 강제
- **Metrics Server**: Horizontal Pod Autoscaler(HPA) 동작을 위한 리소스 메트릭 수집
- **Ingress Controller**: NGINX, Contour, Traefik 등을 통한 L7 로드 밸런싱
- **Cert-Manager**: TLS 인증서 자동 발급 및 갱신 관리

---

## Deployment 매니페스트 작성

다음은 프로덕션 환경을 고려한 기본 Deployment 매니페스트 예시입니다:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels: 
    app: myapp
spec:
  replicas: 2
  revisionHistoryLimit: 10  # 롤백을 위해 이전 리비전 유지
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # 업데이트 중 추가로 생성할 수 있는 파드 수
      maxUnavailable: 0     # 업데이트 중 사용 불가능한 파드 수 (무중단 배포 보장)
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
          image: yourname/myapp:1.0.0   # 다이제스트 사용 권장
          imagePullPolicy: IfNotPresent  # 또는 Always (최신 태그 사용 시)
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
            httpGet: 
              path: /healthz
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:
            httpGet: 
              path: /livez
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
          startupProbe:
            httpGet: 
              path: /startupz
              port: 80
            failureThreshold: 30
            periodSeconds: 3
      terminationGracePeriodSeconds: 30  # 정상 종료 대기 시간
```

**헬스 체크(Probes) 설명**:
- **Readiness Probe**: 애플리케이션이 트래픽을 처리할 준비가 되었는지 확인
- **Liveness Probe**: 애플리케이션 프로세스가 정상적으로 실행 중인지 확인
- **Startup Probe**: 애플리케이션 초기 시작 시 장시간 부팅이 필요한 경우 사용

---

## 서비스 노출 전략

### 기본 ClusterIP 서비스

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector: 
    app: myapp
  ports:
    - name: http
      port: 80         # 클러스터 내부에서 접근하는 포트
      targetPort: 80   # 컨테이너에서 노출하는 포트
  type: ClusterIP      # 기본값, 클러스터 내부 통신용
```

### 외부 노출 옵션

- **NodePort**: 디버깅이나 온프레미스 환경에서 간단한 외부 노출에 사용
- **LoadBalancer**: 클라우드 환경에서 외부 로드 밸런서 자동 생성
- **Ingress**: L7 라우팅, 도메인 기반 라우팅, TLS 종료 등 고급 기능 제공

NodePort 서비스 예시:
```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080   # 30000-32767 범위 내 지정
```

---

## Ingress 및 TLS 구성

Ingress Controller 설치 후 다음과 같이 Ingress 리소스를 정의할 수 있습니다:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
    - hosts: 
        - myapp.example.com
      secretName: myapp-tls  # TLS 인증서가 저장된 Secret 이름
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port: 
                  number: 80
```

**Cert-Manager를 통한 자동 TLS 관리**:
Cert-Manager를 사용하면 Let's Encrypt와 같은 ACME 인증서 발급자를 통해 TLS 인증서를 자동으로 발급하고 갱신할 수 있습니다. Ingress에 `cert-manager.io/cluster-issuer: letsencrypt` 어노테이션을 추가하면 됩니다.

---

## 프라이빗 레지스트리 인증 설정

### 이미지 풀 시크릿 생성

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=DOCKER_USER \
  --docker-password=DOCKER_PASS \
  --namespace=prod
```

### Deployment에서 시크릿 사용

```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

### ServiceAccount에 시크릿 연결 (권장)

여러 파드에서 동일한 시크릿을 사용할 경우 ServiceAccount에 연결하는 것이 효율적입니다:

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
spec:
  template:
    spec:
      serviceAccountName: app-sa  # ServiceAccount 지정
```

---

## 구성 관리: ConfigMap과 Secret

### ConfigMap 정의

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  APP_MODE: production
  APP_REGION: ap-northeast-2
```

### Secret 정의

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
type: Opaque
data:
  DB_PASSWORD: c2VjdXJlX3Bhc3M=   # 'secure_pass'를 base64 인코딩
```

**주의**: Kubernetes Secret은 기본적으로 base64로 인코딩되지만 평문에 가깝습니다. 프로덕션 환경에서는 Vault, SealedSecret, 또는 클라우드 제공자의 KMS를 통한 추가 암호화를 고려하세요.

### 컨테이너에 구성 주입

```yaml
# 환경 변수로 주입
envFrom:
  - configMapRef: 
      name: myapp-config
  - secretRef: 
      name: myapp-secret

# 파일로 마운트
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

## 스케일링 및 가용성 관리

### Horizontal Pod Autoscaler (HPA)

Metrics Server가 설치되어 있어야 HPA를 사용할 수 있습니다.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
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

### Pod Disruption Budget (PDB)

노드 업그레이드나 유지보수 시 최소한의 가용성을 보장합니다.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 1  # 최소 1개의 파드가 항상 실행 중이어야 함
  selector:
    matchLabels: 
      app: myapp
```

### 용량 계획 수립

클러스터 용량을 계획할 때는 다음과 같은 공식을 참고할 수 있습니다:

요청 기준으로 필요한 노드 수 계산:
$$
\text{노드 수} \approx \left\lceil \frac{\sum_{i=1}^{N} \text{pod}_i\_\text{cpu\_request}}{\text{노드당 vCPU}} \cdot \frac{1}{\alpha} \right\rceil
$$

여기서 \(\alpha\)는 여유율(일반적으로 0.7)을 나타냅니다. 메모리 요구사항도 별도로 계산하여 더 큰 값을 채택합니다.

---

## 보안 구성

### RBAC 및 ServiceAccount

최소 권한 원칙에 따라 필요한 권한만 부여합니다.

```yaml
# Role 정의
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]

# RoleBinding 정의
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: config-reader-bind
subjects:
  - kind: ServiceAccount
    name: app-sa
roleRef:
  kind: Role
  name: config-reader
  apiGroup: rbac.authorization.k8s.io
```

### NetworkPolicy

네트워크 트래픽을 세밀하게 제어합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp-allow-frontend
spec:
  podSelector:
    matchLabels: 
      app: myapp
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector:
            matchLabels: 
              role: frontend
      ports:
        - protocol: TCP
          port: 80
```

### SecurityContext

컨테이너 보안 설정을 강화합니다.

```yaml
securityContext:
  runAsNonRoot: true        # root 사용자로 실행 금지
  runAsUser: 1000           # 특정 사용자 ID로 실행
  readOnlyRootFilesystem: true  # 루트 파일 시스템 읽기 전용
  allowPrivilegeEscalation: false  # 권한 상승 방지
  capabilities:
    drop: ["ALL"]           # 모든 Linux capabilities 제거
```

---

## 멀티 컨테이너 패턴

사이드카 패턴을 사용하여 로깅, 프록시, 모니터링 등의 기능을 추가할 수 있습니다.

```yaml
spec:
  containers:
    - name: app
      image: yourname/myapp:1.0.0
      ports: 
        - containerPort: 8080
    - name: envoy
      image: envoyproxy/envoy:v1.30-latest
      ports: 
        - containerPort: 80
      args: ["-c", "/etc/envoy/envoy.yaml"]
      volumeMounts:
        - name: envoy-config
          mountPath: /etc/envoy
  volumes:
    - name: envoy-config
      configMap:
        name: envoy-config
```

---

## 배포 전략

### Blue-Green 배포

두 개의 독립된 환경(Blue와 Green)을 유지하며 트래픽을 전환하는 방식입니다.

```yaml
# Service의 selector만 변경하여 트래픽 전환
spec:
  selector: 
    app: myapp
    color: blue  # green으로 변경 시 신규 버전으로 트래픽 전환
```

### Canary 배포

NGINX Ingress Controller의 Canary 기능을 활용하여 점진적으로 신규 버전에 트래픽을 전환합니다.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% 트래픽만 신규 버전으로
```

---

## 배포 관리 및 롤백

### 배포 상태 모니터링

```bash
# 배포 진행 상태 확인
kubectl rollout status deploy/myapp

# 배포 히스토리 확인
kubectl rollout history deploy/myapp

# 특정 리비전 상세 정보 확인
kubectl rollout history deploy/myapp --revision=3

# 이전 버전으로 롤백
kubectl rollout undo deploy/myapp --to-revision=3
```

문제 발생 시 즉시 이전 안정적인 버전으로 롤백할 수 있어야 합니다.

---

## 모니터링 및 관측성

### 기본 명령어

```bash
# 파드 상태 확인
kubectl get pods -o wide

# 파드 상세 정보
kubectl describe pod <pod-name>

# 로그 확인
kubectl logs -f deploy/myapp

# 리소스 사용량 확인 (Metrics Server 필요)
kubectl top pod
```

### 애플리케이션 레벨 관측성

  - **헬스 엔드포인트**: `/healthz`, `/livez`, `/readyz` 제공
  - **메트릭스 엔드포인트**: Prometheus 형식의 `/metrics` 노출
  - **구조화된 로깅**: JSON 형식 로그 출력 (Loki/ELK와 통합)
  - **분산 추적**: OpenTelemetry SDK를 통한 트레이스 생성 (Jaeger/Tempo와 통합)

---

## 패키지 관리 도구 활용

### Helm을 통한 배포

Helm은 Kubernetes 애플리케이션을 패키징하고 배포하기 위한 차트 관리 도구입니다.

```bash
# Helm 차트 생성
helm create myapp

# values.yaml 파일 수정 (이미지, 레플리카, 환경변수 등)
# 차트 설치
helm install myapp ./myapp -n prod

# 차트 업그레이드
helm upgrade myapp ./myapp -f values-prod.yaml

# 롤백
helm rollback myapp 3
```

### Kustomize를 통한 환경별 구성 관리

Kustomize는 오버레이를 통해 환경별 구성을 관리하는 도구입니다.

디렉토리 구조 예시:
```
k8s/
├─ base/           # 공통 리소스
│  ├─ deployment.yaml
│  └─ kustomization.yaml
└─ overlays/
   ├─ dev/kustomization.yaml  # 개발 환경 구성
   └─ prod/kustomization.yaml # 프로덕션 환경 구성
```

적용 방법:
```bash
kubectl apply -k k8s/overlays/prod
```

---

## 실전 예제: Flask 애플리케이션 배포

### Dockerfile

```dockerfile
FROM python:3.11-alpine
WORKDIR /app
COPY app.py .
RUN pip install --no-cache-dir flask
EXPOSE 80
CMD ["python", "app.py"]
```

### 애플리케이션 코드 (app.py)

```python
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

### Kubernetes 배포 매니페스트

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  labels:
    app: flask-app
spec:
  replicas: 2
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
          image: yourname/flask-app:1.0.0
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 5
          livenessProbe:
            httpGet:
              path: /livez
              port: 80
            initialDelaySeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: flask-svc
spec:
  selector:
    app: flask-app
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
```

### 배포 및 접속 테스트

```bash
# 배포 적용
kubectl apply -f flask.yaml

# 서비스 정보 확인
kubectl get svc flask-svc -o wide

# http://<노드IP>:<NodePort> 로 접속 테스트
```

---

## 운영 트러블슈팅 가이드

| 증상 | 가능한 원인 및 해결 방안 |
|---|---|
| `ImagePullBackOff` | 레지스트리 인증 누락(imagePullSecrets 확인), 이미지 경로 또는 태그 오타 |
| `CrashLoopBackOff` | 애플리케이션 프로세스 즉시 종료 (환경변수, 포트 충돌, DB 연결 문제 점검) |
| `Readiness probe failed` | 애플리케이션 시작 지연 (startupProbe 추가 또는 initialDelaySeconds 조정) |
| 외부 접속 불가 | Ingress 컨트롤러 미설치, Ingress host 설정 오류, TLS 인증서 문제 |
| HPA 작동 안 함 | Metrics Server 미설치, 메트릭 수집 권한 문제 |
| 롤링 업데이트 실패 | readinessProbe 미설정, maxUnavailable 값 과도, DB 마이그레이션 록 |

---

## 보안 모범 사례 요약

1. **이미지 보안**
   - 최소한의 베이스 이미지 사용 (slim, alpine, distroless)
   - 정기적 취약점 스캔 (Trivy 등)
   - 이미지 서명 및 검증 (Cosign)
   - 다이제스트를 통한 버전 고정

2. **파드 보안**
   - 비루트 사용자로 실행
   - 읽기 전용 루트 파일 시스템
   - 불필요한 Linux capabilities 제거
   - AppArmor/SELinux 프로필 적용

3. **네트워크 보안**
   - NetworkPolicy를 통한 트래픽 제어
   - 네트워크 영역 분리 (제어면/데이터면)
   - 아웃바운드 트래픽 제한

4. **비밀 정보 관리**
   - Secret의 기본 암호화 한계 인지
   - Vault, SealedSecret, KMS 등을 통한 추가 암호화

5. **정책 관리**
   - Gatekeeper/Kyverno를 통한 정책 강제
   - 리소스 제한, 프로브 필수, 서비스 계정 제한 등

6. **권한 관리**
   - 최소 권한 원칙 적용
   - 서비스 계정 별도 생성 및 권한 부여
   - Role과 RoleBinding 세분화

---

## 용량 계획 및 비용 관리

애플리케이션 용량을 계획할 때는 다음 공식을 참고할 수 있습니다:

평균 요청당 CPU 시간 \(c\), 초당 요청 수 \(r\), 파드당 CPU 할당량 \(C\)일 때 필요한 파드 수 \(k\)의 근사값:
$$
k \approx \left\lceil \frac{c \cdot r}{C \cdot \beta} \right\rceil
$$

여기서 \(\beta\)는 안전률(일반적으로 0.6-0.7)입니다. 메모리 제약 조건도 별도로 평가하여 더 큰 값을 채택합니다.

---

## GitOps 및 배포 자동화

- **GitOps (Argo CD/Flux)**: 매니페스트와 차트를 Git 저장소에 선언적으로 관리하고, 클러스터가 이를 주기적으로 동기화
- **Helm**: 값 파일을 통해 환경별 구성 관리, 릴리스 이력 및 롤백 기능 제공
- **Kustomize**: 오버레이를 통한 환경별 차이점 최소화, GitOps와의 자연스러운 통합

---

## 결론: 성공적인 Kubernetes 배포를 위한 핵심 원칙

Kubernetes에 Docker 이미지를 배포하는 과정은 기술적 세부사항을 넘어서 체계적인 접근 방식이 필요합니다. 다음 원칙들을 준수하면 더 안정적이고 안전하며 효율적인 배포 환경을 구축할 수 있습니다:

1. **재현성 보장**: 태그 대신 다이제스트를 사용하여 동일한 코드가 항상 동일한 이미지를 참조하도록 합니다. 이는 운영 환경의 예측 가능성을 높이고 장애 조치를 단순화합니다.

2. **보안 우선**: 컨테이너 수준에서 네트워크 수준까지 다층적 보안 방어 체계를 구축합니다. 최소 권한 원칙을 적용하고, 정기적인 취약점 스캔과 이미지 서명을 통해 공급망 보안을 강화합니다.

3. **탄력성 설계**: 헬스 체크, 리소스 제한, 자동 스케일링, 파드 중단 예산 등을 활용하여 시스템이 장애에 견고하게 대응할 수 있도록 합니다.

4. **관측성 내재화**: 로깅, 메트릭스, 추적을 애플리케이션 설계 단계부터 고려합니다. 문제 발생 시 빠른 진단과 해결을 가능하게 하는 모니터링 체계를 구축합니다.

5. **자동화 철학**: 수동 작업을 최소화하고 CI/CD 파이프라인, GitOps, 정책 자동화를 통해 일관되고 감사 가능한 배포 프로세스를 확립합니다.

6. **점진적 개선**: 모든 것을 완벽하게 구현하려는 압박보다는, 핵심 기능부터 시작하여 지속적으로 개선해 나가는 접근 방식을 채택합니다. 각 배포에서 하나의 보안이나 안정성 측면을 개선하는 것부터 시작하세요.

7. **문서화 및 지식 공유**: 배포 절차, 트러블슈팅 가이드, 장애 대응 절차를 문서화하고 팀 내에서 지속적으로 공유합니다. 이는 팀의 운영 역량을 강화하고 새로운 멤버의 온보딩을 용이하게 합니다.

Kubernetes 생태계는 빠르게 발전하고 있지만, 이러한 기본 원칙들은 변하지 않습니다. 도구와 기술이 변화하더라도 재현성, 보안, 탄력성, 관측성, 자동화에 대한 집중은 지속적인 성공을 위한 토대가 될 것입니다.