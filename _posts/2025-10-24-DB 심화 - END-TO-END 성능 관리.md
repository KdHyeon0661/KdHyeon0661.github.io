---
layout: post
title: DB 심화 - END-TO-END 성능 관리
date: 2025-10-24 23:25:23 +0900
category: DB 심화
---
# END-TO-END 성능 관리(End-to-End Performance Management)

## 0. 큰 그림: E2E 성능 관리가 **왜** 어렵나?
- 성능은 **단일 지점**이 아니라 **연쇄 경로**의 함수:  
  **Client(브라우저/앱)** → **CDN/Edge** → **API Gateway/LB** → **Microservices** → **DB/Cache/Queue** → **Storage**.  
- 한 구간의 지연이 전체를 지배(병목), 또한 **부하·분산·키 분포**에 따라 병목 위치가 바뀐다.  
- 그래서 **E2E**는 “**관측(Observability)** → **분해(Response Time Analysis)** → **상관(Trace Correlation)** → **처방”**의 **사이클**이 핵심이다.

---

## 1. 성능 목표 설계: **SLI/SLO/SLA** + **성능 예산**

### 1.1 SLI/SLO 정의
- **SLI(Service Level Indicator)**: 측정 가능한 품질 지표(예: `/search` **p95 응답시간**, 성공률).  
- **SLO(Service Level Objective)**: 목표(예: p95 ≤ **800 ms**, 가용성 ≥ **99.9%**).  
- **SLA(Agreement)**: 계약상 보장(벌점/크레딧 포함).

**지표 예시(엔드포인트별)**
| 경로 | SLI | SLO(예) |
|---|---|---|
| `GET /product/{id}` | p95 응답 | ≤ 400 ms |
| `POST /order` | p99 응답 | ≤ 900 ms |
| 전체 API | 오류율 | ≤ 0.1% |
| DB 호출 | 평균/퍼센타일 | p95 ≤ 50 ms / 호출 |

### 1.2 성능 예산(Performance Budget)
프론트/백엔드/DB를 **할당**:
- **총 SLO**: p95 **800 ms**  
  - 네트워크 왕복 + TLS: **120 ms**  
  - API 게이트웨이/인증: **80 ms**  
  - **서비스** 로직: **250 ms**  
  - **DB**(쿼리+커밋): **220 ms**  
  - 캐시/외부 API: **80 ms**  
  - 여유 버퍼: **50 ms**

> 모든 PR/배포는 **예산 초과 여부**를 자동 검사(성능 회귀 방지).

---

## 2. 이론 감각(필수 최소치)

### 2.1 리틀의 법칙
$$ L = \lambda \times W $$
- \(L\): 시스템 내 **평균 작업 수**(= 평균 활성 세션·큐 길이)  
- \(\lambda\): 처리율(요청/초)  
- \(W\): 평균 응답시간  
→ **활성 세션(AAS)** = 처리율 × 응답시간, 부하 상승 시 **대기 급증** 지점 관찰.

### 2.2 응답시간 분해(전체)
$$ T_{\text{E2E}} = T_{\text{client}} + T_{\text{net}} + T_{\text{gateway}} + \sum T_{\text{service}_i} + T_{\text{cache}} + T_{\text{db}} + T_{\text{ext}} $$
**DB 내부**는
$$ T_{\text{db}} = T_{\text{CPU}} + \sum_k T_{\text{wait},k} $$

### 2.3 Golden Signals/RED/USE
- **Golden Signals**: Latency, Traffic, Errors, Saturation  
- **RED**: Rate, Errors, Duration(서비스 호출 관점)  
- **USE**: Utilization, Saturation, Errors(리소스 관점: CPU/디스크/네트워크)

---

## 3. 계측 전략: **Trace/Metric/Log** 삼각 관측

### 3.1 상관키(Correlation ID) 설계
- **W3C Trace Context**: `traceparent`, `tracestate` 헤더를 전 구간 전파  
- **업무 태그**: `tenant_id`, `user_id`, `order_id` 등(PII 주의)  
- **Oracle 연결 태깅**: `DBMS_APPLICATION_INFO`로 모듈/액션/클라이언트 식별자 설정

**Oracle(세션 태깅)**
```sql
BEGIN
  DBMS_APPLICATION_INFO.SET_MODULE(module_name => 'WEB-API', action_name => 'GET /product/{id}');
  DBMS_SESSION.SET_IDENTIFIER('client:trace-6f8a...'); -- client_identifier
END;
/
```
> 이후 AWR/ASH/V$SESSION에서 **service/module/action/client_id** 차원으로 분석 가능.

