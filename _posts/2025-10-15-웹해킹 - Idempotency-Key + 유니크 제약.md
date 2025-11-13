---
layout: post
title: 웹해킹 - Idempotency-Key + 유니크 제약
date: 2025-10-15 14:25:23 +0900
category: 웹해킹
---
# 28. Idempotency-Key(결제/주문) + 유니크 제약
**— 문제정의 · 설계 원칙 · API 계약 · DB/캐시 스키마 · 트랜잭션/잠금 전략 · 다중 스택 예제(Express/Node + Postgres, Django/DRF + Postgres, Spring Boot + JPA) · Redis/Lua 원자화 · 실패/복구 · 테스트 시나리오 · 운영 체크리스트**

> **핵심 목표**
> 네트워크 재시도·사용자 중복 클릭·서버 타임아웃·멀티 인스턴스 경쟁 상황에서 **이중 청구/중복 주문을 방지**한다.
> 이 문서는 **Idempotency-Key + 유니크 제약**으로 “**같은 의도(同一 요청)**는 **단 한 번만 효과**가 발생”하도록 보장하는 **엔드-투-엔드 패턴**을 제시한다.

---

## 0. 한눈에 보기(Executive Summary)

- **Idempotency-Key**: 클라이언트가 **의도 단위**로 생성하는 **난수 토큰**(예: `Idempotency-Key: 6b3e-...`).
- **유니크 제약**: 서버/DB가 `(tenant|user, endpoint, key)`에 **유일성**을 강제.
- **행동**:
  1) **첫 요청** → 키를 **선점(claim)** 하고 **비즈니스 트랜잭션** 수행 → **결과 영속화 + 캐시**.
  2) **재요청(동일 키)** → **선행 결과 재사용**(동일 응답/코드/헤더).
  3) **바디 불일치**(같은 키 + 다른 입력) → **409 Conflict**.
- **레이스/장애**: **잠금(행 단위)** + **상태머신(Processing→Succeeded/Failed)** + **TTL/스위퍼**로 처리.

---

## 1. 문제 모델(왜 필요한가?)

- 사용자는 결제 버튼을 **두 번 클릭**한다.
- 모바일에서 네트워크가 **재전송**한다(HTTP 클라이언트의 **자동 재시도**).
- 서버가 **타임아웃**을 응답했지만, 내부 결제는 **성공**했다(“**손실된 성공 응답**” 문제).
- 서버가 **수평 확장**되어 서로 다른 인스턴스가 **동일 POST**를 **동시에** 처리한다.

> 비정상 재실행으로 **Order/Charge가 2개** 생기면 **이중 청구**가 발생한다.
> 해법은 “의도(하나의 결제/주문)를 식별”하고 “**그 의도는 단 1회만 성공**”하도록 **시스템 전체**를 설계하는 것.

---

## 2. 설계 원칙

1. **의도 스코프 정의**: 키의 유효범위를 **엔드포인트+주체**에 한정
   - 예: `(merchant_id, POST /v1/orders, idempotency_key)` 유니크.
2. **강한 유일성 보장**: **DB 유니크 인덱스** 또는 **원자적 KV(Lua)** 로 선점.
3. **응답 재생**: 선행 요청의 **status/body/headers**를 **그대로** 반환.
4. **요청 불변성**: 같은 키로 **다른 바디**가 오면 **409**.
5. **짧은 TTL + 스위퍼**: 캐시/행의 **만료/청소**.
6. **종단 일관성**: 결제/주문 레코드와 키 레코드를 **같은 트랜잭션**(또는 보상/정합 절차)으로 묶기.
7. **보안/남용 방지**: 키는 **난수(128bit+)**, 적절한 **레이트 리밋**.

---

## 3. API 계약(권장)

- **요청 헤더**
  - `Idempotency-Key: <base64url-128bit+>` (필수)
  - (옵션) `Idempotency-Key-TTL: 24h` — 서버가 해석할 수 있는 TTL 힌트
- **응답 헤더**
  - `Idempotency-Key: <same>` — 재생 응답에도 동일하게 첨부
  - `Idempotent-Replayed: true|false` — 재생 여부(디버그용)
