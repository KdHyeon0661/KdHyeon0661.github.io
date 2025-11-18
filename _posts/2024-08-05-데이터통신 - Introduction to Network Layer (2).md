---
layout: post
title: 데이터 통신 - Connecting Devices and Virtual LANs (2)
date: 2024-08-05 23:20:23 +0900
category: DataCommunication
---
#  Network Layer Performance와 IPv4 주소

## 18.3 Network-Layer Performance

네트워크 계층에서 “성능”은 보통 다음 네 가지로 요약할 수 있다.

- **Delay(지연)**: 패킷이 출발해서 도착할 때까지 걸리는 시간
- **Throughput(처리량)**: 단위 시간당 전달되는 데이터 양
- **Packet loss(패킷 손실)**: 전달 과정에서 버려지는 패킷 비율
- **Congestion control(혼잡 제어)**: 혼잡을 감지·완화하는 메커니즘

이 네 가지는 서로 강하게 연결되어 있다. 지연이 커지면 큐 길이가 길어지고,  
큐가 가득 차면 패킷 손실이 발생하고, 이를 줄이기 위해 혼잡 제어가 개입한다.

---

## 18.3.1 Delay

### 1) 지연의 네 가지 구성 요소

하나의 홉(라우터 + 링크)을 지날 때 패킷이 겪는 지연은 다음 네 가지로 나눌 수 있다.

$$
D = D_{\text{proc}} + D_{\text{queue}} + D_{\text{trans}} + D_{\text{prop}}
$$

- \(D_{\text{proc}}\) (processing delay):  
  패킷 헤더 검사, 라우팅 테이블 조회, 에러 체크 등 내부 처리 시간
- \(D_{\text{queue}}\) (queuing delay):  
  출력 큐에서 전송을 기다리는 시간 (혼잡할수록 커짐)
- \(D_{\text{trans}}\) (transmission delay):  
  패킷 비트를 링크로 밀어 넣는 시간  
  $$D_{\text{trans}} = \dfrac{L}{R}$$  
  - \(L\): 패킷 길이(bit)  
  - \(R\): 링크 대역폭(bit/s)
- \(D_{\text{prop}}\) (propagation delay):  
  매체를 따라 신호가 이동하는 시간  
  $$D_{\text{prop}} = \dfrac{d}{s}$$  
  - \(d\): 링크 길이(m)  
  - \(s\): 전파 속도(m/s) (광섬유에서 약 \(2 \times 10^8\) m/s 정도 가정)

전체 end-to-end 지연은 각 홉의 지연을 합한 것:

$$
D_{\text{end-to-end}} \approx \sum_{i=1}^{N} \left(
D_{\text{proc},i} + D_{\text{queue},i} + D_{\text{trans},i} + D_{\text{prop},i}
\right)
$$

---

### 2) 예제: 3홉 경로에서의 지연 계산

가정:

- 패킷 크기: 1000 byte = 8000 bit
- 링크 대역폭: 모든 홉 100 Mbps
- 링크 길이:
  - 호스트–R1: 10 km
  - R1–R2: 500 km
  - R2–호스트: 10 km
- 전파 속도: \(2 \times 10^8\) m/s
- 각 라우터 처리 지연 \(D_{\text{proc}} \approx 1\ \mu s\), 큐잉 지연은 거의 없음(가벼운 부하)

전송 지연:

$$
D_{\text{trans}} = \frac{8000}{100\times 10^6} = 8 \times 10^{-5}\ \text{s} = 80\ \mu s
$$

세 링크 모두 80 μs.

전파 지연:

- 10 km 링크:  
  $$D_{\text{prop}} = \dfrac{10\,000}{2\times 10^8} = 5\times 10^{-5}\ \text{s} = 50\ \mu s$$
- 500 km 링크:  
  $$D_{\text{prop}} = \dfrac{500\,000}{2\times 10^8} = 2.5\times 10^{-3}\ \text{s} = 2.5\ \text{ms}$$

각 홉별 대략:

