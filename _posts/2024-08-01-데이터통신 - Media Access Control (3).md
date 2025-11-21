---
layout: post
title: 데이터 통신 - Media Access Control (3)
date: 2024-08-01 16:20:23 +0900
category: DataCommunication
---
# Channelization — FDMA, TDMA, CDMA 완전 정리

12.1에서 **Random Access(ALOHA, CSMA)**, 12.2에서 **Controlled Access(Reservation, Polling, Token Passing)** 을 봤다.
이 둘은 “여러 노드가 하나의 **공유 매체**를 어떻게 나눠 쓰는가”를 MAC 차원에서 다루는 방식이다.

**12.3 Channelization** 은 관점이 조금 다르다:

> 물리 계층 자원(주파수, 시간, 코드)을 **정교하게 쪼개서**
> 여러 사용자에게 동시에 제공하는 **다중 접속(Multiple Access)** 기법.

대표적인 세 가지 축:

- **FDMA (Frequency Division Multiple Access)** — 주파수로 나누기
- **TDMA (Time Division Multiple Access)** — 시간으로 나누기
- **CDMA (Code Division Multiple Access)** — 코드(확산 시퀀스)로 나누기

실제 셀룰러/위성/무선 시스템은 이들을 **조합**해서 쓰는 경우가 많다:

- 1G AMPS: FDMA 중심
- 2G GSM: FDMA(200 kHz carrier) + TDMA(8 time slots)
- 3G UMTS / cdma2000: W-CDMA/DS-CDMA 중심
- 4G LTE / 5G NR: OFDMA(주파수×시간 그리드) — FDMA/TDMA의 진화형

여기서는 **기본 FDMA/TDMA/CDMA** 개념과 수식, 예제, 간단한 시뮬레이션 코드를 중심으로 정리한다.

---

## Channelization 개념

### 1) Multiplexing vs Multiple Access

헷갈리기 쉬운 두 용어:

- **Multiplexing**: 여러 **신호/스트림**을 하나의 물리 채널로 합치는 것
  (TDM, FDM, CDM 등 — 물리 계층 관점)
- **Multiple Access**: 여러 **사용자/단말**이 하나의 매체를 공유하는 것
  (FDMA, TDMA, CDMA 등 — MAC/물리 계층 경계)

실제로는 **같은 원리**를 지칭하지만, 문맥에 따라 이름이 달라진다고 이해하면 된다.

우리가 보는 FDMA/TDMA/CDMA는:

- “하나의 주파수 대역/무선 매체를 여러 사용자에게 나눠주는”
  **channel access method** 이자 **multiple access scheme** 이다.

---

### 2) 세 가지 기본 축

자원을 나누는 축은 크게 세 가지:

1. **주파수(Frequency)**: FDMA
2. **시간(Time)**: TDMA
3. **코드(Code, Spreading Sequence)**: CDMA

개념적으로는 다음과 같이 그림으로 그릴 수 있다.

```text
FDMA:  주파수 축을 분할
        ^
        |   |■■ Ch1 ■■|■■ Ch2 ■■|■■ Ch3 ■■|
        +----------------------------------------> f

TDMA:  시간 축을 분할 (슬롯)
        ^
        |  User1  User2  User3  User1  User2  ...
        +----------------------------------------> t

CDMA:  같은 f, 같은 t 을 쓰되 각 사용자마다 다른 코드 사용
        y(t) = Σ_k  b_k(t) * c_k(t)
        (합쳐져 전송, 수신단에서 코드 c_k 로 상관해서 분리)
```

현대 시스템(OFDMA, SC-FDMA, NOMA 등)은 이 기본 축들을 더 복잡하게 조합한다.

---

## FDMA (Frequency Division Multiple Access)

### 1) 개념

**FDMA** 는 전체 주파수 대역을 여러 **좁은 채널**로 나누고,
각 사용자에게 **서로 다른 주파수 슬롯**을 할당하는 방식이다.

- 각 사용자는 "자기" 대역에서만 송수신한다.
- 시간적으로는 **연속**해서 전송 가능 (회선 교환 느낌).
- 인접 채널 간 간섭을 막기 위해 **가드 대역(guard band)** 필요.

주파수 축 예시:

```text
전체 시스템 대역: B_total

|<-- guard -->|<- B_ch ->|<- guard ->|<- B_ch ->|<- guard ->|<- B_ch ->|...

f:  ──────────────┬───────────┬───────────┬───────────┬─────────────►
                  Ch1        Ch2        Ch3        Ch4
```

간단히 쓰면, 총 대역폭 $$B_{\text{tot}}$$,
각 채널 폭 $$B_c$$, 가드밴드 폭 $$B_g$$ 일 때,
이론적인 최대 채널 수 $$N$$ 는 대략

$$
N \approx \left\lfloor \frac{B_{\text{tot}} + B_g}{B_c + B_g} \right\rfloor
$$

(정확한 값은 양 끝 가드 유무, duplex 구조에 따라 달라진다.)

---

### 2) 실제 예: 1G AMPS 셀룰러

고전 **AMPS(Advanced Mobile Phone System)** 는 1세대 아날로그 셀룰러로, **FDMA** 를 사용했다.

전형적인 설정(북미):

- 총 할당 대역폭: 약 50 MHz
  - 25 MHz: 다운링크(기지국→단말)
  - 25 MHz: 업링크(단말→기지국)
- 한 음성 채널 용 주파수 폭: 30 kHz
- 전체 채널 수: 약 832개

대략 계산:

$$
N \approx \frac{25\ \text{MHz}}{30\ \text{kHz}} \approx \frac{25 \times 10^6}{30 \times 10^3} \approx 833
$$

가드 밴드, 제어 채널 등을 고려하면 **832개 음성 채널**이 나온다.

특징:

- 한 사용자는 통화 동안 **항상 같은 30 kHz 대역** 사용.
- 회선 교환에 매우 잘 어울리는 구조.

---

### 3) 장점 / 단점

장점:

- 개념이 단순, 구현이 직관적.
- 각 사용자 채널이 **좁고 고정**되어 있어,
  수신기 설계(필터, 동조)가 비교적 간단.
- 사용자 간 간섭을 **주로 주파수 분리**로 해결.

단점:

- 가드밴드 때문에 **대역 효율이 떨어짐**.
- 각 채널별로 독립 RF 필터가 필요 → **하드웨어 비용 증가**.
- 대역을 고정으로 쪼개서 할당하므로,
  - 어떤 채널은 계속 바쁘고,
  - 어떤 채널은 비어 있어도
  다른 사용자가 그 대역을 쉽게 활용하기 어렵다.

---

### 4) FDMA 간단 수치 예제

예제 상황:

- 총 대역폭: $$B_{\text{tot}} = 12\ \text{MHz}$$
- 채널 폭: $$B_c = 200\ \text{kHz}$$
- 가드밴드: $$B_g = 25\ \text{kHz}$$

가능 채널 수:

$$
N \approx \left\lfloor \frac{12\,\text{MHz} + 25\,\text{kHz}}{200\,\text{kHz} + 25\,\text{kHz}} \right\rfloor
= \left\lfloor \frac{12{,}025\,\text{kHz}}{225\,\text{kHz}} \right\rfloor \approx \left\lfloor 53.4 \right\rfloor = 53
$$

즉, 이 설정에서는 **최대 53개 사용자**가 동시에 할당 가능.

만약 가드밴드를 0으로 줄일 수 있다면(이상적):

$$
N_{\text{ideal}} \approx \left\lfloor \frac{12\,\text{MHz}}{200\,\text{kHz}} \right\rfloor = 60
$$

- 현실에서는 필터의 비이상성 때문에 가드밴드가 필수 → 60→53으로 감소.

---

### 5) FDMA 간단 계산 코드 예제

```python
def fdma_num_channels(B_total_hz, B_ch_hz, B_guard_hz):
    """
    간단 FDMA 채널 수 계산 (양 끝에 guard band 1개 있다고 가정)
    """
    return int((B_total_hz + B_guard_hz) // (B_ch_hz + B_guard_hz))

if __name__ == "__main__":
    B_total = 12e6      # 12 MHz
    B_ch = 200e3        # 200 kHz
    B_guard = 25e3      # 25 kHz

    N = fdma_num_channels(B_total, B_ch, B_guard)
    print("가능한 채널 수:", N)
```

이 코드를 수정해가면서:

- 가드밴드 폭에 따라 채널 수가 어떻게 변하는지,
- 채널 폭을 증가/감소시키면 어떤 trade-off가 있는지

