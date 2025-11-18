---
layout: post
title: 데이터 통신 - Unicast Routing (1)
date: 2024-08-14 20:20:23 +0900
category: DataCommunication
---
# Chapter 20. Unicast Routing — General Idea, Least-Cost Routing, Distance Vector · Link State · Path Vector

## 20.1 Introduction

### General idea — 유니캐스트 라우팅이란?

**유니캐스트(unicast) 라우팅**은 “**한 출발지 → 한 목적지**”로 가는 패킷의 **경로(path)**를 결정하는 기능이다.

- **입력**: 목적지 IP 주소 (또는 prefix)
- **출력**: 어느 **다음 홉(next hop)**/인터페이스로 보낼지
- **내부**: 라우팅 알고리즘 + 라우팅 프로토콜
  - 알고리즘: 최단 경로, 메트릭 합 최소화
  - 프로토콜: 라우터끼리 라우팅 정보 교환 (RIP, OSPF, IS-IS, BGP 등)

Cisco의 최신 문서 표현을 빌리면, L3 유니캐스트 라우팅은 항상 두 가지 작업으로 요약된다.  

1. **최적 경로 결정 (optimal path selection)**  
2. **패킷 스위칭(packet switching)** — 포워딩 테이블을 보고 실제 패킷을 내보내는 작업

대부분의 교재가 다루는 내용은 ①에 해당하는 **경로 계산**과 ②를 위한 **라우팅 테이블 구축**이다.

---

### 그래프 모델

네트워크를 수학적으로 모델링하면 다음과 같다.

- **노드(node)**: 라우터 \(R_1, R_2, \dots\)
- **링크(link)**: 두 라우터 간의 연결 (물리선/논리 링크)
- **가중치(weight, cost)**: 링크를 지날 때 드는 “비용”
  - 홉 수(hop count)
  - 지연(delay)
  - 대역폭(bandwidth)의 역수
  - 관리자가 임의로 설정한 “관리 거리(administrative cost)” 등

이를 **방향성 그래프 또는 무방향 그래프**로 나타낸다.

#### 예시: 간단한 5-노드 네트워크

```text
   (2)        (1)
A ----- B -------- C
|       \         |
| (5)    \ (2)    | (3)
|         \       |
D --------- E ----+
     (1)      (4)
```

- 각 링크 옆 숫자가 **cost** (예: 지연의 근사값)라고 하자.
- 목표: A → C로 가는 **최소 비용 경로**를 찾기.

가능한 경로:

1. A–B–C  
   - 비용: \(2 + 1 = 3\)
2. A–D–E–C  
   - 비용: \(5 + 1 + 4 = 10\)
3. A–B–E–C  
   - 비용: \(2 + 2 + 4 = 8\)

**최소 비용 경로**는 A–B–C (비용 3)이다.

---

### Least-cost routing — “최단 경로”의 정확한 의미

유니캐스트 라우팅의 일반적인 목표는:

> “**각 목적지에 대해 비용 합이 최소가 되는 경로를 찾고,  
> 그 경로를 따르는 포워딩 테이블을 만들자**”

그래프 이론으로 쓰면:

- 그래프 \(G = (V, E)\), 노드 집합 \(V\), 링크 집합 \(E\)
- 각 링크 \((u,v) \in E\)에 대해, 비용 함수 \(c(u,v) > 0\)
- 한 경로 \(P = (v_0, v_1, \dots, v_k)\)의 비용:

$$
C(P) = \sum_{i=0}^{k-1} c(v_i, v_{i+1})
$$

- 주어진 출발지 \(s\)에서 목적지 \(d\)로 가는 **최단 경로**는

$$
\text{ShortestPath}(s,d) = \arg\min_{P \in \mathcal{P}(s,d)} C(P)
$$

여기서 \(\mathcal{P}(s,d)\)는 s에서 d로 가는 모든 단순 경로 집합이다.

