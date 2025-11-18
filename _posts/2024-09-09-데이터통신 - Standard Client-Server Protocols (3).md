---
layout: post
title: 데이터 통신 - Standard Client-Server Protocols (3)
date: 2024-09-09 20:20:23 +0900
category: DataCommunication
---
# 26.3 Electronic Mail — 아키텍처, 웹 기반 메일, 이메일 보안

이 절에서는 **전자메일 시스템 전체 그림**을 한 번에 잡는다:

- **Architecture**: MUA/MTA/MDA/IMAP/POP3/SMTP 역할과 흐름
- **Web-based mail**: Gmail, Outlook.com 같은 웹 메일의 내부 구조
- **Email security**: TLS(STARTTLS)·MTA-STS·DANE, SPF/DKIM/DMARC, S/MIME/OpenPGP 등 현대 이메일 보안 스

---

## 26.3.1 Architecture — 인터넷 메일 시스템의 구성 요소와 동작

### 1) 기본 컴포넌트와 역할

IETF의 *Internet Mail Architecture* 문서와 NIST 가이드에 따르면, 현대 이메일 시스템은 **역할(role)** 기반으로 나누어 설명하는 것이 일반적이다.   

주요 구성 요소:

| 역할 | 약어 | 설명 |
|------|------|------|
| Mail User Agent | **MUA** | 사용자가 직접 사용하는 메일 클라이언트 (Thunderbird, Outlook, 모바일 메일 앱 등) |
| Mail Submission Agent | **MSA** | 사용자의 메일을 수신·검사 후 MTA로 넘기는 송신 프론트엔드 (보통 SMTP 587/TCP) |
| Mail Transfer Agent | **MTA** | 도메인 간 이메일을 릴레이하는 백엔드 서버 (Postfix, Exim 등) |
| Mail Delivery Agent | **MDA** | 최종 수신자 메일박스에 메시지를 저장하는 컴포넌트 |
| Mail Access Agent | **MAA / IMAP/POP 서버** | 수신자가 IMAP/POP3로 메일을 읽을 수 있게 해주는 서버 |

**프로토콜 관점**:

- **SMTP (Simple Mail Transfer Protocol)**: 메일을 *보낼 때* 사용 (MUA→MSA, MSA→MTA, MTA→MTA, MTA→MDA)   
- **POP3/IMAP**: 메일을 *읽을 때* 사용 (MUA→MAA)   

### 2) 기본 흐름: Alice가 Bob에게 메일을 보낼 때

`alice@example.com` 이 `bob@univ.edu`로 메일을 보내는 상황을 보자.

#### (1) 사용자 단계: MUA

1. Alice는 MUA(예: Thunderbird)에서 새 메일 작성  
2. MUA는 사용자가 `보내기`를 누르는 순간, 메일을 **SMTP 클라이언트** 형태로 변환:
   - 헤더(From, To, Subject, Date…) + 본문(텍스트/HTML)
   - 첨부 파일은 MIME 인코딩(base64 등)으로 포함

#### (2) 송신 도메인 내부: MSA/MTA

3. MUA → 송신 서버(MSA)로 SMTP 제출
   - 보통 포트 **587/TCP (submission)** + TLS(STARTTLS 또는 직접 TLS)
4. MSA는 인증, 스팸·정책 검사 후 내부 MTA에 전달
5. MTA는 DNS에서 `univ.edu`의 MX 레코드 조회
   - 해당 도메인의 수신 MTA IP를 찾음
6. 송신 MTA → 수신 MTA로 SMTP 전송 (보통 25/TCP, 점점 TLS 사용이 증가하는 추세)   

#### (3) 수신 도메인 내부: MDA/MAA

7. 수신 MTA는 **수신 정책, 스팸 필터, 바이러스 검사** 등 수행
8. 최종 승인되면 **MDA**가 Bob의 메일박스(예: `/var/mail/bob` 또는 DB)에 저장
9. Bob의 MUA(데스크톱, 모바일 앱 등)는 POP3 또는 IMAP으로 자신의 서버에 접속해 메일을 읽는다.

이 흐름을 단순 그림으로 나타내면:

