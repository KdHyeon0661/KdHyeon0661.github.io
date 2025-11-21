---
layout: post
title: 데이터 통신 - Peer-to-Peer Paradigm (2)
date: 2024-09-25 21:20:23 +0900
category: DataCommunication
---
# Chapter 29 — Peer-to-Peer Overlays: Pastry, Kademlia, BitTorrent

Pastry·Kademlia는 각각 **prefix 기반 / XOR 기반** DHT 설계의 대표 사례이고,
BitTorrent는 **파일 배포 P2P + DHT(트래커리스 확장)** 의 사실상 표준이다.

---

## Pastry

Pastry는 Rowstron & Druschel(Microsoft Research, 2001)이 제안한 **대규모 P2P 오버레이 라우팅·객체 위치 서비스**이다.
핵심은 다음 세 가지다.

- 모든 노드와 키는 **같은 식별자 공간(identifier space)** 을 공유한다.
- **prefix 기반 라우팅**으로, 각 홉에서 **공통 prefix 길이를 1자리씩 늘려가며** 목적지에 접근한다.
- **Leaf set, Routing table, Neighborhood set**의 세 구조로 **효율·근접성·탄력성**을 동시에 얻는다.

### Identifier Space

Pastry의 식별자 공간은 보통 **128비트 혹은 160비트** 크기의 **순환 공간(ring)** 으로 본다.
각 노드에는 **nodeId**, 각 객체(키)에는 **keyId** 가 할당되고, 두 값 모두 **같은 공간**에서 뽑힌다.

- 보통 **SHA-1, SHA-256의 일부**와 같은 해시 함수를 사용해:
  - 노드의 IP:Port → nodeId
  - 객체 이름(파일 해시 등) → keyId
- 이 공간은 **0 ~ 2^b − 1** 사이의 정수로 볼 수 있다.

예를 들어, b = 16이고, 우리는 **base 4**(2비트씩)로 표현한다고 하자.

| 이진 | base-4 (Pastry digit) | 의미 |
|------|-----------------------|------|
| 0000 | 00                    | 노드/키 ID의 상위 두 비트 |
| 0101 | 11                    | 상위 두 비트 01, 다음 두 비트 01 |

Pastry는 일반적으로 **base 2^b**(예: b = 4 → base 16)에서 digit을 다룬다. 한 digit이 **라우팅 테이블의 한 row**에 해당한다.

#### Leaf set, Routing table, Neighborhood set

각 노드는 다음 세 가지 구조를 유지한다.

1. **Leaf set L**
   - nodeId 기준으로 **양 옆에 있는 L/2개 노드** 목록.
   - 키가 현재 nodeId 근처라면, leaf set 안에서 직접 찾아갈 수 있다.
   - 작은 범위에서의 **정확한 라우팅 및 중복**을 담당.

2. **Routing table R**
   - prefix 길이 1, 2, 3, ... 에 대해, **해당 prefix를 공유하는 노드**를 한 칸씩 유지.
   - 각 row는 `base` 개의 엔트리를 가질 수 있다. (자기 digit에 해당하는 칸은 비움)
   - 라우팅 시, 현재 노드와 목적지 keyId의 **공통 prefix가 1자리씩 길어지도록** 다음 홉을 선택.

3. **Neighborhood set M**
   - 실제 IP 네트워크 상에서 **지리적/지연(latency) 측면에서 가까운 노드** 모음.
   - overlay 상의 논리적 위치와 별개로, **물리적 근접성**을 고려한 최적화.

간단한 예:

- base 16, 4자리 ID (0x0000 ~ 0xFFFF)
- 내 nodeId = 3AF2, 목적 keyId = 3B91

- 공통 prefix: `3`
- 다음 digit: 내 nodeId는 `A`, 목적지는 `B`
- routing table의 row(공통 prefix=1)에, digit `B`로 시작하는 노드 엔트리가 있으면, 그쪽으로 hop.

이 과정을 반복하면, 각 hop마다 **공통 prefix 길이가 +1**이 되어, 최대 $$\log_{16}(N)$$ 단계에서 목적지에 도달한다.

