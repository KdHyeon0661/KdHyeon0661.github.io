---
layout: post
title: 데이터 통신 - Multicast Routing (3)
date: 2024-08-14 23:25:23 +0900
category: DataCommunication
---
# Chapter 21. Multicast Routing — 21.5 IGMP

멀티캐스트의 **“가장 아래”** 에 있는, 호스트와 첫 번째 라우터 사이의 신호 프로토콜인 **IGMP(Internet Group Management Protocol)** 를 정리한다.

> IGMP는 **IPv4 호스트와 라우터가 “어떤 멀티캐스트 그룹을 받고 싶은지”를 서로 알리는 프로토콜**이다.
> 멀티캐스트 라우터는 IGMP 메시지(쿼리/리포트/리브)를 보고, 각 인터페이스별로 **“이 인터페이스 뒤에는 G 그룹 멤버가 있다/없다”** 를 판단한다.

이 절에서는 요구한 세 가지 축을 중심으로 본다.

1. **Messages** — IGMPv1/v2/v3 메시지 타입과 포맷, 실제 동작 예시
2. **Propagation of Membership Information** — 멤버십 정보가 LAN, 라우터, 스위치를 통해 어떻게 전파·유지·갱신되는지
3. **Encapsulation** — IGMP가 IP/Ethernet 프레임에 어떻게 캡슐화되는지, 주요 필드와 캡처 예

---

## IGMP 개요 및 버전 정리

### 1) IGMP의 역할

- 프로토콜 번호: **IP Protocol 2** (ICMP=1, TCP=6, UDP=17)
- 범위: **IPv4 전용** (IPv6는 MLD 사용)
- 기능:
  - 호스트: “**나는 그룹 G(그리고 소스 S 집합)에 관심 있다**”를 **Membership Report**로 알림
  - 라우터: “**이 서브넷에 아직 G를 받는 사람 있냐?**”를 **Query**로 확인
  - 라우터: IGMP 정보 + 멀티캐스트 라우팅 프로토콜(PIM 등)을 결합해 **트리 구축**

### 2) IGMP 버전별 큰 차이

RFC 기준으로 정리하면:

| 버전 | RFC | 주요 특징 |
|------|-----|-----------|
| IGMPv1 | RFC 1112 | 기본 Join/Query, Leave는 타이머 의존 (명시적 Leave 없음) |
| IGMPv2 | RFC 2236 | **Leave Group 메시지 추가**, Group-Specific Query, Querier 선출 규칙 정교화 |
| IGMPv3 | RFC 3376 | **소스 필터링(Source Filtering)**, Include/Exclude 모드, SSM 구현 핵심 |

실무에서는 대부분 **IGMPv2/v3** 가 쓰이며, v3는 IPTV/SSM 환경에서 필수다. Cisco, Juniper, Nokia 등 주요 벤더의 멀티캐스트 가이드는 모두 v2/v3를 중심으로 설명하고 있다.

이제 본격적으로 **Messages → Membership 전파 → Encapsulation** 순으로 들어간다.

---

## IGMP Messages

### 공통 헤더 구조(추상화)

RFC 2236, RFC 3376을 기반으로 IGMP 메시지를 공통된 시각에서 보자.

IGMPv2 기준 단순화된 헤더:

| 필드 | 크기 | 설명 |
|------|------|------|
| Type | 8비트 | 메시지 종류 (Query, Report, Leave 등) |
| Max Response Time | 8비트 | 수신자가 응답해야 하는 최대 시간 (쿼리에서 사용) |
| Checksum | 16비트 | IGMP 헤더의 1의 보수 합 |
| Group Address | 32비트 | 질의/리포트 대상 그룹(0.0.0.0이면 “전체 그룹”) |

IGMPv3에서는 여기에 **Group Record** 배열과 **소스 주소 리스트** 등 더 많은 필드가 추가되어, 한 메시지 안에 여러 그룹/소스 정보를 담을 수 있다.

C 스타일 구조로 보면 대략 이런 느낌이다(실제 RFC 구조를 단순화한 예):

