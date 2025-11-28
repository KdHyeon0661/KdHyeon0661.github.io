---
layout: post
title: 네트워크보안 - DAI/DHCP Snooping/Port Security, HTTPS·HSTS·mTLS, WIDS/WIPS
date: 2025-10-23 19:25:23 +0900
category: 네트워크보안
---
# 방어: DAI/DHCP Snooping/Port Security, HTTPS·HSTS·mTLS, WIDS/WIPS

## 1. L2 스위치 방어: DHCP Snooping / DAI / Port Security

### 1.1 위협 모델: 왜 L2를 방어해야 하는가

L2 보호 기능을 이해하려면 먼저 **공격자의 목표**를 정리하는 편이 좋다.

- **DHCP 스푸핑**  
  - 공격자가 가짜 DHCP 서버를 띄워 클라이언트에게 **엉뚱한 게이트웨이·DNS**를 할당한다.
  - 클라이언트 트래픽이 공격자 장비를 거쳐 가도록 만들어 **MITM(중간자 공격)** 기반 스니핑·변조를 시도한다.

- **ARP 스푸핑 / ARP 포이즈닝**
  - 공격자가 “게이트웨이 IP → 내 MAC” 형태의 ARP Reply를 지속적으로 뿌려  
    **게이트웨이 ARP 캐시**를 오염시킨다.
  - 결과적으로, 게이트웨이와 피해자 사이의 트래픽이 공격자를 경유하게 된다.

- **MAC 플러딩**
  - 스위치 포트에 **수천 개의 MAC**을 흘려보내 MAC 주소 테이블을 가득 채운다.
  - 스위치가 정상 학습을 못 하면 특정 조건에서 **허브처럼 브로드캐스트성 동작**을 하여,
    같은 VLAN 내 공격자가 다른 단말의 프레임을 쉽게 스니핑할 수 있다.

이 세 가지는 서로 얽혀 있다.  
가짜 DHCP 서버로 **잘못된 게이트웨이 IP**를 할당하고, 그 IP에 대응하는 ARP를 **포이즈닝**하는 식으로 체인을 만든다.  
따라서 스위치 입장에서는 아래 세 가지를 동시에 방어하도록 설계한다.

1. **DHCP Snooping**: “누가 DHCP 서버인지”를 스위치가 규정한다.  
2. **DAI(Dynamic ARP Inspection)**: DHCP Snooping으로 얻은 **IP–MAC–Port 바인딩**에 맞지 않는 ARP는 버린다.  
3. **Port Security**: 포트당 MAC 수·패턴을 제한해 MAC 플러딩·스니핑 시도를 초장에 봉쇄한다.

---

### 1.2 DHCP Snooping — “누가 DHCP 서버인지 정한다”

#### 1.2.1 개념

- **역할**: 스위치가 네트워크 내에서 **“공식 DHCP 서버가 있는 방향”을 명시**하고,
  그 포트에서 오는 DHCP Offer/ACK만 허용한다.
- **효과**
  - **신뢰 포트(Trusted)**: DHCP 서버가 있는 업링크/서버 포트 → Offer/ACK 허용.
  - **비신뢰 포트(Untrusted)**: 일반 액세스 포트 → Offer/ACK 차단 (Request는 허용).
- **부가 효과**
  - 모든 DHCP 트랜잭션을 보며 **IP–MAC–VLAN–Port 바인딩 테이블**을 만든다.  
    이 바인딩은 나중에 **DAI·IP Source Guard** 같은 다른 기능이 신뢰할 수 있는 근거가 된다.

즉, DHCP Snooping은 **“공식 DHCP 서버는 여기뿐”**이라고 스위치에 알려주는 기능이다.

#### 1.2.2 기본 구성 예시 (Cisco IOS 스타일)

```text
! 전역 활성화
ip dhcp snooping

! VLAN 10만 보호
ip dhcp snooping vlan 10

! DHCP 서버가 있는 업링크/서버 포트만 "trusted"
interface Gi1/0/48
  description Uplink-to-Core-or-DHCP-Server
  ip dhcp snooping trust
  ! 과도한 Offer/ACK 남발 방지(DoS 억제)
  ip dhcp snooping limit rate 50

! 나머지 액세스 포트는 "untrusted" (기본값)
interface Gi1/0/1
  description Client-Port
  switchport access vlan 10
  ! 필요하면 rate limit 설정(오류난 단말의 DHCP 폭주 방지)
  ip dhcp snooping limit rate 15
```

> 포인트  
> - **서버/업링크 방향만 trusted**, 나머지는 기본 untrusted.  
> - “가짜 DHCP”는 Offer/ACK가 **차단**되어 고립된다.

#### 1.2.3 Snooping 바인딩 테이블

DHCP Snooping이 켜져 있으면 스위치는 대략 다음과 같은 바인딩 정보를 유지한다.

| 필드 | 예시 값        | 의미                         |
|------|----------------|------------------------------|
| VLAN | 10             | VLAN ID                      |
| MAC  | 00:11:22:33:44:55 | 단말 NIC MAC                 |
| IP   | 10.10.10.123   | 할당된 IP 주소               |
| Port | Gi1/0/1        | 단말이 붙은 인터페이스       |
| Type | dhcp-snooping  | DHCP에서 얻은 바인딩임을 표시 |

운영에서는 이 테이블을 **주기적으로 덤프**해서 CMDB/자산관리와 연동하기도 한다.  
예: SSH로 스위치에 접속해 `show ip dhcp snooping binding` 결과를 수집 →  
로그/DB에 축적 → 이상 징후(동일 MAC의 IP 반복 변경 등)를 탐지.

#### 1.2.4 정적 IP 단말 처리

문제는 **정적 IP를 쓰는 서버/프린터/네트워크 장비**이다.  
이들은 DHCP를 사용하지 않으므로 **Snooping 바인딩이 자동으로 생기지 않는다**.

- 해결 방법:
  1. **가능하면 DHCP로 전환**하고 IP는 “예약(Reservation)”으로 할당.  
     (서버/NW장비에도 DHCP Reservation은 충분히 많이 쓰인다.)
  2. 불가피하다면 스위치에 **정적 바인딩**을 수동 등록.

예시:

```text
ip source binding 0011.2233.4455 vlan 10 10.10.10.10 interface Gi1/0/5
```

이는 “Gi1/0/5 포트에 MAC 00:11:22:33:44:55가 붙어 있고, IP는 10.10.10.10이다”라는 사실을  
스위치에게 미리 알려 주는 것이다.  
이 정보는 DAI, IP Source Guard가 참고한다.

---

### 1.3 DAI (Dynamic ARP Inspection) — “게이트웨이 ARP 위조 차단”

#### 1.3.1 개념

- **역할**: VLAN 내에서 오가는 모든 **ARP Request/Reply를 검사**한다.
- **검사 기준**
  - **DHCP Snooping 바인딩**(또는 정적 IP Source Binding)과 비교.
  - ARP 패킷 속의 `IP–MAC` 쌍이 바인딩과 불일치하면 이를 **위조(스푸핑)** 로 판단하고 드롭.
