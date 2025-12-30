---
layout: post
title: DB 심화 - Result Cache
date: 2025-11-04 21:25:23 +0900
category: DB 심화
---
# Oracle Result Cache 완전 가이드

> **핵심 요약**
> Oracle Result Cache는 "동일한 입력 → 동일한 결과" 패턴의 읽기 중심 워크로드에서 쿼리 결과나 함수 반환값을 SGA의 전용 메모리 영역에 저장하여 재사용하는 기능입니다. SQL Result Cache와 PL/SQL Function Result Cache 두 가지 주요 유형이 있으며, 의존 객체가 변경되면 자동으로 무효화되는 특징을 가지고 있습니다. 이 기능은 변경 빈도가 낮고 동일한 쿼리나 함수 호출이 반복되며 계산 비용이 큰 경우에 특히 효과적입니다.

---

## Result Cache 기본 개념

### Result Cache란 무엇인가?

Oracle Result Cache는 데이터베이스 서버 수준에서 제공하는 인메모리 캐싱 메커니즘으로, 자주 실행되는 쿼리나 함수의 결과를 메모리에 저장하여 동일한 요청이 들어왔을 때 빠르게 응답할 수 있도록 합니다. 이는 특히 다음과 같은 상황에서 효과적입니다:

1. **반복적 읽기 작업**: 동일한 쿼리가 여러 번 실행되는 경우
2. **고비용 연산**: 복잡한 계산이나 조인이 포함된 쿼리
3. **변경 빈도 낮음**: 기반 데이터가 자주 변경되지 않는 경우
4. **결과 집합이 작음**: 캐시할 결과의 크기가 적당한 경우

### 두 가지 주요 유형

1. **SQL Result Cache**
   - SELECT 쿼리 결과 집합을 캐싱
   - `/*+ RESULT_CACHE */` 힌트를 사용하거나 파라미터 설정으로 활성화

2. **PL/SQL Function Result Cache**
   - 함수 호출 결과를 캐싱
   - `RESULT_CACHE` 프라그마를 함수 정의에 추가하여 활성화

---

## 환경 설정과 기본 확인

### 필수 파라미터 설정

```sql
-- Result Cache 관련 파라미터 확인
SELECT name, value, description
FROM v$parameter
WHERE name LIKE '%result_cache%'
ORDER BY name;

-- 주요 파라미터 상세 설명
-- RESULT_CACHE_MAX_SIZE: SGA 내 Result Cache 영역의 최대 크기 (바이트 단위)
-- RESULT_CACHE_MAX_RESULT: 단일 결과가 차지할 수 있는 최대 비율 (%)
-- RESULT_CACHE_MODE: MANUAL(힌트 사용 시만) 또는 FORCE(가능하면 항상)
-- RESULT_CACHE_REMOTE_EXPIRATION: 원격 객체 결과의 유효 기간 (분 단위)
```

### 모니터링 뷰와 관리 패키지

```sql
-- 1. 전반적인 통계 확인
SELECT * FROM v$result_cache_statistics;

-- 2. 메모리 사용 현황
SELECT * FROM v$result_cache_memory;

-- 3. 캐시된 객체 목록
SELECT * FROM v$result_cache_objects;

-- 4. 의존성 정보
SELECT * FROM v$result_cache_dependency;

-- 5. 메모리 리포트 생성
SET SERVEROUTPUT ON
BEGIN
  DBMS_RESULT_CACHE.MEMORY_REPORT;
END;
/

-- 6. 캐시 상태 확인을 위한 유틸리티 쿼리
SELECT 
  '캐시 히트율' AS metric,
  ROUND((SELECT value FROM v$result_cache_statistics WHERE name = 'Find Count') * 100.0 /
        NULLIF((SELECT value FROM v$result_cache_statistics WHERE name = 'Find Count') +
               (SELECT value FROM v$result_cache_statistics WHERE name = 'Create Count Success'), 0), 2) || '%' AS value
FROM dual
UNION ALL
SELECT 
  name,
  TO_CHAR(value, '999,999,999') AS value
FROM v$result_cache_statistics
WHERE name IN ('Block Count', 'Block Size Current', 'Result Size Maximum');
```

