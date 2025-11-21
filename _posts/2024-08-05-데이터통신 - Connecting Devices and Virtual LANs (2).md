---
layout: post
title: 데이터 통신 - Connecting Devices and Virtual LANs (2)
date: 2024-08-05 21:20:23 +0900
category: DataCommunication
---
# Chapter 17.2 Virtual LANs (VLANs)

> 목표: **하나의 물리적 이더넷 스위치/백본** 위에 **여러 개의 “논리적 LAN”**을 만드는
> Virtual LAN(VLAN)의 개념을 **회원 구성(membership)** → **구성(configuration)** →
> **스위치 간 통신(communication between switches)** → **장점(advantages)** 순으로 정리한다.
> IEEE 802.1Q 표준과 벤더(주로 미국/유럽) 문서에서 사용하는 최신 개념을 기준으로 한다.

---

## VLAN Membership

### VLAN이란? — “논리적 브리지 도메인”

VLAN은 **하나의 물리 스위치/망을 여러 개의 L2 브로드캐스트 도메인으로 가상 분할한 것**이다.

- 같은 VLAN에 속한 호스트끼리는 **브로드캐스트/ARP**를 공유하고, L2 스위치만으로도 직접 통신 가능.
- 서로 다른 VLAN은 **서로 다른 브로드캐스트 도메인**이며, **라우터(L3)나 L3 스위치가 있어야만** 통신 가능(Inter-VLAN Routing).

IEEE 802.1Q 태그에는 **12비트 VLAN ID(VID)**가 있어,

$$
N_{\text{VLAN}} = 2^{12} = 4096
$$

개의 VLAN ID를 표현할 수 있지만,
일반적으로 **0과 4095는 예약**되고, **1–4094가 실제 VLAN ID로 사용**된다.

---

### VLAN Membership의 종류

스위치가 “이 포트/호스트는 어떤 VLAN에 속하는가?”를 결정하는 기준이다.
현대 네트워크에서는 여러 방식을 **혼합**해서 사용한다.

| 기준 | 설명 | 용도 |
|------|------|------|
| **포트 기반(Port-based)** | “포트 X = VLAN 10”처럼 포트 번호로 결정 | 가장 일반적, 정적 구성 |
| **MAC 기반(MAC-based)** | 접속 호스트의 MAC 주소로 VLAN 결정 | Lab, BYOD, 특정 장비 분류 |
| **프로토콜/IP 기반** | IP 서브넷, 프로토콜 타입(Ethertype) 기반 | Multi-protocol 환경, 일부 고급 스위치 |
| **정책/사용자 기반 (802.1X + RADIUS)** | 사용자 ID, 인증 결과로 VLAN 할당 | NAC, 엔터프라이즈 보안, 무선/유선 통합 |

아래에서 하나씩 살펴보자.

---

### Port-based VLAN Membership (정적 VLAN)

가장 기본이자 시험/실무에서 **가장 많이 쓰는 방식**이다.

- 스위치 포트에 **“Access VLAN”을 지정**한다.
- 해당 포트에 연결된 호스트는 모두 그 VLAN의 **회원**이 된다.
- “사용자가 자리를 옮기면, 그 포트의 VLAN을 다시 설정해야” 하는 방식.

예:

- 포트 1–10: VLAN 10 (Engineering)
- 포트 11–20: VLAN 20 (HR)
- 포트 21–24: VLAN 30 (Guest)

이럴 때:

- 포트 3에 연결된 호스트는 **무조건 VLAN 10**.
- 포트 12에 연결된 호스트는 **무조건 VLAN 20**.

#### 포트 기반 VLAN 간단 예제 (Cisco 스타일 CLI)

```text
vlan 10
 name ENG
vlan 20
 name HR
vlan 30
 name GUEST

interface range gigabitethernet0/1 - 10
 switchport mode access
 switchport access vlan 10

interface range gigabitethernet0/11 - 20
 switchport mode access
 switchport access vlan 20

interface range gigabitethernet0/21 - 24
 switchport mode access
 switchport access vlan 30
```

