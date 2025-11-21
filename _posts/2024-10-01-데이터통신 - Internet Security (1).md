---
layout: post
title: 데이터 통신 - Internet Security (1)
date: 2024-10-01 22:20:23 +0900
category: DataCommunication
---
# 네트워크 계층 보안 — IPsec, IKE, VPN 완전 정리

이 장은 **“네트워크 계층(Network Layer) 보안”** 관점에서
IPsec, IKE, 그리고 VPN을 체계적으로 정리하는 글이다.

- 먼저 **두 가지 모드(two models/modes)** 와
  **두 가지 보안 프로토콜(AH, ESP)** 를 정리하고,
- IPsec이 제공하는 **보안 서비스들**,
- **Security Association(SA)** 개념,
- **Internet Key Exchange(IKE)** (특히 IKEv2 중심),
- IPsec을 이용한 **VPN(가상 사설망)** 구조를
  “실제 운영”을 염두에 두고 설명한다.

실습/운영 상황은
- 리눅스(예: strongSwan, `ip xfrm`),
- 기업 환경에서의 **site-to-site VPN**, **remote access VPN**
을 기준으로 한다.
암호 알고리즘/구성은 2020년 NIST SP 800-77 Rev.1 및 NSA/NCSC 최신 가이드를 반영해
**AES-GCM, SHA-2, IKEv2** 중심으로 설명한다.

---

## 두 가지 모델(모드): Transport Mode vs Tunnel Mode

교재에서는 “two models” 혹은 “two modes”라고 부르지만,
실제로는 **IPsec을 어디까지 감쌀지**에 대한 두 가지 방식이다.

- **Transport Mode**: “IP 헤더는 그대로, 페이로드만 보호”
- **Tunnel Mode**: “IP 헤더까지 포함한 전체 패킷을 새 IP 헤더로 감싸서 보호”

### Transport Mode — end-to-end 보호

Transport 모드는 **호스트 ↔ 호스트** 통신에서
“이미 존재하는 IP 헤더는 그대로 두고, L4 페이로드(TCP/UDP 등)만 보호”한다.

#### 패킷 구조

대략적인 구조는 다음과 같다.

- AH + Transport 모드:

```text
[IP Header][AH Header][TCP/UDP Header + Data]
```

- ESP + Transport 모드:

```text
[IP Header][ESP Header][Encrypted { TCP/UDP Header + Data + ESP Trailer }][ESP Auth(optional)]
```

즉, **원래 IP 헤더는 평문으로 남고**, 그 뒤의 부분만 인증/암호화 대상이 된다.

#### 언제 쓰는가?

- **종단 호스트가 직접 IPsec을 구현**할 수 있을 때
  - 예: 서버와 서버 사이의 통신 (DB 서버 ↔ 애플리케이션 서버)
  - 예: 라우팅 프로토콜(OSPFv3, RIPng 등)에 대한 보호
- 라우팅 기반 정책 대신 **프로세스/소켓 단위 정책**으로 세밀하게 관리하고 싶을 때

#### 장단점

- 장점
  - 헤더가 그대로라서 **라우팅/필터링이 단순**하다.
  - 캡슐화 오버헤드가 상대적으로 적다.
- 단점
  - **IP 헤더가 평문**이라, 출발지/목적지/TTL 등의 메타데이터는 노출된다.
  - 호스트마다 IPsec 스택과 정책 설정이 필요 → 운영 복잡도 증가.

##### 작은 예제: 리눅스 strongSwan에서 Transport 모드 정책

```conf
conn db-end-to-end
    keyexchange=ikev2
    left=10.0.0.10        # App 서버
    leftsubnet=10.0.0.10/32[tcp]
    right=10.0.0.20       # DB 서버
    rightsubnet=10.0.0.20/32[tcp]
    type=transport
    esp=aes256gcm16-prfsha384
    ike=aes256gcm16-prfsha384-modp2048
    auto=start
```

- `type=transport` 로 Transport 모드를 명시.
- DB 서버로 가는 TCP 트래픽을 end-to-end로 보호.

---

### Tunnel Mode — VPN의 핵심

Tunnel 모드는 **원본 IP 패킷 전체를 보호**한다.

#### 패킷 구조

- ESP + Tunnel 모드:

```text
[Outer IP Header][ESP Header][Encrypted { Inner IP Header + TCP/UDP + Data + ESP Trailer }][ESP Auth(optional)]
```

