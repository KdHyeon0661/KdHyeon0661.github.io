---
layout: post
title: 웹해킹 - strict-dynamic + Nonce 회전
date: 2025-10-10 21:25:23 +0900
category: 웹해킹
---
# CSP `strict-dynamic` + Nonce 회전

**— 개념·동작 원리 · 위협 모델 · 실무 설계(Nonce 생성/회전/캐싱 고려) · 프레임워크별 설정(Express/Nginx/Spring/Django/Flask) · 안전한 동적 로더 패턴 · 마이그레이션(인라인 이벤트 제거) · 롤아웃/모니터링(Report-Only/Reporting API) · 테스트/체크리스트**

## 한눈에 보기 (Executive Summary)

- **효과**
  - `script-src 'nonce-…' 'strict-dynamic'`을 쓰면, **Nonce가 붙은 부트스트랩 스크립트**(초기 신뢰 스크립트)가 **동적으로 추가하는 `<script>` 요소**도 **자동으로 신뢰**됩니다(추가 `<script>`에 Nonce를 또 붙이지 않아도 됨).
  - 반대로 **페이지에 주입된 임의 `<script>`**(Nonce 없음)와 **인라인 이벤트 핸들러**(예: `onclick="..."`)는 차단됩니다.
- **주의**
  - **Nonce 재사용 금지(요청/문서마다 고유)**.
  - **인라인 이벤트 핸들러 지양**(필요하면 `'unsafe-hashes'`/`'unsafe-hashed-attributes'`를 쓰는 특수 예외만, 권장 X).
  - **구형 브라우저 호환 전략**(필요 시 **해시 기반** 병행 또는 외부 파일로 분리).

---

## 위협 모델 & 동작 원리 (왜 `strict-dynamic`인가?)

### 기본 CSP의 한계

- 호스트 화이트리스트(예: `script-src 'self' https://cdn.example.com`)만으로는
  1) **서드파티 호스트 감시 어려움**(그 호스트가 변조되면? 경로가 늘어나면?),
  2) **동적 로드** 시 매번 **URL/호스트를 정책에 추가**해야 하는 **운영 부담**이 큽니다.

### `strict-dynamic`의 핵심 의미

- **아이디어**: “**신뢰한 스크립트가 로드한 것**은 **그 자체로 신뢰**하자.”
- 결과: Nonce/해시로 **초기(부트스트랩) 스크립트**만 신뢰하면, 그 스크립트가 `document.createElement('script')` 등으로 불러오는 체인까지 자동 허용됩니다.
- 이때 **호스트 화이트리스트는 2차적**(구형 브라우저 대비)이며, **동적 체인이 우선**합니다.

> 정리: **정말로 필요한 건 “처음에 무엇을 신뢰할 것인가”**입니다 → **그 답이 Nonce(또는 해시)**.

---

## 설계 & 회전(재사용 금지)

### 생성 규칙

- **CSPRNG**(cryptographically secure)로 **128비트 이상**을 생성, **Base64** 등으로 인코딩.
- **요청/문서마다 고유**(페이지 요청 시마다 새로운 Nonce).
- 템플릿 렌더링에서 **모든 `<script>`**(부트스트랩/초기 인라인)에 **동일 Nonce**를 꽂아줌(해당 문서 안에서는 동일 값 사용 가능).

### 보관 & 캐싱 주의

- Nonce는 **서버가 응답마다 만든 비밀**입니다.
- **공유 캐시**(CDN) 레이어에서 **HTML을 캐싱**하면 **Nonce가 재사용**될 수 있으므로 위험:
  - 해결책: **엣지에서 주입**(Edge Function/Workers), **HTML을 private 캐싱**, 또는 **ESI/동적 조립**으로 Nonce만 런타임 삽입.

### 헤더에 Nonce 반영

- `Content-Security-Policy: script-src 'nonce-<Base64>' 'strict-dynamic' ...`
- HTML 내 `<script nonce="<Base64>"> ... </script>` 또는 `<script nonce="<Base64>" src="/app.js"></script>`

---

## 표준 정책 스니펫(추천 기본형)

> **목표**: ① XSS/주입 차단, ② 필요한 동적 로드만 허용, ③ 보고·롤백 용이

