---
layout: post
title: DB 심화 - PQ_DISTRIBUTE
date: 2025-11-25 15:25:23 +0900
category: DB 심화
---
# Oracle **PQ_DISTRIBUTE** 힌트

**주제**  
1) PQ_DISTRIBUTE 힌트의 **용도(왜 쓰는가)**  
2) **구문**과 의미를 실행계획 관점에서 정확히 이해  
3) 지원되는 **분배 방식 키워드**와 조합/패턴  
4) **튜닝 사례**: 브로드캐스트/해시/파티션-와이즈 강제, 스큐 완화, RAC 대기 절감  

---

## 0) 들어가며 — “병렬 성능의 절반은 데이터 분배가 결정한다”

병렬(PQ)은 “여러 PX 슬레이브가 일을 나눠 한다”는 사실만으로 빨라지지 않습니다.  
**PX끼리 데이터를 어떻게 나누고(Distribute), 어떻게 모으느냐(Join/Aggregate)**가 스루풋과 대기 이벤트를 좌우합니다.

옵티마이저는 통계·카디널리티·파티션 구조·조인 형태(outer/inner) 등을 근거로  
- **HASH 재분배**(키 해시로 양쪽을 같은 PX에 모으기),  
- **BROADCAST 복제**(작은 쪽을 모든 PX로 뿌리기),  
- **PARTITION-WISE JOIN**(파티션 맞춤으로 재분배 생략),  
- 그 외 **RANGE/ROUND-ROBIN** 등  
중 하나를 선택합니다.  

하지만 **통계 오류, 키 스큐, 설계 제약, RAC 인터커넥트 상황** 때문에  
옵티마이저의 기본 선택이 **현실에서 최적이 아닐 때**가 많습니다.  
이때 **PQ_DISTRIBUTE**로 “PX SEND 방식”을 **의도대로 고정**해 성능을 안정화합니다. :contentReference[oaicite:0]{index=0}

---

## 1) 병렬 조인의 데이터 흐름 복습(이걸 알아야 힌트를 읽는다)

### 1.1 DFO / TQ / QC / PX의 역할

- **QC(Query Coordinator)**: SQL을 시작/끝내고 결과를 최종 소비하는 조정자.
- **PX 슬레이브**: QC가 만든 실행 계획을 병렬로 실제 수행하는 워커.
- **DFO(Data Flow Operation)**: 병렬 파이프라인의 큰 단위. “한 덩어리 병렬 작업”.
- **TQ(Table Queue)**: DFO 사이에서 **데이터가 이동하는 큐**. 실행계획에 `TQ10000`, `TQ10001`처럼 찍힘.
- **PX SEND / PX RECEIVE**: TQ를 통해 **전송/수신**하는 오퍼레이터.  
  → **어떤 SEND가 찍히는지**가 곧 **분배 방식**입니다.

### 1.2 IN-OUT이 말해주는 것

`DBMS_XPLAN.DISPLAY_CURSOR`에서 **IN-OUT 컬럼**은 row source가 병렬 파이프라인에서  
**어디서 어디로 흐르는지**를 짧게 요약합니다.

- `S->P`: Serial 생산물을 Parallel로 분해(예: QC가 만든 작은 집합을 PX로 뿌림).
- `P->S`: PX가 만든 결과를 QC로 회수.
- `P->P`: PX끼리 **재분배(send/receive)** 후 다음 병렬 단계로 전달.
- `PCWP / PCWC`: “Parent/Child parallel-wise” 결합.  
  **같은 PX 집합이 연속 연산**을 수행해 **TQ와 메시징을 생략**하는 최적화.

### 1.3 PX SEND 타입(분배 방식의 언어)

실행계획에서 가장 중요한 라인:

- `PX SEND HASH`
- `PX SEND BROADCAST`
- `PX SEND PARTITION (RANGE/HASH)`
- `PX SEND ROUND-ROBIN` …  

**PQ_DISTRIBUTE는 바로 이 SEND 타입을 강제**하기 위한 힌트입니다. :contentReference[oaicite:1]{index=1}

---

## 2) PQ_DISTRIBUTE 힌트란 무엇인가(정의와 작동 지점)

### 2.1 정의

