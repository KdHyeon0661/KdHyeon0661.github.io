---
layout: post
title: 웹해킹 - 입력값 검증 & 데이터 Sanitization · 보안 HTTP 헤더 · 세션 관리 보안
date: 2025-09-30 15:25:23 +0900
category: 웹해킹
---
# 🛡️ 입력값 검증 & 데이터 Sanitization · 보안 HTTP 헤더(CSP·XFO·HSTS 등) · 세션 관리 보안(쿠키 Secure/HttpOnly/SameSite)

> ⚠️ **합법·윤리 고지**  
> 본 문서는 **자신의 시스템**과 **허가된 환경**에서 **방어·강화 목적**으로 사용하세요.  
> 난이도 높은 보안 설정을 “한 번에” 적용하기보다는 **점진적 도입 + 관측(로그/리포팅)**을 권장합니다.

---

## 0) 큰 그림(Why)

- **입력값 검증(Validation)**: *의도된 스펙*만 통과시키는 “게이트”. (_들어오기 전_ 막기)
- **Sanitization(정화)**: *콘텐츠(HTML/마크다운 등)*를 제한된 형태로 “무해화”. (_보여주기 전_ 씻기)
- **보안 HTTP 헤더**: 브라우저의 **보안 정책을 서버가 강제**. (_보여주는 동안_ 지키게 하기)
- **세션/쿠키 보안**: 인증 상태의 **탈취·오용을 방지**. (_사용자 상태_ 지키기)

네 가지가 **체인**으로 작동할 때만 효과가 큽니다. 어느 하나라도 빠지면 **우회**가 쉬워집니다.

---

## 1) 입력값 검증(Validation)과 데이터 Sanitization

### 1.1 기본 원칙 7가지

1. **화이트리스트(Allowlist)**: “허용할 것만 정의” — 도메인, 포맷, 길이, 범위, 스킴(https) 등.  
2. **정규화 후 비교**: URL/경로/유니코드 입력은 **정규화(NFKC)** → 비교/검사. (혼동 문자 방지)  
3. **컨텍스트 분리**:  
   - **데이터 필드**(숫자, enum, 날짜) → *검증 후 그대로 저장*  
   - **표시용 필드(HTML/MD)** → *검증 + Sanitization + 출력 인코딩*  
4. **서버가 최종 심판**: 클라이언트 검증은 UX용. 보안은 **서버**에서 결정.  
5. **스키마 기반 검증**: JSON Schema / DTO / Bean Validation 등으로 **한 곳**에서 규정.  
6. **오류는 구체적이되 과도한 힌트 금지**: 어떤 필드가 왜 틀렸는지 정도만.  
7. **로그 & 레이트리미트**: 반복 실패/의심스러운 패턴은 **속도 제한·알림**.

---

### 1.2 검증 레이어 — 언어별 예제

#### 1.2.1 Node.js(Express) — zod 또는 Joi
```javascript
// validation/user.js
import { z } from "zod";

export const RegisterSchema = z.object({
  email: z.string().email().max(254),
  password: z.string().min(12).max(128)
    .regex(/[A-Z]/, "upper")
    .regex(/[a-z]/, "lower")
    .regex(/[0-9]/, "digit")
    .regex(/[^A-Za-z0-9]/, "symbol"),
  displayName: z.string().trim().min(1).max(40)
    .regex(/^[\p{L}\p{N}\s._-]+$/u, "namechars"),
}).strict();
```

```javascript
// routes/auth.js
app.post("/register", async (req, res) => {
  const parsed = RegisterSchema.safeParse(req.body);
  if (!parsed.success) return res.status(400).json({ error: "invalid" });
  // … 생성 로직
  res.sendStatus(201);
});
```

#### 1.2.2 Python(Flask/FastAPI) — Pydantic
```python
from pydantic import BaseModel, EmailStr, constr

class Register(BaseModel):
    email: EmailStr
    password: constr(min_length=12, max_length=128)
    displayName: constr(regex=r'^[\w.\-\s]{1,40}$')

@app.post("/register")
def register():
    data = Register(**request.json)   # 유효성 실패 시 422
    # … 처리
    return "", 201
```

