---
layout: post
title: 데이터 통신 - Transmission Control Protocol (5)
date: 2024-08-30 22:20:23 +0900
category: DataCommunication
---
# — 서비스, 특징, 패킷 포맷 완전 정리

TCP/UDP에 이어 IETF가 설계한 **세 번째 전송 계층 프로토콜**이 바로 **SCTP**다.
원래는 **PSTN(전화망) 시그널링을 IP 위로 운반(SIGTRAN)** 하기 위해 설계되었지만,
지금은 **텔코 코어망, WebRTC 데이터 채널, Diameter, 로밍/모바일 코어** 등에서 널리 쓰인다.

이 글에서는 다음 세 가지를 중심으로 정리한다.

- SCTP services
- SCTP features
- Packet format

---

## SCTP Services

SCTP는 TCP/UDP를 “섞어서 강화한” 느낌의 서비스 집합을 제공한다.
핵심 키워드는 다음 네 가지다.

1. **신뢰성 (reliability)**
2. **메시지 지향 (message-oriented)**
3. **멀티스트리밍 (multistreaming)**
4. **멀티호밍 (multihoming)**

### 기본 서비스 개요

RFC 4960에서 정의하는 SCTP 서비스들은 다음과 같다.

- **신뢰적, 비중복 데이터 전송**
  - 손실 시 재전송, 순서 제어, 중복 제거.
- **메시지 경계 보존(message boundary preservation)**
  - 애플리케이션이 `메시지 1`, `메시지 2`로 보낸 것은
    상대도 같은 단위의 “메시지”로 받는다. (UDP와 비슷)
- **멀티스트리밍(multi-streaming)**
  - 하나의 SCTP association 안에 여러 **스트림(stream)** 을 구성.
  - 각 스트림은 독립적인 순서 번호를 가진다.
- **멀티호밍(multihoming)**
  - 한 endpoint가 **여러 IP 주소**를 가질 수 있다.
  - 한 경로가 끊어져도 다른 IP로 패킷을 보내 association 유지.
- **혼잡 제어, 흐름 제어**
  - TCP와 유사한 AIMD 기반 혼잡 제어, 윈도우 기반 흐름 제어.
- (확장) **부분 신뢰(Partial Reliability)**
  - 특정 메시지를 “기한 내 전송 실패하면 포기”할 수 있는 확장(RFC 3758 등).

#### 예시 상황 1 — SIGTRAN에서의 SCTP

전화 교환기(MSC, HLR, STP 등)는 기존에 **SS7**이라는 회선 교환 시그널링을 사용했다.
IP 코어망으로 옮기면서, 이 SS7 메시지를 IP 위에서 운반해야 했고,
이때 사용된 것이 **SCTP + SIGTRAN(M3UA, M2PA 등)** 이다.

- 시그널링 메시지는 **작지만 매우 중요**하고 손실되면 안 된다.
- 동시에 **여러 signaling link** 를 흉내 내야 한다 → multistreaming 유리.
- 코어망 링크 장애에도 통화가 끊기면 안 됨 → multihoming 필수.

이 특성들이 TCP/UDP보다 SCTP를 선택하게 만든 결정적 이유다.

#### 예시 상황 2 — WebRTC 데이터 채널에서의 SCTP

WebRTC의 **RTCDataChannel**은 내부적으로 **SCTP over DTLS over UDP** 구조를 사용한다.

- 브라우저 탭 간에 **채팅, 파일 전송, 게임 상태 동기화** 등 다양한 데이터를 주고받는다.
- 일부 메시지는 **순서 보장 + 재전송 필요**(채팅 메시지),
  일부는 **순서가 중요해도 손실 허용**(게임 위치),
  또 다른 일부는 **순서/신뢰 모두 덜 중요**할 수 있다.

SCTP는 **각 데이터 채널을 하나의 스트림**으로 매핑하고,
스트림별로 “정렬/비정렬, 신뢰/부분신뢰”를 선택할 수 있어 이런 요구에 잘 맞는다.

---

### 메시지 지향 서비스 (Message-oriented Service)

TCP는 바이트 스트림 프로토콜이기 때문에,
`send()`가 두 번 보내도 `recv()`는 여러 번에 나뉘어 받거나 합쳐 받을 수 있다.

**SCTP는 UDP처럼 “메시지 단위”를 유지한다.**

- 송신 애플리케이션이 `메시지 A(100바이트)`, `메시지 B(200바이트)`로 보내면,
- 수신 측에서 `recvmsg()` 등을 호출할 때 **A, B가 분리된 단위로 도착**한다.

