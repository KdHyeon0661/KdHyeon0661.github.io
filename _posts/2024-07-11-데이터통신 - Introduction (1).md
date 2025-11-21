---
layout: post
title: 데이터 통신 - Introduction (1)
date: 2024-07-11 19:20:23 +0900
category: DataCommunication
---
# Data Communication

## Data Communication (데이터 통신)

두 장치(노드) 간에 유·무선 매체를 통해 데이터를 **전송·교환**하는 과정.

### 통신의 4요소 (Delivery · Accuracy · Timeliness · Jitter)

- **전달성(Delivery)**: 올바른 목적지에 도달해야 한다.
  - 주소지정: MAC(링크 계층), IP(네트워크 계층), 포트(전송 계층)
  - 라우팅/스위칭: 경로 선택 및 프레임 전달
- **정확성(Accuracy)**: 전송 중 오류 없이 도착해야 한다.
  - 오류검출/정정: 패리티, CRC, 해밍, ARQ(Stop-and-Wait, Go-Back-N, Selective Repeat)
- **적시성(Timeliness)**: 지연(latency)이 요구 수준 이하여야 한다.
  - 실시간(음성/영상/제어) 트래픽은 **지터(Jitter)** 제약이 크다.
- **지터(Jitter)**: 연속 패킷의 **도착 간격 변동**.
  - 플레이아웃 버퍼로 흡수하지만, 지터가 크면 **끊김·순서 왜곡** 발생.

#### 지연/지터 정량화 예시

- **종단 간 지연**
  $$ \text{Latency} = \text{Propagation} + \text{Transmission} + \text{Queuing} + \text{Processing} $$
- **전송 시간**
  $$ \text{Tx Time} = \frac{\text{Packet Size (bits)}}{\text{Link Rate (bps)}} $$
- **대역폭-지연 곱 (BDP)**: 파이프에 **동시에 떠 있는 데이터량**
  $$ \text{BDP} = \text{Bandwidth} \times \text{RTT} $$
- **지터(단순 근사)**
  $$ \text{Jitter} \approx \text{StdDev}(\Delta t_i) \quad \text{where } \Delta t_i = \text{arrival}_{i} - \text{arrival}_{i-1} $$

**현업 팁**: WAN에서 TCP 성능이 낮다면 **윈도우 크기**가 BDP 이상인지 확인. QoS(우선순위/대기열)로 지터 민감 트래픽(VoIP/Video)을 따로 다룬다.

---

## Components (구성 요소)

데이터 통신을 이루는 5가지 구성 요소를 **현대 네트워크 관점**으로 풀어쓴다.

1) **Message (메시지)**
- 텍스트, 숫자, 이미지, 오디오, 비디오, 바이너리(파일/패킷) 등.
- 응용계층에서 **직렬화(JSON/Protobuf/Avro)** 를 거쳐 전송.

2) **Sender (송신자)**
- 엔드 시스템(PC/서버/스마트폰/IoT), 퍼블리셔(메시지 브로커의 생산자) 등.
- 전송 시 **세그먼트/프레임화**, **암호화/TLS** 적용 가능.

3) **Receiver (수신자)**
- 구독자/소비자(Consumer), 서버, 엣지 디바이스.
- **순서 재조립**, **오류복구**, **디중복**(idempotency 키) 등 수행.

4) **Transmission Medium (전송 매체)**
- 유선: UTP/동축/광(싱글/멀티모드). 무선: RF(와이파이/셀룰러), 위성.
- 특성: 대역폭, 감쇠/잡음, 지연(전파속도), 반사/다중경로, BER(Bit Error Rate).

5) **Protocol (프로토콜)**
- 표준화된 **규칙·형식·절차**.
- 예: 이더넷(프레임링), IP(주소/라우팅), TCP(연결/흐름/혼잡), UDP(비연결), TLS(암호화), HTTP/2·3(스트리밍/헤더압축/QUIC), RTP/RTCP(미디어), MQTT/AMQP(Kafka/브로커).

