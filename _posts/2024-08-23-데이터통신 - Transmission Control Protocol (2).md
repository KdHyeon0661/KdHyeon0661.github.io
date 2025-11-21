---
layout: post
title: 데이터 통신 - Transmission Control Protocol (2)
date: 2024-08-23 23:20:23 +0900
category: DataCommunication
---
# Chapter 24.3 Transmission Control Protocol (TCP)

## TCP Services

### TCP가 제공하는 서비스 개요

TCP는 오늘날 인터넷에서 **대부분의 중요 애플리케이션**이 사용하는 전송 계층 프로토콜이다.
핵심 서비스는 다음과 같다.

1. **신뢰적인 데이터 전송(Reliable Data Transfer)**
   - 손실 없는 전달(가능한 한): 재전송, 순서 제어, 오류 검출/복구.
2. **순서 보장(Ordered Delivery)**
   - 바이트 스트림의 순서를 보내는 쪽과 동일하게 맞춤.
3. **중복 제거(Duplicate Suppression)**
   - 중복 세그먼트는 수신 측에서 제거.
4. **흐름 제어(Flow Control)**
   - 수신자가 감당할 수 있을 만큼만 보내도록 송신 측 속도 조절.
5. **혼잡 제어(Congestion Control)**
   - 네트워크 전체의 혼잡을 감안해서 송신 윈도우 크기 조절.
6. **전이중 통신(Full-duplex)**
   - 양 방향으로 동시에 데이터 전송 가능.
7. **연결 지향(Connection-Oriented)**
   - 3-way handshake로 **연결 설정 → 데이터 전송 → 연결 종료**.

이 서비스들은 OS 소켓 API로 사용될 때 다음과 같이 느껴진다.

- 애플리케이션 입장에서는 "**중간에서 순서가 바뀌거나 손실되지 않는 바이트 스트림**"을 읽고 쓰는 것처럼 보인다.
- 실제로는 그 아래에서 TCP가 세그먼트 단위로 쪼개고, 번호를 붙이고, ACK와 재전송을 처리한다.

#### 예제 시나리오: HTTPS로 큰 파일 다운로드

1. 브라우저가 `https://example.com/file.iso` 를 요청한다.
2. OS는 `connect()` 호출로 서버와 TCP 연결을 생성한다.
3. 서버는 응답을 수십~수백 개의 TCP 세그먼트에 나누어 전송한다.
4. 어떤 세그먼트가 손실되면, 클라이언트는 해당 범위를 재전송 요청(빠른 재전송/타임아웃)을 하게 되고, TCP가 알아서 다시 받는다.
5. 애플리케이션(브라우저)은 **단순히 read()를 반복**하면서, 완성된 바이트 스트림만 본다.

브라우저는 “중간에 세그먼트 57이 빠졌다” 같은 세부 사항을 전혀 신경 쓰지 않는다.

---

### Connectionless vs Connection-Oriented (TCP 입장에서)

이미 앞 장에서 UDP를 봤듯이, 비교관점에서 한 번 더 정리해보면:

| 항목                  | UDP (Connectionless)             | TCP (Connection-Oriented)                         |
|-----------------------|----------------------------------|---------------------------------------------------|
| 연결 설정             | 없음                             | 3-way handshake (SYN, SYN+ACK, ACK) 필요          |
| 신뢰성                | 없음                             | 재전송, ACK, 순서 보장                           |
| 혼잡/흐름 제어        | 없음                             | AIMD, 슬로우 스타트, 슬라이딩 윈도우 등         |
| 메시지/스트림 모델    | 메시지(데이터그램) 단위          | 바이트 스트림                                    |
| 헤더 크기             | 8바이트                          | 20~60바이트 (옵션에 따라)                        |
| 활용 예               | DNS, VoIP, 게임, NTP             | 웹, 메일, 파일 전송, DB 연결, 대부분의 API 호출 |

