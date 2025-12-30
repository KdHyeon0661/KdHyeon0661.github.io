---
layout: post
title: DB 심화 - PQ_DISTRIBUTE
date: 2025-11-25 15:25:23 +0900
category: DB 심화
---
# Oracle PQ_DISTRIBUTE 힌트: 병렬 조인 데이터 분배 최적화

병렬 쿼리의 성능을 결정하는 가장 중요한 요소 중 하나는 **데이터가 병렬 프로세스(PX 슬레이브) 간에 어떻게 분배되고 교환되는가**입니다. `PQ_DISTRIBUTE` 힌트는 이 데이터 분배 방식을 개발자가 직접 제어할 수 있게 해주는 강력한 도구입니다. 이 글에서는 `PQ_DISTRIBUTE` 힌트의 작동 원리, 다양한 분배 방식, 그리고 실전 적용 방법을 체계적으로 살펴보겠습니다.

---

## 병렬 조인의 데이터 흐름 이해하기

### 병렬 쿼리 아키텍처 기본 개념

**QC(Query Coordinator)**: SQL 실행을 시작하고 최종 결과를 모아서 반환하는 마스터 프로세스입니다.

**PX 슬레이브**: 실제 작업을 병렬로 수행하는 워커 프로세스입니다.

**DFO(Data Flow Operation)**: 병렬로 실행되는 작업의 논리적 단위입니다.

**TQ(Table Queue)**: DFO 간에 데이터가 이동하는 통로로, 실행계획에서 `TQ10000`, `TQ10001` 등으로 표시됩니다.

**PX SEND / PX RECEIVE**: TQ를 통해 데이터를 전송하고 수신하는 오퍼레이션입니다. 이 오퍼레이션의 타입이 바로 데이터 분배 방식을 나타냅니다.

### 실행계획의 IN-OUT 컬럼 의미

실행계획의 IN-OUT 컬럼은 데이터 흐름의 방향을 보여줍니다:

- `S->P`: 직렬(Serial) 프로세스에서 병렬(Parallel) 프로세스로 데이터 전송
- `P->S`: 병렬 프로세스에서 직렬 프로세스로 데이터 수집
- `P->P`: 병렬 프로세스 간 데이터 재분배
- `PCWP / PCWC`: 동일한 병렬 프로세스 집합이 연속 작업 수행 (TQ 통신 최소화)

### PX SEND 타입: 분배 방식의 시각화

실행계획에서 확인할 수 있는 주요 PX SEND 타입:

- `PX SEND HASH`: 해시값 기반 분배
- `PX SEND BROADCAST`: 모든 PX 슬레이브로 브로드캐스트
- `PX SEND PARTITION`: 파티션 단위 분배
- `PX SEND ROUND-ROBIN`: 라운드 로빈 방식 분배

`PQ_DISTRIBUTE` 힌트는 바로 이러한 PX SEND 타입을 명시적으로 지정하기 위한 것입니다.

---

## PQ_DISTRIBUTE 힌트 기본 이해

### 힌트의 목적과 작동 원리

`PQ_DISTRIBUTE` 힌트는 병렬 조인, 집계, 정렬 등에서 특정 테이블의 행이 병렬 프로세스 간에 어떻게 분배될지를 지정합니다. 옵티마이저가 통계 정보를 바탕으로 자동 선택한 분배 방식이 최적이 아닐 때, 개발자가 직접 최적의 방식을 지정할 수 있습니다.

### 기본 구문

```sql
/*+ PQ_DISTRIBUTE(table_alias method_in method_out) */
```

**매개변수 설명**:
- `table_alias`: 힌트를 적용할 테이블 또는 뷰의 별칭
- `method_in`: 해당 테이블이 파이프라인에 "들어올 때"의 분배 방식
- `method_out`: 다른 테이블과 결합한 후 "나갈 때"의 분배 방식

