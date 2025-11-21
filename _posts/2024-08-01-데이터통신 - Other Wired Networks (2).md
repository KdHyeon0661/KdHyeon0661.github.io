---
layout: post
title: 데이터 통신 - Other Wired Networks (2)
date: 2024-08-01 20:20:23 +0900
category: DataCommunication
---
# Chapter 14. Core Synchronous Networks — SONET & ATM

- **SONET (Synchronous Optical Network)**: 광섬유 기반 **동기식 TDM 전송 표준**
- **ATM (Asynchronous Transfer Mode)**: 고정 길이 셀 기반 **셀 스위칭·가상회선** 전송 표준

둘 다 오늘날에는 **IP/MPLS + Ethernet + OTN**에 상당 부분 자리를 내줬지만, 여전히 교과서·자격시험·실제 레거시망에서는 중요하게 등장합니다.

---

## SONET

### Architecture

#### SONET이란?

- **SONET(Synchronous Optical Network)**
  - 북미(ANSI)에서 표준화된 **광 전송 프레임·계층 구조**
  - 기본 전송 단위: **STS-1/OC-1 = 51.84 Mbps**
  - 상위 계층(전통 PDH: T1/E1/DS3, ATM, IP, Ethernet 등)을 **투명하게 싣고 장거리 전송**하는 역할

- 국제적으로는 거의 동일한 개념의 **SDH(Synchronous Digital Hierarchy)**가 ITU-T에서 표준화
  - SONET: **STS-N / OC-N** (미국식)
  - SDH: **STM-N** (유럽·국제)
  - 예: STS-3/OC-3 ≒ STM-1 = 155.52 Mbps

#### “코어 전송망”에서의 위치

간단한 계층 그림:

```text
[사용자 단말] -- [xDSL/케이블/FTTH/이더넷 액세스] -- [지역 POP/CO]
          \                                   /
           ----------(집선·집중)-------------
                            |
                      [SONET/SDH 링/메시]
                            |
                        [국가 백본]
                            |
                        [해저케이블/국제망]
```

- 액세스망(DSLAM, CMTS, OLT 등) → **집선 노드** → 그 사이를 **SONET 링/포인트 투 포인트**가 연결
- 상위에는
  - 예전엔 **ATM·Frame Relay·전용회선**
  - 지금은 **IP/MPLS, Carrier Ethernet, OTN** 등이 올라탐

#### SONET 계층의 큰 방향

SONET은 다음을 목표로 설계되었습니다.

1. **단일한 프레임 구조와 라인 속도**
   - PDH(DS1, DS3, E1, E3 등)의 복잡한 계층을 하나의 동기식 계층으로 통합
2. **강력한 동기·관리 기능**
   - 네트워크 전체에 Stratum-3 이상 기준 클럭 분배, 알람/성능 모니터링, 보호 절체
3. **고속 다중화·복구**
   - 51.84 Mbps를 기본으로 155 Mb/s, 622 Mb/s, 2.5 Gb/s, 10 Gb/s … 계층적으로 확장
   - 링에서 **50 ms 이내 보호 절체**(전통 통신망 설계 목표)

---

### SONET Layers

SONET에는 **“기술 계층(Photonic~Path)”**이 존재합니다. OSI 계층과는 다르지만, 기능을 구분하는 데 유용합니다.

| 계층         | 범위                          | 대표 역할 |
|--------------|-------------------------------|-----------|
| Photonic     | 광섬유·레이저·수광기          | 광 신호의 전기/광 변환, 파장, 전력 |
| Section      | 인접한 SONET 장비 간 링크     | 프레임 동기, BIP 에러 측정, 구간 알람 |
| Line         | 여러 섹션을 포함하는 라인 구간 | 다중화, 보호 절체, 포인터 처리 |
| Path         | 종단 간(사용자 페이로드 경로) | 페이로드 모니터링, 경로 수준 알람/상태 |

간단 그림:

```text
[Payload: DS3/ATM/Eth]  <-- Path
        ↓
[STS-1/STS-N SPE]       <-- Path
        ↓
[Transport Overhead]    <-- Line + Section
        ↓
[광 신호 (OC-N)]        <-- Photonic
```

- **Section Overhead**: 바로 이웃한 장비끼리 “프레임 보이냐?”, “에러율 얼마냐?” 확인
- **Line Overhead**: 여러 섹션을 아우르는 라인 단위에서 **보호 절체(UPSR/BLSR)**, 다중화 관리
- **Path Overhead**: 실제 사용자 페이로드(예: DS3, ATM VC) 흐름을 **종단 간** 모니터링

---

### SONET Frames

SONET의 기본 신호는 **STS-1**입니다. 그 구조를 정확하게 이해하는 것이 SONET의 핵심입니다.

#### STS-1 프레임 구조

- 프레임 크기: **9행 × 90열 = 810 바이트**
- 프레임 주기: **125 µs (초당 8000 프레임)**
- 비트율 계산:

$$
\text{bit rate} = 9 \times 90 \times 8 \times 8000
= 51{,}840{,}000\ \text{bps}
= 51.84\ \text{Mbps}
$$

프레임 도식(개념):

```text
90 columns →
┌───────────────────────────────────────────────────────────────┐
│TOH (3 bytes) │             STS-1 Envelope Capacity           │ row 1
├──────────────┼───────────────────────────────────────────────┤
│TOH           │                  (SPE)                        │ row 2
├──────────────┼───────────────────────────────────────────────┤
│ ...          │                                               │
├──────────────┼───────────────────────────────────────────────┤
│TOH           │                                               │ row 9
└───────────────────────────────────────────────────────────────┘
↑
9 rows
```

- **앞 3열(각 9바이트)**: **Transport Overhead(TOH)** = Section + Line Overhead
- **나머지 87열**: **Envelope Capacity = SPE + Path Overhead + Payload**

#### SPE(Synchronous Payload Envelope)

- STS-1 SPE: **9행 × 87열 = 783 바이트**
  - 이 중 일부(1열)는 **Path Overhead(POH)**
  - 나머지가 **Payload**
- STS-1 payload 용량으로는
  - **28 × DS1(1.544 Mbps)**
  - 또는 **1 × DS3(44.736 Mbps)**
  - 또는 **21 × 2.048 Mbps(E1) 수준의 신호**까지 수용 가능하도록 설계되었습니다.

#### 간단 계산 예제 (파이썬 코드)

STS-1, STS-3, STS-12 등의 비트율을 계산하는 코드 예시:

```python
def sonet_rate(sts_n):
    rows = 9
    cols_per_sts1 = 90
    bits_per_byte = 8
    frames_per_sec = 8000
    bits_per_frame_sts1 = rows * cols_per_sts1 * bits_per_byte
    return sts_n * bits_per_frame_sts1 * frames_per_sec

for n in [1, 3, 12, 48, 192]:
    rate_mbps = sonet_rate(n) / 1e6
    print(f"STS-{n}: {rate_mbps:.3f} Mbps")
```

실제로는 표준이 다음과 같이 정의되어 있습니다.

| 전기 계층(SONET) | 광 계층(SONET) | SDH 계층 | 비트율 |
|------------------|----------------|----------|--------|
| STS-1            | OC-1           | STM-0    | 51.84 Mbps |
| STS-3            | OC-3           | STM-1    | 155.52 Mbps |
| STS-12           | OC-12          | STM-4    | 622.08 Mbps |
| STS-48           | OC-48          | STM-16   | 2.488 Gbps |
| STS-192          | OC-192         | STM-64   | 9.953 Gbps |

---

### STS Multiplexing

**STS-N**은 **N개의 STS-1을 byte-interleaving**해서 만들며, 프레임 주기는 그대로 125 µs입니다.

#### Byte-interleaving

STS-3 예:

- 각 STS-1에서 **바이트 1씩 번갈아 가져와** 3배 폭의 프레임(270열)을 만듦
- 프레임 구조:

```text
STS-1 A: A1, A2, A3, ...
STS-1 B: B1, B2, B3, ...
STS-1 C: C1, C2, C3, ...

STS-3 : A1, B1, C1, A2, B2, C2, ...
```

그래서

- STS-1: 9 × 90 바이트, 51.84 Mbps
- STS-3: 9 × 270 바이트, 155.52 Mbps (3 × 51.84)
- STS-N: **N × 51.84 Mbps**

#### 설계 의도

- 모든 STS-N이 **동일한 125 µs 프레임 주기**를 유지 → 동기식 다중화가 매우 단순
- 상위/하위 계층(DS1, DS3, ATM, Ethernet 등)을 “바이트 마음대로 섞어 넣고, 다시 빼내는” 작업이 구현하기 쉬움
- 하위 장비(멀티플렉서, 크로스커넥트)가 **단순한 byte-switching 하드웨어**로도 구현 가능

#### 간단 예제: 음성 채널 수

STS-1은 DS0(64 kbps) 음성 채널 기준으로 **672 채널**까지 실을 수 있습니다.

- DS0 = 64 kbps
- STS-1 = 51.84 Mbps
- \(\frac{51.84\ \text{Mbps}}{64\ \text{kbps}} = 810\)이지만, 오버헤드·정렬 비트 등 때문에 실제 DS0 채널 수는 672개

이 값들을 코드로 계산하는 예:

```python
sts1_rate = 51.84e6
ds0_rate = 64e3
ideal_ds0 = sts1_rate / ds0_rate
print("이론상 DS0 개수:", ideal_ds0)
print("SONET에서 실제 지원 DS0 수: 대략 672개")
```

---

### SONET Networks

SONET은 단순한 포인트 투 포인트 링크뿐 아니라, **복구·용량 확장에 최적화된 링(topology)** 구조로 많이 사용되었습니다.

#### 기본 토폴로지

1. **Point-to-Point**
   - 두 노드 간 단순 OC-N 링크 (예: 데이터센터–전화국 사이)
2. **링(Ring)**
   - **UPSR(Unidirectional Path-Switched Ring)**
   - **BLSR(Bidirectional Line-Switched Ring)**
   - 2-fiber 혹은 4-fiber 구조
3. **Mesh + SONET**
   - IP/MPLS/OTN과 함께 혼합 메시 구조를 이루기도 함

대표적으로, **대도시를 도는 OC-192 링**에 여러 지역 노드(전화국, MSC, 라우터)가 접속되어 트래픽을 실어 나르는 형태가 고전적인 설계입니다.

#### UPSR vs BLSR (개념적 비교)

간단 ASCII 그림(2-fiber 링 기준):

```text
        [Node A]
         /    \
    OC-N      OC-N
       /        \
 [Node D] --- [Node B]
          OC-N
           |
        [Node C]
```

- **UPSR** (경로 기준 보호)
  - 동일한 페이로드를 **두 방향**으로 동시에 전송
  - 수신 측에서 “더 좋은” 신호를 선택
  - 구현은 간단하지만, 대역폭 효율이 떨어짐(2배 사용)
- **BLSR** (라인 기준 보호)
  - 일정 비율(예: 1/2)을 보호용으로 비워둠
  - 장애 시 스위치가 해당 구간 트래픽만 우회
  - 대역폭 효율이 더 좋고 대형 링에 적합

SONET 설계 목표: **50 ms 이내 보호 절체**
→ 두~네 홉 이내에서 광/섹션/라인 수준 알람을 보고 즉시 스위칭.

#### 간단 시나리오 예제

- 4노드 링: A–B–C–D–A, 4-fiber BLSR, OC-48
- A↔C 간 STS-12급 트래픽이 흐르는 중
- B–C 구간 광단선(스플라이싱 실패 등) 발생

동작:

1. B–C 구간 Section/Line Overhead에서 LOS, AIS, RDI 등 장애 알람 발생
2. B, C 양쪽 노드가 이를 감지하고 **링 보호 상태로 전환**
3. A–B–C–D–A 중 B–C 구간을 건너뛰고
   - A–D–C 방향으로 STS-12를 재라우팅
4. 전체 과정이 50 ms 이내에 이루어져 상위 ATM/IP 계층은 짧은 지터 외에는 거의 영향 없음

---

### Virtual Tributaries (VT)

사용자 질문에 나온 “visual tributaries”는 사실 **Virtual Tributaries(VT)**를 말합니다. SONET에서 **PDH 계층(DS1, E1 등)을 수용하기 위한 작은 컨테이너**입니다.

#### VT의 역할

- STS-1의 SPE에는 **“VT Group”**들이 자리 잡고 있고,
- 각 VT Group 안에 **VT1.5, VT2, VT3, VT6** 같은 소형 페이로드 슬롯이 들어 있습니다.
- 이를 통해 SONET은 다음과 같은 레거시 신호를 효율적으로 실을 수 있습니다.
  - **T1/DS1 (1.544 Mbps) → VT1.5**
  - **E1 (2.048 Mbps) → VT2**

GL 통신 기술 개요에 따르면, STS-1의 payload에는 **7개의 VT 그룹**이 있으며, 각 VT 그룹은 여러 VT1.5/VT2/VT3/VT6 조합으로 구성될 수 있습니다.

#### VT 종류와 용량(대략값)

| VT 타입 | 대략 용량(전송률) | 매핑되는 대표 PDH |
|---------|-------------------|-------------------|
| VT1.5   | 약 1.7 Mbps       | T1/DS1 (1.544 Mbps) |
| VT2     | 약 2.3 Mbps       | E1 (2.048 Mbps)   |
| VT3     | 약 3.5 Mbps       | DS1C 등 일부 신호 |
| VT6     | 약 6 Mbps         | DS2급 신호        |

정확한 값은 SONET 포인터·정렬 오버헤드 등을 포함해 정의되어 있지만, 개념상 “해당 PDH 용량을 조금 여유 있게 담는 작은 슬롯” 정도로 이해하면 충분합니다.

#### VT 다중화 예제

- 하나의 STS-1에는 **7 VT 그룹(VTG)**이 들어간다고 가정
- 각 VTG에는 **4개 VT1.5**를 넣을 수 있다고 하면

```text
STS-1 SPE
 ├─ VTG1: VT1.5 × 4  → T1 × 4
 ├─ VTG2: VT1.5 × 4  → T1 × 4
 ...
 └─ VTG7: VT1.5 × 4  → T1 × 4
합계: T1 × 28 (DS1 × 28)
```

이 구성이 교과서에서 자주 언급되는 **“STS-1은 DS1 28개를 싣는다”**라는 사실과 연결됩니다.

#### 코드 예제: VT로 음성 채널 수 계산

```python
def ds0_channels_from_vt(ds1_per_vt, vtg_per_sts1=7, vt_per_vtg=4, ds0_per_ds1=24):
    total_ds1 = vtg_per_sts1 * vt_per_vtg * ds1_per_vt
    return total_ds1 * ds0_per_ds1

print("STS-1에서 이론상 DS0 채널 수:", ds0_channels_from_vt(1))
```

실제 구현은 훨씬 복잡하지만, 이 정도 계산만으로도 “VT → DS1/E1 → DS0”로 이어지는 계층이 어떻게 쌓이는지 감이 옵니다.

---

### 마무리: SONET의 현재 위치

- SONET/SDH는 한때 **전 세계 통신 사업자 백본의 표준**이었지만,
- 현재는 **IP/MPLS + Carrier Ethernet + OTN**이 백본을 지배하고 있고, SONET/SDH는
  - 레거시 TDM 서비스,
  - 일부 전용회선,
  - 혹은 OTN/이더넷망의 하부 전송계층으로만 남는 추세입니다.