```http
Content-Security-Policy:
  base-uri 'self';
  object-src 'none';
  script-src
    'nonce-<RANDOM_BASE64>'   /* 부트스트랩 신뢰 */
    'strict-dynamic'          /* 동적 체인 신뢰 */
    'report-sample';          /* 차단 샘플 보고(디버깅 도움) */
  script-src-elem
    'nonce-<RANDOM_BASE64>' 'strict-dynamic' 'report-sample';
  script-src-attr
    'none';                   /* 인라인 이벤트 핸들러 금지(onclick 등) */
  connect-src 'self' https:;
  img-src 'self' data: https:;
  style-src 'self' 'unsafe-inline';  /* 점진 도입: 가능하면 nonce/hash로 전환 */
  font-src 'self' https: data:;
  frame-ancestors 'self';
  base-uri 'self';
  upgrade-insecure-requests;
  report-to csp-endpoint;
```

> **메모**
> - `script-src-attr 'none'`로 **인라인 이벤트**를 원천 차단 → `addEventListener`로 전환.
> - `style-src`는 초기에 `'unsafe-inline'`일 수 있으나, 점차 **`'nonce-…'` 또는 해시**로 옮기는 것을 권장.
> - `report-to`(또는 `report-uri`)로 위반 리포트를 수집(아래 §9).

---

## 서버·프레임워크별 구현 예

### Node/Express — **요청마다 Nonce 생성 & 헤더 설정**

```js
// app.js
import crypto from 'node:crypto';
import express from 'express';

const app = express();

app.use((req, res, next) => {
  // 16바이트(128비트) 이상 권장
  const nonce = crypto.randomBytes(16).toString('base64');
  res.locals.nonce = nonce;

  const csp = [
    "base-uri 'self'",
    "object-src 'none'",
    `script-src 'nonce-${nonce}' 'strict-dynamic' 'report-sample'`,
    `script-src-elem 'nonce-${nonce}' 'strict-dynamic' 'report-sample'`,
    "script-src-attr 'none'",
    "connect-src 'self' https:",
    "img-src 'self' data: https:",
    "style-src 'self' 'unsafe-inline'", // 점진 개선
    "font-src 'self' https: data:",
    "frame-ancestors 'self'",
    "upgrade-insecure-requests",
    "report-to csp-endpoint"
  ].join('; ');

  res.setHeader('Content-Security-Policy', csp);
  next();
});

// 템플릿에서 Nonce 사용 예
app.get('/', (_req, res) => {
  const nonce = res.locals.nonce;
  res.send(`
    <!doctype html>
    <meta charset="utf-8">
    <title>CSP strict-dynamic Demo</title>
    <script nonce="${nonce}">
      // 부트스트랩 스크립트(신뢰됨)
      // 동적으로 서브 스크립트 로드
      const s = document.createElement('script');
      s.src = '/static/sub.js'; // 이 sub.js에는 nonce가 없어도 'strict-dynamic' 덕분에 허용됨
      document.head.appendChild(s);
    </script>
    <button id="b">Click</button>
    <script nonce="${nonce}">
      // 인라인 이벤트 핸들러 대신 addEventListener
      document.getElementById('b').addEventListener('click', () => alert('OK'));
    </script>
  `);
});

app.use('/static', express.static('public', { immutable: true, maxAge: '1y' }));

app.listen(8080, () => console.log('http://localhost:8080'));
```

> **포인트**
> - 위 `/static/sub.js`는 **Nonce 없이**도 실행됩니다(부트스트랩이 신뢰되었고 `strict-dynamic` 적용).
> - 주입된 `<script>`(예: 공격자가 DOM에 삽입)에는 Nonce가 없으므로 **차단**됩니다.

---

### Nginx — **헤더 주입**

```nginx
# nginx.conf (애플리케이션에서 nonce를 헤더/템플릿에 삽입할 수 없다면,
# 보통은 앱 레이어에서 CSP를 관리하는 것을 권장합니다.)
# 아래는 정적 정책 예(Nonce 없이 → 해시/엄격 호스트 기반 등으로 구성 필요)

add_header Content-Security-Policy "default-src 'self'; object-src 'none';" always;

# 권장: Nonce/strict-dynamic는 앱 레이어에서 동적으로 생성된 값을 써야 하므로
# Nginx 단독 구성보다는 애플리케이션이 CSP를 설정하도록 설계.

```

