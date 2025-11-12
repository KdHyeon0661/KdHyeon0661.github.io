---
layout: post
title: DB 심화 - Memory vs Disk I/O
date: 2025-11-04 16:25:23 +0900
category: DB 심화
---
# Memory vs Disk I/O — I/O 효율화 튜닝의 중요성·버퍼 캐시 히트율·네트워크 파일시스템 캐시 영향

> **핵심 요약**
> - **메모리 I/O(버퍼 캐시 히트)** 는 **나노~마이크로초** 수준, **디스크 I/O** 는 **밀리~수 ms+**. I/O를 **메모리로 승격**시키면 응답시간이 단계적으로 줄어든다.  
> - 하지만 “**Hit Ratio만 높이면 된다**”는 **오해**다. **직접 경로 읽기(Direct Path Read)**, **순차 스캔**은 **히트율을 낮추더라도** 더 빠를 수 있다.  
> - **네트워크 파일시스템(NFS/NAS, dNFS, ZFS/ARC 등)** 의 **OS/컨트롤러 캐시**는 “2차 캐시”로 작동한다. **중복 버퍼링**을 피하고(**direct I/O**·**dNFS**) **rsize/wsize**·**attribute cache**·**delegation**·**latency**를 튜닝해야 한다.

---

## 0. 기초 개념: 메모리 vs 디스크 I/O의 시간 규모

- 메모리(서버 RAM, SGA 버퍼 캐시): 수십~수백 ns ~ 수 μs  
- 로컬 SSD/NVMe: 수백 μs ~ 수 ms (Queue·Saturation·GC에 따라 변동)  
- SAN/NAS(네트워크 hop 포함): 수 ms ~ 수십 ms (Jumbo/MTU·RTT·컨트롤러 큐 영향)  

**응답시간 근사식**  
$$
\text{RT} \approx \text{CPU} + \text{Latch/Mutex Wait} + \text{Buffer Get Time} + \text{Physical I/O Time} + \text{Network}
$$

- **Buffer Get** 은 **버퍼 캐시 히트**일 때만 발생(매우 짧음).  
- **Physical I/O** 는 히트 실패(PHYSICAL READ)일 때만 발생(상대적으로 큼).  
- 따라서 **I/O 효율화**는 “**물리 읽기 자체를 줄이거나**(히트율↑/플랜 개선), **물리 읽기의 품질을 높이는 것**(순차/병렬/대역폭↑)” 두 축으로 본다.

---

## 1. I/O 효율화 튜닝의 중요성

### 1.1 왜 중요한가?
- OLTP의 많은 대기 이벤트 상위는 대체로 **`db file sequential read`(단일 블록)**, **`log file sync`**, **`buffer busy`**, **`read by other session`** 등 I/O 관여 항목이 차지.  
- DW/리포트는 **`direct path read temp`**, **`cell smart table scan`(Exadata)** 등 **대량 순차 I/O**가 좌우.

### 1.2 잘못된 목표 설정의 위험
- “**버퍼 캐시 히트율 99%**” 같은 목표는 **틀릴 수 있음**.  
  - 대량 보고 작업은 **Full Scan + Direct Path** 가 **더 빠를** 수 있다(히트율은 내려가도 전체 RT는 단축).  
  - 반대로 OLTP 랜덤 읽기는 **히트율 1~2%p**만 올려도 **체감**이 크다.

### 1.3 성능 접근법(OWI/Response Time)
- **대기 기반** 분석: 상위 대기 + 카운터(논리/물리 읽기)로 **가장 큰 RT 기여자**부터 제거.  
- **플랜 기반** 분석: **실행계획**과 **블록 접근 패턴**(Random vs Sequential)을 먼저 본다.  
- **데이터 기반**: **핫 세그먼트/핫 블록**과 **캐시 미스**를 숫자로 확인 후 **정책적**으로 조치(인덱스, 파티션, 클러스터링, 배열 페치 등).

---

## 2. 버퍼 캐시 히트율(Buffer Cache Hit Ratio, BCHR)

### 2.1 정의와 통상 계산식
- 용어:  
  - `db block gets`(CURRENT 모드 읽기)  
  - `consistent gets`(CONSISTENT 모드 읽기, Undo 기반 CR)  
  - `physical reads`(버퍼 캐시에 **없어서 디스크**에서 읽은 횟수)

