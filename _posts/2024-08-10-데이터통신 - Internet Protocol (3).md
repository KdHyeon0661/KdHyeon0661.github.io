---
layout: post
title: 데이터 통신 - Internet Protocol (3)
date: 2024-08-10 22:20:23 +0900
category: DataCommunication
---
# Chapter 19.3 Mobile IP — Addressing, Agents, Three Phases, Inefficiency

> 목표: **Mobile IP (특히 IPv4 기반 Mobile IP, Mobile IPv4)**가
> - 이동 단말의 **주소 체계(addressing)**를 어떻게 설계하고,
> - 어떤 **에이전트(agents)**로 역할을 나누며,
> - 실제 동작이 **세 단계(three phases)**로 어떻게 진행되는지,
> - 그리고 이 설계가 왜 **비효율적(inefficiency)**인지를
> 이론·예제·간단 코드와 함께 정리한다.
> Mobile IPv4는 RFC 5944(기존 RFC 3344 개정)에서 규정된 프로토콜이고, IPv6에서는 RFC 6275(Mobile IPv6)로 계승/확장되었다.

---

## Addressing — Home Address와 Care-of Address

### Mobile IP의 기본 아이디어

전통적인 IPv4 라우팅에서는 **IP 주소 = 네트워크 위치**라는 가정이 있다.

- IP 주소의 **네트워크 부분(prefix)**가 어느 서브넷에 붙어 있는지를 나타내고,
- 라우터는 이 prefix에 기반해 경로를 선택한다.

하지만 **노트북·스마트폰** 같은 이동 단말은 **물리적으로 네트워크를 옮겨 다니면서도,**
기존 TCP 연결이 끊기지 않고 유지되기를 원한다.

Mobile IP의 핵심 아이디어는:

> **“단말은 항상 고정된 Home Address로 식별되지만,
> 실제 위치는 Care-of Address로 표현하고,
> Home Agent가 둘 사이를 매핑해서 패킷 경로를 조정한다.”**

즉, 한 시점에 단말은 **동시에 두 개의 주소**를 가진다.

- **Home Address (HoA)**: 고정된 “논리적” 주소. 사용자가 알리는 주소, DNS에 등록되는 주소.
- **Care-of Address (CoA)**: 현재 위치(접속 네트워크)를 반영한 “일시적” 주소.

---

### Home Address

- Home Address는 **“고정된, 라우팅 가능한 IP 주소”**로, 단말의 “집(home) 네트워크”에 속한다.
- 예:
  - Home 네트워크: `198.51.100.0/24`
  - Mobile Node(MN)의 Home Address: `198.51.100.42`
- 이 네트워크에는 MN을 대신해 동작하는 **Home Agent(HA)**가 있다.
- 전 세계 어느 곳에서 패킷을 보내든, 라우팅 시스템은 Home Address의 prefix를 보고
  **Home Agent가 위치한 Home 네트워크**로 패킷을 전달한다.

수식적으로 표현하면, Home Agent는 내부에 다음과 같은 **바인딩 테이블(binding table)**을 가진다.

$$
\text{BindingTable} = \{ (\text{HoA}_i, \text{CoA}_i, \text{Lifetime}_i, \text{Flags}_i, \dots) \}
$$

각 원소는 “어느 Home Address가 어느 Care-of Address로 이동해 있는지”를 나타낸다.

---

### Care-of Address

Care-of Address는 **현재 단말이 접속해 있는 외부(방문) 네트워크에서 유효한 주소**다.

Mobile IPv4에서는 두 가지 형태가 있다.

1. **Foreign Agent Care-of Address (FA CoA)**
   - 외부 네트워크에 있는 **Foreign Agent(FA)**의 IP 주소를 공유해서 사용.
   - 여러 모바일 노드가 같은 FA CoA를 공유할 수 있다.
   - Home Agent는 “단말의 실제 IP” 대신 **FA의 주소**로 터널을 만든다.
   - FA는 터널을 디캡슐레이션한 뒤, 자신이 관리하는 로컬 링크에서 MN에게 IP 프레임을 전달.