**라우팅 알고리즘**의 역할은:

1. 각 라우터가 링크 비용 정보를 바탕으로,
2. 모든 목적지(또는 관심 있는 prefix)에 대해 위의 최적 경로를 찾아,
3. 그 결과를 **라우팅 테이블**로 정리하는 것.

---

### 메트릭(metric)의 종류

실제 프로토콜마다 “cost”를 정의하는 방법이 다르다.

| 프로토콜 | 종류          | 측정 기준 예시 |
|----------|---------------|----------------|
| RIP      | Distance Vector | 홉 수 (hop count) |
| OSPF     | Link State    | 관리자가 지정한 비용(대역폭 기반 등) |
| IS-IS    | Link State    | 링크 비용 (정수 메트릭) |
| BGP      | Path Vector   | AS-PATH 길이, Local Pref, MED 등 “정책 기반” |

- RIP은 매우 단순하게 “**홉 수**”만 센다.  
- OSPF/IS-IS는 링크 상태 DB를 전체적으로 보고 Dijkstra SPF 알고리즘으로 **최단 비용 경로**를 계산한다.  
- BGP는 비용보다는 **정책(policy)**가 우선이라, “최단 AS-PATH”는 여러 기준 중 하나일 뿐이다.  

---

### 예제 — Python으로 최단 경로 계산

단순한 그래프에 대해 “관리자가 설정한 비용”을 사용해 최단 경로를 계산해 보자.

```python
import heapq

def dijkstra(graph, source):
    # graph: {node: [(neighbor, cost), ...]}
    dist = {node: float('inf') for node in graph}
    prev = {node: None for node in graph}
    dist[source] = 0
    pq = [(0, source)]

    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]:
            continue
        for v, cost in graph[u]:
            nd = d + cost
            if nd < dist[v]:
                dist[v] = nd
                prev[v] = u
                heapq.heappush(pq, (nd, v))

    return dist, prev

def reconstruct_path(prev, dest):
    path = []
    while dest is not None:
        path.append(dest)
        dest = prev[dest]
    return list(reversed(path))

graph = {
    "A": [("B", 2), ("D", 5)],
    "B": [("A", 2), ("C", 1), ("E", 2)],
    "C": [("B", 1), ("E", 3)],
    "D": [("A", 5), ("E", 1)],
    "E": [("B", 2), ("C", 3), ("D", 1)],
}

dist, prev = dijkstra(graph, "A")
print(dist)  # 각 노드까지 최소 비용
print(reconstruct_path(prev, "C"))  # A에서 C까지 경로
```

출력 결과는 대략:

- A→C 최소 비용: 3
- 경로: `["A", "B", "C"]`

이제 이 “최단 경로” 계산에 어떤 방식으로 정보를 공유하느냐에 따라:

- **Distance Vector**  
- **Link State**  
- **Path Vector**  

세 가지 알고리즘이 구분된다.

---

## 20.2 Routing Algorithms — Distance Vector, Link State, Path Vector

### 20.2.1 Distance Vector Routing

#### 개념

**Distance Vector(DV)** 알고리즘의 핵심 아이디어:

> “각 라우터는 **모든 목적지까지의 거리(distance)와 다음 홉(next hop)**을 테이블로 갖고 있고,  
> 이 테이블을 **이웃(neighbor)**에게 주기적으로 광고한다.  
> 이웃의 distance vector를 보고 자신의 테이블을 갱신한다.”

RIP(Routing Information Protocol)은 DV 알고리즘(Bellman-Ford)에 기반한 대표적인 IGP다.  

#### 수식 — Bellman-Ford 식

노드 \(i\), 목적지 \(d\), 이웃 집합 \(N(i)\)라 할 때, DV 알고리즘은 다음 식을 따른다.

- \(D_i(d)\): “i에서 d까지의 현재 비용 추정치”
- \(c(i,j)\): i와 이웃 j 사이의 링크 비용
- 이웃 j가 알고 있는 \(D_j(d)\)