이처럼 **port → VLAN**을 직접 매핑하면 단순하지만,
사용자가 자리를 자주 옮기거나, 회의실/공유공간에서는 관리가 번거롭다.

---

### MAC-based VLAN

**호스트의 MAC 주소**를 기반으로 VLAN을 결정하는 방식이다.

- “이 MAC 주소는 VLAN 10, 저 MAC은 VLAN 20”과 같이 **MAC → VLAN** 매핑 테이블을 가진다.
- 포트가 어디든 간에, 해당 MAC이 접속하면 **동일 VLAN**을 부여할 수 있다.

일부 벤더는 이를 VMPS(Cisco의 구형 매커니즘) 또는 정책 기반 VLAN으로 제공했지만,
최근에는 **RADIUS/802.1X 기반 정책 VLAN**이 더 널리 사용된다.

장점:

- 사용자가 어느 포트에 꽂든 **동일한 VLAN 정책** 적용.
- 노트북/단말을 들고 다니는 환경에 도움.

단점:

- MAC 주소 관리가 번거롭고, MAC 스푸핑에 취약.
- 큰 네트워크에서는 관리 스케일 문제.

---

### IP/프로토콜 기반 VLAN (고급/특수 경우)

일부 스위치는 **프레임의 상위 계층(프로토콜/IP)**를 보고 VLAN을 결정하기도 한다.

예:

- IP 헤더의 **서브넷(192.168.10.0/24)**에 따라 VLAN 10 할당
- IPv4/IPv6, ARP 등 **프로토콜 타입**에 따라 VLAN 분리
- PPPoE, MPLS 같은 특수 프로토콜에 대한 별도 VLAN 정책

이는 Multi-protocol 환경(예: IPv4/IPv6/또는 이더넷에 encapsulated된 다른 L3 프로토콜)이나
서비스 Provider 환경에서 사용되나, **일반 엔터프라이즈 액세스**에서는 잘 쓰이지 않는다.

---

### 동적 VLAN: 802.1X + RADIUS

최근(특히 2020년대 이후) 엔터프라이즈에서는 **802.1X 포트 기반 인증 + RADIUS**를 통해
사용자/단말이 **성공적으로 인증되면 VLAN을 동적으로 할당**하는 방식이 널리 쓰인다.

흐름:

1. 단말이 스위치 포트에 접속.
2. 802.1X(EAP) 핸드셰이크를 통해 **RADIUS 서버**에 사용자 인증 요청.
3. 인증 성공 후, RADIUS가 **VLAN ID(예: Tunnel-Private-Group-ID 속성)**를 포함해 응답.
4. 스위치 포트는 해당 VLAN으로 전환 (또는 특정 매핑 정책 적용).

이렇게 하면:

- **포트는 “색깔 없음(colorless)”**: 포트 설정은 동일하고,
  접속한 **사용자/단말의 유형에 따라 VLAN이 달라지는 구조**를 만들 수 있다.

#### 동적 VLAN 흐름을 단순화한 의사코드

```python
def assign_vlan(user_role):
    # RADIUS에서 내려온 사용자/단말 타입에 따른 VLAN 정책
    if user_role == "employee":
        return 10
    elif user_role == "guest":
        return 30
    elif user_role == "voip_phone":
        return 50
    else:
        return 100  # quarantine VLAN

# 예시

for role in ["employee", "guest", "voip_phone", "unknown"]:
    vlan = assign_vlan(role)
    print(f"역할 {role} -> VLAN {vlan}")
```

실제 장비에서는 RADIUS 서버(예: ISE, FreeRADIUS)와 스위치가
표준/벤더 전용 RADIUS 속성을 주고받으며 비슷한 로직으로 VLAN을 결정한다.

---

## VLAN Configuration

이제 실제 **구성(configuration)** 관점에서 VLAN을 본다.

### Access Port vs Trunk Port

IEEE 802.1Q 세계에서 스위치 포트는 일반적으로 두 종류로 나뉜다.

