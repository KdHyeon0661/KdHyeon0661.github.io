---
layout: post
title: 데이터 통신 - Next Generation IP (2)
date: 2024-08-20 21:20:23 +0900
category: DataCommunication
---
# Chapter 22. Next Generation IP — 22.2 The IPv6 Protocol (Packet Format & Extension Headers)

이 절에서는 **IPv6 패킷 형식**과 **확장 헤더(Extension Header)** 를 집중적으로 파고든다.  

구성:

- **Packet Format**: 40바이트 고정 IPv6 헤더의 필드 구조, IPv4와의 차이, 예제 패킷 해부
- **Extension Header**: 확장 헤더 개념, 종류, 체인 구조, 각 확장 헤더의 역할과 실제 네트워크 이슈

RFC 8200에서 정의한 최신 IPv6 스펙과, 최근의 확장 헤더 처리/보안 관련 문서를 기반으로 설명한다.  

---

## 1. IPv6 패킷 전체 구조

IPv6 패킷은 크게 두 부분으로 나눌 수 있다.

```text
+------------------------+
|  IPv6 Fixed Header     |  40 bytes
+------------------------+
|  Extension Headers     |  0 or more, variable length
+------------------------+
|  Upper-Layer Payload   |  (TCP, UDP, ICMPv6, etc.)
+------------------------+
```

- **IPv6 고정 헤더**: 항상 40바이트, 모든 IPv6 패킷의 첫 부분
- **확장 헤더**: 필요할 때만, 체인 형태로 이어 붙임
- **상위 계층 페이로드**: TCP/UDP/ICMPv6 등 L4 헤더 + 데이터

RFC 8200에서 강조하는 점은:

> “헤더는 공통 부분(40 바이트)을 최대한 단순하게 하고, 나머지 기능은 **확장 헤더**로 분리하여 필요할 때만 사용한다.”  

---

## 2. Packet Format — IPv6 고정 헤더 구조

### 2.1 비트 단위 레이아웃

IPv6 고정 헤더는 항상 40바이트(320비트)이며, 필드는 다음과 같이 배치된다.  

```text
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+---------------------------------------------------------------+
|Version| Traffic Class |           Flow Label                  |
+---------------------------------------------------------------+
|         Payload Length        |  Next Header  |   Hop Limit   |
+---------------------------------------------------------------+
|                                                               |
+                                                               +
|                      Source Address (128 bits)                |
+                                                               +
|                                                               |
+---------------------------------------------------------------+
|                                                               |
+                                                               +
|                   Destination Address (128 bits)              |
+                                                               +
|                                                               |
+---------------------------------------------------------------+
```

각 필드 크기:

| 필드            | 크기   |
|-----------------|--------|
| Version         | 4 bit  |
| Traffic Class   | 8 bit  |
| Flow Label      | 20 bit |
| Payload Length  | 16 bit |
| Next Header     | 8 bit  |
| Hop Limit       | 8 bit  |
| Source Address  | 128 bit|
| Dest Address    | 128 bit|

총합: $$4 + 8 + 20 + 16 + 8 + 8 + 128 + 128 = 320 \text{ bits} = 40 \text{ bytes}$$

### 2.2 각 필드 상세 설명

#### (1) Version (4비트)

- 항상 값 **6** (IPv6).
- IPv4(버전 4)와 구분하기 위한 필드.

#### (2) Traffic Class (8비트)

- IPv4의 **TOS/DSCP** 역할과 유사. QoS, 우선순위 정책을 적용할 때 사용.
- 상위 6비트: DSCP (Differentiated Services Code Point)  
- 하위 2비트: ECN (Explicit Congestion Notification) 용도로 사용 가능.
- 예: VoIP 같은 실시간 트래픽에 높은 우선순위 DSCP 부여.

#### (3) Flow Label (20비트)

- 같은 송신자에서 나오는 **하나의 흐름(flow)** 에 속한 패킷을 식별하기 위해 사용.  
- 라우터는 Flow Label + Source Address 등을 이용해,  
  특정 플로우를 식별하고 **일관된 처리(QoS, ECMP, 로드밸런싱 등)** 를 할 수 있다.  
- RFC 6437은 Flow Label을 pseudo-random 값으로 설정해 해시 기반 로드밸런싱에 활용하도록 권장한다.  

간단한 예:

- 클라이언트가 웹 서버로 HTTP/2 연결을 맺을 때:
  - 동일한 TCP 연결에 속한 패킷에는 모두 같은 Flow Label 값 사용
  - 라우터는 Flow Label을 이용해 같은 경로로 패킷을 라우팅 → 패킷 순서 보존에 도움

