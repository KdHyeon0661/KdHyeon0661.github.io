---
layout: post
title: DB 심화 - Nested Loops 조인
date: 2025-11-10 15:25:23 +0900
category: DB 심화
---
# Nested Loops 조인 — 기본 메커니즘 → 힌트 제어 → 실행계획·통계 해석 → 특징·튜닝 포인트

## 0) 공통 세션 설정 (실습 환경 준비)

Nested Loops 조인 실습에서 **재현 가능한 실행계획/통계**를 얻기 위해, 먼저 세션 환경을 맞춰 둔다.

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';
ALTER SESSION SET statistics_level = ALL;        -- ALLSTATS LAST (실제 수행 통계)
ALTER SESSION SET optimizer_features_enable = '19.1.0'; -- 환경에 맞춰 조정(예: 19c 기준)
```

- `statistics_level = ALL`  
  - 나중에 `DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +IOSTATS +MEMSTATS')` 로 **실행 통계**를 보기 위함.
- `optimizer_features_enable`  
  - 옵티마이저 동작을 특정 버전 기준으로 고정. 실습 재현성이 높아진다.

---

## 1) Nested Loops 조인의 직관과 기본 메커니즘

### 1.1 직관적인 그림

Nested Loops(NL) 조인의 핵심 직관은 아래 한 줄로 요약된다.

> **바깥(Outer)에서 한 행씩 가져온 뒤 → 안쪽(Inner)을 그 키로 즉시 찾아본다.**

조인 과정은 대략 다음과 같다.

1. Outer row-source에서 **한 행**을 꺼낸다.
2. 이 행의 **조인 키**로 Inner row-source에 **인덱스 탐색**을 수행한다.
3. Inner에서 매칭되는 행들을 읽어 **결과로 출력**한다.
4. 다시 Outer의 **다음 행**으로 이동하여 2)~3)을 반복한다.
5. Outer가 끝나면 조인이 종료된다.

이때 핵심은:

- **Outer는 “작게”**  
- **Inner는 “빠르게(=인덱스 lookup)”**

라는 구도이다.

### 1.2 단순 비용 모델

Nested Loops 조인 비용을 아주 단순화하면 다음과 같이 볼 수 있다.

$$
\text{총비용} \approx \text{Outer Rows} \times (\text{Inner Lookup 비용})
$$

여기서 **Inner Lookup 비용**은 보통

- 인덱스 탐색(루트·브랜치·리프 블록 접근)
- 테이블 블록 0~N개 랜덤 I/O

의 합이다.

조인 비용 감각을 키우려면, 실제 실행계획에서 다음을 같이 보는 것이 중요하다.

- Outer row-source의 **A-Rows(실제 출력 행 수)**
- Inner row-source의 **Starts(루프 횟수)**, **Buffers(논리 I/O)**

실제 DBMS_XPLAN 통계에서 자주 보게 될 패턴은:

- Outer 테이블: `Starts = 1`, `A-Rows = n`  
- Inner 테이블: `Starts = n`, `A-Rows = n에 비례`, `Buffers = n × (평균 lookup 비용)`

이다.

### 1.3 언제 좋은가? (적합한 상황)

다음 조건이 맞으면 Nested Loops는 **대단히 강력한** 조인 방식이 된다.

1. Outer가 **작다**  
   - 예: `WHERE` 절 필터로 선택도가 높은 조건, 파티션 프루닝, `ROWNUM` / `FETCH FIRST` 등
2. Inner에 **선두 = 등치 조인**이 걸리고,  
   그 컬럼에 대해 **좋은 인덱스**가 존재
3. 조회 패턴이 **Top-N / 부분범위처리** 중심이다.  
   - 웹 화면에서 상위 N개만 보여주는 경우
   - OLTP 조회: 한 건/몇 건만 빠르게 보고 싶은 경우

이때 Nested Loops의 장점은:

- **첫 번째 행**을 매우 빠르게 보여줄 수 있다.
- **전체 결과가 필요하지 않아도** 앞부분만 보고 즉시 응답을 끝낼 수 있다.

### 1.4 언제 나빠지는가? (주의해야 할 상황)

반대로 아래와 같은 상황에서는 Nested Loops가 **심각하게 느려질 수 있다**.

