---
layout: post
title: 웹해킹 - Host Header Injection
date: 2025-09-30 19:25:23 +0900
category: 웹해킹
---
# 🏷️ 3. Host Header Injection

## 0. 한눈에 보기 (Executive Summary)

- **문제**
  - 프록시/앱이 **`Host`** 또는 **`X-Forwarded-Host`(XFH)** 를 **신뢰**해 절대 URL, 리다이렉트, 이메일의 **비밀번호 재설정 링크** 등을 만들면,
    공격자가 `Host: evil.tld`(또는 `XFH: evil.tld`)로 요청하여 **공격자 도메인으로 서명된/신뢰감 있는 링크**를 만들게 함.
  - **HTTP/2**에서는 `Host` 대신 **`:authority`** (논리상 동일). 게이트웨이/프록시가 **잘못 매핑**해도 위험은 동일.

- **영향**
  - **피싱/계정 탈취**: 합법적인 이메일 템플릿에 **공격자 도메인**으로 된 재설정 링크 포함
  - **오픈 리디렉트/캐시 오염과 결합**: 악성 도메인으로 캐시 키 오염/리디렉트
  - **가짜 절대 URL**: 법적 고지/정책 링크도 공격자 사이트로

- **방어 핵심**
  1) **허용 호스트 목록(Allowlist)** 을 **앱 설정**에 고정.
  2) **프록시에서 Host 재작성**: 업스트림으로 **오직 정해진 Host** 만 전달, `XFH`/변칙 헤더 **제거**.
  3) 앱에서 **절대 URL은 `req.get('Host')` 로 만들지 말고**, **환경설정의 고정 ORIGIN**(예: `APP_ORIGIN=https://app.example.com`)으로 생성.
  4) 다(서브)도메인 지원이 필요한 **멀티테넌트**에서도 **명시적 매핑 테이블**로만 수용(패턴/와일드카드 남용 금지).

---

# 1. Host Header Injection이 발생하는 지점

- **절대 URL 생성**: 이메일·푸시·템플릿(`https://<host>/reset?token=...`)
- **리다이렉트**: 로그인 후 리디렉트·캔니컬·`Location` 헤더
- **외부 링크/메타 태그**: `<link rel="canonical">`, `<meta og:url>`, sitemap.xml
- **멀티테넌트**: `tenant.example.com` 같은 **서브도메인 → 테넌트 식별**
- **프록시/게이트웨이**: H2 `:authority` ↔ H1 `Host` 매핑, XFH 전파

> **규칙**: “**요청이 들고 온 Host**를 **절대 신뢰하지 말 것**”.
> **“앱이 아는 진짜 ORIGIN”**을 기준으로 링크를 만들어야 합니다.

---

# 2. 안전한 재현(테스트 전용) — **우리 앱이 Host를 신뢰하는지** 확인

> **반드시 사내 스테이징/개인 랩**에서만. 운영/외부 타 시스템 금지.

### 2.1 cURL: Host 변조 시도
```bash
# (staging.example.com 은 본인 스테이징)
curl -kis https://staging.example.com/reset/start \
  -H 'Host: evil.tld' \
  -H 'X-Forwarded-Host: evil.tld'
```
- **취약 신호**: 응답 본문/헤더(예: 이메일 미리보기, JSON)에 `https://evil.tld/...` 등장.

### 2.2 Node 스크립트: 이메일-링크 생성 API 점검
```js
// tools/host-check.js  (승인된 스테이징 전용)
import https from "node:https";

function request(headers={}) {
  return new Promise((resolve,reject)=>{
    const req = https.request({
      host: "staging.example.com",
      path: "/api/v1/reset/preview",
      method: "GET",
      headers: { "User-Agent": "host-check", ...headers },
      rejectUnauthorized: false
    }, res => {
      let body=""; res.on("data",d=>body+=d);
      res.on("end",()=>resolve({status:res.statusCode, body, headers:res.headers}));
    });
    req.on("error",reject).end();
  });
}

const run = async () => {
  const a = await request({ "Host": "evil.tld", "X-Forwarded-Host": "evil.tld" });
  console.log(a.status, a.headers["content-type"]);
  if (/evil\.tld/i.test(a.body)) {
    console.error("❌ Host/XFH가 링크 생성에 반영됨 — 취약 가능성");
    process.exit(1);
  } else {
    console.log("✅ 안전 — 고정 ORIGIN 사용으로 보임");
  }
};
run();
```

