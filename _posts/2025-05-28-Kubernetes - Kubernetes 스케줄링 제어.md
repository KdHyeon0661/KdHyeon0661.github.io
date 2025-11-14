---
layout: post
title: Kubernetes - Kubernetes 스케줄링 제어
date: 2025-05-28 20:20:23 +0900
category: Kubernetes
---
# Kubernetes 스케줄링 제어: Pod Affinity & Node Affinity

## 큰 그림: “필터링 → 점수화 → 할당”

스케줄러는 크게 **필터링(노드 후보군)** → **점수화(우선순위)** → **바인딩(선정)** 순서로 동작합니다.
- **Node Affinity**/**(Anti-)Pod Affinity**는 *필터*와 *점수화*에 모두 관여합니다.
- `requiredDuringSchedulingIgnoredDuringExecution` = **필수 조건(필터)**
- `preferredDuringSchedulingIgnoredDuringExecution` = **선호 조건(스코어링)**

> 실행 중엔 대부분 **IgnoredDuringExecution**(실행 중 조건 변화는 강제 재스케줄 안 함).

---

## Node Affinity — “어떤 노드로 갈 수 있는가”

노드 라벨을 기준으로 Pod 배치 허용/선호를 제어합니다. 라벨은 운영팀이 **노드 특성(역할/하드웨어/영역)**을 코딩하는 수단입니다.

### 기본 구문 & 연산자

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
      - weight: 50                 # 1~100
        preference:
          matchExpressions:
          - key: node.kubernetes.io/instance-type
            operator: In
            values: ["c7i.large","c7i.xlarge"]
```

- **required** 블록의 어떤 `nodeSelectorTerms` 하나라도 만족해야 스케줄 가능(OR).
- 각 `matchExpressions`는 AND 결합.

### 운영 라벨링 패턴 예시

```bash
# 하드웨어/역할/조닝 라벨 예

kubectl label node ip-10-0-1-23 disktype=ssd tier=frontend arch=amd64

# 클라우드 표준 토폴로지 라벨(노드에 자동 부여되는 경우가 많음)
# topology.kubernetes.io/zone=ap-northeast-2a
# topology.kubernetes.io/region=ap-northeast-2

```

### 시나리오: “가능하면 SSD, 불가하면 일반 노드라도”

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

- SSD가 없을 땐 일반 노드로도 갈 수 있어 **Pending**을 피함.
- 로드/비용/가용성 균형에 유용.

---

## Pod Affinity — “이 Pod과 **가깝게**(같은 노드/존)”

서로 붙여야 하는 워크로드(예: **웹→사이드캐시**, **API→사이드카**, **분산 스토리지의 피어 근접**)에 사용합니다.

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels: { app: backend }
        topologyKey: "kubernetes.io/hostname"  # 같은 노드
```

- `topologyKey`에 따라 **노드/존/리전** 레벨 근접성을 지정합니다.

### 네임스페이스 선택(크로스-NS 인접)

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels: { app: api }
        namespaces: ["payments","shared"]      # 혹은 namespaceSelector 사용
        topologyKey: "topology.kubernetes.io/zone"
```

또는 `namespaceSelector`로 라벨 기반 NS 세트를 선택할 수도 있습니다.

---

## Pod **Anti**-Affinity — “이 Pod과 **멀리**(노드/존 분산)”

**HA 필수**인 스테이트리스 레플리카는 **서로 다른 장애 도메인(노드/존)**으로 퍼뜨려야 합니다.

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
              matchLabels: { app: frontend }  # 자기 자신 레이블
            topologyKey: "topology.kubernetes.io/zone"
```

- “**셀프 Anti-Affinity**”로 레플리카가 서로 **다른 존**으로 분산됩니다.
- 노드 단위 분산은 `kubernetes.io/hostname`.

> **주의**: `required`는 강력하지만 **노드/존 수 부족 시 Pending**. 대규모 클러스터 또는 불균형 배치 시 `preferred` 혼합을 권장.

---

## `topologyKey` 선택 가이드

| topologyKey | 의미 | 용도 |
|---|---|---|
| `kubernetes.io/hostname` | **노드** 단위 | 레플리카를 서로 다른 노드로 분산 |
| `topology.kubernetes.io/zone` | **존** 단위 | AZ 장애 대비, 비용/대기시간 절충 |
| `topology.kubernetes.io/region` | **리전** 단위 | 지진/대규모 장애 도메인 분리 |

---

## 실전 컴바인: “SSD 선호 + 같은 존 근접 + 동일 앱 간 노드 분산”

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

- **가능하면 SSD**(성능), **가능하면 cache와 같은 존**(지연 감소), **동일 앱 간 노드 분산은 강제**(HA).

---

## Affinity vs **Taints/Tolerations** — “끌림” vs “거부”

| 항목 | Affinity | Taints/Tolerations |
|---|---|---|
| 관점 | **Pod이** “이런 노드로 가고 싶다/가야 한다” | **노드가** “이 조건 없으면 오지 마” |
| 역할 | 위치 선호/제약(**끌림**) | 위치 차단(**거부**) |
| Typical | Node 라벨, Pod 라벨 기반 | GPU 전용 노드, **NoSchedule/PreferNoSchedule/NoExecute** |

### 예: GPU 전용 노드 보호

```bash
# 노드에 Taint 설정 (GPU 전용)

kubectl taint nodes gpu-node accelerator=nvidia:NoSchedule
```

```yaml
# GPU 워크로드 Pod에 Tolerations 추가

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

- **Taint**로 기본 차단, **Toleration** + **Node Affinity**로 **의도한 Pod만** 입장 허용.

---

## **Topology Spread Constraints**(현대적 분산 방법) — Anti-Affinity의 대안/보완

대규모에서 `podAntiAffinity`는 스케줄러 비용이 커질 수 있습니다. **균등 분산**에는 `topologySpreadConstraints`가 더 **확정적이고 스케일 친화적**입니다.

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule   # or ScheduleAnyway
    labelSelector:
      matchLabels: { app: frontend }
```

- **maxSkew**: 가장 많은 영역과 가장 적은 영역의 Pod 수 차이 허용치.
- **DoNotSchedule**: 규칙 위반이면 대기, **ScheduleAnyway**: 선호로 처리.

> **권장**: **HA 분산**은 Spread Constraints로, **특정 Pod 근접/회피**는 Affinity/Anti-Affinity로.

---

## 수학적 직관 — 가중 선호 점수

여러 `preferred`가 있을 때 스케줄러는 내부적으로 각 노드에 가중 점수를 합산합니다. 단순화하면:

$$
\text{score}(node) = \sum_{i=1}^{k} w_i \cdot \mathbf{1}[\text{preference}_i \text{ matches node}]
$$

- \(w_i\): `weight`(1~100)
- **required** 조건은 `\mathbf{1}[...]` 이전의 **필터 단계**에서 탈락/통과를 가름.

**운영 팁**: 여러 선호가 충돌할 때 **weights**로 우선순위를 모델링하세요.

---

## 동시 배포 함정 & 롤링 업데이트 전략

- `podAffinity`는 “이미 존재하는 Pod”를 기준으로 매칭합니다. **동시에 0→N 배포** 시 첫 파드가 없어서 매칭 실패 가능.
- 해결:
  - **선호(preferred)**로 낮춰 초기 배치 자유도 확보.
  - **스테이지드 롤아웃**(replicas를 몇 개씩 증가).
  - **PodManagementPolicy**(StatefulSet) 또는 **maxUnavailable**/`maxSurge`(Deployment) 조정.

