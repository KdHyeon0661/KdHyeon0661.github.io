---
layout: post
title: 웹해킹 - Web Cache Poisoning, Deception
date: 2025-09-30 18:25:23 +0900
category: 웹해킹
---
# Web Cache Poisoning / Deception - 원리, 실전 징후, 안전한 재현·탐지, 프록시·앱·CDN 방어 전략

## 요약(Executive Summary)

### 1. 문제 정의

- **Web Cache Poisoning (WCP)**  
  - 공유 캐시(CDN/프록시)가 같은 URL에 대해 **“어떤 응답을 저장할지”**를 결정하는 기준(=캐시 키)이 잘못 정의되어 있으면,  
    공격자가 **특정 요청 필드(헤더, Query 등)를 조작해 “오염된 응답”을 캐시에 넣고**,  
    이후 정상 사용자가 **그 오염된 응답을 HIT**하도록 만들 수 있다.
  - 특히, **응답 내용에 영향을 주는 헤더/필드인데 캐시 키에는 포함되지 않는 것**을 흔히 **Unkeyed Header/Field**라고 부른다.
    - 예: `X-Forwarded-Host`, `X-Forwarded-Proto`, `X-Original-URL`, 일부 커스텀 헤더 등.

- **Web Cache Deception (WCD)**  
  - 애플리케이션은 “**동적 페이지**”를 반환하면서도,  
    캐시는 URL 또는 확장자(`.css`, `.js`, `.png`, `.ico` 등)만 보고 **“정적 리소스 같다”**고 착각하고 장기간 캐시할 수 있다.
  - 공격자는 **동적·민감 경로를 정적처럼 보이게**(예: `/account.php/notexist.jpg`, `/profile/.css`) 만들어 피해자의 개인화 응답을 캐시에 저장시키고,  
    그 URL을 다시 호출해 **다른 사용자의 정보**를 가져올 수 있다.

- **위협이 여전히 크다는 근거**
  - 2024년 대규모 측정 연구에서는 Tranco Top 1,000 도메인(22k+ 서브도메인)을 자동 스캔해 본 결과,  
    **17%의 도메인(172개)**에서 실제로 WCP 취약점이 발견되었고, 다양한 새로운 공격 벡터(조건부 요청, 범위 요청, HTTP/2 전용 이슈 등)가 보고되었다. :contentReference[oaicite:0]{index=0}  
  - Web Cache Deception 역시 Alexa Top 5K 중 **295개 사이트**에서 실제 취약 구성이 발견되었으며,  
    단순 계정 페이지 뿐 아니라 **보안 토큰·세션 관련 정보 누출**로 이어질 수 있음이 입증되었다. :contentReference[oaicite:1]{index=1}  
  - 최신 데이터에서 전 세계 웹 애플리케이션 공격 패턴 중 상당수가 **설정 오류·오용(misconfiguration)** 에서 비롯되며,  
    특히 웹 애플리케이션을 노린 공격 중 **잘못된 설정을 악용한 공격 비율이 약 45%** 수준으로 보고된다. :contentReference[oaicite:2]{index=2}  
  - 주요 연구에 따르면 Alexa/Tranco 상위 사이트의 **70% 이상이 CDN/프록시 기반 캐시를 사용**하고 있어,  
    캐시 설정 한 번 잘못하면 **“한 번의 오염 → 수백·수천 사용자에게 전파”**되는 구조적 리스크를 안고 있다. :contentReference[oaicite:3]{index=3}  

### 2. 전형적인 실전 징후

운영/모니터링 관점에서 보이는 징후는 다음과 같다.

- **동일 URL**(Host/Path/Query 동일)에 대해
  - 어떤 사용자는 정상 HTML,  
  - 어떤 사용자는 **전혀 다른 HTML/JS**(피싱/에러 페이지/이상한 리다이렉트)를 받는다.
- CDN/프록시의 통계에서 특정 경로의
  - `X-Cache: HIT` 비율이 **갑자기 급등**,  
  - 응답 헤더 `Age` 값이 비정상적으로 크게 유지된다.
- **CDN/프록시를 거친 응답과 원본 서버의 직접 응답이 다르다**.
- `.css`, `.js`, `.png` 같은 정적처럼 보이는 URL인데
  - `Content-Type: text/html` + 쿠키/개인화 내용이 들어있다.
- 특정 에러 응답(예: `Access Denied`, `Maintenance Page`)이 오래 HIT 되며  
  **정상 요청까지 계속 에러 상태**가 된다(CP-DoS 형태의 캐시 기반 DoS).

### 3. 핵심 방어 원칙(요약)