---

## SQL Result Cache 상세 구현

### 기본 사용법

```sql
-- 기본적인 Result Cache 힌트 사용
SELECT /*+ RESULT_CACHE */ 
       department_id,
       COUNT(*) AS employee_count,
       AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 5;

-- NO_RESULT_CACHE 힌트로 캐싱 방지
SELECT /*+ NO_RESULT_CACHE */ 
       employee_id, first_name, last_name
FROM employees
WHERE department_id = :dept_id;
```

### 적절한 사용 사례 파악

Result Cache가 효과적인 상황을 식별하는 것이 중요합니다:

```sql
-- 1. 자주 실행되는 참조 데이터 조회 (변경 빈도 낮음)
SELECT /*+ RESULT_CACHE */ 
       product_id, product_name, category_id, unit_price
FROM products
WHERE discontinued_flag = 'N'
  AND available_flag = 'Y';

-- 2. 복잡한 집계 쿼리
SELECT /*+ RESULT_CACHE */ 
       EXTRACT(YEAR FROM order_date) AS order_year,
       EXTRACT(MONTH FROM order_date) AS order_month,
       customer_country,
       COUNT(DISTINCT customer_id) AS unique_customers,
       SUM(order_amount) AS total_sales,
       AVG(order_amount) AS avg_order_value
FROM orders
WHERE order_date >= ADD_MONTHS(TRUNC(SYSDATE), -12)
  AND order_status = 'COMPLETED'
GROUP BY EXTRACT(YEAR FROM order_date),
         EXTRACT(MONTH FROM order_date),
         customer_country;

-- 3. 여러 테이블 조인 결과
SELECT /*+ RESULT_CACHE */ 
       c.customer_id,
       c.customer_name,
       c.customer_segment,
       COUNT(o.order_id) AS total_orders,
       SUM(o.order_amount) AS lifetime_value
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.registration_date >= DATE '2023-01-01'
  AND c.active_flag = 'Y'
GROUP BY c.customer_id, c.customer_name, c.customer_segment
HAVING COUNT(o.order_id) > 0;
```

### 캐시 효과 모니터링

```sql
-- 특정 SQL의 캐시 사용 현황 모니터링
WITH sql_cache_info AS (
  SELECT 
    sql_id,
    sql_text,
    executions,
    buffer_gets,
    disk_reads,
    ROUND(elapsed_time / 1000000, 2) AS elapsed_seconds,
    ROUND(cpu_time / 1000000, 2) AS cpu_seconds
  FROM v$sql
  WHERE sql_text LIKE '%RESULT_CACHE%'
    AND sql_text NOT LIKE '%v$sql%'
    AND executions > 0
)
SELECT 
  sci.sql_id,
  SUBSTR(sci.sql_text, 1, 100) AS sql_snippet,
  sci.executions,
  sci.buffer_gets,
  sci.disk_reads,
  sci.elapsed_seconds,
  sci.cpu_seconds,
  ROUND(sci.elapsed_seconds / sci.executions, 4) AS avg_elapsed_per_exec,
  ROUND(sci.cpu_seconds / sci.executions, 4) AS avg_cpu_per_exec,
  ROUND(sci.buffer_gets / sci.executions) AS avg_buffer_gets_per_exec,
  ROUND(sci.disk_reads / sci.executions) AS avg_disk_reads_per_exec
FROM sql_cache_info sci
ORDER BY sci.executions DESC;
```

### 주의사항과 제한사항

Result Cache를 사용할 때 고려해야 할 제약사항들:

```sql
-- 1. 비결정적 함수가 포함된 경우 캐시되지 않음
SELECT /*+ RESULT_CACHE */ 
       DBMS_RANDOM.VALUE() AS random_value,  -- 캐시되지 않음
       SYSDATE AS current_time,              -- 캐시되지 않음
       employee_id,
       first_name
FROM employees;

-- 2. FOR UPDATE 절이 있는 경우 캐시되지 않음
SELECT /*+ RESULT_CACHE */ 
       employee_id, salary
FROM employees
WHERE department_id = 50
FOR UPDATE;  -- 캐시되지 않음

-- 3. 결과 집합이 너무 큰 경우 캐시되지 않을 수 있음
-- RESULT_CACHE_MAX_RESULT 파라미터 값 초과 시 캐시 실패
```

