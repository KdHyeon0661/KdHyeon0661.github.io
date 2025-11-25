---
layout: post
title: DB 심화 - Random Access 최소화 튜닝
date: 2025-11-07 18:25:23 +0900
category: DB 심화
---
# 테이블 Random Access 최소화 튜닝

## 1. 문제의 본질 — 왜 “테이블 랜덤 Access”가 병목이 되는가

### 1.1 현대 스토리지에서의 대략적인 비용 감각

최근 미국·유럽 데이터센터 벤치마크를 보면(상용 NVMe SSD 기준 대략 값):

- **연속 읽기**: 수 GB/s 이상
- **랜덤 읽기 IOPS**: 수십만 IOPS(최적 상황)
- **랜덤 읽기 지연(latency)**: 수십~수백 µs

반면 같은 서버의 **CPU L3 캐시/메모리 접근**은 ns ~ 수십 ns 수준이다.  
즉, 아주 거칠게 보면

- 메모리/CPU 연산 대비 **스토리지 랜덤 읽기**는 **수백~수천 배 이상 느리다.**
- DB 버퍼 캐시가 있다고 해도, **캐시에 없는 블록을 랜덤으로 많이 읽는 순간** 전체 응답시간은 바로 I/O 대기 시간에 묶인다.

이를 단순 모델로 쓰면:

$$
\text{응답시간} \approx N_{\text{idx}} \cdot t_{\text{idx}} \;+\; N_{\text{tab}} \cdot t_{\text{random}}
$$

여기서

- \(N_{\text{idx}}\): 인덱스 블록 접근 횟수(대개 빠르고 작다)
- \(N_{\text{tab}}\): **테이블 블록 랜덤 접근** 횟수
- \(t_{\text{random}}\): 랜덤 블록 I/O 평균 소요시간

현대 시스템에서 \(t_{\text{random}}\)이 **지배적**이므로,  
**튜닝의 1순위는 \(N_{\text{tab}}\)를 0에 가깝게 줄이는 것**이다.

이 글의 전개는 모두 이 목표 하나에서 나온다:

- 테이블을 **아예 방문하지 않거나**(Index-Only),
- 방문하더라도 **매우 적게**,  
- 가능하다면 랜덤이 아닌 **준순차** 액세스로 만드는 것.

---

## 2. 실습 스키마 정리 및 현실 가정

### 2.1 기본 실습 스키마

```sql
-- 깨끗이
DROP TABLE ra_orders PURGE;

-- 주문 테이블: 50만 건(랜덤 I/O 체감용)
CREATE TABLE ra_orders (
  order_id    NUMBER        NOT NULL,
  cust_id     NUMBER        NOT NULL,
  order_dt    DATE          NOT NULL,
  status      VARCHAR2(8)   NOT NULL,
  amount      NUMBER(12,2)  NOT NULL,
  note        VARCHAR2(200),
  CONSTRAINT pk_ra_orders PRIMARY KEY (order_id)   -- 단일 PK (surrogate)
);

BEGIN
  FOR i IN 1..500000 LOOP
    INSERT INTO ra_orders
    VALUES (
      i,
      MOD(i, 100000) + 1,                                -- 10만 고객
      DATE '2024-01-01' + MOD(i, 400),                   -- 400일 범위
      CASE MOD(i,5) WHEN 0 THEN 'NEW' WHEN 1 THEN 'PAID'
                    WHEN 2 THEN 'SHIP' WHEN 3 THEN 'DONE' ELSE 'CANC' END,
      ROUND(DBMS_RANDOM.VALUE(10, 99999), 2),
      CASE WHEN MOD(i,137)=0 THEN 'gift' END
    );
  END LOOP;
  COMMIT;
END;
/

-- 조회 패턴용 보조 인덱스
CREATE INDEX ix_orders_cdt ON ra_orders(cust_id, order_dt);

-- 통계
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'RA_ORDERS', cascade => TRUE,
    method_opt => 'for all columns size skewonly');
END;
/

ALTER SESSION SET statistics_level = ALL; -- ALLSTATS LAST를 위해
```

현실 감각:

- 고객 10만 명 중 각자 수 건~수십 건의 주문
- 최근 1년+α 정도의 날짜 분포
- 상태 코드 5종
- PK는 **기계적으로 증가하는 surrogate key** (`order_id`)