Bellman-Ford 업데이트:

$$
D_i(d) = \min_{j \in N(i)} \left\{ c(i,j) + D_j(d) \right\}
$$

- 라우터 i는 **이웃 j들의 distance vector**를 받아서 위 식을 계산,  
  더 작은 값이 나오면 자신의 테이블을 갱신.

---

#### 예제 네트워크

```text
   1          1
A --- B -------- C
      | 1
      |
      D
```

- 링크 비용:
  - A–B: 1
  - B–C: 1
  - B–D: 1
- 목적지: C

초기 상태 (자기 자신만 0, 나머지는 ∞):

| 노드 | D(X→A) | D(X→B) | D(X→C) | D(X→D) |
|------|--------|--------|--------|--------|
| A    | 0      | ∞      | ∞      | ∞      |
| B    | ∞      | 0      | ∞      | ∞      |
| C    | ∞      | ∞      | 0      | ∞      |
| D    | ∞      | ∞      | ∞      | 0      |

1단계 — 각 노드는 이웃까지의 직접 링크 비용을 알고 있으므로:

| 노드 | D(X→A) | D(X→B) | D(X→C) | D(X→D) |
|------|--------|--------|--------|--------|
| A    | 0      | 1      | ∞      | ∞      |
| B    | 1      | 0      | 1      | 1      |
| C    | ∞      | 1      | 0      | ∞      |
| D    | ∞      | 1      | ∞      | 0      |

2단계 — DV 교환 후, A가 C까지 비용 계산:

- A의 이웃: B
- B의 D(B→C) = 1
- A–B 링크 비용: 1

$$
D_A(C) = \min_{j \in \{B\}} \{ c(A,B) + D_B(C) \} = 1 + 1 = 2
$$

따라서:

- A → C 비용 = 2
- A의 다음 홉(next hop)은 B

---

#### Python으로 간단한 Distance Vector 시뮬레이션

아주 간단한 DV 업데이트 루프를 코드로 표현해 보자.

```python
INF = 10**9

# 그래프: cost[u][v] = 링크 비용 (없으면 INF)
nodes = ["A", "B", "C", "D"]
cost = {u: {v: INF for v in nodes} for u in nodes}
for u in nodes:
    cost[u][u] = 0

# 링크 설정
cost["A"]["B"] = cost["B"]["A"] = 1
cost["B"]["C"] = cost["C"]["B"] = 1
cost["B"]["D"] = cost["D"]["B"] = 1

# 초기 distance vectors: 자기 자신만 0
dist = {u: {v: INF for v in nodes} for u in nodes}
for u in nodes:
    dist[u][u] = 0

neighbors = {
    "A": ["B"],
    "B": ["A", "C", "D"],
    "C": ["B"],
    "D": ["B"],
}

def distance_vector_step():
    updated = False
    new_dist = {u: dist[u].copy() for u in nodes}
    for u in nodes:
        for d in nodes:
            # Bellman-Ford: min_j ( c(u,j) + D_j(d) )
            best = dist[u][d]
            for j in neighbors[u]:
                candidate = cost[u][j] + dist[j][d]
                if candidate < best:
                    best = candidate
            if best < new_dist[u][d]:
                new_dist[u][d] = best
                updated = True
    return new_dist, updated

# 반복하면서 수렴할 때까지
for it in range(10):
    dist, changed = distance_vector_step()
    print(f"Iteration {it}, dist from A:", dist["A"])
    if not changed:
        break
```

이 코드를 실행하면 몇 번의 반복 후 다음과 같은 값으로 수렴한다:

- `dist["A"]["C"] = 2`  
- `dist["A"]["D"] = 2`

---

#### Distance Vector의 문제점 — Count to Infinity

**Count to infinity** 문제는 DV 알고리즘의 대표적인 단점이다.  

