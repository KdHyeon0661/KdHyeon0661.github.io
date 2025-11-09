---
layout: post
title: Kubernetes - kubectl 디버깅 명령어
date: 2025-06-04 21:20:23 +0900
category: Kubernetes
---
# kubectl 디버깅 명령어 모음 (실전 진단용)

문제가 터지면 **kubectl만으로도 80% 이상**은 원인 가설을 세우고 첫 복구까지 갈 수 있습니다.  
아래는 기존 목록을 **상황 중심 플레이북**과 **깊이있는 예제**로 확장한 글입니다.  
모든 코드는 바로 복사해 실행 가능하도록 작성했습니다.

---

## 0) 30초 초진단(One-Liners)

```bash
# 0-1. 전체 리소스 건강 점검(현재 ns)
kubectl get pods,svc,deploy,rs,ingress

# 0-2. 최근 이벤트(클러스터 전역, 최신순)
kubectl get events -A --sort-by='.lastTimestamp' | tail -n 30

# 0-3. 문제 파드 한 눈에(상태/재시작/나이)
kubectl get pod -o wide

# 0-4. 충돌/재시작 빠른 스캔
kubectl get pods -A | egrep 'CrashLoopBackOff|Error|ImagePull|OOMKilled|Evicted'

# 0-5. 노드 상태와 테인트
kubectl get nodes -o wide
kubectl describe node <node>

# 0-6. 스케줄링 실패 원인(각 파드 이벤트)
kubectl describe pod <pod>

# 0-7. 직전 크래시 로그
kubectl logs <pod> -c <container> --previous --timestamps | tail -n 100
```

---

## 1) 기본 정보 조회 — 넓게 보고 깊게 들어가기

### 1.1 모든 리소스/특정 리소스

```bash
kubectl get all
kubectl get pods -A
kubectl get deploy -n dev
kubectl get svc
```

### 1.2 컨텍스트/네임스페이스 인식

```bash
kubectl config current-context
kubectl config view --minify | grep namespace
kubectl config set-context --current --namespace=dev
```

> **팁**: 프로젝트별로 ns를 고정해 오작동(다른 ns에 apply)을 줄이세요.

### 1.3 출력 커스터마이징

```bash
# Ready/Restart/Status를 요약
kubectl get pods -o=custom-columns=NAME:.metadata.name,READY:.status.containerStatuses[*].ready,RESTARTS:.status.containerStatuses[*].restartCount,STATUS:.status.phase,AGE:.metadata.creationTimestamp

# jsonpath로 이미지 버전만 빼기
kubectl get deploy api -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

---

## 2) Pod 상태/이벤트 분석 — 원인 힌트는 거의 여기 있다

### 2.1 describe로 전부 훑기

```bash
kubectl describe pod <pod>
```

- **이벤트**: `FailedScheduling`, `Back-off pulling image`, `Liveness probe failed` 등
- **상태**: `State: OOMKilled`, `ExitCode`, `Reason`, `Last State` 등
- **노드**: 어딘지, 어떤 조건인지(Ready/NotReady)

### 2.2 최근 이벤트(시간순)

```bash
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -A --field-selector type=Warning --sort-by='.lastTimestamp' | tail -n 50
```

### 2.3 상태별 필터

```bash
kubectl get pods --field-selector=status.phase=Pending
kubectl get pods | grep CrashLoopBackOff
```

---

## 3) 로그 분석 — 현재/직전/실시간

```bash
# 현재 컨테이너
kubectl logs <pod>
kubectl logs <pod> -c <container>

# 직전 컨테이너(크래시/재시작 원인)
kubectl logs <pod> -c <container> --previous

# 타임윈도우/라인 수/타임스탬프
kubectl logs <pod> --since=10m --tail=500 --timestamps
```

> **여러 파드 동시**: `stern` 도구 추천(아래 “추천 도구” 참고)

---

## 4) exec로 내부 디버깅 — 재현과 관측

```bash
# 쉘 진입
kubectl exec -it <pod> -- /bin/bash
kubectl exec -it <pod> -- /bin/sh

