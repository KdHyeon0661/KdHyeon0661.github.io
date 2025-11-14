---
layout: post
title: 데이터 통신 6장 - Bandwidth Utilization (2)
date: 2024-07-25 19:20:23 +0900
category: DataCommunication
---
# Bandwidth Utilization: Multiplexing and Spectrum Spreading

## Spread Spectrum (SS, 확산 대역 방식)

**핵심 요약**
- 원 신호의 대역폭 \(B\)를 **의도적으로 크게** \(B_{\text{ss}}\)로 확장(Spread)하여 **간섭/도청/페이딩**에 강하고, **다중접속(MA)**을 가능하게 하는 전송 기법.
- 초기에는 **군용(재밍·도청 방지)**에서 출발했지만, 상용 **무선 LAN/셀룰러/위성** 등에서 널리 활용.
- 확산의 대가로 **대역폭**을 더 쓰지만, **처리이득(Processing Gain)**으로 **SNR/간섭 내성**을 보전·향상한다.

### 공통 수학 & 직관

- **스프레딩 팩터(SF)**:
  $$
  \text{SF} \triangleq \frac{B_{\text{ss}}}{B}
  $$
  DSSS에선 \(\text{SF} \approx \frac{R_c}{R_b}\) (칩율/데이터율), FHSS에선 평균 점유 대역 대비 홉 집합 크기 관점으로 해석.

- **처리이득(Processing Gain, \(G_p\))**:
  $$
  G_p \approx \text{SF} = \frac{B_{\text{ss}}}{B}
  $$
  (dB 표현: \(G_p\,[\text{dB}] \approx 10\log_{10} \text{SF}\))
  → 협대역 간섭이 수신단 상관/복원에서 **Spread-out**되어 평균화(전력 밀도 희석).

- **확산 코드 요구 사항**:
  - **긴 주기**, **좋은 자기·상호 상관 특성**(좁은 피크/낮은 사이드로브), **의사난수성**
  - DSSS: PN/골드/카사미/바커 등, FHSS: 홉셋·홉시퀀스

---

## Frequency Hopping Spread Spectrum (FHSS)

**아이디어**: **반송 주파수**를 **의사난수 시퀀스**에 따라 **빠르게 바꿔가며(홉핑)** 전송.
한순간엔 좁은 대역만 쓰지만, **시간에 걸쳐** 넓은 대역을 탐색하므로 **간섭 회피·보안성·다중접속** 장점.

### 용어 & 파라미터

- **홉셋(hop-set)**: 사용 가능한 주파수 리스트 \(\{f_1,\dots,f_M\}\)
- **홉 간격 \(\Delta f\)**: 인접 주파수 간격(가드 포함)
- **정주시간(Dwell time)** \(T_h\): 한 주파수에 머무는 시간
- **홉율(hop rate)** \(R_h = 1/T_h\)
- **Slow FH / Fast FH**: 한 비트에 여러 홉(빠름) vs 여러 비트에 한 홉(느림) 반대로 쓰는 경우가 있으므로, **정주시간과 심볼시간의 상대관계**로 명시하는 것이 안전

### 동작 개념

1. **송신부**:
   - PRNG(의사난수 발생기)가 매 홉 구간마다 **k-bit 패턴** 생성 → **주파수 테이블**에서 \(f_i\) 선택
   - 선택 주파수로 **변조**(대개 FSK/PSK/QAM 가능) 후 전송
2. **수신부**:
   - 동일 키/시드로 **동일 홉 시퀀스** 복원(동기)
   - 해당 주파수 성분만 복조 → 심볼 검출

### 다중접속(FHMA)

- 홉셋을 **서로 다른 시프트/시퀀스**로 사용하여 **여러 링크가 동일 \(B_{\text{ss}}\)**를 공유
- **충돌 확률**(동일 슬롯에서 같은 \(f_i\) 선택)이 낮을수록 성능 우수
  - 이상화하여, \(M\)개 후보 중 균등 선택 시 두 링크 간 충돌 확률 \(\approx 1/M\)

### 대역폭 감각

