---
layout: post
title: DB 심화 - OR Expansion
date: 2025-11-18 23:25:23 +0900
category: DB 심화
---
# OR Expansion(OR-EXPAND)

## 0) 이 글에서 다루는 핵심

- OR Expansion의 **정의/목적/형태(UNION ALL / CONCATENATION)**
- **Legacy OR-expansion(LORE)** vs **Cost-based OR-expansion(ORE)** 차이, 버전/행동 변화
- OR 확장이 **가능(legal)** 한 조건과 **실패/보류** 이유
- **인덱스/파티션 프루닝/조인전략**이 분기별로 달라지는 메커니즘
- 중복(Overlap) 문제와 **상호배타화(AND NOT)** 패턴
- **IN-LIST iterator**, **Bitmap OR**, **OR Expansion**의 선택 기준
- **바인드/히스토그램/카디널리티**와의 상호작용
- 실무 힌트 패키지: `USE_CONCAT`, `OR_EXPAND`, `NO_OR_EXPAND`, `NO_EXPAND`, `LEADING`, `USE_NL/HASH/MERGE`
- **Before/After** 전체 실습(파티션 + 차원/사실 조인)

---

## 1) OR Expansion이 필요한 이유: OR은 왜 플랜 자유도를 죽이나?

### OR 조건의 본질적 문제

아래 쿼리를 보자.

```sql
SELECT *
FROM   t
WHERE  cond_A OR cond_B;
```

CBO는 **하나의 쿼리 블록**에 대해 **하나의 조인 순서**, **하나의 액세스 경로**를 선택해야 한다.
그런데 `cond_A`와 `cond_B`는 대개 다음처럼 **성격이 다르다**.

- 서로 다른 컬럼을 필터링한다.
- 선택도(Selectivity)가 극단적으로 다르다.
- 파티션 키 범위를 다르게 만든다.
- 어떤 쪽은 인덱스가 유리하고, 다른 쪽은 풀스캔이 유리하다.

이런 상황에서 OR을 “한 덩어리”로 두면, CBO는 **둘 다 만족할 수 있는 보수적(비싼) 경로**를 택하기 쉽다.
즉, OR은 **플랜 탐색공간을 좁히는 족쇄**가 된다.

### OR Expansion의 정의

Oracle 문서에서 OR expansion은 **최상위 OR(Disjunction)** 을 가진 쿼리 블록을 **UNION ALL 분기 형태**로 변환하는 CQT(비용기반 쿼리 변환)로 정의된다.

> 변환(개념):

```sql
-- 원형
SELECT ...
FROM   T
WHERE  cond_A OR cond_B;

-- OR Expansion
SELECT /* branch A */ ...
FROM   T
WHERE  cond_A
UNION ALL
SELECT /* branch B */ ...
FROM   T
WHERE  cond_B;
```

각 분기는 **별도의 쿼리 블록**이 되므로,
- 분기마다 **최적 인덱스**를 고를 수 있고,
- 조인 순서/조인 방식도 **분기별로 달라질 수 있으며**,
- 파티션 프루닝(access predicate)이 **명확**해진다.

---

## 2) OR Expansion의 종류: LORE(legacy) vs ORE(cost-based)

Oracle 내부 트레이스/코멘트에서 OR 확장은 두 계열로 언급된다.

### LORE (Legacy OR-Expansion, “Concatenation”)

- 오래된 방식: **모든 분기에서 인덱스 접근이 가능해야** 잘 작동하는 경향
- 옵티마이저 트레이스에서 **CONCATENATION / LORE** 로 표시
- 실무적으로 `USE_CONCAT`로 유도하는 전통적 패턴

### ORE (Cost-based OR-Expansion)

- 12cR2 이후 강화된 방식으로 알려져 있으며,
  **분기마다 서로 다른 조인전략/액세스 경로**를 더 유연하게 허용한다.
