---
layout: post
title: 데이터 통신 - Analog Transmission (1)
date: 2024-07-22 19:20:23 +0900
category: DataCommunication
---
# Analog Transmission

## Digital-to-Analog Conversion (디지털→아날로그 변조)

### 큰그림

디지털 비트열 \( \{0,1\} \)을 **반송파(carrier)** \( c(t)=\cos(2\pi f_c t) \)의 **진폭/주파수/위상**(혹은 I/Q 복소평면의 좌표)로 매핑해 **대역통과(passband)** 신호를 만든다.
핵심 변수:

- **레벨 수 \(L\)** (심볼당 표현 가능한 상태 수), **비트/심볼 수 \(r=\log_2 L\)**
- **데이터율 \(N\) [bps]**, **신호율/심볼률 \(S\) [baud]**
  $$ S=\frac{N}{r} $$
- **펄스성형/롤오프 \(\alpha\)** (RRC/RC 필터), **점유대역폭 \(B\)**
  - PSK/QAM/ASK의 **총 대역폭** 근사:
    $$ B \approx (1+\alpha)\,S $$
  - **FSK**의 총 대역폭은 **주파수 편이 \(\Delta f\)** 까지 고려:
    - **카슨(Carson) 근사)**:
      $$ B \approx 2\big(\Delta f + (1+\alpha)\tfrac{S}{2}\big) \;\;\approx\;\; 2\Delta f + (1+\alpha)S $$

> 주의: 일부 교재/노트의 \(B=(1+d)S\) 표기는 **롤오프 \(\alpha\)** 개념을 단순화한 근사다. 여기서는 **RRC 롤오프 \(\alpha\)** 기준의 현대적 표기를 사용한다.

---

### 변조의 공통 관점

#### 데이터 요소 vs 신호 요소

- **데이터 요소**: 비트(혹은 비트 묶음)
- **신호 요소(심볼)**: 반송파의 **진폭/주파수/위상(I/Q)**로 표현되는 **파형 단위**
- **심볼당 비트수** \(r=\log_2 L\), **심볼률** \(S=N/r\)

#### 대역폭

- RRC 필터(롤오프 \(\alpha\)) 사용 시 **양측 대역폭**은 \((1+\alpha)S/2\), 총점유는 \((1+\alpha)S\)

#### 반송파

- 변조 신호는 일반적으로
  $$ s(t)=\Re\{\,x(t)\,e^{j2\pi f_c t}\} $$
  여기서 \(x(t)\)는 **저역 복소 베이스밴드** (I/Q) 신호.

---

### Amplitude Shift Keying (ASK)

#### Binary ASK (BASK / OOK)

- **아이디어**: 비트에 따라 **진폭**을 바꿈. 대표적으로 **OOK**(On-Off Keying):
  - \(b=1 \Rightarrow A\cos(2\pi f_c t)\), \(b=0 \Rightarrow 0\)
- **대역폭**:
  $$ B \approx (1+\alpha)\,S $$
- **장점**: 구현 간단.
- **단점**: 진폭 잡음에 취약, 전력 효율 낮음.

#### M-ASK (다준위 ASK)

- 레벨 \(L\)개 → 심볼당 \(r=\log_2 L\) 비트. 실무에선 **QAM**이 ASK의 상위호환(진폭+위상)으로 더 자주 사용.

#### 간단한 BASK 파이썬 예

```python
import numpy as np

def rrc_pulse(beta=0.25, span=8, sps=8):
    # RRC 임펄스 응답 생성(학습용 간이판)
    T = 1
    t = np.arange(-span*T, span*T+1/sps, 1/sps)
    h = np.zeros_like(t)
    for i,ti in enumerate(t):
        if abs(1- (4*beta*ti/T)**2) < 1e-12:
            h[i] = (np.pi/4)*np.sinc(1/(2*beta))
        else:
            h[i] = (np.sinc(ti/T) * np.cos(np.pi*beta*ti/T)) / (1-(2*beta*ti/T)**2)
    return h/np.sqrt(np.sum(h*h))

def bask_mod(bits, fc=10e3, fs=200e3, Rs=1e3, A=1.0, alpha=0.25):
    sps = int(fs/Rs) # samples per symbol
    base = np.repeat(bits.astype(float), sps)  # NRZ OOK
    # RRC 성형(옵션)
    h = rrc_pulse(beta=alpha, sps=sps)
    shaped = np.convolve(base, h, 'same')
    t = np.arange(len(shaped))/fs
    carrier = np.cos(2*np.pi*fc*t)
    return shaped*carrier, fs
```

