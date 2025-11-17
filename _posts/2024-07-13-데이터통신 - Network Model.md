---
layout: post
title: 데이터 통신 - Network Model
date: 2024-07-13 19:20:23 +0900
category: DataCommunication
---
# Protocol Layering

본 글은 당신의 원고를 바탕으로 **핵심을 보존**하면서, 실전 예제·수식·코드·ASCII 도식으로 **깊이와 정확성**을 보강한 확장판입니다. 모든 이미지 링크는 요청에 따라 **삭제**했습니다.

---

## Scenarios (왜 레이어링인가)

레이어링은 **복잡한 통신 기능을 계층별로 분해**하여, 각 계층이 **명확한 역할**만 수행하게 만드는 설계 원칙입니다. 그 결과:

- **모듈성**: 한 계층 수정이 다른 계층에 최소 영향.
- **은닉화**: 내부 동작은 블랙박스처럼 감추고 **인터페이스**만 공개.
- **상호운용성**: 표준 인터페이스를 지키면 구현/벤더가 달라도 호환.

간단한 비유:

```
[App] 메시지 생성/해석
[Trans] 세션/순서/복구
[Net] 경로 선택, 주소 지정
[Link] 프레임화/에러검출/접근제어
[Phys] 비트 전송
```

**중간 장비(라우터/스위치/방화벽)는 상위 응용 내용을 보지 않아야** 보안·확장성 측면에서 유리합니다(필요 최소한의 헤더만 처리).

---

## Principles of Protocol Layering (레이어링 원칙)

- **양방향성(Bidirectional)**: 각 계층은 **송·수신** 모두를 처리.
- **일관된 계층 구조(Homogeneous Stacking)**: 통신 양단은 **동일한 계층 스택**을 공유(해당 계층의 의미·PDU 형식의 합의).
- **최소 책임(Minimal Responsibility)**: 계층 간 **책임 중복 최소화**—예: 재전송은 전송계층에서, 링크의 단기 오류는 링크계층에서.
- **서비스·프로토콜 분리**:
  - **서비스(Service)**: 상위에 제공하는 기능(추상 인터페이스).
  - **프로토콜(Protocol)**: **같은 계층**의 동등체(peer) 간 규칙/형식.

---

## Logical Connections (논리적 연결)

논리적 연결은 **동일 계층의 동등체(peer)** 사이에서 개념적으로 형성됩니다.

```
 App ↔ App   (메시지 의미의 합의)
 Trans ↔ Trans (세그먼트 순서/복구)
 Net ↔ Net   (IP 라우팅/주소)
 Link ↔ Link (프레임 전달, 인접 노드)
 Phys ↔ Phys (신호/비트)
```

물리적으로는 **인접 홉**을 따라 전파되지만, 개념적으로는 동등체끼리 통신한다고 **가정**함으로써 설계를 단순화합니다.

---

# TCP/IP Protocol Suite

## Layered Architecture (스택 구성)

현대 인터넷 스택을 실용적으로 5계층으로 나타내면:

```
[Application]
[Transport]
[Network]
[Data Link]
[Physical]
```

- **엔드 시스템(호스트)**: 1~5계층 **모두** 탑재.
- **라우터**: 1~3계층(일반적으로 L3까지), 일부는 L4/L7 기능을 하는 **미들박스** 존재(로드밸런서, 방화벽 등).
- **스위치/브리지**: 주로 1~2계층.

> 중간 경로에서 응용 데이터(컨텐츠)를 열람·변조하지 않는 것이 원칙(종단 간 원칙). 필요한 경우에도 **최소 정보**만 다루는 것이 바람직합니다.

---

## End-to-End vs Hop-to-Hop

- **Application/Transport/Network**: (기능의 관점에서) **종단 간** 의미를 형성.
  - 예: TCP의 재전송·순서 보장은 **종단**에서 성립.
- **Data Link/Physical**: **홉 간** 전달(인접 노드 사이).
  - 예: 이더넷의 FCS 오류검출은 **각 홉**에서 수행.

---

## Description of Each Layer (계층 상세)

### Physical (물리)

- 신호로 **비트 스트림** 전송: 전기/광/무선.
- 주파수 대역·변조·라인코딩·동기화.
- 예: UTP(1000BASE-T), 광(SR/LR), OFDM(무선).

### Data Link (데이터 링크)

- **프레임화**, **MAC 주소**, **접근제어(CSMA/CA/TDMA)**.
- **오류검출(CRC)**, 일부 환경에서 단기 오류 복구(재전송).
- **Hop-to-Hop** 전달—인접 노드까지 **무결성** 보장.

### Network (네트워크)

- **IP**(IPv4/IPv6): **라우팅**, **논리 주소**.
- 비연결·Best Effort, 경로 선택(OSPF/IS-IS/BGP).
- ICMP로 오류·진단(예: TTL Exceeded).

### Transport (전송)

