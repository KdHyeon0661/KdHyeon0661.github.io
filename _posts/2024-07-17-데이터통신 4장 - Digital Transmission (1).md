---
layout: post
title: 데이터 통신 4장 - Digital Transmission (1)
date: 2024-07-17 19:20:23 +0900
category: DataCommunication
---
# 4. Digital Transmission

## 4.1 Digital-to-Digital Conversion (디지털→디지털 변환)
> 디지털 **데이터**를 디지털 **신호**(파형)로 바꾸는 절차. 전송 매체(UTP/동축/광/백플레인 등)에 맞는 파형 특성과 **클럭 복원/오류·잡음 강인성/효율/복잡도**의 균형을 맞추는 것이 목표입니다.

### 변환 분류와 본 장의 범위
- **Digital→Digital Conversion**: **Line Coding**(필수) + **Block Coding**(선택) + **Scrambling**(선택)
- (참고) Analog↔Digital, Digital→Analog, Analog→Analog은 별도 장에서 다룸

---

## 4.1.1 Line Coding (회선 부호화)
디지털 **데이터 요소**(bit)를 디지털 **신호 요소**(펄스/레벨)로 변환.

### 1) 기본 용어·수식
- **data element**: 전달하고 싶은 **비트(0/1)**  
- **signal element**: 전송 파형의 최소 단위(한 심볼/펄스)  
- **비트당 심볼 수(혹은 심볼당 비트 수)**  
  - \(r\): **신호 요소당 데이터 요소 수**(bits/signal-element)  
  - 예: 2레벨 NRZ는 \(r=1\), 4레벨(4-PAM)은 \(r=\log_2 4 = 2\)

- **데이터 전송률 \(N\)**(bit rate, bps) vs **신호 전송률 \(S\)**(symbol/baud rate, baud)
  $$
  S = \frac{N}{r}
  $$

- **평균 신호 전송률**(교재 관용 표기; 코드의 전이 패턴에 따른 **케이스 계수** \(c\))
  $$
  S_{\text{ave}} = c \cdot N \cdot \left(\frac{1}{r}\right)
  $$
  여기서 \(c\)는 **코드별 전이 밀도/스펙트럼 특성**에 좌우됩니다. (대표 예시는 아래 표 참고)

- **대역폭(BW)**: 현실적 근사로 **신호 전송률**에 비례
  $$
  B_{\min} \approx \alpha \cdot S_{\text{ave}} 
  \quad\Rightarrow\quad
  N_{\max} \approx \frac{r}{\alpha c}\, B
  $$
  (상수 \(\alpha\)는 코드/매질/필터에 의존; 실무에선 보통 **차수 근사**로 사용)

- **레벨 수 \(L\)** 와 심볼당 비트수  
  $$
  r = \log_2 L
  $$

### 2) 라인코딩이 해결해야 할 문제들
- **Baseline Wandering(기준선 변동)**  
  연속 0/1 등 불균형 패턴 → 수신단 기준(DC 오프셋)이 흔들려 임계치 판단 오류.
- **DC(직류) 성분**  
  저주파(특히 0 Hz)를 통과시키지 못하는 채널(변압 결합/커패시터 결합)에 **부적합**. DC-free/평균 0 특성이 바람직.
- **Self-Synchronization(자기 동기화, CDR)**  
  수신단이 송신단 **비트 간격**을 추적하려면 **충분한 전이(transition) 밀도** 필요. (중간 전이 보장 Manchester, RLL 제약+블록코딩, 스크램블링 등)
- **Noise/ISI 강인성**  
  전이 위치·레벨 검출의 여유 마진 필요. ISI(Inter-Symbol Interference) 억제·이퀄라이징 가능성 고려.
- **에러/폴라리티 인버전 내성**  
  AMI류의 **Differential**/위반 검출, polarity inversion 무해성 등.
- **복잡도/전력/EMI**  
  하드웨어 구현 용이성, 방사(EMI) 저감, 소비전력 등.

---

## 4.1.1.A 대표 Line Coding 패밀리 요약

