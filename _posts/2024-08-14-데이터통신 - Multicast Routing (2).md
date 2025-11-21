---
layout: post
title: 데이터 통신 - Multicast Routing (2)
date: 2024-08-14 23:20:23 +0900
category: DataCommunication
---
# Chapter 21. Multicast Routing — 21.3 Intradomain / 21.4 Interdomain

앞에서 21.1–21.2에서 **유니캐스트 vs 멀티캐스트 vs 브로드캐스트**,
**멀티캐스트 주소, IGMP/MLD, RPF, Source-based / Group-shared 트리**까지 정리했다.

이제 실제로 이런 개념을 구현하는 **멀티캐스트 라우팅 프로토콜**을 본다.

- **21.3 Intradomain Multicast Protocols**
  - DVMRP
  - MOSPF
  - PIM (DM/SM/SSM 등)
- **21.4 Interdomain Multicast Protocols**
  - AS 사이에서 소스를 발견하고, 멀티캐스트를 확장하는 메커니즘 (MSDP, MBGP, SSM 기반 등)

---

## Intradomain Multicast Protocols

“**Intradomain**”은 한 **자율 시스템(AS)**, 즉 한 조직 네트워크 내부에서 사용하는 멀티캐스트 라우팅을 말한다.

여기서는 세 가지 대표 프로토콜을 본다.

- **DVMRP**: distance-vector + flood & prune, 초기 MBone 기반
- **MOSPF**: OSPF 링크상태 기반 멀티캐스트 확장
- **PIM**: “Protocol Independent” 멀티캐스트, 오늘날 가장 널리 쓰이는 패밀리

### DVMRP (Distance Vector Multicast Routing Protocol)

#### 1) 개요

DVMRP는 이름 그대로 **distance-vector 스타일 멀티캐스트 라우팅 프로토콜**이다.
RFC 1075에서 정의되었고, 초창기 Internet 멀티캐스트 백본(Mbone)에서 쓰였다.

핵심 특징:

- **RIP 비슷한 distance-vector** 메커니즘으로 “소스까지의 거리”를 계산
- 멀티캐스트는 **Reverse Path Multicasting (RPM)**,
  즉 RPF 기반 **flood & prune** 방식으로 전달
- **Intradomain** 용도 (IGP 같은 역할), 도메인 사이에는 부적합하다고 명시

#### 2) DVMRP의 라우팅/포워딩 구조

DVMRP 라우터는 크게 두 표를 가진다고 보면 된다.

1. **Unicast-style distance vector table**
   - 목적지 네트워크(혹은 소스)를 향한 hop 수와 next-hop
2. **Multicast forwarding cache**
   - 항목: (S,G) → 들어온 인터페이스, 나가는 인터페이스 집합

멀티캐스트 트리는 다음 개념으로 만들어진다.

- 소스 S에서 시작되는 트리:
  - S 방향 unicast distance 정보로 **RPF 인터페이스** 결정
  - 라우터는 S로부터 온 패킷을 RPF 방향에서 받았을 때만 계속 포워딩
- 수신자가 없는 쪽은 **Prune 메시지**로 가지를 잘라냄
- 토폴로지 변화나 Prune 타이머 만료 시 다시 flood

#### 3) Flood & Prune 알고리즘 (RPM)

간단한 예시 토폴로지를 보자.

```text
           R2
         /    \
       R1      R3
         \    /
           R4
```

- S는 R1에 직접 연결된 소스, 수신자는 R3, R4에만 있다고 하자.
- 링크는 모두 동일 비용이라고 가정.

**1단계: Flood**

1. S에서 멀티캐스트 패킷 전송 → R1이 수신
2. R1은 (S까지의 RPF 방향이 S 쪽 인터페이스인 것을 확인) → 통과
3. R1은 R2, R4로 패킷 복제 전송
4. R2는 R1/ R3로, R4는 R1/ R3로 또 복제 → 결과적으로 전체 토폴로지를 flood

**2단계: Prune**

수신자가 없는 분기에서는 라우터가 upstream으로 “이 그룹 안 받겠다”는 뜻의 **Prune**을 보낸다.

- 예를 들어 R2 뒤에는 해당 그룹 G를 수신하는 호스트가 전혀 없다면:
  - R2는 upstream(R1, R3 방향)으로 **Prune(S,G)** 전송
  - R1, R3는 그 인터페이스를 **Outgoing Interface List에서 제거**

