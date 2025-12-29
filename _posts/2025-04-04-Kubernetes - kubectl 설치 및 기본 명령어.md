---
layout: post
title: Kubernetes - kubectl 설치 및 기본 명령어
date: 2025-04-04 19:20:23 +0900
category: Kubernetes
---
# kubectl 설치 및 고급 사용법 완벽 가이드

## 소개: kubectl과 현대적인 쿠버네티스 운영

kubectl은 쿠버네티스 클러스터와 상호작용하기 위한 공식 명령줄 도구입니다. 이 가이드는 kubectl의 설치부터 고급 사용법까지 체계적으로 다루며, 특히 최신 패키지 관리 방식, 컨텍스트 관리, 선언적 운영 방법론, 고급 디버깅 기법에 중점을 둡니다.

kubectl은 다음과 같은 핵심 기능을 제공합니다:
- 쿠버네티스 리소스의 생성, 조회, 수정, 삭제
- 애플리케이션 상태 모니터링 및 로그 확인
- 네트워크 디버깅을 위한 포트 포워딩
- 파일 시스템 접근 및 컨테이너 내부 진입
- 배포 롤아웃 관리 및 롤백

## kubectl 설치: 플랫폼별 최신 방법

### macOS (Homebrew)
```bash
# Homebrew를 통한 설치
brew install kubectl

# 설치 확인
kubectl version --client
```

### Ubuntu / Debian (공식 저장소 방식)
최신 패키지 관리 방식을 따라 apt-key 대신 서명 키링을 사용하는 방법:

```bash
# 필수 패키지 설치
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg

# 서명 키 디렉토리 생성 및 키 다운로드
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# APT 저장소 추가
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list

# 패키지 업데이트 및 kubectl 설치
sudo apt update
sudo apt install -y kubectl

# 설치 확인
kubectl version --client
```

**버전 선택**: 필요에 따라 `v1.30`을 다른 버전(예: `v1.29`, `v1.31`)으로 변경할 수 있습니다.

### Windows (Chocolatey)
```powershell
# Chocolatey 패키지 관리자 설치 후
choco install kubernetes-cli

# 설치 확인
kubectl version --client
```

### 직접 바이너리 설치 (대체 방법)
공식 GitHub 릴리스 페이지에서 플랫폼에 맞는 바이너리를 다운로드하여 사용할 수 있습니다.

## kubeconfig, 컨텍스트, 네임스페이스 관리

### 현재 구성 확인
```bash
# 현재 컨텍스트 정보 확인
kubectl config current-context

# 모든 컨텍스트 목록 조회
kubectl config get-contexts

# 현재 컨텍스트의 상세 구성 보기
kubectl config view --minify
```

### 컨텍스트 전환 및 네임스페이스 설정
```bash
# 특정 컨텍스트로 전환
kubectl config use-context my-cluster

# 현재 컨텍스트의 기본 네임스페이스 설정
kubectl config set-context --current --namespace=production
```

**운영 팁**: 셸 프롬프트에 현재 컨텍스트와 네임스페이스를 표시하면 실수로 잘못된 환경에서 명령을 실행하는 것을 방지할 수 있습니다. zsh 테마나 kubectl 프롬프트 플러그인을 활용하세요.

## 주요 리소스 타입 이해

쿠버네티스에서 다루는 주요 리소스 타입들:

| 리소스 타입 | 설명 | 일반적인 사용 사례 |
|---|---|---|
| `pod` | 하나 이상의 컨테이너 그룹 | 애플리케이션 인스턴스 실행 |
| `deployment` | ReplicaSet을 관리하는 선언적 배포 오브젝트 | 무중단 배포 및 롤백 관리 |
| `service` | 파드 집합에 대한 네트워크 엔드포인트 | 내부/외부 트래픽 라우팅 |
| `ingress` | L7 HTTP/HTTPS 라우팅 규칙 | 도메인 기반 라우팅, TLS 종료 |
| `configmap` | 구성 데이터 저장소 | 환경 변수, 설정 파일 관리 |
| `secret` | 민감한 정보 저장소 | 패스워드, 토큰, 인증서 관리 |
| `namespace` | 클러스터 내 논리적 분할 | 환경 분리(개발/스테이징/프로덕션) |

## 고급 조회 및 출력 형식