#### 1.2.3 Java(Spring Boot) — Bean Validation
```java
class RegisterDto {
  @Email @Size(max=254) public String email;
  @Pattern(regexp="(?=.*[A-Z])(?=.*[a-z])(?=.*\\d)(?=.*\\W).{12,128}") public String password;
  @Pattern(regexp="^[\\p{L}\\p{N} ._\\-]{1,40}$") public String displayName;
}

@PostMapping("/register")
public ResponseEntity<?> register(@Valid @RequestBody RegisterDto dto){ … }
```

---

### 1.3 Sanitization(콘텐츠 무해화) — 언제·어떻게?

- **텍스트 데이터**: 단순 출력이면 **출력 인코딩**(HTML 엔티티)만으로 충분.
- **HTML/Markdown 입력을 “제한적 허용”**해야 한다면, **화이트리스트 Sanitizer** 필수.

#### 1.3.1 HTML Sanitizer 예제
- **브라우저(클라)**: DOMPurify  
- **서버**:  
  - Node: `isomorphic-dompurify` 또는 `sanitize-html`  
  - Python: `bleach`  
  - Java: `OWASP Java HTML Sanitizer`

```javascript
// Node 서버 측 — 제한된 허용 태그/속성
import createDOMPurify from "isomorphic-dompurify";
import { JSDOM } from "jsdom";

const window = new JSDOM("").window;
const DOMPurify = createDOMPurify(window);

const POLICY = {
  ALLOWED_TAGS: ["b","i","em","strong","u","p","ul","ol","li","a","br","code","pre"],
  ALLOWED_ATTR: { "a": ["href","title"] },
  ALLOW_UNKNOWN_PROTOCOLS: false
};

export function sanitizeUserHtml(raw) {
  const clean = DOMPurify.sanitize(raw ?? "", POLICY);
  // 링크 스킴 화이트리스트(중요)
  return clean.replace(/href="([^"]+)"/g, (_, href) => {
    try {
      const u = new URL(href, "https://example.com");
      if (u.protocol === "https:" || u.protocol === "http:") return `href="${u.href}"`;
    } catch {}
    return `href="#"`;
  });
}
```

```python
# Python — bleach
import bleach
ALLOWED_TAGS   = ["b","i","em","strong","u","p","ul","ol","li","a","br","code","pre"]
ALLOWED_ATTRS  = {"a": ["href","title"]}
ALLOWED_SCHEMES= ["http","https"]

def sanitize_user_html(raw: str) -> str:
    return bleach.clean(raw or "", tags=ALLOWED_TAGS, attributes=ALLOWED_ATTRS,
                        protocols=ALLOWED_SCHEMES, strip=True)
```

```java
// Java — OWASP Java HTML Sanitizer
PolicyFactory POLICY = Sanitizers.FORMATTING.and(Sanitizers.LINKS);
String safeHtml = POLICY.sanitize(userHtml);
```

> ⚠️ **Anti-Pattern**  
> - 직접 필터/정규식으로 `<script>`만 막기: `onerror`, `javascript:` 등 **수백가지 우회**가 존재합니다.  
> - 출력 시 `innerHTML`/`v-html`/`dangerouslySetInnerHTML` 남용: Sanitizer 이후에만.

---

### 1.4 URL/경로/파일 검증(간단 체크리스트)

- URL: **스킴(https) 화이트리스트**, `new URL(raw, base)`로 정규화 후 **호스트/포트 허용 목록** 확인.  
- 경로: **루트 고정 + 정규화 + prefix 검사**.  
- 파일: **확장자 + MIME + 시그니처(매직넘버)** 3중 확인, **웹루트 외부 저장**, **다운로드 전용 헤더**.

---

## 2) 보안 HTTP 헤더 적용(브라우저 보안 정책)

> **핵심**: CSP(콘텐츠 보안 정책)를 중심으로 **X-Frame-Options·HSTS·nosniff·Referrer-Policy** 등과 **세트**로 설정.

### 2.1 CSP(Content-Security-Policy) — 전략

- **목표**: XSS·리소스 하이재킹을 브라우저 레벨에서 차단.  
- **패턴 1: Nonce 기반 엄격 CSP(권장)**  
  - `script-src 'nonce-<랜덤>' 'strict-dynamic' https:; object-src 'none'; base-uri 'none'`  
  - 모든 인라인 스크립트에 **서버가 부여한 nonce** 필요. 외부 스크립트도 **동적 신뢰 전파(strict-dynamic)**.  
- **패턴 2: 호스트 화이트리스트 중심(과도하게 넓어지기 쉬움)**  
  - `script-src 'self' cdn1.example cdn2.example` … (인라인 금지)  
