---
layout: post
title: 데이터 통신 - Internet Security (2)
date: 2024-10-01 23:20:23 +0900
category: DataCommunication
---
# — 아키텍처와 네 개의 프로토콜

## SSL/TLS의 위치와 기본 개념

### 어디에 위치하는가?

TLS는 **“애플리케이션과 TCP 사이”** 에 끼어 있는 보안 계층이다.

```text
[애플리케이션 계층]  HTTP, SMTP, IMAP, MQTT, ...
         │
         ▼
[TLS / SSL]  ← 이번 챕터의 주인공
         │
         ▼
[전송 계층]   TCP (또는 UDP 위의 DTLS)
         │
         ▼
[네트워크 계층]  IP
```

- 애플리케이션 입장에서는 “보안이 적용된 **가상 TCP**” 를 쓰는 느낌이다.
- TCP 소켓 대신 **TLS 소켓**(예: `SSL*` 구조체, `ssl.SSLSocket`)을 쓰면,
  - 데이터는 자동으로 **암호화·무결성 보호·인증**된 채 전송된다.

### SSL vs TLS, 그리고 오늘날의 현실

역사적으로는:

| 연도 | 프로토콜 | 비고 |
|------|----------|------|
| 1990s 중반 | SSL 2.0 / 3.0 | 오늘날 **심각하게 취약**, IETF에서 공식 폐기 |
| 1999 | TLS 1.0 | SSL 3.0 기반의 표준화 버전 |
| 2006 | TLS 1.1 | 패딩 오라클 방어 등 일부 개선 |
| 2008 | TLS 1.2 | AES, SHA-2, 확장성 개선 — 여전히 널리 쓰임 |
| 2018 | TLS 1.3 | 설계 전면 재구성, 속도·보안 대폭 향상 |

미국 NIST와 NSA, EU 권고를 보면:​

- **SSL 2.0 / 3.0, TLS 1.0 / 1.1** → 사용 금지
- **TLS 1.2, TLS 1.3만 사용** 권고
- 새 프로토콜은 **TLS 1.3만 사용**, 기존 애플리케이션은 최소 TLS 1.2/1.3 지원

즉, 이 장에서 `SSL`이라는 용어가 나오더라도 **실제 운영에서는 TLS 1.2/1.3을 쓴다**고 이해하면 된다.

---

## SSL/TLS 아키텍처(SSL Architecture)

### 큰 그림: Record Layer + 여러 프로토콜

TLS는 크게 두 덩어리로 나눠 볼 수 있다.

1. **Record Protocol (레코드 계층)**
   - “암호화된 파이프”를 만든다.
   - 상위에서 내려오는 데이터를 잘게 잘라 **레코드(record)** 로 만들고,
     - 압축(요즘은 거의 사용 안 함)
     - MAC/AEAD 태그 생성
     - 암호화
   - 다시 TCP 스트림에 실어 보낸다.

2. **네 개의 상위 프로토콜(이번 장의 핵심)**
   Record Protocol 위에서 여러 “내용물(content type)”이 돌아다닌다.
   - **Handshake Protocol**
   - **ChangeCipherSpec Protocol**
   - **Alert Protocol**
   - **Application Data Protocol**

ASCII 구조:

```text
Application Protocols   →  HTTP, SMTP, IMAP, ...
        │
        ▼
   TLS Protocols        →  Handshake / Alert / ChangeCipherSpec / Application Data
        │
        ▼
   TLS Record Protocol  →  {content-type, version, length, fragment(ciphertext)}
        │
        ▼
        TCP
```

### vs 연결(Connection)

교과서에서 자주 강조하는 개념:

- **세션(Session)**
  - 클라이언트와 서버가 한번의 핸드셰이크로 합의한 **공유 보안 상태**(cipher suite, 키, 압축 방법 등)의 묶음.
  - TLS 1.2까지는 세션을 재사용(resume)해서 핸드셰이크를 줄이는 용도로 많이 썼다.
- **연결(Connection)**
  - 하나의 TCP 연결 위에서 동작하는 실제 데이터 교환.
  - 여러 연결이 한 세션을 공유할 수도 있고, 반대로 한 세션만 재사용하는 간단한 구조도 가능.

TLS 1.3에서는 “세션” 개념이 **PSK(Pre-Shared Key) + 세션 티켓** 구조로 재정의되긴 했지만,
“한 번 합의해 둔 공유 키를 이후 접속에서 재사용해서 1-RTT, 0-RTT로 빠르게 붙는다”는 큰 그림은 같다.

### Cipher Suite(암호 스위트)

TLS에서 **어떤 알고리즘을 쓸지** 를 하나로 묶어 이름 붙인 것이 **cipher suite** 다.

고전적인 TLS 1.2 형태:

```text
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

- **키 교환**: ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)
- **인증 알고리즘**: RSA
- **대칭 암호**: AES-128
- **운영 모드 / 무결성**: GCM (AEAD — 암호+무결성)
- **해시**: SHA-256

TLS 1.3에서는 설계가 단순해져서, cipher suite는 **사실상 “AEAD + 해시” 묶음**만 나타내고,
키 교환(ECDHE)·인증(서버 인증서 등)은 별도 확장과 메시지에서 정의한다.

### TLS Record의 구조

TLS 1.2 기준 레코드 헤더:​

| 필드 | 크기 | 설명 |
|------|------|------|
| ContentType | 1 바이트 | 상위 프로토콜 종류 (Handshake, Alert, Application Data 등) |
| Version | 2 바이트 | TLS 버전(형식상 표기, 1.3은 다른 방식) |
| Length | 2 바이트 | fragment 길이 |
| Fragment | 가변 | 실제 데이터(평문 → 압축 → MAC/암호화) |

TLS 1.3에서는 기록 계층이 AEAD 기반으로 바뀌면서:

- 각 레코드마다 **고유 nonce**(IV)를 사용해 재사용 공격을 막고,
- content type도 암호화 내부의 “inner” 타입으로 감싼 다음, 외부 ContentType은 `application_data`로 통일하는 방식을 쓴다.

#### 예: 오버헤드 계산

평문 길이가 $$L$$ 바이트, AEAD 태그 길이가 $$T$$ 바이트(예: 16),
레코드 헤더가 5 바이트라고 하면, **전송되는 총 바이트 수** $$L'$$ 은:

$$
L' = L + T + 5
$$

MTU 1500 바이트 환경에서,
애플리케이션 평문을 1400 바이트짜리 레코드 하나에 넣는다고 가정하면:

- 태그 16 + 헤더 5 = 21바이트 오버헤드
- 실제 IP 패킷 안에는 1421바이트의 TLS 레코드가 들어간다.

---

## 네 개의 TLS 상위 프로토콜 (Four Protocols)

IETF 문서를 보면, Record Protocol 위에서 다음 네 가지 **content type**이 정의되어 있다.

1. **Handshake Protocol**
2. **Alert Protocol**
3. **ChangeCipherSpec (CCS) Protocol**
4. **Application Data Protocol**

(TLS 1.3에서는 CCS의 역할이 거의 사라졌고, 새로운 확장도 추가되지만 **기본 네 가지 틀**은 여전히 교육용으로 중요하다.)

---

## Handshake Protocol

### 목적

Handshake는 한 문장으로 요약하면:

> “클라이언트와 서버가 **서로를 인증**하고, 안전하게 **공유 비밀 키**(session key)를 합의하는 과정”

핵심 목표:

- 사용할 **프로토콜 버전 / cipher suite / 확장**을 협상
- 서버(필요하면 클라이언트도) 인증서로 **상대방 신원 검증**
- ECDHE 같은 키 교환으로 **공유 비밀 키** 생성
- 이 키를 바탕으로 이후 레코드를 암호화

### 전통적 TLS 1.2 핸드셰이크 흐름

가장 전형적인 RSA 인증 + ECDHE 키 교환 예를 보자(단순화).

```text
Client                     Server
  |---- ClientHello -----> |
  |                        |
  | <--- ServerHello ----- |
  | <--- Certificate ----- |
  | <--- ServerKeyExchange|
  | <--- ServerHelloDone  |
  |                        |
  |---- ClientKeyExchange->|
  |---- ChangeCipherSpec ->|
  |---- Finished --------->|
  |                        |
  | <--- ChangeCipherSpec -|
  | <--- Finished -------- |
  |                        |
 Application Data <======> Application Data