### 기본 출력 형식
```bash
# 기본 테이블 형식
kubectl get pods

# 추가 정보 포함 (노드, IP 등)
kubectl get pods -o wide

# YAML 형식 출력
kubectl get pods -o yaml

# JSON 형식 출력
kubectl get pods -o json
```

### 레이블 및 필드 셀렉터
```bash
# 특정 레이블을 가진 파드 조회
kubectl get pods -l app=web

# 특정 상태의 파드만 조회
kubectl get pods --field-selector status.phase=Running
```

### 사용자 정의 출력 형식
사용자 정의 컬럼, JSONPath, Go 템플릿을 활용한 고급 출력:

{% raw %}
```bash
# 사용자 정의 컬럼 출력
kubectl get pods -o=custom-columns=NAME:.metadata.name,IP:.status.podIP,NODE:.spec.nodeName

# JSONPath를 활용한 특정 정보 추출
kubectl get pods -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'

# Go 템플릿을 사용한 데이터 추출
kubectl get service web -o go-template='{{.spec.clusterIP}}{{"\n"}}'
```
{% endraw %}

## 상세 정보 확인 및 이벤트 모니터링

### 리소스 상세 정보 확인
```bash
# 파드 상세 정보
kubectl describe pod <파드-이름>

# 디플로이먼트 상세 정보
kubectl describe deployment <디플로이먼트-이름>

# 서비스 상세 정보
kubectl describe service <서비스-이름>
```

### 클러스터 이벤트 모니터링
```bash
# 시간순 정렬된 이벤트 목록
kubectl get events --sort-by=.lastTimestamp

# 특정 네임스페이스의 이벤트만 필터링
kubectl get events -n production --sort-by=.lastTimestamp
```

**중요성**: 이벤트는 스케줄링 실패, 이미지 풀 실패, 헬스 체크 실패 등의 문제를 가장 빠르게 발견할 수 있는 수단입니다.

## 리소스 생성: 명령형 vs 선언형 접근법

### 명령형(Imperative) 접근
즉석에서 리소스를 생성하는 방법:

```bash
# 디플로이먼트 생성
kubectl create deployment web --image=nginx:1.27-alpine --replicas=2

# 서비스 노출
kubectl expose deployment web --port=80 --type=ClusterIP
```

### 선언형(Declarative) 접근
YAML 파일을 통해 리소스를 정의하고 관리하는 방법:

`web-deployment.yaml` 파일:
```yaml
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
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
```

적용:
```bash
kubectl apply -f web-deployment.yaml
```

**모범 사례**: 운영 환경에서는 선언적 접근 방식을 권장합니다. YAML 파일을 Git으로 버전 관리하고, `kubectl apply` 명령은 3-way merge를 통해 안전하게 변경 사항을 적용합니다.

## 리소스 수정 및 업데이트 전략

### 대화형 편집
```bash
# 대화형 편집기로 리소스 수정
kubectl edit deployment web
```

### 패치(Patch)를 통한 수정
```bash
# 전략적 병합 패치 (Strategic Merge Patch)
kubectl patch deployment web \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"web","image":"nginx:1.27.2"}]}}}}'

# JSON 패치 (JSON Patch)
kubectl patch deployment web --type='json' \
  -p='[{"op":"replace","path":"/spec/replicas","value":3}]'
```

### 선언적 업데이트 (권장)
```bash
# YAML 파일 수정 후 적용
kubectl apply -f web-deployment.yaml
```

### 명령형 업데이트 도구
```bash
# 레플리카 수 조정
kubectl scale deployment/web --replicas=4

# 컨테이너 이미지 업데이트
kubectl set image deployment/web web=nginx:1.27.3

# 환경 변수 설정
kubectl set env deployment/web LOG_LEVEL=debug
```

## 배포 롤아웃 관리

### 롤아웃 상태 모니터링
```bash
# 배포 진행 상태 확인
kubectl rollout status deployment/web

# 배포 히스토리 확인
kubectl rollout history deployment/web

# 특정 리비전 상세 정보
kubectl rollout history deployment/web --revision=2
```

### 롤백 실행
```bash
# 최신 버전으로 롤백
kubectl rollout undo deployment/web

# 특정 리비전으로 롤백
kubectl rollout undo deployment/web --to-revision=3
```

