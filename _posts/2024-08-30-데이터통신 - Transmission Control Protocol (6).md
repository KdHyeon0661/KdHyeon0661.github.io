---
layout: post
title: 데이터 통신 - Transmission Control Protocol (6)
date: 2024-08-30 23:20:23 +0900
category: DataCommunication
---
# SCTP — Association, Flow Control, Error Control

이 글은 앞에서 다룬 **SCTP 서비스·특징·패킷 포맷** 위에 올라가는 심화 파트다.
여기서는 다음 세 가지를 집중적으로 파고든다.

- An SCTP association
- Flow control
- Error control

설명은 주로 **RFC 9260(최신 SCTP 표준, 2022)**, 개요 RFC 3286, 부분 신뢰 확장 RFC 3758 등의 내용을 기반으로 정리한다.

---

## An SCTP Association

### TCP connection과 무엇이 다른가

TCP에서의 “연결(connection)”에 대응하는 개념이 SCTP에서는 **association**이다.
둘 다 “양 끝 단 사이의 논리적 세션”이지만, SCTP association은 다음 점에서 더 일반적이다.

1. 양 끝 단(endpoint)가 **여러 IP 주소**를 가질 수 있다(멀티호밍).
2. association 안에 **여러 스트림(stream)** 이 있다(멀티스트리밍).
3. 하나의 association이 **공통의 흐름/혼잡 제어**를 공유하면서도,
   각 스트림은 자신만의 순서(SSN)를 가진다.

따라서 SCTP association은

> “여러 IP 주소 쌍 위에서, 여러 스트림을 동시에 싣는 하나의 전송 세션”

이라고 볼 수 있다.

---

### Endpoint와 Association의 구조

SCTP에서 **endpoint**는 다음으로 정의된다.

- **하나 이상의 IP 주소 집합**
- **하나의 SCTP 포트 번호**

예: 서버 endpoint

- 주소 집합: `{192.0.2.10, 198.51.100.10}`
- SCTP 포트: `2905`

클라이언트 endpoint

- 주소 집합: `{203.0.113.7}`
- SCTP 포트: `53000` (ephemeral)

이 둘이 association을 맺으면, association은 다음 정보를 가진다:

- 두 endpoint의 주소 집합
- 초기 TSN, Verification Tag 등 보안 식별 값들
- 각 **경로(path)** 에 대한 상태
  - 예: `(203.0.113.7 → 192.0.2.10)`, `(203.0.113.7 → 198.51.100.10)`
- 각 경로별
  - RTT, RTO, cwnd, ssthresh, error counter
- association 전체에 대해
  - 수신 윈도우(rwnd), 보내는 버퍼, 수신 버퍼
- 스트림 정보
  - outbound stream 수, inbound stream 수
  - 각 스트림의 SSN, 재조립 버퍼 등

이를 아주 단순하게 구조체로 쓰면(개념적):

```c
struct sctp_association {
    uint32_t my_verification_tag;
    uint32_t peer_verification_tag;

    struct sctp_endpoint *local_ep;
    struct sctp_endpoint *peer_ep;

    struct sctp_path paths[MAX_PATHS]; // 각 path에 대한 cwnd, RTO, 상태 등
    int num_paths;
    int primary_path_index;

    uint32_t next_tsn;     // 다음에 보낼 TSN
    uint32_t cum_tsn_ack;  // 상대가 인정한 누적 TSN

    uint32_t rwnd;         // 상대 수신 윈도우(advertised)
    uint32_t flight_size;  // 아직 ACK 안 된 데이터 양

    struct sctp_stream out_streams[MAX_STREAMS];
    struct sctp_stream in_streams[MAX_STREAMS];
};
```

실제 구현은 훨씬 복잡하지만, 느낌은 이런 형태다.

---

### Association 수립: 4-way Handshake 복습

association은 다음과 같은 **4-way 핸드셰이크**로 수립된다.

1. 클라이언트 → 서버: `INIT`
2. 서버 → 클라이언트: `INIT-ACK` (State Cookie 포함)
3. 클라이언트 → 서버: `COOKIE-ECHO`
4. 서버 → 클라이언트: `COOKIE-ACK`

중요 포인트:

- 서버는 `INIT` 을 받았을 때 **아직 상태를 만들지 않는다**.
- `INIT-ACK` 에 있는 쿠키 안에 서버가 만들고 싶은 상태를 암호학적으로 넣는다.
- 클라이언트가 `COOKIE-ECHO` 로 그대로 돌려보낼 때까지 기다렸다가,
  쿠키를 검증한 뒤 실제로 state를 만든다.
