---
layout: post
title: DB 심화 - OUTER 조인을 INNER 조인으로 변환
date: 2025-11-21 16:25:23 +0900
category: DB 심화
---
# OUTER 조인을 INNER 조인으로 변환: 원리와 실전 가이드

## 핵심 개념: OUTER 조인 vs INNER 조인

### 기본 이해
- **LEFT OUTER JOIN**: 왼쪽 테이블의 모든 행을 보존하며, 오른쪽 테이블에 매칭되는 행이 없을 경우 오른쪽 컬럼들을 NULL로 채웁니다.
- **INNER JOIN**: 양쪽 테이블 모두에서 매칭되는 행만 결과에 포함시킵니다.

### 변환 가능성의 핵심 조건
특정 조건 하에서는 OUTER 조인이 INNER 조인과 논리적으로 동일한 결과를 생성합니다. 이 변환은 성능 최적화에 중요한 기회를 제공합니다. 주요 변환 조건은 다음과 같습니다.

1. **NULL-배제 조건의 존재**: WHERE 절에 오른쪽 테이블 컬럼에 대한 조건이 있고, 이 조건이 NULL 값을 거부하는 경우
2. **참조 무결성 보장**: 외래키(FK) 제약이 활성화되어 있고, FK 컬럼이 NOT NULL이며, 참조 대상이 PK/Unique인 경우
3. **존재성만 확인하는 집계**: COUNT() > 0 같은 패턴으로 실제로는 존재 여부만 확인하는 경우

## 실습 환경 설정

```sql
-- 기본 테이블 생성
CREATE TABLE d_customer (
  cust_id  NUMBER PRIMARY KEY,
  region   VARCHAR2(8) NOT NULL,
  tier     VARCHAR2(8) NOT NULL
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
  amount   NUMBER(12,2) NOT NULL,
  CONSTRAINT fk_sales_customer FOREIGN KEY (cust_id) REFERENCES d_customer(cust_id),
  CONSTRAINT fk_sales_product FOREIGN KEY (prod_id) REFERENCES d_product(prod_id)
);

-- 샘플 데이터
INSERT INTO d_customer VALUES (1,'KOR','VIP');
INSERT INTO d_customer VALUES (2,'USA','STD');
INSERT INTO d_product VALUES (10,'ELEC','B0');
INSERT INTO d_product VALUES (20,'HOME','B1');
INSERT INTO f_sales VALUES (101,1,10,DATE '2025-02-01',1,10000);
INSERT INTO f_sales VALUES (102,1,20,DATE '2025-02-03',2,15000);
INSERT INTO f_sales VALUES (103,2,10,DATE '2025-02-10',1,9000);
COMMIT;
```

## 변환 규칙 1: NULL-배제 조건이 있는 경우

### NULL-배제 조건이란?
NULL 값이 조건에 포함되면 결과가 FALSE가 되는 조건을 말합니다.
- **NULL-배제 예**: `col = 값`, `col > 값`, `col IN (값들)`, `col LIKE '패턴'`, `col IS NOT NULL`
- **NULL-허용 예**: `col IS NULL`, `NVL(col, '기본값') = '기본값'`

### 변환 예시
```sql
-- 원본 (LEFT OUTER JOIN + WHERE 조건)
SELECT c.cust_id, p.brand
FROM   d_customer c
LEFT  JOIN d_product p ON p.prod_id IN (10,20)
WHERE  p.category = 'ELEC';  -- NULL-배제 조건

-- 변환 후 (INNER JOIN)
SELECT c.cust_id, p.brand
FROM   d_customer c
JOIN   d_product p ON p.prod_id IN (10,20)
WHERE  p.category = 'ELEC';
```

**변환 이유**: LEFT JOIN 결과에서 매칭되지 않은 행은 p.category가 NULL입니다. `p.category = 'ELEC'` 조건은 NULL에 대해 FALSE를 반환하므로, 이러한 행들은 자동으로 필터링됩니다. 결과적으로 INNER JOIN과 동일한 결과가 됩니다.

