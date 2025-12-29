---
layout: post
title: Kubernetes - Kubernetes 리소스 제한
date: 2025-05-28 19:20:23 +0900
category: Kubernetes
---
# Kubernetes 리소스 제한: CPU, Memory Requests/Limits 완전 가이드

## 기본 개념 이해

Kubernetes에서 리소스 관리는 애플리케이션의 성능, 안정성, 비용 효율성을 보장하는 핵심 요소입니다. `requests`와 `limits`의 적절한 설정은 클러스터의 안정적인 운영을 위한 기초입니다.

### Requests와 Limits의 정의
- **Requests**: 컨테이너에 보장되는 최소 리소스 양입니다. Kubernetes 스케줄러는 이 값을 기준으로 Pod를 적절한 노드에 배치합니다.
- **Limits**: 컨테이너가 사용할 수 있는 최대 리소스 양입니다. 이 값을 초과하면 CPU는 스로틀링되고, 메모리는 OOM(Out Of Memory) 킬 대상이 됩니다.

```yaml
resources:
  requests:
    cpu: "200m"      # 0.2 vCPU 보장
    memory: "256Mi"  # 256 메비바이트 보장
  limits:
    cpu: "1"         # 최대 1 vCPU까지 사용 가능
    memory: "512Mi"  # 최대 512 메비바이트까지 사용 가능
```

### 단위 이해
- **CPU**: `1` = 1 vCPU, `500m` = 0.5 vCPU, `100m` = 0.1 vCPU
- **메모리**: `Mi` = 메비바이트(2^20 바이트), `Gi` = 기비바이트(2^30 바이트)

**핵심 원리**: CPU는 공유 가능한 리소스이므로 limit을 초과하면 성능이 저하되지만 컨테이너는 계속 실행됩니다. 반면 메모리는 비공유 리소스이므로 limit을 초과하면 컨테이너가 강제 종료됩니다.

---

## QoS 클래스: 리소스 설정이 안정성에 미치는 영향

Kubernetes는 Pod의 리소스 설정 방식에 따라 세 가지 QoS(Quality of Service) 클래스를 자동으로 할당합니다. 이는 리소스 압박 상황에서 어떤 Pod가 먼저 희생되는지를 결정합니다.

### QoS 클래스 종류와 특징

**Guaranteed (최고 수준 보장)**
- 조건: 모든 컨테이너에 requests와 limits가 명시되어 있고, 두 값이 동일해야 합니다.
- 특징: 가장 강력한 보장을 받습니다. 메모리 압박 상황에서도 마지막까지 보호됩니다.
- 권장 사용처: 데이터베이스, 캐시 서버, 인증 게이트웨이 등 중요한 상태 유지 애플리케이션

**Burstable (일반적 보장)**
- 조건: 일부 리소스에만 requests와 limits가 명시되어 있거나, requests보다 limits가 큰 경우
- 특징: 필요한 경우 버스트(burst) 사용이 가능하지만, 리소스 압박 시 희생될 수 있습니다.
- 권장 사용처: 대부분의 웹 애플리케이션, 마이크로서비스

**BestEffort (최소한의 보장)**
- 조건: requests와 limits 모두 지정되지 않음
- 특징: 리소스가 충분할 때만 사용 가능하며, 압박 상황에서 가장 먼저 종료됩니다.
- 권장 사용처: 임시 작업, 테스트용 Pod (프로덕션에서는 지양)

### QoS 운영 전략
1. **핵심 서비스는 Guaranteed로**: 시스템의 핵심 구성 요소는 항상 Guaranteed QoS를 보장하세요.
2. **일반 워크로드는 Burstable로**: 대부분의 애플리케이션은 Burstable 클래스로 시작하고 데이터에 기반하여 조정하세요.
3. **BestEffort는 제한적으로**: 프로덕션 환경에서는 BestEffort Pod를 최소화하세요.

---

## CPU 리소스 관리: 스로틀링과 성능 최적화

### CPU Limits의 동작 방식
Kubernetes는 CFS(Completely Fair Scheduler) 쿼터 메커니즘을 통해 CPU limits를 구현합니다. 기본적으로 100ms 주기마다 할당된 쿼터만큼의 CPU 시간을 사용할 수 있습니다.

