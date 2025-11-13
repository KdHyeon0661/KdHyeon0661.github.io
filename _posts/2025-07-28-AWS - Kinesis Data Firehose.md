---
layout: post
title: AWS - Kinesis Data Firehose
date: 2025-07-28 17:20:23 +0900
category: AWS
---
# AWS Kinesis Data Firehose 기반 ETL 파이프라인 설계

## 0. 전체 흐름(확장판)

```text
[Producers: App/Batch/IoT/Fluent Bit/CloudFront/ALB Logs]
      │  (PutRecord/PutRecordBatch/Agent)
      ▼
  [Kinesis Data Firehose Delivery Stream]
      ├─ (옵션) Lambda Record Transformation
      ├─ (옵션) Data format conversion (JSON→Parquet/ORC, Glue Schema Registry)
      ├─ (옵션) Dynamic Partitioning (S3 prefix by keys)
      ├─ Retry & Backup to S3 (failures)
      ▼
Destinations:
  • S3 Data Lake (partitioned, compressed)
  • Redshift (COPY via S3 staging)
  • OpenSearch (indexing)
  • HTTP Endpoint (3rd party SIEM/obs)
```

---

## 1. Firehose 핵심 개념 정리

### 1.1 소스(Source) 유형
- **Direct PUT**: 애플리케이션에서 Firehose API로 직접 전송
- **Kinesis Data Streams → Firehose**: Streams를 소스로 연결(샤드 스케일링을 Streams에서)
- **CloudWatch Logs/Events, VPC Flow Logs, CloudFront/ALB Access Logs**: 서비스 통합 경로(리전별 지원)

### 1.2 버퍼링·배치
- **Size**: 1–128 MiB (S3/OpenSearch 기준), 보통 16–64 MiB 권장
- **Interval**: 60–900 s(기본 300s). 저지연 vs 비용(요청/오브젝트 수) 트레이드오프

### 1.3 포맷 변환 & 스키마
- **데이터 포맷 변환**: JSON/CSV → **Parquet/ORC** (열지향, 압축 효율↑, Athena/Redshift Spectrum 친화)
- **Glue Schema Registry**: 스키마 버전 관리(Avro/JSON/Protobuf), 진화 대응

### 1.4 동적 파티셔닝 (S3)
- 레코드의 필드로 **Prefix**를 동적으로 구성: `year=!{timestamp:YYYY}/month=!{timestamp:MM}/country=!{partitionKeyFromQuery:$.country}/`
- Athena/Glue 파티션 비용·성능 최적화의 핵심

### 1.5 변환(Transformation)
- **Lambda Transform**: 레코드 단위/배치 단위 변환, PII 마스킹, 필터링, 정규화
- 입력/출력은 **Base64 인코딩** 컨트랙트 필수

---

## 2. IAM 최소 권한 설계(샘플)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:PutObject","s3:AbortMultipartUpload","s3:ListBucket","s3:GetBucketLocation"], "Resource": ["arn:aws:s3:::firehose-etl-demo-bucket","arn:aws:s3:::firehose-etl-demo-bucket/*"] },
    { "Effect": "Allow", "Action": ["lambda:InvokeFunction","lambda:GetFunctionConfiguration"], "Resource": "arn:aws:lambda:ap-northeast-2:123456789012:function:TransformETLFunction" },
    { "Effect": "Allow", "Action": ["logs:PutLogEvents","logs:CreateLogStream","logs:CreateLogGroup"], "Resource": "*" },
    { "Effect": "Allow", "Action": ["kms:Encrypt","kms:Decrypt","kms:GenerateDataKey"], "Resource": "arn:aws:kms:ap-northeast-2:123456789012:key/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" }
  ]
}
```

> 최소 권한 원칙: 필요 대상 S3 버킷·KMS 키·Lambda에 한정.

---

## 3. Lambda 변환: 실전 패턴(PII 마스킹, 스키마 진화, 에러 분기)

### 3.1 입력/출력 컨트랙트
- `event.records[]` 각 항목: `{ recordId, data, approximateArrivalTimestamp }`
- 응답: `result`: `Ok` | `Dropped` | `ProcessingFailed`

### 3.2 예제: JSON 정규화 + PII 마스킹 + 동적 파티션 키 세팅
```python
import base64, json, re, hashlib, os

SALT = os.getenv("HASH_SALT", "static-salt")

