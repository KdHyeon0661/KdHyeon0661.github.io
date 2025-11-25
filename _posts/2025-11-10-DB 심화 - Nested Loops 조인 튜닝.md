---
layout: post
title: DB 심화 - Nested Loops 조인 튜닝, 테이플 prefetch, 배치 i/o, 버퍼 pinning
date: 2025-11-10 16:25:23 +0900
category: DB 심화
---
# Nested Loops 조인 튜닝 실습 — Table Prefetch, NLJ Batching, Buffer Pinning

## 0. 사전 지식 정리

본격적인 튜닝 실습에 들어가기 전에, 필요한 개념을 짧게 정리해 둔다.

### 0.1 Nested Loops 조인 요약

- Outer에서 **행을 하나씩 꺼내고**, 그 값을 이용해 Inner를 **인덱스 Lookup** 하는 방식.
- 비용을 아주 단순화하면:

  $$
  \text{총비용} \approx \text{Outer Rows} \times (\text{Inner Lookup 비용})
  $$

- Outer가 **작고**, Inner에 **좋은 인덱스**가 있을 때 특히 강력하다.
- Top-N, 페이징, 부분범위 처리(첫 페이지/앞쪽 일부만 빨리 보고 싶은 화면)에 최적.

이 글은 이미 별도의 “Nested Loops 기본편”을 충분히 다루고 있다고 가정하고,  
그 위에 얹어서 **튜닝 관점(특히 I/O 패턴 최적화)** 에 초점을 둔다.

### 0.2 Oracle Buffer Cache, Block, Pinning 감각

실습을 이해하기 위해 최소한 다음은 알고 있다고 가정한다.

- **Block**: Oracle가 디스크/버퍼 캐시를 읽고 쓰는 단위(보통 8KB).  
- **Buffer Cache**: 디스크 블록을 메모리에 캐시하는 영역.
- **Latch/Mutex & Pinning**  
  - Buffer 헤더에 대한 **락(래치/뮤텍스)** 를 잡고  
  - 그 블록을 읽는 동안 **Pin(고정)** 해 둔다.  
- 같은 블록에서 여러 행을 연속으로 읽으면:
  - 한 번 Pin한 상태로 **여러 행 처리** → latch/pin 오버헤드가 줄어든다.
- 반대로 랜덤 I/O로 여러 블록을 이리저리 왔다 갔다 하면:
  - 계속 다른 블록을 Pin/Unpin 해야 해서 CPU/래치 비용이 증가한다.

**Table Prefetch / ROWID Batched Table Access / NLJ Batching** 의 본질은  
결국 **“같은 블록을 한 번 읽었을 때, 그 블록에서 최대한 많은 행을 처리하게 만들어라”** 에 있다.

---

## 1. 공통 세션 설정

먼저 세션 레벨에서 실습 환경을 고정한다.

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD HH24:MI:SS';
ALTER SESSION SET statistics_level = ALL;                 -- DBMS_XPLAN ALLSTATS LAST
ALTER SESSION SET optimizer_features_enable = '19.1.0';   -- (환경에 맞게 조정: 19c/21c 등)
```

- `statistics_level = ALL`  
  - `DBMS_XPLAN.DISPLAY_CURSOR(... 'ALLSTATS LAST +PREDICATE +IOSTATS +MEMSTATS')` 로  
    실제 수행 통계를 확인하기 위한 설정이다.
- `optimizer_features_enable`  
  - 옵티마이저 동작을 Oracle 19c 기준으로 고정.  
  - 실습 재현성 향상(다른 세션/DB에서도 비슷한 플랜이 나오게).

개발 DB에서는 여기에 `timed_statistics = TRUE`, `event 10046` 등을 추가해  
Trace를 더 깊게 보는 것도 가능하지만, 이 글에서는 **DBMS_XPLAN 기반**으로 설명한다.

---

## 2. 실습 데이터 모델 만들기

> 포인트: **Outer(작게) → Inner(인덱스 Lookup)** 패턴을 갖는 전형적인 NL 후보 쿼리 구성  
> - Outer: 주문 헤더(고객별 최근 주문 Top-N)  
> - Inner: 주문 상세(아이템)  
> - 추가로 상품 테이블을 붙여도 되지만, 여기서는 NL 튜닝 핵심을 위해 `t_item`까지 중심으로 본다.

### 2.1 테이블 정의

```sql
-- 고객
DROP TABLE t_cust PURGE;
CREATE TABLE t_cust (
  cust_id   NUMBER       PRIMARY KEY,
  region    VARCHAR2(6)  NOT NULL
);

