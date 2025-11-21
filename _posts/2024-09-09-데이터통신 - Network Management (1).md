---
layout: post
title: 데이터 통신 - Network Management (1)
date: 2024-09-09 23:20:23 +0900
category: DataCommunication
---
# Network Management — FCAPS 관점의 개념과 실전 예제

이 장은 네트워크 관리의 고전적인 **FCAPS 모델**(Fault, Configuration, Accounting, Performance, Security)을 기준으로,
실제 운영 환경에서 어떻게 적용되는지 정리한 것이다. 이 중 27.1에서는 FCAPS의 다섯 영역을 먼저 소개한다.

---

## Introduction — Network Management와 FCAPS

### FCAPS 개요

ISO TMN(전송망 관리)·OSI 네트워크 관리 모델에서 네트워크 관리 기능은 보통 **다섯 영역**으로 나눈다.

| 영역 | 역할(요약) |
|------|-----------|
| Configuration Management | 장비·서비스의 설정·버전·변경 이력 관리 |
| Fault Management        | 장애 감지·격리·복구·알림 |
| Accounting Management   | 사용량·비용·과금·쇼백(showback)/차지백(chargeback) |
| Performance Management  | 성능 지표(지연, 손실, 대역폭 사용률 등) 측정·최적화 |
| Security Management     | 접근 통제, 정책 준수, 취약점·위협 모니터링 |

오늘날 상용 NMS(Network Management System)나 OSS/BSS, SDN 컨트롤러, 클라우드 관리 플랫폼도 대부분 이 FCAPS 영역을 기본 구조로 삼는다.

---

## Configuration Management

### 1) 정의와 목적

NIST는 **구성 관리(Configuration Management)** 를
“시스템 개발 생명주기 전체에 걸쳐, 초기화·변경·모니터링 과정을 통제하여 정보시스템의 무결성을 유지하는 활동 집합”으로 정의한다.

네트워크 관점에서 구성 관리는 다음을 포함한다.

- 어떤 장비(라우터, 스위치, 방화벽, AP)가 어디에 있고
- 어떤 OS/펌웨어 버전을 돌리고 있으며
- 어떤 설정(인터페이스, VLAN, ACL, 라우팅 프로토콜, QoS 정책)을 사용하고
- 변경이 언제, 누가, 왜 수행되었는지

를 **일관되고 재현 가능하게 관리**하는 것.

### 2) 핵심 기능

1. **인벤토리 관리(Inventory)**
   - 장비 모델, 직렬번호, 위치(랙·RU), OS 버전 등 메타데이터 관리
2. **버전 관리(Versioning)**
   - 각 장비 설정을 Git 같은 VCS에 저장해 **diff/rollback** 가능하게
3. **변경 관리(Change Management)**
   - 변경 요청 → 검토/승인 → 테스트 → 배포 → 검증의 워크플로
4. **표준 설정(Baseline)**
   - “운영 표준 설정 템플릿”을 정의하고 이를 벗어나는 값은 **drift**로 탐지
5. **자동화(IaC/Automation)**
   - Ansible, NETCONF/YANG, RESTCONF, 벤더 API 등으로 대량 설정 배포

### 3) 예제 시나리오: 데이터센터 VLAN 변경

**상황**
- 데이터센터에 스위치 200대, 서버 4,000대
- 신규 서비스 때문에 `VLAN 310`, `VLAN 311`을 모든 ToR 스위치에 추가해야 함
- 동시에 관련 ACL, QoS 정책도 함께 반영해야 한다

**구성 관리가 없을 때**

- 엔지니어들이 SSH로 각 스위치 접속 후 수동 설정
- 한두 곳에서 타이포가 나거나 ACL 한 줄이 빠져,
  특정 랙만 통신 문제 발생
- 나중에 장애가 나면, “어느 스위치가 표준과 다른지”를
  한 대씩 로그인해서 확인해야 한다

**구성 관리 도입 후**

- `switch-baseline-v5.conf` 템플릿에 VLAN/ACL/QoS 정의를 추가
- `group_vars/tor.yml` 같은 그룹 변수에 이번 변경 항목만 추가
- 자동화 도구(예: Ansible 플레이북)를 통해
  “ToR 그룹 전체에 일괄 반영 + 결과 수집”
