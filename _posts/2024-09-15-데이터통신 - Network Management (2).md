---
layout: post
title: 데이터 통신 - Network Management (2)
date: 2024-09-15 20:20:23 +0900
category: DataCommunication
---
# 27.2 SNMP — Managers, Agents, SMI, MIB, SNMP 프로토콜 완전 정리

이 절에서는 네트워크 관리에서 가장 많이 쓰이는 관리 프레임워크인 **SNMP(Simple Network Management Protocol)** 를 구성하는 핵심 요소들을 정리한다.

- managers and agents  
- management components  
- an overview (SNMP 관리 프레임워크 개요)  
- SMI (Structure of Management Information)  
- MIB (Management Information Base)  
- SNMP 프로토콜 자체(버전·PDU·보안)

SNMP는 여전히 IETF의 **Internet Standard Management Framework**의 핵심으로, RFC 3411/STD 62(아키텍처), RFC 1157(SNMPv1), RFC 341x 시리즈(SNMPv3), RFC 2578/STD 58(SMIv2) 등에 기반한다. :contentReference[oaicite:0]{index=0}  

---

## 27.2.1 Managers and Agents

### 1) SNMP Manager란?

**SNMP Manager**(관리가, NMS: Network Management Station)는  
관리자가 앉아 있는 콘솔 혹은 그 콘솔이 사용하는 서버/소프트웨어를 의미한다.

기능적으로는:

- **Command Generator**  
  - `Get`, `GetNext`, `GetBulk`, `Set` PDU를 만들어 에이전트에게 전송
- **Notification Receiver**  
  - 에이전트가 보내는 `Trap`/`Inform` 알림을 수신
- **데이터 저장·시각화**  
  - 수집한 MIB 값을 시계열 DB, 이벤트 DB 등에 저장하고
  - 대시보드/리포트로 보여줌

RFC 3411/STD 62는 “하나 이상의 **SNMP entity**가 command generator, notification receiver 애플리케이션을 포함할 때 이를 전통적으로 **manager**라고 했다”고 설명한다. :contentReference[oaicite:1]{index=1}  

**예시**

- 상용 NMS: SolarWinds, Nagios, Zabbix, Cisco Prime, Juniper Junos Space 등
- 직접 구현: Python + Net-SNMP 라이브러리를 사용해 특정 MIB 값만 수집하는 내부 도구

### 2) SNMP Agent란?

**SNMP Agent**는 라우터·스위치·방화벽·서버·프린터 등 **관리 대상 장비**에 올라가는 프로세스/모듈이다.

역할:

- 로컬 OS/커널/드라이버/애플리케이션에서 **상태·통계 정보를 읽어서** MIB 객체로 노출
- Manager가 보내는 `Get`/`Set` 요청을 처리
- 상태 이상(예: 인터페이스 다운, CPU 과다 사용) 발생 시 **Trap/Inform**를 manager로 전송

SNMP 아키텍처 문서는 “하나 이상의 command responder 및 notification originator 애플리케이션을 포함한 SNMP entity를 전통적으로 **agent**라 부른다”고 정의한다. :contentReference[oaicite:2]{index=2}  

**현실 예**

- Cisco IOS/IOS-XE/IOS-XR, Junos OS, Linux net-snmp 데몬 등은 모두 SNMP 에이전트를 내장
- Cisco MDS SNMP 에이전트는 스위치 내부 장비 상태를 CISCO-MIB 계열로 노출한다. :contentReference[oaicite:3]{index=3}  

### 3) Manager–Agent 상호작용 개요

간단한 구조는 다음과 같다.

```text
          +-------------------------+
          |    NMS / SNMP Manager   |
          |  (Get/Set, Trap 수신)   |
          +-----------+-------------+
                      |
                SNMP over UDP
                      |
      ---------------------------------------
      |                  |                 |
+-----+-----+      +-----+-----+     +-----+-----+
|  Router 1 |      | Switch 1 |     |  FW 1     |
| SNMP Agent|      | SNMP Agent|    | SNMP Agent|
+-----------+      +-----------+    +-----------+
```

- Manager는 여러 Agent에게 주기적으로 `Get`/`GetBulk`를 보내 Pull 모니터링
- Agent는 중요 이벤트 발생 시 `Trap`/`Inform`으로 Push 알림

### 4) 간단 시나리오

