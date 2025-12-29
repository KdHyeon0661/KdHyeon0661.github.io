---
layout: post
title: Kubernetes - 리소스 부족 및 스케줄링 실패 대응 가이드
date: 2025-06-04 19:20:23 +0900
category: Kubernetes
---
# Kubernetes 리소스 부족 및 스케줄링 실패 대응 가이드

## 소개: 스케줄링 실패 이해하기

Kubernetes에서 Pod가 `Pending` 상태에 머무르거나 `FailedScheduling` 이벤트를 발생시키는 것은 클러스터 스케줄러가 적절한 노드를 찾지 못했음을 의미합니다. 이러한 문제는 다양한 원인에서 발생할 수 있으며, 효과적인 해결을 위해서는 체계적인 접근이 필요합니다. 이 가이드는 스케줄링 실패의 근본 원인을 식별하고 해결하는 종합적인 방법론을 제공합니다.

## 신속 진단을 위한 3분 체크리스트

스케줄링 문제가 발생했을 때 첫 번째로 확인해야 할 항목들입니다:

```bash
# 1. Pod 상태와 이벤트 확인
kubectl describe pod <pod-name> | grep -A 20 "Events:"
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -n 50

# 2. 노드 상태 및 리소스 가용량 확인
kubectl describe node <node-name> | grep -E "Taints|Pressure|Allocatable|Conditions"
kubectl top nodes
kubectl top pods --all-namespaces

# 3. 스토리지 상태 확인 (해당하는 경우)
kubectl get pvc --all-namespaces
kubectl describe pvc <pvc-name>
```

이 명령어들은 다음과 같은 일반적인 오류 메시지를 식별하는 데 도움이 됩니다:
- `0/N nodes are available: N Insufficient cpu/memory.`
- `node(s) had taint {key:value} that the pod didn't tolerate`
- `no persistent volumes available for this claim`
- `node(s) didn't match node selector`
- `exceeded quota: ...`

## 스케줄링 실패의 주요 원인 분류

스케줄링 실패는 다음과 같은 범주로 분류할 수 있습니다:

### 1. 리소스 부족
**증상**: `Insufficient cpu` 또는 `Insufficient memory` 메시지
**원인**: 노드의 가용 리소스가 Pod의 요청 리소스보다 적음
**확인 방법**: `kubectl top nodes`, `kubectl describe node`

### 2. 정책 제약 위반
**증상**: `had taint ... the pod didn't tolerate`
**원인**: Pod가 노드의 Taint를 허용하지 않음
**확인 방법**: `kubectl describe node | grep Taints`

### 3. 배치 제약 불일치
**증상**: `didn't match node selector/affinity`
**원인**: Pod의 노드 선택기 또는 어피니티 규칙과 일치하는 노드가 없음
**확인 방법**: `kubectl get nodes --show-labels`

### 4. 네임스페이스 할당량 초과
**증상**: `exceeded quota`
**원인**: 네임스페이스의 ResourceQuota 한도를 초과
**확인 방법**: `kubectl describe quota -n <namespace>`

### 5. 스토리지 문제
**증상**: `no persistent volumes available`
**원인**: PVC에 바인딩할 수 있는 PV가 없거나 StorageClass가 일치하지 않음
**확인 방법**: `kubectl get pvc`, `kubectl describe pvc`

## Kubernetes 스케줄링의 기본 개념 재정의

스케줄링 문제를 효과적으로 해결하기 위해서는 Kubernetes의 리소스 관리 기본 개념을 이해하는 것이 중요합니다:

### Requests와 Limits의 차이
```yaml
resources:
  requests:    # 스케줄링 결정 기준 - 최소 보장 리소스
    cpu: "500m"
    memory: "512Mi"
  limits:      # 런타임 제한 - 최대 사용 가능 리소스
    cpu: "1"
    memory: "1Gi"
```

**핵심 원칙**:
- **Requests**: 스케줄러가 Pod를 배치할 때 고려하는 값입니다. 노드의 가용 리소스가 Pod의 requests 합계보다 커야 합니다.
- **Limits**: 컨테이너가 사용할 수 있는 최대 리소스입니다. 초과 시 CPU는 스로틀링되고, 메모리는 OOMKill 됩니다.

### 스케줄링의 수학적 이해 (개념적)
Pod가 노드에 스케줄되기 위해서는 다음 조건을 만족해야 합니다:

```
Σ(노드의 모든 Pod requests) + 새 Pod requests ≤ 노드 Allocatable 리소스
```

여기서 Allocatable 리소스는 노드 총 용량에서 시스템 리소스와 kubelet 예약분을 뺀 값입니다.

