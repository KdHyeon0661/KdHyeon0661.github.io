---
layout: post
title: 웹해킹 - HTTP Parameter Pollution
date: 2025-09-30 23:25:23 +0900
category: 웹해킹
---
# HTTP Parameter Pollution (HPP)

## 1. 한눈에 보기 (Executive Summary)

### 1.1 문제(정의)

- **HTTP Parameter Pollution (HPP)**는 **같은 이름의 파라미터가 여러 번 전달**될 때
  - 프록시 / CDN / WAF / API 게이트웨이 / 웹 프레임워크 / 애플리케이션이
  - 각자 **다른 규칙**(첫 값 우선, 마지막 값 우선, 배열, 무시 등)으로 처리하면서
- 그 틈을 이용해 공격자가
  - **검증·권한 체크 우회**
  - **캐시 키 불일치 → 캐시 오염**
  - **WAF 규칙 우회**
  - **내부 API 호출 파라미터 오염(Server-side Parameter Pollution)**
  를 일으키는 취약점이다.

예:  

```text
/admin?role=user&role=admin
```

- 프런트 검증: `role=user`만 본다고 가정
- 백엔드: 마지막 값만 보는 파서 → `role=admin`
- 결과: **검증은 user 기준**으로 통과했지만, **실제 권한은 admin**으로 처리.

### 1.2 대표 징후

- 동일 요청을
  - **프록시/게이트웨이 앞단 로그**와
  - **애플리케이션 로그**
  에서 비교하면 **파라미터 값이 다르게 보이는** 현상
- `?safe=true&safe=false` 같은 요청 이후
  - 간헐적인 **권한 상승**, **정책 우회**
- CDN/캐시에서
  - 특정 URL의 **응답 내용이 가끔씩 달라짐**
  - 캐시 키와 백엔드 해석 정책이 다를 때 발생
- 보안 스캐너 / WAF 로그에
  - **중복 파라미터, 세미콜론(;) 기반 쿼리, 이중 디코딩 흔적**이 반복적으로 찍힘

### 1.3 핵심 방어 전략 요약

1. **중복 파라미터 원칙 정하기**
   - 기본 정책: **“중복 자체 금지”**
   - 정말 필요하면, **전 계층(프록시–게이트웨이–앱)**에서
     - “**첫 값만 허용**” 또는 “**마지막 값만 허용**” 중 하나로 **완전히 통일**
2. **배열은 명시적으로만 허용**
   - `ids[]=1&ids[]=2`처럼 **배열임을 드러내는 표기**만 허용
   - `ids=1&ids=2`처럼 애매한 중복은 **금지** 또는 **400/422**
3. **게이트웨이/프록시 레벨에서 선차단**
   - Nginx/Envoy/HAProxy/API Gateway에서
     - **중복 키 탐지 → 400**
     - **세미콜론(;) 분리자 금지**
     - **이중 디코딩·이상 인코딩 차단**
4. **애플리케이션 레벨에서 2차 검증**
   - Express / Flask / Django / Spring / Go / ASP.NET Core 등에서
     - **중복 키 / 배열 허용 키 화이트리스트** 미들웨어로 강제
5. **테스트·CI·로깅까지 포함한 운영 프로세스**
   - 스테이징 환경에서 **“막혀야 정상인 케이스”**를 자동화 테스트에 포함
   - 중복 파라미터 거절 이벤트를 **로그·모니터링**으로 집계해 스캐닝·공격을 탐지

---

## 2. 배경과 위협 환경 (2024–2025 관점)

### 2.1 HPP의 등장과 현재 의미

- HPP는 2009년경 공개 컨퍼런스에서 처음 체계적으로 소개된 이후
  - **“파라미터라는 가장 기본 요소”**를 노리는 취약점으로 인식되기 시작했다.
- 전통적인 SQL Injection, XSS에 비해 **덜 주목받지만**
  - 실제로는 **입력 검증·권한 시스템·캐시 설계**의 빈틈을 찌르기 때문에
  - **다른 취약점과 결합**될 때 파급력이 크다.
- 최근의 국제 보고서들에서도
  - **웹 애플리케이션 공격·입력 검증 문제·API 설계 미비**가
  - 여전히 주요 공격 벡터라는 점을 반복해서 강조하고 있고,
  - HPP는 이러한 “입력·파라미터 계층 문제”의 대표적인 사례 중 하나다.

### 2.2 표준의 애매함이 만든 틈

- HTTP / URI 관련 RFC는
  - **같은 이름의 파라미터가 여러 번 올 때 어떻게 해석해야 하는지**를
  - **명시적으로 정의하지 않는다.**
