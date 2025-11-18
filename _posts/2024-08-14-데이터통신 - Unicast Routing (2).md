---
layout: post
title: 데이터 통신 - Unicast Routing (2)
date: 2024-08-14 21:20:23 +0900
category: DataCommunication
---
# Chapter 20.3 — Unicast Routing Protocols: Internet Structure, RIP, OSPF, BGP-4

이 글은 앞에서 다룬 **유니캐스트 라우팅의 일반 개념(20.1)**,  
**distance vector / link state / path vector 라우팅(20.2)** 위에 올라가는  
**실제 프로토콜 수준의 정리**다.

- 먼저 **Internet structure**에서 오늘날 인터넷이  
  자율 시스템(AS), ISP 계층, IGP/EGP 구조로 어떻게 조직되는지 보고,
- 내부 라우팅용 **RIP**(distance vector),
- 내부 라우팅용 **OSPF**(link state),
- 도메인 간 라우팅용 **BGP-4**(path vector)

---

## Internet Structure

### 1.1 자율 시스템(AS)과 ISP 계층 구조

인터넷은 하나의 거대한 “평면 네트워크”처럼 보이지만,  
실제로는 **수만 개의 자율 시스템(AS, Autonomous System)** 이  
**정책(BGP 정책)** 으로 서로 연결된 구조다.:contentReference[oaicite:0]{index=0}  

- **AS 정의**  
  - 하나의 운영 조직이 관리하는 IP prefix + 라우터 집합  
  - 외부에 **일관된 라우팅 정책(one routing policy)** 을 제시  
  - 각 AS는 고유한 **ASN(Autonomous System Number)** 을 가진다.:contentReference[oaicite:1]{index=1}  

- **규모**  
  - 16-bit ASN에서 시작했지만, 이후 32-bit ASN으로 확장  
  - 2025년 기준, 전 세계에 **대략 12만 개 정도의 ASN이 할당**되어 있다.:contentReference[oaicite:2]{index=2}  

#### 1.1.1 ISP 계층: Tier-1 / Tier-2 / Tier-3, Stub / Transit

연구와 운영 보고서에서는 전통적으로 인터넷을 다음과 같이 설명한다.:contentReference[oaicite:3]{index=3}  

| 종류 | 설명 | 예 |
|------|------|----|
| **Tier-1 ISP (백본)** | 전 세계적으로 광범위하게 존재, 다른 Tier-1과는 **정산 없는 상호접속(settlement-free peering)**. 상위 공급자에게 transit을 사지 않음. | Lumen, Cogent, Hurricane Electric 등 |
| **Tier-2 ISP** | 상당히 크지만 글로벌 전체는 아님. 다른 Tier-2와 상호 피어링 + 일부 트래픽은 Tier-1에 유료 transit | 대륙/지역 규모 ISP |
| **Tier-3 ISP** | 지역/국가 수준 접근망, 보통 상위 ISP에 transit만 구매 | 로컬 ISP, 케이블/광 회선 사업자 |

AS 관점에서는 다음 네 가지 타입으로도 부른다.:contentReference[oaicite:4]{index=4}  

- **Stub AS**:  
  - 오직 **상위 ISP 하나만**을 통해 인터넷에 나가는 고객 네트워크  
  - 다른 AS의 트래픽을 “중간에서 전달(transit)”하지 않음  
- **Multihomed AS**:  
  - 여러 상위 ISP와 연결(다중 접속)하지만 **자체는 transit 제공 X**  
- **Transit AS**:  
  - 다른 AS들 간의 트래픽을 전달하는 역할  
  - Tier-1, Tier-2 ISP가 여기에 속함  
- **IXP(Internet Exchange Point) AS**:  
  - 여러 ISP/CDN이 접속하는 교환 지점  
  - 서로 간 트래픽을 로컬에서 교환해 지연과 비용을 줄임

이 구조 덕분에 경로는 흔히 다음과 같이 형성된다.

> 가정용 PC (Stub AS) → 지역 ISP (Tier-3) → 지역/대륙 ISP (Tier-2) →  
> 글로벌 백본 (Tier-1) → 다른 지역 ISP들… → 목적지 Stub AS

