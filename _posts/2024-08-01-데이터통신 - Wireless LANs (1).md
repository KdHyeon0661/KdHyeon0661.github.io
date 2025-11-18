---
layout: post
title: 데이터 통신 - Wireless LANs (1)
date: 2024-08-01 21:20:23 +0900
category: DataCommunication
---
# Chapter 15. Wireless LANs (IEEE 802.11)

이전 장에서 **유선 LAN(Ethernet)**, **유선 백본(SONET/ATM 등)**을 봤다면, 이번 장은 그 위에서 단말을 무선으로 붙이는 **무선 LAN(Wireless LAN, WLAN)**, 특히 **IEEE 802.11 기반 Wi-Fi**에 대한 정리입니다. 오늘날 집·회사·캠퍼스·카페 대부분이 이 표준을 사용하므로, 시험/실무 둘 다에서 핵심입니다.:contentReference[oaicite:0]{index=0}  

---

## 15.1 Introduction

### 15.1.1 Architectural Comparison

#### 1) 유선 LAN vs 무선 LAN vs 셀룰러

아키텍처 관점에서 세 가지를 한번에 비교해 보면 개념이 정리됩니다.

| 항목 | 유선 LAN (Ethernet, 802.3) | 무선 LAN (Wi-Fi, 802.11) | 셀룰러 (LTE/5G 등) |
|------|---------------------------|--------------------------|---------------------|
| 매체 | 꼬임쌍선, 광, DAC 케이블 | 2.4 / 5 / 6 GHz 대역 전파 | 라이선스 주파수 (수 GHz) |
| 표준화 | IEEE 802.3 | IEEE 802.11 | 3GPP (Rel-xx) |
| 접속 방식 | 스위치 기반, 풀 듀플렉스 | AP 기반, 반이중 | 기지국(eNB/gNB) 기반 |
| MAC 기법 | CSMA/CD(고전) → 스위치 | CSMA/CA + 관리 프레임 | 스케줄링(OFDMA, TDM 등) |
| 주소 | 48bit MAC, 2개 필드 | 최대 4 MAC 필드 | RNTI, GUTI 등 논리 ID |
| 전송 보장 | 매우 안정적, 낮은 BER | 간섭·감쇠 영향 큼 | 기지국 제어로 안정성 확보 |
| 이동성 | 케이블 길이에 한정 | AP 간 로밍 | 넓은 셀 간 핸드오버 |

- **유선 LAN**: 물리적으로 케이블을 깔고 스위치로 별도의 충돌 없이 고속 통신.
- **무선 LAN**: 같은 이더넷 프레임을 전파 위로 날리는데, **매체가 공유·보이지 않음** → **CSMA/CA, ACK, 재전송**이 필수.
- **셀룰러**: 사용자 단말이 **“혼자서” 매체를 감지하지 않고**, 기지국이 OFDMA/스케줄링으로 자원을 배분.

즉, **상위 계층(IPv4/IPv6, TCP/UDP)**은 같지만, **2계층/MAC과 물리 계층이 크게 다르기 때문에** 설계와 튜닝 방법도 달라집니다.

#### 2) 802.3 + 802.11 혼합 구조

실무 네트워크는 거의 항상 **유선 + 무선**이 섞여 있습니다.

```text
                 ┌────────────┐
      Internet ──┤  라우터/   │
                 │  게이트웨이 │
                 └─────┬──────┘
                       │ (802.3)
              ┌────────┴────────┐
              │                 │
          ┌───┴───┐         ┌───┴───┐
          │ Switch│         │  AP   │  무선 BSS
          └───┬───┘         └───┬───┘
      유선 PC/서버들           (802.11)
                                 │
                        스마트폰, 노트북 등 STA
```

- **AP(Access Point)**는 한쪽은 802.11, 다른 쪽은 802.3을 말하는 **브리지/포털**.
- 데이터는 보통 “STA ↔ AP (802.11) ↔ 스위치/라우터 (802.3) ↔ 인터넷” 경로로 흐릅니다.:contentReference[oaicite:1]{index=1}  

이 구조를 알고 있으면, 나중에 **802.11 주소 필드 4개**를 이해할 때 훨씬 수월합니다.