def mask_email(v: str) -> str:
    if not v: return v
    h = hashlib.sha256((v + SALT).encode()).hexdigest()[:12]
    return f"email_hash_{h}"

def clean_country(v):
    return v.upper() if isinstance(v, str) else "ZZ"

def lambda_handler(event, context):
    out = []
    for rec in event['records']:
        raw = base64.b64decode(rec['data']).decode('utf-8', errors='ignore')
        try:
            js = json.loads(raw)
            # PII 마스킹
            if 'email' in js:
                js['email'] = mask_email(js['email'])
            # 필수 필드 정규화/기본값
            js['country'] = clean_country(js.get('country','ZZ'))
            js['ts'] = js.get('ts') or js.get('timestamp')
            # 스키마 진화: 누락 필드 채움
            js['event_version'] = js.get('event_version', 1)

            data_b64 = base64.b64encode(json.dumps(js, separators=(',',':')).encode()).decode()

            # 동적 파티션 힌트: metadata에 partitionKeys 제공(Record metadata 사용 시)
            out.append({
                'recordId': rec['recordId'],
                'result': 'Ok',
                'data': data_b64,
                'metadata': { 'partitionKeys': { 'country': js['country'] } }
            })
        except Exception:
            # 실패 레코드는 ProcessingFailed → Backup S3로 이동됨
            out.append({
                'recordId': rec['recordId'],
                'result': 'ProcessingFailed',
                'data': rec['data']
            })
    return { 'records': out }
```

> 포인트
> - **Deterministic Hash**로 PII 비가역화
> - **metadata.partitionKeys** 사용 시 Firehose 동적 파티셔닝 Prefix에 매핑 가능
> - 실패는 자동 백업 버킷(or CloudWatch Logs)로 분리

---

## 4. S3 목적지: 포맷 변환/파티셔닝/Glue/Athena

### 4.1 S3 구성 전략
- Prefix 설계: `s3://datalake/events/year=YYYY/month=MM/day=DD/country=KR/`
- 압축: **GZIP**(JSON) 또는 **Snappy**(Parquet)
- 포맷 변환: **Parquet**로 저장 → Athena/Redshift Spectrum 비용·성능 극대화

### 4.2 Firehose S3 구성(JSON)
```json
{
  "RoleARN": "arn:aws:iam::123456789012:role/FirehoseDeliveryRole",
  "BucketARN": "arn:aws:s3:::firehose-etl-demo-bucket",
  "Prefix": "etl-output/year=!{timestamp:YYYY}/month=!{timestamp:MM}/country=!{partitionKeyFromQuery:$.country}/",
  "ErrorOutputPrefix": "etl-errors/!{firehose:error-output-type}/year=!{timestamp:YYYY}/month=!{timestamp:MM}/",
  "BufferingHints": { "SizeInMBs": 32, "IntervalInSeconds": 120 },
  "CompressionFormat": "GZIP",
  "EncryptionConfiguration": { "KMSEncryptionConfig": { "AWSKMSKeyARN": "arn:aws:kms:ap-northeast-2:123456789012:key/xxx" } },
  "ProcessingConfiguration": {
    "Enabled": true,
    "Processors": [
      {
        "Type": "Lambda",
        "Parameters": [{ "ParameterName": "LambdaArn", "ParameterValue": "arn:aws:lambda:ap-northeast-2:123456789012:function:TransformETLFunction:$LATEST" }]
      }
    ]
  },
  "DynamicPartitioningConfiguration": { "Enabled": true, "RetryOptions": { "DurationInSeconds": 300 } }
}
```

### 4.3 S3 + Parquet 변환(Glue Schema Registry)
```json
{
  "DataFormatConversionConfiguration": {
    "Enabled": true,
    "InputFormatConfiguration": { "Deserializer": { "OpenXJsonSerDe": {} } },
    "OutputFormatConfiguration": { "Serializer": { "ParquetSerDe": { "Compression": "SNAPPY" } } },
    "SchemaConfiguration": {
      "Region": "ap-northeast-2",
      "RoleARN": "arn:aws:iam::123456789012:role/FirehoseDeliveryRole",
      "VersionId": "LATEST",
      "CatalogId": "123456789012",
      "DatabaseName": "datalake",
      "TableName": "events_parquet"
    }
  }
}
```