- **Access 포트**
  - 일반 PC/서버/프린터 등이 연결.
  - **한 VLAN**에만 속함.
  - 프레임은 대개 **untagged**로 주고받고, 스위치가 내부에서 VLAN을 구분.
- **Trunk 포트**
  - **스위치–스위치, 스위치–라우터(ROAS), 스위치–AP** 등을 연결.
  - **여러 VLAN**을 하나의 물리 링크 위로 전송.
  - 802.1Q 태그를 이용해 프레임마다 VLAN ID 부여.

Access/Trunk 포트와 VLAN ID 간의 관계를 간단히 도식화하면:

```text
[호스트] --(untagged)--> [Access 포트: VLAN 10] ==(tagged VLAN 10)==> [Trunk] == ...
```

Access 포트에서 들어온 untagged 프레임은 스위치 내부에서
“Access VLAN ID”로 분류된 다음, 트렁크로 나갈 때 **802.1Q 태그(=VLAN ID)**를 붙여 전송된다.

---

### 802.1Q 태그 구조

802.1Q 표준은 이더넷 프레임에 **4바이트(32비트) 태그**를 삽입한다.

프레임 구조(단순화):

```text
+-----------+-----------+----------+----------------+----------+-----------+
| Dest MAC  | Src MAC   | 802.1Q   | EtherType/Len  | Payload  |   FCS     |
| 6 bytes   | 6 bytes   | 4 bytes  | 2 bytes        | ...      | 4 bytes   |
+-----------+-----------+----------+----------------+----------+-----------+
```

802.1Q 태그(4 bytes)는 다시 다음과 같이 나뉜다.

- **TPID (Tag Protocol Identifier, 16비트)**: 값 0x8100 → “이 프레임은 dot1Q 태그가 있다”는 표시.
- **TCI (Tag Control Information, 16비트)**:
  - PCP (Priority Code Point, 3비트): 802.1p QoS 우선순위.
  - DEI (Drop Eligible Indicator, 1비트): 혼잡 시 drop 우선순위 표시.
  - VID (VLAN ID, 12비트): 실제 VLAN 번호(0–4095 중 보통 1–4094 사용).

즉, 하나의 프레임에 **우선순위 + Drop 여부 + VLAN ID** 정보를 실어 보낼 수 있다.

---

### VLAN ID 범위와 예약

Cisco/Juniper 등 미국/유럽 벤더 문서를 종합하면, 보통 VLAN ID는 다음처럼 나눠 사용한다.

| VLAN ID | 용도 |
|---------|------|
| 0 | Priority tagging(PCP/DEI만) 등 특수 목적, 일반 VLAN X |
| 1 | 기본 VLAN(Default VLAN) – 일부 플랫폼에서 관리적으로 특별 취급 |
| 2–1001 | “일반 VLAN(Normal Range)” – 가장 흔히 사용하는 범위 |
| 1002–1005 | Token Ring/FDDI 레거시용(현대에는 사실상 사용 X) |
| 1006–4094 | 확장 VLAN(Extended Range) – 일부 장비에서 별도 처리 |
| 4095 | 예약 |

실제 장비에서는, 예를 들어:

- Cisco Nexus: VLAN 1–4094 지원, 4094는 예약 등.
- Juniper EX: VLAN 1–4094 사용 가능, 0과 4095는 예약.

---

### Access 포트 구성 예제

가장 기본적인 포트 VLAN 설정 예시 (Cisco 스타일):

```text
vlan 10
 name ENG
vlan 20
 name HR

interface gigabitethernet0/1
 description "Engineer PC"
 switchport mode access
 switchport access vlan 10

interface gigabitethernet0/2
 description "HR PC"
 switchport mode access
 switchport access vlan 20
```

- Gi0/1 포트에는 VLAN 10이 할당 → 해당 포트의 호스트는 ENG VLAN의 멤버.
- Gi0/2 포트에는 VLAN 20이 할당 → HR VLAN의 멤버.

---

### Trunk 포트 구성 예제