---

## 관찰·진단·트러블슈팅

### Pending 원인 보기

```bash
kubectl describe pod <name> | sed -n '/Events:/,$p'
kubectl get events --sort-by=.lastTimestamp -A | tail
```
- “0/NN nodes are available: **node(s) didn't match node affinity** …”
- “pod didn't match pod anti-affinity rules …”
- “Insufficient cpu/memory …”

### 후보 노드 라벨 확인

```bash
kubectl get nodes --show-labels | cut -c -180
kubectl get node <node> --show-labels
```

### 시뮬레이션(드라이런) & 점검

```bash
kubectl apply -f pod.yaml --dry-run=server
kubectl scheduler extenders/플러그인 사용 시 로깅 확인
```

---

## 실전 시나리오 모음

### 프론트엔드 **서로 다른 노드 분산** + API와 **같은 존**

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

### 스토리지 노드로 **강제**(필터) + GPU 노드 **거부**

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

### 동일 앱 **존 균등 분산**(권장 현대식)

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

## 운영 체크리스트

- [ ] **라벨 표준** 수립: `tier`, `role`, `disktype`, `accelerator`, `zone/region`
- [ ] **required**를 남발하지 않기(스케줄 실패 위험↑). 선호+Spread로 먼저 설계.
- [ ] **셀프 Anti-Affinity** 또는 **Spread**로 레플리카 분산(노드/존).
- [ ] GPU/특수 노드는 **Taint**로 보호 + Affinity로 정확 매칭.
- [ ] 대규모에서 `podAntiAffinity(required)`는 비용↑ → **Spread Constraints** 검토.
- [ ] **동시 배포** 시 Affinity 매칭 부재 고려(롤링/선호 사용).
- [ ] **이벤트/라벨**을 먼저 본다(대부분 해답이 여기에).
- [ ] HPA/PodDisruptionBudget와 충돌 없는지(분산 강제 vs 축소/재시작).
- [ ] Descheduler(불균형 해소)와 정책 일관성.

