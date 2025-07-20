---
layout: post
title: 데이터 통신 4장 - Digital Transmission (4)
date: 2024-07-20 19:20:23 +0900
category: DataCommunication
---
## 4.2 Analog-To-Digital Conversion
### 4.2.1 Pulse Code Modulation (PCM, 펄스 부호 변조)
- 대부분의 아날로그 신호를 디지털신호로 바꾸는 기술(digitization)을 PCM이라 부른다. PCM은 3가지 과정이 있다.

#### 과정
1. 아날로그 신호를 샘플화한다.
2. 샘플화 신호를 양자화한다.
3. 양자값을 비트스트림으로 인코딩한다.

#### Sampling
- **샘플링 주기**: $$ T_s $$, **샘플링 주파수**: $$ f_s $$ 이면, $$ f_s = \frac{1}{T_s} $$
- 샘플링 방법에는 아래의 그림과 같이 세 가지가 존재한다.
- 샘플링 기법은 보통 sample and hold로 불리며 flat-top은 회로를 만든다.
- 샘플링 과정은 pulse amplitude modulation (PAM)이 선호된다.
  
#### Sampling rate (샘플링 전송률)
- **Nyquist Sampling Theory (나이퀴스트 샘플링 이론)**
    - 원본 아날로그 신호가 가진 데이터의 최대 주파수 (frequency)의 2배로 샘플링하면, 원본 데이터로 문제 없이 복구 가능하다는 이론. 
    - 수식으로 풀어서 계산하면 이게 완벽하게 들어맞는다. $$ f_s \geq 2 \cdot f_{high} $$
    - 샘플링 전송률이 높으면 그만큼 잘 복구 할 수 있어서, 샘플링 전송률이 높은 컨버터는 비씨다.

> 이 내용을 시계를 샘플링한다고 비유할 수 있다.

#### Quantization (양자화)
- 샘플링의 결과는 신호의 최대 최소 진폭 사이의 진폭들의 펄스 급수이다. 이것은 두 극한에서 비적분적인 연속값이다.
- 과정
    1. 원래의 아날로그 신호에서 최고 전압 $$ V_{max} $$과 최저 전압 $$ V_{min} $$ 사이의 진폭을 가진다고 추측하자.
    2. 각각의 높이 $$ \Delta $$를 Level 범위로 나눈 뒤, 가까운 중간 값으로 변환하여 정규화 시킨다. 
    $$
    \Delta = \frac{V_{max} - V_{min}}{L}
    $$
    3. 정규화된 값들은 0부터 $$ L-1 $$ 사이의 숫자를 사용하여 이진 코드로 변환 (비트화)된다.
    4. 레벨이 작을수록 정규화 에러(오차)가 증가한다. 반대로 레벨이 다양할수록 정밀하게 변환이 가능하다.

#### Quantization Levels (양자화 레벨)
- 양자화할 수 있는 레벨의 개수는 샘플링에 할당되는 비트 수 $$ n $$ (분해능)에 의해 결정된다.
$$
L = 2^n
$$
- 양자화 레벨 수가 많을수록 더 세밀한 표현이 가능해진다.

#### Quantization Error (양자화 잡음)
- 양자화는 근사 과정이기 때문에 실수값이 나오면서 잡음이 생기며 비선형 연산이다.
- 양자화 잡음은 피할 수 없음.
    - 이를 작게 하기 위해 양자화 스텝 크기 $$ \Delta $$가 작아지도록 할 수 있으나, 이는 결국 양자화 레벨 수의 증가 → 양자화 비트 수 증가 → 대역폭 증가를 의미한다.

#### Uniform Versus Nonuniform Quantization (균일 대 비균일 양자화)
- 순간적 진폭 (instantaneous amplitude)에서 아날로그 신호는 균일하지 않다.
- **Uniform Quantization**
    - 양자화 레벨 사이에 동일한 간격을 갖는다. 또한 균일 양자화에는 두 가지 유형이 있다: 중간 트레드 및 중간 상승 양자화.
    - 두 양자화 모두 원점에 대해 대칭이다.
    - 중간 트레드 양자화에서는 원점이 그래프의 계단 중간에 위치하며, 중간 상승 양자화에서는 원점이 계단의 상승 부분에 위치한다.
  
- **Nonuniform Quantization**
    - 비균일 양자화에서 단계 크기는 동일하지 않다.
    - 양자화 후 입력 값과 양자화 된 값의 차이를 양자화 오류라고 한다. 비균일 양자화는 양자화 오류를 줄일 수 있다.

#### PCM Encoding (PCM 인코딩)
- 양자화된 신호를 전송 및 처리에 적합하도록 2진 펄스 형태로 부호화한다.
- 부호화에 사용되는 $$ N $$개의 비트를 사용하면 $$ 2^N $$개의 독립적인 상태를 표현할 수 있다 (양자화 분해능과 같다).
- 예시로, 3비트 부호화는 000(0), 001(1), 010(2), ..., 111(7)로 나타낸다.
- $$ N $$개의 비트 한 세트를 PCM word라고 하며, 8비트가 보통 1 PCM word이다.
- **bit rate**:
$$
\text{bit rate} = (\text{sampling rate}) \times (\text{number of bits per sample}) = f_s \times n_b
$$

#### PCM Signal Restoration (PCM 신호 복원)
- PCM 디코더가 필요하다.

#### PCM Bandwidth (PCM 대역폭)
- 라인 인코딩 신호의 $$ B_{\text{min}} = c \cdot N \cdot \frac{1}{r} $$이고, 치환이 다음과 같이 된다:
$$
B_{\text{min}} = c \cdot N \cdot \frac{1}{r} = c \cdot n_b \cdot f_s \cdot \frac{1}{r} = c \cdot n_b \cdot 2 \cdot B_{\text{Analog}} \cdot \frac{1}{r}
$$
- $$ 1/r = 1 $$ (NRZ, 바이폴라 신호) 또는 $$ c = \frac{1}{2} $$ (평균적인 상황)에서는:
$$
B_{\text{min}} = n_b \cdot B_{\text{Analog}}
$$
- 이것은 아날로그 신호의 대역폭보다 디지털 신호가 $$ n_b $$배 더 크다는 것을 의미한다.

#### Channel Maximum Data Rate (채널의 최대 데이터 비율)
- 나이퀴스트 수식:
$$
\text{bitRate} (N_{\text{max}}) = 2 \cdot B \cdot \log_2 L
$$
- 정의:
    1. 대역폭 $$ B $$에 저주파 채널이 가능하다고 추정한다.
    2. 디지털 신호는 $$ L $$ 레벨로 보내지며, 각 레벨은 신호 요소다.
    3. 저주파 필터를 통한 디지털 신호는 $$ B $$ Hz 위의 주파수는 잘린다.
    4. $$ L $$ 레벨 외의 추가 레벨은 필요하지 않으며 2 * $$ B $$ 초당 샘플은 $$ L $$ 레벨에서 양자화가 된다.
    5. 결과적으로:
    $$
    N_{\text{max}} = 2 \cdot B \cdot \log_2 L
    $$

#### Minimum Required Bandwidth (최소 필요 대역폭)
$$
B_{\text{min}} = \frac{N}{2 \cdot \log_2 L} \text{ Hz}
$$