---

# 3. 프록시/게이트웨이 레벨의 **선제 방어**

> 목표: **엣지에서 Host를 “정답”으로 고정**하고, 앱에는 **오직 정해진 Host**만 도달하게 하기.

## 3.1 Nginx (Reverse Proxy)

```nginx
server {
  listen 443 ssl http2;
  server_name app.example.com www.example.com;

  # 1) 허용하지 않은 Host → 400 (또는 421 misdirected)
  if ($host !~* ^(app\.example\.com|www\.example\.com)$) { return 400; }

  location / {
    # 2) 업스트림으로 전달할 Host는 고정값
    proxy_set_header Host app.example.com;

    # 3) 전달용 헤더 제거/정규화
    proxy_set_header X-Forwarded-Host "";
    proxy_set_header X-Original-Host "";
    proxy_set_header X-Forwarded-Proto $scheme;

    # 4) (선택) 절대 URL을 강제로 www → apex 통일
    # return 301 https://app.example.com$request_uri;

    proxy_pass http://app_backend;
  }
}
```

**핵심**
- **`server_name` 제한** + **허용 외 Host 400**
- **업스트림으로 `Host` 고정**
- **XFH 제거**(앱이 실수로 참조하더라도 영향 없음)

## 3.2 HAProxy

```haproxy
frontend fe_https
  bind :443 ssl crt /etc/haproxy/cert.pem alpn h2,http/1.1
  # Host allowlist
  http-request deny unless { hdr(host) -i app.example.com } or { hdr(host) -i www.example.com }

  # 업스트림으로 Host 고정 전달
  http-request set-header Host app.example.com

  # XFH 제거
  http-request del-header X-Forwarded-Host
  default_backend be_app
```

## 3.3 Envoy (개념 예시)
- **VirtualHost.domains** 로 허용 호스트를 **정확 매칭**
- **H1 `Host` ↔ H2 `:authority`** 변환은 Envoy가 관리하지만, **도메인 미일치 시 421/404**
- **Header sanitizer** 필터로 `X-Forwarded-Host` 제거/무시

```yaml
route_config:
  virtual_hosts:
  - name: app
    domains: ["app.example.com","www.example.com"]  # 허용만
    routes:
      - match: { prefix: "/" }
        route: { cluster: app }
# (추가) header-to-add/delete 로 XFH 제거 가능
```

## 3.4 Apache httpd
```apache
<VirtualHost *:443>
  ServerName app.example.com
  ServerAlias www.example.com
  UseCanonicalName On

  # 허용 외 Host 거절
  <If "%{HTTP_HOST} != 'app.example.com' && %{HTTP_HOST} != 'www.example.com'">
    Require all denied
  </If>

  # XFH 제거
  RequestHeader unset X-Forwarded-Host early

  ProxyPass / http://app:8080/
  ProxyPassReverse / http://app:8080/
</VirtualHost>
```

> **요점**: **Edge에서 정리/차단 → 앱 단에서 실수하더라도 위험이 줄어듦**.

---

# 4. 애플리케이션 레벨 **정석 패턴**

> 원칙: “**앱이 아는 ORIGIN**”으로만 절대 URL/리디렉트를 만들고, **허용 호스트 검증**을 통과하지 못하면 **400/421**으로 거절.

## 4.1 Node/Express

