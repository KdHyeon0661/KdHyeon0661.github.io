---
layout: post
title: DB 심화 - sort area 사용 최소화
date: 2025-11-24 14:25:23 +0900
category: DB 심화
---
# sort area(정렬 작업영역, PGA/TEMP) 사용을 **최소화**하는 SQL 작성법

**주제**:
1) **“소트를 완료하고 나서 데이터 가공하기” Top-N 쿼리** — 정렬 대상을 **가볍게** 만들고, **STOPKEY**로 **부분 정렬**만 수행한 뒤, 이후에 **상세 가공/조인**
2) **분석함수(윈도우 함수)에서의 Top-N** — `ROW_NUMBER()`/`RANK()` 기반 Top-N을 **정렬 최소화/회피** 관점에서 설계

> 핵심 원리
> - **정렬은 행 수 × 행 폭에 비례**하여 sort area(PGA)와 TEMP를 소비한다.
> - **Top-N은 전체 정렬이 아니라 “부분 정렬(힙)”**로 끝낼 수 있다. (실행계획: `SORT ORDER BY STOPKEY`)
> - 따라서 **정렬 전에**: ① **정렬 키 + 식별자(PK/ROWID)만** 투영(얇게 만들기), ② **가능하면 인덱스가 만든 순서**를 이용해 **정렬 자체**를 없애거나 최소화한다.
> - 이후(정렬 후) **필요한 컬럼만 조인/가공**하여 전체 메모리 사용량을 낮춘다.

---

## 0) 실습 스키마 & 준비

```sql
-- 제품/매출 샘플(간단)
CREATE TABLE d_product (
  prod_id   NUMBER PRIMARY KEY,
  category  VARCHAR2(16) NOT NULL,
  brand     VARCHAR2(16) NOT NULL
);

CREATE TABLE f_sales (
  sales_id  NUMBER PRIMARY KEY,
  prod_id   NUMBER NOT NULL REFERENCES d_product(prod_id),
  sales_dt  DATE   NOT NULL,
  qty       NUMBER NOT NULL,
  amount    NUMBER(12,2) NOT NULL,
  big_note  CLOB          -- (정렬 시 투영되면 행폭이 커져 sort area 많이 사용)
);

-- 자주 쓰는 인덱스
CREATE INDEX ix_sales_amount_desc ON f_sales(amount DESC);
CREATE INDEX ix_sales_dt_amount   ON f_sales(sales_dt, amount DESC);
CREATE INDEX ix_sales_prod_dt     ON f_sales(prod_id, sales_dt);

INSERT INTO d_product VALUES (10,'ELEC','BR0');
INSERT INTO d_product VALUES (20,'HOME','BR1');

INSERT INTO f_sales VALUES (1001,10,DATE '2025-02-03',1,10000, RPAD('a',2000,'a'));
INSERT INTO f_sales VALUES (1002,20,DATE '2025-02-05',2,15000, RPAD('b',2000,'b'));
INSERT INTO f_sales VALUES (1003,10,DATE '2025-02-10',1, 9000, RPAD('c',2000,'c'));
COMMIT;
```

실행계획·정렬 사용량 확인 루틴:

```sql
-- 마지막 커서의 실제 실행계획/통계
SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +NOTE +ALIAS'));

-- 세션 정렬 지표(실행 전/후 비교)
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON sn.stat#=ms.stat#
WHERE  sn.name IN ('sorts (memory)','sorts (disk)',
                   'workarea executions - optimal',
                   'workarea executions - onepass',
                   'workarea executions - multipass');
```

---

# 1) “**소트를 끝내고 나서 가공**”하는 **Top-N** 패턴

> 목적: **정렬 대상**을 **정렬 키 + 식별자**만으로 **극단적으로 얇게** 만든 다음, **STOPKEY**로 상위 N건만 뽑고, **그 뒤에**(밖에서) **무거운 컬럼/조인**을 수행한다. → **sort area 최소화**

## 나쁜 예(정렬 대상이 비대)

```sql
-- (안티패턴) 모든 컬럼(심지어 CLOB)까지 끌고 와서 정렬 + 상위 100
SELECT *
FROM   f_sales
WHERE  sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
ORDER  BY amount DESC
FETCH  FIRST 100 ROWS ONLY;
-- PLAN: SORT ORDER BY STOPKEY (부분정렬이지만 행폭이 매우 큼 → PGA/TEMP 낭비)
```

## 좋은 예(얇게 정렬 → 밖에서 확장)