실무에서 이 스키마는 흔히 있는 “주문/로그/트랜잭션” 테이블과 유사하다.

---

## 3. 랜덤 Access 최소화 전략: 3단계 큰 그림

이 글은 다음 3단계를 반복적으로 활용한다.

1. **커버링 인덱스**로 “테이블 미방문(Index-Only)”  
2. **정렬 일치/Stopkey**로 “앞부분만 읽고 끝내기”  
3. **클러스터링 팩터(CF) 개선**으로 “랜덤 → 준순차화”

각 단계에서:

- **핫 쿼리**를 잡고(실제 업무에서 많이 쓰이고 느리면 곤란한 것),
- 해당 쿼리의 **WHERE / ORDER BY / JOIN** 패턴을 보고,
- (1) **커버링 설계**, (2) 정렬 일치/Stopkey, (3) CF 재배치/재적재를 적용한다.

이제 각 단계별로 실습과 함께 본다.

---

## 4. 1단계 — 인덱스 컬럼 추가로 테이블 방문 제거 (Index-Only / 커버링)

### 4.1 베이스라인: 테이블 랜덤 방문이 발생하는 예

핫 쿼리 가정:

```sql
-- 고객 12345의 최근 30일 주문 목록(금액, 상태까지 필요)
SELECT order_id, order_dt, status, amount
FROM   ra_orders
WHERE  cust_id = 12345
AND    order_dt >= SYSDATE - 30
ORDER  BY order_dt, order_id;
```

현재 인덱스:

```sql
CREATE INDEX ix_orders_cdt ON ra_orders(cust_id, order_dt);
```

실행 후:

```sql
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE'));
```

전형적인 계획(개략):

```text
-----------------------------------------------------------------------------------------
| Id  | Operation                           | Name            | Buffers | Reads | Rows  |
-----------------------------------------------------------------------------------------
|*  1 |  SORT ORDER BY                      |                 |    1200 |  800  |   200 |
|*  2 |   TABLE ACCESS BY INDEX ROWID BATCHED| RA_ORDERS      |    1200 |  800  |   200 |
|*  3 |    INDEX RANGE SCAN                 | IX_ORDERS_CDT   |     150 |  100  |   200 |
-----------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
  2 - filter("STATUS" IS NOT NULL) -- 기타 필터
  3 - access("CUST_ID"=TO_NUMBER(:CID) AND "ORDER_DT">=SYSDATE@!)
```

여기서 **병목 후보**:

- `INDEX RANGE SCAN` → 비교적 저렴
- `TABLE ACCESS BY INDEX ROWID BATCHED` → **랜덤 I/O** 주범
- `SORT ORDER BY` → 추가 CPU/TEMP 비용

핵심은 이 부분이다.

1. 인덱스에 `status`/`amount`가 없어서,  
   **각 인덱스 엔트리마다 테이블 블록을 또 읽어야** 한다.
2. 게다가 테이블 물리 순서와 인덱스 순서가 잘 맞지 않으면,  
   **같은 고객이라도 테이블 블록을 여기저기 튕겨 다니게 된다.**

### 4.2 커버링 인덱스로 개선 — 설계 의도

**목표**:

- SELECT-LIST(`order_id, order_dt, status, amount`)와 정렬 키(`order_dt, order_id`)를  
  **인덱스로 전부 충족**시키고,
- ORDER BY까지 인덱스 순서와 일치시켜 **정렬 제거**를 노린다.

설계안:

```sql
-- 커버링 인덱스: (cust_id, order_dt, order_id, status, amount)
-- (Oracle은 INCLUDE 절이 없으므로 "말미에 컬럼 추가"로 구현)
CREATE INDEX ix_orders_cdt_cov
  ON ra_orders(cust_id, order_dt, order_id, status, amount);

BEGIN
  DBMS_STATS.GATHER_INDEX_STATS(USER, 'IX_ORDERS_CDT_COV');
END;
/
```

다시 실행:

```sql
SELECT /* 커버링 기대: 테이블 미방문 */
       order_id, order_dt, status, amount
FROM   ra_orders
WHERE  cust_id = 12345
AND    order_dt >= SYSDATE - 30
ORDER  BY order_dt, order_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE'));
```

예상 계획(개략):