이 특성은 **프로토콜 설계를 단순화**한다.

#### 간단 코드 느낌(리눅스에서의 SCTP recvmsg)

```c
// 가상적인 예시 – 실제로는 sctp_recvmsg 등 사용
char buf[2048];
struct sockaddr_storage peer;
socklen_t peerlen = sizeof(peer);
struct msghdr msg = {0};
struct iovec iov;

iov.iov_base = buf;
iov.iov_len  = sizeof(buf);
msg.msg_name    = &peer;
msg.msg_namelen = peerlen;
msg.msg_iov     = &iov;
msg.msg_iovlen  = 1;

ssize_t n = recvmsg(sctp_fd, &msg, 0);
// n 바이트는 정확히 "한 개의 SCTP user message"이다.
```

TCP라면 이 `recvmsg` 호출이 메시지 경계를 보장하지 않지만,
SCTP에서는 **한 번의 `recvmsg` = 한 개의 user message** 가 원칙이다(단, buffer가 너무 작으면 truncation 플래그로 알려준다).

---

### 멀티스트리밍(Multistreaming)

SCTP association은 여러 **스트림(stream)**을 가진다.

- 각 스트림은 **독립적인 순서 번호(SSN, Stream Sequence Number)**를 가진다.
- 한 스트림에서 패킷 손실/순서 꼬임이 생겨도, **다른 스트림은 영향 없이 계속 전송**된다.

이를 통해 TCP가 갖는 **HoL(Head-of-Line) blocking** 문제를 줄일 수 있다.

#### 예: HTTP/2 스타일 멀티 스트림을 SCTP로 구현

예를 들어 한 앱이 **로그 스트림, 메트릭 스트림, 설정 스트림**을
하나의 SCTP association 위에 올리고 싶다고 해보자.

- Stream 0: 로그 메시지
- Stream 1: 메트릭(주기적 숫자)
- Stream 2: 설정 업데이트

로그 스트림에서 하나의 세그먼트가 손실되어 재전송을 기다리는 동안에도,
메트릭 스트림은 **정상적으로 계속 도착**할 수 있다.

TCP였다면 로그 스트림에서 막힌 부분 때문에 이후 모든 바이트가 막혀 HoL이 발생했을 것이다.

---

### 멀티호밍(Multihoming)

SCTP association의 각 endpoint는 **IP 주소를 여러 개** 가질 수 있다.

- 예: 서버가 `203.0.113.10` 과 `198.51.100.10` 두 개의 NIC로 연결.
- 클라이언트 역시 Wi-Fi, 5G 두 경로를 가질 수 있다.

SCTP는 **primary path**를 하나 정해 데이터 전송에 사용하고,
정기적으로 **HEARTBEAT**를 보내 다른 경로의 상태를 체크한다.

- primary path 장애 → 다른 alive path로 패킷 전송 전환.
- association은 그대로 유지 (TCP처럼 끊어지지 않음).

이 특성은 **통신망 시그널링, 모바일 코어, 고가용성 시스템**에서 특히 중요하다.

---

### 기타 서비스들

SCTP는 다음과 같은 추가 서비스도 제공한다.

- **Path/Association Heartbeat**: 경로 상태 감시.
- **Selective Acknowledgment (SACK)**: 누락된 TSN(전송 시퀀스 번호)를 효율적으로 재전송.
- **Path MTU discovery / Fragmentation**: association별 경로 MTU를 고려한 조각화.
- **Partial Reliability (확장)**: 특정 메시지는 재전송 횟수/시간 제한을 두고, 넘으면 포기.

---

## SCTP Features

services를 “애플리케이션 관점”에서 본 것이라면,
features는 **프로토콜 설계 특징**에 가깝다.

여기서는 TCP와 비교했을 때 중요한 차이점들을 정리한다.

### 4-way Handshake와 쿠키 메커니즘

TCP는 3-way handshake(SYN → SYN/ACK → ACK)를 사용한다.
SCTP는 **4-way handshake**를 사용해 **SYN Flooding 공격에 강하도록 설계**되었다.

절차를 단순화하면 다음과 같다.

1. 클라이언트 → 서버: **INIT** chunk
   - 자신이 지원하는 스트림 수, 초기 TSN 등 제안.
2. 서버 → 클라이언트: **INIT-ACK** + **State Cookie**
   - 서버는 아직 state를 만들지 않고,
     쿠키 안에 state 정보를 암호학적으로 인코딩해서 클라이언트에게 돌려준다.
3. 클라이언트 → 서버: **COOKIE-ECHO**
   - 클라이언트가 받은 쿠키를 그대로 돌려 보냄.
