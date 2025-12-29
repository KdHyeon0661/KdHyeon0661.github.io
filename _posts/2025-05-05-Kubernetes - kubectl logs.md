---
layout: post
title: Kubernetes - kubectl logs
date: 2025-05-05 19:20:23 +0900
category: Kubernetes
---
# Kubernetes 로그 확인: `kubectl logs` 완전 가이드

## 기본 개념과 사용법

`kubectl logs` 명령어는 Kubernetes에서 가장 빈번하게 사용되는 진단 도구 중 하나로, 실행 중인 Pod의 컨테이너 로그를 실시간으로 확인할 수 있는 강력한 기능을 제공합니다. 로깅은 단순히 문제를 해결하는 데 그치지 않고 애플리케이션의 동작 패턴을 이해하고 시스템 건강 상태를 모니터링하는 핵심 수단입니다.

```bash
# 기본 사용법
kubectl logs <pod-name>

# 예시: 기본 Pod 로그 확인
kubectl logs my-nginx-74b46bcf5b-wbp5d

# 특정 네임스페이스의 Pod 로그 확인
kubectl logs -n production my-nginx-74b46bcf5b-wbp5d
```

기본적으로 이 명령은 Pod 내 첫 번째 컨테이너의 로그를 출력합니다. Kubernetes는 컨테이너의 표준 출력(stdout)과 표준 오류(stderr) 스트림을 캡처하여 관리하므로, 애플리케이션은 이러한 스트림으로 로그를 출력하는 것이 모범 사례입니다.

---

## 다양한 컨테이너 시나리오 대응

### 멀티 컨테이너 Pod에서의 로그 확인
현대적인 애플리케이션 아키텍처에서는 사이드카 패턴이 흔히 사용되어 하나의 Pod 안에 여러 컨테이너가 공존합니다. 이 경우 특정 컨테이너의 로그를 확인하려면 컨테이너 이름을 명시해야 합니다.

```bash
# 특정 컨테이너의 로그 확인
kubectl logs my-app-pod -c main-application
kubectl logs my-app-pod -c log-shipper-sidecar

# Pod 내 모든 컨테이너의 로그를 한 번에 확인
kubectl logs my-app-pod --all-containers

# 각 로그 라인에 컨테이너 이름을 접두어로 추가하여 구분
kubectl logs my-app-pod --all-containers --prefix
```

먼저 Pod에 어떤 컨테이너가 있는지 확인하려면 다음 명령을 사용할 수 있습니다.
```bash
kubectl get pod my-app-pod -o jsonpath='{.spec.containers[*].name}'
```

### Init 컨테이너 로그 확인
Init 컨테이너는 주 컨테이너가 시작되기 전에 실행되는 특수한 컨테이너로, 설정 준비나 데이터베이스 마이그레이션과 같은 초기화 작업을 수행합니다. 이들의 로그는 문제 진단에 중요한 단서를 제공할 수 있습니다.

```bash
# Init 컨테이너 로그 확인
kubectl logs my-app-pod -c init-db-setup
```

---

## 문제 진단을 위한 고급 기능

### 이전 컨테이너 인스턴스 로그 확인
애플리케이션이 CrashLoopBackOff 상태에 빠지거나 빈번하게 재시작되는 경우, 현재 실행 중인 컨테이너의 로그만으로는 충분하지 않을 수 있습니다. 종료된 이전 인스턴스의 로그가 실제 문제 원인을 담고 있을 가능성이 높습니다.

```bash
# 가장 최근에 종료된 컨테이너의 로그 확인
kubectl logs my-problematic-pod --previous

# 특정 컨테이너의 이전 인스턴스 로그 확인
kubectl logs my-problematic-pod -c app-container --previous
```

### 실시간 로그 스트리밍
운영 중인 애플리케이션의 동작을 실시간으로 모니터링하거나 배포 과정을 지켜볼 때 유용한 기능입니다. 이는 Unix의 `tail -f` 명령과 유사하게 작동합니다.

```bash
# 실시간 로그 스트리밍 시작
kubectl logs -f my-app-pod

# 특정 컨테이너의 실시간 로그만 스트리밍
kubectl logs -f my-app-pod -c app-container

# 스트리밍 중지: Ctrl + C
```

---

## 효율적인 로그 조회를 위한 필터링 옵션

실제 운영 환경에서는 로그 데이터가 방대할 수 있으므로, 필요한 정보에 집중할 수 있도록 다양한 필터링 옵션을 활용하는 것이 중요합니다.