- 트레이스에 **OR_EXPAND / ORE** 같은 이름으로 등장
- 상황에 따라 자동 적용/자동 거부가 더 정교해졌다.

### 버전 업그레이드 시 주의

11g → 19c/23ai 같은 업그레이드에서
어떤 SQL이 **예전에는 LORE로 잘 풀렸는데**, 새 버전에선 ORE를 선택하지 못해 **확장이 사라지고 플랜이 악화**되는 사례가 있다.

이때는 **`USE_CONCAT`로 legacy 확장을 다시 강제**하는 것이 실무 해법이 된다.

---

## 3) OR Expansion을 제어하는 힌트와 규칙

### `USE_CONCAT` — OR Expansion 강제

`USE_CONCAT`는 OR(및 IN-list)을 **UNION ALL 기반 분기**로 강제로 재작성하게 하는 힌트다.

```sql
SELECT /*+ USE_CONCAT */ ...
FROM   T
WHERE  cond_A OR cond_B;
```

특징:
- 자동으로 확장하지 않거나 주저할 때 “강한 신호”
- 단, **IN-LIST iterator를 꺼버리고** OR을 전부 확장할 수 있으니(= 분기 수 폭발 위험) 주의.

### `OR_EXPAND` / `NO_OR_EXPAND`

- `OR_EXPAND`는 **보다 공격적으로 OR expansion을 강제**하는 힌트로 소개된다.
- 반대로 `NO_OR_EXPAND(@qb)`는 특정 쿼리 블록에서 OR 확장을 막는다.

```sql
SELECT /*+ OR_EXPAND */ ...
FROM   T
WHERE  cond_A OR cond_B;

SELECT /*+ NO_OR_EXPAND(@main) */ ...
FROM   T
WHERE  cond_A OR cond_B;
```

실무 포인트:
- `OR_EXPAND`는 **정말 확장이 필요할 때만**.
- 자동 변환이 과도하거나 잘못될 때는 `NO_OR_EXPAND`로 차단하고 다른 전략을 택한다.

### `NO_EXPAND` — OR/INLIST 확장 방지

전통적인 차단 힌트. OR을 그대로 처리하게 만든다.

```sql
SELECT /*+ NO_EXPAND */ ...
FROM   T
WHERE  cond_A OR cond_B;
```

---

## 4) OR Expansion의 “합법성(legality)”과 자동 적용 조건

CBO는 “아무 OR이나” 다 확장하지 않는다.
**의미 보존(semantic equivalence)** 이 확실해야 하고, 비용도 유리해야 한다.

### 자동 확장이 가능한 전형 조건

- OR이 **WHERE 최상위(top-level disjunction)** 로 존재
- OR 양쪽이 **같은 쿼리 블록**에 있고,
  확장 후에도 **결과가 동일**함이 논리적으로 보장
- 각 분기가 최소한의 플랜(인덱스/조인)으로 **명확히 이득**이 예상될 때

### 자동 확장이 보류/거부되는 대표 사례

- OR이 **복잡하게 중첩**되어 법적 재작성 비용이 큰 경우
- OR 양쪽이 **집계/윈도우/스칼라 서브쿼리/외부 조인 의미**를 깨뜨릴 위험이 있는 경우
- 분기 수가 너무 커져 **UNION ALL 오버헤드가 더 비싼 경우**
- 바인드 변수/통계 부정확으로 **분기별 비용 추정이 불안정**한 경우
- 업그레이드/패치 이후 특정 OR 변환이 **차단되는 회귀**(실무에서 `USE_CONCAT`로 복구)

---

## 5) 실습 스키마(파티션 + 차원/사실)

아래는 이후 예제들의 공통 스키마.
(당신이 준 스키마를 유지하되, 실험 관찰을 위해 약간 확장)

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';
ALTER SESSION SET statistics_level = ALL;

-- 차원 테이블
CREATE TABLE d_customer (
  cust_id NUMBER PRIMARY KEY,
  region  VARCHAR2(8)  NOT NULL,
  tier    VARCHAR2(8)  NOT NULL
);

