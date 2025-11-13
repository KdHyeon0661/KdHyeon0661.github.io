---
layout: post
title: DB - DDL, DML, DCL, TCL
date: 2025-02-15 20:20:23 +0900
category: DB
---
# SQL 명령어 분류: DDL, DML, DCL

## 0. 용어 관점 정리(빠르게 복습)
- **DDL**: 오브젝트(스키마) 정의/변경/삭제. 대부분 DB에서 **암묵적 커밋**(자동 커밋) 동반.
- **DML**: 데이터 조작(SELECT/INSERT/UPDATE/DELETE…); **트랜잭션 제어 대상**.
- **DCL**: 권한 부여/회수(GRANT/REVOKE…); 사용자·롤 기반 접근 제어.
- **TCL**: COMMIT/ROLLBACK/SAVEPOINT…; DML 수행 결과를 확정/철회.

> 안전한 실무 플로우:
> **DDL로 구조 준비 → DCL로 권한 설계 → DML로 데이터 작업 → TCL로 변경 확정**.

---

## 1. DDL(데이터 정의) — 구조 설계·변경의 성능/가용성 이슈까지

### 1.1 핵심 명령과 공통 개념
```sql
-- 공통적 DDL
CREATE TABLE ...;
ALTER TABLE ... ADD COLUMN ...;
ALTER TABLE ... MODIFY/ALTER COLUMN ...;   -- DBMS별 문법 차이
DROP TABLE ...;
TRUNCATE TABLE ...; -- 데이터만 비움(대부분 롤백 불가), 구조 유지
CREATE INDEX ...;   -- B-tree/Hash/GIN/GiST/Bitmap 등 엔진별 다양
DROP INDEX ...;
RENAME TABLE ...;   -- 또는 ALTER ... RENAME TO ...
```

#### 실무 포인트
- **DDL은 암묵적 커밋**: 동일 트랜잭션 내 다른 DML과 **원자적으로 롤백**되지 않을 수 있다(DBMS 차이 큼).
- **온라인 DDL 지원 범위**: 무중단 변경 가능 여부/락 레벨/백그라운드 빌드 비용이 DB마다 다르다.
- **TRUNCATE**: 빠르지만 **회복 불가**(Oracle은 `FLASHBACK` 옵션/리사이클빈 등 예외적 경로 존재), 트리거 미발화가 보통.

### 1.2 DBMS별 Online DDL·락 특성(요약)
| 구분 | Online DDL | 컬럼 추가/변경 | 인덱스 생성/삭제 | 비고 |
|---|---|---|---|---|
| PostgreSQL | 일부(11+부터 인덱스 `CONCURRENTLY`) | ADD COLUMN 기본 O(메타데이터만), 타입 변경은 재작성 가능 | `CREATE INDEX CONCURRENTLY`로 오래 락 피함 | 장시간 `CONCURRENTLY` 실패 시 재시도 필요 |
| MySQL(InnoDB) | `ALGORITHM=INPLACE/INSTANT` 옵션 | 8.0 `INSTANT`로 메타데이터만 추가(일부 유형) | `ONLINE` 유사 기능, 버전에 따라 제약 | 대형 테이블은 백필/리빌드 주의 |
| Oracle | 대부분 온라인 옵션 풍부 | `ONLINE`/`INVISIBLE COLUMN` 등 | `ONLINE` 인덱스 생성 지원 | 파티션/서브파티션 DDL 풍부 |
| SQL Server | `WITH (ONLINE = ON)` | 계산 열/스파스 열 등 | `ONLINE` 인덱스 리빌드 | 에디션/버전에 따라 제약 |

> 운영에서는 **DDL 윈도우**(배포 창)·**세이프티 플래그**(세션 타임아웃/쿼터)·**백업 및 롤백 플랜**을 함께 설계.

### 1.3 확장 DDL 실무 패턴