**3단계: Truncated tree**

모든 라우터가 이런 식으로 Prune을 보내고 나면,
최종적으로는 **실제 수신자가 존재하는 R3, R4 방향으로만 가지가 남는 트리**가 만들어진다.

수식 관점에서 보면, 노드 수 \(N\), 링크 수 \(E\)인 그래프에서
초기 flood는 \(O(E)\) 패킷을 전송하고, 각 링크에 대해
수신자가 없으면 Prune을 통해 트리를 단축해 나간다고 볼 수 있다.

---

#### 4) 예제: 단순 DVMRP 상태 표

아주 단순화된 forwarding cache 예시(각 라우터에서):

| Router | (S,G) | RPF 인터페이스 | Outgoing Interfaces |
|--------|-------|----------------|---------------------|
| R1 | (S,G) | if_S | {if_R2, if_R4} |
| R2 | (S,G) | if_R1 | {} (Prune 후 비어있음) |
| R3 | (S,G) | if_R4 | {if_host_R3} |
| R4 | (S,G) | if_R1 | {if_R3} |

- R2는 수신자 없으므로 Outgoing이 비어 있다.
- R3는 R4에서 패킷을 받고, 호스트쪽 인터페이스로만 포워딩.

---

#### 5) 장점 / 단점

장점:

- 초기 멀티캐스트 환경(수신자가 매우 많은 Dense 환경)에 적합
- 구현이 비교적 단순 (RIP + RPM)

단점:

- **Dense 가정**이 깨지면 flood 자체가 큰 부담
- 멀티캐스트 그룹 / 소스가 많을수록 라우팅 상태 폭증
- distance-vector 특유의 **느린 수렴** 가능성, 루프/Count-to-infinity 문제
- today: 실무 대규모 네트워크에서는 거의 쓰지 않고,
  PIM 계열이 사실상 대체

---

### MOSPF (Multicast OSPF)

#### 1) 개요

MOSPF(Multicast Open Shortest Path First)는 RFC 1584에서 정의된,
**OSPF 링크 상태 라우팅 프로토콜에 멀티캐스트 라우팅을 추가한 확장 세트**다.

- 별도의 “멀티캐스트용 라우팅 프로토콜”이 아니라,
  **기존 OSPF에 멀티캐스트 확장(LSA 타입 추가)** 를 붙인 구조
- OSPF가 이미 가지고 있는 **LSDB(네트워크 토폴로지)** 를 활용해,
  **(S,G)별 최단 경로 트리(SPT)** 를 계산

핵심 아이디어:

> “Unicast OSPF가 이미 네트워크 전체 지도를 가지고 있다면,
> IGMP로 수집한 그룹 멤버 정보를 LSDB에 얹어서
> 최단 경로 멀티캐스트 트리를 바로 계산하자.”

#### 2) MOSPF 데이터 구조

MOSPF 라우터는 기본적으로 OSPF 라우터와 동일한 구성에 다음이 추가된다.

1. **OSPF LSDB**: 링크 상태, 비용, area, ABR 정보 등
2. **Group-membership LSA (Type 6 등)**:
   - 각 네트워크에 **어떤 멀티캐스트 그룹 멤버가 있는지** 나타내는 LSA
   - IGMP를 통해 호스트의 멤버십을 수집한 후, 이를 LSA로 광고

덕분에 MOSPF 라우터는:

- **네트워크 토폴로지 + 각 링크의 그룹 멤버 정보**를 모두 LSDB로 가진다.

---

#### 3) 멀티캐스트 트리 계산

MOSPF는 패킷이 처음 들어왔을 때, **그룹 G에 대한 (S,G) SPT를 on-demand로 계산**하는 방식을 사용한다.

- unicast OSPF의 SPF 알고리즘(Dijkstra)을 재사용
- 다만, 그룹 G에 대해 **멤버가 있는 네트워크만 leaf로 포함**

연산량을 단순화하면:

- 노드 수 \(N\), 링크 수 \(E\)일 때,
  Dijkstra는 \(O(E \log N)\) 정도
- MOSPF는 **그룹마다** 또는 **(S,G)마다** 이 계산을 수행
  → 그룹/소스 수가 많으면 CPU 부담이 커진다.

#### 4) 예제: 간단한 MOSPF 영역

```text
Area 0
  R1---R2---R3
   |         |
  Net-A    Net-B
```