1. Outer가 **매우 크다** (예: 수십만, 수백만 행)  
2. Inner 인덱스가 부실하다
   - 없거나
   - 클러스터링 팩터가 매우 나쁘거나
3. 랜덤 I/O가 폭발하는 구조
   - Outer 행 하나당 Inner 테이블 랜덤 블록 여러 개를 읽어야 하는 경우

이 경우 해시 조인이나 머지 조인이 훨씬 유리하다.

---

## 2) 실습 스키마: customers / orders / order_items / products

Nested Loops 조인의 다양한 패턴을 살펴볼 수 있도록 **실습용 스키마**를 구성한다.  
(기존 초안의 스키마를 기반으로 하되, 그대로 전부 포함하고 설명을 보강한다.)

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

### 2.1 데이터 로딩

실습용으로 **수만~수십만 건 규모**의 데이터를 로딩한다.  
(환경별 성능에 따라 LOOP 상한을 조절하면 된다.)

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
      MOD(o,50000) + 1,  -- 고객 분포
      DATE '2024-01-01' + MOD(o,540), -- 약 1년 반 범위 날짜
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

### 2.2 인덱스 생성 (NL 조인에 적합하도록 설계)

```sql
CREATE INDEX ix_orders_cdt      ON orders(cust_id, order_dt, order_id);
CREATE INDEX ix_orders_status   ON orders(status, order_dt);
CREATE INDEX ix_oi_order        ON order_items(order_id, item_no);
CREATE INDEX ix_oi_prod         ON order_items(prod_id);
CREATE INDEX ix_prod_category   ON products(category, price);
```

- `ix_orders_cdt(cust_id, order_dt, order_id)`  
  - **특정 고객의 주문을 날짜 역순으로 Top-N** 조회하는 데 최적.
- `ix_oi_order(order_id, item_no)`  
  - 주문번호별 주문품목 lookup에 사용.
- `ix_prod_category(category, price)`  
  - 카테고리 + 가격 범위 기반 조회에 사용.

### 2.3 통계 수집

```sql
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'CUSTOMERS',
    cascade    => TRUE,
    method_opt => 'for all columns size skewonly'
  );
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'ORDERS',
    cascade    => TRUE,
    method_opt => 'for all columns size skewonly'
  );
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'ORDER_ITEMS',
    cascade    => TRUE,
    method_opt => 'for all columns size skewonly'
  );
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'PRODUCTS',
    cascade    => TRUE,
    method_opt => 'for all columns size skewonly'
  );
END;
/
```

- `cascade => TRUE`  
  - 테이블뿐만 아니라 관련 인덱스까지 통계 수집.
- `size skewonly`  
  - 데이터 분포가 치우친 컬럼에 대해 히스토그램 생성.

---

## 3) 기본 예제: 고객의 최근 주문 Top-N + 상세 (NL 조인 이상적 케이스)

가장 전형적인 Nested Loops의 강점 상황을 보자.

> "특정 고객의 **최근 주문 상위 20건** + 각 주문의 품목·상품 정보를 조회"

### 3.1 기본 쿼리

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

#### 왜 Nested Loops가 적합한가?

1. `WHERE o.cust_id = :cid`  
   - 특정 고객 한 명의 주문만 보므로, Outer 후보가 **매우 작다**.
2. `ORDER BY o.order_dt DESC, o.order_id DESC`  
   - `ix_orders_cdt` 인덱스를 적절히 사용하면,  
     **정렬을 생략**하면서 **최신 주문부터** 읽을 수 있다.
3. `FETCH FIRST 20 ROWS ONLY`  
   - Stopkey 최적화로 **앞의 몇 블록만 읽고 조기 종료** 가능.
4. 각 `o.order_id`에 대해 `order_items`를 `(order_id, item_no)` 인덱스로 빠르게 lookup.
5. `products`는 PK 인덱스로 즉시 lookup.

즉, 전체적인 패턴은:

- Outer: `ORDERS`  
  - 특정 고객에 대한 **최신 20건**만 읽음
- Inner1: `ORDER_ITEMS`  
  - 각 주문별 품목을 인덱스로 빠르게 lookup
