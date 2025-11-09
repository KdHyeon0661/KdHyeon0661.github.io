---
layout: post
title: Kubernetes - kubectl 설치 및 기본 명령어
date: 2025-04-04 19:20:23 +0900
category: Kubernetes
---
# kubectl 설치 및 기본 명령어 사용법

## 들어가며

- **최신 저장소 방식**(apt-key 미사용)과 플랫폼별 설치
- **kubeconfig/컨텍스트/네임스페이스** 운영 습관
- **조회 출력 고급화**(wide/JSONPath/custom-columns/go-template)
- **편집/패치/롤아웃**과 선언형 관리 베스트 프랙티스
- **디버깅 루틴**(events/top/port-forward/cp/exec), **선택적 위험 명령 주의**
- **자동완성·프롬프트 보조(krew)**, **짧은 치트시트와 트러블슈팅 표**

---

## kubectl이란?
`kubectl`은 쿠버네티스 API 서버와 상호작용하는 **CLI**다. 다음 작업을 수행한다.

- 리소스 **생성/변경/삭제**(imperative & declarative)
- **상태 조회**(get/describe), **로그/셸 진입**(logs/exec)
- **네트워크 프록시**(port-forward), **파일 교환**(cp)
- **롤아웃 관리**(rollout), **스케일**(scale), **이미지 교체**(set image)

기본 구문:
```bash
kubectl [명령] [리소스] [이름] [옵션...]
# 예
kubectl get pods
kubectl describe service web
kubectl delete deployment nginx
```

---

## 설치

### macOS (Homebrew)
```bash
brew install kubectl
kubectl version --client
```

### Ubuntu / Debian (권장: 새 패키지 저장소 방식)
> `apt-key`는 더 이상 권장되지 않는다. 서명 키를 keyring에 저장해 사용하자.

```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubectl
kubectl version --client
```
> 필요에 따라 `v1.29`, `v1.31` 등 트랙을 선택.

### Windows (Chocolatey)
```powershell
choco install kubernetes-cli
kubectl version --client
```

### 바이너리 직접 설치(대체)
공식 릴리스를 내려받아 실행 권한 부여 후 PATH에 배치한다.

---

## kubeconfig/컨텍스트/네임스페이스

### 현재 컨텍스트/클러스터/사용자 확인
```bash
kubectl config view --minify
kubectl config get-contexts
kubectl config current-context
```

### 컨텍스트 전환 & 네임스페이스 기본값 설정
```bash
kubectl config use-context my-ctx
kubectl config set-context --current --namespace=default
```

> **팁**: 컨텍스트/네임스페이스를 프롬프트에 노출(zsh theme or kubectl prompt plugin)하면 오작동을 줄인다.

---

## 리소스 타입(요약)
| 타입 | 설명 |
|---|---|
| `pod` | 컨테이너 실행 단위 |
| `deployment` | 선언형 배포(ReplicaSet 관리) |
| `service` | L4 접근 지점(ClusterIP/NodePort/LB) |
| `ingress` | L7 라우팅(컨트롤러 필요) |
| `configmap`, `secret` | 구성/비밀 데이터 |
| `job`, `cronjob` | 배치/스케줄 실행 |
| `node`, `namespace` | 인프라/논리 구획 |

---

## 조회(get) — 출력 고급화

### 기본/와이드/YAML/JSON
```bash
kubectl get pods
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json
```

### label/field selector
```bash
kubectl get pods -l app=web
kubectl get pods --field-selector status.phase=Running
```

### custom-columns & JSONPath & go-template
```bash
# custom-columns
kubectl get pod -o=custom-columns=NAME:.metadata.name,IP:.status.podIP,NODE:.spec.nodeName

# JSONPath: 이름과 이미지만
kubectl get pod -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'

# go-template
kubectl get svc web -o go-template='{{.spec.clusterIP}}{{"\n"}}'
```

---

## 상세(Describe) & 이벤트

```bash
kubectl describe pod <POD>
kubectl describe deployment <DEPLOY>
kubectl get events --sort-by=.lastTimestamp
```

> 이벤트는 문제(스케줄 실패, 이미지 풀 실패, Liveness 실패 등)를 가장 빨리 드러낸다.

---

## 생성 — Imperative vs Declarative

### 1) Imperative(즉석)
```bash
kubectl create deployment web --image=nginx:1.27-alpine
kubectl expose deployment web --port=80 --type=ClusterIP
```

### 2) Declarative(YAML)
```yaml
# web-deploy.yaml
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
```

```bash
kubectl apply -f web-deploy.yaml
```

> **권장**: 운영은 선언형(YAML/git 관리). `kubectl apply`는 3way-merge로 안전하다.

---

## 수정 — edit / patch / set / scale

### 즉석 편집
```bash
kubectl edit deployment web
```

### 전략 패치(merge/json)
```bash
# 이미지 태그 교체(전략 병합)
kubectl patch deploy web -p '{"spec":{"template":{"spec":{"containers":[{"name":"web","image":"nginx:1.27.2"}]}}}}'

# json-patch 예
kubectl patch deploy web --type='json' \
  -p='[{"op":"replace","path":"/spec/replicas","value":3}]'
```

### 선언형 교체(권장)
```bash
kubectl apply -f web-deploy.yaml
```

### 스케일/이미지 교체(명령식)
```bash
kubectl scale deploy/web --replicas=4
kubectl set image deploy/web web=nginx:1.27.3
```

---

## 롤아웃 — 상태/히스토리/되돌리기

```bash
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web            # 직전 버전으로
kubectl rollout undo deployment/web --to-revision=3
```

> **팁**: 컨테이너 `args`/`env`/`image` 변경 시 새 ReplicaSet이 생성된다. 프로브/readiness로 안정화를 보장.

---

