---
layout: post
title: 웹해킹 - Host Header Injection
date: 2025-09-30 19:25:23 +0900
category: 웹해킹
---
# Host Header Injection — ORIGIN 탈취, 비밀번호 재설정 링크 오염, 프록시·CDN·멀티테넌트

## 1. 한눈에 보기 (Executive Summary)

### 1.1 문제 요약

- HTTP/1.1에서 **`Host` 헤더**, HTTP/2에서 **`:authority`** 는  
  “**이 요청이 어느 가상 호스트/도메인을 대상으로 하는가**”를 알려 주기 위해 **클라이언트가 제공하는 값**이다.
- 많은 웹 애플리케이션이 다음과 같은 이유로 이 값을 **그대로 신뢰**한다:
  - 비밀번호 재설정·회원가입 확인 등 **이메일 링크** 생성
  - 로그인 후 **리다이렉트 URL** 생성
  - `<link rel="canonical">`, `<meta property="og:url">` 등 **절대 URL** 생성
  - 멀티테넌트에서 `tenant.example.com` → **테넌트 식별**
- 공격자는 이 점을 악용해서:

  ```http
  GET /reset/start HTTP/1.1
  Host: evil.tld
  X-Forwarded-Host: evil.tld
  ...
  ```

  처럼 **임의의 Host/XFH 값**을 넣어 보낸 뒤,  
  애플리케이션이 이 값을 신뢰한다면:

  - **비밀번호 재설정 이메일**에 `https://evil.tld/reset/verify?token=...` 같은 **공격자 도메인** 링크를 집어넣게 만들고,
  - 수신자가 “공식 서비스 메일”이라고 믿고 링크를 클릭하도록 유도해 **토큰 탈취·계정 탈취**로 이어지게 만든다.:contentReference[oaicite:0]{index=0}  

- HTTP/2 환경에서는 `Host` 대신 **`:authority` 헤더**가 같은 역할을 한다. 프록시/게이트웨이가 이를 **잘못 매핑**하거나, `X-Forwarded-Host` 를 그대로 뒤로 넘기면 위험은 그대로 유지된다.:contentReference[oaicite:1]{index=1}  

### 1.2 대표 영향

- **피싱·계정 탈취**
  - 비밀번호 재설정 / 이메일 인증 / 초대 링크 등이 **공격자 도메인**으로 생성.
  - 사용자는 메일 콘텐츠·브랜딩을 보고 **합법 서비스**라고 믿고 토큰을 넘김.
- **오픈 리다이렉트와 결합**
  - `Location: https://evil.tld/...` 를 **정상 서버**가 직접 보내므로,  
    많은 브라우저·보안 솔루션이 이를 “신뢰 가능한 리다이렉트”로 인식.
- **캐시 오염 / Web Cache Poisoning**
  - 프론트 프록시·CDN이 **Host를 캐시 키에 포함**할 때, Host 오염이 **캐시 키**와 **콘텐츠**를 함께 오염시킬 수 있다.  
    Host Header 기반 Web Cache Poisoning은 실제 상용 서비스에서도 여러 번 보고되었다.:contentReference[oaicite:2]{index=2}  
- **멀티테넌트 교란**
  - `tenant.example.com` → `tenant_id` 매핑을  
    `Host` 문자열에 의존할 경우,  
    승인받지 않은 도메인으로 **다른 테넌트 데이터에 접근**하는 경로가 생길 수 있다.

### 1.3 방어 핵심 정리

1. **허용 호스트 목록(Allowlist)** 을 **정책/환경설정**으로 고정하고,  
   이 목록에 없는 Host/XFH는 **모두 거절(400/421)** 한다.
2. **프록시·게이트웨이(Edge)** 에서:
   - 업스트림으로 전달할 **`Host` 값을 고정**하고,
   - `X-Forwarded-Host`, `X-Original-Host` 등 **변칙 헤더를 제거/무시**한다.
3. **애플리케이션 코드**에서:
   - 절대 URL, 이메일 링크, 리다이렉트, canonical URL 등은 **항상 환경설정의 고정 ORIGIN**(예: `APP_ORIGIN=https://app.example.com`)으로 생성한다.
   - `req.get("Host")`, `request.host`, `HttpServletRequest.getHeader("Host")` 등 **요청 기반 Host**로 링크를 만들지 않는다.
4. **멀티테넌트**가 필요한 경우:
   - `tenant.example.com` → `tenant_id` 는 **정확한 매핑 테이블**(DB/캐시)로만 처리한다.
   - `*.example.com` 와일드카드, 정규식 기반 느슨한 호스트 수용 등은 최대한 피한다.

