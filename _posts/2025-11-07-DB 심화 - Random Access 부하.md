---
layout: post
title: DB 심화 - Random Access 부하
date: 2025-11-07 17:25:23 +0900
category: DB 심화
---
# 테이블 **Random Access** 부하 완전 정복  
— `TABLE ACCESS BY INDEX ROWID`가 왜 느려지는지, **db file sequential read**를 어떻게 줄이는지, 실전 튜닝 시나리오/스크립트까지

> 목표: 인덱스로 ROWID를 얻은 뒤 **Heap 테이블**을 “톡톡” 찌르는 **랜덤 I/O**가 커지면 왜 전체 성능이 무너지는지,  
> 이를 줄이는 **설계/쿼리/옵티마이저/스토리지** 단계별 해법을 **예제와 실행계획/통계**로 확인합니다. (Oracle 기준)

---

## 0. 실습 스키마 준비

```sql
-- 깨끗이
DROP TABLE ra_demo PURGE;

-- 주문 테이블: 100만 건 (랜덤 I/O가 체감되도록)
CREATE TABLE ra_demo (
  order_id    NUMBER        NOT NULL,
  cust_id     NUMBER        NOT NULL,
  order_dt    DATE          NOT NULL,
  amount      NUMBER(12,2)  NOT NULL,
  status      VARCHAR2(8)   NOT NULL,
  note        VARCHAR2(100),
  CONSTRAINT pk_ra_demo PRIMARY KEY (order_id)
);

BEGIN
  FOR i IN 1..1000000 LOOP
    INSERT INTO ra_demo
    VALUES (
      i,
      MOD(i, 100000) + 1,
      DATE '2024-01-01' + MOD(i, 365),
      ROUND(DBMS_RANDOM.VALUE(1, 100000), 2),
      CASE MOD(i,5) WHEN 0 THEN 'NEW' WHEN 1 THEN 'PAID'
                    WHEN 2 THEN 'SHIP' WHEN 3 THEN 'DONE' ELSE 'CANC' END,
      CASE WHEN MOD(i,97)=0 THEN 'gift' END
    );
  END LOOP;
  COMMIT;
END;
/

-- 인덱스: (cust_id, order_dt, order_id) → 고객/기간 조회
CREATE INDEX ix_ra_cust_dt ON ra_demo(cust_id, order_dt, order_id);

-- 통계
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'RA_DEMO', cascade => TRUE,
    method_opt => 'for all columns size skewonly');
END;
/

ALTER SESSION SET statistics_level = ALL; -- 수행통계 확인용
```

---

# 1. 개념: “**랜덤** vs **순차** I/O”

- **Table Random Access** = 인덱스 Leaf에서 얻은 **ROWID**로 테이블 **임의 블록**을 “툭툭” 찍는 것.  
  - 실행계획 오퍼레이터: `TABLE ACCESS BY INDEX ROWID [BATCHED]`  
  - 대기 이벤트: **`db file sequential read`** (싱글블록 읽기, 랜덤 성격)  
- 반대로 테이블이나 인덱스를 넓게 훑으면 **멀티블록 I/O**(**scattered read**)가 주가 되고 순차성이 커짐.  
- 랜덤 I/O가 많아질수록 **I/O 큐잉**이 증가, **RTT**가 누적되어 응답시간을 키운다.

> 기초 공식 감각  
> $$ \text{Total I/O Cost} \approx N_{\text{sequential}} \cdot t_s + N_{\text{random}} \cdot t_r $$
> 일반적으로 \( t_r \gg t_s \) (랜덤 접근의 지연이 큼).  
> 목표는 **\(N_{\text{random}}\)** 를 줄이거나, **랜덤을 순차화/배치화** 하는 것.

---

# 2. 증상과 지표: 무엇을 보면 랜덤 부하라고 확신할 수 있나?