### 4.1.1 고정 ORIGIN + 허용 호스트 검증
```js
// config/origin.js
import { domainToASCII } from "node:url";
const APP_ORIGIN = process.env.APP_ORIGIN || "https://app.example.com";
const ALLOWED_HOSTS = new Set(["app.example.com","www.example.com"].map(domainToASCII));
export { APP_ORIGIN, ALLOWED_HOSTS };
```

```js
// middleware/host-guard.js
import { ALLOWED_HOSTS } from "../config/origin.js";
export function hostGuard(req, res, next) {
  const rawHost = req.headers.host || "";
  const host = rawHost.toLowerCase().split(",")[0].trim(); // 다중 Host 방지
  if (!ALLOWED_HOSTS.has(host)) return res.status(400).send("Bad host");
  // 앱 전역에서 req.safeHost 로만 사용
  req.safeHost = host;
  next();
}
```

```js
// routes/reset.js
import { APP_ORIGIN } from "../config/origin.js";
app.post("/api/v1/reset/start", async (req, res) => {
  const token = await issueResetToken(req.body.email);
  const link = new URL("/reset/verify", APP_ORIGIN);
  link.searchParams.set("token", token);
  // 절대 URL을 **APP_ORIGIN** 으로만 생성
  await sendResetEmail(req.body.email, { link: link.href });
  res.json({ ok: true });
});
```

### 4.1.2 (선택) 다테넌트 — **명시적 매핑**만 허용
```js
// 예: tenant.example.com → tenantId
import { domainToASCII } from "node:url";
const TENANT_HOST_MAP = new Map([
  ["t1.example.com","tenant_1"],
  ["t2.example.com","tenant_2"]
].map(([h,id]) => [domainToASCII(h), id]));

app.use((req,res,next)=>{
  const h = (req.headers.host||"").toLowerCase();
  const tid = TENANT_HOST_MAP.get(h);
  if (!tid) return res.status(400).send("Unknown tenant host");
  req.tenantId = tid;
  next();
});
```
> **금지**: `*.example.com` 와일드카드, 부분문자열/정규식 **느슨한 허용**.
> **허용**: **정확 매핑**(DB/캐시로 관리), **IDNA/대소문자 정규화** 후 비교.

---

## 4.2 NestJS (유사 원칙)
```ts
// app.module.ts
const APP_ORIGIN = new URL(process.env.APP_ORIGIN ?? "https://app.example.com");
const ALLOWED = new Set(["app.example.com","www.example.com"]);

@Injectable()
export class HostGuard implements CanActivate {
  canActivate(ctx: ExecutionContext) {
    const req = ctx.switchToHttp().getRequest<Request>();
    const host = (req.headers['host']||"").toLowerCase();
    if (!ALLOWED.has(host)) throw new BadRequestException("Bad host");
    (req as any).safeHost = host;
    return true;
  }
}
```

---

## 4.3 Next.js
- **절대 URL 생성**은 환경변수 `NEXT_PUBLIC_APP_ORIGIN` (클라이언트 노출 시 위험) 대신 **서버 전용** `.env` 를 쓰고 서버 컴포넌트/API 라우트에서만 사용.
- 미들웨어에서 **Host allowlist** 검사 가능: `middleware.ts`

```ts
// middleware.ts
import { NextResponse } from "next/server";
const ALLOWED = new Set(["app.example.com","www.example.com"]);
export function middleware(req: Request) {
  const host = req.headers.get("host")?.toLowerCase() ?? "";
  if (!ALLOWED.has(host)) return new NextResponse("Bad host", { status: 400 });
  return NextResponse.next();
}
```

---

## 4.4 Django
```py
# settings.py
ALLOWED_HOSTS = ["app.example.com", "www.example.com"]
CSRF_TRUSTED_ORIGINS = ["https://app.example.com"]

# 절대 URL 생성은 sites framework 또는 환경설정 기반
APP_ORIGIN = "https://app.example.com"
```

