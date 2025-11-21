---
layout: post
title: AWS - Glue + Athena 연동
date: 2025-07-28 20:20:23 +0900
category: AWS
---
# AWS Glue + Athena 연동

## 한눈에 보는 아키텍처와 제작 체크리스트

```text
┌──────────┐   putObject    ┌───────────────┐  crawl(schema)  ┌──────────────┐   query(SQL)
│  Sources  ├──────────────▶│     Amazon     ├────────────────▶│ Glue Catalog │◀────────────┐
│ (apps,    │               │       S3       │                 └──────────────┘             │
│  logs, DB)│               └───────────────┬┘                        ▲                      │
└──────────┘         partitioned layout     │                         │                      │
                                             │   transform(ETL)       │                      ▼
                                             ├───────────────▶  Glue Jobs (Spark/Python)  Amazon Athena
                                             │
                                             └─▶ Glue Crawler / Classifier (schema inference)
```

**필수 체크리스트**
- [ ] S3 **레이아웃/파티션 키** 설계 (`dt=YYYY-MM-DD/…` 또는 다중 키)
- [ ] **파일 포맷**: Parquet(+Snappy) 기본, 텍스트는 임시/원천 보관
- [ ] Glue **데이터베이스/테이블 네이밍** 표준화, 컬럼 타입 고정
- [ ] 크롤러 스코프 최소화(원천/정제 경로 분리), 커스텀 Classifier 필요 시 정의
- [ ] Athena **WorkGroup** 분리, 결과 버킷/암호화 지정, 쿼리 가드레일 설정
- [ ] 파티션 프로젝션/메타 자동화, ETL 시 **MSCK** or `ALTER TABLE ADD PARTITION` 처리
- [ ] 보안: S3 BPA(공개차단), SSE-KMS, Lake Formation 권한 모델, 세분화된 IAM
- [ ] 코스트: Parquet, 파티션 프루닝, 압축, `SELECT *` 금지, Explain 분석

---

## 데이터 레이크 물리 설계: S3 경로·파티션·포맷

### 디렉터리/파티션 레이아웃

일반적인 시간 파티션:

```
s3://datalake/raw/access_logs/dt=2025-11-10/hour=02/part-000.parquet
s3://datalake/refined/access_logs/dt=2025-11-10/hour=02/...
```

권장 규칙
- **소스/영역 분리**: `raw/`, `staged/`, `refined/`, `curated/`
- **키-값 디렉토리**: `key=value` 형식이 Athena/Glue 친화적
- 파티션 키는 **카디널리티**와 **쿼리 필터** 기준으로 선정
- 파일 크기: 128–512MB 권장(Parquet), 너무 작으면 스캔 과다/쿼리 느림

### 포맷 선택

| 포맷 | 장점 | 단점 | 권장 |
|---|---|---|---|
| CSV | 단순 | 스캔량 큼, 타입 불명 | 임시/원본 보관 |
| JSON | 반정형 | 스캔량 큼 | 소규모·가변 스키마 |
| **Parquet** | **열지향/압축/프루닝** | ETL 필요 | **기본** |
| ORC | 열지향 | 혼용 시 관리 복잡 | Parquet 대안 |

---

## Glue 데이터 카탈로그 모델링

### 데이터베이스/테이블 설계

- `db_raw`, `db_refined`, `db_curated` 등 **영역별 DB**
- 테이블 이름: `<도메인>_<엔티티>_<레벨>` 예) `access_logs_raw`

```sql
-- Athena에서 메타스토어로 Glue Catalog 사용
CREATE DATABASE IF NOT EXISTS db_refined;
```

### 외부 테이블 DDL(Parquet)

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS db_refined.access_logs (
  client_ip          string,
  method             string,
  path               string,
  status_code        int,
  bytes_sent         bigint,
  user_agent         string,
  ts                 timestamp
)
PARTITIONED BY (dt string, hour string)
STORED AS PARQUET
LOCATION 's3://datalake/refined/access_logs/'
TBLPROPERTIES (
  'parquet.compression'='SNAPPY'
);
```

**주의**: Parquet는 스키마 강제. ETL에서 컬럼 타입 일관 유지.

---

## Glue Crawler: 스키마 수집·자동화

### 크롤러 스코프 최소화

- `raw` 쪽에만 적용하거나, `refined`도 필요 시 별도 크롤러
- **Include path**를 좁게, **Exclusions**로 불필요 파일 제외(`*.tmp`, `_SUCCESS`)

### 커스텀 분류기(Classifier)

정규식, Grok, JSONPath로 스키마 추론 강화.

```json
{
  "JsonClassifier": {
    "Name": "access-json",
    "JsonPath": "$"
  }
}
```

### CLI 생성 예

```bash
aws glue create-crawler \
  --name access-raw-crawler \
  --role arn:aws:iam::123456789012:role/glue-crawler-role \
  --database-name db_raw \
  --targets S3Targets=[{Path="s3://datalake/raw/access_logs/"}] \
  --schema-change-policy UpdateBehavior=UPDATE_IN_DATABASE,DeleteBehavior=LOG