-- 주문 헤더 (Outer로 자주 사용)
DROP TABLE t_ord PURGE;
CREATE TABLE t_ord (
  order_id  NUMBER       PRIMARY KEY,
  cust_id   NUMBER       NOT NULL,
  order_dt  DATE         NOT NULL,
  status    VARCHAR2(8)  NOT NULL
);

-- 주문 상세 (Inner: NL의 Lookup 대상)
DROP TABLE t_item PURGE;
CREATE TABLE t_item (
  order_id  NUMBER       NOT NULL,
  item_no   NUMBER       NOT NULL,
  prod_id   NUMBER       NOT NULL,
  qty       NUMBER       NOT NULL,
  amount    NUMBER(10,2) NOT NULL,
  CONSTRAINT pk_item PRIMARY KEY(order_id, item_no)
);

-- 제품(참조)
DROP TABLE t_prod PURGE;
CREATE TABLE t_prod (
  prod_id   NUMBER       PRIMARY KEY,
  category  VARCHAR2(10) NOT NULL,
  price     NUMBER(10,2) NOT NULL
);
```

- `t_ord`: 고객별 주문.  
- `t_item`: 한 주문당 1~2개 정도의 아이템을 갖도록 구성.  
- `t_prod`: 상품 카테고리/가격용.

### 2.2 데이터 로딩 (분포 설계)

```sql
BEGIN
  -- 고객 5만 명
  FOR c IN 1..50000 LOOP
    INSERT INTO t_cust
    VALUES (
      c,
      CASE MOD(c,3)
        WHEN 0 THEN 'APAC'
        WHEN 1 THEN 'EMEA'
        ELSE 'AMER'
      END
    );
  END LOOP;

  -- 주문 30만 건
  FOR o IN 1..300000 LOOP
    INSERT INTO t_ord
    VALUES (
      o,
      MOD(o,50000)+1,                                     -- 고객 분포(고객당 여러 주문)
      DATE '2024-01-01' + MOD(o, 540),                    -- 약 1년 반 범위 날짜
      CASE MOD(o,4)
        WHEN 0 THEN 'NEW'
        WHEN 1 THEN 'PAID'
        WHEN 2 THEN 'SHIP'
        ELSE 'DONE'
      END
    );
  END LOOP;

  -- 상품 2만 개
  FOR p IN 1..20000 LOOP
    INSERT INTO t_prod
    VALUES (
      p,
      CASE MOD(p,5)
        WHEN 0 THEN 'ELEC'
        WHEN 1 THEN 'FASH'
        WHEN 2 THEN 'FOOD'
        WHEN 3 THEN 'HOME'
        ELSE 'TOY'
      END,
      ROUND(DBMS_RANDOM.VALUE(5, 1000), 2)
    );
  END LOOP;

  -- 주문 아이템 30만+ 건
  FOR x IN 1..300000 LOOP
    INSERT INTO t_item
    VALUES (
      x,
      1,
      MOD(x,20000)+1,
      TRUNC(DBMS_RANDOM.VALUE(1,5)),
      ROUND(DBMS_RANDOM.VALUE(10, 2000), 2)
    );

    IF MOD(x,3) = 0 THEN
      INSERT INTO t_item
      VALUES (
        x,
        2,
        MOD(x+7,20000)+1,
        TRUNC(DBMS_RANDOM.VALUE(1,5)),
        ROUND(DBMS_RANDOM.VALUE(10, 2000), 2)
      );
    END IF;
  END LOOP;

  COMMIT;
END;
/
```

데이터 분포는 대략 다음과 같다.

- `t_cust`: 50,000 rows
- `t_ord`: 300,000 rows (고객당 평균 6건)
- `t_item`: 300,000 ~ 600,000 rows 수준 (주문당 1~2개)
- `t_prod`: 20,000 rows

분포 확인용 쿼리 예시:

```sql
SELECT COUNT(*) FROM t_cust;
SELECT COUNT(*) FROM t_ord;
SELECT COUNT(*) FROM t_item;
SELECT COUNT(*) FROM t_prod;

