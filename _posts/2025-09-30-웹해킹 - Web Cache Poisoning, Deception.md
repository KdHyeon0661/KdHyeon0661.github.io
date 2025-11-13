---
layout: post
title: 웹해킹 - Web Cache Poisoning, Deception
date: 2025-09-30 18:25:23 +0900
category: 웹해킹
---
# 🧳 Web Cache Poisoning / Deception — 원리, 실전 징후, 안전한 재현·탐지, 프록시·앱·CDN 방어 전략(예제 포함)

> ⚠️ **법·윤리 고지**
> 아래 내용은 **본인/조직의 자산**에서 **방어·검증** 목적으로만 사용하세요.
> 무단 테스트는 불법이며, 본 문서는 “**취약 구성을 인지하고 고치는 법**”에 초점을 둡니다.

---

## 0. 요약(Executive Summary)

- **문제**
  - **Unkeyed header**(캐시 키에 포함되지 않는 헤더, 예: `X-Forwarded-Host`, `X-Original-URL`, `X-Forwarded-Proto`)가 **응답 내용에 영향을 주는데 캐시 키에는 반영되지 않으면** 공격자가 **악성 변형 응답을 캐시에 주입**할 수 있습니다.
  - **Cache Deception(캐시 기만)**: 동적/민감 응답을 **정적처럼 보이게**(`.css`, `.js`, `.png` 등) 만들어 **캐시되게** 하고, 이후 **다른 사용자에게 노출**시키는 패턴.
- **징후**
  - 동일 URL에 대해 **가끔 전혀 다른 HTML/JS**가 내려옴.
  - 캐시 통계에서 특정 경로의 **HIT률 급등**, `Age` 헤더가 큰 값으로 응답.
  - 프록시 앞단(=CDN)과 **원본 직결** 응답이 다름.
- **핵심 방어**
  1) **캐시 키를 명시적으로 구성**: **Host + Path + (정규화된) Query** *만* 사용.
  2) **변칙 헤더를 정규화/삭제**: `X-Forwarded-Host/Proto`, 중복/공백/대소문자 변형 모두 **무시**.
  3) **권한/개인화 응답은 절대 캐시 금지**: `Cache-Control: no-store, private`. `Set-Cookie`가 있으면 **무조건 BYPASS**.
  4) **확장자 기반 휴리스틱 금지**: “.css면 캐시” 같은 규칙 **사용하지 않음**.
  5) **민감 경로/메서드 기본 BYPASS**: `/account`, `/checkout`, `/api/*(상태변경)` 등.

---

# 1. 동작 원리(왜 당하는가)

## 1.1 Unkeyed Header Poisoning

- 캐시는 보통 **키**(Host, Path, Query, 일부 헤더 `Vary`)로 응답을 저장합니다.
- 애플리케이션이 `X-Forwarded-Host` 같은 **프록시 전달용 헤더**를 믿고 **HTML 내 절대 URL**이나 **메타 태그/리다이렉트 위치**를 구성한다면,
  - 공격자: `X-Forwarded-Host: evil.tld`로 요청 → 응답 HTML에 `https://evil.tld/...`가 반영
  - 캐시: 이 헤더를 **키에 넣지 않으면** “정상” 키로 **오염된 응답을 저장**
  - 다른 사용자: 같은 URL 요청 → **오염된 응답**(피싱/스크립트 포함)을 받음

## 1.2 Cache Deception

- CDN/프록시가 **확장자 기반** 또는 **Content-Type 기반** 휴리스틱으로 캐시할 때,
  - 공격자: `/profile/.css` 또는 `/profile?cb=.css`처럼 **동적 경로를 정적으로 보이게** 요청
  - 잘못된 규칙: “`.css`면 캐시(장시간 TTL)” → **HTML(민감 데이터)**이 캐시
  - 이후 다수 사용자에게 **민감 정보/개인화 콘텐츠**가 노출될 위험

---

# 2. 안전한 재현·탐지(자신의 스테이징에서만)

> 목적: **우리가 캐시 키를 잘 구성했는지**, **민감 응답이 캐시되지 않는지** 확인.