**상황**: 운영팀이 “모든 코어 라우터의 CPU 사용률·인터페이스 트래픽·메모리 사용량”을 1분 주기로 수집해 NOC 대시보드를 구성하고자 한다.

- 각 라우터: SNMP 에이전트 활성화, 표준 MIB-II(`ifTable`, `system` 등) + 벤더 MIB(CISCO-CPU, JUNIPER-MIB 등) 제공 :contentReference[oaicite:4]{index=4}  
- NMS: SNMP Manager 역할  
  - 1분마다 각 라우터에 `GetBulk` 요청
  - 응답 받은 `ifInOctets`/`ifOutOctets`를 기반으로 인터페이스 이용률 계산
  - CPU/메모리 값이 임계값을 넘으면 알람 발행

---

## 27.2.2 Management Components

RFC 3411/STD 62는 SNMP Management Framework를 구성하는 **주요 컴포넌트**를 다음과 같이 정리한다. :contentReference[oaicite:5]{index=5}  

### 1) 구성 요소 목록

| 구성 요소 | 설명 |
|----------|------|
| Managed Device | 라우터·스위치·서버·프린터 등 관리 대상 네트워크 요소 |
| SNMP Agent | Managed device에서 동작하며 MIB를 통해 정보를 노출 |
| SNMP Manager | NMS, 모니터링/설정 툴 등, 여러 에이전트와 통신 |
| Management Information | SMI로 정의된 MIB 객체 집합 |
| SNMP Protocol / Engine | 메시지 포맷, PDU, 처리 로직(SNMP 엔진) |
| Security / Access Control | USM, VACM 등 SNMPv3 보안·접근 통제 모델 |

### 2) SNMP 엔진 구조 (RFC 3411)

SNMPv3 아키텍처에서 **SNMP 엔진(SNMP engine)** 은 다음 서브시스템으로 구성된다. :contentReference[oaicite:6]{index=6}  

- **Dispatcher** — 메시지 라우팅
- **Message Processing Subsystem** — SNMPv1/v2c/v3 메시지를 해석·조립
- **Security Subsystem** — USM 등 보안 모델 적용
- **Access Control Subsystem** — VACM 등으로 MIB 객체 접근 통제

각 엔진 위에는 다음과 같은 **애플리케이션**이 올라간다.

- Command Generator / Command Responder
- Notification Originator / Notification Receiver
- Proxy Forwarder 등

이를 텍스트 그림으로 표현하면:

```text
+------------------------------------------------------+
|                SNMP Entity (Manager/Agent)           |
|                                                      |
|  +------------------+  +-------------------------+   |
|  | SNMP Applications|  |       MIB Instruments   |   |
|  +---------+--------+  +-------------------------+   |
|            |                                          
|     +------+-----------------------------+           |
|     |           SNMP Engine              |           |
|     | +----------+  +--------+  +------+ |           |
|     | | Dispatcher| |Security| |Access| |           |
|     | +----------+  +--------+  +------+ |           |
|     |        Message Processing          |           |
|     +------------------------------------+           |
+------------------------------------------------------+
```

이 구조 덕분에 SNMP는 **새 버전(PDU 처리 방식)과 새로운 보안 모델**이 추가되어도 기존 아키텍처 안에서 모듈교체로 확장할 수 있다.

---

## 27.2.3 SNMP Overview — 동작 개요

### 1) SNMP의 기본 역할

SNMP는 크게 두 가지 목적에 쓰인다. :contentReference[oaicite:7]{index=7}  

1. **모니터링(Monitoring)**  
   - 상태·통계를 수집 (폴링 기반 `Get`, `GetBulk`)  
   - 비정상 이벤트 알림 (`Trap`, `Inform`)

2. **구성(Configuration)**  
   - 장비 설정값을 읽고(`Get`), 변경(`Set`)  
   - RFC 3512는 SNMP를 이용한 구성 관리 시 “일괄 변경 시 주의할 점, 롤백 전략” 등을 제시한다. :contentReference[oaicite:8]{index=8}  

### 2) 주요 PDU 타입

RFC 3416은 SNMP 프로토콜이 사용하는 **PDU 타입**을 정의한다. :contentReference[oaicite:9]{index=9}  

