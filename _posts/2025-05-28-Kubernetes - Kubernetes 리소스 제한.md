---
layout: post
title: Kubernetes - Kubernetes 리소스 제한
date: 2025-05-28 19:20:23 +0900
category: Kubernetes
---
# Kubernetes 리소스 제한 (CPU, Memory Requests/Limits)

## 0) 핵심 개념 리마인드

| 용어 | 의미 | 스케줄러/런타임 관점 |
|---|---|---|
| `requests` | **최소 보장량** | 스케줄러가 Pod를 어느 노드에 배치할지 **결정 기준** |
| `limits` | **최대 허용량** | CRI/cgroup가 상한 적용: CPU는 **스로틀링**, Memory는 **OOMKill** |

```yaml
resources:
  requests: { cpu: "200m", memory: "256Mi" }
  limits:   { cpu: "1",    memory: "512Mi" }
```

- CPU 단위: `1`=1 vCPU, `500m`=0.5 vCPU  
- Memory 단위: `Mi/Gi`(2의 거듭제곱; 1Gi=1024Mi)

> CPU는 공유 가능 ⇒ limit 초과 시 **CFS 스로틀링**.  
> Memory는 비공유 ⇒ limit 초과 시 **OOMKilled**.

---

## 1) QoS 클래스 — 왜 같은 상황에서도 어떤 Pod는 죽고, 어떤 Pod는 살아남나?

K8s는 **리소스 기술 방식**으로 Pod QoS를 결정합니다.

| QoS | 결정 조건 | 특징/우선순위 |
|---|---|---|
| **Guaranteed** | **모든 컨테이너**가 `requests==limits` **모두 명시** | 가장 강한 보장. 메모리 압박 시 **마지막까지** 보호 |
| **Burstable** | 일부 리소스만 명시/혹은 `req < limit` | 일반적. 필요 시 burst 가능, 하지만 메모리 압박 시 희생 가능 |
| **BestEffort** | requests/limits **모두 미지정** | 가장 먼저 축출(evict)/OOM 대상 |

**운영 팁**
- 정말 죽으면 안 되는 워크로드(인증, 컨트롤 플레인 근접 시스템)는 `Guaranteed`.
- 일반 트래픽 처리 서버는 `Burstable`로 시작하고 데이터 기반으로 조정.
- 임시/배치 작업은 `BestEffort` 지양(최소 요청은 넣기).

---

## 2) 스케줄링과 노드 할당 가능량(Allocatable) 계산

노드 자원은 `capacity`에서 kubelet/system-reserved/eviction-reserved가 빠진 **allocatable**만 스케줄됩니다.

```bash
kubectl describe node <node> | egrep -i 'capacity|allocatable|cpu|memory'
```

스케줄러는 **requests 합**으로만 배치 가능 여부를 판단합니다.  
limits는 배치 결정엔 영향 없음(단, CPUManager static 정책, NUMA/Toplogy Manager 등 고급 기능 예외 존재).

---

## 3) CPU — CFS 스로틀링과 실제 성능

- CPU limit는 **CFS(quota/period)**로 구현: 기본 `period=100ms`, quota=limit×period.
- 예) limit 1 vCPU → `quota=100ms`, 100ms마다 최대 100ms만 실행. 초과는 **Throttle**.
- **과도한 limit 설정**(특히 너무 낮은 limit)은 p95 지연 급증을 초래할 수 있음.

**관찰**
```bash
kubectl top pod
kubectl exec -it <pod> -- cat /sys/fs/cgroup/cpu.max         # cgroup v2
kubectl exec -it <pod> -- cat /sys/fs/cgroup/cpu.stat        # throttled_usec 등
```

**권장**
- 지연 민감 워크로드는 CPU limit를 넉넉히 주거나 **limit 미설정** 후 `requests`로만 스케줄(팀 정책 따라).
- HPA 사용 시: `requests`는 평균 부하선(예: 200~500m), `limits`는 상한(예: 1~2 vCPU).

---

## 4) Memory — 캐시와 피크, OOMKill의 본질

- Memory limit 초과 시 커널 OOM. 로그:
  - `kubectl describe pod`의 `Last State: OOMKilled`
  - 노드 `/var/log/kern.log` 또는 `dmesg`에 OOM 메시지
