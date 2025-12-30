---
layout: post
title: DB 심화 - DBMS_XPLAN
date: 2025-10-24 16:25:23 +0900
category: DB 심화
---
# Oracle **DBMS_XPLAN** 완전 가이드

## 핵심 개념: 예상 계획 vs 실제 계획 vs 실행 통계

Oracle에서 SQL 성능 튜닝을 위한 가장 중요한 도구 중 하나가 **DBMS_XPLAN** 패키지입니다. 이 패키지를 통해 SQL 실행 계획을 다양한 형태로 분석할 수 있습니다.

- **예상 실행 계획**: SQL이 실행되기 전 옵티마이저가 예측한 계획
  ```sql
  EXPLAIN PLAN FOR <SQL>;
  SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'ALL +PREDICATE +NOTE'));
  ```

- **실제 실행 계획**: 메모리(Shared Pool)에 캐시된 커서의 실제 실행 계획
  ```sql
  -- 방금 실행한 SQL의 마지막 실행 계획
  SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'TYPICAL'));
  ```

- **실행 통계 포함 계획**: 각 실행 단계별 실제 성능 통계를 포함한 상세 계획
  ```sql
  -- 라인별 A-Rows(실제 처리 행수), Starts, Time, Buffers 등의 통계 포함
  SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
      'ALLSTATS LAST +PEEKED_BINDS +PREDICATE +PROJECTION +OUTLINE +NOTE'));
  ```

> **핵심 원칙**: 튜닝의 결론은 반드시 **실제 실행 통계(ALLSTATS)** 를 기반으로 내려야 합니다. `EXPLAIN PLAN`의 예상 계획은 방향성을 잡는 데만 사용하고, 실제 성능 분석에는 실제 실행 계획을 사용하세요.

---

## 실습을 위한 샘플 스키마 및 데이터 준비

```sql
-- 기존 테이블 정리
DROP TABLE customers PURGE;
DROP TABLE orders PURGE;

-- 고객 테이블 생성
CREATE TABLE customers(
  id      NUMBER PRIMARY KEY,
  region  VARCHAR2(10),
  grade   NUMBER,
  name    VARCHAR2(100)
);

-- 주문 테이블 생성
CREATE TABLE orders(
  order_id    NUMBER PRIMARY KEY,
  customer_id NUMBER NOT NULL REFERENCES customers(id),
  order_date  DATE   NOT NULL,
  amount      NUMBER NOT NULL
);

-- 인덱스 생성
CREATE INDEX ix_orders_cust_date      ON orders(customer_id, order_date);
CREATE INDEX ix_cust_region_grade     ON customers(region, grade);

-- 샘플 데이터 삽입
BEGIN
  -- 고객 데이터: 100,000명
  FOR i IN 1..100000 LOOP
    INSERT INTO customers VALUES(
      i,
      CASE MOD(i,10) 
        WHEN 0 THEN 'APAC' 
        WHEN 1 THEN 'EMEA' 
        ELSE 'AMER' 
      END,
      MOD(i,5),
      'C'||i
    );
  END LOOP;
  COMMIT;

  -- 주문 데이터: 2,000,000건
  FOR i IN 1..2000000 LOOP
    INSERT INTO orders VALUES(
      i,
      MOD(i,100000)+1,  -- 1~100,000 사이의 고객 ID
      DATE '2024-01-01' + MOD(i,365),  -- 2024년 내 날짜
      MOD(i,500)+1  -- 1~500 사이의 금액
    );
    IF MOD(i,100000)=0 THEN COMMIT; END IF;
  END LOOP;
  COMMIT;
END;
/

-- 통계 수집
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'CUSTOMERS',cascade=>TRUE);
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'ORDERS',cascade=>TRUE);
```

---

## **예상 실행 계획 분석** — `EXPLAIN PLAN FOR` + `DBMS_XPLAN.DISPLAY`

### 기본 사용법

```sql
-- 예상 실행 계획 생성
EXPLAIN PLAN FOR
SELECT /* plan_test */ SUM(o.amount)
FROM   orders o
WHERE  o.customer_id = :cid
AND    o.order_date BETWEEN :d1 AND :d2;

-- 계획 출력 (기본 형식)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());

-- 상세 계획 출력
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'ALL +PREDICATE +ALIAS +PROJECTION +NOTE'));
```

