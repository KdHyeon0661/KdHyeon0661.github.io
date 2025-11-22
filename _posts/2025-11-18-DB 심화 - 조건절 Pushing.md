---
layout: post
title: DB 심화 - 조건절 Pushing
date: 2025-11-18 20:25:23 +0900
category: DB 심화
---
# 조건절 Pushing 완전 가이드

**주제**
- **Predicate Pushdown(조건절 푸시다운)**: 필터를 **더 아래 단계(데이터 원천 가까이)**로 밀어 넣어 읽는 데이터·연산량을 최소화
- **Predicate Pullup(조건절 풀업/호이스트)**: 안전할 때 조건을 **위로 끌어올려** 조인 재배치·변환 공간을 넓힘
- **Join Predicate Pushdown(조인 조건 푸시다운)**: 조인 동등조건/전이성으로 **상대 테이블에 필터를 전파**하거나, **Join Filter(Bloom)**로 스캔 단계에서 조기 차단

> 핵심 직관  
> **CBO는 “의미 보존이 확실한 범위에서, 비용이 줄어드는 방향으로”** 조건을 위아래로 움직인다.  
> - Pushdown은 **I/O↓·행수↓**가 목적  
> - Pullup은 **의미 보호·변환 자유도↑**가 목적  
> - Join Predicate Pushdown은 **상대편/스토리지까지 필터를 전파**하는 상위 최적화 축

---

## 0. 공통 스키마/관찰 루틴

아래 스키마는 “차원-사실”의 전형적인 튜닝 실습용이다. (필요하면 로우를 수백만 단위로 늘려 실측)

```sql
CREATE TABLE d_customer (
  cust_id NUMBER PRIMARY KEY,
  region  VARCHAR2(8) NOT NULL,
  tier    VARCHAR2(8) NOT NULL
);

CREATE TABLE d_product (
  prod_id  NUMBER PRIMARY KEY,
  category VARCHAR2(16) NOT NULL,
  brand    VARCHAR2(16) NOT NULL
);

CREATE TABLE f_sales (
  sales_id NUMBER PRIMARY KEY,
  cust_id  NUMBER NOT NULL,
  prod_id  NUMBER NOT NULL,
  sales_dt DATE   NOT NULL,
  qty      NUMBER NOT NULL,
  amount   NUMBER(12,2) NOT NULL
);

-- 인덱스 예시
CREATE INDEX ix_fs_cust_dt  ON f_sales(cust_id, sales_dt);
CREATE INDEX ix_fs_prod_dt  ON f_sales(prod_id, sales_dt);
CREATE INDEX ix_prod_cat_br ON d_product(category, brand, prod_id);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'D_CUSTOMER');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'D_PRODUCT');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'F_SALES');
END;
/
```

**관찰은 항상 “실측 플랜”으로**:

```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL, NULL,
  'ALLSTATS LAST +PREDICATE +ALIAS +NOTE +PROJECTION'));
```

- `Predicate Information`에서  
  - **access predicate**(인덱스/스캔 입력 단계)  
  - **filter predicate**(상위에서 걸러지는 단계)  
  - **pushed predicate**(푸시된 조건)  
  이 어떻게 배치됐는지를 읽는다.
- `Notes`에는 **Predicate Move-Around / pushdown / unnest / view merge**의 흔적이 종종 남는다.

---

## 1. Predicate Pushdown의 본질

### 1.1 정의
- **Pushdown**은 “필터를 **더 아래(데이터에 가까운) 오퍼레이터**로 옮겨”  
  **읽는 행/블록을 먼저 줄이는 변환**이다.
- 대상 위치:
  1) **인라인 뷰/CTE 내부**  
  2) **서브쿼리(세미조인) 내부**  
  3) **베이스 테이블 액세스 앞(access predicate)**  
  4) **파티션 프루닝 지점**  
  5) **해시 조인의 Join Filter(Bloom)**  
     → 스캔 중에 “어차피 매칭 불가”를 조기 차단 :contentReference[oaicite:0]{index=0}