#### (4) Payload Length (16비트)

- **IPv6 고정 헤더를 제외한 나머지 전체 길이** (바이트).
  - 확장 헤더 + 상위 계층 헤더 + 데이터 = Payload.
- 범위: $$0 \le \text{Payload Length} \le 65535$$
- 값이 **0**인 경우:
  - “Jumbogram”을 의미하며, 실제 길이는 Hop-by-Hop 옵션 header 안의 `Jumbo Payload` 옵션(32비트)에 들어 있다. 대용량 데이터(예: 초고속 네트워크에서 64KB 초과 MTU)를 위한 기능.  

#### (5) Next Header (8비트)

- **이 헤더 다음에 오는 헤더의 프로토콜 번호**.
- 다음 중 하나를 가리킬 수 있다.  

  - 상위 계층:
    - TCP (6)
    - UDP (17)
    - ICMPv6 (58)
  - 확장 헤더:
    - Hop-by-Hop Options (0)
    - Routing (43)
    - Fragment (44)
    - Encapsulating Security Payload (ESP, 50)
    - Authentication Header (AH, 51)
    - Destination Options (60)
    - Mobility (135), 기타 등

- 이 필드는 IPv4의 “Protocol” 필드 & 옵션과 확장 헤더 체인을 모두 통합하는 역할을 한다.

#### (6) Hop Limit (8비트)

- IPv4의 **TTL(Time To Live)** 에 해당.  
- 패킷이 라우터를 하나 지날 때마다 1씩 감소, 0이 되면 폐기.
- 루프 방지 및 너무 오래된 패킷 제거.

#### (7) Source Address / Destination Address (각 128비트)

- 출발지/목적지 IPv6 주소.
- 앞 절(22.1)에서 설명한 글로벌 유니캐스트/링크-로컬/유니크 로컬 등의 규칙을 모두 따름.

---

### 2.3 IPv4 헤더와의 비교 — 왜 단순화했는가?

IPv4 헤더는 기본 20바이트에 **옵션(optional options)** 이 추가될 수 있어, 길이가 가변이고 필드도 많다(헤더 체크섬, 프래그먼트 관련 필드 등). IPv6는 이를 과감히 정리했다.  

IPv4 → IPv6 변화 요약:

| IPv4 필드            | IPv6에서의 처리                                            |
|----------------------|-----------------------------------------------------------|
| Header Checksum      | **삭제** (L2, L4 계층에 체크섬이 이미 존재)              |
| IHL(헤더 길이)       | **고정 40바이트** 이므로 불필요 → 삭제                    |
| Identification/Flags/Fragment Offset | Fragment Extension Header로 이동           |
| Options              | 확장 헤더(Extension Header) 체인으로 이동                |
| Protocol             | Next Header로 명칭 변경, 확장 헤더 체인과 통합           |

결과:

- 라우터, 호스트에서 **공통 헤더 처리 비용 감소**
- 옵션/추가 기능은 확장 헤더로 분리 → 필요할 때만 처리

---

### 2.4 예제: HTTP over IPv6 패킷 헤더 해부

#### 시나리오

- 클라이언트: `2001:db8:1000:1010::10`
- 서버: `2001:db8:2000:abcd::80`
- 클라이언트가 서버의 TCP 443(HTTPS) 포트로 GET 요청을 보낸다.
- 확장 헤더 없이: IPv6 헤더 + TCP 헤더 + TLS/HTTP 페이로드.

가상의 IPv6 헤더 필드:

| 필드            | 값                          | 설명 |
|-----------------|-----------------------------|------|
| Version         | 6                           | IPv6 |
| Traffic Class   | 0x28                        | DSCP = AF11 (예시), ECN = 0 |
| Flow Label      | 0x12345                     | 특정 TLS 세션을 나타내는 라벨 |
| Payload Length  | 1200                        | TCP 헤더 + TLS/HTTP 페이로드 길이 |
| Next Header     | 6                           | TCP |
| Hop Limit       | 64                          | 초기 값 (운영 정책에 따라 다름) |
| Source Address  | 2001:db8:1000:1010::10      | 클라이언트 주소 |
| Dest Address    | 2001:db8:2000:abcd::80      | 서버 주소 |

이 패킷은 라우터를 통과하면서 **Hop Limit이 63,62,…** 로 줄어든다.  
Flow Label은 전 경로에서 동일하게 유지되어, ECMP/로드밸런싱 장비가 플로우를 식별하는 데 사용될 수 있다.