**중요한 점**: 컨테이너 이미지, 환경 변수, 명령줄 인수 등의 변경은 새로운 ReplicaSet을 생성합니다. 무중단 배포를 위해서는 적절한 readiness probe와 liveness probe 설정이 필수적입니다.

## 디버깅 및 문제 해결 도구

### 로그 확인
```bash
# 디플로이먼트의 모든 파드 로그 확인
kubectl logs deployment/web

# 특정 파드의 로그 확인
kubectl logs pod/<파드-이름>

# 특정 컨테이너 로그 확인
kubectl logs pod/<파드-이름> -c <컨테이너-이름>

# 최근 10분간의 로그 확인
kubectl logs pod/<파드-이름> --since=10m

# 실시간 로그 스트리밍
kubectl logs -f pod/<파드-이름>
```

### 컨테이너 내부 접근
```bash
# 파드 내부 셸 접속
kubectl exec -it pod/<파드-이름> -- /bin/sh

# 특정 컨테이너에 접속
kubectl exec -it pod/<파드-이름> -c web -- bash

# 단일 명령 실행
kubectl exec pod/<파드-이름> -- ls -la /app
```

### 포트 포워딩 (로컬에서 서비스 접근)
```bash
# 서비스 포트 포워딩
kubectl port-forward service/web 8080:80

# 파드 포트 포워딩
kubectl port-forward pod/web-pod 8080:80

# 디플로이먼트 포트 포워딩
kubectl port-forward deployment/web 8080:80
```
접속: http://localhost:8080

### 파일 복사
```bash
# 로컬에서 컨테이너로 파일 복사
kubectl cp ./config.json default/web-pod:/app/config.json

# 컨테이너에서 로컬로 파일 복사
kubectl cp default/web-pod:/app/logs/error.log ./error.log
```

### 리소스 사용량 확인
```bash
# 노드별 리소스 사용량
kubectl top nodes

# 모든 네임스페이스의 파드 리소스 사용량
kubectl top pods -A

# 특정 네임스페이스의 파드 리소스 사용량
kubectl top pods -n production
```

## 네임스페이스, 레이블, 어노테이션 관리

### 네임스페이스 작업
```bash
# 네임스페이스 목록
kubectl get namespaces

# 새 네임스페이스 생성
kubectl create namespace staging

# 네임스페이스에 레이블 추가
kubectl label namespace staging environment=staging

# 네임스페이스 삭제
kubectl delete namespace staging
```

### 레이블 및 어노테이션 관리
```bash
# 리소스에 레이블 추가
kubectl label deployment/web tier=frontend

# 레이블로 리소스 조회
kubectl get pods -l app=web

# 레이블로 리소스 삭제
kubectl delete pods -l app=web

# 어노테이션 추가
kubectl annotate deployment/web last-modified="$(date -Iseconds)"
```

## 노드 유지보수 작업

### 노드 격리 및 드레이닝
```bash
# 노드 스케줄링 비활성화 (신규 파드 배치 방지)
kubectl cordon <노드-이름>

# 노드에서 실행 중인 파드 제거 (DaemonSet 제외)
kubectl drain <노드-이름> --ignore-daemonsets --delete-emptydir-data

# 노드 스케줄링 재활성화
kubectl uncordon <노드-이름>
```

**운영 주의사항**: `drain` 명령은 실행 중인 워크로드에 중단을 일으킬 수 있습니다. Pod Disruption Budget(PDB)을 설정하고 중단 허용 아키텍처를 설계하는 것이 중요합니다.

## 실습 예제: Nginx 웹 서버 배포

### 빠른 배포 및 노출
```bash
# 디플로이먼트 생성
kubectl create deployment nginx --image=nginx:1.27-alpine

# NodePort 서비스로 노출
kubectl expose deployment nginx --type=NodePort --port=80

# 서비스 정보 확인
kubectl get service nginx -o wide
```

### 헬스 체크 추가 (선언적 방법)
`nginx-with-probes.yaml` 파일:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
```

적용 및 확인:
```bash
kubectl apply -f nginx-with-probes.yaml
kubectl rollout status deployment/nginx
```

## 고급 출력 및 선택 패턴 예시

### 최근 재시작 정보 추출
{% raw %}
```bash
kubectl get pods -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[*].lastState.terminated.finishedAt}{"\t"}{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}{end}'
```
{% endraw %}

### 노드-파드 매핑 요약
```bash
kubectl get pods -o=custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP,STATUS:.status.phase --no-headers | head -20
```

## 생산성 향상을 위한 도구 및 설정

### 자동 완성 설정
```bash
# Bash 자동 완성
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# Zsh 자동 완성
source <(kubectl completion zsh)
echo 'source <(kubectl completion zsh)' >> ~/.zshrc

