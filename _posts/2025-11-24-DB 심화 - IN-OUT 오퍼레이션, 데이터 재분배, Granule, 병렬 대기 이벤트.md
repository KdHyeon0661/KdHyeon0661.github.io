---
layout: post
title: DB 심화 - IN-OUT 오퍼레이션, 데이터 재분배, Granule, 병렬 대기 이벤트
date: 2025-11-24 22:25:23 +0900
category: DB 심화
---
# Oracle 병렬 실행 심화 분석: IN-OUT 오퍼레이션, 데이터 재분배, Granule, 병렬 대기 이벤트

## 병렬 실행의 네 가지 핵심 요소: 통합적 이해의 중요성

Oracle의 병렬 실행을 효과적으로 이해하고 최적화하기 위해서는 네 가지 핵심 요소를 통합적으로 분석해야 합니다. 이 네 가지 요소는 서로 깊이 연관되어 있으며, 한 가지를 이해하지 않고서는 다른 요소의 문제를 해결할 수 없습니다.

1. **데이터 흐름의 방향성**: IN-OUT 오퍼레이션을 통해 데이터가 어디서 어디로 이동하는지 이해
2. **데이터 분배 메커니즘**: PX SEND/RECEIVE와 TQ를 통해 데이터가 어떻게 재분배되는지 분석
3. **작업 분할 단위**: Granule이 어떻게 작업을 나누어 분배하는지 파악
4. **성능 병목 지점**: 병렬 대기 이벤트를 통해 시스템의 실제 병목을 식별

이 네 가지 요소를 종합적으로 분석할 때 비로소 실행 계획에서 보이는 원인과 ASH/AWR에서 관찰되는 증상이 정확히 연결됩니다.

---

## 병렬 실행의 기본 아키텍처 이해

### 병렬 실행 구성 요소

**QC(Query Coordinator)**
SQL 실행의 지휘자 역할을 하는 직렬 프로세스로, 실행 계획을 수립하고 병렬 서버들을 관리합니다.

**PX 서버 집합(Parallel Execution Server Set)**
실제 작업을 수행하는 병렬 워커 프로세스들의 집합으로, Producer와 Consumer 그룹으로 구성되어 협업합니다.

**DFO(Data Flow Operation)**
병렬 실행 파이프라인에서 하나의 완전한 데이터 흐름 단위를 의미합니다. 각 DFO 경계마다 서버 집합이 변경되고 새로운 TQ가 생성됩니다.

**TQ(Table Queue)**
PX 서버 집합 간에 데이터를 전달하는 내부 큐 메커니즘으로, 항상 PX SEND와 PX RECEIVE 사이에 존재합니다.

### 전형적인 병렬 실행 흐름

```
Query Coordinator (QC)
    │
    └─── PX Server Set 1 (Producer) ──── PX SEND ────> TQ ────> PX RECEIVE ────┐
                            │                                                 │
                            │                                                 ↓
                            └───────────────────────────────────────────── PX Server Set 2 (Consumer)
                                                                                    │
                                                                                    ↓
                                                                                결과를 QC로 반환
```

이 구조에서 PX SEND와 PX RECEIVE가 보이는 곳은 항상 데이터 재분배가 발생하는 지점입니다.

---

## IN-OUT 오퍼레이션: 데이터 흐름의 방향성 분석

### IN-OUT 표기법의 의미

`DBMS_XPLAN.DISPLAY_CURSOR` 함수에 `+PARALLEL` 옵션을 사용하면 IN-OUT 열이 나타나며, 이는 각 오퍼레이션의 데이터 흐름 방향을 나타냅니다.

**주요 IN-OUT 값과 의미:**

| IN-OUT 값 | 의미 |
|-----------|------|
| **S->P** | 직렬 단계에서 병렬 서버 집합으로 데이터가 전달됨 |
| **P->S** | 병렬 서버 집합에서 직렬 단계(주로 QC)로 데이터가 반환됨 |
| **P->P** | 병렬 서버 집합 간에 데이터가 전달됨 (재분배 발생) |
| **PCWP** | Parallel Combined With Parent - 부모 오퍼레이션과 동일 PX 서버에서 실행 |
| **PCWC** | Parallel Combined With Child - 자식 오퍼레이션과 동일 PX 서버에서 실행 |

