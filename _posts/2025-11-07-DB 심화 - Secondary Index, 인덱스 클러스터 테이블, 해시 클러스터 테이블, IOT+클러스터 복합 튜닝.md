---
layout: post
title: DB 심화 - Secondary Index, 인덱스 클러스터 테이블, 해시 클러스터 테이블, IOT+클러스터 복합 튜닝
date: 2025-11-07 20:25:23 +0900
category: DB 심화
---
# Secondary Index, 인덱스 클러스터 테이블, 해시 클러스터 테이블, IOT+클러스터 복합 튜닝 사례

## 0. 공통 준비

실습 전체에서 사용할 세션 설정은 다음과 같다.

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';
ALTER SESSION SET statistics_level = ALL; -- ALLSTATS LAST 보기
```

- `nls_date_format` : 실행 결과를 눈으로 확인하기 쉽게 날짜 형식을 고정.
- `statistics_level = ALL` : `DBMS_XPLAN.DISPLAY_CURSOR(..., 'ALLSTATS LAST')` 사용 시  
  **실제 수행 통계(버퍼 읽기, 물리 I/O, 메모리 등)** 를 볼 수 있게 한다.

실제 튜닝에서는 다음 패턴으로 실행계획을 항상 확인하는 습관을 들이는 것이 좋다.

```sql
-- 방금 실행한 SQL의 실제 통계
SELECT *
FROM   TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(
    NULL, NULL,
    'ALLSTATS LAST +PREDICATE +PEEKED_BINDS +IOSTATS +MEMSTATS'
  )
);
```

---

## 1. Heap vs IOT의 의미 차이와 Secondary Index

### 1.1 Secondary Index란?

- **테이블을 식별하는 기본 키(Primary Key) 인덱스 외**에 추가로 만드는 모든 인덱스.
- 목적:
  - **비PK 조건**(검색/정렬/조인)을 빠르게 처리
  - **커버링(Covering)** : 필요한 컬럼을 인덱스에 모두 포함해서 **테이블 미방문**
  - 정렬/Top-N/페이지네이션을 **인덱스에서 직접 처리**

일반적인 미국·유럽권 OLTP 시스템(전자상거래 주문, 온라인 뱅킹 거래, 물류 관리 등)에서

- PK: `order_id`, `account_id`, `txn_id` 등 서러게이트 키
- Secondary Index: `customer_id + order_date`, `status`, `product_id` 등  
  비PK 컬럼 조합으로 여러 개를 운영한다.

Secondary Index를 만들 때 항상 생각해야 할 질문:

1. **“이 인덱스가 없으면 어떤 쿼리가 느려지는가?”**
2. **“이 인덱스를 만들면 어떤 쿼리는 테이블을 더 이상 안 읽어도 되는가?”**
3. **“이 인덱스가 추가되면 DML(INSERT/UPDATE/DELETE) 비용과 공간 비용이 얼마나 느는가?”**

---

### 1.2 Heap과 IOT에서의 Secondary Index 차이

#### 1.2.1 Heap 테이블

Heap 테이블은 **물리적인 행 위치**(블록 번호 + 슬롯 번호)를 의미하는 **ROWID** 기반으로 저장된다.

- Heap 테이블의 보조 인덱스 엔트리 구조:

  ```
  (인덱스 키들..., ROWID)
  ```

- 인덱스 스캔→ROWID로 바로 **테이블 블록**에 접근한다.
- 즉, Secondary Index는

  1. 인덱스 구조(B-tree)에서 키를 찾아서
  2. ROWID를 읽고
  3. 해당 블록/슬롯으로 바로 점프

하는 “**바로가기 포인터**” 역할을 한다.

#### 1.2.2 IOT(Indexed Organized Table)

IOT는 **테이블 자체가 PK 인덱스 구조로 저장**된다.  
즉, “PK 인덱스 = 테이블”인 구조이다.

- IOT의 PK 인덱스 엔트리 구조:

  ```
  (PK 컬럼들..., 나머지 컬럼들...)
  ```

- IOT의 Secondary Index 엔트리 구조:

  ```
  (보조 키들..., 논리 ROWID = PK 값)
  ```

- Secondary Index에서 읽는 “ROWID”는 실제로는 **PK 컬럼들의 조합**이다.
- 따라서 보조 인덱스 스캔 후에는 다시

  1. IOT의 PK 인덱스를 **한 번 더 탐색**(논리 ROWID = PK로 Lookup)
  2. 거기서 실제 레코드를 읽는다.

이 차이 때문에:

- Heap:
  - Secondary Index → 테이블 ROWID로 **직접 점프** → 한 번의 인덱스 탐색
- IOT:
  - Secondary Index → PK(논리 ROWID) 획득 → IOT PK 인덱스 **재탐색** → 실제 레코드

즉, **IOT에서 Secondary Index 경유 조회는 Heap보다 한 단계 더 많다**.

#### 1.2.3 IOT에서 커버링 Secondary Index의 의미

IOT 환경에서 비PK 조건 질의가 많다면,

- Secondary Index에 **필요 컬럼을 모두 포함해서 커버링**하면
  - PK 재탐색 없이 **보조 인덱스만** 읽고 결과를 만들 수 있다.
  - 이 경우 Secondary Index의 “논리 ROWID”를 실제로 사용할 필요가 없어진다.

즉, IOT에서 커버링 Secondary Index는

- Heap의 커버링 인덱스보다
- **추가 재탐색(논리 ROWID→PK 인덱스) 비용을 통째로 제거**한다는 점에서 더욱 중요해진다.

---

### 1.3 Secondary Index 설계 체크포인트

다시 정리하면, Secondary Index 설계 시 다음 항목을 반드시 고려해야 한다.

1. **선행 컬럼 선택도 & 업무 패턴 일치**
   - WHERE/JOIN에서 자주 쓰이는 컬럼을 선행에,  
     그 컬럼 조건을 가능한 **등치(=)** 로 사용하도록 쿼리 패턴을 맞춘다.
2. **정렬/Top-N 흡수**
   - `ORDER BY col1 [DESC], col2 [DESC]` 가 잦다면  
     인덱스 `(col1 DESC, col2 DESC, ...)` 를 두어 Sort를 제거하거나 최소화.
3. **커버링**
   - SELECT-LIST에 나오는 컬럼을 인덱스에 포함시켜  
     `TABLE ACCESS BY INDEX ROWID` 단계를 제거할 수 있는지 검토.
   - Oracle에는 `INCLUDE` 절이 없으므로,  
     “키 컬럼 뒤에 추가 컬럼”으로 넣어서 해결.
4. **인덱스의 수를 최소화**
   - Secondary Index 하나당 INSERT/UPDATE/DELETE 시 **추가 작업**이 필요.
   - 너무 많은 인덱스는 DML 성능과 락 경합, 공간 비용을 악화시킨다.
   - “**핵심 질의** 중심으로 설계”하는 것이 중요하다.

---

## 2. Secondary Index 실습 — 커버링과 Top-N 튜닝

### 2.1 샘플 스키마 (미국 전자상거래 주문 테이블 가정)

```sql
DROP TABLE s_order PURGE;

