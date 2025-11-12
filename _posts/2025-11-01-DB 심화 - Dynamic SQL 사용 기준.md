---
layout: post
title: DB 심화 - Dynamic SQL 사용 기준
date: 2025-11-01 14:25:23 +0900
category: DB 심화
---
# Dynamic SQL 사용 기준

> **핵심 요약**
> - **Dynamic SQL은 “구조”가 바뀌어야 할 때만** 쓴다. **“값” 때문에 동적으로 만들지 말고** 무조건 **바인드 변수**를 쓴다.  
> - “선택적 검색 조건(옵셔널 파라미터)” 때문에 **정적 SQL**이 복잡해질 때, **성능·가독성**·**안전성**을 모두 만족하는 대안을 고른다.  
> - 대안은 상황별로 다르다: **`OR :p IS NULL` 패턴**, **`UNION ALL` 분기**, **동적 WHERE 조립 + 바인드**, **배열 바인드(IN-list)**, **GTT/임시 테이블 조인**, **JSON 파라미터** 등.  
> - **성능 비교**는 “**선택도(카디널리티)**, **인덱스 사용가능성(SARGability)**, **플랜 안정성(ACS/히스토그램)**” 관점으로 한다.

---

## 0. 용어와 범위
- **정적 SQL(Static)**: 텍스트가 코드에 고정. 컴파일 시 검증 가능.  
- **동적 SQL(Dynamic)**: 텍스트를 런타임에 구성(`EXECUTE IMMEDIATE`, 애플리케이션 문자열 템플릿).  
- **선택적 검색 조건(Optional Predicate)**: 파라미터가 **NULL이면 필터 미적용**, 값이 있으면 **필터 적용**하는 형태. 예) `region, date_from, date_to, status` 등.

> **문제의 본질**: *Static vs Dynamic*이 아니라 **바인드 변수 사용 여부**다. 값은 항상 **바인드**로 분리해야 커서 공유·보안이 확보된다.

---

## 1. Dynamic SQL **사용에 관한 기본 원칙** (Rule of Thumb)

### 원칙 1 — “값 때문에 동적화하지 않는다”
- 값은 **바인드 변수**로.  
- 문자열 결합으로 값을 텍스트에 끼워 넣지 않는다(성능·보안 모두 악화).

### 원칙 2 — “구조가 바뀌면 동적화한다”
- 테이블/컬럼/조인/정렬 컬럼/힌트/파티션 명 등 **오브젝트·구조**가 런타임에 바뀔 때만 **동적 SQL** 고려.  
- 이때도 오브젝트명은 **화이트리스트**를 반드시 통과시킨다.

### 원칙 3 — “SARGability(인덱스 사용 가능성)를 지킨다”
- **컬럼 변형 함수**(예: `TRUNC(col)`, `NVL(col, ...)`), **비상수 비교**는 인덱스를 막는다.  
- 가능하면 **`col >= :d1 AND col < :d2`** 같은 **범위식**으로 전개.

### 원칙 4 — “플랜 안정성 우선”
- 스큐 컬럼은 **히스토그램+ACS**로 카드 추정 정확화.  
- 자주 쓰는 핵심 SQL은 **SPM(SQL Plan Baseline)** 로 좋은 플랜을 봉인.

### 원칙 5 — “가독성과 테스트 가능성”
- 조건이 많을수록 **분기(UNION ALL / 동적 WHERE)** 로 **명시적** 설계를 고려.  
- **단위 테스트·바인드 값 케이스**를 쉽게 만들 수 있어야 한다.

---

## 2. 원칙이 잘 지켜지지 않는 **대표 이유**: **선택적 검색 조건**

### 2.1 상황 묘사
- 사용자 입력 파라미터가 **있을 수도, 없을 수도** 있다.  
- “없으면 전체 조회, 있으면 필터”를 **한 개 SQL**로 처리하고 싶어진다.  
- 그래서 다음 패턴이 흔해진다:

```sql
-- 흔하지만 성능이 애매해지기 쉬운 패턴
WHERE (region = :r OR :r IS NULL)
  AND (order_dt >= :d1 OR :d1 IS NULL)
  AND (order_dt <  :d2 OR :d2 IS NULL)
  AND (status = :s OR :s IS NULL)
```

