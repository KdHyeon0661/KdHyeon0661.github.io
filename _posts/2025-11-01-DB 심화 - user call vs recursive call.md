---
layout: post
title: DB 심화 - user call vs recursive call
date: 2025-11-01 17:25:23 +0900
category: DB 심화
---
# user call vs recursive call

> **핵심 한 줄**
> - **user call**: **클라이언트(사용자/애플리케이션)**가 서버에 던지는 OCI/드라이버 호출(파싱·실행·페치 등)
> - **recursive call**: 사용자의 SQL을 처리하는 **오라클 내부 동작**(사전 조회, 공간 관리, 시퀀스/제약/뷰 확장, 통계/플랜 관련 등)을 위해 **오라클이 자체적으로 실행하는 추가 SQL 호출**

두 값의 **의미·발생 원인·증상**을 정확히 이해하면,
- “왜 간단한 SELECT가 느리지?”
- “왜 AWR에 Recursive CPU가 높지?”
- “왜 Parse는 적은데 Disk I/O가 많지?”
같은 질문에 **한 번에** 답을 찾을 수 있습니다.

---

## 정의: user call vs recursive call

### user call (사용자 호출)

- **무엇?** 클라이언트가 서버로 보내는 **OCI/드라이버 레벨 호출** 수.
  - 예) `OCIPrepare`, `OCIExecute`, `OCIDefineByPos`/Fetch, Logon/Logoff, Describe 등
- **대표 지표**
  - `V$SYSSTAT / V$SESSTAT`: `user calls`, `bytes received via SQL*Net from client`, `bytes sent via SQL*Net to client`
- **주 용도**
  - 애플리케이션의 **채팅성 왕복**(chattiness) 진단: 너무 잦은 소량 페치, 불필요한 prepare/execute, 커밋 남발

### recursive call (재귀 호출)

- **무엇?** 한 문장을 처리하는 과정에서 **오라클 엔진 내부가 추가로 실행한 SQL** 호출.
- **왜 필요? (대표 사례)**
  1) **데이터 딕셔너리/메타데이터 조회**: Synonym/권한/뷰 확장/파티션 정보 확인
  2) **세그먼트/공간 관리**: 익스텐트 할당, 헤더 갱신(LMT/ASSM에서도 내부적으로)
  3) **시퀀스**: `SEQ.NEXTVAL` 소비/캐시 보충(내부 `SEQ$` 액세스)
  4) **제약/트리거/물질화 뷰 로그**: CHECK/FOREIGN KEY 검증, 트리거 내부 SQL
  5) **통계/플랜 관련**: 동적 샘플링, Outline/SPM, Rule 평가 등(버전·옵션 의존)
  6) **DDL/DDL-유사**: DDL은 대부분 사전 변경을 수반 → 내부적으로 다수의 재귀 SQL 발생
- **대표 지표**
  - `V$SYSSTAT/V$SESSTAT`: `recursive calls`, `recursive cpu usage`, `recursive elapsed time`(버전에 따라 명칭 차)

> **요약**
> - **user call**은 “고객이 벨을 누른 횟수”,
> - **recursive call**은 “직원이 내부적으로 처리한 일(백오피스) 횟수”.

---

## 실습을 위한 준비 스키마

