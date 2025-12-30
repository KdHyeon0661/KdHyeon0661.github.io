---
layout: post
title: DB 심화 - Parallel ORDER BY & Parallel GROUP BY
date: 2025-11-24 23:25:23 +0900
category: DB 심화
---

# Oracle 병렬 실행: Parallel ORDER BY & Parallel GROUP BY

병렬 처리는 대용량 데이터를 효율적으로 처리하기 위한 핵심 기술입니다. 그러나 정렬(ORDER BY)과 집계(GROUP BY) 작업을 병렬로 실행하는 것은 단순한 데이터 분할보다 훨씬 복잡합니다. 이 문서에서는 Oracle 데이터베이스에서 Parallel ORDER BY와 Parallel GROUP BY의 동작 원리, 성능 병목 현상, 그리고 실전 튜닝 기법을 심층적으로 분석합니다.

## 병렬 처리의 기본 개념 이해

### 병렬 처리의 도전과제

병렬 처리는 "같은 작업을 여러 프로세스가 나누어 처리"하는 것으로 보이지만, 정렬과 집계 작업에서는 추가적인 복잡성이 발생합니다. 

- **ORDER BY**: 전역적으로 정렬된 결과를 생성하려면 각 병렬 프로세스가 로컬 정렬을 수행한 후, 이 결과들을 다시 모아 전역 병합을 수행해야 합니다.
- **GROUP BY**: 동일한 그룹 키를 가진 데이터는 최종 집계를 위해 같은 장소로 모여야 합니다.

이러한 특성으로 인해 병렬 정렬과 집계 작업에서는 네트워크 통신, 메모리 관리, 데이터 스큐(skew) 문제가 성능을 좌우하는 핵심 요소가 됩니다.

### 병렬 실행 아키텍처

Oracle의 병렬 실행 아키텍처를 이해하는 것이 중요합니다:

1. **QC(Query Coordinator)**: SQL을 파싱하고 최적화하며, 병렬 서버들을 관리하고 최종 결과를 조립합니다.
2. **PX Servers(Slaves)**: 실제 데이터 스캔, 조인, 정렬, 집계 작업을 수행하는 병렬 프로세스입니다.
3. **DFO(Data Flow Operation)**: 병렬 실행의 작업 파이프라인 단위입니다.
4. **TQ(Table Queue)**: 병렬 프로세스 간 또는 프로세스와 코디네이터 간 데이터를 전달하는 통로입니다.

## Parallel ORDER BY: 2단계 정렬과 RANGE 재분배

### 작동 원리

Parallel ORDER BY는 기본적으로 2단계 정렬 프로세스를 따릅니다:

1. **로컬 정렬(Local Sort)**: 각 병렬 프로세스가 할당받은 데이터 블록을 정렬합니다.
2. **RANGE 재분배**: 정렬 키를 기준으로 데이터를 범위별로 재분배합니다.
3. **전역 병합(Global Merge)**: 재분배된 데이터를 병합하여 최종 정렬 결과를 생성합니다.

### 예제: 기본 Parallel ORDER BY

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) */
       f.sales_id, f.sales_dt, f.amount
FROM   fact_sales f
WHERE  f.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-01'
ORDER  BY f.amount DESC;

SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,
         'BASIC +PARALLEL +ALIAS +PARTITION +PREDICATE +NOTE'));
```

실행 계획에서 다음 패턴을 확인할 수 있습니다:
- `PX BLOCK ITERATOR`: 데이터 블록을 병렬 프로세스에 분배
- `PX SEND RANGE` / `PX RECEIVE`: 정렬 키 기준 범위 재분배
- `SORT ORDER BY`: 전역 정렬 병합

### RANGE 재분배의 중요성

ORDER BY에서 RANGE 재분배를 사용하는 이유는 전역 정렬 순서를 유지하기 위해서입니다. 각 병렬 프로세스가 서로 겹치지 않는 정렬 키 범위를 처리하면, 최종 병합 단계에서 단순히 각 프로세스의 결과를 연결하기만 해도 정렬된 결과를 얻을 수 있습니다.

### Top-N 쿼리의 병렬 처리

Top-N 쿼리(FETCH FIRST N ROWS)가 포함된 경우, 병렬 처리 방식이 변화합니다:

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) */
       f.sales_id, f.sales_dt, f.amount
FROM   fact_sales f
WHERE  f.sales_dt >= DATE '2025-02-01' AND f.sales_dt < DATE '2025-04-01'
ORDER  BY f.amount DESC
FETCH  FIRST 100 ROWS ONLY;
```