### Routing: Prefix 기반 라우팅

Pastry 라우팅 알고리즘의 골자는 다음과 같다.

1. **목적 keyId k** 가 도착하면, 현재 노드 n은:
   - k가 자신의 leaf set 범위 안에 있는지 확인.
     - 있다면, leaf set 내에서 k와 **가장 가까운 nodeId**를 가진 노드로 바로 보내거나, 자신이 책임을 진다.
   - 아니라면, **routing table**에서 공통 prefix를 한 자리 늘려주는 노드를 찾는다.

2. 공통 prefix가 길어지게 만드는 다음 노드 후보가 여러 개라면:
   - 물리적 거리(지연시간, RTT)가 작은 노드를 우선 선택한다.

3. 만약 알맞은 엔트리가 없으면:
   - leaf set과 routing table과 neighborhood set을 통합해서, 목적지와 **숫자상 거리(keyId 차이)** 를 최소화하는 노드를 고른다.

#### 라우팅 예제

간단한 예를 코드 스타일로 표현해 보자.

```python
def pastry_route(current, key_id):
    """
    current: 현재 노드 객체 (nodeId, leaf_set, routing_table, neighbor_set 보유)
    key_id:  목적 키 ID (int 혹은 hex string)
    """
    # 1) Leaf set 범위 검사
    if key_id in current.leaf_range():
        # leaf set에서 key_id와 가장 가까운 노드 선택
        next_hop = current.closest_in_leaf_set(key_id)
        if next_hop == current.node_id:
            return current  # 내가 책임 노드
        return next_hop

    # 2) prefix 길이 증가를 노리는 routing table lookup
    prefix_len = common_prefix_len(current.node_id, key_id)
    next_digit = digit_at(key_id, prefix_len)  # 공통 prefix 바로 다음 digit
    cand = current.routing_table[prefix_len][next_digit]
    if cand is not None:
        return cand

    # 3) fallback: leaf_set + routing_table + neighbor_set 전체에서
    # key_id와 "숫자상"으로 더 가까운 노드 선택
    all_candidates = current.leaf_set + current.routing_table_all() + current.neighbor_set
    better = min(all_candidates, key=lambda n: distance(n.node_id, key_id))
    if distance(better.node_id, key_id) < distance(current.node_id, key_id):
        return better
    return current  # 더 나은 후보가 없으면 내가 책임
```

이 코드는 실제 Pastry 구현이 아니라, **알고리즘 구조를 이해하기 위한 pseudo code**에 가깝다.

#### 복잡도

- 이상적으로, 각 hop마다 **공통 prefix 길이가 1 증가**하므로,
  - base B, 노드 수 N일 때, 평균 홉 수는
    $$O(\log_B N)$$
- leaf set이 충분히 크고, 노드 입·탈퇴(churn)에 대응하는 유지 작업이 잘 되어 있다면, **경로 길이는 안정적으로 log 스케일**에 머무른다.

### Pastry의 응용

Pastry는 “P2P 공용 라이브러리”처럼 사용될 수 있는 **기반 라우팅 계층**이다. 위에 여러 응용이 올라간다.

대표 예:

1. **PAST**: 대규모 분산 파일 저장 시스템
   - 파일 식별자를 해시 → keyId로 사용.
   - keyId에 “가까운” 노드들이 그 파일을 저장 및 복제(replication)한다.
   - 파일 조회 시, 동일한 Pastry 라우팅으로 해당 keyId에 도달하여 파일 위치를 얻는다.

2. **Scribe**: 분산 pub/sub 시스템
   - 각 “topic”에 대해 keyId를 부여한다.
   - topic tree의 루트는 keyId에 가장 가까운 노드.
   - 구독자는 Pastry 라우팅으로 topic의 루트 노드를 찾아가 join 하고,
     발행자는 루트로 메시지를 보내면 트리가 따라 내려가며 multicast.

