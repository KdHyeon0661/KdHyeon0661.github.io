---
layout: post
title: DB 심화 - Nested Loops 조인
date: 2025-11-10 15:25:23 +0900
category: DB 심화
---
# Nested Loops 조인 — 기본 메커니즘 → 힌트 제어 → 실행계획·통계 해석 → 특징·튜닝 포인트

## 공통 세션 설정 (실습 환경 준비)

Nested Loops 조인 실습에서 **재현 가능한 실행계획과 통계**를 얻기 위해, 먼저 세션 환경을 일관되게 맞춰둡니다.

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';
ALTER SESSION SET statistics_level = ALL;
ALTER SESSION SET optimizer_features_enable = '19.1.0';
```

- `statistics_level = ALL`  
  나중에 `DBMS_XPLAN.DISPLAY_CURSOR`로 **실행 통계**를 상세히 확인하기 위함입니다.
- `optimizer_features_enable`  
  옵티마이저 동작을 특정 버전 기준으로 고정하여 실습 재현성을 높입니다.

---

## Nested Loops 조인의 직관과 기본 메커니즘

### 직관적인 그림

Nested Loops 조인의 핵심 직관은 아래와 같습니다.

> **바깥쪽(Outer)에서 한 행씩 가져온 뒤 → 안쪽(Inner)을 그 키로 즉시 찾아본다.**

조인 과정은 다음과 같습니다.

1. Outer row-source에서 **한 행**을 꺼냅니다.
2. 이 행의 **조인 키**를 사용해 Inner row-source에 **인덱스 탐색**을 수행합니다.
3. Inner에서 매칭되는 행들을 읽어 **결과로 출력**합니다.
4. Outer의 **다음 행**으로 이동하여 위 과정을 반복합니다.
5. Outer가 끝나면 조인이 종료됩니다.

핵심은 **Outer는 “작게”**, **Inner는 “빠르게(=인덱스 lookup)”** 유지하는 것입니다.

### 단순 비용 모델

Nested Loops 조인 비용을 단순화하면 다음 공식으로 이해할 수 있습니다.

```
총비용 ≈ Outer Rows × (Inner Lookup 비용)
```

여기서 **Inner Lookup 비용**은 일반적으로 인덱스 탐색과 테이블 블록 랜덤 액세스 비용의 합입니다.

실제 성능 감각을 키우려면 실행계획에서 다음 항목을 함께 확인하는 것이 중요합니다.
- Outer row-source의 **A-Rows(실제 출력 행 수)**
- Inner row-source의 **Starts(루프 횟수)**, **Buffers(논리 I/O)**

실행계획 통계의 전형적인 패턴은 다음과 같습니다.
- Outer 테이블: `Starts = 1`, `A-Rows = n`
- Inner 테이블: `Starts = n`, `Buffers = n × (평균 lookup 비용)`

### 언제 효과적인가?

다음 조건이 충족되면 Nested Loops는 매우 강력한 조인 방식이 됩니다.

1. **Outer 결과 집합이 작다**  
   WHERE 절 필터 선택도가 높거나, 파티션 프루닝, `ROWNUM`/`FETCH FIRST`로 제한된 경우.
2. **Inner에 조인 키 선두 인덱스가 존재**  
   등치 조인 조건이 인덱스 선두 컬럼에 걸려 있어 빠른 탐색이 가능한 경우.
3. **Top-N 처리 또는 부분범위처리가 필요하다**  
   웹 화면 페이징이나 OLTP 조회처럼 처음 몇 건만 빠르게 보여줘야 하는 경우.

이러한 상황에서 Nested Loops의 장점은 **첫 번째 행을 매우 빠르게 반환**할 수 있다는 점이며, 전체 결과 집합을 완성하지 않아도 조기 응답이 가능합니다.

### 주의해야 할 상황

반대로 아래 상황에서는 Nested Loops가 심각한 성능 문제를 일으킬 수 있습니다.

1. **Outer 결과 집합이 매우 크다** (수십만, 수백만 행)
2. **Inner에 적절한 인덱스가 없다**  
   인덱스가 없거나, 클러스터링 팩터가 나쁘거나, 조인 키가 인덱스 선두가 아닌 경우.
3. **랜덤 I/O가 과도하게 발생한다**  
   Outer 행 하나당 Inner 테이블의 여러 블록을 랜덤하게 읽어야 하는 경우.

이런 경우에는 해시 조인이나 머지 조인이 더 나은 선택이 될 수 있습니다.

---

## 실습 스키마: customers / orders / order_items / products

Nested Loops 조인의 다양한 패턴을 실습하기 위한 스키마를 구성합니다.

```sql
DROP TABLE customers PURGE;
DROP TABLE orders PURGE;
DROP TABLE order_items PURGE;
DROP TABLE products PURGE;