## 2.1 Node 스크립트 — “X-Forwarded-Host 변조”가 캐시에 반영되는지(안 돼야 정상)

```js
// test/cache-poisoning-check.js  (스테이징 CDN/프록시에만!)
import https from "node:https";

function req(path, headers = {}) {
  return new Promise((resolve, reject) => {
    const req = https.request({
      host: "cdn.staging.example.com",
      path,
      method: "GET",
      headers: { "User-Agent": "cp-check", ...headers },
      rejectUnauthorized: false
    }, res => {
      let body = "";
      res.on("data", d => body += d);
      res.on("end", () => resolve({ status: res.statusCode, headers: res.headers, body }));
    });
    req.on("error", reject).end();
  });
}

const url = "/landing";

const run = async () => {
  // 1) 변조 헤더로 응답 유도(캐시는 키에 넣지 않아야 함)
  const a = await req(url, { "X-Forwarded-Host": "evil.tld" });
  console.log("A:", a.status, a.headers["x-cache"], a.headers["age"]);

  // 2) 정상 요청: A의 내용이 캐시 HIT이면 "오염" 가능성이 있음
  const b = await req(url);
  console.log("B:", b.status, b.headers["x-cache"], b.headers["age"]);

  if (b.headers["x-cache"]?.includes("HIT") && b.body.includes("evil.tld")) {
    console.error("❌ 캐시 키에 없는 헤더가 응답을 오염시켰을 수 있습니다.");
    process.exit(1);
  } else {
    console.log("✅ 안전: 변조 헤더가 캐시 응답에 반영되지 않음(또는 BYPASS/HIT이더라도 내용 다름).");
  }
};
run();
```

> 기대: **언제나 안전**해야 합니다. `X-Forwarded-Host`를 조작해도 캐시 HIT가 나면서 HTML에 그 값이 반영되면 **오염 위험**.

## 2.2 Curl로 Cache Deception 빠른 점검

```bash
# A: 의도적 "정적처럼 보이는" 경로로 접근
curl -sI https://cdn.staging.example.com/account/.css | egrep "Cache-Control|Age|X-Cache"

# B: 동일 URL 재요청 → HIT + Age 증가 + Content-Type: text/html 이면 **위험신호**
curl -sI https://cdn.staging.example.com/account/.css
```

---

# 3. 프록시/Nginx — “키 명시 + 변칙 헤더 정규화 + 민감 BYPASS”

## 3.1 Nginx(리버스 프록시 + 내부 캐시) 기본 템플릿

```nginx
# 캐시 저장소 및 키(명시: Scheme + Host + Path + Query)
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=app_zone:128m max_size=5g inactive=30m use_temp_path=off;

map $request_method $cacheable_method { default 0; GET 1; HEAD 1; }
map $http_cookie $has_cookie { default 0; "~=" 1; }

# 민감 경로 BYPASS
map $request_uri $sensitive {
  default 0;
  ~^/account 1;
  ~^/checkout 1;
  ~^/api/ 1;        # API 전반은 기본 비캐시(업무정책에 따라)
}

server {
  listen 443 ssl http2;
  server_name app.example.com;

  # 1) 캐시 키 명시
  set $cache_key "$scheme$host$request_uri";  # Host/Path/Query만

  # 2) 변칙/전달용 헤더는 백엔드 전달 전 정리
  #    - 원칙: Host는 Nginx가 보유한 값만 전달, XFH/XP는 무시
  proxy_set_header Host $host;
  proxy_set_header X-Forwarded-Host "";
  proxy_set_header X-Original-URL "";
  proxy_set_header X-Forwarded-Proto $scheme;

  # 3) 정규화: URL Query는 애플리케이션/백엔드에서 화이트리스트 정렬 권장
  #    (Nginx 단에서는 표준화 어려우므로 앱/게이트웨이 단계에서 수행)

  location / {
    proxy_cache app_zone;
    proxy_cache_key $cache_key;

    # 캐시 여부 결정: 메서드, 쿠키, 민감경로, 인증 헤더
    set $bypass 0;
    if ($cacheable_method = 0) { set $bypass 1; }
    if ($has_cookie = 1)      { set $bypass 1; }
    if ($sensitive = 1)       { set $bypass 1; }
    if ($http_authorization)  { set $bypass 1; }

    proxy_no_cache $bypass;
    proxy_cache_bypass $bypass;

    # 원본이 Set-Cookie를 주면 무조건 BYPASS + 저장 금지
    proxy_ignore_headers Set-Cookie;
    proxy_no_cache $upstream_http_set_cookie;
    proxy_cache_bypass $upstream_http_set_cookie;

    # 확장자 휴리스틱 금지: Content-Type 상관없이 원본 Cache-Control을 따르거나 BYPASS
    proxy_ignore_headers Expires;  # 오래된 Expires 기반 휴리스틱 회피(정책에 따라)

    # 민감 응답 강제 헤더(원본이 실수했어도 보호)
    add_header Cache-Control "no-store" always;  # <— 민감/개인화 경로 location 블록에서만 사용 권장

    proxy_pass http://app_backend;
  }
}
```

