---
layout: post
title: 데이터 통신 5장 - Analog Transmission (1)
date: 2024-07-22 19:20:23 +0900
category: DataCommunication
---
# 5. Analog Transmission

## 5.1 Digital-to-Analog Conversion
### 종류

### 5.1.1 Aspects of Digital-to-Analog Conversion
- **데이터 요소 vs 신호 요소**
    - 챕터 4에서 데이터 요소는 정보의 가장 작은 단위가 비트로 치환된 것이고, 신호 요소는 신호의 유닛의 가장 작은 단위가 고정된 것이다. 우리는 신호 요소가 자연에서 아날로그 전송에서 비트가 약간 다르다.
  
- **데이터 속도(data rate) vs 신호 속도(signal rate)**
    - $$ S = \frac{N}{r} \quad \text{baud} $$  
    여기서 N은 데이터 속도이며, S는 신호 속도, r은 한 개의 신호 요소에서 옮겨진 데이터 요소의 수이다.
    - 아날로그 전송에서 $$ r = \log_2 L $$이며, L은 다른 신호 요소의 수이다.
    - **Bitrate**는 시간당 처리하는 비트의 수이고, **Baud**는 처리하는 신호의 수이다.
  
- **대역폭(Bandwidth)**
    - 디지털 데이터의 아날로그 전송 대역폭은 반송파 사이에서 더하는 FSK를 제외하고 신호 속도가 필요하다.

- **반송파(Carrier signal)**
    - 아날로그 전송에서 전송 장치는 높은 주파수의 신호를 정보 신호에서 생성한다. 이 기저 신호는 반송파라 하며, 수신 장치는 반송파의 주파수에 맞춘다. 디지털 신호는 반송파로 바뀌어 여러 특징을 가지며 변조(shift keying)이라 불린다.
    - 통신에서 정보의 전달을 위해 입력 신호를 변조한 전자기파(일반적으로 사인파)를 의미한다.

### 5.1.2 Amplitude Shift Keying
- 반송파의 진폭을 달리하여 변조하는 방식
- **Binary ASK (BASK)** or **on-off keying (OOK)**

    - 반송파는 단순한 사인파를 보인다.
    - $$ B = (1 + d) \times S $$  
      여기서 B는 대역폭, S는 신호 속도이며, 진폭 변수 d는 0과 1 사이의 값이다. 대역폭은 최소 $$ S $$, 최대 $$ 2S $$의 값을 가진다. 대역폭 중간에 $$ f_c $$인 반송파가 있다. 대역폭에서 점유해 변조된다.
  
    - **구현**
        - 디지털 데이터가 unipolar NRZ이면, 1V나 0V가 NRZ에 도달한다. 반송파는 발진기에서 오며, NRZ의 진폭이 1이면 반송파의 진폭은 1을 가지며, 진폭이 0이라면 0을 반환한다.

- **Multilevel ASK**
    - 여기서 2개의 진폭 레벨을 가졌으나, 4, 8, 16 그 이상이 가능하며, 그 때마다 $$ r = 2, 3, 4, \dots $$를 가진다. 이것은 순수한 ASK가 아니며, QAM과 함께 구현된 것이다.

### 5.1.3 Frequency Shift Keying
- 반송파의 주파수를 달리하여 변조하는 방식
- **Binary FSK (BFSK)**
    - 두 개의 반송파 $$ f_1, f_2 $$를 준비하고 각각 0, 1을 넣어 보내는 방식이다. 반송파는 크며, 그 사이 값은 작다.
  
    - 바이너리 FSK는 $$ B = (1 + d) \times S + 2 $$의 대역폭이며, $$ 2f $$는 두 반송파 사이의 거리다. 적어도 $$ S $$는 되어야 변조, 복조 작업에 좋다.
    - BFSK는 **noncoherent**, **coherent**가 있다. noncoherent BFSK는 두 개의 ASK 변조에 두 개의 반송파가 쓰인 BFSK 신호이며, 이산적인 위상과 하나의 신호 요소가 시작과 끝에 있다. coherent BFSK는 하나의 **voltage-controlled oscillator (VCO)**가 쓰였으며, 연속적 위상이 두 신호 경계에 있다.
  
    - 유니폴라 NRZ 신호는 발진기의 입력값이다. NRZ의 진폭이 0이면 발진기는 균일한 주파수를 내며, 진폭이 양수면, 주파수가 늘어난다.

- **Multilevel FSK (MFSK)**
    - MFSK는 FSK에서 일반적이지 않다. 3비트를 보내려면 8개의 주파수가 필요하다. 변조와 복조를 위해 적어도 $$ f $$가 필요하다.  
    $$ B = (1 + d) \times S + (L - 1) \times 2f \quad \Rightarrow \quad B = L \times S $$

### 5.1.4 Phase Shift Keying
- 기준 신호(반송파)의 위상을 변경 또는 변조 함으로써 데이터를 전송하는 디지털 변조 방식
- 일반적으로 **PSK**가 쓰인다.
- **Binary PSK (BPSK)**
    - 디지털 신호(0,1)에 따라, 위상이 180˚(0,π) 다른 두 정현파로, 위상편이변조하는 방식. 정보 신호에 따라 전송 신호의 극성(위상)이 결정됨. 두 신호가 180˚ 위상 차이를 갖는 대척 신호($$ s_1 = -s_2 $$)가 됨.

    - BPSK는 BASK랑 대역폭이 같으나 BFSK보다 작다. 두 반송파로 대역폭이 낭비되지 않는다.
    - ASK와 구조가 비슷하며, 쉬운데 신호 요소가 위상이 180˚ 차이나는 대척 신호이기 때문이다.
    - 폴라 NRZ 신호에 반송 주파수를 곱한다. 1 bit면 0˚, 0 bit면 180˚이다.

- **Quadrature PSK (QPSK)**
    - 위상 변화를 π/2 (90˚)씩 변화를 주어, 4개 종류의 디지털 심볼로 전송하는 4진 PSK 방식(위상은 45˚, -45˚, 135˚, -135˚)

- **Constellation Diagram (성상도)**
    - 전송 가능한 신호 집합을 각각에 상응하는 신호점(메시지점, 디지털 심볼)들의 집합으로 2차원상에서 점으로 나타내 보인 것.
    - x축은 같은 위상에 있음(반송파, 위상직교, in-phase carrier)을 의미하며, y축은 진폭값(직교 위상 성분, quatrature component, quadrature carrier)을 의미한다.
    - ASK, BPSK, QPSK를 그리면:

### 5.1.5 Quadrature Amplitude Modulation
- **QAM**: ASK + PSK
    - a는 유니폴라 NRZ를 쓴 4-QAM으로 ASK에 쓰인다.  
    - b는 폴라 NRZ로 QPSK에 쓰인다.  
    - c는 두 반송파에 변조시킨 양수 레벨의 신호를 쓴 것이다.  
    - d는 16-QAM으로 8 레벨의 신호로 4개의 양수, 4개의 음수로 이뤄져 있다.
    - QAM 전송에 필요한 최소 대역폭은 ASK와 PSK 전송에 필요한 것과 같다. 그래서 QAM은 PSK, ASK의 장점을 가지고 있다.