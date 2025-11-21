---
layout: post
title: AWS - Kinesis Data Analytics
date: 2025-07-28 18:20:23 +0900
category: AWS
---
# AWS Kinesis Data Analytics

## 한눈에 보는 전체 아키텍처

```text
┌────────────┐    PutRecord      ┌────────────────────┐
│  Producers │ ─────────────────▶│ Kinesis Data Streams│
│  (Apps/IoT)│                   └────────────────────┘
└────────────┘                           │
                                         │ (Streams / MSK / Firehose)
                                         ▼
                              ┌────────────────────────────┐
                              │ Kinesis Data Analytics     │
                              │  • SQL  • Apache Flink     │
                              └────────────────────────────┘
                         ┌──────────────┬───────────────┬──────────────┐
                         ▼              ▼               ▼              ▼
                  Amazon S3        Lambda           Redshift     OpenSearch
                  (Data Lake)      (Actions)        (DWH)        (Search/Logs)
```

**핵심 판단 기준**
- **SQL 애플리케이션**: 빠른 구축, 운영 단순, 집계/필터/윈도우/간단 조인.
- **Flink 애플리케이션**: 이벤트타임·상태 기반·CEP·세밀한 backpressure 제어·Exactly-once.

---

## 입력 소스와 레코드 스키마

### 지원 소스

- **Kinesis Data Streams (KDS)**, **Amazon MSK (Kafka)**, **Kinesis Data Firehose**(일부 경로)
- 포맷: JSON/CSV/Avro 등(직접 파싱). **스키마 정의**는 SQL 런타임에 필수.

### JSON 스키마 예시 (클릭 이벤트)

```json
{
  "timestamp": "2025-11-10T02:31:12Z",
  "user_id": "u-1932",
  "action": "click",
  "page": "/products/42",
  "device": "mobile",
  "country": "KR",
  "value": 1
}
```

**이벤트 타임 추출 규칙**
- 원천 메시지의 `timestamp`를 SQL/Flink에서 **ROWTIME**로 매핑해 **Event Time** 기반 처리.

---

## KDA for SQL — 실무 문법 총정리

### 입력 매핑 (Application Schema)

KDA 콘솔/CLI에서 입력 스키마를 정의:

```json
{
  "Input": {
    "NamePrefix": "evt",
    "KinesisStreamsInput": {
      "ResourceARN": "arn:aws:kinesis:ap-northeast-2:123456789012:stream/app-events"
    },
    "InputSchema": {
      "RecordFormat": {"RecordFormatType": "JSON"},
      "RecordColumns": [
        {"Name":"ROWTIME","SqlType":"TIMESTAMP","Mapping":"$.timestamp"},
        {"Name":"user_id","SqlType":"VARCHAR(64)","Mapping":"$.user_id"},
        {"Name":"action","SqlType":"VARCHAR(20)","Mapping":"$.action"},
        {"Name":"page","SqlType":"VARCHAR(256)","Mapping":"$.page"},
        {"Name":"country","SqlType":"VARCHAR(8)","Mapping":"$.country"},
        {"Name":"value","SqlType":"INTEGER","Mapping":"$.value"}
      ]
    }
  }
}
```

> **TIP**: `ROWTIME`는 예약 컬럼(이벤트 타임). 없으면 프로세싱 타임으로 동작해 늦게 도착한 이벤트 보정이 어렵다.

### 인-애플리케이션 스트림/펌프

SQL 모듈은 **STREAM**(결과 버퍼)과 **PUMP**(연산 파이프) 개념 사용.

```sql
-- 결과 스트림 정의
CREATE OR REPLACE STREAM "s_agg_action_country" (
  country VARCHAR(8),
  action  VARCHAR(20),
  cnt     BIGINT
);

-- 1분 Tumbling 윈도우 집계
CREATE OR REPLACE PUMP "p_agg_action_country" AS
INSERT INTO "s_agg_action_country"
SELECT country, action, COUNT(*) AS cnt
FROM "evt_001"
WINDOWED BY TUMBLING (INTERVAL '1' MINUTE)
GROUP BY country, action;
```

### 시간 윈도우

- **TUMBLING(M)**: 고정크기, 겹침 없음
- **SLIDING(M, step)**: 겹치는 창(슬라이드)
- **SESSION(gap)**: 활동-휴지간격으로 묶음

