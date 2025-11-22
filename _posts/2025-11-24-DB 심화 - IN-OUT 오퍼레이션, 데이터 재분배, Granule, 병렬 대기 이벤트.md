---
layout: post
title: DB 심화 - IN-OUT 오퍼레이션, 데이터 재분배, Granule, 병렬 대기 이벤트
date: 2025-11-24 22:25:23 +0900
category: DB 심화
---
# Oracle 병렬 실행 심화 — IN-OUT 오퍼레이션, 데이터 재분배, Granule, 병렬 대기 이벤트

## 0. 왜 이 네 덩어리(IN-OUT/재분배/Granule/대기)가 한 세트인가

병렬 실행은 결국 다음 4개의 질문으로 귀결된다.

1) **데이터는 어디서 어디로 흐르는가?** → `IN-OUT`  
2) **흐르다가 왜/어떻게 섞이는가?** → `PX SEND/RECEIVE`, `TQ`, Distribution  
3) **PX 서버가 “무엇을 얼마나” 나눠 먹는가?** → Granule(BRG/PG/SPG)  
4) **위 1~3이 꼬이면 어떤 기다림이 터지는가?** → 병렬 대기 이벤트

이 네 개를 동시에 읽어야 “플랜에서 보이는 원인 ↔ ASH/AWR에서 보이는 증상”이 1:1로 매칭된다.

---

## 1. 병렬 실행의 기본 구조를 “플랜이 보여주는 단어”로 다시 그리기

### 1.1 QC, PX 서버 집합, DFO, TQ

- **QC(Query Coordinator)**: SQL을 대표해서 계획을 세우고 PX 서버들을 지휘하는 **직렬 루트 프로세스**.  
- **PX 서버 집합(Server Set)**: “Producer 집합 / Consumer 집합”처럼 **병렬 워커 그룹**으로 움직인다.  
- **DFO(Data Flow Operation)**: 병렬 파이프라인의 **한 덩어리 흐름**. 플랜에서 DFO 경계마다 Server Set이 바뀌고 TQ가 끼어든다.  
- **TQ(Table Queue)**: **PX 서버 집합 간에 데이터를 주고받는 큐**. 플랜의 `TQ10000` 같은 표기, 그리고 **언제나 `PX SEND` ↔ `PX RECEIVE` 사이**에 존재한다. :contentReference[oaicite:0]{index=0}

### 1.2 병렬 플로우의 전형적 모양

```
QC
 └─(S->P)  PX Set 1  [Producer]
        └─ PX SEND  <distribution>  -> TQ
               PX RECEIVE
        └─(P->P)  PX Set 2  [Consumer]
                └─(P->S) QC 반환
```

- **`PX SEND <분배방식>`와 `PX RECEIVE`가 보이면**  
  “Producer Set에서 나온 데이터를 TQ로 **재분배해서** Consumer Set이 받는다”는 뜻이다. :contentReference[oaicite:1]{index=1}  
- 이때 `IN-OUT = P->P` 라인이 찍히는 것이 자연스럽다(다음 절).

---

## 2. IN-OUT 오퍼레이션: `DBMS_XPLAN +PARALLEL`을 읽는 진짜 기준

### 2.1 IN-OUT 표기 해석

`DBMS_XPLAN.DISPLAY[_CURSOR](... '+PARALLEL')`에서 `IN-OUT` 열은 **오퍼레이터가 누구에게서 받아 누구에게 주는지**를 나타낸다. Oracle은 이를 “Serial/Parallel 간 흐름”으로 표현한다. :contentReference[oaicite:2]{index=2}

- **S->P**: QC/직렬 단계가 **PX 서버 집합으로 데이터/파라미터를 넘긴다.**
- **P->S**: PX 서버 집합이 결과를 **QC로 돌려준다.**
- **P->P**: **PX 서버 집합 ↔ PX 서버 집합** 사이로 흐른다.  
  → **재분배(TQ)가 있다는 가장 강한 신호**.
