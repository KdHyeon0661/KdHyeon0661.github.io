---
layout: post
title: 데이터 통신 - Transmission Control Protocol (4)
date: 2024-08-30 21:20:23 +0900
category: DataCommunication
---
# Transmission Control Protocol — 혼잡 제어, 타이머, 옵션

앞 절에서 **TCP 서비스, 세그먼트 형식, 창(window), 흐름/오류 제어**를 봤다고 가정하고, 이제 실전에서 TCP 성능과 안정성을 좌우하는 세 가지를 집중적으로 본다.

- 혼잡 제어(congestion control)
- 타이머(timers)
- 옵션(options)

이 세 가지는 OS 커널의 TCP 스택 안에서 동작하므로 애플리케이션에서는 직접 보이지 않지만, 실제 네트워크 튜닝·트러블슈팅·성능 분석에서는 반드시 이해해야 하는 부분이다.

---

## TCP 혼잡 제어 (Congestion Control)

### 혼잡 제어 vs 흐름 제어

- **흐름 제어(flow control)**
  수신 호스트의 버퍼 오버플로를 막기 위한 메커니즘
  → 수신 측이 **수신 윈도우(rwnd)** 를 광고해서 "나 지금 이만큼밖에 못 받는다"라고 알려줌.

- **혼잡 제어(congestion control)**
  **네트워크 자체(라우터·링크)가 혼잡해지는 것**을 막기 위한 메커니즘
  → 송신 측이 **혼잡 윈도우(cwnd)** 를 관리해서, 네트워크가 감당 가능한 양만큼만 패킷을 날리도록 조절.

실제로 송신자가 한 번에 날릴 수 있는 데이터 양(전송 윈도우)은

$$
\text{send\_window} = \min(\text{cwnd}, \text{rwnd})
$$

으로 결정된다. 즉 수신 버퍼와 네트워크 둘 중 더 작은 쪽에 맞춘다.

#### 간단한 비유

- **흐름 제어**: 물을 받는 컵(수신자)의 크기.
- **혼잡 제어**: 수도관(네트워크)의 굵기와 마을 전체 배관 상태.

컵이 크더라도 수도관이 얇으면 쏟아부으면 안 되고, 수도관이 두꺼워도 컵이 작으면 넘치지 않게 조절해야 한다.

---

### 혼잡 윈도우(cwnd)와 AIMD

TCP는 기본적으로 **AIMD (Additive Increase / Multiplicative Decrease)** 전략을 사용한다.

- **혼잡이 없다고 판단되면**
  → 윈도우를 **선형(additive)** 으로 증가
  $$\text{cwnd} \leftarrow \text{cwnd} + \text{MSS} \quad (\text{대략 RTT마다})$$

- **혼잡(손실)이 감지되면**
  → 윈도우를 **곱셈 감소(multiplicative decrease)**
  $$\text{cwnd} \leftarrow \frac{1}{2} \text{cwnd}$$

여기에서 **MSS(Maximum Segment Size)** 는 하나의 TCP 세그먼트에 실릴 수 있는 최대 페이로드 크기(바이트)다.
실제 구현은 더 정교하지만, 개념적으로는 위 공식이 맞다.

---

### 고전적인 TCP Reno 스타일 알고리즘

실습과 시험에서 가장 자주 나오는 모델은 **Tahoe / Reno / NewReno 계열**이다.

#### 핵심 상태값

- `cwnd` : 혼잡 윈도우 (보통 MSS 단위)
- `ssthresh` : slow start threshold (느린 시작과 혼잡 회피를 가르는 기준)
- `rwnd` : 수신 윈도우
- `RTO` : 재전송 타이머 값

#### 연결 초기에: Slow Start

연결을 새로 맺으면 송신자는 네트워크 상태를 모른다.
그래서 아주 작은 윈도우에서 시작해 **지수적으로 증가**시키며 탐색한다.

- 초기:
  - 옛날: $$\text{cwnd}_0 = 1 \text{ MSS}$$
  - 현대 OS: 10 MSS까지 허용하는 경우가 많다 (IW10).
- 매 RTT마다:
  - 모든 세그먼트가 정상 ACK → `cwnd`가 **두 배**로 증가
    $$\text{cwnd}_{k+1} = 2 \cdot \text{cwnd}_k$$

예를 들어 초기 `cwnd=1 MSS`라면 RTT마다 `1, 2, 4, 8, 16, 32, ...` 식으로 증가한다.
`cwnd`가 `ssthresh`에 도달하면 **혼잡 회피(congestion avoidance)** 상태로 전환한다.

