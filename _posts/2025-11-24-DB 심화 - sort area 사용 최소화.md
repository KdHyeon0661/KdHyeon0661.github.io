---
layout: post
title: DB 심화 - sort area 사용 최소화
date: 2025-11-24 14:25:23 +0900
category: DB 심화
---
# Oracle에서 **Sort Area(TEMP/PGA) 사용을 최소화**하는 SQL 작성법

> 목표
1) “**소트를 완료하고 나서 데이터 가공하기**” Top-N 패턴으로 **정렬 입력을 얇게** 만들고, `STOPKEY`를 **부분 정렬(Heap Top-N)** 로 끝내는 설계
2) 분석함수 Top-N(`ROW_NUMBER/RANK`)을 **정렬 최소화/회피 관점**에서 재구성하는 패턴

핵심 원리
- 정렬 비용은 **행 수 × 행 폭**에 비례한다.
- Top-N은 전체 정렬이 아니라 **“N개만 유지하는 부분 정렬(Heap)”** 로 끝낼 수 있고, 실행계획에 `SORT ORDER BY STOPKEY`로 나타난다.
- 따라서 **정렬 전에**:
  ① **정렬키 + 식별자(PK/ROWID)만 투영해서 얇게 만들기**,
  ② **가능하면 인덱스가 만든 순서**를 이용해 **정렬 자체를 없애거나 최소화**한다.
- 이후(정렬 후)에만 **상세 컬럼/무거운 조인/계산**을 수행한다.

---

## 0) 왜 Top-N/SORT 튜닝이 “PGA/TEMP 절반을 결정”하는가

정렬과 윈도우 정렬은 **Workarea(PGA) + TEMP**를 동시에 쓴다.
Oracle은 정렬/해시 작업영역에 대해 **메모리 그랜트(Grant)** 를 주고, 부족하면 TEMP로 **One-pass / Multi-pass** 스필을 한다. 이때 다음이 급격히 악화된다.

- `workarea executions - onepass/multipass` 증가
- `sorts (disk)` 증가
- 대기 이벤트: `direct path write/read temp`, PQ 환경이면 `PX Deq Credit: send blkd`

Top-N이란 “상위 N개만” 필요하다는 뜻이므로, **정렬 입력을 얇게 하고, N개에서 멈추게(Stopkey) 만드는 것**이 가장 강력한 메모리 절감 수단이다.

---

## 1) 실습 스키마 & 측정 루틴

### 샘플 테이블

```sql
-- 제품/매출 샘플
DROP TABLE d_product PURGE;
CREATE TABLE d_product (
  prod_id   NUMBER PRIMARY KEY,
  category  VARCHAR2(16) NOT NULL,
  brand     VARCHAR2(16) NOT NULL
);

DROP TABLE f_sales PURGE;
CREATE TABLE f_sales (
  sales_id  NUMBER PRIMARY KEY,
  prod_id   NUMBER NOT NULL REFERENCES d_product(prod_id),
  sales_dt  DATE   NOT NULL,
  qty       NUMBER NOT NULL,
  amount    NUMBER(12,2) NOT NULL,
  big_note  CLOB                 -- 정렬 투영 시 행폭 폭증의 대표
);

-- 정렬/Top-N에 자주 쓰는 인덱스
CREATE INDEX ix_sales_amount_desc ON f_sales(amount DESC);
CREATE INDEX ix_sales_dt_amount   ON f_sales(sales_dt, amount DESC);
CREATE INDEX ix_sales_prod_amt    ON f_sales(prod_id, amount DESC); -- 그룹 Top-K용

INSERT INTO d_product VALUES (10,'ELEC','BR0');
INSERT INTO d_product VALUES (20,'HOME','BR1');

INSERT INTO f_sales VALUES (1001,10,DATE '2025-02-03',1,10000, RPAD('a',2000,'a'));
INSERT INTO f_sales VALUES (1002,20,DATE '2025-02-05',2,15000, RPAD('b',2000,'b'));
INSERT INTO f_sales VALUES (1003,10,DATE '2025-02-10',1, 9000, RPAD('c',2000,'c'));
COMMIT;

EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'D_PRODUCT');
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'F_SALES');
```

### 실행계획/정렬 사용량 확인

