---
layout: post
title: DB 심화 - Random Access 최소화 튜닝
date: 2025-11-07 18:25:23 +0900
category: DB 심화
---
# 테이블 Random Access 최소화 튜닝

> **핵심 원리**
> 데이터베이스 성능의 주요 병목점은 테이블에 대한 랜덤 접근입니다. 이 접근 방식은 디스크 I/O 대기 시간을 증가시키고 전체 응답 시간을 저하시킵니다. 효과적인 튜닝은 테이블 랜덤 접근을 최소화하거나 완전히 제거하는 것을 목표로 합니다.

---

## 실습 환경 설정

```sql
-- 테스트 테이블 생성
DROP TABLE ra_orders PURGE;

-- 주문 테이블 생성 (50만 건)
CREATE TABLE ra_orders (
  order_id    NUMBER        NOT NULL,
  cust_id     NUMBER        NOT NULL,
  order_dt    DATE          NOT NULL,
  status      VARCHAR2(8)   NOT NULL,
  amount      NUMBER(12,2)  NOT NULL,
  note        VARCHAR2(200),
  CONSTRAINT pk_ra_orders PRIMARY KEY (order_id)
);

-- 샘플 데이터 삽입
BEGIN
  FOR i IN 1..500000 LOOP
    INSERT INTO ra_orders
    VALUES (
      i,
      MOD(i, 100000) + 1,  -- 10만 고객
      DATE '2024-01-01' + MOD(i, 400),  -- 400일 범위
      CASE MOD(i,5) 
        WHEN 0 THEN 'NEW' 
        WHEN 1 THEN 'PAID'
        WHEN 2 THEN 'SHIP' 
        WHEN 3 THEN 'DONE' 
        ELSE 'CANC' 
      END,
      ROUND(DBMS_RANDOM.VALUE(10, 99999), 2),
      CASE WHEN MOD(i,137)=0 THEN 'gift' END
    );
  END LOOP;
  COMMIT;
END;
/

-- 조회 패턴용 인덱스 생성
CREATE INDEX ix_orders_cdt ON ra_orders(cust_id, order_dt);

-- 통계 정보 수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 
    'RA_ORDERS', 
    cascade => TRUE,
    method_opt => 'for all columns size skewonly'
  );
END;
/

-- 상세 실행 통계 수집 활성화
ALTER SESSION SET statistics_level = ALL;
```

---

## 테이블 랜덤 접근의 문제점 이해

### 성능 병목의 본질
현대 데이터베이스 시스템에서 테이블 랜덤 접근은 다음과 같은 이유로 성능에 치명적입니다:

1. **I/O 대기 시간**: 랜덤 디스크 접근은 순차 접근보다 수백 배 이상 느립니다.
2. **캐시 효율성 저하**: 버퍼 캐시 활용도가 낮아져 메모리 효율성이 감소합니다.
3. **확장성 제한**: 데이터량 증가에 따른 성능 저하가 선형 이상으로 발생합니다.

### 성능 모델 간소화
테이블 랜덤 접근의 성능 영향을 다음과 같이 모델링할 수 있습니다:

```
총 응답 시간 ≈ (인덱스 접근 횟수 × 인덱스 접근 시간) 
                + (테이블 랜덤 접근 횟수 × 랜덤 접근 시간)
```

이 공식에서 테이블 랜덤 접근 시간이 지배적인 요소이므로, 튜닝의 최우선 목표는 테이블 랜덤 접근 횟수를 최소화하는 것입니다.

---

## 3단계 최적화 전략

효과적인 테이블 랜덤 접근 최소화를 위한 체계적 접근법:

### 1단계: 커버링 인덱스 활용
테이블 접근 없이 인덱스만으로 쿼리를 완결할 수 있도록 설계합니다.

### 2단계: 정렬 최적화와 Stopkey 활용
필요한 데이터만 읽고 조기 종료하도록 쿼리를 설계합니다.

### 3단계: 클러스터링 팩터 개선
데이터의 물리적 저장 순서를 논리적 접근 패턴과 일치시킵니다.

---

## 1단계: 커버링 인덱스 구현

### 문제점 진단
기존 인덱스로는 테이블 접근이 필수적입니다:

