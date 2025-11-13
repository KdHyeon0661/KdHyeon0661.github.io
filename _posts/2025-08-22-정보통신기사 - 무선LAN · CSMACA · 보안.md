---
layout: post
title: 정보통신기사 - 무선LAN · CSMA/CA · 보안
date: 2025-08-22 14:25:23 +0900
category: 정보통신기사
---
# 무선LAN · CSMA/CA · 보안 개념 총정리

## 0. 큰 그림: 802.11 네트워크 한 장

- **역할**: AP(Access Point) ↔ STA(클라이언트). BSS(셀)들의 집합이 ESS.
- **프레임 3대장**: Management(Beacon, Probe, Auth, Assoc), Control(RTS/CTS/ACK/BA), Data(QoS Data).
- **매체 접근**: 유선의 CSMA/CD 대신 **무선은 CSMA/CA**(충돌 회피) + **백오프**.
- **성능 핵심**: PHY Rate(=MCS) **아닌** **에어타임**이 병목. **오버헤드(프리앰블/IFS/ACK)**와 **재시도**가 실제 처리량을 좌우.
- **보안 핵심**: **WPA2-PSK(CCMP)** 이상, 최신은 **WPA3-SAE/Enterprise**, **PMF(802.11w)** 필수.

---

## 1. 802.11 PHY·대역·채널 빠른 맵

| 세대 | 대역폭 | 변조/특징 | 최대 링크 속도(개념) |
|---|---|---|---|
| **802.11a/g** | 20 MHz | OFDM, 최대 54 Mb/s | 단일 스트림 54 |
| **802.11n (HT)** | 20/40 | OFDM, **MIMO**, **A-MPDU/A-MSDU** | 150 Mb/s/SS @20, 300 @40 |
| **802.11ac (VHT)** | 20/40/80/160 | 256-QAM, **MU-MIMO DL** | 433 Mb/s/SS @80 |
| **802.11ax (HE, Wi-Fi 6/6E)** | 20~160 | **OFDMA**, 1024-QAM, **UL/DL MU-MIMO**, **BSS Coloring**, **TWT** | 600+ Mb/s/SS @80 |
| **6 GHz (Wi-Fi 6E)** | 20~160 | 11ax on 6 GHz, PSC 채널 | 더 넓은 연속 채널 |

- **2.4 GHz**: 비면허, 채널 **1/6/11**(20 MHz 비중첩). 혼잡·간섭 多.
- **5 GHz**: 채널 多, **DFS**(레이다 감지) 유의.
- **6 GHz**: 상호간섭 적음, **WPA3/PMF 필수**, PSC(Preferred Scanning Channels) 존재.

---

## 2. 802.11 프레임 구조 개요

```
[ MAC Header (24/30/34 B) | QoS Control? | HT/VHT/HE Control? | Frame Body | FCS(4) ]
Preamble/PLCP(물리계층 헤더)는 PHY가 붙임(속도·길이 등 포함)
```

- **Management**: Beacon(SSID, TIM/DTIM, 채널, 지원속도), Probe Req/Resp, (Open/Sae) Auth, (Re)Assoc.
- **Control**: **RTS/CTS**(NAV 설정), **ACK**(단일), **Block ACK(BA/BAreq)**.
- **Data**: QoS Data(AC별), Null(PS/Keepalive).
- **Duration/NAV** 필드: 타 단말의 **가상 캐리어 감지**(충돌 회피 핵심).

---

## 3. CSMA/CA(충돌 회피)와 EDCA(802.11e QoS)

### 3.1 DCF 기본(Distributed Coordination Function)
1) **물리 감지**(CCA): 채널이 Busy면 대기.
2) **DIFS** 경과 후 **백오프 카운트다운** 시작.
3) **무작위 백오프**: `backoff = rand[0..CW]` 슬롯, Idle 슬롯마다 1씩 감소.
4) 0이 되면 **프레임 전송**. 성공 시 상대가 **SIFS 후 ACK**. 실패(ACK 미수신) → **CW↑(BEB)** 후 재시도.
- **타이밍**: `SIFS < PIFS < DIFS` (수치 예: 2.4G OFDM 계열 SIFS=10 µs, DIFS=28 µs, Slot=9 µs 등; 표준/대역에 따라 다름)

