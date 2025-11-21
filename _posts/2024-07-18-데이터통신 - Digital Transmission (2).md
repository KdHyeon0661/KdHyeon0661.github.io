---
layout: post
title: 데이터 통신 - Digital Transmission (2)
date: 2024-07-18 19:20:23 +0900
category: DataCommunication
---
# Line Coding Schemes

## 들어가며 — 라인 코딩에서 반드시 점검할 7가지 축

1. **심볼/레벨 수**: 레벨이 \(L\)개면 심볼당 정보량은 \(\log_2 L\) 비트.
   $$ r = \log_2 L $$
2. **신호율 vs 비트율**: 비트율 \(N\) (bps), 신호율 \(S\) (baud).
   $$ S=\frac{N}{r} $$
   같은 \(N\)에서 \(r\)을 키우면(다준위) \(S\)가 줄어 대역폭 요구가 감소.
3. **대역폭 근사**: 실무에서 \(B \propto S\). 펄스 형상(직사/RC/NRZ)/필터링에 따른 상수 차이 존재.
4. **DC(0 Hz) 성분**: 평균 레벨이 0이면 DC-free. 변압/커패시터 결합 링크에서 필수.
5. **자기 동기화(CDR)**: 전이(transition) 밀도가 높을수록 클럭 복원 유리. 런 길이 제한 필요.
6. **런 길이 제한(RLL)/스크램블**: 연속 0/1을 제한하거나, 통계적 전이를 보장해 CDR 안정화.
7. **잡음/ISI/EMI/복잡도 트레이드오프**: 높은 레벨 수는 대역은 절약하지만 **검출 SNR 요구↑**. EMI 규제/하드웨어 비용도 고려.

---

## Unipolar Scheme (단극)

**정의**: 전압이 한쪽 극성(+만 혹은 0/+). 보통 교육용.

### Unipolar NRZ

- 매핑(관례 예): **1 → +V**, **0 → 0V**.
- **장점**: 구현 단순.
- **치명적 단점**
  - **DC 성분 큼**: 평균 레벨이 \(>0\). 0 Hz 차단 채널(트랜스 결합 등) 통과 불가.
    평균 레벨(긴 구간)
    $$ \bar{x} = p(1)\cdot V + p(0)\cdot 0 = p(1)\,V $$
  - **전이 밀도↓**: 긴 1/0 런에서 클럭 복원 실패.
  - **전력 비효율**: 같은 피크 전압 대비 평균전력↑.
- **실전 사용**: 사실상 **비권장**(교육/데모 용도).

**파형 생성(샘플 1/비트)**
```python
def unipolar_nrz(bits, V=1.0):
    return [V if b=='1' else 0.0 for b in bits]
```

**연습**
- 균일 랜덤 비트에서 \(p(1)=p(0)=0.5\)일 때 평균 레벨과 DC 크기를 설명하라.
- 100m 링크가 1 Hz 이하 차단이면, Unipolar가 왜 재생 불가능한지 논하라.

---

## Polar Scheme (양·음 극성)

양/음 두 극성 사용으로 평균 0 근접 설계가 쉬움.

### Polar NRZ

#### NRZ-L (NRZ-Level)

- **0 → −V**, **1 → +V** (또는 반대).
- **장점**: 전이가 적어 대역 요구가 낮음(평균적으로 \(S \approx N/2\)로 설명하곤 함 — 펄스 모양/코딩에 따라 달라짐).
- **약점**: 데이터 패턴 의존 → **런 길이/기준선 변동(Baseline Wandering)/DC 가능성**.

#### NRZ-I (NRZ-Invert)

- **비트 시작의 전이 유무가 데이터**: 전이=1, 무전이=0.
- **장점**: 폴라리티 반전(케이블 뒤집힘)에 상대적으로 강함(차동 수신에서 이점).
- **약점**: **긴 0 런**에서 전이 사라져 CDR 취약.

```python
def nrzi(bits, V=1.0):
    level = -V
    out = []
    for b in bits:
        if b == '1':  # 1이면 전이
            level = +V if level == -V else -V
        out.append(level)
    return out
```

### RZ (Return-to-Zero)

- 비트 중간에 **0 복귀**(예: +V/2 동안 신호 후 0으로).
- **장점**: **비트 내 전이**로 CDR 우수, 평균 레벨 0 근접.
- **단점**: **신호율↑/대역↑/전력↑**.