3. **분산 인덱스·검색**
   - 키워드를 hash → keyId, 그 keyId를 Pastry에 저장하면 **키워드 → 문서 위치** 매핑을 여러 노드에 분산시킬 수 있다.

#### 사례: 분산 키-값 저장

- 키: `user:1234:profile`
- 값: 사용자 프로필 JSON
- 키를 SHA-1으로 해시 → keyId
- keyId를 Pastry로 라우팅 → keyId에 가장 가까운 노드가 저장

심플한 Python 스타일을 상상해볼 수 있다.

```python
def store(key, value):
    key_id = sha1_int(key)
    responsible = pastry_lookup(key_id)  # 위에서 본 routing
    responsible.local_store[key_id] = value

def get(key):
    key_id = sha1_int(key)
    responsible = pastry_lookup(key_id)
    return responsible.local_store.get(key_id)
```

실제 구현에서는 복제, 버전 관리, 장애 처리, 보안 등을 추가하게 된다.

---

## Kademlia

Kademlia는 Maymounkov & Mazieres(NYU, 2002)가 제안한 DHT로,
오늘날 **가장 널리 쓰이는 DHT 알고리즘**이다(예: BitTorrent DHT, eMule Kad 등).

핵심 아이디어:

- 노드와 키는 모두 **고정 길이(보통 160비트)의 ID** 를 가진다.
- 두 ID 사이의 거리는
  - $$d(x, y) = x \oplus y$$
  - 즉, **XOR 거리**로 정의한다.
- XOR 거리를 이용하면, **대칭(symmetric)** 이고 계산이 빠른 메트릭을 형성하며,
  라우팅 테이블 관리 및 탐색 알고리즘이 간결해진다.

### Identifier Space와 XOR 메트릭

식별자 ID는 보통 **SHA-1 출력(160비트)** 를 사용한다.

- 노드 ID: IP:port 등을 해시한 값
- 키 ID: 컨텐츠 식별자(예: infohash)를 해시한 값

**거리 정의:**

$$
d(x, y) = x \oplus y
$$

XOR 결과를 정수로 해석해 “거리”처럼 쓰는데, 다음 성질을 가진다.

1. $$d(x, x) = 0$$
2. $$d(x, y) = d(y, x)$$ (대칭)
3. 두 점이 같지 않으면 거리 > 0
4. 트라이(trie) 구조와 잘 맞고, prefix 기반 분할 전략을 자연스럽게 지원

예:

- x = 0b101100
- y = 0b001000

$$
x \oplus y = 100100_2
$$

거리의 상위 비트 위치는 **두 노드가 어느 “범위”에 속하는지** 를 나타낸다.

### Routing Table: k-Buckets

Kademlia 라우팅 테이블은 **k-bucket** 이라는 구조로 이루어진다.

- 각 노드는 ID 공간을 **거리 범위별로 log 스케일**로 나누고,
  각 범위마다 **최대 k개의 노드**를 기억한다.
- 예를 들어 160비트 ID 공간에서:
  - bucket 0: 거리 [1, 2¹)
  - bucket 1: [2¹, 2²)
  - ...
  - bucket i: [2ⁱ, 2^(i+1))
  - ...
  - bucket 159: [2¹⁵⁹, 2¹⁶⁰)

하지만 실제 구현에 따라 범위 개수나 구분 방식이 조금씩 다를 수 있다.

**특징:**

- **가까운 노드보다 먼 노드를 더 많이 저장**하는 경향.
  - 이는 **fault tolerance** 와 **탐색 다양성**에 도움이 된다.
- 각 버킷에는 최대 `k`개의 노드가 들어간다.
  - k는 보통 8, 16 같은 작은 수 (BitTorrent DHT에서는 8인 경우가 많다).

#### K-Bucket 관리 정책 (LRU 기반)

새로운 노드를 알게 되었을 때:

1. 그 노드와 자신의 ID 사이의 XOR 거리 d를 계산한다.
2. d가 속하는 범위에 해당하는 bucket을 찾는다.
3. 해당 bucket에 빈 자리가 있으면 추가.
4. 꽉 차 있으면:
   - 가장 오래전에 응답했던 노드(least recently seen)를 ping.
   - 응답이 없으면 그 노드를 제거하고 새 노드를 넣는다.
   - 응답이 있으면 새 노드는 버린다.

