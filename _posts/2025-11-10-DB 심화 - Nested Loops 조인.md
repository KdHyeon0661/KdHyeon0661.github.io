---
layout: post
title: DB 심화 - Nested Loops 조인
date: 2025-11-10 15:25:23 +0900
category: DB 심화
---
# Nested Loops 조인

— **기본 메커니즘 → 힌트로 제어 → 수행 과정 분석(실제 실행계획/통계 읽기) → 특징/튜닝 포인트**
(실습 코드는 바로 실행 가능하도록 구성. 실행계획/실행 통계는 `DBMS_XPLAN.DISPLAY_CURSOR('ALLSTATS LAST +PREDICATE')`로 확인)

---

## 0. 공통 준비

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';
ALTER SESSION SET statistics_level = ALL;        -- ALLSTATS LAST
ALTER SESSION SET optimizer_features_enable = '19.1.0'; -- (환경에 맞게, 재현성 높이기)
```

---

# 1. Nested Loops 조인: **기본 메커니즘**

## 1.1 직관
- **Outer(바깥) row-source**에서 **한 행**을 꺼낸다 → **Inner(안쪽) row-source**를 **해당 키로 즉시 조회**한다.
- 이 과정을 **반복(loop)** 하며 **매칭되는 행**을 출력한다.
- **눈에 보이는 핵심**: **바깥은 “적게”**, **안쪽은 “빠르게(=인덱스)”**.
  - 바깥이 **고선택도(= 적은 행)**일수록, 안쪽에 **적절한 인덱스**가 있을수록 **강력**.

> 비용 감각(간단 모델)
> $$ \text{총비용} \approx \text{Outer Rows} \times (\text{Inner Lookup 비용}) $$
> Inner Lookup 비용은 보통 **인덱스 탐색 + 소량 블록 랜덤 I/O**.

## 1.2 동작 스텝 (단순 2-테이블 조인)
1) Outer에서 **첫 행** Fetch
2) 그 행의 **조인 키**로 Inner에 **인덱스 탐색(Index Range/Unique Scan)**
3) 매칭되는 Inner 행들을 읽어 **출력**
4) Outer의 **다음 행**으로 이동 → 2)~3) 반복
5) Outer 끝나면 종료

## 1.3 언제 좋은가
- Outer가 **작다**(혹은 빠르게 줄어든다: `ROWNUM`, Top-N, 선택도 높은 필터/파티션 프루닝)
- Inner에 **선행 = 등치**가 걸리며 **인덱스**로 **짧고 빠르게** 찾을 수 있음
- **조기 종료(Stopkey)**, **부분범위처리**가 유리한 화면/OLTP 조회

---

# 2. 실습 스키마

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

-- 데이터 로딩(규모는 환경에 맞게 조정)
BEGIN
  FOR c IN 1..50000 LOOP
    INSERT INTO customers VALUES(c, 'C'||TO_CHAR(c), CASE MOD(c,3) WHEN 0 THEN 'APAC' WHEN 1 THEN 'EMEA' ELSE 'AMER' END);
  END LOOP;

  FOR o IN 1..300000 LOOP
    INSERT INTO orders VALUES(
      o, MOD(o,50000)+1,
      DATE '2024-01-01' + MOD(o,540),
      CASE MOD(o,4) WHEN 0 THEN 'NEW' WHEN 1 THEN 'PAID' WHEN 2 THEN 'SHIP' ELSE 'DONE' END
    );
  END LOOP;

  FOR p IN 1..20000 LOOP
    INSERT INTO products VALUES(p,
      CASE MOD(p,5) WHEN 0 THEN 'ELEC' WHEN 1 THEN 'FASH' WHEN 2 THEN 'FOOD' WHEN 3 THEN 'HOME' ELSE 'TOY' END,
      ROUND(DBMS_RANDOM.VALUE(5, 1000),2));
  END LOOP;

  FOR x IN 1..300000 LOOP
    INSERT INTO order_items VALUES(
      x, 1, MOD(x,20000)+1,
      TRUNC(DBMS_RANDOM.VALUE(1,5)),
      ROUND(DBMS_RANDOM.VALUE(10, 2000),2)
    );
    IF MOD(x,3)=0 THEN
      INSERT INTO order_items VALUES(x, 2, MOD(x+7,20000)+1, TRUNC(DBMS_RANDOM.VALUE(1,5)), ROUND(DBMS_RANDOM.VALUE(10, 2000),2));
    END IF;
  END LOOP;

  COMMIT;
END;
/

-- 인덱스
CREATE INDEX ix_orders_cdt      ON orders(cust_id, order_dt, order_id);
CREATE INDEX ix_orders_status   ON orders(status, order_dt);
CREATE INDEX ix_oi_order        ON order_items(order_id, item_no);
CREATE INDEX ix_oi_prod         ON order_items(prod_id);
CREATE INDEX ix_prod_category   ON products(category, price);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'CUSTOMERS', cascade=>TRUE, method_opt=>'for all columns size skewonly');
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'ORDERS', cascade=>TRUE, method_opt=>'for all columns size skewonly');
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'ORDER_ITEMS', cascade=>TRUE, method_opt=>'for all columns size skewonly');
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'PRODUCTS', cascade=>TRUE, method_opt=>'for all columns size skewonly');
END;
/
```