---

## PL/SQL Function Result Cache 구현

### 기본 함수 캐싱

```sql
-- 간단한 참조 데이터 조회 함수
CREATE OR REPLACE FUNCTION get_department_name(
    p_department_id IN NUMBER
) RETURN VARCHAR2
RESULT_CACHE
IS
    v_department_name departments.department_name%TYPE;
BEGIN
    SELECT department_name
    INTO v_department_name
    FROM departments
    WHERE department_id = p_department_id;
    
    RETURN v_department_name;
END get_department_name;
/

-- 복잡한 계산 함수
CREATE OR REPLACE FUNCTION calculate_tax(
    p_amount IN NUMBER,
    p_country_code IN VARCHAR2
) RETURN NUMBER
RESULT_CACHE
IS
    v_tax_rate NUMBER;
    v_tax_amount NUMBER;
BEGIN
    -- 세율 조회 (비용이 큰 조회라고 가정)
    SELECT tax_rate
    INTO v_tax_rate
    FROM country_tax_rates
    WHERE country_code = p_country_code
      AND effective_date <= SYSDATE
      AND expiry_date >= SYSDATE;
    
    -- 복잡한 계산 로직
    IF p_amount <= 1000 THEN
        v_tax_amount := p_amount * v_tax_rate;
    ELSIF p_amount <= 10000 THEN
        v_tax_amount := 1000 * v_tax_rate + (p_amount - 1000) * v_tax_rate * 0.8;
    ELSE
        v_tax_amount := 1000 * v_tax_rate + 9000 * v_tax_rate * 0.8 + 
                       (p_amount - 10000) * v_tax_rate * 0.6;
    END IF;
    
    RETURN ROUND(v_tax_amount, 2);
END calculate_tax;
/
```

### 세션 컨텍스트를 고려한 함수 캐싱

```sql
-- 세션별 NLS 설정을 고려한 함수
CREATE OR REPLACE FUNCTION format_currency(
    p_amount IN NUMBER,
    p_currency_code IN VARCHAR2
) RETURN VARCHAR2
RESULT_CACHE RELIES_ON (currency_formats)
IS
    v_format_string VARCHAR2(100);
    v_formatted_amount VARCHAR2(100);
BEGIN
    -- 통화 포맷 정보 조회
    SELECT format_mask
    INTO v_format_string
    FROM currency_formats
    WHERE currency_code = p_currency_code;
    
    -- 현재 세션의 NLS 설정 적용
    v_formatted_amount := TO_CHAR(p_amount, v_format_string, 
                                  'NLS_NUMERIC_CHARACTERS=''.,''');
    
    RETURN v_formatted_amount;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN TO_CHAR(p_amount, 'FM999G999G999G999D00');
END format_currency;
/
```

### 함수 캐시 사용 패턴

```sql
-- 대량 데이터 처리 시 함수 캐시 효과 확인
DECLARE
    TYPE t_sales_tab IS TABLE OF sales%ROWTYPE;
    v_sales_data t_sales_tab;
    v_total_tax NUMBER := 0;
    v_start_time TIMESTAMP;
    v_end_time TIMESTAMP;
BEGIN
    v_start_time := SYSTIMESTAMP;
    
    -- 대량 데이터 조회
    SELECT * 
    BULK COLLECT INTO v_sales_data
    FROM sales
    WHERE sale_date >= DATE '2024-01-01'
      AND sale_date < DATE '2024-02-01';
    
    -- 캐시된 함수 반복 호출
    FOR i IN 1..v_sales_data.COUNT LOOP
        v_total_tax := v_total_tax + 
                      calculate_tax(v_sales_data(i).amount, 
                                   v_sales_data(i).country_code);
    END LOOP;
    
    v_end_time := SYSTIMESTAMP;
    
    DBMS_OUTPUT.PUT_LINE('처리된 레코드 수: ' || v_sales_data.COUNT);
    DBMS_OUTPUT.PUT_LINE('총 세금 계산: ' || v_total_tax);
    DBMS_OUTPUT.PUT_LINE('소요 시간: ' || 
                        EXTRACT(SECOND FROM (v_end_time - v_start_time)) || '초');
END;
/
```

