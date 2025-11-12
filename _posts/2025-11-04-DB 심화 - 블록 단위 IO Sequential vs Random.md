---
layout: post
title: DB 심화 - 블록 단위 I/O Sequential vs Random
date: 2025-11-04 15:25:23 +0900
category: DB 심화
---
# 블록 단위 I/O — **Sequential vs Random** 패턴 이해와 **순차 접근 비중 높이기·랜덤 접근 줄이기**

> **한 줄 요약**  
> Oracle의 물리 I/O는 크게 **Sequential(다중블록)** 과 **Random(단일블록)** 로 나뉜다.  
> - 순차 I/O는 **연속 블록**을 **한 번에** 읽으므로 **처리량(throughput)** 이 높다. (Full Scan, Direct Path Read, Index Fast Full Scan 등)  
> - 랜덤 I/O는 **여기저기 흩어진 블록**을 **한 블록씩** 읽는다. (ROWID 방문, Nested Loops의 테이블 접근 등)  
> 성능의 핵심은  
> 1) **순차 접근의 선택도**를 높여 **읽는 블록 자체를 줄이고**,  
> 2) **랜덤 접근 발생량**을 구조적으로 줄이는 것.  
> 이를 위해 **파티셔닝·부분범위처리(Stopkey)·클러스터링 팩터 개선·해시 조인/블룸 필터·커버링 인덱스·IOT/클러스터 테이블** 등을 활용한다.

---

## 0. 실습 스키마(공통)

```sql
-- 고객, 주문, 주문품목 — 2백만~5백만 행 규모를 가정
DROP TABLE order_items PURGE;
DROP TABLE orders PURGE;
DROP TABLE customers PURGE;

CREATE TABLE customers (
  cust_id      NUMBER PRIMARY KEY,
  region       VARCHAR2(10),
  grade        VARCHAR2(10),
  created_at   DATE
);

CREATE TABLE orders (
  order_id     NUMBER PRIMARY KEY,
  cust_id      NUMBER NOT NULL REFERENCES customers(cust_id),
  order_dt     DATE   NOT NULL,
  status       VARCHAR2(10),
  amount       NUMBER(12,2)
)
PARTITION BY RANGE (order_dt) (
  PARTITION p_2024q4 VALUES LESS THAN (DATE '2025-01-01'),
  PARTITION p_2025q1 VALUES LESS THAN (DATE '2025-04-01'),
  PARTITION p_2025q2 VALUES LESS THAN (DATE '2025-07-01'),
  PARTITION p_2025q3 VALUES LESS THAN (DATE '2025-10-01'),
  PARTITION p_2025q4 VALUES LESS THAN (DATE '2026-01-01')
);

CREATE TABLE order_items (
  order_id   NUMBER NOT NULL REFERENCES orders(order_id),
  line_no    NUMBER NOT NULL,
  product_id NUMBER NOT NULL,
  qty        NUMBER,
  price      NUMBER(10,2),
  CONSTRAINT pk_order_items PRIMARY KEY (order_id, line_no)
);

-- 인덱스
CREATE INDEX ix_orders_cust_dt   ON orders(cust_id, order_dt DESC, order_id DESC);
CREATE INDEX ix_orders_status    ON orders(status);
CREATE INDEX ix_oi_prod          ON order_items(product_id);

-- (테스트용)統계 수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'CUSTOMERS', cascade=>TRUE);
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'ORDERS', cascade=>TRUE);
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'ORDER_ITEMS', cascade=>TRUE);
END;
/
```

> 이후 모든 예제는 **원리·전략 비교**를 위한 코드이며, 실제 성능 측정은 `DBMS_XPLAN.DISPLAY_CURSOR`, TKPROF/SQL Monitor 로 전후를 확인하십시오.

---

## 1. 블록 단위 I/O 개요 — **Sequential vs Random**

### 1.1 Sequential(순차) I/O
- **특징**: **연속된 블록**을 **멀티블록**으로 읽음 → **처리량↑**  
- **대표 연산**:  
  - **Table Full Scan** (버퍼드 또는 Direct Path Read)  
  - **Index Fast Full Scan** (인덱스의 모든 리프 블록을 스캔)  
  - **Hash Join** 빌드/프로브 시 큰 테이블의 **Full Scan**  
  - **Partition Full Scan**(Pruning이 되면 “작은 전체”를 순차 읽기)  
