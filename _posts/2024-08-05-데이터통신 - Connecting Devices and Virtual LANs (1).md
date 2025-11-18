---
layout: post
title: 데이터 통신 - Connecting Devices and Virtual LANs (1)
date: 2024-08-05 20:20:23 +0900
category: DataCommunication
---
# Chapter 17.1 Connecting Devices — Hubs, Link-Layer Switches, Routers

> 목표: 이 장에서는 **허브(hub), 링크 계층 스위치(L2 switch), 라우터(router)**가  
> 이더넷/인터넷에서 **어떤 계층에서**, **어떻게 프레임/패킷을 처리하고**,  
> **충돌 도메인·브로드캐스트 도메인**을 어떻게 나누는지까지 한 번에 정리한다.  
> 이후 VLAN, 다층 스위칭을 이해하기 위한 “기초 블럭”이다.

---

## 1. 큰 그림: 허브–스위치–라우터 비교

먼저 세 장비를 **OSI 계층 / 충돌 도메인 / 브로드캐스트 도메인** 기준으로 비교하자.  
(충돌 도메인/브로드캐스트 도메인 정의는 Cisco/NSRC 등 네트워크 설계 자료 기준.)

| 장비 | OSI 계층 | 프레임/패킷 처리 | 충돌 도메인 | 브로드캐스트 도메인 |
|------|----------|------------------|-------------|----------------------|
| **허브(Hub)** | L1 (물리 계층) | 전기 신호를 증폭·재생해 **모든 포트로 재전송** | **하나** (허브 전체가 공유 매체) | **하나** (프레임이 전 포트로 흘러감) |
| **링크 계층 스위치** | L2 (데이터링크) | MAC 주소를 학습해 **필요한 포트로만 프레임 전달** | **포트마다 하나** (각 포트 독립) | **여전히 하나** (라우터/VLAN 없으면) |
| **라우터** | L3 (네트워크) | IP 헤더를 보고 **다른 네트워크로 패킷 라우팅** | 각 인터페이스별 | **인터페이스별로 분리** (서브넷 단위) |

이 표를 염두에 두고, 각 장비를 상세히 본다.

---

## 2. Hubs — 물리 계층 리피터, “공유 버스”의 잔재

### 2.1 허브의 기본 개념과 동작

**허브(Ethernet hub)**는 **물리 계층(L1)** 장비다.  
허브에 연결된 모든 포트는 **하나의 전기적 공유 매체**처럼 동작한다.

- 한 포트에서 들어온 **신호(전기 파형)**를 증폭/재생(repeat)해서 **모든 다른 포트로 그대로 내보낸다.**
- MAC 주소, IP 주소에 대한 **인식이 전혀 없다.**  
  → “이 프레임이 누구 거인지”에 전혀 관심이 없다.
- IEEE 802.3 표준에서는 허브(리피터)를 더 이상 권장하지 않으며,  
  **스위치로 대체할 것을 권고**한다.  

그래서 오늘날(2020년대 중반 이후) 허브는 **교재/시험용 또는 특별한 테스트 환경**을 제외하고는 사실상 **폐기된 기술**로 취급된다. 현업의 대부분 환경에서는 **스위치가 허브의 일을 완전히 대체**하고 있다.  

### 2.2 충돌 도메인, CSMA/CD, 반이중(half-duplex)

허브는 모든 포트가 하나의 **공유 매체**이기 때문에,  
예전의 **버스형 이더넷(동축 케이블)**과 사실상 동일한 특성을 가진다.  

- **충돌 도메인(collision domain)**:  
  한 번에 **오직 한 장치만 전송할 수 있는** 영역.  
  허브에 연결된 모든 장치는 **한 충돌 도메인**을 공유한다.
- 두 장치가 동시에 전송 시작 → 전기 신호가 섞여 **충돌(collision)** 발생.
- 10/100 Mbps 허브 환경에서는 **CSMA/CD**가 충돌 감지·재전송을 담당했다.  
  (CSMA/CD는 오늘날 **스위치 기반, 전이중 링크**에서는 사실상 사용되지 않는 유산이다.)

