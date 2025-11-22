---
layout: post
title: 데이터 통신 - Media Access Control (1)
date: 2024-08-01 14:20:23 +0900
category: DataCommunication
---
# Chapter 12. MAC — Random Access: ALOHA, CSMA, CSMA/CD, CSMA/CA

이 장의 키워드:

- **ALOHA**: 완전 랜덤, 충돌 허용 → 간단하지만 효율 낮음
- **CSMA**: 보내기 전에 **듣고(Carrier Sense)**, 비어 있으면 보내기
- **CSMA/CD**: 유선 이더넷(802.3)에서 쓰던 **충돌 검출(Collision Detection)** 기반 MAC
- **CSMA/CA**: Wi-Fi(802.11)에서 사용하는 **충돌 회피(Collision Avoidance)** 기반 MAC

> 참고: 현대 이더넷은 대부분 **스위치 + 전이중(full-duplex)** 구조이기 때문에
> CSMA/CD는 **표준에 남아 있지만 실제로는 거의 쓰이지 않는 “레거시” 기술**이다.
> 반면 Wi-Fi(802.11)는 여전히 CSMA/CA를 MAC의 핵심으로 사용 중이다.

---

## Random Access MAC 개요

**Random Access MAC** 공통 특징:

- 노드들이 **사전에 정해진 순서/토큰 없이**,
  **필요할 때 마음대로 전송을 시도**한다.
- 그래서 **충돌(collision)** 가능성이 필연적으로 존재.
- 프로토콜은
  - 충돌을 **허용**할지, **회피**할지,
  - 충돌 후 **재전송을 언제 어떻게 할지**
  를 정의한다.

아주 추상적인 공통 인터페이스는 다음과 같다.

```text
while (전송할 프레임이 있을 때) {
    매체 상태를 확인(또는 안 하거나…)
    전송 시도
    → 성공하면 끝
    → 실패하면 (충돌) → 재전송 시간/방법 결정 → 다시 시도
}
```

이 틀 안에서:

- ALOHA: “매체 확인 안 하고 그냥 보냄, 충돌 나면 나중에 랜덤 시간 뒤 다시 보내자”
- CSMA: “보내기 전에 듣고, 비어 있으면 보내자”
- CSMA/CD: “보내면서도 **충돌을 감지**해서, 감지되는 즉시 중단 + 빠른 재시도”
- CSMA/CA: “무선이라 충돌 검출이 힘들다 → **충돌하기 전에** 랜덤 백오프, RTS/CTS로 회피”

---

## ALOHA — Pure & Slotted

### 역사와 기본 아이디어

- 1970년대 초, **하와이대학**에서 개발된 ALOHAnet에서 등장.
  무선 패킷 라디오 네트워크를 위해 설계된 **최초의 Random Access MAC** 중 하나.
- 핵심 철학: **“노드가 보낼 게 생기면 그냥 보내라. 충돌하면 나중에 다시 보내면 된다.”**

두 변종:

1. **Pure ALOHA**: 언제든지 프레임 시작 가능
2. **Slotted ALOHA**: 시간축을 슬롯으로 나누고, **슬롯의 시작에서만** 전송 시작 허용

결과적으로:

- Pure ALOHA: 최대 효율 ≈ **18.4%**
- Slotted ALOHA: 최대 효율 ≈ **36.8%**

---

### ALOHA 시스템 모델

공통 가정:

- **프레임 시간(frame time)** 을 기준 시간 단위로 사용.
- 평균적으로 프레임 시간당 **G** 개의 전송 시도가 발생한다고 하자.
  (모든 노드의 합을 의미하는 **총 offered load**)
- 이 전송 시도는 **포아송 과정**으로 모델링.

ALOHA에서 관심 있는 값:

- **Throughput S(G)**:
  “프레임 시간당 **성공적으로 전달된 프레임 수**”

즉, **보낸 양(G)** 과 **실제 성공한 양(S)** 의 관계를 본다.

---

### Pure ALOHA

#### 작동 방식

각 노드:

1. 전송할 프레임이 생기면 **즉시 전송**.
2. 수신 측(기지국 등)에서 **ACK** 를 기다림.
3. 일정 시간 내에 ACK를 받지 못하면,
   “충돌 또는 손실”로 간주하고 **랜덤 백오프 후 재전송**.

