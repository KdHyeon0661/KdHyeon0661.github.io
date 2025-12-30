---
layout: post
title: DB 심화 - Array Processing 활용
date: 2025-11-01 19:25:23 +0900
category: DB 심화
---
# Array Processing 활용

> **핵심 요약**  
> Array Processing은 여러 행을 한 번에 처리하는 기법으로, **왕복 횟수를 획기적으로 줄여 성능을 극대화**한다.
> - **SELECT**: `BULK COLLECT` 또는 드라이버의 **배열 페치(Fetch Array)**를 사용해 Fetch 호출 횟수를 줄인다.
> - **DML**: `FORALL` 또는 드라이버의 **배열 바인드/배치 실행**을 사용해 Execute 호출 횟수를 줄인다.
> - **결과**: 네트워크 왕복 감소, `user calls` 감소, `log file sync` 대기 이벤트 및 CPU 사용량 감소로 이어진다.

---

## 실습 환경 구성

```sql
-- 대용량 테이블 생성
DROP TABLE t_sale PURGE;
CREATE TABLE t_sale (
  sale_id     NUMBER PRIMARY KEY,
  cust_id     NUMBER NOT NULL,
  region      VARCHAR2(10),
  order_dt    DATE,
  status      VARCHAR2(10),
  amount      NUMBER(12,2)
);

-- 샘플 데이터 200만 행 삽입
INSERT /*+ APPEND */ INTO t_sale
SELECT level,
       MOD(level, 100000)+1,
       CASE MOD(level,4) WHEN 0 THEN 'APAC' WHEN 1 THEN 'EMEA' WHEN 2 THEN 'AMER' ELSE 'OTHER' END,
       DATE '2025-01-01' + MOD(level, 365),
       CASE WHEN MOD(level,20)=0 THEN 'CANCEL' ELSE 'OK' END,
       ROUND(DBMS_RANDOM.VALUE(10,1000),2)
FROM dual CONNECT BY level <= 2000000;
COMMIT;

-- 인덱스 생성 및 통계 수집
CREATE INDEX ix_sale_region_dt ON t_sale(region, order_dt);
CREATE INDEX ix_sale_cust      ON t_sale(cust_id);
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'T_SALE',cascade=>TRUE,
    method_opt=>'FOR COLUMNS SIZE 75 region SIZE 1 order_dt');
END;
/
```

---

## 성능 모델: 왜 Array Processing인가?

데이터베이스 응답 시간은 단순히 서버 내부의 처리 시간만이 아니라 **네트워크 왕복(Round Trip Time, RTT)에 의한 지연이 크게 기여**합니다.

$$
\text{응답 시간(RT)} \approx \underbrace{\text{RTT} \times N_{\text{calls}}}_{\text{네트워크 왕복 지연}} + \underbrace{\text{CPU} + \text{Waits}}_{\text{서버 내부 처리}}
$$

- **행 단위 처리(Row-by-Row)**: `N_calls`(호출 횟수)가 처리하는 행의 수에 비례하여 증가합니다. 10만 행을 처리하면 최소 10만 번의 네트워크 왕복이 발생합니다.
- **배열 처리(Array Processing)**: `N_calls`가 `행 수 / 배열 크기`로 감소합니다. 배열 크기를 1000으로 설정하면 10만 행 처리 시 왕복은 약 100번으로 급감합니다.
- **결론**: 서버에서 수행하는 실제 작업량은 동일하더라도, **네트워크 왕복 횟수를 줄이는 것만으로도 전체 응답 시간을 획기적으로 개선**할 수 있습니다.

---

## PL/SQL에서의 Array Processing

### 1. BULK COLLECT를 이용한 배열 조회
`BULK COLLECT`는 한 번의 Fetch로 여러 행을 PL/SQL 컬렉션에 담아옵니다. `LIMIT` 절을 활용하면 PGA 메모리 사용량을 제어하면서 최적의 성능을 낼 수 있습니다.

