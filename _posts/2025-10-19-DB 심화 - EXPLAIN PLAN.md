---
layout: post
title: DB 심화 - EXPLAIN PLAN
date: 2025-10-19 23:25:23 +0900
category: DB 심화
---
# Oracle `EXPLAIN PLAN` 완전 가이드

— **PLAN_TABLE**부터 `DBMS_XPLAN`, 카디널리티·코스트 해석, 바인드 피킹/적응 계획, 조인전략 비교, DML/병렬/파티션/힌트 분석까지 (19c 기준)

> 목표
> - `EXPLAIN PLAN`이 **무엇을** 보여주고(“**예상** 계획”), **무엇은 아닌지**(**실제 실행**이 아님)를 정확히 이해.
> - **PLAN_TABLE**과 `DBMS_XPLAN` 옵션(`BASIC/ALL/ALLSTATS/OUTLINE/PREDICATE/PROJECTION/ALIAS/NOTE`)을 익혀 **현장 가독성**을 극대화.
> - 인덱스/조인/정렬/파티션/병렬/서브쿼리/뷰 머지/매터뷰 재작성/Hint/Outline/Plan Hash 등 **핵심 신호**를 읽는 법.
> - **카디널리티(추정행수)**·**코스트(COST)**·**선택도(Selectivity)** 해석 및 **통계·히스토그램·바인드 피킹**의 영향.
> - `EXPLAIN PLAN` vs `DISPLAY_CURSOR(ALLSTATS LAST)` vs SQL Monitor의 **역할/한계/조합 사용법**.
> - **실습 예제**(스크립트)와 **분석 절차 템플릿** 제공.

---

## 한눈 핵심

- `EXPLAIN PLAN FOR <SQL>` 은 **지금 이 세션의 Optimizer**가 **지금 보이는 통계/바인드**(대개 **리터럴**만 반영)로 **예상** 계획을 **PLAN_TABLE**에 기록한다.
- 실제로 실행하지 않는다 → **실제 실행 경로/행수/시간**은 다를 수 있다.
- 실제 실행 계획·실행 통계는 **`DBMS_XPLAN.DISPLAY_CURSOR(..., 'ALLSTATS LAST +PEEKED_BINDS')`** 또는 **SQL Monitor**에서 확인하라.
- `EXPLAIN PLAN`은 **사전 시뮬레이션**과 **힌트 효과/옵티마이저 결정 근거**를 보기 좋다.
- 실무는 보통 **세 가지를 함께** 본다:
  1) `EXPLAIN PLAN`(예상)
  2) `DISPLAY_CURSOR ALLSTATS LAST`(실행 후 실제)
  3) SQL Monitor(실시간/Plan Line별 시간·TEMP·I/O)

---

## 준비 — PLAN_TABLE과 기본 사용

### PLAN_TABLE 확인/생성

```sql
-- 12c+ 대부분 스키마에 PLAN_TABLE 존재. 없으면:
@?/rdbms/admin/utlxplan.sql      -- PLAN_TABLE 생성 스크립트
```

### 기본 문법

```sql
EXPLAIN PLAN FOR
SELECT /* test */ o.order_id, o.amount
FROM   orders o
WHERE  o.customer_id = :cid
AND    o.order_date BETWEEN :d1 AND :d2;

-- 출력
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'BASIC +ALIAS'));
-- 또는 최신 표준 출력
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY()); -- plan_table에서 최신 캡처
```

> `DISPLAY`의 세 번째 인자는 **옵션 문자열**:
> - `BASIC`, `SERIAL`, `TYPICAL`, `ALL`
> - `+PREDICATE`(필터/액세스), `+PROJECTION`, `+ALIAS`, `+OUTLINE`, `+NOTE`, `+PARALLEL` 등

---

## 예제 데이터 준비