#### (A) 제약조건(무결성) 설계
```sql
-- PostgreSQL 예
CREATE TABLE employee (
  emp_id       BIGSERIAL PRIMARY KEY,
  email        CITEXT UNIQUE,              -- 대소문자 무시 유니크(확장)
  dept_id      BIGINT NOT NULL,
  salary       NUMERIC(12,2) CHECK (salary >= 0),
  hired_at     TIMESTAMPTZ DEFAULT now(),
  CONSTRAINT fk_emp_dept FOREIGN KEY (dept_id) REFERENCES dept(id) ON DELETE RESTRICT
);
```
- **CHECK**로 도메인 제약을 DB 레벨에서 보장(앱 로직과 중복하되 DB가 최종 방어선).
- **FK ON DELETE/UPDATE**: CASCADE/SET NULL/RESTRICT의 의미를 비즈니스 규칙과 일치시킬 것.

#### (B) 파티셔닝/파티션 프루닝
```sql
-- Postgres 범위 파티셔닝
CREATE TABLE logs (
  id BIGSERIAL PRIMARY KEY,
  ts TIMESTAMPTZ NOT NULL,
  payload JSONB
) PARTITION BY RANGE (ts);

CREATE TABLE logs_2024_06 PARTITION OF logs
  FOR VALUES FROM ('2024-06-01') TO ('2024-07-01');

-- 쿼리 시 프루닝 자동
SELECT count(*) FROM logs WHERE ts >= '2024-06-01' AND ts < '2024-07-01';
```
- 대용량 테이블에서 **프루닝**은 핵심 성능 요소. **로테이션/아카이브 DDL**을 자동화한다.

#### (C) 가상/생성(Generated) 컬럼, 가시성 제어
```sql
-- MySQL 8.0: 생성 컬럼 + 인덱스
ALTER TABLE orders
  ADD COLUMN order_month DATE GENERATED ALWAYS AS (DATE_FORMAT(order_date,'%Y-%m-01')) STORED;

CREATE INDEX ix_orders_month ON orders(order_month);
```
- 보고서/필터 공통식은 **생성 컬럼 + 인덱스**로 CPU·I/O를 절감.

#### (D) 인덱스 전략(커버링, 파셜/필터드, 함수형)
```sql
-- Postgres: 파셜 인덱스
CREATE INDEX ix_active_users ON users (last_login)
WHERE is_active = true;

-- SQL Server: Filtered Index
CREATE NONCLUSTERED INDEX ix_orders_open ON dbo.Orders(Status)
WHERE Status = 'OPEN';

-- 함수 기반(예: lower(email))
CREATE INDEX ix_users_email_norm ON users ((lower(email)));
```
- **커버링 인덱스**: SELECT 목록·JOIN 키·WHERE 컬럼을 포함해 **테이블 접근 회피**.
- **파셜/필터드**: 기수(카디널리티)가 높은 부분만 인덱싱해 **작고 빠른** 인덱스를 만든다.

---

## 2. DML(데이터 조작) — 트랜잭션, 격리수준, 벌크·머지, UPSERT

### 2.1 기본과 트랜잭션 제어
```sql
BEGIN;  -- 또는 START TRANSACTION
  INSERT INTO employee(emp_id, name, salary) VALUES (1001, '홍길동', 4000.00);
  UPDATE employee SET salary = salary * 1.1 WHERE emp_id = 1001;
  DELETE FROM employee WHERE emp_id = 9999;
COMMIT;  -- 또는 ROLLBACK;
```

#### 격리 수준 개요
| Isolation | 허용 현상 | 설명 |
|---|---|---|
| READ UNCOMMITTED | Dirty Read | 실무 비권장(대부분 엔진에서 READ COMMITTED와 유사/동일 처리) |
| READ COMMITTED | Non-repeatable Read, Phantom | 기본값(Oracle은 일종의 MVCC 일관 읽기) |
| REPEATABLE READ | Phantom(엔진별 상이) | MySQL InnoDB 기본, MVCC로 대부분 방지 |
| SERIALIZABLE | 없음 | 가장 안전·가장 느림(락/리트라이 증가) |

