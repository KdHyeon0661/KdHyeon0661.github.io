---
layout: post
title: 데이터 통신 1장 - Introduction (2)
date: 2024-07-12 19:20:23 +0900
category: DataCommunication
---
# 1.3 Network Types

## 1.3.1 Local Area Network (LAN)

- **정의**: 한정된 영역(가정·소규모 사무실·캠퍼스·데이터센터) 내 장치들을 연결하는 네트워크.  
  스위치, 무선 AP, 라우터 등의 네트워크 장비를 통해 통신한다.
- **주요 기술 스택(현대 LAN의 실무 구성)**  
  - **이더넷(Ethernet)**: 100M/1G/2.5G/5G/10G/25G/40G/100G…  
    매체: UTP, DAC, 광(SR/LR/ER)  
  - **스위칭**: L2 전송, **MAC 학습·포워딩**, **STP/RSTP/MSTP**(루프 방지), **VLAN/Trunk**  
  - **무선 LAN**: 802.11 a/b/g/n/ac/ax(be), **SSID/VLAN 매핑**, **WPA2/WPA3**  
  - **IP 서브넷**: IPv4/IPv6, DHCP, DNS, NAT(가정용), ACL  
  - **가상화/자동화**: VXLAN/EVPN(DC), MLAG/LACP, Netconf/RESTCONF/Ansible

### LAN 설계 핵심 포인트

1) **L2 토폴로지와 루프 방지**
- 스패닝 트리로 루프 차단, **이중화 링크는 LACP(802.3ad)** 로 **논리적 단일 링크**화.
- 액세스–디스트리뷰션–코어의 **계층형(Tree)** 또는 **Spine–Leaf(DC)**.

2) **VLAN·세그먼트화**
- 브로드캐스트 도메인을 분리하여 성능·보안 향상.
- 사용자/서버/관리망/VoIP/Guest 등 논리 분리 + L3 게이트웨이(SVI)에서 라우팅.

3) **QoS와 멀티캐스트**
- 음성/영상/업무 트래픽의 우선순위·대역폭 보장(DSCP/802.1p, 큐잉/셰이핑).
- IPTV/회의 등은 **IGMP 스누핑**으로 브로드캐스트 폭주 방지.

4) **보안**
- 포트 보안(MAC 제한), 802.1X, 동적 VLAN, DHCP 스누핑/DAI, ARP 검사.
- 관리 평면 분리(관리 VLAN, Out-of-Band), 장비 하드닝.

### 간단 실습 1 — VLAN 태깅/트렁크 감 잡기

```
[PC]--(Access VLAN 10)--[SW1]====(Trunk: VLAN 10,20)====[SW2]--(Access VLAN 10)--[Server]
```

- Access 포트: 단일 VLAN에 언태그 프레임
- Trunk 포트: 여러 VLAN을 **태그(802.1Q)** 로 운반

**스위치 구성 예(벤더 중립 Pseudo-CLI)**

```text
interface eth1
  switchport mode access
  switchport access vlan 10

interface eth48
  switchport mode trunk
  switchport trunk allowed vlan 10,20
```

### 간단 실습 2 — 브로드캐스트/플러딩 관찰

```bash
# ARP 브로드캐스트 관찰(리눅스 머신에서)
sudo tcpdump -i eth0 -n 'arp or broadcast'
```

- 같은 VLAN 내에서만 보임. 다른 VLAN이면 **L3 라우팅**이 필요.

---

## 1.3.2 Wide Area Network (WAN)

- **정의**: 지리적으로 떨어진 여러 LAN을 광범위하게 연결하는 네트워크.
- **매체/서비스**: 전용회선(Leased Line), MPLS L3VPN/L2VPN, 브로드밴드(FTTH/케이블/DSL), 위성, 5G/전용 무선.

### WAN 연결 유형

#### 1) Point-to-Point WAN
- **두 지점 간 전용 링크**. 예: 본사–DR 센터 전용회선.  
- 예측 가능한 지연과 대역폭, 보안 우수. **비용↑**.

#### 2) Switched WAN
- 서비스 사업자(운영자)의 **교환망**을 경유해 다수 지점 연결.  
- **MPLS VPN**이 대표적: 라우팅 격리·QoS·트래픽 엔지니어링 가능.

