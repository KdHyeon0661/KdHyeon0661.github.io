---
layout: post
title: 데이터 통신 4장 - Digital Transmission (1)
date: 2024-07-17 19:20:23 +0900
category: DataCommunication
---
# 4. Digital Transmission
## 4.1 digital-to-digital conversion
- 변환제도에는 다음과 같이 있으며 이 단원에서는 Digital-to-Digital를 다룬다.
  - Digital-to-Digital Conversion : 디지털 정보를 디지털 신호로 변환하기
  - Analog-to-Digital Conversion : 아날로그 정보를 디지털 신호로 변환하기
  - Digital-to-Analog Conversion : 디지털 정보를 아날로그 신호로 변환하기
  - Analog-to-Analog Conversion : 아날로그 정보를 아날로그 신호로 변환하기
- Digital-to-Digital Conversion에는 line coding, block coding, scrambling이 있는데 line coding은 반드시 필요하며, 나머지는 상황에 따라 다르다.

### 4.1.1 Line Coding(회선 부호화) : 디지털 데이터를 디지털 신호로 변환하는 과정

- **특징**
  - **signal element vs data element(신호 요소 vs 데이터 요소)**
    - signal element : 정보를 표현할 수 있는 가장 작은 개체로서 비트라고 한다. (우리가 전달해야 하는 것)
    - data element : 시간적으로 볼 때 디지털 신호의 가장 짧은 단위신호. 요소가 데이터 요소를 전달한다. (전달자)
    - $$ r $$ : 매 신호 요소당 전송되는 데이터 요소의 개수. 신호 당 비트
    - $$ r $$을 비유하여 설명하면 $$ r = 1 $$이면, 탈 것 하나에 한 사람씩 타는 것이고, $$ r > 1 $$이면, 탈 것에 두 사람 이상이 타고 있는 것이다. 나머지 경우에는 한 사람이 차에 트레일러를 달고 운전하는 것이다. ($$ r = ½ $$)

  - **data rate vs signal rate(데이터 전송률 vs 신호 전송률)**
    - data rate : 1초 안에 보내지는 데이터 요소의 개수, 초당 비트수(bps)
    - signal rate(=pulse rate, modulation rate, baud rate) : 1초안에 보내지는 신호 요소의 개수, 초당 신호수(baud)
    - 데이터 통신에서의 목표는 신호 전송률을 낮추고, 데이터 전송률을 높이는 것이다.
    - $$ S = \frac{N}{r} $$ (S : 신호전송률, N : 데이터 전송률, r : 신호당 비트) 두 전송률은 이 관계를 가진다.
    - (S : signal rate, N : data rate, c : the case factor, r : ratio)일 때, 평균 신호 전송률은 $$ S_{ave} = c \times N \times \left( \frac{1}{r} \right) $$ baud 이다.

  - **bandwidth(대역폭)**
    - 주파수의 범위(Frequency Spectrum) = 최고 주파수 - 최저 주파수
    - 디지털 신호는 무한한 대역폭을 필요로 하지만 실제 전송매체가 가질 수 있는 대역폭은 제한적이다.(유효 대역폭)
    - 대역폭은 Signal Rate과 비례한다.
    - 디지털 데이터의 실제 대역폭(actual bandwidth)은 무한하나, 효과적인 대역폭(effective bandwidth)은 유한하다.
    - 최소대역폭 : $$ B_{min} = c \times N \times \left(\frac{1}{r}\right) $$, 최대 bps : $$ N_{max} = r \times B \times \left(\frac{1}{c}\right) $$
    - 신호 레벨이 L인 신호는 레벨당 $$ \log_2 L $$비트를 옮긴다. $$ r = \log_2 L $$

  - **Baseline Wandering(기준선 변동)**
    - Baseline : 수신된 신호 파워의 평균
    - 연속된 0이나 1의 비트값을 가진 데이터에 의해 기준선이 움직이는 현상으로, 전압이 오르락내리락한다.

  - **DC(Direct-Current) Component (직류 성분)**
    - 전압 레벨(진폭)이 일시적으로 변하지 않을 때 일어날 수 있는 0인 주파수
    - 주파수와 전압의 관계 : 신호에서 주파수란 “1초당 나타나는 반복적인 패턴의 개수”이며, 일반적으로 “단위시간(1초)당 전압의 변화횟수“이다. 즉, 패턴이 3개이면 전압이 3번 변했음을 의미한다. 주파수가 0이란 뜻은 반복되는 패턴이 0개(전압의 변화횟수가 0)이며 변화가 없는 전압을 의미한다. 변화가 없는 전압(주파수 0)은 직류가 된다. 전압(진폭)의 평균값이 0에서 커질수록 전압의 변화가 적다.
    - 진폭(Amplitude)의 평균값이 0보다 클 경우 DC성분이 있다고 표현한다.
    - 낮은 주파수를 통과시킬 수 없도록 만들어진 시스템이 있어서 DC 성분이 없도록 Encoding 해야 한다.

  - **Self-Synchronization (자기 동기화)**
    - 송신측이 보낸 데이터와 수신측이 받은 데이터에서 비트 간격이 일치하는 것을 의미한다.
    - 자기 동기화 디지털 신호는 전송되는 데이터에 시간정보를 포함한다.
    - 수신측에서 정보를 올바르게 못 받는 경우를 대비하여 처음, 중간, 끝에 Transition을 넣어 고의로 숫자를 바꿔줌.

  - **내제된 오류를 검출하고 잡음과 간섭에 강인하다.**

  - **복잡성**