# Fish 자동 완성
kubectl completion fish | source
```

### 별칭 설정
```bash
# kubectl을 k로 단축
alias k=kubectl

# Bash에서 자동 완성 연결
complete -F __start_kubectl k

# Zsh에서 자동 완성 연결 (별칭 확장 필요 시)
compdef k=kubectl
```

### krew 플러그인 관리자 설치
krew는 kubectl 플러그인을 관리하기 위한 패키지 관리자입니다:

```bash
# krew 설치
(
  set -x
  cd "$(mktemp -d)"
  OS="$(uname | tr '[:upper:]' '[:lower:]')"
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/arm64/arm64/')"
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew-${OS}_${ARCH}.tar.gz"
  tar zxvf "krew-${OS}_${ARCH}.tar.gz"
  ./"krew-${OS}_${ARCH}" install krew
)

# PATH에 krew 추가
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# 설치된 플러그인 확인
kubectl krew list
```

### 유용한 krew 플러그인
```bash
# 컨텍스트 및 네임스페이스 전환 도구
kubectl krew install ctx
kubectl krew install ns

# YAML 정리 및 포맷팅
kubectl krew install neat

# 리소스 가시성 향상
kubectl krew install df-pv
kubectl krew install view-secret
kubectl krew install view-serviceaccount-kubeconfig

# 리소스 상태 요약
kubectl krew install resource-capacity
```

## 안전한 작업 절차 체크리스트

운영 환경에서 kubectl을 사용할 때 다음 체크리스트를 따르면 실수를 방지할 수 있습니다:

1. **작업 환경 확인**: 현재 컨텍스트와 네임스페이스를 명시적으로 확인하세요.
   ```bash
   kubectl config current-context
   kubectl config view --minify --output 'jsonpath={..namespace}'
   ```

2. **대상 리소스 재확인**: 레이블 셀렉터를 사용할 때는 의도한 리소스만 선택되는지 확인하세요.
   ```bash
   kubectl get pods -l app=web --show-labels
   ```

3. **변경 사항 미리보기**: 적용 전 변경 사항을 검토하세요.
   ```bash
   kubectl diff -f deployment.yaml
   ```

4. **점진적 롤아웃 모니터링**: 배포 중 상태를 지속적으로 관찰하세요.
   ```bash
   kubectl rollout status deployment/web --timeout=5m
   ```

5. **롤백 준비**: 문제 발생 시 신속한 롤백 경로를 마련하세요.
   ```bash
   # 롤백 명령 준비
   kubectl rollout undo deployment/web
   ```

## 문제 해결 가이드

| 증상 | 1차 확인 방법 | 가능한 원인 | 해결 방안 |
|---|---|---|---|
| 파드 `Pending` 상태 지속 | `kubectl describe pod` 이벤트 확인 | 리소스 부족, 노드 테인트, 스케줄링 제약 | 리소스 요청량 조정, 노드 풀 확장, 테인트 허용 규칙 추가 |
| `ImagePullBackOff` 오류 | `kubectl describe pod` 이미지 관련 이벤트 | 레지스트리 접근 불가, 인증 정보 부재, 이미지 태그 오류 | `imagePullSecrets` 설정, 네트워크 연결 점검, 이미지 태그 확인 |
| `CrashLoopBackOff` 오류 | `kubectl logs -p` 이전 컨테이너 로그 확인 | 애플리케이션 시작 실패, 의존성 문제, 구성 오류 | 시작 스크립트 디버깅, 구성 파일 검증, 의존 서비스 연결 확인 |
| `Readiness` 프로브 실패 | `kubectl describe pod` 프로브 설정 확인 | 잘못된 경로/포트, 애플리케이션 시작 지연 | 프로브 경로/포트 수정, `initialDelaySeconds` 증가 |
| 서비스 접속 불가 | `kubectl get endpoints` 엔드포인트 확인 | 셀렉터-레이블 불일치, 포트 매핑 오류 | 레이블 일치 여부 확인, `targetPort` 설정 점검 |
| NodePort 타임아웃 | 노드 방화벽 규칙 확인 | 방화벽 차단, 노드 선택기 오류 | 보안 그룹/방화벽 허용 규칙 추가, 서비스 `nodeSelector` 검토 |

### 기본 진단 명령어 모음
```bash
# 클러스터 상태 종합 확인
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -20

