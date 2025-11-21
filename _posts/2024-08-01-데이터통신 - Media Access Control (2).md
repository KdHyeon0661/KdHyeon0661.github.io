---
layout: post
title: 데이터 통신 - Media Access Control (2)
date: 2024-08-01 15:20:23 +0900
category: DataCommunication
---
# Controlled Access — Reservation, Polling, Token Passing

이전 12.1에서 본 **랜덤 접근(Random Access, ALOHA/CSMA/CSMA-CD/CSMA-CA)** 은
“필요하면 알아서 전송을 시도하고, 충돌 나면 다시 보내는” 철학이었다.

**12.2 Controlled Access** 는 정반대에 가깝다:

> 누가 언제 매체를 쓸 수 있는지를 미리 정하거나,
> 중앙 제어자 혹은 순환하는 토큰으로 **엄격하게 통제**한다.

그래서:

- **충돌(collision)** 을 원칙적으로 없애거나 매우 줄이고,
- **지연 지터(jitter)** 를 줄이며,
- **실시간(Real-time)·산업용·케이블 모뎀·USB·토큰 링** 등에서 쓰인다.

이 절에서는 세 가지 대표적인 Controlled Access 기법을 다룬다.

- **Reservation (예약)**
- **Polling (폴링)**
- **Token Passing (토큰 전달)**

---

## Controlled Access 개요

### Random Access vs Controlled Access

간단 비교부터 정리하자.

| 항목 | Random Access (ALOHA/CSMA 등) | Controlled Access (Reservation, Polling, Token) |
|------|-------------------------------|-----------------------------------------------|
| MAC 철학 | “먼저 온 사람이 먼저 말한다. 충돌 나면 다시” | “언제 말해도 되는지 **규칙/순서**가 있다” |
| 충돌 | 빈번, 설계상 허용 | 보통 없음 또는 극히 드묾 |
| 지연 | 평균은 짧을 수 있지만, 분산 큼 | 평균은 길 수 있으나 **상한을 줄 수 있음** |
| 공평성 | 트래픽 많을수록 불공평 가능 | 구조적으로 공평(토큰/폴링 순서 등) |
| 구현 복잡도 | 단순 | 중앙 제어/토큰 관리 필요 |
| 대표 예 | Wi-Fi DCF, 고전 이더넷 CSMA/CD | DOCSIS 케이블 upstream, USB, Token Ring/Bus, TSN 등  |

Controlled Access는 특히

- **케이블 모뎀(DOCSIS)의 upstream 예약/폴링**,
- **USB에서 호스트가 엔드포인트를 주기적으로 폴링하는 구조**,
- **IEEE 802.4 Token Bus, 802.5 Token Ring, FDDI 같은 토큰 패싱 기반 LAN**,
- **산업/자동차용 Deterministic Ethernet·TSN**

에서 필수적인 디자인이다.

---

## Reservation (예약 기반 MAC)

### 직관적인 비유

회의실 예약을 떠올리면 이해하기 쉽다.

- 하루를 30분 슬롯으로 나누고,
- **“내가 어느 슬롯에 쓰겠다”** 를 미리 예약하면,
- 실제 사용 시간에는 **충돌 없이** 회의를 진행할 수 있다.

Reservation MAC도 비슷하다:

1. 시간 축을 **슬롯/프레임** 단위로 나눈다.
2. 각각의 사이클(cycle) 시작 부분에 **예약 기회(reservation phase)** 가 있다.
3. 각 노드는 “나 이 사이클에서 몇 슬롯 필요해” 를 **짧은 비트/메시지로 신호**한다.
4. 스케줄러(또는 분산 알고리즘)가 그 예약 정보를 모아 **슬롯을 할당**한다.
5. 실제 데이터 전송 구간에서, 각 노드는 **할당 받은 슬롯에서만 전송**한다.

이렇게 하면:

- 데이터 전송 구간에서는 충돌이 없다.
- 실시간 트래픽은 **주기적/최소 대역폭을 보장**받을 수 있다.

---

### 추상적인 Reservation 프레임 구조

시간축 그림으로 그려보면:

```text
|<-----------  하나의 MAC 사이클 (T_cycle)  ----------->|
+----------------+--------------------------------------+
| Reservation    |             Data Transmission        |
| (control bits) |    Slot 1   Slot 2   Slot 3   ...    |
+----------------+--------------------------------------+
```

- **Reservation 구간**:
  - N개 노드를 지원한다면, 각 노드를 위한 비트를 하나씩 둘 수도 있고,
  - 더 복잡한 필드(필요 슬롯 수, QoS 요구 등)를 넣을 수도 있다.
- **Data 구간**:
  - 예약/할당 결과에 따라 **Time Slot** 을 각 노드에 배정한다.
  - 예: 노드 1이 슬롯 1,3 / 노드 2가 슬롯 2,4 … 등.

기초적인 수학적 표현:

- 사이클 길이를 $$T_{\text{cycle}}$$,
- 그 중 예약 구간 길이를 $$T_{\text{res}}$$,
- 데이터 구간 길이를 $$T_{\text{data}}$$ 라 하면,

$$
T_{\text{cycle}} = T_{\text{res}} + T_{\text{data}}
$$

유효 데이터 효율(Overhead 제외 순수 payload 비율)은

$$
\eta = \frac{T_{\text{data}}}{T_{\text{cycle}}}
= \frac{T_{\text{data}}}{T_{\text{res}} + T_{\text{data}}}
$$

- Reservation 구간이 너무 크면 $$\eta$$ 가 줄어들지만,
- 너무 작으면 복잡한 QoS/다수 노드 예약을 표현하기 어렵다.

---

### 예제: 간단 TDMA + Reservation

노드 4개 (A,B,C,D)가 하나의 버스를 공유하고 있다고 한다.

- 사이클 = 예약 구간 1 × 4비트 + 데이터 슬롯 4개.
- 예약 구간: 각 비트는 “해당 노드가 이번 사이클에 한 슬롯 필요함”을 의미.
  - 첫 비트= A, 둘째 = B, 셋째 = C, 넷째 = D.
- Data 슬롯: 슬롯 1~4.

사이클 k에서 예약 비트가 `1010` 이었다면:

```text
예약 비트: [A=1][B=0][C=1][D=0]

슬롯 할당:
  Slot 1 → A
  Slot 2 → C
나머지 슬롯은 비워두거나, 다른 정책(중복 예약/베스트 effort)에 사용
```

타임라인(예):

```text
Cycle k:
  Reservation:  A    B    C    D
                 |    |    |    |
                 1    0    1    0

  Data Slots: [Slot1][Slot2][Slot3][Slot4]
               A:data   C:data   idle   idle
```

Cycle k+1에서 예약 비트가 `0111`이면, 그에 맞게 다시 재할당한다.

---

### 실제 시스템 예: DOCSIS 케이블 모뎀 upstream

케이블 인터넷(DOCSIS)은 **케이블 모뎀들이 동일한 HFC(Hybrid Fiber-Coaxial) 업스트림 채널을 공유**한다.
이 MAC은 **순수 랜덤 접근이 아니라, 예약·폴링 기반의 중앙 통제 MAC** 을 사용한다.

구조 요약:

- 중앙 제어자: **CMTS (Cable Modem Termination System)**.
- 각 모뎀(CM: Cable Modem)은 upstream으로 데이터를 보내기 위해
  - **Request Slot** 에서 대역폭을 요청하거나,
  - CMTS가 미리 예약한 **Grant** 를 사용한다.
- CMTS는 **MAP 메시지** 를 downstream으로 브로드캐스트하여,
  - “어느 시간 슬롯에 어떤 모뎀이 전송해라” 라고 공지.

DOCSIS MAC의 특징:

- upstream 채널은 **미니슬롯(minislot)** 단위로 쪼개짐.
- CMTS는 각 서비스 흐름(Service Flow, 예: VoIP, 비디오, BE data)에 대해
  **예약 기반 스케줄링 (UGS, rtPS, nrtPS, BE 등)** 을 수행한다.

예: rtPS(real-time Polling Service) 흐름

