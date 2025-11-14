---
layout: post
title: 웹해킹 - 서브도메인 테이크오버 / Dangling DNS
date: 2025-10-10 16:25:23 +0900
category: 웹해킹
---
# 서브도메인 테이크오버 / Dangling DNS

**— 개념 · 공격 시나리오 · 안전 재현(막혀야 정상) · 클라우드/웹 서비스별 특징 · 즉시 적용 방어(동기화·퍼지 스캔·정책) · IaC/코드 예제 · 운영 체크리스트**

## 한눈에 보기 (Executive Summary)

- **정의**
  **서브도메인 테이크오버**는 `app.example.com` 같은 호스트가 **CNAME/ALIAS/NS** 등으로 **외부 리소스**를 가리키는데,
  **그 리소스가 존재하지 않거나(삭제됨/미개설)** “아직 누구에게도 **바인딩되지 않은 상태**”일 때
  공격자가 **동일 제공자(예: Vercel/Heroku/Netlify/GitHub Pages 등)**에서 **그 이름을 자신 소유 리소스에 연결**해
  `app.example.com` **콘텐츠를 장악**하는 문제입니다. 이러한 부실 참조를 **Dangling DNS**라 부릅니다.

- **핵심 원인**
  1) **CNAME가 비존재 리소스**(예: `app.herokuapp.com` 미배포 앱, `foo.s3-website-...` 버킷 없음)
  2) **서드파티 '커스텀 도메인' 검증 미비**(TXT 검증 없이 CNAME만으로 연결 허용)
  3) **클라우드 자원 삭제 ↔ DNS 레코드 미삭제**(IaC 부재/운영 공백)
  4) **NS Delegation Dangling**: `dev.example.com`을 NS로 **별도 위임했는데** 대상 네임서버 소유권 방치

- **핵심 방어**
  - **DNS·클라우드 자원 동기화**: 리소스 **생성·삭제와 DNS**를 **한 번의 IaC 변경**으로 묶기
  - 주기적 **퍼지(가비지) 스캔**: **모든 레코드**를 실제 타겟과 대조(HTTP 시그니처/서비스별 검사)
  - **짧은 TTL** + **운영 플래그 ‘드레인 모드’**(삭제 전 점검)
  - **“소유권 증명” 강제**: TXT 검증 토큰 기반 **커스텀 도메인 바인딩**만 허용(사내 지침)
  - **위임(Delegation) 최소화** + **NS/Glue 동기화** + **야생 NS 탐지**
  - **Wildcards 관리**: 와일드카드가 문제를 가리거나 확대하지 않도록 **정책화**
  - **CT(인증서 투명성)·로그 모니터링**: 새로운 인증서/콘텐츠 등장 **조기 경보**

---

## 공격 시나리오(개념 → 안전 재현 포인트)

### CNAME → 비존재 리소스

```
app.example.com   CNAME   app.herokuapp.com      # 삭제된 앱
files.example.com CNAME   nonexistent.s3-website-ap-northeast-2.amazonaws.com
static.example.com CNAME  wildcard-project.vercel.app  # 해당 프로젝트에서 미등록
```
- **위험**: 공격자가 **동일 제공자 패널**에서 자신의 앱/프로젝트에 `app.example.com` 을 **연결(Claim)** → 장악.
- **안전 재현(막혀야 정상)**: 해당 제공자가 **소유권 검증(TXT/HTTP 파일)**을 **꼭 요구**해 **타인이 Claim 불가**하거나,
  Dangling 레코드 자체가 **스캔 시 즉시 제거/차단**되어야 함.

### NS Delegation Dangling (하위 도메인 위임)

```
dev.example.com   NS      ns1.old-dns.com.
                  NS      ns2.old-dns.com.       # old-dns 소유권 상실 or 계정 만료
```
- **위험**: 공격자가 `old-dns.com` 측 네임서버를 **인수/구매/등록**하면 **dev.example.com 전체**를 조작.
- **안전 재현**: 사내 스캐너가 `*.example.com`의 **NS 레코드**를 수집해 **권한 보유 여부**를 정기 확인.
  소유권 불분명/만료 임박은 **차단/이관** 워크플로로.

### “프로바이더 시그니처” 기반 탐지

- 예: **Heroku**: `No such app` / **GitHub Pages**: `There isn't a GitHub Pages site here.`
  **S3 Website**: XML `NoSuchBucket` / **Azure**: `404 Web Site not found` 등.
- **안전 재현**: 탐지 스크립트가 HTTP로 접근 → **이런 문구/헤더** 탐지 시 **알람 + 자동 PR**로 레코드 제거.