- **대기 이벤트(개념)**: (버전에 따라 명칭 차이 있지만) `db file scattered read`(Full Scan), `direct path read` 등

### 1.2 Random(랜덤) I/O
- **특징**: **비연속 블록**을 **한 블록씩** 읽음 → **지연↑**  
- **대표 연산**:  
  - **Index Range/Unique Scan + Table by ROWID** (Nested Loops에서 inner 쪽)  
  - **Bitmap→ROWID 변환** 후 테이블 방문  
  - 분산된 ROWID 방문이 많은 **OLTP** 패턴  
- **대기 이벤트(개념)**: `db file sequential read`(단일 블록 읽기) 등

### 1.3 왜 순차가 유리한가?
- 디스크/스토리지·OS·파일 시스템·DB 버퍼에서 **연속 블록**은 **읽기 헤더 이동/컨텍스트 스위칭**이 적음  
- 멀티블록 수(MBRC, $$m$$라 하자)가 크면 **한 I/O당 읽는 블록 수**가 $$m$$  
  - **순차 읽기량**(블록): $$\text{IO 수} \times m$$  
  - **랜덤 읽기량**(블록): $$\text{IO 수} \times 1$$  
- 같은 블록 수를 읽더라도 **호출 횟수**가 적을수록 CPU/락/래치 경합이 적고, 네트워크/컨텍스트 전환도 줄어든다.

---

## 2. Sequential vs Random — 실행계획 관점 비교

### 2.1 대량 범위 조건: 해시 조인 + Full Scan(순차) vs 중첩 루프(랜덤)

**(A) 나쁜 패턴 — NL 조인으로 inner 테이블 랜덤 접근 폭증**
```sql
-- 큰 범위의 고객을 걸고, 각 고객의 최근 주문 합계를 구함 (잘못된 통계/힌트로 NL이 선택되는 경우)
SELECT /* bad */ c.cust_id, SUM(o.amount)
FROM   customers c
JOIN   orders    o ON o.cust_id = c.cust_id
WHERE  c.region = :r
  AND  o.order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -6)
GROUP  BY c.cust_id;
```
- 계획: `NESTED LOOPS` + `INDEX RANGE SCAN orders(cust_id, order_dt)` + **BY ROWID** 반복  
- **Random I/O**: 범위가 넓으면 ROWID 방문이 **수십만~수백만** 건

**(B) 개선 — 해시 조인 + 파티션 프루닝 + Full Scan(순차)**
```sql
SELECT /*+ USE_HASH(o) FULL(o) */
       c.cust_id, SUM(o.amount)
FROM   customers c
JOIN   orders    o
   ON  o.cust_id = c.cust_id
WHERE  c.region = :r
  AND  o.order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -6)
GROUP  BY c.cust_id;
```
- 계획: `HASH JOIN` + `PARTITION RANGE SINGLE/ITERATOR` + `TABLE FULL SCAN ORDERS`  
- **Sequential I/O**: 분기/프루닝된 파티션만 **연속 블록**으로 읽음 → 처리량 유리

> **핵심**: **넓은 범위**·**큰 테이블**·**집계/조인 중심**은 **해시 조인 + 순차 읽기**가 구조적으로 유리하다.

---

## 3. **Sequential 액세스의 “선택도” 높이기** — “작은 전체”만 빠르게 훑기

> 여기서 “선택도”는 “순차 스캔으로 읽는 **범위 자체를 얼마나 줄였는가**”라는 실용적 의미로 사용

### 3.1 파티셔닝 + 프루닝(Pruning)
- **시간/해시/리스트** 파티셔닝으로 **불필요 파티션 제외** → “작은 전체”만 Full Scan  
- **예제: 최근 분기만 스캔**
```sql
SELECT /*+ FULL(o) */
       COUNT(*)
FROM   orders PARTITION (p_2025q4) o
WHERE  o.status = 'OK';
```
- 옵티마이저가 조건으로도 프루닝 가능:
```sql
SELECT COUNT(*) 
FROM   orders o
WHERE  o.order_dt >= DATE '2025-10-01'  -- p_2025q4만
AND    o.status = 'OK';
```