```plsql
DECLARE
  TYPE t_sale_tab IS TABLE OF t_sale%ROWTYPE;
  l_sales t_sale_tab;
  CURSOR cur_sales IS
    SELECT * FROM t_sale WHERE region = 'APAC';
BEGIN
  OPEN cur_sales;
  LOOP
    -- 1000행씩 배열로 페치
    FETCH cur_sales BULK COLLECT INTO l_sales LIMIT 1000;
    EXIT WHEN l_sales.COUNT = 0;

    -- 배열 내 데이터 처리 (예: 로깅, 계산, 외부 전송)
    FOR i IN 1 .. l_sales.COUNT LOOP
      -- 개별 행 처리 로직
      NULL;
    END LOOP;
  END LOOP;
  CLOSE cur_sales;
END;
/
```
**핵심**: `LIMIT` 없이 대량 데이터를 `BULK COLLECT`하면 PGA 메모리를 과도하게 사용할 수 있습니다. 일반적으로 1,000에서 10,000 사이의 값으로 성능 테스트를 통해 최적의 크기를 찾아야 합니다.

### 2. FORALL을 이용한 배열 DML
`FORALL`은 단일 SQL 문을 재사용하면서 컬렉션에 담긴 모든 값에 대해 배열 바인드를 수행합니다. Execute 호출은 단 한 번만 발생합니다.

```plsql
DECLARE
  TYPE t_id_tab IS TABLE OF t_sale.sale_id%TYPE;
  TYPE t_amt_tab IS TABLE OF t_sale.amount%TYPE;
  l_ids t_id_tab;
  l_amts t_amt_tab;
BEGIN
  -- 업데이트할 데이터를 컬렉션에 채움 (예: 5만 행)
  SELECT sale_id, amount * 1.1 -- 10% 인상
  BULK COLLECT INTO l_ids, l_amts
  FROM t_sale
  WHERE region = 'APAC' AND ROWNUM <= 50000;

  -- FORALL로 일괄 업데이트
  FORALL i IN 1 .. l_ids.COUNT
    UPDATE t_sale
    SET amount = l_amts(i)
    WHERE sale_id = l_ids(i);

  DBMS_OUTPUT.PUT_LINE('Updated ' || SQL%ROWCOUNT || ' rows.');
  COMMIT;
END;
/
```

### 3. SAVE EXCEPTIONS으로 부분 실패 처리
대량 업데이트 중 일부 행에서 제약 조건 위반 등의 오류가 발생하더라도 전체 작업이 중단되지 않도록 합니다.

```plsql
DECLARE
  TYPE t_id_tab IS TABLE OF t_sale.sale_id%TYPE;
  l_ids t_id_tab := t_id_tab(1, 2, 999999); -- 존재하지 않는 ID 포함
  l_err_count NUMBER;
BEGIN
  FORALL i IN INDICES OF l_ids SAVE EXCEPTIONS
    DELETE FROM t_sale WHERE sale_id = l_ids(i);

  COMMIT;
EXCEPTION
  WHEN OTHERS THEN
    l_err_count := SQL%BULK_EXCEPTIONS.COUNT;
    DBMS_OUTPUT.PUT_LINE('Number of errors: ' || l_err_count);
    FOR j IN 1 .. l_err_count LOOP
      DBMS_OUTPUT.PUT_LINE('Error ' || j || ': Index ' ||
        SQL%BULK_EXCEPTIONS(j).ERROR_INDEX || ', Code ' ||
        SQL%BULK_EXCEPTIONS(j).ERROR_CODE);
    END LOOP;
    -- 성공한 행은 커밋되었으며, 오류가 발생한 행은 별도로 처리 가능
END;
/
```

### 4. RETURNING BULK COLLECT로 생성된 데이터 회수
INSERT, UPDATE, DELETE 후 영향 받은 행의 정보(예: 새로 생성된 시퀀스 값, ROWID)를 배열로 한 번에 가져올 수 있습니다.

```plsql
DECLARE
  TYPE t_id_tab IS TABLE OF t_sale.sale_id%TYPE;
  TYPE t_rid_tab IS TABLE OF ROWID;
  l_new_ids t_id_tab;
  l_rowids t_rid_tab;
BEGIN
  -- 여러 행 INSERT
  FORALL i IN 1..1000
    INSERT INTO t_sale (sale_id, cust_id, region, order_dt, status, amount)
    VALUES (3000000 + i, MOD(i, 100000)+1, 'APAC', SYSDATE, 'OK', 100)
    RETURNING sale_id, ROWID BULK COLLECT INTO l_new_ids, l_rowids;

  DBMS_OUTPUT.PUT_LINE('Inserted ' || l_new_ids.COUNT || ' rows.');
  COMMIT;
END;
/
```

---

