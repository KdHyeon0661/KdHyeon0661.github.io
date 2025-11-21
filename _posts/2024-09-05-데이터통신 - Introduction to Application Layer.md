---
layout: post
title: 데이터 통신 - Introduction to Application Layer (1)
date: 2024-09-05 20:20:23 +0900
category: DataCommunication
---
# Chapter 25. Introduction to Application Layer — Services, 패러다임, 그리고 기본 클라이언트–서버 프로그래밍

앞에서 **네트워크 계층·전송 계층(TCP/UDP/SCTP)** 까지 올라왔으니, 이제 실제로 우리가 쓰는 **웹, 메일, 스트리밍, 채팅** 같은 **애플리케이션**이 어디에 어떻게 올라타는지 정리해야 한다.

이 글에서는:

- **25.1 Introduction**
  - Providing Services
  - Application Layer Paradigms
- **25.2 Client–Server Programming**
  - Application Programming Interface
  - Using Services of the Transport Layer
  - Iterative Communication using UDP
  - Iterative Communication using TCP
  - Concurrent Communication

까지를 **교재 스타일 + 실전 예제 코드**로 정리한다.
예제 코드는 주로 **POSIX 소켓(C / Python)** 기준으로 설명한다.

---

## Introduction

### Providing Services — 애플리케이션 계층이 실제로 하는 일

**애플리케이션 계층**은 말 그대로 “사용자가 직접 보는 서비스”가 구현되는 계층이다.

- HTTP(S) → 웹 브라우징, REST API
- DNS → 이름 → IP 주소
- SMTP/IMAP/POP3 → 이메일
- FTP/SFTP → 파일 전송
- QUIC/HTTP/3 위의 WebTransport, WebRTC → 실시간 브라우저 애플리케이션

이 계층의 프로토콜은 **전송 계층 위에서 동작**하면서, 다음과 같은 서비스들을 애플리케이션에 제공한다:

1. **이름 기반 통신**
   - 사용자는 `example.com` 같은 **도메인 이름**을 쓰고 싶어 한다.
   - DNS가 이를 IP 주소로 매핑해 준다.
2. **메시지 포맷 정의**
   - HTTP: 요청/응답 메시지 (메서드, 헤더, 바디)
   - SMTP: 메일 헤더(From, To, Subject…) + 바디
3. **상태 관리**
   - HTTP 세션, 쿠키, 토큰, 인증(Authorization 헤더 등)
4. **에러 처리와 의미 있는 응답**
   - HTTP 404 / 500, SMTP 실패 코드 등
5. **보안**
   - HTTPS(HTTP over TLS), HTTP/3 over QUIC에서의 TLS 1.3 통합

전송 계층(TCP/UDP/QUIC 등)이 “**데이터를 안전하게/효율적으로 보내는 법**”을 다룬다면,
애플리케이션 계층은 “**무엇을 어떻게 주고받을지**”를 정의한다고 보면 된다.

---

### 애플리케이션 계층이 전송 계층에게 요구하는 것들

각 애플리케이션 프로토콜은 전송 계층에게 조금씩 다른 요구를 한다. 보통 다음 세 가지 축으로 평가한다:

1. **신뢰성(reliability)**
   - 파일 전송, 결제 API: **손실/중복/순서 뒤틀림이 없어야** 한다.
   - 실시간 음성/영상: 약간의 손실은 허용하지만, 지연이 커지면 안 된다.
2. **지연(latency) / 지터(jitter)**
   - 게임, 화상 회의: **지연과 지터(min/max 편차)** 가 매우 중요.
   - 백업/로그 전송: 약간 느려도 괜찮지만, 안정성이 더 중요.
3. **스루풋(throughput)**
   - 대용량 VOD 스트리밍, 소프트웨어 업데이트: **초당 전송량**이 중요.

이 세 축을 동시에 최대로 만족시키는 것은 불가능해서, 프로토콜마다 **트레이드오프 선택**을 한다.

예:

- HTTP/1.1: TCP 위에서 신뢰성·순서 보장은 좋지만, **HoL(Head-of-Line) blocking** 문제로
  다수 요청에 대한 지연 문제가 생긴다.