충돌의 “취약 구간(vulnerable period)”:

- 한 프레임의 전송 시간이 **1 frame time**이다.
- 내 프레임이 시간축에서 **시작 t에 시작해서 t+1에 끝난다**고 할 때,
- **[t−1, t+1] 구간**에 다른 프레임의 시작이 있으면,
  두 프레임이 부분적으로라도 겹치면서 충돌한다.
- 따라서 vulnerable period 길이 = **2 frame times**.

---

#### Throughput 분석

프레임 당 offered load = $$G$$ (프레임 시간당 평균 전송 시도 수).

- 어떤 특정 프레임이 **충돌 없이 성공**하려면,
  - 그 프레임의 vulnerable period(길이 2 프레임 시간) 동안
  - 다른 프레임의 시작이 **0개**여야 한다.

포아송 과정에서 길이 $$T$$ 동안 사건이 0번 발생할 확률은

$$
P_0(T) = e^{-\lambda T}
$$

여기서 $$\lambda = G / 1$$ (프레임 시간당 G개, 단위 시간=1 frame time) 이므로,

- vulnerable period = 2 → $$T = 2$$

성공 확률:

$$
P_{\text{success}} = e^{-2G}
$$

Throughput $$S$$ (프레임 시간당 성공 프레임 수):

$$
S(G) = G \cdot P_{\text{success}} = G e^{-2G}
$$



---

#### 최대 효율

$$
S(G) = G e^{-2G}
$$ 의 최대값을 구하자.

도함수:

$$
\frac{dS}{dG} = e^{-2G} - 2G e^{-2G} = e^{-2G}(1 - 2G)
$$

$$\frac{dS}{dG} = 0$$ 이 되는 G:

$$
1 - 2G = 0 \Rightarrow G_{\max} = \frac{1}{2}
$$

이때의 최대 throughput:

$$
S_{\max} = \frac{1}{2} e^{-1} \approx 0.1839
$$

즉, **프레임 시간당 채널 용량의 최대 18.4% 정도만** 유효 데이터로 쓸 수 있다.

---

### Slotted ALOHA

Pure ALOHA의 큰 약점: **충돌 취약 구간이 2 frame time** 이다.
이를 줄이기 위해 나온 아이디어:

> 시간축을 **슬롯(slot)** 으로 나누고, **각 슬롯 시작에서만 프레임 전송 시작**을 허용하자.

#### 작동 방식

- 모든 노드는 **공통 시계(clock)** 를 공유(슬롯 동기).
- 프레임 길이 = 슬롯 길이 (1 frame time).
- 노드가 프레임을 보내고 싶을 때:
  - **다음 슬롯의 시작까지 기다렸다가**, 그 시점에 전송을 시작.

이제 충돌 취약 구간:

- 내 프레임이 **슬롯 k** 에서 전송되면,
- 같은 슬롯 k에서 다른 노드가 동시에 전송을 시작할 때만 충돌 발생.
- 다른 슬롯에서는 겹치지 않음.

따라서 vulnerable period 길이는 **1 frame time** 으로 줄어든다.

---

#### Throughput 분석

슬롯당 전송 시도 수의 평균 = $$G$$ (포아송 근사).

- 어떤 슬롯에서 **성공적으로 전송**이 이루어지려면,
  - 그 슬롯에 전송 시도가 **정확히 1개**여야 한다.

포아송 분포에서:

$$
P\{X = 1\} = G e^{-G}
$$

따라서 슬롯당 throughput:

$$
S(G) = G e^{-G}
$$



---

#### 최대 효율

도함수:

$$
\frac{dS}{dG} = e^{-G} - G e^{-G} = e^{-G}(1 - G)
$$

$$\frac{dS}{dG} = 0$$ → $$G_{\max} = 1$$

$$
S_{\max} = 1 \cdot e^{-1} \approx 0.3679
$$

즉, **Slotted ALOHA 최대 효율 ≈ 36.8%**.

Pure ALOHA의 약 2배 효율.

---

### 예제 시나리오 & 타임라인

#### 예제 1: Pure ALOHA 충돌

프레임 길이를 1초로 가정하자.

- 노드 A: t = 0.1초에 전송 시작 → 1.1초에 끝
- 노드 B: t = 0.8초에 전송 시작 → 1.8초에 끝

타임라인:

```text
시간: 0       0.5       1.0       1.5       2.0 (초)

A:   [=======A 프레임=======]
        0.1 ---------------- 1.1

B:                [=======B 프레임=======]
                      0.8 ---------------- 1.8

겹치는 구간: 0.8 ~ 1.1  → 충돌
```

둘 다 ACK를 못 받으므로, 랜덤 백오프 후 재전송.

#### 예제 2: Slotted ALOHA에서의 충돌/성공

슬롯 길이 = 1 frame time = 1초. 슬롯 번호는 정수 시간 [0,1), [1,2), [2,3), …

- 노드 A: 슬롯 3에서 전송
- 노드 B: 슬롯 3에서 전송 → **충돌**
- 노드 C: 슬롯 4에서 전송 → **성공**

```text
슬롯:  0     1     2     3     4     5
시간: [0,1) [1,2) [2,3) [3,4) [4,5) ...

A:                   [A]
B:                   [B]      (슬롯 3, 충돌)
C:                         [C] (슬롯 4, 성공)
```

이처럼 Slotted ALOHA는 **충돌이 슬롯 단위로만 발생**하고,
Pure ALOHA보다 **충돌 확률이 낮다**.

---

### ALOHA 간단 시뮬레이션 코드 (Slotted ALOHA)

아래는 매우 단순화된 **Slotted ALOHA 시뮬레이터** 예시이다.
여기서 각 슬롯마다 노드 수 $$N$$, 각 노드의 전송 확률 $$p$$ 를 사용해 시도 수를 결정한다.

```python
import random
from collections import Counter

def slotted_aloha_sim(num_nodes=20, p=0.05, num_slots=100000):
    """
    num_nodes : 노드 수
    p         : 각 노드가 슬롯마다 전송을 시도할 확률
    num_slots : 시뮬레이션할 슬롯 수
    """
    success = 0
    collision = 0
    idle = 0

    for _ in range(num_slots):
        attempts = 0
        for _ in range(num_nodes):
            if random.random() < p:
                attempts += 1
        if attempts == 0:
            idle += 1
        elif attempts == 1:
            success += 1
        else:
            collision += 1

    return {
        "slots": num_slots,
        "success": success,
        "collision": collision,
        "idle": idle,
        "throughput": success / num_slots,
        "G_estimate": (success + collision) / num_slots,  # 슬롯당 평균 시도 수
    }

if __name__ == "__main__":
    result = slotted_aloha_sim(num_nodes=50, p=0.02, num_slots=50000)
    print(result)
    # throughput ≈ G*e^{-G} 패턴에 근접하는지 확인 가능
```

- 이 코드는 이론식 $$S = G e^{-G}$$ 와 실제 실험적인 결과가
  대략적으로 일치하는지 확인하는 데 사용할 수 있다.

---

## CSMA (Carrier Sense Multiple Access)

ALOHA는 전송 전에 매체를 **전혀 확인하지 않는다**.
그래서 **불필요한 충돌**이 너무 많다.

CSMA의 아이디어:

> “말하기 전에 주변이 조용한지 먼저 들어보자.”
> (Carrier Sense = 매체에 신호가 흐르는지 감지)

### 기본 규칙

모든 노드는 다음과 같이 동작한다.

1. 전송할 프레임이 생기면, **매체를 감시**한다(Carrier Sense).
2. 매체가 **idle(빈)** 이면:
   - **지금 전송을 시작**(다양한 변종에 따라 다소 차이).
3. 매체가 **busy(사용 중)** 이면:
   - 일정 전략에 따라 **기다렸다 다시 감시/전송**.

이때도 여전히 문제가 하나 남는다:

- 모든 노드의 매체 감시는 **전파 지연(propagation delay)** 를 가진다.
- 두 노드가 매체가 빈 것으로 “보이는” 시점이 달라,
  **동시에 전송을 시작 → 충돌**할 수 있다.

그래도 ALOHA에 비해 훨씬 낫다. 최소한 **매체가 분명 바쁜 동안에는 전송을 안 하니까**.

---

### CSMA의 변종: 1-persistent, non-persistent, p-persistent

일반적으로 다음 세 가지 전략을 구분한다.

#### 1-persistent CSMA

규칙:

- 매체가 busy이면 **“쭉” 감시**하다가,
- idle이 되는 순간 **즉시 전송**.