### 실전 주의사항
**자주 발생하는 버그 패턴**:
```sql
-- 잘못된 예: OUTER 조인의 의미가 손실됨
SELECT c.cust_id, s.sales_id
FROM   d_customer c
LEFT  JOIN f_sales s ON s.cust_id = c.cust_id
WHERE  s.sales_dt >= DATE '2025-02-01';  -- OUTER 의미 손실

-- 올바른 예: OUTER 의미 유지
SELECT c.cust_id, s.sales_id
FROM   d_customer c
LEFT  JOIN f_sales s 
       ON s.cust_id = c.cust_id 
      AND s.sales_dt >= DATE '2025-02-01';  -- 조건을 ON 절에 유지
```

## 변환 규칙 2: 참조 무결성으로 인한 매칭 보장

### 자식 → 부모 방향 조인
자식 테이블에서 부모 테이블로의 OUTER 조인은 항상 매칭이 존재하므로 INNER 조인으로 안전하게 변환할 수 있습니다.

```sql
-- 원본 (불필요한 OUTER 조인)
SELECT s.sales_id, c.region
FROM   f_sales s
LEFT  JOIN d_customer c ON c.cust_id = s.cust_id;

-- 변환 후 (효율적인 INNER 조인)
SELECT s.sales_id, c.region
FROM   f_sales s
JOIN   d_customer c ON c.cust_id = s.cust_id;
```

**조건**: 
1. `f_sales.cust_id`가 NOT NULL 컬럼
2. `f_sales.cust_id`가 `d_customer.cust_id`를 참조하는 외래키
3. 외래키 제약이 ENABLE VALIDATE 상태

### 부모 → 자식 방향 조인
반대 방향의 경우 주의가 필요합니다. 부모 테이블에 자식이 없는 행을 포함하려면 OUTER 조인을 유지해야 합니다.

```sql
-- OUTER 조인 유지가 필요한 경우
SELECT c.cust_id, COUNT(s.sales_id) as sales_count
FROM   d_customer c
LEFT  JOIN f_sales s ON s.cust_id = c.cust_id  -- 매출 없는 고객도 포함
GROUP BY c.cust_id;
```

## 변환 규칙 3: 존재성만 확인하는 집계 패턴

### 비효율적인 OUTER + COUNT 패턴
```sql
-- 원본 (비효율적)
SELECT c.cust_id,
       CASE WHEN COUNT(s.sales_id) > 0 THEN 'ACTIVE' ELSE 'NEW' END AS status
FROM   d_customer c
LEFT  JOIN f_sales s ON s.cust_id = c.cust_id
GROUP BY c.cust_id;
```

### 최적화된 EXISTS 패턴
```sql
-- 변환 후 (효율적)
SELECT c.cust_id,
       CASE WHEN EXISTS (SELECT 1 FROM f_sales s WHERE s.cust_id = c.cust_id)
            THEN 'ACTIVE' ELSE 'NEW' END AS status
FROM   d_customer c;
```

**장점**:
1. 조인 대신 세미조인(SEMI JOIN) 사용으로 메모리 사용 감소
2. GROUP BY 제거로 처리 비용 절감
3. 실행 계획이 더 간단하고 예측 가능

## 변환 성능 이점 분석

### 1. 조인 순서 자유도 증가
OUTER 조인은 보존측 테이블이 먼저 처리되어야 하는 제약이 있습니다. INNER 조인으로 변환하면 옵티마이저가 더 많은 조인 순서 후보를 고려할 수 있습니다.

```sql
-- OUTER 조인: c가 항상 드라이빙 테이블
SELECT /*+ LEADING(c s) */ c.cust_id, s.sales_id
FROM   d_customer c
LEFT  JOIN f_sales s ON s.cust_id = c.cust_id
WHERE  s.amount > 5000;

-- INNER 조인: s가 드라이빙 테이블이 될 수 있음
SELECT /*+ LEADING(s c) */ c.cust_id, s.sales_id
FROM   d_customer c
JOIN   f_sales s ON s.cust_id = c.cust_id
WHERE  s.amount > 5000;
```

### 2. 조건절 푸시다운 활성화
INNER 조인에서는 조건절을 더 깊은 단계로 푸시다운할 수 있어 불필요한 데이터 읽기를 줄일 수 있습니다.

### 3. 전이성 최적화 적용
조인 조건의 등치성을 이용한 전이성 최적화가 INNER 조인에서 더 적극적으로 적용됩니다.

## 실전 트러블슈팅 시나리오

### 시나리오 1: 갑작스런 성능 저하
**증상**: OUTER 조인을 사용하는 쿼리의 성능이 특정 시점부터 저하됨

