---
layout: post
title: 데이터 통신 5장 - Analog Transmission (2)
date: 2024-07-23 19:20:23 +0900
category: DataCommunication
---
# 5.2 Analog-to-Analog conversion

## 5.2.0 개요 — 왜 “아날로그→아날로그” 변조인가?

유선/무선 **대역통과(passband)** 매질에서, 저주파(저역) **메시지 \(m(t)\)**를 고주파 **반송파 \(c(t)=A_c\cos(2\pi f_c t)\)**의 **진폭/주파수/위상**으로 실어 보낸다. 이를 통해

- **방송/무선**: 라디오(AM, FM), 아날로그 TV(AM/VSB + FM 음성), 항공 무선 등
- **채널 공유**: 여러 방송국이 **서로 다른 \(f_c\)**를 써서 **주파수 분할 다중화**(FDM)
- **전파 효율**: 고주파 반송파는 안테나 효율·전파 특성이 유리

핵심은 **메시지 스펙트럼 \(B\)**, 반송파 \(f_c\), 그리고 변조 방식에 따른 **점유 대역폭 \(B_{\text{RF}}\)**, **잡음 내성**, **검파 복잡도**의 균형이다.

---

## 5.2.1 Amplitude Modulation (AM)

### 5.2.1.1 기본식과 스펙트럼

표준(DSB-FC) AM 변조:
$$
s_{\text{AM}}(t)
= A_c\left[1 + \mu\, m_n(t)\right]\cos(2\pi f_c t),
$$
- \(m_n(t)\): \(|m_n(t)|\le 1\)로 **정규화된** 메시지(최대치 1)
- \(\mu\): **변조도(modulation index)**, 보통 \(0\le \mu \le 1\) (과변조 \(\mu>1\)는 왜곡/클리핑)

메시지 대역폭이 \(B\) (오디오라면 보통 \(B\approx 5\ \text{kHz}\) 또는 \(15\ \text{kHz}\))이면,
- **상하측파대(USB/LSB)**가 \(f_c\pm f\) (단, \(0\le f \le B\))에 생성
- **총 점유대역폭**:
$$
B_{\text{AM}} = 2B.
$$

### 5.2.1.2 전력·효율

캐리어 전력 \(P_c=\frac{A_c^2}{2R}\) (부하 \(R\))
AM 신호 총전력:
$$
P_{\text{tot}}
= P_c\left(1+\frac{\mu^2}{2}\right),
\qquad
P_{\text{SB}}=P_{\text{tot}}-P_c=\frac{\mu^2}{2}P_c.
$$
**측파대 효율**(정보가 실리는 전력 비율):
$$
\eta_{\text{AM}}=\frac{P_{\text{SB}}}{P_{\text{tot}}}
=\frac{\frac{\mu^2}{2}}{1+\frac{\mu^2}{2}}
\;\;(\mu=1\Rightarrow \eta\approx 33\%).
$$
> **표준 AM(DSB-FC)**는 **캐리어 전력 낭비**가 커서 효율이 낮다.

### 5.2.1.3 변형: DSB-SC / SSB / VSB

- **DSB-SC**(Double-SideBand Suppressed-Carrier):
  $$ s(t)=A_c\,m_n(t)\cos(2\pi f_c t) $$
  - **캐리어 억압** → 전력 효율↑, **동기검파(coherent)** 필요
  - 대역폭 동일: \(2B\)

- **SSB**(Single-SideBand):
  - 상·하측파대 **한쪽만 송신** → **대역폭 \(B\)** 로 절반
  - 스펙트럼 효율 최고, **정교한 필터/위상법** 필요

- **VSB**(Vestigial SideBand):
  - 한쪽 측파대 **거의 억압 + 잔여(vestige)** 유지
  - 아날로그 TV 영상 변조에 사용: **대역 절감**과 **복원성**의 절충

### 5.2.1.4 검파(demodulation)

