---
layout: post
title: 웹해킹 - HTTP Parameter Pollution
date: 2025-09-30 23:25:23 +0900
category: 웹해킹
---
# 7. HTTP Parameter Pollution (HPP)

## 0. 한눈에 보기 (Executive Summary)

- **문제(정의)**: **같은 이름의 파라미터가 다중 전달**될 때(예: `role=user&role=admin`)
  프록시/프레임워크/앱/캐시 **각 계층의 파서가 서로 다르게 해석**(첫 값/마지막 값/배열)하면
  **검증·권한 체크 우회**, **캐시 키 불일치**, **로직 분기 교란**이 발생합니다. → 이것이 **HPP**.
- **대표 징후**
  - 프록시/앱/백엔드에서 **동일 요청에 대한 파라미터 값이 다르게 보임**
  - 간헐적 **권한 상승**(예: `role`), **정책 우회**(예: `safe=true&safe=false`)
  - 특정 URL의 **캐시 응답 불일치**(키와 백엔드 해석 차이)
- **핵심 방어**
  1) **중복 파라미터 금지**(Whitelist 기반의 **정규화**)
  2) **한 정책으로 고정**: *“첫 값만 허용”* 또는 *“마지막 값만 허용”* 중 **하나로 통일**
  3) **배열은 명시적으로만 허용**(예: `ids[]=1&ids[]=2` 같은 **명시적 표기**만)
  4) **게이트웨이/프록시에서 선차단**, **앱에서 2차 검증**, **로그/탐지**로 재발 방지

---

# 1. 왜 HPP가 생기나 — 파서의 “해석 정책” 불일치

같은 쿼리 스트링이라도 **파서마다 정책이 다릅니다.**
아래는 **일반적인 예**(구현/버전에 따라 달라질 수 있음):

- 어떤 파서는 **첫 값 우선**: `a=1&a=2` → `a="1"`
- 어떤 파서는 **마지막 값 우선**: `a=1&a=2` → `a="2"`
- 어떤 파서는 **배열로 수집**: `a=1&a=2` → `a=["1","2"]`
- `a[]=1&a[]=2`처럼 **배열 표기**가 있을 때만 배열로 인식하는 파서도 많음.
- `;`(세미콜론)을 **분리자로 인정**하는 구성도 있음(서버/프록시에 따라 옵션).

> **핵심 위험**: **프론트/프록시/앱**의 **정책이 다르면 검증을 우회**할 수 있습니다.
> (예: 프론트는 첫 값으로 검증, 백엔드는 마지막 값으로 권한 결정)

---

# 2. 영향 시나리오 (개념 예시)

### 2.1 권한 상승 (role 파라미터)
- 프론트엔드: `role`이 하나만 있어야 한다고 **첫 값**만 검사 → `role=user` → 통과
- 백엔드: 라우터 파서는 **마지막 값** 사용 → 실제 처리 시 `role=admin`
- 요청: `POST /invite?role=user&role=admin` → **검증 통과 + 권한 상승**

### 2.2 정책 우회 (feature flag)
- 필터: `safe=true`만 허용
- 요청: `?safe=true&safe=false`
- **어느 계층이 무엇을 읽느냐**에 따라 **안전 기능 비활성화** 가능

### 2.3 캐시 키 vs 백엔드 해석 차이
- CDN 캐시 키는 **첫 값 기준**으로 계산, 백엔드는 **마지막 값**으로 처리
- 서로 다른 콘텐츠가 **같은 캐시 엔트리**에 저장되거나, 반대로 **같은 요청이 다른 키**로 분리

---

# 3. 안전한 재현(스테이징 전용) — “막혀야 정상” 스모크

> 목적: **우리가 중복 파라미터를 금지/정규화**하는지 자동 점검.

### 3.1 `curl` 간단 테스트
```bash
# 1. 중복 role → 400/422 등으로 거절되어야 안전
curl -si 'https://staging.example.com/admin?role=user&role=admin' | head -n1

# 2. 세미콜론 분리자도 차단(구성에 따라)
curl -si 'https://staging.example.com/admin?role=user;role=admin' | head -n1
```

### 3.2 Node 스크립트(스테이징 전용)
```js
import https from "node:https";
function rq(path) {
  return new Promise((resolve) => {
    const req = https.request({ host:"staging.example.com", path, method:"GET", rejectUnauthorized:false }, res => {
      resolve({ code: res.statusCode, hdr: res.headers });
    });
    req.end();
  });
}
const cases = [
  "/admin?role=user&role=admin",
  "/admin?role=user;role=admin",  // 세미콜론 분리자
  "/search?ids=1&ids=2",          // 배열 비허용 엔드포인트면 거절
];
for (const c of cases) { /* eslint-disable no-await-in-loop */
  const r = await rq(c);
  console.log(c, "→", r.code);
}
```