2. **Co-located Care-of Address (Co-located CoA)**
   - 단말이 **방문한 네트워크에서 직접 IP 주소를 하나 할당(DHCP, PPP, etc.)** 받아 사용.
   - 이 주소 자체가 Care-of Address가 된다.
   - Home Agent는 이 주소로 직접 터널을 만든다.
   - Foreign Agent가 필수는 아니며, MN이 직접 터널 종단을 담당.

요약하면:

| 종류                | 주소 주체 | 예시               | 특징 |
|---------------------|----------|--------------------|------|
| FA CoA              | Foreign Agent | `203.0.113.1` (FA) | 여러 MN이 공유, FA가 터널 종료 |
| Co-located CoA      | Mobile Node   | `203.0.113.77` (MN) | MN이 자신 IP를 CoA로 직접 사용 |

---

### 예제 시나리오: 집에서 카페로 이동

**환경**

- Home 네트워크: `198.51.100.0/24`, Home Agent: `198.51.100.1`
- Mobile Node의 Home Address: `198.51.100.42`
- 외부(카페) 네트워크: `203.0.113.0/24`, Foreign Agent: `203.0.113.1`

#### 집에 있을 때

- MN은 Home 네트워크에 직접 연결되어 있고, 주소도 `198.51.100.42`.
- 패킷 경로:
  - CN(예: `203.0.113.10`) → … → `198.51.100.42` (MN)
    일반 IP 라우팅과 동일, Mobile IP 필요 없음.

#### 카페로 이동해서 FA CoA 사용

- MN이 카페 Wi-Fi에 접속.
- 카페의 Foreign Agent(FA)가 “여기가 Mobile IP Foreign Network”라는 내용을 광고하고,
  자기 주소 `203.0.113.1`를 Care-of Address로 제공.
- MN은 “나의 HoA=198.51.100.42, CoA=203.0.113.1”이라는 내용을 **Home Agent에 등록**.
- 이제 CN이 `198.51.100.42`로 패킷을 보내면:
  1. 패킷은 여전히 Home 네트워크(HA)로 라우팅.
  2. HA는 자신 테이블에서 `HoA=198.51.100.42 → CoA=203.0.113.1`을 찾는다.
  3. 원래 패킷을 **IP-in-IP 터널**로 캡슐화하여 `203.0.113.1`(FA)로 보낸다.
  4. FA는 캡슐을 벗겨 실제 패킷을 `198.51.100.42` 목적지 프레임으로 MN에게 보낸다.

이렇게 해서 **MN의 Home Address는 그대로 유지되지만**, 실제 위치는 Care-of Address로 표현되고, Home Agent가 둘 사이의 간극을 메운다.

---

## Agents — Mobile Node, Home Agent, Foreign Agent

Mobile IPv4에서는 세 가지 핵심 역할이 있다.

- **MN (Mobile Node)**: 실제로 움직이는 단말 (노트북, 스마트폰 등)
- **HA (Home Agent)**: Home 네트워크의 라우터로, MN의 대리 역할
- **FA (Foreign Agent)**: 방문 네트워크의 라우터로, MN에게 Care-of 서비스를 제공

### Mobile Node (MN)

- 스스로를 항상 **Home Address**로 식별한다.
- 이동 시:
  - 새 네트워크에서 FA를 찾거나, Co-located CoA를 얻고,
  - Home Agent에 **등록 등록(Registration Request)**을 보낸다.
- 데이터 송신 시:
  - 보통은 그냥 **현재 접속한 네트워크의 기본 라우터**로 패킷을 보낸다.
  - 출발지 주소는 여전히 **Home Address** (기본 동작)
    → 이 때문에 “삼각 라우팅” 문제가 발생.

### Home Agent (HA)

RFC 5944 기준으로 Home Agent는 다음 역할을 수행한다.

1. **Home 네트워크에 대한 광고**
   - 라우팅 프로토콜에 Home 네트워크 prefix를 광고한다.
   - 결과적으로 “MN의 Home Address”로 가는 모든 패킷이 Home Agent로 도달.

