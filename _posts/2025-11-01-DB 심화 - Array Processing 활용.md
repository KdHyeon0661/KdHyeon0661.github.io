---
layout: post
title: DB 심화 - Array Processing 활용
date: 2025-11-01 19:25:23 +0900
category: DB 심화
---
# Array Processing 활용

> **핵심 요약**  
> Array Processing(배열 처리)은 **“여러 행을 한 번에”** 보내거나 받는 기법이다.  
> - **DML**: `FORALL`, 드라이버 **배열 바인드/배치 실행**으로 **Execute 호출 수**를 급감 → `user calls` 감소, `log file sync`·네트워크 오버헤드↓  
> - **SELECT**: `BULK COLLECT`/드라이버 **배열 페치**로 **Fetch 호출 수**를 감소 → 왕복/CPU↓, 응답시간↓  
> - **부가 기능**: `RETURNING BULK COLLECT`, `SAVE EXCEPTIONS`, 배열 바인드 **IN-list**, 파티셔닝/인덱스 설계와 결합 시 최대 효과.

---

## 0. 실습 공통 스키마 & 데이터

```sql
-- 대용량 테이블
DROP TABLE t_sale PURGE;
CREATE TABLE t_sale (
  sale_id     NUMBER PRIMARY KEY,
  cust_id     NUMBER NOT NULL,
  region      VARCHAR2(10),
  order_dt    DATE,
  status      VARCHAR2(10),
  amount      NUMBER(12,2)
);

-- 더미 데이터(200만 행)
INSERT /*+ APPEND */ INTO t_sale
SELECT level,
       MOD(level, 100000)+1,
       CASE MOD(level,4) WHEN 0 THEN 'APAC' WHEN 1 THEN 'EMEA' WHEN 2 THEN 'AMER' ELSE 'OTHER' END,
       DATE '2025-01-01' + MOD(level, 365),
       CASE WHEN MOD(level,20)=0 THEN 'CANCEL' ELSE 'OK' END,
       ROUND(DBMS_RANDOM.VALUE(10,1000),2)
FROM dual CONNECT BY level <= 2000000;
COMMIT;

CREATE INDEX ix_sale_region_dt ON t_sale(region, order_dt);
CREATE INDEX ix_sale_cust      ON t_sale(cust_id);
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'T_SALE',cascade=>TRUE,
    method_opt=>'FOR COLUMNS SIZE 75 region SIZE 1 order_dt');
END;
/
```

---

## 1. 왜 Array Processing인가? — **성능 모델**

**왕복 기반 응답시간 근사**  
$$
\text{RT} \approx \underbrace{\text{RTT} \times N_{\text{calls}}}_{\text{네트워크 왕복}}
+ \underbrace{\text{CPU} + \text{Waits}}_{\text{서버 내부 작업}}
$$

- **Row-by-Row**: `N_calls`가 **행 수**에 비례.  
- **Array Processing**: `N_calls \approx \frac{\text{행 수}}{\text{배열 크기}}` → **선형 감소**.  
- 동일 작업량이라도 **왕복 감소**만으로 **RT**가 크게 줄어든다.

---

## 2. PL/SQL — **BULK COLLECT / FORALL** 기본 패턴

### 2.1 Array SELECT: `BULK COLLECT` (+ LIMIT)

```plsql
DECLARE
  TYPE t_sale_rec IS RECORD(
    sale_id  t_sale.sale_id%TYPE,
    cust_id  t_sale.cust_id%TYPE,
    amount   t_sale.amount%TYPE
  );
  TYPE t_sale_tab IS TABLE OF t_sale_rec;
  v_tab t_sale_tab;
BEGIN
  -- LIMIT: PGA 메모리 사용과 왕복 균형 (예: 10k)
  FOR cur IN (
    SELECT sale_id, cust_id, amount
    FROM   t_sale
    WHERE  region='APAC'
    AND    order_dt >= TRUNC(SYSDATE)-90
  ) LOOP NULL; END LOOP;  -- (비교용: row-by-row)

  v_tab := t_sale_tab();

  FOR i IN 1..100 LOOP
    SELECT /*+ no_merge */ sale_id, cust_id, amount
    BULK COLLECT INTO v_tab
    FROM t_sale
    WHERE region='APAC'
      AND order_dt BETWEEN TRUNC(SYSDATE)-90 AND TRUNC(SYSDATE)
      AND ROWNUM <= 10000; -- 예제 단위
    EXIT WHEN v_tab.COUNT=0;
    -- v_tab 처리 (파일 쓰기, 네트워크 송신 등)
    v_tab.DELETE;
  END LOOP;
END;
/
```