---

### — **필터로 Nonce 생성 & 헤더**

```java
// NonceFilter.java
@Component
public class NonceFilter extends OncePerRequestFilter {
  @Override
  protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
      throws ServletException, IOException {
    String nonce = java.util.Base64.getEncoder()
      .encodeToString(java.security.SecureRandom.getSeed(16)); // 또는 SecureRandom().nextBytes()
    req.setAttribute("cspNonce", nonce);

    String csp = String.join("; ",
      "base-uri 'self'",
      "object-src 'none'",
      "script-src 'nonce-" + nonce + "' 'strict-dynamic' 'report-sample'",
      "script-src-elem 'nonce-" + nonce + "' 'strict-dynamic' 'report-sample'",
      "script-src-attr 'none'",
      "connect-src 'self' https:",
      "img-src 'self' data: https:",
      "style-src 'self' 'unsafe-inline'",
      "font-src 'self' https: data:",
      "frame-ancestors 'self'",
      "upgrade-insecure-requests",
      "report-to csp-endpoint"
    );
    res.setHeader("Content-Security-Policy", csp);
    chain.doFilter(req, res);
  }
}
```

템플릿(Thymeleaf 등)에서:
```html
<script th:nonce="${#httpServletRequest.getAttribute('cspNonce')}">
  // 부트스트랩 코드
</script>
```

---

### Django — **미들웨어로 Nonce & 템플릿**

```python
# middleware.py

import secrets
from django.utils.deprecation import MiddlewareMixin

class CSPNonceMiddleware(MiddlewareMixin):
    def process_request(self, request):
        request.csp_nonce = secrets.token_urlsafe(16)

    def process_response(self, request, response):
        nonce = getattr(request, 'csp_nonce', None)
        if nonce:
            policy = "; ".join([
                "base-uri 'self'",
                "object-src 'none'",
                f"script-src 'nonce-{nonce}' 'strict-dynamic' 'report-sample'",
                f"script-src-elem 'nonce-{nonce}' 'strict-dynamic' 'report-sample'",
                "script-src-attr 'none'",
                "connect-src 'self' https:",
                "img-src 'self' data: https:",
                "style-src 'self' 'unsafe-inline'",
                "font-src 'self' https: data:",
                "frame-ancestors 'self'",
                "upgrade-insecure-requests",
                "report-to csp-endpoint",
            ])
            response['Content-Security-Policy'] = policy
        return response
```

템플릿(Jinja2/DTL):
{% raw %}
```html
<script nonce="{{ request.csp_nonce }}">
  // 부트스트랩
</script>
```
{% endraw %}

---

### Flask — **Nonces in Jinja**

```python
from flask import Flask, g, render_template, make_response
import secrets

app = Flask(__name__)

@app.before_request
def gen_nonce():
    g.nonce = secrets.token_urlsafe(16)

@app.after_request
def set_csp(resp):
    n = getattr(g, 'nonce', None)
    if n:
        csp = "; ".join([
            "base-uri 'self'",
            "object-src 'none'",
            f"script-src 'nonce-{n}' 'strict-dynamic' 'report-sample'",
            f"script-src-elem 'nonce-{n}' 'strict-dynamic' 'report-sample'",
            "script-src-attr 'none'",
            "connect-src 'self' https:",
            "img-src 'self' data: https:",
            "style-src 'self' 'unsafe-inline'",
            "font-src 'self' https: data:",
            "frame-ancestors 'self'",
            "upgrade-insecure-requests",
            "report-to csp-endpoint"
        ])
        resp.headers['Content-Security-Policy'] = csp
    return resp

@app.route("/")
def index():
    return make_response(render_template("index.html", nonce=g.nonce))
```

`index.html`:
{% raw %}
```html
<script nonce="{{ nonce }}">
  // 부트스트랩
  const s = document.createElement('script');
  s.src = "/static/app.js";
  document.head.appendChild(s);  // strict-dynamic으로 허용
</script>
```
{% endraw %}

---

## 안전한 동적 로더 패턴 (허용/차단 데모)

### 허용(의도된 체인)