SELECT status, COUNT(*) FROM t_ord GROUP BY status;
SELECT region, COUNT(*) FROM t_cust GROUP BY region;
```

---

## 3. 인덱스 및 통계

### 3.1 인덱스 설계

```sql
-- Outer 후보: cust_id + 최신 정렬(Stopkey 최적화)
CREATE INDEX ix_ord_c_dt
ON t_ord(cust_id, order_dt DESC, order_id DESC);

-- Inner Lookup: (order_id, item_no) PK 이미 존재 (pk_item)
-- 제품 PK는 이미 존재
-- 카테고리 탐색용 인덱스
CREATE INDEX ix_prod_cat
ON t_prod(category, price);
```

의도:

- `ix_ord_c_dt`:  
  - `WHERE o.cust_id = :cid` + `ORDER BY order_dt DESC, order_id DESC` 패턴을 **인덱스 하나로 처리**  
  - 정렬 흡수 + Stopkey 최적화.
- `pk_item(order_id, item_no)`  
  - 주문별 품목 lookup에 최적.
- `ix_prod_cat`  
  - 카테고리/가격 조건이 있을 때 활용.

### 3.2 통계 수집

```sql
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'T_CUST',
    cascade    => TRUE,
    method_opt => 'for all columns size skewonly'
  );
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'T_ORD',
    cascade    => TRUE,
    method_opt => 'for all columns size skewonly'
  );
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'T_ITEM',
    cascade    => TRUE,
    method_opt => 'for all columns size skewonly'
  );
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'T_PROD',
    cascade    => TRUE,
    method_opt => 'for all columns size skewonly'
  );
END;
/
```

- `cascade => TRUE`: 인덱스까지 같이 분석.
- `size skewonly`: 분포가 치우친 컬럼에만 히스토그램 적용.

---

## 4. 베이스라인: 전형적인 NL 조인 쿼리

> 상황:  
> - “특정 고객의 **최근 50건 주문**과 각 주문의 아이템을 조회하는 화면”  
> - 실제 현업에서 아주 흔한 패턴:  
>   - 고객/계좌/사용자 id로 선행 필터  
>   - 최근 날짜 순 정렬  
>   - Top-N + 페이징

### 4.1 베이스라인 쿼리

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

SELECT *
FROM   TABLE(
         DBMS_XPLAN.DISPLAY_CURSOR(
           NULL, NULL,
           'ALLSTATS LAST +PREDICATE +IOSTATS +MEMSTATS'
         )
       );
```

기대 포인트:

- Outer: `t_ord`  
  - `cust_id = :cid`로 필터  
  - `ix_ord_c_dt` 인덱스를 사용하면 **정렬 흡수 + Stopkey** 가능.
- Inner: `t_item`  
  - `pk_item` 인덱스로 `order_id`별 빠른 lookup.
- 실행계획에서:
  - 조인 메소드: **NESTED LOOPS**  
  - Inner 쪽에 `TABLE ACCESS BY INDEX ROWID` 또는 `TABLE ACCESS BY INDEX ROWID BATCHED` 가 보일 수 있다.

---

## 5. Table Prefetch(ROWID Batched Table Access) 개념

### 5.1 기본 NL 패턴

NL Inner에서 보통 아래 순서를 따른다.

1. 인덱스에서 조인 조건(`order_id`)에 맞는 ROWID들을 가져온다.
2. 각 ROWID가 가리키는 **테이블 블록**을 읽어 실제 행을 가져온다.

전통적인 NL에서는 Outer 행마다:

- (1) 인덱스 Seek → (2) 테이블 블록 0~N개 방문

을 **반복**한다. 이때 ROWID가 가리키는 블록들이 서로 흩어져 있으면  
랜덤 I/O가 많이 발생한다.

### 5.2 ROWID Batched Table Access란?

**Table Prefetch / ROWID Batching** 의 핵심은 다음과 같다.

- ROWID들을 **일정량 모았다가**,  
- **블록 기준으로 묶어서** 테이블 블록을 방문한다.

이를 통해:

- 같은 블록에 속한 여러 ROWID를 **한꺼번에 처리** →  
- 랜덤 I/O 수를 줄이고,  
- **Buffer Pin**을 한 번만 잡은 뒤 여러 행을 처리.