| 계열 | 코드 | 레벨/전이 | DC 성분 | 동기화 | 평균 \(S_{\text{ave}}\) (대략) | 특징/비고 |
|---|---|---|---|---|---|---|
| Unipolar | NRZ (Unipolar) | 1레벨 양 방향 없음 | DC 큼 | 약함 | \( \approx N \) | 실무에서 거의 사용 안 함 |
| Polar | NRZ-L | \(\pm V\), 레벨 고정 | DC 가능 | 중간 | \( \approx \tfrac{N}{2} \) | 전이 밀도 데이터 의존 |
| Polar | NRZ-I | 1에서만 전이 | DC 가능 | 중간 | \( \approx \tfrac{N}{2} \) | 연속 1에서 전이 보장 X |
| Polar | RZ | 각 비트 중간에 0 복귀 | DC 작음 | 강함 | \( \approx N \) | 대역폭 ↑ |
| Biphase | Manchester | 비트 **중간 전이** 강제 | DC 거의 0 | 매우 강함 | \( \approx N \) | 10/100BASE-T |
| Biphase | Differential Manchester | 비트 시작 전이(클럭) + 0에서만 중간 전이 | DC 거의 0 | 매우 강함 | \( \approx N \) | 폴라리티 반전 내성 |
| Bipolar | AMI | 0=0, 1=+/− **교번** | DC 0(균형 가정) | 중간 | \( \approx \tfrac{N}{2} \) | 오류 시 위반 검출 가능 |
| Bipolar | Pseudoternary | 1=0, 0=+/− 교번 | DC 0 | 중간 | \( \approx \tfrac{N}{2} \) |  |
| Multilevel | MLT-3 | 3수준 0/±V 순환 | DC 낮음 | 중간 | \( \approx \tfrac{N}{4} \) | 100BASE-TX(4B/5B+NRZI+MLT-3) |
| RLL+Block | 4B/5B(+NRZ-I) | 5b 코드워드 | DC 낮음 | 강함 | \(\tfrac{5}{4}N\) 부호 오버헤드 | 최대 런 제한으로 CDR 용이 |
| RLL+Block | 8b/10b | DC 균형·런 제한 | DC 0 | 강함 | \(\tfrac{10}{8}N\) | PCIe, SAS, HDMI(초기) 등 |

> \(S_{\text{ave}}\)는 코드/패턴에 의존하는 **경험적 근사**입니다. “Manchester는 대략 \(S\!\approx N\)”, “NRZ는 평균 \(N/2\)” 정도의 감각으로 쓰면 실무 초기 산정에 유용합니다.

---

## 4.1.1.B 코드별 동작/예제/대역폭 감각

### (1) NRZ-L / NRZ-I (Polar)
- **NRZ-L**: 1 ↔ \(+V\), 0 ↔ \(-V\) (혹은 반대로)  
- **NRZ-I**: **1에서만 전이**(토글), 0은 유지  
- 장점: 대역폭 효율 좋음(전이 적음)  
- 단점: **DC 성분**, **런 길이** 증가 시 CDR 불안정(연속 동일 비트)

**대역폭 근사**
$$
S_{\text{ave}} \approx \frac{N}{2}, \quad B \sim \kappa \cdot S_{\text{ave}}
$$

**간단 파이썬: NRZ-I 인코딩**
```python
def nrzi_encode(bits, V=1):
    # bits: e.g., "110100"
    level = -V
    out = []
    for b in bits:
        if b == '1':
            level = V if level == -V else -V  # toggle on '1'
        out.append(level)
    return out
```

---

### (2) RZ (Return-to-Zero)
- 각 비트 기간 **중간에 0**으로 복귀 → **전이 밀도↑**로 CDR 용이  
- 단점: 대역폭 증가, 전력↑

---

### (3) Manchester / Differential Manchester (Biphase)
- **Manchester**: 한 비트 **중간 전이**(클럭 포함). `0`은 하강, `1`은 상승(또는 반대)  
- **Diff-Manchester**: 비트 **시작 전이**로 클럭, `0`일 때 **중간 전이** 추가  
- 장점: **DC-free**, **자기동기화 매우 우수**, 폴라리티 반전 강함(특히 Diff-Manchester)  
- 단점: **대역폭 ≈ N** (NRZ 대비 약 2배 전이)

