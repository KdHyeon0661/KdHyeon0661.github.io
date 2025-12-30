---
layout: post
title: DB 심화 - Nested Loops 조인 튜닝, 테이플 prefetch, 배치 i/o, 버퍼 pinning
date: 2025-11-10 16:25:23 +0900
category: DB 심화
---
# Nested Loops 조인 튜닝 심화 — Table Prefetch, NLJ Batching, Buffer Pinning 실습

## 서론: Nested Loops 조인의 핵심 개념

Nested Loops(NL) 조인은 데이터베이스 최적화에서 여전히 중요한 위치를 차지하는 조인 방식입니다. 특히 OLTP 환경에서 작은 결과 집합을 빠르게 반환해야 하는 쿼리에 매우 효과적입니다. 이 문서에서는 기본적인 NL 조인 개념을 넘어, 현대 Oracle 환경에서 NL 조인의 성능을 극대화하기 위한 고급 기법들을 심층적으로 다룹니다.

NL 조인의 기본 동작 원리는 간단합니다: Outer 테이블에서 한 행씩 읽어들이고, 그 값을 이용해 Inner 테이블을 인덱스 기반으로 조회하는 방식입니다. 이 과정을 수식으로 단순화하면 다음과 같습니다:

$$
\text{총비용} \approx \text{Outer Rows} \times (\text{Inner Lookup 비용})
$$

이 공식에서 알 수 있듯이, NL 조인의 효율성은 두 가지 요소에 크게 의존합니다:
1. Outer 결과 집합의 크기
2. Inner 테이블 조회의 효율성

본 문서에서는 특히 두 번째 요소인 Inner 테이블 조회의 효율성을 높이기 위한 다양한 기법들에 초점을 맞춥니다.

## 실습 환경 설정

실습을 시작하기 전에 필요한 세션 설정을 완료합니다:

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD HH24:MI:SS';
ALTER SESSION SET statistics_level = ALL;
ALTER SESSION SET optimizer_features_enable = '19.1.0';
```

`statistics_level = ALL` 설정은 상세한 실행 통계 수집을 가능하게 하며, `DBMS_XPLAN.DISPLAY_CURSOR`를 통해 실행 계획과 함께 실제 수행 통계를 확인할 수 있게 합니다. `optimizer_features_enable`은 실습의 재현성을 높이기 위해 옵티마이저 동작을 특정 버전으로 고정합니다.

## 실습 데이터 모델 구축

효과적인 NL 조인 튜닝 실습을 위해 전형적인 주문 처리 시스템 모델을 구성합니다:

```sql
-- 고객 테이블
DROP TABLE t_cust PURGE;
CREATE TABLE t_cust (
  cust_id   NUMBER       PRIMARY KEY,
  region    VARCHAR2(6)  NOT NULL
);

-- 주문 헤더 테이블 (주로 Outer 테이블 역할)
DROP TABLE t_ord PURGE;
CREATE TABLE t_ord (
  order_id  NUMBER       PRIMARY KEY,
  cust_id   NUMBER       NOT NULL,
  order_dt  DATE         NOT NULL,
  status    VARCHAR2(8)  NOT NULL
);

-- 주문 상세 테이블 (주로 Inner 테이블 역할)
DROP TABLE t_item PURGE;
CREATE TABLE t_item (
  order_id  NUMBER       NOT NULL,
  item_no   NUMBER       NOT NULL,
  prod_id   NUMBER       NOT NULL,
  qty       NUMBER       NOT NULL,
  amount    NUMBER(10,2) NOT NULL,
  CONSTRAINT pk_item PRIMARY KEY(order_id, item_no)
);

-- 제품 테이블
DROP TABLE t_prod PURGE;
CREATE TABLE t_prod (
  prod_id   NUMBER       PRIMARY KEY,
  category  VARCHAR2(10) NOT NULL,
  price     NUMBER(10,2) NOT NULL
);
```

데이터 모델은 실제 비즈니스 시나리오를 반영합니다:
- `t_cust`: 5만 명의 고객 정보
- `t_ord`: 30만 건의 주문 정보 (고객당 평균 6건)
- `t_item`: 30만~60만 건의 주문 상세 (주문당 1-2개 품목)
- `t_prod`: 2만 개의 제품 정보

## 인덱스 전략 설계

효율적인 NL 조인을 위한 인덱스 설계는 성능에 결정적인 영향을 미칩니다:

```sql
-- 고객별 최근 주문 조회를 위한 복합 인덱스
CREATE INDEX ix_ord_c_dt ON t_ord(cust_id, order_dt DESC, order_id DESC);

