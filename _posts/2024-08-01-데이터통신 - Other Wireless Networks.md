---
layout: post
title: 데이터 통신 - Other Wireless Networks
date: 2024-08-01 23:20:23 +0900
category: DataCommunication
---
# Chapter 16. Other Wireless Networks — WiMAX, Cellular Telephony, Satellite

이 장은 **“무선 LAN(802.11, Bluetooth)”을 넘어선 다른 무선 인프라 네트워크**를 다룹니다.

- 16.1: **WiMAX (IEEE 802.16)** – 한때 “무선 마지막 1마일”과 “4G 후보”였던 광대역 무선 액세스
- 16.2: **셀룰러 통신 (1G → 5G)** – 음성에서 초고속/초저지연/대량 IoT까지의 진화
- 16.3: **위성 네트워크 (GEO / MEO / LEO)** – 전 지구 커버리지와 위도·경도·시간 동기 제공

각 섹션에서 **서비스, 표준, 계층 구조, 동작 방식**을 정리하고,
**수식/예제/코드**까지 포함해 시험·실무에 바로 쓸 수 있는 수준으로 정리합니다.

---

## WiMAX (IEEE 802.16)

WiMAX(Worldwide Interoperability for Microwave Access)는 **IEEE 802.16 계열 표준**에 기반한 **광대역 무선 접속(BWA: Broadband Wireless Access)** 기술입니다.
초기에는 “고정 무선 DSL/케이블 대체”로, 이후 802.16e/m에서는 “모바일 브로드밴드, 4G(IMT-Advanced) 후보”로 발전했습니다.

### WiMAX Services

#### 서비스 범위 개요

ETSI·ITU·IEEE 문서 기준으로 WiMAX가 목표로 했던 서비스는 크게 다음과 같습니다.

1. **고정 무선 접속(Fixed Wireless Access, FWA)**
   - 주거·소규모 사업장에 **DSL/케이블 대체**로 수십 Mbps급 인터넷 제공
   - 주파수: 보통 2–11 GHz (비가시/부분가시), 10–66 GHz(가시) 대역
2. **노매딕/이동형 접속(Nomadic / Portable / Mobile)**
   - 노트북/태블릿용 모바일 모뎀, CPE(Customer Premises Equipment)
   - 802.16e 이후: “Mobile WiMAX”로 이동성 지원(핸드오버 포함)
3. **기업/기관 전용망**
   - 캠퍼스/공장/시청 등을 잇는 **무선 백홀**
   - 긴 거리 점대점 링크로 지사 간 연결, IP VPN, VoIP, 영상회의
4. **공공 안전·재난 통신**
   - 802.16m 요구사항 문서는 **공공안전/응급 통신** 지원을 명시적으로 요구
     (우선순위 부여, 미션 크리티컬 트래픽, 멀티미디어 서비스 등)
5. **4G(IMT-Advanced)급 모바일 광대역 후보**
   - 802.16m은 ITU-R IMT-Advanced 요구사항에 부합하는 4G 시스템 후보로 제출되었고,
     LTE-Advanced와 함께 4G로 분류됨.

#### 실무 예제: “농촌 ISP” 시나리오

미국/유럽의 농촌 ISP가 광케이블을 깔기 힘든 지역에 WiMAX를 쓴다고 가정하자.

- 중앙 마을에 **WiMAX BS (Base Station)** 설치, 광/마이크로파 백홀로 인터넷에 접속
- 주변 10–20 km 반경에 있는 농가에 **외장 CPE(작은 안테나+모뎀)** 설치
- 각 CPE는 10~50 Mbps 정도의 다운로드 속도로 인터넷·VoIP·영상까지 사용

이 경우 서비스는:

- **Downlink/IP 인터넷 액세스**
- **VoIP**(SIP) – UGS(고정 대역) 또는 rtPS 서비스 클래스
- **VPN** – nrtPS 또는 BE 클래스

WiMAX MAC은 각 가입자에게 **서비스 플로우(service flow)** 단위로 QoS를 보장해,
동일 무선 자원을 효율적으로 나누는 것이 핵심입니다.

---

### IEEE Project 802.16

#### 표준 진화

