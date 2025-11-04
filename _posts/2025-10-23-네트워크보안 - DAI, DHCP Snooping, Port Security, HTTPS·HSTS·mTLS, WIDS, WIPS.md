---
layout: post
title: 네트워크보안 - DAI/DHCP Snooping/Port Security, HTTPS·HSTS·mTLS, WIDS/WIPS
date: 2025-10-23 19:25:23 +0900
category: 네트워크보안
---
# 4.5 방어: DAI/DHCP Snooping/Port Security, HTTPS·HSTS·mTLS, WIDS/WIPS

> 목표: 스위치 L2 보호(DAI, DHCP Snooping, Port Security)와 L7 암호화(HTTPS/HSTS/mTLS),  
> 무선 영역 WIDS/WIPS 정책으로 **스니핑/가로채기 시도 자체를 어렵게** 만든다.

---

## 4.5.1 L2 보호: DHCP Snooping → DAI(ARP Inspection) → Port Security

### (1) DHCP Snooping — “누가 DHCP 서버인지 정한다”
- **효과**: 신뢰 포트(Trusted)에서 온 DHCP Offer/ACK만 허용, 나머지는 차단.  
- **부가효과**: 클라이언트 **IP–MAC–VLAN–Port 바인딩 테이블**을 유지 → **DAI가 이 테이블을 신뢰**.

**개념 구성(벤더 공통 개념, CLI는 예시)**  
```text
# 전역 활성화
ip dhcp snooping
# VLAN 10만 보호
ip dhcp snooping vlan 10

# 업링크(진짜 DHCP 서버가 있는 방향) 포트만 "trusted"
interface Gi1/0/48
  ip dhcp snooping trust
  # 과도한 Offer/ACK 남발 방지(DoS 억제)
  ip dhcp snooping limit rate 50

# 나머지 액세스 포트는 "untrusted" (기본)
interface Gi1/0/1
  switchport access vlan 10
  # 필요하면 rate limit 설정
  ip dhcp snooping limit rate 15
```

> 포인트: **서버가 있는 방향만 trusted**, 나머지는 기본 untrusted.  
> “가짜 DHCP”는 Offer/ACK가 **차단**되어 고립된다.

---

### (2) DAI (Dynamic ARP Inspection) — “게이트웨이 ARP 위조 차단”
- **효과**: ARP 요청/응답을 **DHCP Snooping 바인딩**과 대조, 불일치 ARP를 드롭.  
- **추가**: ARP rate limit, 로그.

**개념 구성 예시**  
```text
ip arp inspection vlan 10

# trusted 포트에서는 ARP 검사 우회(업링크/서버 포트)
interface Gi1/0/48
  ip arp inspection trust

# 액세스 포트는 기본 untrusted → ARP 검사 적용
interface Gi1/0/1
  ip arp inspection limit rate 15 burst interval 1
```

> 포인트: DAI가 제대로 동작하려면 **DHCP Snooping 바인딩**이 있어야 한다(정적 IP 단말은 별도 바인딩 등록 필요).

---

### (3) Port Security — “한 포트에 붙을 수 있는 MAC을 제한”
- **효과**: 포트당 MAC 수 제한, 특정 MAC만 허용, 위반 시 shutdown/restrict.  
- **활용**: **MAC 플러딩/스니핑 시도**의 부작용(다중 MAC 유입)을 **초기에 차단**.

**개념 구성 예시**  
```text
interface Gi1/0/1
  switchport mode access
  switchport access vlan 10
  switchport port-security
  switchport port-security maximum 2      # 허용 MAC 수
  switchport port-security violation restrict  # 초과시 드롭+로그(또는 shutdown)
  switchport port-security mac-address sticky  # 관찰 MAC을 자동 학습(운영 절차 주의)
```

> 권장: 초기 배포는 `violation restrict`로 오탐 영향 최소화 → 운영 안정화 후 정책 고도화.

---

## 4.5.2 L7 암호화: HTTPS·HSTS·mTLS

### (1) HTTPS + HSTS — 평문 제거의 기본
- **HSTS**: 브라우저가 **항상 HTTPS만** 사용하도록 강제(HTTP→HTTPS 리다이렉트도 HSTS 없이는 첫 요청은 평문일 수 있음).  
- **프리로드**: hstspreload.org 등록(요건 충족 시) → 브라우저 내장 목록으로 초회도 안전.