---

## 2. Host Header Injection이 발생하는 지점

### 2.1 Host / X-Forwarded-Host / :authority

- HTTP/1.1:
  - `Host` 헤더는 **필수**이며, 가상 호스팅 환경에서 “어느 사이트인가?”를 결정한다.
- HTTP/2:
  - `Host` 대신 **`:authority` pseudo-header**가 같은 역할을 한다.
  - 많은 프록시/게이트웨이가 H2 → H1로 변환하면서 `:authority` 를 `Host` 로 매핑한다.
- 프록시 환경:
  - L7 프록시/로드밸런서/CDN은 클라이언트와 서버 사이에서  
    `Host`, `X-Forwarded-Host`, `X-Forwarded-Proto`, `X-Forwarded-Port` 등을 주고받는다.

### 2.2 애플리케이션에서 Host를 사용하는 패턴

1. **비밀번호 재설정 / 이메일 확인 / 초대 링크**

   ```js
   // 취약한 패턴 (개념)
   app.post("/reset/start", (req, res) => {
     const token = issueToken(req.body.email);
     const base  = `${req.protocol}://${req.get("host")}`;
     const link  = `${base}/reset/verify?token=${token}`;
     sendResetMail(req.body.email, link);
     res.sendStatus(204);
   });
   ```

   - 공격자가 `Host: evil.tld` 로 요청하면,  
     `link`가 `https://evil.tld/reset/verify?token=...` 로 만들어진다.
   - 사용자는 메일 발신자·템플릿을 보고 **합법 서비스**라고 인식한다.:contentReference[oaicite:3]{index=3}  

2. **리다이렉트 / Location 헤더**

   ```js
   // 로그인 후 같은 도메인으로 리다이렉트
   app.post("/login", (req, res) => {
     // ... 인증 로직 ...
     const base = `${req.protocol}://${req.get("host")}`;
     res.redirect(base + "/app/dashboard");
   });
   ```

   - 공격자는 `Host: evil.tld` 로 요청 →  
     서버가 직접 `Location: https://evil.tld/app/dashboard` 를 보냄.
   - 이 리다이렉트는 **정상 서비스가 보내는 3xx 응답**이므로,  
     여러 보안 장비/필터가 이를 “정상적인 리다이렉트”로 보게 된다.

3. **Canonical / OG / Sitemap / RSS**

   - `<link rel="canonical">`, `<meta property="og:url">`, `sitemap.xml`, RSS/Atom feed 내의 `<link>` 태그 등을 만들 때도  
     흔히 `Host`를 사용한다.

   ```js
   app.get("/post/:slug", (req, res) => {
     const base = `${req.protocol}://${req.get("host")}`;
     const url  = `${base}${req.originalUrl}`;
     res.render("post", { canonical: url, ogUrl: url });
   });
   ```

   - 검색엔진이나 소셜 미리보기에서 **악성 도메인**이 canonical로 인덱싱될 수 있다.

4. **멀티테넌트**

   - `tenant.example.com` 에서 `tenant_id`를 파생하는 패턴:

     ```js
     app.use((req, res, next) => {
       const host = (req.headers.host || "").toLowerCase();
       const [sub] = host.split(".");  // sub = tenant
       req.tenantId = sub;
       next();
     });
     ```

   - 이때 **허용하지 않은 Host**(예: `evil.example.com`, `attacker.com`)가 들어와도  
     그대로 테넌트로 인식하면, 인증·권한 로직이 꼬이고  
     다른 테넌트 데이터에 접근하는 통로가 생길 수 있다.

---

## 3. Host Injection이 실제로 어떻게 보이는지 (안전한 스테이징 테스트)

> 아래는 **자기 조직의 스테이징/랩 환경**에서 **“차단이 잘 되는지”** 확인하기 위한 테스트 예제이다.  
> **운영 서비스나 타인의 시스템에는 절대 사용하면 안 된다.**

### 3.1 cURL로 빠른 점검

```bash
# 1) Host/X-Forwarded-Host를 변조해서 비밀번호 재설정 시작 API 호출

curl -kis https://staging.example.com/reset/start \
  -H 'Host: evil.tld' \
  -H 'X-Forwarded-Host: evil.tld'

# 응답이 HTML 이거나 "미리보기" API라면, evil.tld가 포함되어 있는지 grep
curl -ks https://staging.example.com/reset/preview \
  -H 'Host: evil.tld' \
  -H 'X-Forwarded-Host: evil.tld' | grep -i 'evil.tld'