## 리소스 부족 문제 해결 전략

### 즉각적인 대응 조치

**1. Pod 리소스 요청량 조정**
과도한 리소스 요청을 실제 사용 패턴에 맞게 조정합니다:

```yaml
# resources-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: "200m"      # 0.2 CPU 코어
            memory: "256Mi"  # 256 메가바이트
          limits:
            cpu: "1"         # 최대 1 CPU 코어
            memory: "512Mi"  # 최대 512 메가바이트
```

적용 방법:
```bash
kubectl patch deployment web-application --patch-file resources-patch.yaml
```

**2. 노드 확장**
수동으로 노드를 추가하거나 Cluster Autoscaler를 활성화합니다:

```bash
# AWS EKS 노드 그룹 확장 예시
aws eks update-nodegroup-config \
  --cluster-name my-cluster \
  --nodegroup-name ng-1 \
  --scaling-config desiredSize=5,minSize=2,maxSize=10
```

**3. 중요도가 낮은 워크로드 축소**
비핵심 서비스의 복제본 수를 일시적으로 줄입니다:

```bash
kubectl scale deployment background-job --replicas=1
```

### 중장기 전략

**Horizontal Pod Autoscaler (HPA) 구현**
부하에 따라 Pod 수를 자동으로 조정합니다:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-application
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**Vertical Pod Autoscaler (VPA) 도입**
Pod의 리소스 요청을 실제 사용 패턴에 맞게 자동 조정합니다:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: web-application
  updatePolicy:
    updateMode: "Off"  # 추천 모드: "Off", "Initial", "Auto", "Recreate"
```

**Cluster Autoscaler 구성**
노드 풀을 자동으로 확장하여 리소스 부족 문제를 해결합니다:

```yaml
# Cluster Autoscaler 구성 예시 (GKE)
gcloud container clusters update my-cluster \
  --enable-autoscaling \
  --min-nodes=3 \
  --max-nodes=10 \
  --node-pool=default-pool
```

## 정책 및 배치 제약 문제 해결

### Taints와 Tolerations

**문제**: 노드에 Taint가 설정되어 있고 Pod가 이를 허용하지 않음
**해결**: Pod에 적절한 Toleration 추가

```yaml
# Pod 매니페스트에 추가
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  - key: "spot"
    operator: "Exists"
    effect: "PreferNoSchedule"
```

**노드 Taint 확인 및 관리**:
```bash
# 노드 Taint 확인
kubectl describe node <node-name> | grep Taints

# 노드에 Taint 추가
kubectl taint nodes <node-name> dedicated=gpu:NoSchedule

# 노드 Taint 제거
kubectl taint nodes <node-name> dedicated=gpu:NoSchedule-
```

### Node Affinity 및 Selector

**문제**: Pod의 노드 선택 조건을 만족하는 노드가 없음
**해결**: 노드 라벨링 또는 Pod 조건 완화

```yaml
# Node Affinity 예시
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - us-east-1a
          - us-east-1b
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 1
      preference:
        matchExpressions:
        - key: instance-type
          operator: In
          values:
          - m5.large
          - m5.xlarge
```

**노드 라벨 관리**:
```bash
# 노드 라벨 추가
kubectl label node <node-name> instance-type=m5.large

# 노드 라벨 확인
kubectl get nodes --show-labels

# 특정 라벨을 가진 노드 필터링
kubectl get nodes -l instance-type=m5.large
```

### Pod Affinity 및 Anti-Affinity

**문제**: Pod 배치 규칙이 충돌하여 스케줄링 실패
**해결**: 규칙 완화 또는 topologyKey 조정

```yaml
# Pod Anti-Affinity 예시 (고가용성 보장)
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: critical-service
        topologyKey: "kubernetes.io/hostname"
```

## 네임스페이스 제한 문제 해결

### ResourceQuota 초과

**문제**: 네임스페이스의 리소스 할당량 초과
**해결**: 할당량 증가 또는 리소스 사용 최적화

```bash
# 현재 ResourceQuota 상태 확인
kubectl describe resourcequota -n <namespace>

# 네임스페이스별 리소스 사용량 확인
kubectl top pods --namespace <namespace> --containers
```

**ResourceQuota 구성 예시**:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "100"
    services.loadbalancers: "5"
    persistentvolumeclaims: "20"
```

### LimitRange 위반

**문제**: Pod가 LimitRange에서 정의한 제한을 위반
**해결**: Pod 리소스 정의 수정 또는 LimitRange 조정

