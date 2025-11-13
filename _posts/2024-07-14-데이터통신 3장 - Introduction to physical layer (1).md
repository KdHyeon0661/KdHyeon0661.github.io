---
layout: post
title: 데이터 통신 3장 - Introduction to Physical Layer (1)
date: 2024-07-14 19:20:23 +0900
category: DataCommunication
---
# 3. Introduction to Physical Layer

## 3.1 Data and Signals

### 3.1.1 Analog and Digital Data
- **Analog Data(아날로그 데이터)**: 시간에 대해 **연속적(continuous)** 값. 예: 온도, 음압, 빛의 세기.
- **Digital Data(디지털 데이터)**: 시간에 대해 **이산적(discrete)** 값(유한한 기호/비트). 예: 문자 코드, 정수 샘플.

### 3.1.2 Analog and Digital Signals
- **Signal(신호)**: 매체를 통해 전달되는 물리량(전압/전류/광세기/전계 등)의 시간적 변화.
- **Analog Signal(아날로그 신호)**: 진폭이 연속값을 가짐(예: 사인파).
- **Digital Signal(디지털 신호)**: 진폭이 제한된 유한 집합(예: NRZ-L의 {0,1} 레벨).

> 데이터(논리적)와 신호(물리적)는 구분하되, **아날로그 데이터→디지털 신호(PCM)**, **디지털 데이터→아날로그 신호(ASK/FSK/PSK/QAM)** 처럼 서로 변환 가능.

### 3.1.3 Periodic and Nonperiodic Signals
- **Periodic(주기)**: 어떤 **주기 \(T\)** 마다 반복:
  $$ x(t+T)=x(t) $$
- **Nonperiodic(비주기)**: 반복 주기가 없음. (랜덤 트래픽/버스트 신호 등)

### 3.1.4 Baseband vs Passband (추가)
- **Baseband(기저대역)**: 0Hz 근처에 에너지가 존재. 유선 LAN의 라인코딩 신호가 대표적.
- **Passband(통과대역)**: 반송파 주파수 \(f_c\) 주변에 에너지. 무선/광/케이블 모뎀 등 **변조**가 필요.

---

## 3.2 Periodic Analog Signals

### 3.2.1 Sine Wave(사인파) — \(A\), \(f\), \(\phi\)

사인파는 **진폭 \(A\)**, **주파수 \(f\)**, **위상 \(\phi\)** 로 정의됩니다.

$$
x(t)=A\sin(2\pi f t+\phi)
$$

- **Peak Amplitude \(A\)**: 최대 절대 진폭(단위: V, A, W\(^{1/2}\) 등)
- **Angular Frequency**: \(\omega=2\pi f\) (단위 rad/s)
- **Period \(T\)**: 한 사이클 시간, \( T=\frac{1}{f} \) (단위 s)
- **RMS**(교류 전력 계산 시 유용): \( A_\text{RMS}=\frac{A}{\sqrt{2}} \)

#### 단위 변환
- 각도↔라디안:
  $$ 360^\circ =2\pi \text{ rad} \qquad 1^\circ=\frac{\pi}{180}\text{ rad} $$

### 3.2.2 Phase(위상)
- **위상 \(\phi\)**: 기준 시각에서 파형의 위치(각도로 표현).
- 예) **한 사이클의 \(\frac{1}{6}\)** 만큼 앞서면:
  $$ \phi = \frac{1}{6}\times 360^\circ = 60^\circ = \frac{\pi}{3} \text{ rad} $$

### 3.2.3 Wavelength(파장) — **오류 교정 포함**
- **파장 \(\lambda\)**: 공간에서 파형이 한 주기마다 반복되는 길이.
  $$ \lambda = v \cdot T = \frac{v}{f} $$
  - \(v\): 매질에서의 **전파 속도**(m/s).
  - 진공의 빛 속도 \(c \approx 3\times10^8 \,\text{m/s}\).
  - **구리/광섬유**에선 굴절률/유전율에 따라 \(v\approx (1.5\sim2.1)\times 10^8\,\text{m/s}\) 정도.

> (교정) 초안의 “\(\lambda=c\cdot f\)”은 **오류**입니다. 정식은 **\(\lambda=\frac{c}{f}\)** 입니다. 또한 적외선의 전형적 주파수는 \(\sim 10^{14}\)Hz 오더로, \(\lambda\sim \mu\)m 수준이 맞습니다.

**예시 1(동축 케이블)**
구리에서 \(v \approx 2.0\times 10^8\) m/s, \(f=100\) MHz이면
$$ \lambda = \frac{2.0\times10^8}{1.0\times10^8}=2.0\,\text{m} $$

**예시 2(근적외선)**
진공에서 \(f=2.0\times10^{14}\) Hz이면
$$ \lambda = \frac{3.0\times10^8}{2.0\times10^{14}}=1.5\times10^{-6}\,\text{m}=1.5\,\mu\text{m} $$