의사코드:

```python
while True:
    if medium_is_idle():
        send_frame()
        wait_for_ack_or_timeout()
        break
    # 1-persistent: 그냥 계속 listen
```

장점:

- 매체가 비자마자 바로 전송 → **지연이 최소화**.

단점:

- 여러 노드가 동시에 busy → idle 전환을 보게 되면,
  - 모두 동시에 전송 시작 → **높은 충돌 확률**.

---

#### Non-persistent CSMA

규칙:

- 매체가 busy이면, **무작위 시간 동안 대기(backoff)** 했다가 다시 감시.

```python
while True:
    if medium_is_idle():
        send_frame()
        wait_for_ack_or_timeout()
        break
    else:
        wait(random_backoff_time())
        # 다시 medium_is_idle()을 보러 감
```

장점:

- 1-persistent보다 **충돌 확률이 낮음**
  (idle 되자마자 다 같이 덤비지 않으므로).

단점:

- 매체가 비어 있는데도, 일부 노드는 아직 backoff 중일 수 있음 → **지연 증가**.

---

#### p-persistent CSMA (슬롯 기반)

주로 **슬롯형 LAN**(예: slotted CSMA)에서 사용.

규칙:

1. 매체가 idle이면,
   - **각 슬롯의 시작에서** 확률 $$p$$ 로 전송,
   - 확률 $$1-p$$ 로 그 슬롯에서는 쉬고 다음 슬롯으로 넘어간다.
2. 매체가 busy이면,
   - idle이 될 때까지 기다린 뒤, 다시 1 수행.

```python
while True:
    wait_until_slot_boundary()

    if medium_is_idle():
        if random.random() < p:
            send_frame()
            wait_for_ack_or_timeout()
            break
        else:
            # 이번 슬롯은 쉬고 다음 슬롯으로
            continue
    else:
        # busy이면 idle 될 때까지 대기
        wait_until_medium_idle()
```

- $$p$$ 값이 클수록 “공격적”, 작을수록 “보수적”.
- 여러 노드가 동시에 대기 중인 상황에서,
  - 각 노드가 독립적으로 확률 $$p$$ 로 전송을 선택 → **충돌 확률 제어**.

---

### CSMA의 취약 구간과 한계

ALOHA에 비해 충돌 확률이 훨씬 줄어들지만,
CSMA도 여전히 **전파 지연 때문에 완전한 충돌 제거는 불가능**하다.

예:

- 한 케이블 길이가 2km이고, 위/아래 끝에 각각 A, B 노드가 있다고 하자.
- 매체 전파 속도를 약 $$2 \times 10^8\ \text{m/s}$$ 로 잡으면,
  - 한 끝에서 다른 끝까지 전파 지연 ≈ $$10\ \mu s$$.

상황:

1. A는 t=0에서 매체를 보고 idle이라고 판단, 전송 시작.
2. 이 신호가 B에게 도착하는 시점은 t=10µs.
3. B는 t=5µs 시점까지도 매체를 idle로 보고 있다.
4. B도 t=5µs에 전송을 시작할 수 있다.
5. 결국 A의 프레임과 B의 프레임이 **중간 어딘가에서 충돌**한다.

→ 매체 감시를 해도, **동시에 시작해버리는 충돌**은 막을 수 없다.

이때 등장하는 발전형이 **CSMA/CD**, **CSMA/CA** 이다.

---

## CSMA/CD — 유선 이더넷의 충돌 검출

### 개념과 배경

**CSMA/CD(Carrier Sense Multiple Access with Collision Detection)** 는
초기 **공유 버스형 이더넷(IEEE 802.3)** 에서 사용된 MAC이다.

- **CSMA**: 보내기 전에 듣는다.
- **CD**: 보내면서도 **충돌이 일어나는지 감지**한다.
- 충돌이 감지되면:
  - **즉시 전송을 중단**하고,
  - **Jam 신호**를 짧게 내보낸 뒤,
  - **지수 백오프(Exponential Backoff)** 로 랜덤 지연 후 재시도.

> 현대 스위치 기반 전이중 이더넷에서는
> **충돌이 물리적으로 일어날 수 없으므로** CSMA/CD가 필요 없다.
> 하지만 **802.3 표준과 네트워크 교육에서는 여전히 핵심 개념**으로 다뤄진다.

---