### 실무 시나리오: “라이브 주가 스트리밍”

- **메시지**: 종목 코드, 가격, 타임스탬프(밀리·마이크로초)
- **송신자**: 거래소 게이트웨이(UDP 멀티캐스트 또는 QUIC)
- **매체**: 전용 광회선(DC 간) + 퍼블릭 인터넷(클라이언트)
- **프로토콜**: 백본은 UDP(RTP), 클라이언트 구간은 **QUIC(HTTP/3)** 로 암호화+복구
- **지터 관리**: 플레이아웃 버퍼 50–150ms, 손실 시 FEC 또는 보간

---

## Data Representation (데이터 표현)

### Text (텍스트)

- 코드체계: **ASCII(7-bit)**, **Unicode(UTF-8/16)**.
- UTF-8은 가변길이. 영문은 1바이트로 효율적, 한글/이모지는 다바이트.

```python
# 예제: UTF-8/UTF-16 인코딩 차이

s = "데이터"
print(len(s.encode("utf-8")))   # 바이트 길이
print(len(s.encode("utf-16")))  # BOM 포함 길이
```

### Numbers (숫자)

- **이진수** 표현, 2의 보수(정수), IEEE 754(실수).
- 엔디언(Little/Big) 이슈에 주의 — 네트워크 바이트 오더는 **Big Endian**.

```python
# 예제: 정수의 네트워크 바이트 오더 변환

import struct
x = 0x12345678
net = struct.pack("!I", x)  # !: network(big-endian), I: unsigned int
host = struct.unpack("!I", net)[0]
assert x == host
```

### Image (이미지)

- **픽셀 래스터**: RGB/YCbCr/Grayscale, 알파 채널.
- 압축: 무손실(PNG), 손실(JPEG, WebP).
- 전송 시 **색공간/프로파일**(sRGB) 명시가 중요.

### Audio (오디오)

- 아날로그 → **표본화**(샘플링) & **양자화**(비트 폭).
- 나이키스트:
  $$ f_s \ge 2 f_\text{max} $$
- 예: 44.1kHz/16bit 스테레오 PCM. 전송은 AAC/Opus 같은 코덱 활용.

### Video (비디오)

- 프레임(초당 24~120fps), 해상도, 색공간(4:2:0).
- 인코딩: H.264/H.265/AV1. **GOP**(I/P/B 프레임), **비트레이트 제어**(CBR/VBR).

---

## Data Flow (데이터 흐름)

### Simplex (단방향)

- 한쪽 방향으로만 전송(예: 센서→수집기, 방송).
- **장점**: 단순/효율적. **단점**: 피드백 불가(ARQ 미사용).

### Half-Duplex (반이중)

- 양방향 가능하나 **동시 전송 불가**(무전기).
- 충돌 회피/전환 프로토콜 필요(토큰 기반/RTS-CTS 등).

### Full-Duplex (전이중)

- **동시에 양방향** 전송(전화, 스위치 이더넷).
- 물리적/논리적 분리(쌍선 분리, 주파수 분할, 시분할).

#### ASCII 예시: Half vs Full Duplex

```
[Half-Duplex]
A ---> B   (A가 말할 때 B는 듣기만)
A <--- B   (역방향은 전환 후)

[Full-Duplex]
A <---> B  (동시에 송수신)
```

---

# Networks

## Network Criteria (네트워크 기준)

### Performance (성능)

- **처리량(Throughput)**:
  $$ \text{Throughput} = \frac{\text{성공적으로 전달된 비트 수}}{\text{시간}} $$