### 3.2 조인 필터/블룸 필터(Join Filter / Bloom Filter)
- **해시 조인** 단계에서 **필터(블룸) 비트셋**으로 **대상 파티션/블록**을 더 줄임  
- **예제: 고객 후보를 먼저 좁힌 후 주문 스캔**
```sql
WITH c AS (
  SELECT /*+ MATERIALIZE */ cust_id
  FROM customers
  WHERE region='APAC' AND grade='VIP'
)
SELECT /*+ USE_HASH(o) FULL(o) */
       o.cust_id, SUM(o.amount)
FROM   c
JOIN   orders o
  ON   o.cust_id = c.cust_id
WHERE  o.order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -3)
GROUP  BY o.cust_id;
```
- 실행계획에 `JOIN FILTER`(bloom) 가 보이면, **orders** 스캔 중 **대상 아닌 키를 즉시 배제** → **읽기량↓**

### 3.3 **부분범위처리(Stopkey, TOP-N/Keyset)** + 인덱스 정렬 일치
- **목록 첫 페이지/Top-N** 은 **정렬 포함 복합 인덱스** + `FETCH FIRST N` 으로 **STOPKEY** 유도  
```sql
SELECT /*+ index(o ix_orders_cust_dt) */
       o.order_id, o.order_dt, o.amount
FROM   orders o
WHERE  o.cust_id = :cust
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 50 ROWS ONLY;  -- STOPKEY
```
- 인덱스가 정렬을 해결하므로 **정렬 없이** **필요한 앞부분만** 읽고 멈춤

### 3.4 커버링 인덱스 / 인덱스 Fast Full Scan
- 필요한 컬럼이 인덱스에 **다 있으면** 테이블로 내려가지 않고 **연속된 리프 블록**만 스캔  
```sql
-- 간단 집계/조건이 인덱스 컬럼으로만 이루어질 때
SELECT /*+ INDEX_FFS(o ix_orders_status) */
       status, COUNT(*)
FROM   orders o
WHERE  order_dt >= TRUNC(SYSDATE) - 7
GROUP  BY status;
```
- `INDEX FAST FULL SCAN` 은 **순차**로 대량 블록을 읽기 쉬운 패턴

---

## 4. **Random 액세스 발생량 줄이기** — “ROWID 점프”를 구조적으로 억제

### 4.1 **Nested Loops 남발** 방지 (넓은 범위일수록 해시/병합 조인)
- **규칙**: **선택도가 낮거나(=많이 매칭), 범위가 넓으면 해시 조인**  
- **힌트/통계**로 옵티마이저가 **NL→HJ** 를 고르도록 유도  
```sql
SELECT /*+ USE_HASH(o) FULL(o) */ ...
```
- **히스토그램/컬럼 그룹 통계**로 **카디널리티 추정** 정확화 → 잘못된 NL 선택 방지

### 4.2 **클러스터링 팩터(Clustering Factor)** 개선 → 인덱스 Range Scan의 “준-순차”화
- **클러스터링 팩터**가 낮을수록(=테이블 물리 순서가 인덱스 키 순서와 유사) **ROWID 방문이 근접**  
- **개선 방법**: 테이블을 **인덱스 키 순으로 재정렬(CTAS/Move)** 후 인덱스 재생성
```sql
-- 테이블을 주문일자+고객 순으로 재정렬
CREATE TABLE orders_sorted NOLOGGING AS
SELECT * FROM orders
ORDER BY order_dt, cust_id, order_id;

ALTER TABLE orders RENAME TO orders_old;
ALTER TABLE orders_sorted RENAME TO orders;

-- 인덱스 재생성
DROP INDEX ix_orders_cust_dt;
CREATE INDEX ix_orders_cust_dt ON orders(cust_id, order_dt DESC, order_id DESC);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'ORDERS',cascade=>TRUE, method_opt=>'for all columns size skewonly');
END;
/
```
- 이후 `user_indexes.clustering_factor` 를 확인하면 **블록 수에 근접**하게 낮아져야 이상적  
- 결과적으로 **Range Scan + BY ROWID** 패턴의 **랜덤성**이 **크게 줄어듦**

### 4.3 **커버링 인덱스** / **IOT(인덱스 조직 테이블)** / **클러스터 테이블**
- **커버링**: SELECT-LIST/조건/JOIN 컬럼이 인덱스에 모두 존재 → **Table by ROWID 제거**  
- **IOT**: PK 순서로 **데이터 자체가 인덱스에 저장** → PK 기반 접근은 **랜덤성↓**  
- **클러스터 테이블**: 같은 키의 여러 로우를 **한 블록/근접 블록**에 저장 → **멀티 로우 접근 시 효율**