### `DISPLAY` 함수의 주요 파라미터

- **`table_name`**: 기본값 `NULL`로 설정하면 `PLAN_TABLE`에서 가장 최근의 계획을 조회합니다.
- **`statement_id`**: 특정 실행 계획을 식별하기 위한 ID입니다. `NULL`이면 최근 계획을 사용합니다.
- **`format`**: 출력 형식을 제어하는 문자열입니다.
  - 기본값: `'TYPICAL'`
  - 주요 옵션: `'BASIC'`, `'TYPICAL'`, `'ALL'`
  - 추가 옵션: `'+PREDICATE'`, `'+NOTE'`, `'+ALIAS'`, `'+PROJECTION'` 등

### 출력 결과 해석 포인트

1. **Operation 컬럼**: 각 단계의 실행 연산을 나타냅니다.
   - `INDEX RANGE SCAN`: 인덱스 범위 스캔
   - `TABLE ACCESS BY INDEX ROWID`: 인덱스를 통해 테이블 접근
   - `HASH JOIN`: 해시 조인
   - `NESTED LOOPS`: 네스티드 루프 조인
   - `SORT GROUP BY`, `SORT ORDER BY`: 정렬 연산

2. **Rows/Bytes/Cost/Time 컬럼**: 옵티마이저가 **예상한** 값입니다.
   - `E-Rows`: 예상 행 수
   - `Cost`: 상대적 비용
   - `Time`: 예상 실행 시간

3. **Predicate Information 섹션** (`+PREDICATE` 옵션 사용 시):
   - `access()`: 인덱스 접근 조건 (실제로 데이터를 찾는 데 사용된 조건)
   - `filter()`: 접근 후 필터링 조건 (추가적인 필터링 조건)

4. **Note 섹션** (`+NOTE` 옵션 사용 시):
   - 동적 샘플링 사용 여부
   - 옵티마이저의 결정 사항
   - 경고 메시지

> **주의 사항**: 예상 실행 계획은 실제 실행 환경(바인드 변수 값, 세션 설정, 통계 시점 등)과 다를 수 있으므로 참고용으로만 사용해야 합니다.

---

## **실제 실행 계획 분석** — `DBMS_XPLAN.DISPLAY_CURSOR`

### 기본 사용법

```sql
-- 1. 먼저 SQL 실행 (바인드 변수 사용)
VAR cid NUMBER; 
EXEC :cid := 12345;

VAR d1 DATE;   
EXEC :d1 := DATE '2024-06-01';

VAR d2 DATE;   
EXEC :d2 := DATE '2024-06-30';

-- SQL 실행
SELECT /* exec1 */ SUM(o.amount)
FROM   orders o
WHERE  o.customer_id = :cid
AND    o.order_date BETWEEN :d1 AND :d2;

-- 2. 실제 실행 계획 조회 (현재 세션의 마지막 SQL)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'TYPICAL +PREDICATE +NOTE +PEEKED_BINDS'));
```

### `DISPLAY_CURSOR` 함수의 주요 파라미터

- **`sql_id`**: 분석할 SQL의 ID입니다. `NULL`이면 현재 세션의 마지막 SQL을 사용합니다.
- **`child_number`**: Child 커서 번호입니다. `NULL`이면 기본 Child 커서를 사용합니다.
- **`format`**: 출력 형식 문자열입니다. `DISPLAY()`와 유사하지만 추가 옵션이 있습니다.

### Child 커서 이해하기

Oracle에서는 동일한 SQL 텍스트라도 실행 환경(바인드 변수 타입, NLS 설정 등)에 따라 여러 개의 **Child 커서**가 생성될 수 있습니다. 각 Child 커서는 서로 다른 실행 계획을 가질 수 있습니다.

```sql
-- 특정 SQL_ID와 Child 번호로 실행 계획 조회
SELECT sql_id, child_number, plan_hash_value
FROM   v$sql
WHERE  sql_text LIKE '%exec1%' AND sql_text NOT LIKE '%v$sql%';

-- 확인된 sql_id와 child_number로 실행 계획 조회
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('&sql_id', &child_number,
  'TYPICAL +PREDICATE +ALIAS +NOTE'));
```