-- 제품 카테고리 기반 검색을 위한 인덱스
CREATE INDEX ix_prod_cat ON t_prod(category, price);
```

`ix_ord_c_dt` 인덱스는 특히 중요한 설계입니다. 이 인덱스는:
1. `WHERE cust_id = :id` 조건을 효율적으로 처리
2. `ORDER BY order_dt DESC, order_id DESC` 정렬을 인덱스 스캔으로 흡수
3. `FETCH FIRST N ROWS`와 같은 Top-N 쿼리에서 Stopkey 최적화 가능

이러한 인덱스 설계는 실행 계획에서 정렬 연산(SORT ORDER BY)을 제거하고, 필요한 최소한의 행만 읽을 수 있게 합니다.

## 기본 NL 조인 쿼리 분석

전형적인 고객 주문 조회 시나리오를 살펴보겠습니다:

```sql
VAR cid NUMBER;
EXEC :cid := 12345;

SELECT /* BASELINE */
       o.order_id,
       o.order_dt,
       i.item_no,
       i.amount
FROM   t_ord  o
JOIN   t_item i
       ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 50 ROWS ONLY;
```

이 쿼리의 이상적인 실행 계획은 다음과 같아야 합니다:
1. `t_ord` 테이블에서 `ix_ord_c_dt` 인덱스를 사용하여 특정 고객의 주문을 최신순으로 검색
2. 50개 행을 찾으면 즉시 스캔 중단(Stopkey 최적화)
3. 각 주문에 대해 `t_item` 테이블의 PK를 사용하여 상세 정보 조회

## Table Prefetch의 이해와 적용

### Table Prefetch의 기본 개념

전통적인 NL 조인에서 Inner 테이블 접근은 각 Outer 행마다 개별적으로 발생합니다. 이는 다음과 같은 비효율을 초래할 수 있습니다:
- 동일한 테이블 블록을 반복적으로 접근
- 블록 핀(Pin) 횟수 증가
- 래치(Latch) 경합 가능성 증대

Table Prefetch(ROWID Batched Table Access)는 이러한 문제를 해결하기 위해 도입된 기법입니다. 이 기법의 핵심은 여러 ROWID를 수집하여 블록 단위로 그룹화한 후, 동일한 블록에 속한 모든 행을 한 번에 처리하는 것입니다.

### Table Prefetch 비활성화 케이스

```sql
SELECT /*+ LEADING(o) USE_NL(i)
           INDEX(o ix_ord_c_dt) INDEX(i pk_item)
           NO_BATCH_TABLE_ACCESS_BY_ROWID(i) */
       o.order_id,
       o.order_dt,
       i.item_no,
       i.amount
FROM   t_ord  o
JOIN   t_item i
       ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 50 ROWS ONLY;
```

이 쿼리에서는 `NO_BATCH_TABLE_ACCESS_BY_ROWID` 힌트를 사용하여 Table Prefetch를 명시적으로 비활성화했습니다. 실행 계획에서 Inner 테이블 접근은 `TABLE ACCESS BY INDEX ROWID`로 표시됩니다.

### Table Prefetch 활성화 케이스

```sql
SELECT /*+ LEADING(o) USE_NL(i)
           INDEX(o ix_ord_c_dt) INDEX(i pk_item)
           BATCH_TABLE_ACCESS_BY_ROWID(i) */
       o.order_id,
       o.order_dt,
       i.item_no,
       i.amount
FROM   t_ord  o
JOIN   t_item i
       ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 50 ROWS ONLY;
