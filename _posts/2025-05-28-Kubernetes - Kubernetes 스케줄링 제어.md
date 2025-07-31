---
layout: post
title: Kubernetes - Kubernetes 스케줄링 제어
date: 2025-05-28 20:20:23 +0900
category: Kubernetes
---
# Kubernetes 스케줄링 제어: Pod Affinity & Node Affinity

Kubernetes는 기본적으로 **스케줄러가 Pod을 자동으로 적절한 Node에 배치**합니다.  
그러나 때로는 **특정 조건**을 만족하는 노드나, **다른 Pod과 가까이 또는 떨어지도록** 배치하고 싶을 수 있습니다.

이를 위해 Kubernetes는 다음과 같은 기능을 제공합니다:

- **Node Affinity**: 특정 Node에 배치
- **Pod Affinity / Anti-Affinity**: 특정 Pod과 같은(또는 다른) Node에 배치

---

## ✅ Node Affinity란?

**노드에 정의된 라벨(Label)을 기준**으로, Pod을 특정 Node에 배치합니다.  
즉, "이런 조건의 Node에만 배포하라"는 요구사항입니다.

### 🔹 예제

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```

→ `disktype=ssd` 라벨이 있는 노드에만 Pod을 스케줄

---

### 🔹 종류

| 설정 | 의미 |
|------|------|
| `requiredDuringSchedulingIgnoredDuringExecution` | 반드시 조건을 만족해야 스케줄됨 (강제) |
| `preferredDuringSchedulingIgnoredDuringExecution` | 가능하면 조건을 만족하되, 아니어도 됨 (선호) |

---

## ✅ Pod Affinity란?

Pod이 **특정 라벨을 가진 다른 Pod이 존재하는 노드**에 배치되도록 합니다.  
즉, "특정 Pod과 **같은 노드**에 배치하라"는 의미입니다.

### 🔹 예제 (같은 노드 배치)

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: backend
        topologyKey: "kubernetes.io/hostname"
```

→ `app=backend` 라벨을 가진 Pod이 **이미 존재하는 노드(hostname 단위)**에 함께 배치

---

## ✅ Pod Anti-Affinity란?

Pod이 **특정 라벨을 가진 Pod과 다른 노드**에 배치되도록 합니다.  
즉, "특정 Pod과 **같은 노드에는 배치하지 마라**"는 의미입니다.

### 🔹 예제 (다른 노드에 분산)

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: frontend
        topologyKey: "kubernetes.io/hostname"
```

→ `app=frontend` Pod과 같은 노드에는 스케줄되지 않도록 제한

---

## ✅ topologyKey란?

`topologyKey`는 스케줄 조건을 판단할 **물리적/논리적 단위**입니다.

| key | 의미 |
|-----|------|
| `kubernetes.io/hostname` | 노드 단위 |
| `topology.kubernetes.io/zone` | 가용영역 (Zone) |
| `topology.kubernetes.io/region` | 리전 단위 |

> 예: Zone 단위 Anti-Affinity로 장애 도메인을 분리

---

## ✅ 실전 예제: Node Affinity + Pod Anti-Affinity

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd

    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: frontend
        topologyKey: "kubernetes.io/hostname"
```

→ 가능하면 `ssd` 노드에, 그리고 `app=frontend`가 없는 노드에 배치

---

## ✅ Affinity vs Taint/Toleration 차이

| 항목 | Affinity | Taints/Tolerations |
|------|----------|---------------------|
| 방향 | Pod이 노드를 "선택" | Node가 Pod을 "거부" |
| 목적 | Pod 배치 제어 (선호, 강제) | 특정 노드에 접근 제한 |
| 적용 방식 | Label 기반 필터 | Taint 기반 필터링 |

---

## ✅ 운영 전략 팁

| 전략 | 내용 |
|------|------|
| **노드 유형 분리** | GPU, SSD, ARM 노드를 라벨로 구분하고 Node Affinity 적용 |
| **고가용성 보장** | 동일 서비스의 Pod을 서로 다른 노드/Zone에 분산 (Anti-Affinity + Zone Topology) |
| **복잡도 관리** | `preferred` 조건부터 도입하고, 필요한 경우 `required`로 승격 |
| **Pod 레이블 설계** | Affinity 대상이 되는 Pod에 일관된 `label` 설정 필요 |

---

## ✅ 주의사항

- Affinity 조건을 만족하는 노드가 없으면 → Pod은 **Pending** 상태
- `required`는 강제 조건 → 클러스터에 적합한 노드가 없을 수 있음
- Pod Affinity는 **기존에 존재하는 Pod만 탐색** → 처음에 동시에 배포되는 경우 예외 발생 가능

---

## ✅ 결론 요약

| 기능 | 설명 | 사용 예 |
|------|------|---------|
| Node Affinity | 특정 라벨을 가진 노드에만 스케줄 | `disktype=ssd`, `zone=kr1-a` |
| Pod Affinity | 특정 Pod이 존재하는 노드에 함께 배치 | `backend`와 함께 배치 |
| Pod Anti-Affinity | 특정 Pod과 다른 노드에 배치 | `frontend`와 분리 배치 |
| topologyKey | 판단 기준 단위 (노드/존/리전) | 고가용성 설계에 사용 |

---

## ✅ 참고 링크

- [Node Affinity 공식 문서](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity)
- [Pod Affinity 공식 문서](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)
- [topologyKey 목록](https://kubernetes.io/docs/concepts/cluster-administration/topology-manager/)
