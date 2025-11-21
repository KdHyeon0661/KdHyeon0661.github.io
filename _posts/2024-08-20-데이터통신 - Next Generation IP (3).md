---
layout: post
title: 데이터 통신 - Next Generation IP (3)
date: 2024-08-20 22:20:23 +0900
category: DataCommunication
---
# Chapter 22. Next Generation IP — 22.3 The ICMPv6 Protocol (Error / Info / ND / Group Membership)

ICMPv6(Internet Control Message Protocol for IPv6)는 **IPv6의 “관리·제어 채널”** 역할을 한다.
IPv4의 ICMP와 달리, **주소 자동 설정, 이웃 탐색, 멀티캐스트 그룹 관리까지 모두 ICMPv6 위에서 돌아간다.**

이 절에서는 ICMPv6를 네 가지 축으로 정리한다.

1. **Error Reporting Messages** — 오류 보고 메시지(Type 0–127)
2. **Informational Messages** — 정보 메시지(Type 128–255)
3. **Neighbor Discovery Messages** — IPv6의 ARP + Router Discovery + DAD
4. **Group Membership Messages** — IPv6 멀티캐스트 그룹 관리(MLD)

---

## ICMPv6 공통 형식과 큰 그림

### ICMPv6 메시지의 두 클래스

RFC 4443 기준, 모든 ICMPv6 메시지는 **Type 필드(8비트)** 로 구분되며, 가장 상위 비트로 에러/정보 메시지를 나눈다.

- **Type 0–127**: Error Messages
- **Type 128–255**: Informational Messages

따라서 **Destination Unreachable, Packet Too Big, Time Exceeded, Parameter Problem** 등은 에러 메시지이고,
**Echo Request/Reply, Neighbor Solicitation/Advertisement, Router Solicitation/Advertisement, MLD 메시지** 등은 정보 메시지에 속한다.

ICMPv6 메시지는 항상 **IPv6 패킷의 Next Header = 58** 값으로 식별된다.

### ICMPv6 공통 헤더 형식

모든 ICMPv6 메시지는 다음과 같은 공통 헤더를 가진다.

```text
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+---------------------------------------------------------------+
|     Type      |     Code      |          Checksum             |
+---------------------------------------------------------------+
|                      Message Body ...                         |
+---------------------------------------------------------------+
```

- **Type (8비트)**: 메시지 종류 (에러 / 정보 / ND / MLD 등)
- **Code (8비트)**: Type 내 세부 원인 구분
- **Checksum (16비트)**: IPv6 pseudo-header까지 포함한 1’s complement 체크섬

체크섬은 IPv4/ICMP와 비슷하게, **IPv6 pseudo-header + ICMPv6 메시지 전체**에 대해 계산한다.

$$
\text{Checksum} = \operatorname{ones\_complement\_sum}(\text{IPv6 pseudo-header} + \text{ICMPv6 message})
$$

IPv6 pseudo-header는 다음 필드를 포함한다(Upper-Layer Checksum 공통 구조):

- Source Address (128비트)
- Destination Address (128비트)
- Upper-Layer Packet Length (32비트)
- 3바이트 0
- Next Header (8비트, 여기서는 58)

이 구조 덕분에, 수신자는 **ICMPv6 메시지가 어느 Source/Dest 쌍을 대상으로 했는지까지 검증**할 수 있다.

---

## Error Reporting Messages

에러 메시지는 네트워크 상의 **실패·제한을 “바로 원점에 알려주는” 역할**을 한다.
IPv6에서는 다음 네 가지 에러 메시지가 정의되어 있다.

1. Type 1 — Destination Unreachable
2. Type 2 — Packet Too Big
3. Type 3 — Time Exceeded
4. Type 4 — Parameter Problem

각 메시지는 **원본 IPv6 패킷의 앞부분(최소 1280바이트 중 일부)** 을 포함하여, 상위 계층이 어떤 패킷 때문에 에러가 났는지 알 수 있게 한다.

