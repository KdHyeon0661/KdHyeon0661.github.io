---
layout: post
title: 데이터 통신 - Internet Security (4)
date: 2024-10-01 23:20:23 +0900
category: DataCommunication
---
# 32.4 Firewalls — Packet Filter Firewall와 Proxy Firewall

이 절에서는 인터넷 보안 전체 흐름(암호화·IPsec·TLS 등) 속에서 **네트워크 경계에 서 있는 방어자**, 즉 방화벽을 두 가지 고전적 구조를 중심으로 정리한다.

- **Packet Filter Firewall**: IP·포트·프로토콜 수준에서 동작하는 전통적인 필터(라우터 ACL, 클라우드 보안 그룹 등).
- **Proxy Firewall**(Application-Proxy Gateway): 애플리케이션 계층까지 이해하고, 클라이언트 대신 서버와 통신하는 **중개자** 형태의 방화벽.:contentReference[oaicite:0]{index=0}  

실제 기업 환경에서는 이 둘이 단독이 아니라, **상호 보완적으로** 쓰이거나, 현대적인 **Stateful / NGFW** 안에 통합된 형태로 등장한다.

---

## Packet Filter Firewall

### 1. 개념과 정의

**Packet Filter Firewall**은 네트워크 계층(IP)과 전송 계층(TCP/UDP) 정보만 보고, 각 패킷을 **독립적으로 검사**해서 통과(permit) 또는 차단(deny)하는 방화벽이다. 전통적으로는 **라우터에 적용된 ACL(Access Control List)** 이 대표적인 구현이고, 요즘은 클라우드의 보안 그룹(Security Group) 역시 대부분 **패킷 필터 방화벽**이다.:contentReference[oaicite:2]{index=2}  

핵심 특징:

- 검사 기준
  - 소스/목적지 IP 주소
  - 소스/목적지 포트 번호
  - 프로토콜(TCP, UDP, ICMP 등)
  - 방향(Inbound/Outbound)
- 전통적 구현은 **Stateless(무상태)**: 각 패킷을 이전 패킷과 무관하게 독립적으로 판정.:contentReference[oaicite:3]{index=3}  
- 현대 구현은 대부분 **Stateful Inspection** 기능까지 포함해, 연결 상태를 기억하는 동적 패킷 필터로 발전했다.:contentReference[oaicite:4]{index=4}  

NIST SP 800-41(방화벽 가이드)은 방화벽 플랫폼을 설명할 때 **packet filter firewalls**, **stateful inspection firewalls**, **application-proxy gateway firewalls** 등을 구분해 소개한다.:contentReference[oaicite:5]{index=5}  

---

### 2. 동작 원리 – 5-튜플 매칭

패킷 필터는 주로 **IP 5-튜플** 기반으로 정책을 적용한다.

- $$\text{5-tuple} = (\text{src-ip}, \text{src-port}, \text{dst-ip}, \text{dst-port}, \text{protocol})$$

각 규칙(rule)은 다음 형식으로 생각할 수 있다.

- `if (조건) then permit else deny`
  - 조건: 5-튜플 + 방향 + 인터페이스 등

#### 예제 1: 단순 ACL 논리 (의사코드)

```pseudo
rules = [
  { action: PERMIT, src: 10.0.0.0/24, dst_port: 80, proto: TCP, direction: OUT },
  { action: PERMIT, src: 10.0.0.0/24, dst_port: 443, proto: TCP, direction: OUT },
  { action: PERMIT, src: ANY,          dst: 10.0.0.10, dst_port: 22, proto: TCP, direction: IN },
  { action: DENY,   src: ANY,          dst: ANY,                       direction: ANY }
]

function filter(packet):
    for rule in rules:
        if rule.matches(packet):
            return rule.action
    return DENY   # default deny
```

#### 예제 2: 소규모 회사 인터넷 경계 라우터

**상황**

- 내부망: `10.0.0.0/24`
- 경계 라우터가 ISP에 직접 연결.
- 요구 사항:
  - 내부 → 인터넷: HTTP(80), HTTPS(443), DNS(53/UDP) 허용.
  - 인터넷 → 내부: 원칙적 차단, 단 **관리자가 SSH(22/TCP)** 로 특정 서버(10.0.0.10)에 접근하도록 허용.