### 2.2 왜 문제가 될 수 있나?
- **OR**이 많아지면 옵티마이저가 **카디널리티**를 보수적으로 추정 →  
  **여러 조건에서 인덱스가 무력화**되거나, **잘못된 조인 순서/방법**을 택할 수 있다.  
- 특히 `:p IS NULL` 분기와 결합되면 **액세스 조건을 필터 조건**으로 바꿔버리는 일이 잦고,  
  **`OR`-expansion** 최적화가 항상 작동하지도 않는다(버전·상황 의존).

---

## 3. 선택적 검색 조건에 대한 **현실적 대안** (설계 패턴)

아래 패턴은 **서로 대체** 관계가 아니다. **데이터 분포·쿼리 특성**에 맞게 고른다.

### 패턴 A — **OR + IS NULL** (간단하지만 SARGability 이슈)
```sql
WHERE (region = :r OR :r IS NULL)
  AND (order_dt >= :d1 OR :d1 IS NULL)
  AND (order_dt <  :d2 OR :d2 IS NULL)
```
- **장점**: 코드 단순, 동적화 불필요(정적 SQL 유지)  
- **단점**: 인덱스 사용이 **불안정**, OR-Expansion이 항상 되지 않음 → 대량 데이터에서 위험  
- **추천 용도**: **소량 테이블**, **인덱스 없어도 감당 가능한 규모**, 빠른 PoC

### 패턴 B — **`UNION ALL` 분기** (명시적 플랜 분리)
```sql
-- region 유무, 기간 유무 등 “경우의 수”가 적을 때 효과적
SELECT /* base */ ... FROM t
WHERE :r IS NULL AND :d1 IS NULL AND :d2 IS NULL
UNION ALL
SELECT /* r */ ... FROM t
WHERE :r IS NOT NULL AND region = :r
  AND :d1 IS NULL AND :d2 IS NULL
UNION ALL
SELECT /* r+d1d2 */ ... FROM t
WHERE :r IS NOT NULL AND region = :r
  AND :d1 IS NOT NULL AND :d2 IS NOT NULL
  AND order_dt >= :d1 AND order_dt < :d2
-- (필요한 조합만 추가; 모든 조합을 나열할 필요는 없음)
```
- **장점**: 각 분기에서 **인덱스 액세스**를 명확히 유도(플랜 안정)  
- **단점**: 텍스트 길어짐, 경우의 수가 많으면 관리 복잡  
- **추천 용도**: **핵심 조회**, **선택도 차가 큰 조건** 존재 시

### 패턴 C — **동적 WHERE 조립 + 바인드(권장)**  
(구조는 고정, WHERE만 동적)
```plsql
DECLARE
  v_sql CLOB := 'SELECT /* dyn-where */ cols FROM t WHERE 1=1';
  v_binds SYS.ODCIVARCHAR2LIST := SYS.ODCIVARCHAR2LIST(); -- 설명용
  v_rgn  VARCHAR2(10) := :r;
  v_d1   DATE := :d1;
  v_d2   DATE := :d2;
BEGIN
  IF v_rgn IS NOT NULL THEN v_sql := v_sql||' AND region = :b1'; END IF;
  IF v_d1  IS NOT NULL THEN v_sql := v_sql||' AND order_dt >= :b2'; END IF;
  IF v_d2  IS NOT NULL THEN v_sql := v_sql||' AND order_dt <  :b3'; END IF;

  -- USING 절 바인드는 순서대로 전달 (NULL 제외 규칙을 함께 유지)
  CASE
    WHEN v_rgn IS NOT NULL AND v_d1 IS NOT NULL AND v_d2 IS NOT NULL THEN
      EXECUTE IMMEDIATE v_sql USING v_rgn, v_d1, v_d2;
    WHEN v_rgn IS NOT NULL AND v_d1 IS NOT NULL THEN
      EXECUTE IMMEDIATE v_sql USING v_rgn, v_d1;
    WHEN v_rgn IS NOT NULL AND v_d2 IS NOT NULL THEN
      EXECUTE IMMEDIATE v_sql USING v_rgn, v_d2;
    WHEN v_rgn IS NOT NULL THEN
      EXECUTE IMMEDIATE v_sql USING v_rgn;
    WHEN v_d1 IS NOT NULL AND v_d2 IS NOT NULL THEN
      EXECUTE IMMEDIATE v_sql USING v_d1, v_d2;
    WHEN v_d1 IS NOT NULL THEN
      EXECUTE IMMEDIATE v_sql USING v_d1;
    WHEN v_d2 IS NOT NULL THEN
      EXECUTE IMMEDIATE v_sql USING v_d2;
    ELSE
      EXECUTE IMMEDIATE v_sql;
  END CASE;
END;
/
```
- **장점**: 불필요한 조건을 **아예 제거** → **SARGability 보장**  
- **단점**: 동적 SQL 관리 필요, 테스트 케이스 늘어남  
- **추천 용도**: **대형 테이블**·**핵심 조회**, **선택도/스큐가 큰 컬럼** 포함