```

- **취약 신호**:
  - 응답 HTML/JSON 내에 `https://evil.tld/` 링크가 들어 있다.
  - 메일 미리보기 API가 있다면 그 안에 `evil.tld` 가 절대 URL로 등장한다.

### 3.2 Node.js로 자동 스모크 테스트

```js
// tools/host-check.js  (사내 스테이징 전용)
// "Host/XFH를 바꿔도 링크 생성에 반영되지 않는다"는 것을 CI에서 검증
import https from "node:https";

function request(headers = {}) {
  return new Promise((resolve, reject) => {
    const req = https.request({
      host: "staging.example.com",
      path: "/api/v1/reset/preview",
      method: "GET",
      headers: { "User-Agent": "host-check", ...headers },
      rejectUnauthorized: false
    }, res => {
      let body = "";
      res.on("data", d => body += d);
      res.on("end", () => resolve({ status: res.statusCode, body, headers: res.headers }));
    });
    req.on("error", reject).end();
  });
}

const run = async () => {
  const a = await request({
    "Host": "evil.tld",
    "X-Forwarded-Host": "evil.tld"
  });

  console.log(a.status, a.headers["content-type"]);

  if (/evil\.tld/i.test(a.body)) {
    console.error("취약 가능성: Host/XFH가 링크 생성에 반영됨");
    process.exit(1);
  } else {
    console.log("안전해 보임: 고정 APP_ORIGIN으로 링크를 생성하는 것으로 보임");
  }
};

run();
```

- 이 스크립트를 **CI 파이프라인**에서 스테이징에 대해 주기적으로 실행하면,  
  “어느 PR 이후에 Host/XFH가 다시 코드에 스며들었는지”를 빠르게 잡아낼 수 있다.

---

## 4. Edge / 프록시 / 게이트웨이 레벨 선제 방어

> 원칙: **엣지에서 Host를 “정답”으로 고정**하고,  
> 애플리케이션에는 **오직 정해진 Host만 도달**하게 한다.

### 4.1 Nginx Reverse Proxy

```nginx
server {
  listen 443 ssl http2;
  server_name app.example.com www.example.com;

  # 1) 허용하지 않은 Host → 400 (또는 421)
  if ($host !~* ^(app\.example\.com|www\.example\.com)$) {
    return 400;
  }

  location / {
    # 2) 업스트림으로 전달할 Host는 고정값
    proxy_set_header Host app.example.com;

    # 3) 전달용 Host 헤더 제거/정규화
    proxy_set_header X-Forwarded-Host "";
    proxy_set_header X-Original-Host "";
    proxy_set_header X-Forwarded-Proto $scheme;

    # 4) (선택) 항상 app.example.com 으로 정규화하는 301 리다이렉트
    # if ($host = "www.example.com") {
    #   return 301 https://app.example.com$request_uri;
    # }

    proxy_pass http://app_backend;
  }
}
```

**질문**: 이미 `server_name`으로 호스트 가상호스팅을 하니 안전한 것 아닌가?  

- `server_name`은 **TLS SNI + Host 조합으로 어떤 서버 블록에 라우팅할지**를 결정하는 용도이고,
- 위와 같이 **허용하지 않은 Host를 명시적으로 400으로 거절**하고,
- 그 뒤 프록시가 업스트림으로 전달할 `Host` 값을 **확실하게 하나로 고정**해 줘야  
  애플리케이션이 Host를 잘못 사용하더라도 **실제 도달하는 값**이 안전하게 제한된다.:contentReference[oaicite:4]{index=4}  

### 4.2 HAProxy

```haproxy
frontend fe_https
  bind :443 ssl crt /etc/haproxy/cert.pem alpn h2,http/1.1

  # 1) Host allowlist
  http-request deny unless { hdr(host) -i app.example.com } or { hdr(host) -i www.example.com }

  # 2) 업스트림으로 Host 고정
  http-request set-header Host app.example.com

  # 3) XFH 제거
  http-request del-header X-Forwarded-Host

  default_backend be_app

backend be_app
  server app1 app:8080 check
```

### 4.3 Envoy (개념 YAML)

```yaml
static_resources:
  listeners:
  - name: https
    address:
      socket_address: { address: 0.0.0.0, port_value: 443 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: app
              domains: ["app.example.com","www.example.com"]  # 허용 호스트만
              routes:
              - match: { prefix: "/" }
                route: { cluster: app }
          # header_sanitizer 필터 등을 사용해 XFH 제거 가능
  clusters:
  - name: app
    connect_timeout: 2s
    type: logical_dns
    lb_policy: round_robin
    load_assignment:
      cluster_name: app
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: app, port_value: 8080 }
```