Top-N 쿼리에서는:
- 각 병렬 프로세스가 로컬 Top-N만 유지합니다.
- 전달되는 데이터 양이 크게 감소합니다.
- 실행 계획에 `SORT ORDER BY STOPKEY`가 나타납니다.

## Parallel ORDER BY 성능 병목 현상과 해결책

### 1. TEMP 스필(Spill) 문제

정렬 작업이 PGA(Program Global Area) 메모리를 초과하면 디스크 임시 공간(TEMP)으로 데이터가 스필됩니다. 병렬 환경에서는 이 문제가 배로 발생합니다.

**해결 전략:**
- 정렬 대상 데이터 양을 줄입니다(필터 조건 추가, 필요한 컬럼만 선택).
- PGA 메모리를 적절히 구성합니다(`WORKAREA_SIZE_POLICY=AUTO` 권장).
- Top-N 패턴을 활용하여 정렬 데이터를 최소화합니다.

### 2. TQ/네트워크 병목

병렬 프로세스 간 데이터 이동이 성능 병목이 될 수 있습니다. 특히 `PX Deq Credit: send blkd` 대기 이벤트는 생산자 프로세스가 소비자 프로세스의 처리 속도를 따라가지 못할 때 발생합니다.

**해결 전략:**
- 데이터 스큐를 완화합니다(균등한 데이터 분배).
- 적절한 DOP(Degree of Parallelism)를 설정합니다.
- Top-N 쿼리를 사용하여 전송 데이터 양을 줄입니다.

### 3. 정렬 키 스큐(Skew) 문제

특정 정렬 키 값에 데이터가 집중되면, 해당 범위를 처리하는 병렬 프로세스의 부하가 불균형하게 증가합니다.

## Parallel ORDER BY 실전 튜닝 패턴

### 패턴 1: 정렬 전 데이터 최소화

정렬 비용은 (행 수 × 행 폭)에 비례합니다. 정렬 전에 데이터를 최대한 줄이는 것이 핵심입니다.

```sql
WITH topn AS (
  SELECT /*+ parallel(f 8) index_desc(f ix_sales_amount_desc) */
         sales_id, amount
  FROM   fact_sales f
  WHERE  sales_dt >= :d1 AND sales_dt < :d2
  FETCH FIRST 100 ROWS ONLY
)
SELECT /*+ parallel(s 8) */
       s.sales_id, s.prod_id, s.sales_dt, s.qty, s.amount
FROM   topn t
JOIN   fact_sales s ON s.sales_id = t.sales_id;
```

### 패턴 2: 인덱스를 활용한 정렬 제거

적절한 인덱스 설계로 정렬 작업 자체를 제거할 수 있습니다.

```sql
-- (sales_dt, amount DESC) 인덱스가 있다면
SELECT /*+ parallel(f 8) index(f ix_sales_dt_amount_desc) */
       sales_id, sales_dt, amount
FROM   fact_sales f
WHERE  sales_dt >= :d1 AND sales_dt < :d2
ORDER BY sales_dt, amount DESC;
```

### 패턴 3: 파티션 단위 정렬

대용량 테이블이 파티션되어 있다면, 파티션 단위로 정렬을 수행할 수 있습니다.

```sql
-- 파티션별 Top-N 후 병합
SELECT *
FROM (
  SELECT /*+ parallel(f 8) */
         sales_id, sales_dt, amount,
         ROW_NUMBER() OVER (PARTITION BY TRUNC(sales_dt, 'MM') 
                            ORDER BY amount DESC) rn
  FROM   fact_sales f
  WHERE  sales_dt >= :d1 AND sales_dt < :d2
)
WHERE rn <= 100;
```