**진단 단계**:
1. 실행 계획 비교: OUTER 조인이 INNER 조인으로 자동 변환되었는지 확인
2. 통계 갱신 여부 확인: 통계 변경으로 인한 변환 결정 변화 가능성
3. 데이터 분포 변화: NULL-배제 조건의 선택도 변화 확인

### 시나리오 2: 결과 건수 불일치
**증상**: OUTER 조인 쿼리의 결과 건수가 예상과 다름

**원인 분석**:
1. WHERE 절에 오른쪽 테이블의 NULL-배제 조건 존재
2. 참조 무결성 제약 변경
3. 데이터 품질 문제로 인한 FK 위반

**해결책**:
```sql
-- 문제 진단용 쿼리
SELECT 
  COUNT(*) as total_rows,
  COUNT(s.sales_id) as sales_exists,
  COUNT(CASE WHEN s.sales_id IS NULL THEN 1 END) as sales_null
FROM d_customer c
LEFT JOIN f_sales s ON s.cust_id = c.cust_id;
```

## 고급 활용 패턴

### 동적 조건 처리를 위한 안전한 패턴
파라미터 값에 따라 OUTER/INNER 의미를 유연하게 전환하는 패턴:

```sql
SELECT c.cust_id, s.sales_id
FROM   d_customer c
LEFT  JOIN f_sales s
  ON   s.cust_id = c.cust_id
 AND  (:start_date IS NULL OR s.sales_dt >= :start_date)
 AND  (:end_date IS NULL OR s.sales_dt < :end_date);
```

**장점**:
- 파라미터가 NULL일 때: 모든 고객 표시 (OUTER 의미)
- 파라미터에 값이 있을 때: 조건에 맞는 매출만 연결 (조건부 INNER)
- WHERE 절로 조건 이동 시 발생하는 의미 변화 방지

### 보고서 쿼리 최적화 패턴
```sql
-- 비효율적인 리포트 쿼리
SELECT c.cust_id, 
       SUM(s.amount) as total_amount,
       COUNT(s.sales_id) as sales_count
FROM   d_customer c
LEFT  JOIN f_sales s ON s.cust_id = c.cust_id
GROUP BY c.cust_id;

-- 최적화된 리포트 쿼리 (별도 집계 후 조인)
WITH sales_agg AS (
  SELECT cust_id, 
         SUM(amount) as total_amount,
         COUNT(sales_id) as sales_count
  FROM   f_sales
  GROUP BY cust_id
)
SELECT c.cust_id,
       COALESCE(s.total_amount, 0) as total_amount,
       COALESCE(s.sales_count, 0) as sales_count
FROM   d_customer c
LEFT  JOIN sales_agg s ON s.cust_id = c.cust_id;
```

## 변환 검증 방법론

### 1. 논리적 동등성 검증
```sql
-- 변환 전/후 결과 비교
SELECT 'BEFORE' as version, COUNT(*) as row_count
FROM (
  SELECT c.cust_id, s.sales_id
  FROM d_customer c
  LEFT JOIN f_sales s ON s.cust_id = c.cust_id
  WHERE s.sales_dt >= DATE '2025-02-01'
)
UNION ALL
SELECT 'AFTER' as version, COUNT(*) as row_count
FROM (
  SELECT c.cust_id, s.sales_id
  FROM d_customer c
  JOIN f_sales s ON s.cust_id = c.cust_id
  WHERE s.sales_dt >= DATE '2025-02-01'
);
```

### 2. 실행 계획 분석
```sql
-- 상세 실행 계획 확인
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL, NULL, 'ALLSTATS LAST +PREDICATE +ALIAS +NOTE'
));
```

**분석 포인트**:
- 조인 방식 변화 확인 (OUTER → INNER)
- 접근 경로 최적화 여부
- 실제 처리 행수(A-Rows) 비교
- I/O 및 CPU 비용 감소 효과

### 3. 성능 측정
```sql
-- 성능 비교를 위한 세션 통계
ALTER SESSION SET statistics_level = ALL;

-- 변환 전 쿼리 실행
SELECT /*+ GATHER_PLAN_STATISTICS */ ...

-- 변환 후 쿼리 실행  
SELECT /*+ GATHER_PLAN_STATISTICS */ ...

-- 통계 비교
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'));
```

