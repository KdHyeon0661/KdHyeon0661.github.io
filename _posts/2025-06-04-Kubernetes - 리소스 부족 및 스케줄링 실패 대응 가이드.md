---
layout: post
title: Kubernetes - 리소스 부족 및 스케줄링 실패 대응 가이드
date: 2025-06-04 19:20:23 +0900
category: Kubernetes
---
# 리소스 부족 및 스케줄링 실패 대응 가이드 (Kubernetes)

Kubernetes에서 `Pending`, `FailedScheduling`, `Insufficient CPU/Memory` 메시지는 **스케줄러가 배치할 수 있는 노드가 없을 때** 나타납니다.  

---

## 0. 3분 초진단 루틴

```bash
# 1. 파드/이벤트: 왜 못 올라가는지 스케줄러의 근거부터 본다
kubectl describe pod <pod> | sed -n '/Events:/,$p'
kubectl get events -A --sort-by='.lastTimestamp' | tail -n 50

# 2. 노드 상태/압력: 조건부 거부(Pressure, Taints)와 가용량 확인
kubectl describe node <node-name> | egrep -i 'Taints|Pressure|Allocatable|Condition'
kubectl top nodes
kubectl top pod -A

# 3. 스토리지(있다면): PVC 바인딩·프로비저닝 상태
kubectl get pvc -A
kubectl describe pvc <pvc-name>
```

출력에서 가장 빈번한 메시지:
- `0/N nodes are available: N Insufficient cpu/memory.`  
- `node(s) had taint {dedicated=infra: NoSchedule} that the pod didn't tolerate`  
- `no persistent volumes available for this claim`  
- `node(s) didn't match node selector` / `didn't match Pod affinity/anti-affinity rules`  
- `exceeded quota: ...`  

---

## 1. 증상 요약과 기본 예시

```bash
kubectl get pods

NAME            READY   STATUS    RESTARTS   AGE
myapp-xyz       0/1     Pending   0          2m
```

```bash
kubectl describe pod myapp-xyz
```

```
Warning  FailedScheduling  default-scheduler
0/3 nodes are available: 3 Insufficient memory.
```

---

## 2. 원인 분류표 (무엇이 스케줄을 막는가)

| 범주 | 대표 원인 | 스케줄러 메시지 힌트 | 첫 확인 명령 |
|---|---|---|---|
| **리소스 부족** | CPU/Memory 요청치 초과, 가용량 부족 | `Insufficient cpu/memory` | `kubectl top nodes`, `describe node` |
| **정책 제약** | Taints/Tolerations 불일치 | `had taint ... the pod didn't tolerate` | `describe node` |
| **배치 제약** | Node Affinity, Pod Affinity/Anti-Affinity 불만족 | `didn't match node selector/affinity` | `kubectl get nodes --show-labels` |
| **네임스페이스 한도** | ResourceQuota 초과 | `exceeded quota` | `kubectl describe quota -n <ns>` |
| **기본값/상한** | LimitRange 위반 | `must specify limits` / `exceeds limit` | `kubectl get limitrange -n <ns> -o yaml` |
| **스토리지** | PVC Pending/SC 미스매치/Zone 불일치 | `no persistent volumes available` | `kubectl get/describe pvc` |
| **우선순위/선점** | Preemption 필요하지만 실패 | `Preemption victims not found` | `describe pod` 이벤트 |
| **기타** | PodSecurity/PSA/OPA 차단, Admission 실패 | `forbidden`/`denied by` | `kubectl get events -A` |

---

## 3. 리소스 부족(Insufficient CPU/Memory) — 해법 설계

### 3.1 핵심 개념 복습

```yaml
resources:
  requests:   # 스케줄링 기준(최소 보장)
    cpu: "500m"
    memory: "512Mi"
  limits:     # 런타임 상한(초과 시 CPU throttling / OOMKill)
    cpu: "1"
    memory: "1Gi"
```