- **효과**
  - 공격자가 “게이트웨이 IP → 공격자 MAC” 형태의 가짜 ARP Reply를 살포해도,
    바인딩과 맞지 않으면 스위치가 ARP를 버려 게이트웨이/클라이언트의 ARP 캐시가 오염되지 않는다.

공식적으로는 다음과 같은 목적을 가진다.

> “DHCP Snooping으로 검증된 단말만 유효한 ARP 맵핑을 광범위하게 전파할 수 있고,  
> 그 외에서 오는 ARP는 ‘보안 가드’의 통과를 받아야 한다.”

#### 1.3.2 기본 구성 예시

```text
! VLAN 10에 DAI 활성화
ip arp inspection vlan 10

! trusted 포트에서는 ARP 검사 우회(업링크/서버 포트)
interface Gi1/0/48
  ip arp inspection trust

! 액세스 포트는 기본 untrusted → ARP 검사 적용
interface Gi1/0/1
  ip arp inspection limit rate 15 burst interval 1
```

> 포인트  
> - DAI가 제대로 동작하려면 **DHCP Snooping 바인딩**이 있어야 한다.  
>   정적 IP 단말은 별도 바인딩 등록 또는 예외 포트(trusted)로 처리해야 한다.  
> - ARP 트래픽이 많은 환경에서는 `limit rate` 값을 넉넉히 잡지 않으면 **정상 단말이 차단**될 수 있다.

#### 1.3.3 DAI 동작 시나리오

간단한 예로, 다음과 같은 VLAN 10을 가정한다.

- Gateway: `10.10.10.1 / MAC: aa:aa:aa:aa:aa:aa`
- Client A: `10.10.10.100 / MAC: 00:11:22:33:44:55`
- Attacker: `10.10.10.200 / MAC: 66:77:88:99:aa:bb`

공격자가 다음과 같은 ARP Reply를 전송한다고 하자.

```text
Sender IP:   10.10.10.1      (게이트웨이 IP)
Sender MAC:  66:77:88:99:aa:bb (공격자 MAC)
Target IP:   10.10.10.100    (피해자)
Target MAC:  00:11:22:33:44:55
```

- 스위치는 이 ARP Reply를 수신하면,
  - “10.10.10.1에 대한 공식 바인딩은 Gateway MAC aa:aa:aa:aa:aa:aa다.” 라는 정보를 Snooping 테이블에서 본다.
  - 패킷 속 Sender MAC 66:77:88:99:aa:bb는 이와 **불일치**하므로 위조로 판단한다.
- DAI가 활성화된 포트라면, 이 ARP Reply는 **드롭**되고 로그에 남는다.

#### 1.3.4 DAI와 DHCP Snooping 의존성

- DHCP Snooping 없이 DAI만 켜면, 스위치는 어느 쪽이 정상인지 알 수 없다.
- 대부분의 벤더는 다음과 같은 정책을 가진다.
  - DHCP Snooping 바인딩이 있다 → 바인딩을 기준으로 ARP 검사.
  - 바인딩이 없다 → (옵션에 따라) 모든 ARP 허용 또는 검사 불가.

따라서 **항상 “DHCP Snooping → DAI” 순으로 구축**해야 한다.

---

### 1.4 Port Security — “한 포트에 붙을 수 있는 MAC을 제한”

#### 1.4.1 개념

- **역할**
  - 포트별로 허용할 **MAC 주소의 수/패턴**을 제한한다.
  - 정책을 위반하는 MAC이 등장하면 해당 포트에 대해 **restrict/shutdown** 등 보호 조치를 취한다.
- **주요 사용 목적**
  - **MAC 플러딩 공격 방어**
  - **한 포트에 여러 단말이 붙는 상황 제한** (Mini-Hub, 임의 AP, 스위치 연결 방지)
  - 간단한 단말 인증(“이 포트에는 이 장비만 연결”) 수준의 보안

Port Security는 **이더넷 스위치의 “최후의 물리 계층 방어선”** 역할을 한다고 볼 수 있다.

#### 1.4.2 기본 구성 예시

```text
interface Gi1/0/1
  description Client-Port
  switchport mode access
  switchport access vlan 10

  switchport port-security
  switchport port-security maximum 2           ! 허용 MAC 수 (예: 2개)
  switchport port-security violation restrict  ! 초과시 드롭 + 로그 (또는 shutdown)
  switchport port-security mac-address sticky  ! 관찰 MAC을 자동 학습 (운영 절차 주의)
```

- `maximum 2`  
  - 예: 사용자가 노트북 + IP Phone 2개만 쓰는 환경에서, 포트에 3번째 MAC이 등장하면 이상 상황으로 본다.
- `violation restrict`
  - 초과 MAC의 프레임을 드롭하면서 로그만 남기고 포트는 계속 up 상태를 유지.
  - 운영 초기 오탐 영향 최소화에 유리.
- `violation shutdown`
  - 정책 위반 시 포트를 err-disable로 떨어뜨린다.  
    운영 절차가 잘 정비되지 않으면 단말 장애가 빈번해질 수 있다.
- `mac-address sticky`
  - 포트에 유입되는 MAC을 자동으로 학습해서 Running Config에 고정.  
  - 이후에는 “이 포트에는 이 MAC만 허용”처럼 동작한다.
  - 자동 학습한 Config를 저장하지 않고 재부팅하면 다시 학습해야 한다는 점에 주의.

> 권장 운영 패턴  
> 1. 초기에는 `violation restrict` + `maximum` 완화 값으로 시작.  
> 2. 운영 경험이 쌓이면 `maximum`을 타이트하게 줄이고, 일부 포트는 `shutdown`으로 강화.  
> 3. 서버/중요 인프라 포트는 sticky + 지정 MAC 조합으로 **강력 통제**.

#### 1.4.3 Port Security와 DHCP Snooping/DAI의 조합

세 기능을 함께 쓰는 전형적인 패턴은 다음과 같다.

1. **DHCP Snooping**  
   - 모든 클라이언트 IP 할당 경로를 통제.
2. **DAI**  
   - DHCP Snooping 바인딩 기반으로 ARP 위조 차단.
3. **Port Security**  
   - 한 포트에 붙을 수 있는 MAC 수를 제한해 MAC 플러딩 혹은 스위치/허브 무단 연결을 차단.

아키텍처 그림으로 표현하면:

```text
[Client Port Gi1/0/1]
   │
   ├─ DHCP Snooping (Untrusted) → "IP–MAC–Port" 바인딩 생성
   ├─ DAI (Untrusted) → 바인딩과 불일치 ARP 드롭
   └─ Port Security → MAC 수/패턴 위반 시 프레임 드롭 또는 포트 차단
```

이렇게 L2 단계에서 공격 표면을 줄여 놓으면, 이후 L3~L7에서의 HTTPS/mTLS, WIDS/WIPS, SIEM 탐지도 더 **깨끗한 전제** 위에서 작동한다.

---

## 2. L7 암호화: HTTPS · HSTS · mTLS

이제 L2 단에서 “누가 누구에게 프레임을 보내는지”를 통제했다면,  
L7에서는 “**보낸 내용을 중간에서 읽을 수 없게 만드는 것**”이 핵심이다.