```sql
-- 테이블 접근이 필요한 쿼리
SELECT order_id, order_dt, status, amount
FROM ra_orders
WHERE cust_id = 12345
  AND order_dt >= SYSDATE - 30
ORDER BY order_dt, order_id;
```

**실행 계획 분석**:
```
INDEX RANGE SCAN → TABLE ACCESS BY INDEX ROWID BATCHED → SORT ORDER BY
```

### 커버링 인덱스 설계
```sql
-- 모든 필요한 컬럼을 포함하는 커버링 인덱스
CREATE INDEX ix_orders_cdt_cov ON ra_orders(
  cust_id, 
  order_dt, 
  order_id, 
  status, 
  amount
);

-- 인덱스 통계 수집
BEGIN
  DBMS_STATS.GATHER_INDEX_STATS(USER, 'IX_ORDERS_CDT_COV');
END;
/
```

### 성능 개선 효과
커버링 인덱스 적용 후:
- 테이블 접근이 완전히 제거됨
- 정렬 작업이 필요 없어짐 (인덱스 순서와 정렬 요구사항 일치)
- I/O 대기 시간이 현저히 감소

---

## 2단계: 정렬 최적화와 Stopkey 패턴

### 최신 데이터 조회 최적화
```sql
-- DESC 정렬을 위한 인덱스
CREATE INDEX ix_orders_cdt_desc_cov ON ra_orders(
  cust_id, 
  order_dt DESC, 
  order_id DESC, 
  status, 
  amount
);

-- Stopkey를 활용한 효율적인 조회
SELECT /*+ index(ra_orders ix_orders_cdt_desc_cov) */
       order_id, order_dt, amount, status
FROM ra_orders
WHERE cust_id = :cust_id
ORDER BY order_dt DESC, order_id DESC
FETCH FIRST 50 ROWS ONLY;
```

### Keyset 페이지네이션 구현
OFFSET 방식의 비효율성을 해결하기 위한 접근법:

```sql
-- Keyset 페이지네이션 (커서 기반)
SELECT order_id, order_dt, amount, status
FROM ra_orders
WHERE cust_id = :cust_id
  AND (order_dt < :last_date 
       OR (order_dt = :last_date AND order_id < :last_id))
ORDER BY order_dt DESC, order_id DESC
FETCH FIRST :page_size ROWS ONLY;
```

**장점**:
- 각 페이지마다 필요한 데이터만 읽음
- 페이지 번호 증가에 따른 성능 저하 없음
- 대량 데이터에서도 일관된 성능 유지

---

## 3단계: 클러스터링 팩터 개선

### 클러스터링 팩터 이해
클러스터링 팩터는 인덱스 키 순서와 테이블의 물리적 저장 순서 간 일치도를 나타냅니다:

```sql
-- 현재 클러스터링 팩터 확인
SELECT index_name, 
       clustering_factor, 
       leaf_blocks,
       num_rows
FROM user_indexes
WHERE table_name = 'RA_ORDERS';
```

### 물리적 재구성
```sql
-- 1. 정렬된 순서로 새 테이블 생성
CREATE TABLE ra_orders_reorganized AS
SELECT * FROM ra_orders
ORDER BY cust_id, order_dt, order_id;

-- 2. 테이블 교체
ALTER TABLE ra_orders RENAME TO ra_orders_backup;
ALTER TABLE ra_orders_reorganized RENAME TO ra_orders;

-- 3. 인덱스 재생성
CREATE INDEX ix_orders_cdt_opt ON ra_orders(cust_id, order_dt);

-- 4. 통계 재수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'RA_ORDERS', cascade => TRUE);
END;
/
```

### 운영 환경 고려사항
운영 환경에서는 다음 접근법을 고려하세요:

1. **파티션 단위 재구성**: `MOVE PARTITION` 사용
2. **점진적 개선**: 피크 시간대를 피해 단계적 실행
3. **모니터링**: 재구성 전후 성능 비교

---

## 기본 키 인덱스 설계 고려사항

### 현실적인 제약사항
Oracle에서는 기본 키 제약과 연결된 인덱스의 컬럼 집합이 동일해야 합니다. 이는 중요한 설계 제약입니다.