- Inner2: `PRODUCTS`  
  - 각 품목에 대한 상품 정보를 PK로 lookup

### 3.2 Nested Loops를 강하게 유도하는 힌트

실제로 옵티마이저가 Hash Join을 선택할 수도 있으므로,  
NL 조인 패턴을 명확히 보고 싶다면 힌트를 활용한다.

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

SELECT *
FROM   TABLE(
         DBMS_XPLAN.DISPLAY_CURSOR(
           NULL, NULL,
           'ALLSTATS LAST +PREDICATE +PEEKED_BINDS'
         )
       );
```

힌트의 의미:

- `LEADING(o)`  
  - 조인 순서에서 `o`를 가장 먼저 사용.
- `USE_NL(i) USE_NL(p)`  
  - `i`, `p`를 Inner로 하는 NL 조인을 유도.
- `INDEX(o ix_orders_cdt)`  
  - `ix_orders_cdt` 인덱스를 사용하도록 액세스 경로 지정  
    (정렬 흡수 + Stopkey)
- `INDEX(i ix_oi_order)`  
  - `order_items`의 `(order_id, item_no)` 인덱스를 사용하도록 지정.

---

## 4) 힌트로 NL 조인 제어하기 (종합 정리)

### 4.1 조인 순서와 조인 메소드

NL 조인뿐만 아니라 다른 메소드와의 비교 실험을 할 때도 아래 힌트를 자주 사용한다.

- 조인 순서
  - `/*+ ORDERED */`  
    - FROM 절에 나열된 순서대로 조인.
  - `/*+ LEADING(t1 t2 t3) */`  
    - 명시적으로 조인 순서를 지정.
- 조인 메소드
  - `/*+ USE_NL(t) */`  
    - `t`를 Inner로 하는 NL 조인 사용 유도.
  - `/*+ NO_USE_NL(t) */`  
    - `t`에 대해 NL 조인을 금지.
  - `/*+ USE_HASH(t) */`  
    - 해시 조인 유도.
  - `/*+ USE_MERGE(t) */`  
    - 머지 조인 유도.

### 4.2 Inner 액세스 경로 힌트

NL 조인에서는 Inner 액세스를 어떻게 하느냐가 매우 중요하다.

- `/*+ INDEX(t idx) */`  
  - 인덱스 스캔 사용.
- `/*+ INDEX_ASC(t idx) */`, `INDEX_DESC`  
  - 정렬 방향까지 제어 가능.
- `/*+ FULL(t) */`  
  - 풀 테이블 스캔 사용.
- `/*+ USE_NL_WITH_INDEX(t idx) */`  
  - `t`를 NL로 조인하면서 **반드시 이 인덱스를 사용**하도록 강하게 유도.

실제로는 `USE_NL_WITH_INDEX`가 `USE_NL + INDEX`보다 더 강하게 동작하는 경우가 많다.

### 4.3 세미/안티 조인에서의 NL

`EXISTS` / `NOT EXISTS`를 사용하면, 옵티마이저는 종종 **세미/안티 조인을 NL로 구현**한다.

- `/*+ USE_NL_SJ */`  
  - Nested Loops Semi Join
- `/*+ USE_NL_AJ */`  
  - Nested Loops Anti Join  
  (버전에 따라 `NL_SJ`, `NL_AJ` 형식도 사용)

예제:

```sql
-- 세미 조인: 특정 고객이 ELEC 카테고리 상품을 산 적이 있는지
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

- `EXISTS` 서브쿼리는 **첫 매칭 행만 찾으면 바로 종료**하는 특성이 있다.
- NL 세미 조인과 궁합이 매우 좋다.

---

## 5) NL 조인 수행 과정 분석 — DBMS_XPLAN으로 보는 row-source 통계

이제 실제로 실행하고, `ALLSTATS LAST`로 **실행 후 통계**를 보면서  
NL 조인의 동작을 해부해 보자.

### 5.1 예제 쿼리 (고객의 PAID 주문 + 품목)

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

그리고 실행 직후:

```sql
SELECT *
FROM   TABLE(
         DBMS_XPLAN.DISPLAY_CURSOR(
           NULL, NULL,
           'ALLSTATS LAST +PREDICATE +IOSTATS +MEMSTATS'
         )
       );
```