> **기대**: 보안 정책상 **허용되지 않은 중복 키**는 **항상 400/422**.

---

# 4. 백엔드(애플리케이션) 레이어 방어

## 4.1 정책 결정: **첫 값/마지막 값/배열** 중 **하나**로 고정
- **권장**:
  - **기본은 “중복 금지”**.
  - **배열이 필요한 엔드포인트만** 명시적으로 허용(예: `ids[]=1&ids[]=2`).
  - 불가피하다면 “첫 값만” 또는 “마지막 값만”을 **전 계층에서 동일**하게 채택.

## 4.2 Node.js(Express) — **중복 금지 + 배열 화이트리스트**
```js
// security/hpp.js
const MULTI_ALLOWED = new Set(["ids[]", "tags[]"]);  // 배열 허용 키(명시적 표기)
const ALSO_ALLOW_PLAIN = new Set(["ids","tags"]);    // (선택) 레거시 호환

export function hppGuard(req, res, next) {
  // 1) 쿼리 스트링 중복 탐지
  const url = new URL(req.originalUrl, "http://dummy");
  const sp = url.searchParams; // URLSearchParams
  // 모든 키에 대해 getAll로 다중 값 탐지
  for (const key of new Set([...sp.keys()])) {
    const values = sp.getAll(key);
    const isMulti = values.length > 1;
    const allowed = MULTI_ALLOWED.has(key) || ALSO_ALLOW_PLAIN.has(key);
    if (isMulti && !allowed) {
      return res.status(400).json({ error: "duplicate_param", key });
    }
  }

  // 2) 바디 파라미터(폼) 중복 탐지
  //  - JSON은 중복 키가 표준상 불가(파서가 마지막 값으로 덮을 수 있으니 바람직하지 않음)
  //  - x-www-form-urlencoded는 배열로 들어올 수 있음 → 값이 배열인데 허용 리스트에 없으면 차단
  if (req.is("application/x-www-form-urlencoded")) {
    for (const [k, v] of Object.entries(req.body || {})) {
      const isArray = Array.isArray(v);
      const allowed = MULTI_ALLOWED.has(k) || ALSO_ALLOW_PLAIN.has(k);
      if (isArray && !allowed) {
        return res.status(400).json({ error: "duplicate_form_param", key: k });
      }
    }
  }

  // 3) 정규화 정책(선택): 중복 허용되지 않는 키는 첫 값만 남기기
  //    (앱 전역 동일 정책 — 프록시/게이트웨이까지 일치시켜야 함)
  for (const key of new Set([...sp.keys()])) {
    if (MULTI_ALLOWED.has(key) || ALSO_ALLOW_PLAIN.has(key)) continue;
    const all = sp.getAll(key);
    if (all.length > 1) {
      sp.delete(key);
      sp.append(key, all[0]); // '첫 값 우선'로 통일
    }
  }

  return next();
}
```

> **주의**
> - 프레임워크의 기본 파서가 중복을 **배열**로 주기도 하고, **마지막 값**으로 덮기도 합니다.
> - 위처럼 **명시적 허용 목록** 외엔 **중복 거절** + (필요 시) **정규화**를 수행하세요.

### 4.2.1 Express에 적용
```js
import express from "express";
import bodyParser from "body-parser";
import { hppGuard } from "./security/hpp.js";

const app = express();
app.use(bodyParser.json({ limit: "100kb" }));
app.use(bodyParser.urlencoded({ extended: false }));   // 간단 파서(예측 가능)
app.use(hppGuard);

// 예: 관리자 엔드포인트 — role은 단일만 허용
app.get("/admin", (req, res) => {
  const role = req.query.role;  // hppGuard로 중복 금지 보장
  if (role !== "admin") return res.status(403).send("forbidden");
  res.send("ok");
});

// 예: 배열 허용 엔드포인트(명시적)
app.get("/search", (req, res) => {
  const ids = []
    .concat(req.query["ids[]"] || [])
    .concat(req.query["ids"] || []);
  if (!ids.length) return res.json({ rows: [] });
  // ids 배열 사용 …
  res.json({ ids });
});
```

### 4.2.2 **금지**: 혼합 getter
- Express의 과거 API `req.param(name)` 처럼 **route/query/body 혼합 조회**는 **정책 충돌**을 일으킵니다(삭제/미사용 권장).
- **항상 명시적**으로 `req.query`, `req.body`, `req.params`를 **구분**하세요.

---

## 4.3 Python — Flask