---

### 15.1.2 Characteristics of WLANs

유선과 달리 무선 LAN은 물리적인 특성이 다르므로, **설계·튜닝·보안**이 달라집니다.

#### 1) 반이중(half-duplex) + 공유 매체

- Wi-Fi는 **한 채널에서 한 번에 한 쪽만 송신**할 수 있습니다(반이중).
- 여러 STA/AP가 같은 채널을 사용하므로, 실제 **사용 가능한 시간(airtime)을 쪼개 씀**.
- 유선 스위치처럼 “각 포트 1 Gbps”가 아니라, **채널당 N 명이 나눠 쓰는 구조**입니다.:contentReference[oaicite:2]{index=2}  

그래서 **동시 사용자 수·프레임 크기·관리 프레임 비율**이 성능에 크게 영향을 줍니다.

#### 2) 변동하는 전파 품질과 속도 적응(rate adaptation)

전파는 다음 요인들에 의해 시시각각 바뀝니다.

- 거리
- 장애물(벽, 유리, 사람)
- 다중경로(fading)
- 다른 Wi-Fi/비-Wi-Fi 간섭(전자레인지, Bluetooth 등)

AP와 STA는 **SNR, 에러율**을 보며 실시간으로 **MCS(Modulation and Coding Scheme)**를 바꿉니다.

- 가까우면: **고차 변조(QAM-1024, QAM-4096)**, 높은 코딩률 → **고속**
- 멀면: **저차 변조(BPSK/QPSK)**, 낮은 코딩률 → **저속이지만 안정적**

간단히 쓰면, PHY 속도는 대략

$$
R_{\text{PHY}} =
N_{\text{streams}}
\times b_{\text{per symbol}}
\times \frac{N_{\text{subcarriers}}}{T_{\text{symbol}}}
$$

형태를 가집니다(실제 802.11n/ac/ax 공식은 더 복잡).:contentReference[oaicite:3]{index=3}  

#### 3) 세대별(802.11n/ac/ax/be) 특징과 최신 속도

최근 미국·유럽 문서 기준으로 **이론 최대 PHY 속도**는 대략 다음과 같습니다.:contentReference[oaicite:4]{index=4}  

| 세대(마케팅) | 표준 | 주파수 | 이론 최대 속도(대략) |
|--------------|------|--------|----------------------|
| Wi-Fi 4 | 802.11n | 2.4/5 GHz | 최대 600 Mbps (4 스트림, 40 MHz) |
| Wi-Fi 5 | 802.11ac | 5 GHz | 최대 약 3.5 Gbps (8 스트림, 160 MHz) |
| Wi-Fi 6 / 6E | 802.11ax | 2.4/5/6 GHz | 최대 약 9.6 Gbps (8×8, 160 MHz) |
| Wi-Fi 7 | 802.11be (초안) | 2.4/5/6 GHz | 이론적으로 20~40+ Gbps (320 MHz, 4096-QAM, 16×16 등) |

하지만 **실측 속도는 이론 대비 20~30% 수준**인 경우가 많습니다. 미국 NIST의 11ax 측정에서도 9.6 Gbps 이론에 대해 **평균 1.2 Gbps 정도(약 25%)**만 달성된 예가 보고되어 있습니다.:contentReference[oaicite:5]{index=5}  

#### 4) 채널과 규제 (2.4/5/6 GHz, FCC·EU)

- **2.4 GHz**: 전 세계적으로 허가 없는 ISM 대역. 채널 폭 20 MHz 기준 비중첩 3개(1, 6, 11).  
- **5 GHz**: 더 많은 채널(U-NII 대역), 일부 채널은 **DFS(레이더 회피)** 필요.  
- **6 GHz (Wi-Fi 6E)**:
  - 미국 FCC: 2020년에 **5.925–7.125 GHz, 1200 MHz**를 비면허용으로 개방.:contentReference[oaicite:6]{index=6}  
  - 유럽: 현재 약 **480 MHz(5.945–6.425 GHz)** 정도를 개방하는 방향으로 진행.:contentReference[oaicite:7]{index=7}  

