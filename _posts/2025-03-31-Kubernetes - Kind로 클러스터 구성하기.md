---
layout: post
title: Kubernetes - Kind로 클러스터 구성하기
date: 2025-03-31 22:20:23 +0900
category: Kubernetes
---
# Kind(Kubernetes in Docker)로 클러스터 구성하기

## 들어가며

- **커스텀 클러스터 설정**: `extraPortMappings`, `extraMounts`, `kubeadmConfigPatches`, `nodeImage` 선택
- **Ingress(ingress-nginx) 외부 접근**과 **MetalLB + LoadBalancer** 실습
- **로컬/사설 레지스트리 연동**과 이미지 반영 루프 최적화
- **스토리지/볼륨**(호스트 ↔ Pod)과 **네트워크 디버깅**
- **CI(깃허브 액션)**에서 Kind로 E2E 테스트
- **자주 겪는 오류 해결**과 실전 팁

---

## 1. Kind란 무엇인가?
- Docker 컨테이너를 **가상 노드**처럼 사용해 **쿠버네티스 클러스터**를 구성하는 공식 도구.
- 추가 하이퍼바이저 없이 **가볍고 빠르게** 클러스터를 띄움.
- 로컬 개발/테스트, CI 파이프라인에서 **E2E 검증**에 최적.
- `kind` 명령 한 줄로 **생성/삭제** 가능.

---

## 2. 사전 요구사항
- **Docker** 설치 및 실행(데몬 구동 상태).
- `kubectl` 설치 권장.
- Go 환경은 필요 없음(바이너리 사용).

---

## 3. 설치

### macOS(Homebrew)
```bash
brew install kind
```

### Linux (바이너리)
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Windows(Chocolatey)
```powershell
choco install kind
```

설치 확인:
```bash
kind version
```

---

## 4. 가장 빠른 시작(기본 설정)

```bash
kind create cluster
kubectl get nodes
```

- 기본 이름: `kind`
- 구성: 단일 control-plane 노드
- Docker 컨테이너 목록에서 노드 확인 가능:
{% raw %}
```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
```
{% endraw %}

---

## 5. 커스텀 구성 — 멀티 노드, 포트 매핑, 볼륨 마운트

아래 예제는 **1 control-plane + 2 worker**, 호스트의 **80/443 → control-plane의 80/443** 포워딩(ingress 접근용), 그리고 호스트 디렉터리를 노드에 **마운트**한다.

```yaml
# cluster-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: my-cluster
nodes:
  - role: control-plane
    image: kindest/node:v1.30.0
    extraPortMappings:
      - containerPort: 80   # ingress 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443  # ingress 443
        hostPort: 443
        protocol: TCP
    extraMounts:
      - hostPath: /home/you/workspace     # 호스트 경로
        containerPath: /mnt/workspace     # 노드 내부 경로
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
        "feature-gates": "MixedProtocolLBService=true"
networking:
  disableDefaultCNI: false
  kubeProxyMode: "iptables"
```

생성:
```bash
kind create cluster --config cluster-config.yaml
kubectl cluster-info --context kind-my-cluster
kubectl get nodes -o wide
```

---

## 6. Ingress(ingress-nginx) 설치와 외부 접속

### 6.1 Ingress 컨트롤러 설치
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller
```

> 위 매니페스트는 Kind 환경에서의 DaemonSet/NodePort/HostPort 조합을 고려한 배포이며, 앞서 `extraPortMappings(80/443)` 덕에 **호스트 브라우저에서 바로 접근** 가능.

### 6.2 테스트 서비스/인그레스
```yaml
# echo-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: echo, labels: { app: echo } }
spec:
  replicas: 2
  selector: { matchLabels: { app: echo } }
  template:
    metadata: { labels: { app: echo } }
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo
        args: ["-text=hello from kind"]
        ports: [{ containerPort: 5678 }]
---
apiVersion: v1
kind: Service
metadata: { name: echo-svc }
spec:
  selector: { app: echo }
  ports: [{ port: 80, targetPort: 5678 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: echo-ing }
spec:
  ingressClassName: nginx
  rules:
  - host: echo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: echo-svc, port: { number: 80 } } }
```

적용/접속:
```bash
kubectl apply -f echo-app.yaml
# /etc/hosts에 127.0.0.1 echo.local 추가(Kind의 80/443이 호스트로 포워딩됨)
echo "127.0.0.1 echo.local" | sudo tee -a /etc/hosts
curl http://echo.local/
```

---

## 7. MetalLB로 LoadBalancer 타입 실습(옵션)

Kind는 베어메탈과 유사해 **외부 LB가 없다**. L4 LB 실습을 위해 **MetalLB**를 배포한다.

### 7.1 설치 및 IP 풀 구성
```bash
# 설치
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
kubectl -n metallb-system rollout status deploy/controller

