---
layout: post
title: 데이터 통신 - Multimedia (4)
date: 2024-09-19 23:20:23 +0900
category: DataCommunication
---
# 실시간 대화형 멀티미디어 프로토콜 — RTP, RTCP, SIP, H.323 (28.4 Real-Time Interactive Protocols)

## Rationale for New Protocols (왜 새로운 실시간 프로토콜이 필요한가)

### 스트리밍 vs 실시간 대화형

앞선 “스트리밍 멀티미디어(저장/실시간 방송)”에서는 **지연(latency)** 이 조금 커도 상관없는 시나리오가 많았습니다.
예를 들어 YouTube VOD나 라이브 방송은 **수 초 단위 버퍼링**이 허용됩니다. 패킷이 조금 늦거나 재전송돼도 사용자는 “그냥 조금 늦네” 정도로 느낄 뿐입니다.

하지만 **실시간 대화형 멀티미디어(Real-Time Interactive)** — 예를 들어:

- 화상 회의 (Zoom, Teams, Webex)
- 인터넷 전화(VoIP)
- 온라인 게임의 음성 채팅
- 원격 음악 합주, 원격 수술 방제 센터 협업

같은 경우는 상황이 완전히 다릅니다.

- 한 방향 지연이 **150 ms 이하**일 때 “대화가 자연스럽다”고 평가하는 것이 일반적인 지침입니다(ITU-T G.114에서 권고하는 음성 대화 품질 기준에 기반).
- 지연이 **400 ms**를 넘어가면 서로 말이 겹치고, 회의의 품질이 크게 떨어집니다.

즉, **“조금 늦어도 괜찮다”**가 아니라 **“지금 바로 들려야 한다”**가 핵심입니다.

이런 특성 때문에, 기존의 **HTTP/TCP 중심 인터넷**만으로는 한계가 드러났습니다:

- **TCP의 재전송 메커니즘**은 손실에는 강하지만, 재전송이 끝날 때까지 상위 계층에 데이터를 올리지 않습니다(Head-of-Line Blocking).
- 결과: 패킷 하나 손실 → 그 뒤 패킷들도 모두 대기 → 음성이 “뚝뚝 끊기는” 느낌.

따라서 실시간 대화형 애플리케이션은 대개:

- **손실은 조금 나도 된다.**
- 대신 **지연과 지터(jitter)** 를 최대한 낮추고,
- 애플리케이션 레벨에서 **부드러운 보간(PLC, Packet Loss Concealment)**, FEC(Forward Error Correction), 중복 전송 등으로 품질을 유지하는 방향을 택합니다.

이 철학을 반영한 것이 **RTP/RTCP (미디어 전송)** 와 **SIP, H.323 (세션 제어/신호)** 입니다.

---

### 지연·지터·손실 예산

실시간 대화형 시스템의 **엔드-투-엔드 지연**은 대략 다음과 같이 쪼갤 수 있습니다.

$$
T_{\text{end-to-end}}
=
T_{\text{encode}}
+
T_{\text{packetization}}
+
T_{\text{network}}
+
T_{\text{jitter\ buffer}}
+
T_{\text{decode}}
$$

- \(T_{\text{encode}}\): 코덱 인코딩 지연 (예: Opus, G.711 등)
- \(T_{\text{packetization}}\): 샘플을 모아서 하나의 RTP 패킷으로 만드는 데 필요한 시간(예: 20 ms 프레임)
- \(T_{\text{network}}\): 네트워크 전송 지연(왕복이 아니라 한 방향)
- \(T_{\text{jitter buffer}}\): 패킷 도착 시각 변동을 흡수하기 위한 지터 버퍼 지연
- \(T_{\text{decode}}\): 디코더·플레이백 지연

실전에서는:

- Opus 같은 코덱은 **20 ms** 단위 프레임, 네트워크 + 지터 버퍼를 합쳐 **50~100 ms** 정도로 유지하고, 전체를 150~200 ms 이하로 맞추도록 설계합니다.

이런 **시간 제약 조건**을 만족하기 위해:

- **UDP + 애플리케이션 레벨 순서·타임스탬프** → RTP
- **품질 모니터링/피드백** → RTCP
- **세션 설정·종료·참가자 관리** → SIP, H.323

이렇게 역할이 나뉘게 되었습니다.

---

### 설계 원칙 요약