### 실전 분석 기술

1. **P->P 패턴 찾기**: 실행 계획에서 P->P 라인을 먼저 찾아 재분배 발생 지점을 식별합니다.
2. **PX SEND 타입 분석**: 각 P->P 라인의 PX SEND 타입을 확인하여 재분배 방식을 파악합니다.
3. **PCWP/PCWC 확인**: PCWP/PCWC가 많을수록 파이프라인이 잘 최적화되어 메시지 오버헤드가 감소한 상태입니다.

---

## 데이터 재분배(Distribution): PX SEND/RECEIVE의 작동 원리

### 재분배의 필요성

병렬 실행에서 최적의 성능을 달성하려면 작업이 모든 PX 서버에 균등하게 분배되어야 합니다. 그러나 조인, 집계, 정렬 같은 연산은 특정 키 값으로 데이터를 그룹화하거나 전역적으로 정렬해야 합니다. 이를 위해 Oracle은 중간에 데이터를 재분배합니다.

### 재분배 방식의 종류와 특징

| PX SEND 타입 | 사용 시기 | 효과 | 실행 계획 표기 |
|--------------|-----------|------|----------------|
| **HASH** | 해시 조인, 해시 GROUP BY | 동일 키 값을 같은 PX 서버로 모음 | `PX SEND HASH` |
| **BROADCAST** | 작은 차원 테이블과 큰 사실 테이블 조인 | 작은 테이블을 모든 PX 서버에 복제 | `PX SEND BROADCAST` |
| **RANGE** | ORDER BY, Sort-Merge 조인 | 정렬 키 범위별로 분할 | `PX SEND RANGE` |
| **ROUND-ROBIN** | 균등 분배 필요 시 | 순환 방식으로 균등 분배 | `PX SEND ROUND ROBIN` |
| **PARTITION** | 파티션 간 조인 최적화 | 파티션 단위로 분배 최소화 | `PX SEND PARTITION` |

### 재분배 방식별 실습 예제

#### 해시-해시 분배 (대규모 테이블 간 조인)
```sql
EXPLAIN PLAN FOR
SELECT /*+ 
    LEADING(f) USE_HASH(c)
    PARALLEL(f 16) PARALLEL(c 16)
    PQ_DISTRIBUTE(f HASH HASH)
    PQ_DISTRIBUTE(c HASH HASH)
    MONITOR */
    COUNT(*)
FROM fact_sales f
JOIN dim_customer c ON c.cust_id = f.cust_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(
    NULL, NULL, 'BASIC +PARALLEL +ALIAS +NOTE'
));
```

**분석 포인트:**
- PX SEND HASH와 PX RECEIVE 확인
- IN-OUT = P->P 라인 확인
- 키 값 편향(skew)이 있을 경우 특정 PX 서버에 부하 집중 가능

#### 브로드캐스트 분배 (소규모 차원 테이블 조인)
```sql
EXPLAIN PLAN FOR
SELECT /*+ 
    LEADING(c) USE_HASH(f)
    PARALLEL(f 16) PARALLEL(c 16)
    PQ_DISTRIBUTE(c BROADCAST NONE)
    MONITOR */
    SUM(f.amount)
FROM dim_customer c
JOIN fact_sales f ON f.cust_id = c.cust_id;
```

**분석 포인트:**
- 차원 테이블(c)에 PX SEND BROADCAST 확인
- 사실 테이블(f)의 재분배 최소화
- PX Deq Credit: send blkd 이벤트 완화 효과 기대

### 스큐(Skew) 감지: V$PQ_TQSTAT 활용

```sql
-- TQ 통계 분석 쿼리
SELECT 
    dfo_number,
    tq_id,
    server_type,
    process,
    num_rows,
    bytes,
    ROUND(100.0 * num_rows / SUM(num_rows) OVER (
        PARTITION BY dfo_number, tq_id, server_type
    ), 2) as percentage
FROM v$pq_tqstat
ORDER BY dfo_number, tq_id, server_type, num_rows DESC;

-- 스큐 정도 분석
SELECT 
    dfo_number,
    tq_id,
    server_type,
    MIN(num_rows) as min_rows,
    MAX(num_rows) as max_rows,
    ROUND(100.0 * (MAX(num_rows) - MIN(num_rows)) / 
        NULLIF(MAX(num_rows), 0), 2) as skew_percentage
FROM v$pq_tqstat
GROUP BY dfo_number, tq_id, server_type
HAVING MAX(num_rows) > 0
ORDER BY skew_percentage DESC;
```

