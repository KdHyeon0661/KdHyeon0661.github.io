---
layout: post
title: DB 심화 - Secondary Index, 인덱스 클러스터 테이블, 해시 클러스터 테이블, IOT+클러스터 복합 튜닝
date: 2025-11-07 20:25:23 +0900
category: DB 심화
---
# Secondary Index, 인덱스 클러스터 테이블, 해시 클러스터 테이블, IOT+클러스터 복합 튜닝 사례

## 0. 공통 준비

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';
ALTER SESSION SET statistics_level = ALL; -- ALLSTATS LAST 보기
```

---

# 1. Secondary Index(보조 인덱스) — Heap vs IOT의 의미 차이

## 1.1 Secondary Index란?
- **테이블을 식별하는 기본 키(Primary Key) 인덱스 외**에 추가로 만드는 모든 인덱스.
- 목적: **비PK 조건**(검색/정렬/조인)을 빠르게 처리하거나 **커버링**(테이블 미방문) 달성.

## 1.2 Heap과 IOT에서의 차이
- **Heap 테이블**의 보조 인덱스: **물리 ROWID**를 가진다 → 인덱스에서 ROWID로 **테이블 블록을 바로** 튕김.
- **IOT(ORGANIZATION INDEX)**의 보조 인덱스: **논리 ROWID(=IOT PK)**를 가진다 → 보조 인덱스 → **PK 재탐색**이 한 번 더 필요.
  - IOT에서 비PK 조건이 많다면 **보조 인덱스 커버링**(SELECT-LIST 포함)으로 **PK 재탐색 최소화**를 검토.

## 1.3 Secondary Index 설계 체크포인트
- **선행 컬럼 선택도**(카디널리티)와 **업무 조건 패턴**을 일치.
- **정렬/Top-N**을 인덱스 순서/방향으로 흡수(ASC/DESC).
- **커버링**을 위해 SELECT-LIST를 인덱스에 포함(Oracle에는 INCLUDE가 없어 **키 뒤에 추가**).
- **과다한 보조 인덱스**는 DML 비용/락 경합/공간 비용을 키움 → **핵심 질의** 기준 최소화.

---

## 2. Secondary Index 실습

### 2.1 샘플 스키마
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
      MOD(i, 60000) + 1,
      DATE '2024-01-01' + MOD(i, 365),
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
CREATE INDEX ix_so_cdt ON s_order(cust_id, order_dt, order_id);
CREATE INDEX ix_so_status ON s_order(status);
CREATE INDEX ix_so_dt_desc ON s_order(order_dt DESC, order_id DESC);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'S_ORDER', cascade=>TRUE,
    method_opt=>'for all columns size skewonly');
END;
/
```

### 2.2 커버링 없는 보조 인덱스 → 테이블 랜덤 액세스 발생
```sql
-- 최근 30일, 특정 고객의 주문 목록(금액/상태 필요)
SELECT order_id, order_dt, status, amount
FROM   s_order
WHERE  cust_id = :cid
AND    order_dt >= SYSDATE - 30
ORDER  BY order_dt, order_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```
- 계획에 `TABLE ACCESS BY INDEX ROWID S_ORDER`가 보이면 **랜덤 I/O**가 발생(인덱스에는 status/amount가 없어 테이블 방문).

### 2.3 커버링 보조 인덱스 — 테이블 미방문화
```sql
-- 커버링 인덱스: SELECT-LIST 포함
CREATE INDEX ix_so_cdt_cov
ON s_order(cust_id, order_dt, order_id, status, amount);

BEGIN DBMS_STATS.GATHER_INDEX_STATS(USER,'IX_SO_CDT_COV'); END;
/

-- 동일 질의 재실행 → TABLE BY ROWID 단계가 사라지거나 크게 감소
SELECT order_id, order_dt, status, amount
FROM   s_order
WHERE  cust_id = :cid
AND    order_dt >= SYSDATE - 30
ORDER  BY order_dt, order_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```
- Buffers/Reads가 대폭 줄어드는지 확인.