`PQ_DISTRIBUTE`는 **병렬 조인/집계 등에서 특정 row source가 다른 row source와 만날 때,  
어떤 방식으로 PX에 분배될지(=PX SEND 방법)를 지정**하는 옵티마이저 힌트입니다. :contentReference[oaicite:2]{index=2}

즉, “**이 테이블(또는 뷰)의 로우를 해시로 나눌지, 복제할지, 파티션-와이즈로 맞출지**”를 명령합니다.

### 2.2 힌트가 영향을 주는 시점

- **해시 조인 / 소트 머지 조인 / 그룹바이 / DISTINCT / 윈도우 집계** 등  
  “**다수 PX가 같은 키를 기준으로 만나야 하는 연산**”에서 분배가 생깁니다.
- NL 조인은 주로 “드라이빙 로우를 PX에 나눠 인덱스로 탐색”하는 구조라  
  PQ_DISTRIBUTE의 직접적 영향이 제한적일 수 있습니다(다만 NL의 드라이빙 집합을 S->P로 나누는 데 간접 영향).

---

## 3) 구문 — **method_in / method_out의 정확한 의미**

```sql
/*+ PQ_DISTRIBUTE( table_alias  method_in  method_out ) */
```

- `table_alias`  
  힌트를 적용할 **테이블/뷰의 별칭**.  
  **별칭을 정확히 써야** 하며, 조인 순서·뷰 병합 여부에 따라 대상이 달라질 수 있습니다.

- `method_in`, `method_out`  
  이 row source가 **파이프라인에서 “들어올 때(in)”**와  
  **다른 row source와 결합해 “나갈 때(out)”** 어떤 분배 전략을 쓰는지 지정합니다.

실무적으로는 “**조인에 투입되는 방향(in)과, 조인 후 다음 단계로 넘어가는 방향(out)**에 대한 분배 정책”으로 이해하면 안전합니다.  
대부분의 조인 튜닝에선 **in/out을 같은 값으로 주거나**,  
**한쪽만 강조하고 반대쪽은 NONE**으로 둡니다. :contentReference[oaicite:3]{index=3}

---

## 4) 지원되는 분배 키워드와 “현장 표준 조합”

조인 튜닝에서 **명시적으로 쓰는 표준 키워드**는 아래 네 가지입니다. :contentReference[oaicite:4]{index=4}

| 키워드 | 의미 | 데이터 흐름(플랜에서) | 추천 상황 |
|---|---|---|---|
| `HASH` | 조인 키 해시로 균등 재분배 | `PX SEND HASH` + `P->P` | 대형↔대형, 키 균등 |
| `BROADCAST` | 소형 집합을 모든 PX에 복제 | `PX SEND BROADCAST` | 소형 차원 vs 대형 사실 |
| `PARTITION` | 파티션-와이즈 정렬/조인 지향 | `PX PARTITION ...` / SEND 최소 | Full/Partial PWJ 가능 |
| `NONE` | 재분배 없음(로컬 처리) | SEND 없음 또는 상대편만 SEND | 브로드캐스트 반대편, PWJ |

### 4.1 현장 표준 조합 4종

1) **대형-대형 해시 조인**  
```sql
pq_distribute(a HASH HASH)
pq_distribute(b HASH HASH)
```
→ 양쪽을 **같은 키 해시로 재분배**해서 **동일 키가 같은 PX에서 만나게** 함.

2) **소형-대형 브로드캐스트**  
```sql
pq_distribute(dim BROADCAST NONE)
```
→ **dim만 복제**, 사실(fact)은 재분배 없이 로컬 처리.

3) **Full Partition-Wise Join(동일 파티션)**  
```sql
pq_distribute(a PARTITION PARTITION)
pq_distribute(b PARTITION PARTITION)
```
→ 두 테이블이 **조인 키로 “동일한 방식/경계/개수”로 파티셔닝**돼 있을 때  
**재분배를 거의 제거**하고 파티션끼리 조인.  
Full PWJ는 두 테이블이 **조인키 기준으로 equi-partition**일 때만 가능. :contentReference[oaicite:5]{index=5}