실무에서는 대부분 `method_in`과 `method_out`을 동일한 값으로 지정하거나, 한쪽을 `NONE`으로 지정합니다.

---

## 주요 분배 방식과 적용 시나리오

### 1. HASH 분배: 대규모 데이터의 균등 분할

**적용 구문**:
```sql
/*+ PQ_DISTRIBUTE(table_alias HASH HASH) */
```

**적합한 상황**:
- 두 테이블 모두 대용량 데이터
- 조인 키의 값 분포가 비교적 균등한 경우
- 네트워크 대역폭이 충분한 환경

**실제 적용 예제**:
```sql
SELECT /*+ 
    LEADING(sales) USE_HASH(clicks)
    PARALLEL(sales 16) PARALLEL(clicks 16)
    PQ_DISTRIBUTE(sales HASH HASH)
    PQ_DISTRIBUTE(clicks HASH HASH)
*/
    s.cust_id, 
    SUM(s.amount) AS sales_amount,
    SUM(c.cost) AS click_cost
FROM fact_sales sales
JOIN fact_clicks clicks ON clicks.cust_id = sales.cust_id
GROUP BY s.cust_id;
```

**실행계획 분석**:
- 양쪽 테이블 모두 `PX SEND HASH` 오퍼레이션 확인
- `IN-OUT` 컬럼이 `P->P`로 표시
- 동일한 해시값을 가진 행이 동일한 PX 슬레이브에서 처리

### 2. BROADCAST 분배: 소규모 차원 테이블의 복제

**적용 구문**:
```sql
/*+ PQ_DISTRIBUTE(dimension_table BROADCAST NONE) */
```

**적합한 상황**:
- 차원 테이블이 매우 작은 경우 (일반적으로 수백~수천 행)
- 팩트 테이블이 대용량인 경우
- 네트워크 트래픽 최소화가 필요한 경우

**실제 적용 예제**:
```sql
SELECT /*+ 
    LEADING(customers) USE_HASH(sales)
    PARALLEL(customers 4) PARALLEL(sales 16)
    PQ_DISTRIBUTE(customers BROADCAST NONE)
*/
    s.cust_id,
    SUM(s.amount) AS total_sales
FROM dim_customer customers
JOIN fact_sales sales ON sales.cust_id = customers.cust_id
WHERE customers.grade IN ('GOLD', 'SILVER')
GROUP BY s.cust_id;
```

**실행계획 분석**:
- 차원 테이블(customers)에 `PX SEND BROADCAST` 오퍼레이션 확인
- 팩트 테이블(sales)은 재분배 없이 로컬 처리
- 네트워크 트래픽이 크게 감소

### 3. PARTITION 분배: 파티션 정렬 조인

**적용 구문**:
```sql
/*+ PQ_DISTRIBUTE(table_alias PARTITION PARTITION) */
```

**적합한 상황**:
- 두 테이블이 동일한 방식으로 파티셔닝된 경우
- 파티션 키가 조인 키와 동일한 경우
- RAC 환경에서 인터커넥트 트래픽 최소화가 중요한 경우

**파티션 정렬 조인 유형**:
1. **Full Partition-wise Join**: 두 테이블 모두 동일한 파티셔닝 방식
2. **Partial Partition-wise Join**: 한쪽만 파티셔닝된 경우
3. **Dynamic Partition-wise Join**: 런타임에 가상 정렬 수행

**실제 적용 예제**:
```sql
-- 두 테이블 모두 cust_id로 HASH 파티셔닝된 경우
SELECT /*+ 
    LEADING(sales) USE_HASH(customers)
    PARALLEL(sales 16) PARALLEL(customers 16)
    PQ_DISTRIBUTE(sales PARTITION PARTITION)
    PQ_DISTRIBUTE(customers PARTITION PARTITION)
*/
    s.cust_id,
    c.customer_name,
    SUM(s.amount) AS total_amount
FROM sales_partitioned sales
JOIN customers_partitioned customers 
    ON customers.cust_id = sales.cust_id
GROUP BY s.cust_id, c.customer_name;
```

