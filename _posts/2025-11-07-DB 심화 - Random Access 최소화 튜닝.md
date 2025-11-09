---
layout: post
title: DB 심화 - Random Access 최소화 튜닝
date: 2025-11-07 18:25:23 +0900
category: DB 심화
---
# 테이블 **Random Access 최소화 튜닝**

## 0) 실습 스키마 준비

```sql
-- 깨끗이
DROP TABLE ra_orders PURGE;

-- 주문 테이블: 50만 건(랜덤 I/O 체감용)
CREATE TABLE ra_orders (
  order_id    NUMBER        NOT NULL,
  cust_id     NUMBER        NOT NULL,
  order_dt    DATE          NOT NULL,
  status      VARCHAR2(8)   NOT NULL,
  amount      NUMBER(12,2)  NOT NULL,
  note        VARCHAR2(200),
  CONSTRAINT pk_ra_orders PRIMARY KEY (order_id)   -- 단일 PK (surrogate)
);

BEGIN
  FOR i IN 1..500000 LOOP
    INSERT INTO ra_orders
    VALUES (
      i,
      MOD(i, 100000) + 1,                                -- 10만 고객
      DATE '2024-01-01' + MOD(i, 400),                   -- 400일 범위
      CASE MOD(i,5) WHEN 0 THEN 'NEW' WHEN 1 THEN 'PAID'
                    WHEN 2 THEN 'SHIP' WHEN 3 THEN 'DONE' ELSE 'CANC' END,
      ROUND(DBMS_RANDOM.VALUE(10, 99999), 2),
      CASE WHEN MOD(i,137)=0 THEN 'gift' END
    );
  END LOOP;
  COMMIT;
END;
/

-- 조회 패턴용 보조 인덱스
CREATE INDEX ix_orders_cdt ON ra_orders(cust_id, order_dt);

-- 통계
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'RA_ORDERS', cascade => TRUE,
    method_opt => 'for all columns size skewonly');
END;
/

ALTER SESSION SET statistics_level = ALL; -- ALLSTATS LAST를 위해
```

---

# 1) 랜덤 I/O(테이블 랜덤 액세스) 최소화의 **핵심 논리**

- 인덱스 스캔 후 **ROWID로 힙 테이블을 찌르는** 오퍼레이터:  
  `TABLE ACCESS BY INDEX ROWID [BATCHED]` → **db file sequential read**(싱글블록, 랜덤).
- 랜덤 I/O가 많은 이유
  - **SELECT-LIST**에 인덱스에 없는 컬럼이 있어 테이블을 **반드시** 가야 함.
  - 인덱스 **키 순서 ↔ 물리 저장 순서**(CF)가 나빠 **같은 범위**도 **멀리 떨어진 블록**을 계속 건드림.
  - **Nested Loops** 내부 테이블 쿼리에서 **소량 다회** 랜덤 접근 폭증.
- **최소화 전략 3단계**
  1) **커버링(인덱스 컬럼 추가)**로 **테이블 미방문화**.
  2) **정렬 일치/Stopkey**로 **앞부분만** 읽고 종료.
  3) **CF 개선**(물리 재배치/키 순서 설계)로 **랜덤→준순차**화.

수학 감각(아주 간단히):
$$
\text{Total Time} \approx N_{\text{idx}} \cdot t_{\text{idx}} \;+\; N_{\text{tab}} \cdot t_{\text{random}}
$$
여기서 \(t_{\text{random}} \gg t_{\text{idx}}\). **관건은 \(N_{\text{tab}}\)**(테이블 랜덤 접근 수)을 **0 또는 최소**로 만드는 것.

---

# 2) **인덱스 컬럼 추가**로 테이블 방문 제거(=Index-Only/커버링)

## 2.1 베이스라인: 테이블 랜덤 방문이 발생하는 예

```sql
-- 고객 12345의 최근 30일 주문 목록(금액, 상태까지 필요)
SELECT order_id, order_dt, status, amount
FROM   ra_orders
WHERE  cust_id = 12345
AND    order_dt >= SYSDATE - 30
ORDER  BY order_dt, order_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```

