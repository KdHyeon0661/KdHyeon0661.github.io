---
layout: post
title: Kubernetes - Kubernetes 스케줄링 제어
date: 2025-05-28 20:20:23 +0900
category: Kubernetes
---
# Kubernetes 스케줄링 제어: Pod Affinity & Node Affinity

## 스케줄링의 기본 원리: 필터링 → 점수화 → 할당

Kubernetes 스케줄러는 파드를 적절한 노드에 배치하기 위해 세 단계의 프로세스를 거칩니다:

1. **필터링**: 사용 가능한 모든 노드 중에서 파드가 실행될 수 있는 노드 후보군을 선별합니다.
2. **점수화**: 필터링된 노드 후보군에 점수를 부여하여 최적의 노드를 선정합니다.
3. **할당**: 가장 높은 점수를 받은 노드에 파드를 바인딩합니다.

Node Affinity와 Pod Affinity는 이 프로세스의 필터링과 점수화 단계에 모두 영향을 미칩니다:

- `requiredDuringSchedulingIgnoredDuringExecution`: **필수 조건**으로, 이를 만족하지 못하는 노드는 후보군에서 제외됩니다.
- `preferredDuringSchedulingIgnoredDuringExecution`: **선호 조건**으로, 이를 만족하는 노드에 더 높은 점수를 부여합니다.

> **실행 중 처리**: 대부분의 경우 `IgnoredDuringExecution` 접미사가 붙은 설정은 파드가 이미 실행 중일 때 조건이 변경되어도 강제로 재스케줄링하지 않습니다.

---

## Node Affinity: 특정 노드에 파드 배치 제어하기

Node Affinity는 노드의 라벨을 기준으로 파드 배치를 제어합니다. 노드 라벨은 노드의 역할, 하드웨어 특성, 지리적 위치 등을 식별하는 데 사용됩니다.

### 기본 구문과 연산자

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In           # In | NotIn | Exists | DoesNotExist | Gt | Lt
            values: ["ssd"]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50                 # 1~100 사이의 가중치
        preference:
          matchExpressions:
          - key: node.kubernetes.io/instance-type
            operator: In
            values: ["c7i.large","c7i.xlarge"]
```

**주요 특징:**
- `required` 블록 내의 여러 `nodeSelectorTerms` 중 하나라도 만족하면 스케줄링이 가능합니다(OR 관계).
- 각 `matchExpressions` 내의 조건들은 모두 만족해야 합니다(AND 관계).

### 운영 환경에서의 노드 라벨링 패턴

노드 라벨은 클러스터 운영의 핵심 요소입니다. 다음과 같은 패턴으로 라벨을 구성할 수 있습니다:

```bash
# 하드웨어 특성, 역할, 아키텍처 라벨 예시
kubectl label node ip-10-0-1-23 disktype=ssd tier=frontend arch=amd64

# 클라우드 제공자 표준 토폴로지 라벨 (자동 부여되는 경우가 많음)
# topology.kubernetes.io/zone=ap-northeast-2a
# topology.kubernetes.io/region=ap-northeast-2
```

### 실전 시나리오: SSD 노드 선호, 불가능하면 일반 노드

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd"]
```

이 설정은 SSD 노드를 선호하지만, SSD 노드가 없을 경우 일반 노드로도 스케줄링될 수 있어 파드가 Pending 상태에 빠지는 것을 방지합니다. 로드 밸런싱, 비용 최적화, 가용성 유지에 유용한 접근 방식입니다.

---

## Pod Affinity: 특정 파드와 근접하게 배치하기

Pod Affinity는 특정 파드와 가까운 위치(동일 노드, 동일 가용성 영역 등)에 파드를 배치하고자 할 때 사용합니다. 웹 서버와 로컬 캐시, API 서버와 사이드카 컨테이너, 분산 스토리지 시스템의 피어 노드처럼 서로 밀접하게 연동되는 워크로드에 적합합니다.

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels: { app: backend }
        topologyKey: "kubernetes.io/hostname"  # 동일 노드에 배치