### 3.2 애플리케이션(예: Java/Spring)에서 Trace 전파
```java
// build.gradle: io.opentelemetry:opentelemetry-exporter-otlp 등 의존성
// Spring Filter로 traceparent 수신/생성 → MDC 로깅 연동
@Component
public class TraceFilter implements Filter {
  @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
      throws IOException, ServletException {
    // OpenTelemetry SDK가 자동 처리 가능 (instrumentation agent 권장)
    chain.doFilter(req, res);
  }
}
```

**JDBC 호출에 모듈/액션 설정(예: 초기화 SQL)**
```java
try (Connection c = ds.getConnection()) {
  try (CallableStatement cs = c.prepareCall("{call DBMS_APPLICATION_INFO.SET_MODULE(?,?)}")) {
    cs.setString(1, "WEB-API");
    cs.setString(2, "GET /product/{id}");
    cs.execute();
  }
  try (CallableStatement cs = c.prepareCall("{call DBMS_SESSION.SET_IDENTIFIER(?)}")) {
    cs.setString(1, "trace-" + currentTraceId());
    cs.execute();
  }
  // 이후 PreparedStatement 실행...
}
```

### 3.3 메트릭/로그 수집
- **메트릭**: RPS, p50/p90/p99, 에러율, 스레드풀 큐 길이, 커넥션풀 대기, GC, JVM Heap, DB 커넥션 수  
- **로그**: 요청-응답 상관키 포함(JSON), 예외 스택, 타임라인 로그(시작/외부 호출/DB 호출/종료)

---

## 4. E2E 대시보드(필수 위젯)
1. **요청 지연**: p50/p90/p99, 루트 경로/서비스별  
2. **오류율**: HTTP 5xx/4xx  
3. **스루풋**: RPS, 동시세션/큐 길이  
4. **자원**: CPU/메모리/네트워크/디스크, JVM GC  
5. **DB**: AAS, Top Wait Class, Top SQL, Temp 사용, Redo/Commit 지연  
6. **캐시**: 히트율, miss penalty  
7. **알람**: SLO 위반 임계, 급상승 탐지

---

## 5. 성능 테스트 체계(자동화 관점)

### 5.1 타입
- **Smoke/샘플**: 회귀 빠른 감지  
- **부하(Load)**: 목표 RPS에서 안정성/여유 확인  
- **스트레스(Stress)**: 한계치/붕괴 지점 측정  
- **용량(Capacity)**: scale-out/업그레이드 시나리오  
- **소크(Soak)**: 장시간 메모리 누수/열화 체크

### 5.2 작업 예시(k6)
```bash
// script.js
import http from 'k6/http';
import { sleep, check } from 'k6';

export let options = {
  thresholds: { http_req_duration: ['p(95)<800'] }, // p95 800ms 예산
  stages: [
    { duration: '5m', target: 100 }, // ramp
    { duration: '10m', target: 100 }, // steady
    { duration: '5m', target: 0 },
  ],
};

export default function () {
  const res = http.get(`${__ENV.API}/product/123`, {
    headers: { 'traceparent': __ENV.TP || '' }
  });
  check(res, { 'status 200': (r) => r.status === 200 });
  sleep(1);
}
```

---

## 6. 엔드투엔드 병목 해부: **계층별 진단→처방**

### 6.1 클라이언트/브라우저
- **RUM(Real User Monitoring)**: FCP/LCP/INP/CLS, 네트워크 유형(3G/4G/Wi-Fi)  
- **처방**: 이미지 지연 로딩, 번들 분할, HTTP/2/3, CDN 캐시, 서버 측 렌더링/스트리밍

### 6.2 게이트웨이/LB
- **진단**: 접속/생성/큐 대기, Keep-Alive/커넥션 풀, TLS 핸드셰이크 시간  
- **처방**: 커넥션 재사용, 타임아웃/서킷브레이커, 헤더 압축(HTTP/2), 적절한 백엔드 풀 사이즈

### 6.3 애플리케이션(서비스)
- **진단**: 스레드풀 대기, DB 커넥션풀 대기, 외부 API/캐시 호출 지연, GC stop-the-world  
- **처방**: 비동기/배치, Bulk-head, Retries with jitter, Pool 사이징, N+1 제거, DTO 슬림화

### 6.4 캐시
- **진단**: 히트율, key 분포, 핫키, TTL 정책, 일관성 요구도  
- **처방**: 읽기-스루/쓰기-비하인드, 캐시 계층화(L1 JVM + L2 Redis), negative caching, key 샤딩

### 6.5 DB(Oracle) — **AWR/ASH/Plan 라인**으로 닫기
- **진단 루틴**  
  1) AWR/ASH로 Top Wait Class/Event/SQL  
  2) `DBMS_XPLAN.DISPLAY_CURSOR('ALLSTATS LAST')`로 **라인** 매핑  
  3) 대기↔라인 상관(Temp 스필/랜덤 I/O/락/커밋/클러스터)  