IEEE와 ETSI 문서를 바탕으로 802.16의 주요 버전/확장을 정리하면:

| 표준 | 주요 내용 | 특징 |
|------|----------|------|
| 802.16-2001 | 10–66 GHz 고정 BWA | 가시거리(LOS) 기반, 초기 고정 서비스 |
| 802.16-2004 | 2–11 GHz 포함, 고정/노매딕 BWA | NLOS 지원, “Fixed WiMAX” |
| 802.16e-2005 | 802.16-2004 개정 – **모바일** 지원 | Mobile WiMAX, 핸드오버, OFDMA PHY, < 6 GHz 대역 |
| 802.16j-2009 | 멀티홉 릴레이 지원 | 커버리지 확장, 릴레이 BS |
| 802.16m | IMT-Advanced 후보, 고급 에어인터페이스 | MIMO·Beamforming·고속기능, 4G급 |

IEEE 802.16e 소개 페이지에 따르면, 802.16e-2005는
“**fixed와 mobile 광대역 무선 접속을 동시에 지원**하며, 차량 수준 이동성(vehicular speed)을 지원하고, BS 간 핸드오버 기능을 제공한다”라고 설명되어 있습니다.

802.16m 문서는 기존 16e 서비스를 더 효율적으로 지원하고, **IMT-Advanced(4G) 서비스**에 맞춰 성능·MIMO·QoS를 크게 확장하는 요구를 제시합니다.

#### IEEE 802.16과 IMT/3GPP의 관계

- ITU-R는 **IMT-2000(3G), IMT-Advanced(4G), IMT-2020(5G)** 등 글로벌 모바일 표준 가족을 정의.
- 802.16e/16m 기반 WiMAX는 **IMT-2000 / IMT-Advanced 무선 액세스 기술 중 하나**로 포함되며,
  3GPP의 **UMTS/LTE 계열**과 함께 IMT 패밀리로 분류됩니다.

실제 상용화에서는 LTE가 주도적이 되었지만, WiMAX는 여전히 일부 지역 시스템·백홀·특수망에서 사용되거나 과거 네트워크의 레거시로 존재합니다.

---

### Layers in Project 802.16

802.16은 **레이어 1(물리) + 레이어 2(MAC) + 상위 서비스 매핑을 담당하는 Convergence Sublayer(CS)** 구조를 가집니다.

#### 전체 계층 구조

텍스트로 그리면:

```text
┌────────────────────────────────────────────┐
│   IP, Ethernet, ATM, 기타 상위 프로토콜    │
├───────────────────────────┬────────────────┤
│   Convergence Sublayer    │  (CS)          │
│   - IP/Ethernet/ATM 매핑 │                 │
├───────────────────────────┴────────────────┤
│   MAC Common Part Sublayer (MAC CPS)       │
│   - 연결 관리, 스케줄링, QoS, ARQ         │
├────────────────────────────────────────────┤
│   Security Sublayer (PKM, 암호화)          │
├────────────────────────────────────────────┤
│   Physical Layer (PHY, OFDM/OFDMA)         │
└────────────────────────────────────────────┘
```

#### PHY: OFDM / OFDMA, 프레임 구조

- 초기 802.16: 256 서브캐리어 OFDM, 10–66 GHz (LOS)
- 802.16e/m: **OFDMA 기반**, 다수의 서브채널을 사용자별로 할당, MIMO·beamforming 지원.

대표적인 특징:

- **다운링크/업링크 둘 다 OFDMA** (mobile WiMAX)
- **TDD/FDD 모두 지원** (실제 배치에서는 TDD가 일반적)
- 변조: QPSK, 16QAM, 64QAM (코딩률에 따라 다양한 MCS)
- **프레임 구조**:
  - 다운링크 맵(DL-MAP), 업링크 맵(UL-MAP)으로 각 사용자에게 **시간·주파수 자원 할당 정보** 전달
  - DL subframe, UL subframe, TTG/RTG(전환 가드 시간) 등으로 구성

PHY 데이터율은 대략

$$
R_{\text{PHY}} = B \times \eta_{\text{SE}}
$$

- \(B\): 채널 대역폭 (예: 10 MHz)
- \(\eta_{\text{SE}}\): 스펙트럼 효율(예: 3–5 bit/s/Hz)