이 정책은 **오래 살아있는, 안정적인 노드**를 선호하도록 설계되었다.

---

### Lookup 알고리즘과 예제

Kademlia의 핵심 연산은 “주어진 키 k에 대해 **가장 가까운 노드들**을 찾는 것”이다.
기본 절차는 다음과 같다.

1. **시작 노드**는 자신의 라우팅 테이블에서 k에 대해 가장 가까운 노드 α개(동시 요청 수)를 찾는다.
2. 이 α개에 대해 `FIND_NODE(k)` 요청을 보낸다.
3. 각 응답은, 그 노드가 알고 있는 k에 가까운 노드 목록을 포함한다.
4. 새로 알게 된 노드들 중에서 이전보다 **k에 더 가까운 노드**들을 선택하여 다시 α개에 대해 반복.
5. 더 이상 **새로운 더 가까운 노드**가 없으면 종료.

이때 α는 **동시 질의 수(concurrency)** 로, 성능과 비용의 트레이드오프를 조절한다.

간단한 Python 스타일 pseudo code:

```python
def kademlia_lookup(self, key_id, alpha=3, k=8):
    # self: 시작 노드
    # 1) 자신이 알고 있는 노드 중 key_id에 가까운 상위 k개 선택
    candidates = self.closest_nodes(key_id, k)
    probed = set()
    closest_dist = distance(self.node_id, key_id)
    changed = True

    while changed:
        changed = False
        # 아직 질의하지 않은 노드 중 상위 alpha개 선택
        to_query = [n for n in candidates if n not in probed][:alpha]
        if not to_query:
            break

        for node in to_query:
            probed.add(node)
            # FIND_NODE RPC
            new_nodes = node.rpc_find_node(key_id)
            # 새 노드들을 후보에 병합
            candidates = merge_and_sort_by_distance(candidates + new_nodes, key_id, k)

        # 가장 가까운 거리 갱신 여부 확인
        new_closest = distance(candidates[0].node_id, key_id)
        if new_closest < closest_dist:
            closest_dist = new_closest
            changed = True

    # 최종적으로 k개 노드 반환
    return candidates[:k]
```

이 알고리즘은 평균적으로 $$O(\log N)$$ 홉에서 종료하며, 동시에 여러 노드에 질의하기 때문에 **fault tolerance** 가 높고 응답 시간이 짧다.

---

### Kademlia 기반 응용

Kademlia는 이후 수많은 시스템에 채택되어, 오늘날 **사실상 DHT의 표준 알고리즘**이 되었다.

대표 사례:

- **BitTorrent Mainline DHT**:
  - trackerless BitTorrent에서, infohash → peers 매핑을 저장.
  - BEP 5 사양에 따르면, BitTorrent DHT는 Kademlia 기반의 “sloppy DHT”를 사용한다.
- **eMule Kad 네트워크**
- 기타 연구용/상용 P2P 시스템 다수

예를 들어, BitTorrent DHT에서는:

- 키: torrent의 infohash(20바이트 SHA-1)
- 값: 해당 torrent를 공유 중인 peers(IP:port 리스트)
- `get_peers(infohash)` 요청을 DHT에 날려 peers를 찾는다.

---

## BitTorrent — Tracker 기반 / Trackerless

BitTorrent는 **대용량 파일을 효율적으로 배포**하기 위해 설계된 프로토콜이다.
핵심 개념은 **swarm**(같은 파일을 공유하는 peer 집합)과 **piece 단위 교환**, 그리고 **협력적 choke/unchoke 알고리즘**이다.

초기 BitTorrent는 **중앙 Tracker**에 의존했지만, 이후 **DHT 기반 trackerless 모드**가 도입되었다.

### BitTorrent 개요