**Nginx 예시 스니펫(학습용)**  
```nginx
server {
  listen 443 ssl http2;
  server_name example.com;

  ssl_certificate     /etc/nginx/certs/fullchain.pem;
  ssl_certificate_key /etc/nginx/certs/privkey.pem;

  # 강한 기본값(운영은 보안 가이드 준수)
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_prefer_server_ciphers on;

  # HSTS: 서브도메인 포함, 프리로드 준비
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

  # 보안 헤더(요약)
  add_header X-Content-Type-Options nosniff always;
  add_header X-Frame-Options DENY always;

  location / { root /usr/share/nginx/html; index index.html; }
}

# HTTP로 오면 즉시 HTTPS로
server {
  listen 80;
  server_name example.com;
  return 301 https://$host$request_uri;
}
```

> 체크: HSTS **preload**는 장기적 영향 큼(롤백 어려움). 반드시 사전 검증.

---

### (2) 내부/서버간 mTLS — “서버/클라이언트 모두 인증”
- **효과**: 내부 API/관리자 콘솔에 **클라이언트 인증서** 없이는 접근 불가.  
- **장점**: 쿠키·토큰 탈취만으로는 접근 어려움(스니핑 난이도 급상승).

**Nginx Reverse Proxy에서 클라이언트 인증 강제(요약)**  
```nginx
server {
  listen 443 ssl http2;
  server_name internal.example.local;

  ssl_certificate         /etc/nginx/certs/srv_fullchain.pem;
  ssl_certificate_key     /etc/nginx/certs/srv_privkey.pem;

  # 클라이언트 인증서 검증(사내 CA)
  ssl_client_certificate  /etc/nginx/certs/ca_chain.pem;
  ssl_verify_client on;   # optional / on / off / optional_no_ca
  ssl_verify_depth 2;

  location /api/ {
    proxy_pass http://backend:8080/;
    # mTLS 성공 시만 전달(헤더에 DN 적재 등)
    proxy_set_header X-Client-Verify $ssl_client_verify;
    proxy_set_header X-Client-DN     $ssl_client_s_dn;
  }
}
```

**테스트용 CA/클라이언트 인증서(학습용 OpenSSL)**  
```bash
# 1) 루트 CA(테스트용)
openssl req -x509 -newkey rsa:4096 -days 365 -nodes \
  -keyout ca.key -out ca.crt -subj "/CN=LabCA"

# 2) 서버 CSR & 발급
openssl req -newkey rsa:2048 -nodes -keyout srv.key -out srv.csr -subj "/CN=internal.example.local"
openssl x509 -req -in srv.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out srv.crt -days 180
cat srv.crt ca.crt > srv_fullchain.pem

# 3) 클라이언트 CSR & 발급
openssl req -newkey rsa:2048 -nodes -keyout cli.key -out cli.csr -subj "/CN=developer01"
openssl x509 -req -in cli.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out cli.crt -days 180
# 브라우저 테스트용 P12
openssl pkcs12 -export -inkey cli.key -in cli.crt -certfile ca.crt -out cli.p12 -passout pass:
```

> 운영 팁: 발급·폐기 주기, 단말 자격증명 배포/회수 자동화(MDM/Secrets Manager)까지 포함해야 실무에 올라간다.

---

## 4.5.3 무선 WIDS/WIPS(탐지·차단)
- **WIDS**: 무선 이상(Deauth 급증, Rogue/Evil Twin, 불법 AP, WEP/오픈 SSID 등) **탐지**.  
- **WIPS**: 특정 기준 충족 시 **자동 차단**(우선은 탐지→운영 검증→선별 차단 권장).
- **정책 제언**
  - **WPA3-Enterprise + 802.1X(EAP-TLS)** 우선, 최소 **WPA2-Enterprise**  
  - **802.11w(PMF)** 활성화 → 관리 프레임 위변조(Deauth 등) 내성 강화  
  - SSID 프로파일 화이트리스트, **오픈/PSK 자동 연결 금지**  
  - 동일 SSID에 대한 **BSSID 화이트리스트**(허용 AP만), 채널/출력 기준치

---

# 4.6 탐지: ARP Reply 폭증, 게이트웨이 MAC 변화, Suricata/Zeek 룰

> 목표: L2~L7에서 **가시적 신호**를 포착해 경보/대응한다.  
> 아래 룰·스크립트는 **개념 템플릿**이며, 배포 전 **테스트망에서 오탐·성능 평가** 필수.

---