### 5.2 row-source 통계에서 볼 포인트

실제 출력 예시는 환경마다 다르므로, 여기서는 **읽는 법**을 중심으로 설명한다.

중요 항목:

- `Starts`
  - 해당 row-source가 **몇 번 시작**되었는지
  - Inner 쪽이 **Outer Rows 수만큼** 시작되는 패턴이 NL의 특징이다.
- `A-Rows`
  - 실제 출력된 행 수.
- `Buffers`
  - 논리 I/O 블록 수 (캐시 Hit 포함).
- `Reads`
  - 물리 I/O 블록 수.
- `Predicate Information` 섹션
  - `access`  
    - 인덱스/파티션 **진입 조건**. 스캔 범위를 실제로 줄이는 조건.
  - `filter`  
    - 읽은 후에 **거르는 조건**. 스캔량 자체를 줄이지 못한다.

Nested Loops 조인에서는 대개 이런 패턴을 보게 된다.

- Outer row-source(`ORDERS`)
  - `Starts = 1`
  - `A-Rows ≈ 30` (Stopkey)
  - `Buffers` 상대적으로 작음
- Inner row-source(`ORDER_ITEMS`)
  - `Starts ≈ 30` (Outer행 수와 동일)
  - `A-Rows` = 각 주문의 품목 수 총합
  - `Buffers` = 30 × 평균 lookup 비용

### 5.3 NL 조인의 총 비용에 대한 수식적 감각

NL 조인의 비용을 간단히 모델링해 보면:

- Outer row-source에서 행 수를 $$n_o$$  
- Inner row-source에서 각 lookup당 평균 논리 블록 수를 $$c_i$$ 라 하면,

$$
\text{총 Buffers} \approx \text{Outer Buffers} + n_o \times c_i
$$

특히 Inner 쪽의 랜덤 I/O가 전체 비용을 지배하는 경우가 많다.

- $$n_o$$를 줄이는 전략 (Outer Rows 감소)
- $$c_i$$를 줄이는 전략 (더 좋은 인덱스, 더 나은 클러스터링 등)

이 둘이 NL 튜닝의 핵심이다.

---

## 6) NL 조인의 특징 — 장점과 단점

### 6.1 장점

1. **작은 Outer + 빠른 인덱스 Inner**일 때 **매우 빠른 응답**  
2. Top-N / 페이징 / 화면 조회에 **최고의 궁합**
   - 첫 페이지를 빠르게 보여줄 수 있음.
3. **부분범위처리**에 자연스럽게 대응
   - 결과를 **흘려보내면서** 바로 클라이언트에 전달할 수 있다.
4. 인덱스 설계를 잘 하면
   - **정렬 흡수** 가능
   - 심지어 **테이블 방문 없이 인덱스만으로** 결과를 낼 수도 있다 (Index Only).
5. Inner 쪽에 추가 필터가 많아도, 빠른 인덱스 lookup으로 **필요 최소만 읽을 수 있음**.

### 6.2 단점·주의사항

1. Outer가 커지고 Inner 인덱스가 부실하면
   - 랜덤 I/O 폭발
   - 응답 시간이 선형, 때로는 더 크게 증가.
2. 인덱스 클러스터링 팩터가 나쁘면
   - 인덱스 순서와 테이블 물리 순서가 상관이 적어
   - 테이블 블록 랜덤 접근이 심해짐.
3. 대량 집계·배치 처리에는 해시/머지 조인이 더 적합한 경우 많음.
4. NL은 본질적으로 **파이프라인 구조**라
   - 대용량 테이블 풀스캔 + 병렬 처리에는 적합하지 않을 수 있다.

---

## 7) NL 조인을 좋게 만드는 설계 포인트 (인덱스·데이터·쿼리 관점)

### 7.1 인덱스 설계

1. Inner 테이블에 **조인 컬럼을 선두로 하는 복합 인덱스** 설계  
   - 조인 조건이 `t.col = :bind` 형태로 SARGable해야 한다.
2. Outer의 `ORDER BY`를 **인덱스 컬럼 순서와 맞추기**
   - 정렬 생략 + Stopkey 최적화 가능.