그래도 **프레임 구조·STS 계층·VT·링 보호** 개념은 여전히 시험과 설계에서 자주 등장하므로, 논리 구조를 정확히 이해하는 것이 중요합니다.

---

## ATM (Asynchronous Transfer Mode)

이제 SONET 위에서 **“셀 스위칭”**을 했던 대표 기술, **ATM**을 살펴보겠습니다.

### Design Goals

ATM은 1980년대 말 **B-ISDN(Broadband ISDN)**을 위해 설계된 기술로, 목표는 대략 다음과 같았습니다.

1. **하나의 네트워크로 음성·영상·데이터 통합**
   - 회선 교환(전화)과 패킷 데이터(IP)를 하나의 백본에서 처리
2. **엄격한 QoS와 낮은 지터**
   - 음성·실시간 영상에 필요한 **지연·지터 제어**
3. **하드웨어 친화적인 고정 길이 셀**
   - 라우터보다 단순한 **셀 스위치**로 고속 구현 가능
4. **가상회선 기반 연결형(Connection-Oriented) 모델**
   - 각 서비스마다 **트래픽 계약(traffic contract)**을 맺고,
     CBR/VBR/ABR/UBR 등 서비스 클래스로 품질 보장
5. **SONET/SDH와의 긴밀한 결합**
   - 코어 전송 계층(SONET/SDH) 위에 자연스럽게 올릴 수 있도록 설계

#### 53바이트 고정 길이 셀

ATM의 가장 유명한 특징: **고정 길이 53바이트 셀**
- 헤더: 5바이트
- 페이로드: 48바이트

왜 48바이트였나?

- 미국 측: **64바이트** 페이로드 선호 (데이터 효율)
- 유럽 측: **32바이트** 선호 (음성 지연·에코 제거 부하 감소)
- 결국 **48바이트**로 타협 → 6 ms 정도의 음성 데이터(64 kbps 기준)를 담을 수 있음
- 여기에 헤더 5바이트를 붙여 **53바이트 셀**로 표준화

수학적으로 보면,

- 64 kbps 음성 스트림은 초당 8000샘플 × 8비트 = 64,000 bps
- 한 셀 페이로드 48바이트 = 384비트
- 음성 한 셀에 해당하는 시간:

$$
\frac{384\ \text{bits}}{64{,}000\ \text{bps}} = 0.006\ \text{s} = 6\ \text{ms}
$$

6 ms 지연 단위는 에코·버퍼 관리 측면에서 현실적인 타협점으로 간주되었습니다.

#### “Asynchronous Transfer”의 의미

- 기존 TDM은 각 채널에 **고정 타임슬롯**을 제공 → 비어 있어도 슬롯 유지
- ATM은 **“필요할 때만 셀을 보낸다”**는 의미에서 **비동기 시분할(Asynchronous TDM)**
  - 트래픽이 없으면 셀을 보내지 않음
  - 다만, **가상회선(VP/VC)** 단위로 대역폭·지연 특성을 계약

#### QoS·서비스 클래스

ATM은 설계 단계부터 다양한 트래픽 클래스를 정의했습니다.

- **CBR(Constant Bit Rate)**: 전화, TDM 에뮬레이션, 정해진 비트율
- **VBR(Variable Bit Rate)**: 실시간/비실시간 영상·오디오
- **ABR(Available Bit Rate)**: 최소 속도 보장 + 네트워크 가용 대역에 따라 조절
- **UBR(Unspecified Bit Rate)**: 베스트 에포트, 특별한 보장 없음

각 VC는 **PCR, SCR, MBS, CDVT**와 같은 파라미터로 트래픽 계약을 맺고, 스위치는 **GCRA(Leaky Bucket 기반 policing)**로 이를 감시합니다.

#### 예제 시나리오 (1990년대 전형적인 ATM 백본)

한 유럽 통신사의 코어망:

- **엑세스**: xDSL, 프레임릴레이, ISDN
- **집선**: ATM DSLAM, ATM Edge Switch
- **코어**: ATM 스위치 + SONET/SDH 전송

서비스:

- 음성: CBR VC (64 kbps × 다수 채널)
- 화상회의: rt-VBR VC (피크/평균 지정)
- IP 데이터: UBR VC (베스트 에포트)

이렇게 모든 것을 ATM으로 통합해 관리하는 것이 목표였고, 실제로 1990년대~2000년대 초반까지 많은 사업자 백본이 ATM을 사용했습니다.

---

### Problems (현실적인 문제점)

이론적으로는 매우 “깨끗한” 설계였지만, 실전에서는 여러 문제를 드러내며 결국 **Ethernet/IP/MPLS에 밀려났습니다.**

#### 셀 오버헤드(Cell Tax)

- 한 셀: 53바이트 = 5바이트 헤더 + 48바이트 페이로드
- 헤더 비율:

$$
\frac{5}{53} \approx 9.43\%
$$

- 여기에 AAL5 등 상위 적응 계층의 트레일러까지 포함하면
  → IP 패킷 입장에서 **10% 이상 오버헤드** 발생

**예제: 1500바이트 IP 패킷을 AAL5로 전송**

- 1500바이트 + AAL5 트레일러(8바이트) = 1508바이트
- 셀당 48바이트 → 필요한 셀 수:

$$
\left\lceil \frac{1508}{48} \right\rceil
= \left\lceil 31.416\dots \right\rceil
= 32\ \text{셀}
$$

- 실제 전송되는 데이터: 32 × 53 = 1696바이트
- 오버헤드 비율:

$$
\frac{1696 - 1500}{1696}
\approx 11.6\%
$$

즉, **IP 기준 1500바이트 프레임마다 11% 정도 “셀 세금”**을 내야 했습니다.

이 계산을 파이썬으로 표현하면:

```python
payload = 1500
aal5_trailer = 8
cell_payload = 48
cell_size = 53

padded = payload + aal5_trailer
cells = (padded + cell_payload - 1) // cell_payload
total_bytes = cells * cell_size
overhead_ratio = (total_bytes - payload) / total_bytes

print("필요 셀 수:", cells)
print("전송 바이트:", total_bytes)
print("오버헤드 비율:", overhead_ratio * 100, "%")
```

#### 복잡한 스택과 운영

- **AAL1/2/3/4/5**, UNI/NNI, PNNI, 트래픽 계약, GCRA policing 등
- IP/Ethernet에 비해 스택이 복잡하고, 장비·운영 인력 비용이 높음
- 각 계층(ATM·SONET·PDH·IP)의 OAM 기능이 중복되거나 충돌

#### Ethernet·IP의 급격한 성장

- 1990년대 후반~2000년대 초:
  - Fast Ethernet(100 Mbps), **Gigabit Ethernet(1 Gbps)** 등장
  - 나중에는 10G/40G/100G Ethernet까지 보급
- 이 속도 영역에서는
  - **1500바이트 프레임 전송 지연(μs 단위)**이 매우 짧아져
  - ATM이 줄이려고 노력하던 “지터” 문제가 상대적으로 덜 심각해짐
- 동시에 **IP/MPLS가 QoS·트래픽 엔지니어링**을 점차 잘 지원하면서
  - “굳이 ATM까지 써야 할 이유가 줄어든 것”이 큰 이유 중 하나였습니다.

#### Overlay 구조의 비효율

실제 네트워크는 흔히 다음과 같았습니다.

```text
[IP 라우터] --[ATM PVC]-- [ATM 스위치] --[ATM PVC]-- [IP 라우터]
                |
              [SONET 링]
```

- IP 라우팅 + ATM VC + SONET 경로 + PDH 계층까지 겹쳐있는 구조
- 장애·업그레이드 시 어느 계층에서 제어해야 하는지가 모호
- IP/MPLS·Carrier Ethernet은 이 계층들을 통합·단순화하는 방향으로 설계됨