예를 들어, CPU limit가 0.5(`500m`)로 설정된 경우:
- 100ms 주기마다 최대 50ms의 CPU 시간 사용 가능
- 이 시간을 초과하면 나머지 주기 동안 스로틀링(throttling) 발생

### CPU 스로틀링 관찰 방법
```bash
# CPU 사용량 모니터링
kubectl top pods

# cgroup v2에서 CPU 제한 확인 (컨테이너 내에서 실행)
cat /sys/fs/cgroup/cpu.max

# 스로틀링 통계 확인
cat /sys/fs/cgroup/cpu.stat
```

### CPU 설정 권장사항
1. **지연 민감 애플리케이션**: CPU limit를 넉넉히 설정하거나, 경우에 따라 limit 없이 requests만 지정
2. **일반 웹 애플리케이션**: requests는 평균 부하 수준(200-500m), limits는 피크 부하 수준(1-2 vCPU)으로 설정
3. **마이크로서비스**: HPA와 연동하여 동적 조정

**중요**: 지나치게 낮은 CPU limit는 응답 시간 급증과 처리량 저하를 초래할 수 있습니다. 실제 부하 패턴을 모니터링하고 적절히 조정하세요.

---

## 메모리 리소스 관리: OOM 방지 전략

### 메모리 Limits와 OOM 동작
메모리 사용량이 limit을 초과하면 Linux 커널의 OOM 킬러가 컨테이너를 강제 종료합니다. 이는 다음과 같이 확인할 수 있습니다:

```bash
# Pod 상태 확인 (OOMKilled 표시)
kubectl describe pod <pod-name>

# 노드 커널 로그 확인 (필요 시)
dmesg | grep -i oom
```

### 메모리 사용량 모니터링
```bash
# 컨테이너 내에서 현재 메모리 사용량 확인
cat /sys/fs/cgroup/memory.current

# 메모리 제한 확인
cat /sys/fs/cgroup/memory.max
```

### 메모리 설정 권장사항
1. **여유 공간 포함**: limit은 평균 사용량보다 충분한 여유를 두고 설정
2. **애플리케이션 특성 고려**: 캐시, 버퍼, 힙 크기 등을 고려한 메모리 할당
3. **점진적 조정**: 모니터링 데이터 기반으로 지속적으로 최적화

---

## 네임스페이스 수준 리소스 관리

### LimitRange: 기본값 설정과 강제 적용
LimitRange를 사용하면 네임스페이스 내의 컨테이너에 대한 기본 리소스 값과 최소/최대 한도를 정의할 수 있습니다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-resource-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
```

이 구성은:
- 리소스를 명시하지 않은 컨테이너에 기본값 적용
- 컨테이너당 최소/최대 리소스 한도 강제
- 일관된 리소스 정책 적용 보장

### ResourceQuota: 총량 제한
ResourceQuota는 네임스페이스 전체의 리소스 사용량을 제한합니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "100"
```

이러한 제한은:
- 팀별 리소스 할당량 관리
- 비용 통제
- 리소스 과다 사용 방지

**운영 팁**: LimitRange와 ResourceQuota를 조합하여 네임스페이스 수준에서 종합적인 리소스 거버넌스를 구현하세요.

---

## 워크로드 유형별 리소스 설정 가이드

### 웹 API 서버 (고동시성)
- **CPU**: requests 0.25-0.5 vCPU, limits 1-2 vCPU
- **메모리**: requests 실제 사용량 + 20-30%, limits +30-50%
- **특징**: 응답 시간과 스로틀링 지표를 모니터링하여 limits 조정

### 데이터 처리/ETL 작업
- **CPU**: requests 낮게 설정, limits 높게 설정
- **메모리**: I/O 버퍼와 데이터 처리용 메모리 고려
- **특징**: Burstable QoS, 낮은 우선순위, 야간 실행 권장

### 인메모리 캐시/데이터베이스
- **CPU**: requests와 limits 동일하게 설정
- **메모리**: requests와 limits 반드시 동일하게 설정 (Guaranteed QoS)
- **특징**: OOM에 매우 취약하므로 안정성 최우선

### 머신러닝 추론 서버 (비GPU)
- **CPU**: requests 0.5-1 vCPU, limits 2-4 vCPU
- **메모리**: 모델 크기 + 배치 처리 메모리 + 여유 공간
- **특징**: 콜드 스타트 시 메모리 급증 주의