#### C 구조체 예제

C 코드로 IPv6 헤더를 표현하면 대략 다음과 같다 (플랫폼/패딩 고려는 단순화).  

```c
#include <stdint.h>

struct ipv6_header {
    uint32_t v_tc_fl;      // Version(4) + Traffic Class(8) + Flow Label(20)
    uint16_t payload_len;  // Payload Length
    uint8_t  next_header;  // Next Header
    uint8_t  hop_limit;    // Hop Limit
    uint8_t  src_addr[16]; // Source Address
    uint8_t  dst_addr[16]; // Destination Address
};
```

`v_tc_fl`은 **비트 연산**으로 나누어 쓴다:

- Version: `v_tc_fl >> 28`
- Traffic Class: `(v_tc_fl >> 20) & 0xff`
- Flow Label: `v_tc_fl & 0xfffff`

실제 OS 커널이나 패킷 처리 코드에서도 이와 비슷한 방식으로 필드를 다룬다.

---

### 2.5 헤더 오버헤드 수식

예를 들어, MTU가 1500바이트인 링크에서,

- IPv6 헤더: 40바이트
- TCP 헤더: 20바이트 (옵션 없음 가정)
- 애플리케이션 데이터: $$L_{\text{app}}$$ 바이트

IP 패킷 페이로드 길이(Payload Length)는:

$$
L_{\text{payload}} = L_{\text{TCP\_header}} + L_{\text{app}} = 20 + L_{\text{app}}
$$

전체 L2 프레임(예: 이더넷)에서 IP 헤더까지 포함한 전송 효율은:

$$
\text{효율} = \frac{L_{\text{app}}}{\text{MTU}} \approx \frac{L_{\text{app}}}{1500}
$$

단, 실제로는 L2 헤더(14 바이트 이상), VLAN 태그, PPPoE, 확장 헤더 등이 추가되므로  
응용 데이터 비율은 더 줄어든다. 이때 확장 헤더가 많을수록 **L3/L4 헤더 부분이 길어져 효율이 떨어짐**을 직관적으로 이해할 수 있다.

---

## 3. Extension Header — 개념과 구조

### 3.1 확장 헤더란 무엇인가?

RFC 8200 정의: **IPv6 고정 헤더 뒤에 오는 모든 추가 헤더**로, 상위 계층 헤더(TCP, UDP, ICMPv6 등) 앞에 위치한다.  

특징:

- **체인 구조**:
  - 각 헤더에는 `Next Header` 필드가 있어 “다음에 오는 헤더의 타입”을 가리킴.
- 옵션/특수 기능을 “상자”처럼 필요할 때만 추가:
  - Hop-by-Hop 옵션, 경로 지정(Routing), 조각(Fragmentation), 보안(IPsec), 목적지 옵션 등.

### 3.2 확장 헤더 종류와 번호

IANA IPv6 parameters 레지스트리 기준(2025년 6월 갱신) 대표적인 확장 헤더 번호는 다음과 같다.  

| 값  | 이름                       | 설명                           |
|-----|----------------------------|--------------------------------|
| 0   | Hop-by-Hop Options         | 각 라우터가 처리해야 하는 옵션 |
| 43  | Routing Header             | 특수 경로 지정 (현재 대부분 사용 제한) |
| 44  | Fragment Header            | IPv6 조각화 정보                |
| 50  | Encapsulating Security Payload (ESP) | IPsec 데이터 암호화 |
| 51  | Authentication Header (AH) | IPsec 인증                     |
| 60  | Destination Options        | 목적지 노드(혹은 next node)를 위한 옵션 |
| 135 | Mobility Header            | Mobile IPv6 등에서 사용        |

이 외에도 Host Identity Protocol(139), Shim6(140) 등 특정 기술용 확장 헤더가 존재한다.  

### 3.3 확장 헤더 체인 — Next Header 필드

확장 헤더는 다음과 같이 **체인 형태**로 이어진다:

```text
IPv6 Header
  Next Header = 0 (Hop-by-Hop)
    ↓
Hop-by-Hop Options Header
  Next Header = 60 (Destination Options)
    ↓
Destination Options Header
  Next Header = 6 (TCP)
    ↓
TCP Header
  ...
```

일반적인 예(보안 + 조각화 포함):