#### 3) Internetwork (네트워크의 네트워크)
- **다수의 네트워크를 라우터로 상호연결**.  
- 정책 라우팅, 경계 보안, 주소 설계, NAT/CGNAT, IPv6 전개까지 포함.

### WAN 라우팅/최적화 포인트

- 라우팅: OSPF/IS-IS(내부), **BGP**(사업자/인터넷 경계)
- 최적화: SD-WAN(다중 회선·어플리케이션 인식·오버레이/암호화), TCP 프록시, 압축.
- 품질: SLA(지연/지터/손실), FEC/ARQ, QoS(DSCP) 보존.

### 예제 — 간단 BGP 에지 감각 익히기

```text
router bgp 65001
  neighbor 203.0.113.1 remote-as 65002
  network 198.51.100.0/24
  maximum-paths 2
  timers 30 90
```

- 외부 피어와 BGP 세션 형성, 보유 프리픽스 광고.

---

## 1.3.3 Switching

**스위칭**은 입력 인터페이스로 들어온 프레임/패킷을 **어디로 보내야 할지 결정**하고 **전달**하는 기능이다.

### 1) 회선 교환(Circuit Switching)

- **특징**: 통화/전송 동안 **독점 회선·고정 대역폭** 할당. 접속 설정에 시간 소요, 설정 이후 **일정 지연**.  
- **장점**: 일정 품질(지연·지터 예측성), 연속 전송 적합(전통 음성).  
- **단점**: 회선 비활성 시간에도 자원 고정 → **비효율**, 다양한 속도/코드 변환 어려움, 비용↑.

#### 간단 수식 — 회선 용량과 점유율
- 통화 채널 수:  
  $$ N = \left\lfloor \frac{C}{B} \right\rfloor $$
  - \(C\): 총 회선 용량, \(B\): 채널당 대역폭
- 평균 점유율:  
  $$ U = \frac{\sum_i t_i}{T} $$
  - \(t_i\): 세션 i의 점유 시간, \(T\): 관측 시간

### 2) 패킷 교환(Packet Switching)

- **메시지를 패킷**으로 나누어 전송, 각 패킷은 독립 라우팅(데이터그램) 또는 **가상회선(VC)** 을 통할 수 있음.
- **장점**: 자원 **통계적 다중화**로 효율↑, 속도/프로토콜 변환 가능, 장애 시 **우회 가능**, 오류 제어·재전송 가능.  
- **단점**: 교착점/혼잡 시 **가변 지연·지터**, 큐잉 지연 증가.

#### 스위치 동작 방식
- **Store-and-Forward**: 전체 프레임 수신 후 FCS 확인 → 전달(오류 차단, 지연↑)
- **Cut-Through**: 목적지 MAC 확인 즉시 포워딩(지연↓, 오류 프레임 통과 가능)
- **Fragment-Free**: 선두 64바이트 확인 후 포워딩(妥協안)

#### 데이터그램 vs 가상회선(VC)
- **데이터그램**: 패킷별 독립 경로, 인터넷 IP.  
- **VC**: 연결 설정 후 동일 경로 사용, **MPLS LSP**, ATM/Frame Relay 유사 개념.

### 예제 — 스위치의 MAC 학습/포워딩 미니 시뮬레이션(파이썬)

```python
from collections import defaultdict

class Switch:
    def __init__(self):
        self.fdb = {}  # MAC -> port

    def learn(self, mac, in_port):
        self.fdb[mac] = in_port

    def forward(self, dst_mac, in_port, ports):
        if dst_mac in self.fdb:
            out_port = self.fdb[dst_mac]
            if out_port != in_port:
                return [out_port]  # 유니캐스트 포워딩
        # 모르면 플러딩
        return [p for p in ports if p != in_port]

sw = Switch()
ports = [1,2,3,4]
# A(MAC AA) 가 포트1에서 B(MAC BB) 로 전송한다고 가정
sw.learn("AA:AA:AA:AA:AA:AA", 1)
print("to unknown BB -> flood:", sw.forward("BB:BB:BB:BB:BB:BB", 1, ports))
sw.learn("BB:BB:BB:BB:BB:BB", 3)
print("to known BB -> unicast:", sw.forward("BB:BB:BB:BB:BB:BB", 1, ports))
```

---

## 1.3.4 The Internet

