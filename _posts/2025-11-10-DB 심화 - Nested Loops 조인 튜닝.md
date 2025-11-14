---
layout: post
title: DB 심화 - Nested Loops 조인 튜닝, 테이플 prefetch, 배치 i/o, 버퍼 pinning
date: 2025-11-10 16:25:23 +0900
category: DB 심화
---
# Nested Loops 조인 **튜닝 실습**

— **Table Prefetch(ROWID Batched Table Access)** · **Batched I/O(NLJ Batching)** · **Buffer Pinning**까지 전부 뜯어보기 (Oracle 기준)

> 목표: **같은 SQL**이라도 **조인 방법(특히 NL)**과 **I/O 패턴**을 바꾸면 응답시간이 얼마나 달라지는지,
> 실제 **실행계획/통계로 체감**할 수 있도록, **재현 가능한 스크립트**와 함께 설명합니다.
> (실습은 개발/로컬 인스턴스에서 수행하세요. 데이터량은 환경에 따라 줄여도 됩니다.)

---

## 공통 세션 설정

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD HH24:MI:SS';
ALTER SESSION SET statistics_level = ALL;                 -- DBMS_XPLAN ALLSTATS LAST
ALTER SESSION SET optimizer_features_enable = '19.1.0';   -- (환경에 맞게 조정)
```

---

# 실습 데이터 모델 만들기

> 포인트: **Outer(작게) → Inner(인덱스 Lookup)** 패턴을 갖는 전형적인 NL 후보 쿼리를 만들고,
> **ROWID Batched Table Access**(= Table Prefetch) 활성/비활성에 따른 차이와
> **NLJ Batching**(배치 I/O)의 체감을 유도합니다.

## 테이블/데이터 생성

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

-- 데이터 로딩 (규모는 환경에 맞춰 조정)
BEGIN
  FOR c IN 1..50000 LOOP
    INSERT INTO t_cust VALUES (c, CASE MOD(c,3) WHEN 0 THEN 'APAC' WHEN 1 THEN 'EMEA' ELSE 'AMER' END);
  END LOOP;

  FOR o IN 1..300000 LOOP
    INSERT INTO t_ord
    VALUES ( o
           , MOD(o,50000)+1
           , DATE '2024-01-01' + MOD(o, 540)
           , CASE MOD(o,4) WHEN 0 THEN 'NEW' WHEN 1 THEN 'PAID' WHEN 2 THEN 'SHIP' ELSE 'DONE' END );
  END LOOP;

  FOR p IN 1..20000 LOOP
    INSERT INTO t_prod
    VALUES ( p
           , CASE MOD(p,5) WHEN 0 THEN 'ELEC' WHEN 1 THEN 'FASH' WHEN 2 THEN 'FOOD' WHEN 3 THEN 'HOME' ELSE 'TOY' END
           , ROUND(DBMS_RANDOM.VALUE(5, 1000), 2) );
  END LOOP;

  FOR x IN 1..300000 LOOP
    INSERT INTO t_item
    VALUES ( x, 1, MOD(x,20000)+1
           , TRUNC(DBMS_RANDOM.VALUE(1,5))
           , ROUND(DBMS_RANDOM.VALUE(10, 2000), 2) );

    IF MOD(x,3)=0 THEN
      INSERT INTO t_item
      VALUES ( x, 2, MOD(x+7,20000)+1
             , TRUNC(DBMS_RANDOM.VALUE(1,5))
             , ROUND(DBMS_RANDOM.VALUE(10, 2000), 2) );
    END IF;
  END LOOP;

  COMMIT;
END;
/
```

## 인덱스 및 통계

