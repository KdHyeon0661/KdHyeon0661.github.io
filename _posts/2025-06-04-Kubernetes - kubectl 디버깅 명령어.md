---
layout: post
title: Kubernetes - kubectl 디버깅 명령어
date: 2025-06-04 21:20:23 +0900
category: Kubernetes
---
# Kubernetes 디버깅을 위한 kubectl 명령어 가이드

## 빠른 진단을 위한 핵심 명령어

Kubernetes 클러스터에서 문제를 신속하게 진단하기 위한 필수 명령어들입니다:

```bash
# 1. 전체 리소스 상태 점검 (현재 네임스페이스)
kubectl get pods,svc,deploy,rs,ingress

# 2. 최근 이벤트 확인 (클러스터 전체, 최신순 정렬)
kubectl get events -A --sort-by='.lastTimestamp' | tail -n 30

# 3. 파드 상태 개요 (상태, 재시작 횟수, 실행 시간)
kubectl get pod -o wide

# 4. 문제가 있는 파드 검색
kubectl get pods -A | egrep 'CrashLoopBackOff|Error|ImagePull|OOMKilled|Evicted'

# 5. 노드 상태와 테인트 확인
kubectl get nodes -o wide
kubectl describe node <node>

# 6. 스케줄링 실패 원인 분석
kubectl describe pod <pod>

# 7. 이전 컨테이너 크래시 로그 확인
kubectl logs <pod> -c <container> --previous --timestamps | tail -n 100
```

---

## 기본 정보 조회 및 컨텍스트 관리

### 리소스 목록 확인

```bash
# 모든 리소스 타입 확인
kubectl get all

# 모든 네임스페이스의 파드 확인
kubectl get pods -A

# 특정 네임스페이스의 디플로이먼트 확인
kubectl get deploy -n dev

# 서비스 목록 확인
kubectl get svc
```

### 컨텍스트 및 네임스페이스 관리

```bash
# 현재 컨텍스트 확인
kubectl config current-context

# 현재 컨텍스트의 네임스페이스 확인
kubectl config view --minify | grep namespace

# 현재 컨텍스트의 기본 네임스페이스 변경
kubectl config set-context --current --namespace=dev
```

> **운영 팁**: 프로젝트별로 기본 네임스페이스를 설정하여 다른 네임스페이스에 실수로 리소스를 배포하는 오류를 방지하세요.

### 출력 포맷 커스터마이징

```bash
# 사용자 정의 컬럼으로 파드 정보 표시
kubectl get pods -o=custom-columns=NAME:.metadata.name,READY:.status.containerStatuses[*].ready,RESTARTS:.status.containerStatuses[*].restartCount,STATUS:.status.phase,AGE:.metadata.creationTimestamp

# JSONPath를 사용하여 디플로이먼트의 이미지 정보 추출
kubectl get deploy api -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

---

## 파드 상태 및 이벤트 분석

### 상세 정보 확인

```bash
# 파드의 상세 정보 확인 (문제 분석의 가장 중요한 도구)
kubectl describe pod <pod>
```

`describe` 명령어 출력에서 주목해야 할 부분:
- **이벤트 섹션**: `FailedScheduling`, `Back-off pulling image`, `Liveness probe failed` 등의 메시지
- **상태 정보**: `State: OOMKilled`, `ExitCode`, `Reason`, `Last State` 등의 상세 상태
- **노드 정보**: 파드가 실행 중인 노드와 해당 노드의 상태

### 이벤트 분석

```bash
# 시간순으로 이벤트 정렬
kubectl get events --sort-by='.lastTimestamp'

# 경고 이벤트만 필터링하여 확인
kubectl get events -A --field-selector type=Warning --sort-by='.lastTimestamp' | tail -n 50
```

### 상태별 파드 필터링

```bash
# Pending 상태인 파드만 확인
kubectl get pods --field-selector=status.phase=Pending

# CrashLoopBackOff 상태인 파드 검색
kubectl get pods | grep CrashLoopBackOff
```

---

## 로그 분석 기술

```bash
# 현재 실행 중인 컨테이너 로그 확인
kubectl logs <pod>
kubectl logs <pod> -c <container>

# 이전에 종료된 컨테이너 로그 확인 (크래시 분석)
kubectl logs <pod> -c <container> --previous

# 특정 시간대의 로그 확인
kubectl logs <pod> --since=10m --tail=500 --timestamps
```

> **다중 파드 로그 모니터링**: `stern` 도구를 사용하면 레이블로 선택한 여러 파드의 로그를 실시간으로 모니터링할 수 있습니다.

---

## 파드 내부 진단 및 검사

```bash
# 파드 내부로 쉘 접근
kubectl exec -it <pod> -- /bin/bash
kubectl exec -it <pod> -- /bin/sh

