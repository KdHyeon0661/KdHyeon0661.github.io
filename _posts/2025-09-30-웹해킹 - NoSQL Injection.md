---
layout: post
title: 웹해킹 - NoSQL Injection
date: 2025-09-30 21:25:23 +0900
category: 웹해킹
---
# NoSQL Injection (MongoDB / Elasticsearch)

## 0. 요약 (Executive Summary)

이 글은 다음 상황을 전제로 한다.

- 백엔드가 **MongoDB**나 **Elasticsearch**를 쓰고 있고,
- 프런트/게이트웨이/백엔드 사이에 **JSON·쿼리 DSL**이 오가며,
- “유연함”을 이유로 **사용자 입력을 거의 그대로 쿼리에 태우는** 코드가 존재할 수 있다.

이때 발생하는 대표적인 취약점이 **NoSQL Injection**이다. 관계형 DB의 SQL Injection과 마찬가지로, “데이터로 취급해야 할 입력”이 **쿼리 연산자/스크립트/DSL**로 해석되는 순간 문제가 시작된다. 최근 보안 리포트와 실무 분석에 따르면 NoSQL 인젝션은 여전히 인증 우회·데이터 탈취에 실질적으로 악용되는 취약점으로 분류된다. 특히 **MongoDB의 연산자 주입(operator injection)** 과 **Elasticsearch의 DSL/스크립트 주입**이 대표적이다.

### 0.1 문제(패턴)

- **MongoDB**
  - 애플리케이션이 `req.body`, `req.query` 같은 **객체 전체를 그대로** `find()`, `findOne()`에 넘기면,
    - 키가 `$ne`, `$gt`, `$regex`, `$where` 등으로 시작하는 **Mongo 연산자**가 주입되어
    - **인증 우회, 필터 우회, 대량 조회, 서버측 JS 평가**로 이어질 수 있다.
- **Elasticsearch**
  - `/_search?q=<사용자입력>` 같은 **Query String Query**에 사용자를 그대로 태우거나,
  - `script`, `script_score`, 템플릿(예: Mustache) 내부에 사용자가 관여하면
    - **DSL/스크립트 주입**을 통해
    - 필터 우회·대량 스캔·최악의 경우 서버측 코드 실행까지 이어질 수 있다.

### 0.2 핵심 방어 전략 (요약)

1. **스키마 검증(화이트리스트)**  
   - 필드 이름 / 타입 / 길이 / enum 값을 **명시**하고 벗어나면 **즉시 거절**.
2. **연산자 키 차단·정규화**  
   - 외부 입력 JSON에서 **키에 `$`나 `.`가 포함되면 거절**.  
   - `$` 연산자는 **내부 쿼리 빌더**만 사용.
3. **쿼리 빌더만 사용**  
   - “사용자 입력 → 검증/정규화 → **전용 쿼리 빌더** → DB 쿼리” 흐름으로 고정.
4. **민감 API 보호**  
   - 로그인/관리 검색 등은 **레이트 리밋, 페이지 크기 상한, 프로젝션 화이트리스트** 필수.
5. **로그/탐지**  
   - `$`로 시작하는 키, 비정상 필드/연산자, 대량 스캔 패턴을 **따로 로깅·알림**.

---

## 1. 위협 모델과 발생 지점

### 1.1 왜 NoSQL에서 인젝션이 쉬워지는가

전통적인 SQL은 쿼리가 **문자열**로 조립되고, 여기에 문자열 결합으로 사용자 입력이 붙는다는 점에서 인젝션 위험이 존재한다. NoSQL 계열은 표면상 **“JSON/BSON/DSL”** 기반이라서 문자열 인젝션이 덜할 것처럼 보인다. 그러나 실제로는:

- 백엔드가 `Map<String, Object>` 또는 `interface{}`(Go), `any`(TS) 같은 **동적 타입**으로 요청 바디를 받고,
- 이를 **검증 없이** ORM/드라이버로 넘기면,
- 해당 드라이버는 이 **동적 구조를 그대로 쿼리 객체**로 사용한다.

결과적으로, **입력 디코딩 단계와 DB 쿼리 객체 생성 단계가 사실상 동일한 층**이 되어버리고, 이 경계를 악용하는 것이 NoSQL Injection이다. 최근 여러 보안 분석에서도 “NoSQL은 스키마가 느슨한 대신, 방어를 잘못하면 Injection 면이 넓어진다”는 점을 지적한다.