4. 서버 → 클라이언트: **COOKIE-ACK**
   - 서버는 쿠키를 검증하고 실제 association state를 만든 후 ACK.

TCP는 SYN Flooding에서 half-open 상태가 많이 쌓여 문제지만,
SCTP는 **쿠키를 돌려받기 전까지 state를 만들지 않기 때문에** 공격에 훨씬 강하다.

---

### 멀티스트리밍과 TSN vs SSN

SCTP에는 두 종류의 번호가 있다.

- **TSN (Transmission Sequence Number)**
  - association 전체에 대해 단조 증가하는 번호.
  - 신뢰성/재전송 제어, SACK 등에서 사용.
- **SSN (Stream Sequence Number)**
  - 각 스트림마다 별도로 증가.
  - 스트림 내부의 순서 제어에 사용.

즉, 재전송/혼잡 제어는 TSN 기준으로 하고,
애플리케이션이 보는 **“이 스트림에서의 순서”**는 SSN으로 관리한다.

이를 통해 **association 전체가 막히지 않고, 문제 있는 스트림만 영향**을 받게 한다.

---

### 멀티호밍과 Path 관리

SCTP는 각 endpoint의 IP 주소를 **주소 리스트**로 관리하고,
각 경로마다 별도의 상태를 가진다.

- 각 경로별로
  - Path 상태 (Active/Inactive)
  - Path-specific error count
  - Path-specific RTT, RTO
- HEARTBEAT / HEARTBEAT-ACK chunk를 주고받으면서 경로 상태를 모니터링.

예:

1. 서버 주소 집합: `{203.0.113.10, 198.51.100.10}`
2. primary path: `203.0.113.10` (메인 전송 경로)
3. backup path: `198.51.100.10` (대기 경로)

primary path에 대해 HEARTBEAT에 응답이 없는 상태가 계속되면,
error count가 threshold 넘어서면 **path down** 처리, 이후 backup path로 데이터 전송을 전환한다.

---

### 데이터/제어 분리: Chunk 기반 설계

TCP는 헤더 하나에 제어 정보 + 데이터가 들어있다.
SCTP는 **“패킷 = 공통 헤더 + 여러 개의 chunk”**로 구성된다.

- 하나의 SCTP 패킷 안에
  - DATA chunk 여러 개
  - SACK, HEARTBEAT, SHUTDOWN 같은 control chunk 여러 개
  를 **번들링(bundling)** 해서 보낼 수 있다.
- control chunk는 항상 data chunk보다 앞에 온다.

이 chunk 기반 설계 덕분에,
**성능과 확장성** 측면에서 TCP보다 훨씬 유연하게 옵션/확장을 넣을 수 있다.

---

### Selective Acknowledgment와 혼잡 제어

SCTP는 기본적으로 TCP Reno와 유사한 혼잡 제어를 제공하며,
**SACK 스타일의 선택적 ACK**를 mandatory하게 사용한다.

- 수신자는 SACK chunk에 “어떤 TSN 범위들이 도착했고, 어떤 TSN이 비어 있는지”를 싣는다.
- 송신자는 누락된 TSN만 선택적으로 재전송.

이를 바탕으로 CUBIC, BBR 등 다양한 혼잡 제어 알고리즘도 SCTP에 이식할 수 있다(구현 의존).

---

### 예시: WebRTC에서 SCTP 기능 활용

WebRTC 데이터 채널에서 SCTP의 기능이 어떻게 활용되는지 요약해보면:

- 각 data channel → SCTP stream 하나(혹은 둘 이상)로 매핑.
- 메시지마다
  - Ordered / Unordered
  - Reliable / Partially Reliable
  를 선택.
- DTLS 위에 SCTP를 올려 암호화 및 인증 제공.
- NAT/방화벽 문제 해결을 위해 UDP/ICE 위에 올림.

즉, SCTP의 **멀티 스트림 + 메시지 지향 + 부분 신뢰** 특징 덕분에
하나의 association으로 다양한 “성격”의 데이터를 동시에 다루는 것이 가능해진다.

---

## SCTP Packet Format

이제 SCTP의 실제 **패킷 포맷**을 상세하게 보자.

SCTP 패킷은 항상

1. **공통 헤더 (12바이트)**
2. **하나 이상의 chunk**

로 이루어진다.

### 공통 헤더(Common Header)

공통 헤더는 다음 4개 필드로 구성된다(각 32비트 단위).

| 필드            | 길이(비트) | 설명 |
|-----------------|-----------|------|
| Source Port     | 16        | 송신 SCTP 포트 번호 |
| Destination Port| 16        | 수신 SCTP 포트 번호 |
| Verification Tag| 32        | 연결(association) 검증용 태그 |
| Checksum        | 32        | CRC32c 기반 체크섬 |