CREATE TABLE s_order (
  order_id   NUMBER       NOT NULL,
  cust_id    NUMBER       NOT NULL,
  order_dt   DATE         NOT NULL,
  status     VARCHAR2(8)  NOT NULL,
  amount     NUMBER(12,2) NOT NULL,
  note       VARCHAR2(200),
  CONSTRAINT pk_s_order PRIMARY KEY (order_id)
);

BEGIN
  FOR i IN 1..300000 LOOP
    INSERT INTO s_order
    VALUES (
      i,
      MOD(i, 60000) + 1,                    -- 미국 고객 6만명 정도 가정
      DATE '2024-01-01' + MOD(i, 365),      -- 1년 간 주문
      CASE MOD(i,5) WHEN 0 THEN 'NEW' WHEN 1 THEN 'PAID'
                    WHEN 2 THEN 'SHIP' WHEN 3 THEN 'DONE' ELSE 'CANC' END,
      ROUND(DBMS_RANDOM.VALUE(10, 50000),2),
      CASE WHEN MOD(i,97)=0 THEN 'gift' END
    );
  END LOOP;
  COMMIT;
END;
/

-- 비PK 조건 인덱스
CREATE INDEX ix_so_cdt     ON s_order(cust_id, order_dt, order_id);
CREATE INDEX ix_so_status  ON s_order(status);
CREATE INDEX ix_so_dt_desc ON s_order(order_dt DESC, order_id DESC);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'S_ORDER',
    cascade    => TRUE,
    method_opt => 'for all columns size skewonly'
  );
END;
/
```

설명:

- PK `pk_s_order(order_id)` : Heap 테이블 기반 PK 인덱스.
- `ix_so_cdt(cust_id, order_dt, order_id)` : 고객별·기간별 조회에 최적.
- `ix_so_status(status)` : 상태별 집계/필터.
- `ix_so_dt_desc(order_dt DESC, order_id DESC)` : 전체 주문에서 **최신 N건** 조회용.

---

### 2.2 커버링 없는 보조 인덱스 → 테이블 랜덤 액세스

#### 2.2.1 예제 쿼리

```sql
VAR cid NUMBER;
EXEC :cid := 12345;

SELECT order_id, order_dt, status, amount
FROM   s_order
WHERE  cust_id  = :cid
AND    order_dt >= SYSDATE - 30
ORDER  BY order_dt, order_id;

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE')
);
```

- 이 쿼리는 `ix_so_cdt` 를 타는 것이 자연스럽다.
- 하지만 `ix_so_cdt` 에는 `status`, `amount` 컬럼이 없다.
- 따라서 실행계획에는 다음 단계가 등장한다.

  - `INDEX RANGE SCAN ix_so_cdt`
  - `TABLE ACCESS BY INDEX ROWID s_order`

즉, 인덱스에서 `order_id` 를 얻은 뒤  
각 Row에 대해 테이블 블록으로 **랜덤 점프**를 수행한다.

#### 2.2.2 비용 구조

대략적으로,

- 인덱스 스캔에서 **N개의 인덱스 엔트리**를 읽고
- 그 중 일부 또는 전부에 대해 **N번의 ROWID 테이블 접근**이 발생한다.

랜덤 I/O 관점에서 보면,  
같은 고객의 주문이 테이블에서 어느 정도 **군집화**되어 있어도

- 인덱스와 테이블 물리 순서가 다를 수 있고,
- 테이블은 PK `order_id` 순서에 따라 저장되기 때문에

인덱스 스캔이 **테이블 블록 접근을 최적화해 주지는 않는다**.

---

### 2.3 커버링 Secondary Index — 테이블 미방문으로 변경

커버링 인덱스를 만들어 보자.

```sql
-- 커버링 인덱스: SELECT-LIST 포함
CREATE INDEX ix_so_cdt_cov
ON s_order(cust_id, order_dt, order_id, status, amount);