**단순 BCHR**  
$$
\text{BCHR} = 1 - \frac{\text{physical reads}}{\text{db block gets} + \text{consistent gets}}
$$

> **주의**: Direct Path Read는 버퍼 캐시를 **우회**하므로, 위 분모/분자에의 해석이 맥락을 타며, **히트율만으로 결론을 내리면 오판**할 수 있다.

### 2.2 실전: 시스템 뷰로 산출
```sql
-- (세션/시스템 기준) 간단 BCHR
WITH s AS (
  SELECT
    SUM(CASE WHEN name='db block gets'    THEN value ELSE 0 END) db_block_gets,
    SUM(CASE WHEN name='consistent gets'  THEN value ELSE 0 END) consistent_gets,
    SUM(CASE WHEN name='physical reads'   THEN value ELSE 0 END) physical_reads
  FROM v$sysstat
)
SELECT 1 - (physical_reads / NULLIF(db_block_gets + consistent_gets, 0)) AS bchr
FROM s;
```

**특정 세그먼트/SQL 기반으로 보는 게 더 실용적**
```sql
-- 핫 세그먼트: 읽기·물리읽기 상위
SELECT owner, object_name, statistic_name, value
FROM   v$segment_statistics
WHERE  statistic_name IN ('logical reads','physical reads','physical reads direct')
ORDER  BY value DESC FETCH FIRST 20 ROWS ONLY;

-- SQL별 논리/물리 읽기
SELECT sql_id, plan_hash_value,
       buffer_gets, disk_reads, executions,
       ROUND(buffer_gets/NULLIF(executions,0)) avg_buf_gets,
       ROUND(disk_reads/NULLIF(executions,0))  avg_disk_reads
FROM   v$sql
ORDER  BY disk_reads DESC FETCH FIRST 30 ROWS ONLY;
```

### 2.3 해석 포인트
- **히트율↑** 자체보다, **핫 세그먼트·핫 SQL** 을 **줄이는 변화**(인덱스·파티션·플랜)가 핵심.  
- **Direct Path** 작업은 히트율을 낮추지만, **순차 대량 스캔**에선 전체 RT 단축이 목표.  
- **작은 워킹세트(OLTP 핫 블록)** 가 **L2 캐시**처럼 상주하도록 설계(인덱스 정합, 부분범위처리, 커버링 인덱스).

---

## 3. 튜닝 전략: “메모리 승격” + “물리 I/O 품질 개선”

### 3.1 메모리 승격(히트율↑) 전략
1) **워크로드 축소**:  
   - **부분범위처리(Stopkey)**, **Keyset 페이지**로 **읽을 양 자체를 줄임**  
   - **SELECT-LIST 최소화**, **중복 조회 제거**(조인/스칼라 서브쿼리 캐싱/RESULT_CACHE)
2) **플랜 변경**:  
   - 랜덤 I/O 많은 OLTP는 **인덱스 설계**(필터+정렬 복합/커버링)로 **테이블 BY ROWID 제거**  
   - **클러스터링 팩터 개선**(CTAS 재적재)로 Range Scan의 **준-순차화**
3) **캐시 정책**:  
   - **KEEP Pool**(자주 쓰는 작은 Lookup/Hot Segment)  
   - SGA/PGA(워크로드·메모리 여유·TEMP 사용량 기반) **적절히 증설**

**예: KEEP 풀에 핫 개체 고정**
```sql
-- 스키마 해상도에 따라 조정
ALTER TABLE dim_status STORAGE (BUFFER_POOL KEEP);
ALTER INDEX ix_dim_status STORAGE (BUFFER_POOL KEEP);
```

### 3.2 물리 I/O 품질 개선(순차/대역폭↑)
1) **해시 조인 + 파티션 프루닝/블룸**으로 **순차 대량 I/O**  
2) **병렬도** 조정(PX) — 스토리지/네트워크 대역폭과 균형  
3) **Direct Path Read**(대량 읽기): **버퍼 캐시 오염 방지**, 소트/집계/스캔 성능↑  
4) **파일 레이아웃/ASM 스트라이핑**: LUN 분산, IOPS/MB/s 상승

**예: 대량 보고 쿼리 힌트**
```sql
SELECT /*+ parallel(o 8) full(o) use_hash(o) */ ...
FROM   orders o
WHERE  order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -3);
```

