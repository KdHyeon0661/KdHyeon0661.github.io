---
layout: post
title: 데이터 통신 - Connecting Devices and Virtual LANs (3)
date: 2024-08-05 23:25:23 +0900
category: DataCommunication
---
# Forwarding of IP Packets — 목적지 기반, 라벨 기반, 그리고 라우터의 패킷 스위칭 구조

> **포워딩 정보(FIB)**를 실제로 사용해서 **패킷을 어떻게 넘기는지**를 다루는 단계다. 이 장은 “라우터가 실제로 하는 일”의 핵심이다.

구조:

- 18.5.1 Forwarding Based on Destination Address
- 18.5.2 Forwarding Based on Label (MPLS 등)
- 18.5.3 Routers as Packet Switches (라우터 내부 구조)

---

## Forwarding Based on Destination Address

### 라우팅 vs 포워딩

먼저 용어를 분명히 하고 가자.

- **Routing (제어 평면, Control Plane)**
  - OSPF/BGP 같은 라우팅 프로토콜이 **어디로 가는 경로가 좋은지 계산**하는 과정
  - 결과는 **라우팅 테이블(RIB, Routing Information Base)**로 저장
- **Forwarding (데이터 평면, Data Plane)**
  - 실제로 들어온 패킷의 **목적지 주소를 보고**
    “어느 인터페이스로, 어떤 다음 홉(next-hop)으로 보낼지” **즉시 결정하는 동작**
  - 빠른 조회를 위해 별도의 **FIB(Forwarding Information Base)** 자료구조를 사용

RFC 1812(IPv4 라우터 요구사항)는 라우터를
“IP 패킷의 **네트워크 계층 포워딩 기능**을 수행하는 장치”로 정의한다.

즉, **라우팅은 경로를 계산**, **포워딩은 계산된 경로를 이용해 한 홉씩 패킷을 밀어 넣는 행위**다.

---

### 목적지 기반 포워딩의 기본 알고리즘

IPv4/IPv6 라우터가 패킷을 한 홉 포워딩할 때 하는 일을 단계로 써보면:

1. **패킷 수신**
   - 입력 포트에서 IP 헤더까지 디캡슐레이션 (예: 이더넷 프레임 → IP 패킷 추출)

2. **헤더 검사 및 TTL/Hop Limit 처리**
   - IPv4: TTL 1 감소, 0이면 폐기 + ICMP Time Exceeded
   - IPv6: Hop Limit 1 감소, 0이면 폐기
   - IPv4라면 헤더 체크섬 재계산

3. **목적지 주소 추출**
   - IPv4: 32비트 Destination Address
   - IPv6: 128비트 Destination Address

4. **Longest Prefix Match(LPM)로 FIB 조회**
   - FIB에 저장된 prefix 목록 중, **목적지 주소와 가장 긴 prefix가 매칭되는 엔트리**를 찾는다.

5. **결과로 얻은 정보**
   - 출력 인터페이스(예: `GigabitEthernet0/1`)
   - 다음 홉 IP (필요 시)
   - (옵션) QoS 처리, 정책 라우팅 등 부가 정보

6. **링크 계층 캡슐레이션 및 전송**
   - 다음 홉에 대한 MAC 주소(또는 다른 링크 계층 주소)를 ARP/NDP 등으로 알아내고,
   - 해당 목적지 MAC을 넣어 프레임으로 캡슐레이션 후 출력 큐에 넣는다.

---

### Longest Prefix Match 예제

목적지 기반 포워딩의 핵심은 **가장 구체적인(prefix 길이가 가장 긴) 경로를 선택**하는 것이다.

#### 예제 FIB

어떤 라우터의 FIB가 아래와 같다고 하자.

| Prefix            | Prefix length | Next-hop / Out-if         | 설명                      |
|-------------------|--------------|---------------------------|---------------------------|
| 0.0.0.0/0         | 0            | 203.0.113.1, `GE0/0`      | 기본 경로(default route) |
| 192.0.2.0/24      | 24           | 192.0.2.254, `GE0/1`      | 고객 A망                  |
| 192.0.2.128/25    | 25           | 192.0.2.129, `GE0/2`      | 고객 A 내 고가용성망     |
| 198.51.100.0/24   | 24           | 198.51.100.1, `GE0/3`     | 고객 B망                  |