### 1.2 MongoDB: 연산자 주입과 서버 측 JS

MongoDB는 다음과 같은 연산자/기능을 제공한다.

- 비교 연산자: `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`
- 논리 연산자: `$or`, `$and`, `$not`, `$nor`
- 정규식: `$regex`
- JavaScript 기반 조건: `$where` (서버 측 JS 평가)

이 연산자들은 애플리케이션이 정상적으로 쓸 때는 매우 강력하지만, **입력에서 바로 등장할 수 있도록 열어두면** 공격 표면이 된다. 특히 `$where`는 서버 측 JavaScript를 평가하므로, 오늘날 대부분의 보안 가이드에서 **사용 금지 또는 강력 제한**을 권고한다.

### 1.3 Elasticsearch: Query String / DSL / Scripts

Elasticsearch는 HTTP API를 통해 JSON DSL을 받는다. 주요 위험 지점은:

- `/_search?q=...` 형태의 **Query String Query**:  
  사용자가 Lucene 쿼리 문법을 직접 쓰게 할 수 있는데, 필터 우회·대량 스캔에 악용될 수 있다.
- `script`, `script_score`, `painless` 스크립트:  
  잘못된 템플릿/설계로 사용자가 스크립트 일부를 제어하면 **NoSQL·스크립트 인젝션**으로 이어질 수 있다.
- 템플릿 언어(Mustache 등)를 사용한 **서치 템플릿**:  
  쿼리 DSL 문서 전체를 텍스트 템플릿으로 다루다가 사용자 입력을 그대로 삽입하면, DSL 구조 자체를 바꿀 수 있다.

---

## 2. 공격면 유형 (개념 정리)

실제 공격 페이로드는 여기서 다루지 않는다. 대신 어떤 설계 패턴이 위험한지를 **방어 관점**에서 정리한다.

### 2.1 MongoDB — Operator Injection

- 취약 설계 예:
  - `User.findOne(req.body)` 처럼 **요청 바디 전체를 조건 객체로 사용**.
  - `const query = { username: req.body.username };` 처럼 보이지만,  
    실제로 `username`이 **문자열이 아니라 객체**일 수 있도록 열어둔 경우.
- 결과:
  - `{"username": { ...연산자... }}` 형식의 값이 들어오면,  
    의도했던 “단일 값 비교”가 아니라 **복합 조건/우회 조건**이 된다.
- 특히 인증 로직에서 “계정/비밀번호” 조건이 연산자에 의해 무력화될 수 있다.

### 2.2 MongoDB — JavaScript Injection (`$where`)

- `$where` 연산자를 통해 **JavaScript 표현식**을 평가할 수 있다.
- 과거에는 복잡한 조건을 편하게 쓰기 위해 자주 사용됐지만,
  - 오늘날은 **서버 측 코드 실행** 위험 때문에 대부분의 보안 가이드에서 사용 금지.
- 애플리케이션이 `$where: <사용자 입력>` 형태로 조합하면
  - 타임 기반 브루트포스, 데이터 필터 우회 등으로 악용될 수 있다.

### 2.3 MongoDB — 정규식 남용 (`$regex`)

- `%LIKE%` 패턴처럼 “포괄 검색”을 위해 `$regex`를 남용하면,
  - 공격자가 **매우 비싼 정규식**을 넣어서 컬렉션 전체를 스캔시키는 식으로 DoS에 가까운 부하를 줄 수 있다.
- 또한 인덱스가 없는 필드에 `$regex`를 허용하면, 성능이 급격히 저하된다.

### 2.4 Elasticsearch — Query String Injection

- `GET /index/_search?q=<사용자입력>` 패턴은
  - 사용자 입력이 Lucene 쿼리 문법 전체에 노출될 수 있다.
- 입력 검증 없이
  - `q=field:value` 형식을 기대하지만,
  - 실제로는 **연산자·와일드카드·범위 쿼리** 등을 끼워 넣을 수 있다.
- 결과적으로 특정 필터를 우회하거나,
  - 여러 인덱스를 한 번에 뒤지는 등 **과도한 스캔**을 유발할 수 있다.

### 2.5 Elasticsearch — Script / Template Injection