**스큐 판단 기준:**
- skew_percentage가 30% 이상: 경계 수준의 스큐
- skew_percentage가 50% 이상: 심각한 스큐
- 특정 프로세스의 percentage가 70% 이상: 해당 프로세스에 작업 집중

---

## Granule: 병렬 작업의 기본 분할 단위

### Granule의 종류와 특징

**BRG(Block Range Granule)**
- 데이터 블록 범위 단위로 작업 분할
- 실행 계획에서 `PX BLOCK ITERATOR`로 표시
- 장점: 동적 로드밸런싱 우수, Granule 수가 많아 유연한 분배 가능

```sql
-- BRG 예제
EXPLAIN PLAN FOR
SELECT /*+ PARALLEL(f 8) */ SUM(amount)
FROM fact_sales f;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(
    NULL, NULL, 'BASIC +PARALLEL'
));
```

**PG(Partition Granule)**
- 파티션 단위로 작업 분할
- 실행 계획에서 `PX PARTITION RANGE ITERATOR` 등으로 표시
- 장점: 파티션 프루닝과 연계 효율적, 재분배 최소화

```sql
-- PG 예제
EXPLAIN PLAN FOR
SELECT /*+ PARALLEL(f 8) */ SUM(amount)
FROM fact_sales f
WHERE f.sales_dt >= DATE '2025-02-01'
  AND f.sales_dt < DATE '2025-03-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(
    NULL, NULL, 'BASIC +PARALLEL +PARTITION'
));
```

**SPG(Subpartition Granule)**
- 서브파티션 단위로 작업 분할
- 대규모 파티션의 작업 분배 문제 해결

### Granule 선택 전략

1. **균형 부하를 위한 BRG**: 동적 로드밸런싱이 중요한 경우
2. **지역성 최적화를 위한 PG**: 파티션 단위 작업이 효율적인 경우
3. **대규모 파티션 처리를 위한 SPG**: 단일 파티션이 너무 큰 경우

---

## 병렬 대기 이벤트: 성능 병목 분석

### 메시징/큐 관련 대기 이벤트

**PX Deq Credit: send blkd**
- Producer PX 서버가 Consumer PX 서버로 데이터 전송 시 대기
- 원인: Consumer 처리 지연, 큐 크레딧 부족
- 해결: Consumer 작업 최적화, BROADCAST 분배 전환, PGA 크기 조정

**PX Deq: Execute Reply**
- QC가 PX 서버 응답 대기
- 원인: PX 서버 작업 지연
- 해결: PX 서버 작업 최적화, 스큐 해소

### Direct Path I/O 관련 대기 이벤트

**direct path write temp**
- PGA 메모리 초과로 TEMP 테이블스페이스에 쓰기
- 원인: 과도한 정렬/해시 작업, PGA 부족
- 해결: 작업량 감소, PGA 크기 증가, DOP 조정

**direct path read temp**
- TEMP 테이블스페이스에서 읽기
- 원인: 이전에 TEMP에 기록된 데이터 재사용
- 해결: 작업량 감소, PGA 크기 증가

### 대기 이벤트 분석 쿼리

```sql
-- 활성 세션의 병렬 대기 이벤트 분석
SELECT 
    session_id,
    session_serial#,
    event,
    wait_time_micro,
    time_waited_micro,
    p1,
    p2,
    p3
FROM v$active_session_history
WHERE event LIKE 'PX%' 
   OR event LIKE 'direct path%temp'
ORDER BY sample_time DESC;

-- 병렬 대기 이벤트 통계
SELECT 
    event,
    total_waits,
    time_waited_micro,
    average_wait_micro,
    ROUND(100.0 * time_waited_micro / 
        SUM(time_waited_micro) OVER (), 2) as percentage
FROM v$system_event
WHERE event LIKE 'PX%' 
   OR event LIKE 'direct path%temp'
ORDER BY time_waited_micro DESC;
```