> **주의**: 동적화하더라도 **값은 무조건 바인드**. 오브젝트명/ORDER BY는 화이트리스트.

### 패턴 D — **배열 바인드(IN-list)**  
(멀티 선택 옵션: 다중 지역·상태)
```plsql
DECLARE
  TYPE t_vc IS TABLE OF VARCHAR2(10);
  v_regions t_vc := t_vc('APAC','EMEA'); -- UI에서 다중 선택
BEGIN
  SELECT /* in-list */ ...
  FROM   t
  WHERE  region IN (SELECT COLUMN_VALUE FROM TABLE(v_regions))
  -- 다른 조건은 C 패턴처럼 동적 WHERE로 결합
  ;
END;
/
```
- **장점**: IN 리스트를 안전·유연하게 처리(커서 공유 유지)  
- **단점**: 배열 크기가 **매우 클** 경우 성능 고려(수십~수백 개 수준 권장)  
- **추천 용도**: UI 멀티 선택, 권한 기반 영역필터

### 패턴 E — **GTT/임시 테이블 조인**  
(필터 값이 **아주 많을 때**)
```sql
-- (1) 세션 전용 임시 테이블에 필터 키 적재
INSERT INTO filter_keys(key) VALUES (:k1); -- 벌크로 적재
...

-- (2) 메인 쿼리에서 조인
SELECT /* gtt */ t.*
FROM   t JOIN filter_keys f ON t.key = f.key
WHERE  /* 다른 조건은 동적 WHERE or 정적 분기 */
;
```
- **장점**: 대량 키 필터 시 효율적, 통계/인덱스 전략 가능  
- **단점**: 임시 테이블 관리(세션·정리), 트랜잭션 흐름 복잡  
- **추천**: 대규모 IN-list, 보고/배치

### 패턴 F — **JSON 파라미터 + JSON_EXISTS**  
(필수·선택적 파라미터가 많은 API)
```sql
-- 입력 JSON 예: {"region":"APAC","date_from":"2025-10-01","date_to":null}
SELECT /* json */ ...
FROM   t
WHERE  (NOT JSON_EXISTS(:j, '$.region') OR region = JSON_VALUE(:j, '$.region'))
  AND  (NOT JSON_EXISTS(:j, '$.date_from') OR order_dt >= TO_DATE(JSON_VALUE(:j,'$.date_from'),'YYYY-MM-DD'))
  AND  (NOT JSON_EXISTS(:j, '$.date_to')   OR order_dt <  TO_DATE(JSON_VALUE(:j,'$.date_to'),'YYYY-MM-DD'))
;
```
- **장점**: 파라미터 개수가 많을 때 인터페이스 단순화  
- **단점**: JSON 함수가 **인덱스 사용을 어렵게** 만들 수 있음(함수 기반 인덱스·가상컬럼·저장 방식 고려)  
- **추천**: **유연한 API**가 필요하고, 물리 모델 최적화가 가능한 경우

---

## 4. **언어별 안전한 동적 WHERE 조립** 예제

### 4.1 Java (JDBC)
```java
StringBuilder sql = new StringBuilder(
  "SELECT /* dyn-where */ empno, ename, sal, deptno, hiredate FROM emp WHERE 1=1");
List<Object> binds = new ArrayList<>();

if (region != null) { sql.append(" AND region = ?"); binds.add(region); }
if (from   != null) { sql.append(" AND hiredate >= ?"); binds.add(Date.valueOf(from)); }
if (to     != null) { sql.append(" AND hiredate <  ?"); binds.add(Date.valueOf(to)); }
if (status != null) { sql.append(" AND status = ?"); binds.add(status); }

try (PreparedStatement ps = conn.prepareStatement(sql.toString())) {
  for (int i=0;i<binds.size();i++) ps.setObject(i+1, binds.get(i));
  try (ResultSet rs = ps.executeQuery()) { /* ... */ }
}
```
- **값은 전부 바인드**, 동적으로 **WHERE 문자열만** 조립.