CREATE TABLE customers (
  cust_id   NUMBER      PRIMARY KEY,
  name      VARCHAR2(40),
  region    VARCHAR2(10)
);

CREATE TABLE orders (
  order_id  NUMBER      PRIMARY KEY,
  cust_id   NUMBER      NOT NULL,
  order_dt  DATE        NOT NULL,
  status    VARCHAR2(8) NOT NULL,
  CONSTRAINT fk_orders_cust FOREIGN KEY (cust_id) REFERENCES customers(cust_id)
);

CREATE TABLE order_items (
  order_id  NUMBER      NOT NULL,
  item_no   NUMBER      NOT NULL,
  prod_id   NUMBER      NOT NULL,
  qty       NUMBER      NOT NULL,
  amount    NUMBER(10,2) NOT NULL,
  CONSTRAINT pk_oi PRIMARY KEY (order_id, item_no)
);

CREATE TABLE products (
  prod_id   NUMBER      PRIMARY KEY,
  category  VARCHAR2(12),
  price     NUMBER(10,2)
);
```

### 데이터 로딩

실습용으로 적절한 규모의 데이터를 생성합니다.

```sql
BEGIN
  -- 5만 명 고객
  FOR c IN 1..50000 LOOP
    INSERT INTO customers VALUES(
      c,
      'C' || TO_CHAR(c),
      CASE MOD(c,3)
        WHEN 0 THEN 'APAC'
        WHEN 1 THEN 'EMEA'
        ELSE 'AMER'
      END
    );
  END LOOP;

  -- 30만 건 주문
  FOR o IN 1..300000 LOOP
    INSERT INTO orders VALUES(
      o,
      MOD(o,50000) + 1,
      DATE '2024-01-01' + MOD(o,540),
      CASE MOD(o,4)
        WHEN 0 THEN 'NEW'
        WHEN 1 THEN 'PAID'
        WHEN 2 THEN 'SHIP'
        ELSE 'DONE'
      END
    );
  END LOOP;

  -- 2만 개 상품
  FOR p IN 1..20000 LOOP
    INSERT INTO products VALUES(
      p,
      CASE MOD(p,5)
        WHEN 0 THEN 'ELEC'
        WHEN 1 THEN 'FASH'
        WHEN 2 THEN 'FOOD'
        WHEN 3 THEN 'HOME'
        ELSE 'TOY'
      END,
      ROUND(DBMS_RANDOM.VALUE(5, 1000),2)
    );
  END LOOP;

  -- 30만 건 이상의 주문품목
  FOR x IN 1..300000 LOOP
    INSERT INTO order_items VALUES(
      x,
      1,
      MOD(x,20000) + 1,
      TRUNC(DBMS_RANDOM.VALUE(1,5)),
      ROUND(DBMS_RANDOM.VALUE(10, 2000),2)
    );
    IF MOD(x,3)=0 THEN
      INSERT INTO order_items VALUES(
        x,
        2,
        MOD(x+7,20000) + 1,
        TRUNC(DBMS_RANDOM.VALUE(1,5)),
        ROUND(DBMS_RANDOM.VALUE(10, 2000),2)
      );
    END IF;
  END LOOP;

  COMMIT;