6 GHz는 기존 2.4/5 GHz와 달리 **Wi-Fi 6E/7 전용**이라 간섭이 적고, **7개의 160 MHz 채널** 같은 넓은 채널 구성이 가능해 고밀도 환경에 유리합니다.  

---

### 15.1.3 Access Control (CSMA/CA)

유선 Ethernet은 예전에 **CSMA/CD(충돌 탐지)**를 썼지만, 무선에서는 **충돌 자체를 감지하기 어렵기 때문에**, IEEE 802.11은 **CSMA/CA(충돌 회피)**를 사용합니다.:contentReference[oaicite:9]{index=9}  

#### 1) 왜 CSMA/CD가 안 되는가?

- 송신 중인 STA는 **자신이 보낸 신호**가 너무 강해서, 동시에 수신되는 다른 프레임을 구분해 들을 수 없음.
- 수신되는 전파 세기가 일정하지 않고, **페이딩·반사**로 인해 “충돌 여부”를 에너지 레벨만으로 판정하기가 힘듦.
- 그래서 “보내면서 듣고, 에너지가 이상하면 충돌”이라는 CSMA/CD를 쓸 수 없음.

대신:

- 미리 **매체 상태를 잘 듣고**,  
- 충분히 조용할 때만 보내며,  
- 누가 먼저 보낼지 **무작위(backoff)**로 조정.

#### 2) DCF(Distributed Coordination Function) 알고리즘

802.11 기본 MAC은 **DCF**이며, 그 안에서 CSMA/CA가 동작합니다. 흐름을 의사코드로 써보면:

```python
import random

CW_MIN = 15    # 예: 0~15 슬롯
CW_MAX = 1023
SLOT = 9       # μs (예: 11a/g)
DIFS = 34      # μs (예시)

def transmit_frame():
    cw = CW_MIN
    while True:
        # 1. 채널이 DIFS 동안 idle일 때까지 기다림
        wait_until_channel_idle_for(DIFS)

        # 2. 0~cw 사이에서 랜덤 backoff 슬롯 선택
        backoff_slots = random.randint(0, cw)

        # 3. 슬롯 수만큼 카운트다운(채널이 busy면 일시 정지)
        while backoff_slots > 0:
            if channel_idle_for(SLOT):
                backoff_slots -= 1
            else:
                # busy -> 다시 DIFS 이후 카운트다운 재개
                wait_until_channel_idle_for(DIFS)

        # 4. 여기까지 오면 우리 차례, 프레임 전송
        send_frame()

        # 5. SIFS 후 ACK 기다림
        if wait_for_ack():
            # 성공 -> CW를 초기화하고 종료
            cw = CW_MIN
            return True
        else:
            # 실패(충돌/손실) -> CW를 2배(최대 CW_MAX까지)로 늘리고 재시도
            cw = min(2 * cw + 1, CW_MAX)
```

개념상:

1. **채널 감지**: 무선 매체가 DIFS 동안 비어 있어야 함.
2. **백오프**: 여러 STA가 동시에 전송하려는 상황을 피하려 랜덤 대기.
3. **충돌/손실**: ACK를 받지 못하면 “충돌로 간주”하고 **이중 지수 백오프**.

Arista, IEEE 튜토리얼 등에 매우 유사한 설명이 나옵니다.:contentReference[oaicite:10]{index=10}  

#### 3) Interframe Space (IFS) 계층

802.11은 **프레임 사이의 “간격”**을 이용해 우선순위를 구현합니다.

- **SIFS (Short IFS)**: ACK, CTS, 데이터 조각(fragment) 등 **응답 프레임**에 사용. 가장 짧은 간격.
- **DIFS (DCF IFS)**: 일반 데이터 전송 전 대기 시간.
- **PIFS, AIFS, EIFS** 등: PCF/QoS 용으로 정의.

SIFS < PIFS < DIFS 순서라, SIFS를 사용하는 **ACK/CTS가 항상 일반 데이터보다 먼저 발언권**을 가지게 됩니다.

#### 4) Hidden / Exposed Terminal 문제와 RTS/CTS

- **Hidden Terminal**: A와 C가 서로를 듣지 못하지만, 둘 다 B로 전송할 수 있는 상황.  
  - A와 C는 “채널이 비어 있다”고 판단하고 동시에 전송 → B에서 충돌.
