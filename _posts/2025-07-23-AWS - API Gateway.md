---
layout: post
title: AWS - API Gateway
date: 2025-07-23 20:20:23 +0900
category: AWS
---
# API Gateway: API 게이트웨이 구성

## 한 장 요약

- **HTTP API**: 가볍고 저렴/고성능. JWT Authorizer, 통합 단순화. 대부분의 **새로운 REST형 API**에 추천.
- **REST API(클래식)**: 세밀한 **매핑 템플릿, 모델/밸리데이션, 캐시, Usage Plan** 등 **풍부한 기능**. 복잡 변환/레거시 호환 시 적합.
- **WebSocket API**: 양방향 실시간(채팅/IoT/알림).
- **보안**: IAM/Cognito/Lambda Authorizer, WAF, 프라이빗/리소스 정책.
- **백엔드**: Lambda/HTTP(외부)/AWS 서비스/프라이빗(ENI, VPC Link).
- **운영**: 스테이지/배포, Canary, 캐시, 스로틀, Usage Plan+API 키, 로깅/메트릭, 경보, 비용.

---

## API Gateway란? (초안 보강)

**역할**: 클라이언트 요청을 받아 **인증/인가** → **요청 가공** → **백엔드 통합(Integration)** → **응답 가공** → 반환.
**그림**: Client → API Gateway(리소스/메서드/라우트, Authorizer, Mapping, Throttle, Cache) → Lambda/HTTP/AWS 서비스 → 응답.

---

## 유형 비교

| 유형 | 권장 시나리오 | 장점 | 제약 |
|---|---|---|---|
| **HTTP API** | 신규 REST형 API, JWT 기반 인증, 단순 프록시 | **저비용, 낮은 지연, 단순 구성** | REST API 대비 세밀한 매핑/캐시/Usage Plan 한정 |
| **REST API(클래식)** | 정교한 매핑 템플릿, 모델 검증, 스테이지 변수, 캐시/Usage Plan | **기능 풍부** | 비용/지연 ↑, 설정 복잡 |
| **WebSocket API** | 실시간 양방향(채팅, 알림, 트레이딩) | 연결 지속, 서버리스 통합 | 설계 난이도, 상태/스케일링 고려 |

**Tip**: 가능한 **HTTP API 우선**. 복잡 변환/캐시/UsagePlan 필요 시 REST API.

---

## 구성 요소 (초안 보강)

| 구성 요소 | HTTP API | REST API(클래식) |
|---|---|---|
| 라우트/리소스 | 라우트(METHOD + Path) | 리소스(`/users`) + 메서드(GET/POST/…) |
| 통합(Integration) | Lambda/HTTP/서비스 | Lambda/HTTP/서비스/Mock |
| 인증/인가 | JWT Authorizer, Lambda Authorizer, IAM | Cognito, Lambda Authorizer, IAM |
| 트랜스폼 | 단순/헤더/쿼리 전달 위주 | **매핑 템플릿(VTL)**, 모델/검증 |
| 캐시 | 제한적 | **스테이지 캐시** |
| 스테이지/배포 | `$default` 자동 배포 가능 | Stage별 배포, 변수 지원 |
| 사용량/키 | 제한적 | **Usage Plan + API Key** |
| 관측성 | CloudWatch/Access Logs | CloudWatch/Access Logs, X-Ray |

---

## 작동 흐름 (요청 → 검증/인가 → 통합 → 응답)

1) **라우팅**: 경로/메서드 매칭
2) **보안**: Authorizer/JWT/IAM/WAF/리소스 정책
3) **검증/변환**: (REST) 매핑 템플릿/모델 검증
4) **백엔드 통합 호출**: Lambda/HTTP/서비스/프라이빗
5) **응답 변환 & 헤더**
6) **캐시/스로틀/로깅**
7) **클라이언트 반환**

---

## 실습 A: Lambda + HTTP API(권장 경로)

### 람다 함수 (Python)

```python
# app.py

import json

def lambda_handler(event, context):
    # HTTP API의 event 예: {"rawPath":"/hello","queryStringParameters":{"name":"Kim"},...}
    name = None
    qs = event.get("queryStringParameters") or {}
    name = qs.get("name", "world")
    return {
        "statusCode": 200,
        "headers": {"Content-Type":"application/json"},
        "body": json.dumps({"message": f"Hello, {name}!"})
    }
```