- `script_score` 등에서 `script` 값을 사용자가 직접 제어하면,
  - 점수 계산 외의 목적으로 스크립트가 악용될 수 있다.
{% raw %}
- Mustache 템플릿 기반 서치 템플릿에서 `{{{user_input}}}` 같은 **비인코딩 자리표시자**를 쓰면,
{% endraw %}
  - 사용자가 DSL 구조를 깨고 자신의 조건·스크립트를 넣을 수 있다.

---

## 3. 안전한 재현/탐지 (스테이징 환경 전제)

> 목적: **“우리 방어 로직이 제대로 작동하는지”만 확인**한다.  
> 운영 환경·제3자 시스템에 대한 테스트는 허용되지 않는다는 전제를 깔고,  
> 내부 스테이징/랩 환경에서만 쓰는 **차단 스모크 테스트** 관점으로 예제를 구성한다.

### 3.1 MongoDB — 연산자 키 스모크 테스트

`sanitizeDocument()` 같은 함수가 **키에 `$`나 `.`가 포함된 경우 예외를 던지는지** 확인하는 단위 테스트:

```js
// test/opkey-guard.test.js
import assert from "node:assert";
import { sanitizeDocument } from "../src/security/mongo-sanitize.js";

assert.throws(
  () => sanitizeDocument({ username: "u", password: { $ne: "" } }),
  /operator/i
);

assert.throws(
  () => sanitizeDocument({ "profile.pic": "x" }),
  /dot key/i
);

console.log("operator/dot key blocked");
```

의도는 “이런 구조의 JSON이 들어오면 **정상적인 요청이 아니므로 차단**한다”는 것을 자동 테스트로 보장하는 것이다.

### 3.2 Elasticsearch — Query String 차단 확인

Elasticsearch 앞단에 프록시/게이트웨이를 두고, `q=` 파라미터를 사용하지 않겠다는 정책을 세웠다면:

```bash
# q 파라미터가 붙은 요청은 항상 400/403이 나와야 정상

curl -si 'https://es.staging.example.com/my-index/_search?q=anything' | head -n1
# 기대 응답: HTTP/1.1 400 Bad Request (또는 403 Forbidden 등 정책 메시지)
```

---

## 4. MongoDB: 안전한 설계 패턴

### 4.1 입력 키 재귀 필터링 (`sanitizeDocument`)

Mongo 쿼리 객체에 태우기 전에, **모든 키를 재귀적으로 순회**하며 `$`/`.` 를 검사한다.

```js
// src/security/mongo-sanitize.js
export function sanitizeDocument(input, { allowOperators = false } = {}) {
  if (input == null) return input;
  if (typeof input !== "object") return input;

  // 배열이면 각 요소 재귀 처리
  if (Array.isArray(input)) {
    return input.map(v => sanitizeDocument(v, { allowOperators }));
  }

  const out = {};
  for (const [k, v] of Object.entries(input)) {
    if (k.includes(".")) {
      throw new Error("dot key blocked: " + k);
    }
    if (k.startsWith("$") && !allowOperators) {
      throw new Error("operator key blocked: " + k);
    }
    out[k] = sanitizeDocument(v, { allowOperators });
  }
  return out;
}
```

설명:

- **외부 요청** (`req.body`, `req.query`)에는 절대 `$`/`.` 키를 허용하지 않는다.
- 내부적으로 우리가 만드는 쿼리 빌더는 `allowOperators: true`로 호출할 수 있지만,
  - 이때는 **이미 화이트리스트 검증을 통과한 필드에만** `$`를 쓰도록 제한한다.

### 4.2 안전한 인증 로직 예제

취약 패턴:

```js
// 나쁜 예 (피해야 할 패턴) — operator injection 가능성
app.post("/login", async (req, res) => {
  const user = await User.findOne(req.body); // 절대 이런 식으로 쓰지 말 것
  // ...
});
```

안전 패턴:

```js
import bcrypt from "bcrypt";

app.post("/login", async (req, res) => {
  const { email, password } = req.body ?? {};
  if (typeof email !== "string" || typeof password !== "string") {
    return res.status(400).json({ error: "invalid" });
  }

  // 이메일 정규화
  const emailNorm = email.trim().toLowerCase();

  // 1) 조건은 우리가 직접 정의한 단일 필드만 사용
  const user = await User.findOne({ email: emailNorm }).lean();
  if (!user) {
    return res.status(401).json({ error: "auth" });
  }

  // 2) 비밀번호는 항상 해시 비교
  const ok = await bcrypt.compare(password, user.passwordHash);
  if (!ok) {
    return res.status(401).json({ error: "auth" });
  }

  // 3) 세션/토큰 발급
  // ...
  res.sendStatus(204);
});
```