- 전체 확산 대역:
  $$
  B_{\text{ss}} \approx M \cdot \Delta f
  $$
- 한 순간에 쓰는 대역: 변조 스펙에 따르는 **좁은 대역**(예: FSK 톤폭)

### 장단점 요약

- **장점**:
  - 특정 협대역 간섭을 **회피**(해당 순간 피하면 됨)
  - **주파수 다양성**으로 페이딩에 강함
  - **보안성**(시퀀스 없이는 추적/복조 어려움)
- **단점**:
  - 동기 복잡도(시간·주파수·시퀀스)
  - 규제/스캔 시간 제약, 스펙트럼 마스크 준수 필요
  - 고속화 시 PLL/주파수 합성기 성능 요구

### FHSS 미니 코드(개념)

```python
import numpy as np

def prng(seed, length):
    np.random.seed(seed)
    return np.random.randint(0, 256, size=length, dtype=np.uint8)

def hop_sequence(hopset_size, seed, num_hops):
    rnd = prng(seed, num_hops)
    return [int(x % hopset_size) for x in rnd]

# 예: M=79(예시), T_h=625us 등은 시스템별 상이. 여기선 시퀀스만 개념 확인.

M = 79
seq = hop_sequence(M, seed=2025, num_hops=16)
print("Hop indices:", seq)  # 각 인덱스가 hopset 내 주파수 선택을 의미
```

---

## Direct Sequence Spread Spectrum (DSSS)

**아이디어**: 데이터의 각 비트를 **칩열(chip sequence)**로 **직접 확산**.
데이터율 \(R_b\) 대비 칩율 \(R_c\)를 크게 하여 **확산 대역**과 **처리이득**을 확보.

### 수식과 흐름

- 데이터 비트열 \(b[n]\in\{+1,-1\}\)
- 칩 시퀀스 \(c[k]\in\{+1,-1\}\), 길이 \(L=\) **칩 길이**(=SF)
- 한 비트당 \(L\)개 칩: \(s[k]=b[n]\cdot c[k]\)
- **칩율** \(R_c = L\cdot R_b\)
- **확산 대역** \(B_{\text{ss}} \approx R_c\) (RRC 필터/롤오프로 스케일됨)
- **처리이득** \(G_p \approx L = \frac{R_c}{R_b}\)

### 상관 복원(수신)

- 수신 신호 \(r[k] = s[k] + n[k]\) (간섭/잡음 포함)
- 동일 코드로 상관:
  $$
  y[n] = \sum_{k=0}^{L-1} r[nL+k]\cdot c[k]
  $$
  → 의도된 비트는 **코히어런트하게 합산**, 간섭은 평균화 → **검출성 향상**

### 바커(Barker) 코드 예(무선 LAN 등 역사적 활용)

- 길이 11의 바커: \(+1,+1,+1,-1,-1,-1,+1,-1,-1,+1,-1\) (표기 편의를 위해 \(\{1,0\}\)로도 자주 씀)
- **자기상관**이 짧은 지연에서 낮아 **동기 획득**에 유리(짧은 길이일수록 처리이득은 작지만 동기·탐지에 강점)

### DSSS 파이썬(개념 구현)

```python
import numpy as np

def to_pm(bits01):
    # '0'-> -1, '1'-> +1
    return np.array([1 if b=='1' else -1 for b in bits01], dtype=np.int8)

def dsss_spread(bits_pm, chip_pm):
    L = len(chip_pm)
    return np.repeat(bits_pm, L) * np.tile(chip_pm, len(bits_pm))

def dsss_despread(rx_pm, chip_pm):
    L = len(chip_pm)
    rx_blocks = rx_pm.reshape(-1, L)
    corr = (rx_blocks * chip_pm).sum(axis=1)
    # 부호 판단
    return np.where(corr >= 0, 1, -1)

# 예시: 데이터 '10110', Barker(11)로 확산

data = "10110"
bits_pm = to_pm(data)
barker11 = np.array([+1,+1,+1,-1,-1,-1,+1,-1,-1,+1,-1], dtype=np.int8)
tx = dsss_spread(bits_pm, barker11)

# 채널에서 임의 간섭/잡음(개념) 추가

np.random.seed(0)
noise = np.random.normal(scale=2.0, size=tx.size)  # 칩단 가우시안 잡음
rx = tx + noise

# 복원

rec_pm = dsss_despread(rx, barker11)
rec_bits = ''.join('1' if v>0 else '0' for v in rec_pm)
print("TX bits:", data, "| RX bits:", rec_bits)
```

