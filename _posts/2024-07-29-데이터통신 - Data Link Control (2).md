---
layout: post
title: 데이터 통신 - Data Link Control (2)
date: 2024-07-29 22:20:23 +0900
category: DataCommunication
---
# 11.3 HDLC & 11.4 PPP — 데이터링크 제어의 대표 프로토콜

이전 11.1–11.2에서 **DLC 서비스(Framing, Flow/Error Control, Connectionless/Connection-oriented)** 와  
**Simple / Stop-and-Wait / Piggybacking** 같은 추상 모델을 봤다.

이번에는 이 개념들이 **실제 표준 프로토콜**에서 어떻게 구현되는지,  
대표적인 두 프로토콜 **HDLC(High-level Data Link Control)** 와 **PPP(Point-to-Point Protocol)** 를 통해 살펴본다.

---

## 11.3 HDLC

HDLC는 ISO(국제표준화기구)의 **비트 지향(bit-oriented) 데이터링크 계층** 규격으로,  
프레이밍과 ARQ/흐름제어의 “교과서적인 구현”이다. ISO/IEC 13239로 통합되어 있다.   

HDLC의 핵심:

- **구성(Configuration) & 전송 모드(Transfer Mode)**  
  → 어떤 스테이션(Primary/Secondary/Combined)이 어떤 역할로 통신하는가
- **프레이밍(Framing)**  
  → Flag, Address, Control, Information, FCS, Bit stuffing, I/S/U 프레임

### 11.3.1 Configurations and Transfer Modes

HDLC에서는 먼저 **스테이션의 종류**와 **링크 구성**을 정의하고, 그 위에 전송 모드(Transmission Mode)를 올린다.   

#### 1) 스테이션 타입

HDLC에서 한 “스테이션(station)”은 하나의 통신 노드(라우터, 단말 등)를 의미한다.

| 타입 | 역할 |
|------|------|
| Primary station | 링크를 “관리”하는 주체. 통신 시작/종료, 폴링(poll), 오류 복구 책임. |
| Secondary station | Primary의 명령/폴링에 응답하는 종속 노드. |
| Combined station | Primary와 Secondary 역할을 모두 수행하는 대등(peer) 노드. |

- Primary/Secondary 구조는 **마스터–슬레이브** 느낌이고,
- Combined는 **대칭적인 점대점(peer-to-peer)** 구조에 가깝다.   

#### 2) 링크 구성(Link Configurations)

HDLC는 크게 두 가지 구성을 정의한다.   

1. **Unbalanced configuration (불균형 구성)**  
   - 하나의 Primary + 하나 이상의 Secondary  
   - 주로 **멀티드롭(multi-drop) / 멀티포인트** 환경에 사용  
   - Primary가 모든 링크 제어 권한을 갖고, Secondary는 지시에만 응답

2. **Balanced configuration (균형 구성)**  
   - 두 개의 Combined station  
   - **각 스테이션이 Primary이자 Secondary**  
   - 점대점(point-to-point) 링크에 적합

ASCII로 표현하면:

```text
[Unbalanced: Multi-drop]

   Primary
      |
   ---+-----------------------------
      |            |            |
 Secondary 1   Secondary 2   Secondary 3

[Balanced: Point-to-Point]

   Combined A <----------------> Combined B
      (둘 다 Primary + Secondary)
```

---

#### 3) 전송 모드(Transfer Modes)

링크 구성 위에서 HDLC는 세 가지 전송 모드를 정의한다.   

| 모드 | 구성 | 특징 |
|------|------|------|
| NRM (Normal Response Mode) | Unbalanced | Primary만 먼저 전송 가능. Secondary는 Primary의 폴링/명령에 응답할 때만 전송. |
| ARM (Asynchronous Response Mode) | Unbalanced | 여전히 Primary/Secondary 구조이지만, Secondary도 임의 시점에 전송 가능. |
| ABM (Asynchronous Balanced Mode) | Balanced | 두 Combined station 모두 언제든지 전송/제어 가능. 현대 HDLC에서 가장 많이 쓰이는 모드. |

##### (1) NRM — Normal Response Mode

- **전형적인 멀티드롭 환경**을 위한 모드.
- 흐름:
  1. Primary가 Secondary들에게 **폴링(POLL)** 을 보냄.
  2. 해당 Secondary만 응답(데이터 전송 또는 상태 보고).
  3. Primary가 다음 Secondary를 폴링.

타임라인 예:

```text
시간 →
P: POLL(S1?) -------------------------------------------------->
S1: <------------------------ DATA(S1 → P)
P: POLL(S2?) -------------------------------------------------->
S2: <------------------------ DATA(S2 → P)
...
```

장점:

- **충돌이 없다**. Secondary들은 Primary가 지정한 순서대로만 말한다.
- **Half-duplex 링크**에서도 운용 가능 (한 번에 한 방향만 사용).

단점:

- Primary가 전송 스케줄을 잡느라 복잡해질 수 있다.
- Secondary가 급하게 보낼 데이터가 있어도 **폴링을 기다려야** 한다.

---

##### (2) ARM — Asynchronous Response Mode

- 여전히 Primary/Secondary 구성이지만, Secondary가 **언제든지 응답 없이 전송 가능**.   
- 다만 여전히 Primary가
  - 링크 초기화,
  - 오류 복구,
  - 논리적 연결 해제
  를 담당한다.

실전에서는 ABM이 많이 쓰이고, ARM은 상대적으로 드물다.

---

##### (3) ABM — Asynchronous Balanced Mode

- **Balanced configuration** 에서 사용하는 모드.
- 두 스테이션 모두 Combined → Primary/Secondary 구분이 없다.   
- 양쪽 모두:
  - 링크 초기화 가능,
  - 데이터 전송 시작 가능,
  - 오류 복구 요청 가능.

현대의 HDLC 파생 프로토콜(예: LAPB, LAPD, Frame Relay LAPF, PPP HDLC-like framing 등)은 **ABM 스타일**을 기본으로 한다.   

---

#### 4) 예제 상황: 옛날 멀티드롭 vs 현대 라우터 간 링크

1) **과거 멀티드롭 X.25/전용선** (NRM)

- 본사 메인프레임이 Primary,
- 지사 단말기들이 Secondary.
- 본사 장비가 지사 단말기를 폴링하며, 순차적으로 데이터 송수신.

2) **현대 라우터–라우터 직결 링크** (ABM)

- 두 라우터가 전용선/SONET 상에서 HDLC 기반 링크 운용.
- 각 라우터가 언제든지 데이터를 전송할 수 있고, 상대의 오류에 대해 프레임을 보내 복구를 요청.

---

### 11.3.2 Framing (HDLC 프레임 구조)

HDLC는 **비트 지향 프레이밍**의 대표적인 예다.   

기본 프레임 구조:

| 필드 | 설명 |
|------|------|
| Flag | 프레임 시작/끝 표시 (`01111110`, 0x7E) |
| Address | 목적/출발지 주소 (보통 Secondary 주소) |
| Control | 프레임 타입(I/S/U), 시퀀스 번호, P/F 비트 등 |
| Information | 상위 계층 데이터 (없을 수도 있음) |
| FCS | Frame Check Sequence (CRC-16 또는 CRC-32) |
| Flag | 종료 플래그 (다음 프레임의 시작 플래그와 겸용 가능) |

  

ASCII 형식으로:

```text
|  Flag  | Address |   Control   |    Information    |   FCS   |  Flag  |
| 0x7E   | 1+ byte | 1 or 2 byte | 0 ~ N bytes       | 2 or 4B | 0x7E   |
```

---

#### 1) 프레임 타입: I / S / U 프레임

HDLC의 Control 필드 구조는 프레임 타입에 따라 다르다.   

| 종류 | 이름 | 용도 |
|------|------|------|
| I-frame | Information frame | 사용자 데이터 + piggybacked ACK |
| S-frame | Supervisory frame | 순수 흐름/오류 제어 (RR, RNR, REJ, SREJ 등) |
| U-frame | Unnumbered frame | 링크 관리, 모드 설정(SABM, DISC, UA, DM, FRMR…), 일부 데이터 전송(UI) |

##### (1) I-frame (Information)

Control 필드(3비트 번호 사용 시):

```text
bit: 7 6 5  4   3  2 1 0
     N(R)   P/F  N(S) 0
```

- `N(S)`: 송신 시퀀스 번호
- `N(R)`: 다음에 기대하는 수신 시퀀스 번호 (ACK 의미)
- `P/F`: Poll/Final 비트 (명령/응답 의미, 폴링 또는 응답 완료 표시)

즉, I-frame 하나에

- **데이터 전송 + ACK 정보(piggyback)** 를 동시에 얹는다.

##### (2) S-frame (Supervisory)

흔한 S-frame 종류:

- RR (Receive Ready): “N(R)까지 잘 받았고, 더 받을 준비되어 있다”
- RNR (Receive Not Ready): “N(R)까지 잘 받았지만, 지금은 더 받을 수 없다(버퍼 부족 등)”
- REJ (Reject): “N(R) 이후부터 다시 보내라 (Go-Back-N)”
- SREJ (Selective Reject): “N(R)만 다시 보내라 (Selective Repeat)”

Control 필드 형태 (3비트 번호):

```text
bit: 7 6 5 4  3  2 1 0
     N(R) P/F  0  S1 S0 1
```

여기서 (S1, S0) 조합으로 RR/RNR/REJ/SREJ 구분.   