### 동작 절차(고전 버스형 이더넷 기준)

한 노드가 프레임을 전송하는 과정:

1. **Carrier Sense**:
   - 매체(공유 케이블)를 감시.
   - idle이면 전송 시작, busy이면 대기.

2. **전송 시작 후 충돌 감시**:
   - 전송하는 동안 **수신된 신호 세기/패턴**을 모니터링.
   - 내가 보내는 신호와 다르면 → 충돌 발생으로 판단.

3. **Collision Detection(CD)**:
   - 충돌을 감지하면 즉시 **전송 중단**.
   - 케이블 전체가 충돌을 인지할 수 있도록 **Jam 신호**(일정 패턴)를 짧게 전송.

4. **Binary Exponential Backoff**:
   - 재전송을 위해 랜덤 백오프 슬롯 수를 결정.
   - 충돌 횟수 $$k$$ (0부터 시작)에 대해,
     - $$k \le 10$$이면, $$0 \le r < 2^k$$ 중 랜덤 $$r$$ 선택.
     - $$10 < k \le 16$$이면, $$0 \le r < 2^{10}$$.
     - $$k > 16$$이면, 전송 실패로 간주하고 포기.
   - 실제 백오프 시간 = $$r \times \text{slot time}$$.

여기서 **slot time** 은 10Mbps 이더넷에서 512bit-time(= 51.2µs)로 정의되어,
충돌을 전체 네트워크에서 확실히 감지하기 위한 최소 프레임 길이 등을 결정한다.

---

### 최소 프레임 길이와 슬롯 시간

**CSMA/CD가 제대로 동작**하려면:

- 송신자가 프레임을 다 보내기 전에
  **네트워크 상 어떤 두 지점에서 시작된 충돌도 반드시 감지**할 수 있어야 한다.

이를 위해:

- 프레임 전송 시간 ≥ **2 × 최대 전파 지연**
  (프레임이 케이블을 왕복하는 동안은 항상 아직 전송 중이어야 충돌 감지 가능)

10Mbps, 최대 세그먼트 길이/허브 수 등을 고려해 계산한 결과:

- **slot time = 512 bit-times** 로 정의.
- 최소 프레임 크기 = 512 bits = 64 bytes
  (이더넷 헤더/트레일러 합쳐서 64바이트가 나오도록 패딩).

이 값들은 **IEEE 802.3** 의 설계와 실험을 통해 정해졌으며,
Fast Ethernet/Gigabit Ethernet에서도 호환을 위해 slot time과 관련 파라미터를 유지/변형했다.

---

### 동작 예시: 두 노드 A/B 간 충돌

상황:

- 공유 케이블의 양 끝에 A, B 노드가 있다.
- 둘 다 CSMA/CD를 사용한다.

타임라인(단순화):

```text
t=0   : A, 매체 idle 확인 → 전송 시작
t=τ/2 : B, 아직 A의 신호가 도달하기 전 → 매체 idle로 보임 → 전송 시작
t=τ   : 중간 지점에서 A의 프레임과 B의 프레임이 만나 충돌
        충돌 신호가 A와 B 쪽으로 전파되기 시작
t=1.5τ: A, 자신이 보내는 신호와 다른 패턴을 감지 → 충돌 감지 → 전송 중단 + Jam
t=1.5τ: B도 동일하게 충돌 감지
이후   : A, B 모두 Binary Exponential Backoff에 따라 랜덤 슬롯 뒤 재전송
```

여기서 $$τ$$ 는 한쪽 끝에서 다른 끝까지의 전파 지연.

---

### CSMA/CD 전송 절차 의사코드

아래 코드는 **개념적인 수준**의 CSMA/CD 송신 동작을 보여준다.