#### 혼잡 회피(Congestion Avoidance)

slow start에서 어느 정도 윈도우가 커지면 더는 지수 성장하면 위험하므로,
이제는 **선형으로 조금씩 증가**시킨다.

ACK 하나당 증가량은

$$
\Delta \text{cwnd} = \frac{\text{MSS}^2}{\text{cwnd}}
$$

이고 하나의 RTT 동안 약 `cwnd/MSS` 개의 ACK가 오므로, 결과적으로는

$$
\Delta \text{cwnd(per RTT)} \approx \text{MSS}
$$

즉 **RTT마다 MSS 하나만큼** 증가하는 셈이다.

#### 손실 반응: 타임아웃 vs 중복 ACK

TCP는 손실을 두 가지 방식으로 감지한다.

1. **재전송 타임아웃(RTO) 만료**
   - 가장 보수적인 상황.
   - 반응(고전 Reno/Tahoe 스타일):
     - `ssthresh = cwnd / 2`
     - `cwnd = 1 MSS` 로 리셋 → 다시 slow start

2. **중복 ACK 3개(3 Duplicate ACKs)**
   - 예: 세그먼트 #5가 유실되고 #6, #7이 도착하면,
     수신자는 계속 "4번까지 받았다"는 ACK(중복 ACK)를 보낸다.
   - 송신자는 **같은 ACK가 3번 연속** 오면 "중간 어딘가 하나가 빠졌거나 순서가 꼬였다"고 추측.
   - 반응(Reno 기본):
     - **Fast retransmit**: 타이머 기다리지 않고 바로 재전송.
     - `ssthresh = cwnd / 2`
     - `cwnd = ssthresh` 근처로 줄이고 **fast recovery** 단계로 진입.

이 덕분에 **타임아웃을 기다리지 않고 빠르게 복구**할 수 있다.

---

### cwnd 변화 예시 시나리오

**환경 가정**

- 링크 대역폭: 10 Mbps
- RTT: 100 ms
- MSS: 1460 바이트
- 이상적으로 링크를 꽉 채우기 위한 BDP:

$$
\text{BDP} = \text{Bandwidth} \times \text{RTT} = 10^7 \times 0.1 = 10^6 \text{ bit}
$$

$$
\Rightarrow \frac{10^6}{8 \times 1460} \approx 86 \text{ MSS}
$$

즉 **86 MSS 정도의 cwnd**가 되어야 링크를 거의 꽉 채운다.

#### 단계별 cwnd 변화

초기 `cwnd = 1 MSS`, `ssthresh`는 임의로 64 MSS라고 가정:

| RTT 번호 | 상태          | cwnd(MSS) | 설명 |
|---------|---------------|-----------|------|
| 1       | slow start    | 1 → 2     | ACK 수신 후 두 배 |
| 2       | slow start    | 2 → 4     |     |
| 3       | slow start    | 4 → 8     |     |
| 4       | slow start    | 8 → 16    |     |
| 5       | slow start    | 16 → 32   |     |
| 6       | slow start    | 32 → 64   | `ssthresh` 도달 |
| 7       | 혼잡 회피     | 64 → 65   | RTT 동안 ACK들로 MSS 하나 증가 |
| 8       | 혼잡 회피     | 65 → 66   |     |
| ...     | 혼잡 회피     | ...       | 86 MSS에 점근 |

만약 **RTT 9에서 손실**이 발생하고 3중 중복 ACK가 감지되었다고 치자.

- `ssthresh = cwnd / 2`
- `cwnd`를 `ssthresh` 근처로 줄이고 fast recovery로 들어감
(상세 동작은 구현/알고리즘에 따라 조금씩 다르다.)

---

### 현대적인 혼잡 제어 알고리즘 언급

실제 OS들은 단순 Reno 대신 다음과 같은 알고리즘을 사용한다.

- **CUBIC** (Linux 기본)
  - cwnd를 시간에 대한 **3차 함수**로 조정해, 고속·고지연 링크(LFN)에 적합.
- **BBR** (Google 설계, 일부 환경에서 사용)
  - 손실이 아니라 **대역폭·RTT 모델** 기반 제어.
  - 큐가 과도하게 쌓이지 않도록 지연을 직접 제어하려고 시도.

