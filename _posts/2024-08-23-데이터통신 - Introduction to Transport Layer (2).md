---
layout: post
title: 데이터 통신 - Introduction to Transport Layer (2)
date: 2024-08-23 21:20:23 +0900
category: DataCommunication
---
# Chapter 23.2 Transport Layer Protocols — Simple, Stop-and-Wait, Go-Back-N, Selective Repeat, Piggybacking

- Simple Protocol
- Stop-and-Wait Protocol
- Go-Back-N Protocol
- Selective Repeat Protocol
- Bidirectional Protocols: Piggybacking

실제 인터넷에서 그대로 쓰이는 프로토콜이라기보다는, **TCP 같은 실전 프로토콜의 내부 원리를 이해하기 위한 모델**임을 염두에 두면 좋다.

---

## Simple Protocol

### 가정과 특징

**Simple Protocol** 은 가장 단순한 “전송 계층” 모델이며, 보통 이런 가정을 가진다.

1. 채널은 **완벽(perfect)** 하다.
   - 손실 없음, 오류 없음, 중복 없음, 재정렬 없음.
2. 수신자는 항상 데이터 받을 준비가 되어 있다.
   - 흐름 제어 필요 없음.
3. 송신자는 **언제든지** 세그먼트를 보낼 수 있다.
   - ACK, 타임아웃 같은 메커니즘 필요 없음.

즉, 전송 계층은 그저 “응용 계층에서 받은 데이터를 세그먼트에 담아서 IP에게 넘기는” 역할만 한다.

### 동작 흐름

송신자:

1. 응용 계층에서 데이터 청크를 받는다.
2. 전송 계층 헤더를 붙여 세그먼트를 만든다.
3. 즉시 네트워크 계층(IP)에 넘긴다.
4. 반복.

수신자:

1. IP로부터 세그먼트를 받는다.
2. 전송 계층 헤더를 제거하고, payload 데이터를 응용 계층에 넘긴다.
3. 반복.

텍스트로 그리면:

```text
[Sender App] → [Transport: Simple] → [IP] → … → [IP] → [Transport: Simple] → [Receiver App]
```

오류/손실이 없으므로 **오류 제어·흐름 제어·순서 제어는 모두 생략**된다.

### 간단한 코드 예제 (개념)

Python 스타일의 매우 단순한 구조를 생각해보자.
(실제 소켓 API와 다르지만, 아이디어를 표현하기 위한 의사 코드)

```python
class SimpleTransportSender:
    def __init__(self, ip_layer):
        self.ip = ip_layer

    def send(self, dest_ip, dest_port, data: bytes):
        segment = self._make_segment(dest_port, data)
        # 오류/손실이 없다고 가정 → 곧바로 IP에 넘김
        self.ip.send(dest_ip, segment)

    def _make_segment(self, dest_port, data):
        # 아주 단순한 헤더: 목적지 포트만 붙인다고 가정
        header = dest_port.to_bytes(2, "big")
        return header + data


class SimpleTransportReceiver:
    def __init__(self, ip_layer, local_port, app_callback):
        self.ip = ip_layer
        self.local_port = local_port
        self.app_callback = app_callback  # 응용 계층에 데이터 넘기는 함수

    def on_receive(self, segment: bytes):
        dest_port = int.from_bytes(segment[:2], "big")
        if dest_port != self.local_port:
            return  # 다른 포트로 향한 세그먼트
        data = segment[2:]
        self.app_callback(data)
```

이 모델은 너무 이상적이기 때문에 실전에는 그대로 쓰이지 않지만,
**“전송 계층이 최소한으로 하는 일”** 을 이해하는 데 시작점이 된다.

---

## Stop-and-Wait Protocol

### 동기: Simple Protocol의 한계

현실의 채널은 다음 문제를 가진다.

- 패킷이 **손실**(drop)될 수 있다.
- 패킷이 **손상**(bit error)될 수 있다.
- 수신자가 잠시 바빠서 데이터를 받을 수 없는 경우도 있다.

따라서 아래 같은 기능이 필요해진다.

- **오류 검출** (예: 체크섬)
- **재전송** (ACK/타임아웃 기반)
- **흐름 제어** (수신자의 처리 속도에 맞추기)

