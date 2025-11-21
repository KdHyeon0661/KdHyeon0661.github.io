---
layout: post
title: DB 심화 - OR Expansion
date: 2025-11-18 23:25:23 +0900
category: DB 심화
---
# OR Expansion(OR-EXPAND)

> **핵심 요약**
> - **OR expansion**은 `A OR B` 형태의 조건을 **분기(브랜치)** 로 쪼개 **`UNION ALL`(또는 `UNION`)** 으로 재작성하여 **각 분기별로 최적의 인덱스/조인순서/파티션 프루닝**을 선택하게 하는 **쿼리 변환**입니다.
> - 장점: **인덱스 활용 극대화**, **파티션 프루닝** 촉진, **브랜치별 다른 조인전략** 적용, **카디널리티/비용 추정 개선**.
> - 실무 제어: `USE_CONCAT`(OR expansion 유도), 필요 시 분기 **상호배타화(AND NOT …)**, 조인순서/방식은 `LEADING/ORDERED` + `USE_NL/USE_HASH/USE_MERGE` 로 브랜치별 고정.
> - 주의: **중복(Overlap)** 이 있는 OR을 `UNION ALL`로 나누면 **중복행**이 생깁니다 → **상호배타 조건**을 추가하거나 **`UNION`(Distinct)** 을 사용하세요.

---

## 0) 실습 스키마(요약)

```sql
-- 고객/제품 차원 + 매출 사실
CREATE TABLE d_customer (
  cust_id NUMBER PRIMARY KEY,
  region  VARCHAR2(8) NOT NULL,
  tier    VARCHAR2(8) NOT NULL
);

CREATE TABLE d_product (
  prod_id  NUMBER PRIMARY KEY,
  category VARCHAR2(16) NOT NULL,
  brand    VARCHAR2(16) NOT NULL
);

CREATE TABLE f_sales (
  sales_id NUMBER PRIMARY KEY,
  cust_id  NUMBER NOT NULL,
  prod_id  NUMBER NOT NULL,
  sales_dt DATE   NOT NULL,
  qty      NUMBER NOT NULL,
  amount   NUMBER(12,2) NOT NULL
)
PARTITION BY RANGE (sales_dt) (
  PARTITION p202501 VALUES LESS THAN (DATE '2025-02-01'),
  PARTITION p202502 VALUES LESS THAN (DATE '2025-03-01'),
  PARTITION pmax    VALUES LESS THAN (MAXVALUE)
);

-- 대표 인덱스
CREATE INDEX ix_fs_cust_dt ON f_sales(cust_id, sales_dt);
CREATE INDEX ix_fs_prod_dt ON f_sales(prod_id, sales_dt);
CREATE INDEX ix_prod_cat   ON d_product(category, prod_id);
CREATE INDEX ix_cust_tier  ON d_customer(tier, cust_id);
```

관찰 루틴:

```sql
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'ALLSTATS LAST +PREDICATE +ALIAS +NOTE'));
```

---

# 1) OR Expansion 기본

## 개념과 재작성

일반적으로 CBO는 아래를 시도합니다:

```
SELECT …
FROM   T
WHERE  (cond_A) OR (cond_B)
```

→ **OR expansion**:

```
SELECT /* branch A */ …
FROM   T
WHERE  cond_A
UNION ALL
SELECT /* branch B */ …
FROM   T
WHERE  cond_B
```

- **분기별**로 **인덱스/파티션/조인 순서**를 **각자 최적화**.
- **중복(Overlap)** 없는 **상호배타**라면 `UNION ALL`이 이상적(정렬/해시 DISTINCT 비용이 없음).
- **중복 가능**하면:
  - (a) `UNION` 사용(결과 동치 보장, 비용↑), 혹은
  - (b) **두 번째 분기에 `AND NOT(cond_A)`** 를 추가해 **상호배타화**(추천).

**힌트로 유도**: `USE_CONCAT`
(옵티마이저가 자동 변환하지 않거나 주저할 때 강력한 신호)

```sql
SELECT /*+ USE_CONCAT */ …
FROM   …
WHERE  cond_A OR cond_B;
```

## 대표 이득

- OR 한 덩어리로 두면 **인덱스 접근**이 애매 → **Full Scan** 쪽으로 기울 수도 있음.
- `UNION ALL` 분기 후:
  - **브랜치 A**: `cond_A`에 잘 맞는 인덱스/파티션만 접근
  - **브랜치 B**: `cond_B`에 맞게 전혀 다른 경로/조인순서 선택