**규칙 표(단순화)**

| 우선순위 | 방향 | Src IP         | Dst IP        | Proto | Dst Port | 동작   |
|----------|------|----------------|---------------|-------|----------|--------|
| 1        | OUT  | 10.0.0.0/24    | ANY           | TCP   | 80       | Permit |
| 2        | OUT  | 10.0.0.0/24    | ANY           | TCP   | 443      | Permit |
| 3        | OUT  | 10.0.0.0/24    | ANY           | UDP   | 53       | Permit |
| 4        | IN   | 관리자 공인IP  | 10.0.0.10     | TCP   | 22       | Permit |
| 5        | ANY  | ANY            | ANY           | ANY   | ANY      | Deny   |

**Cisco 라우터 ACL 스타일 예시**

```text
! Outbound (inside -> outside)
access-list OUTBOUND permit tcp 10.0.0.0 0.0.0.255 any eq 80
access-list OUTBOUND permit tcp 10.0.0.0 0.0.0.255 any eq 443
access-list OUTBOUND permit udp 10.0.0.0 0.0.0.255 any eq 53
access-list OUTBOUND deny   ip any any

! Inbound (outside -> inside)
access-list INBOUND permit tcp host 203.0.113.10 host 10.0.0.10 eq 22
access-list INBOUND deny   ip any any
```

여기서는 **stateless**하게 동작한다고 가정했기 때문에, 응답 트래픽이 들어올 수 있도록 INBOUND에 **특별히 예외 규칙**을 둬야 한다. 실제 운영에서는 대부분 **stateful** 기능을 써서, 내부에서 나간 세션에 대한 응답은 자동 허용되도록 한다.:contentReference[oaicite:6]{index=6}  

---

### 3. 장점과 단점

#### 3.1 장점

1. **성능**
   - 패킷 헤더 일부(IP/포트/프로토콜)만 보기 때문에 매우 빠르며, 고속 라우터에서도 구현하기 쉽다.:contentReference[oaicite:7]{index=7}  
2. **단순성**
   - 규칙이 비교적 간단해서 이해와 검증이 쉽다.
   - 라우터, 클라우드 보안 그룹 등 다양한 플랫폼에 공통 개념으로 존재.
3. **폭넓은 적용 범위**
   - WAN 경계, 데이터센터 코어, 클라우드 VPC/Subnet, 고객사 전용선 입구 등 여러 위치에서 기본 보안 필터로 활용.

#### 3.2 단점

1. **애플리케이션 가시성 부족**
   - URL, HTTP 메소드, SMTP 명령, SQL 문 등 **애플리케이션 계층**의 정보는 볼 수 없다.
   - 단순히 “TCP 443이면 HTTPS일 것이다” 정도의 추정만 가능.
2. **Stateless 구현의 한계**
   - 단순 구현에서는 세션 상태를 기억하지 않기 때문에, **응답 패킷을 허용하기 위한 추가 규칙**이 필요하고, 일부 공격(예: 세션 하이재킹, 포트 스캐닝)에 취약할 수 있다.:contentReference[oaicite:8]{index=8}  
3. **정교한 공격 탐지는 어려움**
   - 패킷 헤더만 보고는 애플리케이션 레벨의 취약점 공격(SQLi, XSS, RCE 등)을 식별하기 어렵다.
   - 이 때문에 **IDS/IPS, WAF, NGFW** 등의 보완 기술이 함께 사용된다.:contentReference[oaicite:9]{index=9}  

---

### 4. 현대 네트워크에서 Packet Filter의 위치

현대 설계에서 Packet Filter 역할은 다음과 같이 분산된다.

- **경계 라우터 ACL**: 가장 바깥쪽에서 “큰 칼질” — 사설 IP 차단, RFC 1918 Source Drop, Bogon Filter 등.
- **클라우드 보안 그룹·네트워크 ACL**
  - AWS Security Group / NACL, Azure NSG 등은 전형적인 **stateful / stateless 패킷 필터**로 구현된다.
- **내부 세그멘테이션**
  - 서버 간 동서(East-West) 트래픽을 제어하기 위해, L3 스위치·라우터에 세밀한 ACL을 적용.

