---
layout: post
title: 데이터 통신 3장 - Introduction to Physical Layer (2)
date: 2024-07-15 19:20:23 +0900
category: DataCommunication
---
### 3.3.1 Bit Rate

![DC Bit Rate](/../assets/img/network/2024-07-15-DC_Bit Rate.JPG)

- **bit rate**: 초당 전송한 bit의 개수 = bits per second (bps)  
  ex) 100페이지 책을 보내는데 bit rate는?  
  - 평균적으로 80개의 문자로 이뤄진 문장 24개가 있다. 그리고 문자를 전송하는데 8비트가 필요하다. 그러니 100 x 24 x 80 x 8 = 1,536,000 bps = 1.536 Mbps

  ex) HDTV의 비트레이트는?  
  - HDTV는 16:9 비율을 가졌으며, 이것은 1920 x 1080픽셀을 가졌다. 스크린은 초당 30번 교체되며, 색을 표현하는데 24비트가 필요하다. 그래서 1920 x 1080 x 30 x 24 = 1,492,992,000 = 1.5 Gbps.

### 3.3.2 Bit Length
- **bit length**: 전송 중간에 한 비트가 점유하는 거리이다.  
  - $$ \text{bit length} = \text{propagation speed} \times \text{bit duration} $$

### 3.3.3 Digital Signal as a Composite Analog Signal