허브 환경에서 **실제 유효 대역폭**은 장치 수가 늘어날수록 급격히 떨어진다.

- 예: 10 Mbps 허브에 PC 10대가 고르게 트래픽을 발생시키면,  
  각 PC에 돌아가는 평균 처리량은 대략 $$1\ \text{Mbps}$$ 근처까지 떨어지고,  
  충돌 오버헤드까지 고려하면 **더 낮아진다.**

이를 간단히 수식으로 보면, 허브에서의 “평균 유효 처리량”을

$$
R_\text{eff} \approx R_\text{raw} \times (1 - p_\text{coll})
$$

으로 근사할 수 있다.

- \(R_\text{raw}\): 매체의 원시 속도(예: 10 Mbps)
- \(p_\text{coll}\): 충돌로 인해 재전송되는 비율

장치 수가 많아지면 \(p_\text{coll}\)이 증가하고, 결과적으로 \(R_\text{eff}\)가 감소한다.

### 2.3 예제 시나리오: 허브에서 패킷이 흐르는 모습

**상황**

- 4포트 허브에 PC A, B, C, D가 연결되어 있다.
- A가 B에게 프레임을 보낸다.

**프레임 흐름**

1. A의 NIC가 이더넷 프레임 전송 시작 (목적 MAC = B의 주소).
2. 허브는 A 포트에서 받은 **전기 신호**를 **모든 다른 포트(B, C, D)**로 뿌린다.
3. B, C, D 모두 그 신호를 받는다.
   - B의 NIC: 목적 MAC이 자기 것 → 프레임을 상위 계층으로 넘김.
   - C, D의 NIC: 목적 MAC이 자기 것이 아님 → **조용히 폐기**.

C나 D가 동시에 전송하려 한다면?

- C가 A에게, A가 B에게 동시에 전송 → 버스 상에서 파형이 섞이고 **충돌** 발생.
- 두 NIC는 충돌을 감지하고 랜덤 백오프 후 재전송.

### 2.4 간단 허브 시뮬레이션 코드 (Python)

허브의 동작을 아주 단순하게 코드로 모의해 보자.

```python
class Hub:
    def __init__(self):
        self.ports = []  # 각 포트에 연결된 장치 목록

    def connect(self, device):
        self.ports.append(device)
        device.hub = self

    def receive(self, frame, ingress_port):
        # ingress_port로부터 프레임을 받고, 나머지 모든 포트로 브로드캐스트
        for dev in self.ports:
            if dev is not ingress_port:
                dev.receive(frame)

class Device:
    def __init__(self, name, mac):
        self.name = name
        self.mac = mac
        self.hub = None

    def send(self, dst_mac, payload):
        frame = {"src": self.mac, "dst": dst_mac, "payload": payload}
        print(f"[{self.name}] 전송: {frame}")
        self.hub.receive(frame, self)

    def receive(self, frame):
        if frame["dst"] == self.mac:
            print(f"[{self.name}] 수신 (내 MAC): {frame}")
        else:
            print(f"[{self.name}] 수신 했지만 내 MAC이 아니라 버림")

# 예제
hub = Hub()
a = Device("A", "AA:AA:AA:AA:AA:AA")
b = Device("B", "BB:BB:BB:BB:BB:BB")
c = Device("C", "CC:CC:CC:CC:CC:CC")

for dev in [a, b, c]:
    hub.connect(dev)

a.send(b.mac, "Hello B!")
```

이 코드는 충돌/CSMA/CD는 구현하지 않았지만, **허브의 “모든 포트로 전송” 특성**을 보여준다.

---

## 3. Link-Layer Switch — 멀티포트 브리지, 현대 이더넷의 중심

### 3.1 스위치 = 멀티포트 브리지

