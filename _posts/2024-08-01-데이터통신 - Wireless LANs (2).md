---
layout: post
title: 데이터 통신 - Wireless LANs (2)
date: 2024-08-01 22:20:23 +0900
category: DataCommunication
---
# Chapter 15.3 Bluetooth — 저전력 개인 무선 네트워크

블루투스는 **“케이블 대체(cable replacement)” + 저전력 개인 영역 네트워크(PAN)**를 목표로 설계된 단거리 무선 기술입니다.  
처음에는 **클래식 Bluetooth(BR/EDR)**로 헤드셋·키보드·시리얼 케이블 대체에 쓰였고, 이후 **Bluetooth Low Energy(BLE)**가 추가되어 센서·IoT·비콘·스마트워치 등으로 확장되었습니다.

---

## 15.3.1 Architecture

### 1) 설계 목표와 기본 특성

Bluetooth의 핵심 설계 목표는 다음과 같이 요약할 수 있습니다.  

1. **저전력·저비용 단거리 통신**
   - 2.4 GHz ISM 대역 사용 (면허 불필요)
   - 수 m ~ 수십 m 거리, mW 수준 송신 전력
2. **케이블 대체**
   - 키보드, 마우스, 헤드셋, 시리얼 케이블, 게임패드 등
3. **주변기기용 PAN (Personal Area Network)**
   - 스마트폰을 중심으로 센서·웨어러블·차량·가전 연결
4. **주파수 도약(FHSS) 기반 강한 간섭 내성**
   - 2.4 GHz 대역에서 Wi-Fi, 전자레인지, 기타 장비와의 공존
5. **버전 업으로 점진적 확장**
   - BR/EDR → BLE → 5.x에서 속도/거리/방송용량 확장 → 5.2~6.0에서 LE Audio, 방향 탐지, 이소크로너스 채널 등 추가  

### 2) 주파수 대역과 채널 구조

Bluetooth는 모두 **2.4 GHz ISM 대역**을 사용하지만, **클래식 vs BLE**에서 채널 구조가 다릅니다.

#### (1) Classic Bluetooth (BR/EDR)

- 대역: 약 **2402 ~ 2480 MHz**
- **79개 RF 채널**, 채널 폭 **1 MHz**, 각 채널마다 중심 주파수 1 MHz 간격.  
- 기본 변조: **GFSK** (1 Mbps)
- EDR(Enhanced Data Rate)에서는 **π/4-DQPSK, 8-DPSK**로 2~3 Mbps 수준까지 증가.  

#### (2) Bluetooth Low Energy (BLE)

- 동일한 2.4 GHz 대역에 대해, **40개 채널 × 2 MHz**로 분할  
  - 대역: **2402 ~ 2480 MHz**, 각 채널 2 MHz 폭  
  - **광고 채널**: 3개 (index 37, 38, 39)  
  - **데이터 채널**: 37개 (index 0~36)  

- 변조: **GFSK** (Gaussian FSK)
- 여러 PHY 옵션:
  - **LE 1M PHY**: 1 Mbps (기본)
  - **LE 2M PHY**: 2 Mbps (고속)
  - **LE Coded PHY(S=2/S=8)**: 500 kbps / 125 kbps (FEC 포함, 장거리용)  

간단히 요약하면:

| 모드 | 채널 수 × 폭 | 변조 | 특징 |
|------|--------------|------|------|
| Classic BR/EDR | 79 × 1 MHz | GFSK(+DQPSK/DPSK) | 오디오, SPP 등 |
| BLE | 40 × 2 MHz | GFSK | 센서/IoT, 비콘, LE Audio |

#### (3) 주파수 도약(FHSS)과 AFH

Bluetooth는 **주파수 도약 스펙트럼 확산(FHSS)**을 사용합니다.

- 클래식: 초당 약 1600회 도약, 각 슬롯 길이 625 µs.  
- BLE: 광고/데이터 채널 집합에서 **채널 선택 알고리즘**으로 점프.  