포인트:

- DB 쿼리의 where 조건에 **사용자 입력 객체 전체**를 넘기지 말고,
- **명시적으로 정의한 필드만 사용**한다.
- 비밀번호 비교는 **쿼리가 아니라 애플리케이션 로직**에서 처리한다.

### 4.3 “전용 쿼리 빌더” 도입

복잡한 검색 UI에서 다양한 필터를 노출하고 싶다면, 필터를 다음과 같은 구조로 정의할 수 있다.

```js
// 클라이언트 → 서버로 넘어오는 필터 구조 예
// GET /users?f[0].field=email&f[0].op=eq&f[0].value=a@b.com

[
  { field: "email",  op: "eq",     value: "a@b.com" },
  { field: "age",    op: "gte",    value: "20" },
  { field: "status", op: "in",     value: ["active","pending"] }
]
```

이를 위해, 서버 쪽에 **화이트리스트 기반 쿼리 빌더**를 둔다.

```js
// src/security/mongo-query-builder.js
const ALLOWED_FIELDS = {
  email:  "string",
  age:    "number",
  status: ["active","inactive","pending"], // enum
  tags:   "string[]"
};

const ALLOWED_OPS = new Set(["eq","in","gt","gte","lt","lte","prefix"]);

function validateField(field) {
  if (!Object.hasOwn(ALLOWED_FIELDS, field)) {
    throw new Error("field not allowed: " + field);
  }
  return field;
}

function coerce(field, value) {
  const t = ALLOWED_FIELDS[field];

  if (Array.isArray(t)) {
    // enum
    if (!t.includes(value)) throw new Error("enum invalid");
    return value;
  }

  if (t === "number") {
    const n = Number(value);
    if (!Number.isFinite(n)) throw new Error("number invalid");
    return n;
  }

  if (t === "string[]") {
    if (!Array.isArray(value) || value.some(v => typeof v !== "string")) {
      throw new Error("array invalid");
    }
    return value.map(v => v.trim());
  }

  if (t === "string") {
    if (typeof value !== "string") throw new Error("string invalid");
    return value.trim();
  }

  throw new Error("type invalid");
}

function escapeRegex(s) {
  return s.replace(/[-\/\\^$*+?.()|[\]{}]/g, "\\$&");
}

export function buildMongoQuery(filters = []) {
  const query = {};

  for (const f of filters) {
    const field = validateField(String(f.field || ""));
    const op    = String(f.op || "");
    if (!ALLOWED_OPS.has(op)) {
      throw new Error("op not allowed: " + op);
    }

    const val = coerce(field, f.value);

    if (op === "eq") {
      query[field] = val;
    } else if (op === "in") {
      query[field] = { $in: Array.isArray(val) ? val : [val] };
    } else if (op === "gt") {
      query[field] = { $gt: val };
    } else if (op === "gte") {
      query[field] = { $gte: val };
    } else if (op === "lt") {
      query[field] = { $lt: val };
    } else if (op === "lte") {
      query[field] = { $lte: val };
    } else if (op === "prefix") {
      // 정규식은 prefix 전용으로 제한
      const s = String(val);
      if (s.length > 40) throw new Error("prefix too long");
      query[field] = { $regex: new RegExp("^" + escapeRegex(s)) };
    }
  }

  return query;
}
```

라우트에서의 사용 예:

```js
import { sanitizeDocument } from "./security/mongo-sanitize.js";
import { buildMongoQuery } from "./security/mongo-query-builder.js";

app.get("/users", async (req, res) => {
  try {
    // 1) 외부 입력 JSON 키에 $/. 이 있는지 전역 검사
    sanitizeDocument(req.query);

    // 2) 필터 파싱
    const filters = Array.isArray(req.query.f) ? req.query.f : [];
    const query = buildMongoQuery(filters);

    // 3) 페이지네이션 상한
    const limit = Math.min(Number(req.query.limit || 20), 100);
    const page  = Math.max(Number(req.query.page || 1), 1);
    const skip  = (page - 1) * limit;

    // 4) 프로젝션 화이트리스트
    const rows = await User
      .find(query, { email: 1, age: 1, status: 1 })
      .sort({ _id: -1 })
      .skip(skip)
      .limit(limit)
      .lean();

    res.json({ rows, page, limit });
  } catch (e) {
    res.status(400).json({ error: "bad_filter" });
  }
});
```

