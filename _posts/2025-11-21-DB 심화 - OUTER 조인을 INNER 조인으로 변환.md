---
layout: post
title: DB 심화 - OUTER 조인을 INNER 조인으로 변환
date: 2025-11-21 16:25:23 +0900
category: DB 심화
---
# OUTER 조인을 INNER 조인으로 변환 — 원리, 안전 조건, 실전 튜닝 가이드(Oracle 중심)

> **핵심 요약**
> - **LEFT OUTER JOIN**은 **왼쪽(보존측)** 행을 모두 유지하고, 오른쪽에 매칭이 없으면 **NULL 확장**합니다.
> - **INNER JOIN**은 양쪽 모두 **매칭되는 행만** 반환합니다.
> - 다음 **조건** 중 하나라도 만족하면 LEFT OUTER JOIN을 **의미 보존**하며 **INNER JOIN**으로 바꿀 수 있습니다.
>   1) **NULL-배제(null-rejecting) 프레디킷**이 `WHERE` 절에 **오른쪽 컬럼**에 대해 존재 (예: `r.col = ...`, `r.col > ...`, `r.col IN (...)`, `r.col IS NOT NULL`).
>   2) **참조무결성**(FK **ENABLE VALIDATE**) + **FK가 NOT NULL** + 오른쪽이 **PK/Unique** → **항상 매칭 보장**.
>   3) 집계/필터가 **NULL-확장 행을 제거**함이 **논리적으로 확실**(예: `COUNT(r.col) > 0` 로 존재성만 보는 케이스).
>
> - 변환 이득: **조인 순서 자유도 증가**, **프레디킷 푸시다운/트랜지티브 이행** 활성화, **I/O 감소**.
> - 주의: 외부조인 의미( **NULL 보존** )가 필요한 경우에는 변환하면 **결과가 달라질 수 있음**.

---

## 개념 정리 — NULL 보존과 NULL-배제 프레디킷

- **LEFT OUTER JOIN**: 왼쪽 테이블 `L`의 각 행은 결과에 **항상 1회 이상** 나타납니다. 오른쪽 `R`에 매칭이 없으면 **`R.*`가 모두 NULL**로 채워집니다.
- **NULL-배제(null-rejecting) 프레디킷**: 피연산자에 **NULL이 끼면 거짓**이 되는 조건.
  - 예: `r.c = 10`, `r.c > 0`, `r.c IN (...)`, `r.c LIKE 'A%'`, `r.c BETWEEN ...`, `r.c IS NOT NULL`
  - 비예: `r.c IS NULL`(NULL 허용), `NVL(r.c, 'X') = 'X'`(NULL을 허용하도록 우회 가능)

**직관 수식**
$$
\text{LEFT JOIN}(L,R) \ \xrightarrow{\ \text{WHERE에 null-rejecting}(R)\ }\ \text{INNER JOIN}(L,R)
$$

---

## 실습 스키마 (간단)

```sql
CREATE TABLE d_customer (
  cust_id  NUMBER PRIMARY KEY,
  region   VARCHAR2(8) NOT NULL,
  tier     VARCHAR2(8) NOT NULL
);

CREATE TABLE d_product (
  prod_id  NUMBER PRIMARY KEY,
  category VARCHAR2(16) NOT NULL,
  brand    VARCHAR2(16) NOT NULL
);

CREATE TABLE f_sales (
  sales_id NUMBER PRIMARY KEY,
  cust_id  NUMBER NOT NULL,
  prod_id  NUMBER NOT NULL,
  sales_dt DATE   NOT NULL,
  qty      NUMBER NOT NULL,
  amount   NUMBER(12,2) NOT NULL,
  CONSTRAINT fk_sales_customer FOREIGN KEY (cust_id) REFERENCES d_customer(cust_id),
  CONSTRAINT fk_sales_product  FOREIGN KEY (prod_id)  REFERENCES d_product(prod_id)
);

-- 인덱스 예시
CREATE INDEX ix_sales_cust ON f_sales(cust_id);
CREATE INDEX ix_sales_prod ON f_sales(prod_id);

INSERT INTO d_customer VALUES (1,'KOR','VIP');
INSERT INTO d_customer VALUES (2,'USA','STD');
INSERT INTO d_product  VALUES (10,'ELEC','B0');
INSERT INTO d_product  VALUES (20,'HOME','B1');

INSERT INTO f_sales VALUES (101,1,10,DATE '2025-02-01',1,10000);
INSERT INTO f_sales VALUES (102,1,20,DATE '2025-02-03',2,15000);
INSERT INTO f_sales VALUES (103,2,10,DATE '2025-02-10',1, 9000);
COMMIT;
```

