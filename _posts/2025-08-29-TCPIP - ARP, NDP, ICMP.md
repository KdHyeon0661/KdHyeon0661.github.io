---
layout: post
title: TCPIP - ARP, NDP, ICMP
date: 2025-08-29 18:25:23 +0900
category: TCPIP
---
# 4. ARP / NDP / ICMP
**(🔰 ARP 동작·GARP·ARP 캐시, 🔰 NDP/RA/RS·DAD, ⚙️ ICMP 에러/에코와 진단 활용)**

> 이 글은 **주소 ↔ 링크 계층 주소 매핑(ARP/NDP)** 과 **오류·진단 메시지(ICMP/ICMPv6)** 를 **현장 관찰 포인트**와 함께 정리합니다.  
> 코드 대신 **패킷 필드·다이어그램·명령 출력(예시)** 중심이며, 수식은 MathJax로 표기합니다.

---

## 4.1 왜 ARP / NDP / ICMP 인가
- **ARP(IPv4)** & **NDP(IPv6)**: 같은 L2 도메인에서 **IP → MAC** 매핑을 해결합니다. 이 단계가 불안정하면 **핑 타임아웃·간헐 끊김**이 L3/L7 문제처럼 보입니다.
- **ICMP**: 경로 오류/혼잡·MTU 문제를 **표준 방식으로 알림**. ICMP를 막으면 **PMTUD 블랙홀** 등 **“큰 응답만 끊김”** 문제가 생길 수 있습니다.

---

## 4.2 ARP(IPv4) 동작, GARP, ARP 캐시  (🔰)

### 4.2.1 ARP란?
- 같은 L2(브로드캐스트 도메인)에서 **IPv4 → MAC**을 알아내기 위한 프로토콜.
- **프레임 대상**: ARP 요청은 L2 브로드캐스트(`ff:ff:ff:ff:ff:ff`), 응답은 보통 유니캐스트.

#### ARP 패킷(핵심 필드)
```
Hardware Type  : 1 (Ethernet)
Protocol Type  : 0x0800 (IPv4)
HLEN/PLEN      : 6/4 (MAC 6바이트 / IPv4 4바이트)
Opcode         : 1=Request, 2=Reply
Sender MAC/IP  : 요청자 MAC/IP
Target MAC/IP  : (요청 시) 미상/질의 대상 IP
```

### 4.2.2 ARP 기본 시나리오
```text
Host A (192.0.2.10, MAC A)
Host B (192.0.2.20, MAC B)

1) A가 B에게 IP 패킷을 보내려 함 → B의 MAC을 모름
2) A: "누가 192.0.2.20 이니? MAC 알려줘!"  (ARP Request, 브로드캐스트)
3) B: "나야! 내 MAC은 B야."                (ARP Reply, 유니캐스트)
4) A는 (192.0.2.20 → MAC B)를 캐시에 저장하고 L2 유니캐스트로 통신
```

#### 명령 예시(실행 컨셉)
```text
# Linux
ip neigh show
# Windows
arp -a
```

> **관찰 포인트**: “ARP 요청 폭증”은 브로드캐스트 부하와 함께 전체 지연을 키울 수 있으며, **루프/스톰**의 단서가 되기도 합니다.

### 4.2.3 GARP(Gratuitous ARP)
- **ARP 요청**이지만 **대상 IP = 송신자 IP** (응답을 기대하지 않음).
- **용도**
  1) **IP 충돌 탐지**: 동일 IP를 가진 다른 노드가 응답하면 충돌.
  2) **ARP 캐시 갱신**: HA(예: VRRP) 시 Active 인스턴스가 바뀌면, 주변 장비 캐시를 빠르게 새 MAC으로 업데이트.
  3) **이동 알림**: VM/서버가 NIC 이동·vMotion 된 뒤 경로 학습 촉진.

#### GARP 예시(개념)
```text
VRRP 마스터 장애 → 백업이 마스터 승급
→ 백업 장비가 VIP에 대해 GARP 전송
→ 스위치/호스트 ARP 캐시가 새 MAC으로 갱신
→ 서비스 다운타임 단축
```