포인트:

- 외부 입력에서는 `$`/`.` 키를 **무조건 거절**.
- 허용된 필드/연산자만 사용해서 쿼리를 조립.
- `regex`는 prefix 정도로 제한해서 성능·악용면을 줄인다.
- 민감 필드는 프로젝션에서 제외한다.

### 4.4 PyMongo에서도 동일 원칙 적용

```python
# security/mongo_sanitize.py
def sanitize_document(obj, allow_ops=False):
    if obj is None:
        return obj
    if isinstance(obj, list):
        return [sanitize_document(x, allow_ops) for x in obj]
    if not isinstance(obj, dict):
        return obj

    out = {}
    for k, v in obj.items():
        if "." in k:
            raise ValueError("dot key blocked: " + k)
        if k.startswith("$") and not allow_ops:
            raise ValueError("operator key blocked: " + k)
        out[k] = sanitize_document(v, allow_ops)
    return out
```

```python
# 예: Flask 라우트에서 사용
from flask import Flask, request, abort
from security.mongo_sanitize import sanitize_document

app = Flask(__name__)

@app.before_request
def guard_mongo():
    if request.is_json:
        try:
            sanitize_document(request.get_json())
        except ValueError as e:
            abort(400, description=str(e))
```

---

## 5. Mongoose 스키마·옵션으로 추가 방어

Mongoose 자체 기능으로도 방어선을 추가할 수 있다.

```js
import { Schema, model } from "mongoose";

const UserSchema = new Schema({
  email: {
    type: String,
    index: true,
    required: true,
    lowercase: true,
    trim: true
  },
  passwordHash: {
    type: String,
    required: true
  },
  age: {
    type: Number,
    min: 0,
    max: 150
  },
  status: {
    type: String,
    enum: ["active", "inactive", "pending"],
    default: "active"
  }
}, {
  strict: "throw",   // 스키마에 정의되지 않은 필드는 예외
  minimize: true,
  versionKey: false
});

export const User = model("User", UserSchema);
```

설명:

- `strict: "throw"` 를 통해 **예상치 못한 필드**가 들어와도 조용히 무시하지 않고,
  - 애초에 저장/갱신을 막는다.
- 값의 최소/최대, enum 등은 NoSQL Injection 뿐 아니라
  - 비정상 입력 방어에도 중요하다.

---

## 6. Aggregation Pipeline: 화이트리스트 파이프라인만 허용

Aggregation은 기능이 강력한 만큼, 외부 입력을 그대로 조합하면 위험하다.  
따라서 “**허용된 스텝만** 조합할 수 있는 빌더”를 도입한다.

```js
const AGG_ALLOW = new Set(["match", "group", "sort", "limit", "project"]);

export function buildAgg(specs = []) {
  const pipe = [];

  for (const s of specs) {
    if (!AGG_ALLOW.has(s.type)) {
      throw new Error("agg type not allowed: " + s.type);
    }

    if (s.type === "match") {
      pipe.push({ $match: buildMongoQuery(s.filters) });
    } else if (s.type === "group") {
      const field = String(s.field || "");
      if (!ALLOWED_FIELDS[field]) throw new Error("field not allowed");
      pipe.push({
        $group: {
          _id: `$${field}`,
          count: { $sum: 1 }
        }
      });
    } else if (s.type === "sort") {
      const field = String(s.field || "");
      const dir   = s.dir === "asc" ? 1 : -1;
      if (!ALLOWED_FIELDS[field]) throw new Error("field not allowed");
      pipe.push({ $sort: { [field]: dir } });
    } else if (s.type === "limit") {
      const n = Math.min(Number(s.n || 20), 200);
      pipe.push({ $limit: n });
    } else if (s.type === "project") {
      const proj = {};
      for (const f of s.fields || []) {
        if (ALLOWED_FIELDS[f]) proj[f] = 1;
      }
      pipe.push({ $project: proj });
    }
  }

  return pipe;
}
```

`$where`, `$function`, 표현식 기반의 위험 기능은 아예 **DSL에서 배제**한다.

---

## 7. Elasticsearch: 안전한 설계 패턴

### 7.1 Query String 대신 REST/DSL + 빌더