---

### Architecture

그래도 ATM의 아키텍처 자체는 시험·이론에서 중요한 포인트입니다.

#### ATM 레퍼런스 계층

ATM은 다음과 같이 3층 구조로 보는 것이 일반적입니다.

1. **Physical Layer**
   - 실제 전송 매체(동축, 구리, 광섬유)
   - 보통 **SONET/SDH, PDH, xDSL** 위에서 돌아감
2. **ATM Layer**
   - 53바이트 셀을 스위칭/다중화
   - **헤더(VPI/VCI, PT, CLP, HEC)** 처리
3. **AAL(ATM Adaptation Layer)**
   - 상위 트래픽(IP, 음성, 영상, 프레임릴레이 등)을 셀로 쪼개고(SAR) 다시 조립
   - AAL1/2: 음성·실시간 서비스, AAL5: IP/데이터

OSI와의 대략적 대응:

- Physical ≒ OSI L1
- ATM Layer ≒ OSI L2 일부
- AAL ≒ L3/L4에 걸쳐 있음 (엄밀한 대응은 아님)

#### 셀 포맷 (UNI 기준)

UNI(사용자–네트워크 인터페이스) 셀 헤더 구조:​

```text
총 53바이트
┌─────────────────────────────────────────────────────┐
│ GFC (4) │ VPI (8)                                  │ 1바이트
├─────────────────────────────────────────────────────┤
│ VPI (4) │ VCI (4 비트 상위)                         │ 1바이트
├─────────────────────────────────────────────────────┤
│ VCI (8)                                           │ 1바이트
├─────────────────────────────────────────────────────┤
│ VCI (4 비트 하위) │ PT (3) │ CLP (1)               │ 1바이트
├─────────────────────────────────────────────────────┤
│ HEC (8)                                          │ 1바이트
├─────────────────────────────────────────────────────┤
│ Payload (48바이트)                                │
└─────────────────────────────────────────────────────┘
```

- **GFC(Generic Flow Control, 4bit)**: 공유 매체용 로컬 흐름제어(현실에서는 거의 사용 안 함, 0000 고정)
- **VPI(Virtual Path Identifier)**: 가상 경로 번호 (UNI 8bit, NNI 12bit)
- **VCI(Virtual Channel Identifier)**: 가상 채널 번호 (16bit)
- **PT(Payload Type)**: 사용자 데이터 셀인지, 관리 셀인지, 혼잡 플래그(EFCI) 등
- **CLP(Cell Loss Priority)**: 0=높은 우선, 1=먼저 버려도 되는 셀
- **HEC(Header Error Control)**: 헤더용 CRC (단일 비트 오류 정정, 다중 오류 검출에 사용)

#### Virtual Path / Virtual Channel

ATM은 **두 단계의 가상 회선**을 사용합니다.

- **VP(Virtual Path)**: 여러 개의 VC를 묶은 상위 개념
  - VPI 값으로 식별
  - 백본에서는 주로 VP 단위로 라우팅·대역폭 관리
- **VC(Virtual Channel)**: 실제 서비스·연결 단위
  - VCI 값으로 식별 (VPI와 함께)

스위칭 동작(레이블 스와핑과 유사):

```text
[입력 포트, VPI=3, VCI=100] --(스위치 룩업)
→ [출력 포트 2, VPI=5, VCI=20]
```

이 정보는 스위치 내부의 **연결 테이블**에 저장되며, 셀이 통과할 때마다 VPI/VCI를 바꿔 끼우는 방식으로 전송됩니다.

#### ATM 서비스 클래스와 트래픽 계약

각 VC는 생성 시 다음과 같은 파라미터를 네트워크와 협상합니다.

- **PCR(Peak Cell Rate)**
- **SCR(Sustainable Cell Rate)**
- **MBS(Maximum Burst Size)**
- **CDVT(Cell Delay Variation Tolerance)**