CREATE TABLE d_product (
  prod_id  NUMBER PRIMARY KEY,
  category VARCHAR2(16) NOT NULL,
  brand    VARCHAR2(16) NOT NULL
);

-- 사실 테이블(월별 파티션)
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
  PARTITION p202503 VALUES LESS THAN (DATE '2025-04-01'),
  PARTITION pmax    VALUES LESS THAN (MAXVALUE)
);

-- 인덱스
CREATE INDEX ix_fs_cust_dt ON f_sales(cust_id, sales_dt);
CREATE INDEX ix_fs_prod_dt ON f_sales(prod_id, sales_dt);
CREATE INDEX ix_prod_cat   ON d_product(category, prod_id);
CREATE INDEX ix_cust_tier  ON d_customer(tier, cust_id);

-- 통계(실습에서 반드시 최신화)
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'D_CUSTOMER', cascade=>true);
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'D_PRODUCT', cascade=>true);
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'F_SALES', cascade=>true, method_opt=>'FOR ALL COLUMNS SIZE AUTO');
END;
/
```

관찰 루틴:

```sql
SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
         NULL, NULL,
         'ALLSTATS LAST +PREDICATE +ALIAS +NOTE +PEEKED_BINDS'
       ));
```

---

## 6) OR Expansion의 대표 이득 3종 세트

Oracle Optimizer 팀 공식 글에서도 **인덱스/프루닝/조인 전략**의 분기 최적화가 핵심 이득으로 언급된다.

### 분기별 인덱스 극대화

- OR을 한 덩어리로 두면 “둘 다 어설프게 맞는 인덱스”를 쓰거나, 풀스캔으로 기울 수 있다.
- 확장 후엔 분기마다 **가장 맞는 인덱스**만 타게 된다.

### 파티션 프루닝 강화

- OR이 섞여 있으면 파티션 키 조건이 희석되어 `Pstart/Pstop`이 넓어질 수 있다.
- 분기 후엔 파티션 키 범위가 분명해져 **각 분기가 서로 다른 파티션만 읽는 경우**가 흔하다.

### 분기별 다른 조인 순서/방식

- 어떤 분기는 NL-drive가 최적, 다른 분기는 Hash-drive가 최적일 수 있다.
- OR 확장만이 **이질적 전략을 동시에 허용**해준다.

---

## 7) 실전 1: 차원-사실 조인에서 “분기별 드라이빙” 바꾸기

### 요구사항

2025-02월 매출 중:

- **(고객 등급 VIP) OR (제품 카테고리 ELEC)** 을 만족하는 매출 합계.

### OR 그대로(Before)

```sql
SELECT SUM(s.amount) sum_amt
FROM   f_sales s
JOIN   d_customer c ON c.cust_id = s.cust_id
JOIN   d_product  p ON p.prod_id = s.prod_id
WHERE  s.sales_dt >= DATE '2025-02-01'
AND    s.sales_dt <  DATE '2025-03-01'
AND    (c.tier = 'VIP' OR p.category = 'ELEC');
```

**위험 신호**
- `c.tier='VIP'`는 고선택적
- `p.category='ELEC'`는 상대적으로 광범위
- 하나의 조인 순서로 둘을 만족시키려 하므로
  `f_sales`에서 **넓게 읽고 후보를 필터링**하는 풀/대범위 스캔이 나올 가능성이 높다.

### OR Expansion + 브랜치별 최적화(After)

```sql
SELECT /*+ USE_CONCAT LEADING(c s p) USE_NL(s) */
       SUM(s.amount) sum_amt
FROM   d_customer c
JOIN   f_sales   s ON s.cust_id = c.cust_id
JOIN   d_product p ON p.prod_id = s.prod_id
WHERE  c.tier = 'VIP'
AND    s.sales_dt >= DATE '2025-02-01'
AND    s.sales_dt <  DATE '2025-03-01'