1. **캐시 키를 명시적으로 설계**  
   - Scheme + Host + Path + (정규화된 Query) **만** 사용하고,  
     공격자가 조작 가능한 헤더는 기본적으로 **키에 포함하지 않는다**.
2. **변칙 헤더 정규화/삭제**  
   - `X-Forwarded-Host`, `X-Original-URL`, `X-Forwarded-Proto` 등은  
     **프록시/로드밸런서 단계에서 화이트리스트·초기화**하고,  
     애플리케이션은 이를 **무신뢰(untrusted)** 데이터로 취급한다.
3. **권한·개인화 응답은 절대 공유 캐시에 저장하지 않기**  
   - 민감 경로, 인증된 사용자, `Set-Cookie` 동반 응답은  
     무조건 `Cache-Control: no-store` 또는 최소한 **공유 캐시 BYPASS**.
4. **확장자 기반 휴리스틱 금지**  
   - “`.css`로 끝나면 장기 캐시” 같은 규칙을 사용하지 말고,  
     **반드시 원본의 Cache-Control 정책** 또는 **명시적 캐시 정책**을 사용한다.
5. **민감 경로/메서드 BYPASS**  
   - `/account`, `/user`, `/checkout`, `/api/*(상태 변경)` 등은  
     **기본 BYPASS** 후 개별 비즈니스 요구에 따라 예외적으로만 캐시.

---

## 1. HTTP 캐시와 위협 모델

### 1.1 HTTP 캐시 동작 기초

여기서는 **공유 캐시(CDN/프록시)** 를 중심으로 이야기한다.

- **캐시 키(Cache Key)**  
  - 보통 `(Scheme, Host, Path, Query, 일부 헤더)` 조합으로 정의된다.
  - 이 키가 같다면 **같은 오브젝트**로 취급하고, 저장/조회한다.
- **캐시 저장(Write)** 흐름
  1. 클라이언트 → 캐시: 요청
  2. 캐시: 해당 키에 저장된 오브젝트 없으면 MISS → 원본으로 프록시
  3. 원본 → 캐시: 응답 + `Cache-Control`/`Expires`/`Set-Cookie` 등
  4. 캐시: 정책에 따라 저장 여부·TTL 결정 후 저장
- **캐시 조회(Read)** 흐름
  1. 클라이언트 → 캐시: 요청
  2. 캐시: 키를 계산해 저장된 오브젝트 조회
  3. 있으면 HIT → 저장된 응답 반환(`Age` 증가)
  4. 없거나 만료면 MISS → 원본으로 프록시 후 저장(선택)

### 1.2 캐시 키 설계가 왜 중요한가

- **키에 포함되지 않는 값 = 서로 다른 요청도 동일 오브젝트로 취급**  
  - 예: 캐시 키가 `(Host, Path)`만 쓰는데,
    - 애플리케이션이 `X-Forwarded-Host`에 따라 HTML 내용을 바꾸면,
    - 공격자가 `X-Forwarded-Host: evil.tld`로 만든 HTML이  
      “정상 Host + Path” 키 밑으로 저장되어 **모든 사용자가 오염된 응답을 받는** 구조가 된다.
- **반대로, 너무 많은 필드를 키에 넣어도 문제**  
  - `User-Agent`, `Cookie`, `Accept-Language` 등을 무분별하게 키에 포함하면  
    캐시가 **쓸데없이 조각화(fragmentation)** 되어 효율이 떨어진다.
  - 특히 헤더를 키에 넣으면서 **앱이 그 헤더를 신뢰**할 경우,  
    WCP/WCD와는 다른 형태의 **권한 우회/정보 노출**이 생기기도 한다.

### 1.3 위협 모델

- **공격자 능력**
  - HTTP 요청의 **모든 필드**(Method, Path, Query, Header, Body)를 조작 가능
  - 다만, **“다른 피해자 사용자 브라우저가 어떤 요청을 보내게 할지”**에 대해서는
    - 링크 클릭/리다이렉트/이미지 `<img src=...>` 삽입/이메일 링크 등으로 간접 영향.
- **가정**
  - 캐시는 다수 사용자가 공유하는 **CDN/프록시/리버스 프록시**이다.
  - 애플리케이션/프록시 설정은 완벽하지 않으며,
    - 일부 헤더/Query에 대한 **정규화가 미흡**하고,
    - 민감 응답에 `no-store`를 빼먹거나,
    - 확장자 기준 캐시 규칙을 사용하고 있을 수 있다.

---

## 2. Web Cache Poisoning / Deception 개념 확장

### 2.1 Unkeyed Header Poisoning (핵심 패턴)

앞서 언급했듯이, **응답 내용을 바꾸는 헤더인데, 캐시 키에는 들어가지 않는 헤더**를 이용하는 패턴이다.