# 빠른 환경 검사
kubectl exec <pod> -- env
kubectl exec <pod> -- cat /etc/resolv.conf
kubectl exec <pod> -- ss -lntp
kubectl exec <pod> -- curl -sS http://localhost:8080/healthz
```

> **도구가 없는 이미지 대처 방법**: `kubectl run debugbox --rm -it --image=busybox -- /bin/sh` 명령어로 임시 디버그 컨테이너를 실행하거나, ephemeral containers 기능을 활용하세요.

---

## 리소스 및 노드 상태 분석

```bash
# 노드 상태 확인
kubectl get nodes -o wide
kubectl describe node <node>

# 리소스 사용량 모니터링
kubectl top nodes
kubectl top pods -A
```

주요 확인 사항:
- `Allocatable/Capacity`: 리소스 요청 충족 가능 여부 판단
- `Conditions`: `MemoryPressure`, `DiskPressure`, `PIDPressure` 상태 확인
- `Taints`: 노드 스케줄링 제약 조건 확인

---

## 네트워크 문제 진단

```bash
# 서비스 정보 확인
kubectl get svc
kubectl describe svc <svc>

# DNS 확인
kubectl exec -it <pod> -- nslookup <svc>.<ns>.svc.cluster.local
kubectl exec -it <pod> -- getent hosts <svc>.<ns>

# 포트 포워딩을 통한 로컬 접근
kubectl port-forward svc/<svc> 8080:80

# 서비스 간 통신 테스트
kubectl exec -it <pod> -- wget -qO- http://<other-svc>.<ns>.svc.cluster.local:8080/health
```

> **ClusterIP와 Headless 서비스 차이**: `kubectl get endpoints` 명령어로 서비스의 엔드포인트를 확인하여 정상적으로 연결되었는지 검증하세요.

---

## 매니페스트 확인 및 수정

```bash
# 파드 정의 확인
kubectl get pod <pod> -o yaml

# 디플로이먼트 실시간 수정 (긴급 대응용)
kubectl edit deployment <deploy>

# 매니페스트 파일로 저장
kubectl get deployment myapp -o yaml > myapp.yaml

# 수정된 매니페스트 적용
kubectl apply -f myapp.yaml
```

### 안전한 변경 절차

```bash
# 변경 사항 시뮬레이션
kubectl apply -f myapp.yaml --server-side --dry-run=client

# 현재 상태와의 차이점 확인
kubectl diff -f myapp.yaml
```

---

## 디버깅용 임시 컨테이너

```bash
# 간단한 디버그 컨테이너 실행
kubectl run debugbox --rm -it --image=busybox -- /bin/sh

# 네트워크 진단 도구가 포함된 컨테이너
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
```

---

## 특정 문제 시나리오별 진단 절차

### CrashLoopBackOff 문제

```bash
kubectl describe pod <pod>
kubectl logs <pod> -c <container> --previous
```

**주요 원인**: 애플리케이션 예외, 잘못된 커맨드/인수, 프로브 설정 문제, OOMKilled, 환경변수/시크릿 누락

### Pending 상태 문제

```bash
kubectl describe pod <pod>     # FailedScheduling 원인 확인
kubectl get events -A --sort-by='.lastTimestamp'
kubectl get nodes -o wide
```

**주요 원인**: 리소스 요청 과다, PVC 바인딩 실패, Taint 허용 안됨, NodeAffinity 불일치, 쿼터 초과

### ImagePullBackOff 문제

```bash
kubectl describe pod <pod> | sed -n '/Events:/,$p'
kubectl get secret -A | grep docker
```

**주요 원인**: 이미지 경로/태그 오류, 레지스트리 인증 문제, imagePullSecret 연결 누락

### OOMKilled/Throttling 문제

```bash
kubectl describe pod <pod> | egrep 'OOM|Memory' -n
kubectl top pod <pod>
```

**해결 방안**: 메모리 제한 상향, 메모리 누수 분석, CPU 요청 조정, HPA 설정 검토

### 프로브 실패 문제

```bash
kubectl describe pod <pod> | grep -A2 -E 'Liveness|Readiness'
kubectl port-forward <pod> 18080:8080
curl -i localhost:18080/healthz/ready
```

**해결 방안**: `initialDelaySeconds`, `timeoutSeconds` 조정, 경로/포트 재확인

### 볼륨 마운트 실패

```bash
kubectl describe pod <pod>
kubectl get pvc -A
kubectl describe pvc <pvc>
```

**주요 원인**: StorageClass 불일치, AccessMode 제한, 용량/가용성 영역 불일치, PV 상태 문제

### RBAC 권한 문제

```bash
kubectl auth can-i get pods --as user@example.com -n prod
kubectl auth can-i create secrets --as system:serviceaccount:dev:app-sa -n dev
```

---

## Ephemeral Containers: 실행 중 파드에 디버거 주입

Kubernetes 1.25 이상에서 지원되는 기능으로, 기존 컨테이너를 변경하지 않고 임시 디버그 컨테이너를 주입할 수 있습니다.

```bash
kubectl debug pod/<pod> -n <ns> --image=nicolaka/netshoot -it --target=<app-container>
```

**특징**:
- `--target` 옵션으로 특정 컨테이너의 네임스페이스/네트워크 공유
- 원래 파드 스펙은 변경되지 않음
- 디버그 세션 종료 시 임시 컨테이너만 제거됨

---

## 배포 및 스케일링 문제 진단

```bash
# 배포 상태 확인
kubectl rollout status deployment <deploy>