### 2.4 최신순 Top-N을 인덱스에서 흡수(Descending)
```sql
-- 최신 50건(정렬 제거 + Stopkey)
SELECT order_id, order_dt, amount
FROM   s_order
WHERE  cust_id = :cid
ORDER  BY order_dt DESC, order_id DESC
FETCH FIRST 50 ROWS ONLY;

-- ix_so_dt_desc가 있을 때 Sort 제거 + 앞 블록만 읽고 종료
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```

---

# 3. 인덱스 클러스터 테이블(Index Cluster Table)

## 3.1 개념
- **클러스터 키**를 기준으로 **여러 테이블의 행을 물리적으로 같은 블록 근처**에 배치.
- **동일 키 조인**(예: 고객+주문, 게시글+댓글)이나 **키 단건/소수 다건 조회**에서 **랜덤 I/O 극소화**.

## 3.2 실습 — 고객/주문을 클러스터링
```sql
-- 1) 클러스터 정의
DROP CLUSTER c_cust PURGE;
CREATE CLUSTER c_cust (cust_id NUMBER)
SIZE 8192    -- 키당 평균 행 길이/건수 기반 추정 필요(실업무에선 중요)
TABLESPACE users;

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

-- 4) 데이터
BEGIN
  FOR c IN 1..20000 LOOP
    INSERT INTO c_customer VALUES (c, 'name'||c,
      CASE MOD(c,5) WHEN 0 THEN 'VIP' WHEN 1 THEN 'GOLD' WHEN 2 THEN 'SILVER' WHEN 3 THEN 'BRONZE' ELSE 'BASIC' END);
    FOR k IN 1..4 LOOP
      INSERT INTO c_order VALUES (c, c*100+k, SYSDATE - MOD(k,30), ROUND(DBMS_RANDOM.VALUE(10,9999),2), 'PAID');
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

### 3.3 동일 키 조인 비교
```sql
-- 동일 cust_id 조인(랜덤 I/O 감소 기대)
SELECT /* CLUSTER EQ JOIN */
       c.cust_id, c.name, o.order_id, o.amount
FROM   c_customer c
JOIN   c_order    o
ON     o.cust_id = c.cust_id
WHERE  c.cust_id = :cid;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```
- Heap+별도 인덱스 구조 대비 **Buffers/Reads** 감소가 관찰되는지 점검.
- 이유: **같은 cust_id**의 행이 **같은 블록 근처**에 물리적으로 저장되어 **블록 재사용률↑**.

## 3.4 설계 팁
- **SIZE**는 키당 평균 행 길이/건수로 넉넉히 설정(오버플로 방지).
- 데이터 분포가 바뀌면 재클러스터링(CTAS/MOVE) 고려.
- **키 기준 접근**이 아닐 때는 이점이 크지 않을 수 있음.

---

# 4. 해시 클러스터 테이블(Hash Cluster Table)

## 4.1 개념
- **해시 함수**로 **키 → 블록** 위치를 **직접 계산**하여 접속(인덱스 불필요).
- **키=값 단건 조회**가 매우 빠름.
- 범위/정렬/부분 일치에는 부적합. **HASHKEYS/SIZE**를 **정확히 추정**해야 충돌/오버플로가 줄어듦.

## 4.2 실습 — 사용자 프로필 고속 조회
```sql
-- 1) 해시 클러스터 생성
DROP CLUSTER hc_user PURGE;
CREATE CLUSTER hc_user (user_id NUMBER)
  SIZE 8192
  HASHKEYS 400000;  -- 예상 키 개수 근사

-- 2) 클러스터 테이블
DROP TABLE hc_profile PURGE;
CREATE TABLE hc_profile (
  user_id NUMBER PRIMARY KEY,
  name    VARCHAR2(50),
  email   VARCHAR2(100),
  tier    VARCHAR2(10)
) CLUSTER hc_user(user_id);