UNION ALL

SELECT /*+ LEADING(p s c) USE_HASH(s) */
       SUM(s.amount) sum_amt
FROM   d_product p
JOIN   f_sales  s ON s.prod_id = p.prod_id
JOIN   d_customer c ON c.cust_id = s.cust_id
WHERE  p.category = 'ELEC'
AND    s.sales_dt >= DATE '2025-02-01'
AND    s.sales_dt <  DATE '2025-03-01';
```

- 분기1: VIP 고객을 드라이빙 → `ix_cust_tier` 기반 소량 NL
- 분기2: ELEC 제품을 드라이빙 → `ix_prod_cat` / Hash / Bloom 가능
- 두 분기 모두 날짜 조건이 access로 내려가 **p202502만 프루닝**될 여지가 커진다.

> 공식 문서/블로그에서 말하는 OR expansion의 대표 장점이 그대로 구현된다.

---

## 8) Overlap(중복) 문제와 상호배타화의 표준 패턴

### 왜 중복이 생기나?

OR은 “합집합”이다.

- OR 그대로: `cond_A ∪ cond_B`
- UNION ALL 분기: `cond_A` + `cond_B`

따라서 `cond_A ∩ cond_B`(둘 다 만족하는 행)가 있으면
`UNION ALL`에서는 결과가 **2번** 나온다.

### 해결책 1: 상호배타화(추천)

```sql
SELECT /* A */ * FROM T WHERE cond_A
UNION ALL
SELECT /* B */ * FROM T WHERE cond_B AND NOT(cond_A);
```

장점
- `UNION`의 DISTINCT 비용(정렬/해시)이 없다.

주의
- `NOT(cond_A)`가 **SARGABLE을 깨** 마지막 분기 비용이 커질 수 있다.
  이때는 `cond_A`의 핵심 필터만 “인덱스 친화” 형태로 재구성해 NOT을 최소화한다.

### 해결책 2: UNION(중복 제거)

```sql
SELECT /* A */ * FROM T WHERE cond_A
UNION
SELECT /* B */ * FROM T WHERE cond_B;
```

- 간단하지만 DISTINCT 비용이 추가된다.
- 대용량/병렬에서 비용 의미가 커질 수 있음.

---

## 9) 같은 컬럼 OR: IN-LIST iterator vs OR Expansion

Oracle은 같은 컬럼의 OR을 **IN-list iterator**로 처리할 수 있다.

### IN-LIST iterator의 의미

```sql
WHERE status IN ('A','B')
```

이면

- 내부적으로 `status='A' OR status='B'`와 동치지만
- 옵티마이저는 같은 인덱스를 반복 스캔하는 **INLIST ITERATOR** 연산을 선택할 수 있다.

이건 사실상 **“자동 OR 확장”의 특수 케이스**다.

### 언제 IN-LIST가 더 낫나?

- 값 수가 작고
- 선택도/분포가 비슷하고
- 같은 인덱스 하나로 충분히 효율적일 때

```sql
SELECT *
FROM   orders
WHERE  status IN ('A','B');
```

### 언제 OR Expansion이 더 낫나?

- 같은 컬럼이더라도 **분기별 최적 경로가 다를 때**

```sql
-- 'A'는 극도로 선택적, 'B'는 광범위라 Full이 낫다고 가정
SELECT /*+ USE_CONCAT */
       *
FROM   orders
WHERE  status = 'A'
UNION ALL
SELECT /*+ FULL(o) */
       *
FROM   orders o
WHERE  o.status = 'B'
AND    o.status <> 'A';  -- 상호배타
```

- 한 방(IN-LIST)로는
  “A는 인덱스, B는 풀스캔” 식의 **이질적 전략을 동시에 적용**할 수 없다.

---

## 10) 서로 다른 컬럼 OR: 두 인덱스를 살리는 가장 흔한 케이스

```sql
SELECT *
FROM   f_sales
WHERE  cust_id = :cid
   OR  prod_id = :pid;