##### (3) U-frame (Unnumbered)

링크 관리와 기타 기능 제공:

- SABM/SABME: Asynchronous Balanced Mode 설정
- DISC: 연결 해제 요청
- UA: Unnumbered Acknowledge
- DM: Disconnected Mode
- FRMR: Frame Reject (심각한 오류 보고)
- UI: Unnumbered Information (재전송/번호 없는 단순 데이터)

Control 필드 비트 패턴은 다양하며, 모드 설정/에러 보고 등에 사용된다.   

---

#### 2) HDLC 비트 스터핑(Bit Stuffing)과 투명성

HDLC의 Flag는 항상 **`01111110` (0x7E)** 이다.  
데이터 내부에 같은 패턴이 나오면 프레임 경계가 깨지므로, **비트 스터핑(bit stuffing)** 을 사용한다.   

**송신 규칙(비트 스터핑):**

- 데이터 비트를 연속으로 전송하면서,
- “1이 5개 연속” 나타나면, 그 뒤에 **0을 강제로 삽입**한다.
  - 예: `0111110` → 실제 전송은 `0111110 0` (0 하나 더)
- 플래그 `01111110` 의 “6개의 연속 1 + 0” 패턴이 데이터 내부에 자연스럽게 생성되는 것을 막는다.

**수신 규칙(비트 언스터핑):**

- 수신 측은 비트를 읽다가 “1이 5개 연속 + 뒤에 0”이 오면,
  - 그 0을 **삭제**하고 데이터로 복원한다.
- “1이 6개 연속 + 뒤에 0”이면,
  - 이는 **플래그(프레임 경계)** 로 해석한다.

간단 예시:

원래 데이터 비트:

```text
... 0111110 1 010 ...
       ↑
여기서 1이 다섯 개 연속
```

송신 측 비트 스터핑 후:

```text
... 0111110 0 1 010 ...
           ↑
이 0은 스터핑된 0
```

수신 측은 1이 5개 연속 후 등장한 0을 제거하고 원래 데이터로 복원한다.

---

#### 3) FCS(Frame Check Sequence) — CRC

FCS는 Address, Control, Information 필드를 대상으로 계산되는 **CRC-16 또는 CRC-32** 값이다.   

- 전형적인 HDLC 변형들은
  - 16비트 CRC-CCITT(“CRC-16”) 또는
  - 32비트 CRC-32
  를 사용한다.
- FCS는 비트 스터핑이나 플래그, 시작/정지비트 등 **프레이밍용 비트는 포함하지 않는다**.   

오류 검출 확률:

- 특정 길이 이하 프레임에 대해서는 **대부분의 단일/이중/버스트 오류를 검출**할 수 있도록 설계.
- 아주 긴 프레임에서는 이론적으로 **검출 실패 확률**이 증가하지만, 실용적으로는 충분히 낮다.

---

#### 4) 예제: HDLC I-frame 분석

가정:

- 점대점 ABM 모드, 3비트 시퀀스 번호 사용.
- 송신자 A가 수신자 B에게 IP 패킷을 싣고 있는 I-frame 하나를 전송.
- 단순화를 위해 Field 길이는 예시용으로 줄인다.

16진수 형태의 프레임(예):

```text
7E 01 00 45 00 00 54 ... XX XX 7E
```

필드별 해석(가상의 예):

| 필드 | 값 | 의미 |
|------|----|------|
| Flag | `7E` | 프레임 시작 |
| Address | `01` | Secondary 주소 1 (점대점이면 의미 없을 수도 있음) |
| Control | `00` | I-frame, N(S)=0, N(R)=0, P/F=0 (예시) |
| Information | `45 00 00 54 ...` | IPv4 헤더(버전 4, IHL 5, 총 길이 0x0054 등) |
| FCS | `XX XX` | CRC-16 값 (계산 결과) |
| Flag | `7E` | 프레임 끝(다음 프레임 시작과 겸용 가능) |

실제로는 Control 필드에 N(S)/N(R)/P/F 비트가 들어가고,  
FCS는 전체 Address+Control+Information에 대한 CRC-16/32로 계산된다.

---

#### 5) 간단 코드 예: HDLC 비트 스터핑/언스터핑 (개념용)

아래는 **비트 문자열**을 입력으로 받아 HDLC 스타일의 bit stuffing/unstuffing을 수행하는 파이썬 예제다.  
(실제 구현에서는 바이트 배열·비트연산을 사용하지만, 개념 이해를 위해 문자열로 단순화했다.)

