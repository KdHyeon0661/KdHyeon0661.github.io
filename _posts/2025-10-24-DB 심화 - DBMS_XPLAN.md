---
layout: post
title: DB 심화 - DBMS_XPLAN
date: 2025-10-24 16:25:23 +0900
category: DB 심화
---
# Oracle **DBMS_XPLAN**

## 0. 핵심 한 장

- **예상 계획**:
  ```sql
  EXPLAIN PLAN FOR <SQL>;
  SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'ALL +PREDICATE +NOTE'));
  ```
- **실제 계획**(캐시된 커서):
  ```sql
  -- 방금 실행한 SQL의 마지막 실행(Child 자동 선택)
  SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'TYPICAL'));
  ```
- **Row Source별 수행 통계**(라인 단위 **A-Rows/Starts/Time/Buffers/TempSpc**):
  ```sql
  SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
      'ALLSTATS LAST +PEEKED_BINDS +PREDICATE +PROJECTION +OUTLINE +NOTE'));
  ```

> 항상 기억: **튜닝 결론은 ‘실제(ALLSTATS)’에서!** `EXPLAIN(예상)`은 방향만 잡는다.

---

## 1. 실습 스키마(샘플 데이터)

```sql
-- 깨끗이
DROP TABLE customers PURGE;
DROP TABLE orders PURGE;

-- 테이블
CREATE TABLE customers(
  id      NUMBER PRIMARY KEY,
  region  VARCHAR2(10),
  grade   NUMBER,
  name    VARCHAR2(100)
);

CREATE TABLE orders(
  order_id    NUMBER PRIMARY KEY,
  customer_id NUMBER NOT NULL REFERENCES customers(id),
  order_date  DATE   NOT NULL,
  amount      NUMBER NOT NULL
);

-- 인덱스
CREATE INDEX ix_orders_cust_date      ON orders(customer_id, order_date);
CREATE INDEX ix_cust_region_grade     ON customers(region, grade);

-- 데이터(개략)
BEGIN
  FOR i IN 1..100000 LOOP
    INSERT INTO customers VALUES(
      i,
      CASE MOD(i,10) WHEN 0 THEN 'APAC' WHEN 1 THEN 'EMEA' ELSE 'AMER' END,
      MOD(i,5),
      'C'||i
    );
  END LOOP;
  COMMIT;

  FOR i IN 1..2000000 LOOP
    INSERT INTO orders VALUES(
      i,
      MOD(i,100000)+1,
      DATE '2024-01-01' + MOD(i,365),
      MOD(i,500)+1
    );
    IF MOD(i,100000)=0 THEN COMMIT; END IF;
  END LOOP;
  COMMIT;
END;
/

-- 통계
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'CUSTOMERS',cascade=>TRUE);
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'ORDERS',cascade=>TRUE);
```

---

## 2. **예상 실행계획 출력** — `EXPLAIN PLAN FOR` + `DBMS_XPLAN.DISPLAY`

### 2.1 기본 사용
```sql
EXPLAIN PLAN FOR
SELECT /* plan_test */ SUM(o.amount)
FROM   orders o
WHERE  o.customer_id = :cid
AND    o.order_date BETWEEN :d1 AND :d2;

-- 최신 explain 결과를 plan_table에서 꺼냄
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'ALL +PREDICATE +ALIAS +PROJECTION +NOTE'));
```

- **DISPLAY** 서명: `DISPLAY(table_name, statement_id, format)`
  - `table_name` = NULL → `PLAN_TABLE`에서 **최근**
  - `format` → `BASIC/TYPICAL/ALL` + `+PREDICATE/+NOTE/...`

### 2.2 출력 해석 포인트
- **Operation**: `INDEX RANGE SCAN`, `TABLE ACCESS BY ROWID`, `HASH JOIN`, `NESTED LOOPS`, `SORT GROUP BY` 등
- **Rows/Bytes/Cost/Time**: **추정치**(실제 아님)
- **Predicate Information**:
  - `access()` = **인덱스 접근 조건**
  - `filter()` = 접근 후 **필터링** 조건
