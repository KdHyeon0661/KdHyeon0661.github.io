---
layout: post
title: 웹해킹 - NoSQL Injection
date: 2025-09-30 21:25:23 +0900
category: 웹해킹
---
# 🧪 5. NoSQL Injection (MongoDB / Elasticsearch)

## 0. 요약 (Executive Summary)

- **문제(패턴)**  
  - **MongoDB**: 애플리케이션이 사용자 JSON을 **그대로** 쿼리에 바인딩하면, 키로 시작하는 **연산자(`$ne`, `$gt`, `$regex`, `$where` 등)** 가 주입되어 **인증 우회 / 대량 조회 / 서버측 JS 평가** 위험이 발생.  
  - **Elasticsearch**: `q=` 같은 **Query String**에 사용자 입력을 그대로 넣거나, **스크립트/템플릿**에 사용자가 관여하면 **DSL/스크립트 주입**, **필터 우회 / 대량 스캔** 발생.

- **핵심 방어 전략**  
  1) **스키마 검증(화이트리스트)**: 필드/타입/길이/열거값을 **명시**하고 벗어나면 **거절**.  
  2) **연산자 키 차단/정규화**: 입력 JSON의 **키가 `$`/`.` 포함** 시 **거절/무효화**.  
  3) **쿼리 빌더만 사용**: “입력 → (검증/정규화) → **파라미터 빌더** → 쿼리” 흐름 고정.  
  4) **민감 API 보호**: 인증 엔드포인트/관리 검색은 **레이트 리밋·페이지 제한·프로젝션 제한**.  
  5) **로그/탐지**: 연산자 키 탐지, 과도한 스캔·전수 매칭 패턴 알림.

---

# 1. 위협 모델과 발생 지점

### 1.1 MongoDB
- **안티패턴**:  
  - `User.findOne(req.body)` 처럼 **요청 바디 전체**를 조건으로 사용.  
  - `collection.find({ name: input })` 에서 `input`이 **객체**로 들어올 수 있게 허용.  
  - `$where` / 서버측 JS 사용, **임의 코드/표현식 실행**.  
  - `$regex` 의 무제한 사용으로 **전수 스캔/성능 고갈**.

### 1.2 Elasticsearch
- **안티패턴**:  
  - `GET /_search?q=<사용자입력>` (Query String Query)  
  - `script_score`, `painless script` 등에 사용자 문자열 직삽입  
  - `wildcard`, `regexp` 남용으로 대량 스캔, `source` 전체 노출

> 공통 본질: **데이터**로 취급했어야 할 입력이 **연산자/스크립트/DSL**로 해석되는 순간이 위험.

---

# 2. 안전한 재현/탐지 (스테이징 전용, 차단 확인용)

> 아래는 “**우리가 차단에 성공하는지**” 확인하기 위한 **안전 테스트**입니다. (운영/타 시스템 금지)

### 2.1 “연산자 키”가 들어오면 **거절** 기대 (Mongo)
```js
// test/opkey-guard.test.js — 차단 스모크
import assert from "node:assert";
import { sanitizeDocument } from "../src/security/mongo-sanitize.js";

assert.throws(() => sanitizeDocument({ username: "u", password: { $ne: "" } }), /operator/i);
assert.throws(() => sanitizeDocument({ "profile.pic": "x" }), /dot key/i);
console.log("✅ operator/dot key blocked");
```

### 2.2 ES Query String 차단 확인
```bash
# 프록시/게이트웨이 앞단에서 q= 차단 기대
curl -si 'https://es.staging.example.com/my-index/_search?q=anything' | head -n1
# 기대: 400/403 또는 정책 안내
```

---

# 3. Node.js(Express) + Mongo(Mongoose/PyMongo 유사) — **차단/정규화 미들웨어**

## 3.1 “연산자/점(.) 키” 재귀 필터
```js
// src/security/mongo-sanitize.js
export function sanitizeDocument(input, { allowOperators = false } = {}) {
  if (input == null) return input;
  if (typeof input !== "object") return input;

  // 배열: 각 요소 재귀
  if (Array.isArray(input)) return input.map(v => sanitizeDocument(v, { allowOperators }));

  const out = {};
  for (const [k, v] of Object.entries(input)) {
    if (k.includes(".")) throw new Error("dot key blocked");            // 필수
    if (k.startsWith("$") && !allowOperators) throw new Error("operator key blocked");

    // 문자열 길이/유니코드 정규화/트림(선택)
    out[k] = sanitizeDocument(v, { allowOperators });
  }
  return out;
}
```