**분석 포인트**
- RZ의 스펙트럼은 DC 근처 저감 및 주기적 노치가 생기며, NRZ 대비 고조파 성분이 증가. 필터 설계와 BER을 함께 봐야 함.

---

## Biphase (Manchester / Differential Manchester)

비트 **중간 전이**를 의무화해 **CDR 최강**, 평균 0.

### Manchester (IEEE 802.3 관례)

- **0: High→Low**, **1: Low→High** (비트 중간 전이 항상 존재).
- **DC-free**: 랜덤 데이터 기준으로 평균 0.
- **대역 요구↑**: 동일 \(N\)에서 전이 많음 → \(S \approx N\).

```python
def manchester(bits, V=1.0):
    out=[]
    for b in bits:
        out += ([-V, +V] if b=='1' else [+V, -V])
    return out
```

**수식 관찰**
- 랜덤 데이터에서 평균
  $$ \bar{x} \approx 0 $$
- **전이 확률** 거의 1에 수렴 → CDR 안정.

### Differential Manchester

- **비트 시작 전이=클럭**(항상 존재).
- 데이터 규칙(관례 중 하나): **0 → 중간 전이**, **1 → 중간 무전이**.
- **장점**: 폴라리티 반전에 무해, DC 거의 0, CDR 매우 우수.
- **단점**: 대역폭 요구는 Manchester와 유사하게 큼.

**왜 CDR이 강한가**
- 비트마다 최소 1회 전이(시작) + (데이터에 따른 중간 전이) → PLL이 손쉽게 위상 동기.

---

## Bipolar Scheme (삼전위: +V/0/−V)

**핵심**: **1**(또는 0)을 **+/−로 번갈아** 보내 평균 0 유지.

### AMI (Alternate Mark Inversion)

- **1 → +V, −V 교번**, **0 → 0V**.
- **장점**: 평균 레벨≈0, DC-free 성질, **Bipolar Violation(BV)**로 **에러 힌트/동기화 힌트** 제공 가능.
- **단점**: **긴 0 런**에서 전이 부족 → CDR 취약.

```python
def ami(bits, V=1.0):
    last = -V
    out=[]
    for b in bits:
        if b=='1':
            last = +V if last==-V else -V
            out.append(last)
        else:
            out.append(0.0)
    return out
```

### Pseudoternary

- AMI와 반대로 **0이 ±V 교번**, **1은 0V**.
- 사용 빈도는 낮으나, 원리는 동일.

### AMI에서 긴 0 런 해결 — B8ZS/HDB3

- **B8ZS(T1)**: **0이 8개** 연속 시, **2개의 위반 패턴 삽입(BV)** → 수신측이 위반을 감지해 원 데이터 복원.
- **HDB3(E1)**: **0이 4개**마다 위반 삽입. **부호 균형(폴라리티 누적)**을 고려해 +/− 패턴을 선택.
- **효과**: 전이 밀도 보장 → CDR/동기화 회복, DC-free 유지.

**개념적 치환 예시(B8ZS)**
- 원래 비트열: `... 1 0 0 0 0 0 0 0 0 1 ...`
- `00000000` 구간을 `000VB0VB`같은 패턴으로 치환(V/B는 위반 기호) — 실제 표준의 부호 규칙을 따른다.

---

## — 2B1Q / 8B6T / 4D-PAM5

### 매핑 조건과 여분 패턴 활용

- \(m\) 데이터 비트 → \(2^m\) 패턴, \(n\) 신호 요소/레벨 \(L\) → \(L^n\) 패턴.
  $$ 2^m \le L^n $$
- **여분** \(L^n-2^m\)은 **런 제한/전이 보장/에러 검출/균형** 등에 사용.

### 2B1Q (Two Binary, One Quaternary, PAM-4)

- **2비트 → 1 심볼(4레벨)**: 예 \(-3,-1,+1,+3\).
  $$ r=2 \Rightarrow S = \frac{N}{2} $$
- **장점**: 같은 \(N\)에서 **신호율 1/2**, 대역 절약.
- **단점**: 레벨 간격 좁아져 **SNR 요구↑**. DC 성분은 매핑/프리코딩/스펙트럼 형성에 따라 달라짐.
- **적용**: ISDN U-Interface, 일부 xDSL 전송에서 채택 사례.

```python
MAP_2B1Q = {"00":-3,"01":-1,"11":+1,"10":+3}  # 예시 매핑
def encode_2b1q(bitstr):
    assert len(bitstr)%2==0
    return [MAP_2B1Q[bitstr[i:i+2]] for i in range(0,len(bitstr),2)]
```