```yaml
# LimitRange 예시
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: default
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

## 스토리지 관련 스케줄링 문제 해결

### PVC Pending 문제

**문제**: PersistentVolumeClaim이 바인딩되지 않음
**해결**: 적절한 StorageClass 구성 및 PV 프로비저닝

```bash
# PVC 상태 확인
kubectl get pvc --all-namespaces
kubectl describe pvc <pvc-name>

# StorageClass 확인
kubectl get storageclass
kubectl describe storageclass <sc-name>
```

**최적의 StorageClass 구성**:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: optimized-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
```

**WaitForFirstConsumer의 중요성**:
이 모드는 Pod가 스케줄된 후에 볼륨을 생성하여 가용 영역 불일치 문제를 방지합니다.

## 고급 스케줄링 기법

### Pod 우선순위 및 선점

중요한 워크로드가 리소스를 확보할 수 있도록 우선순위 시스템 구성:

```yaml
# PriorityClass 정의
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-workload
value: 1000000           # 높은 값 = 높은 우선순위
globalDefault: false
description: "Critical business workload"

# Pod에 우선순위 적용
spec:
  priorityClassName: critical-workload
```

### PodDisruptionBudget (PDB) 관리

유지보수 중 가용성을 보장하면서도 스케줄링 유연성 유지:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 60%      # 최소 60% 가용성 보장
  selector:
    matchLabels:
      app: web-application
```

**PDB 설계 원칙**:
- 너무 높은 `minAvailable` 값은 스케줄링과 업그레이드를 어렵게 만듭니다
- 비즈니스 요구사항에 맞는 합리적인 값을 설정하세요
- 중요한 서비스에는 더 높은 가용성 보장을, 덜 중요한 서비스에는 더 낮은 값을 설정하세요

### Topology Spread Constraints

워크로드를 여러 장애 도메인에 균등하게 분산:

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web-application
```

## 모니터링 기반 용량 계획

### 실제 사용량 분석을 통한 리소스 최적화

**Prometheus 메트릭을 활용한 분석**:
```promql
# CPU 사용률 분석
rate(container_cpu_usage_seconds_total{container!="", pod=~"web-application-.*"}[5m])

# 메모리 사용량 분석
container_memory_working_set_bytes{container!="", pod=~"web-application-.*"}

# Requests 대비 실제 사용량 비교
# CPU
rate(container_cpu_usage_seconds_total{container!=""}[5m]) / 
container_spec_cpu_quota{container!=""} * 100

# 메모리
container_memory_working_set_bytes{container!=""} / 
container_spec_memory_limit_bytes{container!=""} * 100
```

**리소스 권장값 계산 방법**:
1. **CPU requests**: 과거 2주간 P95 사용률 × 안전 계수(1.2~1.5)
2. **Memory requests**: 과거 2주간 P99 피크 사용량 × 안전 계수(1.1~1.3)
3. **Limits**: requests × 버스트 계수(보통 2~3배)

### 용량 계획 대시보드

Grafana 대시보드를 통해 클러스터 용량을 시각적으로 모니터링:

```
클러스터 용량 대시보드 항목:
1. 노드별 리소스 사용률 (CPU, 메모리)
2. 네임스페이스별 리소스 소비
3. 리소스 요청 대비 실제 사용률
4. 스케줄링 실패 이벤트 추적
5. 예상 용량 소진 시간
```

## 운영 모범 사례

### 예방적 관리

**정기적인 용량 검토**:
- 주간: 노드 리소스 사용률 검토
- 월간: 워크로드 성장 추세 분석
- 분기별: 용량 계획 업데이트

**자동화된 알림 설정**:
```yaml
# Prometheus 경고 규칙 예시
groups:
- name: scheduling.alerts
  rules:
  - alert: HighSchedulingFailureRate
    expr: |
      increase(kube_scheduler_scheduling_attempts_total{result="unschedulable"}[5m])
      / increase(kube_scheduler_scheduling_attempts_total[5m]) > 0.1
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "High scheduling failure rate detected"
      description: "Scheduling failure rate is above 10% for 10 minutes"
```

### 문서화 및 지식 관리

**스케줄링 문제 해결 런북**:
1. 문제 식별: Pod 상태 및 이벤트 확인
2. 원인 분석: 스케줄러 메시지 해석
3. 해결 실행: 적절한 조치 적용
4. 검증: 문제 해결 확인
5. 문서화: 사례 기록 및 팀 공유

**팀 교육**:
- 정기적인 트러블슈팅 워크샵 진행
- 일반적인 스케줄링 문제 패턴 공유
- 모니터링 도구 사용법 교육

## 사례 연구: 실제 문제 해결 예시

### 사례 1: 메모리 부족으로 인한 배포 실패