**링크 계층 스위치(L2 switch)**는 **여러 포트를 가진 브리지(bridge)**라고 볼 수 있다.  
IEEE 802.1D(Bridges) 표준에서 정의한 브리지 기능이, 오늘날 대부분의 스위치 내부에 구현되어 있다.  

- **L2 장비**: 이더넷 프레임의 **MAC 헤더**를 보고 동작.
- 역할:
  - 서로 다른 LAN 세그먼트를 연결.
  - 각 세그먼트의 **충돌 도메인**을 분리.
  - 하나의 **브로드캐스트 도메인**을 확장 (라우터나 VLAN으로 나누기 전까지).

NVIDIA, UMass 등의 교육 자료에서도 **스위치는 “포트마다 브리지 하나씩 붙은 것과 같다”**고 설명한다.  

### 3.2 MAC 학습(Learning)과 포워딩(Forwarding)

스위치의 핵심 로직은 **“누가 어느 포트에 있는지” 학습**하는 것이다.

#### 3.2.1 MAC 주소 테이블

스위치는 내부에 **MAC 주소 테이블(MAC address table, forwarding table)**을 가진다.

- 각 항목: (MAC 주소, 포트 번호, 나이/타이머)
- 프레임 처리 흐름:
  1. 프레임이 **어떤 포트로 들어왔다**.
  2. 프레임의 **출발지 MAC(SA)**를 보고  
     → “이 MAC은 이 포트로 reachable”이라고 테이블에 저장/갱신 (learning).
  3. 목적지 MAC(DA)을 보고:
     - 테이블에 있으면 → 해당 포트로 **Unicast 포워딩**
     - 없으면 → **Flooding** (모든 포트로 전송, 들어온 포트 제외)
  4. 일정 시간이 지나면 쓰지 않는 항목은 **aging**되어 삭제.

이 동작은 IEEE 802.1D에 정의된 **learning bridge 알고리즘**에 해당한다.  

#### 3.2.2 MAC 학습 예

**상황**

- 스위치 S, 포트 1에 호스트 A(MAC A), 포트 2에 B(MAC B), 포트 3에 C(MAC C).
- 초기 MAC 테이블은 비어 있음.

1. A가 B에게 프레임 전송 (포트 1 → 스위치)
   - 스위치: SA = A → MAC[A] = 포트 1 로 기록.
   - DA = B인데 테이블에 없음 → 포트 2,3으로 Flooding.
   - B, C 모두 프레임 수신
     - B: 자기 MAC → 상위로 전달
     - C: 버림
2. B가 A에게 응답 (포트 2 → 스위치)
   - 스위치: SA = B → MAC[B] = 포트 2 로 기록.
   - DA = A → MAC 테이블에 A 포트 1로 기록되어 있으므로 포트 1로만 전송.

이 과정을 반복하면, 스위치는 네트워크 내 모든 MAC → 포트 매핑을 점차 학습한다.

### 3.3 충돌 도메인과 브로드캐스트 도메인

스위치 환경에서는 **각 포트가 독립적인 충돌 도메인**이다.

- CSMA/CD가 필요 없는 **전이중(full-duplex)** 링크가 기본.
- UMass/NSRC 자료에서도 “스위치는 충돌 도메인을 줄이고, 라우터는 브로드캐스트 도메인을 줄인다”고 정리한다.  

하지만 L2 스위치만 사용할 경우:

- **브로드캐스트 도메인**은 여전히 하나다.
- 예: ARP 요청(IPv4)이나 IPv6의 일부 멀티캐스트 기반 디스커버리는 **동일 VLAN/브리지 도메인의 모든 포트**로 전달된다.

> 즉, **스위치 = 충돌 도메인 분리**  
> **라우터 = 브로드캐스트 도메인 분리**라고 기억하면 좋다.

### 3.4 루프와 스패닝 트리(간단 언급)

스위치 여러 대를 **링 구조**로 연결하면 어떤 일이 일어날까?