**포인트**
- **키는 명시**(Scheme/Host/Path/Query). **헤더를 키에 넣지 않음**.
- **전달용 헤더 무시/초기화**: `X-Forwarded-Host`, `X-Original-URL` 등 **백엔드에 영향 못 주게**.
- **민감 경로/쿠키/Authorization 시 BYPASS**.
- **원본이 `Set-Cookie`면 절대 캐시 금지**.
- “확장자 = 캐시” 같은 **휴리스틱 제거**.

> 운영에서는 `add_header Cache-Control "no-store"`는 **민감 location**에서만 사용하고, 공용 정적 리소스 location은 `Cache-Control: public, max-age=31536000, immutable`처럼 **명확히** 다르게 설정하세요.

---

# 4. Varnish(VCL) — 캐시 키·정규화

```vcl
vcl 4.1;

backend app { .host = "app"; .port = "8080"; }

sub vcl_recv {
  # 1) 캐시 키 구성 요소만 사용(Host + URL)
  # Host 정규화: 신뢰 도메인만 허용
  if (req.http.host !~ "(?i)^(www\.)?example\.com$") {
    return (synth(400, "Bad host"));
  }

  # 2) 변칙 헤더 제거
  unset req.http.X-Forwarded-Host;
  unset req.http.X-Original-URL;

  # 3) Query 정렬/화이트리스트(예: page, q만 허용)
  set req.url = std.querysort(req.url);
  # (필요하면) req.url = std.queryparam.filter(req.url, "page|q");

  # 4) 인증/쿠키가 있으면 BYPASS
  if (req.http.Authorization || req.http.Cookie) {
    return (pass);
  }

  # 5) 민감 경로 BYPASS
  if (req.url ~ "^/account" || req.url ~ "^/checkout" || req.url ~ "^/api/") {
    return (pass);
  }
}

sub vcl_backend_response {
  # 원본이 Set-Cookie면 캐시 금지
  if (beresp.http.Set-Cookie) {
    set beresp.uncacheable = true;
    return (deliver);
  }
  # 확장자 휴리스틱 금지: 원본 Cache-Control이 없으면 보수적으로 짧게
  if (!beresp.http.Cache-Control) {
    set beresp.ttl = 0s;
  }
}

sub vcl_deliver {
  set resp.http.X-Cache = obj.hits > 0 ? "HIT" : "MISS";
}
```

---

# 5. CDN(CloudFront/Cloudflare/Fastly) 설계 포인트

- **Cache Policy**:
  - **QueryString**: “모두 전달/모두 키 포함”은 위험. **화이트리스트**(필요한 키만) 권장.
  - **Headers**: 키에 포함할 헤더는 **정말 필요한 것만**(예: `Accept-Language` 등) — 기본은 “없음”.
  - **Cookies**: 웬만하면 **쿠키가 있으면 BYPASS**. 특정 쿠키만 키에 포함해야 한다면 **화이트리스트**.
- **Origin Request Policy**: `X-Forwarded-Host/Proto`는 전달해도 **앱이 신뢰하지 않게**(앱에서 허용목록 검사).
- **Rules**: `/account`, `/user/*`, `/checkout` 등은 **Cache BYPASS**.
- **Deception 방지**: 확장자 기반 TTL/캐싱 금지. **Content-Type**도 신뢰하지 말고 **원본 Cache-Control** 우선.
- **캐시 태그/서로게이트 키**(Fastly/Cloudflare): 공개 정적만 태깅, 개인화는 금지.