#### 2.1.1 단순 시나리오

1. 공격자 → CDN 요청

   ```http
   GET /landing HTTP/1.1
   Host: app.example.com
   X-Forwarded-Host: evil.example.net
   ```

2. 애플리케이션은 `X-Forwarded-Host`를 신뢰하여 HTML 내 링크를 구성:

   ```html
   <a href="https://evil.example.net/login">Login</a>
   ```

3. CDN의 캐시 키가 `scheme + Host + path + query` 뿐이라면,
   - **키**는 `https://app.example.com/landing` 하나뿐.
   - 공격자의 요청도, 다른 사용자의 정상 요청도 **같은 키**.

4. 나중에 일반 사용자:

   ```http
   GET /landing HTTP/1.1
   Host: app.example.com
   ```

   - CDN: HIT  
   - 오염된 HTML을 그대로 전달 → **피싱/리다이렉트** 가능.

#### 2.1.2 변형: 상태 코드·헤더 오염

- `Location` 헤더에 `X-Forwarded-Host`가 반영되어  
  **리다이렉트 대상 URL이 공격자 사이트**가 될 수 있다.
- `Access-Control-Allow-Origin`, `CSP(Content-Security-Policy)`, `Set-Cookie` 등  
  민감한 헤더가 공격자가 조작한 값으로 캐시될 경우,  
  **CORS 우회, CSP 무력화, 세션 고정(session fixation)** 로 이어질 수 있다.

### 2.2 Web Cache Deception (WCD)

#### 2.2.1 전형적인 Path Confusion 패턴

1. 공격자가 피해자에게 다음과 같은 링크를踏ませ도록 유도:

   ```
   https://app.example.com/account.php/nonexist.jpg
   ```

2. 애플리케이션(예: PHP)은 `/account.php`만 보고 뒤의 `/nonexist.jpg`를 무시하고  
   **정상 HTML(사용자의 계정 페이지)** 를 반환한다.

3. 캐시는 URL이 `.jpg`로 끝난다고 보고  
   - 정책: “`.jpg` / `.png` 등 이미지 → 장기간 캐시, `public`”
   - 결과: **민감 HTML**이 “이미지처럼” 캐시된다.

4. 공격자는 나중에 동일 URL을 요청해  
   **다른 사용자의 계정 페이지 HTML**을 그대로 획득한다.

#### 2.2.2 Query 기반 Deception

- URL: `/profile?cb=.css`  
  - 일부 캐시는 Query 규칙을 잘못 설정해 `cb` 파라미터를 무시하거나,  
    `.css` 문자열만 보고 정적 리소스로 간주할 수 있다.
- 또는 `/account.css?user=alice` 처럼  
  **실제는 동적 핸들러**인데 확장자만 `.css`인 경우,  
  잘못된 TTL/정책이 적용되면 마찬가지로 민감 HTML이 캐시된다.

#### 2.2.3 실전 연구 결과 요약

- 대규모 측정 결과, 상위 수천 개 사이트 중 **수백 개**에서  
  WCD로 인해 계정 정보, 토큰, 기타 민감 데이터가 캐시에서 노출될 수 있음이 보고되었다. :contentReference[oaicite:4]{index=4}  
- 많은 CDN이 기본적으로 **확장자 기반 캐시 규칙**을 제공하고,  
  운영자가 별도 검토 없이 이를 사용하면서 문제가 악화된다.

---

## 3. HTTP 캐시 헤더·동작 심화

WCP/WCD를 제대로 막으려면, **캐시 헤더를 올바르게 쓰는 것**이 핵심이다.

### 3.1 Cache-Control

- **민감/개인화 응답**
  - `Cache-Control: no-store`
    - 어떤 캐시에도 저장하지 말라는 의미.
  - 필요하다면 `private`도 함께 사용: `private, no-store`
    - 브라우저의 **개인 캐시**에만 저장(혹은 전혀 저장 X)하게 하고 공유 캐시에서는 제외.
- **인증이 필요하지만, 브라우저 캐시는 허용하는 응답**
  - `Cache-Control: private, no-cache`
    - 브라우저 캐시는 허용하되, 서버에 재검증을 요구.
- **완전 정적 자산(CDN 캐시 적극 활용)**
  - `Cache-Control: public, max-age=31536000, immutable`
    - 1년 캐시, 변경 시에는 파일명을 바꾸는 방식으로 운용.

### 3.2 Vary

- `Vary`는 캐시 키에 **어떤 헤더를 포함시킬지** 알려주는 메커니즘이다.
  - 예: `Vary: Accept-Encoding` → `gzip` 여부에 따라 다른 오브젝트.
  - `Vary: Accept-Language` → 언어별 HTML.