- 파일은 여러 **piece**로 나뉜다.
- 각 piece는 다시 여러 **block** 으로 분할되어 전송.
- 각 peer는 자신이 가진 piece를 광고하고, 가진 peers에게서 부족한 piece를 요청한다.
- **tit-for-tat** 유사 메커니즘으로, 잘 업로드하는 peer에게 더 많은 다운로드 기회를 준다.

단순화된 구성 요소:

- **torrent 메타파일(.torrent)**
  - 파일 정보, piece 크기, 각 piece 해시, tracker URL 등 포함.
- **tracker** (초기 방식)
  - swarm에 참여한 peer 목록을 제공.
- **DHT** (trackerless 확장)
  - infohash → peers 매핑을 분산 저장.

---

### BitTorrent with a Tracker

초기 BitTorrent 설계에서는, **tracker** 가 swarm의 중심 역할을 했다.

#### 동작 흐름

1. 사용자가 `.torrent` 파일을 열면, 그 안에 있는 **announce URL** 로 tracker에 접속.
2. peer는 자신의 info(IP, 포트, 다운로드 상황 등)를 tracker에 **announce** 한다.
3. tracker는 해당 torrent swarm에 참여한 다른 peers의 목록을 반환.
4. peer는 이 목록을 바탕으로, 다른 peers와 직접 TCP 연결을 맺고 piece를 교환.

텍스트로 그려보면:

```text
[Peer A] ---\
[Peer B] ----> [Tracker] <---- [Peer C]
[Peer D] ---/              \
                                --> peers list (A,B,C,D)
```

예제 상황:

- 파일 크기: 700 MB
- piece 크기: 256 KB
- 총 piece 수: 약 2800개
- Peer A는 seeder(모든 piece 보유), B·C·D는 leecher

1. B가 tracker에 접속: `ANNOUNCE infohash=XYZ, peerid=B`
2. tracker가 현재 swarm peers 목록: A, C, D를 B에게 전달.
3. B는 A, C, D 각자에게 “handshake → interested → request” 절차를 수행하면서 piece를 받아온다.

실제 신호 교환은 TCP 위의 BitTorrent 메시지(Handshake, Choke, Unchoke, Interested, Request, Piece 등)로 이루어진다.

#### 장·단점

- 장점:
  - 구현이 단순, 초기 배포·운영이 수월.
- 단점:
  - Tracker 장애 시 swarm 전체가 타격.
  - 법적/관리 관점에서 중앙 점에 대한 공격/차단이 쉬움.
  - 규모·부하·신뢰성 모두 중앙 서버에 부담.

이러한 한계를 보완하기 위해 **trackerless BitTorrent** 가 등장했다.

---

### Trackerless BitTorrent — DHT 기반

Trackerless BitTorrent는 **DHT를 이용해 peers를 찾는 방식**이다.
BEP 5 문서에 따르면, BitTorrent DHT는 **Kademlia 기반 distributed sloppy hash table** 을 사용한다.

#### 기본 개념

- 키: torrent의 **infohash** (SHA-1, 20바이트)
- 값: 해당 torrent를 공유 중인 peers(IP, 포트)의 리스트
- 각 DHT 노드는 `(infohash, peers)` 엔트리 일부를 저장한다.
- `get_peers(infohash)` RPC를 통해, infohash에 가까운 노드들을 찾아가면서 peers 목록을 수집.

DHT 노드의 역할과 BitTorrent peer의 역할을 구분해 보면:

- **peer**: 파일을 실제로 올리고/받는 엔티티 (TCP 포트에서 BitTorrent 프로토콜 지원)
- **DHT node**: UDP 포트에서 DHT 프로토콜을 수행하며, `(infohash → peers)` 인덱스를 저장·조회

실제 클라이언트는 둘 다를 **한 프로그램 안에서 동시에 수행**한다.

#### DHT 동작 예제

1. 새 peer P가 torrent(infohash = X)에 참여.
2. P는 DHT에 `announce_peer` 요청을 보내며 “나 X의 peer다” 라고 알린다.
   - 이 때, Kademlia-style `FIND_NODE`/`GET_PEERS`를 이용해 infohash에 가까운 노드들을 찾고,
   - 그 노드들에게 `(X → {P})` 를 저장하게 한다.