TCP를 선택하는 순간, 애플리케이션은 **“신뢰성과 순서, 혼잡 제어”를 전송 계층에 맡길 수 있다**는 장점이 있다.

---

## TCP Features

TCP가 위 서비스를 구현하기 위해 사용하는 메커니즘을 정리해보자.
아래 내용은 RFC 9293에서 정리된 “TCP 동작”을 기반으로 정리한 것이다.

### 시퀀스 번호와 ACK

- **Sequence Number (32비트)**
  - 바이트 단위 번호. 특정 세그먼트의 **첫 번째 바이트 번호**를 나타낸다.
- **Acknowledgment Number (32비트)**
  - 수신자가 “다음에 받고 싶은 바이트의 번호”를 넣어서 보낸다.
  - 즉, `ACK = N`은 “0~(N-1)번 바이트까지 잘 받았어” 라는 의미.

예를 들어:

- 클라이언트가 처음으로 `SEQ = 1000`부터 `1000~1499` (500바이트)를 보냈다고 하자.
- 서버는 정상 수신 후 `ACK = 1500`을 보낸다.
- 이후 세그먼트가 일부 유실되면, ACK가 앞으로 나아가지 않고 이전 번호에 머무르거나, 중복 ACK가 발생한다.

이 메커니즘만으로도 **순서 제어 + 중복 제거 + 손실 탐지**를 할 수 있다.

### 슬라이딩 윈도우(Flow Control + Reliability 기반)

TCP는 **슬라이딩 윈도우(sliding window)** 를 사용한다.

- 송신자는 “확인(ACK)이 오지 않았지만 보낼 수 있는 바이트 범위”를 **송신 윈도우**로 관리한다.
- 수신자는 자신의 버퍼 여유를 `Window` 필드에 넣어 송신자에게 알려준다 (수신 윈도우).
- 실제 송신 가능 범위는 **min(송신측 윈도우, 수신측 윈도우)** 로 결정된다.

수학적으로 간단히 쓰면, 시점 \(t\)에서 송신 가능한 바이트 수는

$$
W_{\text{send}}(t) = \min\bigl(W_{\text{cwnd}}(t),\ W_{\text{rwnd}}(t)\bigr)
$$

- \(W_{\text{cwnd}}\): 혼잡 윈도우 (네트워크가 허용하는 정도, 혼잡 제어)
- \(W_{\text{rwnd}}\): 수신 윈도우 (수신자의 버퍼 가능량)

이렇게 해서 흐름 제어와 혼잡 제어를 윈도우 개념 하나로 묶어 다룬다.

### 혼잡 제어 (AIMD, Slow Start 등)

전통적인 TCP 혼잡 제어는 다음 개념으로 설명된다.

1. **슬로우 스타트(Slow Start)**
   - 최초 연결 또는 타임아웃 후에는 네트워크 상황을 모르므로,
     작은 윈도우(예: 1 MSS)에서 시작해 ACK 하나마다 윈도우를 1 MSS씩 증가 → RTT마다 대략 두 배씩 (지수 증가).
2. **혼잡 회피(Congestion Avoidance)**
   - 어느 순간부터는 네트워크가 포화에 가까운 것으로 보고,
     RTT마다 1 MSS씩만 증가 (선형 증가).
3. **빠른 재전송(Fast Retransmit) / 빠른 회복(Fast Recovery)**
   - 중복 ACK가 여러 번 오면 (예: 3중복 ACK), 타임아웃까지 기다리지 않고 빠르게 재전송.
   - 윈도우를 절반으로 줄여 AIMD(Additive Increase, Multiplicative Decrease) 패턴을 따른다.

최근에는 BBR, CUBIC 등 다양한 혼잡 제어 알고리즘이 사용되지만, 논리적 구조는 위와 같이 **RTT/손실 기반으로 윈도우를 조절**하는 것이다.

평균 처리량은 대략적으로

$$
\text{Throughput} \approx \frac{W_{\text{cwnd}}}{\text{RTT}}
$$