실시간 대화형 프로토콜들은 다음 철학을 공유합니다.

1. **“최대한 빠르게 전달하자”**
   - 가급적 **UDP** 사용, 재전송 최소화.
2. **“패킷 손실은 애플리케이션이 처리한다”**
   - 코덱(예: Opus)의 PLC, FEC, 중복 패킷, 적응적 비트레이트 등으로 품질 보정.
3. **“전송과 신호를 분리한다”**
   - RTP/RTCP: **미디어 전송**
   - SIP/H.323: **세션 제어, 주소/위치 해석, 참가자 관리**
4. **“멀티 파티와 다양한 토폴로지 지원”**
   - 멀티캐스트, SFU/MCU, 브리지, 게이트웨이 호환 등.

이제 각 프로토콜을 자세히 본 뒤, 어떻게 함께 작동하는지 살펴보겠습니다.

---

## RTP (Real-time Transport Protocol)

### 정의와 위치

RTP는 IETF의 **RFC 3550**에서 정의된 실시간 전송 프로토콜입니다.

- 주 용도: **오디오·비디오 같은 실시간 데이터**의 전송.
- 일반적으로 **UDP 위에서 동작**하지만, 이론상 다른 전송 계층도 가능.
- 제공 기능:
  - **Payload Type 식별**: 어떤 코덱으로 인코딩된 페이로드인지 구분
  - **Sequence Number**: 패킷 손실/순서 뒤바뀜 감지
  - **Timestamp**: 재생 시점 결정, 지터 버퍼 운용
  - **SSRC/CSRC**: 동기화 소스(Sync Source) 및 혼합(Mix)된 기여 소스 식별

중요한 점: **RTP는 QoS를 보장하지 않습니다.**
RFC 3550는 “RTP는 리소스 예약을 다루지 않으며 QoS를 보장하지 않는다”고 명시합니다.
대신, **실시간 데이터에 필요한 메타데이터를 제공**하고, 이를 기반으로 애플리케이션이 품질 제어를 수행하도록 합니다.

오늘날 브라우저 기반 실시간 통신(WebRTC)에서도 **RTP/SRTP**가 기본 미디어 전송 프로토콜로 사용됩니다.

---

### RTP 헤더 구조

RTP 고정 헤더는 최소 12바이트이며, 그 뒤에 옵션 필드와 페이로드가 옵니다.

#### 비트 필드 다이어그램

```
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|X|  CC   |M|     PT      |       sequence number         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           synchronization source (SSRC) identifier            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            contributing source (CSRC) identifiers ...         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **V (Version, 2 bits)**: 현재 버전은 2.
- **P (Padding)**: 패딩 여부.
- **X (Extension)**: 확장 헤더 존재 여부.
- **CC (CSRC Count)**: 뒤에 오는 CSRC 식별자 개수.
- **M (Marker)**: 코덱/프로파일에 따라 의미를 정하는 마커 비트(예: 프레임 경계).
- **PT (Payload Type)**: 코덱/포맷을 나타내는 번호 (정적/동적 할당).
- **Sequence Number**: 패킷마다 1씩 증가하는 16비트 정수.
- **Timestamp**: 샘플링 시각 기준 상대 타임스탬프.
- **SSRC**: 동기화 소스 식별자.
- **CSRC List**: 믹서가 여러 소스를 합쳤을 때 각 기여 소스 ID를 나열.

#### Sequence Number와 손실 검출

- 수신자는 이전에 받은 시퀀스 번호와 현재 패킷의 번호를 비교하여 **패킷 손실**을 추정합니다.
- 예: `1000, 1001, 1003` 을 순서대로 받으면, 1002번 패킷이 손실된 것으로 추정.

#### Timestamp와 지터 버퍼

- 타임스탬프는 “녹음/캡처된 시간 기준”으로 증가합니다.
  - 예: 샘플링 주파수 8 kHz, 20 ms 프레임이라면 각 패킷은 샘플 160개를 포함 → 타임스탬프는 160씩 증가.
- 수신 측은 타임스탬프를 사용해 **지터 버퍼**에서 패킷을 재정렬하고 균일한 재생 간격을 유지합니다.

---

### 예제 시나리오: 간단 VoIP 통화의 RTP 흐름

#### 상황 설정

- 두 단말 A와 B가 Opus 코덱으로 음성 통화.
- 프레임 길이: 20 ms
- 샘플링: 48 kHz
- RTP/UDP/IP

**보내는 쪽(A)의 흐름:**

1. 마이크에서 20 ms 동안 샘플을 모음 → 960 샘플.
2. Opus 인코더로 압축 → 예: 32 kbps → 약 80바이트 payload.
3. RTP 헤더를 붙여 포맷:
   - V=2, P=0, X=0
   - CC=0
   - M: 프레임 경계 표시(코덱/사용에 따라)
   - PT: Opus에 할당된 동적 Payload Type (예: 111)
   - Sequence: 이전 + 1
   - Timestamp: 이전 + 960
   - SSRC: 통화 내에서 유일한 값
4. UDP 포트(예: 5004)로 B에게 전송.

**받는 쪽(B)의 흐름:**

1. UDP 소켓으로 패킷을 수신.
2. Sequence Number를 확인하여 **손실/재정렬** 처리.
3. Timestamp를 기준으로 지터 버퍼에 저장.
4. 제때 버퍼에서 꺼내 Opus 디코더로 전달 후 재생.

간단한 의사 코드 예시는 다음과 같습니다.

```python
# 매우 단순화된 RTP 송신 루프 의사 코드

