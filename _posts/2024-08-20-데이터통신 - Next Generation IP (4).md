---
layout: post
title: 데이터 통신 - Next Generation IP (4)
date: 2024-08-20 23:20:23 +0900
category: DataCommunication
---
# Chapter 22. Next Generation IP — 22.4 Transition from IPv4 to IPv6

IPv4(32비트 주소 공간)는 이미 전 세계 공용 주소가 고갈되었고, 그 결과로 대규모 NAT/CGN(Carrier-Grade NAT)에 의존하는 구조가 되었다.
IPv6(128비트 주소 공간)는 이 문제를 근본적으로 해결하지만, **IPv4와 IPv6는 프로토콜 수준에서 직접 호환되지 않는다.**

그래서 IETF는 “IPv4 → IPv6 전환”을 한 번에 갈아타는 방식이 아니라, **IPv4와 IPv6가 장기간 공존(coexistence)** 하는 것을 목표로 설계했다.
이 절에서는 전환을 위한 **전략(Strategies)** 과, 그 속에서 **IP 주소가 실제로 어떻게 사용되는지(Use of IP Address)** 를 깊게 정리한다.

---

## Strategies

전환 전략은 크게 세 가지 축으로 분류할 수 있다.

1. **Dual Stack** — 노드/라우터가 IPv4와 IPv6 스택을 동시에 운영
2. **Tunneling** — 한 프로토콜을 다른 프로토콜 안에 캡슐화해 전달
3. **Translation** — IPv4 헤더와 IPv6 헤더를 상호 변환(NAT64 등)

실제 네트워크에서는 이 세 가지를 **상황에 맞게 조합**해서 사용한다.

### 설계 목표: 공존과 점진적 전환

전환 메커니즘은 보통 다음 목표를 갖는다.

1. **Coexistence (공존)**
   - IPv4-only, IPv6-only, Dual-stack 노드가 **동시에 존재**해도 통신이 되어야 한다.

2. **Incremental (점진적)**
   - 라우터/서브넷/서비스 단위로 **조금씩 전환**하고, 문제시 되돌릴 수 있어야 한다.

3. **Transparency (투명성)**
   - 가능하면 애플리케이션은 **주소 literal 등 일부만 수정**하고, 나머지는 변경 없이 동작하도록 한다.

4. **Scalability & Security (확장성과 보안)**
   - 대규모 ISP/모바일 네트워크에도 적용 가능해야 하고,
     NAT/터널/번역 구간에서의 보안 리스크를 줄여야 한다.

이제 이 목표를 만족시키기 위해 등장한 각 전략을 구체적으로 보자.

---

## Dual Stack

### 개념

**Dual Stack** 은 호스트나 라우터가 **IPv4와 IPv6 프로토콜 스택을 모두 구현**하고,
하나의 인터페이스에 IPv4·IPv6 주소를 동시에 설정하는 방식이다.

예를 들어, 어떤 서버의 인터페이스가 다음과 같을 수 있다.

- IPv4: `192.0.2.10/24`
- IPv6: `2001:db8:100:1::10/64`

운영체제는 DNS 응답(A/AAAA 레코드)에 따라 **어떤 프로토콜(IP 버전)을 사용할지** 결정한다.
보통 주소 선택 규칙에 따라 IPv6 경로가 존재하면 **IPv6를 우선 사용**한다.

### 예제 — Dual-stack 웹 서버

#### 네트워크 설정 예

데이터센터에 웹 서버 한 대를 두고 전 세계 사용자에게 서비스한다고 하자.
서버 OS에서 다음처럼 설정할 수 있다.

```bash
# IPv4 주소 설정

ip addr add 192.0.2.10/24 dev eth0
ip route add default via 192.0.2.1

# IPv6 주소 설정

ip addr add 2001:db8:100:1::10/64 dev eth0
ip -6 route add default via 2001:db8:100:1::1
```

DNS 레코드는 이렇게 설정한다.

```text
example.com.   IN A     192.0.2.10
example.com.   IN AAAA  2001:db8:100:1::10
```

웹 서버(예: nginx)는 IPv4와 IPv6 모두에서 listen 한다.