정도로 생각할 수 있다. (정확한 모델은 더 복잡하지만 직관 용으로 충분하다.)

### 타이머와 RTT 추정

TCP는 여러 종류의 타이머를 사용한다.

- 재전송 타이머(Retransmission Timer)
- 유지 타이머(Keepalive)
- 지연 ACK 타이머(Delayed ACK)
- 연결 종료를 위한 2MSL 타이머(TIME-WAIT 유지시간)

RTT 추정을 위해 고전적으로 다음과 같은 지수 평균을 사용한다.

- 샘플 RTT: \(RTT_{\text{sample}}\)
- 추정된 RTT: \(SRTT\)
- 스무딩 상수 \(\alpha\) (보통 0.125)

$$
SRTT \leftarrow (1 - \alpha) \cdot SRTT + \alpha \cdot RTT_{\text{sample}}
$$

또한 편차를 고려한 RTO(Retransmission Timeout) 설정을 위해
Jacobson/Karels 알고리즘(편차 기반)도 쓰인다.

### 기타 중요한 기능들

1. **세그먼트 재조립과 바이트 스트림 제공**
   - 아래에서 세그먼트 단위로 쪼개지지만, 애플리케이션에는 바이트 스트림으로 제공.
2. **긴 연결 유지(Keep-alive)**
   - 장시간 데이터가 없어도 연결이 살아있는지 확인할 수 있는 옵션 기능.
3. **옵션(Options) 통한 기능 확장**
   - MSS(Maximum Segment Size)
   - Window Scale
   - SACK Permitted / SACK
   - Timestamps
   - ECN(Explicit Congestion Notification) 관련 플래그(반은 IP, 반은 TCP) 등.

---

## TCP Segment

### 세그먼트의 개념

- TCP는 **세그먼트(segment)** 라는 단위로 데이터를 전송한다.
- 각 세그먼트는
  - **TCP 헤더 (20~60바이트)** +
  - **Payload(데이터)**
  로 구성되고, IP 데이터그램 안에 캡슐화된다.

### TCP 헤더 구조

RFC 9293에 정리된 TCP 헤더는 아래와 같은 필드들로 구성된다. (기본 20바이트)

```text
  0      7 8     15 16    23 24    31  (비트 인덱스)
+--------+--------+--------+--------+
|        Source Port         |   Destination Port       |
+--------+--------+--------+--------+
|                     Sequence Number                   |
+--------+--------+--------+--------+
|                  Acknowledgment Number                |
+--------+--------+--------+--------+
| Data  |Res|U|A|P|R|S|F|           Window             |
|Offset |   |R|C|S|S|Y|I|                               |
+--------+--------+--------+--------+
|           Checksum            |     Urgent Pointer    |
+--------+--------+--------+--------+
|                    Options (if any)                   |
+--------+--------+--------+--------+
|                    Data (Payload)                     |
+-------------------------------------------------------+
```

각 필드의 의미:

- **Source Port (16bit)**: 송신 애플리케이션의 포트 번호.
- **Destination Port (16bit)**: 수신 애플리케이션의 포트 번호.
- **Sequence Number (32bit)**: 세그먼트 데이터의 첫 바이트 번호.
- **Acknowledgment Number (32bit)**: 다음에 받고 싶은 바이트 번호 (ACK 플래그가 1일 때 유효).
- **Data Offset (4bit)**: 헤더 길이 (32비트 워드 단위). 최소값 5 → 5×4 = 20바이트.
- **Reserved (3bit)**: 미래용, 항상 0.
- **Control Flags (9bit)**: URG, ACK, PSH, RST, SYN, FIN 등. RFC 9293에서는 이전 RFC 대비 플래그 비트에 대한 정리가 업데이트되었다.
- **Window (16bit)**: 수신자 윈도우 크기(바이트). 윈도우 스케일 옵션과 함께 확장 가능.
- **Checksum (16bit)**: 헤더 + 데이터 + IP pseudo-header에 대한 체크섬.
- **Urgent Pointer (16bit)**: URG 플래그가 1일 때, 긴급 데이터의 끝 위치.
- **Options (가변 길이)**: MSS, Window Scale, SACK, Timestamps 등.