- **Exposed Terminal**: B→A 전송 중인데, C→D 전송은 사실 B/A에 영향을 주지 않지만, C가 “채널 busy”로 오인해 불필요하게 기다리는 상황.

802.11은 **RTS(Request To Send) / CTS(Clear To Send)** 제어 프레임으로 이를 완화합니다.:contentReference[oaicite:11]{index=11}  

단순한 시퀀스:

```text
송신 STA   채널   수신 STA   주변 STA들
-------    ----   -------    ----------
RTS  ─────►        수신
            RTS duration을 보고 NAV 설정(조용히)
      ◄──── CTS
DATA ─────►        수신
      ◄──── ACK
```

- RTS/CTS에는 “이 뒤에 이어질 DATA+ACK에 필요한 시간”이 적혀 있어, 주변 STA는 **NAV(Network Allocation Vector)**에 이 기간만큼 채널이 점유될 것이라고 기록하고 조용히 대기.

RTS/CTS는 **큰 프레임**에만 적용하는 것이 보통이며, 작은 프레임은 오히려 오버헤드만 증가할 수 있습니다.

#### 5) QoS: EDCA/HCF 개념

802.11e 이후, 실무 Wi-Fi(특히 Wi-Fi 5/6)는 **EDCA**를 통한 우선순위 기반 MAC을 사용합니다.:contentReference[oaicite:12]{index=12}  

- 트래픽을 4개 Access Category(AC_VO, AC_VI, AC_BE, AC_BK)로 나눠  
- 각 카테고리마다 **다른 AIFS, CWmin/max**를 부여  
  → 음성/영상은 더 짧은 backoff로 **우선 접근**.

또한 AP는 **TXOP(Transmission Opportunity)**로 한 번 매체를 잡은 뒤 연속 프레임을 보낼 수 있어, 고속 PHY를 효율적으로 활용하도록 합니다.

---

## 15.2 IEEE 802.11 Project

### 15.2.1 Architecture

IEEE 802.11의 아키텍처는 용어가 많아 헷갈리기 쉬운데, 그림으로 정리하면 단순합니다.:contentReference[oaicite:13]{index=13}  

#### 1) 기본 용어

- **STA(Station)**: 802.11 인터페이스를 가진 단말 (노트북, 스마트폰, AP 포함).
- **AP(Access Point)**: 인프라 모드에서 **BSS를 관리하는 STA**. 802.11과 802.3의 브리지.
- **BSS(Basic Service Set)**: AP 하나와 그와 연결된 STA들의 집합, 또는 Ad-hoc 그룹.
- **BSSID**: BSS를 식별하는 MAC 주소(보통 AP 무선 인터페이스의 MAC).
- **SSID**: 사람이 보는 네트워크 이름(“Home-WiFi”, “Campus-Edu” 등).

#### 2) 인프라 모드 vs Ad-hoc(IBSS) vs ESS

1. **인프라 모드 (Infrastructure BSS)**

```text
      ┌─────────────┐
      │     AP      │  BSSID = 02:11:22:33:44:55
      └─────┬───────┘
            │ (무선)
 ┌──────────┴──────────┐
 │        STA들        │   SSID = "CS-Lab"
 └─────────────────────┘
```

- 대부분의 Wi-Fi가 이 모드.
- AP는 **중앙 조정자** 역할을 하며 **비콘, 인증, 연관(association)**을 처리.

2. **Ad-hoc 모드 (IBSS)**

- AP 없이 STA끼리 직접 통신.
- 시험에는 나오지만, 실제로는 거의 사용되지 않음.

3. **ESS(Extended Service Set)** + DS(Distribution System)

여러 BSS를 묶어 넓은 영역을 커버할 수 있습니다.

```text
   [BSS1]           [BSS2]
 ┌───────┐       ┌────────┐
 │  AP1  │       │  AP2   │
 └───┬───┘       └───┬────┘
     │ (802.3 DS)    │
     └─────Switch/Router───── Internet
```