- CMTS가 특정 모뎀을 **주기적으로 폴링/예약**해,
  - 해당 모뎀이 “이번 주기에서 얼마만큼의 대역폭이 필요하다”고 업스트림으로 요청할 기회를 준다.
- CMTS는 그 정보를 토대로 다음 MAP에서 **정확한 데이터 슬롯을 할당**한다.

DOCSIS 측에서 보면:

- **Request/Grant를 통한 예약** +
- **Service Flow 별 QoS 파라미터(대역폭, 지연, 우선순위)** 를 이용해
  정교한 MAC 스케줄링을 구현한다.

---

### Reservation의 지연/효율 분석 예시

예를 들어:

- 사이클 길이 $$T_{\text{cycle}} = 10\text{ ms}$$,
- 예약 구간 $$T_{\text{res}} = 1\text{ ms}$$,
- 데이터 구간 $$T_{\text{data}} = 9\text{ ms}$$ 라고 하자.

효율은

$$
\eta = \frac{9}{10} = 0.9 = 90\%
$$

이 중 일부는 실제 데이터가 없어서 비워질 수 있다.
특히 예약만 했는데 실제 데이터가 없으면, 해당 슬롯이 낭비된다.

보통은:

- 실시간 서비스(음성·영상)는 **주기적 예약**으로 지연·지터를 제한,
- Best-effort 트래픽은 남은 슬롯에서
  (혹은 별도 우선순위 큐에서) 알고리즘에 따라 전송.

Reservation MAC의 장점:

- 실시간 트래픽에 대해 **최대 지연/대역폭을 보장**하기 쉽다.
- 충돌이 거의 없고 예측 가능한 성능.

단점:

- 수요가 적은 시기에 예약된 슬롯이 **비워지는 비효율**.
- 예약을 위한 **제어 메시지 오버헤드**.

그래서 실제 시스템에서는 Reservation을 Polling, Random Access와 **혼합**해서 사용한다(예: DOCSIS의 혼합 서비스 클래스).

---

### 간단 Reservation 시뮬레이션 코드

아주 단순화된 “예약 비트 + 데이터 슬롯” 구조를 파이썬으로 흉내내 보자.

- N개 노드,
- 각 사이클마다 각 노드가 **새 패킷을 가질 확률** $$p$$,
- 예약 비트가 1인 노드에게 순서대로 슬롯을 배정.

```python
import random
from collections import defaultdict

def simulate_reservation(num_nodes=4, p=0.3, num_cycles=10000, slots_per_cycle=4):
    """
    간단 Reservation 기반 MAC 시뮬레이션.
    - 각 사이클: 예약 비트 → 최대 slots_per_cycle 개의 슬롯 할당
    - 각 노드는 독립적으로 확률 p로 '전송할 패킷 있음' 상태가 됨
    """
    sent = [0] * num_nodes
    queued = [0] * num_nodes  # 간단히 '대기 중 패킷 수'만 추적
    total_slots = num_cycles * slots_per_cycle

    for _ in range(num_cycles):
        # 1. 예약 단계: 새 패킷 생성
        need = []
        for i in range(num_nodes):
            if random.random() < p:
                queued[i] += 1
            if queued[i] > 0:
                need.append(i)

        # 2. 슬롯 할당: need 리스트 순서대로 슬롯 배정
        #    (단순화를 위해, 한 노드는 최대 1 슬롯만)
        allocated = need[:slots_per_cycle]

        # 3. 데이터 전송
        for i in allocated:
            if queued[i] > 0:
                queued[i] -= 1
                sent[i] += 1

    throughput = sum(sent) / total_slots
    return {
        "sent_per_node": sent,
        "throughput_per_slot": throughput,
        "avg_queue": sum(queued) / num_nodes,
    }

if __name__ == "__main__":
    result = simulate_reservation()
    print(result)
```

이 코드는 Reservation MAC의

- 예약 비트 자체의 오버헤드는 무시했지만,
- 예약된 노드에게만 슬롯을 할당 → 충돌 없이 전송,
- 트래픽 수준(p)이 너무 낮을 때 슬롯이 비워질 수 있음