간단한 비트 배치 도식은 다음과 같다.

```text
0                   15 16                  31
+---------------------+---------------------+
|     Source Port     |   Destination Port  |
+---------------------+---------------------+
|                  Verification Tag         |
+-------------------------------------------+
|                  Checksum                 |
+-------------------------------------------+
|                 Chunk #1 ...              |
```

#### Verification Tag

- 각 방향마다 독립적인 **32bit 랜덤 값**을 사용한다.
- association 설정 시 INIT/INIT-ACK 교환 과정에서 합의된다.
- 수신자는 Verification Tag가 현재 association의 기대 값과 맞는지 확인해,
  이전 연결에서 떠돌던 패킷을 걸러낸다.

#### Checksum

- SCTP는 **CRC32c**(Castagnoli polynomial)를 사용한다.
- TCP/UDP의 16비트 1의 보수 체크섬보다 오류 검출 성능이 높다.
- 계산 시, 해당 필드를 0으로 놓고 CRC32c를 계산한 뒤 그 값을 채운다.

---

### Chunk 공통 형식

공통 헤더 뒤에는 하나 이상의 chunk가 연속해서 붙는다. 각 chunk는 다음과 같은 공통 형식을 가진다.

```text
0                   7 8                  15 16                 31
+---------------------+---------------------+-------------------+
|   Chunk Type        |   Chunk Flags       |   Chunk Length    |
+---------------------+---------------------+-------------------+
|                   Chunk Value (Data / Control)               |
|                               ...                             |
+--------------------------------------------------------------+
(필요시 32비트 경계까지 0 패딩)
```

- **Chunk Type (8비트)**
  - chunk의 종류를 나타내는 코드.
  - 예: DATA(0), INIT(1), INIT-ACK(2), SACK(3), HEARTBEAT(4) 등.
- **Chunk Flags (8비트)**
  - 각 chunk 타입마다 별도의 의미.
  - 예: DATA chunk의 U,B,E 비트(Unordered, Beginning, Ending) 등.
- **Chunk Length (16비트)**
  - 이 chunk 전체 길이(헤더 + value). 패딩은 포함되지 않는다.
- **Chunk Value**
  - 실제 데이터 혹은 제어 정보.

모든 chunk는 **4바이트 경계로 패딩**되어야 한다.
즉, `Chunk Length`가 4의 배수가 아니면 뒤를 0으로 채워 4의 배수가 되게 한다.

---

### 주요 Chunk 종류와 예시

RFC 4960 및 이후 확장에서 정의된 chunk 타입 중 핵심적인 것들만 정리하면:

| Type 값 | 이름         | 용도 |
|---------|--------------|------|
| 0       | DATA         | user data 전송 |
| 1       | INIT         | association 설정 시작 |
| 2       | INIT-ACK     | association 설정 응답 + 쿠키 전달 |
| 3       | SACK         | 선택적 ACK, 누락 TSN 정보 전달 |
| 4       | HEARTBEAT    | 경로 상태 확인(keepalive) |
| 5       | HEARTBEAT-ACK| HEARTBEAT 응답 |
| 6       | ABORT        | 비정상 종료 |
| 7       | SHUTDOWN     | 우아한 종료 시작 |
| 8       | SHUTDOWN-ACK | 종료에 대한 응답 |
| 9       | ERROR        | 오류 리포트 |
| 10      | COOKIE-ECHO  | 서버에 쿠키 되돌려 보내기 |
| 11      | COOKIE-ACK   | 쿠키 검증 성공 응답 |

#### 예: 하나의 SCTP 패킷에 SACK + DATA 2개 번들링

```text
[Common Header]
[Chunk 1: SACK]
[Chunk 2: DATA (TSN=1001, Stream=0, SSN=1, Payload=...)]
[Chunk 3: DATA (TSN=1002, Stream=1, SSN=1, Payload=...)]
```

- 같은 association에 대해, 한 패킷 안에 제어 정보(SACK)와
  두 개의 데이터 스트림 메시지가 함께 실려 효율적인 전송이 가능하다.

---

### DATA Chunk 구조 예제

DATA chunk의 구조(핵심 필드만 단순화)

```text
0      7 8     15 16                 31
+--------+--------+-------------------+
|  Type  | Flags  |   Chunk Length    |
+--------+--------+-------------------+
|                      TSN            |
+-------------------------------------+
|      Stream Identifier (SID)        |
+-------------------------------------+
|   Stream Sequence Number (SSN)      |
+-------------------------------------+
|          Payload Protocol ID        |
+-------------------------------------+
|               User Data ...         |
+-------------------------------------+
```

