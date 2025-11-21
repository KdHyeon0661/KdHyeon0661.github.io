---
layout: post
title: 데이터 통신 - Standard Client-Server Protocols (5)
date: 2024-09-09 22:20:23 +0900
category: DataCommunication
---
# Domain Name System (DNS)

DNS는 **도메인 이름 ↔ IP 주소**를 매핑하는, 인터넷 전체가 공유하는 **분산 데이터베이스 + 프로토콜**이다.
이 절에서는 교과서적 구조(네임스페이스, RR, 메시지 형식 등)에 더해, **오늘날 운영 환경(루트 존 규모, 동적 업데이트, DNS 보안)**까지 포함해 정리한다.

---

## Name Space

### 계층적 네임스페이스 개념

DNS 이름 공간은 **역트리(inverted tree)** 구조를 갖는다. 가장 위는 **루트(root, “.”)** 이고, 그 아래에 `.com`, `.org`, `.de` 같은 **TLD(Top-Level Domain)** 가 있으며, 그 아래에 `example.com`, `mit.edu` 같은 **2차 도메인**, 그 아래에는 `www.example.com` 같은 **호스트/서비스 이름**이 온다. 이 구조와 RR 형식은 RFC 1034/1035에 정의되어 있다.

예: `www.cs.mit.edu.`

| 레이블 | 의미 |
|--------|------|
| `.` | 루트 도메인 |
| `edu` | TLD (교육 기관용 gTLD) |
| `mit` | 2차 도메인 (MIT) |
| `cs` | 서브도메인 (컴퓨터 과학 학과 등) |
| `www` | 호스트/서비스 이름 |

FQDN(Fully Qualified Domain Name)은 마지막에 점(.)이 붙은 **절대 이름**을 뜻한다.
일반적으로 클라이언트는 `www.example.com`처럼 마지막 점을 생략하지만, 내부적으로는 `www.example.com.`으로 취급된다.

### 도메인 vs 존(Zone)

- **도메인(domain)**: 네임스페이스 상의 논리적 서브트리
- **존(zone)**: 특정 네임 서버 집합이 **권한(authority)** 을 갖는, 한 도메인(또는 그 일부)의 데이터 집합

예:

- `example.com` 도메인을 통째로 하나의 존으로 운영하면:
  - SOA, NS, A/AAAA, MX, TXT 등 모든 레코드가 `example.com` 존 파일에 있음
- 대규모 조직에서 `corp.example.com`과 `prod.example.com`을 별도 팀이 운영하고 싶다면:
  - `example.com` 존에서 두 서브도메인에 대해 NS를 위임
  - `corp.example.com` 과 `prod.example.com`은 각각 **독립 존**이 됨

---

## DNS in the Internet

### 계층과 역할

인터넷 전체 관점에서 DNS는 대략 네 가지 역할로 나뉜다.

1. **루트 서버(root servers)**
   - 루트 존(`.`)에 대한 권한 보유
   - 각 TLD(`.com`, `.org`, `.fr`, ...)에 대한 NS 레코드 제공
   - 실제 물리 서버는 전 세계에 분산되고, 12개 독립 조직이 루트 서버 인스턴스를 운영한다.

2. **TLD 서버(TLD name servers)**
   - `.com`, `.net`, `.org`, `.de` 등 각 TLD 존에 대한 권한
   - 2차 도메인의 NS 레코드를 제공(예: `example.com`에 대한 NS)

3. **권한(authoritative) 네임 서버**
   - `example.com`, `university.edu` 같은 개별 도메인/존을 실제로 관리
   - A/AAAA, MX, TXT, CNAME, SOA 등의 레코드 저장
   - 조직(기업, 대학 등)이나 호스팅 업체가 운영

4. **재귀(Recursive) 리졸버**
   - **클라이언트를 대신해 전체 질의 체인을 수행**
   - ISP, 기업, 공용 리졸버(예: 9.9.9.9, 1.1.1.1 등)가 제공하는 “DNS 서버”가 여기에 해당
   - 클라이언트(Stub Resolver)는 보통 재귀 리졸버에게만 질의를 보내고, 나머지 계층은 리졸버가 처리

### 루트 존과 TLD 현황 (2024–2025 기준 개략)