네트워크는 **UPC/NPC(Usage/Network Parameter Control)**로 이 계약을 감시하며, GCRA(Leaky Bucket)를 이용해:

- 계약 위반 셀에 **CLP 비트=1**을 세팅하거나
- 아예 셀을 드롭(drop)할 수 있습니다.

##### 간단 예제: CBR 음성 + VBR 영상 + UBR 데이터

하나의 ATM 링크(155 Mbps)에서 세 가지 VC가 있다고 가정:

1. VC1: 음성 CBR, PCR = 2 Mbps
2. VC2: 영상 rt-VBR, PCR = 20 Mbps, SCR = 10 Mbps
3. VC3: 데이터 UBR, 최소 보장 없음

링크를 통과하는 셀 흐름을 단순 모델로 표현하면:

```python
# 단순한 ATM VC 구성 예시(실제 구현 X, 개념 전달용)

vcs = [
    {"name": "VC1-Voice", "class": "CBR", "pcr_mbps": 2},
    {"name": "VC2-Video", "class": "rt-VBR", "pcr_mbps": 20, "scr_mbps": 10},
    {"name": "VC3-Data", "class": "UBR", "min_mbps": 0},
]

for vc in vcs:
    print(vc)
```

실제 스위치는 각 VC별 셀 도착 시간을 측정해
- **계약된 패턴보다 너무 자주 오면** CLP=1 또는 drop
- 그렇지 않으면 정상 통과

#### ATM과 SONET의 결합

현실 네트워크에서는 다음과 같은 구조가 흔했습니다.

```text
[고객 라우터] -- [ATM UNI] -- [ATM 스위치] -- [ATM NNI] -- [ATM 스위치] -- ...
                                         |
                                    [SONET/SDH 전송]
```

- ATM 셀은 SONET SPE 안에 매핑되며, STS-n을 통해 장거리 전송
- ATM 스위치는 SONET 링크 상태를 모니터링하며 VP/VC를 다시 라우팅
- DSL 초창기(ATM 기반 DSLAM)에서 ATM은 DSLAM~BRAS 구간에 많이 사용되었고, 그 위에 PPPoA/PPPoE로 IP를 실었습니다.

---

## 전체 요약

1. **SONET**
   - **동기식 광 전송 표준**으로, STS-1(51.84 Mbps) 프레임을 기본으로
     STS-N/OC-N 계층을 형성
   - Section/Line/Path 계층과 풍부한 Overhead를 통해
     **동기, 관리, 보호 절체**를 제공
   - VT(Virtual Tributary)를 통해 T1/E1/DS3 등 레거시 PDH 신호를 효율적으로 수용
   - 링(UPSR/BLSR) 기반 보호 구조로 **50 ms 수준의 복구** 목표 달성
   - 현재는 IP/MPLS·이더넷·OTN 중심의 네트워크로 이동했지만,
     여전히 레거시 전송망·시험문제에서 중요

2. **ATM**
   - B-ISDN을 목표로 설계된 **고정 53바이트 셀 스위칭** 기술
   - 작은 셀(48바이트 payload)·가상회선(VC/VP)·트래픽 계약(CBR/VBR/ABR/UBR)으로
     음성·영상·데이터를 통합하고 QoS를 제공하려 했음
   - 그러나 셀 오버헤드, 복잡한 스택, IP/Ethernet의 급성장으로 인해
     오늘날에는 대부분 **IP/MPLS + Ethernet**으로 대체
   - 그래도 **고정 길이 셀, VPI/VCI, AAL5, 트래픽 policing** 개념은
     네트워크 이론·자격시험·레거시 장비 이해에 필수

이 정리를 기반으로, 나중에 **MPLS·OTN·Carrier Ethernet**을 공부할 때
“왜 SONET/ATM이 이런 설계를 했고, 왜 다음 세대 기술이 그것을 대체했는지”를
비교·비판적으로 볼 수 있을 것입니다.