가장 단순한 신뢰성 프로토콜이 바로 **Stop-and-Wait** 이다.

### 기본 아이디어

- 송신자는 **한 번에 하나의 세그먼트만 outstanding** 상태로 둔다.
- 세그먼트를 보낸 뒤, 반드시 **ACK를 받을 때까지 기다렸다가** 다음 세그먼트를 보낸다.
- ACK를 일정 시간 안에 못 받으면 **타임아웃**으로 간주하고 같은 세그먼트를 **재전송**한다.

또한, 손실/중복을 구별하기 위해 **시퀀스 번호(sequence number)** 를 사용한다.
보통 예제에서는 1비트 번호(0과 1)만으로도 충분하다.

수식적으로 보면:

- 한 RTT 동안 보낼 수 있는 데이터량을 $L$이라 하고,
- Stop-and-Wait에서는 그 중 **단 한 세그먼트의 데이터만 사용**하는 구조다.

### 시퀀스 번호와 ACK 번호

- 송신 데이터 세그먼트에는 **sequence number**(예: 0 또는 1)를 붙인다.
- 수신자는 정상 세그먼트를 받으면 **ACK(seq)** 를 돌려보낸다.

기본 규칙:

- 송신자는 `seq = 0` 세그먼트를 보낸 후, `ACK(0)` 를 기다림.
- 그 후 `seq = 1` 세그먼트를 보내고, `ACK(1)`를 기다림.
- 다시 0, 1, 0, 1… 반복.

### 타임라인 예제

간단한 텍스트 그림:

```text
Time →
Sender:   [Frame 0]------------------------>
          <-----------[ACK 0]--------------
          [Frame 1]------------------------>
          <-----------[ACK 1]--------------
          [Frame 0]------------------------> ...
Receiver:        (수신)   (ACK 전송)   (수신)   (ACK 전송)
```

손실 상황 예:

```text
Sender:   [Frame 0]----X  (손실)
          --- 타임아웃 ---
          [Frame 0]------------------------>
          <-----------[ACK 0]--------------
Receiver:        (아무 것도 못받음)
                 (재전송 Frame 0 수신) (정상)
```

수신자는 재전송된 Frame 0을 정상적으로 처리하고 ACK 0을 보낸다.

### Stop-and-Wait 효율 분석

네트워크가 좋을수록, Stop-and-Wait은 **대역폭을 잘 못 쓰게 된다.**

변수 정의:

- 전송 속도: $$R$$ [bit/s]
- 세그먼트 길이: $$L$$ [bit]
- 전송 시간: $$T_{tx} = \dfrac{L}{R}$$
- 왕복 지연(RTT): $$RTT$$ [s]

하나의 세그먼트를 보낸 뒤 ACK를 받기까지 걸리는 시간은 대략 $$RTT$$ 라고 할 수 있다.
Stop-and-Wait에서 **유효 전송 시간(실제로 데이터를 보내는 시간)** 은 $$T_{tx}$$ 이고,
그 사이에 **기다리는 시간**은 $$RTT$$ 에 들어 있다 (왕복 지연 동안 대부분 대기).

**채널 이용률(utilization)** 을

$$
U_{\text{SW}} = \frac{\text{데이터 전송에 쓰인 시간}}{\text{총 시간}}
$$

이라고 정의하면, 한 세그먼트 기준:

$$
U_{\text{SW}} = \frac{T_{tx}}{T_{tx} + RTT}
$$

예를 들어:

- 링크 속도: 100 Mbps
- 세그먼트 크기: 1500 Byte ≈ 12000 bit
- RTT: 50 ms (지구 반 바퀴 정도의 WAN)

전송 시간:

$$
T_{tx} = \frac{12000}{100 \times 10^6} = 1.2 \times 10^{-4} \text{ s} = 0.12 \text{ ms}
$$

이용률:

$$
U_{\text{SW}} = \frac{0.12\ \text{ms}}{0.12\ \text{ms} + 50\ \text{ms}}
\approx \frac{0.12}{50.12} \approx 0.0024 \approx 0.24\%
$$