- 브리지 도메인에 루프가 생기면, **브로드캐스트 프레임이 무한 순환** → 네트워크 마비.
- 이를 막기 위해 IEEE 802.1D는 **Spanning Tree Protocol(STP)**를 정의한다.  

STP의 아이디어:

- 여러 스위치 중 **root bridge**를 선정.
- 루프가 생기지 않는 **트리 구조**가 되도록 일부 포트를 **blocking 상태**로 둔다.
- 링크/스위치 장애 시, 다시 트리를 계산해 다른 포트를 forwarding으로 변경.

(자세한 STP 알고리즘은 VLAN 챕터에서 다루면 좋다.)

### 3.5 현대 이더넷 스위치의 속도와 스케일

오늘날(2024–2025년경) 상용 스위치는 다음과 같은 속도를 지원한다.  

- 액세스/엔터프라이즈: 포트당 1G, 2.5G, 5G, 10G
- 데이터센터: 포트당 25G, 40G, 50G, 100G, 200G, 400G, 800G
  - 예: 일부 벤더의 7050X/9000 시리즈는 1/10/25/40/100/400/800G 혼합 포트 지원
  - 48×25G + 6×100G, 또는 32×400G 같은 구성

즉, “스위치”는 더 이상 단순한 작은 SOHO 장비가 아니라,
**수백 Tbps급 백플레인**과 **레이어 3/라우팅·VLAN·VXLAN**까지 지원하는 거대 장비까지 포함하는 개념이 되었다.

### 3.6 Python으로 간단 MAC 학습 스위치 구현 예

허브 예제와 비교하기 위해, 아주 단순한 “학습 스위치”를 코드로 만들어 보자.

```python
class LearningSwitch:
    def __init__(self):
        self.ports = []  # 연결된 장치 리스트
        self.mac_table = {}  # MAC -> device

    def connect(self, device):
        self.ports.append(device)
        device.switch = self

    def receive(self, frame, ingress_dev):
        src = frame["src"]
        dst = frame["dst"]

        # 1) 학습
        self.mac_table[src] = ingress_dev

        # 2) 목적지 결정
        if dst in self.mac_table:
            # Unicast 포워딩
            out_dev = self.mac_table[dst]
            if out_dev is not ingress_dev:
                out_dev.receive(frame)
        else:
            # Flooding
            for dev in self.ports:
                if dev is not ingress_dev:
                    dev.receive(frame)

class Host:
    def __init__(self, name, mac):
        self.name = name
        self.mac = mac
        self.switch = None

    def send(self, dst_mac, payload):
        frame = {"src": self.mac, "dst": dst_mac, "payload": payload}
        print(f"[{self.name}] 전송: {frame}")
        self.switch.receive(frame, self)

    def receive(self, frame):
        if frame["dst"] == self.mac:
            print(f"[{self.name}] 수신: {frame}")
        else:
            # 스위치의 Flooding으로 인한 불필요 수신
            print(f"[{self.name}] 나에게 온 프레임이 아니므로 폐기")

# 예제
sw = LearningSwitch()
a = Host("A", "AA:AA:AA:AA:AA:AA")
b = Host("B", "BB:BB:BB:BB:BB:BB")
c = Host("C", "CC:CC:CC:CC:CC:CC")

for h in [a, b, c]:
    sw.connect(h)

# 첫 통신: Flooding 발생
a.send(b.mac, "첫 메시지")
print("MAC 테이블:", {k: v.name for k, v in sw.mac_table.items()})

# 두 번째 통신: Unicast만
b.send(a.mac, "두 번째 메시지")
```

이 코드를 돌려 보면:

- 첫 프레임에서 스위치는 A의 MAC을 학습하고 B를 찾지 못해 Flooding.
- B가 응답하면서 B MAC을 학습, 이후 통신에서는 Flooding 없이 **필요한 포트로만 전송**된다.

---

## 4. Routers — 네트워크 계층, 서브넷과 브로드캐스트 도메인 분리

### 4.1 라우터의 역할과 OSI 계층