- 결과: **I/O 감소, CPU 감소, 플랜 안정성↑**

---

# 2) 브랜치별 조인 순서 최적화

## 예제: 차원-사실 조합에서 분기별 다른 드라이빙

**요구**: 2025-02월 매출 중, **(고객 등급 VIP) OR (제품 카테고리 ELEC)** 를 만족하는 매출 합계.

### (A) 원본(OR 그대로)

```sql
SELECT SUM(s.amount)
FROM   f_sales s
JOIN   d_customer c ON c.cust_id = s.cust_id
JOIN   d_product  p ON p.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-02-28'
AND    (c.tier = 'VIP' OR p.category = 'ELEC');
```

- OR 때문에 CBO가 **하나의 조인순서**와 **하나의 접근 경로**만 선택 →
  둘 다 만족시키려면 **광범위 스캔** 위험.

### (B) OR expansion + 브랜치별 최적화

```sql
-- 분기1: 고객 VIP로 먼저 좁히고 → 사실 → 제품
SELECT /*+ USE_CONCAT LEADING(c s p) USE_NL(s) */
       SUM(s.amount) sum_amt
FROM   d_customer c
JOIN   f_sales   s ON s.cust_id = c.cust_id
JOIN   d_product p ON p.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-02-28'
AND    c.tier = 'VIP'

UNION ALL

-- 분기2: 제품 ELEC으로 먼저 좁히고 → 사실 → 고객
SELECT /*+ LEADING(p s c) USE_HASH(s) */
       SUM(s.amount) sum_amt
FROM   d_product p
JOIN   f_sales  s ON s.prod_id = p.prod_id
JOIN   d_customer c ON c.cust_id = s.cust_id
WHERE  s.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-02-28'
AND    p.category = 'ELEC';
```

- **분기1**: `c.tier='VIP'`가 **고선택적**이라면 `LEADING(c s p)` + `USE_NL(s)`로 **인덱스 랜덤 소량 접근**
- **분기2**: `p.category='ELEC'`가 더 광범위라면 `LEADING(p s c)` + `USE_HASH(s)`로 **해시 + 블룸 필터**
- `s.sales_dt` 범위가 **파티션 프루닝**을 유발 → 두 분기 **모두 동일 이득**

> **중복 주의**: (VIP **이면서** ELEC) 레코드는 **두 분기에서 중복**됩니다.
> 안전하게 하려면 2번째 분기에 **`AND NOT(c.tier='VIP')`** 추가 혹은 최종을 `UNION`으로.

---

# 3) 같은 컬럼에 대한 OR expansion

## `status='A' OR status='B'`

### (A) IN-LIST로 묶기(단순·가독성·플랜 안정)

```sql
SELECT *
FROM   orders
WHERE  status IN ('A','B');
```

- 오라클은 인덱스에서 **`INLIST ITERATOR`** 로 **각 값별 인덱스 탐색**을 순회(= OR 확장과 유사 효과).
- **두 값의 선택도/분포**가 비슷하고, **같은 인덱스**로 접근 가능하면 가장 간결.

### (B) 분기별 전략이 달라야 한다면 OR expansion

```sql
-- 가정: 'A'는 극도로 선택적(인덱스 스캔 유리), 'B'는 상대적으로 광범위(Full 또는 다른 경로 유리)
SELECT /*+ USE_CONCAT */ *
FROM   orders
WHERE  status = 'A'
UNION ALL
SELECT /*+ FULL(o) */ *
FROM   orders o
WHERE  o.status = 'B'
AND    o.status <> 'A';   -- 상호배타화(중복방지)
```

- **브랜치1**: 인덱스 **고효율**
- **브랜치2**: 차라리 **Full/멀티블록**이 유리할 수 있음
- IN-LIST 한 방으로는 **두 전략을 동시에** 적용하기 어려움 → **OR expansion 가치**.

## 같은 컬럼의 **범위 OR 범위**

예) `price < 100 OR price BETWEEN 900 AND 1000`

```sql
SELECT /*+ USE_CONCAT */ *
FROM   product
WHERE  price < 100
UNION ALL
SELECT *
FROM   product
WHERE  price BETWEEN 900 AND 1000
AND    NOT(price < 100);  -- 상호배타화
```

- **각 범위**가 **전혀 다른 리프/블록 구간**이라면, **브랜치별 범위 스캔**이 매우 효율적.
- 단일 OR로 두면 옵티마이저가 ‘보수적’ 비용을 잡아 **Full**을 택할 수도 있음.