```sql
-- 1단계: 정렬 필요한 "키만" (정렬키 + 식별자) 투영하고 Top-N
WITH topn AS (
  SELECT /*+ INDEX_DESC(f_sales ix_sales_amount_desc) */
         sales_id, amount
  FROM   f_sales
  WHERE  sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
  FETCH FIRST 100 ROWS ONLY
)
-- 2단계: Top-N 식별자로 원테이블(+다른 테이블들)에서 "필요 컬럼만" 조인/가공
SELECT s.sales_id, s.prod_id, s.sales_dt, s.qty, s.amount, s.big_note
FROM   topn t
JOIN   f_sales s ON s.sales_id = t.sales_id;
```

**포인트**
- 내부 뷰 `topn`의 **투영 컬럼을 최소화** → sort area 사용량 **극소화**
- 인덱스 `ix_sales_amount_desc`로 **정렬 자체를 줄이거나**(DESC 순서) **STOPKEY**와 결합
- 무거운 CLOB/넓은 컬럼, 타 테이블 조인은 **정렬 이후**에 수행

## Top-N + 추가 필터/집계 “정렬 후 가공” 응용

```sql
-- 상위 200건의 상세 중, 특정 브랜드만 필터 + 추가 계산
WITH topn AS (
  SELECT /*+ INDEX_DESC(f_sales ix_sales_amount_desc) */
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

- **정렬 대상**이 작아진 상태에서 **다음 단계의 가공/조인**을 하므로 전체 메모리/디스크 사용 감소.
- 윗단에서 정렬 키만 다루므로 **정렬 시 행폭(바이트/행)이 작아져** PGA 압박 ↓

## Top-N을 **더 일찍** 적용하기(조인/집계 이전 정렬 축소)

```sql
-- (안티패턴) 조인/가공 다 한 후에 ORDER BY + FETCH → 불필요한 대량 정렬
SELECT s.*, p.category
FROM   f_sales s JOIN d_product p ON p.prod_id = s.prod_id
WHERE  p.category = 'ELEC'
ORDER  BY s.amount DESC
FETCH  FIRST 100 ROWS ONLY;

-- (개선) Top-N 후보를 먼저 좁히고, 그 뒤에 조인/가공
WITH topn AS (
  SELECT /*+ INDEX_DESC(f_sales ix_sales_amount_desc) */
         sales_id, amount
  FROM   f_sales
  FETCH FIRST 100 ROWS ONLY
)
SELECT s.*, p.category
FROM   topn t
JOIN   f_sales s ON s.sales_id = t.sales_id
JOIN   d_product p ON p.prod_id = s.prod_id
WHERE  p.category = 'ELEC';
```

> **주의**: 비즈니스 요구에 따라 결과가 달라질 수 있다.
> - “카테고리 ELEC **중에서** 상위 100”이면 **WHERE→정렬** 순서를 유지해야 하므로 내부에서 카테고리 필터 뒤 Top-N을 적용한다.
> - “전체 상위 100 **중** ELEC만 보여주기”이면 위 개선안처럼 먼저 Top-N 후 필터가 맞다.
> 요구 사항을 명확히!

---

# 2) **분석함수에서의 Top-N** — 정렬 최소화·회피 전략

분석함수 Top-N의 대표 형태:

```sql
SELECT *
FROM (
  SELECT s.*,
         ROW_NUMBER() OVER (ORDER BY amount DESC) AS rn
  FROM   f_sales s
) x
WHERE  x.rn <= 100;
```

위 쿼리는 보통 **`WINDOW SORT`**(전역) 또는 **`WINDOW SORT` + `SORT ORDER BY STOPKEY`**가 필요하다.
아래의 방법으로 **정렬 비용을 줄이거나 우회**한다.

---

## 전역 Top-N: `ROW_NUMBER()` 대신 **인덱스 + ROWNUM** (정렬 회피)

```sql
-- 인덱스: (amount DESC)
SELECT /*+ INDEX_DESC(f_sales ix_sales_amount_desc) */
       sales_id, prod_id, sales_dt, qty, amount
