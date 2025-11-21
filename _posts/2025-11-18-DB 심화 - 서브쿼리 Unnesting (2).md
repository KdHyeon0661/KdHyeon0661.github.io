---
layout: post
title: DB 심화 - 서브쿼리 Unnesting (2)
date: 2025-11-18 17:25:23 +0900
category: DB 심화
---
# 서브쿼리 Unnesting (심화)

## 0) 배경 요약 — 왜 “m쪽 & non-unique 인덱스”가 핵심인가?

- 대부분의 실무 스키마는 `고객(1) : 매출(M)`, `상품(1) : 판매(M)` 같은 **1:M 관계**가 기본.
- 이때 M테이블(사실 테이블)은 **동일 키 값이 다수** → 관련 인덱스는 **non-unique**가 정상.
- **EXISTS/IN** 계열 서브쿼리에선 **“한 건만 존재하면 된다(존재성)”**가 많으므로,
  - 비-언네스트(FILTER)는 **상관 서브쿼리**를 “행마다” 평가(캐시가 도와주긴 하지만 한계).
  - 언네스트(SEMI JOIN)는 **첫 매칭에서 즉시 중단(early-out)** + **해시/블룸/전용 탐색**으로 훨씬 유리.

---

## 1) 준비 스키마 (간단 데이터)

```sql
-- 고객(1) : 매출(M)
CREATE TABLE d_customer(
  cust_id NUMBER PRIMARY KEY,
  region  VARCHAR2(8) NOT NULL,
  tier    VARCHAR2(8) NOT NULL
);

CREATE TABLE f_sales(
  sales_id NUMBER PRIMARY KEY,
  cust_id  NUMBER NOT NULL,
  prod_id  NUMBER NOT NULL,
  sales_dt DATE   NOT NULL,
  qty      NUMBER NOT NULL,
  amount   NUMBER(12,2) NOT NULL
);

-- m쪽(사실) 테이블용 non-unique 인덱스
CREATE INDEX ix_fs_cust_dt ON f_sales(cust_id, sales_dt);   -- non-unique
CREATE INDEX ix_fs_cust_only ON f_sales(cust_id);            -- non-unique(의도적으로)

-- 예시 데이터(요약)
BEGIN
  FOR i IN 1..50000 LOOP
    INSERT INTO d_customer VALUES (i,
      CASE MOD(i,5) WHEN 0 THEN 'KOR' WHEN 1 THEN 'APAC' WHEN 2 THEN 'EMEA' WHEN 3 THEN 'AMER' ELSE 'JPN' END,
      CASE MOD(i,4) WHEN 0 THEN 'VIP' WHEN 1 THEN 'GOLD' WHEN 2 THEN 'SILVER' ELSE 'GEN' END);
  END LOOP;

  FOR s IN 1..800000 LOOP
    INSERT INTO f_sales VALUES (
      s,
      MOD(s,50000)+1,
      MOD(s,20000)+1,
      DATE '2024-03-01' + MOD(s,20),
      1 + MOD(s,3),
      ROUND(DBMS_RANDOM.VALUE(10,500),2)
    );
  END LOOP;
  COMMIT;
END;
/
```

---

## 2) 문제 정의 — “3월에 거래가 ‘존재’하는 고객만”

같은 의미의 질의를 두 가지 방식으로 작성:

### 비-언네스트(FILTER) — 상관 서브쿼리

```sql
-- Q1: FILTER(비-언네스트) 버전
SELECT /*+ qb_name(main) */ c.cust_id
FROM   d_customer c
WHERE  EXISTS (
  SELECT 1
  FROM   f_sales s
  WHERE  s.cust_id  = c.cust_id
  AND    s.sales_dt >= DATE '2024-03-01'
  AND    s.sales_dt <  DATE '2024-04-01'
);
```
- 전형적인 **FILTER 오퍼레이션**이 플랜에 등장(비-언네스트).
- 동작: C의 각 행에 대해 **상관 조건(c.cust_id)**를 바인드로 s를 탐색.
- **서브쿼리 캐싱**(값-결과 캐시)이 **있을 수** 있으나, **중복 키가 많고 분포가 넓으면** 캐시 효율이 떨어짐.
- 인덱스는 **non-unique** → **같은 cust_id에 대한 다수 엔트리**를 찾아가며 **첫 매칭**에서 반환.
  - 그러나 FILTER는 “C의 많은 행 × (반복적인 s조회)” 형태가 됨 → **Probe 횟수**가 많아지기 쉽다.

