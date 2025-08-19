---
layout: post
title: AWS - Kinesis Data Firehose
date: 2025-07-28 17:20:23 +0900
category: AWS
---
# 🛠️ AWS Kinesis Data Firehose 기반 ETL 파이프라인 설계

---

## 📌 1. ETL이란?

ETL은 **Extract(추출)**, **Transform(변환)**, **Load(적재)** 의 약자로, 다양한 원천 데이터 소스로부터 데이터를 가져와, 이를 가공한 뒤, 분석 및 저장에 적합한 형태로 저장소에 저장하는 과정을 의미합니다.

---

## 📌 2. AWS 기반의 ETL 아키텍처

AWS에서 ETL을 수행하는 데 사용되는 주요 서비스는 다음과 같습니다:

| 역할 | 서비스 | 설명 |
|------|--------|------|
| 추출 | Kinesis Data Firehose | 실시간 데이터 수집 |
| 변환 | Lambda | 실시간 데이터 처리 |
| 적재 | S3, Redshift, Elasticsearch | 저장소 |

---

## 📌 3. Kinesis Data Firehose란?

**Amazon Kinesis Data Firehose**는 **스트리밍 데이터를 실시간으로 수집하여 자동으로 저장소에 전송하는 fully managed 서비스**입니다.

- 실시간 수집
- 자동 배치 처리
- 데이터 변환 기능 제공 (Lambda 연결)
- 목적지: S3, Redshift, OpenSearch, HTTP endpoint 등

---

## 📌 4. Firehose 기반 ETL 전체 흐름

1. 애플리케이션/서비스가 데이터를 Firehose로 전송
2. Firehose가 데이터를 수신
3. Firehose가 **Lambda 함수를 통해 데이터를 실시간 변환**
4. 변환된 데이터를 S3/Redshift 등에 저장

---

## 📌 5. 실습: S3로의 Firehose ETL 파이프라인 구성

### 5.1 S3 버킷 생성

```bash
aws s3 mb s3://firehose-etl-demo-bucket
```

버킷 정책 예시:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowFirehoseDelivery",
      "Effect": "Allow",
      "Principal": {
        "Service": "firehose.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::firehose-etl-demo-bucket/*"
    }
  ]
}
```

---

### 5.2 IAM 역할 생성 (Firehose용)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "lambda:InvokeFunction",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

역할에 `FirehoseDeliveryRole` 이름을 부여하고, 해당 정책 연결.

---

### 5.3 Lambda 함수 작성 (데이터 변환)

**목표**: JSON 데이터를 필터링하거나 구조 변경

```python
import base64
import json

def lambda_handler(event, context):
    output = []

    for record in event['records']:
        payload = base64.b64decode(record['data']).decode('utf-8')

        # 예: 특정 필드만 추출
        try:
            data = json.loads(payload)
            transformed_data = {
                "username": data.get("user"),
                "timestamp": data.get("time"),
                "action": data.get("action")
            }

            output_record = {
                'recordId': record['recordId'],
                'result': 'Ok',
                'data': base64.b64encode(json.dumps(transformed_data).encode('utf-8')).decode('utf-8')
            }
        except:
            output_record = {
                'recordId': record['recordId'],
                'result': 'ProcessingFailed',
                'data': record['data']
            }

        output.append(output_record)

    return {'records': output}
```

---

### 5.4 Firehose 전송 스트림 생성

```bash
aws firehose create-delivery-stream \
  --delivery-stream-name firehose-etl-stream \
  --s3-destination-configuration file://s3-config.json
```

`s3-config.json` 예시:

```json
{
  "RoleARN": "arn:aws:iam::123456789012:role/FirehoseDeliveryRole",
  "BucketARN": "arn:aws:s3:::firehose-etl-demo-bucket",
  "Prefix": "etl-output/",
  "BufferingHints": {
    "SizeInMBs": 5,
    "IntervalInSeconds": 300
  },
  "CompressionFormat": "UNCOMPRESSED",
  "ProcessingConfiguration": {
    "Enabled": true,
    "Processors": [
      {
        "Type": "Lambda",
        "Parameters": [
          {
            "ParameterName": "LambdaArn",
            "ParameterValue": "arn:aws:lambda:ap-northeast-2:123456789012:function:TransformETLFunction"
          }
        ]
      }
    ]
  }
}
```

---

### 5.5 Firehose에 테스트 데이터 전송

```python
import boto3
import json
import base64

client = boto3.client('firehose', region_name='ap-northeast-2')

data = {
  "user": "kimdh",
  "time": "2025-08-07T10:15:00Z",
  "action": "login"
}

encoded_data = base64.b64encode(json.dumps(data).encode('utf-8'))

client.put_record(
    DeliveryStreamName='firehose-etl-stream',
    Record={'Data': encoded_data}
)
```

---

### 5.6 S3에서 결과 확인

- 버킷 접속 → `etl-output/` 경로 확인
- 변환된 JSON 데이터 파일 존재 확인

---

## 📌 6. S3 외의 목적지 설정

### 🔹 Redshift 적재

- `COPY` 명령을 자동 수행
- S3를 중간 staging 영역으로 사용

필요 조건:
- Redshift 클러스터
- 사용자 테이블 생성
- S3 접근 가능한 IAM 역할

### 🔹 OpenSearch 적재

- 실시간 로그 분석
- 인덱스 이름 설정 가능
- 필드 기반 분석

---

## 📌 7. 성능 및 비용 고려 사항

| 항목 | 설명 |
|------|------|
| 버퍼링 | 데이터 크기 또는 시간 기준으로 전송 |
| 변환 함수 | Lambda 호출 시간 및 실패율 고려 |
| 압축 | GZIP 사용 시 스토리지 절약 |
| 샘플링 | 로그 데이터를 100% 저장하지 않고 일부만 |

---

## 📌 8. 실전 활용 사례

- **IoT 데이터 수집**: 디바이스에서 Firehose로 직접 전송
- **웹 로그 분석**: CloudFront 로그 → Firehose → S3 → Athena
- **애플리케이션 이벤트 트래킹**: 프론트에서 직접 전달

---

## 📌 9. 보안 설정

- **KMS 암호화**: S3 저장 시 암호화
- **VPC 인터페이스 엔드포인트**: 내부 네트워크 전용 Firehose
- **IAM 제한**: Firehose → S3만 허용

---

## 📌 10. 정리

| 구성 요소 | 역할 |
|-----------|------|
| Producer | 데이터를 Firehose로 전송 |
| Firehose | 데이터 수신, 변환, 저장 |
| Lambda | 사용자 정의 변환 처리 |
| S3/Redshift | 최종 데이터 저장 |

---

## ✅ 마무리

Amazon Kinesis Data Firehose는 복잡한 스트리밍 파이프라인을 코드 없이 구성할 수 있는 강력한 도구입니다. 특히 **Lambda를 통한 실시간 변환**, **자동 버퍼링**, **복수 목적지 전송** 기능은 실시간 ETL을 단순화하는 핵심입니다.

향후에는 Glue, Athena, QuickSight 등과 연계하여 **서버리스 데이터 분석 플랫폼**으로 확장해보세요!