```py
# views.py
from django.conf import settings
from urllib.parse import urlencode
def start_reset(request):
    token = issue_reset_token(request.POST["email"])
    link = f"{settings.APP_ORIGIN}/reset/verify?{urlencode({'token':token})}"
    send_mail("Reset", f"Click: {link}", ...)
    return JsonResponse({"ok":True})
```

---

## 4.5 Flask
```python
ALLOWED_HOSTS = {"app.example.com","www.example.com"}
APP_ORIGIN = "https://app.example.com"

@app.before_request
def host_guard():
    host = (request.host or "").lower()
    if host not in ALLOWED_HOSTS:
        abort(400)

@app.post("/reset/start")
def start_reset():
    token = issue_reset_token(request.json["email"])
    link = f"{APP_ORIGIN}/reset/verify?token={token}"
    send_email(..., link=link)
    return "", 204
```

---

## 4.6 Spring Boot
```java
// application.yml
app:
  origin: "https://app.example.com"
  allowed-hosts:
    - app.example.com
    - www.example.com
```

```java
@Component
public class HostFilter implements Filter {
  @Value("${app.allowed-hosts}") List<String> allowed;
  @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
      throws IOException, ServletException {
    HttpServletRequest r = (HttpServletRequest) req;
    String host = Optional.ofNullable(r.getHeader("Host")).orElse("").toLowerCase();
    if (!allowed.contains(host)) {
      ((HttpServletResponse) res).sendError(400, "Bad host");
      return;
    }
    chain.doFilter(req,res);
  }
}
```

```java
@Service
public class ResetService {
  @Value("${app.origin}") String origin;
  public String buildResetLink(String token) {
    return UriComponentsBuilder.fromHttpUrl(origin)
      .path("/reset/verify").queryParam("token", token).toUriString();
  }
}
```

---

## 4.7 Rails
```rb
# config/environments/production.rb
config.hosts << "app.example.com"
config.hosts << "www.example.com"

# 기본 URL
Rails.application.routes.default_url_options[:host] = "app.example.com"
Rails.application.routes.default_url_options[:protocol] = "https"
```

```rb
# mailer
def reset_email(user)
  @link = reset_verify_url(token: user.reset_token)   # 위 default_url_options 사용
  mail(to: user.email)
end
```

---

## 4.8 Go (net/http)
```go
var allowed = map[string]bool{"app.example.com":true, "www.example.com":true}
var appOrigin = "https://app.example.com"

func hostGuard(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    h := strings.ToLower(r.Host)
    if !allowed[h] { http.Error(w, "Bad host", http.StatusBadRequest); return }
    next.ServeHTTP(w,r)
  })
}

func resetStart(w http.ResponseWriter, r *http.Request) {
  token := issueResetToken(r.FormValue("email"))
  link := fmt.Sprintf("%s/reset/verify?token=%s", appOrigin, url.QueryEscape(token))
  sendMail(link)
  w.WriteHeader(204)
}
```

---

## 4.9 ASP.NET Core
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
var allowedHosts = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
  { "app.example.com","www.example.com" };
var appOrigin = builder.Configuration["App:Origin"] ?? "https://app.example.com";

app.Use((ctx,next) => {
  var host = ctx.Request.Host.Value?.ToLowerInvariant() ?? "";
  if (!allowedHosts.Contains(host)) {
    ctx.Response.StatusCode = 400; return ctx.Response.WriteAsync("Bad host");
  }
  return next();
});
```

```csharp
// ResetService.cs
public string BuildResetLink(string token, string origin)
  => $"{origin}/reset/verify?token={WebUtility.UrlEncode(token)}";
```

---

## 4.10 Laravel
```php
// app/Http/Middleware/HostGuard.php
public function handle($request, Closure $next) {
  $allowed = ['app.example.com','www.example.com'];
  if (!in_array(strtolower($request->getHost()), $allowed)) {
    abort(400, 'Bad host');
  }
  return $next($request);
}

