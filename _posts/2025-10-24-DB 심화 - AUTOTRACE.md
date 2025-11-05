---
layout: post
title: DB 심화 - AUTOTRACE
date: 2025-10-24 14:25:23 +0900
category: DB 심화
---
# Oracle **AUTOTRACE** 완전 가이드

> 목표  
> - `SET AUTOTRACE`의 **모드/옵션**을 정확히 이해하고, **무엇을 보여주며(세션 통계)** 무엇은 **보여주지 않는지(라인별 실제 실행 통계)** 구분합니다.  
> - **환경 준비(PLAN_TABLE, PLUSTRACE 역할)** 부터 **실행/해석/튜닝 루틴**을 실습 예제로 익힙니다.  
> - AUTOTRACE 결과를 **실행계획(DBMS_XPLAN.DISPLAY_CURSOR)**, **SQL Monitor**, **AWR/ASH**와 **연결**하는 방법을 제시합니다.

---

## 0) 한눈 요약

- `AUTOTRACE`는 **SQL*Plus/SQLcl**에서 **SQL 실행 직후**에  
  1) **예상 실행계획(EXPLAIN PLAN 기반)** 과  
  2) **세션 통계(일부 v$sesstat 기반)**  
  을 **요약**해 보여주는 **경량 분석 도구**입니다.
- **장점**: 가볍고 빠르며, **쿼리 결과를 감추고 통계만** 볼 수 있음(`TRACEONLY`).  
- **한계**:  
  - **계획은 “예상”**(EXPLAIN)이지 **실제 실행 라인 통계가 아님**  
  - **라인별 A-Rows/버퍼/Temp**를 **표시하지 않음**  
  - **적응계획/병렬/스필**의 런타임 전환은 **반영 한계**  
- **권장 조합**: AUTOTRACE로 **빠른 스크리닝** → 필요 시 **`DBMS_XPLAN.DISPLAY_CURSOR('ALLSTATS LAST')`** / **SQL Monitor**로 **정밀 분석**.

---

## 1) 설치·권한(최초 1회)

### 1.1 PLAN_TABLE 준비
```sql
-- 이미 있는 경우가 많지만, 없으면 스키마에서 실행
@?/rdbms/admin/utlxplan.sql
```

### 1.2 PLUSTRACE 역할(세션 통계 접근 권한)
> AUTOTRACE의 “Statistics” 출력에 필요(세션이 v$sesstat 등의 조회 권한).

```sql
-- SYS로 접속하여 1회 설치
@?/sqlplus/admin/plustrce.sql

-- 분석할 사용자에게 권한 부여
GRANT plustrace TO your_user;
-- 필요시 PLAN_TABLE 권한(스키마 내부면 불필요)
```

**참고**: 보안 정책상 `plustrace` 대신 특정 v$ 뷰에 **선택적 권한 부여**로 대체하는 환경도 있습니다.

---

## 2) 기본 사용법(옵션 총정리)

```sql
-- 결과 + 계획 + 통계(둘 다). 쿼리 결과도 화면에 출력
SET AUTOTRACE ON;

-- 쿼리 결과는 감추고(미표시) 계획+통계만 출력
SET AUTOTRACE TRACEONLY;

-- EXPLAIN PLAN만 (실행 없이 계획만)
SET AUTOTRACE EXPLAIN;

-- 통계만 (실행은 함, 결과는 출력) — 계획은 생략
SET AUTOTRACE STATISTICS;

-- TRACEONLY + 세부 옵션
SET AUTOTRACE TRACEONLY EXPLAIN;     -- 실행하지 않고 계획만
SET AUTOTRACE TRACEONLY STATISTICS;  -- 실행은 하고 통계만(결과 미표시)

-- 해제
SET AUTOTRACE OFF;
```

> **팁**  
> - `TRACEONLY`는 **대량 결과를 화면에 쏟지 않고** 통계만 보기에 유용(네트워크 전송 영향 제거에 도움).  
> - `EXPLAIN`은 **실행하지 않음**: 데이터/통계 최신성, 바인드 값에 따라 **실행시 실제 경로**는 달라질 수 있음.

---

## 3) 출력 형식 이해(무엇을 보여주나?)

AUTOTRACE는 보통 **두 블록**을 보여줍니다.

1) **Execution Plan**: EXPLAIN PLAN 기반 예상 계획  
2) **Statistics**: 아래와 같이 **세션 차원**의 누적치 요약(대표 예)
   - `recursive calls`  
   - `db block gets` / `consistent gets`  
   - `physical reads` / `physical writes`  
   - `redo size` / `bytes sent via SQL*Net to client`  
   - `sorts (memory)` / `sorts (disk)`  
   - `rows processed`  
   - … 등