예를 들어, 10 MHz 대역폭, 4 bit/s/Hz의 효율을 달성하면

$$
R_{\text{PHY}} \approx 10\,\text{MHz} \times 4\,\text{bit/s/Hz} = 40\,\text{Mbit/s}
$$

이 값은 셀 전체에 대한 이론적인 PHY 속도이며, 실제 사용자당 goodput은 MAC 오버헤드·채널 품질에 따라 달라집니다.

##### 코드 예제: 간단 WiMAX 셀 용량 계산

```python
def wimax_cell_capacity(bw_mhz, spectral_eff, overhead_ratio=0.25):
    """
    bw_mhz: 채널 대역폭 (MHz)
    spectral_eff: 스펙트럼 효율 (bit/s/Hz)
    overhead_ratio: 프레임 제어, 보호구간, 재전송 등 오버헤드 비율 (0~1)
    """
    # PHY 속도 (bps)
    r_phy = bw_mhz * 1e6 * spectral_eff
    # MAC/프로토콜 오버헤드를 제거한 유효 셀 용량
    capacity = r_phy * (1 - overhead_ratio)
    return r_phy, capacity

if __name__ == "__main__":
    r_phy, cap = wimax_cell_capacity(10, 4.0, overhead_ratio=0.3)
    print("이론 PHY:", r_phy/1e6, "Mbps")
    print("유효 셀 용량(대략):", cap/1e6, "Mbps")
```

- 이 값은 실제 설계(사용자 수, QoS 클래스, 채널 품질)에 따라 달라지며,
- ETSI/IEEE 문서에서는 실제 셀 당 수십 Mbps 수준의 유효 용량을 목표로 했습니다.

#### MAC: 연결 기반, 서비스 플로우, QoS

WiMAX MAC은 **“connection-oriented”** 설계입니다.

- 각 단말은 BS와 **연결(connection)**을 맺고, 그 안에 하나 이상의 **서비스 플로우(service flow)**를 가짐.
- 서비스 플로우는 **QoS 파라미터(최소/최대 대역폭, 지연, 우선순위, ARQ 여부)**를 포함.

대표 QoS 클래스:

| 클래스 | 용도 | 특성 |
|--------|------|------|
| UGS (Unsolicited Grant Service) | 고정 비트레이트 VoIP 등 | 고정 크기 주기적 그랜트 |
| rtPS (real-time Polling Service) | 가변 비트레이트 실시간 스트림 | 주기적 폴링, 지연 제약 |
| nrtPS (non-real-time PS) | 대규모 파일 전송 등 | 평균 대역 보장, 지연 덜 민감 |
| BE (Best Effort) | 웹 브라우징 등 | 남는 대역폭 사용 |
| ertPS (extended rtPS) | VoIP with silence suppression | 침묵 구간 처리에 적합 |

BS의 스케줄러는 프레임마다 각 서비스 플로우에 자원을 할당해, **셀 전체 자원을 QoS-aware하게 분배**합니다.

---

## Cellular Telephony (1G ~ 5G)

셀룰러 시스템은 ITU-IMT/3GPP 계열의 **광역 이동통신** 기술입니다.

- **1G**: 아날로그 음성
- **2G**: 디지털 음성 + SMS
- **3G**: 모바일 데이터(수백 kbps~수 Mbps)
- **4G**: LTE/WiMAX, 전 IP, 수백 Mbps~1 Gbps
- **5G**: NR, eMBB/URLLC/mMTC, Gbps급, 저지연, 네트워크 슬라이싱

ITU-R 문서는 3G/4G/5G를 각각 **IMT-2000, IMT-Advanced, IMT-2020**으로 정의합니다.

### Operation — 셀룰러 시스템의 기본 동작

#### 셀 구조와 주파수 재사용

셀룰러 네트워크는 지리적 영역을 **셀(cell)**로 나누고, 각 셀마다 **기지국(BS, eNB, gNB)**를 둡니다.

- 각 셀은 하나 이상의 주파수 블록을 할당받음.
- 인접 셀에는 다른 블록을 주어 **간섭을 줄이고**, 떨어진 셀 사이에는 블록을 재사용 → **주파수 재사용(frequency reuse)**.