**예상 계획(핵심)**
- `INDEX RANGE SCAN IX_ORDERS_CDT`
- `TABLE ACCESS BY INDEX ROWID RA_ORDERS`  ← **여기서 랜덤 I/O 발생**
- Buffers/Reads가 생각보다 큼(행 수↑일수록 체감↑).

## 2.2 커버링 인덱스로 개선

> **목표**: SELECT-LIST(`order_id, order_dt, status, amount`)와 정렬 키를 **인덱스로 충족** → 테이블 미방문.

```sql
-- 커버링 인덱스 추가: (cust_id, order_dt, order_id, status, amount)
-- (Oracle은 INCLUDE 절이 없으므로 "말미에 컬럼을 추가"해 키에 포함)
CREATE INDEX ix_orders_cdt_cov
  ON ra_orders(cust_id, order_dt, order_id, status, amount);

BEGIN
  DBMS_STATS.GATHER_INDEX_STATS(USER, 'IX_ORDERS_CDT_COV');
END;
/
```

다시 실행:

```sql
SELECT /* 커버링 기대: 테이블 미방문 */
       order_id, order_dt, status, amount
FROM   ra_orders
WHERE  cust_id = 12345
AND    order_dt >= SYSDATE - 30
ORDER  BY order_dt, order_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```

**개선 포인트**
- `TABLE ACCESS BY INDEX ROWID` 단계가 **사라지거나** 극히 감소(=인덱스만)  
- **정렬 일치**(order_dt, order_id)로 `SORT ORDER BY`도 제거 가능  
- **Buffers/Reads** 대폭 감소 → **응답시간 급감**

> Trade-off: 인덱스 폭/크기 증가(쓰기 부하↑, 저장공간↑). **핫 경로**에만 선택적으로 적용하세요.

---

# 3) 정렬까지 인덱스로 해결(Stopkey/Keyset와 궁합)

```sql
-- 최신 50건(특정 고객)
SELECT /* Stopkey로 앞부분만 읽고 끝내기 */
       order_id, order_dt, amount, status
FROM   ra_orders
WHERE  cust_id = :cid
ORDER  BY order_dt DESC, order_id DESC
FETCH FIRST 50 ROWS ONLY;
```

- 인덱스를 `(cust_id, order_dt DESC, order_id DESC, ...)`로 정의하면  
  **정렬 제거** + **맨 앞 Leaf 몇 블록**만 읽고 종료(**랜덤 I/O 최소**).

```sql
-- 정렬 일치 인덱스
CREATE INDEX ix_orders_cdt_desc_cov
  ON ra_orders(cust_id, order_dt DESC, order_id DESC, status, amount);

-- 검증
SELECT /*+ index_desc(ra_orders ix_orders_cdt_desc_cov) */
       order_id, order_dt, amount, status
FROM   ra_orders
WHERE  cust_id = :cid
ORDER  BY order_dt DESC, order_id DESC
FETCH FIRST 50 ROWS ONLY;
```

---

# 4) **PK 인덱스에 컬럼 추가** — 무엇이 가능하고, 어떻게 해야 안전한가

## 4.1 중요한 전제
- **PK 인덱스는 PK 제약의 컬럼 집합**과 **동일**해야 합니다.  
  “**기존 PK 컬럼은 그대로 두고 인덱스에만 컬럼을 더하는**” 행위는 **불가**합니다.  
- 즉, **PK 인덱스에 컬럼을 더하려면 PK 제약(컬럼 집합) 자체를 변경**해야 합니다.

> 대부분의 경우, **PK는 그대로** 두고 **별도 커버링 인덱스**를 추가하는 것이 **정답**입니다.  
> (논리 키 변경은 파급이 매우 큼)

## 4.2 그러나, **PK 기반 조회가 압도적**이고 **IOT/PK 조인**로직이 많아  
**PK 인덱스 자체를 커버링**하고 싶다면? (드문 케이스, DW/Read-heavy에서만 고려)