이제 여러 목적지 주소에 대해 LPM을 해보자.

- **패킷 1**: 목적지 192.0.2.5
  - 매칭 가능한 prefix: 192.0.2.0/24
  - 192.0.2.128/25는 192.0.2.128–255 // 5는 이 범위에 속하지 않음
  → **192.0.2.0/24** 선택 → Next-hop 192.0.2.254

- **패킷 2**: 목적지 192.0.2.200
  - 매칭 가능한 prefix: 192.0.2.0/24, 192.0.2.128/25
  - /25가 /24보다 더 긴 prefix → **192.0.2.128/25** 선택 → Next-hop 192.0.2.129

- **패킷 3**: 목적지 8.8.8.8
  - 매칭 가능한 prefix: 0.0.0.0/0 (기본 경로밖에 없음)
  → **0.0.0.0/0** 선택 → Next-hop 203.0.113.1

이렇게 “더 구체적인 경로가 있으면 항상 그것을 우선”하는 규칙이 LPM이다.

---

#### 비트 연산으로 보는 prefix 매칭

IP 주소와 prefix/mask 관계는 수학적으로는 **AND 연산**으로 표현된다.

- 어떤 prefix `P`와 mask `M`에 대해,
- 목적지 주소 `D`가 그 prefix에 속한다는 건

$$
(D \land M) = P
$$

라는 뜻이다.

예: 192.0.2.0/24

- 192.0.2.0 → 11000000 00000000 00000010 00000000
- /24 mask → 11111111 11111111 11111111 00000000

D = 192.0.2.5일 때,

- D & M = 11000000 00000000 00000010 00000000 = 192.0.2.0 → prefix에 속함.

CIDR을 사용하면 prefix 길이가 다양해지므로, 라우터는 이 관계를 빠르게 평가해야 한다.

---

### 실제 구현: 소프트웨어 vs 하드웨어

현대 라우터/스위치는 **소프트웨어만으로 LPM을 하는 경우는 드물고**,
대부분 **하드웨어(FIB, TCAM, 전용 LPM 메모리)**를 사용한다.

- **RIB**: 라우팅 프로토콜(OSPF/BGP 등)이 관리하는 논리 테이블
- **FIB**: 라우터 OS가 RIB에서 “최종 사용” 경로만 골라 압축한 테이블
  - TCAM, 전용 LPM 메모리, 해시 테이블 등으로 구현

예: Cisco 데이터센터 스위치의 FIB 구조

- LPM(Lookup) 테이블: IPv4/IPv6 prefix 저장
- Exact Match(CEM/Hash) 테이블: /32, /128 같은 host route 저장

IEEE/학술 연구에서는 FIB 규모가 수백만 prefix 이상으로 커질 때의 고속 LPM 구현(트라이, TCAM 최적화 등)을 다루며,
실제 상용 라우터에서도 이런 구조가 반영된다.

---

### 파이썬으로 LPM 포워딩 시뮬레이션

간단한 LPM 예제를 코드로 표현해보자.

```python
import ipaddress

# FIB 엔트리: prefix -> (next_hop, interface)

fib = {
    ipaddress.IPv4Network("0.0.0.0/0"):        ("203.0.113.1", "GE0/0"),
    ipaddress.IPv4Network("192.0.2.0/24"):     ("192.0.2.254", "GE0/1"),
    ipaddress.IPv4Network("192.0.2.128/25"):   ("192.0.2.129", "GE0/2"),
    ipaddress.IPv4Network("198.51.100.0/24"):  ("198.51.100.1", "GE0/3"),
}

def lookup(ip_str):
    ip = ipaddress.IPv4Address(ip_str)
    best = None
    for net, (nh, iface) in fib.items():
        if ip in net:
            if best is None or net.prefixlen > best[0].prefixlen:
                best = (net, nh, iface)
    return best

for dst in ["192.0.2.5", "192.0.2.200", "8.8.8.8"]:
    net, nh, iface = lookup(dst)
    print(f"{dst} -> match {net} via {nh} out {iface}")
```

