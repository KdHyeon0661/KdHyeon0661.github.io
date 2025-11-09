---
layout: post
title: Kubernetes - kubectl logs
date: 2025-05-05 19:20:23 +0900
category: Kubernetes
---
# Kubernetes 로그 확인: `kubectl logs`

## 1. 기본 사용

```bash
kubectl logs <pod-name>
# 예
kubectl logs my-nginx-74b46bcf5b-wbp5d
```

- Pod 내 첫 번째 컨테이너의 로그를 출력한다.
- 네임스페이스가 다르면 `-n <ns>`를 붙인다.

```bash
kubectl logs -n prod my-nginx-74b46bcf5b-wbp5d
```

---

## 2. 멀티 컨테이너 Pod

컨테이너가 2개 이상이면 `-c/--container`로 명시한다.

```bash
kubectl logs <pod> -c <container>
# 예
kubectl logs my-app-pod -c main
kubectl logs my-app-pod -c sidecar
```

컨테이너 이름 조회:

```bash
kubectl get pod my-app-pod -o jsonpath='{.spec.containers[*].name}'
```

모든 컨테이너 로그를 한 번에:

```bash
kubectl logs <pod> --all-containers
# prefix를 붙여 구분
kubectl logs <pod> --all-containers --prefix
```

Init 컨테이너 로그:

```bash
kubectl logs <pod> -c <init-container-name>
```

---

## 3. 이전 인스턴스 로그(재시작 후)

CrashLoopBackOff 등으로 컨테이너가 재시작됐다면 **이전 컨테이너**의 로그가 중요하다.

```bash
kubectl logs <pod> --previous
# 특정 컨테이너의 이전 로그
kubectl logs <pod> -c <container> --previous
```

---

## 4. 실시간 스트리밍

```bash
kubectl logs -f <pod>
kubectl logs -f <pod> -c <container>
```

- 파일의 `tail -f`처럼 **새로 쓰이는 로그를 지속 출력**한다.
- 중지: `Ctrl + C`

---

## 5. 시간/줄/바이트 기반 제한

배포 자동화/디버깅에서 **로그 절제 출력**이 필요할 수 있다.

```bash
# 최근 N줄
kubectl logs <pod> --tail=100

# 특정 시점 이후
kubectl logs <pod> --since=10m          # 10분 이내
kubectl logs <pod> --since=2h           # 2시간 이내
kubectl logs <pod> --since-time="2025-11-09T06:00:00Z"

# 타임스탬프 포함
kubectl logs <pod> --timestamps

# 바이트 제한(긴 로그 차단; K8s 버전에 따라 제한적)
kubectl logs <pod> --limit-bytes=1048576
```

---

## 6. 여러 Pod 동시 조회(레이블/컨트롤러)

### 6.1 레이블 셀렉터

```bash
# 같은 라벨을 가진 Pod들의 로그를 병렬로 요청
kubectl logs -l app=my-api --all-containers --tail=200 \
  --max-log-requests=5
```

- `--max-log-requests`: 동시 요청 개수 제한(클러스터/로컬 리소스 보호)
- `--prefix`: 컨테이너/Pod 구분 접두어 부여

```bash
kubectl logs -l app=my-api --all-containers --prefix --tail=50
```

### 6.2 컨트롤러(Deployment/StatefulSet/Job) 대상

```bash
kubectl logs deployment/my-api -c app --tail=100
kubectl logs statefulset/my-db -c postgres --since=30m
kubectl logs job/batch-etl --all-containers --tail=500
```

- 내부적으로 **소유한 Pod들을 해석**해 각 Pod 로그를 조회한다.

---

## 7. 파일로 저장/가공

```bash
# 파일 저장
kubectl logs <pod> > app.log

# 스트리밍 저장
kubectl logs -f <pod> | tee -a app-follow.log

# 에러 행만 추출
kubectl logs <pod> | grep -E "ERROR|FATAL|SEVERE" > errors.log

# JSON 로그라면 jq 활용
kubectl logs <pod> | jq -r 'select(.level=="error") | .message'
```

멀티 Pod를 한 파일로 묶기:

```bash
kubectl get pods -l app=my-api -o name \
| xargs -I {} sh -c 'kubectl logs {} --tail=200 --prefix' \
> all-my-api.log
```

---

## 8. 운영 시나리오별 스니펫

### 8.1 CrashLoopBackOff 디버깅 원샷

```bash
pod=$(kubectl get pods -l app=myapp --sort-by=.status.startTime | tail -n1 | awk '{print $1}')
kubectl describe pod "$pod"
kubectl logs "$pod" --previous --all-containers --tail=200
```

### 8.2 Readiness/Liveness 실패 시 엔드포인트 검증

```bash
pod=$(kubectl get pods -l app=web --field-selector=status.phase=Running -o name | head -n1)
kubectl logs $pod --tail=50
kubectl port-forward $pod 8080:8080 &
sleep 1
curl -i http://localhost:8080/ready || true
curl -i http://localhost:8080/health || true
kill %1
```

### 8.3 특정 기간 경고 행 추출

```bash
# 지난 15분
kubectl logs <pod> --since=15m --timestamps \
| awk '$0 ~ /WARN|ERROR|FATAL/'
```

### 8.4 사이드카/프록시와 앱 로그 동시 감시

```bash
kubectl logs -f <pod> --all-containers --prefix --tail=20
```

---

## 9. Job/CronJob 로그

Job은 여러 Pod(재시도 포함)를 만들 수 있어 **컨트롤러 대상 조회**가 편리하다.

```bash
# 최근 N줄씩, 모든 컨테이너
kubectl logs job/my-job --all-containers --tail=200

# 실패한 Pod의 이전 로그(재시도 직전)
for p in $(kubectl get pods -l job-name=my-job -o name); do
  echo "=== $p (previous) ==="
  kubectl logs "$p" --previous --all-containers --tail=100 || true
done
```

CronJob은 생성된 Job 이름을 확인한 뒤 동일한 패턴으로 조회한다.

```bash
kubectl get jobs -l cronjob-name=my-cron
kubectl logs job/<that-job> --tail=200
```

---

## 10. 네임스페이스/컨텍스트/권한

```bash
# 네임스페이스 전환
kubectl logs -n staging <pod>

# kubeconfig 컨텍스트를 바꿔서 다른 클러스터 조회
kubectl config use-context my-eks
kubectl logs -n prod <pod>
```