---

# 3. 기본 예제: 고객의 **최근 주문 Top-N** + 상세 (NL 조인 이상적)

```sql
-- Outer(작음): 고객 필터 + 최신순 Stopkey
VAR cid NUMBER
EXEC :cid := 12345;

SELECT /* BASELINE */
       o.order_id, o.order_dt, i.item_no, i.amount, p.category
FROM   orders o
JOIN   order_items i ON i.order_id = o.order_id
JOIN   products    p ON p.prod_id  = i.prod_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 20 ROWS ONLY;
```

**왜 NL이 적합?**
- `o`는 **cust_id=**로 **짧게** 시작 + 정렬 일치 인덱스가 있으면 **Stopkey**로 **앞 블록**만 읽고 종료.
- 각 `o.order_id`에 대해 `order_items`를 **(order_id, item_no)** 인덱스로 빠르게 lookup.
- `i.prod_id`로 `products`를 PK로 즉시 lookup.
- Outer가 매우 작아 **Inner 랜덤 I/O**의 총량이 작음 → **응답 빠름**.

실행계획을 **NL 조인**으로 보고 싶다면 힌트를 써도 좋다:

```sql
SELECT /*+ LEADING(o) USE_NL(i) USE_NL(p) INDEX(o ix_orders_cdt) INDEX(i ix_oi_order) */
       o.order_id, o.order_dt, i.item_no, i.amount, p.category
FROM   orders o
JOIN   order_items i ON i.order_id = o.order_id
JOIN   products    p ON p.prod_id  = i.prod_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 20 ROWS ONLY;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +PEEKED_BINDS'));
```

- `LEADING(o)` : 조인 순서 고정(먼저 o)
- `USE_NL(i)`, `USE_NL(p)` : 각각에 대해 NL 조인 유도
- `INDEX(o ...)`, `INDEX(i ...)` : 액세스 경로 지정(정렬 흡수/Stopkey 가능)

---

# 4. **힌트**로 NL 조인 제어하기

## 4.1 조인 순서/메소드
- `/*+ ORDERED */` : FROM 순서대로 조인
- `/*+ LEADING(t1 t2 t3) */` : 정확한 조인 순서 지정
- `/*+ USE_NL(t) */` : `t`를 Inner로 NL 사용
- `/*+ NO_USE_NL(t) */` : `t`에 NL 금지(다른 메소드 선택 유도)
- (대비) `USE_HASH(t)`, `USE_MERGE(t)` : 각각 Hash/Merge 유도

## 4.2 Inner 액세스 경로
- `/*+ INDEX(t idx) */`, `INDEX_ASC/DESC` : 인덱스 스캔 지정
- `/*+ FULL(t) */` : 풀스캔
- `/*+ USE_NL_WITH_INDEX(t idx) */` : **t**를 NL로 조인하며 **해당 인덱스**를 반드시 사용(경우에 따라 강제 수준↑)

## 4.3 세미/안티 조인(NL)
- `EXISTS`/`NOT EXISTS`는 **NL 세미/안티** 조인으로 가기 쉬움
- 힌트: `/*+ USE_NL_SJ */`(세미), `/*+ USE_NL_AJ */`(안티) — 버전에 따라 `NL_SJ`, `NL_AJ` 형태도 사용

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

---

# 5. NL 조인 **수행 과정 분석** (실행계획/통계 해석)

다음 쿼리를 실행하고, `ALLSTATS LAST`로 **row-source 통계**를 본다.

