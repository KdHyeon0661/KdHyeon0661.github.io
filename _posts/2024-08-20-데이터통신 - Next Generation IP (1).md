---
layout: post
title: 데이터 통신 - Next Generation IP (1)
date: 2024-08-20 20:20:23 +0900
category: DataCommunication
---
# Chapter 22. Next Generation IP — 22.1 IPv6 Addressing

이 장에서는 IPv4 이후 세대를 위한 프로토콜인 **IPv6의 주소 체계**를 깊게 파본다.
특히 22.1 절에서 요구하는 다섯 가지 축:

- **Representation**: 128비트 IPv6 주소를 텍스트로 표현하는 법
- **Address Space**: 거대한 128비트 주소 공간의 구조
- **Address Space Allocation**: IANA → RIR → ISP → 엔터프라이즈로 내려가는 할당 구조
- **Autoconfiguration**: SLAAC, DHCPv6를 통한 자동 주소 설정
- **Renumbering**: ISP나 프리픽스 변경 시, IPv6가 어떻게 “덜 아픈” 재번호 부여를 지원하는지

---

## Representation — IPv6 주소 표기법

### 128비트 구조와 기본 표기

IPv6 주소는 **128비트** 길이이며, 보통 **16비트 단위 8개**를 16진수로 표현하고, 각 16진수 그룹을 `:` 로 구분한다. 이를 흔히 *colon-hexadecimal notation*이라고 부른다. 기본 형식:

```text
xxxx:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx
```

각 `x` 는 16진수 한 자리(0–9, a–f)이고, 각 그룹은 4자리 16진수 → 16비트이다.

예:

```text
2001:0db8:0000:0000:0000:ff00:0042:8329
fe80:0000:0000:0000:021c:7eff:fe39:8abc
```

- 앞 예제의 `2001:0db8::/32` 는 **문서 예제용(prefix 2001:db8::/32)** 으로 IETF에서 예약한 공간이다.
- 실제 전 세계 ISP/엔터프라이즈에 배포되는 글로벌 유니캐스트 주소는 주로 **2000::/3** 블록에서 할당된다.

### 생략과 `::` 압축 — RFC 5952 규칙

128비트를 매번 풀로 쓰면 너무 길기 때문에, IPv6는 **0을 생략하고 연속된 0을 `::`로 압축하는 규칙**을 제공한다. 텍스트 표현에 대한 사실상 표준은 **RFC 5952**에 정의된 “canonical representation”이다.

핵심 규칙:

1. **leading zero 제거**
   - 각 16비트 그룹에서 앞쪽 0은 모두 생략
   - 예: `00ab` → `ab`, `0000` → `0`
2. **연속된 0 그룹을 `::` 하나로 압축**
   - 한 주소에서 **단 한 번만** 사용 가능
   - 가장 긴 0 시퀀스를 압축 (동률이면 왼쪽 것을 선택)
3. 16진수는 **소문자** 사용 (`a`–`f`)
4. 8개 그룹이 항상 동일한 순서로 유지되며, **압축은 중간 또는 양 끝에만 위치** 가능

예제 변환:

- 원본: `2001:0db8:0000:0000:0000:ff00:0042:8329`
  1) leading zero 제거 → `2001:db8:0:0:0:ff00:42:8329`
  2) 연속된 `0:0:0` 을 `::` 로 압축 →
     `2001:db8::ff00:42:8329`

- 원본: `fe80:0000:0000:0000:021c:7eff:fe39:8abc`
  → `fe80::21c:7eff:fe39:8abc`

잘못된 압축 예:

```text
# 잘못된 예 (두 번 :: 사용)

2001:db8::ff00::8329   # X

# 잘못된 예 (가장 긴 0시퀀스를 압축하지 않음)

2001:db8:0:0:0:ff00:42:8329 → 2001:db8:0:0::ff00:42:8329  # X
```

### IPv6 prefix 표기(/64 등)

IPv4의 `192.0.2.0/24` 와 같은 방식으로, IPv6 역시 **CIDR 스타일 prefix**를 사용한다.

```text
2001:db8:1234:5678::/64
```

- 앞 64비트: 네트워크 prefix
- 뒤 64비트: 인터페이스 식별자

**IPv6의 일반 원칙**: 대부분의 일반 LAN 서브넷은 `/64` 를 사용한다 (SLAAC, 보안·성능·아이덴티티 관련 RFC들이 `/64` 가정).

### IPv4-임베디드 주소