- **포인트**
  - `LIMIT` 미지정은 대량 데이터에서 **PGA 폭증** 위험.  
  - 일반적으로 **1k~20k** 사이에서 실측으로 최적점 찾기.

### 2.2 Array DML: `FORALL` (+ `SAVE EXCEPTIONS`)

```plsql
DECLARE
  TYPE t_amt IS TABLE OF NUMBER INDEX BY PLS_INTEGER;
  TYPE t_id  IS TABLE OF NUMBER INDEX BY PLS_INTEGER;
  v_ids  t_id;
  v_amts t_amt;
  l_errs EXCEPTION;
  PRAGMA EXCEPTION_INIT(l_errs, -24381); -- SAVE EXCEPTIONS 집계 오류
BEGIN
  -- 바꿀 값 준비 (예: 배치 50,000행)
  FOR i IN 1..50000 LOOP
    v_ids(i)  := i;
    v_amts(i) := 999.99;
  END LOOP;

  BEGIN
    FORALL i IN INDICES OF v_ids SAVE EXCEPTIONS
      UPDATE t_sale
      SET    amount = v_amts(i)
      WHERE  sale_id = v_ids(i);

  EXCEPTION WHEN l_errs THEN
    FOR j IN 1..SQL%BULK_EXCEPTIONS.COUNT LOOP
      DBMS_OUTPUT.PUT_LINE('ERR IDX='||SQL%BULK_EXCEPTIONS(j).ERROR_INDEX||
                           ' CODE='||SQL%BULK_EXCEPTIONS(j).ERROR_CODE);
    END LOOP;
  END;

  COMMIT;
END;
/
```

- **포인트**
  - `FORALL`은 **SQL 1번 파싱, 바인드만 배열** → Execute 왕복 수 **급감**.  
  - `SAVE EXCEPTIONS`로 **문제 행만 분리**(대부분 성공 → 성능 유지).  
  - 대량 DML은 **주기적 COMMIT**(예: 5k~20k 묶음)로 `log file sync` 부하를 균형화.

### 2.3 `RETURNING BULK COLLECT` — 생성 키 일괄 회수

```plsql
DECLARE
  TYPE t_id  IS TABLE OF NUMBER;
  TYPE t_rid IS TABLE OF UROWID;
  v_ids  t_id  := t_id();
  v_rids t_rid := t_rid();
BEGIN
  FOR i IN 1..10000 LOOP v_ids.EXTEND; v_ids(i):=1000000+i; END LOOP;

  FORALL i IN 1..v_ids.COUNT
    INSERT INTO t_sale(sale_id, cust_id, region, order_dt, status, amount)
    VALUES (v_ids(i), MOD(v_ids(i),100000)+1, 'APAC', SYSDATE, 'OK', 1)
    RETURNING ROWID BULK COLLECT INTO v_rids;

  DBMS_OUTPUT.PUT_LINE('Inserted='||v_rids.COUNT);
  COMMIT;
END;
/
```

---

## 3. 클라이언트 드라이버 — **배열 바인드/배치 & 배열 페치**

### 3.1 JDBC

#### 3.1.1 대량 SELECT: `setFetchSize`
```java
String sql = """
  SELECT /* array-fetch */ sale_id, amount
  FROM t_sale
  WHERE region=? AND order_dt BETWEEN ? AND ?
""";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
  ps.setString(1, "APAC");
  ps.setDate(2, Date.valueOf("2025-09-01"));
  ps.setDate(3, Date.valueOf("2025-11-01"));
  ps.setFetchSize(1000); // 권장: 500~2000 (행당 크기 고려)
  try (ResultSet rs = ps.executeQuery()) {
    while (rs.next()) { /* consume */ }
  }
}
```