```text
IPv6
  Next Header = 0 (Hop-by-Hop)
Hop-by-Hop
  Next Header = 60 (Destination Options)
Destination Options
  Next Header = 44 (Fragment)
Fragment
  Next Header = 50 (ESP)
ESP
  Next Header = 6 (TCP)
TCP
  Payload (HTTP, etc.)
```

RFC 8200은 **노드는 어떤 순서의 확장 헤더도 수용해야 하지만**,  
송신자는 **권고 순서**를 최대한 따르도록 권장한다.  

### 3.4 권장 순서와 실제 운영 이슈

***권장 순서(요약)*** (RFC 8200 및 주요 벤더 문서 기준):  

1. IPv6 기본 헤더
2. Hop-by-Hop Options
3. Destination Options (for next node)
4. Routing Header
5. Fragment Header
6. AH
7. ESP
8. Destination Options (for final destination)
9. Upper-layer header (TCP/UDP/ICMPv6…)

***운영상 이슈***:

- RFC 9098, RFC 9099 등은 실제 인터넷에서 **확장 헤더를 포함한 패킷이 종종 드롭**된다는 사실을 분석한다. 많은 라우터/방화벽이  
  - 처리 비용 및
  - 보안상의 이유  
  때문에 복잡한 확장 헤더 체인을 싫어하기 때문이다.  
- 최근 문서들은 “허용할 확장 헤더를 화이트리스트로 제한”하는 필터링 전략을 권장하기도 한다. (예: ESP, AH 등 필요한 것만 허용)  

---

## 4. 주요 확장 헤더 상세

### 4.1 Hop-by-Hop Options Header (값 0)

#### 4.1.1 목적과 구조

- 라우터를 포함한 **경로 상 모든 노드가 처리해야 하는 옵션**을 담는다.
- 구조(요약):

```text
+-------------------------------+
| Next Header (8)              |
+-------------------------------+
| Hdr Ext Len (8)              |
+-------------------------------+
|   Options ... (variable)     |
+-------------------------------+
```

- `Hdr Ext Len`: 이 헤더의 전체 길이(8바이트 단위, 첫 8바이트 제외).
- Options는 **TLV 형식(Type, Length, Value)** 으로 이어진다.  

예: `Router Alert` 옵션, `Jumbo Payload` 옵션 등이 여기에 들어간다.

#### 4.1.2 예: Jumbo Payload

- IPv4/IPv6의 기본 MTU 한계를 넘는 **“Jumbogram”** 지원을 위해 사용.
- IPv6 헤더의 Payload Length를 0으로 두고,  
  Hop-by-Hop 옵션 안에 실제 길이(32비트)를 적는다.  

#### 4.1.3 실제 운영 현실과 RFC 9673

실제 인터넷에서는 Hop-by-Hop 옵션이 있는 패킷을 라우터가 처리하기 어렵다는 문제가 반복적으로 보고되었다. 그래서 **RFC 9673 및 관련 드래프트**는 Hop-by-Hop 처리 절차를 현실화하고,  
“특정 Hop-by-Hop 옵션만 제한적으로 처리하자”는 방향으로 규정을 조정하고 있다.  

핵심 포인트:

- 라우터는 모든 Hop-by-Hop 옵션을 항상 깊게 처리하지 않아도 된다.
- 필터링/드롭 정책, 성능 한계를 고려해 **선별적인 처리**를 허용.

### 4.2 Destination Options Header (값 60)

#### 4.2.1 목적

- **목적지 노드 또는 특정 “다음 노드”** 에서만 해석해야 하는 옵션을 담는다.  

두 가지 상황에서 사용:

1. 최종 목적지에 전달되기 전에 일시적으로 패킷을 처리해야 하는 노드(예: 특수 라우터)에 대한 옵션
2. 실제 최종 목적지에서 처리해야 하는 옵션

#### 4.2.2 구조

Hop-by-Hop과 동일한 형식:

```text
+-------------------------------+
| Next Header (8)              |
+-------------------------------+
| Hdr Ext Len (8)              |
+-------------------------------+
|   Options ... (TLV)          |
+-------------------------------+
```

### 4.3 Routing Header (값 43)

#### 4.3.1 개념

- 송신자가 “이 패킷이 거쳐야 할 중간 노드”를 지정하는 기능 (source routing).
- Type에 따라 의미가 다르며, 과거에 정의된 **Routing Header Type 0** 은 보안 문제(RFC 5095)로 폐기되었다.  

#### 4.3.2 구조 개요