- 홉1 (호스트→R1):
  - \(D_{\text{proc}} \approx 1\ \mu s\) (호스트 NIC)
  - \(D_{\text{queue}} \approx 0\)
  - \(D_{\text{trans}} = 80\ \mu s\)
  - \(D_{\text{prop}} = 50\ \mu s\)
- 홉2 (R1→R2):
  - \(D_{\text{proc}} \approx 1\ \mu s\)
  - \(D_{\text{queue}} \approx 0\)
  - \(D_{\text{trans}} = 80\ \mu s\)
  - \(D_{\text{prop}} = 2.5\ \text{ms}\)
- 홉3 (R2→호스트):
  - 동일 (80 μs + 50 μs 정도)

대략 합하면:

- 전송 지연: \(3 \times 80\ \mu s = 240\ \mu s\)
- 전파 지연: \(50 + 2500 + 50 = 2600\ \mu s = 2.6\ \text{ms}\)
- 처리 지연: 3 μs

→ 약 \(2.84\ \text{ms}\) 정도로, **장거리에서는 전파 지연이 지배적**임을 알 수 있다.

---

### 3) 큐잉 지연과 트래픽 부하

큐잉 지연은 트래픽 부하가 커질수록 크게 증가한다.  
단순 M/M/1 큐 모델에서, 평균 큐잉 지연은

$$
D_{\text{queue}} = \frac{\rho}{\mu - \lambda}
$$

- \(\lambda\): 평균 패킷 도착률
- \(\mu\): 서비스율(링크가 처리할 수 있는 패킷/초)
- \(\rho = \lambda/\mu\): 이용률

\(\rho\)가 1에 가까워질수록(링크가 포화상태에 가까울수록) \(D_{\text{queue}}\)가 폭발적으로 증가한다.

---

### 4) 간단한 파이썬 지연 계산 예제

```python
def link_delay(bits, rate_bps, length_m, speed=2e8,
               proc_us=1.0, queue_us=0.0):
    d_trans = bits / rate_bps
    d_prop = length_m / speed
    d_proc = proc_us * 1e-6
    d_queue = queue_us * 1e-6
    return d_proc + d_queue + d_trans + d_prop

packet_bits = 8000
rate = 100e6

hops = [
    {"length_m": 10_000},
    {"length_m": 500_000},
    {"length_m": 10_000},
]

total = 0.0
for i, h in enumerate(hops, 1):
    d = link_delay(packet_bits, rate, h["length_m"])
    print(f"Hop {i} delay = {d*1000:.3f} ms")
    total += d

print(f"Total end-to-end delay = {total*1000:.3f} ms")
```

이 코드는 위에서 손으로 계산한 값과 비슷한 숫자를 보여준다.

---

## 18.3.2 Throughput

### 1) 정의

**Throughput(처리량)**은 단위 시간당 도착지에 실제로 전달된 데이터 양이다.

- 순간 처리량: 매우 짧은 구간에서의 비트/초
- 평균 처리량: 일정 기간 동안의 평균 비트/초

네트워크 계층 관점에서, 한 end-to-end 경로의 유효 처리량은 보통 **가장 느린 링크(병목)**에 의해 결정된다.

$$
T_{\text{path}} = \min_i R_i
$$

- \(R_i\): 경로 상 i번째 링크의 대역폭

---

### 2) 예제: 경로 처리량

경로 상 링크 속도가 다음과 같다고 하자.

| 링크 | 대역폭 |
|------|--------|
| L1   | 100 Mbps |
| L2   | 10 Mbps  |
| L3   | 1 Gbps   |

그러면:

$$
T_{\text{path}} = \min(100, 10, 1000) = 10\ \text{Mbps}
$$

즉, **가장 느린 링크(L2)**가 전체 경로 처리량을 결정한다.

---

### 3) 여러 흐름이 공유할 때

한 링크(예: 100 Mbps)를 여러 흐름이 공유할 때,  
스케줄링 정책에 따라 각 흐름이 얻는 처리량은 달라진다.

- 단순하게 “동일 가중치”로 공평하게 나눌 경우,  
  4개의 흐름이 있다면 각 흐름은 이론상 25 Mbps까지 얻을 수 있다.