- 이 방식 때문에 **SYN Flooding(반쯤 열린 연결이 쌓이는 공격)** 에 훨씬 강하다.

연속된 association 수립 과정을 표로 표현하면:

| 단계 | 발신자 | 메시지    | 주요 내용 |
|------|--------|-----------|-----------|
| 1    | C      | INIT      | 초기 TSN, 지원 스트림 수, 주소 등 |
| 2    | S      | INIT-ACK  | Initiate Tag, Cookie, 주소/스트림 수 |
| 3    | C      | COOKIE-ECHO | 쿠키 그대로 echo |
| 4    | S      | COOKIE-ACK | 쿠키 검증 후 association 생성 완료 |

---

### 스트림과 association의 관계

association에는 **N개의 스트림**이 있다. 각 스트림은:

- `Stream Identifier (SID)` 로 구분
- `Stream Sequence Number (SSN)` 으로 순서 제어
- `DATA` chunk 안에 SID, SSN이 들어간다.

하지만 **흐름 제어와 혼잡 제어는 스트림별이 아니라 “association 전체”에 대해 적용**된다.
즉, 모든 스트림이 **공통의 rwnd, 공통의 cwnd** 를 공유한다.

이 구조는 다음과 같은 효과가 있다.

- 장점:
  - 같은 association에 속한 여러 스트림이 공정하게 대역폭을 나누고,
    TCP 여러 개를 열었을 때처럼 “공정성 붕괴”를 줄인다.
- 단점:
  - 특정 스트림에서 심각한 손실/재전송이 있으면
    association 전체의 cwnd가 줄어든다(혼잡 제어 측면).

멀티스트리밍은 **순서 보장 관점의 HoL blocking 완화**에 초점이 있고,
순수 대역폭 측면에서는 stream 간에 강하게 분리되지 않는다는 점이 중요하다.

---

### 메시지 한 개의 생명주기 예시

예를 들어, 하나의 애플리케이션 메시지가 SCTP association 위로 이동하는 과정을 살펴보자.

1. 애플리케이션이 stream 2로 길이 10 KB의 user message를 전송 요청
2. SCTP는 이 메시지를 하나 또는 여러 개의 **DATA chunk**로 쪼갠다.
3. 각 DATA chunk에:
   - TSN (association 공통 시퀀스 번호)
   - SID = 2
   - SSN (stream 2의 순서 번호)
   - Payload Protocol ID (상위 프로토콜 번호)
4. 현재 path의 `cwnd` 와 association의 `rwnd` 를 고려해
   보낼 수 있는 만큼 패킷으로 묶어 송신
5. 네트워크에서 일부 패킷 손실/지연
6. 수신자는 TSN 기준으로 gap을 파악하고 **SACK** chunk로 알려줌
7. 송신자는 누락된 TSN에 대해 재전송, SRTT/RTO 업데이트, cwnd 조정
8. 수신자가 stream 2의 SSN 순서를 맞춰 user message를 재조립 후 애플리케이션에 인도

이 과정 전체가 하나의 association 내부에서 일어난다.

---

## Flow Control in SCTP

SCTP의 흐름 제어는 전통적인 TCP와 비슷하지만,
**association 전체에 대해 하나의 수신 윈도우(rwnd)** 를 갖는다는 점이 특징이다.

### 수신측 흐름 제어: rwnd 개념

수신측은 자신의 수신 버퍼 크기와 현재 사용량에 따라
송신측이 더 보낼 수 있는 데이터 양을 **rwnd (Receiver Window Credit)** 로 광고한다.

개념적으로:

- 수신 버퍼 총 크기: $$B$$
- 이미 버퍼에 쌓인, 아직 애플리케이션에 인도 안 된 데이터 양: $$U$$

이라면, 수신측이 광고할 rwnd는

$$
\mathrm{rwnd} = B - U
$$

로 나타낼 수 있다.

rwnd는 다음 경로로 전달된다.

1. association 설정 시 `INIT`, `INIT-ACK` chunk 안의
   “Advertised Receiver Window Credit” 값으로 초기값 교환
2. 이후 각 **SACK chunk** 안에 현재 rwnd를 싣는다.

송신측은 최신 SACK에서 받은 rwnd를 저장해 두고,
그 크기보다 더 많은 데이터를 “flight” 시키지 않는다.