> 실무 기본: **READ COMMITTED** 또는 **REPEATABLE READ**. **핫 라인·정합성 중시** 구간만 SERIALIZABLE.

### 2.2 UPSERT/머지 이식성
```sql
-- PostgreSQL
INSERT INTO users(id, email, name)
VALUES (1, 'a@x.com', 'A')
ON CONFLICT (id) DO UPDATE SET email = EXCLUDED.email, name=EXCLUDED.name;

-- MySQL
INSERT INTO users(id, email, name)
VALUES (1, 'a@x.com', 'A')
ON DUPLICATE KEY UPDATE email=VALUES(email), name=VALUES(name);

-- SQL Server / Oracle
MERGE INTO users u
USING (SELECT 1 AS id, 'a@x.com' AS email, 'A' AS name) s
ON (u.id = s.id)
WHEN MATCHED THEN UPDATE SET u.email=s.email, u.name=s.name
WHEN NOT MATCHED THEN INSERT (id,email,name) VALUES (s.id,s.email,s.name);
```
- **충돌 키**와 **멱등성** 확보(동일 입력 반복 시 동일 결과).
- 고부하 UPSERT는 **배치/버퍼**로 묶고, **락 경합**·**보조 인덱스 갱신 비용**을 감안.

### 2.3 벌크 로딩/삭제 전략
```sql
-- (예) PostgreSQL: COPY(가장 빠른 바닥 프로토콜)
COPY big_table(col1,col2,...) FROM '/path/data.csv' CSV HEADER;

-- MySQL: 대량 삽입
LOAD DATA INFILE '/path/data.csv' INTO TABLE big_table
FIELDS TERMINATED BY ',' ENCLOSED BY '"' LINES TERMINATED BY '\n' IGNORE 1 LINES;

-- 대량 삭제는 파티션 DROP/스위치 또는 청크 삭제
DELETE FROM logs WHERE created_at < now() - INTERVAL '90 days' LIMIT 10000; -- 루프
```
- **대량 삭제는 청크/파티션 드롭**으로. 단일 대량 DELETE는 언두/리두·락·아카이빙 비용 폭증.

### 2.4 읽기 성능 패턴(커버링·스캔 회피·페이징)
```sql
-- 커버링 인덱스: 선택한 컬럼만 포함
CREATE INDEX ix_emp_cover ON employee (dept_id, salary) INCLUDE (name);  -- SQL Server식
-- Postgres는 INCLUDE, MySQL은 복합키 + 필요한 컬럼만 쿼리로 가져와서 커버링 유도

-- 키셋 페이징(Seek 기반)
SELECT id, title
FROM posts
WHERE (id, created_at) < (:last_id, :last_ts)
ORDER BY created_at DESC, id DESC
LIMIT 50;
```
- `OFFSET … LIMIT …`의 큰 OFFSET은 비용 큼 → **키셋 페이징**으로 변환.

---

## 3. DCL(접근 제어) — 롤 기반 설계, 최소권한, RLS

### 3.1 기본 권한
```sql
-- PostgreSQL
CREATE ROLE app_readonly;
GRANT CONNECT ON DATABASE appdb TO app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO app_readonly;

-- MySQL
CREATE ROLE 'app_readonly';
GRANT SELECT ON appdb.* TO 'app_readonly';
GRANT 'app_readonly' TO 'user_dev'@'%';

-- Oracle/SQL Server 유사: 역할 생성 → 객체/스키마 범위 부여
```
- **역할(Role) 우선**: 사용자에게 직접 권한 주지 말고, 역할에 GRANT → 사용자에 역할만 부여.

