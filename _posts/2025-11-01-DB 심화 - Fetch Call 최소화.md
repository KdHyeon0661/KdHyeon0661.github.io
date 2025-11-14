---
layout: post
title: DB 심화 - Fetch Call 최소화
date: 2025-11-01 20:25:23 +0900
category: DB 심화
---
# Fetch Call 최소화 — **부분범위처리(Stopkey) × Array Fetch**로 왕복·I/O·대기를 줄이는 방법

> **핵심 요약**
> - **Fetch call**은 SELECT 결과를 클라이언트로 가져오는 **왕복(round-trip)** 이다.
> - **부분범위처리(PRP, Partial Range Processing; STOPKEY)** 로 **필요한 상위 N행만** 읽으면 **I/O 자체가 작아진다**.
> - **ArraySize(배열 페치 크기)** 를 키우면 **Fetch 왕복 횟수**가 $$\left\lceil \frac{\text{rows}}{\text{ArraySize}} \right\rceil$$ 로 감소 → 네트워크/컨텍스트 스위칭/호출 오버헤드가 크게 줄어든다.
> - OLTP에서는 “**짧고 자주**” 모델이므로, **PRP + Array Fetch** 만으로도 p95 지연이 폭발적으로 개선되는 경우가 많다.

---

## 실습 데이터 준비

```sql
-- 대용량 샘플 테이블
DROP TABLE orders PURGE;

CREATE TABLE orders (
  order_id     NUMBER PRIMARY KEY,
  customer_id  NUMBER NOT NULL,
  order_dt     DATE   NOT NULL,
  status       VARCHAR2(10) NOT NULL,
  amount       NUMBER(12,2) NOT NULL
);

INSERT /*+ APPEND */ INTO orders
SELECT level,
       MOD(level, 500000) + 1,
       (TRUNC(SYSDATE) - MOD(level, 365)) + (MOD(level, 86400) / 86400),
       CASE WHEN MOD(level, 11)=0 THEN 'CANCEL' ELSE 'OK' END,
       ROUND(DBMS_RANDOM.VALUE(10, 1000), 2)
FROM dual CONNECT BY level <= 2000000;
COMMIT;

-- PRP(상위 N)와 필터를 지원하는 인덱스
CREATE INDEX ix_orders_cust_dt  ON orders(customer_id, order_dt DESC);
CREATE INDEX ix_orders_status   ON orders(status);
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'ORDERS', cascade=>TRUE);
END;
/
```

---

## Fetch Call과 실행 단계의 관계(짧은 복습)

- **Parse**: 파싱/최적화/권한 → 보통 1회
- **Execute**: 실행계획 실행 시작 → 보통 1회
- **Fetch**: 결과를 **배열 단위**로 전송 → **여러 회**(ArraySize에 의해 결정)

### Fetch 왕복 수 근사

$$
N_{\text{fetch calls}} \approx \left\lceil \frac{\text{전송할 행 수}}{\text{ArraySize}} \right\rceil
$$

- **ArraySize↑** → Fetch call↓ → **네트워크 RTT 누적 감소**
- 같은 행 수를 가져오더라도 왕복 수가 줄어 **경과시간(Elapsed)** 이 줄어든다.

> ※ **중요 구분**:
> - **PRP(Stopkey)** 는 **가져올 행 수 자체**를 줄여 I/O를 줄인다.
> - **ArraySize** 는 **왕복 횟수**를 줄여 네트워크/호출 오버헤드를 줄인다.
> → **둘을 함께 적용**하면 효과가 곱해진다.

---

## 부분범위처리(PRP; Stopkey) 원리

### 개념

- “전체를 끝까지 읽고 **그중 일부만** 쓰는” 대신, **필요한 앞부분만** 읽고 **즉시 멈춘다**.
- Oracle 실행계획에는 **`STOPKEY`** 연산자로 표현되며, 보통 다음과 결합한다.
  - `FETCH FIRST N ROWS ONLY` (또는 `ROWNUM <= :N`)
  - 인덱스가 **필터/정렬 순서**를 만족 → **Index Range Scan + STOPKEY**
  - 가능하면 **커버링 인덱스**(필요 컬럼이 인덱스에 모두 포함)로 **Table Access BY ROWID** 조차 회피