- 루트 존에는 `.com`, `.org`, `.uk`, 신규 gTLD들(`.app`, `.dev`, `.bank` 등)을 포함해 **약 1450개 수준의 TLD**가 등록되어 있다(2024년 말 기준, ISC와 ICANN 자료 기준).
- 각 TLD는 **레지스트리(Registry)** 가 운영하며, 해당 TLD의 존 파일과 NS 인프라를 관리한다.

---

## Resolution (이름 해석 과정)

### Stub Resolver ↔ Recursive Resolver ↔ Authoritative

일반 PC/스마트폰에서 DNS 해석 흐름은 다음과 같다.

1. **애플리케이션**(브라우저 등)이 `getaddrinfo("www.example.com")` 호출
2. **Stub Resolver** (OS 내 라이브러리)가 로컬 캐시 확인 후, 재귀 리졸버(예: 192.0.2.53)에 질의 전송
3. 재귀 리졸버가 캐시에 없으면:
   - 루트 서버에게 `.com` NS 확인
   - `.com` TLD 서버에게 `example.com` NS 확인
   - `example.com` 권한 서버에 `www.example.com`에 대한 A/AAAA를 질의
4. 재귀 리졸버는 답변을 **캐시에 저장**하고 클라이언트에게 전송

텍스트 흐름 예 (단순화):

```text
Client (Stub) → Recursive → Root → .com → ns1.example.com → Recursive → Client
```

### 재귀(Recursive) vs 반복(Iterative)

- Stub Resolver ↔ Recursive Resolver: 보통 **재귀 질의(recursive)**
  - 클라이언트는 “최종 답을 달라”고 요청
- Recursive Resolver ↔ 상위 네임 서버: 보통 **반복 질의(iterative)**
  - 루트 서버는 “내가 모르는 이름이니 이 TLD 서버에 물어봐라”라고 **참조(referral)** 를 줌

---

### 예제: `dig`로 전체 해석 경로 보기

```bash
dig +trace www.ietf.org
```

- `+trace` 옵션은 재귀 리졸버처럼 루트→TLD→권한 서버 순서로 질의를 수행하면서,
  각 단계의 응답을 보여준다.
- 출력에는:
  - 루트 NS 목록
  - `.org` TLD NS 목록
  - `ietf.org` 존의 권한 NS 목록
  - 최종 `www.ietf.org`의 A/AAAA 레코드
  가 순서대로 나타난다.

---

## Caching

### TTL(Time To Live)

각 RR에는 **TTL(Time To Live)** 값이 있고, 이는 캐시에 해당 레코드를 **얼마 동안 유지할 수 있는지**를 지정한다.

- 단위: 초(Seconds)
- 예: `3600` → 1시간 동안 유효
- 캐시는 TTL이 0이 될 때까지 재사용하고, 0이 되면 새로 질의한다.

단순하게, 요청 도착률을 $$\lambda$$ (초당 요청 수), TTL을 $$T$$라고 할 때,
Poisson 도착을 가정하면 **캐시 히트 확률**의 근사값은:

$$
P_{\text{hit}} \approx 1 - e^{-\lambda T}
$$

(요청 간 간격이 TTL보다 짧을수록 히트 확률이 높아진다는 감각적 설명용이다.)

### 네거티브 캐싱(Negative Caching)

존에는 **해당 이름이 존재하지 않음(NXDOMAIN)** 또는 **타입이 없음(NODATA)** 이라는 정보도 포함된다.

- SOA 레코드의 `MINIMUM` 필드와 별도의 네거티브 TTL 설정에 따라
  “없는 이름”에 대한 결과도 일정 시간 캐시된다.
- 효과:
  - 반복적인 오타 요청으로 인한 부하 감소
  - 공격자가 존재하지 않는 이름으로 대량 질의를 보내는 경우에도 일정 완충

### 캐시 일관성과 운영 이슈

- TTL을 너무 크게 잡으면:
  - 변경(예: IP 변경, 장애 조치)이 늦게 반영 → 운영 리스크
- TTL을 너무 작게 잡으면:
  - 재귀 리졸버와 권한 서버 트래픽 증가
- 대형 서비스(예: CDN, 대형 웹 서비스)는
  - Top-level NS, SOA에는 상대적으로 긴 TTL
  - A/AAAA(특히 로드밸런싱용)에는 짧은 TTL(수십~수백 초)
  를 혼합하여 사용한다.

---

## Resource Records (RR)

RFC 1035는 RR 포맷을 다음처럼 정의한다.