#### 예제: 실제 캡처에서 본 TCP 헤더 (요약)

아래는 패킷 캡처 툴에서 볼 수 있는 요약 예시이다.

```text
Source Port: 52344
Destination Port: 443
Sequence Number: 1234567890
Acknowledgment Number: 987654321
Header Length: 32 bytes
Flags: ACK, PSH
Window: 65535
Checksum: 0x1a2b (correct)
Options: NOP, NOP, Timestamps
```

여기서

- 헤더 길이가 32바이트이므로, 옵션이 12바이트 포함된 상태 (기본 20 + 옵션12).
- PSH 플래그는 “버퍼링하지 말고 즉시 애플리케이션에 전달” 힌트를 주는 용도.

### 세그먼트 분할과 재조립 예제

애플리케이션이 10KB를 한 번에 `send()`한다고 가정:

- MSS가 1460바이트라면, TCP는 대략
  - 1460바이트 × 6개 = 8760바이트
  - + 마지막 1240바이트 1개
  - → 총 7개의 세그먼트로 쪼갠다.
- 각 세그먼트에 적절한 시퀀스 번호를 매긴 뒤 전송.
- 수신자는 세그먼트를 받고 **시퀀스 번호 순으로 재조립**해서 애플리케이션에 연속된 바이트 스트림으로 넘긴다.

만약 중간에 하나가 손실되면?

- 예: 세그먼트 3이 손실
- 수신자 ACK는 세그먼트 2까지에 해당하는 바이트 번호에 머무르고, **중복 ACK**가 발생.
- 송신자는 이를 보고 세그먼트 3만 재전송 → 수신 후 ACK가 앞으로 전진.

---

## A TCP Connection

### 연결의 식별 — 4-튜플

TCP 연결은 다음 4가지 값으로 식별된다.

$$
(\text{Source IP}, \text{Source Port}, \text{Destination IP}, \text{Destination Port})
$$

예:

- 클라이언트: `198.51.100.10:53000`
- 서버: `203.0.113.20:443`

연결 ID: `(198.51.100.10, 53000, 203.0.113.20, 443)`

서버는 보통 “서버 포트 번호(예: 443)”만 고정하고, 나머지는 클라이언트에 따라 달라진다.

### 3-Way Handshake (연결 설정)

고전적인 3-way handshake는 다음 순서로 진행된다. RFC 9293에서도 이 기본 개념을 유지한다.

1. **SYN (클라이언트 → 서버)**
   - 클라이언트가 임의의 ISN(Initial Sequence Number) = `x`를 선택.
   - `SEQ = x`, `SYN = 1`, `ACK = 0`.

2. **SYN+ACK (서버 → 클라이언트)**
   - 서버도 자신의 ISN = `y`를 선택.
   - `SEQ = y`, `ACK = x+1`, `SYN = 1`, `ACK = 1`.

3. **ACK (클라이언트 → 서버)**
   - `SEQ = x+1`, `ACK = y+1`, `ACK = 1`.
   - 이 시점 이후, 연결은 양쪽 모두 ESTABLISHED 상태가 된다.

이 과정을 통해 양측은

- 서로의 초기 시퀀스 번호를 교환하고,
- 양방향으로 데이터 전송 가능한 상태가 된다.

#### 예제: 브라우저가 HTTPS 서버에 연결

1. 클라: `SYN, SEQ=1000` (랜덤)
2. 서버: `SYN,ACK, SEQ=5000, ACK=1001`
3. 클라: `ACK, SEQ=1001, ACK=5001`

이제 양쪽은 `1001`부터, `5001`부터 데이터를 주고받는다.

### 연결 종료 (4-way Handshake)