> **원칙**: **애플리케이션 입력**에서는 `$`/`.` 키를 **절대 허용하지 않음**.  
> 내부적으로 우리가 만든 **쿼리 빌더가 생성하는 경우**에만 일부 `$` 허용(아래 §3.3).

## 3.2 인증 예제(안전 설계)
```js
// ❌ 나쁜 예: User.findOne(req.body) — 주입 허용
// ✅ 좋은 예: 허용 필드만 추출 + 평문 비교 금지(해시)
import bcrypt from "bcrypt";

app.post("/login", async (req, res) => {
  const { email, password } = req.body ?? {};
  if (typeof email !== "string" || typeof password !== "string") {
    return res.status(400).json({ error: "invalid" });
  }

  // 이메일 정규화
  const emailNorm = email.trim().toLowerCase();

  // 찾기(항상 단일 필드)
  const user = await User.findOne({ email: emailNorm }).lean();
  if (!user) return res.status(401).json({ error: "auth" });

  const ok = await bcrypt.compare(password, user.passwordHash);
  if (!ok) return res.status(401).json({ error: "auth" });

  // 세션 발급 …
  res.sendStatus(204);
});
```

## 3.3 **쿼리 빌더**(화이트리스트 필드/연산자만)
```js
// src/security/mongo-query-builder.js
const ALLOWED_FIELDS = {
  email:  "string",
  age:    "number",
  status: ["active","inactive","pending"], // enum
  tags:   "string[]"
};
const ALLOWED_OPS = new Set(["eq","in","gt","gte","lt","lte","prefix"]); // 우리가 정의

function validateField(field) {
  if (!Object.hasOwn(ALLOWED_FIELDS, field)) throw new Error("field not allowed");
  return field;
}
function coerce(field, value) {
  const t = ALLOWED_FIELDS[field];
  if (Array.isArray(t)) { if (!t.includes(value)) throw new Error("enum"); return value; }
  if (t === "number") {
    const n = Number(value); if (!Number.isFinite(n)) throw new Error("num"); return n;
  }
  if (t === "string[]") {
    if (!Array.isArray(value) || value.some(v => typeof v !== "string")) throw new Error("arr");
    return value.map(v => v.trim());
  }
  if (t === "string") { if (typeof value !== "string") throw new Error("str"); return value.trim(); }
  throw new Error("type");
}

export function buildMongoQuery(filters = []) {
  // filters: [{ field, op, value }]
  const query = {};
  for (const f of filters) {
    const field = validateField(String(f.field || ""));
    const op    = String(f.op || "");
    if (!ALLOWED_OPS.has(op)) throw new Error("op not allowed");

    const val = coerce(field, f.value);

    // 내부에서만 연산자 사용 → 외부 입력에서는 `$` 금지
    if (op === "eq")       query[field] = val;
    else if (op === "in")  query[field] = { $in: Array.isArray(val) ? val : [val] };
    else if (op === "gt")  query[field] = { $gt: val };
    else if (op === "gte") query[field] = { $gte: val };
    else if (op === "lt")  query[field] = { $lt: val };
    else if (op === "lte") query[field] = { $lte: val };
    else if (op === "prefix") {
      // 정규식 남용 금지 → prefix 전용 앵커 + 길이 제한
      const s = String(val);
      if (s.length > 40) throw new Error("too long");
      query[field] = { $regex: new RegExp("^" + escapeRegex(s)) };
    }
  }
  return query;
}
function escapeRegex(s){ return s.replace(/[-\/\\^$*+?.()|[\]{}]/g, "\\$&"); }
```