- **internet(소문자)**: 공통 프로토콜로 상호 연결된 **임의의 네트워크 집합**.  
- **Internet(대문자)**: **TCP/IP** 로 상호연결된 전 세계 네트워크들의 집합(“네트워크의 네트워크”).  
- **ISP(Internet Service Provider)**: 인터넷 접속·전송·백본 서비스를 제공하는 사업자.

### 인터넷의 구조적 레이어

- **Tier-1 캐리어**: 전 세계 상호정산 없이 상호접속 가능한 백본 보유.  
- **Tier-2/3 ISP**: 상위 캐리어 트랜짓 구매/피어링으로 연결.  
- **IXP(Internet Exchange Point)**: 트래픽을 로컬에서 교환해 지연·비용 절감.  
- **CDN(Content Delivery Network)**: 엣지 캐시로 컨텐츠를 근접 제공.

### 경계 라우팅 개요(BGP)

- **AS(자율시스템)** 단위 정책 라우팅, **피어링/트랜짓** 계약.  
- 경로 선택: **최우선 정책(LOCAL_PREF)** → AS PATH → MED → IGP cost…

---

## 1.3.5 Accessing the Internet

> 다양한 **라스트마일**(가정/지사) 접속 기술과 **엔터프라이즈 직결** 유형을 정리한다.

### 1) 전화망 기반

- **Dial-up**: 모뎀으로 PSTN 회선 사용(레거시, 매우 저속).  
- **DSL(ADSL/VDSL)**: 동축 쌍선 기반 고속 전송. 거리 의존, 업/다운 비대칭(ADSL).

### 2) 케이블/DOCSIS

- 케이블 방송 동축망 공유, **DOCSIS** 규격. 다운스트림 대역폭 우수, **세그먼트 혼잡** 변동.

### 3) 광(FTTx)

- **FTTH/FTTP(PON: GPON/XG-PON/25G-PON)**: 고속·저지연·대역폭 대칭 가능.  
- **Active Ethernet**: 전용 광포트, 고품질/비용↑.

### 4) 무선

- **Wi-Fi**: 가정/사무실 액세스, 백홀은 유선.  
- **4G/5G**: eMBB/URLLC/mMTC 등 프로필. **5G FWA(고정 무선)** 로 유선 대체도 가능.  
- **위성(Low Earth Orbit)**: 광역 커버리지, 지연/이중화 고려.

### 5) 엔터프라이즈/기관 전용

- **전용회선**(L2/L3), **MPLS VPN**, **SD-WAN(오버레이+멀티회선)**, **클라우드 전용 회선**(Direct Connect/ExpressRoute 등).

### 접속 품질/용량 추정의 기초

- **대역폭-지연 곱(BDP)**  
  $$ \text{BDP} = \text{Bandwidth} \times \text{RTT} $$
- **TCP 스루풋 근사식(손실률 p, MSS, RTT)**  
  $$ \text{Throughput} \approx \frac{\text{MSS}}{\text{RTT}\sqrt{p}} $$

**간단 계산기(파이썬)**

```python
def bdp_bytes(mbps, rtt_ms):
    return int(mbps * 1_000_000 / 8 * (rtt_ms/1000.0))

def tcp_throughput_mbps(mss_bytes, rtt_ms, p):
    # 매우 단순화한 근사
    import math
    if p <= 0: return float('inf')
    return (mss_bytes * 8) / (rtt_ms/1000.0) / math.sqrt(p) / 1_000_000

print("BDP(100Mbps, 40ms)=", bdp_bytes(100,40), "bytes")
print("TCP Throughput≈", tcp_throughput_mbps(1460,40,1e-3), "Mbps")
```

### NAT/CGNAT/IPv6 메모

- 가정/모바일 접속은 **NAT** 또는 **CGNAT** 뒤에 위치 → **포트 포워딩/UPnP**, P2P 제약.  
- **IPv6** 도입 시 E2E 통신·대규모 주소 공간, 단 정책/방화벽 설계 중요.

---

# 1.4 Internet History

> **초기 이론 → ARPANET → TCP/IP 전환(Flag Day) → NSFNET/상용화 → 웹/브로드밴드/모바일 → 오늘날** 로 이어지는 큰 흐름.

## 1.4.1 Early History