---

# 6. 애플리케이션 방어(보조 안전망)

## 6.1 “절대 URL/리다이렉트”는 **고정 원본**을 사용

**❌ 취약(Express):** `X-Forwarded-Host`를 신뢰
```js
app.get("/abs", (req, res) => {
  // 공격자가 X-Forwarded-Host: evil.tld → HTML/redirect에 반영되고 캐시되면 위험
  const origin = `${req.protocol}://${req.headers["x-forwarded-host"] || req.headers.host}`;
  res.type("html").send(`<a href="${origin}/login">Login</a>`);
});
```

**✅ 안전: 허용 호스트/고정 ORIGIN 사용**
```js
const APP_ORIGIN = new URL(process.env.APP_ORIGIN || "https://app.example.com");
const ALLOWED_HOSTS = new Set(["app.example.com", "www.example.com"]);

app.get("/abs", (req, res) => {
  const host = req.headers.host?.toLowerCase() || "";
  const origin = ALLOWED_HOSTS.has(host) ? `https://${host}` : APP_ORIGIN.href.replace(/\/$/,"");
  res.type("html").send(`<a href="${origin}/login" rel="noopener">Login</a>`);
});
```

## 6.2 민감 응답은 **무조건 캐시 금지**

```js
app.use((req, res, next) => {
  // 민감 경로/인증된 사용자
  const sensitive = /^\/(account|checkout|settings|user\.)/.test(req.path) || Boolean(req.user);
  if (sensitive) {
    res.set("Cache-Control", "no-store");
    res.set("Pragma", "no-cache");
    res.set("Expires", "0");
  }
  next();
});
```

## 6.3 `Set-Cookie`가 있으면 **절대 캐시 금지**(양쪽에서)

```js
// 응답 직전에 미들웨어로 보수적 차단
app.use((req,res,next) => {
  const setCookie = res.getHeader("Set-Cookie");
  if (setCookie) res.set("Cache-Control", "no-store");
  next();
});
```

## 6.4 “확장자 덫” 무력화

- 라우터에서 **경로 정규화**: `/account/.css` → **404**
- 콘텐츠 협상: “`.css` 요청이면 무조건 `text/css`만” — **HTML 반환 금지**

```js
app.get(/^\/account\/.+\.(css|js|png)$/, (_req, res) => res.status(404).send("Not Found"));
```

---

# 7. 헤더·표준의 올바른 사용

- **Cache-Control**
  - **민감/개인화**: `no-store`
  - **인증 필요하지만 공용 캐시 금지**: `private, no-store`(혹은 최소 `private, no-cache`)
  - **공용 정적**: `public, max-age=31536000, immutable`
- **Vary**
  - 최소화: `Accept-Encoding`, 필요한 경우 `Accept-Language` 정도.
  - 앱이 `Vary`를 남발하면 **캐시 조각화/오염 위험** 증가 → 보수적으로.
- **ETag/Last-Modified**
  - 정적에만. 동적/개인화에선 **지양**(또는 사용자별 ETag → 사실상 BYPASS).

---

# 8. 모니터링·탐지

- **지표**: 경로별 `X-Cache: HIT` 비율, `Age` 분포, `Set-Cookie` 동반 응답의 HIT=0 확인.
- **로그 규칙 예시 (Loki / Splunk)**
  - `X-Forwarded-Host`가 클라이언트 요청에서 **빈도가 비정상적** → 경보.
  - `Content-Type: text/html` + URL이 `\.(css|js|png)$` → 경보.
  - 민감 경로에서 `Cache-Control`이 `no-store`가 아닌 경우 → 경보.

**Loki(LogQL)**
```logql
{service="edge", msg="http_access"} |= "X-Forwarded-Host"
| count_over_time(5m) > 100
```

**Splunk(SPL)**
```spl
index=edge sourcetype=nginx "text/html" "X-Cache:HIT" (uri="*.css" OR uri="*.js")
| stats count by uri, host
```

---

# 9. 사고 대응(런북 요약)

1. **식별**: 특정 URL에서 **의심스런 HIT 증가/콘텐츠 불일치** 확인.
2. **격리**:
   - CDN/프록시에서 **문제 URL BYPASS** 또는 **캐시 무효화(PURGE)**
   - `X-Forwarded-*` 등 **전달 헤더 제거** 강제
3. **조사**: 공격 요청 패턴(IP/UA/리퍼러), 응답 샘플, 앱의 Host/URL 생성 로직 점검
4. **근절**: 키 명시화, 민감 경로 `no-store`, 확장자 휴리스틱 제거, 앱의 **고정 ORIGIN** 적용
5. **복구**: 캐시 전면 재가온(warm-up), 모니터링 알림 상향
6. **교훈화**: CI에 **캐시 스모크 테스트** 추가(§2 스크립트), 보안 리뷰 체크리스트 업데이트

---

# 10. “끝에서 끝까지” 작은 실습(스테이징용)

## 10.1 Express 앱(간단)

```js
import express from "express";
const app = express();