### 실용적 해결책
```sql
-- 별도 커버링 인덱스 생성 (가장 안전한 접근법)
CREATE INDEX ix_orders_pk_cover ON ra_orders(
  order_id, 
  order_dt, 
  status, 
  amount
);

-- 필요한 경우 유니크 인덱스 활용
CREATE UNIQUE INDEX ux_orders_pk_cover ON ra_orders(
  order_id, 
  order_dt, 
  status
);
```

### 기본 키 변경의 위험성
기본 키 재정의는 다음 경우에만 고려하세요:
- 읽기 전용 또는 거의 읽기 전용 시스템
- 참조 무결성 영향이 없는 경우
- 업무 규칙에 중대한 변화가 없는 경우

---

## 조인 쿼리에서의 랜덤 접근 최적화

### 네스티드 루프 조인 최적화
```sql
-- 내부 테이블에 커버링 인덱스 적용
CREATE INDEX ix_orders_join_cover ON orders(
  customer_id,
  order_date,
  order_id,
  total_amount
);

-- 최적화된 조인 쿼리
SELECT /*+ ORDERED USE_NL(o) index(o ix_orders_join_cover) */
       c.customer_id, 
       c.customer_name, 
       o.order_id, 
       o.total_amount
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE c.region = :region
  AND c.active_flag = 'Y';
```

### 해시 조인 전환 고려
내부 테이블 튜닝이 어려운 경우 조인 전략을 변경하세요:

```sql
SELECT /*+ LEADING(c) USE_HASH(o) FULL(o) */
       c.customer_id, 
       c.customer_name, 
       o.order_id, 
       o.total_amount
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE c.region = :region
  AND c.active_flag = 'Y';
```

---

## 안전한 배포를 위한 검증 프로세스

### Invisible 인덱스 활용
```sql
-- 검증용 인덱스 생성 (기본적으로 사용되지 않음)
CREATE INDEX ix_orders_test_cover ON ra_orders(
  cust_id, 
  order_dt, 
  order_id, 
  status
) INVISIBLE;

-- 테스트 세션에서만 인덱스 사용
ALTER SESSION SET optimizer_use_invisible_indexes = TRUE;

-- 성능 테스트 실행
SELECT order_id, order_dt, status, amount
FROM ra_orders
WHERE cust_id = :cust_id
  AND order_dt >= SYSDATE - 30
ORDER BY order_dt, order_id;

-- 실행 계획 분석
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL, NULL, 'ALLSTATS LAST +PREDICATE'
));
```

### 검증 후 운영 반영
```sql
-- 검증 완료 후 인덱스 활성화
ALTER INDEX ix_orders_test_cover VISIBLE;

-- 기존 인덱스 제거 (필요시)
DROP INDEX ix_orders_cdt;
```

---

## 종합 성능 테스트 프레임워크

### A/B 테스트 구현
```sql
-- 성능 측정 환경 설정
ALTER SESSION SET statistics_level = ALL;

-- 테스트 케이스 A: 기존 방식
SELECT /*+ TEST_A */ order_id, order_dt, status, amount
FROM ra_orders
WHERE cust_id = 12345
  AND order_dt >= SYSDATE - 30
ORDER BY order_dt, order_id;

-- 테스트 케이스 B: 최적화된 방식
SELECT /*+ TEST_B index(ra_orders ix_orders_cdt_cov) */
       order_id, order_dt, status, amount
FROM ra_orders
WHERE cust_id = 12345
  AND order_dt >= SYSDATE - 30
ORDER BY order_dt, order_id;

-- 성능 비교 분석
SELECT sql_id,
       plan_hash_value,
       executions,
       buffer_gets,
       disk_reads,
       elapsed_time/1000000 as elapsed_seconds,
       SUBSTR(sql_text, 1, 50) as sql_snippet
FROM v$sql
WHERE sql_text LIKE '%TEST_%'
ORDER BY sql_id;
```