### 언네스트(SEMI JOIN) — 조인으로 재작성

```sql
-- Q2: Unnesting → SEMI JOIN
SELECT /*+ UNNEST qb_name(main) */
       c.cust_id
FROM   d_customer c
WHERE  EXISTS (
  SELECT 1
  FROM   f_sales s
  WHERE  s.cust_id  = c.cust_id
  AND    s.sales_dt >= DATE '2024-03-01'
  AND    s.sales_dt <  DATE '2024-04-01'
);
```
- 옵티마이저가 **SEMI JOIN**으로 변환 (NESTED LOOPS SEMI 또는 HASH JOIN SEMI).
- 장점: **존재성** 판정은 첫 매칭에서 **즉시 중단(early-out)**, 내부적으로는
  - **NL SEMI**: 바깥 키마다 인덱스 탐색하지만 *첫 행에서 stop* (random I/O 줄어듦).
  - **HASH SEMI**: s의 **조인키(=cust_id) 집합**을 해시 구조로 구축 → **멤버십 테스트**는 O(1) 평균,
    더 나아가 **Join Filter(Bloom)**가 생성되면 **바깥 스캔 단계에서 조기 필터링**까지 가능.

---

## 3) FILTER(비-언네스트)의 작동 & 캐싱 메커니즘

### FILTER 오퍼레이션의 의미

- 실행계획에 `FILTER` 노드가 보이면, 옵티마이저가 서브쿼리를 **조인으로 풀지 않고**
  **바깥 행마다 조건을 평가**한다는 뜻(일반적으로 “상관 서브쿼리” 패턴).

### FILTER의 “서브쿼리 캐싱”

- Oracle은 상관 서브쿼리 평가 시, **최근 평가한 상관 값(key)과 결과(true/false 혹은 소수의 결과 값)**를
  내부적으로 **캐시**하여 **같은 key가 반복되면 재조회 없이 재사용**하기도 한다.
- 하지만 이 캐시는
  - **값 영역이 매우 넓거나**(cust_id 다양),
  - 바깥 집합에서 **동일 key 반복 빈도**가 낮으면,
  - **캐시 히트율이 낮아** 성능 향상 폭이 제한된다.
- 즉, **m쪽/분산 큰 키**에서는 FILTER의 캐시만으론 근본적 이득이 작다.

### FILTER + non-unique 인덱스의 비용상상

- non-unique 인덱스는 **동일 키**의 **리프 체인이 길 수** 있다.
- 상관 바인드로 인덱스 *range*를 잡아도, **첫 행**을 찾기까지 **여러 리프를 더듬는 비용**이 반복된다.
- 키 분포가 넓으면 **블록 캐시 지역성**도 낮아져 **버퍼 미스**/랜덤 I/O가 많아진다.

> 결론: **FILTER 자체는 나쁜 게 아니다.** 소량·중복키 다수·인덱스 적중이 매우 좋은 상황에선
> 간단하고 충분히 빠를 수 있다. 하지만 **대량(M측)/비고유/분산 넓음**이면 **SEMI JOIN 변환**이 보통 우월하다.

---

## 4) SEMI JOIN(언네스트)의 “캐싱/얼리-아웃” 효과

### NL SEMI의 *early-out*

- NL SEMI는 바깥 행마다 인덱스 탐색을 하지만, **첫 매칭을 찾는 순간 종료**한다.
- non-unique 인덱스라도 “첫 리프 매칭”에 도달하기만 하면 **추가 탐색이 불필요**.
- 즉, FILTER와 비교하면 **같은 인덱스 경로라도 “불필요 후속 탐색”을 다수 제거**한다.

### HASH SEMI의 “해시 캐시”

- HASH SEMI는 내부적으로 **Build Input(s)**의 **조인 키 집합**을 해시에 저장 →
  바깥 행은 **멤버십 테스트**만 하면 된다(평균 O(1)).
- s가 매우 큰 M집합이어도, **집합 자체를 해싱**해두면 **반복 Probe 비용**을 상수로 묶는다.
- 여기에 **Join Filter(Bloom)**가 활성화되면, 바깥 스캔 중 **가능성이 낮은 키를 초기에 drop**하여
  **프로브 자체를 줄이는**(I/O 절감) 효과가 추가된다.
  - 실행계획에 `JOIN FILTER CREATE/USE`가 보일 수 있다(버전/상황 의존).

> 실무 감각: **존재성 판정** + **M측 non-unique** + **대량**이면, *HASH SEMI(+Join Filter)*가
> 대체로 가장 안전하고 빠른 “캐시-기반” 전략이다.