IPv4와 IPv6 공존 및 전환을 위해 **IPv4-mapped IPv6 address** 등의 형식도 존재한다 (RFC 4291, RFC 5952).

대표적인 형식:

- IPv4-mapped: `::ffff:192.0.2.1`

예: `192.0.2.1` 을 IPv6로 표현:

```text
::ffff:c000:0201
```

혹은 혼합 표기: `::ffff:192.0.2.1`

이는 주로 OS 내부 스택, API 등에서 IPv4/IPv6를 공통 인터페이스로 다루기 위해 사용된다.

### 코드로 IPv6 문자열 다루기 (파이썬 예제)

실제 블로그 글에서 IPv6 문자열을 검증·정규화할 때는 OS/언어 라이브러리를 사용하는 것이 안전하다.
파이썬 표준 라이브러리 `ipaddress`는 RFC 5952를 따라 IPv6 텍스트를 canonical 형태로 반환한다.

```python
import ipaddress

def normalize_ipv6(addr: str) -> str:
    """
    IPv6 문자열을 표준 형태(RFC 5952 스타일)로 정규화한다.
    잘못된 주소면 ValueError 예외 발생.
    """
    ip = ipaddress.IPv6Address(addr)
    return ip.compressed  # leading zero 제거, :: 압축 적용

# 사용 예

samples = [
    "2001:0db8:0000:0000:0000:ff00:0042:8329",
    "2001:db8::ff00:42:8329",
    "fe80:0000:0000:0000:021c:7eff:fe39:8abc",
]

for s in samples:
    print(s, "→", normalize_ipv6(s))
```

이 코드를 실행하면, 서로 다른 표현이 동일한 canonical 표현으로 정규화되는 것을 확인할 수 있다.

---

## Address Space — IPv6 주소 공간의 구조

### 2¹²⁸ 이라는 규모

IPv6 주소는 128비트이므로, 전체 가능한 주소 개수는

$$
2^{128} \approx 3.4 \times 10^{38}
$$

이다. 이는 대략:

- IPv4의 $$2^{32} \approx 4.3\times 10^9$$ 개보다
- $$2^{96} \approx 7.9\times 10^{28}$$ 배 더 많은 크기이다.

즉, “지구 상의 모든 사람에게 수많은 주소를 나눠줘도 남는다” 수준이 아니라,
**전 우주에 모래알 단위로 뿌려도 남는다**에 가까운 규모다.

### IANA IPv6 Address Space Registry

IANA는 전체 IPv6 공간을 여러 블록으로 나누고, 일부는 특별한 용도에 예약해 두었다.
2025년 10월 기준, IANA의 IPv6 주소 공간 레지스트리는 다음과 같은 주요 블록을 정의한다.

대표적인 범위만 요약하면:

| 프리픽스 | 용도(요약) |
|---------|-----------|
| `::/128` | Unspecified address (0.0.0.0에 해당) |
| `::1/128` | Loopback 주소 (127.0.0.1에 해당) |
| `::/96` | IPv4-compatible (과거 용도, 현재는 deprecated) |
| `::ffff:0:0/96` | IPv4-mapped IPv6 주소 |
| `2000::/3` | Global Unicast Address (GUA) – 인터넷용 유니캐스트 주소 |
| `fc00::/7` | Unique Local Address (ULA) – IPv4의 RFC1918 유사, 사설·내부용 |
| `fe80::/10` | Link-Local Unicast – 로컬 링크에서만 사용, 라우팅되지 않음 |
| `ff00::/8` | Multicast |

`2000::/3` 블록만 해도 전체 IPv6의 1/8 (2⁻³)을 차지하며, 이 범위에서 전 세계 ISP/엔터프라이즈에 글로벌 주소를 할당한다.

### 네트워크/인터페이스 구조 (GUA 예)

RFC 3587 등에서는 글로벌 유니캐스트 주소를
다음과 같은 3단 구조로 이해하도록 권장한다.

```text
|   글로벌 라우팅 프리픽스   | 서브넷 ID |   인터페이스 ID   |
|-------- n bits -----------| 16 bits   |     64 bits        |
```

예를 들어 `/48` 사이트 프리픽스를 받은 경우:

- 상위 n비트: ISP에서 준 사이트 프리픽스 (예: `2001:db8:1234::/48`)
- 그 다음 16비트: 사이트 내부 서브넷 ID (`0001`, `0002`, … → /64 서브넷)
- 마지막 64비트: 인터페이스 ID (SLAAC, 수동 설정, 랜덤 등)