간단 히스토그램/그림:

```text
     (F1)        (F2)        (F3)
   ●─────●     ●─────●     ●─────●
   │     │     │     │     │     │
(F3)   (F2)  (F1)   (F3)  (F2)   (F1)
```

#### 이동성과 위치 관리

셀룰러 시스템은 사용자의 **위치 등록/갱신, 페이징, 핸드오버**를 통해 항상 가까운 기지국과 연결을 유지합니다.

- **Location Update / Registration**
  - 단말이 새로운 위치 영역(Location Area / Tracking Area)에 들어가면, 코어 네트워크(HLR/HSS/UDM)에 자신의 위치를 등록.
- **Paging**
  - 누군가 단말에 전화/메시지를 보내면, 단말이 속한 영역에 방송(paging)을 보내 탐색.
- **Handover / Handoff**
  - 단말이 이동하며 다른 셀의 신호가 더 강해질 때, 세션을 끊지 않고 기지국을 전환.

##### 의사코드: 단순 핸드오버 결정 알고리즘

```python
def should_handover(rssi_serving, rssi_neighbor, threshold=3, hysteresis=2):
    """
    rssi_serving: 현재 셀 RSSI (dBm)
    rssi_neighbor: 이웃 셀 RSSI (dBm)
    threshold: 이웃 셀 RSSI가 현재 셀보다 최소 몇 dB 더 커야 하는지
    hysteresis: 핸드오버에 쓰는 히스테리시스
    """
    # 이웃 셀이 충분히 강하고 (threshold),
    # 현재 셀이 일정 이하로 떨어졌을 때 (hysteresis)
    if rssi_neighbor >= rssi_serving + threshold and rssi_serving < -80 + hysteresis:
        return True
    return False
```

실제 3GPP 표준에서는 훨씬 복잡한 **Event A3, A4, A5** 기반 측정 보고와
RRC 핸드오버 절차를 사용하지만, 기본 아이디어는 “이웃 셀이 더 좋으면 핸드오버”입니다.

---

### 1G — Analog Cellular

- 시기: 1980년대~1990년대 초
- 대표 시스템: AMPS(미국), TACS(유럽), NMT(북유럽) 등
- 기술적 특징:
  - 아날로그 **FM 음성**
  - 주로 **FDMA** – 30 kHz 또는 유사한 좁은 채널
  - 보안 취약(도청 쉬움), 데이터 서비스 거의 없음

**예시**: AMPS 시스템에서 30 kHz 채널 하나에 한 통화만 할당, 재사용 거리를 확보하기 위해 넓은 셀 간격이 필요 → 스펙트럼 효율 낮음.

---

### 2G — Digital Cellular (GSM, CDMA)

- 시기: 1990년대~2000년대 초
- ITU/3GPP 자료 기준, 주요 기술:

| 표준 | 접속 방식 | 특징 |
|------|-----------|------|
| GSM | FDMA + TDMA | 200 kHz 채널, 8타임슬롯 |
| IS-95 (cdmaOne) | CDMA | 넓은 대역폭에 코드 분할 |

특징:

- **디지털 음성**(GMSK 등), 향상된 음질/보안
- **SMS**: 160자 문자 메시지
- **회선 교환 데이터**: 9.6 kbps 등, 이후 GPRS/EDGE로 패킷 데이터(수십~수백 kbps)

---

### 3G — IMT-2000

ITU는 3G를 **IMT-2000**으로 표준화했습니다. 대표 기술:

- WCDMA/UMTS (3GPP)
- CDMA2000 (3GPP2)
- TD-SCDMA 등

특징:

- **패킷 데이터 중심**: 수백 kbps~수 Mbps
- 비디오 통화, 모바일 인터넷
- 넓은 채널 대역폭(5 MHz 등), **CDMA 기반** 다중접속

---

### 4G — IMT-Advanced (LTE, WiMAX)

ITU-R는 4G를 **IMT-Advanced**로 정의하며, 대표 기술로 LTE-Advanced, 802.16m 기반 WiMAX를 포함합니다.

공통 특징:

- **OFDMA + MIMO** (DL), SC-FDMA(UL, LTE)
- **All-IP Core (EPC)** – 음성까지 VoIP/IMS 기반
- **정교한 QoS, 스케줄링, 고차 변조(64QAM 이상)**

LTE-Advanced 요구사항 예시:

- 다운링크 peak ~1 Gbps, 업링크 ~500 Mbps (이론)
- 스펙트럼 효율: 수 bit/s/Hz 이상

---

### 5G — IMT-2020

ITU-R는 5G를 **IMT-2020**으로 규정하며, 3GPP의 NR(New Radio)이 대표 기술입니다.

5G의 세 가지 대표 서비스 카테고리:

1. **eMBB (enhanced Mobile Broadband)**
   - Gbps급 모바일 브로드밴드
2. **URLLC (Ultra-Reliable Low Latency Communications)**
   - ms 단위 지연과 99.999% 수준 가용성
3. **mMTC (massive Machine Type Communications)**
   - km²당 수십만 단말의 저속 IoT 연결

전파/표준 특징:

- 주파수:
  - FR1: Sub-6 GHz (예: 3.5 GHz)
  - FR2: mmWave (예: 26, 28 GHz 등)
- 기술:
  - OFDM 기반, 넓은 채널폭(최대 400 MHz)
  - Massive MIMO, 빔포밍
  - 네트워크 슬라이싱, MEC(엣지 컴퓨팅)

##### 코드 예제: 단순 세대별 스펙 요약 출력

```python
generations = {
    "1G": {"peak_mbps": 0.01, "tech": "Analog, FDMA"},
    "2G": {"peak_mbps": 0.1, "tech": "Digital, GSM/CDMA, TDMA/CDMA"},
    "3G": {"peak_mbps": 42, "tech": "WCDMA/HSPA, CDMA2000"},
    "4G": {"peak_mbps": 1000, "tech": "LTE-Advanced, OFDMA + MIMO"},
    "5G": {"peak_mbps": 20000, "tech": "NR, mmWave, Massive MIMO"},
}

for gen, info in generations.items():
    print(f"{gen}: 최고 약 {info['peak_mbps']} Mbps, 기술 = {info['tech']}")
```

여기서 peak 값은 ITU/3GPP의 이론적 목표값에 가깝고, 실제 상용망에서의 측정치는 환경에 따라 훨씬 낮습니다.

---

## Satellite Networks

위성 네트워크는 **지상과 우주를 연결하는 무선 링크**입니다.

- **GEO (정지궤도)**: 적도 상공 약 35,786 km, 지표에서 정지처럼 보임
- **MEO (중궤도)**: 약 8,000–20,000 km, GNSS(GPS, Galileo 등)에 사용
- **LEO (저궤도)**: 약 500–2,000 km, 인터넷(Starlink 등)에 사용

미국 FCC/ITU/NASA 문서 기준으로, 이 궤도들은 위성통신·위치·시간동기(PNT) 서비스에 사용됩니다.

### Operation — 위성 네트워크의 기본 동작

#### 기본 구성

- **우주 세그먼트(Space Segment)**: 위성 자체 (GEO/MEO/LEO)
- **지상 세그먼트(Ground Segment)**:
  - 게이트웨이/지구국(earth station)
  - 사용자 단말(위성 모뎀, 위성전화 등)
- **제어 세그먼트(Control Segment)**:
  - 위성 궤도/상태 제어, 네트워크 관리

링크는 보통

```text
사용자 단말 → (업링크) → 위성 → (다운링크) → 게이트웨이 → 인터넷
```

형태이며, 일부 LEO 시스템은 **위성 간 링크(ISL)**로 우주에서 패킷을 라우팅하기도 합니다.

#### 지연과 경로손실

간단 모델로, 위성 시스템의 **전파 지연**은 대략

$$
T_{\text{prop}} \approx \frac{d}{c}
$$

- \(d\): 신호 경로 길이 (m)
- \(c\): 빛의 속도 (\(\approx 3\times10^8\ \text{m/s}\))

**GEO**의 경우:

- 지상–위성 거리 약 36,000 km
- 지구–위성–지구 왕복 거리 ≈ 72,000 km
  ⇒ **한 번 위성까지 갔다 오는 데만** 약