### 3.2 세분화: 컬럼/행 단위
```sql
-- PostgreSQL: Row Level Security(RLS)
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY own_orders ON orders
  USING (customer_id = current_setting('app.user_id')::BIGINT);

-- SQL Server: Security Policy + Predicate Function
CREATE FUNCTION dbo.fn_rowpredicate(@CustId INT)
RETURNS TABLE WITH SCHEMABINDING AS
RETURN SELECT 1 AS fn_result WHERE @CustId = CAST(SESSION_CONTEXT(N'user_id') AS INT);

CREATE SECURITY POLICY OrderFilter
ADD FILTER PREDICATE dbo.fn_rowpredicate(customer_id) ON dbo.Orders
WITH (STATE = ON);
```
- RLS는 강력하지만 오류/오동작 시 **전면 차단** 가능 → 철저한 통합 테스트가 필요.

### 3.3 REVOKE/소유권/위임
```sql
-- WITH GRANT OPTION 로 위임 시, 피수여자가 다시 제3자에게 권한 부여 가능
GRANT SELECT ON employee TO analyst WITH GRANT OPTION;
REVOKE GRANT OPTION FOR SELECT ON employee FROM analyst; -- 위임능력 회수
```
- 소유권(OWNER)·스키마 USAGE/CREATE 권한 혼동 주의.
- 회수 시 **의존 객체**(뷰/프로시저) **실행 실패** 가능성 분석 필요.

---

## 4. TCL(트랜잭션 제어) — 세이브포인트, 예외 복구, 배포 전략

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  SAVEPOINT after_debit;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;

  -- 예외 발생 시
  ROLLBACK TO SAVEPOINT after_debit; -- 두 번째 업데이트만 되돌림
COMMIT;
```

### 4.1 예외/리트라이(낙관적 동시성)
- **유니크 충돌/시리얼라이즈 실패**는 정상 경로. 클라이언트에서 **리트라이 루프**(최대 N회, 지터 포함) 구현.
- **비즈니스 멱등키**(예: `request_id`)를 DDL로 보장:
```sql
ALTER TABLE payments ADD CONSTRAINT uq_req UNIQUE (request_id);
```

### 4.2 배포 시퀀스(Blue/Green)
1) **DDL 사전 배포**: 새 칼럼 추가(널 허용/디폴트), 인덱스 백그라운드 빌드
2) **앱 이중 쓰기**: 구/신 칼럼 동시 기록(멱등)
3) **백필(Backfill)**: 배치로 과거 데이터 이전
4) **스위치 오버**: 앱 읽기 경로를 신 칼럼으로
5) **구 칼럼 정리**(DDL Drop) — 충분한 관찰 기간 후

---

## 5. 케이스 스터디 — 쇼핑몰 주문/정산 도메인

### 5.1 DDL: 핵심 테이블·제약
```sql
-- PostgreSQL
CREATE TABLE customer(
  id          BIGSERIAL PRIMARY KEY,
  email       CITEXT UNIQUE NOT NULL,
  name        TEXT NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE product(
  id          BIGSERIAL PRIMARY KEY,
  sku         TEXT UNIQUE NOT NULL,
  name        TEXT NOT NULL,
  price_cents INT  NOT NULL CHECK (price_cents >= 0)
);

CREATE TABLE "order"(
  id          BIGSERIAL PRIMARY KEY,
  customer_id BIGINT NOT NULL REFERENCES customer(id) ON DELETE RESTRICT,
  status      TEXT NOT NULL CHECK (status IN ('PENDING','PAID','CANCELLED','SHIPPED')),
  ordered_at  TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (ordered_at);

-- 월 파티션 예
CREATE TABLE order_2024_11 PARTITION OF "order"
  FOR VALUES FROM ('2024-11-01') TO ('2024-12-01');

CREATE TABLE order_item(
  order_id    BIGINT NOT NULL REFERENCES "order"(id) ON DELETE CASCADE,
  product_id  BIGINT NOT NULL REFERENCES product(id),
  qty         INT    NOT NULL CHECK (qty > 0),
  price_cents INT    NOT NULL,
  PRIMARY KEY(order_id, product_id)
);

-- 조회 최적화 인덱스
CREATE INDEX ix_order_customer_time ON "order"(customer_id, ordered_at DESC);
CREATE INDEX ix_item_product ON order_item(product_id);
```

### 5.2 DML: 멱등 주문 생성(UPSERT) + 재고 차감
```sql
-- 멱등키 도입
ALTER TABLE "order" ADD COLUMN request_id TEXT;
CREATE UNIQUE INDEX uq_order_request ON "order"(request_id);

-- 트랜잭션
BEGIN;
  -- 1) 주문 헤더(멱등)
  INSERT INTO "order"(customer_id, status, request_id)
  VALUES (:cid, 'PENDING', :req)
  ON CONFLICT (request_id)
  DO UPDATE SET customer_id = EXCLUDED.customer_id
  RETURNING id INTO :order_id;

  -- 2) 아이템 삽입
  INSERT INTO order_item(order_id, product_id, qty, price_cents)
  SELECT :order_id, p.id, i.qty, p.price_cents
  FROM product p
  JOIN UNNEST(:items) AS i(product_sku, qty) ON i.product_sku = p.sku;

  -- 3) 재고 차감(예: 별도 테이블 inventory로 관리한다면)
  UPDATE inventory SET stock = stock - i.qty
  FROM UNNEST(:items) AS i(product_sku, qty)
  WHERE inventory.sku = i.product_sku
    AND stock >= i.qty;

  -- 재고 부족 검증
  -- 영향 행수 검사 → 부족 시 ROLLBACK