- Net-A: 호스트들이 그룹 G1에 가입
- Net-B: 호스트들이 그룹 G1, G2에 가입

과정:

1. R1, R3는 각각 IGMP를 통해 Net-A, Net-B의 멤버십(G1, G2)을 수집
2. R1, R3는 이 정보를 **Group-membership LSA로 OSPF area에 광고**
3. 영역 내 모든 MOSPF 라우터는 LSDB에
   - Net-A: G1 멤버
   - Net-B: G1, G2 멤버
   정보를 가지게 된다.
4. S에서 G1으로 멀티캐스트 패킷이 들어오면:
   - S를 root로 하는 SPT 계산
   - leaf로 Net-A와 Net-B가 포함된 tree 도출
   - 그 tree에 따라 포워딩 entry (S,G1)이 구성

---

#### 5) MOSPF의 한계와 현재 위치

장점:

- OSPF 기반 네트워크에서 **추가 프로토콜 도입 없이** 멀티캐스트 가능
- link-state 기반이라 **최단경로 보장**, OSPF의 계층 구조와도 잘 맞음

하지만 단점이 크다:

- 그룹/소스 수가 많아지면, LSDB가 커지고 SPT 계산 비용 급증
- Any-Source Multicast(ASM) 기반으로 설계되어 **SSM + PIM-SM** 시대와 맞지 않음
- 실무에서 크게 확산되지 못했고,
  오늘날 대부분의 벤더/운영자는 **PIM-SM/SSM** 을 사용

즉, MOSPF는 “OSPF + 멀티캐스트를 link-state 관점에서 설계하면 어떤 느낌인지”
이해하기에 좋은 역사적 예제이지만, 최신 네트워크 설계에서는 주연이 아니다.

---

### PIM (Protocol Independent Multicast)

#### 1) 개요

PIM(Protocol Independent Multicast)은 이름 그대로
**특정 unicast 라우팅 프로토콜에 종속되지 않는 멀티캐스트 라우팅 패밀리**이다.

- 토폴로지 정보는 **OSPF, IS-IS, BGP, static route 등**
  어떤 unicast 라우팅 테이블이든 사용 가능
- PIM 자체는 **RPF 검사를 위해 unicast routing table만 참조**
- 여러 모드/변형이 있다.
  - **PIM-DM (Dense Mode)**: flood & prune, dense 환경
  - **PIM-SM (Sparse Mode)**: RP 기반 shared tree + SPT 전환
  - **Bidirectional PIM**: 양방향 공유 트리
  - **PIM-SSM**: SSM (Source-specific) 모델, (S,G) 채널 중심

오늘날 IPTV, 기업 WAN, ISP 멀티캐스트 대부분이 PIM-SM/SSM 기반으로 설계된다.

---

#### 2) 공통 개념 – PIM이 “Protocol Independent”인 이유

PIM은 “멀티캐스트 전용” 라우팅 테이블을 갖지 않는다.

- **Unicast routing table**(RIB/FIB)을 그대로 참조하여
  - 소스 S로 가는 RPF 인터페이스 계산
  - RPF 기반 포워딩 결정

그래서 PIM 라우터는 다음과 같이 동작한다.

1. unicast 라우팅 프로토콜(OSPF, IS-IS, EIGRP, BGP 등)이 먼저 수렴
2. PIM은 해당 라우팅 정보 위에서 RPF 검사를 수행하고 Join/Prune 트리를 구축

---

#### 3) PIM-Dense Mode (PIM-DM)

PIM-DM은 DVMRP와 유사한 **flood & prune, dense-mode** 프로토콜이다.

- 가정: 멀티캐스트 그룹 멤버가 **네트워크 대부분에 걸쳐 분포**
- 동작:
  1. 소스에서 패킷이 나오면 **도메인 전체로 flood**
  2. 멤버가 없는 가지에서는 **Prune**으로 가지 제거
  3. 주기적으로 **State Refresh** 또는 re-flood를 통해 토폴로지 변화 반영

**장점:**

- 구현과 개념이 단순
- 작은 LAN이나 자그마한 도메인에서 빠르게 구축 가능

**단점:**

- sparse 환경에서는 불필요한 flood가 매우 비효율적
- 대규모 도메인, WAN에는 적합하지 않음

##### 예제: PIM-DM 동작 흐름