**파이썬: Manchester 파형 시퀀스(샘플2/비트)**
```python
def manchester(bits, V=1):
    # sample 2 points per bit: [first_half, second_half]
    out = []
    for b in bits:
        if b == '1':
            out += [-V, +V]  # low->high
        else:
            out += [+V, -V]  # high->low
    return out
```

---

### (4) AMI / Pseudoternary (Bipolar)
- **AMI**: `0`은 0V, `1`은 +V/−V **교번**  
  - 평균 0 → **DC 성분 억제**, **위반(두 번 연속 같은 극성)**은 에러 힌트  
- **장점**: DC-free(이상적), 전이 밀도는 데이터 의존  
- **단점**: 긴 0 런에서 CDR 취약 → **스크램블/B8ZS/HDB3**와 결합

**파이썬: AMI 인코딩**
```python
def ami(bits, V=1):
    last = -V
    out = []
    for b in bits:
        if b == '1':
            last = +V if last == -V else -V
            out.append(last)
        else:
            out.append(0)
    return out
```

---

### (5) MLT-3 (Multilevel 3)
- 상태 순환: `0 → +V → 0 → −V → 0 → ...` (데이터 ‘1’일 때만 다음 상태로 진전)  
- 고주파수 성분 감소(**EMI 저감**)  
- 100BASE-TX: **4B/5B + NRZ-I + MLT-3** 파이프라인이 고전적 사례

---

## 4.1.2 Block Coding (블록 부호화)
> 원 비트열을 **고정 길이 코드워드**로 매핑하여 **런 길이 제한(RLL)**, **전이 밀도 확보**, **DC 균형**, **특수 패턴/제어 심볼** 제공.

### 대표 예: 4B/5B, 8b/10b
- **4B/5B**: 4비트 → 5비트 매핑(**25% 오버헤드**). 5b는 **1의 최소 개수/최대 0 런 제한**을 만족하는 테이블 선택 → CDR 안정. 보통 NRZ-I/MLT-3 등과 조합.
- **8b/10b**: 8비트 → 10비트(**20% 오버헤드**). DC **러닝 디스패리티** 균형, **제어 심볼(K-codes)**, **러닝 길이 제한** 모두 달성. (PCIe/SAS/FC 등)

**간단 4B/5B 일부 매핑 예**
```
0000 -> 11110
0001 -> 01001
0010 -> 10100
0011 -> 10101
...
1110 -> 11100
1111 -> 11101
```

**장단점**
- + CDR/런 제한/오류 검출(불법 코드워드)  
- − 오버헤드로 인한 **순수 유효율 감소**(Goodput↓)

---

## 4.1.3 Scrambling (스크램블링)
> **긴 0/1 런**을 해소하고 **스펙트럼 평탄화(DC/저주파 억제)**, **전이 밀도 확보**를 위해 **의사난수 LFSR**이나 **패턴 치환**을 적용.

### 1) LFSR 기반 스크램블러(디지털 회선/고속 인터커넥트)
- 랜덤같은 전이분포를 만들어 CDR/EMI/스펙트럼 측면 개선
- **송수신 양단**이 동일 LFSR로 **서로 상쇄**(descramble)

**의사코드**
```python
def lfsr_scramble(bits, poly=(7,4), seed=0b1111111):
    # x^7 + x^4 + 1 예시. XOR 탭: 7,4
    state = seed & 0b1111111
    out = []
    for b in bits:
        tap = ((state >> (7-1)) ^ (state >> (4-1))) & 1
        out_bit = b ^ tap
        out.append(out_bit)
        # shift in out_bit as new LSB
        state = ((state << 1) & 0b1111111) | out_bit
    return out
```

### 2) 패턴 치환형: B8ZS, HDB3 (T-carrier/E-carrier)
- **목적**: AMI와 결합하여 **긴 0 런**을 **의도된 바이폴라 위반** 패턴으로 치환 → **전이 삽입**  
- **B8ZS**(미국 T1): 연속 **0이 8개** 나오면 **특정 위반 패턴**으로 대체  
- **HDB3**(유럽 E1): 연속 **0이 4개**마다 **위반 패턴** 삽입(부호의 누적 균형 고려)

> 수신측은 “규칙에 맞지 않는 위반”을 보면 **치환 패턴**으로 복원.  
> 보너스: 불법 위반은 **에러 힌트**로도 이용 가능.