> **중요**: Statistics는 **쿼리 실행으로 증가한 세션 통계의 차분**에 해당합니다(출력 형식/항목은 버전/클라이언트에 따라 조금씩 다름).

---

## 4) 예제 스키마 준비

```sql
DROP TABLE customers PURGE;
DROP TABLE orders PURGE;

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

CREATE INDEX ix_orders_cust_date ON orders(customer_id, order_date);
CREATE INDEX ix_cust_region_grade ON customers(region, grade);

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
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'CUSTOMERS',cascade=>TRUE);
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'ORDERS',cascade=>TRUE);
```

---

## 5) 케이스 1 — **인덱스 범위 스캔 vs 풀스캔** 비교

### 5.1 인덱스 활용 쿼리
```sql
SET AUTOTRACE TRACEONLY;

VAR cid NUMBER;      EXEC :cid := 12345;
VAR d1  DATE;        EXEC :d1  := DATE '2024-06-01';
VAR d2  DATE;        EXEC :d2  := DATE '2024-06-30';

SELECT /* IDX */ SUM(o.amount)
FROM   orders o
WHERE  o.customer_id = :cid
AND    o.order_date BETWEEN :d1 AND :d2;
```

**해석 포인트**
- **Execution Plan**: `INDEX RANGE SCAN IX_ORDERS_CUST_DATE` → `TABLE ACCESS BY ROWID`  
- **Statistics**:  
  - `consistent gets` 는 **랜덤 읽기 수**에 비례(행 수 적으면 작음)  
  - `physical reads` 가 0이면 **버퍼캐시 히트**  
  - `bytes sent via SQL*Net to client` 가 작으면 결과 출력 부담 미미

### 5.2 풀스캔 유도(힌트로 비교)
```sql
SELECT /*+ FULL(o) */ SUM(o.amount)
FROM   orders o
WHERE  o.customer_id = :cid
AND    o.order_date BETWEEN :d1 AND :d2;
```
**예상 변화**
- **Plan**: `TABLE ACCESS FULL ORDERS`  
- **Statistics**: `consistent gets`/`physical reads` 증가, `sorts` 없이도 I/O 비용 상승

> **요지**: AUTOTRACE로 **전략 차이**가 **세션 통계**에 미치는 영향을 빠르게 비교 가능.

---

## 6) 케이스 2 — **정렬/스필** 감지(Temp 사용)

```sql
-- 정렬이 필요한 쿼리
SET AUTOTRACE TRACEONLY;

SELECT /* ORDER BY test */ c.region, SUM(o.amount) AS amt
FROM   orders o JOIN customers c ON c.id=o.customer_id
WHERE  o.order_date BETWEEN :d1 AND :d2
GROUP  BY c.region
ORDER  BY amt DESC;
```

**해석 포인트**
- **Plan**: `HASH JOIN` + `SORT GROUP BY`, 마지막에 `SORT ORDER BY`  
- **Statistics**:  
  - `sorts (memory)` vs `sorts (disk)` : **디스크 스필 여부**  
  - `physical writes`/`reads` 증가 시 **TEMP** 사용 가능성↑  
- AUTOTRACE만으로 **정확한 TempSpc(바이트)** 는 안 보임 → 필요 시 **SQL Monitor** 또는 `DISPLAY_CURSOR('ALLSTATS LAST')`에서 `TempSpc` 확인.

---

## 7) 케이스 3 — **조인 전략(NL vs HASH)** 빠른 스크리닝

```sql
-- 기본 (옵티마이저 기본 선택)
SET AUTOTRACE TRACEONLY;

SELECT /* join test */ COUNT(*)
FROM   orders o JOIN customers c ON c.id=o.customer_id
WHERE  c.grade >= 3;

-- NL 강제
SELECT /*+ USE_NL(o c) LEADING(c) */ COUNT(*)
FROM   customers c JOIN orders o ON o.customer_id=c.id
WHERE  c.grade >= 3;

-- HASH 강제
SELECT /*+ USE_HASH(o c) */ COUNT(*)
FROM   orders o JOIN customers c ON c.id=o.customer_id
WHERE  c.grade >= 3;
```