```text
    R1 --- R2 --- R3
     \            /
        \      /
           R4
```

- S는 R1에, 수신자는 R3와 R4에만 있다고 하자.

1. R1에서 PIM-DM 활성화, S에서 멀티캐스트 트래픽 발생
2. R1은 R2, R4로 패킷 전송 (flood)
3. R2는 R1, R3로 further flood, R4는 R1, R3로 flood
4. R2, R4 뒤에 수신자 없는 방향 인터페이스들은 **Prune** 송신
5. 최종적으로 R1→R2→R3, R1→R4→R3 같이 실제 멤버가 있는 가지만 남는다.

실제 구현에서는 Cisco/Juniper 등에서 **PIM-DM State Refresh** 기능을 제공해,
주기적으로 전체 re-flood 대신 **refresh 메시지**로 Prune 상태를 갱신하여 효율을 개선한다.

---

#### 4) PIM-Sparse Mode (PIM-SM)

PIM-SM은 2020년대 실무에서 가장 널리 쓰이는 멀티캐스트 라우팅 프로토콜이다.

핵심 아이디어:

> 멤버가 “흩어져 있는(sparse)” 환경에서는,
> flood & prune 대신 **중앙 집합점 RP(Rendezvous Point)** 를 두고
> 수신자들이 **명시적으로 Join** 하도록 하자.

구성 요소:

- **RP (Rendezvous Point)**:
  - 그룹 G의 “중앙 지점” 역할
  - 각 수신자 근처 라우터들이 RP 방향으로 Join을 보냄
- **Shared tree (*,G)**:
  - RP를 root로 하는 group-shared distribution tree
- **Shortest Path Tree (S,G)**:
  - 필요 시 소스 S에서 수신자 집합까지 직접 최단 경로 트리로 전환

##### 동작 흐름 요약

1. **수신자 Join**

- 호스트 H가 그룹 G에 가입(IGMP Report)
- H 근처 라우터 R_H는 **PIM Join(*,G)** 메시지를 RP 방향으로 전송
- 경로상의 라우터들은 (*,G) 상태를 만들고, RP로 향하는 shared tree 형성

2. **소스 트래픽 시작**

- S가 G로 패킷 전송 시작
- S 근처 라우터 R_S는 패킷을 **PIM Register** 메시지로 RP에게 캡슐화 전송
- RP는 decap 후 shared tree(*,G)를 통해 패킷을 수신자 쪽으로 전달

3. **SPT 전환 (선택 사항)**

- 수신자 라우터 R_H 입장에서 S→H 트래픽량이 많으면,
  **PIM Join(S,G)** 메시지를 소스 S 방향으로 전송하여
  SPT(Shortest Path Tree)를 형성
- SPT가 형성되면, R_H는 RP 경유 경로를 **Prune(S,G)** 하여
  패킷이 RP를 거치지 않도록 할 수 있다.

##### 예제: PIM-SM IPTV 구조

```text
   IPTV Source S
        |
       R_S
        |
   ----(WAN)-----
        |
       RP (R_RP)
      /   \
   R_H1   R_H2
   /        \
 Host A    Host B
```

- Host A, B는 IPTV 채널 G1(239.1.1.1)에 가입
- R_H1, R_H2는 RP(R_RP)를 향해 Join(*,G1)
- S 근처 R_S는 소스 트래픽을 RP에게 Register
- RP는 (*,G1) shared tree로 트래픽 전달
- 트래픽이 커지면 R_H1, R_H2는 SPT Join(S,G1)으로 전환

현대 IPTV/사내 방송 설계 문서들은 거의 모두 이 패턴을 기반으로 설명한다.

---

#### 5) PIM-SSM / Bidirectional PIM 한줄 정리

**PIM-SSM (Source-Specific Multicast)**

- 채널: (S,G) 형태, **소스가 명시**됨
- 수신자는 “나는 이 소스 S의 G만 받고 싶다”라고 구독
- RP나 MSDP 없이도 interdomain에서 잘 동작
- IPv4: 232.0.0.0/8, IPv6: SSM 전용 멀티캐스트 범위 사용

**Bidirectional PIM**

- RP를 중심으로 하는 **양방향 shared tree**
- 소스와 수신자가 모두 여러 곳에서 발생하는 many-to-many 환경에 유리
- SPT를 따로 만들지 않고, 상태가 단순하지만 딜레이는 늘어날 수 있음

---