BEGIN
  DBMS_STATS.GATHER_INDEX_STATS(USER,'IX_SO_CDT_COV');
END;
/
```

다시 같은 쿼리를 실행한다.

```sql
SELECT order_id, order_dt, status, amount
FROM   s_order
WHERE  cust_id  = :cid
AND    order_dt >= SYSDATE - 30
ORDER  BY order_dt, order_id;

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE')
);
```

이제 실행계획이 다음과 같이 바뀔 가능성이 크다.

- `INDEX RANGE SCAN IX_SO_CDT_COV`
- **`TABLE ACCESS BY INDEX ROWID` 단계 없음**

즉, 인덱스에서 **모든 필요한 컬럼**을 가져올 수 있으므로  
테이블 블록에 갈 필요가 없어졌다.

랜덤 I/O 비용은

- Heap에서는 “인덱스 블록만” 읽으면 끝난다.
- IOT였다면, “Secondary Index → PK 재탐색”까지 제거되기 때문에  
  IOT 환경에서는 커버링 Secondary Index의 이점이 더 크다.

#### 2.3.1 DML 비용과의 트레이드오프

물론 `ix_so_cdt_cov`는 컬럼이 많아 **인덱스 엔트리 크기**가 크다.

- INSERT: 인덱스에 더 많은 정보 기록 → I/O 증가
- UPDATE: `status`, `amount` 변경 시 인덱스에도 반영 필요
- DELETE: 삭제 시 인덱스에서도 삭제

즉, **읽기 성능을 올리는 대신** DML 성능과 공간을 희생하는 구조이다.  
실제로 미국/유럽의 전자상거래 시스템에서는

- “주문 조회 API”가 매우 중요하면서  
- “주문 완료 후 변경 가능성이 낮아지는 컬럼”이라면

이처럼 **읽기 최적화 인덱스**를 추가하는 패턴이 자주 쓰인다.

---

### 2.4 최신순 Top-N 튜닝 — Descending 인덱스 + Stopkey

최신 주문 N건을 가져오는 API는 웹/모바일 서비스에서 흔하다.

```sql
SELECT order_id, order_dt, amount
FROM   s_order
WHERE  cust_id = :cid
ORDER  BY order_dt DESC, order_id DESC
FETCH FIRST 50 ROWS ONLY;

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST')
);
```

- 인덱스 `ix_so_dt_desc(order_dt DESC, order_id DESC)` 가 존재하므로,
- 옵티마이저는 쿼리 정렬 방향과 인덱스 방향을 맞춰  
  **Sort 연산 없이** Top-50 을 읽을 수 있다.

실행계획에서 기대하는 모습:

- `INDEX RANGE SCAN DESCENDING IX_SO_DT_DESC`
- `FETCH FIRST 50 ROWS ONLY` 가 Stopkey로 작동 →  
  인덱스 앞부분 **몇 개 블록만 읽고 종료**.

만약 Desc 인덱스가 없다면:

- 보통 `(cust_id, order_dt)` 인덱스를 사용하면서  
  별도의 Sort 연산 또는 분산된 랜덤 I/O가 발생할 수 있다.

---

## 3. 인덱스 클러스터 테이블(Index Cluster Table)

### 3.1 개념과 동작 원리

Index Cluster는 **클러스터 키**를 기준으로

- 여러 테이블의 행을 물리적으로 **같은 블록 근처**에 모아두는 구조이다.
- 카디널리티가 적당하고, 특정 키에 대한 **단건/소수 다건 조회**,  
  **동일 키 조인**이 많으면 **랜덤 I/O를 극단적으로 줄일 수 있다.**

구성 요소:

1. **클러스터 정의**  
   - 키 컬럼(들) + SIZE(키당 예상 공간)을 지정.
2. **클러스터 인덱스**  
   - 클러스터 키 → 클러스터 블록을 가리키는 인덱스.
3. **클러스터 테이블**  
   - `CLUSTER cluster_name(key_column)` 형태로 정의된 테이블.

직관적인 그림:

```text
[Cluster Index]  (cust_id)
    |
    +--> [Cluster Block for cust_id=100]
    |        - c_customer(cust_id=100, ...)
    |        - c_order  (cust_id=100, order1...)
    |        - c_order  (cust_id=100, order2...)
    |
    +--> [Cluster Block for cust_id=101]
             - c_customer(...)
             - c_order(...)
             ...
```

---

### 3.2 실습 — 고객/주문 클러스터링

```sql
-- 1) 클러스터 정의
DROP CLUSTER c_cust PURGE;