# IPAddressPool + L2Advertisement (Kind Docker 네트워크 대역 고려)
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: kind-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.18.255.200-172.18.255.250  # docker network 대역 확인 후 알맞게 조정
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

### 7.2 LoadBalancer 서비스 테스트
```yaml
apiVersion: v1
kind: Service
metadata: { name: echo-lb }
spec:
  type: LoadBalancer
  selector: { app: echo }
  ports:
    - port: 80
      targetPort: 5678
```

적용 후 **EXTERNAL-IP** 확인:
```bash
kubectl apply -f echo-lb.yaml
kubectl get svc echo-lb
curl http://<EXTERNAL-IP>/
```

> Docker 네트워크 대역(`docker network inspect kind`)을 확인해 IP 풀을 맞춰야 한다.

---

## 8. 로컬/사설 레지스트리 연동과 이미지 반영

### 8.1 로컬 이미지 로딩(간단)
```bash
docker build -t my-app:dev .
kind load docker-image my-app:dev --name my-cluster
# Deployment에서 image: my-app:dev + imagePullPolicy: IfNotPresent
```

### 8.2 로컬 레지스트리 컨테이너와 연결(권장)
킨드 문서의 패턴을 따라 **도커 레지스트리**를 띄우고, 킨드 노드에 **미러로 인식**시키면 팀/CI와 연계가 쉬워진다.

```bash
# 1) 레지스트리 컨테이너
docker run -d --restart=always -p "5001:5000" --name registry registry:2

# 2) registry 미러를 사용하는 kind 클러스터
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

# 3) 클러스터 생성
kind create cluster --config kind-with-registry.yaml

# 4) registry 컨테이너를 kind 네트워크에 붙임
docker network connect kind registry || true
```

이미지 푸시/사용:
```bash
# 호스트에서 빌드 → 로컬 레지스트리에 푸시
docker build -t localhost:5001/my-app:dev .
docker push localhost:5001/my-app:dev

# K8s 매니페스트에서는 image: localhost:5001/my-app:dev 로 참조
```

---

## 9. 볼륨/데이터 — 호스트↔Pod 개발 루프

`extraMounts`로 **호스트 디렉터리**를 노드에 마운트해두면, 그 경로를 **hostPath**로 Pod에 연결해 빠른 개발 루프를 만든다.

```yaml
# cluster-config.yaml의 extraMounts로 /mnt/workspace 마운트했다고 가정
apiVersion: apps/v1
kind: Deployment
metadata: { name: web-dev }
spec:
  replicas: 1
  selector: { matchLabels: { app: web-dev } }
  template:
    metadata: { labels: { app: web-dev } }
    spec:
      containers:
      - name: web
        image: nginx:1.27-alpine
        volumeMounts:
        - name: code
          mountPath: /usr/share/nginx/html
      volumes:
      - name: code
        hostPath:
          path: /mnt/workspace/site    # 노드 내부 경로(= 호스트가 마운트한 결과)
          type: Directory
```

호스트에서 `workspace/site` 파일을 수정하면, Pod에서 **즉시 반영**된다(정적 파일 기준).

---

## 10. 네트워크/클러스터 디버깅

### 10.1 내부 통신 점검
```bash
kubectl run -it --rm netshoot --image=nicolaka/netshoot -- /bin/bash
# 컨테이너 내부
curl -v http://echo-svc
dig echo-svc.default.svc.cluster.local
```

### 10.2 이벤트/리소스
```bash
kubectl get events -A --sort-by=.lastTimestamp
kubectl top pod -A
kubectl get endpointslices -A
```

---

## 11. CI(깃허브 액션)에서 Kind로 E2E 테스트

```yaml
# .github/workflows/e2e.yml
name: e2e-kind
on: [push, pull_request]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4
        with:
          version: 'v1.30.0'

      - name: Install kind
        run: |
          curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
          chmod +x ./kind
          sudo mv ./kind /usr/local/bin/kind

      - name: Create cluster
        run: kind create cluster --wait 120s

      - name: Load app image
        run: |
          docker build -t app:test .
          kind load docker-image app:test

      - name: Deploy manifests
        run: kubectl apply -f k8s/

      - name: Wait for rollout
        run: kubectl -n default rollout status deploy/app --timeout=120s

      - name: Run tests
        run: |
          kubectl run tester --image=radial/busyboxplus:curl -it --restart=Never -- \
            curl -sS http://app-svc | grep "OK"
```

> 로컬 빌드 이미지를 **kind load**로 싱크하고, 간단한 인그레스/서비스 검증을 수행한다.

---

## 12. 자주 겪는 문제 해결(확장판)