FROM   f_sales
WHERE  ROWNUM <= 100;
```

- **ROW_NUMBER() 제거**로 **WINDOW SORT** 자체 회피
- 인덱스가 금액 내림차순 순서를 제공 → **정렬 없이** 상위 N건
- 이후에 상세 가공/조인은 **밖에서** 수행(§1 패턴 결합)

---

## 그룹별 Top-N (예: 상품별 Top-3) — 3가지 패턴 비교

### 패턴 A) 분석함수 표준 패턴 (단순/가독성 ↑, 정렬 비용 高)

```sql
-- (표준) 각 prod_id별 amount 내림차순 Top-3
SELECT *
FROM (
  SELECT s.*,
         ROW_NUMBER() OVER (PARTITION BY prod_id ORDER BY amount DESC) AS rn
  FROM   f_sales s
  WHERE  sales_dt >= :lo AND sales_dt < :hi
) x
WHERE x.rn <= 3;
```

- 장점: 간단·가독성
- 단점: 보통 **`WINDOW SORT`(PARTITION BY+ORDER BY)** 가 필요 → sort area/TEMP 부담

> **정렬을 덜 쓰는 요령**
> - 입력을 **이미 (prod_id, amount DESC) 순서**로 공급
>   ```sql
>   CREATE INDEX ix_prod_amt ON f_sales(prod_id, amount DESC);
>   SELECT /*+ INDEX(f_sales ix_prod_amt) */ ... ROW_NUMBER() OVER (PARTITION BY prod_id ORDER BY amount DESC) ...
>   ```
>   옵티마이저가 순서를 신뢰하면 `WINDOW SORT`가 최소화/생략될 수 있다(버전/통계 의존 → XPLAN 확인).

### 패턴 B) **상관 서브쿼리 + STOPKEY** (정렬 최소, NL 스케일 패턴)

```sql
-- 각 상품(prod_id)별 Top-3: LATERAL/CROSS APPLY (12c+)
SELECT p.prod_id, s.sales_id, s.amount, s.sales_dt
FROM   d_product p
CROSS APPLY (
  SELECT /*+ INDEX_DESC(f_sales ix_sales_prod_dt) */   -- 인덱스: (prod_id, sales_dt) 또는 (prod_id, amount DESC)
         s.sales_id, s.amount, s.sales_dt
  FROM   f_sales s
  WHERE  s.prod_id = p.prod_id
  ORDER  BY s.amount DESC
  FETCH  FIRST 3 ROWS ONLY
) s
WHERE  p.category = 'ELEC';
```

- 장점: **그룹별로 N개만** 인덱스 **부분범위 + STOPKEY** 로 긁음 → **대규모 글로벌 정렬 없음**
- 단점: **드라이빙(row 소수)** + **NL 접근** 전제 시 강력. 대량 그룹/광범위 범위면 비용↑ 가능
- TIP: 파티션/인덱스가 `(prod_id, amount DESC)`로 정렬 제공하면 **정렬 거의 없음**

> 12c 미만이면 상관 서브쿼리로 같은 효과:
```sql
SELECT p.prod_id, s.sales_id, s.amount, s.sales_dt
FROM   d_product p
JOIN   (
  SELECT s1.prod_id, s1.sales_id, s1.amount, s1.sales_dt
  FROM   f_sales s1
  WHERE  ROWNUM <= 0  -- 옵티마이저 페널티 회피용 더미(상황별 생략)
) dummy ON dummy.prod_id = p.prod_id
WHERE  p.category = 'ELEC';
-- 실제 구현은 상관 서브쿼리 형태:
-- SELECT ... FROM d_product p
-- WHERE EXISTS (
--   SELECT 1 FROM (SELECT ... FROM f_sales WHERE f_sales.prod_id = p.prod_id ORDER BY amount DESC FETCH FIRST 3 ROWS ONLY)
-- )
```

### 패턴 C) **Top-1이라면** `KEEP(DENSE_RANK FIRST/LAST)` 또는 `MIN/MAX` 응용

```sql
-- 각 상품별 최댓값(Top-1)만 필요할 때
SELECT prod_id,
       MAX(amount) KEEP (DENSE_RANK FIRST ORDER BY amount DESC) AS top_amount,
       MIN(sales_dt) KEEP (DENSE_RANK FIRST ORDER BY amount DESC) AS top_amount_dt
FROM   f_sales
GROUP  BY prod_id;