직접 확인해 볼 수 있다.

---

## TDMA (Time Division Multiple Access)

### 1) 개념

**TDMA** 는 하나의 주파수 채널을 시간 축으로 나누어,
서로 다른 사용자가 **서로 다른 시간 슬롯(time slot)** 에서 번갈아 송수신하는 방법이다.

- 모든 사용자가 **같은 주파수**를 사용한다.
- 각 사용자는 자신에게 할당된 **슬롯**에서만 송신.
- 나머지 시간에는 송신하지 않고 기다린다.

시간 축 예시:

```text
하나의 TDMA 프레임:

| Slot1 | Slot2 | Slot3 | Slot4 | Slot5 | Slot6 | Slot7 | Slot8 |

사용자 할당:
  Slot1 → User A
  Slot2 → User B
  Slot3 → User C
  ...
  Slot8 → User H
```

한 사용자의 **유효 사용률(duty cycle)**:

$$
\text{duty} = \frac{\text{사용자 슬롯 수}}{\text{프레임 내 전체 슬롯 수}}
$$

예: 한 프레임에 8 슬롯, 사용자당 1슬롯이면 duty = 1/8.

---

### 2) 대표 예: GSM 2G 셀룰러

GSM(2G)는 **FDMA + TDMA** 를 조합한다.

- 주파수 측면:
  - 다운링크 935–960 MHz, 업링크 890–915 MHz
  - 200 kHz 간격의 캐리어(FDMA)
- 시간 측면:
  - 각 캐리어마다 TDMA 프레임 구조
  - 한 프레임은 8개의 **타임슬롯**으로 구성.

일부 중요한 숫자:

- **프레임 길이**: 약 4.615 ms
- **타임슬롯 길이**: 약 577 µs (정확히는 546.5 µs 유용 구간 + 가드)
- 한 타임슬롯(“버스트”)에:
  - tail bit, data bit, training sequence, guard space 포함.

즉, 한 200 kHz 채널에서 **동시에 8명의 사용자**를 수용할 수 있다.
(실제 채널 구성은 음성/제어 채널 조합이라 좀 더 복잡하다.)

---

### 3) TDMA 프레임/슬롯 구조 예시

개념적으로 GSM downlink 한 캐리어에 대해:

```text
주파수: f_c (200 kHz 대역)

시간:
   ┌───────────────────────────────────────────────────┐
   │           TDMA Frame (4.615 ms)                  │
   ├─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┤
   │Slot0│Slot1│Slot2│Slot3│Slot4│Slot5│Slot6│Slot7│
   └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
    577µs each (≈ 156.25 symbol periods)
```

각 슬롯 내부(“Normal Burst”)는 다시:

- `tail bits` (동기용)
- `data bits` (payload)
- `training sequence` (채널 추정)
- `guard bits`

로 구성된다.

---

### 4) 사용자 체감 비트율 계산 예시

예제:

- GSM 기본 모드에서 한 음성 사용자가
  - 한 프레임(8슬롯) 중 1슬롯만 사용한다고 하자.
- RF 계층에서 1 슬롯당 유효 비트 수를 $$N_{\text{bit/slot}}$$ 라 하자 (부호화/채널 코딩 포함).
- 프레임 주기 $$T_f = 4.615\ \text{ms}$$.

사용자당 gross 비트율 $$R_u$$ 는

$$
R_u = \frac{N_{\text{bit/slot}}}{T_f}
$$

GSM에서 한 timeslot의 물리 비트율은 약 270.833 kbps이지만,
채널 코딩/인터리빙 등을 거치면 사용자 음성 코덱 비트율은 13 kbps(GSM-FR 등) 정도가 된다.

중요 포인트:

- **무선 구간의 raw 비트율**과,
- **사용자에게 보이는 코덱 비트율**은 다르다.
- TDMA는 **슬롯 단위 자원**을 유연하게 묶어
  다양한 서비스(음성, 제어, 데이터)에 할당할 수 있다.

---

### 5) TDMA의 장단점

장점:

- FDMA보다 **필터/튜너 설계 부담이 적음**
  (같은 캐리어 내에서 시간만 나누면 되기 때문에).
- 여러 사용자가 동적/유연하게 **슬롯 수를 더 많이/적게** 받을 수 있음.
- 데이터 통신에 적절한 **버스트형 자원 할당** 구조.