이렇게 하면, `/48` 하나로 **최대 2¹⁶ = 65,536 개의 /64 서브넷**을 만들 수 있다.

---

## Address Space Allocation — IANA → RIR → ISP → 엔터프라이즈

### 상위 구조: IANA와 RIR

**IANA**는 IPv6 글로벌 유니캐스트 주소의 할당을 관리하며,
현재는 주로 `2000::/3` 블록에서 **RIR(Regional Internet Registry)** 들에게 대규모 프리픽스를 할당한다.

RIR 예:

- **ARIN**: 북미
- **RIPE NCC**: 유럽, 중동, 일부 중앙아시아
- APNIC 등도 있지만, 질문에서 “아시아권 자료는 참고하지 말라”고 했으므로 여기서는 설명만 하고 구체 정책은 유럽/미국 기준(RIPE/ARIN) 자료를 중심으로 정리한다.

IANA → RIR로 내려가는 할당은 대개 `/12`, `/16` 같은 거대한 블록 단위다.

### RIR → LIR/ISP → 엔터프라이즈

예를 들어 RIPE NCC 정책 문서를 보면, 아래와 같은 일반 원칙을 볼 수 있다.

- RIR는 **LIR(Local Internet Registry)** 또는 ISP에 `/29`, `/32`, `/29` 등 비교적 큰 프리픽스를 할당한다.
- ISP/LIR는 이 블록을 잘게 쪼개어:
  - 자체 백본·포인트 간 링크 등에 `/48`, `/64` 등 부여
  - 엔터프라이즈/고객에게 `/48`, `/56`, `/64` 등 부여

전통적인 권고(여러 RFC와 RIR 정책에서 반복 등장):

- **엔터프라이즈/사이트**: `/48`
- **중소 규모 / 가정용 브로드밴드**: `/56` 또는 `/64`
- **단일 링크**만 필요하면 `/64`

예시 할당 흐름:

```text
IANA
  └─ 2000::/3 (Global Unicast)
      └─ RIPE NCC: 2001:db8::/29   (예시)
          └─ ISP A: 2001:db8:1000::/32
              ├─ 엔터프라이즈 X: 2001:db8:1000:1000::/48
              ├─ 엔터프라이즈 Y: 2001:db8:1000:2000::/48
              └─ 가정용 고객 Z: 2001:db8:1000:3000::/56
```

엔터프라이즈 X는 `/48` 하나로 내부에 65,536개의 `/64` 서브넷을 만들 수 있으므로,
IPv4에서 흔히 겪었던 “주소가 모자라서 서브넷 쪼개기, NAT 꼼수” 문제를 크게 줄일 수 있다.

### 서브넷 할당 예제

**엔터프라이즈 X** 가 `2001:db8:1000:1000::/48` 을 받았다고 가정하자.

서브넷 설계 예:

| 서브넷 ID (16비트) | /64 프리픽스                            | 용도         |
|--------------------|------------------------------------------|--------------|
| `0000`             | `2001:db8:1000:1000::/64`                | 코어 백본    |
| `0001`             | `2001:db8:1000:1001::/64`                | 서버 존 1    |
| `0002`             | `2001:db8:1000:1002::/64`                | 서버 존 2    |
| `0010`             | `2001:db8:1000:1010::/64`                | 무선 LAN     |
| `00ff`             | `2001:db8:1000:10ff::/64`                | 관리 전용    |

IPv6에서는 “서브넷이 아까워서 서버/클라이언트를 같은 브로드캐스트 도메인에 다 욱여 넣는” 식의 구조를 피하고,
**보안·운영 편의를 위해 서브넷을 넉넉하게 쪼개는 방식**이 권장된다.

---

## Autoconfiguration — 자동 주소 설정 (SLAAC, DHCPv6)

IPv6 설계의 큰 목표 중 하나는:

> “**IPv4보다 훨씬 자동화된 주소 설정과 재구성**”

이다. 이 부분은 **Stateless Address Autoconfiguration(SLAAC)** 을 정의한 RFC 4862가 핵심이다.

### SLAAC 기본 흐름

SLAAC의 중심 메커니즘:

1. **링크-로컬 주소 생성 (fe80::/64)**
2. **Router Solicitation / Router Advertisement** 를 통해 프리픽스 학습
3. 프리픽스 + 인터페이스 ID → 글로벌 유니캐스트 주소 생성
4. **Duplicate Address Detection(DAD)** 로 중복 확인
5. 성공 시, 주소를 “preferred” 상태로 사용