---

### 송신측: cwnd와 rwnd의 결합

송신측은 두 가지 제약을 동시에 고려한다.

1. **혼잡 제어(cwnd)**: 네트워크가 허용하는 범위
2. **흐름 제어(rwnd)**: 수신 버퍼가 허용하는 범위

따라서 실제로 한번에 outstanding 상태가 될 수 있는 데이터 양,
즉 **effective send window**는 다음과 같이 정의할 수 있다.

$$
\text{send\_window} = \min\left(\mathrm{cwnd}, \mathrm{rwnd}\right)
$$

송신측은 현재 flight_size(아직 ACK 안 된 데이터 양)를 추적하다가,

- $$ \text{flight\_size} < \text{send\_window} $$
  인 동안만 새로운 DATA chunk를 전송한다.

개념적 코드로 쓰면:

```c
while (flight_size < min(cwnd, rwnd) && app_has_data()) {
    struct data_chunk *c = build_next_data_chunk();
    send_chunk_on_primary_path(c);
    flight_size += c->len;
}
```

이렇게 함으로써,

- 네트워크가 버티는 양(cwnd) 이상은 넣지 않고,
- 수신 버퍼가 감당 가능한 양(rwnd) 이상은 넣지 않도록 제어한다.

---

### SACK를 통한 윈도우 업데이트

수신자는 **SACK(Selective Acknowledgment)** chunk를 통해 다음 정보를 보낸다.

- 누적 ACK TSN (cumulative TSN ack)
- 하나 이상의 gap ack block (중간에 수신된 TSN 범위들)
- 중복 TSN 목록
- 현재 수신 윈도우(rwnd)

예를 들어, 송신측이 TSN 100~110을 보냈고,
수신측이 100, 101, 103~110 을 받았다고 해보자.

수신측 SACK 내용(개념):

- 누적 ACK TSN: 101
- Gap Ack Blocks:
  - [103, 110] (그러면 102가 비어 있다는 의미)
- rwnd: 65535

송신측은 이 SACK를 받고:

1. 100, 101은 완전히 ACK 됨 → flight_size에서 해당 데이터 크기만큼 감소
2. 102는 누락됨(손실 또는 지연 의심) → 재전송 후보
3. 103~110은 “부분 ACK” 상태 → 재전송 불필요
4. rwnd 업데이트 → 다음 전송 가능량 계산에 사용

SCTP는 이런 **gap 기반 SACK** 를 기본으로 사용하므로,
TCP의 옵션 형식 SACK보다 **선택적 재전송이 훨씬 정교하고 필수적**이다.

---

### 예제: 수신 버퍼가 좁을 때의 흐름 제어

가상의 예를 들어보자.

- 수신 버퍼 크기 $$B = 64 \text{ KiB}$$
- 현재까지 애플리케이션이 데이터를 읽지 않아서
  버퍼 사용량 $$U = 48 \text{ KiB}$$
- 그러면 수신측이 광고할 rwnd:

$$
\mathrm{rwnd} = 64 - 48 = 16 \text{ KiB}
$$

송신측 상황:

- 현재 cwnd = 32 KiB (네트워크는 여유가 있음)
- 최신 SACK에서 rwnd = 16 KiB 로 보고됨
- 현재 flight_size = 4 KiB

그러면 effective window:

$$
\text{send\_window} = \min(32, 16) = 16 \text{ KiB}
$$

따라서 새로 보낼 수 있는 양:

$$
16 - 4 = 12 \text{ KiB}
$$

송신측은 최대 12 KiB까지 추가 데이터만 전송하고,
그 이후에는 **새로운 SACK에서 더 큰 rwnd가 광고될 때까지** 기다린다.

수신 애플리케이션이 데이터를 읽어서 버퍼 사용량이 줄면,
다음 SACK에서 rwnd가 커지고, 송신측은 다시 전송을 늘릴 수 있게 된다.

---

### 멀티호밍과 flow control

멀티호밍이 있는 경우:

- cwnd, RTO 등 **혼잡 제어 관련 값은 path마다 따로 유지**한다.
- 그러나 rwnd는 **association 전체에 대해 하나**다.

예를 들어:

- Path A: RTT 짧고 loss 적음 → cwnd 큼
- Path B: RTT 길고 loss 많음 → cwnd 작음