### 8B6T (Eight Binary, Six Ternary)

- **8비트(256)** → **3진 6심볼(729)**.
- **여분 473 패턴**으로 **전이 밀도/런 제한/평균 0** 특성을 가지는 **테이블 설계**.
- **장점**: 스펙트럼 제어/동기화에 유리.
- **단점**: **테이블 설계/복호화 복잡**, 3레벨 검출은 **SNR 더 요구**.

### 4D-PAM5 (Four-Dimensional, Five-Level PAM)

- **기가비트 이더넷(1000BASE-T)**의 핵심.
- **4차원(4 페어 동시), 각 차원 5레벨 \(\{-2,-1,0,+1,+2\}\)**.
- **트렐리스 코딩, 에코/크로스토크 상쇄, 이퀄라이제이션, FEC** 결합 → **125 MBd로 1 Gbps** 달성.
- **의미**: 단순 라인코딩을 넘어 **시스템 레벨 신호처리**와 결합되는 현대 PHY의 전형.

**핵심 수식 감각**
- 한 페어당 심볼율 \(125\) MBd, 공간 다중화(4D)와 코딩 이득으로 총 \(1\) Gbps.
- 레벨 수 증가에 따른 **최소 신호간 거리 \(d_{min}\)** 감소 →
  $$ \text{BER} \approx Q\!\left(\frac{d_{min}}{2\sigma}\right) $$
  여기서 \(\sigma^2\)는 잡음 분산. 레벨이 많을수록 \(d_{min}\)이 작아지고 더 높은 SNR 필요.

---

## Multi-transition: MLT-3

- **상태 순환**: \(-V \rightarrow 0 \rightarrow +V \rightarrow 0 \rightarrow -V \rightarrow \cdots\)
- **입력 ‘1’에서만 다음 상태로 이동**, ‘0’이면 유지.
- **효과**: 고주파 성분 대폭 감소(EMI 완화).
- **최대 전이 주파수**: 이상적 랜덤에서 **\(f_{\max} \approx N/4\)** 에 수렴(‘1’을 2번 받아야 같은 극성 피크 재방문).

```python
def mlt3(bits, V=1.0):
    states = [-V, 0.0, +V, 0.0]
    idx = 0
    out = []
    for b in bits:
        if b == '1':
            idx = (idx + 1) % 4
        out.append(states[idx])
    return out
```

**실전**: **100BASE-TX**에서 **4B/5B → NRZ-I → MLT-3** 파이프라인으로 라인 주파수 약 **31.25 MHz**(@125 Mbps) 달성(EMI/대역/복잡도 균형점).

---

## 스펙트럼(정성), DC, 전이, CDR — 종합 비교

| 계열 | 대표 | 평균(DC) | 전이 밀도/동기화 | 대역 감각(근사) | 장단점 요약 |
|---|---|---|---|---|---|
| Unipolar | NRZ | DC 큼 | 취약 | \(S\!\approx\!N\) | 단순하나 실전 비권장 |
| Polar | NRZ-L/I | 패턴 의존 | 중간(런 발생) | \(S\!\approx\!N/2\) | 대역 효율↑, CDR 보완 필요 |
| Polar | RZ | 0 근접 | 강 | \(S\!\approx\!N\) | CDR↑/대역↑ |
| Biphase | Manchester/DM | 0 | **최강** | \(S\!\approx\!N\) | 확실한 동기, 대역↑ |
| Bipolar | AMI | 0 | 중간 | \(S\!\approx\!N/2\) | B8ZS/HDB3로 런 해결 |
| Multilevel | 2B1Q/8B6T | 설계 의존 | 중~강 | \(S\le N\) | 대역↓vs SNR/복잡도↑ |
| Multi-trans | MLT-3 | 낮음 | 중간 | \(f_{\max}\!\approx\!N/4\) | EMI↓, 100BASE-TX 핵심 |

---

## 스크램블·블록코딩·RLL과의 결합

### 스크램블러(예: LFSR)

- **목표**: 데이터 통계에 상관없이 **전이/스펙트럼**을 평탄화 → **CDR/EMI 개선**.
- **예시**: 100BASE-TX의 **4B/5B**(런 제한) + **스크램블**로 패턴 의존성 완화.