### 4.4 **Bitmap Index + Bitmap Combine** (DW/보고계)
- **저카디널리티** 다수 조건을 **비트 연산**으로 결합 후 한 번에 ROWID 변환  
- 단, **DML 많은 OLTP**에는 부적합(잠금/유지비용 큼)

### 4.5 **서브쿼리/세미조인**으로 **불필요한 BY ROWID 방문** 차단
- `EXISTS`/`IN` 을 활용하여 **테이블 방문 전** 후보 축소  
```sql
SELECT /*+ SEMIJOIN */ o.*
FROM   orders o
WHERE  EXISTS (
  SELECT 1 FROM customers c
  WHERE  c.cust_id = o.cust_id
  AND    c.region = :r
);
```

### 4.6 **통계·히스토그램**으로 계획 오류 방지
- 선택도 오판 → NL 선택 → 랜덤 I/O 폭증  
- **히스토그램**(skew), **컬럼 그룹 통계**(상관관계), **정확한 NUM_ROWS/NDV** 로 예측 개선

---

## 5. “Sequential 선택도↑ + Random↓” 통합 시나리오

### 5.1 요구사항  
- “APAC VIP 고객의 **최근 3개월 주문 합계 Top-50** 보여줘”

**(Before) — NL + OFFSET + 정렬 비용 + 랜덤 I/O 다발**
```sql
SELECT /* before */
       c.cust_id, SUM(o.amount) AS s
FROM   customers c
JOIN   orders    o ON o.cust_id = c.cust_id
WHERE  c.region='APAC' AND c.grade='VIP'
  AND  o.order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -3)
GROUP  BY c.cust_id
ORDER  BY s DESC
OFFSET :skip ROWS FETCH NEXT :take ROWS ONLY;
```

**문제점**  
- 범위 넓음 + NL 선택 시 **ROWID 랜덤 방문 폭증**  
- `OFFSET` 으로 **앞 페이지 버리기** → **불필요 I/O**

**(After) — 해시 조인 + 파티션 프루닝 + 블룸 + TOP-N(Stopkey)**
```sql
WITH c AS (
  SELECT /*+ MATERIALIZE */ cust_id
  FROM customers
  WHERE region='APAC' AND grade='VIP'
)
SELECT /*+ USE_HASH(o) FULL(o) */ 
       o.cust_id, SUM(o.amount) AS s
FROM   c
JOIN   orders o
  ON   o.cust_id = c.cust_id
WHERE  o.order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -3)
GROUP  BY o.cust_id
ORDER  BY s DESC
FETCH FIRST 50 ROWS ONLY;  -- STOPKEY
```
**효과**  
- **프루닝**: 최근 분기/월 파티션만 읽음 → **Sequential “작은 전체”**  
- **블룸 필터**: `c.cust_id` 로 **불필요 키** early drop  
- **해시 조인**: 큰 테이블 스캔을 **순차**로, 랜덤 BY ROWID 제거  
- **TOP-N**: 정렬/집계 후 **앞부분만** 출력 → I/O/CPU 절감

---

## 6. 클러스터링 팩터와 예상 랜덤 I/O 근사

### 6.1 개념 공식 (직관적 근사)
- **랜덤 I/O(블록 수)** 대략:
$$
\text{Random Blocks} \approx \frac{\text{방문 로우 수}}{\text{평균 로우/블록}} \times \alpha
$$
여기서 \(\alpha \in [1,\frac{\text{블록 수}}{\text{클러스터링팩터}}]\) 는 **물리적 흩어짐 정도**(클러스터링 팩터가 낮을수록 \(\alpha \to 1\)).  
- **Sequential I/O(블록 수)** 대략:
$$
\text{Seq IO Calls} \approx \frac{\text{스캔 블록 수}}{\text{MBRC}}
$$
(\(\text{MBRC}\): 멀티블록 읽기 크기)

> **해석**: 같은 로우 수를 읽어도 **물리적 근접성**(클러스터링 팩터 개선)으로 랜덤 블록 수가 크게 준다.

### 6.2 팩터 확인
```sql
SELECT index_name, clustering_factor, leaf_blocks, num_rows
FROM   user_indexes
WHERE  table_name='ORDERS';
```
- **이상적**: `clustering_factor` 가 **테이블 블록 수**에 근접  
- **개선 전후**를 비교하여 **Range Scan** 기반 쿼리의 랜덤 I/O 감소를 체감