간단한 시나리오:

```text
A ---1--- B ---1--- C
```

1. 초기에는 각 노드가 서로까지의 홉 수를 알고 있고,  
   A→C, B→C 등 모든 경로가 정상.
2. C와 B 사이 링크가 끊어진다면,  
   B는 “C까지 갈 수 없다(∞)”라고 업데이트해야 한다.
3. 하지만 A는 여전히 “B를 통해 C까지 2 홉에 갈 수 있다”고 믿고 있을 수 있다.
4. B는 A의 정보를 보고  
   “A를 통해 C까지 3 홉이네?”라고 생각하면서 **자신의 거리**를 3으로 설정.
5. A는 다시 “B가 C까지 3 홉이라니, 그럼 나는 4 홉이네?”라고 갱신…  
   이렇게 서로 “조금씩 더 나빠진 거리”를 교환하며 **거리 값이 점점 커지다가**  
   결국 무한대(예: 16) 도달 시 “도달 불가”로 취급하는 상황.

대응 기법:

- **Split Horizon**
- **Split Horizon with Poisoned Reverse**
- **Triggered Update**, **Holddown Timer** 등

RIP 표준은 이런 기법들을 적절히 조합해, convergence 속도와 안정성을 맞추고 있다.  

---

### 20.2.2 Link State Routing

#### 개념

**Link State(LS)** 알고리즘의 핵심 아이디어:

> “각 라우터는 **자신과 직접 연결된 링크 상태(link state)**를 전체 네트워크에 **플러딩(flooding)**하고,  
> 모든 라우터가 **동일한 링크 상태 데이터베이스(LSDB)**를 갖도록 한다.  
> 그 뒤 각 라우터가 **자신 로컬에서 Dijkstra SPF 알고리즘**으로 최단 경로 트리를 계산한다.”

대표적인 프로토콜:

- **OSPF (Open Shortest Path First)** — 인터넷에서 가장 널리 쓰이는 IGP  
- **IS-IS (Intermediate System to Intermediate System)** — 대형 ISP/백본망에서 많이 사용  

Link-state 프로토콜의 공통 특징:

1. **링크 상태 광고(LSA/LSP) 플러딩**
2. **각 라우터의 LSDB가 동일해야 함**
3. **SPF(Dijkstra) 알고리즘으로 최단 경로 계산**
4. **규모가 크면 영역/레벨로 나눔(OSPF area, IS-IS level)**

---

#### Link State Routing의 단계

1. **이웃 탐색 & Link-state 생성**
   - 각 라우터는 직접 연결된 인터페이스와 그 비용을 정리해 **Link State Advertisement(LSA)** 또는 **Link State PDU(LSP)**를 만든다.
2. **플러딩**
   - 이웃에게 보낸 LSA는 다시 그 이웃의 이웃들에게 전달…  
     이렇게 **네트워크 전체로 전파**된다.
3. **LSDB 구축**
   - 모든 라우터는 자신이 받은 LSAs를 모아 **동일한 LSDB(Topology Map)**를 가진다.
4. **Dijkstra SPF 실행**
   - 각 라우터는 “자신”을 루트로 하는 최단 경로 트리를 계산한다.
5. **라우팅 테이블 생성**
   - 최단 경로 트리에서 각 목적지로 가는 첫 번째 링크를 추려 **포워딩 테이블**을 만든다.

Cisco/Juniper 문서 모두 “OSPF/IS-IS는 링크 상태를 플러딩하고 SPF 알고리즘을 사용한다”고 명시하고 있다.  

---

#### 예제 — Dijkstra SPF 단계별 수행

아까의 5-노드 예제를 그대로 사용하자.

```text
   (2)        (1)
A ----- B -------- C
|       \         |
| (5)    \ (2)    | (3)
|         \       |
D --------- E ----+
     (1)      (4)
```

가정:

- 모든 라우터가 LSDB를 공유하고 있다 (LSA 플러딩 완료).
- 비용은 링크 옆 숫자.

**A를 기준(root)으로 SPF 수행**:

초기화:

- \(D(A)=0\), 나머지는 \(\infty\)
- 확정 집합 \(S = \{A\}\)

1단계 — A의 이웃 비용 설정

- 이웃: B(2), D(5)
- 현재 거리:
  - \(D(B)=2\), \(D(D)=5\), \(D(C)=\infty\), \(D(E)=\infty\)

가장 작은 D: B(2) → B를 집합 S에 추가

- \(S = \{A, B\}\)

2단계 — B를 통해 갈 수 있는 노드 갱신

- B의 이웃: A(2), C(1), E(2)
- C:
  - 기존 \(D(C)=\infty\)
  - 새로운 후보: \(D(B)+c(B,C)=2+1=3\) → D(C)=3
- E:
  - 기존 \(D(E)=\infty\)
  - 후보: \(D(B)+c(B,E)=2+2=4\) → D(E)=4
- A는 이미 S에 있음

현재 거리:
- \(D(A)=0, D(B)=2, D(C)=3, D(D)=5, D(E)=4\)

가장 작은 D 중 S에 없는 노드: C(3) → S에 추가

- \(S = \{A,B,C\}\)

3단계 — C를 통해 갱신 시도

- C의 이웃: B(1), E(3)
- E:
  - 기존 \(D(E)=4\)
  - 후보: \(D(C)+c(C,E)=3+3=6\) → 개선 안 됨

최단 거리 중 S에 없는 노드: E(4) → S에 추가

- \(S = \{A,B,C,E\}\)

4단계 — E를 통해 갱신 시도

- E의 이웃: B(2), C(3), D(1)
- D:
  - 기존 \(D(D)=5\)
  - 후보: \(D(E)+c(E,D)=4+1=5\) → 동일(동일 경로가 여러 개)

마지막 남은 D(5)를 S에 추가하면 종료.

**최종 거리/경로**

- A→C 거리: 3, 경로: A–B–C
- A→D 거리: 5, 경로: A–D 또는 A–B–E–D (비용 동일)
- A→E 거리: 4, 경로: A–B–E

---

#### Python으로 Dijkstra 구현 (라우터 관점)

```python
import heapq

def shortest_paths_from(source, graph):
    # graph: {node: [(neighbor, cost), ...]}
    dist = {node: float('inf') for node in graph}
    prev = {node: None for node in graph}
    dist[source] = 0

    pq = [(0, source)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]:
            continue
        for v, cost in graph[u]:
            nd = d + cost
            if nd < dist[v]:
                dist[v] = nd
                prev[v] = u
                heapq.heappush(pq, (nd, v))

    # 포워딩 테이블: 목적지별 첫 번째 홉
    fwd = {}
    for dest in graph:
        if dest == source:
            continue
        # prev 체인을 거꾸로 따라가서 source 직후 노드가 next hop
        path = []
        cur = dest
        while cur is not None:
            path.append(cur)
            cur = prev[cur]
        path = list(reversed(path))
        if len(path) >= 2:
            fwd[dest] = path[1]
    return dist, fwd

graph = {
    "A": [("B", 2), ("D", 5)],
    "B": [("A", 2), ("C", 1), ("E", 2)],
    "C": [("B", 1), ("E", 3)],
    "D": [("A", 5), ("E", 1)],
    "E": [("B", 2), ("C", 3), ("D", 1)],
}

distA, fwdA = shortest_paths_from("A", graph)
print("Distances from A:", distA)
print("Forwarding table at A:", fwdA)
```

결과:

- `Distances from A`는 앞에서 계산한 값과 동일.
- `Forwarding table at A`는:
  - C → B
  - D → B 또는 D (구현에 따라 하나 선택)
  - E → B

이게 라우터 A의 **포워딩 테이블**에 해당한다.