실제 라우터는 이런 로직을 C/ASIC/TCAM 기반으로 하드웨어 수준에서 매우 빠르게 수행한다.

---

## Forwarding Based on Label

이번에는 목적지 주소 대신 **“레이블(label)”**을 사용해 포워딩하는 방식이다.
대표적인 기술이 **MPLS(Multiprotocol Label Switching)**이다.

### 왜 라벨 기반인가?

순수 IP 포워딩의 한계:

- 매 패킷마다 **LPM 검색** → 큰 FIB에서는 비용이 크다.
- IP 헤더만으로 **서비스 클래스, VPN, TE 경로** 등을 표현하기 어렵다.
- 경로를 **굉장히 세밀하게 제어(트래픽 엔지니어링)**하고 싶어도, IP 라우팅만으로는 한계.

MPLS는 **짧은 고정 길이 레이블**을 패킷 앞에 붙여,
라우터가 **레이블로만 빠르게 포워딩**하게 만든다.
IP 패킷이 MPLS 코어 네트워크 안으로 들어가면,
핵심 라우터들은 IP 헤더를 거의 보지 않고 **레이블만 보고 포워딩**한다.

---

### MPLS 라벨 스위칭의 기본 요소

RFC 3031에서 정의하는 MPLS의 핵심 개념들:

- **Label Switch Router(LSR)**
  - MPLS 코어에서 라벨을 기반으로 패킷을 스위칭하는 라우터
- **Label Edge Router(LER)**
  - MPLS 도메인 “입구/출구”에서 IP 패킷 ↔ 라벨 스택 변환을 담당하는 라우터
  - ingress LER, egress LER
- **FEC(Forwarding Equivalence Class)**
  - “동일한 방식으로 포워딩되는 패킷들의 집합”
  - 예: 같은 목적지 prefix, 같은 VPN, 같은 QoS 클래스 등
- **LSP(Label Switched Path)**
  - FEC를 위해 설정된 **레이블 기반 경로**
  - IP routing이나 RSVP-TE, LDP 등에 의해 설정

MPLS 포워딩 테이블은 IP FIB와는 약간 다른 **Label Forwarding Information Base(LFIB)**에 저장된다.

---

### 라벨 스택과 기본 연산

MPLS 헤더는 IP 헤더 앞에 붙는 **라벨 스택** 형태로 존재한다.

각 라벨 엔트리는:

- Label (20bit)
- Exp / TC (3bit, QoS/TOS 관련)
- S (1bit, Bottom-of-Stack 표시)
- TTL (8bit)

LSR가 할 수 있는 연산:

1. **PUSH(라벨 부착)**:
   - 입구 LER가 IP 패킷을 MPLS 도메인에 넣을 때 IP 앞에 라벨을 붙임
2. **SWAP(라벨 교체)**:
   - 코어 LSR가 label X → label Y로 교체
3. **POP(라벨 제거)**:
   - 출구 쪽에서 라벨을 제거하고 IP 패킷으로 다시 나감
   - 또는 penultimate hop popping(PHP)으로 마지막 전 홉에서 POP 수행

---

### 라벨 기반 포워딩 예제

ASCII 토폴로지:

```text
[고객 LAN] -- R1 (Ingress LER) -- R2 (LSR) -- R3 (Egress LER) -- [목적지 LAN]

IP 목적지 prefix: 198.51.100.0/24
MPLS LSP: R1 -> R2 -> R3
```

- R1: ingress LER
- R2: 코어 LSR
- R3: egress LER

각 라우터의 LFIB(단순화):

**R1 (Ingress)**

| In label | In if | Out label | Out if | FEC                |
|----------|-------|-----------|--------|--------------------|
| –        | GE0/0 | 100       | GE0/1  | 198.51.100.0/24    |

동작:

- R1은 198.51.100.0/24로 가는 IP 패킷을 받으면,
  IP 헤더 앞에 **label=100**을 PUSH하고 R2로 전송.

**R2 (Core)**

| In label | In if | Out label | Out if |
|----------|-------|-----------|--------|
| 100      | GE0/0 | 200       | GE0/1  |

동작:

- R2는 label 100이 붙은 패킷을 받으면,
  label 100 → 200으로 SWAP하고 R3로 전송.

**R3 (Egress)**

| In label | In if | Out label | Out if | 동작 |
|----------|-------|-----------|--------|------|
| 200      | GE0/0 | POP       | GE0/1  | 라벨 제거 후 IP로 포워딩 |

동작:

- R3는 label 200이 붙은 패킷을 받으면 POP(제거)한 후,
  남은 IP 패킷을 일반 IP FIB를 사용해 목적지 LAN으로 포워딩.

이 과정에서, R2는 **IP 헤더를 거의 보지 않고 label만 보고 포워딩**하므로,
코어에서 forwarding을 매우 단순/고속화할 수 있다.

---

### 라벨 기반과 목적지 기반의 비교

| 항목                     | 목적지 주소 기반 포워딩                | 라벨 기반 포워딩(MPLS)         |
|--------------------------|-----------------------------------------|---------------------------------|
| lookup key               | IP dest prefix (32/128bit)             | 고정 길이 라벨(20bit)          |
| 테이블 구조              | LPM FIB                                | LFIB (정확 매칭)               |
| 헤더 검사                | IP 헤더 파싱 필요                      | 라벨 헤더만 보면 됨            |
| QoS/TE/VPN               | IP TOS/DSCP, policy routing 등으로 구현 | 레이블로 FEC를 직접 표현       |
| 사용 위치                | 모든 IP 라우터                          | 주로 ISP 코어, VPN, TE         |

실제 대형 ISP 코어 망에서는:

- IP 경계에서 ingress LER가 “목적지 기반”으로 FEC를 결정하고
- 코어에서는 라벨 기반으로 빠르게 포워딩
- egress LER에서 다시 IP로 “목적지 기반” 포워딩

처럼 **두 메커니즘이 혼합**되어 쓰인다.

---

### 간단한 라벨 포워딩 시뮬레이션 코드

```python
# 예제

lfib_R1 = {  # ingress
    "198.51.100.0/24": 100  # 이 prefix는 label 100 부착
}

lfib_R2 = {  # core
    100: 200
}

lfib_R3 = {  # egress
    200: "POP"
}

def ingress_R1(ip_dst):
    # 실제로는 prefix 매칭을 거치지만 단순화해서 exact match 가정
    label = lfib_R1.get("198.51.100.0/24")
    print(f"R1: IP({ip_dst}) -> push label {label}")
    return {"label": label, "ip": ip_dst}

def core_R2(pkt):
    new_label = lfib_R2[pkt["label"]]
    print(f"R2: swap label {pkt['label']} -> {new_label}")
    pkt["label"] = new_label
    return pkt

def egress_R3(pkt):
    act = lfib_R3[pkt["label"]]
    if act == "POP":
        print(f"R3: pop label {pkt['label']} -> IP({pkt['ip']})")
        del pkt["label"]
    return pkt

packet = ingress_R1("198.51.100.5")
packet = core_R2(packet)
packet = egress_R3(packet)
print("Final:", packet)
```

이 코드는 MPLS의 PUSH/SWAP/POP 흐름을 단순하게 보여주는 데 목적이 있다.

---

## Routers as Packet Switches

마지막으로, 라우터를 **“패킷 스위치”**라는 관점에서 본다.

앞 장(8장)에서는 **switching**이란:

- circuit switching
- packet switching
- message switching

중, **packet switching**에 해당하는 구조를 다루었다.
IP 라우터는 바로 이 **packet switching 노드**의 한 형태다.

---

### 라우터 내부 구조 개요

현대 라우터/스위치는 다음과 같이 나뉜다.

1. **입력 포트(Input Port)**
   - 물리 계층(PCS/PMA/PHY)
   - 링크 계층(MAC, 프레이밍)
   - 간단한 FIB 캐시/ACL, QoS 등
2. **스위칭 패브릭(Switching Fabric)**
   - 공유 버스, 크로스바, 공유 메모리 등 구조로
     입력 포트 → 출력 포트로 패킷을 옮기는 역할
3. **출력 포트(Output Port)**
   - 출력 큐, 스케줄러(QoS, priority)
   - MAC/PHY를 통해 실제 전송