> RFC 4890 등은 ICMPv6 에러 메시지를 **망 차단하면 안 되는 중요 제어 트래픽**으로 분류하고, 방화벽 필터링 가이드라인을 제시한다.

### Destination Unreachable (Type 1)

#### 용도와 Code 값

Type 1은 목적지에 도달할 수 없는 여러 상황을 표현한다. 대표적인 Code 값(요약):

| Code | 의미 (예시)                            |
|------|----------------------------------------|
| 0    | No route to destination                |
| 1    | Communication with dest administratively prohibited |
| 3    | Address unreachable                    |
| 4    | Port unreachable                       |

(정확한 코드 목록은 RFC 4443 참조)

#### 메시지 형식

```text
Type = 1 (Destination Unreachable)
Code = (0,1,3,4 등)
Checksum
Unused (32비트, 0)
Original IPv6 Header + Leading Octets of Original Payload
```

#### 예제 시나리오 1 — 방화벽에 의해 차단된 UDP 포트

- 클라이언트: UDP 포트 50000에서 서버의 포트 9999로 패킷 전송
- 중간 방화벽: 정책상 포트 9999 접근 “금지”

이 경우 방화벽 혹은 최종 노드는 **Type 1, Code 1 (Administratively prohibited)** 를 보낼 수 있다.

Wireshark에서 볼 수 있는 예:

```text
Internet Control Message Protocol v6
    Type: 1 (Destination Unreachable)
    Code: 1 (Communication with destination administratively prohibited)
    Checksum: 0x8e30 [correct]
    Unused: 00000000
    As much of invoking packet as possible:
        Internet Protocol Version 6, Src: 2001:db8:1::10, Dst: 2001:db8:2::20
        User Datagram Protocol, Src Port: 50000, Dst Port: 9999
```

상위 계층(예: UDP 애플리케이션) 입장에서는, 이 메시지를 보고 **포트가 막혀 있다는 사실을 명확히 알 수 있다.**

#### 예제 시나리오 2 — 포트 미열림 (Port Unreachable)

IPv6에서 UDP 소켓이 열려 있지 않은 포트로 패킷이 도착하면,
호스트는 Type 1, Code 4(Port unreachable)를 보내는 것이 일반적이다. (UDP “pseudo-connectionless” 특성 상, 이 에러를 받지 못할 수도 있다.)

---

### Packet Too Big (Type 2)

#### Path MTU Discovery의 핵심

IPv6에서는 **중간 라우터가 패킷을 조각화하지 않는다.**
대신, 너무 큰 패킷을 받은 라우터는 **Packet Too Big** 메시지를 보내 **Path MTU Discovery(PMTUD)** 를 수행하게 한다.

#### 메시지 형식

```text
Type = 2 (Packet Too Big)
Code = 0
Checksum
MTU (32비트)  <-- 이 경로에서 허용되는 MTU
Original IPv6 Header + Leading Octets of Original Payload
```

- `MTU` 필드: 이 패킷이 지나가려다 막힌 링크의 MTU 값.

#### 예제 시나리오 — 1500 → 1280 MTU 다운 링크

1. 송신자 A는 MTU 1500 환경에서, 1400바이트짜리 IPv6 패킷을 B에게 전송.
2. 중간 라우터 R이 MTU 1280 링크를 향해 패킷을 전송하려다가, **링크 MTU(1280)를 초과**하는 것을 발견.
3. R은 패킷을 드롭하고, A에게 다음과 같은 ICMPv6 Packet Too Big 메시지 전송:

```text
Type: 2 (Packet Too Big)
Code: 0
MTU: 1280
Original Packet Header ...
```

4. A는 앞으로 이 목적지에 대해 1280 이하 크기로 패킷을 보내도록 Path MTU 정보를 갱신한다.

---

### Time Exceeded (Type 3)

#### 목적

IPv4의 “TTL Exceeded”에 해당.
**라우팅 루프**나 **지나치게 긴 경로**에서 패킷이 무한 순환하지 않도록 막는다.