- Outer IP: **VPN 게이트웨이 사이의 주소**
- Inner IP: **실제 통신하는 호스트의 주소**

이 구조는 RFC 4301의 IPsec 아키텍처에서 정의된다.

#### 언제 쓰는가?

- **Site-to-Site VPN (게이트웨이 ↔ 게이트웨이)**
  - 본사 ↔ 지사 연결
  - 데이터센터 ↔ 클라우드 VPC 연결
- **Host-to-Gateway VPN (원격 접속 VPN)**
  - 재택근무자가 회사 게이트웨이로 VPN 접속

#### 장단점

- 장점
  - 내부 주소(Inner IP)를 포함해 전체 패킷이 **암호화**되어,
    내부 네트워크 구조를 숨길 수 있다.
  - 게이트웨이에서만 IPsec을 설정하면 되므로,
    **클라이언트/서버는 평범한 IP만 사용**해도 된다 (site-to-site).
- 단점
  - **Outer IP 헤더 + ESP 헤더** 등 추가로 붙는 오버헤드 증가.
  - MTU 고려 필요 (경우에 따라 패킷 분할/Path MTU Discovery).

##### 예제: Site-to-Site VPN 시나리오

- 본사 LAN: 10.0.0.0/24, 게이트웨이: 203.0.113.1
- 지사 LAN: 10.1.0.0/24, 게이트웨이: 198.51.100.1

사용자는 단순히 `10.1.0.x` 서버에 접속하지만, 실제로는
게이트웨이 사이에 아래와 같은 **IPsec Tunnel 패킷**이 흐른다.

```text
[Outer IP: 203.0.113.1 → 198.51.100.1]
  ESP
    [Inner IP: 10.0.0.50 → 10.1.0.20]
      TCP/HTTP Payload
```

운영자는
- `Outer IP` 로 라우팅/방화벽 규칙을 잡고,
- 안쪽은 일반 내부 LAN처럼 운영한다.

---

## 두 가지 보안 프로토콜: AH와 ESP

IPsec은 두 가지 핵심 보안 프로토콜을 정의한다.

1. **AH (Authentication Header)**
2. **ESP (Encapsulating Security Payload)**

오늘날 실제 VPN/트래픽 보호에서는 **거의 항상 ESP만 사용**하고,
AH는 특수한 상황(헤더 인증만 필요할 때)에만 고려된다.

### AH — 인증 전용(헤더+페이로드)

AH는 **무결성/출처 인증**에 초점이 있는 프로토콜이다.

- 보호 범위
  - IP 헤더의 변경 불가능한 필드 (source, destination 등)
  - 상위 계층 페이로드(TCP/UDP/데이터)
- 제공 서비스
  - 데이터 무결성 (Integrity)
  - 데이터 출처 인증 (Origin authentication)
  - 재전송 공격 방지 (Anti-replay)

IP 헤더의 `Protocol` 필드는 AH를 나타내는 값(51)로 설정된다.
인증 데이터는 HMAC-SHA-256 같은 MAC으로 계산된다.

#### AH가 거의 안 쓰이는 이유

- NAT 환경과 잘 맞지 않는다.
  - NAT는 IP 헤더의 주소/포트를 수정하는데, AH는 “헤더까지 인증”한다.
  - NAT가 헤더를 바꾸면 인증이 깨진다.
- 실제 보안 요구 사항에서 **암호화(기밀성)** 도 함께 요구되는 경우가 많다.
- 따라서 **“헤더까지 인증 + 페이로드 암호화”** 를 ESP만으로도 충분히 제공할 수 있어
  AH는 사실상 거의 쓰이지 않는다 (표준에는 남아 있지만 실무에서 드묾).

### ESP — 암호화 + 인증(실전 주력)

ESP(Encapsulating Security Payload)는
**기밀성 + 무결성 + 출처 인증 + 재전송 방지**를 한 번에 제공한다.

#### 제공 서비스

- **기밀성(Confidentiality)**
  - AES-GCM, AES-CBC 등 대칭키 암호로 페이로드 암호화
- **무결성 + 출처 인증**
  - ESP-AES-GCM처럼 “암호화+인증 통합 모드(AEAD)”를 사용하거나
    별도 HMAC-SHA-2 MAC을 사용
- **재전송 공격 방지(Anti-Replay)**
  - ESP 헤더의 Sequence Number와 sliding window로 재전송 탐지

NIST SP 800-77 Rev.1 및 여러 서구 가이드에서는
**AES-GCM + SHA-2 + IKEv2** 조합을 기본 권장으로 보고 있다.