# 빠른 검사
kubectl exec <pod> -- env
kubectl exec <pod> -- cat /etc/resolv.conf
kubectl exec <pod> -- ss -lntp
kubectl exec <pod> -- curl -sS http://localhost:8080/healthz
```

> **이미지에 curl이 없다면?**  
> `kubectl run debugbox --rm -it --image=busybox -- /bin/sh` 로 옆에서 검사.  
> 또는 **ephemeral containers** 활용(아래 10장).

---

## 5) 리소스/노드 상태 — 스케줄러 관점과 런타임 관점

```bash
kubectl get nodes -o wide
kubectl describe node <node>
kubectl top nodes
kubectl top pods -A
```

- `Allocatable/Capacity`로 **requests 충족 가능 여부** 판단
- `Conditions`: `MemoryPressure`, `DiskPressure`, `PIDPressure` 확인
- `Taints`: 스케줄 차단 요인

---

## 6) 네트워크 디버깅 — DNS/Service/라우팅

```bash
# 서비스 존재/포트/엔드포인트
kubectl get svc
kubectl describe svc <svc>

# DNS 조회
kubectl exec -it <pod> -- nslookup <svc>.<ns>.svc.cluster.local
kubectl exec -it <pod> -- getent hosts <svc>.<ns>

# 내부→외부 통신/포트 개방 확인
kubectl port-forward svc/<svc> 8080:80
kubectl exec -it <pod> -- wget -qO- http://<other-svc>.<ns>.svc.cluster.local:8080/health
```

> **ClusterIP vs Headless**(selectors/endpoints) 차이를 `kubectl get endpoints`로 확인.

---

## 7) YAML 확인·수정 — 원하는 상태(Desired State)를 보라

```bash
kubectl get pod <pod> -o yaml
kubectl edit deployment <deploy>          # 즉석 수정(긴급 대응)
kubectl get deployment myapp -o yaml > myapp.yaml
# 수정 후
kubectl apply -f myapp.yaml
```

### 7.1 안전한 변경: Dry-Run & Diff

```bash
kubectl apply -f myapp.yaml --server-side --dry-run=client
kubectl diff -f myapp.yaml
```

---

## 8) 트러블슈팅 유틸 — 임시 파드·테스트 툴킷

```bash
# 간편 디버그 파드
kubectl run debugbox --rm -it --image=busybox -- /bin/sh

# (옵션) 기능 풍부한 도구
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
```

---

## 9) 특수 상황 진단 — 시나리오별 빠른 절차

### 9.1 CrashLoopBackOff
```bash
kubectl describe pod <pod>
kubectl logs <pod> -c <container> --previous
```
- **원인 패턴**: 앱 예외, CMD/ARGS 틀림, 프로브 과격, OOMKilled, 환경변수/시크릿 누락

### 9.2 Pending
```bash
kubectl describe pod <pod>     # FailedScheduling 이유 문자열
kubectl get events -A --sort-by='.lastTimestamp'
kubectl get nodes -o wide
```
- **원인 패턴**: requests 과다, PVC 미바인딩, Taint 미허용, NodeAffinity 불일치, Quota 초과

### 9.3 ImagePullBackOff
```bash
kubectl describe pod <pod> | sed -n '/Events:/,$p'
kubectl get secret -A | grep docker
```
- 이미지 경로/태그/리포 권한 점검, imagePullSecret 연결 여부 확인

### 9.4 OOMKilled/Throttling
```bash
kubectl describe pod <pod> | egrep 'OOM|Memory' -n
kubectl top pod <pod>
```
- 메모리 limit 상향/누수 점검, CPU throttling은 HPA/requests 조정

### 9.5 프로브 실패
```bash
kubectl describe pod <pod> | grep -A2 -E 'Liveness|Readiness'
kubectl port-forward <pod> 18080:8080
curl -i localhost:18080/healthz/ready
```
- `initialDelaySeconds`, `timeoutSeconds`, 경로/포트 재검증

### 9.6 볼륨 마운트 실패
```bash
kubectl describe pod <pod>
kubectl get pvc -A
kubectl describe pvc <pvc>
```
- StorageClass, AccessMode, size/zone 일치, PV 상태 확인

### 9.7 RBAC 거부(권한 문제)
```bash
kubectl auth can-i get pods --as user@example.com -n prod
kubectl auth can-i create secrets --as system:serviceaccount:dev:app-sa -n dev
```

---

## 10) Ephemeral Containers — 실행 중 파드에 디버거 주입

Kubernetes 1.25+에서 사용. 원 컨테이너 변경 없이 **임시 컨테이너**를 붙여 검사.

```bash
kubectl debug pod/<pod> -n <ns> --image=nicolaka/netshoot -it --target=<app-container>
```

- `--target`에 원 컨테이너 이름 지정(네임스페이스/네트워크 공유)
- 파드 스펙은 불변, 종료 시 임시 컨테이너만 사라짐

---

## 11) 스케줄링/배포 관점 진단 — rollout, scale, pdb

```bash
# 배포 진행/이력
kubectl rollout status deployment <deploy>
kubectl rollout history deployment <deploy>
kubectl rollout undo deployment <deploy> --to-revision=3