즉, 링크는 **99.7%의 시간 동안 놀고 있다.**
이 비효율을 해결하기 위해 **파이프라이닝(pipelining)** 개념이 등장하고, 그것이 Go-Back-N, Selective Repeat로 이어진다.

### Stop-and-Wait 의사 코드 예제

송신자:

```python
def stop_and_wait_send(ip, dest, data_chunks):
    seq = 0  # 0 또는 1
    for chunk in data_chunks:
        while True:
            segment = make_segment(seq, chunk)
            ip.send(dest, segment)
            start_timer()
            ack = wait_for_ack_or_timeout()
            if ack is not None and ack.seq == seq:
                stop_timer()
                seq = 1 - seq  # 다음 시퀀스로 전환
                break
            else:
                # 타임아웃 or 잘못된 ACK → 재전송
                continue
```

수신자:

```python
def stop_and_wait_receive(ip, deliver_to_app):
    expected_seq = 0
    while True:
        segment = ip.receive()
        if segment.error:
            # 손상된 세그먼트: 무시(또는 마지막 ACK 재전송)
            continue
        if segment.seq == expected_seq:
            deliver_to_app(segment.data)
            send_ack(segment.seq)
            expected_seq = 1 - expected_seq
        else:
            # 중복 프레임: 다시 ACK만 보내고, 데이터는 무시
            send_ack(1 - expected_seq)
```

---

## Go-Back-N Protocol (GBN)

### 동기: Stop-and-Wait의 저효율

Stop-and-Wait에서 채널 이용률이 낮았던 이유는,

- **한 번에 하나의 세그먼트**만 네트워크 위로 나가 있기 때문이다.

만약 송신자가 **여러 세그먼트를 “연속해서” 보내고**, ACK가 오는 동안도 계속 새 데이터를 보낼 수 있다면,
RTT 동안 채널을 훨씬 더 잘 활용할 수 있다. 이것이 **슬라이딩 윈도우(sliding window)** 와 **파이프라이닝** 의 기본 아이디어다.

### Go-Back-N의 기본 개념

Go-Back-N(GBN)은 다음과 같이 동작한다.

1. 송신자는 크기 $$N$$ 인 **송신 윈도우**를 가진다.
   - 한 번에 최대 $$N$$ 개의 세그먼트를 outstanding 상태로 둘 수 있다.
2. 각 세그먼트에는 시퀀스 번호를 붙인다.
   - 예: 0, 1, 2, …, $$k$$ (모듈러 연산으로 순환).
3. 수신자는 **가장 최근까지 올바르게 순서대로 받은 시퀀스 번호를 ACK로 회신**한다. (누적 ACK, cumulative ACK)
4. 중간에 어떤 세그먼트가 손실되면,
   - 그 이후로 오는 세그먼트는 수신자가 **폐기**(버림)하고,
   - 송신자는 타임아웃 시점에 **손실된 세그먼트부터 윈도우 끝까지를 한 번에 재전송**한다.
   → “Go back N(이전으로 돌아가서 N개를 다시)”라는 이름의 기원.

### Go-Back-N 윈도우 동작

송신자 측 상태 변수:

- `base`: 아직 ACK를 받지 못한 가장 오래된 시퀀스 번호
- `nextseqnum`: 새로 보낼 다음 세그먼트의 시퀀스 번호
- 윈도우 조건:

$$
\text{base} \le \text{nextseqnum} < \text{base} + N
$$

즉, `nextseqnum` 가 `base + N` 보다 작을 때까지는 새 세그먼트를 계속 보낼 수 있다.

수신자는 **다음에 기대하는 시퀀스 번호 `expected_seq`** 를 하나만 기억한다.

- 세그먼트 `seq` 가 들어왔을 때:
  - `seq == expected_seq` → 데이터 상위 전달, ACK(seq) 전송, `expected_seq++`
  - `seq != expected_seq` → 데이터 폐기, **직전까지 받은 마지막 순서 번호에 대한 ACK** 를 재전송

### 예제: 손실 시나리오

윈도우 크기 $$N = 4$$, 시퀀스 번호 0~7(모듈러스 8) 사용.

송신하려는 프레임: 0, 1, 2, 3, 4, 5 …

