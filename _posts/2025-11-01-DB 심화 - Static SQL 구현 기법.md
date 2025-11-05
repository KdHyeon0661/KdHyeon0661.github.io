---
layout: post
title: DB 심화 - Static SQL 구현 기법
date: 2025-11-01 15:25:23 +0900
category: DB 심화
---
# “Static SQL 구현 기법”
**— IN-list가 가변일 때(소수/대량), 체크 조건이 가변일 때, `SELECT` 목록이 바뀔 때, 연산자가 바뀔 때 — 동적 SQL 없이 풀기**

> **핵심 메시지**  
> 동적 SQL을 쓰지 않고도 대부분의 “가변 요구”를 해결할 수 있다.  
> 비결은 **(1) 바인드 변수**(값만 바뀌고 텍스트는 고정), **(2) 정규화된 패턴**(범위·가드 분기·배열 바인드·GTT), **(3) 옵티마이저 친화성(SARGability)** 을 동시에 만족시키는 것이다.

---

## 0) 실습 공통 스키마 준비

```sql
-- 판매 테이블 (2M rows 가정)
DROP TABLE sales PURGE;

CREATE TABLE sales (
  sale_id     NUMBER PRIMARY KEY,
  region      VARCHAR2(10),      -- 'APAC' | 'EMEA' | 'AMER' | 'OTHER'
  order_dt    DATE,
  status      VARCHAR2(10),      -- 'OK' | 'CANCEL' | ...
  amount      NUMBER(12,2),
  customer_id NUMBER
);

-- 가짜 데이터 (요약)
INSERT /*+ APPEND */ INTO sales
SELECT level,
       CASE MOD(level,10) WHEN 0 THEN 'APAC'
                          WHEN 1 THEN 'EMEA'
                          WHEN 2 THEN 'AMER'
                          ELSE 'OTHER' END,
       DATE '2025-01-01' + MOD(level, 365),
       CASE WHEN MOD(level,20)=0 THEN 'CANCEL' ELSE 'OK' END,
       ROUND(DBMS_RANDOM.VALUE(10,1000),2),
       MOD(level, 100000) + 1
FROM dual CONNECT BY level <= 2000000;

COMMIT;

-- 인덱스 & 통계
CREATE INDEX ix_sales_region_dt ON sales(region, order_dt);
CREATE INDEX ix_sales_status    ON sales(status);
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'SALES', cascade=>TRUE,
    method_opt=>'FOR COLUMNS SIZE 75 region SIZE 75 status SIZE 1 order_dt');
END;
/
```

---

## 1) IN-list 항목이 **가변**이지만 “최대 경우 수가 **적은**” 경우

> **상황**: `region IN (...)` 처럼 가변이지만 **최대 3~5개** 정도로 제한됨.  
> **목표**: 텍스트는 정적, **바인드 재사용**과 **인덱스 범위 스캔**을 보장.

### 1.1 고정 슬롯 + NULL 패딩 (가장 단순, 완전 정적)

```sql
-- 최대 3개까지 선택 가능하다고 가정
-- 사용하지 않는 슬롯은 NULL로 바인딩하고, NVL2 가드로 NULL 제거
SELECT /* static-inlist(≤3) */
       SUM(amount) AS tot_amt
FROM   sales
WHERE  (
         (region = :r1) OR
         (region = :r2) OR
         (region = :r3)
       )
AND    (:r1 IS NOT NULL OR :r2 IS NOT NULL OR :r3 IS NOT NULL)
AND    order_dt BETWEEN :d1 AND :d2;
```

- **포인트**  
  - IN 대신 **OR 3개**를 쓰는 이유: 옵티마이저가 **region** 결합 인덱스를 더 안정적으로 range-scan 하는 케이스가 많다.  
  - **불필요 슬롯에 NULL** 바인딩 → 조건 자체가 **거짓**이 되어 영향 없음.  
  - 텍스트 100% 고정 → **커서 공유 100%**.

### 1.2 고정 IN 목록 + sentinel 값 (정수/숫자 키일 때)