seq = 0
timestamp = 0
SSRC = random_32bit()

while call_active:
    # 20ms 오디오 캡처
    pcm = capture_20ms_audio()            # 960 samples @ 48kHz
    payload = opus_encode(pcm)            # 압축

    rtp_header = build_rtp_header(
        version=2, padding=0, extension=0,
        cc=0, marker=0, payload_type=111,
        sequence_number=seq,
        timestamp=timestamp,
        ssrc=SSRC
    )
    packet = rtp_header + payload
    udp_socket.sendto(packet, (peer_ip, 5004))

    seq = (seq + 1) & 0xFFFF
    timestamp = (timestamp + 960) & 0xFFFFFFFF
```

실제 구현에서는 SRTP, FEC, RED(Redundant Audio Data) 등 더 많은 기능이 추가됩니다.

---

### RTP와 코덱, 프로파일, WebRTC 맥락

RTP는 **코덱에 중립적**이며, 실제 페이로드 포맷은 각 코덱별 RFC에서 정의합니다.

- 예: **Opus** 오디오 코덱은 RFC 6716에 정의되어 있고, RTP 페이로드 포맷은 RFC 7587에 정의됩니다.
- WebRTC에서는 Opus가 **사실상 표준 음성 코덱**으로 사용됩니다.

또한 RTP에는 **프로파일(Profile)** 개념이 있어, 특정 환경(예: 오디오/비디오 회의)에 맞게 PT 번호와 헤더 확장, 제어 규칙 등을 묶어 정의합니다.

WebRTC에서는 RFC 8834에 따라 RTP의 사용 방식(필수/권장 확장, RTX, FEC 등)이 상세히 규정되어 있습니다.

---

## RTCP (RTP Control Protocol)

### 역할과 기본 개념

RTCP는 RTP와 **쌍으로 동작하는 제어 프로토콜**입니다.

- RTP는 **미디어 데이터**를 보냅니다.
- RTCP는 **통계와 제어 정보**를 보냅니다.

RTCP의 주요 목적:

1. **QoS 모니터링**
   - 패킷 손실률, 지터, RTT 등을 측정.
2. **참가자 정보 교환**
   - CNAME, 사용자 이름, 이메일 등 SDES 항목.
3. **세션 제어**
   - 참가 종료(BYE), 애플리케이션 정의 제어(APP) 등.
4. **대역폭 제어**
   - 전체 세션 대역폭의 약 5%만 RTCP에 사용되도록 설계.

실무적으로는 RTP 포트가 **짝수 번호**, RTCP 포트는 **그 다음 홀수 번호**를 사용하는 것이 일반적입니다(예: RTP: 5004, RTCP: 5005).

---

### RTCP 패킷 유형

RTCP에는 여러 타입의 패킷이 있습니다.

- **SR (Sender Report)**
  - RTP 패킷을 보내는 송신자가 송신 통계와 NTP 타임스탬프, RTP 타임스탬프를 보고.
- **RR (Receiver Report)**
  - 수신자가 받은 패킷에 대한 통계(손실률, 지터 등)를 보고.
- **SDES (Source Description)**
  - CNAME, NAME, EMAIL, TOOL 등 소스의 메타데이터.
- **BYE**
  - 세션을 떠날 때 사용.
- **APP**
  - 애플리케이션 정의 패킷.

예를 들어, 수신자가 보내는 RR에는 다음과 같은 필드가 들어갑니다.

- fraction lost (손실률)
- cumulative number of packets lost
- extended highest sequence number received
- interarrival jitter
- LSR, DLSR (RTT 측정용)

송신자는 이 정보를 바탕으로:

- 비트레이트 감소
- 코덱 모드 변경(예: Opus의 비트레이트 다운스케일)
- 해상도·프레임레이트 조절

등의 적응적 제어를 수행합니다.

---

### 예제: RTCP를 이용한 적응 스트림

**상황:**

- 화상 회의 SFU(Server-side Forwarding Unit)가 다수 참여자에게 비디오를 전송.
- 각 참여자는 주기적으로 RTCP RR을 SFU에 보냄.
- 특정 참여자의 손실률이 10% 이상으로 증가하고, RTT가 300 ms 수준으로 커졌다고 가정.

**동작:**

1. SFU는 해당 참여자의 RTCP RR에서 높은 손실률과 지터를 감지.
2. 해당 참여자에게 전송하는 비디오 스트림의:
   - 해상도를 1080p → 720p → 480p로 단계적으로 낮추거나,
   - 프레임레이트를 30 fps → 15 fps로 낮춤.
3. 손실률이 회복되면 다시 해상도/프레임을 상향.

이런 식으로 **RTCP는 실시간 QoS 피드백 채널**로 사용됩니다.

---

## Session Initiation Protocol (SIP) — Session Initialization Protocol

### 개요와 역할

SIP(Session Initiation Protocol)는 IETF의 RFC 3261에서 정의된 **애플리케이션 계층 시그널링 프로토콜**입니다.

- 기능:
  - 세션 **생성(Create)**, **변경(Modify)**, **종료(Terminate)**
  - 참여자 초대, 벨 울림, 연결, 전환, 종료에 해당하는 상태 머신 제공.
- 세션 유형:
  - 인터넷 전화(VoIP)
  - 멀티미디어 회의
  - 스트림 배포 등

SIP는 **미디어 자체를 운반하지 않으며**, 대신:

- **RTP/RTCP** 같은 미디어 프로토콜이 사용할:
  - IP 주소, 포트
  - 코덱 목록
  - 암호화 파라미터
를 협상하기 위한 **SDP(Session Description Protocol)를 담은 메시지**를 주고받습니다.

---

### SIP 엔티티 구조

SIP 네트워크는 여러 엔티티로 구성됩니다.

- **UA (User Agent)**
  - UA Client(UAC): 요청을 보내는 쪽(예: 전화를 거는 단말).
  - UA Server(UAS): 요청을 받는 쪽(예: 전화를 받는 단말).
- **Proxy Server**
  - 요청을 다른 서버 또는 UA로 라우팅.
  - 정책(예: 인증, 로드 밸런싱) 적용 가능.
- **Registrar**
  - 사용자의 현재 위치(접속 IP/포트)를 등록하는 서버.
- **Redirect Server**
  - 요청을 실제 목적지 URI로 재지정.
- **Location Server**
  - 사용자의 위치 정보를 저장/조회.

SIP 주소는 보통 **URI 형태**를 사용합니다.

```text
sip:alice@example.com
sip:bob@voip.example.org:5060
```

---

### SIP 메시지 형식과 주요 메서드

SIP는 HTTP와 유사한 **텍스트 기반 프로토콜**입니다.

#### 요청 메시지 예제 (INVITE)

```text
INVITE sip:bob@example.com SIP/2.0
Via: SIP/2.0/UDP pc33.example.com;branch=z9hG4bK776asdhds
Max-Forwards: 70
From: Alice <sip:alice@example.com>;tag=1928301774
To: Bob <sip:bob@example.com>
Call-ID: a84b4c76e66710@pc33.example.com
CSeq: 314159 INVITE
Contact: <sip:alice@pc33.example.com>
Content-Type: application/sdp
Content-Length: 147