TCP 연결 종료는 Half-close를 허용하기 때문에 보통 **양방향 각각 FIN을 주고받는 4-way handshake**로 설명된다.

1. A → B : `FIN, SEQ=u`
2. B → A : `ACK, ACK=u+1`
3. B → A : `FIN, SEQ=v`
4. A → B : `ACK, ACK=v+1`

상황에 따라 3,4가 합쳐져서 3-way처럼 보이기도 한다 (동시 종료 등).

### Half-close

한쪽이만 먼저 `FIN`을 보내고, 다른 쪽은 그 후에도 일정 시간 동안 데이터를 더 보낼 수 있다.

- 예: HTTP/1.0에서 서버가 응답을 다 보내고 나서 클라이언트 입력은 더 받지 않는 경우 등.

---

## TCP State Transition Diagram

TCP의 동작을 정확히 이해하려면 **상태(state)** 개념이 중요하다.
RFC 793와 9293에는 유명한 **TCP 상태 전이 다이어그램**이 등장한다.

### 주요 상태 목록

전통적인 TCP 상태는 다음과 같다.

| 상태 이름      | 의미 요약 |
|----------------|-----------|
| CLOSED         | 연결 없음 (가상의 상태) |
| LISTEN         | 서버가 연결 대기 중 |
| SYN-SENT       | 클라이언트가 SYN을 보낸 후 응답 대기 |
| SYN-RECEIVED   | 서버가 SYN을 받고 SYN+ACK를 보낸 상태 |
| ESTABLISHED    | 연결이 성립되어 데이터 송수신 중 |
| FIN-WAIT-1     | 한쪽이 FIN을 보내고 ACK 기다리는 상태 |
| FIN-WAIT-2     | FIN에 대한 ACK를 받았고, 상대의 FIN을 기다리는 상태 |
| CLOSE-WAIT     | 상대가 FIN을 보내왔고, 내가 ACK는 보냈지만 아직 CLOSE 호출 전 |
| CLOSING        | 양쪽이 거의 동시에 FIN을 보낸 상태 |
| LAST-ACK       | 내가 FIN을 보낸 뒤 상대 ACK 기다리는 상태 |
| TIME-WAIT      | 최종 ACK를 보내고 일정 시간(2MSL) 대기하는 상태 |

### 간단한 상태 전이 그림 (요약 버전)

RFC 793의 기본 다이어그램을 텍스트로 단순화하면 다음과 같다:

```text
           active OPEN                         passive OPEN
               |                                   |
               v                                   v
            SYN-SENT <------ CLOSED -------> LISTEN
               |   \                        /   ^
      rcv SYN,ACK   \                    /     |
               |     \ rcv SYN         /       | CLOSE
               v      \              /         |
          ESTABLISHED  \         SYN-RCVD -----
               |  \       \       /   ^
           CLOSE  \       \     /     |
               |    \       \ /       |
               v     \     CLOSE-WAIT |
          FIN-WAIT-1  \              /
               |       \           /
       rcv ACK of FIN   \        /  rcv FIN
               v         \     /    (from other side)
          FIN-WAIT-2      CLOSING
               |             |
       rcv FIN v             v rcv ACK of FIN
            TIME-WAIT     LAST-ACK
               \             /
                \           /
                 v         v
                    CLOSED
```

실제 RFC의 다이어그램은 더 많은 세부 전이를 포함하지만, 큰 흐름은 위와 같다.

### 상태 전이 예제: 클라이언트-서버 통신

#### 서버 측

1. 서버 프로세스가 `listen()` 호출 → TCP 상태: **LISTEN**
2. 클라이언트의 SYN 도착 → **SYN-RECEIVED**
3. 서버가 SYN+ACK 전송, 클라이언트 ACK 도착 → **ESTABLISHED**
4. 클라이언트가 먼저 `FIN` 전송 → 서버는 FIN 수신 후 ACK → **CLOSE-WAIT**
5. 서버 애플리케이션이 `close()` 호출 → FIN 전송 → **LAST-ACK**
6. 클라이언트 ACK 수신 → **CLOSED**