-- 또는 인덱스 MIN/MAX STOPKEY(Top-1)
SELECT s.*
FROM   f_sales s
WHERE  s.sales_dt = (
  SELECT MAX(s2.sales_dt) -- 또는 MIN/조건 변경
  FROM   f_sales s2
  WHERE  s2.prod_id = s.prod_id
);
```

- Top-1은 **정렬/윈도우 없이**도 해결 가능한 경우가 많음(인덱스가 순서 제공)
- **Top-K(2,3,...)**부터는 분석함수 or §2.2B 패턴이 필요

---

## “전역 Top-N + 추가 가공” — 분석함수 vs 인덱스 방식 비교

### (1) 분석함수 방식

```sql
WITH ranked AS (
  SELECT s.sales_id, s.prod_id, s.amount,
         ROW_NUMBER() OVER (ORDER BY s.amount DESC) AS rn
  FROM   f_sales s
  WHERE  s.sales_dt >= :lo AND s.sales_dt < :hi
)
SELECT r.sales_id, r.prod_id, r.amount, p.brand
FROM   ranked r
JOIN   d_product p ON p.prod_id = r.prod_id
WHERE  r.rn <= 100;
```

- **WINDOW SORT** 발생 가능 → 데이터 많으면 sort area 소비
- 장점: 명료, Top-N 정의가 SQL 하나로 끝남

### (2) 인덱스 + ROWNUM + 사후 가공(권장)

```sql
-- 인덱스 (amount DESC) 활용
WITH topn AS (
  SELECT /*+ INDEX_DESC(f_sales ix_sales_amount_desc) */
         sales_id, prod_id, amount
  FROM   f_sales
  WHERE  sALES_dt >= :lo AND sALES_dt < :hi
  AND    ROWNUM <= 100
)
SELECT t.sales_id, t.prod_id, t.amount, p.brand
FROM   topn t
JOIN   d_product p ON p.prod_id = t.prod_id;
```

- **정렬 최소/회피** + **사후 가공**으로 sort area 절약
- 실제 **행폭**과 **행 수**를 내부에서 최대한 작게 만들어라

---

# 3) 추가 테크닉 — sort area 줄이는 세부 요령

## “정렬 전” 투영 최소화: **정렬키 + 식별자(PK/ROWID)** 만

```sql
WITH topn AS (
  SELECT /*+ INDEX_DESC(f_sales ix_sales_amount_desc) */
         ROWID rid, amount         -- 또는 PK
  FROM   f_sales
  FETCH FIRST 200 ROWS ONLY
)
SELECT s.*
FROM   topn t
JOIN   f_sales s ON s.ROWID = t.rid;
```
- 내부 정렬 행폭이 **최소치**가 되도록 설계(정렬 대상 바이트 ↓)
- 이후 조인에서만 필요한 칼럼을 **늦게** 끌고 온다

## 파티션/로컬 인덱스로 **부분 Top-N 병렬화**

- RANGE 파티션(예: 월별) + 로컬 DESC 인덱스면 각 파티션에서 **STOPKEY**로 **N’**씩 뽑고,
  최종적으로 **작은 집합 병합**만 수행 → 대규모 글로벌 정렬 회피/축소

## 인덱스 선택 유도: `INDEX_DESC`, `LEADING`, `NO_USE_MERGE`

- **해시/SMJ**가 입력 순서를 깨뜨리면 결국 추가 정렬/윈도우 정렬이 필요해질 수 있음
- **NL + 인덱스 스캔**으로 **자연 순서 유지** 유도

## **Top-N 추정 범위**를 먼저 줄이기(전처리 필터)

- 일/주/월 등 **좁은 기간**으로 후보를 줄인 뒤 정렬
- `amount >= :threshold` 같은 **선필터**로 후보 감소 → sort area 절약

---

# 4) 성능 측정 & 검증(전/후)

```sql
-- (1) 대상 SQL 실행 전 지표
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON sn.stat#=ms.stat#
WHERE  sn.name IN ('sorts (memory)','sorts (disk)',
                   'workarea executions - optimal',
                   'workarea executions - onepass',
                   'workarea executions - multipass');

-- (2) 대상 SQL 실행

-- (3) 실행 후 지표(증가량 확인)
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON sn.stat#=ms.stat#
WHERE  sn.name IN ('sorts (memory)','sorts (disk)',
                   'workarea executions - optimal',
                   'workarea executions - onepass',
                   'workarea executions - multipass');