```sql
-- 고객·주문 예제
DROP TABLE customers PURGE;
DROP TABLE orders PURGE;

CREATE TABLE customers(
  id        NUMBER PRIMARY KEY,
  region    VARCHAR2(10),
  grade     NUMBER,
  name      VARCHAR2(100)
);

CREATE TABLE orders(
  order_id    NUMBER PRIMARY KEY,
  customer_id NUMBER NOT NULL REFERENCES customers(id),
  order_date  DATE   NOT NULL,
  amount      NUMBER NOT NULL
);

-- 인덱스
CREATE INDEX ix_orders_cust_date ON orders(customer_id, order_date);
CREATE INDEX ix_cust_region_grade ON customers(region, grade);

-- 데이터(개략)
BEGIN
  FOR i IN 1..100000 LOOP
    INSERT INTO customers VALUES (i, CASE MOD(i,10) WHEN 0 THEN 'APAC' WHEN 1 THEN 'EMEA' ELSE 'AMER' END, MOD(i,5), 'C'||i);
  END LOOP;
  COMMIT;

  FOR i IN 1..2000000 LOOP
    INSERT INTO orders VALUES (i, MOD(i,100000)+1, DATE '2024-01-01'+MOD(i,365), MOD(i,500)+1);
    IF MOD(i,100000)=0 THEN COMMIT; END IF;
  END LOOP;
  COMMIT;
END;
/
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'CUSTOMERS',cascade=>TRUE);
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'ORDERS',cascade=>TRUE);
```

---

## `EXPLAIN PLAN` 출력 읽기 — 핵심 칼럼/라인 해석

### 샘플 플랜

```sql
EXPLAIN PLAN FOR
SELECT /* plan1 */ SUM(o.amount)
FROM   orders o
WHERE  o.customer_id = :cid
AND    o.order_date  BETWEEN DATE '2024-06-01' AND DATE '2024-06-30';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'ALL +PREDICATE +ALIAS +PROJECTION'));
```

**읽는 포인트**
- **Id/Operation/Name**: 연산자(Access/Join/Sort/Group), 대상 이름
- **Rows(=카디널리티 추정)**, **Bytes**, **Cost**, **Time**: 옵티마이저 추정치
- **Predicate Information**:
  - **access**: 인덱스 접근/조인 키
  - **filter**: 접근 후 필터링(추가 조건)
- **Note**: 동적 샘플링/카디널리티 피드백/적응 통계 등의 힌트

> **중요**: `Rows`는 **추정**이며 실제와 다를 수 있음. 실제 행수는 `ALLSTATS LAST`에서 **A-Rows**로 확인.

### 자주 보는 Operation

- `INDEX RANGE SCAN`, `INDEX UNIQUE SCAN`, `TABLE ACCESS BY ROWID`
- `HASH JOIN`, `NESTED LOOPS`, `MERGE JOIN`
- `SORT ORDER BY` / `GROUP BY` / `FILTER`
- `PARTITION RANGE SINGLE/ITERATOR`(파티션 프루닝)
- `PX` 관련(`PX COORDINATOR`, `PX SEND`, `PX BLOCK`) — **병렬**

---

## `EXPLAIN PLAN` vs 실제 — 차이와 함께 보기

### 실제 실행 계획·통계(권장)

```sql
-- (실행 후) 실제 계획/통계
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'ALLSTATS LAST +PEEKED_BINDS +OUTLINE +PREDICATE +PROJECTION +NOTE'));
```
- **`ALLSTATS LAST`**: 마지막 실행의 **라인별 A-Rows, Buffers, TempSpc, Time** 표시
- **`PEEKED_BINDS`**: 바인드 값 스니핑(피킹) 상태
- **`OUTLINE`**: 옵티마이저 Outline(힌트 세트)
- **실제 대기**는 SQL Monitor나 10046 trace에서 확인

### 왜 다를 수 있나?

- `EXPLAIN PLAN`은 실제 **바인드 값**이 없거나 **대표 값**으로 추정 → **바인드 피킹/적응 통계** 미반영 가능
- 실행 중 **적응 조인/부분 수행**(Adaptive)로 경로 변경 가능
- `EXPLAIN PLAN`은 **실행 안 함**: TEMP 스필, 병렬 그레인, RAC GC 등 **런타임 요소** 반영 불가

---

## 카디널리티(행수)·코스트 모델 빠르게 이해