```html
<script nonce="RANDOM_NONCE">
  // 1) 신뢰된 부트스트랩 코드
  const s1 = document.createElement('script');
  s1.src = 'https://cdn.example.com/vendor.bundle.js'; // Nonce 없음
  document.head.appendChild(s1);

  s1.onload = () => {
    // 2) 벤더가 또 다른 스크립트 로드
    const s2 = document.createElement('script');
    s2.src = '/static/app.bundle.js'; // Nonce 없음
    document.head.appendChild(s2);
  };
</script>
```
- **결과**: 모두 허용(`'strict-dynamic'` 덕분).

### 차단(주입/인라인 이벤트)

```html
<!-- 공격자가 주입한 스크립트 (Nonce 없음) -->
<script>alert('pwn');</script>        <!-- 차단 -->
<button onclick="steal()">Click</button> <!-- script-src-attr 'none' 로 차단 -->
```

---

## 인라인 이벤트 핸들러 → **addEventListener** 마이그레이션

### 나쁜 예

```html
<button onclick="doSend()">Send</button>
<script nonce="...">
  function doSend(){ /* ... */ }
</script>
```
- **문제**: `script-src-attr 'none'`에서 **차단**.

### 좋은 예

```html
<button id="send">Send</button>
<script nonce="...">
  document.getElementById('send').addEventListener('click', () => {
    // 핸들러 로직
  });
</script>
```

> **참고**: 정말 불가피할 때만 `'unsafe-hashes'`/`'unsafe-hashed-attributes'`로 특정 인라인 핸들러 해시를 허용할 수 있으나, **권장하지 않음**.

---

## 구형 브라우저/레거시 호환 전략

- **원칙**: Nonce를 인식하지 못하는 환경을 위해
  1) **부트스트랩을 외부 파일**로 분리(호스트 화이트리스트와 병행), 또는
  2) **해시 기반(`'sha256-...'`)**을 병행해 **핵심 인라인 부트스트랩**만 허용.
- 예(해시 병행):
```http
Content-Security-Policy:
  script-src 'nonce-<...>' 'strict-dynamic' 'sha256-<INLINE_BOOT_HASH>';
```
- 단, **`'unsafe-inline'`은 절대 추가하지 않기**(보안 약화).

---

## CDN/캐싱/엣지 고려

- **HTML 캐싱 주의**: Nonce가 **응답별로 달라야** 하므로, **캐시 키**에 세션/쿠키/변수 반영 또는 **엣지에서 Nonce 주입**.
- **정적 자산**(JS/CSS)은 **캐시 가능**. Nonce는 **HTML에만**(부트스트랩 또는 외부 로더에 필요 시).

---

## & 리포팅

### 단계적 적용

1) **Report-Only**로 시작:
   `Content-Security-Policy-Report-Only: script-src 'nonce-…' 'strict-dynamic'; report-to csp-endpoint`
   → 실제 차단 대신 **위반 리포트**만 수집.
2) 위반 패턴을 정리/수정 후 **Enforce**로 전환.

### Reporting API(백엔드)

```http
Report-To: {"group":"csp-endpoint","max_age":10886400,"endpoints":[{"url":"https://report.example.com/csp"}]}
Content-Security-Policy: ...; report-to csp-endpoint
```

간단 수집기(Express):
```js
app.post('/csp-report', express.json({ type: ['json', 'application/csp-report'] }), (req, res) => {
  console.log('CSP report', JSON.stringify(req.body));
  res.sendStatus(204);
});
```

---

## 흔한 실수(안티패턴)

- **Nonce 재사용**(캐시/템플릿 버그) → **주입 재이용** 가능성↑
- **`'unsafe-inline'` 추가** → Nonce의 의미 퇴색
- **인라인 이벤트(… onClick=)** 지속 사용 → `script-src-attr 'none'`에서 깨짐
- **정적 HTML로 Nonce 하드코딩** → 의미 없음
- **Nginx만으로 Nonce 만들기** 시도 → 각 `<script>`에 같은 Nonce를 일관 주입하기 어려움(앱 레이어 권장)
- **동적 로드 남발**(광고/3rd-party 위젯 삽입) → **정말 필요한 로드만** 허용되도록 **부트스트랩 로직**을 엄격히 관리

---

## 테스트 시나리오(“막혀야 정상”)