- Envoy는 H2/H1을 내부적으로 관리하므로, **VirtualHost.domains** 로 **정확한 도메인 집합**만 열어 두고  
  나머지를 404/421로 처리하면 된다.

### 4.4 Apache httpd

```apache
<VirtualHost *:443>
  ServerName app.example.com
  ServerAlias www.example.com

  UseCanonicalName On  # 리다이렉트/URL 생성 시 ServerName 기준

  # 허용 외 Host 거절
  <If "%{HTTP_HOST} != 'app.example.com' && %{HTTP_HOST} != 'www.example.com'">
    Require all denied
  </If>

  # XFH 제거
  RequestHeader unset X-Forwarded-Host early

  ProxyPass        / http://app:8080/
  ProxyPassReverse / http://app:8080/
</VirtualHost>
```

---

## 5. 애플리케이션 레벨 정석 패턴

> 프록시에서 Host를 고정해도,  
> “절대 URL을 **환경설정의 ORIGIN**으로만 생성한다”는 원칙을 **애플리케이션 코드**에서 반드시 지켜야 한다.

### 5.1 Node.js / Express

#### 5.1.1 고정 ORIGIN + 허용 호스트 검증

```js
// config/origin.js
import { domainToASCII } from "node:url";

const APP_ORIGIN = process.env.APP_ORIGIN || "https://app.example.com";
const ALLOWED_HOSTS = new Set(
  ["app.example.com", "www.example.com"].map(domainToASCII)
);

export { APP_ORIGIN, ALLOWED_HOSTS };
```

```js
// middleware/host-guard.js
import { ALLOWED_HOSTS } from "../config/origin.js";

export function hostGuard(req, res, next) {
  const rawHost = req.headers.host || "";
  const host = rawHost.toLowerCase().split(",")[0].trim(); // 다중 Host 방지
  if (!ALLOWED_HOSTS.has(host)) {
    return res.status(400).send("Bad host");
  }
  req.safeHost = host; // 필요하다면 이후 로직에서 사용
  next();
}
```

- 이 미들웨어를 **전역으로 적용**해 두면,  
  이후 코드에서는 `req.safeHost` 만 신뢰하면 된다.

#### 5.1.2 비밀번호 재설정 — APP_ORIGIN만 사용

```js
// routes/reset.js
import express from "express";
import { APP_ORIGIN } from "../config/origin.js";
const router = express.Router();

async function issueResetToken(email) {
  // 토큰 발급 로직(예: DB에 저장)
  return "random-token";
}

async function sendResetEmail(email, { link }) {
  // 실제 메일 발송 로직
  console.log("send mail", email, link);
}

router.post("/api/v1/reset/start", async (req, res) => {
  const { email } = req.body ?? {};
  if (typeof email !== "string") return res.status(400).json({ error: "invalid" });

  const token = await issueResetToken(email);
  const link = new URL("/reset/verify", APP_ORIGIN);
  link.searchParams.set("token", token);

  await sendResetEmail(email, { link: link.href });
  res.json({ ok: true });
});

export default router;
```

- 어디에도 `req.get("host")` 가 등장하지 않는다.
- `APP_ORIGIN` 은 **배포 환경별로만** 바꾸면 되고,  
  애플리케이션 로직은 그대로 재사용 가능하다.

#### 5.1.3 멀티테넌트 — 명시적 매핑

```js
// tenant-map.js
import { domainToASCII } from "node:url";

const TENANT_HOST_MAP = new Map(
  [
    ["t1.example.com", "tenant_1"],
    ["t2.example.com", "tenant_2"]
  ].map(([h, id]) => [domainToASCII(h), id])
);

export function resolveTenantFromHost(hostHeader) {
  const host = (hostHeader || "").toLowerCase().split(",")[0].trim();
  const id = TENANT_HOST_MAP.get(host);
  if (!id) throw new Error("Unknown tenant host");
  return id;
}
```

```js
// app.js
import { resolveTenantFromHost } from "./tenant-map.js";

app.use((req, res, next) => {
  try {
    req.tenantId = resolveTenantFromHost(req.headers.host);
    next();
  } catch {
    res.status(400).send("Unknown tenant");
  }
});
```

