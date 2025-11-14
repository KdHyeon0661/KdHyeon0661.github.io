---
layout: post
title: 정보통신기사 - VLAN, STP, RSTP
date: 2025-08-22 17:25:23 +0900
category: 정보통신기사
---
# VLAN / STP / RSTP 개념과 설계 총정리

## 큰 그림: “레이어2의 분리(브로드캐스트 도메인)와 루프 억제”

- **VLAN**: 하나의 스위치/트렁크 위에서 **여러 개의 L2 브로드캐스트 도메인**(논리 스위치) 생성.
- **STP**: L2 루프를 **자동 차단**하고 **루트 브리지**를 중심으로 트리 구성.
- **RSTP**: STP 개선판. **빠른 수렴**(손실/추가 시 수~초 → 수백 ms~1초대), **역할/상태 단순화**.
- **핵심 직감**
  1) VLAN으로 **분리**하고,
  2) (필요하면) **VLAN마다** 또는 **인스턴스별**로 **STP 트리**를 만들며,
  3) **루프는 언젠가 반드시 생긴다**고 가정하고 **보호 기능**(BPDU Guard/Root Guard/Loop Guard/UDLD)을 켠다.

---

## VLAN (IEEE 802.1Q) 기본

### Access/Trunk 포트

- **Access 포트**: **단일 VLAN**의 **언태그드** 프레임만 전송/수신. PC/프린터/서버(싱글 NIC)에 사용.
- **Trunk 포트**: **여러 VLAN**을 **태그**(802.1Q Tag)로 구분하여 운반. 스위치↔스위치, 스위치↔서버(팀/하이퍼바이저) 등.

### 802.1Q 태그와 Native VLAN

- 802.1Q 태그: **TPID 0x8100**, **TCI(PCP 3b / DEI 1b / VID 12b)**.
- **Native VLAN**: 트렁크 상에서 **태그 없이(언태그드)** 전송되는 VLAN.
  - **권장**: Native VLAN을 **미사용(전용/빈 VLAN)** 혹은 **각 단에 동일하게 고정**하고, **허용 VLAN 목록을 엄격히**.
  - **Native VLAN 불일치**는 **프레임 누수/보안 이슈**의 대표 원인.

### VLAN 범위와 네이밍

- **1–4094**(0, 4095 예약).
- **1, 1002–1005(전통)**: 특수 취급하는 장비 존재 → 운영에서는 **사용 지양**.
- **이름 규칙**: `SITE-FLOOR-PURPOSE` 등으로 **정책/문서화一致**.

### VLAN 프레임 길이

- 태그 4바이트 추가 → **1522B**(FCS 포함)까지. 점보프레임 환경이면 **MTU 일치** 필요.

---

## STP(802.1D) 개념

### 왜 필요한가?

- 스위치 링/메시에선 **브로드캐스트 프레임이 순환** → **브로드캐스트 스톰**, **MAC 플래핑**, 네트워크 마비.
- **STP**가 루프를 **논리적으로 차단**하고, 링크 장애 시 **대체 경로**를 **천천히** 올려줌.

### 루트 브리지와 경로 비용

- **루트 브리지**: **가장 낮은 Bridge ID(BID)** 보유 스위치가 선출.
  - $\text{BID} = \text{Priority(4bit)} \parallel \text{Extended System ID(VLAN)} \parallel \text{MAC}$
  - **우선순위** 기본 `32768`(PVST+/RSTP-VLAN별), **작을수록 우선**.
- **경로 비용(Path Cost)**: 링크 속도에 따라 **합산**.
  - **클래식 코스트(예)**: 10M=100, 100M=19, 1G=4, 10G=2
  - **롱 코스트(권장)**: 더 큰 수(예: 10M=2,000,000 … 10G=2,000 … 100G=200 …)로 **고속 링크 구분 용이**

> 설계 Tip: 코어/분배에 **루트 우선순위**를 명시(`spanning-tree vlan X priority 4096/8192`)해 **의도한 루트**를 만든다.

### 포트 역할/상태 (802.1D 전통)