- 외부 사용자를 향한 웹 서비스 → **HTTPS + HSTS**  
- 내부 서비스/관리 콘솔 → **mTLS(서버·클라이언트 양방향 인증)**

### 2.1 HTTPS 기본 위협 모델

HTTP는 기본적으로 **평문**이다.

- 같은 L2 도메인에 있는 공격자는
  - ARP 스푸핑/스위칭 미스설정 등을 이용해 트래픽을 **가로채거나 복제**할 수 있다.
  - 그 패킷 안의 HTTP 요청/응답을 그대로 읽을 수 있고, 쿠키·세션 토큰을 탈취할 수 있다.
- VPN·프록시를 우회해서 애플리케이션 계층까지 들어온 공격자도,
  - 서버/역프록시 구간에서 TLS가 빠져 있으면 “내부 평문 구간”을 노릴 수 있다.

따라서 실무에서는 다음 두 가지를 기본 원칙으로 삼는다.

1. **“사용자가 접속하는 입구”부터 HTTPS**  
2. 내부 마이크로서비스/관리 API도 **가능한 한 TLS(mTLS)로 보호**

---

### 2.2 HTTPS + HSTS — “평문 제거의 기본값”

#### 2.2.1 HSTS가 필요한 이유

- HTTPS만 쓰자는 조직도, **“첫 요청”**은 종종 HTTP로 들어온다.  
  예: 사용자가 브라우저 주소창에 `example.com`만 입력 → 기본 80 포트 HTTP 접속 시도.
- 서버가 `301` 리다이렉트로 HTTPS로 돌려보내는 동안,
  - 첫 HTTP 요청은 여전히 평문이다.
  - 이때 공격자가 중간에 끼어 **HTTP 응답을 가로채거나 변조**할 수 있다.
    - 예: HTTP 응답에 악성 자바스크립트 삽입.
    - 혹은 사용자를 전혀 다른 피싱 사이트로 리다이렉트.

**HSTS(HTTP Strict Transport Security)**는 브라우저에게 “이 도메인은 무조건 HTTPS만 써라”라고 알려주는 **정책 헤더**다.

- 브라우저가 한 번 HSTS 정책을 받으면,
  - **지정된 기간 동안(예: 1년)** 이 도메인에 대해 HTTP 연결을 시도하지 않고  
    바로 HTTPS를 사용한다.
  - 사용자가 주소창에 `http://example.com`을 입력하더라도 **브라우저 내부에서 HTTPS로 변환**한다.

#### 2.2.2 HSTS 활성화 예시 (Nginx)

```nginx
server {
  listen 443 ssl http2;
  server_name example.com;

  ssl_certificate     /etc/nginx/certs/fullchain.pem;
  ssl_certificate_key /etc/nginx/certs/privkey.pem;

  # TLS 버전/암호군은 조직의 보안 가이드에 맞게 설정
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_prefer_server_ciphers on;

  # HSTS: 서브도메인 포함, 프리로드 준비
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

  # 기타 보안 헤더(요약)
  add_header X-Content-Type-Options nosniff always;
  add_header X-Frame-Options DENY always;
  add_header Referrer-Policy strict-origin-when-cross-origin always;
  add_header X-XSS-Protection "0" always;  # 현대 브라우저 기준

  location / {
    root /usr/share/nginx/html;
    index index.html;
  }
}

# HTTP로 접속하면 즉시 HTTPS로 리다이렉트
server {
  listen 80;
  server_name example.com;
  return 301 https://$host$request_uri;
}
```

> 운영 시 주의  
> - `preload` 옵션을 쓰고 hstspreload.org에 등록하면 **브라우저 내장 목록**에 올라간다.  
>   롤백이 매우 어렵기 때문에, **서브도메인/인증서 체계/운영 프로세스**가 충분히 안정된 뒤에 적용해야 한다.  
> - 테스트 도메인/스테이징에서는 `max-age`를 짧게 두고 preload는 사용하지 않는 편이 안전하다.

---

### 2.3 TLS 설정 베이스라인과 운영 팁

TLS는 “켜기만 하면 끝”이 아니다.  
**버전, 암호군, 인증서 정책, 세션 재사용 전략** 등에서 현실적인 선택이 필요하다.

#### 2.3.1 TLS 버전 선택

- 실무 기본값
  - **TLS 1.2 / 1.3만 허용**  
  - TLS 1.0/1.1은 이미 주요 브라우저·클라우드에서 지원 중단 흐름이 확실하다.
- 레거시 클라이언트가 꼭 필요하다면
  - 전용 레거시 도메인/별도 리버스 프록시로 격리하고,  
    접근 통제·모니터링을 강화하는 편이 낫다.

#### 2.3.2 암호군(Cipher Suite) 고려

- TLS 1.3  
  - 프로토콜 수준에서 모던 암호군만 지원하므로, 대체로 선택이 단순하다.
- TLS 1.2  
  - 기본적으로 **ECDHE + AES-GCM** 계열을 우선한다.
  - **RC4, 3DES, NULL, EXPORT, MD5 기반** 등 오래된 암호군은 모두 비활성화.

예시(단순화):

```nginx
ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:
             ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
ssl_prefer_server_ciphers on;
```

> 실제 운영에서는 조직 보안 정책, 외부 규제(예: 금융 규제),  
> 클라이언트 호환성(모바일 앱/임베디드 단말) 등을 종합해 결정해야 한다.

#### 2.3.3 인증서 수명·발급 전략

- **짧은 수명 + 자동 갱신**이 권장되는 추세이다.
  - 예: 공인 인증서는 90일 또는 1년 단위, 내부 인증서는 1년 이하.
  - 모든 인증서는 **자동 갱신 + 자동 롤아웃 파이프라인**에 올려 두어야 한다.
- 키 관리
  - 프라이빗 키는 HSM/전용 비밀 관리 시스템에 보관.
  - CI/CD 파이프라인에서 인증서를 배포할 때는  
    “빌드 아티팩트에 키 포함” 같은 안티패턴을 피한다.

---

### 2.4 내부/서버간 mTLS — “서버/클라이언트 모두 인증”

#### 2.4.1 개념

- 일반 TLS:
  - **서버만 인증**(서버 인증서).
  - 클라이언트는 ID/패스워드·세션 토큰·쿠키 같은 **애플리케이션 계층** 자격 증명을 사용.
- mTLS(mutual TLS):
  - 서버와 클라이언트 모두 **X.509 인증서**를 가진다.
  - TLS 핸드셰이크 과정에서 클라이언트가 자신의 인증서를 제시하고 검증을 통과해야만 연결이 완료.

효과:

- 네트워크 내부에서 쿠키·토큰이 노출되더라도,  
  **클라이언트 인증서를 가진 단말이 아니면 전혀 연결 자체가 안 된다.**
- VPN 터널 안에 또 다른 **강력한 신뢰 경계**를 한 번 더 두는 셈이다.

#### 2.4.2 Nginx Reverse Proxy에서 mTLS 강제 (요약)