단점:

- 각 사용자 단말은 중간에 송신/수신을 멈추었다 다시 시작 →
  정확한 **시간 동기**가 필요.
- 실시간 부하가 적은 구간에서는 일부 슬롯이 **비워질 수 있음**.
- 다중 경로/페이딩 환경에서는 **슬롯간 간섭/동기/채널추정**이 중요 이슈.

---

### 6) TDMA 자원 할당 예제

상황:

- 한 TDMA 프레임에 슬롯 8개.
- 음성 사용자 A,B,C,D 4명 (각 1슬롯씩 필요).
- 데이터 사용자 E는 높은 데이터율을 원해서 2슬롯 필요.
- 남은 두 슬롯은 제어용(SI, BCCH 등).

하나의 프레임에 대한 할당 예:

| Slot | 사용자/용도 |
|------|-------------|
| 0    | A (음성)    |
| 1    | B (음성)    |
| 2    | C (음성)    |
| 3    | D (음성)    |
| 4    | E (데이터)  |
| 5    | E (데이터)  |
| 6    | BCCH(제어)  |
| 7    | 빈 슬롯/예약 |

사용자 E는 A~D에 비해 **2배 슬롯**을 받으므로,
하나의 캐리어 내에서 **상대적으로 두 배의 데이터율**을 갖는다.

---

### 7) 간단 TDMA 시뮬레이션 코드

파이썬으로 아주 단순 화된 TDMA 스케줄러를 만들어 볼 수 있다.

```python
import random
from collections import defaultdict

def simulate_tdma(num_slots=8, num_frames=10000):
    """
    - num_slots: 프레임당 슬롯 수
    - num_frames: 시뮬레이션할 프레임 수
    예시:
      A,B,C,D: 음성 사용자 (슬롯 1개 필요)
      E: 데이터 사용자 (슬롯 2개 필요)
    """
    users = ["A", "B", "C", "D", "E"]
    sent = defaultdict(int)

    # 프레임별 고정 할당 구조
    # 실제 시스템에서는 스케줄링이 동적일 수 있음
    slot_allocation = {
        0: "A",
        1: "B",
        2: "C",
        3: "D",
        4: "E",
        5: "E",
        6: "CTRL",  # 제어용
        7: None     # 비어있음
    }

    for _ in range(num_frames):
        for s in range(num_slots):
            user = slot_allocation.get(s)
            if user in users:
                # 간단히: 슬롯 하나가 '패킷 하나'를 실어 나른다고 가정
                sent[user] += 1

    return sent

if __name__ == "__main__":
    result = simulate_tdma()
    print(result)
    # E가 A, B, C, D보다 약 2배 패킷을 보냈음을 확인
```

위 결과에서:

- `A,B,C,D` 는 유사한 수의 패킷,
- `E` 는 그 약 2배 패킷을 전송하게 된다.

이를 통해 **슬롯 수가 곧 평균 데이터율 비율**에 대응한다는 것을 직관적으로 확인할 수 있다.

---

## CDMA (Code Division Multiple Access)

### 1) 직관적 개념

**CDMA** 는 주파수, 시간 모두 동시에 공유하면서,
각 사용자를 **서로 다른 코드(확산 시퀀스)** 로 구분하는 방식이다.

- 전체 대역폭 $$W$$ 를 **모든 사용자가 동시에 사용**한다.
- 각 사용자는 자신의 데이터 심볼을 **고속의 코드 시퀀스** 로 곱해
  - 스펙트럼을 넓게 “확산(spread)”시킨다.
- 수신 측에서는 원하는 사용자 코드로 **상관(correlation)** 을 계산해,
  - 그 사용자 신호만 복원한다.

핵심 아이디어:

> “주파수나 시간을 나눠 쓰기보다,
>  서로 다른 코드로 같은 자원을 쓰고,
>  수신단에서 신호 처리를 통해 분리하자.”

---

### 2) Direct Sequence Spread Spectrum (DS-CDMA) 원리

가장 널리 알려진 CDMA는 **DS-CDMA (직접 직렬 확산)** 방식이다.

기본 모델(1 사용자):