### 4.2.4 ARP 캐시와 상태
- **캐시 항목 상태**(OS별 표기 다름): INCOMPLETE/REACHABLE/STALE/DELAY/PROBE 등.
- **에이징**: 일정 시간 통신이 없으면 STALE → 필요 시 재검증(NS/Probe) 후 REACHABLE.
- **문제 패턴**
  - **플래핑**: 같은 IP가 다른 MAC으로 자주 바뀜(HA 전환, 루프, 스푸핑 의심).
  - **캐시 포이즈닝**: 악의적 스푸핑으로 게이트웨이 MAC을 공격자 MAC으로 바꿔 L3 단절/중간자 공격 가능 → **DAI(동적 ARP 검사)** 등 방어 필요.

> **보안 메모**: 여기서는 **공격 방법**이 아닌 **방어 개념**만 다룹니다. 엔터프라이즈 네트워크는 **DAI/포트 보안/분리·세그멘테이션**을 권장합니다.

---

## 4.3 NDP(IPv6): RS/RA, NS/NA, DAD (🔰)

> IPv6에는 ARP가 없고, **ICMPv6 기반의 NDP**가 그 역할을 합니다. IPv6는 **브로드캐스트가 없으며**, 필수 멀티캐스트를 사용합니다.

### 4.3.1 NDP 메시지 타입(핵심)
- **Router Solicitation (RS, Type 133)**: 호스트 → 라우터. “RA 좀 주세요.”
- **Router Advertisement (RA, Type 134)**: 라우터 → 호스트. 프리픽스·게이트웨이·MTU·M/O 플래그 등.
- **Neighbor Solicitation (NS, Type 135)**: 이웃의 MAC 질의, 또는 DAD.
- **Neighbor Advertisement (NA, Type 136)**: NS에 대한 응답(자신의 L2 주소 알림).
- **Redirect (Type 137)**: 더 좋은 다음 홉 알림(보안상 제한적 사용 권장).

### 4.3.2 RA/RS와 SLAAC
- **SLAAC**: RA가 제공하는 보라벨(prefix) 정보로 호스트가 **자율적으로 주소 구성**.
- **M/O 플래그**
  - **M(Managed)**=1 → DHCPv6로 주소/옵션 받기
  - **O(Other)**=1 → 주소는 SLAAC, 추가 옵션은 DHCPv6
- **RDNSS(Recursive DNS Server) 옵션**: RA에 DNS 정보 포함 가능(RFC 8106).

#### RS/RA 흐름(예시)
```text
호스트 부팅 → RS 전송(ff02::2 모든 라우터)
라우터 → RA 전송(프리픽스 2001:db8:1::/64, MTU, 게이트웨이, RDNSS 등)
호스트 → 인터페이스 식별자 + 프리픽스로 글로벌 주소 구성 (SLAAC)
```

### 4.3.3 NDP의 이웃 탐색(NS/NA)
- **Solicited-Node 멀티캐스트**(ff02::1:ffXX:XXXX) 활용: 필요한 대상만 깨워 효율적.
- **NS**: “이 IPv6 주소의 L2 주소는?”  
- **NA**: “내 MAC은 이것이야.” (플래그: Solicited, Override 등)

#### 예시(개념)
```text
클라이언트(2001:db8:1::abcd:1234)가 게이트웨이(2001:db8:1::1)의 MAC을 모름
→ 대상의 Solicited-Node 그룹으로 NS 전송
→ 게이트웨이가 NA로 MAC 알림
→ 이후 유니캐스트 프레임 수송
```

### 4.3.4 DAD(Duplicate Address Detection: 중복주소 탐지)
- 호스트가 주소를 **tentative(가시적 사용 전)** 상태로 잡고 **NS를 자신의 주소에 대해** 송신(출발지 `::`).
- **응답(NA)이 오면 충돌** → 주소 사용 포기/재시도.
- **가상화·HA 전환·컨테이너** 환경에서 DAD 미스 시 순간 충돌/끊김 가능 → RA/RS 타이밍·ND 프록시 고려.