```nginx
server {
  listen 443 ssl http2;
  server_name internal.example.local;

  ssl_certificate         /etc/nginx/certs/srv_fullchain.pem;
  ssl_certificate_key     /etc/nginx/certs/srv_privkey.pem;

  # 클라이언트 인증서 검증 (사내 CA)
  ssl_client_certificate  /etc/nginx/certs/ca_chain.pem;
  ssl_verify_client on;       # optional / on / off / optional_no_ca
  ssl_verify_depth 2;

  location /api/ {
    proxy_pass http://backend:8080/;

    # mTLS 성공 시에만 전달되는 메타
    proxy_set_header X-Client-Verify $ssl_client_verify;
    proxy_set_header X-Client-DN     $ssl_client_s_dn;
    proxy_set_header X-Client-Cert   $ssl_client_cert;
  }
}
```

- `ssl_client_certificate`  
  - 클라이언트 인증서 체인을 검증할 때 사용할 CA 번들.
- `ssl_verify_client on`  
  - 인증서가 없거나 검증에 실패하면 TLS 핸드셰이크 단계에서 연결 종료.
- 백엔드 애플리케이션은 `X-Client-Verify / X-Client-DN` 등의 헤더를 통해
  - “어느 단말/사용자의 인증서였는지”를 추가로 확인할 수 있다.

#### 2.4.3 테스트용 CA/클라이언트 인증서 발급 (OpenSSL)

```bash
# 1) 루트 CA (테스트용)
openssl req -x509 -newkey rsa:4096 -days 365 -nodes \
  -keyout ca.key -out ca.crt -subj "/CN=LabCA"

# 2) 서버 CSR & 발급
openssl req -newkey rsa:2048 -nodes \
  -keyout srv.key -out srv.csr -subj "/CN=internal.example.local"

openssl x509 -req -in srv.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out srv.crt -days 180

cat srv.crt ca.crt > srv_fullchain.pem

# 3) 클라이언트 CSR & 발급
openssl req -newkey rsa:2048 -nodes \
  -keyout cli.key -out cli.csr -subj "/CN=developer01"

openssl x509 -req -in cli.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out cli.crt -days 180

# 4) 브라우저 테스트용 P12 (비밀번호는 공란)
openssl pkcs12 -export \
  -inkey cli.key -in cli.crt -certfile ca.crt \
  -out cli.p12 -passout pass:
```

이렇게 만든 `cli.p12`를 브라우저 인증서 저장소에 가져오면,  
`https://internal.example.local` 접속 시 브라우저가 클라이언트 인증서를 제시하고  
Nginx에서 `ssl_verify_client on` 정책에 따라 접속 허용/차단이 결정된다.

#### 2.4.4 mTLS 운영 팁

- 인증서 수명
  - 내부 단말용 인증서는 1년 이하로 설정하고,  
    MDM/엔드포인트 관리 도구로 자동 갱신·폐기를 관리하는 것이 바람직하다.
- 인증서 폐기
  - 분실·퇴사·단말 교체 시에는 즉시 CRL/OCSP 응답 혹은 CA 측에서 인증서를 폐지해야 한다.
- 롤아웃 전략
  - “인증서 필수”로 바로 전환하기 전에 **옵션 모드**(`optional`)로 두고
    어느 단말들이 인증서를 쓰고 있는지 관찰하며 점진적으로 강제하는 방식이 안정적이다.

---

## 3. 무선 WIDS/WIPS: WPA3, 802.1X, Evil Twin 대응

이제 **유선 L2**와 **L7 암호화**를 다뤘으니, 무선 환경을 보자.  
무선은 **매체 자체가 공유**되어 있기 때문에 “스니핑”이 근본적으로 더 쉽다.  
따라서 보안 기본값은 더 엄격해야 한다.

### 3.1 무선 위협 모델

주요 위협 요소는 다음과 같다.

- **오픈/약하게 보호된 SSID**
  - WEP, TKIP, 공용 오픈 망 등은 통신 내용이 쉽게 복호화되거나, 전혀 암호화되지 않는다.
- **Evil Twin / Rogue AP**
  - 공격자가 동일 SSID/비슷한 BSSID로 가짜 AP를 띄우고,  
    더 강한 전파/해당 위치 근접 등으로 사용자를 그 AP에 붙게 만든다.
- **Deauth/Disassoc 공격**
  - 관리 프레임이 보호되지 않을 경우, 공격자가 Deauth 패킷을 살포해
    사용자의 연결을 끊고 자신이 원하는 AP로 재연결을 유도할 수 있다.
- **무단 AP 설치**
  - 직원이 개인용 공유기를 가져와 사무실 유선망에 꽂고, 무단 무선망을 만들 수 있다.

### 3.2 WIDS (Wireless Intrusion Detection System)

- **역할**
  - 상시로 무선 스펙트럼을 모니터링하여,
    - Rogue/Evil Twin AP
    - Deauth/Disassoc 폭주
    - WEP/Open SSID
    - 비인가 채널/출력
    등을 탐지한다.
- **동작 방식**
  - 대부분의 엔터프라이즈 AP/컨트롤러는
    - 클라이언트 용 AP와 별개로 “센서 모드” AP를 두거나,
    - 일부 라디오를 스캐닝 용도로 할당하여 주변 BSSID/채널/프레임을 지속적으로 관찰한다.

### 3.3 WIPS (Wireless Intrusion Prevention System)

- **역할**
  - WIDS가 탐지한 이상 행위에 대해 자동으로
    - 해당 AP/클라이언트에 **Deauth**를 날리거나,
    - 스위치 수준에서 포트를 차단하거나,
    - NAC와 연동해 단말을 격리하는 등 **능동적 차단**을 수행한다.
- **권장 운영 모드**
  1. 처음에는 **WIDS 모드(탐지만)**로 시작하여 오탐/운영 영향도를 충분히 검증.
  2. 특정 케이스(예: 확실한 Rogue AP)에 대해서만 선별적으로 WIPS 차단 정책을 적용.

### 3.4 무선 보안 정책 제언

- 암호화/인증
  - **WPA3-Enterprise + 802.1X(EAP-TLS)** 우선.
  - 최소한 **WPA2-Enterprise** 이상, PSK는 가능한 한 줄이는 방향.
- 802.11w(PMF – Protected Management Frames)
  - 관리 프레임 위변조(Deauth, Disassoc 등)에 대한 내성을 높이기 위해 **PMF 필수**.
- SSID 정책
  - SSID 이름/암호화 방식/허용 채널/출력에 대한 **프로파일 화이트리스트**를 관리.
  - 클라이언트 측에서도 **오픈/PSK SSID 자동 연결 금지** 정책을 모바일/PC MDM에 반영.
- BSSID 화이트리스트
  - 동일 SSID라도 **허용된 AP BSSID 목록**을 관리해, WIDS가 이외 BSSID를 Rogue로 탐지도록 설정.
- 스위치 연동
  - 무선 컨트롤러와 유선 스위치/NAC가 연동되면,
    - Rogue AP가 발견된 스위치 포트를 **자동 Shutdown** 하거나,
    - 해당 포트 VLAN을 “격리 VLAN”으로 옮기는 등 신속한 대응이 가능하다.

---

## 4. 탐지: ARP Reply 폭증, GW MAC 변화, DHCP 다중 서버, TLS 메타 (Suricata/Zeek/SIEM)