```c
struct igmpv2_header {
    uint8_t  type;
    uint8_t  max_resp_time;
    uint16_t checksum;
    uint32_t group_address; // network byte order
};
```

실제 RFC의 필드 이름과 순서를 참고하는 것이 정확하지만, **Type / MaxRespTime / Checksum / Group Address** 라는 큰 틀은 그대로다.

---

### IGMP 메시지 타입(버전별)

RFC 3376에 따르면 IGMPv3 구현은 이전 버전과 호환을 위해 다음 메시지 타입을 모두 이해해야 한다.

| Type 값 | 이름 | 버전 / 용도 |
|---------|------|-------------|
| 0x11 | Membership Query | v1/v2/v3 공통, 라우터가 멤버십 확인할 때 |
| 0x12 | Version 1 Membership Report | v1 호스트 리포트 |
| 0x16 | Version 2 Membership Report | v2 호스트 리포트 |
| 0x17 | Version 2 Leave Group | v2 명시적 Leave |
| 0x22 | Version 3 Membership Report | v3 리포트 (Group Records 포함) |

다음 섹션에서는 **Query / Report / Leave** 를 중심으로, v2/v3 관점에서 구체적인 예와 함께 본다.

---

### Membership Query

**Membership Query**는 멀티캐스트 라우터(혹은 IGMP Querier)가 보내는 메시지로,
“**이 서브넷에 아직 그룹 G를 받고 싶은 호스트가 있는지**”를 확인하는 역할을 한다.

Query에는 세 가지 타입이 있다 (v3 기준 이름):

1. **General Query** — “이 인터페이스 상의 모든 그룹에 대해 멤버 있냐?”
2. **Group-Specific Query** — 특정 그룹 G 하나에 대해서만
3. **Group-and-Source-Specific Query** — 특정 그룹 G와 소스 S 집합에 대해서만 (v3)

#### General Query

- 목적지 IP: **224.0.0.1 (All Hosts)**
- Group Address 필드: **0.0.0.0** (v2)
- 의미: “이 서브넷에 있는 모든 호스트, 너희가 속한 그룹들에 대해 리포트 보내라”

타이밍:

- Querier 라우터는 **Query Interval** (예: 125초) 에 한 번씩 General Query 브로드캐스트
- Max Response Time 필드는 호스트들이 **응답을 랜덤 지연**시키는 데 사용된다.
  - 예: MaxRespTime=10초 → 호스트는 0~10초 랜덤 타이머 후 Report 전송

#### Group-Specific / Group-and-Source-Specific Query

- **Group-Specific Query** (v2/v3): Group Address에 대상 그룹을 넣고 전송
- **Group-and-Source-Specific Query** (v3): 그룹 + 소스 리스트

주요 용도:

- 특히 v2에서 **Leave Group** 메시지를 받은 후 “정말 이 그룹을 더 이상 아무도 안 듣는지” 확인할 때:
  - Last Member Query Interval 동안, Group-Specific Query를 빠르게 여러 번 전송
  - 응답이 없으면 해당 인터페이스 뒤에는 이제 그 그룹 멤버가 없다고 판단

예를 들어, Cisco/Nokia 등의 장비는 “Last Member Query Interval” 과 “Last Member Query Count” 파라미터로 이 동작을 튜닝하게 한다.

---

### Membership Report

**Membership Report**는 호스트가 “**나는 그룹 G를 받고 싶다**” 또는 “**나는 이 그룹을 더 이상 안 듣는다(IGMPv3에서 Exclude 리스트만 남기기 등)**”를 표현하는 메시지다.

#### IGMPv1/v2 Report (Version 1/2 Membership Report)

- Type 0x12 (v1), 0x16 (v2)
- Group Address: 관심 있는 멀티캐스트 그룹 G
- 소스 필터링 없음 (ASM 모델)

기본 동작:

1. 호스트가 새 그룹 G에 Join하면:
   - 즉시 Membership Report(G)를 한 번 전송
   - 일정 시간 뒤 다시 한 번 (unsolicited report) 보내서 첫 번째가 손실되었을 경우 대비