- **역할**: Root Port(각 브리지에서 루트까지 최단), Designated Port(세그먼트 대표), Non-Designated(차단).
- **상태**: Blocking → Listening → Learning → Forwarding (장애/추가 시 **수~수십초** 소요).
- **타이머**: Hello=2s, MaxAge=20s, ForwardDelay=15s (기본값, 도메인 하나로 일관해야 안정).

---

## RSTP(802.1w) — 빠른 수렴

### 차이점 요약

- **상태**: Discarding / Learning / Forwarding (단순화).
- **역할**: Root / **Alternate**(루트로 가는 백업) / **Backup**(세그먼트 내 백업) / Designated.
- **핸드셰이크**: **Proposal/Agreement**(P/A)로 포인트투포인트 링크에서 **즉시 전환** 가능.
- **에지 포트**: **PortFast=Edge**; BPDU 수신 시 **즉시 에지 취소**(Loop 보호).
- **링크 타입**: **Point-to-Point**(풀듀플렉스) vs **Shared**(허브/반이중). P2P일수록 **빠른 합의**.

### 수렴 직감

- 포인트투포인트 + 에지 포트 조합 → **수백 ms~1초 내** 트리 재구성.
- 다만 **멀티액세스/공유** 세그먼트, **BPDU 손실/단방향 링크**에선 보호 기능이 필요.

---

## PVST+/Rapid-PVST vs MSTP(802.1s)

- **PVST+**: **VLAN마다 별도의** (R)STP 인스턴스. **세밀한 제어**(VLAN별 루트/부하분산) 가능, **규모↑ 시 CPU/메모리/TCN↑**.
- **MSTP**: VLAN들을 **몇 개의 MST 인스턴스(MSTI)**로 **묶어** 트리를 줄임.
  - **Region**(Name/Revision/Mapping)이 일치해야 **인스턴스 공유**.
  - 코어 대규모 환경에 권장: **관리 효율↑** + **부하분산**(VLAN→MSTI 매핑).

---

## VLAN 설계 — 분리·할당·트렁크

### 분리 기준

- **보안/역할**(서버/사용자/프린터/IoT), **L3 경계**(서브넷 1개=VLAN 1개 권장), **브로드캐스트 통제**.
- **멀티캐스트/VoIP**: 음성 VLAN, 멀티캐스트 전송 영역 고려(IGMP Snooping/Querier).

### 트렁크 설계

- **허용 VLAN 목록**을 **명시**(Pruning)하고, **Native VLAN 고정**(가능하면 전용 빈 VLAN).
- **DTP(자동안됨)**: `switchport nonegotiate`로 **수동 트렁크** 권장.
- **포트채널(EtherChannel)** 묶음: STP는 **포트채널을 하나**로 인식 → 루프 위험↓/대역폭↑.

### L3 게이트웨이

- **SVI**(Switch Virtual Interface) 또는 **라우터 서브인터페이스**(Router-on-a-Stick).
- **인터-VLAN 라우팅** 정책/ACL/방화벽 경계 설계.

---

## STP/RSTP 동작을 수식으로 요약

- **BID 비교**:
  $$
  \text{BID} = \text{Priority} \cdot 2^{52} + \text{VLAN(Ext Sys ID)} \cdot 2^{48} + \text{MAC}
  $$
  → **가장 작은 BID**가 루트. (개념: 우선순위가 **압도적 비중**)

- **Path Cost 합**:
  $$
  \text{Root Path Cost} = \sum \text{(링크 코스트)}
  $$
  → 각 브리지는 **Root Path Cost가 최소**인 이웃을 **Root Port**로 선택.

- **포트 역할 선택 규칙(요지)**
  1) **Root Bridge**의 모든 포트는 **Designated**
  2) 비-루트는 **Root Path Cost** 최소인 포트를 **Root Port**
  3) 각 세그먼트당 **가장 우월한 광고(BPDU)** 송신 포트가 **Designated** (나머지 **Alternate/Blocking**)

---

## 보호 기능(운영 필수)

