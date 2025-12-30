---
layout: post
title: DB 심화 - Fetch Call 최소화
date: 2025-11-01 20:25:23 +0900
category: DB 심화
---
# Array Fetch로 성능을 극대화하는 방법

> **핵심 요약**
> - **Fetch Call**은 SELECT 결과를 클라이언트로 가져오는 네트워크 왕복 작업입니다.
> - **부분범위처리(PRP, Partial Range Processing)** 를 활용하면 필요한 상위 N행만 읽어 I/O를 크게 줄일 수 있습니다.
> - **ArraySize(배열 페치 크기)** 를 증가시키면 Fetch 왕복 횟수가 감소하여 네트워크 오버헤드와 컨텍스트 스위칭 비용이 크게 절감됩니다.
> - OLTP 환경에서 **PRP와 Array Fetch의 조합**은 응답 시간을 획기적으로 개선하는 가장 효과적인 방법 중 하나입니다.

---

## 실습 환경 구성

```sql
-- 대용량 샘플 테이블 생성
DROP TABLE orders PURGE;

CREATE TABLE orders (
  order_id     NUMBER PRIMARY KEY,
  customer_id  NUMBER NOT NULL,
  order_dt     DATE   NOT NULL,
  status       VARCHAR2(10) NOT NULL,
  amount       NUMBER(12,2) NOT NULL
);

-- 샘플 데이터 200만 행 삽입
INSERT /*+ APPEND */ INTO orders
SELECT level,
       MOD(level, 500000) + 1,
       (TRUNC(SYSDATE) - MOD(level, 365)) + (MOD(level, 86400) / 86400),
       CASE WHEN MOD(level, 11)=0 THEN 'CANCEL' ELSE 'OK' END,
       ROUND(DBMS_RANDOM.VALUE(10, 1000), 2)
FROM dual CONNECT BY level <= 2000000;
COMMIT;

-- 부분범위처리와 필터링을 지원하는 인덱스 생성
CREATE INDEX ix_orders_cust_dt  ON orders(customer_id, order_dt DESC);
CREATE INDEX ix_orders_status   ON orders(status);

-- 통계 수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'ORDERS', cascade=>TRUE);
END;
/
```

---

## Fetch Call의 중요성과 성능 영향

### 실행 단계와 Fetch Call의 관계
데이터베이스 쿼리 실행은 일반적으로 세 단계로 이루어집니다:
1. **Parse**: SQL 문을 분석하고 최적화된 실행 계획을 생성
2. **Execute**: 실행 계획을 시작하고 필요한 리소스를 할당
3. **Fetch**: 결과를 클라이언트로 전송

Fetch 단계는 결과 집합을 작은 단위(배열)로 나누어 전송하며, 각 전송이 하나의 **Fetch Call**을 의미합니다. 이는 네트워크 왕복을 수반하므로 성능에 직접적인 영향을 미칩니다.

### Fetch Call 수 계산 공식
Fetch Call 횟수는 다음 공식으로 근사할 수 있습니다:

$$
N_{\text{fetch calls}} \approx \left\lceil \frac{\text{전송할 행 수}}{\text{ArraySize}} \right\rceil
$$

- **ArraySize 증가** → Fetch Call 감소 → **네트워크 왕복 지연 누적 감소**
- 동일한 행 수를 전송하더라도 Fetch Call이 줄어들면 **전체 경과 시간이 크게 단축**됩니다.

### 부분범위처리와 ArraySize의 차이점
두 개념은 성능 개선에 서로 다른 방식으로 기여합니다:
- **부분범위처리(PRP)**: 데이터베이스 서버가 **읽어야 할 행 수 자체를 줄여** I/O 작업량을 감소시킵니다.
- **ArraySize**: **네트워크 왕복 횟수를 줄여** 호출 오버헤드를 감소시킵니다.

이 두 기법을 함께 적용하면 효과가 시너지로 발휘됩니다.

---

## 부분범위처리(PRP)의 원리와 구현

### 개념적 이해
부분범위처리는 "전체 결과 집합을 모두 읽은 후 필요한 부분만 추출"하는 대신, **필요한 상위 행만 읽고 즉시 작업을 중단**하는 방식입니다. Oracle 실행 계획에서는 **`STOPKEY`** 연산자로 표시되며, 일반적으로 다음 조건과 결합됩니다:
- `FETCH FIRST N ROWS ONLY` 또는 `ROWNUM <= N` 구문 사용
- 적절한 인덱스를 통한 **Index Range Scan**
- 가능한 경우 **커버링 인덱스**로 테이블 접근까지 회피