---

## 4. 네트워크 파일시스템(NFS/NAS) 캐시가 I/O 효율에 미치는 영향

### 4.1 레이어별 캐시
- **Oracle SGA 버퍼 캐시**: 데이터 블록 캐시(1차)  
- **OS 페이지 캐시**: 파일 시스템 읽기 결과 캐시(2차) — NFS 클라이언트 측  
- **NAS 컨트롤러 캐시 / ZFS ARC**: 스토리지 어플라이언스 캐시(3차)  
- **중복 버퍼링(Double Buffering)** 위험: 동일 블록이 **SGA**와 **OS/NAS**에 **이중 캐시** → 메모리 낭비·일부 워크로드에 비효율

### 4.2 Oracle Direct NFS(dNFS)
- Oracle 프로세스가 **유저스페이스에서 직접 NFS I/O** 를 관리(커널 NFS 경유 X)  
- **장점**: 더 큰 I/O 사이즈, 더 적은 컨텍스트 스위칭, 다중 세션/파이프라인, **캐싱·Lock 처리 최적화**  
- 구성 파일: `$ORACLE_HOME/dbs/oranfstab`

**샘플 `oranfstab`**
```ini
server: nas01
local:  10.10.1.101
path:   10.10.1.201
export: /vol/oradata mount: /u02/oradata
nfs_version: nfs3
dontroute: true
mnt_timeout: 30
```

**활성화 확인**
```sql
SELECT * FROM v$dnfs_servers;
SELECT * FROM v$dnfs_channels;
```

### 4.3 커널 NFS 사용 시 권장 옵션(예시)
- **rsize/wsize**: I/O 크기(예: 1M) 확대로 **순차 대역폭↑**  
- **hard,timeo,retrans**: 안정성  
- **noatime**: 메타데이터 업데이트 최소화  
- **actimeo**(attribute cache): 메타데이터 캐시 유지시간 — 너무 길면 변경 감지 지연  
- **proto,port,mtu/jumbo**: 네트워크 튜닝(RTT↓, 재전송↓)

**리눅스 마운트 예**
```bash
mount -t nfs -o vers=3,proto=tcp,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noatime \
  nas01:/vol/oradata /u02/oradata
```

> **포인트**: 큰 연속 I/O는 **rsize/wsize↑** 로 이득. 작은 랜덤 I/O는 **RTT**가 지배하므로 **네트워크 지연**이 핵심.

### 4.4 ZFS/ARC, NAS 컨트롤러 캐시와의 상호작용
- **ZFS ARC**(메모리 캐시) + **L2ARC**(SSD 캐시) → **자주 읽는 블록**의 **네트워크 왕복** 감소  
- 하지만 **SGA 버퍼 캐시**와 **중복** 될 수 있으니, **DB 버퍼 캐시를 과도 확장**하기보다 **스토리지 캐시**와 **균형**을 잡는다.  
- **Direct I/O**(파일시스템 캐시 우회) vs **Buffered I/O**(OS 캐시 사용) **실측 비교**가 중요.

---

## 5. 측정과 진단: “히트율 숫자”보다 “어디서 낭비되는가”

### 5.1 상위 대기·I/O 프로파일
```sql
-- AWR Top Timed Events / ASH Top Events (리포트로 확인)
-- SQL별 I/O 프로파일
SELECT sql_id, plan_hash_value, executions,
       buffer_gets, disk_reads,
       ROUND(disk_reads/NULLIF(executions,0))  avg_phyr,
       ROUND(buffer_gets/NULLIF(executions,0)) avg_lgr
FROM   v$sql
ORDER  BY avg_phyr DESC FETCH FIRST 50 ROWS ONLY;
```

### 5.2 세그먼트/파일 Hot Spot
```sql
-- 물리 읽기/쓰기 상위 세그먼트
SELECT owner, object_name, statistic_name, value
FROM   v$segment_statistics
WHERE  statistic_name IN ('physical reads','physical writes','buffer busy waits')
ORDER  BY value DESC FETCH FIRST 20 ROWS ONLY;

-- 파일별 I/O
SELECT file#, phyrds, phywrts, readtim, writetim
FROM   v$filestat ORDER BY phyrds DESC;
```

