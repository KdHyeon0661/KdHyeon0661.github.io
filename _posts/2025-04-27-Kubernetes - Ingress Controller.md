---
layout: post
title: Kubernetes - Ingress Controller
date: 2025-04-27 20:20:23 +0900
category: Kubernetes
---
# Kubernetes Ingress Controller로 외부 접근 설정하기

## Ingress 개요와 구성 요소

- **Ingress**: 외부 HTTP(S) 요청을 내부 Service로 라우팅하는 **규칙 집합** (API 오브젝트)
- **Ingress Controller**: Ingress 규칙을 실제로 반영하여 트래픽을 처리하는 **리버스 프록시/로드 밸런서**(예: NGINX, HAProxy, Traefik, Istio IngressGateway 등)
- **Service**: Pod 앞의 안정된 엔드포인트 (ClusterIP/NodePort/LoadBalancer 등). Ingress는 **보통 ClusterIP** Service를 백엔드로 둔다.

구성 관계:

```
[ Client ] → [ LB / Node IP ] → [ Ingress Controller Service ] → [ Ingress 규칙 ]
                                                       └→ [ Service A ] → [ Pod A들 ]
                                                       └→ [ Service B ] → [ Pod B들 ]
```

---

## Ingress Controller 설치

### Minikube(가장 간단)
```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
kubectl get svc  -n ingress-nginx
```

### 일반 클러스터(Helm 권장)
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# 기본 설치
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --create-namespace -n ingress-nginx
```

설치 확인:
```bash
kubectl get pods -n ingress-nginx
kubectl get svc  -n ingress-nginx
```

> 클라우드 환경에서 `ingress-nginx-controller` Service가 **LoadBalancer** 타입으로 외부 IP를 받을 수 있다. 온프렘/가상환경이면 NodePort 또는 MetalLB 등 별도 L2/L3 LB가 필요.

---

## 예제 애플리케이션 두 개 배포

### app1
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app1-svc
spec:
  selector:
    app: app1
  ports:
    - port: 80
      targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 1
  selector:
    matchLabels: { app: app1 }
  template:
    metadata:
      labels: { app: app1 }
    spec:
      containers:
      - name: app1
        image: hashicorp/http-echo
        args: ["-text=Hello from App1", "-listen=:5678"]
        ports: [{ containerPort: 5678 }]
```

### app2
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app2-svc
spec:
  selector:
    app: app2
  ports:
    - port: 80
      targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 1
  selector:
    matchLabels: { app: app2 }
  template:
    metadata:
      labels: { app: app2 }
    spec:
      containers:
      - name: app2
        image: hashicorp/http-echo
        args: ["-text=Hello from App2", "-listen=:5678"]
        ports: [{ containerPort: 5678 }]
```

적용:
```bash
kubectl apply -f app1.yaml
kubectl apply -f app2.yaml
```

---

## 기본 Ingress: 경로 기반 라우팅(+Rewrite)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-svc
            port: { number: 80 }
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-svc
            port: { number: 80 }
```

적용/확인:
```bash
kubectl apply -f ingress.yaml
kubectl get ingress
```

### 테스트(Minikube)
```bash
minikube ip
# 예: 192.168.49.2
curl http://192.168.49.2/app1
curl http://192.168.49.2/app2
```

> NodePort만 노출된 경우:
```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
curl http://localhost:8080/app1
```

---

## 호스트 기반 라우팅(도메인별 분기)

로컬 /etc/hosts에 Ingress LB IP를 매핑(테스트용):
```
<LB_IP> app.local api.local
```

Ingress(호스트 규칙):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-routing
spec:
  ingressClassName: nginx
  rules:
  - host: app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: { name: app1-svc, port: { number: 80 } }
  - host: api.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: { name: app2-svc, port: { number: 80 } }
```

---

## 고급 라우팅 기능(주요 애노테이션)

> 컨트롤러별로 키가 다를 수 있다. 아래는 **NGINX Ingress Controller** 기준.

### 1) 정규식/캡처 기반 리라이트
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - http:
      paths:
      - path: /shop/?(.*)
        pathType: Prefix
        backend:
          service: { name: app1-svc, port: { number: 80 } }
```

### 2) 요청/응답 헤더 조작
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-set-headers: "configmap-req-headers"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-req-headers
  namespace: default
data:
  X-Request-From: "ingress"
```

### 3) 타임아웃/바디 사이즈
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
```

### 4) 기본 인증(베이직)
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-type: "basic"
    nginx.ingress.kubernetes.io/auth-secret: "basic-auth"
    nginx.ingress.kubernetes.io/auth-realm: "Restricted"
---
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
type: Opaque
data:
  auth: |-
    # htpasswd -nb user pass | base64
    dXNlcjokYXByMSR6bkQxcy9aJGFqWkdScXBiT3lCUzZ0Q2ZxLw==
```

### 5) CORS
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET,PUT,POST,DELETE,PATCH,OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization,Content-Type"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
```

### 6) Sticky 세션(쿠키 기반)
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "86400"
```

### 7) Rate Limit(초당/분당)
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "20"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "2"
```

### 8) gRPC / WebSocket
- gRPC: Service 포트 명을 `grpc`로 지정하거나 `nginx.ingress.kubernetes.io/backend-protocol: "GRPC"`
- WebSocket: NGINX는 기본 업그레이드를 지원(대개 추가 설정 불필요)

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
```

---

## TLS(HTTPS) 설정

### 1) 수동 Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-tls
type: kubernetes.io/tls
data:
  tls.crt: <base64>
  tls.key: <base64>
```