```

Table Prefetch를 활성화한 이 쿼리의 실행 계획에서는 Inner 테이블 접근이 `TABLE ACCESS BY INDEX ROWID BATCHED`로 표시됩니다. 이는 Oracle이 ROWID들을 배치로 처리하고 있음을 의미합니다.

### 성능 비교 분석

두 접근 방식의 성능 차이는 주로 다음과 같은 지표에서 확인할 수 있습니다:

| 측정 항목 | NO_BATCH | BATCHED | 개선 효과 |
|-----------|----------|---------|-----------|
| 버퍼 읽기(Buffers) | 높음 | 낮음 | 캐시 효율 향상 |
| 물리적 읽기(Reads) | 높음 | 낮음 | I/O 감소 |
| 실행 시간(Elapsed) | 길음 | 짧음 | 응답 시간 개선 |

Table Prefetch의 효과는 특히 다음과 같은 조건에서 두드러집니다:
1. Inner 테이블의 클러스터링 팩터가 좋을 때(같은 주문의 품목들이 물리적으로 가까이 위치)
2. 동일한 테이블 블록을 여러 번 접근해야 할 때
3. 버퍼 캐시 히트율이 중요할 때

## NLJ Batching의 심층 이해

### NLJ Batching의 동작 원리

NLJ Batching은 Table Prefetch보다 한 단계 발전된 최적화 기법입니다. Table Prefetch가 테이블 블록 접근을 최적화하는 데 집중한다면, NLJ Batching은 조인 프로세스 자체를 재구성합니다.

NLJ Batching의 동작 과정:
1. Outer 테이블에서 여러 조인 키를 수집
2. 수집된 키들을 Inner 인덱스에 배치(batch)로 질의
3. 결과 ROWID들을 블록 단위로 그룹화
4. 그룹화된 블록들을 효율적으로 접근

이 과정을 도식화하면 다음과 같습니다:

```
전통적 NL: Outer행1 → Inner조회 → Outer행2 → Inner조회 → ...
NLJ Batching: [Outer행1,행2,...] → [Inner배치조회] → [블록그룹화] → [효율적접근]
```

### NLJ Batching 효과 측정

NLJ Batching의 효과는 Outer 결과 집합의 크기에 따라 다르게 나타납니다:

```sql
-- 작은 결과 집합 (50행)
SELECT /*+ BATCH_TABLE_ACCESS_BY_ROWID(i) */
       COUNT(*)
FROM   t_ord  o
JOIN   t_item i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
FETCH FIRST 50 ROWS ONLY;

-- 큰 결과 집합 (500행)
SELECT /*+ BATCH_TABLE_ACCESS_BY_ROWID(i) */
       COUNT(*)
FROM   t_ord  o
JOIN   t_item i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
FETCH FIRST 500 ROWS ONLY;
```

NLJ Batching이 효과적으로 작동할 때 관찰되는 패턴:
- Outer 행 수가 10배 증가해도 I/O는 10배 미만으로 증가
- 버퍼 캐시 재사용률 향상
- 평균 블록 접근 비용 감소

## Buffer Pinning의 성능 영향

### Buffer Pinning 메커니즘

Oracle 데이터베이스에서 블록 접근은 다음과 같은 단계를 거칩니다:
1. 버퍼 캐시에서 대상 블록 검색
2. 블록이 없을 경우 디스크에서 읽기
3. 버퍼 헤더에 래치 획득
4. 블록 핀(Pin) 설정
5. 데이터 읽기
6. 핀 해제 및 래치 반환

핀(Pin)은 블록이 메모리에서 안전하게 접근될 수 있도록 보장하는 메커니즘입니다. 핀을 과도하게 자주 설정하고 해제하면 다음과 같은 문제가 발생할 수 있습니다:
- 래치 경합 증가
- CPU 오버헤드 증대
- 동시성 저하

### Batched 접근의 Pinning 최적화

Table Prefetch와 NLJ Batching은 Buffer Pinning 관점에서도 큰 이점을 제공합니다:

```
전통적 접근: Pin → 행1처리 → Unpin → Pin → 행2처리 → Unpin → ...
Batched 접근: Pin → 행1처리 → 행2처리 → ... → 행N처리 → Unpin
```

Batched 접근 방식은 동일한 블록에 대한 반복적인 Pin/Unpin 작업을 줄여줌으로써:
1. 래치 횟수 감소
2. CPU 사용량 절감
3. 동시 처리 능력 향상

이러한 최적화는 특히 다음과 같은 환경에서 두드러진 효과를 발휘합니다:
- 고성능 SSD 저장 장치 사용 시
- 다중 코프 프로세서 환경
- 높은 동시성 워크로드

## 실무 적용을 위한 튜닝 전략

### 효과적인 힌트 사용법

NL 조인 튜닝을 위한 주요 힌트들을 상황에 맞게 활용해야 합니다:

```sql
-- 조인 순서 및 메소드 제어
/*+ LEADING(outer_table) */      -- 조인 시작점 지정
/*+ USE_NL(inner_table) */       -- 특정 테이블을 NL Inner로 사용
/*+ ORDERED */                   -- FROM 절 순서대로 조인