- **Note**: 동적 샘플링, 적응 기능, 스칼라 서브쿼리 변환 등 *옵티마이저 메시지*

> 주의: **예상**은 런타임의 바인드 값/통계 수집 시점/적응 행태 때문에 **실행과 달라질 수 있음**.

---

## 3. **캐싱된 커서의 실제 실행계획 출력** — `DBMS_XPLAN.DISPLAY_CURSOR`

### 3.1 방금 실행한 SQL의 실제 계획
```sql
-- 쿼리 실행(바인드)
VAR cid NUMBER; EXEC :cid := 12345;
VAR d1  DATE;   EXEC :d1  := DATE '2024-06-01';
VAR d2  DATE;   EXEC :d2  := DATE '2024-06-30';

SELECT /* exec1 */ SUM(o.amount)
FROM   orders o
WHERE  o.customer_id = :cid
AND    o.order_date BETWEEN :d1 AND :d2;

-- 실제 실행 계획(Child 자동 추적)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'TYPICAL +PREDICATE +NOTE +PEEKED_BINDS'));
```

**포인트**
- `DISPLAY_CURSOR(sql_id, child_number, format)`
  - `NULL,NULL` → **현재 세션의 마지막 실행 SQL**
  - `+PEEKED_BINDS` : **바인드 피킹 값** 확인(값 스큐/ACS 분석에 중요)
  - 이 시점엔 라인별 통계(A-Rows)는 없음 → 다음 장의 `ALLSTATS` 사용

### 3.2 특정 SQL_ID/Child 지정
```sql
-- 세션에서 SQL_ID/CHILD_NUMBER 확인
SELECT sql_id, child_number
FROM   v$sql
WHERE  sql_text LIKE '%exec1%' AND ROWNUM=1;

-- 명시 조회
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('&&SQL_ID', 0,
  'TYPICAL +PREDICATE +ALIAS +NOTE'));
```

> **Child Cursor** 는 바인드 타입/길이/NLS/환경에 따라 분기. **문제 Child**를 정확히 골라 본다.

---

## 4. **Row Source별 수행 통계** — `ALLSTATS`

> 라인 기준 **A-Rows(실행 행수)**, **Starts**, **E-Rows(추정)**, **Buffers**, **Reads**, **Time**, **TempSpc** 등 **실행 통계**를 표로 제공.
> **성능 분석의 결론**은 거의 항상 여기서 난다.

### 4.1 `ALLSTATS LAST` — 마지막 실행의 실측
```sql
-- 대상 SQL 한 번 더 실행(동일 바인드로 재현)
SELECT /* exec1 */ SUM(o.amount)
FROM   orders o
WHERE  o.customer_id = :cid
AND    o.order_date BETWEEN :d1 AND :d2;

-- 라인별 수행 통계(마지막 실행)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'ALLSTATS LAST +PEEKED_BINDS +PREDICATE +PROJECTION +OUTLINE +NOTE'));
```

**해석 체크리스트**
- **A-Rows vs E-Rows**: **추정 오류**(카디널리티 오판) 여부
- **Buffers/Read**: I/O 비용, 캐시 적중/미스 경향
- **Time(라인별)**: 어디서 시간이 많이 소모되는지
- **TempSpc**: 정렬/해시 *스필* 존재 여부
- **Outline/Note**: 옵티마이저가 선택한 힌트 세트, 적응/변환 메시지

### 4.2 `ALLSTATS FIRST` — 첫 실행 통계
```sql
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'ALLSTATS FIRST +NOTE'));
```
- 캐시 워밍/바인드 재사용에 따라 **첫 실행과 이후**가 다른 경우 비교에 유용