```text
[ Alice(MUA) ]
      |
      | SMTP (submission, 587, TLS)
      v
[ MSA/MTA @ example.com ]
      |
      | SMTP (relay, 25, often TLS)
      v
[ MTA/MDA @ univ.edu ]
      |
      | IMAP/POP3 (143/993, 110/995, TLS)
      v
[ Bob(MUA) ]
```

### 3) 메일 주소, DNS, MX

**메일 주소**: `local-part@domain`

- `local-part`: 사용자 ID, alias 등 (예: `alice`, `john.smith`)
- `domain`: DNS 도메인 이름 (예: `example.com`)

메일을 라우팅할 때는 **domain 부분만** 보며, DNS에서 **MX 레코드**를 조회한다.

예를 들어 `example.com` 의 DNS 일부:

```txt
example.com.      IN MX 10 mx1.example.com.
example.com.      IN MX 20 mx2.example.com.
mx1.example.com.  IN A     203.0.113.10
mx2.example.com.  IN A     203.0.113.20
```

- 송신 MTA는 우선순위 10, 20을 고려해 서버를 선택한다.
- 다수의 MX를 둠으로써 **장애 내성**을 확보한다.

### 4) 메시지 포맷: RFC 5322, MIME

인터넷 이메일 메시지는 크게 두 부분으로 구성된다:

```text
헤더(Headers)
빈 줄
본문(Body)
```

간단한 예:

```text
From: Alice <alice@example.com>
To: Bob <bob@univ.edu>
Subject: Meeting Schedule
Date: Mon, 17 Nov 2025 10:00:00 +0900
Message-ID: <1234.5678@example.com>
MIME-Version: 1.0
Content-Type: text/plain; charset="utf-8"

Hi Bob,
Can we move our meeting to 3 PM?
Thanks,
Alice
```

- 헤더: 라우팅·표시·정책에 중요한 메타데이터들
- 본문: 텍스트 또는 MIME 구조
- 첨부 파일, HTML 본문, 다국어 등은 모두 **MIME 확장**으로 제공된다.

### 5) SMTP 세션 예제 (단순 형태)

테스트용으로 로컬 MTA에 직접 SMTP를 던져볼 수 있다. (실제 인터넷에서는 인증·TLS를 반드시 사용해야 한다.)

```text
$ telnet mail.example.com 25
220 mail.example.com ESMTP Postfix

HELO client.example.net
250 mail.example.com

MAIL FROM:<alice@example.com>
250 2.1.0 Ok

RCPT TO:<bob@univ.edu>
250 2.1.5 Ok

DATA
354 End data with <CR><LF>.<CR><LF>
From: alice@example.com
To: bob@univ.edu
Subject: Test

This is a test message.
.
250 2.0.0 Ok: queued as ABCDE12345

QUIT
221 2.0.0 Bye
```

실무에서는 이 모든 과정을 MUA나 서버 소프트웨어(Postfix, Exim 등)가 자동으로 처리한다.   

---

## 26.3.2 Web-based Mail — 웹 기반 메일의 구조와 동작

이제 데스크톱 MUA 대신 **브라우저**에서 Gmail/Outlook.com 같은 웹 메일을 쓸 때 구조가 어떻게 달라지는지 보자.

### 1) 전통적인 MUA vs 웹 메일

| 항목 | 전통 MUA (Outlook, Thunderbird 등) | 웹 메일 (Gmail, Outlook.com 등) |
|------|------------------------------------|----------------------------------|
| 클라이언트 | 로컬 설치 앱 | 웹 브라우저(HTML/JS) |
| 서버와 통신 | SMTP(587), IMAP/POP3(143/993/110/995) | HTTPS (TLS) |
| 메시지 저장 | 선택적으로 로컬 캐시 + 서버 | 대부분 서버 측 스토리지 (클라우드) |
| 업데이트 | 클라이언트 업그레이드 필요 | 서버/웹앱 자동 업데이트 |

내부적으로는 웹 메일도 **백엔드에서 SMTP/IMAP 계열 프로토콜**이나 유사한 메일 저장소 프로토콜을 사용하지만,  
사용자 입장에서는 **HTTP/HTTPS API**만 보게 된다.

### 2) 웹 메일 아키텍처

개념적으로 다음과 같이 볼 수 있다:

```text
[Browser: HTML/JS SPA]
        |
        | HTTPS (REST/JSON, WebSocket 등)
        v
[Web Frontend / API Gateway]
        |
        | 내부 RPC / IMAP-like / DB 접근
        v
[Mail Storage + Index + MTA/Delivery 시스템]
```

웹 메일에서는:

- 브라우저는 **단순한 클라이언트**가 아니라,
  - SPA(Single Page Application) 형태로 동작
  - JavaScript로 메일 목록, 검색, 라벨링, 쓰레딩(threading)을 처리
- 서버는
  - 메일박스 저장소 (대규모 분산 저장)
  - 인덱스/검색(예: inverted index, full-text search)
  - 스팸 필터, 바이러스 검사, 정책 엔진
  - SMTP/IMAP 게이트웨이 등

### 3) 웹 메일에서 메일 보내기: 예시 시나리오

`alice@gmail.com` 이 브라우저에서 웹 메일로 메일을 보낼 때:

1. 사용자가 브라우저에서 새 메일 편집, **“Send”** 클릭
2. 브라우저 → Gmail 서버로 **HTTPS POST** 요청:

   ```http
   POST /gmail/v1/users/me/messages/send HTTP/1.1
   Host: mail.example.com
   Authorization: Bearer <OAuth token>
   Content-Type: application/json

   {
     "raw": "<base64url encoded RFC 5322 message>"
   }
   ```

3. 웹 서버는 인증(OAuth 토큰 검증) 후:
   - 스팸/정책 검사
   - 내부 송신 MTA에 메시지 전달
4. 이후 **MTA→상대 도메인 MTA** 로 SMTP 전송 (앞에서 본 것과 동일)

즉, **사용자–서버 구간은 HTTP(S)**,  
**서버–서버 구간은 여전히 SMTP**라는 점이 중요하다.

### 4) 웹 메일에서 메일 읽기

메일 읽기 역시 REST/GraphQL 스타일 API를 통해 이루어질 수 있다:

```http
GET /gmail/v1/users/me/messages?label=INBOX&maxResults=50
Authorization: Bearer <token>
```

서버는:

- 메일 저장소에서 INBOX의 최신 메시지 50개 메타데이터를 조회
- JSON 형태로 브라우저에 반환

브라우저는:

- 목록을 렌더링
- 특정 메시지를 클릭하면 다시:

  ```http
  GET /gmail/v1/users/me/messages/<id>?format=full
  ```

와 같이 메시지 전체를 가져와 화면에 표시한다.

### 5) 푸시, 알림, 오프라인

현대 웹 메일은 다음과 같은 추가 기능을 제공한다:

- **푸시 알림**: WebSocket / HTTP/2 서버 푸시 / Web Push API 활용
- **오프라인 모드**: Service Worker + IndexedDB 로 일부 메일 캐시
- **스마트 기능**:
  - **자동 분류**(Primary, Promotions, Social 등)
  - **스팸·피싱 탐지**
  - 자동 답장 제안, 스레드 묶음 등

이러한 기능들은 모두 **서버 측 ML 모델 + 클라이언트 측 UI**가 함께 구현하는 부분이다.

---

## 26.3.3 Email Security — 현대 이메일 보안 스택

이제 가장 중요한 부분인 **이메일 보안**을 정리하자.  
위협과 방어 메커니즘을 크게 네 가지 축으로 보면 이해가 쉽다:

1. **전송 구간 보호**: TLS(STARTTLS), MTA-STS, DANE
2. **도메인·발신자 인증**: SPF, DKIM, DMARC
3. **종단 간 암호화**: S/MIME, OpenPGP
4. **계정·운영 보안**: MFA, 스팸 필터, 정책·로깅 등

유럽의 최신 분석에 따르면, EU 도메인에서 **STARTTLS, SPF, DKIM, DMARC**의 도입률은 이미 80~90% 이상으로,  
전통적인 이메일 보안 표준은 상당히 널리 채택된 상태다. 다만 DNSSEC·DANE 같은 고급 메커니즘의 도입률은 아직 낮다.   

### 3.1 위협 모델 — 무엇을 막으려고 하는가?

대표적인 이메일 위협:

- **도청(eavesdropping)**: 전송 중 이메일 내용을 몰래 보는 공격
- **위·변조(tampering)**: 메일 내용을 중간에서 바꾸기
- **스푸핑(spoofing)**: From 주소를 조작해 **보낸 사람을 위장**
- **피싱(phishing)·BEC(Business Email Compromise)**:
  - 공격자가 CEO나 파트너를 사칭해 송금/정보 탈취
  - 미국 FBI는 BEC 사기가 수백억 달러 규모에 이른다고 경고한다.   
- **멀웨어·랜섬웨어 첨부 파일**
- **계정 탈취**: 비밀번호 탈취, MFA 미적용 계정 공격 등

각 보안 기술이 어떤 위협을 겨냥하는지 항상 연결해서 기억하는 것이 중요하다.

---

### 3.2 전송 구간 암호화 — TLS, STARTTLS, MTA-STS, DANE

#### 1) TLS와 STARTTLS

**TLS(Transport Layer Security)** 는 HTTP뿐 아니라 SMTP, IMAP, POP3 등에도 적용되는 **전송 계층 암호화 프로토콜**이다.   

- SMTP: 포트 25/587에서 **STARTTLS** 확장 사용
- IMAP: 143/993, POP3: 110/995 등에서 TLS 사용

예: SMTP STARTTLS 흐름 (단순화):

```text
S: 220 mail.univ.edu ESMTP
C: EHLO mail.example.com
S: 250-STARTTLS
   250-SIZE 35882577
   250 AUTH LOGIN PLAIN
C: STARTTLS
S: 220 Ready to start TLS
--- TLS 핸드셰이크 ---
C: EHLO mail.example.com
S: 250-SIZE 35882577
   250 AUTH LOGIN PLAIN
...
```

- 1차 EHLO에서 서버가 `STARTTLS` 지원을 알림
- 클라이언트는 `STARTTLS` 명령으로 TLS 핸드셰이크 수행
- 이후 모든 메일 데이터(헤더·본문 포함)는 암호화된 터널 안에서 전송

하지만 **기본 SMTP는 “암호화 필수”가 아니라 “있으면 사용”** 모델이라,
중간자 공격으로 STARTTLS를 다운그레이드시키는 공격이 가능하다는 점이 발견되었다.   

#### 2) MTA-STS (Mail Transfer Agent – Strict Transport Security)

**MTA-STS(RFC 8461)** 는 도메인 소유자가

> “우리 도메인으로 오는 메일은 반드시 TLS로, 그리고 유효한 인증서를 가진 서버로만 받아라”

라는 정책을 게시할 수 있는 메커니즘이다.   

핵심 아이디어:

- 수신 도메인(`example.com`)은 **HTTPS로 정책 파일**(MTA-STS policy)을 호스팅
- 송신 MTA는 이 정책을 미리 캐시해두고, 메일을 보낼 때:
  - TLS가 성립하지 않거나
  - 인증서 검증에 실패하면
  - **메일 전송을 포기하거나(report-only 모드에서는 보고만)** 한다.

간단한 정책 예:

```txt
version: STSv1
mode: enforce
mx: mx1.example.com
mx: mx2.example.com
max_age: 86400
```

이렇게 하면:

- `example.com` 에 메일을 보내는 SMTP 클라이언트는:
  - 반드시 `mx1` 또는 `mx2` 를 사용
  - TLS가 필수
- 중간자 공격으로 암호화를 제거하거나, MX를 다른 곳으로 바꾸려 해도 차단 가능

#### 3) DANE for SMTP

**DANE(DNS-based Authentication of Named Entities)** 는 DNSSEC을 이용해  
SMTP 서버의 TLS 인증서를 **DNS에 직접 게시**하는 방식이다.   

- 도메인에 DNSSEC을 적용
- `TLSA` 레코드를 통해 메일 서버의 인증서 또는 공인 인증 기관(CA)에 대한 정보를 명시
- 송신 MTA는 DNSSEC 검증 + TLSA 레코드를 확인해:
  - 우회된 인증서·가짜 서버를 탐지
  - STARTTLS 다운그레이드 공격을 방지