- 하지만 `Vary`를 남발하면 캐시 조각화 + 오염 리스크를 키운다.
  - 최소한으로 유지: `Accept-Encoding`, 필요 시 `Accept-Language` 정도.

### 3.3 ETag / Last-Modified

- 정적 리소스에서는 효율적인 재검증 수단.
- 하지만 **개인화된 HTML**에 ETag를 부여해 공유 캐시가 재사용하려 하면  
  사용자간 정보 섞임이 생길 수 있으므로,  
  **개인화/민감 응답에서는 되도록 사용하지 않거나, private 캐시에 국한**해야 한다.

---

## 4. Unkeyed Header Poisoning — 예제 기반 상세 분석

### 4.1 취약한 Express 코드 예제

#### 4.1.1 취약한 절대 URL 생성

```js
// ❌ 취약 패턴: X-Forwarded-Host 신뢰
app.get("/abs", (req, res) => {
  const origin = `${req.protocol}://${
    req.headers["x-forwarded-host"] || req.headers.host
  }`;

  const html = `
    <!doctype html><meta charset="utf-8">
    <a href="${origin}/login">Login</a>
  `;
  res.type("html").send(html);
});
```

- 공격자:
  ```http
  GET /abs HTTP/1.1
  Host: app.example.com
  X-Forwarded-Host: evil.example.net
  ```
- 애플리케이션은 `https://evil.example.net/login`을 포함한 HTML을 만든다.
- 캐시 키가 `scheme + Host + path + query`라면,
  - 키: `https://app.example.com/abs`
  - 오염된 HTML이 캐시에 들어가고,
  - 이후 정상 사용자는 **오염된 HTML**을 받는다.

#### 4.1.2 안전한 패턴: 고정 ORIGIN + 허용 호스트

```js
const APP_ORIGIN = new URL(process.env.APP_ORIGIN || "https://app.example.com");
const ALLOWED_HOSTS = new Set(["app.example.com", "www.example.com"]);

app.get("/abs", (req, res) => {
  const host = (req.headers.host || "").toLowerCase();
  const origin = ALLOWED_HOSTS.has(host)
    ? `https://${host}`
    : APP_ORIGIN.href.replace(/\/$/, "");

  const html = `
    <!doctype html><meta charset="utf-8">
    <a href="${origin}/login" rel="noopener">Login</a>
  `;
  res.type("html").send(html);
});
```

- `X-Forwarded-Host`는 **무시**하고,
- Host 헤더 역시 **허용 목록에 없으면 고정 ORIGIN**으로 강제.

### 4.2 Node 스크립트로 실제 캐시 오염 가능성 확인

이 스크립트는 **스테이징 CDN/프록시에서만** 사용해야 한다.

```js
// test/cache-poisoning-check.js
import https from "node:https";

function req(path, headers = {}) {
  return new Promise((resolve, reject) => {
    const r = https.request(
      {
        host: "cdn.staging.example.com",
        path,
        method: "GET",
        headers: { "User-Agent": "cp-check", ...headers },
        rejectUnauthorized: false,
      },
      (res) => {
        let body = "";
        res.on("data", (d) => (body += d));
        res.on("end", () =>
          resolve({ status: res.statusCode, headers: res.headers, body })
        );
      }
    );
    r.on("error", reject).end();
  });
}

const url = "/landing";

const run = async () => {
  // 1) 변조 헤더로 오염 시도
  const a = await req(url, { "X-Forwarded-Host": "evil.tld" });
  console.log("A:", a.status, a.headers["x-cache"], a.headers["age"]);

  // 2) 정상 요청
  const b = await req(url);
  console.log("B:", b.status, b.headers["x-cache"], b.headers["age"]);

  if (b.headers["x-cache"]?.includes("HIT") && b.body.includes("evil.tld")) {
    console.error("❌ 캐시 키에 없는 헤더가 응답을 오염시켰을 수 있습니다.");
    process.exit(1);
  } else {
    console.log(
      "✅ 안전: 변조 헤더가 캐시 응답에 반영되지 않음(또는 BYPASS/HIT이더라도 내용 다름)."
    );
  }
};

run().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

- **기대 결과**
  - `A`는 MISS/HIT 어떤 것이든 상관없지만,  
    `B`가 HIT일 때도 HTML 내에 `evil.tld`가 나타나면 **취약 가능성**.
- 이 스크립트를 CI/CD에서 **스모크 테스트**로 돌리면
  - 새로운 캐시 규칙/프록시 설정을 도입할 때마다  
    **예상치 못한 Unkeyed Header 의존이 생겼는지 자동 감지**할 수 있다.