실제 실행계획에서는 Inner 테이블 액세스 노드에  
다음과 같은 옵션이 붙는다.

- `TABLE ACCESS BY INDEX ROWID`
- `TABLE ACCESS BY INDEX ROWID BATCHED`  ← Batched 활성 상태

힌트:

- `/*+ BATCH_TABLE_ACCESS_BY_ROWID(alias) */`
- `/*+ NO_BATCH_TABLE_ACCESS_BY_ROWID(alias) */`

버전·옵티마이저 설정에 따라 자동 적용되기도 하고,  
힌트로 강제/해제할 수도 있다.

---

## 6. 실습: Table Prefetch 활성/비활성 비교

### 6.1 강제 비활성화 케이스

먼저 Batched를 끈 상태에서 실행한다.

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

SELECT *
FROM   TABLE(
         DBMS_XPLAN.DISPLAY_CURSOR(
           NULL, NULL,
           'ALLSTATS LAST +PREDICATE +IOSTATS'
         )
       );
```

확인할 점:

- Inner에 `TABLE ACCESS BY INDEX ROWID` (BATCHED 없음) 이어야 한다.
- `Buffers`, `Reads`, `Starts`, `A-Rows` 값을 기록해 둔다.
  - 특히 `t_item` 노드의 Buffers/Reads가 중요하다.

### 6.2 강제 활성화 케이스

이제 Batched를 켠 상태에서 실행한다.

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

SELECT *
FROM   TABLE(
         DBMS_XPLAN.DISPLAY_CURSOR(
           NULL, NULL,
           'ALLSTATS LAST +PREDICATE +IOSTATS'
         )
       );
```

확인 포인트:

- Inner에 `TABLE ACCESS BY INDEX ROWID BATCHED` 가 표시되는가?
- **같은 SQL**, **같은 바인드**지만:
  - `Buffers`(논리 I/O), `Reads`(물리 I/O)가 이전보다 줄었는지 비교.
- Batched가 실제로 작동했다면:
  - `t_item` 노드의 Buffers/Reads가 **눈에 띄게 감소**하는 경우가 많다.

### 6.3 결과 비교를 위한 요약 표 (예시 형식)

실제 값은 환경마다 다르니, 아래는 비교용 포맷만 예로 든다.

| 케이스 | 조인 메소드 | 테이블 액세스 옵션 | t_item Starts | t_item A-Rows | t_item Buffers | t_item Reads |
|--------|-------------|--------------------|---------------|---------------|----------------|--------------|
| A (NO_BATCH) | NESTED LOOPS | BY INDEX ROWID | 50 | 70 | 800 | 100 |
| B (BATCHED)   | NESTED LOOPS | BY INDEX ROWID BATCHED | 50 | 70 | 500 | 60 |

- A→B로 갈수록 Buffers/Reads가 감소했다면,  
  **같은 행 수를 더 적은 I/O로 처리**하게 된 것이다.

---

## 7. NLJ Batching 개념 및 실습

### 7.1 NLJ Batching이란?

Table Prefetch가 **테이블 블록 접근을 Batch로 묶는 것**이라면,  
NLJ Batching은 한 단계 더 나아가 **“Outer 키들을 묶어서 Inner Lookup 자체를 배치로 처리”** 하는 개념이다.

직관적인 그림:

1. Outer에서 조인 키(예: `order_id`)를 **여러 개 모은다**.
2. 그 키들을 한꺼번에 Inner 인덱스/테이블에 던져 **Batch Lookup** 수행.
3. 이때 **같은 블록에 속하는 ROWID들끼리 묶여** 접근하므로,
   - 인덱스 블록, 테이블 블록 모두에서 캐시 재사용이 좋아진다.
4. 결과적으로:
   - `consistent gets` 증가 속도가 느려지고,
   - `db file sequential read` 호출 수가 줄어들 수 있다.

NLJ Batching은 옵티마이저의 내부 알고리즘/파라미터에 의해 동작하며,  
실행계획에 항상 명시적으로 드러나지는 않는다.  
그러나 **결과(통계)** 로는:

- NL 조인인데도 예상보다 **I/O가 적게** 나오거나,
- Outer Rows를 늘렸을 때 I/O 증가가 **선형보다 완만**해지는 식으로 감지할 수 있다.