실제 TCP는 **혼잡 제어 알고리즘**에 따라 처리량을 나눠 가지며,  
RTT, 손실률, 혼잡 윈도우 크기 등에 영향을 받는다.

간단한 추정식(이론적인 TCP Reno의 평균 처리량 근사):

$$
T \approx \frac{1.22 \cdot \text{MSS}}{\text{RTT}\sqrt{p}}
$$

- MSS: Maximum Segment Size
- RTT: 왕복 지연
- \(p\): 손실 확률

손실률이 커질수록 처리량이 급격히 떨어지는 걸 수식으로 볼 수 있다.

---

### 4) 파이썬 예제: 병목 링크와 흐름당 처리량 추정

```python
def path_throughput(link_rates_mbps):
    return min(link_rates_mbps)

def per_flow_throughput(path_rate_mbps, num_flows):
    return path_rate_mbps / num_flows

links = [100, 10, 1000]
path_rate = path_throughput(links)
print("Path throughput =", path_rate, "Mbps")

for flows in [1, 2, 4, 8]:
    print(f"{flows} flows -> approx {per_flow_throughput(path_rate, flows):.2f} Mbps/flow")
```

이 예제는 “경로의 병목”과 “flow 수 증가에 따른 per-flow 처리량 감소”를 직관적으로 보여준다.

---

## 18.3.3 Packet Loss

### 1) 패킷 손실 원인

네트워크 계층에서 **패킷 손실**은 주로 다음 상황에서 발생한다.

1. **버퍼 오버플로우**  
   - 라우터 출력 큐가 가득 찬 상태에서 새 패킷 도착 → 드롭.
2. **링크 에러**  
   - 물리 계층에서 비트 에러가 나고,  
     링크 계층 CRC 체크에서 오류가 검출되면 해당 프레임을 폐기.
3. **TTL(Time To Live)/Hop Limit 초과**  
   - 라우팅 루프 등으로 패킷이 너무 오래 돌다가 TTL이 0이 되어 폐기.
4. **정책적 드롭**  
   - 방화벽, ACL, 보안 정책에 의해 의도적으로 버려지는 경우.

라우터는 패킷을 버렸을 때, 필요시 **ICMP 메시지**를 통해 출발지에 “Destination Unreachable” 또는 “Time Exceeded”를 통보할 수 있다.  

---

### 2) 경로 상 손실 확률

간단한 모델에서, 각 홉에서 독립적으로 손실 확률 \(p_i\)를 가진다면,  
엔드-투-엔드 손실 확률은

$$
p_{\text{end-to-end}} = 1 - \prod_{i=1}^{N} (1 - p_i)
$$

예:

- 3홉에서 각 홉 손실 확률이 1% (\(p_i=0.01\))라면,

$$
p_{\text{end-to-end}} = 1 - (0.99)^3 \approx 1 - 0.970299 = 0.029701 \approx 2.97\%
$$

→ 홉 수가 늘어날수록 end-to-end 손실 확률이 증가함을 알 수 있다.

---

### 3) 간단 파이썬 예제: 손실 확률 계산

```python
def end_to_end_loss(loss_probs):
    p_keep = 1.0
    for p in loss_probs:
        p_keep *= (1 - p)
    return 1 - p_keep

for p in [0.001, 0.01, 0.05]:
    loss = end_to_end_loss([p, p, p])
    print(f"3 hops, {p*100:.1f}% per-hop loss -> end-to-end loss {loss*100:.2f}%")
```

이 코드는 각 홉 손실률에 따른 엔드-투-엔드 손실률 증가를 보여준다.

---

## 18.3.4 Congestion Control

### 1) 혼잡 vs 단순 높은 지연

**혼잡(congestion)**은 네트워크 자원(링크, 버퍼)을 초과하는 트래픽이 유입돼  
큐가 길게 늘어나고, 처리량이 떨어지며, 손실이 발생하는 상태를 말한다.