```sql
-- 5분 윈도우를 1분 간격으로 굴리고 상위 페이지 계산
CREATE OR REPLACE STREAM "s_top_pages" (
  page VARCHAR(256),
  cnt  BIGINT
);

CREATE OR REPLACE PUMP "p_top_pages" AS
INSERT INTO "s_top_pages"
SELECT page, COUNT(*) cnt
FROM "evt_001"
WINDOWED BY SLIDING (INTERVAL '5' MINUTE, INTERVAL '1' MINUTE)
GROUP BY page;
```

### 조인(스트림-스트림, 스트림-레퍼런스)

- 스트림-스트림 조인: 동일 윈도우 또는 키 기반, 시간 조건 필요.
- 레퍼런스(정적) 조인: S3에 주기 싱크된 **Reference Table** 지원.

```sql
-- KV 형태의 레퍼런스(타겟 국가만 패스)
CREATE OR REPLACE REFERENCE TABLE "t_allowed_country" (
  country VARCHAR(8) PRIMARY KEY
);

CREATE OR REPLACE STREAM "s_allowed_clicks" (
  user_id VARCHAR(64),
  country VARCHAR(8)
);

CREATE OR REPLACE PUMP "p_allowed" AS
INSERT INTO "s_allowed_clicks"
SELECT e.user_id, e.country
FROM "evt_001" AS e
JOIN "t_allowed_country" r
ON e.country = r.country
WHERE e.action = 'click';
```

### 결과 출력(DESTINATION)

- Output을 **Kinesis stream**, **Lambda**, **Firehose→S3**, **MSK** 등으로 보냄.

```sql
-- 출력 스트림 메타
CREATE OR REPLACE STREAM "s_out_to_s3" (
  country VARCHAR(8),
  action  VARCHAR(20),
  cnt     BIGINT
);
-- 실제 펌프에선 s_agg_action_country를 s_out_to_s3로 라우팅
CREATE OR REPLACE PUMP "p_out" AS
INSERT INTO "s_out_to_s3"
SELECT * FROM "s_agg_action_country";
```

> 콘솔에서 **Output**을 생성하며 대상과 포맷/배치 옵션을 지정한다.

### 지연 이벤트와 Out-of-Order

- SQL 런타임은 **Event Time** 및 **Out-of-Order 허용**(윈도우 폐쇄 시점 설정) 지원.
- **RUNTIME 설정**에서 늦게 도착 허용(Time to Live) 파라미터를 보정.

---

## KDA for Apache Flink — 프로덕션 패턴

### 필수 개념

- **Event Time + Watermark**
- 상태(State): **Keyed State**, RocksDB 백엔드
- **Exactly-once**: Kinesis/Checkpointing 설정 필요
- **Window/CEP**: Session·Sliding·Pattern

### Maven 의존성(개요)

```xml
<dependencies>
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-streaming-java</artifactId>
    <version>${flink.version}</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kinesis</artifactId>
    <version>${flink.version}</version>
  </dependency>
  <!-- OpenSearch/S3/Parquet sink 등 필요에 맞게 추가 -->
</dependencies>
```

### Flink 코드 예제 (Kinesis → 윈도우 집계 → Kinesis)

```java
import org.apache.flink.api.common.eventtime.*;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.*;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kinesis.FlinkKinesisConsumer;
import org.apache.flink.streaming.connectors.kinesis.FlinkKinesisProducer;
import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;

import java.time.Duration;
import java.util.Properties;

public class KdaFlinkApp {
  public static void main(String[] args) throws Exception {
    final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

    // Checkpointing: Exactly-once를 위해 필수
    env.enableCheckpointing(60000); // 60s
    env.getCheckpointConfig().setMinPauseBetweenCheckpoints(30000);

    // Kinesis Consumer
    Properties consumerProps = new Properties();
    consumerProps.setProperty("aws.region", "ap-northeast-2");
    consumerProps.setProperty("flink.stream.initpos", "LATEST");

    DataStream<String> raw = env.addSource(
        new FlinkKinesisConsumer<>("app-events", new SimpleStringSchema(), consumerProps));

    // Timestamp/Watermark (5초 지연 허용)
    DataStream<Event> events = raw
      .map(JsonParsers::toEvent) // user code: String -> Event(timestamp, userId, action, page, country, value)
      .assignTimestampsAndWatermarks(
        WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
          .withTimestampAssigner((e, ts) -> e.getTimestamp()));

    // 상태 기반 집계 (5분 윈도우를 1분 슬라이드)
    DataStream<Aggregate> agg = events
      .keyBy(Event::keyByCountryAction) // country#action
      .window(SlidingEventTimeWindows.of(Time.minutes(5), Time.minutes(1)))
      .aggregate(new Aggregators.CountPerKey());

    // Kinesis Producer
    Properties producerProps = new Properties();
    producerProps.setProperty("aws.region", "ap-northeast-2");
    FlinkKinesisProducer<String> sink = new FlinkKinesisProducer<>(
        new SimpleStringSchema(), producerProps);
    sink.setDefaultStream("analytics-agg");
    sink.setDefaultPartition("0");

    agg.map(Aggregate::toJson).addSink(sink);

    env.execute("KDA Flink Analytics");
  }
}
```

