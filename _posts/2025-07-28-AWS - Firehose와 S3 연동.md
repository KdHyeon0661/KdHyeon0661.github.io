---
layout: post
title: AWS - Firehose와 S3 연동
date: 2025-07-28 22:20:23 +0900
category: AWS
---
# 🔥 Firehose와 S3 연동: 실시간 데이터 수집과 저장

## ✅ 1. 개요

Amazon Kinesis Data Firehose는 **실시간 스트리밍 데이터를 자동으로 수집하고 변환하여 대상에 전달**하는 Fully Managed 서비스입니다.  
주요 대상은 다음과 같습니다:

- Amazon S3 (데이터 저장)
- Amazon Redshift (데이터 웨어하우스)
- Amazon OpenSearch (검색/로그 분석)
- Splunk 등 외부 서비스

이번 정리에서는 **Kinesis Data Firehose → Amazon S3 연동**을 중심으로 구성합니다. 이는 로그 저장, 실시간 분석 데이터 적재 등 많은 실무에서 사용됩니다.

---

## ✅ 2. 구성 아키텍처

```
[Producer App or Kinesis Data Stream]
            ↓
     [Kinesis Data Firehose]
            ↓
 [Optional: Lambda 변환 (ETL)]
            ↓
         [Amazon S3]
```

---

## ✅ 3. Firehose 전달 스트림 생성 및 S3 연동

### 🔹 3.1 Firehose 전달 스트림 생성

1. **AWS Management Console** → "Kinesis" → "Delivery streams" → "Create delivery stream"
2. **Source 선택**: 다음 중 하나 선택
   - Direct PUT (프로듀서 앱에서 직접 Firehose에 PUT)
   - Kinesis Data Stream (KDS와 연동 가능)
3. **Destination 선택**: Amazon S3

### 🔹 3.2 S3 버킷 설정

- Firehose는 데이터를 전달할 **S3 버킷 경로**를 지정해야 합니다.
- 예: `s3://my-data-bucket/firehose-output/`

### 🔹 3.3 S3 버킷 정책 예시

Firehose가 데이터를 업로드할 수 있도록 다음과 같은 IAM 정책 필요:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "FirehoseS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::my-data-bucket/*"
    }
  ]
}
```

### 🔹 3.4 버퍼링 설정

Firehose는 데이터를 일정 시간 또는 크기 기준으로 버퍼링 후 S3에 전달합니다.

- **Buffer Size**: 1MiB ~ 128MiB (기본: 5MiB)
- **Buffer Interval**: 60초 ~ 900초 (기본: 300초)

### 🔹 3.5 압축 및 포맷

S3에 데이터를 저장할 때 다음 중 하나를 선택할 수 있습니다.

- **압축 형식**: GZIP, ZIP, Snappy
- **포맷 변환**: Parquet, ORC (Glue Schema Registry를 이용)

---

## ✅ 4. (선택) Lambda를 이용한 데이터 변환

Firehose는 데이터를 S3에 저장하기 전에 **Lambda 함수를 통해 변환 처리**가 가능합니다.

### 🔸 사용 예시

- JSON 로그 → 특정 필드 제거/추가
- CSV 변환
- PII(개인정보) 마스킹

### 🔸 설정 방법

1. “Transform source records with Lambda” 옵션 선택
2. 사용할 Lambda 함수 지정
3. Lambda는 `batch` 형태로 데이터를 입력받아 변환 후 반환해야 함

예시:

```python
import base64
import json

def lambda_handler(event, context):
    output = []

    for record in event['records']:
        payload = base64.b64decode(record['data']).decode('utf-8')
        transformed = json.loads(payload)
        # 예: 사용자 이메일 제거
        transformed.pop('email', None)
        output_record = {
            'recordId': record['recordId'],
            'result': 'Ok',
            'data': base64.b64encode(json.dumps(transformed).encode('utf-8')).decode('utf-8')
        }
        output.append(output_record)

    return {'records': output}
```

---

## ✅ 5. 실시간 테스트 방법

Firehose가 직접 데이터를 수신하는 경우 CLI로 테스트할 수 있습니다.

```bash
aws firehose put-record \
  --delivery-stream-name my-firehose-stream \
  --record='{"Data":"{\"user\":\"doe\",\"action\":\"login\"}\n"}'
```

- base64로 인코딩된 문자열이 저장됨
- 일정 시간 후, S3 버킷 내 경로에 파일이 생성됨

---

## ✅ 6. 모니터링 및 오류 처리

### 🔸 CloudWatch Logs 연동

- Firehose는 실패한 레코드 또는 Lambda 실패 등을 CloudWatch Logs로 전송
- 모니터링 지표:
  - `DeliveryToS3.Records`
  - `DeliveryToS3.DataFreshness`
  - `ThrottledRecords`

### 🔸 오류 처리

- 전송 실패 시 **Retry** 후에도 실패하면 **S3 Backup Bucket** 또는 **CloudWatch Logs**로 전송 가능

---

## ✅ 7. 실무 예시

### 📌 예제 1: 웹 서버 액세스 로그 저장

1. Apache/Nginx에서 로그를 실시간 수집
2. Agent (ex. Fluent Bit, Kinesis Agent)가 Firehose로 전송
3. Firehose → S3 버킷에 압축 저장
4. Athena 또는 Glue Crawler로 분석

### 📌 예제 2: IoT 센서 데이터 저장

1. IoT 디바이스가 MQTT → IoT Core → Firehose
2. Firehose는 JSON 센서 데이터를 Parquet로 변환
3. S3에 저장 후 Redshift Spectrum으로 분석

---

## ✅ 8. 비용 고려 사항

- Firehose는 **전송된 데이터 양(GB)** 기준 요금
- 압축 및 Parquet 변환으로 S3 저장 비용 최적화 가능
- Lambda 변환 시 **추가 비용 발생**

---

## ✅ 9. 보안 설정

- S3 버킷: 버전 관리, 암호화(SSE-S3, SSE-KMS), 퍼블릭 차단 설정
- Firehose IAM 역할: 최소 권한의 원칙 적용
- 데이터: 전송 중 HTTPS, 저장 시 암호화 적용

---

## ✅ 10. 정리

| 구성 요소         | 역할                           |
|------------------|--------------------------------|
| Firehose         | 데이터 수집/버퍼링/전송        |
| Lambda (선택)     | 데이터 실시간 변환             |
| S3               | 영구 저장소 (압축/변환 가능)   |
| CloudWatch       | 모니터링 및 오류 추적         |

---

## 🔚 결론

Amazon Kinesis Firehose는 실시간 데이터 수집과 저장을 위한 강력한 도구입니다. 특히 **S3와 연계하면 구조화/비구조화 데이터를 저렴하고 안전하게 저장**할 수 있으며, 이후 **Athena, Glue, Redshift 등 분석 도구와 연계**해 다양한 실시간/배치 분석 파이프라인을 구축할 수 있습니다.