```text
-----------------------------------------------------------------------------------------
| Id  | Operation             | Name              | Buffers | Reads | Rows  |
-----------------------------------------------------------------------------------------
|   1 |  INDEX RANGE SCAN     | IX_ORDERS_CDT_COV |     150 |  100  |   200 |
-----------------------------------------------------------------------------------------
Predicate Information:
  1 - access("CUST_ID"=TO_NUMBER(:CID) AND "ORDER_DT">=SYSDATE@!)
```

변화:

- `TABLE ACCESS BY INDEX ROWID` 가 **사라졌다** → **테이블 랜덤 I/O = 0**
- `SORT ORDER BY`가 사라질 가능성이 높다(인덱스 순서와 정렬이 일치)

즉, \(N_{\text{tab}}\)가 **0** 또는 극소량으로 줄어든다.

### 4.3 커버링 인덱스의 비용·효과 정리

효과:

- **읽기 쿼리 응답시간 급감**
- 버퍼 캐시 입장에선 **인덱스 블록만 읽으면 되므로** 캐시 효율↑
- NL 조인의 **내부 테이블**에서 커버링이 되면  
  **조인 전체 응답시간이 극적으로 감소** 가능

비용:

- 인덱스 폭/크기 증가 → DML 시 업데이트/인서트/삭제 비용 증가
- 저장공간 증가
- 너무 많이 만들면 **변경 경합·락 경합**이 증가

원칙:

- **모든 쿼리**를 커버링으로 만들 수는 없다.
- **핫 경로(자주 호출 + 느리면 곤란)**에 **선별적으로** 적용하라.

---

## 5. 2단계 — 정렬까지 인덱스로 해결 (Stopkey/Keyset Pagination)

### 5.1 최신 N건 조회 — Stopkey 패턴

대표적인 패턴:

```sql
-- 최신 50건(특정 고객)
SELECT /* Stopkey로 앞부분만 읽고 끝내기 */
       order_id, order_dt, amount, status
FROM   ra_orders
WHERE  cust_id = :cid
ORDER  BY order_dt DESC, order_id DESC
FETCH FIRST 50 ROWS ONLY;
```

여기서 핵심:

- 정렬 열이 `order_dt DESC, order_id DESC`
- WHERE는 `cust_id = :cid` (등치)

이 쿼리를 위해 인덱스를:

```sql
CREATE INDEX ix_orders_cdt_desc_cov
  ON ra_orders(cust_id, order_dt DESC, order_id DESC, status, amount);
```

로 정의하면,

1. `cust_id = :cid` 로 **짧은 구간**을 잡고
2. 그 안에서 `order_dt DESC, order_id DESC` 순으로 이미 정렬되어 있으므로
3. 옵티마이저는 **정렬을 생략**하고, **앞 부분만 읽고 Stopkey로 멈출 수 있다.**

실제 실행:

```sql
SELECT /*+ index_desc(ra_orders ix_orders_cdt_desc_cov) */
       order_id, order_dt, amount, status
FROM   ra_orders
WHERE  cust_id = :cid
ORDER  BY order_dt DESC, order_id DESC
FETCH FIRST 50 ROWS ONLY;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE'));
```

계획(개략):

```text
-----------------------------------------------------------------------------------------
| Id  | Operation             | Name                    | Buffers | Rows | Pstart/Pstop |
-----------------------------------------------------------------------------------------
|   1 |  COUNT STOPKEY        |                         |      10 |   50 |              |
|*  2 |   INDEX RANGE SCAN    | IX_ORDERS_CDT_DESC_COV  |      10 |   50 |              |
-----------------------------------------------------------------------------------------
Predicate Information:
  2 - access("CUST_ID"=TO_NUMBER(:CID))
```

여기서 중요한 점:

- `COUNT STOPKEY` 연산자가 나타나면 **앞에서부터 N행만 읽고 멈추는** 계획이다.
- 랜덤 I/O는 **앞 몇 개의 Leaf 블록**으로 한정된다.

즉, `FETCH FIRST N ROWS ONLY` + 정렬 일치 인덱스를 잘 설계하면  
**“필요한 만큼만 읽고 멈추는”** 구조가 된다.

### 5.2 Keyset Pagination vs OFFSET

페이징을 다음처럼 작성하는 경우가 많다.