> ※ **실제 텍스트는 변동 가능**하므로 **공급자 공지/운영 로그**로 주기 유지보수(시그니처 룰 업데이트)가 필요합니다.

---

## 방어 아키텍처

1) **IaC 중심 동기화**
   - Terraform/CloudFormation/ARM 등으로 **DNS 레코드와 해당 리소스**를 **같은 모듈**에서 생성/삭제.
   - 리소스 Destroy 시 **레코드가 먼저 제거**되도록 `depends_on` 설계.

2) **퍼지(가비지) 스캔 파이프라인**
   - 모든 존의 `A/AAAA/CNAME/NS/ALIAS`를 크롤링 → 실제 응답/HTTP 배너/WHOIS/네임서버 상태 점검.
   - **Dangling 후보 목록** 자동 생성 → **Issue/PR** 오픈(“이 레코드 삭제/이관” 제안).

3) **소유권 검증 강제 지침**
   - 사내 표준: **TXT 검증 토큰**(예: `_vercel`, `_github-pages-challenge`, `_acme-challenge`)이 **없으면**
     해당 서브도메인을 **어떤 SaaS에도 연결 금지**.

4) **삭제·이관 절차**
   - 서비스 종료 시: **(1) DNS TTL ↓ → (2) SaaS 바인딩 해제 → (3) 레코드 제거** 순서.
   - 긴급: **와일드카드 Parking(대응 도메인)** + **HTTP 410** 서빙 → 오남용 차단.

5) **모니터링**
   - **CT 로그**(예: 특정 서브도메인에 새 인증서 발급) → **경보**.
   - **변조 탐지**: 스테이터스 페이지/헬스 체크 URL **해시** 기준 상이 시 경보.

---

## 운영팀용 “막혀야 정상” 점검 시나리오

- [ ] **삭제된 리소스 CNAME**을 만들고(스테이징) → **사내 스캐너가 24h 내 자동 PR** 생성?
- [ ] **TXT 검증 토큰 없이** CNAME만 배포 → **SaaS 측 바인딩 실패**가 기본 정책인가?
- [ ] **하위 도메인 위임 NS**를 임시로 **불능 네임서버**로 바꿨을 때 → 스캐너가 **Dangling NS 경보**?
- [ ] **CT 모니터**가 신규 인증서에 **우리 도메인**이 등장하면 슬랙/이메일 알림?

---

## 툴링: 빠른 수동 점검 (현장 명령어)

```bash
# 레코드 확인

dig +nocmd app.example.com CNAME +noall +answer
host app.example.com

# 대상 호스트 존재 여부(예: S3 Website)

dig nonexistent-bucket.s3-website-ap-northeast-2.amazonaws.com

# HTTP 배너/시그니처

curl -i http://app.example.com
curl -i https://app.example.com

# NS Delegation

dig dev.example.com NS +short
whois old-dns.com | egrep -i 'Registrar|Expiry|Status'
```

---

## 자동 스캐너 (Python, 비침투형)

> **기능**:
> (1) 존의 CNAME/NS 수집 → (2) HTTP/HTTPS 배너 확인 → (3) 서비스 시그니처 매칭 → (4) Dangling 후보 리포트.
> 실제 운영에서는 **DNS 제공자 API**(Route53/Cloudflare 등)로 레코드 목록을 가져오는 부분을 추가하세요.

```python
# scan_dangling_dns.py

import asyncio, re, socket, ssl
from contextlib import closing

SIGNATURES = [
    ("heroku", re.compile(r"No such app", re.I)),
    ("github_pages", re.compile(r"There isn.?t a GitHub Pages site here", re.I)),
    ("s3_nosuchbucket", re.compile(r"NoSuchBucket", re.I)),
    ("azure_app", re.compile(r"Azure.+(not|no) (found|configured)", re.I)),
    ("netlify", re.compile(r"Not Found - Request ID", re.I)),
    ("vercel", re.compile(r"DEPLOYMENT_NOT_FOUND|Project Not Found", re.I)),
]

TARGETS = [
  # (host, cname_target or None)
  ("app.example.com", None),
  ("static.example.com", None),
  ("dev.example.com", None),   # NS 위임 검사는 별도 함수에서
]

async def fetch_head(host, https=True, timeout=5):
    port = 443 if https else 80
    req = f"GET / HTTP/1.1\r\nHost: {host}\r\nUser-Agent: ddscan/1.0\r\nConnection: close\r\n\r\n".encode()
    try:
        with closing(socket.create_connection((host, port), timeout=timeout)) as sock:
            if https:
                ctx = ssl.create_default_context()
                with closing(ctx.wrap_socket(sock, server_hostname=host)) as ssock:
                    ssock.sendall(req)
                    return ssock.recv(4096, socket.MSG_WAITALL)
            else:
                sock.sendall(req); return sock.recv(4096, socket.MSG_WAITALL)
    except Exception as e:
        return b""

def check_signature(body: bytes):
    text = body.decode(errors="ignore")
    hits = [name for name, rx in SIGNATURES if rx.search(text)]
    return hits

async def main():
    findings = []
    for host, _ in TARGETS:
        body = await fetch_head(host, True)
        sig = check_signature(body)
        if not sig:
            body = await fetch_head(host, False)
            sig = check_signature(body)
        if sig:
            findings.append((host, sig))
    print("# Dangling candidates:")
    for h, sigs in findings:
        print(f"- {h} -> signatures: {','.join(sigs)}")

if __name__ == "__main__":
    asyncio.run(main())
```