---

### Frequency Shift Keying (FSK)

#### Binary FSK (BFSK)

- **아이디어**: 두 주파수로 비트를 표현
  - \(b=0 \Rightarrow f_1=f_c-\Delta f\), \(b=1 \Rightarrow f_2=f_c+\Delta f\)
- **정합 주파수 간격**:
  - **직교(orthogonal) BFSK**: \( \Delta f = \tfrac{S}{2} \) (coherent) 또는 \( \Delta f = S \) (noncoherent) 설계가 흔함
- **대역폭(Carson 근사)**:
  $$ B \approx 2\Delta f + (1+\alpha)S $$
- **검파**:
  - **Coherent**(동기): 반송 위상 동기 필요, 성능 ↑
  - **Noncoherent**(비동기): 구현 단순, 성능 ↓

#### M-FSK

- 레벨 \(L\)개 → \(L\)개의 주파수 사용. **대역폭↑**, 심볼 에너지 분산. 고차 M에서는 대체로 **QAM/PSK** 선호.

---

### Phase Shift Keying (PSK)

#### BPSK

- **아이디어**: 위상 0°/180° 두 상태
  - \(b\in\{0,1\}\Rightarrow s=\pm A\cos(2\pi f_c t)\)
- **대역폭**:
  $$ B \approx (1+\alpha)\,S $$
- **BER(이상채널, 코히어런트)**:
  $$ P_b = Q\!\Big(\sqrt{2\,\frac{E_b}{N_0}}\Big) $$

#### QPSK

- 심볼당 2비트(\(r=2\)), 위상 4상태(45°, 135°, −45°, −135° 등), **대역폭은 BPSK와 동일한 심볼률 기준**.
- **BER(그레이 코딩)**: BPSK와 동일식
  $$ P_b = Q\!\Big(\sqrt{2\,\frac{E_b}{N_0}}\Big) $$

#### OQPSK, π/4-DQPSK

- **OQPSK**: I/Q 반 비트 시프트로 **위상 점프**(180° 등) 제한 → **피크 전력/필터링** 안정
- **π/4-DQPSK**: 위상 변화가 \(\{\pm\pi/4, \pm 3\pi/4\}\)로 제한, **비동기/이동통신** 친화

---

### Quadrature Amplitude Modulation (QAM)

#### 개념

- **ASK + PSK**의 결합 = **I/Q 평면 상의 격자(성상도)**에 심볼 배치
- \(M\)-QAM: \(L=M\)개의 점 → 심볼당 \(r=\log_2 M\) 비트
- **대역폭**:
  $$ B \approx (1+\alpha)\,S = (1+\alpha)\,\frac{N}{\log_2 M} $$
  → **같은 데이터율 N**에서 **M 증가** 시 **심볼률 S↓ → 대역폭 절약**

#### BER 근사(그레이 코딩, 코히어런트)

- **M-PSK**(큰 \(M\) 근사):
  $$ P_b \approx \frac{2}{\log_2 M}\,Q\!\Big(\sqrt{2\,\frac{E_s}{N_0}}\sin\frac{\pi}{M}\Big) $$
- **M-QAM**(정사각형 QAM):
  $$ P_b \approx \frac{2(1-1/\sqrt{M})}{\log_2 M}\;
  Q\!\Bigg(\sqrt{\frac{3\,\log_2 M}{M-1}\,\frac{E_b}{N_0}}\Bigg) $$

> **트레이드오프**: \(M\)을 키우면 **대역폭 절약**↑, 그러나 **심볼간 최소거리**↓ → **요구 SNR↑**.

#### 간단 16-QAM 매퍼 예(그레이 코딩 예시 중 하나)

```python
import numpy as np

# 16-QAM: I,Q ∈ {-3,-1,+1,+3} (정규화 전)

GRAY2 = { '00':-3, '01':-1, '11':+1, '10':+3 }

def bits_to_syms_16qam(bits):
    # 입력 길이는 4의 배수라고 가정 (I용 2비트, Q용 2비트)
    syms = []
    for i in range(0, len(bits), 4):
        bi = ''.join(bits[i:i+2])
        bq = ''.join(bits[i+2:i+4])
        I = GRAY2[bi]
        Q = GRAY2[bq]
        syms.append(complex(I, Q))
    # 평균 에너지가 1이 되도록 정규화(선택)
    avgE = np.mean(np.abs(syms)**2)
    syms = [s/np.sqrt(avgE) for s in syms]
    return np.array(syms, dtype=complex)

def qam_passband(syms, fc=20e3, fs=400e3, Rs=10e3, alpha=0.25):
    sps = int(fs/Rs)
    # 업샘플 & RRC 성형은 생략/과제: rrc_pulse 사용
    base = np.zeros(sps*len(syms), dtype=complex)
    base[::sps] = syms
    t = np.arange(len(base))/fs
    i = np.real(base); q = np.imag(base)
    carrier_i = np.cos(2*np.pi*fc*t)
    carrier_q = -np.sin(2*np.pi*fc*t) # e^{j2πfct}의 허수부 부호 고려
    s = i*carrier_i + q*carrier_q
    return s, fs
```

