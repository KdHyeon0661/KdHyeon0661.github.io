---
layout: post
title: AWS - Firehose와 S3 연동
date: 2025-07-28 22:20:23 +0900
category: AWS
---
# Kinesis Data Firehose → S3 연동

## 0) 한눈에 보는 아키텍처

```
[Producers: App/Agent/IoT/KDS] --(PutRecord/Batched)-->
   [Kinesis Data Firehose Delivery Stream]
        |--(Optional Lambda Transform)--> [Format: JSON/CSV/Parquet]
        |--(Buffer: Size/Interval, Compression: GZIP/ZSTD/Snappy)-->
        |--(Dynamic Partitioning: Keys, Prefix Templates)-->
        +--> [S3 Data Lake Landing Zone]
                ├─ raw/  (원본, 압축)
                ├─ refined/ (정제, Parquet)
                └─ _error/ (변환 실패/파싱 오류 백업)
```

**핵심 포인트**
- **버퍼링**: `Buffer size`(MiB) 또는 `Buffer interval`(초) 충족 시 S3로 플러시  
- **변환**: **Lambda**(자유로운 ETL) 또는 **원클릭 Parquet/ORC 변환 + Glue 스키마**  
- **동적 파티셔닝**: 레코드의 키 값으로 S3 폴더(파티션) 분배 → Athena/Spark 쿼리 최적화  
- **백업**: 실패 레코드는 별도 S3 prefix로 **보존/재처리**  
- **보안**: S3 SSE-KMS, 전달 스트림 IAM Role 최소권한, VPC 엔드포인트(PrivateLink)

---

## 1) 소스 옵션(Direct PUT vs Kinesis Data Streams)

| 소스 | 특징 | 권장 상황 |
|---|---|---|
| **Direct PUT** | 애플리케이션에서 Firehose에 직접 전송. 단순·저지연 | 트래픽 예측 가능, 재처리 필요 낮음 |
| **Kinesis Data Streams(KDS)** | KDS로 먼저 수집(Shard 확장/리플레이) 후 Firehose가 폴링 | 급격한 버스트·재처리·샤딩 관리 필요 |

**레코드 크기 제한**: PutRecord **최대 ~1MB**(권장: 수십 KB). 대형 페이로드는 **압축/분할** 권장.

---

## 2) S3 대상 설계(폴더 구조·프리픽스·압축·포맷)

### 2.1 S3 Prefix(폴더) 설계

- **시간 파티션 + 도메인 키**를 조합한 프리픽스가 운영/분석 모두에 유리:
```
s3://data-lake/landing/app=web/region=kr/yyyymmdd=2025-11-10/hh=02/
```

**Firehose Prefix 템플릿 예**
```
app=!{partitionKeyFromQuery:app}/region=!{partitionKeyFromQuery:region}/yyyymmdd=!{timestamp:yyyy-MM-dd}/hh=!{timestamp:HH}/
```

> 동적 파티셔닝과 템플릿은 **쿼리 파티션 프리뷰**(미리보기)로 검증 권장.

### 2.2 압축·포맷 선택

| 포맷 | 장점 | 단점 | 권장 |
|---|---|---|---|
| **JSON (GZIP)** | 단순, 유연 | 스캔 비용 큼 | 초기/탐색 |
| **Parquet (Snappy)** | 열 지향, 압축 효율·스캔 비용↓ | 스키마 관리 필요 | 분석/장기 |

**권장 조합**: `Parquet + Snappy` + **Glue 스키마** → Athena·EMR·Spark 최적화

---

## 3) IAM·S3 정책(필수 권한 템플릿)

### 3.1 Firehose 전송 역할(IAM Role)

- 신뢰 정책(Trust): `firehose.amazonaws.com`
- S3 PutObject, GetBucketLocation, ListBucket 등 최소 권한
- KMS 사용 시 `kms:Encrypt/Decrypt/GenerateDataKey` (리소스 기반에 역할 허용 필요)