| 기능 | 목적 | 동작 |
|---|---|---|
| **PortFast / Edge** | 단말 포트 **즉시 포워딩** | BPDU 수신 시 **에지 해제** |
| **BPDU Guard** | 단말 포트에 **BPDU 들어오면 다운** | **루프/로구 루트** 차단 |
| **Root Guard** | 엣지/하향 포트에서 **루트 선출 방지** | 더 좋은 BPDU 수신 시 **일시 블록(Root-Inconsistent)** |
| **Loop Guard** | **BPDU 소실**로 인한 일시적 루프 방지 | 예상 BPDU 미수신 시 **Loop-Inconsistent** |
| **UDLD** | **단방향 링크** 탐지(특히 광) | Unidirectional 감지 시 **에러디세이블** |
| **Storm Control** | 브로드/멀티/유니캐스트 **바람** 제한 | 임계 초과 시 차단/로그 |

> 현장 규칙: **엣지 포트 = PortFast + BPDU Guard**, **코어 하향 = Root Guard**, **업링크 = Loop Guard/UDLD**.

---

## 부하 분산(Per-VLAN Root 최적화)

- **VLAN10 루트 = SW-A**, **VLAN20 루트 = SW-B**처럼 **루트 스위치 분담**으로 **양 방향 트래픽 분산**.
- 포트채널을 사용하면 STP는 **한 링크**로 보지만, **LACP 해시**로 **유량 분산**.

**예) Cisco**
```bash
spanning-tree mode rapid-pvst
spanning-tree vlan 10 priority 4096
spanning-tree vlan 20 priority 4096
! A/B에서 VLAN별 priority 반대로 설정하여 루트 분산
```

---

## 타이머·코스트 표(참고)

### STP 기본 타이머(도메인 공통)

- **Hello**: 2 s
- **MaxAge**: 20 s
- **Forward Delay**: 15 s
- 전통 STP 수렴: **최대 ~50 s**(차단→포워드), RSTP: **<<**.

### 링크 속도별 코스트(예시)

- **클래식**: 10M=100, 100M=19, 1G=4, 10G=2
- **롱 코스트(권장)**: 10M=2,000,000 / 100M=200,000 / 1G=20,000 / 10G=2,000 / 100G=200 …
  (벤더/OS에서 **자동 전환 옵션** 존재: `spanning-tree pathcost method long`)

---

## 예시 토폴로지 & 설계

### 3-스위치 코어/액세스

```
        [SW-A]====(Po1)====[SW-B]
           ||                 ||
          (Po2)             (Po3)
           ||                 ||
         [SW-Edge1]       [SW-Edge2]
```

- A–B: 포트채널 Po1(트렁크), A–Edge1: Po2, B–Edge2: Po3
- VLAN10(사용자), VLAN20(서버), VLAN30(VoIP)
- **루트 분산**: VLAN10 루트=A, VLAN20/30 루트=B
- **Edge 포트**(사용자/단말): **PortFast + BPDU Guard**
- **업링크**(코어↔액세스): **Loop Guard + UDLD**
- 트렁크: **Native VLAN 999(빈)**, **허용 VLAN = 10,20,30만**

---

## 설정 예시(Cisco IOS 스타일)

### VLAN/트렁크

```bash
vlan 10
 name USER
vlan 20
 name SERVER
vlan 30
 name VOICE
vlan 999
 name NATIVE-EMPTY

interface range gi1/0/1 - 2
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,999
 switchport nonegotiate
 spanning-tree guard loop
 udld port aggressive

interface range gi1/0/10 - 48
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 storm-control broadcast level 1.00 0.50
```

### STP 모드/루트 분산

```bash
spanning-tree mode rapid-pvst
spanning-tree extend system-id
spanning-tree vlan 10 priority 4096
spanning-tree vlan 20 priority 8192
spanning-tree vlan 30 priority 8192
spanning-tree portfast default
spanning-tree pathcost method long
```

### MSTP(대규모 코어)

```bash
spanning-tree mode mst
spanning-tree mst configuration
 name DC-REGION
 revision 10
 instance 1 vlan 10,30
 instance 2 vlan 20
exit
! 루트 분산
spanning-tree mst 1 priority 4096
spanning-tree mst 2 priority 4096
```

---

## 검증/운영 명령

```bash
show spanning-tree summary
show spanning-tree vlan 10
show spanning-tree mst 1
show spanning-tree interface gi1/0/1 detail
show spanning-tree inconsistentports

show interfaces trunk
show vlan brief

show udld interface
show errdisable recovery
```

