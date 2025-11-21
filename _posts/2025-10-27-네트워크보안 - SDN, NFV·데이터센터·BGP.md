---
layout: post
title: 네트워크보안 - SDN/NFV·데이터센터·BGP
date: 2025-10-27 20:25:23 +0900
category: 네트워크보안
---
# SDN/NFV·데이터센터·BGP

> 목표: **SDN/NFV** 환경에서의 위협과 보호 포인트를 정리하고,
> **BGP 하이재킹·루트 리크**를 막는 **RPKI/ROA·필터링·세션 하드닝**을 체화한다.
> DC 패브릭(EVPN/VXLAN)의 보안 지점(ARP 억제, Type-2/5 경계, MAC Move 절연 등)을 짚고,
> 마지막으로 **라우트 필터/Max-Prefix/RTBH**를 **동작하는 예제**로 실습한다.
> (벤더/버전 차이는 존재. 아래 예제는 **FRRouting**, **BIRD/GoBGP**, **JunOS/IOS-XR** 풍의 구성 스니펫을 혼합해 설명.)

---

## SDN 위협/보호 포인트

### 위협 표면

- **컨트롤 플레인 집중화(Controller)**
  - DoS/리소스 고갈, 권한 탈취(남용), 앱(북향 API) 취약점 → **Tenant 전체 영향**.
- **Southbound 채널(OpenFlow/OVSDB/NETCONF/gNMI)**
  - 평문/TLS 미사용, 약한 인증 → **컨트롤 채널 탈취/주입** 위험.
- **멀티테넌시/오버레이(VXLAN/Geneve)**
  - VNI/VRF 오배치, 라우트 타겟 오염 → **다른 테넌트로 우회**.
- **가상 네트워크 함수(NFV)**
  - 공용 호스트 상의 성능·격리 문제, 패킷 사본/미러링의 **데이터 유출**.
- **정책/오케스트레이션**
  - IaC/컨트롤러 정책의 “잘못된 기본값/광역 허용”, 승인 없는 변경.

### 보호 포인트(체크리스트)

- **컨트롤러 하드닝**
  - 전용 관리망/Out-of-Band, **RBAC/Just-in-time** 접근, **감사 로그/서명된 변경**.
  - **HA**(컨트롤러 이중화) + **Rate-limit/큐 보호**(Southbound 세션 수 제한).
- **Southbound 보안**
  - **mTLS**(클라이언트/서버 상호 인증) + **TLS 1.2/1.3** 강제, **TLS 핀닝**(가능 시).
  - 장비 등록 시 **부팅스트랩 PKI**(온보딩 토큰 만료/1회성).
- **폴리시 가드레일**
  - 기본 **Deny** + 최소 허용, **테넌트/네임스페이스 분리**, 라우트 타겟/VRF **화이트리스트**.
  - **변경 사전 검증**: 시뮬레이터(What-if), **정책 유닛 테스트**.
- **텔레메트리/감사**
  - **gNMI/GNOI** 체크섬/버전, **BMP/NetFlow/IPFIX**로 외부 수집,
    컨트롤러의 북향 API 호출은 **서명/승인 워크플로우**.
- **NFV 호스트 격리**
  - **SR-IOV/DPDK** 시 NIC/IOMMU 격리, 가상 스위치(OVS/eBPF) 룰 서명,
  - **CPU pinning/NUMA**와 **거친 테넌트 간 co-location 제한**(성능·측면채널 완화).

**Open vSwitch(OVSDB + TLS) 예시**
```bash
# CA/서버/클라이언트 키 준비됨 가정

ovs-vsctl set-ssl /etc/ssl/ovs/client.key /etc/ssl/ovs/client.crt /etc/ssl/ovs/ca.crt
ovs-vsctl set-manager pssl:6640:0.0.0.0
# 컨트롤러(OVSDB 서버)는 ptcp 대신 pssl

ovs-vsctl set-manager pssl:6640
```

**OpenFlow 채널 TLS(개념)**
```bash
ovs-vsctl set-controller br0 ssl:ctrl.example.net:6653
ovs-vsctl set-controller connection-mode=out-of-band
```

---