```sql
-- 마지막 커서의 실제 실행계획/통계
SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
       'ALLSTATS LAST +PREDICATE +NOTE +ALIAS'));

-- 세션 정렬/워크에어리어 지표(실행 전/후 비교)
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON sn.stat#=ms.stat#
WHERE  sn.name IN ('sorts (memory)','sorts (disk)',
                   'workarea executions - optimal',
                   'workarea executions - onepass',
                   'workarea executions - multipass');
```

---

## 2) Top-N의 내부 동작을 먼저 이해하자

### `STOPKEY`가 뭔가?

Oracle이 “상위 N개”를 처리할 때는 **전체를 끝까지 정렬하지 않는다.**
실행계획에 `SORT ORDER BY STOPKEY`가 나타나면, **정렬 중 힙에 N개만 유지하며 멈추는 부분 정렬**이 발생했다는 뜻이다. (전체 정렬 대비 PGA/TEMP를 크게 줄임)

### Row Limiting Clause vs ROWNUM

12c+에서는 `FETCH FIRST N ROWS ONLY` / `OFFSET`이 Row Limiting Clause로 네이티브 지원된다.
11g 이하에선 `ROWNUM <= N` 패턴이 Top-N의 표준이었고, 지금도 내부적으로 **Stopkey 최적화**를 잘 유도한다.

**중요한 규칙 하나:**
`ROWNUM`은 **ORDER BY보다 먼저** 부여된다.
따라서 `WHERE ROWNUM <= N ORDER BY …`는 Top-N이 아니라 **랜덤 N개의 정렬**이 된다.
Top-N을 하려면 “정렬/인덱스 순서”가 먼저, 그 다음이 ROWNUM/Fetch다.

---

## 3) “**소트를 끝내고 나서 가공**”하는 Top-N 패턴

> 목적: 내부에서 **정렬키 + 식별자만** 정렬/Stopkey로 상위 N건을 확보하고,
> 바깥에서만 **무거운 컬럼/조인/계산**을 수행한다.

### 안티패턴: 정렬 대상이 비대(행폭 폭증)

```sql
-- 모든 컬럼(심지어 CLOB)까지 정렬 대상으로 투영
SELECT *
FROM   f_sales
WHERE  sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
ORDER  BY amount DESC
FETCH  FIRST 100 ROWS ONLY;
```

**문제점**
- Top-N이라도 **정렬 입력의 행폭이 크면** PGA가 커지고 TEMP 스필 위험이 높다.
- 특히 LOB/긴 문자열이 투영되면 정렬 단계에서 **수배~수십배** 메모리가 든다.

---

### 정석: **얇게 정렬 → Stopkey → 밖에서 확장**

```sql
WITH topn AS (
  SELECT /*+ INDEX_DESC(f_sales ix_sales_dt_amount) */
         sales_id, amount          -- 정렬키 + 식별자만
  FROM   f_sales
  WHERE  sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
  FETCH FIRST 100 ROWS ONLY        -- STOPKEY 유도
)
SELECT s.sales_id, s.prod_id, s.sales_dt, s.qty, s.amount, s.big_note
FROM   topn t
JOIN   f_sales s ON s.sales_id = t.sales_id
ORDER  BY t.amount DESC;
```

**효과**
- 내부 `topn`의 정렬 입력이 **극단적으로 얇아져** Sort Area/TEMP가 크게 준다.
- 외부 조인은 Top-N 확정 이후라 **적은 로우만** 확장한다.

---

### **ROWID 기반 Late Materialization** (가장 얇은 정렬)

```sql
WITH topn AS (
  SELECT /*+ INDEX_DESC(f_sales ix_sales_dt_amount) */
         ROWID rid, amount
  FROM   f_sales
  WHERE  sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
  FETCH FIRST 200 ROWS ONLY
)
SELECT s.*
FROM   topn t
JOIN   f_sales s ON s.ROWID = t.rid
ORDER  BY t.amount DESC;
```

- 내부는 **ROWID(또는 PK) + 정렬키만** → 행폭 최저.
- 바깥에서만 `SELECT s.*`로 확장.

---

### “Top-N을 더 **일찍** 적용하기” — 조인/가공 이전에 후보 축소

#### 안티패턴

```sql
SELECT s.*, p.category
FROM   f_sales s
JOIN   d_product p ON p.prod_id = s.prod_id
WHERE  p.category = 'ELEC'
ORDER  BY s.amount DESC
FETCH  FIRST 100 ROWS ONLY;
```

- 조인 후 정렬 → 정렬 입력이 **조인 결과 전체**가 되어 불필요한 대형 소트.

#### 개선(요건이 허용될 때)

