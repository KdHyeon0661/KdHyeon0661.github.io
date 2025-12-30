---
layout: post
title: DB 심화 - Secondary Index, 인덱스 클러스터 테이블, 해시 클러스터 테이블, IOT+클러스터 복합 튜닝
date: 2025-11-07 20:25:23 +0900
category: DB 심화
---
# Secondary Index, 인덱스 클러스터, 해시 클러스터, IOT의 복합 튜닝 실전 가이드

## 실습 환경 설정

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';
ALTER SESSION SET statistics_level = ALL;
```

성능 분석을 위한 기본 설정입니다. `statistics_level = ALL`은 실행 계획 분석 시 실제 수행 통계를 확인하는 데 필요합니다.

## 1. Secondary Index의 핵심 이해

### 1.1 Secondary Index란 무엇인가?
Secondary Index는 기본 키(PK) 인덱스 외에 추가로 생성되는 모든 인덱스를 의미합니다. 주된 목적은 다음과 같습니다:

- **비PK 조건 최적화**: PK가 아닌 컬럼들의 검색, 정렬, 조인 성능 향상
- **커버링 인덱스**: 필요한 모든 컬럼을 인덱스에 포함시켜 테이블 접근 제거
- **정렬 및 페이지네이션**: 인덱스 자체에서 정렬 및 Top-N 처리

### 1.2 Heap 테이블 vs IOT의 Secondary Index 차이

**Heap 테이블에서의 Secondary Index:**
```sql
-- Heap 테이블 보조 인덱스 구조
(인덱스 키값, 실제 물리적 ROWID)
```
인덱스 탐색 후 ROWID를 통해 테이블 블록에 직접 접근합니다.

**IOT에서의 Secondary Index:**
```sql
-- IOT 보조 인덱스 구조  
(인덱스 키값, 논리적 ROWID = PK 값)
```
인덱스 탐색 후 PK 값을 얻고, 다시 IOT의 PK 인덱스를 탐색해야 합니다.

### 1.3 Secondary Index 설계 원칙

1. **선택도 우선**: WHERE 절에서 자주 사용되면서 선택도가 높은 컬럼을 선행
2. **정렬 패턴 반영**: 자주 사용되는 ORDER BY 패턴을 인덱스 설계에 반영
3. **커버링 전략**: 자주 조회되는 컬럼을 인덱스에 포함시켜 테이블 접근 회피
4. **인덱스 수 최소화**: 불필요한 인덱스는 DML 성능을 저하시킵니다

## 2. Secondary Index 실습: 커버링과 최적화

### 2.1 샘플 스키마 구성

```sql
-- 주문 테이블 생성
CREATE TABLE orders (
  order_id   NUMBER       NOT NULL,
  customer_id NUMBER       NOT NULL,
  order_date DATE         NOT NULL,
  status     VARCHAR2(8)  NOT NULL,
  amount     NUMBER(12,2) NOT NULL,
  notes      VARCHAR2(200),
  CONSTRAINT pk_orders PRIMARY KEY (order_id)
);

-- 데이터 적재
BEGIN
  FOR i IN 1..300000 LOOP
    INSERT INTO orders VALUES (
      i,
      MOD(i, 60000) + 1,
      DATE '2024-01-01' + MOD(i, 365),
      CASE MOD(i,5) 
        WHEN 0 THEN 'NEW' 
        WHEN 1 THEN 'PROCESSING'
        WHEN 2 THEN 'SHIPPED' 
        WHEN 3 THEN 'DELIVERED' 
        ELSE 'CANCELLED' 
      END,
      ROUND(DBMS_RANDOM.VALUE(10, 50000), 2),
      CASE WHEN MOD(i,97)=0 THEN 'Special handling required' END
    );
  END LOOP;
  COMMIT;
END;
/