- HTTP/2: 하나의 TCP 연결에 여러 스트림을 멀티플렉싱하지만,
  여전히 TCP 레벨에서 HoL blocking 존재.
- HTTP/3: **QUIC(UDP 기반)** 위에서 HTTP를 올려서,
  **스트림 단위의 독립적인 흐름 제어·손실 복구**를 제공, HoL 문제를 줄인다.

---

### Application Layer Paradigms — 서비스 구조 패턴

애플리케이션 계층에서 서비스가 구성되는 방식(패러다임)은 크게 세 가지로 많이 구분한다:

1. **클라이언트–서버(Client–Server)**
2. **피어-투-피어(Peer-to-Peer, P2P)**
3. **하이브리드(Hybrid: Client–Server + P2P + CDN 등)**

#### (1) Client–Server

가장 전통적인 구조:

- 항상 켜져 있고, 고정된 주소/도메인을 가진 **서버**가 중심
- 다수의 **클라이언트**가 서버에 요청을 보내고 응답을 받는다.
- 예: 웹 사이트, REST API, 전통적인 DB 서버, 메일 서버

특징:

- 중앙 집중 관리(보안/로그/업데이트 쉽다)
- 서버가 과부하되면 병목(Bottleneck)
- 확장: 수평 확장(로드 밸런서), 수직 확장(더 좋은 서버)

#### (2) Peer-to-Peer (P2P)

P2P에서는 각 노드를 **동시에 클라이언트이자 서버(peer)** 로 본다.

- 파일 또는 스트리밍 콘텐츠를 서로에게 직접 전송
- 중앙 서버는 있을 수도 있고(트래커/부트스트랩 노드), 없을 수도 있다.
- 예: BitTorrent, 일부 P2P 스트리밍, P2P 기반 overlay network 등

특징:

- 확장성: 참여 노드가 많을수록 전체 용량이 늘어남
- 관리 난이도: 노드가 항상 출입·변경되므로, **라우팅, 검색, 신뢰성**이 복잡
- IETF의 여러 P2P 관련 RFC에서 overlay 설계, 구조화/비구조화 P2P 분류 등을 정리하고 있다.

#### (3) 하이브리드 구조 — CDN·P2P·에지 컴퓨팅의 혼합

최근 대규모 서비스는 순수 Client–Server, 순수 P2P보다는 **혼합형 구조**를 많이 쓴다:

- **CDN(Content Delivery Network)**:
  - 중앙 오리진 서버 + 세계 곳곳의 캐시 서버
- **P2P+CDN** 하이브리드:
  - 기본은 CDN에서 콘텐츠를 받고,
  - 일부는 동시에 P2P로 서로 공유해 백본 트래픽을 줄임
- **Edge Computing + P2P**:
  - 에지 노드에서 캐시 및 처리,
  - P2P overlay로 peers 간 직접 전달

여러 연구 결과에서, **하이브리드 P2P-CDN 아키텍처**가 지연과 비용 측면에서 장점을 갖는다는 보고가 있다.

이런 패러다임들은 모두 **애플리케이션 계층 논리**로 구현되며,
전송 계층은 기본적인 비트 전송(UDP/TCP/QUIC)을 제공할 뿐이다.

---

### 간단한 구조 비교 표

| 패러다임 | 장점 | 단점 | 예시 |
|---------|------|------|-----|
| Client–Server | 중앙 관리, 보안/정책 일관성, 구현 용이 | 서버 병목, 인프라 비용 | 전통 웹 서비스, API 서버 |
| P2P | 높은 확장성, 자원 공유 | 설계 복잡, 보안/신뢰 어려움 | BitTorrent, P2P 스트리밍 |
| Hybrid (CDN+P2P 등) | 비용 절감 + 성능 향상 | 설계·운영 난이도 높음 | 하이브리드 스트리밍 플랫폼 |

---

## Client–Server Programming

이제 실제로 **애플리케이션 코드**가 어떻게 전송 계층 서비스를 사용하는지 살펴보자.

