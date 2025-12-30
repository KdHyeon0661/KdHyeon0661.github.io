---
layout: post
title: DB 심화 - Sort Area 크기 조정
date: 2025-11-24 15:25:23 +0900
category: DB 심화
---

# Sort Area 크기 조정: Oracle PGA 메모리 관리의 원리와 실전

정렬(SORT)과 해시(HASH) 작업은 데이터베이스에서 가장 자원을 많이 소모하는 작업 중 하나입니다. Oracle은 이러한 작업을 효율적으로 처리하기 위해 PGA(Program Global Area) 내에 작업 영역(workarea)을 할당합니다. 이 문서에서는 Oracle의 PGA 메모리 관리 방식, 특히 `WORKAREA_SIZE_POLICY=AUTO` 모드에서의 동작 원리를 심층적으로 분석하고, 실전에서의 효과적인 튜닝 방법을 제시합니다.

## 작업 영역(Workarea)의 이해

### 작업 영역이 사용되는 연산

작업 영역은 단순히 정렬 메모리뿐만 아니라 PGA 기반의 다양한 중간 연산을 위한 버퍼로 사용됩니다:

- **정렬 관련 연산**: `SORT ORDER BY`, `SORT GROUP BY`, `SORT UNIQUE`, `WINDOW SORT`
- **해시 관련 연산**: `HASH JOIN`, `HASH GROUP BY`, `DISTINCT` 처리
- **기타 연산**: Bitmap 병합/생성, 대용량 XML/JSON 처리 등

이러한 작업들은 모두 PGA 메모리를 사용하며, 작업 영역 크기가 성능에 직접적인 영향을 미칩니다.

### 작업 처리의 세 가지 모드

Oracle은 작업 영역 크기에 따라 데이터 처리 방식을 세 가지로 구분합니다:

1. **Optimal 모드**: 모든 작업이 메모리 내에서 완료됩니다. 디스크 I/O가 발생하지 않으므로 가장 효율적입니다.
2. **One-pass 모드**: 작업 데이터가 메모리를 초과하여 디스크 임시 공간(TEMP)으로 한 번 스필(spill)됩니다. 스필된 데이터를 한 번 재읽어 작업을 완료합니다.
3. **Multi-pass 모드**: 데이터가 여러 번 디스크와 메모리 간을 오가며, 상당한 성능 저하를 초래합니다.

실무에서의 목표는 "Multi-pass를 근절하고, One-pass를 최소화하며, Optimal을 최대화"하는 것입니다.

## PGA 메모리 관리 방식

### MANUAL 모드

MANUAL 모드는 사용자가 직접 `SORT_AREA_SIZE`와 `HASH_AREA_SIZE` 파라미터를 고정값으로 설정하는 방식입니다.

```sql
ALTER SYSTEM SET workarea_size_policy = MANUAL;
ALTER SYSTEM SET sort_area_size = 128M;
ALTER SYSTEM SET hash_area_size = 128M;
```

**단점:**
- 병렬 처리나 다중 세션 환경에서 예측 불가능한 동작
- 고정값이 과대면 PGA 폭주, 과소면 Multi-pass 발생
- 현대 버전에서는 특수한 경우를 제외하고 권장하지 않습니다.

### AUTO 모드 (권장)

AUTO 모드는 Oracle이 동적으로 작업 영역 크기를 관리하는 방식입니다.

```sql
ALTER SYSTEM SET workarea_size_policy = AUTO;
ALTER SYSTEM SET pga_aggregate_target = 8G;
ALTER SYSTEM SET pga_aggregate_limit = 16G;
```

**작동 원리:**
1. Oracle은 현재 활성화된 작업 영역 수를 모니터링합니다.
2. PGA 예산을 고려하여 Global Memory Bound(GMB)를 계산합니다.
3. 각 작업 영역에 `Grant = min(ExpectedOptimalSize, GlobalMemoryBound)` 크기를 할당합니다.

### AMM(Automatic Memory Management)

AMM은 SGA와 PGA를 통합하여 관리하는 방식입니다.

```sql
ALTER SYSTEM SET memory_target = 16G;
ALTER SYSTEM SET memory_max_target = 32G;
```

**특징:**
- SGA와 PGA 간의 메모리 재분배가 동적으로 발생
- PGA의 변동 폭이 커질 수 있어 모니터링 강화 필요

## AUTO 모드의 크기 결정 원리

### Global Memory Bound(GMB)의 의미

GMB는 "현재 시스템에서 개별 작업 영역이 할당받을 수 있는 최대 메모리 크기"를 의미합니다. 이 값은 다음과 같은 요소에 의해 결정됩니다:

```
GMB ≈ min(개별작업영역상한, 가용PGA / 활성작업영역수)
```

GMB는 시스템 부하에 따라 실시간으로 변동합니다:
- 시스템 부하가 낮을 때: GMB 증가 → Optimal 모드 가능성 증가
- 시스템 부하가 높을 때: GMB 감소 → One-pass/Multi-pass 가능성 증가

### 실제 할당 크기(Grant) 결정

각 작업 영역에 실제로 할당되는 메모리 크기는 다음 공식으로 결정됩니다:

```
Grant = min(ExpectedOptimalSize, GlobalMemoryBound)
```

여기서 `ExpectedOptimalSize`는 옵티마이저가 추정한 "디스크 스필 없이 메모리에서 작업을 완료할 수 있는 이상적인 크기"입니다.

### 실행 모드 판단 기준

할당된 메모리 크기에 따라 실행 모드가 결정됩니다:

```sql
IF Grant >= ExpectedOptimalSize THEN
    실행 모드 = Optimal
ELSIF Grant >= ExpectedOnePassSize THEN
    실행 모드 = One-pass
ELSE
    실행 모드 = Multi-pass
END IF;
```

## 실전 모니터링과 분석

### 시스템 전반 상태 확인

```sql
-- PGA 전반 상태 확인
SELECT name, value/1024/1024 AS value_mb
FROM v$pgastat
WHERE name IN (
    'total PGA allocated',
    'total PGA inuse',
    'global memory bound',
    'over allocation count',
    'pga aggregate target',
    'pga aggregate limit'
);

-- 작업 영역 실행 통계
SELECT sn.name, ms.value
FROM v$mystat ms 
JOIN v$statname sn ON sn.stat# = ms.stat#
WHERE sn.name IN (
    'workarea executions - optimal',
    'workarea executions - onepass',
    'workarea executions - multipass',
    'sorts (memory)',
    'sorts (disk)'
);
```

### 활성 작업 영역 분석

```sql
-- 현재 실행 중인 작업 영역 분석
SELECT 
    sid,
    operation_type,
    ROUND(expected_optimal_size/1024/1024, 2) AS exp_opt_mb,
    ROUND(expected_onepass_size/1024/1024, 2) AS exp_one_mb,
    ROUND(last_memory_used/1024/1024, 2) AS last_mb,
    active_time/100 AS active_sec,
    CASE 
        WHEN last_memory_used >= expected_optimal_size THEN 'Optimal'
        WHEN last_memory_used >= expected_onepass_size THEN 'One-pass'
        ELSE 'Multi-pass'
    END AS execution_mode
FROM v$sql_workarea_active
ORDER BY active_time DESC;
```

### 과거 작업 통계 분석

```sql
-- 완료된 작업의 통계 분석
SELECT 
    operation_type,
    optimal_executions,
    onepass_executions,
    multipasses_executions,
    ROUND(optimal_executions * 100.0 / 
          NULLIF(optimal_executions + onepass_executions + multipasses_executions, 0), 2) AS optimal_pct,
    ROUND(onepass_executions * 100.0 / 
          NULLIF(optimal_executions + onepass_executions + multipasses_executions, 0), 2) AS onepass_pct
FROM v$sql_workarea
WHERE total_executions > 0
ORDER BY total_executions DESC;
```

## 실습: 정렬 작업의 메모리 사용 관찰

### 테스트 데이터 준비

```sql
-- 대용량 테스트 테이블 생성
CREATE TABLE test_sort_data AS
SELECT 
    rownum AS id,
    DBMS_RANDOM.STRING('A', 50) AS data1,
    DBMS_RANDOM.STRING('A', 50) AS data2,
    DBMS_RANDOM.STRING('A', 50) AS data3
FROM dual
CONNECT BY LEVEL <= 1000000;

-- 통계 수집
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'TEST_SORT_DATA');
```

### 다양한 PGA 설정에서의 성능 비교

```sql
-- 케이스 1: 낮은 PGA 설정에서 정렬 수행
ALTER SYSTEM SET pga_aggregate_target = 512M;

-- 정렬 작업 실행
SELECT /*+ MONITOR */ *
FROM test_sort_data
ORDER BY data1, data2, data3;

-- 작업 영역 상태 확인
SELECT 
    operation_type,
    ROUND(expected_optimal_size/1024/1024, 2) AS exp_opt_mb,
    ROUND(last_memory_used/1024/1024, 2) AS last_mb,
    active_time/100 AS active_sec
FROM v$sql_workarea
WHERE sql_id = (SELECT prev_sql_id FROM v$session WHERE sid = SYS_CONTEXT('USERENV', 'SID'));
```