### 선택지 A) **PK 재정의(복합 PK)** — 실제로 PK 컬럼 집합을 확장
- 장점: **PK 기반 대부분 쿼리 커버링** 가능, 테이블 미방문화↑  
- 단점: **업무 스키마 전반 영향(외래키, 조인, 중복 허용성 등)**

**절차 예시(테스트 전용! 운영은 신중히)**
```sql
-- 1) 기존 PK 제약 이름 확인
SELECT constraint_name FROM user_constraints
WHERE table_name = 'RA_ORDERS' AND constraint_type = 'P';

-- 2) PK 드롭(외래키 등 의존성 고려!)
ALTER TABLE ra_orders DROP CONSTRAINT pk_ra_orders;

-- 3) PK 재생성: (order_id, order_dt, status) 예시
--    (논리적으로 "고유성"이 보장되어야만 안전)
ALTER TABLE ra_orders
  ADD CONSTRAINT pk_ra_orders
  PRIMARY KEY (order_id, order_dt, status)
  USING INDEX;

-- 4) 통계 수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'RA_ORDERS', cascade => TRUE);
END;
/
```
> **강조**: 이 방식은 **진짜 PK**가 바뀝니다. 대부분 시스템에서는 **권장하지 않습니다**.

### 선택지 B) **PK는 유지**, **PK + 추가 컬럼**의 **유니크 인덱스**를 별도 생성
- PK 제약은 안 건드림.  
- PK 조인에 **추가 컬럼**이 자주 필요할 때, 옵티마이저가 **유니크 인덱스**를 더 선호하기도.

```sql
CREATE UNIQUE INDEX ux_orders_pk_cover
  ON ra_orders(order_id, order_dt, status);

-- 쿼리 상황에 따라 pk 대신 이 인덱스를 타며 커버링 효과 획득
```

### 선택지 C) **별도 비유니크 커버링 인덱스**(가장 현실적)
```sql
-- PK는 그대로. 커버링만 달성
CREATE INDEX ix_orders_pk_cover
  ON ra_orders(order_id, order_dt, status, amount);
```

**결론**  
- **대부분**: **PK는 변경하지 말고** 커버링 인덱스(혹은 INDEX JOIN)로 **랜덤 I/O 제거**.  
- **정말 특수한 읽기 전용 시나리오**에서만 PK 재정의 고려.

---

# 5) **클러스터링 팩터(CF)**와 “컬럼 추가”의 상호작용

## 5.1 CF란?
- 인덱스 **키 순서대로** 테이블을 읽을 때, **테이블 블록 재사용 정도**를 나타내는 척도.  
- `USER_INDEXES.CLUSTERING_FACTOR` 로 조회(작을수록 좋음, 테이블 블록 수에 근접할수록 이상적).
- **나쁜 CF**: 인덱스 순서로 걷는 동안 **테이블 블록 점프**가 빈번 → 랜덤 I/O↑

```sql
SELECT index_name, clustering_factor, leaf_blocks, distinct_keys
FROM   user_indexes
WHERE  table_name = 'RA_ORDERS';
```

## 5.2 컬럼 추가가 CF에 미치는 영향
- **선행 컬럼이 CF를 사실상 지배**합니다.  
  - 인덱스 키 순서 = `(선행1, 선행2, …, 후행…)`  
  - **선행 컬럼**에 의해 Leaf 체인 순서가 결정 → **CF의 핵심**은 **선행 컬럼과 물리 순서의 상관**.
- **후행 컬럼만 추가**해도 상황이 변할 수 있음  
  - **후행 컬럼에 따라 Leaf 엔트리 재배치**가 발생 → **스캔 순서**가 미세 조정  
  - 후행 컬럼이 **물리 저장 순서와 상관↑**이면 **CF가 개선**될 수 있고,  
    상관이 낮으면 **오히려 악화**될 수도.

## 5.3 실험: 같은 선행, 후행만 다르게