3. **커버링 인덱스**  
   - Inner에서 필요한 컬럼을 인덱스에 모두 포함시키면  
     테이블 블록 방문 없이 인덱스만으로 결과를 만들 수 있다.
4. 함수기반 인덱스, 가상 컬럼
   - `UPPER(col) = 'ABC'` 같은 조건이 필요하다면,
   - `CREATE INDEX ... ON t(UPPER(col))` 처럼 함수기반 인덱스로 SARGability 확보.

### 7.2 데이터·통계 관점

1. **통계 최신화**
   - `DBMS_STATS.GATHER_TABLE_STATS`를 정기적으로 수행.
2. **히스토그램으로 스큐 보정**
   - 선택도가 크게 치우친 컬럼(예: status, category)에 대해
   - 히스토그램을 두어 옵티마이저의 Cardinality 추정을 개선.
3. **클러스터링 팩터 모니터링**
   - 인덱스가 순차 접근에 가까운지, 랜덤 접근에 가까운지의 지표.
4. **바인드 피킹 / ACS(Adaptive Cursor Sharing)**
   - 바인드 변수 값에 따라 선택도가 크게 다를 경우
   - 계획 안정성을 위해 값별로 전략을 분리할지,  
     플랜 관리(SQL Plan Baseline)를 적용할지 결정.

### 7.3 쿼리 작성 관점

1. Outer에 **강한 필터**를 최대한 밀어 넣기
   - 가능하면 Outer가 되는 테이블 쪽에 `WHERE` 조건을 집약.
2. `ROWNUM`, `FETCH FIRST` 등 Top-N 구조를 적극 활용
3. `EXISTS`를 이용한 세미 조인
   - 필요 없는 중간 결과(중복 행)를 줄인다.
4. 필요하다면 쿼리를 분리
   - “먼저 키만 가져오기 → 키 리스트 기반으로 In-list lookup” 같은 구조로 바꾸는 것도 고려.

---

## 8) NL vs Hash vs Merge — 간단 감별법과 예제

Nested Loops가 항상 좋은 것은 아니다. Hash/Merge와의 비교 기준을 잡아보자.

### 8.1 감별 표

| 상황 | 추천 조인 방식 |
|------|---------------|
| Outer가 **작고**, Inner 인덱스가 좋으며, Top-N/페이징 | **Nested Loops** |
| 양쪽 테이블 모두 **대량 스캔**, 조인 키로 해시 분배 유리, 메모리 충분 | **Hash Join** |
| 양쪽이 **정렬/정렬된 액세스**를 쉽게 할 수 있음, 범위 조인·등치 조인 혼합 | **Merge Join** |

- 현실에서는 옵티마이저가 비용 기반으로 선택.
- **핵심 쿼리**는 설계·힌트로 **의도한 조인 방식이 나오도록 길을 깔아주는 것**이 튜닝의 핵심이다.

### 8.2 NL이 나쁜 후보인 예제 (Outer 큼 + Inner 인덱스 부적절)

```sql
-- Outer가 방대하고 Inner 인덱스가 없으면 랜덤 I/O 폭발
SELECT /* BAD_NL_CANDIDATE */
       o.order_id,
       i.amount
FROM   orders      o
JOIN   order_items i ON i.order_id = o.order_id
WHERE  o.order_dt >= DATE '2024-01-01'  -- 범위가 매우 넓다
AND    o.status IN ('PAID','SHIP');     -- 선택도 낮다(행이 많다)
```

이 쿼리는:

- `orders`의 필터가 약해 Outer가 매우 커질 수 있다.
- 만약 `order_items`에 `(order_id)` 인덱스가 없다면,
  - `order_items`를 반복적으로 풀스캔하는 NL 조인이 될 위험이 크다.

해결 전략:

1. Inner 테이블에 `(order_id)` 인덱스를 만든다 (`ix_oi_order`).
2. 그래도 Outer가 너무 크다면, 아래처럼 해시 조인을 시도해 볼 수 있다.

```sql
SELECT /*+ USE_HASH(i) FULL(o) FULL(i) */
       o.order_id,
       i.amount
FROM   orders      o
JOIN   order_items i ON i.order_id = o.order_id
WHERE  o.order_dt >= DATE '2024-01-01'
AND    o.status IN ('PAID','SHIP');
```