### 4.3 병렬(PX)과 메모리
```sql
-- 병렬 힌트 예
SELECT /*+ parallel(o 8) monitor */ COUNT(*)
FROM   orders o
WHERE  o.order_date BETWEEN :d1 AND :d2;

-- 병렬/메모리 관련 표기 강화
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL,
  'ALLSTATS LAST +PARALLEL +NOTE'));
```
- `PX COORDINATOR/SEND/Q1,Q2` 라인 확인, 각 라인 **Time/Temp**로 **스필/스큐** 감지

---

## 5. 시나리오 실습

### 5.1 **NL vs HASH** 실제 비교 (행수 적중/오판)
```sql
-- 후보 1: NL 유도(드라이빙을 소량으로)
EXPLAIN PLAN FOR
SELECT /* NL */ /*+ USE_NL(o) LEADING(c) */
       o.order_id, c.name
FROM   customers c JOIN orders o ON o.customer_id=c.id
WHERE  c.region='APAC' AND o.order_date BETWEEN :d1 AND :d2;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'ALL +PREDICATE +NOTE'));

-- 실행 후 실제 통계
VAR d1 DATE; VAR d2 DATE;
EXEC :d1 := DATE '2024-06-01'; EXEC :d2 := DATE '2024-06-30';
SELECT /* NL */ /*+ USE_NL(o) LEADING(c) */
       COUNT(*)
FROM   customers c JOIN orders o ON o.customer_id=c.id
WHERE  c.region='APAC' AND o.order_date BETWEEN :d1 AND :d2;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
  'ALLSTATS LAST +PREDICATE +PEEKED_BINDS +NOTE'));
```

**읽는 법**
- HASH가 선택되었을 때 `TempSpc` 크고 **Time 대부분이 해시 빌드/프로브**에 몰리면 → **NL이 유리**
- NL에서 `A-Rows`가 **E-Rows 대비 과대**이면 → 인덱스 선택성/히스토그램/필터 선행 필요

### 5.2 **정렬·스필 감지** (ORDER BY / GROUP BY)
```sql
SELECT /* sort test */ c.region, SUM(o.amount) amt
FROM   orders o JOIN customers c ON c.id=o.customer_id
WHERE  o.order_date BETWEEN :d1 AND :d2
GROUP  BY c.region
ORDER  BY amt DESC;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
  'ALLSTATS LAST +PREDICATE +PROJECTION +NOTE'));
```
- `SORT GROUP BY` / `SORT ORDER BY` 라인 **TempSpc** 확인 → **PGA↑/카디널리티↓/인덱스 정렬 회피** 검토

### 5.3 **바인드 피킹/스큐** — `+PEEKED_BINDS` 활용
```sql
VAR r VARCHAR2(10); EXEC :r := 'APAC';
SELECT /* region test */ COUNT(*) FROM customers WHERE region=:r;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
  'ALLSTATS LAST +PEEKED_BINDS +NOTE'));
```
- 같은 SQL이라도 바인드 값에 따라 Child Cursor/플랜이 달라질 수 있음 → **ACS(Adaptive Cursor Sharing)** 진단의 출발점

---

## 6. 출력 옵션 모음(실전 단골)

- `ALL` : 가능한 많은 열
- `+PREDICATE` : access/filter 조건
- `+ALIAS` : 테이블/컬럼 **별칭**
- `+PROJECTION` : 각 라인 **투영 컬럼**(연산/함수 포함)
- `+NOTE` : 옵티마이저 메시지(동적 샘플링/뷰 머지/적응 등)
- `+OUTLINE` : **Outline(힌트 세트)** — 플랜 고정/재현에 유용
- `+PEEKED_BINDS` : 바인드 피킹 값
- `+PARALLEL` : PX 흐름/Distribution 표시
- `ALLSTATS FIRST/LAST` : **라인별 실행 통계**(필수)

**추천 프리셋**
```sql
-- 예상(간단)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'TYPICAL +PREDICATE +NOTE'));

-- 실제(정밀)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
  'ALLSTATS LAST +PEEKED_BINDS +PREDICATE +PROJECTION +OUTLINE +NOTE'));
```

---