- **스케줄링은 requests 합계 ≤ 노드 Allocatable**이어야 성립.
- CPU는 **공유** 가능(초과 시 throttling), Memory는 **불공유**(초과 시 OOMKill).

#### 간단한 용량 추정(개념식)

메모리 여유의 충분조건(개념):
$$ \sum\_{p \in \text{node}} \text{requests\_mem}(p) + \text{requests\_mem}(\text{new}) \le \text{Allocatable\_mem(node)} $$

여러 노드에 대한 배치 가능성(개념):
$$ \exists n \in \text{Nodes} : 
\sum\_{p \in n} \text{requests\_cpu}(p) + \text{requests\_cpu}(\text{new}) \le \text{Allocatable\_cpu}(n) \land 
\sum\_{p \in n} \text{requests\_mem}(p) + \text{requests\_mem}(\text{new}) \le \text{Allocatable\_mem}(n) $$

> 위 식은 직관을 위한 개념 설명이며, 실제 스케줄러는 더 많은 필터·스코어링을 수행합니다.

### 3.2 즉시 조치(전술)

- **요청치 낮추기**: 과도한 `requests`를 실제 사용량 근처로
- **노드 증설**: 수평 확장(수동 또는 Cluster Autoscaler)
- **Pod 수 조정**: 비핵심 워크로드 축소/스케일인
- **우선순위/선점**: 중요 파드에 `priorityClassName` 부여

**요청치 조정 예시 (Kustomize patch):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1"
            memory: "512Mi"
```

### 3.3 중장기(전략)

- **HPA**: 부하에 따른 **Pod 수 자동 조절**
- **VPA**: 관측 기반 **requests 자동 추천/적용**(운영 주의)
- **Cluster Autoscaler**: 노드 풀을 자동 증감
- **SLO 기반 라이트사이징**: 관측(프로메테우스) 기반 50/95p 사용량에 여유율 적용

**HPA 예시 (CPU 60% 타깃):**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

**Cluster Autoscaler(매니지드 예)**  
- EKS/GKE/AKS에서 노드그룹/노드풀 **자동 확장** 옵션 활성화.  
- 단, 파드에 **어피니티/톨러레이션/인스턴스 타입 요구**가 있으면 해당 풀로 확장되게 설계.

---

## 4. 정책·배치 제약으로 인한 실패

### 4.1 Taints/Tolerations

```bash
kubectl describe node <node> | grep -i Taints
# 예) Taints: dedicated=infra:NoSchedule
```

**해결: Pod에 toleration 부여**
```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "infra"
  effect: "NoSchedule"
```

### 4.2 Node Affinity / Node Selector

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values: ["ssd"]
```

- **조건을 만족하는 라벨이 없으면 모두 배제 → Pending**
- 빠른 해결: 노드 라벨링
```bash
kubectl label node <node> disktype=ssd
```
- 전략: 필수(required)→선호(preferred)로 완화하여 배치 탄력성 확보.

### 4.3 Pod Affinity / Anti-Affinity

- 동일 노드/존에 묶거나 분산
- 초기 배치에서 상호 의존 파드가 **동시에 생성되면 교착**될 수 있음 → 일부를 먼저 띄우거나 규칙을 완화

예시(프런트엔드 분산):
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: frontend
      topologyKey: "kubernetes.io/hostname"
```

---

## 5. 네임스페이스 한도·기본값에 막히는 경우

### 5.1 ResourceQuota 초과

```bash
kubectl describe quota -n <ns>
```

**해결**
- 사용량 정리 / 스케일인
- Quota 상향 (플랫폼 정책과 협의)
- requests/limits 합계가 hard 한도를 넘지 않도록 라이트사이징

### 5.2 LimitRange 위반/미지정

- 일부 네임스페이스는 **requests/limits 미지정 시 거부** 또는 **자동 기본값** 적용
```bash
kubectl get limitrange -n <ns> -o yaml
```

**권장 기본값(LimitRange) 예시**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: dev
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
```

---

## 6. 스토리지(PVC)로 인한 Pending

### 6.1 상태 확인