- **상태 코드**
  - 첫 성공: `201 Created`(자원 생성) 또는 `200 OK`
  - 재생 성공: **처음과 동일 코드**(권장, 아니면 `200 OK` + `Idempotent-Replayed: true`)
  - 바디 불일치: `409 Conflict`
  - 처리중: `202 Accepted` (+ 폴링 URI), 또는 **요청을 대기**(짧은 타임아웃)
- **요청 바디 해싱**
  - 키와 함께 **요청 바디의 해시**(`SHA-256`)를 테이블에 저장하여 **불변성 검증**.

---

## 4. 데이터 모델 & 인덱스(예: PostgreSQL)

### 4.1 테이블

```sql
-- (1) 아이템포턴시 키 레지스트리
CREATE TABLE idempotency_keys (
  tenant_id        TEXT        NOT NULL,   -- 상점/조직 구분
  endpoint         TEXT        NOT NULL,   -- ex) POST:/v1/orders
  idempotency_key  TEXT        NOT NULL,
  request_hash     TEXT        NOT NULL,   -- SHA-256(정규화된 요청 바디)
  status           TEXT        NOT NULL CHECK (status IN ('PROCESSING','SUCCEEDED','FAILED')),
  response_code    INT,
  response_body    JSONB,                  -- 재생용 스냅샷
  response_headers JSONB,                  -- 필요시 화이트리스트 헤더만
  resource_id      TEXT,                   -- 생성된 주문/결제의 ID
  locked_at        TIMESTAMPTZ,
  locked_by        TEXT,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at       TIMESTAMPTZ              -- TTL
);

-- 유니크 제약: (tenant, endpoint, key)
CREATE UNIQUE INDEX uq_idem ON idempotency_keys(tenant_id, endpoint, idempotency_key);

-- 만료(청소) 보조 인덱스
CREATE INDEX idx_idem_exp ON idempotency_keys(expires_at);
```

> **왜 DB 보관?**
> “한 번만 성공”은 **영속 레벨**에서 보장해야 한다. 인메모리/단일 인스턴스 캐시는 노드가 바뀌면 깨진다.

### 4.2 상태 전이(State Machine)

```
CLAIM(INSERT .. ON CONFLICT)
  └─> PROCESSING (잠금 획득, 비즈니스 수행)
       ├─> SUCCEEDED (응답/리소스ID 스냅샷 저장)
       └─> FAILED    (실패 사유 기록; 재시도 정책)
```

- **중복 요청**: `SUCCEEDED` 라면 **스냅샷 재생**.
- **처리 중**: 동일 키가 들어오면 **대기/폴링 or 202**.

---

## 5. 트랜잭션/잠금 패턴

### 5.1 “선점 + 같은 트랜잭션에서 생성” (Postgres)

1) **선점**: `INSERT ... ON CONFLICT DO NOTHING`
2) **행 잠금**: `SELECT ... FOR UPDATE`(**SKIP LOCKED** 활용 가능)
3) **주문/결제 생성**과 **키 상태 업데이트**를 **동일 트랜잭션**으로 커밋

> 장점: **정합성 보장**. 장애 시 **부분 성공** 방지.

---

## 6. 구현 예시 A — **Node/Express + PostgreSQL**

### 6.1 헬퍼: 요청 바디 해시

```js
// lib/hash.js
import crypto from 'node:crypto';

export function normalizeAndHash(body) {
  // JSON 직렬화 표준화(키 정렬) → 동일 의미의 바디는 동일 해시
  const sorted = JSON.stringify(sortKeys(body));
  return crypto.createHash('sha256').update(sorted).digest('hex');
}

function sortKeys(obj) {
  if (Array.isArray(obj)) return obj.map(sortKeys);
  if (obj && typeof obj === 'object') {
    return Object.keys(obj).sort().reduce((acc, k) => {
      acc[k] = sortKeys(obj[k]); return acc;
    }, {});
  }
  return obj;
}
```

### 6.2 라우트: `POST /v1/orders`