### PRP가 잘 되는 조건

1) **WHERE + ORDER BY** 가 **같은 복합 인덱스**로 **SARG** 가능
2) 정렬이 인덱스 순서와 일치 (예: `order_dt DESC` 인덱스, `ORDER BY order_dt DESC`)
3) 필요한 컬럼이 인덱스에 **다 있으면** 최고 (커버링)
4) TOP-N, 페이지네이션(**Keyset** 방식: `WHERE (order_dt, order_id) < (:last_dt, :last_id)`)

### PRP 예제: 고객의 최신 주문 상위 20건만

```sql
-- “APAC 고객 123의 최신 주문 20건”
SELECT /*+ index(o ix_orders_cust_dt) */
       o.order_id, o.order_dt, o.amount, o.status
FROM   orders o
WHERE  o.customer_id = :cust
ORDER  BY o.order_dt DESC
FETCH FIRST 20 ROWS ONLY;  -- STOPKEY
```

**좋은 점**
- `customer_id = :cust` 로 **선택도 쿼리**
- `order by order_dt desc` 를 복합 인덱스의 **정렬 순서**로 해결
- **상위 20건**만 읽으면 나머지 범위는 **탐색하지 않음** → **I/O 급감**

> **실행계획(요지)**
> - `INDEX RANGE SCAN ix_orders_cust_dt` (Predicate: `customer_id=:cust`)
> - `STOPKEY` (Fetch First 20)
> - 필요 컬럼이 인덱스에 있지 않으면 `TABLE ACCESS BY ROWID` 로 보강

### (반례) PRP가 깨지는 경우

```sql
-- 나쁜 예: 정렬이 인덱스로 해결되지 않음 → 전체 정렬 후 상위 N
SELECT order_id, order_dt, amount
FROM   orders
WHERE  customer_id = :cust
ORDER  BY amount DESC
FETCH FIRST 20 ROWS ONLY;
```
- 인덱스가 `(customer_id, order_dt)` 뿐이라면 `amount DESC` 정렬을 위해 **SORT** 필요
- 보통 **대부분을 읽고** 정렬해야 하므로 **STOPKEY 이점 상실**
- **해결**: `(customer_id, amount DESC)` 인덱스(업무상 정렬 의미가 있을 때만) 또는 요구사항을 **시간순**으로 변경

---

## OLTP 환경에서 PRP가 성능을 끌어올리는 원리

OLTP는 **짧은 쿼리**가 **매우 자주** 발생한다. PRP는 아래 효과를 합산해 **p95/최대 응답시간**을 낮춘다.

1) **I/O 감소**: 상위 N만 읽으므로 **읽을 블록 자체가 적다**
2) **락 경합/래치/뮤텍스 감소**: 적은 읽기는 적은 경합
3) **CPU/메모리 압력 완화**: 정렬·해시·필터가 작아짐
4) **캐시 친화성**: 인덱스 스캔의 **짧은 범위**만 탐색 → 버퍼 캐시 히트율에 유리
5) **사용자 체감 개선**: 필요한 정보(예: 목록 첫 페이지)만 빠르게 반환

> 정리하면, PRP는 **근본적인 작업량 자체**를 줄이는 기법이다. ArraySize는 **왕복 오버헤드**를 줄이고,
> PRP는 **읽을 데이터 자체**를 줄인다. **OLTP**에서 둘의 결합은 **최강 조합**이다.

---

## ArraySize 조정에 의한 Fetch call 감소 및 (간접) I/O 효과

### Fetch call 수 감소 공식

$$
N_{\text{fetch}} = \left\lceil \frac{R}{A} \right\rceil,
$$
여기서 \(R\)은 가져올 행 수, \(A\)는 ArraySize.

- 예) 10,000행을 ArraySize=50 → **200회** 왕복
- ArraySize=1000 → **10회** 왕복
- RTT(왕복 지연)가 3ms만 되어도, **200회 ↦ 10회**는 왕복 지연만 **570ms 절감** 가능

### ArraySize가 **블록 I/O 자체를 줄이진 않는다**, 다만…