COMMIT;
```

### 5.3 DCL: 역할 기반 접근 + RLS(고객 자신만 읽기)
```sql
-- 역할
CREATE ROLE app_read, app_write, app_admin;
GRANT USAGE ON SCHEMA public TO app_read, app_write, app_admin;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_write;

-- 고객별 RLS
ALTER TABLE "order" ENABLE ROW LEVEL SECURITY;
CREATE POLICY order_owner ON "order"
  USING (customer_id = current_setting('app.user_id')::BIGINT);
```

### 5.4 TCL: 부분 실패 복구
```sql
BEGIN;
  SAVEPOINT sp1;
  -- 결제 승인
  -- 실패 시 ROLLBACK TO sp1; 로 결제만 되돌리고 주문 헤더 유지(상태 UPDATE)
COMMIT;
```

---

## 6. 성능/안전 안티패턴과 교정

| 안티패턴 | 결과 | 교정 |
|---|---|---|
| WHERE 없이 UPDATE/DELETE | 전행 변경/삭제 | 테스트에 `SET ROWCOUNT`(MSSQL)/`LIMIT`(MySQL) + COALESCE 조건, 운영은 필수 키 조건 |
| 큰 OFFSET 페이징 | p95, p99 급증 | **키셋 페이징**, 커버링 인덱스 |
| 비선택적 인덱스 남발 | DML 느림, 스토리지 증대 | 실행계획·사용통계로 정리 |
| 대량 삭제 단발성 | 언두/리두 폭증, 락 경합 | **파티션 DROP** 또는 청크 삭제 배치 |
| 컬럼 타입 다운사이징 과소평가 | 오버플로·암묵 캐스팅 비용 | 도메인 정의서로 타입 범위 명확화 |
| TRUNCATE를 롤백 기대 | 데이터 영구 소실 | 운영 전 백업/스냅샷, 권한 분리 |

---

## 7. 이식성 체크리스트(DDL/DML/DCL/TCL)

- **자동 증가 키**: Postgres `SERIAL/IDENTITY`, MySQL `AUTO_INCREMENT`, SQL Server `IDENTITY`, Oracle `IDENTITY/SEQUENCE` — **표준화는 `GENERATED ... AS IDENTITY`**.
- **UPSERT**: `ON CONFLICT`(PG) / `ON DUPLICATE KEY`(MySQL) / `MERGE`(MSSQL/Oracle).
  프로젝트 표준 API로 추상화.
- **시간/타임존**: `TIMESTAMPTZ` vs `DATETIME`(TZ 없음). **UTC 저장** + 표시 시 변환.
- **텍스트/Collation**: 대소문자/정렬·비교 규칙 차이가 유니크 제약에 영향.
  Postgres `CITEXT`/정규화 인덱스, MySQL `utf8mb4` + collation 통일.
- **권한·스키마 모델**: 스키마/데이터베이스/사용자/롤 개념 차이. **롤 기반 표준**을 정해 DB별 매핑.

---

## 8. 감사(Compliance)·운영 거버넌스

- **DDL 변경 기록**: 마이그레이션 도구(Flyway/Liquibase)로 **버전드 DDL**.
- **DML 중요 테이블**: CDC(Change Data Capture)/감사 트리거/로그 테이블.
- **DCL 변경**: 권한 변경은 **티켓·리뷰** 필수, 정기 스캔(권한 드리프트 방지).
- **백업/복구 리허설**: PITR(Point-in-time Recovery) 정기 점검.

---

## 9. 연습용 미니 프로젝트 스크립트(통합)

### 9.1 스키마(DDL)
```sql
-- 공용 스키마
CREATE TABLE dept (
  id   BIGINT PRIMARY KEY,
  name TEXT UNIQUE NOT NULL
);

