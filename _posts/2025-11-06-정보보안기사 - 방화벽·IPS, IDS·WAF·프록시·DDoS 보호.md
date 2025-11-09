---
layout: post
title: 정보보안기사 - 방화벽·IPS/IDS·WAF·프록시·DDoS 보호
date: 2025-11-06 21:25:23 +0900
category: 정보보안기사
---
# 방화벽·IPS/IDS·WAF·프록시·DDoS 보호

## 0) 전체 아키텍처 한눈에 보기

```
[ Internet ]  ──(BGP/Anycast/CDN/Cloud DDoS)── [ Edge Router ]
      │                                   │
      └───▶  L3/4 DDoS Scrubbing(업체/자체) │
                                            └── CoPP, uRPF, Edge ACL(BCP38)
                                                │
                                          [ NGFW (Zones) ]
                                         /       |        \
                                        /        |         \
                                 [Reverse Proxy/WAF]   [IPS/IDS TAP/Inline]
                                        │
                                  [App Tier (L7)]  ← WAF 룰/RateLimit/봇관리
                                        │
                                 [DB/Stateful Tier] ← 마이크로세그먼트(ACL)
                                        │
                                 [Logging/SIEM/NDR] ← EVE(JSON)/NetFlow/sFlow
```

- **핵심 계층**:  
  L3/4(방화벽·uRPF·CoPP) → IDS/IPS(서명/행위) → WAF(L7) → 프록시(Forward/Reverse) → DDoS(네트워크/애플리케이션)  
- **철학**: “**드롭은 최대한 앞단에서, 정밀 검사는 뒤에서**”, “**허용은 최소권한(Whitelist)**”.

---

# 1) 방화벽(Firewall) — L3/4 기반의 **기본 방어선**

## 1.1 개념·유형
- **Stateless** vs **Stateful**: 연결 추적(conntrack) 유무
- **NGFW(차세대)**: 앱 인지/사용자 인지/IPS/WAF 기능 통합(벤더 종속)
- **Zone 기반 설계**: `Untrust(외부)`, `DMZ`, `App`, `DB`, `Mgmt`  
  → **Zone-to-Zone 정책**으로 허용 경로 명확화

## 1.2 리눅스 경계 방화벽 예시(nftables, Zone 기반)

```bash
# 1) 테이블/체인
sudo nft add table inet edge
sudo nft add chain inet edge input   { type filter hook input priority 0 ; policy drop ; }
sudo nft add chain inet edge forward { type filter hook forward priority 0 ; policy drop ; }
sudo nft add chain inet edge output  { type filter hook output priority 0 ; policy drop ; }
# 2) 공통 허용
sudo nft add rule inet edge input ct state established,related accept
sudo nft add rule inet edge input iif lo accept
# 3) 관리(SSH) 화이트리스트(예: 사무실 고정IP)
sudo nft add rule inet edge input ip saddr 203.0.113.10 tcp dport 22 ct state new accept
# 4) 공개서비스(HTTPS만)
sudo nft add rule inet edge input tcp dport 443 ct state new accept
# 5) 기본 드롭 로깅(속도 제한)
sudo nft add rule inet edge input limit rate 10/second counter log prefix "EDGE_DROP " drop
```

> **팁**:  
> - `ipset`(nft의 `set`)으로 국가/ASN 블록, 악성 IP 레퓨테이션 목록을 별도 set으로 관리  
> - 프로덕션은 **정책=Drop 기본 + 허용만 명시(Whitelist)** 가 안전

### Conntrack 및 성능 커널 튜닝(발췌)
```bash
# 연결 추적 테이블 크기(트래픽 패턴/메모리 고려)
sudo sysctl -w net.netfilter.nf_conntrack_max=1048576
# SYN Cookies (L4 SYN Flood 대비)
sudo sysctl -w net.ipv4.tcp_syncookies=1
```

## 1.3 클라우드 기본: NACL vs 보안그룹(예: AWS)
- **NACL**: 서브넷 경계, **Stateless**, In/Out **모두** 허용 필요
- **보안그룹(SG)**: ENI/인스턴스에 적용, **Stateful**, In만 정의하면 Out은 응답 허용
- 원칙: **SG 최소권한 → NACL은 coarse-grained 차단**(예: 외부 인바운드 전부 차단)

---

# 2) IDS/IPS — **탐지/차단**의 엔진