- **같은 행 수**를 읽는다면, 논리/물리 블록 I/O는 **대체로 동일**하다.
- 다만 **왕복 경계**가 줄어 **스캔 모멘텀**이 유지되고, 서버/네트워크 컨텍스트 스위칭이 적어져
  **경합/대기**(예: `SQL*Net message to/from client`)가 줄고, 일부 환경에선 **락/래치 재진입**이 줄면서
  **부수적으로** I/O 대기가 낮아 보이는 경우가 있다.
- **명확한 I/O 감소**는 **PRP**(읽는 양 자체 축소), **커버링 인덱스**, **필터/정렬 인덱스화**에서 온다.

---

## 언어별 Array Fetch 세팅과 예제

> **중요**: ArraySize는 “행 갯수”인 경우(JDBC, Python 등), “바이트”인 경우(ODP.NET `FetchSize`)가 있다.
> LOB/폭넓은 컬럼이 많으면 **메모리와 네트워크 프레임**을 고려하여 **중간값**을 찾자.

### JDBC (Oracle JDBC Thin)

```java
String sql = """
  SELECT /* PRP + ArrayFetch */ order_id, order_dt, amount, status
  FROM orders
  WHERE customer_id = ?
  ORDER BY order_dt DESC
  FETCH FIRST 20 ROWS ONLY
""";

try (PreparedStatement ps = conn.prepareStatement(sql)) {
  ps.setInt(1, 12345);
  // PRP가 있으므로 rows 자체가 20 이하 → fetchSize는 100~1000으로 충분
  ps.setFetchSize(200); // 일반 조회는 500~2000 권장, PRP이면 100~500도 충분
  try (ResultSet rs = ps.executeQuery()) {
    while (rs.next()) {
      // consume
    }
  }
}
```

> **팁**
> - 대용량 스트리밍 조회라면 `setFetchSize(1000~5000)`로 시작해 실측.
> - Statement Cache 활성(Implicit/Explicit)로 Parse 경감.

### ODP.NET (C#)

```csharp
using var cmd = new OracleCommand(@"
  SELECT order_id, order_dt, amount, status
  FROM orders
  WHERE customer_id = :cust
  ORDER BY order_dt DESC
  FETCH FIRST 20 ROWS ONLY", conn);

cmd.Parameters.Add(":cust", OracleDbType.Int32).Value = 12345;

// ODP.NET은 FetchSize(바이트 단위). 행당 평균 크기를 추정해 4~16MB 사이 권장.
cmd.FetchSize = 8 * 1024 * 1024;

using var rdr = cmd.ExecuteReader();
while (rdr.Read()) { /* consume */ }
```

### Python (python-oracledb)

```python
cur.arraysize = 1000  # 행 단위
cur.execute("""
  SELECT order_id, order_dt, amount, status
  FROM orders
  WHERE customer_id = :cust
  ORDER BY order_dt DESC
  FETCH FIRST 20 ROWS ONLY
""", cust=12345)
rows = cur.fetchall()  # PRP라 20행 내외 → 왕복 1회면 끝
```

### Node.js (node-oracledb)

```js
const result = await connection.execute(
  `
  SELECT order_id, order_dt, amount, status
  FROM orders
  WHERE customer_id = :cust
  ORDER BY order_dt DESC
  FETCH FIRST 20 ROWS ONLY
  `,
  { cust: 12345 },
  { prefetchRows: 500, fetchArraySize: 1000 } // 실측으로 조정
);
```

---

## PRP를 위한 **SQL 패턴**과 인덱스 설계

### TOP-N(최신 N) 리스트

```sql
SELECT /*+ index(o ix_orders_cust_dt) */
       o.order_id, o.order_dt, o.amount, o.status
FROM   orders o
WHERE  o.customer_id = :cust
ORDER  BY o.order_dt DESC
FETCH FIRST :N ROWS ONLY;
```
- 인덱스 `(customer_id, order_dt DESC)` → `INDEX RANGE SCAN + STOPKEY`
- 필요한 열이 인덱스에 다 있다면 **커버링**으로 테이블 액세스 제거

### Keyset Pagination(다음 페이지)