시험이나 기본 교재는 주로 **Reno/Tahoe/NewReno + SACK** 정도를 가정하므로,
CUBIC/BBR은 "현대 OS에서 실제 쓰는 것" 정도로 이해하고,
기본 개념은 **cwnd, AIMD, slow start, congestion avoidance, fast retransmit/recovery**를 확실히 잡으면 된다.

---

### 간단한 cwnd 시뮬레이션 코드 예제

아래는 **slow start + 혼잡 회피**를 단순화해 시뮬레이션하는 파이썬 예제다.

```python
def simulate_cwnd(ssthresh=64, max_rtt=15, loss_rtt=10):
    cwnd = 1.0  # MSS 단위
    history = []
    for rtt in range(1, max_rtt + 1):
        state = "slow-start" if cwnd < ssthresh else "congestion-avoidance"
        history.append((rtt, cwnd, state))

        # 손실 발생 시점
        if rtt == loss_rtt:
            # Reno 스타일 반응 (단순화)
            ssthresh = cwnd / 2
            cwnd = ssthresh  # fast recovery 이후 상태라고 가정
            continue

        # 혼잡이 없을 때 증가 규칙
        if cwnd < ssthresh:  # slow start
            cwnd *= 2
        else:                # congestion avoidance
            cwnd += 1.0      # RTT당 1 MSS 증가(근사)

    return history

for rtt, cwnd, state in simulate_cwnd():
    print(f"RTT={rtt:2d}, cwnd={cwnd:5.1f} MSS, state={state}")
```

실제 TCP 구현은 더 복잡하지만, 이 정도 시뮬레이션만으로도 cwnd가
- 초기에 **지수 증가**,
- 어느 순간부터 **선형 증가**,
- 손실 시 **급감**
하는 큰 흐름을 직관적으로 볼 수 있다.

---

## TCP 타이머 (TCP Timers)

TCP는 신뢰성과 효율을 위해 여러 종류의 타이머를 사용한다.

대표적인 것만 정리하면:

1. **재전송 타이머 (Retransmission Timer, RTO)**
2. **지연 ACK 타이머 (Delayed ACK Timer)**
3. **영 윈도우 프로브 타이머 (Persist / Zero-Window Probe Timer)**
4. **Keepalive 타이머**
5. **TIME-WAIT / 2MSL 관련 타이머**

### 재전송 타이머 (RTO)

송신자가 세그먼트를 보낼 때마다 **재전송 타이머**를 건다.

- 그 세그먼트의 ACK가 **RTO 안에 도착**하면 → 타이머 취소
- RTO가 만료될 때까지 ACK가 없으면
  → 세그먼트를 재전송하고, 혼잡 제어 측면에서는 **심각한 손실**로 간주하여 `cwnd`를 크게 줄인다.

#### RTO 계산 — SRTT / RTTVAR

표준 알고리즘은 다음과 같은 개념을 쓴다.

- $$\text{RTT}$$: 개별 세그먼트의 왕복 시간 측정값
- $$\text{SRTT}$$: smoothed RTT (지수 이동 평균)
- $$\text{RTTVAR}$$: RTT 변동성(편차)
- $$\text{RTO}$$: 재전송 타임아웃

초기 RTT 측정값을 $$R_0$$라고 할 때:

- $$\text{SRTT} \leftarrow R_0$$
- $$\text{RTTVAR} \leftarrow R_0 / 2$$
- $$\text{RTO} \leftarrow \text{SRTT} + 4 \cdot \text{RTTVAR}$$

추가 측정값 $$R_n$$이 들어올 때마다:

- $$\text{RTTVAR} \leftarrow (1-\beta)\cdot \text{RTTVAR} + \beta \cdot |\text{SRTT}-R_n|$$
- $$\text{SRTT} \leftarrow (1-\alpha)\cdot \text{SRTT} + \alpha \cdot R_n$$
- $$\text{RTO} \leftarrow \text{SRTT} + 4 \cdot \text{RTTVAR}$$

보통 $$\alpha = 1/8$$, $$\beta = 1/4$$ 정도를 쓴다.

또한 RTO는 너무 작거나 크지 않도록 아래처럼 **하한/상한**을 둔다.

- $$\text{RTO} \ge \text{RTO}_{\min}$$ (예: 1초)
- $$\text{RTO} \le \text{RTO}_{\max}$$ (예: 수십 초)

#### RTO 계산 간단 예

초기 RTT 측정값이 100 ms, 그 다음이 120 ms, 80 ms라고 해 보자.

