---
layout: post
title: AWS - Kinesis Data Analytics
date: 2025-07-28 18:20:23 +0900
category: AWS
---
# 📊 AWS Kinesis Data Analytics 완전 정복

**Kinesis Data Analytics(KDA)**는 실시간으로 들어오는 스트리밍 데이터를 **SQL, Java, Apache Flink**를 활용해 분석할 수 있게 해주는 완전관리형 서비스입니다.  
빅데이터, 실시간 대시보드, 이상 탐지, IoT 등 다양한 분야에서 활용됩니다.

---

## 1. 🔍 Kinesis Data Analytics 개요

Kinesis Data Analytics는 스트리밍 데이터를 실시간으로 처리하고 분석하는 완전 관리형 서비스입니다.  
데이터 소스는 다음과 같은 서비스와 연결됩니다:

- **Kinesis Data Streams**
- **Kinesis Data Firehose**
- **Amazon MSK (Managed Kafka)**

이 데이터를 SQL 또는 Flink 애플리케이션으로 처리하여:

- 실시간 대시보드로 시각화
- 이상 탐지
- 알림 발송
- 데이터 적재(S3, Redshift, Lambda)

등을 수행할 수 있습니다.

---

## 2. 🧩 주요 특징

| 기능 | 설명 |
|------|------|
| 실시간 스트리밍 분석 | 실시간으로 수집된 데이터에 대해 분석 실행 |
| SQL/Flink 지원 | SQL 기반 또는 Apache Flink 애플리케이션 생성 가능 |
| 자동 확장 | 데이터량에 따라 자동 확장 |
| 통합 | Kinesis, Firehose, MSK 등과 통합 |
| 운영 용이성 | 완전관리형으로 유지보수 부담 없음 |

---

## 3. 🏗 아키텍처 구성

```text
Data Source → Kinesis Data Stream / Firehose / MSK
           → Kinesis Data Analytics(SQL 또는 Flink)
           → S3, Redshift, OpenSearch, Lambda, CloudWatch 등
```

---

## 4. 💻 지원 언어: SQL vs Apache Flink

### SQL 기반 KDA
- 초보자 친화적
- 빠른 시나리오 구축 가능
- 기본적인 필터링, 집계, 조인 가능

### Apache Flink 기반 KDA
- Java/Scala 기반
- 복잡한 상태 기반 연산, 타임윈도우, 이벤트 시간 지원
- 강력한 커스터마이징 가능

---

## 5. 🔌 데이터 소스 연결 (Input)

애플리케이션을 만들면 Input source를 지정해야 합니다.

### 지원 소스
- Amazon Kinesis Data Streams
- Amazon Kinesis Data Firehose
- Amazon MSK (Kafka)

### 예: Data Stream 연결

```json
{
  "Input": {
    "NamePrefix": "MyInput",
    "KinesisStreamsInput": {
      "ResourceARN": "arn:aws:kinesis:region:acct:stream/MyStream"
    },
    "InputSchema": {
      "RecordFormat": {
        "RecordFormatType": "JSON"
      },
      "RecordColumns": [
        {"Name": "event_time", "SqlType": "TIMESTAMP", "Mapping": "$.timestamp"},
        {"Name": "user_id", "SqlType": "VARCHAR(64)", "Mapping": "$.user_id"},
        {"Name": "action", "SqlType": "VARCHAR(20)", "Mapping": "$.action"}
      ]
    }
  }
}
```

---

## 6. ⚙️ 데이터 처리 (SQL)

SQL 기반에서는 다음과 같은 연산을 수행할 수 있습니다:

- 필터링 (`WHERE`)
- 집계 (`SUM`, `AVG`, `COUNT`)
- 윈도우 함수 (`TUMBLING`, `SLIDING`, `SESSION`)
- JOIN

### 윈도우 예제

```sql
CREATE OR REPLACE STREAM "DEST_STREAM" (
  user_id VARCHAR(64),
  click_count INTEGER
);

CREATE OR REPLACE PUMP "STREAM_PUMP" AS
INSERT INTO "DEST_STREAM"
SELECT user_id, COUNT(*) AS click_count
FROM "SOURCE_STREAM"
WINDOWED BY TUMBLING (INTERVAL '1' MINUTE)
GROUP BY user_id;
```