## 디버깅 — logs / exec / port-forward / cp / top

### 로그
```bash
kubectl logs deploy/web
kubectl logs pod/<POD> -c <CONTAINER> --since=10m
kubectl logs -f pod/<POD>    # 스트리밍
```

### 셸 진입
```bash
kubectl exec -it pod/<POD> -- /bin/sh
# 혹은 컨테이너 지정
kubectl exec -it pod/<POD> -c web -- bash
```

### 포트 포워드(로컬→Pod/Service)
```bash
kubectl port-forward svc/web 8080:80
# 브라우저: http://localhost:8080
```

### 파일 복사
```bash
kubectl cp ./index.html default/web-xxxxx:/usr/share/nginx/html/index.html
```

### 리소스 사용량(top)
```bash
kubectl top nodes
kubectl top pods -A
```

---

## 네임스페이스/라벨/어노테이션 관리

```bash
kubectl get ns
kubectl create ns staging
kubectl label ns staging env=staging
kubectl annotate deploy web owner=platform-team
```

라벨로 조회/삭제:
```bash
kubectl get pods -l app=web
kubectl delete pods -l app=web
```

---

## 노드운영(주의) — cordon/uncordon/drain
> **운영 주의**: 드레인은 중단을 유발할 수 있다. PDB/중단허용설계 필수.

```bash
kubectl cordon <NODE>     # 스케줄 금지
kubectl drain <NODE> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <NODE>
```

---

## 컨텍스트별 빠른 실습(요약)

### 1) Nginx 배포 & NodePort 노출
```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=NodePort --port=80
kubectl get svc nginx -o wide
```

### 2) Health Probe 추가(선언형)
```yaml
# patch-probe.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: nginx }
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        readinessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 5
          periodSeconds: 5
```
```bash
kubectl apply -f patch-probe.yaml
kubectl rollout status deploy/nginx
```

---

## 출력/선택 고급 패턴 스니펫

### 마지막 재시작 시각·이유
```bash
kubectl get pods -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[*].lastState.terminated.finishedAt}{"\t"}{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}{end}'
```

### Node/Pod 매핑 요약
```bash
kubectl get pod -o=custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP --no-headers
```

---

## 자동완성 & 별칭 & krew(플러그인)

### 자동완성
```bash
# bash
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# zsh
source <(kubectl completion zsh)
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
```

### 별칭(예)
```bash
alias k=kubectl
complete -F __start_kubectl k  # bash
```

### krew 설치(플러그인 매니저)
```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/arm64/arm64/')" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew-${OS}_${ARCH}.tar.gz" &&
  tar zxvf "krew-${OS}_${ARCH}.tar.gz" &&
  ./"krew-${OS}_${ARCH}" install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
kubectl krew list
```

유용한 플러그인:
```bash
kubectl krew install ctx ns neat df-pv view-secret
```

---

## 보너스 — 안전한 작업 루틴(체크리스트)
1) **컨텍스트/네임스페이스** 확인: `kubectl config current-context`, `-n ...`
2) **대상 라벨** 재확인: `-l` 사용 시 미스매치 주의
3) **적용 전 차이 확인**: `kubectl diff -f ...`
4) **롤아웃 모니터링**: `kubectl rollout status ...`
5) **실패 시 복구**: `kubectl rollout undo ...`

---

## Troubleshooting 확장표

| 증상 | 1차 확인 | 원인 후보 | 해결 가이드 |
|---|---|---|---|
| `Pending` 지속 | `describe pod` 이벤트 | 자원/스케줄 정책/노드 taint | 요청/제약 재설계, 노드풀 용량 확보, tolerations |
| `ImagePullBackOff` | `describe pod` | 레지스트리/비밀/태그 오타 | `imagePullSecrets`/네트워크/태그 점검 |
| `CrashLoopBackOff` | `logs -p` | 시작 스크립트/의존 실패 | entrypoint 수정, readiness 지연 |
| `Readiness` 미통과 | `describe pod` | 포트/헬스 경로 오타 | probe 경로/포트 교정 |
| `Service` 접속불가 | `get ep` | selector-라벨 불일치 | 라벨/셀렉터 정합성 수정 |
| `NodePort` 타임아웃 | 노드IP 방화벽 | 방화벽/노드 선택 | 보안그룹/방화벽 허용, 백엔드 분포 |

---

## 치트시트 — 자주 쓰는 명령 한 화면
```bash
# 컨텍스트/네임스페이스
kubectl config get-contexts
kubectl config use-context my-ctx
kubectl config set-context --current --namespace=default

# 조회
kubectl get all
kubectl get pods -o wide
kubectl get pods -A --field-selector status.phase=Running
kubectl describe pod <POD>

# 생성/적용
kubectl create deploy web --image=nginx:1.27-alpine
kubectl expose deploy web --port=80 --type=ClusterIP
kubectl apply -f k8s/

# 수정/배포
kubectl set image deploy/web web=nginx:1.27.3
kubectl rollout status deploy/web
kubectl rollout undo deploy/web

# 디버깅
kubectl logs -f deploy/web
kubectl exec -it deploy/web -- /bin/sh
kubectl port-forward svc/web 8080:80
kubectl get events --sort-by=.lastTimestamp

# 리소스
kubectl top pods -A
kubectl top nodes

# 삭제
kubectl delete -f k8s/
```

---

## 결론
`kubectl`은 K8s 운영의 **핵심 인터페이스**다.  
설치 후 **컨텍스트/네임스페이스 안전장치**를 갖추고, **선언형 관리(apply/diff/rollout)** 를 습관화하면 장애 확률이 크게 줄어든다.  
**logs/describe/events/top/port-forward**를 조합한 **디버깅 루틴**과 **krew/자동완성**으로 생산성을 끌어올리자.