#### 메시지 형식과 Code

- Code 0: Hop limit exceeded in transit
- Code 1: Fragment reassembly time exceeded

형식:

```text
Type = 3 (Time Exceeded)
Code = 0 or 1
Checksum
Unused (32비트, 0)
Original IPv6 Header + Leading Octets of Original Payload
```

#### 예제 — traceroute6

Linux/Unix의 `traceroute6` 는 Hop Limit 값을 1부터 증가시키며 패킷을 보내고,
각 홉에서 반환되는 **ICMPv6 Time Exceeded (Code 0)** 을 이용해 경로를 알아낸다.

```bash
# 예: traceroute6 google.com

$ traceroute6 2001:4860:4860::8888

 1  2001:db8:1::1   1.123 ms  (ICMPv6 Time Exceeded from 라우터1)
 2  2001:db8:10::1  5.456 ms  (ICMPv6 Time Exceeded from 라우터2)
 ...
```

각 중간 라우터는 Hop Limit이 0이 된 패킷을 버리고, **Type 3, Code 0**으로 응답하여 자신을 드러낸다.

---

### Parameter Problem (Type 4)

#### 목적

IPv6 헤더나 확장 헤더, 상위 계층 헤더에서
**“이 값은 도저히 해석할 수 없다”** 라는 상황을 보고하는 에러.

#### Code 값

- Code 0: Erroneous header field encountered
- Code 1: Unrecognized Next Header type encountered
- Code 2: Unrecognized IPv6 option encountered

메시지 형식:

```text
Type = 4 (Parameter Problem)
Code = 0/1/2
Checksum
Pointer (32비트)  <-- 문제가 된 바이트의 오프셋
Original IPv6 Header + Leading Octets of Original Payload
```

`Pointer` 필드는 **IPv6 패킷 전체(고정 헤더 + 확장 헤더 + 상위 계층 헤더)** 에서
문제가 발생한 바이트의 offset을 나타낸다.

#### 예제 — 잘못된 Next Header 값

어떤 노드가 고의적이든 버그든, `Next Header = 255` 같은 정의되지 않은 값을 넣었다고 하자.
경로 상 라우터 혹은 목적지 노드는 이 값을 이해할 수 없으므로 다음과 같이 응답할 수 있다.

```text
Type = 4 (Parameter Problem)
Code = 1 (Unrecognized Next Header type)
Pointer = 6 (IPv6 헤더의 Next Header 필드 위치)
Original packet header ...
```

수신자(즉, 원 발신자)는 이 메시지를 보고 **자신이 만든 헤더 값이 잘못되었음을 알게 된다.**

---

## Informational Messages

Informationals는 “에러는 아니지만, **상태 조회 / 제어 / 관리** 를 위한 메시지들”이다.
여기에는 **Echo Request/Reply**, 그리고 **Neighbor Discovery, MLD 메시지** 등도 포함된다.

이 절에서는 먼저 **순수 정보 메시지(Echo 등)** 를 보고,
이후 절에서 ND와 MLD를 별도로 자세히 다룬다.

### / Echo Reply (Type 129)

#### 용도

- IPv6에서 **ping** 을 구현하는 기본 도구.
- 노드의 reachability, 왕복 시간(RTT)을 측정하기 위해 사용.

#### 형식

```text
Type = 128 (Echo Request) / 129 (Echo Reply)
Code = 0
Checksum
Identifier (16비트)
Sequence Number (16비트)
Data ...
```

- Identifier, Sequence Number: 요청/응답 매칭, RTT 측정에 사용.

#### 예제 — ping6

```bash
$ ping6 2001:db8:100::1
PING 2001:db8:100::1(2001:db8:100::1) 56 data bytes
64 bytes from 2001:db8:100::1: icmp_seq=1 ttl=64 time=0.654 ms
64 bytes from 2001:db8:100::1: icmp_seq=2 ttl=64 time=0.612 ms
...
```

Wireshark로 보면:

- 클라이언트 → 서버: Type 128(Echo Request)
- 서버 → 클라이언트: Type 129(Echo Reply), 같은 Identifier/Sequence 값

---

## Neighbor Discovery Messages

Neighbor Discovery(ND)는 **IPv4 ARP + ICMP Router Discovery + 일부 DHCP 기능**을 합쳐 놓은 프로토콜이다.
하지만 **프로토콜 메시지 자체는 모두 ICMPv6 Type으로 정의**된다.

ND 관련 메시지 Type:

- 133 — Router Solicitation (RS)
- 134 — Router Advertisement (RA)
- 135 — Neighbor Solicitation (NS)
- 136 — Neighbor Advertisement (NA)
- 137 — Redirect

ND는 RFC 4861에서 정의되며, IPv6 Node Requirements(RFC 8504)에 따르면 **노드는 ND를 MUST 지원**해야 한다(일부 예외 링크 타입 제외).

### Router Solicitation (Type 133)

#### 목적

호스트가 링크에 붙은 직후,

- “나에게 라우터 정보 좀 주세요” 라고 **라우터에게 먼저 물어보는** 메시지.

RA를 기다리기만 하면 시간이 오래 걸릴 수 있으므로,
호스트는 RS를 보내서 **RA를 즉시 받도록** 유도한다.

#### 형식(요약)

```text
Type = 133
Code = 0
Checksum
Reserved (32비트)
Options...
```

옵션 예:

- Source Link-Layer Address Option (호스트의 MAC 주소)

#### 예제 시나리오 — 부팅 직후 SLAAC

1. 노트북이 Wi-Fi에 처음 연결되고, 링크-로컬 주소를 먼저 설정한다.
2. 곧바로 RS를 `ff02::2` (all-routers multicast)로 전송.
3. 링크 상 라우터들이 이에 대해 RA를 보낸다.

`tcpdump` 예시:

```bash
$ sudo tcpdump -n -i eth0 icmp6 and 'ip6[40] == 133'
12:00:01.123456 fe80::1 > ff02::2: ICMP6, router solicitation, length 16
```

---

### Router Advertisement (Type 134)

#### 목적

라우터가 주기적으로 또는 RS에 응답하여 다음 정보를 제공:

- Prefix 정보 (전역 IPv6 주소 생성용, SLAAC)
- 기본 라우터(Default Router)로서의 자신 정보
- MTU, DNS 서버(RFC 8106), 기타 옵션

이 메시지 덕분에, 호스트는 **DHCP 서버 없이도 IPv6 주소와 기본 경로를 자동 설정**할 수 있다.

#### 형식(중요 필드만)

```text
Type = 134
Code = 0
Checksum
Cur Hop Limit (8)
M/O 플래그 + 기타 플래그 (8)
Router Lifetime (16)
Reachable Time (32)
Retrans Timer (32)
Options...
```

대표 옵션:

- Prefix Information Option
- MTU Option
- Source Link-Layer Address Option
- Recursive DNS Server Option(RDNSS, RFC 8106)

#### 예제 — SLAAC로 글로벌 주소 만들기

상황:

- 라우터가 `/64` prefix `2001:db8:100:1::/64` 를 광고.
- RA 메시지 안에 Prefix Information Option 포함.

호스트는 다음 절차를 따른다.

1. 링크-로컬 주소(예: `fe80::a00:27ff:fe12:3456`) 먼저 생성.
2. RA 분석:
   - Prefix: `2001:db8:100:1::/64`
   - L flag(주소로 사용 가능) = 1
   - A flag(자동 설정 가능) = 1
3. 자신의 인터페이스 ID(예: 64비트)를 Prefix 뒤에 붙여 **글로벌 주소 생성**:
   - `2001:db8:100:1:xxxx:xxxx:xxxx:xxxx`
4. NS/NA로 DAD(중복 주소 탐지) 수행 후 사용.

---

### / Neighbor Advertisement (Type 136)