-- 3) 데이터
BEGIN
  FOR u IN 1..300000 LOOP
    INSERT INTO hc_profile VALUES (u, 'u'||u, 'u'||u||'@mail.com', 'BASIC');
  END LOOP;
  COMMIT;
END;
/

BEGIN DBMS_STATS.GATHER_TABLE_STATS(USER,'HC_PROFILE',cascade=>TRUE); END;
/
```

### 4.3 포인트 조회 성능 확인
```sql
SELECT /* HASH CLUSTER POINT LOOKUP */
       name, email, tier
FROM   hc_profile
WHERE  user_id = :uid;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```
- 해시 계산으로 **바로 블록**에 접근 → 인덱스 탐색 생략.
- 충돌/오버플로가 많아지면 성능 저하 → `HASHKEYS/SIZE` 재설정/재구성 필요.

---

# 5. IOT + 클러스터 **복합 튜닝 사례**

> 시나리오:
> - **Account(계정) 테이블**: **PK 기반 단건/범위 조회**가 대부분(최근 로그인/정렬/Top-N). → **IOT**가 적합.
> - **Account에 종속된 여러 세부 테이블**(예: Profile, Settings, Small Logs)을 **동일 키로 자주 조인**. → **인덱스 클러스터**로 **동일 키 묶기**.
> - 추가로 **특정 Lookup**(e.g., `token_id`)은 **키=값** 단건 조회를 극대화 → **해시 클러스터**.

## 5.1 스키마 구성

### A) 계정(마스터): IOT
```sql
DROP TABLE iot_account PURGE;

CREATE TABLE iot_account (
  acct_id    NUMBER       NOT NULL,
  created_at DATE         NOT NULL,
  last_login DATE,
  status     VARCHAR2(8),
  tier       VARCHAR2(8),
  CONSTRAINT pk_iot_account PRIMARY KEY (acct_id, created_at)  -- PK로 정렬
) ORGANIZATION INDEX;

-- 사용 프로파일/변경 가능성이 큰 컬럼은 OVERFLOW로 분리하고 싶다면:
-- PCTTHRESHOLD/INCLUDING/OVERFLOW 옵션을 상황에 맞게 설정
```

### B) 동일 키 종속 테이블: 인덱스 클러스터
```sql
DROP CLUSTER c_acct PURGE;
CREATE CLUSTER c_acct (acct_id NUMBER)
SIZE 8192;

CREATE INDEX c_acct_idx ON CLUSTER c_acct;

DROP TABLE c_profile PURGE;
DROP TABLE c_settings PURGE;

CREATE TABLE c_profile (
  acct_id NUMBER PRIMARY KEY,
  nickname VARCHAR2(50),
  bio      VARCHAR2(200)
) CLUSTER c_acct(acct_id);

CREATE TABLE c_settings (
  acct_id NUMBER PRIMARY KEY,
  opt_in  CHAR(1),
  theme   VARCHAR2(20)
) CLUSTER c_acct(acct_id);
```

### C) 키=값 포인트 조회 전용: 해시 클러스터
```sql
DROP CLUSTER hc_token PURGE;
CREATE CLUSTER hc_token (token_id VARCHAR2(64))
  SIZE 4096
  HASHKEYS 1000000;

DROP TABLE hc_token_map PURGE;
CREATE TABLE hc_token_map (
  token_id VARCHAR2(64) PRIMARY KEY,
  acct_id  NUMBER NOT NULL
) CLUSTER hc_token(token_id);
```

## 5.2 데이터 적재 & 통계
```sql
-- IOT account
BEGIN
  FOR a IN 1..200000 LOOP
    INSERT INTO iot_account VALUES (
      a,
      DATE '2024-01-01' + MOD(a, 365),
      DATE '2024-01-01' + MOD(a, 365) + MOD(a,30),
      CASE MOD(a,5) WHEN 0 THEN 'ACTIVE' WHEN 1 THEN 'PAUSE'
                    WHEN 2 THEN 'LOCK' WHEN 3 THEN 'LEAVE' ELSE 'NEW' END,
      CASE MOD(a,4) WHEN 0 THEN 'VIP' WHEN 1 THEN 'GOLD' WHEN 2 THEN 'SILVER' ELSE 'BASIC' END
    );
  END LOOP;
  COMMIT;
