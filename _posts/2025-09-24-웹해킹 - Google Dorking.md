---
layout: post
title: 웹해킹 - Google Dorking
date: 2025-09-24 22:25:23 +0900
category: 웹해킹
---
# Google Dorking

## 0) 개념 요지 — Google Dorking이란?
- **정의**: 검색엔진(대표적으로 Google)의 **고급 검색 연산자**를 활용해 **공개적으로 인덱싱된 정보**에서 특정 자산이나 이슈를 빠르게 찾아내는 **OSINT 기법**입니다.  
- **수비 측 가치**:  
  - 우리 조직 도메인/브랜드의 **노출 자산 인벤토리** 파악(섀도 IT, 방치된 서브도메인, Staging 잔재 등)  
  - **데이터 유출 징후** 조기 탐지(공개 문서, 페이지 캐시, 디렉터리 인덱스 등)  
  - **검색엔진 정책/메타태그/robots 설정** 검증  
  - **브랜드 악용·피싱** 모니터링(상표 도용, 스쿼팅 페이지 등)

> 핵심: “**검색엔진이 이미 본**” 공개 데이터만 다루며, **우리의 자산만** 대상으로 삼습니다.

---

## 1) 기본 연산자 101 (수비용)
아래 예시는 모두 **당신의 자산**을 점검한다는 전제에서, **PLACEHOLDER**를 사용합니다.
- `site:YOUR_DOMAIN` — 특정 도메인/서브도메인 범위로 결과를 한정.  
- `-term` — 특정 단어 제외.  
- `"정확 일치"` — 정확 구문 검색.  
- `OR` — 다중 키워드.  
- `inurl:fragment` — URL 경로/쿼리에 포함된 조각.  
- `intitle:fragment` — 페이지 제목 조각.  
- `filetype:EXT` or `ext:EXT` — 확장자별 문서 검색.  
- `before:YYYY-MM-DD` / `after:YYYY-MM-DD` — 대략적 시점 필터.  
- `cache:URL` — Google 캐시(참고용, 캐시가 없을 수 있음).

> ⚠️ **금칙**: 제3자 도메인이나 민감 키워드 조합을 이용한 “**타인 민감정보 수색형**” 쿼리의 생성/자동화는 **금지**합니다.

---

## 2) 방어 시나리오별 “합법·자가점검” 예시
> **모두 YOUR_DOMAIN/YOUR_BRAND 자산에만 적용.**

### 2.1 섀도 자산/예전 환경 노출 파악
- **목표**: 개발/스테이징 페이지, 문서 저장 리포지터리, 임시 공개 자료 등 **운영팀이 인지하지 못한 공개 자산**을 파악.  
- **예시 쿼리(자산 한정)**:
  - `site:YOUR_DOMAIN "staging"`  
  - `site:YOUR_DOMAIN inurl:dev`  
  - `site:YOUR_DOMAIN intitle:"index of"`  ← *디렉터리 인덱싱 흔적 점검(자사만)*  
- **조치**: 발견 시 **접근제어/인덱스 차단**, 또는 **삭제·통합**.

### 2.2 공개 문서·자료 정리(정보 과다 노출)
- **목표**: 공개가 적절한지, 메타데이터(저자/버전)에 민감 요소가 없는지 점검.  
- **예시 쿼리(자산 한정)**:
  - `site:YOUR_DOMAIN filetype:pdf`  
  - `site:YOUR_DOMAIN (filetype:ppt OR filetype:pptx OR filetype:key)`  
- **조치**: 문서 재검토 → 민감자료 **비공개 전환** 또는 **편집 후 재배포**, 필요 시 `X-Robots-Tag: noindex` 적용.

### 2.3 브랜딩·피싱 모니터링(허가된 범위)
- **목표**: 우리 브랜드명과 혼동되는 페이지를 파악(법무/대외대응 영역).  
- **예시 쿼리**: `"YOUR_BRAND" site:KNOWN_MARKETPLACE` (파트너/공식 계정 확정용)  
- **조치**: 정식 제휴/소유 여부 확인, 오탐은 제외, 악용 의심은 **법무 채널**로.