---

## 종합 실습: 스큐 문제 진단과 해결

### 문제 시나리오: 고객 ID 편향 데이터

```sql
-- 편향 데이터 생성
INSERT /*+ APPEND */ INTO fact_sales
SELECT 
    ROWNUM as sales_id,
    DATE '2025-02-10' + MOD(ROWNUM, 30) as sales_dt,
    CASE 
        WHEN MOD(ROWNUM, 10) < 8 THEN 202  -- 80%가 고객 202
        ELSE 101 + MOD(ROWNUM, 100)        -- 나머지 20%는 다양한 고객
    END as cust_id,
    ROUND(DBMS_RANDOM.VALUE(1, 1000), 2) as amount
FROM dual
CONNECT BY LEVEL <= 5000000;

COMMIT;
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'FACT_SALES');
```

### 해시 분배의 스큐 문제 분석

```sql
-- 스큐 발생 쿼리 실행
SELECT /*+ 
    LEADING(f) USE_HASH(c)
    PARALLEL(f 16) PARALLEL(c 16)
    PQ_DISTRIBUTE(f HASH HASH)
    MONITOR */
    c.cust_id,
    COUNT(*) as transaction_count,
    SUM(f.amount) as total_amount
FROM fact_sales f
JOIN dim_customer c ON c.cust_id = f.cust_id
GROUP BY c.cust_id;

-- 실행 계획 분석
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
    NULL, NULL, 'BASIC +PARALLEL +ALIAS +NOTE +MEMSTATS'
));

-- TQ 스큐 분석
SELECT 
    dfo_number,
    tq_id,
    server_type,
    MIN(num_rows) as min_rows,
    MAX(num_rows) as max_rows,
    ROUND(100.0 * (MAX(num_rows) - MIN(num_rows)) / 
        NULLIF(MAX(num_rows), 0), 2) as skew_percentage
FROM v$pq_tqstat
GROUP BY dfo_number, tq_id, server_type
ORDER BY skew_percentage DESC;
```

### 스큐 해결 방안 1: 브로드캐스트 분배

```sql
-- 브로드캐스트 분배 적용
SELECT /*+ 
    LEADING(c) USE_HASH(f)
    PARALLEL(f 16) PARALLEL(c 16)
    PQ_DISTRIBUTE(c BROADCAST NONE)
    MONITOR */
    c.cust_id,
    COUNT(*) as transaction_count,
    SUM(f.amount) as total_amount
FROM dim_customer c
JOIN fact_sales f ON f.cust_id = c.cust_id
GROUP BY c.cust_id;
```

### 스큐 해결 방안 2: SALT 기법 적용

```sql
-- SALT 기법을 통한 분산
WITH salted_data AS (
    SELECT /*+ PARALLEL(f 16) */
        cust_id,
        amount,
        MOD(ORA_HASH(cust_id), 16) as salt  -- DOP와 동일한 salt 수
    FROM fact_sales f
)
SELECT 
    cust_id,
    COUNT(*) as transaction_count,
    SUM(amount) as total_amount
FROM salted_data
GROUP BY cust_id
ORDER BY total_amount DESC;
```

### 스큐 해결 방안 3: 파티션 그룹핑

```sql
-- 파티션 단위 집계 후 통합
SELECT 
    cust_id,
    SUM(transaction_count) as total_transactions,
    SUM(total_amount) as grand_total
FROM (
    SELECT /*+ PARALLEL(f 8) */
        cust_id,
        COUNT(*) as transaction_count,
        SUM(amount) as total_amount
    FROM fact_sales f
    GROUP BY cust_id, 
             TRUNC(sales_dt, 'MM')  -- 월별 파티션 키
)
GROUP BY cust_id
ORDER BY grand_total DESC;
```

---

## 병렬 실행 최적화를 위한 단계별 접근법

### 1. 사전 계획 단계: 기반 구조 설계
병렬 실행의 성공은 적절한 사전 계획에 달려 있습니다. 이 단계에서는 시스템 자원과 데이터 구조를 분석하여 병렬 실행의 기반을 마련합니다.