```sql
-- Outer 후보: cust_id + 최신 정렬(Stopkey) 유리
CREATE INDEX ix_ord_c_dt ON t_ord(cust_id, order_dt DESC, order_id DESC);

-- Inner Lookup: (order_id, item_no) PK 이미 존재 (pk_item)
-- 제품 PK
-- 카테고리 탐색용
CREATE INDEX ix_prod_cat ON t_prod(category, price);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'T_CUST', cascade=>TRUE, method_opt=>'for all columns size skewonly');
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'T_ORD' , cascade=>TRUE, method_opt=>'for all columns size skewonly');
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'T_ITEM', cascade=>TRUE, method_opt=>'for all columns size skewonly');
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'T_PROD', cascade=>TRUE, method_opt=>'for all columns size skewonly');
END;
/
```

---

# 베이스라인: 전형적인 NL 조인 쿼리

```sql
VAR cid NUMBER;
EXEC :cid := 12345;

SELECT /* BASELINE */
       o.order_id, o.order_dt, i.item_no, i.amount
FROM   t_ord  o
JOIN   t_item i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 50 ROWS ONLY;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +IOSTATS +MEMSTATS'));
```

**기대 포인트**
- Outer `t_ord`는 `cust_id = :cid` + 정렬 일치 → **앞 블록 몇 개만** 읽고 **Stopkey**로 종료.
- Inner `t_item`은 각 `order_id`에 대해 **PK 인덱스**로 **짧은 Lookup** 반복.
- 실행계획에서 조인 메소드는 보통 **NESTED LOOPS**.
- Inner 쪽에서 **`TABLE ACCESS BY INDEX ROWID`**가 보이면 정상적인 NL Lookup 패턴입니다.

---

# **Table Prefetch**(= ROWID Batched Table Access)

## 개념 정리

- NL에서 Inner는 보통:
  **(1)** 인덱스로 `ROWID`들을 **획득** → **(2)** 각 `ROWID`가 가리키는 **Table Block을 읽음**.
- 이때 **ROWID들을 “한꺼번에” 모아** **Block별로 묶어** 읽으면,
  **랜덤 I/O**를 줄이고 **버퍼 Pin 재사용**도 쉬워집니다.
- Oracle 실행계획에는 다음처럼 표시됩니다.

  - **`TABLE ACCESS BY INDEX ROWID`** (기본)
  - **`TABLE ACCESS BY INDEX ROWID BATCHED`** (배치/프리패치 활성) ← **중요!**

- 힌트/옵션
  - `/*+ BATCH_TABLE_ACCESS_BY_ROWID(t_item) */` : 배치 테이블 액세스 유도
  - `/*+ NO_BATCH_TABLE_ACCESS_BY_ROWID(t_item) */` : 비활성
  - (파라미터/버전/옵티마이저 설정에 의해 자동 선택되기도 함)

> 요점: **“ROWID들을 모아서, 같은 블록끼리 묶어 읽는다.”** → **랜덤 I/O↓, latch/pin 재사용↑**
> **NL 유지**하면서도 Inner의 **물리 I/O 패턴을 더 순차적으로 가깝게** 만들어 준다고 이해하면 편합니다.

## 실습: 활성/비활성 비교

### (A) **강제 비활성화** 후 실행

```sql
SELECT /*+ LEADING(o) USE_NL(i)
           INDEX(o ix_ord_c_dt) INDEX(i pk_item)
           NO_BATCH_TABLE_ACCESS_BY_ROWID(i) */
       o.order_id, o.order_dt, i.item_no, i.amount
FROM   t_ord  o
JOIN   t_item i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 50 ROWS ONLY;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +IOSTATS'));
```

- Inner에 **`TABLE ACCESS BY INDEX ROWID`** (BATCHED가 **아님**)이어야 합니다.
- I/O 통계(특히 Buffers/Reads)를 메모해 둡니다.

### (B) **강제 활성화** 후 실행

```sql
SELECT /*+ LEADING(o) USE_NL(i)
           INDEX(o ix_ord_c_dt) INDEX(i pk_item)
           BATCH_TABLE_ACCESS_BY_ROWID(i) */
       o.order_id, o.order_dt, i.item_no, i.amount
FROM   t_ord  o
JOIN   t_item i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 50 ROWS ONLY;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +IOSTATS'));
```