#### 3.1.2 대량 DML: `addBatch/executeBatch`
```java
String ins = "INSERT INTO t_sale(sale_id,cust_id,region,order_dt,status,amount) VALUES (?,?,?,?,?,?)";
try (PreparedStatement ps = conn.prepareStatement(ins)) {
  int batch = 0;
  for (int i=1;i<=100000;i++) {
    ps.setInt(1, 3000000+i);
    ps.setInt(2, i % 100000 + 1);
    ps.setString(3, "APAC");
    ps.setDate(4, new Date(System.currentTimeMillis()));
    ps.setString(5, "OK");
    ps.setBigDecimal(6, new BigDecimal("12.34"));
    ps.addBatch();
    if (++batch % 1000 == 0) {
      ps.executeBatch();  // 1,000행씩 한 번 호출
      conn.commit();      // 주기적 커밋
    }
  }
  ps.executeBatch();
  conn.commit();
}
```

- **튜닝 팁**
  - Statement Cache(Implicit/Explicit) 활성 → Parse 감소  
  - Auto-commit OFF → per-row COMMIT 금지  
  - 네트워크 RTT 큰 환경일수록 Batch/Fetch 효과 큼

### 3.2 ODP.NET (C#)

```csharp
// 배열 바인드 DML
using var cmd = new OracleCommand("INSERT INTO t_sale(sale_id, amount) VALUES (:1,:2)", conn);
cmd.ArrayBindCount = ids.Length;
cmd.Parameters.Add(":1", OracleDbType.Int32, ids, ParameterDirection.Input);
cmd.Parameters.Add(":2", OracleDbType.Decimal, amts, ParameterDirection.Input);
cmd.ExecuteNonQuery(); // 1회 호출로 N행 처리
conn.Commit();
```

```csharp
// 배열 페치
using var cmd = new OracleCommand(@"
  SELECT sale_id, amount FROM t_sale
  WHERE region=:r AND order_dt BETWEEN :d1 AND :d2", conn);
cmd.Parameters.Add(":r", OracleDbType.Varchar2).Value = "APAC";
cmd.Parameters.Add(":d1", OracleDbType.Date).Value = new DateTime(2025,9,1);
cmd.Parameters.Add(":d2", OracleDbType.Date).Value = new DateTime(2025,11,1);
cmd.FetchSize = 8 * 1024 * 1024; // 바이트 기준 (행당 크기×행수만큼)
using var rdr = cmd.ExecuteReader();
while (rdr.Read()) { /* consume */ }
```

### 3.3 Python (oracledb)

```python
# 배열 페치
cur.arraysize = 1000
cur.execute("""
  SELECT sale_id, amount FROM t_sale
  WHERE region=:r AND order_dt BETWEEN :d1 AND :d2
""", r='APAC', d1=date(2025,9,1), d2=date(2025,11,1))
for row in cur:
    pass
```

```python
# 배열 바인드 DML
rows = [(3000000+i, Decimal('12.34')) for i in range(1, 100000+1)]
cur.executemany("INSERT INTO t_sale(sale_id, amount) VALUES (:1,:2)", rows)
conn.commit()
```

- **팁**: `batcherrors=True` 옵션으로 부분 오류 수집 가능(버전별 기능 확인).

---

## 4. 고급: **배열 바인드로 IN-list** 처리(정적 SQL 유지)

### 4.1 스키마 타입 + TABLE 연산자

```sql
CREATE OR REPLACE TYPE t_num_list AS TABLE OF NUMBER;
```

```sql
-- 정적 SQL: 텍스트 고정, 커서 공유 유지
SELECT /* array-inlist */ sale_id, amount
FROM   t_sale
WHERE  cust_id IN (SELECT COLUMN_VALUE FROM TABLE(:cust_ids))
AND    order_dt BETWEEN :d1 AND :d2;
```

- **드라이버 바인딩**: `:cust_ids`에 숫자 배열을 바인드  
- **장점**: 대량 IN도 동적 문자열 없이 처리, Parse 폭증 방지  
- **주의**: 실행 계획은 **HASH JOIN** 경향 → 통계/인덱스/카디널리티 점검