**실행계획 분석**:
- `PX PARTITION HASH` 오퍼레이션 확인
- `PX SEND` 오퍼레이션이 최소화됨
- 각 파티션이 특정 PX 슬레이브에 고정되어 처리

### 4. NONE 분배: 재분배 생략

**적용 구문**:
```sql
/*+ PQ_DISTRIBUTE(table_alias NONE NONE) */
```

**적합한 상황**:
- 이미 적절히 분배된 데이터에 추가 재분배 불필요한 경우
- 브로드캐스트의 반대편 테이블에 적용
- 로컬 처리만으로 충분한 경우

---

## 실전 튜닝 사례 연구

### 사례 1: 데이터 스큐 문제 해결

**문제 상황**: 특정 고객의 거래량이 매우 많아 HASH 분배 시 특정 PX 슬레이브에 부하 집중

**해결책**: SALT 기법을 통한 균등 분배

```sql
WITH sales_salted AS (
    SELECT /*+ PARALLEL(16) */
        cust_id,
        amount,
        MOD(ORA_HASH(cust_id), 8) AS salt  -- 8개의 SALT 버킷 생성
    FROM fact_sales
),
clicks_salted AS (
    SELECT /*+ PARALLEL(16) */
        cust_id,
        cost,
        MOD(ORA_HASH(cust_id), 8) AS salt
    FROM fact_clicks
)
SELECT /*+ 
    LEADING(sales_salted) USE_HASH(clicks_salted)
    PARALLEL(16)
    PQ_DISTRIBUTE(sales_salted HASH HASH)
    PQ_DISTRIBUTE(clicks_salted HASH HASH)
*/
    sales_salted.cust_id,
    SUM(sales_salted.amount) AS total_sales,
    SUM(clicks_salted.cost) AS total_cost
FROM sales_salted
JOIN clicks_salted 
    ON clicks_salted.cust_id = sales_salted.cust_id
    AND clicks_salted.salt = sales_salted.salt
GROUP BY sales_salted.cust_id;
```

**효과**: 데이터 스큐를 인위적으로 분산시켜 PX 슬레이브 간 부하 균형 달성

### 사례 2: 브로드캐스트 대상 최적화

**문제 상황**: 차원 테이블이 예상보다 커서 브로드캐스트 시 네트워크 부하 증가

**해결책**: 사전 필터링과 MATERIALIZE 활용

```sql
WITH filtered_customers AS (
    SELECT /*+ MATERIALIZE */
        cust_id, grade, active_yn
    FROM dim_customer
    WHERE active_yn = 'Y'
        AND grade IN ('GOLD', 'SILVER')
        AND region = 'APAC'  -- 추가 필터링
)
SELECT /*+ 
    LEADING(filtered_customers) USE_HASH(sales)
    PARALLEL(filtered_customers 4) PARALLEL(sales 16)
    PQ_DISTRIBUTE(filtered_customers BROADCAST NONE)
*/
    sales.cust_id,
    SUM(sales.amount) AS total_amount
FROM filtered_customers
JOIN fact_sales sales ON sales.cust_id = filtered_customers.cust_id
GROUP BY sales.cust_id;
```

**효과**: 브로드캐스트 대상 데이터 크기 최소화로 네트워크 효율성 향상

---

## 실행계획 분석과 모니터링

### 실행계획 확인 방법

```sql
-- 상세 실행계획 확인
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
    NULL, NULL,
    'BASIC +PARALLEL +ALIAS +PREDICATE +NOTE'
));
```

**분석 포인트**:
1. `PX SEND` 오퍼레이션 타입 확인
2. `IN-OUT` 컬럼의 데이터 흐름 확인
3. `NOTE` 섹션의 힌트 적용 여부 확인
4. `BUFFERED` 표시 여부 확인 (임시 버퍼링 발생 시)

### PX 분배 통계 분석