Adaptive Frequency Hopping(AFh)는 **간섭이 심한 채널을 피하기 위해 채널맵을 동적으로 수정**합니다.  

수학적으로 보면, 어느 주어진 간섭 채널 집합 \(I\)를 제외한 유효 채널 수 \(N_{\text{good}}\)는

$$
N_{\text{good}} = N_{\text{total}} - |I|
$$

이고, AFH는 채널 선택 알고리즘에서 이 \(N_{\text{good}}\)만 사용해 도약 시퀀스를 구성합니다.

### 3) 네트워크 토폴로지: Piconet과 Scatternet

클래식 Bluetooth의 기본 토폴로지는 **마스터–슬레이브 기반 piconet**입니다.  

#### (1) Piconet

- 하나의 **마스터(master)** + 최대 **7개의 active slave**, 추가로 parked slave까지 포함 가능.
- 모든 장치는 마스터의 **시계(clock)**와 도약 시퀀스를 공유.
- 통신은 항상 **“마스터 ↔ 슬레이브”** 방향으로만, 슬레이브끼리 직접 통신은 없음.

간단 그림:

```text
          [ Master ]
          /   |   \
      S1      S2   S3      ...  (최대 S7)
```

현대 문서에서는 master/slave 대신 **primary/secondary** 또는 **central/peripheral** 같은 용어를 더 권장하지만, 많은 교과서와 표준에는 여전히 master/slave가 등장합니다.

#### (2) Scatternet

- 여러 piconet이 겹쳐 **Scatternet**을 형성할 수 있음.
- 어떤 노드는 한 piconet에서 master, 다른 piconet에서 slave 역할을 동시에 수행 가능.  

```text
Piconet A:  MA ─ S1 ─ S2
                         \
                          (브리지 노드)
                         /
Piconet B:  MB ─ S3 ─ S4
```

이 구조로 **멀티홉·다중 piconet** 구성이 가능하지만, 표준화된 라우팅 프로토콜은 없고, 연구/특수 시스템에서 주로 논의됩니다.

#### (3) BLE 토폴로지

BLE는 **역할(role)** 중심으로 설명합니다.  

- **Advertiser / Scanner**: 광고를 송신하는 장치 vs 광고를 수신(스캔)하는 장치.
- **Initiator / Peripheral**: 연결을 시작하는 장치 vs 연결을 수락하는 장치.
- **Central**: 한 번에 여러 Peripheral을 관리할 수 있는 장치 (사실상 “BLE 마스터”).

BLE는 논리적으로는 여전히 master–slave형이지만, **central/peripheral 용어**를 쓰고, 한 central이 많은 peripheral과 **동시에 연결**될 수 있도록 설계되었습니다.

### 4) Bluetooth Classic vs BLE vs LE Audio/5.x

버전과 용도 관점에서 정리하면:

| 타입 | 주요 버전 | 용도 | 특징 |
|------|-----------|------|------|
| BR/EDR (Classic) | 1.x~3.x, 5.x에도 유지 | 헤드셋, 스피커, 키보드, SPP 등 | 79×1 MHz, 최대 수 Mbps, 비교적 전력↑ |
| BLE | 4.0 이후 | 센서/IoT, 비콘, 스마트워치 | 40×2 MHz, 저전력, GATT 기반 |
| BLE 5.x | 5.0~5.4, 6.0 | 장거리, 고속, 방송 확장 | 2M, Coded PHY, 확장 광고, 방향탐지, LE Audio |
| LE Audio (5.2+) | 5.2~6.0 | 이어폰, 보청기, 브로드캐스트 오디오 | LC3 코덱, 이소크로너스 채널(ISOC), 멀티스트림  

#### 예제 시나리오: 스마트폰–이어폰–워치–센서