4) **Partial PWJ / Dynamic 보정(비파티션 정렬)**  
```sql
pq_distribute(np HASH HASH)
```
→ 비파티션 큰 테이블을 조인 키 기준으로 재분배해 **파티션 테이블과 “가상 정렬”**.  
오라클은 이런 형태를 **partial / dynamic partition-wise**로 구현합니다. :contentReference[oaicite:6]{index=6}

---

## 5) 실행계획에서 “힌트가 먹었는지” 확인하는 법

### 5.1 가장 확실한 출력

```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
       NULL, NULL,
       'BASIC +PARALLEL +ALIAS +PREDICATE +NOTE'));
```

확인 포인트:

1) **`PX SEND ...` 라인**  
   - HASH / BROADCAST / PARTITION 중 **정확히 원하는 타입이 찍혔는가**

2) **`IN-OUT` 컬럼**  
   - HASH/BROADCAST면 대개 `P->P`  
   - PWJ면 `P->P`가 줄고 PARTITION row source가 앞단에서 나타남

3) **`NOTE` 섹션**  
   - 힌트 사용/무시 이유가 적히고,  
   - PWJ가 성립하면 partition-wise 관련 NOTE가 등장할 수 있습니다. :contentReference[oaicite:7]{index=7}

4) **버퍼링 여부**  
   - 해시 조인에서 `BUFFERED`가 찍히면 **재분배 결과를 임시 버퍼링**하는 경우(메모리/Temp 영향).  
     분배 방식 자체를 바꾸거나 PGA 여유를 늘려야 할 때가 많습니다. :contentReference[oaicite:8]{index=8}

### 5.2 스큐/바이트는 V$PQ_TQSTAT로

```sql
SELECT dfo_number, tq_id, server_type, inst_id, process,
       num_rows, bytes
FROM   v$pq_tqstat
ORDER  BY dfo_number, tq_id, server_type, inst_id, process;
```

- **슬레이브별 num_rows 편차**가 크면 **키 스큐**.  
- HASH/HASH일 때 편차가 크면 `PX Deq Credit: send blkd`가 치솟기 쉽습니다.

---

## 6) 예제로 배우는 PQ_DISTRIBUTE (확장판)

이 절의 예제는 사용자가 준 공통 스키마를 그대로 씁니다.

---

### 6.1 대형↔대형: **HASH-HASH 강제**

#### 6.1.1 기본형

```sql
EXPLAIN PLAN FOR
SELECT /*+ leading(s) use_hash(c)
           parallel(s 16) parallel(c 16)
           pq_distribute(s HASH HASH)
           pq_distribute(c HASH HASH)
           monitor */
       s.cust_id, SUM(s.amount) amt, SUM(c.cost) cost
FROM   fact_sales s
JOIN   fact_clicks c
  ON   c.cust_id = s.cust_id
GROUP  BY s.cust_id;

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,
     'BASIC +PARALLEL +ALIAS +NOTE'));
```

#### 6.1.2 계획 해석

- `PX SEND HASH`가 **양쪽에 각각 1회 이상** 나타나는지 확인.
- `IN-OUT = P->P`가 조인 바로 위/아래에 찍히는 게 일반적.

#### 6.1.3 언제 최고인가

- 두 집합이 모두 커서 **브로드캐스트가 불가능**하고,
- 조인 키가 **비교적 균등**할 때.

#### 6.1.4 스큐가 있으면 어떻게 망가지는가

- 특정 키가 전체의 상당 부분을 차지하면  
  → 그 키를 담당한 PX 한두 개만 과부하  
  → 다른 PX는 놀고, 과부하 PX의 전송 큐가 막혀  
  → `PX Deq Credit: send blkd` / `PX Deq Credit: need buffer`가 급증.

**해시-해시의 최대 약점은 스큐**입니다.

#### 6.1.5 스큐 완화(실전형 SALT + 2단계 집계)

