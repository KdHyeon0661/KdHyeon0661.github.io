---
layout: post
title: Kubernetes - Kind로 클러스터 구성하기
date: 2025-03-31 22:20:23 +0900
category: Kubernetes
---
# Kind(Kubernetes in Docker)로 클러스터 구성하기

## Kind 소개

**Kind(Kubernetes in Docker)** 는 Docker 컨테이너를 가상 노드처럼 사용하여 가볍고 빠른 로컬 쿠버네티스 클러스터를 생성하는 도구입니다. 별도의 하이퍼바이저(가상머신)가 필요 없어 설치와 실행이 매우 간단하며, 몇 초 만에 완전한 쿠버네티스 클러스터를 띄울 수 있습니다. 로컬 개발, CI/CD 파이프라인에서의 엔드투엔드(E2E) 테스트, 그리고 교육 목적으로 널리 사용되고 있습니다.

## 시작하기 전 준비사항

- **Docker**: Docker 데몬이 설치되어 실행 중이어야 합니다.
- **kubectl** (권장): 생성된 클러스터와 상호작용하기 위한 쿠버네티스 클라이언트 도구입니다.
- **Kind 바이너리**: 아래 설치 가이드를 따라 Kind를 설치합니다.

## Kind 설치

### macOS (Homebrew 사용)
```bash
brew install kind
```

### Linux (직접 바이너리 다운로드)
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Windows (Chocolatey 사용)
```powershell
choco install kind
```

설치 확인:
```bash
kind version
```

## 빠른 시작: 기본 클러스터 생성

가장 간단한 방법으로 단일 컨트롤 플레인 노드를 가진 클러스터를 생성할 수 있습니다.
```bash
kind create cluster
```
생성 후 `kubectl`을 사용하여 노드 상태를 확인합니다.
```bash
kubectl get nodes
```
생성된 노드는 Docker 컨테이너로 실행되며, 다음 명령어로 확인할 수 있습니다.
```bash
docker ps
```

---

## 고급 구성: 멀티 노드 클러스터 및 기능 활성화

실제 운영 환경에 더 가까운 테스트를 위해, 또는 특정 기능을 실습하기 위해 클러스터 구성을 커스터마이징할 수 있습니다. 아래 YAML 파일은 1개의 컨트롤 플레인 노드와 2개의 워커 노드로 구성된 클러스터를 정의하며, Ingress 컨트롤러 접근을 위한 포트 포워딩과 호스트 디렉토리 마운트를 설정합니다.

```yaml
# cluster-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: my-cluster
nodes:
  - role: control-plane
    image: kindest/node:v1.30.0
    extraPortMappings:
      - containerPort: 80   # Ingress HTTP 트래픽용
        hostPort: 80
        protocol: TCP
      - containerPort: 443  # Ingress HTTPS 트래픽용
        hostPort: 443
        protocol: TCP
    extraMounts:
      - hostPath: /home/user/workspace
        containerPath: /mnt/workspace
  - role: worker
    image: kindest/node:v1.30.0
  - role: worker
    image: kindest/node:v1.30.0
kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        "service-node-port-range": "30000-32767"
```

커스텀 설정 파일로 클러스터를 생성합니다.
```bash
kind create cluster --config cluster-config.yaml
```
생성 후 컨텍스트를 확인하고 노드 정보를 조회합니다.
```bash
kubectl cluster-info --context kind-my-cluster
kubectl get nodes -o wide
```

---

## Ingress 컨트롤러 설정 및 외부 접근

Kind 클러스터에 애플리케이션을 배포하고 외부(호스트 머신)에서 접근하려면 Ingress 컨트롤러가 필요합니다. Nginx Ingress Controller는 가장 일반적인 선택입니다.