```

- `ix_fs_cust_dt`, `ix_fs_prod_dt`가 **서로 다른 컬럼 기반**
- OR 그대로면 **어느 인덱스를 기준으로 드라이빙해야 할지 애매**
- 결과적으로 Full로 기울 가능성이 높다.

### OR Expansion 표준

```sql
SELECT /*+ USE_CONCAT */ *
FROM   f_sales
WHERE  cust_id = :cid

UNION ALL

SELECT *
FROM   f_sales
WHERE  prod_id = :pid
AND    ( :cid IS NULL OR cust_id <> :cid );
```

- 분기1은 `cust_id` 인덱스
- 분기2는 `prod_id` 인덱스
- 입력 조합에 따라 중복 여부를 적절히 제거

---

## 11) NVL/DECODE/CASE 안티패턴을 OR Expansion으로 복구하기

공식적으로 OR expansion은 **SARGABLE 회복**에 매우 유용하다.

### NVL 컬럼가공 안티패턴

```sql
SELECT *
FROM   d_customer
WHERE  NVL(:p_tier, tier) = tier;
```

문제
- `tier`가 가공됨 → 인덱스/프루닝/푸시다운이 흐려진다.

### OR로 의미를 드러내고 확장

```sql
SELECT /*+ USE_CONCAT */ *
FROM   d_customer
WHERE  :p_tier IS NULL

UNION ALL

SELECT *
FROM   d_customer
WHERE  tier = :p_tier
AND    :p_tier IS NOT NULL;  -- 상호배타
```

- 파라미터 NULL → 전량(Full/병렬/샘플링 가능)
- 파라미터 값 → `tier=:p_tier`로 인덱스 확실

바인드 기반 검색 화면에서 이 패턴은 “정석”이다.

---

## 12) 복합 OR(여러 분기)에서의 비용/오버헤드 균형

OR 조건이 늘수록 분기 수도 늘어나며, 이는 **UNION ALL 오버헤드**를 만든다.

### 분기 수 폭발의 치명점

- 분기가 2개일 때는 대개 이득이 크다.
- 5~10개 이상으로 늘어나면
  - Parse/Optimization 비용 증가
  - 분기마다 조인/스캔이 반복되어 CPU가 증가
  - 병렬에서 UNION ALL Merge 비용 증가

### 실무 규칙

- 값 개수 OR(같은 컬럼)이라면 **IN-LIST** 우선 검토
- 서로 다른 컬럼 OR이라면
  **가장 선택적인 분기만 확장하고 나머지는 묶는 “부분 확장”**이 더 낫기도 하다.

---

## 13) 카디널리티 추정과 히스토그램의 중요성

OR expansion의 성패는 **분기별 선택도 추정 정확도**에 달린다.

### 히스토그램이 없으면?

- OR 전체를 “평균적” 선택도로 추정
- 분기별 비용을 **정확히 비교하지 못해 확장을 포기**하거나
  잘못된 확장을 수행할 수 있다.

### 해결

```sql
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'D_CUSTOMER',
    method_opt => 'FOR COLUMNS tier SIZE 254'
  );