```text
NAME   TYPE   CLASS   TTL   RDLENGTH   RDATA
```

- **NAME**: 도메인 이름(FQDN 또는 상대 이름)
- **TYPE**: RR 유형(A, AAAA, MX, NS, CNAME, TXT, …)
- **CLASS**: 주로 IN(Internet)
- **TTL**: 캐시 유효 시간(초)
- **RDLENGTH**: RDATA 길이(바이트)
- **RDATA**: 타입별 실제 데이터(IP, 텍스트, 우선순위 등)

### 대표 RR 타입과 예

| 타입 | 용도 | 예시 |
|------|------|------|
| A | IPv4 주소 | `www.example.com. IN A 93.184.216.34` |
| AAAA | IPv6 주소 | `www.example.com. IN AAAA 2606:2800:220:1:248:1893:25c8:1946` |
| NS | 존에 대한 권한 네임 서버 | `example.com. IN NS ns1.example.net.` |
| SOA | Start of Authority, 존 메타데이터 | `example.com. IN SOA ns1.example.net. hostmaster.example.com. ...` |
| MX | 메일 서버 | `example.com. IN MX 10 mail.example.com.` |
| CNAME | 별칭(alias) | `www.example.net. IN CNAME www.example.com.` |
| TXT | 임의 텍스트, SPF/DKIM/정책 등 | `example.com. IN TXT "v=spf1 include:_spf.example.net ~all"` |
| SRV | 서비스 위치 | `_sip._tcp.example.com. IN SRV 10 60 5060 sip1.example.com.` |
| PTR | 역방향 조회 | `34.216.184.93.in-addr.arpa. IN PTR www.example.com.` |
| DNSKEY/DS/RRSIG/NSEC… | DNSSEC 관련 | 루트/상위 존과 연계된 서명 체인 |

#### 예: 간단한 존 파일 스니펫

```text
$TTL 3600
@   IN SOA ns1.example.com. hostmaster.example.com. (
        2025111701 ; serial
        3600       ; refresh
        900        ; retry
        604800     ; expire
        86400      ; minimum
)
    IN NS  ns1.example.com.
    IN NS  ns2.example.com.

www     IN A     203.0.113.10
api     IN A     203.0.113.11
mail    IN A     203.0.113.20
        IN MX 10 mail.example.com.
```

---

## DNS Messages

DNS 메시지 포맷은 RFC 1035의 **고정 12바이트 헤더 + 최대 4개 섹션** 구조를 따른다.
주요 운영체제 문서(예: Windows Server DNS 문서)에서도 동일 구조를 설명한다.

### 전반적인 구조

```text
+---------------------+
|        Header       | 12 bytes
+---------------------+
|       Question      | variable
+---------------------+
|        Answer       | variable
+---------------------+
|      Authority      | variable
+---------------------+
|      Additional     | variable
+---------------------+
```

헤더에는 **ID, 플래그, 섹션별 RR 개수**가 들어 있고,
각 섹션은 RR 형식(Question만 TYPE/CLASS까지)으로 구성된다.

### 헤더 필드

헤더(12 bytes)는 다음 비트 필드를 가진다.

- **ID (16비트)**: 질의/응답 매칭용 식별자
- **Flags (16비트)**:
  - QR (1bit): 질의(0) / 응답(1)
  - Opcode(4bit): 표준 질의(0), 역조회 등
  - AA (Authoritative Answer): 응답이 권한 서버에서 왔는지
  - TC (Truncated): UDP에서 잘린 경우(>512 bytes 등)
  - RD (Recursion Desired): 재귀 요청 여부
  - RA (Recursion Available): 서버가 재귀 지원 여부 표시
  - Z (3bit): 예약
  - RCODE(4bit): 응답 코드 (NOERROR, NXDOMAIN, SERVFAIL 등)
- **QDCOUNT**: Question 섹션 RR 수(보통 1)
- **ANCOUNT**: Answer RR 수
- **NSCOUNT**: Authority RR 수
- **ARCOUNT**: Additional RR 수

### Question / Answer / Authority / Additional

- **Question**: NAME, TYPE, CLASS (실제 RDATA 없음)
- **Answer**: 질문에 대한 정답 RR들
- **Authority**: 해당 존에 대한 NS, SOA 등 “권한 정보”
- **Additional**: 질의와 관련 있지만 **직접적인 답은 아닌** RR들
  (예: NS 레코드에 대응하는 A/AAAA 기록)