- 단순히 “지연이 조금 큰 상태”와는 다르다.
- 혼잡이 심해지면 **처리량이 오히려 감소**하는 구간이 나온다.

전형적인 “지연–처리량” 그래프를 그려 보면,  
트래픽 부하가 적당할 땐 처리량이 증가하다가,  
어느 지점을 넘으면 큐잉 지연과 손실 때문에 처리량이 감소하는 모습을 볼 수 있다.

---

### 2) 혼잡 제어 vs 흐름 제어

- **흐름 제어(flow control)**:  
  한 송신자가 수신자의 버퍼를 감당하지 못할 정도로  
  너무 빨리 보내지 않도록 **송신자–수신자 간** 속도를 조절. (예: TCP window, receiver window)
- **혼잡 제어(congestion control)**:  
  네트워크 전체에서 라우터/링크가 과부하되지 않도록  
  **네트워크 상태(손실, 지연 등)를 보고 송신 속도를 조절**.

네트워크 계층 자체에서는 **혼잡을 감지·표기하는 메커니즘**을 제공하고,  
실제 전송 속도 조절은 전송 계층(TCP 등)이 수행하는 경우가 많다.

---

### 3) 네트워크 계층에서의 혼잡 완화 메커니즘

1. **Queue 관리**  
   - Drop-tail: 큐가 가득 차면 들어오는 패킷을 그냥 버림.
   - RED(Random Early Detection): 큐가 꽉 차기 전에 랜덤 드롭으로 혼잡을 “비용 신호”로 노출.
   - ECN(Explicit Congestion Notification): 패킷을 드롭하는 대신 **헤더의 ECN 비트를 세팅**해 혼잡을 알려줌.  

2. **Traffic Engineering & 경로 제어**  
   - 링크/경로 용량을 고려해 트래픽을 분산 (예: MPLS-TE, Equal-Cost Multi-Path 등).  

3. **Admission Control/Call Control (VC 기반)**  
   - Virtual Circuit/ATM/MPLS-TP 등에서는 새로운 흐름을 받기 전에  
     “자원이 충분한지” 평가하고, 부족하면 연결 자체를 거부.

---

### 4) 간단 AIMD(가감적 증가, 곱셈적 감소) 의사코드

전송 계층(TCP 등)에서 흔히 쓰는 혼잡 제어 알고리즘의 추상적인 형태:

```python
cwnd = 1.0  # 초기 혼잡 윈도우 (단위: MSS)
alpha = 1.0  # additive 증가량
beta = 0.5   # multiplicative 감소 비율

def on_ack():
    global cwnd
    cwnd += alpha / cwnd  # TCP Reno의 congestion avoidance 근사

def on_loss_or_ecn():
    global cwnd
    cwnd *= beta         # 손실/ECN 시 윈도우 절반으로

# 예시: 100개의 RTT 동안 ack만 왔다가, 손실이 한 번 발생하는 상황을 시뮬레이션해볼 수 있다.
```

네트워크 계층(라우터)은 **드롭/ECN 비트**를 통해 “혼잡 신호”를 제공하고,  
전송 계층은 이를 보고 윈도우를 줄이는 식으로 협력한다.

---

## 18.4 IPv4 Addresses

이제 네트워크 계층의 핵심 자원인 **IPv4 주소**를 본다.

---

## 18.4.1 Address Space

### 1) IPv4 주소 공간의 크기

IPv4 주소는 32비트 숫자다.  

$$
\text{총 주소 수} = 2^{32} = 4\,294\,967\,296
$$

사람은 이것을 보통 **dotted decimal** 형태(`a.b.c.d`)로 본다.

예:

- 192.0.2.53
- 203.0.113.10

IANA의 IPv4 Address Space Registry는 이 32비트 공간을  
각 Regional Internet Registry(RIR)에 어떻게 할당했는지,  
어떤 블록이 예약되었는지 등을 관리한다.  

또한 **특수 용도 주소 블록(0.0.0.0/8, 10.0.0.0/8, 127.0.0.0/8 등)**은  
일반 글로벌 유니캐스트로 사용되지 않고,  
로컬/테스트/루프백/멀티캐스트/브로드캐스트 등으로 예약되어 있다.  

