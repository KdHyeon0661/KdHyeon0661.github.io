---
layout: post
title: 데이터 통신 - Introduction to Network Layer (1)
date: 2024-08-05 22:20:23 +0900
category: DataCommunication
---
# Chapter 18. Introduction to Network Layer — Services와 Packet Switching (Datagram vs Virtual Circuit)

> 목표
> - **18.1 Network-layer services**: 패킷화(packetizing), 라우팅·포워딩(routing and forwarding), 그 밖의 서비스(주소, 오류 보고, QoS, 보안 등)를 “IP(IPv4/IPv6)를 중심으로” 정리.
> - **18.2 Packet switching**:
>   - **Datagram approach: connectionless service** (인터넷의 IP 모델)
>   - **Virtual-circuit approach: connection-oriented service** (ATM, MPLS, 일부 통신망)
>   를 서로 비교하면서 예제·의사코드·수식을 곁들여 설명한다.

Internet Protocol(IP)는 **패킷 교환(packet switching)** 기반의 네트워크 계층 프로토콜로,
“Datagram”이라는 독립적인 패킷 단위를 서로 다른 네트워크 경계를 넘어 전달한다.

---

## Network-Layer Services

네트워크 계층(Network layer)은 크게 다음 일을 담당한다.

1. **Packetizing**
   - 상위 계층(주로 Transport: TCP/UDP)이 넘겨주는 데이터 단위를 **패킷(= datagram)**으로 캡슐화.
2. **Routing and Forwarding**
   - 목적지까지 “어떤 경로(route)”를 사용할지 결정 (라우팅)
   - 실제 각 라우터가 “다음 홉(next hop)”으로 패킷을 넘기는 동작 (포워딩)
3. **Other services**
   - 주소 지정(IP 주소)
   - 오류·진단(ICMP)
   - 혼잡 제어·품질(QoS, DiffServ)
   - 보안(IPsec)
   - 조각화/재조립(fragmentation/reassembly) 등

---

## Packetizing

### Packetizing이란?

전송 계층(예: TCP, UDP)은 애플리케이션 데이터를 **세그먼트(segment)**나 **데이터그램** 단위로 다룬다.
네트워크 계층 IP 모듈은 이를 받아 **IP 헤더를 붙여 “인터넷 데이터그램(Internet datagram)”**을 만든다.

- 입력:
  - TCP 세그먼트 (TCP 헤더 + 데이터)
  - 또는 UDP 데이터그램
- 출력:
  - IP 데이터그램 (IP 헤더 + TCP/UDP + 데이터)

수식으로 쓰면, IP 계층에서 다루는 한 패킷의 길이는

$$
L_{\text{IP}} = L_{\text{IP-header}} + L_{\text{payload}}
$$

여기서 payload는 TCP/UDP 헤더 + 애플리케이션 데이터까지 포함한 전체 상위 계층 데이터다.

---

### 캡슐화·역캡슐화 과정

예를 들어, 웹 브라우저가 HTTP 요청을 보낼 때의 흐름을 단계별로 보면:

1. **애플리케이션 계층**
   - 브라우저가 HTTP 요청 메시지 생성: `GET /index.html HTTP/1.1 ...`
2. **전송 계층(TCP)**
   - HTTP 요청을 TCP 세그먼트로 캡슐화 (포트 80/443 등 포함).
3. **네트워크 계층(IP)**
   - TCP 세그먼트를 **IP 데이터그램으로 캡슐화**
   - 목적지 IP, 출발지 IP, TTL, DS 필드(서비스 품질), 식별자 등 설정.
4. **링크 계층(Ethernet)**
   - IP 데이터그램을 이더넷 프레임으로 캡슐화.

ASCII “레이어 캡슐화” 그림:

```text
[Application]   HTTP 요청
[Transport]     TCP 헤더 + HTTP 데이터
[Network]       IP 헤더 + TCP + HTTP 데이터
[Link]          MAC 헤더 + IP + TCP + HTTP 데이터 + FCS
[Physical]      전기/광 신호로 전송
```