실제 기업 환경에서는 여기에 **Stateful Inspection 기능이 기본 포함**되고, 상위 계층의 프록시/NGFW를 덧붙이는 것이 일반적이다.:contentReference[oaicite:10]{index=10}  

---

## Proxy Firewall

### 1. 개념과 정의

**Proxy Firewall**(Application-Proxy Gateway)은 클라이언트와 서버 사이에 **중개자(proxy)** 를 두고, 이 프록시가 애플리케이션 계층까지 분석·검증한 뒤 트래픽을 전달하는 방화벽이다. NIST는 이를 “하위 계층 접근 제어와 상위 계층 기능을 결합하고, 통신하려는 두 호스트 사이에서 **중개자 역할을 하는 프록시 에이전트를 포함하는 방화벽 능력**”으로 정의한다.:contentReference[oaicite:11]{index=11}  

핵심 특징:

- **클라이언트 ↔ 프록시 ↔ 서버** 두 개의 논리적 연결
- 프로토콜별(HTTP, SMTP, FTP 등) **전용 프록시**를 사용
- 애플리케이션 레벨 정보(URI, 메소드, 헤더, 커맨드 등)를 기반으로 정책 적용
- 인증·로깅·콘텐츠 필터링을 정교하게 수행

---

### 2. 동작 흐름 (그림)

**HTTP Proxy Firewall** 예를 간단한 ASCII 그림으로 표현하면 다음과 같다.

```text
[내부 클라이언트]         [Proxy Firewall]               [외부 웹 서버]
      |                          |                                |
      | 1. HTTP 요청 (GET /...)  |                                |
      |------------------------->|                                |
      |                          | 2. 요청 검사(정책/URL/헤더)    |
      |                          | 3. 새 TCP 연결 생성            |
      |                          |------------------------------->|
      |                          |   4. 서버로 HTTP 요청 전달     |
      |                          |<-------------------------------|
      |                          | 5. 응답 검사(악성 콘텐츠 등)   |
      |<-------------------------|    6. 클라이언트에게 응답 전달 |
```

- **클라이언트 입장**: 항상 프록시와만 통신.
- **서버 입장**: 실제 요청자는 프록시(프록시의 IP/포트)를 클라이언트로 인식.
- 방화벽은 두 방향의 HTTP 메시지를 모두 파싱하여 **정책 적용·로깅·변환** 가능.

---

### 3. 예제 – 기업 HTTP Proxy Firewall 정책

**상황**

- 회사는 **업무 중 유튜브·게임 사이트 차단**, 악성 사이트 접근 차단, 사용자별 로그 기록이 필요.
- 모든 사용자 브라우저는 HTTP/HTTPS 프록시로 `proxy.corp.local:3128` 을 사용.

**정책 요구사항**

1. 사내 AD 계정으로 프록시에 인증해야 인터넷 사용 가능.
2. 카테고리 기반으로 **Entertainment(게임, 스트리밍)** 사이트 차단.
3. 알려진 악성 도메인/URL 차단.
4. 로그에 `사용자ID, 목적지URL, 카테고리, 허용/차단 결과` 기록.

**프록시 정책(개념적)**

```pseudo
on HTTP_REQUEST(request, user):
    if not user.authenticated:
        deny("AUTH_REQUIRED")

    category = classify_url(request.url)  # URL 카테고리 DB/서비스 사용
    if category in ["GAMBLING", "ADULT", "GAMES", "VIDEO_STREAMING"]:
        log(user, request.url, category, "BLOCK")
        deny("CATEGORY_BLOCKED")

    if is_malicious_url(request.url):
        log(user, request.url, "MALICIOUS", "BLOCK")
        deny("SECURITY_BLOCKED")

    log(user, request.url, category, "ALLOW")
    forward_to_server(request)
```

현실에서는 **전문 URL 필터링 엔진**과 **위협 인텔리전스 피드**를 연동해 카테고리·위험도 판정을 자동화한다. 이러한 기능이 발전한 형태가 오늘날의 **Secure Web Gateway(SWG)**, **Next-Generation Firewall(NGFW)** 에 통합되어 있다.:contentReference[oaicite:12]{index=12}  