### 4.2 GTT 조인(초대량 키)

```sql
CREATE GLOBAL TEMPORARY TABLE t_filter(id NUMBER) ON COMMIT DELETE ROWS;
-- 앱에서 벌크 Insert 후
SELECT s.*
FROM   t_sale s
JOIN   t_filter f ON f.id = s.cust_id
WHERE  s.order_dt BETWEEN :d1 AND :d2;
```

---

## 5. 오류 처리 — **배치와 부분 실패**

### 5.1 PL/SQL: `SAVE EXCEPTIONS` + `SQL%BULK_EXCEPTIONS`
- 이미 예제 제시. 문제 행만 로깅/분리 처리 → **대부분 성공 유지**.

### 5.2 JDBC Batch
```java
try {
  ps.executeBatch();
  conn.commit();
} catch (BatchUpdateException e) {
  int[] counts = e.getUpdateCounts(); // 성공/실패 행 식별
  // 실패 행만 재시도/로깅
}
```

### 5.3 Python `executemany` + batcherrors(버전 기능)
- 드라이버 옵션으로 **부분 오류 수집** 후 재처리.

---

## 6. Array 크기 결정 — **메모리 vs 왕복의 균형**

- **SELECT**: `fetchSize/arraysize`를 **행당 평균 크기 × N** 이 **수 MB~수십 MB** 정도가 되도록.  
  - 예) 행당 200B × 1000행 ≈ 200KB (작다) → 5000~10000으로 키우기.  
  - 너무 크면 **PGA/클라이언트 메모리** 부담 + GC/직렬화 비용 증가.

- **DML**: Batch/ArrayBind 크기를 **1k~10k** 범위에서 실측.  
  - 너무 작으면 왕복 많음, 너무 크면 `log file sync`·락 경쟁·Undo/Redo 압박.

- **경험 법칙**:  
  1) RTT 큰 환경(지리적 분산) → **큰 배열**이 유리.  
  2) 로우당 LOB/큰 컬럼多 → **중간 크기**로 타협.  
  3) 트리거/제약 검증多 → **중간 크기 + SAVE EXCEPTIONS**.

---

## 7. 검증: SQL Trace/TKPROF로 **전/후 비교**

### 7.1 트레이스 켜기
```sql
ALTER SESSION SET statistics_level=ALL;
ALTER SESSION SET events '10046 trace name context forever, level 8';
-- 케이스 A) fetchSize=50
-- 케이스 B) fetchSize=1000
ALTER SESSION SET events '10046 trace name context off';
```

### 7.2 TKPROF 지표 포인트
- `call` 표에서 **Fetch count**(SELECT), **Execute count**(DML) 급감 확인  
- `bytes sent/received via SQL*Net` 단위당 행수 증가(효율↑)  
- 동일 rows 대비 **elapsed** 감소

---

## 8. 시나리오 별 **설계 레시피**

### 8.1 대량 마이그레이션/적재
- `INSERT /*+ APPEND */` + **배치 커밋** + **인덱스/트리거 최소화**  
- 대상 인덱스는 적재 후 생성(병렬 빌드) or **무효화→재빌드**  
- `FORALL`/JDBC Batch로 1만~10만 행 단위 처리  
- AWR 상위 이벤트: `direct path write temp` / `DB CPU` → 정상

### 8.2 실시간 API 다량 읽기
- **Keyset Pagination** + **배열 페치**(큰 fetchSize)  
- 열 최소화(필요 컬럼만), SARG 가능한 WHERE  
- 네트워크 RTT 크면 **배열 크기↑**로 왕복 최소화

### 8.3 이벤트 로그/트래킹 수집
- **배열 바인드 INSERT** + `COMMIT` 주기화(예: 5k)  
- 시퀀스 `CACHE` 크게(1000 이상) → 재귀 호출 감소  
- 파티션 테이블(일/시간)로 **세그먼트 경합 분산**

### 8.4 리포트/집계 배치
- **집계는 가급적 SQL 한 방**(윈도우 함수/집계) → 불필요 페치 제거  
- 불가피하면 `BULK COLLECT LIMIT` + 외부 시스템 전송  
- 임시 결과는 **GTT/임시 테이블**로 합리화