### 3.2 EDCA (Enhanced DCF, 802.11e/WMM)
- 4개 **AC**(BK, BE, VI, VO)별 **AIFS**, **CWmin/max**, **TXOP** 차등.
  - **VO**: AIFS↓, CWmin↓ → 먼저 잡음. **TXOP**로 **연속 프레임** 전송(ACK 포함) 허용.
- **목적**: 음성/영상의 **지연·지터** 보장.

### 3.3 RTS/CTS & CTS-to-Self
- **RTS/CTS**: RTS→CTS 교환으로 **NAV** 셋업, **Hidden Node** 충돌 예방. 작은 프레임엔 오버헤드.
- **CTS-to-Self**: 송신자가 자기에게 CTS 발행, **단일 프레임 보호**. 11n 보호/혼재 환경에서 사용.

### 3.4 BEB (Binary Exponential Backoff)
- 재시도 i번째: `CW = min(2^i*(CWmin+1)-1, CWmax)`
- **기대 백오프 슬롯** $\mathbb{E}[k] \approx \frac{CW}{2}$.
- 재시도 한계/드롭은 드라이버/칩셋 정책.

### 3.5 NAV/가상 캐리어 감지
- 프레임의 **Duration**을 이용해 타 STA들이 **대기**. 물리 감지와 **둘 다 만족**해야 송신.

---

## 4. 802.11n/ac/ax MAC 효율 기능

- **프레임 집계**
  - **A-MSDU**: 상위 MSDU 여러 개를 **하나의 MPDU Payload**에 결합(헤더 오버헤드↓, 에러 시 전체 재전송).
  - **A-MPDU**: 여러 **MPDU**를 하나의 **PPDU**(PHY 프레임)로 묶음(개별 CRC, **Block ACK**로 선택 재전송).
- **Block ACK (BA)**: BAreq/BA로 다중 MPDU 승인을 **한 번에**.
- **RIFS/SIFS**: 11n은 **RIFS**(짧은 간격) 있었으나 보통 SIFS 사용.
- **Rate Adaptation**: Minstrel류 알고리즘으로 **MCS** 동적 선택(에러율/에어타임 기반).
- **채널 본딩**: 20→40(HT), 80/160(VHT/HE). 인접 BSS 간 **공존·CCA** 한계 주의.

---

## 5. 802.11ax(HE, Wi-Fi 6/6E) 핵심

- **OFDMA**: RU(Resource Unit) 단위로 **다수 STA 동시** UL/DL 전송 → **에어타임 효율↑**(작은 패킷/IoT 유리).
- **MU-MIMO UL/DL**: 공간 다중화 극대화(채널 상태 정보 필요).
- **BSS Coloring**: 이웃 BSS의 프레임에 **컬러** 태깅 → 약한 OBSS 신호는 **무시 임계(OBSS-PD)** 높여 **Spatial Reuse**.
- **TWT(Target Wake Time)**: 저전력 기기의 **예약 기반** 깨어남(배터리↑).
- **HE-SU/HE-MU** PPDU 포맷, **HE Control** 필드, **1024-QAM** 등.

---

## 6. 처리량(Throughput)과 에어타임(Airtime) 계산

### 6.1 기본 아이디어
- **유효 처리량** $\approx \dfrac{\text{유저 페이로드}}{\text{에어타임 총합}}$
- 총합 = **프리앰블/PLCP + (DIFS/AIFS + 백오프) + PPDU + SIFS + ACK** (+ RTS/CTS/BA 등).
- **Aggregation/BA**를 쓰면 페이로드↑, ACK 수↓ → 효율↑.

### 6.2 근사 수식(단일 프레임)
$$
T_\text{air} \approx T_\text{preamble} + T_\text{AIFS} + T_\text{bo} + T_\text{data} + T_\text{SIFS} + T_\text{ACK}
$$
- $T_\text{data}=\dfrac{L_\text{MAC+LLC+IP+TCP}+ \text{MAC 헤더}+FCS}{R_\text{PHY, data}}$ (물리 부호화 반올림 존재)
- **블록ACK + A-MPDU**라면 `ACK` 1회로 다수 MPDU 승인.

