---
layout: post
title: 데이터 통신 - Peer-to-Peer Paradigm (1)
date: 2024-09-25 20:20:23 +0900
category: DataCommunication
---
# Chapter 29. Peer-to-Peer Paradigm

## Introduction

### P2P Networks

#### P2P의 기본 아이디어

**클라이언트–서버 모델**과 **P2P 모델**을 비교해 보자.

- **클라이언트–서버**
  - 서버: 항상 켜져 있고, 데이터/서비스를 중앙에서 제공
  - 클라이언트: 요청을 보내고 응답을 받는 쪽
  - 장점: 관리/제어가 쉽고, 일관된 정책 적용이 가능
  - 단점: 서버/네트워크 병목, 단일 장애점(SPOF), 비용 증가

- **P2P (Peer-to-Peer)**
  - 모든 참여자는 **peer**이며, 동시에 **클라이언트이자 서버** 역할을 수행
  - 자원(파일, 저장공간, 계산능력)을 서로 직접 교환
  - **중앙 서버 없이도** 검색, 라우팅, 데이터 공유를 수행할 수 있음
  - 장점
    - 자원/트래픽이 자연스럽게 분산
    - 노드가 늘수록 총 용량과 처리능력도 증가(“scales with users”)
  - 단점
    - 노드 churn(가입/탈퇴)이 많아 **라우팅 구조 유지가 어려움**
    - 악의적 노드, 보안, 신뢰 문제

현대 인터넷에서 P2P는
- **파일 공유(BitTorrent, IPFS 등)**
- **분산 스토리지/분산 파일 시스템**
- **블록체인/분산 원장**
등 다양한 형태로 쓰이고 있다.

#### 구조에 따른 분류 — Unstructured vs Structured vs Hybrid

P2P 네트워크는 통상 다음과 같이 나눈다.

| 유형 | 구조 | 예 | 장점 | 단점 |
|------|------|----|------|------|
| Unstructured | 명시적인 구조 없음, 임의 연결 | 초기 Gnutella 등 | 구현이 단순, 로버스트 | 검색 성능이 네트워크 전체 broadcast 탐색에 의존 |
| Structured | **Distributed Hash Table(DHT)** 기반, 명시적 오버레이 구조 | Chord, Pastry, CAN, Kademlia 등 | $$O(\log N)$$ 수준의 scalable lookup | 디자인/유지 복잡, 노드 churn 처리 필요 |
| Hybrid / Super-peer | 일부 노드(super-peer)가 집계/인덱싱 역할 | 현대 BitTorrent, Skype 초기 설계 등 | 성능과 유연성의 절충 | super-peer가 부분적 병목/핵심 자원 |

---

#### P2P 네트워크에서 공통적으로 필요한 기능

P2P 시스템은 보통 다음 기본 기능을 제공해야 한다.

1. **Join / Leave**
   - 새로운 노드가 네트워크에 진입할 수 있어야 하고
   - 기존 노드는 언제든 떠날 수 있어야 하며
   - 이때도 **데이터 유실 없이** 라우팅 구조가 유지되어야 한다.

2. **Lookup(k)**
   - 키(파일 ID, 콘텐츠 이름 등) $$k$$에 대해
     “어느 노드가 이 키를 담당하는가?”를 효율적으로 찾아야 한다.

3. **Publish / Store(k, v)**
   - 키–값 쌍을 네트워크 어딘가에 저장하고,
     나중에 누구든 lookup으로 찾을 수 있어야 한다.

4. **Replication / Fault Tolerance**
   - 노드가 갑자기 사라져도 데이터가 유지되도록
     여러 노드에 복제, 혹은 인접 노드에 백업을 유지한다.

5. **Load Balancing**
   - 일부 노드에만 키가 몰리지 않도록
     해시함수/consistent hashing 등을 통해 **키를 균일하게 분산**한다.

이 중 **lookup과 key 분배 문제**를 시스템적으로 해결한 것이 바로 **Distributed Hash Table(DHT)** 이다.

---