RBAC로 로그 조회를 허용하려면 최소 `pods/log`에 대한 `get` 권한이 필요하다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: read-logs, namespace: prod }
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get","list","watch"]
```

---

## 11. 로그가 보이지 않을 때(한눈에 보는 원인/해결)

| 증상 | 원인 | 확인 | 해결 |
|---|---|---|---|
| `no logs available` | 컨테이너 미시작/즉시 종료 | `kubectl describe pod`의 상태/이벤트 | `--previous`로 직전 인스턴스, 시작 스크립트/이미지 점검 |
| 로그가 매우 짧음 | 컨테이너가 빠르게 크래시 | `Restart Count`, `--previous` | 원인 코드/환경변수 수정, 리소스/포트 충돌 해결 |
| 아무 것도 안 나옴 | 앱이 파일로만 기록 | 컨테이너 명령/이미지 엔트리포인트 | 애플리케이션을 **stdout/stderr**로 쓰도록 변경 |
| 중간에 끊김 | 컨테이너 로그 로테이션 | 노드 `/var/log/containers`/kubelet 설정 | 로그 축적은 수집기(Fluent Bit 등) 도입 |
| 권한 거부 | RBAC 제한 | `kubectl auth can-i get pods/log -n <ns>` | Role/ClusterRole에 `pods/log` 권한 추가 |
| 멀티라인 깨짐 | 스택 트레이스 등 줄 단위 | 원 로그 포맷 점검 | JSON 구조화+수집기 멀티라인 옵션 적용 |
| 간헐적 공백 | 사이드카/버퍼링 | 출력 버퍼 모드 | `STDOUT` line-buffered/flush 설정, 프록시 버퍼 확인 |

---

## 12. `kubectl logs` 동작 이해(요점)

- 컨테이너 런타임(containerd 등)은 **컨테이너 stdout/stderr**을 파일로 기록한다.  
  일반 경로: `/var/log/containers/*.log` → `/var/log/pods/` → 런타임 내부 경로.
- kubelet은 해당 파일을 **스트림**으로 제공하고, `kubectl logs`는 이를 읽는다.
- 따라서 **장기 보관/검색/집계**는 `kubectl logs`의 역할이 아니며, EFK/PLG/Cloud Logging 등 **수집 파이프라인**이 필요하다.

---

## 13. 구조화 로그와 필터링(추천)

애플리케이션 로그를 JSON으로 구조화하면 CLI에서 즉시 필터링/집계가 가능하다.

```bash
# 예: level=error만, message 필드 출력
kubectl logs <pod> | jq -r 'select(.level=="error") | .message'

# 5xx 응답만 추출
kubectl logs <pod> | jq -c 'select(.status>=500) | {ts, path, status}'
```

---

## 14. 팀 운영을 위한 표준 스크립트 예제

### 14.1 최근 배포된 Revision의 모든 Pod 로그 묶기

```bash
ns=prod
deploy=my-api

# 최신 ReplicaSet 파악
rs=$(kubectl get rs -n "$ns" \
  --selector=app=$deploy \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1:].metadata.name}')

# 해당 RS의 Pod 로그 수집
kubectl get pods -n "$ns" -l "app=$deploy,replica-set=$rs" -o name \
| xargs -I {} sh -c 'echo "=== {} ==="; kubectl logs -n '"$ns"' {} --all-containers --tail=200 --prefix' \
> logs-${deploy}-latest.txt
```

### 14.2 “지난 10분 ERROR” 대시보드용 파이프

```bash
kubectl logs -l app=checkout --since=10m --timestamps --all-containers \
| awk '/ERROR|FATAL/' \
| sed 's/\.000000000Z/Z/' \
> checkout-errors-10m.log
```

---

## 15. 모범 사례

- **stdout/stderr**에 로그를 남긴다. 파일만 쓰면 `kubectl logs`와 수집기가 못 본다.  
- **구조화(JSON) 로깅**을 권장한다. 필터링/집계가 쉬워진다.  
- 멀티라인(스택 트레이스)은 수집기에서 멀티라인 파서를 설정한다.  
- 대량 조회 시 `--tail`, `--since`, `--max-log-requests`로 **클러스터/로컬 보호**.  
- 컨트롤러(Deployment/StatefulSet/Job)를 대상으로 **집합 조회**를 습관화한다.  
- 장기 보관/검색/알림은 **Promtail/Loki**, **Fluent Bit/Elasticsearch/Kibana**, **Vector/OpenSearch**, **Cloud Logging** 등 스택을 사용한다.

---

## 16. 빠른 레퍼런스(치트시트)

```bash
# 단일 컨테이너
kubectl logs <pod>

# 특정 컨테이너
kubectl logs <pod> -c <container>

# 이전 인스턴스(재시작 전)
kubectl logs <pod> --previous

# 실시간 스트림
kubectl logs -f <pod>

# 최근 50줄 / 10분 이내
kubectl logs <pod> --tail=50
kubectl logs <pod> --since=10m

# 모든 컨테이너 + 접두어
kubectl logs <pod> --all-containers --prefix

# 레이블 셀렉터(여러 Pod)
kubectl logs -l app=my-api --tail=200 --all-containers

# 컨트롤러 대상
kubectl logs deployment/my-api -c app --tail=100
kubectl logs job/nightly-etl --all-containers --tail=500

# 타임스탬프, 바이트 제한
kubectl logs <pod> --timestamps --limit-bytes=1048576

# 파일 저장
kubectl logs <pod> > pod.log
```

---

## 17. 결론

`kubectl logs`는 **가장 빠르고 가벼운 1차 진단 도구**다.  
레이블/컨트롤러 기반 조회, 시간/줄 제한, 이전 인스턴스 조회, 실시간 스트리밍을 조합하면  
대부분의 애플리케이션 이슈를 **분 단위로 가설 → 검증**할 수 있다.  
장기 보관/탐색은 별도의 로깅 스택으로 보완하고, 애플리케이션은 stdout/stderr에 **구조화 로그**를 남기는 습관을 들이자.