```sql
-- 기준 인덱스: (cust_id, order_dt)
CREATE INDEX ix_cdt_base ON ra_orders(cust_id, order_dt);
-- 변형1: (cust_id, order_dt, order_id)   -- 물리 삽입 순서와 상관성↑ 가정
CREATE INDEX ix_cdt_oid  ON ra_orders(cust_id, order_dt, order_id);
-- 변형2: (cust_id, order_dt, status)     -- 상태는 물리 순서와 상관 낮을 수 있음
CREATE INDEX ix_cdt_st   ON ra_orders(cust_id, order_dt, status);

SELECT index_name, clustering_factor
FROM   user_indexes
WHERE  table_name='RA_ORDERS'
  AND  index_name IN ('IX_CDT_BASE','IX_CDT_OID','IX_CDT_ST');
```

**관찰 가이드**
- `IX_CDT_OID`의 CF가 `IX_CDT_BASE`보다 **작아질 수 있음**(케이스에 따라)  
- `IX_CDT_ST`는 오히려 **커질 수도**(상관 낮을 때)  
- **정답은 실측**: 업무 데이터 분포, 로딩 방식, 파티셔닝 유무에 따라 달라짐.

## 5.4 CF “근본 개선”: 물리 재배치(CTAS) + 인덱스 재생성

```sql
CREATE TABLE ra_orders_sorted NOLOGGING AS
SELECT * FROM ra_orders
ORDER  BY cust_id, order_dt, order_id;

ALTER TABLE ra_orders RENAME TO ra_orders_old;
ALTER TABLE ra_orders_sorted RENAME TO ra_orders;

-- 필요한 인덱스 재생성
DROP INDEX ix_orders_cdt;
CREATE INDEX ix_orders_cdt ON ra_orders(cust_id, order_dt, order_id);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'RA_ORDERS', cascade=>TRUE);
END;
/

-- CF 비교
SELECT index_name, clustering_factor
FROM   user_indexes
WHERE  table_name='RA_ORDERS';
```

**효과**: 같은 인덱스 구조라도 **물리 정렬**만으로 CF 큰 폭 개선 → **랜덤 I/O↓**.

---

# 6) 운영 반영 시 **안전 절차**: Invisible/Online/검증

1) **Invisible Index**로 새 설계를 **그림자 배치**
```sql
CREATE INDEX ix_orders_cdt_cov_inv
  ON ra_orders(cust_id, order_dt, order_id, status, amount) INVISIBLE;
```
2) 테스트 세션에서만 사용
```sql
ALTER SESSION SET optimizer_use_invisible_indexes = TRUE;

-- 후보 경로가 붙는지, Buffers/Reads 개선되는지 검증
SELECT order_id, order_dt, status, amount
FROM ra_orders
WHERE cust_id = :cid AND order_dt >= SYSDATE-30
ORDER BY order_dt, order_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```
3) **안정성 확인 후 Visible 전환**
```sql
ALTER INDEX ix_orders_cdt_cov_inv VISIBLE;
```
4) **Online**(에디션/다운타임 제약 시) 옵션 검토(버전별 기능 상이)

---

# 7) 조인/페이지 시나리오와의 결합

- **Nested Loops** 내부 테이블이 `TABLE BY ROWID` 폭탄이면  
  → **커버링 인덱스**로 내부 테이블을 **미방문화**하거나, **Hash Join**으로 전략 전환.  
- **페이징**은 **Keyset**으로: `OFFSET`은 앞 페이지를 버리느라 **랜덤 누적**.  
- **정렬 요구**가 강하면 **인덱스 정렬과 일치**하도록 컬럼 순서/방향(DESC)까지 설계.

---

# 8) 종합 실험 템플릿

```sql
ALTER SESSION SET statistics_level = ALL;

-- A. 베이스라인
SELECT /*A*/ order_id, order_dt, status, amount
FROM ra_orders
WHERE cust_id = 12345
  AND order_dt >= SYSDATE - 30
ORDER BY order_dt, order_id;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));

-- B. 커버링 인덱스 적용 후
SELECT /*B*/ order_id, order_dt, status, amount
FROM ra_orders
WHERE cust_id = 12345
  AND order_dt >= SYSDATE - 30
ORDER BY order_dt, order_id;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));

-- C. DESC + Stopkey (최신 50건)
SELECT /*C*/ order_id, order_dt, status, amount
FROM ra_orders
WHERE cust_id = 12345
ORDER BY order_dt DESC, order_id DESC
FETCH FIRST 50 ROWS ONLY;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));

-- D. CF 비교
SELECT index_name, clustering_factor
FROM   user_indexes
WHERE  table_name='RA_ORDERS';
```