- AP1·AP2는 **같은 SSID**와 **보안 설정**을 사용.
- 이 둘과 DS(보통 Ethernet) 전체를 합쳐 **ESS**라고 부름.
- STA는 이동하면서 AP1 ↔ AP2로 **로밍**하지만, 상위 IP 세션은 유지되도록 설계.

#### 3) Portal / Distribution System

- **DS**: 여러 BSS를 연결하는 백본. 보통 802.3 스위치/라우터 네트워크.
- **Portal**: 802.11 ESS와 다른 LAN(예: 유선 Ethernet)을 잇는 **브리지 역할**의 논리 엔티티.
  - 보통 AP나 WLAN 컨트롤러에 구현.

데이터 흐름:

```text
STA1 (무선, BSS1)
  ↓ 802.11
AP1
  ↓ 802.3
Switch/Router (DS, Portal)
  ↓ 802.3
Server/Internet
```

---

### 15.2.2 MAC Sublayer

MAC 서브레이어는 **프레임 포맷, 프레임 종류, 재전송, 순서 제어, 전력 관리** 등을 담당합니다.:contentReference[oaicite:14]{index=14}  

#### 1) MAC 프레임의 큰 틀

802.11 MAC 프레임(데이터 기준)은 대략 다음 구조입니다.

```text
┌────────────┬─────────┬────────┬────────┬────────┬────────┬─────────────┬──────┐
│ Frame Ctrl │ Duration│ Addr1  │ Addr2  │ Addr3  │ Seq Ctl│ Addr4(opt.) │  FCS │
└────────────┴─────────┴────────┴────────┴────────┴────────┴─────────────┴──────┘
   2 bytes     2 bytes  6 bytes  6 bytes  6 bytes   2 bytes    6 bytes    4 bytes
```

- **Frame Control**: Type, Subtype, ToDS, FromDS, Retry, More Frag, Protected 등 제어 비트.
- **Duration/ID**: 이 프레임과 응답(ACK 등)이 채널을 점유할 예상 시간 → NAV 계산에 사용.
- **Address1~4**: 최대 4개 MAC 주소 (수신자/송신자/BSSID/다른 STA).
- **Sequence Control**: 시퀀스 번호 + fragment 번호 (중복 제거, 재조립).
- **FCS**: CRC-32.

#### 2) 프레임 종류

IEEE 802.11은 MAC 프레임을 크게 네 가지 타입으로 나눕니다.:contentReference[oaicite:15]{index=15}  

1. **관리(Management)**: Beacon, Probe Request/Response, Authentication, Association Request/Response, Reassociation, Disassociation, Deauthentication 등.
2. **제어(Control)**: RTS, CTS, ACK, Block ACK, PS-Poll, CF-End 등.
3. **데이터(Data)**: 실제 상위 계층 데이터(IP 패킷 등)를 운반.
4. **확장(Extension)**: 802.11n 이후 일부 확장에 사용.

예:

- AP가 주기적으로 **Beacon**을 보내 SSID, 지원 속도, 채널, 보안 옵션을 광고.
- STA는 **Probe Request**를 브로드캐스트해서 “이 주변 AP들, 누구 있나요?”를 묻고, AP는 Probe Response로 답.
- 연결 설정은 보통
  - Authentication → Association → (WPA2/3 4-Way Handshake) 순으로 진행.

#### 3) 신뢰성: ACK, 시퀀스 번호, 재전송

802.11은 **링크 계층 수준 신뢰성**을 자체적으로 제공합니다.:contentReference[oaicite:16]{index=16}  

- 각 유니캐스트 데이터 프레임은 **ACK 프레임**으로 확인.
- 송신자는 ACK를 기다리다가 타임아웃 시 재전송 + 백오프.
- **Sequence Control** 필드로 프레임 ID를 부여해, 중복 프레임을 필터링(ACK 손실 등으로 중복 수신 가능).

의사코드:

```python
seq = 0

def send_reliable(dst, payload):
    global seq
    max_retry = 7
    attempt = 0
    while attempt <= max_retry:
        frame = make_data_frame(dst, seq, payload)
        tx(frame)
        if wait_ack(seq):
            # 성공
            seq = (seq + 1) % 4096  # 시퀀스 번호는 12비트
            return True
        attempt += 1
        increase_backoff()
    # 실패 -> 상위 계층에 에러 통보
    return False
```