| PDU | 방향 | 용도 |
|-----|------|------|
| GetRequest | Manager → Agent | 지정된 OID 값 읽기 |
| GetNextRequest | M → A | 트리에서 “다음” OID 읽기 (walk) |
| GetBulkRequest | M → A | 많은 OID를 한 번에 가져오기(SNMPv2 이상) |
| SetRequest | M → A | 값 쓰기(구성 변경) |
| Response | A → M | 위 요청들에 대한 응답 |
| Trap | A → M | 비동기 알림(응답 없음) |
| Inform | A ↔ M | 확인 가능한 알림(응답 받음) |
| Report | v3 전용 | 보안/엔진 에러 정보 교환 등 |

### 3) 메시지 구조 (SNMPv3 기준)

SNMPv3 메시지는 대략 다음과 같은 계층 구조를 가진다. :contentReference[oaicite:10]{index=10}  

```text
+------------------------------+
| msgVersion (v3)              |
+------------------------------+
| msgGlobalData                |
+------------------------------+
| msgSecurityParameters (USM)  |
+------------------------------+
| msgData = encrypted(ScopedPDU)
+------------------------------+

ScopedPDU:
+---------------------------+
| contextEngineID, contextName |
+---------------------------+
| PDU (Get/Set/...)        |
+---------------------------+
```

- `msgSecurityParameters` 는 USM(User-based Security Model) 구조를 따르며,  
  인증(무결성)·암호화(기밀성)를 제공한다. :contentReference[oaicite:11]{index=11}  
- `ScopedPDU`는 “어떤 엔진ID/컨텍스트의 MIB”를 대상으로 하는지를 함께 포함해,  
  프록시나 멀티 컨텍스트 장비 환경에서 유연하게 관리할 수 있게 한다. :contentReference[oaicite:12]{index=12}  

### 4) Get/Response 흐름 예

```text
Manager                             Agent
  |                                   |
  |  GetRequest (ifInOctets.1)        |
  |---------------------------------->|
  |                                   |
  |            Response               |
  |   (ifInOctets.1 = 12345678)       |
  |<----------------------------------|
  |                                   |
```

여기서 `ifInOctets.1`은 `MIB-II::ifTable` 안 첫 번째 인터페이스의 수신 바이트 카운터를 의미한다. :contentReference[oaicite:13]{index=13}  

---

## 27.2.4 SMI (Structure of Management Information)

### 1) SMI의 역할

**SMI(Structure of Management Information)** 는 “MIB를 어떤 규칙으로 정의할 것인지”를 명시한 메타 규격이다.

- 어떤 **데이터 타입**을 쓸 수 있는지
- 어떤 **명명 규칙**과 **OID 트리 구조**를 따라야 하는지
- 각 객체에 어떤 메타데이터(상태, 접근 권한 등)를 붙여야 하는지

현재 표준은 **SMIv2**, RFC 2578/STD 58로 정의된다. :contentReference[oaicite:14]{index=14}  

### 2) SMIv2의 핵심 특징

1. **기본 데이터 타입**  

   - `INTEGER`, `OCTET STRING`, `OBJECT IDENTIFIER`
   - 부가 타입: `Unsigned32`, `Counter32`, `Gauge32`, `TimeTicks`, `Counter64` 등

2. **텍스트 규약(Textual Conventions)**  

   RFC 2579는 `DisplayString`, `PhysAddress`, `MacAddress` 같은 텍스트 규약을 정의해,  
   동일한 표현이 여러 MIB에서 일관되게 사용되도록 한다. :contentReference[oaicite:15]{index=15}  

3. **모듈 구조**  

   - MIB는 하나 이상의 **MODULE** 단위로 정의
   - 각 MODULE은 OID 트리 안에서 자신의 모듈 ID를 가진다.

4. **OBJECT-TYPE 매크로**

   각 객체는 `OBJECT-TYPE` 매크로로 정의하며, 다음 정보를 포함한다. :contentReference[oaicite:16]{index=16}  

   - `SYNTAX` — 데이터 타입(예: `Counter32`, `DisplayString`)
   - `MAX-ACCESS` — `read-only`, `read-write`, `not-accessible` 등
   - `STATUS` — `current`, `deprecated`, `obsolete`
   - `DESCRIPTION` — 사람이 읽을 수 있는 설명
   - `INDEX` — 테이블의 인덱스(행 식별자)
   - OID — 트리에서의 고유 위치

### 3) OID 트리 구조 예

SMI는 전체 MIB 공간을 **OID(Object Identifier) 트리**로 표현한다.  