---

## 5) 실험 예제 — FILTER vs SEMI JOIN

### FILTER (비-언네스트) 강제

```sql
EXPLAIN PLAN FOR
SELECT /*+ NO_UNNEST qb_name(main) */
       c.cust_id
FROM   d_customer c
WHERE  EXISTS (
  SELECT 1
  FROM   f_sales s
  WHERE  s.cust_id  = c.cust_id
  AND    s.sales_dt >= DATE '2024-03-01'
  AND    s.sales_dt <  DATE '2024-04-01'
);
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```
**예상 플랜(개념):**
```
FILTER
  TABLE ACCESS FULL D_CUSTOMER
  INDEX RANGE SCAN IX_FS_CUST_DT
  TABLE ACCESS BY INDEX ROWID F_SALES
```
- `FILTER` 아래에 s 접근이 붙는다.
- 바깥 C의 각 행마다 상관바인드로 s를 “평가”.

### 언네스트(SEMI JOIN) + 해시 유도

```sql
EXPLAIN PLAN FOR
SELECT /*+ UNNEST USE_HASH(s) LEADING(s c) */
       c.cust_id
FROM   d_customer c
WHERE  EXISTS (
  SELECT 1
  FROM   f_sales s
  WHERE  s.cust_id  = c.cust_id
  AND    s.sales_dt >= DATE '2024-03-01'
  AND    s.sales_dt <  DATE '2024-04-01'
);
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```
**예상 플랜(개념):**
```
HASH JOIN SEMI
  VIEW (혹은 PARTITION RANGE ... ) on F_SALES subset
    TABLE/INDEX ACCESS on F_SALES (조건으로 3월만 프루닝/범위)
    JOIN FILTER CREATE (있을 수 있음)
  TABLE ACCESS FULL D_CUSTOMER
    JOIN FILTER USE (있을 수 있음)
```
- **HASH JOIN SEMI**로 바뀌고, (버전에 따라) **JOIN FILTER**가 동반될 수 있다.
- 의미: s의 3월 cust_id 집합을 **해시 캐시**로 만들고, C 스캔 중에 **멤버십만 검사**.

### 측정 팁 (세션 지표)

```sql
-- 1) 실행 전 지표 스냅샷
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON ms.stat#=sn.stat#
WHERE  sn.name IN ('session logical reads','physical reads','consistent gets');

-- 2) 쿼리 실행(각각 FILTER/SEMI 버전)

-- 3) 실행 후 동일 지표 비교 → FILTER보다 SEMI에서
--    logical/physical reads가 유의미하게 줄어드는지를 관찰.
```

---

## 6) 스칼라 서브쿼리(SELECT-list) — “서브쿼리 캐시” vs “조인+집계 캐시”

### FILTER(스칼라) — 캐시가 있긴 하다

```sql
-- 각 고객의 3월 매출 합(스칼라)
SELECT c.cust_id,
       (SELECT SUM(s.amount)
        FROM   f_sales s
        WHERE  s.cust_id  = c.cust_id
        AND    s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31') AS m_sum
FROM   d_customer c
WHERE  c.region = 'KOR';
```
- Oracle은 스칼라 서브쿼리에 대해 **subquery cache**를 두어,
  **같은 cust_id가 반복**되면 재사용한다(버전/상황 의존).
- 하지만 `cust_id` **분산이 넓고 중복이 적으면** 캐시 히트율이 낮아 **Probe 폭증**.

### 언네스트(조인+집계) — 집합으로 “한 번” 계산

```sql
-- 사전 집계 → 조인 (언네스트 기대)
SELECT /*+ UNNEST */ c.cust_id, ss.m_sum
FROM   d_customer c
LEFT JOIN (
  SELECT s.cust_id, SUM(s.amount) m_sum
  FROM   f_sales s
  WHERE  s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
  GROUP  BY s.cust_id
) ss
ON ss.cust_id = c.cust_id
WHERE  c.region = 'KOR';
```
- 내부에서 **한 번만 집계**하고 **조인**.
- 결과 캐싱이 아닌 **구조적 캐시(해시/배열)**로 set-based 처리 → **대량 호출 제거**.

---

## 7) m쪽/non-unique 인덱스에서의 **조인 방법 선택 가이드**

### NL SEMI가 유리한 경우

- **바깥 집합이 매우 작고**, 해당 키가 s에서 **매우 높은 선택도**(= 적은 매칭)라면
  - 인덱스 탐색 → **첫 매칭에서 stop** → 빠르다.
