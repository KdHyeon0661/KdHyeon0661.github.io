---
layout: post
title: 데이터 통신 - Data Link Layer (1)
date: 2024-07-27 16:20:23 +0900
category: DataCommunication
---
# the Data-Link Layer — Nodes & Links, Services, Link Categories, and Sublayers (LLC/MAC)

## Introduction — 데이터링크 계층이 하는 일

데이터링크 계층(Data-Link, **L2**)은 **이웃 노드 간** 신뢰 가능한 프레이밍과 접근 제어를 제공하여, 상위 계층(네트워크/L3)이 **엔드-투-엔드 라우팅**에 집중하게 만든다. 핵심 기능은 다음과 같다.

- **프레이밍(Framing)**: 비트 스트림을 **프레임**(헤더+페이로드+트레일러) 단위로 경계 지정
- **주소 지정(Addressing)**: 링크-로컬 식별자(예: 48-bit MAC, 그룹/개인 비트, OUI)
- **접근 제어(Medium Access Control)**: 공유 매체에서 충돌 회피/일정 정책(무선: CSMA/CA+EDCA, 유선: 스위칭 기반 풀듀플렉스로 충돌 제거)
- **오류 검출/정정(Error Control)**: **CRC-FCS** 중심의 검출, 무선 등에서 **ARQ/재전송**(Block-ACK)
- **흐름 제어(Flow Control)**: 필요 시 링크 레벨의 정지-대기/슬라이딩 윈도우류(HDLC/LLC Type 2/무선 MAC)
- **품질/우선순위(QoS)**: 802.1p(PCP), 무선 EDCA(Voice/Video 등), TSN(시간 기반 스케줄)
- **보안(Security)**: 유선 MACsec(802.1AE), 무선 WPA3(802.11i 계열)
- **가상화/다중화(Virtualization)**: VLAN(802.1Q), Q-in-Q, 링크 집성(802.1AX LACP)

> 현대 유선 이더넷은 **스위치 기반 풀듀플렉스**가 표준이므로 **CSMA/CD는 사실상 역사적 개념**이다. 반면 **무선**은 여전히 **공유 매체**이므로 MAC 프로토콜(백오프, RTS/CTS, 블록 ACK)이 중요하다.

---

## Nodes and Links — 노드와 링크

### 1) 노드(Node)의 범주

- **호스트**: 엔드 시스템(서버/PC/임베디드/IoT)
- **스위치/브리지**: MAC 학습/포워딩, 브로드캐스트 도메인 관리, VLAN/스패닝트리/EVPN-VXLAN 게이트웨이 등
- **AP(무선 액세스 포인트)**: 무선 MAC(EDCA/OFDM/OFDMA, 블록 ACK) 담당, L2 브리지 역할 연계
- **모뎀/ONU/기타 L2 장치**: PON, DOCSIS 등 매체 종속 링크 제공

### 2) 링크(Link)의 범주

- **유선**: 트위스티드 페어(구리), 광(싱글/멀티모드), 백플레인(데이터센터/통신 장비)
- **무선**: Wi-Fi(802.11 a/b/g/n/ac/ax/be), 전용 마이크로웨이브/밀리미터파 메쉬 등
- **P2P vs 멀티액세스**: P2P(전용선·직결), 멀티액세스(여러 노드가 하나의 매체 공유)

### 3) 링크 성능 구성 요소

총 지연:
\[
D_{\text{total}} = D_{\text{proc}} + D_{\text{queue}} + D_{\text{trans}} + D_{\text{prop}}
\]
- \(D_{\text{trans}} = \frac{L}{R}\) (패킷 직렬화)
- \(D_{\text{prop}} = \frac{d}{s}\) (전파)
- **BDP**(대역폭-지연 곱): \( \text{BDP} = R \cdot \text{RTT} \) — 버퍼/창 크기 산정의 1차 기준

---

## Service — L2가 L3/상위에 제공하는 서비스

### 1) 서비스 유형(고전 분류를 현대 관점으로 정리)

| 서비스 | 연결 | 신뢰성 | 사용 사례 |
|---|---|---|---|
| **Unacknowledged Connectionless** | 없음 | 비보장(오류 시 폐기) | 이더넷 스위치(기본), 대부분의 유선 LAN 전송 |
| **Acknowledged Connectionless** | 없음 | 링크 레벨 ACK/재전송 | 무선 LAN(802.11 블록 ACK), 일부 산업용 링크 |
| **Connection-Oriented (Reliable)** | 설정 | 순서/재전송/흐름제어 | HDLC/LLC Type 2, 전용선/산업 제어 링크 |