### Distributed Hash Table (DHT)

#### 일반적인 Hash Table 복습

전통적인 해시 테이블은:

- 키 공간: 논리적으로 매우 큼
- 해시 함수:
  $$h(k) \to \{0,1,\dots, M-1\}$$
- 테이블: 길이 $$M$$인 배열, `table[h(k)]`에 데이터 저장
- Lookup: 평균 $$O(1)$$

하지만 이 구조를 **여러 노드에 분산**하려면:

- 테이블의 일부 index 범위를 노드들에 나눠 갖게 해야 함
- 노드가 동적으로 들어오고 나가도,
  “키 → 어떤 노드?”를 빠르게 찾아야 함

이 아이디어를 확장한 것이 DHT이다.

---

#### DHT의 기본 개념

**Distributed Hash Table**은 다음을 만족하는 분산 데이터 구조이다.

- 대규모 P2P 네트워크에서 **키–값 매핑** 제공
- 각 노드가 **키 공간의 일부**를 담당
- Lookup 및 Insert/Delete 연산의 복잡도는 보통
  $$O(\log N)$$ (여기서 $$N$$은 노드 수)
- 노드 가입/탈퇴에도
  - 재구성 비용이 $$O(\log^2 N)$$ 이하
  - 혹은 평균/분산 측면에서 안정적인 동작을 목표로 한다.

DHT는 보통 이런 연산을 제공한다:

- `put(key, value)`
- `get(key) -> value`
- `remove(key)`

DHT 시스템(Chord, Kademlia 등)은
“어떤 노드가 **어떤 키 범위**를 책임지는지”를 정의하고,
“해당 노드를 어떻게 찾아가는지”에 대한 규칙을 정한다.

---

#### DHT와 Unstructured P2P의 차이 예제

- **Unstructured**
  - 예: Flooding 기반 검색
  - 노드 A가 “파일 X”를 찾고 싶을 때:
    - 이웃들에게 쿼리를 보내고,
    - 그 이웃이 또 이웃들에게 보내면서
    - Time-To-Live(TTL) 내에 응답하는 노드가 있으면 OK
  - 단점: 네트워크 전체에 **브로드캐스트성 트래픽** 발생

- **DHT 기반 Structured P2P**
  - 각 파일을 **키값으로 해시**하고
  - 키를 담당하는 노드를 **수학적으로 결정** 가능
  - “파일 X”의 키 $$k = H(\text{"파일 X"})$$를 구한 후
    - DHT 라우팅 규칙을 이용해
      $$O(\log N)$$ 단계 만에 담당 노드까지 이동

즉, DHT는 **검색 효율이 구조적으로 보장**되고
노드 수가 늘어도 성능이 확장 가능하다.

---

#### 실제 DHT 적용 사례

- **Chord, Pastry, CAN, Tapestry**
  → 초기 연구용/프로토타입 P2P 시스템들
- **Kademlia 기반 DHT**
  - BitTorrent Mainline DHT, Vuze 등 대규모 파일 공유 시스템에서 사용
  - 2010년대 기준으로 동시 수천만 사용자 규모의 overlay로 관측됨

이 글에서는 DHT의 대표 구현으로 **Chord**에 초점을 맞춘다.

---

## Chord

### Identifier Space

#### Chord의 핵심 아이디어

**문제 정의**:
“키 $$k$$가 주어졌을 때, 이 키를 담당하는 노드가 누구인지 찾아라.”

Chord는 이를 위해:

1. **모든 노드와 키를 동일한 identifier 공간에 매핑**하고
2. identifier 공간을 **원형 링(ring)**으로 본다.
3. 키 $$k$$는 “**시계 방향으로 처음 등장하는 노드**”가 담당한다.

Chord는 MIT에서 2001년에 제안되었고,
**consistent hashing**을 이용해 키 분산과 노드별 부하를 균일하게 유지한다.

---

#### Identifier 공간 정의

- $$m$$비트 identifier 공간 사용
- 가능한 identifier는
  $$\{0, 1, 2, \dots, 2^m - 1\}$$