### 4.4 Athena 쿼리 예시
```sql
-- Glue Crawler로 카탈로그 등록 후
SELECT country, action, count(*)
FROM datalake.events_parquet
WHERE year=2025 AND month=11 AND country='KR'
GROUP BY country, action
ORDER BY 3 DESC
LIMIT 20;
```

---

## 5. Redshift 목적지: COPY 자동화

### 5.1 설계 포인트
- Firehose가 **S3 staging → Redshift COPY**를 자동 실행
- Target 테이블 스키마와 S3 포맷 일치, IAM Role의 `redshift:CopyFromS3` 권한 필요
- **원자성**: COPY 실패 시 staging 보존/알람

### 5.2 적재 컬럼 타입과 DIST/SORT KEY
- 사실 테이블: `DISTKEY(user_id)` or AUTO, `SORTKEY(ts)`
- WLM/Auto WLM과 COPY 동시성 고려

---

## 6. OpenSearch 목적지

### 6.1 색인 설계
- Index: `events-YYYY.MM.DD` 롤오버 정책
- Mapping 동적 필드 제한, 키 필드에 `keyword` 타입 적용
- Bulk size/Flush interval은 Firehose가 관리

---

## 7. 오류 처리·백업·재처리

- **Retry**: 네트워크/스로틀링에 대해 내부 재시도
- **Backup Bucket**: 변환 실패/전송 실패를 `etl-errors/`로 백업
- **리플레이**: 백업 버킷→새 Firehose/Batch(Glue/Spark)로 재처리 파이프라인 구축

```text
[Failures S3] → [Glue Job/Spark] → [Cleaned S3] → [Analytics]
```

---

## 8. 관측성(Observability)

### 8.1 CloudWatch 메트릭 핵심
- `DeliveryToS3.Records`, `DeliveryToS3.Success`
- `ThrottledRecords`, `KMSKeyAccessDenied`, `DataFreshness`
- `FailedConversionRecords`, `BackupToS3.Success`

### 8.2 알람 예시(CLI)
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name Firehose-Delivery-Failures \
  --metric-name DeliveryToS3.DataFreshness \
  --namespace AWS/Firehose \
  --statistic Average --period 300 --threshold 300 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:ap-northeast-2:111122223333:Ops
```

---

## 9. 성능·비용 최적화

### 9.1 버퍼·압축·포맷
- S3: **Size 32–64 MiB / Interval 60–180 s** → 적절한 파일 크기(Parquet 128–256MiB)
- Parquet+Snappy → S3 저장·Athena 스캔 비용 절감(10~100배)

### 9.2 동적 파티셔닝 비용 밸런스
- 파티션이 과도하면 **파일 수 증가 → List 비용/작은 파일 문제**
- 시간 파티션(시간/일)+소수 키(country 등) 조합으로 상한 관리

### 9.3 Lambda 변환 한계
- 타임아웃(최대 15분), 동시성, **payload 크기(6MB/레코드)**, 배치 크기
- 변환 로직은 **O(n)** 선형, 외부 호출 최소화. PII 해시는 CPU 바운드 → 메모리 상향

### 9.4 간이 비용 모델
$$
\text{Total Monthly Cost} \approx \text{Firehose Ingest GB} \times p_F
+ \text{Lambda GB-s} \times p_L + \text{S3 Storage GB} \times p_{S3}
+ \text{Athena Scan GB} \times p_A + \text{OpenSearch Ingest/Storage}
$$

> 파레토: **Parquet 전환 + 파티셔닝**이 Athena 비용의 80%를 절감

---

## 10. 보안 설계

- **SSE-KMS**: S3 객체 암호화, Firehose 전송 시 KMS 사용
- **VPC 엔드포인트(Interface/Gateway)**: 사설 경로로 S3/Kinesis/KMS/Logs 접근
- **IAM SCP/Permission Boundary**: Data Lake 버킷 접근 범위 제한
- **데이터 마스킹/토큰화**: Lambda에서 비가역 처리, 원문 저장 금지

---

## 11. IaC: Terraform 스니펫

```hcl
resource "aws_kinesis_firehose_delivery_stream" "etl" {
  name        = "firehose-etl-stream"
  destination = "extended_s3"

  extended_s3_configuration {
    role_arn           = aws_iam_role.firehose_role.arn
    bucket_arn         = aws_s3_bucket.datalake.arn
    buffer_interval    = 120
    buffer_size        = 32
    compression_format = "GZIP"
    prefix             = "etl-output/year=!{timestamp:YYYY}/month=!{timestamp:MM}/country=!{partitionKeyFromQuery:$.country}/"
    error_output_prefix= "etl-errors/!{firehose:error-output-type}/year=!{timestamp:YYYY}/month=!{timestamp:MM}/"

    processing_configuration {
      enabled = true
      processors {
        type = "Lambda"
        parameters {
          parameter_name  = "LambdaArn"
          parameter_value = aws_lambda_function.transform.arn
        }
      }
    }

    dynamic_partitioning_configuration {
      enabled = true
      retry_options { duration_in_seconds = 300 }
    }

    encryption_configuration {
      kms_encryption_config { awskms_key_arn = aws_kms_key.data.arn }
    }
  }
}
```

---

## 12. 테스트 & 프로듀서 예시

### 12.1 Python(Direct PUT)
```python
import boto3, json, base64
firehose = boto3.client('firehose', region_name='ap-northeast-2')