- **포락선(에너벨로프) 검파**(다이오드+RC): DSB-FC에서 간단·저가
- **동기 검파**(코히어런트, 곱셈기+mixer): DSB-SC/SSB 필요, 위상·주파수 동기 필수

### 5.2.1.5 예제 계산

1) **AM 방송**: 오디오 \(B=5\,\text{kHz}\) →
   $$ B_{\text{AM}}=2B=10\,\text{kHz}. $$
2) **효율**: \(P_c=1\,\text{W},\ \mu=0.8\)
   총전력 \(=1\cdot(1+0.8^2/2)=1.32\,\text{W}\),
   효율 \(\eta= (0.8^2/2)/1.32\approx 0.242 \) (24.2%)

### 5.2.1.6 파이썬 AM 실습(학습용)

```python
import numpy as np

def am_mod(m, fs, fc, Ac=1.0, mu=0.8):
    # m: 정규화 메시지(|m|<=1), fs: 샘플링, fc: 반송
    t = np.arange(len(m))/fs
    s = Ac*(1+mu*m)*np.cos(2*np.pi*fc*t)
    return s

def envelope_detect(s, fs, cutoff=5000):
    # 매우 단순한 절대값+저역통과 근사(실습용)
    env = np.abs(s)
    # FIR/IIR 저역통과 필터를 쓰는 편이 좋음(여기서는 단순 이동평균)
    n = int(fs/(2*cutoff)+1)
    if n<1: n=1
    k = np.ones(n)/n
    return np.convolve(env, k, 'same')
```

---

## 5.2.2 Frequency Modulation (FM)

### 5.2.2.1 기본식과 변조지수

반송파 **순시 주파수**가 메시지에 비례:
$$
s_{\text{FM}}(t)=A_c\cos\Big(2\pi f_c t + 2\pi k_f \int_{-\infty}^{t} m(\tau)\,d\tau\Big).
$$
- \(k_f\): 주파수 감도 [Hz/V]
- **최대 주파수 편이** \(\Delta f = k_f \cdot |m(t)|_{\max}\)
- **메시지 최고 주파수** \(B(=f_m)\) 일 때 **변조지수**:
$$
\beta = \frac{\Delta f}{B}.
$$
- **협대역(NBFM)**: \(\beta \ll 1\), **광대역(WBFM)**: \(\beta \gg 1\)

### 5.2.2.2 대역폭 — 카슨의 법칙

FM 스펙트럼은 무한 측파대를 갖지만, **유효 대역폭**은 **카슨(Carson) 근사**:
$$
B_{\text{FM}} \approx 2(\Delta f + B)
= 2(1+\beta)\,B.
$$
- 질문의 표기 \(B_{FM}=2(1+\Delta)B\)는 \(\Delta\equiv\beta\)로 해석 가능.

### 5.2.2.3 잡음 내성/포만성

- **진폭 잡음**에 **강하다**(한계기/리미터 + 주파수/위상 검파) → **라디오 방송**에 적합
- 대역폭은 AM보다 넓을 수 있음(특히 WBFM)

### 5.2.2.4 검파

- **주파수/위상 편이 검파기**(비동기 가능)
- **한계기(limiter)**로 진폭 변화를 제거한 뒤 **주파수 검파** → SNR 이득

### 5.2.2.5 예제

- **방송 FM**: \(\Delta f=75\,\text{kHz}\), 오디오 \(B=15\,\text{kHz}\)
  \(\beta=75/15=5\) →
  $$ B_{\text{FM}}\approx 2(75k+15k)=180\,\text{kHz}. $$
  실제 채널 간격 200 kHz와 잘 맞는다.

- **협대역 FM(NBFM)**: 예) \(\Delta f=2.5\,\text{kHz}, B=3\,\text{kHz}\) →
  \(B_{\text{FM}}\approx 2(2.5k+3k)=11\,\text{kHz}.\)

### 5.2.2.6 파이썬 FM 실습(학습용)