- 인덱스 **선두 컬럼**이 상관키와 정확히 맞고, **범위 조건**까지 잘 붙는다면 유리.

### HASH SEMI(+Join Filter)가 유리한 경우

- 바깥 집합이 **크거나 전수 스캔**에 가깝고, s가 **매우 크며 분포 넓음**.
- **멤버십 테스트**를 해시에 전가 + **Join Filter**로 바깥 단계에서 **조기 제거**.
- s의 **부분 프루닝**(예: 파티션 by month)이 가능하면 더 강력.

### FILTER(비-언네스트)를 남길 수도 있는 경우

- 바깥 집합이 **극소량**이고, 상관키 **중복이 매우 높아** *subquery cache* 히트율이 탁월한 경우.
- 의미 보존/외부조인 규칙 등으로 **언네스트가 위험할 때**(NULL 보존 등).

---

## 8) “같은 키 반복”을 만들고 **캐시 체감**하기(실험 트릭)

```sql
-- 바깥에서 같은 cust_id를 강제로 여러 번 만들자
WITH base AS (
  SELECT c.cust_id
  FROM   d_customer c
  WHERE  c.region='KOR'
),
rep10 AS ( -- 동일 집합을 10배로 증폭 (키 반복)
  SELECT b1.cust_id FROM base b1
  UNION ALL SELECT b2.cust_id FROM base b1 JOIN base b2 ON b2.cust_id=b1.cust_id WHERE ROWNUM<=0 -- no-op
  UNION ALL SELECT b1.cust_id FROM base b1
  UNION ALL SELECT b1.cust_id FROM base b1
  UNION ALL SELECT b1.cust_id FROM base b1
  UNION ALL SELECT b1.cust_id FROM base b1
  UNION ALL SELECT b1.cust_id FROM base b1
  UNION ALL SELECT b1.cust_id FROM base b1
  UNION ALL SELECT b1.cust_id FROM base b1
  UNION ALL SELECT b1.cust_id FROM base b1
)
SELECT /*+ NO_UNNEST */
       r.cust_id
FROM   rep10 r
WHERE  EXISTS (
  SELECT 1 FROM f_sales s
  WHERE  s.cust_id=r.cust_id
  AND    s.sales_dt>=DATE '2024-03-01' AND s.sales_dt<DATE '2024-04-01'
);
```
- 위처럼 **키 반복**을 만들면 FILTER의 **subquery cache**가 히트해 “그나마” 빨라질 수 있다.
- 하지만 **언네스트 + HASH SEMI(+Join Filter)**가 가능하면, 대개 그보다 더 안정적으로 빠르다.

---

## 9) 비-언네스트 ↔ 언네스트 **전환/제어** 힌트 요약

- 강제 언네스트: `UNNEST`
- 금지: `NO_UNNEST`
- 조인 순서: `LEADING(...)`, `ORDERED`
- 조인 방식: `USE_NL(t)`, `USE_HASH(t)`, `USE_MERGE(t)`
- (옵션) `SWAP_JOIN_INPUTS(t)` : 해시 빌드/프로브 전환으로 메모리/카디널리티 균형

---

## 10) 케이스 스터디 — “같은 의미, 다른 비용”

### FILTER가 비싸지는 상황

```sql
-- 조건이 완만(3월 전수에 가까움) + cust_id 분포 넓음
SELECT /*+ NO_UNNEST */ c.cust_id
FROM   d_customer c
WHERE  EXISTS (SELECT 1 FROM f_sales s
               WHERE s.cust_id=c.cust_id
                 AND s.sales_dt>=DATE '2024-03-01' AND s.sales_dt<DATE '2024-04-01');
```
- **Probe 횟수 = 바깥 행 수**(≈ 고객 수).
- **non-unique 인덱스**에서 **첫 매칭** 찾을 때까지 **리프 체인 탐색** 반복 → 버퍼 미스 ↑.

### HASH SEMI(+Join Filter)로 전환

```sql
SELECT /*+ UNNEST USE_HASH(s) LEADING(s c) */
       c.cust_id
FROM   d_customer c
WHERE  EXISTS (
  SELECT 1 FROM f_sales s
  WHERE  s.cust_id=c.cust_id
    AND  s.sales_dt>=DATE '2024-03-01' AND s.sales_dt<DATE '2024-04-01'
);
```
- s(3월분)에서 **cust_id 집합**을 **해시-캐시**로 만들고,
- C 스캔 시 **멤버십 테스트**만 수행(필요 시 **JOIN FILTER USE**로 조기 컷).
- 보통 **session logical reads/physical reads**가 크게 준다.