2. **패킷 가로채기 & 터널링**
   - 목적지가 이동 중인 MN의 Home Address이면,
   - 해당 패킷을 **가로채고(intercept)**,
   - 바인딩 테이블에 등록된 CoA로 **터널링하여 재전송**.

3. **등록(Registration) 처리**
   - MN 또는 Foreign Agent로부터 들어오는 **Registration Request**를 검증:
     - 인증(MD5 기반 인증 확장, RFC 5944에서 요구)
     - HoA, CoA, Lifetime 등 파라미터 검증
   - 성공 시 바인딩 테이블 업데이트 후, **Registration Reply** 반환.

4. **보안**
   - 등록 메시지는 반드시 강한 인증을 거쳐야 한다 (keyed MD5, Mobile-Home Authentication Extension 등).
   - 그렇지 않으면 공격자가 임의의 CoA를 등록해 **트래픽 하이재킹**이 가능하다.

---

### Foreign Agent (FA)

Foreign Agent는 방문 네트워크의 라우터로, MN에게 다음 서비스를 제공한다.

1. **Care-of Address 제공 (FA CoA)**
   - 자신 주소를 FA CoA로 광고한다.
   - 여러 MN이 이 주소를 공유, HA → FA 구간은 **터널 한 개**로 여러 MN을 수용할 수 있다.

2. **Agent Advertisement**
   - ICMP Router Advertisement에 **Mobile IP Agent Advertisement 확장 옵션**을 붙여,
     “나는 HA/FA 역할을 한다, 사용 가능한 CoA는 이것이다, 지원 기능은 이것이다” 등의 정보를 브로드캐스트/멀티캐스트로 알린다.

3. **Registration Relay**
   - MN이 FA를 통해 HA에 **Registration Request**를 보내고,
   - HA의 **Registration Reply**를 다시 MN에게 전달한다.

4. **터널 종단 및 전달**
   - HA가 보낸 IP-in-IP 캡슐화 패킷을 수신.
   - 바깥 IP 헤더(HA→FA)를 벗긴 후, 내부 IP 패킷(MN의 HoA 목적지)을 로컬 링크에서 MN에게 전달.

3GPP Release 17 문서에서도 IPv4 기반 단말을 위해 Mobile IPv4의 HA/FA 개념을 일부 사용하는 구성이 여전히 정의되어 있다(예: PDN Gateway가 HA 기능을 수행).

---

### 전체 구조 그림

텍스트 기반으로 Mobile IPv4 구조를 요약하면:

```text
          +--------------------------- Internet ---------------------------+
          |                                                                |
          |                (packets always go to HoA)                      |
          |                                                                |
   Correspondent Node (CN)                                         Home Agent (HA)
       203.0.113.10                                         198.51.100.1 (router)
          |                                                            |
          |                                                            |
          +----------------- Home Network 198.51.100.0/24 -------------+
                                   |
                                   |
                          Mobile Node (MN)
                         HoA: 198.51.100.42
                         (when at home)

[Later, MN moves]

          +--------------------------- Internet ---------------------------+
          |                                                                |
          |                                                  (Tunnel)      |
          |                                         +-------------------+  |
          |                                         | Outer: src=HA,    |  |
          |                                         |        dst=CoA   |  |
          |                                         | Inner: dst=HoA   |  |
          |                                         +-------------------+  |
          |                                                                |
   Correspondent Node (CN)                                         Home Agent (HA)
          |                                                            |
          +-----------------------------+------------------------------+
                                        |
                                Foreign Network 203.0.113.0/24
                                        |
                                  Foreign Agent (FA)
                                  CoA: 203.0.113.1
                                        |
                                        |
                                  Mobile Node (MN)
                                   HoA: 198.51.100.42
```

---

## Three Phases of Mobile IP Operation

많은 교재와 IETF 문서에서는 Mobile IP 동작을 **세 단계**로 설명한다.