일부 상위 노드는 다음과 같다. :contentReference[oaicite:17]{index=17}  

```text
iso(1)
 └── org(3)
     └── dod(6)
         └── internet(1)
             ├── directory(1)
             ├── mgmt(2)
             │    └── mib-2(1)
             ├── experimental(3)
             └── private(4)
                  └── enterprises(1)
```

- 표준 MIB-II는 `1.3.6.1.2.1` (`iso.org.dod.internet.mgmt.mib-2`)에 위치  
- 각 벤더는 `1.3.6.1.4.1` 이하에서 자신만의 enterprise OID를 할당받는다.

### 4) 간단한 OBJECT-TYPE 정의 예 (개념)

아래는 “장비의 가상 CPU 사용률”을 표현하는 새 MIB 객체를 정의한다고 가정한 예시이다.

```asn1
myCpuUsage OBJECT-TYPE
    SYNTAX      Gauge32
    MAX-ACCESS  read-only
    STATUS      current
    DESCRIPTION
        "Current CPU utilization in percent (0..100)."
    ::= { myDevice 1 }
```

여기서 `myDevice`는 해당 벤더의 MIB 모듈 안에서 정의된 상위 OID이다.

---

## 27.2.5 MIB (Management Information Base)

### 1) MIB의 의미

**MIB**는 “네트워크 장비의 관리 정보를 객체 집합으로 정의한 **가상 데이터베이스**”라고 볼 수 있다. :contentReference[oaicite:18]{index=18}  

- SNMP Manager와 Agent는 공통의 MIB 정의를 공유해야 한다.
- MIB는 “어떤 OID에 어떤 의미의 값이 저장되어 있는지”를 명확히 규정한다.
- 실제 값은 에이전트 내부(커널·프로세스·드라이버 등)에 있지만,  
  논리적으로는 MIB 트리의 노드에 대응된다.

Cisco의 SNMP 문서는 “MIB는 SNMP 네트워크 요소를 데이터 객체 목록으로 표현한 구조이며,  
Manager는 각 장비 타입의 MIB 파일을 컴파일해야 해당 장비를 모니터링할 수 있다”고 설명한다. :contentReference[oaicite:19]{index=19}  

### 2) 표준 MIB-II 예

대표적인 표준 MIB-II에는 다음 같은 그룹들이 포함된다. :contentReference[oaicite:20]{index=20}  

| 그룹 | 예시 객체 | 의미 |
|------|-----------|------|
| `system` | `sysDescr`, `sysUpTime` | 장비 설명, 가동 시간 |
| `interfaces` | `ifNumber`, `ifTable` | 인터페이스 개수·속성·트래픽 |
| `ip` | `ipForwarding`, `ipInReceives` | IPv4 관련 통계/설정 |
| `tcp`, `udp` | `tcpInSegs`, `udpInDatagrams` | TCP/UDP 통계 |
| `snmp` | `snmpInPkts`, `snmpOutPkts` | SNMP 자체 통계 |

**예시**: `sysUpTime.0`  

- OID: `1.3.6.1.2.1.1.3.0`  
- 의미: 장비가 부팅된 후 경과한 시간(1/100초 단위 `TimeTicks`)  
- Manager는 이 값을 주기적으로 읽어, 부팅/리로드 여부를 감지할 수 있다.

### 3) Enterprise MIB 예

벤더별로 **확장 MIB**를 정의한다.

- Cisco: `CISCO-CPU-MIB`, `CISCO-IF-EXTENSION-MIB`, `CISCO-CVP-MIB` 등 :contentReference[oaicite:21]{index=21}  
- Juniper: `JUNIPER-MIB`, `JUNIPER-IF-MIB` 등

예를 들어 Cisco CVP SNMP 에이전트는 `CISCO-CVP-MIB`를 통해 장비 등록 상태, IP 주소, 모델 타입 등을 노출한다. :contentReference[oaicite:22]{index=22}  

### 4) 테이블형 MIB 예: ifTable

`ifTable`은 인터페이스별 속성을 행(row) 단위로 가지는 대표적인 테이블이다. :contentReference[oaicite:23]{index=23}  