- `*.example.com` 패턴 매칭 대신, **정확한 호스트 이름만** 허용한다.
- 신규 테넌트를 온보딩할 때는 운영 툴을 통해  
  **검증된 도메인만 매핑 테이블에 추가**한다.

---

### 5.2 Python / Django

```python
# settings.py

ALLOWED_HOSTS = ["app.example.com", "www.example.com"]
CSRF_TRUSTED_ORIGINS = ["https://app.example.com"]

APP_ORIGIN = "https://app.example.com"  # 절대 URL 생성에 사용
```

```python
# views.py

from django.conf import settings
from django.http import JsonResponse
from urllib.parse import urlencode

def issue_reset_token(email: str) -> str:
  # 토큰 발급 로직
  return "random-token"

def start_reset(request):
    if request.method != "POST":
        return JsonResponse({"error":"method"}, status=405)

    email = request.POST.get("email")
    if not isinstance(email, str):
        return JsonResponse({"error":"invalid"}, status=400)

    token = issue_reset_token(email)
    link = f"{settings.APP_ORIGIN}/reset/verify?{urlencode({'token':token})}"

    # send_mail(..., link 포함)
    return JsonResponse({"ok": True})
```

- Django는 `ALLOWED_HOSTS` 기반의 Host 검증을 제공하므로,  
  Apache/Nginx와 함께 사용하면 **2중 검증**이 가능하다.

---

### 5.3 Python / Flask

```python
from flask import Flask, request, abort
from urllib.parse import urlencode

app = Flask(__name__)

ALLOWED_HOSTS = {"app.example.com", "www.example.com"}
APP_ORIGIN = "https://app.example.com"

@app.before_request
def host_guard():
    host = (request.host or "").lower()
    if host not in ALLOWED_HOSTS:
        abort(400)

def issue_reset_token(email: str) -> str:
    return "random-token"

@app.post("/reset/start")
def start_reset():
    data = request.get_json(force=True, silent=True) or {}
    email = data.get("email")
    if not isinstance(email, str):
        abort(400)
    token = issue_reset_token(email)
    link = f"{APP_ORIGIN}/reset/verify?{urlencode({'token': token})}"
    # send_email(email, link)
    return ("", 204)
```

---

### 5.4 Java / Spring Boot

```yaml
# application.yml
app:
  origin: "https://app.example.com"
  allowed-hosts:
    - app.example.com
    - www.example.com
```

```java
// HostFilter.java
@Component
public class HostFilter implements Filter {

  @Value("${app.allowed-hosts}")
  private List<String> allowed;

  @Override
  public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
      throws IOException, ServletException {
    HttpServletRequest r = (HttpServletRequest) req;
    String host = Optional.ofNullable(r.getHeader("Host")).orElse("").toLowerCase(Locale.ROOT);
    if (!allowed.contains(host)) {
      ((HttpServletResponse) res).sendError(400, "Bad host");
      return;
    }
    chain.doFilter(req, res);
  }
}
```

```java
// ResetService.java
@Service
public class ResetService {

  @Value("${app.origin}")
  private String origin;

  public String buildResetLink(String token) {
    return UriComponentsBuilder.fromHttpUrl(origin)
      .path("/reset/verify")
      .queryParam("token", token)
      .toUriString();
  }
}
```

```java
// ResetController.java
@RestController
@RequestMapping("/api/v1/reset")
public class ResetController {

  private final ResetService service;

  public ResetController(ResetService service) {
    this.service = service;
  }

  @PostMapping("/start")
  public ResponseEntity<?> start(@RequestBody Map<String,String> body) {
    String email = body.get("email");
    if (email == null) return ResponseEntity.badRequest().build();
    String token = "random-token";
    String link = service.buildResetLink(token);
    // 이메일 발송...
    return ResponseEntity.ok(Map.of("ok", true));
  }
}
```

---

### 5.5 Go (net/http)

```go
var allowedHosts = map[string]bool{
  "app.example.com":  true,
  "www.example.com":  true,
}

var appOrigin = "https://app.example.com"

func hostGuard(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    h := strings.ToLower(r.Host)
    if !allowedHosts[h] {
      http.Error(w, "Bad host", http.StatusBadRequest)
      return
    }
    next.ServeHTTP(w, r)
  })
}

func resetStart(w http.ResponseWriter, r *http.Request) {
  if r.Method != http.MethodPost {
    http.Error(w, "method", http.StatusMethodNotAllowed)
    return
  }
  email := r.FormValue("email")
  if email == "" {
    http.Error(w, "invalid", http.StatusBadRequest)
    return
  }
  token := "random-token"
  link := fmt.Sprintf("%s/reset/verify?token=%s", appOrigin, url.QueryEscape(token))
  // sendMail(email, link)
  w.WriteHeader(http.StatusNoContent)
}
```