```sql
-- 나쁜 예: OFFSET 기반 페이징
SELECT order_id, order_dt, amount, status
FROM   ra_orders
WHERE  cust_id = :cid
ORDER  BY order_dt DESC, order_id DESC
OFFSET :offset ROWS FETCH NEXT :page_size ROWS ONLY;
```

문제:

- OFFSET은 **앞 페이지 행들을 모두 읽고 버린 뒤** 그 다음 페이지를 반환한다.
- 페이지가 뒤로 갈수록, 읽힌 총 행 수 = `offset + page_size` → 랜덤 I/O 누적.

반면 Keyset Paging(커서 기반)으로 바꾸면:

```sql
-- Keyset Pagination (커서 기반)
-- :last_dt, :last_id 는 이전 페이지의 마지막 행
SELECT order_id, order_dt, amount, status
FROM   ra_orders
WHERE  cust_id = :cid
AND   (order_dt < :last_dt
       OR (order_dt = :last_dt AND order_id < :last_id))
ORDER  BY order_dt DESC, order_id DESC
FETCH FIRST :page_size ROWS ONLY;
```

이 패턴은 “**지난 페이지의 마지막 키 이후**의 행만” 가져오므로

- 매 페이지마다 **앞에서부터 다시 읽지 않는다.**
- 정렬 일치 인덱스 + Stopkey 조합에서 **최소한의 블록만 추가로 읽는다.**

즉, Keyset Pagination은 **I/O와 CPU 모두 선형 증가**가 아니라  
“페이지당 일정”에 가깝게 만들기 위한 핵심 패턴이다.

---

## 6. 3단계 — 클러스터링 팩터(CF)와 물리 재배치

### 6.1 CF의 정의와 해석

Oracle의 `CLUSTERING_FACTOR`:

- 인덱스 키 순서대로 테이블 행을 방문할 때,
- **테이블 블록을 얼마나 자주 갈아타는지**를 근사한 값
- 이상적인 경우: CF ≈ **테이블 블록 수**
- 최악의 경우: CF ≈ **테이블 행 수** (행마다 블록이 다른 느낌)

조회:

```sql
SELECT index_name, clustering_factor, leaf_blocks, distinct_keys
FROM   user_indexes
WHERE  table_name = 'RA_ORDERS';
```

감각:

- 예를 들어 테이블에 10,000 블록이 있고,
  - CF가 12,000이면 “꽤 좋음”
  - CF가 450,000이면 “인덱스 순서대로 읽어도 테이블 블록을 계속 튕긴다”

CF가 나쁘면,

- 인덱스 Range Scan → 이어지는 테이블 BY ROWID에서  
  **랜덤 I/O가 많이 발생**하게 된다.

### 6.2 컬럼 추가가 CF에 미치는 영향

중요 포인트:

- 인덱스 순서는 `(선행1, 선행2, …, 후행N)`  
- **선행1**이 인덱스 클러스터링의 핵심을 결정한다.
- 후행 컬럼을 추가하면
  - Leaf 엔트리 정렬 순서가 세밀하게 바뀌고
  - 이로 인해 **테이블 블록 접근 순서**가 달라질 수 있다.

실험:

```sql
-- 기준 인덱스: (cust_id, order_dt)
CREATE INDEX ix_cdt_base ON ra_orders(cust_id, order_dt);

-- 변형1: (cust_id, order_dt, order_id)   -- 물리 삽입 순서와 상관성↑ 가정
CREATE INDEX ix_cdt_oid  ON ra_orders(cust_id, order_dt, order_id);

-- 변형2: (cust_id, order_dt, status)     -- 상태는 물리 순서와 상관 낮을 수 있음
CREATE INDEX ix_cdt_st   ON ra_orders(cust_id, order_dt, status);

SELECT index_name, clustering_factor
FROM   user_indexes
WHERE  table_name='RA_ORDERS'
  AND  index_name IN ('IX_CDT_BASE','IX_CDT_OID','IX_CDT_ST');
```

관찰:

- `IX_CDT_OID`의 CF가 `IX_CDT_BASE`보다 **개선될 수도** 있다.  
  (데이터 로딩 방식상 `order_id`가 삽입 순서와 가깝기 때문)