1. 송신자는 0~3까지 연속으로 전송
2. 수신자는 0,1은 잘 받고, 2 번 프레임이 손실된 상황 예:

```text
Time →
Sender:   [0]----> [1]----> [2]----X [3]---->
Receiver:     0        1       (미수신)   (3 수신 but out-of-order)

Receiver 행동:
- 0 수신 → 상위 전달, ACK 0
- 1 수신 → 상위 전달, ACK 1
- 2 미수신
- 3 도착: expected_seq=2인데 seq=3이므로 데이터 폐기, ACK 1 재전송

Sender:
- ACK 0,1 수신 → base = 2
- ACK 1을 반복해서 받지만, 아직 2에 대한 ACK는 없음
- 타임아웃 발생 → base부터 (2,3,4,5...) 다시 전송
```

즉, 프레임 2 하나만 잃었는데도, **그 이후로 보낸 3,4,5 등도 전부 다시 보내야** 한다.
이것이 GBN의 비효율 요소다.

### 비교

Stop-and-Wait에서 이용률은:

$$
U_{\text{SW}} = \frac{T_{tx}}{T_{tx} + RTT}
$$

GBN에서 윈도우 크기를 $$N$$ 라고 하면, 이상적으로는 **RTT 동안 최대 $$N$$ 개** 의 프레임을 pipeline할 수 있다.
대략적인 이용률 근사(채널 오류 없다고 가정):

$$
U_{\text{GBN}} \approx \min\left(1,\ \frac{N \cdot T_{tx}}{T_{tx} + RTT}\right)
$$

또는

$$
U_{\text{GBN}} \approx \min\left(1,\ N \cdot U_{\text{SW}}\right)
$$

즉, $$N$$ 배까지 개선될 수 있지만, 결국 $$U \le 1$$ 이므로 채널 용량을 넘어설 수는 없다.

실전 설계에서는 대략

$$
N \gtrsim \frac{RTT}{T_{tx}} + 1
$$

이면 링크를 꽉 채울 수 있다고 본다 (이 이상 늘려도 더 빨라지지 않는다).

### GBN 송신자/수신자 의사 코드

송신자:

```python
class GoBackNSender:
    def __init__(self, ip, dest, window_size, timeout_interval):
        self.ip = ip
        self.dest = dest
        self.N = window_size
        self.base = 0
        self.nextseqnum = 0
        self.timeout_interval = timeout_interval
        self.timer_running = False
        self.buffer = {}  # seqnum -> data

    def send(self, data_chunks):
        for chunk in data_chunks:
            while self.nextseqnum >= self.base + self.N:
                # 윈도우가 꽉 찼으므로 ACK를 기다려야 한다.
                self._wait_for_event()
            self._send_one(chunk)

        # 모든 데이터 전송 후, 모든 ACK 수신까지 대기
        while self.base != self.nextseqnum:
            self._wait_for_event()

    def _send_one(self, chunk):
        seq = self.nextseqnum
        segment = make_segment(seq, chunk)
        self.ip.send(self.dest, segment)
        self.buffer[seq] = segment
        if self.base == self.nextseqnum:
            self._start_timer()
        self.nextseqnum += 1

    def on_ack(self, ack_seq):
        # 누적 ACK: base ~ ack_seq 까지 모두 확인된 것으로 본다
        if ack_seq >= self.base:
            self.base = ack_seq + 1
            if self.base == self.nextseqnum:
                self._stop_timer()
            else:
                self._start_timer()

    def on_timeout(self):
        # base부터 nextseqnum-1 까지 다시 전송
        self._start_timer()
        for seq in range(self.base, self.nextseqnum):
            segment = self.buffer[seq]
            self.ip.send(self.dest, segment)

    # 타이머 관련 함수는 환경에 따라 구현
```

수신자:

```python
class GoBackNReceiver:
    def __init__(self, ip, deliver_to_app):
        self.ip = ip
        self.expected_seq = 0
        self.deliver_to_app = deliver_to_app
        self.last_acked = -1

    def on_receive(self, segment):
        if segment.error:
            # 손상된 세그먼트: 마지막 유효 seq에 대한 ACK 재전송
            self._send_ack(self.last_acked)
            return

        if segment.seq == self.expected_seq:
            self.deliver_to_app(segment.data)
            self.last_acked = self.expected_seq
            self._send_ack(self.last_acked)
            self.expected_seq += 1
        else:
            # out-of-order: 데이터 폐기, 마지막 ACK 재전송
            self._send_ack(self.last_acked)

    def _send_ack(self, seq):
        ack_seg = make_ack_segment(seq)
        self.ip.send_back(ack_seg)
```

---

## Selective Repeat Protocol (SR)

### 동기: GBN의 비효율

GBN에서는 **하나의 프레임 손실** 때문에

- 그 뒤의 모든 프레임을 버리고,
- 송신자는 윈도우 내 모든 프레임을 다시 보내야 했다.

대역폭이 비싸고 손실율이 낮은 링크에서는 큰 낭비가 된다.
이를 개선하려는 것이 **Selective Repeat (선택적 반복)** 이다.

### 기본 개념

Selective Repeat의 핵심:

1. 수신자는 **순서가 어긋난(out-of-order) 프레임도 내부 버퍼에 저장**한다.
2. 송신자는 **ACK 되지 않은 프레임만 선택적으로 재전송**한다.
3. 각 프레임에 대해 **개별 타이머**를 둘 수 있다 (혹은 타임아웃 관리 전략을 세분화).

즉,

- GBN: “하나 틀리면, 그 뒤로는 다 다시 보내라.”
- SR: “틀린 것(혹은 빠진 것)만 골라서 다시 보내라.”

### SR 송신/수신 윈도우

SR에서는 송신자뿐 아니라 **수신자도 윈도우를 가진다.**

- 송신자 윈도우: 아직 ACK 받지 못한 프레임 집합
- 수신자 윈도우: “수용 가능한 시퀀스 번호 범위”

수신자는 수신 윈도우 안에 있는 프레임이면

- 순서와 무관하게 **버퍼에 저장**
- 해당 프레임에 대한 **ACK를 개별적으로 전송**

그리고 `base` 부터 연속해서 도착한 프레임들을 상위에 인도한 뒤,
수신 윈도우를 앞으로 슬라이드한다.

### 시퀀스 번호 공간 제약 (윈도우 ≤ 공간/2)

Selective Repeat에서는 **시퀀스 번호가 모듈러 구조**이기 때문에,
윈도우를 너무 크게 잡으면 **오래된 프레임과 새로운 프레임을 구분하지 못하는 모호성(ambiguity)** 이 생긴다.

일반적인 규칙:

- 시퀀스 번호 공간 크기: $$2^k$$
- 윈도우 크기: $$W$$
- 조건:

$$
W \le \frac{2^k}{2} = 2^{k-1}
$$

이유(예시):

- 시퀀스 번호 0~7 (3비트, $$2^3 = 8$$), 윈도우 크기 $$W = 5$$ 라고 하자.
- 어떤 상황에서 **예전 윈도우에 있던 프레임 #1** 이 늦게 도착했을 때,
  수신자는 그것이 “예전 윈도우의 1”인지, “새 윈도우의 1”인지 구분할 수 없게 된다.
- 윈도우 크기를 공간의 절반 이하로 제한하면,
  항상 “현재 윈도우와 한 바퀴 전/후 윈도우”가 겹치지 않도록 설계할 수 있다.

직관적으로,

> **전체 번호 공간을 반으로 쪼개서, 한 번에 하나의 반쪽만 “유효 범위”로 쓰자**

는 의미이다.

### Selective Repeat 동작 예제

시퀀스 번호 0~7, 윈도우 크기 $$W = 4$$ (규칙 만족).
송신하려는 프레임: 0,1,2,3,4,5,6…

1. 송신자는 0~3까지 전송
2. 수신자는 0,1,3을 수신하고, 2는 손실된 상황:

```text
Receiver side buffer:

seq: 0 [OK, 상위 인도]
seq: 1 [OK, 상위 인도]
seq: 2 [X, 비어있음]   ← 손실
seq: 3 [OK, 버퍼에 저장 (out-of-order)]
```

수신자 행동:

- 0 도착 → 상위 인도, ACK 0
- 1 도착 → 상위 인도, ACK 1
- 3 도착 → 아직 2를 안 받았으므로, 상위에는 인도하지 않고 **버퍼에 저장**, ACK 3

송신자:

- ACK 0,1,3을 받는다.
- 2에 대한 ACK가 없으므로, 프레임 2만 재전송
- 2가 다시 도착하면:

수신자:

- 2 수신 → 버퍼에 저장되어 있던 3과 함께 상위에 연속해서 인도
  (2, 그 다음 3 인도)
- ACK 2 (또는 누적 ACK 전략을 쓸 수도 있음)

이렇게 **프레임 2만 선택적으로 재전송**하기 때문에,
GBN보다 대역폭 효율이 높다.

### SR 의사 코드 (개념)

송신자:

```python
class SelectiveRepeatSender:
    def __init__(self, ip, dest, window_size, timeout_interval):
        self.ip = ip
        self.dest = dest
        self.N = window_size
        self.base = 0
        self.nextseqnum = 0
        self.timeout_interval = timeout_interval
        self.buffer = {}    # seq -> data
        self.timers = {}    # seq -> timer

    def send(self, data_chunks):
        for chunk in data_chunks:
            while self._window_full():
                self._wait_for_event()  # ACK나 타임아웃 대기
            self._send_one(chunk)

        while not self._all_acked():
            self._wait_for_event()

    def _window_full(self):
        return (self.nextseqnum - self.base) >= self.N

    def _send_one(self, chunk):
        seq = self.nextseqnum
        self.buffer[seq] = chunk
        self._transmit(seq)
        self.nextseqnum += 1

    def _transmit(self, seq):
        segment = make_segment(seq, self.buffer[seq])
        self.ip.send(self.dest, segment)
        self._start_timer(seq)

    def on_ack(self, ack_seq):
        if ack_seq in self.buffer:
            # 해당 프레임만 개별적으로 완료 처리
            self._stop_timer(ack_seq)
            del self.buffer[ack_seq]
            # base 슬라이드
            while self.base not in self.buffer and self.base < self.nextseqnum:
                self.base += 1

    def on_timeout(self, seq):
        # 해당 seq만 재전송
        self._transmit(seq)
```

수신자:

```python
class SelectiveRepeatReceiver:
    def __init__(self, ip, deliver_to_app, window_size, seq_space):
        self.ip = ip
        self.deliver_to_app = deliver_to_app
        self.N = window_size
        self.seq_space = seq_space
        self.base = 0
        self.buffer = {}  # seq -> data

    def in_window(self, seq):
        # base <= seq < base + N 인지 모듈러 공간에서 판단
        if self.base <= seq < self.base + self.N:
            return True
        # 모듈러 wrap-around 고려 필요 (여기서는 단순화)
        return False

    def on_receive(self, segment):
        seq = segment.seq
        if segment.error or not self.in_window(seq):
            # 유효하지 않은 seq → 무시 (또는 적절한 ACK)
            return

        # 버퍼에 저장
        if seq not in self.buffer:
            self.buffer[seq] = segment.data
            self._send_ack(seq)

        # base부터 연속으로 도착한 데이터 상위 인도
        while self.base in self.buffer:
            self.deliver_to_app(self.buffer[self.base])
            del self.buffer[self.base]
            self.base = (self.base + 1) % self.seq_space
```

SR은 **수신자 쪽 버퍼, 타이머, 윈도우 관리가 GBN보다 훨씬 복잡**하지만,
패킷 손실이 드문 환경에서 **대역폭 효율을 크게 높여준다.**

---

## Bidirectional Protocols: Piggybacking

지금까지는 대부분 **단방향(unidirectional)** 전송을 가정했다.

- A → B로만 데이터가 전달되고, B → A는 ACK만 보내는 구조.

하지만 전송 계층은 일반적으로 **전이중(full-duplex)** 이다.

- A ↔ B 모두 서로 데이터와 ACK를 주고받는다.

이때 단순히 생각하면:

- A는 B에게 데이터 + ACK를 보냄
- B도 A에게 데이터 + ACK를 보냄