## BGP 하이재킹·루트 리크, RPKI/ROA

### 위협 시나리오

- **하이재킹(Hijack)**: 다른 AS가 **귀사의 프리픽스**(더 특정/동일 길이)를 기원(Origin) → 트래픽 납치.
- **루트 리크(Route Leak)**: 타 AS에서 받은 경로를 **부적절한 방향**으로 재전파(정책 위반).
- **AS-PATH 위조**: 더 짧은 AS-PATH 꾸미기, **ORIGIN 조작**.
- **모범 사례 미적용**: 기본 Accept-all, 마티안(Bogon) 수용, **Max-Prefix 없이 수천 경로 유입**.

### 방어의 큰 축

1) **세션 하드닝**:
   - **MD5/TCP-AO** 비밀키, **GTSM**(TTL Security, TTL=255), **Graceful Restart/LLGR** 튜닝.
2) **유입 필터(Import)**:
   - **Prefix-List/AS-PATH** 필터, **Bogon(Reserved/RFC1918/0/8/127/8/…)** 차단,
   - **Max-Prefix** 제한(임계 초과 시 drop 또는 shutdown).
3) **유출 필터(Export)**:
   - **Your ASN만 ORIGIN**, **NO_EXPORT/COMMUNITY 가드**, **remove-private-as**.
4) **RPKI/ROA 검증**:
   - **Valid/Invalid/NotFound** 정책, **Invalid = Drop**(점진 전환), **RTR 캐시** 이중화.
5) **모니터링/BMP**:
   - BMP/Streaming Telemetry로 피어별 이상(경로 급증, ORIGIN 변화) 경보.

### FRRouting: 세션 하드닝

```text
router bgp 65010
 bgp router-id 10.0.0.1
 neighbor 203.0.113.1 remote-as 64501
 neighbor 203.0.113.1 description UPSTREAM-1
 neighbor 203.0.113.1 password 7 <md5_secret>        ! MD5
 neighbor 203.0.113.1 ttl-security hops 1             ! GTSM
 neighbor 203.0.113.1 timers 30 90
 ! 선택: graceful-restart/llgr 정책
 bgp graceful-restart
```

### Prefix/Bogon 필터(입력)

```text
ip prefix-list BOGONS seq 5 deny 0.0.0.0/0
ip prefix-list BOGONS seq 10 deny 10.0.0.0/8 le 32
ip prefix-list BOGONS seq 20 deny 172.16.0.0/12 le 32
ip prefix-list BOGONS seq 30 deny 192.168.0.0/16 le 32
ip prefix-list BOGONS seq 40 deny 100.64.0.0/10 le 32
ip prefix-list BOGONS seq 50 deny 127.0.0.0/8 le 32
ip prefix-list BOGONS seq 60 deny 169.254.0.0/16 le 32
ip prefix-list BOGONS seq 999 permit 0.0.0.0/0 le 32

route-map IN-FILTER deny 5
 match ip address prefix-list BOGONS
!
route-map IN-FILTER permit 100
 set local-preference 100

router bgp 65010
 neighbor 203.0.113.1 route-map IN-FILTER in
```

### Max-Prefix

```text
router bgp 65010
 neighbor 203.0.113.1 maximum-prefix 200000 90 restart 10
! 20만 경로 임계, 90% 경고, 10분 후 세션 재시도
```

### RPKI/ROA(RTR 캐시와 연동)

**frr.conf (RPKI)**
```text
rpki
 rpki cache 198.51.100.10 3323 preference 1     ! RPKI-RTR 캐시 1
 rpki cache 198.51.100.11 3323 preference 2     ! 캐시 2
 exit
!
router bgp 65010
 bgp rpki server 198.51.100.10 3323
 bgp rpki server 198.51.100.11 3323
 bgp bestpath prefix-validate                     ! ROV 활성
 bgp bestpath prefix-validate allow-invalid-as    ! 점진 전환 시 옵션
```

**정책(Invalid Drop)**
```text
route-map RPKI-IMPORT deny 10
 match rpki invalid
!
route-map RPKI-IMPORT permit 100
!
router bgp 65010
 neighbor 203.0.113.1 route-map RPKI-IMPORT in
```