### 4.3.5 NDP 보안 개요
- **RA-Guard**: 비인가 RA를 차단(스위치 엣지).  
- **ND Inspection**(벤더별 명칭): IPv6에 대한 ARP Inspection 유사 개념.  
- **SEND(SeND)**: Cryptographically Generated Address로 ND 보안 강화(보급은 제한적).

---

## 4.4 ICMP / ICMPv6: 에러·에코, 진단 활용 (⚙️)

### 4.4.1 ICMP(IPv4) & ICMPv6(IPv6) 큰 그림
- **오류 알림과 진단**을 위한 제어 메시지. 라우터/호스트가 **왜 패킷을 전달/처리할 수 없었는지** 표준화된 이유를 돌려줍니다.
- **막으면**: PMTUD 실패, 트레이스/핑 불가, 원인 분석 난이도 급상승.

### 4.4.2 대표 타입·코드(요약)

#### IPv4 ICMP (일부)
| Type | 이름 | 주요 코드 | 의미/활용 |
|---|---|---|---|
| 0   | Echo Reply | 0 | 핑 응답 |
| 3   | Destination Unreachable | 0=Net, 1=Host, 3=Port, 4=Frag Needed | 경로/포트 불가, **Frag Needed**는 PMTUD 핵심 |
| 4   | Source Quench (구식) | - | 혼잡 알림(폐기됨) |
| 5   | Redirect | - | 더 좋은 게이트웨이(보안상 제한적) |
| 8   | Echo Request | 0 | 핑 요청 |
| 11  | Time Exceeded | 0=전송, 1=재조립 | 트레이스루트 핵심 |

#### IPv6 ICMPv6 (일부)
| Type | 이름 | 의미 |
|---|---|---|
| 1   | Destination Unreachable | 도달 불가 |
| 2   | Packet Too Big | **PMTUD 핵심**(경로 MTU 알림) |
| 3   | Time Exceeded | 홉 제한 초과(트레이스) |
| 4   | Parameter Problem | 헤더 문제 |
| 128 | Echo Request | 핑 요청 |
| 129 | Echo Reply | 핑 응답 |
| 133 | Router Solicitation | NDP |
| 134 | Router Advertisement | NDP |
| 135 | Neighbor Solicitation | NDP |
| 136 | Neighbor Advertisement | NDP |
| 137 | Redirect | NDP |

> **핵심**: IPv4의 `Frag Needed`와 IPv6의 `Packet Too Big`은 **PMTUD에 필수**입니다.

### 4.4.3 Traceroute의 원리
- **TTL/Hop Limit을 1부터 증가**시키며 전송 → 각 홉 라우터가 **Time Exceeded** ICMP를 회신 → **경로 지도** 획득.
- UDP/ICMP/TCP 기반 구현이 있으며, 방화벽 정책에 따라 일부 타입이 차단될 수 있음.

### 4.4.4 PMTUD와 ICMP 필터링
- PMTUD는 **ICMP에 의존**(v6은 `Packet Too Big`, v4는 `Frag Needed`).
- **과도한 ICMP 차단**은 “작은 요청 OK, **큰 응답만 Hang**” 과 같은 **블랙홀**을 유발.

> **실무 팁**: 경계 방화벽에서는 **필수 ICMP 타입·코드만 허용**(예: Too Big/Frag Needed, Time Exceeded)을 권장. **모두 차단 ≠ 보안 강화**가 아닙니다.

---

## 4.5 엔드투엔드 예제(현장 체감)

### 4.5.1 “작은 JSON은 되는데, 큰 다운로드만 끊긴다”
```text
증상: 작은 API 호출 OK, 큰 파일 다운로드 Hang/느림
가설: 오버레이(VXLAN)/QinQ/태깅으로 실효 MTU ↓ + ICMP 필터링 → PMTUD 실패
관찰: (v4) Frag Needed 없음 / (v6) Packet Too Big 없음, 재전송 반복
조치: 필수 ICMP 허용, MSS 클램핑, 경로 MTU 표준화
결과: 대용량 응답도 정상
```