```python
from flask import Flask, request, abort

app = Flask(__name__)
ALLOWED_MULTI = {"ids[]", "tags[]"}  # 명시적 배열 허용

@app.before_request
def hpp_guard():
    # 1) 쿼리스트링 검사
    # request.args.getlist(key) → 모든 값을 리스트로 반환
    for key in request.args.keys():
        vals = request.args.getlist(key)
        if len(vals) > 1 and key not in ALLOWED_MULTI:
            abort(400, description=f"duplicate param: {key}")

    # 2) 폼 데이터 검사
    if request.content_type and "application/x-www-form-urlencoded" in request.content_type:
        for key in request.form.keys():
            vals = request.form.getlist(key)
            if len(vals) > 1 and key not in ALLOWED_MULTI:
                abort(400, description=f"duplicate form param: {key}")
```

---

## 4.4 Django (배열/중복 명시적 처리)

```python
from django.http import HttpResponseBadRequest

ALLOWED_MULTI = {"ids[]","tags[]"}

def hpp_guard(get_response):
    def middleware(request):
        for k in request.GET.keys():
            vals = request.GET.getlist(k)
            if len(vals) > 1 and k not in ALLOWED_MULTI:
                return HttpResponseBadRequest(f"duplicate param: {k}")
        if request.method in ("POST","PUT","PATCH") and request.content_type == "application/x-www-form-urlencoded":
            for k in request.POST.keys():
                vals = request.POST.getlist(k)
                if len(vals) > 1 and k not in ALLOWED_MULTI:
                    return HttpResponseBadRequest(f"duplicate form param: {k}")
        return get_response(request)
    return middleware
```

---

## 4.5 Java — Spring Boot(Filter)

```java
@Component
public class HppFilter implements Filter {
  private static final Set<String> ALLOWED_MULTI = Set.of("ids[]","tags[]");

  @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
      throws IOException, ServletException {
    HttpServletRequest r = (HttpServletRequest) req;
    var names = r.getParameterNames();
    while (names.hasMoreElements()) {
      String name = names.nextElement();
      String[] vals = r.getParameterValues(name);
      if (vals != null && vals.length > 1 && !ALLOWED_MULTI.contains(name)) {
        ((HttpServletResponse) res).sendError(400, "duplicate param: " + name);
        return;
      }
    }
    chain.doFilter(req, res);
  }
}
```

> 스프링에서는 `@RequestParam List<String>` 로 **일부 엔드포인트에만** 중복을 허용(배열)하고, **나머지는 기본적으로 중복 거절**이 이상적입니다.

---

## 4.6 Go — net/http

```go
func HPPGuard(next http.Handler) http.Handler {
  allowed := map[string]bool{"ids[]": true, "tags[]": true}
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    q := r.URL.Query()
    for k, vals := range q {
      if len(vals) > 1 && !allowed[k] {
        http.Error(w, "duplicate param: "+k, http.StatusBadRequest)
        return
      }
    }
    next.ServeHTTP(w, r)
  })
}
```

---

## 4.7 ASP.NET Core

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

var allowed = new HashSet<string>(StringComparer.OrdinalIgnoreCase){ "ids[]", "tags[]" };

app.Use(async (ctx, next) => {
  foreach (var kv in ctx.Request.Query) {
    if (kv.Value.Count > 1 && !allowed.Contains(kv.Key)) {
      ctx.Response.StatusCode = 400;
      await ctx.Response.WriteAsync("duplicate param: " + kv.Key);
      return;
    }
  }
  await next();
});
```

---

# 5. 게이트웨이/프록시 레벨 선차단

## 5.1 Nginx (정규식 기반 간단 차단 — 민감 키 예: role)
```nginx
# role 키가 두 번 이상 등장하면 거절
set $dup_role 0;
if ($query_string ~* "(^|[;&])role=[^;&]*([;&].*role=)") { set $dup_role 1; }
if ($dup_role) { return 400; }

# 세미콜론(;)을 분리자로 쓰지 않도록 전역적으로 비허용(정책에 따라)
if ($query_string ~* ";") { return 400; }
```

> 정규식만으로 **모든 키의 중복**을 일반화하기는 어렵습니다.
> **권장**: OpenResty(ngx_lua)나 게이트웨이(예: Kong, APIM)에서 **파싱 후 검사**.

### 5.1.1 OpenResty(Lua) — 정확한 파싱 후 차단
```nginx
location / {
  access_by_lua_block {
    local allowed = { ["ids[]"]=true, ["tags[]"]=true }
    local args = ngx.req.get_uri_args(1000)  -- 1000개까지 파싱
    for k, v in pairs(args) do
      if type(v) == "table" and not allowed[k] then
        return ngx.exit(400)
      end
    end
  }
  proxy_pass http://app;
}
```

## 5.2 HAProxy (민감 키별 ACL)
```haproxy
frontend fe
  bind :443 ssl crt /etc/haproxy/cert.pem
  # role 키 중복 탐지(간단 정규식)
  http-request deny if { query -m reg "(^|[&;])role=[^&;]*[&;].*role=" }
  default_backend be