-- 다양한 인덱스 생성
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date, order_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_date_desc ON orders(order_date DESC, order_id DESC);
```

### 2.2 커버링 인덱스의 성능 영향

**커버링 없는 인덱스 사용 시:**
```sql
SELECT order_id, order_date, status, amount
FROM   orders
WHERE  customer_id = :cust_id
AND    order_date >= SYSDATE - 30;
```
이 쿼리는 인덱스에서 ROWID를 얻은 후 각 행마다 테이블 블록을 랜덤 액세스합니다.

**커버링 인덱스 적용:**
```sql
-- 필요한 컬럼을 모두 포함한 커버링 인덱스
CREATE INDEX idx_orders_customer_date_cov 
ON orders(customer_id, order_date, order_id, status, amount);
```
이제 동일한 쿼리는 인덱스만 읽고 끝나므로 테이블 랜덤 액세스가 완전히 제거됩니다.

### 2.3 Descending 인덱스를 활용한 Top-N 최적화

```sql
-- 최근 주문 50건 조회
SELECT order_id, order_date, amount
FROM   orders
WHERE  customer_id = :cust_id
ORDER  BY order_date DESC, order_id DESC
FETCH FIRST 50 ROWS ONLY;
```
`idx_orders_date_desc` 인덱스가 있으면 정렬 없이 바로 Top-50 결과를 얻을 수 있습니다.

## 3. 인덱스 클러스터 테이블 심층 분석

### 3.1 인덱스 클러스터 개념
인덱스 클러스터는 동일한 클러스터 키 값을 가진 여러 테이블의 행을 물리적으로 같은 블록 근처에 저장하는 구조입니다.

### 3.2 실전 구현 예제

```sql
-- 1. 클러스터 정의
CREATE CLUSTER customer_cluster (customer_id NUMBER)
SIZE 8192;

-- 2. 클러스터 인덱스
CREATE INDEX idx_customer_cluster ON CLUSTER customer_cluster;

-- 3. 클러스터 테이블 생성
CREATE TABLE cluster_customers (
  customer_id NUMBER PRIMARY KEY,
  name        VARCHAR2(100),
  email       VARCHAR2(200),
  tier        VARCHAR2(20)
) CLUSTER customer_cluster(customer_id);

CREATE TABLE cluster_orders (
  order_id    NUMBER PRIMARY KEY,
  customer_id NUMBER NOT NULL,
  order_date  DATE   NOT NULL,
  amount      NUMBER(12,2),
  status      VARCHAR2(20),
  CONSTRAINT fk_orders_customers 
    FOREIGN KEY (customer_id) 
    REFERENCES cluster_customers(customer_id)
) CLUSTER customer_cluster(customer_id);
```

### 3.3 클러스터의 성능 이점

```sql
-- 동일 고객의 정보와 주문을 함께 조회
SELECT c.customer_id, c.name, o.order_id, o.amount
FROM   cluster_customers c
JOIN   cluster_orders    o ON o.customer_id = c.customer_id
WHERE  c.customer_id = :cust_id;
```
이 조인은 동일한 클러스터 블록 내에서 처리되므로 랜덤 I/O가 최소화됩니다.

### 3.4 클러스터 설계 시 고려사항

1. **키 선택**: 클러스터 키는 조인이 빈번하게 발생하는 컬럼이어야 합니다
2. **크기 산정**: SIZE 파라미터는 키당 평균 행 수와 행 크기를 기반으로 설정
3. **데이터 분포**: 키 값의 분포가 균등하지 않으면 핫스팟 문제 발생 가능
4. **유지 관리**: 데이터 증가에 따른 주기적인 재구성 필요

## 4. 해시 클러스터 테이블 활용

### 4.1 해시 클러스터 개념
해시 클러스터는 해시 함수를 사용하여 키 값을 직접 블록 위치로 변환하는 구조입니다.

### 4.2 구현 예제

```sql
-- 해시 클러스터 생성
CREATE CLUSTER user_hash_cluster (user_id NUMBER)
  SIZE 1024
  HASH IS user_id
  HASHKEYS 100000;