- AWR/ASH/Statspack에서 **Top Events**가 `db file sequential read` 우세.  
- SQL 실행계획에 **`TABLE ACCESS BY INDEX ROWID`** 반복, `Buffers/Reads`가 높음.  
- 인덱스 스캔 대비 테이블 랜덤 블록 접근이 과다(`Clustering Factor` 나쁨).  
- Nested Loops 조인에서 **Inner 테이블** 랜덤 방문이 폭증.

**빠른 확인 템플릿**

```sql
-- 마지막 실행 SQL 수행통계
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST +PEEKED_BINDS +OUTLINE'));

-- 시스템 대기/통계 지표 샘플
SELECT name, value
FROM v$sysstat
WHERE name IN ('session logical reads','physical reads','physical reads cache')
ORDER BY name;

-- Top wait event 확인(AWR/ASH 이용 권장)
```

---

# 3. 데모 ①: 범위는 작지만 **테이블 랜덤**이 누적되는 케이스

```sql
-- 고객 12345의 최근 90일 주문 목록
SELECT /* BASELINE: RANGE + TABLE BY ROWID */
       order_id, order_dt, amount, status
FROM   ra_demo
WHERE  cust_id = 12345
AND    order_dt >= SYSDATE - 90
ORDER  BY order_dt, order_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```

**관찰 포인트**
- `INDEX RANGE SCAN IX_RA_CUST_DT` → 좋아 보이나,  
- 뒤이어 `TABLE ACCESS BY INDEX ROWID`에서 **Buffers/Reads**가 기대보다 많이 늘 수 있음.  
- 이유: 해당 고객의 행들이 **테이블 물리 순서**와 **인덱스 순서**가 잘 맞지 않으면(= **Clustering Factor 나쁨**)  
  인덱스 순서대로 ROWID를 찍을 때 **서로 먼 블록**을 계속 방문 → 랜덤 I/O 폭증.

---

# 4. 데모 ②: **Nested Loops**로 **랜덤 폭탄**

```sql
-- 고객 Master와 주문을 NL 조인 (많은 고객을 선택하는 상황 가정)
DROP TABLE ra_cust PURGE;
CREATE TABLE ra_cust AS
SELECT /* 10만 고객 */
       LEVEL AS cust_id, 'ACTIVE' AS cstate
FROM   dual CONNECT BY LEVEL <= 100000;

CREATE INDEX ix_ra_cust ON ra_cust(cust_id);

-- NL Join: 외부(ra_cust) 한 줄당 내부(ra_demo) 인덱스→ROWID random 액세스
SELECT /* NL 폭탄 가능 */
       c.cust_id, d.order_id, d.amount
FROM   ra_cust c
JOIN   ra_demo d
  ON   d.cust_id = c.cust_id
WHERE  c.cstate = 'ACTIVE'
AND    d.order_dt >= DATE '2024-09-01';
```

- 외부에서 **많은 건**을 던지면 내부 테이블에 대한 **랜덤 점프**가 누적 → `db file sequential read` 급증.  
- 선택도가 낮으면 NL 대신 **Hash Join + Partition Pruning/Full Scan**이 나은 경우가 많다.

---

# 5. 원인 별 해법 총정리

## 5.1 **테이블 방문 자체를 없애기** — 커버링/Index-Only/INDEX JOIN
- **커버링 인덱스**: SELECT-LIST 컬럼을 인덱스에 포함 → 테이블 미방문.  
- **INDEX JOIN**: 둘 이상의 인덱스에서 필요한 컬럼을 모아 테이블을 대체.