실제 측정 연구에서는 이 계층 구조가 완전히 “나무형 트리”라기보다는  
**계층 + 피어링이 섞인 scale-free 그래프**에 가깝다는 결과도 많이 보고된다.:contentReference[oaicite:5]{index=5}  

---

### 1.2 IGP vs EGP, 그리고 유니캐스트 라우팅 프로토콜의 위치

AS 내부와 AS 사이에서 쓰는 라우팅 프로토콜이 다르다.

| 범위 | 용어 | 대표 프로토콜 | 라우팅 방식 |
|------|------|----------------|-------------|
| **AS 내부** | **IGP (Interior Gateway Protocol)** | RIP, OSPF, IS-IS, EIGRP | distance vector 또는 link state |
| **AS 간** | **EGP (Exterior Gateway Protocol)** | BGP-4 | path vector |

- **IGP**는 빠른 수렴, 단일 관리자, 비교적 단순한 정책이 목표  
- **EGP(BGP)**는  
  - 전 세계 다른 조직과의 연결,  
  - 규모(수십만 prefix),  
  - 사업/정책(누구를 통해 나가고 누구는 거부할지)  
  를 다루기 때문에 **정책 기반 routing** 이 핵심이다.:contentReference[oaicite:6]{index=6}  

이제 구체적인 프로토콜들을 본다.

---

## Routing Information Protocol (RIP)

### 2.1 개요

RIP는 가장 오래된 IGP 중 하나로, **distance vector 라우팅**의 대표 예다.

- **metric**: 오직 **hop count** (라우터 개수)
- **최대 15 hop** 까지만 도달 가능, 16은 무한대(도달 불가)로 간주:contentReference[oaicite:7]{index=7}  
- **주기적 업데이트**: 기본 30초마다 전체 라우팅 테이블을 이웃에게 광고
- **RIP v1**: classful, VLSM 지원 X  
- **RIP v2**: classless, VLSM, 인증 등 지원 (RFC 2453):contentReference[oaicite:8]{index=8}  

distance vector의 Bellman-Ford 식을 상기해 보자.

$$
D_x(y) = \min_{v \in N(x)}\{ c(x,v) + D_v(y)\}
$$

- \(D_x(y)\): x에서 y까지의 최소 비용  
- \(c(x,v)\): x→v 링크 비용(여기서는 hop 1)  
- \(N(x)\): x의 이웃 집합  

RIP에서는 **모든 링크의 비용이 1** 이므로,

$$
D_x(y) = 1 + \min_{v \in N(x)} D_v(y)
$$

만족하는 값이 곧 **hop count**이다.

---

### 2.2 RIP 동작 예제

#### 2.2.1 작은 토폴로지 예

```text
  Net-A     R1       R2       R3     Net-D
  -----  --[ ]------[ ]------[ ]--   -----
           |                   |
         Net-B               Net-C
```

- R1은 Net-A, Net-B에 directly connected
- R3는 Net-C, Net-D에 directly connected
- 링크는 모두 1 hop

초기(직접 연결만 아는 상태에서):

| Router | Destination | Next hop | Metric (hop) |
|--------|-------------|----------|--------------|
| R1 | Net-A | Direct | 0 |
| R1 | Net-B | Direct | 0 |
| R2 | (없음) | - | - |
| R3 | Net-C | Direct | 0 |
| R3 | Net-D | Direct | 0 |

1) **R1이 R2에게 업데이트 전송**

- R1은 자신의 테이블( Net-A, Net-B )을 R2에게 전송  
- R2는 “R1을 통해 갈 수 있다”는 의미로 **+1 hop** 해서 저장

| Router | Destination | Next hop | Metric |
|--------|-------------|----------|--------|
| R2 | Net-A | R1 | 1 |
| R2 | Net-B | R1 | 1 |

2) **R3도 R2에게 업데이트 전송**

- R3 → R2: Net-C, Net-D 광고
- R2 입장: R3를 거쳐서 Net-C, Net-D 도달 가능

| Router | Destination | Next hop | Metric |
|--------|-------------|----------|--------|
| R2 | Net-A | R1 | 1 |
| R2 | Net-B | R1 | 1 |
| R2 | Net-C | R3 | 1 |
| R2 | Net-D | R3 | 1 |