2. 라우터의 General Query를 수신했을 때:
   - 각 그룹에 대해, 랜덤 타이머를 설정하고 Report 전송
   - 같은 그룹에 대한 다른 호스트의 Report를 먼저 보면, **자기 Report는 억제**(suppress)하여 트래픽 최적화
3. Querier는 “해당 그룹 G에 대한 Report를 직전에 본 인터페이스”를 멤버가 있는 인터페이스로 표시

예를 들어 O’Reilly “Internet Core Protocols” 에서는 IGMPv2 Membership Report가 **“호스트가 그룹 모니터링을 시작하거나, Query를 보고 응답할 때 생성”** 된다고 설명한다.

#### IGMPv3 Report (Version 3 Membership Report)

IGMPv3의 Report(Type 0x22)는 구조가 훨씬 복잡하다.

포인트:

- 하나의 메시지에 **여러 Group Record** 포함 가능
- 각 Group Record는:
  - 그룹 주소 (G)
  - 소스 주소 리스트 {S1, S2, …}
  - **Record Type (모드)**:
    - MODE_IS_INCLUDE, MODE_IS_EXCLUDE
    - CHANGE_TO_INCLUDE_MODE, CHANGE_TO_EXCLUDE_MODE
    - ALLOW_NEW_SOURCES, BLOCK_OLD_SOURCES 등

즉, v3 Report는

> “나는 그룹 G를, **이 소스들만 포함해서(INCLUDE)**
> 혹은 **이 소스를 제외하고(EXCLUDE)** 받겠다”

를 정확하게 표현할 수 있다. 이를 통해 **SSM(예: 232.0.0.0/8)** 같은 “소스 특정 멀티캐스트” 모델을 그대로 구현한다.

##### 예제: IGMPv3 Report 구조 (추상)

중요 필드만 추린 구조:

```c
struct igmpv3_report {
    uint8_t  type;        // 0x22
    uint8_t  reserved1;
    uint16_t checksum;
    uint16_t reserved2;
    uint16_t num_group_records;
    struct group_record records[];
};

struct group_record {
    uint8_t  record_type;     // MODE_IS_INCLUDE, etc.
    uint8_t  aux_data_len;
    uint16_t num_sources;
    uint32_t group_address;   // G
    uint32_t source_addrs[];  // S1..Sn
    // aux data ...
};
```

실제 구현에서는 이 구조를 파싱하여, 라우터가
**(인터페이스, 그룹 G) → [INCLUDE 리스트 / EXCLUDE 리스트]** 를 갱신한다.

---

### Leave Group

IGMPv1에는 **명시적 Leave 메시지가 없고**, 단지 Report가 더 이상 보이지 않을 때 타이머로 멤버십 만료를 판단했다.
IGMPv2는 **Type 0x17 Leave Group** 메시지를 도입해서 반응 속도를 높였다.

- 목적지 IP: **224.0.0.2 (All Routers)**
- Group Address: 떠나는 그룹 G

기본 동작(IGMPv2):

1. 호스트가 G 그룹 수신을 중단할 때:
   - 해당 인터페이스에서 마지막 Report를 보냈다면 → **Leave Group(G)** 전송
2. Querier 라우터는 Leave를 받으면:
   - 즉시 **Group-Specific Query(G)** 를 Last Member Query Interval 동안 전송
   - 응답(Report)이 들어오지 않으면, 그 인터페이스 뒤에는 더 이상 G 멤버 없다고 판단하고 멀티캐스트 포워딩 중단

Cisco 블로그 등에서는 “Leave Group은 사실 특별한 포맷의 Membership Report로 볼 수 있으며, 라우터가 이후 Group-Specific Query를 통해 마지막 멤버 여부를 확인한다”고 설명한다.

IGMPv3에서는 Leave 개념이 좀 더 복잡해져, “해당 소스를 INCLUDE 리스트에서 제거”하는 Record Type( BLOCK_OLD_SOURCES 등)으로 표현되기도 한다.

---

### 간단한 메시지 교환 시나리오

예제 토폴로지:

```text
Host H1, H2 ---- Switch ---- Router R (IGMP Querier)
                       |
                   PIM Router / Multicast Core
```

- 그룹: G = 239.1.1.1 (IPTV 채널)
- IGMPv2 사용

1. **H1이 G에 Join**
   - H1: Report(G) 전송 (두 번, unsolicited)
   - R: 인터페이스 S에 대해 “G 멤버 존재”로 상태 생성
2. **일정 시간 후 R이 General Query 전송**
   - 모든 호스트(H1, H2)는 자신이 가입한 그룹에 대해 Report 타이머 설정
   - H1: 랜덤값 3초, H2: 7초 가정
   - 3초 뒤 H1이 Report(G) 전송 → H2는 이 Report를 보고 자기 Report 억제
3. **H2가 뒤늦게 Join**
   - H2: 즉시 Report(G) 전송
   - R: 이미 인터페이스 S에 G 멤버 있음 상태를 유지하지만, “멤버가 2명”이 되었음을 내부적으로 기록 (타이머/카운트 등 방식은 구현 의존)
4. **H1이 G에서 Leave**
   - H1: Leave Group(G) 전송
   - R: Group-Specific Query(G) 를 S 인터페이스로 몇 번 전송
   - H2는 여전히 G를 듣고 있으므로 Report(G) 응답
   - R: “아직 G 멤버 있음”으로 상태 유지
5. **H2마저 Leave**
   - H2: Leave Group(G) 전송
   - R: 다시 Group-Specific Query(G) 전송
   - 더 이상 Report 없음 → S 인터페이스의 G 상태 제거

이런 메시지 교환을 Wireshark로 캡처하면, `igmp` 필터로 쉽게 볼 수 있다.

---

## Propagation of Membership Information

이제 IGMP 메시지가 **어떻게 네트워크 속에서 멤버십 정보를 전파하고 유지하는지**를 살펴보자.

핵심 주제:

1. Querier 선출과 Query 전파
2. 호스트의 Report/Leave와 **타이머/억제 메커니즘**
3. L2 스위치에서의 **IGMP Snooping**
4. 멀티캐스트 라우터(PIM 등)로의 정보 전달

---

### Querier 선출과 Query 전파

한 브로드캐스트 도메인(LAN)에는 **기본적으로 IGMP Querier가 하나만 있어야 한다.**
서로 다른 라우터가 동시에 Query를 뿌리면 불필요한 트래픽이 증가하므로, IGMP에는 **Querier Election** 규칙이 있다.

- IGMPv2/v3의 기본 규칙:
  - **가장 낮은 IP 주소를 가진 라우터가 Querier** 가 된다.
- 각 라우터는 자신도 Query를 전송하면서, 다른 라우터가 보내는 Query의 **Source IP**를 확인
  - 자신보다 낮은 IP를 가진 Querier를 발견하면 **Querier 역할을 포기**하고 Non-Querier로 전환
  - 일정 시간 동안 Querier의 Query가 보이지 않으면, 다시 Querier Election 시작

이렇게 해서 하나의 LAN에 딱 한 Querier만 남게 되고,
이를 통해 **General Query가 주기적으로 브로드캐스트**되어 호스트들의 멤버십 상태를 갱신한다.

---

### 타이머와 멤버십 유지

IGMP는 여러 종류의 타이머를 사용해 **멤버십을 “자동으로 만료”** 시킨다.

주요 파라미터(추상):

- **Query Interval**: General Query 주기 (기본 125초 정도)
- **Max Response Time**: Query에 대한 응답 최대 지연 (기본 10초 등)
- **Group Membership Interval**:
  - 대략 $$\text{(Robustness Variable)} \times \text{Query Interval} + \text{Max Resp Time}$$
  - 이 시간 동안 Report가 안 보이면 해당 그룹 멤버십 만료
- **Last Member Query Interval**:
  - Leave 후 그룹이 실제 사라졌는지 확인하기 위한 Group-Specific Query 간격

수식 예시:

Robustness Variable = 2, Query Interval = 125초, MaxRespTime = 10초라면

