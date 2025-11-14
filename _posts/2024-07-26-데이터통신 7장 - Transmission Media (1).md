---
layout: post
title: 데이터 통신 7장 - Transmission Media (1)
date: 2024-07-26 19:20:23 +0900
category: DataCommunication
---
# Transmission Media

## Transmission Media Introduction

- **Transmission medium(매질)**: 송신자와 수신자 사이에서 **전기적/광학적 에너지**로 정보를 전달하는 모든 물리적 통로.
- 분류
  - **Guided(유도) 매체**: 전파가 **도체/유전체 경로**에 **구속**됨(예: Twisted-Pair, Coaxial, Fiber-Optic).
  - **Unguided(비유도) 매체**: 자유공간을 **안테나**로 방사(예: 무선, 위성, 마이크로웨이브). *(본 장에서는 유도 매체 중심)*

**핵심 지표**
- **대역폭**(Hz/GHz): 통과 가능한 주파수 범위.
- **감쇠(Attenuation)**: 거리당 신호 전력 손실(보통 dB/km).
- **지연(Delay)**: 전파 속도로 결정되는 전파 지연 + 매질/회로 처리 지연.
- **잡음/누화(EMI/Xtalk)**: 외생 간섭 저항성.
- **임피던스 정합**: 반사/반사계수/VSWR.
- **보안/탐침 난이도**: 도청 용이성/탐침 난이도.
- **설치/운영 비용**: 케이블/커넥터/시공/장비 복잡도.

---

## Guided Media (유도 매체)

금속 매체는 **전류/전계**로, 광섬유는 **광파**로 정보를 운반한다.

### Twisted-Pair Cable (연선, UTP/STP)

**구조**: 두 가닥의 구리선(Conductor)을 규칙적으로 **트위스트**하고 절연(Insulator).
**목적**: 동일 페어의 반대 전류가 **상쇄**되어 **EMI/근단누화(NEXT)** 억제.

#### UTP vs STP

- **UTP**: 차폐가 없고 **저비용/유연성**이 크며, 오늘날 **이더넷**의 표준 매체.
- **STP/S-FTP 등**: **포일/브레이드 차폐**로 누화·외부침투 억제 ↑, 그러나 **비용·접지·시공 난이도** ↑. (특정 환경/레거시 IBM Token Ring 등에서 활용)

#### 카테고리(예시적 감각)

- **Cat5e**: 1 Gbps(최대 100 m),
- **Cat6**: 1 Gbps(100 m), 10 Gbps(55 m 근방),
- **Cat6a**: 10 Gbps(100 m),
- **Cat7/7a/8**: 차폐 중심, 25/40 Gbps(데이터센터 단거리).
> 실제 성능은 **케이블/커넥터 품질, 설치 품질(분리, 굴곡, 접지), 채널 구성**에 강하게 좌우됨.

#### 커넥터

- **RJ-45(8P8C)**: 이더넷/PoE 표준.
- 핀 배열은 **T568A/B**; 혼용 시 크로스/페어 뒤틀림 주의.

#### 성능 & 손실 모델

- **감쇠(대략)**:
  $$
  \alpha_{\text{UTP}}(f)\ \text{[dB/m]} \approx a\sqrt{f} + b f
  $$
  (스킨 효과/유전체 손실 항. 상수는 케이블 구조/게이지에 의존)
- **전파 속도**:
  $$
  v \approx \frac{c}{\sqrt{\varepsilon_r}} \quad\Rightarrow\quad \text{NVP} \approx 0.6\sim0.8
  $$
  (NVP: Nominal Velocity of Propagation)
- **누화**: **NEXT(근단) / FEXT(원단)**, **Alien crosstalk**(다른 케이블 간) 설계 고려.

#### PoE(전력공급)

- 802.3af/at/bt 등으로 **데이터+전력** 공급.
- **DC 저항/발열/번들링** 고려 → 케이블 품질과 설치 밀도 중요.

#### 계산 예 (UTP 링크 버짓)

- 100 m Cat6 채널, 주파수 250 MHz에서 케이블 감쇠가 22 dB(예)이고 커넥터/패치 감쇠 총 3 dB라면 링크 감쇠는:
  $$
  A_{\text{link}} = 22 + 3 = 25\ \text{dB}
  $$
  PHY 송수신 **링크 마진**을 이 손실/누화/리턴로스/잡음 대비로 확인해야 함.

**실습 코드: dB 변환/누화 마진 감각**
```python
def db_add(*vals_db):  # 직렬 손실 합산
    return sum(vals_db)

def db_from_ratio(p2_over_p1):
    import math
    return 10*math.log10(p2_over_p1)

def ratio_from_db(db):
    import math
    return 10**(db/10)

# 예: 케이블 22 dB + 커넥터 3 dB

link_loss_db = db_add(22, 3)
print("Link loss:", link_loss_db, "dB")
```

---

### Coaxial Cable (동축)