수신 측에서는 정확히 반대 순서로 **역캡슐화**가 된다.

---

### MTU와 조각화(Fragmentation)

각 링크에는 **MTU(Maximum Transmission Unit)**가 있다.

- 예: 전통적인 이더넷 MTU = 1500 bytes (IP 헤더 + IP payload 포함).
- 하나의 IP 데이터그램이 MTU보다 크면, IPv4에서는 **조각화(fragmentation)**가 발생한다.

예를 들어:

- IP 헤더 20바이트
- TCP 헤더 20바이트
- 애플리케이션 데이터 4000바이트

총 길이:

$$
L_{\text{total}} = 20 + 20 + 4000 = 4040\ \text{bytes}
$$

이를 MTU 1500바이트 링크 위로 보내야 한다고 하자.

- 한 패킷에 실을 수 있는 IP payload 최대는
  $$1500 - 20 = 1480\ \text{bytes}$$ (IPv4 헤더 20바이트 가정)
- payload 4000바이트를 1480씩 쪼개면:

$$
\left\lceil \frac{4000}{1480} \right\rceil = \left\lceil 2.7027... \right\rceil = 3
$$

→ 최소 3개 조각(fragment)로 전송해야 한다.

이때 네트워크 계층은:

1. 원래 IP 데이터그램을 여러 조각으로 나누고,
2. 각 조각에 같은 “식별자(ID)”를 넣어,
3. 목적지에서 다시 합칠 수 있도록 한다.

**IPv6는 중간 라우터 조각화를 금지**하고, 출발지 호스트가 Path MTU Discovery를 통해 적절한 크기로 세그먼트를 조정하는 방향으로 바뀌었다.

---

### 간단 Packetizing 의사코드 예제

IPv4에서 “너무 큰” 세그먼트를 받아 조각하는 느낌의 의사코드를 파이썬으로 적어보면:

```python
MTU = 1500
IP_HEADER = 20  # 단순 가정

def packetize(payload: bytes):
    max_payload = MTU - IP_HEADER
    packets = []
    offset = 0
    ident = 12345  # 같은 Datagram 식별자

    while offset < len(payload):
        chunk = payload[offset: offset + max_payload]
        more_frag = (offset + max_payload) < len(payload)
        ip_header = {
            "id": ident,
            "offset": offset // 8,  # IPv4는 8바이트 단위
            "MF": 1 if more_frag else 0
        }
        packets.append((ip_header, chunk))
        offset += max_payload

    return packets

# 사용 예

data = b"A" * 4000
fragments = packetize(data)
for h, c in fragments:
    print(h, "len:", len(c))
```

실제 IPv4는 8바이트 단위 offset, 플래그 비트 등 세부 규칙이 있지만,
핵심은 “**하나의 상위 계층 데이터가 여러 데이터그램으로 쪼개질 수 있다**”는 점이다.

---

## Routing and Forwarding

네트워크 계층의 핵심 서비스는 **“어디로 보낼지 결정”**하는 것이다.

- **Routing (라우팅)**:
  - 네트워크 전체의 토폴로지 정보를 이용해서
    “각 목적지(prefix)에 대해 어떤 출구 인터페이스를 사용해야 하는가?”를 결정하고 **라우팅 테이블**을 계산·갱신하는 **제어 평면(Control plane)** 역할.
- **Forwarding (포워딩)**:
  - 수신한 각 패킷을 라우팅 테이블/FIB를 참고해서 **실제로 다음 홉으로 넘기는 데이터 평면(Data plane)** 역할.

---

### Routing: 제어 평면

#### 정적 라우팅 vs 동적 라우팅

- **정적 라우팅(Static)**: 관리자가 직접 “목적지 prefix → next hop”을 입력.
- **동적 라우팅(Dynamic)**: 라우팅 프로토콜(OSPF, IS-IS, BGP 등)이
  네트워크 변화를 감지하고 자동으로 테이블을 계산.

라우팅 알고리즘 자체는 일반적으로:

- **거리 벡터(Distance Vector)**: Bellman-Ford류 알고리즘 (RIP 등)
- **링크 상태(Link State)**: Dijkstra 기반 최단 경로 알고리즘 (OSPF, IS-IS 등)

을 사용해 “목적지까지 비용(cost)을 최소화하는 경로”를 찾는다.

수식으로 쓰면, 링크 가중치 \(w(u,v)\)를 가진 그래프 \(G=(V,E)\)에서
어떤 출발지 \(s\)에 대해 목적지 \(t\)까지의 경로 비용은

$$
\text{cost}(s \to t) = \min_{\text{path } P: s\to t} \sum_{(u,v) \in P} w(u,v)
$$

이다. 링크 상태 라우팅 프로토콜은 각 라우터가 전체 그래프를 알고 있다고 가정하고,
Dijkstra 알고리즘으로 위 최소 비용을 계산한다.

#### 간단 라우팅 계산 예 (Python)

작은 그래프에서 최단 경로를 구하는 Dijkstra 예제를 보자:

```python
import heapq

graph = {
    "R1": {"R2": 1, "R3": 4},
    "R2": {"R1": 1, "R3": 2, "R4": 5},
    "R3": {"R1": 4, "R2": 2, "R4": 1},
    "R4": {"R2": 5, "R3": 1},
}

def dijkstra(start):
    dist = {node: float("inf") for node in graph}
    dist[start] = 0
    pq = [(0, start)]

    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]:
            continue
        for v, w in graph[u].items():
            nd = d + w
            if nd < dist[v]:
                dist[v] = nd
                heapq.heappush(pq, (nd, v))
    return dist

print(dijkstra("R1"))
```

라우터 R1이 이 그래프를 알고 있다면,
각 라우터까지의 최단 비용을 `dijkstra("R1")`을 통해 계산하고,
그 결과를 기반으로 **“각 목적지에 대해 첫 홉(next hop)을 누구로 할지”**를 결정한다.

---

### Forwarding: 데이터 평면

라우팅 테이블이 준비되면, 이제 실제 패킷이 들어올 때 수행하는 것은 **단순한 lookup**이다.

#### Longest Prefix Match

IPv4/IPv6 라우팅에서 가장 중요한 규칙은 **최장 접두사 매칭(longest prefix match)**이다.

- 라우팅 테이블에 다음과 같은 엔트리가 있다고 하자.

| Prefix | Next Hop |
|--------|----------|
| 0.0.0.0/0 | ISP (default) |
| 10.0.0.0/8 | 내부 Backbone |
| 10.1.0.0/16 | DataCenter |
| 10.1.1.0/24 | 특정 서버 팜 |

목적지 IP가 10.1.1.42라면,
위 네 prefix와 모두 매칭되지만, **마스크 길이가 가장 긴 /24**가 선택된다.

수식:

$$
\text{route}(d) = \arg\max_{i: d \in p_i} \text{prefixlen}(p_i)
$$

#### Forwarding 의사코드

```python
import ipaddress

routes = [
    (ipaddress.IPv4Network("10.1.1.0/24"), "DataCenterFarm"),
    (ipaddress.IPv4Network("10.1.0.0/16"), "DataCenter"),
    (ipaddress.IPv4Network("10.0.0.0/8"), "Backbone"),
    (ipaddress.IPv4Network("0.0.0.0/0"), "DefaultISP"),
]

def lookup(dst_ip):
    dst = ipaddress.IPv4Address(dst_ip)
    best_next = None
    best_len = -1
    for net, nh in routes:
        if dst in net and net.prefixlen > best_len:
            best_len = net.prefixlen
            best_next = nh
    return best_next

for addr in ["10.1.1.42", "10.1.2.5", "8.8.8.8"]:
    print(addr, "->", lookup(addr))
```

- `10.1.1.42` → DataCenterFarm (/24)
- `10.1.2.5` → DataCenter (/16)
- `8.8.8.8` → DefaultISP (/0)

실제 라우터는 이를 TCAM/ASIC에 넣어 **라인 레이트(line rate)**로 처리한다.

---