---

#### Link State vs Distance Vector

한눈에 비교:

| 항목             | Distance Vector (RIP)             | Link State (OSPF/IS-IS)                     |
|------------------|-----------------------------------|---------------------------------------------|
| 알고 있는 정보   | 목적지까지 거리(메트릭)만        | 전체 토폴로지(LSDB)                         |
| 정보 교환 대상   | 이웃에게만 DV 광고               | 모든 라우터에게 LSA 플러딩                  |
| 루프 방지        | 카운트-투-인피니티, split horizon 등 | 시퀀스 번호, LSA age, SPF 자체로 대부분 회피 |
| 수렴 속도        | 느릴 수 있음                     | 일반적으로 빠름                             |
| 규모/확장성      | 소규모/단순 네트워크              | 대규모/복잡 네트워크, area/level 지원       |

Cisco NX-OS 가이드도 “링크 상태 프로토콜(OSPF)이 일반적으로 거리 벡터보다 더 확장성이 좋다”고 요약한다.  

---

### 20.2.3 Path Vector Routing

#### 개념

**Path Vector** 알고리즘은 주로 **자율 시스템(AS)** 간 라우팅(BGP)에 사용된다.

아이디어:

> “각 라우터는 목적지까지의 **경로 전체(AS들의 시퀀스)**를 저장/광고한다.  
> 거리는 단순 숫자(메트릭)가 아니라, **경로(path) 벡터**다.”

BGP는 공식 Cisco 문서 표현 그대로 **“distance-vector + AS-PATH loop detection을 결합한 path-vector 알고리즘”**을 사용한다.  

- DV처럼 이웃과 경로 정보를 교환하지만,
- **경로 전체(예: AS1-AS3-AS5)**를 알려 주기 때문에
  - 루프를 쉽게 탐지 가능 (**자기 AS 번호가 AS-PATH에 있으면 버림**).
  - 정책 기반 라우팅(AS_PATH, Local Pref, MED 등)을 적용하기 쉬움.

---

#### 예제 — 세 개의 AS

```text
        AS 65001
      (ISP A backbone)
           |
           | BGP
           |
        AS 65002
      (Regional ISP)
           |
           | BGP
           |
        AS 65003
       (Enterprise)
```

- 목적지 prefix: `203.0.113.0/24` — AS 65003에 있음.
- 초기 광고:
  - AS 65003 → "Path: [65003], NLRI: 203.0.113.0/24"를 상위 ISP(65002)에 광고.
- 65002는 이 경로를 받아:
  - 자신의 AS를 앞에 붙여 "Path: [65002, 65003]"로 65001에 광고.
- 65001은 다시:
  - "Path: [65001, 65002, 65003]" 형태로 다른 피어들에게 광고.

이 때 각 AS는 BGP의 **경로 선택 규칙**에 따라 여러 경로 중 하나를 고른다.  
(보통 Local Pref → AS-PATH 길이 → ORIGIN → MED → eBGP/iBGP 등 순서)

---

#### Path Vector 알고리즘 개요

각 BGP 스피커는 “목적 prefix → 최선 경로(best path)”를 유지한다.

- 경로는:
  - **AS-PATH**: [AS1, AS2, AS3]
  - 기타 Path Attribute: NEXT_HOP, LOCAL_PREF, MED …

업데이트 루프(개념)를 Python 스타일 의사 코드로 쓰면:

```python
class BGPSpeaker:
    def __init__(self, asn):
        self.asn = asn
        self.adj = []  # 이웃 BGP 피어
        self.routes = {}  # {prefix: path_vector}

    def recv_update(self, prefix, path):
        # path: [ASn, ASn-1, ..., origin_AS]
        # Loop 방지: 내 AS가 path에 있으면 버림
        if self.asn in path:
            return  # discard, loop prevention

        # 내 AS를 맨 앞에 붙여 새로운 경로 만들기
        new_path = [self.asn] + path

        # 기존 경로와 비교(단순히 짧은 path를 선호한다고 가정)
        old_path = self.routes.get(prefix)
        if old_path is None or len(new_path) < len(old_path):
            self.routes[prefix] = new_path
            # 이웃들에게 다시 광고
            for peer in self.adj:
                peer.recv_update(prefix, new_path)
```