### 2.4 검색엔진 정책 검증
- **목표**: `noindex`/`nofollow`/`robots.txt`가 의도대로 작동하는지 확인.  
- **예시**: `cache:https://YOUR_DOMAIN/some-url` 로 캐시 잔존 확인(없어야 정상인 페이지는 캐시가 없어야 함).

---

## 3) 수비팀 플레이북 — “인덱스 노출”을 줄이는 절차
1) **범위 선언**: 점검할 도메인/서브도메인/오브젝트 저장소 목록.  
2) **기본 인벤토리**: Search Console(소유 도메인)과 사내 자산관리(ADS/CMDB) 대조.  
3) **자가 검색**: 위의 연산자를 **YOUR_DOMAIN**으로만 사용해 스냅샷을 만들고, 발견물(페이지/문서/캐시)을 분류.  
4) **제거·수정**: 사내 프로세스(아래 §6)를 통해 제거 요청, 인덱스 차단, 접근제어 강화.  
5) **지속 모니터링**: 주기적 재검색(수동/자동 알림) + 로그 기반 탐지(§5 스크립트).

---

## 4) 방어 설정 — “보이지 않게, 노출되어도 안전하게”
- **인덱스 제어**  
  - HTML `<meta name="robots" content="noindex,nofollow">`  
  - HTTP `X-Robots-Tag: noindex, nofollow`  
  - `robots.txt`(가이드일 뿐 보안통제는 아님)  
- **접근통제**  
  - 민감 리소스는 **인증이 기본**(사내 VPN/SSO), 공개 페이지도 **최소 권한**  
- **디렉터리 인덱싱 비활성**  
  - Nginx: `autoindex off;`  
  - Apache: `Options -Indexes`  
- **헤더 하드닝**  
  - `X-Content-Type-Options: nosniff`, `Content-Security-Policy`, `Referrer-Policy`  
- **스토리지 정책**  
  - S3/Cloud Storage는 **퍼블릭 금지 기본** + 서빙은 CDN/OAC 통해서만  
- **수명 관리**  
  - 운영 종료 페이지/문서는 **410 Gone** 또는 **정적 tombstone**로 제거, 필요 시 **URL 제거 도구** 사용(§6).

---

## 5) 실전 스크립트 — “내 자산만” 자동 점검
> 아래 코드는 **당신(또는 조직)**이 소유·관리하는 도메인에서만 사용하세요.  
> 검색엔진 스크래핑을 자동화하지 않고, **직접 HTTP로 우리 자산**만 점검합니다.

### 5.1 Python: 노출 가능 파일·디렉터리 빠른 점검
```python
#!/usr/bin/env python3
"""
Self-audit scanner for YOUR DOMAIN (authorized use only).
- 점검: 디렉터리 인덱스, 숨김/백업/메타 파일 노출, 헤더/인덱스 정책
"""
import sys, re, argparse, requests
from urllib.parse import urljoin

COMMON_PATHS = [
    "/", "/.git/", "/.svn/", "/.DS_Store", "/backup/", "/old/", "/tmp/",
    "/logs/", "/admin/", "/.well-known/security.txt",
]
SUSPECT_FILES = [
    ".git/config", ".git/HEAD", "server-status", "readme.md", "changelog.txt",
    "index.php~", "config.php.bak", "db.sql", "dump.sql", "backup.zip",
]

def get(u, timeout=5):
    try:
        return requests.get(u, timeout=timeout, allow_redirects=False)
    except Exception as e:
        return None

def check_dir_listing(resp):
    if not resp: return False
    ct = resp.headers.get("Content-Type","").lower()
    return (resp.status_code == 200 and
            ("index of /" in resp.text.lower() or "directory listing" in resp.text.lower()) and
            "text/html" in ct)

def check_noindex(resp):
    if not resp: return False
    h = resp.headers
    xrobots = h.get("X-Robots-Tag","").lower()
    meta_noindex = "noindex" in resp.text.lower() and "<meta" in resp.text.lower() and "robots" in resp.text.lower()
    return ("noindex" in xrobots) or meta_noindex

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("base", help="Base URL, e.g., https://YOUR_DOMAIN")
    args = ap.parse_args()
    base = args.base.rstrip("/")

    print(f"[i] Scanning base: {base}")
    for p in COMMON_PATHS:
        url = urljoin(base + "/", p.lstrip("/"))
        r = get(url)
        if not r: 
            print(f"[-] {url:60} timeout/error"); 
            continue

        mark = []
        if check_dir_listing(r): mark.append("DIR-LISTING")
        if r.status_code == 200 and any(name in url.lower() for name in (".git","backup","old","tmp","logs")):
            mark.append("SENSITIVE-PATH-200")
        if "nosniff" not in (r.headers.get("X-Content-Type-Options","").lower()):
            mark.append("NO-NOSNIFF")

        tag = ", ".join(mark) if mark else "ok"
        print(f"[{r.status_code}] {url:60} {tag}")

        # 샘플 파일 탐색
        if p == "/":
            for f in SUSPECT_FILES:
                u2 = urljoin(base + "/", f)
                r2 = get(u2)
                if r2 and r2.status_code == 200:
                    print(f"[!] POSSIBLE-LEAK 200 {u2}")

    # robots.txt / security.txt 확인
    for u in (urljoin(base + "/", "robots.txt"), urljoin(base + "/", ".well-known/security.txt")):
        r = get(u)
        if r and r.status_code == 200:
            print(f"[i] {u} exists ({len(r.text)} bytes)")
        else:
            print(f"[-] {u} missing or not 200")

if __name__ == "__main__":
    main()
```

