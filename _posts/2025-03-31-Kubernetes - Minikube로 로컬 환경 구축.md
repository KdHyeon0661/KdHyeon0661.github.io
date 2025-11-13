---
layout: post
title: Kubernetes - Minikube로 로컬 환경 구축
date: 2025-03-31 21:20:23 +0900
category: Kubernetes
---
# Minikube로 로컬 환경 구축하기

## 들어가며

- 드라이버 선택(가상화 vs Docker)과 OS별 주의점(WSL2/Hyper-V/KVM)
- **개발 루프 최적화**(이미지 빌드/반영, `minikube image load`, `imagePullPolicy`)
- **Ingress, Metrics Server, HPA** 실전 예제
- **퍼시스턴트 볼륨, 로컬 스토리지** 실습
- **멀티 프로필/리소스 제한**, **프록시/사설 레지스트리**, **디버깅**과 **자주 겪는 오류 해결**

---

## Minikube란?
로컬 머신에서 **단일 노드 K8s**를 손쉽게 띄우는 경량 도구. 학습·개발·테스트에 적합하며 `kubectl`과 동일한 UX를 제공한다.

- 단일 노드(옵션으로 멀티 노드 실험 가능)
- 다양한 드라이버: Hyper-V/HyperKit/KVM/VirtualBox/**Docker**
- **애드온(addons)** 으로 Ingress, Metrics Server 등 손쉬운 활성화

---

## 1. 사전 준비(요구 사항·드라이버 선택)

### OS & 가상화
- **Windows**: Hyper-V 또는 **WSL2 + Docker Desktop** 권장
- **macOS(Intel/Apple Silicon)**: HyperKit(구), **Docker Desktop** 권장
- **Linux**: KVM/VirtualBox/**Docker** 중 택1 (개발자 경험은 Docker가 단순)

### 드라이버 선택 가이드
| 상황 | 권장 드라이버 | 비고 |
|---|---|---|
| 가상화 권한 불가 / 가볍게 | **Docker** | 추가 VM 없이 컨테이너로 동작 |
| 가상화 성능/격리 필요 | Hyper-V/HyperKit/KVM | 자원 여유 필요 |
| CI 환경 | Docker | 일관성/속도 유리 |

---

## 2. 설치

### macOS (Homebrew)
```bash
brew install minikube
brew install kubectl     # 필요 시
```

### Ubuntu / Debian
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
sudo apt-get install -y kubectl  # 필요 시
```

### Windows (Chocolatey)
```powershell
choco install minikube
choco install kubernetes-cli   # kubectl
```

설치 검증:
```bash
minikube version
kubectl version --client
```

---

## 3. 클러스터 시작/중지/삭제

### 기본 시작(가상 머신/드라이버 자동 결정)
```bash
minikube start
```

### Docker 드라이버로 시작(가상화 없이)
```bash
minikube start --driver=docker
```

### 자원 제한(개발용 권장 예)
```bash
minikube start --driver=docker --cpus=4 --memory=8g --disk-size=50g
```

상태/중지/삭제:
```bash
minikube status
minikube stop
minikube delete
```

> **WSL2(Windows)**: Docker Desktop에서 **Use the WSL 2 based engine** 활성화 권장.

---

## 4. 대시보드와 기본 배포

### 대시보드
```bash
minikube dashboard
```

### Nginx 배포 + Service(NodePort)
```bash
kubectl create deployment nginx --image=nginx:1.27-alpine
kubectl expose deployment nginx --type=NodePort --port=80
minikube service nginx  # 브라우저 자동 오픈
```

---

## 5. 개발 루프 최적화 — 이미지 반영 흐름

로컬에서 코드를 바꾸고 **즉시** Minikube에 반영하려면:

### 5.1 `minikube image load` (권장)
```bash
# 로컬에서 이미지 빌드
docker build -t demo/web:dev .

# Minikube 노드 캐시에 푸시 없이 적재
minikube image load demo/web:dev

# 매니페스트에서 동일 태그 사용 & Pull 정책 주의
# (개발 시: IfNotPresent 또는 Never)
```

Deployment 예시 스니펫:
```yaml
spec:
  template:
    spec:
      containers:
      - name: web
        image: demo/web:dev
        imagePullPolicy: IfNotPresent
```

### 5.2 Minikube 내 Docker 데몬 사용(대안)
```bash
eval $(minikube docker-env)  # 이후 docker build가 Minikube 내부에 빌드됨
docker build -t demo/web:dev .
```

> CI/팀 공유는 **레지스트리** 사용 권장(아래 참조).

---

## 6. 애드온(addons) — Ingress·Metrics Server 등

### 목록/활성화
```bash
minikube addons list
minikube addons enable ingress
minikube addons enable metrics-server
```

- **ingress**: NGINX Ingress Controller 설치
- **metrics-server**: HPA 동작에 필요한 리소스 메트릭 제공

---

## 7. Ingress 실전 (호스트 라우팅)

### 7.1 Deployment/Service
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: web }
spec:
  replicas: 2
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
      - name: web
        image: nginx:1.27-alpine
        ports: [{ containerPort: 80 }]
---
apiVersion: v1
kind: Service
metadata: { name: web-svc }
spec:
  selector: { app: web }
  ports:
  - port: 80
    targetPort: 80
```

### 7.2 Ingress 리소스
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ing
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
            name: web-svc
            port:
              number: 80
```

호스트네임 매핑:
```bash
# Linux/macOS
echo "$(minikube ip) web.local" | sudo tee -a /etc/hosts

# Windows (관리자) - C:\Windows\System32\drivers\etc\hosts 파일에 추가
# <MINIKUBE_IP> web.local
```
접속: `http://web.local/`

---

## 8. HPA(오토스케일링)와 Metrics Server

### 8.1 로드 테스트용 샘플 앱
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: cpu-demo }
spec:
  replicas: 1
  selector: { matchLabels: { app: cpu-demo } }
  template:
    metadata: { labels: { app: cpu-demo } }
    spec:
      containers:
      - name: app
        image: vish/stress
        args: ["--cpu", "2", "--timeout", "300s"]
        resources:
          requests: { cpu: "100m" }
          limits: { cpu: "500m" }
---
apiVersion: v1
kind: Service
metadata: { name: cpu-demo-svc }
spec:
  selector: { app: cpu-demo }
  ports:
  - port: 80
    targetPort: 8080
```

### 8.2 HPA 생성
```bash
kubectl autoscale deployment cpu-demo --cpu-percent=50 --min=1 --max=5
kubectl get hpa
```

> **metrics-server** 애드온이 켜져 있어야 한다.

---

## 9. 퍼시스턴트 볼륨(PV/PVC) — 로컬 개발 데이터 보존

Minikube 기본 StorageClass(운영체제/드라이버에 따라 상이)를 활용해 PVC 바인딩:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data-pvc }
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: app }
spec:
  replicas: 1
  selector: { matchLabels: { app: app } }
  template:
    metadata: { labels: { app: app } }
    spec:
      containers:
      - name: app
        image: nginx:1.27-alpine
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: data-pvc
```

---

## 10. 멀티 프로필·멀티 노드·리소스 조정

### 10.1 프로필
```bash
minikube start -p team-a --driver=docker --cpus=4 --memory=6g
minikube start -p team-b --driver=docker --cpus=2 --memory=4g
minikube profile list
minikube delete -p team-b
```

### 10.2 (실험) 멀티 노드
```bash
minikube start --nodes=2 --driver=docker
kubectl get nodes
```

### 10.3 동작 중 리소스 확장
```bash
minikube stop
minikube start --memory=10g --cpus=6
```

---

## 11. 프록시·사설 레지스트리 연동

### 11.1 HTTP 프록시 하에서 시작
```bash
minikube start \
  --docker-env HTTP_PROXY=http://proxy.example.com:3128 \
  --docker-env HTTPS_PROXY=http://proxy.example.com:3128 \
  --docker-env NO_PROXY=localhost,127.0.0.1,10.96.0.0/12
```

### 11.2 프라이빗 레지스트리 인증
```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=USER --docker-password=PASS --docker-email=dev@example.com
```

Deployment에서 사용:
```yaml
imagePullSecrets:
- name: regcred
```

---

## 12. 디버깅·관측 기본기

### 12.1 이벤트/로그/리소스
```bash
kubectl get events --sort-by=.lastTimestamp -A
kubectl logs deploy/nginx
kubectl top pod -A
kubectl describe pod <name>
```

### 12.2 노드/네트워크 경로 점검
```bash
kubectl get nodes -o wide
kubectl get svc,ep -A
kubectl get endpointslices -A
```

---

## 13. 자주 발생하는 문제 해결(확장판)

| 증상 | 원인 | 해결 |
|---|---|---|
| **Docker 드라이버 시작 실패** | 오래된 컨테이너/이미지 충돌 | `minikube delete` 후 `minikube start --driver=docker` 재생성 |
| **`kubectl`이 다른 컨텍스트를 봄** | kubeconfig 컨텍스트 오프셋 | `kubectl config use-context minikube` |
| **가상화 오류** | VT-x/AMD-V 비활성 | BIOS/UEFI에서 가상화 옵션 활성화 |
| **Ingress 404** | IngressClass/호스트/경로 불일치 | `kubectl describe ingress`, `/etc/hosts` 매핑 재확인 |
| **HPA 미작동** | metrics-server 비활성/권한 | `minikube addons enable metrics-server`, `kubectl top pod` 확인 |
| **이미지 최신 반영 실패** | Pull 정책/태그 캐싱 | `imagePullPolicy: IfNotPresent` 또는 태그 변경, `minikube image load` 재실행 |
| **PVC 바인딩 실패** | SC/접근모드 불일치 | `kubectl get sc`, `kubectl describe pvc`로 SC/용량/모드 점검 |

---

## 14. 간단 성능/용량 개념식(학습용)

요청률(QPS)과 평균 처리 시간 \(t\)일 때 동시 처리량은
$$
\text{동시 처리량} \approx \text{QPS} \cdot t
$$

파드 1개가 처리 가능한 동시 처리량 \(\text{cap}_{pod}\)라면, 최소 레플리카 수:
$$
\text{replicas}_{\min}=
\left\lceil
\frac{\text{QPS}\cdot t}{\text{cap}_{pod}}
\right\rceil \cdot \text{버퍼}
$$

> 학습용 개념치로 버퍼 1.2~1.5를 적용해 스파이크 흡수 후, HPA로 동적 보정.

---

## 15. 하나로 묶기 — 로컬에서 “웹+API+Ingress+HPA” 미니 스택

```yaml
# web
apiVersion: apps/v1
kind: Deployment
metadata: { name: web }
spec:
  replicas: 2
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
      - name: web
        image: nginx:1.27-alpine
        ports: [{ containerPort: 80 }]
---
apiVersion: v1
kind: Service
metadata: { name: web-svc }
spec:
  selector: { app: web }
  ports: [{ port: 80, targetPort: 80 }]
---
# api
apiVersion: apps/v1
kind: Deployment
metadata: { name: api }
spec:
  replicas: 1
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args: ["-text=hello from api"]
        ports: [{ containerPort: 5678 }]
        resources:
          requests: { cpu: "100m" }
---
apiVersion: v1
kind: Service
metadata: { name: api-svc }
spec:
  selector: { app: api }
  ports: [{ port: 80, targetPort: 5678 }]
---
# ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: mini-stack }
spec:
  ingressClassName: nginx
  rules:
  - host: web.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: web-svc, port: { number: 80 } } }
  - host: api.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: api-svc, port: { number: 80 } } }
---
# HPA for api
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 60 }
```

실행 순서:
```bash
minikube addons enable ingress
minikube addons enable metrics-server
kubectl apply -f mini-stack.yaml
echo "$(minikube ip) web.local api.local" | sudo tee -a /etc/hosts
```
접속: `http://web.local/`, `http://api.local/`

---

## 16. 정리 및 다음 단계
- Minikube는 **로컬 학습/개발 환경의 표준**. 드라이버/애드온/이미지 루프/Ingress/HPA/스토리지를 익히면 실전 감각을 빠르게 얻는다.
- 다음 단계 제안:
  1) **Helm/Kustomize**로 매니페스트 템플릿화
  2) **GitOps(Argo CD/Flux)**로 선언형 배포 습관화
  3) **OpenTelemetry + Prom/Grafana**로 관측 기초 정착
  4) **Kind**와 비교 학습(빠른 CI용), **로컬 레지스트리 미러링** 실습

부록: 자주 쓰는 명령 모음
```bash
minikube start --driver=docker --cpus=4 --memory=8g
minikube status
minikube dashboard
minikube service <svc>
minikube ip
minikube addons list
minikube addons enable ingress
minikube addons enable metrics-server
minikube image load demo/web:dev
minikube ssh
minikube profile list
minikube stop && minikube delete
```