취약 패턴:

```js
// 나쁜 예 — q=에 사용자 입력 그대로
client.search({
  index: "users",
  q: req.query.q
});
```

안전 패턴:

- `q` 대신, `filters` 구조를 정의하고,
- 서버에서 화이트리스트 기반 DSL 빌더로 변환.

```js
import { Client } from "@elastic/elasticsearch";

const es = new Client({ node: process.env.ES_URL });

const ES_FIELDS = {
  "name.keyword":  "keyword",
  "email.keyword": "keyword",
  "age":           "long",
  "status":        "keyword"
};

const ES_OPS = new Set(["term","terms","range","prefix"]);

function buildEsQuery(filters = []) {
  const must = [];

  for (const f of filters) {
    const field = String(f.field || "");
    const op    = String(f.op || "");
    const val   = f.value;

    if (!Object.hasOwn(ES_FIELDS, field)) {
      throw new Error("field not allowed: " + field);
    }
    if (!ES_OPS.has(op)) {
      throw new Error("op not allowed: " + op);
    }

    if (op === "term") {
      must.push({ term: { [field]: val } });
    } else if (op === "terms") {
      const arr = Array.isArray(val) ? val : [val];
      must.push({ terms: { [field]: arr } });
    } else if (op === "prefix") {
      const s = String(val);
      if (s.length > 30) throw new Error("prefix too long");
      must.push({ prefix: { [field]: s.toLowerCase() } });
    } else if (op === "range") {
      const r = {};
      if (val.gte != null) r.gte = Number(val.gte);
      if (val.lte != null) r.lte = Number(val.lte);
      must.push({ range: { [field]: r } });
    }
  }

  return { bool: { must } };
}
```

검색 라우트 예:

```js
app.get("/search", async (req, res) => {
  try {
    // 1) q=는 정책상 허용하지 않음
    if ("q" in req.query) {
      return res.status(400).json({ error: "use filters instead of q" });
    }

    const filters = Array.isArray(req.query.f) ? req.query.f : [];
    const query   = buildEsQuery(filters);

    const size = Math.min(Number(req.query.size || 20), 100);
    const from = Math.max(Number(req.query.from || 0), 0);

    const r = await es.search({
      index: "users",
      query,
      size,
      from,
      track_total_hits: false,
      _source: ["name","email","status"] // 프로젝션 화이트리스트
    });

    res.json(r.hits.hits.map(h => ({ id: h._id, ...h._source })));
  } catch (e) {
    res.status(400).json({ error: "bad_filter" });
  }
});
```

### 7.2 프록시에서 `q=` 선차단

Elasticsearch 앞단에 Nginx가 있다면:

```nginx
location ~ ^/.+/_search$ {
  if ($arg_q) {
    return 400;
  }
  proxy_pass http://es:9200;
}
```

이렇게 하면 **API 컨슈머가 실수로 q=를 사용하는 것** 자체를 막을 수 있다.

### 7.3 Script / Template 방어

- `script`, `script_score`는
  - **서버 쪽에서 고정된 스크립트** + **바인딩 파라미터만** 사용.
- 템플릿(Mustache 등)에선

{% raw %}
  - `{{{user_input}}}` (raw) 대신 `{{{user_input}}}` (escape) 사용.
{% endraw %}

- 가능하면 스크립트·템플릿은:
  - 내부 전용 관리 API에서만 쓰고,
  - 일반 검색 API에는 노출하지 않는다.

---

## 8. 프레임워크별 공통 미들웨어 패턴

### 8.1 Node/Express — NoSQL Injection 가드

```js
// Mongo + ES를 모두 사용하는 서비스에서,
// 공통적으로 키/타입을 검사하는 미들웨어 예

import { sanitizeDocument } from "./security/mongo-sanitize.js";

const MAX_BODY_SIZE = 100 * 1024; // 100KB 예시

export function nosqlGuard(req, res, next) {
  try {
    // 1) JSON body 크기 제한 (압축 해제 후 기준)
    const len = Number(req.headers["content-length"] || 0);
    if (len > MAX_BODY_SIZE) {
      return res.status(413).json({ error: "payload_too_large" });
    }

    // 2) JSON body 키 검사
    if (req.is("application/json") && req.body) {
      sanitizeDocument(req.body);
    }

    // 3) 쿼리스트링을 객체로 보고 키 검사
    const url = new URL(req.originalUrl, "http://dummy");
    const q   = {};
    for (const [k, v] of url.searchParams) {
      q[k] = v;
    }
    sanitizeDocument(q);

    return next();
  } catch (e) {
    return res.status(400).json({ error: "bad_input" });
  }
}
```