두 스위치 간에 여러 VLAN을 싣고 다니려면 **Trunk 포트**를 사용해야 한다.

```text
interface gigabitethernet0/24
 description "Uplink to Switch B"
 switchport trunk encapsulation dot1q   ! (플랫폼에 따라 자동)
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 switchport trunk native vlan 99
```

- VLAN 10, 20, 30 프레임은 **tagged**로 이 포트를 통해 전달.
- VLAN 99는 **native VLAN**:
  - 이 포트로 들어오는 untagged 프레임은 VLAN 99로 분류.
  - 이 포트에서 나가는 VLAN 99 프레임은 tag 없이 나갈 수 있음(장비/설정에 따라 다름).

---

### Linux에서 VLAN 인터페이스 구성 예 (코드)

리눅스 서버에서 802.1Q VLAN 인터페이스를 만들 때는 다음과 같이 한다:

```bash
# eth0에 VLAN 10 서브인터페이스 생성

ip link add link eth0 name eth0.10 type vlan id 10

# IP 주소 부여

ip addr add 192.168.10.10/24 dev eth0.10

# 인터페이스 up

ip link set dev eth0.10 up
```

- 스위치 쪽에서는 eth0이 연결된 포트를 **Trunk 포트**로 만들고 VLAN 10을 허용해야 한다.
- 리눅스는 eth0 위에 “가상 인터페이스” eth0.10을 만들어 VLAN 10 트래픽만 처리한다.

---

### 예제: 간단 VLAN 설계와 IP 서브넷 매핑

실무에서 흔한 패턴:

| VLAN | 이름 | IP 서브넷 |
|------|------|-----------|
| 10 | ENG | 10.10.10.0/24 |
| 20 | HR | 10.10.20.0/24 |
| 30 | GUEST | 10.10.30.0/24 |
| 99 | NATIVE/MGMT | 10.10.99.0/24 |

- 스위치의 SVI(또는 라우터의 서브인터페이스)에 이 서브넷별 게이트웨이 IP를 부여하고,
  이를 통해 **Inter-VLAN Routing**을 수행한다.

---

## Communication Between Switches

VLAN은 보통 **여러 스위치에 걸쳐 확장**된다.
이때 스위치들 간의 통신은 주로 **802.1Q Trunk**를 통해 이루어진다.

### Access Link vs Trunk Link (다시 정리)

- **Access Link**
  - 단일 VLAN만 운반.
  - 프레임은 보통 **untagged**.
- **Trunk Link**
  - 여러 VLAN 프레임을 하나의 링크 위로 운반.
  - 각 프레임에 **802.1Q 태그** 삽입 (TPID=0x8100, PCP/DEI/VID).

### 스위치 두 대 간 VLAN 확장 예

예제 토폴로지:

```text
[Host A]          [Host B]
   |                 |
  SW1 p1          SW2 p1
   | VLAN 10        | VLAN 10
   +------ SW1 p24 --- p24 SW2 ------+
             (Trunk: VLAN 10,20,30)
```

- A와 B는 서로 다른 스위치에 있지만 **둘 다 VLAN 10에 속함**.
- SW1, SW2를 연결하는 p24–p24 링크는 **Trunk**로 설정.

프레임 흐름:

1. Host A → SW1 p1 (Access VLAN 10, untagged).
2. SW1 내부에서 “VLAN 10”으로 분류.
3. SW1 p24(Trunk)로 내보낼 때, 프레임에 **VLAN 10 태그(802.1Q)**를 붙임.
4. SW2 p24(Trunk)에서 프레임 수신:
   - 태그를 보고 “이건 VLAN 10 프레임”이라고 인식.
5. SW2 내부 VLAN 10 브리지 도메인으로 포워딩 → p1(Access VLAN 10)으로 전송.
6. SW2 p1에서 나갈 때는 태그 제거(untagged) 후 Host B에게 전달.

즉, **스위치–스위치 사이에서만 태그를 유지**하고,
호스트–스위치 사이에서는 태그를 감추는 것이 일반적인 모델이다.