필요 권한: `AWSLambdaBasicExecutionRole`.

### HTTP API 생성(간단 콘솔 절차)

- API Gateway → **HTTP API** → 라우트 `GET /hello` → 통합: Lambda(`app.lambda_handler`) → 배포 `$default`
- 실행:
```bash
curl "https://<api-id>.execute-api.<region>.amazonaws.com/hello?name=Kim"
```

---

## 실습 B: REST API(클래식) + 매핑 템플릿 + 캐시

### 람다 (Node)

```js
// index.js
exports.handler = async (event) => {
  // event.body는 매핑 템플릿 결과(JSON 문자열)
  const req = JSON.parse(event.body || "{}");
  const x = (req.a || 0) + (req.b || 0);
  return {
    statusCode: 200,
    headers: {"Content-Type":"application/json"},
    body: JSON.stringify({sum: x})
  };
};
```

### 요청 매핑 템플릿(VTL)

```vtl
#set($inputRoot = $input.params())

{
  "a": $util.parseJson($input.body).a,
  "b": $util.parseJson($input.body).b,
  "callerIp": "$context.identity.sourceIp",
  "path": "$context.resourcePath",
  "qs": {
    "debug": "$util.escapeJavaScript($input.params('debug'))"
  }
}
```

### 응답 매핑 템플릿(VTL)

```vtl
#set($resp = $util.parseJson($input.body))

{
  "result": $resp.sum,
  "timestamp": "$context.requestTime"
}
```

### 캐시(REST API 스테이지)

- 스테이지 → **Cache Enabled** → TTL 설정
- 캐시 키 파라미터: `Integration Request`에서 **쿼리/헤더**를 캐시 키에 포함.

---

## 인증/인가

### IAM 인증(서명 요청)

- 서버-서버/사내 콜에서 주로 사용.
- 클라이언트가 **SigV4 서명**으로 호출.

### Cognito(JWT)

- User Pool로 로그인 → **ID 토큰/Access 토큰(JWT)** 발급 → HTTP API에서 **JWT Authorizer**로 검증.

#### HTTP API의 JWT Authorizer 예(콘솔 요약)

- Authorizers → **JWT** → Issuer/오디언스 설정 → 라우트에 연결

### Lambda Authorizer(토큰 커스텀 로직)

```python
# 람다 권한자(HTTP API or REST API)

def lambda_handler(event, context):
    token = event.get("headers",{}).get("authorization","")
    # 검증 로직(예: HMAC, 외부 권한 서버 질의)
    effect = "Allow" if token == "allow" else "Deny"
    methodArn = event["routeArn"] if "routeArn" in event else event["methodArn"]
    return {
      "principalId":"user-123",
      "policyDocument":{
        "Version":"2012-10-17",
        "Statement":[{"Action":"execute-api:Invoke","Effect":effect,"Resource":methodArn}]
      },
      "context":{"role":"basic"}
    }
```

**Tip**: 토큰 파싱/검증의 **지연**을 최소화(캐시/짧은 네트워크 경로).

---

## 요청/응답 매핑 & 유효성(REST API 중심)

- **매핑 템플릿(VTL)**: 바디/헤더/쿼리/경로 → 백엔드 계약형으로 변환.
- **모델/Validator**: OpenAPI/스키마 기반 검증(에러를 게이트웨이에서 반환해 백엔드 부담 완화).

**요청 모델 예 (JSON Schema)**
```json
{
  "type":"object",
  "required":["a","b"],
  "properties":{
    "a": {"type":"integer"},
    "b": {"type":"integer"}
  }
}
```

---

## CORS

- **HTTP API**: CORS 탭에서 Origin/Headers/Methods 설정.
- **REST API**: `OPTIONS` 메서드 + 응답 헤더 추가 또는 콘솔의 CORS 활성화.

필수 헤더 예:
```
Access-Control-Allow-Origin: https://www.example.com
Access-Control-Allow-Methods: GET,POST,OPTIONS
Access-Control-Allow-Headers: Content-Type,Authorization
```

---

## 스테이지/버전/도메인/배포

- **Stage**: `dev`, `test`, `prod` 분리.
- **Stage Variables(REST)**: 백엔드 엔드포인트/Lambda 버전 바인딩.
- **Custom Domain**: ACM 인증서 + API 매핑(HTTP/REST) → 사용자 도메인 공개.
- **Canary Deployment(REST)**: 트래픽 일부를 새 배포에 라우팅.