- 그 결과, 각 계층·각 제품은 저마다의 정책을 갖게 되었고:
  - “첫 값만 사용”
  - “마지막 값만 사용”
  - “배열로 모두 수집”
  - “첫 값 + 나머지는 무시”
  - “에러 처리”
- 이런 **“표준 부재 + 구현 다양성”** 자체가 HPP의 근본 원인이다.

---

## 3. 왜 HPP가 생기나 — 파서의 “해석 정책” 불일치

### 3.1 파라미터가 지나가는 경로

HTTP 요청 하나의 파라미터는 보통 다음과 같은 단계를 거친다.

```text
[브라우저 / 모바일 앱]
   ↓ (URL·폼 인코딩)
[CDN / WAF / 리버스 프록시 / L7 LB]
   ↓ (쿼리 파싱 / 재조합 / 라우팅)
[API 게이트웨이 / 마이크로서비스 게이트웨이]
   ↓ (매핑 / 변환 / 리트라이)
[프레임워크 라우터 / 파서]
   ↓
[비즈니스 로직 / ORM / DB / 캐시 / 외부 API]
```

각 단계마다

- 쿼리스트링/바디를 **다시 파싱**하거나
- 일부 파라미터를 **삭제·추가·변환**할 수 있다.

어느 한 계층이라도 **“중복 허용 + 애매한 정책”**을 쓰면, 다른 계층과 조합될 때 HPP의 공격면을 만든다.

### 3.2 전형적인 파싱 전략 4패턴

엄밀한 표는 프레임워크마다 달라지므로, 여기서는 **추상적인 정책 유형**만 정리한다.

| 정책 유형       | 예시 동작                             | 장점                                   | 단점 / HPP 관점 위험                     |
|----------------|----------------------------------------|----------------------------------------|-------------------------------------------|
| **첫 값 우선** | `a=1&a=2` → `"1"`                      | 단순·예측 가능                         | 프록시·게이트웨이가 마지막 값 기준이면 충돌 |
| **마지막 우선**| `a=1&a=2` → `"2"`                      | 최근 값 우선(덮어쓰기)                 | 검증은 첫 값, 권한은 마지막 값으로 갈라질 수 있음 |
| **배열 수집**  | `a=1&a=2` → `["1","2"]`               | 모든 값 관측 가능                      | 로직이 “단일 값” 전제로 작성되면 혼선     |
| **에러 처리**  | `a=1&a=2` → 400/422                    | 보안상 가장 안전                       | 기존 클라이언트/레거시 호환성 이슈       |

문제는 **같은 요청에 대해**:

- CDN: “배열 수집”으로 캐시 키 생성
- 앱: “마지막 우선”으로 권한 로직 수행

처럼 **정책이 어긋날 때** 발생한다.

---

## 4. 공격 표면과 분류

HPP는 크게 다음 네 가지 관점으로 나눠서 보는 것이 실무에 도움이 된다.

### 4.1 서버사이드 HPP (Server-Side Parameter Pollution, SSPP)

- 외부로부터 들어온 파라미터가
  1. 서버 내부에서 다시 URL을 만들거나
  2. 내부 API / 마이크로서비스 / SaaS에 **서버사이드 요청**을 보낼 때
- 적절히 인코딩·검증되지 않고 그대로 들어가면서
  - 내부 요청 내 **기존 파라미터를 덮어쓰거나**
  - **추가 파라미터를 주입**하게 되는 유형.

예:

```js
// 취약 예시 — 서버가 내부 API에 쿼리를 중계
app.get("/user", async (req, res) => {
  const q = req.query.q || "";
  const url = "https://internal.api/search?q=" + q; // 그대로 이어 붙임
  const r = await fetch(url);
  const data = await r.json();
  res.json(data);
});
```

요청:

```text
/user?q=kim&role=admin
```

내부 요청:

```text
https://internal.api/search?q=kim&role=admin
```

만약 내부 API가 `role` 파라미터를 사용한다면, **외부 요청이 내부 권한 파라미터를 오염**시킬 수 있다.

### 4.2 클라이언트사이드 HPP

- 클라이언트(브라우저)에서 실행되는 JavaScript가
  - `location.search`를 적절한 검증 없이 **다른 URL에 덧붙이는** 경우,

예:

```js
// 취약 예시 — 현재 쿼리를 그대로 다른 엔드포인트에 붙임
const url = "/api/search" + window.location.search;
fetch(url);
```

현재 URL이 `/page?term=phone&role=admin`이면,

- 브라우저는 `/api/search?term=phone&role=admin`으로 요청
- `/api/search`가 원래 `role`을 받지 말아야 하는 엔드포인트라면
  - **클라이언트 측에서 이미 파라미터 오염이 일어난 상황**이 된다.