```sql
VAR cid NUMBER
EXEC :cid := 23456;

SELECT /* NL_ANALYZE */
       o.order_id, o.order_dt, i.item_no, i.amount
FROM   orders o
JOIN   order_items i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
AND    o.status  = 'PAID'
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 30 ROWS ONLY;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +IOSTATS +MEMSTATS'));
```

**읽는 법(핵심 항목)**
- **Starts**: 해당 row-source가 **몇 번 시작**되었나
  - Inner가 **Outer Rows 수만큼** 시작되는 것을 확인(= 루프)
- **A-Rows**: 실제 출력된 행 수
- **Buffers/Reads**: 논리/물리 I/O (Buffers가 특히 중요)
- **Predicate Information**에서
  - **access**: 인덱스/파티션 **진입 조건**(스캔량 감소)
  - **filter**: 읽고 **거르는** 조건(스캔량에 영향 X)
- **NL 루프 해석**:
  - `ORDERS`에서 **A-Rows=30** (Stopkey)
  - `ORDER_ITEMS`가 **Starts=30**(Outer 30번에 대해 30번 lookup)
  - `ORDER_ITEMS`의 Buffers/Reads가 작으면 인덱스 lookup이 **짧고 빠름**을 의미

> 추정과 비교
> - 예상 총 비용 ≒ `Outer Rows(30)` × `Inner Lookup 비용(평균 Buffers)`
> - `INDEX(ORDER_ITEMS)`의 **클러스터링 팩터**가 낮으면 **랜덤 I/O** 증가 → Buffers↑

---

# 6. NL 조인의 **특징** 정리

## 6.1 장점
- Outer가 작고 Inner lookup이 빠르면 **가장 빠른 응답**
- **Stopkey/Top-N/페이징**과 **궁합 최고** (앞 블록만 읽고 끝)
- **부분범위처리**: 결과를 **흘려보내며** 즉시 화면에 렌더
- 인덱스가 잘 설계되어 있으면 **정렬 생략**(정렬 일치 인덱스) 가능
- 조인 조건 외에도 Inner에서 **추가 필터**를 바로 적용 가능(필요 최소만 읽음)

## 6.2 단점/주의
- Outer가 커지고 Inner 인덱스가 부실하면 **랜덤 I/O 폭증**
- **클러스터링 팩터** 나쁨(= 인덱스 순서와 테이블 물리 순서 상관↓)이면 랜덤 I/O↑
- **배치 집계(대량 스캔)**에는 해시 조인/머지 조인이 유리한 경우 많음
- NL은 **파이프라인** 방식이라 **병렬 스캔 대역폭**을 크게 활용하기 어렵다(상황 따라 다름)

---

# 7. NL 조인 성능을 **좋게 만드는 설계 포인트**

## 7.1 인덱스 설계
- Inner에 **선두 = 등치**가 오도록 **복합 인덱스** 설계
- Outer의 정렬 요구까지 **인덱스로 흡수**(DESC/ASC + Stopkey)
- **커버링 인덱스**로 Inner 테이블 **미방문**(Index Only) 가능성 확보
- **함수기반/가상 컬럼**으로 SARGability 확보(형변환/함수로 인덱스 무력화 방지)

## 7.2 데이터/통계
- 통계 최신화(히스토그램으로 스큐 보정)
- **바인드 피킹/ACS**에 의한 플랜 흔들림 대비(플랜 관리 혹은 값별 전략)
- **클러스터링 팩터** 확인/개선(테이블 재정렬은 어렵지만 파티셔닝/저장 설계/인덱스 재구성 검토)

## 7.3 쿼리 작성
- **필요 최소 조건**을 Outer에 최대한 적용(Outer Rows ↓)
- **ROWNUM/Top-N**을 적극 활용
- `EXISTS`(세미 조인)로 **있음/없음** 판정 → **중간 결과 최소화**

---

# 8. 대조: NL vs Hash vs Merge (간단 감별법)

| 상황 | 추천 |
|---|---|
| **Outer 작음**, Inner 인덱스 좋음, Top-N/페이징 | **NL** |
| 양쪽 **대량 스캔**, 조인 키로 **해시 분배 유리**, 메모리 충분 | **HASH** |
| 양쪽 **정렬/정렬된 액세스** 용이, 등치/범위 조합, 큰 중간 결과를 정렬로 정리 | **MERGE** |

> 현실에서는 옵티마이저가 비용 기반으로 선택. **핵심 쿼리**는 힌트/설계로 의도된 메소드가 나오도록 **길을 깔아준다**.