> 오늘날 IP 위주의 LAN은 **Unacknowledged Connectionless**가 일반적이다. **무선**은 프레임 손실율/간섭이 커서 **링크 레벨 ARQ**를 흔히 사용한다.

### 2) 서비스 프리미티브(개념적)

- **REQUEST / INDICATION / RESPONSE / CONFIRM**
  예: 상위가 **DATA.request**(송신) → 피어에 **DATA.indication**(수신). 연결형은 **CONNECT.\***, **DISCONNECT.\*** 프리미티브 포함.

### 3) 품질/우선순위 서비스

- **802.1p PCP(3bit, 0–7)**: 우선순위/클래스(Voice/Video/BE/Background 등)
- **무선 EDCA**: Voice/Video/Best-Effort/Background 큐로 분리
- **TSN(802.1Qbv, 802.1AS 등)**: 시간 분할 게이팅·정밀 동기(µs급)로 **결정적 지연** 제공(산업/자동차)

### 4) 보안 서비스

- **MACsec(802.1AE)**: 포트 간 프레임 무결성/암호화
- **802.1X**: 포트 기반 접근 제어(EAP 상호작용)
- **무선 WPA3**: SAE 기반 상호 인증/암호화

---

## Two Categories of Links — 링크의 두 범주

> 관점에 따라 다양한 분류가 있으나, 데이터링크 교과서·표준에서 실무적으로 가장 중요한 두 범주는 **Point-to-Point** 와 **Broadcast/Multiple-Access** 이다.

### 1) Point-to-Point Link (P2P)

- **정의**: 두 노드만 접속. **풀듀플렉스**가 일반(동시 송수신, 충돌 개념 무의미).
- **프레이밍**: 길이 필드/플래그 기반(HDLC/PPP), CRC-FCS.
- **흐름/오류 제어**: 필요 시 단순한 ARQ(Stop-and-Wait/Sliding Window).
- **사용처**: 전용 광링크(백본), 데이터센터 스위치–서버(직결), 통신 장비 간 동기화 라인.

**Stop-and-Wait 효율(예시 수식)**
\[
\text{효율 } \eta = \frac{\frac{L}{R}}{\text{RTT} + \frac{L}{R}}
\]
RTT가 크면 효율 급락 → 고속/장거리 P2P는 **슬라이딩 윈도우** 또는 상위 계층(TCP) 윈도우가 필수.

### 2) Broadcast / Multiple-Access Link

- **정의**: 여러 노드가 하나의 매체를 **공유**. 유선(허브/버스)은 역사적; 현대는 **무선**(WLAN), **세그먼트 내 브로드캐스트 도메인**의 개념이 중요.
- **접근 제어**:
  - 유선(과거)**CSMA/CD** — 충돌 탐지. *현대 스위치+풀듀플렉스에서는 사용되지 않음*.
  - 무선 **CSMA/CA** — 충돌 **회피**(백오프, RTS/CTS, 블록 ACK, OFDMA 조합).
- **브로드캐스트/멀티캐스트 프레임**: ARP, NDP, DHCP 등 제어 트래픽 전파 → 스위치의 **플러딩/학습**과 라우터의 **브로드캐스트 도메인 경계** 관리가 필수.

---

## Two Sublayers — LLC와 MAC

IEEE 802 아키텍처에서 L2는 **LLC(Logical Link Control)** 와 **MAC(Medium Access Control)** **두 서브레이어**로 나뉜다.

### 1) MAC Sublayer

**역할**: 프레임 **주소/접근제어/프레이밍/오류검출**의 핵심. 매체 특성(유선/무선)에 강하게 종속.

- **주소 체계(48-bit MAC, 64-bit EUI)**
  - 최상위 비트: **I/G**(Individual/Group), 그다음: **U/L**(Universally/Locally administered)
  - **OUI(24bit)** + 벤더 확장.
- **프레임 형식(예: Ethernet)**
  ```
  DestMAC(6) | SrcMAC(6) | EtherType/Len(2) | Payload(46–1500*) | FCS(4)
  *점보 프레임(옵션) 최대 ~9KB
  ```
  - **FCS**: CRC-32 (다항식 0x04C11DB7) 기반.
- **매체접근**
  - 유선 스위치: **풀듀플렉스, 충돌 X, 큐/스케줄링 중심**
  - 무선: **백오프(EDCA), RTS/CTS(숨은 노드), 블록-ACK**
- **스위칭/브리징 동작**
  - **MAC 학습**: (입력포트, SrcMAC) → 포워딩 데이터베이스(CAM)
  - **포워딩**: Unicast는 알려진 포트로, **Unknown Unicast/브로드캐스트/멀티캐스트**는 **플러딩**
  - **루프 방지**: **STP/RSTP/MSTP** 계열, L2 루프 억제
  - **VLAN(802.1Q)**: 태그(4B: TPID 0x8100 + PCP/DEI/VID), **Q-in-Q(스택드 VLAN)**
  - **LACP(802.1AX)**: 링크 집성(다중 물리 링크를 논리 1개로)