```python
def hdlc_bit_stuff(bits: str) -> str:
    """HDLC 비트 스터핑: 1이 5개 연속 나오면 0을 삽입."""
    count = 0
    out = []
    for b in bits:
        out.append(b)
        if b == '1':
            count += 1
            if count == 5:
                out.append('0')  # 스터핑
                count = 0
        else:
            count = 0
    return ''.join(out)

def hdlc_bit_unstuff(bits: str) -> str:
    """HDLC 비트 언스터핑: 1이 5개 연속 + 뒤에 0이면 그 0을 제거."""
    count = 0
    out = []
    i = 0
    while i < len(bits):
        b = bits[i]
        out.append(b)
        if b == '1':
            count += 1
            if count == 5:
                # 다음 비트가 0이면 스터핑된 것 → 건너뜀
                i += 1
                if i < len(bits) and bits[i] != '0':
                    # 프로토콜 오류 상황이지만, 여기서는 단순히 진행
                    out.append(bits[i])
                count = 0
        else:
            count = 0
        i += 1
    return ''.join(out)

# 예시
data = "011111010111110"
stuffed = hdlc_bit_stuff(data)
restored = hdlc_bit_unstuff(stuffed)
print("원본   :", data)
print("스터핑:", stuffed)
print("복원   :", restored)
```

---

## 11.4 PPP (Point-to-Point Protocol)

PPP는 IETF가 정의한 **포인트-투-포인트 링크용 데이터링크 계층 프로토콜**이다.  
기본 설계는 1990년대에 나왔지만, **오늘날에도** 다음 환경에서 널리 쓰인다.   

- 옛날: 다이얼업 모뎀 연결(전화선)
- 현재:
  - xDSL에서 PPPoE/PPPoA 형태로 많이 사용
  - 전용선/직렬 회선 상의 라우터–라우터 연결
  - L2TP/IPsec, SSH, SSL VPN 터널 내부의 **가상 포인트-투-포인트 링크** (예: `ppp0` 인터페이스)   

PPP의 구성 요소(표준 정의):

1. **Encapsulation**: 여러 네트워크 계층 프로토콜(IPv4, IPv6, IPX, AppleTalk 등)을 캡슐화하는 방법   
2. **LCP (Link Control Protocol)**: 링크 설정/구성/테스트를 위한 제어 프로토콜
3. **NCP (Network Control Protocols)**: 각 네트워크 계층 프로토콜(IP, IPv6, IPX 등)을 설정하는 프로토콜 집합

  

---

### 11.4.1 PPP Services

PPP가 제공하는 서비스는 크게 다음과 같이 정리할 수 있다.   

1. **다중 프로토콜 캡슐화 (Multiprotocol Encapsulation)**
2. **링크 제어(LCP)와 자동 설정**
3. **인증(Authentication)**
4. **에러 검출 및 품질 모니터링(Quality monitoring)**
5. **선택적 압축/암호화/멀티링크(Multilink)**

#### 1) 다중 프로토콜 캡슐화

PPP는 하나의 포인트투포인트 링크 위에서 **여러 네트워크 계층 프로토콜을 동시에 운용**하도록 설계되었다.   

- 예:
  - IPv4 → IPCP (IP Control Protocol)
  - IPv6 → IPv6CP
  - IPX → IPXCP
  - AppleTalk → ATCP

각 네트워크 계층 프로토콜마다 **별도의 NCP**를 두고,  
PPP 프레임 헤더의 **Protocol 필드**로 어떤 프로토콜의 데이터인지 구분한다.   

---

#### 2) LCP (Link Control Protocol) — 링크 자동 설정

LCP는 PPP 링크가 **동작 가능한 상태가 되도록 협상**하는 프로토콜이다.   

협상 항목 예:

- 최대 수신 단위 (MRU, Maximum Receive Unit)
- ACCM(Async Control Character Map, 비동기 전송 시 어떤 문자들을 이스케이프할지)
- Magic Number(루프 검출용 값)
- 압축 옵션, 인증 방식(PAP/CHAP/EAP) 등

LCP는 다음과 같은 메시지를 사용한다.

- Configure-Request (내 옵션 제안)
- Configure-Ack (상대 제안 수락)
- Configure-Nak/Reject (상대 제안 일부 수정/거부)
- Terminate-Request/Ack
- Code-Reject, Protocol-Reject 등

실제 구현에서는 LCP가 PPP의 **상태 머신**을 구동하면서,

- 링크 “Establish” 단계에서 옵션을 협상하고,
- 인증/네트워크 계층 설정이 끝난 뒤에도 링크 품질을 모니터링한다.

---

#### 3) 인증(Authentication)

PPP는 링크 계층 수준에서 다음과 같은 인증 메커니즘을 제공한다.   

- PAP (Password Authentication Protocol)
- CHAP (Challenge-Handshake Authentication Protocol)
- EAP (Extensible Authentication Protocol, 확장가능한 인증 프레임워크)

대표적인 시나리오: “가정용 DSL 모뎀”