---

# 9. 추가 예제 모음

## 9.1 고객 최근 주문 + 품목 Top-N (정렬/Stopkey 결합)

```sql
-- 인덱스: (orders) (cust_id, order_dt DESC, order_id DESC)
--         (order_items) (order_id, item_no)
SELECT /*+ LEADING(o) USE_NL(i) */
       o.order_id, o.order_dt, i.item_no, i.amount
FROM   orders o
JOIN   order_items i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 50 ROWS ONLY;
```

## 9.2 EXISTS 기반 세미 조인(NL 유도)

```sql
SELECT /*+ USE_NL_SJ(o) */
       c.cust_id
FROM   customers c
WHERE  c.region = 'APAC'
AND    EXISTS (
         SELECT /*+ USE_NL_SJ(i) */ 1
         FROM   orders o
         JOIN   order_items i ON i.order_id = o.order_id
         WHERE  o.cust_id = c.cust_id
         AND    o.status  = 'PAID'
       );
```

## 9.3 NL이 **나빠지는** 케이스(Outer 큼 + Inner 인덱스 부적절)

```sql
-- Outer가 방대하고 Inner 인덱스가 없으면 랜덤 I/O 폭발
SELECT /* BAD_NL_CANDIDATE */
       o.order_id, i.amount
FROM   orders o
JOIN   order_items i ON i.order_id = o.order_id
WHERE  o.order_dt >= DATE '2024-01-01'  -- 범위가 매우 넓다
AND    o.status IN ('PAID','SHIP');     -- 선택도 낮다(행이 많다)
```

해결책:
- Inner에 `(order_id)` 인덱스 **있어야 함**(여기선 이미 있음)
- 그래도 Outer가 너무 크면 **해시 조인** 고려: `/*+ USE_HASH(i) */`

---

# 10. NL 수행 과정 **깊이 있는** 관찰 팁

```sql
-- 1) 쿼리 실행
SELECT /* NL_TRACE */
       o.order_id, o.order_dt, i.item_no
FROM   orders o
JOIN   order_items i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
AND    o.status  = 'PAID'
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 10 ROWS ONLY;

-- 2) 실행계획/통계
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +PEEKED_BINDS'));

-- 3) row-source 스탯에서 확인할 점
--    - ORDERS: A-Rows(=10), Buffers(작은가?), Starts(=1)
--    - ORDER_ITEMS: Starts(=10), A-Rows(=각 주문의 품목 수 총합), Buffers(작은가?)
--    - access predicate: i.order_id = :B1 ← NL 인덱스 lookup 키
```

---

# 11. 체크리스트

- [ ] **Outer를 작게**: 선행 등치, 강한 필터, 파티션 프루닝, Top-N/ROWNUM
- [ ] **Inner 인덱스**: 조인 키 선두 = 등치 / 커버링 고려 / 클러스터링 팩터 점검
- [ ] **정렬 흡수**: ORDER BY 일치 인덱스 + Stopkey
- [ ] **SARGability 확보**: 함수/형변환 제거(가상 컬럼/함수기반 인덱스)
- [ ] **ALLSTATS LAST**로 Starts/A-Rows/Buffers 비교(전/후 수치로 판단)
- [ ] Outer가 크거나 집계 위주라면 **해시/머지**도 검토(힌트로 대조 실험)
- [ ] 바인드/스큐/ACS에 따른 플랜 변동성 관리(플랜 관리/SQL 분리)

---

## 12. 요약

- **Nested Loops**는 **바깥(적은 행)** × **안쪽(빠른 인덱스 lookup)**의 구조로,
  **화면 조회/Top-N/부분범위처리**에서 **최고의 응답**을 내는 조인 방식이다.
- 힌트 `LEADING/ORDERED/USE_NL/USE_NL_WITH_INDEX/INDEX`로 **조인 순서/메소드/경로**를 제어할 수 있다.
- **실행계획의 Starts/A-Rows/Buffers**를 읽어 루프 패턴과 I/O 총량을 체감하라.
- Outer가 커지거나 Inner 인덱스가 부실하면 **랜덤 I/O 폭발** → **해시/머지**로의 전환을 고민.
- 결국 **좋은 NL**의 핵심은 **작은 Outer + 강력한 Inner 인덱스 + 정렬 흡수**다. 숫자로 검증하고, 필요한 곳에만 쓴다.