**해석 포인트**
- `Root ID`가 **의도한 스위치**인지.
- 포트 역할이 **Root/Designated/Alternate**로 기대대로인지.
- `PortFast` 적용 포트에 **BPDU 수신**이 기록되면 **배선 오류** 의심.
- `inconsistent` 포트가 있으면 **Loop/Root Guard 동작** 확인.

---

## 자주 발생하는 문제 & 대처

| 증상 | 원인 | 해결 |
|---|---|---|
| **Native VLAN 불일치** | 양단 Native 다름 | **같게 통일**, Native는 **빈 VLAN** |
| **브로드캐스트 스톰** | 루프, 멀티플 패스 | **STP 활성**, 포트채널로 **의도된 멀티링크**, Storm Control |
| **루트가 엣지로 변경** | 외부 스위치 삽입 | **Root Guard** 배포, 우선순위 명시 |
| **MAC 플래핑** | 2곳에서 같은 MAC 관측 | 루프 탐지, **BPDU Guard**/Loop Guard/UDLD 확인 |
| **수렴 느림** | STP 전통 모드, 타이머 | **RSTP** 전환, P2P 링크, Edge 적절설정 |
| **VLAN 누수** | Trunk 허용 과다/Native | **Allowed VLAN 제한**, Native 빈 VLAN |
| **Errdisable** | BPDU Guard/UDLD 트립 | 원인 해결 후 `errdisable recovery` 정책/로그 점검 |

---

## 계산/미니 도구(개념)

### 경로 코스트 합 비교(예)

- 링크: 액세스(1G)→분배(10G)→코어(10G)
- **롱 코스트**: 1G=20,000, 10G=2,000
- 경로1(1G+10G)=22,000, 경로2(1G+10G)=22,000 → **동일 비용** → **RSTP Alternate** 결정은 BPDU 우월성(MAC/Port ID) 비교.

### 포트 우월성 타이브레이커(요지)

1) Root Path Cost ↓
2) Sender Bridge ID ↓
3) Sender Port ID ↓

```python
# 교육용: RSTP 우월성 비교

from dataclasses import dataclass
@dataclass(order=True)
class BPDU:
    root_cost:int; sender_bridge:str; sender_port:int
# 낮을수록 우월

bpdu_a = BPDU(22000, "32768.0011.2233.4455", 1)
bpdu_b = BPDU(22000, "32768.0011.2233.4466", 1)
print("Winner:", "A" if bpdu_a<bpdu_b else "B")
```

---

## 디자인 패턴

### 소규모 캠퍼스(2계층)

- **Rapid-PVST** + **VLAN별 루트 분산**.
- 모든 액세스 포트 **Edge+BPDU Guard**, 업링크 **Loop Guard/UDLD**.

### 대규모 캠퍼스(3계층/MSTP)

- **MSTP**로 VLAN→MSTI 매핑(예: MST1=사용자/IoT, MST2=서버/VoIP).
- **분배층**을 루트, **코어**는 **라우팅(ECMP)** 우선으로 L3 업링크(가능하면 **L3 to Access**로 STP 도메인 축소).

### 데이터센터(권장: L3/패브릭)

- 가능하면 **L2 도메인 축소**, **VxLAN/EVPN** 등 L3 패브릭.
- L2가 필요하면 **포트채널 + RSTP Edge/Backbone 보호** **+ MLAG/스택**.

---

## 보안/컴플라이언스 체크

- **미사용 포트**: `shutdown`, Access+VLAN Blackhole(999), BPDU Guard on.
- **VTP**: 대형 사고 방지 위해 **Transparent** 권장(수동 관리).
- **DHCP Snooping / Dynamic ARP Inspection / IP Source Guard**: 사용자 VLAN 위조 방지.
- **Mgmt VLAN 분리**: out-of-band 또는 전용 관리 VLAN + ACL.

---

## 로그 사례(의미 파악)

```
%SPANTREE-2-ROOTGUARD_BLOCK: Root guard blocking port Gi1/0/24 on VLAN0010.
# 하향 포트에서 더 우월한 BPDU 수신 → 루트 가드가 루트 승격 시도 차단

%UDLD-4-UDLD_PORT_DISABLED: UDLD disabled interface Gi1/0/1 on Tx
# 단방향 링크 감지 → 루프 위험 차단

```