$$
\text{GroupMembershipInterval} \approx 2 \times 125 + 10 = 260 \text{초}
$$

라우터는 대략 이 정도 시간 동안 해당 인터페이스에서 Report를 보지 못하면,
“이제 이 인터페이스 뒤에는 G 그룹 멤버가 없다”고 판단한다.

#### 랜덤 지연과 억제(suppression)

Query에 대한 Report는 **랜덤 지연** 메커니즘을 사용한다.

- 이유:
  - 호스트가 100개인 서브넷에서, 각각이 동시에 Report를 보내면
    “폭발적 리포트(implosion) 문제” 발생
- 동작:
  1. 호스트는 Query를 받고, 각 그룹에 대해
     $$T \sim U(0, \text{MaxRespTime})$$
     로 랜덤 타이머 선택
  2. 타이머 만료 전, 같은 그룹에 대한 다른 호스트의 Report를 보면,
     “필요한 정보는 이미 라우터에게 전달됐다”고 보고 **자기 Report 취소**

결과:

- 최악의 경우 한 그룹당 Report 1개만 라우터에 전달
- 호스트 수가 많아도 네트워크 트래픽이 크게 증가하지 않음

##### 간단한 파이썬 시뮬레이션 (개념용)

아래 코드는 Query에 대한 Report 억제를 아주 단순하게 시뮬레이션한다.

```python
import random

def simulate_igmp_suppression(num_hosts, max_resp_time=10):
    # 각 호스트가 선택한 랜덤 타이머
    timers = [random.uniform(0, max_resp_time) for _ in range(num_hosts)]
    # 가장 작은 타이머를 가진 호스트 혼자 Report를 보낸다고 가정
    first_report_time = min(timers)
    reports_sent = 1  # 그 호스트 1명만 보냈다고 가정
    return first_report_time, reports_sent

for n in [1, 5, 50, 500]:
    t, reports = simulate_igmp_suppression(n)
    print(f"{n}명의 호스트 → 실제 Report 수 ≈ {reports}, 첫 Report 시각 ≈ {t:.2f}초")
```

실제 IGMP 알고리즘은 좀 더 정교하지만,
**호스트가 많아져도 그룹당 Report 수는 작게 유지**된다는 개념을 이해하기에 충분하다.

---

### IGMP Snooping: 스위치에서의 멤버십 전파

라우터만 IGMP를 이해하면 충분할 것 같지만,
실제 이더넷 스위치 레벨에서도 멀티캐스트 트래픽을 최적화하기 위해 **IGMP Snooping** 이라는 기능이 사용된다.

- 원리:
  - L2 스위치는 **IGMP를 이해하지 않아도 된다**가 원래 원칙이지만,
  - 실제로는 스위치가 **IGMP Query / Report / Leave를 “훔쳐보기(snoop)”** 해서
    “어떤 포트에 어떤 그룹 멤버가 있는지”를 알면,
    **그룹 G에 대한 멀티캐스트 프레임을 해당 포트에만 포워딩**할 수 있다.
- 구현:
  - 스위치는 CPU에서 IGMP 패킷을 분석하고,
    VLAN/포트별 멀티캐스트 포워딩 테이블을 유지
  - Ex) Cisco Catalyst, HPE Aruba, Nokia 등 L2/L3 스위치는 IGMP Snooping을 제공한다.

예제 동작:

```text
Switch 포트:
  1: Router R (Querier)
  2: Host H1 (G1 가입)
  3: Host H2 (G1, G2 가입)
  4: Host H3 (어떤 그룹에도 가입 X)
```

동작 흐름:

1. H1이 G1에 Join → IGMP Report(G1) 이 포트 2에서 전송
   - 스위치: “포트 2에 G1 멤버 있음” 기록
2. H2가 G1, G2에 Join → 포트 3에서 Report(G1), Report(G2)
   - 스위치: “포트 3에 G1, G2 멤버 있음” 기록
3. R 또는 상위 라우터에서 G1 스트림 수신 → 스위치로 유입
   - 스위치: G1 트래픽을 **포트 2와 3에만** 포워딩, 4는 제외