## Parallel GROUP BY: 부분 집계와 HASH 재분배

### 작동 원리

Parallel GROUP BY는 일반적으로 2단계 집계 프로세스를 따릅니다:

1. **부분 집계(Partial Aggregation)**: 각 병렬 프로세스가 로컬 데이터에 대한 부분 집계를 수행합니다.
2. **HASH 재분배**: 그룹 키를 기준으로 데이터를 재분배하여 동일한 키를 가진 데이터를 같은 프로세스로 모읍니다.
3. **최종 집계(Final Aggregation)**: 모인 데이터에 대한 최종 집계를 수행합니다.

### 예제: 기본 Parallel GROUP BY

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 16) */
       f.cust_id, SUM(f.amount) amt
FROM   fact_sales f
WHERE  f.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-01'
GROUP  BY f.cust_id;

SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL +ALIAS +PARTITION +NOTE'));
```

실행 계획에서 확인할 수 있는 패턴:
- `HASH GROUP BY (PARTIAL)`: 부분 집계 단계
- `PX SEND HASH` / `PX RECEIVE`: 해시 기반 재분배
- `HASH GROUP BY`: 최종 집계 단계

### 부분 집계의 중요성

부분 집계 단계는 병렬 GROUP BY 성능의 핵심입니다. 각 병렬 프로세스가 로컬에서 집계를 수행하면, 네트워크를 통해 전송해야 할 데이터 양이 크게 감소합니다.

## Parallel GROUP BY 성능 병목 현상과 해결책

### 1. 그룹 키 스큐 문제

특정 그룹 키에 데이터가 집중되면 해당 키를 처리하는 병렬 프로세스의 부하가 불균형해집니다. 이는 병렬 처리의 가장 큰 도전과제 중 하나입니다.

**해결 전략 - Salting 기법:**
```sql
WITH salted AS (
  SELECT /*+ parallel(f 16) */
         cust_id,
         MOD(ORA_HASH(cust_id), 8) salt,  -- 8은 DOP에 맞춰 조정
         SUM(amount) amt
  FROM   fact_sales f
  GROUP  BY cust_id, MOD(ORA_HASH(cust_id), 8)
)
SELECT cust_id, SUM(amt) amt
FROM   salted
GROUP  BY cust_id;
```

Salting 기법은 동일한 그룹 키에 소금(salt) 값을 추가하여 데이터를 균등하게 분배한 후, 최종적으로 소금 값을 제거하고 재집계합니다.

### 2. 해시 테이블 스필

집계 작업에서 사용하는 해시 테이블이 PGA 메모리를 초과하면 TEMP 스필이 발생합니다.

**해결 전략:**
- 집계 대상 데이터를 사전에 필터링합니다.
- 필요한 컬럼만 선택하여 행 폭을 줄입니다.
- PGA 메모리를 적절히 구성합니다.

### 3. 소형 차원 테이블의 BROADCAST 분배

작은 차원 테이블을 조인에 사용할 때는 BROADCAST 분배가 효율적일 수 있습니다.

```sql
SELECT /*+ leading(c) use_hash(f) parallel(f 16) parallel(c 16)
           pq_distribute(c BROADCAST NONE) */
       c.region_name, SUM(f.amount)
FROM   dim_customer c
JOIN   fact_sales  f ON f.cust_id = c.cust_id
GROUP  BY c.region_name;
```

## 병렬 실행 모니터링과 진단

### 실행 계획 분석

```sql
ALTER SESSION SET statistics_level = ALL;

-- 쿼리 실행
SELECT /*+ monitor parallel(f 16) */
       cust_id, SUM(amount)
FROM   fact_sales f
GROUP  BY cust_id;

-- 실행 계획 확인
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
     'BASIC +PARALLEL +ALIAS +PARTITION +PREDICATE +NOTE'));
```

### TQ 분배 상태 확인

```sql
SELECT dfo_number, tq_id, server_type,
       process, num_rows, bytes,
       ROUND(100 * num_rows / SUM(num_rows) OVER (PARTITION BY dfo_number, tq_id), 2) pct