```sql
-- 실험 테이블
DROP TABLE demo_call PURGE;
CREATE TABLE demo_call (
  id        NUMBER PRIMARY KEY,
  c_region  VARCHAR2(10),
  c_status  VARCHAR2(10),
  amt       NUMBER(12,2),
  dte       DATE
);

BEGIN
  INSERT /*+ APPEND */ INTO demo_call
  SELECT level,
         CASE MOD(level,4) WHEN 0 THEN 'APAC' WHEN 1 THEN 'EMEA' WHEN 2 THEN 'AMER' ELSE 'OTHER' END,
         CASE MOD(level,5) WHEN 0 THEN 'X' ELSE 'OK' END,
         ROUND(DBMS_RANDOM.VALUE(10,1000),2),
         TRUNC(SYSDATE) - MOD(level, 365)
  FROM dual CONNECT BY level <= 500000;
  COMMIT;
END;
/

CREATE INDEX ix_demo_call_r_d ON demo_call(c_region, dte);
BEGIN DBMS_STATS.GATHER_TABLE_STATS(USER,'DEMO_CALL', cascade=>TRUE); END;
/

-- 시퀀스 + 트리거(재귀 호출 데모)
DROP SEQUENCE demo_seq;
CREATE SEQUENCE demo_seq CACHE 100;

CREATE OR REPLACE TRIGGER demo_call_bi
BEFORE INSERT ON demo_call
FOR EACH ROW
BEGIN
  IF :NEW.id IS NULL THEN
    :NEW.id := demo_seq.NEXTVAL; -- 내부적으로 SEQ$ 액세스(재귀)
  END IF;
END;
/
```

---

## 빠른 관측 도구: 세션 통계 스냅샷 함수

```sql
-- 간단 스냅샷: 특정 통계만 추출
WITH s AS (
  SELECT sid, sn.name, ss.value
  FROM   v$sesstat ss
  JOIN   v$statname sn ON sn.statistic#=ss.statistic#
  WHERE  ss.sid = SYS_CONTEXT('USERENV','SID')
  AND    sn.name IN ('user calls','recursive calls','recursive cpu usage',
                     'parse count (total)','parse count (hard)',
                     'bytes sent via SQL*Net to client',
                     'bytes received via SQL*Net from client')
)
SELECT * FROM s;
```

> **팁**: 실행 전/후를 각각 찍어 **델타**를 보십시오.
> `user calls`와 `recursive calls`의 **상대적 증가폭**이 무엇을 의미하는지 금방 체감됩니다.

---

## TKPROF/SQL Trace에서 보이는 “User vs Recursive”

### 트레이스 켜기/끄기

```sql
ALTER SESSION SET statistics_level=ALL;
ALTER SESSION SET events '10046 trace name context forever, level 8';
-- ... 실험 SQL ...
ALTER SESSION SET events '10046 trace name context off';
```

### TKPROF 결과(예시)

TKPROF 상단에 종종 다음과 같은 **요약**이 나옵니다:

```
Misses in library cache during parse: 0
Optimizer mode: ALL_ROWS
Parsing user id: 107
Number of plan statistics captured: 1

Recursive calls: 42
user  calls: 23
```

- **user calls**: 클라이언트에서 온 호출 수
- **Recursive calls**: 위 SQL(들)을 처리하며 내부적으로 발생한 추가 호출 수

> **주요 관찰 포인트**
> - **Recursive calls**가 비정상적으로 크면:
>   - DDL/사전 액세스 과다?
>   - 시퀀스 캐시 너무 작아 잦은 보충?
>   - 트리거/제약 내부 SQL 빈번?
>   - 공간 관리(익스텐트 할당) 이슈?
> - **user calls**가 과다하면:
>   - Fetch Array Size 너무 작아 왕복 과다?
>   - 같은 문장 준비/실행 반복(Statement Cache 없음)?
>   - 커밋 남발?

---

## 케이스 스터디 — 원인별 재현과 해석

### 케이스 A: **단순 SELECT** (user call 중심)

```sql
VAR r VARCHAR2(10); EXEC :r := 'APAC';

-- 배열 페치 크게(왕복 최소화)
SELECT /* A:user */ COUNT(*)
FROM   demo_call
WHERE  c_region = :r
AND    dte >= TRUNC(SYSDATE)-30
AND    dte <  TRUNC(SYSDATE);
```

- **기대**:
  - `user calls`는 **Parse 1 + Execute 1 + Fetch n회**(n은 페치 배열 크기에 의존)
  - `recursive calls`는 **낮음**(딕셔너리 캐시가 워밍업된 상태라면 거의 0~소수)

- **페치 사이즈를 일부러 작게** 하면? `user calls` 크게 증가(왕복 증가)
  - 해결: JDBC `setFetchSize`, ODP.NET `FetchSize`, Python `cursor.arraysize` 확대로 감소

