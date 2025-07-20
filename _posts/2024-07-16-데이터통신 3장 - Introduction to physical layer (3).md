---
layout: post
title: 데이터 통신 3장 - Introduction to Physical Layer (3)
date: 2024-07-16 19:20:23 +0900
category: DataCommunication
---
## 3.5 Data Rate Limits
- 데이터 전송에는 빠른 전송을 위해 중요한 것이 있는데 세 가지 요소가 있다.
  - The bandwidth available (가용 대역폭)
  - The level of the signals we use (사용 가능한 신호의 level)
  - The quality of the channel (the level of noise) (채널의 품질)

### 3.5.1 Noiseless Channel: Nyquist Bit Rate
- 나이퀴스트 수식 (Nyquist Bit Rate): 채널에 노이즈가 없을 경우 이론상 최대 bit rate는 다음과 같다.
  - $$ \text{bitRate} = 2 \times \text{bandwidth} \times \log_2 L $$
    (bandwidth는 채널의 대역폭을 의미하고, L은 신호의 level 개수를 의미한다.)
- 수식에서 신호의 level 개수인 L을 증가시킬수록 비트율은 증가하지만, 신호의 level이 많아질수록 특정 진폭에 대해 그 진폭의 level을 결정하기 애매해질 수 있다. 즉, L을 증가시킬수록 신뢰도의 저하가 발생할 수 있다.

### 3.5.2 Noisy Channel: Shannon Capacity
- 섀논 수식 (Shannon Capacity): 현실에 노이즈 없는 채널은 없다. 섀논 수식은 노이즈가 존재하는 채널에서의 이론적인 최대 데이터 전송률을 다음과 같이 정의한다.
  - $$ \text{capacity} = \text{bandwidth} \times \log_2(1+\text{SNR}) $$
    (수식에서 수용량(Capacity)은 bps 단위의 채널 용량을 의미한다.)

### 3.5.3 Using Both Limits
- 실제로는 어떤 신호 level에 대해 어떤 대역폭이 필요한지 알기 위해, 두 가지 수식을 모두 사용한다.
  
**ex)** 1MHz의 대역폭을 갖는 채널이 있다. 이 채널의 SNR이 63일 때, 적절한 전송률과 신호 level의 수는 얼마인가?
  - 섀논 수식을 쓰면 노이즈 채널에서의 최대 수용량을 구할 수 있다. 
  $$ C = B \log_2(1+SNR) = 10^6 \log_2(1+63) = 6 \text{Mbps} $$ 
  하지만 이는 이론적인 최대 수용량일 뿐이므로, 더 나은 성능을 위해 조금 낮은 값을 선택한다. 여기선 4Mbps를 선택해보자. 이후 신호에 필요한 level 수를 구하기 위해 나이퀴스트 수식을 사용한다. 
  $$ 4\text{Mbps} = 2 \times 1\text{MHz} \times \log_2 L $$
  -> $$ L = 4 $$ 
  따라서 4개의 level을 가지며 4Mbps의 전송률을 가지는 신호가 적절할 것이다.
- 섀논 공식은 수용량의 한계치를 주고, 나이퀴스트 공식은 많은 신호의 레벨을 우리가 필요로 하는 숫자를 알 수 있다.

## 3.6 Performance

### 3.6.1 Bandwidth
- bandwidth in Hertz: 합성신호의 주파수의 범위 또는 채널이 넘기는 주파수의 범위를 나타낸다.
- bandwidth in BPS: 채널이나 링크에서 비트 전송의 속도를 나타낸다.

### 3.6.2 Throughput (처리율)
- Throughput: 매체의 어떤 한 지점을 데이터가 얼마나 빨리 지나가는가를 나타낸 값이다. (단위는 bps)
- $$ \text{Total Bits} = \text{Number of Frames} \times \text{Average Bits per Frame} $$
- $$ \text{Throughput} = \frac{\text{Total Bits}}{\text{Time (seconds)}} $$
- **ex)** 10Mbps의 대역폭을 갖는 네트워크가 매분 평균 10,000비트의 12,000개 프레임을 통과시킬 때 처리율은
  - $$ \text{throughput} = \frac{(12000 \times 10000)}{60} = 2 \text{Mbps} $$

### 3.6.3 Latency (Delay, 지연)
- 송신 측에서 첫 번째 비트를 보낸 시간부터 전체 메시지가 수신 측에 도착할 때까지 걸린 시간을 말한다.
- $$ \text{Latency} = \text{propagation time} + \text{transmission time} + \text{queuing time} + \text{processing delay} $$
- **propagation time (지연 시간)**: 비트가 전송지에서 도착지까지 가는데 필요한 시간.
  - $$ \text{propagation time} = \frac{\text{Distance}}{\text{Propagation speed}} $$
- **transmission time (전송 시간)**: 메시지가 완전히 전송되는데 걸리는 시간. 메시지 크기와 대역폭에 따라 시간이 달라진다.
  - $$ \text{transmission time} = \frac{\text{Messaging size}}{\text{Bandwidth}} $$
- **queuing time (대기 시간)**: 처리하기 전에 중간이나 종단 장치에서 메시지를 보관하는 시간이다. 네트워크에 부하를 한다. 네트워크에 트래픽이 무거워지면 대기 시간도 늘어난다. 라우터에서 큐처럼 하나씩 도착하고 처리한다.

### 3.6.4 Bandwidth-delay product
- 대역폭 지연 장비들은 비트 수를 정의해 링크를 채운다.

### 3.6.5 Jitter (파형 난조)
- 지연에서 쓰인다.