이 패턴은 **클라이언트사이드 HPP**로 분류되며, CSP·프론트엔드 코드 리뷰로 방어해야 한다.

### 4.3 API·마이크로서비스·게이트웨이 환경의 HPP

- 최근 시스템은
  - 외부 → API Gateway → BFF(Backend for Frontend) → Microservice → DB
  - 이런 구조를 많이 쓴다.
- 각 단계에서
  - 쿼리/바디를 다시 파싱하고
  - 다른 서비스로 재전달한다.
- 이 과정에서:
  - 게이트웨이: 배열로 수집
  - BFF: 마지막 값 기준
  - 내부 서비스: 첫 값 기준
- 처럼 정책이 뒤섞일 때, **검증–권한–로깅–캐시**가 서로 다른 값을 보게 된다.

### 4.4 WAF·보안 우회

- 일부 WAF는 특정 파라미터에 대해
  - “첫 값만 검사”
  - “특정 길이 이상만 검사”
  같은 규칙을 도입한다.
- 공격자는 페이로드를
  - WAF가 보지 않는 위치의 값으로 쪼개서 넣고
  - 실제로 사용되는 값만 악의적인 내용으로 설정하는 방식으로
- **정책 우회 + 공격 페이로드 주입**을 동시에 노릴 수 있다.

---

## 5. 영향 시나리오 — 구체 예제들

### 5.1 권한 상승 — `role=user&role=admin`

#### (1) 시나리오

1. 프런트엔드/게이트웨이 검증

   - `role`이 하나만 있어야 한다고 가정
   - 첫 값만 읽어서 검증

2. 백엔드 비즈니스 로직

   - 프레임워크의 `getParameter("role")`가
   - 마지막 값을 반환하는 구현

3. 요청:

   ```text
   /admin/invite?role=user&role=admin
   ```

4. 결과:

   - 검증: `role=user` → 통과
   - 처리: `role=admin` → 관리자 권한으로 실행

#### (2) 실제 코드 예시 (취약 Java Spring 컨트롤러)

```java
@GetMapping("/admin/invite")
public String invite(@RequestParam String role, Model model) {
    // 여기서 role은 마지막 값 기준일 수 있음
    if (!"admin".equals(role)) {
        return "error/403";
    }
    // 관리자 로직...
    return "admin/invite";
}
```

파라미터가 중복될 때의 동작이 **문서화·통제되지 않으면**, 이런 우연한 정책이 HPP 공격면이 된다.

### 5.2 안전 플래그 우회 — `safe=true&safe=false`

```text
/download?file=...&safe=true&safe=false
```

- 앞단 필터: `safe=true`가 있는지만 확인 → 통과
- 실제 다운로드 로직: 마지막 값 기준으로 `safe=false` → 위험한 기능 활성화

### 5.3 캐시 키 vs 백엔드 해석 차이

- CDN 캐시 키:
  - 쿼리 파라미터를
    - key → 첫 값
    - 형태로 정규화해 키 생성
- 백엔드:
  - 마지막 값 기준으로 처리

요청:

```text
/article?id=10&id=11
```

- CDN 키: `id=10`
- 백엔드 처리: `id=11`

공격자는 HPP를 이용해,

1. 먼저 `/article?id=10&id=11`으로 **오염된 응답을 캐시에 넣고**
2. 일반 사용자가 `/article?id=10`으로 접근하면,
3. **캐시는 10에 대한 오염 응답**을 제공한다.

### 5.4 리다이렉트 / OAuth / `redirect_uri` 조작

```text
/login?redirect=/profile&redirect=https://evil.example
```

- 안전 검증 로직: 첫 번째 `redirect`만 검사 → `/profile`만 허용
- 실제 리다이렉트: 마지막 값 기준 → `https://evil.example`

HPP + Open Redirect가 결합해 **피싱·세션 탈취** 등으로 이어질 수 있다.

### 5.5 로깅·탐지 회피

- WAF/IDS/로그 시스템은
  - 첫 값만 기록
- 애플리케이션은
  - 마지막 값 사용

이 경우:

```text
/search?q=normal&q=' OR 1=1 --
```

- 로그: `q=normal`
- 실제 DB 쿼리: `" ' OR 1=1 -- "`
- 보안팀이 로그만 봐서는 **공격을 파악하기 어렵다.**

---

## 6. “막혀야 정상” 스모크 테스트

> 이 섹션은 **내부 스테이징·보안 연구 환경에서** HPP 방어가 제대로 동작하는지 확인하기 위한 테스트 아이디어다.  
> 실서비스·타인의 시스템에 임의로 보내면 안 된다.