### 라우트에서 사용
```js
app.get("/users", async (req,res) => {
  // 클라이언트는 우리가 정의한 필드/OP만 전달
  // 예: ?f[0].field=email&f[0].op=eq&f[0].value=a@b.com
  const filters = Array.isArray(req.query.f) ? req.query.f : [];
  let query;
  try {
    // 1) 외부 입력 전역 차단
    sanitizeDocument(req.query); // throws if $/.
    // 2) 허용된 스키마/OP만 빌드
    query = buildMongoQuery(filters);
  } catch (e) {
    return res.status(400).json({ error: "bad_filter" });
  }

  const limit = Math.min(Number(req.query.limit || 20), 100); // 상한
  const page  = Math.max(Number(req.query.page || 1), 1);
  const skip  = (page - 1) * limit;

  const rows = await User.find(query, { email:1, age:1, status:1 }) // projection 화이트리스트
                         .sort({ _id: -1 }).skip(skip).limit(limit).lean();
  res.json({ rows, page, limit });
});
```

> 포인트  
> - **외부 입력에서는 `$`/`.` 금지**(sanitize).  
> - **쿼리 생성은 항상 내부 빌더**로만.  
> - 정규식은 **prefix 전용** 등 **표현력 제한**.  
> - **프로젝션 화이트리스트**로 민감 필드 노출 금지.

---

# 4. Python(FastAPI/Flask) + PyMongo — 동일 원칙

```python
# security/mongo_sanitize.py
def sanitize_document(obj, allow_ops=False):
    if obj is None or not isinstance(obj, (dict, list)): return obj
    if isinstance(obj, list):
        return [sanitize_document(x, allow_ops) for x in obj]
    out = {}
    for k,v in obj.items():
        if "." in k: raise ValueError("dot key")
        if k.startswith("$") and not allow_ops: raise ValueError("op key")
        out[k] = sanitize_document(v, allow_ops)
    return out
```

```python
# query builder (간략)
ALLOWED_FIELDS = {"email":"str","age":"num","status":{"enum":["active","inactive","pending"]}}
ALLOWED_OPS = {"eq","in","gt","gte","lt","lte","prefix"}

def build_query(filters):
    q = {}
    for f in filters:
        field = f.get("field"); op = f.get("op"); val = f.get("value")
        if field not in ALLOWED_FIELDS or op not in ALLOWED_OPS: raise ValueError("bad")
        # … 타입 강제(coerce) …
        if op=="eq": q[field]=val
        elif op=="in": q[field]={"$in":val}
        elif op=="prefix":
            s = str(val)
            if len(s)>40: raise ValueError("long")
            q[field]={"$regex": f"^{re.escape(s)}"}
        # gt/gte/lt/lte 유사
    return q
```

---

# 5. Mongoose 스키마/옵션으로 **추가 방어**

```js
const UserSchema = new Schema({
  email: { type: String, index: true, required: true, lowercase: true, trim: true },
  passwordHash: { type: String, required: true },
  age: { type: Number, min: 0, max: 150 },
  status: { type: String, enum: ["active","inactive","pending"], default: "active" }
}, {
  strict: "throw",               // 스키마 밖 필드는 예외
  minimize: true,                // 빈 객체 저장 방지
  versionKey: false
});

// $where 사용 금지(쿼리 수준에서는 우리가 사용 안함)
// 정규식 인덱스/성능: prefix만 허용(빌더에서 보장)
```

---

# 6. MongoDB Aggregation — **화이트리스트 파이프라인**만 허용

```js
// 안전한 aggregation 빌더 예시(정렬/집계 몇 가지만 허용)
const AGG_ALLOW = new Set(["match","group","sort","limit","project"]);
function buildAgg(specs=[]){
  const pipe = [];
  for (const s of specs) {
    if (!AGG_ALLOW.has(s.type)) throw new Error("agg not allowed");
    if (s.type === "match") pipe.push({ $match: buildMongoQuery(s.filters) });
    else if (s.type === "group") {
      // 허용 필드만 그룹키로
      const field = String(s.field||"");
      if (!ALLOWED_FIELDS[field]) throw new Error("field");
      pipe.push({ $group: { _id: `$${field}`, count: { $sum: 1 } } });
    } else if (s.type === "sort") {
      const field = String(s.field||""); const dir = s.dir === "asc" ? 1 : -1;
      if (!ALLOWED_FIELDS[field]) throw new Error("field"); pipe.push({ $sort: { [field]: dir } });
    } else if (s.type === "limit") {
      const n = Math.min(Number(s.n||20), 200); pipe.push({ $limit: n });
    } else if (s.type === "project") {
      // 화이트리스트 프로젝션
      const proj = {}; for (const f of s.fields||[]) if (ALLOWED_FIELDS[f]) proj[f]=1;
      pipe.push({ $project: proj });
    }
  }
  return pipe;
}
```

