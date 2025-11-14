---
layout: post
title: 데이터 통신 3장 - Introduction to Physical Layer (2)
date: 2024-07-15 19:20:23 +0900
category: DataCommunication
---
# Digital Signaling Fundamentals

본 글은 기존 초안을 **정확성 보강**(잘못된/모호한 부분 교정) + **실전 수치·코드** + **추가 이론**으로 확장했습니다.
요청에 따라 **그림 링크는 모두 제거**했으며 수식은 **MathJax( \$\$ ... \$\$ )**, 코드는 ```로 표기합니다.

---

## Bit Rate (비트율)

- **정의**: 단위 시간(초)당 전송되는 **비트 수**
  $$ \text{bit rate } R \;[\text{bps}] = \frac{\text{전송 비트 수}}{\text{초}} $$

### 예시 1) “100페이지 책” 전송량과 비트율 구분

- 가정(초안 보정):
  - 한 페이지에 **문장 24개**, 각 문장 **문자 80개**, 문자 1개 **8비트**(ASCII)
  - 총 데이터량(비트):
    $$ 100 \times 24 \times 80 \times 8 = 1{,}536{,}000\ \text{bits} \ (=192{,}000\ \text{B} \approx 187.5\ \text{KiB}) $$
- 위 계산은 **데이터량**이지 **비트율**이 아닙니다.
  비트율을 알려면 **전송에 걸린 시간**이 필요합니다.
  - 예) 1.536 Mb를 **1초**에 보냈다면 **1.536 Mbps**
  - 예) 0.5초에 보냈다면 **3.072 Mbps**

### 예시 2) “HDTV 비트레이트”

- 가정(원시 비압축 RGB):
  - 해상도 \(1920\times1080\), 프레임 레이트 \(30\ \text{fps}\), 픽셀당 24비트(RGB 8:8:8)
  - **원시 비트율**:
    $$ 1920\times1080\times 30\times 24 \approx 1.492{,}992{,}000\ \text{bps} \approx 1.49\ \text{Gbps} $$
- 실제 방송/스트리밍은 **압축 코덱**(H.264/H.265/AV1) + **샘플링 방식**(예: 4:2:0, 8~10bit) 사용 → 수~십 Mbps~수백 Mbps까지 다양.
  즉 “1.5 Gbps”는 **비압축**일 때의 이론치임을 명확히 하자.

### 필수 팁

- **bit rate**(bps) vs **symbol rate**(Baud) 구분:
  \(M\)레벨 심볼이면
  $$ R_{\text{bit}} = R_{\text{symbol}}\cdot \log_2 M $$

---

## Bit Length (비트 길이)

- **정의**: 전송 중 **공간상** 한 비트가 차지하는 **길이**
  $$ \text{bit length } \ell = v \times \tau $$
  - \(v\): 매체의 **전파 속도**(m/s)
  - \(\tau\): **비트 지속시간**(s) \( = \frac{1}{R}\)

따라서
$$
\ell = \frac{v}{R}
$$

### 예시

- 트위스티드 페어에서 \(v\approx 2.0\times 10^8\ \text{m/s}\), \(R=100\ \text{Mbps}\) →
  $$ \ell \approx \frac{2.0\times10^8}{1.0\times10^8} = 2.0\ \text{m/bit} $$
- 같은 매체에서 \(R=1\ \text{Gbps}\) →
  $$ \ell \approx 0.2\ \text{m/bit} $$

**의미**: 고속일수록 **한 비트의 물리적 길이가 짧아지며**, 긴 케이블/다중경로 환경에서 **ISl(Inter-Symbol Interference)**, 반사/정재파 등의 영향을 더 쉽게 받습니다.

---

## Digital Signal as a Composite Analog Signal

(디지털 파형의 주파수 성분)

- 디지털(예: 사각) 신호는 **무한한 고조파**를 갖는 **합성 아날로그 신호**로 이해할 수 있음.
- **푸리에 해석**: 비주기 신호 → **연속 스펙트럼**, 주기 신호 → **이산 고조파**.
- 이 관점에서 디지털 신호의 **대역폭(BW)** 을 정의/근사할 수 있음.

### 정사각파의 이상적 스펙트럼(복습)

$$
x(t)=\frac{4A}{\pi}\big(\sin(\omega_0 t)+\frac{1}{3}\sin(3\omega_0 t)+\frac{1}{5}\sin(5\omega_0 t)+\cdots\big)
$$
- 홀수 고조파가 **무한대**까지 존재 → “완벽한” 사각파 재현에는 **무한 BW** 필요.
- 실제 시스템은 **유한 BW** → 파형의 모서리가 **완만**해지고, 링잉/오버슈트 가능.

---

## Transmission of Digital Signals

(Baseband vs Broadband, 실무적 해석)

### Baseband Transmission (기저대역 전송)

- **정의**: 디지털 신호를 **그대로(라인코딩 등)** 전송.
- **채널**: **Low-pass** 채널(0 Hz부터 에너지 통과).
  > 초안에 “필요로 하지 않는다”라는 문장은 정정 필요: **Baseband는 Low-pass 채널이 필요**합니다.
- **BW 요구**: 대략 **비트율에 비례**.
  - 이상적 NRZ 기준, 필요한 대역폭의 1차 근사:
    $$ B \approx \frac{R}{2} \ \ (\text{NRZ의 주된 고조파까지 고려한 근사}) $$
  - 더 충실한 파형(엣지 재현)을 원하면 **상위 고조파**까지 포함 필요 → BW ↑

### Limited-BW Low-pass 채널에서의 근사

- 제한된 BW에서 디지털 파형은 **저역 고조파**들의 합으로 **근사**됨.
  - \(R/2, \; 3R/2, \; 5R/2,\dots\)를 더 포함할수록 파형이 원형에 근접(링잉/오버슈트 주의).
- **클럭 복원**/ISI 억제를 위해 **라인코딩**(Manchester, 4B/5B, 8b/10b 등) 채택.

### Broadband Transmission (대역통과/변조)

- **정의**: 디지털 데이터를 **반송파**(carrier)로 **변조**(ASK/FSK/PSK/QAM/OFDM 등)하여 **Band-pass** 채널을 통해 전송.
- 예: 무선, 케이블 모뎀, 위성, 광 무선 링크 등.
- 장점: 채널의 가용 **주파수 대역(비영점 시작)** 을 효율적으로 활용, **주파수 재사용**, 전파특성 최적화.

---

## 실습 코드 1 — 비트 길이·필요 대역폭 근사 계산기

```python
def bit_length_m(v_m_per_s=2.0e8, R_bps=1.0e9):
    # v: 전파 속도, R: 비트율
    return v_m_per_s / R_bps