### 6.1 단순 `curl` 테스트

```bash
# [케이스1] 중복 role — 관리자 엔드포인트
# 기대: 400 / 422 / 403 등 "정상 흐름"이 아님

curl -si 'https://staging.example.com/admin?role=user&role=admin' | head -n1

# [케이스2] 세미콜론(;) 분리자
# 세미콜론을 분리자로 사용하지 않도록 정책을 잡았으면, 역시 거절이 바람직

curl -si 'https://staging.example.com/admin?role=user;role=admin' | head -n1

# [케이스3] 배열 비허용 엔드포인트에서 ids 중복
curl -si 'https://staging.example.com/report?ids=1&ids=2' | head -n1
```

### 6.2 Node.js로 간단 스모크

```js
// test/hpp-smoke.test.mjs (사내 스테이징 전용)
import https from "node:https";

function rq(path) {
  return new Promise((resolve, reject) => {
    const req = https.request(
      {
        host: "staging.example.com",
        path,
        method: "GET",
        rejectUnauthorized: false,
        headers: { "User-Agent": "hpp-smoke" }
      },
      (res) => {
        resolve({ code: res.statusCode, headers: res.headers });
      }
    );
    req.on("error", reject).end();
  });
}

const cases = [
  "/admin?role=user&role=admin",
  "/admin?role=user;role=admin",
  "/search?ids=1&ids=2",
];

for (const c of cases) {
  // eslint-disable-next-line no-await-in-loop
  const r = await rq(c);
  console.log(c, "→", r.code);
}
```

**기대 행동 예시**

- `/admin?role=user&role=admin` → 400, 422, 혹은 적어도 403
- `/search?ids=1&ids=2` → 이 엔드포인트에서 배열을 허용하지 않으면 400/422

---

## 7. 레이어별 방어 전략 — 큰 그림

### 7.1 공통 설계 원칙

1. **전 계층 동일 정책**
   - “중복 파라미터는 원칙적으로 허용하지 않는다”
   - 예외(배열 등)는 **스키마·문서·코드**에서 명시
2. **배열은 명시적 표기**
   - `ids[]=1&ids[]=2` / JSON 바디 등
   - `ids=1&ids=2`는 **오류로 처리**
3. **게이트웨이에서 선차단 + 애플리케이션에서 2차 검증**
   - 게이트웨이 / WAF / API Gateway: 중복 탐지 → 즉시 4xx
   - 애플리케이션: 예상치 못한 배열·중복이 들어오면 다시 거절
4. **로그·모니터링까지 설계에 포함**
   - 중복 파라미터 거절 이벤트를 **보안 이벤트로 취급**

### 7.2 단순 구조도

```text
[Client] 
   ↓ (쿼리/바디 생성 — 중복 파라미터 없음이 기본)
[CDN / WAF / L7 LB]
   ↓ (중복 파라미터 차단, 세미콜론 금지)
[API Gateway / Service Mesh]
   ↓ (스키마 기반 검증, 배열 허용 키 화이트리스트)
[Application Framework]
   ↓ (쿼리/바디 파서 + HPP Guard 미들웨어)
[Business Logic / DB / Cache]
```

---

## 8. 애플리케이션 레이어 방어 — 언어·프레임워크별 예제

### 8.1 공통 아이디어: “배열 화이트리스트 + 중복 금지”

핵심 로직은 모든 언어에서 거의 같다.

1. **허용할 “여러 값” 키 목록**을 정한다. (예: `ids[]`, `tags[]`)
2. 그 외의 키에서 **같은 이름이 여러 번 등장**하면:
   - 400 / 422 등으로 거절
3. 필요하다면
   - “중복 금지” 정책을 유지함과 동시에
   - 내부적으로는 “첫 값만 사용” 또는 “마지막 값만 사용”으로 통일해도 된다.
   - 단, 이 정책은 **프록시·게이트웨이와 맞춰야 한다.**

---

### 8.2 Node.js / Express — `hppGuard` 미들웨어