---

### 2) IPv4 고갈과 현재 상태 (2025년 기준)

IPv4 주소는 이미 **RIR 수준에서 사실상 고갈**되었고,  
IANA는 “Recovered IPv4 Pool”에서 아주 작은 잔여 주소만 관리하고 있다.  

IANA의 2025년 3월 Number Resource Performance Report에 따르면:  

- “Unallocated remaining Recovered IPv4 Address Space is: 768 (9.6 bits) available to allocate.”

이는 **32비트 전체 공간 중 극히 일부만** 남아 있음을 의미하며,  
실제 신규 서비스는 **IPv6, NAT, 주소 공유(A+P 등) 기술**을 통해 확장하고 있다.  

---

## 18.4.2 Classful Addressing (역사적인 모델)

### 1) 클래스 기반 주소 체계

초기 IPv4는 **클래스 A/B/C**로 구분되는 고정 길이 네트워크/호스트 부분 구조를 가졌다.

| 클래스 | 첫 비트 패턴 | 첫 옥텟 범위 | 네트워크 비트 | 호스트 비트 | 네트워크 수 | 네트워크당 호스트 수 |
|--------|--------------|--------------|---------------|-------------|-------------|----------------------|
| A      | 0xxxxxxx     | 0–127        | 8             | 24          | 128         | 약 1,600만           |
| B      | 10xxxxxx     | 128–191      | 16            | 16          | 16,384      | 65,534               |
| C      | 110xxxxx     | 192–223      | 24            | 8           | 약 200만    | 254                  |
| D      | 1110xxxx     | 224–239      | Multicast     | –           | –           | –                    |
| E      | 1111xxxx     | 240–255      | Experimental  | –           | –           | –                    |

클래스 D/E는 일반 호스트 주소에는 사용되지 않는다.

네트워크/호스트 수는 대략:

- 클래스 A: \(2^7 = 128\)개 네트워크, 각 \(2^{24} \approx 16.7\)M 호스트
- 클래스 B: \(2^{14} = 16384\)개 네트워크, 각 \(2^{16}-2 = 65534\) 호스트
- 클래스 C: \(2^{21} \approx 2\)M 네트워크, 각 \(2^8-2 = 254\) 호스트

---

### 2) 예제: 클래스 판별

IP 주소 130.45.2.1을 보자.

- 첫 옥텟: 130 → 128–191 범위
- 따라서 **클래스 B 주소**

또 다른 예:

- 10.0.0.1 → 첫 옥텟 10 → 0–127 → 클래스 A

하지만 이 주소는 현재 RFC1918에 따라 **사설용(private-use) 주소**로 예약되어 있다.  

---

### 3) 클래스 기반 주소의 문제점

1. **낭비**:  
   - 중간 규모 조직에게 클래스 B(65k 호스트)는 너무 크고,  
     클래스 C(254 호스트)는 너무 작은 경우가 많았다.
2. **라우팅 테이블 폭발**:  
   - 많은 수의 클래스 C 네트워크가 개별적으로 라우팅 테이블에 올라가면서  
     Backbone 라우터의 라우팅 테이블이 급격히 커졌다.
3. **주소 공간 파편화**:  
   - 사용자 수에 맞춰 적절히 쪼개어 배포하기 어려웠다.

이를 해결하기 위해 등장한 것이 **CIDR(Classless Inter-Domain Routing)**이다.  

현재 인터넷은 **클래스리스 체계**를 사용하며, 클래스풀 주소는 역사적인 개념으로만 남아있다.

---

## 18.4.3 Classless Addressing (CIDR)

### 1) 기본 개념

CIDR은 `네트워크/호스트` 경계를 고정 길이 클래스 대신  
**가변 길이 prefix 길이(/n)**로 표현한다.  

- 표기: `a.b.c.d/n`
  - `a.b.c.d`: 네트워크 주소
  - `/n`: 앞에서부터 n비트가 네트워크 부분

예:

- 192.0.2.0/24 → 192.0.2.0–192.0.2.255 (256개 주소)
- 198.51.100.0/25 → 첫 128개 주소 (198.51.100.0–198.51.100.127)
- 198.51.100.128/25 → 나머지 128개 주소

네트워크당 주소 수:

$$
\text{주소 수} = 2^{32-n}
$$

과거에는 네트워크/브로드캐스트 주소를 제외해  
호스트 수를 \(2^{32-n}-2\)로 계산했지만, 일부 환경에서는 제한이 완화된 경우도 있다.

---

### 2) 예제: 주소 범위 계산

`10.0.0.0/12` 네트워크를 보자.

- /12 → 네트워크 비트 12개, 호스트 비트 20개
- 주소 수: \(2^{20} = 1,\!048,\!576\)개

주소 범위:

- 시작: 10.0.0.0
- 끝: 10.15.255.255 (0x0–0xF까지 네 번째 nibble가 변할 수 있음)

---

### 3) 서브넷팅 예

예: `192.168.1.0/24`를 4개 서브넷으로 나누고 싶다.

- 4개 서브넷 → 2비트 서브넷 필요 (`/26`)
- 새로운 prefix: `/26` → 64개 주소 per subnet

서브넷:

1. 192.168.1.0/26   (0–63)
2. 192.168.1.64/26  (64–127)
3. 192.168.1.128/26 (128–191)
4. 192.168.1.192/26 (192–255)

---

### 4) 라우트 집계(Route Aggregation) 예

ISP가 다음 세 개의 prefix를 고객에게 할당했다고 가정하자.

- 203.0.113.0/24
- 203.0.114.0/24
- 203.0.115.0/24
- 203.0.116.0/24

이 네 개는 연속되어 있으므로, 상위 ISP에는 **단일 요약 경로**로 광고할 수 있다.

- 203.0.112.0/20 (12–27까지 포함)

이를 통해 **글로벌 라우팅 테이블 성장**을 억제하는 것이 CIDR의 핵심 목적 중 하나다.  

---

### 5) 간단 코드: prefix에서 주소 개수 계산

```python
import ipaddress

def info(network_str):
    net = ipaddress.IPv4Network(network_str, strict=False)
    print(f"Network: {net}")
    print(f"Prefix length: {net.prefixlen}")
    print(f"Total addresses: {net.num_addresses}")
    print(f"Host range: {net.network_address} - {net.broadcast_address}")
    print()

for n in ["192.168.1.0/24", "192.168.1.0/26", "10.0.0.0/12"]:
    info(n)
```

이 코드는 파이썬 표준 라이브러리를 이용해 CIDR prefix 정보를 요약해 준다.

---

## 18.4.4 Dynamic Host Configuration Protocol (DHCP)

### 1) DHCP의 목적

**DHCP(Dynamic Host Configuration Protocol)**는 호스트가  
IP 주소, 서브넷 마스크, 기본 게이트웨이, DNS 서버 등 네트워크 설정을  
자동으로 할당받을 수 있게 해주는 프로토콜이다. DHCPv4의 표준은  
RFC 2131(프로토콜)과 RFC 2132(옵션)에서 정의되어 있고, IETF DHC 작업 그룹에서 관리한다.  

DHCP를 사용하면:

- 네트워크 관리자 입장에서 **수작업으로 IP를 할당할 필요가 줄어들고**,  
- 호스트 입장에서는 “플러그 앤 플레이”처럼 네트워크에 꽂기만 하면  
  자동으로 설정을 받는다.

---

### 2) DHCPv4 기본 절차 (DORA)

유선/무선 LAN에서 흔히 볼 수 있는 DHCPv4 절차는 **DORA** 네 단계로 요약된다.

1. **Discover**  
   - 클라이언트가 자신의 IP가 없는 상태에서  
     브로드캐스트로 “누가 DHCP 서버냐?”를 묻는 패킷 전송.
2. **Offer**  
   - DHCP 서버가 사용할 수 있는 IP와 각종 옵션을 담아 응답.
