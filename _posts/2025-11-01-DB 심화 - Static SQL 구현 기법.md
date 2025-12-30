---
layout: post
title: DB 심화 - Static SQL 구현 기법
date: 2025-11-01 15:25:23 +0900
category: DB 심화
---
# 정적 SQL을 활용한 동적 요구사항 처리 기법

> **핵심 메시지**  
> 동적 SQL 없이도 대부분의 가변적 요구사항을 처리할 수 있습니다. 비결은 (1) 바인드 변수를 활용하고, (2) 정규화된 패턴을 적용하며, (3) 옵티마이저 친화적인 조건을 유지하는 것입니다.

---

## 실습 환경 구성

```sql
-- 판매 테이블 생성 (200만 행 가정)
DROP TABLE sales PURGE;

CREATE TABLE sales (
  sale_id     NUMBER PRIMARY KEY,
  region      VARCHAR2(10),      -- 'APAC', 'EMEA', 'AMER', 'OTHER'
  order_dt    DATE,
  status      VARCHAR2(10),      -- 'OK', 'CANCEL'
  amount      NUMBER(12,2),
  customer_id NUMBER
);

-- 샘플 데이터 삽입
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

-- 인덱스 생성 및 통계 수집
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

## IN-list 항목이 가변적이지만 항목 수가 적은 경우

**상황**: `region IN (...)`과 같이 조건이 가변적이지만, 최대 3-5개 정도로 항목 수가 제한된 경우입니다.  
**목표**: SQL 텍스트는 정적이면서 바인드 변수 재사용과 인덱스 범위 스캔을 보장하는 것입니다.

### 고정 슬롯과 NULL 패딩 방식 (가장 단순한 완전 정적 방법)

```sql
-- 최대 3개 지역까지 선택 가능하다고 가정
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

**핵심 포인트**:
- IN 대신 **OR 3개**를 사용하는 이유는 옵티마이저가 `region` 결합 인덱스를 더 안정적으로 range-scan 할 수 있기 때문입니다.
- **불필요한 슬롯에는 NULL**을 바인딩하여 조건 자체가 거짓이 되도록 합니다.
- SQL 텍스트가 100% 고정되어 **커서 공유율 100%**를 달성합니다.

### 고정 IN 목록과 센티널 값 방식 (정수 또는 숫자 키인 경우)

```sql
-- 숫자 키의 경우 "존재할 수 없는 값"으로 패딩
SELECT /* static-inlist(≤5) */
       COUNT(*)
FROM   customers
WHERE  customer_id IN (:c1, :c2, :c3, :c4, :c5)
AND    signup_dt >= :d1
AND    signup_dt <  :d2;
```

**구현 방법**: 사용하지 않는 자리에는 **-1** 등 실제 데이터 도메인과 충돌하지 않는 센티널 값을 바인딩합니다.  
**장점**: 간단하고 완전히 정적입니다.  
**주의사항**: 실제 데이터 도메인과 충돌하지 않도록 센티널 값을 신중하게 선택해야 합니다.

---

## IN-list 항목이 가변적이고 항목 수가 많은 경우

**상황**: 선택된 ID나 코드가 수십 개에서 수천 개에 이르는 경우입니다.  
**목표**: SQL 텍스트는 정적이고 커서 공유를 유지하면서 대량 키 필터를 효율적으로 처리하는 것입니다.

### 스키마 타입과 배열 바인드 방식 (정적 SQL 유지)

```sql
-- 1) 스키마 타입 생성 (한 번만 실행)
CREATE OR REPLACE TYPE t_vc10 AS TABLE OF VARCHAR2(10);

-- 2) 정적 SQL: TABLE(:list) 사용
SELECT /* array-bind */
       s.sale_id, s.region, s.amount
FROM   sales s
JOIN   TABLE(:regions) r
  ON   s.region = r.COLUMN_VALUE
WHERE  s.order_dt BETWEEN :d1 AND :d2;
```

**PL/SQL 실행 예시**:
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