#### 링크-로컬 주소 생성

IPv6 인터페이스는 **반드시** 링크-로컬 주소를 가져야 한다.
형식: `fe80::/64 + 인터페이스 ID(64비트)`

인터페이스 ID는 과거에는 **EUI-64** 기반(맥주소에서 도출)이 일반적이었지만,
프라이버시 이유로 현재는 랜덤 또는 임의 값으로 생성되는 경우가 많다 (RFC 8981의 temporary addresses).

#### Router Advertisement 수신

호스트는 부팅 시:

- 자신이 속한 링크에 **라우터가 있는지** 확인하기 위해 **Router Solicitation (RS)** 메시지를 보낼 수 있다.
- 라우터는 주기적으로 또는 RS에 응답하여 **Router Advertisement (RA)** 를 보낸다.

RA에는 다음 정보가 포함될 수 있다.

- Prefix 정보: 예) `2001:db8:1000:1010::/64`
- 플래그:
  - **A(Autonomous)**: SLAAC로 이 프리픽스를 사용해 주소를 만들어도 되는지
  - **M(Managed)**: DHCPv6를 통해 주소를 받아야 하는지
  - **O(Other)**: DNS 등 기타 설정은 DHCPv6로 받는지

#### 글로벌 주소 생성

RA에서 `A=1` 인 프리픽스를 받으면, 호스트는:

```text
글로벌 프리픽스 (예: 2001:db8:1000:1010::/64)
+ 인터페이스 ID (64비트, 랜덤/임의/EUI-64)
= 글로벌 유니캐스트 주소
예: 2001:db8:1000:1010:abcd:ef01:2345:6789
```

이렇게 생성된 주소는 “tentative” 상태이며,
다음 단계인 **DAD** 를 통해 중복 여부를 확인한다.

#### Duplicate Address Detection(DAD)

- 호스트는 자신이 사용하려는 주소에 대해 **Neighbor Solicitation** 메시지를 보내고,
- 아무도 응답하지 않으면 해당 주소를 사용해도 된다고 판단한다.

만약 응답이 오면, 충돌이므로:

- 다른 인터페이스 ID를 생성해 다시 시도하거나
- 관리자가 수동 설정해야 한다.

### 프라이버시 확장: Temporary / Stable-privacy 주소

MAC에서 인터페이스 ID를 만드는 전통적 방식은,
“한 번 만들어진 주소가 네트워크를 바꿔도 그대로 유지”되는 특성이 있어서 **트래킹 위험**이 있다.

이에 따라:

- **RFC 8981** 등은 **랜덤화된 인터페이스 ID** 를 만들어,
  - 추적 위험을 줄이기 위해 일정 시간마다 새 임시(temporary) 주소를 만들고,
  - 동시에 안정적인 통신을 위해 stable한 주소도 유지하라고 권장한다.

실제 OS들(Windows, Linux, macOS)은 기본적으로 이 프라이버시 확장을 활성화하는 경우가 많다.

### DHCPv6와의 관계

IPv6에서 주소 설정은 보통 **SLAAC + DHCPv6** 의 조합으로 설계된다.

- RA의 M/O 플래그로 정책 표시:
  - M=1, O=0: 주소 및 기타 정보 모두 DHCPv6
  - M=0, O=1: 주소는 SLAAC, DNS 등 기타 정보는 DHCPv6
  - M=0, O=0: 주소와 기타 정보를 모두 SLAAC/수동으로

엔터프라이즈에서는 **주소는 SLAAC, DNS·NTP·기타 옵션은 DHCPv6** 조합이 꽤 흔하다(예: RFC 7381에서 enterprise deployment 가이드라인으로 언급).

### 실제 Linux 예시

리눅스에서 네트워크에 연결하고 `ip -6 addr` 을 실행하면 다음과 같은 결과를 볼 수 있다 (개념 예):

```bash
$ ip -6 addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet6 fe80::21c:7eff:fe39:8abc/64 scope link
       valid_lft forever preferred_lft forever
    inet6 2001:db8:1000:1010:21c:7eff:fe39:8abc/64 scope global dynamic
       valid_lft 2592000sec preferred_lft 604800sec
    inet6 2001:db8:1000:1010:abcd:ef01:2345:6789/64 scope global temporary dynamic
       valid_lft 604800sec preferred_lft 86400sec
```