### 4.5.2 “갑자기 같은 층만 핑이 튄다”
```text
증상: 동일 VLAN 사용자들이 간헐적 타임아웃/지연
가설: ARP 요청 폭증(루프/스톰), 게이트웨이 MAC 플래핑
관찰: 스위치에서 MAC move/TC 이벤트, 브로드캐스트 카운터 상승, ARP 테이블 급변
조치: STP/RSTP 점검, BPDU Guard/PortFast, 스톰 컨트롤, 사용자 허브 제거
결과: 안정화
```

### 4.5.3 “IPv6만 느리다 (Happy Eyeballs로 v4가 더 빨리 선택)”
```text
증상: 듀얼스택 환경에서 v6 연결이 일관되게 느려 v4로 폴백
가설: RA/RS 지연, NDP 캐시 문제, MTU/Packet Too Big 차단
관찰: RA 간격/지연, NS/NA 응답 시간, Too Big 부재
조치: RA-Guard 재검토(오탐 차단), ND Inspection, MTU 경로·ICMPv6 허용 조정
결과: v6 경로도 안정적 선택
```

---

## 4.6 운영 체크리스트

### 4.6.1 ARP / NDP
```text
- ARP/NDP 캐시 상태: INCOMPLETE/STALE 과다 여부
- GARP/NA(Override) 이벤트: HA 전환 시점과 사용자 끊김 상관
- 브로드캐스트/멀티캐스트 트래픽 비율(도메인 과대 여부)
- DAI/RA-Guard/ND Inspection 활성화(오탐/미적용 점검)
- 게이트웨이 MAC 플래핑/이동 흔적
```

### 4.6.2 ICMP / PMTUD
```text
- 필수 ICMP 타입/코드 허용(Frag Needed / Packet Too Big / Time Exceeded)
- 경로 MTU 표준값과 MSS 클램핑 일관성
- 트레이스(시간대별)로 경로 길이·손실·Too Big 관측
```

### 4.6.3 로그·가시성
```text
- 스위치: MAC Move/TC, 스톰 카운터, BPDU Guard 트리거
- 라우터/방화벽: ICMP rate-limit/드롭 로그
- 호스트: ip neigh / ndp 테이블 상태, DAD 실패 사례
```

---

## 4.7 미니 실습(의도 중심)

### 4.7.1 “GARP로 HA 전환 체감”
```text
의도: VRRP/Keepalived 전환 시 ARP 갱신 체감
절차: Active 장애 유도 → 백업 승급(GARP 발생) → 단말 통신 중단 시간 측정
기록: 끊김 시간(평균/최대), 스위치 로그의 MAC 이동
결과: GARP 유무/수량에 따른 복구 시간 차이 체감
```

### 4.7.2 “NDP DAD 관찰”
```text
의도: IPv6 주소 충돌 방지 절차 이해
절차: 동일 링크에서 중복 주소 설정 시도(테스트망) → NS(DAD) 수신/응답 관찰
결과: DAD 실패 시 주소 미활성화/재시도 로직 확인
```

### 4.7.3 “PMTUD 블랙홀 재현”
```text
의도: 큰 응답 실패 ↔ ICMP 차단의 상관
절차: DF 설정 큰 패킷 전송 → (v4) Frag Needed / (v6) Too Big 수신 여부 확인
대안: ICMP 허용 전후, MSS 클램핑 전후 비교
결과: 필수 ICMP 차단이 야기하는 블랙홀 확인
```

### 4.7.4 “Traceroute로 비대칭 경로 감지”
```text
의도: 왕복 경로가 다를 때 진단의 어려움 체감
절차: 양방향에서 트레이스 → 홉/RTT 차이 기록
결과: L4/L7 증상이 L3 정책/비대칭 때문임을 연결
```

---

## 4.8 수치·수식 메모

### 4.8.1 ARP/NDP 트래픽과 부담
- 브로드캐스트/멀티캐스트 증가로 **모든 노드가 인터럽트**를 받으므로,  
  링크가 빠른 환경에서도 **CPU 지연**이 늘 수 있습니다.
- 도메인 분할(VLAN), **프록시 ARP/ND**, 캐시 타이머 튜닝, 스톰 컨트롤로 완화.