연구와 업계 문서에서는, DANE가 MTA-STS보다 보안적으로 강력하지만  
DNSSEC 도입이 어렵다는 현실적 문제가 있다고 평가한다.   

---

### 3.3 발신자 인증 — SPF, DKIM, DMARC

이제 **스푸핑/피싱** 문제를 다루는 핵심 스택: **SPF, DKIM, DMARC**를 보자.   

#### 1) SPF (Sender Policy Framework)

**문제**: 아무나 `From: alice@example.com` 으로 이메일을 보낼 수 있다.

**해결 아이디어**:

> “`example.com` 이라고 주장하는 메일은 **이 IP/호스트 목록에서만 보내도록 하겠다**”  
> 라는 정책을 DNS에 TXT 레코드로 게시한다.

SPF 레코드 예:

```txt
example.com.  IN TXT  "v=spf1 ip4:203.0.113.10 ip4:203.0.113.11 include:_spf.mailprovider.com -all"
```

의미:

- `203.0.113.10`, `203.0.113.11`, 그리고 `_spf.mailprovider.com` 에 정의된 IP들이 **합법적 발신자**
- 나머지는 모두 `-all` → fail

수신 MTA는:

1. SMTP 세션에서 *Envelope From*(MAIL FROM) 도메인 확인
2. 그 도메인의 SPF TXT 레코드를 DNS 검색
3. 실제 발신 IP가 해당 레코드에 포함되어 있는지 검사
4. pass / fail / softfail / neutral 등의 결과를 바탕으로 스팸 점수에 반영

#### 2) DKIM (DomainKeys Identified Mail)

DKIM은 **도메인 기반 디지털 서명**이다.

- 발신 도메인은 **개인키**를 사용해 메일 헤더/본문 일부에 서명
- 수신자는 DNS에 게시된 **공개키**를 사용해 서명 검증

예시 DKIM 헤더:

```text
DKIM-Signature: v=1; a=rsa-sha256; d=example.com; s=mail;
 h=from:to:subject:date;
 bh=Jv2+XoGBw... (본문 해시)
 b=QGEvGQq8G0... (서명 값)
```

DNS의 공개키:

```txt
mail._domainkey.example.com. IN TXT "v=DKIM1; k=rsa; p=MIIBIjANBgkq..."
```

의미:

- 메시지가 **발송 후 위조되지 않았는지** 검증
- 발신 도메인(`example.com`)이 실제로 이 메시지를 **승인**했는지 증명

#### 3) DMARC (Domain-based Message Authentication, Reporting and Conformance)

DMARC는 SPF·DKIM을 **‘조합’해서 정책과 리포팅을 제공**하는 프레임워크다.   

핵심 개념:

1. SPF 또는 DKIM 중 **하나 이상이 PASS여야 한다.**
2. PASS 결과와 **From 헤더 도메인이 “alignment(정렬)”되는지** 확인
3. 도메인은 DNS TXT 레코드로 “검증 실패 시 어떻게 할지” 정책을 게시

DMARC 레코드 예:

```txt
_dmarc.example.com. IN TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@example.com; adkim=s; aspf=s"
```

- `p=quarantine`: DMARC 검사에 실패한 메일은 격리(스팸함 이동) 권장
- `rua`: DMARC Aggregate Report 수신 이메일 주소
- `adkim=s`, `aspf=s`: DKIM/SPF alignment를 strict로 요구

이렇게 하면:

- **발신자 도메인 스푸핑**(예: `From: ceo@bigbank.com` 으로 사칭) 차단에 큰 효과
- 대규모 피싱·BEC 공격 완화

EU의 2024년 분석 결과에서도 SPF, DKIM, DMARC는 대부분의 주요 도메인에서 채택되고 있으며,  
도입률은 80% 이상에 달한다고 보고한다.   

---

### 3.4 종단 간 암호화 — S/MIME, OpenPGP

전송 구간(TLS) 보호만으로는:

- 메일 서버 운영자
- 서버에 침투한 공격자
- 법적 요청에 따라 서버에 접근하는 3자

가 내용을 볼 수 있다는 한계가 있다.

이를 보완하는 개념이 **End-to-End Encryption(E2EE)** 으로,  
대표적인 기술이 **S/MIME**와 **OpenPGP**다.   