### DSSS 장단점

- **장점**:
  - 협대역 간섭에 매우 강함(상관 복원으로 **Spread-out**된 간섭 전력 희석)
  - 코드 분할 다중접속(**CDMA**) 가능(코드 상호상관이 낮을수록 간섭 적음)
  - 동기 획득/추적을 위한 **파일럿/바커/골드 코드** 활용
- **단점**:
  - **칩율↑ → 대역폭↑**, RF/아날로그/디지털 프런트엔드 부담
  - 코드 동기(수신) 복잡성, 멀티유저 간섭(MAI) 관리 필요
  - 코드 관리(보안/재사용/상호상관) 설계 중요

---

## FHSS vs DSSS — 설계 관점 비교

| 항목 | FHSS | DSSS |
|---|---|---|
| 확산 방식 | 시간축에서 **주파수 홉핑** | 비트를 **칩열로 직접 확산** |
| 순간 점유 대역 | 좁음(홉 당) | 넓음(칩율로 결정) |
| 간섭 대응 | 협대역 간섭 **회피**(홉) | 협대역 간섭 **평균화**(상관) |
| 동기 | 시간·주파수·시퀀스 동기 | 코드 동기(칩 타이밍/위상) |
| 다중접속 | FHMA (시퀀스/홉셋 분리) | CDMA (코드 분리) |
| 구현 복잡도 | 빠른 주파수 합성/PLL | 고칩율/상관기/RAKE(채널따라) |
| 처리이득 | 홉셋/점유비로 해석 | \(G_p \approx R_c/R_b\) |
| 보안성/탐지난이도 | 홉 패턴 비밀 | 코드 비밀/상관성 |

---

## 미니 실전 시나리오 & 계산

### 시나리오 A — DSSS로 1 Mbps 링크 설계

- 데이터율 \(R_b=1\ \text{Mbps}\), 칩율 \(R_c=11\ \text{Mcps}\) (바커-11 가정)
- 스프레딩 팩터 \(L=11\), 처리이득(dB)
  $$
  G_p \approx 10\log_{10}(11) \approx 10.4\ \text{dB}
  $$
- 롤오프 \(\beta\) 가진 RRC 필터를 쓰면 점유 대역폭은
  $$
  B_{\text{ss}} \approx (1+\beta)R_c
  $$
  (예: \(\beta=0.22\Rightarrow B_{\text{ss}}\approx 13.42\ \text{MHz}\) 근사)

### 시나리오 B — FHMA 충돌 확률 감각

- 홉셋 크기 \(M=50\), 두 사용자 링크가 같은 슬롯에서 **독립 균등** 선택 시
  $$
  P(\text{충돌}) \approx \frac{1}{M} = 2\%
  $$
- 동시 사용자 \(U\)에서의 기대 충돌 수/슬롯은 조합적으로 증가 → **MAC/재전송/에러정정**과 **홉 설계**로 보완

---

## 구현/동기 팁 (요약)

- **FHSS**:
  - 정밀한 **주파수 합성기**(스위칭 속도/위상 잡음), **홉 타이밍** 정확도
  - 수신부 **획득 단계**: 와이드 스캔 → 후보 홉패턴 동기 → 트래킹
- **DSSS**:
  - **코드 타이밍 추정**(슬라이딩 상관/코스·파인 서치)
  - **RAKE 수신기**: 다중경로 잔향을 **핑거**로 분할 상관 후 **최대비합성(MRC)**
  - **코드 관리**: 상호상관 낮은 시퀀스, 키·시드 보안

---

## 체크 문제