- PPPoE 세션이 올라오면,
- ISP 액세스 서버가 PPP LCP를 통해 링크 구성,
- 이어서 CHAP를 통해 가입자 인증,
- 성공하면 IPCP를 통해 IP 주소 할당.

---

#### 4) 에러 검출과 품질 모니터링

PPP는 HDLC와 비슷한 방식으로 **FCS(CRC 기반)** 필드를 사용해 프레임 단위 에러를 검출한다.   

- FCS는 Address, Control, Protocol, Information(+Padding) 필드를 대상으로 계산.
- FCS에 포함되지 않는 비트(Flag, Stuffing bits)는 링크 투명성만을 위한 부분이다.

추가로 LCP 옵션(예: Magic Number, Link-Quality Monitoring)을 사용해,

- 루프백(link loop)을 검출하거나,
- 과도한 에러가 발생하면 링크를 자동 종료하는 등의 정책을 구현할 수 있다.   

---

#### 5) 선택적 기능: 압축, 암호화, Multilink

PPP는 base 프로토콜 외에도 다양한 선택적 기능을 포함하는 확장 RFC들이 존재한다.   

- 헤더/데이터 압축 (Deflate, BSD compress 등)
- 암호화(예전 MPPE 등, 오늘날엔 주로 상위 계층에서 처리)
- Multilink PPP (MLPPP, 여러 링크를 묶어 하나의 논리 링크처럼 사용)

이들 기능은 대부분 **LCP/NCP 옵션**으로 협상된다.

---

### 11.4.2 PPP Framing

PPP 프레임은 **HDLC와 매우 비슷하지만, 단순화된 “HDLC-like framing”** 을 사용한다.   

기본 포맷(비트/바이트 단위):

```text
Flag | Address | Control | Protocol | Information | FCS | Flag
 1B  |   1B    |   1B    |   2B     | 0~N bytes   |2 or 4B| 1B
```

| 필드 | 값(기본) | 설명 |
|------|----------|------|
| Flag | `0x7E` | HDLC와 같은 프레임 시작/끝 플래그 |
| Address | `0xFF` | “브로드캐스트 주소” 의미, PPP에서는 거의 고정 |
| Control | `0x03` | Unnumbered Information(UI) 프레임 의미, 고정 |
| Protocol | ex) `0x0021`, `0x0057` | 상위 프로토콜 (IPv4, IPv6 등) 식별자 |
| Information | 가변 | 상위 계층 패킷(IP 헤더 포함) |
| FCS | 2 or 4B | CRC-16 혹은 CRC-32 |
| Flag | `0x7E` | 종료 플래그 |

  

PPP의 특징:

- HDLC와 달리 **Address, Control 필드를 논리적으로 사용하지 않고 고정값**으로 둔다.
- 네트워크 계층 프로토콜 선택은 모두 **Protocol 필드**에 맡긴다.
- 따라서 PPP는 사실상 **“단일 UI 프레임 타입”** 만 사용하는 HDLC의 특수 케이스라고 볼 수 있다.

---

#### 1) Protocol 필드 예시

일부 대표적인 PPP Protocol 값:   

| Protocol 값 (16진) | 의미 |
|---------------------|------|
| `0x0021` | IPv4 (IPCP로 설정) |
| `0x0057` | IPv6 (IPv6CP로 설정) |
| `0xC021` | LCP (Link Control Protocol) |
| `0x8021` | IPCP (IP Control Protocol, IPv4용 NCP) |
| `0x8057` | IPv6CP (IPv6용 NCP) |
| `0x8029` | AppleTalk Control Protocol |
| `0x802B` | IPXCP (IPX Control Protocol) |

---

#### 2) 투명성(Transparency): Byte Stuffing & ACCM

PPP는 **비트 지향이라기보다는 “옥텟(바이트) 지향”** 이다.  
비동기 RS-232 같은 환경에서 다음 2가지 문제를 해결해야 한다.   

1. 데이터 내에 `0x7E`(Flag)나 `0x7D`(Escape)가 등장
2. 제어 문자인 XON/XOFF(`0x11`, `0x13`) 등이 흐름제어에 쓰이므로, 데이터에서는 피해야 함

해결책:

- **Control Escape 옥텟(0x7D)** 와 **Async-Control-Character-Map(ACCM)** 을 사용해 **Byte stuffing** 수행.

규칙(간단화):

- 프레임을 전송할 때, 각 바이트가
  - Flag(0x7E), Escape(0x7D), ACCM에 지정된 제어 바이트면:
    - 먼저 `0x7D`를 보내고,
    - 다음 바이트는 원래 바이트의 5번째 비트(0x20)를 XOR한 값으로 보낸다.
    - 예: `0x7E` → `0x7D 0x5E`