**구조**: 중심 도체—유전체—외도체(브레이드/포일)—재킷.
**특징**
- **차폐 우수**, 비교적 **넓은 대역**, **정합 용이(특성 임피던스 50Ω/75Ω)**.
- **마이크로웨이브/케이블TV/계측** 등에서 널리 사용.

#### 규격 & 커넥터

- **RG-58(50Ω)**: 예전 10BASE2 등,
- **RG-59(75Ω)**: CATV,
- **RG-6(75Ω)**: CATV/위성.
- 커넥터: **BNC**(측정/방송), **F-type**(CATV), **N/SMA**(RF/마이크로웨이브).

#### 손실/대역

- 감쇠(근사):
  $$
  \alpha_{\text{coax}}(f) \approx A\sqrt{f} + B f \quad[\text{dB/m}]
  $$
  고주파로 갈수록 급격히 증가 → **리피터/증폭기** 필요성 ↑.

#### 임피던스 정합

- 소스/라인/부하 임피던스가 **불일치**하면 **반사** 발생.
- 반사계수/VSWR:
  $$
  \Gamma = \frac{Z_L - Z_0}{Z_L + Z_0},\qquad \text{VSWR}=\frac{1+|\Gamma|}{1-|\Gamma|}
  $$
  정합( \(Z_L=Z_0\) )일수록 반사↓, 유효 전력 전달↑.

**실습 코드: 반사/VSWR**
```python
def vswr(zl, z0=50.0):
    gamma = (zl - z0) / (zl + z0)
    g = abs(gamma)
    return (1+g)/(1-g)

print("VSWR for ZL=75Ω on 50Ω line:", round(vswr(75,50),3))
```

---

### Fiber-Optic Cable (광섬유)

**핵심**: 빛(레이저/LED)을 **유전체 파이프(코어+클래드)** 내부의 **전반사**로 유도.

#### 구조

- **Core(코어)**: 굴절률 \(n_1\), 직경이 작을수록(단일모드) 모드 수 감소.
- **Cladding(클래드)**: 굴절률 \(n_2<n_1\)
- **Coating/Jacket/Strength Members**: 보호/인장 보강.

#### 수치개구(NA) & 임계각

- 전반사 조건(입사각 \(\theta\), 임계각 \(\theta_c\)):
  $$
  \sin \theta_c = \frac{n_2}{n_1},\qquad \text{NA}=\sqrt{n_1^2 - n_2^2}
  $$
  NA가 클수록 **수광 각도** ↑(접속 쉬움), 대신 모드 수 ↑(다중모드).

#### 전파 모드

- **Multimode (MMF)**:
  - **Step-index**: 계단형 굴절률 → **모드 분산** 큼(대역폭 ↓).
  - **Graded-index (GI)**: 중심에서 높은 굴절률 → **속도 보정**으로 분산 낮춤.
  - 일반적으로 짧은/중거리(수백 m ~ 수 km), **850/1300 nm** 창 사용.
- **Singlemode (SMF)**:
  - 코어 ~ 8~10 μm, **1310/1550 nm** 창, **장거리/고비트**에 최적.
  - 제약은 **색분산(크로매틱)**, **편광 모드 분산(PMD)**.

#### 감쇠/창

- **감쇠**:
  $$
  \alpha_{\text{fiber}} \approx 0.2\ \text{dB/km @ 1550 nm} \quad(\text{현대 SSMF, 실측에 좌우})
  $$
- **창(Window)**: 850/1310/1550 nm 등 **손실/증폭기/디스퍼전** 트레이드오프.

#### 분산(Dispersion)

- **크로매틱 디스퍼전**: 파장별 군지연 차이 → **펄스 확대**
  (파장대역 \(\Delta \lambda\), 길이 \(L\), 계수 \(D\,[\text{ps}/(\text{nm}\cdot \text{km})]\))
  $$
  \Delta T \approx D\cdot \Delta \lambda \cdot L
  $$
- **PMD**: 편광 모드 간 지연 차이 → 고속 시 성능 저하.

#### 커넥터/종단

- **SC/ST/LC/MT-RJ** 등. 폴리싱(PC/UPC/APC), **삽입손실/반사손실** 관리.

#### 링크 버짓(예)

- 송신 파워 \(P_T=0\ \text{dBm}\) (1 mW), 수신 감도 \(P_R=-20\ \text{dBm}\).
- **허용 손실** \(=20\ \text{dB}\).
- 광섬유 50 km @0.2 dB/km → 선로 손실 **10 dB**.
- 커넥터/스플라이스 10개×0.3 dB = **3 dB**.
- 마진 3 dB 가정 → 총 16 dB < 20 dB ⇒ **링크 가능(여유 4 dB)**.

**링크 버짓 코드**
```python
def link_budget_db(tx_dbm, rx_sens_dbm, span_losses_db, margin_db=0.0):
    avail = tx_dbm - rx_sens_dbm
    req = span_losses_db + margin_db
    return avail - req  # >0 이면 여유(dB)

# 예: 0 dBm → -20 dBm, 선로 10 dB + 접속 3 dB, 마진 3 dB

delta = link_budget_db(0, -20, 13, margin_db=3)
print("링크 마진(dB):", delta)
```