### 용어 감각

- **Selectivity(선택도)**: 조건이 행을 통과시킬 확률 \(0..1\)
- **Cardinality(카디널리티)**: 추정 행수 \(= \text{Table Rows} \times \text{Selectivity}\)
- **Cost**: IO/CPU/네트워크 등 내부 가중 합(단위는 상대치)

> 개념 수식
> $$ \text{Cardinality} \approx N \times \prod_k \text{Selectivity}(Predicate_k) $$
> 히스토그램·조인 카디널리티·상관관계 보정이 들어가면서 실제는 더 복잡.

### 히스토그램·바인드 영향

- 스큐(skew) 심한 컬럼은 **히스토그램**이 없으면 선택도 오판.
- 바인드 변수 사용 시 **바인드 피킹** 상황/ACS(Adaptive Cursor Sharing)에 따라 실행 시점에 **플랜 분기**.

**실습**
```sql
-- 스큐 있는 region을 히스토그램 없이
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'CUSTOMERS',method_opt=>'FOR COLUMNS SIZE 1 region');

EXPLAIN PLAN FOR
SELECT /* region test */ *
FROM   customers
WHERE  region='APAC';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'ALL +PREDICATE +NOTE'));

-- 히스토그램 부여
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'CUSTOMERS',method_opt=>'FOR COLUMNS SIZE 75 region');

EXPLAIN PLAN FOR
SELECT /* region test */ *
FROM   customers WHERE region='APAC';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'ALL +PREDICATE +NOTE'));
```
- **Note**에 **histogram** 언급/선택도 변화 체크.

---

## 조인 전략 읽기 — NL/Hash/Merge 비교

### NL (Nested Loops)

- **드라이빙 소량** + **인덱스 탐색** 유리, OLTP 선호.
- 플랜: 상단에 소량 소스 → 아래쪽 `NESTED LOOPS` → 우측 자식이 **인덱스 + BY ROWID**.

**예제**
```sql
EXPLAIN PLAN FOR
SELECT /* NL preferred */ o.order_id, c.name
FROM   customers c
JOIN   orders o
  ON   o.customer_id = c.id
WHERE  c.region='APAC' AND o.order_date BETWEEN :d1 AND :d2;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'ALL +PREDICATE'));
```

### Hash Join

- **대량 조인/등가 조인**·인덱스 부재·병렬/배치에 유리.
- TEMP 스필 → `direct path read temp` 위험 → **PGA 조정/카디널리티 정확화** 필요.

**예제**
```sql
EXPLAIN PLAN FOR
SELECT /* HASH */ /*+ USE_HASH(o c) */ /* 의도적 강제 */
       COUNT(*)
FROM   orders o JOIN customers c ON o.customer_id=c.id
WHERE  c.grade >= 3;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'ALL +PREDICATE +NOTE'));
```

### Merge Join

- **두 입력이 정렬**되어 있거나 **정렬 비용이 적당**할 때. 범위 조인에 적합.

---

## 파티션/병렬/서브쿼리/뷰

### 파티션 프루닝

```sql
-- 날짜 파티션 테이블이라 가정
EXPLAIN PLAN FOR
SELECT SUM(amount)
FROM   orders
WHERE  order_date BETWEEN DATE '2024-06-01' AND DATE '2024-06-30';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'ALL +PREDICATE +PARTITION'));
```
- Plan에 `PARTITION RANGE SINGLE/ITERATOR` 및 **Pruning 정보** 확인.

### 병렬

```sql
ALTER SESSION SET parallel_degree_policy=MANUAL;

EXPLAIN PLAN FOR
SELECT /*+ parallel(o 8) */ COUNT(*) FROM orders o;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'ALL +PARALLEL +NOTE'));
```
- `PX COORDINATOR`, `PX SEND`, `PX BLOCK` 라인 확인.

### 서브쿼리 방식

- **SEMI/ANTI JOIN**(`IN`, `EXISTS`, `NOT EXISTS`) 변환 여부, **SUBQUERY UNNESTING** 확인(플랜 Note/Outline).

---