### Checkpoint/Savepoint/Parallelism

- KDA 콘솔에서 **병렬도**, **Autoscaling**, **Checkpoint interval**, **State backend**를 설정
- 롤링 배포: **Savepoint** 생성 → 새 버전으로 **Restore** (무중단/데이터 무손실)

### 예

```java
// 클릭 후 10분 내 구매 패턴
Pattern<Event, ?> pattern = Pattern.<Event>begin("click")
  .where(e -> "click".equals(e.getAction()))
  .next("purchase").where(e -> "purchase".equals(e.getAction()))
  .within(Time.minutes(10));
```

---

## 전략

### S3 (Parquet)로 적재 — 배치/마이크로배치

- Flink: **FileSystem sink** + **Bucket assigner**로 파티션(날짜/시간) 저장
- SQL: **Output→Firehose→S3** 경로가 운영 단순

### OpenSearch/Redshift/Lambda

- 집계/지표는 OpenSearch 대시보드
- 사실 테이블은 S3→Glue→Athena/Redshift Spectrum
- 실시간 액션은 Lambda 구독(알림/차단 등)

---

## 성능·비용 최적화

### 처리량과 레이턴시 모델

$$
\text{Throughput} \approx \text{Parallelism} \times \text{Per-Task Rate} \times \text{Scale Factor}
$$

- Kinesis 샤드 수(읽기 2MB/s·초당 5회 읽기), Flink 병렬도, 다운스트림 쓰기 한계 고려
- **Backpressure** 지표 모니터링, Operator 별 busy-time 확인

### SQL 최적화

- **SELECT * 금지**: 필요한 컬럼만
- 윈도우 크기 최소화(지연 허용·정확도 균형)
- 레퍼런스 테이블 **키 인덱스** 구성, 갱신 주기 관리

### Flink 최적화

- **RocksDB State** 사용 + 블룸필터/압축 튜닝
- **Watermark** 허용 지연 최소화
- **Rescale**: 병렬도 상향, Operator Chain 활용
- S3 sink: 파일 **소형화 방지**(Rolling policy), Parquet 128~256MB 타깃

### 코스트 모델(개략)

$$
\text{Total Cost} \approx \text{KDS Shards} + \text{KDA (vCPU·Mem·Hours)} + \text{Downstream (e.g., S3 GB, OpenSearch ingests)}
$$

- SQL: vCPU 시간 기반
- Flink: JobManager/TaskManager 리소스·병렬도·운영시간

---

## 보안·네트워킹·권한

### VPC 연동

- KDA 애플리케이션을 **VPC에 연결**해 프라이빗 리소스(RDS/ElastiCache) 접근
- **서브넷/보안그룹** 최소 권한, NAT/엔드포인트 구성

### IAM 역할 (필수 권한 예시)

```json
{
  "Version":"2012-10-17",
  "Statement":[
    {"Effect":"Allow","Action":["kinesis:*"],"Resource":"arn:aws:kinesis:ap-northeast-2:123456789012:stream/*"},
    {"Effect":"Allow","Action":["logs:*"],"Resource":"*"},
    {"Effect":"Allow","Action":["s3:GetObject","s3:PutObject","s3:ListBucket"],"Resource":["arn:aws:s3:::datalake","arn:aws:s3:::datalake/*"]},
    {"Effect":"Allow","Action":["kms:Decrypt","kms:Encrypt","kms:GenerateDataKey"],"Resource":"arn:aws:kms:ap-northeast-2:123456789012:key/xxxx"}
  ]
}
```

### 암호화

- Kinesis/KDA/S3/OpenSearch 전 구간 **KMS** 암호화
- MSK는 TLS/SASL, VPC 보안그룹 세분화

---

## 모니터링·가시성·운영

### CloudWatch 지표

- **KDA SQL**: Input/Output 레코드, MillisBehindLatest, BackpressuredTime
- **KDA Flink**: Checkpoint Duration/Alignment, Busy Time, Watermark Lag
- 경보(Alarm) → SNS/Lambda 조치

### 로그

- **애플리케이션 로그**(stderr/stdout)
- Flink **TaskManager/JobManager** 로그
- 실패 레코드 Dead-letter 경로(Firehose Backup/S3) 설계