### 6.3 예제(개념 수치)
- **가정**: 11n, 20 MHz, MCS 7(65 Mb/s, GI=800ns), A-MPDU로 20개 MPDU(각 1500B 유저 페이로드), BA 사용.
- 대략:
  - **Data 에어타임** ≈ (20 × (1500+MAC/FCS 등) / 65 Mb/s)
  - **오버헤드**: 프리앰블(PLCP~), AIFS+백오프(평균 $CW/2$ 슬롯×슬롯타임), SIFS, BA.
- **교훈**: A-MPDU 미사용/RTS/CTS 상시 → 처리량 큰 폭 감소. 실무는 **AMPDU+BA** 활성, **RTS 임계** 튜닝.

> 엄밀 계산은 PHY 표 기반 심볼/서브캐리어/코딩율 반올림을 적용해야 합니다. 여기선 조망만 제공합니다.

---

## 7. 로밍과 관리: 802.11k/v/r

- **11k**: Neighbor Report로 **인근 AP 목록** 제공 → 탐색 부담↓.
- **11v**: BSS Transition(스티어링), 타임 동기 등 관리 프레임.
- **11r**: **Fast BSS Transition(FT)**, PMK-R0/R1 키 계층으로 **핸드오버 지연↓**(VoWiFi에 중요).
- 실무 팁: **k/v/r 지원 단말**인지 확인, AP 벤더별 **Sticky Client** 완화 기능(최저 RSSI/Minimum Rate) 병행.

---

## 8. 전력절감: Power Save·U-APSD·TWT

- **Legacy PS-Poll**: TIM/DTIM에 따라 버퍼된 프레임 수신.
- **U-APSD(WMM-PS)**: 트리거 기반 절전(VoIP에 유리).
- **TWT(11ax)**: AP-STA 간 **깨우기 스케줄** 합의 → IoT/배터리 기기 생명↑.

---

## 9. 무선 보안: 세대와 개념

### 9.1 WEP(역사·비권고)
- **RC4 + 24-bit IV**, 키 재사용/IV 고갈 문제 → **취약**. 사용 금지.

### 9.2 WPA/WPA2
- **WPA-TKIP**: WEP 보완(임시), 현재 비권고.
- **WPA2-PSK(CCMP-AES)**: 가정/소규모 표준. **4-Way Handshake**로 PTK 생성, **GTK**(그룹 키) 별도.
- **WPA2-Enterprise(802.1X/EAP + RADIUS)**: 기업용. 사용자/디바이스별 인증(**EAP-TLS/PEAP** 등), **동적 키**.

### 9.3 WPA3
- **WPA3-SAE (Personal)**: 오프라인 추측 공격 방지, **비밀번호 기반 상호 인증**(Dragonfly).
- **WPA3-Enterprise**: 192-bit Suite 옵션, **EAP-TLS 권장**.
- **PMF(802.11w)**: **관리프레임 보호**(Deauth/Disassoc 스푸핑 방지). WPA3에선 **필수**.

### 9.4 키 계층·핸드셰이크 요약
- **PMK**: PSK 또는 802.1X(EAP)로 생성.
- **4-Way**: $\text{PTK} = \mathrm{PRF}(\text{PMK}, \text{ANonce}, \text{SNonce}, \text{AA}, \text{SA})$
- **GTK**: 브로드/멀티캐스트용 **그룹키**(주기적 재키).
- **회선 보안**: **CCMP(AES-CTR+CBC-MAC)**, 최신은 **GCMP**(성능↑).

### 9.5 개방형/캡티브 포털 주의
- **Open SSID + Captive**는 **평문**. 민감 데이터는 **상위 계층 TLS**에 의존. 가능하면 **OWE(Enhanced Open)** 고려.

---

## 10. 전파 이슈와 설계

### 10.1 채널 계획
- **2.4 GHz**: **1/6/11** 고정, AP Tx 파워 **균형**(셀 오버랩 15~20%).
- **5 GHz**: 40/80 MHz는 **채널 재사용·DFS** 고려. 혼잡 환경이면 20/40로 **셀 밀도↑**.
- **6 GHz**: PSC 우선, WPA3 필수, 외부 간섭 적음.

### 10.2 최소 데이터레이트/저속 비활성
- **1/2/5.5/11 Mb/s**(2.4G) 같은 저속을 **비활성** → **에어타임 낭비 감소**, 로밍 개선.

### 10.3 밴드 스티어링·Airtime Fairness
- 단말을 5/6 GHz로 유도(밴드 스티어링).
- **Airtime Fairness**: 느린 단말이 **셀 전체**를 잡아먹지 않게 **에어타임 기준** 스케줄.