### 8.2 Flask — before_request 훅

```python
from flask import Flask, request, abort
from security.mongo_sanitize import sanitize_document

app = Flask(__name__)

@app.before_request
def nosql_guard():
    # JSON body 검사
    if request.is_json:
        try:
            sanitize_document(request.get_json())
        except ValueError as e:
            abort(400, description=str(e))

    # 쿼리스트링 검사
    if request.query_string:
        # 간단히 parse
        from urllib.parse import parse_qsl
        qs = dict(parse_qsl(request.query_string.decode("utf-8")))
        try:
            sanitize_document(qs)
        except ValueError as e:
            abort(400, description=str(e))
```

---

## 9. 로깅·모니터링·탐지

### 9.1 구조화 로그 설계

로그 필드 예:

- `ts`: 타임스탬프
- `route`: 엔드포인트
- `user_id`: 로그인 사용자 ID (가능하다면)
- `src_ip`: 클라이언트 IP
- `rejected_reason`:  
  - `operator_key`, `dot_key`, `bad_field`, `bad_op`, `too_large`, …
- `filters_count`: 필터 개수
- `db`: `"mongo" | "es"`

예시 로그:

```js
logger.warn({
  event: "reject_nosql_input",
  reason: "operator_key",
  route: "/users",
  src_ip: req.ip,
  user_id: req.user?.id ?? null
});
```

### 9.2 SIEM 룰 예시

Loki(LogQL):

```logql
{service="web", event="reject_nosql_input"} | json
| count_over_time(5m) > 20
```

Splunk(SPL):

```spl
index=app event=reject_nosql_input
| stats count by reason, src_ip
| where count > 5
```

- `operator_key` 같은 이유로 거절된 요청이 짧은 시간에 많이 생기면,
  - 의도적인 스캔/공격 시도로 볼 수 있다.

### 9.3 DB 레벨 지표

- MongoDB:
  - 컬렉션 스캔 비율, `$regex` 비인덱스 경로 비율,
  - 슬로우 쿼리 로그에 특정 API 관련 항목이 집중되는지 확인.
- Elasticsearch:
  - 특정 인덱스/엔드포인트에 대한 `search` 요청 수 급증,
  - `size`·`from` 값이 비정상적으로 큰 요청 패턴,
  - slolog/쿼리 캐시 미스 패턴.

---

## 10. 구성/플랫폼 레벨 수칙

### 10.1 MongoDB

- `$where`/서버 측 JS:
  - 가능하면 **기능 자체를 사용하지 않거나**,  
  - 드라이버 레벨에서 해당 API를 사용하지 않도록 한다.
- 최소 권한 계정:
  - 애플리케이션용 Mongo 사용자에게는
    - 해당 DB/컬렉션에 대한 **필요 최소 read/write 권한만** 부여.
    - admin 권한을 부여하지 않는다.
- 인덱스:
  - 쿼리 빌더에서 허용한 필드/연산자(prefix 등)에 맞춰 인덱스를 설계한다.
  - 허용하지 않는 쿼리는 인덱스를 만들지 않고, API에서 거절한다.

### 10.2 Elasticsearch

- 인덱스 매핑:
  - 검색 대상 필드는 `keyword` / `text`를 명확히 구분해,
    - 쿼리 동작이 예측 가능하도록 한다.
- 관리 API 보호:
  - `_scripts`, `_template`, `_cluster`, `_cat` 등은
    - 내부망·관리자 전용으로 제한.
- slowlog:
  - 슬로우 쿼리 로그를 활성화하여
    - 비정상적인 DSL/검색 패턴을 조기에 파악한다.

### 10.3 API 게이트웨이·프록시

- JSON Schema 기반 유효성:
  - OpenAPI + JSON Schema를 기반으로
    - 필드·타입·최대 길이 등을 **게이트웨이 레벨에서 검증**.
- Rate Limit:
  - 사용자/토큰/엔드포인트별 레이트 리밋.
- 응답 캐시:
  - 민감 쿼리 결과는 캐시하지 않거나,
  - 사용자별 private 캐시로만 사용.