**적절한 DOP(병렬도) 결정**
- CPU 코어 수와 동시 처리 능력을 고려하여 최적의 병렬도를 산정합니다.
- I/O 하위시스템의 처리 용량과 대역폭을 평가합니다.
- 사용 가능한 메모리 자원, 특히 PGA 메모리 제약을 고려합니다.
- 시스템의 전체적인 부하 패턴과 다른 작업들과의 조화를 고려합니다.

**테이블 파티셔닝 전략 검토**
- Partition-wise 조인 가능성을 평가하고, 필요한 경우 파티셔닝 방식을 조정합니다.
- 데이터 접근 패턴을 분석하여 가장 효율적인 파티셔닝 키를 선정합니다.
- 파티션 크기의 균형을 확인하고 필요시 재조정합니다.

**인덱스 설계 최적화**
- 병렬 스캔에 적합한 인덱스 구성을 설계합니다.
- 인덱스의 물리적 배치와 크기가 병렬 접근에 최적화되었는지 확인합니다.
- 필요시 파티션 인덱스와 로컬 인덱스 전략을 검토합니다.

### 2. 실행 계획 분석 단계: 흐름 이해
실행 계획을 심층 분석하여 병렬 실행의 실제 흐름을 이해합니다.

**IN-OUT 패턴 분석**
- P->P 발생 지점을 식별하고 그 빈도를 분석하여 재분배 비용을 평가합니다.
- 데이터 흐름의 전체적인 패턴을 이해하고 불필요한 이동을 식별합니다.
- 각 오퍼레이션의 입력/출력 관계를 명확히 합니다.

**PX SEND 타입 평가**
- 사용된 분배 방식(HASH, BROADCAST, RANGE 등)의 적절성을 평가합니다.
- 각 분배 방식이 데이터 특성과 작업 요구사항에 부합하는지 확인합니다.
- 더 효율적인 분배 방식으로의 변경 가능성을 검토합니다.

**Granule 타입 분석**
- BRG(Block Range Granule)와 PG(Partition Granule)의 선택 적절성을 평가합니다.
- Granule 크기와 개수가 병렬 서버 간 작업 분배에 최적화되었는지 확인합니다.
- 데이터 접근 패턴에 맞는 Granule 전략을 수립합니다.

**파이프라인 최적화 정도 평가**
- PCWP/PCWC 비율을 분석하여 파이프라인 최적화 정도를 파악합니다.
- 오퍼레이션 간의 결합 정도를 평가하고 개선 가능성을 탐색합니다.
- 불필요한 컨텍스트 전환과 데이터 이동을 식별합니다.

### 3. 성능 모니터링 단계: 실시간 진단
실제 실행 중인 병렬 작업의 성능을 모니터링하고 문제점을 진단합니다.

**스큐 정도 분석**
- V$PQ_TQSTAT를 활용하여 각 병렬 서버의 작업량 편차를 수치화합니다.
- 스큐 발생 원인을 분석하고 데이터 분포의 불균형을 식별합니다.
- 시간 경과에 따른 스큐 패턴을 모니터링합니다.

**병렬 대기 이벤트 모니터링**
- `PX Deq Credit: send blkd` 같은 메시징 관련 대기 이벤트를 주시합니다.
- `direct path read/write temp` I/O 대기 이벤트를 모니터링합니다.
- 대기 이벤트의 패턴과 원인을 연관 지어 분석합니다.

**메모리 사용 모니터링**
- PGA 메모리 사용량을 모니터링하고 TEMP 스필 발생 여부를 확인합니다.
- Workarea의 optimal, one-pass, multi-pass 실행 비율을 분석합니다.
- 메모리 압력과 관련된 성능 저하 지표를 식별합니다.

### 4. 튜닝 실행 단계: 체계적 개선
분석 결과를 바탕으로 체계적인 튜닝을 수행합니다.

**스큐 문제 해소**
- BROADCAST 분배 방식으로 전환하여 소규모 차원 테이블의 스큐를 해소합니다.
- SALT 기법을 적용하여 편향된 키 값을 균등하게 분산시킵니다.
- 데이터 파티셔닝을 재조정하여 자연스러운 작업 분배를 유도합니다.