- 수신 측은 `0x7D`를 만나면 다음 바이트를 읽고 다시 비트 복원(0x20 XOR)하여 원래 바이트로 되돌린다.

이는 HDLC의 비트 스터핑과 같은 역할을 **바이트 단위로 수행**하는 것이다.

---

#### 3) PPP 프레임 예제

가상의 IPv4 패킷(헤더 20바이트 + 데이터 32바이트)을 PPP로 전송한다고 하자.  
비동기 링크, 16비트 FCS 사용, Byte stuffing 결과는 단순화한다.

프레임(일부만, 16진):

```text
7E FF 03 00 21 45 00 00 34 ... XX XX 7E
```

해석:

- `7E` : Flag
- `FF` : Address (기본)
- `03` : Control (UI)
- `00 21` : Protocol = IPv4
- `45 00 00 34 ...` : IPv4 헤더 + 데이터 (길이 0x0034 = 52바이트 가정)
- `XX XX` : CRC-16
- `7E` : 종료 Flag

만약 정보필드에 `0x7E` 가 포함되어 있다면, 실제 전송에서는 그 부분이 `0x7D 0x5E`로 바뀐다.

---

#### 4) 효율 계산 간단 예

PPP 프레임의 오버헤드는 대략:

- 헤더: Flag(1) + Address(1) + Control(1) + Protocol(2) = 5바이트
- 트레일러: FCS(2 또는 4) + Flag(1) = 3~5바이트

따라서, 페이로드 길이를 $$L$$ 바이트라 할 때 **오버헤드 비율**은

$$
\text{overhead} = \frac{H + T}{H + T + L}
$$

여기서

- $$H = 5$$ (헤더),
- $$T = 4$$ (FCS 2바이트 + Flag 1바이트, 시작 플래그는 이전 프레임과 공유한다고 가정)라 하면,

예를 들어, $$L = 1500$$ 바이트(IP 패킷)일 때:

$$
\text{overhead}
= \frac{5 + 4}{5 + 4 + 1500}
\approx \frac{9}{1509}
\approx 0.6\%
$$

실제 환경에서는 Byte stuffing이 추가 오버헤드를 만들 수 있지만, 정상 트래픽에서는 보통 낮은 편이다.

---

### 11.4.3 PPP Transition Phases (PPP 단계/상태)

PPP 링크는 **여러 단계(Phase)를 지나는 상태 머신**으로 모델링된다.   

대표적인 단계:

1. Link Dead
2. Link Establishment (LCP)
3. Authentication (선택적)
4. Network-Layer Protocol (NCP)
5. Link Termination

이를 텍스트 상태도로 그려 보면:

```text
   [Link Dead]
        |
        | 물리계층 up (Carrier On)
        v
[Link Establishment] --(LCP 협상 실패)--> [Link Termination] --> [Link Dead]
        |
        | 인증 사용시
        v
  [Authentication] --(실패)--> [Link Termination] --> [Link Dead]
        |
        | 성공 또는 인증 생략
        v
[Network-Layer Protocols (NCP)] <--> (데이터 전송)
        |
        v
 [Link Termination] --> [Link Dead]
```

---

#### 1) Link Dead

- 물리 계층이 아직 준비되지 않았거나,  
  이전 세션이 종료되어 **링크가 비활성화된 상태**.
- 외부 이벤트(모뎀 carrier on, 인터페이스 up 설정 등)로부터 **Link Establishment** 단계로 진입.   

---

#### 2) Link Establishment Phase (LCP)

- LCP가 **Configure-Request/ Ack/Nak/Reject** 메시지를 교환하며:
  - MRU, ACCM, Magic Number, 인증 옵션 등 협상.
- 협상 성공 시:
  - 인증을 요구한다면 **Authentication** 단계로.
  - 아니면 즉시 **NCP** 단계로.   

예: DSL PPPoE 세션(단순화):

1. 물리 계층: xDSL 동기화 → PPPoE Discovery 완료 → PPP 세션 ID 할당
2. PPP: LCP Configure-Request/ACK 교환
3. LCP Opened 상태로 진입

---

#### 3) Authentication Phase (선택적)

- LCP에서 “인증 필요”로 설정된 경우,
  - PAP, CHAP, EAP 등으로 상대를 인증한다.   
- 인증 실패:
  - Link Termination으로 이동.
- 인증 성공:
  - Network-Layer Protocol Phase (NCP)로 이동.

시나리오 예(간단 PAP):

1. Network Access Server(NAS)가 PAP Request(Username/Password) 전송.
2. 클라이언트가 PAP Response(자격 증명) 전송.
3. NAS가 인증 서버(RADIUS 등)와 연동해 결과 확인 후 PAP Ack/NAK.

---

#### 4) Network-Layer Protocol Phase (NCP)