1. 초기:
   - $$R_0 = 0.1s$$
   - $$\text{SRTT} = 0.1, \quad \text{RTTVAR} = 0.05$$
   - $$\text{RTO} = 0.1 + 4 \times 0.05 = 0.3s$$

2. 두 번째 측정 $$R_1 = 0.12s$$ 일 때:

   - $$|\text{SRTT} - R_1| = 0.02$$
   - $$\text{RTTVAR} = (1-0.25)\cdot 0.05 + 0.25\cdot 0.02 = 0.05 \times 0.75 + 0.005 = 0.0425$$
   - $$\text{SRTT} = (1-0.125)\cdot 0.1 + 0.125\cdot 0.12 \approx 0.1025$$
   - $$\text{RTO} \approx 0.1025 + 4 \times 0.0425 \approx 0.2725s$$

3. 세 번째 측정 $$R_2 = 0.08s$$ 에 대해 같은 식으로 갱신하면,

RTO가 RTT 변화에 부드럽게 따라가도록 조정되는 것을 볼 수 있다.

#### 파이썬으로 RTO 계산 예제

```python
def rto_update(samples, alpha=1/8, beta=1/4):
    R0 = samples[0]
    srtt = R0
    rttvar = R0 / 2
    rto = srtt + 4 * rttvar
    history = [(0, srtt, rttvar, rto)]

    for i, R in enumerate(samples[1:], start=1):
        rttvar = (1 - beta) * rttvar + beta * abs(srtt - R)
        srtt = (1 - alpha) * srtt + alpha * R
        rto = srtt + 4 * rttvar
        history.append((i, srtt, rttvar, rto))

    return history

samples = [0.10, 0.12, 0.08, 0.11, 0.09]
for i, srtt, rttvar, rto in rto_update(samples):
    print(f"sample#{i}, SRTT={srtt:.3f}, RTTVAR={rttvar:.3f}, RTO={rto:.3f}")
```

이 코드를 실행해보면 RTT가 조금씩 변해도 RTO가 안정적으로 추적하는 모습을 확인할 수 있다.

---

### 지연 ACK 타이머 (Delayed ACK)

많은 TCP 구현은 **두 세그먼트마다 하나의 ACK만 보내거나**,
혹은 **최대 지연 시간(수십 ms 수준)** 안에 도착한 여러 세그먼트를 묶어서 ACK한다.

- 장점:
  - ACK 수를 줄여 오버헤드 감소
- 단점:
  - RTT 측정이 부정확해질 수 있고, 상호 작용이 많은 애플리케이션에서는 지연이 체감될 수 있다.

보통 OS는 **최대 지연 ACK 시간**을 적당한 값으로 설정하고,
이 타이머가 만료되기 전에 새로운 세그먼트가 오면 ACK를 합쳐서 보낸다.

---

### Persist / Zero-Window Probe 타이머

수신자가 "지금은 버퍼가 꽉 찼으니 더 이상 보내지 마라"며 **윈도우 크기 0**을 광고할 수 있다.

- 문제:
  - 이 상태에서 **윈도우 업데이트 세그먼트가 손실**되면, 송신자는 영원히 기다리게 된다.

이를 막기 위해 송신자는 **Persist 타이머**를 둔다.

1. 수신자가 `rwnd = 0`을 광고하면 송신자는 전송을 멈추고 Persist 타이머를 건다.
2. 타이머가 만료되면 송신자는 **1바이트짜리 probe** 세그먼트를 보내 본다.
3. 수신자가 다시 ACK를 하면서 새로운 윈도우 크기를 광고하면 전송 재개.

단순하지만, 실제로는 이 메커니즘이 없으면 **사용자 프로세스가 멈춰버린 것처럼 느껴지는 버그**가 생길 수 있다.

---

### Keepalive 타이머

TCP 자체는 연결 수명을 강제하지 않는다. 애플리케이션이 전혀 데이터를 보내지 않고 양쪽이 조용해도 연결은 남아 있을 수 있다.

- **Keepalive 옵션**을 켜면:
  - 일정 시간(전통적으로 수 시간) 동안 아무 데이터도 오가지 않으면
    송신자가 **keepalive probe**를 보내, 상대가 아직 살아 있는지 확인한다.
  - 응답이 없으면 연결을 끊는다.

웹 서버나 데이터베이스 서버에서 **유휴 연결을 정리**할 때 흔히 쓰인다.
클라우드/로드밸런서 환경에서는 OS의 keepalive 외에 L7/L4 장비가 자체 타이머를 두기도 한다.