- 각 스위치의 설정은 Git에 저장되어,
  “변경 전/후 diff”, “누가 언제 커밋”했는지 명확

### 4) 간단 코드 예: 설정 템플릿 적용(개념적인 예)

```yaml
# 예: Ansible 스타일의 매우 단순화된 스위치 VLAN 구성 플레이북

- hosts: tor_switches
  gather_facts: no
  vars:
    vlans:
      - { id: 310, name: "app-prod" }
      - { id: 311, name: "db-prod" }
  tasks:
    - name: Configure VLANs
      ios_config:
        lines:
          - "vlan {{ item.id }}"
          - " name {{ item.name }}"
        with_items: "{{ vlans }}"
```

이런 도구를 쓰면 “설정 그 자체”를 코드로 관리(Infrastructure as Code)할 수 있고,
구성 관리는 Git·CI 파이프라인과 결합되어 **자동 검증 + 롤백**까지 가능하다.

### 5) 구성 관리 품질 지표 예

구성 관리 역시 측정 가능한 지표로 관리할 수 있다.

- **구성 드리프트율(Configuration Drift Rate)**

  $$
  \text{Drift Rate} = \frac{\text{표준에서 벗어난 장비 수}}{\text{전체 장비 수}}
  $$

- **변경 성공률(Change Success Rate)**

  $$
  \text{Success Rate} = 1 - \frac{\text{변경 후 장애 건수}}{\text{총 변경 건수}}
  $$

- **평균 변경 리드타임(Change Lead Time)**
  요청 생성부터 실제 적용까지 걸린 평균 시간

운영 조직은 보통 “드리프트율 0% 유지”, “중요 변경은 리드타임 3일 이내” 같은 SLO를 세우고 관리한다.

---

## Fault Management

### 1) 개념

Fault Management는 **장애를 빠르게 발견·격리·복구**하고,
향후 재발을 막기 위해 분석·보고하는 활동을 말한다.

구체적으로는:

- **Detect**: SNMP 트랩, syslog, 메트릭, 헬스 체크로 이상 신호 탐지
- **Isolate**: 토폴로지·로그 분석으로 어떤 요소(링크, 장비, 모듈)가 문제인지 확인
- **Notify**: NOC 대시보드, 이메일, SMS, 티켓 시스템에 알림
- **Recover**: 재시작, 우회 경로 설정, 대체 장비로 페일오버 등
- **Document**: Root Cause Analysis(RCA), 재발 방지 대책 기록

### 2) MTTR, MTBF, 가용성

Fault Management에서 자주 쓰는 기준 지표는 다음과 같다.

- **MTBF(Mean Time Between Failures)** — 평균 고장 간격
- **MTTR(Mean Time To Repair)** — 평균 복구 시간
- **가용성(Availability)**

$$
\text{Availability} = \frac{\text{서비스 가동 시간}}{\text{총 시간}}
= \frac{\text{MTBF}}{\text{MTBF} + \text{MTTR}}
$$

**예제 계산**

- 어떤 코어 스위치의 MTBF = 2000시간
- MTTR = 1시간 이라고 하자.

$$
\text{Availability}
= \frac{2000}{2000 + 1}
\approx 0.9995
= 99.95\%
$$

연간 허용 다운타임은:

$$
\text{연간 시간} = 365 \times 24 = 8760 \text{시간}
$$

$$
\text{다운타임} \approx 8760 \times (1 - 0.9995) \approx 4.38 \text{시간/년}
$$

“99.95% SLA”가 어느 정도 다운타임을 의미하는지 감을 잡을 수 있다.

### 3) 예제 시나리오: 장애 탐지와 알림

**상황**

- ISP 코어망에 있는 라우터 R1–R4 중,
  R3–R4 사이의 광 링크가 광 모듈 불량으로 다운
- OSPF/IS-IS 라우팅은 대체 경로를 통해 즉시 수렴했지만,
  전체 경로에 지연이 약간 늘어나고,
  일부 VPN 고객은 짧은 시간 패킷 손실 경험

**Fault Management 흐름**

1. **실시간 탐지**
   - 인터페이스가 down → 라우터에서 SNMP 트랩 + syslog 생성
   - NMS가 트랩을 수신 → “링크 다운” 알람 생성