위에서 본 **L2·L7 방어**는 “공격을 어렵게 만드는 장치”이다.  
그러나 실전에서는 **완벽한 설정**이 불가능하며,  
우회/오탐/미설정 영역을 보완하기 위해 **탐지 체인**이 반드시 필요하다.

- **Suricata**: IDS/IPS/NDR 엔진.
- **Zeek**: 프로토콜 메타데이터/로그 분석에 특화된 네트워크 보안 모니터링 도구.
- **SIEM/NDR 플랫폼**: Suricata/Zeek/스위치/무선 컨트롤러의 로그를 모아서 상관 분석.

### 4.1 Suricata 룰 – ARP Reply 폭증 탐지

#### 4.1.1 개념

- ARP 스푸핑/포이즈닝은 일반적으로 **짧은 시간에 ARP Reply가 급증**하는 패턴을 보인다.
- Suricata 룰에서 `detection_filter`를 사용하면,
  - “특정 src MAC(혹은 IP)에서 N초 동안 M개 이상 ARP Reply” 같은 조건을 정의할 수 있다.

#### 4.1.2 예시 룰 (개념 템플릿)

```conf
# 간단 예시: 짧은 시간 한 소스에서 과도한 ARP Reply
alert ether any any -> any any (msg:"L2 ARP Reply storm suspected";
  ether type 0x806;           # ARP
  # detection_filter는 by_src MAC 추적에서 한계가 있어 환경 맞춤 필요
  detection_filter:track by_src, count 60, seconds 10;
  sid:4202001; rev:1;)
```

- `ether type 0x806` → ARP 프레임 선택.
- `detection_filter:track by_src, count 60, seconds 10`  
  - 10초 동안 동일 src에 대해 60번 이상 매치되면 알림.
- 실제 환경에서는
  - VDI, 대규모 무선 로밍 등에서 **정상적으로도 ARP가 많을 수 있으므로**  
    임계치를 현실적으로 조정해야 한다.

#### 4.1.3 룰 테스트 (pcap 재생)

테스트망에서 다음과 같은 방식으로 pcap을 재생하며 룰을 검증할 수 있다.

```bash
# Suricata를 오프라인 모드로 실행
suricata -r arp_storm_test.pcap -l out/arp_test --runmode=single

# EVE JSON에서 해당 SID 히트 수 확인
jq -r 'select(.alert and .alert.signature_id==4202001) | .timestamp' \
  out/arp_test/eve.json | wc -l
```

이렇게 횟수를 측정해가며 `count`/`seconds` 값을 조정하면,  
“완전한 공격 상황에서는 반드시 울리되, 정상 트래픽에서는 거의 울리지 않는” 임계치를 찾을 수 있다.

---

### 4.2 Suricata 룰 – 게이트웨이 MAC 변동 감지(개념)

게이트웨이 MAC이 갑자기 바뀌는 것은 상당히 강력한 시그널이다.

- 정상 상황
  - 코어 장비 교체, 재부팅, 링크 전환, L2 토폴로지 변경 등.
- 비정상 상황
  - 외부에서 ARP 포이즈닝 시도, 실수/설정오류로 인한 잘못된 이중화 설정 등.

Suricata 룰만으로 “이전 MAC과 다른지”를 기억하는 것은 어렵다.  
따라서 보통은:

1. Suricata에서 “게이트웨이 IP에서 오는 ARP Reply”를 태깅하는 룰을 만들고,
2. 후단에서 SIEM/NDR이 이전 MAC과 비교하여 변화 여부를 판단하는 구조를 사용한다.

#### 4.2.1 ARP Reply 태깅 룰 (개념)

```conf
alert ether any any -> any any (msg:"ARP Reply from gateway IP";
  ether type 0x806;
  content:"\x00\x02"; offset: 6; depth: 2;  # op=2 (reply)
  # 게이트웨이 IP는 환경별로 payload 오프셋 조정 필요
  sid:4202002; rev:1;)
```

EVE JSON 로그에는 아래와 비슷한 필드가 남는다.

```json
{
  "timestamp": "2025-10-23T10:00:00.000Z",
  "event_type": "alert",
  "src_mac": "aa:aa:aa:aa:aa:aa",
  "alert": {
    "signature_id": 4202002,
    "signature": "ARP Reply from gateway IP"
  },
  "arp": {
    "sender_mac": "aa:aa:aa:aa:aa:aa",
    "sender_ip": "10.10.0.1"
  }
}
```

후단 SIEM에서는 `sender_ip == 10.10.0.1`을 키로 사용해  
`sender_mac`이 이전에 기록된 값과 다르면 **High Severity 알림**을 생성하도록 규칙을 만들 수 있다.

---

### 4.3 Zeek 스크립트 – ARP/DHCP/TLS 메타 분석

Zeek는 프로토콜 메타데이터를 고수준 레코드로 뽑아주는 데 특화되어 있다.  
간단한 스크립트로 ARP·DHCP·TLS를 분석해보자.

#### 4.3.1 ARP 게이트웨이 MAC 변경 감지 스크립트

```zeek
# file: arp_gateway_watch.zeek

@load protocols/arp

const gateway_ip: addr = 10.10.0.1;  # 환경에 맞추어 설정
global gw_mac: string &persistent;   # Zeek state persistence 활용

event zeek_init()
    {
    if ( gw_mac == "" )
        print fmt("[GW-ARP] No baseline yet for %s", gateway_ip);
    }

event arp_reply(c: connection, spa: addr, sha: string, tpa: addr, tha: string)
    {
    # spa: sender protocol address(IP), sha: sender hardware address(MAC)
    if ( spa == gateway_ip )
        {
        if ( gw_mac == "" )
            {
            gw_mac = sha;
            print fmt("[GW-ARP] Baseline set %s -> %s", gateway_ip, gw_mac);
            }
        else if ( gw_mac != sha )
            {
            print fmt("[ALERT] Gateway MAC changed! %s: %s -> %s",
                      gateway_ip, gw_mac, sha);
            gw_mac = sha;
            }
        }
    }
```

- `&persistent` 속성을 사용하면 Zeek 재시작 후에도 `gw_mac` 상태를 유지할 수 있다.
- 실제 운영에서는 `print` 대신 `NOTICE`/`LOG` 메커니즘을 통해  
  SIEM/로그 수집기로 알림을 보낸다.

#### 4.3.2 DHCP 다중 서버 관찰 스크립트

```zeek
# file: dhcp_multi_server_watch.zeek

@load protocols/dhcp

global dhcp_servers: set[addr] &create_expire=10mins;

event dhcp_offer(c: connection, msg: dhcp_msg, yiaddr: addr,
                 siaddr: addr, subnet_mask: addr,
                 router: dhcp_option_list)
    {
    add dhcp_servers[siaddr];

    if ( |dhcp_servers| > 1 )
        {
        print fmt("[ALERT] Multiple DHCP servers observed (last 10 mins): %s",
                  dhcp_servers);
        }
    }
```

- `siaddr` 필드는 DHCP 서버의 IP를 가리킨다.
- 10분 동안 2개 이상의 DHCP 서버가 관찰되면 알림을 발생시킨다.
- 실제 운영에서는
  - “테스트용 DHCP가 특정 서브넷에 한시적으로 배치된 상황” 등 예외를 고려해  
    화이트리스트/스케줄을 함께 관리해야 한다.