- **PCWP / PCWC**:  
  - **PCWP**(Parallel Combined With Parent)  
  - **PCWC**(Parallel Combined With Child)  
  같은 Px 서버에서 부모/자식 오퍼레이터가 **합쳐져 실행**되어 **TQ 메시지 비용을 줄인다**. :contentReference[oaicite:3]{index=3}

> 실전 팁  
> 1) **P->P를 먼저 찾고**, 그 라인의 `PX SEND` 타입을 읽어 재분배 이유를 확정한다.  
> 2) PCWP/PCWC가 많이 보이면 “**파이프가 잘 붙어 있다(=메시지 오버헤드가 줄었다)**”는 뜻이다. :contentReference[oaicite:4]{index=4}

---

## 3. 데이터 재분배(Distribution): `PX SEND/RECEIVE`와 `PQ_DISTRIBUTE`로 보는 내부 동작

### 3.1 왜 재분배가 필요한가

병렬에서 성능이 좋아지려면 두 입력이 “비슷한 양/비슷한 비용”으로 **균등하게 분산**되어야 한다.  
그런데 조인/집계/정렬은 **키 단위로 모이거나(=GROUP BY/조인), 전역 순서를 맞춰야(=ORDER BY)** 한다.  
그래서 Oracle은 **중간에 한 번(혹은 여러 번) 데이터를 섞는다.** 이 섞임이 `PX SEND`/`PX RECEIVE`로 드러난다. :contentReference[oaicite:5]{index=5}

### 3.2 재분배 방식의 종류와 플랜 표기

플랜에 `PX SEND <TYPE>`로 나오는 대표 방식들:

| PX SEND 타입 | 언제 쓰나 | 핵심 효과 | 플랜/징후 |
|---|---|---|---|
| **HASH** | 해시 조인, 해시 GROUP BY | 동일 키를 같은 서버로 모음 | `PX SEND HASH` |
| **BROADCAST** | 작은 테이블을 큰 테이블로 조인할 때 | 작은 쪽을 **모든 PX에 복제** | `PX SEND BROADCAST` |
| **RANGE** | 전역 ORDER BY, Sort-Merge 조인 | 정렬키 **범위별로 분할** | `PX SEND RANGE` |
| **ROUND-ROBIN/RANDOM** | 키에 상관없는 균등 분산 | 단순 균등화 | `PX SEND ROUND ROBIN` 등 |
| **PARTITION / PARTITION HASH** | Partition-wise 조인/집계 | 파티션 소유를 맞춰 재분배 최소화 | `PX SEND PARTITION` |

`PQ_DISTRIBUTE` 힌트는 위 분배를 강제/유도한다. 특히 **작은 테이블 브로드캐스트**는 공식 힌트로 지원된다. :contentReference[oaicite:6]{index=6}

### 3.3 분배 방식별 실습 예제와 기대 플랜

#### (A) 대-대 조인: HASH-HASH

```sql
EXPLAIN PLAN FOR
SELECT /*+ leading(f) use_hash(c)
           parallel(f 16) parallel(c 16)
           pq_distribute(f HASH HASH)
           pq_distribute(c HASH HASH) */
       COUNT(*)
FROM   fact_sales f
JOIN   dim_customer c
  ON   c.cust_id = f.cust_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL +ALIAS +NOTE'));
```

**기대 포인트**
- `PX SEND HASH` / `PX RECEIVE`  
- `IN-OUT = P->P` (재분배 존재)  
- 두 집합이 해시 기반으로 균등 분산되지만 **키 스큐**가 있으면 한 PX로 쏠린다.

#### (B) 소-대 조인: BROADCAST

```sql
EXPLAIN PLAN FOR
SELECT /*+ leading(c) use_hash(f)
           parallel(f 16) parallel(c 16)
           pq_distribute(c BROADCAST NONE) */
       SUM(f.amount)
FROM   dim_customer c
JOIN   fact_sales  f
  ON   f.cust_id = c.cust_id;
```

**기대 포인트**
- `PX SEND BROADCAST`가 **c 쪽**에 찍힘  
- 큰 fact는 재분배가 거의 사라져 P->P가 줄어듦  
- **PX Deq Credit: send blkd** 같은 backpressure가 완화되는 경우가 많다(§5).