---

## 실전 적용 예제: 대시보드 성능 최적화

### 대시보드용 데이터 모델 설계

```sql
-- 대시보드 성능 최적화를 위한 테이블 설계
CREATE TABLE dashboard_metrics (
    metric_date    DATE NOT NULL,
    metric_type    VARCHAR2(50) NOT NULL,
    region_code    VARCHAR2(10),
    metric_value   NUMBER,
    last_updated   TIMESTAMP DEFAULT SYSTIMESTAMP,
    CONSTRAINT pk_dashboard_metrics 
        PRIMARY KEY (metric_date, metric_type, region_code)
);

-- 자주 사용되는 집계 함수
CREATE OR REPLACE FUNCTION get_metric_value(
    p_metric_date  IN DATE,
    p_metric_type  IN VARCHAR2,
    p_region_code  IN VARCHAR2 DEFAULT NULL
) RETURN NUMBER
RESULT_CACHE
IS
    v_metric_value NUMBER;
BEGIN
    IF p_region_code IS NULL THEN
        -- 전체 지역 집계
        SELECT SUM(metric_value)
        INTO v_metric_value
        FROM dashboard_metrics
        WHERE metric_date = p_metric_date
          AND metric_type = p_metric_type;
    ELSE
        -- 특정 지역 집계
        SELECT metric_value
        INTO v_metric_value
        FROM dashboard_metrics
        WHERE metric_date = p_metric_date
          AND metric_type = p_metric_type
          AND region_code = p_region_code;
    END IF;
    
    RETURN NVL(v_metric_value, 0);
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN 0;
END get_metric_value;
/

-- 대시보드 메인 쿼리
SELECT /*+ RESULT_CACHE */ 
       '일별 매출' AS metric_name,
       metric_date,
       get_metric_value(metric_date, 'DAILY_SALES', 'ASIA') AS asia_sales,
       get_metric_value(metric_date, 'DAILY_SALES', 'EUROPE') AS europe_sales,
       get_metric_value(metric_date, 'DAILY_SALES', 'AMERICA') AS america_sales,
       get_metric_value(metric_date, 'DAILY_SALES') AS global_sales
FROM (
    SELECT DISTINCT metric_date
    FROM dashboard_metrics
    WHERE metric_date >= TRUNC(SYSDATE) - 30
      AND metric_type = 'DAILY_SALES'
)
ORDER BY metric_date DESC;
```

### 복합 캐싱 전략 구현

```sql
-- 1. 자주 사용되는 참조 데이터를 위한 함수
CREATE OR REPLACE FUNCTION get_exchange_rate(
    p_from_currency IN VARCHAR2,
    p_to_currency   IN VARCHAR2,
    p_rate_date     IN DATE DEFAULT TRUNC(SYSDATE)
) RETURN NUMBER
RESULT_CACHE
IS
    v_exchange_rate NUMBER;
BEGIN
    SELECT exchange_rate
    INTO v_exchange_rate
    FROM exchange_rates
    WHERE from_currency = p_from_currency
      AND to_currency = p_to_currency
      AND rate_date = p_rate_date;
    
    RETURN v_exchange_rate;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        -- 기본 환율 또는 가장 가까운 날짜의 환율 조회
        SELECT exchange_rate
        INTO v_exchange_rate
        FROM exchange_rates
        WHERE from_currency = p_from_currency
          AND to_currency = p_to_currency
          AND rate_date = (
              SELECT MAX(rate_date)
              FROM exchange_rates
              WHERE from_currency = p_from_currency
                AND to_currency = p_to_currency
                AND rate_date <= p_rate_date
          );
        
        RETURN v_exchange_rate;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN 1;  -- 기본값
END get_exchange_rate;
/

-- 2. 환율 적용된 금액 계산 함수
CREATE OR REPLACE FUNCTION convert_currency(
    p_amount        IN NUMBER,
    p_from_currency IN VARCHAR2,
    p_to_currency   IN VARCHAR2,
    p_transaction_date IN DATE
) RETURN NUMBER
RESULT_CACHE
IS
    v_exchange_rate NUMBER;
BEGIN
    v_exchange_rate := get_exchange_rate(p_from_currency, p_to_currency, 
                                         p_transaction_date);
    
    RETURN ROUND(p_amount * v_exchange_rate, 2);
END convert_currency;
/

-- 3. 캐시된 함수를 활용한 복합 쿼리
SELECT /*+ RESULT_CACHE */ 
       customer_country,
       SUM(amount) AS local_amount,
       SUM(convert_currency(amount, original_currency, 'USD', order_date)) AS usd_amount
FROM international_orders
WHERE order_date >= DATE '2024-01-01'
  AND order_status = 'COMPLETED'
GROUP BY customer_country
ORDER BY usd_amount DESC;
```