```sql
-- PX 테이블 큐 통계 확인
SELECT 
    dfo_number,
    tq_id,
    server_type,
    inst_id,
    process,
    num_rows,
    bytes,
    ROUND(bytes / 1024 / 1024, 2) AS mb_size
FROM v$pq_tqstat
ORDER BY dfo_number, tq_id, server_type, inst_id, process;
```

**분석 포인트**:
- PX 슬레이브 간 `num_rows` 편차 확인 (스큐 감지)
- 총 이동 데이터량(`bytes`) 분석
- 각 슬레이브의 처리량 비교

### 성능 문제 진단

```sql
-- 병렬 관련 대기 이벤트 확인
SELECT 
    event,
    COUNT(*) AS wait_count,
    ROUND(SUM(time_waited) / 1000, 2) AS total_wait_sec
FROM v$session_event
WHERE sid IN (
    SELECT sid FROM v$px_session WHERE qcsid = SYS_CONTEXT('USERENV','SID')
)
AND event LIKE 'PX%'
GROUP BY event
ORDER BY total_wait_sec DESC;
```

**주요 대기 이벤트**:
- `PX Deq Credit: send blkd`: 송신 블록킹 (스큐 또는 버퍼 부족)
- `PX Deq: Table Q qref latch`: 테이블 큐 래치 경합
- `direct path write/read temp`: TEMP I/O (PGA 부족)

---

## 분배 방식 선택 의사결정 가이드

### 선택 의사결정 매트릭스

| 시나리오 | 권장 분배 방식 | 근거 | 주의사항 |
|---------|---------------|------|----------|
| 소형 차원 + 대형 팩트 | BROADCAST | 팩트 재분배 제거, 네트워크 효율 | 차원 크기 검증 필수 |
| 대형 + 대형, 키 균등 | HASH | 균등 분배, 지역 조인 최적화 | 스큐 모니터링 필요 |
| 동일 파티셔닝 구조 | PARTITION | 재분배 최소, RAC 최적화 | 파티션 정합성 확인 |
| 비파티션 + 파티션 조인 | HASH(비파티션) | 가상 정렬, 부분 PWJ | 비파티션 크기 고려 |
| 심한 데이터 스큐 | SALT + HASH | 인위적 균등화 | 추가 집계 필요 |
| 네트워크 제약 환경 | BROADCAST 또는 PARTITION | 데이터 이동 최소화 | 대상 크기 제한 |

### 실전 적용 원칙

1. **측정 기반 결정**: 실행계획과 실제 성능 측정을 통한 검증
2. **점진적 적용**: 한 번에 하나의 변경사항 적용 및 효과 측정
3. **통계 정확성**: 최신 통계 정보 확보가 올바른 분배 방식 선택의 기초
4. **환경 고려**: 네트워크 대역폭, RAC 구성, 스토리지 성능 등 인프라 요소 반영

### 일반적 최적화 흐름

```sql
-- 1. 기본 실행계획 분석
EXPLAIN PLAN FOR ...;

-- 2. 실제 성능 측정 (통계 수집)
ALTER SESSION SET statistics_level = ALL;
SELECT /*+ MONITOR */ ...;

-- 3. 문제 진단
--    - V$PQ_TQSTAT로 스큐 확인
--    - ASH로 대기 이벤트 분석
--    - 실행계획의 PX SEND 타입 확인

-- 4. PQ_DISTRIBUTE 힌트 적용
SELECT /*+ 
    LEADING(...) USE_HASH(...)
    PARALLEL(...)
    PQ_DISTRIBUTE(...)
*/ ...;

-- 5. 효과 검증
--    - 실행 시간 비교
--    - 리소스 사용량 비교
--    - 스케일링 효율성 평가
```

---

## 주의사항과 일반적인 문제점

### 힌트 적용 실패 원인