이렇게 해서 브로드캐스트 도메인의 **불필요한 멀티캐스트 flooding**을 줄일 수 있다.
Cisco, Juniper, HPE 문서들도 IGMP Snooping을 “multicast flooding 최소화, 효율적 포워딩” 방식으로 설명한다.

---

### 멀티캐스트 라우팅 프로토콜로의 전달

IGMP는 **호스트 ↔ 첫 번째 홉 라우터** 사이의 프로토콜이고,
이 라우터는 자기 인터페이스별 IGMP 상태를 **멀티캐스트 라우팅 프로토콜(PIM 등)** 에 전달한다.

요약:

- PIM-SM/SSM 등은 각 인터페이스에 대해
  - “어떤 그룹 G, 혹은 어떤 (S,G)에 **다운스트림 리시버**가 있는지” 를 확인해야 한다.
- IGMP 라우터 모듈은:
  - IGMP Report/Leave/Query 응답 상태를 보고
  - “이 인터페이스에 G(또는 S,G) 멤버가 1명 이상 있다”면
    → PIM에 “이 인터페이스를 outgoing 인터페이스로 포함하라”고 통보
  - 멤버가 0이 되면
    → PIM에 “이 인터페이스를 트리에서 제거하라”고 통보

결국 **IGMP는 멀티캐스트 트리의 “잎사귀(leaf)” 정보**를 제공해,
라우팅 프로토콜이 올바른 트리를 구성할 수 있게 한다.

---

## Encapsulation

마지막으로, IGMP가 **IP 패킷과 이더넷 프레임 안에 어떻게 실리는지**를 정리해 보자.

---

### IP 안에서의 IGMP 캡슐화

IGMP 메시지는 항상 **IPv4 헤더 뒤에 바로 붙는** L4 프로토콜이며,
IP 헤더의 Protocol 필드는 **2**로 설정된다.

간단한 구조:

```text
+----------------------------+
| Ethernet Header            |
+----------------------------+
| IPv4 Header                |  Protocol = 2 (IGMP), TTL=1
+----------------------------+
| IGMP Header & Payload      |
+----------------------------+
```

주요 IP 헤더 설정:

- **Source IP**
  - Query: Querier 라우터의 인터페이스 주소
  - Report/Leave:
    - v1/v2: 호스트 인터페이스 주소 사용
    - v3: 0.0.0.0 또는 인터페이스 주소 허용 (RFC 3376 명시)
- **Destination IP**
  - General Query: 224.0.0.1 (All Hosts)
  - v3 Report: 224.0.0.22 (All IGMPv3-capable Routers)
  - v1/v2 Report: 가입하려는 그룹 G 주소 (예: 239.1.1.1)
  - v2 Leave: 224.0.0.2 (All Routers)
- **TTL (Time To Live)**: 항상 **1**
  - 로컬 서브넷에서만 유효, 라우터를 넘지 않도록 보장

이 구조 때문에 IGMP 패킷은 **LAN 범위를 넘어 인터넷 전체로 전파되지 않는다.**

---

### Ethernet에서의 MAC 주소 매핑

IP 멀티캐스트는 이더넷 상에서 **멀티캐스트 MAC 주소**로 매핑된다.

간단한 규칙(IPv4):

- 멀티캐스트 MAC 주소 범위: `01:00:5e:xx:xx:xx`
- 하위 23비트는 IPv4 멀티캐스트 주소의 하위 23비트에서 가져온다.

예:

- 224.0.0.1 (All Hosts) → `01:00:5e:00:00:01`
- 224.0.0.2 (All Routers) → `01:00:5e:00:00:02`
- 224.0.0.22 (All IGMPv3 routers) → `01:00:5e:00:00:16`

IGMP Query/Report/Leave는 해당 목적지 IP에 대응하는 멀티캐스트 MAC으로 전송되며,
스위치는 이를 기반으로 프레임을 브로드캐스트/멀티캐스트 처리한다
(IGMP Snooping이 없으면 관련 VLAN 전체에 flood).