# 확장/축소
kubectl scale deploy <deploy> --replicas=5

# PDB 영향
kubectl get pdb -A
kubectl describe pdb <pdb>
```

---

## 12) 노드 유지보수/격리 — cordon/drain/uncordon (안전조치)

```bash
kubectl cordon <node>                                   # 새 스케줄 막기
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# 작업 후
kubectl uncordon <node>
```

- PDB로 중단 한도 보호, DaemonSet 제외

---

## 13) 네임스페이스/쿼터/리밋 — 정책에 막히는 경우

```bash
kubectl get resourcequota -n <ns>
kubectl describe resourcequota <rq> -n <ns>
kubectl get limitrange -n <ns> -o yaml
```

- Quota `used >= hard`이면 생성/확장 불가
- LimitRange가 기본 requests/limits를 강제

---

## 14) JSONPath/Label·Field Selector — “정확히” 집어내기

```bash
# 레이블로 특정 앱 팟만
kubectl get pods -l app=api -o wide

# 특정 필드로 필터(펜딩만)
kubectl get pods --field-selector status.phase=Pending

# jsonpath로 재시작 횟수만
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].restartCount}'; echo

# 지금 ns의 파드 이름만(스크립트 친화)
kubectl get po -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

---

## 15) 네트워크·DNS 정밀 체커 — 아키텍처 단위로 테스트

```bash
# 클러스터 DNS 레졸버 파악
kubectl exec -it <pod> -- cat /etc/resolv.conf

# 서비스 엔드포인트 직접 조회
kubectl get endpoints <svc>
kubectl describe endpoints <svc>

# Cross-NS FQDN 테스트
kubectl exec -it <pod> -- nslookup api.default.svc.cluster.local
```

> **Mesh 사용 시**: 사이드카 프록시(15090 metrics 등) 헬스/정책도 함께 점검.

---

## 16) 보안/권한/Secret — 최소권한·가시성

```bash
# 권한 검증
kubectl auth can-i list secrets -n dev --as system:serviceaccount:dev:app-sa
kubectl get role,rolebinding,clusterrole,clusterrolebinding -A

# Secret 평문 보기(필요시)
kubectl get secret my-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
```

- **주의**: 감사로그 활성화로 Secret 접근 추적 추천

---

## 17) 스토리지/CSI 문제 — 바인딩/접근모드/존 일치

```bash
kubectl get sc
kubectl describe sc <storageclass>
kubectl get pvc -A
kubectl describe pvc <pvc>
kubectl get pv
```

- `WaitForFirstConsumer`로 **존 불일치** 예방
- AccessMode(RWO/RWX)와 워크로드 패턴 확인

---

## 18) 자주 쓰는 alias/함수 — 손에 익히기

```bash
# ~/.bashrc or ~/.zshrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgn='kubectl get nodes -o wide'
alias kge='kubectl get events --sort-by=.lastTimestamp'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias kex='kubectl exec -it'
```

```bash
# 특정 레이블 가진 파드들의 최근 오류 로그 100줄
kerr () {
  ns=${1:-default}; label=${2:-app}
  pods=$(kubectl get po -n "$ns" -l "$label" -o name)
  for p in $pods; do
    echo "=== $p ==="
    kubectl logs -n "$ns" "$p" --tail=100 | egrep -i 'error|exception|fail' || true
  done
}
```