---

## 9. 자주 하는 실수 & 교정

1) **Auto-commit ON**으로 배치 무력화 → **OFF**로 전환, 주기 커밋  
2) IN-list 문자열 조립(동적 SQL 폭증) → **스키마 타입 배열 바인드**/GTT  
3) `BULK COLLECT` 무제한 → PGA 폭증 → **LIMIT** 필수  
4) 대량 DML에 row-by-row 트리거/복잡한 FK → **배치 크기 축소**·**로직 슬림화**  
5) fetchSize만 키우고 **네트워크/ORM 변환** 병목 방치 → DTO/직렬화 최적화 병행

---

## 10. 미니 벤치(요약 스크립트): Row-by-Row vs Array

```plsql
-- Row-by-Row
DECLARE
  v_cnt NUMBER:=0;
BEGIN
  FOR r IN (SELECT sale_id FROM t_sale WHERE region='APAC' AND ROWNUM<=100000) LOOP
    v_cnt:=v_cnt+1;
  END LOOP;
  DBMS_OUTPUT.PUT_LINE('row-by-row='||v_cnt);
END;
/

-- Array (BULK COLLECT LIMIT 5000)
DECLARE
  TYPE t_ids IS TABLE OF NUMBER;
  v_ids t_ids;
  v_cnt NUMBER:=0;
BEGIN
  LOOP
    SELECT sale_id BULK COLLECT INTO v_ids
    FROM   t_sale
    WHERE  region='APAC' AND ROWNUM<=5000;
    EXIT WHEN v_ids.COUNT=0;
    v_cnt:=v_cnt+v_ids.COUNT;
    v_ids.DELETE;
  END LOOP;
  DBMS_OUTPUT.PUT_LINE('array='||v_cnt);
END;
/
```

> TKPROF에서 **Fetch count**, `elapsed` 비교: Array 방식이 왕복/시간 모두 우세.

---

## 11. 보안·무결성·락 관점 주의

- 대량 DML은 **락 유지 시간**이 길어질 수 있음 → **배치 크기 조절**  
- FK/Unique 제약 위반은 **SAVE EXCEPTIONS**로 분기 처리  
- 트리거·감사 로직이 **행당 실행**되면 Array 이득이 줄어듦 → 로직 슬림화/비동기화

---

## 12. 체크리스트 (운영 투입 전)

- [ ] SELECT: **배열 페치 크기** 실측 튜닝(네트워크/메모리 밸런스)  
- [ ] DML: **Batch/Array 크기**·**커밋 주기** 설정, `SAVE EXCEPTIONS` 적용  
- [ ] 시퀀스 `CACHE` 확대, Auto-commit OFF  
- [ ] IN-list는 **배열 바인드/GTT**, 동적 문자열 금지  
- [ ] TKPROF/AWR로 **전/후 CALL 통계** 비교: Fetch/Execute **count↓**, elapsed↓  
- [ ] 에러 수집/재처리 파이프라인 준비 (BatchUpdateException, SQL%BULK_EXCEPTIONS 등)

---

## 13. 결론

Array Processing은 **“적게, 크게, 한 번에”**의 구현 기술이다.  
- **적게**: 호출 수(user calls)를 줄이고,  
- **크게**: 한 번에 다량의 행을 처리하며,  
- **한 번에**: 커서 재사용·배치 커밋으로 서버/네트워크 모두 효율화한다.

PL/SQL의 `BULK COLLECT/FORALL`, 드라이버의 **배열 페치/배열 바인드/배치 실행**을 표준으로 삼으면  
**응답시간·처리량·DB 부하**가 동시에 좋아진다. 항상 **SQL Trace/TKPROF**로 수치 검증하며,  
배열/배치 크기는 **실측**으로 최적화하라 — 그러면 “느린 시스템”은 눈에 띄게 빨라진다.

> **한 줄 요약**  
> **왕복을 줄이고(배열 페치/배치 DML), 실패는 예외 저장으로 분리, IN-list는 배열 바인드.**  
> 이것이 Oracle에서 Array Processing을 제대로 쓰는 방법이다.