- **도입 단계**: **Report-Only** 모드로 먼저 배포 → 위반 리포트 수집 → 본 모드 전환.

#### 2.1.1 Express(Helmet) + Nonce 예제
```javascript
import crypto from "node:crypto";
import helmet from "helmet";

app.use((req, res, next) => {
  res.locals.nonce = crypto.randomBytes(16).toString("base64");
  next();
});

app.use((req, res, next) => {
  helmet.contentSecurityPolicy({
    useDefaults: false,
    directives: {
      "default-src": ["'none'"],
      "base-uri": ["'none'"],
      "object-src": ["'none'"],
      "img-src": ["'self'", "data:", "https:"],
      "style-src": ["'self'"],             // 인라인 스타일 금지
      "font-src": ["'self'", "https:"],
      "connect-src": ["'self'", "https:"], // API, WebSocket wss:
      "form-action": ["'self'"],
      "frame-ancestors": ["'self'"],
      "script-src": ["'strict-dynamic'", `'nonce-${res.locals.nonce}'`, "https:"],
      // "report-to": "csp-endpoint" // 보고 채널(Reporting-Endpoints 병행 가능)
    }
  })(req, res, next);
});

app.get("/", (req, res) => {
  res.send(`
<!doctype html><meta charset="utf-8">
<script nonce="${res.locals.nonce}">
  // 인라인 스크립트는 nonce 필수
  console.log("CSP OK");
</script>
`);
});
```

#### 2.1.2 Nginx에서 CSP(정적) — 간단형
```nginx
add_header Content-Security-Policy "default-src 'none'; base-uri 'none'; object-src 'none'; script-src 'self'; style-src 'self'; img-src 'self' data: https:; connect-src 'self' https:; frame-ancestors 'self'; form-action 'self'" always;
```
> 동적 nonce가 필요하면 **애플리케이션**에서 설정하고, Nginx는 **기타 헤더**를 보조합니다.

---

### 2.2 X-Frame-Options vs `frame-ancestors`
- 현대 표준은 **CSP의 `frame-ancestors`**.  
- 구형 호환을 위해 **X-Frame-Options: DENY** 또는 `SAMEORIGIN`을 함께 추가.

```nginx
add_header X-Frame-Options "DENY" always;  # 또는 SAMEORIGIN
add_header Content-Security-Policy "frame-ancestors 'self'" always;
```

---

### 2.3 HSTS(Strict-Transport-Security)

- **목표**: 브라우저가 **항상 HTTPS로만** 접속하게 강제.  
- **주의**: HTTPS가 완전히 준비되지 않은 서브도메인이 있다면 **includeSubDomains** 사용 전 신중.  
- **예시**:
```nginx
# HSTS는 오직 HTTPS 응답에서!
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```
- `preload`는 브라우저 내장 목록에 등록을 의미(도입 전 충분한 검증 필요).

---

### 2.4 기타 중요 헤더

```nginx
# MIME 스니핑 금지
add_header X-Content-Type-Options "nosniff" always;

# 리퍼러 최소화(권장 기본)
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# 권한(센서/카메라 등) 최소화
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

# 교차 출처 격리(필요 서비스에 한해)
add_header Cross-Origin-Opener-Policy "same-origin" always;
add_header Cross-Origin-Embedder-Policy "require-corp" always;
add_header Cross-Origin-Resource-Policy "same-site" always;

# 민감 응답은 캐시 금지
add_header Cache-Control "no-store" always;
```

> COOP/COEP/CORP는 **SharedArrayBuffer** 등 특수 기능이나 강력한 격리 필요 시에만.  
> 무조건 켜면 **서드파티 위젯/리소스**가 깨질 수 있습니다.

---

## 3) 세션 관리 보안 — 쿠키 속성과 운영 전략

### 3.1 쿠키 속성 요약

- **Secure**: HTTPS 연결에서만 전송.  
- **HttpOnly**: JavaScript에서 **접근 불가**(XSS 시 탈취 난이도 상승).  
- **SameSite**: 크로스사이트 전송 제약.
  - `Lax`(권장 기본): **톱레벨 내비게이션**만 가능(일반 링크·GET 폼 전송)  
  - `Strict`: 동일 사이트에서만 전송(UX 보수적)  
  - `None`: 크로스사이트 허용(반드시 **Secure** 필요)