---

## 성능 모니터링과 튜닝

### 종합 성능 분석 쿼리

```sql
-- Result Cache 성능 종합 분석 리포트
WITH cache_stats AS (
    SELECT 
        name,
        value
    FROM v$result_cache_statistics
),
sql_perf AS (
    SELECT 
        sql_id,
        sql_text,
        executions,
        buffer_gets,
        disk_reads,
        elapsed_time,
        cpu_time
    FROM v$sql
    WHERE UPPER(sql_text) LIKE '%RESULT_CACHE%'
      AND executions > 0
      AND parsing_schema_id != 0
),
cache_objects AS (
    SELECT 
        type,
        COUNT(*) AS object_count,
        SUM(bytes) AS total_bytes,
        AVG(bytes) AS avg_bytes
    FROM v$result_cache_objects
    WHERE status != 'Invalid'
    GROUP BY type
)
SELECT 
    '캐시 통계' AS section,
    cs.name AS metric,
    TO_CHAR(cs.value, '999,999,999,999') AS value
FROM cache_stats cs
UNION ALL
SELECT 
    'SQL 성능' AS section,
    '평균 실행 시간(ms)' AS metric,
    TO_CHAR(ROUND(AVG(sp.elapsed_time / sp.executions / 1000), 2), '999,999.99') AS value
FROM sql_perf sp
UNION ALL
SELECT 
    '캐시 객체' AS section,
    co.type || ' 수' AS metric,
    TO_CHAR(co.object_count, '999,999') AS value
FROM cache_objects co
ORDER BY section, metric;
```

### 상세 모니터링을 위한 저장 프로시저