#### 4) 전력 관리

모바일 단말은 배터리를 아끼기 위해 **Doze/Awake** 상태를 오갑니다.

- AP는 Beacon에 **TIM(트래픽 지시자 맵)**를 넣어, “어떤 STA에게 버퍼링된 프레임이 있는지” 광고.
- STA는 주기적으로만 깨어나 Beacon을 듣고, 자신 비트가 세트되어 있으면 **PS-Poll** 또는 QoS Null 등을 보내 **버퍼링된 데이터**를 요구.
- 802.11ax에서는 **TWT(Target Wake Time)**로 더 정교하게 “언제 깨어날지” 협상해 IoT·배터리 수명을 크게 늘립니다.:contentReference[oaicite:17]{index=17}  

---

### 15.2.3 Addressing Mechanism

802.3 Ethernet은 **Source, Destination 두 MAC 주소**만 있지만, 802.11은 **최대 4개의 주소 필드**를 가집니다. 이는 무선-유선 브리징, WDS 등 다양한 시나리오를 지원하기 위함입니다.:contentReference[oaicite:18]{index=18}  

#### 1) ToDS / FromDS 비트

Frame Control의 두 비트가 프레임의 방향을 나타냅니다.

- **ToDS = 0, FromDS = 0**: DS(Distribution System) 바깥만 사용 (Ad-hoc 등)
- **ToDS = 1, FromDS = 0**: STA → AP (무선 → 유선)
- **ToDS = 0, FromDS = 1**: AP → STA (유선 → 무선)
- **ToDS = 1, FromDS = 1**: WDS(Wireless Distribution System), AP↔AP 무선 백홀

#### 2) 주소 필드의 의미

NetworkAcademy·IEEE 튜토리얼을 참고하면, 주소 필드 의미는 대략 다음과 같습니다.:contentReference[oaicite:19]{index=19}  

| 경우 | ToDS | FromDS | Addr1 | Addr2 | Addr3 | Addr4 |
|------|------|--------|-------|-------|-------|-------|
| Ad-hoc(IBSS) | 0 | 0 | Receiver | Transmitter | BSSID | – |
| STA→AP | 1 | 0 | BSSID(AP MAC) | SA(무선 STA) | DA(최종 목적지) | – |
| AP→STA | 0 | 1 | DA(무선 STA) | BSSID(AP MAC) | SA(원본 송신자) | – |
| AP↔AP(WDS) | 1 | 1 | Receiver AP | Transmitter AP | DA | SA |

예제 시나리오를 하나 보겠습니다.

#### 3) 예제: 무선 노트북에서 인터넷 서버까지

상황:

- STA 노트북 MAC: **STA = 00:AA:AA:AA:AA:AA**
- AP 무선 MAC: **AP = 00:BB:BB:BB:BB:BB** (BSSID)
- 유선 게이트웨이(라우터) MAC: **GW = 00:CC:CC:CC:CC:CC**
- 인터넷 서버 MAC: (바깥이므로 직접 보이지 않음)

1. **STA → AP (무선 구간)**  
   - ToDS=1, FromDS=0  
   - Addr1 = BSSID = AP  
   - Addr2 = SA = STA  
   - Addr3 = DA = GW (게이트웨이 MAC; 라우터가 ARP로 알려줌)

2. **AP → 게이트웨이 (유선)**  
   - 일반 802.3 프레임  
   - Dest = GW, Src = AP(혹은 AP의 유선 MAC)

반대 방향도 비슷하게 Address1/2/3의 역할이 달라집니다.

이 구조 덕분에 AP는

- “이 프레임의 **무선 수신자**는 누구이고(Address1)”
- “**무선 송신자**는 누구이며(Address2)”
- “이 프레임의 **논리적 L3 목적지**는 어디인가(Address3)”

를 모두 알고, 유선/무선 브리징을 수행할 수 있습니다.

#### 4) SSID vs BSSID vs ESSID