# 특정 리소스 상세 분석
kubectl describe node <노드-이름>
kubectl describe pod <파드-이름> -n <네임스페이스>
kubectl logs <파드-이름> -n <네임스페이스> --previous

# 네트워크 연결 테스트
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
```

## 자주 사용하는 명령어 치트시트

```bash
# 컨텍스트 및 네임스페이스 관리
kubectl config get-contexts
kubectl config use-context my-cluster
kubectl config set-context --current --namespace=production

# 리소스 조회
kubectl get all
kubectl get pods -o wide
kubectl get pods -A --field-selector status.phase=Running
kubectl describe pod <파드-이름>

# 리소스 생성 및 적용
kubectl create deployment web --image=nginx:1.27-alpine
kubectl expose deployment web --port=80 --type=ClusterIP
kubectl apply -f k8s-manifests/

# 수정 및 배포 관리
kubectl set image deployment/web web=nginx:1.27.3
kubectl rollout status deployment/web
kubectl rollout undo deployment/web

# 디버깅 및 모니터링
kubectl logs -f deployment/web
kubectl exec -it deployment/web -- /bin/sh
kubectl port-forward service/web 8080:80
kubectl get events --sort-by=.lastTimestamp

# 리소스 사용량 확인
kubectl top pods -A
kubectl top nodes

# 리소스 삭제
kubectl delete -f k8s-manifests/
kubectl delete pod <파드-이름>
kubectl delete deployment web
```

## 결론: kubectl을 통한 효과적인 쿠버네티스 운영

kubectl은 단순한 명령줄 도구를 넘어 쿠버네티스 클러스터 운영의 핵심 인터페이스입니다. 효과적인 kubectl 사용법을 익히는 것은 안정적이고 효율적인 클라우드 네이티스 환경 운영의 기초가 됩니다.

### 핵심 원칙 요약:

1. **안전한 작업 환경 구축**: 컨텍스트와 네임스페이스 관리를 체계화하고, 실수로 잘못된 환경에서 명령을 실행하는 것을 방지하는 습관을 들이세요.

2. **선언적 관리 방식 채택**: YAML 파일을 Git으로 버전 관리하고, `kubectl apply`를 사용한 선언적 관리를 표준으로 삼으세요. 이는 재현성과 일관성을 보장합니다.

3. **체계적인 디버깅 프로세스 확립**: `describe`, `logs`, `events`, `top`, `port-forward` 명령어를 조합한 문제 해결 루틴을 개발하세요.

4. **생산성 도구 활용**: 자동 완성, krew 플러그인, 유용한 별칭을 통해 반복 작업을 최소화하세요.

5. **안전 장치 마련**: 중요한 작업 전에는 `--dry-run` 옵션으로 테스트하고, 변경 사항은 `kubectl diff`로 검토하며, 롤백 경로를 항상 준비하세요.

### 지속적인 학습과 개선:

쿠버네티스 생태계는 빠르게 발전하고 있으며, kubectl도 지속적으로 새로운 기능을 추가하고 있습니다. 다음 단계로 고려할 수 있는 항목들:

- **kustomize 통합**: `kubectl kustomize`를 활용한 환경별 구성 관리
- **플러그인 개발**: 조직의 특정 요구사항에 맞는 커스텀 kubectl 플러그인 개발
- **자동화 스크립트**: 반복적인 운영 작업을 자동화하는 스크립트 작성
- **협업 프로세스**: 팀 내 kubectl 사용 모범 사례 문서화 및 공유

kubectl 숙련도는 쿠버네티스 운영 능력과 직접적으로 연결됩니다. 이 가이드의 내용을 기반으로 실무에 적용하면서 점진적으로 고급 기능을 익히고, 조직의 운영 프로세스에 맞게 도구와 워크플로우를 발전시켜 나가세요.

기술은 변하지만, 재현성, 안전성, 관측성, 자동화라는 기본 원칙은 변하지 않습니다. kubectl을 효과적으로 활용하여 이러한 원칙을 실천하는 것이 장기적인 운영 성공의 열쇠입니다.