### 1.2 왜 이득인가
- **I/O 감소**: 불필요한 블록을 읽기 전에 배제
- **카디널리티 감소**: 이후 조인/집계/정렬의 입력이 줄어듦
- **인덱스/파티션 프루닝 촉진**: 조건이 access predicate가 되면 인덱스 범위/파티션 범위를 직접 좁힘
- **Bloom Join Filter와 결합**: 해시 조인 빌드 집합에서 만든 키 집합으로  
  프로브 테이블 스캔을 파티션/블록 단위로 제거 :contentReference[oaicite:1]{index=1}

---

## 2. Pushdown이 일어나는 “수준(Level)” 커리큘럼

### 2.1 Level A — 뷰/인라인뷰 내부로의 Pushdown

#### (A) 비머지·비푸시: 조건이 위에 남는 나쁜 패턴
```sql
SELECT s.sales_id, s.amount
FROM   f_sales s
JOIN  (SELECT p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC' AND p.brand='B0') v
ON     v.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN :d1 AND :d2;
```

- 뷰가 병합되지 않으면 `VIEW v` 노드가 남고  
  `v`의 필터가 `s`의 접근 전에 충분히 활용되지 못할 수 있음.

#### (B) 머지+푸시를 유도: 좋은 패턴
```sql
SELECT /*+ MERGE(v) PUSH_PRED(v) USE_HASH(v) LEADING(v s) */
       s.sales_id, s.amount
FROM   f_sales s
JOIN  (SELECT p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC' AND p.brand='B0') v
ON     v.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN :d1 AND :d2;
```

- `MERGE(v)`: 인라인 뷰를 **평탄화(view merging)**  
- `PUSH_PRED(v)`: v의 조건이 **가능한 아래로 들어가도록 선호**  
  (이 계열 힌트의 의미는 오래 전부터 동일) :contentReference[oaicite:2]{index=2}

**성공 신호**
- `VIEW` 노드가 사라지거나 단순화됨
- `Predicate Information`에서 `pushed predicate`가 더 아래 라인에 보임
- 해시 조인이라면 `JOIN FILTER CREATE/USE`가 생성될 수 있음

---

### 2.2 Level B — 서브쿼리 Pushdown(PUSH_SUBQ)과 Unnesting

#### 2.2.1 왜 “서브쿼리를 먼저 줄이는가”
`IN/EXISTS` 서브쿼리는 **세미조인(SEMijoin)**으로 바뀌며,  
CBO는 “서브쿼리 결과를 먼저 축소할수록 좋다”고 판단하면  
그 평가를 앞으로 당긴다. `PUSH_SUBQ`는 그 결정을 **강화**한다. :contentReference[oaicite:3]{index=3}

```sql
SELECT /*+ PUSH_SUBQ UNNEST USE_HASH(p) LEADING(p s) */
       SUM(s.amount)
FROM   f_sales s
WHERE  s.prod_id IN (
  SELECT p.prod_id
  FROM   d_product p
  WHERE  p.category='ELEC' AND p.brand='B0'
)
AND    s.sales_dt BETWEEN :d1 AND :d2;
```

- `UNNEST`: 서브쿼리를 **조인 형태로 재작성**  
- `PUSH_SUBQ`: 그 조인을 **먼저 계산/축소**하는 방향으로 힌팅

**성공 신호**
- `HASH JOIN SEMI` 또는 `NESTED LOOPS SEMI`
- `d_product`가 **드라이빙(build)**이 되고 `f_sales`가 **프로브(probe)**로 내려감
- `JOIN FILTER`가 생기면 더 강력한 pushdown

---

### 2.3 Level C — 베이스 테이블 Access Predicate Pushdown

#### 2.3.1 access vs filter
- **access predicate**: 인덱스 범위, 파티션 프루닝, 스캔 입력을 **직접 제한**
- **filter predicate**: 이미 읽은 후에 위에서 거르는 조건

CBO는 조건이 인덱스/파티션 키에 “알맞게” 놓일수록 access로 바꿔 아래로 내린다.