---

### TIME-WAIT / 2MSL 타이머

TCP 연결을 정상 종료(FIN/ACK 교환)하면, **마지막 ACK를 보낸 쪽**은
일정 시간 동안 `TIME-WAIT` 상태로 남는다.

- 목적:
  1. 마지막 ACK가 손실되었을 경우, 상대가 다시 FIN을 보냈을 때 응답 가능하게.
  2. 오래된 세그먼트가 네트워크 안에서 떠다니다가 **새 연결로 잘못 인식**되는 것을 방지.

이 시간은 보통 **MSL(Maximum Segment Lifetime)의 두 배**, 즉 **2MSL**로 잡는다.

- 실무적으로:
  - 전통적으로 MSL = 2분 → 2MSL = 4분 같은 값이 들어 있었지만,
  - 현대 OS들은 더 짧게(수십 초 수준) 설정하기도 한다 (커널 옵션으로 조정 가능).

`TIME-WAIT` 소켓이 너무 많으면 서버 포트 고갈 문제를 일으키므로,
대규모 서버에서는 **포트 재사용 정책, TIME-WAIT 튜닝**이 중요한 운영 포인트가 된다.

---

## TCP 옵션 (TCP Options)

### 옵션 필드 구조

TCP 헤더의 기본 길이는 20바이트이지만, **옵션 필드**를 붙여 40바이트까지 확장할 수 있다.

- 필드는 32비트(4바이트) 단위로 정렬되어야 한다.
- 옵션 리스트는 다음 두 특수 옵션으로 끝난다.
  - **EOL (End of Option List, kind=0)**: 옵션 리스트 끝
  - **NOP (No Operation, kind=1)**: 패딩용 (옵션을 4바이트 정렬하기 위해 삽입)

일반적인 옵션 형식(예: MSS, Window Scale, Timestamp 등)은

- Kind (1바이트)
- Length (1바이트)
- 그 이후는 옵션별 데이터

형식을 가진다.

---

### 최대 세그먼트 크기 옵션 (MSS Option)

**Kind=2, Length=4, 값=2바이트 MSS**.

- 의미:
  - "나는 한 세그먼트 당 최대 이만큼까지 받을 수 있다"라는 수신 측의 선언.
- 사용:
  - 보통 **SYN, SYN-ACK**에만 포함된다.
  - 양쪽이 MSS를 광고하면, 실제 데이터 세그먼트 크기는 **더 작은 값**으로 제한된다.

#### 예: Ethernet + IPv4 조합

- Ethernet MTU: 1500 바이트
- IPv4 헤더: 20 바이트 (옵션 없음 가정)
- TCP 헤더: 20 바이트 (옵션 없음 가정)

따라서,

$$
\text{MSS} = 1500 - 20 - 20 = 1460 \text{ bytes}
$$

만약 PPPoE 같은 환경에서 MTU가 1492로 줄었다면,

$$
\text{MSS} = 1492 - 20 - 20 = 1452 \text{ bytes}
$$

따라서 양쪽 호스트의 SYN에 다음과 같은 옵션이 실릴 수 있다.

```text
클라이언트 SYN: MSS=1460
서버 SYN-ACK:   MSS=1452
→ 실제 사용 MSS = min(1460, 1452) = 1452
```

이는 **IP 단편화(fragmentation)** 를 피하고, 네트워크 장비의 효율을 높이기 위해 필수적이다.

---

### 윈도우 스케일 옵션 (Window Scale Option)

TCP 헤더의 **윈도우 필드**는 16비트이므로 최대 값은 $$2^{16}-1 = 65535$$ 바이트이다.
그러나 현대 네트워크(고속·고지연 링크)에서는 이 정도 윈도우로는 대역폭을 활용하기에 턱없이 부족하다.

이를 위해 나온 것이 **윈도우 스케일 옵션**이다.

- Kind = 3
- Length = 3
- 값: **shift count (0~14)**

유효 윈도우는

$$
\text{effective\_rwnd} = \text{rwnd\_field} \times 2^{\text{shift}}
$$

예:

- SYN 시 Window Scale=7(즉, `2^7 = 128`)을 협상.
- 수신 측이 헤더의 윈도우 필드에 `65535`를 넣어 광고하면,

$$
\text{effective\_rwnd} = 65535 \times 2^7 \approx 8 \text{ MiB}
$$

이렇게 하면 수십 ms의 RTT를 가진 수 Gbps 링크에서도
TCP가 대역폭을 충분히 활용할 수 있다.