def baseband_bw_nrz(R_bps):
    # NRZ 기준 1차 근사 (주된 고조파까지만)
    return R_bps / 2.0

print("Bit length @v=2e8 m/s, R=1 Gbps:", bit_length_m(2.0e8, 1.0e9), "m")
print("BW (approx) for NRZ @R=1 Gbps:", baseband_bw_nrz(1.0e9)/1e6, "MHz")
```

---

# Transmission Impairment (전송 장애)

현실의 매체는 **완벽하지 않다**. 대표적인 장애:
**감쇠(Attenuation)**, **왜곡(Distortion)**, **노이즈(Noise)**.

## Attenuation (감쇠)

- 매체 저항·누설 등으로 **전력 손실** 발생 → 거리 ↑, 주파수 ↑에서 심화.
- **대응**: **증폭기(Amplifier)** 또는 **리피터(디지털 재생)** 를 적절 간격으로 배치.

### 데시벨(dB) 표기

- 전력 기준:
  $$ \text{dB} = 10\log_{10}\!\left(\frac{P_2}{P_1}\right) $$
- 전압/전류(임피던스 동일 가정) 기준:
  $$ \text{dB} = 20\log_{10}\!\left(\frac{V_2}{V_1}\right) $$
- **연쇄 구간**: dB 값은 **합산**됨(편리!).

### 예시

- 구간 A: \(-3\ \text{dB}\), 구간 B: \(-7\ \text{dB}\), 증폭 +10 dB → 총합 \(=0\ \text{dB}\) → 입력과 출력 전력이 같음.

### 코드 — dB 유틸

```python
import math

def db_from_power_ratio(p2_over_p1):
    return 10*math.log10(p2_over_p1)

def power_ratio_from_db(db):
    return 10**(db/10.0)