#### 예: `dig www.ietf.org` 요약

```text
;; HEADER: ID=0x1234, qr=1, ra=1, aa=0, rcode=NOERROR
;; QUESTION SECTION:
;www.ietf.org.          IN  A

;; ANSWER SECTION:
www.ietf.org.   300  IN  A  4.31.198.44

;; AUTHORITY SECTION:
ietf.org.       172800  IN  NS ns1.amsl.com.
...

;; ADDITIONAL SECTION:
ns1.amsl.com.   172800  IN  A   64.170.98.32
...
```

---

## Registrars

DNS에서 **도메인 이름 등록 구조**는 다음 네 주체를 중심으로 돌아간다.

1. **ICANN (Internet Corporation for Assigned Names and Numbers)**
   - 글로벌 정책 조정
   - gTLD 계약 관리, 레지스트리/레지스트라 승인 등

2. **Registry(레지스트리)**
   - 각 TLD(예: `.com`, `.org`)의 “제조사” 역할
   - 예: `.com` 레지스트리는 VeriSign
   - TLD **존 파일**과 TLD 네임 서버를 운영

3. **Registrar(레지스트라)**
   - 도메인 판매 “대리점”
   - ICANN 및 레지스트리로부터 승인 받은 사업자
   - 예: Cloudflare, GoDaddy, Network Solutions, 여러 유럽/미국 기반 업체들
   - 사용자의 도메인 등록/연장/이전, WHOIS/등록 정보 관리, 네임 서버 설정 인터페이스 제공

4. **Registrant(등록자)**
   - 실제 도메인 이름을 사용하는 개인/기업/기관

### 도메인 등록 흐름 예

“alice”가 `example.com` 도메인을 구입하는 과정을 단계별로 보면:

1. Alice → Registrar 웹사이트 접속
2. `example.com`에 대해 **중복 여부 검사**
   - 레지스트라가 `.com` 레지스트리(VeriSign)의 EPP(Extensible Provisioning Protocol) API를 통해 확인
3. 사용 가능하면:
   - Alice가 연락처/결제 정보 입력
   - 레지스트라가 레지스트리에 “새 도메인 등록” 요청
4. 레지스트리는:
   - `example.com`을 **zone 데이터베이스**에 추가
   - 해당 도메인의 **권한 네임 서버(NS)** 를 기록
   - `.com` TLD 존이 업데이트 → 루트 존에서 `.com` NS를 통해 전 세계에 전파
5. Alice는 레지스트라(또는 직접 운영하는 NS)를 통해 A/AAAA, MX 등 RR을 구성

이 과정에서 DNS 프로토콜 자체는 **네임 해석**에만 관여하고,
등록/결제/소유권 관리 등은 **레지스트리·레지스트라 비즈니스 레이어**에서 처리된다.

---

## DDNS (Dynamic DNS)

“Dynamic DNS”는 크게 두 가지 의미로 쓰인다.

1. **RFC 2136 기반 동적 업데이트(DNS Update)**
   - 전통적 존을 **프로그램적으로 업데이트** (예: BIND, Windows DNS 서버)
2. **상용 DDNS 서비스**
   - 가정용/소규모 네트워크에서 **변하는 공용 IP**를 특정 도메인 이름에 매핑해주는 서비스

### RFC 2136 Dynamic Update

RFC 2136은 DNS UPDATE 메시지 유형을 정의해,
존 파일을 수동 편집하지 않고도 **RR 집합을 동적으로 추가/삭제/변경**할 수 있게 한다.

UPDATE 메시지 구조:

- **Header**: 이 메시지가 UPDATE임을 나타냄
- **Zone Section**: 어느 존을 업데이트할지 지정
- **Prerequisite Section**: “현재 존이 어떤 상태여야 한다”는 전제 조건
  (예: 이 레코드가 이미 존재해야 한다 / 존재하면 안 된다)
- **Update Section**: 실제 추가/삭제할 RR 정보
- **Additional Section**: 필요시 추가 데이터

실제 구현 예(개념적):

```text
; host123.example.com A 레코드를 203.0.113.55로 갱신

Zone Section: zone "example.com"
Prerequisite: host123.example.com A record exists
Update: delete host123.example.com A
        add host123.example.com A 203.0.113.55
```