- 노드와 키는 각각 해시 함수(예: SHA-1)를 사용해
  이 공간으로 매핑된다.

수식으로 쓰면:

- 노드 $$n$$의 IP:port 문자열을 $$s_n$$이라고 할 때
  $$\text{id}(n) = H(s_n) \bmod 2^m$$
- 키 $$k$$ (예: 파일 이름, 콘텐츠 ID)에 대해
  $$\text{id}(k) = H(k) \bmod 2^m$$

여기서 $$H$$는 충돌이 적고 균일하게 퍼지는 해시 함수이다.
Chord 원 논문에서는 SHA-1 기반 160비트 공간을 사용한다.

---

#### 원형 링 구조와 Successor/Predecessor

identifier 공간을 **모듈러 $$2^m$$** 원형으로 생각하면:

- 0 바로 뒤에는 $$2^m-1$$이 아닌,
  $$2^m-1$$ 바로 다음이 0이 아니라
  0의 왼쪽이 $$2^m-1$$, 오른쪽으로 1, 2, … 순서로 돈다고 보면 된다.

각 노드 $$n$$에 대하여:

- **successor(n)**:
  - 링에서 시계 방향으로 처음 만나는 노드
- **predecessor(n)**:
  - 링에서 시계 반대 방향으로 처음 만나는 노드

**키 할당 규칙**:

- 키 $$k$$의 담당 노드는
  $$\text{successor}(k)$$
  (즉, $$k \le \text{id}(n)$$인 노드 중 가장 작은 id를 가진 노드)

---

#### 작은 예제: m = 4, N = 8

간단히 $$m = 4$$ (identifier 0~15)인 상황을 생각해 보자.

- 참여 노드: id = 1, 4, 8, 11, 14

원형으로 그리면(개념적):

```
0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
    N1          N4          N8      N11          N14
```

키 할당 예:

- 키 id=2 → 시계 방향 첫 노드: 4 → 노드 4 담당
- 키 id=9 → 시계 방향 첫 노드: 11 → 노드 11 담당
- 키 id=15 → 시계 방향 첫 노드: (wrapped) 1 → 노드 1 담당

이렇게 키들은 링 위에 균등하게 분배된다
(해시 함수가 균일하다고 가정할 때).

---

### Finger Table

Chord는 단순히 successor 포인터만 쓰면
최악의 경우 $$O(N)$$ hop이 걸리고,
이는 대규모 시스템에서는 비효율적이다.

그래서 각 노드는 **finger table**이라는
“멀리 있는 노드들을 가리키는 점프 포인터 집합”을 유지한다.

#### Finger table 정의

노드 $$n$$의 finger table은 $$m$$개의 엔트리를 가진다.

- $$\text{finger}[i].\text{start} = n + 2^{i-1} \ (\bmod 2^m), \quad i=1..m$$
- $$\text{finger}[i].\text{node} = \text{successor}(\text{finger}[i].\text{start})$$

즉, 각 엔트리는 현재 노드로부터
2의 거듭제곱만큼 떨어진 지점의 successor를 가리킨다.

이렇게 하면, 찾고자 하는 키 $$k$$에 대해
“$$k$$를 **넘지 않는 범위에서** 최대한 멀리 점프”하며
로그 단계로 빠르게 접근할 수 있다.

---

#### 예제: m = 4, 노드 1의 finger table

다시 $$m = 4$$, 노드 {1,4,8,11,14} 예제를 사용해 보자.

- 노드 1의 finger start 값:

| i | start = 1 + 2^{i-1} (mod 16) | successor(start) |
|---|------------------------------|------------------|
| 1 | 1 + 1 = 2                    | 4                |
| 2 | 1 + 2 = 3                    | 4                |
| 3 | 1 + 4 = 5                    | 8                |
| 4 | 1 + 8 = 9                    | 11               |

따라서 노드 1의 finger table은:

| i | interval [start, next_start) | finger[i].node |
|---|------------------------------|----------------|
| 1 | [2, 3)                       | 4              |
| 2 | [3, 5)                       | 4              |
| 3 | [5, 9)                       | 8              |
| 4 | [9, 1) (wrap)                | 11             |

이 정보로 노드 1은 많은 키에 대해
한 번에 2, 4, 8 단위로 점프하며
빠르게 담당 노드에 접근할 수 있다.

---

#### Finger table을 이용한 lookup 절차 예제

**목표**: 노드 1에서 키 id=14의 담당 노드를 찾고 싶다.

1. 노드 1은 자신의 successor(=4)를 알고 있다.
2. 키 14는 (시계 방향으로) 14 → (wrap) 0 → 1 … 에서 14의 successor는 1이다.
3. 하지만 finger table 없이 linear search를 하면
   1 → 4 → 8 → 11 → 14 → 1로 돌아가며 확인해야 한다.

finger table을 사용한 **Chord lookup 알고리즘**은 대략 다음과 같다.

```pseudo
function find_successor(id):
    n0 = this_node
    if id ∈ (n0, successor(n0)]:
        return successor(n0)
    else:
        n' = closest_preceding_finger(id)
        return n'.find_successor(id)   // RPC
```

여기서 `closest_preceding_finger(id)`는
finger table에서 $$id$$보다 작은 노드 중
가장 큰 식별자를 가진 노드를 반환한다.

실제 Chord에서는 이 과정을 통해
**최대 $$O(\log N)$$ hop** 만에
대상 노드에 도달할 수 있음을 증명한다.

---

### Interface — Join, Lookup, Stabilization

Chord 프로토콜이 외부에 제공하는 인터페이스는
“논리적으로” 매우 단순하다.

#### 외부에서 보는 인터페이스

- **`lookup(key)` / `find_successor(id)`**
  - 특정 키에 대한 담당 노드를 찾는다.
- **`join(existing_node)`**
  - 기존 네트워크에 새 노드가 합류한다.
- **`leave()` (또는 실패)**
  - 노드가 정상적으로 떠나거나, 비정상 종료된다.
- **배경 프로시저 (Background tasks)**
  - `stabilize()`, `fix_fingers()`, `check_predecessor()`

이러한 인터페이스를 바탕으로 상위 애플리케이션은
단지 **“키 → 노드”**만 알면 되며,
P2P 라우팅과 재구성은 Chord 구현체가 책임진다.

---

#### Join 예제 시나리오

예를 들어 기존에 노드 id = {1, 4, 8, 11, 14}가 있고,
새로운 노드 id=6이 Chord 링에 합류한다고 하자.

1. 노드 6은 기존 네트워크의 임의 노드(예: 노드 1)의 주소를 알고 있다고 가정.
2. 노드 6이 `join(1)` 수행:
   - 6은 “내 successor는 누구인가?”를 찾기 위해
   - 기존 노드 1에게 `find_successor(6)`을 요청
   - 1 → (finger table 활용) → 8을 반환 (6의 successor는 8)
3. 노드 6은 이제 `successor(6)`으로 8을 설정
4. 노드 6의 predecessor는 4 (4 < 6 < 8)
5. 4와 8의 predecessor/successor 정보도 업데이트
6. 6의 책임 범위가 생기므로,
   - 8이 담당했던 키 중 일부(4 < key ≤ 6)를 6으로 이전

이후 `stabilize()`/`fix_fingers()` 같은 주기적 프로시저가
finger table과 successor 정보를 점차 정확히 보정한다.

---

#### 안정화(stabilization) 및 장애 처리

Chord는 노드 churn을 가정하기 때문에
**백그라운드에서 주기적으로** 다음과 같은 작업을 수행한다.

- `stabilize()`:
  - 현재 successor의 predecessor를 확인하고
  - 더 적절한 successor가 있으면 갱신
- `fix_fingers()`:
  - finger table 엔트리들을 점검하고 새 값으로 보정
- `check_predecessor()`:
  - predecessor가 죽었는지 확인하고, 죽었다면 nil로 설정

