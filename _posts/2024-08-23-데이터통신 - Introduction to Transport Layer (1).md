---
layout: post
title: 데이터 통신 - Introduction to Transport Layer (1)
date: 2024-08-23 20:20:23 +0900
category: DataCommunication
---
# Chapter 23. Introduction to Transport Layer

이제까지는 **네트워크 계층(Network Layer)** 을 중심으로 “패킷을 어디로 보낼 것인가(라우팅)”에 초점을 맞췄다면,  
**전송 계층(Transport Layer)** 는 “프로세스끼리 어떻게 안전하고 효율적으로 데이터를 주고받게 할 것인가”에 초점을 둔다.

- 네트워크 계층: **호스트 대 호스트 (host-to-host)**  
- 전송 계층: **프로세스 대 프로세스 (process-to-process)**

즉, IP가 “어느 컴퓨터”까지 데려다주는 택배 회사라면,  
전송 계층(TCP/UDP)은 그 컴퓨터 안에서 “어느 프로그램(배송 담당자)”에게 전달할지를 결정하고,  
필요하다면 **신뢰성·순서·흐름 제어**까지 책임진다.

---

## 1. Transport Layer Services

### 1.1 네트워크 계층과 전송 계층의 역할 분리

먼저 계층 간 역할을 다시 정리해 보자.

| 계층          | 주요 단위     | 주요 역할                             |
|---------------|---------------|----------------------------------------|
| 네트워크 계층 | 패킷(packet)  | 호스트 간 전달, 라우팅, 홉 간 포워딩  |
| 전송 계층     | 세그먼트(segment) | 프로세스 간 전달, 신뢰성/순서/흐름/혼잡 제어 |

전송 계층은 **“호스트 내부”에 있는 여러 프로세스들**을 구분하기 위해 **포트 번호(port number)** 를 사용한다.

- IP 주소: 호스트를 구분 (집 주소)  
- 포트 번호: 프로세스를 구분 (집 안의 방 번호 / 수신 담당자)

텍스트 그림으로 표현하면:

```text
[프로세스 A:포트 5000] --\
                         \
[프로세스 B:포트 5001] ----[전송 계층: TCP/UDP]---- [네트워크 계층: IP] ---- 인터넷
                         /
[프로세스 C:포트 8080] --/
```

같은 호스트에서도 여러 프로세스가 동시에 통신할 수 있고,  
각 프로세스는 `(IP 주소, 포트 번호)` 쌍으로 구분된다.

### 1.2 전송 계층이 제공하는 핵심 서비스들

전송 계층의 대표적인 서비스는 다음과 같이 정리할 수 있다.

1. **프로세스 간 주소 지정 (Process-to-Process Delivery)**  
2. **다중화와 역다중화 (Multiplexing / Demultiplexing)**  
3. **신뢰성 있는 데이터 전송 (Reliable Data Transfer)**  
4. **순서 보장 (Ordered Delivery)**  
5. **흐름 제어 (Flow Control)**  
6. **혼잡 제어 (Congestion Control)**  
7. **오류 검출 및 처리 (Error Detection & Handling)**

각 항목을 예제와 함께 자세히 보자.

---

### 1.2.1 프로세스 간 주소 지정 (Port Number)

**포트 번호(port number)** 는 전송 계층에서 프로세스를 식별하기 위한 숫자 식별자이다.

- 0–1023: 잘 알려진 포트(Well-known Port) (예: HTTP 80, HTTPS 443, DNS 53 등)  
- 1024–49151: 등록된 포트(Registered Port) (응용 프로그램들이 등록해서 사용)  
- 49152–65535: 동적/임시 포트(Dynamic/Ephemeral Port) (클라이언트 측에서 임시로 사용)

예를 들어, 다음과 같은 TCP 연결이 있다고 하자.

```text
클라이언트: (192.0.2.10, 51000)
서버:       (198.51.100.20, 443)
```

이 연결은 네 개의 값 `(소스 IP, 소스 포트, 목적지 IP, 목적지 포트)` 로 고유하게 식별된다.

```text
(192.0.2.10, 51000, 198.51.100.20, 443)
```