```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"S3Write",
      "Effect":"Allow",
      "Action":[ "s3:PutObject", "s3:AbortMultipartUpload", "s3:ListBucket", "s3:GetBucketLocation" ],
      "Resource":[
        "arn:aws:s3:::data-lake",
        "arn:aws:s3:::data-lake/*"
      ]
    },
    {
      "Sid":"KMSForSSE",
      "Effect":"Allow",
      "Action":[ "kms:Encrypt","kms:Decrypt","kms:GenerateDataKey","kms:DescribeKey" ],
      "Resource":"arn:aws:kms:ap-northeast-2:123456789012:key/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    }
  ]
}
```

### 3.2 S3 버킷 정책(전달 역할 허용)

```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"AllowFirehoseWrite",
      "Effect":"Allow",
      "Principal":{ "AWS":"arn:aws:iam::123456789012:role/firehose-delivery-role" },
      "Action":["s3:PutObject","s3:AbortMultipartUpload","s3:ListBucket","s3:GetBucketLocation"],
      "Resource":[
        "arn:aws:s3:::data-lake",
        "arn:aws:s3:::data-lake/*"
      ]
    }
  ]
}
```

> **주의**: S3 객체 ARN(`.../*`)과 버킷 ARN(`...`) 모두 포함.

---

## 4) 버퍼링·전달 파라미터(튜닝 전략)

- **Buffer Size**: 1~128MiB (기본 5MiB)  
- **Buffer Interval**: 60~900초 (기본 300초)

**튜닝 원칙**  
- 실시간성↑ → **Interval↓**(파일 많아짐, S3 PUT 비용↑)  
- 비용/쿼리 효율↑ → **Size↑**(파일 크기↑, 쿼리 파일 수↓)  

**시간당 파일 수 근사**  
$$
\text{files/hour} \approx \frac{3600}{\text{BufferInterval}} \times \text{Shards(or parallelism)}
$$

---

## 5) Lambda 변환(ETL) — PII 마스킹·정규화

### 5.1 입력·출력 계약

- 입력: `event.records[]` (Base64 인코딩 데이터, ID, 파티션키 등)
- 출력: 각 레코드별 `recordId`, `result`(`Ok`/`Dropped`/`ProcessingFailed`), `data`

### 5.2 예제: 이메일·카드번호 마스킹 + 파티션키 추출

```python
import base64, json, re

EMAIL = re.compile(r'([A-Za-z0-9._%+-]+)@([A-Za-z0-9.-]+\.[A-Za-z]{2,})')
CARD  = re.compile(r'\b(\d{4})\d{8,11}(\d{4})\b')

def mask(text: str) -> str:
    text = EMAIL.sub(lambda m: m.group(1)[:3] + "****@" + m.group(2), text)
    text = CARD.sub(lambda m: f"{m.group(1)}********{m.group(2)}", text)
    return text

def lambda_handler(event, context):
    out = []
    for rec in event['records']:
        raw = base64.b64decode(rec['data']).decode('utf-8', 'ignore')
        try:
            obj = json.loads(raw)
            # 예: 필수 필드 없으면 드롭
            if 'app' not in obj or 'region' not in obj:
                out.append({"recordId": rec['recordId'], "result":"Dropped"})
                continue
            obj['message'] = mask(obj.get('message',''))
            # 동적 파티셔닝 키 힌트(추출 파티션 키)
            # Firehose Dynamic Partitioning: metadata에 추가하는 구현도 가능(서비스 업데이트사항에 따라)
            obj['__partitionKeys'] = { "app": obj['app'], "region": obj['region'] }

            data = base64.b64encode(json.dumps(obj, ensure_ascii=False).encode()).decode()
            out.append({"recordId": rec['recordId'], "result":"Ok", "data": data})
        except Exception:
            out.append({"recordId": rec['recordId'], "result":"ProcessingFailed"})
    return {"records": out}
```