### 클러스터링 팩터 모니터링
```sql
-- 인덱스 효율성 지표 추적
CREATE TABLE index_efficiency_monitor (
    monitor_date DATE DEFAULT SYSDATE,
    index_name VARCHAR2(30),
    clustering_factor NUMBER,
    leaf_blocks NUMBER,
    distinct_keys NUMBER,
    num_rows NUMBER
);

-- 정기적 모니터링
INSERT INTO index_efficiency_monitor 
SELECT SYSDATE,
       index_name,
       clustering_factor,
       leaf_blocks,
       distinct_keys,
       num_rows
FROM user_indexes
WHERE table_name = 'RA_ORDERS';
COMMIT;
```

---

## 일반적인 문제와 해결책

### 문제 1: 인덱스가 사용되지 않는 경우
**증상**: 옵티마이저가 테이블 풀 스캔을 선택

**해결책**:
1. 통계 정보 갱신
2. 인덱스 힌트 활용
3. 인덱스 재생성
4. 쿼리 재작성

### 문제 2: 과도한 랜덤 I/O
**증상**: 높은 `db file sequential read` 대기 시간

**해결책**:
1. 커버링 인덱스 도입
2. 클러스터링 팩터 개선
3. 배치 처리 크기 조정
4. 조인 전략 변경

### 문제 3: 인덱스 유지보수 오버헤드
**증상**: DML 성능 저하, 인덱스 크기 과다

**해결책**:
1. 필요한 인덱스만 유지
2. 정기적 인덱스 재구성
3. 파티셔닝 활용
4. 압축 인덱스 고려

---

## 결론: 테이블 랜덤 접근 최소화의 체계적 접근법

효과적인 데이터베이스 성능 최적화를 위해서는 테이블 랜덤 접근을 체계적으로 최소화해야 합니다. 이를 위한 종합적인 전략은 다음과 같습니다:

### 1. 근본적 설계 원칙
- **커버링 인덱스 우선**: 자주 실행되는 쿼리의 SELECT 리스트를 인덱스로 커버하도록 설계
- **정렬 요구사항 반영**: ORDER BY 절을 인덱스 정렬 순서와 일치시킴
- **조인 패턴 고려**: 자주 조인되는 컬럼을 인덱스에 포함

### 2. 구현 단계 최적화
- **Stopkey 활용**: `FETCH FIRST N ROWS`와 정렬 일치 인덱스 조합
- **Keyset 페이지네이션**: OFFSET 방식 대신 커서 기반 페이지네이션 구현
- **물리적 데이터 정렬**: 클러스터링 팩터 개선을 위한 데이터 재배치

### 3. 성능 측정과 검증
- **실제 데이터 기반 테스트**: 운영 환경과 유사한 데이터 볼륨에서 테스트
- **A/B 테스트 구현**: 변경 전후 성능을 정량적으로 비교
- **실행 계획 분석**: `DBMS_XPLAN`을 활용한 상세 분석

### 4. 운영 환경 고려사항
- **점진적 배포**: Invisible 인덱스를 통한 안전한 테스트
- **모니터링 체계**: 성능 지표의 지속적 추적
- **유지보수 계획**: 정기적인 인덱스 재구성 및 통계 갱신

### 5. 조직적 협업
- **개발 표준 수립**: 인덱스 설계와 쿼리 작성 가이드라인 마련
- **교육 프로그램**: 개발자 대상 성능 최적화 교육
- **코드 리뷰 프로세스**: 성능 관련 코드 리뷰 항목 추가

### 성공적인 최적화의 핵심
테이블 랜덤 접근 최소화의 성공은 단순한 기술적 조치를 넘어 데이터 접근 패턴 이해, 비즈니스 요구사항 분석, 시스템 아키텍처 고려가 통합된 종합적 접근에서 비롯됩니다. 모든 최적화 결정은 실제 성능 데이터에 기반해야 하며, 지속적인 모니터링과 개선을 통해 안정적이고 효율적인 데이터베이스 환경을 구축할 수 있습니다.

최종적으로, 랜덤 접근 최소화는 더 빠른 응답 시간, 더 높은 처리량, 더 낮은 운영 비용이라는 가시적 성과로 이어집니다. 이러한 개선은 단순한 기술적 성취를 넘어 비즈니스 가치 창출에 직접적으로 기여합니다.