- **처방 요약**  
  - 인덱스/카디널리티/파티션/정렬 회피/PGA 정책  
  - FK 인덱스/ITL/트랜잭션 단축/커밋 배치  
  - RAC 로컬리티/키 분산/NOORDER

### 6.6 스토리지/네트워크
- **진단**: P99 I/O 지연, 디스크 대역포화, 네트워크 RTT/패킷 손실  
- **처방**: 로그/데이터/Temp 분리, NVMe/저지연, MTU/ECN/큐 관리, 스토리지 캐시 정책

---

## 7. **Oracle 연동 심화**: 요청→세션→SQL→라인까지 **한 줄로**

### 7.1 요청-단위 태깅(모듈/액션/클라이언트)
```sql
BEGIN
  DBMS_APPLICATION_INFO.SET_MODULE('WEB-API','POST /order');
  DBMS_SESSION.SET_IDENTIFIER('trace-0af3aa.../user-42');
END;
/
```

### 7.2 ASH로 “그 요청들”만 필터
```sql
VAR t1 TIMESTAMP; VAR t2 TIMESTAMP;
EXEC :t1 := SYSTIMESTAMP - INTERVAL '10' MINUTE; EXEC :t2 := SYSTIMESTAMP;

SELECT module, action, wait_class, event, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time BETWEEN :t1 AND :t2
  AND  session_type='FOREGROUND'
  AND  module='WEB-API' AND action='POST /order'
GROUP  BY module, action, wait_class, event
ORDER  BY samples DESC;
```

### 7.3 Top SQL과 Plan 라인
```sql
-- Top SQL
SELECT sql_id, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time BETWEEN :t1 AND :t2
  AND  module='WEB-API' AND action='POST /order'
GROUP  BY sql_id
ORDER  BY samples DESC FETCH FIRST 3 ROWS ONLY;

-- 라인 통계
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(:sql_id, NULL,
 'ALLSTATS LAST +PREDICATE +PROJECTION +PEEKED_BINDS +OUTLINE +NOTE'));
```

### 7.4 대기→라인 처방 예시
- `direct path read temp` @ `SORT GROUP BY`: **히스토그램+정렬 회피 인덱스+PGA↑**  
- `db file sequential read` @ `INDEX RANGE SCAN + BY ROWID`: **커버링 인덱스/프루닝/조인순서**  
- `enq: TX - row lock contention`: **FK 인덱스/긴 트랜잭션 단축/업데이트 순서 정렬**  
- `log file sync`: **배치 커밋/로그 경로 지연 개선**  
- `gc current block busy`(RAC): **서비스 로컬리티/키 분산/NOORDER**

---

## 8. 알람 설계(잡음 억제 + 빠른 탐지)

### 8.1 임계치
- API 별 **p95 응답**: 예산 초과 **3분 연속** 시 경보  
- 오류율: 5분 이동창 **>0.2%**  
- DB **AAS**: **코어 수 × 1.2** 초과 시 경보  
- DB **Temp I/O**: 분당 **X GB** 이상 지속

### 8.2 SLO 버짓 소모(에러/지연 예산)
- 일정 기간 SLO 버짓을 **얼마나 소모**했는지 추적 → **릴리즈 블록** 기준

---

## 9. 변화 관리: 성능 회귀를 막는 **게이트**

### 9.1 CI에서 성능 테스트 게이트
- 가벼운 **k6 smoke**: p95 예산 초과/오류율 초과 시 **PR 실패**  
- 쿼리 플랜 **잠금**: 주요 SQL은 **SQL Plan Baseline**으로 안정화

### 9.2 배포 후 **카나리** + 자동 롤백
- 카나리 트래픽에 대해 SLO/알람 감시 → 실패 시 자동 롤백

---

## 10. 운영 레시피(자주 겪는 상황 4선)

### 10.1 월말 리포트 느림(정렬 스필 급증)
- **징후**: p95 ↑, DB AWR에서 `direct path read temp` 상위  
- **원인**: 카디널리티 오판, 정렬/해시 과대  
- **처방**: 히스토그램/정렬 회피 인덱스/PGA↑ → **awrdiff**로 효과 검증

### 10.2 피크 OLTP 타임아웃(행잠금)
- **징후**: API p99 급등, `enq: TX`/`buffer busy`  
- **원인**: FK 미인덱스, 긴 트랜잭션, 키 편향  
- **처방**: FK 인덱스/커밋 주기/키 분산/INITRANS ↑

### 10.3 커밋 지연(`log file sync`)
- **징후**: POST/PUT만 느림, 커밋 이벤트 상위  
- **원인**: 행 단위 커밋 + 로그 I/O 지연  
- **처방**: 배치 커밋, 로그 디스크/네트워크 경로 개선