```sql
WITH S AS (
  SELECT /*+ parallel(16) */
         cust_id,
         amount,
         MOD(ORA_HASH(cust_id), 8) AS salt
  FROM fact_sales
),
C AS (
  SELECT /*+ parallel(16) */
         cust_id,
         cost,
         MOD(ORA_HASH(cust_id), 8) AS salt
  FROM fact_clicks
)
SELECT /*+ leading(S) use_hash(C)
           parallel(16)
           pq_distribute(S HASH HASH)
           pq_distribute(C HASH HASH)
           monitor */
       S.cust_id,
       SUM(S.amount) amt,
       SUM(C.cost)   cost
FROM   S
JOIN   C
  ON   C.cust_id = S.cust_id
 AND   C.salt    = S.salt
GROUP  BY S.cust_id;
```

- 내부 분배는 `(cust_id, salt)`로 **강제 균등화**  
- 최종 결과는 `cust_id`로 다시 합산.  
- 스큐가 심한 DW에서 **가장 보편적인 해시-기반 스큐 완화 패턴**입니다.

---

### 6.2 소형↔대형: **BROADCAST 강제**

#### 6.2.1 기본형

```sql
EXPLAIN PLAN FOR
SELECT /*+ leading(d) use_hash(s)
           parallel(d 8) parallel(s 16)
           pq_distribute(d BROADCAST NONE)
           monitor */
       s.cust_id, SUM(s.amount) amt
FROM   dim_customer d
JOIN   fact_sales   s
  ON   s.cust_id = d.cust_id
WHERE  d.grade IN ('GOLD','SILVER')
GROUP  BY s.cust_id;

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,
     'BASIC +PARALLEL +ALIAS +NOTE'));
```

#### 6.2.2 계획 해석

- dim 쪽에서 `PX SEND BROADCAST`가 찍히는지 확인.
- fact 쪽은 **재분배 SEND가 최소**(또는 없음).

#### 6.2.3 왜 빠른가

큰 fact를 해시로 재분배하면  
**fact 전체가 네트워크를 타고 흔들립니다.**

브로드캐스트는  
**작은 dim만 네트워크를 타고 복제**되고  
fact는 **각 PX가 자기 스캔 구간을 그대로 해시조인**하니  
**네트워크·큐잉 대기가 획기적으로 줄어듭니다.** :contentReference[oaicite:9]{index=9}

#### 6.2.4 브로드캐스트의 한계

- dim이 “작다고 생각했는데” 실제로는 크면  
  → 모든 PX에 복제되는 바이트가 **DOP 배수로 커짐**  
  → 오히려 역효과.  

따라서 **브로드캐스트 대상은 먼저 필터·프루닝으로 소형화**하는 것이 핵심입니다.

#### 6.2.5 소형화 + 브로드캐스트(현장 패턴)

```sql
WITH D AS (
  SELECT /*+ materialize */
         cust_id, grade, active_yn
  FROM dim_customer
  WHERE active_yn = 'Y'
    AND grade IN ('GOLD','SILVER')
)
SELECT /*+ leading(D) use_hash(s)
           parallel(D 4) parallel(s 16)
           pq_distribute(D BROADCAST NONE)
           monitor */
       s.cust_id, SUM(s.amount)
FROM   D
JOIN   fact_sales s
  ON   s.cust_id = D.cust_id
GROUP  BY s.cust_id;
```

- `materialize`는 **필터된 결과를 먼저 만들고**  
  그 작은 결과를 브로드캐스트하도록 유도하는 흔한 테크닉.

---

### 6.3 Full Partition-Wise Join: **PARTITION PARTITION 강제**

#### 6.3.1 기본형

```sql
EXPLAIN PLAN FOR
SELECT /*+ leading(s) use_hash(c)
           parallel(s 16) parallel(c 16)
           pq_distribute(s PARTITION PARTITION)
           pq_distribute(c PARTITION PARTITION)
           monitor */
       s.cust_id, SUM(s.amount) amt, SUM(c.cost) ad_cost
FROM   fact_sales  s
JOIN   fact_clicks c
  ON   c.cust_id = s.cust_id
GROUP  BY s.cust_id;

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,
     'BASIC +PARALLEL +ALIAS +NOTE +PARTITION'));
```

#### 6.3.2 Full PWJ 요건(정확히)

Full PWJ는  
- **조인 키가 파티션 키와 같고**,  
- **양쪽이 동일한 파티션 방식/경계/개수(equi-partition)로 구성**돼야 합니다. :contentReference[oaicite:10]{index=10}

