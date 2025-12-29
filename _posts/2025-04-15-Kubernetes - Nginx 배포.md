---
layout: post
title: Kubernetes - Nginx 배포
date: 2025-04-15 20:20:23 +0900
category: Kubernetes
---
# Kubernetes 예제: 간단한 Nginx 배포부터 운영까지

## 목표

이 실습 가이드는 단순한 Nginx 배포에서 시작하여 점진적으로 운영 환경에 필요한 기능들을 추가해 나가는 과정을 제공합니다. 초보자도 따라할 수 있는 기본적인 배포부터 시작하여, 점차 프로덕션 수준의 구성으로 발전시켜 나갑니다.

---

## 사전 준비

이 실습을 시작하기 전에 다음 요구사항을 확인하세요:

### 필수 도구
- **Kubernetes 클러스터**: Minikube, Kind, 또는 클라우드 Kubernetes 서비스(EKS, GKE, AKS)
- **kubectl**: Kubernetes CLI 도구

### 환경 확인
```bash
# kubectl 클라이언트 버전 확인
kubectl version --client

# 클러스터 연결 확인
kubectl cluster-info

# 노드 상태 확인
kubectl get nodes -o wide
```

**권장 환경**: 처음 실습하는 경우 가벼운 로컬 클러스터인 Minikube나 Kind를 사용하는 것을 권장합니다.

---

## 1단계: 기본 Nginx Deployment 구성

### 간단한 Nginx Deployment 정의

첫 번째 단계로 가장 기본적인 Nginx Deployment를 구성해 보겠습니다.

```yaml
# nginx-deployment-basic.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  # 복제본 수: 2개의 Pod를 실행합니다
  replicas: 2
  
  # Pod 선택기: 어떤 Pod를 관리할지 결정
  selector:
    matchLabels:
      app: nginx
  
  # Pod 템플릿: 새로운 Pod가 생성될 때 사용할 스펙
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        # 이미지 태그: 실무에서는 'latest' 대신 명시적 버전 사용 권장
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 주요 구성 요소 설명

| 구성 요소 | 설명 | 실무 권장사항 |
|-----------|------|---------------|
| `replicas: 2` | 실행할 Pod의 개수 | 가용성을 위해 최소 2개 이상 권장 |
| `selector.matchLabels` | 관리할 Pod를 선택하는 라벨 | 템플릿의 라벨과 정확히 일치해야 함 |
| `image: nginx:latest` | 사용할 컨테이너 이미지 | 재현성과 보안을 위해 `nginx:1.27-alpine`처럼 명시적 버전 태그 사용 권장 |
| `containerPort: 80` | 컨테이너 내부에서 노출할 포트 | 애플리케이션이 실제로 사용하는 포트 지정 |

### 주의사항
실무 환경에서는 `latest` 태그를 사용하지 않는 것이 좋습니다. 명시적인 버전 태그를 사용하면:
- 재현 가능한 배포 보장
- 보안 취약점 스캔 정확도 향상
- 의도치 않은 변경 방지

---

## 2단계: Service를 통한 외부 노출

### NodePort Service 정의

Deployment만으로는 외부에서 Nginx에 접근할 수 없습니다. Service를 사용하여 Pod를 외부에 노출합니다.

```yaml
# nginx-service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  # Service 유형: NodePort는 각 노드의 특정 포트를 통해 서비스 노출
  type: NodePort
  
  # Pod 선택기: 트래픽을 전달할 Pod 선택
  selector:
    app: nginx
  
  # 포트 구성
  ports:
  - port: 80        # Service의 포트
    targetPort: 80  # Pod 내 컨테이너의 포트
    nodePort: 30080 # 노드에서 노출할 포트 (30000-32767 범위)