- **SSID**: 사람이 보는 “네트워크 이름” (문자열)
- **BSSID**: 그 SSID를 브로드캐스트하는 **하나의 BSS를 대표하는 MAC**
- **ESSID**: 여러 BSS(여러 AP)로 확장된 네트워크 전체 이름(실제로는 SSID=ESSID로 쓰는 경우가 많음)

실무에서 “이 AP의 BSSID는 몇이야?”와 “이 Wi-Fi 이름(SSID)은?”을 구별하는 연습을 해 두어야, 캡처한 트레이스 분석이 쉽습니다.

---

### 15.2.4 Physical Layer

마지막으로, 802.11 PHY 계층의 발전과 주요 개념을 정리합니다.:contentReference[oaicite:20]{index=20}  

#### 1) 세대별 PHY 개요

| 표준 / 세대 | 대역 | 메인 기술 | 최고 PHY 속도(대략) | 특이점 |
|-------------|------|-----------|----------------------|--------|
| 802.11b (Wi-Fi 1) | 2.4 GHz | HR-DSSS | 11 Mbps | CCK, 22 MHz 채널 |
| 802.11a | 5 GHz | OFDM | 54 Mbps | 5 GHz 전용, 20 MHz 채널 |
| 802.11g | 2.4 GHz | OFDM | 54 Mbps | a의 PHY를 2.4 GHz로 |
| 802.11n (Wi-Fi 4) | 2.4/5 | MIMO-OFDM | 600 Mbps | 최대 4스트림, 40 MHz |
| 802.11ac (Wi-Fi 5) | 5 | VHT-OFDM | ~3.5 Gbps | 최대 8스트림, 160 MHz, 256-QAM |
| 802.11ax (Wi-Fi 6/6E) | 2.4/5/6 | OFDMA + MU-MIMO | 9.6 Gbps | BSS coloring, TWT, 1024-QAM |
| 802.11be (Wi-Fi 7, 초안) | 2.4/5/6 | EHT, 4096-QAM | 20~40+ Gbps | 320 MHz, 멀티링크, 16×16 MU-MIMO |

Wi-Fi 6의 9.6 Gbps 이론값은 **8×8 MIMO, 160 MHz 채널**을 가정한 것입니다.:contentReference[oaicite:21]{index=21}  

#### 2) OFDM과 서브캐리어

802.11a/g 이후 PHY는 **OFDM** 기반입니다.

- 채널을 여러 **서브캐리어(예: 64개, 256개 …)**로 나누고,
- 각 서브캐리어에 QAM 심볼을 실어 전송.
- 서브캐리어 일부는 **파일럿·가드 서브캐리어**로 사용.

데이터율은 대략

$$
R_{\text{PHY}} = N_{\text{streams}} \times N_{\text{data subcarriers}} \times b_{\text{per symbol}} \times \frac{R_{\text{code}}}{T_{\text{symbol}}}
$$

형태로 계산됩니다. 여기서

- \(N_{\text{streams}}\): 공간 스트림 수,
- \(b_{\text{per symbol}}\): 변조 차수(예: 256-QAM → 8비트/심볼),
- \(R_{\text{code}}\): 코딩률(예: 3/4),
- \(T_{\text{symbol}}\): OFDM 심볼 지속시간.

#### 3) 802.11ax (Wi-Fi 6/6E)의 핵심 PHY 기능

미국 NIST, 여러 벤더 백서 기준으로 Wi-Fi 6의 핵심 포인트는 다음과 같습니다.:contentReference[oaicite:22]{index=22}  

- **OFDMA**: 하나의 채널을 작은 **Resource Unit(RU)**로 쪼개 여러 STA에 동시에 배정.
- **UL/DL MU-MIMO**: 업링크/다운링크 모두 다중 사용자 MIMO 지원.
- **BSS Coloring**: 인접 BSS의 트래픽을 컬러로 구분해, **셀 간 간섭 환경에서 더 공격적인 재사용**이 가능.
- **TWT(Target Wake Time)**: STA와 AP가 “언제 깨어날지” 약속해 전력 소비 감소.
- **2.4/5/6 GHz 동시 지원**: Wi-Fi 6E부터 6 GHz 대역 확장.

#### 4) 6 GHz 대역과 채널화