- **QoS**
  - **802.1p PCP**: 3bit 우선순위(0–7), 스케줄링(SP/WRR/DRR/WFQ)과 결합
  - **DCB**: **PFC(802.1Qbb)**, **ETS(802.1Qaz)** 로 손실 민감 트래픽(RDMA 등) 지원
  - **TSN**: **802.1Qbv**(Time-Aware Shaper), **802.1AS**(동기), **802.1Qbu/802.3br**(프레임 프리엠션)

- **보안**
  - **MACsec(802.1AE)**: L2 프레임 암호화/무결성(장비 간 hop-by-hop 보안)
  - 포트 인증 **802.1X**: 사용자/디바이스 식별 후 VLAN/ACL 부여

### 2) LLC Sublayer

**역할**: 상위(네트워크/L3)에 **일관된 서비스 인터페이스** 제공. 오늘날 IP 위주 환경에서는 상대적으로 얕게 쓰이지만, **SAP(서비스 액세스 포인트), DSAP/SSAP** 개념, **신뢰성/흐름제어 옵션(Type 1/2/3)** 을 정의한다.

- **LLC Type 1**: 비연결·무확인(가벼움; IP에 적합)
- **LLC Type 2**: 연결형·확인(슬라이딩 윈도우/순서 보장)
- **LLC Type 3**: 비연결·확인(드묾)

LLC는 **매체 종속 세부(무선/유선 MAC)를 상위에 추상화**하는 계층으로 이해하면 된다. 실제 구현에서는 **MAC 상단에 얇은 적응층**으로 존재하며, IP는 EtherType 기반으로 바로 매핑되는 경우가 많다.

---

## Framing & Error Control — 프레이밍과 오류 제어(세부)

### 1) 프레이밍 방식

- **길이-필드 기반**: Ethernet
- **플래그/코드 스터핑**: HDLC/PPP
  - **비트 스터핑**: 플래그 패턴(예: 01111110)과 구분하기 위해 1이 5번 연속되면 0 삽입
  - **바이트 스터핑**: 특수 바이트 앞에 이스케이프(0x7D 등) 삽입

### 2) 오류 검출 — CRC의 핵심

- 프레임을 다항식 \(M(x)\)로 보고 **생성다항식 \(G(x)\)** 로 나눈 나머지 \(R(x)\)가 **FCS**
- 수신 측은 \((M(x)\cdot x^r + R(x)) \bmod G(x) = 0\)인지 검사
- 이론적으로 길이 \(r\)의 CRC는 많은 오류 패턴(단일/이중 비트, 홀수 길이 버스트 등)을 높은 확률로 검출

**Ethernet CRC-32**
\[
G(x) = x^{32} + x^{26} + x^{23} + \dots + 1\quad (\text{표준 다항식})
\]

### 3) 링크-레벨 ARQ(무선 등)

- **Stop-and-Wait**: 간단하지만 RTT가 길면 비효율
- **체인 ARQ/블록-ACK(802.11)**: 여러 프레임을 묶어 효율 향상
- 혼잡이 아닌 **무선 오류율**로 인한 손실은 링크-로컬에서 **빠르게 회복**하는 편이 전체 지연/재전송 폭증을 줄인다.

---

## 예제·시나리오(이론 중심, 코드 없이)

### 예제 1 — 데이터센터 서버 ↔ 스위치(유선 P2P)

- **링크**: 100 GbE, 풀듀플렉스, 점보 MTU 9000
- **MAC**: CSMA/CD 없음, **큐/스케줄링 + PFC/ETS(DCB)** 로 손실 민감 트래픽 보장
- **서비스**: Unacknowledged Connectionless(L2), 상위 TCP가 신뢰성 보장
- **효율 포인트**: RTT 수 µs–수십 µs → BDP가 커서 NIC/스위치 버퍼·TCP 윈도 튜닝이 중요

### 예제 2 — 무선 AP ↔ 단말(멀티액세스)

- **링크**: 802.11ax/802.11be, OFDMA + MU-MIMO
- **MAC**: EDCA(Voice/Video 우선), **RTS/CTS**(숨은 노드), **Block-ACK**(프레임 묶음 확인)
- **서비스**: Acknowledged Connectionless(링크 레벨 재전송)
- **주의**: 혼잡+무선 오류 복합 → QoS(EDCA)와 프레임 집계(AMPDU/AMSDU)로 처리량·지연 최적화