---

## 테스트용 최소 재현 매니페스트(한꺼번에 실습)

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

- `frontend`는 **서로 다른 노드**로, **가능하면 backend와 같은 존**으로 간다.

---

## 자주 하는 실수 & 해결

| 증상 | 원인 | 대응 |
|---|---|---|
| 영원한 Pending | `required` 조건 과도, 라벨 불일치 | 라벨/존 수 확인, `preferred`로 완화, Spread로 대체 |
| 특정 존에 몰림 | Anti-Affinity만 사용, 자원 불균형 | `topologySpreadConstraints` 추가 |
| GPU 노드로 비GPU가 감 | Taint 부재 | GPU 노드 `NoSchedule` Taint + Toleration 관리 |
| 배포 초기에 Affinity 미매칭 | 동시 배포 & 타겟 Pod 없음 | 비율 롤아웃, 선호로 완화, 단계적 확장 |
| 스케줄러 느림 | 대규모 + 복잡한 anti-affinity | 표현 간소화, Spread 사용, 스케줄러 플러그인/파라미터 조정 |

---

## 마무리 요약

| 기능 | 핵심 | 주 사용처 |
|---|---|---|
| **Node Affinity** | 노드 라벨 기반 필터/선호 | GPU/SSD/ARM, 특정 풀 고정 |
| **Pod Affinity** | 대상 Pod **근접** | 로컬 캐시/사이드카/데이터 로컬리티 |
| **Pod Anti-Affinity** | 대상 Pod **분리** | HA 분산(노드/존) |
| **Topology Spread** | 균등 분산(스케일 친화) | 대규모, 확정적 분산 |
| **Taints/Tolerations** | 노드의 **거부** | 전용 노드 보호 |

**원칙**: “**필요 최소한만 강제(required), 나머지는 선호(preferred)로 점진적 최적화**”.
라벨 표준화와 이벤트 기반 진단이 성공의 90%입니다.

---

## 참고

- Node Affinity: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity
- Pod (Anti-)Affinity: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity
- Topology Spread: https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/
- Taints/Tolerations: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