```sql
-- 숫자키라면 “존재할 수 없는 값”으로 패딩
SELECT /* static-inlist(≤5) */
       COUNT(*)
FROM   customers
WHERE  customer_id IN (:c1, :c2, :c3, :c4, :c5)
AND    signup_dt >= :d1
AND    signup_dt <  :d2;
```

- **패딩 방법**: 사용하지 않는 자리는 **-1** 등 존재 불가 값을 바인딩.  
- **장점**: 간단, 완전 정적. **주의**: 실제 데이터 도메인과 상충하지 않게 sentinel 관리.

---

## 2) IN-list 항목이 **가변**이고 “최대 경우 수가 **아주 많은**” 경우

> **상황**: 선택된 ID/코드가 **수십~수천 개**.  
> **목표**: 텍스트 정적, 커서 공유 유지, **대량 키 필터**를 효율적으로.

### 2.1 **스키마 타입 + 배열 바인드** (정적 SQL 유지)

```sql
-- 1) 스키마 타입 생성 (1회)
CREATE OR REPLACE TYPE t_vc10 AS TABLE OF VARCHAR2(10);

-- 2) 정적 SQL: TABLE(:list) 사용
SELECT /* array-bind */
       s.sale_id, s.region, s.amount
FROM   sales s
JOIN   TABLE(:regions) r
  ON   s.region = r.COLUMN_VALUE
WHERE  s.order_dt BETWEEN :d1 AND :d2;
```

- **실행 측**(PL/SQL 예시)
```plsql
DECLARE
  v_regions t_vc10 := t_vc10('APAC','EMEA','AMER', 'OTHER'); -- 가변 크기
BEGIN
  -- 값은 :regions 바인드 하나로 전달 → SQL 텍스트는 완전 정적
  FOR r IN (
    SELECT /* array-bind */ COUNT(*) AS cnt
    FROM   sales s
    JOIN   TABLE(v_regions) r
      ON   s.region = r.COLUMN_VALUE
    WHERE  s.order_dt >= DATE '2025-10-01'
    AND    s.order_dt <  DATE '2025-11-01'
  ) LOOP
    DBMS_OUTPUT.PUT_LINE('cnt='||r.cnt);
  END LOOP;
END;
/
```

- **장점**: **정적 SQL + 커서 공유**, 가변 길이 지원.  
- **주의**: 조인 키에 **인덱스** 필수. 테이블 함수 조인은 **HASH JOIN** 경향 → 통계/플랜 확인.

### 2.2 **GTT(글로벌 임시 테이블) 조인** (수천~수만 건 키)

```sql
-- 1) 세션 전용 임시 테이블
CREATE GLOBAL TEMPORARY TABLE key_filter (key VARCHAR2(10)) ON COMMIT DELETE ROWS;

-- 2) 애플리케이션/배치에서 대량 키 벌크 인서트
-- 3) 정적 SQL로 조인
SELECT /* gtt */ s.sale_id, s.amount
FROM   sales s
JOIN   key_filter k ON k.key = s.region
WHERE  s.order_dt BETWEEN :d1 AND :d2;
```

- **장점**: **아주 큰 IN-list**도 효율적, **인덱스**(임시 테이블)·통계 전략 가능.  
- **주의**: 라이프사이클(세션/커밋) 관리, 임시 세그먼트 I/O 고려.

> **선택 기준**  
> - 수십~수백건: **스키마 타입 + 배열 바인드**  
> - 수천~수만건: **GTT 조인**(또는 영속 “필터 테이블” + 배치 처리)

---

## 3) **체크 조건** 적용이 가변적인 경우 (옵셔널 필터) — 정적 SQL로

> **문제**: `region`, `status`, `기간` 등이 **있을 때만** 필터하고, 없으면 전체.  
> **나쁜 패턴**: `(col = :p OR :p IS NULL)` — 간단하지만 **SARGability**가 약하다.  
> **정석 패턴**: **가드된 `UNION ALL` 분기** 또는 **가드된 인라인뷰**.

### 3.1 `UNION ALL` 가드 분기 (텍스트는 1문장, 분기 플랜 명확)