---

### 여러 스위치 간 VLAN 설계 예와 코드 시뮬레이션

#### 예제 설계

- VLAN 10: 개발팀(ENG)
- VLAN 20: 인사팀(HR)
- VLAN 30: 손님용(GUEST)

두 스위치 SW1, SW2가 Trunk로 연결되며,
각 스위치의 특정 포트가 위 VLAN들에 Access로 매핑되어 있다고 가정하자.

이를 단순하게 파이썬으로 시뮬레이션해 보자.

```python
class Frame:
    def __init__(self, src, dst, vlan, payload):
        self.src = src
        self.dst = dst
        self.vlan = vlan  # VLAN ID
        self.payload = payload

    def __repr__(self):
        return f"Frame(src={self.src}, dst={self.dst}, vlan={self.vlan}, payload={self.payload})"

class Switch:
    def __init__(self, name):
        self.name = name
        self.access_ports = {}  # port -> vlan
        self.trunk_ports = set()
        self.neighbors = {}  # port -> (neighbor_switch, neighbor_port)
        self.mac_table = {}  # (mac, vlan) -> (port)

    def connect(self, my_port, neighbor, neighbor_port):
        self.neighbors[my_port] = (neighbor, neighbor_port)

    def set_access(self, port, vlan):
        self.access_ports[port] = vlan

    def set_trunk(self, port):
        self.trunk_ports.add(port)

    def receive(self, port, frame):
        print(f"[{self.name}] 포트 {port} 수신: {frame}")
        # 학습
        self.mac_table[(frame.src, frame.vlan)] = port

        # 도착지 포트 찾기
        out_port = self.mac_table.get((frame.dst, frame.vlan))
        if out_port is None:
            # Flooding (같은 VLAN에 한해서)
            for p, nbr in self.neighbors.items():
                if p != port:
                    if p in self.trunk_ports or self.access_ports.get(p) == frame.vlan:
                        nbr_sw, nbr_port = nbr
                        nbr_sw.receive(nbr_port, frame)
        else:
            # Unicast 포워딩
            if out_port != port:
                nbr_sw, nbr_port = self.neighbors[out_port]
                nbr_sw.receive(nbr_port, frame)

# 스위치 2대와 호스트 MAC

sw1 = Switch("SW1")
sw2 = Switch("SW2")

# 포트 연결

sw1.connect(24, sw2, 24)
sw2.connect(24, sw1, 24)

# 포트 유형/ VLAN 설정

sw1.set_trunk(24)
sw2.set_trunk(24)

sw1.set_access(1, 10)   # A: VLAN 10
sw2.set_access(1, 10)   # B: VLAN 10
sw2.set_access(2, 20)   # C: VLAN 20

# 호스트 → 스위치로 들어오는 초기 프레임을 직접 호출해 시뮬레이션

frame_A_to_B = Frame(src="MAC_A", dst="MAC_B", vlan=10, payload="Hello B")
sw1.receive(1, frame_A_to_B)
```

이 매우 단순한 코드는 “**VLAN 10 트래픽만 트렁크를 타고 이동하고,
VLAN 20 트래픽은 별도로 격리**되는” 개념을 보여준다.

실제 스위치는 포트별 Access/Trunk 특성을 더 엄격히 적용하고,
STP, MSTP, RSTP, PVST+ 등 다양한 다중 트리 알고리즘을 사용해
VLAN별 포워딩 경로를 안정적으로 유지한다.

---

### Inter-VLAN Routing과 스위치 간 연계

VLAN은 L2 관점에서 **브로드캐스트 도메인**을 나누는 것이고,
VLAN 간 통신은 반드시 L3 장비(라우터 또는 L3 스위치)를 거쳐야 한다.

전형적인 방법:

1. **Router-on-a-stick (ROAS)**
   - 스위치–라우터 사이를 802.1Q Trunk로 연결.
   - 라우터의 서브인터페이스에 VLAN ID와 IP 게이트웨이 부여.