### 배포·버전 관리

- SQL: 애플리케이션 버전 저장, **롤백 버튼**
- Flink: **Savepoint** 기반 blue/green 롤아웃
- IaC: CloudFormation/SAM/Terraform으로 KDS/KDA/Outputs 일괄 프로비저닝

---

## 운영급 실전 예제

### 시나리오

- 목표: **1분 단위 국가·액션별 카운트**, 5분 이동 윈도우 트렌드, S3 파이프라인
- 입력: `app-events` (JSON)
- 출력: `analytics-agg` (Kinesis) + Firehose→S3 (Parquet)

#### SQL 애플리케이션 정의

```sql
-- 1) 결과 스트림들
CREATE OR REPLACE STREAM "s_cnt_1m" (country VARCHAR(8), action VARCHAR(20), cnt BIGINT);
CREATE OR REPLACE STREAM "s_trend_5m" (country VARCHAR(8), action VARCHAR(20), cnt BIGINT);

-- 2) 1분 튜블링
CREATE OR REPLACE PUMP "p_cnt_1m" AS
INSERT INTO "s_cnt_1m"
SELECT country, action, COUNT(*) AS cnt
FROM "evt_001"
WINDOWED BY TUMBLING (INTERVAL '1' MINUTE)
GROUP BY country, action;

-- 3) 5분 슬라이딩(1분 단위)
CREATE OR REPLACE PUMP "p_trend_5m" AS
INSERT INTO "s_trend_5m"
SELECT country, action, COUNT(*) AS cnt
FROM "evt_001"
WINDOWED BY SLIDING (INTERVAL '5' MINUTE, INTERVAL '1' MINUTE)
GROUP BY country, action;
```

> 콘솔에서 `s_cnt_1m`, `s_trend_5m`를 **Output**으로 등록(하나는 Kinesis, 하나는 Firehose→S3).

#### Firehose → S3 (Parquet) 설정 포인트

- **Buffer Size/Interval**: 64–128MiB / 60–300s
- **Parquet + Snappy**, Glue Schema Registry(선택)

---

## 고급 Flink 예제 (CEP + 상태)

### “의심 행동” 탐지: 10분 내 5회 이상 ‘login_failed’ → 알림

```java
events
  .filter(e -> "login_failed".equals(e.getAction()))
  .keyBy(Event::getUserId)
  .window(SlidingEventTimeWindows.of(Time.minutes(10), Time.minutes(1)))
  .process(new ProcessWindowFunction<Event, Alert, String, TimeWindow>() {
    @Override
    public void process(String userId, Context ctx, Iterable<Event> input, Collector<Alert> out) {
      long c = 0;
      for (Event e : input) c++;
      if (c >= 5) out.collect(new Alert(userId, c, ctx.window().getEnd()));
    }
  })
  .map(Alert::toJson)
  .addSink(new FlinkKinesisProducer<>(new SimpleStringSchema(), props));
```

### Exactly-once 싱크(OpenSearch 예)

- Flink OpenSearch Sink v2는 **two-phase commit** 유사 시맨틱 지원(버전에 따름)
- 실패 시 재시도 + backoff, **index template** 선등록

---

## 테스트·검증·데이터 품질

- **로컬/샘플 스트림**으로 SQL/Flink 로직 단위 테스트
- 데이터 품질 규칙: **Null/타입/범위/카디널리티** 쿼리로 자동 점검
```sql
-- 품질 검증 예: 허용되지 않은 국가
SELECT country, COUNT(*) c
FROM "evt_001"
WHERE country NOT IN ('KR','US','JP','DE')
WINDOWED BY TUMBLING (INTERVAL '5' MINUTE)
GROUP BY country;
```
- **리플레이**: 테스트 스트림에 과거 데이터 재생, 윈도우/지연 처리 확인

---

## 트러블슈팅 모음

| 증상 | 원인 | 해결 |
|---|---|---|
| SQL 결과가 비거나 늦음 | ROWTIME 미설정/포맷 문제 | 입력 스키마의 `ROWTIME` 매핑 재확인, 시간대·ISO8601 통일 |
| 지연 이벤트 누락 | 허용 지연 짧음 | Out-of-order 허용 시간 확대, 윈도우 폐쇄 지연 |
| Flink 백프레셔 | 다운스트림 불능/느린 sink | 병렬도 상향, 배치 사이즈 조정, sink 재시도/버퍼 튜닝 |
| Checkpoint 실패 | 네트워크/KMS 권한 | VPC/NAT/KMS 권한/엔드포인트 확인, interval 증가 |
| 비용 급증 | 샤드 과다/쿼리 비효율 | 샤드·병렬도/윈도우 크기 최적화, 필터/컬럼 최소화 |
| 중복/유실 의심 | Exactly-once 미구현 | Checkpoint/commit 보장 sink 사용, 멱등키 설계 |