```bash
# 최근 100줄만 확인
kubectl logs my-app-pod --tail=100

# 특정 시간 이후의 로그만 확인
kubectl logs my-app-pod --since=10m   # 지난 10분간의 로그
kubectl logs my-app-pod --since=2h    # 지난 2시간간의 로그

# 정확한 시간 기준으로 로그 필터링
kubectl logs my-app-pod --since-time="2024-01-15T10:30:00Z"

# 각 로그 라인에 타임스탬프 추가
kubectl logs my-app-pod --timestamps

# 출력 크기를 바이트 단위로 제한
kubectl logs my-app-pod --limit-bytes=1048576  # 1MB로 제한
```

---

## 배포 단위의 통합 로그 확인

개별 Pod 단위가 아닌 Deployment, StatefulSet, Job과 같은 컨트롤러 수준에서 로그를 확인하는 것이 실제 운영 시나리오에서 더 효과적일 수 있습니다.

### 레이블 기반 다중 Pod 로그 확인
동일한 애플리케이션의 여러 인스턴스(복제본) 로그를 한 번에 확인해야 할 때 유용합니다.

```bash
# 특정 레이블을 가진 모든 Pod의 로그 확인
kubectl logs -l app=api-service --all-containers --tail=100

# 동시 요청 수를 제한하여 클러스터 부하 관리
kubectl logs -l app=api-service --all-containers --max-log-requests=5

# 각 Pod와 컨테이너를 구분할 수 있도록 접두어 추가
kubectl logs -l app=api-service --all-containers --prefix
```

### 컨트롤러 직접 참조
컨트롤러 이름을 직접 지정하면 Kubernetes가 해당 컨트롤러가 관리하는 모든 Pod의 로그를 자동으로 조회합니다.

```bash
# Deployment 관리하는 모든 Pod의 로그 확인
kubectl logs deployment/api-deployment -c app-container --tail=100

# StatefulSet의 로그 확인
kubectl logs statefulset/database -c postgres --since=30m

# Job 실행 로그 확인
kubectl logs job/data-migration --all-containers --tail=500
```

---

## 로그의 저장, 분석 및 파이프라이닝

### 파일 저장 및 후처리
로그를 파일로 저장하여 장기 보관하거나 다른 도구로 분석할 수 있습니다.

```bash
# 로그를 파일로 저장
kubectl logs my-app-pod > application.log

# 실시간 스트리밍 로그를 파일로 저장하면서 동시에 화면에 출력
kubectl logs -f my-app-pod | tee -a live-application.log

# 에러 로그만 필터링하여 저장
kubectl logs my-app-pod | grep -E "ERROR|FATAL|Exception" > errors.log

# 여러 Pod의 로그를 하나의 파일로 통합
kubectl get pods -l app=api-service -o name \
  | xargs -I {} sh -c 'echo "=== {} ==="; kubectl logs {} --tail=200' \
  > all-api-pods.log
```

### 구조화된 로그(JSON) 처리
애플리케이션이 JSON 형식으로 구조화된 로그를 출력하는 경우, `jq` 도구를 활용하여 강력한 필터링과 변환이 가능합니다.

```bash
# 레벨이 "error"인 로그 메시지만 추출
kubectl logs my-app-pod | jq -r 'select(.level=="error") | .message'

# 특정 HTTP 상태 코드(5xx)를 가진 요청만 추출
kubectl logs my-app-pod | jq -c 'select(.status>=500) | {timestamp: .ts, endpoint: .path, status: .status}'
```

---

## 실전 운영 시나리오별 접근법

### CrashLoopBackOff 상태 빠른 진단
애플리케이션이 지속적으로 재시작되는 경우, 다음 원샷 명령으로 신속하게 문제 원인을 파악할 수 있습니다.

```bash
# 가장 최근에 생성된 Pod 식별
pod=$(kubectl get pods -l app=problematic-app --sort-by=.status.startTime | tail -n1 | awk '{print $1}')

# Pod 상세 상태 확인
kubectl describe pod "$pod"

# 이전 컨테이너의 로그 확인 (실패 원인)
kubectl logs "$pod" --previous --all-containers --tail=200
```

### 상태 점검 프로브 실패 분석
Readiness 또는 Liveness 프로브가 실패하는 경우, 애플리케이션 내부 상태를 확인해야 합니다.

```bash
# 실행 중인 Pod 선택
pod=$(kubectl get pods -l app=web-app --field-selector=status.phase=Running -o name | head -n1)

# 최근 로그 확인
kubectl logs $pod --tail=50

# 직접 엔드포인트 테스트
kubectl port-forward $pod 8080:8080 &
sleep 2
curl -f http://localhost:8080/health || echo "Health check failed"
curl -f http://localhost:8080/ready || echo "Readiness check failed"
kill %1  # 포트 포워딩 종료
```