---

## 규칙 #1 — WHERE절의 **NULL-배제 프레디킷**이 있으면 INNER와 동치

### 가장 흔한 패턴

```sql
-- (원형) LEFT OUTER JOIN + 오른쪽 조건을 WHERE에 둠 → 실은 INNER 의미
SELECT c.cust_id, p.brand
FROM   d_customer c
LEFT  JOIN d_product p
       ON p.prod_id IN (10,20)           -- 조인키는 매칭 조건
WHERE  p.category = 'ELEC';              -- ← NULL-배제(오른쪽 컬럼 조건)

-- (동치) INNER JOIN
SELECT c.cust_id, p.brand
FROM   d_customer c
JOIN   d_product p
  ON   p.prod_id IN (10,20)
WHERE  p.category = 'ELEC';
```
- `LEFT` 결과에서 **매칭 실패한 행은 `p.*`가 NULL** → `p.category='ELEC'`에서 **탈락** → 결과적으로 **INNER와 동일**.

### IS NOT NULL / 비교 연산자

```sql
-- (원형) 매칭 없는 행 제거 목적 (NULL 제거)
SELECT c.cust_id
FROM   d_customer c
LEFT  JOIN f_sales s ON s.cust_id = c.cust_id
WHERE  s.sales_id IS NOT NULL;   -- ← NULL-배제

-- (동치) INNER
SELECT c.cust_id
FROM   d_customer c
JOIN   f_sales s ON s.cust_id = c.cust_id;
```

### Oracle 구문(옛 스타일 `(+ )`)도 동일

```sql
-- (원형) p가 외부측
SELECT c.cust_id, p.brand
FROM   d_customer c, d_product p
WHERE  p.prod_id(+) = 10
AND    p.category = 'ELEC';    -- ← 이 WHERE가 NULL을 배제 → 사실 INNER

-- (동치)
SELECT c.cust_id, p.brand
FROM   d_customer c
JOIN   d_product p ON p.prod_id = 10
WHERE  p.category = 'ELEC';
```

> **주의**: 의도치 않게 OUTER를 **INNER로 망가뜨리는** 전형적 버그는, **오른쪽 조건을 JOIN의 ON이 아니라 WHERE에 쓰는 것**입니다.
> OUTER 의미를 유지하려면 **오른쪽 조건은 반드시 ON 절**에 두세요.

---

## 규칙 #2 — **무결성으로 매칭이 항상 보장**되는 경우

### FK **ENABLE VALIDATE** + FK 컬럼 **NOT NULL** + 오른쪽 **PK/Unique**

```sql
-- (원형) 고객과 판매를 OUTER로 조인 (의미상 INNER와 동일)
SELECT c.cust_id, s.sales_id
FROM   d_customer c
LEFT  JOIN f_sales s
       ON s.cust_id = c.cust_id;

-- 조건: s.cust_id는 NOT NULL + FK VALIDATE, c.cust_id는 PK
-- → 각 s 행은 반드시 c가 있고, 각 c는 s가 없을 수 있으나
--    아래처럼 WHERE에 s 컬럼 필터가 없으면 LEFT 유지 의미가 있을 수 있음
```
- 이 케이스에서 **항상 매칭**이 보장되는 건 **자식→부모**(s→c) 방향입니다.
- 반대로 `c LEFT JOIN s`는 **고객은 매출이 없을 수도** 있으므로 **항상 INNER로 바꾸면 안 됩니다**.
- **항상 매칭**이 되는지의 판단은 **좌/우와 방향**에 민감합니다.

#### 3.1-A 부모→자식(Left가 부모)인 경우