### 5.2 Nginx: 디렉터리 인덱스/숨김 파일 차단, 다운로드 전용 헤더
```nginx
# 디렉터리 인덱싱 금지
autoindex off;

# 숨김 파일/버전 파일/백업 파일 직접 접근 차단
location ~ (^|/)\.(git|svn|hg) { deny all; }
location ~ .*?\.(swp|swo|bak|old|orig)$ { deny all; }

# 민감 문서는 렌더하지 말고 다운로드로
location /publicdocs/ {
  add_header X-Content-Type-Options nosniff;
  add_header Content-Disposition "attachment";
  types { } default_type application/octet-stream;
}
```

### 5.3 robots.txt·메타태그(참고: 보안 통제가 아닌 인덱스 힌트)
```txt
# robots.txt (힌트일 뿐, 접근 제어 아님)
User-agent: *
Disallow: /admin/
Disallow: /tmp/
Disallow: /internal/
Sitemap: https://YOUR_DOMAIN/sitemap.xml
```
```html
<!-- 인덱스 금지(의도적 비공개 페이지에만) -->
<meta name="robots" content="noindex, nofollow">
<!-- 또는 서버에서 -->
<!-- X-Robots-Tag: noindex, nofollow -->
```

### 5.4 로깅으로 “수상한 검색 패턴” 감시(예: Nginx)
```nginx
# user agent / referer 기반의 단순 감시(과도 차단은 지양)
map $http_referer $suspect {
  default 0;
  "~*google\..*q=.*(inurl|intitle|filetype)" 1; # 단순 예시(참고/알림 용도)
}
log_format main '$remote_addr - $time_local "$request" $status '
                '"$http_referer" "$http_user_agent" sus=$suspect';
access_log /var/log/nginx/access.log main;
```

---

## 6) “발견했을 때” 정리·제거 절차(운영 가이드)
1) **소유자 확인**: 문서/페이지의 시스템·데이터 소유자를 식별.  
2) **결정**: (A) 즉시 비공개/삭제, (B) 편집 후 재배포, (C) 공개 유지.  
3) **인덱스 조치**: 비공개 전환 후 404/410 또는 `noindex` 부여 → **검색엔진 제거 요청 도구** 제출.  
4) **로그/사건 대응**: 접근 로그로 과거 열람·지표 확인, 필요 시 알림/IR(Incident Response) 프로세스.  
5) **재발 방지**: 게시 프로세스/CI 파이프라인에 **민감파일 차단 규칙**(예: `.git`, 백업, DB 덤프 등) 추가, 스토리지 정책 강화.

---