### Application Programming Interface — 소켓 API

현대 OS에서 네트워크 프로그래밍의 표준 인터페이스는 거의 **소켓(socket) API**다.
POSIX 계열(Linux, BSD, macOS)에서는 **Berkeley sockets**가 사실상 표준이고,
Windows도 거의 동일한 WinSock API를 제공한다.

대표적인 시스템 콜(서버 기준):

1. `socket()`
   - 통신 endpoint를 만든다. (파일 디스크립터 반환)
2. `bind()`
   - 이 소켓에 **로컬 주소(IP, 포트)** 를 할당한다.
3. (TCP일 때) `listen()`
   - 이 소켓을 **수신 대기(리슨)** 상태로 만든다.
4. (TCP일 때) `accept()`
   - 대기열에 있는 연결 요청 하나를 꺼내 **새로운 연결 소켓**을 만든다.
5. `connect()`
   - 클라이언트 측에서 서버 주소로 연결을 건다.
6. `send() / recv()` or `write() / read()`
   - 연결형(스트림) 소켓에서 데이터 송수신
7. `sendto() / recvfrom()`
   - UDP 같은 비연결형 소켓에서 데이터 송수신
8. `close()`
   - 소켓을 닫는다.

#### 서버 관점의 전형적인 흐름 (TCP)

1. `socket()`
2. `bind()`
3. `listen()`
4. 루프:
   - `accept()` 로 새 연결 수락
   - 해당 연결에 대해 `recv() / send()`로 처리
   - 종료 시 `close()`

#### 클라이언트 관점의 흐름 (TCP)

1. `socket()`
2. `connect()` 로 서버에 연결
3. `send() / recv()`
4. 끝나면 `close()`

이 API 위에 Python의 `socket` 모듈, Java의 `Socket/ServerSocket`, .NET의 `TcpClient/TcpListener` 등 **언어별 래퍼**들이 올라간다.

---

### Using Services of the Transport Layer

애플리케이션 개발자는 **전송 계층의 여러 선택지(TCP, UDP, QUIC, SCTP 등)** 에서 프로토콜을 골라 쓴다.

#### TCP 기반 소켓

- 타입: `SOCK_STREAM`
- 특징:
  - **바이트 스트림** (메시지 경계 없음)
  - 신뢰성: 손실·중복·순서 뒤틀림을 전송 계층이 해결
  - 흐름 제어, 혼잡 제어 포함
- 예:
  - HTTP/1.1, HTTP/2, SMTP, IMAP, SSH 등 대부분의 전통적 애플리케이션 프로토콜

#### UDP 기반 소켓

- 타입: `SOCK_DGRAM`
- 특징:
  - **데이터그램**(메시지 단위) 전달
  - 연결 설정 없음 (connectionless)
  - 신뢰성/재전송 없음 → 애플리케이션이 필요 시 직접 구현
- 예:
  - DNS, NTP, 일부 스트리밍, 게임, VoIP (RTP/RTCP)

#### QUIC 기반 (HTTP/3, WebTransport 등)

- **UDP 위에 사용자의 레벨에서 구현된 신형 전송 프로토콜**
- 특징:
  - 스트림 멀티플렉싱
  - 스트림 단위 흐름 제어
  - 빠른 연결 설정(0-RTT)
  - 사용자 공간 구현, TLS 1.3 통합
- HTTP/3는 QUIC 위에서 HTTP 의미론을 매핑한 것.

현대 애플리케이션은 TCP/UDP만 쓰지 않고, **QUIC, WebTransport** 같은 상위 계층 프레임워크를 통해 전송 계층 서비스를 소비하기도 한다.

---

### Iterative Communication using UDP

**Iterative 서버**란:

> “한 번에 하나의 클라이언트 요청만 처리하고,
> 그 요청을 모두 처리한 뒤 다음 요청을 받는 서버”

를 의미한다.

UDP는 **연결 개념이 없고** 서버가 그저 `recvfrom()`/`sendto()` 루프를 돌면 되므로,
자연스럽게 iterative 서버 구조를 만들기 쉽다.