```js
// routes/orders.js
import express from 'express';
import { normalizeAndHash } from '../lib/hash.js';
import pg from 'pg';

const router = express.Router();
const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL });

router.post('/v1/orders', async (req, res) => {
  const tenantId = req.headers['x-tenant-id'];   // 멀티테넌트 예시
  const idk = req.headers['idempotency-key'];
  if (!tenantId || !idk) return res.status(400).json({ error: 'missing headers' });

  const endpoint = 'POST:/v1/orders';
  const bodyHash = normalizeAndHash(req.body);
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    // 1) 선점 or 조회
    const insert = await client.query(
      `INSERT INTO idempotency_keys
         (tenant_id, endpoint, idempotency_key, request_hash, status, locked_at, locked_by, expires_at)
       VALUES ($1,$2,$3,$4,'PROCESSING', now(), $5, now() + interval '24 hours')
       ON CONFLICT (tenant_id, endpoint, idempotency_key) DO NOTHING`,
      [tenantId, endpoint, idk, bodyHash, process.pid.toString()]
    );

    let row;
    if (insert.rowCount === 0) {
      // 이미 존재 → 조회
      const r = await client.query(
        `SELECT * FROM idempotency_keys WHERE tenant_id=$1 AND endpoint=$2 AND idempotency_key=$3 FOR UPDATE`,
        [tenantId, endpoint, idk]
      );
      row = r.rows[0];

      // 2) 바디 해시 불일치 → 409
      if (row.request_hash !== bodyHash) {
        await client.query('ROLLBACK');
        return res.status(409).json({ error: 'body mismatch for idempotency key' });
      }

      // 3) 상태별 분기
      if (row.status === 'SUCCEEDED') {
        await client.query('COMMIT');
        res.set('Idempotency-Key', idk);
        res.set('Idempotent-Replayed', 'true');
        return res.status(row.response_code || 200).json(row.response_body || {});
      }

      if (row.status === 'PROCESSING') {
        // 동시 처리 → 짧게 대기하거나 202
        await client.query('ROLLBACK');
        return res.status(202).json({ status: 'processing' });
      }
      // FAILED → 정책에 따라 재시도 허용/거부
    } else {
      // 방금 선점됨 → 행을 FOR UPDATE로 확보
      const r = await client.query(
        `SELECT * FROM idempotency_keys WHERE tenant_id=$1 AND endpoint=$2 AND idempotency_key=$3 FOR UPDATE`,
        [tenantId, endpoint, idk]
      );
      row = r.rows[0];
    }

    // 4) 비즈니스 트랜잭션(주문 생성)
    const created = await client.query(
      `INSERT INTO orders (tenant_id, amount, currency, user_id, status)
       VALUES ($1,$2,$3,$4,'CREATED')
       RETURNING id, amount, currency, status`,
      [tenantId, req.body.amount, req.body.currency, req.body.user_id]
    );
    const order = created.rows[0];

    // 5) 응답 스냅샷 저장 + 상태 전이
    const responseBody = { order_id: order.id, status: order.status };
    await client.query(
      `UPDATE idempotency_keys
       SET status='SUCCEEDED',
           response_code=$1,
           response_body=$2::jsonb,
           response_headers=$3::jsonb,
           resource_id=$4,
           updated_at=now()
       WHERE tenant_id=$5 AND endpoint=$6 AND idempotency_key=$7`,
      [201, responseBody, { 'Content-Type': 'application/json' }, order.id, tenantId, endpoint, idk]
    );

    await client.query('COMMIT');
    res.set('Idempotency-Key', idk);
    return res.status(201).json(responseBody);

  } catch (e) {
    await client.query('ROLLBACK');
    // 실패 로그 후 FAILED 표시(선택)
    try {
      await client.query(
        `UPDATE idempotency_keys SET status='FAILED', updated_at=now()
         WHERE tenant_id=$1 AND endpoint=$2 AND idempotency_key=$3`,
        [tenantId, endpoint, idk]
      );
    } catch { /* noop */ }
    return res.status(500).json({ error: 'internal_error', detail: e.message });
  } finally {
    client.release();
  }
});

export default router;
```

> **특징**
> - **DB 유니크 인덱스**로 **중복 키** 원천 차단.
> - **요청 바디 해시**로 **입력 불변성** 검증.
> - **응답 스냅샷**을 저장하여 **재생**.
> - **PROCESSING→SUCCEEDED** 전이까지 **하나의 트랜잭션**.

---

## 7. 구현 예시 B — **Django/DRF + PostgreSQL**

### 7.1 모델