이라면, sender는 자연스럽게 **A 경로로 더 많이 보내고 B경로로는 적게 보내게** 된다.
하지만 두 path에서 전송된 데이터는 모두 association의 rwnd에 의해 제한된다.

---

### WebRTC 데이터 채널에서의 흐름 제어 직관

WebRTC 데이터 채널에서는 SCTP over DTLS over UDP 구조를 쓴다.
이때도 SCTP의 흐름 제어는 동일하게 적용된다.

- 브라우저 엔진 내부에서 association 단위로 rwnd를 관리
- 각각의 data channel은 stream 단위로 관리되지만,
- 송신 측에서 **association 전체 flight_size가 rwnd를 초과하지 않도록** 제어

따라서 큰 파일을 보내는 data channel 하나가 rwnd를 꽉 채우면,
다른 channel의 데이터도 전송이 지연될 수 있다.
이 때문에 WebRTC 구현에서는 **application 레벨에서 자체 큐 관리**를 병행하기도 한다.

---

## Error Control in SCTP

SCTP의 오류 제어는 크게 세 단계로 나눌 수 있다.

1. **오류 검출**: CRC32c 체크섬 (및, 최신에는 특정 상황에서의 zero checksum 확장)
2. **전달 오류(손실, 중복) 제어**: TSN + SACK 기반 신뢰 전송
3. **경로/association 수준 오류 처리**: HEARTBEAT/ABORT/SHUTDOWN 처리, PR-SCTP 등

### 오류 검출: CRC32c와 Zero Checksum 확장

기본적으로 SCTP는 각 패킷 공통 헤더의 `Checksum` 필드에
**32비트 CRC32c(Castagnoli 다항식)** 를 사용한다.

- TCP/UDP의 16비트 1의 보수 체크섬보다 검출 능력이 훨씬 좋다.
- 송신측은 Checksum 필드를 0으로 두고 CRC32c를 계산한 뒤 해당 값으로 채운다.
- 수신측은 동일한 계산을 해서 일치하지 않으면 패킷을 버린다.

2024년에는 **Zero Checksum for SCTP** 라는 확장 RFC 9653이 나왔다.

- IPsec, DTLS 같은 **하위 계층에서 이미 강한 무결성 보호**가 있는 환경에서
- SCTP 체크섬을 0으로 설정(실질적으로 비활성화)할 수 있는 옵션을 정의한다.
- 이 경우 CPU 오버헤드를 줄이고, NIC offload와의 충돌을 피하는 것이 목적이다.
- 다만, 모든 환경에서 허용되는 것은 아니며,
  보안/신뢰 요구 사항에 따라 신중하게 선택해야 한다.

기본 교재 수준에서는 “SCTP는 CRC32c로 에러 검출을 하고,
최신 표준에서는 일부 환경에서 zero checksum 옵션이 추가되었다” 정도로 이해하면 충분하다.

---

### 신뢰 전달: TSN과 SACK

SCTP의 신뢰성 보장은 **TSN(Transmission Sequence Number)** 와
**SACK** 메커니즘으로 이루어진다.

- 각 DATA chunk에는 **TSN** 이 들어간다.
  - association 전체에서 단조 증가하는 번호
- 수신측은 보유한 TSN들의 집합으로부터 “누적 ACK”와 “gap”을 계산해 SACK으로 회신한다.
- 송신측은 SACK 정보로부터
  - 어떤 TSN이 확실히 도착했는지
  - 어떤 TSN이 손실/지연되었는지
  를 판단하여 재전송을 결정한다.

#### 예제: TSN 100~104 전송 후 일부 손실

1. 송신측: TSN 100, 101, 102, 103, 104 전송
2. 네트워크: 100, 101, 103, 104만 수신, 102는 손실
3. 수신측이 보유 TSN: {100, 101, 103, 104}
4. SACK 작성:
   - 누적 ACK TSN = 101
   - Gap Ack Block = [103, 104]
   - rwnd = (현재 버퍼 상황)
5. 송신측:
   - 100, 101은 완전 ACK → flight_size 감소
   - 102는 누락 → 손실/지연 후보로 마킹
   - 103, 104는 부분 ACK → 재전송 필요 없음

이후에도 몇 번 더 SACK에서 102가 gap으로 남아 있으면
**Fast Retransmit** 규칙에 따라 재전송할 수 있다(구체 임계값은 구현에 따라 다름).

---

### RTO 계산과 타이머

SCTP의 **재전송 타임아웃(RTO)** 는 TCP와 유사한 방식으로 계산한다.