---

## 7. 📤 결과 저장 (Output)

처리된 결과는 다양한 서비스로 출력할 수 있습니다:

- **S3** (ETL 후 저장)
- **Lambda** (이벤트 기반 후속 처리)
- **Redshift** (분석용 데이터 적재)
- **OpenSearch** (로그 검색)
- **Firehose** (중간 전달)

---

## 8. 📘 실시간 분석 예제 (SQL 기반)

### 사용자 이벤트 실시간 집계

```sql
-- CREATE STREAM
CREATE OR REPLACE STREAM "OUTPUT_STREAM" (
  action VARCHAR(20),
  count_per_min INTEGER
);

-- PUMP 정의
CREATE OR REPLACE PUMP "ACTION_PUMP" AS
INSERT INTO "OUTPUT_STREAM"
SELECT action, COUNT(*) as count_per_min
FROM "INPUT_STREAM"
WINDOWED BY TUMBLING (INTERVAL '1' MINUTE)
GROUP BY action;
```

---

## 9. 🚀 Flink 기반 고급 분석

Flink를 사용하면 다음과 같은 기능을 사용할 수 있습니다:

- 이벤트 시간 기반 처리
- 복잡한 타임윈도우, 세션윈도우
- 상태 저장 (Stateful Processing)
- CEP (Complex Event Processing)

```java
DataStream<Event> events = env
  .addSource(new FlinkKinesisConsumer<>(...))
  .assignTimestampsAndWatermarks(...);

events
  .keyBy(event -> event.getUserId())
  .window(SlidingEventTimeWindows.of(Time.minutes(5), Time.minutes(1)))
  .aggregate(new EventCountAggregator())
  .addSink(new FlinkKinesisProducer(...));
```

---

## 10. 🔄 통합 예시

### Kinesis → KDA → S3 저장

1. Kinesis Data Stream에 JSON 로그 삽입
2. KDA SQL 애플리케이션에서 필터링 및 집계
3. Firehose를 통해 S3 버킷에 저장

---

## 11. 🧪 운영 및 모니터링

- **CloudWatch**: 메트릭/로그 확인
- **Checkpointing**: 상태 기반 Flink 애플리케이션에서는 Checkpoint 기능 지원
- **Application Versioning**: 버전별 롤백 가능
- **Monitoring UI**: Input/Output 레코드 수, 처리 지연 확인 가능

---

## 12. 💰 비용

- **SQL 기반**: 사용한 vCPU 시간당 요금
- **Flink 기반**: 병렬 작업 수, 메모리, vCPU 기준 요금
- 데이터 전송량은 별도 과금 (예: Kinesis, S3, Firehose 등)

**예시:**
- $0.11/vCPU-시간 (SQL)
- $0.10/GB (Firehose → S3)

---

## 13. 🔐 보안

- VPC 통합 가능
- IAM 역할/정책으로 접근 제어
- 데이터 암호화 지원 (KMS)
- 소스/싱크 서비스의 권한 위임 필요

---

## 14. 🧠 사용 사례

| 사용 사례 | 설명 |
|-----------|------|
| 실시간 대시보드 | 웹 이벤트를 실시간으로 시각화 |
| IoT 센서 분석 | 시간 기반 윈도우로 이상 탐지 |
| 마케팅 캠페인 분석 | 유저 행동 기반 실시간 피드백 |
| 보안 로그 분석 | 실시간 공격 탐지 |
| 금융 트랜잭션 분석 | 이상 탐지 및 경보 |

---

## 15. 📌 요약

| 항목 | 내용 |
|------|------|
| 데이터 처리 | 실시간 스트리밍 데이터 처리 |
| 입력 소스 | Kinesis Stream, Firehose, MSK |
| 처리 방식 | SQL 또는 Apache Flink |
| 출력 대상 | S3, Redshift, Lambda, Firehose 등 |
| 확장성 | 자동 확장, 상태 저장 |
| 운영 | 완전관리형, CloudWatch 통합 |

---

## ✅ 마무리

Kinesis Data Analytics는 AWS의 스트리밍 분석 중심에 있는 서비스입니다.  
SQL의 단순함과 Flink의 복잡한 기능 모두를 지원하므로, 데이터 파이프라인 및 실시간 분석에 매우 유용한 도구입니다.