- **지연(Latency), 지터(Jitter), 손실률(Loss Rate)**, **사용률(Utilization)**.
- **영향 요소**
  - 사용자 수/트래픽 패턴(버스트/평균), 프로토콜 오버헤드(MSS/MTU), RTT
  - 매체: UTP/동축/광/무선(감쇠/간섭), 장비 성능(스위치/라우터/방화벽)
  - 소프트웨어 스택: 커널 네트워킹(Zero-Copy), NIC 오프로드(TSO, LRO), 멀티큐

**실험 코드: 지터/손실이 처리량에 미치는 영향 (간이 시뮬레이션)**

```python
import random, statistics

def simulate(n=1000, base_gap_ms=10, jitter_ms=5, loss_prob=0.02):
    arrivals = []
    t = 0.0
    received = 0
    for _ in range(n):
        t += base_gap_ms + random.uniform(-jitter_ms, jitter_ms)
        if random.random() > loss_prob:
            arrivals.append(t)
            received += 1
    gaps = [arrivals[i]-arrivals[i-1] for i in range(1, len(arrivals))]
    return {
        "recv": received,
        "avg_gap_ms": sum(gaps)/len(gaps),
        "jitter_ms": statistics.pstdev(gaps),
        "throughput_pktps": 1000.0 / (sum(gaps)/len(gaps))
    }

print(simulate())
```

### Reliability (신뢰도)

- **MTBF(평균고장간격)**, **MTTR(평균복구시간)**.
  가용도(Availability):
  $$ A = \frac{\text{MTBF}}{\text{MTBF} + \text{MTTR}} $$
- **지점별 중복**(링크/장비/전원), **경로 다양성**, **Failover/VRRP/HSRP**, **ECMP**.

### Security (보안)

- **CIA**(기밀성·무결성·가용성).
- 위협: 무단 접근, 스푸핑/스니핑, MITM, DoS/DDoS, 악성코드, 공급망.
- 대책: **암호화(TLS/IPsec)**, **인증·인가(OAuth2, mTLS)**, **네트워크 분리**, **WAF/IDS/IPS**, **제로트러스트**.

---

## Physical Structures (물리적 구조)

### Type of Connection (연결 유형)

- **Point-to-Point (점대점)**: 두 노드가 직접 연결.
  - 예: 서버↔스위치 업링크, 전용회선, 직결 DAC/광모듈.
  - **장점**: 예측 가능 지연/대역폭. **단점**: 확장성 낮음(연결 수 증가).
- **Multipoint (다중 접속)**: 하나의 매체를 여러 노드가 공유.
  - **시분할(TDMA)**: 시간 슬롯 단위 공유.
  - **공간분할(SDMA)**: 안테나/빔포밍 등으로 공간적으로 분리.

#### ASCII 예시

```
Point-to-Point
 A -------- B

Multipoint (Shared Bus/Medium)
 A --+
     +-- M --+-- C
 B --+       +-- D
```

### Physical Topology (물리적 토폴로지)

#### Mesh Topology (그물망형)

- 각 노드가 다수의 노드와 직접 연결(완전/부분 메쉬).
- **연결 수(완전연결)**:
  $$ \frac{n(n-1)}{2} $$
- **장점**: 경로 다양성, 장애 격리 우수.
- **단점**: 회선/포트 비용 급증, 설계 복잡도.

**현업 예시**: 데이터센터 Spine-Leaf는 **부분 메쉬**(풀매시 아님). ECMP로 다경로 부하분산.

#### Star Topology (성형)

- 중앙 **허브/스위치**를 통해 통신.
- **장점**: 관리 용이, 확장/장애 격리 쉬움.
- **단점**: **중앙 장비 단일 장애점**(HA 필요: 스택/MLAG/VRRP).

#### Bus Topology (버스형)

- 하나의 공통 매체(동축/공유버스)에 여러 노드 연결.
- **장점**: 배선 단순, 장치 추가 쉬움.
- **단점**: 충돌(CSMA/CD), 대역폭 공유로 확장성 제한.

#### Ring Topology (링형)