```

### Service 구성 요소 설명

| 구성 요소 | 설명 | 참고사항 |
|-----------|------|----------|
| `type: NodePort` | Service 유형 | 각 노드의 지정된 포트로 서비스 노출 |
| `selector.app: nginx` | 트래픽 대상 Pod 선택 | Deployment의 Pod 라벨과 일치해야 함 |
| `port: 80` | Service의 가상 포트 | 클러스터 내부에서 접근할 때 사용 |
| `targetPort: 80` | 컨테이너의 실제 포트 | Pod 내 Nginx 컨테이너가 수신하는 포트 |
| `nodePort: 30080` | 외부 접근 포트 | 30000-32767 범위에서 선택 |

### 리소스 배포 및 확인

```bash
# Deployment와 Service 배포
kubectl apply -f nginx-deployment-basic.yaml
kubectl apply -f nginx-service-nodeport.yaml

# 배포 상태 확인
kubectl get deployments
kubectl get pods -o wide
kubectl get services
kubectl describe deployment nginx-deployment

# Service와 연결된 엔드포인트 확인
kubectl get endpoints nginx-service -o wide
```

예상 출력 예시:
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           1m

NAME             TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-service    NodePort   10.96.22.121   <none>        80:30080/TCP   1m
```

---

## 3단계: 애플리케이션 접근 테스트

### 환경별 접근 방법

**Minikube 환경**:
```bash
# Minikube에서 자동으로 브라우저 열기
minikube service nginx-service

# 또는 서비스 URL 확인
minikube service nginx-service --url
```

**일반 Kubernetes 환경**:
```bash
# 노드 IP 확인
kubectl get nodes -o wide

# curl로 접근 테스트
curl -I http://<노드-IP>:30080
```

**포트 포워딩을 통한 로컬 테스트**:
```bash
# 로컬 포트 8080을 서비스 포트 80으로 포워딩
kubectl port-forward service/nginx-service 8080:80

# 다른 터미널에서 테스트
curl -I http://localhost:8080
```

### 성공적인 응답 확인
Nginx가 정상적으로 실행되면 다음과 유사한 응답을 받을 수 있습니다:
```
HTTP/1.1 200 OK
Server: nginx/1.27.0
Date: [날짜]
Content-Type: text/html
Content-Length: 615
Last-Modified: [날짜]
Connection: keep-alive
ETag: [ETag 값]
Accept-Ranges: bytes
```

---

## 4단계: 운영 환경을 위한 고급 Deployment 구성

기본적인 배포가 작동하는 것을 확인했다면, 이제 프로덕션 환경에 적합한 고급 구성을 추가해 보겠습니다.

### 운영용 Nginx Deployment

```yaml
# nginx-deployment-production.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app.kubernetes.io/name: nginx
    app.kubernetes.io/version: "1.27"
    environment: production
spec:
  # 복제본 관리
  replicas: 3
  revisionHistoryLimit: 10  # 롤백을 위한 이력 보존 개수
  
  # 업데이트 전략
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 업데이트 중 추가로 생성할 수 있는 Pod 수
      maxUnavailable: 0  # 업데이트 중 사용 불가능할 수 있는 Pod 수
  
  # Pod 선택기
  selector:
    matchLabels:
      app: nginx
      environment: production
  
  # Pod 템플릿
  template:
    metadata:
      labels:
        app: nginx
        environment: production
    spec:
      # 보안 컨텍스트
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      
      containers:
      - name: nginx
        # 명시적 버전 태그 사용
        image: nginx:1.27-alpine
        imagePullPolicy: IfNotPresent
        
        # 포트 설정
        ports:
        - containerPort: 80
          name: http
        
        # 리소스 제한
        resources:
          requests:
            cpu: "100m"      # 0.1 CPU 코어
            memory: "128Mi"  # 128 메가바이트
          limits:
            cpu: "500m"      # 0.5 CPU 코어
            memory: "256Mi"  # 256 메가바이트
        
        # 헬스 체크
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3   # 컨테이너 시작 후 첫 검사까지 대기 시간
          periodSeconds: 5         # 검사 간격
          timeoutSeconds: 2        # 검사 타임아웃
          failureThreshold: 3      # 실패 횟수 제한
        
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15  # liveness는 더 긴 초기 지연
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 3
        
        # 환경 변수
        env:
        - name: NGINX_ENV
          value: "production"
        - name: TZ
          value: "Asia/Seoul"
```