**나쁜 예(컬럼 가공으로 푸시 불가)**
```sql
SELECT COUNT(*)
FROM f_sales s
WHERE TO_CHAR(s.sales_dt,'YYYY')='2024';
```

**좋은 예(SARGABLE → access predicate 가능)**
```sql
SELECT COUNT(*)
FROM f_sales s
WHERE s.sales_dt >= DATE '2024-01-01'
  AND s.sales_dt <  DATE '2025-01-01';
```

- 타입 변환/함수 가공이 사라지면 인덱스 범위/파티션 프루닝으로 내려갈 가능성이 열린다.

---

### 2.4 Level D — 파티션 프루닝으로의 Pushdown

`f_sales`가 `sales_dt`로 RANGE 파티션이라 가정하자.

```sql
SELECT /*+ MERGE(v) PUSH_PRED(v) */
       COUNT(*)
FROM   f_sales s
JOIN  (SELECT p.prod_id FROM d_product p WHERE p.brand='B0') v
ON     v.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN :d1 AND :d2;
```

- `MERGE+PUSH_PRED`는 뷰 조건을 풀어서 `s` 접근 단계로 밀어넣고,
- `sales_dt` 기간 조건은 파티션 프루닝을 만들어
  **PSTART/PSTOP**이 좁아져야 한다.

**성공 신호**
- 플랜의 `PARTITION RANGE`에서 `PSTART/PSTOP`이 상수/범위로 좁혀짐
- `Predicate Information`에 기간 조건이 `access`로 내려감

---

### 2.5 Level E — Join Filter(Bloom)까지 내려가는 Pushdown

해시 조인에서는 빌드 입력으로 **Bloom Filter(Join Filter)**를 만들고  
프로브 입력을 스캔할 때 “매칭 불가능한 블록/파티션”을 제거한다.  
플랜에 `JOIN FILTER CREATE/USE`가 뜨면 “스캔 레벨 pushdown”이다. :contentReference[oaicite:4]{index=4}

```sql
SELECT /*+ USE_HASH(p) LEADING(p s) */
       SUM(s.amount)
FROM   f_sales s
JOIN   d_product p
  ON   p.prod_id = s.prod_id
WHERE  p.category='ELEC'
AND    s.sales_dt BETWEEN :d1 AND :d2;
```

**해석**
- `p`(작은 집합)에서 해시 테이블을 만들면서 Join Filter 생성
- `s`(큰 집합) 스캔 중 `SYS_OP_BLOOM_FILTER`가 평가되어 조기 차단 :contentReference[oaicite:5]{index=5}

**실전 팁**
- 빌드 집합이 충분히 작고 키 선택도가 높을수록 Bloom이 강하게 작동
- 통계가 틀리면 CBO가 Bloom을 **과소/과대 사용**할 수 있으므로 정확한 통계가 필수 :contentReference[oaicite:6]{index=6}

---

## 3. Join Predicate Pushdown(전이성/전파)

### 3.1 전이성(Transitivity)의 푸시

```sql
SELECT /*+ USE_NL(s) LEADING(c s) */
       COUNT(*)
FROM   d_customer c
JOIN   f_sales s
  ON   s.cust_id = c.cust_id
WHERE  c.cust_id BETWEEN :lo AND :hi
AND    s.sales_dt BETWEEN :d1 AND :d2;
```

- `c.cust_id BETWEEN :lo AND :hi`가 **s.cust_id에도 전파**되어  
  `ix_fs_cust_dt`의 범위가 줄어드는 것이 목표.
- `Predicate Information`에서
  `access("S"."CUST_ID" BETWEEN :LO AND :HI)`가 보이면 성공.

**전이성 푸시가 잘 되는 조건**
- **등치 조인**(=)  
- 양쪽 키의 **타입이 동일**  
- 한쪽에 **상수/좁은 범위 조건**  
- 통계가 전제와 맞음(카디널리티 과대평가면 전파가 억제될 수 있음)