---

### 4. Proxy Firewall의 장점과 단점

#### 4.1 장점

1. **풍부한 애플리케이션 계층 정보**
   - HTTP URI, 메소드, 헤더, 쿠키, JSON Body 등까지 해석 가능.
   - SMTP 명령, 수신자, 첨부 파일 등 애플리케이션 프로토콜 내용 기반 필터링 가능.:contentReference[oaicite:13]{index=13}  
2. **강력한 인증·로깅**
   - 사용자 계정 단위 인증(AD/LDAP 통합), 사용자·그룹별 정책 적용 가능.
   - “어느 사용자”가 “어느 사이트”에 “언제” 접속했는지를 상세하게 기록.
3. **콘텐츠 필터링 및 DLP와 결합 용이**
   - 파일 확장자·MIME 타입 제한, 악성 코드 탐지(안티바이러스), 민감정보 유출 탐지(DLP) 등.
4. **주소 스푸핑 공격에 상대적으로 강함**
   - 실제 서버와 분리된 **게이트웨이** 형태이므로, 네트워크 레벨의 스푸핑에 덜 직접 노출된다.:contentReference[oaicite:14]{index=14}  

#### 4.2 단점

1. **성능 오버헤드**
   - 패킷을 그대로 전달하는 것이 아니라, **애플리케이션 메시지를 파싱하고 재작성**해야 하므로 CPU·메모리·지연 시간이 증가.
2. **프로토콜 종속성**
   - 각 애플리케이션 프로토콜(HTTP, SMTP, FTP 등)마다 별도 프록시 로직이 필요.
   - 새로운 프로토콜·커스텀 프로토콜에 대한 지원이 어려울 수 있다.
3. **암호화(TLS) 환경에서의 한계**
   - HTTPS는 **종단 간 암호화** 때문에, 프록시가 내용(URI, 헤더)을 보려면 **TLS 중간자(MITM) 방식**을 써야 한다.
   - 이를 위해 조직 내에서 **사설 CA 인증서 배포, TLS 재암호화** 등을 해야 하는데, 프라이버시·보안 정책 설계가 까다롭다.
4. **복잡한 설계 및 운영**
   - 클라이언트 프록시 설정(직접 설정 or WPAD or 투명 프록시), 인증 연동, 로그 분석 시스템 연계 등 고려 요소가 많다.

---

### 5. Forward Proxy vs Reverse Proxy Firewall

Proxy Firewall은 크게 두 가지 배치 형태로 생각할 수 있다.

1. **Forward Proxy** (클라이언트 앞쪽)
   - 내부 클라이언트 → 인터넷으로 나가는 트래픽 중개.
   - URL 필터링, 사용자 인증, DLP, 악성 사이트 차단 등에 적합.
2. **Reverse Proxy**
   - 인터넷 사용자 → 내부 웹 애플리케이션으로 들어오는 트래픽 중개.
   - 웹 애플리케이션 방화벽(WAF), DDoS 방어, SSL 오프로딩, 인증 연계에 활용.

현대의 **Application Firewall / Web Application Firewall (WAF)** 는 대부분 **Reverse Proxy 형태**로 동작하며, OWASP Top 10 공격(SQLi, XSS, CSRF 등)을 탐지·차단한다.:contentReference[oaicite:15]{index=15}  

---

## Packet Filter vs Proxy Firewall, 그리고 Stateful / NGFW까지

실제 설계에서는 Packet Filter와 Proxy Firewall을 **대체 관계**가 아니라, **계층화된 보안(Defense in Depth)** 의 일부로 본다.

### 1. 비교 표

| 구분 | Packet Filter Firewall | Proxy Firewall (Application-Proxy) |
|------|------------------------|-------------------------------------|
| 주요 레벨 | L3/L4 (IP·포트·프로토콜) | L7 (HTTP, SMTP, FTP 등) |
| 상태 추적 | 전통적: Stateless, 현대: Stateful | 논리적 연결 2개(C↔P, P↔S) 모두 관리 |
| 정책 기준 | IP, 포트, 프로토콜, 방향, 인터페이스 | URL, 메소드, 헤더, 사용자, 콘텐츠 |
| 성능 | 고성능, 저지연 | 비교적 느림, CPU·메모리 요구 높음 |
| 구현 위치 | 라우터 ACL, 클라우드 SG/NACL, L3 스위치 | 전용 프록시 장비/소프트웨어, SWG, WAF |
| 대표 용도 | 기본 접근제어, 세그멘테이션, DDoS 1차 완화 | 웹 접근제어, 악성 사이트 차단, DLP, WAF |