### PRP가 효과적인 조건
부분범위처리가 효과적으로 작동하려면 다음 조건을 충족해야 합니다:

1. **WHERE 조건과 ORDER BY 절이 동일한 복합 인덱스로 처리 가능**해야 합니다.
2. **정렬 방향이 인덱스 순서와 일치**해야 합니다(예: `order_dt DESC` 인덱스에 `ORDER BY order_dt DESC`).
3. **필요한 모든 컬럼이 인덱스에 포함**되면 최적의 성능을 발휘합니다(커버링 인덱스).
4. TOP-N 쿼리나 Keyset 페이지네이션에 적합합니다.

### 실전 예제: 고객의 최근 주문 20건 조회

```sql
-- 고객 ID 12345의 최신 주문 20건 조회
SELECT /*+ index(o ix_orders_cust_dt) */
       o.order_id, o.order_dt, o.amount, o.status
FROM   orders o
WHERE  o.customer_id = 12345
ORDER  BY o.order_dt DESC
FETCH FIRST 20 ROWS ONLY;
```

**이 쿼리의 장점:**
1. `customer_id = 12345` 조건으로 인덱스 범위 스캔 시작
2. `order_dt DESC` 정렬을 인덱스의 내림차순 정렬로 해결
3. 상위 20건만 읽은 후 즉시 스캔 중단 → **불필요한 I/O 제거**

**실행 계획 특징:**
- `INDEX RANGE SCAN ix_orders_cust_dt` (조건: `customer_id=12345`)
- `STOPKEY` (Fetch First 20)
- 필요한 컬럼이 인덱스에 없으면 `TABLE ACCESS BY ROWID` 추가 수행

### PRP가 실패하는 일반적인 패턴

```sql
-- 비효율적 예제: 정렬 조건이 인덱스로 해결되지 않음
SELECT order_id, order_dt, amount
FROM   orders
WHERE  customer_id = 12345
ORDER  BY amount DESC
FETCH FIRST 20 ROWS ONLY;
```

**문제점:**
- 인덱스가 `(customer_id, order_dt)`로만 구성된 경우 `amount DESC` 정렬을 위해 전체 결과 정렬 필요
- **대부분 또는 전체 행을 읽어야 하므로 STOPKEY의 이점 상실**
- **해결책**: 업무 요구사항을 재검토하거나 `(customer_id, amount DESC)` 인덱스 추가 고려

---

## OLTP 환경에서의 PRP 효과

OLTP(Online Transaction Processing) 시스템은 **짧은 트랜잭션이 매우 빈번하게 발생**하는 환경입니다. 이런 환경에서 부분범위처리는 다음과 같은 다각적인 이점을 제공합니다:

1. **I/O 부하 감소**: 실제 필요한 행만 읽으므로 물리적/논리적 I/O가 크게 줄어듭니다.
2. **락 경합 완화**: 적은 데이터 접근은 동시성 문제와 래치 경합을 감소시킵니다.
3. **CPU 및 메모리 효율**: 정렬, 해시 조인, 필터링 등 중간 처리 작업이 최소화됩니다.
4. **캐시 효율성 향상**: 좁은 인덱스 범위 스캔은 버퍼 캐시 히트율을 높입니다.
5. **사용자 경험 개선**: 첫 페이지 결과를 즉시 반환하여 체감 응답 시간을 개선합니다.

부분범위처리는 근본적인 작업량 자체를 줄이는 기술이며, ArraySize는 왕복 오버헤드를 줄이는 기술입니다. OLTP 환경에서 이 두 기술의 조합은 가장 강력한 성능 개선 조합 중 하나입니다.

---

## I/O 및 네트워크 최적화 효과

### Fetch Call 감소의 수학적 효과
ArraySize를 증가시키면 Fetch Call 횟수가 선형적으로 감소합니다. 예를 들어:
- 10,000행을 ArraySize=50으로 처리 → **200회** Fetch Call
- 10,000행을 ArraySize=1000으로 처리 → **10회** Fetch Call

네트워크 왕복 시간(RTT)이 평균 3ms인 환경에서는 이 차이만으로 **570ms(0.57초)** 의 순수 네트워크 지연을 절감할 수 있습니다.