```python
# app/models.py
from django.db import models

class IdempotencyKey(models.Model):
    tenant_id       = models.CharField(max_length=64)
    endpoint        = models.CharField(max_length=128)
    idempotency_key = models.CharField(max_length=128)
    request_hash    = models.CharField(max_length=64)
    status          = models.CharField(max_length=16, choices=[('PROCESSING','PROCESSING'),('SUCCEEDED','SUCCEEDED'),('FAILED','FAILED')])
    response_code   = models.IntegerField(null=True)
    response_body   = models.JSONField(null=True)
    response_headers= models.JSONField(null=True)
    resource_id     = models.CharField(max_length=64, null=True)
    locked_at       = models.DateTimeField(null=True)
    locked_by       = models.CharField(max_length=64, null=True)
    created_at      = models.DateTimeField(auto_now_add=True)
    updated_at      = models.DateTimeField(auto_now=True)
    expires_at      = models.DateTimeField(null=True)

    class Meta:
        unique_together = ('tenant_id','endpoint','idempotency_key')
```

### 7.2 뷰

```python
# app/views.py
import hashlib, json, os, time
from django.db import transaction
from django.http import JsonResponse
from .models import IdempotencyKey

def hash_body(d):
    s = json.dumps(d, sort_keys=True, separators=(',',':'))
    return hashlib.sha256(s.encode()).hexdigest()

@transaction.atomic
def create_order(request):
    if request.method != 'POST':
        return JsonResponse({'error':'method_not_allowed'}, status=405)

    tenant = request.META.get('HTTP_X_TENANT_ID')
    idem   = request.META.get('HTTP_IDEMPOTENCY_KEY')
    if not tenant or not idem:
        return JsonResponse({'error':'missing_headers'}, status=400)

    body = json.loads(request.body or '{}')
    h = hash_body(body)
    endpoint = 'POST:/v1/orders'

    # 선점(없으면 생성)
    obj, created = IdempotencyKey.objects.get_or_create(
        tenant_id=tenant, endpoint=endpoint, idempotency_key=idem,
        defaults={'request_hash':h, 'status':'PROCESSING'}
    )

    if not created:
        if obj.request_hash != h:
            return JsonResponse({'error':'body_mismatch'}, status=409)
        if obj.status == 'SUCCEEDED':
            resp = JsonResponse(obj.response_body or {}, status=obj.response_code or 200)
            resp['Idempotency-Key'] = idem
            resp['Idempotent-Replayed'] = 'true'
            return resp
        if obj.status == 'PROCESSING':
            return JsonResponse({'status':'processing'}, status=202)

    # 비즈니스(예시)
    order_id = f"ord_{int(time.time())}"
    # … 실제로 Order 모델을 생성/저장 …

    # 스냅샷
    body_out = {'order_id': order_id, 'status':'CREATED'}
    obj.status = 'SUCCEEDED'
    obj.response_code = 201
    obj.response_body = body_out
    obj.resource_id   = order_id
    obj.save(update_fields=['status','response_code','response_body','resource_id','updated_at'])

    resp = JsonResponse(body_out, status=201)
    resp['Idempotency-Key'] = idem
    return resp
```

---

## 8. 구현 예시 C — **Spring Boot + JPA**

> 포인트: **`@Transactional`** 내에서 키 선점 → 비즈니스 → 스냅샷 저장.

```java
// entity
@Entity @Table(name="idempotency_keys",
  uniqueConstraints=@UniqueConstraint(columnNames={"tenantId","endpoint","idempotencyKey"}))
public class IdempotencyKeyEntity {
  @Id @GeneratedValue private Long id;
  private String tenantId;
  private String endpoint;
  private String idempotencyKey;
  private String requestHash;
  private String status;
  private Integer responseCode;
  @Lob private String responseBody; // JSON
  private String resourceId;
  // ... timestamps
}
```