3) **R2가 R1/R3에게 다시 광고**

- R2는 자신이 아는 Net-A,B,C,D 경로를 이웃에게 전송  
- R1은 R2를 통해 Net-C, Net-D 경로를 알게 되고 metric = 2  
- R3도 반대 방향으로 Net-A,B를 2 hop으로 알게 된다.

이런 식으로, 각 라우터가 **이웃으로부터 거리 벡터를 받고, +1을 더해 최소값 선택**을 반복하여 전체 네트워크에 경로가 전파된다.

---

### 2.3 Count-to-Infinity 문제와 15-hop 제한

distance vector의 고전적인 문제는 **경로가 끊겼을 때, 잘못된 경로 정보가 점점 증가하며 수렴이 매우 느려질 수 있다는 것**이다(Count-to-Infinity).:contentReference[oaicite:9]{index=9}  

간단한 예:

```text
Net-X -- R1 -- R2
```

- 초기:  
  - R1: Net-X (metric 0, direct)  
  - R2: Net-X (metric 1, via R1)

이제 Net-X가 R1에서 끊어졌다고 하자.

1) R1은 Net-X를 잃었지만,  
   **R2가 “Net-X까지 1 hop”** 이라고 광고하면  
   R1은 “R2를 통해 Net-X까지 2 hop”이라고 오해할 수 있다.  

2) 반대로, R2도 R1의 정보를 기반으로 metric을 늘리며 오해할 수 있고,  
   이렇게 둘 사이에서 metric이 3,4,5,… **무한대로 증가**한다.

이를 막기 위해 RIP에서는:

- **최대 hop 수를 15로 제한** (16은 무한대)  
- **split horizon, poisoned reverse, triggered update** 등을 도입하여  
  루프와 count-to-infinity를 완화한다.:contentReference[oaicite:10]{index=10}  

---

### 2.4 간단한 설정 예 (Cisco 스타일)

```text
router rip
 version 2
 network 10.0.0.0
 network 192.168.1.0
 no auto-summary
```

이 설정의 의미:

- RIP v2를 사용 (classless, VLSM 가능)  
- 10.0.0.0/8, 192.168.1.0/24가 붙어 있는 인터페이스에서 RIP를 활성화  
- classful auto summary 비활성화 (CIDR 기반 요약을 명시적으로 설정)

운영 시 주의점:

- 고속 백본이나 대규모 네트워크에서는  
  15-hop 제한, 느린 수렴 때문에 RIP 대신 OSPF/IS-IS를 사용한다.:contentReference[oaicite:11]{index=11}  

---

## Open Shortest Path First (OSPF)

### 3.1 개요

OSPF는 현대 IGP의 사실상 표준 중 하나로,  
**link state 라우팅 프로토콜** 이다.

핵심 특징:​:contentReference[oaicite:12]{index=12}  

- 전체 토폴로지를 **링크 상태 데이터베이스(LSDB)** 로 유지  
- 각 라우터가 Dijkstra **SPF(Shortest Path First)** 알고리즘 실행  
- **area** 기반 계층 구조: Backbone area(0) + 여러 내부 area  
- cost metric은 보통  
  $$\text{cost} = \dfrac{\text{reference\_bandwidth}}{\text{interface bandwidth}}$$  
  와 같이 정의 (예: 100 Mbps 기준 1, 1 Gbps는 1로 수동 튜닝)

### 3.2 OSPF 동작 단계

일반적인 설명은 다음 5단계로 요약할 수 있다.:contentReference[oaicite:13]{index=13}  

1. **프로세스 활성화 / Router ID 결정**  
2. **Neighbor adjacency 형성**  
3. **LSA 교환 및 LSDB 구축**  
4. **SPF 실행 → 최단 경로 트리(SPT)** 계산  
5. **Routing table 갱신**

#### 예제: 4-노드 OSPF 영역

```text
    R1----R2
     \    /
      \  /
       R3
       |
       R4
```

1) **이웃 발견 & Hello**  
   - 각 라우터는 OSPF Hello 패킷을 멀티캐스트로 송신  
   - Hello 설정이 맞으면(hello/dead interval, area, 인증 등) neighbor 형성  