```python
import numpy as np

def fm_mod(m, fs, fc, Ac=1.0, kf=5000.0):
    # m: 메시지(최대 |m|<=1 가정), kf: Hz/단위진폭
    t = np.arange(len(m))/fs
    phi = 2*np.pi*fc*t + 2*np.pi*kf*np.cumsum(m)/fs  # 적분 근사
    return Ac*np.cos(phi)

# 예: 1 kHz 톤(B=1k)과 Δf=5k → kf=Δf/|m|max
fs, fc = 200_000, 20_000
dur = 0.01
t = np.arange(int(fs*dur))/fs
m = np.sin(2*np.pi*1_000*t) # |m|<=1
s = fm_mod(m, fs, fc, kf=5_000)  # Δf≈5k
```

---

## 5.2.3 Phase Modulation (PM)

### 5.2.3.1 기본식과 변조지수

PM은 메시지가 **순시 위상**에 비례:
$$
s_{\text{PM}}(t) = A_c\cos\big(2\pi f_c t + k_p\, m(t)\big),
$$
- \(k_p\): 위상 감도 [rad/단위진폭]
- **최대 위상 편이** \(\Delta\theta = k_p |m(t)|_{\max}\)
- **PM 변조지수** \(\beta_p=\Delta\theta\).

### 5.2.3.2 FM vs PM 관계

메시지 \(m(t)\)에 대해
- **FM**: 위상 \(\propto \int m(t)\,dt\) (**적분**)
- **PM**: 위상 \(\propto m(t)\) (**직접**)

따라서 **\(m(t)\)를 미분/적분**하면 FM/PM을 **상호 구현** 가능:
- PM에 \( \int m(t)dt \)를 넣으면 FM과 유사
- FM에 \( \frac{d}{dt}m(t) \)를 넣으면 PM과 유사

### 5.2.3.3 대역폭(카슨 근사)

PM도 유효 대역폭을 유사하게 잡는다:
$$
B_{\text{PM}} \approx 2(\Delta f_{\text{eq}} + B)\quad
\text{(동등한 주파수 편이로 환산)},
$$
실무적으로는 **FM과 동일 형태의 근사**를 많이 사용:
$$
B_{\text{PM}} \approx 2(1+\beta_p)\,B
$$
(문헌에 따라 정의 차이가 있으므로 **정의된 \(\beta_p\)**와 **메시지 스케일**을 명확히 둘 것.)

### 5.2.3.4 파이썬 PM 실습(학습용)

```python
import numpy as np

def pm_mod(m, fs, fc, Ac=1.0, kp=np.pi/2):
    # kp: 최대 위상 편이(라디안) ~ pi/2 등
    t = np.arange(len(m))/fs
    phi = 2*np.pi*fc*t + kp*m
    return Ac*np.cos(phi)
```

---

## 5.2.4 AM/FM/PM 비교 요약

| 항목 | AM(DSB-FC) | DSB-SC | SSB | FM | PM |
|---|---|---|---|---|---|
| 점유 대역폭 | \(2B\) | \(2B\) | \(B\) | \(\approx 2(\Delta f + B)\) | \(\approx 2(1+\beta_p)B\) 근사 |
| 검파 | 포락선(간단) | 동기 | 동기 | 주파수/위상(한계기) | 위상 |
| 잡음 내성 | 진폭잡음 취약 | 동일 | 동일 | 진폭잡음에 강함 | FM와 유사 |
| 효율(전력) | 캐리어 낭비 큼 | 캐리어 없음 | 가장 효율적 | 전력 효율 양호 | 유사 |
| 구현 복잡도 | 낮음 | 중 | 높음 | 중~높음 | 중~높음 |
| 대표 용도 | AM 라디오 | 링크/스튜디오 | 장거리/군용 | 방송 FM/무전 | 디지털 PSK의 기반 개념 |

> **실전 팁**
> - **대역이 귀하면**: **SSB** 또는 **협대역 FM/PM**
> - **잡음/페이딩 강하면**: **FM/PM**(+ 한계기/디엠퍼시스)
> - **저가 간단 수신기**: **AM(포락선 검파)**
> - 아날로그 TV: **영상(VSB-AM)** + **음성(FM)**의 혼합