---

## 관측성 & 제어

### 로깅/메트릭

- **Access Logs**: 요청/응답 요약(HTTP API/REST 모두).
- **Execution Logs(REST)**: 통합/매핑 디버깅.
- **CloudWatch Metrics**: `4XXError`, `5XXError`, `Latency`, `Count`, `IntegrationLatency`.

### 스로틀/쿼터

- **Account/Region 기본 한도** + **Usage Plan(REST)** + **Route별 제한(HTTP/REST)**
- **WAF** 연계로 L7 보호(봇/SQLi/XSS 룰셋).

---

## 캐싱(REST API)

- **스테이지 캐시**: 응답 TTL, 캐시 키 셀렉터(쿼리/헤더/경로).
- DB 조회/가격표/정적 변환 결과 캐시 → 비용/지연 절감.

---

## 프라이빗/하이브리드 통합

- **Private Integration**: VPC 내부 **NLB** 뒤의 서비스로 연결(**VPC Link**).
- **Private API(REST)**: 엔드포인트 타입 Private + 리소스 정책(한정된 VPC/엔드포인트에서만 접근).
- **Lambda VPC**: 람다를 VPC에 붙여 내부 자원 접근.

---

## 파일 업로드(대용량) 패턴

- **API Gateway 직접 업로드**는 비효율.
- **S3 Pre-signed URL** 발급 API → 클라이언트가 **S3로 직접 업/다운로드**.

예: Pre-signed URL 발급 람다
```python
import boto3, os
def lambda_handler(event, context):
    key = event["queryStringParameters"]["key"]
    s3 = boto3.client("s3")
    url = s3.generate_presigned_url(
        "put_object",
        Params={"Bucket": os.environ["BUCKET"], "Key": key, "ContentType":"application/octet-stream"},
        ExpiresIn=300
    )
    return {"statusCode":200,"headers":{"Content-Type":"application/json"},"body":json.dumps({"url":url})}
```

---

## 비용 모델 직관

$$
\text{월 비용} \approx \sum_a (R_a \cdot p_a) + C_{\text{cache}} + B_{\text{egress}}
$$
- \(R_a\): API 유형/스테이지별 **요청 수**, \(p_a\): **요청 단가**
- \(C_{\text{cache}}\): 캐시 메모리(REST)
- \(B_{\text{egress}}\): 외부 전송
- **HTTP API**가 일반적으로 **REST API보다 저렴/지연 낮음**.
- **Lambda/백엔드 비용**도 함께 최적화(콜 수/지연/메모리).

---

## OpenAPI로 API 정의(HTTP API 예)

```yaml
openapi: 3.0.1
info:
  title: My HTTP API
  version: '1.0'
paths:
  /hello:
    get:
      responses:
        '200':
          description: OK
      x-amazon-apigateway-integration:
        payloadFormatVersion: '2.0'
        type: aws_proxy
        httpMethod: POST
        uri: arn:aws:apigateway:ap-northeast-2:lambda:path/2015-03-31/functions/arn:aws:lambda:ap-northeast-2:123456789012:function:hello/invocations
components: {}
```

배포:
```bash
aws apigatewayv2 import-api --body file://openapi.yaml
```

---

## SAM으로 배포(HTTP API + Lambda + CORS)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  HelloFn:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: app.lambda_handler
      Runtime: python3.12
      Policies: AWSLambdaBasicExecutionRole
      Events:
        HttpGet:
          Type: HttpApi
          Properties:
            Path: /hello
            Method: GET
  HttpApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      CorsConfiguration:
        AllowOrigins: ['https://www.example.com']
        AllowMethods: ['GET','OPTIONS']
        AllowHeaders: ['Authorization','Content-Type']
```

배포:
```bash
sam build && sam deploy --guided
```

---

## Terraform(요약: HTTP API + 람다 통합)

```hcl
provider "aws" { region = "ap-northeast-2" }

resource "aws_lambda_function" "hello" {
  function_name = "hello"
  handler       = "app.lambda_handler"
  runtime       = "python3.12"
  filename      = "build/hello.zip"
  role          = aws_iam_role.lambda_exec.arn
}