1. **Agent/Network Discovery** — 어디에 어떤 Home/Foreign Agent가 있는지 발견
2. **Registration with Home Agent** — HoA ↔ CoA 바인딩 등록
3. **Tunneling & Indirect Routing** — HA가 CN↔MN 데이터 전송을 중계

아래에서는 각 단계를 예제와 함께 풀어 본다.

---

### Phase 1: Agent / Network Discovery

#### ICMP Router Advertisement + Agent Advertisement

Mobile IPv4는 기존의 **ICMP Router Advertisement/Router Solicitation(RFC 1256)**를 확장해,
라우터가 동시에 “Mobile IP 에이전트” 역할 정보를 광고하도록 한다.

- **Router Advertisement** 메시지 안에 **Mobile IP Agent Advertisement 확장 옵션**이 포함.
- 이 옵션에는 다음과 같은 정보가 들어 있다.
  - 이 라우터가 **Home Agent인지, Foreign Agent인지**(또는 둘 다인지)
  - 사용할 수 있는 **Care-of Address 목록** (FA인 경우)
  - 지원 기능: 예를 들어,
    - 최소/최대 Registration Lifetime,
    - 지원하는 터널링 방식(IP-in-IP, GRE 등),
    - 방문 네트워크 프리픽스 등

Mobile Node는 무선 인터페이스를 켜 두고 이 광고를 **수동적으로 듣거나**,
필요 시 **Router Solicitation**을 보내 “지금 누구 있으면 알려 달라”고 요청할 수 있다.

---

#### 예제 — 카페 입장 시 Agent 발견

- MN이 카페에 들어오자마자 Wi-Fi에 접속.
- 몇 밀리초 후, 카페 라우터(FA)가 다음과 같은 광고를 보낸다고 가정해 보자 (개념적):

| 필드                        | 값                           |
|-----------------------------|------------------------------|
| Router Address              | 203.0.113.1                 |
| Mobile IP Role              | Foreign Agent               |
| Offered Care-of Address(es) | 203.0.113.1 (FA CoA)        |
| Max Registration Lifetime   | 3600 seconds                |
| Supported Tunneling         | IP-in-IP (mandatory), GRE   |

MN은 이 광고를 수신하고:

- “여기는 Mobile IP Foreign Network고, 사용할 CoA는 `203.0.113.1`이구나.”
- “앞으로 Home Agent에게 등록할 때는 이 CoA를 쓰면 되겠다.”
라는 결론을 내려 **다음 단계(Registration)**로 넘어간다.

---

### Phase 2: Registration with Home Agent

Agent를 발견했다면, 이제 MN은 Home Agent에게

> “나 지금 이 CoA에 있으니, 이 쪽으로 내 트래픽을 보내 줘.”

라고 알려야 한다. 이것이 **Registration** 단계다.

#### Registration Request (RRQ)

MN은 **Registration Request(RRQ)** 메시지를 만든다.

필수 필드(개념적 요약):

- Home Address (HoA)
- Home Agent 주소 (HA)
- Care-of Address (CoA)
- Lifetime (이 바인딩을 얼마나 유지해야 하는지, 초 단위)
- Identification (재전송/재사용 방지를 위한 토큰)
- 인증 확장 (MN-HA Authentication Extension, keyed MD5 등)

이 RRQ는 두 가지 경로로 HA에 도달할 수 있다.

1. **Through Foreign Agent**
   - MN → FA: RRQ 전송
   - FA → HA: RRQ를 다시 IP 패킷으로 감싸거나 그대로 포워딩
2. **Direct (Co-located CoA)**
   - MN이 CoA를 직접 갖고 있다면, RRQ를 바로 HA로 보냄

#### Registration Reply (RRP)

Home Agent는 RRQ를 받으면 다음을 수행한다.

1. **인증 검증**
   - MN과 미리 공유한 키로 MD5 기반 인증 확장을 검증.
2. **파라미터 검증**
   - 요청 Lifetime이 허용 범위인지,
   - CoA가 유효한 위치(예: 자신이 알고 있는 FA나 합법적인 주소)인지.
3. **BindingTable 갱신**
   - `(HoA, CoA, Lifetime, flags, …)`를 테이블에 추가/갱신.