print("dB for half power (-3.01dB):", db_from_power_ratio(0.5))
print("Power ratio for +10dB:", power_ratio_from_db(10))
```

---

## Distortion (왜곡)

- **정의**: 파형 모양이 변형되는 현상(주파수 성분별 크기/위상 응답의 차이).
- **원인**:
  - **주파수 의존 감쇠**: 고주파 성분이 더 감쇠 → 엣지 둔화.
  - **군지연(Group Delay) 왜곡**: 주파수별 **지연 차이** → 펄스가 퍼짐 → **ISI**.
  - **멀티패스**(무선): 여러 경로가 **시간차**로 합쳐지며 페이딩/에코.
- **대응**:
  - **이퀄라이저**(채널 역특성), **프리엠퍼시스/디엠퍼시스**, **코딩/인터리빙**,
  - **OFDM**(주파수 선택적 페이딩 분산), **사이클릭 프리픽스**로 ISI 억제.

---

## Noise (노이즈)

- **열 노이즈(Thermal/Johnson-Nyquist)**:
  - 대역폭 \(B\)에서의 평균 잡음 전력:
    $$ N = kTB $$
    - \(k\): 볼츠만 상수 \(1.38\times10^{-23}\ \text{J/K}\)
    - \(T\): 절대온도(K)
    - \(B\): 대역폭(Hz)
- **유도 노이즈(Induced)**: 모터/전력선 등 외부 전자기 유도.
- **혼선(Crosstalk)**: 인접 선로 간 결합(NEXT/FEXT).
- **충격 노이즈(Impulse/Burst)**: 순간적 큰 펄스(스파크, 릴레이, ESD 등).
- **양자화 노이즈**(A/D), **위상 잡음**(발진기) 등도 실무에서 중요.

### SNR (Signal-to-Noise Ratio)

- **선형비**:
  $$ \text{SNR} = \frac{P_{\text{signal}}}{P_{\text{noise}}} $$
- **dB**:
  $$ \text{SNR}_{\text{dB}} = 10\log_{10}(\text{SNR}) $$
- **해석**: SNR이 클수록(신호 ≫ 잡음) **복원/복호**가 쉬움.
  너무 낮으면 BER 급증, 용량 \(C=B\log_2(1+\text{SNR})\)도 제한.

### 예시 — SNR과 채널 용량

- \(B=1\ \text{MHz}\), \(\text{SNR}_{\text{dB}}=20\) dB → \(\text{SNR}=100\)
  $$ C = 10^6 \log_2(101) \approx 6.66\ \text{Mbps} $$

### 코드 — SNR/용량 계산기

```python
import math

def snr_db_to_linear(db): return 10**(db/10)
def shannon_capacity_bps(B_hz, snr_db):
    snr = snr_db_to_linear(snr_db)
    return B_hz * math.log2(1+snr)

print("SNR 10dB linear:", snr_db_to_linear(10))
print("Capacity @B=5MHz,SNR=10dB:", round(shannon_capacity_bps(5e6,10)/1e6,2),"Mbps")
```

---

## 3.4.x (추가) Noise → BER 감각

- 변조/부호에 따라 **BER vs \(E_b/N_0\)**(비트당 에너지 대비 잡음밀도) 곡선이 정해짐.
- 예: BPSK의 AWGN 채널 BER 근사
  $$ \text{BER} \approx Q\!\left(\sqrt{2\frac{E_b}{N_0}}\right) $$
- **교훈**: 같은 BW라도 **코딩 이득**(FEC), **다이버시티**를 쓰면 SNR 요구가 내려감.

---

# 종합 요약

- **Bit Rate**는 “초당 비트 수”, 데이터량과 **구분**할 것.
- **Bit Length**는 \( \ell=\frac{v}{R} \), 고속일수록 짧아지고 전송선 효과 민감.
- **디지털 파형**은 합성 아날로그 신호(무한 고조파) → **BW 유한**이면 파형 왜곡 불가피.
- **Baseband**는 **Low-pass** 채널 필요, 대략 \(B\approx R/2\) 근사(NRZ 기준).
- **Broadband**는 **변조**를 통해 Band-pass 채널 사용(무선/케이블 등).
- **Impairments**: 감쇠(dB 합산), 왜곡(주파수/군지연), 노이즈(kTB, SNR) → **BER/용량** 좌우.
- 설계는 언제나 **대역폭·SNR·라인코딩/변조·FEC·이퀄라이징**의 **트레이드오프**.

---

## 빠른 체크 문제

1) 트위스티드 페어 \(v=2.0\times10^8\ \text{m/s}\), \(R=250\ \text{Mbps}\). **Bit length**?
   $$ \ell = \frac{2.0\times10^8}{2.5\times10^8} = 0.8\ \text{m} $$
2) NRZ 기준 \(R=200\ \text{Mbps}\). **필요 대역폭 1차 근사**?
   $$ B\approx \frac{R}{2}=100\ \text{MHz} $$
3) 감쇠 \(-8\ \text{dB}\), 증폭 \(+5\ \text{dB}\), 추가 감쇠 \(-4\ \text{dB}\). **총 dB**?
   \( -8+5-4=-7\ \text{dB}\) → 전력비 \(\approx 10^{-0.7}\approx 0.2\)
4) \(B=2\ \text{MHz}\), \(\text{SNR}_{\text{dB}}=6\) dB. **Shannon 용량** 대략?
   \(\text{SNR}\approx 3.98\) → \(C\approx 2\times10^6\log_2(1+3.98)\approx 2\times10^6\times 2.32\approx 4.64\ \text{Mbps}\)

---

## 부록: 현업 메모

- 고주파·고속 링크는 **채널 균일성**보다 **임피던스 정합/반사** 관리가 성능을 좌우.
- 케이블/커넥터/비아/PCB 트레이스까지 포함한 **채널 모델**(S-파라미터)로 **아이 다이어그램** 평가 권장.
- 무선은 **다중경로·시간/주파수 셀렉티브 페이딩** → OFDM + **채널 추정/등화** + **코딩**이 표준.