#### (C) ORDER BY: RANGE 분배

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) */
       f.sales_id, f.amount
FROM   fact_sales f
ORDER  BY f.amount DESC;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL +NOTE'));
```

**기대 포인트**
- `PX SEND RANGE` → 전역 정렬키 범위 분할  
- 상단에 `SORT ORDER BY` 또는 Top-N이면 `SORT ORDER BY STOPKEY`  
- **전역 순서 보장**을 위한 필수 단계. :contentReference[oaicite:7]{index=7}

### 3.4 스큐(불균형) 감지: `V$PQ_TQSTAT`

`V$PQ_TQSTAT`는 TQ별로 각 PX 서버가 **얼마나 보내고/받았는지(ROWS/BYTES)** 보여준다. :contentReference[oaicite:8]{index=8}

```sql
SELECT dfo_number, tq_id, server_type, process, num_rows, bytes
FROM   v$pq_tqstat
ORDER  BY dfo_number, tq_id, server_type, process;
```

**스큐 판독**
- 같은 TQ/Server_type에서 `num_rows`가 한두 프로세스에 몰리면  
  → **HASH 스큐** 또는 RANGE 경계 비대칭이란 뜻.
- 해결은 §6에서 종합 실습으로 확인한다.

---

## 4. Granule: “PX가 나눠 먹는 작업 조각”의 실체

Oracle 병렬 스캔은 데이터를 Granule 단위로 쪼개 PX 서버에 **동적 할당**한다. Granule의 종류는 크게 **BRG/PG/SPG**로 나뉜다. :contentReference[oaicite:9]{index=9}

### 4.1 BRG(Block Range Granule)

- 세그먼트를 **블록 범위**로 잘라 분배.  
- 플랜에서 **`PX BLOCK ITERATOR`**로 표시된다. :contentReference[oaicite:10]{index=10}  
- 장점: Granule 개수가 많아져 **동적 로드밸런싱**이 좋다  
  (빠른 PX가 더 많은 Granule을 계속 가져감).

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) */ SUM(amount)
FROM   fact_sales f;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL'));
```

### 4.2 PG(Partition Granule)