하지만 매 데이터에 대해 별도의 ACK 세그먼트를 보내면 **헤더와 대역폭 낭비**가 크다.
그래서 등장하는 개념이 **Piggybacking(피기백킹)** 이다.

### Piggybacking 개념

Piggybacking:

- 수신 측이 “바로 다음에 보낼 데이터 세그먼트”가 있다면,
  **그 세그먼트의 헤더에 ACK 정보를 함께 실어 보내는 것**이다.
- 즉, **데이터와 ACK를 한 패킷으로 합치는 것**.

예:

```text
A → B: [seq=10, ack=20, data="Hello"]   # A가 B에게 데이터 보내면서, B의 19까지 수신했음을 ACK
B → A: [seq=21, ack=11, data="Hi"]      # B도 A에게 데이터 보내면서, A의 10까지 수신했음을 ACK
```

이렇게 하면 별도의 ACK-only 세그먼트를 보내야 할 경우가 줄어들어 **오버헤드를 줄일 수 있다.**

### Piggybacking이 없을 때와 있을 때 비교

#### Piggybacking 없음

양방향 트래픽에서, 단순한 구조:

- A → B: 데이터 1개 + ACK 1개
- B → A: 데이터 1개 + ACK 1개

각각의 데이터에 대해 별도의 ACK 프레임이 날아간다.

#### Piggybacking 있음

동일한 양의 데이터 트래픽이라도, 다음처럼 합쳐서 보낼 수 있다.

- A → B: 데이터(헤더에 B의 이전 데이터에 대한 ACK 포함)
- B → A: 데이터(헤더에 A의 이전 데이터에 대한 ACK 포함)

ACK-only 세그먼트는 **“보낼 데이터가 없을 때 일정 시간 기다렸다가 그래도 없으면 그때 전송”** 하는 식으로 제한할 수 있다.

### Piggybacking이 중요한 이유

1. **헤더 오버헤드 감소**
   - 예: TCP/IP 헤더만 해도 수십 바이트인데, 이를 ACK-only 세그먼트에 사용하면 낭비가 크다.
2. **네트워크 부담 감소**
   - 패킷 수를 줄이면, 라우터 큐, 처리 부하, 혼잡 가능성을 줄일 수 있다.
3. **전력/자원 절감**
   - 무선 환경에서는 패킷 수 자체가 전력 소모와 직결된다.

그렇다고 모든 ACK를 무조건 Piggybacking만으로 처리할 수 있는 것은 아니다.
**너무 오래 기다리면 송신 측 재전송 타임아웃이 길어지고, 지연이 커지기 때문**이다.

따라서 현실적인 설계는 다음과 같은 정책을 사용한다.

- 일정 시간 이내에 보낼 데이터가 생기면 → 그 데이터에 ACK를 실어서 보냄
- 그 시간 안에 데이터가 생기지 않으면 → ACK-only 세그먼트 전송

TCP도 비슷한 개념을 사용한다.

- 데이터 세그먼트에 항상 ACK 번호를 포함
- 데이터가 없는데도 ACK가 필요하면, “pure ACK” 패킷을 보냄

### Piggybacking 있는 양방향 Stop-and-Wait 예제

단방향 Stop-and-Wait에서 양방향으로 확장하되, Piggybacking을 이용하는 예제.

각 방향에 대해 별도의 시퀀스 번호와 ACK 번호를 유지한다.

- A → B: `seq_A`, `ack_B`
- B → A: `seq_B`, `ack_A`

각 노드는 “보낼 데이터가 있을 때만” 세그먼트를 만들어 보내는데, 그 안에 **상대방에 대한 ACK** 도 같이 포함한다.

의사 코드(양방향 노드 하나 기준):