2) **Database Description / LSA 교환**  
   - “나는 이런 링크 상태 정보를 가지고 있다”는 meta 정보를 교환  
   - 필요한 LSA를 서로 요청/응답하여 LSDB를 동기화  

3) **SPF 계산**  
   - R1 기준으로 보면, R1이 root인 그래프에서  
     각 노드로 가는 최소 cost 경로를 Dijkstra로 계산  
   - 예를 들어 모든 링크 cost=10이라면,  
     - R1→R2: cost 10  
     - R1→R3: cost 10  
     - R1→R4: cost 20(R1→R3→R4)  

4) **라우팅 테이블 설치**  
   - best path만 실제 RIB(라우팅 테이블)에 설치  

---

### 3.3 OSPF 영역(Area)와 LSA 타입

OSPF는 **대규모 네트워크를 area로 분할**하여 확장성과 안정성을 높인다.:contentReference[oaicite:14]{index=14}  

- **Backbone area (0)**: 모든 다른 area를 연결하는 중심  
- **Regular area**, **Stub area**, **NSSA** 등 다양한 변형

LSA 타입 예시(IPv4 OSPFv2 기준):​:contentReference[oaicite:15]{index=15}  

| LSA Type | 이름 | 설명 |
|---------|------|------|
| 1 | Router LSA | 동일 area 내에서 라우터의 인터페이스/링크 정보를 광고 |
| 2 | Network LSA | DR(Designated Router)가 브로드캐스트/NBMA 네트워크 정보를 광고 |
| 3 | Summary LSA | ABR가 한 area의 정보를 다른 area로 요약 광고 |
| 4 | ASBR Summary LSA | ASBR을 향한 경로 요약 |
| 5 | AS External LSA | OSPF 도메인 외부(예: BGP)에서 학습한 경로 광고 |

Area 경계(ABR)에서 예를 들면:

- Area 1 내부 prefix들: 10.1.0.0/24, 10.1.1.0/24, 10.1.2.0/24  
- ABR는 이를 **10.1.0.0/22** 같은 요약 경로로 area 0에 광고할 수 있다.

---

### 3.4 OSPF와 RIP 비교 예제

같은 토폴로지에서 RIP vs OSPF를 비교해보자.

```text
     1Gbps          10Mbps
LAN-A ---- R1 ----- R2 ---- LAN-B
```

- 링크 R1-R2: 1Gbps  
- LAN-A, LAN-B는 각각 스위치 뒤의 네트워크라고 하자.

**RIP**:

- 모든 링크의 비용이 동일(1)  
- R1, R2는 서로를 거쳐서 상대 네트워크를 hop=1로 인식  
- **대역폭 차이를 고려하지 못함**

**OSPF**:

- reference bandwidth = 100 Mbps라고 할 때,  
  - 1Gbps 링크 cost = 1  
  - 10Mbps 링크 cost = 10  

- 만약 R1과 R2 사이에 다음과 같이 두 경로가 있다고 하자.

```text
R1 ---1Gbps--- R2
R1 ---10Mbps--- R2
```

- RIP: 둘 다 hop=1, 차이를 구분 못함  
- OSPF: 1Gbps 경로 cost=1, 10Mbps 경로 cost=10 → 항상 1Gbps 경로 선택

이처럼 OSPF는 **토폴로지 + 링크 특성(대역폭 등)** 을 모두 반영한  
최단 경로 계산이 가능하다.

---

### 3.5 간단한 OSPF 구성 예 (Cisco 스타일)

```text
router ospf 1
 router-id 1.1.1.1
 network 10.0.0.0 0.0.0.255 area 0
 network 10.0.1.0 0.0.0.255 area 1

interface GigabitEthernet0/0
 ip ospf cost 1

interface FastEthernet0/1
 ip ospf cost 10
```

의미:

- OSPF 프로세스 ID 1, Router ID 1.1.1.1  
- 10.0.0.0/24 인터페이스는 area 0, 10.0.1.0/24는 area 1  
- 1Gbps 링크에는 cost 1, 100Mbps 링크에는 cost 10을 수동 설정