---

## 5. Web Cache Deception — 예제와 탐지

### 5.1 Curl로 빠르게 Deception 의심 패턴 찾기

```bash
# A: "정적처럼 보이는" 경로 접근
curl -sI https://cdn.staging.example.com/account/.css \
  | egrep "HTTP/|Cache-Control|Age|X-Cache|Content-Type"

# B: 동일 URL 재요청
curl -sI https://cdn.staging.example.com/account/.css \
  | egrep "HTTP/|Cache-Control|Age|X-Cache|Content-Type"
```

- 위험 신호:
  - 두 번째 요청에서 `X-Cache: HIT` + `Age` 증가
  - `Content-Type: text/html`
  - `Cache-Control`이 `public` 혹은 TTL이 길게 설정

### 5.2 Express 기반 테스트 앱 (정상 설계 예)

```js
import express from "express";
const app = express();

const APP_ORIGIN = "https://app.staging.example.com";

// 공용 랜딩 페이지
app.get("/landing", (req, res) => {
  const html = `
    <!doctype html><meta charset="utf-8">
    <link rel="canonical" href="${APP_ORIGIN}/landing">
    <a href="${APP_ORIGIN}/login">Login</a>
  `;

  // 공용 페이지지만 쿠키가 있으면 private/no-store
  if (req.headers.cookie) {
    res.set("Cache-Control", "private, no-store");
  } else {
    res.set("Cache-Control", "public, max-age=60");
  }
  res.type("html").send(html);
});

// 민감 페이지: 절대 no-store
app.get("/account", (req, res) => {
  res.set("Cache-Control", "no-store");
  res.type("html").send("<h1>My Account</h1>");
});

// "확장자 덫" 무력화: /account/**.css, .js, .png → 404
app.get(/^\/account\/.+\.(css|js|png)$/, (_req, res) => res.sendStatus(404));

app.listen(8080, () => console.log("app on 8080"));
```

- `/account`는 **무조건 no-store**.
- `/account/**.css` 같은 경로를 통한 WCD 시도는 **404**로 차단.

---

## 6. 프록시/Nginx 레벨 방어 — 키 명시 + 헤더 정규화 + 민감 BYPASS

### 6.1 Nginx 기본 템플릿

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=app_zone:128m \
  max_size=5g inactive=30m use_temp_path=off;

map $request_method $cacheable_method { default 0; GET 1; HEAD 1; }
map $http_cookie $has_cookie        { default 0; "~=" 1; }

# 민감 경로 BYPASS
map $request_uri $sensitive {
  default 0;
  ~^/account   1;
  ~^/checkout  1;
  ~^/api/      1;
}

server {
  listen 443 ssl http2;
  server_name app.example.com;

  # 1) 캐시 키 명시: Scheme + Host + Path + Query
  set $cache_key "$scheme$host$request_uri";

  # 2) 변칙/전달용 헤더 정리
  proxy_set_header Host $host;
  proxy_set_header X-Forwarded-Host "";
  proxy_set_header X-Original-URL "";
  proxy_set_header X-Forwarded-Proto $scheme;

  location / {
    proxy_cache      app_zone;
    proxy_cache_key  $cache_key;

    # 3) 캐시 BYPASS 판단
    set $bypass 0;
    if ($cacheable_method = 0) { set $bypass 1; }  # 비 GET/HEAD
    if ($has_cookie = 1)       { set $bypass 1; }  # 쿠키 존재
    if ($sensitive = 1)        { set $bypass 1; }  # 민감 경로
    if ($http_authorization)   { set $bypass 1; }  # 인증 헤더

    proxy_no_cache     $bypass;
    proxy_cache_bypass $bypass;

    # 4) 원본 Set-Cookie가 있으면 절대 캐시 금지
    proxy_ignore_headers Set-Cookie;
    proxy_no_cache       $upstream_http_set_cookie;
    proxy_cache_bypass   $upstream_http_set_cookie;

    # 5) 오래된 Expires 기반 휴리스틱 회피
    proxy_ignore_headers Expires;

    # (선택) 민감 location에서만:
    # add_header Cache-Control "no-store" always;

    proxy_pass http://app_backend;
  }
}
```

**핵심 포인트**

- 캐시 키는 **명확하고 단순하게**.
- `X-Forwarded-Host`, `X-Original-URL` 등은 **백엔드에 전달하지 않거나,  
  애플리케이션이 무시하도록** 설계한다.
- 민감 경로/쿠키/Authorization/`Set-Cookie` 존재 시 **항상 BYPASS**.

---

## 7. Varnish/VCL 레벨 방어

```vcl
vcl 4.1;