Windows Server DNS나 BIND는 DHCP 서버와 연동해
클라이언트 IP가 바뀔 때 자동으로 A/AAAA 및 PTR 레코드를 업데이트한다.

### 상용/경량 DDNS 서비스

소규모 네트워크에서 공용 IP가 계속 변하는 경우, 예를 들어:

- 가정용 인터넷 회선(동적 IP)
- 소형 지점 사무실

**DDNS 제공업체**는 다음과 같이 동작한다.

1. 사용자는 `myhome.example-ddns.net` 같은 도메인을 발급
2. 공유기/클라이언트가 주기적으로:
   - 자신의 현재 공용 IP를 DDNS 서버에 HTTP/HTTPS 혹은 전용 프로토콜로 보고
3. DDNS 서버는 해당 이름의 A/AAAA 레코드를 갱신
4. 외부에서 `myhome.example-ddns.net`으로 접속하면 항상 최신 IP로 연결

이 방식은 RFC 2136을 내부적으로 사용할 수도 있고,
자체 API·데이터베이스를 사용할 수도 있다.

---

## Security of DNS

DNS는 본질적으로 **평문, 인증·무결성 없는 질의/응답**으로 설계되었다.
오늘날에는 다양한 공격과 이에 대한 방어 기술이 중요하다.

### 주요 위협

1. **스푸핑 및 캐시 포이즈닝(Cache Poisoning)**
   - 공격자가 재귀 리졸버에게 **위조 응답**을 먼저 도착시키면,
     리졸버의 캐시에 **가짜 IP**가 저장될 수 있다.
   - 유명한 사례로 Kaminsky 공격(2008)이 있고, 이후 소스 포트 랜덤화, 0x20 인코딩 등이 도입되었다.

2. **DNS 리플렉션·앰플리피케이션 DDoS**
   - 공격자가 출발지 IP를 피해자 IP로 위조한 DNS 질의를
     **오픈 리졸버**나 권한 서버 여러 곳에 뿌리고,
     응답 크기가 질의보다 훨씬 커지는 특성을 이용해 대량 트래픽을 증폭시킨다.

3. **프라이버시 침해**
   - DNS 질의는 사용자가 어떤 사이트/서비스를 사용하는지 그대로 드러낸다.
   - 일부 규제 기관은 DNS 로그를 개인정보로 간주하고 보호 지침을 제시한다.

4. **존 전송(AXFR) 오남용**
   - 잘못 구성된 권한 서버가 전체 존을 AXFR로 누구에게나 내주면,
     내부 호스트명, 네트워크 구조 등이 노출된다.

### DNSSEC (DNS Security Extensions)

DNSSEC는 **DNS 데이터에 대한 무결성과 출처 인증**을 제공한다.

핵심 아이디어:

- 각 존은 자체 공개키/비밀키 쌍을 가지고,
  RRset에 대해 **디지털 서명(RRSIG)** 을 생성
- 상위 존은 하위 존 키의 요약(DS 레코드)을 저장
  → 루트에서 시작해 **신뢰 사슬(chain of trust)** 형성
- 검증하는 리졸버는:
  - 루트의 신뢰점(trust anchor)을 알고 있고
  - 하위로 내려가며 DS, DNSKEY, RRSIG를 검증하여
    응답이 변조되지 않았음을 확인

2010년대부터 루트 존과 대부분의 주요 TLD(.com, .org 등)에서 DNSSEC 서명이 활성화되었고, 2024년 ICANN은 루트 존 DNSSEC 알고리즘 롤오버에 관한 연구 보고서를 발표하며 운영 현황을 분석하고 있다.

예: DNSSEC 결과 보기

```bash
dig +dnssec www.nic.cz

;; ANSWER SECTION:
www.nic.cz.  300  IN  A      185.28.193.95
www.nic.cz.  300  IN  RRSIG  A 13 3 300 ...
```

- `RRSIG` 레코드는 A 레코드에 대한 서명
- 검증 가능한 리졸버라면, 변조된 응답에 대해 `SERVFAIL` 등을 반환

### 전송 계층 암호화 — DoT, DoH

DNSSEC는 데이터의 무결성과 출처를 보호하지만,
**질의 내용이 평문으로 보인다는 사실**은 변하지 않는다.
이를 보완하기 위해 IETF는 **DNS-over-TLS(DoT)** 와 **DNS-over-HTTPS(DoH)** 를 표준화했다.