4. **Registration Reply 전송**
   - 성공/실패 코드 포함.
   - FA 경유였다면 FA로, 아니면 직접 MN으로 전달.

이 과정을 간단한 Python 스타일 의사 코드로 나타내면:

```python
class HomeAgent:
    def __init__(self):
        self.bindings = {}  # {HoA: (CoA, expires_at, flags, ...)}
        self.shared_keys = {}  # {HoA: key}

    def handle_registration_request(self, rrq, now):
        # 1. 인증
        key = self.shared_keys.get(rrq.home_address)
        if key is None or not verify_md5(rrq, key):
            return RegistrationReply(code="AUTH_FAILED")

        # 2. Lifetime 제한
        max_lifetime = 3600
        lifetime = min(rrq.lifetime, max_lifetime)
        if lifetime == 0:
            # 바인딩 해제 요청
            self.bindings.pop(rrq.home_address, None)
            return RegistrationReply(code="DEREG_OK", lifetime=0)

        # 3. 바인딩 갱신
        expires_at = now + lifetime
        self.bindings[rrq.home_address] = (rrq.care_of_address, expires_at, rrq.flags)

        return RegistrationReply(code="OK", lifetime=lifetime)
```

실제 구현은 훨씬 복잡하지만,
**“RRQ → 인증/검증 → 바인딩 테이블 갱신 → RRP”**라는 흐름이 핵심이다.

---

#### 이동 중 재등록 / 해제

- MN이 **새로운 Foreign Network**로 이동하면:
  - 새로운 FA의 광고를 수신 → 새 CoA 획득 → 다시 RRQ 전송.
  - HA는 기존 바인딩을 덮어쓰거나 삭제 후 새 바인딩을 만든다.
- MN이 **집(Home Network)**으로 돌아오면:
  - 더 이상 Mobile IP가 필요 없으므로 Lifetime=0으로 RRQ를 보내 바인딩을 해제할 수 있다.
  - 이후에는 Home Network에서 일반 호스트처럼 동작.

---

### Phase 3: Tunneling & Indirect Routing

등록이 끝나면, 이제부터 Home Agent는 **MN의 위치(CoA)를 알고** 있게 된다.
이 때 Mobile IP의 가장 중요한 동작이 **터널링(tunneling)**과 **간접 라우팅(indirect routing)**이다.

#### — Indirect Routing

1. Correspondent Node(CN)는 여전히 MN의 **Home Address**(예: `198.51.100.42`)로 패킷을 보낸다.
2. 라우팅 시스템은 이 주소를 보고 패킷을 Home Network, 즉 Home Agent에게 전달.
3. HA는 자신 바인딩 테이블에서
   - `HoA=198.51.100.42 → CoA=203.0.113.1`을 찾는다.
4. HA는 원본 IP 패킷을 **IP-in-IP**로 캡슐화:
   - 외부 IP 헤더: src=HA, dst=CoA
   - 내부 IP 헤더: src=CN, dst=HoA
5. 이 캡슐화된 패킷은 CoA(FA 또는 MN)에게 전달된다.
6. CoA 측에서 터널을 종료(디캡슐레이션)하고, 내부 IP 패킷을 MN에게 전달.

이 과정을 간단히 수식으로 나타내면,

$$
\text{OuterIP}: (s = \text{HA}, d = \text{CoA}),\quad
\text{InnerIP}: (s = \text{CN}, d = \text{HoA})
$$

#### — Direct Routing

기본 Mobile IPv4에서는:

- MN이 CN으로 패킷을 보낼 때 **그냥 현재 네트워크의 기본 라우터**로 보낸다.
- 이 패킷의 **출발지 주소는 여전히 HoA**로 설정하는 것이 전형적인 동작이다.

즉:

- CN → MN: CN → HA → (터널) → CoA → MN
- MN → CN: MN → 현재 네트워크 라우터 → … → CN

이 비대칭 경로가 바로 **Triangle Routing**의 핵심이 된다 (4장에서 자세히).

---

### 간단한 터널링 코드 예시 (개념용)