```sql
-- 마지막으로 본 행 (last_dt, last_id) 이후 상위 20
SELECT order_id, order_dt, amount, status
FROM   orders
WHERE  customer_id = :cust
  AND (order_dt, order_id) < (:last_dt, :last_id)  -- 튜플 비교
ORDER  BY order_dt DESC, order_id DESC
FETCH FIRST 20 ROWS ONLY;
```
- `(customer_id, order_dt DESC, order_id DESC)` 인덱스 추천
- **OFFSET … FETCH** 대비 **PRP가 확실**히 작동 (OFFSET은 앞부분을 버리느라 비효율)

### 주문 목록 + 상태 필터

```sql
SELECT /*+ index(o ix_orders_cust_dt) */
       o.order_id, o.order_dt, o.amount
FROM   orders o
WHERE  o.customer_id = :cust
  AND  o.status = 'OK'
ORDER  BY o.order_dt DESC
FETCH FIRST 20 ROWS ONLY;
```
- 인덱스 선택: `(customer_id, status, order_dt DESC)` vs `(customer_id, order_dt DESC, status)`
- **선택도 높은 컬럼** → 앞쪽에 배치, **정렬 컬럼**은 끝에 **DESC** 로 배치가 일반적
- 실제 데이터 분포로 확인(AWR, XPLAN)

---

## TKPROF/Trace로 **전/후** 확인하는 절차

1) 세션에서 Trace ON
```sql
ALTER SESSION SET statistics_level=ALL;
ALTER SESSION SET events '10046 trace name context forever, level 8';
```

2) **비교 시나리오**
   - **A**: TOP-N 없이 전체 조회 + ArraySize=50
   - **B**: PRP(`FETCH FIRST 20`) + ArraySize=1000

3) Trace OFF → TKPROF
```bash
tkprof your.trc out_a.tkprof sys=no sort=prsela,exeela,fchela
tkprof your.trc out_b.tkprof sys=no sort=prsela,exeela,fchela
```

4) **비교 포인트**
   - `call` 표의 **Fetch count**(B가 현저히 작음)
   - **Elapsed**(Fetch/Execute) 감소
   - `bytes sent/received via SQL*Net` 당 rows 증가(효율↑)
   - PRP가 있다면 **논리/물리 I/O**도 함께 감소

---

## (중요) 안티 패턴과 교정

| 안티 패턴 | 문제 | 교정 |
|---|---|---|
| `OFFSET :skip FETCH :take` | 앞부분을 읽고 버림 → PRP 불가 | **Keyset Pagination** 로 전환 |
| 정렬 컬럼이 인덱스에 없음 | 전체 읽고 정렬 → STOPKEY 무력화 | 정렬 포함 **복합 인덱스** 설계 |
| Leading wildcard `LIKE '%abc'` | 인덱스 스캔 불가 → Full Scan | 접두사 인덱싱/역색인/검색엔진 |
| 필요한 열이 인덱스 밖 | BY ROWID 테이블 재참조 많음 | **커버링 인덱스** 고려 |
| ArraySize=디폴트(작음) | Fetch 왕복 과다 | `setFetchSize/FetchSize/arraysize` 상향 |
| ORM N+1 | 호출 폭증 | JOIN/배열 IN 바인드로 합치기 |

---

## OLTP 튜닝 시나리오 3선

### 주문 목록 첫 페이지 API

- **Before**: `OFFSET 0 FETCH 50` + 정렬 인덱스 없음 → 평균 400ms
- **After**: `(customer_id, order_dt DESC)` 인덱스 + `FETCH FIRST 50` + fetchSize=1000
- **결과**: **Index Range Scan + STOPKEY**, Fetch 왕복 1~2회 → 평균 40ms

### 알람 피드(무한스크롤)

- **Keyset**: `(tenant_id, created_at DESC, id DESC)`
- API는 `(last_ts, last_id)` 전달
- PRP로 **연속 페이지**도 매번 상위 50만 읽고 종료 → 안정적인 지연

### 백오피스 그리드 조회

- 컬럼 최소화(보여줄 것만), 정렬·필터 컬럼 **복합 인덱스화**
- 사용자 스크롤 시 **Lazy Fetch**(다음 페이지 요청 시만)
- fetchSize=2000, 네트워크 RTT 큰 사내망은 5000도 시도