CREATE CLUSTER c_cust (cust_id NUMBER)
SIZE 8192;    -- 키당 평균 행 길이/건수 기반 추정 필요(실업무에선 중요)

-- 2) 클러스터 인덱스
CREATE INDEX c_cust_idx ON CLUSTER c_cust;

-- 3) 클러스터 테이블
DROP TABLE c_customer PURGE;
DROP TABLE c_order    PURGE;

CREATE TABLE c_customer (
  cust_id NUMBER PRIMARY KEY,
  name    VARCHAR2(50),
  grade   VARCHAR2(10)
) CLUSTER c_cust(cust_id);

CREATE TABLE c_order (
  cust_id  NUMBER NOT NULL,
  order_id NUMBER PRIMARY KEY,
  order_dt DATE   NOT NULL,
  amount   NUMBER(12,2),
  status   VARCHAR2(8),
  CONSTRAINT fk_c_order_cust FOREIGN KEY(cust_id) REFERENCES c_customer(cust_id)
) CLUSTER c_cust(cust_id);

-- 4) 데이터 적재
BEGIN
  FOR c IN 1..20000 LOOP
    INSERT INTO c_customer VALUES (
      c,
      'name'||c,
      CASE MOD(c,5)
        WHEN 0 THEN 'VIP'
        WHEN 1 THEN 'GOLD'
        WHEN 2 THEN 'SILVER'
        WHEN 3 THEN 'BRONZE'
        ELSE 'BASIC'
      END
    );

    FOR k IN 1..4 LOOP
      INSERT INTO c_order VALUES (
        c,
        c*100 + k,
        SYSDATE - MOD(k,30),
        ROUND(DBMS_RANDOM.VALUE(10,9999),2),
        'PAID'
      );
    END LOOP;
  END LOOP;
  COMMIT;
END;
/

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'C_CUSTOMER', cascade=>TRUE);
  DBMS_STATS.GATHER_TABLE_STATS(USER,'C_ORDER',    cascade=>TRUE);
END;
/
```

---

### 3.3 동일 키 조인에서의 이점

#### 3.3.1 단일 고객 조인

```sql
VAR cid NUMBER;
EXEC :cid := 12345;

SELECT /* CLUSTER_EQ_JOIN */
       c.cust_id, c.name, o.order_id, o.amount
FROM   c_customer c
JOIN   c_order    o
ON     o.cust_id = c.cust_id
WHERE  c.cust_id = :cid;

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST')
);
```

- 클러스터 인덱스 `c_cust_idx` 로 `cust_id=:cid` 에 해당하는 클러스터 블록을 찾으면,
- 그 블록 안에
  - c_customer(row)
  - c_order(row1, row2, …)
  가 모두 몰려 있다.
- Heap+별도 인덱스 구조라면 고객/주문이 서로 다른 블록에 분산되어  
  **추가 랜덤 I/O** 가 발생했을 것이다.

#### 3.3.2 여러 고객 조인

```sql
-- VIP 고객 100명에 대한 최근 주문
WITH vip AS (
  SELECT cust_id
  FROM   c_customer
  WHERE  grade = 'VIP'
  FETCH FIRST 100 ROWS ONLY
)
SELECT v.cust_id, o.order_id, o.amount
FROM   vip v
JOIN   c_order o
ON     o.cust_id = v.cust_id
WHERE  o.order_dt >= SYSDATE - 7;
```

- VIP 고객 100명 정도의 조인이면  
  각 cust_id에 대해 거의 한두 블록에서 데이터가 정리되어 있어  
  전체 I/O가 크게 줄어든다.

---

### 3.4 Index Cluster 설계 팁

1. **클러스터 키 기준 접근이 많을 때만 사용**
   - `cust_id` 기준 단건/소수 다건 조회 + 동일 키 조인이 많을 때.
   - Full Scan/다른 조건으로 접근이 많다면 이점이 줄어든다.
2. **SIZE 추정이 중요**
   - 키당 평균 행 길이/행 수 기반으로 충분한 공간 설정.
   - 너무 작게 잡으면 **오버플로 블록** 증가 → 군집성 저하.
3. **데이터 분포 변화에 유연하지 않음**
   - 키당 행 수가 시간이 지나며 크게 증가하면  
     초기 SIZE 설계가 어긋날 수 있다.
   - 주기적으로 재클러스터링(CTAS/MOVE) 전략을 가져가는 것이 좋다.
4. **DML 비용**
   - Insert 시 같은 키의 데이터를 같은 영역에 넣어야 하므로  
     동시성·블록 경합을 잘 관리해야 한다.
   - 미국/유럽 상용 DB 튜닝 사례에서도 Index Cluster는  
     “정적/준정적 데이터 + 키 기반 조회 중심” 시나리오에서 주로 사용된다.

---

## 4. 해시 클러스터 테이블(Hash Cluster Table)

### 4.1 개념

해시 클러스터는 **해시 함수**를 사용해

- **키 → 블록** 위치를 계산하고
- 그 블록에 레코드를 저장하는 구조이다.

특징:

- 키=값 단건 조회에서 **인덱스 탐색 없이 한 번의 해시 계산으로 블록 접근**.
- `HASHKEYS` / `SIZE` 를 적절히 설정하면  
  포인트 조회 성능이 매우 빠르다.
- 반대로,
  - 범위 검색, 정렬, 부분 일치에는 적합하지 않고
  - 키 분포 변화를 잘못 예측하면 **충돌/오버플로**로 성능이 떨어질 수 있다.

---

### 4.2 실습 — 미국 웹 서비스 사용자 프로필 고속 조회

```sql
-- 1) 해시 클러스터 생성
DROP CLUSTER hc_user PURGE;