- **스마트폰**
  - BLE Central로 스마트워치, 심박 센서, 스마트 락 등과 다수 연결.
  - BR/EDR로 무선 이어폰과 A2DP 오디오 스트리밍 또는 LE Audio 멀티스트림.
- **스마트워치**
  - BLE Peripheral로 스마트폰과 연결.
  - 동시에 BLE Central로 심박 벨트, BLE 헤드폰 등과 연결.

이 구조에서 각 링크는 서로 다른 **연결 간격(connection interval)**, **PHY**, **보안 레벨**을 갖습니다.

---

## 15.3.2 Bluetooth Layers

Bluetooth 스택은 보통 **Controller vs Host**로 나누어 설명하고,  
그 위에 **프로파일(profiles)**이 올라가는 구조입니다.  

### 1) 전체 계층 개요

대부분의 문서에서는 다음과 같은 형태로 Bluetooth (LE 기준) 스택을 그립니다:

```text
┌────────────────────────────────────────────┐
│  Application / Profiles                   │ ← GATT 기반 서비스, A2DP, HID 등
├────────────────────────────────────────────┤
│  Host: GAP, GATT, L2CAP, SMP, RFCOMM, SDP │
├───────────────────────────────┬────────────┤
│         HCI (Host Controller Interface)   │
├───────────────────────────────┴────────────┤
│  Controller: Link Layer / Baseband, PHY   │
└────────────────────────────────────────────┘
```

- **Controller**: 기기 안의 RF·베이스밴드·링크 계층. 보통 SoC/모듈의 펌웨어.
- **Host**: 운영체제/펌웨어에서 돌아가는 상위 프로토콜(L2CAP, ATT/GATT, SMP 등).
- **HCI**: Host와 Controller를 잇는 표준 인터페이스 (UART, USB, SDIO 등).  

이제 각 계층을 아래에서 위로 살펴보겠습니다.

---

### 2) PHY / Radio Layer

#### (1) Classic vs LE 공통

- **2.4 GHz ISM 대역**
- FHSS 기반 채널 도약
- 인증 시험 규격(NI, Bluetooth SIG 문서)에 명시된 **송신 전력, 주파수 오차, EVM, BER 한계** 등을 만족해야 함.  

#### (2) BLE PHY 상세

앞에서 언급했듯이 BLE는 40 채널, 2 MHz 간격, **GFSK 변조**를 사용하고, 여러 PHY를 지원합니다.  

- LE 1M
- LE 2M
- LE Coded (S=2, S=8)

**거리–속도 트레이드오프**는 대략 아래와 같이 표현할 수 있습니다.

$$
R_{\text{eff}} \approx R_{\text{PHY}} \times (1 - p_{\text{FER}})
$$

- \(R_{\text{PHY}}\): 선택한 PHY의 표기 속도 (1M, 2M, 500k, 125k)
- \(p_{\text{FER}}\): 프레임 에러율(Frame Error Rate)

장거리 PHY에서는 \(R_{\text{PHY}}\)가 작아지지만 FEC 덕분에 \(p_{\text{FER}}\)이 작아져, “멀리서도 유효 데이터율”을 확보하는 것이 목적입니다.

---

### 3) Baseband / Link Layer (Classic & LE)

#### (1) Classic Baseband + Link Manager

클래식 Bluetooth에서 **Baseband + Link Manager Protocol (LMP)**는 다음을 담당합니다.  

- **접속 제어**
  - inquiry, paging, piconet 형성
  - master/slave 역할 배정, 주소 할당(Active Member Address, AM_ADDR)
- **링크 상태 관리**
  - active, sniff, hold, park 모드 등 (전력–지연 트레이드오프)
- **보안**
  - 링크 키, 암호화 설정
- **QoS**
  - 슬롯 예약, 패킷 타입 선택, 재전송 정책

#### (2) BLE Link Layer