backend app {
  .host = "app";
  .port = "8080";
}

sub vcl_recv {
  # 1) Host 정규화
  if (req.http.host !~ "(?i)^(www\.)?example\.com$") {
    return (synth(400, "Bad host"));
  }

  # 2) 변칙 헤더 제거
  unset req.http.X-Forwarded-Host;
  unset req.http.X-Original-URL;

  # 3) Query 정렬/필터링
  set req.url = std.querysort(req.url);
  # 필요 시 특정 파라미터만 허용:
  # set req.url = std.queryparam.filter(req.url, "page|q");

  # 4) 인증/쿠키 있으면 BYPASS
  if (req.http.Authorization || req.http.Cookie) {
    return (pass);
  }

  # 5) 민감 경로 BYPASS
  if (req.url ~ "^/account" || req.url ~ "^/checkout" || req.url ~ "^/api/") {
    return (pass);
  }
}

sub vcl_backend_response {
  # 6) 원본이 Set-Cookie를 주면 캐시 금지
  if (beresp.http.Set-Cookie) {
    set beresp.uncacheable = true;
    return (deliver);
  }

  # 7) 원본이 Cache-Control을 지정하지 않은 경우 보수적 정책
  if (!beresp.http.Cache-Control) {
    set beresp.ttl = 0s;
  }
}

sub vcl_deliver {
  set resp.http.X-Cache = (obj.hits > 0) ? "HIT" : "MISS";
}
```

- WCP/WCD 방어를 위해
  - **Host 정규화** + **Query 정규화** + **민감 경로 BYPASS** +  
    **Set-Cookie 응답 캐시 금지**를 일괄 적용.

---

## 8. CDN/클라우드 인프라에서의 정책 설계

여기서는 특정 벤더 이름 대신 **일반적인 개념**으로 설명한다.

### 8.1 Cache Policy

- **QueryString 처리**
  - “모든 쿼리를 키에 포함” 또는 “모든 쿼리를 무시”는 둘 다 위험.
  - 비즈니스에서 사용하는 소수의 파라미터만 화이트리스트로 명시:
    - 예: `page`, `q`, `sort`, `lang` 등.
- **Headers**
  - 키에 포함할 헤더는 정말 필요한 것만:
    - `Accept-Encoding`, `Accept-Language` 등.
  - `X-Forwarded-*`, `X-Original-*` 등은  
    **키에도, 앱 로직에도 영향을 주지 않게 설계**.

### 8.2 Origin Request Policy

- 캐시 → 원본으로 전달되는 요청에서
  - `Host`, `X-Forwarded-Host`, `X-Forwarded-Proto`, `X-Real-IP` 등은  
    **게이트웨이가 통제**해야 한다.
  - 앱이 이를 읽더라도 **허용 목록/고정값 기반으로만** 동작하게 한다.

### 8.3 WCD 방지 정책

- “확장자 기반 TTL” 기능이 있더라도
  - `.css`, `.js`, `.png`, `.jpg`만 따로 묶어서 정책을 정의하되,
  - 해당 경로에 동적 핸들러가 섞여 있는지 **애플리케이션 팀과 함께 점검**.
- `Content-Type`을 기반으로 캐시 정책을 정의할 수 있다면
  - `.css` 요청인데 `text/html`이 오면 **BYPASS + 경보**를 띄우는 규칙을 추가.

---

## 9. 애플리케이션 레벨 방어 (보조 안전망)

프록시/캐시 레벨에서 막더라도, 애플리케이션이 올바르게 대응하지 않으면 **취약 구성이 다시 등장**할 수 있다.

### 9.1 절대 URL 생성 규칙

- 항상 **고정 ORIGIN + 허용 호스트** 패턴을 사용한다.
- 프레임워크별 팁:
  - **Express / Node.js**
    - `req.protocol`, `req.headers.host`를 그대로 신뢰하지 말고,  
      환경 변수 `APP_ORIGIN`이나 구성 파일에서 ORIGIN을 가져와 사용.
  - **Spring (Java)**
    - `ServletUriComponentsBuilder.fromCurrentContextPath()` 등을 사용할 때도  
      프록시가 삽입한 헤더에 의존하는 옵션을 최소화.
  - **Django / Flask**
    - `request.host_url` 대신 설정된 `SITE_URL`/`PREFERRED_URL_SCHEME` 등 고정 값 우선.

### 9.2 민감 응답 캐시 금지 미들웨어

#### Express 예제

```js
function cacheGuard(req, res, next) {
  const sensitivePath = /^\/(account|checkout|settings|user\b)/.test(req.path);
  const isAuthenticated = Boolean(req.user); // 예: Passport.js 등

  if (sensitivePath || isAuthenticated) {
    res.set("Cache-Control", "no-store");
    res.set("Pragma", "no-cache");
    res.set("Expires", "0");
  }
  next();
}