또한 **successor list** (예: 다음 몇 개의 후속 노드 목록)를 유지해
한 노드가 갑자기 죽더라도 **다음 successor로 fallback**할 수 있게 한다.

---

#### 간단한 Python 유사 코드 예시

아래는 Chord의 핵심을 간단히 모사한 Python 스타일 의사코드이다
(실제 네트워크 RPC 구현은 생략).

```python
class Node:
    def __init__(self, node_id, m=160):
        self.id = node_id
        self.m = m
        self.successor = self
        self.predecessor = None
        self.finger = [self for _ in range(m)]
        self.data = {}  # key -> value

    def in_interval(self, x, a, b, inclusive_right=True):
        # x ∈ (a, b] (mod 2^m) 여부 판단
        mod = 2 ** self.m
        if a < b:
            if inclusive_right:
                return a < x <= b
            return a < x < b
        else:
            # wrap
            if inclusive_right:
                return x > a or x <= b
            return x > a or x < b

    def find_successor(self, id_):
        if self.in_interval(id_, self.id, self.successor.id, inclusive_right=True):
            return self.successor
        else:
            n0 = self.closest_preceding_finger(id_)
            if n0 == self:
                return self.successor
            return n0.find_successor(id_)  # 실제 구현에선 RPC

    def closest_preceding_finger(self, id_):
        for i in reversed(range(self.m)):
            f = self.finger[i]
            if self.in_interval(f.id, self.id, id_, inclusive_right=False):
                return f
        return self

    def join(self, known_node):
        if known_node is None:
            # 최초 노드
            self.predecessor = None
            self.successor = self
        else:
            self.predecessor = None
            self.successor = known_node.find_successor(self.id)
            # 이후 stabilize() 호출로 보정

    def put(self, key, value, hash_func):
        k_id = hash_func(key) % (2 ** self.m)
        owner = self.find_successor(k_id)
        owner.data[k_id] = value

    def get(self, key, hash_func):
        k_id = hash_func(key) % (2 ** self.m)
        owner = self.find_successor(k_id)
        return owner.data.get(k_id, None)
```

이러한 구조 위에
분산 key-value 저장소, 분산 파일 시스템 등
다양한 응용을 얹을 수 있다.

---

### Application — Chord 위에서 무엇을 만들 수 있는가?

Chord는 기본적으로 **“scalable lookup service”** 이다.

즉, 응용은 “`lookup(key)` → node”만 믿고
나머지 데이터 저장/복제 로직만 구현하면 된다.

#### 분산 key-value 저장소 예제

- 키: 파일의 해시, 문서 ID, 사용자 ID 등
- 값: 실제 데이터 or 다른 위치 정보

**단순 모델**:

1. 클라이언트가 `put(k, v)`를 호출
   - 클라이언트는 Chord 오버레이 상의 임의 노드에게 RPC
2. 해당 노드는 `find_successor(id(k))`로 담당 노드를 찾음
3. 담당 노드가 `v`를 local storage에 저장
4. `get(k)` 호출 시 동일 방식으로 담당 노드를 찾아 반환

이 경우 중앙 디렉터리 서버 없이도
대규모 분산 key-value 저장소가 만들어진다.

#### 분산 파일 시스템 / 콘텐츠 주소 스토리지

- 파일 내용 전체를 해시한 값이 키가 된다:
  $$k = H(\text{file contents})$$
- 장점:
  - 동일한 내용을 가진 파일은 항상 동일한 키를 갖는다 (중복 제거)
  - 키만 알면 어느 노드에서든 내용 검색 가능
- 구현:
  - 파일을 여러 chunk로 나누고 각 chunk에 대한 키를 만들거나
  - 메타데이터(파일 이름 → chunk 리스트)를 DHT에 저장

이 아이디어는 현대의 여러 분산 스토리지/파일 시스템,
콘텐츠 주소 스토리지(IPFS 등) 개념과 직간접적으로 연결된다.

---

#### 예제 시나리오: 분산 로그 인덱스 서비스