현대의 **Next-Generation Firewall(NGFW)** 는 Gartner가 정의한 바와 같이 “포트/프로토콜 검사뿐 아니라 애플리케이션 레벨 검사, 침입 방지(IPS), 외부 인텔리전스 연계를 포함하는 **Deep Packet Inspection 방화벽**” 이며, 사실상 **패킷 필터 + Stateful Inspection + 애플리케이션 프록시 기능**을 통합한 형태라고 볼 수 있다.:contentReference[oaicite:16]{index=16}  

---

## 설계 예제 1 – 3계층 웹 애플리케이션

**상황**

- DMZ에 웹 서버, 내부망에 API 서버와 DB 서버가 있는 3계층 구조.
- 외부 사용자는 HTTPS(443)으로 웹 애플리케이션 접속.
- 내부 사용자는 프록시를 통해 인터넷 브라우징.

**구성**

1. **경계 라우터 (Packet Filter)**
   - 인터넷 ↔ DMZ 사이에서 최소한의 ACL 적용.
   - 예: `ANY -> DMZ-Web:443` 허용, 나머지 차단.
2. **DMZ Firewall (Stateful Packet Filter + WAF/Proxy)**
   - Reverse Proxy / WAF가 웹 서버 앞에 위치.
   - HTTP 요청을 파싱하여 악성 패턴 필터링, TLS 종료.
3. **내부 Firewall (Packet Filter)**
   - Web ↔ API, API ↔ DB 트래픽을 포트·IP 기준으로 최소화.
4. **내부 Secure Web Proxy**
   - 업무용 인터넷 접근 정책 적용(카테고리 차단, DLP).

이처럼 Packet Filter는 **레이어 3/4에서 “누가 누구와 연결 가능한가”** 를 정의하고, Proxy Firewall은 **레이어 7에서 “무엇을 주고받을 수 있는가”** 를 정밀하게 제어한다.

---

## 설계 예제 2 – 클라우드 환경

**상황**

- AWS VPC에 Web, App, DB 서브넷이 각각 존재.
- 인터넷 출구에 NAT Gateway와 **클라우드 기반 프록시 서비스(SWG)** 사용.

**구성 포인트**

- **Security Group / NACL**: 전형적인 **stateful/stateless packet filter firewall** 역할을 수행.
  - 예: Web SG는 `0.0.0.0/0 -> 443/TCP` 만 허용.
  - App SG는 `Web-SG -> 8080/TCP`, DB SG는 `App-SG -> 5432/TCP` 만 허용.
- **클라우드 프록시**: SWG가 HTTP/HTTPS 트래픽을 프록시하며, 카테고리 필터링·AV·DLP 적용.
- 클라이언트는 로컬 PAC/WPAD를 통해 프록시 경유.

이 구조에서도 **패킷 필터**와 **프록시**가 각자의 레벨에서 역할을 분담한다.

---

## 요약

- **Packet Filter Firewall**
  - IP/포트/프로토콜 기반의 L3/L4 필터링.
  - 전통적으로 Stateless, 현대에는 Stateful Inspection을 포함하는 경우가 일반적.
  - 고성능·저지연이 장점이지만, 애플리케이션 수준 가시성이 부족하다.
- **Proxy Firewall**
  - 클라이언트와 서버 사이에서 애플리케이션 레벨까지 분석하는 중개자.
  - HTTP, SMTP 등 프로토콜 내용을 기준으로 인증, 로깅, 콘텐츠 필터링, DLP 등을 수행.
  - 강력한 보안을 제공하지만, 오버헤드와 복잡성이 크다.
- 실제 설계에서는 두 방식을 **배타적 선택**이 아니라 **계층적 조합**으로 사용하며, 그 발전형이 **NGFW, SWG, WAF** 등으로 구현되고 있다.