#### ESP 헤더/트레일러 구조 (간략)

```text
ESP Header:
  - SPI (Security Parameters Index)
  - Sequence Number

Encrypted Payload:
  - Original Payload (IP header+L4 or just L4, 모드에 따라)
  - Padding
  - Pad Length
  - Next Header

ESP Auth (선택 또는 AEAD 내부 태그)
```

#### 예시: 리눅스의 ESP SA 조회

리눅스에서 IPsec ESP SA를 보면 대략 이렇게 보일 수 있다:

```bash
ip xfrm state
```

예시 출력(요약 형태):

```text
src 203.0.113.1 dst 198.51.100.1
    proto esp spi 0xc0ffee01 reqid 1 mode tunnel
    replay-window 32 flag af-unspec
    auth-trunc hmac(sha256) 0x... 128
    enc aead(aes-gcm) 0x... 128
    sel src 10.0.0.0/24 dst 10.1.0.0/24
```

- `mode tunnel`: 터널 모드 ESP
- `enc aead(aes-gcm)`: 현대적인 AES-GCM 사용
- `replay-window 32`: 32개 패킷 윈도우 기준 anti-replay 동작

---

## IPsec이 제공하는 서비스(Services Provided by IPsec)

IPsec은 네트워크 계층에서 다음과 같은 보안 서비스를 제공한다.

### 기밀성(Confidentiality)

페이로드를 **대칭키 암호(AES 등)** 로 암호화한다.

- 보통 ESP를 통해 제공.
- Modern: AES-GCM, AES-CCM 등의 AEAD 모드 권장.

수식으로 표현하면, 평문 패킷 \(P\) 와 키 \(K\) 에 대해:

$$
C = \mathrm{Enc}_K(P), \quad
P = \mathrm{Dec}_K(C)
$$

여기서 \(\mathrm{Enc}\), \(\mathrm{Dec}\) 는 ESP에서 사용하는 블록 암호/모드를 의미한다.

### 및 출처 인증(Data Origin Authentication)

HMAC-SHA-256 같은 MAC을 사용하거나, AES-GCM과 같은 AEAD 모드에서
**Tag(인증 태그)** 로 무결성과 출처를 확인한다.

$$
\text{Tag} = \mathrm{MAC}_K(P)
$$

검증 시:

$$
\mathrm{MAC}_K(P) \stackrel{?}{=}\text{Tag}
$$

같으면 “수정되지 않았고, 해당 키를 가진 참여자에게서 왔다”고 본다.

### 재전송 공격 방지(Anti-Replay)

각 ESP 패킷에는 **Sequence Number**가 있고, 수신 측은
sliding window(예: 32 또는 64, ESN 사용 시 64bit) 로
“이미 본 번호”를 거부한다.

### 접근 제어/트래픽 선택(Access Control & Traffic Selector)

IPsec 정책은 “어떤 트래픽에 어떤 SA를 적용할지”를 정의한다.

예를 들어,

```text
from 10.0.0.0/24 to 10.1.0.0/24 proto tcp dport 443 → SA#1(강한 암호)
from 10.0.0.0/24 to 10.1.0.0/24 proto icmp        → SA#2(인증만)
```

처럼 **IP/포트/프로토콜 기반으로 보안 정책을 세밀하게 나눌 수 있다.

### 제한적인 트래픽 흐름 기밀성(Traffic Flow Confidentiality)

Tunnel 모드, 패딩, 고정 크기 패킷 등을 활용하면

- 어떤 호스트끼리,
- 얼마나 자주,
- 어느 방향으로

통신하는지 추론하기 어려워진다. 완전한 익명성은 아니지만
기본적인 “트래픽 패턴 노출"을 줄이는 효과가 있다.

---

## Security Association (SA)

**SA(Security Association)** 는
“**한 방향 단일 보안 관계**”를 의미한다.

> IPsec에서 실제 암호 알고리즘, 키, 모드,
> 수명(lifetime) 등을 담고 있는 **논리적 객체**.

### SA의 특징

- **단방향**: 하나의 SA는 “한 방향”에 대해서만 유효
  - A→B 와 B→A 통신에는 최소 2개의 SA 필요
- 프로토콜별: AH용 SA, ESP용 SA가 따로 존재
- 식별자: SPI(Security Parameters Index), IP 주소, 프로토콜 번호 조합

### SA에 들어가는 정보