```java
@Service
public class OrderService {
  @Autowired IdempotencyKeyRepo idemRepo;
  @Autowired OrderRepo orderRepo;

  @Transactional
  public ResponseEntity<String> create(String tenant, String idem, String bodyJson) {
    String hash = sha256(sortedJson(bodyJson));
    String ep = "POST:/v1/orders";

    Optional<IdempotencyKeyEntity> opt = idemRepo.findByTenantIdAndEndpointAndIdempotencyKeyForUpdate(tenant, ep, idem);
    IdempotencyKeyEntity key;
    if (opt.isEmpty()) {
      key = new IdempotencyKeyEntity();
      key.setTenantId(tenant); key.setEndpoint(ep);
      key.setIdempotencyKey(idem); key.setRequestHash(hash);
      key.setStatus("PROCESSING");
      idemRepo.saveAndFlush(key);
    } else {
      key = opt.get();
      if (!hash.equals(key.getRequestHash())) return ResponseEntity.status(409).body("{\"error\":\"body_mismatch\"}");
      if ("SUCCEEDED".equals(key.getStatus())) {
        return ResponseEntity.status(key.getResponseCode()).header("Idempotency-Key", idem).body(key.getResponseBody());
      }
      if ("PROCESSING".equals(key.getStatus())) {
        return ResponseEntity.status(202).body("{\"status\":\"processing\"}");
      }
    }

    // business
    Order o = new Order(); o.setStatus("CREATED"); orderRepo.save(o);

    // snapshot
    String out = "{\"order_id\":\""+o.getId()+"\",\"status\":\"CREATED\"}";
    key.setStatus("SUCCEEDED"); key.setResponseCode(201); key.setResponseBody(out); key.setResourceId(o.getId().toString());
    idemRepo.save(key);

    return ResponseEntity.status(201).header("Idempotency-Key", idem).body(out);
  }
}
```

> `…ForUpdate` 쿼리는 `@Query("select k from ... where ... for update")` 형태로 구현.

---

## 9. Redis + Lua(원자 선점) 패턴

> DB 대신 분산 KV를 쓸 때, **Lua 스크립트**로 “키 선점/검증/TTL 설정”을 **원자**로 처리.

```lua
-- claim_idempotency.lua
-- ARGV: request_hash, ttl_seconds, now_millis
-- KEYS[1] = idem:{tenant}:{endpoint}:{key}
local v = redis.call('HGETALL', KEYS[1])
if next(v) ~= nil then
  local m = {}
  for i=1,#v,2 do m[v[i]]=v[i+1] end
  if m['request_hash'] ~= ARGV[1] then
    return {err="MISMATCH"}
  end
  return {m['status'] or 'PROCESSING', m['response_code'] or '', m['response_body'] or ''}
else
  redis.call('HMSET', KEYS[1], 'request_hash', ARGV[1], 'status', 'PROCESSING', 'created_at', ARGV[3])
  redis.call('EXPIRE', KEYS[1], ARGV[2])
  return {'CLAIMED'}
end
```

Node 사용 예:
```js
const key = `idem:${tenant}:${endpoint}:${idk}`;
const h = normalizeAndHash(req.body);
const r = await redis.evalsha(SHA1, 1, key, h, 86400, Date.now());
if (Array.isArray(r) && r[1] === 'MISMATCH') return 409;
// 'CLAIMED' → 비즈니스 수행 후 HSET status='SUCCEEDED', response_*
```

> **주의**: 최종 정합성은 **영속 저장소(DB)** 에 기록해야 한다(주문 생성 등). Redis는 **선점/중간상태** 용도로 적합.

---

## 10. 결제 게이트웨이 연동

- Stripe 등 일부 PG는 자체 **Idempotency-Key**를 지원.
- **서버 내부 키**와 **PG 키**를 **둘 다** 사용하라.
  - 서버: **내부 DB 정합** 보장
  - PG: **외부 결제 재실행 방지**
- 매핑 테이블 예: `(tenant, local_key) → (pg_key, charge_id)`

---

## 11. 실패·타임아웃·충돌 처리

- **서버 타임아웃** 후 내부에서 성공했을 수 있다 → **재시도는 동일 키**로 오게 하라(클라이언트 SDK 가이드).
- **PROCESSING 장기화**: `locked_at` + **워치독**(예: 2분) 초과 시 **FAILED**로 전이 or **해제** 후 재시도 허용.
- **409 Conflict**: 동일 키로 **다른 바디** → 클라이언트 버그/남용. 로그/알람.
- **서버 다운 중**: 트랜잭션 경계 내 쓰기가 완결되지 않았다면, 다음 호출이 **다시 선점**하여 진행.
- **TTL**: 결과 캐시를 **업무 맥락**에 맞게(예: 24시간~7일) 보관 후 파기.

---

## 12. 테스트 시나리오(자동화 권장)