```sql
CREATE OR REPLACE PROCEDURE monitor_result_cache_performance AS
    v_report CLOB;
    
    PROCEDURE add_line(p_text VARCHAR2) IS
    BEGIN
        v_report := v_report || p_text || CHR(10);
    END;
    
BEGIN
    v_report := 'Result Cache 성능 모니터링 리포트' || CHR(10);
    v_report := v_report || '생성 시간: ' || TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS') || CHR(10);
    v_report := v_report || REPEAT('=', 80) || CHR(10) || CHR(10);
    
    -- 1. 기본 통계
    add_line('1. 기본 통계');
    add_line(REPEAT('-', 40));
    FOR rec IN (
        SELECT name, value
        FROM v$result_cache_statistics
        ORDER BY 
            CASE name
                WHEN 'Create Count Success' THEN 1
                WHEN 'Find Count' THEN 2
                WHEN 'Invalidation Count' THEN 3
                ELSE 4
            END
    ) LOOP
        add_line(RPAD(rec.name, 30) || ': ' || TO_CHAR(rec.value, '999,999,999'));
    END LOOP;
    add_line('');
    
    -- 2. 히트율 계산
    DECLARE
        v_find_count NUMBER;
        v_create_count NUMBER;
        v_hit_ratio NUMBER;
    BEGIN
        SELECT 
            MAX(CASE WHEN name = 'Find Count' THEN value END),
            MAX(CASE WHEN name = 'Create Count Success' THEN value END)
        INTO v_find_count, v_create_count
        FROM v$result_cache_statistics;
        
        v_hit_ratio := ROUND(v_find_count * 100.0 / NULLIF(v_find_count + v_create_count, 0), 2);
        
        add_line('2. 캐시 히트율');
        add_line(REPEAT('-', 40));
        add_line('히트율: ' || v_hit_ratio || '%');
        add_line('');
    END;
    
    -- 3. 메모리 사용 현황
    add_line('3. 메모리 사용 현황');
    add_line(REPEAT('-', 40));
    FOR rec IN (
        SELECT 
            SUM(CASE WHEN type = 'Result' THEN 1 ELSE 0 END) AS result_count,
            SUM(CASE WHEN type = 'Result' THEN bytes ELSE 0 END) AS result_bytes,
            SUM(CASE WHEN type = 'Dependency' THEN 1 ELSE 0 END) AS dependency_count,
            COUNT(*) AS total_objects,
            SUM(bytes) AS total_bytes
        FROM v$result_cache_objects
        WHERE status != 'Invalid'
    ) LOOP
        add_line('결과 객체: ' || rec.result_count || '개 (' || 
                 ROUND(rec.result_bytes / 1024 / 1024, 2) || ' MB)');
        add_line('의존성 객체: ' || rec.dependency_count || '개');
        add_line('총 객체: ' || rec.total_objects || '개 (' || 
                 ROUND(rec.total_bytes / 1024 / 1024, 2) || ' MB)');
    END LOOP;
    add_line('');
    
    -- 4. 상위 캐시 사용 SQL
    add_line('4. 상위 캐시 사용 SQL');
    add_line(REPEAT('-', 40));
    FOR rec IN (
        SELECT 
            sql_id,
            executions,
            ROUND(elapsed_time / 1000000, 2) AS elapsed_seconds,
            ROUND(cpu_time / 1000000, 2) AS cpu_seconds,
            buffer_gets,
            disk_reads,
            SUBSTR(sql_text, 1, 100) AS sql_snippet
        FROM v$sql
        WHERE UPPER(sql_text) LIKE '%RESULT_CACHE%'
          AND executions > 10
        ORDER BY elapsed_time DESC
        FETCH FIRST 5 ROWS ONLY
    ) LOOP
        add_line('SQL ID: ' || rec.sql_id);
        add_line('실행 횟수: ' || rec.executions);
        add_line('총 소요 시간: ' || rec.elapsed_seconds || '초');
        add_line('평균 시간: ' || ROUND(rec.elapsed_seconds / rec.executions, 4) || '초/회');
        add_line('SQL: ' || rec.sql_snippet);
        add_line('');
    END LOOP;
    
    -- 리포트 출력
    DBMS_OUTPUT.PUT_LINE(v_report);
    
    -- 파일로 저장 (선택 사항)
    /*
    DECLARE
        v_file UTL_FILE.FILE_TYPE;
        v_filename VARCHAR2(100) := 'result_cache_report_' || 
                                   TO_CHAR(SYSDATE, 'YYYYMMDD_HH24MI') || '.txt';
    BEGIN
        v_file := UTL_FILE.FOPEN('REPORTS_DIR', v_filename, 'W');
        UTL_FILE.PUT_LINE(v_file, v_report);
        UTL_FILE.FCLOSE(v_file);
    END;
    */
END monitor_result_cache_performance;
/

-- 리포트 실행
EXEC monitor_result_cache_performance;
```

### 자동화된 튜닝 추천 시스템