- `c LEFT JOIN s`는 **매출 없는 고객**을 포함하는 게 목적이면 OUTER를 유지해야 합니다.
- 하지만 **추가적으로 `WHERE s.sales_id IS NOT NULL` 같은 NULL-배제 필터가 있다면**, 결국 **INNER**로 동치가 됩니다(규칙 #1).

#### 3.1-B 자식→부모(Left가 자식)인 경우

```sql
-- (자식이 왼쪽) 매출에서 고객을 OUTER로 조인 (항상 존재)이므로 곧바로 INNER로 안전 변환
SELECT s.sales_id, c.region
FROM   f_sales s
LEFT  JOIN d_customer c
       ON c.cust_id = s.cust_id;

-- (동치) INNER
SELECT s.sales_id, c.region
FROM   f_sales s
JOIN   d_customer c
       ON c.cust_id = s.cust_id;
```
- **자식→부모** OUTER는 **불필요**(항상 존재) → **INNER/조인제거**까지 가능(부모 컬럼 미참조 시).
- 이는 **조인 제거**(Join Elimination) 주제와 맞닿아 있습니다.

---

## 규칙 #3 — **존재성만 확인**하는 패턴

### > 0` / `EXISTS` 관계

```sql
-- (원형) LEFT 후 집계로 존재성만 확인
SELECT c.cust_id,
       CASE WHEN COUNT(s.sales_id) > 0 THEN 'ACTIVE' ELSE 'NEW' END AS status
FROM   d_customer c
LEFT  JOIN f_sales s ON s.cust_id = c.cust_id
GROUP  BY c.cust_id;

-- (대체) SEMI-JOIN (EXISTS) 또는 INNER + DECODE
SELECT c.cust_id,
       CASE WHEN EXISTS (SELECT 1 FROM f_sales s WHERE s.cust_id = c.cust_id)
            THEN 'ACTIVE' ELSE 'NEW' END AS status
FROM   d_customer c;
```
- 목적이 **존재성**이면 OUTER + COUNT 대신 **SEMI-JOIN(EXISTS)** 를 쓰는 편이 **명확하고 빠른** 경우가 많습니다.
- 단, **왼쪽 모든 행을 반드시 내보내야 한다**는 목적이면 EXISTS와 스칼라 서브쿼리를 조합하거나 **OUTER**를 유지해야 합니다.

---

## 안전하지 않은 변환(주의/반례)

### OUTER 의미가 필요한데 WHERE에 오른쪽 조건을 둔 경우

```sql
-- (의도) 매출이 없는 고객도 보여주되, 매출이 있으면 ELEC만 가져오고 싶다
SELECT c.cust_id, s.sales_id
FROM   d_customer c
LEFT  JOIN f_sales s
       ON s.cust_id = c.cust_id
      AND s.prod_id IN (SELECT prod_id FROM d_product WHERE category='ELEC')
-- WHERE s.prod_id IN (...)  -- ← 이렇게 WHERE로 내리면 OUTER가 INNER로 변질(버그!)
;
```
- **해결**: **오른쪽 조건은 반드시 ON 절**에 유지해야 OUTER 의미 보존.

### `FULL OUTER JOIN`은 거의 변환 불가

- 양측 **모두**를 NULL-보존 → 특별한 **양방향 1:1 필수 매칭**(둘 다 NOT NULL FK/PK, 상호참조) 같은 아주 드문 경우에만 INNER와 같아집니다.
- 대부분의 FOJ는 **INNER로 바꾸면 안 됩니다**.

---

## 변환이 성능에 주는 이점

- **조인 순서 자유도 증가**: OUTER는 **보존측** 제약으로 인해 옵티마이저의 재배치를 제한합니다. INNER로 바뀌면 **LEADING/USE_NL/USE_HASH** 등 전략 선택 폭이 커짐.
- **프레디킷 푸시다운**: INNER에서는 오른쪽 조건을 더 일찍/더 깊게 **푸시**하여 **읽기량 감소**.
- **트랜지티브 이행**: `L.k = R.k AND R.k = :b` → `L.k = :b` 같은 이행이 INNER에서 더 적극적으로 적용.

**관찰 루틴**
```sql
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +NOTE'));
-- 변환 전/후 플랜 비교: 조인 형태(OUTER→INNER), 접근 경로, A-Rows, 읽기량 변화
```

---

## 케이스 스터디

### WHERE의 오른쪽 조건으로 인한 자동 변환

```sql
-- Before (문법은 LEFT지만 WHERE가 INNER를 강제)
SELECT /* before */ c.cust_id, s.sales_id
FROM   d_customer c
LEFT  JOIN f_sales s ON s.cust_id = c.cust_id
WHERE  s.sales_dt >= DATE '2025-02-01';

-- After (명시적 INNER로 동일 의미)
SELECT /* after  */ c.cust_id, s.sales_id
FROM   d_customer c
JOIN   f_sales s ON s.cust_id = c.cust_id
WHERE  s.sales_dt >= DATE '2025-02-01';
```
- OUTER 결과의 NULL 확장 행은 `s.sales_dt >= ...`에서 **걸러짐**.

### + 더 나아가 조인 제거

```sql
-- 부모 컬럼 미참조 + 무결성 보장 시
SELECT /* before */ s.sales_id
FROM   f_sales s
LEFT  JOIN d_customer c ON c.cust_id = s.cust_id;

-- 부모 컬럼을 쓰지 않으면 조차 '조인 자체'가 필요 없음
SELECT /* after  */ s.sales_id
FROM   f_sales s;
```
- 이는 OUTER→INNER를 넘어 **조인 제거**로 연결됩니다.

### 보고서 패턴: 조건부 OUTER → 조건에 따라 INNER

```sql
-- '필터가 주어지면' INNER, 아니면 OUTER 유지하고 싶을 때(동적)
-- (안전 패턴) 조건을 ON에 유지하고, 파라미터가 NULL이면 필터를 스킵
SELECT c.cust_id, s.sales_id
FROM   d_customer c
LEFT  JOIN f_sales s
  ON   s.cust_id = c.cust_id
 AND  (:lo IS NULL OR s.sales_dt >= :lo)
 AND  (:hi IS NULL OR s.sales_dt <  :hi);
```
- 파라미터 유무에 따라 **의미를 명확히 통제**합니다.
- `WHERE`로 내리면 OUTER 의미가 **예상치 않게** 바뀔 수 있습니다.

---

## 변환 템플릿(치트시트)

| 패턴(LEFT 기준) | 변환 |
|---|---|
| `LEFT JOIN R ON ...` **`WHERE R.col IS NOT NULL`** | `INNER JOIN R ON ...` + `WHERE` 그대로 |
| `LEFT JOIN R ON ...` **`WHERE R.col = :x`** | `INNER JOIN R ON ... AND R.col = :x` (또는 WHERE 유지) |
| `LEFT JOIN R ON ...` **`WHERE R.col > ... / IN (...) / LIKE ...`** | 동일하게 **INNER** 로 변환 가능 |
| `S.LEFT JOIN R ... GROUP BY ... HAVING COUNT(R.key) > 0` | `S INNER JOIN R ...` 또는 `S WHERE EXISTS(...)` (세미조인) |
| 자식→부모 OUTER, **부모 컬럼 미참조**, FK VALIDATE + NOT NULL | `INNER` → 더 나아가 **조인 제거** 가능 |

> **유지 템플릿(반대로 OUTER를 지키고 싶을 때)**
> 오른쪽 조건은 반드시 **`ON`** 절에 두고, **`WHERE R.col ...`** 을 피한다.

---

## 테스트 & 검증 쿼리

```sql
-- 전/후 행수 확인
SELECT COUNT(*) FROM (
  SELECT c.cust_id, s.sales_id
  FROM   d_customer c
  LEFT  JOIN f_sales s ON s.cust_id = c.cust_id
  WHERE  s.sales_dt >= DATE '2025-02-01'
);

SELECT COUNT(*) FROM (
  SELECT c.cust_id, s.sales_id
  FROM   d_customer c
  JOIN   f_sales s ON s.cust_id = c.cust_id
  WHERE  s.sales_dt >= DATE '2025-02-01'
);

-- 실행계획 비교
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +ALIAS +NOTE'));
```

---

## 자주 나오는 Q&A

**Q1. WHERE에 `NVL(r.col, 'X') = 'X'`는 OUTER를 INNER로 바꾸나요?**
- 이 조건은 **NULL을 ‘X’로 치환**해 **참**이 되므로 **NULL-배제가 아닙니다**. 즉, OUTER 의미를 보존합니다.
- 하지만 **인덱스/SARGABLE**에 불리하고, 의도 오해 소지가 큽니다.

**Q2. RIGHT OUTER JOIN도 같은가요?**
- 네, 좌우만 바뀌었습니다. **WHERE의 왼쪽(NULL-확장측 반대편)** 컬럼에 NULL-배제 조건이 있으면 **INNER로 동치**입니다.

**Q3. 옵티마이저가 자동으로 바꿔주나요?**
- Oracle CBO는 **Outer-to-Inner 변환**을 **안전 조건**에서 자동 수행합니다.
- 그러나 **ANSI OUTER 의미를 보존해야 하는** 경우에는 변환을 막아야 합니다(오른쪽 조건은 ON에 유지).

---

## 마무리 체크리스트

- [ ] WHERE에 **오른쪽 컬럼**의 **NULL-배제** 조건이 있는가? → **INNER 변환 가능**
- [ ] 변환 후 **의미(행수/NULL 보존)** 가 동일한가? → **샘플/테스트**로 검증
- [ ] **자식→부모** OUTER인가? FK/PK로 **항상 매칭**이면 불필요한 OUTER/INNER/조인 제거 고려
- [ ] OUTER 의미를 유지해야 하는가? → 오른쪽 조건은 **ON 절**에 유지
- [ ] 변환 후 플랜: **조인 순서/푸시다운/읽기량** 개선 확인 (`DBMS_XPLAN ... ALLSTATS LAST`)

---

## 추가 실전 예제 모음

### 보고서: “매출 있으면 ELEC만, 없으면 NULL”

```sql
-- OUTER 유지 (정답)
SELECT c.cust_id, s.sales_id
FROM   d_customer c
LEFT  JOIN f_sales s
  ON  s.cust_id = c.cust_id
 AND EXISTS (SELECT 1 FROM d_product p
             WHERE p.prod_id = s.prod_id
               AND p.category = 'ELEC');

-- 잘못된 예(버그): WHERE로 내리면 INNER화
-- WHERE EXISTS (SELECT 1 FROM d_product p ... )  ← X
```

### 존재성만 필요할 때의 단순화

```sql
-- Before: OUTER + 그룹 + COUNT
SELECT c.cust_id,
       CASE WHEN COUNT(s.sales_id) > 0 THEN 1 ELSE 0 END AS has_sales
FROM   d_customer c
LEFT  JOIN f_sales s ON s.cust_id = c.cust_id
GROUP  BY c.cust_id;

-- After: EXISTS (SEMI-join)
SELECT c.cust_id,
       CASE WHEN EXISTS (SELECT 1 FROM f_sales s WHERE s.cust_id = c.cust_id)
            THEN 1 ELSE 0 END AS has_sales
FROM   d_customer c;
```

### 스타일 변환 예

```sql
-- Before (LEFT OUTER 의미지만 WHERE가 INNER화)
SELECT c.cust_id, s.sales_id
FROM   d_customer c, f_sales s
WHERE  s.cust_id(+) = c.cust_id
AND    s.sales_dt   >= DATE '2025-02-01';

-- After (ANSI INNER)
SELECT c.cust_id, s.sales_id
FROM   d_customer c
JOIN   f_sales s ON s.cust_id = c.cust_id
WHERE  s.sales_dt >= DATE '2025-02-01';
```

---

### 결론

- **LEFT/RIGHT OUTER**를 **INNER**로 바꾸는 핵심은 **NULL-배제 조건**과 **무결성 보장**입니다.
- 변환은 단순 문법 치환이 아니라 **의미 보존 검증** 과정입니다.
- 변환에 성공하면 옵티마이저 최적화 여지가 커져 **체감 성능 개선**을 얻는 경우가 많습니다.
- 항상 **플랜/행수/통계**로 **전/후 검증**하세요.