1. 비트율 $$R_b$$ 의 데이터 비트열 \\(b_k \in \{-1,+1\}\\) (BPSK라 가정).
2. 칩(chip)율 $$R_c$$의 코드 시퀀스 \\(c_n \in \{-1,+1\}\\), 길이 $$L$$ chips/bit.
3. 한 데이터 비트를 코드 시퀀스로 “늘린다”:

   - 데이터 비트 1개 기간 안에 **L개의 칩**을 전송.
   - 칩율 $$R_c = L \cdot R_b$$.

4. 결과 신호는 원래 비트열보다 **L배 넓은 대역폭**을 사용.

**확산 이득(Processing Gain)**:

$$
G_p = \frac{R_c}{R_b} = L = \frac{W}{R_b}
$$

- $$W$$: 확산된 신호의 대역폭(대략 칩율과 비례).
- $$R_b$$: 데이터 비트율.

$$G_p$$ 가 클수록:

- 같은 파워에서 **에너지/비트가 넓게 퍼져**,
  협대역 간섭에 대해 강해진다.
- 여러 사용자 신호가 동시에 존재해도
  상관 처리로 분리가 쉬워진다 (단, 완전한 정교한 설계는 아님).

---

### 3) 다중 사용자 CDMA 수식

동기 DS-CDMA의 간단한 베이스밴드 모델:

- 사용자 수: $$K$$
- 사용자 k의 데이터 비트열: \\(b_k[i] \in \{-1, +1\}\\)
- 사용자 k의 칩 시퀀스: \\(c_k[n] \in \{-1, +1\}\\), 길이 L

한 비트 구간 내에서 송신 신호:

$$
s(t) = \sum_{k=1}^{K} \sqrt{P_k} \sum_{n=0}^{L-1} b_k \, c_k[n] \, p(t - nT_c)
$$

- $$P_k$$: 사용자 k의 송신 파워
- $$T_c$$: 칩 간격
- $$p(t)$$: 기본 펄스

수신단에서 사용자 m 을 복원하려면:

1. 수신 신호를 사용자 m의 코드 \\(c_m[n]\\) 으로 곱해 상관.
2. 칩들을 적분/합산하여 비트 결정.

이 때, 코드들 사이의 **상호 상관(cross-correlation)** 이 작을수록
다른 사용자 신호는 **잡음 수준**으로 보인다.

---

### 4) 작은 예제: 2 사용자 CDMA

정수 계열로 아주 단순한 예를 만들어 보자.

- 사용자 1 코드: \\(c_1 = [+1, +1, +1, +1]\\)
- 사용자 2 코드: \\(c_2 = [+1, -1, +1, -1]\\)

코드 길이 L = 4.

각 비트에 대해:

- 사용자 1 데이터: \\(b_1 \in \{-1,+1\}\\)
- 사용자 2 데이터: \\(b_2 \in \{-1,+1\}\\)

칩 시퀀스:

- 사용자 1: \\(b_1 c_1 = [b_1, b_1, b_1, b_1]\\)
- 사용자 2: \\(b_2 c_2 = [b_2, -b_2, b_2, -b_2]\\)

합산 신호(칩별):

$$
x[n] = b_1 c_1[n] + b_2 c_2[n], \quad n=0,1,2,3
$$

예를 들어:

- \\(b_1 = +1\\), \\(b_2 = +1\\):

  - 사용자 1: [1, 1, 1, 1]
  - 사용자 2: [1, -1, 1, -1]
  - 합: [2, 0, 2, 0]

수신단에서 사용자 1 복원:

- 사용자 1 코드와 곱:
  \\(x[n] c_1[n] = [2, 0, 2, 0]\\).
- 합산: \\(\sum_n x[n] c_1[n] = 4\\) → 양수이므로 비트 +1.

사용자 2 복원:

- \\(x[n] c_2[n] = [2, 0, 2, 0]\\) (우연히 같음).
- 합산: 4 → +1.

이 단순 예에서 코드들이 완전히 직교는 아니지만,
어느 정도 분리가 가능하다는 것을 보여준다.

실제 CDMA 시스템에서는 **Walsh-Hadamard 코드**, **Gold 코드**, **Kasami 코드** 등
좋은 상관 특성을 가진 코드 집합을 사용한다.

---

### 5) CDMA의 장단점

장점:

- 주파수/시간 자원을 세밀히 나누지 않고도,
  많은 사용자가 **동시에 동일 대역**을 사용할 수 있다.