```

핵심 메시지:

- **ClientHello**:
  - 클라이언트 랜덤 값
  - 지원하는 cipher suite 목록
  - TLS 버전, 확장(SNI, ALPN 등)
- **ServerHello**:
  - 서버 선택 cipher suite
  - 서버 랜덤 값
- **Certificate**: 서버 인증서 체인
- **ServerKeyExchange / ClientKeyExchange**:
  - (ECDHE의 경우) 공개키 값을 주고받아 **공유 비밀** 생성
- **ChangeCipherSpec**:
  - “이제부터 합의된 키로 암호화해서 보내겠다”는 신호
- **Finished**:
  - 지금까지의 모든 핸드셰이크 메시지에 대해 MAC/해시를 계산한 값
  - 이 Finished가 성공적으로 검증되면 “서로 같은 키를 가졌고, 중간에서 조작된 적이 없다”는 것을 확인

### TLS 1.3에서의 간소화

TLS 1.3은 RT(T) 수를 줄이고, 불필요한 메시지를 제거했다.

- 기본 핸드셰이크는 **1-RTT**, 재접속은 **0-RTT** 까지 가능
- `ChangeCipherSpec` 실질적으로 폐지(형식적 backward compat 정도만 남음)
- `ClientKeyExchange` / `ServerKeyExchange` 등은 사라지고, `KeyShare` 확장에 통합

---

## ChangeCipherSpec Protocol

### 역할과 구조

**ChangeCipherSpec(이하 CCS)** 는 내용이 매우 단순한 프로토콜이다.

- 실제 payload는 **1바이트** 값(보통 `0x01`) 뿐이다.
- 의미:
  “**이제부터 이 방향의 레코드들은 새로 합의한 키/알고리즘으로 암호화할 것이다**”

TLS 1.2에서:

- 클라이언트가 `ChangeCipherSpec`을 보내면, 그 이후부터 보내는 레코드는 **새 세션 키**로 암호화.
- 서버도 마찬가지.

### TLS 1.3에서의 변화

TLS 1.3에서 CCS는 **기능적으로 필요 없**지만, 중간 장비/구형 스택과의 호환성 때문에 **사실상 “dummy”** 로만 남아 있다.

- 실질적인 “키 전환”은 Handshake 메시지(EncryptedExtensions, Finished 등)를 교환하는 과정에서 암호학적으로 결정된다.
- 그러나 오래된 방화벽/중간 프록시가 “핸드셰이크 중에 CCS가 보일 것”을 기대하는 경우가 있어서, **호환성용**으로 제한적으로 사용되기도 한다.

---

## Alert Protocol

### 목적

Alert Protocol은 TLS 세션 중 발생하는 **각종 오류·경고·세션 종료** 상태를 전달하는 데 사용된다.

- 심각한 오류: 즉시 연결 종료
- 단순 경고: 복구 가능한 문제, 또는 정보 전달

### 필드 구조

Alert 메시지는 두 개 필드를 가진다.

| 필드 | 설명 |
|------|------|
| Level | `warning` 또는 `fatal` |
| Description | 구체적인 에러 코드 (예: `bad_record_mac`, `handshake_failure`, `close_notify` 등) |

예: 서버 인증서 검증 실패

- 클라이언트가 서버 인증서를 검증하다가 **이름 불일치** 또는 **신뢰할 수 없는 CA**를 발견하면,
- `alert (level=fatal, description=bad_certificate)` 를 보내고,
- 바로 TLS 연결을 끊는다.

### `close_notify` 예시

애플리케이션이 정상적으로 연결을 종료하고 싶을 때:

1. 송신측 TLS 스택은 마지막 데이터를 암호화된 Application Data로 보내고,
2. 그 다음 레코드로 `alert (level=warning, description=close_notify)` 를 보낸다.
3. 수신측은 이 알림을 받고 “상대가 정상 종료했구나” 라고 알고, 자신도 `close_notify` 를 보내거나 TCP를 닫는다.

이 메커니즘을 쓰면, **패딩 오라클**이나 **중간자 공격** 등의 일부 정보를 줄이는 데도 도움이 된다(명시적인 종료 시점).

---

## Application Data Protocol

### 하는 일

Application Data는 말 그대로 **애플리케이션 레벨의 Payload** 를 담는 content type이다.

- HTTP, SMTP, IMAP, MQTT, gRPC, PostgreSQL 프로토콜 등 모든 상위 프로토콜이 이 안에 들어간다.
- TLS 입장에서는 “그냥 바이트 스트림”이다. 내용이 HTTP인지, DB 프로토콜인지 구분하지 않는다.

### 레코드 단위 분할과 재조립

- 애플리케이션은 `send()` 호출로 수십 KB를 한꺼번에 보낼 수 있지만,
- TLS 레코드는 **최대 길이 제한**(보통 16KB 수준)을 가진다.
- 따라서, 한 번의 `write()` 호출이 여러 개의 TLS 레코드로 쪼개질 수 있다.

반대로 수신 측에서는:

- 여러 TLS 레코드에서 평문을 꺼내서 하나의 애플리케이션 버퍼로 이어 붙이고,
- 애플리케이션에선 마치 **연속된 바이트 스트림**처럼 읽는다.

#### 예: HTTP 요청이 레코드 몇 개에 나뉘는 경우

1. 브라우저가 20KB짜리 HTTP POST 요청을 보낸다.
2. TLS 스택은 이를 16KB + 4KB 두 개의 레코드로 나누어 암호화하여 전송한다.
3. 서버 측 TLS 스택은 레코드 두 개를 복호화해 20KB의 평문을 합쳐서 HTTP 서버에 넘긴다.

---

## 실습 예제 1 — Python으로 TLS 클라이언트 작성하기

아래 코드는 Python 표준 라이브러리 `ssl`과 `socket`을 이용해 간단한 TLS 클라이언트를 만드는 예이다.

```python
import socket
import ssl