```python
import random
import time

SLOT_TIME = 51.2e-6   # 10Mbps 기준 예시 (초 단위), 개념용
MAX_RETRY = 16

def transmit_frame(frame):
    """
    frame: 전송할 프레임 (bytes 등)
    실제 채널/하드웨어 인터페이스는 생략하고
    CSMA/CD 흐름만 개념적으로 표현
    """
    attempt = 0

    while attempt <= MAX_RETRY:
        # 1. Carrier Sense
        while medium_is_busy():
            # 다른 노드가 사용 중이면 기다린다.
            time.sleep(SLOT_TIME)

        # 2. 전송 시작
        start_transmission(frame)

        # 3. 전송 중 충돌 감지
        while is_transmitting():
            if collision_detected():
                # 충돌 처리
                stop_transmission()
                send_jam_signal()
                attempt += 1

                if attempt > MAX_RETRY:
                    print("전송 실패: 최대 재시도 횟수 초과")
                    return False

                # Binary Exponential Backoff
                k = min(attempt, 10)
                r = random.randint(0, (1 << k) - 1)
                backoff_time = r * SLOT_TIME
                time.sleep(backoff_time)
                break
        else:
            # while is_transmitting() 루프를 충돌 없이 끝냈다면 성공
            print("전송 성공")
            return True

    return False
```

실제 NIC/드라이버는 더 복잡한 상태머신과 하드웨어 타이머, PHY 레벨 감지 기능을 사용하지만,
**CSMA/CD의 논리 구조 자체는 위와 비슷한 형태**를 가진다.

---

### 현대 이더넷에서 CSMA/CD의 위치

- **Hub 기반 공유 미디어(버스/리피터)** 환경이 사실상 사라짐.
- 오늘날 대부분의 이더넷 링크는:
  - 스위치 ↔ 호스트, 스위치 ↔ 스위치
  - **각 링크가 독립된 점대점 전이중(full-duplex)** 연결
  - 충돌이 물리적으로 불가능 → **CSMA/CD 불필요**

그래서:

- IEEE 802.3 표준에는 여전히 CSMA/CD와 half-duplex 모드가 정의되어 있지만,
  **실제 상용 네트워크에서는 거의 모든 링크가 full-duplex, CSMA/CD 미사용**이다.

하지만 네트워크 이론/시험에서는 **고전 이더넷 구조와 CSMA/CD 알고리즘**이 여전히 필수 지식이다.

---

## CSMA/CA — 무선 LAN(Wi-Fi)의 충돌 회피

### 왜 무선에서는 CSMA/CD가 안 되는가?

유선에서는 송신 노드가 **송신 신호 vs 수신 신호**를 비교해
충돌(왜곡)을 직접 감지할 수 있다(CD).

하지만 **무선(802.11 WLAN)** 에서는:

- 송신 노드는 자기 송신 신호가 너무 강해서,
  - 동시에 들어오는 다른 노드의 신호를 **실시간으로 구분/비교하기 어렵다.**
- 무선 채널은 **다중 경로, 페이딩, 잡음** 등으로 인해
  충돌 여부를 **즉시 판단하기 어렵다.**
- 또한 **Hidden Node 문제**:
  - AP와는 통신 가능하지만, 서로는 들리지 않는 두 단말이
    동시에 AP로 전송해 충돌하는 상황.

그래서 IEEE 802.11은 **CSMA/CD 대신 CSMA/CA(Carrier Sense Multiple Access with Collision Avoidance)** 를 사용한다.

---

### CSMA/CA의 기본 원칙

요약:

> **충돌을 감지하는 대신, 최대한 “충돌하지 않도록” 설계**한다.

핵심 메커니즘:

1. **Carrier Sense**:
   - 채널이 idle인지 busy인지 감지(물리/가상 캐리어 감지).
2. **랜덤 백오프**:
   - 채널이 idle이 된 후에도, **바로 보내지 않고 랜덤 시간 동안 대기**.
3. **ACK 기반 오류 검출**:
   - 수신 측이 데이터 프레임마다 **ACK 프레임**을 보냄.
   - ACK를 받지 못하면 충돌 또는 손실로 판단 → 재전송.
4. **RTS/CTS (옵션)**:
   - 긴 프레임 전송 전에 소규모 제어 프레임으로
     **채널 예약(reservation)** 을 수행해 Hidden Node 문제 완화.

이 구조는 **실시간 충돌 검출 대신, 사전 회피 + 사후 ACK** 로 신뢰성을 확보한다.

---

### DCF(Distributed Coordination Function)의 CSMA/CA 절차

802.11의 기본 MAC은 DCF로, CSMA/CA를 기반으로 한다. (단순화 버전)

#### 용어

- **IFS(Inter-Frame Space)**:
  - 프레임 사이에 두는 정해진 시간 간격.
  - DIFS, SIFS 등 여러 종류.