### 케이스 B: **INSERT with Sequence + Trigger** (recursive call 상승)

```sql
BEGIN
  FOR i IN 1..5000 LOOP
    INSERT INTO demo_call(id, c_region, c_status, amt, dte)
    VALUES (NULL, 'APAC', 'OK', 100, SYSDATE);
    -- demo_call_bi 트리거에서 demo_seq.NEXTVAL 호출 → 내부 SEQ$ 접근 → 재귀 호출 발생
  END LOOP;
  COMMIT;
END;
/
```

- **기대**:
  - `user calls`는 루프 횟수/배치 정책에 좌우(위 예제는 루프당 1 Execute)
  - **`recursive calls` 증가**: 시퀀스 캐시 보충 시 SEQ$ 접근, 트리거/제약 확인 등

- **튜닝 포인트**
  - 시퀀스 `CACHE` 사이즈 충분히 크게(예: `CACHE 1000`) → 캐시 보충 빈도↓ → 재귀 호출↓
  - 벌크 바인드/배치 INSERT 사용(PL/SQL FORALL, JDBC batch)

### 케이스 C: **DDL/메타데이터 접근** (recursive call 급증)

```sql
-- 인덱스 생성/삭제 같은 DDL은 내부적으로 딕셔너리 테이블 다수 갱신
CREATE INDEX ix_demo_call_status ON demo_call(c_status);

-- 동적으로 자주 DDL을 발행하는 애플리케이션 패턴일수록 재귀 호출 급증
```

- **기대**: `recursive calls` 급증, `recursive cpu usage`도 상승
- **Bad Pattern**: 런타임에 객체 생성/삭제 반복(임시 테이블 대신 DDL), 매 요청마다 권한/동의어를 재검증하게 만드는 설계
- **대응**:
  - **DDL은 배포·관리 시간에만**, 런타임은 DML 중심
  - **전용 GTT/임시 테이블** 사용(DDL 대체)
  - **딕셔너리 캐시(Shared Pool) 적정화**: 캐시 미스로 매번 메타데이터 조회하지 않도록

### 케이스 D: **공간(Extent) 관리** (대량 DML/세그먼트 확장 시)

```sql
-- ASSM/LMT라도 대량 INSERT로 익스텐트 확장 → 내부 재귀
-- 재현은 환경/여유공간에 따라 다르지만, 대량 APPEND/병렬 부하에선 관측 쉬움
INSERT /*+ APPEND */ INTO demo_call
SELECT * FROM demo_call WHERE ROWNUM <= 100000;
COMMIT;
```

- **기대**: **세그먼트 확장**에 따른 내부 딕셔너리/헤더 갱신 → `recursive calls` 증가
- **대응**:
  - **적절한 익스텐트 크기/autoallocate**, 대량 적재는 **사전 공간 확보**
  - 잦은 작은 익스텐트 확장 방지

### 케이스 E: **뷰/동의어/권한** (딕셔너리 접근 경로)

```sql
-- Public Synonym 경유, Grant/Role 복잡
CREATE OR REPLACE VIEW v_demo AS
SELECT * FROM demo_call WHERE c_status='OK';

SELECT COUNT(*) FROM v_demo WHERE c_region='APAC';
```

- **기대**: 초반엔 뷰 확장/동의어 해석/권한 확인으로 **재귀 SQL**이 몇 번 등장(캐시 후 감소)
- **대응**:
  - **딕셔너리 캐시 히트율** 유지(Shared Pool), 불필요한 뷰 중첩/동적 권한 부여 최소화

---

## AWR/ASH에서 어떻게 보이나?

### AWR Report (DB Time/AAS 하위)

- **“Load Profile”**: `User calls per sec`
- **“Instance Efficiency Percentages”**: Dictionary/Library Cache 관련 히트율
- **“SQL ordered by … Recursive CPU Time”**: **재귀 SQL** 중 CPU 소비가 큰 것 나열
  - 여기서 나오는 SQL은 사용자가 직접 던진 게 아니라 **내부 재귀 SQL**일 수 있음