FROM   v$pq_tqstat
ORDER  BY dfo_number, tq_id, server_type, process;
```

이 쿼리는 각 병렬 프로세스의 처리 데이터 양과 비율을 보여줍니다. 비율이 균등하지 않으면 데이터 스큐가 존재함을 의미합니다.

### 워크에어리어 모니터링

```sql
SELECT sid, operation_type, policy,
       expected_optimal_size/1024/1024 exp_opt_mb,
       last_memory_used/1024/1024 last_mb,
       active_time/100 active_sec
FROM   v$sql_workarea_active
WHERE  operation_type IN ('SORT', 'HASH-JOIN', 'GROUP BY')
ORDER  BY active_time DESC;
```

### TEMP 사용량 모니터링

```sql
SELECT tablespace, 
       SUM(blocks)*8192/1024/1024 temp_mb,
       COUNT(DISTINCT session_addr) sessions
FROM   v$tempseg_usage
WHERE  tablespace = 'TEMP'
GROUP  BY tablespace;
```

## DOP와 메모리의 상호 관계

병렬 처리에서 DOP(Degree of Parallelism)와 메모리 요구량은 직접적인 상관 관계가 있습니다:

```
총 메모리 요구량 ≈ (프로세스당 메모리 요구량) × DOP
```

이 관계를 이해하는 것이 병렬 처리 튜닝의 핵심입니다:
- DOP를 증가시키면 메모리 요구량도 선형적으로 증가합니다.
- 메모리가 부족하면 TEMP 스필이 발생하여 오히려 성능이 저하됩니다.
- 적절한 DOP는 사용 가능한 메모리 리소스와 데이터 특성을 고려하여 결정해야 합니다.

## 결론

Parallel ORDER BY와 Parallel GROUP BY는 대용량 데이터 처리에서 필수적인 기술이지만, 올바르게 사용하지 않으면 오히려 성능을 저하시킬 수 있습니다. 효과적인 병렬 처리 튜닝을 위한 핵심 원칙은 다음과 같습니다:

### Parallel ORDER BY 요약

1. **2단계 정렬 구조 이해**: 로컬 정렬 → RANGE 재분배 → 전역 병합의 흐름을 이해해야 합니다.
2. **Top-N 최적화**: `FETCH FIRST N ROWS`는 병렬 정렬 성능을 크게 향상시킬 수 있습니다.
3. **데이터 최소화**: 정렬 전에 필터링과 프로젝션으로 데이터 양을 줄이는 것이 가장 효과적입니다.
4. **인덱스 활용**: 적절한 인덱스 설계로 정렬 작업 자체를 제거할 수 있습니다.

### Parallel GROUP BY 요약

1. **부분 집계의 중요성**: 로컬 부분 집계는 네트워크 부하를 크게 감소시킵니다.
2. **데이터 스큐 관리**: Salting 기법과 같은 스큐 완화 전략이 필수적입니다.
3. **메모리 관리**: PGA 메모리와 DOP의 균형을 맞추어 TEMP 스필을 방지해야 합니다.
4. **분배 전략 선택**: 데이터 크기와 특성에 맞는 분배 전략(BROADCAST, HASH 등)을 선택해야 합니다.

### 공통 튜닝 원칙

1. **측정 기반 접근**: 실행 계획, TQ 분배 상태, 메모리 사용량을 정기적으로 모니터링합니다.
2. **점진적 최적화**: DOP, 메모리 설정, 쿼리 구조를 점진적으로 조정하며 효과를 측정합니다.
3. **시스템 전체 관점**: 단일 쿼리 최적화가 아닌 시스템 전체 리소스 사용 관점에서 접근합니다.
4. **실제 데이터 특성 반영**: 테스트 데이터가 아닌 실제 운영 데이터 특성을 고려한 튜닝이 필요합니다.

병렬 처리는 강력한 도구이지만, 이 도구를 효과적으로 사용하기 위해서는 데이터베이스 내부 동작 원리에 대한 깊은 이해와 체계적인 접근 방법이 필요합니다. 본 문서에서 소개한 원리와 기법을 바탕으로 실제 환경에 적용해 보시기 바랍니다.