END;
/
```

- 고왜도(Zipf-like) 컬럼에는 **Histogram**을 적극 사용
- OR 분기 목표가 되는 인덱스 컬럼은 특히 중요

---

## 14) 바인드 변수와 OR Expansion

19c 이후 ORE는 바인드 스큐에서도 꽤 잘 동작하는 사례가 보고된다.
하지만 여전히 다음 문제는 존재한다.

### Bind Peeking/Adaptive Cursor Sharing과의 결합

- 처음 바인드로 확장 플랜이 고정된 뒤
  다른 값에서 비용이 뒤집히면
  - ACS가 커서를 갈라주거나
  - 최악의 경우 잘못된 플랜이 오래 유지될 수 있다.

### 실무 대응

- `ALLSTATS LAST +PEEKED_BINDS`로 **실측 플랜** 확인
- 필요하면
  - `USE_CONCAT`로 확장을 “항상” 하게 하거나
  - 반대로 `NO_OR_EXPAND`로 “항상” 막고
    **SPM Baseline**으로 플랜을 안정화한다.

---

## 15) 파티션 프루닝을 극대화하는 OR Expansion 패턴

### 분기별 다른 날짜 범위

```sql
SELECT /*+ USE_CONCAT */ SUM(s.amount)
FROM   f_sales s
JOIN   d_product p ON p.prod_id = s.prod_id
WHERE  ( p.category='ELEC'
         AND s.sales_dt >= DATE '2025-02-01'
         AND s.sales_dt <  DATE '2025-03-01' )
   OR  ( p.category='HOME'
         AND s.sales_dt >= DATE '2025-01-01'
         AND s.sales_dt <  DATE '2025-02-01' );
```

- OR 그대로면 날짜 범위가 합쳐져
  파티션이 **p202501+p202502** 둘 다 열릴 수 있다.

### 확장 후

```sql
SELECT /*+ USE_CONCAT */ SUM(s.amount)
FROM   f_sales s
JOIN   d_product p ON p.prod_id = s.prod_id
WHERE  p.category='ELEC'
AND    s.sales_dt >= DATE '2025-02-01'
AND    s.sales_dt <  DATE '2025-03-01'

UNION ALL

SELECT SUM(s.amount)
FROM   f_sales s
JOIN   d_product p ON p.prod_id = s.prod_id
WHERE  p.category='HOME'
AND    s.sales_dt >= DATE '2025-01-01'
AND    s.sales_dt <  DATE '2025-02-01';
```

- 분기1은 p202502만
- 분기2는 p202501만
**각 분기 프루닝이 완벽히 분리**된다.

---

## 16) Bitmap OR vs OR Expansion vs INLIST — 최종 선택 표

Oracle Optimizer 팀은 **Bitmap OR, OR expansion, INLIST iterator**를 구분해 설명한다.

| 상황 | 최적 후보 | 이유 |
|---|---|---|
| 같은 컬럼, 값 수 적음, 분포 유사, B-tree 인덱스 선택적 | IN-LIST iterator | 같은 인덱스를 반복 스캔하는 게 가장 싸다 |
| 서로 다른 컬럼, 각자 다른 인덱스 존재 | OR Expansion | 분기별 최적 인덱스를 동시에 살림 |
| DW/Bitmap 인덱스 기반 다중 조건 | Bitmap OR | 비트맵 결합이 비용 최저 |
| 분기별 조인/스캔 전략이 달라야 함 | OR Expansion | 분기마다 다른 플랜 허용 |
| OR이 단순하지만 확장이 과도해 오버헤드가 큼 | NO_EXPAND/NO_OR_EXPAND | OR 유지가 더 저렴 |

---

## 17) 종합 실습: Before vs After 전체 비교

### 데이터 준비(간단 샘플)

```sql
INSERT INTO d_customer VALUES (1,'US','VIP');
INSERT INTO d_customer VALUES (2,'DE','STD');
INSERT INTO d_customer VALUES (3,'US','STD');

INSERT INTO d_product VALUES (10,'ELEC','BR1');
INSERT INTO d_product VALUES (20,'HOME','BR2');
INSERT INTO d_product VALUES (30,'ELEC','BR3');

INSERT INTO f_sales VALUES (1001,1,10,DATE '2025-02-03',1,10000);
INSERT INTO f_sales VALUES (1002,1,20,DATE '2025-01-15',2,15000);
INSERT INTO f_sales VALUES (1003,2,10,DATE '2025-02-10',1, 9000);
INSERT INTO f_sales VALUES (1004,3,30,DATE '2025-02-21',1, 7000);
COMMIT;

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'D_CUSTOMER',cascade=>true);
  DBMS_STATS.GATHER_TABLE_STATS(USER,'D_PRODUCT',cascade=>true);
  DBMS_STATS.GATHER_TABLE_STATS(USER,'F_SALES',cascade=>true);