---

## 11. OpenAPI + JSON Schema 예

검색 API에 대한 스펙 예 (언어 무관):

```yaml
paths:
  /users:
    get:
      parameters:
        - in: query
          name: f
          schema:
            type: array
            items:
              type: object
              required: [field, op, value]
              properties:
                field:
                  type: string
                  enum: [email, age, status, tags]
                op:
                  type: string
                  enum: [eq, in, gt, gte, lt, lte, prefix]
                value: {}
        - in: query
          name: limit
          schema:
            type: integer
            minimum: 1
            maximum: 100
      responses:
        "200":
          description: ok
```

런타임에서 이 스펙을 기반으로 자동 검증을 수행하면,  
**스펙 밖의 필드/연산자/타입**이 들어오는 순간 **자동 400**을 반환하게 되어, 주입 표면을 크게 줄일 수 있다.

---

## 12. CI / SAST 방어선

### 12.1 정규식 탐지

Mongo 관련 위험 패턴:

- `findOne(req.body)`
- `find(req.query)`
- `collection.find(req.body)`

Elasticsearch 관련 위험 패턴:

- `q: req.`
- `search({ ..., q:`
- `script: req.`

간단한 SAST 단계 예:

```yaml
- name: SAST guard (grep)
  run: |
    set -e

    # Mongo: 요청 객체를 쿼리에 그대로 넘기는 패턴 차단
    if grep -R -nE "findOne\\(\\s*req\\.(body|query)" src/; then
      echo "::error title=Mongo risk::Do not pass raw request objects into queries"
      exit 1
    fi

    # ES: q=에 사용자 입력을 직접 넣는 패턴 차단
    if grep -R -nE "q:\\s*req\\." src/; then
      echo "::error title=ES risk::Do not use query string with user input"
      exit 1
    fi

    # ES: script에 사용자 입력을 직접 쓰는 패턴 차단
    if grep -R -nE "script\\s*:\\s*req\\." src/; then
      echo "::error title=ES risk::Do not build scripts from user input"
      exit 1
    fi
```

---

## 13. 체크리스트 (현장용)

- [ ] **스키마 검증**: 필드/타입/열거값 화이트리스트 정의
- [ ] **연산자/점 키 차단**: 입력 JSON 재귀 필터, `$`/`.` 키 거절
- [ ] **쿼리 빌더만 사용**: 허용 연산만 매핑, 정규식은 prefix 수준으로 제한
- [ ] **프로젝션 화이트리스트**: 민감 필드(비밀번호, 토큰 등) 응답에서 제외
- [ ] **페이지/사이즈 상한**: 전수 스캔 방지
- [ ] **Mongo `$where`/서버측 JS 금지**, ES `query_string`/`script`에 외부 입력 직접 연결 금지
- [ ] **로그/탐지**: 연산자 키 거절 카운트, 슬로우 쿼리/스캔 지표 모니터링
- [ ] **DB 권한 최소화**: 애플리케이션 계정에 최소 권한만 부여
- [ ] **OpenAPI/JSON Schema 검증**: 게이트웨이 레벨에서 파라미터 스펙 검증
- [ ] **CI/SAST 가드**: 위험 API 사용 패턴을 정규식으로 차단

---

## 14. 맺음말

NoSQL Injection의 본질은 **“입력이 연산자/DSL/스크립트로 해석되는 경계 붕괴”**다.  
MongoDB·Elasticsearch 모두 최신 버전에서 많은 보안 기능과 권장 설정을 제공하고 있지만, 애플리케이션 코드가 **입력 검증 없이 쿼리 객체를 조립**하는 순간, 여전히 고전적인 인젝션 취약점이 그대로 발생한다.

이 글에서 정리한 세 가지 축:

1. **스키마 검증(화이트리스트)**
2. **연산자 키 차단 (`$`/`.`)**
3. **전용 쿼리 빌더를 통한 안전한 DSL 생성**

을 일관되게 적용하면, MongoDB / Elasticsearch 기반 서비스에서도 **인젝션에 의한 인증 우회·대량 조회·데이터 노출 리스크**를 구조적으로 크게 줄일 수 있다.  
추가로, 로그/모니터링, 최소 권한, CI/SAST를 더해 **“초기 설계 → 운영 → 변경/배포” 전 주기에서 인젝션을 관리하는 체계**를 갖추는 것이 중요하다.