## 주의사항과 한계점

### FULL OUTER JOIN 변환의 어려움
FULL OUTER JOIN은 양쪽 테이블 모두를 NULL-보존하므로 INNER 조인으로의 변환은 매우 제한적입니다. 특수한 경우(상호 1:1 필수 매칭)에만 가능합니다.

### NULL-허용 조건의 함정
```sql
-- 주의: 이 조건은 NULL을 배제하지 않음
SELECT c.cust_id, s.sales_id
FROM d_customer c
LEFT JOIN f_sales s ON s.cust_id = c.cust_id
WHERE NVL(s.amount, 0) > 0;  -- NULL은 0으로 변환되어 조건 통과
```

### 옵티마이저의 자동 변환 한계
Oracle 옵티마이저는 안전한 경우에만 OUTER→INNER 변환을 수행합니다. 복잡한 조건이나 사용자 정의 함수가 포함된 경우 변환이 제한될 수 있습니다.

## 실무 적용 가이드라인

### 변환을 적용해야 하는 경우
1. WHERE 절에 오른쪽 테이블의 NULL-배제 조건이 명확히 존재할 때
2. 자식→부모 참조 관계가 명확히 보장될 때
3. 성능 문제가 발생하고 실행 계획 분석 결과 OUTER 조인이 병목일 때
4. 존재성만 확인하는 COUNT() > 0 패턴을 사용할 때

### 변환을 주의해야 하는 경우
1. NULL 보존이 비즈니스적으로 중요한 의미를 가질 때
2. 복잡한 조건식으로 NULL-배제 여부가 모호할 때
3. 동적 SQL이나 파라미터에 따라 의미가 변화할 수 있을 때
4. 레거시 시스템에서 예상치 못한 영향이 발생할 수 있을 때

### 점진적 개선 접근법
1. **1단계**: 변환 가능성 분석 (논리적 동등성 검증)
2. **2단계**: 제한적 적용 (새로운 리포트나 모듈부터 시작)
3. **3단계**: 성능 측정 및 모니터링
4. **4단계**: 점진적 확장 (안정성 확인 후 핵심 쿼리 적용)

## 마무리: 변환의 예술과 과학

OUTER 조인을 INNER 조인으로의 변환은 단순한 문법 변경이 아닌, 쿼리의 의미 체계를 깊이 이해하고 데이터 모델링의 본질을 파악하는 과정입니다. 이 변환을 성공적으로 적용하기 위해서는 다음 세 가지 차원의 이해가 필요합니다.

**첫째, 논리적 이해**: NULL-배제 조건과 참조 무결성이 어떻게 쿼리 의미를 변화시키는지 명확히 이해해야 합니다. WHERE 절의 작은 조건 하나가 전체 쿼리의 의미를 근본적으로 바꿀 수 있음을 인지하는 것이 출발점입니다.

**둘째, 성능적 통찰**: 변환이 가져올 성능 이익을 정량화할 수 있어야 합니다. 실행 계획을 읽고, 조인 순서의 자유도 증가, 조건절 푸시다운의 활성화, I/O 감소 효과 등을 체계적으로 분석할 줄 알아야 합니다.

**셋째, 실용적 지혜**: 모든 이론적 변환 가능성이 실제 적용 가능성을 의미하지는 않습니다. 레거시 시스템의 복잡성, 비즈니스 로직의 미묘함, 데이터 품질의 현실을 고려한 균형 잡힌 결정이 필요합니다.

가장 효과적인 접근법은 작은 변화부터 시작하여 체계적으로 검증하는 것입니다. 한 번에 많은 쿼리를 변경하기보다, 가장 성능 영향이 큰 핵심 쿼리부터 시작하여 변환의 효과를 측정하고, 그 경험을 바탕으로 점진적으로 확장해 나가는 것이 장기적인 성공을 보장합니다.

이 변환 기술을 마스터하는 것은 단순한 성능 최적화를 넘어, 데이터베이스 쿼리의 본질적 이해를深める 여정이 될 것입니다. 각 변환 결정이 데이터 흐름과 비즈니스 의미에 미치는 영향을 꼼꼼히 검토하며, 이 지식이 축적될 때 비로소 복잡한 시스템에서도 안정적이고 효율적인 쿼리를 설계할 수 있는 진정한 전문가로 성장할 수 있습니다.