#### 클라이언트 측

1. `connect()` 호출 → SYN 전송 → **SYN-SENT**
2. SYN+ACK 수신 후 ACK 전송 → **ESTABLISHED**
3. `close()` 호출 → FIN 전송 → **FIN-WAIT-1**
4. 서버 ACK 수신 → **FIN-WAIT-2**
5. 서버의 FIN 수신 후 ACK 전송 → **TIME-WAIT**
6. 2MSL 시간 경과 → **CLOSED**

여기서 **TIME-WAIT** 상태는 아주 중요하다.

- 마지막 ACK가 손실될 경우를 대비해,
  **상대방이 FIN을 다시 보내면 확인 ACK를 다시 보내줄 수 있는 기간**.
- 동시에, 같은 4-튜플을 가진 옛 패킷이 네트워크에서 완전히 사라질 때까지 기다리는 역할도 한다.

### 실습 예제: 소켓 상태 변화 관찰

간단한 서버/클라이언트를 실행하면서 `netstat`/`ss` 같은 명령으로 TCP 상태를 볼 수 있다.

#### 서버 코드 (Python, 매우 단순화)

```python
import socket
import time

HOST = "0.0.0.0"
PORT = 5000

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.bind((HOST, PORT))
    s.listen(1)
    print("Listening...")
    conn, addr = s.accept()
    with conn:
        print("Connected by", addr)
        data = conn.recv(1024)
        print("Received:", data.decode(errors="ignore"))
        conn.sendall(b"OK")
        time.sleep(10)  # 일부러 지연
```

#### 클라이언트 코드 (Python)

```python
import socket
import time

HOST = "127.0.0.1"
PORT = 5000

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.connect((HOST, PORT))
    s.sendall(b"Hello TCP")
    data = s.recv(1024)
    print("Received:", data.decode(errors="ignore"))
    time.sleep(5)  # 연결 유지
```

이걸 실행한 뒤, 서버 측에서 `netstat -tnp` 또는 `ss -tan`을 보면

- LISTEN
- ESTABLISHED
- CLOSE-WAIT / FIN-WAIT-*
- TIME-WAIT

등의 상태가 실제로 바뀌는 것을 확인할 수 있다.

---

## 정리

이 절에서 본 TCP의 핵심을 다시 정리하면:

1. **TCP 서비스**
   - 신뢰적, 순서 보장, 전이중, 연결 지향, 흐름·혼잡 제어를 제공한다.
2. **TCP 특징(Features)**
   - 시퀀스/ACK 번호, 슬라이딩 윈도우, 재전송 타이머, 혼잡 제어(AIMD, Slow Start), 옵션(MSS, SACK, Window Scale, Timestamps) 등을 통해 구현된다.
3. **TCP 세그먼트**
   - 헤더(20~60B) + 데이터로 구성되며, 헤더에는 포트, 시퀀스/ACK 번호, Window, 플래그, 체크섬, 옵션 등이 포함된다.
4. **TCP 연결**
   - 3-way handshake로 설정되고, 4-way handshake(또는 변형)로 종료된다.
   - 연결은 (Src IP, Src Port, Dst IP, Dst Port) 4-튜플로 식별된다.
5. **상태 전이 다이어그램**
   - LISTEN, SYN-SENT, SYN-RECEIVED, ESTABLISHED, FIN-WAIT-1/2, CLOSE-WAIT, CLOSING, LAST-ACK, TIME-WAIT, CLOSED 등 상태들 사이를
     SYN/ACK/FIN 패킷과 애플리케이션 호출(OPEN, CLOSE)에 따라 이동한다.

다음 절에서는 이 TCP 위에서 동작하는 고수준의 애플리케이션 프로토콜(HTTP, TLS 등)을 비롯해,
TCP 성능 튜닝과 실제 운영에서 고려해야 할 이슈들을 더 깊게 살펴볼 수 있다.