**해석 포인트**
- **Plan**: `NESTED LOOPS` vs `HASH JOIN` 비교  
- **Statistics**:  
  - NL의 경우 **random I/O**가 늘 가능성 → `consistent gets` 증가  
  - HASH의 경우 **PGA 부족 시 스필** → `sorts (disk)`/`physical reads/writes` 증가 경향

> AUTOTRACE는 **결과값 출력 억제(TRACEONLY)** 로 네트워크 비용 가림 → **엔진 내부 비용 차이**에 더 집중 가능.

---

## 8) 케이스 4 — **DML**(UPDATE/INSERT)에서 AUTOTRACE

```sql
SET AUTOTRACE ON;

UPDATE /* upd */ orders
SET    amount = amount * 1.05
WHERE  customer_id = :cid
AND    order_date  >= :d1;

-- AUTOTRACE 출력:
-- 1) Execution Plan: UPDATE STATEMENT + 액세스 경로
-- 2) Statistics: redo size/consistent gets/physical reads/rows processed 등
```

**해석 포인트**
- DML은 **Redo 생성량(`redo size`)** 과 **Row Lock으로 인한 consistent gets 증가**를 관찰.  
- **행수** 많으면 `rows processed`↑, **인덱스 유지비용** 반영되어 `consistent gets`/`physical reads`↑ 가능.

---

## 9) 케이스 5 — **TRACEONLY STATISTICS**로 **쿼리 결과 전송 비용 제거**

```sql
SET AUTOTRACE TRACEONLY STATISTICS;

SELECT * FROM orders WHERE customer_id = :cid;
```

- **실행은 하지만 결과를 출력하지 않으므로** `bytes sent via SQL*Net to client` 가 **상대적으로 작게** 측정  
- 실제 화면 출력/네트워크 전송 비용과 **분리**하여 엔진 내부 비용을 관찰할 수 있음.

---

## 10) AUTOTRACE vs `DISPLAY_CURSOR(ALLSTATS LAST)` 차이

| 항목 | AUTOTRACE | DISPLAY_CURSOR('ALLSTATS LAST') |
|---|---|---|
| 계획 | **EXPLAIN 기반 예상** | **실제 실행 계획** |
| 통계 | **세션 통계 요약**(쿼리 레벨) | **플랜 라인별** A-Rows/Buffer/Time/Temp |
| 실행 필요 | `EXPLAIN` 모드면 불필요 | **필수**(마지막 실행) |
| 적응 플랜/병렬 | 반영 한계 큼 | 반영(실행 결과 그대로) |
| 가벼움 | **매우 가벼움** | 상대적으로 무거움(하지만 일반적) |

> **권장 루틴**:  
> - **빠른 스크리닝**: `SET AUTOTRACE TRACEONLY`  
> - **정밀 분석**: 실행 후  
>   ```sql
>   SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
>     'ALLSTATS LAST +PEEKED_BINDS +OUTLINE +PROJECTION +ALIAS +NOTE'));
>   ```

---

## 11) AUTOTRACE 출력 해석 팁(지표별)

- `recursive calls`: 데이터 딕셔너리 접근/자동 통계/뷰 변환 등 **부수 SQL**. 과다 시 **원인 SQL** 확인.  
- `db block gets`(현재 읽기) vs `consistent gets`(CR 읽기): DML/갱신 경로에서 **db block gets** 증가 경향.  
- `physical reads/writes`: 버퍼캐시 미스/스필/TEMP I/O 지표.  
- `redo size`: DML/Direct-path 등 **로그 생성량**.  
- `sorts (memory/disk)`: 정렬·해시 워크에 대한 **PGA vs TEMP** 사용 지표.  
- `bytes sent/received via SQL*Net`: **결과/바인드 전송량**(TRACEONLY로 억제 가능).  
- `rows processed`: 처리된 행 개수(SELECT는 **페치한 행 기준** 주의).

---

## 12) AUTOTRACE 한계 & 주의점

1) **계획은 EXPLAIN**: **실제 실행과 다를 수 있음**(바인드 피킹/적응 계획/병렬/자료 변동).  
2) **라인 단위 분해 없음**: 어느 Plan Id에서 시간을 쓰는지 **직접 알 수 없음**.  
3) **결과 출력 비용 포함 가능**: `SET AUTOTRACE ON` 상태로 **결과를 화면에 뿌리면** 네트워크/클라이언트 렌더 비용까지 섞임 → **`TRACEONLY`** 활용.  
4) **DML/PL/배치**: **여러 단계**(커서 반복/배치 바인드)가 **통계에 합산**되어 보일 수 있음.  
5) **통계 해석은 상대 비교** 중심: Before/After 비교로 추세를 보라.