```js
// security/hpp.js
const MULTI_ALLOWED = new Set(["ids[]", "tags[]"]);  // 배열 허용 키
const ALSO_ALLOW_PLAIN = new Set(["ids", "tags"]);   // 필요시 레거시 호환

export function hppGuard(req, res, next) {
  // 1) 쿼리스트링 중복 검사
  const url = new URL(req.originalUrl, "http://dummy");
  const sp = url.searchParams;

  for (const key of new Set([...sp.keys()])) {
    const values = sp.getAll(key);
    const isMulti = values.length > 1;
    const allowed = MULTI_ALLOWED.has(key) || ALSO_ALLOW_PLAIN.has(key);
    if (isMulti && !allowed) {
      return res.status(400).json({ error: "duplicate_param", key });
    }
  }

  // 2) 폼 바디 중복 검사 (x-www-form-urlencoded 전용)
  if (req.is("application/x-www-form-urlencoded")) {
    for (const [k, v] of Object.entries(req.body || {})) {
      const isArray = Array.isArray(v);
      const allowed = MULTI_ALLOWED.has(k) || ALSO_ALLOW_PLAIN.has(k);
      if (isArray && !allowed) {
        return res.status(400).json({ error: "duplicate_form_param", key: k });
      }
    }
  }

  // 3) 정책 선택: 중복 금지 키는 '첫 값만' 남기기(선택)
  for (const key of new Set([...sp.keys()])) {
    if (MULTI_ALLOWED.has(key) || ALSO_ALLOW_PLAIN.has(key)) continue;
    const all = sp.getAll(key);
    if (all.length > 1) {
      sp.delete(key);
      sp.append(key, all[0]); // '첫 값 우선'으로 정규화
    }
  }

  return next();
}
```

Express에 적용:

```js
import express from "express";
import bodyParser from "body-parser";
import { hppGuard } from "./security/hpp.js";

const app = express();
app.use(bodyParser.json({ limit: "100kb" }));
app.use(bodyParser.urlencoded({ extended: false })); // 단순 파서
app.use(hppGuard);

// 관리자 엔드포인트 — role은 단일만 허용
app.get("/admin", (req, res) => {
  const role = req.query.role;
  if (role !== "admin") return res.status(403).send("forbidden");
  res.send("ok");
});

// 배열 허용 엔드포인트
app.get("/search", (req, res) => {
  const ids = []
    .concat(req.query["ids[]"] || [])
    .concat(req.query["ids"] || []);
  res.json({ ids });
});

export default app;
```

#### “혼합 getter” 금지

- `req.param(name)`처럼 라우트·쿼리·바디를 **섞어서 가져오는 API**는
  - **출처·우선순위를 흐리게** 만들어 HPP뿐 아니라 다른 취약점도 키운다.
- 항상:
  - `req.params` (경로 파라미터)
  - `req.query` (쿼리)
  - `req.body` (바디)
  를 **명확히 구분해 사용**한다.

---

### 8.3 Python / Flask

```python
from flask import Flask, request, abort

app = Flask(__name__)
ALLOWED_MULTI = {"ids[]", "tags[]"}  # 명시적 배열 허용 키

@app.before_request
def hpp_guard():
    # 1) 쿼리스트링 검사
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

### 8.4 Django

```python
# middleware.py
from django.http import HttpResponseBadRequest

ALLOWED_MULTI = {"ids[]", "tags[]"}

def hpp_guard(get_response):
    def middleware(request):
        # GET
        for k in request.GET.keys():
            vals = request.GET.getlist(k)
            if len(vals) > 1 and k not in ALLOWED_MULTI:
                return HttpResponseBadRequest(f"duplicate param: {k}")
        # POST (폼 바디)
        if request.method in ("POST", "PUT", "PATCH") and \
           request.content_type == "application/x-www-form-urlencoded":
            for k in request.POST.keys():
                vals = request.POST.getlist(k)
                if len(vals) > 1 and k not in ALLOWED_MULTI:
                    return HttpResponseBadRequest(f"duplicate form param: {k}")
        return get_response(request)
    return middleware
```

`settings.py`에 등록:

```python
MIDDLEWARE = [
    # ...
    "myproject.middleware.hpp_guard",
    # ...
]
```

---

### 8.5 Java / Spring Boot

```java
@Component
public class HppFilter implements Filter {

  private static final Set<String> ALLOWED_MULTI =
      Set.of("ids[]", "tags[]");