#### 4.3.3 TLS 메타/JA3 분석의 개념

Zeek는 기본적으로 `ssl.log`(또는 최신 버전에서는 `tls.log`)에 다음과 같은 정보를 남긴다.

- SNI (Server Name Indication)
- TLS 버전
- Cipher Suite
- JA3 / JA3S Fingerprint
- 인증서 Subject/Issuer/Validity 등

이 정보를 SIEM에서 집계하면:

- 비정상적으로 오래된 TLS 버전/암호군을 사용하는 세션
- 알려진 악성 인프라와 유사한 JA3/JA3S Fingerprint
- 일반 사용자 PC에서 보기 어려운 “이상한 클라이언트” (특수 스캐너 등)

을 쉽게 찾아낼 수 있다.

---

### 4.4 SIEM 룰 설계 (의사코드/Sigma 스타일)

SIEM에서는 Suricata/Zeek/스위치/Wi-Fi 컨트롤러 로그를 **상관 분석**한다.  
아래는 개념적인 룰 예시이다.

#### 4.4.1 게이트웨이 MAC 변경

```text
title: Gateway MAC Address Change
logsource:
  product: zeek
  category: network
detection:
  selection:
    event: GW_MAC_CHANGE
  condition: selection
level: high
```

- Zeek 스크립트에서 MAC 변경 시 `GW_MAC_CHANGE` 이벤트를 발생시키도록 구현하고,
- SIEM은 이를 High Severity 알림으로 처리한다.

#### 4.4.2 ARP Reply 폭증

```text
title: ARP Reply Storm by Source
logsource:
  product: suricata
  category: network
detection:
  selection:
    event_type: alert
    signature_id: 4202001     # "L2 ARP Reply storm suspected"
  condition: selection
level: medium
```

- 향후 튜닝
  - 특정 VLAN/서브넷/망 구간에서만 이 룰을 활성화하거나,
  - VDI/무선 로밍 등 정당한 ARP 폭증 패턴을 화이트리스트 처리한다.

#### 4.4.3 DHCP 다중 서버

```text
title: Multiple DHCP Servers in VLAN
logsource:
  product: zeek
  category: network
detection:
  selection:
    event: DHCP_MULTI_SERVER
  condition: selection
level: high
```

- Zeek의 `dhcp_multi_server_watch.zeek` 스크립트가  
  `DHCP_MULTI_SERVER` 이벤트를 생성하도록 하고,
- SIEM에서 해당 이벤트를 High Severity로 처리.

---

## 5. 랩: L2 ARP 기반 가로채기, 무선 오픈망 캡처, 탐지 룰 제작 (공격 재현 없이)

> 핵심 취지: **실제 공격 행위를 하지 않고도**,  
> “탐지/방어가 제대로 동작하는지”를 검증할 수 있는 랩 환경을 만든다.  
> PCAP 재생, 합성 이벤트, **무해한 ARP 변동** 등을 활용한다.

절대 **자신이 통제하지 않는 네트워크나 허가받지 않은 환경**에서  
ARP 변조/무선 공격 실험을 해서는 안 된다.

---

### 5.1 네임스페이스 기반 미니 랩 토폴로지

리눅스 네임스페이스를 활용한 단일 호스트 랩 구조 예:

```text
[ns-client]──veth-c  \
                      (br0)──veth-g──[ns-gw]
[ns-observer]──veth-o /
```

- `ns-client`: 트래픽을 발생시키는 클라이언트.
- `ns-gw`: 라우터/게이트웨이 역할.
- `ns-observer`: tcpdump/Zeek/Suricata를 실행하는 패시브 캡처 전용.

#### 5.1.1 네임스페이스 생성 스크립트 예시

```bash
#!/usr/bin/env bash
set -e

# 네임스페이스 생성
ip netns add ns-client
ip netns add ns-gw
ip netns add ns-observer

# 브리지와 veth 생성
ip link add br0 type bridge

ip link add veth-c type veth peer name veth-c-br
ip link add veth-g type veth peer name veth-g-br
ip link add veth-o type veth peer name veth-o-br

ip link set veth-c netns ns-client
ip link set veth-g netns ns-gw
ip link set veth-o netns ns-observer

ip link set veth-c-br master br0
ip link set veth-g-br master br0
ip link set veth-o-br master br0

ip link set br0 up
ip link set veth-c-br up
ip link set veth-g-br up
ip link set veth-o-br up

# IP 할당
ip netns exec ns-client ip addr add 10.10.0.10/24 dev veth-c
ip netns exec ns-gw     ip addr add 10.10.0.1/24  dev veth-g
ip netns exec ns-observer ip addr add 10.10.0.99/24 dev veth-o

ip netns exec ns-client ip link set veth-c up
ip netns exec ns-gw     ip link set veth-g up
ip netns exec ns-observer ip link set veth-o up

# 기본 게이트웨이 설정
ip netns exec ns-client ip route add default via 10.10.0.1
```

이제:

- `ns-client`에서 ping/curl 등 트래픽을 발생.
- `ns-observer`에서 `tcpdump -i veth-o -nn -e -w /tmp/lab.pcap`으로 캡처.
- `ns-gw`에서는 간단한 iptables/라우팅 규칙을 넣어 NAT 등 실험을 할 수 있다.

---

### 5.2 시나리오 1 – “L2 ARP 기반 가로채기” **탐지만** 검증

실제 ARP 포이즈닝 공격은 하지 않고,  
**무해한 GARP(Gratuitous ARP) 반복**으로 “ARP 변동” 신호만 발생시켜  
DAI/Suricata/Zeek/SIEM의 룰이 제대로 울리는지 확인한다.

#### 5.2.1 GARP 생성 파이썬 스크립트 (예시)

아래 예시는 **Scapy**를 사용해 “자기 자신 IP/MAC에 대한 GARP”만 반복 전송한다.  
실제 게이트웨이 IP/MAC을 위조하지 않으며, **자기 ARP 정보를 주기적으로 알리는 수준**이다.

```python
#!/usr/bin/env python3
# safe_arp_churn_generator.py
#
# 목적: 자신의 IP/MAC로 Gratuitous ARP를 반복 전송해,
#       ARP 트래픽이 증가했을 때 탐지 체인이 어떻게 반응하는지 확인한다.

from scapy.all import ARP, Ether, sendp
import time

def main():
    iface = "veth-c"          # ns-client 내 인터페이스 이름
    src_ip = "10.10.0.10"     # 자신의 IP
    src_mac = "00:11:22:33:44:55"  # 자신의 MAC (ip link show로 확인 후 수정)
    interval = 0.2            # 초 단위 간격
    count = 50                # 총 전송 횟수

    pkt = Ether(dst="ff:ff:ff:ff:ff:ff", src=src_mac) / ARP(
        op=2,                 # reply
        hwsrc=src_mac,
        psrc=src_ip,
        hwdst="00:00:00:00:00:00", # 널 MAC
        pdst=src_ip
    )

    print(f"Sending {count} GARP packets on {iface}...")
    for i in range(count):
        sendp(pkt, iface=iface, verbose=False)
        time.sleep(interval)

    print("Done.")

if __name__ == "__main__":
    main()
```