---

## 4.1.4 성능·스펙트럼·CDR 관점의 정량 감각

### 1) 신호율/대역폭 근사
- **목표 bit rate** \(N\)과 **코드 선택** → \(S_{\text{ave}}\) 산정 → 필요 **대역폭** 근사
  $$
  S_{\text{ave}} \approx c \cdot \frac{N}{r}, \quad B \sim \kappa\, S_{\text{ave}}
  $$
  - 예) **Manchester**: \(c\approx 1\Rightarrow S\approx \tfrac{N}{r}\). 2레벨이므로 \(r=1\) → \(S\approx N\).  
  - 예) **NRZ**: \(c\approx \tfrac{1}{2}\Rightarrow S\approx \tfrac{N}{2}\) (평균 전이)

### 2) DC 성분/스펙트럼
- 평균 레벨 0이면 DC 억제. (Manchester/AMI/8b10b 등)  
- **고주파 스펙트럼**이 적을수록 EMI 낮음(MLT-3, 신호 스무딩)

### 3) CDR/런 길이 제한
- **중간 전이 보장**(Manchester) 또는 **RLL 제약**(4B5B/8b10b) 또는 **스크램블**로 전이 밀도 확보
- **최대 런 길이**가 짧을수록 **PLL/PI**가 추적 용이

---

## 4.1.5 예제 — 코드 선택·대역폭·전이밀도

### 예제 1) 100 Mbps 링크, Manchester vs NRZ
- **Manchester**: \(r=1,\; S\approx N=100\ \text{MBd}\) → 대역폭 요구 큼, **CDR 매우 안정**  
- **NRZ**: \(S\approx N/2=50\ \text{MBd}\) → 대역폭 절감, **런 길이 위험**(추가 대책 필요)

### 예제 2) 125 Mbps 링크, 4B/5B + NRZ-I + MLT-3 (100BASE-TX)
- 4B/5B로 bit rate가 **1.25×**: 125 Mbps → 5/4 오버헤드  
- NRZ-I로 부호화 후 MLT-3로 전이 주파수 저감 → UTP에서 EMI·대역 요구 완화  
- 실무 감각: **전이 주파수**는 NRZ 대비 대략 **1/4 수준**까지 낮출 수 있음

---

## 4.1.6 “Baseline Wandering & DC” 더 깊게
- **Baseline**: 수신 파형 평균 레벨(임계치 기준선)  
- **문제**: 긴 0/1 런은 필터/수신단 기준선을 밀어내 오검출  
- **해결**:  
  - DC-free 코드(Manchester/AMI/8b10b)  
  - 런 제한/블록코딩(4B/5B, 8b/10b)  
  - 스크램블러로 스펙트럼 평탄화

**수식 관점**: 롱런은 **저주파(특히 0Hz)** 에너지를 키움 → 하이패스 결합된 채널에서 **왜곡/베이스라인 쉬프트** 유발.

---

## 4.1.7 Self-Synchronization (자기 동기화)
- **전이(transition)**가 **타이밍 정보**. 전이가 적으면 CDR이 난해.  
- **보장 방법**:
  - Manchester: **비트 중간 전이** 항상 존재  
  - RLL: 특정 길이 이상 연속 0/1 금지(4B/5B, 8b/10b)  
  - Scrambling: 통계적으로 전이 밀도 확보  
  - Differential 코딩: 폴라리티 반전 무해(전이가 **의미**)

---

## 4.1.8 오류 검출/강인성(라인코딩 관점)
- **Bipolar Violation**: AMI 계열에서 규칙 위반은 **에러 힌트**  
- **Differential**: 케이블 뒤집힘/폴라리티 반전에도 의미 불변  
- **불법 코드워드**: 8b/10b, 4B/5B에서 **예약 패턴** 검출로 에러·제어 심볼 구분  
- **스펙트럼/전이 안정**: CDR 안정 = **비트 샘플링 타이밍** 안정 → BER 저감

---

## 4.1.9 복잡도(Complexity)
- **CDR/PLL** 회로와 **이퀄라이저/DFE** 필요성  
- **엔코더/디코더**(블록/스크램블) 로직 비용과 지연  
- **전력/EMI/ESD** 고려, PCB/커넥터/케이블 채널 모델링(S-파라미터)  
- **테스트 용이성**(아이 다이어그램, 패턴 발생기/분석기)