### Delay 관점에서 본 라우팅/포워딩

네트워크 계층에서 보통 한 패킷의 end-to-end 지연은

$$
D_{\text{end-to-end}} \approx \sum_{i=1}^{N} \big( D_{\text{proc},i} + D_{\text{queue},i} + D_{\text{trans},i} + D_{\text{prop},i} \big)
$$

- \(N\): 거치는 라우터/링크 수
- \(D_{\text{proc}}\): 헤더 검사·라우팅 테이블 lookup 같은 처리 지연
- \(D_{\text{queue}}\): 큐잉/혼잡 지연
- \(D_{\text{trans}}\): 패킷을 링크로 밀어 넣는 전송 지연 (패킷 길이 / 링크 속도)
- \(D_{\text{prop}}\): 물리 매체를 따라 전파되는 지연

라우팅 알고리즘은 주로 **경로 상의 총 비용(지연/대역폭/정책)**을 최소화하는 방향으로 설계된다.

---

## Other Network-Layer Services

네트워크 계층은 패킷화·라우팅·포워딩 외에도 다양한 기능을 제공한다.

### Addressing (IPv4 / IPv6)

- **IPv4 주소 (32비트)**: `a.b.c.d` 형태, 예: 192.0.2.10
- **IPv6 주소 (128비트)**: `xxxx:xxxx:...` 형태, 예: 2001:db8::1

네트워크 계층은:

- **호스트를 전 세계적으로 식별**할 수 있는 주소 체계를 제공하고,
- 이 주소를 기반으로 라우팅을 수행한다.

IP 헤더의 출발지/목적지 주소 필드 길이:

- IPv4: 32비트 → 최대 주소 수 $$2^{32}$$ (약 43억 개)
- IPv6: 128비트 → $$2^{128}$$, 사실상 고갈 문제 없는 수준

---

### 오류 보고·진단 (ICMP)

IP 자체는 “Best effort”라서 패킷 손실·중복·순서 변경이 자연스럽게 발생할 수 있다.

- IP는 손실을 자동 복구하지 않는다.
- 대신, 오류·진단 메시지를 위한 별도 프로토콜 **ICMP(Internet Control Message Protocol)**를 제공한다.

예:

- **Destination Unreachable**: 라우터가 더 이상 경로를 찾지 못할 때.
- **Time Exceeded**: TTL이 0이 되었을 때 (traceroute에 사용).
- **Echo Request/Reply**: ping에서 사용하는 ICMP 메시지.

이런 메시지는 **네트워크 계층의 상태를 상위 계층/관리자가 알 수 있게 하는 서비스**다.

---

### Congestion Control & QoS (DiffServ, ECN 등)

네트워크 계층은 패킷 헤더에 **우선순위/서비스 클래스**를 표시함으로써
라우터/스위치가 **트래픽을 분류·우선 처리**할 수 있도록 돕는다.

- IPv4/IPv6 헤더에는 **Differentiated Services Field(DS Field)**가 있으며,
  이 중 상위 6비트를 **DSCP**로 사용해 서비스 클래스를 표현한다.
- 혼잡 알림을 위한 **ECN(Explicit Congestion Notification)**도 DS Field의 일부 비트를 사용한다.

예:

- DSCP 46(Expedited Forwarding) → VoIP/실시간 트래픽 우선 처리
- DSCP 0 → Best effort 기본 트래픽

수식 관점에서 보면, 링크에 대해 큐잉 정책을 다르게 설정해
각 서비스 클래스별 **평균 지연**을 다르게 만들 수 있다.

- \(D_{\text{avg}}^{\text{EF}} < D_{\text{avg}}^{\text{BE}}\)
  (EF: Expedited Forwarding, BE: Best Effort)

---

### 보안(Security): IPsec

네트워크 계층에서 동작하는 대표적인 보안 기술이 **IPsec**이다.

- **AH(Authentication Header)**, **ESP(Encapsulating Security Payload)**를 통해
  무결성, 인증, 기밀성을 제공.