- `fe80::21c:7eff:fe39:8abc/64` : 링크-로컬
- `2001:db8:...:21c:7eff:fe39:8abc` : stable 글로벌 주소
- `2001:db8:...:abcd:ef01:2345:6789` : 프라이버시용 temporary 주소

`valid_lft`/`preferred_lft`는 이 주소의 **유효/선호 기간**으로,
뒤에서 설명할 **renumbering** 에서 중요한 역할을 한다.

---

## 기법

IPv4에서 “ISP를 바꾼다” 혹은 “주소 계획을 전면 수정한다”는 것은 큰 고통이었다.
NAT, 수동 설정, 하드코딩된 주소, ACL/방화벽 규칙 등 수많은 곳을 수정해야 했기 때문이다.

IPv6는 설계 단계에서 **“더 쉬운 네트워크 재번호 부여”** 를 목표로 했다.
RFC 6879는 엔터프라이즈 IPv6 재번호 부여 시나리오와 방법을 분석하며,
RFC 7381은 엔터프라이즈 IPv6 도입 가이드에서 renumbering을 주요 고려사항으로 다룬다.

### 핵심 아이디어

1. **프리픽스 수준에서 재번호 부여**
   - 주소의 상위 prefix(예: `/48`, `/64`)만 바꿔도 자동으로 새 주소가 생성되도록 SLAAC 설계
2. **동시 다중 프리픽스 지원**
   - 하나의 링크에 **여러 프리픽스를 동시에 광고** 가능 (old + new)
3. **주소 유효/선호 기간으로 부드러운 전환**
   - RA에서 프리픽스마다 `Valid Lifetime`, `Preferred Lifetime` 을 조정하여
     *구(prefix)는 점점 덜 선호되고, 결국 제거*되게 함
4. DHCPv6도 마찬가지로 새로운 prefix로 설정을 갱신할 수 있고,
   - DNS 기록, 방화벽, ACL 등을 자동화 스크립트로 갱신하도록 설계

### RA 기반 renumbering 예제

엔터프라이즈 X가 기존 ISP A의 프리픽스 `2001:db8:1000:1000::/48` 을 사용하다가,
ISP B로 옮기면서 새 프리픽스 `2001:db8:abcd:0000::/48` 을 받았다고 하자.

각 서브넷에서 기존에 RA를 통해:

```text
Prefix: 2001:db8:1000:1010::/64
Valid Lifetime: 2592000 sec (30일)
Preferred Lifetime: 604800 sec (7일)
```

로 광고하던 것을, 재번호 부여 계획에 따라 다음과 같이 조정할 수 있다.

1단계: 새 프리픽스 추가

```text
기존: 2001:db8:1000:1010::/64 (old)
새로: 2001:db8:abcd:1010::/64 (new)

RA 설정:
- old 프리픽스: Valid 30일, Preferred 3일
- new 프리픽스: Valid 30일, Preferred 30일
```

호스트 동작:

- 새 프리픽스를 받아 **새 글로벌 주소** 생성
- old 주소는 유효하지만, preferred 기간이 짧으므로
  → 새 outgoing 연결에는 **새 주소**를 선호

2단계: old 프리픽스 제거

- 일정 기간이 지나면, RA에서 old 프리픽스의 `Valid Lifetime` 도 매우 짧게 줄이거나 0으로 설정
- 호스트는 old 주소를 더 이상 사용하지 않고 삭제

이렇게 하면 **연결 중단 없이 천천히 주소를 교체**할 수 있고,
서버 측은 DNS RR, 방화벽, ACL 등의 prefix를 스크립트/자동화로 함께 갱신하면 된다.

### 수식으로 보는 prefix 분할과 재번호

예를 들어, 엔터프라이즈가 기존에 `/48` 이고 새로도 `/48` 을 받는 경우:

- 기존 `/48`: `P_old`
- 새 `/48`: `P_new`

기존 서브넷 `/64` 들은:

$$
P_{\text{old},i} = P_{\text{old}} \oplus \text{subnet\_id}_i
$$

새 서브넷은:

$$
P_{\text{new},i} = P_{\text{new}} \oplus \text{subnet\_id}_i
$$

여기서 `⊕` 는 단순한 비트 결합(상위 prefix + 하위 subnet ID)을 의미한다.

즉, **서브넷 ID 설계를 그대로 유지**한 채
prefix만 바꾸면 전체 서브넷 구조를 그대로 옮길 수 있다.