### 7.2 Outer Rows 확대 실습

NLJ Batching의 이득은 **재사용할 키/블록이 많을수록** 커진다.  
그래서 Stopkey를 늘려 Outer Rows를 크게 만든 케이스를 비교해 본다.

```sql
-- Stopkey를 500으로 확대 (환경에 따라 시간 주의)
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
FETCH FIRST 500 ROWS ONLY;

SELECT *
FROM   TABLE(
         DBMS_XPLAN.DISPLAY_CURSOR(
           NULL, NULL,
           'ALLSTATS LAST +PREDICATE +IOSTATS'
         )
       );
```

이때 비교 포인트:

1. `FETCH FIRST 50`일 때와 `FETCH FIRST 500`일 때의 I/O 패턴 차이
2. Batched 적용 여부에 따른 차이
3. Outer Rows 증가량 대비:
   - `t_item` Buffers/Reads 증가 비율이 **얼마나 완만한지** 확인

NLJ Batching이 잘 작동하면:

- Outer Rows를 10배 늘렸는데 I/O는 10배까지 증가하지 않는 경우를 볼 수 있다.

(실제 수치는 데이터 분포, 버전, 파라미터에 따라 달라지므로,  
자신의 환경에서 직접 비교해 보는 것이 중요하다.)

---

## 8. 왜 Batched가 더 유리한가 — Buffer Pinning 관점

### 8.1 Buffer Pinning 개요

Oracle에서 한 블록을 읽는 흐름을 단순화하면:

1. 버퍼 캐시에서 해당 블록을 찾는다.
2. 없으면 디스크에서 읽어와 버퍼에 올린다.
3. 버퍼 헤더에 대해 래치를 잡고, 블록을 Pin한다.
4. 블록의 필요한 행을 읽고, Pin을 해제한다.

동시에 여러 세션이 같은 블록을 건드리면:

- Pin 가능한 순간까지 대기해야 할 수 있고,
- 래치 경합 등으로 인해 CPU가 더 많이 소모될 수 있다.

### 8.2 Batched + Pinning의 상관관계

ROWID Batched Table Access / NLJ Batching은 다음을 노린다.

- **같은 블록에서 읽을 행들을 최대한 한 번에 처리**하도록 재배열.
- 한 블록을 Pin한 상태에서 **여러 행을 연속 처리** →  
  - Pin 횟수 ↓  
  - 래치 횟수 ↓  
  - 캐시 재사용 ↑

반대로:

- 랜덤한 ROWID들로 **한 건씩** 테이블을 튀듯이 접근하면:
  - 매번 다른 블록을 Pin/Unpin 해야 하고
  - 동일 블록이라도 여러 번 Pin/Unpin을 반복할 수 있다.

결과적으로:

- Batched는 **동일 I/O(또는 더 적은 I/O)** 로 **더 많은 행**을 처리하는 경향.
- 특히 Inner 테이블의 **클러스터링 팩터**가 어느 정도 괜찮고,
  조인 키들이 **특정 구간에 몰려 있다**면 효과가 크다.

---

## 9. NL 튜닝 레시피 — 힌트/옵션/설계 포인트 정리

### 9.1 조인 메소드/순서/경로 힌트

```sql
/*+ LEADING(o)          */  -- 조인 시작(Outer) 고정
/*+ USE_NL(i)           */  -- Inner는 Nested Loops
/*+ USE_NL(p)           */
/*+ ORDERED             */  -- FROM 순서대로 조인
/*+ INDEX(o ix_ord_c_dt)*/  -- Outer 액세스 경로(정렬 흡수 + Stopkey)
```

- `LEADING`, `ORDERED`: 조인 순서 고정.
- `USE_NL`: 해당 테이블을 Inner로 하는 NL 조인 사용 유도.
- `INDEX`: 인덱스 액세스 경로 지정 (정렬 흡수/Stopkey에 매우 중요).

### 9.2 Table Prefetch(Batched Table Access) 힌트

```sql
/*+ BATCH_TABLE_ACCESS_BY_ROWID(i)    */  -- Inner 테이블 방문을 배치/프리패치
/*+ NO_BATCH_TABLE_ACCESS_BY_ROWID(i) */  -- Batched 비활성(비교 실험용)
```