- IPv4/IPv6 상에서 동작하며, **Site-to-Site VPN**, **Remote Access VPN** 등에서 사용.

IPsec은 IP 패킷을 다시 감싸서 터널링하는 형태도 가능하며,
네트워크 계층 자체에서 “보안 채널”을 제공하는 셈이다.

---

### 멀티캐스트, Anycast

네트워크 계층은 Unicast(1:1) 외에도:

- **Multicast(1:多)**: 특정 그룹 주소로 패킷을 보내면,
  그 그룹에 가입한 수신자에게만 복사·전달.
- **Anycast**: 여러 노드가 같은 IP를 가지되,
  “가장 가까운(라우팅 상 비용이 최소인) 노드”에게만 패킷을 전달.

DNS 루트 서버들, CDN 엣지 노드들이 Anycast를 많이 활용한다.

---

## Packet Switching

**Packet switching(패킷 교환)**은 메시지를 **짧은 패킷들로 나누어**,
각각을 독립적으로(또는 VC 기반으로) 네트워크를 통해 전달하는 방식이다.

여기에는 두 가지 주요 접근법이 있다.

1. **Datagram approach** — connectionless service
   - 인터넷의 IP 모델.
2. **Virtual-circuit approach** — connection-oriented service
   - ATM, Frame Relay, MPLS-TP 등에서 사용되는 개념.

---

## Datagram Approach: Connectionless Service

### 개념 요약

Datagram 방식의 핵심 특징:

- **사전 연결 설정 없음(No setup)**
  - 각 패킷(datagram)은 독립적으로 취급.
- **각 패킷이 독립적으로 라우팅**
  - 같은 송수신 쌍이라도 각 패킷이 **서로 다른 경로**를 택할 수 있다.
- **Best effort 서비스**
  - 손실·중복·순서 변경 가능.
  - 재전송, 순서 정렬 등은 **상위 계층(TCP 등)**이 담당.
- **IP(IPv4/IPv6)**가 대표적인 datagram network layer이다.

RFC 791/STD 5에서는 **“인터넷 프로토콜은 각 데이터그램을 서로 관계없는 독립적인 엔티티로 다룬다”**고 명시한다.

---

### 예제 토폴로지와 패킷 경로

간단한 네트워크를 가정하자.

```text
Host A --- R1 --- R2 --- R4 --- Host B
          \          /
           \-- R3 --/
```

- R1–R2–R4 경로와 R1–R3–R4 경로 두 개가 있다.
- 라우팅 프로토콜이나 정책에 따라,
  R1은 B로 가는 트래픽을 **두 경로로 나눠 보낼 수도** 있고,
  링크 상태에 따라 어느 순간 R2 방향, 어느 순간 R3 방향을 택할 수도 있다.

Host A가 B에게 순서대로 3개의 패킷을 보낸다고 하자.

1. 패킷 1: A → R1 → R2 → R4 → B
2. 패킷 2: A → R1 → R3 → R4 → B
3. 패킷 3: A → R1 → R2 → R4 → B (예)

도중에 R3–R4 링크가 혼잡하거나 장애이면, 패킷 2가 지연되거나 손실될 수 있다.

- B가 받는 순서는 **3, 1, 2**일 수도 있고,
- **1번 패킷은 누락**될 수도 있다.

이 모든 것이 **네트워크 계층의 정상 동작 범위**이며,
상위 계층 TCP가 재전송/순서 재정렬을 담당하게 된다.

---

### Datagram Forwarding 의사코드 예제

간단한 datagram 라우터를 파이썬으로 흉내 내면:

```python
import ipaddress
import random

class DatagramRouter:
    def __init__(self, name):
        self.name = name
        self.routes = []  # (prefix, next_hop_router)

    def add_route(self, prefix, next_hop):
        self.routes.append((ipaddress.IPv4Network(prefix), next_hop))

    def forward(self, dst_ip, packet_id):
        dst = ipaddress.IPv4Address(dst_ip)
        best = None
        best_len = -1
        for net, nh in self.routes:
            if dst in net and net.prefixlen > best_len:
                best = nh
                best_len = net.prefixlen
        print(f"[{self.name}] 패킷 {packet_id} -> next hop {best.name if best else 'DROP'}")
        if best:
            best.forward(dst_ip, packet_id)

# 라우터 생성

r1 = DatagramRouter("R1")
r2 = DatagramRouter("R2")
r3 = DatagramRouter("R3")
r4 = DatagramRouter("R4")

# 간단히 '라스트' 라우터는 포워딩 대신 도착 처리

def make_sink(name):
    class Sink(DatagramRouter):
        def forward(self, dst_ip, packet_id):
            print(f"[{self.name}] 패킷 {packet_id} 도착 (dst={dst_ip})")
    return Sink(name)

b = make_sink("B")("B")  # 타입 꼬임 피하기 위한 트릭

# 라우팅 테이블 구성 (단순 예)

r1.add_route("10.0.0.0/24", random.choice([r2, r3]))  # 매 호출마다 선택될 수 있다고 가정
r2.add_route("10.0.0.0/24", r4)
r3.add_route("10.0.0.0/24", r4)
r4.add_route("10.0.0.0/24", b)

# A에서 B(10.0.0.1)로 패킷 3개 전송

for i in range(1, 4):
    print(f"\n=== 패킷 {i} 전송 ===")
    r1.forward("10.0.0.1", i)
```

여기서는 `r1.add_route` 부분이 랜덤 choice라고 가정해
각 패킷이 R2 또는 R3를 거칠 수 있게 만들었다고 상상하면 된다.

핵심: **각 패킷은 독립적으로 포워딩**되고,
라우터의 정책/상태에 따라 **선택되는 경로가 달라질 수 있다**는 점이다.

---

### Datagram 서비스의 장단점

**장점**

- **유연성과 확장성**
  - 네트워크 전체가 거대한 “베스트 에포트 클라우드”로 구성되므로,
    새로운 경로를 추가하거나 라우터를 증설하기 쉽다.
- **고장 회복성**
  - 일부 링크/라우터 장애 시에도 다른 경로로 우회할 수 있다.
- **단순한 상태(state)**
  - 라우터는 “연결 상태”가 아니라 **라우팅 테이블만 유지**하면 된다.

**단점**

- **지연·손실·순서 혼란**을 상위 계층에서 처리해야 한다.
- **QoS 보장이 어려움**
  - DiffServ/Traffic Engineering 등 다양한 기술로 보완하지만,
    기본 모델은 여전히 Best effort이다.

---

## Virtual-Circuit Approach: Connection-Oriented Service

Datagram과 달리, **Virtual Circuit(VC)** 기반 패킷 교환은
먼저 네트워크 내에 **논리적 연결(virtual connection)**을 설정하고,
그 후에 해당 경로 위로 패킷을 보내는 방식이다.

대표적인 예:

- **ATM (Asynchronous Transfer Mode)**
- **Frame Relay**
- **MPLS/MPLS-TP Label Switched Path(LSP)**

---

### Virtual Circuit의 기본 아이디어

특징:

- **연결 설정 단계(Setup)**
  - 송신 호스트가 네트워크에 “이 목적지와 통신하겠다”고 요청.
  - 네트워크는 경로를 계산하고, 각 라우터/스위치에
    “이 연결에 대한 상태(state)”를 저장.
- **데이터 전송 단계(Data transfer)**
  - 패킷에는 **목적지 전체 주소 대신 짧은 VC ID(또는 Label)**만 든다.
  - 중간 라우터는 이 VC ID를 보고, 미리 정해진 출력 인터페이스로 포워딩.
  - 모든 패킷은 **동일 경로**를 따라가므로, **순서가 자연스럽게 유지**된다.
- **해제 단계(Teardown)**
  - 통신 종료 시 “disconnect” 요청을 보내고,
    각 노드는 해당 VC 상태를 삭제.

ITU와 IETF 문서에서는 MPLS-TP 같은 기술을
“**Connection-oriented packet-switched network**”라고 정의한다.

---

### VC Switch에서의 상태 테이블

