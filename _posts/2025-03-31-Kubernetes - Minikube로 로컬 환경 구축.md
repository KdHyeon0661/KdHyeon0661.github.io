---
layout: post
title: Kubernetes - Minikube로 로컬 환경 구축
date: 2025-03-31 21:20:23 +0900
category: Kubernetes
---
# Minikube로 로컬 Kubernetes 환경 구축하기

## 개요: 왜 Minikube를 사용해야 하는가?

Minikube는 로컬 머신에서 단일 노드 Kubernetes 클러스터를 쉽게 실행할 수 있는 경량 도구입니다. Kubernetes 학습, 애플리케이션 개발, CI/CD 파이프라인 테스트에 이상적인 환경을 제공하며, 실제 Kubernetes와 동일한 `kubectl` 명령어 인터페이스를 사용할 수 있습니다.

Minikube의 주요 장점은 다음과 같습니다:
- **간편한 설치와 실행**: 복잡한 설정 없이 몇 분 안에 Kubernetes 클러스터를 실행할 수 있습니다.
- **다양한 드라이버 지원**: Docker, Hyper-V, KVM, VirtualBox 등 다양한 가상화 기술을 지원합니다.
- **통합된 애드온 시스템**: Ingress, Metrics Server, Dashboard 등 유용한 컴포넌트를 쉽게 설치하고 관리할 수 있습니다.
- **개발 생산성 향상**: 로컬에서 빠르게 반복적인 개발과 테스트가 가능합니다.

이 가이드는 Minikube를 효과적으로 사용하기 위한 실용적인 정보를 제공하며, 드라이버 선택부터 개발 워크플로우 최적화, 실전 예제까지 포괄적으로 다룹니다.

---

## 시스템 요구사항과 드라이버 선택

### 운영체제별 요구사항

**Windows 환경**
- Hyper-V 또는 WSL2 + Docker Desktop 권장
- Windows 10/11 Pro, Enterprise, Education 버전 필요 (Hyper-V 사용 시)
- 최소 4GB RAM, 20GB 디스크 공간 권장

**macOS 환경** (Intel 및 Apple Silicon 모두 지원)
- Docker Desktop 권장
- 최소 4GB RAM, 20GB 디스크 공간 권장

**Linux 환경**
- KVM, VirtualBox 또는 Docker 드라이버 사용 가능
- 가상화 지원 CPU 권장 (KVM 사용 시)
- 최소 2GB RAM, 20GB 디스크 공간 권장

### 드라이버 선택 가이드

드라이버 선택은 사용 환경과 요구사항에 따라 결정해야 합니다:

| 사용 시나리오 | 권장 드라이버 | 장점 | 고려사항 |
|---------------|---------------|------|----------|
| 가상화 권한이 없거나 경량 환경 | **Docker** | 추가 VM 없이 컨테이너 기반으로 실행, 빠른 시작 | 격리 수준이 낮음 |
| 성능과 격리가 필요한 환경 | **Hyper-V** (Windows)<br>**HyperKit** (macOS)<br>**KVM** (Linux) | 완전한 가상화 격리, 자원 제어 용이 | 추가 리소스 소비, 설치 복잡성 |
| CI/CD 파이프라인 | **Docker** | 일관된 환경, 빠른 실행, 자원 효율 | 격리 제한 |

**실용적 조언**: 대부분의 개발 환경에서는 Docker 드라이버가 가장 간편하고 효율적입니다. 특히 Windows WSL2 환경에서는 Docker 드라이버가 우수한 성능을 제공합니다.

---

## Minikube 설치하기

### macOS (Homebrew 사용)
```bash
# Minikube 설치
brew install minikube

# Kubernetes CLI (kubectl) 설치
brew install kubectl

# 또는 Kubernetes CLI만 별도 설치
brew install kubernetes-cli
```

### Ubuntu/Debian Linux
```bash
# Minikube 바이너리 다운로드 및 설치
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# kubectl 설치
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Windows (PowerShell 관리자 모드)
```powershell
# Chocolatey 패키지 관리자 사용
choco install minikube
choco install kubernetes-cli

# 또는 직접 다운로드
# https://minikube.sigs.k8s.io/docs/start/ 에서 Windows 설치 프로그램 다운로드
```

### 설치 확인
```bash
# Minikube 버전 확인
minikube version

# kubectl 버전 확인
kubectl version --client
```

---

## Minikube 클러스터 시작과 관리

### 기본 클러스터 시작
```bash
# 가장 간단한 방법 (자동으로 적절한 드라이버 선택)
minikube start

# Docker 드라이버로 시작 (권장)
minikube start --driver=docker