### 3.2 인덱스 컬럼 구성에 따른 푸시 성공/실패
전이성/푸시가 **인덱스 선두 컬럼 구성**에 의해 갈리는 케이스가 있다.  
예를 들어 조건이 인덱스에 포함되지 않으면 푸시가 늦어질 수 있다. :contentReference[oaicite:7]{index=7}

```sql
-- 인덱스가 (a, b)일 때 b 조건만으로는 access로 못 내려갈 수 있음.
-- 해결: (b, a) 또는 (b) 인덱스를 별도로 두면 전이성/푸시 효과가 커짐.
```

즉, **“푸시다운이 곧 인덱스 설계”**로 이어진다.

---

## 4. Predicate Pullup(호이스트) — 왜 위로 끌어올리나

### 4.1 Pullup의 역할
- Pushdown이 “항상 정답”이면 CBO는 늘 아래로만 내릴 것이다.  
  하지만 실제로는
  1) **의미 보존(특히 외부조인)**  
  2) **다른 변환(뷰 병합, 조인 재배치, 집계 푸시)**의 공간 확보  
  때문에 조건을 **일부 위로 끌어올리는 것이 더 안전/유리할 때가 있다**.

이 전체적인 움직임을 Oracle은 **Predicate Move-Around**라고 부른다.  
`PUSH_PRED / NO_PUSH_PRED` 등은 이 과정의 선호를 바꾸는 힌트다. :contentReference[oaicite:8]{index=8}

### 4.2 외부조인에서 Pullup이 필요한 이유

**안전한 패턴(내부측 필터는 ON절)**
```sql
SELECT c.cust_id, s.amount
FROM   d_customer c
LEFT JOIN f_sales s
  ON   s.cust_id = c.cust_id
 AND  s.sales_dt BETWEEN :d1 AND :d2
WHERE  c.region='USA';
```

**위험한 패턴(내부측 필터를 WHERE에 둠)**
```sql
SELECT c.cust_id, s.amount
FROM   d_customer c
LEFT JOIN f_sales s
  ON   s.cust_id = c.cust_id
WHERE  c.region='USA'
  AND  s.amount > 100;  -- NULL 보존이 깨지며 LEFT→INNER 의미가 됨
```

- CBO는 이 조건을 **아래로 푸시하면 의미가 깨지는지**를 먼저 본다.
- 깨질 위험이 있으면 **Pullup/재배치로 안전한 위치를 찾거나**  
  변환 자체를 포기한다.

---

## 5. Pushdown을 막는 “대표적인 장벽”과 해법

| 장벽 | 왜 막히나 | 해법 |
|---|---|---|
| 외부조인 NULL 보존 | 내부측 필터 푸시가 LEFT 의미를 깨뜨림 | 내부측 조건은 ON로 이동 |
| 분석함수/ROWNUM/CONNECT BY | 순서·상태 의존이라 “미리 필터”가 위험 | 수동 재작성, 단계 분리 |
| 비결정적/부작용 함수 | 실행 시점에 따라 값이 바뀌어 의미 보존 어려움 | 함수 제거/사전 계산 |
| 컬럼 가공/형변환 | access predicate로 바뀌지 못해 늦게 필터 | SARGABLE로 변환 |
| OR 조건 | 한 분기가 못 내려가면 전체가 못 내려감 | `USE_CONCAT`로 OR-EXPAND |
| DISTINCT/UNION/집계 | 중복/집계 의미 때문에 “안전한 범위만” 푸시 | 뷰 병합·사전집계 설계 |

### 5.1 OR-EXPAND로 Pushdown 회복

```sql
SELECT /*+ USE_CONCAT */
       s.*
FROM   f_sales s
WHERE  (s.prod_id IN (SELECT p.prod_id
                      FROM d_product p WHERE p.brand='B0'))
   OR  (s.cust_id IN (SELECT c.cust_id
                      FROM d_customer c WHERE c.tier='VIP'));
```

- `USE_CONCAT`는 OR을 **UNION ALL 분기**로 풀어  
  **각 분기에서 독립적 pushdown**이 발생하도록 만든다.

---