서버 입장에서는 **포트 443에 도착하는 TCP 세그먼트**를 HTTPS 서버 프로세스로 넘겨주면 된다.

---

### 1.2.2 다중화와 역다중화 (Multiplexing / Demultiplexing)

전송 계층은 하나의 호스트에서 **여러 프로세스의 데이터를 동시에 섞어** 네트워크로 내보내고,  
반대로 도착한 세그먼트를 **적절한 프로세스로 분배**해야 한다.

- **다중화(Multiplexing)**: 여러 응용 프로세스에서 올라오는 데이터를 한 개의 IP 흐름 위로 “합치는 것”  
- **역다중화(Demultiplexing)**: 수신한 세그먼트를 포트 번호를 기준으로 다시 응용 프로세스에 나누어 주는 것

예시 상황:

- 브라우저(포트 51000)에서 웹 서버(443)로 HTTPS 연결  
- 동시에 메신저(포트 52000)에서 채팅 서버(5222)로 연결  

전송 계층은 두 응용의 데이터를 다음과 같이 처리한다.

```text
응용 계층
  ↓        ↓
[브라우저] [메신저]
  ↓        ↓
[ TCP 세그먼트 (브라우저) ][ TCP 세그먼트 (메신저) ] → 전송 계층에서 다중화
  ↓
[ IP 패킷 스트림 ] → 네트워크
```

수신 측에서는, 도착한 세그먼트의 `(목적지 포트 번호)` 를 보고 다시 각각 **브라우저 프로세스, 메신저 프로세스**로 분배한다.

---

### 1.2.3 신뢰성 있는 데이터 전송 (Reliable Data Transfer)

**신뢰성(reliability)** 이란 “송신자가 보낸 데이터를 **손실·중복·손상 없이**, 그리고 **중복 없이** 정확히 수신하는 것”을 의미한다.

전송 계층이 제공하는 전형적인 신뢰성 기능:

1. **재전송 (Retransmission)**  
   - 수신자가 세그먼트를 받지 못했거나 손상되었을 때, 송신자가 다시 보내는 것
2. **응답(ACK)와 타임아웃**  
   - 수신자가 정상적으로 받으면 ACK 세그먼트를 보낸다.  
   - 송신자는 일정 시간(타임아웃) 내 ACK를 받지 못하면 재전송.
3. **순서 번호(Sequence Number)**  
   - 각 데이터 바이트 혹은 세그먼트에 번호를 매겨 순서를 추적.
4. **중복 제거(Duplicate Detection)**  
   - 같은 시퀀스 번호를 가진 세그먼트가 여러 번 도착하면, 중복으로 간주하고 하나만 상위에 전달.

TCP는 이러한 메커니즘을 통해 “**신뢰성 있는 바이트 스트림**”을 제공한다.

간단한 추상 수식으로 보면, 송신자 S가 보낸 바이트 스트림을 $b_1, b_2, \dots, b_n$ 이라 할 때,  
신뢰성 있는 전송은 수신자 R이 **동일한 순서**와 **동일한 내용**의 바이트 스트림을 받도록 하는 것이다.

$$
(b_1, b_2, \dots, b_n)_\text{송신} \longrightarrow (b_1, b_2, \dots, b_n)_\text{수신}
$$

---

### 1.2.4 순서 보장 (Ordered Delivery)

IP 계층은 패킷들을 서로 다른 경로로 보낼 수 있기 때문에,  
패킷 A가 먼저 보내졌는데 나중에 도착할 수 있다.

**순서 보장** 서비스는 다음을 의미한다.

- 송신자가 보낸 순서대로 수신자의 응용 계층에 데이터를 인도  
- 패킷이 뒤섞여 도착해도 전송 계층에서 재정렬(reordering)

예:

- 송신: [세그먼트 #1], [세그먼트 #2], [세그먼트 #3]  
- 네트워크에서 도착: #2, #1, #3 순서  
- 전송 계층(TCP)은 내부 버퍼에서 1,2,3 순서로 정렬 후 응용 계층에 전달

---

### 1.2.5 흐름 제어 (Flow Control)

**흐름 제어(flow control)** 는 “수신자가 처리할 수 있는 속도보다 송신자가 너무 빨리 보내지 않도록 조절하는 것”이다.

- 수신 버퍼가 작다면,  
  → 송신자는 그 크기를 초과하지 않도록 전송 속도를 줄여야 한다.

간단한 개념 수식:

- 수신 버퍼 크기: $R$ 바이트  
- 송신자가 이미 보낸 데이터 중 아직 ACK를 받지 못한(즉, 수신자의 버퍼에 쌓여 있을 가능성이 있는) 데이터 총량: $U$ 바이트  

흐름 제어 조건은:

$$
U \le R
$$

TCP는 수신자가 **"윈도우 크기(window size)"** 를 알려주어, 송신자가 한 번에 보낼 수 있는 미확인 데이터 양을 조절하도록 한다.

---

### 1.2.6 혼잡 제어 (Congestion Control)

**혼잡(congestion)** 은 네트워크(특히 라우터의 큐)가 과부하되어 패킷이 지연되거나 버퍼 오버플로로 손실되는 상태를 말한다.

- 흐름 제어: **송신자 ↔ 수신자의 속도 차이** 문제  
- 혼잡 제어: **송신자 ↔ 네트워크(라우터) 용량** 문제

혼잡 제어 알고리즘은 대개 다음 방식으로 동작한다.

1. 송신자는 처음에는 작은 전송 윈도우(예: 1 MSS)를 사용  
2. 네트워크가 혼잡해 보이지 않으면 윈도우를 점점 키움 (cwnd 증가)  
3. 손실(타임아웃, 중복 ACK 등)이 감지되면 윈도우를 급격히 줄이고(혼잡 신호), 다시 천천히 증가

이를 간단한 수식으로 표현하면, TCP Tahoe/Reno 계열에서 흔히 나오는 AIMD(Additive Increase, Multiplicative Decrease) 패턴이다.

- 혼잡 윈도우 크기 $W$ 에 대해:

$$
W \leftarrow W + 1 \quad (\text{congestion-free, 한 RTT당})
$$

$$
W \leftarrow \frac{W}{2} \quad (\text{packet loss 발생 시})
$$

이를 통해 네트워크의 혼잡이 심해지지 않도록 스스로 조절하는 특성이 생긴다.

---

### 1.2.7 오류 검출 및 처리 (Error Detection & Handling)

전송 계층은 세그먼트에 **체크섬(checksum)** 을 포함시켜,  
데이터가 전달 중에 손상되었는지 확인한다.

- 송신자:
  - 세그먼트 헤더와 데이터 일부 또는 전체에 대해 체크섬 계산
  - 결과를 헤더 필드에 넣어 보냄
- 수신자:
  - 동일한 방식으로 체크섬을 계산하여 헤더의 값과 비교
  - 일치하지 않으면 오류로 간주

오류 검출 후 행동은 프로토콜에 따라 다르다.

- UDP: 오류 세그먼트를 **단순히 폐기**, 재전송은하지 않는다.  
- TCP: 오류 또는 손실로 인해 세그먼트가 제대로 수신되지 않았음을 감지하는 경우,  
  재전송(retransmission)을 통해 복구한다.

---

## 2. Connectionless and Connection-Oriented Protocol

전송 계층의 대표적인 프로토콜:

- **UDP(User Datagram Protocol)**: connectionless, 비신뢰성, 단순/저오버헤드  
- **TCP(Transmission Control Protocol)**: connection-oriented, 신뢰성, 순서 보장

둘 다 전송 계층이지만, 제공하는 서비스 수준이 확연히 다르다.

---

### 2.1 Connectionless Protocol — UDP

#### 2.1.1 개념

**Connectionless**란, 송신자와 수신자가 실제 데이터 전송 전에 **논리적인 연결(setup)을 하지 않고**  
그때그때 메시지(datagram)를 독립적으로 전달하는 방식을 의미한다.

UDP의 특징:

- 연결 설정/해제 과정 없음 (3-way handshake 없음)  
- 최소한의 헤더(소스/목적지 포트, 길이, 체크섬)  
- **신뢰성 제공 안 함**: 손실, 순서 뒤바뀜, 중복을 그대로 상위 계층으로 올릴 수 있음  
- 흐름 제어, 혼잡 제어 없음 (애플리케이션이 직접 구현해야 함)  
- 그 대신 **지연과 오버헤드가 적고 단순함**

대표적인 사용 예:

- DNS 조회  
- 스트리밍(VoIP, 실시간 게임, 실시간 멀티미디어 전송에서 자체적인 오류복구를 구현)  
- 간단한 요청/응답 프로토콜

#### 2.1.2 UDP 헤더 구조 (고수준 개념)

UDP 헤더는 크게 4개 필드(각 16비트)로 구성된다.

| 필드            | 설명                    |
|-----------------|-------------------------|
| Source Port     | 송신자 포트 번호        |
| Destination Port| 수신자 포트 번호        |
| Length          | UDP 헤더 + 데이터 길이  |
| Checksum        | 오류 검출용 체크섬      |

이렇게 최소한의 정보만 넣는 이유는 **단순성과 속도**를 우선하기 때문이다.

#### 2.1.3 예제 시나리오 — UDP 기반 간단한 로깅 시스템

상황:

- 서버: 중앙 로깅 서버 (UDP 포트 514, 예: syslog 스타일)  
- 클라이언트: 수많은 장비들(네트워크 장비, IoT 장치 등)

장비는 로그가 발생할 때마다 다음과 같이 간단한 UDP 메시지를 보낸다.

```text
[장비 A: 10.0.0.1:50000] → [로그 서버: 203.0.113.10:514]

"2025-11-16T12:00:00Z LOGIN FAILED user=admin ip=203.0.113.200"
```

- 연결 설정 없음 → 지연 없이 바로 전송  
- 혹시 로그 메시지가 몇 개 손실되더라도, 전체 시스템에는 크게 치명적이지 않은 경우  
- 로깅 서버는 UDP 포트 514에서 오는 메시지를 받아 파일/DB에 저장

이런 경우 UDP의 “가벼움”이 오히려 장점이다.

#### 2.1.4 간단한 코드 예제 (UDP 에코 서버/클라이언트)

전송 계층 개념을 체감하기 위해, 간단한 UDP 에코 예제를 보자.  
(언어는 이해하기 쉬운 Python 스타일로 예시)

- 서버: 특정 포트에서 UDP 세그먼트를 받고, **그대로 되돌려주는** 역할

```python
# udp_echo_server.py
import socket

SERVER_IP = "0.0.0.0"
SERVER_PORT = 9999

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind((SERVER_IP, SERVER_PORT))

print("UDP echo server listening on port", SERVER_PORT)

while True:
    data, addr = sock.recvfrom(4096)   # 세그먼트 수신
    print("Received from", addr, ":", data.decode("utf-8", errors="ignore"))
    sock.sendto(data, addr)            # 그대로 되돌려 보냄
```

- 클라이언트: 사용자가 입력한 문자열을 서버로 보내고, 돌아온 응답을 출력

```python
# udp_echo_client.py
import socket

SERVER_IP = "203.0.113.10"   # 서버 공인 IP라고 가정
SERVER_PORT = 9999

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

while True:
    msg = input("메시지 입력 (quit 입력 시 종료): ")
    if msg == "quit":
        break
    # UDP 세그먼트 전송
    sock.sendto(msg.encode("utf-8"), (SERVER_IP, SERVER_PORT))
    # 응답 수신 (손실될 수도 있음을 염두에 둔다)
    data, _ = sock.recvfrom(4096)
    print("서버 응답:", data.decode("utf-8", errors="ignore"))
```

특징:

- 별도의 “연결” 단계가 없다. `sendto()` 할 때마다 독립적인 datagram 전송.  
- 서버는 `recvfrom()` 을 호출해 **어떤 클라이언트에서 온 것인지 (addr)** 를 매번 확인하고, 그 주소로 응답을 돌려준다.

---

### 2.2 Connection-Oriented Protocol — TCP

#### 2.2.1 개념

**Connection-oriented**란, 실제 데이터 전송에 앞서 **논리적인 연결(connection)** 을 먼저 설정하고,  
그 연결을 통해 순서대로 데이터 스트림을 전달한 뒤, 마지막에 연결을 종료하는 것을 의미한다.

TCP의 주요 특성:

1. **연결 기반 (connection-oriented)**  
   - 3-way handshake를 통해 연결 설정  
   - 4-way close 또는 기타 절차로 연결 해제
2. **신뢰성 있는 전달 (reliable)**  
   - 재전송, ACK, 시퀀스 번호, 오류 검출, 중복 제거
3. **순서 보장 (ordered)**  
   - 바이트 스트림에 순서 번호를 매겨 도착 순서와 상관없이 재조립
4. **흐름 제어 (flow control)**  
   - 수신자의 버퍼 여유에 맞춰 송신 윈도우 크기를 조절
5. **혼잡 제어 (congestion control)**  
   - 네트워크 혼잡 신호(손실, RTT 변화 등)를 바탕으로 송신 속도를 조절
6. **전이중(Full-duplex)**  
   - 양방향으로 동시에 데이터 전송 가능

#### 2.2.2 연결의 개념적 수식 표현

송신자 S와 수신자 R 간의 TCP 연결을 $(S, R)$ 라고 했을 때,  
전송되는 바이트 스트림을 $b_1, b_2, \dots, b_n$ 이라 하면

$$
(S, R) \colon (b_1, b_2, \dots, b_n)_S \longrightarrow (b_1, b_2, \dots, b_n)_R
$$

를 만족하도록, TCP는 시퀀스 번호·ACK·윈도우·타임아웃 등을 사용해 내부적으로 복잡한 동작을 수행한다.

#### 2.2.3 예제 시나리오 — 웹 브라우징

사용자가 웹 브라우저에서 `https://example.com` 을 열었을 때, 아래와 같이 TCP 연결이 사용된다.

1. DNS 조회 → 웹 서버의 IP 주소 획득 (예: `198.51.100.20`)  
2. 브라우저(클라이언트)는 서버의 443 포트(TLS/HTTPS)에 TCP 연결 시도  
   - 클라이언트: (192.0.2.10, 51000)  
   - 서버: (198.51.100.20, 443)
3. TCP 3-way handshake  
   - SYN → SYN+ACK → ACK  
   - 이 과정이 완료되면 “연결이 수립되었다”고 본다.
4. TLS 핸드셰이크, HTTP 요청/응답 전송  
5. 페이지 로딩이 끝나면 TCP 연결을 종료(FIN/ACK 등)

TCP 덕분에 애플리케이션(HTTP/TLS)은  
“손실·순서 뒤바뀜·중복” 등을 전혀 신경 쓰지 않고, 단순히 **바이트 스트림**으로 요청/응답을 주고받을 수 있다.

#### 2.2.4 간단한 코드 예제 (TCP 에코 서버/클라이언트)

UDP 예제와 대조해 보자.

- TCP 에코 서버 예시:

```python
# tcp_echo_server.py
import socket

SERVER_IP = "0.0.0.0"
SERVER_PORT = 9999

server_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_sock.bind((SERVER_IP, SERVER_PORT))
server_sock.listen()

print("TCP echo server listening on port", SERVER_PORT)

while True:
    conn, addr = server_sock.accept()   # 연결 수락 (connection-oriented)
    print("Connected by", addr)
    while True:
        data = conn.recv(4096)
        if not data:
            break  # 클라이언트가 연결을 종료한 경우
        conn.sendall(data)  # 받은 데이터 그대로 에코
    conn.close()
```

- TCP 클라이언트 예시:

```python
# tcp_echo_client.py
import socket

SERVER_IP = "203.0.113.10"
SERVER_PORT = 9999

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect((SERVER_IP, SERVER_PORT))  # 연결 설정 (3-way handshake)

while True:
    msg = input("메시지 입력 (quit 입력 시 종료): ")
    if msg == "quit":
        break
    sock.sendall(msg.encode("utf-8"))   # 바이트 스트림 전송
    data = sock.recv(4096)              # 서버 에코 수신
    print("서버 응답:", data.decode("utf-8", errors="ignore"))

sock.close()   # 연결 종료
```

특징:

- `connect()` 호출 시 **연결 설정** (3-way handshake) 수행  
- 이후에는 `sendall()` / `recv()` 를 통해 **지속적인 바이트 스트림** 전송  
- `close()` 를 호출하여 연결 종료

---

### 2.3 Connectionless vs Connection-Oriented — 비교 정리

아래 표는 UDP(대표적인 connectionless)와 TCP(대표적인 connection-oriented)의 차이점을 정리한 것이다.

| 항목                   | UDP (Connectionless)                     | TCP (Connection-Oriented)                         |
|------------------------|------------------------------------------|---------------------------------------------------|
| 연결 설정              | 없음                                    | 3-way handshake 필요                              |
| 신뢰성                 | 보장하지 않음                           | 재전송·ACK·순서 보장 등으로 높은 신뢰성 제공     |
| 순서 보장              | 없음                                    | 보장                                              |
| 흐름 제어              | 없음                                    | 있음 (수신자 advertised window)                  |
| 혼잡 제어              | 없음 (애플리케이션이 직접 구현 가능)    | 있음 (cwnd 등 다양한 알고리즘)                   |
| 헤더 크기              | 매우 작음(8바이트 기본)                 | 비교적 큼 (20바이트 이상)                         |
| 지연/오버헤드          | 매우 낮음                               | 더 높음                                           |
| 사용 예                | DNS, VoIP, 스트리밍 일부, 게임, 시그널링| 웹, 이메일, 파일 전송, 대부분의 일반 TCP 서비스 |
| 전송 단위              | 메시지(datagram)                        | 바이트 스트림                                      |

---

### 2.4 실전 설계에서의 선택 기준

실제 시스템 설계 시, 어떤 전송 계층 프로토콜을 선택할지 판단할 때 보통 다음 기준을 사용한다.

1. **신뢰성이 중요한가?**  
   - 금융 거래, 파일 전송, 웹, API 호출 → TCP (혹은 TCP와 동등한 신뢰층이 필요)  
   - 약간의 손실이 허용되는 실시간 멀티미디어 → UDP + 자체 보정

2. **순서 보장이 필요한가?**  
   - 순서가 바뀌면 안 되는 텍스트/파일/영상 스트림 → TCP  
   - 순서가 바뀌어도 상관 없거나, 앱에서 직접 처리 → UDP 가능

3. **지연(latency)에 민감한가?**  
   - 몇 ms 단위의 지연 차이가 큰 의미를 가지는 실시간 게임, 음성 통화 → UDP 선호  
   - 지연보다 신뢰성이 중요한 서비스 → TCP

4. **네트워크 환경**  
   - 방화벽/라우터가 UDP를 제한하는 경우도 있어, 실제 배포 환경을 고려해야 한다.  
   - 최근에는 QUIC(UDP 위에서 동작하는 신뢰성·암호화·혼잡 제어 구현) 같은 프로토콜도 사용된다.

---

## 3. 정리

23.1 절에서 다루는 핵심은 다음과 같다.

1. **Transport Layer Services**  
   - 전송 계층은 프로세스 간 통신, 다중화/역다중화, 신뢰성, 순서, 흐름 제어, 혼잡 제어, 오류 검출을 제공한다.  
   - 포트 번호를 이용해 하나의 호스트 안에서 여러 프로세스를 구분한다.

2. **Connectionless vs Connection-Oriented**  
   - UDP: 연결 없는, 비신뢰성, 단순하고 빠른 datagram 전송  
   - TCP: 연결 기반, 신뢰성과 순서 보장, 흐름·혼잡 제어를 갖춘 바이트 스트림  
   - 실전에서는 서비스의 특성과 요구사항(신뢰성 vs 지연, 순서, 네트워크 환경)을 고려해 선택하거나, 별도의 프로토콜(예: QUIC)을 설계한다.

이 관점을 충분히 잡아두면, 이후 장(예: TCP 상세, UDP 상세, 혼잡 제어 알고리즘, 핸드셰이크/종료 절차)를 공부할 때 **“각 메커니즘이 어떤 서비스를 실현하기 위해 존재하는지”** 를 자연스럽게 연결할 수 있다.