## DML의 `EXPLAIN PLAN` (INSERT/UPDATE/DELETE/MERGE)

```sql
EXPLAIN PLAN FOR
UPDATE /* upd test */ orders
SET    amount = amount * 1.1
WHERE  customer_id = :cid
AND    order_date  >= :d;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'ALL +PREDICATE +ALIAS'));
```
- `UPDATE`/`DELETE`의 핵심은 **타겟 행 탐색 계획**(보통 인덱스), 그 뒤 **`UPDATE STATEMENT`** 노드.
- `MERGE`는 **`VIEW  MERGE$`** 아래에 **WHEN MATCHED/NOT MATCHED** 블록이 보이며 각각의 액세스 전략 확인.

---

## `DBMS_XPLAN` 옵션 모음 (실무 즐겨 쓰기 세트)

- `ALL +PREDICATE +ALIAS +PROJECTION +NOTE` : **예상** 계획 상세
- `DISPLAY_CURSOR(..., 'ALLSTATS LAST +PEEKED_BINDS +OUTLINE +PROJECTION +ALIAS +NOTE')` : **실제**
- `+OUTLINE` : 옵티마이저가 내부적으로 적용한 힌트 세트(비즈니스 레벨에서 힌트화하거나 Baseline 만들 때 유용)

---

## 힌트/아웃라인/베이스라인과의 관계

- `EXPLAIN PLAN` 결과에 `OUTLINE`을 붙여보면 옵티마이저가 선택한 **힌트 세트**가 보인다.
- 동일한 결과를 반복 강제하려면 **SQL Plan Baseline**/**SQL Profile**/힌트의 적절한 조합을 고려.
- **주의**: 힌트 남발은 유지보수 악화. **원인(통계/인덱스/설계)**을 먼저 개선.

---

## 바인드 피킹/ACS(Adaptive Cursor Sharing)와 `EXPLAIN PLAN`

- `EXPLAIN PLAN`은 종종 **대표 바인드** 가정으로 추정 → **특정 바인드 값에 따른 플랜 차이** 미표시.
- 실제로는 **`DISPLAY_CURSOR +PEEKED_BINDS`** 로 값별 플랜이 갈리는지 확인.
- 값 분포 스큐가 크면 **히스토그램** + **ACS** 조합으로 해결.

---

## 적응 계획(Adaptive)·통계 피드백

- 12c+에서는 실행 중 **부분 통계**를 보고 **조인 전략 전환**(예: NL ↔ Hash) 가능.
- `EXPLAIN PLAN`엔 **최종 실행 경로**가 반영되지 않을 수 있음 → **SQL Monitor/ALLSTATS LAST** 필수.

---

## 실행계획 분석 **표준 절차** (템플릿)

1) **문제 SQL/상황 정의**: 요구/입력 범위/슬로우 지표
2) **EXPLAIN PLAN**으로 **예상 경로** 파악, `+PREDICATE/+NOTE`로 **프루닝/힌트/동적샘플링** 확인
3) **실행 후 `DISPLAY_CURSOR ALLSTATS LAST`** 로 **실행 라인별 A-Rows/Time/Temp/Buffer** 확인
4) **차이 분석**: `Rows(추정)` vs `A-Rows(실제)` 큰乂 → 통계/히스토그램/바인드/조인 순서 재검토
5) **개선안**: 인덱스/통계/히스토그램/조인순서/힌트(임시)
6) **재검증**: 동일 데이터에서 Before/After **플랜/실행통계** 비교
7) **회귀 방지**: 베이스라인/프로파일/테스트케이스

---

## 케이스 스터디

### 잘못된 카디널리티로 해시 스필

```sql
-- 문제 SQL
EXPLAIN PLAN FOR
SELECT SUM(o.amount)
FROM   orders o JOIN customers c ON o.customer_id=c.id
WHERE  c.region='APAC' AND o.order_date BETWEEN :d1 AND :d2;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'ALL +PREDICATE +NOTE'));

-- 실행 후 확인(실제 TempSpc, Time)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PEEKED_BINDS'));

-- 원인: region 히스토그램 부재로 selectivity 오판 → Hash Join + Sort/Spill
-- 조치: region 히스토그램 생성, 또는 NL 유도(인덱스 우선 탐색)
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'CUSTOMERS',method_opt=>'FOR COLUMNS SIZE 75 region');
```