---

### 예: IGMPv2 General Query 패킷 구조

텍스트로 패킷을 쪼개 보면 다음과 같다 (필드 이름 중심의 예):

```text
Ethernet:
  Dest MAC: 01:00:5e:00:00:01 (224.0.0.1)
  Src  MAC: 00:11:22:33:44:55 (Router R 인터페이스 MAC)
  EtherType: 0x0800 (IPv4)

IPv4 Header:
  Version/IHL: 4 / 5
  Total Length: 32 bytes (예시)
  Protocol: 2 (IGMP)
  Src IP: 192.0.2.1 (R 인터페이스)
  Dst IP: 224.0.0.1
  TTL: 1

IGMPv2 Query:
  Type: 0x11 (Membership Query)
  Max Resp Time: 10 (1초 단위, 즉 10초)
  Checksum: 0xXXXX
  Group Address: 0.0.0.0 (General)
```

Wireshark에서는 `Internet Group Management Protocol` 이라는 레이어로 표시되며,
필드별 디코딩을 바로 확인할 수 있다.

---

### IGMP와 터널/가상화 환경

실무에서는 다음과 같은 환경에서 IGMP 캡슐레이션이 한 단계 더 올라가기도 한다.

- **GRE/IPv4 터널 위의 IGMP**
  - 데이터센터/ISP 백본에서 멀티캐스트 도메인을 확장하기 위해
    GRE/IP-in-IP 터널을 사용할 수 있다.
  - 이 경우 IGMP는 “내부 IPv4 헤더”에 실리고,
    그 외곽에 또 다른 IP 헤더(터널용)가 붙는다.
- **MVPN / MPLS 네트워크**
  - Provider는 고객 VRF에서 발생하는 IGMP를
    P-Tunnel(MPLS LSP, P2MP LSP 등)로 캡슐화해 운반한다.
  - 외형상 Provider 라우터는 **VRF 인터페이스에서 IGMP를 받고,
    Provider 내부에서는 BGP/MPLS 기반 MVPN signaling으로 변환**한다.

이 부분은 멀티캐스트 VPN 챕터나 서비스 프로바이더 설계 문서에서 자세히 다루지만,
**IGMP는 언제나 “가장 안쪽 레벨의 IPv4 멀티캐스트 신호”**라는 관점은 변하지 않는다.

---

## 마무리 요약

이 절(21.5 IGMP)의 핵심을 정리하면 다음과 같다.

1. **IGMP Messages**
   - Query (General, Group-Specific, Group-and-Source-Specific)
   - Membership Report (v1/v2 단일 그룹, v3 다중 그룹/소스 + Include/Exclude)
   - Leave Group (v2 명시적 Leave, v3에서는 Record Type으로 표현)
   - Type 필드(0x11, 0x12, 0x16, 0x17, 0x22)와 Group Address, Max Response Time, Checksum 등 공통 구조

2. **Propagation of Membership Information**
   - 한 브로드캐스트 도메인당 하나의 IGMP Querier (가장 낮은 IP)
   - Query Interval / Group Membership Interval / Last Member Query Interval 등
     타이머 기반으로 멤버십 자동 갱신·만료
   - Query에 대한 Report는 랜덤 지연 + 억제(suppression)로 **트래픽을 최소화**
   - L2 스위치의 IGMP Snooping은 Report/Leave를 “훔쳐보며”
     포트별 멀티캐스트 포워딩 테이블을 만들어 flood를 줄인다.

3. **Encapsulation**
   - IGMP는 **IPv4 Protocol 2**, TTL=1, 로컬 서브넷을 벗어나지 않음
   - 목적지 IP:
     - 224.0.0.1(All Hosts), 224.0.0.2(All Routers), 224.0.0.22(All IGMPv3 routers), 혹은 그룹 G 자체
   - 이더넷 멀티캐스트 MAC(`01:00:5e:..`)으로 매핑되어 프레임이 전송
   - 터널/MPLS 환경에서도 **가장 안쪽의 IPv4 멀티캐스트 신호 계층**으로 사용