#### 6) Intradomain에서의 프로토콜 선택

정리하면, 하나의 AS 내부(Intradomain)에서 멀티캐스트를 설계할 때:

| 환경 | 추천 프로토콜 패턴 |
|------|---------------------|
| 아주 작은 LAN, 모든 곳이 수신자 | PIM-DM 또는 DVMRP (교육/연구/레거시) |
| 일반적인 기업/ISP, IPTV, 사내 방송 | **PIM-SM** + (필요 시 SPT 전환) |
| 대규모 Any-Source Multicast 피하고 싶음, 소스가 명확 | **PIM-SSM** (IGMPv3/MLDv2 기반) |

MOSPF는 **OSPF 세계 안에서의 실험적인 멀티캐스트**라서,
현대 네트워크에서는 거의 사용되지 않고 PIM 계열이 사실상 표준이다.

---

## Interdomain Multicast Protocols

이제 범위를 **도메인(AS) 사이**로 넓힌다.

### 문제 정의: Interdomain Multicast란?

지금까지의 프로토콜(DVMRP, MOSPF, PIM-SM/DM)은
**“한 AS 내부”**에서 멀티캐스트 트리를 만드는 역할이 중심이었다.

하지만 실전에서는:

- AS 65001 (기업 A)
- AS 65002 (ISP X)
- AS 65003 (ISP Y)

등 서로 다른 자율 시스템들이 **멀티캐스트 트래픽을 주고받아야** 한다.

문제:

1. AS 경계에서 **어떤 그룹 G의 소스 S가 어느 도메인에 있는지**를 알아야 한다.
2. 각 AS는 **자체적인 PIM-SM 도메인 / 자기만의 RP**를 가지는데,
   이들을 서로 연결해야 한다.
3. 오늘날 IPv4에서는 ASM보다 **SSM**으로 많이 전환되는 추세라,
   interdomain 설계도 과거와 달라졌다.

이를 위해 등장한 핵심 요소들이:

- **MBGP / Multiprotocol BGP**: 멀티캐스트용 라우팅 정보 운반
- **MSDP (Multicast Source Discovery Protocol)**: ASM에서 RP 간 소스 정보 교환
- **SSM + PIM-SSM**: 소스가 명확한 모델로 MSDP 필요성을 줄임
- **MVPN (Multicast VPN)**: 서비스 제공자(Provider) 내부 IP/MPLS망에서 다중 고객 멀티캐스트를 운반

---

### MBGP / Multiprotocol BGP

BGP-4는 원래 Unicast 용도였지만,
확장 기능인 **Multiprotocol Extensions for BGP-4**(RFC 4760 등)를 통해
**멀티캐스트 라우팅 정보**도 운반할 수 있게 되었다.

아이디어:

- BGP NLRI(Network Layer Reachability Information)에
  **“Address Family Identifier(AF), Subsequent Address Family Identifier(SAFI)”** 필드를 추가
- 예:
  - AFI=IPv4, SAFI=Unicast
  - AFI=IPv4, SAFI=Multicast
  - AFI=IPv6, SAFI=Unicast
  - AFI=IPv6, SAFI=Multicast 등

따라서 ISP/대형 도메인들은:

- **Unicast RIB**: 일반 폴백 라우팅
- **Multicast RIB**: 멀티캐스트 RPF를 위해 별도 유지

이렇게 해서 PIM-SM/SSM 등이 **interdomain에서도 올바른 RPF 인터페이스**를 찾도록 돕는다.

---

### MSDP (Multicast Source Discovery Protocol)

#### 1) 개요

MSDP는 **여러 개의 PIM-SM 도메인(각자 독립적인 RP를 가진 도메인)** 을
서로 연결하기 위한 프로토콜이다.

- Experimental RFC 3618로 정의되었고,
  BCP(RFC 4611)에서는 intra/interdomain 배치 권고 사항을 제시한다.
- IPv4 ASM(Any-Source Multicast) 환경에서 사실상 표준처럼 쓰였다.
- TCP 포트 639 위에서 동작, **각 도메인의 RP들끼리 peer 관계**를 맺는다.

#### 2) 동작 개념

PIM-SM 환경을 생각해보자.

- 각 도메인은 자신의 RP가 있고, 도메인 내부에서만 PIM-SM이 동작
- 외부 도메인에 있는 소스가 **내 도메인의 수신자와 같은 그룹 G에 트래픽**을 보낼 수 있다.
- 그러려면 “이 그룹 G의 소스가 어디에 있는지”를 알아야 한다.