```

- `topologyKey`: 파드 근접성을 결정하는 기준이 되는 토폴로지 도메인을 지정합니다. 노드, 가용성 영역, 리전 등 다양한 수준에서 근접성을 정의할 수 있습니다.

### 네임스페이스 간 파드 근접성 제어

서로 다른 네임스페이스에 있는 파드들 사이에도 근접성을 제어할 수 있습니다:

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels: { app: api }
        namespaces: ["payments","shared"]      # 특정 네임스페이스 지정
        topologyKey: "topology.kubernetes.io/zone"
```

`namespaceSelector`를 사용하면 라벨 기반으로 네임스페이스 그룹을 선택할 수도 있습니다.

---

## Pod Anti-Affinity: 특정 파드와 분리하여 배치하기

Pod Anti-Affinity는 고가용성이 중요한 스테이트리스 레플리카를 서로 다른 장애 도메인(노드, 가용성 영역)에 분산 배치할 때 사용합니다.

```yaml
spec:
  replicas: 3
  template:
    metadata:
      labels: { app: frontend }
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels: { app: frontend }  # 자기 자신과의 분리
            topologyKey: "topology.kubernetes.io/zone"
```

이 설정은 동일한 앱의 레플리카들이 서로 다른 가용성 영역에 배치되도록 강제합니다. 노드 수준 분산이 필요하다면 `kubernetes.io/hostname`을 사용합니다.

> **주의사항**: `required` 설정은 강력한 제약이지만, 충분한 노드나 영역이 없을 경우 파드가 Pending 상태에 빠질 수 있습니다. 대규모 클러스터나 불균형한 배치 환경에서는 `preferred` 설정을 혼합 사용하는 것이 좋습니다.

---

## topologyKey 선택 가이드

| topologyKey | 의미 | 사용 사례 |
|---|---|---|
| `kubernetes.io/hostname` | **노드** 단위 분리/근접 | 레플리카를 서로 다른 노드로 분산 배치 |
| `topology.kubernetes.io/zone` | **가용성 영역** 단위 분리/근접 | AZ 장애 대비, 비용과 대기 시간 절충 |
| `topology.kubernetes.io/region` | **리전** 단위 분리/근접 | 지진이나 대규모 장애에 대비한 도메인 분리 |

---

## 실전 조합 예제: 성능, 근접성, 고가용성 통합

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 4
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api, tier: backend }
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 70
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 30
            podAffinityTerm:
              labelSelector:
                matchLabels: { app: cache }
              topologyKey: topology.kubernetes.io/zone
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels: { app: api }
            topologyKey: kubernetes.io/hostname
```

이 설정은 다음과 같은 복합적인 목표를 달성합니다:
- **성능**: 가능하면 SSD 노드에 배치 (70% 가중치)
- **지연 최적화**: 가능하면 캐시와 동일 가용성 영역에 배치 (30% 가중치)
- **고가용성**: 동일 앱의 파드들을 서로 다른 노드에 강제 분산

---

## Affinity와 Taints/Tolerations 비교: 끌어당김 vs 차단

Kubernetes에서는 파드 배치를 제어하는 두 가지 상호보완적인 메커니즘이 있습니다:

| 특성 | Affinity | Taints/Tolerations |
|---|---|---|
| 관점 | **파드**가 "이런 노드로 가고 싶다" | **노드**가 "이 조건 없으면 오지 마" |
| 역할 | 위치 선호 및 제약(**끌어당김**) | 위치 차단(**거부**) |
| 일반적 사용 사례 | 노드 라벨, 파드 라벨 기반 배치 | GPU 전용 노드, 특수 목적 노드 보호 |

### GPU 전용 노드 보호 예제

```bash
# GPU 노드에 Taint 설정 (GPU 전용으로 지정)
kubectl taint nodes gpu-node accelerator=nvidia:NoSchedule
```

```yaml
# GPU 워크로드 파드에 Toleration과 Affinity 설정
spec:
  tolerations:
  - key: "accelerator"
    operator: "Equal"
    value: "nvidia"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: accelerator
            operator: In
            values: ["nvidia"]