## 6. 힌트로 제어하는 Push/Pull 전략

Oracle은 전통적으로 다음 힌트로 predicate move-around의 방향을 바꾼다.  
(힌트 의미 자체는 버전이 올라가도 안정적으로 유지된다.) :contentReference[oaicite:9]{index=9}

### 6.1 핵심 힌트 세트

- `PUSH_PRED(qb)` / `NO_PUSH_PRED(qb)`  
  : **인라인 뷰/서브쿼리 내부**로 필터를 밀어 넣을지 선호/억제
- `MERGE(qb)` / `NO_MERGE(qb)`  
  : **View Merging(평탄화)** 허용/금지  
  → 병합되면 푸시 공간이 커진다.
- `PUSH_SUBQ` / `NO_PUSH_SUBQ`  
  : 서브쿼리를 **먼저 평가(축소)**하도록 선호/억제
- `PUSH_JOIN_PRED` / `NO_PUSH_JOIN_PRED`  
  : **조인 조건·전이성 푸시** 선호/억제
- (병렬/해시) `PX_JOIN_FILTER(alias)`  
  : Join Filter(Bloom) 생성을 **강제**하는 계열 :contentReference[oaicite:10]{index=10}

### 6.2 힌트 조합의 실전 패턴

#### (A) “차원 먼저 축소 → 사실 스캔 최소화”
```sql
SELECT /*+ PUSH_SUBQ UNNEST LEADING(p s) USE_HASH(p) */
       SUM(s.amount)
FROM   f_sales s
WHERE  s.prod_id IN (
  SELECT p.prod_id
  FROM d_product p
  WHERE p.category='ELEC'
)
AND    s.sales_dt BETWEEN :d1 AND :d2;
```

#### (B) “뷰 평탄화 후, 조건을 더 아래로”
```sql
SELECT /*+ MERGE(v) PUSH_PRED(v) */
       COUNT(*)
FROM   f_sales s
JOIN  (SELECT prod_id
       FROM d_product
       WHERE brand='B0') v
ON v.prod_id = s.prod_id
WHERE s.sales_dt BETWEEN :d1 AND :d2;
```

#### (C) “전이성을 적극 활용해 인덱스 범위를 줄이기”
```sql
SELECT /*+ PUSH_JOIN_PRED USE_NL(s) LEADING(c s) */
       COUNT(*)
FROM   d_customer c
JOIN   f_sales s ON s.cust_id = c.cust_id
WHERE  c.cust_id BETWEEN :lo AND :hi
AND    s.sales_dt BETWEEN :d1 AND :d2;
```

---

## 7. 실전 시나리오로 끝까지 읽기

### 7.1 시나리오 1 — 뷰 푸시다운 유/무 비교

```sql
-- (1) NO_MERGE/NO_PUSH
SELECT /*+ NO_MERGE(v) NO_PUSH_PRED(v) */
       COUNT(*)
FROM   f_sales s
JOIN  (SELECT p.prod_id
       FROM d_product p
       WHERE p.category='ELEC' AND p.brand='B0') v
ON v.prod_id = s.prod_id
WHERE s.sales_dt BETWEEN :d1 AND :d2;

-- (2) MERGE/PUSH
SELECT /*+ MERGE(v) PUSH_PRED(v) */
       COUNT(*)
FROM   f_sales s
JOIN  (SELECT p.prod_id
       FROM d_product p
       WHERE p.category='ELEC' AND p.brand='B0') v
ON v.prod_id = s.prod_id
WHERE s.sales_dt BETWEEN :d1 AND :d2;
```

**판독 체크**
- (2)에서 `VIEW v`가 단순화/소거되는지  
- pushed predicate가 해시 조인 build/probe 단계로 더 내려갔는지  
- A-Rows/버퍼 읽기/물리 읽기가 감소했는지

---

### 7.2 시나리오 2 — Join Filter로 스캔 조기 차단

```sql
SELECT /*+ LEADING(p s) USE_HASH(p) */
       COUNT(*)
FROM   f_sales s
JOIN   d_product p ON p.prod_id=s.prod_id
WHERE  p.category='ELEC';
```