  @Override
  public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
      throws IOException, ServletException {

    HttpServletRequest r = (HttpServletRequest) req;
    Enumeration<String> names = r.getParameterNames();

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

컨트롤러에서는,

- 배열이 필요한 경우에만 `List<String>` 또는 `String[]`를 사용하고,
- 그렇지 않은 필드는 기본적으로 단일 값만 허용한다.

---

### 8.6 Go / net/http

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

### 8.7 ASP.NET Core

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

var allowed = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
{
    "ids[]", "tags[]"
};

app.Use(async (ctx, next) =>
{
    foreach (var kv in ctx.Request.Query)
    {
        if (kv.Value.Count > 1 && !allowed.Contains(kv.Key))
        {
            ctx.Response.StatusCode = 400;
            await ctx.Response.WriteAsync("duplicate param: " + kv.Key);
            return;
        }
    }
    await next();
});

// 이후 엔드포인트들...
app.Run();
```

---

### 8.8 JSON / GraphQL API에서의 HPP

- JSON Body 기반 API는
  - 표준상 **같은 키를 여러 번 두지 말아야** 하지만,
  - 실제 구현에 따라 “마지막 값 우선” 등으로 처리되기도 한다.
- 특히 GraphQL에서
  - 쿼리 스키마는 엄격하지만
  - **HTTP 레벨 쿼리스트링**(예: `?operationName=`)과 결합해 쓰일 때
  - HPP가 여전히 문제를 일으킬 수 있다.

실무 원칙:

1. GraphQL / JSON API에서도
   - HTTP 쿼리스트링은 **HPP Guard**를 통과시킨다.
2. JSON 바디는
   - 스키마 검증(JOI, marshmallow, class-validator 등)에서
   - **중복 키**를 허용하지 않는 파서를 선택하거나
   - 중복 키 발견 시 에러를 내도록 설정한다.

---

## 9. 게이트웨이·프록시 레벨 선차단

### 9.1 왜 게이트웨이에서 막아야 하는가

- 게이트웨이는
  - **모든 요청이 반드시 지나가는 choke point**다.
- 여기에서 HPP 패턴을 차단하면:
  - 애플리케이션 수정 없이도
  - 상당한 비율의 HPP·SSPP 시도를 사전에 거를 수 있다.

---

### 9.2 Nginx — 정규식 기반 간단 차단

```nginx
# role 키가 두 번 이상 등장하면 거절 (간단 예시)

set $dup_role 0;
if ($query_string ~* "(^|[;&])role=[^;&]*([;&].*role=)") {
    set $dup_role 1;
}
if ($dup_role) {
    return 400;
}

# 세미콜론(;)을 분리자로 쓰지 않도록 전역적으로 금지(정책에 따라)

if ($query_string ~* ";") {
    return 400;
}
```

단점:

- 모든 키에 대한 일반적인 중복 검사에는 한계가 있다.
- 복잡한 케이스에 대응하려면 **정확한 파싱**이 필요하다.

---

### 9.3 Nginx + Lua(OpenResty) — 정확한 파싱 후 차단

```nginx
location / {
  access_by_lua_block {
    local allowed = { ["ids[]"]=true, ["tags[]"]=true }
    local args = ngx.req.get_uri_args(1000)  -- 최대 1000개 파라미터 파싱

    for k, v in pairs(args) do
      if type(v) == "table" and not allowed[k] then
        return ngx.exit(400)
      end
    end
  }

  proxy_pass http://app;
}
```

장점:

- Nginx 기반에서 **실제 파서를 사용해 정확히 판단**할 수 있다.
- 배열 허용 키 화이트리스트도 쉽게 적용 가능.

---

### 9.4 HAProxy — ACL 기반

```haproxy
frontend fe
  bind :443 ssl crt /etc/haproxy/cert.pem

  # role 키 중복 탐지(간단 정규식)
  http-request deny if { query -m reg "(^|[&;])role=[^&;]*[&;].*role=" }

  default_backend be
```

보다 일반적인 HPP 방어는

- HTTP 파서(HTX)를 활용한
  - Lua / 도메인별 규칙
  - 또는 API Gateway 수준의 검증으로 확장할 수 있다.

---

### 9.5 클라우드/API 게이트웨이

#### (1) AWS API Gateway + Lambda

- `multiValueQueryStringParameters`로
  - 중복 파라미터가 모두 들어온다.
- Lambda 핸들러에서 **중복 검사 후 거절**:

```python
def handler(event, context):
    multi = event.get("multiValueQueryStringParameters") or {}
    allowed_multi = {"ids[]","tags[]"}

    for k, vals in multi.items():
        if len(vals) > 1 and k not in allowed_multi:
            return {
                "statusCode": 400,
                "body": f"duplicate param: {k}"
            }

    # 이후 정상 처리...
```

#### (2) CloudFront Functions / Workers

- 엣지에서 URL 쿼리 파라미터를 파싱하여
  - 중복 발견 시
    - 즉시 4xx 반환
    - 또는 로깅 후 제거/정규화

---

## 10. 프런트엔드 / 클라이언트 수칙

### 10.1 URL 생성 시 중복 방지

```js
function buildQuery(params) {
  const usp = new URLSearchParams();

  for (const [k, v] of Object.entries(params)) {
    if (Array.isArray(v)) {
      for (const x of v) {
        usp.append(`${k}[]`, String(x));   // 배열은 [] 표기
      }
    } else {
      if (usp.has(k)) {
        throw new Error("duplicate param: " + k); // 실수로 중복 생성 방지
      }
      usp.set(k, String(v));
    }
  }

  return usp.toString();
}
```

- Axios, fetch 등에 사용할 때
  - 이처럼 **별도 헬퍼**를 만들어 두면
  - 클라이언트에서 불필요한 HPP를 만들 여지를 줄일 수 있다.

### 10.2 클라이언트사이드 HPP 방지

- `location.search`를 그대로 다른 URL에 붙이지 않는다.
- 필요하다면:
  1. `new URLSearchParams(location.search)`로 파싱
  2. 필요한 키만 **화이트리스트**로 추출
  3. 새로운 `URLSearchParams`에 재구성

---

## 11. 분리자, 이중 디코딩, 인코딩 이슈

### 11.1 세미콜론(;) 분리자

- 일부 서버/프록시는
  - `;`를 `&`와 동일한 파라미터 분리자로 취급할 수 있다.
- 예:
  - `/admin?role=user;role=admin`
- 어떤 계층은
  - `role=user;role=admin` 전체를 **하나의 값**으로 보고,
- 다른 계층은
  - `role=user`와 `role=admin` **두 개의 파라미터**로 본다.

**대응**:

- 정책상 `;`를 쿼리스트링에서 사용하지 않기로 했으면, **게이트웨이에서 즉시 차단**한다.
- 아니면, **모든 계층에서 `;`는 값으로만 인정**하고 분리자로 쓰지 않도록 설정한다.

### 11.2 이중 디코딩(Double-decoding)

```text
/search?q=1%26admin=true
```

- 첫 번째 디코딩: `q="1&admin=true"`
- 어떤 계층은 여기서 다시 디코딩하거나, `&`를 분리자로 쓸 수 있다.
- 결과적으로
  - 처음에는 파라미터 하나로 들어왔지만
  - 나중에는 `q=1` / `admin=true` 두 개의 파라미터가 된 것처럼 보일 수 있다.

**대응**:

- **디코딩은 한 번만** 수행하고, 그 이후에는 해당 값을 **순수 값으로만 취급**한다.
- 게이트웨이/프록시에서
  - “디코딩 → 재인코딩” 단계를 한 번만 수행하고,
  - 이후 계층은 **다시 디코딩하지 않도록** 설계한다.

### 11.3 인코딩 조합 공격

- `%3D`, `%26`, `%3B` 등
  - `=`, `&`, `;`를 인코딩한 값이
- 어떤 계층에서는 그대로 값으로 남고,
- 어떤 계층에서는 다시 분리자로 사용될 수 있다.

**실무 팁**

- 가능한 한 **입력 계층에서 fully decoded 상태**를 만든 후
  - 내부 시스템에는
  - “재인코딩된 안정된 표현”으로만 전달한다.
- WAF/게이트웨이에서
  - 다양한 인코딩 조합을 미리 정규화(normalization)해 단일 형태로 바꾼 뒤
  - 그 기반으로 탐지·제어한다.

---

## 12. 테스트·CI — 재발 방지 자동화

### 12.1 단위 테스트 (Express 예시)

```js
import request from "supertest";
import app from "../app.js"; // hppGuard 적용된 서버

test("중복 role 거절", async () => {
  const r = await request(app).get("/admin?role=user&role=admin");
  expect([400, 422, 403]).toContain(r.status);
});

test("ids[] 배열만 허용", async () => {
  const ok = await request(app).get("/search?ids[]=1&ids[]=2");
  expect(ok.status).toBe(200);

  const bad = await request(app).get("/search?ids=1&ids=2");
  expect([400, 422]).toContain(bad.status);
});
```

### 12.2 정적 분석/grep 가드

```bash
# 예: req.param 사용 금지
if grep -R --line-number -E "req\.param\(" src/; then
  echo "::error title=HPP risk::Do not use req.param() (ambiguous source)" >&2
  exit 1
fi
```

- 프레임워크마다
  - “출처가 섞인 파라미터 접근 메서드”
  를 금지하는 규칙을 만들어 CI에 넣을 수 있다.

---

## 13. 로깅·모니터링·탐지

### 13.1 로깅 스키마 설계

예: JSON 로그 필드

```json
{
  "ts": "2025-11-26T10:12:34Z",
  "service": "edge",
  "route": "/admin",
  "ip": "203.0.113.10",
  "user_id": "u12345",
  "dup_keys": ["role"],
  "policy": "reject",
  "status": 400
}
```

- `dup_keys`: 중복이 탐지된 파라미터 목록
- `policy`: `"reject"`, `"normalize_first"`, `"normalize_last"` 등

### 13.2 탐지 룰 예시

- **중복 거절 카운트**:
  - 5분 동안 평소보다 갑자기 많은 중복 키 거절이 발생하면
  - 스캐닝/공격 시도로 볼 수 있다.
- **세미콜론 포함 쿼리 증가**:
  - 평소 거의 없다가 특정 시점에 급증하면
  - 분리자 악용/HPP 시도로 의심 가능
- **민감 키 중복**:
  - `role`, `admin`, `debug`, `safe`, `redirect` 등의 키에 대한 중복 시도는
  - 별도의 알림을 줄 가치가 있다.

---

## 14. 운영 체크리스트 (요약)

- [ ] **전 계층 동일 정책**: 중복 파라미터는 원칙적으로 허용하지 않음
- [ ] **배열 허용은 명시적**: `[]` 표기 또는 JSON 바디만 허용
- [ ] **게이트웨이/프록시**:
  - 중복 탐지/차단
  - 세미콜론 금지 또는 값으로만 취급
  - 이중 디코딩·이상 인코딩 차단
- [ ] **애플리케이션 미들웨어**:
  - 중복 거절 + 정규화
  - 혼합 getter (`req.param`, `request["all"]`) 사용 금지
- [ ] **민감 키**:
  - `role`, `admin`, `safe`, `debug`, `redirect`, `feature`, `flag` 등에 대해
  - 중복이 오면 특별히 차단/알림
- [ ] **테스트/CI**:
  - 중복/세미콜론/대량 파라미터 스모크 테스트
  - 위험한 API 호출 패턴을 정적 분석으로 차단
- [ ] **로그/탐지**:
  - 거절 카운트, 키/경로별 통계, 이상 징후 알림
- [ ] **문서화**:
  - “배열은 `[]` 표기만 허용”, “쿼리에서 세미콜론 사용 금지” 등
  - 클라이언트·프론트엔드·외부 파트너용 가이드 배포

---

## 15. 미니 실습 — 취약 서비스 → 방어 적용

### 15.1 취약 Express 코드

```js
import express from "express";
const app = express();

app.get("/admin", (req, res) => {
  // role이 admin일 때만 허용한다고 생각
  const role = req.query.role;
  if (role !== "admin") return res.status(403).send("forbidden");
  res.send("ok");
});

app.listen(3000);
```

- `GET /admin?role=user&role=admin` 요청 시
  - Express의 쿼리 파서 설정에 따라
  - `role`이 `"admin"`으로 들어올 수 있다.
- 프록시/프런트 검증이 단일 `role`만 가정했다면,
  - HPP로 권한 상승이 가능하다.

### 15.2 방어 적용

1) **HPP Guard 미들웨어 추가**

```js
import bodyParser from "body-parser";
import { hppGuard } from "./security/hpp.js";

app.use(bodyParser.urlencoded({ extended: false }));
app.use(bodyParser.json());
app.use(hppGuard);
```

2) **테스트 코드로 검증**

```js
import request from "supertest";

test("role 중복 거절", async () => {
  const r = await request(app).get("/admin?role=user&role=admin");
  expect([400,403]).toContain(r.status);
});
```

3) **게이트웨이에서도 선차단**

- Nginx / HAProxy / API Gateway 설정으로
  - `role` 키 중복을 **엣지에서 이미 거절**하도록 한다.

이렇게 하면, HPP가 실제 비즈니스 로직에 영향을 미치기 전에 **여러 겹의 방어막**을 거치게 된다.

---

## 16. 맺음말

- HPP는 “**각 계층이 파라미터를 다르게 해석한다**”는 아주 작은 설계 차이에서 출발하지만,
  - 실제로는 **권한 상승, 검증 우회, 캐시 오염, WAF 회피** 같은 큰 사고로 이어질 수 있다.
- 반대로 말하면,
  - **중복 금지/정규화**
  - **배열의 명시적 허용**
  - **전 계층 정책 통일**
  만 잘 지켜도, HPP의 대부분을 **구조적으로 제거**할 수 있다.
- 최근 국제 보고서들에서 반복해서 지적하듯,
  - 웹 애플리케이션과 API 계층의 취약점은 여전히 주요 침해 원인이다.
- HPP를 단순한 “특수 취약점”이 아니라,
  - **입력/파라미터 설계 전반을 검증하는 리트머스 시험지**로 삼아
  - 게이트웨이–애플리케이션–테스트–로깅까지
  - 한 번에 정리해 두는 것이 장기적으로 큰 이득을 준다.