1. **DNS-over-TLS (DoT, RFC 7858)**
   - TCP 853 포트에서 TLS로 DNS를 캡슐화
   - 일반 HTTPS 트래픽과 구분 가능(전용 포트 사용)
   - 많은 리졸버 소프트웨어에서 옵션으로 DoT 서버/클라이언트 지원

2. **DNS-over-HTTPS (DoH, RFC 8484)**
   - DNS 질의를 HTTPS 요청(HTTP/2/3)으로 전송
   - 웹 트래픽과 동일 포트(443)를 사용, 방화벽·프록시가 DNS를 구분하기 어려움
   - 브라우저가 자체적으로 DoH 리졸버를 선택하는 경우(예: 특정 공용 리졸버) 프라이버시·중앙 집중화 논의가 활발하다.

3. **DNS-over-QUIC (DoQ)**
   - UDP 기반 QUIC 위에서 DNS를 전송
   - 지연 감소와 연결 복구에 유리

이들 프로토콜은 **전송 경로 상의 도청/변조**를 어렵게 만들어,
ISP나 중간자에 의한 질의 내용 노출을 줄인다.

### 기타 보안 메커니즘

- **QNAME Minimization**
  - 리졸버가 상위 서버에 **필요 최소한의 이름 정보만** 보내도록 하여,
    예를 들어 `.com` TLD에 `www.example.com` 전체가 아니라 `example.com`만 전달
- **DNS Cookies (RFC 7873)**
  - 클라이언트/서버 간 “쿠키”를 사용해 응답이 예상된 출처에서 온 것인지 확인,
    일부 스푸핑·앰플리피케이션 공격을 줄이는 데 기여
- **Response Rate Limiting (RRL)**
  - 권한 서버에서 동일 쿼리/목적지에 대한 응답 속도를 제한해
    앰플리피케이션 공격에서 악용되는 것을 완화

유럽 및 북미의 데이터 보호 기관들은,
IoT 기기·엔드 유저 단말의 DNS 구현 시 **최소한의 로깅, 암호화 채널, 신뢰할 수 있는 리졸버 선택** 등을 권고하는 기술 가이드를 내고 있다.

---

## 간단 실습 예제 요약

마지막으로, 이 절에서 설명한 개념을 직접 체험해볼 수 있는 간단 실습들을 모아 두면 블로그 연재의 “실습 섹션”으로 재사용하기 좋다.

### 도메인 구조/네임스페이스 관찰

```bash
# 특정 이름의 AUTHORITY / ADDITIONAL 섹션 보기

dig www.example.com

# 루트부터 전체 추적

dig +trace www.ietf.org
```

### 리소스 레코드 관찰

```bash
# A, AAAA, MX, NS, TXT 등 개별 타입 조회

dig A www.example.com
dig AAAA www.example.com
dig MX example.com
dig NS example.com
dig TXT example.com
```

### 캐시 확인(리졸버 수준 실험)

1. 로컬 리졸버를 BIND/Unbound 등으로 구성
2. TTL이 짧은 테스트 레코드 생성(ex: 30초)
3. 몇 초 간격으로 반복 질의하며 **응답의 TTL 감소**를 관찰

### 동적 업데이트(DNS Update) 실험 (테스트 존에서만)

```bash
# 예: nsupdate 도구 사용 (BIND)

nsupdate
> server ns1.example.com
> zone example.com
> update add host123.example.com. 300 A 203.0.113.55
> send
```

- 이후 `dig host123.example.com`으로 결과 확인
- 실제 환경에서는 TSIG 키 기반 인증 필수

### DNSSEC/DoT/DoH 관찰

- `dig +dnssec`로 RRSIG/DS/DNSKEY 확인
- DoT/DoH 지원 리졸버에 대해:
  - `kdig @dns.example.net +tls`
  - 브라우저의 DoH 설정 활성화 후 패킷 캡처에서 평문 DNS가 사라지는지 확인

---

이렇게 정리하면 “26.6 Domain Name System” 절은 단순 이론 정리를 넘어서,
- **네임스페이스/메시지 구조**(RFC 기준),
- **운영 현황**(루트, TLD, 레지스트리/레지스트라),
- **운영 기술**(캐싱, DDNS),
- **보안 기술**(DNSSEC, DoT/DoH, RRL 등)