app.use(cacheGuard);
```

### 9.3 Set-Cookie 동반 응답 자동 보호

```js
// 응답 직전에 보수적으로 체크
app.use((req, res, next) => {
  const originalSetHeader = res.setHeader.bind(res);
  res.setHeader = (name, value) => {
    if (name.toLowerCase() === "set-cookie") {
      // 어떤 Set-Cookie든 나오면 no-store 부여
      const existing = res.getHeader("Cache-Control");
      if (!existing) {
        res.setHeader("Cache-Control", "no-store");
      } else if (!/no-store/i.test(String(existing))) {
        res.setHeader("Cache-Control", existing + ", no-store");
      }
    }
    return originalSetHeader(name, value);
  };
  next();
});
```

- 이를 통해 개발자가 `Set-Cookie`를 추가하면서 `Cache-Control`을 잊어도,  
  **공유 캐시에 저장되는 것을 자동으로 차단**할 수 있다.

### 9.4 “확장자 덫” 무력화 라우팅

```js
// /account/*.{css,js,png} 는 절대 HTML을 반환하지 않도록 강제
app.get(/^\/account\/.+\.(css|js|png)$/, (_req, res) => {
  res.status(404).send("Not Found");
});
```

- 실제 비즈니스 요구가 있더라도,  
  `/static/account/...`와 같이 **명확하게 분리된 경로**로 정적 리소스를 제공한다.

---

## 10. 모니터링·탐지 전략

### 10.1 주요 지표

- **경로별 X-Cache HIT 비율**
  - `/account`, `/user`, `/checkout` 등에서 HIT 비율이 0이 아니면 의심.
- **Age 분포**
  - 특정 경로에서 `Age` 값이 비정상적으로 크고 일정하게 유지되면  
    캐시 TTL이 과도하거나, 에러 페이지가 오랫동안 HIT 되는 중일 수 있다.
- **민감 경로의 Cache-Control 준수 여부**
  - `/account`, `/user`에서 `Cache-Control: no-store`가 아닌 응답이 있으면 경보.

### 10.2 로그 분석 예시 (Loki)

```logql
# 1) X-Forwarded-Host 요청이 비정상적으로 많은 경우
{service="edge", msg="http_access"} |= "X-Forwarded-Host"
| count_over_time(5m) > 100