- **파티션 단위**로 Granule을 잡는다.  
- 플랜에서 `PX PARTITION RANGE [SINGLE|ITERATOR]` 같은 형태로 보인다. :contentReference[oaicite:11]{index=11}  
- **프루닝이 잘 먹을수록 PG 개수가 줄어** 재분배·I/O가 같이 준다.

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) */ SUM(amount)
FROM   fact_sales f
WHERE  f.sales_dt >= DATE '2025-02-01'
AND    f.sales_dt <  DATE '2025-03-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL +PARTITION'));
```

### 4.3 SPG(Subpartition Granule)

- 서브파티션 단위.  
- 대형 파티션 한 개가 DOP 전체를 먹어버리는 문제를 줄이려면  
  **서브파티셔닝으로 SPG를 늘려 Granule 다양성**을 확보한다. :contentReference[oaicite:12]{index=12}

### 4.4 Granule이 성능에 주는 3가지 영향

1) **균형**: Granule 수가 DOP보다 충분히 많을수록 로드밸런싱↑  
2) **지역성**: 파티션/서브파티션 단위(PG/SPG)면 파티션-와이즈 조인/집계가 쉬워짐  
3) **재분배 비용**: Partition-wise가 성립하면 `P->P` 자체가 크게 줄어든다.

---

## 5. 병렬 대기 이벤트: “플랜 구조 ↔ 대기 이벤트” 매칭법

병렬 실행이 느릴 때 대기는 크게 두 갈래로 터진다.

1) **메시징/큐(backpressure, admission)**  
2) **Direct Path I/O(TEMP 포함)**

### 5.1 메시징/큐 계열

#### `PX Deq Credit: send blkd`

- Producer PX가 Consumer 쪽 큐 크레딧을 못 받아 **전송이 막힌 상태**.  
- 실제 병렬 튜닝 사례에서 이 이벤트가 **스큐/Temp 병목과 함께** 상위에 나타난다. :contentReference[oaicite:13]{index=13}  
- **해석 공식**  
  - Producer는 빠른데 Consumer가 느림  
  - 원인은 대개  
    1) Consumer 쪽 **대형 해시/소트가 TEMP로 스필**,  
    2) **키 스큐**로 특정 Consumer만 과부하,  
    3) 또는 RAC/네트워크 병목.

**대응**
- **스큐 완화(§6.2)**  
- **BROADCAST 전환(소-대 조인)**  
- **PGA/Workarea 확대**로 one-pass/multi-pass를 줄여 Consumer를 가볍게

#### 기타 PX Deq 계열

- `PX Deq: Execute Reply` 등은 **QC/상위 단계가 슬레이브 응답을 기다리는** 형태다.  
- 단독으로 길게 나오면 “슬레이브의 실제 작업이 길다”는 뜻이므로 **플랜에서 가장 무거운 Consumer 단계**를 찾아야 한다.

### 5.2 Direct Path / TEMP I/O 계열

병렬 정렬·해시가 메모리를 넘기면 TEMP로 빠지고, 병렬은 버퍼캐시를 우회하기 때문에 **direct path read/write temp**가 핵심 신호가 된다. 실제 병렬 그룹바이/조인에서 TEMP direct path가 폭증하는 사례가 보고되어 있다. :contentReference[oaicite:14]{index=14}  

- **`direct path write temp`**  
  - 해시/소트가 **PGA를 넘어서 TEMP에 쓴다**.  
- **`direct path read temp`**  
  - 위에서 쓴 TEMP를 다시 읽는다(멀티-패스/재사용).

**대응**
- **입력 자체를 줄인다**: 프루닝, 선행 필터, Top-N  
- **PGA/Workarea를 늘린다**  
- **DOP를 과하게 키우지 않는다**: DOP↑가 TEMP 경합↑로 돌아설 수 있음

---

## 6. 종합 실습: “스큐 → send blkd → 개선”을 플랜/뷰로 끝까지 확인

### 6.1 (가정) fact에서 특정 키(cust_id=202)가 80% 이상인 스큐 데이터

데이터 준비는 대량 INSERT로 만들면 된다(여기선 패턴만 제시).

```sql
-- 스큐 데이터 예시(개념)
INSERT /*+ APPEND */ INTO fact_sales
SELECT level,
       DATE '2025-02-10',
       CASE WHEN MOD(level,10)<8 THEN 202 ELSE 101 END cust_id,
       'US',
       DBMS_RANDOM.VALUE(1,1000)
FROM dual CONNECT BY level<=5000000;
COMMIT;

EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'FACT_SALES');
```

### 6.2 A안: HASH-HASH 조인(스큐 악화)

```sql
SELECT /*+ leading(f) use_hash(c)
           parallel(f 16) parallel(c 16)
           pq_distribute(f HASH HASH)
           pq_distribute(c HASH HASH)
           monitor */
       COUNT(*)
FROM   fact_sales f
JOIN   dim_customer c ON c.cust_id=f.cust_id;
```

**확인 1) 플랜**

```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
     'BASIC +PARALLEL +ALIAS +NOTE'));
```

- `PX SEND HASH`  
- `IN-OUT = P->P` 명확  
- Consumer 쪽에 `HASH JOIN`/`HASH GROUP BY`가 빌드되어 **한 PX가 과다 처리**할 가능성.

**확인 2) 스큐**

```sql
SELECT dfo_number, tq_id, server_type,
       MIN(num_rows) mn, MAX(num_rows) mx,
       ROUND(100*(mx-mn)/NULLIF(mx,0),1) skew_pct