- 적절한 코드와 처리 이득 덕분에:
  - 협대역 간섭에 강하다.
  - 셀 간 **소프트 핸드오버** 구현이 용이하다(2G IS-95, 3G W-CDMA에서 큰 장점).
- 셀 경계에서 **주파수 재사용 계수 1** 을 쓰기 유리
  (모든 셀에서 동일 주파수 대역을 사용해도, 코드와 파워 제어로 제한 가능).

단점:

- **파워 제어 문제(near-far problem)**:
  - 기지국에서 볼 때, 가까운 단말이 너무 세게 보내면
    먼 단말 신호가 묻혀버리므로,
  - 엄격한 **폐루프/개루프 파워 제어**가 필수.
- 코드 간 상호 간섭(MAI: Multi-Access Interference)으로 인해,
  - 사용자 수가 많아질수록 **이론적 capacity 계산이 복잡**해진다.
- 수신기 구현(예: RAKE, multiuser detection) 복잡도 ↑.

---

### 6) CDMA의 처리 이득(Processing Gain) 예제

예제:

- 데이터 비트율 $$R_b = 12.2\ \text{kbps}$$ (3G W-CDMA 음성 코덱 중 하나)
- 칩율 $$R_c = 3.84\ \text{Mcps}$$ (W-CDMA 기본 칩율)

처리 이득:

$$
G_p = \frac{R_c}{R_b} \approx \frac{3.84 \times 10^6}{12.2 \times 10^3} \approx 315
$$

dB 스케일:

$$
G_p(\text{dB}) = 10 \log_{10}(G_p) \approx 10\log_{10}(315) \approx 24.98\ \text{dB}
$$

이론적으로 이 정도 처리 이득은

- 동일 잡음 하에서 **약 25dB의 “유효 SNR 향상”** 과 비슷한 효과를 낼 수 있다
  (정확한 수치는 변조/부호화/수신기 구조에 따라 달라짐).

---

### 7) 간단 CDMA 합성/복원 코드 예제

아주 단순한 **DS-CDMA 2 사용자** 예제를 파이썬으로 흉내내 보자.

```python
import numpy as np

def spread(bits, code):
    """
    bits: 1D 배열 (±1)
    code: 1D 배열 (±1), 길이 L
    반환: 칩열
    """
    L = len(code)
    chips = np.repeat(bits, L) * np.tile(code, len(bits))
    return chips

def despread(chips, code):
    """
    chips: 수신 칩열
    code: 사용자 코드
    반환: 복원된 비트(부호)
    """
    L = len(code)
    num_bits = len(chips) // L
    chips_reshaped = chips.reshape(num_bits, L)
    # 상관
    corr = chips_reshaped.dot(code)
    # 부호 결정
    return np.sign(corr)

if __name__ == "__main__":
    # 코드 정의
    c1 = np.array([1, 1, 1, 1])
    c2 = np.array([1, -1, 1, -1])

    # 데이터 비트 (±1)
    b1 = np.array([1, -1, 1])
    b2 = np.array([1, 1, -1])

    # 확산
    chips1 = spread(b1, c1)
    chips2 = spread(b2, c2)

    # 동시 전송 → 합성 신호
    y = chips1 + chips2

    # 수신 측 복원
    b1_hat = despread(y, c1)
    b2_hat = despread(y, c2)

    print("원래 b1:", b1)
    print("복원 b1:", b1_hat)
    print("원래 b2:", b2)
    print("복원 b2:", b2_hat)
```

이 간단한 예제에서는 코드 길이가 짧고, 잡음/채널이 이상적이므로
두 사용자 모두 오류 없이 복원될 가능성이 높다.

코드를 변형해:

- 사용자 수 증가,
- 코드 상호 상관이 나쁜 경우,
- 잡음/페이딩 추가

등을 실험해 보면 CDMA의 장단점을 더 직관적으로 체감할 수 있다.

---

### 8) CDMA의 실제 적용 (2G/3G, 이후)

간단한 역사:

- **IS-95 (cdmaOne)**: 2G CDMA 셀룰러, 1.25 MHz 대역폭, 칩율 1.2288 Mcps.
- **cdma2000**: IS-95 발전형, 1x/3x 구조, 다양한 데이터율 지원.
- **W-CDMA (UMTS)**: 3G Europe/Global 표준, 5 MHz carrier, 3.84 Mcps 칩율.