// config/app.php
'app_origin' => env('APP_ORIGIN', 'https://app.example.com'),
```

```php
// Mail
$link = config('app.app_origin') . '/reset/verify?token=' . urlencode($token);
```

---

# 5. 이메일/리다이렉트 **안전 패턴**

- **이메일 템플릿**: `{{APP_ORIGIN}}/reset/verify?token=...` 만 사용. `req.host` 사용 금지.
- **리다이렉트**: `return redirect(APP_ORIGIN + safePath)` — **외부 도메인 금지**.
- **캔니컬/OG**: `<link rel="canonical" href="{{APP_ORIGIN}}{{path}}">`
- **다언어/브랜드 도메인**: `allowedHosts` 맵을 **정책 레이어**에서 관리, 운영툴로 승인 후 반영.

---

# 6. 로깅/모니터링/탐지 룰

- **이상치**
  - `Host` 가 서비스 도메인이 아닌 값으로 들어옴 (빈번/다양)
  - `X-Forwarded-Host` 가 비정상 빈도로 등장
  - **TLS SNI** vs `Host` **불일치**(가능하면 LB 로그에서 비교)
- **경보 예시 (Loki / Splunk)**

**Loki(LogQL)**
```logql
{service="edge", msg="http_access"} | json | host != "app.example.com" and host != "www.example.com"
| count_over_time(5m) > 10
```

**Splunk(SPL)**
```spl
index=edge msg=http_access NOT (host="app.example.com" OR host="www.example.com")
| stats count by host, src_ip, ua
```

---

# 7. 캐시/리버스 프록시와의 **상호작용 위험**(간략)

- Host 오염이 **Web Cache Poisoning**과 결합하면 “**오염된 절대 URL**이 캐시에 장시간 보존”될 수 있음.
- 방어는 다음과 동일: **캐시 키 명시화(Host/Path/Query)**, 전달용 헤더(XFH) 제거, **민감 `no-store`**.

---

# 8. 사고 대응(런북 요약)

1. **식별**: 재설정 링크/이메일·HTML 내 도메인이 서비스 도메인이 아닌 사례 포착.
2. **격리**: 프록시에서 즉시 **XFH 제거 + Host 고정** 배포, 비정상 Host 400/421.
3. **조사**: 로그에서 비정상 Host 빈도, 링크 생성 코드 경로, 멀티테넌트 매핑 검증.
4. **근절**: 앱 전역을 **APP_ORIGIN 기반**으로 전환, 허용 호스트 체크 미들웨어 추가.
5. **복구**: 잘못 발급된 링크 **무효화**(토큰 폐기), 필요 시 공지/지원.
6. **교훈화**: CI에 **Host 변조 스모크 테스트**(§2) 추가, 운영 룰북 업데이트.

---

# 9. 체크리스트 (현장용)

- [ ] 프록시에서 **Host 고정 전달**, **XFH 제거**
- [ ] 앱에서 **허용 호스트 검증**(IDNA/소문자 정규화 후 **정확 매칭**)
- [ ] 절대 URL/메일/리다이렉트는 **APP_ORIGIN**으로만 생성
- [ ] 멀티테넌트는 **정확 매핑 테이블**로만 식별(와일드카드 금지)
- [ ] 로그에 **비정상 Host/XFH** 탐지 규칙
- [ ] CI 스모크: `Host/XFH=evil.tld` 로 호출 시 **오염 링크 생성 금지**
- [ ] CDN/캐시와 결합: **키 명시화** + 민감 `no-store`

---

## 맺음말

**Host/X-Forwarded-Host**는 **신뢰할 수 없는 입력**입니다.
“**우리 서비스가 아는 ORIGIN**”을 기준으로 **항상** 절대 URL을 만들고, 엣지에서 Host를 **정답으로 고정**하세요.
위 템플릿을 작은 범위부터 적용하고, **로그 기반 탐지**와 **CI 스모크 테스트**로 재발 가능성을 구조적으로 줄이세요.