**운영 팁**
- `Dropped` 비율 상승 시 **에러 prefix** 확인·스키마 점검  
- `ProcessingFailed` → Firehose가 **백업 S3**로 원본 저장 (재처리 파이프라인 연결)

---

## 6) Parquet 변환 + Glue 스키마 레지스트리

- 콘솔/CLI에서 **Record format conversion** 활성화 → 출력 포맷 Parquet/ORC 지정  
- **Glue Schemas**에서 **JSON→Avro→Parquet** 변환 파이프라인을 자동화  
- 스키마 진화: **필드 추가**는 하위호환(기본값), **삭제/타입 변경**은 새 버전

**선언형 스키마(예시, Avro)**
```json
{
  "type":"record",
  "name":"WebLog",
  "namespace":"com.myorg",
  "fields":[
    {"name":"ts","type":"long"},
    {"name":"app","type":"string"},
    {"name":"region","type":"string"},
    {"name":"userId","type":["null","string"],"default":null},
    {"name":"message","type":["null","string"],"default":null}
  ]
}
```

---

## 7) 동적 파티셔닝(Dynamic Partitioning)

- 레코드 내 키(예: `app`, `region`)를 파티션 키로 사용하여 S3 프리픽스 분배  
- **장점**: 파티션 프루닝으로 Athena·Spark 비용/속도 개선  
- **주의**: 키 **카디널리티** 과도할 경우 **소파일 폭증** → 상위 수준에서 **버킷팅**·**롤업** 병행

**설정 아이디어**
- 파티션 키 추출(Lambda 변환 시 `__partitionKeys`)  
- 프리픽스 템플릿: `app=!{partitionKeyFromQuery:app}/region=!{partitionKeyFromQuery:region}/yyyymmdd=!{timestamp:yyyy-MM-dd}/`

---

## 8) CLI로 빠른 실습(Direct PUT)

### 8.1 전달 스트림 생성(JSON→GZIP→S3)

```bash
aws firehose create-delivery-stream \
  --delivery-stream-name web-logs \
  --delivery-stream-type DirectPut \
  --s3-destination-configuration '{
    "BucketARN":"arn:aws:s3:::data-lake",
    "RoleARN":"arn:aws:iam::123456789012:role/firehose-delivery-role",
    "Prefix":"landing/app=web/yyyymmdd=!{timestamp:yyyy-MM-dd}/hh=!{timestamp:HH}/",
    "ErrorOutputPrefix":"_error/!{firehose:error-output-type}/!{timestamp:yyyy/MM/dd}/",
    "BufferingHints":{"SizeInMBs":32,"IntervalInSeconds":120},
    "CompressionFormat":"GZIP",
    "EncryptionConfiguration":{"KMSEncryptionConfig":{"AWSKMSKeyARN":"arn:aws:kms:ap-northeast-2:123456789012:key/xxxx"}},
    "CloudWatchLoggingOptions":{"Enabled":true,"LogGroupName":"/aws/kinesisfirehose/web-logs","LogStreamName":"delivery"}
  }'
```

### 8.2 테스트 전송

```bash
aws firehose put-record \
  --delivery-stream-name web-logs \
  --record Data='{"ts":1731186600,"app":"web","region":"kr","userId":"u-77","message":"user foo@example.com paid card 1234567812345678"}\n'
```

> 1~2분 내 S3에 `landing/app=web/yyyymmdd=.../hh=...` 경로로 `.gz` 파일 생성 확인.

---

## 9) 모니터링·알람(CloudWatch)

**주요 메트릭**
- `DeliveryToS3.Records`, `DeliveryToS3.DataFreshness`, `DeliveryToS3.Success`, `DeliveryToS3.Failures`
- 변환 실패: `DataReadFromLambda.Bytes`, `LambdaUserErrors`, `LambdaThrottledRecords`
- 쓰로틀: `ThrottledRecords`, `BackupToS3.Success/Failures`