#### 6.3.3 계획 해석

- `PX PARTITION HASH` 같은 partition row source가  
  **조인(FULL PWJ) 앞에 나타나면** PWJ 성립. :contentReference[oaicite:11]{index=11}
- `PX SEND HASH/BROADCAST`가 **사라지거나 아주 약해지는지** 확인.

#### 6.3.4 왜 선형 확장성이 최고인가

- **재분배(PX SEND) 자체가 줄어** 네트워크 바이트와 큐잉이 작고,
- 각 파티션이 고유 PX에 고정되어  
  **지역성(locality)**이 극대화됩니다.

특히 RAC에서  
파티션-서비스 affinity가 맞으면  
인터커넥트 `gc cr/current request`도 크게 줄어  
PWJ는 **RAC-친화적 최고 패턴**이 됩니다. :contentReference[oaicite:12]{index=12}

---

### 6.4 Partial PWJ / Dynamic 보정: **비파티션을 HASH로 맞추기**

#### 6.4.1 기본(Partial)

```sql
EXPLAIN PLAN FOR
SELECT /*+ leading(s) use_hash(o)
           parallel(s 16) parallel(o 16)
           monitor */
       s.cust_id, SUM(s.amount), SUM(o.amt)
FROM   fact_sales s
JOIN   orders_np  o
  ON   o.cust_id = s.cust_id
GROUP  BY s.cust_id;
```

- fact_sales는 파티션 키가 조인 키라 **파티션 단위 병렬 스캔**이 유리.
- 하지만 orders_np는 비파티션이라  
  옵티마이저가 HASH-재분배를 선택해도 **교차 통신이 남는** 경우가 많습니다. :contentReference[oaicite:13]{index=13}

#### 6.4.2 비파티션 보정(동적 정렬)

```sql
EXPLAIN PLAN FOR
SELECT /*+ leading(s) use_hash(o)
           parallel(s 16) parallel(o 16)
           pq_distribute(o HASH HASH)
           monitor */
       s.cust_id, SUM(s.amount), SUM(o.amt)
FROM   fact_sales s
JOIN   orders_np  o
  ON   o.cust_id = s.cust_id
GROUP  BY s.cust_id;

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
     'BASIC +PARALLEL +ALIAS +NOTE'));
```

- orders_np를 **cust_id 해시로 재분배**해  
  fact_sales의 해시-파티션과 **“가상으로 맞춘다”**는 발상.
- 실행계획에서 orders_np 쪽 `PX SEND HASH`가 찍히는지 확인.

#### 6.4.3 언제 유리한가

- 비파티션 쪽이 충분히 크고(브로드캐스트 불가),
- 조인 키가 비교적 균등하며,
- fact의 물리 파티션 이점을 살리고 싶을 때.

---

## 7) 실전 튜닝 케이스(현장형 확장)

### 7.1 해시-해시 vs 브로드캐스트 “결정 트리”

1) **작은 쪽이 진짜 작다**  
   - 필터/프루닝 후 크기 확인  
   → **BROADCAST**가 1순위.

2) **둘 다 크다**  
   - 키 균등  
   → **HASH-HASH**.

3) **키 스큐가 크다**  
   - HASH-HASH 유지 시 SALT/부분집계  
   - 또는 **작은 쪽이면 BROADCAST로 회피**.

4) **동일 파티션 가능**  
   → **PARTITION-WISE(최우선)**.

### 7.2 조인 순서와의 결합

PQ_DISTRIBUTE는 “**그 row source가 어떤 입력 위치에 서느냐**”에 따라 효과가 달라집니다.

따라서  
- `LEADING`, `USE_HASH`, `SWAP_JOIN_INPUTS` 등으로  
  **드라이빙/빌드/프로브 순서를 고정**한 뒤  
- PQ_DISTRIBUTE로 **분배까지 고정**하는 게 가장 안전합니다. :contentReference[oaicite:14]{index=14}

### 7.3 분배 비용은 플랜에 잘 드러나지 않는다