---

## 13) AUTOTRACE + 정밀 도구 조합

- **세션 통계만으로 부족**하면:  
  - **SQL Monitor**(Real-Time Report): Plan 라인별 경과시간/Temp/읽기/병렬 그레인  
  - **10046 Trace + TKPROF**: Parse/Exec/Fetch 호출별 시간/대기  
  - **ASH/AWR**: 시스템/시간대 상위 이벤트/SQL, Plan Line 샘플(`sql_plan_line_id`)

**예시: 같은 SQL을 정밀 확인**
```sql
-- 실행
SELECT /* target */ c.region, COUNT(*) 
FROM   customers c JOIN orders o ON o.customer_id=c.id
WHERE  o.order_date BETWEEN :d1 AND :d2
GROUP  BY c.region;

-- 실제 라인별 통계
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
  'ALLSTATS LAST +PEEKED_BINDS +OUTLINE +PROJECTION +ALIAS +NOTE'));
```

---

## 14) 자주 쓰는 운영 패턴(템플릿)

### 14.1 튜닝 5분 루틴
```sql
SET AUTOTRACE TRACEONLY;
-- 1) 후보 SQL 실행 → 세션 통계(consistent/physical/sorts/redo) 확인
-- 2) EXPLAIN 계획 스냅샷(인덱스/조인/정렬 구조 확인)
SET AUTOTRACE OFF;

-- 3) 실제 실행 계획/라인 통계
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
  'ALLSTATS LAST +PEEKED_BINDS +NOTE'));

-- 4) 수정(인덱스/통계/조인순서/힌트) → 반복
```

### 14.2 대량 배치 전 검증
- `TRACEONLY EXPLAIN` 으로 **대략의 경로** 확인  
- 소량 샘플로 `TRACEONLY STATISTICS` → **Redo/Sort/Reads** 규모 감지  
- 예상보다 `sorts (disk)` / `physical writes` 가 크면 **PGA/TEMP/조인 전략 조정**

---

## 15) (참고) SQLcl / SQL Developer에서의 Autotrace

- **SQLcl**: `SET AUTOTRACE` 동일, 출력이 `DBMS_XPLAN` 기반으로 다소 **가독성 향상**.  
- **SQL Developer**: **Autotrace 탭** 제공(실행 후 Plan/Statistics), GUI에서 **실행 결과 숨기기** 옵션 존재.

---

## 16) 미니 FAQ

**Q1. AUTOTRACE의 Plan이 실제와 달라 보여요.**  
A. 정상입니다. AUTOTRACE Plan은 **EXPLAIN** 기반입니다. **실제는 `DISPLAY_CURSOR(ALLSTATS LAST)`** 로 보세요.

**Q2. Statistics에서 ‘rows processed’가 기대와 달라요.**  
A. SELECT의 경우 **페치된 행 수** 영향이 있으며, 일부 클라이언트 설정/배열 페치 크기에 따라 달라질 수 있습니다.

**Q3. 권한 오류(세션 통계 표시가 안 됨)가 납니다.**  
A. **PLUSTRACE** 역할이 없거나 v$ 뷰 권한 부족입니다. `plustrce.sql` 후 `GRANT plustrace`를 받아야 합니다.

**Q4. 대량 결과 쿼리에서 AUTOTRACE가 너무 느립니다.**  
A. `SET AUTOTRACE TRACEONLY` 로 **결과 출력**을 생략하세요. 네트워크/클라이언트 렌더 비용을 제거합니다.

---

## 17) 핵심 정리

- **AUTOTRACE** = **빠른 스크리닝 도구**:  
  - **예상 계획**(EXPLAIN) + **세션 통계 요약**으로 **쿼리 성격**을 즉시 파악  
  - 결과 출력이 방해되면 `TRACEONLY`로 **통계만**  
- **정밀 분석**은 **실제 실행**을 반영하는 **`DISPLAY_CURSOR('ALLSTATS LAST')`/SQL Monitor** 로 연결  
- **튜닝 루틴**: AUTOTRACE로 대략 → 실행/라인통계로 정확 → 인덱스/통계/조인/정렬/파티션/메모리 조정 → 재확인

> 한 줄 결론  
> **AUTOTRACE로 “크게” 보고**, **DBMS_XPLAN/Monitor로 “정밀”하게 조인다.**  
> 빠른 감은 AUTOTRACE가 주고, 결론은 **실제 실행 통계**가 말해준다.