실무에서는 **area 설계, 요약, Stub/NSSA 활용**을 통해  
라우팅 테이블과 LSDB 크기를 관리한다.:contentReference[oaicite:16]{index=16}  

---

## Border Gateway Protocol Version 4 (BGP-4)

### 4.1 개요와 역할

BGP-4는 **인터넷 상호 연결의 핵심 프로토콜**로,  
서로 다른 AS 사이에서 **IP prefix 도달성(reachability)** 을 교환한다.:contentReference[oaicite:17]{index=17}  

- **EGP(Exterior Gateway Protocol)** 의 사실상 유일한 표준  
- **path vector 라우팅**: AS-PATH 등 경로 속성을 포함한 형태로 경로 광고  
- **TCP 포트 179**로 피어 간 세션을 형성  
- IGP(예: OSPF) 위에서 동작하며, IGP가 “내부 경로 비용”을 제공하면  
  BGP는 “어느 출구로 나갈지, 어떤 AS를 통해 갈지”를 정책으로 결정

BGP-4의 큰 발전 포인트:​:contentReference[oaicite:18]{index=18}  

- **CIDR(Classless Inter-Domain Routing)** 지원 → 경로 요약, 라우팅 테이블 크기 감소  
- **Multiprotocol BGP (MP-BGP)** 확장으로 IPv4/IPv6, VPN, Multicast 등 다양한 주소 패밀리 운반

---

### 4.2 BGP Path Vector 모델

BGP는 distance vector처럼 **이웃으로부터 경로를 받지만**,  
“**어떤 AS들을 거쳐 왔는지(AS-PATH)**”를 함께 싣는다는 점이 다르다.:contentReference[oaicite:19]{index=19}  

각 경로에는 다음과 같은 속성이 붙는다(중요한 것만):

- **AS-PATH**: 이 경로가 지나온 AS 번호들의 리스트  
- **NEXT-HOP**: 다음 홉 라우터의 IP  
- **LOCAL_PREF**: AS 내부에서 선호도 (값이 **클수록 더 선호**):contentReference[oaicite:20]{index=20}  
- **MED(Multi-Exit Discriminator)**: 인접 AS에게 “이 링크를 우선 사용해 달라”는 힌트 (작을수록 선호):contentReference[oaicite:21]{index=21}  
- **COMMUNITY**: 라우팅 정책 제어용 tag  
- 기타: ORIGIN, WEIGHT(벤더 종속) 등

BGP의 경로 선택 과정은 구현마다 조금씩 다르지만,  
전형적으로 다음 순서를 따른다.:contentReference[oaicite:22]{index=22}  

1. **가장 높은 LOCAL_PREF**  
2. **가장 짧은 AS-PATH**  
3. **최저 ORIGIN 타입(IGP < EGP < incomplete)**  
4. **최저 MED**  
5. **eBGP 경로 > iBGP 경로**  
6. **가장 낮은 IGP metric (NEXT-HOP까지)**  
7. **가장 낮은 BGP router ID** …

---

### 4.3 Internet Structure와 BGP 예제

#### 4.3.1 단순 3-AS 시나리오

```text
   AS 65001 (Stub 고객)
        |
      (eBGP)
        |
   AS 65010 (Regional ISP)
        |
      (eBGP)
        |
   AS 65000 (Tier-1 ISP)
```

- AS 65001: 기업 네트워크, 외부로는 65010 ISP 하나만 사용  
- AS 65010: 지역 ISP, 상위 Tier-1(65000)에 transit 구매  
- AS 65000: 글로벌 백본, 여러 AS와 peering

BGP 경로 예:

- AS 65001이 자사 prefix **203.0.113.0/24**를 65010에 광고  
  - BGP UPDATE:  
    - NLRI: 203.0.113.0/24  
    - AS-PATH: (65001)  
- AS 65010은 이 prefix를 상위 AS 65000에 다시 광고  
  - UPDATE:  
    - NLRI: 203.0.113.0/24  
    - AS-PATH: (65010 65001)

인터넷 다른 지역의 라우터들은  
`AS-PATH: 65000 65010 65001` 을 보고,
- 65001이 최종 목적지 고객(Stub AS)이고
- 65010, 65000을 거쳐 가야 한다는 것을 이해한다.

---

### 4.4 BGP 경로 선택 정책 예제