---

## 4.1.10 실습 코드: 전이 밀도·대역폭 근감각

> 아래 코드는 몇 가지 라인코딩으로 **전이 횟수**와 **대략적 신호율**을 비교합니다. (시각화 생략)

```python
def nrz_l(bits, V=1):
    return [V if b=='1' else -V for b in bits]

def nrz_i(bits, V=1):
    level = -V; out=[]
    for b in bits:
        if b=='1': level = V if level==-V else -V
        out.append(level)
    return out

def manchester_samples(bits, V=1):
    # 2 samples/bit
    out=[]
    for b in bits:
        out += ([-V, +V] if b=='1' else [+V, -V])
    return out

def transitions(seq):
    return sum(1 for i in range(1,len(seq)) if seq[i]!=seq[i-1])

bits = "11010000111100001010101010100011110000"
w_nrz  = nrz_l(bits)
w_nrzi = nrz_i(bits)
w_m    = manchester_samples(bits)

print("NRZ-L transitions:", transitions(w_nrz))
print("NRZ-I transitions:", transitions(w_nrzi))
print("Manchester transitions:", transitions(w_m))
# 전이 수 ~ 신호 전송률에 비례 → 대역폭 감각(대략) 비교 가능
```

---

## 4.1.11 계산 예제 모음

### 예제 A) NRZ-L로 200 Mbps 달성 시 신호율·대역폭
- \(r=1\), 평균 \(S\approx N/2 = 100\ \text{MBd}\)  
- 근사 대역폭 \(B\sim \kappa \cdot 100\ \text{MHz}\)  
- 전이 밀도 데이터 의존 → CDR 안정 위해 **블록코딩/스크램블러** 고려

### 예제 B) Manchester로 200 Mbps 달성
- \(S\approx N = 200\ \text{MBd}\)  
- 대역폭 요구 ↑, 대신 **CDR 최강/폴라리티 변화 견고**  
- 케이블 짧고 클럭회로 간단히 하고 싶을 때 유리

### 예제 C) 1 Gbps UTP 설계 감각
- 순수 NRZ는 EMI/스펙트럼·런 길이 문제가 큼  
- **RLL+블록(8b/10b or 4B/5B)** + **MLT-3/스크램블**로  
  - DC/런 제한 + 스펙트럼 저주파화 + CDR 안정 → 실무적 해법

---

## 4.1.12 체크 문제

1) **신호율**: \(N=100\) Mbps, 4-PAM ( \(L=4\Rightarrow r=2\) ), 코드 계수 \(c\simeq 0.5\).  
   $$ S_{\text{ave}} \approx c\cdot \frac{N}{r} = 0.5 \cdot \frac{100}{2} = 25\ \text{MBd} $$
2) **대역폭 비교**: 같은 \(N\)에서 **Manchester** vs **NRZ** 중 어느 쪽이 대역폭 더 큰가?  
   → **Manchester**(전이 보장, \(S\approx N\) vs NRZ \(S\approx N/2\))
3) **DC 성분** 억제에 유리한 코드 2개 쓰시오. → **Manchester, AMI(균형 가정), 8b/10b**
4) **긴 0 런**을 치환하는 스크램블링 기법 이름 2개. → **B8ZS, HDB3**
5) **폴라리티 반전**에 의미가 보존되는 라인코딩은? → **Differential Manchester**, **NRZ-I(차동 해석 시)**

---

## 4.1.13 요약
- **라인코딩**은 디지털 데이터를 **매질 친화적 파형**으로 바꾸는 핵심 기술.  
- 지표: **대역폭/전이 밀도/DC 성분/런 제한/CDR 안정/EMI/복잡도**.  
- **Manchester**는 동기화/DC-free 최강(대역폭↑), **NRZ**는 효율↑(동기화 대책 필요).  
- **AMI/MLT-3/4B5B/8b10b/스크램블** 등으로 **현실 채널** 제약(저주파 차단, EMI, CDR)을 해결.  
- 설계 프로세스: **목표 N → 후보 코드 스펙트럼 → CDR/BER/EMI/복잡도**를 균형 최적화.