**문제 상황**:
- 새 버전의 웹 애플리케이션 배포 시 50% 이상의 Pod가 Pending 상태
- 이벤트 메시지: `0/6 nodes are available: 6 Insufficient memory.`

**해결 과정**:
1. **현황 분석**: 
   ```bash
   kubectl top nodes
   kubectl describe pod <pending-pod> | grep -A 10 Events
   ```

2. **원인 발견**:
   - Pod의 메모리 requests가 512Mi로 설정됨
   - 실제 14일간 P95 메모리 사용량은 180Mi 수준
   - 노드 메모리 여유 공간 부족

3. **해결 조치**:
   ```yaml
   # 리소스 요청량 조정
   resources:
     requests:
       memory: "256Mi"  # 512Mi → 256Mi로 조정
     limits:
       memory: "512Mi"
   ```

4. **예방 조치**:
   - HPA 구성으로 자동 확장 활성화
   - 주간 모니터링으로 리소스 사용 패턴 추적

### 사례 2: Taint로 인한 DaemonSet 배포 실패

**문제 상황**:
- 로깅 에이전트 DaemonSet이 특정 노드에서 실행되지 않음
- 노드에 `dedicated=infra:NoSchedule` Taint 설정됨

**해결 과정**:
1. **노드 상태 확인**:
   ```bash
   kubectl describe node <node-name> | grep Taints
   ```

2. **DaemonSet 수정**:
   ```yaml
   spec:
     template:
       spec:
         tolerations:
         - key: "dedicated"
           operator: "Equal"
           value: "infra"
           effect: "NoSchedule"
   ```

3. **적용 및 확인**:
   ```bash
   kubectl apply -f daemonset.yaml
   kubectl get pods -n logging -o wide
   ```

## 종합적인 문제 해결 체크리스트

### 1. 초기 진단
- [ ] Pod 상태 및 이벤트 확인
- [ ] 노드 리소스 가용량 확인
- [ ] 네임스페이스 할당량 확인
- [ ] 스토리지 상태 확인

### 2. 원인 분석
- [ ] 스케줄러 실패 메시지 분석
- [ ] 리소스 사용 패턴 검토
- [ ] 정책 제약 조건 확인
- [ ] 네트워크 및 스토리지 연결 확인

### 3. 해결 실행
- [ ] 리소스 요청량 조정
- [ ] 정책 제약 완화
- [ ] 노드 확장 또는 워크로드 재배치
- [ ] 할당량 증가 요청

### 4. 예방 조치
- [ ] 모니터링 및 알림 설정
- [ ] 용량 계획 업데이트
- [ ] 문서화 및 지식 공유
- [ ] 정기적인 검토 주기 설정

## 결론

Kubernetes 스케줄링 실패는 단순한 기술적 문제를 넘어서 시스템의 건강 상태와 용량 계획의 적절성을 반영하는 지표입니다. 효과적인 대응을 위해서는 다음과 같은 원칙을 준수하는 것이 중요합니다:

### 핵심 원칙
1. **근본 원인 분석**: 증상만 해결하는 것이 아닌 근본적인 원인을 찾아 해결하세요.
2. **데이터 기반 결정**: 모니터링 데이터를 기반으로 한 객관적인 의사결정을 내리세요.
3. **자동화와 예방**: 반복적인 문제는 자동화로 해결하고, 예방 조치를 마련하세요.
4. **지속적인 학습**: 각 문제 해결 경험을 문서화하고 팀과 공유하세요.

### 성공적인 운영을 위한 체계
- **모니터링**: 실시간 모니터링과 경고 시스템 구축
- **용량 계획**: 성장 추세를 반영한 주기적인 용량 계획 수립
- **자동화**: 반복적인 작업의 자동화 구현
- **문서화**: 문제 해결 과정과 학습 내용의 체계적 문서화
- **교육**: 팀 구성원의 지속적인 기술 교육과 역량 강화

스케줄링 문제는 Kubernetes 운영에서 피할 수 없는 부분이지만, 체계적인 접근과 적절한 도구를 통해 효과적으로 관리할 수 있습니다. 이 가이드가 제공하는 방법론과 모범 사례를 활용하여 안정적이고 효율적인 Kubernetes 환경을 구축하고 유지하시기 바랍니다.

가장 중요한 것은 문제가 발생했을 때 당황하지 않고 체계적으로 접근하는 것입니다. 각 스케줄링 실패는 시스템을 더 잘 이해하고 개선할 수 있는 기회로 삼으세요. 이러한 마음가짐이 장기적으로 더 견고하고 회복력 있는 시스템을 만드는 데 기여할 것입니다.