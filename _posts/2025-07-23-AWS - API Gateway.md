---
layout: post
title: AWS - API Gateway
date: 2025-07-23 20:20:23 +0900
category: AWS
---
# 🧩 API Gateway: API 게이트웨이 구성

AWS API Gateway는 클라이언트와 백엔드 서비스 간의 **인터페이스** 역할을 하는 **완전관리형 서비스**입니다. API를 손쉽게 생성, 배포, 유지보수할 수 있도록 하며, Lambda, EC2, HTTP 엔드포인트, AWS 서비스 등과 통신할 수 있게 해줍니다.

---

## 📌 1. API Gateway란?

### ✅ API Gateway의 정의

> API Gateway는 다양한 API 요청을 수신하여 적절한 AWS 리소스 또는 외부 서비스로 전달하고, 응답을 다시 클라이언트에게 반환하는 **프록시(Proxy) 서버** 역할을 합니다.

클라이언트 ↔️ API Gateway ↔️ Lambda, EC2, S3, RDS, 외부 서비스

### ✅ API Gateway의 주요 기능

- REST API, WebSocket API, HTTP API 지원
- 인증/인가 (IAM, Cognito, Lambda Authorizer)
- 요청/응답 변환 (Mapping Template)
- API 제한 (Rate limiting, Throttling)
- 캐싱
- 모니터링 (CloudWatch 연동)
- Swagger / OpenAPI 지원

---

## 📌 2. API Gateway 작동 방식

1. 클라이언트가 API Gateway로 HTTP 요청을 보냄
2. Gateway는 정의된 **리소스 경로**와 **HTTP 메서드**에 따라 처리
3. 요청을 연결된 백엔드(예: Lambda, EC2)에 전달
4. 백엔드 응답을 다시 가공 후 클라이언트에게 반환

---

## 📌 3. API Gateway 유형

| 유형 | 설명 | 특징 |
|------|------|------|
| REST API | 기존 RESTful API용 | 기능 다양, 무거움 |
| HTTP API | 가볍고 빠른 REST API용 | 성능 우수, 제한된 기능 |
| WebSocket API | 실시간 양방향 통신 | IoT, 채팅 등에 적합 |

**Tip:**  
단순한 REST API라면 `HTTP API` 사용 권장 (저렴하고 빠름)  
복잡한 인증, 통합, 변환이 필요하면 `REST API` 사용

---

## 📌 4. 구성 요소 설명

| 구성 요소 | 설명 |
|-----------|------|
| 리소스(Resource) | `/users`, `/products` 같은 경로 |
| 메서드(Method) | GET, POST, PUT, DELETE 등 |
| 통합(Integration) | 요청을 전달할 대상: Lambda, HTTP, Mock 등 |
| 스테이지(Stage) | 배포 환경: dev, test, prod 등 |
| 매핑 템플릿 | 요청/응답 형식 변환 |
| 인증(Auth) | Cognito, Lambda Authorizer, IAM 역할 기반 등 |
| 캐싱 | API 응답을 일정 시간 저장 |

---

## 📌 5. 실습: Lambda와 연결된 REST API 만들기

### ① Lambda 함수 생성

```python
# hello-world.py
def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Hello from Lambda!'
    }
```

IAM Role에는 `AWSLambdaBasicExecutionRole` 포함 필요

### ② API Gateway 생성 (REST API)

1. **API Gateway → 새 API 생성 → REST API (클래식) 선택**
2. 이름: `MyAPI`
3. 리소스 생성: `/hello`
4. 메서드 추가: `GET`
5. 통합 타입: **Lambda 함수**
6. 함수 이름: `hello-world`
7. `Deploy API` → 새 스테이지(`dev`) 생성

### ③ 호출

```bash
curl https://<api-id>.execute-api.<region>.amazonaws.com/dev/hello
```

응답:
```json
Hello from Lambda!
```

---

## 📌 6. 통합 유형(Integration Type)

| 유형 | 설명 |
|------|------|
| Lambda | AWS Lambda 함수 호출 |
| HTTP | 외부 HTTP 엔드포인트 |
| AWS 서비스 | SNS, SQS 등 AWS 서비스 직접 호출 |
| Mock | 백엔드 없이 테스트용 응답 반환 |

---

## 📌 7. 인증/인가 방식

| 방식 | 설명 |
|------|------|
| IAM 인증 | IAM 사용자의 서명 요청 |
| Cognito | 사용자 풀 기반 인증/인가 |
| Lambda Authorizer | 사용자 정의 Lambda로 토큰 검사 |

**예: Lambda Authorizer**

```python
def lambda_handler(event, context):
    token = event['authorizationToken']
    if token == "allow":
        return {
            'principalId': 'user',
            'policyDocument': {
                'Version': '2012-10-17',
                'Statement': [{
                    'Action': 'execute-api:Invoke',
                    'Effect': 'Allow',
                    'Resource': event['methodArn']
                }]
            }
        }
```

---

## 📌 8. 요청 및 응답 매핑

요청/응답을 Lambda나 백엔드 서비스에서 요구하는 형식으로 변환할 수 있음

예시:

```json
# Mapping Template (Request)
{
  "username": "$input.params('username')"
}
```

```json
# Mapping Template (Response)
{
  "result": "$input.body"
}
```

---

## 📌 9. CORS 설정

다른 도메인에서 호출하려면 CORS 설정 필요

1. `OPTIONS` 메서드 추가
2. 헤더 수동 지정:
   - `Access-Control-Allow-Origin: *`
   - `Access-Control-Allow-Methods: GET, POST`
   - `Access-Control-Allow-Headers: Content-Type`

또는 콘솔에서 "CORS 활성화" 자동 설정 기능 제공

---

## 📌 10. API 버전 관리 및 스테이지

- **스테이지**로 dev, test, prod 구분 가능
- Stage Variable을 이용해 Lambda 버전 지정 가능
- 예: `dev` 스테이지는 v1 Lambda, `prod`는 v2 Lambda 사용

---

## 📌 11. 로깅 및 모니터링

- CloudWatch Logs와 통합 가능
- 요청/응답 로그, 에러 로그 확인
- Usage Plan + API Key로 사용량 추적 가능

---

## 📌 12. 캐싱

- 응답 캐싱 설정 가능 (TTL 조정)
- 외부 DB 조회 결과 캐싱하여 비용 절감 및 속도 향상

---

## 📌 13. 비용 구조

| 항목 | 설명 |
|------|------|
| API 호출 수 | 월별 요청 수에 따라 과금 |
| 캐시 용량 | 캐시 메모리 GB당 과금 |
| 데이터 전송 | 응답 크기 및 전송량에 따라 과금 |

---

## 📌 14. 베스트 프랙티스

- **HTTP API 사용 우선 고려**
- 인증은 가능한 `Cognito` 활용
- Lambda는 타임아웃 관리 필수
- CloudWatch 로깅 활성화
- API Gateway + Lambda + DynamoDB 조합 활용

---

## ✅ 마무리

API Gateway는 서버 없는 아키텍처(serverless)를 설계할 때 핵심적인 역할을 하는 서비스입니다. Lambda, 인증, 외부 시스템 연동까지 하나의 인터페이스로 관리할 수 있어, 개발과 운영의 생산성을 높여주는 강력한 도구입니다.