```bash
kubectl get pvc
kubectl describe pvc <pvc>
```

**전형적 이벤트**
```
no persistent volumes available for this claim
```

### 6.2 해결 체크리스트

- **StorageClass** 이름·파라미터 일치 여부
- **동일 Zone/Topology** 제약 (AZ 맞춤) → `WaitForFirstConsumer` 사용 권장
- **수동 PV** 필요한 경우, 사이즈/접근모드/RWO·RWX 일치
- CSI 드라이버 상태:
```bash
kubectl -n kube-system get pods | grep csi
kubectl -n kube-system logs <csi-node-pod> --all-containers --tail=200
```

**StorageClass 예시(WaitForFirstConsumer):**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2-wffc
provisioner: kubernetes.io/aws-ebs
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp2
```

---

## 7. 우선순위·선점(Preemption)·PDB 상호작용

### 7.1 PriorityClass로 중요도 선언

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-tenant
value: 100000
globalDefault: false
description: "Critical tenant workloads"
```

```yaml
spec:
  priorityClassName: critical-tenant
```

- 높은 우선순위 파드는 낮은 파드를 **선점**하여 공간 확보 가능.
- 단, **PDB(PodDisruptionBudget)**가 엄격하면 선점이 차단될 수 있음.

### 7.2 PDB가 선점/드레인과 충돌하는 경우

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 90%
  selector:
    matchLabels:
      app: web
```

- 너무 빡빡하면 업그레이드·노드 드레인·선점 모두 막힌다 → **SLO에 맞는 적정치**로 조정.

---

## 8. 관측 기반 용량 추정 — 실무 팁

### 8.1 프로메테우스 지표로 requests 대비 실제 사용률 측정

- 컨테이너 CPU:
  - `rate(container_cpu_usage_seconds_total[5m])` vs. `container_spec_cpu_quota`/`container_spec_cpu_shares`
- 메모리:
  - `container_memory_working_set_bytes` vs. `container_spec_memory_limit_bytes`

### 8.2 라이트사이징(권장 휴리스틱)

- **CPU requests**: 최근 1~2주 **P95 사용률** × 안전계수(1.2~1.5)  
- **Memory requests**: 최근 **P99 피크** × 안전계수(1.1~1.3)  
- HPA가 있는 경우 **낮은 requests + 충분한 maxReplicas** 조합이 유효.

---

## 9. 케이스 스터디

### 9.1 `Insufficient memory`로 신규 릴리즈가 Pending

**상황**
- `deploy/web` 신규 이미지 배포 후 절반 이상 Pending
- 이벤트: `0/6 nodes are available: 6 Insufficient memory.`

**조치**
1) 지난 14일 메모리 사용률 P95 확인 → 180Mi 근처  
2) 현재 requests: 512Mi → 과대  
3) Kustomize로 **requests 256Mi / limits 512Mi**로 하향 패치  
4) HPA min=6, max=18로 확장 탄력 부여  
5) `kubectl rollout status` 모니터링 → 정상 완료

### 9.2 Taint된 인프라 노드로만 가능한 워크로드

**상황**
- 인프라 노드 `dedicated=infra:NoSchedule` 태인트
- 로그 에이전트 DaemonSet이 Pending

**조치**
- DaemonSet에 톨러레이션 추가:
```yaml
tolerations:
- key: dedicated
  operator: Equal
  value: infra
  effect: NoSchedule
```
- 즉시 스케줄 성공

### 9.3 PVC Pending — Zone 미스매치

**상황**
- SC `standard`는 Z1·Z2 제공, 파드는 `nodeAffinity`로 Z3 강제
- PVC 이벤트: `no PV available`

**조치**
- `WaitForFirstConsumer` SC 사용으로 파드가 어느 존으로 갈지 정해진 뒤 디스크 프로비저닝  
- 또는 Z3 지원 SC로 변경 → 해결

---

## 10. 운영 템플릿 모음

### 10.1 ResourceQuota (팀 한도 관리)
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "200"
```

### 10.2 PodTopologySpread로 균등 분산
```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: web
```