---

## 11) 튜닝 체크리스트 (m쪽/비고유 인덱스 전제)

- [ ] **언네스트 가능성** 확인 → `DBMS_XPLAN.DISPLAY_CURSOR('ALLSTATS LAST +NOTE +PREDICATE')`
  - `FILTER`가 보이면 비-언네스트, `... SEMI/ANTI`가 보이면 언네스트 성공.
- [ ] **조인 방식** 유도
  - 소량/고선택도: `USE_NL(s)` + **early-out** 기대
  - 대량/분포 넓음: `USE_HASH(s)` + **해시 캐시/Join Filter**
- [ ] **파티션 프루닝**(월/일 파티션이면 필수)로 **Build 입력 축소**
- [ ] **인덱스 선두 컬럼**이 상관키와 정렬 → NL의 탐색 비용 최소화
- [ ] **히스토그램/통계** 정밀 → 비용 오판 방지(특히 s.sales_dt, s.cust_id, c.region 등)
- [ ] FILTER를 남겨야 한다면, **키 반복률**을 높여 **subquery cache**가 먹히게(가능한 경우에 한함)

---

## 12) 현업 Q&A

**Q1. FILTER와 NL SEMI, 무엇이 근본적으로 다르죠?**
- FILTER는 “바깥 행마다 **서브쿼리 평가**” 모델. 캐시가 있더라도 **Probe 자체**는 일어난다.
- NL SEMI는 “**조인**으로 풀고 **첫 매칭에서 stop**”하는 모델. 같은 non-unique 인덱스라도 **후속 스캔**을 줄인다.

**Q2. HASH SEMI의 캐싱이란?**
- Build 입력(s)의 **키 집합**을 **해시 테이블**에 담아 **멤버십 테스트**를 상수 시간으로 만든다.
- 추가로 **JOIN FILTER**(Bloom)가 있으면 바깥 탐색 단계에서 **프로브 전**에 많은 키를 **차단**(오탐 약간 허용)한다.

**Q3. FILTER의 서브쿼리 캐싱으로 충분할 때는?**
- 바깥에서 **같은 상관키가 자주 반복**될 때(예: 해시/소트로 우연히 같은 키가 뭉쳐진 스캔).
- 그 외엔 언네스트/SEMI가 보통 더 안정적으로 빠르다.

---

## 13) 마무리 — “존재성 판정은 조인이 유리하다”

- **m쪽/비고유 인덱스**에서 **존재성(EXISTS/IN)** 판정은,
  대개 **SEMI JOIN(특히 HASH SEMI + Join Filter)**가 **FILTER**보다 **I/O·CPU** 측면에서 우월하다.
- FILTER의 **subquery cache**는 “좋을 때만 좋은” 최적화다.
- **언네스트 → 조인 방식/순서**를 의도적으로 설계하면, 같은 의미의 쿼리로 **수배~수십배** 개선이 가능하다.

---
### 부록: 실무용 미니 템플릿

```sql
-- 1) 언네스트 + 해시 세미 기본형
SELECT /*+ UNNEST USE_HASH(s) LEADING(s c) */
       c.cust_id
FROM   d_customer c
WHERE  EXISTS (
  SELECT 1 FROM f_sales s
  WHERE  s.cust_id=c.cust_id
    AND  s.sales_dt BETWEEN :d1 AND :d2
);

-- 2) 언네스트 + NL 세미 (바깥 소량/내부 고선택도)
SELECT /*+ UNNEST USE_NL(s) LEADING(c s) */
       c.cust_id
FROM   d_customer c
WHERE  EXISTS (
  SELECT 1 FROM f_sales s
  WHERE  s.cust_id=c.cust_id
    AND  s.sales_dt BETWEEN :d1 AND :d2
);

-- 3) 비-언네스트(FILTER) 비교 실험
SELECT /*+ NO_UNNEST */ c.cust_id
FROM   d_customer c
WHERE  EXISTS (SELECT 1 FROM f_sales s
               WHERE s.cust_id=c.cust_id
                 AND s.sales_dt BETWEEN :d1 AND :d2);
```

> 마지막으로, 항상 **실측 플랜(DBMS_XPLAN.DISPLAY_CURSOR … ALLSTATS LAST)**과
> **E-Rows vs A-Rows**를 확인해, **진짜 I/O/CPU**가 줄었는지 검증하세요. 그것이 정답입니다.