END;
/
```

### 인덱스 생성

NL 조인에 적합하도록 인덱스를 설계합니다.

```sql
CREATE INDEX ix_orders_cdt      ON orders(cust_id, order_dt, order_id);
CREATE INDEX ix_orders_status   ON orders(status, order_dt);
CREATE INDEX ix_oi_order        ON order_items(order_id, item_no);
CREATE INDEX ix_oi_prod         ON order_items(prod_id);
CREATE INDEX ix_prod_category   ON products(category, price);
```

- `ix_orders_cdt`: 특정 고객의 주문을 날짜 역순으로 Top-N 조회하는 데 최적화된 인덱스입니다.
- `ix_oi_order`: 주문번호별 주문품목을 빠르게 찾기 위한 인덱스입니다.
- `ix_prod_category`: 카테고리와 가격 조건으로 상품을 검색하기 위한 인덱스입니다.

### 통계 수집

```sql
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'CUSTOMERS', cascade => TRUE, method_opt => 'for all columns size skewonly');
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'ORDERS', cascade => TRUE, method_opt => 'for all columns size skewonly');
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'ORDER_ITEMS', cascade => TRUE, method_opt => 'for all columns size skewonly');
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'PRODUCTS', cascade => TRUE, method_opt => 'for all columns size skewonly');
END;
/
```

---

## 기본 예제: 고객의 최근 주문 Top-N + 상세 (NL 조인 이상적 케이스)

Nested Loops의 강점이 발휘되는 가장 전형적인 상황을 살펴봅니다.

> "특정 고객의 **최근 주문 상위 20건**과 각 주문의 품목·상품 정보를 조회"

### 기본 쿼리

```sql
VAR cid NUMBER
EXEC :cid := 12345;

SELECT /* BASELINE */
       o.order_id,
       o.order_dt,
       i.item_no,
       i.amount,
       p.category
FROM   orders       o
JOIN   order_items  i ON i.order_id = o.order_id
JOIN   products     p ON p.prod_id  = i.prod_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 20 ROWS ONLY;
```

#### Nested Loops 적합성 분석

1. **Outer가 작다**: `WHERE o.cust_id = :cid` 조건으로 특정 고객 한 명의 주문만 대상이 됩니다.
2. **정렬 흡수 가능**: `ORDER BY` 절과 `ix_orders_cdt` 인덱스의 컬럼 순서와 방향이 일치하여 정렬 작업 없이 인덱스 스캔으로 결과를 얻을 수 있습니다.
3. **Top-N 처리**: `FETCH FIRST 20 ROWS ONLY`로 Stopkey 최적화가 동작하여 필요한 최소 블록만 읽고 조기 종료됩니다.
4. **효율적인 Inner 탐색**: 각 주문에 대해 `order_items`는 `(order_id, item_no)` 인덱스로, `products`는 PK 인덱스로 즉시 조회됩니다.

이 쿼리의 조인 패턴은 다음과 같습니다.
- Outer: `ORDERS` - 특정 고객의 최신 20건 주문
- Inner1: `ORDER_ITEMS` - 주문별 품목을 인덱스로 빠르게 조회
- Inner2: `PRODUCTS` - 품목의 상품 정보를 PK로 조회

### Nested Loops를 강제하는 힌트

옵티마이저가 다른 조인 방식을 선택할 수 있으므로, NL 조인 패턴을 명확히 관찰하려면 힌트를 활용합니다.

```sql
SELECT /*+ 
          LEADING(o)
          USE_NL(i) USE_NL(p)
          INDEX(o ix_orders_cdt)
          INDEX(i ix_oi_order)
       */
       o.order_id,
       o.order_dt,
       i.item_no,
       i.amount,
       p.category