- `IX_CDT_ST`는 상태 코드가 뒤섞여 있어 CF가 **악화될 수도** 있다.

이 말은 “**컬럼을 뒤에 살짝 더하면 CF는 안 바뀐다**”가 아니라,  
“**후행 컬럼도 CF에 영향을 줄 수 있으니 항상 측정해야 한다**”에 가깝다.

### 6.3 물리 재배치(CTAS)로 CF 구조적 개선

가장 강력한 방법:

1. **원하는 인덱스 키 순서**대로 테이블을 정렬해서 다시 만든다.
2. 다시 인덱스를 생성한다.

예:

```sql
-- 1) 정렬된 테이블 생성
CREATE TABLE ra_orders_sorted NOLOGGING AS
SELECT *
FROM   ra_orders
ORDER  BY cust_id, order_dt, order_id;

-- 2) 테이블 교체(테스트 환경 기준 절차)
ALTER TABLE ra_orders RENAME TO ra_orders_old;
ALTER TABLE ra_orders_sorted RENAME TO ra_orders;

-- 3) 인덱스 재생성
DROP INDEX ix_orders_cdt;
CREATE INDEX ix_orders_cdt ON ra_orders(cust_id, order_dt, order_id);

-- 4) 통계 수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'RA_ORDERS', cascade=>TRUE);
END;
/

-- 5) CF 비교
SELECT index_name, clustering_factor
FROM   user_indexes
WHERE  table_name='RA_ORDERS';
```

효과:

- `cust_id, order_dt, order_id` 순으로 **물리 저장**이 이루어졌으므로
- 동일 인덱스 키 순서로 스캔할 때
  - **같은 블록에서 많은 행**을 처리할 수 있고
  - 랜덤 I/O가 **준순차 I/O에 가까운 비용**으로 떨어진다.

운영에서는:

- 파티션 단위로 `MOVE PARTITION` + `UPDATE INDEXES` 사용
- 주기적으로 **저피크 시간대**에 재적재/리빌드 전략 수립

---

## 7. PK 인덱스와 컬럼 추가 — 무엇이 가능하고, 어떻게 설계해야 하는가

### 7.1 PK 인덱스는 “PK 제약과 동일한 컬럼 집합”이어야 한다

Oracle에서:

- PK 제약은 특정 인덱스를 사용한다.
- 그 인덱스의 컬럼 집합은 **PK 제약의 컬럼 집합과 동일**해야 한다.

따라서, 다음은 불가능하다.

- “PK 제약은 `(order_id)` 그대로 두고, PK 인덱스에만 `order_dt`를 뒤에 추가하자”

그렇게 하려면:

- **PK 제약 자체를 `(order_id, order_dt)`로 재정의**해야 한다.

이는 대부분 시스템에서:

- 모든 FK, 조인, 비즈니스 규칙에 파급이 가므로
- **쉽게 시도하면 안 되는** 작업이다.

### 7.2 현실적인 선택지

현실적인 튜닝 순서는 다음과 같다.

1. **PK는 그대로 유지**하고,
2. **별도 커버링 인덱스**를 만든다.

```sql
-- 가장 현실적인 방법
CREATE INDEX ix_orders_pk_cover
  ON ra_orders(order_id, order_dt, status, amount);
```

또는, PK 기반 조인에서 커버링을 활용하게 하려면 **유니크 인덱스**로 만들 수도 있다.

```sql
CREATE UNIQUE INDEX ux_orders_pk_cover
  ON ra_orders(order_id, order_dt, status);
```

옵티마이저는 상황에 따라

- 기존 PK 인덱스 대신 이 유니크 인덱스를 사용하며,
- PK 기반 조인에서 **추가 컬럼까지 커버링** 되면 테이블 방문이 줄어든다.

### 7.3 PK 재정의는 언제 고려할 수 있는가

매우 제한된 케이스:

- **읽기 전용** 또는 거의 읽기 전용 시스템  
  (예: 과거 데이터 Archive, DW Fact 테이블)
- 비즈니스적으로 “**새로운 PK**”를 도입해도 무방하거나,
- 기존 PK가 이미 **의미가 거의 없는 Surrogate**이고  
  앞으로도 이 테이블이 **다른 곳에서 참조되지 않을** 경우

이런 특수한 케이스에서만:

```sql
ALTER TABLE ra_orders DROP CONSTRAINT pk_ra_orders;

ALTER TABLE ra_orders
  ADD CONSTRAINT pk_ra_orders
  PRIMARY KEY (order_id, order_dt, status)
  USING INDEX;
```

같은 방식의 재정의를 신중하게 검토할 수 있다.  
그러나 이 글의 결론은 **“대부분의 경우 PK 변경은 하지 말자”** 이다.

---

## 8. Nested Loops와 테이블 랜덤 Access — 조인까지 확장

지금까지는 단일 테이블 기준으로 봤다.  
실무에서 랜덤 Access가 진짜 위험한 부분은 **조인 안쪽**이다.

### 8.1 Nested Loops 내부 테이블의 랜덤 폭탄

전형적인 패턴:

```sql
-- 외부 테이블: 고객
SELECT /*+ ORDERED USE_NL(o) */
       c.cust_id, c.name, o.order_id, o.amount
FROM   customer c
JOIN   ra_orders o
ON     o.cust_id = c.cust_id
WHERE  c.region = :region
AND    c.valid_yn = 'Y';
```

운이 나쁘면:

- `customer`에서 region 고객이 1만 명,
- 각 고객마다 `ra_orders`에 5건씩 있다면

Nested Loops에서 내부 테이블 `ra_orders`는:

- 인덱스 Range Scan + `TABLE BY ROWID`를 **1만 번** 반복하고,
- 매번 **랜덤 I/O**를 수행하게 된다.

### 8.2 내부 테이블에서의 커버링 인덱스의 위력

만약 `ra_orders`가 다음 인덱스를 가진다면:

```sql
CREATE INDEX ix_orders_c_cov
  ON ra_orders(cust_id, order_dt, order_id, amount);
```

그리고 조인 쿼리가 `order_dt`/`order_id`까지 정확히 필요하다면, 내부 테이블 접근은:

- `INDEX RANGE SCAN`만으로 대부분 처리되고,
- `TABLE BY ROWID`가 크게 줄어든다.

Nested Loops 전체 관점에서:

- “안쪽 루프 한 번당 랜덤 I/O 5번” → “안쪽 루프 한 번당 랜덤 I/O 0~1번”
- 즉, 전체 랜덤 I/O는 **루프 횟수 × 행당 랜덤 I/O**에서  
  “행당 랜덤 I/O”를 거의 0으로 만들어 줄이는 것이 핵심이다.

### 8.3 Hash Join으로 전략 전환

반대로, 내부 테이블 튜닝이 어렵고

- 외부 테이블 결과가 크고,
- 정렬/집계 요구가 있고,
- 랜덤 Access가 이미 심각하다면,

Nested Loops 대신 **Hash Join**이 더 나을 수 있다.

```sql
SELECT /*+ LEADING(c) USE_HASH(o) */
       c.cust_id, c.name, o.order_id, o.amount
FROM   customer c
JOIN   ra_orders o
ON     o.cust_id = c.cust_id
WHERE  c.region = :region
AND    c.valid_yn = 'Y';
```

Hash Join에서는:

- `customer` 결과를 해시 테이블로 만들고,
- `ra_orders`를 스캔하면서 해시 Lookup을 한다.
- 이때 `ra_orders`는 주로 **순차 I/O**에 가까운 방식으로 읽힌다.

랜덤 Access를 줄이는 전략은 결국,

- 내부 테이블을 **커버링 인덱스로 치유**하거나,
- 아예 **조인 전략을 바꿔** 전체 I/O 패턴을 바꾸는 것이다.

---

## 9. 운영 반영 절차 — Invisible 인덱스와 검증 패턴

튜닝을 실제 DB에 반영할 때는 항상 “**그림자 검증 → 가시화**” 패턴을 사용해야 한다.

### 9.1 Invisible Index로 후보 설계를 미리 올려 두기

```sql
CREATE INDEX ix_orders_cdt_cov_inv
  ON ra_orders(cust_id, order_dt, order_id, status, amount) INVISIBLE;
```

- 이 인덱스는 **디폴트로는 옵티마이저가 사용하지 않는다.**

테스트 세션에서만:

```sql
ALTER SESSION SET optimizer_use_invisible_indexes = TRUE;

SELECT order_id, order_dt, status, amount
FROM ra_orders
WHERE cust_id = :cid AND order_dt >= SYSDATE-30
ORDER BY order_dt, order_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE'));
```