```sql
-- Result Cache 적용 후보 식별 쿼리
SELECT 
    sql_id,
    plan_hash_value,
    executions,
    elapsed_time,
    cpu_time,
    buffer_gets,
    disk_reads,
    rows_processed,
    SUBSTR(sql_text, 1, 200) AS sql_snippet,
    -- 캐싱 적합성 점수 계산
    CASE 
        WHEN executions > 1000 
             AND disk_reads > buffer_gets * 0.1  -- 디스크 읽기가 많은
             AND elapsed_time / executions > 100000  -- 평균 100ms 이상
             AND UPPER(sql_text) NOT LIKE '%SYSDATE%'
             AND UPPER(sql_text) NOT LIKE '%DBMS_RANDOM%'
             AND UPPER(sql_text) NOT LIKE '%FOR UPDATE%'
        THEN '적합'
        ELSE '부적합'
    END AS cache_suitability,
    -- 예상 개선 효과
    ROUND((elapsed_time * 0.7) / 1000000, 2) AS estimated_saving_seconds
FROM v$sql
WHERE command_type = 3  -- SELECT 문만
  AND executions > 100
  AND parsing_schema_id != 0
  AND UPPER(sql_text) NOT LIKE '%V$%'  -- 시스템 뷰 제외
ORDER BY estimated_saving_seconds DESC
FETCH FIRST 20 ROWS ONLY;
```

---

## 운영 모범 사례와 문제 해결

### 운영을 위한 권장 설정

```sql
-- 안정적인 운영을 위한 파라미터 설정
-- 1. 모드는 MANUAL로 유지 (명시적 제어)
ALTER SYSTEM SET RESULT_CACHE_MODE = MANUAL SCOPE=BOTH;

-- 2. 적절한 메모리 크기 설정 (시스템 메모리의 1-2%)
ALTER SYSTEM SET RESULT_CACHE_MAX_SIZE = 1G SCOPE=SPFILE;

-- 3. 단일 결과 크기 제한 (너무 큰 결과 방지)
ALTER SYSTEM SET RESULT_CACHE_MAX_RESULT = 5 SCOPE=SPFILE;

-- 4. 원격 객체 TTL 설정
ALTER SYSTEM SET RESULT_CACHE_REMOTE_EXPIRATION = 60 SCOPE=SPFILE; -- 60분

-- 변경 후 재시작 필요
-- SHUTDOWN IMMEDIATE
-- STARTUP
```

### 일반적인 문제와 해결책

```sql
-- 문제 1: 캐시가 생성되지 않음
-- 원인: 비결정적 요소 포함
SELECT /*+ RESULT_CACHE */ 
       employee_id,
       first_name,
       DBMS_RANDOM.STRING('A', 10) AS random_string  -- 문제 원인
FROM employees;

-- 해결: 비결정적 요소 제거
SELECT /*+ RESULT_CACHE */ 
       employee_id,
       first_name
FROM employees;

-- 문제 2: 자주 무효화됨
-- 원인: 기반 테이블 변경 빈도가 높음
-- 해결: 스냅샷 테이블이나 MView 사용 고려
CREATE MATERIALIZED VIEW mv_daily_sales
REFRESH COMPLETE NEXT SYSDATE + 1/24  -- 1시간마다 갱신
AS
SELECT /*+ RESULT_CACHE */
       TRUNC(sale_date) AS sale_day,
       product_category,
       SUM(amount) AS daily_sales
FROM sales
GROUP BY TRUNC(sale_date), product_category;

-- 문제 3: 메모리 부족
-- 원인: 캐시 크기 부족
-- 해결: 메모리 증가 또는 캐시 대상 조정
ALTER SYSTEM SET RESULT_CACHE_MAX_SIZE = 2G SCOPE=SPFILE;

-- 문제 진단을 위한 쿼리
SELECT 
    '메모리 상태' AS category,
    name,
    value
FROM v$result_cache_memory
UNION ALL
SELECT 
    '통계' AS category,
    name,
    TO_CHAR(value)
FROM v$result_cache_statistics
WHERE name IN ('Block Count Maximum', 'Block Count Current', 
               'Create Count Failure', 'Free Count');
```

### RAC 환경에서의 고려사항