### 10.3 Kustomize로 환경별 리소스 오버레이
```yaml
# overlays/prod/patch-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests: { cpu: "300m", memory: "256Mi" }
          limits:   { cpu: "1500m", memory: "768Mi" }
```

---

## 11. 스케줄링 실패 트러블슈팅 명령 세트

```bash
# 이벤트 타임라인
kubectl get events -A --sort-by='.lastTimestamp' | tail -n 100

# 파드 디테일
kubectl describe pod <pod>
kubectl get pod <pod> -o yaml

# 노드 라벨/테인트/가용량
kubectl get nodes --show-labels
kubectl describe node <node> | sed -n '/Taints:/,/Non-terminated/p'
kubectl top nodes

# 네임스페이스 정책
kubectl get resourcequota -n <ns> -o wide
kubectl get limitrange -n <ns> -o yaml

# 스토리지
kubectl get sc
kubectl get pvc -A
kubectl describe pvc <pvc>
```

---

## 12. 베스트 프랙티스 요약

- **항상 requests 지정**: 스케줄러 정확도와 비용 최적화의 출발점
- **메모리는 보수적으로**, CPU는 HPA 전제라면 낮게 시작 후 확장
- **PDB는 SLO에 맞게**: 드레인/선점이 작동하도록 과도한 minAvailable은 지양
- **배치 제약은 선호부터**: required는 신중하게, 운영 중 교착/대기 장기화 원인
- **Autoscaler 3종( HPA/VPA/Cluster Autoscaler ) 조합**: 과금·성능·안정의 균형점
- **관측-조정 루프**: 프로메테우스·그라파나로 주기적 라이트사이징

---

## 13. 결론

| 키포인트 | 요약 |
|---|---|
| 원인 파악 | 이벤트/노드 상태/네임스페이스 정책/스토리지에서 **첫 힌트**를 잡는다 |
| 전술 조치 | 요청치 조정, 라벨·테인트 정합, 톨러레이션, Quota·LimitRange 대응 |
| 전략 설계 | HPA/VPA/Cluster Autoscaler + 우선순위·PDB로 **예측 가능한 스케줄링** |
| 운영 문화 | **관측→검증→튜닝**을 반복하고, 템플릿/오버레이로 표준화한다 |

---

## 부록 A. 예제 매니페스트 묶음

### A.1 문제 재현용 Deployment (과도한 requests)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stress-cpu
spec:
  replicas: 4
  selector:
    matchLabels: { app: stress }
  template:
    metadata:
      labels: { app: stress }
    spec:
      containers:
      - name: stress
        image: alpine
        command: ["sh","-c","i=0; while true; do i=$((i+1)); done"]
        resources:
          requests:
            cpu: "1500m"    # 의도적으로 큼
            memory: "256Mi"
          limits:
            cpu: "2000m"
            memory: "512Mi"
```

### A.2 완화된 리소스 + HPA
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests: { cpu: "200m", memory: "256Mi" }
          limits:   { cpu: "1000m", memory: "512Mi" }
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 12
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

### A.3 우선순위 + PDB(적정)
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: biz-critical
value: 90000
globalDefault: false
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: biz
spec:
  replicas: 4
  selector: { matchLabels: { app: biz } }
  template:
    metadata: { labels: { app: biz } }
    spec:
      priorityClassName: biz-critical
      containers:
      - name: app
        image: my/biz:1.0
        resources:
          requests: { cpu: "300m", memory: "512Mi" }
          limits:   { cpu: "1500m", memory: "1Gi" }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: biz-pdb
spec:
  minAvailable: 75%
  selector:
    matchLabels: { app: biz }
```

---

## 참고 링크

- Kubernetes Scheduling 개요: <https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/>
- 리소스 관리: <https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/>
- Cluster Autoscaler: <https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler>
- Pod Topology Spread Constraints: <https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/>
- PodDisruptionBudget: <https://kubernetes.io/docs/concepts/workloads/pods/disruptions/>