3. **Request**  
   - 클라이언트가 특정 서버의 Offer를 선택해 “그 IP 쓰겠다”고 요청.
4. **ACK**  
   - 서버가 최종 승인(DHCPACK)을 보내고, 임대 기간(lease time) 등을 포함.

이때 DHCP 메시지는 UDP 67/68 포트를 사용하여 전달된다.

---

### 3) 예시 시나리오: 노트북이 사무실에 접속할 때

1. 노트북을 스위치에 연결하고 켠다.
2. 네트워크 인터페이스가 **DHCP Discover**를 브로드캐스트(255.255.255.255)로 전송.
3. 동일 브로드캐스트 도메인에 있는 DHCP 서버가 **DHCP Offer**를 보냄  
   (예: IP 192.168.10.42, 서브넷 255.255.255.0, 게이트웨이 192.168.10.1, DNS 8.8.8.8).
4. 노트북은 Offer 중 하나를 골라 **DHCP Request**로 요청.
5. 서버는 **DHCP ACK**를 보내며, “이 IP를 8시간 동안 사용해라” 같은 lease 정보를 포함.
6. 노트북은 그 IP를 인터페이스에 설정하고, 필요하다면 **ARP로 충돌 여부를 확인**한 뒤 사용.  

일부 구현에서는 DHCPACK을 받은 뒤,  
해당 IP가 다른 노드와 충돌하지 않는지 ARP probe를 보내도록 요구한다.  

---

### 4) 리눅스/윈도우에서의 DHCP 사용 예

- 리눅스 (systemd 기반):

```bash
# NetworkManager/네트워크 설정을 통해 자동 DHCP 사용이 기본
# 수동으로 dhclient 실행 예(환경에 따라 다름)
sudo dhclient eth0
```

- 윈도우:

```text
제어판 → 네트워크 및 공유 센터 → 어댑터 설정 변경
→ 해당 인터페이스 속성 → IPv4 → 
"자동으로 IP 주소 받기", "자동으로 DNS 서버 주소 받기"
```

---

## 18.4.5 Network Address Resolution (주소 해석)

여기서 “Network Address Resolution”은 **IPv4 주소를 링크 계층 주소(MAC)**로  
매핑하는 과정을 중심으로 설명하되, 이름 해석(DNS)과도 구분해 정리한다.

### 1) ARP(Address Resolution Protocol)

IPv4 네트워크에서 호스트가 **같은 링크 상의 다른 호스트**와 통신하려면:

- 상대의 IPv4 주소(예: 192.168.1.10)는 알고 있지만,
- 이더넷 프레임을 보내기 위해 **상대 MAC 주소**가 필요하다.

이를 위해 쓰는 프로토콜이 **ARP(Address Resolution Protocol)**이다.

동작 요약:

1. 송신 호스트가 자신의 ARP 캐시를 확인해  
   `192.168.1.10 → MAC ?` 매핑이 있는지 본다.
2. 없다면, **브로드캐스트 ARP Request** 전송:
   - “192.168.1.10 가지고 있는 노드, 너 MAC 주소 뭐냐?”
3. 대상 호스트(192.168.1.10)는 **ARP Reply**로 자신의 MAC을 유니캐스트로 응답.
4. 송신자는 `192.168.1.10 → xx:xx:xx:xx:xx:xx` 매핑을 ARP 캐시에 저장하고,  
   이후 일정 시간 동안 재사용한다.

---

### 2) 예제: ARP 동작 흐름

호스트 A: 192.168.1.20 / MAC AA:AA:AA:AA:AA:AA  
호스트 B: 192.168.1.30 / MAC BB:BB:BB:BB:BB:BB

A가 B에게 패킷을 보내고 싶다.

1. A는 라우팅 테이블을 보고, 192.168.1.30이 같은 서브넷에 있다고 판단.
2. A는 ARP 캐시에 192.168.1.30에 대한 엔트리가 없음을 확인.
3. A:

   - 프레임 목적지 MAC: ff:ff:ff:ff:ff:ff (브로드캐스트)
   - ARP Request: “Who has 192.168.1.30? Tell 192.168.1.20”