4. **라우팅 프로세서(Routing Processor, Control Plane)**
   - OSPF/BGP 등 라우팅 프로토콜
   - RIB 계산, FIB/TCAM 프로그래밍
   - 관리/제어 기능

ASCII 블록 다이어그램:

```text
        +-------------------------------+
        |         Routing CPU           |
        |    (Control Plane, RIB->FIB)  |
        +-------------------------------+
                       |
                +------+------+------+
                |             |      |
        +-------+----+ +------+--+ +-+-------+
In Port |  MAC/PHY  | |Switch  | | MAC/PHY | Out Port
   1    |  + FIB    | | Fabric | | + Queue |   1
        +------------+ +--------+ +---------+
                      ... (여러 포트)
```

**포워딩 과정**은 데이터 평면에서 다음처럼 진행된다.

1. 패킷이 입력 포트에서 들어온다.
2. 입력 포트의 하드웨어가 **현지 FIB(또는 중앙 FIB)를 조회**해
   어느 출력 포트로 보낼지 결정한다.
3. 스위칭 패브릭이 그 포트로 패킷을 전달한다.
4. 출력 포트 큐에서 스케줄링 후 실제 링크로 전송한다.

제어 평면의 CPU는 이 과정에 직접 관여하지 않고,
**FIB/TCAM을 사전에 프로그래밍 해 두는 역할**만 한다.

---

### 라우터 vs 링크 계층 스위치

둘 다 “패킷”을 보고 포트를 결정하는 장치지만, 레이어가 다르다.

| 항목                | 링크 계층 스위치 (LAN 스위치)     | 라우터 (IP 라우터)                  |
|---------------------|------------------------------------|-------------------------------------|
| OSI 계층           | L2 (Data Link)                    | L3 (Network)                        |
| 포워딩 키          | MAC 주소                          | IP 주소 (또는 라벨)                 |
| 브로드캐스트 도메인| 하나의 브로드캐스트 도메인 공유   | 브로드캐스트 도메인 분리           |
| 라우팅 프로토콜    | 없음(보통)                         | OSPF/BGP/IS-IS 등                   |
| FIB의 내용         | MAC → 포트 매핑                    | prefix/라벨 → 포트/다음 홉          |
| TTL/Hop Limit       | 처리 안 함                         | 감소, 0이면 폐기                    |

그러나 **내부 구조(입력 포트, 출력 포트, 스위칭 패브릭)**라는 관점에서는
두 장치 모두 “패킷 스위치”라서 매우 비슷한 구조를 가진다.

---

### 예제: 작은 라우터의 패킷 스위칭 시나리오

다음 구조를 생각해 보자.

```text
LAN A ---[Port1] R ---[Port2]--- LAN B
            |
          [Port3]
            |
           WAN
```

- Port1: LAN A (192.0.2.0/24)
- Port2: LAN B (198.51.100.0/24)
- Port3: WAN(0.0.0.0/0 → ISP)

FIB:

| Prefix           | Out port | Next-hop         |
|------------------|----------|------------------|
| 192.0.2.0/24     | Port1    | 직접 연결        |
| 198.51.100.0/24  | Port2    | 직접 연결        |
| 0.0.0.0/0        | Port3    | 203.0.113.1      |

#### LAN A → LAN B (내부 스위칭)

- 호스트 A(192.0.2.10)가 B(198.51.100.20)로 패킷 전송.
- 패킷은 Port1으로 들어옴.
- 포트1 입력에서 FIB 조회 → 198.51.100.0/24 → Port2 선택.
- 스위칭 패브릭이 Port1 → Port2로 패킷 전달.
- Port2에서 적절한 MAC(호스트 B 또는 게이트웨이)로 프레임 캡슐레이션 후 전송.

이 과정은 **WAN을 거치지 않는 내부 스위칭**이지만,
라벨이 아닌 **IP 목적지 기반**으로 수행된다.

#### LAN A → 인터넷

- 호스트 A가 8.8.8.8로 패킷 전송.
- Port1에서 들어와 FIB 조회 → 0.0.0.0/0 → Port3, next-hop 203.0.113.1
- 스위칭 패브릭이 Port3로 전달하고,
  Port3에서 next-hop의 MAC 주소로 캡슐레이션하여 ISP로 전송.