> SPA + 외부 도메인 인증/결제 리디렉트 등 **크로스사이트 플로우**가 있으면 `SameSite=None; Secure` 조합을 신중히 사용.

---

### 3.2 Express — 세션/쿠키 예제

```javascript
import session from "express-session";
import crypto from "node:crypto";

app.set("trust proxy", 1); // 프록시 뒤라면 설정

app.use(session({
  name: "sid",
  secret: [process.env.S1, process.env.S2], // 키 로테이션
  cookie: {
    httpOnly: true,
    secure: true,           // HTTPS만
    sameSite: "lax",        // 기본
    path: "/",
    maxAge: 1000 * 60 * 30  // 30분(유휴 만료)
  },
  rolling: true,            // 요청 시 만료 연장(선택)
  resave: false,
  saveUninitialized: false, // 동의 전 세션 생성 금지
}));

// 로그인 시 세션 고정 공격 방지: 세션 ID 재발급
app.post("/login", (req, res, next) => {
  // 자격 검증…
  req.session.regenerate(err => {
    if (err) return next(err);
    req.session.userId = user.id;
    res.sendStatus(204);
  });
});

// 로그아웃
app.post("/logout", (req, res) => {
  req.session.destroy(() => res.clearCookie("sid").sendStatus(204));
});
```

#### CSRF 토큰(세션 쿠키와 함께)
```javascript
import csurf from "csurf";
app.use(csurf({ cookie: false })); // 세션 기반 토큰
// 폼에 {{ csrfToken }} 삽입, AJAX는 헤더(X-CSRF-Token)로 전송
```

> 프론트에서 쿠키에 접근할 필요가 **없습니다**. 토큰은 서버가 렌더링(또는 별도의 엔드포인트에서 JSON으로) 내려주고, 요청 헤더/바디에 담아 보냅니다.

---

### 3.3 Flask — 쿠키 & 세션

```python
app.config.update(
    SESSION_COOKIE_NAME="sid",
    SESSION_COOKIE_HTTPONLY=True,
    SESSION_COOKIE_SECURE=True,
    SESSION_COOKIE_SAMESITE="Lax",  # "Strict" | "None"
    PERMANENT_SESSION_LIFETIME=1800  # 30분
)

# 로그인 후 세션 고정 방지: 새 세션 발급
from flask import session
@app.post("/login")
def login():
    # 검증…
    session.clear()        # 기존 데이터 제거
    session["uid"] = user.id
    return "", 204

@app.post("/logout")
def logout():
    session.clear()
    resp = make_response("", 204)
    resp.delete_cookie("sid")
    return resp
```

- CSRF: `Flask-WTF` 또는 커스텀 토큰(더블 서브밋 패턴 가능)  
- 민감 뷰는 `@login_required` + **Absolute Timeout**(예: 8~12시간) 재로그인 요구

---

### 3.4 Spring Security — 세션/쿠키/세션 고정 방지

```java
@EnableWebSecurity
class SecurityConfig {
  @Bean
  SecurityFilterChain filter(HttpSecurity http) throws Exception {
    http
      .requiresChannel(ch -> ch.anyRequest().requiresSecure())
      .sessionManagement(sm -> sm
          .sessionFixation(sf -> sf.migrateSession()) // 세션 고정 방지
          .invalidSessionUrl("/login")
          .maximumSessions(1))
      .csrf(csrf -> csrf.ignoringRequestMatchers("/api/public"))
      .headers(h -> h
          .xssProtection(x -> x.block(true))      // (구형 브라우저)
          .frameOptions(f -> f.deny())
          .contentTypeOptions(Customizer.withDefaults())
          .referrerPolicy(r -> r.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
      );
    return http.build();
  }
}
```

- SameSite 설정은 앱 서버/프록시에서 쿠키 생성 시 지정(스프링 6+: `ResponseCookie` 사용 가능).  
- 로그인 후 **세션 ID 교체**가 기본(`migrateSession`).

---

### 3.5 JWT를 사용할 때(필독)

- **Access JWT**는 **짧은 TTL(5~15분)**, **Refresh Token**은 **길다란 TTL + 서버 저장(블랙리스트/로테이션)**.  
- 전달 방식:  
  - **권장**: Access JWT = **Authorization 헤더**(`Bearer`) / Refresh = **HttpOnly, Secure, SameSite 쿠키**  
  - 전부 쿠키로 운용할 경우에도 **HttpOnly+Secure+SameSite** 철저, CSRF 토큰과 결합.  