### I/O에 대한 미묘한 영향
ArraySize 증가 자체가 직접적으로 블록 I/O를 줄이지는 않습니다. 동일한 행 수를 읽는다면 논리적/물리적 I/O는 기본적으로 동일합니다. 그러나 다음과 같은 간접적 이점이 있습니다:

- **스캔 연속성 유지**: 왕복이 줄어들어 데이터베이스의 스캔 모멘텀이 유지됩니다.
- **컨텍스트 스위칭 감소**: 서버와 네트워크 스택의 오버헤드가 감소합니다.
- **대기 이벤트 감소**: `SQL*Net message to/from client` 대기 이벤트가 줄어듭니다.
- **경합 완화**: 락과 래치 재진입 횟수가 감소합니다.

**명확한 I/O 감소**는 부분범위처리, 커버링 인덱스, 효율적인 필터링과 정렬 인덱스에서 비롯됩니다.

---

## 프로그래밍 언어별 Array Fetch 구현

### JDBC (Oracle JDBC Thin 드라이버)
```java
String sql = """
  SELECT order_id, order_dt, amount, status
  FROM orders
  WHERE customer_id = ?
  ORDER BY order_dt DESC
  FETCH FIRST 20 ROWS ONLY
""";

try (PreparedStatement ps = conn.prepareStatement(sql)) {
  ps.setInt(1, 12345);
  // 부분범위처리가 적용된 경우 100-500으로 충분
  // 일반 대량 조회의 경우 500-2000 권장
  ps.setFetchSize(200);
  
  try (ResultSet rs = ps.executeQuery()) {
    while (rs.next()) {
      // 결과 처리
      int orderId = rs.getInt("order_id");
      Date orderDate = rs.getDate("order_dt");
      // ... 추가 처리
    }
  }
}
```

**추가 팁:**
- 대용량 데이터 스트리밍 시 `setFetchSize(1000~5000)`로 시작하여 실측 최적화
- Statement Cache를 활성화하여 Parse 오버헤드 감소
- Auto-commit 모드 비활성화 및 명시적 트랜잭션 관리

### Python (python-oracledb)
```python
import oracledb

connection = oracledb.connect(user="user", password="password", dsn="dsn")
cursor = connection.cursor()

# 배열 페치 크기 설정 (기본값 100보다 크게 설정)
cursor.arraysize = 1000

cursor.execute("""
  SELECT order_id, order_dt, amount, status
  FROM orders
  WHERE customer_id = :cust_id
  ORDER BY order_dt DESC
  FETCH FIRST 20 ROWS ONLY
""", cust_id=12345)

# 모든 결과 한 번에 가져오기 (부분범위처리로 인해 20행만 존재)
rows = cursor.fetchall()
for row in rows:
    order_id, order_dt, amount, status = row
    # 처리 로직
```

### C# (ODP.NET)
```csharp
using Oracle.ManagedDataAccess.Client;

using (var cmd = new OracleCommand(@"
  SELECT order_id, order_dt, amount, status
  FROM orders
  WHERE customer_id = :cust_id
  ORDER BY order_dt DESC
  FETCH FIRST 20 ROWS ONLY", connection))
{
    cmd.Parameters.Add(":cust_id", OracleDbType.Int32).Value = 12345;
    
    // ODP.NET은 바이트 단위의 FetchSize 사용
    // 행당 평균 크기를 추정하여 4-16MB 범위에서 시작
    cmd.FetchSize = 8 * 1024 * 1024; // 8MB
    
    using (var reader = cmd.ExecuteReader())
    {
        while (reader.Read())
        {
            int orderId = reader.GetInt32(0);
            DateTime orderDate = reader.GetDateTime(1);
            decimal amount = reader.GetDecimal(2);
            string status = reader.GetString(3);
            // 처리 로직
        }
    }
}
```

### Node.js (node-oracledb)
```javascript
const result = await connection.execute(
  `
  SELECT order_id, order_dt, amount, status
  FROM orders
  WHERE customer_id = :cust_id
  ORDER BY order_dt DESC
  FETCH FIRST 20 ROWS ONLY
  `,
  { cust_id: 12345 },
  { 
    prefetchRows: 500,      // 서버가 미리 가져오는 행 수
    fetchArraySize: 1000    // 클라이언트가 한 번에 받는 행 수
  }
);

for (const row of result.rows) {
  const [orderId, orderDate, amount, status] = row;
  // 처리 로직
}
```