#### 개념 구조

1. 서버:
   - `socket(AF_INET, SOCK_DGRAM, 0)`
   - `bind()` 로 포트에 바인딩 (예: 9999)
   - 무한 루프:
     - `recvfrom()` 으로 메시지 수신
     - 처리
     - `sendto()` 로 응답
2. 클라이언트:
   - `socket(AF_INET, SOCK_DGRAM, 0)`
   - 매번 `sendto()` 로 서버에 메시지 전송
   - `recvfrom()` 로 응답(필요 시)

UDP는 패킷 안에 발신자 주소가 들어 있으므로,
서버가 **별도의 소켓이나 연결 상태 없이도** 해당 주소로 응답을 보낼 수 있다.

#### 예제 상황: 간단한 “대문자 변환” UDP 서버 (Python)

- 서버: 받은 문자열을 대문자로 바꿔서 돌려준다.
- iterative: 한 번에 하나의 요청만 처리하지만, UDP 특성상 처리 속도가 빠르면 여러 클라이언트도 충분히 감당.

```python
# udp_upper_server.py

import socket

SERVER_IP = "0.0.0.0"
SERVER_PORT = 9999
BUF_SIZE = 2048

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind((SERVER_IP, SERVER_PORT))

print(f"[*] UDP Uppercase Server listening on {SERVER_PORT}")

while True:
    data, addr = sock.recvfrom(BUF_SIZE)  # blocking
    print(f"[+] Received from {addr}: {data!r}")

    text = data.decode("utf-8", errors="ignore")
    response = text.upper().encode("utf-8")

    sock.sendto(response, addr)
    print(f"[+] Sent back to {addr}: {response!r}")
```

클라이언트:

```python
# udp_upper_client.py

import socket

SERVER_IP = "127.0.0.1"
SERVER_PORT = 9999

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

while True:
    msg = input("> ")
    if not msg:
        break
    sock.sendto(msg.encode("utf-8"), (SERVER_IP, SERVER_PORT))
    data, _ = sock.recvfrom(2048)
    print("RECV:", data.decode("utf-8", errors="ignore"))
```

특징:

- 서버는 **한 루프에 하나의 요청**만 처리 → iterative
- UDP 특성상, 클라이언트 수가 아주 많아지고 메시지가 복잡해지면:
  - 패킷 손실/순서 문제를 직접 처리해야 한다.
  - 더 고급 구조(타임아웃, 재전송, 시퀀스 번호)를 직접 코드로 작성해야 한다.

---

### Iterative Communication using TCP

TCP는 **연결 기반**이기 때문에, iterative 서버는 “한 번에 하나의 연결만 처리하는” 구조가 된다.

#### 개념 구조

1. 서버:
   - `socket(AF_INET, SOCK_STREAM, 0)`
   - `bind()`
   - `listen()`
   - 무한 루프:
     - `accept()` 로 **연결 하나 수락** (blocking)
     - 이 연결에서 `recv()/send()`로 요청 처리
     - 연결 종료 후 `close()`
   - 그 다음 다시 `accept()` 로 돌아감
2. 클라이언트는 평범한 TCP 클라이언트 (`socket + connect + send/recv`)

이 구조에서는 **다른 클라이언트가 접속을 해도, 서버가 현재 처리 중인 클라이언트가 끝날 때까지 기다려야** 한다. 그래서 트래픽이 많지 않고, 처리 시간이 매우 짧은 경우에나 실용적이다.

#### 예제 상황: iterative TCP 에코 서버 (Python)

```python
# tcp_echo_server_iterative.py

import socket

HOST = "0.0.0.0"
PORT = 8888
BUF_SIZE = 4096

srv_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
srv_sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv_sock.bind((HOST, PORT))
srv_sock.listen(5)

print(f"[*] Iterative TCP Echo Server listening on {PORT}")

while True:
    conn, addr = srv_sock.accept()
    print(f"[+] New connection from {addr}")

    # 이 연결에서만 루프를 돈다 → iterative
    while True:
        data = conn.recv(BUF_SIZE)
        if not data:
            break
        print(f"[+] Received from {addr}: {data!r}")
        conn.sendall(data)  # echo back

    print(f"[-] Connection closed: {addr}")
    conn.close()
```