```sql
-- 케이스 2: 높은 PGA 설정에서 정렬 수행
ALTER SYSTEM SET pga_aggregate_target = 4G;

-- 동일한 정렬 작업 실행
SELECT /*+ MONITOR */ *
FROM test_sort_data
ORDER BY data1, data2, data3;

-- 결과 비교
SELECT 
    operation_type,
    ROUND(expected_optimal_size/1024/1024, 2) AS exp_opt_mb,
    ROUND(last_memory_used/1024/1024, 2) AS last_mb,
    active_time/100 AS active_sec
FROM v$sql_workarea
WHERE sql_id = (SELECT prev_sql_id FROM v$session WHERE sid = SYS_CONTEXT('USERENV', 'SID'));
```

## 병렬 처리와 작업 영역

### 병렬 처리에서의 메모리 요구량

병렬 처리는 작업 영역 요구량에 곱셈 효과를 가져옵니다:

```
총 메모리 요구량 ≈ (작업당 메모리 요구량) × DOP(Degree of Parallelism)
```

예를 들어 DOP=8인 해시 조인은 빌드 해시 테이블이 8개로 분산될 수 있으며, 각 병렬 프로세스마다 별도의 작업 영역이 필요합니다.

### 병렬 처리 모니터링

```sql
-- 병렬 작업 영역 모니터링
SELECT 
    sid,
    qcsid,
    degree,
    ROUND(expected_optimal_size/1024/1024, 2) AS exp_opt_mb,
    ROUND(last_memory_used/1024/1024, 2) AS last_mb,
    active_time/100 AS active_sec
FROM v$sql_workarea_active
WHERE degree > 1
ORDER BY last_memory_used DESC;
```

## 성능 튜닝 전략

### 1. 작업 자체를 최소화하는 SQL 재작성

메모리 확장보다 작업 자체를 줄이는 것이 가장 효과적인 튜닝입니다.

#### 인덱스를 활용한 정렬 제거

```sql
-- 정렬이 필요한 쿼리
SELECT sales_id, amount
FROM sales
WHERE sale_date >= :start_date
ORDER BY amount DESC
FETCH FIRST 100 ROWS ONLY;

-- 인덱스 생성으로 정렬 제거
CREATE INDEX ix_sales_amount_desc ON sales(amount DESC);

SELECT /*+ INDEX_DESC(sales ix_sales_amount_desc) */
       sales_id, amount
FROM sales
WHERE sale_date >= :start_date
FETCH FIRST 100 ROWS ONLY;
```

#### 행 폭 축소를 통한 정렬 최적화

```sql
-- 원본: 모든 컬럼을 정렬
SELECT * FROM large_table ORDER BY sort_key;

-- 개선: 필요한 컬럼만 정렬 후 조인
WITH sorted_keys AS (
    SELECT rowid AS rid, sort_key
    FROM large_table
    WHERE conditions = :value
    ORDER BY sort_key
)
SELECT lt.*
FROM sorted_keys sk
JOIN large_table lt ON lt.rowid = sk.rid;
```

### 2. 해시 조인 최적화

```sql
-- 비효율적인 해시 조인
SELECT /*+ USE_HASH(large) */
       large.*, small.description
FROM large_table large
JOIN small_table small ON small.id = large.small_id;

-- 효율적인 해시 조인 (작은 테이블을 빌드로)
SELECT /*+ LEADING(small) USE_HASH(large) */
       large.*, small.description
FROM small_table small
JOIN large_table large ON large.small_id = small.id
WHERE small.category = 'ACTIVE';
```

### 3. PGA 타깃 조정 가이드라인

#### PGA 증설 효과 예측

```sql
-- PGA 타깃 조정 조언 확인
SELECT 
    ROUND(pga_target_for_estimate/1024/1024, 2) AS target_mb,
    estd_pga_cache_hit_percentage AS hit_pct,
    estd_extra_bytes_rw/1024/1024 AS extra_io_mb,
    estd_overalloc_count AS overalloc_cnt
FROM v$pga_target_advice
ORDER BY pga_target_for_estimate;
```

#### PGA 설정 권장사항

1. **초기 설정**: `PGA_AGGREGATE_TARGET`을 전체 메모리의 20-25%로 설정
2. **모니터링**: `V$PGASTAT`와 `V$SQL_WORKAREA`를 정기적으로 확인
3. **조정**: hit_pct가 95% 미만이면 점진적으로 증설
4. **제한**: `PGA_AGGREGATE_LIMIT`을 타깃의 2배 정도로 설정

### 4. 시스템 부하 관리

#### 동시성 제어