```

## 5.3 클라우드/API 게이트웨이
- **AWS API Gateway**: `multiValueQueryStringParameters`를 **전달하지 않게** 매핑하거나, Lambda/Integration에서 **중복 검사** 후 거절.
- **Kong/Apigee**: Request Transformer/Plugin으로 **중복 제거/차단** 정책 구현.
- **Cloudflare/CloudFront**: Function/Worker로 URI args 파싱 → **중복 거절**.

---

# 6. 프런트엔드/클라이언트 수칙

- **URL 생성 시** `URLSearchParams` 사용 + **기존 키 존재 여부 검사**
- **배열은 명시적 표기**(`ids[]`) 또는 JSON 바디를 사용(REST: GET 대신 POST/검색 바디 등)
- **쿼리 표준화**: 키 **정렬** + **중복 제거**(사인/캐시 키 일관성 향상)

**예 — 브라우저**
```js
function buildQuery(params) {
  const usp = new URLSearchParams();
  for (const [k,v] of Object.entries(params)) {
    if (Array.isArray(v)) {
      for (const x of v) usp.append(`${k}[]`, String(x));
    } else {
      // 이미 존재하면 덮지 않음(또는 에러 throw) → 중복 방지
      if (usp.has(k)) throw new Error("duplicate param: "+k);
      usp.set(k, String(v));
    }
  }
  return usp.toString();
}
```

---

# 7. “세미콜론(;) 분리자”, “이중 디코딩” 이슈

- 일부 서버/프록시는 `;`를 **파라미터 분리자**로 해석할 수 있습니다. 정책적으로 **금지**하거나, **정규화 시 모두 `&`로 통일**하세요.
- **이중 디코딩(Double-decoding)**으로 `a=1%26a%3D2` 같은 값이 **나중 단계에서 다시 분리**되지 않도록, **한 번만 디코딩**하고 **값으로 취급**해야 합니다.

---

# 8. 테스트/CI — 자동화로 재발 방지

## 8.1 단위 테스트(Express 예)
```js
import request from "supertest";
import app from "../app.js"; // hppGuard 적용된 서버

test("중복 role 거절", async () => {
  const r = await request(app).get("/admin?role=user&role=admin");
  expect([400,422]).toContain(r.status);
});

test("ids[] 배열만 허용", async () => {
  const ok = await request(app).get("/search?ids[]=1&ids[]=2");
  expect(ok.status).toBe(200);
  const bad = await request(app).get("/search?ids=1&ids=2");
  expect([400,422]).toContain(bad.status);
});
```

## 8.2 파이프라인 가드(간단 grep 룰)
```bash
# 위험 API 사용(혼합 getter) 탐지
grep -R --line-number -E "req\.param\(" src/ && \
  echo "::error title=HPP risk::Do not use req.param()" && exit 1
```

---

# 9. 로깅/모니터링/탐지

- **로그 필드**: `ts`, `route`, `ip`, `user_id`, `dup_keys`(배열), `policy`(first/last/none), `status`
- **탐지 룰 예시**
  - **중복 거절 카운트**(5분 동안 N회 이상) 알림 → 악성 스캐닝 의심
  - **세미콜론 포함 쿼리** 빈도 증가 알림
  - **민감 키(role, admin, safe, debug, redirect)** 중복 시도 모니터링

**Loki(LogQL)**
```logql
{event="reject_hpp"} | json | count_over_time(5m) > 20
```

**Splunk(SPL)**
```spl
index=edge event=reject_hpp
| stats count by key, src_ip
| where count > 5
```

---

# 10. 운영 체크리스트 (요약)

- [ ] **전 계층 동일 정책**: *중복 금지* + 필요 시 *첫 값만 허용*으로 **통일**
- [ ] **배열 허용은 명시적**(`[]` 표기 or JSON 바디)
- [ ] **게이트웨이/프록시**: 중복 탐지/차단, 세미콜론 금지, 이중 디코딩 방지
- [ ] **앱 미들웨어**: 중복 거절 + 정규화, 혼합 getter 금지
- [ ] **민감 키**(role, safe, admin, redirect, debug 등) **특별 차단 룰**
- [ ] **테스트/CI**: 중복/세미콜론/대량 파라미터 스모크
- [ ] **로그/탐지**: 거절 카운트, 키/경로별 통계, 이상 징후 알림
- [ ] **문서화**: “배열은 `[]` 표기만 허용” 같은 **클라이언트 가이드** 배포

---

## 맺음말

HPP는 “**각 계층이 파라미터를 다르게 해석**한다”는 아주 작은 틈에서 시작해,
**권한 상승·검증 우회·캐시 오류** 같은 큰 사고로 번질 수 있습니다.
**중복 금지/정규화**, **배열의 명시적 허용**, **전 계층 정책 통일**만으로도
대부분의 HPP를 **구조적으로 제거**할 수 있습니다.