> 주의  
> - 이 스크립트는 “자기 자신” 정보만 뿌리므로, 게이트웨이 ARP 캐시를 악의적으로 오염시키지 않는다.  
> - 그래도 프로덕션 환경이 아닌 **랩/테스트 네트워크**에서만 실행해야 한다.

#### 5.2.2 테스트 절차 개요

1. `ns-observer`에서 ARP 트래픽 캡처 시작  
   ```bash
   ip netns exec ns-observer tcpdump -i veth-o -nn -e -w /pcap/arp_test.pcap 'arp'
   ```
2. `ns-client` 네임스페이스에서 `safe_arp_churn_generator.py` 실행.
3. Suricata/Zeek가 `arp_test.pcap`을 오프라인으로 분석.
4. “ARP Reply storm” 룰/Zeek 스크립트/ SIEM 룰이 예상대로 울리는지 확인.

---

### 5.3 시나리오 2 – “무선 오픈망 캡처(패시브)” + WIDS 룰 검증

**공격을 수행하지 않고**, 자신이 소유/관리하는 **테스트 AP**만 사용해서  
오픈 SSID 혹은 다른 설정에서 관리 프레임/데이터 프레임 메타만 관찰한다.

#### 5.3.1 절차 개요

1. 무선 테스트 장비 준비 (모니터 모드 지원 카드).
2. 테스트용 AP에 오픈 SSID 또는 약한 암호화(WEP/TKIP) 설정.  
   (테스트 환경에서만, 외부 인터넷 연결 없이 사용)
3. `tshark`/`Wireshark`로 관리 프레임 캡처.
4. 캡처 로그를 기반으로 WIDS/SIEM에 **합성 이벤트**를 주입해
   - Deauth 폭주, WEP/Open SSID 탐지 룰의 동작 여부를 검증.

#### 5.3.2 합성 Deauth 이벤트 생성기 (의사 코드)

```python
#!/usr/bin/env python3
# synth_deauth_events.py
#
# 목적: 실제 무선 공격을 하지 않고도,
#       WIDS/SIEM이 Deauth 폭주 이벤트를 어떻게 처리하는지 테스트할 수 있는
#       NDJSON 형식의 합성 이벤트를 생성한다.

import json
import time
import random

def main():
    count = 100
    ts_base = int(time.time())

    for i in range(count):
        ev = {
            "@timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime(ts_base + i)),
            "event_type": "WIFI_DEAUTH_OBSERVED_SYNTH",
            "src_bssid": "aa:bb:cc:dd:ee:ff",
            "dst_mac": "11:22:33:44:55:66",
            "channel": 6,
            "rssi": -40 + random.randint(-5, 5),
            "count_in_burst": i
        }
        print(json.dumps(ev))

if __name__ == "__main__":
    main()
```

생성된 NDJSON을 WIDS/SIEM에 테스트 인제션 파이프라인으로 넣어,  
해당 이벤트에 대한 룰이 정상적으로 발화하는지 확인할 수 있다.

---

### 5.4 회귀 테스트 자동화 – “룰 업데이트가 기존 탐지를 깨지 않는지”

탐지 엔진(룰셋)을 업데이트할 때마다,  
기존에 잘 검출되던 시나리오가 **조용히 사라지는 회귀(regression)** 를 막기 위해  
간단한 자동화 스크립트를 둘 수 있다.

#### 5.4.1 테스트 데이터셋 구성

- `pcap/arp_churn_ok.pcap` : GARP 10회 (정상/경계 상황)
- `pcap/dhcp_dual_offer.pcap` : 두 서버가 Offer를 보내는 합성 pcap
- `pcap/tls_normal.pcap` : 정상 브라우저 TLS 트래픽
- `pcap/tls_ja3_suspicious.pcap` : 의심스러운 JA3 Fingerprint 세션들

#### 5.4.2 Suricata EVE 기반 회귀 테스트 스크립트 (예시)

```bash
#!/usr/bin/env bash
set -euo pipefail

PCAP_DIR="pcap"
OUT_DIR="out"

mkdir -p "$OUT_DIR"

run_test() {
  local pcap="$1"
  local sid="$2"
  local min_hits="$3"
  local name="$4"

  echo "== Running test: $name =="

  suricata -r "$PCAP_DIR/$pcap" -l "$OUT_DIR/$name" --runmode=single >/dev/null 2>&1

  local hits
  hits=$(jq -r "select(.alert and .alert.signature_id==$sid)" \
    "$OUT_DIR/$name/eve.json" | wc -l)

  if [ "$hits" -lt "$min_hits" ]; then
    echo "[FAIL] $name: expected at least $min_hits hits for SID $sid, got $hits"
    exit 1
  else
    echo "[PASS] $name: $hits hits (SID $sid)"
  fi
}

run_test "arp_churn_ok.pcap"         4202001 1 "arp_storm"
run_test "dhcp_dual_offer.pcap"      4202003 1 "dhcp_multi_server"
run_test "tls_ja3_suspicious.pcap"   4202004 1 "tls_ja3_suspicious"

echo "All tests passed."
```

- Suricata 룰셋이 바뀔 때마다 이 스크립트를 CI 파이프라인에서 실행해,
  - 특정 시나리오에 대한 탐지가 완전히 사라졌는지 여부를 빠르게 확인할 수 있다.

---

## 6. 운영 반영 체크리스트 (L2·L7·무선·탐지·프로세스)

실제 운영에 반영할 때 고려해야 할 항목들을 영역별로 정리해보자.

### 6.1 L2 스위치 보안

- [ ] **DHCP Snooping**
  - [ ] VLAN별 활성화 여부 확인 (유저 VLAN만, 서버 VLAN은 예외 등)
  - [ ] Trusted 포트 목록 정의 (업링크, DHCP 서버 연결 포트)
  - [ ] Rate limit 설정 (Offer/ACK 폭주 방지)
  - [ ] Snooping 바인딩 테이블 자동 수집/점검(정적 IP 단말 누락 여부) 프로세스 정의

- [ ] **DAI (Dynamic ARP Inspection)**
  - [ ] Snooping/Source Binding 없이는 동작하지 않는다는 점을 이해하고 설계
  - [ ] VLAN별 활성화
  - [ ] 정적 IP 단말 바인딩 수동 등록 절차 문서화
  - [ ] ARP rate limit 값 선정(VDI·무선 로밍 환경 고려)

- [ ] **Port Security**
  - [ ] 포트 그룹(서버/사용자/VoIP/AP 등)별 `maximum`/`violation` 정책 설계
  - [ ] Sticky MAC 사용 시, 운영 절차(포트 변경/단말 교체/재이미징 등) 문서화
  - [ ] 위반 발생 시 알림/자동 티켓 생성/현장 대응 프로세스 연결

### 6.2 L7 암호화 (HTTPS/HSTS/mTLS)