**관찰 포인트**
- `TABLE ACCESS BY INDEX ROWID` 유무/비율, `Buffers`, `Reads`  
- `SORT ORDER BY` 제거 여부(정렬 일치/Stopkey)  
- CF 변화(작을수록 좋음) 및 계획 안정성

---

# 9) 자주 묻는 질문(FAQ)

**Q1. “PK 인덱스에 컬럼만 ‘추가’하면 되나요?”**  
A. **아니요.** PK 인덱스는 **PK 제약 컬럼과 동일**해야 합니다. 컬럼을 더하려면 **PK 자체를 재정의**해야 하며,  
대부분 시스템에서 **권장되지 않습니다**. 대신 **별도 커버링 인덱스**를 만드세요.

**Q2. 커버링 인덱스를 많이 만들면 왜 안 좋죠?**  
A. 쓰기 비용/공간/경합이 증가합니다. **핫 경로**(QPS·SLI에 직접 영향) 위주로 **최소 집합**만 유지하세요.

**Q3. 컬럼을 뒤에만 살짝 더하면 CF는 안 바뀌죠?**  
A. **바뀔 수 있습니다.** 인덱스 엔트리의 정렬/분포가 달라지면 **Leaf 순회 순서**가 달라지고, 그 순서와 물리 저장 순서의 **상관**에 따라 CF가 **개선/악화**될 수 있습니다. **실측**이 정답입니다.

**Q4. INCLUDE(비키 열)로 가볍게 커버링하면 좋을 텐데요?**  
A. Oracle **B-tree**에는 **INCLUDE**가 없습니다. **키 말미에 컬럼 추가**로 대체합니다(키 폭 증가에 주의).

**Q5. Batched BY ROWID는 강제할 수 있나요?**  
A. 옵티마이저가 조건/통계에 따라 선택합니다. 우리는 **커버링/정렬 일치/CF 개선**으로 **Batched가 붙기 쉬운 환경**을 마련하는 게 현실적입니다.

---

# 10) 체크리스트

- [ ] **핫 경로** 쿼리의 SELECT-LIST가 **인덱스만으로 충족**되게 설계(커버링).  
- [ ] **정렬 일치**(+DESC)와 **Stopkey/Keyset**으로 **앞부분만** 읽기.  
- [ ] PK 변경은 **최후의 수단**. 가능하면 **별도 인덱스**로 커버링.  
- [ ] 컬럼 추가 후 **CF 변화**를 **실측**(USER_INDEXES)하고, 필요 시 **CTAS 재적재**.  
- [ ] NL에서 랜덤 폭탄이면 **Hash Join/Pruning** 고려.  
- [ ] Invisible Index로 **그림자 검증 → 가시화** 프로세스 준수.  
- [ ] 통계/카디널리티/바인드 타입을 **정상화**(SARGable 유지).  
- [ ] DBMS_XPLAN `ALLSTATS LAST`로 **Buffers/Reads** 비교로 개선을 **숫자**로 입증.

---

## 결론

테이블 랜덤 액세스는 **“인덱스는 빨랐는데 마지막에 느려지는”** 전형적 병목입니다.  
해법의 순서는 언제나 **(1) 커버링으로 미방문, (2) 정렬 일치·Stopkey, (3) CF 개선**입니다.  
**PK 인덱스 변경은 예외적 선택**이며, 대부분의 경우 **별도 커버링 인덱스**가 가장 현실적이고 안전합니다.  
보너스로, 주기적인 **CF 모니터링 + CTAS 재적재** 전략을 갖추면 랜덤 I/O의 복귀를 장기적으로 차단할 수 있습니다.