**장점**:
- **정적 SQL과 커서 공유**를 완전히 유지합니다.
- 가변 길이 리스트를 완벽하게 지원합니다.

**주의사항**:
- 조인 키에 **인덱스가 필수**입니다.
- 테이블 함수 조인은 **HASH JOIN** 경향이 있으므로 통계와 실행 계획을 확인해야 합니다.

### GTT(Global Temporary Table) 조인 방식 (수천 개에서 수만 개의 키 처리)

```sql
-- 1) 세션 전용 임시 테이블 생성
CREATE GLOBAL TEMPORARY TABLE key_filter (key VARCHAR2(10)) ON COMMIT DELETE ROWS;

-- 2) 애플리케이션 또는 배치에서 대량 키를 벌크 인서트
-- 3) 정적 SQL로 조인
SELECT /* gtt */ s.sale_id, s.amount
FROM   sales s
JOIN   key_filter k ON k.key = s.region
WHERE  s.order_dt BETWEEN :d1 AND :d2;
```

**장점**:
- **매우 큰 IN-list**도 효율적으로 처리할 수 있습니다.
- 임시 테이블에 **인덱스**를 생성하고 통계 전략을 활용할 수 있습니다.

**주의사항**:
- 임시 테이블의 라이프사이클(세션/커밋)을 관리해야 합니다.
- 임시 세그먼트 I/O를 고려해야 합니다.

**선택 기준**:
- **수십~수백 개**: 스키마 타입과 배열 바인드 방식
- **수천~수만 개**: GTT 조인 방식

---

## 조건부 필터링을 정적 SQL로 처리

**문제**: `region`, `status`, 기간 등이 있을 때만 필터링하고, 없으면 전체 데이터를 처리해야 합니다.  
**비효율적 패턴**: `(col = :p OR :p IS NULL)` - 간단하지만 SARGability가 약합니다.  
**권장 패턴**: **가드된 `UNION ALL` 분기** 또는 **가드된 인라인 뷰**.

### UNION ALL 가드 분기 방식 (텍스트는 단일 문장, 분기별 실행 계획 명확)

```sql
-- 바인드 플래그로 "어떤 조건이 활성화되었는지" 지정
-- :use_r, :use_s, :use_d는 0 또는 1
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

**특징**: 각 분기에서 **불필요한 조건이 완전히 제거**되어 인덱스 접근 경로가 명확합니다.  
**실행**: 런타임에 가드 조건(:use_*)이 **단 하나의 분기만 참**이 되도록 값을 설정합니다. 나머지 분기는 **0행**(빠르게 short-circuit)을 반환합니다.

### 가드 인라인 뷰 방식 (필요한 조건만 살아있는 뷰)

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

**장점**: SQL 구조가 간결합니다.  
**주의**: 여전히 OR 조건이 있으므로, 데이터 양이 많으면 UNION ALL 패턴이 더 안정적인 경우가 많습니다.

---

## SELECT 리스트가 동적으로 변경되는 경우

**목표**: "보여줄 컬럼이 가변적"이더라도 SQL 텍스트는 고정된 상태를 유지합니다.  
**핵심**: "열 개수나 이름이 변하는" 것이 아니라, **고정된 열 집합**을 항상 반환하고 **값만 조건부로 채우거나 숨기는** 방식입니다.

### CASE와 ABSENT ON NULL 방식 (JSON을 활용한 유연한 방법)

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

**장점**: 컬럼 선택을 클라이언트가 "플래그"로 제어할 수 있으며, **열 개수 변동을 JSON에서 처리**합니다.  
**정적**: SQL 텍스트가 불변하므로 커서 공유가 완벽합니다.

### 고정 열과 조건부 값 방식 (NULL로 마스킹)

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

**장점**: BI/그리드 등 **스키마가 고정**되어야 하는 환경에 적합합니다.  
**주의**: "컬럼 자체를 제거하는" 것이 아니라 값을 **NULL**로 만드는 방식입니다.

---

## 연산자가 동적으로 변경되는 경우

**목표**: "연산자 종류"가 달라도 SQL 텍스트는 고정하고, 인덱스 사용성은 유지합니다.

### 정규화된 범위 표현 방식 (가장 강력한 패턴)

```sql
-- 하나의 범위로 통합: [low, high)
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