# 자원 제한과 함께 시작
minikube start --driver=docker --cpus=4 --memory=8g --disk-size=50g
```

### 클러스터 상태 확인 및 관리
```bash
# 클러스터 상태 확인
minikube status

# Kubernetes 대시보드 열기 (웹 브라우저에서)
minikube dashboard

# 클러스터 중지
minikube stop

# 클러스터 삭제
minikube delete

# 클러스터 IP 확인
minikube ip

# SSH 접속
minikube ssh
```

### Windows WSL2 특별 지침
Windows에서 WSL2를 사용하는 경우:
1. Docker Desktop 설치 후 "Use the WSL 2 based engine" 옵션 활성화
2. WSL2 배포판에 Docker CLI 설치
3. 다음 명령어로 Minikube 시작:
```bash
minikube start --driver=docker --container-runtime=containerd
```

---

## 기본적인 Kubernetes 워크로드 실행

### 간단한 Nginx 배포
```bash
# Nginx 디플로이먼트 생성
kubectl create deployment nginx --image=nginx:1.27-alpine

# 디플로이먼트 상태 확인
kubectl get deployments

# 포드 상태 확인
kubectl get pods

# 서비스 노출 (NodePort 타입)
kubectl expose deployment nginx --type=NodePort --port=80

# 서비스 확인
kubectl get services

# 브라우저에서 접속
minikube service nginx
```

### YAML 파일을 통한 선언적 배포
```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
```

```bash
# YAML 파일 적용
kubectl apply -f nginx-deployment.yaml

# 배포 상태 확인
kubectl get deployments
kubectl get pods
```

---

## 개발 워크플로우 최적화: 빠른 이미지 반영

로컬 개발 환경에서 코드 변경을 빠르게 테스트하려면 효율적인 이미지 빌드와 배포 프로세스가 필요합니다.

### 방법 1: Minikube 이미지 로드 (권장)
이 방법은 로컬에서 빌드한 이미지를 Minikube 클러스터에 직접 로드합니다.

```bash
# 로컬에서 Docker 이미지 빌드
docker build -t myapp:latest .

# Minikube에 이미지 로드
minikube image load myapp:latest

# 디플로이먼트 매니페스트에서 이미지 사용
# imagePullPolicy를 IfNotPresent 또는 Never로 설정
```

디플로이먼트 YAML 예시:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        imagePullPolicy: IfNotPresent  # 중요: Always 대신 사용
```

### 방법 2: Minikube 내부 Docker 데몬 사용
이 방법은 Minikube의 Docker 환경에서 직접 이미지를 빌드합니다.

```bash
# Minikube의 Docker 환경 사용
eval $(minikube docker-env)

# Minikube 내부에서 이미지 빌드
docker build -t myapp:latest .

# 환경 변수 원래대로 복원
eval $(minikube docker-env -u)
```

### 방법 3: 레지스트리 사용 (팀 개발 환경)
팀 환경이나 CI/CD 파이프라인에서는 레지스트리를 사용하는 것이 좋습니다.

```bash
# 이미지 태그에 레지스트리 주소 포함
docker build -t registry.example.com/myapp:v1.0.0 .

# 레지스트리에 푸시
docker push registry.example.com/myapp:v1.0.0

# Kubernetes에서 이미지 풀
kubectl set image deployment/myapp myapp=registry.example.com/myapp:v1.0.0
```

---

## 고급 기능 설정과 사용

### 애드온(Addons) 활성화
Minikube는 유용한 Kubernetes 컴포넌트를 애드온 형태로 제공합니다.

```bash
# 사용 가능한 애드온 목록 확인
minikube addons list

# Ingress 컨트롤러 활성화 (URL 기반 라우팅)
minikube addons enable ingress

# Metrics Server 활성화 (리소스 모니터링 및 HPA 지원)
minikube addons enable metrics-server

# Dashboard 활성화 (웹 기반 관리 인터페이스)
minikube addons enable dashboard

# Helm 활성화 (패키지 관리자)
minikube addons enable helm
```

### Ingress를 통한 호스트 기반 라우팅
Ingress를 사용하면 단일 IP 주소로 여러 서비스를 도메인 기반으로 라우팅할 수 있습니다.

1. **디플로이먼트와 서비스 생성**
```yaml
# web-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

2. **Ingress 리소스 생성**
```yaml
# web-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: webapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

3. **호스트 파일에 항목 추가**
```bash
# Minikube IP 확인
minikube ip

# /etc/hosts 파일에 항목 추가 (Linux/macOS)
echo "$(minikube ip) webapp.local" | sudo tee -a /etc/hosts

# Windows의 경우 C:\Windows\System32\drivers\etc\hosts 파일 수정
```