### ASH

```sql
-- 최근 15분간 'recursive' 관련 상위 SQL (내부 SQL 포함)
SELECT sql_id, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time > SYSTIMESTAMP - INTERVAL '15' MINUTE
GROUP  BY sql_id
ORDER  BY samples DESC FETCH FIRST 20 ROWS ONLY;
```

- 내부 재귀 SQL도 **sql_id**로 관측되며, **SQL Text**를 보면 `obj$`, `tab$`, `seg$`, `ind$`, `seq$` 등 **사전/시스템 오브젝트**가 보이기도 합니다.

---

## 원인→해법 맵 (Checklist)

| 증상/지표 | 추정 원인 | 처방 |
|---|---|---|
| **user calls↑** + 페치 많음 | Fetch Array Size 작음, 과도한 왕복 | **배열/페치 사이즈 확대**, 결과 페이지네이션 설계 |
| **user calls↑** + parse↑ | 커서 재사용 안 함 | 바인드 변수, Statement Cache(드라이버), `SESSION_CACHED_CURSORS` |
| **recursive calls↑** + DML | 시퀀스 캐시 작은/트리거 내부 SQL | **Sequence CACHE 확대**, 트리거 로직 슬림화/배치화 |
| **recursive calls↑** + DDL | 런타임 DDL/권한/동의어 탓 | DDL은 배포 시간에, 런타임은 **GTT/파라미터**로 대체 |
| **recursive calls↑** + 공간 확장 | 작은 익스텐트 반복 | 사전 **공간 확보/적절한 익스텐트 정책**, APPEND/병렬 전략 점검 |
| **recursive cpu usage↑** | 내부 재귀 SQL이 CPU 소모 | 상위 재귀 SQL 식별(AWR)→원인별(시퀀스/사전/공간/트리거) 조치 |
| 딕셔너리 캐시 미스↑ | shared pool/라이브러리 캐시 부족 | 메모리 적정화, SQL 템플릿 고정, 바인드로 child 폭증 억제 |

---

## 드라이버 관점 튜닝(사용자 호출 줄이기)

### JDBC

```java
try (PreparedStatement ps = conn.prepareStatement(SQL)) {
  ps.setFetchSize(1000);        // 페치 왕복 감소 → user calls 감소
  ps.setQueryTimeout(30);
  try (ResultSet rs = ps.executeQuery()) {
    while (rs.next()) { /* ... */ }
  }
}
```
- **Statement Cache** 활성(Oracle JDBC implicit/explicit), 바인드 사용 → **Parse user calls** 감소

### ODP.NET

```csharp
cmd.FetchSize = 8 * 1024 * 1024;  // 8MB 등
cmd.ArrayBindCount = ids.Length;   // 벌크 바인드로 execute 호출 수 절감
```

### Python (oracledb)

```python
cur.arraysize = 1000           # fetch 왕복 감소
cur.executemany(sql, params)   # 벌크 DML → user calls 감소
```

---

## “숫자로 보는” 판단 기준

간단한 비율/휴리스틱으로도 상황을 가늠할 수 있습니다.

- **채팅성 왕복 지표**
  $$ \text{Rows per User Call} = \frac{\text{총 페치 Rows}}{\text{user calls}} $$
  값이 **작을수록** 왕복이 많음 → **배열/페치 사이즈 확대** 검토

- **재귀 과다 의심**
  $$ \text{Recursive/User Ratio} = \frac{\text{recursive calls}}{\text{user calls}} $$
  특정 배치/트랜잭션에서 평소 대비 **유의미하게 상승**하면 원인 파악(시퀀스·DDL·공간·사전)

- **파싱 비중**
  $$ \text{Parse per Exec} = \frac{\text{parse count (total)}}{\text{execute count}} $$
  **1에 근접/초과**하면 재사용 부족