---

## 7. 인덱스 전략으로 랜덤 I/O 줄이기

### 7.1 **커버링 인덱스** 설계
- 자주 쓰는 조회(필터/정렬/SELECT-LIST)를 분석해 **복합 인덱스**로 덮기  
- 테이블 방문 제거(= BY ROWID 제거) → 랜덤 I/O 대폭 감소  
```sql
-- 예: 고객별 최근 주문 리스트 API
CREATE INDEX ix_orders_cover ON orders(cust_id, order_dt DESC, order_id DESC, amount, status);
```

### 7.2 **Index Fast Full Scan** 활용
- 집계/COUNT/조건이 인덱스에 다 있을 때 **순차적 인덱스 스캔**으로 고속 처리  
- 필요 시 `INDEX_FFS` 힌트로 유도

### 7.3 **IOT/클러스터 테이블**
- **IOT**: PK접근은 곧 데이터 접근 → **랜덤 I/O**가 **인덱스 수준**  
- **클러스터 테이블**: 같은 키(예: `cust_id`)의 로우를 **근접 저장** → **여러 로우를 한 블록에서**

---

## 8. DW/리포트에서의 패턴

### 8.1 Bitmap 인덱스 + Bitmap Combine
```sql
-- 지역/등급/상태 저카디널리티 조합을 bitmap 으로 결합 후 일괄 ROWID 변환
SELECT /* DW */
       COUNT(*)
FROM   orders
WHERE  region='APAC'
AND    status='OK'
AND    amount BETWEEN 100 AND 500;
```
- **장점**: 여러 조건을 **비트 연산**으로 빠르게 결합 → 실제 테이블 방문 전 **후보 극소화**  
- **주의**: DML 많은 OLTP에는 **부적합**

### 8.2 파티션 와이즈 조인(Partition-Wise Join)
- 동일 파티션 키로 분할된 두 테이블을 **파티션 단위로 독립 조인** → **순차 I/O** + **병렬 효율↑**

---

## 9. 클라이언트/애플리케이션 레벨의 보조

- **부분범위처리(Stopkey)** 로 “읽을 양” 자체를 줄인다(Top-N, Keyset 페이지).  
- **배열 페치(Array Fetch)** 로 **Fetch 왕복 수**를 줄여 I/O 대기와 컨텍스트 전환을 줄인다.  
- **불필요 컬럼**을 빼서 **전송 바이트**와 **캐시 압력**을 줄인다.  
- **바인드 변수** 사용으로 커서/플랜 재사용 → 불필요 파싱/재컴파일/스핀락 감소.

---

## 10. 안티 패턴 → 교정 요약

| 안티 패턴 | 결과 | 교정 |
|---|---|---|
| 큰 범위인데 **Nested Loops** | **ROWID 랜덤 I/O 폭증** | **Hash Join** + **Full/Partition Scan** |
| **OFFSET** 페이지 | 앞부분 버리기 → **읽기 낭비** | **Keyset + Stopkey** |
| 정렬/필터 인덱스 불일치 | **SORT + FullScan** | **복합 인덱스**(필터+정렬 포함) |
| 클러스터링 팩터 높음 | Range Scan도 **랜덤화** | **CTAS(정렬 후 재적재)** + 인덱스 재구성 |
| 커버링 부족 | BY ROWID 빈번 | **커버링 인덱스**, **IOT** |
| 통계 부정확 | 잘못된 NL 선택 | **히스토그램/컬럼 그룹 통계** |

---

## 11. BEFORE → AFTER: 실전형 코드 모음

### 11.1 대량 범위 + 합계

**BEFORE (랜덤 I/O 다발)**
```sql
SELECT /* before */
       c.cust_id, SUM(o.amount)
FROM   customers c
JOIN   orders    o ON o.cust_id = c.cust_id
WHERE  c.region = :r
  AND  o.order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -12)
GROUP  BY c.cust_id;
```

**AFTER (순차 I/O 중심)**
```sql
SELECT /*+ USE_HASH(o) FULL(o) */
       c.cust_id, SUM(o.amount)
FROM   (SELECT /*+ MATERIALIZE */ cust_id
        FROM customers
        WHERE region = :r) c
JOIN   orders o
  ON   o.cust_id = c.cust_id
WHERE  o.order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -12)
GROUP  BY c.cust_id;
```

### 11.2 Top-N 목록 API