> **주의**: 저장 프로시저·서버측 JS·`$where` 등 **표현식 기반**은 **기능적으로 제거**.

---

# 7. Elasticsearch — 안전 설계 가이드 & 코드

## 7.1 **Query String** 사용 금지, **Query DSL**만
```js
// ❌ 나쁜 예
// client.search({ index, q: req.query.q });

// ✅ 좋은 예: 우리가 정의한 DSL 빌더만
import { Client } from "@elastic/elasticsearch";
const es = new Client({ node: process.env.ES_URL });

const ES_FIELDS = {
  "name.keyword": "keyword",
  "email.keyword": "keyword",
  "age": "long",
  "status": "keyword"
};
const ES_OPS = new Set(["term","terms","range","prefix"]); // 제한된 연산만

function buildEsQuery(filters=[]) {
  const must = [];
  for (const f of filters) {
    const field = String(f.field||"");
    const op    = String(f.op||"");
    const val   = f.value;

    if (!Object.hasOwn(ES_FIELDS, field)) throw new Error("field");
    if (!ES_OPS.has(op)) throw new Error("op");

    if (op === "term")  must.push({ term: { [field]: val } });
    if (op === "terms") must.push({ terms: { [field]: Array.isArray(val)?val:[val] } });
    if (op === "prefix"){
      const s = String(val); if (s.length > 30) throw new Error("long");
      must.push({ prefix: { [field]: s.toLowerCase() } });   // text analyzer 회피: keyword에만
    }
    if (op === "range") {
      const r = {};
      if (val.gte != null) r.gte = Number(val.gte);
      if (val.lte != null) r.lte = Number(val.lte);
      must.push({ range: { [field]: r } });
    }
  }
  return { bool: { must } };
}
```

## 7.2 검색 API
```js
app.get("/search", async (req, res) => {
  try {
    // 외부 입력 정규화/차단(여기선 쿼리 스트링 전체 검사)
    if ("q" in req.query) return res.status(400).json({ error: "use filters" });

    const filters = Array.isArray(req.query.f) ? req.query.f : [];
    const query = buildEsQuery(filters);

    const size = Math.min(Number(req.query.size || 20), 100);
    const from = Math.max(Number(req.query.from || 0), 0);

    const r = await es.search({
      index: "users",
      query,
      size,
      from,
      track_total_hits: false,  // 대량 카운트 방지
      _source: ["name","email","status"] // 프로젝션 화이트리스트
    });
    res.json(r.hits.hits.map(h => ({ id: h._id, ...h._source })));
  } catch (e) {
    res.status(400).json({ error: "bad_filter" });
  }
});
```

## 7.3 **금지/제한** 권고
- `query_string`, `simple_query_string` **금지**(또는 내부 전용으로만).  
- `script`, `script_score`, `painless` 등 **사용자 입력 직결 금지**. 필요 시 **서버 저장 템플릿 + 바인딩 파라미터만**.  
- `wildcard`, `regexp` 는 `keyword` 필드에 **길이 제한** + 관리자 전용.  
- `_source` 전체 반환 금지(필드 화이트리스트).  
- 인덱스 수준에서 **필드 타입 통일**(`keyword` vs `text`)로 예측 가능한 쿼리만 허용.

## 7.4 프록시에서 Query String 차단(선택)
```nginx
# Nginx 앞단에서 q= 를 아예 받지 않게
location ~ ^/.+/_search$ {
  if ($arg_q) { return 400; }
  proxy_pass http://es:9200;
}
```

---

# 8. 로깅/탐지/모니터링

## 8.1 애플리케이션 로그(구조화)
- 필드: `ts`, `route`, `user_id`, `filters_count`, `rejected_reason(opkey|dotkey|badfield)`, `limit`, `source=api|ui`  
- **연산자 키 탐지**: 입력 JSON 스캔 중 `$`/`.` **감지 횟수** 지표화

```js
logger.warn({
  event: "reject_filter",
  reason: "operator_key",
  src_ip, user_id, route: "/users"
});
```

## 8.2 SIEM 룰 예시

**Loki(LogQL)**
```logql
{service="web", event="reject_filter"} | json | reason="operator_key"
| count_over_time(5m) > 5
```