---

## 거버넌스/보안 체크리스트

- [ ] **IAM 최소 권한**: Kinesis, Logs, S3, KMS, OpenSearch
- [ ] **전송·저장 암호화**: TLS, SSE-KMS
- [ ] **VPC 엔드포인트**: S3/Kinesis/Logs 엔드포인트로 사설 경로
- [ ] **비밀 관리**: Secrets Manager/Parameter Store
- [ ] **데이터 마스킹/PII 제거**: SQL/Lambda/Flink 단계에서 필수

---

## 비용 통제 레시피

- SQL: **vCPU 축소 + 윈도우/집계 최소화**, 필요 스트림만 출력
- Flink: 병렬도/체크포인트 간격/상태 크기 관리, **compact sink**
- 다운스트림: S3 **Parquet + 파티션**, OpenSearch 인덱스 정책(ILM/수명)

---

## 산출물·IaC 예시 스니펫

### Kinesis Stream (CloudFormation)

```yaml
Resources:
  AppEvents:
    Type: AWS::Kinesis::Stream
    Properties:
      Name: app-events
      ShardCount: 4
```

### KDA SQL App (요지)

```yaml
Resources:
  KdaSqlApp:
    Type: AWS::KinesisAnalyticsV2::Application
    Properties:
      RuntimeEnvironment: SQL-1_0
      ServiceExecutionRole: arn:aws:iam::123456789012:role/kda-sql-role
      ApplicationConfiguration:
        SqlApplicationConfiguration:
          Inputs:
            - NamePrefix: evt
              InputSchema: ... # 위 스키마
              KinesisStreamsInput:
                ResourceARN: arn:aws:kinesis:ap-northeast-2:123456789012:stream/app-events
        ApplicationCodeConfiguration:
          CodeContent:
            TextContent: |
              -- SQL들 (CREATE STREAM/PUMP ...)
```

---

## 요약

- **KDA SQL**: 가장 빠르게 **실시간 집계/필터/윈도우**를 제품화. 운영 단순.
- **KDA Flink**: **Event Time·State·CEP·Exactly-once**가 필요한 **엔터프라이즈 스트리밍**의 정석.
- 핵심 성공 요인: **정확한 이벤트타임/워터마크 설계**, **상태와 체크포인트**, **적절한 싱크 패턴**, **모니터링·알람·자동복구**, **보안·VPC·KMS**.

---

## SQL 참조 — 세션 윈도우

```sql
CREATE OR REPLACE STREAM "s_session" (user_id VARCHAR(64), clicks BIGINT);

CREATE OR REPLACE PUMP "p_session" AS
INSERT INTO "s_session"
SELECT user_id, COUNT(*) AS clicks
FROM "evt_001"
-- 10분 동안 아무 이벤트 없으면 세션 종료
WINDOWED BY SESSION (INTERVAL '10' MINUTE)
GROUP BY user_id;
```

## Flink — 파케이 싱크 스니펫(파일 롤링)

```java
final FileSink<String> sink = FileSink
  .forRowFormat(new Path("s3://datalake/realtime/"), new SimpleStringEncoder<String>("UTF-8"))
  .withRollingPolicy(DefaultRollingPolicy.builder()
     .withRolloverInterval(Duration.ofMinutes(5)).withInactivityInterval(Duration.ofMinutes(1))
     .withMaxPartSize(MemorySize.ofMebiBytes(256)).build())
  .build();

events.map(Event::toJson).sinkTo(sink);
```

## 간단 비용 산식

$$
\text{KDA Cost} \approx \text{vCPU Hours} \times \text{Unit Price} \quad (+ \text{Memory Premium if any})
$$
$$
\text{Total} \approx \text{KDS Shards} + \text{KDA} + \text{S3/OpenSearch/Firehose} + \text{Data Transfer}
$$

---

### 마무리

Kinesis Data Analytics는 **“즉시 쓸 수 있는 SQL”**과 **“무한히 정교해지는 Flink”** 두 축으로 실시간 파이프라인을 완성한다.
본 가이드의 설계/코드/운영 체크리스트를 템플릿 삼아, **작게 시작해 확장 가능하게** 만들자. 그러면 대시보드·알림·머신러닝 피쳐링까지 **한 흐름**으로 자연스럽게 이어진다.