```bash
# nginx 예시

server {
    listen 80 default_server;       # IPv4
    listen [::]:80 default_server;  # IPv6
    server_name example.com;
    ...
}
```

#### 클라이언트 동작

- IPv6를 지원하는 ISP 사용자는 AAAA 레코드를 우선 사용 → IPv6로 접속
- IPv4-only 사용자는 A 레코드를 사용 → IPv4로 접속

애플리케이션 관점에서는 “같은 도메인, 같은 포트(80/443)를 사용하는 하나의 서비스”로 보이지만,
실제로는 **두 주소 패밀리(IPv4, IPv6)** 를 동시에 쓰는 구조다.

### Dual Stack의 장단점

**장점**

- 개념적으로 가장 단순하다.
  → “IPv4 네트워크 + IPv6 네트워크”를 그냥 나란히 운용.
- 애플리케이션 수정 최소화.
  → OS 수준에서 주소 선택만 잘 하면 된다.
- 문제 발생 시 IPv4로 쉽게 롤백할 수 있다.

**단점**

- 라우팅·방화벽·모니터링·주소 계획을 **두 벌 관리**해야 한다.
- 공인 IPv4 주소가 더 이상 충분하지 않으면, 신규 서비스에 dual-stack을 적용하기 어렵다.
- 대형 ISP 입장에서는 코어망을 dual-stack으로 유지하는 것이 비용/복잡도 측면에서 부담.

그래서 실제로는 **코어는 IPv6-only, 엣지에서 IPv4를 번역/터널**하는 패턴이 증가하는 추세다.

---

## Tunneling

IPv6가 아직 도입되지 않은 구간(IPv4-only 백본)을 지나야 할 때,
**IPv6 패킷을 IPv4 패킷 안에 캡슐화(encapsulation)** 해서 보내는 방식이다.
반대로, IPv4를 IPv6 위에 싣는 터널도 있다.

### 기본 원리

IPv6-over-IPv4 일반 형식:

```text
[IPv4 header][IPv6 header][IPv6 payload]
```

IPv4 헤더의 `Protocol` 필드 값이 41이면 “이 패킷 payload는 IPv6”라는 의미가 된다.

수식으로 표현하면:

$$
\text{IPv4\_packet} = \text{IPv4\_header} \parallel \text{IPv6\_packet}
$$

여기서 $\parallel$ 는 단순한 바이트 연결(concatenation)을 의미한다.

### Configured Tunnel (수동 터널)

관리자가 양 끝단의 IPv4 주소를 미리 알고 설정해 두는 방식을 흔히 **Configured Tunneling** 이라고 부른다.

#### 시나리오

- 지역 A: IPv6-only 사이트 A
- 지역 B: IPv6-only 사이트 B
- A와 B 사이: IPv4-only 백본

이를 텍스트 그림으로 표현하면:

```text
[IPv6 LAN A]--(R1)--<IPv6 over IPv4 터널>--(R2)--[IPv6 LAN B]
                 ===== IPv4-only 백본 =====
```

R1과 R2는 서로의 IPv4 주소를 알고 있으며, 그 사이에 IPv6-over-IPv4 터널 인터페이스를 만든다.

#### 리눅스 예시 (개념적)

```bash
# R1에서

ip tunnel add tun0 mode sit remote 198.51.100.2 local 198.51.100.1 ttl 64
ip addr add 2001:db8:10:1::1/64 dev tun0
ip link set tun0 up

# R2에서

ip tunnel add tun0 mode sit remote 198.51.100.1 local 198.51.100.2 ttl 64
ip addr add 2001:db8:10:1::2/64 dev tun0
ip link set tun0 up
```

이제 `2001:db8:10:1::/64` 는 R1–R2 사이의 터널 위에 존재하는 IPv6 링크처럼 동작한다.

### 자동/특수 터널 (6to4, Teredo, ISATAP 등)

과거에는 “IPv6 인프라가 거의 없던 시절”에 다음과 같은 자동 터널 메커니즘이 많이 쓰였다.

- **6to4**: 공인 IPv4 주소를 가진 노드가 자동으로 `2002:IPv4_hex::/48` prefix를 얻는 방식
- **Teredo**: NAT 뒤 클라이언트가 IPv6를 UDP 위에 캡슐화하여 외부로 나가는 방식
- **ISATAP**: 기업 내부에서 IPv4-only 인프라 위에 IPv6 “섬”을 만드는 구조