**재분배 최소화**
- Partition-wise 조인을 구현하여 파티션 경계 내에서 작업을 완료합니다.
- 불필요한 데이터 이동을 식별하고 제거합니다.
- 조인 순서와 방식을 최적화하여 재분배 비용을 최소화합니다.

**메모리 최적화**
- PGA 크기를 조정하여 optimal 실행 비율을 높입니다.
- Workarea 크기 정책을 검토하고 필요시 조정합니다.
- 메모리 사용 패턴을 분석하고 효율적인 할당 전략을 수립합니다.

**DOP 재조정**
- 실제 리소스 사용률을 기반으로 DOP를 재조정합니다.
- 시스템의 전반적인 부하와 다른 작업과의 조화를 고려합니다.
- 성능 모니터링 결과를 바탕으로 동적 조정 가능성을 검토합니다.

---

## 종합 결론: 병렬 실행의 예술적 균형

Oracle 병렬 실행의 효과적인 최적화는 단순한 기술적 조작을 넘어 시스템의 다양한 요소들 사이에서 적절한 균형을 찾는 예술적 과정입니다.

### 핵심 통찰 1: 데이터 흐름의 가시화
IN-OUT 오퍼레이션 분석은 단순한 기술적 작업이 아니라, 데이터가 시스템을 통해 어떻게 흐르는지에 대한 깊은 이해를 제공합니다. P->P 패턴을 식별하고 각 재분배 지점의 PX SEND 타입을 분석함으로써, 우리는 데이터 이동의 비용과 이점을 정량적으로 평가할 수 있습니다.

### 핵심 통찰 2: 분배의 과학
데이터 재분배는 병렬 실행의 핵심이자 가장 비용이 큰 작업 중 하나입니다. HASH, BROADCAST, RANGE 등 다양한 분배 방식은 각기 다른 장단점을 가지며, 데이터 특성과 비즈니스 요구사항에 맞게 선택되어야 합니다. 특히 스큐 문제는 단순히 기술적 문제가 아니라 데이터의 비즈니스적 특성이 반영된 현상임을 이해해야 합니다.

### 핵심 통찰 3: 작업 분할의 전략
Granule 선택은 병렬 실행의 효율성을 결정하는 미시적이면서도 중요한 결정입니다. BRG의 동적 로드밸런싱, PG의 지역성 최적화, SPG의 대규모 파티션 처리 각각은 특정 상황에서 최적의 선택이 될 수 있습니다. 이 선택은 데이터의 물리적 구조와 쿼리의 논리적 요구사항 사이의 조화를 이루어야 합니다.

### 핵심 통찰 4: 대기의 언어 해석
병렬 대기 이벤트는 시스템이 우리에게 보내는 메시지입니다. `PX Deq Credit: send blkd`는 Consumer의 처리 지연을, `direct path write temp`는 메모리 부족을 이야기합니다. 이러한 대기 이벤트를 올바르게 해석하고 근본 원인을 찾아 해결하는 것이 진정한 전문가의 역량입니다.

### 최종 원칙: 상황에 맞는 최적화
모든 최적화 결정은 특정 상황에서의 트레이드오프(trade-off)입니다. 더 많은 병렬화는 더 많은 통신 오버헤드를, 더 적은 재분배는 더 많은 스큐 위험을 의미할 수 있습니다. 가장 효과적인 접근법은 시스템의 실제 동작을 측정하고, 데이터의 고유한 특성을 이해하며, 비즈니스 요구사항을 충족하는 범위 내에서 점진적으로 개선하는 것입니다.

병렬 실행 최적화의 여정은 기술적 숙련도와 직관적 이해력이 조화를 이루는 과정입니다. 이 가이드가 제시한 프레임워크와 도구들을 활용하여, 각자의 환경에서 가장 적합한 병렬 실행 전략을 개발하고 발전시켜 나가시기 바랍니다. 기억하세요, 진정한 최적화는 단순히 쿼리를 빠르게 만드는 것이 아니라, 시스템 전체의 건강과 지속 가능성을 보장하는 것입니다.