## 7. DISPLAY_CURSOR와 주변 도구의 조합

- **SQL Monitor**: *라인별 경과 시간/대기/Temp*의 실시간 시각화(특히 장수행/병렬)
- **ASH/AWR**: *상위 이벤트/SQL/Plan Line 샘플*로 시스템 시야
- **10046 Trace + TKPROF**: *Parse/Exec/Fetch* 호출 단위 **대기/시간** 근거
- **DBMS_XPLAN.DISPLAY_AWR**: AWR에 저장된 **옛 플랜** 비교(과거 회귀 분석)

```sql
-- AWR 상의 과거 계획 보기(허용 환경)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_AWR('&&SQL_ID', NULL, NULL,
  'TYPICAL +PREDICATE +NOTE'));
```

---

## 8. 자주 부딪히는 함정 & 대처

1) **`EXPLAIN`과 실제 상이**: 바인드 값/적응 기능/통계 시점 → **반드시 ALLSTATS로 검증**
2) **Child 폭증**: `v$sql_shared_cursor`로 원인(바인드 타입/길이/NLS/환경) 확인, **SQL 표준화/바인드 고정**
3) **TempSpc=0인데 느림**: CPU 바운드/락/네트워크 가능 → SQL Monitor/ASH/락 진단 병행
4) **PX 스큐**: 특정 라인 `A-Rows` 편중 + 한쪽 라인 Time 급증 → **분배 키/파티셔닝/재해시** 검토
5) **Outline만 믿고 고정**: 데이터 변동 시 취약 → **통계/히스토그램/조인순서** 원인 교정 먼저

---

## 9. 튜닝 루틴(현장 템플릿)

1) **문제 SQL 실행**(가능하면 실제 바인드)
2) `DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PEEKED_BINDS +PREDICATE +NOTE')`
3) **라인별** `A-Rows` vs `E-Rows`, **Time/Temp/Buffers** 최대 라인 찾기
4) **원인 가설**: 카디널리티/접근경로/조인전략/정렬/스필/병렬스큐
5) **작은 변경**: 인덱스/통계(히스토그램)/조인순서/뷰 머지/필터 푸시/함수제거/바인드 정규화
6) **다시 실행** → 같은 포맷으로 전/후 비교(표를 나란히 저장)
7) **재발방지**: Baseline/Profile(필요시), 배포 전 회귀 테스트 케이스화

---

## 10. 미니 FAQ

- **Q. `ALLSTATS`가 비어 있어요.**
  A. 해당 커서를 **실행**한 뒤에 보세요. `EXPLAIN`만 하고 `DISPLAY_CURSOR` 하면 통계가 없습니다.

- **Q. 어떤 옵션을 늘 붙일까요?**
  A. `ALLSTATS LAST +PEEKED_BINDS +PREDICATE +OUTLINE +NOTE` 를 기본으로, 필요 시 `+PROJECTION/+PARALLEL`.

- **Q. 실행 중 적응 계획 전환은 보이나요?**
  A. `NOTE`/`OUTLINE`에 흔적이 남고, `ALLSTATS`에서 라인 Time 분포가 달라집니다. SQL Monitor가 더 선명합니다.

---

## 11. 결론

- `DBMS_XPLAN`은 **세 가지 모드의 하모니**다.
  1) **DISPLAY**: *예상 지도*
  2) **DISPLAY_CURSOR**: *실제 경로*
  3) **ALLSTATS**: *어디에서 시간이 들었는지*
- 튜닝은 **라인별 사실**에서 출발한다. **A-Rows vs E-Rows**, **Time/Temp/Buffers**를 통해 **원인**을 짚고, **작고 정확한 수정**으로 **Before/After**를 증명하라.

> 한 줄 정리
> **DISPLAY**로 “갈 길”을 보고, **DISPLAY_CURSOR(ALLSTATS)**로 “걸은 길”과 “넘어진 지점”을 확인하라.
> 답은 **Row Source 표**에 있다.