CREATE CLUSTER hc_user (user_id NUMBER)
  SIZE 8192
  HASHKEYS 400000;  -- 예상 키 개수 근사(미국 사용자 40만명 정도 가정)

-- 2) 클러스터 테이블
DROP TABLE hc_profile PURGE;

CREATE TABLE hc_profile (
  user_id NUMBER PRIMARY KEY,
  name    VARCHAR2(50),
  email   VARCHAR2(100),
  tier    VARCHAR2(10)
) CLUSTER hc_user(user_id);

-- 3) 데이터 적재
BEGIN
  FOR u IN 1..300000 LOOP
    INSERT INTO hc_profile VALUES (
      u,
      'u'||u,
      'u'||u||'@mail.com',
      CASE MOD(u,4)
        WHEN 0 THEN 'VIP'
        WHEN 1 THEN 'GOLD'
        WHEN 2 THEN 'SILVER'
        ELSE 'BASIC'
      END
    );
  END LOOP;
  COMMIT;
END;
/

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'HC_PROFILE',cascade=>TRUE);
END;
/
```

---

### 4.3 포인트 조회 성능 확인

```sql
VAR uid NUMBER;
EXEC :uid := 123456;  -- 존재하는 사용자 중 하나

SELECT /* HASH_CLUSTER_POINT */
       name, email, tier
FROM   hc_profile
WHERE  user_id = :uid;

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST')
);
```

- 실행계획에서 인덱스 탐색 없이  
  `TABLE ACCESS CLUSTER` 또는 `HASH CLUSTER` 형태의 접근을 볼 수 있다.
- I/O 입장에서 보면
  - **해시 계산으로 한 번에 블록 위치**를 알 수 있으므로
  - B-tree 인덱스 탐색(루트→브랜치→리프)을 거치는 것보다 단계 수가 줄어든다.

---

### 4.4 HASHKEYS / SIZE 설계 감각

해시 구조에서 성능을 좌우하는 요소는

1. **HASHKEYS** : 해시 버킷 개수
2. **SIZE** : 해시 버킷당 평균 공간

이다.

직관적으로,

- 해시 충돌이 없다고 가정하면  
  한 키는 **한 버킷에만 저장**되고, 포인트 조회 시 한두 블록만 읽으면 된다.
- 충돌이 많아지면
  - 같은 버킷에 여러 키가 섞여 들어가고
  - 포인트 조회 시 여러 블록을 읽거나  
    체이닝을 따라가야 할 수 있다.

해시 충돌을 줄이려면,

- `HASHKEYS` 를 **예상 키 개수 이상**으로 잡고
- `SIZE` 를 키당 평균 행 길이·개수를 반영하여  
  **버킷당 충분한 공간**을 확보해야 한다.

실무에서는

1. 테스트 환경에서 다양한 HASHKEYS/SIZE 조합으로  
   포인트 조회 성능/공간 사용량을 측정하고
2. 운영 환경에서는 주기적으로 키 개수 변화를 모니터링하여  
   필요 시 CTAS로 재구성하는 전략을 사용한다.

---

## 5. IOT + 클러스터 + 해시 클러스터 복합 튜닝 사례

이제 개별 구조를 넘어서,  
IOT + Index Cluster + Hash Cluster 를 **같은 시스템 안에서 어떻게 조합**할 수 있는지 본다.

### 5.1 시나리오 정의

가상의 미국/유럽권 SaaS 서비스(계정 기반 웹/모바일 서비스)를 가정한다.

- **Account(계정)** 테이블
  - PK 기반 단건/범위 조회가 대부분
  - 계정 생성일/최근 로그인일 기준 Top-N/리포트가 잦음
  → **IOT** 적합
- Account에 종속된 **Profile, Settings** 등 서브 테이블
  - Account 키 기준으로 자주 조인
  → **Index Cluster** 로 묶으면 랜덤 I/O 감소
- 로그인 세션/Remember-me 토큰 등
  - 토큰 문자열 → 계정 ID 매핑이 **포인트 조회** 중심
  → **Hash Cluster** 적합

---

### 5.2 계정 테이블: IOT로 구성

```sql
DROP TABLE iot_account PURGE;

CREATE TABLE iot_account (
  acct_id    NUMBER       NOT NULL,
  created_at DATE         NOT NULL,
  last_login DATE,
  status     VARCHAR2(8),
  tier       VARCHAR2(8),
  CONSTRAINT pk_iot_account PRIMARY KEY (acct_id, created_at)
) ORGANIZATION INDEX;
```

설명:

- PK `(acct_id, created_at)` 순으로 IOT 정렬.
- `created_at DESC` 기준 Top-N 조회에 유리하다.
- 필요하다면 `PCTTHRESHOLD`, `INCLUDING`, `OVERFLOW` 옵션으로  
  일부 컬럼을 OverFlow 세그먼트에 분리할 수 있다.

---

### 5.3 동일 키 서브테이블: Index Cluster로 묶기

```sql
DROP CLUSTER c_acct PURGE;