클라이언트는:

```python
# tcp_echo_client.py

import socket

HOST = "127.0.0.1"
PORT = 8888

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect((HOST, PORT))

while True:
    msg = input("> ")
    if not msg:
        break
    sock.sendall(msg.encode("utf-8"))
    data = sock.recv(4096)
    print("RECV:", data.decode("utf-8", errors="ignore"))

sock.close()
```

단점:

- 하나의 클라이언트가 **오래 걸리는 작업**을 요청하면,
  - 다른 클라이언트들은 연결 시도에서 타임아웃이 나거나
  - accept 큐에서 오래 기다리게 된다.

그래서 **대부분의 실제 서버**는 iterative 구조 대신에, 다음 절의 **Concurrent 서버**를 사용한다.

---

### Concurrent Communication

**Concurrent 서버**는 “여러 클라이언트와 동시에 통신”하는 서버다. 구현 방식은 크게 세 가지:

1. 프로세스 기반(concurrent server with `fork()` 등)
2. 스레드 기반(concurrent server with threads)
3. 이벤트 기반(Non-blocking I/O + `select`/`epoll` + 이벤트 루프)

#### 개념 비교: Iterative vs Concurrent

| 유형 | 처리 방식 | 장점 | 단점 |
|------|-----------|------|------|
| Iterative | 한 번에 하나의 연결만 처리 | 구현 간단, 디버깅 쉬움 | 동시성 X, 작은 실험용에만 적합 |
| Concurrent (Process/Thread) | 연결마다 프로세스/스레드 생성 | 코드 구조가 직관적, 개별 요청 간 독립 | 많은 동시 접속 시 문맥 전환/자원 오버헤드 |
| Concurrent (Event-driven) | 하나(or 소수)의 스레드에서 다수 연결 관리 | 매우 많은 연결에 효율적, modern high-performance 서버 구조 | 상태 머신 작성이 복잡, 디버깅 어려움 |

#### 프로세스 기반 Concurrent TCP 서버 (C 개념 코드)

여기서는 POSIX `fork()` 를 사용해, **각 클라이언트를 별도 자식 프로세스**가 처리하도록 하는 구조를 보자.

> 실제 컴파일 가능한 예제는 Beej’s Guide나 RPI/CMU 강의 자료에서 많이 볼 수 있다.

개념 코드(핵심 구조만):

```c
int main(void) {
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    // bind(), listen() ...

    while (1) {
        int connfd = accept(listenfd, (struct sockaddr *)&cliaddr, &len);

        pid_t pid = fork();
        if (pid == 0) {
            // child
            close(listenfd);
            handle_client(connfd);  // 블로킹 처리
            close(connfd);
            exit(0);
        } else {
            // parent
            close(connfd);  // 자식에게 맡겼으므로 부모 쪽은 닫음
        }
    }
}
```

특징:

- 각각의 클라이언트는 **자식 프로세스**가 담당 → 서로 독립적인 주소 공간
- OS가 프로세스 스케줄링을 통해 동시 실행처럼 보이게 한다.
- 많은 클라이언트가 동시에 접속해도, 기본적으로 **동시 처리** 된다.
- 단, 프로세스는 무겁기 때문에 요청 수가 매우 많아지면 부담.

#### 스레드 기반 Concurrent TCP 서버 (Python 예제)

Python의 `threading` 모듈을 사용해도 구조를 쉽게 잡을 수 있다.

```python
# tcp_echo_server_threads.py

import socket
import threading

HOST = "0.0.0.0"
PORT = 8888
BUF_SIZE = 4096

def handle_client(conn, addr):
    print(f"[+] New thread for {addr}")
    with conn:
        while True:
            data = conn.recv(BUF_SIZE)
            if not data:
                break
            print(f"[{addr}] {data!r}")
            conn.sendall(data)
    print(f"[-] Connection closed: {addr}")

srv_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
srv_sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv_sock.bind((HOST, PORT))
srv_sock.listen(50)

print(f"[*] Threaded TCP Echo Server listening on {PORT}")

while True:
    conn, addr = srv_sock.accept()
    t = threading.Thread(target=handle_client, args=(conn, addr), daemon=True)
    t.start()
```