- **재발급(Refresh)** 시 **재사용 감지**(동일 토큰 중복 제출 → 세션 강제 종료)  
- 클라이언트 저장소(localStorage)는 **XSS에 취약** → 되도록 피함.  
- 쿠키 Domain은 **최소 범위(호스트 전용)**, Path도 최소화.

---

## 4) “끝에서 끝까지” 예제 — Express 미니 앱

### 4.1 기능
- 입력 검증(zod)  
- HTML sanitization(서버)  
- 헤더(Helmet, CSP Nonce)  
- 세션 + CSRF + 보안 쿠키

```javascript
import express from "express";
import session from "express-session";
import crypto from "node:crypto";
import helmet from "helmet";
import csurf from "csurf";
import { z } from "zod";
import createDOMPurify from "isomorphic-dompurify";
import { JSDOM } from "jsdom";

const app = express();
app.set("trust proxy", 1);
app.use(express.json());
app.use(express.urlencoded({ extended: false }));

// 1) CSP Nonce
app.use((req,res,next)=>{ res.locals.nonce = crypto.randomBytes(16).toString("base64"); next(); });

// 2) Helmet + CSP
app.use(helmet({
  xssFilter: false,
  contentSecurityPolicy: {
    useDefaults: false,
    directives: {
      "default-src": ["'none'"],
      "base-uri": ["'none'"],
      "object-src": ["'none'"],
      "frame-ancestors": ["'self'"],
      "img-src": ["'self'", "data:", "https:"],
      "style-src": ["'self'"],
      "connect-src": ["'self'", "https:"],
      "form-action": ["'self'"],
      "script-src": ["'strict-dynamic'", (req, res) => `'nonce-${res.locals.nonce}'`, "https:"],
    }
  },
  referrerPolicy: { policy: "strict-origin-when-cross-origin" },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
  frameguard: { action: "deny" },
  noSniff: true,
  crossOriginOpenerPolicy: { policy: "same-origin" },
  crossOriginEmbedderPolicy: { policy: "require-corp" },
  crossOriginResourcePolicy: { policy: "same-site" }
}));

// 3) Session
app.use(session({
  name: "sid",
  secret: [process.env.S1, process.env.S2],
  cookie: { httpOnly: true, secure: true, sameSite: "lax", maxAge: 30*60*1000, path:"/" },
  resave: false, saveUninitialized: false, rolling: true
}));

// 4) CSRF
app.use(csurf());

// 5) Validation + Sanitization
const window = new JSDOM("").window;
const DOMPurify = createDOMPurify(window);
const PostSchema = z.object({ title: z.string().trim().min(1).max(120), body: z.string().min(1).max(5000) });

app.get("/", (req,res)=>{
  res.type("html").send(`
<!doctype html><meta charset="utf-8">
<h1>New Post</h1>
<form method="post" action="/post">
  <input type="hidden" name="_csrf" value="${req.csrfToken()}">
  <input name="title" placeholder="title"><br>
  <textarea name="body" placeholder="allowed: b/i/strong/a …"></textarea><br>
  <button>Publish</button>
</form>
<script nonce="${res.locals.nonce}">console.log('ready')</script>
`);
});

app.post("/post", (req,res)=>{
  const parsed = PostSchema.safeParse(req.body);
  if (!parsed.success) return res.status(400).send("invalid");
  const { title, body } = parsed.data;
  const clean = DOMPurify.sanitize(body, {
    ALLOWED_TAGS:["b","i","em","strong","u","p","ul","ol","li","a","br","code","pre"],
    ALLOWED_ATTR: { "a": ["href","title"] },
    ALLOW_UNKNOWN_PROTOCOLS:false
  });
  res.type("html").send(`<h2>${escapeHtml(title)}</h2><div>${clean}</div>`);
});

function escapeHtml(s=""){ return s.replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c])); }

app.listen(3000, ()=>console.log("https://localhost:3000"));
```

---

## 5) 리버스 프록시(Nginx) 스니펫