아주 단순화된 “Home Agent 터널링 로직”을 Python 느낌으로 써 보자.

```python
def tunnel_to_mobile(ip_packet, binding_table):
    # ip_packet.dest = HoA
    hoa = ip_packet.dest
    entry = binding_table.get(hoa)
    if not entry:
        # 등록된 CoA가 없으면 그냥 로컬 네트워크로 포워딩
        forward_normally(ip_packet)
        return

    coa, expires_at, flags = entry
    if time.time() > expires_at:
        # 바인딩 만료
        del binding_table[hoa]
        forward_normally(ip_packet)
        return

    # IP-in-IP 캡슐화
    outer = IPHeader(
        src=HOME_AGENT_IP,
        dst=coa,
        protocol=IPPROTO_IPIP  # IP-in-IP
    )
    tunneled_packet = outer + ip_packet.serialize()
    send_to_network(tunneled_packet)
```

실제 구현은 GRE, NAT, IPsec 등 수많은 요소와 얽히지만,
논리적 핵심은 “**HoA → CoA 바인딩을 보고 IP-in-IP 터널링**”이라는 점이다.

---

## Inefficiency in Mobile IP

Mobile IPv4의 설계는 **“이론적으로는 깔끔하지만, 실제 대규모 인터넷에서 비효율/비현실적인 부분”**이 많이 지적되어 왔다.
IETF의 Mobility 관련 설문(예: RFC 6301)과 다양한 연구 논문들이 이 문제들을 분석한다.

대표적인 문제들을 정리하면 다음과 같다.

1. **Triangle Routing** — CN → HA → MN 간 비효율적인 우회 경로
2. **Tunneling Overhead & MTU 문제**
3. **Signaling Overhead & Handoff 지연**
4. **보안·배포 난이도**
5. **NAT/VPN 환경과의 충돌**

---

### Triangle Routing

Triangle Routing은 다음과 같은 세 지점 경로 때문에 붙은 이름이다.

- CN → HA
- HA → MN
- MN → CN (직접)

#### 예제: 유럽 CN, 미국 HA, 유럽 카페 MN

가정:

- CN: 독일 프랑크푸르트 데이터센터(`203.0.113.10`)
- HA: 미국 동부(버지니아) (`198.51.100.1`)
- MN: 프랑스 파리 카페에서 접속 (CoA `203.0.113.1`)

대략적인 RTT:

- 프랑크푸르트 ↔ 버지니아: 80 ms
- 버지니아 ↔ 파리: 80 ms
- 프랑크푸르트 ↔ 파리: 20 ms

**직접 경로 사용 시** (Mobile IP 없다고 가정):

- CN ↔ MN: 약 20 ms

하지만 Mobile IP Triangle Routing에서는:

- CN → HA: 80 ms
- HA → MN: 80 ms
  → 편도 160 ms, 왕복 320 ms 수준

이것을 수식으로 표현하면,

$$
D_{\text{direct}} \approx d(\text{CN}, \text{MN})
$$

$$
D_{\text{triangle}} \approx d(\text{CN}, \text{HA}) + d(\text{HA}, \text{MN})
$$

위 예에서는

- $$D_{\text{direct}} \approx 20\ \text{ms}$$
- $$D_{\text{triangle}} \approx 80 + 80 = 160\ \text{ms}$$

**8배** 수준의 지연 차이가 발생한다.

#### Route Optimization (특히 Mobile IPv6)

Mobile IPv6(RFC 6275)는 Route Optimization을 통해 이 문제를 줄이려 한다.

- MN이 CN에게 자신의 CoA를 직접 알리는 **Binding Update**를 보내고,
- CN은 이후 패킷을 곧바로 CoA로 보낸다.
- 이로써 CN → HA 단계가 제거되어 직접 경로가 된다.

하지만:

- CN이 MN을 신뢰해야 하고, **보안상 복잡한 인증/검증**이 필요.
- 최적 경로가 항상 “직접 경로”라는 보장은 없어서, 오히려 out-of-order 패킷과 성능 저하가 보고되기도 한다.