```sql
-- RAC 환경에서의 Result Cache 관리
-- 1. 인스턴스별 캐시 모니터링
SELECT 
    inst_id,
    name,
    value
FROM gv$result_cache_statistics
ORDER BY inst_id, name;

-- 2. 인스턴스별 캐시 효율성 비교
SELECT 
    inst_id,
    ROUND(
        SUM(CASE WHEN name = 'Find Count' THEN value ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN name IN ('Find Count', 'Create Count Success') 
                   THEN value ELSE 0 END), 0), 2
    ) AS hit_ratio_percent
FROM gv$result_cache_statistics
GROUP BY inst_id
ORDER BY inst_id;

-- 3. 서비스 기반 라우팅으로 캐시 효율 향상
-- 리포팅 서비스를 특정 인스턴스로 라우팅
BEGIN
    DBMS_SERVICE.CREATE_SERVICE(
        service_name => 'REPORTING_SERVICE',
        network_name => 'reporting.example.com'
    );
    
    DBMS_SERVICE.START_SERVICE('REPORTING_SERVICE');
END;
/

-- 서비스에 인스턴스 할당
EXEC DBMS_SERVICE.MODIFY_SERVICE('REPORTING_SERVICE', 'PREFER_INSTANCE', '1');
```

---

## 결론: 효과적인 Result Cache 활용 전략

Oracle Result Cache는 데이터베이스 성능 최적화를 위한 강력한 도구이지만, 적절한 상황에서 올바르게 사용해야 그 진가를 발휘합니다. 효과적인 활용을 위한 핵심 원칙들을 정리해보겠습니다:

### 1. 적절한 사용 사례 식별이 우선입니다

Result Cache는 다음과 같은 특징을 가진 워크로드에 가장 효과적입니다:
- 동일한 입력에 대해 항상 동일한 결과를 반환하는 결정적 작업
- 기반 데이터의 변경 빈도가 낮은 경우
- 실행 빈도가 높고 계산 비용이 큰 쿼리나 함수
- 결과 집합의 크기가 적당한 경우

### 2. 점진적이고 측정 가능한 접근이 필요합니다

성공적인 Result Cache 도입을 위해서는:
1. 후보 쿼리나 함수를 식별하고 우선순위를 매깁니다
2. 작은 규모로 시작하여 효과를 측정합니다
3. AWR, ASH, SQL 모니터 등을 활용한 객관적 성능 데이터를 수집합니다
4. 문제가 발생하면 신속하게 원인을 분석하고 조치합니다

### 3. 운영 환경에 맞는 설정이 필수적입니다

안정적인 운영을 위해:
- `RESULT_CACHE_MODE`는 MANUAL로 설정하여 명시적 제어를 유지합니다
- 시스템 리소스와 워크로드 패턴에 맞는 적절한 메모리 크기를 할당합니다
- 정기적인 모니터링으로 캐시 효율성을 확인합니다
- RAC 환경에서는 인스턴스별 특성을 고려합니다

### 4. 대안을 함께 고려해야 합니다

Result Cache가 적합하지 않은 경우 다른 최적화 기법을 고려하세요:
- 머티리얼라이즈드 뷰: 디스크 기반의 지속적 캐싱이 필요할 때
- 애플리케이션 레벨 캐싱: 네트워크 지연까지 포함한 종단간 최적화가 필요할 때
- 인덱스 최적화: I/O 자체를 줄이는 근본적 해결이 필요할 때

### 5. 지속적인 모니터링과 튜닝이 성공의 열쇠입니다

Result Cache는 한 번 설정하고 잊어버리는 기능이 아닙니다:
- 정기적인 성능 리포트를 생성하고 분석합니다
- 애플리케이션 변화에 따라 캐시 전략을 조정합니다
- 새로운 워크로드 패턴을 식별하고 적응합니다
- 문제 발생 시 체계적으로 진단하고 해결합니다

가장 중요한 것은 Result Cache를 은탄환이 아닌 도구상자의 하나로 이해하는 것입니다. 올바르게 사용할 때 데이터베이스 성능을 획기적으로 향상시킬 수 있는 강력한 도구이지만, 모든 상황에 적합한 것은 아닙니다. 데이터와 워크로드를 이해하고, 측정하고, 검증하는 과학적 접근법이 지속 가능한 성능 개선의 핵심입니다.