이제:

- 후보 인덱스가 실제로 붙는지,
- `Buffers/Reads`가 기존 대비 얼마나 줄었는지,
- 실행 계획이 의도대로 나오는지

를 **실 데이터로** 검증할 수 있다.

### 9.2 검증 후 Visible 전환

검증이 끝나고 안전하다고 판단되면:

```sql
ALTER INDEX ix_orders_cdt_cov_inv VISIBLE;
```

그리고 필요하다면 기존 인덱스를 제거한다.

이 패턴의 장점:

- 운영 세션에는 영향 없이,
- 특정 세션에서만 인덱스 효과를 **A/B 테스트**할 수 있다.

---

## 10. 종합 실험 템플릿 — 전/후 비교 스크립트

다음은 여러분이 자신의 환경에서 그대로 변형해 사용할 수 있는 틀이다.

```sql
ALTER SESSION SET statistics_level = ALL;

--------------------------------------------------------------------------------
-- A. 베이스라인
--------------------------------------------------------------------------------
SELECT /*A*/ order_id, order_dt, status, amount
FROM ra_orders
WHERE cust_id = 12345
  AND order_dt >= SYSDATE - 30
ORDER BY order_dt, order_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE'));

--------------------------------------------------------------------------------
-- B. 커버링 인덱스 적용 후 (예: ix_orders_cdt_cov)
--------------------------------------------------------------------------------
SELECT /*B*/ order_id, order_dt, status, amount
FROM ra_orders
WHERE cust_id = 12345
  AND order_dt >= SYSDATE - 30
ORDER BY order_dt, order_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE'));

--------------------------------------------------------------------------------
-- C. DESC + Stopkey (최신 50건)
--------------------------------------------------------------------------------
SELECT /*C*/ order_id, order_dt, status, amount
FROM ra_orders
WHERE cust_id = 12345
ORDER BY order_dt DESC, order_id DESC
FETCH FIRST 50 ROWS ONLY;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE'));

--------------------------------------------------------------------------------
-- D. CF 비교
--------------------------------------------------------------------------------
SELECT index_name, clustering_factor, leaf_blocks
FROM   user_indexes
WHERE  table_name='RA_ORDERS';
```

관찰 포인트:

- `TABLE ACCESS BY INDEX ROWID`의 유/무
- `Buffers` / `Reads` 변화
- `SORT ORDER BY` 유/무
- CF 변화 및 조인 전략 변화

---

## 11. 자주 묻는 질문(FAQ) — 조금 더 깊게

### Q1. “PK 인덱스에 컬럼만 ‘살짝’ 더하면 안 되나요?”

A. Oracle에서는 **안 된다.**

- PK 인덱스는 **PK 제약과 동일한 컬럼 집합**을 가져야 한다.
- 컬럼을 더하고 싶다면 PK 제약 자체를 바꾸어야 한다.
- 대부분의 OLTP 시스템에서는 FK·조인·중복 검증 등 많은 부분에 영향을 주므로  
  **PK 자체 변경은 최후의 수단**으로만 고려해야 한다.

현실적으로는:

- **PK는 건드리지 않고**,  
- 별도의 **커버링 인덱스** 또는 **유니크 인덱스**를 만든다.

### Q2. 커버링 인덱스를 많이 만들면 왜 안 좋나요?

A. 이유는 세 가지다.

1. **DML 비용 증가**  
   - INSERT/UPDATE/DELETE 때마다 업데이트해야 할 인덱스가 늘어난다.
2. **공간·캐시 효율 저하**  
   - 인덱스가 비대해지면 버퍼 캐시 활용에도 부담.
3. **경합 증가**  
   - 인덱스 블록에 대한 **람/락 경합**이 늘어나 동시성 저하.

따라서 **핵심 지표**(QPS, SLA)에 결정적 영향을 미치는 **핫 쿼리 중심**으로  
커버링 인덱스를 최소한으로 설계해야 한다.

### Q3. 컬럼을 뒤에만 살짝 더하면 CF는 안 바뀌나요?

A. **바뀔 수 있다.**