```nginx
server {
  listen 443 ssl http2;
  server_name app.example.com;

  # HSTS (HTTPS 응답만)
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

  # XFO + CSP(Frame Ancestors)
  add_header X-Frame-Options "DENY" always;
  add_header Content-Security-Policy "frame-ancestors 'self'" always;

  # 기타 보안 헤더
  add_header X-Content-Type-Options "nosniff" always;
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;
  add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

  # 민감 응답 캐시 금지
  add_header Cache-Control "no-store" always;

  location / {
    proxy_set_header Host $host;
    proxy_pass http://app:3000;
  }
}

# HTTP → HTTPS 리다이렉트(필수)
server {
  listen 80;
  server_name app.example.com;
  return 301 https://$host$request_uri;
}
```

---

## 6) 테스트 페이로드 & 기대 행동(“막혀야 정상”)

- **HTML 주입**: `<img src=x onerror=alert(1)>` → Sanitizer로 제거  
- **자바스크립트 링크**: `<a href="javascript:alert(1)">` → 링크 스킴 검사로 거절/무력화  
- **메타 리다이렉트**: `<meta http-equiv=refresh content="0;url=...">` → 태그 불허  
- **Frame 공격**: 외부 도메인에서 `<iframe src="...">` → XFO/CSP로 차단  
- **혼합콘텐츠(HTTP 이미지)**: CSP `img-src`/HSTS 정책에 의해 차단  
- **크로스사이트 요청 쿠키 전송**: SameSite=Lax로 대부분 차단(Top-level 링크 GET만 허용)  
- **세션 고정**: 로그인 후 **세션 ID 변경** 확인  
- **HTTP로 접근**: HSTS + 301 리다이렉트 + 브라우저 강제 HTTPS

---

## 7) “마이그레이션” 전략(CSP·세션·헤더)

1. **관측 먼저**: CSP **Report-Only**로 위반 리포트 수집 → 외부 스크립트/스타일 정리  
2. **Nonce 전환**: 인라인 스크립트에 nonce 부여, `unsafe-inline` 제거  
3. **세션 재설계**: 로그인 시 세션 재발급, `HttpOnly+Secure+SameSite` 기본화  
4. **헤더 번들링**: Nginx/앱에서 **공통 미들웨어**로 일괄 적용  
5. **릴리즈 전략**: Canary → 전체 롤아웃, 모니터링/롤백 계획 명시

---

## 8) 자주 하는 실수 ↔ 교정

| 실수 | 문제 | 교정 |
|---|---|---|
| 클라이언트 검증만 믿음 | 프록시/봇으로 쉽게 우회 | **서버 검증**이 최종 |
| `<script>`만 금지 | 이벤트/링크/스타일로 우회 | **화이트리스트 Sanitizer** |
| CSP에 `unsafe-inline` 남김 | XSS 차단력 급감 | **Nonce 기반**으로 전환 |
| HSTS를 HTTP 응답에도 추가 | 무의미/혼동 | **HTTPS 응답에만** |
| SameSite=None인데 Secure 미적용 | 최신 브라우저에서 **무시** | `None; Secure` **쌍** 유지 |
| 로그인 후 세션 ID 유지 | **세션 고정** 위험 | **세션 재발급** |
| 쿠키 Domain을 상위 도메인에 | 서브도메인 탈취 시 위험 | **호스트 전용**(가능한 최소) |

---

## 9) 보안 체크리스트(현장용 요약)

- [ ] 모든 입력 **스키마 검증**(서버) + 오류/속도 제한  
- [ ] 표시용 콘텐츠는 **Sanitizer**로 정화 + **링크 스킴 화이트리스트**  
- [ ] CSP(Nonce, strict-dynamic), XFO/`frame-ancestors`, HSTS, nosniff, Referrer-Policy, Permissions-Policy  
- [ ] 민감 응답은 **Cache-Control: no-store**  
- [ ] 세션: **Secure+HttpOnly+SameSite**, 로그인 시 **세션 ID 재발급**, 유휴/절대 만료  
- [ ] CSRF: 토큰(세션 기반 또는 더블 서브밋) + 상태 변경은 **POST/PUT/DELETE**  
- [ ] 로깅/경보: CSP 위반, 프레임 차단, 반복 실패, 의심 IP/UA  
- [ ] 문서화/훈련: 운영팀이 헤더/CSP/세션 전략을 **이해**하고 **조정**할 수 있게

---

### 맺음말
입력 검증·Sanitization·보안 헤더·세션 보안은 **맞물려 돌아가는 하나의 시스템**입니다.  
이 문서의 샘플을 **작은 범위**에서 먼저 적용해 보고, **로그/리포트**로 관찰하면서 **점진**적으로 확대하세요.