FROM   orders       o
JOIN   order_items  i ON i.order_id = o.order_id
JOIN   products     p ON p.prod_id  = i.prod_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 20 ROWS ONLY;
```

힌트 설명:
- `LEADING(o)`: 조인 순서에서 `o`(orders)를 가장 먼저 처리합니다.
- `USE_NL(i) USE_NL(p)`: `i`와 `p`를 Inner로 하는 NL 조인을 사용합니다.
- `INDEX(o ix_orders_cdt)`: orders 테이블에 대해 `ix_orders_cdt` 인덱스를 사용하도록 지정합니다.
- `INDEX(i ix_oi_order)`: order_items 테이블에 대해 `ix_oi_order` 인덱스를 사용하도록 지정합니다.

---

## 힌트로 NL 조인 제어하기

### 조인 순서와 조인 메소드 제어

- **조인 순서 힌트**
  ```sql
  /*+ ORDERED */              -- FROM 절 순서대로 조인
  /*+ LEADING(t1 t2 t3) */   -- 명시적 조인 순서 지정
  ```

- **조인 메소드 힌트**
  ```sql
  /*+ USE_NL(t) */           -- t를 Inner로 하는 NL 조인
  /*+ NO_USE_NL(t) */        -- t에 대해 NL 조인 금지
  /*+ USE_HASH(t) */         -- t를 Inner로 하는 해시 조인
  /*+ USE_MERGE(t) */        -- t를 Inner로 하는 머지 조인
  ```

### Inner 액세스 경로 힌트

NL 조인에서 Inner 테이블의 액세스 경로는 성능에 결정적 영향을 미칩니다.

```sql
/*+ INDEX(t idx) */           -- 특정 인덱스 사용
/*+ INDEX_ASC(t idx) */       -- 인덱스 오름차순 스캔
/*+ INDEX_DESC(t idx) */      -- 인덱스 내림차순 스캔
/*+ FULL(t) */                -- 풀 테이블 스캔
/*+ USE_NL_WITH_INDEX(t idx) */ -- NL 조인 + 특정 인덱스 강제
```

`USE_NL_WITH_INDEX`는 `USE_NL + INDEX`보다 더 강력하게 NL 조인과 인덱스 사용을 유도합니다.

### 세미/안티 조인에서의 NL 힌트

`EXISTS`/`NOT EXISTS` 서브쿼리는 종종 NL 세미/안티 조인으로 실행됩니다.

```sql
/*+ USE_NL_SJ */   -- Nested Loops Semi Join
/*+ USE_NL_AJ */   -- Nested Loops Anti Join
```

예제:
```sql
SELECT /*+ LEADING(c) USE_NL_SJ(o) USE_NL_SJ(i) USE_NL_SJ(p) */
       c.cust_id
FROM   customers c
WHERE  c.cust_id = :cid
AND    EXISTS (
         SELECT /*+ USE_NL_SJ(i) USE_NL_SJ(p) */
                1
         FROM   orders o
         JOIN   order_items i ON i.order_id = o.order_id
         JOIN   products p    ON p.prod_id  = i.prod_id
         WHERE  o.cust_id = c.cust_id
         AND    p.category = 'ELEC'
       );
```

`EXISTS` 서브쿼리는 첫 번째 매칭 행을 찾으면 바로 종료되므로, NL 세미 조인과 매우 잘 어울립니다.

---

## NL 조인 수행 과정 분석 — DBMS_XPLAN으로 보는 row-source 통계

실제 실행 통계를 분석하면서 NL 조인의 동작을 깊이 이해해 보겠습니다.

### 예제 쿼리 실행

```sql
VAR cid NUMBER
EXEC :cid := 23456;

SELECT /* NL_ANALYZE */
       o.order_id,
       o.order_dt,
       i.item_no,
       i.amount
FROM   orders      o
JOIN   order_items i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
AND    o.status  = 'PAID'
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 30 ROWS ONLY;
```

### 실행계획 통계 확인

```sql
SELECT *
FROM   TABLE(
         DBMS_XPLAN.DISPLAY_CURSOR(
           NULL, NULL,
           'ALLSTATS LAST +PREDICATE +IOSTATS +MEMSTATS'
         )
       );