### 10.4 Hidden/Exposed Node
- **Hidden**: 서로 안 들리는 STA끼리 충돌 → **RTS/CTS**, **CCA 임계 조정**, **셀 크기 재설계**.
- **Exposed**: 불필요 회피로 동시성↓ → BSS Coloring/OBSS-PD가 완화.

---

## 11. 트러블슈팅·카운터 해석

| 증상/지표 | 의심 | 처방 |
|---|---|---|
| **Retry/Fail↑, ACK Timeout** | 간섭/약한 RSSI/Hidden | 채널 변경, RTS 임계↓, 전력/안테나 조정 |
| **Channel Utilization↑** | 과밀 셀/멀티캐스트 폭주 | AP 증설(셀 분할), 멀티캐스트→유니캐스트, DTIM 튜닝 |
| **Low PHY rate** | SNR↓/저속 강제 | 최소 데이터율 상향, 밴드 스티어링 |
| **Sticky Client** | 로밍 미흡 | 11k/v/r, Min RSSI, Client Match |
| **Deauth 폭탄** | PMF 미사용 | **PMF 켜기**, WIPS/WIDS, 관리평문 차단 |
| **Captive 연결은 되는데 안전?** | 평문 | TLS 앱 사용, OWE 도입 |

**Wireshark 필터**
```
wlan.fc.type_subtype == 0x08   # Beacon
wlan.fc.type_subtype == 0x0b   # RTS
wlan.fc.type_subtype == 0x1d   # Block Ack
wlan.ta == aa:bb:cc:dd:ee:ff || wlan.ra == ...
wlan.fc.retry == 1
```

---

## 12. 계산 예제 (빠른 감 잡기)

### 12.1 평균 백오프 시간(EDCA-BE 예)
- **2.4G OFDM 가정**: Slot=9 µs, `CWmin=15` → 평균 `CW/2=7.5 슬롯` ≈ **67.5 µs**
- `CWmax=1023`까지 증가 가능(혼잡시).

### 12.2 A-MPDU의 효과(개념 수치)
- 단일 MPDU: 매 전송마다 **SIFS+ACK** 필요.
- A-MPDU 32개 묶음: **프리앰블+백오프+SIFS+BA** 한 번 → 헤더/IFS 비율↓ → 처리량 ↑.
- 실무 체감: **2~3배**↑도 흔함(환경/에러율 의존).

### 12.3 Airtime 비중
$$
\text{Airtime} = \frac{\text{Bytes} \times 8}{\text{PHY Rate}} + \sum \text{오버헤드(µs)}
$$
- PHY Rate↑라도 **오버헤드(µs)**는 동일/가깝다 → **짧은 프레임**일수록 오버헤드 지배.

---

## 13. 설정 체크리스트(현장용)

- [ ] **보안**: WPA2-AES 이상, **WPA3-SAE/Enterprise + PMF** 권장(6 GHz 필수)
- [ ] **SSID 개수 최소화**(브로드캐스트 오버헤드↓)
- [ ] **채널 고정/출력 밸런스**, 2.4=1/6/11, 5/6=현장 스캔 기반
- [ ] **최저 데이터율 상향**, Airtime Fairness, 밴드 스티어링
- [ ] **AMPDU/BA On**, RTS 임계 적정(혼재/히든 시)
- [ ] **11k/v/r** 활성(가능 시), 로밍 정책/Min RSSI
- [ ] 멀티캐스트/브로드캐스트 관리(DTIM, Mcast-to-Ucast)
- [ ] **WIPS/WIDS** 정책 및 **관리프레임 보호(PMF)**
- [ ] AP 전원/백홀 안정(PoE Budget/케이블 품질)

---

## 14. 자주 틀리는 포인트(정오표)

1) **CSMA/CD**는 **유선**. 무선은 **CSMA/CA**(+NAV/백오프/ACK).
2) PHY 속도=사용자 속도 **아님**. **에어타임/오버헤드**가 좌우.
3) **RTS/CTS**는 만능이 아니다(오버헤드↑). **히든 노드**일 때만.
4) **WEP/TKIP** 금지. **WPA2-CCMP/ WPA3-SAE** 사용.
5) **PMF** 미사용 시 Deauth 스푸핑 취약.
6) **SSID 많을수록** 비콘 오버헤드 ↑.
7) **2.4 GHz 채널 가변폭/중첩**은 재앙. **1/6/11 고정**.
8) **저속 허용**은 셀 전체 속도를 잡아먹는다 → **Disable**.