- Inner에 **`TABLE ACCESS BY INDEX ROWID BATCHED`**가 보이면 성공.
- **Buffers/Reads**가 이전(A) 대비 **감소**했다면, **ROWID 묶음(프리패치/배치)** 덕분에 **랜덤 I/O → 더 적은 I/O**로 전환된 것입니다.

> 체감 포인트
> - Outer Rows(= 50개 주문) × 각 주문의 아이템 수(평균 1~2개) 만큼 Inner 접근.
> - Batched가 **같은 블록의 여러 ROWID를 묶어** 읽으면서,
>   **“db file sequential read(단건)”** 위주였던 패턴이 **더 적은 호출 수**로 처리되는 경향을 보입니다.

---

# **Batched I/O (NLJ Batching)**

## 개념

- **NLJ Batching**은 말 그대로 **NL 루프의 Lookup을 묶어서 처리**하는 최적화입니다.
- 핵심 아이디어: Outer에서 뽑힌 조인 키들을 **일정 크기로 모아** Inner 인덱스/테이블 접근을 **배치**로 수행 →
  **캐시 적중/Block 재사용**이 좋아지고, **Latch/PIN** 횟수도 줄어듭니다.

> 용어 구분
> - 위의 **ROWID Batched Table Access**는 **“테이블 블록 방문”**의 배치/프리패치.
> - **NLJ Batching**은 보다 넓은 개념으로 **“Outer 키를 모아 Inner Lookup 자체를 묶는”** 전략.
>   (버전/옵션에 따라 자동 적용/완화되며, 표시가 노출되지 않을 수도 있습니다.)

## 실습: Outer 결과를 **조금 더 크게** 만들어 배치 이득 확인

> 배치의 이득은 **“재사용할 수 있는 블록이 많을수록”** 커집니다. Outer를 50 → 500/1000으로 늘려보세요.

```sql
-- Stopkey를 500으로 확대 (환경 따라 실행시간 주의)
SELECT /*+ LEADING(o) USE_NL(i)
           INDEX(o ix_ord_c_dt) INDEX(i pk_item)
           BATCH_TABLE_ACCESS_BY_ROWID(i) */
       o.order_id, o.order_dt, i.item_no, i.amount
FROM   t_ord  o
JOIN   t_item i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 500 ROWS ONLY;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +IOSTATS'));
```

- 같은 500건이라도 **BATCHED**와 **NO_BATCH** 비교 시 **Buffers/Reads** 차이가 더 커지는 경향을 볼 수 있습니다.
- (환경에 따라 옵티마이저가 조인 메소드를 바꿀 수 있으니, **`USE_NL` 힌트**로 고정하는 걸 권장)

---

# **Buffer Pinning** (버퍼 핀ning) — 왜 Batched가 더 유리한가

## 개념

- Oracle은 블록을 버퍼 캐시에 읽어올 때 **버퍼 헤더 래치(latch)**를 잡고,
  읽는 동안 **버퍼를 “핀(pin)”** 해 둡니다(= 다른 세션이 위험하게 건드리지 못하도록 **고정**).
- 같은 블록에서 **여러 행**을 읽는다면, **핀을 유지한 채** 연속으로 처리할 수 있어 **래치/핀 횟수**가 줄고 **CPU/경합**이 감소합니다.

## NL + Batched가 주는 이득

- **ROWID Batched Table Access**가 **같은 블록의 행들을 묶어서** 방문하도록 만들면,
  한 번 **블록을 Pin**한 상태에서 **여러 행**을 연속 확인 → **Pin 유지 시간 대비 효율↑**,
  블록을 **반복 재진입**하거나 **다시 찾는 비용↓**.
- 결과적으로, `consistent gets`(CR 블록) **증가율**이 **완만**해지고, **“buffer busy”/“read by …”**류 경합 위험도 낮아집니다.
  (물론 데이터/동시성 상황에 따라 차이는 큼)