### 4.2 Python (oracledb)
```python
sql = ["SELECT /* dyn-where */ empno, ename, sal FROM emp WHERE 1=1"]
params = {}
if region is not None: sql.append(" AND region = :r"); params["r"] = region
if d1     is not None: sql.append(" AND hiredate >= :d1"); params["d1"] = d1
if d2     is not None: sql.append(" AND hiredate <  :d2"); params["d2"] = d2
q = " ".join(sql)
cur.execute(q, params)
for row in cur: ...
```

---

## 5. **성능 비교 실험**: 데이터·인덱스·케이스 설계

### 5.1 테스트 데이터
```sql
DROP TABLE t_demo PURGE;
CREATE TABLE t_demo AS
SELECT level id,
       CASE MOD(level,10)
         WHEN 0 THEN 'APAC' WHEN 1 THEN 'EMEA' WHEN 2 THEN 'AMER'
         ELSE 'OTHER' END      AS region,
       TRUNC(DATE '2025-10-01' + MOD(level, 90)) AS order_dt,
       CASE WHEN MOD(level,20)=0 THEN 'CANCEL' ELSE 'OK' END status,
       ROUND(DBMS_RANDOM.VALUE(10,1000),2) amt
FROM dual CONNECT BY level <= 2e6;

CREATE INDEX ix_demo_region_dt ON t_demo(region, order_dt);
CREATE INDEX ix_demo_status    ON t_demo(status);

BEGIN DBMS_STATS.GATHER_TABLE_STATS(USER,'T_DEMO',cascade=>TRUE,
  method_opt=>'FOR COLUMNS SIZE 75 region SIZE 75 status SIZE 1 order_dt'); END;
/
```

### 5.2 비교할 기법
1) **A패턴**: `(col=:p OR :p IS NULL)`  
2) **B패턴**: **`UNION ALL` 분기**  
3) **C패턴**: **동적 WHERE 조립 + 바인드**  
4) **D패턴**: **배열 바인드(IN-list)**  

### 5.3 측정 방법(권장)
- **`SET autotrace traceonly statistics`** 또는 **`DBMS_XPLAN.DISPLAY_CURSOR('sql_id',NULL,'ALLSTATS LAST +PEEKED_BINDS')`**  
- 다음 지표를 비교:
  - **Consistent Gets(논리 읽기)**, **Physical Reads(물리 읽기)**  
  - **Elapsed(Time)**, **CPU Time**  
  - **Plan Hash, Access/Filter Predicates**, **Cardinality**  
- **파라미터 케이스**:
  - {region=APAC(고빈도), region=AMER(저빈도), region=NULL}  
  - {기간 없음, 최근 7일, 90일}  
  - {status=NULL, 'OK'}

> **기대치**  
> - **B/C**(UNION ALL 또는 동적 WHERE)는 **인덱스 액세스**가 깨끗하게 잡혀 **논리 읽기/시간**이 안정적으로 낮다.  
> - **A**(OR+IS NULL)는 케이스에 따라 인덱스가 덜 쓰이거나, **OR-Expansion 실패** 시 풀스캔 경향.

---

## 6. 각 기법의 **구체 예제**와 **장단점 요약**

### 6.1 A — OR + IS NULL (정적, 가장 단순)
```sql
SELECT /* A */ SUM(amt)
FROM   t_demo
WHERE  (region = :r OR :r IS NULL)
  AND  (order_dt >= :d1 OR :d1 IS NULL)
  AND  (order_dt <  :d2 OR :d2 IS NULL)
  AND  (status = :s OR :s IS NULL);
```
- **장점**: 한 문장, 구현 간단, 커서 공유 용이  
- **단점**: **SARGability 불안**(인덱스 활용 저하), 카디널리티 추정 난해  
- **Tip**: 소량 테이블/보조 조회에만