---

## **실행 통계 분석** — `ALLSTATS` 옵션의 중요성

실제 성능 분석의 핵심은 각 실행 단계의 **실제 통계**를 확인하는 것입니다. `ALLSTATS` 옵션을 사용하면 라인별로 다음과 같은 실제 실행 통계를 확인할 수 있습니다.

### 주요 통계 항목

- **`Starts`**: 해당 연산이 시작된 횟수
- **`E-Rows`**: 옵티마이저가 예상한 행 수
- **`A-Rows`**: 실제로 처리된 행 수
- **`A-Time`**: 실제 소요된 시간
- **`Buffers`**: 논리적 I/O 횟수
- **`Reads`**: 물리적 I/O 횟수
- **`Writes`**: 쓰기 I/O 횟수
- **`OMem`/`1Mem`/`Used-Mem`**: 메모리 사용량 (정렬/해시 작업 시)
- **`Used-Tmp`**: 디스크 임시 공간 사용량

### 사용 예제

```sql
-- 1. SQL 실행 (바인드 변수 재사용)
SELECT /* exec1 */ SUM(o.amount)
FROM   orders o
WHERE  o.customer_id = :cid
AND    o.order_date BETWEEN :d1 AND :d2;

-- 2. 실행 통계 포함한 상세 계획 조회
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'ALLSTATS LAST +PEEKED_BINDS +PREDICATE +PROJECTION +OUTLINE +NOTE'));
```

### 통계 분석 체크리스트

1. **카디널리티 오류 확인**: `A-Rows` vs `E-Rows` 비교
   - 실제 처리 행수가 예상보다 훨씬 크다면 옵티마이저가 잘못된 선택을 했을 가능성이 큽니다.
   - 원인: 통계 부실, 히스토그램 부재, 바인드 변수 피킹 문제 등

2. **I/O 패턴 분석**: `Buffers`와 `Reads` 값 확인
   - 높은 `Buffers` 값은 많은 논리적 I/O를 의미합니다.
   - 높은 `Reads` 값은 많은 물리적 I/O를 의미합니다.
   - 캐시 효율성 판단: `Reads` / `Buffers` 비율이 낮을수록 좋음

3. **임시 공간 사용 확인**: `Used-Tmp` 값 확인
   - 값이 크다면 디스크로 스필(Spill)이 발생했음을 의미합니다.
   - 정렬(SORT)이나 해시 조인(HASH JOIN)에서 주로 발생합니다.

4. **실행 시간 분석**: `A-Time` 값 확인
   - 각 단계별 실제 소요 시간을 확인하여 병목 지점을 파악합니다.
   - 상위 단계의 시간은 하위 단계 시간의 합입니다.

---

## 고급 분석 기법

### 1. Nested Loops vs Hash Join 비교 분석

```sql
-- Nested Loops 조인 유도
VAR d1 DATE; VAR d2 DATE;
EXEC :d1 := DATE '2024-06-01'; 
EXEC :d2 := DATE '2024-06-30';

SELECT /* NL_TEST */ /*+ USE_NL(o) LEADING(c) */
       COUNT(*)
FROM   customers c JOIN orders o ON o.customer_id = c.id
WHERE  c.region = 'APAC' 
AND    o.order_date BETWEEN :d1 AND :d2;

-- 실행 계획 조회
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'ALLSTATS LAST +PREDICATE +PEEKED_BINDS +NOTE'));
```

**분석 포인트**:
- NL 조인에서 내부 테이블(`orders`) 접근 횟수(`Starts`) 확인
- 각 내부 테이블 접근 시 처리 행수(`A-Rows`) 확인
- 총 `Buffers`와 `A-Time`으로 성능 평가

### 2. 정렬 작업과 임시 공간 사용 분석

```sql
SELECT /* SORT_TEST */ c.region, SUM(o.amount) amt
FROM   orders o JOIN customers c ON c.id = o.customer_id
WHERE  o.order_date BETWEEN :d1 AND :d2
GROUP  BY c.region
ORDER  BY amt DESC;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'ALLSTATS LAST +PREDICATE +PROJECTION +NOTE'));
```