> 현장 감각
> - Batched가 **같은 블록 재사용** 기회를 늘리므로 **Pin 유지**에 유리 → **Latch/PIN Overhead 감소**
> - 반대로 **산재(랜덤)**된 ROWID로 한 건씩 접근하면 **핀→해제→다시핀**이 반복되어 **오버헤드↑**

---

# NL 튜닝 **레시피**: 실전 힌트/옵션 모음

## 조인 메소드/순서/경로

```sql
/*+ LEADING(o)          */  -- 조인 시작(Outer) 고정
/*+ USE_NL(i)           */  -- Inner는 NL로
/*+ USE_NL(p)           */
/*+ ORDERED             */  -- FROM 순서대로 조인
/*+ INDEX(o ix_ord_c_dt)*/  -- Outer 액세스 경로(정렬 흡수 + Stopkey)
```

## Table Prefetch (ROWID Batched Table Access)

```sql
/*+ BATCH_TABLE_ACCESS_BY_ROWID(i)    */  -- Inner 테이블 방문을 배치/프리패치
/*+ NO_BATCH_TABLE_ACCESS_BY_ROWID(i) */  -- 비활성(비교용)
```

## NLJ Batching(배치 I/O) 관점

- 버전/옵티마이저에 따라 **자동** 적용되며, 별도 표시가 없을 수 있습니다.
- 핵심은 **Outer를 한꺼번에 조금 더 많이 뽑아**(Stopkey를 넉넉히/부분범위처리와 균형)
  Inner에서 **같은 블록 재사용** 기회를 **극대화**하는 것입니다.

## 기타(정렬 흡수/커버링/SARG)

```sql
-- 정렬 흡수: (cust_id, order_dt DESC, order_id DESC)
-- 커버링 인덱스: Inner 테이블 방문 자체를 줄임(SELECT-LIST 포함해 설계; DML 비용/공간은 늘어남)
-- 함수기반/가상컬럼: WHERE에서 함수/형변환을 제거해 SARGability 확보
```

---

# 비교 실습 세트 — “나쁜→좋은” 전환

## Batched 미적용 (비교 기준)

```sql
SELECT /*+ LEADING(o) USE_NL(i)
           INDEX(o ix_ord_c_dt) INDEX(i pk_item)
           NO_BATCH_TABLE_ACCESS_BY_ROWID(i) */
       o.order_id, o.order_dt, i.item_no, i.amount
FROM   t_ord  o
JOIN   t_item i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 300 ROWS ONLY;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +IOSTATS'));
```

## Batched 적용

```sql
SELECT /*+ LEADING(o) USE_NL(i)
           INDEX(o ix_ord_c_dt) INDEX(i pk_item)
           BATCH_TABLE_ACCESS_BY_ROWID(i) */
       o.order_id, o.order_dt, i.item_no, i.amount
FROM   t_ord  o
JOIN   t_item i ON i.order_id = o.order_id
WHERE  o.cust_id = :cid
ORDER  BY o.order_dt DESC, o.order_id DESC
FETCH FIRST 300 ROWS ONLY;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +IOSTATS'));
```

**확인 포인트**
- Inner가 **`TABLE ACCESS BY INDEX ROWID BATCHED`**로 바뀌었는가?
- **Buffers/Reads/Elapsed**가 **감소**했는가?
- 동일 서버/세션 조건에서 **일관되게** 줄어드는지 여러 번 실행해 숫자를 고정하세요.

---

# ‘대량 Outer’ 상황에서의 선택: NL vs Hash

> NL을 고집하면 안 되는 경우도 있습니다. **Outer가 큰데** Inner 인덱스 재사용이 낮고,
> 블록 재사용/Pinning 이득이 적다면 **해시 조인**이 유리합니다.