앞서 봤듯 미국 FCC, 유럽 규제기관은 6 GHz 대역을 Wi-Fi에 열었습니다.:contentReference[oaicite:23]{index=23}  

- 미국: 5.925–7.125 GHz 전체 1200 MHz (저전력 실내 / AFC 기반 표준 전력)
- 유럽: 약 5.945–6.425 GHz, 480 MHz

이를 20/40/80/160 MHz 채널로 나누면, 대략:

- 20 MHz 채널: 최대 59개까지 가능(이론상).
- 80 MHz 채널: 약 14개(전 세계 기준).
- 160 MHz 채널: 7개 (US 기준).

Wi-Fi 6E/7 설계 시에는 **160 MHz 이상의 넓은 채널을 몇 개나 쓸 수 있는지**, DFS 제약은 어떤지, 2.4/5/6 GHz를 어떻게 병행할지 설계해야 합니다.

#### 5) 간단 속도 계산 예제

802.11ax, 80 MHz, 2×2, 1024-QAM(10비트/심볼), 코딩률 5/6, 심볼시간 13.6 μs(가드 포함)이라고 가정하고 대략적인 PHY 속도를 Python으로 계산해보면:

```python
# 단순 예시: 실제 802.11ax 공식과 약간 다를 수 있음
N_streams = 2
N_data_subcarriers = 980   # 예: 80 MHz에서 대략적인 데이터 서브캐리어 수
bits_per_symbol = 10       # 1024-QAM
code_rate = 5/6
T_symbol = 13.6e-6         # seconds

R_phy = N_streams * N_data_subcarriers * bits_per_symbol * code_rate / T_symbol
print("대략적인 PHY 속도:", R_phy / 1e9, "Gbps")
```

실제 표준과 제조사 스펙을 보면 80 MHz, 2×2 MIMO에서 약 1.2 Gbps 정도의 PHY 속도가 흔히 나옵니다.:contentReference[oaicite:24]{index=24}  

#### 6) 실제 Goodput과 효율

PHY 속도 대비 실제 TCP/UDP goodput은 프레임 오버헤드, MAC/PHY preamble, ACK, 관리 프레임, 백오프, 상위 계층 헤더 등을 빼면 다음과 같이 표현할 수 있습니다.

$$
\eta = \frac{T_{\text{payload}}}{T_{\text{payload}} + T_{\text{overhead}}}
$$

즉, **페이로드 전송 시간 대비 총 전송 시간 비율**이 효율입니다.

- VoIP처럼 작은 패킷(예: 80바이트)을 자주 보내면, **헤더/ACK 비율이 커져 효율이 낮음**.
- 대용량 파일 전송처럼 큰 프레임을 보내면, **헤더·백오프를 여러 바이트에 나눠 먹으므로 효율↑**.:contentReference[oaicite:25]{index=25}  

NIST·연구 결과를 보면, 고급 AP라도 실제 환경에서 **이론 대비 20–40% 수준**의 goodput이 흔합니다.

---

이 장에서는 **무선 LAN의 구조(Architecture)와 802.11 MAC/PHY의 핵심 요소**를 유선 LAN과 비교하며 정리했습니다.

- 15.1에서는 **유선 vs 무선 vs 셀룰러** 아키텍처 비교, 무선의 고유 특성(공유 매체, 반이중, 속도 적응, 6 GHz 규제), 그리고 **CSMA/CA 기반 DCF, RTS/CTS, EDCA**를 다뤘고,
- 15.2에서는 **802.11 아키텍처 (BSS/ESS/DS/AP)**, **MAC 서브레이어(프레임 구조, 관리/제어/데이터 프레임, ACK/재전송, 전력 관리)**, **4-주소 방식 주소 체계**, **세대별 PHY(OFDM, MIMO, OFDMA, 6 GHz)**를 정리했습니다.

이 내용을 기반으로 다음 장에서 **보안(WPA2/3), 로밍, 고급 QoS, 실제 무선 설계(WLAN 설계·사이트 서베이)** 쪽으로 확장해 나가면, 시험·실무 둘 다에서 바로 써먹을 수 있는 수준의 무선 LAN 이해에 도달하게 됩니다.