이 둘은 **ARP의 IPv6 버전**이라 볼 수 있다.

#### Neighbor Solicitation(NS)

목적:

- **로컬 링크에서 “이 IPv6 주소를 가진 노드가 누구냐”** 를 알아내는 메시지.
- DAD(중복 주소 탐지)에도 사용.

형식(요약):

```text
Type = 135
Code = 0
Checksum
Reserved (32비트)
Target Address (128비트)
Options...
```

옵션:

- Source Link-Layer Address Option (요청자 MAC 주소) – 일반 ARP 요청의 sender MAC 역할

#### Neighbor Advertisement(NA)

NS에 대한 응답 혹은 특정 이벤트(주소 변경 등)를 알리는 메시지.

```text
Type = 136
Code = 0
Checksum
Router / Solicited / Override 플래그 + Reserved
Target Address (128비트)
Options...
```

옵션:

- Target Link-Layer Address Option (응답자 MAC 주소)

#### 예제 1 — ARP에 해당하는 주소 해석

- 호스트 A: `2001:db8:1::10`
- 호스트 B: `2001:db8:1::20` (같은 링크, 이더넷)

A가 B에게 TCP SYN을 보내려면, B의 MAC 주소가 필요하다.

1. A: `ff02::1:ff00:20` (Solicited-Node multicast) 로 NS 전송
   - Target Address = `2001:db8:1::20`
2. B: NA로 응답
   - Target Address = `2001:db8:1::20`
   - Target Link-Layer Address Option = B의 MAC

`tcpdump` 예시:

```bash
# Neighbor Solicitation

fe80::a -> ff02::1:ff00:20: ICMP6, neighbor solicitation, who has 2001:db8:1::20

# Neighbor Advertisement

fe80::b -> fe80::a: ICMP6, neighbor advertisement, tgt is 2001:db8:1::20, mac 00:11:22:33:44:55
```

#### 예제 2 — Duplicate Address Detection(DAD)

호스트가 새 주소를 만들었을 때,
**이미 다른 노드가 쓰고 있지 않은지 확인**해야 한다.

1. 호스트 X가 주소 `2001:db8:1::30`을 만들려 함.
2. Source Address를 **Unspecified (::)** 으로 두고, Target Address에 그 주소를 넣어 NS 전송.
3. 만약 다른 노드가 이 주소를 이미 쓰고 있다면, NA를 보냄.
4. NA가 오면 X는 주소를 버리고 다른 주소를 생성.

---

### Redirect (Type 137)

#### 목적

라우터가 호스트에게 다음과 같이 알려줄 수 있다.

- “나를 거쳐 가는 대신, **다른 라우터/노드를 직접 향해 보내는 게 더 최적이야.**”

IPv4의 ICMP Redirect와 유사하지만, ND와 결합되어 훨씬 구조적이다.

#### 형식(요약)

```text
Type = 137
Code = 0
Checksum
Reserved
Target Address
Destination Address
Options...
```

- Target Address: 더 나은 next-hop 주소
- Destination Address: 이 Redirect가 적용되는 원래 목적지

실무에서는, 보안 위협(스푸핑, MITM) 때문에 많은 환경에서 Redirect를 **차단**하거나,
라우터가 기본적으로 Redirect를 **비활성화**하는 경우가 많다.

---

## Group Membership Messages (MLD — Multicast Listener Discovery)

IPv6에서는 **멀티캐스트 그룹 가입/탈퇴** 기능을
ICMPv6 기반의 **MLD(Multicast Listener Discovery)** 가 담당한다.
MLD는 유선/데이터센터/ISP 네트워크에서 멀티캐스트 트래픽을 **필요한 링크에만 전달하도록** 하는 핵심 프로토콜이다.

### MLD와 ICMPv6의 관계

- MLD는 **ICMPv6의 몇 가지 Type 값으로 정의된 “서브 프로토콜”** 이라고 볼 수 있다.

대표 타입:

| 프로토콜 | Type | 의미                           |
|---------|------|--------------------------------|
| MLDv1   | 130  | Multicast Listener Query       |
| MLDv1   | 131  | Multicast Listener Report      |
| MLDv1   | 132  | Multicast Listener Done        |
| MLDv2   | 143  | Multicast Listener Report v2   |

(MLDv2 Query는 여전히 Type 130인데, 포맷이 확장됨)

### MLDv1 메시지 (RFC 2710)

#### Multicast Listener Query (Type 130)

- 라우터가 링크 상의 호스트들에게
  “어떤 멀티캐스트 그룹에 아직 관심이 있느냐” 를 묻는 메시지.

형식(요약):

```text
Type = 130
Code = 0
Checksum
Maximum Response Delay
Reserved
Multicast Address
```

- Multicast Address가 0이면 **General Query** (링크 전체 그룹들 대상)
- 특정 멀티캐스트 주소이면 **Group-Specific Query**

#### Multicast Listener Report (Type 131)

- 호스트가 “나는 이 그룹에 가입하고 싶다/유지하고 있다” 를 라우터에게 알려주는 메시지.
- IGMPv2의 Report와 동일한 역할.

#### Multicast Listener Done (Type 132)

- 호스트가 멀티캐스트 그룹에서 **탈퇴**할 때, 라우터에게
  “나는 더 이상 이 그룹 트래픽이 필요 없다”를 알려주는 메시지.

라우터는 Done을 받으면, 같은 그룹에 남은 가입자가 있는지
Group-Specific Query를 보내 확인할 수 있다.

### MLDv2 (RFC 3810)

MLDv2는 IGMPv3의 기능을 IPv6로 옮겨온 것으로, 다음을 지원한다.

- 소스 필터링(source filtering):
  - “이 그룹 중에서 특정 소스들만 받고 싶다 (INCLUDE)”
  - “특정 소스들만 빼고 받고 싶다 (EXCLUDE)”
- Type 143 — Multicast Listener Report v2에 **여러 그룹, 여러 소스** 정보를 담을 수 있음.

MLDv2 Query(Type 130)는 v1보다 헤더 필드가 늘어나
소스 필터링 관련 정보(쿼리 타입, 소스 리스트 등)를 추가로 포함한다.

### 예제 — IPv6 IPTV/스트리밍 멀티캐스트

상황:

- ISP가 IPv6 멀티캐스트로 IPTV 채널을 송출.
- 채널 A: 그룹 주소 `ff3e::1234`
- 라우터 R이 가입자들이 있는 접속 네트워크에 위치.

#### 가입 과정

1. 셋톱박스(호스트 H)가 채널 A를 보고 싶어함.
2. H는 인터페이스에서 `ff3e::1234` 에 대해 **MLD Report(Type 131/143)** 전송.
3. R은 “이 링크에 ff3e::1234 그룹 가입자가 있다” 라는 상태를 기록.
4. R은 상위 멀티캐스트 라우터/코어로 **PIM/MLD snooping/YANG 모델(RFC 9166 등)** 을 통해 트리 정보를 전달할 수 있다.

#### 유지/쿼리

- R은 주기적으로 MLD Query(Type 130)를 보내
  “아직도 이 그룹 필요해?” 라고 묻는다.
- H는 계속 Report를 보내거나,
  일정 시간 내 응답이 없으면 R은 “더 이상 이 링크에는 가입자가 없다”고 간주하고
  트래픽 전달을 중단.

#### 탈퇴

- H가 채널 A 시청을 종료하면 MLD Done(Type 132) 혹은 v2 Report를 통해
  해당 그룹에서 빠진다는 정보를 전송.
- R은 그 그룹에 남은 가입자가 없으면,
  해당 링크에 더 이상 멀티캐스트 패킷을 포워딩하지 않는다.

---

## 종합 예제 — 하나의 링크에서 ICMPv6가 하는 일들

상황: 회사 사무실의 하나의 유선 LAN(IPv6-only)에서, PC와 서버, 멀티캐스트 스트리밍 장비가 동작한다.