```

이 접근 방식은 Taint로 기본적인 접근을 차단한 후, Toleration과 Node Affinity를 조합하여 의도된 파드만 해당 노드에 배치되도록 제어합니다.

---

## Pod Anti-Affinity의 대안: Topology Spread Constraints

대규모 클러스터에서 Pod Anti-Affinity는 스케줄러 성능에 부정적인 영향을 줄 수 있습니다. **균등 분산**을 위해서는 `topologySpreadConstraints`가 더 예측 가능하고 확장성이 좋은 대안입니다.

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule   # 또는 ScheduleAnyway
    labelSelector:
      matchLabels: { app: frontend }
```

**주요 매개변수:**
- `maxSkew`: 가장 많은 파드를 가진 토폴로지 도메인과 가장 적은 파드를 가진 도메인 간의 최대 차이
- `whenUnsatisfiable`: 제약 조건을 만족할 수 없을 때의 동작 (`DoNotSchedule` 또는 `ScheduleAnyway`)

> **권장 사항**: 고가용성을 위한 분산에는 Topology Spread Constraints를, 특정 파드와의 근접성이나 회피가 필요할 때는 Affinity/Anti-Affinity를 사용하세요.

---

## 가중 선호 점수 계산 방식

여러 `preferred` 조건이 있을 때 스케줄러는 각 노드에 대해 가중 점수를 계산합니다. 간단히 표현하면 다음과 같은 방식으로 작동합니다:

```
노드 점수 = Σ(조건_i의 가중치 × 조건_i_만족_여부)
```

- 조건_i의 가중치: `weight` 값 (1~100)
- 조건_i_만족_여부: 조건을 만족하면 1, 아니면 0

**운영 팁**: 여러 선호 조건이 상충될 때는 `weight` 값을 조정하여 우선순위를 명확히 표현하세요.

---

## 동시 배포 시 주의사항과 롤링 업데이트 전략

Pod Affinity는 **이미 존재하는 파드**를 기준으로 매칭합니다. 따라서 처음으로 배포할 때(0 → N 개의 레플리카) 첫 번째 파드가 없어 매칭에 실패할 수 있습니다.

**해결 방안:**
- 선호(`preferred`) 조건으로 완화하여 초기 배치 유연성 확보
- 단계적 롤아웃: 레플리카를 점진적으로 증가시키는 방식 적용
- 배포 전략 조정: `maxUnavailable`과 `maxSurge` 값을 적절히 설정

---

## 모니터링, 진단, 문제 해결

### Pending 상태 원인 분석

```bash
# 파드 이벤트 확인
kubectl describe pod <파드-이름> | sed -n '/Events:/,$p'

# 클러스터 전체 이벤트 확인
kubectl get events --sort-by=.lastTimestamp -A | tail
```

주요 에러 메시지 패턴:
- "0/NN nodes are available: **node(s) didn't match node affinity** …"
- "pod didn't match pod anti-affinity rules …"
- "Insufficient cpu/memory …"

### 노드 라벨 확인

```bash
# 모든 노드의 라벨 확인
kubectl get nodes --show-labels | cut -c -180

# 특정 노드의 라벨 상세 확인
kubectl get node <노드-이름> --show-labels
```

### 사전 검증

```bash
# 실제 적용 없이 스케줄링 가능성 테스트
kubectl apply -f pod.yaml --dry-run=server
```

---

## 실전 시나리오 모음

### 시나리오 1: 프론트엔드 노드 분산 + API 동일 영역 배치

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector: { matchLabels: { app: fe } }
        topologyKey: kubernetes.io/hostname
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        podAffinityTerm:
          labelSelector: { matchLabels: { app: api } }
          topologyKey: topology.kubernetes.io/zone
```

### 시나리오 2: GPU 노드 배제 + NVME 고가용성 스토리지 요구

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: storagetier
            operator: In
            values: ["nvme-ha"]
        - matchExpressions:
          - key: accelerator
            operator: DoesNotExist
```