- 전체를 풀스캔하여 Hash Join을 수행하는 방식.
- 대량 배치/집계에는 이쪽이 더 효율적일 수 있다.

---

## 9) 추가 예제 모음 — 다양한 NL 패턴

### 9.1 고객 최근 주문 + 품목 Top-N (정렬·Stopkey 결합)

```sql
-- 인덱스: (orders) (cust_id, order_dt DESC, order_id DESC)
--         (order_items) (order_id, item_no)
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

- Outer: `orders` (특정 고객, 정렬 일치 인덱스 → Stopkey)
- Inner: `order_items` (order_id 기반 인덱스 lookup)

### 9.2 EXISTS 기반 세미 조인 (NL 유도)

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

- `EXISTS` 안에서 Inner 조인이 NL로 풀리도록 유도.
- Outer: `customers` (region='APAC')
- Inner: `orders`, `order_items`

### 9.3 Outer Join에서의 NL

```sql
-- 고객과 최근 주문을 Outer Join으로 가져오기 (주문이 없는 고객도 보고 싶을 때)
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

- Outer: `customers`
- Inner: `orders`
- Outer Join에서도 NL 방식이 자주 사용된다.

---

## 10) NL 수행 과정을 깊이 있게 관찰하는 팁

다음은 기존 초안을 바탕으로, NL 수행 과정을 단계적으로 분석하는 흐름이다.

```sql
-- 1) 쿼리 실행
SELECT /* NL_TRACE */
       o.order_id,
       o.order_dt,
       i.item_no
FROM   orders      o
JOIN   order_items i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
AND    o.status  = 'PAID'
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 10 ROWS ONLY;

-- 2) 실행계획/통계 확인
SELECT *
FROM   TABLE(
         DBMS_XPLAN.DISPLAY_CURSOR(
           NULL, NULL,
           'ALLSTATS LAST +PREDICATE +PEEKED_BINDS'
         )
       );
```

여기서 row-source 통계를 해석할 때 확인할 점:

- `ORDERS` 노드
  - `Starts = 1`인지
  - `A-Rows = 10` (Stopkey)
  - `Buffers`가 작은지
- `ORDER_ITEMS` 노드
  - `Starts = 10`
  - `A-Rows = 각 주문의 품목 수 총합`
  - `Buffers`가 적당한지 (과도하게 크지 않은지)
- `Predicate Information`에서  
  - `access` 조건에 `i.order_id = :B1` 같은 형태가 있는지  
    → NL 조인에서 Inner 인덱스 lookup의 키.

---

## 11) 고급 주제: NL과 병렬 처리, PARTITION, 카디널리티 추정

### 11.1 병렬 처리와 NL

- 일반적으로 Hash Join이 병렬 처리에 더 어울리지만,
- 특정 상황에서는 NL + 인덱스 스캔도 병렬을 사용할 수 있다.
- 다만 NL의 **루프 구조** 특성상,  
  Outer 쪽이 병렬로 분할되고 Inner 쪽 인덱스 lookup이 병렬 세션으로 분배되는 식으로 동작한다.

병렬 NL이 좋은 경우:

- Outer가 꽤 크지만,
- Inner lookup이 매우 빠르고,
- 시스템 I/O 대역폭이 넉넉할 때.

### 11.2 파티션 테이블·인덱스와 NL

- Outer 테이블에 파티션 프루닝이 강하게 걸리면
  - Outer Rows 자체가 크게 줄고
  - 이 상태에서 Inner NL lookup이 매우 효율적으로 동작.
- Inner 테이블이 파티션 인덱스를 사용하면
  - 각 lookup이 특정 파티션으로 제한되어 탐색 범위가 더 줄어듦.

### 11.3 카디널리티 추정 실패와 잘못된 NL 선택

카디널리티 추정이 실패하면, 옵티마이저는 잘못된 조인 방식을 선택할 수 있다.

예:

- 실제로는 Outer Rows가 **수만 건**인데,
- 통계 부족/스큐로 인해 옵티마이저는 **수십 건**으로 추정하고
- NL + 인덱스 lookup 플랜을 선택 → 실제로는 랜덤 I/O 폭발.

이런 상황에서는:

1. 통계·히스토그램 재수집.
2. 필요하다면 힌트로 Hash Join 유도.
3. 플랜 관리로 안정성을 확보.

---

## 12) 실무용 체크리스트 (정리)

아래 체크리스트는 기존 초안을 살리면서, 문장을 다듬고 약간 보강한 버전이다.

- [ ] **Outer를 작게 만들 수 있는가?**
  - 선행 등치 조건, 강한 필터, 파티션 프루닝, Top-N/ROWNUM 등.
- [ ] **Inner 인덱스는 적절한가?**
  - 조인 키가 선두 = 등치 조건으로 사용되는가?
  - 커버링 인덱스를 구성할 수 있는가?
  - 클러스터링 팩터는 어떤 상태인가?
- [ ] **정렬을 인덱스로 흡수할 수 있는가?**
  - `ORDER BY`와 인덱스 컬럼 순서·방향이 일치하는가?
  - Stopkey 최적화를 활용하고 있는가?
- [ ] **SARGability는 확보되어 있는가?**
  - `WHERE` 절 조건에서 불필요한 함수/형변환을 제거했는가?
  - 필요하다면 가상 컬럼/함수기반 인덱스를 활용했는가?
- [ ] **ALLSTATS LAST로 Starts/A-Rows/Buffers를 비교했는가?**
  - 튜닝 전/후 수치를 실제로 비교해 보았는가?
- [ ] Outer가 커지면 해시/머지도 검토했는가?
  - 힌트로 NL / Hash / Merge를 각각 유도해 보고, 계획과 통계를 비교했는가?
- [ ] 바인드·스큐·ACS에 따른 플랜 변동성을 관리하고 있는가?
  - 값에 따라 전략이 달라야 한다면 SQL을 분리했는가?
  - 플랜 관리(Plan Baseline, SQL Profile 등)를 사용할지 검토했는가?

---

## 13) 요약

마지막으로, 전체 내용을 한 번에 정리하면 다음과 같다.

1. **Nested Loops 조인**은  
   - 바깥(Outer)에서 적은 수의 행을 가져와  
   - 안쪽(Inner)을 **인덱스 lookup**으로 반복 탐색하는 구조이다.

2. 비용은 대략

   $$
   \text{총비용} \approx \text{Outer Rows} \times (\text{Inner Lookup 비용})
   $$

   으로 볼 수 있으며,  
   특히 **Inner 쪽 랜덤 I/O**가 성능을 좌우한다.

3. NL이 **특히 빛나는 상황**은:
   - Outer Rows가 **작고**,
   - Inner 인덱스 설계가 좋으며,
   - Top-N, 페이징, 부분범위처리 형태인 **OLTP/화면 조회**.

4. 힌트 `LEADING`, `ORDERED`, `USE_NL`, `USE_NL_WITH_INDEX`, `INDEX`,  
   그리고 `USE_NL_SJ`, `USE_NL_AJ`(세미/안티 조인)를 활용하면  
   **조인 순서/메소드/액세스 경로**를 세밀하게 제어할 수 있다.

5. `DBMS_XPLAN.DISPLAY_CURSOR(... 'ALLSTATS LAST +PREDICATE +IOSTATS +MEMSTATS')` 로  
   - `Starts`, `A-Rows`, `Buffers`, `Reads`를 확인하면서
   - NL의 루프 패턴과 I/O 총량을 숫자로 검증하는 것이 중요하다.

6. Outer가 커지거나 Inner 인덱스가 부실하면  
   - NL은 랜덤 I/O 폭발로 이어지고,  
   - Hash/Merge 조인을 쓰는 편이 더 낫다.

7. 결국 **좋은 Nested Loops**의 핵심은:

   - **작은 Outer**
   - **강력한 Inner 인덱스**
   - **정렬 흡수 가능 인덱스 + Stopkey**

   이 세 가지를 동시에 만족시키는 설계이다.

이 글의 실습 스키마와 예제 쿼리를 실제 환경에서 실행해 보고,  
각 조인 메소드(NL / Hash / Merge)를 힌트로 유도해 가며  
`ALLSTATS LAST` 통계를 비교해 보면,  
Nested Loops 조인의 동작과 장단점을 몸으로 느낄 수 있을 것이다.