```text
+-------------------------------+
| Next Header (8)              |
+-------------------------------+
| Hdr Ext Len (8)              |
+-------------------------------+
| Routing Type (8)             |
+-------------------------------+
| Segments Left (8)            |
+-------------------------------+
|   Type-specific data ...     |
+-------------------------------+
```

- `Segments Left`: 남은 경유 노드 수.
- 경유 노드 리스트를 순서대로 담고, 각 라우터가 자신을 지나갈 때 이 필드와 목적지 주소를 갱신.

#### 4.3.3 보안/운영상 주의

- 과거 IPv4 source routing이 공격 벡터가 되었듯,  
  IPv6 Routing Header도 **보안상 민감**하다.
- 오늘날 대부분의 운영 환경에서는 Type 0를 폐기하고,  
  남은 타입도 제한된 특수 환경에서만 사용하도록 권장한다.  

### 4.4 Fragment Header (값 44)

#### 4.4.1 IPv6에서의 조각화

- IPv4와 달리, IPv6에서는 **중간 라우터가 조각화를 하지 않는다**.  
- 오직 **송신 노드**만 패킷을 조각내어 Fragment Header를 붙인다.  

배경:

- 중간 라우터가 조각화를 수행하면 처리 부하가 크고,  
  보안/운영 측면에서 복잡도 증가.
- 대신, **Path MTU Discovery** 등의 메커니즘을 사용해  
  송신자가 목적지까지의 최소 MTU를 추정하고, 그 크기 이하로 패킷을 보낸다.

#### 4.4.2 구조

```text
+---------------------------------------------------------------+
| Next Header (8)  |  Reserved (8)                             |
+---------------------------------------------------------------+
| Fragment Offset (13) | Res (2) | M flag (1) | Identification |
+---------------------------------------------------------------+
```

- `Fragment Offset`: 이 조각의 시작 위치(8바이트 단위)
- `M flag` (More fragments):
  - 1: 뒤에 더 조각이 있음
  - 0: 마지막 조각
- `Identification`: 모든 조각이 같은 값을 가지며,  
  수신 측이 같은 원본 패킷에 속한 조각들을 그룹핑할 때 사용.

#### 4.4.3 예제: MTU 1280 환경에서 3000바이트 페이로드

- IPv6 최소 MTU는 1280바이트.  
- 3000바이트 TCP 세그먼트를 전송해야 한다고 가정.
- 송신 노드는 Path MTU가 1280으로 판정되면,  
  (추상화해서) IP 레벨에서 3000바이트를 여러 조각으로 나눈다.

간단한 수식:

- 한 조각에서 허용되는 페이로드 최대 크기:

$$
L_{\text{frag\_payload}} = \text{MTU} - L_{\text{IPv6 header}} - L_{\text{Fragment header}}
$$

IPv6 헤더 40바이트, Fragment 헤더 8바이트 가정:

$$
L_{\text{frag\_payload}} = 1280 - 40 - 8 = 1232 \text{ bytes}
$$

3000바이트 페이로드를 나누면:

- 1번째 조각: 1232바이트
- 2번째 조각: 1232바이트
- 3번째 조각: 나머지 536바이트

각 조각에는 동일한 Identification과 서로 다른 Fragment Offset, M flag가 설정된다.

### 4.5 AH (51), ESP(50) — IPsec 확장 헤더

#### 4.5.1 AH (Authentication Header)

- IP 패킷의 무결성과 출처 인증을 제공.
- IP 헤더와 payload 일부를 포함한 **인증 데이터(Integrity Check Value)** 를 담는다.
- 암호화는 하지 않고, 변조/위조 여부 검증에 중점.

#### 4.5.2 ESP (Encapsulating Security Payload)

- IPsec에서 **암호화와 무결성 보호**를 제공.
- 일반적으로:
  - ESP Header + 암호화된 페이로드 + ESP Trailer + ESP Auth
- `Next Header` 필드를 통해 내부의 상위 계층 프로토콜을 가리키며,  
  IPsec VPN, 사이트-투-사이트 터널 등에서 광범위하게 사용된다.  

### 4.6 기타 확장 헤더 예

- **Mobility Header (135)**: Mobile IPv6에서 바인딩 업데이트 등 이동성 관련 시그널링에 사용.  
- **HIP (139)**, **Shim6(140)** 등은 식별자/경로 여러 개를 가진 호스트의 멀티홈/다중 경로 사용을 위해 정의된 프로토콜용.

---

## 5. 예제: Scapy로 IPv6 + 확장 헤더 패킷 만들기

실제 실습에서는 Python + Scapy를 사용하면 확장 헤더 체인을 쉽게 구성해 볼 수 있다.