같은 특성을 간단히 실험해 볼 수 있다.

---

## Polling (폴링 기반 MAC)

### 직관적인 비유

교실에서 선생님이 질문을 던지고, **학생들을 한 명씩 이름을 부르며** 답하게 하는 상황:

- 학생이 마음대로 손들고 동시에 떠들면 엉망이 되니,
- 선생님이 **“A, 너 먼저 말해봐 → B, 다음은 너”** 식으로 순서를 정한다.

네트워크에서 Polling MAC도 똑같다.

> **중앙 제어자(Master, Host, Headend)** 가
> 각 노드를 순서대로 “지금 말해도 됨” 이라고 묻고,
> 노드는 그때만 전송한다.

충돌은 발생하지 않는다. 다만:

- 중심이 되는 **마스터 노드가 반드시 존재**해야 하고,
- 마스터와 각 노드 사이의 **폴링 오버헤드**가 있다.

---

### Polling MAC의 기본 구조

N개 노드가 있을 때, 단순 Polling 알고리즘:

1. Master가 노드 1에게 Poll 메시지 송신.
2. 노드 1이
   - 데이터가 있으면 → 데이터 전송 후 “끝났음” 표시.
   - 없으면 → 그대로 “없음” 응답 혹은 Timeout.
3. Master가 노드 2로 이동, … … N까지 반복.
4. N까지 끝나면 다시 1로 돌아와서 반복.

타임라인 예시:

```text
시간 →
Master:  P(1)             P(2)                  P(3)        ...
Node1:       [DATA1........]
Node2:                    [no data]
Node3:                                         [DATA3....]
```

여기서 P(i)는 i번 노드를 폴링하는 control 프레임.

---

### Polling의 종류

- **Roll-call polling**:
  - 순서대로 1→2→…→N→1→… 폴링.
- **Hub polling (polling-by-token)**:
  - Master가 첫 노드에 poll를 보내고,
  - 각 노드가 poll를 **다음 노드로 넘기는 방식**.
  - Poll 프레임이 링을 도는 구조라 **토큰 패싱과 유사**하다.

---

### 실제 시스템 예 1: USB — Host 중심 Polling

USB는 논리적으로 **항상 Host(PC, 스마트폰 등)가 주도**하는 버스다.

- USB에서는 장치가 임의로 호스트를 “인터럽트”할 수 없고,
- 호스트가 **주기적으로(주어진 인터벌에 따라)** 각 엔드포인트를 **폴링**해서,
  - 데이터가 준비됐으면 그때 읽어가는 구조이다.

중요 포인트:

- **Interrupt transfer** 라는 이름이 있지만, 실제로는 **polling**:
  - 호스트 컨트롤러가 지정된 주기(예: 1ms frame, USB 2.0에서 125µs micro-frame 등)마다
    해당 엔드포인트에 IN 토큰을 보내 “데이터 있냐?” 를 물어본다.
- 장치 입장에서는 **“데이터가 준비되어 있으면 다음 폴링 때 응답”** 하는 식이다.

이 구조 덕분에:

- 버스상 충돌이 없고,
- 호스트가 전력/대역폭 관리를 중앙에서 통제할 수 있다.

---

### 실제 시스템 예 2: DOCSIS Real-time Polling Service (rtPS)

앞에서 Reservation에서 언급한 **DOCSIS** 는,
특정 서비스 흐름에 대해 **Real-time Polling Service(rtPS)** 를 제공한다.

개략적 동작:

1. CMTS(헤드엔드)가 각 CM(케이블 모뎀)을 주기적으로 **폴링**한다.
2. 폴링 시점에 CM은
   - 현재 실시간 흐름(예: VBR 비디오 스트림)의 **즉시 필요 대역폭**을 업스트림으로 보고.
3. CMTS는 그 정보를 모아서,
   - 다음 MAP 메시지에서 해당 흐름에 적절한 **upstream 슬롯을 예약**.

즉, rtPS는

- 순수 reservation(일정한 고정 슬롯 부여)이 아니라,
- **polling을 통한 동적 reservation** 의 형태다.