CREATE CLUSTER c_acct (acct_id NUMBER)
SIZE 8192;

CREATE INDEX c_acct_idx ON CLUSTER c_acct;

DROP TABLE c_profile  PURGE;
DROP TABLE c_settings PURGE;

CREATE TABLE c_profile (
  acct_id  NUMBER PRIMARY KEY,
  nickname VARCHAR2(50),
  bio      VARCHAR2(200)
) CLUSTER c_acct(acct_id);

CREATE TABLE c_settings (
  acct_id NUMBER PRIMARY KEY,
  opt_in  CHAR(1),
  theme   VARCHAR2(20)
) CLUSTER c_acct(acct_id);
```

설명:

- `acct_id` 기준으로 프로필/설정을 물리적으로 같은 클러스터 블록에 모은다.
- Account는 IOT, Profile/Settings는 Cluster로 두어도  
  logical 키 `acct_id` 기준으로 언제든 조인 가능.

---

### 5.4 토큰 매핑: 해시 클러스터로 빠른 포인트 조회

```sql
DROP CLUSTER hc_token PURGE;

CREATE CLUSTER hc_token (token_id VARCHAR2(64))
  SIZE 4096
  HASHKEYS 1000000;  -- 미국/유럽 사용자를 합친 토큰 수를 가정

DROP TABLE hc_token_map PURGE;

CREATE TABLE hc_token_map (
  token_id VARCHAR2(64) PRIMARY KEY,
  acct_id  NUMBER NOT NULL
) CLUSTER hc_token(token_id);
```

- 토큰 문자열을 해시 클러스터 키로 사용.
- 토큰 → 계정 매핑은 **항상 키=값 단건 조회**이므로  
  해시 클러스터 구조에 잘 맞는다.

---

### 5.5 데이터 적재 및 통계

```sql
-- IOT account 데이터
BEGIN
  FOR a IN 1..200000 LOOP
    INSERT INTO iot_account VALUES (
      a,
      DATE '2024-01-01' + MOD(a, 365),
      DATE '2024-01-01' + MOD(a, 365) + MOD(a,30),
      CASE MOD(a,5)
        WHEN 0 THEN 'ACTIVE'
        WHEN 1 THEN 'PAUSE'
        WHEN 2 THEN 'LOCK'
        WHEN 3 THEN 'LEAVE'
        ELSE 'NEW'
      END,
      CASE MOD(a,4)
        WHEN 0 THEN 'VIP'
        WHEN 1 THEN 'GOLD'
        WHEN 2 THEN 'SILVER'
        ELSE 'BASIC'
      END
    );
  END LOOP;
  COMMIT;
END;
/

-- Cluster tables
BEGIN
  FOR a IN 1..200000 LOOP
    INSERT INTO c_profile VALUES (
      a,
      'nick'||a,
      CASE WHEN MOD(a,97)=0 THEN 'gift' END
    );
    INSERT INTO c_settings VALUES (
      a,
      CASE WHEN MOD(a,2)=0 THEN 'Y' ELSE 'N' END,
      CASE MOD(a,3)
        WHEN 0 THEN 'dark'
        WHEN 1 THEN 'light'
        ELSE 'system'
      END
    );
  END LOOP;
  COMMIT;
END;
/

-- Hash cluster (token→acct_id 매핑)
BEGIN
  FOR t IN 1..500000 LOOP
    INSERT INTO hc_token_map VALUES (
      STANDARD_HASH('T'||t, 'SHA256'),
      MOD(t,200000)+1
    );
  END LOOP;
  COMMIT;
END;
/

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'IOT_ACCOUNT', cascade=>TRUE);
  DBMS_STATS.GATHER_TABLE_STATS(USER,'C_PROFILE',   cascade=>TRUE);
  DBMS_STATS.GATHER_TABLE_STATS(USER,'C_SETTINGS',  cascade=>TRUE);
  DBMS_STATS.GATHER_TABLE_STATS(USER,'HC_TOKEN_MAP',cascade=>TRUE);
END;
/
```

---

### 5.6 접근 패턴별 튜닝 포인트

#### 5.6.1 계정 타임라인/최신 가입자 — IOT의 강점

```sql
-- 최근 생성된 계정 20명
SELECT acct_id, created_at, status, tier
FROM   iot_account
ORDER  BY created_at DESC, acct_id DESC
FETCH FIRST 20 ROWS ONLY;

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST')
);
```

- IOT는 PK(=인덱스) 자체가 테이블이므로
  - PK 범위 스캔만으로 데이터를 얻는다.
  - 별도 테이블 접근이 없다.
- Top-N에서는 PK 정렬 순서와 ORDER BY 방향이 일치하면  
  **정렬 비용 + 랜덤 I/O를 동시에 최소** 할 수 있다.

#### 5.6.2 계정 상세 조회 — IOT + Index Cluster

```sql
VAR acct NUMBER;
EXEC :acct := 12345;