- [ ] **HTTPS**
  - [ ] 전 서비스 도메인에 HTTPS 적용 여부 확인
  - [ ] TLS 1.0/1.1 비활성화, TLS 1.2/1.3 중심 구성
  - [ ] 암호군 구성 검토 (취약/레거시 cipher 제거)
  - [ ] OCSP/CRL 등 인증서 폐기 메커니즘 검토

- [ ] **HSTS**
  - [ ] HSTS 헤더 적용 범위 (서브도메인 포함 여부)
  - [ ] Preload 도메인 vs 비-Preload 도메인 구분
  - [ ] 롤백이 어려운 점에 대한 문서화 및 Change Management 절차 마련

- [ ] **mTLS**
  - [ ] 내부 API/관리 콘솔에서 mTLS 적용 대상 선정
  - [ ] 사내 CA/PKI 구조, 클라이언트 인증서 발급/폐기 프로세스 정의
  - [ ] 클라이언트·서버 측 라이브러리/프레임워크의 인증서 검증 설정 확인
  - [ ] 롤아웃 단계: optional → on 순 증강 전략

### 6.3 무선(Wi-Fi) 보안

- [ ] **암호화/인증**
  - [ ] WPA3-Enterprise 우선, 불가 시 WPA2-Enterprise
  - [ ] 802.1X(EAP-TLS/EAP-PEAP 등) 인증 방식 설계
- [ ] **WIDS/WIPS**
  - [ ] WIDS 모드로 운영 후 오탐 분석
  - [ ] Rogue AP/Evil Twin/Deauth 폭주 등 WIPS 자동 차단 대상 선정
  - [ ] WIPS 차단 범위·법적/규제 측면 검토
- [ ] **PMF(802.11w)**
  - [ ] 지원 단말 비율 확인 후 Mandatory/Optional 정책 결정
- [ ] **SSID/BSSID 정책**
  - [ ] SSID 이름/암호화 방식/허용 채널 정리
  - [ ] BSSID 화이트리스트 관리
  - [ ] 오픈/PSK SSID 자동 연결 금지 정책의 엔드포인트 반영

### 6.4 탐지·로깅·분석

- [ ] **Suricata/Zeek 배치**
  - [ ] 미러 포트/TAP 위치 선정 (북-남, 동-서 모두 고려)
  - [ ] TLS 복호화 여부/범위 결정 (법·규제·성능 고려)
- [ ] **SIEM/NDR 통합**
  - [ ] ARP/DHCP/TLS/DNS/Wi-Fi 로그를 공통 스키마로 정규화
  - [ ] 게이트웨이 MAC 변경/ARP Reply 폭증/DHCP 다중 서버 등 핵심 룰 셋 구성
  - [ ] JA3/JA3S/TLS 버전·암호군 기반 탐지 룰 설계
- [ ] **회귀 테스트**
  - [ ] pcap·합성 이벤트 세트 정의
  - [ ] 룰셋 업데이트 시 자동 회귀 테스트 파이프라인 구축

### 6.5 프로세스/운영 측면

- [ ] **“탐지 → 알림 → 대응” 체인**
  - [ ] 알림 발생 후 실제로 어떤 팀이 무엇을 언제까지 해야 하는지 R&R 정의
  - [ ] 포렌식/증적 보존 절차 마련
- [ ] **오탐 관리**
  - [ ] 특정 기간 동안의 알림을 분류·태깅하여 “정상 패턴”과 “진짜 위협”을 구분
  - [ ] 오탐 패턴에 대한 화이트리스트·튜닝 전략 수립
- [ ] **정기 리뷰**
  - [ ] L2/L7/Wi-Fi 설정과 탐지 룰을 정기적으로 리뷰하고,  
    인프라 변화(새 VLAN, 새 서비스, 새 지사 등)에 따라 갱신

---

## 7. 결론/요약 – L2·L7·무선·탐지까지 한 번에 묶기

마지막으로 지금까지의 내용을 압축해 보자.

1. **L2 방어 삼총사**
   - **DHCP Snooping**: DHCP 서버가 있는 방향만 trusted로 지정하고,
     IP–MAC–VLAN–Port 바인딩 테이블을 만든다.
   - **DAI (Dynamic ARP Inspection)**: Snooping 바인딩에 맞지 않는 ARP는 드롭해  
     게이트웨이 ARP 위조를 어렵게 만든다.
   - **Port Security**: 한 포트에 들어오는 MAC 수/패턴을 제한해  
     MAC 플러딩과 무단 AP/스위치 연결을 초기에 차단한다.

2. **L7 방어**
   - **HTTPS + HSTS**로 평문 HTTP를 없애고,
   - **mTLS**로 내부 API/관리 콘솔에 클라이언트 인증서 없이는 접근하지 못하게 한다.
   - TLS 버전/암호군/인증서 수명/자동 갱신 파이프라인까지 포함해야 실제 운영에서 안전하다.

3. **무선 영역**
   - **WPA3-Enterprise + 802.1X(EAP-TLS)**, **PMF(802.11w)** 활성화로  
     데이터·관리 프레임 모두를 보호한다.
   - **WIDS/WIPS**로 Rogue/Evil Twin/Deauth 폭주를 탐지·선별 차단한다.
   - SSID/BSSID/채널/출력 등 프로파일을 관리해 “허용된 무선 인프라”를 명확히 규정한다.

4. **탐지 체인**
   - Suricata 룰로 **ARP Reply 폭증, 게이트웨이 ARP 패턴**을 감시한다.
   - Zeek 스크립트로 **게이트웨이 MAC 변화, DHCP 다중 서버, TLS 메타**를 관찰한다.
   - SIEM/NDR에서 이 신호들을 상관 분석해,
     **“실제 위협 패턴”과 “단순 이상 패턴”**을 구분해낸다.

5. **랩과 회귀 테스트**
   - 네임스페이스·Docker·합성 이벤트·pcap 재생을 활용해,
     프로덕션을 건드리지 않고 탐지/대응 파이프라인을 검증한다.
   - Suricata/Zeek 룰셋 업데이트 시마다 회귀 테스트를 돌려  
     기존 탐지 품질이 깨지지 않았는지 확인한다.

6. **운영 팁**
   - **탐지만 울리고 끝나는 시스템**이 아니라,  
     룰 → 알림 → **조치(격리/차단/원복)**까지 연결된 체인을 설계해야 한다.
   - ARP/DHCP/TLS/Wi-Fi는 환경별 특성이 강하므로,  
     **베이스라인 측정–오탐 분석–튜닝–재측정**의 반복이 필수다.
   - 평문 트래픽과 무방비 무선 구간을 최대한 제거하여,  
     공격자에게 **“할 수 있는 것 자체가 적은”** 네트워크를 만들자.

이 글에서 다룬 DHCP Snooping/DAI/Port Security, HTTPS/HSTS/mTLS, WIDS/WIPS,  
그리고 Suricata/Zeek/SIEM 기반 탐지 랩은  
현대 기업 네트워크에서 “**스니핑/가로채기 시도 자체를 어렵게 만들고,  
설령 시도가 있어도 빠르게 눈치채는**” 최소한의 프레임워크라 할 수 있다.

이 프레임워크를 기반으로, 조직의 규모·규제·위협 모델에 맞게 룰을 확장하고  
자동화·관측성을 더해 나가면 된다.