---

### Polling의 지연/효율 분석 (기본)

N개 노드, 각 포링당 오버헤드 시간을 $$T_p$$,
각 노드의 평균 전송 시간(데이터) $$T_d$$라고 해 보자.

#### 이상적인 경우: 모든 노드가 항상 데이터 보유

한 라운드(1→2→…→N) 동안:

- Control 시간: $$N T_p$$
- Data 시간: $$N T_d$$

한 라운드 길이:

$$
T_{\text{round}} = N(T_p + T_d)
$$

이 때 **유효 데이터 효율**:

$$
\eta = \frac{N T_d}{N(T_p + T_d)} = \frac{T_d}{T_p + T_d}
$$

- $$T_p$$ 가 충분히 작으면 효율이 높다.
- USB처럼 host가 여러 엔드포인트를 작은 오버헤드로 관리하면 효율이 좋아진다.

#### 일부 노드만 데이터가 있는 경우

데이터 없는 노드에도 Poll을 해야 한다면,
그 노드들은 오버헤드만 증가시키는 역할을 한다.

이를 개선하기 위해:

- **동적 Poll 리스트 관리**:
  - 최근에 트래픽 있었던 노드만 자주 Poll,
  - 한참 동안 트래픽 없던 노드는 Poll 주기 늘리기
- 혹은 **Random Access + Polling 혼합**:
  - 초기 진입/소량 트래픽은 랜덤 접근,
  - 안정적으로 트래픽이 계속되는 스트림은 Polling/Reservation으로 전환.

실제 DOCSIS나 무선 시스템들이 이런 복합 설계를 많이 사용한다.

---

### 간단 Polling 시뮬레이션 코드

중앙 마스터가 N개 노드를 순환 Polling하는 간단 모델:

```python
import random

def simulate_polling(num_nodes=5, p=0.3, num_rounds=10000):
    """
    - num_nodes: 노드 수
    - p: 각 노드가 새 패킷을 생성할 확률 (라운드당)
    - num_rounds: 폴링 라운드 수
    """
    queue = [0] * num_nodes
    sent = [0] * num_nodes
    polls = 0

    for _ in range(num_rounds):
        # 각 라운드 시작 시, 노드들이 새 패킷 생성
        for i in range(num_nodes):
            if random.random() < p:
                queue[i] += 1

        # 마스터가 1→2→...→N 순으로 Poll
        for i in range(num_nodes):
            polls += 1
            if queue[i] > 0:
                # 한 번 Poll 때 최대 1개만 전송한다고 가정
                queue[i] -= 1
                sent[i] += 1

    total_sent = sum(sent)
    return {
        "sent_per_node": sent,
        "avg_queue": sum(queue) / num_nodes,
        "polls": polls,
        "efficiency": total_sent / polls if polls else 0,  # Poll당 평균 전송성공
    }

if __name__ == "__main__":
    result = simulate_polling()
    print(result)
```

- `efficiency` 는 “Poll 당 성공 전송 수”로,
  Poll 오버헤드 대비 얼마나 유효 전송이 많은지 보는 간단 지표다.
- 실제 USB/DOCSIS에서는 Poll당 여러 바이트를 보내므로 훨씬 높은 효율을 갖는다.

---

## Token Passing (토큰 전달)

### 직관적 비유

**릴레이 경주에서 바통(baton)** 을 생각하면 좋다.

- 바통을 가지고 있는 선수만 달릴 수 있고,
- 바통을 다음 사람에게 넘겨줘야만 그 사람도 달릴 수 있다.

Token Passing MAC도 동일한 발상이다.

> 네트워크 상에 **“전송 권한”을 나타내는 작은 프레임(토큰)** 이
> 논리적인 순서(링/버스)를 따라 계속 순환하고,
> 토큰을 가진 노드만 데이터를 전송할 수 있다.

이 방식의 중요한 성질:

- **충돌 없음**: 토큰은 한 번에 한 노드만 가진다.
- **결정론적(Deterministic)**: 토큰이 한 바퀴 도는 시간과 Token Holding Time 등을 알면,
  최대 접근 지연의 상한을 계산할 수 있다.