---

## 15. 연습문제(풀이 포함)

**Q1.** CSMA/CA에서 전송 직전 대기하는 간격은?
**A.** **AIFS(또는 DIFS)** 후 **무작위 백오프**. 성공 시 수신측은 **SIFS 후 ACK**.

**Q2.** 평균 백오프 슬롯 수는? (CW=31)
**A.** $\mathbb{E}[k]=\frac{CW}{2}=15.5$ 슬롯.

**Q3.** 히든 노드 완화 2가지는?
**A.** **RTS/CTS(또는 CTS-to-Self)**, **셀 설계/출력 조정**(CCA 임계, AP 배치).

**Q4.** WPA3-SAE의 장점 한 줄?
**A.** **암호 추측의 오프라인 공격에 강함**, 상호 인증, PMF 필수.

**Q5.** 2.4 GHz에서 비중첩 채널은?
**A.** **1/6/11**.

**Q6.** A-MPDU의 장점은?
**A.** **ACK/IFS 오버헤드 감소**와 **부분 재전송(Block ACK)**로 **효율↑**.

**Q7.** PMK/PTK/GTK의 관계를 요약하라.
**A.** PMK(마스터)로 **4-Way**에서 **PTK(유니캐스트 키)** 파생, **GTK(그룹키)**는 브로드/멀티캐스트용.

---

## 16. 미니 계산기 (개념용 Python)

```python
# 단순 Airtime 근사 계산기
def airtime_us(payload_bytes, mac_header=34, llc_snap=8, fcs=4,
               phy_rate_mbps=65.0, preamble_us=36, aifs_us=28, sifs_us=10, ack_us=30,
               cw_slots=7.5, slot_us=9.0):
    data_bits = (payload_bytes + mac_header + llc_snap + fcs) * 8
    t_data = (data_bits / (phy_rate_mbps * 1e6)) * 1e6
    t_bo = cw_slots * slot_us
    return preamble_us + aifs_us + t_bo + t_data + sifs_us + ack_us

for p in [256, 1500]:
    print(p, "B airtime(us) ~", round(airtime_us(p),1))
```

- 위 값은 **거친 근사**입니다(AMPDU/BA/OFDMA/HE-preamble 미반영). 추세 파악용으로 활용하세요.

---

## 17. 압축 요약(시험 직전 12줄)

1) **CSMA/CA**: AIFS/DIFS → **Backoff** → Tx → **SIFS+ACK**, 실패 시 **BEB**.
2) **EDCA**: AC별 AIFS/CW/TXOP 차등(VO가 우선).
3) **RTS/CTS**: NAV로 보호(히든 대응), 소프레임엔 역효과.
4) **A-MPDU + Block ACK**: 효율 필수템.
5) **11ax**: **OFDMA/MU-MIMO**, **BSS Coloring/OBSS-PD**, **TWT**.
6) 처리량은 **에어타임 분모**가 좌우(프리앰블/IFS/ACK/재시도).
7) 로밍: **11k/v/r**로 탐색·핸드오버 최적화.
8) 보안: **WPA2-AES** 기본, **WPA3-SAE/Enterprise + PMF** 권장.
9) 2.4 GHz는 **1/6/11**, 5/6 GHz는 DFS/PSC 고려.
10) **저속 비활성 + Airtime Fairness**로 느린 단말의 독점 방지.
11) **SSID 최소화**, 비콘/브로드캐스트 오버헤드 관리.
12) **WIPS/WIDS**와 로그로 Deauth/간섭 조기 탐지.

---

## 18. 마무리

무선 LAN의 성패는 **PHY 속도**가 아니라 **에어타임**을 누구에게 어떻게 배분하느냐에 달려 있습니다.
**CSMA/CA의 대기·충돌·재시도**를 줄이고, **집계/BA/EDCA/11ax 기능**을 제대로 쓰면 같은 하드웨어로도 **두 배 이상의 체감 차이**가 납니다.
보안은 **WPA3 + PMF**를 기본으로, 기업 환경은 **802.1X/EAP-TLS**로 **ID·키 관리**를 체계화하세요.