- NL 메소드는 유지하면서,  
  **Inner 테이블 방문 방식(I/O 패턴)** 만 바꿔 보는 용도로 사용.

### 9.3 정렬 흡수 / 커버링 인덱스 / SARGability

1. 정렬 흡수
   ```sql
   -- (cust_id, order_dt DESC, order_id DESC)
   -- WHERE cust_id = :cid
   -- ORDER BY order_dt DESC, order_id DESC
   ```
   - 인덱스 설계에서 **정렬 컬럼 순서와 방향**까지 맞춰 두면,
   - `ORDER BY`를 인덱스로 처리 (Sort 생략 + Stopkey 최적화).

2. 커버링 인덱스
   - Inner에서 필요한 컬럼들을 인덱스에 모두 포함하면:
     - `TABLE ACCESS BY INDEX ROWID` 자체를 없애고  
       `INDEX RANGE SCAN` 만으로 결과를 만들 수도 있다.
   - 단, 인덱스 폭·DML 비용 증가 등 트레이드오프가 있다.

3. SARGability
   - `WHERE` 절 조건에 **함수/형변환**이 걸려 있으면 인덱스를 못 쓰는 경우가 많다.
   - 필요시 **가상 컬럼 + 인덱스** 또는 **함수기반 인덱스**로 해결.

---

## 10. “나쁜 → 좋은” 비교 실습 세트

### 10.1 Batched 미적용 (비교 기준)

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
FETCH FIRST 300 ROWS ONLY;

SELECT *
FROM   TABLE(
         DBMS_XPLAN.DISPLAY_CURSOR(
           NULL, NULL,
           'ALLSTATS LAST +PREDICATE +IOSTATS'
         )
       );
```

### 10.2 Batched 적용

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
FETCH FIRST 300 ROWS ONLY;

SELECT *
FROM   TABLE(
         DBMS_XPLAN.DISPLAY_CURSOR(
           NULL, NULL,
           'ALLSTATS LAST +PREDICATE +IOSTATS'
         )
       );
```

비교 포인트:

- Inner에 `BATCHED` 옵션이 붙는지 확인.
- `t_item` 노드의:
  - `Starts` (Outer Rows 수와 비슷해야 함)
  - `A-Rows` (실제 출력 행 수 합)
  - `Buffers`, `Reads` 비교
- 동일 조건에서 **반복 실행**해 숫자를 어느 정도 안정화시킨 후 비교하는 것이 좋다.

---

## 11. Outer가 큰 경우: NL vs Hash 선택

### 11.1 대량 Outer에서의 NL 한계

아래와 같이 Outer 범위가 매우 넓은 경우를 생각해 보자.

```sql
SELECT /* BAD_NL_CANDIDATE */
       o.order_id,
       i.amount
FROM   t_ord  o
JOIN   t_item i
       ON i.order_id = o.order_id
WHERE  o.order_dt >= DATE '2024-01-01'   -- 범위가 상당히 넓고
AND    o.status IN ('PAID','SHIP');      -- 선택도도 높지 않음
```

만약:

- `o.order_dt` 조건으로 수십만 건이 선택되고,
- Inner 인덱스 구조나 클러스터링 팩터가 좋지 않다면,

NL은 Outer 행마다 Inner를 랜덤 Lookup 하는 형태가 되어  
**랜덤 I/O 폭발**을 일으키기 쉽다.

### 11.2 Hash Join으로 전환

같은 쿼리를 Hash Join으로 유도해 비교해 본다.

```sql
SELECT /*+ USE_HASH(i) FULL(o) FULL(i) */
       o.cust_id,
       SUM(i.amount)
FROM   t_ord  o
JOIN   t_item i
       ON i.order_id = o.order_id
WHERE  o.order_dt >= DATE '2024-01-01'
GROUP  BY o.cust_id;

SELECT *
FROM   TABLE(
         DBMS_XPLAN.DISPLAY_CURSOR(
           NULL, NULL,
           'ALLSTATS LAST +PREDICATE +IOSTATS +MEMSTATS'
         )
       );
```

비교 포인트:

- Hash Join에서는
  - 양쪽 테이블을 **풀스캔**하면서
  - 조인 키로 해시 버킷을 만들어 **대량 조인**을 수행.