- 인접 노드 간 고리 구조. 토큰링/SONET/SDH 등.
- **장점**: 일정한 경로, **보호 절체**(링 보호 스위칭).
- **단점**: 삽입/제거 작업 난이도, 한 구간 장애 시 우회 필요.

#### Tree / Hybrid / Wireless

- **Tree**: 스타의 계층화(캠퍼스/엔터프라이즈 코어-디스트리뷰션-엑세스).
- **Hybrid**: 스타+링 혼합, 이중화 경로.
- **Wireless**: 인프라(AC/AP), 애드혹/메시(WMN).
  - 메시에선 링크 품질/간섭/채널 재사용이 설계를 좌우.

### 토폴로지 비교 요약

| 토폴로지 | 설치 난이도 | 확장성 | 장애 격리 | 비용 | 대표 사례 |
|---|---|---|---|---|---|
| Mesh | 높음 | 중 | 매우 좋음 | 매우 높음 | DC Spine-Leaf(부분) |
| Star | 낮음 | 높음 | 보통(중앙 의존) | 중 | 사무실/캠퍼스 |
| Bus | 낮음 | 낮음 | 낮음 | 낮음 | 구형 LAN/특수 현장 |
| Ring | 중 | 중 | 중 | 중 | 전송망/산업 |
| Tree | 중 | 높음 | 보통 | 중 | 엔터프라이즈 LAN |

---

# 실전 확장: 정확성(오류 제어)·흐름/혼잡 제어·QoS

## 오류 검출·정정

- **패리티**: 저비용 검출(단일 비트 오류만).
- **CRC**: 다항식 기반 강력 검출(이더넷 FCS).
- **해밍 코드**: 단일 비트 **정정**(SEC), 이중 비트 **검출**(DED).

### 간단 예제

$$
d_1d_2d_3d_4 \rightarrow c_1c_2d_1c_3d_2d_3d_4
$$

오류 위치를 패리티 검사로 탐지, 단일 비트 뒤집어 복구.

```python
# 예제

def add_parity(bits):
    p = sum(bits) % 2
    bits.append(p)  # 짝수 패리티
    return bits

def check_parity(bits):
    return sum(bits) % 2 == 0

msg = [1,0,1,1]
tx = add_parity(msg.copy())
rx = tx.copy()
rx[1] ^= 1  # 전송 중 1비트 오류 유발
print("valid?", check_parity(rx))  # False -> 재전송(ARQ)
```

## & 혼잡 제어(Congestion Control)

- **흐름 제어**: 송신 속도 ≤ 수신 처리능력(버퍼 오버런 방지).
  - Stop-and-Wait, 슬라이딩 윈도우
- **혼잡 제어**: 네트워크 내부 혼잡 완화.
  - TCP Tahoe/Reno/CUBIC, BBR(대역폭·RTT 모델링)

**TCP 윈도우와 BDP**
윈도우 크기가 BDP보다 작으면 링크가 **미가동**(underutilization).

```python
# 윈도우 최적값 계산 도우미

def optimal_window_bytes(bw_mbps, rtt_ms):
    bw_bps = bw_mbps * 1_000_000
    rtt_s = rtt_ms / 1000
    return int(bw_bps * rtt_s)

print(optimal_window_bytes(100, 50))  # 100Mbps, RTT 50ms일 때 윈도우 바이트
```

## QoS(품질 보장)

- 분류/마킹(DSCP), 큐잉(LLQ/CBWFQ), 셰이핑/폴리싱, WRED.
- 미디어 스트림: **지터 버퍼**, FEC, 적응형 비트레이트(ABR).

---

# 사례 중심 학습 시나리오

## “원격 제어 로봇” 링크 설계

**요구**: 720p@30fps 영상 + 제어명령 왕복 지연 < 80ms, 손실 1% 이내.

