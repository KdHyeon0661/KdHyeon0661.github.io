---
layout: post
title: 데이터 통신 - Network Management (3)
date: 2024-09-15 21:20:23 +0900
category: DataCommunication
---
# NETCONF/YANG과 스트리밍 텔레메트리 — SNMP 이후의 네트워크 관리

**NETCONF/YANG + 스트리밍 텔레메트리**는 “모델 기반(Model-driven)·API 기반”의 **2세대 관리 프레임워크**라고 볼 수 있다.

이 글에서는:

- 왜 SNMP만으로는 부족해졌는지
- **NETCONF**(프로토콜)와 **YANG**(데이터 모델)의 역할 분담
- NETCONF/YANG 기반 구성 자동화 워크플로
- **스트리밍 텔레메트리(Streaming Telemetry)** 의 개념
- IETF의 **YANG-Push** 계열과, OpenConfig의 **gNMI 기반 텔레메트리** 개념

까지 한 번에 묶어서 정리한다.

---

## 1. 왜 NETCONF/YANG·스트리밍 텔레메트리가 필요했는가?

SNMP는 지금도 모니터링에 널리 쓰이지만, 다음 한계가 명확했다.

1. **구성(Configuration)에는 부적합**
   - SNMP `Set`으로 장비를 설정할 수는 있지만,
     - “여러 파라미터를 하나의 트랜잭션으로 묶어서 적용”
     - “롤백”
     - “상태 검증 후 커밋”
     같은 고급 기능이 매우 부족하다.
2. **데이터 모델이 느슨함**
   - MIB는 있지만, 실제 장비들이 구현하는 MIB가 제각각이다.
   - 설정/상태를 풍부하게 표현하기 어렵고, 벤더마다 표현이 다르다.
3. **폴링 스케일의 한계**
   - 1분마다 수천·수만 장비를 폴링하면 CPU·대역폭에 부담.
   - 고주파수(수백 ms 단위) 데이터는 거의 불가능.

이에 대해 IETF는

- **NETCONF** — XML 기반 **구성 관리 프로토콜**(RPC·트랜잭션·락 지원)   
- **YANG** — NETCONF/RESTCONF 등에서 사용할 **정형 데이터 모델 언어**(구성·상태·RPC·알림)   

을 정의했고, 이후 **YANG-Push·데이터스토어 텔레메트리**가 등장했다.   
또한 업계에서는 OpenConfig의 **gNMI 기반 스트리밍 텔레메트리**가 사실상의 표준으로 자리 잡아가고 있다.   

---

## 2. NETCONF — XML/RPC 기반 구성·운영 프로토콜

### 2.1 NETCONF 개요

**NETCONF(Network Configuration Protocol)** 는

- 장비의 **구성(config)** 을 설치·조작·삭제하고
- **상태(state)** 를 조회하며
- **알림(notification)** 을 주고받기 위한

네트워크 관리 프로토콜이다. RFC 6241은 NETCONF를 XML 기반 데이터 인코딩과 RPC 연산을 사용하는 구성 프로토콜로 정의한다.   

핵심 특징:

- **RPC 스타일**  
  - `get-config`, `edit-config`, `lock`, `commit` 같은 연산을 RPC로 정의.
- **트랜잭션적 구성 변경**
  - 단일 RPC로 여러 설정을 하나의 트랜잭션처럼 처리.
- **여러 데이터스토어 지원**
  - `running`, `candidate`, `startup` 등 논리적 데이터스토어 개념.
- **보안 전제**
  - 일반적으로 **SSH 위에서 동작** (또는 TLS 등).
- **데이터 모델 독립**  
  - 프로토콜 자체는 모델 독립적이지만, **YANG 사용을 권장**.   

### 2.2 NETCONF 계층 구조

RFC 6241과 IETF NETCONF WG 문서는 NETCONF를 다음 네 계층으로 바라본다.   

1. **내용(Content) 계층**
   - 실제 **구성/상태/알림 데이터** (대부분 YANG 모델로 정의)
2. **연산(Operations) 계층**
   - `get`, `get-config`, `edit-config`, `copy-config`, `lock`, `commit`, `close-session` 등
3. **메시지(Message) 계층**
   - RPC/응답 메시지 구조, `<rpc>`, `<rpc-reply>` 등의 포맷
4. **보안(SSH/TLS) 운반 계층**
   - SSH, TLS 등 보안 전송 프로토콜 위에서 동작

### 2.3 주요 NETCONF 연산(RPC)

대표적인 연산들:

- `get-config` — 특정 데이터스토어의 구성 읽기
- `edit-config` — 구성 변경(추가/수정/삭제)
- `copy-config` — 한 데이터스토어를 다른 데이터스토어로 복사
- `lock` / `unlock` — 특정 데이터스토어를 잠가 동시수정 방지
- `commit` / `discard-changes` — `candidate` → `running` 적용·폐기
- `get` — 상태/운영상 데이터 포함 조회
- `create-subscription` — 알림(Notification) 스트림 구독 (RFC 5277 등)

**예: 인터페이스 설명 변경(개념적 XML)**

```xml
<rpc message-id="101"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <running/>
    </target>
    <config>
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
        <interface>
          <name>ge-0/0/0</name>
          <description>Uplink to core switch</description>
        </interface>
      </interfaces>
    </config>
  </edit-config>
</rpc>
```

위 XML은 NETCONF 클라이언트가 장비에 보내는 RPC의 구조만 보여준다. 실제 장비 CLI 대신, **정형 모델(YANG) + NETCONF RPC**로 구성 변경을 수행하는 방식이다.

### 2.4 데이터스토어 개념

NETCONF는 구성을 하나의 덩어리가 아닌 **여러 데이터스토어**로 구분한다.   

- `running` — 현재 활성 구성
- `candidate` — 커밋 전 임시 작업 영역
- `startup` — 재부팅 시 로드될 구성

**실전 워크플로 예**

1. 자동화 시스템이 `candidate`에 변경사항을 `edit-config`로 적용
2. `validate` or `commit` 테스트 (장비에 따라 differ)
3. 문제가 없으면 `commit`
4. 문제가 있으면 `discard-changes`

이로써 “대규모 설정 변경 후 장애 시 롤백” 같은 시나리오를  
장비 자체의 트랜잭션으로 처리할 수 있다.

---

## 3. YANG — NETCONF/RESTCONF를 위한 데이터 모델 언어

### 3.1 YANG 개요

**YANG(Yet Another Next Generation)** 은 네트워크 설정·상태 데이터, RPC, 알림을 모델링하는 데이터 모델 언어이다. RFC 7950은 YANG 1.1을 정의하면서, YANG이 구성/상태 데이터, RPC, 알림을 모델링하고, NETCONF/RESTCONF와 함께 사용된다고 설명한다.   

핵심 특징:

- **트리 구조**  
  - 컨테이너, 리스트, 리프(leaf), 선택(choice) 등으로 구성.
- **타입 시스템**  
  - `string`, `uint32`, `enumeration`, `leafref`, `union`, `identityref` 등 풍부한 타입.
- **제약(Constraints)**  
  - `must`, `when`, `pattern` 등으로 값과 구조에 제약.
- **RPC·알림 지원**
  - 특정 작업을 RPC로 정의하고, 알림(notifications)을 모델링.
- **모듈·서브모듈 구조**
  - 재사용 가능한 모듈 시스템.

### 3.2 YANG 모듈 구조 예 (개념)

간단한 YANG 모듈 예시:

```yang
module example-interfaces {
  namespace "urn:example:interfaces";
  prefix "exif";

  import ietf-yang-types { prefix yang; }

  container interfaces {
    list interface {
      key "name";
      leaf name {
        type string;
      }
      leaf enabled {
        type boolean;
        default "true";
      }
      leaf description {
        type string;
      }
      leaf mtu {
        type uint32 {
          range "1280..9000";
        }
      }
    }
  }
}
```

이 모듈은

- `interfaces/interface` 리스트
- `name`·`enabled`·`description`·`mtu` 리프

를 정의한다. NETCONF `edit-config`는 이 모델에 맞춰 XML을 보낸다.

### 3.3 YANG으로 무엇을 모델링하는가?

IETF와 운영자 커뮤니티는 다음과 같은 범위의 YANG 모델을 정의해왔다.

- **코어 프로토콜 모델**
  - TCP, NTP, MPLS LDP 등. 예를 들어 RFC 9648은 TCP를 위한 최소 YANG 데이터 모델을 정의한다.   
- **장비·서비스 모델**
  - 인터페이스(ietf-interfaces)
  - 라우팅(ietf-routing)
  - QoS, ACL, VPN, 서비스 체이닝 등

YANG 모델들은 장비·서비스의 구성을 **표준화된 스키마**로 만드는 역할을 한다.

---

## 4. NETCONF + YANG 기반 구성 자동화 워크플로

### 4.1 SNMP vs NETCONF/YANG 구성 방식 비교