## 2.1 개요
- **IDS**: 수동 탐지(미러/TAP), 알림·로그 중심  
- **IPS**: 인라인 차단(허용/드롭), 레이턴시·안정성 설계 중요
- 탐지 방식: **서명(Signature)** + **행위/통계(Anomaly/Threshold)**

## 2.2 배치 토폴로지
- **Inline**: Edge ↔ Core 사이(고가용 이중화, Bypass NIC 필요)  
- **Tap**: SPAN/TAP 포트, **제어영향 0**(대신 직접 차단은 불가 → 방화벽 연동)

## 2.3 Suricata(오픈소스 IPS/IDS) — 빠른 시작(IDS 모드)

```bash
# 1) 패키지 설치(배포판에 따라 상이)
sudo apt-get install -y suricata
# 2) 인터페이스 설정 (suricata.yaml에서 'af-packet' or pcap)
# 3) 룰셋 취득(ET Open 등)
sudo suricata-update
# 4) 실행
sudo systemctl enable --now suricata
# 5) 로그(EVE JSON)
tail -f /var/log/suricata/eve.json
```

### 간단 룰(예시): DNS 대량 쿼리(비정상 행위 감지)
```text
alert dns any any -> any any (msg:"POSSIBLE DNS flood (rate)"; dns.query; threshold: type both, track by_src, count 100, seconds 1; classtype:attempted-dos; sid:100001; rev:1;)
```

> **운영 팁**  
> - 초기에 **IDS 모드**로 배치 → 오탐/튜닝 → 중요 트래픽에 한해 **IPS 차단 정책** 점진 적용  
> - EVE JSON을 **SIEM(예: OpenSearch/Elasticsearch)** 으로 집계, 룰별 FP/FN 모니터링

## 2.4 Zeek(행위/메타 로그) — 보완적 활용
- 프로토콜 파서 기반 **풍부한 메타 로그**(http.log, ssl.log, dns.log 등)
- “**무엇이 얼마나 자주/오래**” 같은 행태 분석에 강점 (WAF/방화벽 룰의 맹점 보완)

---

# 3) WAF(Web Application Firewall) — L7 보호막

## 3.1 전략
- **음성(Allow-List, Positive Security)**: API 스키마/메서드/경로/헤더 화이트리스트  
- **음성 + 서명(OWASP CRS)** 결합 → 가시성과 보안의 균형
- **가벼운 레이트리밋/챌린지**로 봇/스크래핑 억제

## 3.2 NGINX + ModSecurity(Open Source) + OWASP CRS 예제

### 설치/로드(발췌)
```nginx
# nginx.conf
load_module modules/ngx_http_modsecurity_module.so;

http {
  modsecurity on;
  modsecurity_rules_file /etc/nginx/modsec/main.conf;
  ...
}
```

`/etc/nginx/modsec/main.conf`:
```apache
# 코어 설정
Include /usr/local/owasp-modsecurity-crs/crs-setup.conf
Include /usr/local/owasp-modsecurity-crs/rules/*.conf

# 로깅 최소화/샘플링(성능/가독성 고려)
SecRuleEngine On
SecDebugLogLevel 0
SecAuditEngine RelevantOnly
SecAuditLogParts ABIJDEFHZ
```

### L7 화이트리스트(Positive) 샘플
```apache
# /api/v1/orders 는 POST만, JSON만 허용
SecRule REQUEST_URI "@beginsWith /api/v1/orders" "id:200001,phase:1,pass,ctl:ruleEngine=DetectionOnly"
SecRule REQUEST_METHOD "!@streq POST" "id:200002,phase:1,deny,status:405,log,msg:'Method Not Allowed'"
SecRule REQUEST_HEADERS:Content-Type "!@streq application/json" "id:200003,phase:1,deny,status:415,log,msg:'Unsupported Media Type'"
```

> **튜닝 절차**:  
> 1) `DetectionOnly`(탐지 모드) → 2) FP 분석/화이트리스트 보완 → 3) 중요 엔드포인트 차단 On → 4) 정기 룰업데이트/성능 점검

### NGINX 단 레이트 리밋(간단 L7 보호)
```nginx
http {
  limit_req_zone $binary_remote_addr zone=req_per_ip:10m rate=5r/s;
  server {
    location /api/ {
      limit_req zone=req_per_ip burst=20 nodelay;
      proxy_pass http://app:8080;
    }
  }
}
```

---

# 4) 프록시(Proxy) — Forward/Reverse로 **제어면/데이터면** 강화