### 4.8.2 PMTUD와 처리량 직관
- **큰 세그먼트가 지속 드롭**되면 TCP는 재전송/혼잡 윈도우 축소 → 처리량 급감.  
- 안전판으로 **MSS 클램핑**을 두되, 장기적으로 **경로 MTU 정합/ICMP 허용**이 근본 해결.

\[
\text{실효 MSS} = \text{MTU} - \bigl(\text{IP 헤더} + \text{TCP 헤더} + \text{옵션/오버레이}\bigr)
\]

---

## 4.9 베스트 프랙티스(요약 카드)
- **ARP/NDP 안정화**: VLAN 세분화, DAI/RA-Guard/ND Inspection, GARP/NA 활용, 캐시·타이머 일관성.
- **ICMP 필수 허용**: (v4) **Frag Needed**, (v6) **Packet Too Big**, **Time Exceeded**. “ICMP 올차단”은 **안티 패턴**.
- **문서화**: 게이트웨이 MAC, VRRP/HSRP/VIP, ND/ARP 정책, MTU·MSS 기준.
- **가시성**: MAC Move/TC/스톰/ICMP 드롭 알람, ND/ARP 실패 카운터 수집.
- **변경관리**: HA 전환·펌웨어 업그레이드·VLAN 변경 시 **변경 창/롤백**.

---

## 4.10 한 장 요약(포스터)
- **ARP(IPv4)**: IP→MAC 브로드캐스트 질의/유니캐스트 응답, **GARP**로 충돌탐지/캐시갱신.  
- **NDP(IPv6)**: RS/RA로 경로·프리픽스, NS/NA로 이웃 MAC, **DAD**로 중복 방지. 브로드캐스트 **없음**, 필수 멀티캐스트 사용.  
- **ICMP/ICMPv6**: PMTUD·트레이스·오류 알림의 생명선. **필수 타입 차단 금지**.  
- **현장 증상**: “작은 건 OK/큰 건 Fail”, “같은 층만 핑 튐”, “IPv6만 느림” → **L2/ICMP/MTU**부터 점검.

---

### 부록 A — 명령 출력 예시(개념)

```text
# Linux: 이웃 테이블
$ ip neigh
192.0.2.1 dev eth0 lladdr 00:11:22:33:44:55 REACHABLE
2001:db8:1::1 dev eth0 lladdr 00:11:22:33:44:66 STALE

# Windows: ARP 테이블
C:\> arp -a
인터페이스: 192.0.2.10 --- 0x9
  인터넷 주소      물리적 주소           형식
  192.0.2.1        00-11-22-33-44-55     동적
```

```text
# 스위치(벤더 예시 개념): ARP/NDP Inspection/RA-Guard 상태
Switch# show ip arp inspection
Switch# show ipv6 nd inspection
Switch# show ipv6 ra-guard policy
```

### 부록 B — 패킷 필드 스냅샷(요약)

```text
ARP Request:
- htype=1, ptype=0x0800, hlen=6, plen=4
- op=1 (request)
- sha=MAC(A), spa=192.0.2.10
- tha=00:00:00:00:00:00, tpa=192.0.2.20
(L2 dst=ff:ff:ff:ff:ff:ff)

ICMPv6 Neighbor Solicitation:
- Type=135 (NS), Code=0
- Target Address=2001:db8:1::1
- Option: Source Link-Layer Address (요청자 MAC)
(L2 dst=ff02::1:ff00:0001의 맵핑 멀티캐스트 MAC)
```

### 부록 C — 관찰 템플릿

```text
[날짜/구간] L2 VLAN-10(사무동 3층) / L3 GW(192.0.2.1) / 방화벽(경계)
[증상] 큰 파일 다운로드 Hang, ARP 요청 급증, v6만 느림
[가설] PMTUD 블랙홀, 게이트웨이 MAC 플래핑, RA 오탐 차단
[근거] ICMP Too Big/Frag Needed 부재, MAC Move/TC 로그, RA-Guard 카운터
[조치] 필수 ICMP 허용, BPDU Guard/Storm Control, RA-Guard 정책 조정, MSS 클램핑
[결과/후속] 복구 시간, 표준 MTU/MSS/ND/ARP 정책 문서 업데이트
```