| 관점 | SNMP | NETCONF + YANG |
|------|------|----------------|
| 데이터 표현 | MIB 기반, 비교적 단순 타입 | YANG 기반, 구조화된 트리, 풍부한 타입 |
| 구성 변경 | `Set`(OID별), 트랜잭션·롤백 부족 | `edit-config`, `candidate/running`, `commit/rollback` |
| 검증 | 제한적 | `validate`, `must/when` 제약 조건, 스키마 검증 |
| 사용 패턴 | 주로 모니터링 | 구성 + 모니터링 (운영 데이터 조회) |

### 4.2 실제 사용 시나리오

**시나리오: ISP의 L3 코어 라우터 50대에 BGP 피어 구성을 자동 배포**

1. 네트워크 아키텍처 팀이 BGP 서비스에 대한 **YANG 모델**(또는 IETF/벤더 표준 모델)을 선정.
2. 자동화 도구(예: Python, Ansible, NSO)가
   - 사이트별 파라미터(AS 번호, 피어 IP, 정책)를 입력받아
   - YANG 모델 인스턴스(XML/JSON)를 생성.
3. NETCONF 클라이언트가 각 라우터에 `edit-config`로 `candidate`에 적용.
4. `validate` → `commit` 순서로 반영.
5. 텔레메트리(YANG-Push 또는 gNMI)로 BGP 세션 상태를 모니터링.

---

## 5. 스트리밍 텔레메트리 — 폴링에서 Pub/Sub로

### 5.1 개념

**스트리밍 텔레메트리(Streaming Telemetry)** 는

- 애플리케이션이 **“구독(subscription)”** 을 만들고
- 장비가 조건에 따라 **실시간/주기적으로 데이터를 푸시**하는 모델이다.

IETF 문서는 이를 YANG 데이터스토어를 대상으로 한 **pub/sub 서비스**라고 묘사한다.   

장점:

- **폴링의 비효율 제거**
  - 필요할 때마다 매번 `Get`을 날리는 대신, 한 번 구독해두고 계속 받는다.
- **고주파수 데이터**
  - 1초, 100ms 단위 등 고빈도 측정이 가능.
- **정형 구조 유지**
  - YANG 모델 기반 경로를 그대로 사용.

### 5.2 대표 구현 라인

1. **IETF YANG-Push / Subscribed Notifications**
   - YANG 데이터스토어에 대한 구독·푸시를 RFC 8639 등에서 정의.   
   - NETCONF/RESTCONF 기반.
2. **OpenConfig gNMI Telemetry**
   - gRPC 기반 프로토콜로, 구성+상태 조회+스트리밍 텔레메트리를 모두 커버.   

---

## 6. IETF YANG-Push 계열 텔레메트리

### 6.1 Subscribed Notifications / YANG-Push 개요

- **RFC 7923**: YANG 데이터스토어 구독 요구사항
- **RFC 8639, 8640, 8641**: Subscribed Notifications 및 YANG-Push 메커니즘 정의   

핵심 아이디어:

- 관리 애플리케이션(구독자)이 장비(퍼블리셔)에 **구독 요청**(subscription)을 생성
- 구독은
  - 어떤 **YANG 경로**(예: `/interfaces/interface/state/counters`)
  - 어떤 **조건**(주기 샘플링 vs 상태 변경 시)
  - 어떤 **필터**(특정 인터페이스만)
  를 포함
- 장비는 구독 조건에 맞춰 데이터를 푸시한다.

### 6.2 구독 유형

YANG-Push 관련 문서와 이후 작업(draft, YANG Push Lite 등)은 대략 다음 세 가지 유형을 다룬다.   

1. **주기(PERIODIC)**
   - 일정 주기로 전체 데이터를 푸시
2. **변경 기반(ON-CHANGE)**
   - 값이 바뀔 때만 푸시
3. **혼합/최적화 모드**
   - 최초 full 상태 + 이후 변경 이벤트

예를 들어, 인터페이스 카운터는 `PERIODIC`(1초 간격),  
BGP 세션 상태는 `ON-CHANGE` 로 받는 식이다.

### 6.3 NETCONF/RESTCONF를 통한 동작

- NETCONF: `establish-subscription`, `modify-subscription`, `delete-subscription` 같은 RPC로 구독을 관리.
- RESTCONF: HTTP 기반으로 동일한 구독 모델 구현.

개념적 RPC 예(기본 구조만):

```xml
<rpc message-id="101"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <establish-subscription
     xmlns="urn:ietf:params:xml:ns:yang:ietf-subscribed-notifications">
    <stream>yang-push</stream>
    <filter xmlns:if="urn:ietf:params:xml:ns:yang:ietf-interfaces">
      /if:interfaces/if:interface/if:state/if:counters
    </filter>
    <period>1000</period> <!-- 1000 ms -->
  </establish-subscription>
</rpc>
```