```sql
WITH topn AS (
  SELECT /*+ INDEX_DESC(f_sales ix_sales_amount_desc) */
         sales_id, prod_id, amount
  FROM   f_sales
  FETCH  FIRST 100 ROWS ONLY
)
SELECT s.*, p.category
FROM   topn t
JOIN   f_sales   s ON s.sales_id = t.sales_id
JOIN   d_product p ON p.prod_id  = t.prod_id
WHERE  p.category = 'ELEC';
```

**요건 주의**
- “ELEC 중 상위 100”이면 **카테고리 필터 → Top-N**이 맞고,
- “전체 상위 100 중 ELEC만 표시”라면 위 패턴이 맞다.
Top-N 위치는 **결과 의미를 바꾸므로** 비즈니스 정의를 먼저 확정해야 한다.

---

### Top-N + **추가 필터/계산은 정렬 이후에**

```sql
WITH topn AS (
  SELECT /*+ INDEX_DESC(f_sales ix_sales_dt_amount) */
         sales_id, amount
  FROM   f_sales
  WHERE  sales_dt >= :lo AND sales_dt < :hi
  FETCH FIRST 200 ROWS ONLY
)
SELECT s.sales_id,
       p.brand,
       s.amount,
       s.qty,
       (s.amount / NULLIF(s.qty,0)) AS unit_price
FROM   topn t
JOIN   f_sales   s ON s.sales_id = t.sales_id
JOIN   d_product p ON p.prod_id  = s.prod_id
WHERE  p.brand LIKE 'BR%';
```

- 정렬 이후에만 **비싼 가공/조인** → 전체 메모리/Temp 소모 감소.

---

## 4) 분석함수(윈도우 함수) Top-N에서 **정렬을 아끼는 설계**

분석함수 Top-N의 표준형:

```sql
SELECT *
FROM (
  SELECT s.*,
         ROW_NUMBER() OVER (ORDER BY amount DESC) AS rn
  FROM   f_sales s
) x
WHERE rn <= 100;
```

이 방식은 대부분 `WINDOW SORT`(전역 정렬)가 필요하다.
따라서 **“분석함수를 꼭 써야 하는가?”**부터 판단하고, 가능하면 더 싼 패턴으로 치환한다.

---

### 전역 Top-N은 **인덱스 + ROWNUM/FETCH**로 대체(정렬 회피)

```sql
SELECT /*+ INDEX_DESC(f_sales ix_sales_amount_desc) */
       sales_id, prod_id, sales_dt, qty, amount
FROM   f_sales
WHERE  ROWNUM <= 100;
```

- 인덱스가 `amount DESC` 순서를 **그대로 제공** → `WINDOW SORT`가 사라진다.
- Top-N 확정 후 상세 가공이 필요하면 3장의 “얇게 뽑고 확장” 패턴과 결합.

---

### 전역 Top-N + 기간 필터: **(sales_dt, amount DESC) 인덱스 + Stopkey**

```sql
-- 기간+Top-N에서 가장 강력한 커버 인덱스
CREATE INDEX ix_dt_amt_desc ON f_sales(sales_dt, amount DESC);

WITH topn AS (
  SELECT /*+ INDEX(f_sales ix_dt_amt_desc) */
         sales_id, amount
  FROM   f_sales
  WHERE  sales_dt >= :lo AND sales_dt < :hi
  FETCH  FIRST 100 ROWS ONLY
)
SELECT s.*, p.brand
FROM   topn t
JOIN   f_sales s ON s.sales_id = t.sales_id
JOIN   d_product p ON p.prod_id = s.prod_id;
```

- 기간 필터로 인덱스 범위를 좁히고, 범위 내에서 Stopkey로 N개만.
- 대형 `WINDOW SORT`/`SORT ORDER BY`를 거의 제거한다.

---

### **그룹별 Top-K(예: prod_id별 Top-3)** — 세 가지 패턴 비교

#### A) 표준 분석함수(가독성↑, 정렬 비용↑)

```sql
SELECT *
FROM (
  SELECT s.*,
         ROW_NUMBER() OVER (PARTITION BY prod_id ORDER BY amount DESC) AS rn
  FROM   f_sales s
  WHERE  sales_dt >= :lo AND sales_dt < :hi
) x
WHERE rn <= 3;
```

- 보통 `WINDOW SORT` 발생 → PGA/TEMP 부담.

