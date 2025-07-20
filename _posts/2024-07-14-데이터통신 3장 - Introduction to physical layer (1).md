---
layout: post
title: 데이터 통신 3장 - Introduction to Physical Layer (1)
date: 2024-07-14 19:20:23 +0900
category: DataCommunication
---
# 3. Introduction to Physical Layer

## 3.1 Data and Signals

![DC physical_layer](/../assets/img/network/2024-07-14-DC_physical_layer.jpg)

### 3.1.1 Analog and Digital Data
- **Analog Data**: 연속적인 데이터
- **Digital Data**: 이산적인 데이터

### 3.1.2 Analog and Digital Signals

![DC signal](/../assets/img/network/2024-07-14-DC_signal.jpg)

- **Signal (신호)**: 이산적이거나 연속적일 수 있다.
- **Analog Signal**: 주기 동안 수많은 파장의 세기로 구성되어 있다.
- **Digital Signal**: 정의된 숫자만 사용할 수 있다.

### 3.1.3 Periodic and Nonperiodic Signals
- **Periodic Signal**: 패턴이나 주기가 있는 신호
- **Nonperiodic Signal**: 패턴이나 주기가 없는 신호

## 3.2 Periodic Analog Signals

### 3.2.1 Sine Wave
- 사인파는 **peak amplitude**(진폭), **frequency**(주파수), **phase**(위상)으로 이루어져 있습니다.
  - **Peak Amplitude**: 신호에서 가장 강한 세기의 절대값을 의미합니다.

  ![DC Amplitude](/../assets/img/network/2024-07-14-DC_amplitude.jpg)

  - **Period**: 시간의 양으로, 보통 1 사이클을 의미합니다.
  - **Frequency**: 1초에 주기(period)의 양을 나타낸다. 주기와 주파수는 서로 역관계가 있습니다:
    - $$ f = \frac{1}{T} $$, $$ T = \frac{1}{f} $$
    - 단위는 초당 Hertz (Hz)입니다.

  ![DC Frequency](/../assets/img/network/2024-07-14-DC_frequency.jpg)

### Units of Period and Frequency

![DC Units of Period](/../assets/img/network/2024-07-14-DC_Units of Period.jpg)

- **Frequency**: 시간에 따른 변화율을 의미한다. 짧은 시간의 변화는 높은 빈도를 의미하며, 긴 시간의 변화는 낮은 빈도를 의미합니다.
- 신호에 변화가 없다면 주파수는 0이며, 순간적으로 계속 변한다면 주파수는 무한대입니다.

### 3.2.2 Phase (위상)
- **Phase**: 반복되는 파형의 한 주기에서 첫 시작점의 각도 또는 특정 순간의 위치를 나타낸다. 각도나 라디안으로 표시된다:
  - $$ 360^\circ = 2\pi $$

![DC Phase](/../assets/img/network/2024-07-14-DC_phase.jpg)

- **예시**: 사인파가 0에서 다시 0으로 돌아오는데 $$ \frac{1}{6} $$ 사이클이 걸린다면, 위상은:
  - $$ \frac{1}{6} \times 360^\circ = 60^\circ = \frac{\pi}{3} $$ 라디안이다.

### 3.2.3 Wavelength (파장)
- **Wavelength**: 공간에서 파동이 주기적으로 반복되는 구간의 길이.
  - $$ \text{Wavelength} = \text{Propagation Speed} \times \text{Period} $$
  - $$ \text{Wavelength} = \frac{\text{Propagation Speed}}{\text{Frequency}} = c \cdot f $$
  - 전파 속도는 매체와 신호의 주파수에 따라 다르다.
    - 예: 진공 상태에서 빛의 전파 속도는 $$ 3 \times 10^8 $$ m/s로, 케이블이나 공기보다 빠르다.
- **예시**: 적외선 주파수 $$ 4 \times 10^4 $$와 공기에서의 전파 속도일 때:
  - $$ \lambda = \frac{3 \times 10^8}{4 \times 10^4} = 0.75 \times 10^{-6} $$ m = 0.75 m.

### 3.2.4 Time and Frequency Domains
- **Time Domain**: 신호 진폭이 시간에 따라 어떻게 변하는지 보여준다.
- **Frequency Domain**: 진폭과 주파수 간의 관계를 보여준다.
  - 주파수 영역에서는 주파수와 최대 진폭을 즉시 파악할 수 있다.
  - 시간 영역에서 완벽한 사인파는 주파수 영역에서 하나의 스파이크로 나타낼 수 있다.

![DC Time and Frequency Domains](/../assets/img/network/2024-07-14-DC_Time and Frequency Domains.jpg)
![DC Time and Frequency Domains2](/../assets/img/network/2024-07-14-DC_Time and Frequency Domains2.jpg)

### 3.2.5 Composite Signals (합성 신호)

![DC Composite](/../assets/img/network/2024-07-14-DC_Composite_new.jpg)

![DC Composite2](/../assets/img/network/2024-07-14-DC_Composite2.jpg)

- 합성 신호는 여러 개의 사인파로 만들어진다.
  - 단일 주파수 사인파는 데이터 통신에서는 필요하지 않으며, 여러 사인파로 구성된 합성 신호가 필요하다.
  - **Fourier Analysis**: 합성 신호를 주파수, 진폭, 위상이 다른 사인파로 분해한다.
  - 주기적인 합성 신호는 이산 주파수 시리즈로 분해되며, 비주기적인 신호는 연속 주파수 시리즈로 분해된다.
  - **Fundamental Frequency (기본 주파수, 조파, first harmonic)**: 합성 신호에서 기준이 되는 주파수.
  - $$ 3f $$는 $$ f $$의 3배 주파수를 가진다.

![DC nonperiodic signal](/../assets/img/network/2024-07-14-DC_nonperiodic signal.jpg)

- 다음과 같이 비주기적인 신호는 신호 분해시 연속적인 숫자들로 표현된다. 
- 비주기적인 신호의 수직선 높이는 주파수 진폭에 해당한다.

### 3.2.6 Bandwidth (대역폭)

![DC Bandwidth](/../assets/img/network/2024-07-14-DC_bandwidth.jpg)

- **Bandwidth**: 특정 기능을 수행하는 주파수의 범위. 단위는 Hz이다.
- 합성 신호의 대역폭은 최대 주파수와 최소 주파수 차이이다.

- **예시**: 주파수가 100, 300, 500, 700, 900 Hz인 신호의 대역폭:
  - Bandwidth = $$ 900 - 100 = 800 $$ Hz.