- 인덱스 엔트리 정렬 순서가 달라지면,
- 테이블 블록 접근 순서도 바뀌고,
- 그에 따라 **CLUSTERING_FACTOR가 좋아지거나 나빠질 수 있다.**

그래서 “컬럼만 살짝 추가했으니 CF는 그대로겠지”라고 생각하지 말고,

```sql
SELECT index_name, clustering_factor
FROM   user_indexes
WHERE  table_name='RA_ORDERS';
```

로 실제 값을 확인해야 한다.

### Q4. Oracle에는 INCLUDE(비키 열) 같은 게 없나요?

A. **B-tree 인덱스에는 없다.**

- 일부 RDBMS는 `INCLUDE` 절로 “인덱스 키에는 포함하지 않지만 리프에 같이 저장하는”  
  비키 열을 지원한다.
- Oracle B-tree에서는 그런 개념이 없고,
  **키 말미에 컬럼을 추가**하는 방식으로 커버링을 구현해야 한다.

결과적으로:

- 인덱스 키 자체가 넓어진다.
- 조인/정렬에서 인덱스 사용 여부에 미묘한 영향을 줄 수 있다.

### Q5. Batched BY ROWID는 강제할 수 있나요?

A. 옵티마이저가 조건/통계에 따라 판단한다.

- 우리는 **커버링**과 **CF 개선**으로,  
  “Batched BY ROWID”가 선택되더라도 **읽어야 할 블록 자체가 적은 상황**을 만드는 것이 핵심이다.
- Batched가 붙는지 여부는 말 그대로 “최적화” 차원이며,
  근본 해법은 **테이블 방문 횟수 자체를 줄이는 것**이다.

---

## 12. 체크리스트 — 실무 적용 시 점검 사항

- [ ] **핫 경로 쿼리**의 SELECT-LIST가 **인덱스만으로 충족**되게 설계했는가?  
      (가능한 경우, 커버링 인덱스를 고려했는가?)
- [ ] 최신 N건·Top-N 쿼리에 대해 **정렬 일치 인덱스(ASC/DESC)** + **Stopkey** 패턴을 적용했는가?
- [ ] 페이징에서 **OFFSET** 대신 **Keyset Pagination**을 사용하도록 API/쿼리 설계가 되어 있는가?
- [ ] PK 변경을 시도하기 전에, **별도 커버링 인덱스**로 해결 가능한지 충분히 검토했는가?
- [ ] 인덱스 컬럼 추가 후, `USER_INDEXES.CLUSTERING_FACTOR`를 확인해 CF가 어떻게 변했는가?
- [ ] Nested Loops 내부 테이블에서 **랜덤 I/O 폭탄**이 발생하는지 확인했고,  
      필요 시 커버링 인덱스 또는 Hash Join으로 전략을 바꾸었는가?
- [ ] Invisible Index를 사용한 **그림자 검증 → 가시화** 절차를 통해  
      운영에 안전하게 반영했는가?
- [ ] 모든 튜닝에서 `DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE')`로  
      **Buffers/Reads/Rows**를 숫자로 비교해 개선 여부를 입증했는가?

---

## 13. 결론

테이블 랜덤 Access 최소화 튜닝은 한마디로 정리하면 다음과 같다.

1. **테이블 방문을 없애라**  
   - 커버링 인덱스, Index-Only Scan으로 \(N_{\text{tab}} \to 0\).
2. **정렬을 인덱스가 맡게 하라**  
   - 정렬 일치 인덱스 + Stopkey/Keyset Pagination.
3. **물리 순서를 맞춰라**  
   - 클러스터링 팩터를 낮추기 위한 CTAS/파티션 MOVE/재적재.

PK 변경 같은 극단적인 선택은 대부분의 시스템에서 위험이 크다.  
대부분의 경우, **별도 커버링 인덱스 + CF 개선 + 조인 전략 변경**만으로도  
랜덤 I/O를 **구조적으로 줄일 수 있다.**

마지막으로, 이 글 전체를 통틀어 가장 중요한 문장은 이것이다.

> 체감이 아니라 **숫자**가 답이다.  
> 튜닝 전/후 실행 계획의 **Buffers/Reads/Rows**를 실제로 비교하고,  
> 랜덤 Access가 얼마나 줄었는지 **수치로 확인**한 뒤에야  
> “튜닝에 성공했다”고 말할 수 있다.