### 6.2 B — UNION ALL 분기 (정적, 명시적 플랜)
```sql
-- region + 기간이 모두 있을 때
SELECT /* B1 */ SUM(amt) FROM t_demo
WHERE region = :r AND order_dt >= :d1 AND order_dt < :d2
UNION ALL
-- region만 있을 때
SELECT /* B2 */ SUM(amt) FROM t_demo
WHERE region = :r AND :d1 IS NULL AND :d2 IS NULL
UNION ALL
-- 아무 것도 없을 때(필요시)
SELECT /* B3 */ SUM(amt) FROM t_demo
WHERE :r IS NULL AND :d1 IS NULL AND :d2 IS NULL;
```
- **장점**: 인덱스 경로를 **강력히 유도**, 플랜 안정성↑  
- **단점**: 경우의 수 많으면 SQL 길어짐  
- **Tip**: **핵심 API**·실시간 OLTP 핵심 질의

### 6.3 C — 동적 WHERE 조립 + 바인드 (동적, 권장)
```plsql
DECLARE
  v_sql CLOB := 'SELECT /* C */ SUM(amt) FROM t_demo WHERE 1=1';
  v_binds NUMBER := 0;
BEGIN
  IF :r  IS NOT NULL THEN v_sql := v_sql||' AND region   = :b'||TO_CHAR(v_binds+1); v_binds:=v_binds+1; END IF;
  IF :d1 IS NOT NULL THEN v_sql := v_sql||' AND order_dt >= :b'||TO_CHAR(v_binds+1); v_binds:=v_binds+1; END IF;
  IF :d2 IS NOT NULL THEN v_sql := v_sql||' AND order_dt <  :b'||TO_CHAR(v_binds+1); v_binds:=v_binds+1; END IF;
  IF :s  IS NOT NULL THEN v_sql := v_sql||' AND status   = :b'||TO_CHAR(v_binds+1); v_binds:=v_binds+1; END IF;

  -- USING 전달은 조건 존재 순서대로
  CASE v_binds
    WHEN 4 THEN EXECUTE IMMEDIATE v_sql USING :r,:d1,:d2,:s;
    WHEN 3 THEN
      IF :r IS NULL THEN EXECUTE IMMEDIATE v_sql USING :d1,:d2,:s;
      ELSIF :d1 IS NULL THEN EXECUTE IMMEDIATE v_sql USING :r,:d2,:s;
      ELSIF :d2 IS NULL THEN EXECUTE IMMEDIATE v_sql USING :r,:d1,:s;
      ELSE EXECUTE IMMEDIATE v_sql USING :r,:d1,:d2; END IF;
    WHEN 2 THEN ... -- (생략 없이 실제 구현 시 케이스 완비)
    WHEN 1 THEN ...
    ELSE EXECUTE IMMEDIATE v_sql;
  END CASE;
END;
/
```
- **장점**: 필요 조건만 남겨 **인덱스 경로 극대화**, OR/함수 회피  
- **단점**: 동적 SQL 관리 필요, 단위테스트 케이스 증가  
- **Tip**: **대형 테이블**·**선택도 큰 컬럼** 포함 시 최우선 고려

### 6.4 D — 배열 바인드(IN-list)
```plsql
DECLARE
  TYPE t_vc IS TABLE OF VARCHAR2(10);
  v_rgns t_vc := t_vc('APAC','EMEA','AMER');
  v_sum  NUMBER;
BEGIN
  SELECT /* D */ SUM(amt) INTO v_sum
  FROM   t_demo
  WHERE  region IN (SELECT COLUMN_VALUE FROM TABLE(v_rgns))
    AND  order_dt >= :d1
    AND  order_dt <  :d2;
END;
/
```
- **장점**: 멀티 선택 안전 처리, 커서 공유 유지  
- **단점**: 리스트 매우 크면 비효율 → GTT 고려  
- **Tip**: 다중 선택 필터(권역, 카테고리)

---

## 7. **기법별 성능 비교 요약표**