- 각 네트워크 계층 프로토콜(IP, IPv6 등)에 대해 **NCP 메시지** 교환.   
- 예: IPCP(IPv4용 NCP)에서 하는 일:
  - IP 주소 할당(예: Peer에게 IP 주소 또는 네트워크 정보 제공)
  - DNS 서버 주소 전달 등
- IPv6CP(IPv6용 NCP)는 링크 로컬 주소 기반으로 인터페이스 ID 설정 등.

이 단계에서는:

- PPP 프레임 내에서 LCP, NCP, 실제 네트워크 계층 데이터(IP 패킷)가 **혼재**해서 흐를 수 있다.   

---

#### 5) Link Termination Phase

- 다음과 같은 경우에 발생:   
  - 사용자가 접속 종료
  - 과도한 에러 발생, 품질 기준 미달
  - 인증 실패
  - 관리자가 다운

- Terminate-Request / Terminate-Ack 교환 후,
  - 물리 계층을 내려 Link Dead로 돌아감.

---

#### 6) 예제: 가정용 PPPoE 세션의 실제 흐름 (개념)

단순화된 타임라인:

```text
[1] xDSL 링크 up                → [물리 계층 ready]
[2] PPPoE Discovery              → PPPoE 세션 ID 확보
[3] PPP LCP 협상                 → Link Establishment Phase
[4] CHAP 인증                    → Authentication Phase
[5] IPCP/IPv6CP                  → Network-Layer Protocol Phase
[6] IP 데이터 전송                → 일반 통신
[7] 사용자 로그아웃/모뎀 off       → Link Termination → Link Dead
```

여기서 3~7이 PPP 관점의 Transition Phases에 해당한다.

---

### 11.4.4 PPP Multiplexing

PPP의 “Multiplexing”은 크게 두 가지 차원에서 이해할 수 있다.

1. **하나의 PPP 링크 위에 여러 네트워크 계층 프로토콜 동시 운용 (Protocol multiplexing)**
2. **여러 물리 링크를 하나로 묶는 Multilink PPP (Link aggregation)**

#### 1) Protocol Multiplexing (NCP 기반)

PPP는 단일 포인트-투-포인트 링크를 만들고, 그 위에 **여러 네트워크 계층 프로토콜을 동시에 얹는다.**   

핵심 포인트:

- 각 네트워크 계층 프로토콜마다 **별도의 NCP 인스턴스** 사용.
  - IP → IPCP
  - IPv6 → IPv6CP
  - IPX → IPXCP
  - AppleTalk → ATCP
- 프레임 헤더의 **Protocol 필드**로 어떤 프로토콜인지 구분.
- 링크가 Open된 상태에서,
  - LCP 패킷, NCP 패킷, 네트워크 계층 데이터 패킷이 **동시에 interleave**되어 전송 가능.

예: IPv4/IPv6 듀얼스택 PPP 링크

- 하나의 PPP 세션에서
  - IPCP로 IPv4 주소 협상
  - IPv6CP로 IPv6 링크-로컬/프리픽스 정보 협상
- 이후 같은 PPP 링크 위에서
  - Protocol=0x0021 (IPv4) 패킷과
  - Protocol=0x0057 (IPv6) 패킷이 번갈아 전송.

---

#### 2) Multilink PPP (MLPPP)

Multilink PPP(MLPPP)는 **여러 물리 링크(예: 두 개의 56kbps 모뎀, 여러 T1/E1 회선)를 하나의 논리 PPP 링크처럼 사용하는** 링크 집합(link aggregation) 기술이다.   

특징:

- 상위 계층 입장에서는 **하나의 “대역폭이 더 큰” PPP 링크로 보인다.**
- 실제로는 여러 PPP 링크가 있으며, 데이터가 **조각(fragment)** 으로 쪼개져 각 링크로 분산 전송.
- 수신 측은 조각에 붙은 시퀀스 번호를 보고 **원래 순서로 재조립**한다.

대표적인 이점:

- 두 개의 2Mbps 회선을 MLPPP로 묶으면,
  - 단일 4Mbps PPP 링크처럼 사용할 수 있다(오버헤드 제외).

단, 한 링크가 끊어지거나 지연이 달라질 수 있으므로,

- 조각 순서 관리를 위한 **부가적인 헤더/시퀀스 번호**가 필요하다.

---

#### 3) Multiclass PPP

Multiclass PPP는 **여러 “트래픽 클래스”마다 별도의 시퀀스 공간과 재조립 버퍼**를 두는 확장이다.   

아이디어:

- 실시간 트래픽(음성, 비디오)과 비실시간 트래픽(파일 다운로드)을 서로 다른 클래스에 넣고,
- 각 클래스별로 **독립적인 순서 보장과 재조립**을 수행한다.