대표적인 예:

- **IEEE 802.5 Token Ring**
- **IEEE 802.4 Token Bus**
- **FDDI**, 일부 산업용 필드버스

---

### Token Ring (IEEE 802.5) 개요

#### 구조

- 물리 토폴로지: **스타형(멀티스테이션 액세스 유닛/MSAU)** 으로 보일 수 있으나,
  논리적으로는 **단방향 링**.
- MAC 방식: **Token Passing**.
- 토큰은 3bytes(또는 24비트)의 특수한 프레임 형식.

링 상에서 일어나는 일:

1. 토큰이 링을 계속 돌면서 각 스테이션을 지나간다.
2. 스테이션이 전송할 데이터가 있다면:
   - 토큰을 **“잡는다(capture)”** → 토큰 프레임을 링에서 제거.
   - 그 대신 **데이터 프레임**을 전송.
3. 목적지 스테이션은 데이터 프레임을 읽고, 복사 후 그대로 링으로 내보낸다.
4. 송신 스테이션은 자기 데이터가 한 바퀴 돌아오는 것을 보고,
   프레임을 제거하고 **새 토큰을 투입**한다.

#### 모니터 스테이션

Token Ring에는 **Monitor(Active Monitor)** 라는 특별한 스테이션이 있다.

역할:

- 토큰이 사라졌거나(Orphan),
  잘못된 데이터 프레임이 링에 남아있는 것을 감지.
- 필요한 경우 새 토큰을 만들어 넣거나,
  잘못된 프레임을 제거.
- 링 전체 지연을 일정 수준 이상 유지하기 위해
  **토큰 인서트 지연(Token insertion delay)** 을 관리.

이를 통해:

- 토큰이 **영원히 사라지지 않도록**,
- 데이터 프레임이 **영원히 링에서 떠돌지 않도록** 시스템을 안정화한다.

#### Token Holding Time

Token Ring에서는 각 스테이션이 토큰을 잡았을 때
최대 얼마나 오래 가지고 있을 수 있는지 **Token Holding Time(THT)** 를 설정한다.

예:

- THT = 10 ms라면,
  스테이션은 토큰을 잡은 뒤 최대 10ms 동안만 데이터 전송 가능.
- 10ms가 지나면, 더 보낼 데이터가 있더라도
  반드시 토큰을 다음 스테이션으로 넘겨야 한다.

이렇게 하면:

- 어느 한 스테이션도 버스를 독점할 수 없고,
- **최대 접근 지연** 에 상한을 줄 수 있다.

---

### Token Bus (IEEE 802.4) 개요

Token Bus는 물리적으로는 **버스/트리 구조**,
논리적으로는 **순서가 정해진 “가상 링”** 을 형성한다.

#### 동작 방식

1. 스테이션들은 고유 주소를 갖고 있고,
   **논리적인 순서(가상 링)** 를 형성한다.
2. 토큰은 이 순서대로 `S1 → S2 → ... → SN → S1 → ...` 이동.
3. 토큰을 받은 스테이션은
   - 자신이 전송할 데이터가 있으면,
     **정해진 시간·길이만큼 전송**하고 다시 토큰을 넘긴다.
   - 데이터가 없으면 즉시 토큰을 넘긴다.

Token Ring과 비슷하지만:

- 물리 매체가 **버스(예: 동축 케이블)** 라는 점이 다르다.
- 제조/제어 시스템 등에서,
  버스 배선이 유리한 환경에 사용되었다.

#### 동기/비동기 트래픽, 타이머 기반 제어

802.4는 **동기(synchronous) + 비동기(asynchronous)** 트래픽을 동시에 지원하기 위해
다양한 타이머를 도입한다.

- 동기 트래픽:
  - 공장 자동화/제어용 실시간 데이터.
  - **최대 지연을 강하게 제한**해야 함.
- 비동기 트래픽:
  - 파일 전송, 일반 데이터.

타이머 설정을 잘 하면:

- 동기 트래픽은 **지연 상한이 보장**되고,
- 남는 시간은 비동기 트래픽에 할당 가능.

NASA/산업 연구 등에서 802.4 Token Bus의
throughput, 지연, 노이즈에 따른 성능 저하 등을 분석한 결과,
**적절한 파라미터 선택으로 안정적인 실시간 성능**을 얻을 수 있음이 보고되었다.

---

### Token Passing의 지연/효율 분석 (기초)

N개 스테이션, 토큰 한 바퀴 도는 평균 시간 = $$T_{\text{token\_cycle}}$$ 이라고 하자.

간단 모델:

- 각 스테이션은 토큰을 잡았을 때 **최대 $$T_h$$ 만큼 전송** 가능(Token Holding Time).
- 실제 평균 전송 시간은 $$\bar{T}_d \le T_h$$.

#### 최대 접근 지연 (Worst-case access delay)

특정 스테이션 S가 토큰을 막 넘긴 직후,
다시 토큰을 받게 될 **최악의 경우**:

- 그 사이에 다른 (N−1)개 스테이션이 순서대로 토큰을 잡아,
  각자 최대 $$T_h$$ 만큼 전송한다고 가정.

그러면 최악의 접근 지연은 대략

$$
T_{\text{access\_max}} \approx (N - 1) \cdot T_h
$$

여기에 토큰 전파 지연/프레임 전파 지연 등을 고려하면 좀 더 커지지만,
핵심은 **지연 상한이 선형으로 계산 가능**하다는 점이다.

#### 채널 효율

토큰 자체와 제어 프레임이 차지하는 overhead를 $$T_{\text{ovh}}$$,
데이터 전송 시간 총합을 $$T_{\text{data}}$$ 라고 하면,

$$
\eta = \frac{T_{\text{data}}}{T_{\text{data}} + T_{\text{ovh}}}
$$

- 스테이션이 데이터가 많으면,
  - 각 토큰 잡을 때 $$T_h$$ 를 충분히 가득 채워 전송 → 효율 ↑
- 데이터가 거의 없으면,
  - 토큰이 빠르게 돌면서 대부분 **짧은 control 전송**만 수행 → 효율 ↓

그래서 Token Passing은

- **부하가 높은 실시간 시스템**에서 특히 잘 맞는다
  (스테이션들이 계속 데이터가 있고, 토큰 잡을 때마다 충분히 전송).

---

### Token Passing 타임라인 예제

4개 스테이션 A,B,C,D, 링 구조, Token Ring 스타일:

```text
시간 →
Token:    T(A)               T(B)               T(C)              T(D)      T(A) ...
A:       [DATA from A........]                         idle
B:                       [short DATA]                 idle
C:                                         [no data]
D:                                                       [DATA from D......]
```

- A는 토큰을 잡고 길게 전송 후 토큰 전달.
- B는 짧게 전송.
- C는 데이터가 없어 토큰을 거의 바로 넘김.
- D는 자신의 데이터 전송 후 토큰을 A에게 넘김.

---

### 간단 Token Ring 시뮬레이션 코드

링에 N개 노드가 있고, 토큰이 시계 방향으로 한 번씩 모든 노드를 방문하는 단순 모델:

```python
import random

def simulate_token_ring(num_nodes=5, p=0.4, token_holding_slots=4, num_rounds=10000):
    """
    - num_nodes: 링의 노드 수
    - p: 각 노드가 매 라운드마다 새 패킷을 생성할 확률
    - token_holding_slots: 토큰을 잡았을 때 최대 몇 개의 패킷까지 보낼 수 있는지 (슬롯 단위)
    - num_rounds: 토큰이 '링을 한 바퀴 돈 횟수'
    """
    queue = [0] * num_nodes
    sent = [0] * num_nodes

    for _ in range(num_rounds):
        # 각 라운드 시작 시, 새 패킷 생성
        for i in range(num_nodes):
            if random.random() < p:
                queue[i] += 1

        # 토큰이 0→1→...→N-1 순으로 한 바퀴 돌며,
        # 각 노드는 최대 token_holding_slots만큼 패킷 전송
        for i in range(num_nodes):
            slots = token_holding_slots
            while slots > 0 and queue[i] > 0:
                queue[i] -= 1
                sent[i] += 1
                slots -= 1

    total_sent = sum(sent)
    avg_queue = sum(queue) / num_nodes
    return {
        "sent_per_node": sent,
        "avg_queue": avg_queue,
        "total_sent": total_sent,
    }

if __name__ == "__main__":
    result = simulate_token_ring()
    print(result)
```