## 7) 프레임워크·플랫폼별 체크포인트(요약)
- **CMS(WordPress 등)**: 인덱스 플러그인/SEO 설정 확인, 디버그/백업 플러그인 공개 경로 차단.  
- **정적 호스팅(S3/CF, Pages)**: **퍼블릭 버킷 금지**, OAC/전용 Origin만 허용, 리스트 인덱싱 OFF.  
- **CI/CD**: 산출물에 **디버그·맵·백업 파일** 포함 여부 검사(Job에 denylist).  
- **팀 절차**: 문서 공개 전 **리뷰+메타데이터 제거(저자/경로)**, 민감어 감지(내부명/프로젝트 코드명 등) 룰.

---

## 8) 자주 묻는 질문(FAQ)
**Q1. robots.txt로 숨기면 되나요?**  
A. **아니요.** robots는 “힌트”이며, 과거에 이미 크롤링된 자료나 규약을 무시하는 크롤러/미러에는 효과가 없습니다. **접근제어**가 답입니다.

**Q2. `noindex`만 붙이면 끝?**  
A. 인덱스 제거에 도움이 되지만, **직접 URL 접근**은 여전히 가능합니다. 민감자료는 **인증/권한**으로 보호하세요.

**Q3. Google 캐시에 남은 페이지는?**  
A. 원본을 비공개/삭제하고, `noindex` 또는 404/410 후 **URL 제거 요청**을 하세요. 시간이 지남에 따라 갱신/삭제됩니다.

**Q4. “공격형 쿼리 리스트”가 필요합니다**  
A. 제공할 수 없습니다. 대신 **우리 자산**을 체계적으로 점검하고 **노출면을 줄이는 방법**(본 문서)으로 방어력을 높이세요.

---

## 9) 요약 체크리스트
- [ ] **범위 정의**(우리 도메인/저장소)  
- [ ] **Search Console** 등록·소유권 검증  
- [ ] **자가 검색**(연산자 101로 OUR_DOMAIN만) & 발견물 분류  
- [ ] **인덱스 제어**(`noindex`, X-Robots-Tag) & **디렉터리 인덱스 금지**  
- [ ] **스토리지/서버 정책**(퍼블릭 금지, 다운로드 전용 헤더, 숨김파일 차단)  
- [ ] **자동 점검 스크립트**(§5) 운영  
- [ ] **제거·사건 대응** 프로세스 문서화  
- [ ] **주기적 모니터링**(검색·알림·로그 룰)

---

## 10) 부록 — “내 자산만”을 위한 안전한 쿼리 템플릿 모음
> **모두 YOUR_DOMAIN/YOUR_BRAND에만 사용.** 타 도메인/조합으로 전용해 **남의 민감정보 수색**을 유도하는 사용은 **금지**합니다.

- 전체 인덱스 대략 보기:  
  - `site:YOUR_DOMAIN`
- 공개 문서 범주 파악:  
  - `site:YOUR_DOMAIN (filetype:pdf OR filetype:ppt OR filetype:doc OR filetype:xls)`
- 제목/경로 특정 키워드(내부 자산 운영용 메시지 등) 확인:  
  - `site:YOUR_DOMAIN intitle:"YOUR_INTERNAL_BANNER"`  
  - `site:YOUR_DOMAIN inurl:YOUR_TEAM_PREFIX`
- 오래된 자료(시점 필터) 훑기:  
  - `site:YOUR_DOMAIN before:2023-01-01`
- 캐시 확인(의도치 않게 남은 페이지):  
  - `cache:https://YOUR_DOMAIN/some/page`

> 다시 한 번 강조: **타 도메인**의 민감자료를 노리는 조합·자동화는 **불가**합니다.

---

### 맺음말
Google Dorking은 **공개 인덱스**를 더 잘 읽는 기술입니다. **방어자**에게 이것은 **섀도 자산을 발견**하고 **불필요한 노출을 줄이는** 강력한 도구입니다.  
위 가이드와 스크립트를 **본인 소유의 자산**에만 적용하고, 발견물을 **즉시 정리·비공개 전환**하세요. 보안은 **보이지 않게 하는 것**과 **노출되어도 안전하게 만드는 것**—두 축을 동시에 강화할 때 가장 탄탄해집니다.