```sql
-- 바인드 플래그로 "어떤 조건이 활성인지"를 지정
-- :use_r, :use_s, :use_d 는 0/1
SELECT /* static-guard */ SUM(amount) tot_amt
FROM (
  SELECT /* r+d */ amount
  FROM sales
  WHERE :use_r=1 AND :use_d=1
    AND region  = :r
    AND order_dt>=:d1 AND order_dt<:d2
  UNION ALL
  SELECT /* r only */ amount
  FROM sales
  WHERE :use_r=1 AND :use_d=0
    AND region = :r
  UNION ALL
  SELECT /* d only */ amount
  FROM sales
  WHERE :use_r=0 AND :use_d=1
    AND order_dt>=:d1 AND order_dt<:d2
  UNION ALL
  SELECT /* none */ amount
  FROM sales
  WHERE :use_r=0 AND :use_d=0
) x;
```

- **특징**: 각 분기에서 **불필요 조건이 완전히 사라짐** → 인덱스 경로 **선명**.  
- **실행**: 런타임엔 가드(:use_*)가 **단 하나의 분기만 참**이 되도록 값 세팅 → 나머지 분기는 **0 rows**(빠르게 short-circuit).

### 3.2 가드 인라인뷰 (필요한 조건만 살아있는 뷰)

```sql
-- WHERE 조건을 '가드 테이블'에 위임
WITH guard AS (
  SELECT :use_r AS use_r, :use_d AS use_d FROM dual
)
SELECT /* static-guard-view */ SUM(s.amount)
FROM   sales s
JOIN   guard g ON 1=1
WHERE  (g.use_r=0 OR s.region  = :r)
AND    (g.use_d=0 OR (s.order_dt>=:d1 AND s.order_dt<:d2));
```

- **장점**: SQL 구조가 간결.  
- **주의**: 여전히 OR이 있으므로, 데이터량이 크면 3.1 패턴(UNION ALL)이 더 안정적인 경우가 많다.

---

## 4) `SELECT`-list 자체가 **동적으로 바뀌는** 경우 — 정적 SQL로

> **목표**: “보여줄 컬럼이 가변”해도 SQL 텍스트는 고정.  
> **핵심**: “열 개수/이름이 변하는” 것이 아니라, **고정된 열 집합**을 항상 반환하고 **값만 조건부로 채우거나 숨긴다**.

### 4.1 **CASE + ABSENT ON NULL** (JSON으로 유연하게)

```sql
-- JSON_OBJECT + ABSENT ON NULL: 값이 NULL이면 키 자체를 생략
SELECT /* static-select-list */
       JSON_OBJECT(
         'saleId'     VALUE sale_id,
         'region'     VALUE CASE WHEN :sel_region=1 THEN region END ABSENT ON NULL,
         'amount'     VALUE CASE WHEN :sel_amount=1 THEN amount END ABSENT ON NULL,
         'status'     VALUE CASE WHEN :sel_status=1 THEN status END ABSENT ON NULL,
         'orderDate'  VALUE CASE WHEN :sel_orderdt=1 THEN order_dt END ABSENT ON NULL
       ) AS payload
FROM   sales
WHERE  order_dt BETWEEN :d1 AND :d2;
```

- **장점**: 컬럼 **선택**을 클라이언트가 “플래그”로 제어, **열 개수 변동을 JSON에서 처리**.  
- **정적**: SQL 텍스트 불변, 커서 공유 완벽.

### 4.2 고정 열 + 조건부 값 (NULL로 마스킹)

```sql
SELECT /* static-columns */
  sale_id,
  CASE WHEN :sel_region=1  THEN region END AS region,
  CASE WHEN :sel_amount=1  THEN amount END AS amount,
  CASE WHEN :sel_status=1  THEN status END AS status,
  CASE WHEN :sel_orderdt=1 THEN order_dt END AS order_dt
FROM sales
WHERE order_dt BETWEEN :d1 AND :d2;
```

- **장점**: BI/그리드 등 **스키마 고정**이 필요한 환경에 적합.  
- **주의**: “컬럼 자체를 없애는” 건 아님. 값을 **NULL**로 만들 뿐.

---

## 5) **연산자**(=, <, <=, BETWEEN, LIKE …)가 **바뀌는** 경우 — 정적 SQL로

> **목표**: “연산자 종류”가 달라도 텍스트는 고정, 인덱스 사용성은 유지.