- NL 대비:
  - Outer Rows가 매우 크다면 Hash Join이 훨씬 적은 랜덤 I/O로 처리할 수 있다.
  - 특히 GROUP BY 같은 **대량 집계** 쿼리는 Hash Join과 궁합이 좋다.

정리하면:

- Top-N/부분범위/화면 조회: **NL + Batched**
- 대량 스캔/집계/리포트: **Hash Join**(또는 Merge Join) 우선 고려

---

## 12. Buffer Pinning 효과를 체감하는 방법

Buffer Pinning 자체를 직접 카운트하는 것은 쉽지 않지만,  
다음과 같은 지표로 간접적으로 느낄 수 있다.

1. 동일 SQL, 동일 바인드 조건에서
   - **NO_BATCH** vs **BATCHED** 비교 시:
   - `Buffers`(consistent gets) 총량이 감소하는지
   - `db file sequential read`(단건 블록 읽기) 호출 수가 감소하는지
2. Outer Rows를 적당히 늘렸을 때:
   - I/O 증가 비율이 **선형보다 완만**해지는지

동시성 부하가 있는 환경이라면,  
시스템 뷰/Wait 이벤트에서:

- `buffer busy waits`
- `read by other session`

같은 이벤트가 줄어드는 흐름을 관찰할 수도 있다(단, 여러 요인이 섞이므로 주의해서 해석해야 한다).

---

## 13. 확인/진단 스니펫

실제 환경에서 튜닝할 때 자주 쓰게 될 진단 SQL들을 정리한다.

### 13.1 최근 실행계획에서 BATCHED 사용 확인

```sql
SELECT sql_id,
       child_number,
       plan_hash_value,
       operation,
       options,
       object_name
FROM   v$sql_plan
WHERE  operation LIKE 'TABLE ACCESS%'
AND    options  LIKE '%BATCHED%'
ORDER  BY last_change_time DESC
FETCH FIRST 20 ROWS ONLY;
```

- Inner 테이블에 대한 `TABLE ACCESS BY INDEX ROWID BATCHED`가  
  실제 어떤 SQL에서 쓰이고 있는지 빠르게 확인할 수 있다.

### 13.2 인덱스 품질(클러스터링 팩터) 확인

```sql
SELECT index_name,
       blevel,
       leaf_blocks,
       clustering_factor
FROM   user_indexes
WHERE  table_name IN ('T_ITEM','T_ORD')
ORDER  BY table_name, index_name;
```

- `clustering_factor`가 테이블 블록 수와 비슷하면:
  - 인덱스 순서와 테이블 물리 순서가 비교적 잘 정렬된 상태.
  - NL + 인덱스 lookup에서 랜덤 I/O가 상대적으로 적게 발생할 수 있다.
- 반대로 `clustering_factor`가  
  테이블 row 수와 비슷하게 큰 값이면:
  - 랜덤 I/O 위험이 크다.
  - Batched가 있어도 한계가 있으며, Hash Join 등의 대안이 필요할 수 있다.

---

## 14. 실무 스타일 튜닝 절차 예시

마지막으로, 실제 현장에서 이 글의 내용을 어떻게 활용할지  
하나의 시나리오로 정리해 본다.

1. 현상
   - “고객 주문 조회 화면이 느리다. 특정 고객으로 조회하면 2~3초 걸린다.”
2. 1차 진단
   - 해당 SQL의 실행계획 확인 → NL 조인  
   - `ALLSTATS LAST`로 `Starts/A-Rows/Buffers` 확인
   - Outer Rows(Top-N)가 작고, Inner는 인덱스 lookup 구조.
3. 2차 진단
   - Inner 테이블 인덱스 구조 확인 (`user_indexes`)  
   - 클러스터링 팩터와 인덱스 컬럼 구성 점검.
   - 실행계획에서 `BATCHED` 여부 확인.
4. 1차 튜닝
   - Outer 인덱스를 정렬 흡수 가능하게 조정  
     (`(cust_id, order_dt DESC, order_id DESC)` 등)
   - Inner 테이블에 필요한 인덱스 보강 (조인 키 선두 등치).
5. 2차 튜닝
   - `BATCH_TABLE_ACCESS_BY_ROWID` 힌트 적용 후  
     `Buffers/Reads` 감소 여부 확인.
   - Stopkey(Top-N) 범위를 적당히 조정해  
     NLJ Batching 이득 체감.