# 2) 정적 확장자인데 HTML + HIT
{service="edge", msg="http_access"} |= ".css" |= "text/html" |= "X-Cache:HIT"
```

### 10.3 Splunk 예시

```spl
# 정적 확장자(.css/.js) + Content-Type: text/html + HIT
index=edge sourcetype=nginx "text/html" "X-Cache:HIT" (uri="*.css" OR uri="*.js")
| stats count by uri, host
```

- 이 쿼리를 기반으로 대시보드를 구성하면,
  - 특정 URL에서 **“정적처럼 보이는데 HTML이 오고 있다”**는 패턴을 빠르게 잡을 수 있다.

---

## 11. 사고 대응(런북) 정리

### 11.1 식별 단계

1. 사용자 제보 혹은 모니터링에서
   - “같은 URL인데 다른 HTML/JS가 뜬다”
   - “특정 URL에서 계속 Access Denied/에러 페이지만 뜬다”
2. 로그/메트릭에서
   - 해당 URL의 `X-Cache: HIT` 비율, `Age` 값, `Content-Type` 등을 확인.
   - 원본 서버를 직접 호출했을 때와 CDN/프록시를 통했을 때 응답 차이 비교.

### 11.2 격리 단계

1. CDN/프록시 설정에서 **문제 URL에 대해 BYPASS** 적용.
2. 이미 캐시된 객체를 **PURGE/Invalidate**.
3. `X-Forwarded-*`, `X-Original-*` 등 전달용 헤더를  
   최소한 일시적으로라도 **초기화/삭제**하도록 게이트웨이 규칙 수정.

### 11.3 조사 단계

1. 공격자의 요청 패턴 분석
   - IP, User-Agent, Referer, 사용된 Query/헤더.
2. 캐시 로그 재검토
   - 어떤 요청이 최초로 오염된 응답을 생성했는지 확인.
3. 애플리케이션 코드 리뷰
   - Host/URL/리다이렉트 생성 로직,
   - Cache-Control/Set-Cookie 헤더 처리 로직 점검.

### 11.4 근절 단계

1. **캐시 키 명시화**  
   - CDN/프록시/Varnish 등 모든 레이어에서 키 구성 요소를 재검토.
2. **민감 경로 `no-store` 강제**  
   - `/account`, `/user`, `/checkout`, `/api/*` 등.
3. **확장자 휴리스틱 제거**  
   - `.css`, `.js`만 보고 TTL을 주는 규칙을 없애고,  
     원본의 Cache-Control 또는 별도의 명시 정책으로 전환.
4. **앱 레벨에서 고정 ORIGIN 패턴 적용**  
   - `X-Forwarded-Host`에 의존하는 코드를 제거 혹은 allow-list 기반으로 재작성.

### 11.5 복구 단계

1. 주요 경로에 대해 캐시를 **재가온(warm-up)**  
   - 비정상적인 TTL이나 헤더가 더 이상 나오지 않는지 확인.
2. 모니터링 룰 상향
   - 민감 경로에서 HIT가 발생하면 **경보**가 뜨게 조정.

### 11.6 교훈화 / DevSecOps 반영

- CI 파이프라인에 **캐시 스모크 테스트** 포함:
  - Unkeyed Header 변조,
  - Web Cache Deception 패턴(`/account.php/nonexist.jpg`, `/account/.css`) 등을 자동 시도.
- 보안 리뷰 체크리스트에
  - “캐시 키 설계 검토”
  - “민감 경로/쿠키/Authorization 응답 캐시 정책 검토” 항목 추가.

---

## 12. “끝에서 끝까지” 작은 실습 정리

1. **Express 앱** (위 §5.2 예제) 준비
   - `/landing` → 공용 페이지 (60초 캐시, 쿠키 있으면 `no-store`)
   - `/account` → 무조건 `no-store`
   - `/account/**.css` → 404
2. **Nginx/프록시 템플릿** (위 §6) 적용
   - `proxy_cache_key = $scheme$host$request_uri`
   - 민감 경로/쿠키/Authorization → BYPASS
   - `Set-Cookie` 응답은 캐시 저장 금지
3. **Node 스크립트 (§4.2) + curl (§5.1)** 로 스모크 테스트
   - `X-Forwarded-Host` 변조 후에도 캐시에 오염된 응답이 남지 않는지 확인.
   - `/account/.css`는 항상 MISS + `no-store` + 404임을 확인.

---

## 13. 최종 체크리스트(현장용)

- [ ] **캐시 키 명시**  
      - Scheme + Host + Path + 정규화된 Query  
      - 헤더 기반 키 사용 최소화 (`Vary` 최소화)
- [ ] **전달용 헤더 무시/삭제**  
      - `X-Forwarded-Host`, `X-Original-URL` 등
- [ ] **민감 경로/쿠키/Authorization → BYPASS**  
      - `/account`, `/user`, `/checkout`, `/api/*` 등
- [ ] **Set-Cookie 응답 → 절대 공유 캐시 저장 금지**
- [ ] **확장자 기반 휴리스틱 제거**  
      - “`.css`면 캐시” 같은 규칙 사용 금지
- [ ] **Cache-Control 정책 일관성**  
      - 민감: `no-store`  
      - 정적: `public, max-age, immutable`
- [ ] **Vary 헤더 최소화**  
      - `Accept-Encoding`, 필요시 `Accept-Language` 정도만
- [ ] **모니터링/경보**  
      - 정적 확장자 + `Content-Type: text/html` + `X-Cache:HIT`  
      - 민감 경로에서 HIT 발생 시 경보
- [ ] **런북/무효화 플로우 정비**  
      - 특정 URL/패턴에 대해 즉시 BYPASS/PURGE 가능
- [ ] **CI 스모크 테스트**  
      - Unkeyed Header 변조, Cache Deception 패턴 자동화

---

## 14. 맺음말

Web Cache Poisoning / Deception은 **“설정 한 줄 실수”**가 **대규모 사용자에게 확산**되는 취약 구성이다.  
최신 대규모 측정 연구와 업계 데이터는,  
오늘날에도 상위 도메인의 상당수가 여전히 이 문제에 취약하며,  
웹 애플리케이션 공격 패턴에서 **설정 오류·오용이 큰 비중**을 차지하고 있음을 보여준다. :contentReference[oaicite:5]{index=5}  

이 문서에서 정리한 네 가지 축:

1. **캐시 키 명시화**
2. **전달 헤더 정규화**
3. **민감 응답 BYPASS / no-store**
4. **확장자 기반 휴리스틱 제거 + 모니터링·CI 테스트**

를 **프록시·CDN·앱·DevSecOps 파이프라인 전체에 일관되게 적용**하면,  
Web Cache Poisoning / Deception을 **구조적으로 재발하기 어렵게 만드는 안전한 기본선**을 구축할 수 있다.