-- 접근 경로 제어
/*+ INDEX(table index_name) */   -- 특정 인덱스 사용 강제
/*+ INDEX_DESC(table index_name) */ -- 인덱스 역방향 스캔

-- 배치 처리 제어
/*+ BATCH_TABLE_ACCESS_BY_ROWID(table) */    -- Table Prefetch 활성화
/*+ NO_BATCH_TABLE_ACCESS_BY_ROWID(table) */ -- Table Prefetch 비활성화
```

### 인덱스 설계 가이드라인

효율적인 NL 조인을 위한 인덱스 설계 원칙:

1. **정렬 흡수 인덱스**: WHERE 조건과 ORDER BY 절을 동시에 만족하는 인덱스 설계
2. **커버링 인덱스**: 자주 접근하는 컬럼들을 인덱스에 포함하여 테이블 접근 회피
3. **조인 조건 최적화**: 조인 컬럼이 인덱스 선두에 위치하도록 설계

### 클러스터링 팩터 관리

클러스터링 팩터는 NL 조인 성능에 중요한 영향을 미칩니다:

```sql
-- 클러스터링 팡터 확인
SELECT index_name,
       clustering_factor,
       leaf_blocks,
       ROUND(clustering_factor / leaf_blocks, 2) as cf_ratio
FROM   user_indexes
WHERE  table_name = 'T_ITEM';
```

클러스터링 팩터 해석:
- 값이 작을수록(테이블 블록 수에 가까울수록): 물리적 정렬이 좋음
- 값이 클수록(행 수에 가까울수록): 랜덤 I/O 발생 가능성 높음

클러스터링 팩터가 나쁜 경우 개선 방법:
1. 테이블 재구성(ALTER TABLE ... MOVE)
2. 인덱스 재생성(ALTER INDEX ... REBUILD)
3. 파티셔닝 적용

## NL 조인 vs Hash 조인 선택 가이드

NL 조인이 항상 최선의 선택은 아닙니다. 상황에 맞는 조인 방식을 선택해야 합니다:

### NL 조인이 적합한 경우
- 작은 결과 집합(수십~수백 행)
- 부분 범위 처리(Top-N, 페이징)
- OLTP 트랜잭션 처리
- Outer 테이블이 작고 Inner 인덱스 효율이 좋을 때

### Hash 조인이 적합한 경우
- 대량 데이터 조인
- 전체 결과 집합 처리
- 배치 처리, 리포트 생성
- 등치 조인(=)이 주를 이루는 경우
- 메모리 여유가 충분할 때

```sql
-- Hash Join 적용 예시
SELECT /*+ USE_HASH(i) */
       o.cust_id,
       SUM(i.amount) as total_amount
FROM   t_ord o
JOIN   t_item i ON i.order_id = o.order_id
WHERE  o.order_dt >= DATE '2024-01-01'
GROUP BY o.cust_id;
```

## 고급 모니터링 및 진단 기법

### 실행 계획 심층 분석

```sql
-- 상세 실행 통계 확인
SELECT *
FROM   TABLE(
         DBMS_XPLAN.DISPLAY_CURSOR(
           NULL, NULL,
           'ALLSTATS LAST +PREDICATE +IOSTATS +MEMSTATS +COST +BYTES'
         )
       );
```

주요 분석 포인트:
- `Starts`: 각 작업의 실행 횟수
- `A-Rows`: 실제 반환 행 수
- `Buffers`: 논리적 I/O 수
- `Reads`: 물리적 I/O 수
- `Cost`: 옵티마이저 예상 비용

### 시스템 성능 지표 모니터링

```sql
-- NL 관련 대기 이벤트 모니터링
SELECT event, total_waits, time_waited_micro
FROM   v$system_event
WHERE  event LIKE '%buffer%' OR event LIKE '%latch%'
ORDER BY time_waited_micro DESC;