-- (4) 실행계획: STOPKEY/INDEX_DESC, WINDOW SORT 존재 여부 체크
SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +NOTE +ALIAS'));
```

- 기대: `SORT ORDER BY STOPKEY`는 보일 수 있으나 **TEMP 사용/onepass/multipass**가 **제로에 가깝게**
- `WINDOW SORT`가 보이면 §2의 대체 패턴으로 재작성하여 제거/축소 시도

---

# 5) 안티패턴 → 리팩터링 요약표

| 항목 | 안티패턴 | 리팩터링(Top-N/분석함수) | 기대효과 |
|---|---|---|---|
| 정렬 대상 | `SELECT *` 후 정렬 | **정렬키+식별자만** 정렬 → 밖에서 조인/가공 | 행폭↓, PGA/TEMP↓ |
| 전역 Top-N | `ROW_NUMBER() OVER(ORDER BY ...)` | **INDEX DESC + ROWNUM** + 사후 가공 | WINDOW SORT 회피 |
| 그룹별 Top-N | `ROW_NUMBER() OVER(PARTITION BY ...)` | **CROSS APPLY + STOPKEY**(인덱스 `(grp, measure DESC)`) | 글로벌 정렬 회피 |
| 조인 후 정렬 | 대량 조인 → ORDER BY | **Top-N 먼저**(요건 검토) → 이후 조인 | 정렬 입력량↓ |
| 표현식 정렬 | `ORDER BY SUBSTR(...)` | **FBI/가상 컬럼** + 인덱스 | 정렬 제거 |

---

# 6) 종합 예제 — **sort area 최소화** 설계 끝판왕

## 요구

- 2025-02 월 전체 거래 중 **금액 상위 100**의 상세 정보 + 제품명/브랜드, `big_note(CLOB)` 포함
- **정렬 메모리 최소화**가 최우선

### A안(분석함수, 단순하지만 메모리 부담)

```sql
WITH ranked AS (
  SELECT s.sales_id, s.prod_id, s.amount, s.sales_dt,
         ROW_NUMBER() OVER (ORDER BY s.amount DESC) AS rn
  FROM   f_sales s
  WHERE  s.sales_dt >= DATE '2025-02-01' AND s.sales_dt < DATE '2025-03-01'
)
SELECT r.sales_id, r.prod_id, r.amount, r.sales_dt, s.big_note, p.brand
FROM   ranked r
JOIN   f_sales   s ON s.sales_id = r.sales_id
JOIN   d_product p ON p.prod_id  = r.prod_id
WHERE  r.rn <= 100;
-- PLAN: WINDOW SORT 가능성 높음 → sort area↑
```

### B안(권장: 인덱스 + STOPKEY + 사후 가공)

```sql
-- 1) 인덱스가 정렬 제공
--    (sales_dt, amount DESC) 또는 (amount DESC) + 기간 필터 조합
CREATE INDEX ix_dt_amt_desc ON f_sales(sales_dt, amount DESC);

-- 2) 얇게 뽑고(정렬키+PK) → 밖에서 가공
WITH topn AS (
  SELECT /*+ INDEX(f_sales ix_dt_amt_desc) */
         sales_id, amount
  FROM   f_sales
  WHERE  sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
  FETCH  FIRST 100 ROWS ONLY                 -- STOPKEY
)
SELECT s.sales_id, s.prod_id, s.sales_dt, s.amount, s.big_note, p.brand
FROM   topn t
JOIN   f_sales   s ON s.sales_id = t.sales_id
JOIN   d_product p ON p.prod_id  = s.prod_id;
-- 기대: 내부는 INDEX Range + STOPKEY (정렬 최소), 외부에서만 상세 가공
```

**검증 포인트**
- `DBMS_XPLAN`: 내부 뷰가 `SORT ORDER BY STOPKEY` 또는 아예 `INDEX DESC + STOPKEY`로 소트 최소화되는지 확인
- `v$mystat`: `sorts (disk)`/`workarea executions - onepass/multipass` 증가 **0** 근접

---

# 7) 체크리스트 (Top-N & 분석함수로 sort area 줄이기)

1. **정렬 전 투영 최소화**: 정렬키 + PK/ROWID만. (CLOB/긴 문자열 절대 금지)
2. **인덱스가 정렬 제공**: `(정렬키, …)`+**DESC/ASC 일치**, 가능하면 **INDEX DESC** 사용.
3. **STOPKEY 활용**: `FETCH FIRST N ROWS ONLY` 또는 `ROWNUM <= N`.
4. **정렬 후 가공**: Top-N 확정 뒤에만 큰 컬럼/조인/계산.
5. **분석함수는 신중히**: 전역 Top-N은 인덱스+ROWNUM으로 대체.
   그룹별 Top-N은 가능하면 **CROSS APPLY + STOPKEY**(인덱스 `(grp, measure DESC)`)로.
6. **기간/선필터 우선**: 후보 집합을 최대한 줄이고 정렬.
7. **파티션/로컬 인덱스**: 파티션별 Top-N 추출 → 결과 병합.
8. **실증**: `DBMS_XPLAN` + `v$mystat`로 **정렬/워크에어리어** 지표가 **유의미하게 감소**했는지 확인.

---

## 맺음말

Top-N은 “**전체 정렬**”이 아니라 “**필요 상위 N개만**”을 추려내는 문제다.
**정렬키 중심의 얇은 투영** + **인덱스 순서** + **STOPKEY** + **사후 가공**을 조합하면,
`WINDOW SORT`/대형 `SORT ORDER BY` 없이도 **응답시간**과 **PGA/TEMP 사용량**을 큰 폭으로 줄일 수 있다.
항상 **실행계획**과 **세션 통계**로 **전/후 차이**를 확인하라.