### 5.3 Direct Path Read/Temp 확인
```sql
SELECT name, value
FROM   v$sysstat
WHERE  name LIKE 'physical reads direct%';  -- direct path read, direct temp
```

### 5.4 OS·스토리지 관측(샘플)
```bash
# iostat -x 1
# nfsiostat 1
# sar -n DEV 1
# nstat -z | grep -i retrans
```

---

## 6. 시나리오별 실전 처방

### 6.1 OLTP: 랜덤 I/O가 상위(단일 블록)
**증상**: `db file sequential read` 상위, SQL은 **NL + BY ROWID** 반복, 인덱스 부적합  
**처방**  
- **인덱스 재설계**(필터+정렬 복합, 커버링), **클러스터링 팩터 개선**  
- **부분범위처리**(리스트 화면) + **Keyset 페이지**  
- **KEEP Pool**: 작은 Lookup 상주  
- **배열 페치/배치 처리**로 **왕복↓**

**예**
```sql
-- 고객별 최근 50건: 정렬 포함 복합 인덱스 + Stopkey
CREATE INDEX ix_o_cust_dt ON orders(cust_id, order_dt DESC, order_id DESC, amount);

SELECT /*+ index(o ix_o_cust_dt) */
       order_id, order_dt, amount
FROM   orders o
WHERE  cust_id=:cust
ORDER  BY order_dt DESC, order_id DESC
FETCH FIRST 50 ROWS ONLY;
```

### 6.2 DW/리포트: 대량 순차 I/O가 상위
**증상**: `direct path read temp`, `cell smart table scan`(Exadata), 정렬/집계 비중 큼  
**처방**  
- **해시 조인 + 파티션 프루닝/블룸**으로 작은 전체만 순차 스캔  
- **병렬도**와 **I/O 사이즈** 최적화, **Jumbo/MTU**·**rsize/wsize** 확대  
- **dNFS/ASM** 으로 파이프라인·스트라이핑 최적화

**예**
```sql
SELECT /*+ full(o) use_hash(o) parallel(o 8) */ cust_id, SUM(amount)
FROM   orders o
WHERE  order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -6)
GROUP  BY cust_id;
```

### 6.3 NFS/NAS에서 지연이 높은 경우
**증상**: 평균 RT 대비 tail latency 높음, 재전송, 작은 랜덤 I/O 다발  
**처방**  
- **dNFS 활성화** 또는 **rsize/wsize↑**  
- **Keyset 페이지**로 작은 랜덤 읽기 빈도 자체를 낮춤  
- **네트워크 튜닝**(RTT↓, Jumbo, 큐 길이), **속도/듀플렉스 고정**  
- **스토리지 캐시**(ARC/L2ARC) 사이징·Pin set 검토

---

## 7. “히트율 교조주의”에 대한 반례와 균형

- **반례 1**: 1TB 보고 쿼리, 히트율 60% → 해시 조인 + Direct Path로 **히트율 30%**가 되었지만 **RT 40% 단축**.  
- **반례 2**: OLTP 단건 조회, 히트율 99.5% → 인덱스 커버링으로 **99.3%**가 되었지만 **RT 20% 단축**(BY ROWID 제거).  
- **결론**: 히트율은 **결과 지표**일 뿐 **목표 그 자체가 아니다**. **RT/스루풋** 중심으로 판단.

---

## 8. 실습: “히트율·I/O·계획” 전·후 비교 루틴

```sql
ALTER SESSION SET statistics_level=ALL;
ALTER SESSION SET events '10046 trace name context forever, level 8';

-- BEFORE: 기존 쿼리 실행
-- AFTER : 인덱스/플랜/페이지/병렬/dNFS 등 적용 후 동일 페이로드 실행

ALTER SESSION SET events '10046 trace name context off';

-- SQL Monitor / DBMS_XPLAN.DISPLAY_CURSOR 로
-- buffer_gets, disk_reads, direct path reads, timed statistics, STOPKEY, FULL/PX 여부 확인
```

---

## 9. 네트워크 파일시스템 구성 체크리스트