| 열(OID suffix) | 의미 |
|----------------|------|
| `ifIndex`      | 인터페이스 고유 인덱스 |
| `ifDescr`      | 인터페이스 설명 (예: `GigabitEthernet0/1`) |
| `ifType`       | 매체 유형 (ethernetCsmacd 등) |
| `ifSpeed`      | 속도(bps) |
| `ifAdminStatus`| 관리 상태(up/down) |
| `ifOperStatus` | 실제 동작 상태(up/down/testing) |
| `ifInOctets`   | 수신 바이트 수 |
| `ifOutOctets`  | 송신 바이트 수 |

예를 들어 `ifInOctets.3`은 `ifIndex=3`인 인터페이스의 수신 바이트 수이다.

---

## 27.2.6 SNMP — 프로토콜 동작과 버전

### 1) SNMP 버전별 특징

| 버전 | 요약 | 특징 |
|------|------|------|
| SNMPv1 | 초기 버전(이제 Historic) | 단순 기능, community 기반 “보안” (실질적으로 평문 비밀번호 수준), 제한된 오류 처리 |
| SNMPv2c | 개선된 PDU, bulk 전송 | `GetBulk`, 향상된 에러 처리, 여전히 community 기반 |
| SNMPv3 | 현재 표준 | **USM**(User-based Security Model)과 **VACM**(View-based Access Control Model)로 **인증·암호화·접근통제** 제공 :contentReference[oaicite:24]{index=24} |

- RFC 1157은 SNMPv1의 기본 프레임워크를 정의한다. :contentReference[oaicite:25]{index=25}  
- RFC 3411/STD 62 아키텍처와 RFC 3414 USM, RFC 3415 VACM이 SNMPv3를 구성한다. :contentReference[oaicite:26]{index=26}  

실무에서는 **읽기 전용 모니터링**은 여전히 v2c를 쓰는 곳도 많지만,  
새로운 설계에서는 가능한 한 **v3(authPriv)** 를 기본으로 한다.

### 2) SNMPv3 보안 모델 개요

- **USM (User-based Security Model)** :contentReference[oaicite:27]{index=27}  
  - 사용자 ID 기반  
  - 인증: HMAC-SHA, HMAC-MD5 등  
  - 암호화: DES, AES 등 (AES가 사실상 권장)

- **VACM (View-based Access Control Model)** :contentReference[oaicite:28]{index=28}  
  - MIB “뷰”를 정의하여 사용자·그룹마다 읽기/쓰기 가능한 OID 범위를 제한  
  - 예: 운영팀 계정은 모든 인터페이스 통계 read-only,  
    보안팀 계정은 방화벽 정책 MIB read/write 등

### 3) PDU별 동작 예 — GetBulk

**상황**: Manager가 `ifTable` 전체를 한 번에 가져오고 싶다.

- v1 시절: `GetNext`를 반복해서 수행해야 했음
- v2c/v3: `GetBulk`로 **한 번 요청에 여러 행**을 가져올 수 있음

개념적 흐름:

```text
Manager                           Agent
  |                                 |
  |  GetBulk(ifTable, max-repetitions=10)  |
  |--------------------------------------->|
  |                                 |
  |       Response(최대 10행 데이터)       |
  |<---------------------------------------|
```

이를 반복하며 테이블 전체를 효율적으로 walk할 수 있다.

---

## 27.2.7 실전 예제: 간단한 SNMP 모니터링/알림

이 절은 이해를 돕기 위한 예제 코드와 시나리오를 보여준다. 실제 운영 환경에서는 보안 설정(SNMPv3 사용자, 암호, 뷰 등)을 더 엄격히 구성해야 한다.

### 1) 예제 1 — Python으로 SNMPv3 `Get` 수행 (개념)

```python
from pysnmp.hlapi import *

def get_snmpv3(oid, host='192.0.2.1'):
    iterator = getCmd(
        SnmpEngine(),
        UsmUserData('monitorUser', 'authPass123', 'privPass123',
                    authProtocol=usmHMACSHAAuthProtocol,
                    privProtocol=usmAesCfb128Protocol),
        UdpTransportTarget((host, 161), timeout=1, retries=2),
        ContextData(),
        ObjectType(ObjectIdentity(oid))
    )

    errorIndication, errorStatus, errorIndex, varBinds = next(iterator)

    if errorIndication:
        raise RuntimeError(errorIndication)
    elif errorStatus:
        raise RuntimeError(f'{errorStatus.prettyPrint()} at {errorIndex}')
    else:
        for varBind in varBinds:
            return varBind

result = get_snmpv3('1.3.6.1.2.1.1.3.0')  # sysUpTime.0
print("sysUpTime:", result)
```