| 기법 | 인덱스 사용성 | 플랜 안정성 | 코드 복잡도 | 장점 | 주의 |
|---|---|---|---|---|---|
| A OR+IS NULL | △ (상황 의존) | △ | ◎ | 가장 간단 | 대용량/스큐에서 위험 |
| B UNION ALL | ◎ | ◎ | △ | 명시적 인덱스 경로 | 분기 수 많으면 관리 난이도 |
| C 동적 WHERE+바인드 | ◎ | ◎ | △ | 필요조건만 적용 | 동적 SQL 관리/테스트 |
| D 배열 바인드(IN) | ◎ | ○ | ○ | 멀티 선택 깔끔 | 매우 큰 IN은 GTT 권장 |
| E GTT/임시 테이블 | ◎ | ◎ | △ | 대량 키 필터 효율 | 라이프사이클 관리 |
| F JSON 파라미터 | △ | △ | ○ | 인터페이스 간결 | 함수 기반 인덱스 설계 필요 |

> **권장 기본 순서**: **C**(동적 WHERE) → **B**(UNION ALL) → **D**(배열 바인드) → (대량) **E**(GTT).  
> 소규모/단순은 **A**도 가능하나, 핵심 트랜잭션에는 지양.

---

## 8. 실전 팁 — 히스토그램·ACS·SPM로 **플랜 흔들림 방지**
- 스큐 컬럼(`region`, `status`)에 **히스토그램** → 카드 정확화.  
- **ACS(Adaptive Cursor Sharing)** 로 값 구간별 Child 플랜 분기.  
- 핵심 SQL은 **SPM**으로 좋은 플랜 고정(릴리즈·통계 변화에 안전).

---

## 9. 보안·운영 체크리스트
- [ ] 동적 SQL이라도 **값은 100% 바인드** (`EXECUTE IMMEDIATE ... USING`)  
- [ ] 오브젝트/컬럼/ORDER BY는 **화이트리스트**에서만 문자열 삽입  
- [ ] NLS/세션 파라미터 표준화(Child 폭증 방지)  
- [ ] 드라이버 **Statement Cache** + 서버 **SESSION_CACHED_CURSORS**  
- [ ] 배포/통계는 저부하 시간, 대규모 변경은 분산/순차  
- [ ] AWR/ASH로 파싱·뮤텍스 대기 모니터링

---

## 10. 미니 벤치 워크북 (재현 스크립트)

### 10.1 비교 실행 + XPLAN
```sql
-- 케이스 1: region='APAC', 최근 7일
VAR r VARCHAR2(10); VAR d1 DATE; VAR d2 DATE;
EXEC :r :='APAC'; EXEC :d1 := SYSDATE-7; EXEC :d2 := SYSDATE;

-- A
SELECT /* A */ COUNT(*) FROM t_demo
WHERE (region = :r OR :r IS NULL)
  AND (order_dt >= :d1 OR :d1 IS NULL)
  AND (order_dt <  :d2 OR :d2 IS NULL);

-- B
SELECT /* B */ COUNT(*) FROM (
  SELECT /* B1 */ 1 x FROM t_demo
   WHERE region = :r AND order_dt >= :d1 AND order_dt < :d2
  UNION ALL
  SELECT /* B2 */ 1 FROM t_demo
   WHERE :r IS NULL AND :d1 IS NULL AND :d2 IS NULL
) s;

-- C (동적 WHERE는 프로시저로 감싸 호출한다고 가정)

-- 실행 후
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PEEKED_BINDS +PREDICATE +NOTE'));
```

> **관찰 포인트**: Access Predicates(인덱스 조건) vs Filter Predicates 비중, Consistent Gets, Elapsed Time.

---

## 11. 결론 — **Dynamic SQL 사용 기준 요약**
1) **값 때문에 동적화 금지**, 값은 **항상 바인드**.  
2) **구조(오브젝트/정렬/조인)** 가 바뀌면 **동적 SQL** + 화이트리스트.  
3) 선택적 조건은 **동적 WHERE**(C) 또는 **UNION ALL 분기**(B)로 **SARGability** 확보.  
4) 멀티 선택은 **배열 바인드**(D), 대량 키는 **GTT**(E).  
5) 스큐 컬럼엔 **히스토그램+ACS**, 핵심 플랜은 **SPM**.  
6) 드라이버/서버 **커서 캐시**로 파싱 오버헤드 최소화.

> **한 줄 요약**  
> *Dynamic vs Static*의 문제는 아니다. **SARGability와 바인드**를 지키는 설계를 고르고,  
> 선택적 조건은 **동적 WHERE/UNION ALL/배열 바인드/GTT**로 **명료하게** 풀어라. 그러면 **빠르고 안정적**이다.