**판독 체크**
- `JOIN FILTER CREATE` / `JOIN FILTER USE` 등장 여부  
- `Predicate Information`에 `SYS_OP_BLOOM_FILTER`가 보이는지  
- 파티션 테이블이면 `PX PARTITION HASH JOIN-FILTER`처럼  
  **Bloom 기반 프루닝이 추가로 내려가는지** :contentReference[oaicite:11]{index=11}

---

### 7.3 시나리오 3 — OR-EXPAND로 푸시 회복

```sql
-- OR 때문에 푸시가 막힐 수 있음
SELECT COUNT(*)
FROM f_sales s
WHERE (s.prod_id IN (SELECT prod_id FROM d_product WHERE brand='B0'))
   OR (s.cust_id IN (SELECT cust_id FROM d_customer WHERE tier='VIP'));

-- OR-EXPAND
SELECT /*+ USE_CONCAT */
       COUNT(*)
FROM f_sales s
WHERE (s.prod_id IN (SELECT prod_id FROM d_product WHERE brand='B0'))
   OR (s.cust_id IN (SELECT cust_id FROM d_customer WHERE tier='VIP'));
```

- OR-EXPAND 후 각 분기에서 `PUSH_SUBQ/UNNEST/전이성`이 개별로 작동하는지 확인.

---

## 8. 카디널리티가 Push/Pull을 결정한다

CBO는 “얼마나 줄어드는가?”를 추정해 움직인다.
그래서 **통계가 틀리면 Pushdown도 틀린다.**

### 8.1 꼭 점검할 통계
- 기본 테이블 통계
- **히스토그램(스큐 컬럼)**  
- **확장 통계(조합 상관관계)**  
- 필요하면 `dynamic_sampling`의 개입 여부

```sql
-- 히스토그램/확장 통계 예
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'F_SALES',
    METHOD_OPT => 'FOR ALL COLUMNS SIZE AUTO'
  );
  DBMS_STATS.CREATE_EXTENDED_STATS(
    USER, 'F_SALES', '(cust_id, sales_dt)'
  );
END;
/
```

**진짜 핵심**
- Pushdown은 **비용 기반**이므로  
  “**카디널리티 추정이 정확해질수록 Pushdown 품질도 좋아진다**.”

---

## 9. 요약 체크리스트

- [ ] 조건은 **SARGABLE**하게(컬럼 가공·형변환 금지)
- [ ] 인라인 뷰/CTE는 **MERGE + PUSH_PRED**로 평탄화/푸시 공간 확보
- [ ] 서브쿼리는 **UNNEST + PUSH_SUBQ**로 “먼저 축소”
- [ ] 전이성(Join Predicate Pushdown)이 **상대편 access predicate로 내려가는지** 확인
- [ ] 해시 조인에서 `JOIN FILTER CREATE/USE` 등장 여부 관찰
- [ ] 외부조인의 내부 필터는 **ON절**에 둬 의미를 지킨다(필요시 Pullup)
- [ ] OR은 `USE_CONCAT`로 분기해 푸시 회복
- [ ] 통계(히스토그램/확장 통계)로 **카디널리티 오판**을 줄인다
- [ ] 모든 판단은 **실측 플랜 + A-Rows + 세션 통계**로 검증

---

## 맺음말

- Predicate Pushdown은 **“읽기 전에 줄이는”** 가장 강력한 1차 최적화다.  
- Predicate Pullup은 **“의미를 지키고 변환 자유도를 확보하는”** 안전장치이자 보완축이다.  
- Join Predicate Pushdown(전이성/Join Filter)은 필터를 **조인 상대/스토리지까지 전파**해  
  스캔 비용 자체를 크게 줄인다. :contentReference[oaicite:12]{index=12}  
- 결국 튜닝의 핵은 하나:  
  **“조건이 어디까지 내려갔는지, 왜 그 위치가 선택됐는지”를 DBMS_XPLAN에서 끝까지 읽는 것.**