```

---

## Glue ETL(Job/Studio): 정제·포맷 변환

### 예제: 원천→정제(Parquet) + 파티션 작성

```python
import sys
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from pyspark.sql.functions import col, to_timestamp, regexp_extract

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'src', 'dst'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext); job.init(args['JOB_NAME'], args)

# 로드 (원본 텍스트/CSV/JSON 가정)

df = spark.read.json(args['src'])  # 예: s3://datalake/raw/access_logs/...

# 정제/타이핑

df2 = (df
   .withColumn("status_code", col("status").cast("int"))
   .withColumn("bytes_sent", col("size").cast("long"))
   .withColumn("ts", to_timestamp(col("time"), "yyyy-MM-dd'T'HH:mm:ssX"))
   .withColumn("dt", col("ts").cast("date").cast("string"))
   .withColumn("hour", regexp_extract(col("ts").cast("string"), "T(\\d{2})", 1))
   .select("client_ip","method","path","status_code","bytes_sent","user_agent","ts","dt","hour"))

# 쓰기(Parquet, 파티션)

(df2.write
   .mode("append")
   .partitionBy("dt","hour")
   .format("parquet")
   .option("compression","snappy")
   .save(args['dst']))  # s3://datalake/refined/access_logs/

job.commit()
```

**포인트**
- ETL에서 **파티션 디렉토리**를 **명시적으로** 생성
- 스키마/타입을 **엄격히** 변환하고 저장

### 파티션 메타 반영

ETL 완료 후:
```sql
MSCK REPAIR TABLE db_refined.access_logs; -- 디렉토리 스캔 후 파티션 반영
-- 또는
ALTER TABLE db_refined.access_logs ADD IF NOT EXISTS
  PARTITION (dt='2025-11-10', hour='02')
  LOCATION 's3://datalake/refined/access_logs/dt=2025-11-10/hour=02/';
```

대량 파티션: **파티션 프로젝션**(아래 §6.3)으로 메타 반영 비용 제거.

---

## Athena: 쿼리, 비용·성능 최적화, 관리

### 결과 저장 버킷/암호화

- WorkGroup별 결과 경로 지정, SSE-KMS 권장
- Settings에서 `enforce` 옵션으로 강제

```text
s3://athena-query-results/prod/
```

### 대표 쿼리

```sql
-- 1) 에러 카운트(파티션 프루닝 필수)
SELECT dt, hour, count(*) AS errors
FROM db_refined.access_logs
WHERE dt='2025-11-10' AND hour BETWEEN '00' AND '03' AND status_code >= 500
GROUP BY dt, hour;

-- 2) 상위 경로
SELECT path, count(*) cnt
FROM db_refined.access_logs
WHERE dt='2025-11-10'
GROUP BY path ORDER BY cnt DESC LIMIT 20;