1) **더블 클릭**: 10ms 간격으로 동일 키 2회 → **주문 1개**만 생성, 1회는 **재생**.
2) **서버 지연**: 첫 요청 2초 슬립 중, 동일 키 재호출 → **202** 또는 대기 후 동일 결과.
3) **바디 불일치**: 동일 키 + 금액만 다름 → **409**.
4) **노드 A/B 동시**: 서로 다른 인스턴스에서 동시 INSERT → **유니크 충돌**을 1개가 이기고, 다른 1개는 **재생 루트**로.
5) **TTL 경과**: TTL 지난 키는 재사용 불가(권장) 또는 재사용시 신규 의도(정책 선택).
6) **PG 재시도**: 외부 결제 API 429/500 재시도 → PG의 idem키와 서버 idem키 모두 유지 → 이중 과금 없음.

---

## 13. 운영 & 모니터링

- **지표**:
  - `idem.claimed.count`, `idem.replayed.count`, `idem.mismatch.count`, `idem.processing.timeout.count`
  - 평균 처리시간, 202 비율
- **로그**: 키/테넌트/엔드포인트/바디해시/리소스ID/최종코드
- **청소잡**: `expires_at < now()` 삭제(배치)
- **보안**: 키는 **예측 불가**(128bit+). **고정 문자열** 금지. 필요시 **HMAC(user_id + nonce)**.

---

## 14. 클라이언트 가이드(UX)

- **액션당 1개 키** 생성(버튼 클릭 시 즉시 생성).
- 실패/시간초과가 나더라도 **같은 키로 재시도**.
- 성공 응답을 받으면 **키 폐기**(앱 로컬 저장소에서 삭제).
- 결제/주문 화면 **중복 제출 방지 UI**(스피너/버튼 비활성화)와 함께 사용.

---

## 15. 수학적 관점(직관)

> **멱등성(idempotence)**: 연산 \( f \) 에 대해
> $$ f(f(x)) = f(x) $$
> 서버에서 “같은 의도”는 \( x \) 를 식별하는 **키**로 모델링한다.
> **Idempotency-Key**는 “**의도의 동일성**”을 표현하고, **유니크 제약**은 \( f \) 가 **한 번만 효과**를 갖도록 보장한다.

---

## 16. “복붙” 최소 구현 템플릿(Express + Postgres)

```sql
-- migration
CREATE TABLE orders (
  id TEXT PRIMARY KEY DEFAULT concat('ord_', floor(extract(epoch from now())*1000)::text),
  tenant_id TEXT NOT NULL,
  amount NUMERIC(12,2) NOT NULL,
  currency TEXT NOT NULL,
  user_id TEXT NOT NULL,
  status TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```js
// app.js
import express from 'express';
import bodyParser from 'body-parser';
import orders from './routes/orders.js';

const app = express();
app.use(bodyParser.json());
app.use(orders);
app.listen(3000);
```

> 위 템플릿으로 **즉시 테스트 가능한 최소 골격**이 완성된다.

---

## 17. 체크리스트

- [ ] `Idempotency-Key` 필수 + **난수성**(128bit+)
- [ ] `(tenant, endpoint, key)` **유니크 인덱스**
- [ ] **요청 바디 해시** 저장 및 **불일치=409**
- [ ] **응답 스냅샷 재생**(status/body/headers)
- [ ] **트랜잭션 내** 키상태 전이 + 리소스 생성
- [ ] **PROCESSING 타임아웃**/워치독/재시도
- [ ] **TTL/청소** 잡
- [ ] **멀티 인스턴스 경쟁** 테스트
- [ ] **PG(IdP) 자체 idem-key** 병행(있다면)
- [ ] **지표/로그/알람** 탑재

---

### 마무리
Idempotency-Key + 유니크 제약은 **간단하지만 강력한 방패**다.
핵심은 **의도 식별 → 강한 유일성 → 응답 재생 → 일관된 상태 전이**이며,
DB 트랜잭션과 결합하면 **클러스터/재시도/장애**에도 **이중 청구 없는** 견고한 결제/주문 시스템을 만들 수 있다.
당신의 스택/PG(Stripe, PayPal, Toss 등)/DB(Cloud SQL, Aurora, AlloyDB…) 정보를 알려주면,
**완전한 마이그레이션 플랜과 마이그레이션 스크립트**까지 맞춤 제작해줄게.
