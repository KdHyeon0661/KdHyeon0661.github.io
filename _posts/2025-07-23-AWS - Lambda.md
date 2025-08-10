---
layout: post
title: AWS - Lambda
date: 2025-07-23 19:20:23 +0900
category: AWS
---
# AWS Lambda: 서버리스 함수 실행 완전 가이드

AWS Lambda는 **서버를 직접 관리하지 않고도 코드 실행이 가능한 서버리스 컴퓨팅 서비스**입니다. 인프라 관리를 AWS가 대신 해주며, 사용자는 오직 코드 작성과 이벤트 트리거 구성에만 집중하면 됩니다. 이 글에서는 Lambda의 핵심 개념부터 구조, 동작 방식, 실전 예제까지 자세히 설명합니다.

---

## 🧠 Lambda의 핵심 개념

### ✅ 서버리스란?

- **서버가 없다는 의미는 아님**
  - "서버리스(Serverless)"는 사용자가 직접 서버를 **프로비저닝, 유지관리, 확장**하지 않아도 된다는 뜻입니다.
- 서버 관리, OS 패치, 스케일링, 로드 밸런싱은 **AWS가 자동 처리**
- 사용자는 함수 단위로 코드 작성 → 이벤트 발생 시 자동 실행

### ✅ AWS Lambda란?

- AWS가 제공하는 FaaS(Function as a Service)
- **함수 단위로 실행** (Function-based computing)
- **이벤트 기반 실행**: S3 업로드, DynamoDB 변경, API Gateway 요청 등
- **자동 스케일링** 지원 (사용량에 따라 자동으로 인스턴스 증가)
- **과금**: 코드가 실행된 시간만큼만 과금 (100ms 단위)

---

## 🧱 Lambda 구조

```text
+-------------+        +-------------+       +-------------+
| Event Source| -----> | Lambda 함수 | --->  | 결과 반환/처리 |
+-------------+        +-------------+       +-------------+
     ↑ 이벤트 발생                        코드 실행
```

### 주요 구성 요소

- **이벤트 소스(Event Source)**: Lambda 함수를 트리거하는 서비스
  - 예: S3, API Gateway, EventBridge, DynamoDB, SNS, SQS 등
- **핸들러 함수(Handler)**: 실제로 실행되는 코드 진입점
- **Lambda 함수 설정**
  - 실행 시간 제한, 메모리 크기, IAM 역할, 환경변수 등
- **권한 (IAM Role)**: Lambda가 리소스에 접근할 수 있도록 하는 역할

---

## 🛠️ Lambda 생성과 구성

### 1. Lambda 함수 생성

- AWS 콘솔 > Lambda > 함수 생성
- `런타임 선택`: Python, Node.js, Java, Go, .NET 등
- `함수 권한`: 기존 역할 선택 or 새 역할 생성 (ex. S3 읽기 권한)

### 2. 핸들러 예시 (Python)

```python
def lambda_handler(event, context):
    print("이벤트 수신:", event)
    return {
        'statusCode': 200,
        'body': 'Hello from Lambda!'
    }
```

### 3. 환경 변수 사용

- 함수 내에서 사용할 수 있는 값 (ex. DB 접속 문자열)
- `os.environ`으로 접근 가능

```python
import os
def lambda_handler(event, context):
    api_key = os.environ['API_KEY']
```

---

## ⚙️ 트리거 예제

### S3 이벤트로 Lambda 실행

1. S3 버킷 > 속성 > 이벤트 > 새 이벤트 알림
2. Lambda 함수 선택
3. 특정 경로에 파일 업로드 시 자동 실행

### API Gateway로 HTTP 요청 처리

1. API Gateway 생성
2. Lambda 함수 연결
3. REST API or HTTP API 구성 가능

```http
POST /hello -> Lambda 함수 실행
```

---

## 🧪 실전 예제: 이미지 리사이징 함수

1. 사용자가 이미지를 S3에 업로드
2. Lambda 함수가 실행 → 썸네일 생성
3. 결과를 다른 S3 버킷에 저장

```python
import boto3
from PIL import Image
import io

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    image_obj = s3.get_object(Bucket=bucket, Key=key)
    image = Image.open(image_obj['Body'])
    image.thumbnail((128, 128))
    
    buffer = io.BytesIO()
    image.save(buffer, 'JPEG')
    buffer.seek(0)
    
    s3.put_object(Bucket='thumbnail-bucket', Key=key, Body=buffer)
    return {'statusCode': 200, 'body': 'Thumbnail created'}
```

---

## 📊 Lambda 과금 구조

| 항목         | 내용                                  |
|--------------|---------------------------------------|
| 호출 수      | 월 100만 건 무료, 이후 $0.20/백만 건 |
| 실행 시간     | 100ms 단위, 메모리 크기에 따라 요금 차등 |
| 데이터 전송   | AWS 외부로 나가는 트래픽은 과금됨     |

---

## 📌 Lambda의 장점

- 서버 관리 불필요
- 비용 효율적 (짧은 실행, 비정기 호출)
- Auto Scaling 기본 내장
- 다양한 AWS 서비스와 연동성 탁월

---

## ⚠️ Lambda의 한계와 주의점

- **실행 시간 제한**: 기본 3초~최대 15분
- **Cold Start**: 첫 실행 시 지연 발생 가능
- **디버깅 어려움**: 로컬 디버깅이 제한적
- **복잡한 로직에는 부적합**

---

## 🧩 Lambda와 다른 서비스 통합 예시

| 서비스       | 통합 방식 예시                                |
|--------------|-----------------------------------------------|
| S3           | 객체 업로드 트리거                            |
| DynamoDB     | 테이블 변경 감지 트리거                       |
| API Gateway  | HTTP 요청 핸들링                              |
| EventBridge  | 일정 기반 함수 실행                           |
| Step Functions | 여러 Lambda 함수 순차 실행 (워크플로우)   |
| SNS/SQS      | 메시지 기반 트리거                            |
| CloudWatch   | 로그/지표 기반 알람 실행                      |

---

## 🔐 Lambda와 보안

- **IAM 역할 부여**: 최소 권한 원칙 적용 (Least Privilege)
- **환경 변수 암호화**: KMS를 통해 민감 정보 보호
- **VPC 연동**: Lambda를 VPC에 연결하여 DB 접근 등 가능

---

## 🧰 개발 및 배포 도구

| 도구              | 설명                                     |
|-------------------|------------------------------------------|
| AWS SAM           | Lambda 배포 자동화 도구 (Serverless Application Model) |
| Serverless Framework | 멀티 클라우드 지원 오픈소스 프레임워크 |
| Terraform         | 인프라 코드(IaC)로 Lambda 관리 가능       |
| CloudFormation    | AWS 자원 템플릿 관리                     |

---

## ✅ 결론

Lambda는 "필요할 때만 실행되는" 함수 단위의 서버리스 컴퓨팅 플랫폼으로, **비용 절감**, **운영 단순화**, **빠른 개발 및 배포**를 가능하게 해줍니다. 다양한 AWS 서비스와 쉽게 통합되어 이벤트 기반 아키텍처를 구축할 수 있어 현대 클라우드 애플리케이션의 핵심 구성 요소로 자리잡고 있습니다.