2. **알림**
   - NOC 대시보드에 Critical 빨간 알람
   - On-call 엔지니어에게 SMS, 티켓 자동 생성
3. **격리**
   - 토폴로지 상에서 R3–R4 링크만 장애 표시
   - 링크 양단 인터페이스의 에러 카운터, 광 레벨 등을 확인
4. **복구**
   - 현장 엔지니어 파견 → 광 모듈 교체 또는 패치 케이블 교체
   - 링크 up 후, 플랩이 없는지 몇 분간 모니터링
5. **분석**
   - MTTR, 장애 발생 시각과 탐지 시각 차이(Detection Time) 측정
   - 비슷한 장비/모듈에 예방적 교체가 필요한지 검토

### 4) 간단 예제: 핑 기반 헬스체크 스크립트(개념)

```python
import subprocess, time, smtplib

targets = ["192.0.2.1", "192.0.2.2", "192.0.2.3"]

def is_alive(ip):
    result = subprocess.run(
        ["ping", "-c", "2", "-W", "1", ip],
        stdout=subprocess.DEVNULL
    )
    return result.returncode == 0

while True:
    for ip in targets:
        if not is_alive(ip):
            print(f"[ALERT] Host {ip} unreachable")
            # 여기서 실제로는 메일, Slack, 티켓 시스템 연동
    time.sleep(30)
```

실제 상용 NMS는 이런 단순 핑을 넘어서,
SNMP/스트리밍 텔레메트리, BFD, 장비 자체 health 상태까지 종합적으로 모니터링한다.

---

## Performance Management

### 1) 지연·손실·지터·대역폭

성능 관리의 목표는 **서비스 수준(SLA)에 맞게 네트워크 품질을 유지**하는 것이다.
ITU-T는 IP 네트워크 품질 측정을 위해 지연, 지터, 패킷 손실률, 가용성 등을 정의하고, 측정·목표치 프레임워크를 제공한다.

주요 지표:

- **One-Way Delay (편도 지연)**
- **Round Trip Time (RTT)**
- **Packet Delay Variation (PDV, 지터)**
- **Packet Loss Ratio (PLR)**
- **Throughput / Goodput**
- **Availability, Path Unavailability**

예를 들어 ITU-T Y.1541, Y.1731 등에서는 이런 값들을 측정하고, 서비스 종류(음성, 비디오, 데이터)에 따라 허용 범위를 제안한다.

### 2) 활용 예: 링크 이용률 계산

인터페이스의 수신·송신 바이트 카운터를 이용해 이용률을 계산한다.

- 측정 간격: $$\Delta t$$ 초
- 이전/현재 바이트 카운터: $$C_{\text{prev}}, C_{\text{curr}}$$
- 링크 속도: $$R$$ (bps)

$$
\text{Utilization} =
\frac{(C_{\text{curr}} - C_{\text{prev}}) \times 8}{R \times \Delta t}
$$

**예제**

- 지난 5분(300초) 동안 바이트 카운터가 1,200,000,000 바이트 증가
- 링크 속도: 1 Gbps

$$
\text{Utilization} =
\frac{1{,}200{,}000{,}000 \times 8}{1{,}000{,}000{,}000 \times 300}
= \frac{9{,}600{,}000{,}000}{300{,}000{,}000{,}000}
\approx 0.032 = 3.2\%
$$

이 값을 시간대별로 저장하면, 피크 시간대 패턴을 분석하고 용량 증설 여부를 판단할 수 있다.

### 3) 예제 시나리오: SLA 모니터링

**상황**

- 클라우드 사업자가 고객에게 L3 VPN 서비스를 제공
- SLA: “한 달 기준, **지연 평균 20ms 이하**, **패킷 손실률 0.1% 이하**, **가용성 99.95% 이상**”

**Performance Management 활동**

1. **측정 인프라 구축**
   - 각 PoP에 측정용 프로브 배치
   - 정기적으로 유니캐스트·멀티캐스트 트래픽을 송신하여 RTT, PLR 측정
2. **메트릭 수집 및 저장**
   - 1분 단위 측정 → TSDB(time-series DB)에 저장
   - Y.1731 OAM 프레임이나 TWAMP 같은 프로토콜을 사용하면
     표준화된 형식으로 측정 가능