### Ingress-NGINX 설치
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller
```
위 매니페스트는 Kind 환경에 최적화되어 있으며, 앞서 `cluster-config.yaml`에서 설정한 `extraPortMappings`(호스트 80/443 포트)를 통해 Ingress 트래픽을 수신합니다.

### 테스트 애플리케이션 배포
다음 YAML은 간단한 에코 서비스를 배포하고 Ingress 규칙을 설정합니다.

```yaml
# echo-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  labels:
    app: echo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo
        args: ["-text=hello from kind"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: echo-svc
spec:
  selector:
    app: echo
  ports:
  - port: 80
    targetPort: 5678
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ing
spec:
  ingressClassName: nginx
  rules:
  - host: echo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: echo-svc
            port:
              number: 80
```

애플리케이션을 적용하고 로컬 호스트 파일에 도메인을 추가한 후 접속을 테스트합니다.
```bash
kubectl apply -f echo-app.yaml
echo "127.0.0.1 echo.local" | sudo tee -a /etc/hosts
curl http://echo.local/
# 응답: "hello from kind"
```

---

## MetalLB를 이용한 LoadBalancer 서비스 실습

기본 Kind 클러스터에는 클라우드 제공자와 같은 외부 로드 밸런서가 없어 `LoadBalancer` 타입의 서비스가 `Pending` 상태로 머무릅니다. **MetalLB**는 베어메탈 환경에서 로드 밸런서 서비스를 제공하는 CNCF 프로젝트로, Kind에서 `LoadBalancer` 타입을 실습하는 데 적합합니다.

### MetalLB 설치 및 구성
```bash
# MetalLB 설치
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
kubectl -n metallb-system rollout status deploy/controller

# MetalLB에 IP 주소 풀 제공 (Kind Docker 네트워크 대역 확인 필요)
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: kind-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.18.255.200-172.18.255.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2adv
  namespace: metallb-system
spec:
  ipAddressPools:
  - kind-pool
EOF
```
> **중요**: `IPAddressPool`의 주소 범위는 Kind가 사용하는 Docker 네트워크(`docker network inspect kind`)의 서브넷 내에 있어야 합니다.

### LoadBalancer 서비스 테스트
앞서 배포한 에코 서비스를 `LoadBalancer` 타입으로 다시 노출합니다.
```yaml
# echo-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-lb
spec:
  type: LoadBalancer
  selector:
    app: echo
  ports:
    - port: 80
      targetPort: 5678
```
```bash
kubectl apply -f echo-lb.yaml
kubectl get svc echo-lb # EXTERNAL-IP가 할당되었는지 확인
curl http://<EXTERNAL-IP>/
```

---

## 로컬 이미지 사용 및 개발 워크플로 최적화

### 방법 1: 직접 이미지 로드 (간단)
로컬에서 빌드한 Docker 이미지를 Kind 클러스터에 직접 로드할 수 있습니다.
```bash
docker build -t my-app:dev .
kind load docker-image my-app:dev --name my-cluster
```
이후 Deployment 매니페스트에서 `image: my-app:dev`와 `imagePullPolicy: IfNotPresent`를 사용하면 됩니다.

### 방법 2: 로컬 레지스트리 연동 (권장, 팀/CI 환경)
개발 및 CI 환경에서 이미지 푸시/풀을 표준화하려면 로컬 Docker 레지스트리 컨테이너를 실행하고 Kind 클러스터가 이를 미러 레지스트리로 인식하도록 구성합니다.

```bash
# 1. 로컬 레지스트리 컨테이너 실행
docker run -d --restart=always -p "5001:5000" --name registry registry:2

# 2. 레지스트리 미러를 참조하는 Kind 구성 파일 생성
cat <<EOF > kind-with-registry.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: my-cluster
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:5001"]
    endpoint = ["http://registry:5000"]
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

# 3. 클러스터 생성 및 레지스트리 컨테이너를 Kind 네트워크에 연결
kind create cluster --config kind-with-registry.yaml
docker network connect kind registry
```

이제 이미지를 로컬 레지스트리에 푸시하고 쿠버네티스에서 참조할 수 있습니다.
```bash
docker build -t localhost:5001/my-app:dev .
docker push localhost:5001/my-app:dev
# Deployment YAML: image: localhost:5001/my-app:dev
```

---

## 스토리지: 호스트와 Pod 간 데이터 공유

`cluster-config.yaml`의 `extraMounts`를 사용하면 호스트 디렉토리를 Kind 노드에 마운트할 수 있습니다. 이를 활용하여 Pod의 `hostPath` 볼륨으로 연결하면, 호스트에서 코드를 수정할 때 Pod에서 즉시 변경 사항을 반영할 수 있는 빠른 개발 루프를 구성할 수 있습니다.

**구성 예시 (정적 웹사이트 개발):**
1.  `cluster-config.yaml`에 `extraMounts`로 `/home/user/workspace/site`를 노드의 `/mnt/workspace/site`에 마운트합니다.
2.  아래와 같이 Deployment를 생성하여 해당 노드 경로를 Pod에 마운트합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-dev
  template:
    metadata:
      labels:
        app: web-dev
    spec:
      containers:
      - name: web
        image: nginx:alpine
        volumeMounts:
        - name: code
          mountPath: /usr/share/nginx/html
      volumes:
      - name: code
        hostPath:
          path: /mnt/workspace/site # Kind 노드 내부 경로
          type: Directory
```
이제 호스트의 `/home/user/workspace/site` 디렉토리에서 HTML 파일을 수정하면, Nginx Pod에서 즉시 서빙되는 내용이 업데이트됩니다.

---

## 네트워크 문제 해결 및 디버깅

쿠버네티스 네트워크 문제를 진단할 때 유용한 도구와 명령어입니다.

- **네트워크 진단 도구 Pod 실행**: `netshoot` 이미지는 네트워크 문제 해결에 필요한 다양한 도구를 포함하고 있습니다.
    ```bash
    kubectl run -it --rm netshoot --image=nicolaka/netshoot -- /bin/bash
    # 컨테이너 내부에서 다음 명령 실행
    curl -v http://echo-svc.default.svc.cluster.local
    nslookup echo-svc.default.svc.cluster.local
    ```
- **이벤트 및 리소스 상태 확인**:
    ```bash
    kubectl get events -A --sort-by=.lastTimestamp
    kubectl top pod -A
    kubectl get endpointslices -A
    ```

---

## CI/CD 파이프라인에서의 Kind 활용 (GitHub Actions 예시)

Kind는 헤드리스 환경에서도 잘 동작하여 CI 파이프라인에서 쿠버네티스 매니페스트의 유효성과 애플리케이션의 기본 기능을 검증하는 데 이상적입니다.

```yaml
# .github/workflows/e2e-kind.yml
name: e2e-kind
on: [push, pull_request]
jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up kubectl
        uses: azure/setup-kubectl@v4
      - name: Install kind
        run: |
          curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
          chmod +x ./kind
          sudo mv ./kind /usr/local/bin/kind
      - name: Create Kind cluster
        run: kind create cluster --wait 120s
      - name: Load Docker image into Kind
        run: |
          docker build -t app:test .
          kind load docker-image app:test
      - name: Deploy application
        run: kubectl apply -f k8s/
      - name: Wait for rollout
        run: kubectl rollout status deployment/app --timeout=120s
      - name: Run simple smoke test
        run: |
          kubectl run tester --image=curlimages/curl -it --restart=Never --rm -- \
            curl -f http://app-service || exit 1
```

---

## 자주 발생하는 문제 및 해결 방법

| 증상 | 가능한 원인 | 해결 방안 |
|---|---|---|
| Ingress 접속 시 404 또는 연결 실패 | IngressClass 미일치, 포트 포워딩 설정 안 됨 | `ingressClassName: nginx` 확인, `cluster-config.yaml`에 `extraPortMappings` 설정 |
| LoadBalancer 서비스의 EXTERNAL-IP가 `<pending>` | MetalLB 미설치 또는 IP 풀 구성 오류 | MetalLB 설치 및 IP 주소 풀이 Kind Docker 네트워크 범위에 있는지 확인 |
| `ImagePullBackOff` 에러 | 이미지가 레지스트리에 없거나 태그 오류 | `kind load docker-image`로 이미지 로드 또는 로컬 레지스트리 구성 확인 |
| CoreDNS Pod이 `CrashLoopBackOff` | CNI 구성 충돌 | 클러스터 재생성 또는 `disableDefaultCNI: true` 설정 시 CNI 플러그인(예: Calico) 명시적 설치 |
| 노드 상태가 `NotReady` | kube-proxy 또는 네트워크 플러그인 문제 | `kubectl -n kube-system logs ds/kube-proxy` 로그 확인 |
| 성능이 매우 느림 | 호스트 시스템 리소스(CPU/메모리) 부족 | Kind 노드 수 감소, 불필요한 워크로드 정리, 호스트 리소스 확보 |
| 호스트 파일 변경이 Pod에 반영되지 않음 | `extraMounts`와 `hostPath` 경로 불일치 | Kind 구성의 `containerPath`와 Pod 매니페스트의 `hostPath` 경로가 동일한지 재확인 |

기본적인 문제 해결을 위한 명령어:
```bash
kubectl get pods -A -o wide
kubectl describe pod <문제되는-pod-이름>
kubectl logs <pod-이름>
docker logs <kind-node-container-name>
```

---

## Kind와 Minikube 비교

로컬 쿠버네티스 환경을 위한 다른 인기 도구인 Minikube와의 간단한 비교입니다.

| 기준 | Kind | Minikube |
|---|---|---|
| **기반 기술** | Docker 컨테이너 | 가상머신(기본) 또는 Docker |
| **시작 속도 및 자원 사용** | 매우 빠르고 가벼움 | 상대적으로 느리고 무거움 |
| **멀티 노드 클러스터** | 구성 파일에 노드를 추가하면 간단히 생성 가능 | 가능하나 설정이 더 복잡함 |
| **외부 접근 (Ingress)** | 명시적인 `extraPortMappings` 설정 필요 | `minikube addons enable ingress`로 비교적 간단 |
| **LoadBalancer 서비스** | MetalLB 등의 추가 구성 필요 | 일부 드라이버에서 내장 지원 |
| **CI/CD 통합** | **탁월함** (경량, 헤드리스 동작) | 가능하지만 VM 오버헤드로 인해 느릴 수 있음 |

**선택 가이드**:
- **속도와 가벼움, CI/CD 통합, 멀티 노드 테스트가 우선** → **Kind**
- **단일 노드에서의 다양한 부가 기능(대시보드, 메트릭스) 및 간편한 설정이 우선** → **Minikube**

---

## 종합 예제: 웹 프론트엔드와 API 백엔드 스택 배포

다음 YAML 파일은 Nginx 웹 서버와 간단한 API 서비스, 그리고 이를 하나의 Ingress로 노출하는 완전한 예시입니다.

```yaml
# mini-stack.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
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
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
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
        args: ["-text={\"message\": \"Hello from API\"}"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 5678
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mini-stack-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: web.kind.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
  - host: api.kind.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
```

배포 및 테스트:
```bash
# Ingress Controller가 설치되어 있다고 가정
kubectl apply -f mini-stack.yaml
echo "127.0.0.1 web.kind.local api.kind.local" | sudo tee -a /etc/hosts
curl http://web.kind.local
curl http://api.kind.local
```

---

## 결론

Kind는 로컬 개발과 CI/CD 테스트를 위한 강력하고 현대적인 쿠버네티스 환경을 제공합니다. Docker 컨테이너 기반의 경량 아키텍처 덕분에 빠르게 클러스터를 생성하고 제거할 수 있으며, 구성 파일을 통해 멀티 노드, 포트 포워딩, 스토리지 마운트 등을 유연하게 설정할 수 있습니다.

핵심 장점을 정리하면 다음과 같습니다:
1.  **빠르고 가벼움**: VM 오버헤드 없이 초 단위로 클러스터 생성.
2.  **표준 쿠버네티스**: 실제 쿠버네티스 배포판을 사용하여 프로덕션 환경과의 호환성 보장.
3.  **재현성과 일관성**: 구성 파일을 버전 관리하여 동일한 환경을 반복적으로 생성 가능.
4.  **CI/CD 친화적**: GitHub Actions, GitLab CI 등에서 손쉽게 통합하여 E2E 테스트 파이프라인 구축.

Kind를 효과적으로 사용하려면 Ingress 컨트롤러 설치, MetalLB를 통한 로드 밸런서 실습, 그리고 로컬 레지스트리 연동과 같은 고급 구성을 익히는 것이 좋습니다. 이를 통해 개인 학습뿐만 아니라 팀 단위의 효율적인 개발과 테스트 프로세스를 구축할 수 있습니다. 다음 단계로는 Kind 클러스터 위에 Helm을 이용한 애플리케이션 배포, Prometheus와 Grafana를 통한 모니터링 구축, 또는 네트워크 정책(NetworkPolicy) 실습을 진행해 보시기 바랍니다.