```

### row-source 통계 해석 포인트

실행계획 통계에서 NL 조인의 특징을 확인할 수 있는 핵심 항목들입니다.

- **Starts**: 해당 연산이 시작된 횟수입니다. Inner 테이블의 Starts 수가 Outer 행 수와 같으면 NL 조인의 전형적인 패턴입니다.
- **A-Rows**: 실제 출력된 행 수입니다. Outer가 적고 Stopkey 최적화가 작동하면 예상보다 훨씬 작을 수 있습니다.
- **Buffers**: 논리적 I/O 블록 수입니다. NL 조인의 총 비용을 평가하는 핵심 지표입니다.
- **Reads**: 물리적 I/O 블록 수입니다. 버퍼 캐시 적중률을 파악할 수 있습니다.
- **Predicate Information**: 
  - `access`: 인덱스나 파티션의 진입 조건으로, 스캔 범위를 실제로 줄이는 조건입니다.
  - `filter`: 읽은 후에 거르는 조건으로, 스캔량 자체는 줄이지 못합니다.

NL 조인의 전형적인 통계 패턴:
- Outer row-source: `Starts = 1`, `A-Rows = n` (n은 작은 수)
- Inner row-source: `Starts = n`, `Buffers = n × (평균 lookup 비용)`

### NL 조인 비용 모델링

NL 조인의 총 비용을 다음과 같이 모델링할 수 있습니다.

```
총 Buffers ≈ Outer Buffers + (Outer 행 수 × Inner 평균 lookup 비용)
```

여기서 `Outer 행 수`와 `Inner 평균 lookup 비용`이 NL 조인 튜닝의 두 가지 핵심 포인트입니다.

---

## NL 조인의 특징 — 장점과 단점

### 장점

1. **빠른 첫 행 응답**: 작은 Outer와 효율적인 Inner 인덱스가 결합되면 첫 번째 결과 행을 매우 빠르게 반환할 수 있습니다.
2. **Top-N/페이징 처리에 최적**: 부분범위처리가 자연스럽게 지원되어 웹 화면 조회에 적합합니다.
3. **정렬 흡수 가능**: ORDER BY 절을 인덱스 스캔 순서와 일치시켜 정렬 작업을 제거할 수 있습니다.
4. **인덱스만 스캔 가능**: 커버링 인덱스를 구성하면 테이블 액세스 없이 인덱스만으로 결과를 생성할 수 있습니다.
5. **선택적 필터링**: Inner 조인 시 추가 필터 조건이 있어도 인덱스 lookup으로 필요한 최소 행만 읽을 수 있습니다.

### 단점 및 주의사항

1. **Outer가 크면 성능 저하**: Outer 행 수가 많아지면 랜덤 I/O가 선형적으로 증가합니다.
2. **인덱스 품질에 민감**: Inner 인덱스의 클러스터링 팩터가 나쁘면 테이블 랜덤 액세스 비용이 급증합니다.
3. **대량 배치 처리에는 부적합**: 대규모 집계나 배치 작업에서는 해시 조인이 더 효율적일 수 있습니다.
4. **병렬 처리의 한계**: 파이프라인 구조상 병렬 처리에 제약이 있어 대용량 풀스캔 시 비효율적일 수 있습니다.

---

## NL 조인을 최적화하는 설계 포인트

### 인덱스 설계 전략

1. **Inner 테이블에 조인 키 선두 인덱스 구성**: 등치 조인 조건이 인덱스 선두 컬럼에 오도록 설계합니다.
2. **커버링 인덱스 활용**: 자주 조회되는 컬럼을 인덱스에 포함시켜 테이블 액세스를 제거합니다.
3. **정렬 흡수 인덱스 설계**: 자주 사용되는 ORDER BY 절을 인덱스 컬럼 순서와 일치시킵니다.
4. **함수기반 인덱스 고려**: `UPPER(col)`, `TO_CHAR(date)` 등의 함수 조건에 대해 인덱스를 구성합니다.

### 데이터 및 통계 관리

1. **통계 최신화**: 데이터 분포 변화에 따라 주기적으로 통계를 수집합니다.
2. **히스토그램 활용**: 치우친 데이터 분포를 가진 컬럼에 히스토그램을 생성해 카디널리티 추정 정확도를 높입니다.
3. **클러스터링 팩터 모니터링**: 인덱스와 테이블 데이터의 물리적 정렬 상태를 주기적으로 점검합니다.
4. **바인드 피킹 대응**: 바인드 변수 값에 따라 실행계획이 크게 달라질 경우, SQL 플랜 관리를 활용합니다.

### 쿼리 작성 가이드라인

1. **Outer 필터 강화**: 가능한 한 Outer 테이블 쪽에 강력한 필터 조건을 배치합니다.
2. **Top-N 구조 활용**: `ROWNUM`, `FETCH FIRST`를 적극 사용해 불필요한 처리량을 줄입니다.
3. **EXISTS 활용**: IN 서브쿼리보다 EXISTS를 사용해 불필요한 중복 행 생성을 방지합니다.
4. **쿼리 분리 고려**: 복잡한 조인을 "키 추출 → In-list 조회"의 2단계로 분리하는 것을 검토합니다.

---

## NL vs Hash vs Merge — 상황별 선택 가이드

### 선택 기준 비교

| 상황 | 추천 조인 방식 | 이유 |
|------|---------------|------|
| Outer 작음, Inner 인덱스 좋음, Top-N/페이징 | **Nested Loops** | 첫 행 빠른 응답, 부분범위처리 적합 |
| 대량 데이터 등치 조인, 메모리 충분 | **Hash Join** | 선형 확장성, 대량 처리 효율적 |
| 범위 조인, 양쪽 정렬된 입력 | **Merge Join** | 정렬된 입력 활용, 등치/범위 조인 혼합 가능 |
| Outer 매우 큼, Inner 인덱스 부실 | **Hash Join** | 랜덤 I/O 폭발 방지 |

### NL이 부적합한 예제

```sql
-- Outer가 매우 크고 Inner 인덱스가 부적절한 경우
SELECT /* BAD_NL_CANDIDATE */
       o.order_id,
       i.amount