### 5.1 **정규화된 범위 표현**(최강 패턴)

```sql
-- 하나의 범위로 축약: [low, high)
-- = :v        → low=:v,     high=:v + ε (또는 동일 값 상한) 
-- < :v        → low=-∞,     high=:v
-- <= :v       → low=-∞,     high=:v + ε
-- > :v        → low=:v + ε, high=+∞
-- BETWEEN a,b → low=:a,     high=:b + ε
SELECT /* normalized-range */
       COUNT(*)
FROM   sales
WHERE  order_dt >= :low
AND    order_dt <  :high;
```

- **사용법**: 애플리케이션에서 **연산자에 맞게** `:low/:high` 값을 계산해 바인딩(텍스트 불변).  
- **장점**: **인덱스 범위 스캔**으로 통일, **플랜 안정성** 극대화.  
- **Tip**: DATE 타입은 `high = :v + 1`(일 단위), TIMESTAMP는 **마이크로초** 정밀도 고려.

### 5.2 LIKE vs = (접두사 검색을 범위로 정규화)

```sql
-- LIKE 'ABC%'  →  [ 'ABC', 'ABC' || highest_char ]
SELECT /* like->range */
       COUNT(*)
FROM   customers
WHERE  name >= :p
AND    name <  :p||CHR(UNICODE_MAX);  -- 환경에 맞는 최댓값 사용
```

- **장점**: LIKE를 **인덱스 범위 탐색**으로 정규화.  
- **주의**: 캐릭터셋/정렬 규칙(NLS_SORT)에 따라 상한 계산을 안전하게 구현.

### 5.3 “여러 연산자 중 하나” — **가드 분기 + UNION ALL**

```sql
-- :op = 'EQ'|'LT'|'LE'|'GE'|'GT'|'BETWEEN'
SELECT /* static-op */ COUNT(*)
FROM (
  SELECT sale_id FROM sales
   WHERE :op='EQ' AND order_dt = :v
  UNION ALL
  SELECT sale_id FROM sales
   WHERE :op='LT' AND order_dt < :v
  UNION ALL
  SELECT sale_id FROM sales
   WHERE :op='LE' AND order_dt <= :v
  UNION ALL
  SELECT sale_id FROM sales
   WHERE :op='GT' AND order_dt > :v
  UNION ALL
  SELECT sale_id FROM sales
   WHERE :op='GE' AND order_dt >= :v
  UNION ALL
  SELECT sale_id FROM sales
   WHERE :op='BETWEEN' AND order_dt BETWEEN :v1 AND :v2
);
```

- **장점**: 각 분기에서 **인덱스 SARG** 보존.  
- **실행**: 런타임에 **단 하나의 분기만 참**이 되도록 :op 바인딩.

---

## 6) 언어별 구현 패턴 (정적 SQL 보존)

### 6.1 Java — **정적 SQL + 가드/범위 바인드**

```java
// 1) IN-list(≤3) : 고정 슬롯
String sql1 =
  "SELECT /* static-inlist */ COUNT(*) FROM sales " +
  "WHERE (region=:r1 OR region=:r2 OR region=:r3) " +
  "  AND (:r1 IS NOT NULL OR :r2 IS NOT NULL OR :r3 IS NOT NULL) " +
  "  AND order_dt BETWEEN :d1 AND :d2";

// 2) 연산자 정규화: [low, high)
String sql2 =
  "SELECT /* normalized-range */ COUNT(*) FROM sales " +
  "WHERE order_dt >= :low AND order_dt < :high";

// 3) SELECT-list 가변: 값 마스킹
String sql3 =
  "SELECT sale_id, " +
  "CASE WHEN :sel_region=1 THEN region END region, " +
  "CASE WHEN :sel_amount=1 THEN amount END amount " +
  "FROM sales WHERE order_dt BETWEEN :d1 AND :d2";
```

> **주의**: 실제 JDBC 바인딩은 자리표시자 `?` 를 쓰고 순서대로 세팅. 위 예시는 가독성용.

### 6.2 Python (oracledb) — **배열 바인드 타입**