**지연 알람(예: DataFreshness > 5분)**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "FirehoseFreshnessHigh" \
  --metric-name DeliveryToS3.DataFreshness \
  --namespace AWS/Firehose \
  --dimensions Name=DeliveryStreamName,Value=web-logs \
  --statistic Average \
  --period 60 \
  --threshold 300 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:ap-northeast-2:123456789012:OpsAlert
```

---

## 10) 분석: Athena DDL & CTAS

### 10.1 Hive 스타일 파티션 테이블(JSON)

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS dlf.web_logs_json (
  ts bigint,
  app string,
  region string,
  userId string,
  message string
)
PARTITIONED BY (yyyymmdd string, hh string)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://data-lake/landing/app=web/';
```

**파티션 로드**
```sql
MSCK REPAIR TABLE dlf.web_logs_json;
```

### 10.2 Parquet 변환(CTAS)

```sql
CREATE TABLE dlf.web_logs_pq
WITH (
  format='PARQUET',
  parquet_compression='SNAPPY',
  external_location='s3://data-lake/refined/app=web/'
) AS
SELECT ts, app, region, userId, message, yyyymmdd, hh
FROM dlf.web_logs_json
WHERE yyyymmdd='2025-11-10';
```

---

## 11) IaC 모음

### 11.1 Terraform(핵심만)

```hcl
resource "aws_kinesis_firehose_delivery_stream" "web_logs" {
  name        = "web-logs"
  destination = "s3"

  s3_configuration {
    role_arn           = aws_iam_role.firehose.arn
    bucket_arn         = aws_s3_bucket.datalake.arn
    buffering_size     = 32
    buffering_interval = 120
    compression_format = "GZIP"
    prefix             = "landing/app=web/yyyymmdd=!{timestamp:yyyy-MM-dd}/hh=!{timestamp:HH}/"
    error_output_prefix= "_error/!{firehose:error-output-type}/!{timestamp:yyyy/MM/dd}/"

    cloudwatch_logging_options {
      enabled         = true
      log_group_name  = "/aws/kinesisfirehose/web-logs"
      log_stream_name = "delivery"
    }
  }
}
```

### 11.2 AWS SAM(Parquet 변환 + Lambda ETL)

```yaml
Transform: AWS::Serverless-2016-10-31
Resources:
  TransformFn:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: etl/
      Handler: app.lambda_handler
      Runtime: python3.12
      Policies: AWSLambdaBasicExecutionRole

  DeliveryStream:
    Type: AWS::KinesisFirehose::DeliveryStream
    Properties:
      DeliveryStreamType: DirectPut
      ExtendedS3DestinationConfiguration:
        BucketARN: arn:aws:s3:::data-lake
        RoleARN: arn:aws:iam::123456789012:role/firehose-delivery-role
        Prefix: landing/app=web/yyyymmdd=!{timestamp:yyyy-MM-dd}/hh=!{timestamp:HH}/
        ErrorOutputPrefix: _error/!{firehose:error-output-type}/!{timestamp:yyyy/MM/dd}/
        BufferingHints: { IntervalInSeconds: 120, SizeInMBs: 32 }
        CompressionFormat: UNCOMPRESSED
        DataFormatConversionConfiguration:
          SchemaConfiguration:
            CatalogId: "123456789012"
            DatabaseName: "glue_db"
            TableName: "web_log_schema"
            RoleARN: arn:aws:iam::123456789012:role/firehose-delivery-role
          InputFormatConfiguration:
            Deserializer: { OpenXJsonSerDe: {} }
          OutputFormatConfiguration:
            Serializer: { ParquetSerDe: { Compression: "SNAPPY" } }
          Enabled: true
        ProcessingConfiguration:
          Enabled: true
          Processors:
            - Type: Lambda
              Parameters:
                - ParameterName: LambdaArn
                  ParameterValue: !GetAtt TransformFn.Arn
```

### 11.3 CDK(Python, 요약)