**설명**

- `UsmUserData` 를 통해 SNMPv3 USM 사용자/패스워드/알고리즘 지정
- `ObjectIdentity`에 MIB 이름을 사용할 수도 있다(`SNMPv2-MIB`, `sysUpTime.0` 형태)
- 실제 코드는 재시도, 예외처리, 로그 등을 더 강화해야 한다.

### 2) 예제 2 — 인터페이스 이용률 계산 시나리오

**목표**: 1분 간격으로 `ifInOctets`/`ifOutOctets`를 읽어 인터페이스 이용률을 계산하고, 80% 이상이면 경고.

이론적으로는 앞서 언급한 식을 사용한다.

$$
\text{Utilization} = 
\frac{\Delta \text{Octets} \times 8}{R \times \Delta t}
$$

여기서:
- $$\Delta \text{Octets} = \text{Octets}_{\text{now}} - \text{Octets}_{\text{prev}}$$  
- $$R$$: 링크 속도(bps)  
- $$\Delta t$$: 측정 간격(초)

간단한 파이썬 형태 예:

```python
import time
from pysnmp.hlapi import *

def get_counter(oid):
    # 위 get_snmpv3 함수와 유사하다고 가정
    # 반환값은 정수형 카운터라고 가정
    ...

if_speed = 1_000_000_000  # 1 Gbps
interval = 60

prev_in = get_counter('1.3.6.1.2.1.2.2.1.10.3')   # ifInOctets.3
time.sleep(interval)
curr_in = get_counter('1.3.6.1.2.1.2.2.1.10.3')

delta = curr_in - prev_in
util = delta * 8 / (if_speed * interval)

print(f"Interface utilization: {util * 100:.2f}%")
if util > 0.8:
    print("WARNING: High utilization")
```

실제 운영에서는 **64-bit 카운터(`ifHCInOctets`) 사용**, 카운터 wrap-around 처리, SNMP 타임아웃, 에러 처리 등을 고려해야 한다.

### 3) 예제 3 — Trap 기반 장애 알림 시나리오

**상황**

- 스위치 인터페이스가 down되면, SNMP Trap을 NMS로 전송하고 싶다.

구성 흐름(개념):

1. 스위치에서 SNMP 에이전트 설정
   - SNMPv3 사용자 생성
   - Trap 대상 NMS IP/포트 등록
2. 인터페이스 링크 변화 시 이벤트 발생
   - 내부적으로 `linkDown` Trap이 생성되어 NMS로 전송
3. NMS 측
   - Trap 리스너가 `linkDown` 이벤트를 수신
   - 해당 인터페이스 상태를 DB에 기록, 알람 생성, 온콜에게 통지

이 과정에서 Trap 메시지는 `SNMPv2-MIB::snmpTrapOID` 와 인터페이스 인덱스, `ifDescr` 등을 포함한다.

---

## 27.2.8 정리

27.2 절에서 다룬 내용을 간단히 요약하면 다음과 같다.

- **Managers and Agents**  
  - Manager는 `Get`/`Set`을 보내고 Trap/Inform을 받는 쪽,  
    Agent는 장비 측에서 MIB를 노출하고 알림을 보내는 쪽이다.
- **Management Components**  
  - SNMP 엔진(Dispatcher, Message Processing, Security, Access Control)과  
    그 위의 애플리케이션(Command Generator/Responder, Notification Originator/Receiver) 구조를 가진다.
- **Overview**  
  - SNMP는 모니터링·구성 관리용 프로토콜이며, `Get`/`Set`/`GetBulk`/`Trap`/`Inform` 등 PDU를 사용한다.
- **SMI**  
  - SMIv2(표준: RFC 2578/STD 58)는 데이터 타입, OID 트리, OBJECT-TYPE 구조 등을 정의한다.
- **MIB**  
  - MIB는 관리 정보를 객체 트리로 정의한 가상 DB이며, MIB-II와 Enterprise MIB로 구성된다.
- **SNMP (Protocol)**  
  - SNMPv3는 USM·VACM을 통해 인증·암호화·접근통제를 제공하고,  
    RFC 3411 아키텍처 하에서 모듈식으로 확장 가능한 구조를 가진다.