2. **L3 스위치(SVI)**
   - 스위치 자체에서 VLAN별 인터페이스(SVI)를 만들고, IP를 부여.
   - 스위치가 직접 라우팅 수행(Inter-VLAN Routing).

이 영역은 “VLAN” 자체보다는 “라우팅/게이트웨이” 주제에 더 가깝기 때문에,
세부 라우팅 구성은 별도 장에서 다루되,
**“서로 다른 VLAN = 서로 다른 서브넷 = 라우터 필요”**라는 점만 확실히 기억하면 된다.

---

## Advantages of VLANs

마지막으로 VLAN을 쓰면 어떤 장점이 있는지 정리한다.

### 브로드캐스트 제어와 성능 향상

동일 L2 도메인에 호스트가 많을수록, ARP/브로드캐스트 트래픽도 많아진다.

- 호스트 수를 \(N\), VLAN 수를 \(k\)라고 할 때,
- 이상적으로 호스트가 균등하게 분배되면, VLAN당 호스트는 \(N/k\) 정도가 된다.

각 VLAN 내 브로드캐스트 부하를 \(B\)라고 할 때, 전체 평균 관점에서 보면

$$
B_{\text{per\ host}} \propto \frac{N/k}{N} = \frac{1}{k}
$$

즉, **VLAN 수(k)를 늘려 브로드캐스트 도메인을 잘게 나누면,
각 호스트가 보는 브로드캐스트 부하는 대략 1/k 수준까지 줄어든다고 볼 수 있다.**

실제로는 패턴/응용에 따라 달라지지만, 큰 캠퍼스에서 VLAN을 적절히 나누는 것이
브로드캐스트 스톰과 CPU/메모리 부담을 줄이는 핵심이라는 점은 변하지 않는다.

---

### 보안 분리 및 정책 적용

VLAN은 **논리적 격리(logical isolation)** 수단으로 많이 사용된다.

예:

- 사내망 → VLAN 10 (사원)
- 손님망 → VLAN 30 (Guest)
- 서버망 → VLAN 40 (Server)
- 관리망 → VLAN 99 (Mgmt)

서로 다른 VLAN 간에는 L3 장비(게이트웨이)를 거치지 않으면 통신할 수 없으므로:

- 방화벽/ACL을 **게이트웨이**에 적용해 **VLAN 간 통신을 강력하게 제어**할 수 있다.
- 802.1X + RADIUS를 통해 **사용자 타입에 따라 VLAN을 다르게 할당**하면,
  동일 물리 포트에서 사용자 그룹별로 서로 다른 보안 정책을 적용할 수 있다.

---

### 조직 구조/업무 기반 설계

물리 위치(층/건물)가 아니라 **조직/업무/서비스 단위**로 네트워크를 설계할 수 있다.

- VLAN 10: 개발팀, VLAN 20: 영업팀, VLAN 30: 생산팀 …
- 같은 팀/서버가 여러 층/건물에 흩어져 있어도,
  VLAN을 통해 **“논리적으로 한 LAN”처럼 묶을 수 있음**.

이렇게 하면:

- IP 서브넷/주소 계획을 **조직/서비스 중심**으로 짜기 쉬움.
- 팀 이동/조직 개편 시 VLAN/서브넷 단위로 정책을 변경하기 쉬움.

---

### 멀티테넌시와 서비스 제공

서비스 Provider나 데이터센터에서는 VLAN을 통해 고객/테넌트를 격리한다.

- 각 고객/테넌트에게 **별도 VLAN/서브넷**을 할당.
- 고객 간에는 기본적으로 통신이 불가능하며,
  필요시 L3 구간에서 정책적으로 허용.

단, VLAN ID가 12비트(최대 4094개)로 제한되어
대규모 데이터센터/클라우드에서는 **Q-in-Q(802.1ad)나 VXLAN(24비트 VNI)** 같은
오버레이 기술로 한계를 넘어서기도 한다.

---

### QoS, 트래픽 엔지니어링과의 연계