```python
# 스키마 타입 t_vc10 사용 전제
sql = """
SELECT /* array-bind */ s.sale_id, s.amount
FROM sales s JOIN TABLE(:regions) r
ON s.region = r.COLUMN_VALUE
WHERE s.order_dt BETWEEN :d1 AND :d2
"""
regions = ['APAC','EMEA','AMER']  # 가변
cur.setinputsizes(regions=oracledb.DB_TYPE_VARCHAR)  # 드라이버/버전에 따라 설정
cur.execute(sql, regions=regions, d1=date(2025,10,1), d2=date(2025,10,31))
```

---

## 7) 성능/안정성 논의 요약

| 요구 | 권장 정적 기법 | 장점 | 주의 |
|---|---|---|---|
| IN-list 가변(소수) | **고정 슬롯 + NULL 패딩** | 가장 간단, 완전 정적 | 슬롯 수를 현실적으로 설정 |
| IN-list 가변(대량) | **스키마 타입 배열 바인드** | 정적+가변 길이 동시 만족 | 테이블 함수 조인 통계/플랜 확인 |
| IN-list 초대량 | **GTT 조인** | 수천~수만 키 효율 | 세션/인덱스/통계 관리 |
| 체크 조건 가변 | **UNION ALL 가드 분기** | 인덱스/SARG 확실 | 분기 수 과다 시 관리 복잡 |
| SELECT-list 가변 | **CASE/JSON ABSENT ON NULL** | 텍스트 고정, 커서 공유 | 소비측에서 NULL/JSON 처리 |
| 연산자 가변 | **범위 정규화** / **가드 분기** | 플랜 안정, 인덱스 스캔 | 경계값(시간/문자) 정확히 |

---

## 8) 자주 하는 실수와 교정

1) **(col=:p OR :p IS NULL) 남발** → 대용량에서 느림  
   - **교정**: **UNION ALL 가드**, 또는 **동적 WHERE**(필요시). 정적만 고집하면 **가드 분기**가 무난.

2) **IN-list 문자열 붙이기**(동적 SQL)  
   - **교정**: **스키마 타입 + 배열 바인드** 또는 **GTT**.

3) **연산자별로 동적 SQL 생성**  
   - **교정**: **범위 정규화**로 연산자 의미를 [low, high)로 환원.

4) **열 수가 바뀌는 SELECT**  
   - **교정**: **고정 컬럼 + CASE NULL** 또는 **JSON ABSENT ON NULL**.

---

## 9) 미니 벤치 가이드 (무지성 튜닝 방지)

- **지표**: `consistent gets`, `physical reads`, `elapsed`, `cpu`, XPLAN의 Access/Filter Predicates  
- **케이스**:  
  - IN-list 크기 {1, 3, 10, 100, 1000}  
  - 체크 조건 on/off 조합  
  - 연산자 {=, <, <=, >, >=, BETWEEN, LIKE 'ABC%'}  
- **기대**:  
  - **UNION ALL 가드**와 **범위 정규화**가 **라인 통계**에서 안정적으로 우세.  
  - **배열 바인드/GTT** 는 IN-list 규모가 커질수록 유리.

> 간단한 카드 모델: 선택도 $$ s = \prod_i s_i $$ 로 보고,  
> **OR 결합**은 옵티마이저가 보수적으로 잡아 **s ↑** → 쓸데없는 범위 확대 가능.  
> 반면 **가드 분기**는 “불필요 조건이 소거”되어 **정확한 s** 확보가 쉽다.

---

## 10) 결론

- “가변성” 대부분은 **정적 SQL**로도 표현 가능하다.  
- 핵심은  
  1) **값은 바인드**,  
  2) **SARGability 보존**(범위 정규화/가드 분기/인덱스 경로 명시),  
  3) **규모별 패턴 선택**(소수=슬롯, 대량=배열 바인드, 초대량=GTT),  
  4) **SELECT-list/연산자 가변성**은 **CASE/JSON/범위 환원**으로 처리.  
- 이 원칙을 따르면 **커서 공유**·**플랜 안정**·**성능 예측성**을 모두 얻을 수 있다.

> **한 줄 요약**  
> *SQL 텍스트는 고정*, *값은 바인드*, *의미는 가드와 범위로 환원* — 이것이 “정적 SQL로 가변 요구를 품는 법”의 전부다.