- **패킷 스위칭 이론(1960s)**:  
  - Paul Baran(랜덤 라우팅·내결함성), Donald Davies(“Packet” 용어), Leonard Kleinrock(큐잉 이론 응용).  
  - 핵심 아이디어: **데이터를 패킷으로 쪼개고, 공유망에서 효율적으로 전송**.
- **ARPANET(1969)**:  
  - 최초의 패킷 스위칭 네트워크.  
  - 초창기 4개 노드(UTA, UCLA, SRI, UCSB) → 이후 확장.  
  - NCP(초기 전송 프로토콜) 기반, 이후 TCP/IP 로 진화.

## 1.4.2 Birth of the Internet

- **TCP/IP**: **IP**(라우팅/베스트에포트) + **TCP**(신뢰전송/흐름·혼잡제어).  
- **1983-01-01**: “**Flag Day**” — ARPANET의 **TCP/IP 전환**.  
- **DNS(1983)**: 호스트 파일의 한계를 넘어 **분산 명명 체계** 도입.  
- **CSNET(1981)**: 학술기관 연결, ARPANET 직접 접속 어려운 기관을 포괄.  
- **MILNET**: 군사 트래픽 분리.  
- **NSFNET(1985~1995)**: 연구·교육 네트워크 백본(T1→T3), 상호연결의 기반.  
- **상용화(1990s)**: 백본의 민영화/상업 ISP 성장, **웹(WWW, 1989 제안/1991 공개)** 대중화.

## 1.4.3 Internet Today

- **WWW/브라우저**: Mosaic(1993), Netscape/IE → 크롬/사파리/파폭.  
- **멀티미디어 스트리밍**: HLS/DASH, RTP/RTCP, CDN·엣지 컴퓨팅.  
- **P2P**: 중앙 서버 없이 직접 교환(BitTorrent 등), 분산 시스템·블록체인 응용.  
- **보안 기본값**: HTTPS 보편화, TLS 1.3, **QUIC/HTTP/3** 로 지연 저감·혼잡제어 개선.  
- **클라우드/모바일 퍼스트**: 대규모 오토스케일·글로벌 네트워킹·제로트러스트.

---

# 1.5 Standards and Administration

> **표준화·번호 자원·거버넌스**의 흐름을 이해하면, **인터넷이 왜 호환성과 개방성을 유지**하는지 보인다.

## 1.5.1 Internet Standards (IETF·RFC·STD·BCP)

### RFC와 표준 트랙 개요

- **RFC(Request for Comments)**: 인터넷 기술 문서의 공개 시리즈.  
- **스트림**: IETF, IAB, IRTF, Independent Submission 등.  
- **문서 유형**:
  - **Standards Track**: **Proposed Standard → Internet Standard(STD)**  
    *(Draft Standard 단계는 2011년 RFC 6410 개정으로 폐지되어, 대부분의 실무 표준은 Proposed Standard 단계에서도 폭넓게 채택된다.)*
  - **BCP(Best Current Practice)**: 운영상의 모범사례.  
  - **FYI/Informational/Experimental/Historic**: 참고·실험·역사적 문서.
- **STD 번호**: Internet Standard로 승격 시 **STDxx** 로도 식별.

### 요건 레벨(Requirements Keywords)

- **MUST/SHALL**, **SHOULD/RECOMMENDED**, **MAY/OPTIONAL** 등은 **RFC 2119**, **RFC 8174** 에 정의된 키워드로 해석한다.

### 예시 — RFC 찾고 읽는 팁

```text
# 표준 트랙 예
- RFC 791  : Internet Protocol (역사적)
- RFC 8200 : IPv6 Specification (Internet Standard)
- RFC 793  : TCP (역사적), 최신은 RFC 9293 (TCP Specifications)
- RFC 8446 : TLS 1.3
- RFC 9000 : QUIC
```

### 미니 과제 — RFC 본문에서 키워드 찾기(파이썬)

```python
import re
text = """
Endpoints MUST validate certificates. Clients SHOULD send SNI.
"""
print(re.findall(r"\b(MUST|SHOULD|MAY)\b", text))
```

- 표준 문서 해석 시 MUST/SHOULD 의 차이가 구현 의무를 좌우.

## 1.5.2 Internet Administration (조직/역할)