실제 BGP는 훨씬 많은 정책/속성을 고려하지만,  
핵심 아이디어는:

- **경로 전체를 벡터로 가지고**
- **자신이 이미 포함된 경로는 버림(AS-PATH loop prevention)**  
이라는 점이다.  

---

#### Path Vector vs Distance Vector

| 항목             | Distance Vector               | Path Vector (BGP)                                         |
|------------------|--------------------------------|-----------------------------------------------------------|
| 광고 내용        | 목적지까지의 거리(숫자)       | 목적지까지의 **경로 전체(AS 시퀀스)**                    |
| 루프 방지        | 카운트-투-인피니티, 보조 기법 | AS-PATH에서 **자기 AS가 보이면 폐기**                    |
| 주 사용 범위     | 내부 라우팅(IGP, 작은 규모)   | AS 간 라우팅(EGP, 전 인터넷 백본)                        |
| 최적화 기준      | 숫자 메트릭(비용 최소)        | 숫자+정책(Local Pref, MED, Community 등 복합 기준)       |
| 수렴 속도/안정성 | 네트워크 크기 증가 시 문제    | 폴리시 복잡성 때문에 수렴이 느릴 수 있으나 루프는 확실히 방지 |

현대 인터넷에서 **전 세계 AS 간 유니캐스트 라우팅**은 사실상 **BGP(path vector)**에 의해 이루어진다.  

---

## 정리

이 장에서는 **유니캐스트 라우팅**의 개념과 세 가지 대표 알고리즘을 정리했다.

1. **Introduction / Least-cost routing**
   - 라우팅은 “**최적 경로 선택 + 패킷 스위칭**”의 결합이다.  
   - 네트워크를 그래프와 비용 함수로 모델링하고,  
     각 목적지에 대해 **비용 합이 최소인 경로**를 찾는 것이 기본 목표.
   - 비용은 홉 수, 지연, 대역폭, 관리 거리, 정책 등 여러 형태가 될 수 있다.

2. **Distance Vector Routing**
   - 각 라우터는 목적지까지의 **거리(vector)**를 이웃과 교환.
   - Bellman-Ford 식  
     $$D_i(d) = \min_{j \in N(i)} \{ c(i,j) + D_j(d) \}$$  
     를 반복 적용해 수렴.
   - RIP는 DV 알고리즘 기반 IGP로, “홉 수”를 메트릭으로 사용.
   - Count-to-infinity, 루프, 느린 수렴 문제를 **split horizon, poisoned reverse, triggered updates** 등으로 완화한다.  

3. **Link State Routing**
   - 각 라우터가 **자신의 링크 상태**를 전체 네트워크에 플러딩 →  
     모든 라우터가 동일한 **LSDB(톱올로지 맵)** 보유.
   - 이후 각자 **Dijkstra SPF 알고리즘**으로 최단 경로 트리를 계산해 라우팅 테이블 생성.
   - OSPF, IS-IS가 대표적인 링크 상태 IGP이며, 대규모 네트워크에서 RIP보다 훨씬 잘 확장된다.  

4. **Path Vector Routing**
   - BGP가 사용하는 알고리즘으로, **거리 대신 전체 경로(AS-PATH)**를 벡터로 광고.
   - 루프 방지는 단순: AS-PATH에 자기 AS가 보이면 그 경로는 폐기.
   - 비용보다 **정책(Local Pref, MED, Community 등)**이 우선이며, 전 인터넷 규모의 AS 간 유니캐스트 라우팅을 담당한다.  