**분석 포인트**:
- `SORT GROUP BY` 또는 `SORT ORDER BY` 단계의 `Used-Tmp` 확인
- 메모리 내 정렬 vs 디스크 스필 판단
- `OMem`, `1Mem`, `Used-Mem` 값으로 PGA 메모리 사용량 확인

### 3. 바인드 변수 피킹 분석

```sql
VAR region VARCHAR2(10); 
EXEC :region := 'APAC';

SELECT /* BIND_TEST */ COUNT(*) 
FROM   customers 
WHERE  region = :region;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'ALLSTATS LAST +PEEKED_BINDS +NOTE'));
```

**분석 포인트**:
- `PEEKED_BINDS` 섹션에서 실제로 사용된 바인드 값 확인
- 바인드 값에 따른 실행 계획 변화 분석
- Adaptive Cursor Sharing(ACS) 동작 확인

---

## 출력 옵션 상세 설명

### 기본 출력 레벨
- **`BASIC`**: 최소한의 정보만 표시
- **`TYPICAL`**: 일반적인 튜닝에 필요한 정보 표시 (기본값)
- **`ALL`**: 가능한 모든 정보 표시

### 주요 추가 옵션
- **`+PREDICATE`**: 접근 및 필터 조건 표시
- **`+ALIAS`**: 테이블 및 컬럼 별칭 표시
- **`+PROJECTION`**: 각 단계에서 반환하는 컬럼 정보 표시
- **`+NOTE`**: 옵티마이저 메시지 표시
- **`+OUTLINE`**: 실행 계획을 재현할 수 있는 힌트 세트 표시
- **`+PEEKED_BINDS`**: 바인드 변수 피킹 값 표시
- **`+PARALLEL`**: 병렬 실행 관련 정보 강화 표시
- **`+ADAPTIVE`**: 적응형 계획 관련 정보 표시

### 추천 조합
```sql
-- 빠른 확인용
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'TYPICAL +PREDICATE +NOTE'));

-- 상세 분석용
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'ALLSTATS LAST +PEEKED_BINDS +PREDICATE +PROJECTION +OUTLINE +NOTE'));

-- 병렬 쿼리 분석용
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'ALLSTATS LAST +PARALLEL +NOTE'));
```

---

## DBMS_XPLAN과 연계 도구

### 1. SQL Monitor
장시간 실행되는 쿼리나 병렬 쿼리의 실시간 모니터링에 유용합니다.
```sql
-- SQL Monitor 리포트 생성
SELECT DBMS_SQLTUNE.REPORT_SQL_MONITOR(
  sql_id       => '&sql_id',
  type         => 'TEXT',
  report_level => 'ALL'
) AS monitor_report
FROM dual;
```

### 2. AWR 리포트의 실행 계획
과거의 실행 계획을 AWR에서 조회할 수 있습니다.
```sql
-- AWR에 저장된 실행 계획 조회
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_AWR(
  sql_id        => '&sql_id',
  plan_hash_value => NULL,
  db_id         => NULL,
  format        => 'TYPICAL +PREDICATE +NOTE'
));
```

### 3. 10046 트레이스
저수준의 세부 실행 정보를 분석할 때 사용합니다.
```sql
-- 세션 레벨 트레이스 활성화
ALTER SESSION SET EVENTS '10046 trace name context forever, level 8';

-- SQL 실행
SELECT /*+ TRACE_TEST */ * FROM orders WHERE order_id = 12345;

-- 트레이스 파일을 TKPROF로 변환하여 분석
-- $ tkprof ora_12345.trc output.txt sys=no sort=prsela,exeela,fchela
```

---

## 실전 튜닝 워크플로우

1. **문제 SQL 식별**
   - AWR/ASH 리포트, V$SQL 뷰, 애플리케이션 모니터링 등을 통해 성능 문제를 일으키는 SQL 식별

2. **현황 분석**
   ```sql
   -- 현재 실행 계획 및 통계 수집
   SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
     '&problem_sql_id', 
     &child_number,
     'ALLSTATS LAST +PEEKED_BINDS +PREDICATE +PROJECTION +OUTLINE +NOTE'
   ));
   ```

3. **문제 진단**
   - `A-Rows` vs `E-Rows` 비교: 카디널리티 오류 확인
   - `Buffers`/`Reads` 분석: I/O 병목 확인
   - `Used-Tmp` 분석: 임시 공간 사용 확인
   - `A-Time` 분석: 시간 소비 지점 확인