FROM   orders      o
JOIN   order_items i ON i.order_id = o.order_id
WHERE  o.order_dt >= DATE '2024-01-01'
AND    o.status IN ('PAID','SHIP');
```

이 쿼리의 문제점:
- `orders` 테이블 필터 선택도가 낮아 Outer 결과 집합이 매우 큽니다.
- `order_items`에 적절한 인덱스가 없다면 NL 조인 시 반복적인 풀스캔이 발생합니다.

해결 방안:
1. Inner 테이블에 `(order_id)` 인덱스 생성
2. 또는 해시 조인으로 전환

```sql
SELECT /*+ USE_HASH(i) FULL(o) FULL(i) */
       o.order_id,
       i.amount
FROM   orders      o
JOIN   order_items i ON i.order_id = o.order_id
WHERE  o.order_dt >= DATE '2024-01-01'
AND    o.status IN ('PAID','SHIP');
```

---

## 추가 예제 모음 — 다양한 NL 패턴

### 고객 최근 주문 + 품목 Top-N

```sql
SELECT /*+ LEADING(o) USE_NL(i) */
       o.order_id,
       o.order_dt,
       i.item_no,
       i.amount
FROM   orders      o
JOIN   order_items i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 50 ROWS ONLY;
```

### EXISTS 기반 세미 조인

```sql
SELECT /*+ USE_NL_SJ(o) */
       c.cust_id
FROM   customers c
WHERE  c.region = 'APAC'
AND    EXISTS (
         SELECT /*+ USE_NL_SJ(i) */
                1
         FROM   orders      o
         JOIN   order_items i ON i.order_id = o.order_id
         WHERE  o.cust_id = c.cust_id
         AND    o.status  = 'PAID'
       );
```

### Outer Join에서의 NL

```sql
SELECT /*+ LEADING(c) USE_NL(o) */
       c.cust_id,
       c.name,
       o.order_id,
       o.order_dt
FROM   customers c
LEFT JOIN orders o
       ON o.cust_id = c.cust_id
      AND o.order_dt >= DATE '2024-01-01'