#### 1) S/MIME (Secure/Multipurpose Internet Mail Extensions)

- X.509 인증서를 사용하는 **PKI 기반** 암호화/서명 표준
- Outlook, Apple Mail 등 기업 환경에서 널리 지원
- 구조:
  - 발신자는 수신자의 **공개키**로 본문/첨부를 암호화
  - 수신자는 자신의 **개인키**로 복호화
  - 발신자는 자신의 개인키로 서명 → 수신자는 공개키로 검증

헤더 예(서명/암호화 된 메시지):

```text
Content-Type: multipart/signed; protocol="application/pkcs7-signature"; ...
...
Content-Type: application/pkcs7-signature; name="smime.p7s"
Content-Transfer-Encoding: base64
...
```

장점:

- 기업 CA를 통한 중앙 관리 용이
- Outlook/Exchange 같은 엔터프라이즈 스택과 친화적

단점:

- 인증서 발급·갱신 관리 부담
- 개인 사용자 환경에서 설정이 어렵다는 usability 문제 (연구에서도 반복적으로 지적).   

#### 2) OpenPGP (PGP/GPG 메일 암호화)

- PGP 철학 기반의 **Web-of-Trust** 모델
- GnuPG(GPG) 클라이언트와 연동되는 다양한 MUA 플러그인 존재
- 메일 본문은 보통 다음처럼 표시:

```text
-----BEGIN PGP MESSAGE-----
Version: GnuPG v2

hQEMA5...
-----END PGP MESSAGE-----
```

OpenPGP의 특징:

- 중앙 CA 없이 사용자가 서로의 공개키를 직접 교환/검증
- 오픈소스 생태계에서 선호
- 역시 키 관리·사용성의 어려움이 실무 도입의 큰 장애 요인으로 연구되고 있다.   

---

### 3.5 계정·운영 보안 — MFA, 스팸 필터, 정책

마지막으로, **프로토콜 수준 보안** 외에 운영·사용자 보안 관점에서 중요한 요소들:

1. **MFA(다단계 인증)**:
   - 웹 메일·기업 메일 계정에 **비밀번호 + OTP·FIDO2** 등 적용
   - 피싱에 취약한 SMS보다는 앱 기반 OTP·보안키(FIDO2) 권장
2. **스팸·피싱 필터**:
   - 헤더·본문·URL·첨부·IP 평판 등을 종합한 ML 기반 필터
3. **악성 첨부 파일 탐지**:
   - 샌드박스 실행, 알려진 시그니처, 행위 기반 분석
4. **로그·감사·경보**:
   - 비정상 로그인 시도, 해외 IP 접속, 대량 발송 탐지
5. **정책/교육**:
   - 임직원 대상 피싱 모의 훈련
   - 송금·결제 관련 승인 프로세스(“이메일 지시만으로는 송금하지 않는다”)

최근 시장 조사에 따르면, 이메일 암호화·보안 솔루션 시장은 2025년 약 90억 달러에서 2030년 230억 달러 이상으로 성장할 것으로 전망된다. 이는 **이메일이 여전히 핵심 비즈니스 채널**이며, 동시에 주요 공격 대상이라는 현실을 반영한다.   

---

## 마무리 정리

이 절에서 본 내용들을 한 번에 요약하면:

1. **Architecture**:
   - MUA → MSA/MTA → MTA/MDA → IMAP/POP3 → MUA 구조
   - SMTP는 “보내기”, IMAP/POP3는 “읽기”
   - DNS MX, RFC 5322 메시지 포맷, MIME

2. **Web-based mail**:
   - 사용자–서버: HTTPS 기반 웹 애플리케이션 (SPA + REST/JSON)
   - 서버–서버: 여전히 SMTP
   - 대규모 분산 저장, 검색, 스팸 필터, 푸시/오프라인 기능

3. **Email security**:
   - 전송 구간 암호화: TLS/STARTTLS, MTA-STS, DANE
   - 발신자 인증: SPF, DKIM, DMARC (EU 도메인에서 높은 도입률)
   - 종단 간 암호화: S/MIME, OpenPGP
   - 계정·운영 보안: MFA, 스팸/피싱 필터, 로깅·정책