# 배포 이력 확인
kubectl rollout history deployment <deploy>

# 특정 리비전으로 롤백
kubectl rollout undo deployment <deploy> --to-revision=3

# 레플리카 수 조정
kubectl scale deploy <deploy> --replicas=5

# PodDisruptionBudget 확인
kubectl get pdb -A
kubectl describe pdb <pdb>
```

---

## 노드 유지보수 작업

```bash
# 노드에 새로운 파드 스케줄링 차단
kubectl cordon <node>

# 노드에서 실행 중인 파드 안전하게 이동
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# 노드 스케줄링 재개
kubectl uncordon <node>
```

**참고사항**: PodDisruptionBudget을 통해 중단 허용 한도를 보호하고, DaemonSet은 제외 처리합니다.

---

## 리소스 쿼터 및 제한 확인

```bash
# 네임스페이스의 리소스 쿼터 확인
kubectl get resourcequota -n <ns>
kubectl describe resourcequota <rq> -n <ns>

# 기본 리소스 제한 확인
kubectl get limitrange -n <ns> -o yaml
```

**주요 확인점**: 사용량(`used`)이 한도(`hard`)에 도달하면 새로운 리소스 생성이나 확장이 불가능합니다. LimitRange는 파드/컨테이너의 기본 리소스 요청과 제한을 강제합니다.

---

## 정밀한 리소스 필터링

```bash
# 레이블로 특정 애플리케이션의 파드만 선택
kubectl get pods -l app=api -o wide

# 특정 상태의 파드만 필터링
kubectl get pods --field-selector status.phase=Pending

# JSONPath로 재시작 횟수 추출
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].restartCount}'; echo

# 네임스페이스의 모든 파드 이름 추출 (스크립트 호환)
kubectl get po -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

---

## 네트워크 및 DNS 상세 진단

```bash
# DNS 설정 확인
kubectl exec -it <pod> -- cat /etc/resolv.conf

# 서비스 엔드포인트 직접 확인
kubectl get endpoints <svc>
kubectl describe endpoints <svc>

# 크로스 네임스페이스 DNS 확인
kubectl exec -it <pod> -- nslookup api.default.svc.cluster.local
```

> **서비스 메시 환경**: 사이드카 프록시(예: 15090 메트릭 포트)의 헬스 및 정책 설정도 함께 점검해야 합니다.

---

## 보안 및 권한 검증

```bash
# 특정 권한 검증
kubectl auth can-i list secrets -n dev --as system:serviceaccount:dev:app-sa

# RBAC 구성 확인
kubectl get role,rolebinding,clusterrole,clusterrolebinding -A

# Secret 내용 확인 (필요시)
kubectl get secret my-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
```

**보안 권고**: Secret 접근 이력을 추적하기 위해 감사 로깅을 활성화하세요.

---

## 스토리지 문제 진단

```bash
# StorageClass 확인
kubectl get sc
kubectl describe sc <storageclass>

# PVC 상태 확인
kubectl get pvc -A
kubectl describe pvc <pvc>

# PV 상태 확인
kubectl get pv
```

**주요 확인점**: `WaitForFirstConsumer` 정책으로 가용성 영역 불일치를 예방하고, AccessMode(RWO/RWX)가 워크로드 패턴과 일치하는지 확인하세요.

---

## 효율성을 위한 Alias 및 함수 정의

```bash
# ~/.bashrc 또는 ~/.zshrc에 추가

# 기본 Alias
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgn='kubectl get nodes -o wide'
alias kge='kubectl get events --sort-by=.lastTimestamp'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias kex='kubectl exec -it'

# 특정 레이블의 파드에서 오류 로그 검색 함수
kerr () {
  ns=${1:-default}
  label=${2:-app}
  pods=$(kubectl get po -n "$ns" -l "$label" -o name)
  for p in $pods; do
    echo "=== $p ==="
    kubectl logs -n "$ns" "$p" --tail=100 | egrep -i 'error|exception|fail' || true
  done
}
```