WHERE  c.region = 'EMEA'
AND    ROWNUM <= 100;
```

---

## 고급 주제: 병렬 처리, 파티션, 카디널리티 추정

### 병렬 처리와 NL 조인

일반적으로 Hash Join이 병렬 처리에 더 적합하지만, 특정 조건에서는 병렬 NL도 유용할 수 있습니다.

**병렬 NL이 효과적인 경우:**
- Outer 결과 집합이 중간 규모
- Inner 인덱스 lookup이 매우 빠름
- 시스템 I/O 대역폭이 충분함

**동작 방식:** Outer 결과 집합이 병렬 서버 간에 분할되고, 각 서버가 자신의 Outer 행에 대해 Inner 인덱스를 독립적으로 탐색합니다.

### 파티션 테이블과 NL 조인

파티션을 활용하면 NL 조인의 효율성을 더욱 높일 수 있습니다.

1. **파티션 프루닝**: Outer 테이블에서 불필요한 파티션을 제외하면 Outer 행 수가 크게 줄어듭니다.
2. **파티션 인덱스**: Inner 테이블이 파티션 인덱스를 사용하면 각 lookup이 특정 파티션으로 제한되어 탐색 범위가 축소됩니다.

### 카디널리티 추정 실패와 NL 선택

옵티마이저의 카디널리티 추정이 실패하면 잘못된 조인 방식을 선택할 수 있습니다.

**문제 시나리오:**
- 실제 Outer 행 수: 수만 건
- 옵티마이저 추정: 수십 건
- 결과: NL 조인 선택 → 실제 실행 시 랜덤 I/O 폭발

**해결 방안:**
1. 통계 및 히스토그램 재수집
2. 필요 시 힌트로 Hash Join 유도
3. SQL 플랜 베이스라인으로 안정적인 실행계획 고정

---

## NL 조인 성능 분석 실전 가이드

### 단계별 분석 프로세스

1. **쿼리 실행 및 통계 수집**
   ```sql
   SELECT /*+ GATHER_PLAN_STATISTICS */ ... FROM ...;
   SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
   ```

2. **핵심 지표 확인**
   - Outer `A-Rows` vs Inner `Starts`: NL 패턴 확인
   - Inner `Buffers` / `Starts`: 평균 lookup 비용 계산
   - 총 `Buffers`: 전체 I/O 비용 평가

3. **튜닝 포인트 식별**
   - Outer 크기 감소 가능성 검토
   - Inner 인덱스 개선 방안 모색
   - 조인 순서 변경 고려

4. **대안 조인 방식 검토**
   - Hash/Merge 조인으로 변경 시 예상 비용 비교
   - 실제 데이터 볼륨과 사용 패턴 고려

### 실제 모니터링 사례

```sql
-- 튜닝 전
SELECT * FROM orders o JOIN order_items i ON i.order_id = o.order_id WHERE o.cust_id = :cid;

-- 실행계획 통계:
-- | Id | Operation           | Name | Starts | A-Rows | Buffers |
-- |----|---------------------|------|--------|--------|---------|
-- |  0 | SELECT STATEMENT    |      |      1 |    150 |    2100 |
-- |  1 |  NESTED LOOPS       |      |      1 |    150 |    2100 |
-- |  2 |   INDEX RANGE SCAN  | IX1  |      1 |    150 |      15 |
-- |  3 |   TABLE ACCESS BY...| ITEMS|    150 |    150 |    2085 |
```

분석 결과:
- Outer(`orders`): 150행
- Inner(`order_items`): 150회 탐색, 평균 13.9블록/탐색
- 개선 포인트: Inner 테이블 액세스 비용이 높음 → 인덱스 추가 또는 커버링 인덱스 고려

---

## 결론

Nested Loops 조인은 특정 조건에서 매우 우수한 성능을 발휘하는 조인 방식입니다. 효과적인 NL 조인을 위해서는 다음 세 가지 원칙을 기억해야 합니다.

1. **Outer를 작게 유지하라**  
   강력한 필터 조건, 파티션 프루닝, Top-N 제한 등을 활용해 Outer 결과 집합의 크기를 최소화합니다.

2. **Inner를 빠르게 탐색하라**  
   조인 키 선두 인덱스, 커버링 인덱스, 좋은 클러스터링 팩터를 통해 Inner lookup 비용을 낮춥니다.

3. **정렬을 인덱스로 흡수하라**  
   ORDER BY 조건을 인덱스 스캔 순서와 일치시켜 정렬 작업을 제거하고 Stopkey 최적화를 활용합니다.

NL 조인은 OLTP 환경의 대화형 애플리케이션, 웹 페이징, 실시간 조회 등에서 빛을 발합니다. 그러나 대량 데이터 배치 처리나 리포트 생성에는 해시 조인이 더 적합할 수 있습니다. 각 조인 방식의 특징을 이해하고 상황에 맞게 선택하는 것이 데이터베이스 성능 최적화의 핵심입니다.

실제 환경에서는 `DBMS_XPLAN`의 `ALLSTATS LAST` 옵션을 활용해 실행 통계를 꼼꼼히 분석하고, 힌트를 사용해 다양한 조인 전략을 실험해 보는 것이 중요합니다. 데이터 분포, 인덱스 설계, 쿼리 패턴을 종합적으로 고려해 최적의 조인 방식을 선택하세요.