---

### 옵션

고전 TCP는 **누적 ACK(cumulative ACK)** 만 제공한다.

- 예: "1000번까지 데이터 다 받았다"는 뜻으로 `ACK=1001` 같은 식.

이 방식은 **여러 세그먼트가 동시에 빠진 경우 비효율적**이다.
이를 개선하기 위해 나온 것이 **SACK 옵션**이다.

#### SACK Permitted 옵션

- Kind = 4, Length = 2
- SYN/SYN-ACK에만 사용
- "나는 SACK을 이해할 수 있다"는 의미

#### SACK 옵션

- Kind = 5
- Length = 가변 (보통 10, 18, 26, 34 등)
- 하나의 옵션 안에 **최대 4개 정도의 "받은 구간"**(left edge, right edge)을 기술.

예:

```text
SACK 블록 1: [2001, 3001)
SACK 블록 2: [4001, 4501)
```

수신자는 "누적 ACK는 1001까지지만, 2001~3000, 4001~4500도 받았다"고 알려주는 것이다.

송신자는 이렇게 "어디가 비어 있는지"를 정확히 알 수 있어,
**여러 패킷이 빠진 경우에도 최소한의 재전송으로 복구**할 수 있다.

---

### Timestamp 옵션

**Timestamp 옵션**은 두 가지 목적에 쓰인다.

1. RTT 측정을 더 정확하게 하기 (RTTM)
2. 시퀀스 번호 래핑(2^32 순환)으로 인한 문제를 막기 (PAWS)

형식은 대략 다음과 같다.

- Kind = 8
- Length = 10
- 필드:
  - TSval (4 bytes): 송신자의 타임스탬프 값
  - TSecr (4 bytes): 수신자가 마지막으로 받은 상대의 TSval 값

#### RTT 측정 사용 예

1. A가 B에게 세그먼트를 보내면서 `TSval = 1000`을 싣는다.
2. B가 응답 세그먼트에 `TSecr = 1000`을 싣고 보낸다.
3. A는 응답이 도착했을 때 자신의 현재 시간과 `TSecr`을 비교해 RTT를 바로 계산.

이 방식은 **지연 ACK, 패킷 재전송 등으로 혼동되는 경우**에도 안정적인 RTT 측정을 하도록 도와준다.

#### PAWS (Protect Against Wrapped Sequence numbers)

고속 링크에서는 시퀀스 번호(32비트)가 **매우 빠르게 한 바퀴를 도는 문제**가 생길 수 있다.
Timestamp 옵션을 사용하면 오래된 세그먼트의 타임스탬프가 새로운 것에 비해 너무 작다는 사실로
**오래된 세그먼트**임을 구별할 수 있다.

---

### 기타 옵션

교재/시험에서 자주 나오지 않지만 실무에서 만날 수 있는 옵션들:

- **TCP MD5 Signature, TCP-AO**: BGP 등에서 TCP 세션을 인증하기 위한 보안 옵션.
- **MPTCP 관련 옵션**: 다중 경로 TCP(Multipath TCP)에서 서브플로 정보를 주고받기 위한 옵션들.
- **Fast Open 옵션**: TCP Fast Open에서 쿠키, 데이터 등을 SYN에 실어 미리 전송하기 위한 옵션.

이들은 각각 별도의 RFC에서 정의되며, 특정 프로토콜(예: BGP, HTTP/2 일부 환경, MPTCP)에서만 사용된다.

---

### 실전 예: tcpdump 스타일 세그먼트 헤더

다음은 실제 Wireshark/tcpdump에서 관찰할 수 있는 SYN 패킷의 헤더를 텍스트로 재구성한 예시다.

```text
IP 192.0.2.10.54321 > 198.51.100.20.443: Flags [S], seq 1000, win 64240, options [
    MSS 1460,
    SACKOK,
    TS val 123456789 ecr 0,
    WS 7
], length 0
```

- `MSS 1460`: 최대 세그먼트 크기
- `SACKOK`: SACK Permitted
- `TS val ...`: Timestamp 옵션
- `WS 7`: Window Scale shift = 7 → 유효 윈도우는 `win * 2^7`

여기까지가 TCP 혼잡 제어·타이머·옵션의 핵심 구조이다.
앞에서 공부한 **세그먼트 구조, 윈도우 기반 흐름 제어** 위에
이 세 가지가 겹쳐져서, 실제 Internet에서 보는 TCP의 성능·안정성이 결정된다.