## 4.1 Forward Proxy (사내 → 인터넷)
- 목적: **출구 통제(Egress)**, 캐싱, DLP/콘텐츠 필터, 사용자 인증(AD/SSO)
- HTTPS는 **CONNECT 터널** 기반 → 필요 시 **합법/정책** 범위 내 TLS 가시화 고려(주의: 개인정보/규정 준수)

### Squid 간단 예제
```conf
# squid.conf (요지)
http_port 3128
acl corpnet src 10.0.0.0/8
http_access allow corpnet
http_access deny all

# 특정 도메인/카테고리 차단(블랙리스트 파일 연동 가능)
acl badsites dstdomain .malicious.example .tracker.example
http_access deny badsites

# 로그 포맷(JSON)
logformat json %ts.%03tu {"client":"%>a","user":"%un","method":"%rm","uri":"%ru","status":%>Hs,"bytes":%<st}
access_log stdio:/var/log/squid/access_json.log json
```

## 4.2 Reverse Proxy (인터넷 → 사내)
- 목적: TLS 종료, 라우팅, **WAF 전단/후단**, 캐싱, 압축, HTTP/2/3, mTLS, 인증 연계
- 예: **Nginx/HAProxy/Envoy** + **ModSecurity/WAF**

### HAProxy L7 ACL/Rate-Limit 예시
```haproxy
frontend fe_https
  bind :443 ssl crt /etc/haproxy/certs/site.pem alpn h2,http/1.1
  acl api_path path_beg /api/
  http-request deny if METH_GET api_path { src_get_gpc0 gt 50 }  # 단순 per-IP 카운터(샘플)
  default_backend be_app

backend be_app
  server app1 127.0.0.1:8080 check
```

---

# 5) DDoS 보호 — L3/4/7 계층별 **공격 분류와 대응 전술**

## 5.1 공격 분류
- **Volumetric(L3/4)**: 대역폭 고갈(UDP Flood/Amplification: DNS/NTP/CLDAP/SSDP 등)  
- **Protocol(L4)**: **SYN/ACK/RST** 남용, 연결 상태 고갈  
- **Application(L7)**: HTTP GET/POST Flood, 로그인/검색 엔드포인트 집중, 비정상 헤더/쿠키 확장

## 5.2 계층별 대응 맵

### (1) **네트워크/라우터(Edge)**  
- **RTBH(Remote Triggered Blackhole)**, **FlowSpec**(프로바이더 협력)  
- **uRPF(Loose)**: 스푸핑 소스 차단  
- **CoPP**: 제어평면 레이트 제한  
- **Anycast/CDN**: 트래픽 분산

### (2) **방화벽(L3/4)**  
- **SYN Cookies**(호스트/OS), **초기 핸드셰이크 레이트 제한**  
- `nft/iptables` 레이트 제한(아래 예시)

```bash
# nftables: SYN 초당 200개 이상은 Drop(샘플, 환경에 맞게)
sudo nft add table inet ddos
sudo nft add chain inet ddos input { type filter hook input priority -100; }
sudo nft add rule inet ddos input tcp flags syn limit rate 200/second accept
sudo nft add rule inet ddos input tcp flags syn drop
```

### (3) **WAF/Reverse Proxy(L7)**  
- **자연어/봇탐지** 대신 먼저 **정책 기반**: 고비용 엔드포인트 RateLimit, **CAPTCHA/JS-Challenge**  
- **캐싱**: 동일 GET은 CDN/Edge 캐시 유도  
- **Session/Token 비용화**: 비로그인 고비용 API 보호(큐/슬로틀)

```nginx
# 고비용 /search 엔드포인트 rate 제한
location /search {
  limit_req zone=req_per_ip burst=10 nodelay;
  proxy_pass http://app:8080;
}
```

### (4) **클라우드 보호(요지)**  
- AWS Shield/Azure DDoS Protection/GCP Armor: **대용량 스크러빙** + 오토튜닝 룰  
- CDN(WAF 연계): HTTP L7 대규모 분산/캐시 Offload

> ⚠️ **주의**: 실제 공격 재현은 법적/운영 리스크가 큽니다. 테스트는 **자체 랩**에서 **소규모 합법적 부하 도구**로만 수행하세요.

## 5.3 간단 수용력 계산(개념)
$$
\text{Conn\_Capacity} \approx \frac{\text{메모리 크기}}{\text{커넥션당 상태 메모리}} \quad,\quad
\text{Pkt/s 처리량} \approx \text{CPU 코어} \times \text{pps/코어}
$$
- **튜닝 포인트**: NIC RSS/RPS, IRQ 분리, 커널 오프로딩, XDP Drop(선단), 고성능 프록시(멀티코어 스케일)