**Splunk(SPL)**
```spl
index=app event=reject_filter reason IN ("operator_key","dot_key","bad_field")
| timechart span=5m count by reason
```

## 8.3 성능/남용 탐지
- **Mongo**: collection scan 증가, `$regex` 비인덱스 경로 비율 상승  
- **ES**: `search.total` 급증, `query_cache` 미스 폭증, 너무 큰 `size/from` 반복

---

# 9. 구성/플랫폼 수준 수칙

- **MongoDB**
  - 앱에서 `$where` **사용 금지**. (서버측 JS 의존 로직 제거)  
  - **권한 최소화**: 앱 계정에 `read`/필요 최소 권한만, `admin` 금지.  
  - **인덱스 설계**: 허용한 질의 패턴(prefix/term/range)에 맞추어 생성.  
  - **서버/드라이버 최신화**: 드라이버가 **키 필터/컬레이스** 기능을 제공하는 경우 활성화.

- **Elasticsearch**
  - 인덱스 매핑에서 **keyword vs text** 명확화(검색 동작 예측).  
  - `_scripts` 등 관리 API는 **내부 네트워크/관리자만**.  
  - slowlog 활성화로 비정상 패턴 조기 탐지.

- **API 게이트웨이/프록시**  
  - **바디 스키마 검증**(JSON Schema) — OpenAPI + 유효성 미들웨어  
  - **Rate limit**: 엔드포인트/사용자/토큰 별  
  - **응답 캐시** 금지(민감 쿼리), **압축** 제한(압축 폭탄 방지)

---

# 10. OpenAPI + JSON Schema 검증 (언어 무관 패턴)

```yaml
# openapi.yaml (일부) — /users 검색
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
                field: { type: string, enum: [email, age, status, tags] }
                op:    { type: string, enum: [eq, in, gt, gte, lt, lte, prefix] }
                value: {}
        - in: query
          name: limit
          schema: { type: integer, minimum: 1, maximum: 100 }
      responses:
        "200": { description: ok }
```

> 런타임에서 **스펙 기반 검증**을 통과하지 못하면 **자동 400** → 주입 표면 축소.

---

# 11. CI/SAST 방어선

- **정규식/grep 룰 예시**
  - Mongo: `findOne\(\s*req\.body` / `find\(\s*req\.query`  
  - ES: `q:\s*req\.` / `search\(\{[^\}]*q:` / `script:\s*req\.`
- **PR 가드**: 위험 API 호출 감지 시 파이프라인 실패
```yaml
- name: SAST guard (grep)
  run: |
    set -e
    if grep -R -nE "findOne\\(\\s*req\\.(body|query)" src/; then
      echo "::error title=Mongo risk::Do not pass raw request objects into queries"; exit 1; fi
    if grep -R -nE "q:\\s*req\\." src/; then
      echo "::error title=ES risk::Do not use query string with user input"; exit 1; fi
```

---

# 12. 체크리스트 (현장용)

- [ ] **스키마 검증**: 필드/타입/열거값 화이트리스트  
- [ ] **연산자/점 키 차단**: 입력 JSON 재귀 필터, `$`/`.` 거절  
- [ ] **쿼리 빌더**만 사용: 허용 연산만 매핑, 정규식은 **prefix** 수준으로 제한  
- [ ] **프로젝션 화이트리스트**, 민감 필드 제외  
- [ ] **페이지/사이즈 상한**, 전수 스캔 방지  
- [ ] **`$where`/서버측 JS 금지**, ES **query_string/script** 외부입력 금지  
- [ ] **로그/탐지**: 연산자 키 거절 카운트, 스캔 지표, slowlog  
- [ ] **권한 최소화**(DB 계정/관리 API), 인덱스 설계 일치  
- [ ] **OpenAPI/JSON Schema**로 게이트웨이 레벨 유효성  
- [ ] **CI/SAST**로 위험 패턴 차단

---

## 맺음말
NoSQL 인젝션의 핵심은 **“입력이 연산자/DSL로 해석되는 경계 붕괴”**입니다.  
**스키마 검증 → 연산자 키 차단 → 전용 빌더로만 쿼리 생성**이라는 3단계 가드를 적용하면,  
MongoDB/Elasticsearch 모두에서 **우회·대량 조회** 리스크를 구조적으로 줄일 수 있습니다.