SELECT /* ACCOUNT_DETAIL */
       a.acct_id,
       a.status,
       a.tier,
       p.nickname,
       s.theme
FROM   iot_account a
JOIN   c_profile  p ON p.acct_id = a.acct_id
JOIN   c_settings s ON s.acct_id = a.acct_id
WHERE  a.acct_id = :acct;

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST')
);
```

동작 흐름:

1. IOT `iot_account` 에서 `acct_id=:acct` 를 PK Lookup
2. `c_acct_idx` 로 `acct_id=:acct` 에 해당하는 클러스터 블록을 찾고
3. 같은 블록에서 `c_profile`, `c_settings` 레코드를 함께 읽는다.

만약 `c_profile`, `c_settings` 가 Heap이었다면,

- 각 테이블에 대해 별도의 인덱스→테이블 랜덤 I/O가 발생했을 것이다.
- Index Cluster 덕분에 두 서브 테이블이 **같은 블록 근처**에서 해결된다.

#### 5.6.3 토큰 기반 로그인 — Hash Cluster

```sql
VAR tok VARCHAR2(64);
-- 예: 토큰 하나를 미리 조회해 저장해 둔다
SELECT token_id INTO :tok
FROM   hc_token_map
WHERE  ROWNUM = 1;

-- 로그인 요청 시 토큰→계정 ID 조회
SELECT acct_id
FROM   hc_token_map
WHERE  token_id = :tok;

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST')
);
```

- 해시 계산으로 곧바로 버킷/블록을 찾는다.
- 읽어야 하는 블록 수가 매우 적어,  
  사용자 수가 수십만~수백만 명 수준이어도 **로그인 지연 시간이 안정적**이다.

#### 5.6.4 전체 플로우 (의사 코드)

```text
1) 토큰 값으로 hc_token_map 포인트 조회 → acct_id 획득 (해시 클러스터)
2) acct_id로 iot_account PK Lookup → 계정 상태/티어/최근 로그인 확인 (IOT)
3) acct_id로 c_profile, c_settings 조회 → UI용 프로필/설정 획득 (Index Cluster)
```

각 단계에서의 I/O:

- 단계 1: 1~2블록 (해시 클러스터)
- 단계 2: IOT 인덱스 블록 1~2개
- 단계 3: 클러스터 인덱스 + 클러스터 블록 1~2개

전체적으로 “**랜덤 I/O를 구조적으로 제한**”한 설계가 된다.

---

## 6. 설계·운영 체크리스트

### 6.1 Secondary Index

- [ ] 자주 쓰는 WHERE/JOIN/ORDER BY 패턴에 맞춰 선행 컬럼을 정했는가?
- [ ] “핵심 조회 API”에 대해 커버링 인덱스를 제공하여  
      `TABLE ACCESS BY INDEX ROWID` 를 제거할 수 있는가?
- [ ] 같은 기능을 하는 인덱스가 중복되어 있지 않은가?
- [ ] INSERT/UPDATE/DELETE 성능 요구사항을 고려했을 때  
      인덱스 개수가 과하지 않은가?

### 6.2 IOT

- [ ] PK가 접근/정렬 패턴과 일치하는가?  
      (예: PK에 시간 컬럼을 포함시켜 최신순 조회 최적화)
- [ ] UPDATE로 인한 행 길이 변화가 크다면  
      PCTTHRESHOLD/OVERFLOW 구성으로 블록 분할 폭주를 방지했는가?
- [ ] 비PK 조건 질의가 많다면,  
      Secondary Index의 커버링 설계를 통해 PK 재탐색을 줄였는가?

### 6.3 Index Cluster

- [ ] 클러스터 키 기준 단건/소수 다건 조회 및 동일 키 조인이 **압도적으로 많다**는 가정이 타당한가?
- [ ] SIZE를 키당 평균 행 길이/행 수를 기준으로 적절히 산정했는가?
- [ ] 데이터 성장/분포 변화를 추적하고,  
      필요 시 재클러스터링(CTAS/MOVE) 계획을 세웠는가?

### 6.4 Hash Cluster

- [ ] 해시 클러스터에 올리는 테이블의 주요 패턴이 **키=값 포인트 조회**인가?
- [ ] HASHKEYS, SIZE를 실제 키 개수·행 길이에 맞게 주기적으로 재점검하는가?
- [ ] 범위 검색/정렬이 필요하면 별도 B-tree 인덱스/테이블로 보완하고 있는가?

### 6.5 공통

- [ ] 모든 구조 변경 전/후에 `DBMS_XPLAN` 의 Buffers/Reads/Rows를 비교하여  
      실제 개선 여부를 수치로 검증했는가?
- [ ] 운영 환경 트래픽 패턴(피크/오프피크, 읽기/쓰기 비율)에 맞게  
      IOT/Cluster/Hash Cluster 도입 시점을 조정했는가?

---

## 7. 실패·주의 사례와 대응 전략

### 7.1 IOT 남용

- 증상:
  - 비PK 조건 쿼리가 많고, Secondary Index를 통해 PK 재탐색이 남발.
  - 기대만큼 I/O 절감이 안 보임.
- 대응:
  - “PK 기반 접근이 대부분인 테이블”과  
    “그렇지 않은 테이블”을 구분해 IOT 적용 대상을 줄인다.
  - 비PK 조건이 많은 테이블은 **Heap + 적절한 Secondary Index**가 나을 수 있다.
  - IOT 테이블의 Secondary Index에 대해  
    커버링 설계를 검토해 PK 재탐색 횟수를 줄인다.

### 7.2 Index Cluster 사이징 실패

- 증상:
  - 클러스터 블록이 자주 오버플로 발생.
  - Clustering Factor가 점점 나빠짐.
- 대응:
  - 실제 데이터를 기준으로 키당 행 수/행 길이를 다시 산출하여 SIZE 재설정.
  - 데이터가 크게 늘어날 경우,  
    파티셔닝과 함께 사용하거나 클러스터를 재설계.

### 7.3 Hash Cluster 충돌 증가

- 증상:
  - 포인트 조회에서 읽는 블록 수가 증가.
  - 기댓값보다 높은 Buffers/Reads.
- 대응:
  - HASHKEYS를 늘려 충돌을 줄이고,  
    SIZE를 조정해 버킷당 공간을 넉넉히 확보.
  - 테스트 환경에서 다양한 파라미터 조합을 평가한 뒤 운영에 적용.

---

## 8. 검증용 템플릿 모음

실제 튜닝 시 자주 사용하는 SQL 템플릿을 정리해 두면 좋다.

```sql
-- 1) 최근 실행 SQL의 실제 수행 통계
SELECT *
FROM   TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(
    NULL, NULL,
    'ALLSTATS LAST +PEEKED_BINDS +PREDICATE +OUTLINE +IOSTATS +MEMSTATS'
  )
);