---

## kubectl 확장 도구

```bash
# krew를 통한 플러그인 설치
kubectl krew install ctx ns neat df-pv access-matrix resource-capacity

# 접근 권한 매트릭스 확인
kubectl access-matrix -n default

# 리소스 용량 확인
kubectl resource-capacity

# 매니페스트 정리
kubectl neat -f obj.yaml
```

---

## 실전 문제 해결 플레이북

1. **성능 저하 문제**
   `kubectl top pods` → CPU 사용량 확인 → `kubectl describe hpa` 검토 → 임시 스케일 업 → 근본 원인 분석

2. **간헐적 5xx 오류**
   `kubectl logs -l app=api --since=10m` → 프로브 실패 확인 → `port-forward`로 직접 헬스 체크

3. **배포 후 장애**
   `kubectl rollout status` 확인 → `kubectl diff`로 변경사항 확인 → 즉시 `rollout undo` 실행

4. **이미지 풀 실패**
   `describe pod`의 이벤트 확인 → 레지스트리 인증/이미지 경로/태그 점검 → imagePullSecret 연결 확인

5. **Pending 상태 지속**
   `describe pod`의 `FailedScheduling` 메시지 확인 → 노드 용량/테인트/어피니티/PVC 상태 확인

6. **OOMKilled 반복 발생**
   `describe pod`로 메모리 상태 확인 → limit 상향 조정 → 애플리케이션 메모리 사용 패턴 분석

7. **DNS 실패**
   resolv.conf/dnsPolicy 확인 → CoreDNS 파드 및 ConfigMap 점검 → FQDN으로 `nslookup` 테스트

8. **서비스 라우팅 문제**
   `get svc/endpoints`로 엔드포인트 확인 → 셀렉터와 디플로이먼트 라벨 일치성 검증

9. **PVC 바인딩 실패**
   `describe pvc`로 상태 확인 → StorageClass/용량/가용성 영역/AccessMode 검토

10. **권한 거부 오류**
    `kubectl auth can-i ...`로 권한 검증 → Role/Binding 구성 확인 → 최소 권한 원칙 적용

---

## 핵심 명령어 요약

| 목적 | 명령어 |
|---|---|
| 파드 상태 분석 | `kubectl describe pod <pod>` |
| 로그 확인 (이전 포함) | `kubectl logs <pod> [-c <c>] [--previous]` |
| 최신 이벤트 확인 | `kubectl get events -A --sort-by='.lastTimestamp'` |
| 노드 상태 확인 | `kubectl get/describe nodes`, `kubectl top nodes` |
| 리소스 사용량 확인 | `kubectl top pods [-A]` |
| 네트워크 진단 | `describe svc`, `nslookup`, `port-forward` |
| 배포 관리 | `rollout status/history/undo`, `scale` |
| 스토리지 확인 | `get/describe pvc,pv,sc` |
| 권한 검증 | `kubectl auth can-i ...` |
| 임시 디버깅 | `kubectl run debugbox ...`, `kubectl debug --target` |

---

## 추천 보조 도구

- **[k9s](https://k9scli.io)**: 터미널 기반 Kubernetes 클러스터 관리 UI
- **[stern](https://github.com/stern/stern)**: 레이블 기반 다중 파드 로그 모니터링 도구
- **[kubectx/kubens](https://github.com/ahmetb/kubectx)**: 컨텍스트 및 네임스페이스 전환 도구
- **[Lens](https://k8slens.dev)**: 그래픽 Kubernetes 클러스터 관리자
- **[krew](https://krew.sigs.k8s.io)**: kubectl 플러그인 관리자

---

## 결론

효과적인 Kubernetes 디버깅은 체계적인 접근 방식을 요구합니다:

1. **진단 삼박자 활용**: `describe`, `events`, `logs` 명령어를 조합하여 문제 가설을 수립하세요.
2. **실증적 검증**: `exec`와 `port-forward`를 통해 문제를 재현하고 검증하세요.
3. **안전한 복구**: `rollout`, `scale`, `patch` 명령어로 1차 복구를 수행하세요.
4. **근본 원인 분석**: 정책, 리소스, 스토리지 관점에서 문제의 근본 원인을 해결하세요.

자주 사용하는 명령어는 alias나 함수로 등록하여 효율성을 높이고, 조직 내에서 표준 운영 절차(Runbook)를 공유하여 일관된 문제 해결 방식을 확립하세요. kubectl 명령어만 능숙하게 활용해도 대부분의 Kubernetes 문제를 신속하게 진단하고 해결할 수 있습니다. 이 가이드의 명령어들과 접근 방식을 숙지하면 실제 운영 환경에서 발생하는 다양한 문제 상황에 효과적으로 대응할 수 있을 것입니다.