### 배포 버전별 로그 비교
새 버전 배포 후 문제가 발생한 경우, 이전 버전과의 로그 패턴을 비교할 수 있습니다.

```bash
namespace=production
deployment=user-service

# 최신 ReplicaSet 확인
latest_rs=$(kubectl get rs -n "$namespace" \
  --selector=app=$deployment \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1:].metadata.name}')

# 이전 ReplicaSet 확인
previous_rs=$(kubectl get rs -n "$namespace" \
  --selector=app=$deployment \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-2:-1].metadata.name}')

# 두 버전의 로그 비교
echo "=== Latest version ($latest_rs) ==="
kubectl logs -n "$namespace" -l "replica-set=$latest_rs" --tail=50

echo -e "\n=== Previous version ($previous_rs) ==="
kubectl logs -n "$namespace" -l "replica-set=$previous_rs" --tail=50
```

---

## Job과 CronJob 로그 관리

Job은 한 번 실행되고 종료되는 워크로드로, 성공 또는 실패 상태로 완료됩니다. 여러 재시도를 포함할 수 있어 로그 확인 방식이 조금 다릅니다.

```bash
# Job의 모든 Pod 로그 확인
kubectl logs job/data-processing-job --all-containers --tail=200

# 실패한 Pod의 이전 실행 로그 확인
for pod in $(kubectl get pods -l job-name=data-processing-job -o name); do
  echo "=== Checking $pod ==="
  kubectl logs "$pod" --previous --all-containers --tail=100 2>/dev/null || echo "No previous logs"
done
```

CronJob의 경우 생성된 Job을 통해 로그에 접근합니다.

```bash
# CronJob이 생성한 최근 Job 확인
kubectl get jobs -l cronjob-name=daily-backup

# 특정 Job의 로그 확인
kubectl logs job/daily-backup-28374234 --tail=200
```

---

## 권한 및 접근 제어

### 다중 클러스터 및 네임스페이스 관리
여러 환경을 오가며 작업할 때는 컨텍스트와 네임스페이스를 명시적으로 지정하는 것이 좋습니다.

```bash
# 다른 네임스페이스의 로그 확인
kubectl logs -n staging my-app-pod

# 다른 Kubernetes 클러스터로 전환 후 로그 확인
kubectl config use-context production-cluster
kubectl logs -n prod my-app-pod
```