Ingress에 연결:
```yaml
spec:
  tls:
  - hosts: ["app.local"]
    secretName: example-tls
  rules:
  - host: app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: app1-svc, port: { number: 80 } } }
```

### 2) cert-manager(권장: 자동 발급/갱신)
- ClusterIssuer/Issuer 생성(Let’s Encrypt HTTP-01/ DNS-01)
- Ingress에 `cert-manager.io/cluster-issuer: "letsencrypt-prod"` 애노테이션 추가
- cert-manager가 적절한 TLS Secret을 생성/갱신

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts: ["api.example.com"]
    secretName: api-tls
```

---

## Canary / Blue-Green 라우팅(간단 패턴)

### 1) Canary by Header
두 개의 Ingress를 같은 호스트/경로로 정의하고, 하나에 Canary 플래그 부여.

```yaml
# 기본 트래픽
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations: { }
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: api-v1-svc, port: { number: 80 } } }
---
# Canary
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "1"
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: api-v2-svc, port: { number: 80 } } }
```

### 2) Canary by Weight
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10%
```

> Blue-Green은 DNS 스위치 또는 Ingress 백엔드 전환으로 구현. 롤백이 단순하다.

---

## 관측/모니터링

- **로그**: Ingress Controller Pod 로그
- **메트릭**: NGINX Ingress의 Prometheus Exporter(요청 수/지연/5xx 등)
- **대시보드**: Grafana + 커뮤니티 대시보드
- **Access Log 필드 확장**: 사용자/세션/헤더/업스트림 응답시간 등을 기록하여 SLO/에러 버짓 분석

---

## 한계/안티패턴/설계 기준

- Ingress는 **L7 HTTP/HTTPS 중심**(TCP/UDP는 별도 설정 필요 또는 다른 컨트롤러 사용)
- 과도한 **애노테이션 집약**은 관리 복잡도 증가 → 공통 정책은 **ConfigMap/Policy 객체**나 **Gateway API** 검토
- 대규모 환경: 수천 규칙 이상이면 **Gateway API**, **Service Mesh**(Istio, Linkerd) 고려
- WebSocket/gRPC 장수연결: **Idle/Read 타임아웃** 조정 필요
- 파일 업로드 대용량: `proxy-body-size` 상향 및 백엔드/스토리지 업로드 패턴 재검토(직접 S3 업로드 등)

---

## 트러블슈팅 체크리스트

1. **IngressClass** 불일치: `ingressClassName: nginx` vs 컨트롤러 실제 클래스
2. **Service 타입/포트/타겟포트** 불일치
3. **DNS/호스트 헤더**: 로컬 테스트 시 `/etc/hosts` 또는 `Host:` 헤더 지정
4. **Rewrite/Regex 충돌**: `use-regex` + `rewrite-target` 조합 확인
5. **TLS**: Secret 타입 `kubernetes.io/tls`, 키 이름 `tls.crt/tls.key` 확인
6. **헬스체크/백엔드 5xx**: 백엔드 Readiness/Liveness와 리소스 제한(CPU/메모리) 점검
7. **CORS**: 사전 요청(OPTIONS) 허용/헤더 목록/크리덴셜 일치
8. **Rate Limit/Timeout**: 의도치 않은 제한으로 429/504 발생 여부
9. **컨트롤러 로그**: 라우팅 미적용/파싱 에러/권한 이슈 파악

진단 명령:
```bash
kubectl get ingress -A
kubectl describe ingress sample-ingress
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller
kubectl get svc,pods -o wide
```

---

## 운영 팁 요약

- **기본 라우팅**: Prefix/Host 규칙 + 명시적 `ingressClassName`
- **보안**: TLS 필수(certificate 자동화: cert-manager), 보안 헤더 추가
- **성능/과금**: 클라우드 LB 수를 최소화하고 Ingress 뒤에 **여러 서비스**를 모은다
- **가용성**: HPA + 백엔드 Readiness, Ingress NGINX 복수 레플리카, PodDisruptionBudget
- **릴리즈 전략**: Canary/Blue-Green, 해더/가중치 기반 점진 이행
- **관측**: 로그/메트릭/트레이싱으로 SLO 관리

---

## 전체 예시(경로/호스트/리라이트/TLS/기본 보안 헤더)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prod-edge
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-body-size: "20m"
    nginx.ingress.kubernetes.io/server-snippet: |
      add_header X-Content-Type-Options nosniff;
      add_header X-XSS-Protection "1; mode=block";
      add_header Referrer-Policy strict-origin-when-cross-origin;
spec:
  ingressClassName: nginx
  tls:
  - hosts: ["app.example.com","api.example.com"]
    secretName: wildcard-example-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /shop
        pathType: Prefix
        backend: { service: { name: app1-svc, port: { number: 80 } } }
      - path: /help
        pathType: Prefix
        backend: { service: { name: app2-svc, port: { number: 80 } } }
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: app2-svc, port: { number: 80 } } }
```

---

## 결론

- **Ingress = 라우팅 규칙**, **Ingress Controller = 실제 트래픽 처리기**  
- NodePort/LoadBalancer 대비 **도메인·경로 단일 진입점**으로 다수 서비스를 **일관된 보안/정책** 아래 운영 가능  
- 실무 핵심: **올바른 클래스 지정, TLS 자동화(cert-manager), 적절한 애노테이션(리라이트/헤더/타임아웃/CORS/Sticky/RateLimit), Canary/Blue-Green, 관측**  
- 대규모·고난도 요구가 늘면 **Gateway API/Service Mesh**로 확장하는 것이 자연스러운 다음 단계다.