각 VC 기반 스위치/라우터는 **입력 인터페이스 + 입력 VC ID → 출력 인터페이스 + 출력 VC ID** 매핑 테이블을 가진다.

예를 들어, 한 노드의 VC 테이블이 다음과 같다고 하자.

| In IF | In VC | Out IF | Out VC |
|-------|-------|--------|--------|
| 1     | 10    | 3      | 7      |
| 2     | 5     | 4      | 5      |
| 3     | 8     | 1      | 9      |

데이터 플레인에서는:

1. 패킷 헤더에서 “VC ID = 10”을 읽고,
2. (In IF=1, VC=10) 엔트리를 찾아,
3. Out IF=3, Out VC=7 로 바꿔서 포워딩.

이렇게 하면, 전체 경로를 패킷에 싣지 않고도 **간단한 테이블 lookup만으로** 경로를 따라가게 된다.

---

### VC 설정·전송·해제 예제

간단한 4노드 네트워크:

```text
Host A -- R1 -- R2 -- R3 -- Host B
```

1. **설정(Setup)**
   - A가 B로 VC를 설정하고 싶다고 R1에 요청.
   - R1은 R2에 “VC 설정 메시지” 전달, R2는 다시 R3에 전달.
   - 각 라우터는 “입력 인터페이스/VC ↔ 출력 인터페이스/VC” 엔트리를 만든다.
2. **데이터 전송(Data transfer)**
   - A는 VC ID=5를 담은 패킷들을 계속 전송.
   - R1, R2, R3는 사전에 설치된 테이블을 보고, 빠르게 포워딩.
3. **해제(Teardown)**
   - A 또는 B가 “연결 종료” 메시지 전송.
   - 중간 노드들은 해당 VC ID와 관련된 테이블 엔트리를 제거.

---

### 간단 VC 스위칭 의사코드 예제

```python
class VCSwitch:
    def __init__(self, name):
        self.name = name
        self.table = {}  # (in_if, in_vc) -> (out_if, out_vc)
        self.neighbors = {}  # if -> (neighbor_switch, neighbor_if)

    def connect(self, my_if, neighbor, neighbor_if):
        self.neighbors[my_if] = (neighbor, neighbor_if)

    def add_entry(self, in_if, in_vc, out_if, out_vc):
        self.table[(in_if, in_vc)] = (out_if, out_vc)

    def forward(self, in_if, vc_id, payload):
        key = (in_if, vc_id)
        if key not in self.table:
            print(f"[{self.name}] VC {vc_id} unknown on if {in_if} -> DROP")
            return
        out_if, out_vc = self.table[key]
        nbr, nbr_if = self.neighbors[out_if]
        print(f"[{self.name}] (if{in_if},vc{vc_id}) -> (if{out_if},vc{out_vc}) to {nbr.name}")
        nbr.forward(nbr_if, out_vc, payload)

# 스위치들 생성

r1 = VCSwitch("R1")
r2 = VCSwitch("R2")
r3 = VCSwitch("R3")

# 링크 연결

r1.connect(1, r2, 1)
r2.connect(1, r1, 1)
r2.connect(2, r3, 1)
r3.connect(1, r2, 2)

# VC 경로 설정: A-?-?-B 경로라고 가정
# R1: in_if=0(Host A), VC 5 -> out_if=1, VC 10

r1.add_entry(0, 5, 1, 10)
# R2: in_if=1, VC 10 -> out_if=2, VC 20

r2.add_entry(1, 10, 2, 20)
# R3: in_if=1, VC 20 -> out_if=0(Host B), VC 7

r3.add_entry(1, 20, 0, 7)

# A가 보낸 셀/패킷 하나를 R1 입장에서 forward 호출

r1.forward(0, 5, "Hello VC")  # in_if=0, VC=5, payload="Hello VC"
```

여기서는 `in_if=0`과 `out_if=0`을 각각 호스트 A/B에 연결된 인터페이스라고 가정했다.

- A는 항상 VC=5로 보낸다고 생각하면,
- 네트워크 내부에서는 각 홉마다 VC 번호가 5→10→20→7처럼 바뀌면서 전달된다.