$$
T_{\text{RT}} \approx \frac{72{,}000\,\text{km}}{3\times10^5\,\text{km/s}} \approx 0.24\ \text{s} = 240\ \text{ms}
$$

여기에 지상 라우팅·처리 지연이 더해져, 실제 GEO 위성 회선의 왕복 지연은 500 ms 수준이 흔합니다.

**자유공간 경로손실(FSPL)**은

$$
\text{FSPL(dB)} = 20\log_{10}(d) + 20\log_{10}(f) + 32.44
$$

(여기서 \(d\)는 km, \(f\)는 MHz). 쿄도(수만 km)와 고주파수(Ku/Ka 대역) 때문에, 매우 높은 이득을 가진 안테나와 저잡음 증폭기(LNA)가 필요합니다.

##### 코드 예제: 궤도 고도별 대략적인 왕복지연 추정

```python
import math

C = 3e8  # 빛의 속도 (m/s)
R_E = 6371e3  # 지구 반지름 (m)

def rtt_for_altitude(alt_km):
    h = alt_km * 1000
    # 단순 근사: 지상-위성 거리 ≈ R_E + h (수평 거리 무시)
    d_one_way = R_E + h
    t_one_way = d_one_way / C
    # 지상-위성-지상 왕복
    return 2 * t_one_way

for alt in [600, 20000, 35786]:
    rtt_ms = rtt_for_altitude(alt) * 1000
    print(f"높이 {alt} km 위성 대략 RTT: {rtt_ms:.1f} ms (지상-위성-지상만)")
```

이 근사는 매우 단순하지만, **LEO가 GEO보다 지연이 훨씬 낮다**는 점을 직관적으로 보여줍니다.
실제 지연은 궤도 기하·지상망·위성 간 링크 등으로 달라집니다.

---

### GEO Satellite

#### 특성

FCC 문서에 따르면, 정지궤도 위성은 **지구 적도 상공 약 22,300 마일(≈35,786 km)** 높이에서
하루(24시간)와 동일한 주기로 지구를 공전해, 지상에서 보면 **하늘에 고정**되어 보입니다.

특징:

- **고정된 위치** – 안테나를 한 번 맞추면 계속 동일 방향
- **넓은 커버리지** – 지구 표면의 약 1/3을 커버
- **높은 지연** – 왕복 500 ms 수준
- 주로 사용:
  - 방송 TV/라디오
  - 고정 위성 서비스(FSS) – VSAT 네트워크, 백홀
  - 일부 위성 인터넷/전화

#### 예제: GEO 위성 인터넷 VPN 회선

기업이 미국–유럽 간 백업 회선으로 GEO 위성 링크를 사용한다고 하자.

- 왕복 지연 ≈ 550 ms
- TCP를 사용할 경우, **윈도 크기(window size)**가 작으면 채널 용량을 제대로 활용 못 함.

TCP의 대략적인 최대 처리량은

$$
R_{\text{TCP}} \approx \frac{\text{Window Size}}{\text{RTT}}
$$

예를 들어, 윈도 64 kB, RTT 0.5 s라면

$$
R_{\text{TCP}} \approx \frac{64\times 1024 \text{ bytes}}{0.5\ \text{s}} \approx 1\ \text{Mbyte/s} \approx 8\ \text{Mbps}
$$

GEO 위성 링크에서는 **윈도 크기·TCP 튜닝·가속기**가 매우 중요하다는 것을 보여주는 간단한 예입니다.

---

### MEO Satellite

#### 특성

ITU·FCC·ESA 문서 기준으로, MEO는 대략 **8,000~20,000 km** 범위의 궤도입니다.

- 대표 예:
  - **GNSS 위성**: GPS(미국), Galileo(유럽), GLONASS, BeiDou 등
    (일반적으로 20,000 km 근처)
- 특징:
  - GEO보다 **지연이 낮고**, LEO보다 **커버리지 넓음**
  - 주로 **위치·항법·시간동기(PNT)** 서비스에 사용
  - 일부 통신/데이터 위성도 MEO 사용 가능

#### 예제: GPS 신호의 전파 지연