- 런타임/언어 런타임 튜닝 필수 (아래 §10 언어별).

**관찰**
```bash
kubectl top pod
kubectl exec -it <pod> -- cat /sys/fs/cgroup/memory.max      # cgroup v2
kubectl exec -it <pod> -- cat /sys/fs/cgroup/memory.current
```

**권장**
- 피크 여유 포함해 `limits`를 잡고, `requests`는 **평균+알파**.  
- 잦은 GC pause/OOM이 난다면 언어 런타임 힙/버퍼/스레드 재조정.

---

## 5) 네임스페이스 기본값: **LimitRange** + 용량 상한: **ResourceQuota**

### 5.1 LimitRange — 미지정 시 기본값/최대-최소 강제

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
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

### 5.2 ResourceQuota — 총합 가드레일

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-dev-budget
  namespace: dev
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "200"
```

> 팀/테넌트별 오버유스 방지. CI가 실패하며 **초과 사용을 조기 감지**.

---

## 6) 실제 배포 스니펫(Deployment)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
spec:
  replicas: 3
  selector:
    matchLabels: { app: sample }
  template:
    metadata:
      labels: { app: sample }
    spec:
      containers:
      - name: app
        image: ghcr.io/example/app:1.2.3
        ports: [{ containerPort: 8080 }]
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
```

- 평균 0.2 vCPU 보장, 최대 1 vCPU까지 burst.  
- 메모리 256Mi 보장, 512Mi 넘으면 OOM.

---

## 7) 워크로드 유형별 설정 가이드(현업 감각)

| 유형 | CPU | Memory | 비고 |
|---|---|---|---|
| 웹 API(동시성 높음) | `req=0.25~0.5`, `limit=1~2` | `req=실사용+20~30%`, `limit=+30~50%` | p95 지연/Throttle 관찰하여 limit 확장 |
| 배치/ETL | `req` 낮게, `limit` 높게 | I/O 버퍼 고려 | QoS Burstable, 우선순위 낮게, 야간 실행 |
| 캐시/인메모리 | `req≈limit` **동일** | **절대 동일(Guaranteed)** | Eviction/OOM에 매우 취약 → Guaranteed 추천 |
| ML 추론(비GPU) | `req=0.5~1`, `limit=2~4` | 모델 크기+배치 고려 | 피크/콜드스타트 시 메모리 급증 주의 |
| 메시지 컨슈머 | `req` 낮게, HPA로 scale | 메시지 버퍼에 비례 | Lag 기반 HPA(외부 메트릭) 권장 |

---

## 8) HPA/VPA/Cluster Autoscaler와의 상호작용

- **HPA**: 기본 CPU/Memory **Utilization = 실제 사용량 / requests**.  
  - 너무 큰 `requests` ⇒ 낮은 Utilization ⇒ 스케일아웃 지연  
  - 너무 작은 `requests` ⇒ 항상 과다 ⇒ 불필요한 확장
- **VPA**: 리소스 추천/자동조정(재시작 발생). HPA와 **동시 사용 시 충돌 주의**:  
  - 일반적으로 **HPA(replica)** + **VPA(recommendation-only)** 조합
- **Cluster Autoscaler**: 스케줄 불가(Pending) → 노드 증설.  
  - 지나치게 큰 `requests.memory` 하나가 **스케줄 불가**의 주범이 되기도.

---

## 9) 테스트·관측·경계값 설정

### 9.1 인위적 부하로 HPA/Throttle/OOM 재현

```bash
# 부하 Pod
kubectl run load --rm -it --image=alpine:3 -- sh
apk add --no-cache curl
while true; do curl -s http://sample-app:8080/health > /dev/null; done
```

### 9.2 Top/Describe/Events

```bash
kubectl top pod -A
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp -A
```

### 9.3 대시보드 지표(예: Prometheus)

- `container_cpu_usage_seconds_total`
- `container_cpu_cfs_throttled_seconds_total`
- `container_memory_working_set_bytes`
- `kube_pod_container_resource_requests` / `_limits`

---

## 10) 언어/런타임별 메모리/CPU 튜닝