4. **브라우저에서 접속**
`http://webapp.local` 에 접속하여 웹 애플리케이션에 접근할 수 있습니다.

### Horizontal Pod Autoscaler(HPA) 설정
Metrics Server가 활성화된 상태에서 HPA를 사용하여 부하에 따라 파드 수를 자동으로 조정할 수 있습니다.

```bash
# CPU 사용률을 기준으로 하는 HPA 생성
kubectl autoscale deployment web-app --cpu-percent=50 --min=1 --max=5

# HPA 상태 확인
kubectl get hpa

# 상세 정보 확인
kubectl describe hpa web-app
```

부하 테스트를 위한 샘플 애플리케이션:
```yaml
# stress-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stress-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stress
  template:
    metadata:
      labels:
        app: stress
    spec:
      containers:
      - name: stress
        image: vish/stress
        args: ["--cpu", "2", "--timeout", "300s"]
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

---

## 영구적 스토리지 사용하기

로컬 개발 환경에서도 데이터를 영구적으로 저장해야 하는 경우가 있습니다. Minikube는 다양한 스토리지 옵션을 제공합니다.

### 퍼시스턴트 볼륨(Persistent Volume) 사용
```yaml
# persistent-volume-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```yaml
# deployment-with-pvc.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-storage
spec:
  replicas: 1
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
        image: nginx:1.27-alpine
        volumeMounts:
        - name: data-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: data-storage
        persistentVolumeClaim:
          claimName: my-data-pvc
```

### 호스트 디렉토리 마운트
개발 환경에서는 호스트 디렉토리를 직접 마운트하는 것이 편리할 수 있습니다.

```yaml
# host-path-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: myapp
    image: nginx:1.27-alpine
    volumeMounts:
    - name: host-data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: host-data
    hostPath:
      path: /data/myapp
      type: DirectoryOrCreate
```

---

## 고급 구성 옵션

### 다중 프로필 관리
여러 개의 독립적인 클러스터를 관리해야 할 때 프로필 기능을 사용할 수 있습니다.

```bash
# 팀별로 다른 프로필 생성
minikube start -p team-a --driver=docker --cpus=4 --memory=6g
minikube start -p team-b --driver=docker --cpus=2 --memory=4g

# 프로필 목록 확인
minikube profile list

# 특정 프로필 활성화
minikube profile team-a

# 프로필 삭제
minikube delete -p team-b
```

### 다중 노드 클러스터
Minikube에서도 다중 노드 클러스터를 구성할 수 있습니다.

```bash
# 2개 노드로 클러스터 시작
minikube start --nodes=2 --driver=docker

# 노드 상태 확인
kubectl get nodes -o wide

# 특정 노드에 파드 스케줄링
# nodeSelector 또는 nodeAffinity 사용
```

### 프록시 환경에서의 Minikube 사용
회사 네트워크나 프록시 환경에서 Minikube를 사용해야 하는 경우:

```bash
# 프록시 설정과 함께 Minikube 시작
minikube start \
  --docker-env HTTP_PROXY=http://proxy.example.com:3128 \
  --docker-env HTTPS_PROXY=http://proxy.example.com:3128 \
  --docker-env NO_PROXY=localhost,127.0.0.1,10.96.0.0/12,192.168.59.0/24
```

### 프라이빗 레지스트리 인증
사설 컨테이너 레지스트리를 사용하는 경우:

```bash
# Docker 레지스트리 시크릿 생성
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=your-username \
  --docker-password=your-password \
  --docker-email=your-email@example.com
```

디플로이먼트에서 시크릿 사용:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-app
spec:
  template:
    spec:
      imagePullSecrets:
      - name: regcred
      containers:
      - name: app
        image: registry.example.com/your-app:latest
```

---

## 문제 해결과 디버깅

### 일반적인 문제 해결

**문제 1: Docker 드라이버 시작 실패**
```bash
# 오래된 Minikube 리소스 정리
minikube delete --all --purge

# Docker 서비스 재시작
sudo systemctl restart docker  # Linux
# 또는 Docker Desktop 재시작 (Windows/macOS)

# 다시 시도
minikube start --driver=docker --force
```

**문제 2: kubectl이 다른 컨텍스트를 가리킴**
```bash
# 현재 컨텍스트 확인
kubectl config current-context

# Minikube 컨텍스트로 전환
kubectl config use-context minikube

# 사용 가능한 컨텍스트 목록
kubectl config get-contexts
```