---

# 4) NVL/DECODE 조건식에 대한 OR expansion

실무에서 **선택적 조건** 처리를 위해 흔히 쓰는 패턴:

## “바인드가 NULL이면 필터하지 않기” — NVL 잘못된 사용

```sql
-- (안티패턴) 컬럼 가공으로 SARGABLE 깨짐
-- NVL(:p, col) = col  →  :p가 NULL이면 항상 TRUE, 아니면 col=:p
SELECT *
FROM   d_customer
WHERE  NVL(:p_tier, tier) = tier;
```

- 옵티마이저 입장에서는 **컬럼 가공** → 인덱스 사용/푸시/프루닝이 어려움.

### (개선) OR로 명시화 + OR expansion

```sql
-- 의미를 OR로 다시 표현
-- (:p_tier IS NULL) OR (tier = :p_tier)
SELECT /*+ USE_CONCAT */
       *
FROM   d_customer
WHERE  :p_tier IS NULL
UNION ALL
SELECT *
FROM   d_customer
WHERE  tier = :p_tier
AND    :p_tier IS NOT NULL;  -- 상호배타화
```

- **분기1**: 파라미터가 NULL이면 **그냥 전량**(필터 없음) — 인덱스 강제 말고 상황따라 Full/샘플링 등
- **분기2**: 파라미터가 값이면 **`tier = :p_tier`** 로 **인덱스 범위 스캔** **확실화**

> 이 패턴은 **보고서/검색 화면**에서 선택적 필터가 많은 경우에 **핵심**.
> 각 필드에 대해 동일하게 **분기화**하면, CBO가 **선택적인 분기**에서 **정확히 인덱스**를 타게 할 수 있음.

## 범위 파라미터의 선택적 적용

```sql
-- (자주 보는 형태)
-- ( :lo IS NULL OR s.sales_dt >= :lo )
-- AND ( :hi IS NULL OR s.sales_dt <  :hi )
SELECT COUNT(*)
FROM   f_sales s
WHERE  ( :lo IS NULL OR s.sales_dt >= :lo )
AND    ( :hi IS NULL OR s.sales_dt <  :hi );
```

- 이대로 두면 OR 두 개가 섞여 **복합 OR** → 접근 경로가 **흐려짐**.

### (개선) 경우의 수로 OR expansion

경우의 수:
1) `:lo`와 `:hi` **둘 다 NULL** → 날짜 필터 **없음**
2) `:lo` **만 채움** → `>= :lo`
3) `:hi` **만 채움** → `< :hi`
4) 둘 다 채움 → `>= :lo AND < :hi`

```sql
SELECT /*+ USE_CONCAT */ COUNT(*)
FROM   f_sales s
WHERE  :lo IS NULL AND :hi IS NULL

UNION ALL
SELECT COUNT(*)
FROM   f_sales s
WHERE  :lo IS NOT NULL AND :hi IS NULL
AND    s.sales_dt >= :lo

UNION ALL
SELECT COUNT(*)
FROM   f_sales s
WHERE  :lo IS NULL AND :hi IS NOT NULL
AND    s.sales_dt < :hi

UNION ALL
SELECT COUNT(*)
FROM   f_sales s
WHERE  :lo IS NOT NULL AND :hi IS NOT NULL
AND    s.sales_dt >= :lo
AND    s.sales_dt <  :hi;
```

- 각 분기에서 **파티션 프루닝**이 명확해지고, 인덱스/조인전략을 따로 고를 수 있음.
- 실무에서는 **동적 SQL**로 필요한 분기만 생성하는 방식을 많이 사용(분기 1~4 중 해당만).

## DECODE/CASE 기반 선택적 조건

```sql
-- (안티패턴) :p_category가 NULL이면 전체, 아니면 해당 값만
-- DECODE(:p_category, NULL, 1, CASE WHEN p.category = :p_category THEN 1 END) = 1
SELECT *
FROM   d_product p
WHERE  DECODE(:p_category, NULL, 1, CASE WHEN p.category = :p_category THEN 1 END) = 1;
```

### (개선) OR 명시 + OR expansion

```sql
SELECT /*+ USE_CONCAT */ *
FROM   d_product p
WHERE  :p_category IS NULL
UNION ALL
SELECT *
FROM   d_product p
WHERE  p.category = :p_category
AND    :p_category IS NOT NULL;  -- 상호배타화
```

- 복잡한 `DECODE/CASE`의 **의미를 OR로 풀어** CBO가 이해하기 쉽게.