```python
class BiDirectionalNode:
    def __init__(self, ip):
        self.ip = ip
        self.nextseq_out = 0     # 내가 보내는 데이터의 시퀀스 번호
        self.expected_in = 0     # 상대가 보내는 데이터에 대해 기대하는 시퀀스
        self.last_ack_out = -1   # 내가 보낸 데이터에 대해 상대가 ACK해준 마지막 seq
        self.pending_ack = False # 아직 보내지 않은 ACK가 있는지
        self.ack_to_send = None
        self.timer = None

    def send_data(self, dest, payload):
        # 보낼 데이터가 생겼을 때 호출
        seg = self._build_segment(payload)
        self.ip.send(dest, seg)
        self._start_timer()
        self.pending_data = seg  # 간단히 하나만 관리한다고 가정

    def _build_segment(self, payload):
        header = {
            "seq": self.nextseq_out,
            "ack": self.ack_to_send if self.pending_ack else None
        }
        self.pending_ack = False
        return Segment(header, payload)

    def on_receive(self, segment):
        # 데이터 처리
        if segment.seq == self.expected_in:
            # 올바른 순서
            deliver_to_app(segment.data)
            self.expected_in += 1
            self.ack_to_send = segment.seq
            self.pending_ack = True
        else:
            # out-of-order 혹은 중복 → 여기서는 단순화해서 무시
            pass

        # ACK 처리
        if segment.ack is not None and segment.ack == self.nextseq_out:
            # 내 데이터가 인정됨
            self._stop_timer()
            self.nextseq_out += 1

    def tick(self):
        # 주기적으로 호출되는 타이머 루프
        if self._timer_expired():
            # 재전송
            self.ip.send(self.pending_data)
            self._start_timer()

    def flush_ack_if_needed(self, dest):
        # 일정 시간 대기 후에도 보낼 데이터가 없으면 ACK-only 세그먼트
        if self.pending_ack:
            seg = Segment({"seq": None, "ack": self.ack_to_send}, payload=b"")
            self.ip.send(dest, seg)
            self.pending_ack = False
```

여기서 핵심은:

- 데이터가 있을 때는 **항상 ACK를 같이 실어 보내도록** 하고,
- 데이터가 없는데 ACK를 보내야 하면 **ACK-only 세그먼트**를 보낸다는 점이다.

Piggybacking은 Stop-and-Wait, GBN, SR 등 **어떤 ARQ 프로토콜과도 결합 가능**하다.
단지 ACK 정보를 **별도의 프레임이 아닌, 반대 방향 데이터 프레임에 얹어서 보내는 패턴**일 뿐이다.

---

## 요약

23.2 절에서 다룬 프로토콜들은 실제 인터넷에서 그대로 쓰이는 것이 아니라,
**전송 계층의 신뢰성·순서·흐름/혼잡 제어를 이해하기 위한 계단식 모델**이다.

- **Simple Protocol**
  - 오류·손실이 전혀 없는 이상적인 채널에서, 전송 계층이 하는 최소한의 일만 모델링.
- **Stop-and-Wait Protocol**
  - 시퀀스 번호, ACK, 타임아웃을 통해 신뢰성을 제공하지만,
  - 한 번에 한 세그먼트만 outstanding 상태로 둘 수 있어 이용률이 낮다.
- **Go-Back-N Protocol**
  - 슬라이딩 윈도우를 이용해 여러 세그먼트를 파이프라인 전송.
  - 손실 발생 시 손실된 세그먼트 이후 모든 세그먼트를 재전송 → 구조는 단순하지만 대역폭 낭비 가능.
- **Selective Repeat Protocol**
  - 수신자가 out-of-order 프레임을 버퍼에 저장하고,
  - 송신자는 ACK되지 않은 프레임만 선택적으로 재전송.
  - 시퀀스 번호 공간/윈도우 크기 제약(윈도우 ≤ 공간/2)이 필요하며, 구현이 더 복잡하지만 효율이 높다.
- **Bidirectional Protocols & Piggybacking**
  - 양방향 트래픽에서 ACK-only 세그먼트를 줄이기 위해,
  - 데이터 세그먼트의 헤더에 ACK 정보를 **피기백(piggyback)** 하는 기법.
  - 오버헤드와 네트워크 부하를 줄이고, 전력/자원을 절약하는 데 기여한다.

이 모델들을 잘 이해해두면, 이후 등장하는 **TCP의 슬라이딩 윈도우, 순서/신뢰성 보장, ACK 구조, RTT 기반 타임아웃, 혼잡 제어** 등을 “이론적으로” 해석하는 데 큰 도움이 된다.
~~~markdown
:
````