-- 3) 시간대별 평균 응답 바이트
SELECT dt, hour, avg(bytes_sent) avg_bytes
FROM db_refined.access_logs
WHERE dt BETWEEN '2025-11-01' AND '2025-11-10'
GROUP BY dt, hour
ORDER BY dt, hour;
```

### 비용 추정 감

Athena 과금은 **스캔 바이트** 기준. 간단 모델:
$$
\text{Cost} \approx \frac{\text{Bytes Scanned}}{1~\text{TB}} \times \text{Price per TB}
$$
- `Bytes Scanned` ↓ = **Parquet + 파티션 프루닝 + 필요한 컬럼만** 조회

### Explain/분석

```sql
EXPLAIN SELECT ...;
```
- 파티션 프루닝/프로젝션 적용 여부, 스캔 컬럼 수 확인

---

## 심화: 파티션 프로젝션, CTAS/UNLOAD, Iceberg

### CTAS (Create Table As Select)

정제 결과를 **또 다른 테이블**로 생성(Parquet/압축/정렬).
```sql
CREATE TABLE db_curated.access_top_paths
WITH (
  format = 'PARQUET',
  parquet_compression = 'SNAPPY',
  external_location = 's3://datalake/curated/access_top_paths/',
  partitioned_by = ARRAY['dt']
) AS
SELECT dt, path, count(*) cnt
FROM db_refined.access_logs
WHERE dt BETWEEN '2025-11-01' AND '2025-11-10'
GROUP BY dt, path;
```

### UNLOAD (쿼리 결과를 S3로)

```sql
UNLOAD (SELECT * FROM db_refined.access_logs WHERE dt='2025-11-10')
TO 's3://exports/access_logs_2025-11-10/'
WITH (format = 'PARQUET', compression = 'SNAPPY');
```

### 파티션 프로젝션(메타 자동 추론)

Glue/메타 변경 없이 Athena가 **경로 규칙을 추론**해 파티션 스캔.

```sql
ALTER TABLE db_refined.access_logs
SET TBLPROPERTIES (
  'projection.enabled'='true',
  'projection.dt.type'='date',
  'projection.dt.range'='2025-01-01,NOW',
  'projection.dt.format'='yyyy-MM-dd',
  'projection.hour.type'='integer',
  'projection.hour.range'='00,23',
  'storage.location.template'='s3://datalake/refined/access_logs/dt=${dt}/hour=${hour}/'
);
```

쿼리 시 **WHERE dt/hour** 조건이 **필수**.

### Apache Iceberg 테이블 (ACID/스키마 진화)

Athena는 Iceberg 지원. 머지/스냅샷/타임트래블 등.

```sql
CREATE TABLE db_refined.events_iceberg (
  id bigint,
  ts timestamp,
  payload struct<...>
)
PARTITIONED BY (dt)
LOCATION 's3://datalake/iceberg/events/'
TBLPROPERTIES ('table_type'='ICEBERG');
```

스키마 진화:
```sql
ALTER TABLE db_refined.events_iceberg ADD COLUMN new_col string;
```

---

## 권한/보안: IAM, KMS, Lake Formation

### IAM 최소권한

- Glue Crawler/Job Role: S3 경로 제한, Glue Catalog 특정 DB/테이블 권한
- Athena WorkGroup 실행 권한 분리

예시(요지):
```json
{
  "Version":"2012-10-17",
  "Statement":[
    {"Effect":"Allow","Action":["glue:GetDatabase","glue:GetTable","glue:UpdateTable"],"Resource":"*"},
    {"Effect":"Allow","Action":["s3:ListBucket"],"Resource":"arn:aws:s3:::datalake"},
    {"Effect":"Allow","Action":["s3:GetObject","s3:PutObject"],"Resource":"arn:aws:s3:::datalake/*"}
  ]
}
```

### 암호화

- S3: **SSE-KMS**(버킷 기본 암호화)
- Glue/Athena: 결과/임시 파일 KMS 키 지정
- KMS 키 정책에 ETL/Lake Formation/Athena Role 허용

### Lake Formation

- 테이블·컬럼 단위 권한 부여, 데이터 마스킹/행 필터
- Glue Catalog 권한 관리를 Lake Formation 단일 진입점으로 일원화

---

## 운영 자동화: 워크플로, 스케줄, 파이프라인

### Glue Workflow/Trigger

- **크롤러 → ETL → 파티션 메타 반영** 순의 DAG 구성
- 실패/성공 분기, 재시도 정책

### IaC 예(CloudFormation 스니펫)

```yaml
Resources:
  GlueDatabase:
    Type: AWS::Glue::Database
    Properties:
      CatalogId: 123456789012
      DatabaseInput:
        Name: db_refined

  GlueCrawler:
    Type: AWS::Glue::Crawler
    Properties:
      Name: access-raw-crawler
      Role: arn:aws:iam::123456789012:role/glue-crawler-role
      DatabaseName: db_raw
      Targets:
        S3Targets:
          - Path: s3://datalake/raw/access_logs/