### 시나리오 3: 현대적인 영역 균등 분산 (권장)

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels: { app: svc-a }
```

---

## 테스트용 실습 매니페스트

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sched-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: sched-demo
spec:
  replicas: 2
  selector:
    matchLabels: { app: backend }
  template:
    metadata:
      labels: { app: backend }
    spec:
      containers:
      - name: app
        image: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: sched-demo
spec:
  replicas: 3
  selector:
    matchLabels: { app: frontend }
  template:
    metadata:
      labels: { app: frontend }
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 50
            podAffinityTerm:
              labelSelector:
                matchLabels: { app: backend }
              topologyKey: topology.kubernetes.io/zone
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels: { app: frontend }
            topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: nginx
```

이 예제에서 `frontend` 파드들은 다음과 같은 규칙으로 배치됩니다:
- 서로 다른 노드에 강제 분산 배치
- 가능한 경우 `backend` 파드와 동일한 가용성 영역에 배치 선호

---

## 자주 발생하는 문제와 해결 방안

| 증상 | 원인 | 해결 방안 |
|---|---|---|
| 영원한 Pending 상태 | `required` 조건이 너무 엄격하거나 라벨 불일치 | 라벨과 영역 수 확인, `preferred`로 완화, Spread Constraints로 대체 |
| 특정 영역에 파드가 집중됨 | Anti-Affinity만 사용하고 자원 불균형 존재 | Topology Spread Constraints 추가 적용 |
| GPU 노드에 비GPU 워크로드 배치 | Taint가 설정되지 않음 | GPU 노드에 `NoSchedule` Taint 설정 및 Toleration 관리 |
| 초기 배포 시 Affinity 조건 미충족 | 동시 배포 시 대상 파드가 존재하지 않음 | 단계적 롤아웃, 선호 조건 사용, 점진적 확장 |
| 스케줄러 성능 저하 | 대규모 클러스터 + 복잡한 anti-affinity 규칙 | 규칙 단순화, Spread Constraints 사용, 스케줄러 파라미터 조정 |

---

## 결론

Kubernetes의 스케줄링 제어 메커니즘은 복잡한 배치 요구사항을 효율적으로 관리할 수 있는 강력한 도구들을 제공합니다. 각 메커니즘의 특성과 적절한 사용 시나리오를 이해하는 것이 중요합니다:

| 기능 | 핵심 개념 | 주요 사용 사례 |
|---|---|---|
| **Node Affinity** | 노드 라벨 기반 필터링/선호 | GPU/SSD/ARM 특화 노드, 특정 노드 풀 고정 |
| **Pod Affinity** | 대상 파드와의 **근접성** | 로컬 캐시/사이드카, 데이터 지역성 최적화 |
| **Pod Anti-Affinity** | 대상 파드와의 **분리** | 고가용성을 위한 노드/영역 분산 |
| **Topology Spread Constraints** | 균등 분산 (확장성 우수) | 대규모 클러스터, 예측 가능한 분산 배치 |
| **Taints/Tolerations** | 노드의 **접근 제한** | 전용 노드 보호, 특수 목적 노드 관리 |

효율적인 스케줄링 정책 수립을 위한 핵심 원칙은 **"필요한 최소한만 강제(required)하고, 나머지는 선호(preferred)로 점진적 최적화"** 입니다. 라벨 표준화와 이벤트 기반 진단은 성공적인 스케줄링 정책 구현의 90%를 차지합니다.

실제 운영 환경에서는 이러한 메커니즘들을 조합하여 성능, 비용, 가용성, 보안 요구사항을 균형 있게 충족하는 배치 전략을 수립해야 합니다. 각 애플리케이션의 특성과 비즈니스 요구사항에 맞춰 적절한 스케줄링 정책을 설계하고, 지속적으로 모니터링하며 최적화하는 것이 Kubernetes 클러스터 운영의 성공 비결입니다.