1. **PC가 부팅**
   - 링크-로컬 주소 생성
   - RS(Type 133) 전송 → RA(Type 134) 수신 → SLAAC로 글로벌 주소 생성
   - NS/NA(Type 135/136)로 DAD 및 이웃 MAC 해석

2. **PC가 서버에 ping**
   - Echo Request(Type 128), Echo Reply(Type 129) 왕복

3. **라우팅 문제로 인한 PMTU 이슈 발생**
   - 일부 큰 패킷이 중간 라우터에서 MTU 초과
   → Packet Too Big(Type 2) 수신 후, PC는 전송 크기 줄임

4. **멀티캐스트 세션 참여**
   - PC가 멀티캐스트 그룹에 들어가고 싶어 MLD Report(Type 131/143) 전송
   - 라우터는 링크에 누가 어느 그룹에 가입했는지 테이블 유지
   - 주기적으로 MLD Query(Type 130)로 membership 확인

이 모든 흐름이 **ICMPv6 Type/Code 조합** 위에서 돌아간다.
ICMPv6를 단순히 “ping용 프로토콜”로만 이해하면,
IPv6의 실제 동작(주소 자동 설정, 이웃 탐색, 멀티캐스트)이 거의 보이지 않는다.

---

## 구현·운영 시 주의점 (요약)

1. **ICMPv6 필터링은 매우 신중하게**
   - RFC 4890은 에러/정보 메시지별로 “허용 권장/차단 가능” 기준을 제시한다.
   - 특히 Packet Too Big, Neighbor Solicitation/Advertisement, MLD Query/Report 등은
     차단하면 네트워크가 “비정상”이 된다.

2. **Neighbor Discovery와 MLD는 IPv6 노드 필수 기능**
   - Node Requirements(RFC 8504)에 따르면 일반 호스트는 ND와 ICMPv6 기본 기능을 MUST 지원한다.
   - MLD 역시 멀티캐스트를 사용한다면 필수적인 서브프로토콜이다.

3. **보안**
   - 악의적인 NS/NA, RA, Redirect, MLD 메시지로 인한 공격(ND spoofing, Router spoofing, bogus RA 등)에 대비해
     RA Guard, ND Inspection, MLD Snooping 등 L2/L3 장비 기능을 적절히 사용해야 한다.

---

## 간단한 코드 예시 — Scapy로 ICMPv6 Echo + NS 만들기

마지막으로, Python Scapy를 이용해 간단한 ICMPv6 메시지를 만드는 예를 들어보자.

```python
from scapy.all import IPv6, ICMPv6EchoRequest, ICMPv6ND_NS, send

# 단순 Echo Request (ping)

echo_pkt = IPv6(src="2001:db8:1::10", dst="2001:db8:1::20") / ICMPv6EchoRequest()
send(echo_pkt)

# Neighbor Solicitation (이웃 MAC 주소 알아내기)

ns_pkt = IPv6(src="fe80::a", dst="ff02::1:ff00:20") / ICMPv6ND_NS(
    tgt="2001:db8:1::20"
)
send(ns_pkt)
```

위 예제를 통해 실제 패킷을 캡처(tcpdump/wireshark)하면,
**Type 128/135** ICMPv6 메시지가 실제로 어떻게 헤더에 채워지는지 직접 확인할 수 있다.

---

이로써 22.3절에서 요구한 네 가지 범주:

- **Error Reporting Messages**
- **Informational Messages**
- **Neighbor Discovery Messages**
- **Group Membership Messages(MLD)**

를 모두 실제 메시지 형식, 필드, 예제 시나리오를 중심으로 정리했다.
앞 절(22.1, 22.2)의 IPv6 헤더/확장 헤더 내용과 연결해서 보면,
**IPv6가 “어떻게 스스로 구성되고, 이웃을 찾고, 멀티캐스트 그룹을 유지하며, 실패를 보고하는지”** 전체 그림이 자연스럽게 이어질 것이다.