1. **별칭 불일치**: 힌트의 테이블 별칭과 실제 별칭이 다를 경우
2. **뷰 병합**: 인라인 뷰나 CTE가 병합되면서 힌트 적용 대상 소실
3. **힌트 충돌**: 여러 힌트 간 상충되는 지시 발생
4. **파티션 조건 불충분**: PARTITION 분배를 위한 전제조건 미충족
5. **통계 부정확**: 잘못된 통계로 인한 옵티마이저 판단 오류

### 성능 악화 가능성

1. **과도한 브로드캐스트**: 큰 테이블 브로드캐스트 시 네트워크 폭주
2. **비균등 HASH 분배**: 스큐 심한 데이터의 HASH 분배 시 특정 슬레이브 과부하
3. **부적절한 DOP**: 병렬도가 너무 높아 관리 오버헤드 증가
4. **메모리 부족**: 분배 방식에 따른 PGA/TEMP 사용량 증가

---

## 결론: PQ_DISTRIBUTE의 전략적 활용

`PQ_DISTRIBUTE` 힌트는 단순한 성능 최적화 도구를 넘어, 대규모 병렬 처리 시스템의 아키텍처 설계에 영향을 미치는 중요한 요소입니다. 효과적인 활용을 위한 핵심 원칙을 정리해보겠습니다.

### 체계적 접근의 중요성

성공적인 `PQ_DISTRIBUTE` 적용은 체계적인 접근 없이는 불가능합니다:

1. **분석 단계**: 실행계획, 성능 통계, 리소스 사용 패턴 이해
2. **설계 단계**: 데이터 특성, 인프라 제약, 비즈니스 요구사항 반영
3. **구현 단계**: 적절한 힌트 적용과 조인 순서/방식 최적화
4. **검증 단계**: 실제 성능 측정과 예상 효과 비교
5. **모니터링 단계**: 지속적인 성능 관찰과 필요시 조정

### 상황별 최적 전략

**데이터 웨어하우스 환경**:
- 파티션 정렬 조인(PARTITION) 우선 고려
- 대규모 팩트 테이블 조인 시 HASH 분배
- 소형 차원 테이블은 브로드캐스트 활용

**OLTP 병렬 처리**:
- 브로드캐스트 최소화 (네트워크 부하 고려)
- HASH 분배의 스큐 모니터링 강화
- DOP 적정화로 관리 오버헤드 제어

**RAC 환경**:
- 파티션 정렬 조인으로 인터커넥트 트래픽 최소화
- 인스턴스 간 데이터 이동 고려한 분배 방식 선택
- 글로벌 캐시 대기 이벤트 모니터링

### 지속적 최적화 문화

`PQ_DISTRIBUTE` 힌트는 일회성 최적화가 아닌 지속적인 관리의 대상입니다:

- **정기적 성능 검토**: 데이터 증가와 접근 패턴 변화에 따른 재평가
- **자동화 모니터링**: 핵심 쿼리의 분배 효율성 자동 감시
- **지식 공유**: 최적화 경험과 모범 사례 팀 내 공유
- **문서화**: 적용된 힌트와 그 이유, 효과에 대한 체계적 기록

### 최종 권고사항

1. **기본값 신뢰**: 옵티마이저의 기본 선택을 먼저 신뢰하고 측정
2. **필요시 개입**: 명확한 문제 진단 후에만 `PQ_DISTRIBUTE` 적용
3. **단계적 접근**: 한 번에 하나의 파라미터 변경과 효과 측정
4. **환경 고려**: 특정 인프라와 데이터 특성에 맞춘 맞춤형 최적화
5. **미래 대비**: 확장성과 유지보수성을 고려한 설계

`PQ_DISTRIBUTE` 힌트의 효과적 활용은 기술적 이해를 넘어 데이터 특성, 비즈니스 요구, 인프라 제약을 종합적으로 고려하는 전략적 사고를 요구합니다. 체계적인 접근과 과학적인 검증을 통해 예측 가능한 성능 향상을 달성하세요.