**정렬을 덜 쓰는 조건**
- 입력이 이미 `(prod_id, amount DESC)` 순서라면 `WINDOW SORT`가 **생략/축소**될 수 있다.
- 이를 위해 **복합 DESC 인덱스**를 준다.

```sql
CREATE INDEX ix_prod_amt_desc ON f_sales(prod_id, amount DESC);

SELECT /*+ INDEX(f_sales ix_prod_amt_desc) */
       *
FROM (
  SELECT s.*,
         ROW_NUMBER() OVER (PARTITION BY prod_id ORDER BY amount DESC) AS rn
  FROM   f_sales s
) x
WHERE rn <= 3;
```

- XPLAN에서 `WINDOW SORT`가 **없거나 매우 얕게** 되는지 확인.

---

#### B) LATERAL/CROSS APPLY + STOPKEY (그룹 드라이빙이 작을 때 최강)

```sql
SELECT p.prod_id, s.sales_id, s.amount, s.sales_dt
FROM   d_product p
CROSS APPLY (
  SELECT /*+ INDEX_DESC(f_sales ix_sales_prod_amt) */
         sales_id, amount, sales_dt
  FROM   f_sales s
  WHERE  s.prod_id = p.prod_id
  ORDER  BY s.amount DESC
  FETCH  FIRST 3 ROWS ONLY     -- 그룹 내부 STOPKEY
) s
WHERE  p.category = 'ELEC';
```

**장점**
- 각 그룹에서 **상위 K개만** 인덱스로 “긁어 오고 끝” → 대규모 글로벌 정렬 없음.
- **드라이빙(그룹 수/선택도)이 작을수록** 분석함수보다 훨씬 싸다.

**단점**
- 그룹 수가 크고 범위가 넓으면 NL 반복 비용이 커질 수 있음.

---

#### C) Top-1이라면 `KEEP(DENSE_RANK FIRST/LAST)`로 **정렬/윈도우 없이 해결**

```sql
-- prod_id별 최댓값(Top-1)
SELECT prod_id,
       MAX(amount) KEEP (DENSE_RANK FIRST ORDER BY amount DESC) AS top_amount,
       MIN(sales_dt) KEEP (DENSE_RANK FIRST ORDER BY amount DESC) AS top_dt
FROM   f_sales
GROUP  BY prod_id;
```

- `ROW_NUMBER()` 없이 **그룹 Top-1 집계만** 산출한다.
- Top-1 문제는 해시/그룹 집계로 끝나며, 윈도우 정렬 비용이 크게 줄어든다.
- `DENSE_RANK`는 동점 처리에서 `RANK`와 달리 **순번의 공백을 만들지 않으며**, Top-1/Top-K 선정에 자주 쓰인다.

---

## 5) Sort Area를 더 줄이는 세부 테크닉

### 정렬 전에 **행폭 줄이기** (Projection Pushdown)

- 정렬 단계 내부 뷰는 **정렬키 + PK/ROWID만**
- CLOB/긴 문자열/사용하지 않는 컬럼은 **Top-N 확정 후**에만 조인/확장

### 정렬 전에 **행 수 줄이기**

- 기간/범주/임계치 필터 **선 적용**
- 필요하면 `LEADING`, `USE_NL`로 **작은 집합 선행**
- 파티션 테이블이면 파티션 프루닝을 깨지 말 것
  (파티션 키에 함수/묵시적 형변환 금지)

### “인덱스가 정렬을 대신하게 하라”

- 정렬키 순서와 **인덱스 순서(ASC/DESC) 일치**
- Top-N 전용이라면 **(필터키, 정렬키 DESC)** 인덱스를 고려
- 커버링이 가능하면 **Table Access 최소화**

### Top-N을 **파티션 단위로 분해**

- RANGE 파티션(월별 등) + 로컬 DESC 인덱스가 있으면
  각 파티션에서 **N’씩 Stopkey로 뽑고**, 최종 집합만 합쳐서 재정렬
- 전체 정렬을 “작은 합병 정렬”로 바꾼다.

### 분석함수는 “필수일 때만”

- 전역 Top-N은 **99% 인덱스+Stopkey로 대체 가능**
- 그룹별 Top-K는
  ① 복합 DESC 인덱스로 `WINDOW SORT` 축소, 또는
  ② APPLY/상관 서브쿼리 Stopkey로 글로벌 정렬 제거

---

## 6) 전/후 효과를 **수치로 확인하는 방법**

### 실행계획에서 볼 것

```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
     'ALLSTATS LAST +PREDICATE +NOTE +ALIAS'));
```