| 증상/오류 | 원인 후보 | 해결 |
|---|---|---|
| Ingress 접근 404/연결 불가 | IngressClass 불일치/포트 미포워딩 | `ingressClassName: nginx` 확인, `extraPortMappings(80/443)` 적용 |
| LoadBalancer EXTERNAL-IP 미할당 | MetalLB 미설치/IP 풀 불일치 | MetalLB 설치, `IPAddressPool` 범위를 Docker 네트워크에 맞춤 |
| `ImagePullBackOff` | 로컬 태그/풀 정책 문제 | `kind load docker-image` 또는 레지스트리 사용, `IfNotPresent` |
| DNS 실패(CoreDNS CrashLoop) | CNI 꼬임/네트워크 대역 충돌 | 클러스터 재생성, `disableDefaultCNI` 사용 시 CNI 명시 설치 |
| 노드 NotReady | kube-proxy/CNI 문제 | `kubectl -n kube-system logs ds/kube-proxy`, CNI 재설치 |
| 퍼포먼스 저하 | 호스트 리소스 부족 | Kind 노드 수/리소스 조정, 불필요한 워크로드 정리 |
| 볼륨 업데이트 미반영 | 마운트 경로 착오 | `extraMounts`의 **노드 경로** ↔ `hostPath` 일치 재확인 |

기본 점검:
```bash
kubectl get pods -A -o wide
kubectl describe pod <pod>
kubectl get svc,ep -A
docker ps
docker logs <kind-node-container>
```

---

## 13. 간단 수용량 계산(학습용)
서비스 목표 QPS, 평균 처리시간 \(t\), 파드당 안정 처리량 \(\text{cap}_{pod}\) 이면:
$$
\text{필요 레플리카} \approx
\left\lceil
\frac{\text{QPS}\cdot t}{\text{cap}_{pod}}
\right\rceil \cdot \text{버퍼}
$$
Kind는 **로컬**이므로 과도한 부하는 호스트 스펙에 좌우된다. E2E 검증은 **기능/호환성 중심**으로 가볍게.

---

## 14. 한 번에 실행해 보는 미니 스택(웹+API+Ingress)

```yaml
# mini-stack.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: web, labels: { app: web } }
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
apiVersion: apps/v1
kind: Deployment
metadata: { name: api, labels: { app: api } }
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
---
apiVersion: v1
kind: Service
metadata: { name: api-svc }
spec:
  selector: { app: api }
  ports: [{ port: 80, targetPort: 5678 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: mini }
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
```

실행 순서:
```bash
# 클러스터는 extraPortMappings(80/443) 포함 구성으로 생성되어 있다고 가정
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl apply -f mini-stack.yaml
echo "127.0.0.1 web.local api.local" | sudo tee -a /etc/hosts
curl http://web.local/
curl http://api.local/
```

---

## 15. Minikube와 비교 — 어떤 상황에 어떤 도구?
| 항목 | Kind | Minikube |
|---|---|---|
| 기반 | Docker 컨테이너 | VM 또는 Docker |
| 속도/가벼움 | 매우 빠름/가벼움 | 상대적으로 무거움 |
| 멀티 노드 | 쉬움(컨테이너 추가) | 제한적(가능하나 구성 다름) |
| Ingress 외부 접근 | HostPort/포워딩 구성 필요 | 애드온으로 간결 |
| LoadBalancer 실습 | MetalLB 설치 필요 | 애드온/내장(환경 따라 다름) |
| CI 적합성 | **최고** (헤드리스) | 가능하나 속도/복잡도 ↑ |

---

## 16. 정리 및 다음 단계
- Kind는 **가볍고 빠른 개발/테스트/CI용 쿠버네티스**의 표준급 도구다.
- 핵심 포인트 요약:
  1) **커스텀 설정**으로 포트/마운트/노드 수를 환경에 맞게 조정  
  2) **ingress-nginx + extraPortMappings**로 브라우저 접근 간편화  
  3) **MetalLB**로 LoadBalancer 타입 테스트  
  4) **로컬 레지스트리/이미지 로딩**으로 개발 루프 단축  
  5) **CI 파이프라인**에 Kind를 넣어 E2E 신뢰성 강화
- 다음 학습:
  - **Helm/Kustomize** 템플릿화
  - **OpenTelemetry + Prom/Grafana**로 관측 강화
  - **네트워크 폴리시/보안 컨텍스트**로 보안 기초 체화

부록: 자주 쓰는 명령 모음
```bash
kind create cluster --config cluster-config.yaml
kind get clusters
kind export kubeconfig --name my-cluster
kind load docker-image my-app:dev --name my-cluster
kind delete cluster --name my-cluster

kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl describe ingress mini
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller
```