- 각 path마다 RTT 샘플을 측정
- 지수 가중 이동 평균으로 $$SRTT$$(smoothed RTT)와 $$RTTVAR$$(RTT 변동)을 계산
- RTO 공식은 대략

$$
\mathrm{RTO} = SRTT + 4 \times RTTVAR
$$

(정확한 상수나 최소/최대 RTO 값은 표준에 정의되어 있음)

송신측은 각 DATA chunk에 대해:

1. 전송 시점에 해당 path의 RTO를 기록
2. 타이머가 만료되기 전 SACK으로 ACK되면
   - RTT 샘플을 사용해 SRTT/RTTVAR/RTO 업데이트
3. 타이머 만료되면
   - 해당 TSN과 그 path에 대해 **Timeout Retransmission** 수행
   - 해당 path의 cwnd 감소, ssthresh 조정(혼잡 회복 알고리즘에 따름)

---

### Fast Retransmit: SACK gap 기반

TCP에서는 “중복 ACK 3개” 같은 규칙으로 Fast Retransmit를 한다.
SCTP에서는 **SACK gap 정보**를 기반으로 보다 일반적인 fast retransmit를 한다.

간단히 말해:

- 특정 TSN이 “여러 번 SACK에 gap으로 언급”되면
  (즉, 그 TSN 이후의 것들은 여러 번 ACK 되었는데,
  해당 TSN은 여전히 누락 상태라면)
- 타이머 만료를 기다리지 않고 바로 재전송할 수 있다.

예제:

- TSN 100~110 전송
- SACK #1: Cumulative TSN ack = 101, Gap [103, 110]
- SACK #2: 여전히 Gap [103, 110]
- SACK #3: 여전히 Gap [103, 110]

이때 TSN 102는 삼차례나 gap으로 확인되었으므로
송신측은 타임아웃 전에 102를 fast retransmit할 수 있다.

---

### 부분 신뢰(Partial Reliability)와 FORWARD TSN

RFC 3758은 **SCTP Partial Reliability Extension** 을 정의한다.

이 확장은 “어떤 데이터는 기한이 지나면 굳이 재전송할 필요 없다”는 요구를 반영한다.

- 새로운 chunk 타입: **FORWARD-TSN**
- 송신측이 “이 TSN까지의 일부 데이터는 더 이상 보내지 않겠다”고
  수신측에 알려 누적 ACK 지점을 “앞으로 건너뛰는” 역할을 한다.
- 수신측은 아직 받지 못한 해당 TSN들을 “버리고”
  그 이후 TSN부터를 기준으로 재조립을 진행한다.

예제 시나리오 (실시간 게임 상태):

- 매 50ms마다 캐릭터 위치 업데이트 전송
- 100ms 안에 도착하지 못한 업데이트는 이미 “시효 만료”
- 이런 메시지는 timeout 후 FORWARD-TSN으로 건너뛰고,
  최신 상태만 유지함으로써 딜레이/재전송 오버헤드를 줄일 수 있다.

WebRTC 데이터 채널은 SCTP를 사용할 때 **이 PR-SCTP 확장을 필수로 사용**하도록 정의한다.

---

### 경로·association 수준 오류 처리

SCTP는 다음과 같은 control chunk들을 이용해 **경로, association 수준의 오류를 처리**한다.

- HEARTBEAT / HEARTBEAT-ACK
  - 각 path의 상태를 주기적으로 확인
  - HEARTBEAT에 응답이 없으면 error counter를 올리고,
    threshold를 넘으면 path를 “down”으로 표시
- ABORT
  - 치명적인 오류 발생 시 즉시 association 종료
  - 예: 수신측이 “보낸 적 없는 TSN이 ACK되었다”는 SACK을 받는 경우,
    이는 심각한 오류 또는 공격으로 간주하고 ABORT를 보낼 수 있다.
- SHUTDOWN / SHUTDOWN-ACK / SHUTDOWN-COMPLETE
  - 우아한 종료: 모든 outstanding DATA가 ACK된 뒤 종료

이 메커니즘으로, SCTP는

- 패킷 손실 같은 일시적인 문제는 재전송과 혼잡 제어로 복구하고,
- 패킷 변조, 잘못된 SACK 같은 심각한 문제는 ABORT로 association을 종료하며,
- 네트워크 경로 장애는 HEARTBEAT 기반으로 감지하고 대체 경로로 failover 한다.