- Flags 예:
  - U(Unordered): 순서 보장 없이 전달할지
  - B(Beginning): 메시지 시작 조각인지
  - E(Ending): 메시지 끝 조각인지
- TSN: 재전송/신뢰성 관리를 위한 association 전체 번호
- SID/SSN: 해당 스트림에서의 메시지 순서 관리
- Payload Protocol ID: 상위 프로토콜 식별(예: 무엇을 싣고 있는지)

---

### INIT / INIT-ACK Chunk 예시 (간단화)

INIT chunk는 association 설정을 위한 **제안 값**들을 담는다.

- Initiate Tag
- Advertised Receiver Window Credit (rwnd)
- Number of Outbound Streams
- Number of Inbound Streams
- Initial TSN
- Optional parameters (Supported Address Types 등)

INIT-ACK는 이에 대한 응답과 함께 **State Cookie** 를 담는다.

State Cookie는 서버의 내부 상태(예: TSN, 윈도우 등)를 암호학적으로 캡슐화한 값이며,
클라이언트가 COOKIE-ECHO로 돌려줌으로써 **서버가 실제 state를 만들 수 있게** 한다.

---

### 실제 트레이스에 가까운 예시 (tcpdump 스타일)

실제로 Wireshark/tcpdump에서 볼 수 있는 형식에 가깝게 표현해보면:

```text
IP 192.0.2.10.5400 > 198.51.100.20.2905: sctp (1) [INIT]
    Verification tag: 0x00000000
    INIT chunk:
        Initiate tag: 0x1a2b3c4d
        Advertised receiver window credit: 131072 (128KiB)
        Number of outbound streams: 10
        Number of inbound streams: 10
        Initial TSN: 123456789

IP 198.51.100.20.2905 > 192.0.2.10.5400: sctp (1) [INIT ACK]
    Verification tag: 0x1a2b3c4d
    INIT ACK chunk:
        Initiate tag: 0x55667788
        State Cookie: <...opaque...>
        ...
```

이후 COOKIE-ECHO, COOKIE-ACK를 거쳐 association이 성립한다.

---

### SCTP 소켓 사용 예제 (리눅스, 개념 코드)

리눅스에서는 `IPPROTO_SCTP` 및 `SOCK_SEQPACKET` 등을 사용해 SCTP 소켓을 열 수 있다
(커널/라이브러리 지원 필요).

```c
int fd = socket(AF_INET, SOCK_SEQPACKET, IPPROTO_SCTP);

struct sockaddr_in addr = {0};
addr.sin_family = AF_INET;
addr.sin_port   = htons(2905);
inet_pton(AF_INET, "0.0.0.0", &addr.sin_addr);

bind(fd, (struct sockaddr*)&addr, sizeof(addr));
listen(fd, 5);

int sctp_assoc_fd = accept(fd, NULL, NULL);

// 스트림 0으로 메시지 전송 (실제론 sctp_sendmsg 등 사용)
const char* msg = "hello over SCTP stream 0";
send(sctp_assoc_fd, msg, strlen(msg), 0);
```

실제 구현에서는 `sctp_sendmsg`, `sctp_recvmsg`, `setsockopt`(SCTP_INITMSG, SCTP_EVENTS 등)를 사용해
스트림 수, 이벤트 구독, 멀티호밍 주소 등을 설정한다.

---

## 정리

- **SCTP services**
  - 신뢰성 + 메시지 경계 유지 + 멀티스트리밍 + 멀티호밍 + 혼잡/흐름 제어 + 확장 가능한 부분 신뢰.
- **SCTP features**
  - 4-way handshake + 쿠키 메커니즘으로 SYN Flooding 방어,
  - TSN/SSN 이중 시퀀스,
  - multihoming 기반 고가용성,
  - chunk 기반 구조로 제어/데이터를 유연하게 번들링.
- **Packet format**
  - 12바이트 공통 헤더(포트, Verification Tag, CRC32c checksum) 뒤에
    4바이트 정렬된 chunk들이 연속해서 붙는다.
  - DATA/INIT/SACK/HEARTBEAT/SHUTDOWN/COOKIE 류 chunk들이 다양한 제어·데이터 기능을 담당.

이 설계 덕분에 SCTP는 **텔코 시그널링, WebRTC 데이터 채널, 고가용성 코어망** 등 TCP/UDP로는 구현하기 까다로운 시나리오에서 강력한 선택지가 된다.