-- 2) 인덱스/클러스터/세그먼트 공간 확인
SELECT segment_name, segment_type, bytes/1024/1024 AS mb
FROM   user_segments
WHERE  segment_name IN (
  'S_ORDER','IOT_ACCOUNT','C_ACCT','C_ACCT_IDX',
  'HC_USER','HC_PROFILE','HC_TOKEN','HC_TOKEN_MAP'
);

-- 3) 인덱스 메타데이터
SELECT index_name, index_type, table_name, uniqueness
FROM   user_indexes
WHERE  table_name IN (
  'S_ORDER','IOT_ACCOUNT','C_PROFILE','C_SETTINGS','HC_PROFILE','HC_TOKEN_MAP'
);

-- 4) 클러스터 정보
SELECT cluster_name, key_size, type
FROM   user_clusters;

-- 5) 컬럼 통계
SELECT table_name, column_name, num_distinct, density, num_nulls, histogram
FROM   user_tab_col_statistics
WHERE  table_name IN ('S_ORDER','IOT_ACCOUNT','C_PROFILE','HC_PROFILE');
```

---

## 9. 마무리 정리

지금까지 다룬 내용을 간단히 다시 정리하면 다음과 같다.

1. **Secondary Index**
   - Heap에서는 **ROWID 바로가기**로 동작.
   - IOT에서는 논리 ROWID(PK) → PK 재탐색 한 번 더 필요.
   - 커버링 인덱스로 테이블/IOT 재탐색을 제거하면  
     랜덤 I/O를 크게 줄일 수 있다.

2. **Index Cluster**
   - 동일 키 기준으로 여러 테이블의 행을 **같은 블록 근처**에 배치.
   - 키 기준 단건/소수 다건 조회 + 동일 키 조인에서 압도적 효율.
   - SIZE/데이터 분포 변화에 민감하므로 설계·운영 시 주의가 필요하다.

3. **Hash Cluster**
   - 키=값 포인트 조회에서 **인덱스 탐색 없는 직접 블록 접근**을 제공.
   - HASHKEYS/SIZE 설계를 잘못하면 충돌/오버플로로 성능이 떨어진다.
   - 범위/정렬/부분 일치에는 적합하지 않다.

4. **IOT**
   - PK 기반 단건/범위/Top-N에 특화된 구조.
   - 테이블 BY ROWID 단계를 제거해 읽기 I/O를 줄인다.
   - 비PK 질의와 DML 패턴을 잘 분석해서 도입해야 한다.

5. **복합 튜닝**
   - IOT + Index Cluster + Hash Cluster 를  
     계정(마스터) + 서브테이블 + 토큰 매핑에 적절히 배치하면,
   - 전체 시스템의 **랜덤 I/O를 구조적으로 제한**하고  
     **응답 시간의 일관성**을 크게 높일 수 있다.

궁극적으로 중요한 것은,

> “어떤 물리 구조가 어떤 쿼리 패턴에서  
>  얼마나 I/O를 줄여주는지 숫자로 이해하고 선택하는 것”

이다.  
이 문서의 예제 스키마와 SQL을 직접 실행해 보고  
`DBMS_XPLAN` 의 Buffers/Reads/Rows 를 비교해 가며 실험해 보면,  
Secondary Index, Index Cluster, Hash Cluster, IOT 각각의 쓰임새와 한계를  
몸으로 익힐 수 있을 것이다.