---

## 효율적인 SQL 패턴과 인덱스 설계

### 기본 리스트 조회 패턴
```sql
SELECT /*+ index(o ix_orders_cust_dt) */
       o.order_id, o.order_dt, o.amount, o.status
FROM   orders o
WHERE  o.customer_id = :cust_id
ORDER  BY o.order_dt DESC
FETCH FIRST :page_size ROWS ONLY;
```
**인덱스 전략:** `(customer_id, order_dt DESC)`
**최적화:** 필요한 모든 컬럼을 인덱스에 포함시켜 커버링 인덱스로 변환

### Keyset 페이지네이션 (무한 스크롤)
```sql
-- 마지막으로 본 행(last_dt, last_id) 이후의 다음 20행
SELECT order_id, order_dt, amount, status
FROM   orders
WHERE  customer_id = :cust_id
  AND (order_dt, order_id) < (:last_dt, :last_id)  -- 튜플 비교
ORDER  BY order_dt DESC, order_id DESC
FETCH FIRST 20 ROWS ONLY;
```
**인덱스 전략:** `(customer_id, order_dt DESC, order_id DESC)`
**장점:** OFFSET 방식보다 효율적(앞부분 행을 읽고 버리지 않음)

### 필터링이 포함된 조회
```sql
SELECT /*+ index(o (customer_id, status, order_dt)) */
       o.order_id, o.order_dt, o.amount
FROM   orders o
WHERE  o.customer_id = :cust_id
  AND  o.status = 'OK'
ORDER  BY o.order_dt DESC
FETCH FIRST 20 ROWS ONLY;
```
**인덱스 선택 고려사항:**
- `(customer_id, status, order_dt DESC)`: 상태 필터링에 최적
- `(customer_id, order_dt DESC, status)`: 범위 스캔에 최적
- 실제 데이터 분포와 선택도를 기준으로 결정

---

## 성능 검증 방법론

### SQL 트레이스 활성화
```sql
-- 세션 통계 수집 레벨 설정
ALTER SESSION SET statistics_level = ALL;

-- 세부 트레이스 활성화 (바인드 변수 값 포함)
ALTER SESSION SET events '10046 trace name context forever, level 8';

-- 테스트 쿼리 실행
SELECT order_id, order_dt, amount, status
FROM orders
WHERE customer_id = 12345
ORDER BY order_dt DESC
FETCH FIRST 20 ROWS ONLY;

-- 트레이스 비활성화
ALTER SESSION SET events '10046 trace name context off';
```

### TKPROF 분석 비교
```bash
# 트레이스 파일 변환
tkprof test.trc output_a.txt sys=no sort=prsela,exeela,fchela

# 비교 포인트
# 1. Fetch Call 횟수 감소 확인
# 2. 경과 시간(Elapsed Time) 감소 확인
# 3. SQL*Net 메시지 전송량 대비 행 수 증가 확인
# 4. 논리적/물리적 I/O 감소 확인(부분범위처리 적용 시)
```

### 모니터링 쿼리
```sql
-- 현재 세션의 Fetch Call 통계 확인
SELECT sid, serial#, sql_id, fetches, executions,
       ROUND(fetches/NULLIF(executions,0), 2) as fetches_per_exec
FROM v$sqlstats
WHERE sql_id = 'your_sql_id_here';
```

---

## 일반적인 실패 패턴과 해결책

### 1. OFFSET 페이지네이션 사용
**문제점:** `OFFSET :skip FETCH :take` 패턴은 앞부분 행을 모두 읽고 버리므로 부분범위처리 불가
**해결책:** **Keyset 페이지네이션**으로 전환하여 마지막 조회 값 기준으로 다음 페이지 조회

### 2. 정렬 컬럼이 인덱스에 없음
**문제점:** ORDER BY 절의 컬럼이 인덱스에 포함되지 않아 전체 결과 정렬 필요
**해결책:** 정렬 기준을 포함한 **복합 인덱스** 설계 또는 요구사항 재검토

### 3. 선행 와일드카드 LIKE 패턴
**문제점:** `LIKE '%검색어'` 패턴은 인덱스 스캔 불가
**해결책:** 역색인, 전문 검색 인덱스 또는 검색 엔진 도입 고려