END;
/

-- Cluster tables
BEGIN
  FOR a IN 1..200000 LOOP
    INSERT INTO c_profile  VALUES (a, 'nick'||a, CASE WHEN MOD(a,97)=0 THEN 'gift' END);
    INSERT INTO c_settings VALUES (a, CASE WHEN MOD(a,2)=0 THEN 'Y' ELSE 'N' END,
                                      CASE MOD(a,3) WHEN 0 THEN 'dark' WHEN 1 THEN 'light' ELSE 'system' END);
  END LOOP;
  COMMIT;
END;
/

-- Hash cluster
BEGIN
  FOR t IN 1..500000 LOOP
    INSERT INTO hc_token_map VALUES (STANDARD_HASH('T'||t, 'SHA256'), MOD(t,200000)+1);
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

## 5.3 접근 패턴별 성능 포인트

### (1) 계정 타임라인/최신순 Top-N: **IOT로 PK 범위 + Stopkey**
```sql
-- 계정 생성일 기준 최신 20건 조회(정렬 제거 + Stopkey)
SELECT acct_id, created_at, status, tier
FROM   iot_account
ORDER  BY created_at DESC, acct_id DESC
FETCH FIRST 20 ROWS ONLY;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```
- **TABLE BY ROWID 단계가 없거나 최소** → 랜덤 I/O 급감.
- IOT는 **PK가 곧 테이블**이므로 최신/Top-N에 특히 강함.

### (2) 동일 키 조인(프로필/세팅): **인덱스 클러스터로 랜덤 I/O 최소화**
```sql
-- 단일 계정 상세 조회(프로필/세팅 조인)
SELECT /* CLUSTER EQ JOIN */
       a.acct_id, a.status, a.tier, p.nickname, s.theme
FROM   iot_account a
JOIN   c_profile  p ON p.acct_id = a.acct_id
JOIN   c_settings s ON s.acct_id = a.acct_id
WHERE  a.acct_id = :acct;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```
- `p`와 `s`가 **같은 클러스터 블록 근처**에 있어 **블록 재사용률↑**.

### (3) 토큰→계정 매핑: **해시 클러스터 포인트 조회**
```sql
-- 토큰으로 계정 찾기(키=값 단건 조회 최적)
SELECT acct_id
FROM   hc_token_map
WHERE  token_id = :token_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```
- 인덱스 탐색 없이 **해시로 곧바로 블록 접근**.
- 토큰 기반 인증/세션 매핑 등에서 초저지연 목표 달성.

### (4) 앱 API 종단 간 — 세 경로 결합
```sql
-- 1) token→acct_id (해시 클러스터)
-- 2) acct 기본(최근 상태) (IOT)
-- 3) profile/settings (인덱스 클러스터)

-- Pseudocode
-- acct := SELECT acct_id FROM hc_token_map WHERE token_id=:token;
-- head := SELECT status, tier, created_at FROM iot_account WHERE acct_id=:acct ORDER BY created_at DESC FETCH FIRST 1 ROW;
-- prof := SELECT nickname FROM c_profile WHERE acct_id=:acct;
-- setg := SELECT theme FROM c_settings WHERE acct_id=:acct;
```
- **포인트 조회 + 최신 Top-N + 동일 키 조인**이 각 물리구조에서 **최소 I/O**로 처리.

## 5.4 튜닝 체크리스트
- **IOT**
  - PK가 접근/정렬 패턴과 일치하는가?
  - UPDATE로 행 길이 변동이 많다면 **OVERFLOW(PCTTHRESHOLD/INCLUDING)** 적용 검토.
  - 보조 인덱스는 최소화(논리 ROWID 재탐색 비용). 필요한 경우 **커버링**으로 PK 재탐색 최소화.