**상황**
여러 데이터센터에 분산된 서버들이 있고,
각 서버는 자신이 생성한 로그들을
중앙 서버 없이도 “검색 가능하게” 만들고 싶다.

**설계**:

1. 각 로그 파일 혹은 로그 레코드에 대해
   - 키: `H(timestamp + host + path)` 또는 특정 필드 조합
2. 모든 서버는 Chord 오버레이에 참여
   - 각 서버는 자신이 가진 로그의 `(key, location)`을 DHT에 등록
   - `location`은 해당 로그를 실제로 읽을 수 있는 URL, host/path 등
3. 분석/검색 도구는
   - 특정 키/패턴에 해당하는 key set을 만들고
   - Chord를 통해 해당 key들의 담당 노드를 찾아
   - 위치 정보를 수집한 뒤, 각 서버에서 로그를 병렬로 가져온다.

이렇게 하면
**중앙 로그 인덱스 서버 없이도**
대규모 분산 로그 검색 인프라를 구성할 수 있다.

---

#### 다른 DHT와의 비교 – 개념 확장

- **Kademlia**
  - 거리 개념으로 XOR metric 사용
  - `k-bucket` 기반 라우팅 테이블 유지
  - BitTorrent Mainline DHT에서 사용, 수천만 노드 규모 운용 사례 있음
- **Chord**
  - 원형 identifier 공간 + finger table
  - 수학적으로 간결하고 분석이 쉬운 구조
- **Pastry/Tapestry**
  - prefix 기반 라우팅 테이블
  - 객체/노드 ID의 prefix를 공유하는 노드를 따라 hop-by-hop 라우팅

Chord를 이해하면,
다른 DHT들(Kademlia, Pastry 등)은
“거리 정의 / 라우팅 테이블 구조만 조금 다른 변형”이라는 사실을
보다 쉽게 받아들일 수 있다.

---

#### 간단한 실험 아이디어

블로그/포트폴리오용으로는,
전체 분산 구현 대신 **로컬에서 동작하는 작은 Chord 시뮬레이터**를 만들어 볼 수 있다.

아이디어:

1. Python에서 `Node` 클래스를 만들고,
   각 인스턴스가 Chord 노드를 의미하게 한다.
2. 네트워크 통신 대신 **함수 호출로 RPC를 모사**한다.
3. 여러 노드를 생성하고, 랜덤 키에 대해 `put/get`을 수행해본다.
4. 각 lookup에서 몇 hop이 걸리는지 측정해
   노드 수를 늘려가며 **평균 hop 수가 $$O(\log N)$$에 근접하는지** 확인한다.

이를 통해
- DHT 구조가 왜 scale-out에 유리한지,
- finger table이 왜 중요한지
직접 체감할 수 있다.

---

## 정리

이 글에서 다룬 내용:

1. **P2P Networks**
   - 클라이언트–서버 vs P2P
   - Unstructured / Structured(DHT) / Hybrid P2P
   - Join/Leave, Lookup, Publish, Replication 등 공통 기능

2. **Distributed Hash Table (DHT)**
   - 해시 테이블을 분산 환경으로 확장
   - 키–노드 매핑, $$O(\log N)$$ 수준의 확장 가능한 lookup
   - Chord, Kademlia, Pastry 등 대표 DHT

3. **Chord**
   - **Identifier space**: $$2^m$$ 크기의 원형 링, SHA-1 해시 등 사용
   - **Finger table**: 2의 거듭제곱 간격으로 노드를 가리키는 점프 포인터
   - **Interface**: `lookup`, `join`, `stabilize` 등 최소한의 API
   - **Application**: 분산 key-value 저장소, 분산 파일 시스템, 분산 로그 인덱스 등

이제 다음 단계로,
- 실제 코드를 조금 더 확장해 시뮬레이터를 만들거나
- Kademlia, Pastry와 비교 정리,
- 현대 P2P 시스템(BitTorrent DHT, IPFS, 블록체인 노드 검색 등)에
  Chord/DHT 개념이 어떻게 녹아 있는지 분석