### 예제 3 — 공장 TSN 링

- **링크**: 기가비트 이더넷, **802.1AS**(시간 동기), **802.1Qbv**(Time-Aware Shaper)
- **MAC**: 시간 슬롯별 전송 게이트로 **결정적 지연 보장**
- **서비스**: 스케줄 기반의 준-결정적 L2 서비스 → 상위 제어 트래픽의 주기/데드라인 충족

---

## 표/도해 요약

### 표 1 — 링크 범주 핵심 비교

| 항목 | Point-to-Point | Broadcast/Multiple-Access |
|---|---|---|
| 토폴로지 | 1:1 | 1:N(공유) |
| 충돌 | 없음(풀듀플렉스) | 회피/탐지 필요(무선은 회피) |
| 제어 | 간단(프레이밍+CRC) | MAC 정책(백오프/우선순위/RTS-CTS) |
| 예 | 서버-스위치, 장비 간 광 | 무선 LAN, 과거 버스형 LAN |

### 표 2 — LLC vs MAC

| 서브레이어 | 주요 기능 | 현대적 메모 |
|---|---|---|
| **MAC** | 주소, 프레이밍, FCS, 접근제어, 스위칭/브리징, VLAN/QoS/보안 | 실질 L2 핵심. 유선/무선 별도 규격 |
| **LLC** | 추상화 인터페이스, SAP, (옵션) 신뢰/흐름 제어 | IP 주류 환경에서 얇게 사용 |

### ASCII — 스위치의 L2 포워딩 동작(개념)

```
Ingress Port
  ├─ (SrcMAC, InPort) 학습 → CAM 업데이트
  ├─ DestMAC 조회 → {Known Port | Unknown/Broadcast}
  └─ {Unicast 포워드 | VLAN 내 플러딩}
Egress Port
  └─ 큐/스케줄러(PCP/WRR/DRR/WFQ) → 전송
```

---

## 부록 A — 흐름/혼잡·지연 관련 간단 공식

- **직렬화 지연**: \( D_{\text{trans}} = \frac{L}{R} \)
- **전파 지연**: \( D_{\text{prop}} = \frac{d}{s} \)
- **총 지연**: \( D_{\text{total}} = D_{\text{proc}} + D_{\text{queue}} + D_{\text{trans}} + D_{\text{prop}} \)
- **BDP**: \( \text{BDP} = R \cdot \text{RTT} \)
- **Stop-and-Wait 효율**: \( \eta = \frac{L/R}{RTT + L/R} \)

---

## 체크리스트 — 최신 환경에서의 L2 설계/운영

1. **유선**: 스위치+풀듀플렉스(충돌 X), **VLAN/Trunk**, **STP/RSTP**, **LACP**, **DCB(PFC/ETS)**, 필요 시 **MACsec**
2. **무선**: EDCA/RTS-CTS/Block-ACK, 프레임 집계(AMSDU/AMPDU), QoS 맵핑(DSCP↔UP)
3. **QoS/우선순위**: 802.1p PCP, SP+WRR/DRR/WFQ 조합, 지연 민감 트래픽 별도 큐
4. **TSN/산업**: 시간 동기(802.1AS)와 게이팅(802.1Qbv) 계획, 프리엠션 고려(802.1Qbu/802.3br)
5. **보안/접근**: 802.1X+MACsec(유선), WPA3(무선), 동적 VLAN/ACL
6. **브로드캐스트 억제**: L3 경계/ARP 조절, 멀티캐스트 관리(IGMP/MLD 스누핑)
7. **가시성/운영**: 스패닝트리 상태, CAM/ARP 테이블, 큐 지연/드롭, FCS 오류율, 무선 재시도율/링크 품질(-dBm, SNR)

---

## 결론

- 데이터링크 계층은 **이웃 간 신뢰 가능한 프레이밍과 매체 접근 제어**를 제공하여 L3의 엔드-투-엔드 라우팅을 뒷받침한다.
- **Two Categories of Links**: **P2P**는 단순·예측적, **Broadcast/Multiple-Access**는 공유 제어가 핵심(오늘날 무선 중심).
- **Two Sublayers**: **MAC**이 실질 핵심(주소/프레이밍/접근/스위칭/QoS/보안), **LLC**는 상위 인터페이스 및 선택적 신뢰 기능.
- 현대 유선 LAN은 **스위치+풀듀플렉스**로 진화하여 충돌/CSMA/CD는 역사적 개념이 되었고, 무선은 여전히 **MAC 설계**가 성능/지연을 좌우한다. VLAN/QoS/보안/TSN/DCB 등 **확장 기능**은 L2를 단순 전달층을 넘어 **정밀 제어층**으로 만든다.