3. **알람 및 보고**
   - 지연/손실이 임계값을 초과하면 NOC에 경고
   - 고객에게 월간 SLA 리포트 제공
4. **역방향 분석**
   - 특정 시간대에만 지연이 높다면,
     그 시간대의 플로우 기록(NetFlow/IPFIX)을 분석해
     특정 애플리케이션이나 DDoS 트래픽이 원인인지 파악

### 4) 코드 예: SNMP 카운터 기반 이용률 계산(개념)

```python
# 실제 SNMP 라이브러리 대신, 카운터 값이 있다고 가정한 간단 예제

import time

if_octets_prev = 1000000000  # 이전 읽어온 바이트
if_octets_curr = 1012000000  # 현재 읽어온 바이트
delta_t = 300                # 5분
link_speed = 1_000_000_000   # 1 Gbps

utilization = (if_octets_curr - if_octets_prev) * 8 / (link_speed * delta_t)
print(f"Utilization: {utilization * 100:.2f}%")
```

실제 환경에서는 `pysnmp` 등으로 `ifHCInOctets`/`ifHCOutOctets`를 읽어오고,
TSDB에 저장해 대시보드(Grafana 등)로 시각화한다.

---

## Security Management

### 1) 개념과 역할

Security Management는 네트워크와 그 위에서 동작하는 서비스가
**기밀성(Confidentiality), 무결성(Integrity), 가용성(Availability)** 을 유지하도록 관리하는 영역이다.

주요 기능:

- 장비·사용자·API에 대한 **인증/권한 부여/계정관리(AAA)**
- 방화벽, IDS/IPS, WAF, NAC, VPN 등의 정책 구성·모니터링
- 취약점 관리, 패치 관리
- 로그·이벤트 상관분석(SIEM), 침해사고 대응

NIST SP 800-137은 이런 활동을 **정보보안 지속 모니터링(ISCM)** 으로 묶어,
조직이 위협·취약점·통제 효과성을 지속적으로 파악하도록 가이드한다.

또한 NIST SP 800-128은 구성 관리와 보안을 결합한
**Security-focused Configuration Management(SecCM)** 를 다룬다.

최근에는 “Security Configuration Management”라는 이름으로
**지속적 구성 검사·drift 탐지·자동 교정**을 강조하는 연구·제품도 많다.

### 2) 예제 시나리오: NAC 기반 접속 제어

**상황**

- 사내 Wi-Fi·유선 네트워크에 BYOD 기기가 자주 접속
- 악성코드 감염된 노트북이 내부망에 직접 들어오는 것을 막고 싶음

**Security Management 설계**

1. **인증 및 장비 프로파일링**
   - 802.1X/EAP + RADIUS 서버를 통해 사용자/디바이스 인증
   - NAC(Network Access Control) 시스템이
     OS 버전, 패치 수준, 안티바이러스 상태 등을 점검
2. **정책**
   - 정책 예:
     - “사내 관리가 안 된 개인 장비는 인터넷 전용 VLAN으로만”
     - “패치가 일정 이상 오래된 장비는 격리망으로”
3. **모니터링**
   - NAC 로그를 SIEM으로 보내,
     특정 사용자 계정에서 비정상 접속 시도를 탐지
4. **조치**
   - 이상 징후 발생 시 자동으로 포트 차단 또는 격리 VLAN으로 이동

이 전체 파이프라인을 구성 관리(802.1X 설정, RADIUS 정책)·fault/performance 관리와 연결해야
잘못된 정책으로 전체 직원이 인터넷을 못 쓰는 “보안 장애”를 막을 수 있다.

### 3) 보안 관리 지표 예

- **Mean Time To Detect (MTTD)**
  보안 이벤트 발생부터 탐지까지 걸리는 평균 시간
- **Mean Time To Respond (MTTR, 보안 관점)**
  탐지부터 효과적인 대응(차단, 패치 완료)까지 걸리는 시간
- **취약점 잔존 수(Outstanding Vulnerabilities)**
  CVSS 기준 High/CRITICAL 취약점 수
- **정책 위반 세션 비율**

  $$
  \text{Policy Violation Rate} =
  \frac{\text{정책 위반이 발생한 세션 수}}{\text{전체 세션 수}}
  $$

---

## Accounting Management

### 1) 개념