**라우터(router)**는 **네트워크 계층(L3)** 장비로,  
**IP 헤더(IPv4/IPv6)**를 보고 **다른 네트워크/서브넷으로 패킷을 전달**한다.  

- 각 인터페이스에 **L2 주소(MAC)**뿐 아니라 **L3 주소(IP)**가 있음.
- 라우팅 테이블(Forwarding Information Base, FIB)을 기반으로 **최장 접두사 일치(longest prefix match)**로 다음 홉(next hop)을 결정.
- IP 패킷을 새 이더넷 프레임에 실어 **다음 홉의 MAC**으로 전송.

결정적으로:

- **각 라우터 인터페이스는 별도의 브로드캐스트 도메인**이다.
- 동일 서브넷(예: 192.168.1.0/24)에 속한 호스트만 서로의 L2 브로드캐스트(ARP 등)를 보게 된다.

### 4.2 라우팅 테이블과 최장 접두사 매칭

라우터는 “목적지 IP → 다음 홉/인터페이스”를 저장한 **라우팅 테이블**을 가진다.

간단한 예:

| Prefix | Next Hop / Interface |
|--------|----------------------|
| 10.0.0.0/8 | ISP1 |
| 192.168.1.0/24 | LAN1 |
| 192.168.2.0/24 | LAN2 |
| 0.0.0.0/0 | 기본 게이트웨이(Upstream) |

**최장 접두사 매칭** 규칙:

- 목적지 IP가 여러 prefix와 일치하면, **마스크가 가장 긴(=가장 구체적인)** prefix를 선택.

수식으로 표현하면, 라우팅 테이블 항목 집합 \(\{(p_i, l_i)\}\) (p_i: prefix, l_i: 길이)에 대해  
목적지 IP \(d\)에 대해

$$
\text{route}(d) = \arg\max_{i: d \in p_i} l_i
$$

를 선택하는 것과 같다.

### 4.3 IP 패킷 처리 과정(단순화)

스위치와 비교해, 라우터에서 한 패킷이 처리되는 과정을 정리하면:

1. 라우터의 한 인터페이스(예: eth0)로 **이더넷 프레임** 도착.
2. 프레임의 목적 MAC이 **라우터 인터페이스 MAC**인지 확인.
3. 프레임 디캡슐레이션 → **IP 패킷 추출**.
4. IP 헤더의 TTL 감소, 체크섬 재계산(IPv4).
5. 목적지 IP에 대해 라우팅 테이블 조회 → 다음 홉 및 출력 인터페이스 결정.
6. 출력 인터페이스의 L2 타입(이더넷이면 ARP/ND)으로 **다음 홉 MAC** 결정.
7. 새 이더넷 프레임을 만들어 전송.

이 과정에서:

- 라우터는 **L2 헤더를 버리고 새로 만든다** (L3 이상은 유지).
- 스위치는 **L2 헤더만 보고 재전송**(L3 이상은 건드리지 않음).

### 4.4 라우터와 브로드캐스트 도메인

라우터는 **브로드캐스트를 넘어가지 못하게 하는 장벽** 역할을 한다.  

예제:

```text
[ LAN1: 192.168.1.0/24 ]  ---  (R1)  --- [ LAN2: 192.168.2.0/24 ]

LAN1 스위치  --  R1.eth0 (192.168.1.1)
LAN2 스위치  --  R1.eth1 (192.168.2.1)
```

- LAN1에서 발생한 ARP 브로드캐스트(FF:FF:FF:FF:FF:FF)는 LAN1 내부로만 전달.
- R1은 ARP 브로드캐스트를 **다른 인터페이스로 전달하지 않는다.**

따라서:

- **브로드캐스트 도메인**: LAN1, LAN2 각각 별개.
- **충돌 도메인**: 각 스위치 포트별로 별개.

스위치로만 구성된 환경에서는 브로드캐스트 도메인이 너무 커져서  
ARP/브로드캐스트/멀티캐스트가 전체 네트워크를 떠다니는 문제가 생길 수 있다.  
라우터(또는 L3 스위치)가 이를 논리적으로 쪼개는 데 핵심 역할을 한다.