대응: 상위 장비 루트/타이머 확인, 광패치 재연결, 포트 재가동 전 원인 제거.

---

## 자주 틀리는 포인트(정오표)

1) **RSTP**는 STP의 “빠른 수렴”판이지 **프로토콜 목적**(루프 방지)이 바뀐 게 아니다.
2) **PortFast(Edge)**는 **단말 포트**에만. **트렁크나 서버-브리징**엔 신중(`portfast trunk`는 루프 가능성 이해 후).
3) **PVST+**는 **VLAN마다 인스턴스** → VLAN 수↑ 시 **CPU/메모리** 고려. 대규모는 **MSTP**.
4) **Native VLAN 불일치**는 보안/운용 사고 다발. “빈 VLAN” 사용 습관화.
5) **Path Cost** 표 혼동 주의. 고속 링크 많으면 **롱 코스트** 적용.
6) **BPDU Filter**는 오남용 주의(실수로 BPDU 무시→루프). 기본은 **BPDU Guard**.

---

## 연습문제(풀이 포함)

**Q1.** PVST+에서 VLAN 10의 루트를 SW-A로 강제하려면?
**A.** `spanning-tree vlan 10 priority 4096`(SW-A), 타 스위치는 기본(32768) 유지.

**Q2.** RSTP에서 에지 포트가 BPDU를 받으면?
**A.** 에지 속성이 **즉시 해제**되고 **일반 포트**로 전환(루프 보호).

**Q3.** 트렁크 양단 Native VLAN이 1/999로 다르다. 현상과 해결은?
**A.** **프레임 누수/이상 브로드캐스트**, STP 문제. **동일한 Native VLAN**으로 통일(또는 빈 VLAN).

**Q4.** 링크: 1G–10G–10G(롱 코스트 20000+2000+2000=24000)와 1G–1G–10G(20000+20000+2000=42000). 어느 경로가 Root Port?
**A.** **24000** 비용 경로.

**Q5.** 루트가 엣지 스위치로 바뀐다. 예방책 2가지?
**A.** **Root Guard**(하향), **루트 스위치 priority 낮추기**(4096 등).

**Q6.** 액세스 포트에서 브로드캐스트 스톰 발생을 제어하려면?
**A.** **Storm Control**(브로드/멀티/유니캐스트), 필요 시 **Port Security**.

**Q7.** MSTP에서 VLAN 10/30은 MSTI1, VLAN 20은 MSTI2로 묶었다. 루트 분산을 하려면?
**A.** **MSTI1 루트=SW-A**, **MSTI2 루트=SW-B**로 **instance priority**를 다르게 설정.

---

## 구축/점검 체크리스트

- [ ] **VLAN 설계**: 서브넷=VLAN 1:1, 네이밍/문서화, 멀티캐스트/VoIP 정책
- [ ] **트렁크**: Allowed VLAN 제한, **Native=빈 VLAN**, `nonegotiate`
- [ ] **STP 모드**: Rapid-PVST(소규모)/MSTP(대규모), **Pathcost Long**
- [ ] **루트 분산**: VLAN/MSTI별 priority, 포트채널 경로
- [ ] **보호 기능**: PortFast+BPDU Guard(엣지), Root Guard(하향), Loop Guard/UDLD(업링크)
- [ ] **VTP Transparent**, 미사용 포트 `shutdown`
- [ ] **검증**: `show spanning-tree`, 트렁크/errdisable/udld/로그
- [ ] **운영**: 변경 전/후 **루트/역할 캡처**, 문서화, 롤백 계획

---

## 마무리

- **VLAN**으로 **논리 격리**를 하고, **(R)STP**로 **루프 없는 트리**를 만든 뒤,
- **루트/코스트/보호 기능**을 **의도적으로** 설계하면 **안정**과 **빠른 복구**를 동시에 얻습니다.
- 소규모는 **Rapid-PVST + VLAN별 루트 분산**, 대규모는 **MSTP**로 인스턴스를 **집약**하세요.
- 마지막으로, **Native VLAN**과 **보호 기능** 설정은 “기본 위생”입니다. 이 두 가지만 잘해도 사고의 80%는 막습니다.