```

### Step Functions

크롤러→잡→MSCK→검증→알림까지 오케스트레이션.

---

## 모니터링/품질/카탈로그 거버넌스

- Glue Job Metrics: Spark UI/CloudWatch 지표(입출력, task 실패)
- Athena WorkGroup: 쿼리/스캔 바이트 대시보드, 가드레일(쿼리 제한)
- **Data Quality**: Glue Data Quality 룰, 또는 쿼리 기반 검증 테이블 운영

예: 단순 카드inality 체크
```sql
SELECT count(DISTINCT client_ip) FROM db_refined.access_logs WHERE dt='2025-11-10';
```

---

## 트러블슈팅 모음

| 문제 | 원인 | 해결 |
|---|---|---|
| 테이블은 있는데 결과 0건 | 파티션 미등록 | `MSCK` 또는 프로젝션 설정 및 WHERE 파티션 조건 |
| 스캔 바이트 과다 | CSV/`SELECT *` | Parquet 변환, 필요한 컬럼만 조회, 프루닝 |
| 크롤러가 스키마 자주 바꿈 | 원천 변동/혼합 파일 | 경로 분리, Classifier, 스키마 레벨 고정 |
| 타입 캐스팅 실패 | 문자열/타입 혼재 | ETL에서 명시 캐스팅/정규화 |
| 권한 에러 | KMS/S3/LF 권한 부족 | IAM/LF/KMS 정책 교차 점검 |
| 작은 파일 폭증 | 마이크로배치 | Glue에서 **repartition/coalesce**, 파일 크기 정규화 |

---

## 비용 최적화 레시피

- Parquet(+Snappy), **컬럼·파티션 프루닝**
- 파티션 프로젝션으로 메타 관리 비용↓
- WorkGroup별 **쿼리 제한/알림**, 가드레일
- ETL 시 **소파일 컴팩션**, 스냅샷/아카이브 주기 설정
- **데이터 레이아웃**: 쿼리 패턴과 파티션 키 정렬

---

## 실전 예제: Nginx 액세스 로그 → Refined Parquet → Athena 대시보드

### 원천 업로드(예)

```
s3://datalake/raw/access_logs/2025/11/10/10-00.json
```

샘플
```
{"client_ip":"192.168.1.1","method":"GET","path":"/","status":200,"size":512,"time":"2025-11-10T10:00:00Z","ua":"Mozilla/5.0"}
```

### Glue Job 파라미터

```bash
--src s3://datalake/raw/access_logs/ \
--dst s3://datalake/refined/access_logs/
```

### 파티션 메타

```sql
MSCK REPAIR TABLE db_refined.access_logs;
```

### Athena 질의

```sql
-- 시간대별 에러율
WITH base AS (
  SELECT dt, hour, count(*) total,
         sum(CASE WHEN status_code>=500 THEN 1 ELSE 0 END) err
  FROM db_refined.access_logs
  WHERE dt BETWEEN '2025-11-08' AND '2025-11-10'
  GROUP BY dt, hour
)
SELECT dt, hour, err*1.0/total AS error_rate
FROM base
ORDER BY dt, hour;
```

---

## 검증/리그레션 테스트(개발자 관점)

- **샘플 파티션**에 대해 ETL 실행 → 기대 결과와 **골든 결과** 비교
- 스키마 변경 시 **DDL/뷰/쿼리 호환성** 회귀 테스트
- 커스텀 UDF/정규화 로직은 **유닛/통합 테스트**로 고정

---

## 수학적 감: 스캔 바이트와 파티션 프루닝 효과

$$
\text{Bytes Scanned} \approx \sum_{f \in \text{Files Read}} \text{FileSize}(f) \times \text{Selectivity}(columns)
$$

- Parquet에서 **Selectivity(columns)** 는 필요한 컬럼만 로드할 때 크게 감소.
- 파티션 프루닝으로 **Files Read** 자체를 급감.

---

## 요약 체크리스트 (현장 적용)

- [ ] **S3 파티션/포맷**: Parquet, 128–512MB, 키=값 디렉토리
- [ ] **Glue Crawler**: 범위 최소화, Classifier, 스키마 안정화
- [ ] **ETL**: 명시적 캐스팅, 파티션 생성, 소파일 방지
- [ ] **Catalog/DDL**: 파티션 정의, 프로젝션, CTAS/UNLOAD 활용
- [ ] **Athena**: WorkGroup, 결과 버킷/KMS, Explain, 프루닝
- [ ] **보안**: S3 BPA, SSE-KMS, 최소권한 IAM, Lake Formation
- [ ] **운영**: Workflow/Trigger/Step Functions, 모니터링/가드레일
- [ ] **코스트**: 스캔 바이트 절감(포맷/파티션/컬럼 제한)

---

## Athena JSON 테이블(원천)

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS db_raw.access_logs_json (
  client_ip string, method string, path string,
  status int, size bigint, time string, ua string
)
PARTITIONED BY (ingest_dt string)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://datalake/raw/access_logs/'
TBLPROPERTIES ('has_encrypted_data'='false');
```

## Glue Job 북마크(증분 처리)

```python
datasource0 = glueContext.create_dynamic_frame.from_options(
  connection_type="s3",
  connection_options={"paths":[args['src']], "recurse":True, "groupFiles":"inPartition"},
  format="json", transformation_ctx="datasource0")

# Job Bookmark를 켜면 이전 처리 지점 이후만 읽음(Glue Job 옵션에서 활성화)

```

## Athena WorkGroup 예(가드레일)

```bash
aws athena create-work-group \
  --name "prod-analytics" \
  --configuration-result-configuration OutputLocation="s3://athena-query-results/prod/",EncryptionConfiguration={EncryptionOption=SSE_KMS,KmsKey="arn:aws:kms:..."} \
  --state ENABLED \
  --description "Production analytics group"
```

---

### 결론

Glue + Athena는 **S3 데이터 레이크**의 중심축이다. 올바른 **레이아웃/파티션/포맷**과 **카탈로그/ETL 운영 규율**만 지키면, 대규모 로그/이벤트 데이터도 **서버리스·저비용**으로 실시간에 가깝게 분석할 수 있다. 본 문서의 설계·운영 패턴을 템플릿처럼 적용해 **처음부터 비용과 성능, 보안과 거버넌스를 모두 갖춘** 분석 환경을 구현하자.
~~~markdown