def send(user, action, ts, country):
    rec = {"user": user, "action": action, "ts": ts, "country": country, "email": f"{user}@ex.com"}
    data = (json.dumps(rec) + "\n").encode()
    firehose.put_record(
        DeliveryStreamName="firehose-etl-stream",
        Record={"Data": data}  # Direct PUT는 Base64 필요 X (SDK가 처리)
    )

send("kimdh","login","2025-11-10T02:45:00Z","kr")
```

### 12.2 대량 배치
```python
def put_batch(records):
    entries = [{"Data": (json.dumps(r)+"\n").encode()} for r in records]
    return firehose.put_record_batch(DeliveryStreamName="firehose-etl-stream", Records=entries)
```

---

## 13. 운영 체크리스트

- [ ] S3 Prefix/파티션 키 설계 확정(쿼리 패턴 역설계)
- [ ] Lambda 변환: 타임아웃/동시성/메모리 적정치 튜닝
- [ ] Backup Prefix 모니터링 및 재처리 잡(Glue/Spark) 준비
- [ ] CloudWatch 경보(Success/Failure/DataFreshness/KMS 오류)
- [ ] Glue Crawler 주기 & 스키마 드리프트 대응(새 필드 Nullable)
- [ ] Athena/Redshift 압축·파티션 통계 최신화
- [ ] KMS Key 정책: Firehose/Lambda/Crawler/Athena 모두 허용

---

## 14. 자주 묻는 질문(FAQ)

**Q1. 레코드 순서는 보장되나?**
- Firehose는 **전송 순서 보장하지 않음**. 순서 의존 시 Kinesis Data Streams + 소비자 애플리케이션(Flink/KDA) 고려.

**Q2. 중복 방지는?**
- Firehose 자체 중복제거 없음. **idempotent sink** 설계(예: S3 ObjectKey에 해시 포함/머지 작업) 또는 다운스트림에서 **Dedup**.

**Q3. 작은 파일 문제?**
- Buffer Size/Interval 상향, 다운스트림에서 **Compaction**(Athena CTAS/Glue Job) 수행.

**Q4. PII 보호는 어디서?**
- 입력단(Lambda 변환)에서 **비가역 마스킹**. 원문 저장 금지, 접근 경로 최소화.

---

## 15. 미니 실습 시나리오(엔드투엔드)

1) **S3 버킷** 생성, KMS 키 준비
2) **Lambda 변환** 배포(PII 마스킹/partitionKeys 세팅)
3) **Firehose Delivery Stream** 생성(동적 파티셔닝+GZIP)
4) **샘플 데이터** PutRecord/PutRecordBatch로 주입
5) S3 `etl-output/`에 파일 생성 확인, **Glue Crawler**로 카탈로그 반영
6) **Athena**에서 파티션 프루닝 쿼리로 검증
7) 실패 레코드 `etl-errors/` 모니터링 및 Glue Job 재처리

---

## 16. 결론

- Firehose는 **운영 난이도를 낮춘 실시간 ETL**의 표준 경로다.
- 핵심은 **(1) 변환 품질(Lambda), (2) 포맷 전환(Parquet), (3) 파티션 설계, (4) 에러 백업/리플레이, (5) 보안·관측성**이다.
- 이 가이드를 템플릿으로 삼아 **작게 시작→지속적 튜닝(버퍼/압축/파티션/알람/IaC)**을 반복하면, 로그·이벤트·IoT까지 공통 파이프라인으로 안정화할 수 있다.