### 3.2.4 Time/Frequency Domains(시간/주파수 영역)
- **Time Domain**: \(x(t)\) vs \(t\), 파형의 시간적 변화.
- **Frequency Domain**: \(X(f)\) vs \(f\), 각 주파수 성분의 **진폭/위상**.
  - 순수 사인파는 \(f_0\)에서 **스파이크**(델타) 하나로 표현.

### 3.2.5 Composite Signals(합성 신호) & Fourier
- **Fourier 분석**: 임의의 신호를 **서로 다른 \(f,A,\phi\)** 를 갖는 사인파들의 **합**으로 분해.
  - **주기 신호**: **푸리에 급수** — 이산 주파수 성분.
  - **비주기 신호**: **푸리에 변환** — 연속 스펙트럼.

**정사각파 예시(이상적)**
기본주파수 \(f_0\), 홀수 고조파만 존재(진폭 \(\propto 1/n\)).
$$
x(t) = \frac{4A}{\pi}\left(\sin(2\pi f_0 t)+\frac{1}{3}\sin(2\pi 3f_0 t)+\frac{1}{5}\sin(2\pi 5f_0 t)+\cdots\right)
$$

### 3.2.6 Bandwidth(대역폭)
- **정의**: 스펙트럼에서 **최대 주파수 \(-\)** 최소 주파수. (단위 Hz)
  $$ \text{BW} = f_\text{max} - f_\text{min} $$
- 예) 주파수 \(\{100,300,500,700,900\}\) Hz →
  $$\text{BW}=900-100=800\text{ Hz}$$

> **실효 대역폭**: 실제로 **의미 있는 에너지**가 분포한 구간(예: 99% 에너지 포함 구간)을 의미하기도 함.

---

## 3.3 신호·대역폭·데이터율의 관계(핵심 이론 보강)

### 3.3.1 나이퀴스트 한계(무잡음·기저대역)
무잡음 채널에서 레벨 \(M\)을 사용하는 경우 **최대 비트율** 근사:
$$
R_\text{max} = 2B \log_2 M \quad (\text{bps})
$$
- \(B\): 채널 대역폭(Hz), \(M\): 신호 레벨(예: 2레벨=이진).

**예시**: \(B=3\) kHz, \(M=4\) →
\(R_\text{max}=2\cdot 3000\cdot \log_2 4 = 2\cdot 3000 \cdot 2 = 12\) kbps

### 3.3.2 섀넌 용량(잡음 존재·패스밴드/일반)
잡음이 존재하는 현실 채널의 **이론적 최대 전송 용량**:
$$
C = B \log_2 (1+\text{SNR}) \quad (\text{bps})
$$
- \(\text{SNR}\): 선형 비(신호전력/잡음전력).
- dB 변환:
  $$ \text{SNR}_{\text{dB}} = 10\log_{10}\text{SNR} \quad \Longleftrightarrow \quad \text{SNR}=10^{\text{SNR}_{\text{dB}}/10} $$

**예시**: \(B=1\) MHz, \(\text{SNR}_{\text{dB}}=20\) dB → \(\text{SNR}=100\)
\(C = 10^6 \log_2(101) \approx 10^6 \times 6.658 \approx 6.66\) Mbps

> **나이퀴스트**는 **레벨 수/기저대역**에 초점, **섀넌**은 **잡음/SNR**에 초점. 두 관점의 **교집합 영역에서 실제 가능한 설계 범위**를 가늠합니다.

---

## 3.4 라인코딩·샘플링·변조(실무 감각 보강)

### 3.4.1 라인코딩(디지털→기저대역)
- **NRZ-L/NRZ-I**, **RZ**, **Manchester**, **4B/5B** 등.
- 스펙트럼 특성(DC 성분·대역폭)과 **클럭 복원** 용이성의 트레이드오프.

### 3.4.2 샘플링·양자화(아날로그→디지털)
- **샘플링 정리(나이퀴스트-섀넌)**:
  $$ f_s \ge 2 f_\text{max} $$
- **양자화**: 샘플을 근접한 레벨로 반올림 → **양자화 노이즈** 발생.
- **PCM**: 표본화→양자화→부호화.

### 3.4.3 변조(디지털→패스밴드)
- **ASK/FSK/PSK**, **QAM**(진폭·위상 동시 변조), **OFDM**(직교 부반송파).
- 스펙트럼 효율(비트/초/Hz), **Eb/N0**, BER 성능이 핵심.

---

## 3.5 코드 실습 — 합성 신호/FFT/대역폭 감각 익히기

> 그래프 렌더링 없이 **수치·텍스트** 중심으로 **주파수 피크와 대역폭**을 확인합니다.