- [ ] **dNFS** 사용 가능한가? (`v$dnfs_servers`/`v$dnfs_channels` 확인)  
- [ ] 커널 NFS라면 **rsize/wsize**, **hard/timeo/retrans**, **noatime**, **actimeo** 적정?  
- [ ] **Jumbo MTU**(9k)·스위치 큐·ECN/RED 조정  
- [ ] **스토리지 캐시**(ARC/L2ARC/컨트롤러)와 **SGA 버퍼 캐시** **균형**  
- [ ] **혼합 워크로드**에서 **Double Buffering** 과다 여부 확인(Direct I/O 고려)  
- [ ] PX·HASH·Full Scan 경로에서 **순차 대역폭**이 목표대로 나오는지 `iostat`·`nfsiostat` 확인

---

## 10. 요약 처방전

1) **I/O 효율화의 1원칙**: **읽지 않는 것이 최고** — 부분범위처리, 커버링 인덱스, 중복 연산 제거.  
2) **2원칙**: **읽어야 한다면 잘 읽자** — 해시 조인 + 프루닝/블룸, 순차/병렬, Direct Path.  
3) **히트율은 수단**: OLTP는 워킹세트 상주(히트율↑), DW는 순차/대역폭 극대화(히트율↓라도 OK).  
4) **NFS/NAS**: dNFS/마운트 옵션/스토리지 캐시/네트워크 튜닝으로 **RTT·대역폭** 최적화.  
5) **측정으로 검증**: AWR/ASH/SQL Monitor/TKPROF 로 전·후 수치 비교.

---

## 부록 A: 샘플 스크립트 모음

**A.1 BCHR와 Direct Path 지표 함께 보기**
```sql
WITH s AS (
  SELECT name, value FROM v$sysstat
  WHERE  name IN ('db block gets','consistent gets','physical reads',
                  'physical reads direct','physical reads direct temporary')
)
SELECT
  (SELECT value FROM s WHERE name='db block gets')       AS db_block_gets,
  (SELECT value FROM s WHERE name='consistent gets')     AS consistent_gets,
  (SELECT value FROM s WHERE name='physical reads')      AS physical_reads,
  (SELECT value FROM s WHERE name='physical reads direct') AS direct_reads,
  (SELECT value FROM s WHERE name='physical reads direct temporary') AS direct_temp,
  1 - (
    (SELECT value FROM s WHERE name='physical reads') /
    NULLIF(
      (SELECT value FROM s WHERE name='db block gets') +
      (SELECT value FROM s WHERE name='consistent gets'), 0
    )
  ) AS bchr
FROM dual;
```

**A.2 핫 세그먼트 Top-N**
```sql
SELECT owner, object_name, statistic_name, value
FROM   v$segment_statistics
WHERE  statistic_name IN ('logical reads','physical reads','buffer busy waits')
ORDER  BY value DESC FETCH FIRST 30 ROWS ONLY;
```

**A.3 SQL별 평균 물리/논리 읽기**
```sql
SELECT sql_id, plan_hash_value, executions,
       ROUND(buffer_gets/NULLIF(executions,0)) avg_buf_gets,
       ROUND(disk_reads/NULLIF(executions,0))  avg_disk_reads
FROM   v$sql
WHERE  executions > 0
ORDER  BY avg_disk_reads DESC FETCH FIRST 50 ROWS ONLY;
```

**A.4 dNFS 확인**
```sql
SELECT svrname, local, path, mnt, nfsversion FROM v$dnfs_servers;
SELECT * FROM v$dnfs_channels;
```

**A.5 Keyset 페이지(Stopkey) 예**
```sql
SELECT /*+ index(o ix_o_cust_dt) */
       order_id, order_dt, amount
FROM   orders o
WHERE  cust_id=:cust
ORDER  BY order_dt DESC, order_id DESC
FETCH FIRST :take ROWS ONLY;
```

---

## 결론

- **Memory vs Disk I/O** 는 “속도 차”가 아닌 “**설계의 방향**” 문제다.  
- **OLTP** 는 작은 워킹세트를 **메모리에 붙여두고(히트율↑)**, **랜덤 I/O** 를 최소화한다.  
- **DW/리포트** 는 **순차 대역폭**과 **Direct Path** 를 극대화한다(히트율 집착 금지).  
- **NFS/NAS** 환경에서는 **dNFS/마운트 옵션/스토리지 캐시/네트워크**의 **캐시·대역폭·지연**을 함께 조율하라.  
- 모든 변화는 **AWR/ASH/SQL Monitor/TKPROF** 로 **숫자**로 확인하고, “히트율”이 아니라 **응답시간·처리량**의 개선을 목표로 삼아라.