**사용법**: 애플리케이션에서 **연산자에 맞게** `:low/:high` 값을 계산하여 바인딩합니다(텍스트는 불변).  
**장점**: **인덱스 범위 스캔**으로 통일되어 **실행 계획 안정성**이 극대화됩니다.  
**팁**: DATE 타입은 `high = :v + 1`(일 단위), TIMESTAMP는 **마이크로초** 정밀도를 고려해야 합니다.

### LIKE를 범위로 정규화하는 방식

```sql
-- LIKE 'ABC%'  →  [ 'ABC', 'ABC' || highest_char ]
SELECT /* like->range */
       COUNT(*)
FROM   customers
WHERE  name >= :p
AND    name <  :p||CHR(UNICODE_MAX);  -- 환경에 맞는 최댓값 사용
```

**장점**: LIKE를 **인덱스 범위 탐색**으로 정규화합니다.  
**주의**: 캐릭터셋과 정렬 규칙(NLS_SORT)에 따라 상한 계산을 안전하게 구현해야 합니다.

### "여러 연산자 중 하나" 처리 방식 — 가드 분기와 UNION ALL

```sql
-- :op = 'EQ','LT','LE','GE','GT','BETWEEN'
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

**장점**: 각 분기에서 **인덱스 SARG**가 보존됩니다.  
**실행**: 런타임에 **단 하나의 분기만 참**이 되도록 :op를 바인딩합니다.

---

## 프로그래밍 언어별 구현 패턴

### Java — 정적 SQL과 가드/범위 바인드

```java
// 1) IN-list(≤3) : 고정 슬롯 방식
String sql1 =
  "SELECT /* static-inlist */ COUNT(*) FROM sales " +
  "WHERE (region=:r1 OR region=:r2 OR region=:r3) " +
  "  AND (:r1 IS NOT NULL OR :r2 IS NOT NULL OR :r3 IS NOT NULL) " +
  "  AND order_dt BETWEEN :d1 AND :d2";

// 2) 연산자 정규화: [low, high) 범위 방식
String sql2 =
  "SELECT /* normalized-range */ COUNT(*) FROM sales " +
  "WHERE order_dt >= :low AND order_dt < :high";

// 3) SELECT-list 가변: 값 마스킹 방식
String sql3 =
  "SELECT sale_id, " +
  "CASE WHEN :sel_region=1 THEN region END region, " +
  "CASE WHEN :sel_amount=1 THEN amount END amount " +
  "FROM sales WHERE order_dt BETWEEN :d1 AND :d2";
```

### Python — 배열 바인드 타입

```python
# 스키마 타입 t_vc10 사용 전제
import oracledb

sql = """
SELECT /* array-bind */ s.sale_id, s.amount
FROM sales s JOIN TABLE(:regions) r
ON s.region = r.COLUMN_VALUE
WHERE s.order_dt BETWEEN :d1 AND :d2
"""