- **UDP**: 단순·비연결, 지연 민감/스트리밍에 적합.
- **TCP**: 연결지향, **3-way handshake**, 순서/흐름/혼잡 제어, **신뢰성**.
- **SCTP**: 멀티스트림/멀티홈, 메시지 지향(**Chunk** 기반).

### Application (응용)

- HTTP(S), DNS, SMTP, SSH, FTP, SNMP, RTP/RTCP, **SIP(세션 시그널링)** 등.
- 종종 세션/프레젠테이션 기능(암호화/직렬화)을 포함하여 **상위 OSI 계층 역할**을 흡수.

---

## Encapsulation & Decapsulation (캡슐화/역캡슐화)

각 계층은 상위 계층 PDU 앞에 **자신의 헤더(필요 시 트레일러)** 를 붙입니다.

```
App:        [DATA]
Transport:  [TCP hdr][DATA]
Network:    [IP hdr][TCP hdr][DATA]
Link:       [ETH hdr][IP hdr][TCP hdr][DATA][FCS]
Physical:   ...비트/신호...
```

라우터의 동작:

1) **프레임 수신 → FCS 검사 → 링크 헤더 제거**
2) **IP 헤더 검사(목적지/TTL/체크섬) → 라우팅 결정**
3) **새 링크 헤더 생성 → 다음 홉으로 전송**

수식 — 헤더 오버헤드(간단 근사):

$$
\text{Efficiency} \approx \frac{L_\text{payload}}{L_\text{payload} + L_\text{L2} + L_\text{L3} + L_\text{L4}}
$$

예: 이더넷(14B) + IPv4(20B) + TCP(20B), 페이로드 1460B → 효율 ≈ 1460/(1460+54) ≈ 96.4%

---

## Addressing (주소 계층)

- **Link(MAC)**: 동일 **링크/브로드캐스트 도메인**에서 장치 식별.
  예: `00:1A:2B:...`
- **Network(IP)**: 전 지구적 **라우팅 가능한 논리 주소**.
  예: `203.0.113.7`, `2001:db8::7`
- **Transport(Port)**: 동일 호스트 내 **프로세스/소켓 식별**.
  예: `(IP, Port)` 쌍 → `(203.0.113.7, 443)`
- **Application(Name/URL)**: 사람 친화적 이름.
  예: `www.example.com`, `https://...`

**소켓 5-튜플**으로 흐름 식별:

```
(src IP, src Port, dst IP, dst Port, Transport)
```

---

## Multiplexing / Demultiplexing (다중화/역다중화)

- **Multiplexing(송신)**: 다양한 응용의 데이터가 **하나의 하위 계층**으로 내려오며, **포트/프로토콜 번호**로 구분되도록 캡슐화.
- **Demultiplexing(수신)**: 수신 측에서 **헤더의 식별자**(예: IP proto=6, dst port=443)를 읽어 **올바른 상위 엔드포인트**로 분배.

파이썬 미니 예제—UDP 역다중화 감각:

```python
import socket, threading

def serve(port):
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    s.bind(("0.0.0.0", port))
    print(f"UDP server on {port}")
    while True:
        data, addr = s.recvfrom(2048)
        s.sendto(f"[{port}] {data.decode()}".encode(), addr)

for p in (9001, 9002):
    threading.Thread(target=serve, args=(p,), daemon=True).start()

# 클라이언트는 포트 번호로 원하는 "상위 애플리케이션"을 구분
# $ echo hi | nc -u 127.0.0.1 9001
# $ echo hi | nc -u 127.0.0.1 9002

```

---

# The OSI Model

> OSI(Open Systems Interconnection) **기준 참조 모델**은 **ISO**가 1980년대에 제정(참조 모델은 1984년 표준화)한 **7계층** 이론적 프레임워크입니다. (※ ISO는 1947년 설립, OSI 모델 자체가 1947년에 제정된 것은 아닙니다.)

```
7 Application
6 Presentation
5 Session
4 Transport
3 Network
2 Data Link
1 Physical
```

현대 인터넷은 **TCP/IP 모델**을 실무 표준으로 채택하지만, OSI 7계층은 **교육/개념 정리**에 널리 사용됩니다.

---

## OSI vs TCP/IP (대응 관계와 차이)

대응(대표적 매핑):

| OSI | TCP/IP | 역할 요지 |
|---|---|---|
| 7 응용 | Application | HTTP/DNS/SMTP/SSH… |
| 6 표현 | (Application에 흡수) | 암호화/압축/직렬화 등 |
| 5 세션 | (Application/Transport에 분산) | 세션 관리/동기화 |
| 4 전송 | Transport | TCP/UDP/SCTP |
| 3 네트워크 | Network | IP/라우팅/ICMP |
| 2 데이터링크 | Data Link | 이더넷/802.11/PPP |
| 1 물리 | Physical | 신호/매체/속도 |

차이 핵심:

- **TCP/IP의 상위(응용) 계층은 OSI의 표현·세션 기능을 상당 부분 흡수**. 예컨대 TLS(암호화)는 “표현 계층” 개념에 해당하나 TCP/IP에서는 응용/전송 사이에 배치.
- 전송 계층의 **프로토콜 다양성**(TCP/UDP/SCTP/QUIC*)은 OSI 관점에선 전송/세션/표현의 경계를 흐리기도 함.
  (*QUIC은 전송 성격을 갖지만 사용자 공간·UDP 상에서 동작)

---

## Why OSI Didn’t Win (OSI 모델의 한계와 비성공 요인)

- **시장 타이밍과 구현 복잡성**: TCP/IP가 **이미 광범위하게 구현·배포**되는 동안 OSI 기반 스택과 프로토콜은 **무겁고 복잡**하다는 인식을 받음.
- **상호운용/성능 이슈**: 벤더마다 다른 해석·프로파일, **고성능 구현 난도**.
- **개방 생태계의 차이**: IETF(RFC) 중심의 빠른 반복/구현 vs. OSI 스펙의 상대적 **표준화 지연**.
- **현실적 융합**: 오늘날은 OSI의 **개념적 계층화**와 TCP/IP의 **실용 모델**을 함께 사용.

---

# 실전 추가: 캡슐화/다중화/주소처리 깊이 파보기

## A. PDU 명칭과 디버깅 포인트

- L4: **Segment**(TCP)/**Datagram**(UDP)
- L3: **Packet**(IP)
- L2: **Frame**(Ethernet/802.11)

디버깅은 **아래에서 위로**(신호→링크→IP→TCP/UDP→App) 혹은 **위에서 아래로**(앱 오류→패킷 흐름) 접근.

## B. 3-Way Handshake & 종료

간단 도식:

```
SYN(seq=x)  --->
           <---  SYN+ACK(ack=x+1, seq=y)
ACK(ack=y+1) --->
```

연결 종료는 보통 **4-way FIN/ACK**, 또는 예외적으로 RST.

## C. MTU·MSS·단편화

- **MTU**(L2 최대 프레임 페이로드), **MSS**(TCP 페이로드 상한).
- Path MTU가 작으면 **IP 단편화**(IPv4) 또는 **PMTUD** 필수.
- 효율 근사(IPv4+TCP, MSS=1460 기준):

$$
\text{Goodput} \approx \text{Throughput} \times \frac{\text{MSS}}{\text{MSS} + 40}
$$

## D. 포트와 다중화

- **Well-known**(0–1023), **Registered**(1024–49151), **Dynamic**(49152–65535).
- 동일 호스트에서 여러 애플리케이션이 **포트**로 **역다중화**.

간단 서버(파이썬, TCP 멀티스레드):

```python
import socket, threading

def handle(c, addr):
    with c:
        while data := c.recv(2048):
            c.sendall(b"echo:" + data)

def serve(port):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind(("0.0.0.0", port)); s.listen()
    print("listen", port)
    while True:
        c, addr = s.accept()
        threading.Thread(target=handle, args=(c,addr), daemon=True).start()

for p in (8080, 9090):
    threading.Thread(target=serve, args=(p,), daemon=True).start()

# 서로 다른 포트(애플리케이션)로 역다중화됨

```

---

# 퀵 실습: 캡슐화 관찰하기

## 패킷 캡처 필터

```bash
# TCP 443(HTTPS) 트래픽 캡처

sudo tcpdump -i eth0 -n -vv 'tcp port 443' -w tls.pcap
```

- 캡슐화 단계별 헤더를 **Wireshark**에서 관찰(L2/L3/L4/App).

## ICMP와 라우팅 (Hop-to-Hop 확인)

```bash
traceroute 1.1.1.1
# 각 홉에서 TTL 감소/ICMP Time Exceeded 응답으로 경로 확인

```

---

# 체크 문제

1) 아래 중 **종단 간** 의미가 성립되는 계층을 모두 고르시오.
   `Application / Transport / Network / Data Link / Physical`
   **정답**: Application, Transport, Network(의미 관점에서).
2) TCP MSS=1200, IPv4+TCP 헤더 40바이트. **오버헤드 비율**은?
   $$ \text{Overhead} = \frac{40}{1200+40} \approx 3.23\% $$
3) 스위치가 목적지 MAC을 모를 때 수행하는 동작은?
   **플러딩**(수신 포트 제외 모든 포트로 전송).
4) 소켓 5-튜플의 역할은?
   **흐름의 고유 식별자**(다중 연결의 구분자).

---

# 요약

- **레이어링**은 설계 복잡도를 낮추고 상호운용성을 높인다.
- **End-to-End vs Hop-to-Hop** 구분은 구현/디버깅에 필수.
- **캡슐화/역캡슐화**는 네트워크가 데이터를 운반하는 **표준 메커니즘**.
- **주소 계층**(MAC/IP/Port/Name)과 **다중화**는 한 호스트에서 수많은 플로우가 공존하도록 만든다.
- **OSI는 개념 프레임**, **TCP/IP는 실무 표준**—둘을 함께 이해하자.