### Go
- GOGC(기본 100). 메모리 압박 시 `GOGC=50~100` 조정.  
- `-gcflags`로 최적화 확인, 각종 버퍼 pooling.

### Java/JVM
- cgroup 인식 JDK(8u191+, 11+) 사용.  
- `-XX:MaxRAMPercentage`, `-XX:InitialRAMPercentage`로 **limit 기반** 힙 설정.  
- 예) limit 512Mi 시: `-XX:MaxRAMPercentage=60` → ~307Mi 힙.

### Node.js
- `--max-old-space-size=<MB>`로 힙 상한 명시(컨테이너 limit 근거).  
- 이벤트 루프 블로킹 방지(워커/클러스터링).

### Python
- 멀티프로세스(gunicorn)로 GIL 회피, 워커 수 = vCPU×N 공식을 실측 기반으로.  
- 대형 객체/NumPy/Pandas 캐시 주의(피크 대비 limit 여유 확보).

---

## 11) Pod·컨테이너 수명주기별 리소스

- **initContainers**도 `resources` 별도 지정 가능(초기 마이그레이션/다운로드 시 피크에 맞춰).  
- **사이드카**(예: Envoy, Fluent Bit)는 누적 오버헤드 → 반드시 리소스 명시.

```yaml
initContainers:
- name: db-migrate
  image: ghcr.io/example/migrator:1.0.0
  resources:
    requests: { cpu: "100m", memory: "128Mi" }
    limits:   { cpu: "500m", memory: "256Mi" }
```

---

## 12) 노드/커널 레벨: Eviction & OOMScore

- 노드 메모리 압박 시 kubelet이 **Soft/Hard Eviction**.  
  - `--eviction-hard=memory.available<500Mi` 등
- 동일 노드의 여러 Pod 중 **QoS/oom_score_adj**로 희생 순서 결정.

**메모리 여유 확보 전략**
- 워크로드별 `requests`를 실제 평균에 맞추고, 전체 네임스페이스에 ResourceQuota 적용.
- 캐시/버퍼를 파일시스템에만 두지 말고 애플리케이션 레벨에서 TTL/상한 설정.

---

## 13) Ephemeral Storage(임시 디스크)도 관리

로그/템프 파일로 노드 디스크 고갈 → 전체 장애.  
`ephemeral-storage` requests/limits를 반드시 넣습니다.

```yaml
resources:
  requests:
    ephemeral-storage: "256Mi"
  limits:
    ephemeral-storage: "1Gi"
```

---

## 14) 스케줄링 품질 향상 옵션(심화)

- **Topology Spread Constraints**: 노드/존 분산
- **Pod Priority & Preemption**: 더 중요한 워크로드에 자원 선점 허용
- **CPUManager (static)**: CPU 코어 pinning(지연 민감 워크로드)
- **HugePages**: ML/DB 고성능 메모리 최적화(요건 엄격)

---

## 15) 단위·표기 실수 방지

- CPU `m` vs Memory `Mi` 혼동 금지.  
  - `500m`(CPU) ↔ `512Mi`(Memory).  
- Memory `M`(SI) vs `Mi`(IEC) 주의. K8s 문맥에선 `Mi/Gi` 사용 권장.

---

## 16) 데이터 기반 설정(간단 모델)

모니터링 윈도우 \(T\) 동안
- 평균 CPU 사용률 \(\bar{u}_{cpu}\), p95 \(\hat{u}_{cpu,95}\)
- 평균 RSS \(\bar{m}\), p95 \(\hat{m}_{95}\)

권장 초깃값(웹 API 예):

$$
\text{cpu.requests} \approx \hat{u}_{cpu,50} \quad,\quad
\text{cpu.limits} \approx \hat{u}_{cpu,95}\times(1.2\sim1.5)
$$

$$
\text{mem.requests} \approx \hat{m}_{50}\times(1.2\sim1.5) \quad,\quad
\text{mem.limits} \approx \hat{m}_{95}\times(1.2\sim1.5)
$$

이후 주간 단위로 p95/Throttle/OOM 이벤트를 보고 **점진 조정**.

---

## 17) 트러블슈팅 시나리오