MSDP는 다음 구조를 사용한다.

- 각 도메인의 RP는 **MSDP peer**와 TCP 세션을 형성
- 외부 도메인 RP에서 새로운 (S,G) 소스가 활성화되면,
  - **Source-Active(SA) 메시지**를 MSDP peer들에게 전파
- 다른 도메인의 RP는 SA를 받고,
  - “우리 도메인에 이 그룹 G 수신자가 있나?” 확인
  - 있다면, PIM-SM Join(S,G)를 소스 도메인 쪽으로 보내 SPT를 형성

#### 3) 예제: 두 PIM-SM 도메인 연결

```text
Domain A (AS 65001)         Domain B (AS 65002)
  RP-A  --- MSDP Peering ---  RP-B

Source S_A in A             Receiver R_B in B
Group G = 239.1.1.1
```

과정:

1. S_A가 G에 대해 트래픽 시작
   → Domain A 내부에서 RP-A가 Register를 받고 기존 PIM-SM 트리 형성
2. RP-A는 “새로운 (S_A,G) 소스가 있다”는 **SA 메시지**를 RP-B에게 MSDP로 전송
3. RP-B는 SA 수신 후
   - 자기가 관리하는 도메인 B에 G 수신자가 있는지 확인
   - R_B가 이미 IGMP로 G에 가입되어 있으면,
     - RP-B는 **PIM-SM Join(S_A,G)** 를 Domain A 쪽으로 전송,
     - S_A 도메인까지 SPT를 형성
4. 이제 S_A → R_B 트래픽이 interdomain으로 흐르게 된다.

#### 4) MSDP의 한계와 현대 설계

장점:

- ASM 기반 PIM-SM 도메인들을 쉽게 연결
- 각 도메인은 자체 RP를 유지하면서도, 외부 소스를 수용 가능

단점:

- SA flooding 구조 → 대규모 환경에서 확장성 문제
- 보안/정책 제어 복잡 (원하지 않는 소스를 필터링해야 함)
- IPv6에서는 MSDP가 명시적으로 비권장 (SSM/Embedded-RP 등의 대안 등장)

그래서 최근 베스트 프랙티스 문서들은 **신규 설계에서는 ASM+MSDP 모델 대신 SSM 사용을 권장**하는 경향이 강하다.

---

### SSM( Source-Specific Multicast ) 기반 Interdomain

SSM 모델에서는 수신자가 **(S,G) 채널을 명시적으로 구독**한다.

- IPv4: 232.0.0.0/8 등 SSM 범위 사용
- IGMPv3/MLDv2가 “include S” 형태의 소스 필터링을 제공
- RP나 MSDP 없이, PIM-SSM이 직접 S 쪽으로 Join

Interdomain 관점에서:

- 소스 S의 유니캐스트 경로는 **BGP/MBGP** 를 통해 이미 주어진다.
- 수신 도메인 라우터는 BGP 경로를 보고 “S까지 가는 RPF 방향”을 계산
- PIM-SSM Join(S,G)을 그 경로를 따라 전송해 SPT 형성

따라서:

- **“이 그룹 G에 어떤 소스들이 있는지”라는 MSDP-style discovery가 필요 없다.**
- 대신, 애플리케이션/신호 채널(예: SIP/SDP, HTTP API 등)이
  참여자에게 **S,G 정보를 직접 제공**하는 구조가 된다.

#### 예제: 글로벌 방송 서비스 SSM 설계

```text
  Source S (CDN POP in US)
       |
   ISP/Backbone (MBGP + PIM-SSM)
       |
  ISP-X in Europe
       |
    Receivers (IGMPv3 join (S,G))
```

- 유럽 수신자들은 애플리케이션에서
  “이 채널은 S=203.0.113.10, G=232.0.0.10” 을 전달받는다.
- 수신자 라우터는 IGMPv3 Report를 받으면,
  - PIM-SSM Join(S,G)를 S 방향으로 보낸다.
- BGP가 이미 S까지의 경로를 제공하므로, RPF 계산이 가능.
- MSDP 없음, RP 없음 → 구조 단순, 보안/스케일링 유리.

---

### Multicast VPN (MVPN) 및 프로바이더 네트워크