실행계획의 cost는  
**PX 분배 네트워크 비용을 충분히 외부화하지 못하는 한계**가 있어  
“플랜만 보면 좋아 보이는데 실제는 느린” 일이 자주 생깁니다.  
이 때문에 실무에서 PQ_DISTRIBUTE로 **분배를 실측 기반으로 고정**합니다. :contentReference[oaicite:15]{index=15}

---

## 8) 힌트가 무시되는 대표 원인(함정)

1) **별칭 불일치 / 뷰 병합**  
   - 힌트 대상 alias가 플랜에서 사라지면 힌트도 사라짐.  
   - 뷰는 `MERGE/NO_MERGE`, `INLINE/MATERIALIZE`를 함께 고려.

2) **힌트 상충**  
   - 예: `USE_HASH`와 `USE_NL`, 또는 분배 힌트끼리 충돌.  
   - `DISPLAY_CURSOR ... +NOTE`에서 “why not used”를 확인. :contentReference[oaicite:16]{index=16}

3) **PWJ 전제 불충족**  
   - PARTITION 힌트를 줘도  
     파티션 키/경계가 다르면 Full PWJ 불가. :contentReference[oaicite:17]{index=17}

4) **브로드캐스트 대상이 크다**  
   - 통계가 틀려 “작다고 착각”하면 대참사.  
   - **필터 선적용 → 통계 갱신 → 브로드캐스트** 순서가 정석.

5) **과대 DOP**  
   - 분배 전략이 맞아도 DOP가 과하면  
     큐잉/Temp/메모리 경합이 성능을 무너뜨림.

---

## 9) 모니터링 루틴(고정 템플릿)

```sql
-- 1) 실제 실행 플랜
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
     NULL, NULL,
     'BASIC +PARALLEL +ALIAS +PREDICATE +NOTE'));

-- 2) TQ 분배/스큐
SELECT dfo_number, tq_id, server_type, inst_id, process,
       num_rows, bytes
FROM   v$pq_tqstat
ORDER  BY dfo_number, tq_id, server_type, inst_id, process;

-- 3) 병렬 관련 대기 이벤트(ASH)
SELECT event, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time > SYSDATE - 1/288
AND    session_type = 'FOREGROUND'
AND    (event LIKE 'PX Deq%'
        OR event LIKE 'direct path%'
        OR event LIKE 'gc %')
GROUP  BY event
ORDER  BY samples DESC;
```

- `PX Deq Credit: send blkd`  
  → 해시 재분배 스큐/역압 대표 신호.  
- `direct path write/read temp`  
  → 해시 조인/정렬이 PGA 부족으로 Temp 스필 중.  
- `gc cr/current request`  
  → RAC 인터커넥트 병목(분배 최소화/PWJ/affinity 고려).

---

## 10) 의사결정 표(실전 한 장 요약)

| 상황 | PQ_DISTRIBUTE 추천 | 이유 |
|---|---|---|
| 소형 차원 vs 대형 사실 | `dim BROADCAST NONE` | fact 재분배 제거 |
| 대형↔대형, 키 균등 | 양쪽 `HASH HASH` | 동일 키 수렴, 지역 해시 조인 |
| Full PWJ 전제 성립 | 양쪽 `PARTITION PARTITION` | 재분배 최소, RAC-friendly |
| Partial PWJ, 비파티션 큼 | 비파티션 `HASH HASH` | 런타임 가상 정렬 |
| 스큐 심함 | SALT + 부분집계(해시 유지) | 슬레이브 불균형 해소 |
| 브로드캐스트 대상이 실제 큼 | HASH로 전환 | 복제 바이트 폭증 방지 |

---

## 11) 결론

- **PQ_DISTRIBUTE = 병렬 쿼리에서 “데이터가 어디로 어떻게 이동할지”를 지휘하는 스위치**입니다.  
- 실전에서 쓰는 키워드는 사실상 네 개( **HASH / BROADCAST / PARTITION / NONE** )면 충분합니다.
- 항상  
  1) **플랜의 PX SEND 타입**,  
  2) **IN-OUT 흐름**,  
  3) **V$PQ_TQSTAT 스큐**,  
  4) **ASH 대기 이벤트**  
  를 한 세트로 읽어야 “힌트를 주고 끝”이 아니라 **원인-처방-검증까지 닫힌 튜닝**이 됩니다.