### 운영용 구성 적용 및 관리

```bash
# 운영용 Deployment 적용
kubectl apply -f nginx-deployment-production.yaml

# 롤아웃 상태 모니터링
kubectl rollout status deployment/nginx-deployment

# 업데이트 이력 확인
kubectl rollout history deployment/nginx-deployment

# 이미지 업데이트
kubectl set image deployment/nginx-deployment nginx=nginx:1.27.1-alpine

# 업데이트 상태 확인
kubectl rollout status deployment/nginx-deployment

# 문제 발생 시 롤백
kubectl rollout undo deployment/nginx-deployment

# 특정 리비전으로 롤백
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

---

## 5단계: ConfigMap을 통한 동적 설정 관리

실제 운영 환경에서는 정적 파일이나 설정을 ConfigMap을 통해 관리하는 것이 일반적입니다.

### ConfigMap 생성

```yaml
# nginx-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  # 커스텀 HTML 페이지
  index.html: |
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>Welcome to Kubernetes Nginx</title>
      <style>
        body {
          font-family: Arial, sans-serif;
          margin: 40px;
          background-color: #f5f5f5;
        }
        .container {
          max-width: 800px;
          margin: 0 auto;
          background: white;
          padding: 30px;
          border-radius: 8px;
          box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        h1 { color: #333; }
        .status { color: green; font-weight: bold; }
      </style>
    </head>
    <body>
      <div class="container">
        <h1>🚀 Nginx on Kubernetes</h1>
        <p class="status">✓ Deployment Successful</p>
        <p>This page is served from a ConfigMap mounted to the Nginx container.</p>
        <p>Deployment details:</p>
        <ul>
          <li>Image: nginx:1.27-alpine</li>
          <li>Environment: production</li>
          <li>Timestamp: <span id="timestamp"></span></li>
        </ul>
      </div>
      <script>
        document.getElementById('timestamp').textContent = new Date().toISOString();
      </script>
    </body>
    </html>
  
  # Nginx 설정 파일
  nginx.conf: |
    server {
        listen 80;
        server_name _;
        
        root /usr/share/nginx/html;
        index index.html;
        
        # 로깅 설정
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;
        
        # 캐시 설정
        location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
        
        # 헬스 체크 경로
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
```

### ConfigMap을 사용하는 Deployment 수정

```yaml
# nginx-deployment-with-configmap.yaml (일부)
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        volumeMounts:
        - name: nginx-config
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx.conf
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
          items:
          - key: index.html
            path: index.html
          - key: nginx.conf
            path: nginx.conf
```

### ConfigMap 적용 및 확인

```bash
# ConfigMap 생성
kubectl apply -f nginx-configmap.yaml

# ConfigMap이 마운트된 Deployment 적용
kubectl apply -f nginx-deployment-with-configmap.yaml

# Deployment 재시작 (설정 변경 적용)
kubectl rollout restart deployment/nginx-deployment

# 변경 사항 확인
kubectl get pods -o wide
curl http://<노드-IP>:30080 | head -20
```

---

## 6단계: Ingress를 통한 도메인 기반 라우팅

프로덕션 환경에서는 NodePort보다 Ingress를 사용하여 도메인 기반 라우팅과 TLS 종료를 구현하는 것이 일반적입니다.

### Ingress 활성화 (Minikube)

```bash
# Minikube에서 Ingress 컨트롤러 활성화
minikube addons enable ingress

# Ingress 컨트롤러 상태 확인
kubectl get pods -n ingress-nginx
```

### Ingress 리소스 정의

```yaml
# nginx-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    # NGINX Ingress 컨트롤러 어노테이션
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.example.local  # 테스트용 호스트명
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

### Ingress 적용 및 테스트

```bash
# Ingress 리소스 생성
kubectl apply -f nginx-ingress.yaml

# Ingress 상태 확인
kubectl get ingress

# 호스트 파일에 항목 추가 (로컬 테스트용)
# Linux/macOS:
echo "$(minikube ip) nginx.example.local" | sudo tee -a /etc/hosts

# Windows (관리자 권한 PowerShell):
# Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "$(minikube ip) nginx.example.local"

# Ingress를 통한 접근 테스트
curl -H "Host: nginx.example.local" http://$(minikube ip)
```

---

## 7단계: HPA(Horizontal Pod Autoscaler) 구성

애플리케이션 부하에 따라 자동으로 Pod 수를 조정하는 HPA를 구성해 보겠습니다.

### Metrics Server 설치 (필요한 경우)

```bash
# Minikube에서 Metrics Server 활성화
minikube addons enable metrics-server

# Metrics Server 상태 확인
kubectl get pods -n kube-system | grep metrics-server

# 리소스 사용량 확인
kubectl top nodes
kubectl top pods
```

### HPA 생성

```bash
# CPU 사용률 50%를 목표로 HPA 생성
kubectl autoscale deployment nginx-deployment \
  --cpu-percent=50 \
  --min=2 \
  --max=10 \
  --name=nginx-hpa

# HPA 상태 확인
kubectl get hpa nginx-hpa -w
```

### 부하 테스트

```bash
# 부하 생성 테스트 Pod 실행
kubectl run load-test \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://nginx-service.default.svc.cluster.local; done"

# 또는 hey 도구를 사용한 부하 테스트
kubectl run hey-load-test \
  --image=rakyll/hey \
  --restart=Never \
  --command -- hey \
  -z 30s \
  -c 50 \
  -q 10 \
  http://nginx-service.default.svc.cluster.local

# HPA 상태 모니터링
kubectl get hpa nginx-hpa -w
kubectl get pods -l app=nginx
```

---

## 8단계: 모니터링, 로깅 및 문제 해결

### 기본 모니터링 명령어

```bash
# 전체 상태 확인
kubectl get all -l app=nginx

# 상세 정보 확인
kubectl describe deployment nginx-deployment
kubectl describe service nginx-service
kubectl describe ingress nginx-ingress

# 이벤트 확인 (시간순 정렬)
kubectl get events --sort-by=.lastTimestamp | tail -20

# 리소스 사용량
kubectl top pods -l app=nginx
kubectl top nodes

# 네트워크 엔드포인트 확인
kubectl get endpoints nginx-service -o yaml
```

### 로그 확인

```bash
# 특정 Pod의 로그 확인
kubectl logs -l app=nginx --tail=100

# 모든 Pod의 로그 확인
kubectl logs -l app=nginx --all-containers=true --tail=50

# 실시간 로그 모니터링
kubectl logs -l app=nginx -f

# 이전 컨테이너 로그 확인 (CrashLoopBackOff 상황)
kubectl logs -l app=nginx --previous
```

### 컨테이너 내부 진단

```bash
# 실행 중인 Pod에 접속
kubectl exec -it $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- sh

# 컨테이너 내부에서 명령 실행
kubectl exec deploy/nginx-deployment -- nginx -t
kubectl exec deploy/nginx-deployment -- ls -la /usr/share/nginx/html/

# 네트워크 연결 테스트
kubectl run -it --rm network-test --image=nicolaka/netshoot -- /bin/bash
# 컨테이너 내부에서:
# curl -I http://nginx-service.default.svc.cluster.local
# nslookup nginx-service
```

---

## 문제 해결 가이드

### 일반적인 문제와 해결 방법

**문제 1: Service에 연결할 수 없음**
```bash
# 엔드포인트 확인
kubectl get endpoints nginx-service

# Service와 Pod 라벨 일치 확인
kubectl get pods --show-labels
kubectl describe service nginx-service | grep -i selector

# Pod의 readinessProbe 상태 확인
kubectl describe pod <pod-name> | grep -A 10 "Readiness"
```

**문제 2: Pod가 CrashLoopBackOff 상태**
```bash
# 이전 로그 확인
kubectl logs <pod-name> --previous

# Pod 상세 정보 확인
kubectl describe pod <pod-name>

# ConfigMap 마운트 문제 확인
kubectl exec -it <pod-name> -- ls -la /usr/share/nginx/html/
```

**문제 3: Ingress가 작동하지 않음**
```bash
# Ingress 컨트롤러 상태 확인
kubectl get pods -n ingress-nginx

# Ingress 리소스 확인
kubectl describe ingress nginx-ingress

# Ingress 컨트롤러 로그 확인
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=100
```

**문제 4: HPA가 작동하지 않음**
```bash
# Metrics Server 상태 확인
kubectl get pods -n kube-system -l k8s-app=metrics-server

# HPA 이벤트 확인
kubectl describe hpa nginx-hpa

# Pod의 리소스 요청 확인 (HPA는 requests 기준으로 계산)
kubectl get pod <pod-name> -o yaml | grep -A 5 resources
```

### 진단 체크리스트

1. **Pod 상태 확인**: `kubectl get pods -o wide`
2. **Service와 엔드포인트 확인**: `kubectl get svc,ep`
3. **이벤트 로그 확인**: `kubectl get events --sort-by=.lastTimestamp`
4. **애플리케이션 로그 확인**: `kubectl logs -l app=nginx`
5. **리소스 사용량 확인**: `kubectl top pods`
6. **구성 검증**: `kubectl describe` 관련 리소스
7. **네트워크 연결 테스트**: 디버깅 Pod를 사용한 내부 연결 테스트

---

## 정리 및 리소스 삭제

### 리소스 정리

```bash
# 모든 리소스 삭제
kubectl delete -f nginx-deployment-production.yaml
kubectl delete -f nginx-service-nodeport.yaml
kubectl delete -f nginx-ingress.yaml
kubectl delete -f nginx-configmap.yaml
kubectl delete hpa nginx-hpa

# 또는 라벨을 사용한 일괄 삭제
kubectl delete all -l app=nginx
kubectl delete configmap -l app=nginx
kubectl delete ingress -l app=nginx
```

### 클러스터 정지 (Minikube)

```bash
# Minikube 정지
minikube stop

# Minikube 삭제 (필요한 경우)
minikube delete
```

---

## 결론

이 실습을 통해 Kubernetes에서 Nginx를 배포하고 운영하는 전 과정을 경험했습니다. 기본적인 Deployment와 Service 구성부터 시작하여 점차적으로 운영 환경에 필요한 기능들을 추가하는 방식으로 진행했습니다.

### 핵심 학습 내용 요약

1. **기본 워크로드 배포**: Deployment를 통한 Pod 관리와 Service를 통한 외부 노출
2. **운영 최적화**: 헬스 체크, 리소스 제한, 보안 컨텍스트를 통한 안정성 확보
3. **설정 관리**: ConfigMap을 통한 동적 설정 관리와 파일 주입
4. **트래픽 관리**: Ingress를 통한 도메인 기반 라우팅 구현
5. **자동 확장**: HPA를 통한 부하 기반 자동 확장 구성
6. **문제 해결**: 체계적인 모니터링과 디버깅 방법 습득

### 실무 적용을 위한 권장사항

1. **버전 관리**: 항상 명시적인 이미지 태그 사용 (`latest` 태그 피하기)
2. **헬스 체크**: readinessProbe와 livenessProbe 필수 구현
3. **리소스 관리**: 적절한 requests와 limits 설정
4. **보안 강화**: 비루트 사용자 실행, seccomp 프로필 적용
5. **설정 외부화**: ConfigMap과 Secret을 통한 설정 관리
6. **롤링 업데이트**: 무중단 배포를 위한 적절한 업데이트 전략 수립
7. **모니터링**: 지속적인 상태 모니터링과 로그 분석

이 실습을 템플릿으로 삼아 다양한 애플리케이션을 Kubernetes에 배포하고 운영하는 경험을 쌓아보시기 바랍니다. 각 애플리케이션의 특성에 맞게 구성 요소를 조정하고, 지속적으로 모니터링과 최적화를 진행하면 프로덕션 환경에서도 안정적인 서비스를 제공할 수 있습니다.