regions = ['APAC','EMEA','AMER']  # 가변 리스트
cur.setinputsizes(regions=oracledb.DB_TYPE_VARCHAR)  # 드라이버/버전에 따라 설정
cur.execute(sql, regions=regions, d1=date(2025,10,1), d2=date(2025,10,31))
```

---

## 성능과 안정성 고려사항

| 요구사항 | 권장 정적 기법 | 주요 장점 | 주의사항 |
|---|---|---|---|
| IN-list 가변(소수) | **고정 슬롯 + NULL 패딩** | 가장 간단하고 완전 정적 | 슬롯 수를 현실적으로 설정 |
| IN-list 가변(대량) | **스키마 타입 배열 바인드** | 정적과 가변 길이 동시 만족 | 테이블 함수 조인 통계와 실행 계획 확인 |
| IN-list 초대량 | **GTT 조인** | 수천~수만 개 키 효율적 처리 | 세션 관리, 인덱스, 통계 관리 |
| 조건부 필터링 | **UNION ALL 가드 분기** | 인덱스/SARG 확실 보장 | 분기 수가 많아지면 관리 복잡 |
| SELECT-list 가변 | **CASE/JSON ABSENT ON NULL** | 텍스트 고정, 커서 공유 완벽 | 클라이언트에서 NULL/JSON 처리 필요 |
| 연산자 가변 | **범위 정규화** / **가드 분기** | 실행 계획 안정, 인덱스 스캔 | 경계값(시간/문자) 정확한 계산 필요 |

---

## 일반적인 실수와 개선 방법

### 1. `(col=:p OR :p IS NULL)` 패턴의 과도한 사용
**문제**: 대용량 데이터에서 성능이 저하됩니다.  
**개선**: **UNION ALL 가드 분기** 또는 **동적 WHERE 절**을 사용하세요. 정적 SQL만 고집한다면 **가드 분기**가 무난한 선택입니다.

### 2. IN-list 문자열 동적 생성
**문제**: 하드 파싱 증가와 SQL 인젝션 위험이 있습니다.  
**개선**: **스키마 타입 + 배열 바인드** 또는 **GTT** 방식을 사용하세요.

### 3. 연산자별 동적 SQL 생성
**문제**: 실행 계획 캐시 효율성이 떨어집니다.  
**개선**: **범위 정규화**로 연산자 의미를 [low, high) 범위로 통일하세요.

### 4. 열 수가 변하는 SELECT 구문
**문제**: 클라이언트 코드가 복잡해집니다.  
**개선**: **고정 컬럼 + CASE NULL** 또는 **JSON ABSENT ON NULL** 방식을 사용하세요.

---

## 성능 측정 가이드라인

성능 튜닝은 무분별한 변경보다 체계적인 측정을 바탕으로 해야 합니다.

### 주요 측정 지표
- `consistent gets`: 논리적 읽기 횟수
- `physical reads`: 물리적 읽기 횟수
- `elapsed time`: 총 소요 시간
- `cpu time`: CPU 사용 시간
- 실행 계획(XPLAN)의 Access/Filter Predicates

### 테스트 케이스 설계
- IN-list 크기: {1, 3, 10, 100, 1000}
- 조건부 필터링: on/off 조합
- 연산자 유형: {=, <, <=, >, >=, BETWEEN, LIKE 'ABC%'}

### 예상 결과
- **UNION ALL 가드**와 **범위 정규화**가 일관된 성능 지표로 우수한 결과를 보일 것입니다.
- **배열 바인드/GTT**는 IN-list 규모가 커질수록 더욱 유리해집니다.

---

## 결론

정적 SQL을 활용한 동적 요구사항 처리는 성능, 보안, 유지보수성 측면에서 많은 이점을 제공합니다. 핵심 원칙은 다음과 같습니다:

1. **SQL 텍스트는 고정**하고, **값만 바인드 변수**로 처리합니다.
2. **SARGability를 보존**하여 옵티마이저가 최적의 실행 계획을 선택할 수 있도록 합니다.
3. **규모에 적합한 패턴을 선택**합니다:
   - 소수 항목: 고정 슬롯 방식
   - 중간 규모: 배열 바인드 방식
   - 대규모: GTT 조인 방식
4. **SELECT 리스트와 연산자 가변성**은 CASE/JSON/범위 정규화로 처리합니다.

이러한 원칙을 따르면 **커서 공유**, **실행 계획 안정성**, **성능 예측성**을 모두 확보할 수 있습니다. 마지막으로, 모든 성능 개선은 실제 환경에서의 측정과 검증을 통해 이루어져야 합니다. 정적 SQL은 강력한 도구이지만, 올바르게 적용되었는지 지속적으로 모니터링하고 최적화해야 지속적인 성능 향상을 기대할 수 있습니다.