4. **개선 방안 수립**
   - 인덱스 추가/수정
   - 통계 갱신 (히스토그램 추가)
   - SQL 재작성 (조인 순서 변경, 서브쿼리 변환 등)
   - 힌트 추가
   - 파티셔닝 검토

5. **개선 효과 검증**
   ```sql
   -- 개선된 SQL 실행
   SELECT /*+ IMPROVED */ ...;
   
   -- 개선 전후 비교
   -- BEFORE: 원본 SQL 실행 계획 저장
   -- AFTER: 개선된 SQL 실행 계획 저장
   -- A-Rows, Buffers, A-Time 등 주요 지표 비교
   ```

6. **모니터링 및 유지관리**
   - SQL Plan Baseline 등록
   - SQL Profile 적용
   - 정기적인 성능 모니터링

---

## 자주 발생하는 문제와 해결 방안

### 1. 예상 계획과 실제 계획이 다를 때
**원인**: 바인드 변수 피킹, 동적 샘플링, 적응형 계획 등
**해결**: `DISPLAY_CURSOR`의 `ALLSTATS`와 `PEEKED_BINDS` 옵션으로 실제 실행 환경 분석

### 2. Child 커서가 너무 많을 때
**원인**: 바인드 변수 데이터 타입 불일치, NLS 설정 차이 등
**해결**: 
```sql
-- Child 커서 분기 원인 분석
SELECT child_number, reason 
FROM   v$sql_shared_cursor 
WHERE  sql_id = '&sql_id';

-- 바인드 변수 타입 통일
-- 애플리케이션 코드에서 동일한 데이터 타입 사용 보장
```

### 3. 임시 테이블 스페이스 사용이 많을 때
**원인**: 큰 정렬 또는 해시 조인 작업
**해결**:
- `PGA_AGGREGATE_TARGET` 증가
- 인덱스를 이용한 정렬 회피
- 배치 크기 조정

### 4. 병렬 실행 시 성능 저하
**원인**: 데이터 분배 불균형(DOP 스큐)
**해결**:
```sql
-- 병렬 분배 방식 확인
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'ALLSTATS LAST +PARALLEL +NOTE'));
```
- `PX SEND`/`PX RECEIVE` 단계의 `A-Rows` 비교
- 분배 키 변경 또는 해시 분배 방식 변경 검토

---

## 결론

Oracle DBMS_XPLAN 패키지는 SQL 성능 튜닝을 위한 필수 도구입니다. 효과적인 사용을 위한 핵심 원칙은 다음과 같습니다.

1. **실제 데이터를 기반으로 분석하라**: `EXPLAIN PLAN`의 예상 계획은 참고용일 뿐이며, 실제 성능 분석에는 반드시 `DISPLAY_CURSOR`의 `ALLSTATS` 옵션을 사용하여 실제 실행 통계를 확인해야 합니다.

2. **체계적으로 접근하라**: 
   - 먼저 `A-Rows`와 `E-Rows`를 비교하여 옵티마이저의 카디널리티 추정 오류를 확인합니다.
   - 다음으로 `Buffers`와 `Reads`를 분석하여 I/O 패턴을 이해합니다.
   - 마지막으로 `A-Time`을 통해 실제 시간 소비 지점을 파악합니다.

3. **다양한 관점에서 분석하라**: 단순히 실행 계획만 보는 것이 아니라, 바인드 변수 피킹(`PEEKED_BINDS`), 옵티마이저 힌트 세트(`OUTLINE`), 적응형 계획 정보(`ADAPTIVE`) 등을 종합적으로 분석해야 합니다.

4. **지속적으로 모니터링하라**: 일회성 튜닝으로 끝내지 말고, SQL Plan Baseline, SQL Profile 등을 활용하여 안정적인 성능을 유지할 수 있도록 합니다.

성능 튜닝은 과학적 접근이 필요한 작업입니다. DBMS_XPLAN은 이러한 과학적 접근을 가능하게 하는 강력한 현미경과 같습니다. 이 도구를 숙련되게 사용한다면, 복잡한 SQL의 성능 문제를 체계적으로 진단하고 효과적으로 해결할 수 있을 것입니다.