---

## 5.2.5 “라디오” 관점의 다중화/채널 계획

- 각 방송국은 **서로 다른 \(f_c\)**를 할당(FDM).
- 수신기는 원하는 \(f_c\) 근방만 **동조(tuning)**하여 IF(중간주파)로 하향변환 → 검파.
- **채널 간섭** 방지를 위해 **채널 간격**(AM MW: 9/10 kHz, FM: 200 kHz 등)을 표준화.

---

## 5.2.6 미니 실험: 동일 메시지로 AM·FM·PM 생성(학습용)

```python
import numpy as np

fs, fc = 200_000, 20_000
dur = 0.02
t = np.arange(int(fs*dur))/fs

# 메시지(복합 톤): 1 kHz + 2 kHz
m = 0.6*np.sin(2*np.pi*1_000*t) + 0.3*np.sin(2*np.pi*2_000*t)
m = m/np.max(np.abs(m))  # 정규화(|m|<=1)

# AM (μ=0.8), FM (Δf≈5 kHz), PM (Δθ≈π/3)
s_am = 1.0*(1+0.8*m)*np.cos(2*np.pi*fc*t)
phi_fm = 2*np.pi*fc*t + 2*np.pi*5_000*np.cumsum(m)/fs
s_fm = np.cos(phi_fm)
phi_pm = 2*np.pi*fc*t + (np.pi/3)*m
s_pm = np.cos(phi_pm)

# 여기서 s_am/s_fm/s_pm을 스펙트럼 비교(FFT)하면 각 대역 점유/사이드밴드 특성이 보인다.
```

**관찰 포인트**
- AM: \(f_c\pm\{1k,2k\}\)에 측파대, 총 대역폭 ~ \(2\times 2\,\text{kHz}=4\,\text{kHz}\)
- FM: \(\Delta f=5\text{kHz}\), \(B\approx 2(5k+2k)=14\text{kHz}\) 근방
- PM: \(\Delta\theta=\pi/3\)일 때 FM과 유사 폭(메시지·스케일에 의존)

---

## 5.2.7 체크 문제

1) **AM**에서 \(B=6\,\text{kHz}\). DSB-FC와 SSB의 점유 대역폭은?
   **해**: \(B_{\text{AM}}=12\text{kHz}\), \(B_{\text{SSB}}=6\text{kHz}\).

2) **FM**: \(\Delta f=50\text{kHz}\), \(B=10\text{kHz}\). 카슨 근사 대역폭?
   **해**: \(B\approx 2(50k+10k)=120\text{kHz}\).

3) **효율**: AM(DSB-FC) \(\mu=1\)일 때 측파대 전력 비율 \(\eta\)?
   **해**: \(\eta=\frac{1/2}{1+1/2}=1/3\approx 33\%\).

4) **PM↔FM 등가**: PM 변조기 입력에 \(\int m(t)dt\)를 넣으면 어떤 변조가 되는가?
   **해**: FM과 동등한 효과(위상 파라미터가 적분된 메시지에 비례).

5) **방송 FM 채널 간격**이 200 kHz인 이유를 카슨 법칙과 오디오/서브캐리어 구성(스테레오, RDS 등) 관점에서 개략 설명하라.

---

## 5.2.8 핵심 요약

- **AM**: 간단/저가 수신, 대역 \(2B\), **캐리어 낭비**. DSB-SC/SSB/VSB로 효율/대역 개선.
- **FM**: **진폭 잡음에 강함**, 카슨 \(2(\Delta f + B)\), 방송·무전에 적합.
- **PM**: 메시지→**위상** 직접 제어, FM과 미분/적분 관계, 대역 근사 동일형.
- 운용 선택: **대역/잡음/수신기 복잡도**와 **규제/표준**을 동시에 만족하는 지점에서 결정.