---

## 5.1.x 성상도(Consetllation)와 그레이 코딩

- **성상도**: I/Q 평면의 **심볼 점 배치**. 인접 점 간 **해밍거리 1**이 되도록 **그레이 코딩**을 사용해 **비트 오류 → 심볼 오류 1비트화**를 유도, **BER 개선**.
- **정규화**: 평균 심볼 에너지 \(E_s\)를 1로 맞춰 **SNR 비교**를 공정화.

---

## 5.1.y 펄스 성형, 롤오프, ISI

- **RRC(근근) / RC(근) 필터**: 심볼 간 간섭(**ISI**) 억제, 스펙트럼 제어.
- **롤오프 \(\alpha\)**: 0(이상적 Nyquist)~1. \(\alpha↑\Rightarrow\) 대역폭↑, 구현 용이/타이밍 여유↑.
  $$ B \approx (1+\alpha)\,S $$

---

## 5.1.z 성능(이상 채널) 요약

- **BPSK/QPSK (coherent)**:
  $$ P_b = Q\!\Big(\sqrt{2\,E_b/N_0}\Big) $$
- **Noncoherent BFSK**:
  $$ P_b \approx \frac{1}{2}\exp\!\Big(-\frac{E_b}{2N_0}\Big) $$
- **Coherent BFSK**(직교):
  $$ P_b = Q\!\Big(\sqrt{2\,E_b/N_0}\Big) $$
- **M-QAM(정사각형, 근사)**:
  $$ P_b \approx \frac{2(1-1/\sqrt{M})}{\log_2 M}\;
  Q\!\Bigg(\sqrt{\frac{3\,\log_2 M}{M-1}\,\frac{E_b}{N_0}}\Bigg) $$

---

## 실전 시나리오와 계산

### 시나리오 A — **데이터율 20 Mbps, 롤오프 0.25, 16-QAM**

- \(M=16 \Rightarrow r=4\), \(N=20\) Mbps → **심볼률**
  $$ S=\frac{N}{r}=5\ \text{MBd} $$
- **대역폭(근사)**:
  $$ B\approx (1+0.25)\times 5=6.25\ \text{MHz} $$
- 동일 \(N\)에서 **QPSK**라면 \(r=2\Rightarrow S=10\ \text{MBd}\Rightarrow B\approx 12.5\ \text{MHz}\) → **16-QAM이 대역 절반**!

### 시나리오 B — **BFSK, 직교, S=200 kBd**

- 직교 BFSK(코히어런트)라면 \(\Delta f=S/2=100\,\text{kHz}\)
- **대역폭(카슨 근사)**:
  $$ B\approx 2\Delta f + (1+\alpha)S \approx 2\cdot 100k + 1.25\cdot 200k = 450\ \text{kHz} $$

### 시나리오 C — **목표 BER=10⁻⁵에서 QPSK가 요구하는 \(E_b/N_0\)**

- \(P_b = Q(\sqrt{2\gamma_b})=10^{-5}\)에서 \(\sqrt{2\gamma_b}\approx 4.265\Rightarrow \gamma_b\approx 9.1\ \text{(linear)}\approx 9.6\ \text{dB}\)

---

## 실습 미니 모뎀 (BPSK/QPSK/16-QAM) — 베이스밴드 AWGN