-- 세션별 NL 조인 통계
SELECT sid, sql_id, executions, buffer_gets, disk_reads
FROM   v$sql
WHERE  sql_text LIKE '%USE_NL%'
ORDER BY buffer_gets DESC;
```

## 실무 적용 시나리오

### 시나리오 1: 고객 주문 조회 성능 개선

**문제**: 특정 고객의 최근 주문 조회가 3초 이상 소요

**진단**:
1. 실행 계획 확인: NL 조인 사용 중
2. Inner 테이블 접근이 `TABLE ACCESS BY INDEX ROWID` (Batched 없음)
3. `t_item` 테이블의 클러스터링 팩터 확인

**해결**:
```sql
-- 1. Table Prefetch 활성화
SELECT /*+ BATCH_TABLE_ACCESS_BY_ROWID(i) */
       o.order_id, o.order_dt, i.item_no, i.amount
FROM   t_ord o
JOIN   t_item i ON i.order_id = o.order_id
WHERE  o.cust_id = :cust_id
ORDER BY o.order_dt DESC
FETCH FIRST 100 ROWS ONLY;

-- 2. 인덱스 튜닝 (필요시)
CREATE INDEX ix_item_order ON t_item(order_id) COMPRESS;
```

### 시나리오 2: 대량 데이터 조회 시 Hash Join 전환

**문제**: 월별 판매 리포트 생성이 10분 이상 소요

**진단**:
1. NL 조인으로 인한 랜덤 I/O 폭발
2. Outer 결과 집합이 수만 행에 달함

**해결**:
```sql
-- Hash Join으로 전환
SELECT /*+ USE_HASH(i) FULL(o) FULL(i) */
       TO_CHAR(o.order_dt, 'YYYY-MM') as month,
       SUM(i.amount) as monthly_sales
FROM   t_ord o
JOIN   t_item i ON i.order_id = o.order_id
WHERE  o.order_dt >= ADD_MONTHS(SYSDATE, -12)
GROUP BY TO_CHAR(o.order_dt, 'YYYY-MM');
```

## 결론

Nested Loops 조인은 현대 Oracle 데이터베이스 환경에서 여전히 중요한 조인 방식입니다. 특히 OLTP 시스템에서 작은 결과 집합을 빠르게 반환해야 하는 시나리오에서는 NL 조인이 가장 효율적인 선택일 수 있습니다.

Table Prefetch와 NLJ Batching은 NL 조인의 성능을 한 단계 높여주는 핵심 기법들입니다. 이러한 기법들은 단순히 실행 계획을 변경하는 것을 넘어, 데이터베이스의 내부 메커니즘(Buffer Pinning, 래치 관리 등)을 최적화함으로써 성능 향상을 이루어냅니다.

효과적인 NL 조인 튜닝을 위해서는:
1. **상황에 맞는 조인 방식 선택**: NL 조인과 Hash 조인의 장단점을 이해하고 적절히 적용
2. **인덱스 설계 최적화**: 정렬 흡수, 커버링 인덱스 등 NL 조인에 특화된 인덱스 설계
3. **고급 최적화 기법 활용**: Table Prefetch, NLJ Batching 등의 기법을 적극 활용
4. **체계적인 모니터링**: 실행 계획, I/O 통계, 시스템 지표를 종합적으로 분석

이 문서에서 제시한 실습 예제들과 튜닝 기법들은 실제 프로덕션 환경에서 검증된 접근법들입니다. 각자의 시스템 환경에 맞게 적용하고, 지속적인 모니터링과 최적화를 통해 데이터베이스 성능을 극대화할 수 있을 것입니다.

데이터베이스 튜닝은 단순히 쿼리 속도를 높이는 기술이 아니라, 시스템 자원을 효율적으로 활용하는 철학입니다. NL 조인 최적화를 통해 더 빠른 응답 시간, 더 높은 동시성, 더 낮은 인프라 비용을 동시에 달성할 수 있습니다.