Accounting Management는 네트워크 자원의 사용량을 측정·기록하고,
**과금(billing)·쇼백(showback)·차지백(chargeback)**, 그리고 용량 계획에 활용하는 영역이다.

전통 통신사에서는:

- 음성 통화 시간, 데이터 사용량, 메시지 수 등 기반의 **요금 청구**

기업/캠퍼스 환경에서는:

- 부서별/서비스별 사용량을 측정해,
  IT 비용을 적절히 배분하거나,
  “누가 네트워크를 많이 쓰는지”를 파악하는 용도로 사용

오늘날 클라우드에서도 비슷한 개념이 적용된다(트래픽 GB당 비용, egress 요금 등).

### 2) 데이터 소스

Accounting을 위해 수집할 수 있는 데이터는 많다.

- **RADIUS Accounting**
  - 사용자가 VPN 또는 Wi-Fi에 접속/종료할 때 세션 시간, 전송 바이트 수 등을 RADIUS 서버에 기록
- **NetFlow/IPFIX/sFlow**
  - 플로우 단위(5-tuple)로 바이트/패킷 수를 기록
  - 애플리케이션별, 목적지별, 사용자별 트래픽 분석
- **방화벽·프록시 로그**
  - 도메인/URL/애플리케이션 기준 사용량 집계
- **클라우드 제공업체의 사용량 리포트**
  - egress 트래픽, VPN, Direct Connect 등 항목별

### 3) 예제 시나리오: 부서별 네트워크 비용 배분

**상황**

- 대기업 본사에서 IT 부서는
  “연간 네트워크 비용을 부서별 사용자 수가 아니라,
   실제 트래픽 사용량에 따라 배분하고 싶다”
- 부서마다 VLAN/VRF가 달리 구성되어 있음

**Accounting Management 구현**

1. 모든 코어라우터에서 NetFlow/IPFIX를 활성화해,
   각 VRF/VLAN별 트래픽 플로우를 수집
2. 수집 서버에서 플로우 데이터를
   **부서 코드**와 매핑하여 월별 집계
3. 각 부서별로 “월간 평균 대역폭 사용량, 피크 사용률, 총 전송 바이트”를 리포트
4. 회계 시스템과 연동해, 네트워크 비용을 부분적으로 차지백

### 4) 95th Percentile Billing 예

통신사·데이터센터에서 자주 쓰는 과금 방식은
“**5분 단위 트래픽 측정 → 한 달간 측정값 중 상위 5% 제거 → 나머지 중 최대값으로 과금**” 이다.

측정값이 $$x_1, x_2, \dots, x_n$$ (Mbps)라고 할 때:

- 오름차순 정렬하여 $$x_{(1)} \le x_{(2)} \le \dots \le x_{(n)}$$
- 상위 5% 값은 제외 → 인덱스 $$k = \lfloor 0.95 \times n \rfloor$$
- 95th percentile:

$$
P_{95} = x_{(k)}
$$

이 값에 약정 단가를 곱해 해당 기간의 트래픽 비용을 산정한다.

### 5) 간단 코드 예: 95번째 퍼센타일 계산(개념)

```python
def percentile95(samples):
    samples = sorted(samples)
    k = int(len(samples) * 0.95) - 1
    k = max(0, min(k, len(samples)-1))
    return samples[k]

values = [100, 120, 80, 90, 200, 300, 150, 400]  # Mbps 예시
print("95th percentile:", percentile95(values), "Mbps")
```

실제 환경에서는 수천·수만 개의 샘플이 있고,
각데이터는 5분 또는 1분 단위 인터페이스 이용률이다.

---

## 정리

27.1에서는 FCAPS 모델을 따라 **구성·장애·성능·보안·회계** 다섯 영역을 개념·지표·예제로 살펴봤다.

- **Configuration Management**: 장비·설정·변경을 표준화·자동화하여
  안정된 네트워크를 유지하는 기반
- **Fault Management**: 장애를 빠르게 발견·격리·복구해 SLA를 보장
- **Performance Management**: 지연·손실·대역폭을 상시 모니터링해
  용량·품질을 최적화
- **Security Management**: AAA·정책·취약점·로그를 관리하여
  네트워크를 안전하게 유지
- **Accounting Management**: 사용량을 측정해 과금·비용 배분·용량계획에 활용