### 파티션 프루닝 누락

```sql
-- 조건이 함수로 감싸져 프루닝 실패
EXPLAIN PLAN FOR
SELECT SUM(amount) FROM orders
WHERE TRUNC(order_date) BETWEEN :d1 AND :d2;

-- 조치: 함수 제거/함수-기반 인덱스/Virtual Column
EXPLAIN PLAN FOR
SELECT SUM(amount) FROM orders
WHERE order_date >= :d1 AND order_date < :d2 + 1;
```

### NL vs HASH 선택 교정

```sql
-- 예상: NL이 유리해야 하는데 HASH를 택함(추정 과대)
EXPLAIN PLAN FOR
SELECT /* want NL */ o.order_id
FROM   orders o
WHERE  o.customer_id IN (
  SELECT id FROM customers WHERE region='APAC' AND grade>=4
);

-- 임시 힌트(원인 수정 전):
EXPLAIN PLAN FOR
SELECT /*+ USE_NL(o) LEADING(c) */
       o.order_id
FROM   customers c
JOIN   orders o ON o.customer_id=c.id
WHERE  c.region='APAC' AND c.grade>=4;
```

---

## 자주 쓰는 `DBMS_XPLAN` 레시피

```sql
-- 1) 최신 explain plan
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'ALL +PREDICATE +NOTE'));

-- 2) 실행 후 실제
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
  'ALLSTATS LAST +PEEKED_BINDS +OUTLINE +PROJECTION +ALIAS +NOTE'));

-- 3) 특정 SQL_ID/Child
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('6u9v2c3q1k4g9', 0,
  'ALLSTATS LAST +PEEKED_BINDS +NOTE'));

-- 4) 병렬 상황 가독성
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'TYPICAL +PARALLEL +NOTE'));
```

---

## 주의할 함정(요약)

- `EXPLAIN PLAN`만 보고 튜닝 결론 내리면 **반쯤 틀릴 수 있다**: 실제는 **ALLSTATS LAST/Monitor**로 검증.
- 바인드·스큐·적응 계획은 **런타임**에 갈려서 **예상 플랜과 달라진다**.
- **통계 최신화/히스토그램** 없이 힌트로만 고정하면 **데이터 변동에 취약**.
- **뷰/인라인 뷰 머지** 여부(성능에 큰 영향)를 `NOTE/OUTLINE`에서 반드시 확인.

---

## 수학 한 줄(개념)

- 총 코스트 근사:
  $$ \text{Cost} \approx \sum_i \big( \alpha \cdot \text{IO}_i + \beta \cdot \text{CPU}_i \big) $$
  실제 계수/가중은 내부적. **카디널리티 오차**가 **Cost 오차**로 전파되어 **잘못된 경로**를 고른다.

---

## 마무리

- `EXPLAIN PLAN`은 **예상**이다. **실행**의 진실은 `DISPLAY_CURSOR(ALLSTATS LAST)`와 **SQL Monitor**가 말해 준다.
- 분석은 **(1) 예상 → (2) 실제 → (3) 차이 원인(통계/히스토그램/바인드/조인순서/프루닝)** 의 3단 고정 루틴으로.
- 작은 팁: `+PREDICATE +OUTLINE +NOTE` 를 항상 켜서 **옵티마이저의 의도**와 **숨은 변환**을 드러내라.
- 결국 튜닝의 본질은 **카디널리티를 맞추고**, **올바른 접근경로**로, **필요한 것만** 읽게 만드는 일이다.

> 한 줄 결론
> **EXPLAIN PLAN**은 **지도를 보여준다**. 길을 걸었는지는 **ALLSTATS LAST**가,
> 어디서 막혔는지는 **SQL Monitor/ASH**가 알려준다.
> **세 장의 사진**을 겹쳐 보고, **원인**을 고쳐라.