**BEFORE (OFFSET + 정렬 인덱스 없음)**
```sql
SELECT order_id, order_dt, amount
FROM   orders
WHERE  cust_id = :cust
ORDER  BY order_dt DESC
OFFSET :skip ROWS FETCH NEXT :take ROWS ONLY;
```

**AFTER (정렬 포함 인덱스 + Stopkey)**
```sql
SELECT /*+ index(o ix_orders_cust_dt) */
       order_id, order_dt, amount
FROM   orders o
WHERE  o.cust_id = :cust
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST :take ROWS ONLY;
```

### 11.3 Range Scan 품질(클러스터링) 개선

**개선 전**
```sql
SELECT /* Range Scan + BY ROWID 많음 */
       SUM(amount)
FROM   orders
WHERE  cust_id = :cust
AND    order_dt BETWEEN :d1 AND :d2;
```

**개선 후(테이블 재정렬 + 인덱스 재생성)**
```sql
-- 위 4.2절의 CTAS/재생성 수행 후 동일 SQL 실행
-- 기대: 같은 Range Scan이라도 BY ROWID 방문이 근접하여 물리 I/O 감소
```

---

## 12. 체크리스트(배포 전)

- [ ] 큰 범위·저선택도 쿼리는 **해시 조인 + 순차 읽기**인가?  
- [ ] **파티션 프루닝**이 정확히 되고 있는가? (`PARTITION RANGE` 연산 확인)  
- [ ] **정렬/필터**를 만족하는 **복합 인덱스**(가능하면 커버링)인가?  
- [ ] Top-N/페이지는 **Keyset + Stopkey**로 구현했는가?  
- [ ] Range Scan 중심 작업에서 **클러스터링 팩터**가 낮은가? (필요 시 **재정렬**)  
- [ ] 통계(히스토그램/컬럼 그룹)로 **조인 방식 오판**을 막았는가?  
- [ ] DW용 다중 조건은 **Bitmap/Combine**으로 후보를 줄였는가?  
- [ ] 실행계획에서 **`STOPKEY`/`FULL/PARTITION SCAN`/`USE_HASH`** 등이 의도대로 잡혔는가?

---

## 결론

- **Sequential vs Random** 의 차이는 곧 **처리량 vs 지연**의 차이이다.  
- 성능 최적화는  
  1) **순차 접근의 “선택도”** 를 높여 **읽을 범위를 최소화**하고(파티셔닝·블룸·Stopkey·커버링),  
  2) **랜덤 접근 발생량**을 구조적으로 줄이는 것(해시 조인, 클러스터링 팩터 개선, IOT/클러스터 테이블, 커버링 인덱스).  
- 이 원리를 실행계획과 통계로 검증하며 적용하면, **대규모 데이터**에서도 **일관된 저지연**을 달성할 수 있다.

---

## 부록: 수식/직관

- **Random I/O 비율**  
  $$\text{Random Ratio} \approx \frac{\text{Random IO Calls}}{\text{Total IO Calls}}$$  
  → **해시 조인/Full Scan**으로 **분모(순차 IO)** 를 늘리고, **클러스터링 개선/커버링**으로 **분자(랜덤 IO)** 를 줄여라.

- **Stopkey 효과(Top-N)**  
  $$\text{읽기 블록 수} \approx \min\{\text{필요 블록 수}, \text{MBRC} \times \text{IO Calls}\}$$  
  → 정렬이 인덱스로 해결되면 **앞부분만** 읽고 멈춘다.

- **클러스터링 팩터 영향**  
  $$\text{ROWID 방문당 추가 블록} \propto \frac{\text{클러스터링 팩터}}{\text{테이블 블록 수}}$$  
  → **재정렬(CTAS)** 로 분모 대비 분자를 줄이면 **랜덤성↓**.

```sql
-- 요약 예: “작은 전체 + 순차 + Stopkey”의 정석
WITH vip AS (
  SELECT /*+ MATERIALIZE */ cust_id FROM customers
  WHERE region='APAC' AND grade='VIP'
)
SELECT /*+ USE_HASH(o) FULL(o) */
       o.cust_id, SUM(o.amount) s
FROM   vip v
JOIN   orders o ON o.cust_id = v.cust_id
WHERE  o.order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -3)
GROUP  BY o.cust_id
ORDER  BY s DESC
FETCH FIRST 50 ROWS ONLY; -- STOPKEY
```