```python
import numpy as np
from numpy.random import default_rng
from math import erfc, sqrt, log

rng = default_rng(0)

def qfunc(x):
    return 0.5*erfc(x/np.sqrt(2))

def ebn0_to_sigma2(ebn0_db, r):
    # r = bits/symbol, Eb/N0(dB) -> 노이즈 분산 (Es=1 정규화 가정)
    ebn0 = 10**(ebn0_db/10)
    esn0 = ebn0 * r
    N0 = 1/esn0
    sigma2 = N0/2
    return sigma2

def bpsk_mod(bits):
    return 2*bits.astype(float)-1  # 0->-1, 1->+1

def bpsk_demod(y):
    return (y>0).astype(int)

def qpsk_mod(bits):
    # 그레이: 00:+1+1j, 01:-1+1j, 11:-1-1j, 10:+1-1j (정규화)
    bits = bits.reshape(-1,2)
    m = []
    for b0,b1 in bits:
        if   (b0,b1)==(0,0): z=1+1j
        elif (b0,b1)==(0,1): z=-1+1j
        elif (b0,b1)==(1,1): z=-1-1j
        else:                z=1-1j
        m.append(z/np.sqrt(2))
    return np.array(m)

def qpsk_demod(z):
    z = z*np.sqrt(2)
    out=[]
    for s in z:
        b0 = 0 if s.real>0 else 1
        b1 = 0 if s.imag>0 else 1
        out += [b0,b1]
    return np.array(out)

def awgn(x, sigma2):
    n = np.sqrt(sigma2)*(rng.standard_normal(x.shape) + 1j*rng.standard_normal(x.shape))
    return x + n

# 스위프: QPSK

nBits = 200000
bits = rng.integers(0,2,size=nBits)
x = qpsk_mod(bits)
for ebn0_db in [4,6,8,10]:
    sigma2 = ebn0_to_sigma2(ebn0_db, r=2) # QPSK r=2
    y = awgn(x, sigma2)
    bhat = qpsk_demod(y)
    ber = np.mean(bhat!=bits)
    theo = qfunc(np.sqrt(2*10**(ebn0_db/10)))
    print(ebn0_db, "dB  sim:", round(ber,5), "  th:", round(theo,5))
```

---

## 정리

- **공식 묶음**
  - 심볼률:
    $$ S=\frac{N}{\log_2 L} $$
  - PSK/QAM/ASK 대역폭:
    $$ B\approx (1+\alpha)S $$
  - BFSK 대역폭(근사):
    $$ B\approx 2\Delta f + (1+\alpha)S $$
  - BPSK/QPSK BER(coherent, AWGN):
    $$ P_b=Q\!\Big(\sqrt{2E_b/N_0}\Big) $$
  - M-QAM BER(근사):
    $$ P_b \approx \frac{2(1-1/\sqrt{M})}{\log_2 M}\,Q\!\Big(\sqrt{\tfrac{3\log_2 M}{M-1}\tfrac{E_b}{N_0}}\Big) $$

- **선택 가이드**
  - **대역폭이 귀하다** → **고차 QAM/PSK**, 좋은 SNR/FEC 필요
  - **잡음/페이딩 심함** → **저차 변조(BPSK/QPSK)** + 강한 FEC
  - **비동기/저복잡** → **FSK/noncoherent**
  - **EMI/스펙트럼 제약** → **펄스성형/RRC, \(\alpha\) 조절, OQPSK/π/4-DQPSK**

---

## 체크 문제

1. **QPSK**로 50 Mbps 전송, \(\alpha=0.35\). 점유 대역폭을 구하라.
   $$S=\frac{N}{2}=25\,\text{MBd},\quad B\approx(1+0.35)S=33.75\,\text{MHz}$$
2. **직교 BFSK(coherent)**에서 \(\Delta f=S/2\). \(S=100\) kBd, \(\alpha=0.25\). 대역폭은?
   $$B\approx 2\cdot 50k + 1.25\cdot 100k = 250\,\text{kHz}$$
3. **16-QAM**에서 \(N=24\) Mbps, \(\alpha=0.25\). \(B\)는?
   $$S=N/4=6\,\text{MBd},\; B\approx 1.25\cdot 6=7.5\,\text{MHz}$$
4. **BER=10^{-6}**를 원할 때 QPSK의 요구 \(E_b/N_0\) (dB)는 대략 얼마인가?
   \(Q(\sqrt{2\gamma_b})=10^{-6}\Rightarrow \sqrt{2\gamma_b}\approx 4.753\Rightarrow \gamma_b\approx 11.3\,(10.5\text{ dB})\)
5. RRC 롤오프 \(\alpha\)를 키우면 \(B\)와 시간영역 파형은 각각 어떻게 변하는가?

---

## 보너스: 표준과의 연결(감)

- **100BASE-TX**: 4B/5B → NRZ-I → **MLT-3**(베이스밴드)
- **LTE/5G/와이파이**: **OFDM**(다중 부반송), 각 부반송에서 **QPSK/16-QAM/64-QAM/256-QAM**
- **광 전송/고속 직렬 링크**: **PAM-4**(레벨 4, 심볼당 2비트) + 강력한 **FEC**