### 10.4 RAC 글로벌 캐시 경합
- **징후**: Cluster wait 상위, 인스턴스간 불균형  
- **원인**: 증가키/특정 파티션 집중  
- **처방**: 서비스 로컬리티, Reverse Key, NOORDER, 파티션 분해

---

## 11. 표준 스크립트 모음(복붙)

### 11.1 최근 15분 E2E 프로파일(서비스/모듈 기준)
```sql
-- ASH: 서비스/모듈별 Wait Profile
VAR t1 TIMESTAMP; VAR t2 TIMESTAMP;
EXEC :t1 := SYSTIMESTAMP - INTERVAL '15' MINUTE; EXEC :t2 := SYSTIMESTAMP;

SELECT NVL((SELECT name FROM v$active_services s WHERE s.service_hash=a.service_hash),'N/A') AS service,
       module, action, wait_class, event, COUNT(*) AS samples
FROM   v$active_session_history a
WHERE  a.sample_time BETWEEN :t1 AND :t2
  AND  a.session_type='FOREGROUND'
GROUP  BY a.service_hash, module, action, wait_class, event
ORDER  BY samples DESC FETCH FIRST 30 ROWS ONLY;
```

### 11.2 해당 모듈 Top SQL → 라인
```sql
SELECT sql_id, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time BETWEEN :t1 AND :t2
  AND  module='WEB-API'
GROUP  BY sql_id
ORDER  BY samples DESC FETCH FIRST 10 ROWS ONLY;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(:sql_id, NULL,
 'ALLSTATS LAST +PREDICATE +PROJECTION +PEEKED_BINDS +OUTLINE +NOTE'));
```

### 11.3 DB Time/CPU/Temp 단서(Time Model + Temp)
```sql
SELECT stat_name, ROUND(value/1e6,1) AS seconds
FROM   v$sys_time_model
WHERE  stat_name IN ('DB time','DB CPU','sql execute elapsed time','parse time elapsed')
ORDER  BY 2 DESC;

SELECT tablespace_name, SUM(blocks) blocks
FROM   v$sort_usage
GROUP  BY tablespace_name
ORDER  BY blocks DESC;
```

### 11.4 V$SQL Top-N(평균 단가 포함)
```sql
SELECT sql_id, child_number, executions,
       ROUND(elapsed_time/1e6,1) elapsed_s,
       ROUND(elapsed_time/1e6/NULLIF(executions,0),3) s_per_exec,
       buffer_gets, disk_reads, plan_hash_value, last_active_time
FROM   v$sql
ORDER  BY elapsed_time DESC
FETCH FIRST 30 ROWS ONLY;
```

---

## 12. 보안/개인정보 & 비용 고려
- **클라이언트 식별자/로그**에 PII 포함 시 **마스킹/보존주기/접근권한** 정책 필수.  
- **진단팩 라이선스** 준수(ASH/AWR/ADDM/Real-Time SQL Monitor 등).  
- 과도한 고해상도 추적은 **스토리지/쿼리 비용** 폭증 → **샘플링/요약** 전략 설계.

---

## 13. 체크리스트(요약)

1) **SLO**를 먼저 쓰고, **성능 예산**으로 나눈다.  
2) **Trace Context**와 **Oracle 세션 태깅**으로 호출을 **끝까지 관통**시킨다.  
3) 대시보드는 **p95/99, 오류율, RPS, 자원, DB AAS/Top Wait/Top SQL** 을 한 화면에.  
4) 문제 시 **Top Wait/Event → Top SQL → Plan 라인**으로 3단 점프.  
5) PR/배포에 **성능 게이트**를 설치하여 **회귀**를 막는다.  
6) 전/후(AWR Diff, p95)로 반드시 **효과를 수치로 증명**한다.

---

## 14. 결론
엔드투엔드 성능 관리는 **측정되지 않으면 존재하지 않는다**는 명제를 전제로,  
**SLI/SLO-예산 → 계측(Trace/Metric/Log) → 분해(RTA) → 상관(Trace↔DB) → 처방 → 재검증**의 루프를 자동화하는 일이다.  
**Oracle** 계층은 `DBMS_APPLICATION_INFO + AWR/ASH/DBMS_XPLAN`으로 **서비스 호출-단위**로 정밀하게 닫을 수 있다.  
이 루프를 팀의 **습관**으로 만드는 순간, 성능은 **운**이 아니라 **공정**이 된다.

> 한 줄 정리  
> **E2E = “사용자 체감”을 목표로, 호출을 헤더로 묶고(DB 태깅), 메트릭/트레이스/플랜으로 분해한 뒤, 성능 예산을 지키는 게임**이다.