---

### 5.6 ASP.NET Core

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

var allowedHosts = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
{
    "app.example.com",
    "www.example.com"
};

var appOrigin = builder.Configuration["App:Origin"] ?? "https://app.example.com";

var app = builder.Build();

app.Use(async (ctx, next) =>
{
    var host = ctx.Request.Host.Value?.ToLowerInvariant() ?? "";
    if (!allowedHosts.Contains(host))
    {
        ctx.Response.StatusCode = StatusCodes.Status400BadRequest;
        await ctx.Response.WriteAsync("Bad host");
        return;
    }
    await next();
});

app.MapPost("/reset/start", async (HttpContext ctx) =>
{
    var form = await ctx.Request.ReadFormAsync();
    var email = form["email"].ToString();
    if (string.IsNullOrWhiteSpace(email))
    {
        ctx.Response.StatusCode = StatusCodes.Status400BadRequest;
        return;
    }
    var token = "random-token";
    var link = $"{appOrigin}/reset/verify?token={WebUtility.UrlEncode(token)}";
    // send mail...
    ctx.Response.StatusCode = StatusCodes.Status204NoContent;
});

app.Run();
```

---

### 5.7 Ruby on Rails

```rb
# config/environments/production.rb

config.hosts << "app.example.com"
config.hosts << "www.example.com"

Rails.application.routes.default_url_options[:host]     = "app.example.com"
Rails.application.routes.default_url_options[:protocol] = "https"
```

```rb
# app/mailers/user_mailer.rb

class UserMailer < ApplicationMailer
  def reset_email(user)
    @user = user
    @link = reset_verify_url(token: user.reset_token)  # default_url_options 사용
    mail(to: @user.email, subject: "Reset your password")
  end
end
```

---

### 5.8 Laravel (PHP)

```php
// app/Http/Middleware/HostGuard.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class HostGuard
{
    public function handle(Request $request, Closure $next)
    {
        $allowed = ['app.example.com','www.example.com'];
        if (!in_array(strtolower($request->getHost()), $allowed, true)) {
            abort(400, 'Bad host');
        }
        return $next($request);
    }
}
```

```php
// config/app.php
return [
    // ...
    'app_origin' => env('APP_ORIGIN', 'https://app.example.com'),
];
```

```php
// app/Mail/ResetMail.php
public function build()
{
    $origin = config('app.app_origin');
    $link = $origin.'/reset/verify?token='.urlencode($this->token);
    return $this->view('emails.reset')->with(['link' => $link]);
}
```

---

## 6. 이메일 / 리다이렉트 / Canonical 안전 패턴

### 6.1 이메일 템플릿

- 템플릿 언어에서는 항상 `{{APP_ORIGIN}}` 를 사용하도록 규칙화한다.

```html
<p>비밀번호를 재설정하려면 아래 링크를 클릭하세요.</p>
<p><a href="{{APP_ORIGIN}}/reset/verify?token={{token}}">비밀번호 재설정</a></p>
```

- 여기서 `APP_ORIGIN` 은 애플리케이션 레이어에서 **환경설정**으로 주입한다.

### 6.2 리다이렉트

- 인증/로그인 후 리다이렉트는 **내부 경로**만 허용하고,  
  외부 도메인으로의 리다이렉트는 **명시적으로 금지**해야 한다.

```js
// Node 예시
const SAFE_PATHS = new Set(["/app/dashboard","/app/home"]);