```python
def lfsr_scramble(bits, taps=(7,4), seed=0b1111111):
    # 간단 예시(LFSR 길이 7). taps는 x^7 + x^4 + 1 형태.
    state = seed & 0x7f
    out=[]
    for b in bits:
        fb = ((state>>6) ^ (state>>3)) & 1  # 7,4 탭 XOR
        sbit = fb ^ int(b)  # 데이터 XOR
        out.append('1' if sbit else '0')
        state = ((state<<1)&0x7e) | fb
    return ''.join(out)
```

### 블록코딩(4B/5B, 8b/10b 등)

- **런 길이 상한 보장**(예: 4B/5B는 최대 3~4개의 연속 0만 허용), **DC 균형**, **제어 심볼** 제공.
- PCIe/파이버채널 등은 **8b/10b**, 최근엔 **64b/66b**(오버헤드↓)도 다수.

---

## 실전 PHY 파이프라인 예

### 100BASE-TX (Cat5 UTP)

- **PCS**: 4B/5B 블록코딩 → NRZ-I
- **PMA/PMD**: **MLT-3** 라인코딩, 아날로그 프런트엔드
- **효과**: 대역을 31.25 MHz 수준으로 억제, EMI/케이블 호환성 확보.

### 1000BASE-T

- **라인코딩**: **4D-PAM5**
- **DSP**: 에코/크로스토크 상쇄, 채널 이퀄라이저, 타이밍 복원, FEC
- **효과**: 125 MBd로 1 Gbps 달성(UTP 상).

---

## 계산/연습 문제 (해설 힌트 포함)

### Q1. Manchester의 DC-free 증명(평균)

랜덤 \(p(1)=p(0)=0.5\)에서 한 비트 평균을 구하라.
**힌트**: 절반은 \(+V\to -V\), 절반은 \(-V\to +V\). 평균 0.

### Q2. NRZ-I에서 긴 0 런이 CDR을 깨는 이유

**힌트**: 0에서 전이가 없으니 PLL 입력에 에지가 사라짐. 스크램블/RLL로 방지.

### Q3. 2B1Q의 신호율

\(r=2\)이므로 \(S=N/2\). 동일 채널에서 NRZ 대비 대역 이득은?
**힌트**: 대역 근사 \(B \propto S\).

### Q4. MLT-3의 최대 전이 주파수

왜 \(N/4\)에 수렴하는가?
**힌트**: 상태가 4개이고 ‘1’에서만 다음 상태로 이동.

### Q5. AMI+B8ZS

8개의 0이 나타났을 때 수신기가 위반 패턴을 어떻게 복원하는가?
**힌트**: 불가능한 극성 패턴(위반)을 신호로 사용.

---

## 실습 코드 번들 — 전이수/평균레벨/간단 비교

```python
def transitions(seq):
    return sum(1 for i in range(1,len(seq)) if seq[i]!=seq[i-1])

def avg_level(seq):
    return sum(seq)/len(seq)

bits = "11010000111100001010101000011110000"

w_uni  = unipolar_nrz(bits)
w_nrzi = nrzi(bits)
w_man  = manchester(bits)
w_ami  = ami(bits)
w_mlt3 = mlt3(bits)

print("Transitions:")
print("  Unipolar NRZ :", transitions(w_uni))
print("  NRZ-I        :", transitions(w_nrzi))
print("  Manchester   :", transitions(w_man))
print("  AMI          :", transitions(w_ami))
print("  MLT-3        :", transitions(w_mlt3))

print("Average level (DC clue):")
print("  Unipolar NRZ :", avg_level(w_uni))
print("  NRZ-I        :", avg_level(w_nrzi))
print("  Manchester   :", avg_level(w_man))
print("  AMI          :", avg_level(w_ami))
print("  MLT-3        :", avg_level(w_mlt3))
```

---

## 결론 — 선택의 기준을 한 장으로

- **CDR 최강/간단한 수신**: Manchester/DM
- **대역 절약**: 다준위(PAM-4/5/…) + 블록코딩 + 이퀄라이저
- **EMI 억제/중간 주파수로 타협**: MLT-3
- **DC-free + 런 해결**: AMI + B8ZS/HDB3, 8b/10b, 스크램블
- **현대 PHY**: 라인코딩 + DSP + 채널 보정(에코/크로스토크/이퀄/프리코딩) **통합 설계**

이 모든 결정은 **케이블/커넥터/주파수 응답/SNR/규제/비용/목표 BER**을 함께 놓고 **시스템 레벨에서 최적화**해야 합니다.