### DHCPv6 기반 renumbering

SLAAC 대신, 혹은 함께 DHCPv6를 쓰는 경우:

- DHCPv6 서버에서 prefix/pool을 `P_old` → `P_new` 로 변경하고
- 임대 기간(lease time)을 조정하여 **old 주소의 만료를 가속**할 수 있다.
- 클라이언트가 갱신(renew)을 하면서 새 prefix 주소를 받게 되고,
  RA의 prefix lifetime과 함께 조합하면 부드러운 전환 구현 가능

RFC 6879는 엔터프라이즈가 실제로 renumbering 할 때 고려해야 할:

- DNS, NTP, VPN, 방화벽, NAT64 등 주변 인프라 변경
- 주소 계획 문서화와 자동화 도구 사용

등을 자세히 다룬다.

### 간단한 자동화 스크립트 예 (prefix 치환)

주소가 설정 파일 여러 곳에 하드코딩되어 있다면,
프리픽스 기반 치환을 간단한 스크립트로 처리할 수 있다. 아래는 Python 예제:

```python
from ipaddress import IPv6Address, IPv6Network

OLD_PREFIX = IPv6Network("2001:db8:1000:1000::/48")
NEW_PREFIX = IPv6Network("2001:db8:abcd:0000::/48")

def renumber_ipv6(addr_str: str) -> str:
    """
    기존 prefix(OLD_PREFIX)를 새 prefix(NEW_PREFIX)로 치환하여
    재번호 부여된 주소를 반환한다. prefix가 다르면 그대로 반환.
    """
    ip = IPv6Address(addr_str)
    if ip in OLD_PREFIX:
        # 인터페이스 ID + 서브넷 ID 부분만 추출
        host_bits = int(ip) & ((1 << (128 - OLD_PREFIX.prefixlen)) - 1)
        new_int = (int(NEW_PREFIX.network_address) |
                   host_bits)
        return str(IPv6Address(new_int))
    else:
        return addr_str

# 사용 예

old = "2001:db8:1000:1010:21c:7eff:fe39:8abc"
print("OLD:", old)
print("NEW:", renumber_ipv6(old))
```

실제로는 방화벽 설정, 서버 설정 파일, DNS zone 파일 등을
이런 치환 함수와 정규표현식 조합으로 자동 변환하는 식으로 renumbering 작업을 단순화할 수 있다.

---

## 정리

22.1 IPv6 Addressing에서 다룬 핵심 내용을 정리하면:

1. **Representation**
   - 128비트 주소를 16비트 그룹 8개로 `:` 로 구분하는 colon-hex 표기
   - leading zero 제거, 가장 긴 0 시퀀스를 `::`로 한 번만 압축하는 RFC 5952 규칙
   - IPv4-mapped 등 전환용 주소 형식

2. **Address Space**
   - $$2^{128} \approx 3.4\times 10^{38}$$ 의 거대한 공간
   - IANA가 `2000::/3` 를 글로벌 유니캐스트로 운영하고,
     `fe80::/10`(link-local), `fc00::/7`(ULA), `ff00::/8`(multicast) 등 특별 블록 정의

3. **Address Space Allocation**
   - IANA → RIR(ARIN/RIPE 등) → LIR/ISP → 엔터프라이즈/고객 순의 계층적 할당
   - 엔터프라이즈에는 보통 `/48` 또는 `/56` 정도, 서브넷은 `/64` 사용
   - 풍부한 주소 공간 덕분에 보안·운영 관점에서 서브넷을 넉넉하게 쪼갤 수 있음

4. **Autoconfiguration**
   - SLAAC: 링크-로컬 생성 → RA 기반 prefix 학습 → 글로벌 주소 생성 → DAD 수행
   - RA의 A/M/O 플래그와 DHCPv6 의 조합으로 주소 및 기타 설정 자동화
   - RFC 8981 등에 정의된 프라이버시 확장으로 랜덤 인터페이스 ID, temporary 주소 사용

5. **Renumbering**
   - 라우터가 old/new 프리픽스를 동시에 광고하고, prefix lifetime을 조정하여
     “천천히 old 주소를 퇴출하는” 부드러운 재번호 부여 가능
   - SLAAC + DHCPv6 + 자동화 스크립트 조합으로,
     IPv4 시대보다 훨씬 덜 고통스러운 ISP 변경, 주소 계획 재구성이 가능