```python
import math, random
import numpy as np

# 1. 합성 신호 만들기: 150Hz, 450Hz, 900Hz 성분 + 약간의 잡음
fs = 8000            # 샘플링 주파수(Hz)
T  = 1.0             # 관측 시간(초)
N  = int(fs*T)       # 샘플 수
t  = np.arange(N)/fs

freqs = [150, 450, 900]
amps  = [1.0, 0.6, 0.4]
phi   = [0, math.pi/6, math.pi/3]

x = np.zeros(N)
for (f, a, p) in zip(freqs, amps, phi):
    x += a*np.sin(2*math.pi*f*t + p)
x += 0.05*np.random.randn(N)      # 소량의 가우시안 잡음

# 2. FFT로 주파수 성분 관찰
X = np.fft.rfft(x)                 # 실수 신호의 양수 주파수만
F = np.fft.rfftfreq(N, d=1.0/fs)
mag = np.abs(X)/ (N/2)             # 대략적 스케일링

# 3. 상위 피크 주파수 Top-k 출력
k = 5
idx = np.argsort(mag)[-k:][::-1]
peaks = [(F[i], mag[i]) for i in idx]
print("Top peaks (Hz, magnitude):")
for f, m in peaks:
    print(f"{f:.1f} Hz -> {m:.3f}")

# 4. 실효 대역폭(예: 에너지 99% 포함 구간) 근사
power = mag**2
cum = np.cumsum(np.sort(power))
total = power.sum()
th = 0.99*total

# 역정렬로 임계치 초과 지점 찾아 경계 근사
sorted_idx = np.argsort(power)[::-1]
acc = 0.0
active_bins = []
for i in sorted_idx:
    if acc >= th: break
    acc += power[i]
    active_bins.append(i)

f_active = F[active_bins]
bw_est = (f_active.max() - f_active.min()) if len(f_active) else 0.0
print(f"Estimated effective bandwidth ≈ {bw_est:.1f} Hz")
```

- 예상 출력(예): `150/450/900 Hz` 부근이 상위 피크로 잡힘.
- **실효 대역폭**은 **최저~최고 주파수** 범위에 근거해 대략 추정.

---

## 3.6 SNR·dB 계산기(코드)

```python
import math

def db_to_linear(db):
    return 10**(db/10.0)

def linear_to_db(x):
    return 10*math.log10(x)

def shannon_capacity(B_hz, SNR_db):
    SNR = db_to_linear(SNR_db)
    return B_hz * math.log2(1+SNR)

print("SNR 20 dB -> linear:", db_to_linear(20))
print("Capacity @B=1e6Hz, 20dB:", round(shannon_capacity(1_000_000, 20)/1e6, 3), "Mbps")
```

---

## 3.7 간단 연습 문제

1) 구리 매체에서 \(v=2.0\times10^8\,\text{m/s}\), \(f=50\) MHz일 때 파장 \(\lambda\)는?
   $$ \lambda=\frac{2.0\times10^8}{5.0\times10^7}=4.0\,\text{m} $$
2) \(B=4\) kHz, 무잡음, \(M=8\) 레벨이면 **나이퀴스트 비트율**은?
   $$ R_\text{max}=2\cdot 4000 \cdot \log_2 8=8000\cdot 3=24\text{ kbps} $$
3) \(B=5\) MHz, \(\text{SNR}_{\text{dB}}=10\) dB에서 **섀넌 용량**은?
   \(\text{SNR}=10\) → \(C=5\times10^6 \log_2(11)\approx 5\times10^6\times 3.459\approx 17.3\) Mbps
4) 정사각파의 스펙트럼에서 **어떤 고조파**들이 존재하는가?
   **홀수 고조파(1,3,5,...)** 만 존재, 진폭은 \(\propto 1/n\).

---

## 3.8 요약 체크리스트

- **사인파**는 \(A,f,\phi\)로 완전 규정. \(\omega=2\pi f\), \(T=1/f\).
- **파장**은 \(\lambda=\frac{v}{f}\) (초안의 \(c\cdot f\)는 오류). 매질에 따라 \(v\)가 변함.
- **시간/주파수 영역**을 왕복하며 **Fourier**로 복잡 신호를 해석.
- **대역폭(BW)** 는 스펙트럼 폭. **나이퀴스트/섀넌**으로 **데이터율 한계**를 가늠.
- **라인코딩/샘플링/변조**는 실제 시스템 설계의 핵심 도구.
- **SNR(dB)** 계산에 익숙해지면 용량·BER·링크버짓 추정이 쉬워진다.

---

# 부록 A: 단위·상수(자주 쓰는 값)
- \(c\approx 3.0\times10^8\) m/s (진공)
- 구리/트위스티드페어 전파속도 \(v\approx (1.9\sim2.1)\times10^8\) m/s
- \(1\text{ kHz}=10^3\text{ Hz}\), \(1\text{ MHz}=10^6\text{ Hz}\), \(1\text{ GHz}=10^9\text{ Hz}\)

# 부록 B: 현업 팁
- **케이블 길이/주파수**가 커지면 \(\lambda\) 대비 길이가 커져 **전송선 효과**(반사/정재파)가 나타남 → 임피던스 정합/터미네이션 필수.
- **무선**은 다중경로 페이딩으로 **주파수 선택적 페이드** 발생 → OFDM·다이버시티·채널코딩.