```sql
-- Outer 범위가 과도하게 커진 예 (해시 조인이 더 빠를 수 있음)
SELECT /*+ USE_HASH(i) */
       o.cust_id, SUM(i.amount)
FROM   t_ord  o
JOIN   t_item i ON i.order_id = o.order_id
WHERE  o.order_dt >= DATE '2024-01-01'
GROUP  BY o.cust_id;
```

**튜닝 원칙**
- **Top-N/페이징/부분범위**: **NL + Batched**
- **대량 집계/광범위 스캔**: **HASH** (혹은 MERGE; 정렬/메모리 상황 따라)

---

# Buffer Pinning을 체감하는 팁

> 직접 “핀 수”를 카운트하긴 어렵지만, **동일 쿼리**에서 Batched 여부에 따라
> **Bufffers(논리 읽기)**, **db file sequential read 호출 수**가 **눈에 띄게 달라지는지** 보세요.

- **Batched 활성** → **같은 블록에서 여러 행**을 처리 →
  **블록 재사용↑**, **핀 유지 시간 대비 처리량↑** → **논리 읽기/물리 I/O** 모두 **낮아지는 경향**
- (세션 대기/이벤트를 더 깊게 본다면, “buffer busy”류 경합이 줄어드는 흐름을 관찰할 수도 있습니다. 다만 동시성/환경 의존적)

---

# 확인/진단 스니펫

```sql
-- 최근 실행계획 중 BATCHED 등장 여부
SELECT sql_id, child_number, plan_hash_value, operation, options, object_name
FROM   v$sql_plan
WHERE  operation LIKE 'TABLE ACCESS%'
AND    options  LIKE '%BATCHED%'
ORDER  BY last_change_time DESC
FETCH FIRST 20 ROWS ONLY;

-- 인덱스/클러스터링 팩터(Inner측 품질)
SELECT index_name, blevel, leaf_blocks, clustering_factor
FROM   user_indexes
WHERE  table_name IN ('T_ITEM','T_ORD');
```

---

# 체크리스트

- [ ] NL이 **맞는 상황**인가? (Outer 작음/Top-N/부분범위, Inner 인덱스 양호)
- [ ] **정렬 흡수**(ORDER BY 일치 인덱스)로 **Stopkey**가 작동하는가?
- [ ] Inner에 **`BATCH_TABLE_ACCESS_BY_ROWID`**가 쓰이고 있는가? (또는 적용해봤는가?)
- [ ] Outer Rows를 **적절히 확대**했을 때(Batch 기회↑) **Buffers/Reads**가 더 좋아지는가?
- [ ] **클러스터링 팩터**가 지나치게 나쁘지 않은가? (랜덤 I/O 폭증 요인)
- [ ] **ALLSTATS LAST**로 **Starts/A-Rows/Buffers/Reads**가 전/후 개선됐는가?
- [ ] Outer가 커지면 **해시 조인**도 실험했는가?

---

## 요약

- **Table Prefetch(ROWID Batched Table Access)**는 NL의 **Inner 테이블 방문을 묶어서** 처리하여
  **랜덤 I/O를 줄이고** **버퍼 Pin 재사용**을 촉진합니다. (계획에 **BATCHED**로 표시)
- **NLJ Batching**은 Outer 키들을 **배치로** 모아 Inner Lookup을 효율화하는 큰 틀의 아이디어이며,
  **Batched Table Access**와 결합하면 **NL 고유의 응답성(Top-N/부분범위)**을 유지하면서도 **I/O를 크게 절감**할 수 있습니다.
- **Buffer Pinning**은 같은 블록을 **핀 고정 상태로 연속 처리**할 수 있게 해주며,
  Batched가 이를 **더 자주** 만들어 주기 때문에 **CPU/래치 오버헤드**까지 줄이는 데 도움이 됩니다.
- 결론: **작은 Outer + 정렬 흡수 + Batched + 적절한 인덱스** → **NL 최적 패턴**.
  늘 **숫자(ALLSTATS LAST)**로 검증하고, **Outer가 커지면 해시**도 시험하세요.