오늘날에는 운영/보안/성능 문제로 인해 대부분 **비권장(deprecated/obsolete)** 취급이며,
실제 설계에서는 사용하지 않는 것이 일반적이다.

### 현대식 활용 — IPv6-only 코어 + 터널 기반 IPv4 운반

요즘 ISP 설계에서 많이 보는 패턴:

- 코어망: MPLS + IPv6-only
- 고객 IPv4 트래픽: GRE, L2TPv3, IPv4-in-IPv6 터널로 코어를 통과
- 경계(CGN이나 BR)에서 NAT, 변환, 분기

이 경우 터널은 단순 전환 수단이 아니라,
**“논리적인 서비스/고객 경계”** 로 상시 활용된다.

---

## Translation (NAT64, DS-Lite, 464XLAT 등)

Translation 계열은 **IPv4와 IPv6 헤더를 상호 변환**해
IPv6-only 노드와 IPv4-only 노드가 서로 통신할 수 있게 한다.

### NAT64 + DNS64

**NAT64** 는 “IPv6-only 클라이언트 ↔ IPv4-only 서버”를 연결하는 기술이다.
핵심 개념:

1. NAT64 게이트웨이는 **IPv6 prefix** 와 **IPv4 주소 풀**을 가진다.
2. DNS64는 IPv4-only 서버의 A 레코드를 보고,
   NAT64 prefix를 붙여 **합성 AAAA 레코드** 를 만들어 준다.

#### 합성 AAAA 수식 예

NAT64 prefix가 `/96` (예: `64:ff9b::/96`) 라고 할 때,

$$
\text{Synthesized IPv6} = \text{NAT64\_prefix}_{96\text{bits}} \parallel \text{IPv4\_addr}_{32\text{bits}}
$$

예를 들어:

- 원래 서버 IPv4: `203.0.113.10`
- 10진수 → 16진수: `CB00:710A`
- 합성 AAAA: `64:ff9b::cb00:710a`

IPv6-only 클라이언트는 `64:ff9b::cb00:710a` 로 접속하지만,
중간 NAT64 장비에서 IPv4로 변환되어 실제 서버(`203.0.113.10`)와 통신한다.

#### 시나리오 — IPv6-only 모바일 네트워크

모바일 사업자가 다음과 같은 구조를 사용한다고 하자.

```text
[단말(IPv6-only)] -- IPv6 RAN/코어 -- [NAT64/DNS64] -- [IPv4 Internet]
```

- 단말은 오직 IPv6 주소만 사용한다.
- IPv4-only 웹사이트를 접속하려 할 때:
  1. DNS64가 A 레코드를 보고 합성 AAAA를 만든다.
  2. 단말은 IPv6 주소로 연결을 시도한다.
  3. NAT64에서 IPv4로 변환해 실제 서버에 접속한다.

단말 입장에서는 “IPv6로만” 통신하고, “IPv4 호환성”은 전부 네트워크가 대신 처리한다.

### DS-Lite, LW4o6, MAP-T/E 등 (요약)

IPv6-only 코어망 위에 IPv4 서비스를 싣기 위한 다른 방식들도 있다.

- **DS-Lite (Dual-Stack Lite)**
  - 가정용 CPE(LAN쪽)는 IPv6-only
  - CPE 내부의 IPv4 트래픽은 IPv4-in-IPv6 캡슐화로 ISP의 AFTR(주소/포트 변환 라우터)까지 전달
  - AFTR에서 CGN과 비슷한 방식으로 공인 IPv4를 공유

- **Lightweight 4over6 (LW4o6)**
  - DS-Lite에서 CGN 상태 부담을 줄이기 위해
    일부 포트 매핑 정보를 CPE에 분산해서 갖도록 한 방식.

- **MAP-E / MAP-T**
  - IPv4 주소와 포트 범위를 IPv6 prefix에 매핑해
    라우팅 기반으로 IPv4 트래픽을 분배하는 방식.

공통점은 **“IPv4는 점점 엣지/경계로 밀어내고, 코어는 IPv6-only”** 로 설계한다는 점이다.