> 운영 팁: 초기엔 `invalid → lower local-pref`로 경고 모드 → **충분한 관찰 후 Drop** 전환.

### AS-PATH/Export 衛生

```text
ip as-path access-list ASPATH-BAD permit _0_         ! 잘못된/빈 AS, 예시
ip as-path access-list ASPATH-BAD permit _23456_     ! 예약 AS
!
route-map IN-ASPATH deny 10
 match as-path ASPATH-BAD
!
router bgp 65010
 neighbor 203.0.113.1 route-map IN-ASPATH in

! Export 시 사설 AS 제거
router bgp 65010
 neighbor 203.0.113.1 remove-private-as
 neighbor 203.0.113.1 send-community both
```

### Route Leak 방지(기본 거부, RFC 8212 정신)

- 벤더별로 **BGP 기본 임포트/익스포트 정책을 명시적 허용만**으로 변경.
- **EBGP/IBGP**별 **Policy 존재하지 않으면 광고 금지**.

**JunOS(개념)**
```text
policy-options {
  policy-statement EBGP-IMPORT { then reject; }
  policy-statement EBGP-EXPORT { then reject; }
}
protocols {
  bgp {
    group EBGP {
      type external;
      import [ EBGP-IMPORT ];
      export [ EBGP-EXPORT ];
    }
  }
}
```

---

## EVPN/VXLAN 패브릭 보안

### 배경

- **EVPN + VXLAN**: L2/IRB 확장, **Type-2(MAC/IP)**, **Type-5(IP Prefix)** 경로로 컨트롤 플레인 광고.
- 보안 목표:
  - **테넌트 격리**(VNI/VRF/RT), **게이트웨이 일관성**(Distributed Anycast GW)
  - **ARP/ND 억제**로 L2 노이즈 감소, **MAC 이동/루프** 감지 및 완화
  - **Type-5 경계**에서 **정책 라우팅/필터**로 L3 확산 통제

### 보안 포인트

- **RT/RT-Filter 상한**: 허용된 라우트타겟만 수용(다른 테넌트 RT 거부).
- **MAC Move Dampening**: 짧은 시간 다수 이동 → **도난/스푸핑 의심** → 페널티.
- **ARP/ND Proxy**: **Type-2** 테이블/프록시로 **그레어/스푸핑 완화**(DAI 유사).
- **IRB 정책**: L3 경계에서 **ACL/uRPF**(가능 시), **분산 Anycast GW**의 **ARP suppression**.

**Arista/EVPN(개념)**
```text
router bgp 65050
 neighbor EVPN-PEERS peer-group
 neighbor EVPN-PEERS update-source Loopback0
 neighbor EVPN-PEERS ebgp-multihop 2
 address-family evpn
  neighbor EVPN-PEERS activate
  neighbor EVPN-PEERS route-target filter
  advertise-all-vni
!
evpn mac-learning l2evpn
evpn mac-mobility dampening half-life 15 reuse 300 suppress 600 max-suppress 900
```

**JunOS: Type-5 수용 제한(개념)**
```text
policy-options {
  policy-statement EVPN-TYPE5-IN {
    term ALLOW-SUBNETS {
      from route-filter 10.10.0.0/16 orlonger;
      then accept;
    }
    then reject;
  }
}
protocols {
  evpn {
    encapsulation vxlan;
    ip-prefix-routes {
      advertise direct-nh;
      import EVPN-TYPE5-IN;   # 승인된 prefix만 수용
    }
  }
}
```

### DHCP 보안/게이트웨이 연계

- **DHCP Relay** 시 Option-82/Client-ID 검증, **Snooping** 유사 스택(벤더 종속).
- **IRB/Distributed GW**에서 **RA Guard/ND Inspection**(IPv6) 사용.

### 데이터플레인 안전장치

- **Storm Control**(Bcast/UnknownUcast/Multicast), **MLAG** fast-converge + loop-protect.
- **Control Plane Policing(CoPP)**: BGP/EVPN 세션, OSPF, ICMP Rate-limit.

---

## 실습: 라우트 필터/Max-Prefix/RTBH