- **전송**: 영상은 RTP/UDP + FEC, 제어는 QUIC(낮은 지연·암호화).
- **매체**: 5GHz Wi-Fi + 전용 채널, 장애물 환경은 MIMO/빔포밍 고려.
- **QoS**: 제어 패킷 DSCP EF, 영상은 AF41, 백그라운드 BE.
- **버퍼링**: 지터 버퍼 30–50ms, 제어채널은 우선 큐(LLQ).

간이 용량 추정:
$$
\text{Video Rate} \approx \text{Res} \times \text{fps} \times \text{bits/pixel} \times \text{압축효율}
$$

## “사무실 VoIP” 최적화 체크리스트

- 스위치: 음성 VLAN 분리, 포트 트러스트, PoE 안정성.
- 라우터: LLQ(음성 우선), 지터 < 30ms, 패킷 손실 < 1%.
- 방화벽: SIP ALG 검토(문제시 비활성), RTP 포트 범위 허용.
- 모니터링: RTCP QoS 통계, MOS(주관적 품질) 추적.

---

# 간단 실습: 패킷 지연/지터 관측

## ping/iperf 조합

```bash
# 지연/손실 관측

ping -i 0.2 -c 50 target.example.com

# 처리량 측정 (TCP)

iperf3 -c target -t 15
# UDP에서 손실/지터

iperf3 -c target -u -b 20M -t 15 --get-server-output
```

## 캡처/분석 스니펫

```bash
# tcpdump로 5060(SIP)와 RTP(16384-32767)만 캡처

sudo tcpdump -i eth0 -w voip.pcap 'port 5060 or portrange 16384-32767'
```

Wireshark 디스플레이 필터 예:
```
rtp || sip
tcp.analysis.retransmission
udp && frame.len > 1200
```

---

# 문제풀이 감각을 위한 퀵 퀴즈

1) **BDP**가 12.5 MB인 회선에서 윈도우가 4 MB라면 링크 사용률은? (대략 **32%**)
2) **Half-Duplex** 환경에서 두 노드가 동시에 전송하려 할 때 필요한 절차는? (예: **RTS/CTS**, 백오프)
3) **UTF-8**에서 ‘가’(U+AC00)는 몇 바이트? (3바이트: `EAB080`)
4) **링 토폴로지**의 장애 복구 장점은? (링 보호 스위칭으로 **우회 신속**)

---

# 체크리스트 요약

- **전달·정확성·적시성·지터**를 각기 **주소지정/오류제어/지연최적화/버퍼링**으로 대응.
- **데이터 표현**은 인코딩·엔디언·코덱의 호환성을 우선 검토.
- **흐름/혼잡 제어**·**QoS**·**암호화**를 서비스 특성에 맞게 조합.
- **물리 구조/토폴로지**는 장애 모델과 운영 편의성을 기준으로 선택.
- **측정**(핑/iperf/RTCP/캡처) → **가설** → **개선** → **재측정**의 루프를 습관화.

---

# 부록 A: 용어 정리(핵심만)

- **Latency**: 왕복(RTT) or 단방향(One-way) 지연.
- **Jitter**: 도착 간격의 표준편차.
- **Throughput/Goodput**: 초당 처리 비트/실제 유효 페이로드.
- **BER**: 비트 오류율.
- **MTBF/MTTR/Availability**: 신뢰도 지표.
- **P2P/Multipoint**: 연결 유형.
- **Mesh/Star/Bus/Ring/Tree**: 물리 토폴로지.

---

# 부록 B: 연습 과제

1) **캠퍼스 네트워크**를 Star(Tree 계층)로 설계하고, 코어 이중화(MLAG/VRRP)를 포함한 장애 시나리오와 페일오버 시간을 제시하라.
2) **VoIP** 트래픽을 위해 DSCP 마킹과 큐잉 정책을 설계하고, 예상 지터/손실 예산을 계산하라.
3) **원격 로봇** 제어에서 UDP 기반 RTP와 QUIC 기반 제어채널을 혼용하는 이유를 장단점으로 설명하라.