이후 장비는 `notification` 메시지 형태로 주기마다 인터페이스 카운터를 푸시한다.

### 6.4 YANG Push Lite / Observability 작업

최근 IETF는 **YANG Push Lite** 와 Observability 향상 초안을 통해  
YANG 데이터스토어 텔레메트리의 경량화·확장성을 논의하고 있다.   

- 목적: 초저지연·대량 데이터 환경에서의 효율적 운영데이터 관측
- 메시지 포맷, 메타데이터 구조, 분산 수집기 연동 등을 다룸

세부 문법은 계속 변동 중이지만, **“YANG 모델 + pub/sub + 푸시”** 라는 큰 구조는 유지된다.

---

## 7. OpenConfig gNMI 기반 스트리밍 텔레메트리

### 7.1 gNMI란?

**gNMI(gRPC Network Management Interface)** 는 OpenConfig 커뮤니티가 정의한  
gRPC 기반 관리 프로토콜이다. 문서와 초안은:

- OpenConfig gNMI 스펙: gRPC와 YANG 트리 기반 구성·상태 조회·텔레메트리용 서비스 정의   

특징:

- **gRPC** 기반 — HTTP/2 위에서 바이너리(Protocol Buffers) 전송
- **하나의 서비스**로
  - 구성 변경(Set)
  - 상태 조회(Get)
  - 스트리밍 텔레메트리(Subscribe)
  를 모두 제공
- 데이터 모델로 **OpenConfig YANG** 또는 IETF YANG 사용

### 7.2 gNMI 주요 RPC

gNMI 스펙은 대략 다음 RPC들을 정의한다.   

- `Get` — 현재 상태/구성 읽기
- `Set` — 구성 변경(Replace, Update, Delete)
- `Subscribe` — 스트리밍 텔레메트리 구독

Subscribe 요청에는 다음 요소가 포함된다.

- 하나 이상의 **Subscription**
  - 경로(Path): YANG 트리 경로
  - 모드: SAMPLE, ON_CHANGE, TARGET_DEFINED 등   
  - 간격(sample_interval) 등

### 7.3 구독 모드 예시

벤더 문서(NVIDIA NVOS, Juniper 등)는 gNMI 스트리밍 모드를 다음처럼 설명한다.   

1. **SAMPLE**
   - 설정한 간격마다 샘플 데이터 전송
   - 예: 10초마다 인터페이스 카운터 전체
2. **ON_CHANGE**
   - 값이 변경될 때만 전송
   - 예: 링크 up/down, BGP 세션 상태
3. **TARGET_DEFINED**
   - 장비가 최적 모드를 자체 결정

### 7.4 gNMI Subscribe 예(개념적 코드)

아래는 Python/gRPC 스타일의 **개념 코드**로, 실제 라이브러리 호출 방식은 다를 수 있다.

```python
import gnmi_pb2
import gnmi_pb2_grpc
import grpc

channel = grpc.secure_channel('router1.example.net:9339', creds)
stub = gnmi_pb2_grpc.gNMIStub(channel)

# 인터페이스 카운터 10초 간격 샘플링
subscription = gnmi_pb2.Subscription(
    path=gnmi_pb2.Path(elem=[
        gnmi_pb2.PathElem(name='interfaces'),
        gnmi_pb2.PathElem(name='interface', key={'name': 'ge-0/0/0'}),
        gnmi_pb2.PathElem(name='state'),
        gnmi_pb2.PathElem(name='counters')
    ]),
    mode=gnmi_pb2.SubscriptionMode.SAMPLE,
    sample_interval=10_000_000_000  # 10초, ns 단위
)

sub_list = gnmi_pb2.SubscriptionList(
    subscription=[subscription],
    mode=gnmi_pb2.SubscriptionList.Mode.STREAM
)

request = gnmi_pb2.SubscribeRequest(subscribe=sub_list)

for response in stub.Subscribe(request):
    for update in response.update.update:
        print(update.path, update.val)
```

실제 구현에서는 인증(TLS), 재연결, 버퍼링, 백프레셔, 메트릭 전송 등을 추가해야 한다.

### 7.5 OpenConfig 텔레메트리 아키텍처 예

OpenConfig·넷플릭스 등은 gNMI 텔레메트리를 기반으로 한  
게이트웨이/수집기 아키텍처를 소개한다.   