결국 **공용 인터넷 전체**에 Route Optimization을 잘 적용하기는 쉽지 않다는 것이 여러 연구의 결론이다.

---

### Tunneling Overhead & MTU 문제

Mobile IP는 HA → CoA 구간에서 **IP-in-IP, GRE 등 터널링**을 사용한다.

#### 헤더 오버헤드

- IP-in-IP: **외부 IP 헤더 20바이트** 추가
- GRE 등 다른 터널: 추가 헤더 더 붙음

즉, 유효 페이로드 크기가 줄어든다.

예를 들어, 이더넷 MTU가 1500바이트이고, IPv4 헤더 20바이트를 가정하면:

- 원래 IP 페이로드: 최대 1480바이트
- 여기에 IP-in-IP 터널을 씌우면:
  - 내부 IP 헤더 20바이트 + 실제 TCP/UDP 등 데이터
  - 외부 IP 헤더 20바이트 추가
  - 유효 데이터 크기는 그만큼 감소, 또는 **추가 단편화(fragmentation)**가 필요해진다.

#### Path MTU Discovery와의 상호작용

- 터널 위에서 다시 Path MTU Discovery(PMTUD)를 수행해야 한다.
- ICMP “Fragmentation Needed” 메시지가 방화벽 등에서 막히면,
  - MTU 블랙홀 같은 문제가 발생할 수 있다.
- RFC 4093, RFC 5265 등은 Mobile IPv4가 VPN/IPsec, NAT 환경을 통과할 때의 문제와 해결책을 별도로 정의할 정도로 복잡하다.

---

### Signaling Overhead & Handoff 지연

Mobile IP는 **움직일 때마다** 다음 작업이 필요하다.

1. 새 네트워크에서 Agent Advertisement를 기다리거나 Solicitation을 보냄.
2. CoA 확보 (FA CoA 또는 Co-located CoA)
3. Home Agent로 Registration Request 전송, 인증/검증, Reply 수신
4. Binding Table 갱신 후에야 **정상적인 데이터 포워딩** 시작

무선 환경에서는:

- L2 handoff(기지국/접속점 변경) 자체도 수 ms~수십 ms가 걸리고,
- 그 뒤 Mobile IP 신호 절차까지 수행해야 하므로,
  - TCP 세션에 **눈에 띄는 지연/손실**이 나타날 수 있다.
- 특히 VoIP, 실시간 스트리밍처럼 민감한 트래픽에 불리하다.

이 문제를 완화하기 위해 Mobile IPv6, Proxy Mobile IPv6, Fast Handover 등 다양한 최적화 기술이 제안되었으나,
대부분 **제한된 도메인(예: 이동통신망 내부, 기업망 내부)**에서만 사용된다.

---

### 보안·배포 난이도

#### Home Agent 글로벌 배포 문제

- 각 MN마다 자신을 대신해줄 **Home Agent**가 필요하고,
- HA는 전세계에서 **라우팅 가능한 주소와 prefix**를 가져야 한다.
- 대규모 ISP/사업자 입장에서, 이런 HA 인프라를 범용 인터넷 전체에 배포하는 것은 비용·복잡도가 크다.

#### 등록 프로토콜 보안

- Registration은 “HoA ↔ CoA 바인딩”이라는 **아주 민감한 정보**를 다룬다.
  - 공격자가 이 정보를 위조하면, 트래픽 하이재킹이 가능.
- RFC 5944는 강한 인증(키 기반 MD5 등)을 필수로 요구한다.
- 하지만 전 세계 이질적인 운영자/망에서 공통의 키 관리·AAA 인프라를 구축하는 것은 쉽지 않다.

#### NAT/VPN과의 충돌

- Mobile IP는 “Home Address = 외부에서 접근 가능한 글로벌 주소”라는 전제를 갖고 있다.
- 대량의 NAT 사용, VPN, 사설망 구성에서는 다음 문제가 발생한다.
  - HoA가 사설 주소이면 외부 CN이 접근 불가.
  - HA/FA/MN 중 하나 또는 모두가 NAT 뒤에 있으면, 터널/등록 메시지 경로가 깨진다.
  - VPN/IPsec과 Mobile IP가 겹치면, 캡슐화/주소 매핑 구조가 매우 복잡해진다.