CREATE TABLE employee (
  id        BIGSERIAL PRIMARY KEY,
  email     TEXT UNIQUE NOT NULL,
  name      TEXT NOT NULL,
  dept_id   BIGINT NOT NULL REFERENCES dept(id),
  salary    NUMERIC(12,2) CHECK (salary >= 0),
  hired_at  TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX ix_emp_dept_hired ON employee(dept_id, hired_at DESC);
```

### 9.2 데이터 조작(DML) + 트랜잭션(TCL)
```sql
BEGIN;
  INSERT INTO dept(id,name) VALUES (10,'Sales'),(20,'Tech') ON CONFLICT DO NOTHING;
  INSERT INTO employee(email,name,dept_id,salary)
  VALUES ('a@x.com','A',20,5500.00), ('b@x.com','B',10,4300.00);

  UPDATE employee SET salary = salary * 1.05 WHERE dept_id = 20;

  -- 안전 삭제: 키 조건 사용
  DELETE FROM employee WHERE email = 'none@x.com';
COMMIT;
```

### 9.3 권한(DCL)
```sql
-- 읽기 전용 롤과 사용자
CREATE ROLE report_reader;
GRANT USAGE ON SCHEMA public TO report_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO report_reader;

-- 개발자 계정에 롤 부여
GRANT report_reader TO dev_analyst;
```

---

## 10. 정리 — 실무에서의 작동 순서·원칙

1. **DDL**: 스키마/인덱스/제약 설계 → 온라인 DDL 여부 확인 → 배포 창/백업 계획.
2. **DCL**: 롤 기반 최소권한 → 컬럼/행 단위 제어 필요 시 RLS·마스킹.
3. **DML**: 키셋 페이징·커버링 인덱스·UPSERT 멱등키 → 대량 작업은 벌크/파티션.
4. **TCL**: 격리 수준 합의 → SAVEPOINT/리트라이 패턴 → 배포 단계별 이중쓰기·백필.
5. **운영**: 마이그레이션 버저닝·CDC/감사·권한 리뷰 → 백업/복구 연습.

> **핵심 요지**:
> - DDL은 **가용성/락/암묵 커밋**을 동반하니, **온라인 전략**과 **배포 계획**이 필요하다.
> - DML은 **인덱스/격리/멱등성**으로 안전·빠르게.
> - DCL은 **롤 중심 최소권한**으로 감사를 견딘다.
> - TCL은 **리트라이·세이브포인트**로 부분 실패를 흡수한다.
> - 네 요소를 한 세트로 운영할 때, 스키마는 **안정적**이고 쿼리는 **빠르며**, 보안은 **감사 가능**해진다.