### RBAC 권한 구성
로그 조회를 위한 최소 권한을 가진 Role을 정의하려면 다음 구성을 참고할 수 있습니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-log-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
```

현재 사용자의 권한을 확인하려면:
```bash
kubectl auth can-i get pods/log -n production
```

---

## 일반적인 문제 해결 가이드

로그 확인 중 발생할 수 있는 일반적인 문제와 해결 방안을 이해하는 것이 중요합니다.

**로그가 전혀 표시되지 않을 때:**
- 컨테이너가 아직 시작되지 않았거나 즉시 종료된 경우일 수 있습니다. `kubectl describe pod`로 Pod 상태와 이벤트를 먼저 확인하세요.
- 애플리케이션이 파일 시스템에만 로그를 기록하고 stdout/stderr로 출력하지 않는 경우입니다. 애플리케이션 로깅 구성을 수정해야 합니다.

**로그가 예상보다 짧을 때:**
- 컨테이너가 너무 빨리 충돌하여 제대로된 로그를 남기지 못한 경우입니다. `--previous` 플래그로 이전 인스턴스의 로그를 확인하세요.
- 로그 회전 정책이나 리텐션 설정으로 인해 오래된 로그가 삭제되었을 수 있습니다.

**권한 관련 오류:**
- RBAC 설정이 로그 조회를 허용하지 않는 경우입니다. 클러스터 관리자에게 적절한 권한이 부여되었는지 확인하세요.

**로그 형식 문제:**
- 멀티라인 스택 트레이스가 깨져 보이는 경우, 로그 수집기(예: Fluentd, Fluent Bit)에서 멀티라인 파서를 구성해야 합니다.
- 애플리케이션이 버퍼링된 로그 출력을 사용하는 경우, 라인 버퍼링을 활성화하거나 주기적으로 flush하도록 설정해야 할 수 있습니다.

---

## `kubectl logs`의 내부 동작 이해

Kubernetes 로깅 시스템의 작동 방식을 이해하면 문제 해결에 도움이 됩니다.

1. **로그 수집 경로**: 컨테이너 런타임(containerd, Docker 등)은 각 컨테이너의 stdout/stderr을 노드의 파일 시스템(일반적으로 `/var/log/containers/`)에 기록합니다.
2. **Kubelet의 역할**: kubelet은 이러한 로그 파일을 읽고 Kubernetes API 서버를 통해 제공합니다.
3. **`kubectl logs`의 동작**: 이 명령은 API 서버를 통해 kubelet에 로그 스트림을 요청하고, kubelet은 해당 로그 파일에서 내용을 읽어 반환합니다.

중요한 점은 `kubectl logs`가 장기적인 로그 보관이나 복잡한 검색을 위한 도구가 아니라는 것입니다. 이러한 요구사항에는 Elasticsearch, Loki, Cloud 제공자의 관리형 로깅 서비스와 같은 전용 로그 수집 및 분석 스택이 필요합니다.

---

## 모범 사례와 권장사항

### 애플리케이션 로깅 원칙
1. **항상 stdout/stderr으로 로그 출력**: 파일 기반 로깅은 Kubernetes 환경에서 관리하기 어렵고 `kubectl logs`로 접근할 수 없습니다.
2. **구조화된 로깅 채택**: JSON 형식으로 로그를 출력하면 필터링, 집계, 분석이 훨씬 수월해집니다.
3. **적절한 로그 레벨 사용**: DEBUG, INFO, WARN, ERROR 레벨을 일관되게 사용하여 로그의 중요도를 나타내세요.
4. **컨텍스트 정보 포함**: 요청 ID, 사용자 ID, 세션 ID와 같은 트랜잭션 식별자를 로그에 포함시키세요.

### 운영 효율성 팁
1. **레이블 활용**: 애플리케이션 Pod에 의미 있는 레이블(app, version, component 등)을 부여하여 선택적 로그 조회를 쉽게 만드세요.
2. **로그 볼륨 관리**: 과도한 상세 로그(DEBUG 레벨)는 프로덕션 환경에서 주의해서 사용하세요.
3. **자동화 스크립트 활용**: 반복적인 로그 확인 작업은 스크립트로 자동화하여 일관성과 효율성을 높이세요.

### 보안 고려사항
1. **민감 정보 노출 금지**: 비밀번호, API 키, 개인 식별 정보는 절대로 로그에 출력하지 마세요.
2. **최소 권한 원칙**: 로그 조회 권한은 필요한 사용자에게만 제한적으로 부여하세요.
3. **감사 로그 유지**: 중요한 운영 작업에 대한 감사 로그를 별도로 유지하고 보호하세요.

---

## 빠른 참조 치트시트

```bash
# 기본 명령어
kubectl logs <pod-name>                    # 기본 Pod 로그
kubectl logs <pod> -c <container>          # 특정 컨테이너 로그
kubectl logs <pod> --previous              # 이전 컨테이너 인스턴스 로그
kubectl logs -f <pod>                      # 실시간 로그 스트리밍

# 필터링 옵션
kubectl logs <pod> --tail=50               # 최근 50줄만
kubectl logs <pod> --since=10m             # 지난 10분간 로그
kubectl logs <pod> --timestamps            # 타임스탬프 표시
kubectl logs <pod> --limit-bytes=1048576   # 출력 크기 제한(1MB)

# 집계 조회
kubectl logs -l app=api --tail=100         # 레이블로 여러 Pod 로그
kubectl logs deployment/app --tail=100     # Deployment의 모든 Pod 로그
kubectl logs --all-containers --prefix     # 모든 컨테이너 + 접두어

# 파일 저장
kubectl logs <pod> > pod.log               # 로그 파일 저장
kubectl logs -f <pod> | tee live.log       # 실시간 로그 저장 및 출력
```

---

## 결론

`kubectl logs`는 Kubernetes 환경에서 애플리케이션 문제를 신속하게 진단하고 이해하는 데 없어서는 안 될 필수 도구입니다. 기본적인 사용법부터 고급 필터링, 다중 컨테이너 관리, 배포 단위의 로그 집계까지 다양한 기능을 제공합니다.

효율적인 로그 관리는 단순히 명령어 사용법을 아는 것에서 그치지 않습니다. 애플리케이션의 로그 출력 방식을 표준화하고, 구조화된 형식을 채택하며, 적절한 로그 레벨을 사용하는 것이 더 중요합니다. 또한 `kubectl logs`는 실시간 문제 해결에 최적화되어 있으므로, 장기적인 로그 보관과 분석을 위해서는 Loki, Elasticsearch, 또는 클라우드 제공자의 관리형 로깅 서비스와 같은 전용 로깅 스택을 도입하는 것이 현명한 선택입니다.

궁극적으로 효과적인 로깅 전략은 문제 해결 시간을 단축시키고 시스템의 가시성을 높이며, 더 나은 의사 결정을 지원하는 핵심 인프라가 됩니다. 이 가이드에 제시된 패턴과 사례를 참고하여 팀의 로깅 관행을 표준화하고 운영 효율성을 극대화하시기 바랍니다.