---

# 5) 성능 포인트 — 프루닝/인덱스/조인

## 파티션 프루닝 & 전이성

OR을 분기화하면, 각 분기에서 **파티션 키**가 **명확**해져 **프루닝**이 강력해집니다.

```sql
SELECT /*+ USE_CONCAT */ SUM(s.amount)
FROM   f_sales s
JOIN   d_product p ON p.prod_id = s.prod_id
WHERE  ( p.category='ELEC' AND s.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-02-28' )
   OR  ( p.category='HOME' AND s.sales_dt BETWEEN DATE '2025-01-01' AND DATE '2025-01-31' );
```

- 분기1은 **p202502**만, 분기2는 **p202501**만 읽는 식으로 나뉨.
- 조인 등치 + **조건절 이행(Transitivity)** 로 `s`에도 범위가 **access**로 내려감.

## 브랜치별 다른 인덱스/조인

- 분기1: `c.tier='VIP'` → `ix_cust_tier` 타고 **NL**
- 분기2: `p.category='ELEC'` → `ix_prod_cat` 타고 **HASH** + **Join Filter(Bloom)**
- 동일 OR 조건 한 방으론 불가했던 **이질적 전략**을 동시에 실현.

---

# 6) 중복(Overlap) 처리 전략

OR을 `UNION ALL`로 나누면, **양쪽 조건을 모두 만족**하는 행이 **중복**됩니다.

### 상호배타화(권장)

```sql
-- B 분기에 A가 아닌 것만
SELECT /* A */ * FROM T WHERE cond_A
UNION ALL
SELECT /* B */ * FROM T WHERE cond_B AND NOT(cond_A);
```

- 장점: **정렬/해시 DISTINCT 비용 없음**
- 단점: `NOT(cond_A)` 가 **SARGABLE**이 아닐 수도 있음 → 주의

### `UNION` 사용(중복제거)

```sql
SELECT /* A */ * FROM T WHERE cond_A
UNION
SELECT /* B */ * FROM T WHERE cond_B;
```

- 장점: **간단/정확**
- 단점: **정렬/해시** 비용, 대용량/병렬에서는 비용 의미 큼

---

# 7) 종합 실습: Before vs After

## 데이터 준비(요약)

```sql
-- 샘플 삽입 (생략 없이 일부만 예시)
INSERT INTO d_customer VALUES (1,'KOR','VIP');
INSERT INTO d_customer VALUES (2,'USA','STD');
INSERT INTO d_product  VALUES (10,'ELEC','B0');
INSERT INTO d_product  VALUES (20,'HOME','B1');

INSERT INTO f_sales VALUES (1001,1,10,DATE '2025-02-03',1,10000);
INSERT INTO f_sales VALUES (1002,1,20,DATE '2025-01-15',2,15000);
INSERT INTO f_sales VALUES (1003,2,10,DATE '2025-02-10',1, 9000);
COMMIT;
```

## OR 그대로(Before)

```sql
SELECT SUM(s.amount)
FROM   f_sales s
JOIN   d_customer c ON c.cust_id = s.cust_id
JOIN   d_product  p ON p.prod_id = s.prod_id
WHERE  (c.tier='VIP' OR p.category='ELEC')
AND    s.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-02-28';
```

- 플랜: 한 가지 조인순서/접근 경로만 → **광범위 스캔** 위험

## OR expansion(After) + 브랜치별 최적화

```sql
-- A: VIP 우선
SELECT /*+ USE_CONCAT LEADING(c s p) USE_NL(s) */
       SUM(s.amount) sum_amt
FROM   d_customer c
JOIN   f_sales   s ON s.cust_id = c.cust_id
JOIN   d_product p ON p.prod_id = s.prod_id
WHERE  c.tier='VIP'
AND    s.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-02-28'

UNION ALL

-- B: ELEC 우선 + 중복 방지
SELECT /*+ LEADING(p s c) USE_HASH(s) */
       SUM(s.amount) sum_amt
FROM   d_product p
JOIN   f_sales  s ON s.prod_id = p.prod_id
JOIN   d_customer c ON c.cust_id = s.cust_id
WHERE  p.category='ELEC'
AND    s.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-02-28'
AND    NOT (c.tier='VIP');  -- 상호배타화
```

관찰 포인트:

```sql
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'ALLSTATS LAST +PREDICATE +ALIAS +NOTE'));
-- 각 브랜치에 서로 다른 LEADING/조인 방식/액세스 프레디킷/프루닝이 나타나는지 확인
```