- SPI (32비트)
- 목적지 IP 주소
- 프로토콜 (AH/ESP)
- 암호 알고리즘 (예: AES-GCM-256)
- 인증 알고리즘 (예: HMAC-SHA-256, 혹은 AEAD 내부)
- 키, 키 길이
- 수명 (바이트 기준, 시간 기준)
- 모드 (transport/tunnel)
- 트래픽 선택자(Selector): src/dst subnet, 포트, 프로토콜 등

### 예제: SA 설계 시나리오

회사 A가 아래 요구사항을 가진다고 해보자.

- 본사 ↔ 지사 사이 트래픽은
  - AES-GCM-256
  - 1시간마다 키 교체
  - 특정 서브넷만 보호(10.0.0.0/24 ↔ 10.1.0.0/24)

IKE 협상 시

1. A→B, B→A 방향으로 각각 ESP SA 생성
2. 각 SA는 SPI, 키, 알고리즘, 수명 정보를 가진다.
3. 라우팅 테이블과 정책 테이블이
   “해당 서브넷으로 가는 트래픽은 SA#1/SA#2 사용” 이라고 지정.

운영자는 `ip xfrm state`, `ip xfrm policy` (혹은 strongSwan `ipsec status`)로
SA 상태를 확인하며 트래픽이 올바르게 보호되는지 모니터링 한다.

---

## Internet Key Exchange (IKE)

IPsec 자체는 “데이터 패킷 포맷과 처리”만 정의한다.
**키 교환, 인증, SA 협상**을 위해서는 별도의 프로토콜이 필요하고,
그 역할을 하는 것이 **IKE(Internet Key Exchange)** 이다.

오늘날 새로 설계하는 환경에서는 사실상
**IKEv2(RFC 7296, 및 업데이트 RFC 8247/8983 등)** 만 사용한다고 보면 된다.

### IKE의 역할

1. **상호 인증(Mutual Authentication)**
   - 사설 CA 발행 X.509 인증서, PSK, EAP 등 사용
2. **암호 알고리즘/매개변수 협상**
   - “AES-GCM-256 + SHA-256 + ECDH P-256” 같은 조합 협상
3. **키 교환 및 도출**
   - Diffie-Hellman(또는 ECDH) 기반으로 세션 키 생성
4. **IPsec SA 생성/관리**
   - SPI, 키, 수명, 트래픽 선택자를 포함한 SA 생성
   - 재키잉(rekeying), 삭제(delete) 처리

IKE 자체도 **암호화/인증된 채널**을 사용한다.

### IKEv1 vs IKEv2 (간단 비교)

- IKEv1
  - 복잡한 Phase 1/Phase 2 구조, 여러 모드(main, aggressive 등)
  - 현재는 “레거시”로 간주, 새로운 설계에서는 비권장
- IKEv2
  - 메시지 교환이 단순해지고 에러 처리 향상
  - NAT traversal, MOBIKE, DoS 방지 기능 등 강화
  - NIST, NSA 등에서 **IKEv2 + 현대 암호(AES-GCM, SHA-2)** 사용을 권고

### IKEv2 교환의 개략

IKE_SA_INIT → IKE_AUTH → (CREATE_CHILD_SA 등)

1. **IKE_SA_INIT**
   - 암호 스위트 제안/선택
   - DH/ECDH 키 교환
   - Nonce 교환
2. **IKE_AUTH**
   - 인증서/PSK/EAP를 이용한 상호 인증
   - 첫 IPsec Child SA 생성
3. 이후 필요시 **CREATE_CHILD_SA** 메시지로 SA 재키잉/추가

##### 예제: strongSwan에서 IKEv2 제안

```conf
conn vpn
    keyexchange=ikev2
    ike=aes256gcm16-prfsha384-modp3072
    esp=aes256gcm16
```

운영자는
- `ike=` 파라미터로 IKE SA 암호 스위트,
- `esp=` 파라미터로 IPsec SA 암호 스위트를 제안하는 구조다.

---

## — IPsec의 대표 응용

IPsec의 대표적인 활용이 바로 **VPN(Virtual Private Network)** 이다.
NIST SP 800-77 Rev.1은 IPsec 기반 VPN 설계와 운영에 대한
대표적인 가이드 문서다.

### VPN의 개념

> “공용 네트워크(인터넷)를 통해 사설 네트워크를 확장”하는 기술

- **암호화된 터널**을 통해 “논리적으로는 하나의 사설망처럼” 동작
- 조직 내부의 프라이빗 IP 주소를 그대로 사용하면서
  지사, 클라우드, 원격 근무자 등을 연결