1) **Nonce 없는 인라인 `<script>`**
   - 기대: 차단, 콘솔에 CSP 위반 기록.
2) **인라인 이벤트 핸들러**
   - 기대: 차단(`script-src-attr 'none'`).
3) **부트스트랩 → 동적 로드 체인**
   - 기대: 허용(부트스트랩 Nonce + `'strict-dynamic'`).
4) **Report-Only에서 위반 리포트 수집**
   - 기대: `/csp-report`에 위반 JSON 도착.

---

## 보조 강화 옵션

- **`require-trusted-types-for 'script'` + Trusted Types**: DOM XSS 표면 감소(크롬 계열).
- **`object-src 'none'` / `base-uri 'self'` / `frame-ancestors 'self'`**: 레거시 플러그인/프레이밍 차단.
- **SRI**(Subresource Integrity)와 병행: 외부 리소스 무결성 보강(단, `'strict-dynamic'` 체인에서는 Nonce 신뢰가 우선).

---

## 미니 체크리스트

- [ ] **요청마다 Nonce** 생성(128비트+, CSPRNG, Base64)
- [ ] `script-src 'nonce-…' 'strict-dynamic'` + `script-src-attr 'none'`
- [ ] **인라인 이벤트 → addEventListener** 마이그레이션
- [ ] **Report-Only**로 린치핀 파악 → Enforce 전환
- [ ] **HTML 캐싱에서 Nonce 재사용 방지**(엣지 주입/프라이빗 캐시)
- [ ] 3rd-party 스크립트는 **부트스트랩 로더**가 명시적으로 삽입
- [ ] 위반 리포트 수집/대시보드화

---

## FAQ

- **Q. 모든 동적 로드가 자동 허용되나요?**
  A. **“신뢰된(Nonce/해시로 허용된) 스크립트가 추가한 것”**만 허용됩니다. 임의로 DOM에 삽입된 `<script>`는 Nonce가 없으므로 차단됩니다.

- **Q. 호스트 화이트리스트가 필요 없나요?**
  A. `strict-dynamic`가 있으면 **동적 체인에 대해** 호스트 화이트리스트보다 **Nonce 신뢰가 우선**합니다. 다만 **구형 브라우저** 대응/심리적 안전망으로 `https:`/특정 호스트를 병행 명시하는 경우가 있습니다.

- **Q. 인라인 스크립트를 계속 쓰고 싶다면?**
  A. **Nonce 또는 해시**가 필요합니다. 인라인 이벤트 핸들러는 지양하고, 불가피하면 `'unsafe-hashes'`로 **특정 해시만** 허용할 수 있으나 권장하지 않습니다.

---

## 요약 템플릿(복붙)

### 서버(Express)

```js
const nonce = crypto.randomBytes(16).toString('base64');
res.set('Content-Security-Policy',
  `base-uri 'self'; object-src 'none'; ` +
  `script-src 'nonce-${nonce}' 'strict-dynamic' 'report-sample'; ` +
  `script-src-elem 'nonce-${nonce}' 'strict-dynamic' 'report-sample'; ` +
  `script-src-attr 'none'; connect-src 'self' https:; img-src 'self' data: https:; ` +
  `style-src 'self' 'unsafe-inline'; font-src 'self' https: data:; ` +
  `frame-ancestors 'self'; upgrade-insecure-requests; report-to csp-endpoint`
);
```

### HTML

{% raw %}
```html
<script nonce="{{nonce}}">
  // 부트스트랩 → 동적 로드
  const s = document.createElement('script');
  s.src = 'https://cdn.example.com/vendor.js';
  document.head.appendChild(s);
</script>
```
{% endraw %}

> 이 구성을 기반으로, **스타일/폰트/연결 정책**은 서비스 요구에 맞게 더욱 제한적으로 다듬으세요.

---

### 맺음말

**`'strict-dynamic'` + Nonce 회전**은 “**우리가 신뢰한 부트스트랩**만 실행하고, **그 스크립트가 책임지고 로드한 체인만 허용**”한다는 명확한 모델입니다.
운영 핵심은 **Nonce 재사용 금지**, **인라인 이벤트 제거**, **캐싱/엣지 주입 설계**, 그리고 **Report-Only → Enforce**의 점진 롤아웃입니다.