FROM   (
  SELECT dfo_number, tq_id, server_type, num_rows
  FROM   v$pq_tqstat
)
GROUP BY dfo_number, tq_id, server_type
ORDER BY skew_pct DESC;
```

- `skew_pct`가 크면 **HASH 스큐 확정**.

**확인 3) 대기**

ASH/Monitor에서
- `PX Deq Credit: send blkd`  
- `direct path write/read temp`  
가 함께 올라오면 **“스큐 → Consumer 과부하 → backpressure”** 시나리오가 완성된다. :contentReference[oaicite:15]{index=15}

### 6.3 B안: 소형 차원 브로드캐스트(재분배 제거)

```sql
SELECT /*+ leading(c) use_hash(f)
           parallel(f 16) parallel(c 16)
           pq_distribute(c BROADCAST NONE)
           monitor */
       COUNT(*)
FROM   fact_sales f
JOIN   dim_customer c ON c.cust_id=f.cust_id;
```

**기대 변화**
- c쪽 `PX SEND BROADCAST`  
- fact 쪽 **재분배량 감소**, TQ 스큐가 완화  
- send blkd 감소

### 6.4 C안: 스큐 완화용 SALT(키 분산 → 외부 재집계)

```sql
WITH salted AS (
  SELECT /*+ parallel(f 16) */
         cust_id,
         amount,
         MOD(ORA_HASH(cust_id), 8) salt
  FROM   fact_sales f
)
SELECT cust_id, SUM(amount) amt
FROM   salted
GROUP  BY cust_id;
```

- 내부적으로는 `(cust_id, salt)`로 해시 분산이 더 잘 되기 때문에  
  `V$PQ_TQSTAT`의 로우 편차가 줄어든다.

---

## 7. 병목 원인별 해결 전략 정리

### 7.1 `P->P + PX SEND HASH + skew_pct↑ + send blkd↑`

- **HASH 스큐**  
- **해결 순서**
  1) BROADCAST 가능한지 확인(소-대 조인일 때)  
  2) 불가하면 SALT/복합 키/UNION ALL 분해로 **hash 키 균형**  
  3) 그래도 TEMP가 문제면 PGA/Workarea 조정

### 7.2 `direct path write/read temp↑`

- **Workarea 스필**  
- **해결**
  - 입력을 줄이는 방향이 1순위  
    (파티션 프루닝, Top-N, 조인 필터)  
  - 그 다음에 PGA/Workarea, 마지막이 DOP

### 7.3 `IN-OUT=PCWP/PCWC`가 안 보이고 TQ가 많다

- 파이프가 **끊겨서 메시지 비용이 커진 상태**  
- **해결**
  - 불필요한 재분배 제거(Partition-wise/브로드캐스트)  
  - 조인 순서(LEADING)와 방식(USE_HASH/USE_MERGE)을 확정해  
    **Producer→Consumer 흐름을 단순화**

---

## 8. 최종 체크리스트(운영 런북)

1. **플랜에서 먼저 본다**
   - `IN-OUT`에서 **P->P 라인 개수**  
   - 각 P->P 라인의 **PX SEND 타입**  
   - `PX BLOCK ITERATOR` vs `PX PARTITION ...`(Granule 종류)  
2. **그 다음 V$PQ_TQSTAT로 스큐를 수치화**
3. **ASH/Monitor로 대기를 매칭**
   - send blkd ↔ Consumer 과부하/스큐  
   - direct path read/write temp ↔ Workarea spill  
4. **해결은 “입력↓ → 재분배↓ → 메모리↑ → DOP 조정” 순서**
5. **Partition-wise가 가능하면 최우선**
   - 같은 파티션 키/경계/개수 설계  
   - 로컬 인덱스 선호  
6. **소-대 조인은 기본적으로 BROADCAST 우선 검토**
7. **HASH-HASH는 스큐가 없는지 항상 의심하고 들어간다**

---

## 한 줄 결론

병렬 실행은  
**(1) Granule로 나누고, (2) PX SEND로 섞고, (3) IN-OUT으로 흐르며, (4) 그 결과가 대기 이벤트로 드러난다.**  
플랜의 `IN-OUT / PX SEND / TQ`와 `V$PQ_TQSTAT` 스큐만 정확히 읽으면,  
병렬 성능 문제의 대부분은 “원인-대응”이 즉시 정리된다.