app.post("/login", (req, res) => {
  // 인증 성공 후
  const next = typeof req.body.next === "string" ? req.body.next : "/app/dashboard";
  const safe = SAFE_PATHS.has(next) ? next : "/app/dashboard";
  res.redirect(safe); // host 없이 path만 사용
});
```

- 절대 URL로 리다이렉트를 보내야 하는 경우에도:

```js
const link = new URL("/app/dashboard", APP_ORIGIN);
res.redirect(link.href);
```

처럼 **APP_ORIGIN** 기반으로만 생성한다.

### 6.3 Canonical / OG / Sitemap

```html
<link rel="canonical" href="{{APP_ORIGIN}}{{request_path}}">
<meta property="og:url" content="{{APP_ORIGIN}}{{request_path}}">
```

- 검색엔진과 소셜 플랫폼에서 **항상 서비스 도메인**으로 인덱싱/미리보기되도록 보장한다.

---

## 7. 로그·모니터링·탐지

> Host 헤더는 공격자가 마음대로 조작할 수 있으므로,  
> “**서비스 도메인이 아닌 Host/XFH** 요청”을 **항상 모니터링**하는 것이 좋다.:contentReference[oaicite:5]{index=5}  

### 7.1 로그 필드

- 공통 필드 예:
  - `ts` (timestamp)
  - `host` (요청 Host)
  - `authority` (HTTP/2의 :authority, 가능하면 별도)
  - `sni` (TLS SNI, LB/프록시 레벨에서)
  - `xfh` (X-Forwarded-Host)
  - `src_ip`, `user_agent`, `path`, `status`

### 7.2 탐지 룰 예시

#### Loki(LogQL)

```logql
{service="edge", msg="http_access"} | json
| host != "app.example.com" and host != "www.example.com"
| count_over_time(5m) > 10
```

- 정해진 서비스 도메인이 아닌 Host가  
  5분 동안 10번 이상 등장하면 경보.

#### Splunk(SPL)

```spl
index=edge msg=http_access NOT (host="app.example.com" OR host="www.example.com")
| stats count by host, src_ip, user_agent
| where count > 5
```

- 특정 `src_ip` 또는 `user_agent` 가 **다양한 Host를 시도**하면  
  Host Header fuzzing/HV 스캐닝으로 의심할 수 있다.

### 7.3 SNI vs Host 불일치

- L7 LB/프록시에서 TLS SNI와 HTTP Host를 함께 로깅하면:
  - `SNI=app.example.com`, `Host=evil.tld` 같은 **불일치 패턴**을 탐지할 수 있다.
- 많은 대형 클라우드·APM·WAF 솔루션이 이런 **헤더/필드 불일치 탐지 규칙**을 내장하고 있다.:contentReference[oaicite:6]{index=6}  

---

## 8. 캐시·웹 캐시 포이저닝과의 상호 작용

> Host Header Injection은 **Web Cache Poisoning**과 결합했을 때  
> 파급력이 훨씬 커진다.  

- CDN/캐시는 일반적으로 **캐시 키**로  
  `Scheme + Host + Path (+ Query)` 조합을 사용한다.
- 공격자가 Host를 오염시켜 캐시 키를 바꾸거나,  
  **Host에 따라 다른 응답을 캐시에 주입**할 수 있다.:contentReference[oaicite:7]{index=7}  

### 8.1 위험 패턴

- 애플리케이션이 `Host` 에 따라 **HTML 내 절대 URL**을 바꾸고,
- CDN 캐시가 Host를 키에 포함할 경우:

  1. 공격자: `Host: evil.tld` 로 요청 → HTML 안에 `https://evil.tld/...` 포함된 응답을 캐시에 저장.
  2. 동일한 Host로 오는 후속 요청은 **모두 오염된 응답**을 받는다.
  3. 만약 **역방향 프록시 구성 오류**로 `Host` 를 잘못 재작성하면,  
     동일 캐시 엔트리를 정상 사용자에게도 제공하는 상황이 생길 수 있다.

### 8.2 방어 포인트 (캐시 관점)

- 캐시 키를 **명시적으로** 구성: `scheme + host + path + normalized query`.
- Host Header Injection 방어(allowlist, 고정 전달)를 먼저 구현해  
  캐시 앞단으로 **비정상 Host가 들어오지 못하게** 한다.
- 캐시 정책에서:
  - `Host` 나 `X-Forwarded-Host` 를 기반으로 TTL을 다르게 두는 휴리스틱은 피한다.
  - 민감 경로( `/reset`, `/account`, `/login` 등)는  
    항상 `Cache-Control: no-store` 로 처리한다.

---

## 9. 사고 대응 런북 (요약)

Host Header Injection이 의심되는 상황에서  
“현장에서 실제로 무엇을 해야 하는가?”를 정리한다.

1. **식별**
   - 비밀번호 재설정·초대 링크·이메일 등의 링크에  
     서비스 도메인이 아닌 URL이 포함된 사례 확인.
   - HTML/로그에서 `Host: evil.tld` 류 패턴과 함께  
     비정상적인 링크 생성 흔적이 있는지 분석.