BLE에서 **Link Layer(LL)**는 비슷한 역할을 하지만, 상태 머신이 다소 단순화되어 있습니다.  

- **상태**: Advertising, Scanning, Initiating, Connected
- **연결 이벤트(connection event)** 단위로 양쪽이 동시에 깨어나 통신
- **연결 간격(connection interval)**, **슬레이브 지연(slave latency)**, **supervision timeout** 등의 파라미터로 전력/지연/신뢰성을 조정

예를 들어, 연결 간격을 \(T_{\text{interval}}\), 슬레이브 지연을 \(L\)이라 하면, 슬레이브는 최대 \(L\)개 이벤트를 건너뛰고 \(T_{\text{interval}} \times (L+1)\)마다 한 번만 깨어나도 됩니다. 평균 전력은 대략

$$
P_{\text{avg}} \propto \frac{T_{\text{active}}}{T_{\text{interval}} \times (L+1)}
$$

로 떨어집니다(활성 시간 \(T_{\text{active}}\)가 고정이라고 가정하면).

---

### 4) HCI (Host Controller Interface)

**HCI**는 Host와 Controller 사이의 표준 인터페이스입니다.  

- 물리 인터페이스: UART, USB, SPI, SDIO 등
- HCI 패킷 유형:
  - HCI Command (Host→Controller)
  - HCI Event (Controller→Host)
  - ACL Data, SCO/ISO Data 등

#### 예제 코드: Python에서 BLE 디바이스 스캔

아래 예제는 리눅스/윈도우에서 널리 쓰이는 **`bleak`** 라이브러리를 사용해 BLE 장치를 스캔하는 코드입니다 (실행 환경에 따라 root 권한, BLE 어댑터가 필요합니다).

```python
import asyncio
from bleak import BleakScanner

async def scan():
    print("BLE 장치 스캔 중...")
    devices = await BleakScanner.discover(timeout=5.0)
    for d in devices:
        print(f"{d.address}  RSSI={d.rssi}  name={d.name}")

if __name__ == "__main__":
    asyncio.run(scan())
```

이 코드의 내부에서는 OS의 Bluetooth 스택이 **HCI 명령(HCI_LE_SET_SCAN_PARAMETERS, HCI_LE_SET_SCAN_ENABLE 등)**을 컨트롤러로 보내고, 컨트롤러가 수신한 광고 패킷을 HCI 이벤트로 다시 올려줍니다.

---

### 5) L2CAP (Logical Link Control and Adaptation Protocol)

**L2CAP**는 Bluetooth의 “논리 링크 계층/적응 계층”입니다. 클래식과 LE 모두에서 사용됩니다.  

주요 역할:

1. **다중화 & 디멀티플렉싱**
   - 하나의 물리 링크 위에 여러 **논리 채널(PSM, CID)**을 올림.
   - 예: BR/EDR에서 RFCOMM, SDP, AVDTP 등을 구분; LE에서 ATT, SMP 등을 구분.
2. **MTU 관리 & 세그먼트/재조립(SDU/PDUs)**
   - 상위 계층이 보내는 서비스 데이터 단위(SDU)를 L2CAP PDU로 분할/재조립.
3. **QoS, 흐름 제어/재전송 (클래식에서)**
   - ERTM, Streaming 모드 등.

#### 예제: BLE에서 ATT와 SMP을 L2CAP 위에 올리는 구조

BLE에서 L2CAP 위에는 보통 다음과 같은 채널이 올라갑니다.  

| PSM / CID | 프로토콜 | 역할 |
|-----------|----------|------|
| 0x0004    | ATT      | GATT 읽기/쓰기, 알림/인디케이션 |
| 0x0006    | SMP      | 페어링, 키 교환, 보안 |
| 0x0005 등 | L2CAP Credit-based | CoC 기반 데이터 채널 |

즉, 한 BLE 연결 안에서 **“ATT 채널”과 “SMP 채널”이 동시에 존재**하고, L2CAP이 이들을 구분해줍니다.