---

# 8) 체크리스트 & 팁

- [ ] **`USE_CONCAT`** 로 OR expansion 유도(자동 변환이 불안정할 때)
- [ ] **상호배타화**(AND NOT …) 또는 **`UNION`** 로 중복 제거
- [ ] 브랜치별 **`LEADING/ORDERED` + `USE_NL/USE_HASH/USE_MERGE`** 로 전략 고정
- [ ] **파티션 키**가 분기별로 **명확**해지도록 WHERE를 정리(프루닝 이득)
- [ ] 같은 컬럼의 OR는 **IN-LIST**/`INLIST ITERATOR`가 더 나을 수도(간단·안정)
- [ ] **NVL/DECODE/CASE** 기반의 선택적 조건은 **OR로 풀어** 분기화 → SARGABLE 회복
- [ ] 오버헤드 점검: 분기 수가 많아지면 **UNION ALL** 오버헤드 vs 이득 균형
- [ ] **실측 플랜/세션 통계**로 전/후 비교(논리/물리 읽기, A-Rows 등)
- [ ] 히스토그램/통계 **정확화**로 브랜치별 카디널리티 오판 방지

---

## 부록 A. 미니 패턴 모음

### A-1. 선택적 LIKE 필터

```sql
-- (원형) :kw가 NULL이면 전체, 아니면 LIKE
WHERE (:kw IS NULL OR name LIKE :kw || '%')

-- (OR expansion)
SELECT /*+ USE_CONCAT */ *
FROM   members
WHERE  :kw IS NULL
UNION ALL
SELECT *
FROM   members
WHERE  :kw IS NOT NULL
AND    name LIKE :kw || '%';
```

### A-2. IN-LIST vs OR expansion

```sql
-- (간단) 인덱스에 잘 맞고 값 수가 소량 → INLIST ITERATOR 기대
WHERE state IN ('CA','NY','TX');

-- (분기 필요) 'CA'는 인덱스 고선택, 'NY','TX'는 Full 유리
SELECT /*+ USE_CONCAT */ *
FROM   addr WHERE state = 'CA'
UNION ALL
SELECT /*+ FULL(a) */ *
FROM   addr a WHERE a.state IN ('NY','TX') AND a.state <> 'CA';
```

### A-3. 동일 테이블 다른 컬럼 OR

```sql
-- (원형) cust_id=… OR prod_id=…
SELECT *
FROM   f_sales
WHERE  cust_id = :cid
   OR  prod_id = :pid;

-- (분기화) 각자 다른 인덱스 사용
SELECT /*+ USE_CONCAT */ *
FROM   f_sales WHERE cust_id = :cid
UNION ALL
SELECT *
FROM   f_sales WHERE prod_id = :pid
AND    ( :cid IS NULL OR cust_id <> :cid );  -- 중복 방지(상황별 조정)
```

---

## 부록 B. 자주 묻는 질문

**Q1. 항상 `USE_CONCAT`가 좋은가요?**
- 아닙니다. **분기 수 증가**는 **오버헤드**를 동반. OR이 단순하고 **INLIST ITERATOR**가 잘 작동하면 그대로 두는 게 낫습니다.

**Q2. 중복 방지를 꼭 상호배타로 해야 하나요?**
- 상호배타화가 가장 **비용이 낮고 예측 가능**합니다. 다만 구현이 복잡하면 `UNION` 사용도 현실적입니다.

**Q3. NVL/DECODE를 전부 OR로 바꿔야 하나요?**
- **선택적 필터**(파라미터 NULL이면 패스) 패턴은 OR로 **의미를 드러내야** CBO가 정확히 최적화합니다.

**Q4. OR expansion을 막고 싶을 때는?**
- (상황에 따라) 변환 억제 힌트나, **뷰 물리화/NO_MERGE**, 혹은 **한 쿼리 블록에 담아** 분기화를 피합니다.
  일반적으로는 **막을 일보다 유도**할 일이 훨씬 많습니다.

---

### 맺음말

- OR 한 줄이 **플랜의 자유도**를 억누르기도 합니다.
- **OR expansion**은 그 자유도를 회복시켜 **브랜치별 최적 전략**을 고르게 하는 강력한 방법입니다.
- 핵심은 **상호배타/중복 처리**, **브랜치별 조인/인덱스 전략 고정**, **SARGABLE 회복**.
- 항상 **실측 플랜과 읽기량**으로 “정말 이득이 났는지”를 확인하세요.