#### 4.4.1 두 ISP를 통한 멀티홈 AS

```text
      ISP-A (AS 64501)
        /\
       /  \
      /    \
  eBGP    eBGP
    /        \
AS 65050   ISP-B (AS 64502)
(기업 AS)
```

기업 AS 65050은 **ISP-A, ISP-B** 두 곳과 모두 eBGP를 맺는다.

요구사항:

1. **정상 시에는**  
   - 인터넷으로 나가는 기본 트래픽은 **ISP-A**를 우선 사용  
2. **백업** 용으로 ISP-B는 남겨둔다.

구현 아이디어(AS 65050 내부 라우터에서):

- ISP-A로부터 받은 경로는 **LOCAL_PREF = 200**  
- ISP-B로부터 받은 경로는 **LOCAL_PREF = 100**  

```text
# 의사 구성 예 (Junos 스타일 느낌)
policy-options {
  policy-statement FROM-ISP-A {
    term 1 {
      then {
        local-preference 200;
        accept;
      }
    }
  }
  policy-statement FROM-ISP-B {
    term 1 {
      then {
        local-preference 100;
        accept;
      }
    }
  }
}
protocols {
  bgp {
    group ISP-A {
      type external;
      peer-as 64501;
      neighbor 203.0.113.1 {
        import FROM-ISP-A;
      }
    }
    group ISP-B {
      type external;
      peer-as 64502;
      neighbor 198.51.100.1 {
        import FROM-ISP-B;
      }
    }
  }
}
```

BGP path selection 규칙에서 **LOCAL_PREF가 가장 먼저 비교**되기 때문에,  
정상 상황에서는 항상 ISP-A 경로가 선택된다.:contentReference[oaicite:23]{index=23}  

---

### 4.5 BGP와 보안, 안정성 이슈 (간단 요약)

BGP는 기본적으로 **경로 광고를 신뢰**하는 설계라서,  
- 잘못된 설정이나 악의적 AS가 **거짓 경로를 광고(BGP hijack)** 하면  
  전 세계 트래픽이 잘못된 곳으로 흘러갈 수 있다.:contentReference[oaicite:24]{index=24}  

이를 완화하기 위해 최근에는:

- **RPKI(Route Origin Validation)** 기반 원본 인증  
- BGPsec, ROA(Route Origin Authorization)  
- 대형 IX/백본에서의 필터링 정책 강화

같은 방향으로 진화 중이다. (자세한 보안 챕터에서 별도 정리해도 좋다.)

---

## 정리: Internet Structure와 RIP/OSPF/BGP-4의 큰 그림

- **Internet Structure**
  - 수만 개 AS가 **stub / transit / multihomed / IXP** 역할로 연결된 거대한 그래프
  - 상업적 측면에서는 Tier-1/2/3 ISP 계층 구조로 설명 가능:contentReference[oaicite:25]{index=25}  

- **RIP**
  - distance vector, hop count metric, 15-hop 제한  
  - 단순하지만 느린 수렴, 대규모 네트워크에는 부적합

- **OSPF**
  - link state, LSDB + SPF, area 기반 계층 구조  
  - 링크 대역폭, 정책에 따른 cost 조정, 빠른 수렴

- **BGP-4**
  - path vector, AS-PATH/LOCAL_PREF/MED 등 속성 기반  
  - Internet 상호 연결의 핵심 EGP  
  - 정책과 비즈니스 관계를 라우팅에 직접 반영

실무 설계에서는 보통:

- **AS 내부**: OSPF(or IS-IS) + 일부 구간에서 EIGRP/RIP  
- **AS 간**: BGP-4  
- 그리고 필요하면 **정책 라우팅, 트래픽 엔지니어링**,  
  그리고 **보안(RPKI, 필터링)** 을 덧붙여 전체 구조를 완성한다.:contentReference[oaicite:26]{index=26}  

이 글은 20.1, 20.2의 이론을 **실제 프로토콜 수준**으로 끌어내려 정리한 것이고,  
이어지는 장에서는 멀티캐스트, IPv6, MPLS 등 위에서 동작하는  
다른 기능들과 어떻게 맞물리는지도 붙여나가면 좋다.