---

### 예제: 무선 환경에서의 오류 제어

무선 링크(BER가 비교적 높은 환경)에서 SCTP를 사용할 경우:

- CRC32c가 **비트 오류 검출**을 담당
- 손실된 패킷은 SACK gap으로 감지되고, RTO/fast retransmit로 재전송
- 그러나 손실이 너무 잦으면 cwnd가 계속 줄어
  유효 대역폭이 크게 감소할 수 있다.

이를 개선하기 위해, 일부 연구에서는
**패킷 내부를 부분적으로 나눠서 CRC를 적용하는 스킴**(제안 수준)을 연구해
일부 손상된 chunk만 재전송하는 방법을 모색하기도 한다.

표준 SCTP 관점에서 중요한 점은:

- 에러 검출은 패킷 단위(CRC32c)
- 손실/재전송 제어는 TSN+SACK 기반
- path-level 오류는 HEARTBEAT/ABORT/SHUTDOWN으로 처리

라는 구조를 이해하는 것이다.

---

### 오류 제어 루프의 개념적 의사코드

SCTP 송신측의 핵심 오류 제어 루프를 아주 단순하게 의사코드로 표현하면:

```c
on_data_to_send(msg, sid):
    chunks = fragment_into_data_chunks(msg, sid)
    for c in chunks:
        c.tsn = next_tsn++
        add_to_retransmission_queue(c)
    try_send_data()

try_send_data():
    while (flight_size < min(cwnd, rwnd) && has_unsent_chunks()):
        c = get_next_unsent_chunk()
        send_on_best_path(c)
        c.sent_time = now()
        c.path = best_path
        flight_size += c.len
        start_timer_if_needed()

on_sack_received(sack):
    update_rwnd(sack.rwnd)
    mark_acked_tsns(sack)
    detect_lost_tsns(sack)  // gap 기반
    for each newly_acked_chunk:
        flight_size -= chunk.len
        update_srtt_rto(chunk.path, now() - chunk.sent_time)
    if (some_tsns_likely_lost && no_timer_running_for_them):
        fast_retransmit(lost_tsns)
    adjust_cwnd_based_on_sack()
    try_send_data()

on_timeout(path):
    // 해당 path의 일부 TSN이 타임아웃
    retransmit_oldest_unacked_on_path(path)
    reduce_cwnd(path)
    recalc_rto(path)
    restart_timer_if_needed()
```

실제 구현은 훨씬 복잡하지만,
**SACK 수신 → ACK/손실 판단 → 재전송/윈도우 조정 → 새 데이터 전송**
이라는 루프가 SCTP 오류 제어의 핵심이다.

---

## 정리

- **An SCTP association**
  - 여러 IP 주소를 가진 endpoint 쌍 사이의 논리 세션으로,
    멀티호밍과 멀티스트리밍을 지원한다.
  - 4-way handshake와 쿠키 메커니즘으로 강한 DoS 저항성을 가진다.
  - 각 스트림은 독립적인 SSN을 갖지만,
    흐름·혼잡 제어는 association 수준에서 공통으로 수행된다.

- **Flow control**
  - 수신측 rwnd와 송신측 cwnd를 결합해
    $$\min(\mathrm{cwnd}, \mathrm{rwnd})$$ 만큼만 “날려 보낼 수” 있다.
  - rwnd는 수신 버퍼 상태에 따라 SACK으로 계속 광고된다.
  - 멀티호밍 환경에서도 rwnd는 association 전체에 대해 하나이며,
    cwnd, RTO 등은 path별로 유지된다.

- **Error control**
  - CRC32c 기반 체크섬으로 에러 검출 (일부 환경에서 zero checksum 확장 존재).
  - TSN + SACK 기반 선택적 재전송으로 신뢰성 확보.
  - RTO, fast retransmit, PR-SCTP(FORWARD-TSN) 등을 통해
    다양한 서비스 특성(완전 신뢰, 부분 신뢰)을 조합할 수 있다.
  - HEARTBEAT, ABORT, SHUTDOWN 등으로 path/association 수준 오류를 처리한다.

이 구조 덕분에 SCTP는 **전화망 시그널링, 모바일 코어, WebRTC 데이터 채널** 등에서
TCP/UDP만으로는 구현하기 까다로운 요구사항(멀티스트리밍, 멀티호밍, 부분 신뢰)을
유연하게 만족시키는 전송 계층 프로토콜로 자리잡고 있다.