### 464XLAT — NAT64를 보완하는 단말 측 변환

모바일 네트워크에서 자주 쓰이는 기법으로, 다음 두 요소를 갖는다.

- **CLAT (Customer-side Translator)**: 단말 내부에서 IPv4 앱 트래픽을 IPv6로 변환
- **PLAT (Provider-side Translator)**: 네트워크 측 NAT64

구조:

```text
[IPv4-only App] --(CLAT)--> IPv6-only 네트워크 --(PLAT/NAT64)--> IPv4 서버
```

덕분에 단말 내부 앱은 여전히 IPv4 소켓만 사용해도 되고,
네트워크는 IPv6 중심 구조를 유지할 수 있다.

---

## Use of IP Address

전환 과정에서 IP 주소의 사용 방식은 단순히 “32비트 → 128비트” 변화가 아니라,
**주소를 설계하고, 배포하고, 선택하는 방법 전체가 바뀌는 과정**이다.

여기서는 다음 네 가지 관점을 중심으로 본다.

1. Dual Stack 환경에서 주소 사용
2. 전환용 특별 IPv6 주소들
3. IPv4 사설주소/CGNAT와 IPv6 전역 주소의 역할 분담
4. 주소 계획 및 재번호(renumbering)

---

## Dual Stack 환경에서 주소 사용

### 호스트 한 대가 여러 주소를 가지는 구조

Dual-stack 호스트는 일반적으로 다음과 같이 여러 주소를 가진다.

- IPv4
  - 공인: `198.51.100.10/24`
  - 또는 사설: `10.0.1.10/24` (NAT 뒤)

- IPv6
  - 링크-로컬: `fe80::a00:27ff:fe12:3456/64`
  - 전역 유니캐스트: `2001:db8:10:1::10/64`
  - 임시(privacy) 주소: `2001:db8:10:1:abcd::1234/64` 등

OS는 목적지와 자신의 주소 리스트를 놓고
**어떤 소스·목적지 조합이 가장 적절한지**를 정책적으로 선택한다.

이를 수식으로 표현하면:

$$
(\text{src}, \text{dst})_{\text{best}} = \arg\max f(\text{src}_i, \text{dst}_j)
$$

여기서 $f$ 는 우선순위, 프리픽스 일치 길이, 스코프, 정책 테이블 등을 반영한 점수 함수다.

### 예제 — 같은 도메인, 서로 다른 주소 패밀리

DNS 레코드:

```text
www.example.net. IN AAAA 2001:db8:50::20
www.example.net. IN A     203.0.113.20
```

클라이언트 인터페이스:

- IPv4: `192.0.2.50/24`
- IPv6: `2001:db8:10:1::50/64`

라우터가 IPv6 경로를 광고하고 있고, 클라이언트는 IPv6 연결이 된다면 IPv6를 우선 사용한다.

접속 흐름:

1. 브라우저가 `www.example.net` 에 대한 DNS 질의
2. AAAA와 A 모두 응답으로 받음
3. 정책상 IPv6 우선 → 목적지 `2001:db8:50::20` 선택
4. 소스는 같은 스코프의 `2001:db8:10:1::50` 선택
5. TCP 3-way handshake는 IPv6로 진행

패킷 캡처에서는 **IPv6 헤더**만 보이지만, 같은 호스트가 동시에 IPv4 주소도 가지고 있다.

---

## IPv4와 연관된 특별 IPv6 주소들

전환기에 자주 언급되는 몇 가지 IPv6 주소 형식이 있다.

### IPv4-mapped IPv6 주소(::ffff:a.b.c.d)

형식:

```text
::ffff:192.0.2.33
```

비트 구성:

- 상위 80비트: 0
- 그 다음 16비트: 1 (`0xffff`)
- 하위 32비트: IPv4 주소

$$
\text{IPv6} = 0^{80} \parallel 1^{16} \parallel \text{IPv4}
$$

용도:

- OS 내부에서 `AF_INET6` 소켓 하나로 IPv4와 IPv6를 모두 처리할 때 사용
- “와이어 상에서” 실제 패킷에 넣어 보내는 용도가 아니라, **내부 표현용**이다.

예제 (개념):