- **DIFS (DCF IFS)**:
  - 일반 데이터 프레임을 보내기 전에 기다리는 최소 시간.
- **SIFS (Short IFS)**:
  - ACK, CTS 등 **우선순위 높은 응답 프레임**이 보내질 때 사용하는 더 짧은 간격.
- **CW(Contension Window)**:
  - 백오프 카운트 범위 [0, CW] (슬롯 단위).

#### 절차 (데이터 전송):

1. 전송할 데이터가 생김.
2. 채널이 **idle** 인지 감시.
   - busy이면, 채널이 idle이 될 때까지 기다림.
3. 채널이 idle이 된 후, **DIFS 만큼 더 기다림**.
4. 그 후, [0, CW] 범위에서 **무작위 백오프 카운트**를 선택.
5. 백오프 카운트가 0이 될 때까지,
   - 각 슬롯마다 카운트를 1씩 줄여 나감.
   - 이때 **채널이 busy로 변하면**, 카운트 감소를 멈추고,
     - 다시 idle + DIFS 이후에 남은 카운트부터 재개.
6. 카운트가 0이 되면 데이터 프레임 전송.
7. 수신 측은 프레임을 받으면, **SIFS 후 ACK 프레임**을 보냄.
8. 송신자가 **ACK를 일정 시간 안에 받으면 성공**,
   받지 못하면 실패로 간주하고 CW를 늘려 **재전송**.

이 과정에서 **모든 노드가 랜덤한 백오프 카운트**를 사용하므로,
동시에 전송할 확률이 낮아지고, 충돌이 크게 줄어든다.

---

### RTS/CTS 메커니즘 (옵션)

**Hidden Node 문제**를 줄이기 위한 선택적 기능.

- 두 노드 A, C가 AP B와는 통신 가능하지만,
  - 서로는 듣지 못하는 상황에서,
  - A, C가 동시에 B에게 데이터 전송 → 충돌.

RTS/CTS 절차(간단 버전):

1. 데이터가 큰 프레임일 경우, 송신자 A는 먼저 작은 제어 프레임 **RTS(Request To Send)** 전송.
2. AP B는 RTS를 받으면 **CTS(Clear To Send)** 를 브로드캐스트.
3. CTS를 들은 모든 노드는 (A 포함)
   - CTS에 포함된 **Duration** 필드를 보고,
   - 그 시간 동안 **채널을 비워야 한다(NAV 설정)** 라는 사실을 안다.
4. A는 CTS를 받은 후, 본 데이터 프레임을 전송.
5. B는 데이터 수신 후 SIFS 뒤 ACK를 보냄.

Hidden Node C의 관점:

- C는 A의 RTS는 못 들었지만,
  B의 CTS는 들을 수 있다.
- CTS에서 “A가 지금부터 몇 µs 동안 전송할 예정”이라는 정보를 보고,
  - NAV(Network Allocation Vector)를 설정해 **그동안은 전송을 자제**한다.

이렇게 RTS/CTS는:

- 긴 프레임이 충돌했을 때 손실되는 시간/대역폭을 줄이고,
- Hidden Node로 인한 충돌을 완화한다.

---

### CSMA/CD vs CSMA/CA 비교

| 항목 | CSMA/CD | CSMA/CA |
|------|---------|---------|
| 사용 매체 | 주로 유선(고전 이더넷, 802.3) | 무선 LAN (Wi-Fi, 802.11)  |
| 핵심 아이디어 | 충돌이 나면 **감지 후 중단 + 재시도** | 충돌이 나기 전에 **랜덤 백오프, RTS/CTS로 회피** |
| 충돌 검출 | 송신 중에 전압/신호 비교로 감지 | 현실적으로 불가 (무선 환경) |
| 오류 확인 | 충돌 감지 + 재전송, 상위 계층 | 데이터마다 ACK 프레임, ACK 없으면 재전송 |
| 프레임 전송 후 | 별도 ACK 필요 없음 (링크 아래층) | 수신 측이 ACK 프레임 전송 |
| 현대 사용 | 스위치 기반 전이중에서 거의 미사용 | Wi-Fi MAC의 핵심 메커니즘 |

---

### 간단 CSMA/CA 백오프 시뮬레이션 코드 (개념)

아주 단순화된 DCF의 백오프 부분만 파이썬으로 표현해 보자.