-- 해시 클러스터 테이블
CREATE TABLE hash_user_profiles (
  user_id    NUMBER PRIMARY KEY,
  username   VARCHAR2(50),
  email      VARCHAR2(100),
  created_at DATE DEFAULT SYSDATE
) CLUSTER user_hash_cluster(user_id);
```

### 4.3 해시 클러스터 성능 특성

```sql
-- 포인트 조회 (매우 빠름)
SELECT username, email
FROM   hash_user_profiles
WHERE  user_id = :user_id;
```

**장점:**
- 키=값 조회에서 인덱스 탐색 없이 직접 블록 접근
- 예측 가능한 성능 (일정한 해시 계산 시간)

**단점:**
- 범위 조회, 정렬, 부분 일치 검색에 부적합
- 해시 충돌 시 성능 저하
- HASHKEYS와 SIZE 파라미터의 정확한 설정 필요

### 4.4 파라미터 튜닝 가이드

1. **HASHKEYS**: 예상되는 고유 키 수보다 약간 크게 설정
2. **SIZE**: 키당 평균 행 크기와 예상 행 수를 고려
3. **모니터링**: 주기적인 충돌률과 공간 사용률 확인

## 5. IOT(Index-Organized Table) 심화

### 5.1 IOT 기본 구조

```sql
-- 기본 IOT 생성
CREATE TABLE iot_accounts (
  account_id   NUMBER       NOT NULL,
  created_date DATE         NOT NULL,
  last_login   DATE,
  status       VARCHAR2(8),
  tier         VARCHAR2(8),
  CONSTRAINT pk_iot_accounts 
    PRIMARY KEY (account_id, created_date)
) ORGANIZATION INDEX;
```

### 5.2 IOT의 장점과 적용 시나리오

**적합한 시나리오:**
1. PK 기반의 단건 또는 범위 조회가 주류인 경우
2. 정렬된 데이터 접근 패턴이 필요한 경우
3. 공간 효율성이 중요한 경우

**비적합한 시나리오:**
1. 비PK 컬럼 기반의 조회가 빈번한 경우
2. 자주 변경되는 대용량 행을 처리하는 경우
3. 전체 테이블 스캔이 빈번한 경우

### 5.3 IOT에 대한 오해와 진실

**오해:** IOT가 모든 경우에 힙 테이블보다 빠르다
**진실:** IOT는 특정 액세스 패턴에서만 우수한 성능을 발휘합니다

**오해:** IOT의 보조 인덱스는 항상 느리다
**진실:** 커버링 인덱스로 설계하면 보조 인덱스도 우수한 성능을 낼 수 있습니다

## 6. 복합 구조 설계: SaaS 시스템 사례

### 6.1 시스템 요구사항 분석
가상의 SaaS 시스템에서 다음과 같은 요구사항이 있다고 가정합니다:

1. 계정 정보는 PK 기반 조회가 대부분
2. 계정과 프로필/설정은 자주 함께 조회
3. 세션 토큰은 매우 빠른 조회가 필요
4. 시간 기반의 리포트 생성 필요

### 6.2 최적화된 구조 설계

```sql
-- 1. 계정 정보: IOT (PK 기반 조회 최적화)
CREATE TABLE accounts (
  account_id   NUMBER       NOT NULL,
  created_date DATE         NOT NULL,
  last_login   DATE,
  status       VARCHAR2(8),
  tier         VARCHAR2(8),
  CONSTRAINT pk_accounts 
    PRIMARY KEY (account_id, created_date)
) ORGANIZATION INDEX;

-- 2. 프로필/설정: 인덱스 클러스터 (함께 조회 최적화)
CREATE CLUSTER account_cluster (account_id NUMBER)
SIZE 8192;

CREATE INDEX idx_account_cluster ON CLUSTER account_cluster;

CREATE TABLE profiles (
  account_id NUMBER PRIMARY KEY,
  nickname   VARCHAR2(50),
  bio        VARCHAR2(500)
) CLUSTER account_cluster(account_id);

CREATE TABLE settings (
  account_id NUMBER PRIMARY KEY,
  theme      VARCHAR2(20),
  language   VARCHAR2(10)
) CLUSTER account_cluster(account_id);

-- 3. 세션 토큰: 해시 클러스터 (초고속 조회)
CREATE CLUSTER token_hash_cluster (token VARCHAR2(64))
  SIZE 512
  HASHKEYS 500000;

CREATE TABLE session_tokens (
  token      VARCHAR2(64) PRIMARY KEY,
  account_id NUMBER NOT NULL,
  expires_at DATE   NOT NULL
) CLUSTER token_hash_cluster(token);
```

### 6.3 복합 구조의 성능 이점

**사용자 로그인 플로우:**
```sql
-- 1. 토큰 검증 (해시 클러스터)
SELECT account_id 
FROM   session_tokens 
WHERE  token = :token 
AND    expires_at > SYSDATE;

-- 2. 계정 정보 조회 (IOT)
SELECT status, tier 
FROM   accounts 
WHERE  account_id = :account_id;

-- 3. 프로필/설정 조회 (인덱스 클러스터)
SELECT p.nickname, s.theme
FROM   profiles p
JOIN   settings s ON s.account_id = p.account_id
WHERE  p.account_id = :account_id;
```

이 구조는 각 단계에서 최적의 I/O 패턴을 제공합니다.

## 7. 성능 검증 방법론

### 7.1 실행 계획 분석

```sql
-- 상세한 실행 계획 확인
SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(
    NULL, NULL,
    'ALLSTATS LAST +PEEKED_BINDS +PREDICATE +OUTLINE'
  )
);
```

### 7.2 주요 성능 메트릭

1. **버퍼 읽기 수 (Buffer Gets)**: 논리적 I/O 양
2. **디스크 읽기 수 (Disk Reads)**: 물리적 I/O 양  
3. **실행 시간 (Elapsed Time)**: 전체 소요 시간
4. **행 처리 수 (Rows Processed)**: 처리된 데이터 양

### 7.3 AWR 리포트 분석

```sql
-- Top SQL 식별
SELECT sql_id, executions, buffer_gets, disk_reads,
       ROUND(buffer_gets/NULLIF(executions,0)) avg_lio,
       ROUND(disk_reads/NULLIF(executions,0))  avg_pio