이를 통해, 큰 파일 조각들이 지연되더라도  
음성 패킷 조각은 그 영향을 덜 받도록 할 수 있다.

---

### 11.4.5 PPP 예제 코드: PPP 헤더 파싱 (Python)

마지막으로, PPP 프레임의 **헤더 부분을 파싱**하는 간단 파이썬 예제를 보자.  
(실전에서는 pcap 라이브러리 등을 사용하지만, 여기서는 순수 파이썬으로 헤더만 해석한다.)

```python
import struct

# PPP 헤더: Flag(1) + Address(1) + Control(1) + Protocol(2)
# 여기서는 이미 Flag, FCS가 제거된 상태라고 가정하고, Address~Protocol만 파싱한다.

PPP_HEADER_FMT = "!BBBH"  # Address(1), Control(1), Protocol(2) - Network(big-endian)

PPP_PROTOCOLS = {
    0x0021: "IPv4",
    0x0057: "IPv6",
    0xC021: "LCP",
    0x8021: "IPCP",
    0x8057: "IPv6CP",
    0x8029: "ATCP",
    0x802B: "IPXCP",
}

def parse_ppp_frame(payload: bytes):
    """
    payload: Flag와 FCS를 제거한 PPP 프레임 바디 (Address부터 시작한다고 가정).
    """
    if len(payload) < struct.calcsize(PPP_HEADER_FMT):
        raise ValueError("프레임이 너무 짧습니다.")

    address, control, protocol = struct.unpack(PPP_HEADER_FMT, payload[:5])
    info = payload[5:]

    proto_name = PPP_PROTOCOLS.get(protocol, f"Unknown(0x{protocol:04X})")

    return {
        "address": address,
        "control": control,
        "protocol": protocol,
        "protocol_name": proto_name,
        "information": info,
    }

# 사용 예시
# 예: Flag와 FCS를 제외한 PPP IPv4 프레임 (간단 예시, 실제 데이터와 다를 수 있음)
example = bytes.fromhex("FF 03 00 21 45 00 00 34 00 00 40 00 40 06 00 00")

result = parse_ppp_frame(example)
print("Address:", hex(result["address"]))
print("Control:", hex(result["control"]))
print("Protocol:", result["protocol_name"])
print("Information (앞부분):", result["information"][:8].hex())
```

이 코드는 PPP 프레임의 Address/Control/Protocol 필드를 파싱하고,  
Protocol 값에 따라 IPv4/IPv6/LCP/NCP 등으로 구분하는 최소한의 구조를 보여준다.

---

## 마무리 정리

- **HDLC**
  - ISO 표준 비트 지향 데이터링크 프로토콜로,  
    Unbalanced/Balanced 구성과 NRM/ARM/ABM 모드를 정의한다.   
  - 프레임 구조: Flag–Address–Control–Information–FCS–Flag  
    + 비트 스터핑을 통한 투명성, I/S/U 프레임을 통한 데이터/제어 분리.
  - 많은 WAN/전용선/텔코 프로토콜(LAPB, LAPD, LAPF 등)의 기반이 되었고,  
    PPP의 프레이밍도 HDLC에서 파생되었다.

- **PPP**
  - IETF가 정의한 포인트투포인트 링크용 프로토콜로,  
    HDLC-like framing 위에서 LCP/NCP를 통해 링크 및 네트워크 계층을 자동 설정한다.   
  - 서비스:
    - Multiprotocol Encapsulation (IPv4/IPv6/IPX/AppleTalk …)
    - LCP를 통한 링크 설정/테스트, 인증(PAP/CHAP/EAP), FCS에 의한 에러 검출,  
      선택적 압축/암호화/Multilink.
  - 프레이밍:
    - Flag–Address(FF)–Control(03)–Protocol–Information–FCS–Flag  
      + Byte stuffing(0x7D/ACCM) 기반 투명성.
  - Transition Phases:
    - Link Dead → LCP(Link Establishment) → Authentication → NCP(Network) → Termination.
  - Multiplexing:
    - 하나의 PPP 링크 위에 여러 네트워크 계층 프로토콜을 동시에 운용(NCP 기반),
    - 여러 물리 링크를 하나의 논리 PPP로 묶는 Multilink PPP.

이전 절에서 배운 추상적인 DLC 개념(프레이밍, 흐름/오류 제어, 연결형/비연결형 서비스, Stop-and-Wait, Piggybacking 등)이  
HDLC와 PPP에서 **구체적인 필드/모드/상태머신**으로 어떻게 구현되는지를 연결해서 보면,  
다른 데이터링크·전송 계층 프로토콜(TCP, Frame Relay, 802.11, LTE/5G 링크 등)을 읽을 때도 구조를 훨씬 쉽게 파악할 수 있다.