3. 다른 peer Q가 같은 torrent를 다운로드하려고 할 때:
   - DHT에 `get_peers(X)` 요청을 보내 peers 목록을 얻는다.
   - 얻은 peers와 BitTorrent TCP 연결을 맺고 piece 교환을 시작한다.

간단한 pseudo code:

```python
def announce_peer(self, infohash, port):
    # 1. infohash 근처 노드를 찾는다
    nodes = kademlia_lookup(infohash)
    # 2. 각 노드에 announce_peer 전송
    for n in nodes:
        n.rpc_announce_peer(infohash, self.ip, port)

def get_peers(self, infohash):
    # 1. infohash 근처 노드를 찾으며 동시에 peers를 요청
    nodes = kademlia_lookup(infohash)
    peers = set()
    for n in nodes:
        peers |= n.rpc_get_peers(infohash)
    return list(peers)
```

위의 `kademlia_lookup`은 29.4에서 설명한 Kademlia 루틴이다.

#### “sloppy” DHT

BitTorrent DHT는 엄밀한 의미의 DHT라기보다는 **sloppy DHT** 라고 불린다.

- 각 노드가 **정확히 key에 가장 가까운** 노드만 가져가는 것이 아니라,
- 일정 범위 내의 여러 노드들이 `(infohash → peers)` 를 중복 저장한다.
- 이는 churn(노드의 빈번한 입·탈퇴)에 대비한 **중복성·탄력성** 을 높인다.

#### 장·단점

- 장점:
  - 중앙 tracker 없이도 peers를 찾을 수 있다.
  - 특정 tracker가 차단되어도 swarm은 계속 운영 가능.
  - ISP나 네트워크 관점에서, 트래픽이 여러 노드로 분산.

- 단점:
  - DHT 트래픽과 복잡성이 추가된다.
  - DHT 자체의 보안 문제(크롤링을 통한 프라이버시 침해, DHT 공격 등)가 등장한다.

---

### BitTorrent with Tracker vs Trackerless — 비교

| 항목 | Tracker 기반 | Trackerless(DHT) |
|------|--------------|------------------|
| peer 검색 | 중앙 tracker에 질의, peers 목록 획득 | DHT lookup으로 infohash에 근접한 노드에서 peers 목록 획득 |
| 장애 영향 | tracker 다운 시 swarm 영향 큼 | 일부 DHT 노드 실패는 영향 제한적 |
| 운영 복잡도 | 서버 운영 필요, but 구현 단순 | DHT 알고리즘/보안/트래픽 고려 필요 |
| 차단 가능성 | 중앙 서버 차단으로 전체 영향 | 완전 차단이 매우 어려움 |
| 프라이버시 | tracker 로그에 의존 | DHT 크롤링으로 전체 swarm 추적 가능성 |

현대 BitTorrent 클라이언트 대부분은 **tracker + DHT + peer exchange(PEX)** 를 함께 사용해서
탐색 효율성과 탄력성을 동시에 추구한다.

---

## 정리

- **Pastry**: prefix 기반 라우팅과 leaf set / routing table / neighborhood set 구조를 이용해
  $$O(\log N)$$ 홉의 효율적인 객체 위치 서비스를 제공하는 오버레이.
- **Kademlia**: XOR 거리와 k-bucket을 이용해, 간결하면서도 fault-tolerant한 DHT를 구현.
  오늘날 가장 널리 쓰이는 DHT 설계.
- **BitTorrent**:
  - 초창기에는 중앙 tracker에 의존해서 peer discovery를 수행.
  - 이후 Kademlia 기반 DHT를 이용해 **trackerless** 모드를 지원.
  - 실제 환경에서는 tracker, DHT, PEX를 조합해 사용.

이 세 설계를 함께 보면, P2P 시스템이 **중앙 집중형 → 분산형 인덱스 → 완전 분산형 피어 검색** 으로 진화해 온 흐름을 자연스럽게 이해할 수 있다.