```sql
-- 커버링 인덱스(필요 컬럼을 인덱스에)
CREATE INDEX ix_ra_cover ON ra_demo(cust_id, order_dt, order_id, status, amount);

-- 이제 주문 목록이 인덱스만으로 (정렬도 맞추면 SORT 제거)
SELECT /* INDEX-ONLY */
       order_id, order_dt, amount, status
FROM   ra_demo
WHERE  cust_id = :cid
AND    order_dt >= :from_dt
ORDER  BY order_dt, order_id;

-- INDEX JOIN 예: (status), (order_dt,order_id,amount) 두 인덱스로 커버
SELECT /*+ INDEX_JOIN(d ix_ra_cover pk_ra_demo) */  -- 예시
       d.order_id, d.order_dt, d.amount, d.status
FROM   ra_demo d
WHERE  d.cust_id = :cid
ORDER  BY d.order_dt, d.order_id
FETCH FIRST 50 ROWS ONLY;
```

> 장점: **랜덤 테이블 I/O 0**.  
> 주의: 인덱스 폭/유지비 증가(쓰기부하), 선택적 적용.

---

## 5.2 **Random → 순차화** — **Clustering Factor** 개선(물리 재배치)

- **CF(Cluster Factor)**가 나쁠수록 인덱스 순서대로 읽을 때 테이블 블록 **재사용이 적다**.  
- 해법: **인덱스 키 순서**로 테이블을 **재적재(CTAS)** 후 인덱스 재생성.

```sql
-- 1) 인덱스 키 순서로 물리 정렬
CREATE TABLE ra_demo_sorted NOLOGGING AS
SELECT * FROM ra_demo ORDER BY cust_id, order_dt, order_id;

-- 2) 이름 스왑
ALTER TABLE ra_demo RENAME TO ra_demo_old;
ALTER TABLE ra_demo_sorted RENAME TO ra_demo;

-- 3) 인덱스 재생성 + 통계
DROP INDEX ix_ra_cust_dt;
CREATE INDEX ix_ra_cust_dt ON ra_demo(cust_id, order_dt, order_id);
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'RA_DEMO', cascade=>TRUE);
END;
/

-- 동일 쿼리 재실행 후 Buffers/Reads 비교: 랜덤 I/O가 확 줄어든다.
```

> 근본: 인덱스 순서와 **물리 저장 순서**를 맞춰 **순차성**을 높이는 것.

---

## 5.3 **Batched Rowid** — “조금 모았다가” 접근

- Oracle은 상황에 따라 `TABLE ACCESS BY INDEX ROWID **BATCHED**`를 사용,  
  **ROWID를 모아** I/O를 **줄 맞춰** 던지며 랜덤성을 완화.  
- 옵티마이저가 자동으로 선택(버전/패치/통계 의존). 힌트로 강제하기는 제한적.  
- Batched가 보인다면 **좋은 신호**: 같은 블록으로 묶어 읽는 비율이 올라감.

**관찰 예시(계획):**
```
INDEX RANGE SCAN IX_RA_CUST_DT
TABLE ACCESS BY INDEX ROWID BATCHED RA_DEMO
```

---

## 5.4 **조인 전략 전환** — NL → HASH, 혹은 **Bloom Filter** 활용

- 선택도가 낮아 **내부 테이블**을 많이 건드리면 **NL**은 랜덤 I/O 폭탄.  
- **Hash Join**으로 전환해 **대량 스캔 + 해시**로 풀면 랜덤 점프를 순차화.

```sql
-- 힌트로 HJ 유도
SELECT /*+ LEADING(c) USE_HASH(d) */
       c.cust_id, d.order_id, d.amount
FROM   ra_cust c
JOIN   ra_demo d
  ON   d.cust_id = c.cust_id
WHERE  c.cstate = 'ACTIVE'
AND    d.order_dt >= DATE '2024-09-01';
```

- 대용량 조인에서 **Partitioning + Pruning**, **Bloom Filter**가 있으면 더욱 강력.

---

## 5.5 **Keyset Paging / Stopkey** — “앞부분만 읽고 끝내기”

- OFFSET 페이징은 앞쪽을 **버리기 위해** 불필요하게 많이 읽는다(랜덤 I/O 누적).  
- **Keyset Paging**으로 **마지막 키**를 기억해 이어서 읽는다.