![DC DSCAS]((/../assets/img/network/2024-07-15-DC_DSCAS.JPG)

- 디지털 신호는 여러 주파수로 구성된 복합 신호로 볼 수 있으며, 이를 주파수 영역으로 분석할 수 있다. 디지털 신호의 주파수는 비연속적이지만, 모든 구성 요소의 주파수를 합치면 원래의 디지털 신호가 됩니다. 이를 설명하기 위해 푸리에 변환(Fourier analysis)이 사용되며, 이는 주파수 대역의 넓이를 측정하여 디지털 신호의 대역폭을 알 수 있도록 도와줍니다. 또한, 주기적인 신호와 비주기적인 신호가 어떻게 다르게 나타나는지 위의 이미지에서 보여주고 있습니다.

### 3.3.4 Transmission of Digital Signals
- 디지털 신호는 무한인 대역폭에 있는 합성 아날로그 신호이다.  
- **baseband transmission**: 디지털 신호를 아날로그 신호로 바꾸지 않고 채널로 디지털 신호를 보내는 것이다.

![DC baseband transmission](/../assets/img/network/2024-07-15-DC_baseband transmission.JPG)

  - baseband transmission은 low-pass channel (0부터 시작하는 대역폭을 갖는 채널)이 필요로 하지 않는다.

![DC low-pass channel](/../assets/img/network/2024-07-15-DC_low-pass channel.JPG)

  - **case 1**: low-pass channel with wide bandwidth
    - 비주기적인 디지털 신호의 정확한 형태를 유지하고자 한다면, 0에서 무한대에 이르는 주파수 전체 스펙트럼을 전송해야 합니다. 이는 컴퓨터 내부 (예: CPU와 메모리 사이)에서는 가능할 수 있지만, 두 장치 사이에서는 불가능합니다. 그러나 일부 주파수의 진폭은 가장자리에서 매우 작아 무시할 수 있습니다. 이는 동축 케이블이나 광섬유 케이블과 같이 매우 넓은 대역폭을 가진 매체가 있는 경우, 두 개의 스테이션이 매우 정확한 디지털 신호로 통신할 수 있음을 의미합니다. 출력 신호가 원래 신호와 정확히 동일한 복제본은 아니지만, 데이터를 수신된 신호로부터 여전히 유추할 수 있습니다.

![DC case1](/../assets/img/network/2024-07-15-DC_case1.JPG)

  - **case 2**: low-pass channel with limited bandwidth  
    - 이 경우 대역폭이 한정되어 있기 때문에, 디지털 신호를 통해 모양이 비슷한 아날로그 신호를 추정 (approximate)한다. 디지털 신호 대역폭의 대략적인 근사치는 비트율 N에 대해 N/2가 된다.  
    - 더 가까운 근사치를 얻기 위해서는 더 많은 진동수들을 조화시켜야 한다. N/2, 3N/2, 5N/2를 조화한 경우 디지털 신호에 좀 더 가까운 모양이 나왔음을 확인할 수 있습니다.  
  - 기저대역 전송에서 필요한 대역폭은 비트 전송률에 비례합니다. 비트를 더 빠르게 보내야 하는 경우 더 많은 대역폭이 필요합니다.

  - **broadband transmission** (modulation, 광대역 전송): 디지털 신호를 전송하기 위해 아날로그 신호로 변조 후 사용한다. 이때 밴드패스 채널 (Bandpass Channel, 대역폭이 0으로 시작하지 않는 채널)을 사용한다. 이것은 로우패스 채널보다 더 가능하게 한다.

![DC broadband transmission](/../assets/img/network/2024-07-15-DC_broadband transmission.JPG)

  - 아래 그림은 디지털 신호를 변조하는 과정을 나타낸다. 디지털 신호는 복합 아날로그 신호로 변환되는데, Carrier라는 단일 주파수 신호를 이용한다. Carrier의 진폭 변화를 통해 아날로그 신호를 디지털 신호처럼 보이도록 하는 것이다.

![DC modulation process](/../assets/img/network/2024-07-15-DC_modulation process.JPG)

## 3.4 Transmission Impairment
- 전송 매체를 통한 신호는 전송 장애에 의해 완벽하게 전송되지 않는다. 이는 매체를 통하기 전의 신호와 통한 후의 신호가 완전히 같지 않다는 것을 의미한다. 이러한 전송 장애에는 세 가지(감쇠, 왜곡, 노이즈)가 있다.

### 3.4.1 Attenuation (감쇠)

![DC Attenuation](/../assets/img/network/2024-07-15-DC_Attenuation.JPG)

- 전송 매체의 저항으로 인해 에너지가 손실되어 생기는 현상.  
  - 감쇠를 방지하기 위해 매체 사이에 증폭기 (Amplifier)를 설치해 중간중간 신호의 진폭을 끌어올려준다.  
  - **데시벨 (Decibel, dB)**: 신호가 손실되거나 증폭되는 정도를 표시한 단위. dB = 10 log10(P2/P1)  
    - P1, P2는 1, 2번 지점에서의 신호 세기이다. 데시벨 값이 음수이면 신호가 감쇠했다는 것을 의미하고, 양수이면 신호가 증폭되었다는 것을 의미한다.

### 3.4.2 Distortion (왜곡)

![DC Distortion](/../assets/img/network/2024-07-15-DC_Distortion.JPG)

- 신호의 모양이 변하는 것  
  - 서로 다른 주파수의 여러 신호로 만들어진 복합 신호에서 주로 발생한다. 각 신호는 신호마다 전송 속도 (Propagation Speed)가 다르기 때문에, 전송 매체에 따라 각 신호마다 지연이 발생할 수 있다.  
  - 복합 신호를 이루는 신호들의 위상 (Phase)의 차이가 발생해 왜곡이 일어난다.

### 3.4.3 Noise(노이즈)

![DC Noise](/../assets/img/network/2024-07-15-DC_Noise.JPG)

- 노이즈는 여러 원인이 있다.  
  - **열 노이즈 (Thermal Noise)**: 회선 내의 전자의 무작위 움직임으로 발생하는 추가 신호에 의해 일어난다.  
  - **유도 노이즈 (Induced Noise)**: 모터나 가전제품 등이 안테나의 역할을 하여 발생하는 노이즈다.  
  - **혼선 (Crosstalk)**: 서로 다른 회선끼리 신호에 영향을 주어 발생한다.  
  - **충격 노이즈 (Impulse Noise)**: 회선에 가해지는 충격에 의해 발생한다.

- **Signal-to-Noise Ratio (SNR)**: 노이즈 세기에 대한 신호 세기의 비율  
  - $$ \text{SNR} = \frac{\text{average signal power}}{\text{average noise power}} $$
  - SNR이 낮다는 것은 그만큼 원래의 신호가 노이즈에 의해 많이 손상되었다는 것을 의미한다. (노이즈 세기가 약하므로)  
  - SNR은 두 세기의 비를 나타낸 것이기 때문에, 데시벨로 나타낸 SNR은 다음과 같다.  
    - $$ \text{SNR}_{\text{dB}} = 10 \log_{10}(\text{SNR}) $$