VLAN 태그에는 VLAN ID 외에도 **PCP(우선순위)**와 **DEI**가 포함되어 있다.

- 특정 VLAN(예: Voice VLAN)에 대해 PCP를 높게 설정하면,
  스위치/라우터에서 **QoS 기반 트래픽 분류/우선순위 처리**를 하기 쉽다.

예:

- VLAN 50: IP 전화(Voice) → PCP=5 이상
- VLAN 10/20: 일반 데이터 → PCP=0 또는 1
- 혼잡 시 DEI=1인 프레임을 먼저 drop

이렇게 하면 하나의 물리 망에서 **음성/영상/데이터**를 공존시키면서도
음성/영상 품질을 일정 수준 보장할 수 있다.

---

### 코드 예제: VLAN 개수와 브로드캐스트 부하 간단 계산

마지막으로, VLAN 수에 따라 각 호스트가 보는 브로드캐스트 부하가
어떻게 변하는지 매우 단순하게 시뮬레이션해 보자.

```python
def broadcast_per_host(num_hosts, num_vlans, base_broadcast_per_vlan=100):
    """
    num_hosts: 전체 호스트 수
    num_vlans: VLAN 수
    base_broadcast_per_vlan: VLAN당 초당 브로드캐스트 프레임 수(가정)
    """
    hosts_per_vlan = num_hosts / num_vlans
    # VLAN마다 브로드캐스트는 base_broadcast_per_vlan
    # 각 호스트는 '자기 VLAN' 브로드캐스트만 본다고 가정
    per_host = base_broadcast_per_vlan / hosts_per_vlan
    return per_host

for k in [1, 2, 4, 8, 16]:
    per_host = broadcast_per_host(num_hosts=800, num_vlans=k)
    print(f"VLAN {k}개일 때, 호스트당 브로드캐스트 강도 ≈ {per_host:.2f} (상대 단위)")
```

출력(상대적인 값)은 대략:

- VLAN 1개: 높은 브로드캐스트 부하
- VLAN 16개: 호스트당 부하는 1/16 수준

실제 네트워크에서는 트래픽 패턴이 훨씬 복잡하지만,
VLAN을 늘려 **브로드캐스트 도메인을 작게 유지하는 것이 성능과 안정성에 중요**하다는
직관을 주는 예제로 볼 수 있다.

---

## 정리

- **Membership**
  - Port-based, MAC-based, Protocol/IP-based, 802.1X+RADIUS 기반 동적 VLAN 등
    여러 방식으로 “어떤 호스트/포트가 어떤 VLAN에 속하는지”를 결정한다.
- **Configuration**
  - IEEE 802.1Q 태그는 TPID(0x8100) + PCP/DEI/VID(12비트 VLAN ID)로 구성되고,
    VLAN ID 1–4094(0,4095 예약)를 사용해 최대 4094 VLAN을 만들 수 있다.
    Access/Trunk 포트, Native VLAN, Voice/Mgmt VLAN, 리눅스/서버 VLAN 인터페이스까지 함께 이해해야 한다.
- **Communication Between Switches**
  - 스위치–스위치, 스위치–라우터, 스위치–AP 등은 대부분 802.1Q Trunk를 사용해
    여러 VLAN 트래픽을 한 링크로 전달한다. 각 프레임의 VLAN ID에 따라
    해당 VLAN 브리지 도메인으로만 포워딩된다.
- **Advantages**
  - 브로드캐스트 도메인 분리 → 부하/성능 향상
  - 보안/정책 분리 → 게이트웨이/방화벽에서 VLAN 간 통신 제어
  - 조직/업무/서비스 단위 설계 → 관리·확장성 향상
  - 멀티테넌시, QoS(PCP/DEI)를 통한 서비스 품질 보장

17.1절의 Hub/Switch/Router 개념 위에 17.2절의 VLAN 이해를 얹으면,
실무에서 사용하는 **Access/Distribution/Core + VLAN + Inter-VLAN Routing + ACL/QoS** 구조를
자연스럽게 해석할 수 있게 된다.