> **확장**
> - **CNAME 체인 추적**: `dnspython` 으로 `CNAME` 타겟 수집
> - **NS Delegation 검사**: 위임 네임서버의 **소유권/만료/권한** 확인
> - **CI 통합**: 결과가 비어있지 않으면 **빌드 실패/PR 코멘트**.

---

## IaC(예: Terraform) — 리소스·DNS 동수명(Lifecycle) 설계

### AWS S3 Static + CloudFront(권장: 직접 S3 Website CNAME 지양)

> **목표**: 서브도메인은 **CloudFront 배포(고유 도메인)**를 가리키고,
> S3 버킷은 **OAC(Origin Access Control)** 로 **직접 접근 금지**. CloudFront 배포가 삭제되면
> 동일 도메인을 공격자가 **재사용할 수 없으므로** 테이크오버 표면을 **크게 축소**합니다.

```hcl
# s3.tf

resource "aws_s3_bucket" "site" {
  bucket = "example-site-prod"
  force_destroy = false
}

# cloudfront.tf

resource "aws_cloudfront_origin_access_control" "oac" {
  name                              = "site-oac"
  description                       = "OAC for S3 site"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

resource "aws_cloudfront_distribution" "cdn" {
  enabled = true
  aliases = ["www.example.com"]   # 커스텀 도메인
  origin {
    domain_name              = aws_s3_bucket.site.bucket_regional_domain_name
    origin_id                = "s3-site"
    origin_access_control_id = aws_cloudfront_origin_access_control.oac.id
  }
  # ... (기타 캐시/정책)
  restrictions { geo_restriction { restriction_type = "none" } }
  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate.cert.arn
    ssl_support_method  = "sni-only"
  }
}

# route53.tf

resource "aws_route53_record" "www" {
  zone_id = var.zone_id
  name    = "www.example.com"
  type    = "A"
  alias {
    name                   = aws_cloudfront_distribution.cdn.domain_name
    zone_id                = aws_cloudfront_distribution.cdn.hosted_zone_id
    evaluate_target_health = false
  }
  depends_on = [aws_cloudfront_distribution.cdn] # 배포 없으면 레코드 생성 안 함
}
```

### “리소스 삭제 → DNS 삭제” 보장

```hcl
# 안전장치: 폐기 시 레코드 우선 삭제

lifecycle {
  # prevent_destroy = true  # 운영 정책에 따라, 무분별 삭제 방지
  ignore_changes = []       # 변경 추적 일관성
}
# 모듈 설계에서 'output'으로 distribution 존재 여부를 체크해 레코드 생성 분기

```

> **권장 패턴**: **S3 Website 엔드포인트를 직접 CNAME으로 노출**하지 말고,
> **CloudFront 배포(유일 식별자)** 앞단에 두세요. 많은 테이크오버 사례가 **직접 Website CNAME**에서 발생합니다.

---

## NS Delegation 위험 줄이기

- **위임 최소화**: 꼭 필요한 서브도메인만 NS 위임.
- **소유권 모니터링**: 위임 대상 네임서버 도메인의 **등록자/만료** 자동 검사.
- **Glue Sync**: 상위/하위 존의 네임서버 **IP(Glue)**가 일치하는지 정기 확인.
- **위임 철회**: 서비스 종료/벤더 종료 시 **즉시 위임 해제** + **와일드카드 Parking**으로 임시 봉쇄.

간단 체크(파이썬):

```python
# ns_delegation_check.py

import dns.resolver, whois

def check_ns(domain):
    ans = dns.resolver.resolve(domain, 'NS')
    hosts = [str(r.target).rstrip('.') for r in ans]
    return hosts

def whois_summary(host):
    try:
        w = whois.whois(host.split('.',1)[1])
        return (w.registrar, w.expiration_date)
    except Exception:
        return (None, None)

for sub in ["dev.example.com", "legacy.example.com"]:
    ns = check_ns(sub)
    print(sub, " -> ", ns)
    for h in ns:
        reg, exp = whois_summary(h)
        print("  ", h, reg, exp)
```