이 코드는 아주 단순화된 모델이지만,

- 토큰을 잡았을 때 최소/최대 전송 가능량(`token_holding_slots`)이
  지연·큐 길이에 어떤 영향을 주는지,
- 노드 수가 늘어날수록 평균 지연이 어떻게 증가하는지

등을 실험적으로 볼 수 있다.

---

## Reservation vs Polling vs Token Passing 비교

마지막으로, 세 가지 Controlled Access 기법을 한눈에 정리해 보자.

| 항목 | Reservation | Polling | Token Passing |
|------|------------|---------|---------------|
| 제어 방식 | Time slot/사이클을 미리 예약 | 중앙 마스터가 각 노드를 순서대로 폴링 | 토큰 프레임이 논리 링을 순환 |
| 중앙 노드 필요 | 경우에 따라 다름 (DOCSIS 등은 CMTS) | 필수 (Host/Headend) | 필요 없음(분산) 또는 특별한 monitor만 |
| 충돌 | 예약된 슬롯에서는 없음 | 폴링 구간에서는 없음 | 원칙적으로 없음 |
| 대표 사례 | DOCSIS upstream, TSN, 전용선 TDMA | USB, DOCSIS rtPS, 일부 무선 시스템 | IEEE 802.5 Token Ring, IEEE 802.4 Token Bus, FDDI |
| 지연 특성 | 예약 주기·슬롯에 따라 상한 존재 | Poll 주기·노드 수에 따라 상한 계산 가능 | 토큰 회전 시간·holding time으로 상한 계산 가능 |
| 오버헤드 | 예약용 제어 비트/프레임 | Poll 프레임 + 응답 | 토큰 프레임 + 관리 프레임 |
| 장점 | 실시간 QoS, 대역폭 보장에 유리 | 중앙에서 전력/스케줄 제어, 구현 직관적 | 완전 분산, 높은 결정론성, 공평 |
| 단점 | 트래픽 적을 때는 슬롯 낭비 | 많은 노드/빈 노드 많으면 Poll 오버헤드 큼 | 토큰 분실·링 변경 시 복구 복잡, 구현 비용 |

현대 네트워크에서는

- **랜덤 액세스 MAC만** 사용하는 경우는 드물고,
- **Controlled Access 기법과 적절히 혼합**해서 사용한다.

예를 들어:

- DOCSIS: **예약 + Polling + Best-effort** 를 조합해
  음성/영상/일반 데이터 스트림을 동시에 처리.
- TSN(Time-Sensitive Networking):
  이더넷 위에 **시간-예약(Time-aware shaper), 크레딧 기반 셰이퍼** 등을 도입해
  정밀한 시간 예약을 통해 실시간 트래픽을 지원.
- USB: 호스트 중심 Polling 구조를 유지하면서,
  전력 관리·대역폭 예약(인터벌/MaxPacketSize)·우선순위 등을 함께 다룸.

---

이렇게 12.2의 **Reservation / Polling / Token Passing** 을
12.1에서 다룬 Random Access(Aloha/CSMA 계열)와 나란히 놓고 보면,

- “충돌을 감수하고 단순하게 갈 것인가”
- “복잡한 제어를 도입해서라도 지연/공평성을 보장할 것인가”

라는 MAC 설계의 큰 축을 더 분명히 볼 수 있다.

이후 Ethernet 스위칭, Wi-Fi QoS(802.11e), TSN, DOCSIS, 산업용 필드버스 등을 공부할 때,
각 프로토콜이 언제 Random / Controlled 방식을 선택하고,
또 둘을 어떻게 혼합하는지가 자연스럽게 보이게 된다.