```text
   +--------------------+
   |  Analytics / DB    |
   | (Prometheus, TSDB) |
   +----------+---------+
              ^
              |  Kafka / gNMI-gateway / Collector
              |
   +----------+----------+
   |   gNMI Gateway      |
   +----------+----------+
              ^
              | gNMI Subscribe
   -----------------------------------------------
   |                  |                         |
+--+--------+   +-----+-------+          +------+------+
| Router A  |   | Router B    |          | Router C    |
| gNMI srv  |   | gNMI srv    |          | gNMI srv    |
+-----------+   +-------------+          +-------------+
```

- 각 라우터가 gNMI 서버 역할
- 중앙의 gNMI 게이트웨이가 데이터를 수집·정규화 후
- Kafka/TSDB 등으로 전달

---

## 8. NETCONF/YANG vs 스트리밍 텔레메트리 vs SNMP — 정리

### 8.1 역할 분담

| 항목 | SNMP | NETCONF/YANG | 스트리밍 텔레메트리(YANG-Push, gNMI 등) |
|------|------|--------------|------------------------------------------|
| 주 용도 | 모니터링 중심 | 구성 + 운영 상태 | 고주파수/대량 상태 데이터 수집 |
| 데이터 모델 | MIB | YANG | YANG (동일/유사 모델) |
| 전송 방식 | 폴링(Get) + Trap | RPC 기반 요청/응답 | Pub/Sub, 지속 스트림 |
| 보안 | v3에서 인증/암호화 | SSH/TLS 기반 | TLS + gRPC 등 |
| 트랜잭션 | X | O (`candidate`/`commit`) | 해당 없음(읽기 전용) |

현대 네트워크에서는 대략 다음처럼 쓰는 패턴이 많다.

- **SNMP**: 레거시 시스템, 간단한 모니터링
- **NETCONF + YANG**: 장비 구성·서비스 프로비저닝
- **YANG-Push / gNMI Telemetry**: 실시간 운영 데이터 수집, 분석, AIOps

---

## 9. 실전 시나리오 정리

마지막으로, **하나의 운영 시나리오** 안에  
NETCONF/YANG과 스트리밍 텔레메트리를 함께 사용하는 모습을 정리해보자.

### 9.1 시나리오: 데이터센터 패브릭 구축 및 운영

**환경**

- 데이터센터 리프/스파인 스위치 수십대
- BGP EVPN 패브릭
- 원하는 것:
  - 선언적(desired state) 방식 구성
  - 초 단위 텔레메트리로 장애 예방

**구성 단계**

1. 설계팀이 패브릭 토폴로지와 주소 계획을 YANG 기반 서비스 모델로 정의.
2. 자동화 시스템(예: 자체 Python 앱 또는 상용 NMS)이
   - 노드별 파라미터(노드 이름, 루프백, ASN 등)를 입력으로
   - YANG 모델 인스턴스를 생성.
3. NETCONF 클라이언트가 각 스위치에 대해
   - `edit-config`(candidate)에 구성 반영
   - `validate` 후 `commit`으로 활성화
4. 필요 시 롤백/리비전 관리도 NETCONF 레벨에서 수행.

**운영 단계**

- 텔레메트리 수집기:
  - 인터페이스/큐/버퍼/CPU/메모리·BGP/EVPN 상태를
  - gNMI Subscribe 혹은 YANG-Push 구독으로 수집
  - SAMPLE(주기) + ON_CHANGE(이벤트 기반) 조합
- 분석:
  - TSDB에 시계열로 저장, SLO/SLA 기반 대시보드·알람
  - Machine Learning 기반 이상 탐지 등에 활용

**장점**

- **정형 모델(YANG)** 하나만 이해하면
  - 구성(NETCONF/RESTCONF)
  - 상태 조회(NETCONF/gNMI Get)
  - 텔레메트리(YANG-Push/gNMI Subscribe)
  를 모두 같은 “데이터 스키마”로 다룰 수 있다.
- 운영 자동화·검증·리포팅·분석 파이프라인까지  
  하나의 공통 데이터 모델에 수렴하게 된다.

---

이 글에서는 **NETCONF/YANG**과 **스트리밍 텔레메트리(YANG-Push, gNMI)** 를  
SNMP 이후의 **현대 네트워크 관리 스택의 핵심**으로 묶어 살펴보았다.

실제 구현으로 들어가려면:

- 특정 벤더(예: Cisco, Juniper)의 NETCONF/YANG 지원 범위
- OpenConfig vs IETF YANG 모델 선택
- gNMI Collector/게이트웨이 도입 (예: gnmi-gateway, 자체 collector)

같은 현실적인 결정을 해야 한다.  
그러나 큰 그림에서는 **“모델 기반 + API 기반 + 스트리밍 기반”** 이라는 흐름이 명확하며,  
앞으로의 네트워크 자동화/관측성(Observability)의 기본 축이 된다.