## 4.6.1 Suricata 예시 룰(EVE JSON 기반 운영)

### (1) ARP Reply 비정상 빈도(폭증)  
```conf
# 간단 예시: 짧은 시간 한 소스에서 과도한 ARP Reply
alert ether any any -> any any (msg:"L2 ARP Reply storm suspected";
  ether type 0x806;           # ARP
  # detection_filter는 by_src MAC 추적에서 한계가 있어 환경 맞춤 필요
  detection_filter:track by_src, count 60, seconds 10;
  sid:4202001; rev:1;)
```

### (2) 게이트웨이 MAC 변동 감지(개념)  
> Suricata 룰만으로 “MAC 변경” 상태를 기억하기 어렵다 → **로그 후단(파이프라인)**에서 **스테이트풀 비교** 추천.  
아래는 “게이트웨이 IP에서 오는 ARP Reply”를 태깅 → 후단에서 MAC change diff.

```conf
alert ether any any -> any any (msg:"ARP Reply from gateway IP";
  ether type 0x806; content:"\x00\x02"; offset: 6; depth: 2;  # op=2 (reply)
  # 게이트웨이 IP를 페이로드에서 추출(환경별 ARP TLV 오프셋 조정 필요)
  # 실전은 Suricata Lua/JA3같은 확장, 또는 Zeek가 더 편하다.
  sid:4202002; rev:1;)
```

**후단(예: Logstash/Fluentd/SIEM)에서 Diff 로직**  
- 키: `gateway_ip`  
- 값: `gateway_mac`  
- 이전 값 ≠ 현재 값 → **High 경보** (변경 시간, 포트, 스위치 인터페이스 추가 메타 필요)

---

## 4.6.2 Zeek 스크립트(ARP/DHCP/TLS 메타)

### (1) ARP 게이트웨이 MAC 변경 감지(개요)
```zeek
# file: arp_gateway_watch.zeek
@load protocols/arp

const gateway_ip: addr = 10.10.0.1;  # 환경에 맞추어 설정
global gw_mac: string &persistent;

event arp_reply(c: connection, spa: addr, sha: string, tpa: addr, tha: string) {
  # spa: sender protocol address(IP), sha: sender hardware address(MAC)
  if ( spa == gateway_ip ) {
    if ( gw_mac == "" ) {
      gw_mac = sha;
      print fmt("[GW-ARP] baseline set %s -> %s", gateway_ip, gw_mac);
    } else if ( gw_mac != sha ) {
      print fmt("[ALERT] Gateway MAC changed! %s: %s -> %s", gateway_ip, gw_mac, sha);
      gw_mac = sha;
    }
  }
}
```

### (2) DHCP 다중 서버 관찰(개요)
```zeek
# file: dhcp_multi_server_watch.zeek
@load protocols/dhcp

global dhcp_servers: set[addr] &create_expire=10mins;

event dhcp_offer(c: connection, msg: dhcp_msg, yiaddr: addr, siaddr: addr, subnet_mask: addr, router: dhcp_option_list) {
  add dhcp_servers[siaddr];
  if ( |dhcp_servers| > 1 ) {
    print fmt("[ALERT] Multiple DHCP servers observed: %s", dhcp_servers);
  }
}
```

### (3) TLS 메타(예: JA3 상위·비정상 도메인 패턴)
Zeek는 기본적으로 TLS 이벤트를 레코드화. SIEM에서 **도메인 평판/SNI 패턴**과 결합하면 효과적.

---

## 4.6.3 SIEM 룰(의사코드·Sigma 스타일)

**[Rule] 게이트웨이 MAC 변경**  
```
title: Gateway MAC Address Change
logsource: arp/nd logs (Suricata EVE, Zeek, Switch syslog)
detection:
  selection:
    event: ARP_REPLY
    ip: 10.10.0.1
  condition: track ip over 1h and alert when mac changes
level: high
```

**[Rule] ARP Reply 폭증**  
```
title: ARP Reply Storm by Source
detection:
  selection:
    event: ARP_REPLY
  condition: same src_mac > 100 in 10s
level: medium
```

**[Rule] DHCP Multi-Server**  
```
title: Multiple DHCP Servers in VLAN
detection:
  selection:
    event: DHCP_OFFER
  condition: distinct siaddr > 1 within 1m
level: high
```

---

# 4.7 랩: L2 ARP 기반 가로채기, 무선 오픈망 캡처, 탐지 룰 제작