---

## 19) krew(플러그인 매니저) — kubectl 확장

```bash
# 설치(공식 문서 참고)
kubectl krew install ctx ns neat df-pv access-matrix resource-capacity
kubectl access-matrix -n default      # 주체별 접근 행렬
kubectl resource-capacity             # 리소스 집계
kubectl neat -f obj.yaml              # 매니페스트 정리
```

---

## 20) 실전 플레이북 — 10분 내 10가지 사고 대응

1. **느려짐(레이트 상승)**  
   `kubectl top pods` → 과도한 CPU? → `kubectl describe hpa`(있다면) → 임시 scale ↑ → 원인 탐색

2. **간헐 500**  
   `kubectl logs -l app=api --since=10m` → 프로브 실패? DB 타임아웃? → `port-forward`로 헬스 직접 확인

3. **배포 후 장애**  
   `kubectl rollout status` → `kubectl app diff`(GitOps) or `kubectl diff` → 즉시 `rollout undo`

4. **이미지 풀 실패**  
   `describe pod` 이벤트 → 레지스트리 인증/이미지 경로/태그 확인 → imagePullSecret 연결

5. **Pending 지속**  
   `describe pod`의 `FailedScheduling` → 노드 capacity/taint/affinity/pvc 확인 → requests 조정 or 노드 증설

6. **OOMKilled 연쇄**  
   `describe pod` → limit↑ + 앱 메모리 분석 → 캐시/버퍼 줄이기, VPA 추천 활용

7. **DNS 실패**  
   resolv.conf/dnsPolicy 파악 → CoreDNS 파드/ConfigMap 확인 → `nslookup`으로 FQDN 테스트

8. **서비스 라우팅 오동작**  
   `get svc/endpoints` → 엔드포인트 0? 셀렉터 라벨 틀림 → Deployment 라벨/셀렉터 재검증

9. **PVC 바인딩 실패**  
   `describe pvc` → SC/용량/존/AccessMode 확인 → PV 준비 또는 SC 파라미터 수정

10. **권한 거부**  
    `kubectl auth can-i ...` → Role/Binding 확인 → 최소권한 SA 바인딩

---

## 21) 요약 치트시트

| 목적 | 명령어 |
|---|---|
| Pod 상태 | `kubectl describe pod <pod>` |
| 로그(직전 포함) | `kubectl logs <pod> [-c <c>] [--previous]` |
| 이벤트 최근순 | `kubectl get events -A --sort-by='.lastTimestamp'` |
| 노드 상태 | `kubectl get/describe nodes`, `kubectl top nodes` |
| 리소스 사용량 | `kubectl top pods [-A]` |
| 네트워크 | `describe svc`, `nslookup`, `port-forward` |
| 스케줄/배포 | `rollout status/history/undo`, `scale` |
| 스토리지 | `get/describe pvc,pv,sc` |
| 보안/RBAC | `kubectl auth can-i ...` |
| 임시 디버그 | `kubectl run debugbox ...`, `kubectl debug --target` |

---

## 22) 추천 도구

- [`k9s`](https://k9scli.io) — 터미널 UI로 리소스 탐색/조작
- [`stern`](https://github.com/stern/stern) — 레이블로 다중 파드 로그 실시간 tail
- [`kubectx`/`kubens`](https://github.com/ahmetb/kubectx) — 컨텍스트/네임스페이스 전환
- [`lens`](https://k8slens.dev) — GUI 클러스터 뷰어
- [`krew`](https://krew.sigs.k8s.io) — kubectl 플러그인 매니저

---

## 23) 마무리

- **describe + events + logs**의 삼단 콤보로 가설을 세우고,  
- **exec/port-forward**로 재현·검증,  
- **rollout/scale/patch**로 1차 복구,  
- **정책/리소스/스토리지**로 근본 원인을 닫습니다.

잦은 명령은 **alias/함수화**하고, 조직 표준 **런북**으로 공유하세요.  
“kubectl만 잘 써도” 진단/복구 속도는 놀랄 만큼 빨라집니다.