- IETF는 RFC 4093, RFC 5265 등에서 Mobile IPv4와 VPN/IPsec 공존 문제를 정리하고 있지만,
  현실적으로는 **단말 호스트에서 Mobile IP 스택을 올리는 대신**,
  **네트워크 코어 안에서만 터널을 관리하는 방식(GTP, PMIPv6 등)**이 주류가 되었다.

---

### 현대 인터넷에서의 위치 — 모바일 코어 vs 호스트 기반 Mobile IP

IETF의 “Survey of Mobility Support in the Internet”(RFC 6301)는,
Mobile IPv4/IPv6가 표준으로 존재하지만 공용 인터넷에서 **광범위하게 배포되지는 않았다**고 정리한다.

실제 4G/5G 이동통신망을 보면:

- 단말(스마트폰)은 **일반 IP 호스트**처럼 동작한다.
- 단말의 이동성은 **네트워크 코어 내부**에서 GTP 터널, Proxy Mobile IPv6, 3GPP 전용 프로토콜로 관리된다.
- 즉, 호스트 기반 Mobile IP 대신,
  **“네트워크 기반 모빌리티(Network-Based Mobility)”**가 주류가 되었다.

그래도 Mobile IP의 개념(홈 주소, Care-of Address, home/foreign agent, 바인딩, 삼각 라우팅, 터널링)은:

- 현대 이동성 설계(예: PMIPv6, 3GPP 코어, SD-WAN mobility)의 기본 개념으로 여전히 중요하고,
- 이론·시험·면접에서 “네트워크 계층 수준의 이동성 지원”을 설명할 때 쓰이는 표준 모델이다.

---

## 요약

이번 19.3 절에서는 **Mobile IPv4 기반 Mobile IP**의 구조와 문제점을 정리했다.

1. **Addressing**
   - 단말은 항상 **Home Address(고정)**로 식별된다.
   - 실제 위치는 **Care-of Address(일시)**로 표현되며,
   - Home Agent는 `(HoA, CoA)` 바인딩을 가지고 **둘 사이를 매핑**한다.

2. **Agents**
   - **Mobile Node(MN)**: 실제로 이동하는 호스트.
   - **Home Agent(HA)**: Home 네트워크에서 MN을 대신하는 라우터, 바인딩 관리·터널 종단.
   - **Foreign Agent(FA)**: 방문 네트워크에서 CoA 제공, Registration relay, 터널 종단 담당.

3. **Three Phases**
   - **Agent/Network Discovery**: ICMP Router Advertisement + Agent Advertisement로 HA/FA 발견.
   - **Registration**: MN이 HA에 RRQ를 보내 HoA ↔ CoA 바인딩 등록, HA는 인증/검증 후 RRP.
   - **Tunneling & Indirect Routing**:
     - CN → HA → (터널) → CoA → MN (Downlink)
     - MN → CN (Direct, Uplink)
     - IP-in-IP/GRE 터널 사용, 바인딩 만료·재등록으로 이동성 유지.

4. **Inefficiency in Mobile IP**
   - **Triangle Routing**: CN → HA → MN → CN의 비대칭·우회 경로로 지연 증가.
   - **Tunneling Overhead & MTU**: 추가 헤더 + 단편화 + PMTUD 복잡성.
   - **Signaling & Handoff Delay**: 이동 시마다 Agent 발견 + 등록 등 제어 트래픽, 실시간 서비스에 부담.
   - **보안/배포 난이도 & NAT/VPN 문제**: 글로벌 HA, 강한 인증, NAT 뒤 단말 등 현실적 도입 장벽.
   - 현대 4G/5G에서는 호스트 기반 Mobile IP보다는,
     **네트워크 기반 모빌리티(Proxy Mobile IPv6, GTP 터널링 등)**가 실제 이동성 처리를 담당한다.