```sql
-- 리소스 관리자를 통한 동시성 제어
BEGIN
    DBMS_RESOURCE_MANAGER.CREATE_PENDING_AREA();
    
    DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP(
        consumer_group => 'REPORT_GROUP',
        comment => '리포트 전용 그룹'
    );
    
    DBMS_RESOURCE_MANAGER.CREATE_PLAN(
        plan => 'DAYTIME_PLAN',
        comment => '주간 운영 계획'
    );
    
    -- 리포트 그룹의 병렬도 제한
    DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
        plan => 'DAYTIME_PLAN',
        group_or_subplan => 'REPORT_GROUP',
        comment => '리포트 작업 제한',
        parallel_degree_limit_p1 => 4
    );
    
    DBMS_RESOURCE_MANAGER.VALIDATE_PENDING_AREA();
    DBMS_RESOURCE_MANAGER.SUBMIT_PENDING_AREA();
END;
/
```

#### 작업 스케줄링

```sql
-- DBMS_SCHEDULER를 이용한 작업 스케줄링
BEGIN
    DBMS_SCHEDULER.CREATE_JOB(
        job_name => 'NIGHTLY_REPORT_JOB',
        job_type => 'PLSQL_BLOCK',
        job_action => 'BEGIN run_complex_reports(); END;',
        start_date => SYSTIMESTAMP,
        repeat_interval => 'FREQ=DAILY; BYHOUR=2; BYMINUTE=0',
        enabled => TRUE,
        comments => '야간 리포트 배치 작업'
    );
END;
/
```

## 문제 진단과 해결

### 일반적인 문제 패턴과 해결책

| 증상 | 가능한 원인 | 해결 방안 |
|------|-------------|-----------|
| 낮 시간대 성능 저하 | 동시 작업 증가로 GMB 감소 | 작업 스케줄 분산, DOP 제한 |
| 특정 쿼리만 느림 | 과도한 정렬/해시 작업 | 쿼리 재작성, 인덱스 추가 |
| TEMP 공간 급증 | Multi-pass 작업 발생 | PGA 증설, 작업 최적화 |
| 해시 조인 성능 저하 | 빌드 테이블 과대 | 조인 순서 변경, 필터 조건 추가 |

### 진단 체크리스트

1. **기본 설정 확인**: `WORKAREA_SIZE_POLICY=AUTO`인지 확인
2. **현황 분석**: `V$PGASTAT`로 GMB와 할당 현황 확인
3. **작업 패턴 파악**: `V$SQL_WORKAREA`로 Optimal/One-pass/Multi-pass 비율 분석
4. **문제 쿼리 식별**: 가장 많은 Multi-pass를 발생시키는 쿼리 찾기
5. **SQL 최적화**: 인덱스 활용, 정렬 최소화, 해시 조인 개선
6. **시스템 조정**: 필요시 PGA 증설, 동시성 제어
7. **모니터링 유지**: 지속적인 성능 모니터링과 튜닝

## 결론

Oracle의 PGA 메모리 관리는 복잡하지만 체계적인 원리를 가지고 있습니다. 효과적인 튜닝을 위해서는 다음 원칙을 기억하세요:

### 핵심 원칙

1. **작업 최소화가 최선**: 메모리 확장보다 SQL 재작성을 통한 작업 자체 감소가 가장 효과적입니다.
2. **시스템적 접근**: 개별 쿼리 최적화와 시스템 전체 리소스 관리의 균형이 필요합니다.
3. **증거 기반 튜닝**: 모니터링 데이터를 기반으로 한 과학적 접근이 필수적입니다.
4. **예방적 관리**: 사후 대응보다 사전 예방과 지속적인 모니터링이 중요합니다.

### 실전 적용 가이드라인

1. **초기 설정**: `PGA_AGGREGATE_TARGET`을 전체 메모리의 적절한 비율로 설정하고 AUTO 모드 사용
2. **정기 모니터링**: `V$PGASTAT`, `V$SQL_WORKAREA` 뷰를 활용한 주기적 점검
3. **SQL 품질 관리**: 대용량 정렬/해시 작업이 포함된 쿼리의 지속적 최적화
4. **시스템 운영**: 피크 시간대 작업 분산, 리소스 그룹을 활용한 워크로드 관리
5. **용량 계획**: `V$PGA_TARGET_ADVICE`를 참고한 미래 용량 계획 수립

PGA 메모리 관리는 단순한 파라미터 튜닝을 넘어 SQL 품질, 시스템 아키텍처, 운영 프로세스 전반에 걸친 종합적인 접근이 필요합니다. 본 문서에서 소개한 원리와 기법을 바탕으로 체계적인 성능 관리 체계를 구축하시기 바랍니다.