hostname = "example.com"
port = 443

# 평범한 TCP 소켓 생성

sock = socket.create_connection((hostname, port))

# TLS 컨텍스트 생성 (최신 보안 수준 권장)

context = ssl.create_default_context()
# 필요하다면 TLS 1.2/1.3 강제, 약한 cipher 제외 등 추가 설정 가능

# TCP 소켓을 TLS로 감싼다 (핸드셰이크 수행)

tls_sock = context.wrap_socket(sock, server_hostname=hostname)

print("협상된 프로토콜 버전:", tls_sock.version())
print("협상된 Cipher Suite:", tls_sock.cipher())

# HTTPS 요청 전송 (Application Data 레코드로 전송됨)

request = b"GET / HTTP/1.1\r\nHost: " + hostname.encode() + b"\r\nConnection: close\r\n\r\n"
tls_sock.sendall(request)

# 응답 읽기

response = tls_sock.recv(4096)
print(response.decode(errors="ignore"))

tls_sock.close()
```

이 코드에서 내부적으로 일어나는 일:

1. `wrap_socket()` 호출 시점에 **Handshake Protocol** 이 동작한다.
2. 합의된 키가 준비되면, 이후의 `sendall()` 호출은 **Application Data** 레코드를 통해 암호화되어 나간다.
3. 서버가 연결을 정상 종료하면, 마지막에 `close_notify` Alert를 보내고 TCP를 닫는다.

---

## 실습 예제 2 — OpenSSL로 핸드셰이크/알림 살펴보기

리눅스/맥 환경이라면 다음과 같이 `openssl`을 사용해 실제 TLS 핸드셰이크를 직접 볼 수 있다.

```bash
# 서버와 TLS 핸드셰이크를 수행하고, 핸드셰이크 메시지들을 출력

openssl s_client -connect example.com:443 -msg -state
```

출력에 나타나는 것들:

- `>>> TLS 1.3 Handshake [ClientHello]`
- `<<< TLS 1.3 Handshake [ServerHello]`
- `<<< TLS 1.3 Handshake [EncryptedExtensions]`
- `<<< TLS 1.3 Handshake [Certificate]`
- `<<< TLS 1.3 Handshake [CertificateVerify]`
- `<<< TLS 1.3 Handshake [Finished]`
- `>>> TLS 1.3 Handshake [Finished]`

또는 오류 발생 시:

- `<<< TLS 1.2 Alert [fatal] bad_certificate` 와 같은 Alert를 볼 수 있다.

이런 식으로 실제 네트워크에서 TLS가 어떻게 동작하는지 **“와이어 레벨”** 을 관찰하는 것은, 이론을 몸으로 익히는 데 큰 도움이 된다.

---

## 현대 TLS 보안 설정 모범 사례 요약

마지막으로 SSL/TLS 아키텍처와 네 프로토콜을 이해했다면, 실제 서비스 구성 시 아래 원칙들을 기억해 두면 좋다.

1. **프로토콜 버전**
   - SSL 2.0 / 3.0, TLS 1.0 / 1.1 → **완전 차단** (심각한 취약점 + IETF/NIST/NSA 모두 비권고)
   - TLS 1.2 / 1.3 → 기본 지원, 신규 설계는 1.3 우선

2. **암호 알고리즘**
   - TLS 1.2에서도 더 이상 **RSA 키 교환, 정적 DH** 는 사용하지 말 것 (IETF에서 폐기 초안 진행 중)
   - 키 교환은 **ECDHE**, 대칭 암호는 **AES-GCM, CHACHA20-POLY1305** 같은 AEAD 권장

3. **인증서와 키 관리**
   - 충분히 긴 키(예: RSA 2048 이상 또는 ECDSA P-256 이상)를 사용
   - 신뢰할 수 있는 CA, 적절한 만료 기간, 최신 서명 알고리즘(SHA-256 이상)

4. **핸드셰이크/Alert 관찰**
   - 모니터링 시 Handshake 실패율, Alert 통계(예: `handshake_failure`, `bad_certificate`)를 살펴보면
     - misconfiguration, 공격, 클라이언트 호환성 문제를 파악하는 데 큰 도움이 된다.

이 정도를 이해하면, TLS의 **아키텍처**와 **네 개의 주요 프로토콜**(Handshake / ChangeCipherSpec / Alert / Application Data)이
실제 **HTTPS, VPN, 메일, 메시징** 등 다양한 애플리케이션의 보안을 어떻게 지탱하는지 전체 그림이 잡힐 것이다.