## 애플리케이션(드라이버) 레벨에서의 Array Processing

### JDBC
**배열 페치 (Fetch Array)**
```java
String sql = "SELECT sale_id, amount FROM t_sale WHERE region = ?";
try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
  pstmt.setString(1, "APAC");
  // 한 번의 네트워크 왕복으로 가져올 행 수 설정
  pstmt.setFetchSize(1000);
  try (ResultSet rs = pstmt.executeQuery()) {
    while (rs.next()) {
      // 결과 소비
    }
  }
}
```

**배치 실행 (Batch Execution)**
```java
String sql = "UPDATE t_sale SET amount = ? WHERE sale_id = ?";
try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
  conn.setAutoCommit(false); // Auto-commit 비활성화 필수
  for (int i = 1; i <= 10000; i++) {
    pstmt.setBigDecimal(1, new BigDecimal("150.00"));
    pstmt.setInt(2, i);
    pstmt.addBatch(); // 배치에 작업 추가

    if (i % 1000 == 0) {
      pstmt.executeBatch(); // 1000개 단위로 실행
      conn.commit(); // 주기적 커밋
    }
  }
  pstmt.executeBatch(); // 나머지 실행
  conn.commit();
}
```

### Python (oracledb)
**배열 페치**
```python
import oracledb
connection = oracledb.connect(user="scott", password="tiger", dsn="localhost/orclpdb")
cursor = connection.cursor()

cursor.arraysize = 1000  # 배열 페치 크기 설정
cursor.execute("""
    SELECT sale_id, amount
    FROM t_sale
    WHERE region = :region
""", region="APAC")

for row in cursor:
    # 행 처리
    pass
```

**배치 실행**
```python
data_to_insert = [
    (4000001, 50001, 'EMEA', datetime.date(2025, 10, 1), 'OK', 200.00),
    (4000002, 50002, 'EMEA', datetime.date(2025, 10, 2), 'OK', 250.00),
    # ... 더 많은 데이터
]
cursor.executemany("""
    INSERT INTO t_sale (sale_id, cust_id, region, order_dt, status, amount)
    VALUES (:1, :2, :3, :4, :5, :6)
""", data_to_insert)
connection.commit()
```

---

## 고급 기법: 배열 바인드를 활용한 IN-List 처리

동적으로 수백, 수천 개의 값을 가진 IN-list를 처리할 때 SQL 문을 문자열 조합으로 생성하면 하드 파싱이 심각하게 증가합니다. 배열 바인드를 사용하면 이를 해결할 수 있습니다.

### 1. 스키마 수준 컬렉션 타입 활용
```sql
CREATE OR REPLACE TYPE num_list_t AS TABLE OF NUMBER;
/
```

```plsql
DECLARE
  l_cust_ids num_list_t := num_list_t(101, 205, 307, 409);
BEGIN
  FOR rec IN (
    SELECT * FROM t_sale
    WHERE cust_id IN (SELECT COLUMN_VALUE FROM TABLE(:ids))
  ) LOOP
    DBMS_OUTPUT.PUT_LINE(rec.sale_id);
  END LOOP;
END;
/
-- 애플리케이션에서는 :ids 바인드 변수에 배열을 전달
```

### 2. 임시 테이블 활용 (매우 큰 리스트)
```sql
CREATE GLOBAL TEMPORARY TABLE temp_ids (id NUMBER) ON COMMIT DELETE ROWS;
```
애플리케이션에서 `temp_ids` 테이블에 ID 목록을 벌크 인서트한 후, 이 테이블과 조인하여 원본 데이터를 조회합니다. 이 방법은 최적화기가 통계 정보를 활용해 더 나은 실행 계획을 수립할 수 있는 장점이 있습니다.

---

## 성능 검증: SQL 트레이스 분석

Array Processing 적용 전후의 성능 차이는 SQL 트레이스를 통해 명확하게 확인할 수 있습니다.