대형 서비스 제공자(Provider)는 하나의 MPLS/IP 백본 위에서
다수 고객(Enterprise)에게 **L3VPN** 서비스를 제공한다.
이때 **각 고객 VRF마다 멀티캐스트**를 분리해서 제공해야 할 필요가 있다.

이를 위해 나온 것이 **BGP/MPLS IP Multicast VPN (MVPN)** 이다.

개념적으로:

- 고객 VRF 내부에서는 **PIM-SM/SSM** 등 일반 멀티캐스트 프로토콜 사용
- Provider 백본에서는 **BGP + MPLS 터널**로
  여러 고객의 멀티캐스트 트래픽을 분리/전달
- MVPN에서는 **P-tunnel (Provider tunnel)**, **C-tree vs P-tree** 개념 등을 사용

이 부분은 훨씬 복잡하고 VPN/BGP/MPLS 챕터에 속하지만,
Interdomain multicast가 실제로 **상용 서비스(L3VPN, IPTV, VOD)** 에서
어떻게 쓰이는지 이해하는 데 중요하다.

---

### 간단한 “Interdomain ASM vs SSM” 비교 예제

다음은 Python 스타일 의사코드로, interdomain에서
ASM+MSDP와 SSM+PIM-SSM의 상태 복잡도를 아주 거칠게 비교한 것이다.

```python
def asm_msdp_state(num_domains, num_sources, num_groups):
    # 각 도메인 RP가 SA를 교환한다고 가정
    sa_state = num_sources * num_groups          # 전체 (S,G)
    msdp_peers = num_domains * (num_domains - 1) // 2  # 완전 mesh라고 가정
    return sa_state, msdp_peers

def ssm_state(num_domains, num_sources, num_groups, receivers_per_group):
    # SSM에서는 MSDP 없음, 도메인 수는 RPF에 영향만 줌
    # 상태는 단순히 수신자 도메인 내부 PIM-SM/SSM 상태로 근사
    s_g_state = num_sources * num_groups * receivers_per_group
    return s_g_state

domains = 10
sources = 100
groups = 10
receivers = 5

asm_state, asm_peers = asm_msdp_state(domains, sources, groups)
ssm_state_count = ssm_state(domains, sources, groups, receivers)

print("ASM + MSDP 상태 개략:")
print("  (S,G) SA 상태 =", asm_state)
print("  MSDP Peering =", asm_peers, "sessions")
print("SSM 상태 개략:")
print("  (S,G) per receiver =", ssm_state_count)
```

이 코드는 단순한 근사에 불과하지만,
- ASM+MSDP에서는 **전체 소스-그룹 조합 (S,G)** 에 대해 SA 상태를 유지하고
- 도메인 수가 늘면 **peering 세션 수가 급증**한다는 인상을 주고,
- SSM에서는 “수신자 수 × (S,G)” 정도로 상태를 생각하면 된다는 감각을 준다.

실제 설계에서는 이보다 훨씬 복잡한 요소(타이머, 정책, RPF 인프라)가 있지만,
왜 현대 문서들이 **SSM + PIM-SSM + MBGP** 조합을 권장하는지를 이해하는 데 도움 된다.

---

## 전체 요약

- **Intradomain 프로토콜**
  - **DVMRP**: RIP 스타일 distance vector + RPM; Dense 환경, 초기 MBone, 현재는 거의 레거시
  - **MOSPF**: OSPF 링크상태 + Group-membership LSA; 이론적으로 깔끔하지만 확장성과 SSM 부재로 실무 채택 적음
  - **PIM 패밀리**:
    - PIM-DM: Dense-mode flood & prune
    - PIM-SM: RP 중심 shared tree + SPT 전환, 오늘날 ASM의 사실상 표준
    - PIM-SSM: 소스 명시 (S,G), RP/MSDP 없이 interdomain까지 간결하게 확장

- **Interdomain 프로토콜**
  - **MBGP(Multiprotocol BGP)**: 유니캐스트/멀티캐스트 RIB를 분리해 RPF 기반 제공
  - **MSDP**: PIM-SM 도메인 간 ASM 소스 발견 프로토콜, SA 메시지로 (S,G) 알림
  - 최근 설계: **ASM+MSDP → SSM+PIM-SSM** 으로 이동,
    BGP/MBGP는 여전히 경로 제공, MVPN은 MPLS 기반 멀티캐스트 VPN 구현에 사용