> **취지**: 실제 공격 행위를 하지 않고도, **탐지/방어가 제대로 동작하는지** 확인한다.  
> (PCAP 재생, 합성 이벤트, 샌드박스 캡처, **무해한 ARP 변동** 등)

---

## 4.7.1 랩 토폴로지(로컬 네임스페이스 또는 Docker)

### (A) 네임스페이스 미니 랩(요약)
```
[ns-client]──veth-c  \
                      (br0)──veth-g──[ns-gw]
[ns-observer]──veth-o /
```
- `ns-observer`: **패시브 캡처 전용**(tcpdump/Zeek/Suricata)
- `ns-gw`: 라우터 역할(필요 시 NAT)
- `ns-client`: 트래픽 발생

### (B) Docker Compose 랩(요약)
- `client` / `server` / `observer(tcpdump|zeek|suricata)`  
- `observer`는 `pcap` 볼륨에 저장, 후단 도구가 읽어 분석

**Compose 예시(발췌)**  
```yaml
version: "3.9"
services:
  server: { image: nginx:alpine, networks: [labnet] }
  client: { image: curlimages/curl:8.10.1, entrypoint: ["sleep","infinity"], networks: [labnet] }
  observer:
    image: corfr/tcpdump
    command: ["-i","any","-s","0","-w","/pcap/lab.pcap"]
    cap_add: ["NET_ADMIN","NET_RAW"]
    volumes: ["./pcap:/pcap"]
    networks: [labnet]
networks: { labnet: { driver: bridge } }
```

---

## 4.7.2 시나리오 1 — “L2 ARP 기반 가로채기” **탐지만** 검증

> 실제 ARP 포이즈닝은 하지 않는다. 대신 **무해한 GARP 반복**으로 “ARP 변동” 시그널만 발생시켜  
> **DAI/Zeek/Suricata/SIEM 룰이 울리는지** 확인한다.

**합성 스크립트(앞 4.2.1에서 제시, 재사용)**  
```python
# safe_arp_churn_generator.py (요약)
# 목적: 자신의 IP/MAC로 GARP 반복 → ARP 변동 신호를 탐지 체인으로 밀어넣기
```

**절차**
1) `observer`에서 캡처 시작: `tcpdump -i any -nn -e -w /pcap/arp_test.pcap 'arp'`  
2) `client` 네임스페이스/컨테이너에서 GARP 합성 스크립트 실행(10~20회)  
3) Zeek/Suricata에서 이벤트 수집 → SIEM 룰 트리거 확인  
4) 스위치가 있다면(물리 랩): DAI 로그/카운터 변화 확인

**성공 기준**
- Zeek: `[ALERT] Gateway MAC changed!` 또는 ARP churn 관련 경보  
- Suricata: `L2 ARP Reply storm suspected` 카운트 증가(튜닝값 내)  
- SIEM: “Gateway MAC change” 룰 발화

---

## 4.7.3 시나리오 2 — “무선 오픈망 캡처(패시브)”

> **공격 불가**. 자신이 소유/관리하는 **테스트 AP**만 사용. 오픈 SSID에서 **패시브 캡처**로  
> 관리 프레임(Beacon/Probe)와 데이터 프레임 메타만 관찰 → **WIDS 룰 테스트**.

**절차(개요)**  
1) 모니터 모드 인터페이스 준비(드라이버 지원 카드)  
2) 오픈 SSID로 테스트 AP 구성(격리)  
3) `tshark`/`Wireshark`로 관리 프레임 캡처  
4) **합성 WIDS 이벤트**(Deauth 카운트 NDJSON) 생성 → WIDS/SIEM에 주입(아래 코드)

**합성 Deauth 이벤트(4.3.3 재사용)**  
```python
# synth_deauth_events.py (요약)
# "DEAUTH_OBSERVED_SYNTH" 이벤트를 NDJSON으로 생성해 테스트 인제션
```

**성공 기준**
- WIDS/SIEM에서 **Deauth 이벤트 룰**이 동작(“합성”임을 태깅)  
- Beacon RSN IE 분석: WEP/오픈/취약 설정 감지 룰 정상 동작  
- 운영 정책: 오픈 SSID 금지/PSK 자동 연결 금지 확인

---

## 4.7.4 시나리오 3 — “탐지 룰 제작 & 회귀(Regression) 테스트”

> 탐지 체인을 **CI 처럼** 돌려서, 룰 업데이트가 **기존 탐지 품질**을 깨지 않는지 확인.