> **운영 정책**: 우리 존을 받는 **서드파티 DNS**는 **계약 종료·계정 폐기** 시 즉시 위임 철회가 자동으로 열리도록 **오프보딩 체크리스트**에 포함.

---

## SaaS(커스텀 도메인) — 안전한 연결 원칙

- **반드시 TXT 검증**(도메인 소유권 토큰) 후에만 연결 승인.
- **CNAME만으로 연결 허용 금지**(사내 표준).
- **이관 절차**:
  1) 새 SaaS에 TXT 추가 → 검증 성공
  2) DNS CNAME 절체(짧은 TTL)
  3) 구 SaaS 바인딩 제거 → TXT 정리
- **정책 문서화**: 어떤 벤더는 “검증 없이 CNAME만” 허용 → **승인 금지** 목록 관리.

---

## 다운로드/렌더링 리스크 제한(피해 최소화)

테이크오버를 **완전히** 막지 못했을 때의 피해를 줄이기 위한 보조수단:

- **CAA 레코드**: 인증서 발급 기관 제한 → 공격자가 손쉽게 유효 TLS 인증서를 받는 것을 어렵게.
- **HSTS(특히 preload)**: 중간자/혼합 콘텐츠 리스크 감소.
- **서브도메인 분리**: 민감 쿠키/로그인 도메인과 **정적/UGC 도메인 분리**(`Auth` 쿠키는 `.example.com` 범위로 주지 않기).
- **다운로드 전용 도메인**: `Content-Disposition: attachment` + `nosniff` 강제.

> 이들은 **근본 해결책이 아니라 피해 경감 수단**입니다. **Dangling DNS 제거**가 최우선.

---

## 지속 퍼지 스캔 — CI/스케줄 예시

### GitHub Actions (매일 1회)

```yaml
name: dangling-dns-scan
on:
  schedule: [{ cron: "13 3 * * *" }]
  workflow_dispatch: {}
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install deps
        run: pip install dnspython httpx
      - name: Run scan
        run: python tools/scan_dangling_dns.py > findings.txt
      - name: Fail on findings
        run: |
          if grep -q "Dangling candidates" findings.txt && \
             [ $(wc -l < findings.txt) -gt 1 ]; then
            echo "::error::Dangling DNS candidates found"; cat findings.txt; exit 1;
          fi
```

### 결과 처리

- **Fail 시 PR 차단** + **Slack 알림** → 담당팀이 레코드 삭제/이관.

---

## “현장형” 체크리스트

- [ ] **DNS ↔ 리소스** 동수명(IaC 한 모듈)
- [ ] **S3 Website 직접 CNAME 금지**(CDN/고유 엔드포인트 사용)
- [ ] **모든 SaaS 커스텀 도메인에 TXT 검증 필수**
- [ ] **주기적 퍼지 스캔**(HTTP 시그니처, CNAME 체인, NS Delegation)
- [ ] **Dangling 후보 자동 PR/알람** 파이프라인
- [ ] **NS 위임 최소화**, 위임 대상 **만료/소유권 모니터링**
- [ ] 삭제·이관 절차: **TTL↓ → 바인딩 해제 → 레코드 제거**
- [ ] **CT 로그 모니터링**(새 인증서 발급 감시)
- [ ] **CAA/HSTS/서브도메인 분리**로 피해 반경 축소
- [ ] 문서화: **허용 벤더 목록/검증 요건**, 오프보딩 체크리스트

---

## 안티패턴(피해야 할 것)

- 리소스 삭제 후 **DNS 방치**(가장 흔함)
- “테스트용”으로 만든 **와일드카드 CNAME**(전체 서브도메인 노출)
- **TXT 검증 토큰 없이** CNAME만으로 SaaS 연결
- NS Delegation을 벤더에게 맡기고 **계정/도메인 갱신 관리 부재**
- **S3 Website**를 **공개 버킷 이름=CNAME 호스트** 조합으로 직접 노출
- IaC 없이 **수동 콘솔 작업**으로 생성·삭제

---

## 맺음말

**서브도메인 테이크오버**는 “**DNS 레코드는 남고, 리소스는 사라지는 순간**” 시작됩니다.
**IaC로 동수명**, **정기 퍼지 스캔**, **소유권 검증 강제**라는 3축을 표준으로 삼으면
대부분의 Dangling DNS를 **사전에 제거**하거나 **즉시 탐지**할 수 있습니다.