---

### Virtual Circuit의 장단점

**장점**

1. **예측 가능한 QoS**
   - 연결을 설정할 때 링크 자원을 예약하거나,
     특정 경로/우선순위를 보장할 수 있다.
   - MPLS-TE, MPLS-TP, LTE/5G의 “Bearer” 개념 등이 이런 방식이다.
2. **패킷 순서 보장**
   - 모두 같은 경로를 가므로, 일반적으로 순서가 유지된다.
3. **헤더가 상대적으로 짧을 수 있음**
   - 전체 IP 주소 대신 짧은 VC ID/Label만 실으면 되므로,
     내부 코어망에서는 처리량을 높이기 유리하다.

**단점**

1. **상태 유지 부담**
   - 각 라우터가 **VC당 상태**를 기억해야 한다.
   - 활성 VC 수가 많으면, 각 노드의 테이블 크기가 증가.
2. **설정/해제 지연**
   - 연결을 만들고 없애는 과정에서 **호출 지연(call setup delay)**이 발생.
3. **고장 시 복잡한 복구**
   - 경로 중간이 끊어지면, 관련 VC들에 대한 재설정·경로 전환이 필요.

단순하게 VC 수를 \(F\), 각 VC가 평균 \(k\)개의 홉을 거친다고 하면,
네트워크 전체에서 유지해야 할 상태 엔트리 수는

$$
N_{\text{state}} \approx F \times k
$$

정도로 증가한다.
대규모 인터넷에서는 \(F\)가 매우 크기 때문에,
“완전한 end-to-end VC 모델”은 확장성 문제가 있다.

---

### Datagram vs Virtual Circuit 요약 비교

| 항목 | Datagram | Virtual Circuit |
|------|----------|-----------------|
| 연결 설정 | 없음 | 필요 (Setup/Teardown) |
| 상태(State) | 라우팅 테이블 중심 | VC마다 상태 엔트리 필요 |
| 경로 | 패킷별로 독립적 선택 가능 | 설정 시 결정된 경로 고정 |
| 순서 보장 | X (상위 계층이 처리) | 일반적으로 O |
| QoS | Best effort, DiffServ 등으로 “클래스” 단위 | 연결 단위 전략/예약에 유리 |
| 확장성 | 매우 좋음 (인터넷 구조에 적합) | 상태 관리 부담으로 제약 |
| 대표 사례 | IP, UDP, TCP over IP | ATM, Frame Relay, MPLS-TP LSP 등 |

현대 인터넷은 **Datagram + 일부 구간에서의 VC-like 기술(MPLS)** 조합을 사용한다:

- 엣지/엔드 호스트: IP datagram, TCP/UDP
- 코어/프로바이더: MPLS Label Switched Path로 트래픽을 “튜널링”
- 필요시 TE/트래픽 엔지니어링, VPN, QoS 정책 적용

---

## 마무리

- **18.1 Network-layer services**
  - Packetizing: 상위 계층 데이터를 IP 데이터그램으로 캡슐화, 필요시 조각화.
  - Routing & Forwarding: 제어 평면에서 경로 계산, 데이터 평면에서 최장 접두사 매칭으로 패킷 전달.
  - Other services: 주소 지정, 오류·진단(ICMP), QoS(DiffServ/ECN), 보안(IPsec), 멀티캐스트 등.
- **18.2 Packet switching**
  - Datagram approach: 연결·상태를 가지지 않는 Best effort 패킷 교환, 인터넷의 기본 모델.
  - Virtual-circuit approach: VC 상태를 유지하고, 고정된 경로를 따라 패킷을 포워딩하는 연결 지향 패킷 교환.

이 장을 기반으로 앞으로 나올 **IP 프로토콜 상세(IPv4/IPv6 헤더)**,
**라우팅 프로토콜(ARP, OSPF, BGP)**,
**혼잡 제어 및 QoS**,
**MPLS/트래픽 엔지니어링** 등을 이해할 수 있게 된다.