**테스트 데이터 셋**
- `pcap/arp_churn_ok.pcap` : GARP 10회  
- `pcap/dhcp_dual_offer.pcap` : 두 서버 Offer(합성)  
- `pcap/tls_normal.pcap` : 정상 브라우저 트래픽  
- `pcap/tls_ja3_suspicious.pcap` : 알려진 의심 지문(합성)

**TShark 자동 채점 예시**  
```bash
# 1) ARP Reply 카운트(윈도 내) 기준
tshark -r pcap/arp_churn_ok.pcap -Y "arp.opcode == 2" | wc -l

# 2) DHCP Offer 송신자 수
tshark -r pcap/dhcp_dual_offer.pcap -Y "bootp.option.dhcp == 2" -T fields -e ip.src | sort -u | wc -l
```

**Suricata EVE 기반 채점(예: jq)**
```bash
# ARP storm 룰 hit 수
jq -r 'select(.alert and .alert.signature_id==4202001) | .timestamp' eve.json | wc -l

# TLS JA3 상위 Top N(의심 지문 확인)
jq -r 'select(.event_type=="tls") | .tls.ja3_hash' eve.json | sort | uniq -c | sort -nr | head
```

**회귀 테스트 스크립트(개념)**  
```bash
#!/usr/bin/env bash
set -euo pipefail
# 1) Suricata로 오프라인 분석
suricata -r pcap/arp_churn_ok.pcap -l out/arp --runmode=single
# 2) 기대치와 비교
hits=$(jq -r 'select(.alert and .alert.signature_id==4202001)' out/arp/eve.json | wc -l)
if [ "$hits" -lt 1 ]; then
  echo "[FAIL] ARP storm alert not triggered"; exit 1
fi
echo "[PASS] ARP storm minimal hit: $hits"
```

---

## 4.7.5 운영 반영 체크리스트

- [ ] **DHCP Snooping**: trusted 포트 정의, rate limit, 바인딩 테이블 점검 자동화  
- [ ] **DAI**: VLAN별 활성화, 정적 IP 단말 바인딩 수동 등록 플로우  
- [ ] **Port Security**: max MAC, sticky 운영 기준, 위반 처리(Restrict→Shutdown) 단계적 적용  
- [ ] **HTTPS+HSTS+mTLS**: 프리로드 체크, 내부 mTLS 발급/폐기 자동화, 보안 헤더 표준  
- [ ] **WIDS/WIPS**: Deauth/Disassoc 임계치, Rogue/Evil Twin 탐지 기준, PMF 활성화  
- [ ] **SIEM/NDR**: ARP/DHCP/TLS(JA3/JA4)/DNS 룰과 대시보드, 보존/마스킹/암호화 정책  
- [ ] **회귀 테스트**: pcap·합성 이벤트 세트로 룰 업데이트 시 자동 채점

---

## 4.7.6 성공/실패를 가르는 운영 팁

- **탐지만 울리고 끝나지 말 것**: 룰 → 알림 → **운영 대응(격리/차단/원복)** 절차가 체인으로 묶여야 한다.  
- **오탐 관리**: ARP/DHCP는 변동이 잦은 환경(VDI/무선 로밍)에서 오탐 유발 → **베이스라인/예외 목록/시간 창 튜닝**이 핵심.  
- **가시성 우선**: 복호화가 어려운 시대, **메타데이터(SNI/JA3/JA4), NetFlow, DNS 텔레메트리**를 적극 활용.  
- **암호화 기본값**: 평문은 즉시 제거, 내부는 mTLS·접근제어·비밀관리(주기 로테이션)까지 포함.

---
# 요약

- **L2 방어 삼총사**: DHCP Snooping(바인딩) → DAI(ARP 검증) → Port Security(MAC 제한).  
- **L7 방어**: HTTPS+HSTS로 평문 제거, 내부는 **mTLS**로 신뢰 경계 축소.  
- **무선**: WIDS/WIPS + WPA3/PMF로 관리 프레임 위변조·사칭 내성.  
- **탐지**: ARP Reply 폭증·GW MAC 변경·DHCP 다중 서버·TLS 메타 지문을 **Suricata/Zeek/SIEM**으로 상관 분석.  
- **랩**: 공격 재현 없이 **무해한 합성 이벤트·pcap 회귀 세트**로 **탐지/대응 파이프라인**을 검증·자동화한다.