- **ISOC(Internet Society)**: 인터넷 발전·정책·교육 지원, **IETF/IAB** 후원.  
- **IAB(Internet Architecture Board)**: 아키텍처 감독·자문, RFC 시리즈 전반 정책.  
- **IETF(Internet Engineering Task Force)**: **실무 표준 개발**(WG 중심, 오픈 참여).  
- **IRTF(Internet Research Task Force)**: 장기 연구 주제, RG 중심.  
- **IANA(Internet Assigned Numbers Authority)**: 전역 **번호 자원 관리**(IP 주소 블록, AS 번호, 프로토콜 파라미터, 루트 존 관리 등).  
  - 지역 인터넷 레지스트리(**RIR**: ARIN/RIPE/APNIC/LACNIC/AFRINIC) 로 **주소 자원 분배**.

### 현업 연결 — 도메인/주소/포트 파라미터

- **DNS**: 루트 → TLD → 권한 서버 위임 체계, **IANA 루트 존** 관리.  
- **포트 번호**: 잘 알려진(0–1023), 등록(1024–49151), 동적(49152–65535).  
- **프로토콜 파라미터**: DHCP 옵션, TLS 확장, HTTP 메서드 등 **IANA 레지스트리**로 관리.

---

## 부록: 손에 잡히는 네트워킹 실습 스니펫

### 1) 소켓 에코 서버/클라이언트(로컬 테스트)

```python
# server.py
import socket, threading
def handle(c, addr):
    with c:
        while data := c.recv(4096):
            c.sendall(data)

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.bind(("0.0.0.0", 9000))
    s.listen()
    print("listening 0.0.0.0:9000")
    while True:
        c, addr = s.accept()
        threading.Thread(target=handle, args=(c,addr), daemon=True).start()
```

```python
# client.py
import socket
with socket.create_connection(("127.0.0.1", 9000)) as s:
    s.sendall(b"hello lan/wan")
    print(s.recv(4096))
```

### 2) 경로/지연 가시화(traceroute + 간이 파서)

```bash
traceroute example.com
```

```python
import re, sys
lines = sys.stdin.read().splitlines()
hops = []
for ln in lines:
    m = re.match(r"\s*(\d+)\s+([^\s]+)\s+\(([\d\.]+)\)\s+([\d\.]+)\s+ms", ln)
    if m:
        hops.append((int(m.group(1)), m.group(2), float(m.group(4))))
print("Hops:", len(hops), "Last Hop:", hops[-1] if hops else None)
```

### 3) iperf3 로 접속 품질 보기

```bash
# 서버
iperf3 -s
# 클라이언트 TCP
iperf3 -c <server-ip> -t 20
# 클라이언트 UDP (손실/지터 관찰)
iperf3 -c <server-ip> -u -b 50M -t 20 --get-server-output
```

---

## 요약 체크리스트

- **LAN**: VLAN/스패닝·LACP·QoS·보안(802.1X, DHCP 스누핑)·무선 설계(채널/전력).  
- **WAN**: 전용/사업자 망, MPLS/SD-WAN, SLA(지연·지터·손실), 경계 라우팅(BGP).  
- **스위칭**: 회선 vs 패킷, 데이터그램 vs VC, Store-and-Forward vs Cut-Through.  
- **인터넷 구조**: ISP 티어/IXP/CDN, BGP 정책 라우팅.  
- **접속 기술**: DSL/케이블/FTTH/5G/위성/전용/SD-WAN, NAT·IPv6 고려.  
- **표준/거버넌스**: IETF/IAB/IRTF/ISOC/IANA, RFC/STD/BCP, 요구 키워드 해석.

---

## 연습 문제

1) 1G 링크, RTT 30ms 에서 **BDP**는?  
   $$ \text{BDP} = 1{,}000{,}000{,}000/8 \times 0.03 \approx 3.75\,\text{MB} $$  
   윈도우가 512KB라면 링크 활용률은 대략 얼마인가?

2) MPLS L3VPN 으로 5개 지사를 허브&스포크로 연결한다. 인터넷 백업 회선을 SD-WAN으로 묶어 **어플리케이션 기반 경로 선택**을 구현하는 구성을 개략 설계하라.

3) 스위치가 **Cut-Through** 모드일 때와 **Store-and-Forward** 모드일 때, **짧은 프레임 폭주**와 **오류 프레임 비율 상승** 상황에서 각각의 장단점을 기술하라.