1. 처리이득 \(G_p\)가 100(=20 dB 근사)인 DSSS 시스템에서, 동일 SNR 환경에서 비확산 대비 어떤 이득이 있는지 **상관 복원 관점**으로 설명하라.
2. FHMA에서 홉셋 크기 \(M\)과 충돌 확률의 관계를 쓰고, 충돌이 **링크 처리율**에 미치는 영향을 간단 모델로 기술하라.
3. 바커-11 코드로 DSSS를 구성했을 때, 데이터율이 2 Mbps가 되면 칩율과 점유 대역폭은 어떻게 변하는가(롤오프 \(\beta\) 포함 근사).
4. DSSS 수신기에서 **동기 획득 단계**(코스 서치→파인 튜닝)와 **트래킹**의 차이를 서술하라.
5. FHSS의 **Slow/Fast** 구분을 **정주시간 \(T_h\)**과 **심볼시간 \(T_s\)**의 비교로 명확히 정리하라.

---

## 실습 코드 스니펫 — FHSS+DSSS 하이브리드(개념)

> 일부 시스템은 **하이브리드** 설계를 사용한다(예: DSSS 위에 FH, 또는 OFDM과 결합 등). 아래는 **개념**을 보여주는 소규모 실습용.

```python
import numpy as np

def to_pm(bits01):
    return np.array([1 if b=='1' else -1 for b in bits01], dtype=np.int8)

def dsss_spread(bits_pm, chip_pm):
    L = len(chip_pm)
    return np.repeat(bits_pm, L) * np.tile(chip_pm, len(bits_pm))

def awgn(x, snr_db):
    # 간단 AWGN 채널: 목표 SNR(dB)로 노이즈 추가
    p_sig = np.mean(x**2)
    snr_lin = 10**(snr_db/10)
    p_n = p_sig/snr_lin
    n = np.random.normal(scale=np.sqrt(p_n), size=len(x))
    return x + n

def hop_sequence(M, seed, num_hops):
    np.random.seed(seed)
    return [int(x % M) for x in np.random.randint(0, 255, size=num_hops)]

# 파이프라인: DSSS -> (개념적으로) 각 홉에 실음 -> 수신 상관 복원

data = "1011001110001111"
bits = to_pm(data)

# Barker-11

chip = np.array([+1,+1,+1,-1,-1,-1,+1,-1,-1,+1,-1], dtype=np.int8)
tx_dsss = dsss_spread(bits, chip)

# FHSS 개념: 구간을 나눠 홉 인덱스를 찍어보기만(실주파 변조는 생략)

M = 32
L = len(chip)
num_hops = len(tx_dsss)//L
hops = hop_sequence(M, seed=7, num_hops=num_hops)

# 각 홉 구간별로 채널 상태가 다르다고 가정(개념)

np.random.seed(1)
fading = 0.6 + 0.8*np.random.rand(num_hops)   # 0.6~1.4 배
rx = np.zeros_like(tx_dsss, dtype=float)
for i in range(num_hops):
    seg = tx_dsss[i*L:(i+1)*L].astype(float) * fading[i]
    rx[i*L:(i+1)*L] = seg

rx = awgn(rx, snr_db=5)

# DSSS 복원

rx_blocks = rx.reshape(-1, L)
corr = (rx_blocks * chip).sum(axis=1)
rec_bits_pm = np.where(corr >= 0, 1, -1)
rec_bits = ''.join('1' if v>0 else '0' for v in rec_bits_pm)
print("TX:", data)
print("RX:", rec_bits)
print("Hop indices(sample):", hops[:8])
```

---

## 결론 요약

- **FHSS**: 시간에 따라 **주파수를 바꾸며** 간섭을 **회피**, **주파수 다양성** 확보, 다중접속은 **FHMA**.
- **DSSS**: **칩열로 직접 확산**, 상관 복원으로 **협대역 간섭 평균화**·**처리이득** 확보, 다중접속은 **CDMA**.
- 실제 시스템은 **확산·다중화·코딩·이퀄라이제이션**을 조합해 목표 **BER/처리율/지연/EMI/보안**을 만족시키는 지점을 찾는다.