FROM   v$sql
WHERE  parsing_schema_name = USER
ORDER  BY buffer_gets DESC
FETCH FIRST 10 ROWS ONLY;
```

## 8. 일반적인 실수와 해결책

### 8.1 과도한 인덱스 생성

**문제:** 너무 많은 인덱스로 인한 DML 성능 저하
**해결:** 실제로 사용되는 인덱스만 유지, 인덱스 사용 모니터링

### 8.2 잘못된 클러스터 크기

**문제:** SIZE 파라미터 부적절로 인한 오버플로우 발생
**해결:** 실제 데이터 패턴 분석 후 SIZE 재조정

### 8.3 IOT의 오용

**문제:** 비PK 조회가 많은 테이블에 IOT 적용
**해결:** 액세스 패턴 분석 후 힙 테이블로 전환 또는 보조 인덱스 최적화

### 8.4 해시 충돜 무시

**문제:** HASHKEYS 부족으로 인한 충돜 증가
**해결:** 주기적인 충돜률 모니터링과 HASHKEYS 조정

## 9. 운영 모니터링 체크리스트

### 인덱스 모니터링
```sql
-- 사용되지 않는 인덱스 확인
SELECT index_name, table_name
FROM   user_indexes
WHERE  index_name NOT IN (
  SELECT object_name
  FROM   v$sql_plan
  WHERE  object_owner = USER
  AND    operation LIKE '%INDEX%'
);
```

### 클러스터 상태 모니터링
```sql
-- 클러스터 공간 사용률
SELECT segment_name, bytes/1024/1024 as mb, blocks
FROM   user_segments
WHERE  segment_type LIKE '%CLUSTER%';
```

### IOT 상태 모니터링
```sql
-- IOT 행 이동 현황
SELECT table_name, num_rows, chain_cnt,
       ROUND(chain_cnt/NULLIF(num_rows,0)*100,2) as chain_pct
FROM   user_tables
WHERE  iot_type IS NOT NULL;
```

## 결론

Secondary Index, 인덱스 클러스터, 해시 클러스터, IOT는 각각 고유한 장점과 적용 시나리오를 가진 강력한 데이터베이스 구조 최적화 도구입니다. 효과적인 활용을 위한 핵심 원칙은 다음과 같습니다:

### 설계 원칙 요약

1. **데이터 액세스 패턴 중심 설계**: 실제 비즈니스에서 어떻게 데이터에 접근하는지 분석
2. **점진적 최적화**: 한 번에 모든 것을 변경하지 말고, 측정 가능한 단계별 개선
3. **트레이드오프 이해**: 읽기 성능과 쓰기 성능, 공간 효율성과 성능 간의 균형 유지
4. **모니터링 기반 운영**: 지속적인 성능 모니터링과 필요시 조정

### 구조 선택 가이드라인

| 데이터 특성 | 권장 구조 | 주요 이유 |
|------------|-----------|-----------|
| PK 기반 조회 위주 | IOT | 테이블 랜덤 액세스 제거 |
| 동일 키 다중 테이블 조인 | 인덱스 클러스터 | 관련 데이터 물리적 군집화 |
| 정확한 키=값 초고속 조회 | 해시 클러스터 | 인덱스 탐색 회피 |
| 다양한 액세스 패턴 | 힙 테이블 + 적절한 인덱스 | 유연성과 범용성 |

### 실전 적용 권장사항

1. **증명 기반 접근**: 모든 최적화는 실제 성능 데이터를 통해 검증
2. **비즈니스 연계**: 기술적 최적화가 비즈니스 가치에 어떻게 기여하는지 명확히 이해
3. **유연성 유지**: 비즈니스 요구사항 변화에 대응할 수 있는 구조 설계
4. **문서화**: 설계 결정의 이유와 예상 효과를 명확히 문서화

이러한 구조들을 올바르게 이해하고 적용하면, 단순한 쿼리 튜닝을 넘어 시스템 전체의 성능과 확장성을 근본적으로 개선할 수 있습니다. 가장 중요한 것은 특정 기술의 맹목적 적용이 아니라, 실제 비즈니스 요구사항과 데이터 액세스 패턴에 부합하는 최적의 구조를 선택하는 것입니다.