### 대표적인 IPsec VPN 아키텍처

NIST는 주로 세 가지 아키텍처를 구분한다.

1. **Gateway-to-Gateway (Site-to-Site)**
   - 본사 ↔ 지사, 데이터센터 ↔ 클라우드
   - 두 게이트웨이가 서로를 인증하고 Tunnel SA를 생성
   - 내부 호스트에는 별도 VPN 클라이언트가 필요 없음
2. **Host-to-Gateway (Remote Access)**
   - 사용자의 노트북/PC/모바일이 VPN 클라이언트를 실행하여
     조직의 VPN 게이트웨이에 접속
   - 원격 근무, 재택근무, 출장이 대표 사례
3. **Host-to-Host**
   - 두 호스트 간 end-to-end IPsec
   - 보통은 터널 모드보다는 Transport 모드 or 특별 환경에서 사용

### 간단 시나리오: Site-to-Site VPN vs Remote Access VPN

#### Site-to-Site VPN

- 본사 10.0.0.0/24, 지사 10.1.0.0/24
- 본사 Router A, 지사 Router B 가 서로 IKEv2로 인증
- 외부 인터넷은 203.0.113.1 ↔ 198.51.100.1 사이의 IPsec 터널만 인식

운영자 입장에서:

- 본사 라우터: `ip route 10.1.0.0/24 via 198.51.100.1 dev ipsec0`
- 지사 라우터: `ip route 10.0.0.0/24 via 203.0.113.1 dev ipsec0`
- IPsec 정책: `10.0.0.0/24 <-> 10.1.0.0/24` 모든 트래픽 ESP 터널로 보호

#### Remote Access VPN

- 직원 노트북: 공용 Wi-Fi에서 203.0.113.10(게이트웨이)로 IKEv2 접속
- 게이트웨이는 사용자를 인증(EAP-TLS 등) 후
  **가상 내부 주소(예: 10.0.10.25)** 를 할당하고 터널 생성
- 이후 사용자는 사내 자원(10.0.0.0/24)에 일반 내부 사용자처럼 접근 가능

### IPsec VPN vs TLS VPN (간단 비교)

- IPsec VPN
  - 네트워크 계층, IP 기반 보호
  - 모든 애플리케이션 트래픽을 한 번에 보호
  - 라우팅/정책 기반으로 섬세한 트래픽 선택
- TLS VPN (예: OpenVPN, HTTPS 기반 VPN)
  - 전송 계층 상단에서 “가상의 TCP/UDP 터널” 형식
  - 방화벽 우회를 위한 443/TCP 재사용에 유리
  - 특정 애플리케이션(웹, 프록시 등)에 최적화된 경우 많음

실무에서는

- **Site-to-Site → IPsec(VPN 게이트웨이 간 Tunnel)**
- **특수/제약 환경 또는 App-단 터널링 → TLS-기반 VPN**

조합으로 쓰는 경우도 많다.

---

## 정리: 네트워크 계층 보안의 위치

지금까지 정리한 내용은 크게 다음 흐름으로 이해할 수 있다.

1. **Two Modes(Models)**
   - Transport: end-to-end, 헤더는 평문
   - Tunnel: gateway 기반, 전체 패킷 보호 → VPN의 핵심
2. **Two Security Protocols**
   - AH: 헤더+페이로드 인증만, 실무에선 거의 안 씀
   - ESP: 암호화 + 인증 + anti-replay → 실전 표준
3. **IPsec Services**
   - Confidentiality, Integrity, Origin Authentication, Anti-Replay,
     Access control, 제한적 Traffic Flow Confidentiality
4. **Security Association**
   - “한 방향 보안 컨텍스트”
   - SPI, 키, 알고리즘, 수명, 선택자 등 포함
5. **IKE (특히 IKEv2)**
   - 상호 인증, 키 교환, SA 생성/관리 담당
   - 현대 암호 스위트(AES-GCM, SHA-2, ECDH)와 함께 사용
6. **VPN**
   - Site-to-Site, Remote Access, Host-to-Host
   - 조직이 공용 인터넷 위에 “자신들만의 사설망”을 만드는 핵심 도구

이 구조를 이해해 두면 이후
- **TLS**,
- **응용 계층 보안(PGP, S/MIME)**,
- **방화벽, IDS/IPS**
같은 상위·주변 기술을 공부할 때도