### 메시지 컨슈머
- **CPU**: requests 낮게 설정, HPA로 스케일링
- **메모리**: 메시지 버퍼 크기에 비례
- **특징**: 지연 기반 HPA와 연동 권장

---

## 자동 스케일링 시스템과의 상호작용

### Horizontal Pod Autoscaler (HPA)
HPA는 리소스 사용률을 기준으로 Pod 복제본 수를 조정합니다. 사용률 계산 공식은 다음과 같습니다:

```
사용률 = (실제 사용량) / (requests)
```

이 공식에서 중요한 점은:
- **너무 큰 requests**: 낮은 사용률로 인해 스케일아웃이 지연될 수 있음
- **너무 작은 requests**: 높은 사용률로 인해 불필요한 스케일아웃이 발생할 수 있음

### Vertical Pod Autoscaler (VPA)
VPA는 Pod의 requests와 limits를 자동으로 조정합니다. 주의사항:
- HPA와 VPA를 동시에 사용할 경우 충돌 가능성 있음
- 일반적으로 HPA(복제본 조정) + VPA(권장값만) 조합 권장
- VPA 업데이트 시 Pod 재시작 발생

### Cluster Autoscaler
Cluster Autoscaler는 스케줄링 불가능한 Pod가 있을 때 노드를 추가합니다:
- 지나치게 큰 requests 하나가 전체 클러스터 스케일링을 방해할 수 있음
- 노드 풀 전략과 requests 설정 간의 균형 중요

---

## 프로그래밍 언어별 메모리 관리 전략

### Java/JVM 애플리케이션
```bash
# cgroup 인식 JDK 사용 (8u191+, 11+)
# 컨테이너 메모리 limit 기반 힙 크기 자동 조정
-XX:+UseContainerSupport
-XX:MaxRAMPercentage=75.0  # limit의 75%를 힙으로 사용
-XX:InitialRAMPercentage=50.0  # 초기 힙 크기
```

### Go 애플리케이션
```bash
# GC 빈도 조정으로 메모리 사용 최적화
GOGC=50  # 기본값 100에서 조정
# 메모리 풀링으로 할당 감소
sync.Pool 활용
```

### Node.js 애플리케이션
```bash
# 힙 메모리 제한 설정
--max-old-space-size=4096  # 4GB 힙 제한
# 워커 스레드 활용으로 이벤트 루프 블로킹 방지
```

### Python 애플리케이션
```bash
# Gunicorn 워커 수 최적화
workers = (2 * CPU 코어 수) + 1
# 대형 데이터프레임 메모리 관리
chunksize 적절히 설정
```

---

## Ephemeral Storage 관리

임시 디스크 공간도 리소스 관리의 중요한 부분입니다. 로그 파일이나 임시 파일로 인한 디스크 고갈은 전체 노드 장애로 이어질 수 있습니다.

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"
  limits:
    ephemeral-storage: "5Gi"
```

**모범 사례**:
1. 모든 프로덕션 워크로드에 ephemeral-storage requests/limits 설정
2. 로그 로테이션 정책 구현
3. 불필요한 임시 파일 자동 정리

---

## 모니터링 및 문제 해결

### 주요 모니터링 지표
```bash
# Pod 리소스 사용량 확인
kubectl top pods --all-namespaces

# 상세 리소스 설정 확인
kubectl get pod <pod-name> -o=jsonpath='{.spec.containers[*].resources}'