> 랩 목표: FRRouting 두 대(R1=자사, R2=업스트림)로 **필터/Max-Prefix/RTBH**를 구성하고,
> 추가로 **RPKI** 캐시를 붙여 **Invalid Drop**을 시험한다. (Docker/컨테이너 기반 권장)

### 토폴로지

```
        (RTR-CACHE)
         198.51.100.10:3323
               |
[ R1 (AS65010) ]－－eBGP－－[ R2 (AS64501) ]
   10.0.0.1                         10.0.0.2
   광고: 203.0.113.0/24
```

### – 기본 보안 정책

```text
router bgp 64501
 bgp router-id 10.0.0.2
 neighbor 10.0.0.1 remote-as 65010
 neighbor 10.0.0.1 password 7 <md5_secret>
 neighbor 10.0.0.1 ttl-security hops 1
 neighbor 10.0.0.1 maximum-prefix 100 90 warning-only
 !
 ip prefix-list CUST-IN seq 5 permit 203.0.113.0/24
 ip prefix-list CUST-IN seq 100 deny 0.0.0.0/0 le 32
 route-map CUST-IN permit 10
  match ip address prefix-list CUST-IN
 !
 neighbor 10.0.0.1 route-map CUST-IN in

! RPKI 적용(있다면)
rpki
 rpki cache 198.51.100.10 3323
exit
router bgp 64501
 bgp bestpath prefix-validate
 route-map RPKI-IN deny 10
  match rpki invalid
 route-map RPKI-IN permit 100
 neighbor 10.0.0.1 route-map RPKI-IN in
```

- 의미: **고객이 광고 가능한 프리픽스만 화이트리스트**, **Invalid ROA는 Drop**, **Max-Prefix** 경고.

### – Export 衛生 & RTBH

```text
router bgp 65010
 bgp router-id 10.0.0.1
 neighbor 10.0.0.2 remote-as 64501
 neighbor 10.0.0.2 password 7 <md5_secret>
 neighbor 10.0.0.2 ttl-security hops 1
 !
 ip prefix-list MY-OUT seq 5 permit 203.0.113.0/24
 ip prefix-list MY-OUT seq 10 deny 0.0.0.0/0 le 32
 route-map MY-EXPORT permit 10
  match ip address prefix-list MY-OUT
 !
 neighbor 10.0.0.2 route-map MY-EXPORT out

! 사설 AS 제거
 neighbor 10.0.0.2 remove-private-as

! RTBH: 192.0.2.1로 유도(Null0 static)
ip route 192.0.2.1/32 Null0
route-map RTBH permit 10
 set community 65535:666 additive
 set ip next-hop 192.0.2.1
!
! 공격 타깃 /32에 RTBH 적용(정적 prepend용 route-map과 별개로)
ip prefix-list BLACKHOLE seq 5 permit 203.0.113.123/32
route-map RTBH-MATCH permit 10
 match ip address prefix-list BLACKHOLE
 set community 65535:666 additive
 set ip next-hop 192.0.2.1
!
! 필요 시 특정 네이버에게만 RTBH export
neighbor 10.0.0.2 route-map RTBH-MATCH out
```

**운영 절차(모의)**
1) 공격 타깃 식별: `203.0.113.123/32`
2) R1에서 **BLACKHOLE 프리픽스** 광고(상기 구성은 항상 out 매칭 방식. 운영에선 스크립트/전용 VRF 권장).
3) R2/업스트림이 커뮤니티/넥스트홉 규칙으로 **Null0** 처리를 적용.
4) 공격 종료 후 철회.

> 주의: RTBH는 **마지막 수단**. 스크러빙 센터/리다이렉션(FlowSpec/미러)와 병행 검토.

### 하이재킹 모의(실험)와 차단

- **상황**: R1이 실수로 `198.51.100.0/24`를 광고.
- **R2 정책**: `CUST-IN` 화이트리스트에 없으므로 **거부**.
- **관찰**: `show bgp ipv4 unicast neighbors 10.0.0.1 routes`에 **수용 안 됨**.