> 수식은 **정확한 SLA**라기보다 **현상 추정**용입니다. 반드시 TKPROF/AWR과 교차 검증하세요.

---

## End-to-End 실습: 전/후 비교

### Before: 나쁜 패턴

- SELECT를 **작은 fetchSize(=10)** 로 수천 번 호출
- 트리거 + 시퀀스 `CACHE 20` 상태에서 대량 INSERT

**관찰(예시)**
- `user calls`: 매우 높음(수천~수만)
- `recursive calls`: INSERT 구간에서 눈에 띄게 상승(시퀀스 보충/트리거 내부 SQL)

### After: 개선

- fetchSize=1000, 배열 바인드/벌크 DML, 시퀀스 `CACHE 1000`

**관찰(예시)**
- `user calls`: **10~100배 감소**(왕복 축소)
- `recursive calls`: **시퀀스 관련 급감**, 전체 경과 시간도 동반 감소

---

## 자주 묻는 질문(FAQ)

**Q1. recursive call은 나쁜 건가요?**
A. **필수**입니다. 오라클이 일을 하려면 내부 질의가 필요합니다. “높다”가 문제라기보다, **평시 기준 대비 비정상적 증가**가 문제입니다.

**Q2. 왜 간단한 SELECT에 recursive call이 찍히죠?**
A. 초기에는 **뷰 확장/동의어/권한/파티션 메타** 확인 때문에 딕셔너리 접근이 필요합니다. 캐시가 워밍업되면 줄어듭니다.

**Q3. AWR의 “SQL ordered by Recursive CPU Time”에 이상한 SQL이 떠요.**
A. **내부 재귀 SQL**입니다. Text/OBJECT 명(`obj$`, `seg$`, `ind$`, `seq$`)을 보고 **원인 계열**(사전/공간/시퀀스)을 분류하세요.

**Q4. user calls를 줄이려면?**
A. **배열/페치 사이즈** 확대, **Statement/Session Cursor Cache** 활성, **배치/벌크** 사용, **불필요한 커밋/핑퐁 제거**.

**Q5. recursive calls를 줄이려면?**
A. 사용 패턴에 따라: **시퀀스 CACHE 증가**, **DDL 런타임 금지**, **공간 사전 확보**, **딕셔너리/라이브러리 캐시 히트율 유지**, **트리거 슬림화**.

---

## 최종 체크리스트

- [ ] **user call** 모니터링: 왕복/배열/배치/커밋 빈도
- [ ] **recursive call** 모니터링: 시퀀스/DDL/공간/사전/트리거
- [ ] TKPROF의 **CALL 표** + AWR의 **Recursive CPU/Calls** 교차 확인
- [ ] 드라이버 **Statement Cache** + 서버 `SESSION_CACHED_CURSORS`
- [ ] **바인드 변수 전면 사용**(커서 공유·파싱·딕셔너리 접근 감소)
- [ ] **시퀀스 CACHE/NOORDER(필요 시)**, **ASSM/LMT**와 공간 전략
- [ ] **DDL은 배포/운영창**에서만, 런타임은 GTT/파라미터로

---

## 결론

- **user call**은 “클라이언트↔서버 왕복 양”, **recursive call**은 “서버 내부 처리 양”입니다.
- 느린 시스템은 대개 **둘 중 하나**가 비정상:
  - **user call 과다**(채팅성 페치/파싱/커밋)
  - **recursive call 과다**(시퀀스/DDL/공간/사전/트리거)
- **정량 지표 + 원인별 패턴**으로 접근하면 해법은 명확합니다.
  **배열/배치/바인드/캐시/공간/시퀀스**—이 여섯 축만 잘 잡아도 대부분의 병목을 빠르게 제거할 수 있습니다.

> **한 줄 요약**
> user call은 **밖**에서 온 호출, recursive call은 **안**에서의 추가 일.
> 숫자를 보고 **왕복을 줄이고(배열/배치)**, **내부 일을 덜 만들면(시퀀스/DDL/공간 최적화)** 시스템은 즉시 빨라진다.