### 4.5 간단 라우팅 선택 코드 예제 (IPv4, Python)

아주 단순한 형태의 **최장 접두사 매칭** 구현 예시:

```python
import ipaddress

class SimpleRouter:
    def __init__(self):
        self.routes = []  # (network, next_hop) 튜플

    def add_route(self, prefix, next_hop):
        network = ipaddress.IPv4Network(prefix)
        self.routes.append((network, next_hop))

    def lookup(self, dst_ip):
        dst = ipaddress.IPv4Address(dst_ip)
        best = None
        best_prefixlen = -1

        for net, nh in self.routes:
            if dst in net and net.prefixlen > best_prefixlen:
                best = nh
                best_prefixlen = net.prefixlen

        return best

# 예제
router = SimpleRouter()
router.add_route("10.0.0.0/8", "ISP1")
router.add_route("192.168.1.0/24", "LAN1")
router.add_route("192.168.2.0/24", "LAN2")
router.add_route("0.0.0.0/0", "DEFAULT")

for dst in ["192.168.1.10", "192.168.2.50", "8.8.8.8"]:
    print(dst, "→", router.lookup(dst))
```

출력은 대략:

- 192.168.1.10 → LAN1
- 192.168.2.50 → LAN2
- 8.8.8.8 → DEFAULT

실제 라우터는 하드웨어 TCAM, 다양한 라우팅 프로토콜(OSPF, BGP 등)을 사용하지만,  
모두 이 “최장 접두사 매칭” 개념 위에 세워져 있다.

### 4.6 허브–스위치–라우터를 섞은 소규모 예제

마지막으로, 세 장비가 섞인 작은 네트워크를 생각해 보자.

```text
PC A ---+
        |  (옛날) 허브  ---  스위치 --- R1 --- 인터넷
PC B ---+
                 |        (여러 VLAN/LAN)
             서버들...
```

1. 허브 부분
   - A, B는 허브에 연결되어 **하나의 충돌 도메인**을 공유.
   - 오늘날에는 이 부분도 **스위치로 대체**하는 것이 일반적이다.
2. 스위치 부분
   - 스위치 각 포트는 개별 충돌 도메인.
   - 하지만 스위치 전체는 (VLAN을 나누지 않는 한) **하나의 브로드캐스트 도메인**.
3. 라우터(R1)
   - 스위치의 VLAN 또는 서브넷별로 인터페이스를 두고,  
     각 VLAN/서브넷을 **별도의 브로드캐스트 도메인**으로 분할.
   - 인터넷으로 나가는 **디폴트 게이트웨이** 역할.

이 구조에서:

- **성능 최적화**: 허브 대신 스위치를 사용해 충돌 최소화.
- **규모 확장/보안**: 라우터(L3)로 서브넷/VLAN을 나누어 브로드캐스트·보안·정책을 분리.

---

## 정리

- **허브(hub)**  
  - L1 리피터, 모든 포트가 하나의 충돌 도메인/브로드캐스트 도메인.  
  - CSMA/CD 기반, 반이중, 오늘날은 IEEE 802.3에서 사실상 **폐기/비권장**.  
- **링크 계층 스위치(L2 switch)**  
  - 멀티포트 브리지, MAC 학습·필터링으로 **필요한 포트로만 프레임 전달**.  
  - 충돌 도메인을 포트 단위로 분리, 브로드캐스트 도메인은 그대로 유지.  
  - 오늘날 1G~800G까지 다양한 속도의 스위치가 데이터센터·엔터프라이즈에서 쓰임.  
- **라우터(router)**  
  - L3 장비, IP 헤더를 보고 최장 접두사 매칭으로 패킷을 라우팅.  
  - **브로드캐스트 도메인 분리**의 핵심 장비. 각 인터페이스는 별도의 L2 도메인.  