### 17.1 CPU가 한계인데 지연만 커지고 Pod는 안 죽음
- 증상: p95/timeout↑, `container_cpu_cfs_throttled_seconds_total`↑
- 조치: CPU limit↑ 또는 limit 제거, HPA 목표치 조정, 동시성/큐 길이 제한.

### 17.2 OOMKilled 반복
- 증상: `Last State: OOMKilled`, dmesg에 OOM 로그
- 조치: mem limit↑, 힙/버퍼 상한 설정, 캐시 축소, 누수 점검, 교차 로드테스트.

### 17.3 Pending에서 스케줄 안 됨
- 증상: `0/10 nodes are available: Insufficient memory.`
- 조치: requests 축소/분리, 노드 풀 확장, Cluster Autoscaler 확인.

### 17.4 노드 디스크 꽉 참
- 증상: `evicted: The node was low on resource: ephemeral-storage`
- 조치: `ephemeral-storage` requests/limits 도입, 로그 로테이션, 사이드카 디스크 절약.

### 17.5 팀이 디폴트 값 안 지킴
- 조치: **LimitRange + ResourceQuota** 강제, Admission(PSA/Gatekeeper/Kyverno)로 정책 준수.

---

## 18) 팀 템플릿(Helm/베이스 YAML) 제안

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
    ephemeral-storage: 256Mi
  limits:
    cpu: "1"
    memory: 512Mi
    ephemeral-storage: 1Gi
```

- 네임스페이스에는 아래 둘을 항상 배치:
  - `LimitRange`: default/defaultRequest/max/min
  - `ResourceQuota`: 팀별 총합 상한

---

## 19) 실전 점검 명령 모음

```bash
# 리소스 사용
kubectl top pod -A
kubectl top node

# 상세 자원 설정
kubectl get pod <pod> -o=jsonpath='{.spec.containers[*].resources}'
kubectl describe pod <pod> | egrep -i 'limit|request|oom|throttle'

# 이벤트/스케줄 실패 원인
kubectl get events -A --sort-by=.lastTimestamp

# cgroup v2 내부(컨테이너 쉘에서)
cat /sys/fs/cgroup/cpu.max
cat /sys/fs/cgroup/cpu.stat
cat /sys/fs/cgroup/memory.max
cat /sys/fs/cgroup/memory.current
```

---

## 20) 체크리스트 요약

- [ ] **모든 컨테이너에 최소 requests 지정** (BestEffort 금지)  
- [ ] 메모리는 OOM 방지 위해 **여유 포함 limits**  
- [ ] CPU는 지연/스루풋 균형… 스로틀링 지표 기반으로 조정  
- [ ] 네임스페이스별 **LimitRange/ResourceQuota** 강제  
- [ ] HPA 목표치와 `requests` 스케일 적합성 검증  
- [ ] 언어 런타임 힙/스레드/버퍼 상한 컨테이너 limit와 정렬  
- [ ] Ephemeral-storage도 requests/limits 지정  
- [ ] QoS(Guaranteed/Burstable) 전략적 사용(핵심 서비스는 Guaranteed)  
- [ ] 정기 리뷰: p95, Throttle, OOM, Pending, 비용

---

## 결론

| 포인트 | 요약 |
|---|---|
| 스케줄링 | `requests`의 합이 **배치 가능성**을 결정 |
| 실행상한 | CPU limit=**스로틀링**, Memory limit=**OOMKill** |
| 안정성 | QoS 클래스/LimitRange/ResourceQuota로 **계약을 코드화** |
| 확장성 | HPA/VPA/CA와 **데이터 기반**으로 상호 최적화 |
| 운영 | 언어별 튜닝·피크/캐시 관리·관측 지표로 **지속적 리밸런싱** |

**“명확한 요청, 합리적 상한, 측정 기반 조정”**  
이 세 가지만 지키면 성능·안정성·비용을 모두 잡을 수 있습니다.

---

## 참고
- Manage Resources (Kubernetes 공식): https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/  
- Pod Quality of Service: https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/  
- Troubleshoot OOM: https://kubernetes.io/docs/tasks/debug/debug-application/determine-reason-pod-failure/  
- LimitRange: https://kubernetes.io/docs/concepts/policy/limit-range/  
- ResourceQuota: https://kubernetes.io/docs/concepts/policy/resource-quotas/