특징:

- 각 연결마다 스레드 1개 → 코드 구조는 iterative 서버와 거의 같아서 쉽게 이해 가능.
- 스레드 수가 수천·수만으로 가면 컨텍스트 스위칭 비용과 메모리 사용 증가.
- 따라서 대규모 서비스는 이벤트 기반 구조나, 비동기 I/O (`asyncio`, epoll) 를 사용한다.

#### Event-driven 서버 개념

이벤트 기반 서버는 **하나의(또는 소수의) 스레드**가 다음을 반복한다:

1. `select()` / `poll()` / `epoll()` / `kqueue()` 로 “어떤 소켓이 읽기/쓰기 가능한지” 감시
2. 준비된 소켓들에 대해 **조금씩** 처리
3. 상태는 메모리 내 상태 머신으로 관리

이 구조는:

- **수 만~수 십 만** 단위의 동시 연결을 처리하기에 적합(현대 고성능 HTTP 리버스 프록시 등)
- 대신 코드가 복잡하고, 디버깅이 어려울 수 있다.

Node.js, Nginx, Envoy, 일부 HTTP/3/QUIC 서버 구현들이 이런 이벤트 기반 구조 위에 서 있다.

---

### UDP vs TCP vs QUIC: 애플리케이션 코드 관점 요약

| 전송 | 소켓 타입 | 연결 | 순서/신뢰성 | 애플리케이션 책임 | 예시 |
|------|-----------|------|-------------|-------------------|------|
| TCP  | SOCK_STREAM | 필요 (`connect`) | 제공 (스트림 수준) | 메시지 경계를 잡아줘야 함(프레이밍, 길이 헤더 등) | HTTP/1.1/2, SMTP, DB 프로토콜 |
| UDP  | SOCK_DGRAM  | 없음 | 없음        | 재전송, 순서, 손실 허용 여부 등 **직접 설계** | DNS, VoIP, 게임 |
| QUIC | (사용자 공간) | 필요 (최초 핸드셰이크) | 스트림 단위 제공 | 보통 라이브러리/프레임워크가 프레이밍까지 제공 | HTTP/3, WebTransport, 일부 새로운 서비스 |

애플리케이션 계층에서 “**어떤 성질의 데이터를 어떻게 보내고 싶냐**”에 따라,
위 세 가지 중 무엇을 선택할지 결정한다.

---

## 마무리 정리

이 글에서 다룬 핵심 포인트를 다시 정리하면:

1. **Application Layer의 역할**
   - 실제 서비스(웹·메일·DNS·스트리밍 등)를 정의하고,
     메시지 포맷/상태/에러 처리/보안 정책을 담당한다.
2. **애플리케이션 패러다임**
   - Client–Server, P2P, Hybrid 구조가 있으며,
     현대 인터넷 서비스는 보통 **Hybrid + CDN + Edge + P2P** 식으로 구성된다.
3. **소켓 API 기반 클라이언트–서버 프로그래밍**
   - `socket`, `bind`, `listen`, `accept`, `connect`, `send/recv`, `sendto/recvfrom` 등으로
     전송 계층(TCP/UDP/QUIC)의 서비스를 소비한다.
4. **Iterative vs Concurrent 서버**
   - Iterative: 구현은 단순하지만 동시성이 없다.
   - Concurrent: 프로세스/스레드/이벤트 기반으로 다수 클라이언트를 동시에 처리한다.
5. **UDP·TCP·QUIC 선택**
   - 신뢰성·지연·스루풋 요구사항을 보고,
     UDP(가벼운 비신뢰), TCP(강한 신뢰), QUIC(유연한 스트림 기반 신뢰) 중에서
     또는 그 조합(예: QUIC 위의 HTTP/3)을 선택한다.