---

# 6) 레퍼런스 설계 3종

## 6.1 SMB 인터넷 경계(온프레미스)
- Edge Router(uRPF/CoPP) → NGFW(Zone) → Reverse Proxy+WAF → App
- 로그: 방화벽/IDS/WAF → 중앙 SIEM(경보 임계, 자동 티켓)

## 6.2 엔터프라이즈(하이브리드)
- Anycast-CDN/WAF(클라우드) + 온프레 NGFW/IPS 이중화 + RTBH/FlowSpec 계약
- 마이크로세그먼트(동서 트래픽: SDN/분산 방화벽/NSX/ACI)

## 6.3 클라우드 네이티브(Kubernetes)
- Ingress Controller(NGINX/Envoy) + WAF(모듈/사이드카)  
- NetworkPolicy(CNI)로 Pod간 L3/4 ACL  
- eBPF/XDP로 노드 선단 RateLimit/Drop

---

# 7) 엔드투엔드 시나리오 (구성→검증)

## 시나리오 A — “인터넷 공개 웹에 L7 방어막 얹기”

**목표**: HTTPS 종단에 **Reverse Proxy + WAF + RateLimit** 배치

1) NGINX TLS 종료 + ModSecurity + CRS 설치  
2) `/api/`에 Positive 룰(POST/JSON만), `/search`에 RateLimit 5r/s  
3) `DetectionOnly` → FP 조정 → `/api/orders` 차단 모드 전환  
4) 로그를 EFK(Elasticsearch/Fluent/ Kibana)로 송신, 대시보드 구성

**검증**:  
- 정상 트래픽 200 OK, 과도 호출 429 Too Many Requests, 비허용 메서드 405  
- WAF AuditLog에 차단 이벤트 기록 확인

---

## 시나리오 B — “경계 드롭 확대: Edge ACL + uRPF + CoPP”

1) Edge 인터페이스 Inbound에 **사설/Bogon 소스 Drop**  
2) uRPF Loose(`reachable-via any`)로 스푸핑 소스 차단  
3) CoPP로 ICMP Time-Exceeded, OSPF/BGP 제어 트래픽 **폴리싱**

**Cisco IOS-XE 예시**
```cisco
ip access-list extended EDGE-IN
 deny   ip 10.0.0.0 0.255.255.255 any
 deny   ip 172.16.0.0 0.15.255.255 any
 deny   ip 192.168.0.0 0.0.255.255 any
 permit ip any any
!
interface gi0/0
 ip access-group EDGE-IN in
 ip verify unicast source reachable-via any
!
control-plane
 service-policy input CONTROL-PLANE-POLICY   ! (섹션 CoPP 예시 참조)
```

**검증**:  
`show ip interface | i verify` / `show policy-map control-plane` / 드롭 카운터 상승

---

## 시나리오 C — “IPS Inline(부분) + 방화벽 오케스트레이션”

1) Suricata 두 포트 브리징(af-packet IPS)로 특정 **DMZ ↔ App** 구간만 인라인  
2) 차단 규칙: 명백한 익스플로잇/봇넷 C2 도메인  
3) 이벤트 발생 시 **방화벽 set(IPSet)** 으로 **임시 차단(자동화)**

**Suricata IPS 실행 예(개념)**
```bash
# /etc/suricata/suricata.yaml에서 af-packet: copy-mode IPS, cluster_type cluster_flow 설정
sudo suricata -c /etc/suricata/suricata.yaml -i eth1 -i eth2 --af-packet
```

**자동화 훅(발췌: EVE → 스크립트)**
```bash
# 의사코드: C2 탐지 시 IP를 nft set에 추가(시간 제한)
if jq '.alert.signature_id==<C2_SID>' eve.json; then
  nft add element inet edge bad_ips { 198.51.100.23 timeout 1h }
fi
```

---

# 8) 모니터링·지표·알림

## 8.1 필수 로그
- **방화벽**: 허용/차단 카운터, 규칙 Hit Top-N, 세션 수, conntrack 사용률  
- **WAF**: Rule ID별 차단, 경로/메서드 Top-N, FP Rate  
- **IDS/IPS**: 서명별 알림 수, 심각도, FQDN/IP Top-N  
- **프록시**: 사용자/그룹별 도메인, 차단 이벤트  
- **DDoS**: PPS/BPS 스파크, 4xx/5xx 비율, 레이턴시 SLI/SLO