```sql
-- OFFSET (비추천, 앞부분 많은 랜덤/정렬 부하)
SELECT order_id, order_dt
FROM   ra_demo
WHERE  cust_id=:cid
ORDER  BY order_dt, order_id
OFFSET 100000 ROWS FETCH NEXT 50 ROWS ONLY;

-- Keyset (권장)
SELECT order_id, order_dt
FROM   ra_demo
WHERE  cust_id=:cid
  AND (order_dt > :last_dt
       OR (order_dt = :last_dt AND order_id > :last_id))
ORDER  BY order_dt, order_id
FETCH FIRST 50 ROWS ONLY;
```

> 결과: 인덱스 Leaf 체인 **앞부분만** 읽고 종료 → **랜덤 접근량 최소화**.

---

## 5.6 **Partitioning / Pruning** — “읽을 영역 자체를 쪼개기”

- Range 혹은 Hash Partition으로 **물리 데이터를 작게 분할**,  
  파티션 조건으로 **불필요한 구역 건드리지 않게**.  
- 파티션 로컬 인덱스와 함께 사용하면 **인덱스/테이블 랜덤 범위**가 더 작아진다.

```sql
-- (개념 예) 일자 Range 분할 + 로컬 인덱스
-- SELECT ... WHERE order_dt >= :from_dt AND order_dt < :to_dt
-- → 필요한 Partition만 스캔 → 랜덤 수 감소
```

---

## 5.7 **IOT/Cluster/Heap 대안** — 저장 구조 자체 바꾸기

- **IOT(Index-Organized Table)**: PK 순서로 **테이블 자체가 인덱스 구조**. PK 기반 조회는 **랜덤 없음**.  
  - 단, **넓은 로우/비PK 액세스**가 많은 경우 불리.  
- **Hash/Index Cluster**: 키 근접 저장으로 **랜덤 감소**(설계 난이도/제약 있음).  
- 일반 **Heap**은 랜덤 발생이 쉬움(특히 CF 나쁠 때).

---

## 5.8 **결과/함수 캐시** — “아예 읽지 않게”

- **SQL Result Cache / PL/SQL RESULT_CACHE** / **Materialized View** / **어플리케이션 캐시**  
- 동일 질의 반복이면 “읽지 않는 게 최고”다.  
- 단, 일관성과 무효화 전략을 명확히.

---

## 5.9 **프리패치/Array Fetch/배치** — 콜/왕복 최소화

- 드라이버에서 **Fetch Size**(e.g., `arraysize`)를 키워 **네트워크 왕복** 감소.  
- **Array DML**(배치 삽입/갱신)은 랜덤 I/O와는 별개지만 **DB Call** 회수를 줄여 전체 체감 성능 개선.

```python
-- Python oracledb 예: arraysize 상향
cur.arraysize = 1000
cur.execute("""
  SELECT order_id, order_dt, amount
  FROM   ra_demo
  WHERE  cust_id=:cid
  AND    order_dt >= :from_dt
  ORDER  BY order_dt, order_id
""", cid=12345, from_dt=datetime.date(2024,9,1))
rows = cur.fetchall()
```

---

# 6. 데모 ③: CTAS로 **CF 개선** 전/후 비교

```sql
-- BEFORE
SELECT /* BEFORE */ COUNT(*)
FROM   ra_demo
WHERE  cust_id=12345
AND    order_dt >= DATE '2024-09-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));

-- CF 개선(§5.2) 적용 후 동일 질의 다시 실행 → Buffers/Reads 비교
```

**예상 결과**: `TABLE ACCESS BY INDEX ROWID` 단계의 블록 접근이 큰 폭으로 줄어든다.

---

# 7. 데모 ④: 조인 전략 전환 효과