이 역시 패킷 스위칭의 한 형태다.

---

### 혼잡과 큐잉 관점에서 본 라우터 스위칭

라우터는 각 출력 포트마다 큐를 가지고 있고,
포워딩 결정이 난 패킷들은 그 큐에 쌓인다.

- 큐가 짧으면 → 지연이 적지만, 순간적인 버스트에 취약 (드롭 발생 가능)
- 큐가 길면 → 드롭이 줄지만, 혼잡 시 큐잉 지연이 매우 커진다.

18.3에서 본 것처럼:

- **큐잉 지연(D_queue)**은 트래픽 부하가 링크 용량에 가까워질수록 급격히 증가.
- 라우터의 패킷 스위칭 구조는 **큐 길이, 스케줄링(QoS), WRED/ECN** 등과
  밀접하게 연결된다.

**패킷 스위치로서의 라우터**를 이해하면:

- 왜 **QoS 정책(우선순위 큐, WFQ 등)**이 필요한지,
- 왜 **WRED/ECN 같은 혼잡 신호 기능**이 존재하는지,
- 왜 ISP 코어에서 **라인 카드, 스위칭 패브릭, FIB/TCAM 용량**이 중요한지

를 구조적 관점에서 볼 수 있다.

---

### 간단한 큐잉 시뮬레이션 코드 (개념용)

마지막으로, 루프에서 패킷이 들어오고 나가는 과정을
아주 단순한 코드로 표현해보자.

```python
from collections import deque

queue = deque()
link_rate_pkts_per_tick = 1  # 한 tick 당 1 패킷만 나갈 수 있다고 가정

def arrive(packet):
    queue.append(packet)
    print(f"arrive: {packet}, queue_len={len(queue)}")

def depart():
    if queue:
        pkt = queue.popleft()
        print(f"depart: {pkt}, queue_len={len(queue)}")

# 예: 한 tick에 3개 패킷 도착, 1개만 나갈 수 있다면 큐잉이 증가

arrive("pkt1")
arrive("pkt2")
arrive("pkt3")
depart()
depart()
depart()
```

실제 라우터에서는 이 큐에 대한 **스케줄링, 드롭 정책, ECN 마킹** 등을 통해
“패킷 스위치로서의 행동”을 미세하게 제어한다.

---

## 정리

18.5에서 다룬 핵심 포인트를 다시 정리하면:

1. **Forwarding based on destination address**
   - IP 패킷의 목적지 주소로 **Longest Prefix Match**를 수행해
     FIB에서 출력 포트/다음 홉을 결정한다.
   - 하드웨어 FIB/TCAM, 전용 LPM 메모리 등을 사용해 대규모 prefix에서도
     빠른 조회를 보장한다.

2. **Forwarding based on label**
   - MPLS와 같은 라벨 스위칭 기술을 이용해 **짧은 라벨**로 포워딩한다.
   - ingress LER에서 라벨을 PUSH, 코어 LSR에서 SWAP, egress LER에서 POP.
   - 코어에서는 IP 헤더를 보지 않고 라벨만 보고 포워딩하므로,
     고속·TE·VPN 구현에 유리하다.

3. **Routers as packet switches**
   - 라우터는 입력 포트, 스위칭 패브릭, 출력 포트, 라우팅 프로세서로 구성된
     **패킷 스위치**로 볼 수 있다.
   - 링크 계층 스위치와 구조는 비슷하지만, **포워딩 키(레이어)가 다르다**.
   - 출력 큐, 혼잡, QoS, ECN 등은 모두 “패킷 스위치로서의 라우터” 관점에서
     이해해야 하는 요소들이다.

이제 이 개념을 바탕으로, 다음 장들에서 나올 **구체적인 IPv4/IPv6 헤더 처리, NAT, MPLS TE, VPN, 라우팅 프로토콜(OSPF, BGP)** 등을 보면,
“실제로 라우터가 어떤 테이블을 가지고 어떤 순서로 포워딩을 수행하는지”를 훨씬 입체적으로 이해할 수 있게 된다.