#### 광의 장단점

- **장점**: 초고대역/장거리, 낮은 감쇠, EMI 면역, 가볍고 도청 난이도↑.
- **단점**: **시공/접속 난이도**, 단일방향(일반적으로 Tx/Rx 별도), 광 부품/장비 비용.

---

## 매체 선택 가이드 (실전 감각)

### 시나리오 A: 사무실 플로어 90 m, 1/10 Gbps

- **Cat6a UTP** 권장(10G, 100 m 지원), PoE 필요 시 케이블 품질/온도/번들링 고려.
- 간섭 심한 공장 환경/고전력 인접 시 **F/UTP, S/FTP** 등 차폐형 + 접지 설계.

### 시나리오 B: 빌딩 간 2~5 km 백본

- **SMF(1310/1550 nm)** + 광 SFP(또는 QSFP) → 낮은 감쇠/높은 신뢰성.
- WDM(DWDM/CWDM)로 파장 다중화하여 회선 확장 용이.

### 시나리오 C: 케이블 TV/광동축 혼합(HFC)

- **광 트렁크 + 코액스 분배**: 광 구간으로 장거리 저감쇠, 마지막 마일은 동축.

---

## 수식/계산 모음

### dB 변환

- **전력비**:
  $$
  L_{\text{dB}} = 10\log_{10}\!\left(\frac{P_2}{P_1}\right)
  $$
- **전압비(동일 임피던스)**:
  $$
  L_{\text{dB}} = 20\log_{10}\!\left(\frac{V_2}{V_1}\right)
  $$

### 전송 지연

- 거리 \(d\), 전파속도 \(v\)일 때
  $$
  t_p = \frac{d}{v}
  $$
  구리/연선은 \(v\approx 2\times 10^8\ \text{m/s}\) 내외, 광섬유는 \( \sim 2\times10^8\ \text{m/s} \) 수준(굴절률에 따라).

### 대역폭·감쇠 트레이드오프(직관)

- 금속 매체: \( \alpha(f)\uparrow \) with \( \sqrt{f} \) (스킨 효과) + \(f\) 항(유전 손실).
- 광: 감쇠 낮으나 분산/코넥터 손실/OSNR/FEC 등 시스템 요소가 한계 결정.

---

## 점검 문제

1. UTP에서 **트위스트 피치**가 누화/NEXT에 미치는 영향을 설명하라. 카테고리별 설계 차이를 예로 들어라.
2. 코액셜의 **특성 임피던스**가 50Ω/75Ω로 표준화된 역사적/공학적 이유를 간략히 기술하라.
3. 광 링크에서 **크로매틱 디스퍼전**이 비트율 한계에 미치는 영향과, **분산 보상(DCF/전자보상)**의 개념을 설명하라.
4. 10GBASE-T를 100 m로 제공하기 위한 케이블/채널 조건(Alien crosstalk, 차폐, 설치 품질)과 PHY 기법(에코/XT/LDPC 등)을 요약하라.
5. **링크 버짓**에 **마진**을 3~6 dB 두는 이유(노화, 온도, 커넥터 편차, 패치 증감)를 설명하라.

---

## 핵심 정리

- **Twisted-Pair**: 저비용/유연, 설치 쉬움. 누화/EMI 관리와 카테고리/설치 품질이 성능의 관건. PoE 시 발열/번들링 주의.
- **Coax**: 차폐 우수/정합 용이, 그러나 고주파 감쇠↑. CATV/계측/마이크로웨이브에 강세.
- **Fiber**: 초고대역·장거리·EMI 무관. 감쇠/분산/접속 품질/장비 비용이 설계 포인트.
- **실무 선택**은 **거리·처리율·환경(EMI)·보안·예산·운영** 제약을 종합해 결정한다.

---

## 부록 — 간단 설계 유틸

```python
import math

def power_dbm_to_mw(dbm):
    return 10**(dbm/10)

def mw_to_dbm(mw):
    return 10*math.log10(mw)

def fiber_span_loss_db(length_km, alpha_db_per_km, n_connectors=0, conn_loss_db=0.3, n_splices=0, splice_db=0.1):
    return length_km*alpha_db_per_km + n_connectors*conn_loss_db + n_splices*splice_db

# 예: 80 km, 0.23 dB/km, 커넥터 8개, 스플라이스 10개

L = fiber_span_loss_db(80, 0.23, n_connectors=8, conn_loss_db=0.3, n_splices=10, splice_db=0.1)
print("Fiber span loss:", round(L,2), "dB")

# dB 합산 도우미

def add_db(*args_db):
    return sum(args_db)

# VSWR/반사계수

def gamma(zl, z0=50.0):
    return (zl - z0)/(zl + z0)

def vswr_from_gamma(g):
    g = abs(g)
    return (1+g)/(1-g)

print("Gamma(75Ω on 50Ω):", round(abs(gamma(75,50)),3))
print("VSWR:", round(vswr_from_gamma(gamma(75,50)),3))
```