4. 모든 호스트가 프레임을 수신하지만,  
   192.168.1.30을 가진 B만 응답:

   - 프레임 목적지 MAC: AA:AA:AA:AA:AA:AA
   - ARP Reply: “192.168.1.30 is at BB:BB:BB:BB:BB:BB”

5. A는 ARP 캐시에 이 매핑을 저장하고,  
   이후 실제 IPv4 패킷을 **MAC=BB:BB:...** 주소로 전송.

---

### 3) ARP 캐시 확인 예 (리눅스/윈도우)

- 리눅스:

```bash
ip neigh
# 또는
arp -n
```

- 윈도우:

```bash
arp -a
```

이 명령을 사용하면 “현재 기억하고 있는 IP→MAC 매핑”을 볼 수 있다.

---

### 4) DHCP와 ARP의 관계

앞서 언급했듯, DHCP 클라이언트는 **새로운 IP를 받았을 때**,  
해당 IP가 이미 다른 장비가 사용 중인지 확인하기 위해 **ARP Probe**를 수행하도록  
권고된다.  

- 충돌이 감지되면(다른 장비가 ARP Reply를 보낸다면),  
  클라이언트는 그 IP를 사용하지 않고, DHCP 서버에게 재요청할 수 있다.

---

### 5) 이름 해석(DNS) vs 주소 해석(ARP)

- **DNS**: 호스트 이름(FQDN) → IP 주소 변환  
  예: `www.example.com` → 192.0.2.53
- **ARP**: IP 주소 → MAC 주소 변환  
  예: 192.0.2.53 → 00:11:22:33:44:55

전형적인 흐름:

1. 애플리케이션이 DNS로 이름을 IP로 해석.
2. 네트워크 계층/링크 계층이 ARP로 IP를 MAC으로 해석.
3. 이더넷 프레임에 해당 MAC을 넣고 전송.

---

## 정리

- **18.3 Network-layer performance**
  - Delay: 처리/큐잉/전송/전파 지연으로 구성되며, 장거리에서는 전파, 혼잡 시에는 큐잉이 지배적.
  - Throughput: 경로 처리량은 병목 링크에 의해 결정되며, 여러 흐름이 공유하면 per-flow 처리량은 그에 따라 나뉜다.
  - Packet loss: 버퍼 오버플로우, 링크 에러, TTL 초과, 정책적 드롭 등으로 발생하며, 홉 수가 늘수록 end-to-end 손실 확률이 증가한다.
  - Congestion control: 네트워크 계층은 드롭, ECN, TE 등을 통해 혼잡을 신호/완화하고, 전송 계층은 AIMD 등으로 실제 보내는 속도를 조절한다.

- **18.4 IPv4 addresses**
  - Address space: 32비트, 총 \(2^{32}\)개 주소이며, 특수 용도 블록과 RIR 할당이 IANA Registry에 정의되어 있다. IPv4는 이미 고갈 상태이며, Recovered Pool에 극히 일부만 남아 있다.
  - Classful addressing: A/B/C/D/E 클래스 구조는 낭비와 라우팅 테이블 폭발 문제를 낳아 현재는 역사적 개념으로만 남아 있다.
  - Classless addressing (CIDR): `/n` prefix로 주소를 표현하고, 서브넷팅과 라우트 집계를 통해 주소 공간과 라우팅 테이블 성장을 효율적으로 관리한다.
  - DHCP: 호스트에 IP 설정을 자동으로 할당하는 프로토콜로, Discover–Offer–Request–ACK(DORA) 절차를 거친다.
  - Network address resolution: ARP를 통해 IPv4 주소를 MAC 주소로, DNS를 통해 이름을 IPv4/IPv6 주소로 해석하며, DHCP와 ARP는 IP 충돌 검출 과정에서도 결합된다.

이 내용을 기반으로 이후 장에서 다룰 **IPv4 헤더 포맷, IPv6 설계, 라우팅 프로토콜(OSPF/BGP), NAT, MPLS** 등을 이해하는 토대를 마련할 수 있다.