---

## 프로그램 언어별 “Array 단위 Fetch” 활용 모음

### JDBC

```java
PreparedStatement ps = conn.prepareStatement(SQL);
ps.setFetchSize(1000);         // 500~2000 권장 (행당 크기 고려)
ResultSet rs = ps.executeQuery();
// 대량일수록 효과 큼. PRP면 100~500도 충분.
```

### ODP.NET

```csharp
cmd.FetchSize = 8 * 1024 * 1024; // 바이트 단위, 4~16MB부터 실측
using var rdr = cmd.ExecuteReader();
while (rdr.Read()) { /* ... */ }
```

### Python (python-oracledb)

```python
cur.arraysize = 2000  # 기본 100~50 → 1000~5000으로 키워보며 측정
cur.execute(SQL, ...)
rows = cur.fetchall()
```

### Node.js (node-oracledb)

```js
const result = await connection.execute(SQL, binds, {
  prefetchRows: 500,      // 서버가 미리 보내는 행
  fetchArraySize: 2000    // 클라이언트 한 번에 받는 행
});
```

---

## 정량 지표(간이)

- **Fetch 효율**
  $$
  \text{Rows per Fetch Call} = \frac{\text{총 결과 행 수}}{\text{Fetch calls}}
  $$
  값이 **클수록** 좋다(대개 ArraySize와 유사).
- **왕복 오버헤드 추정**
  $$
  \text{RTT Cost} \approx \text{RTT} \times N_{\text{fetch calls}}
  $$
  RTT가 2~5ms만 되어도 **수백 회 왕복**이면 수백 ms가 날아간다.
- **PRP 효율**
  $$
  \text{I/O 절감률} \approx 1 - \frac{\text{PRP 읽은 블록}}{\text{전체 읽기 블록}}
  $$
  인덱스 정렬/커버링이 좋을수록 절감률이 커진다.

---

## 종합 실습: “느린 목록”을 10배 빠르게

**문제**: 고객 포털에서 “주문 목록” 첫 페이지가 p95 1.1s
**원인**: `OFFSET/FETCH`, 정렬 인덱스 없음, fetchSize=50

**개선**
1) 인덱스 `(customer_id, order_dt DESC)`
2) SQL을 `FETCH FIRST 50 ROWS ONLY` 로 변경
3) JDBC `setFetchSize(1000)`
4) 보여주는 컬럼 최소화(커버링 고려)

**결과(TKPROF 요지)**
- Fetch calls: **22 → 1~2**
- Execute Elapsed: **대폭 감소**(STOPKEY)
- 전체 Elapsed: **~1.1s → ~90ms**

---

## 체크리스트

- [ ] **PRP 가능**한가? (TOP-N/Keyset)
- [ ] **정렬/필터**가 **복합 인덱스**로 해결되는가? (DESC 포함)
- [ ] **커버링 인덱스**로 테이블 접근을 줄일 수 있는가?
- [ ] **ArraySize/prefetchRows/FetchSize** 를 현실적으로 키웠는가?
- [ ] **OFFSET** 대신 **Keyset** 을 쓰는가?
- [ ] TKPROF에서 **Fetch count/Elapsed** 가 줄었는가?
- [ ] **행 수 자체**(PRP)와 **왕복 수**(Array)는 별개임을 이해하고 둘 다 줄였는가?

---

## 결론

- **부분범위처리(PRP)** 는 **읽을 양 자체**를 줄여 **I/O·CPU·대기**를 감소시킨다.
- **Array Fetch** 는 **왕복 수**를 줄여 **네트워크/컨텍스트 스위칭** 비용을 제거한다.
- OLTP의 “짧고 잦은” 쿼리에서 두 기법을 결합하면, 종종 **10배** 이상의 체감 성능 개선이 가능하다.
- 인덱스(필터·정렬·커버링)와 SQL 패턴(TOP-N, Keyset), 드라이버의 Array 옵션을 **세트**로 설계하라.
- 마지막으로, **TKPROF/Trace** 로 전후를 **수치로 증명**하라 — **Fetch call**과 **Elapsed** 는 거짓말을 하지 않는다.