**문제 3: Ingress가 작동하지 않음**
```bash
# Ingress 컨트롤러 상태 확인
kubectl get pods -n ingress-nginx

# Ingress 리소스 상세 정보 확인
kubectl describe ingress my-ingress

# 서비스와 엔드포인트 확인
kubectl get svc,ep

# 호스트 파일 확인
cat /etc/hosts  # 또는 Windows의 hosts 파일
```

**문제 4: HPA가 동작하지 않음**
```bash
# Metrics Server 상태 확인
kubectl get pods -n kube-system | grep metrics-server

# Metrics Server 로그 확인
kubectl logs -n kube-system deployment/metrics-server

# 파드 메트릭 확인
kubectl top pods

# HPA 이벤트 확인
kubectl describe hpa my-hpa
```

### 디버깅 명령어 모음
```bash
# 클러스터 이벤트 확인
kubectl get events --sort-by=.lastTimestamp -A

# 파드 로그 확인
kubectl logs <pod-name>
kubectl logs -f <pod-name>  # 실시간 로그

# 파드 상세 정보
kubectl describe pod <pod-name>

# 리소스 사용량 확인
kubectl top pods -A
kubectl top nodes

# 네트워크 진단
kubectl get svc,ep -A
kubectl get endpointslices -A

# 구성 확인
kubectl config view
kubectl cluster-info
```

---

## 종합 실전 예제: 완전한 애플리케이션 스택

다음은 웹 애플리케이션, API 서비스, Ingress, HPA를 포함한 완전한 예제입니다.

```yaml
# full-stack-example.yaml
---
# 웹 애플리케이션 디플로이먼트
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
---
# 웹 애플리케이션 서비스
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
---
# API 서비스 디플로이먼트
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args: ["-text=Hello from API v1.0"]
        ports:
        - containerPort: 5678
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
---
# API 서비스
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 5678
---
# Ingress 리소스
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: full-stack-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: web.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: api.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
---
# API HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

실행 방법:
```bash
# 필수 애드온 활성화
minikube addons enable ingress
minikube addons enable metrics-server

# YAML 파일 적용
kubectl apply -f full-stack-example.yaml

# 호스트 파일에 항목 추가
echo "$(minikube ip) web.local api.local" | sudo tee -a /etc/hosts

# 브라우저에서 접속
# http://web.local/
# http://api.local/
```

---

## 결론

Minikube는 Kubernetes 학습과 개발을 위한 강력하고 필수적인 도구입니다. 이 가이드에서 소개한 내용을 통해 다음과 같은 역량을 기를 수 있습니다:

1. **효율적인 개발 환경 구축**: OS와 요구사항에 맞는 최적의 드라이버 선택과 구성
2. **생산적인 개발 워크플로우**: 빠른 이미지 빌드와 배포를 통한 빠른 개발 사이클
3. **실전 Kubernetes 기능 활용**: Ingress, HPA, 영구적 스토리지 등 프로덕션 수준의 기능 사용
4. **문제 해결 능력**: 일반적인 문제에 대한 진단과 해결 방법 습득

### 다음 단계 제안

Minikube에 익숙해진 후에는 다음 단계로 나아가시기를 권장합니다:

1. **헬름(Helm) 또는 Kustomize 학습**: Kubernetes 매니페스트의 템플릿화와 관리 자동화
2. **GitOps 도구 탐색**: Argo CD 또는 Flux를 통한 선언적 배포 파이프라인 구축
3. **모니터링 스택 구현**: Prometheus, Grafana, OpenTelemetry를 통한 관측 가능성 확보
4. **다른 로컬 Kubernetes 도구 비교**: Kind, K3d, MicroK8s와의 비교 학습
5. **CI/CD 파이프라인 통합**: Minikube를 활용한 로컬 CI/CD 테스트 환경 구축

Minikube는 Kubernetes 여정의 시작점일 뿐입니다. 이 기초를 바탕으로 실제 프로덕션 환경, 멀티 클러스터 관리, 클라우드 네이티브 애플리케이션 개발로 나아가시기 바랍니다. 가장 중요한 것은 지속적인 실험과 학습을 통해 Kubernetes 생태계에 대한 이해를 깊게 하는 것입니다.

### 유용한 명령어 참고서

```bash
# 클러스터 관리
minikube start --driver=docker --cpus=4 --memory=8g
minikube status
minikube stop
minikube delete --all

# 애드온 관리
minikube addons list
minikube addons enable ingress
minikube addons enable metrics-server

# 개발 도구
minikube dashboard
minikube service <service-name>
minikube image load <image-name>:<tag>

# 진단 도구
minikube ssh
minikube ip
minikube logs
```

이제 Minikube를 활용하여 Kubernetes 학습과 개발의 새로운 장을 열어보시기 바랍니다. 행운을 빕니다