```python
from aws_cdk import aws_s3 as s3, aws_iam as iam, aws_kinesisfirehose as fh, Stack
from constructs import Construct

class FirehoseStack(Stack):
    def __init__(self, scope: Construct, id: str, **kw):
        super().__init__(scope, id, **kw)
        bucket = s3.Bucket(self, "DataLake")
        role = iam.Role(self, "FirehoseRole",
                        assumed_by=iam.ServicePrincipal("firehose.amazonaws.com"))
        bucket.grant_write(role)

        fh.CfnDeliveryStream(self, "WebLogs",
            delivery_stream_type="DirectPut",
            extended_s3_destination_configuration=fh.CfnDeliveryStream.ExtendedS3DestinationConfigurationProperty(
                bucket_arn=bucket.bucket_arn,
                role_arn=role.role_arn,
                prefix="landing/app=web/yyyymmdd=!{timestamp:yyyy-MM-dd}/hh=!{timestamp:HH}/",
                buffering_hints=fh.CfnDeliveryStream.BufferingHintsProperty(
                    interval_in_seconds=120, size_in_m_bs=32),
                compression_format="GZIP"
            )
        )
```

### 11.4 CloudFormation(요약)

```yaml
Resources:
  WebLogs:
    Type: AWS::KinesisFirehose::DeliveryStream
    Properties:
      DeliveryStreamType: DirectPut
      ExtendedS3DestinationConfiguration:
        BucketARN: arn:aws:s3:::data-lake
        RoleARN: arn:aws:iam::123456789012:role/firehose-delivery-role
        Prefix: landing/app=web/yyyymmdd=!{timestamp:yyyy-MM-dd}/hh=!{timestamp:HH}/
        BufferingHints: { SizeInMBs: 32, IntervalInSeconds: 120 }
        CompressionFormat: GZIP
```

---

## 12) 에이전트·수집기(Fluent Bit/Kinesis Agent)

### 12.1 Fluent Bit → Firehose(HTTP)

```ini
[INPUT]
    Name              tail
    Path              /var/log/nginx/access.log
    Parser            apache2

[OUTPUT]
    Name              kinesis_firehose
    Match             *
    region            ap-northeast-2
    delivery_stream   web-logs
    data_keys         ts,app,region,userId,message
    log_key           message
```

> IAM Role 또는 자격증명 제공 필요. 네트워크는 **VPC 엔드포인트**로 사설 경로 권장.

---

## 13) 백업·에러 경로·재처리

- **ErrorOutputPrefix** 하위에 Firehose가 실패 유형별 폴더 생성(예: 변환 실패/대상 실패)  
- 재처리 파이프라인:
  1) `_error/` 경로를 **S3 Event** → **SQS** → **Lambda 보정**  
  2) 보정 완료 데이터를 **새 Delivery Stream**(정제 경로)로 재주입

---

## 14) 보안·네트워킹

- **SSE-KMS**: S3·Firehose 모두 KMS 키 사용, **키 정책**에 Firehose Role 허용  
- **VPC 엔드포인트(Interface/Gateway)**: S3·Firehose PrivateLink → 인터넷 없이 전송  
- **S3 Block Public Access**: 데이터 레이크 버킷 기본 ON  
- **액세스 로그**: S3 서버 액세스 로그 또는 CloudTrail + Lake Formation 로깅

---

## 15) 비용 모델 & 계산 감각

- Firehose **수집/전송 요금**: GB당  
- **변환 비용**: Parquet/ORC 변환 사용 시 추가 과금  
- **Lambda 변환**: GB-초·호출 수  
- **S3 저장**: 스토리지·요청·전송  
- **Athena 쿼리**: 스캔 데이터량(압축/열지향 포맷으로 최소화)

**간단 계산 예**  
$$
\text{월 Firehose 비용} \approx \text{수집 GB} \times \text{단가} + \text{포맷 변환료}
$$

---

## 16) 성능·운영 베스트 프랙티스