```sql
-- NL (랜덤 폭탄 가능)
SELECT /* NL */ c.cust_id, d.order_id
FROM   ra_cust c
JOIN   ra_demo d ON d.cust_id = c.cust_id
WHERE  c.cstate='ACTIVE' AND d.order_dt>=DATE '2024-09-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));

-- HJ (순차화 유도)
SELECT /*+ USE_HASH(d) */ c.cust_id, d.order_id
FROM   ra_cust c
JOIN   ra_demo d ON d.cust_id = c.cust_id
WHERE  c.cstate='ACTIVE' AND d.order_dt>=DATE '2024-09-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```

**관찰**: `db file sequential read` 비중 감소, `buffers/reads` 패턴 변화.

---

# 8. 운영 현장에서의 **원인→해법** 매핑

| 증상/원인 | 설명 | 해법(우선순) |
|---|---|---|
| CF 나쁨 | 인덱스 순서 ↔ 테이블 물리 순서 불일치 | **CTAS 재적재** → 인덱스 재생성, 파티셔닝 |
| SELECT-LIST 많음 | 테이블까지 가야 함 | **커버링 인덱스**, **INDEX JOIN** |
| NL 남발 | 내부 테이블 랜덤 폭탄 | **Hash Join**, 조인 순서/드라이빙 적정화 |
| Massive OFFSET | 앞부분 버리느라 과다 랜덤/정렬 | **Keyset Paging**, Stopkey |
| 조건 약함 + 정렬 필요 | IFFS는 순서 없음 | **INDEX FULL SCAN (ASC/DESC)** 설계 |
| 저카디널리티 다중 필터 | B-tree Range+ROWID 비효율 | **Bitmap 인덱스 + 결합**(DW), `INDEX_COMBINE` |
| 반복 동일 질의 | 캐시 미활용 | **SQL/PLSQL Result Cache**, MV/앱 캐시 |
| 드라이버 Fetch 빈번 | 왕복 과다 | **arraysize 상향**, Array DML |

---

# 9. RAC/스토리지/버퍼캐시 관점 한 줄 요약

- **RAC**: 같은 블록을 여러 인스턴스가 번갈아 랜덤으로 잡으면 **캐시 퓨전** 교통량 증가. 파티셔닝/서비스 로컬리티로 **데이터/워크로드 분리**.  
- **스토리지**: NVMe/Flash에서도 **랜덤은 여전히 비싸다**(큐잉/락/왕복). IOPS/큐뎁스를 고려.  
- **버퍼 캐시**: 자주 쓰는 블록이 **핫 블록**이 되면 래치/뮤텍스 경합도. **배치/순차화**가 경합 완화에 도움.

---

# 10. 최종 체크리스트

- [ ] 실행계획에 `TABLE ACCESS BY INDEX ROWID`가 과도한가? `Buffers/Reads` 수치 확인  
- [ ] 커버링 인덱스/INDEX JOIN으로 **테이블 미방문**화 가능한가?  
- [ ] **CF** 개선(CTAS 재적재) 시나리오 검토했는가?  
- [ ] **Keyset Paging**/Stopkey로 앞부분만 읽도록 했는가?  
- [ ] NL이 랜덤 폭탄이라면 **Hash Join**/파티셔닝/Pruning으로 순차화했는가?  
- [ ] Bitmap 결합/INDEX_COMBINE이 더 유리한 조건인가(DW)?  
- [ ] 드라이버 **arraysize**/배치/바인드 타입 최적화는 되어 있는가?  
- [ ] AWR/ASH에서 `db file sequential read` 중심의 병목이 실측으로 줄었는가?

---

## 마무리

랜덤 I/O는 **빠르게 여러 곳을** 찌르는 것처럼 보이지만, 실제로는 **왕복 지연**과 **큐잉**이 누적되어 전체 응답시간을 갉아먹습니다.  
핵심은 세 가지:

1) **테이블을 아예 안 가거나**(커버링/INDEX JOIN/캐시)  
2) **랜덤을 순차로 바꾸거나**(CF 개선/Hash Join/Pruning/Batched)  
3) **아주 조금만 읽고 끝내기**(Keyset/Stopkey)