const APP_ORIGIN = "https://app.staging.example.com";

// 절대 URL 생성: 고정 ORIGIN만 사용
app.get("/landing", (req, res) => {
  const html = `
    <!doctype html><meta charset="utf-8">
    <link rel="canonical" href="${APP_ORIGIN}/landing">
    <a href="${APP_ORIGIN}/login">Login</a>
  `;
  // 공용 페이지지만 쿠키가 있으면 private/no-store
  if (req.headers.cookie) res.set("Cache-Control", "private, no-store");
  else res.set("Cache-Control", "public, max-age=60");
  res.type("html").send(html);
});

// 민감 페이지: 절대 no-store
app.get("/account", (req, res) => {
  res.set("Cache-Control", "no-store");
  res.type("html").send("<h1>My Account</h1>");
});

app.get(/^\/account\/.+\.(css|js|png)$/, (_req, res) => res.sendStatus(404));

app.listen(8080, () => console.log("app on 8080"));
```

## 10.2 Nginx 프록시(요약) — §3 템플릿 적용
- `proxy_cache_key` = `$scheme$host$request_uri`
- `X-Forwarded-Host`/`X-Original-URL` 제거
- 쿠키/Authorization/민감 경로 BYPASS
- 정적 휴리스틱 제거

## 10.3 스모크 테스트
- §2 Node 스크립트로 **X-Forwarded-Host 변조**가 캐시에 남는지 확인 → **항상 안전**
- `/account/.css` 두 번 `curl -I` → **항상 MISS/Age 없음** + 404/또는 no-store

---

# 11. 체크리스트(현장용)

- [ ] **캐시 키 명시**(Scheme+Host+Path+Query). 헤더 기반 키 지양.
- [ ] **전달용 헤더 무시/삭제**: XFH, XOP, XP 등.
- [ ] **민감 경로/쿠키/Authorization** → **BYPASS**.
- [ ] **Set-Cookie 응답 절대 캐시 금지**.
- [ ] **확장자 휴리스틱 제거**(정적=캐시라는 가정 금지).
- [ ] **Cache-Control**: 민감 `no-store`, 정적 `public, max-age, immutable`.
- [ ] **Vary 최소화**.
- [ ] **모니터링**: X-Cache/ Age/ HIT률, HTML이 정적 확장자로 캐시되는 패턴 탐지.
- [ ] **런북/무효화 플로우**: 즉시 BYPASS/PURGE 가능해야 함.
- [ ] **CI 스모크**: 모순 헤더/변조 헤더/확장자 덫 테스트 자동화.

---

## 맺음말

Web Cache Poisoning/Deception은 **작은 설정 실수**에서 시작해 **대규모 사용자에게 오염 응답을 전파**할 수 있는 고위험 취약 구성입니다.
**키 명시화**, **전달 헤더 정규화**, **민감 BYPASS**, **휴리스틱 제거**라는 4대 원칙을 적용하고, 간단한 **스모크 테스트**를 CI에 포함시켜 **재발 가능성을 구조적으로 줄이세요**.