END;
/
```

### OR 그대로 (Before)

```sql
SELECT SUM(s.amount) sum_amt
FROM   f_sales s
JOIN   d_customer c ON c.cust_id = s.cust_id
JOIN   d_product  p ON p.prod_id = s.prod_id
WHERE  (c.tier='VIP' OR p.category='ELEC')
AND    s.sales_dt >= DATE '2025-02-01'
AND    s.sales_dt <  DATE '2025-03-01';
```

관찰:
- 플랜에서 OR 조건이 한 번에 처리되며
  인덱스/조인순서가 “절충안”으로 고정될 수 있다.

### OR Expansion + 상호배타화 (After)

```sql
SELECT /*+ USE_CONCAT LEADING(c s p) USE_NL(s) */
       SUM(s.amount) sum_amt
FROM   d_customer c
JOIN   f_sales   s ON s.cust_id = c.cust_id
JOIN   d_product p ON p.prod_id = s.prod_id
WHERE  c.tier='VIP'
AND    s.sales_dt >= DATE '2025-02-01'
AND    s.sales_dt <  DATE '2025-03-01'

UNION ALL

SELECT /*+ LEADING(p s c) USE_HASH(s) */
       SUM(s.amount) sum_amt
FROM   d_product p
JOIN   f_sales  s ON s.prod_id = p.prod_id
JOIN   d_customer c ON c.cust_id = s.cust_id
WHERE  p.category='ELEC'
AND    s.sales_dt >= DATE '2025-02-01'
AND    s.sales_dt <  DATE '2025-03-01'
AND    NOT (c.tier='VIP');  -- 중복 제거
```

관찰 포인트:

```sql
SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
       'ALLSTATS LAST +PREDICATE +ALIAS +NOTE'));
```

- 각 분기에 **다른 LEADING/조인 방식/인덱스 경로**가 보이는가?
- `sales_dt`가 **각 분기에서 access predicate**로 내려가
  `p202502`만 읽고 있는가?

---

## 18) 실무 체크리스트

- [ ] OR이 **서로 다른 인덱스/파티션 범위/조인전략**을 요구하는가?
  → YES면 OR Expansion 후보.
- [ ] OR이 같은 컬럼 값 나열인가?
  → 먼저 IN-LIST/INLIST ITERATOR 가능성 검토.
- [ ] OR 분기 간 Overlap이 있는가?
  → `AND NOT(cond_A)`로 상호배타화, 또는 `UNION` 사용.
- [ ] 자동 확장이 사라져 성능이 떨어졌는가(업그레이드/패치 후)?
  → `USE_CONCAT`로 legacy 확장을 복구.
- [ ] 분기 수가 너무 많은가?
  → 부분 확장/동적 SQL/IN-LIST 묶기 고려.
- [ ] 통계/히스토그램이 분기 선택도에 맞게 최신인가?
  → 오판 방지 핵심.
- [ ] 바인드 스큐가 큰가?
  → 실측 플랜으로 확인하고 SPM/힌트로 안정화.
- [ ] 최종 판단은 항상 **실측(A-Rows, IO)** 으로.

---

## 19) 맺음말

- OR은 “의미는 단순”하지만 **플랜에겐 매우 어려운 조건**이다.
- OR Expansion은 OR이 만든 족쇄를 풀고,
  **분기별 최적화를 통해 인덱스/프루닝/조인전략을 동시에 살리는** 강력한 CQT다.
- 핵심은
  1) **분기별 최적 경로가 정말 다른가**,
  2) **Overlap을 어떻게 제거할 것인가**,
  3) **통계와 바인드에서 오판이 없는가**
  이 세 가지다.