```sql
-- 트레이스 활성화
ALTER SESSION SET statistics_level = ALL;
ALTER SESSION SET events '10046 trace name context forever, level 8'; -- 바인드 값 포함

-- 테스트 쿼리 실행 (예: fetchSize=1 vs fetchSize=1000)
SELECT sale_id FROM t_sale WHERE region = 'APAC';

-- 트레이스 종료
ALTER SESSION SET events '10046 trace name context off';
```
생성된 트레이스 파일을 `tkprof` 유틸리티로 분석하면 다음 지표를 중점적으로 비교하십시오.
- **Fetch Calls**: SELECT 시 Fetch 횟수가 배열 크기에 반비례하여 감소해야 합니다.
- **Execute Calls**: DML 시 Execute 횟수가 1에 가까워져야 합니다.
- **Elapsed Time**: 총 소요 시간이 크게 단축되어야 합니다.
- **Network Traffic**: `SQL*Net message to client`, `SQL*Net message from client` 대기 이벤트가 줄어들어야 합니다.

---

## 시나리오별 최적화 가이드라인

### 데이터 마이그레이션/대량 적재
- **INSERT** 시 `/*+ APPEND */` 힌트와 `NOLOGGING` 모드를 고려하세요.
- 인덱스와 트리거는 가능하면 비활성화하고, 작업 완료 후 재생성하세요.
- `FORALL` 또는 드라이버 배치를 사용하고, 10,000행 내외 단위로 커밋하여 Undo 부하와 `log file sync` 대기를 균형 있게 관리하세요.

### 실시간 대량 조회 API
- 페이지네이션 시 **Keyset Pagination**(마지막 조회 값 기준)을 사용하세요. OFFSET은 성능에 불리합니다.
- 조회 컬럼을 필요한 최소한으로 제한하고, `WHERE` 절은 인덱스를 탈 수 있도록 설계하세요.
- 네트워크 지연이 큰 환경(예: 다른 리전의 DB 접속)에서는 배열 페치 크기를 더 크게 설정하는 것이 유리합니다.

### 트랜잭션 로그 적재
- 배열 바인드를 이용한 INSERT를 사용하세요.
- 시퀀스를 사용한다면 `CACHE` 크기를 충분히(예: 1000 이상) 늘리세요.
- 테이블을 일별/시간별 파티셔닝하여 핫 블록 경합을 분산시키세요.

---

## 주의사항과 자주 하는 실수

1.  **PGA 메모리 고려**: `BULK COLLECT`를 무제한으로 사용하면 PGA 메모리가 고갈될 수 있습니다. 항상 `LIMIT`과 함께 사용하세요.
2.  **트랜잭션 크기 관리**: 너무 큰 배치 크기로 한 번에 커밋하면 롤백 세그먼트 부족(`ORA-01555`)이나 장시간 로크 유지로 인한 경합이 발생할 수 있습니다. 적절한 배치 크기(예: 5,000 - 20,000)를 유지하세요.
3.  **Auto-Commit 비활성화**: 애플리케이션 레벨 배치 실행 시 Auto-Commit 모드를 반드시 끄고, 명시적인 주기적 커밋을 구현하세요. 행마다 커밋하면 배치 실행의 이점이 사라집니다.
4.  **예외 처리 설계**: `SAVE EXCEPTIONS`나 드라이버의 배치 예외 처리 기능을 활용하여 일부 행의 실패가 전체 작업을 중단시키지 않도록 하세요. 오류 난 행은 별도 로깅 후 재처리하도록 합니다.
5.  **실행 계획 변화**: 배열 바인드를 이용한 IN-list 처리 등 새로운 방식은 때로는 다른 실행 계획을 유도할 수 있습니다. 변경 후 반드시 실행 계획과 성능을 검증하세요.

---

## 결론

Array Processing은 데이터베이스 연산의 효율성을 높이는 **근본적이고 강력한 패러다임**입니다. 핵심 원리는 간단합니다. **네트워크 왕복이라는 가장 비싼 비용을 줄이는 것**입니다.

PL/SQL의 `BULK COLLECT`와 `FORALL`, 그리고 각 언어 드라이버가 제공하는 배열 페치와 배치 실행 기능은 이 원리를 구현하는 표준 도구입니다. 이러한 기법을 적용하면 동일한 비즈니스 로직을 수행하더라도 애플리케이션의 응답 속도가 빨라지고, 데이터베이스 서버의 CPU 및 I/O 부하가 감소하며, 결과적으로 시스템 전체의 확장성이 향상됩니다.

새로운 기능을 개발하거나 기존 성능 개선 작업을 할 때, "이 작업을 한 번에 더 많은 행을 처리할 수는 없을까?"라고 항상 질문하십시오. 작은 습관이 쌓여 결국 견고하고 빠른 시스템을 구축하는 토대가 될 것입니다.