**검증 명령(FRR)**
```text
show bgp summary
show bgp ipv4 unicast
show bgp ipv4 unicast neighbors 10.0.0.1 advertised-routes
show bgp ipv4 unicast neighbors 10.0.0.1 routes
show rpki
```

### BIRD/GoBGP 대안 스니펫

**BIRD v2: RPKI**
```conf
roa4 table roa_v4;
protocol rpki rpki1 {
  roa4 { table roa_v4; };
  remote "198.51.100.10" port 3323;
}
filter rpki_in {
  if roa_check(roa_v4, net, bgp_path.last) = ROA_INVALID then reject;
  accept;
}
protocol bgp up1 {
  neighbor 10.0.0.2 as 64501;
  import filter rpki_in;
  export filter my_export;
}
```

**GoBGP: Max-Prefix/Policy(개념)**
```toml
[[neighbors]]
  [neighbors.config]
    neighbor-address = "10.0.0.2"
    peer-as = 64501
  [neighbors.transport.config]
    passive-mode = false
  [neighbors.timers.config]
    hold-time = 90
  [neighbors.afi-safis.config]
    afi-safi-name = "ipv4-unicast"
  [neighbors.apply-policy.config]
    import-policy-list = ["IMPORT-IN"]
    export-policy-list = ["EXPORT-OUT"]
```

---

## 보너스: BMP/텔레메트리 & 탐지 룰 아이디어

**BMP 수집기(예: OpenBMP/PMACCT)로 탐지**
- 피어별 **광고 경로 수 급증**(>X%)
- **ORIGIN-AS 변경**(우리 프리픽스의 원발AS가 타인)
- **더 특정(/25~)** 경로 출현(허용 범위 밖)
- **Invalid(ROA) 비율 증가**

**PromQL(개념)**
```promql
increase(bgp_announcements_total{peer="10.0.0.2"}[5m]) > 100
```

---

## 트러블슈팅 빠른 표

| 증상 | 가능 원인 | 조치 |
|---|---|---|
| 세션은 오르나 경로 無 | Import 정책 전부 reject | Route-map/Prefix-list 매칭 재검 |
| 경로 급증 후 다운 | Max-Prefix 초과 | `maximum-prefix` 임계·동작 확인, 피어 협의 |
| 우리 프리픽스 Invalid | ROA 미등록/오류 | RIR 포털에서 **ROA 생성/갱신**, RTR 캐시 최신화 |
| EVPN 테넌트 간 통신됨 | RT Filter 누락 | EVPN route-target filter/VRF 바인딩 재검 |
| MAC flapping | Loop/스푸핑 | MAC Move Dampening, 포트 보안/Storm Control |

---

## 운영 체크리스트(요점)

- [ ] **MD5/TCP-AO, GTSM**로 BGP 세션 하드닝
- [ ] **Import/Export 기본 거부 + 화이트리스트**(**RFC 8212 정신**)
- [ ] **Max-Prefix** 임계 & 경보, **Bogon 필터** 상시
- [ ] **RPKI ROV**(Invalid Drop) 단계적 도입, **RTR 캐시 이중화**
- [ ] **remove-private-as**, 커뮤니티 가드, **NO_EXPORT** 규율
- [ ] **BMP/Flow/IPFIX**로 가시성, **ORIGIN 변화/경로 급증** 탐지 룰
- [ ] EVPN/VXLAN: **RT Filter, ARP/ND Proxy, MAC Move Dampening, CoPP**
- [ ] SDN/NFV: **mTLS Southbound, RBAC+감사, 정책 가드레일, 호스트 격리**

---

## 마무리

- **SDN/NFV**는 **컨트롤 채널 보안**과 **정책 기본 거부**가 심장이다.
- **BGP 보안**은 **세션 하드닝 + 필터 + RPKI + 관측**의 합으로 이뤄진다.
- **EVPN/VXLAN**은 **RT/정책 경계**와 **L2 소음 억제**로 테넌트 안전을 유지한다.
- 실습의 **필터/Max-Prefix/RTBH**는 “**사고를 퍼지게 하지 않는 최저선**”이다.
  여기에 **RPKI/ROA**와 **BMP 기반 탐지**를 더해 **방어·가시성·회복력**을 동시에 올려라.