resource "aws_apigatewayv2_api" "http_api" {
  name          = "my-http-api"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "lambda" {
  api_id                 = aws_apigatewayv2_api.http_api.id
  integration_type       = "AWS_PROXY"
  integration_uri        = aws_lambda_function.hello.arn
  payload_format_version = "2.0"
}

resource "aws_apigatewayv2_route" "hello" {
  api_id    = aws_apigatewayv2_api.http_api.id
  route_key = "GET /hello"
  target    = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}

resource "aws_lambda_permission" "apigw" {
  statement_id  = "AllowAPIGWInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.hello.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.http_api.execution_arn}/*/*"
}
```

---

## WebSocket API(간단 개념 & 예시)

- **경로**: `$connect`, `$disconnect`, `$default`
- **상태**: 연결ID(ConnectionId)를 **DynamoDB/ElastiCache** 등에 저장해 타깃 브로드캐스트.
- **응답**: `@connections` API로 특정 연결에 푸시.

예: Python으로 특정 연결에 메시지
```python
import boto3, os
def push_message(api_id, region, connection_id, body):
    gw = boto3.client("apigatewaymanagementapi",
                      endpoint_url=f"https://{api_id}.execute-api.{region}.amazonaws.com/prod")
    gw.post_to_connection(ConnectionId=connection_id, Data=body.encode())
```

---

## 베스트 프랙티스

- **HTTP API 우선**, 레거시/정밀 변환/캐시 필요 시 REST API.
- 인증은 **Cognito JWT** 또는 **Lambda Authorizer**(필요 시) 사용.
- **S3 업로드는 Pre-signed URL**로, API는 **URL 발급**만.
- **Idempotency-Key**(헤더)로 **POST 멱등성** 보장.
- **Cold Start** 줄이기: 람다 메모리/언어/프리프로비전 컨커런시.
- **WAF**/리소스 정책으로 공격면 최소화.
- **Observability**: Access Logs, 메트릭 경보, 트레이싱(X-Ray).
- **Canary**로 점진 배포, 에러 상승 시 즉시 롤백.

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| 403 Forbidden | 인증 실패/IAM 권한 부족 | Authorizer/JWT 범위, IAM 정책 점검 |
| 415/502(REST) | 매핑 템플릿 오류 | VTL 로그/Execution Logs 확인 |
| CORS 실패 | 헤더/메서드 미노출 | CORS 설정/프리플라이트 허용 재확인 |
| 타임아웃 | 백엔드 지연 | 통합/람다 타임아웃 상향, 백엔드 성능 개선 |
| 비용 급증 | 과도 호출/대형 응답 | 캐시/압축/쿼터/요청 축소 |
| Private 통합 실패 | VPC Link/보안그룹/라우팅 미스 | NLB 타깃, SG, 라우트 테이블 확인 |

---

## CI/CD(예: GitHub Actions)

```yaml
name: Deploy API
on:
  push: { branches: [ main ] }
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/oidc-apigw-deploy
          aws-region: ap-northeast-2
      - name: Build Lambda
        run: |
          zip -r build/hello.zip src/
      - name: Terraform Apply
        run: |
          terraform init
          terraform apply -auto-approve
```

---

## 보안/규정/거버넌스

- **Least Privilege**: 배포 역할/람다 역할 최소화.
- **키/비밀**: Secrets Manager/SSM Parameter Store.
- **데이터 보호**: TLS(엔드투엔드), WAF, 스키마 검증.
- **감사**: CloudTrail(구성 변경/호출 추적), Access Logs 보관(S3 Object Lock 옵션).

---

## 마무리 요약(초안 정리 + 보강)

- **HTTP API**로 **간단/저렴/고성능** 경로를 우선 채택,
- 필요 시 **REST API**의 **세밀 매핑/캐시/Usage Plan**을 사용,
- 인증/인가(**Cognito/JWT/Lambda Authorizer/IAM**)를 요구에 맞춰 결합,
- **프라이빗 통합(VPC Link/Private API)**와 **WAF/스로틀**로 보안/안정성을 강화,
- **관측성/경보/CI/CD/IaC**로 **운영 자동화**를 정착하라.

이 문서의 예제들을 **당신의 리전/계정/도메인/정책/역할**로 치환하면 즉시 프로덕션에 적용 가능한 **API Gateway 표준 패턴**을 구축할 수 있다.