```python
from scapy.all import IPv6, IPv6ExtHdrHopByHop, IPv6ExtHdrDestOpt, IPv6ExtHdrFragment
from scapy.all import TCP, send

# 1) IPv6 기본 헤더
ip = IPv6(
    src="2001:db8:1000:1010::10",
    dst="2001:db8:2000:abcd::80",
    tc=0x20,          # Traffic Class
    fl=0x12345,       # Flow Label
    hlim=64           # Hop Limit
)

# 2) Hop-by-Hop 확장 헤더 (간단히 빈 옵션)
hbh = IPv6ExtHdrHopByHop()

# 3) Destination Options 확장 헤더
dest_opt = IPv6ExtHdrDestOpt()

# 4) TCP 헤더
tcp = TCP(
    sport=50000,
    dport=443,
    flags="S",        # SYN
    seq=1000
)

pkt = ip / hbh / dest_opt / tcp

# 실제 전송 (테스트 환경에서만!)
# send(pkt)
print(pkt.show(dump=True))
```

이 코드를 통해:

- IPv6 고정 헤더 + Hop-by-Hop + Destination Options + TCP 구조를 눈으로 확인할 수 있다.
- `show()` 결과를 보면 `Next Header` 체인이 어떻게 설정되는지 확인 가능.

---

## 6. 운영·보안 관점에서 확장 헤더

### 6.1 확장 헤더와 드롭 문제

RFC 9098은 실제 인터넷에서 **확장 헤더를 포함한 패킷이 종종 드롭되는 이유**를 자세히 분석한다.  

대표적인 이유:

1. **라우터의 패킷 처리 엔진 한계**  
   - 깊은 패킷 검사(Deep inspection)를 위해 L4 포트까지 보려면,  
     확장 헤더 체인을 모두 따라가야 한다.
   - 너무 많은 확장 헤더/비정상적인 체인을 가진 패킷은 **성능/보안 위험**으로 간주.

2. **보안 정책**  
   - 일부 방화벽/IPS는 **알 수 없는 확장 헤더** 나  
     **너무 긴 헤더 체인을 가진 패킷**을 차단하도록 설정된다.

이를 해결하기 위해, 최근 드래프트(RFC 9288, EH filtering draft 등)는 다음을 권장한다.  

- 필요한 확장 헤더만 허용하는 **permit-list(화이트리스트)** 전략
- 비정상적인 확장 헤더 체인(너무 길거나, 순서가 이상한 등) 드롭
- 애플리케이션/프로토콜 설계 시 **확장 헤더를 남용하지 말 것**

### 6.2 한계값 설정: EH Limits

`draft-ietf-6man-eh-limits` 문서는 **확장 헤더가 들어간 패킷의 처리 한계값** (최대 길이, 헤더 수 등)을 제안한다.  

목적:

- 모든 노드가 최소한 어느 정도까지는 확장 헤더를 수용하도록 공통 기준 제시
- 너무 긴 헤더 체인을 허용하면 공격 표면 확대 → 적절한 상한 필요

실무적으로는:

- “확장 헤더가 L4 헤더를 너무 멀리 밀어내지 않도록”  
  헤더 체인의 총 길이에 제한을 두는 경우가 많다.

---

## 7. 요약 정리

**22.2 The IPv6 Protocol** 에서 핵심은:

1. **Packet Format**  
   - IPv6 고정 헤더는 40바이트, 필수 필드만 남기고 최대한 단순화.
   - Traffic Class, Flow Label, Payload Length, Next Header, Hop Limit, Source/Dest Address로 구성.
   - IPv4의 옵션, 조각화 관련 필드는 모두 확장 헤더로 이동.

2. **Extension Header**  
   - 고정 헤더 뒤에 이어지는 옵션/기능 모듈.
   - Hop-by-Hop, Destination Options, Routing, Fragment, AH, ESP, Mobility 등 다양한 타입.
   - Next Header 체인을 통해 순차적으로 이어 붙임.
   - 권장 순서와 실제 라우터/방화벽의 처리 특성을 이해해야 함.

3. **현실적인 운영 관점**  
   - 확장 헤더는 매우 강력하지만, 성능·보안상의 이유로 **인터넷 상에서 자주 드롭**되는 문제가 있다.
   - RFC 9098, RFC 9099, EH filtering/limits 문서 등은  
     “필요한 확장 헤더만 사용하고, 체인을 과도하게 복잡하게 만들지 말 것”을 강조.