---

### 6) Classic 상위 프로토콜: RFCOMM, SDP, AVDTP/AVCTP …

클래식 Bluetooth(브로드밴드/오디오)는 다양한 상위 프로토콜을 사용합니다.  

- **SDP(Service Discovery Protocol)**
  - “이 장비가 제공하는 서비스(프로파일)는 무엇인가?”
  - 각 서비스는 **UUID와 프로토콜 스택(예: L2CAP→RFCOMM→프로토콜)**으로 기술됨.
- **RFCOMM**
  - “가상 시리얼 포트” 프로토콜. RS-232를 에뮬레이션.
- **AVDTP / AVCTP**
  - A2DP(오디오 스트리밍), AVRCP(리모컨) 등을 위한 Audio/Video 전송 및 제어.
- 기타 HID, BNEP, OBEX 등.

#### 예제 시나리오: 클래식 SPP를 통한 시리얼 통신

센서 모듈이 클래식 Bluetooth SPP(Serial Port Profile)를 제공하고, PC가 이를 통해 데이터를 읽는 시나리오를 생각해봅시다.

1. PC는 디바이스 검색(inquiry)을 통해 센서를 찾음.
2. SDP로 “SPP 서비스”를 질의 → RFCOMM 채널 번호를 얻음.  
3. L2CAP 위에 RFCOMM 채널을 열고, 그 위에 TCP 같은 바이트 스트림 인터페이스를 사용.
4. 애플리케이션은 그냥 “시리얼 포트(COMx)”처럼 사용.

이 경우 스택은 대략:

```text
Application(센서 로거)
  ↓
RFCOMM
  ↓
L2CAP
  ↓
Baseband / Link Manager
  ↓
RF (2.4 GHz)
```

---

### 7) BLE 상위 프로토콜: ATT, GATT, GAP, SMP

BLE의 핵심은 **“GATT 기반 서비스 모델”**입니다.  

#### (1) GAP (Generic Access Profile)

- 장치 발견, 광고, 스캔, 연결 수락/요청, 이름 노출 등 **기본 동작 정의**.
- GAP 역할: Broadcaster, Observer, Peripheral, Central.

#### (2) ATT (Attribute Protocol) & GATT (Generic Attribute Profile)

- ATT: **속성(attribute)** 단위의 간단한 요청/응답 프로토콜.
  - 각 attribute는 Handle, Type(UUID), Value를 가짐.
- GATT: ATT 위에서 **서비스/캐릭터리스틱/디스크립터 구조** 정의.

예를 들어, 간단한 배터리 서비스:

```text
Service "Battery Service" (UUID 0x180F)
  └─ Characteristic "Battery Level" (UUID 0x2A19)
        Properties: Read, Notify
        Value: 0~100 (%)
```

클라이언트(스마트폰)는:

1. **Service Discovery**로 0x180F 서비스 존재 확인.
2. 해당 서비스 내 0x2A19 캐릭터리스틱 찾기.
3. READ 요청으로 현재 배터리 값 읽기.
4. NOTIFY 활성화로 향후 변화 시 알림 수신.

#### (3) SMP (Security Manager Protocol)

- BLE 보안: 페어링, 인증, 키 교환(장기 키, 세션 키, 확인 값) 담당.  
- 페어링 모드: Just Works, Passkey, Numeric Comparison 등.
- IoT 장치에서는 이 단계를 잘못 설정하면 **암호화되지 않은 평문 BLE 트래픽**이 되어버리므로 매우 중요.

#### 예제 시나리오: BLE 배터리 서비스 읽기

스마트폰 앱에서 BLE 배터리 서비스를 읽는 과정을 간단한 의사코드로 표현해보면:

```python
from bleak import BleakClient

BATTERY_SERVICE = "0000180F-0000-1000-8000-00805f9b34fb"
BATTERY_LEVEL   = "00002A19-0000-1000-8000-00805f9b34fb"

async def read_battery(addr):
    async with BleakClient(addr) as client:
        # 1. 연결 (GAP에서 Peripheral과 연결)
        services = await client.get_services()
        # 2. GATT 서비스/캐릭터리스틱 탐색
        for s in services:
            if s.uuid == BATTERY_SERVICE:
                for c in s.characteristics:
                    if c.uuid == BATTERY_LEVEL:
                        value = await client.read_gatt_char(c)
                        print("Battery level:", int(value[0]), "%")
```

내부적으로는:

- GAP: 연결 설정
- L2CAP: ATT/SMP 채널을 유지
- SMP: 페어링/암호화 (필요 시)
- ATT: `Read Request` / `Read Response` PDU로 값 교환
- GATT: 이를 “배터리 레벨”이라는 의미 있는 개체로 해석

---

### 8) LE Audio와 Isochronous Channels (5.2~6.0)

Bluetooth 5.2에서 **LE Isochronous Channels(ISOC)**가 도입되면서, BLE 위에서 **LE Audio**가 가능해졌습니다.  

- **CIS(Connected Isochronous Stream)**: 연결 기반 이소크로너스 스트림
- **BIS(Broadcast Isochronous Stream)**: 브로드캐스트 오디오(하나의 송신자가 여러 수신자에게 동기화된 오디오 전송)

여기서 이소크로너스(isochronous)는 **“시간에 민감한 데이터가 일정한 주기와 지연 한도 내에서 도착해야 함”**을 의미합니다.  
LE Audio는 전통적인 SCO 링크 대신 BLE 위에서 **LC3 코덱**과 ISOC를 사용해, **더 적은 비트레이트로 더 좋은 품질 + 멀티스트림 지원**을 제공합니다.

수학적으로, 이소크로너스 스트림은 대략

$$
T_{\text{interval}},\ D_{\text{max}},\ L_{\text{packet}}
$$

(전송 간격, 허용 최대 지연, 패킷 길이)의 조합으로 정의되며,  
지연 보장이 필요한 오디오 애플리케이션에서 **버퍼 크기와 드롭 정책** 설계의 핵심 파라미터가 됩니다.

---

### 정리

- **Architecture**
  - 2.4 GHz ISM 대역, **classic(79×1 MHz)** vs **BLE(40×2 MHz)** 채널 구조.
  - 주파수 도약(FHSS)과 AFH(Adaptive Frequency Hopping)으로 간섭에 강한 설계.
  - **Piconet(1 master + 최대 7 slaves)**, 여러 piconet의 **Scatternet** 구조.
  - BLE에서는 **central/peripheral, advertiser/scanner**로 역할 정리, 5.x/6.0에서 LE Audio, 방향 탐지, 장거리/고속 모드 추가.

- **Bluetooth Layers**
  - 스택은 **Controller(PHY/LL) – HCI – Host(L2CAP, ATT/GATT, SMP, RFCOMM, SDP, 프로파일)**로 구성.
  - Classic 상위 계층: RFCOMM(가상 시리얼), SDP(서비스 검색), AVDTP/AVCTP(오디오/비디오).
  - BLE 상위 계층: GAP(접근), ATT/GATT(데이터 모델), SMP(보안).
  - 5.2+에서는 **Isochronous Channels + LE Audio**로 BLE 위에서 고급 오디오 전달.

이 구조를 이해하면, **Wi-Fi(802.11)와 Bluetooth가 2.4 GHz에서 어떻게 공존하는지**,  
그리고 실제 코드(예: `bleak`, 모바일 OS API, 펌웨어 스택)에서 어떤 계층을 다루는지 훨씬 명확해집니다.  
실무에서는 이 계층 구조를 바탕으로 **전력–지연–신뢰성–대역폭** 트레이드오프를 설계하는 것이 핵심입니다.