- `SORT ORDER BY STOPKEY` 존재 여부
- `WINDOW SORT` 존재/깊이
- 인덱스 접근이 기대한 순서를 제공하는지 (`INDEX RANGE SCAN DESCENDING` 등)

### 세션 정렬/Workarea 지표

```sql
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON sn.stat#=ms.stat#
WHERE  sn.name IN ('sorts (memory)','sorts (disk)',
                   'workarea executions - optimal',
                   'workarea executions - onepass',
                   'workarea executions - multipass');
```

- 리팩터링 후 목표:
  - `sorts (disk)` 증가량이 **0에 수렴**
  - `onepass/multipass` 증가량이 **유의미하게 감소**
  - XPLAN에서 `WINDOW SORT`가 제거되거나 얕아짐

---

## 7) 종합 예제 — “정렬 메모리 최소화” 실전 완성형

**요구**
- 2025-02 월 거래 중 금액 상위 100의 상세(+제품 브랜드, CLOB 포함)

### A안(분석함수, 단순하지만 보통 WINDOW SORT)

```sql
WITH ranked AS (
  SELECT sales_id, prod_id, amount, sales_dt,
         ROW_NUMBER() OVER (ORDER BY amount DESC) AS rn
  FROM   f_sales
  WHERE  sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
)
SELECT r.sales_id, r.prod_id, r.amount, r.sales_dt, s.big_note, p.brand
FROM   ranked r
JOIN   f_sales   s ON s.sales_id = r.sales_id
JOIN   d_product p ON p.prod_id  = r.prod_id
WHERE  r.rn <= 100;
```

### B안(권장: 얇게 정렬 + Stopkey + 사후 가공)

```sql
CREATE INDEX ix_dt_amt_desc ON f_sales(sales_dt, amount DESC);

WITH topn AS (
  SELECT /*+ INDEX(f_sales ix_dt_amt_desc) */
         sales_id, amount
  FROM   f_sales
  WHERE  sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
  FETCH  FIRST 100 ROWS ONLY
)
SELECT s.sales_id, s.prod_id, s.sales_dt, s.amount, s.big_note, p.brand
FROM   topn t
JOIN   f_sales   s ON s.sales_id = t.sales_id
JOIN   d_product p ON p.prod_id  = s.prod_id
ORDER  BY t.amount DESC;
```

**기대 결과**
- 내부는 “얇은 인덱스 범위 + Stopkey”
- 바깥은 Top-N 100건만 확장
- `WINDOW SORT` 제거 가능성 매우 높음

---

## 8) 안티패턴 → 리팩터링 요약표

| 문제 | 안티패턴 | 리팩터링 | 기대효과 |
|---|---|---|---|
| 정렬 입력 행폭 | `SELECT *` 후 정렬 | 정렬키+PK/ROWID만 정렬 후 확장 | PGA/TEMP 대폭 감소 |
| 전역 Top-N | `ROW_NUMBER() OVER(ORDER BY)` | 인덱스 순서 + ROWNUM/FETCH | WINDOW SORT 회피 |
| 그룹 Top-K | 분석함수 전역 정렬 | `(grp, measure DESC)` 인덱스 또는 APPLY+STOPKEY | 글로벌 정렬 제거/축소 |
| 조인 후 정렬 | 대량 조인 → ORDER BY | 후보 Top-N 먼저(요건 확인) → 조인 | 정렬 입력량 감소 |
| 표현식 정렬 | `ORDER BY SUBSTR(...)` | FBI/가상컬럼 + 인덱스 | 정렬 제거 |

---

## 9) 결론

Top-N은 “전체를 다 정렬해 줄 세우는 문제”가 아니다.
**정렬 전에 얇게 만들고**, **Stopkey로 N개에서 멈추고**, **정렬 뒤에만 무거운 일을 하라.**
전역 Top-N은 거의 항상 **인덱스 순서 + ROWNUM/FETCH**로 치환 가능하고,
그룹별 Top-K는 **복합 DESC 인덱스** 또는 **APPLY+STOPKEY**가 분석함수 정렬 비용을 크게 줄인다.

마지막으로, **XPLAN(`STOPKEY`/`WINDOW SORT`) + workarea/sort 통계**를 항상 같이 보라.
정렬을 줄이는 튜닝은 **PGA/TEMP, 응답시간, PQ 스루풋**을 동시에 개선하는 가장 확실한 지름길이다.