# 이벤트 로그 확인
kubectl get events --sort-by=.lastTimestamp --all-namespaces
```

### Prometheus 메트릭 (권장)
- `container_cpu_usage_seconds_total`: CPU 사용량
- `container_cpu_cfs_throttled_seconds_total`: CPU 스로틀링 시간
- `container_memory_working_set_bytes`: 메모리 사용량
- `kube_pod_container_resource_requests`: 요청된 리소스
- `kube_pod_container_resource_limits`: 리소스 제한

### 일반적인 문제 진단

**문제: CPU 스로틀링으로 인한 지연 증가**
- 증상: 응답 시간 증가, `container_cpu_cfs_throttled_seconds_total` 메트릭 상승
- 해결: CPU limit 증가 또는 HPA 목표값 조정

**문제: 반복적인 OOMKilled**
- 증상: Pod 재시작, `Last State: OOMKilled` 표시
- 해결: 메모리 limit 증가, 애플리케이션 힙 설정 최적화, 메모리 누수 점검

**문제: Pod 스케줄링 실패**
- 증상: `Pending` 상태, `Insufficient memory/cpu` 이벤트
- 해결: requests 축소, 노드 풀 확장, Cluster Autoscaler 구성 확인

**문제: 노드 디스크 부족**
- 증상: `evicted: The node was low on resource: ephemeral-storage`
- 해결: ephemeral-storage limits 설정, 로그 관리 정책 강화

---

## 데이터 기반 리소스 최적화 접근법

효과적인 리소스 관리는 데이터에 기반한 의사결정이 필수적입니다. 다음 단계를 따라 최적화를 진행하세요:

### 1. 데이터 수집 단계
```bash
# 과거 부하 패턴 분석을 위한 데이터 수집
# Prometheus, Datadog, 또는 클라우드 모니터링 솔루션 활용
```

### 2. 분석 및 초기값 설정
```
평균 CPU 사용률(ū_cpu)와 p95 사용률(û_cpu) 분석
평균 메모리 사용량(ū_mem)와 p95 사용량(û_mem) 분석

초기 권장값:
CPU requests ≈ ū_cpu
CPU limits ≈ û_cpu × (1.2 ~ 1.5)

Memory requests ≈ ū_mem × (1.2 ~ 1.5)
Memory limits ≈ û_mem × (1.2 ~ 1.5)
```

### 3. 지속적인 모니터링과 조정
- 주간 단위로 성능 지표 검토
- 스로틀링, OOM, 스케줄링 실패 이벤트 추적
- 비즈니스 요구사항 변화에 따른 리소스 조정

---

## 운영 모범 사례 종합

### 설계 단계
1. **애플리케이션 프로파일링**: 개발 단계부터 리소스 사용 패턴 분석
2. **환경별 설정**: 개발, 스테이징, 프로덕션 환경에 맞는 리소스 설정
3. **문서화**: 리소스 설정 기준과 조정 프로세스 명확히 문서화

### 구현 단계
1. **모든 컨테이너에 requests 지정**: BestEffort Pod 최소화
2. **적절한 limits 설정**: 성능과 안정성 균형 유지
3. **네임스페이스 정책 적용**: LimitRange와 ResourceQuota 필수 적용

### 운영 단계
1. **지속적인 모니터링**: 주요 메트릭에 대한 대시보드 구축
2. **정기적인 검토**: 분기별 리소스 사용 패턴 분석 및 조정
3. **비용 관리**: 리소스 효율성과 비용 최적화 주기적 평가
4. **자동화**: 리소스 권장값 자동 계산 및 적용 파이프라인 구축

### 거버넌스
1. **리뷰 프로세스**: 리소스 변경에 대한 코드 리뷰 의무화
2. **정책 강제**: 어드미션 컨트롤러를 통한 리소스 정책 준수 강제
3. **감사**: 리소스 사용 패턴과 조정 내역 기록 관리

---

## 결론

Kubernetes 리소스 관리는 단순한 기술적 설정을 넘어서 비즈니스 요구사항, 성능 목표, 비용 제약 사이의 균형을 찾는 전략적 과제입니다. 효과적인 리소스 관리를 통해 다음과 같은 이점을 얻을 수 있습니다:

1. **안정성 보장**: 적절한 requests 설정으로 스케줄링 실패 방지, 적절한 limits로 OOM 및 성능 저하 예방
2. **성능 최적화**: 데이터 기반 리소스 할당으로 애플리케이션 성능 극대화
3. **비용 효율성**: 낭비되는 리소스 최소화로 인프라 비용 절감
4. **예측 가능성**: 일관된 리소스 정책으로 시스템 동작 예측성 향상

성공적인 리소스 관리는 "설정하고 잊어버리는" 일회성 작업이 아닌, 지속적인 모니터링, 분석, 최적화의 순환적 프로세스입니다. 이 가이드의 원칙과 모범 사례를 적용하여 조직의 특정 요구사항에 맞는 리소스 관리 전략을 수립하고 실행하시기 바랍니다.

리소스 관리는 Kubernetes 운영의 핵심 역량이며, 이를 효과적으로 수행하는 조직은 더 안정적이고 효율적이며 비용 효과적인 클라우드 네이티브 환경을 구축할 수 있을 것입니다.