```python
import random
import time

SLOT_TIME = 9e-6      # 예: 802.11a/g의 기본 슬롯 시간(개념용)
DIFS = 34e-6          # 예: DIFS (개념용)
CW_MIN = 15
CW_MAX = 1023

def csma_ca_transmit(frame, cw_min=CW_MIN, cw_max=CW_MAX, max_retry=7):
    """
    frame: 전송 데이터 (생략)
    cw_min, cw_max: 컨텐션 윈도우 범위
    max_retry: 최대 재시도 횟수 (단순화)
    """

    cw = cw_min
    retry = 0

    while retry <= max_retry:
        # 1. 채널이 idle이 될 때까지 기다림
        while medium_is_busy():
            time.sleep(SLOT_TIME)

        # 2. DIFS 동안 채널이 idle인지 확인
        time.sleep(DIFS)
        if medium_is_busy():
            # 다시 처음부터
            continue

        # 3. 백오프 카운트 선택
        backoff_slots = random.randint(0, cw)
        while backoff_slots > 0:
            if medium_is_busy():
                # 채널이 busy가 되면, idle + DIFS 이후에 남은 카운트부터 다시 시작
                while medium_is_busy():
                    time.sleep(SLOT_TIME)
                time.sleep(DIFS)
                continue
            time.sleep(SLOT_TIME)
            backoff_slots -= 1

        # 4. 카운트가 0이 되면 전송 시도
        send_frame(frame)

        # 5. ACK 대기 (간단화)
        if wait_for_ack(timeout=0.1):   # 100ms 타임아웃 예시
            print("전송 성공")
            return True
        else:
            print("충돌 또는 손실, 재시도")
            retry += 1
            # 이진 지수 백오프
            cw = min((cw + 1) * 2 - 1, cw_max)

    print("전송 실패: 최대 재시도 횟수 초과")
    return False
```

실제 802.11 스택은:

- SIFS, PIFS, DIFS, EIFS 등 여러 IFS
- RTS/CTS 핸들링
- 에너지 검출, NAV, 멀티 레이트, QoS(EDCA) 등
을 포함하지만, 위 코드는 CSMA/CA의 **핵심 아이디어(랜덤 백오프 + ACK 기반 재시도)** 를 보여준다.

---

## 정리

이 섹션에서 다룬 **Random Access MAC** 을 다시 정리하면:

1. **ALOHA**
   - 순수 랜덤 접근, 매체 감시 없이 “생기면 바로 전송”.
   - Pure ALOHA: $$S = G e^{-2G}$$, 최대 $$S_{\max} = \frac{1}{2e} \approx 18.4\%$$.
   - Slotted ALOHA: $$S = G e^{-G}$$, 최대 $$S_{\max} = \frac{1}{e} \approx 36.8\%$$.

2. **CSMA**
   - “보내기 전에 듣기”로 ALOHA의 불필요한 충돌을 크게 줄임.
   - 1-persistent / non-persistent / p-persistent 전략으로
     지연 vs 충돌 확률을 절충.

3. **CSMA/CD**
   - 고전 유선 이더넷(802.3)의 MAC.
   - Carrier Sense + Collision Detection + Binary Exponential Backoff.
   - 최소 프레임 길이, slot time 같은 설계가 **충돌 검출 가능성**을 보장.
   - 오늘날엔 **스위치 + 전이중 링크**가 기본이어서 실제로는 거의 사용되지 않음.

4. **CSMA/CA**
   - Wi-Fi(802.11)의 MAC.
   - 무선 환경에서 충돌을 실시간 검출할 수 없으므로,
     - 랜덤 백오프, DIFS/SIFS,
     - ACK 프레임,
     - RTS/CTS(옵션)
     로 충돌 **회피** 및 재전송을 구현.
   - Hidden Node 문제 대응, QoS 확장(EDCA) 등에 기반.

이 구조를 이해하면, 이후에 나오는 **802.3(Ethernet) 상세 MAC, 802.11(Wi-Fi) MAC, 셀룰러 Random Access(5G RACH) 등**을 볼 때
각 프로토콜이 ALOHA/CSMA/CSMA/CD/CSMA/CA 패밀리 중 어디에 위치하고,
어떤 현실적인 제약(유선/무선, 전파지연, 하드웨어) 때문에 설계가 달라졌는지 쉽게 연결해서 이해할 수 있다.