- `getaddrinfo()` 로 IPv4 주소만 받아도,
  IPv6 소켓 구조체 내부에서는 `::ffff:a.b.c.d` 로 나타날 수 있다.

### — 역사적 개념

초기에는 `::192.0.2.33`처럼 IPv4를 바로 뒤에 붙이는 **IPv4-compatible IPv6** 도 정의되어 있었다.
하지만 혼란과 실제 사용 방식 변화 때문에 오늘날에는 **사용하지 않는 것이 원칙**이다.

교과서에는 여전히 등장하지만, 실제 설계에서는 **IPv4-mapped 주소, NAT64 prefix 등**으로 완전히 대체되었다고 봐도 좋다.

### 6to4 주소 (2002::/16)

6to4에서 IPv6 prefix는 다음처럼 만든다.

- 공인 IPv4: `192.0.2.4` → 16진: `C000:0204`
- 6to4 prefix:

```text
2002:C000:0204::/48
```

이 prefix를 로컬 LAN에 광고해서, LAN 호스트들이 이를 기반으로 IPv6 주소를 생성하게 했다.
그러나 현재는 운영 난이도와 보안 문제 때문에 **새 설계에서 사용하지 않는 것이 일반적**이다.

### NAT64 Prefix (예: 64:ff9b::/96)

NAT64에서는 IPv6 prefix와 IPv4 주소를 96비트 + 32비트로 이어 붙인다.

$$
\text{IPv6\_synth} = \text{NAT64\_prefix} \parallel \text{IPv4\_addr}
$$

예:

- NAT64 prefix: `64:ff9b::/96`
- IPv4: `203.0.113.10` (`CB00:710A`)

합성 IPv6: `64:ff9b::cb00:710a`

이 주소는 “IPv6-only 클라이언트의 관점에서 IPv4 서버를 가리키는 주소”가 된다.

---

## IPv4 사설주소/CGNAT와 IPv6 전역 주소의 역할 분담

현실적인 네트워크에서는 다음과 같은 역할 분담이 많이 보인다.

### 가정용 CPE 예

가정용 공유기(CPE)를 예로 들면:

- WAN(ISP 방향)
  - IPv4: `100.64.1.10/24` (CGN 주소 공간)
  - IPv6 prefix: `2001:db8:abcd:100::/56` (고객에게 할당된 전역 prefix)

- LAN(내부)
  - IPv4: `192.168.0.0/24`
  - IPv6: `2001:db8:abcd:100::/64`, `2001:db8:abcd:101::/64` …

역할:

- IPv4는 대부분 **다중 NAT 구조**:
  - 내부 호스트 → 가정용 NAT → ISP CGN → 공인 IPv4
- IPv6는 **엔드-투-엔드 전역 유니캐스트**:
  - 내부 호스트가 자신의 전역 IPv6 주소로 인터넷에 직접 연결

전환의 흐름은 보통 이렇다.

1. 신규 서비스는 가능한 한 **IPv6를 우선 지원**
2. IPv4-only 서비스는 NAT/CGN 뒤로 몰려 들어간다.
3. 점차 IPv4는 “레거시 호환 레이어”, IPv6는 “정상적인 엔드-투-엔드 네트워크” 역할을 담당.

---

## 주소 계획과 재번호(Renumbering)

IPv6는 주소가 넉넉하다고 해서 “아무렇게나 써도 된다”는 의미는 아니다.
전환 과정에서 특히 **주소 계획(Address Planning)** 과 **재번호(renumbering)** 전략이 중요해진다.

### 예 — 사이트 /48, 서브넷 /64

한 엔터프라이즈 네트워크 예:

- ISP가 사이트에 `/48` prefix 제공: `2001:db8:100::/48`

사이트 내부:

| 영역           | IPv6 prefix                       |
|----------------|-----------------------------------|
| DMZ            | `2001:db8:100:0001::/64`         |
| 서버 존        | `2001:db8:100:0002::/64`         |
| 사무실 네트워크 | `2001:db8:100:0010::/64`         |
| Wi-Fi          | `2001:db8:100:0100::/64`         |

IPv4처럼 주소 부족에 쫓기지 않고, **기능·보안영역 단위로 넉넉한 /64** 를 할당할 수 있다.

### — IPv4보다 쉬워야 한다