하지만:

- 4G LTE/5G NR로 넘어오면서,
  셀룰러의 주요 다중 접속은 **OFDMA/SC-FDMA** 로 이동했다.

그럼에도 CDMA/Spread Spectrum 개념은

- GNSS(GPS 등),
- 군용 통신,
- 저전력/저데이터율 IoT

등에서 여전히 중요한 역할을 한다.

---

## FDMA vs TDMA vs CDMA 비교

마지막으로 세 기법을 한 표에 정리해 보자.

| 항목 | FDMA | TDMA | CDMA |
|------|------|------|------|
| 자원 축 | 주파수 | 시간 | 코드(확산 시퀀스) |
| 사용자 분리 방식 | 서로 다른 주파수 대역 할당 | 같은 주파수, 서로 다른 시간 슬롯 | 같은 주파수, 같은 시간, 서로 다른 코드 |
| 대표 시스템 | 1G AMPS, 일부 위성/아날로그 무선 | GSM, 위성 TDMA, 일부 무선 백홀 | IS-95, cdma2000, W-CDMA, 일부 군용/위성 |
| 충돌 여부 | 채널 간 간섭은 있지만 충돌 개념은 없음 | 슬롯 스케줄링에 의해 충돌 없음 | 코드 간 MAI, near-far로 인한 간섭 존재 |
| 장점 | 개념/하드웨어 단순(특히 아날로그) | 동적 자원 할당, 디지털과 궁합 좋음 | 소프트 핸드오버, 협대역 간섭에 강함, 재사용 계수 1에 유리 |
| 단점 | 가드밴드로 인한 대역 낭비, 채널 고정 | 정확한 시간 동기 필요, 빈 슬롯 발생 가능 | 파워 제어, 코드 설계, 수신기 복잡도, MAI |
| 현대적 위치 | 단독 사용은 적고, OFDMA의 기본 아이디어로 흡수 | TDMA 틀은 OFDMA/프레임 구조에 녹아 있음 | 3G 이후 셀룰러 주류에서는 OFDMA에 자리를 넘겨줌 |

실제 시스템 설계에서는:

- **FDMA로 대역을 큰 블록으로 나누고**,
- 각 블록 안에서 **TDMA 프레임 구조**를 사용하며,
- 일부 제어/특수 채널은 **CDMA/확산**을 쓰거나,
- 4G/5G처럼 **OFDMA**로 시간–주파수 그리드를 동시에 쪼개고,
  거기에 QoS/스케줄링 알고리즘을 입힌다.

---

## 정리

12.3 Channelization에서 본 것:

1. **FDMA**
   - 주파수 축 분할. 사용자마다 주파수 슬롯 고정.
   - 아날로그/초기 셀룰러/위성에서 널리 사용.
   - 가드밴드와 RF 필터 비용이 주요 이슈.

2. **TDMA**
   - 시간 축 분할. 사용자마다 정해진 타임슬롯.
   - GSM/위성/무선 백홀 등에서 핵심 기술.
   - 시간 동기와 프레임 구조 설계가 중요.

3. **CDMA**
   - 코드 축 분할. 모든 사용자가 동시에 같은 대역 사용.
   - Spread Spectrum, Processing Gain, Power Control, RAKE 등이 키워드.
   - 2G/3G 셀룰러 시대의 핵심, 4G/5G에서 OFDMA에 자리를 넘겼지만
     여전히 군용, GNSS, 특수 무선에서 중요.

12.1 Random Access, 12.2 Controlled Access 와 함께 보면:

- **랜덤 접근(MAC 레벨)** vs
- **중앙 제어/토큰 기반 MAC** vs
- **물리 자원(channelization) 설계**

라는 세 축이 합쳐져 오늘날의 복잡한 무선 시스템이 만들어진다.

실제로는

- FDMA/TDMA/CDMA/OFDMA 등으로 **물리 자원을 어떻게 쪼갤지** 결정하고,
- 그 위에서 ALOHA/CSMA/예약/폴링/토큰 등으로 **다수 사용자 접근을 어떻게 제어할지**를 결정하는 식으로,
- 두 계층이 **겹쳐서 설계**된다고 보는 것이 좋다.