2. **격리**
   - 프록시/게이트웨이에서:
     - `X-Forwarded-Host` 전면 제거 또는 무시.
     - 업스트림으로 전달하는 `Host` 값을 **고정 값**으로 설정.
     - 허용하지 않은 Host는 **바로 400/421**.
   - 캐시/CDN이 있다면:
     - 문제가 된 경로/Host 조합에 대한 **캐시 무효화(PURGE)**.
     - 민감 경로에 대한 캐시 **일시 BYPASS**.

3. **조사**
   - 얼마나 많은 링크/토큰이 잘못 발급되었는지,
   - 어떤 Host 값이 사용되었는지,
   - 멀티테넌트·SSO/OIDC 콜백·리다이렉트·canonical/OG 등  
     어떤 코드 경로가 Host를 사용했는지 파악한다.

4. **근절**
   - **APP_ORIGIN 기반 절대 URL 생성**으로 코드 전면 교체.
   - 프록시/애플리케이션 전 계층에 **Host allowlist / XFH 제거** 반영.
   - 멀티테넌트라면 **정확 매핑 테이블**을 도입하고, 기존 패턴 제거.

5. **복구**
   - 잘못 발급된 토큰/링크는 **폐기**하고, 필요한 경우 사용자 공지·지원.
   - 로그/모니터링 대시보드를 보강해  
     Host/XFH 이상치 패턴을 상시 감시하게 한다.

6. **교훈화**
   - CI 파이프라인에 **Host 변조 스모크 테스트**(§3.2) 추가.
   - 보안 코드 리뷰 체크리스트에  
     “절대 URL/리다이렉트/메일 링크에서 요청 Host 사용 금지” 항목 추가.
   - OWASP Top 10, 주요 벤더 가이드(IBM, Acunetix 등)에서  
     Host Header 관련 권고사항을 정기적으로 검토해 반영한다.:contentReference[oaicite:8]{index=8}  

---

## 10. 운영 체크리스트 (현장용)

- [ ] 프록시/게이트웨이에서 **Host allowlist** 적용 (`server_name`, `domains`, ACL 등)
- [ ] 업스트림으로 전달하는 `Host` 값을 **고정 ORIGIN**으로 설정
- [ ] `X-Forwarded-Host`, `X-Original-Host` 등은 기본적으로 **제거/무시**
- [ ] 애플리케이션에서 **허용 호스트 검증 미들웨어** 적용
- [ ] 비밀번호 재설정·초대·이메일 링크는 **APP_ORIGIN 기반 절대 URL**로만 생성
- [ ] 리다이렉트는 **내부 path 기반** 또는 APP_ORIGIN 기반으로만 구성
- [ ] 멀티테넌트는 **정확 매핑 테이블**로만 처리(와일드카드/정규식 금지)
- [ ] canonical / OG / sitemap / RSS 링크도 모두 APP_ORIGIN 기준
- [ ] 로그에 Host/XFH/SNI를 기록하고, **비정상 Host 빈도**에 대한 경보 설정
- [ ] CI에 **Host/XFH 변조 테스트** 추가, 실패 시 빌드 차단

---

## 11. 맺음말

Host Header Injection은 한 줄로 요약하면:

> “**신뢰할 수 없는 Host/X-Forwarded-Host 값으로  
>  절대 URL·링크·리다이렉트를 만들었을 때 발생하는 공격**”

이다.

이 취약점 자체는 오래전부터 알려져 있었지만,  
오늘날에도 여전히 **비밀번호 재설정 링크·멀티테넌트·CDN·캐시** 등  
복잡한 아키텍처와 맞물리면서 **큰 사고**로 이어지고 있다.:contentReference[oaicite:9]{index=9}  

이 글에서 정리한 것처럼:

- **엣지에서 Host를 고정**하고,
- 애플리케이션에서는 **APP_ORIGIN이라는 하나의 진실한 ORIGIN**을 기준으로 링크를 만들며,
- 멀티테넌트는 **정확한 매핑 테이블**로만 관리하고,
- CI/로그/모니터링으로 **Host 변조**를 상시 감시하면,

Host Header Injection과 그 파생 공격(피싱, 계정 탈취, 캐시 오염)을  
**구조적으로 제거**할 수 있다.

실제 시스템에 바로 가져다 쓸 수 있는  
Nginx/HAProxy/Envoy/Apache 설정과  
Express/Django/Flask/Spring/Rails/ASP.NET/Laravel/Go 예제를  
각자의 코드베이스에 맞게 조금씩 변형해 적용해 보자.  
작은 규칙 몇 개만 지켜도, **전체 ORIGIN 신뢰 모델**이 훨씬 단단해지는 것을  
운영 과정에서 체감할 것이다.