IPv4에서 주소를 재번호 하는 것은 굉장히 고통스러운 작업이었다.

- 코드/스크립트/설정 파일 곳곳에 박혀 있는 IPv4 literal
- 수많은 정적 ACL, NAT 규칙
- 하드코딩된 IP를 일일이 찾아 교체

IPv6 전환 전략은 다음과 같은 메커니즘을 활용해 **재번호를 더 수월하게** 만드는 것을 목표로 한다.

- DNS 이름 사용 (애플리케이션은 IP literal 대신 이름을 사용)
- SLAAC, DHCPv6, RA로 호스트 주소 자동 설정
- Prefix 변경 시 RA/DHCPv6 정보만 바꾸면 호스트가 새 주소를 자동으로 받도록 설계

예:

1. ISP prefix가 `2001:db8:100::/48` → `2001:db8:200::/48` 으로 변경
2. 경계 라우터에서 새 prefix로 RA/DHCPv6 설정 변경
3. 호스트는 자동으로 새 주소를 받고, 일정 시간 동안 옛 주소와 새 주소를 같이 갖다가, 옛 주소를 폐기
4. 애플리케이션은 IP literal 대신 DNS 이름을 사용했으므로, 별도 수정 없이 동작

---

## 종합 실습 예 — Dual Stack + NAT64

마지막으로, 작은 랩 환경에서 전환 전략을 실제로 실험하는 구성을 상상해보자.

### 구성

```text
[IPv6-only Client] -- [Dual-stack router + NAT64/DNS64] -- [IPv4-only Web Server]
```

- 클라이언트: `2001:db8:10:1::100/64`
- NAT64 prefix: `64:ff9b::/96`
- 서버: `198.51.100.20`

### 단계별 동작

1. **라우터(Dual Stack) 설정**
   - IPv4: `203.0.113.1` (서버/인터넷 쪽)
   - IPv6: `2001:db8:10:1::1` (클라이언트 쪽)
   - NAT64: prefix `64:ff9b::/96`, IPv4 풀 `203.0.113.0/24`

2. **DNS64 동작**
   - 클라이언트가 `www.v4only.example` 에 대한 AAAA 질의
   - DNS64가 A 레코드 `198.51.100.20`을 보고, `64:ff9b::c633:6414` 같은 합성 AAAA 생성

3. **클라이언트 접속**
   - 클라이언트는 `64:ff9b::c633:6414` 로 TCP 연결
   - NAT64는 이 패킷을 받아 IPv4 헤더를 생성하고, 서버 `198.51.100.20`으로 전달

4. **서버 응답**
   - 서버는 IPv4로 응답
   - NAT64는 응답을 다시 IPv6로 변환해 클라이언트에게 전달

이 실습을 통해, **Dual Stack(라우터) + Tunneling/Translation(NAT64/DNS64)** 전략이 실제로 어떻게 조합되어 동작하는지 체감할 수 있다.

---

## 요약

- **Strategies**
  - Dual Stack: IPv4/IPv6를 나란히 운용
  - Tunneling: IPv6를 IPv4 위에(또는 그 반대로) 캡슐화
  - Translation: NAT64/DS-Lite/464XLAT 등으로 IPv4↔IPv6 헤더 변환
- **Use of IP Address**
  - Dual-stack 노드는 여러 IPv4/IPv6 주소를 동시에 갖고, OS가 정책에 따라 적절한 소스/목적지를 선택한다.
  - IPv4-mapped 주소, NAT64 prefix, (역사적으로) 6to4 주소 같은 특별한 IPv6 주소가 전환 과정에 쓰인다.
  - IPv4는 점점 사설주소/CGNAT 영역으로 밀리고, IPv6 전역 주소가 “정상적인 엔드-투-엔드” 주소 역할을 맡는다.
  - 주소 계획과 재번호 전략이 중요해지며, DNS·SLAAC·DHCPv6·RA 등을 활용해 IPv4보다 더 쉽게 재번호할 수 있도록 설계한다.

이 관점을 잡고 전체 IPv6/라우팅/보안 챕터를 다시 보면,
각 기술이 “전환기의 어떤 문제를 해결하려고 나왔는지”를 훨씬 명확하게 이해할 수 있을 것이다.