- **인덱스 클러스터**
  - **SIZE**(클러스터 블록 공간) 산정이 정확한가(키당 평균 행수/길이 기준)?
  - 조인/조회가 **클러스터 키 기준**으로 일어난다는 가정이 유지되는가?

- **해시 클러스터**
  - **HASHKEYS/SIZE**를 정확히 추정했는가? 충돌/오버플로가 늘면 성능 하락.
  - 범위/정렬 질의는 외부 인덱스/별도 구조로 보완.

- **전반**
  - `ALLSTATS LAST`로 Buffers/Reads/Starts/Rows를 **숫자**로 검증.
  - 데이터 분포 변화 시 **재클러스터링/CTAS** 주기 정의.
  - DML 동시성/공간/재구성 비용과의 **트레이드오프**를 수용 가능한지 확인.

---

# 6. 실패/주의 사례와 대응

- **IOT를 남용**: 비PK 질의가 다수 → 보조 인덱스 경유 + PK 재탐색이 쌓여 기대만큼 이득이 없음.
  → **핫 경로**만 IOT, 나머지는 Heap 유지/혼용.

- **인덱스 클러스터 사이징 미흡**: SIZE 작아 오버플로 → 블록 분산 + I/O 증가.
  → 키당 평균 행수/행 길이 기반 재설계, 데이터 증가율 반영.

- **해시 클러스터 과소/과대 HASHKEYS**: 충돌↑ 혹은 공간 낭비/캐시 비효율.
  → 모니터링 기반 재구성(저피크 시간대), 가능하면 **테스트 환경에서 리허설**.

---

# 7. 검증 템플릿 모음

```sql
-- 최근 실행 SQL 실제 수행 통계
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PEEKED_BINDS +OUTLINE'));

-- 세그먼트/공간 파악
SELECT segment_name, segment_type, bytes/1024/1024 MB
FROM   user_segments
WHERE  segment_name IN ('IOT_ACCOUNT','C_ACCT','C_ACCT_IDX','HC_TOKEN','HC_TOKEN_MAP');

-- 인덱스/클러스터/파티션 구조 확인
SELECT index_name, index_type, table_name, uniqueness
FROM   user_indexes
WHERE  table_name IN ('IOT_ACCOUNT','C_PROFILE','C_SETTINGS');

-- 클러스터 구성 정보
SELECT cluster_name, key_size, type
FROM   user_clusters;

-- 해시 클러스터 파라미터 확인(딕셔너리 뷰/DDL로 재확인 권장)
```

---

## 8. 핵심 요약

- **Secondary Index**는 **조건/정렬/커버링**을 위해 필수이되, DML·공간 비용을 감안해 **핵심 질의 중심 최소화**.
- **인덱스 클러스터**는 **동일 키 조인/같은 키의 소수 다건 조회**에서 **랜덤 I/O**를 구조적으로 줄인다.
- **해시 클러스터**는 **키=값 포인트 조회**에서 **초저지연**을 가능케 한다.
- **IOT**는 **PK 기반 단건/범위/Top-N**에서 **테이블 BY ROWID**를 제거하여 **최소 I/O**를 달성한다.
- **복합 튜닝**: IOT(마스터) + 인덱스 클러스터(동일 키 서브테이블) + 해시 클러스터(포인트 매핑)를 결합하면,
  **종단 간(End-to-End)**으로 **랜덤 I/O를 최소화**하고 **예측 가능한 응답시간**을 얻을 수 있다.

> 최종 판단은 **실측치**가 합니다. `DBMS_XPLAN`의 **Buffers/Reads/Rows**가 좋아졌는지 확인하고,
> 데이터 분포/트래픽 패턴의 변화를 주기적으로 점검하여 **구조적 선택**을 보정하세요. 숫자가 답입니다.