- **소파일 문제**: 버퍼 사이즈↑, Interval↑, 다운스트림 **Compact Job(CTAS)** 주기화  
- **스키마 진화**: 필드 추가 중심, 중대한 변경은 **새 경로/테이블**  
- **동적 파티션 카디널리티 관리**: 상위 파티션(날짜/시간) → 하위(도메인 키)  
- **에러율 모니터링**: `ProcessingFailed/Dropped` 비율 알람  
- **IaC 표준화**: 스테이지(dev/test/prod) 간 prefix·KMS 키·파티션 정책 일관성

---

## 17) 트러블슈팅 체크리스트

- [ ] S3 권한 오류: 버킷/오브젝트 ARN, 역할 Principal, KMS 키 정책 점검  
- [ ] 변환 실패 급증: Lambda 타임아웃/메모리/로그, 레코드 크기·인코딩 확인  
- [ ] 지연 상승: `DataFreshness`, 버퍼 튜닝, 대상 S3 리전 일치 여부  
- [ ] 소파일 과다: Buffer/Interval 상향, 파티션 키 축소·롤업 파이프라인  
- [ ] KMS 권한: `kms:GenerateDataKey` 누락 여부

---

## 18) 엔드투엔드 실전 시나리오(요약)

**목표**: 웹 액세스 로그 → Firehose → S3(Parquet, 파티션) → Athena 분석

1) S3 `data-lake` 버킷 생성, SSE-KMS·Block Public Access ON  
2) Firehose 전달 스트림(`web-logs`) 생성:  
   - Buffer 32MiB/120s, Compression GZIP  
   - Data format conversion → Parquet(SNAPPY) + Glue 스키마  
   - Prefix: `landing/app=web/yyyymmdd=!{timestamp:yyyy-MM-dd}/hh=!{timestamp:HH}/`  
   - ErrorOutputPrefix: `_error/...`  
3) Fluent Bit 또는 CLI로 레코드 전송  
4) Athena DDL 작성 → `MSCK REPAIR TABLE`  
5) 대시보드(SQL): 시간대별 요청수, 상위 IP/URI, 에러율  
6) CloudWatch 알람: Freshness>5분, Failures>0  
7) 비용 점검: 스캔량 관찰 → 추가 파티션/압축·Parquet 검토

---

## 19) 참고 쿼리(Athena)

**시간대별 요청 수**
```sql
SELECT yyyymmdd, hh, count(*) AS cnt
FROM dlf.web_logs_pq
WHERE yyyymmdd BETWEEN '2025-11-09' AND '2025-11-10'
GROUP BY 1,2
ORDER BY 1,2;
```

**메시지 내 에러 키워드**
```sql
SELECT yyyymmdd, hh, regexp_extract(message, '(ERROR|WARN)', 1) AS level, count(*) c
FROM dlf.web_logs_json
WHERE yyyymmdd='2025-11-10'
GROUP BY 1,2,3
ORDER BY c DESC;
```

---

## 20) 마무리 요약

| 영역 | 핵심 |
|---|---|
| 수집 | Direct PUT(단순) vs KDS(버스트/리플레이) |
| 저장 | 시간/도메인 기반 프리픽스 + **동적 파티셔닝** |
| 포맷 | **Parquet+Snappy** + Glue 스키마(분석 최적) |
| 변환 | Lambda로 **PII 마스킹/정규화**, 실패는 `_error/`로 백업 |
| 보안 | SSE-KMS, 버킷 정책·역할 최소권한, VPC 엔드포인트 |
| 운영 | DataFreshness/Failures 알람, 소파일·스키마 진화 관리 |
| 비용 | 압축·열지향 포맷·파티션 프루닝으로 **스캔 최소화** |

**결론**: Firehose→S3 파이프라인은 **단순·탄탄한 실시간 데이터 레이크 인제스트 표준**이다. 이 글의 설계/보안/운영/IaC 템플릿을 적용하면 **안정성과 비용 효율**을 동시에 달성할 수 있다.