## 8.2 예시 질의(Elasticsearch/Kibana, 개념)
```text
index: waf-*
query: action:"Blocked" AND request_uri:"/api/"
agg: terms(field: rule_id), date_histogram(@timestamp, 1m)
```

---

# 9) 운영 체크리스트

- [ ] **경계 기본**: uRPF(Loose), Edge ACL(사설/Bogon), CoPP 적용  
- [ ] **방화벽 정책**: 기본 Drop, 허용만 명시, 서비스 범위 최소, 로깅 샘플링  
- [ ] **WAF**: DetectionOnly → FP 조정 → 중요 경로 차단, 정기 룰 업데이트  
- [ ] **프록시**: Forward egress 통제(카테고리/도메인), Reverse 캐시/압축/HTTP2  
- [ ] **IPS/IDS**: Inline 범위 제한, Bypass 계획, 룰셋 관리/성능 모니터링  
- [ ] **DDoS**: RateLimit/MSS/SYN Cookies, CDN/클라우드 보호 계약, RTBH/FlowSpec 연습  
- [ ] **가시성**: EVE/Access/Firewall/WAF 로그 중앙화, 대시보드/월간 리뷰  
- [ ] **변경관리**: 파일럿 → 링 배포 → 롤백 테스트

---

# 10) 트러블슈팅 & 롤백

- **과도한 차단**: 룰 Hit 분석 → **DetectionOnly** 회귀 → 예외(화이트리스트) → 재적용  
- **False Positive**(WAF/IPS): 요청 본문/컨텍스트 확인, 서명 조정(`ctl:ruleRemoveById`)  
- **성능 이슈**: 룰/정책 순서 최적화, 로그 레벨 축소, NIC 오프로딩·IRQ 튜닝, 스케일아웃  
- **DDoS 중 장애**: 우선순위 = **서비스 지속** → CDN 캐시 강화, 비핵심 경로 임시 차단, RTBH 발동

**롤백 스니펫(예: nftables 정책 중지)**
```bash
sudo nft list ruleset > /var/backups/nft_rollback_$(date +%F).conf
# 임시 완화로 허용 확대(긴급)
sudo nft flush chain inet edge input
sudo nft add rule inet edge input accept
# 공격 종식 후 백업에서 복구
sudo nft -f /var/backups/nft_rollback_YYYY-MM-DD.conf
```

---

# 11) 보안·규정·윤리 주의

- 본 문서는 **방어/보호 목적**이며, 실습/부하 테스트는 **자체 소유 랩**에서만 수행하세요.  
- TLS 가시화/프록시 등은 **개인정보 보호·규정 준수** 범위내에서, 사전 고지·승인 필수.

---

## 부록 A) 빠른 스니펫 모음

### A-1) iptables 간단 경계(대응용)
```bash
iptables -P INPUT DROP
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -m conntrack --ctstate NEW -j ACCEPT
iptables -A INPUT -p tcp --syn -m limit --limit 200/second --limit-burst 400 -j RETURN
iptables -A INPUT -p tcp --syn -j DROP
```

### A-2) NGINX 보안 헤더(요약)
```nginx
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy "default-src 'self'" always;
```

### A-3) Suricata EVE 출력(Logstash/Fluent로 연동)
```yaml
# suricata.yaml (발췌)
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: /var/log/suricata/eve.json
      types:
        - alert
        - http
        - dns
        - tls
        - flows
```

---

## 부록 B) 미니 퀴즈 (자체 점검)

1) uRPF **Strict**와 **Loose**의 차이와, 어느 지점에서 무엇을 선택할지 설명하시오.  
2) WAF에서 **Positive 모델**을 적용할 때 필수로 고려할 메타데이터(메서드/경로/헤더/스키마)는?  
3) CoPP가 필요한 이유와 과도한 폴리싱의 부작용은?  
4) DDoS 방어에서 Anycast/CDN의 장점과 전제조건은?  
5) IDS를 **Inline**으로 전환하기 전 체크해야 할 3가지는?

---

## 마무리

- **방화벽**은 어뷰즈를 **초기에 드롭**, **IPS/IDS**는 **정밀 탐지/선별 차단**, **WAF**는 **어플리케이션 레벨 보호**,  
  **프록시**는 **출구/입구 제어 + 가시성**, **DDoS**는 **분산·흡수·한계 설정**이 핵심입니다.  
- 모든 구성은 **가시성(로그/지표)**가 없으면 개선할 수 없습니다. 먼저 **보는 눈**을 만들고, **작게 적용→검증→확장**하세요.