### 4. 필요한 모든 컬럼이 인덱스에 없음
**문제점:** 커버링 인덱스가 아니어서 테이블 재접근 필요
**해결책:** 자주 조회되는 컬럼을 인덱스에 포함시켜 커버링 인덱스 구성

### 5. 기본 ArraySize 사용
**문제점:** 드라이버 기본값(예: JDBC 10, Python oracledb 100)으로 인한 과다한 Fetch Call
**해결책:** 애플리케이션 특성에 맞는 최적의 ArraySize 설정

### 6. ORM N+1 문제
**문제점:** 연관된 객체를 별도 쿼리로 조회하여 호출 수 폭증
**해결책:** JOIN 또는 배열 IN 바인드를 사용한 일괄 조회로 변경

---

## 실전 시나리오: 주문 관리 시스템 성능 개선

### 문제 상황
고객 포털의 "주문 목록" 첫 페이지 응답 시간이 p95 기준 1.1초로 지연 발생

### 원인 분석
1. `OFFSET 0 FETCH 50` 방식의 페이지네이션 사용
2. 정렬을 위한 적절한 인덱스 부재
3. JDBC FetchSize 기본값(10) 사용
4. 불필요한 컬럼까지 조회

### 개선 조치
1. **인덱스 설계:** `(customer_id, order_dt DESC, order_id DESC)` 복합 인덱스 생성
2. **SQL 패턴 변경:** `FETCH FIRST 50 ROWS ONLY`로 전환
3. **ArraySize 조정:** `setFetchSize(1000)` 설정
4. **컬럼 최소화:** 실제 표시에 필요한 컬럼만 조회
5. **커버링 인덱스 고려:** 자주 조회되는 컬럼을 인덱스에 포함

### 개선 결과
- **Fetch Call 횟수:** 22회 → 1-2회
- **실행 시간:** 약 1.1초 → 약 90ms
- **I/O 작업량:** 85% 감소
- **CPU 사용량:** 70% 감소

---

## 성능 측정 지표

### Fetch 효율성 측정
$$
\text{행당 Fetch 효율} = \frac{\text{총 결과 행 수}}{\text{Fetch Call 횟수}}
$$
이 값이 클수록 배열 페치가 효과적으로 작동하고 있음을 의미합니다.

### 네트워크 왕복 비용 추정
$$
\text{순수 네트워크 지연} \approx \text{RTT} \times N_{\text{fetch calls}}
$$
네트워크 왕복 시간(RTT)이 2-5ms인 환경에서 수백 번의 Fetch Call은 수백 ms의 순수 지연을 발생시킵니다.

### 부분범위처리 I/O 절감 효과
$$
\text{I/O 절감률} \approx 1 - \frac{\text{부분범위처리 시 읽은 블록 수}}{\text{전체 조회 시 읽은 블록 수}}
$$
효율적인 인덱스와 정렬 전략은 이 절감률을 크게 높입니다.

---

## 결론

Array Fetch와 부분범위처리는 Oracle 데이터베이스 성능 최적화의 핵심 기법입니다. 이 두 기술을 효과적으로 결합하면 다음과 같은 이점을 얻을 수 있습니다:

1. **근본적인 성능 개선**: 부분범위처리는 읽어야 할 데이터 양 자체를 줄여 I/O, CPU, 메모리 사용량을 감소시킵니다.
2. **네트워크 효율화**: ArraySize 증가는 네트워크 왕복 횟수를 줄여 응답 시간을 단축합니다.
3. **시스템 확장성 향상**: 적은 자원으로 더 많은 트랜잭션을 처리할 수 있게 합니다.
4. **예측 가능한 성능**: 일관된 응답 시간을 제공하여 사용자 경험을 개선합니다.

성공적인 구현을 위해서는 인덱스 설계, SQL 패턴, 애플리케이션 설정이 조화를 이루어야 합니다. 항상 실제 환경에서 성능 테스트를 수행하고, SQL 트레이스와 실행 계획 분석을 통해 개선 효과를 검증하십시오.

**최종 조언**: 모든 성능 개선 작업은 "측정-분석-개선-검증"의 순환 과정을 따르세요. 부분범위처리와 Array Fetch는 강력한 도구이지만, 올바르게 적용되었는지 지속적으로 모니터링하고 튜닝해야 지속적인 성능 향상을 기대할 수 있습니다.