6. 3차 튜닝 (필요 시)
   - Outer Rows가 커지는 사용 패턴(필터 없이 전체 기간 조회 등)에 대해서는
     Hash Join을 사용하는 별도 SQL 또는 힌트 적용.
7. 검증
   - 튜닝 전/후의 응답시간, I/O 통계, CPU 사용량을 비교.
   - Regression(다른 화면/쿼리 영향) 여부 확인.
8. 문서화
   - 어떤 SQL에 어떤 힌트를 적용했고,  
     어떤 인덱스/통계를 조정했는지 기록.
   - 이후 버전 업그레이드/옵티마이저 변경 시 다시 검증할 수 있도록 남긴다.

---

## 15. 체크리스트

- [ ] 이 쿼리에 **Nested Loops가 정말 적합한가**?  
      (Outer 작음? Top-N/부분범위? OLTP 조회 패턴?)
- [ ] Outer 테이블 액세스가 **정렬 흡수 + Stopkey 최적화**에 맞춰 설계되어 있는가?  
      (`WHERE` + `ORDER BY` 컬럼/방향이 인덱스와 일치하는가?)
- [ ] Inner 테이블 인덱스 설계는 적절한가?  
      - 조인 키가 선두 컬럼인가?  
      - 커버링 인덱스로 테이블 방문을 줄일 수 있는가?  
      - 클러스터링 팩터는 수용 가능한 수준인가?
- [ ] 실행계획에서 Inner 테이블 액세스가  
      `TABLE ACCESS BY INDEX ROWID BATCHED` 인가?  
      필요 시 `BATCH_TABLE_ACCESS_BY_ROWID` 힌트를 시도했는가?
- [ ] `ALLSTATS LAST` 기준으로  
      `Starts`, `A-Rows`, `Buffers`, `Reads`를 튜닝 전/후로 비교했는가?
- [ ] Outer Rows를 늘렸을 때(예: 50 → 500)  
      NLJ Batching/Batched 효과가 나타나는지 확인했는가?
- [ ] Outer 범위가 큰 케이스에 대해서는  
      Hash Join 등 다른 조인 메소드를 실험했는가?
- [ ] 튜닝 결과가 특정 바인드 값에만 의존하지 않고,  
      다양한 값에서 안정적인지 확인했는가?

---

## 16. 요약

- **Table Prefetch(ROWID Batched Table Access)**  
  - Inner 테이블 방문 시 ROWID들을 **블록 기준으로 묶어** 처리하여  
    **랜덤 I/O를 줄이고**, **Buffer Pin 재사용 효율을 높인다**.  
  - 실행계획에 `TABLE ACCESS BY INDEX ROWID BATCHED`로 표시된다.

- **NLJ Batching**  
  - Outer에서 나온 조인 키들을 **배치로 모아**  
    Inner 인덱스/테이블 Lookup을 효율화하는 전략이다.  
  - 실행계획에 명시적으로 드러나지 않을 수 있지만,  
    I/O 통계(특히 Buffers/Reads 증가 패턴)에서 그 효과를 확인할 수 있다.

- **Buffer Pinning**  
  - 같은 블록에서 여러 행을 처리할 때 Pin/Unpin 횟수가 줄어들고,  
    래치/CPU 오버헤드가 감소한다.  
  - Batched/NLJ Batching은 **동일 블록 재사용 기회를 극대화**하기 때문에  
    Buffer Pinning 측면에서도 이득을 준다.

- 종합적으로, **작은 Outer + 정렬 흡수 인덱스 + Batched Table Access + 적절한 인덱스 설계**를 갖추면  
  **Nested Loops 조인**은 여전히 현대 Oracle 환경(19c/21c 기준)에서도  
  Top-N/부분범위/화면 조회에서 **가장 강력한 무기**가 된다.

이 문서에서 제시한 스키마/쿼리/진단 스니펫을 그대로 실행하고,  
같은 SQL을 가지고 Batched/NLJ Batching 적용 여부에 따라  
응답 시간과 I/O 패턴이 어떻게 달라지는지 직접 숫자로 확인해 보는 것이  
Nested Loops 튜닝 감각을 몸에 익히는 가장 빠른 방법이다.