v=0
o=alice 2890844526 2890844526 IN IP4 pc33.example.com
s=VoIP Call
c=IN IP4 pc33.example.com
t=0 0
m=audio 49170 RTP/AVP 0 111
a=rtpmap:0 PCMU/8000
a=rtpmap:111 opus/48000/2
```

여기서:

- `m=audio 49170 RTP/AVP 0 111`
  → 49170 포트를 사용해 RTP/AVP 프로파일로 오디오 전송, Payload Type 0(PCMU)와 111(Opus)를 지원.
- 실제 통화에서는 수신 측이 자신이 지원하는 코덱을 선택합니다.

#### 주요 메서드

- `INVITE`: 세션 생성 요청.
- `ACK`: 최종 응답 확인.
- `BYE`: 세션 종료.
- `REGISTER`: 위치 등록(단말이 “나 여기 있다”고 알림).
- `OPTIONS`: 상대가 지원하는 기능 조회.
- `CANCEL`: 진행 중인 요청 취소.

---

### 예제: 단순 VoIP 통화 시나리오

**상황:**

- Alice가 `sip:alice@example.com`
- Bob이 `sip:bob@example.com`
- 둘 다 example.com 도메인의 SIP Proxy/Registrar를 사용.

**흐름 개요:**

1. Bob은 단말이 켜질 때 `REGISTER` 보내 현재 IP/포트 등록.
2. Alice가 Bob에게 전화를 걸기 위해 `INVITE sip:bob@example.com` 전송.
3. Proxy는 Location Server에서 Bob의 현재 위치를 조회한 뒤, 해당 UA로 INVITE 전달.
4. Bob UA는 `180 Ringing`, `200 OK` 응답.
5. Alice UA는 `ACK`를 보내고, SDP에 합의된 포트/코덱으로 RTP 스트림을 시작.
6. 통화 종료 시 한쪽이 `BYE`, 다른 쪽이 `200 OK` 응답.

이 전체 과정에서:

- **SIP**: 세션 제어 및 코덱 협상.
- **RTP/RTCP**: 실제 음성/영상 데이터와 품질 제어.

---

### 최신 동향: SIP와 WebRTC

오늘날 브라우저 기반 실시간 통신(WebRTC)은 브라우저 내부적으로:

- 미디어 전송에 **SRTP(Secure RTP)**,
- 코덱에 Opus, VP8/VP9/AV1 등을 사용하고,

시그널링 프로토콜로는:

- 직접 SIP을 사용하기도 하고(SIP over WebSocket),
- 애플리케이션별 맞춤 JSON/REST 기반 신호를 사용하기도 합니다.

핵심은 **“시그널링과 미디어는 분리되어 있다”**는 점이며, SIP는 여전히 많은 VoIP 시스템과 기업 통신 환경에서 표준으로 쓰이고 있습니다.

---

## H.323 — ITU-T 패킷 기반 멀티미디어 시스템

### 개요와 역사

H.323은 ITU-T에서 정의한 **패킷 기반 멀티미디어 통신 시스템** 권고안입니다.

- 초기 목표: LAN 상에서 화상 회의 제공.
- 시간이 지나면서:
  - VoIP, 기업 간 영상 회의 등으로 영역 확대.
  - 여러 차례 개정되어 현재 Version 8까지 존재.

H.323은 다음을 포함하는 **시스템 스펙**입니다.

- 콜 시그널링(설정/종료)
- 멀티미디어 전송 제어
- 대역폭 제어
- 보안, 부가 서비스 등

RTP를 미디어 전송에 사용하며, H.225, H.245 등의 ITU-T 프로토콜들을 조합하여 전체 시스템을 구성합니다.

---

### H.323 아키텍처 구성 요소

H.323 시스템에는 여러 네트워크 요소가 있습니다.

- **Terminal**
  - 사용자의 엔드포인트(소프트폰, 하드웨어 비디오폰 등).
- **Gateway**
  - H.323 네트워크와 PSTN/ISDN/다른 VoIP망(예: SIP) 사이를 중계.
- **Gatekeeper**
  - 주소 해석, 등록, 인증, 대역폭 제어.
  - H.323망의 “소프트 스위치” 역할.
- **Multipoint Control Unit (MCU)**
  - 다자간 회의를 위한 믹서/브리지.
- **Border Element**
  - 도메인 간 라우팅, 정책, 보안.

각 요소는 여러 프로토콜을 사용합니다.

- H.225.0 RAS: 등록/허가/상태(Registration/Admission/Status).
- H.225.0 Call Signaling: 콜 설정, Q.931 기반.
- H.245: 멀티미디어 능력 교환, 논리 채널 제어.
- RTP: 오디오/비디오 전송.

---

### H.323 프로토콜 스택

H.323 단말의 프로토콜 스택을 단순화하면 다음과 같습니다.

| 계층 | 프로토콜 예 |
|------|-------------|
| 애플리케이션 | H.323, H.245, H.225, RAS |
| 전송 | TCP(시그널링), UDP(RTP/RTCP) |
| 네트워크 | IP |
| 링크/물리 | Ethernet, Wi-Fi 등 |

- **시그널링**: 주로 TCP 1720 포트 사용(H.225 콜 시그널링).
- **미디어**: RTP/RTCP over UDP.

---

### 예제: H.323 화상 회의 콜 플로우(간단 버전)

**상황:**

- 두 H.323 단말 A, B가 Gatekeeper를 통해 화상 회의를 설정.

**단계:**

1. A, B는 부팅 시 Gatekeeper에 **RAS Registration**으로 자신을 등록.
2. A가 B에게 전화를 걸기 위해 Gatekeeper에 **Admission Request(ARQ)** 전송.
3. Gatekeeper는 정책을 확인하고 **Admission Confirm(ACF)** 로 허용, B의 위치 정보 제공.
4. A와 B는 H.225 콜 시그널링(Q.931 스타일로) 교환:
   - Setup, Call Proceeding, Alerting, Connect 등.
5. H.245를 통해:
   - 서로의 오디오/비디오 능력 교환(코덱 목록).
   - RTP 채널 열기.
6. RTP/RTCP 세션 시작:
   - G.711, H.264 등으로 인코딩된 미디어 스트림 전송.
7. 통화 종료 시:
   - H.225/H.245 메시지로 세션 종료.
   - Gatekeeper에 Disengage(탈퇴) 알림.

---

### SIP vs H.323 비교 및 현대적 위치

**공통점:**

- 둘 다 **멀티미디어 세션 시그널링 프로토콜**.
- RTP/RTCP를 사용해 미디어 전송.
- 음성/영상/데이터 회의, VoIP에 사용 가능.

**차이점(개략):**

| 항목 | SIP | H.323 |
|------|-----|-------|
| 표준화 조직 | IETF (RFC 3261) | ITU-T (H.323 시리즈) |
| 메시지 형식 | 텍스트 기반(HTTP 유사) | 이진/ASN.1 기반 프로토콜 조합 |
| 확장성 | 새 헤더/메서드 추가 용이 | ASN.1 정의 확장 필요 |
| 구현/디버깅 | 패킷 보기 쉬움(텍스트) | 디코더 필요 |
| 사용 현황 | 현대 VoIP/IMS/WebRTC 연계에 널리 사용 | 레거시 영상 회의, 일부 엔터프라이즈 환경 |

현대 인터넷 통신에서는 SIP와 WebRTC 기반 기술이 주도하지만, H.323은 여전히:

- 기업/공공기관의 기존 회의 시스템,
- ISDN/전용선 기반 인프라와 연결된 환경

에서 사용되고 있습니다.

---

## 정리

이 절에서 다룬 **실시간 대화형 멀티미디어 프로토콜**은 역할이 분리되어 있으면서도 서로 긴밀하게 협력합니다.

- **RTP**: 실제 실시간 데이터(오디오/비디오)를 전달하는 **전송 프로토콜**.
- **RTCP**: 품질 모니터링과 제어를 위한 **피드백 채널**.
- **SIP**: 세션 생성/수정/종료를 담당하는 **시그널링 프로토콜** (IETF).
- **H.323**: 패킷 기반 멀티미디어 시스템을 포괄적으로 규정한 **ITU-T 시스템 스펙**.

실무에서 VoIP/화상회의/게임 음성 채팅 등을 설계할 때는:

1. **시그널링을 무엇으로 할 것인가?** (SIP, 자체 프로토콜, H.323, WebRTC 신호 등)
2. **미디어 전송은 RTP/RTCP(SRTP)로 어떻게 구성할 것인가?**
3. **코덱 선택과 지연/지터/손실 예산을 어떻게 잡을 것인가?**

를 함께 고려해야 합니다.