GPS 위성은 약 20,200 km 고도에서 운용됩니다. 한 번의 지상–위성 경로는 약

$$
d \approx 20{,}200 + 6{,}371 \ \text{km} \approx 26{,}571\ \text{km}
$$

(실제 경로는 궤도 기하에 따라 달라짐). 전파 지연은

$$
T \approx \frac{d}{c} \approx \frac{26{,}571\times10^3}{3\times10^8} \approx 0.089\ \text{s} \approx 89\ \text{ms}
$$

실제 GNSS 수신기에서는 이 지연과 도플러 효과, 이온층·대류권 오차 등을 보정해 위치를 계산합니다.

---

### LEO Satellite

#### 특성

FCC·기술 자문 문서에서는 LEO를 대략 **500~2,000 km 고도**의 비지구정지 궤도로 정의하며,
NGSO(Non-Geostationary Orbit) 위성 시스템의 대표 사례로 듭니다.

특징:

- **낮은 지연** – 왕복 수십 ms 수준도 가능
- **작은 커버리지** – 한 위성이 커버하는 면적이 GEO보다 훨씬 좁음
- **대규모 별자리(constellation)** 필요
  - Starlink, OneWeb 등 수백~수천기의 위성을 사용
- **빠른 이동** – 수 분 단위로 머리 위를 지나가므로, 사용자 단말은 자주 다른 위성으로 핸드오버

#### 예제: LEO 기반 위성 인터넷

LEO 인터넷 시스템(예: 여러 미국/유럽 사업자)은 다음과 같은 구조를 가집니다.

- 사용자 터미널 ↔ LEO 위성 ↔ (위성 간 링크) ↔ 게이트웨이 ↔ 인터넷
- 고도 550 km, 왕복 지연 수십 ms
- 수많은 위성 간 **인터-위성 링크(ISL)**를 통해, 지상 경로보다 짧을 수도 있는 우주경로를 구성

##### 코드 예제: LEO vs GEO 환경에서 단순 HTTP 왕복 시간 비교

```python
def http_rtt(rtt_link_ms, server_processing_ms=20):
    """
    rtt_link_ms: 링크 계층 왕복 지연 (ms)
    server_processing_ms: 서버 처리 시간 (ms)
    간단히: TCP 핸드셰이크(1 RTT) + HTTP 요청/응답(1 RTT) + 서버 처리 시간
    """
    return 2 * rtt_link_ms + server_processing_ms

for name, rtt_ms in [("LEO", 40), ("GEO", 550)]:
    total = http_rtt(rtt_ms)
    print(f"{name} 링크에서 단순 HTTP 왕복 시간 ≈ {total} ms")

```

출력은 대략:

- LEO: 40×2 + 20 ≈ 100 ms
- GEO: 550×2 + 20 ≈ 1,120 ms

즉, 같은 HTTP 요청이라도 **위성 궤도에 따라 체감 속도가 크게 달라짐**을 보여 줍니다.

---

## 정리

이 장에서는 **WiMAX, 셀룰러, 위성 네트워크**라는 세 가지 “기타 무선 네트워크”를 다뤘습니다.

- **WiMAX(802.16)**: 고정·모바일 광대역 무선 접속, OFDMA/MIMO 기반, 802.16e/m으로 발전해 IMT-Advanced 후보까지 갔지만, 시장에서는 LTE 계열에 밀렸음.
- **셀룰러(1G~5G)**: 아날로그 음성(1G) → 디지털(2G) → 모바일 데이터(3G) → IP 기반 광대역(4G) → 초고속·초저지연·대량 IoT(5G)로 진화.
- **위성 네트워크**: GEO의 광범위 커버리지와 높은 지연, MEO의 PNT 중심 역할, LEO의 저지연·대규모 별자리 구조가 상호 보완적으로 쓰임.

앞서 다룬 **무선 LAN/블루투스**와 함께, 이들 기술을 계층/서비스/주파수/지연 관점에서 비교하면
실제 네트워크 설계(예: “농촌 지역에 광대역을 어떻게 넣을까?”, “항공기·선박 인터넷은 무엇을 쓸까?”)에서 어떤 조합이 적절한지 판단하는 데 큰 도움이 됩니다.
