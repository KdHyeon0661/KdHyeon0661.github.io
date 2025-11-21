---
layout: post
title: AWS - EventBridge
date: 2025-07-28 14:20:23 +0900
category: AWS
---
# Amazon EventBridge

## 큰 그림 — 왜 EventBridge인가?

- **디커플링**: 프로듀서와 컨슈머를 시간·공간·규모 면에서 분리
- **서버리스 확장성**: 운영 부담 없이 대량 이벤트 라우팅
- **정교한 필터링**: Event Pattern, Input Transformer, Content-based Routing
- **리플레이/아카이브**: 과거 이벤트로 재처리·재시험 가능
- **크로스-어카운트·SaaS**: 버스 정책·파트너 버스로 도메인 경계 넘는 통합

---

## 핵심 개념 정리(확장)

### 이벤트(Event)

- **사실을 담은 불변 JSON**. 스키마(필드·타입)가 명확할수록 다운스트림 안정성↑
- 필수 키(일반적): `version`, `id`, `time`, `source`, `detail-type`, `detail`
- 권장: **코릴레이션 ID(`correlationId`)** / **트레이스(`traceparent`)**

```json
{
  "version": "0",
  "id": "7f0d1f0e-1234-4d9a-9a44-18a1c4e6d111",
  "time": "2025-11-10T02:31:12Z",
  "source": "com.myorg.orders",
  "detail-type": "OrderCreated",
  "detail": {
    "orderId": "ORD-2025-1101-001",
    "userId": "U-9f1",
    "amount": 129000,
    "items": [{"sku":"BK-001","qty":1}]
  },
  "correlationId": "c-7db6af",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
}
```

### 이벤트 버스(Event Bus)

- `default`: 대부분의 AWS 서비스 이벤트 유입
- `custom`: 애플리케이션·도메인별 생성(권장: **바운디드 컨텍스트 단위**)
- `partner`: SaaS(예: Zendesk, Datadog) 연결

### 규칙(Rule)과 패턴(Event Pattern)

- **패턴 필터**로 이벤트 매칭 → 0..N **대상(Target)** 으로 라우팅
- OR/AND/Prefix/Anything-but/Exists/数値範囲 등 **고급 매칭** 지원

```json
{
  "source": ["com.myorg.orders"],
  "detail-type": ["OrderCreated"],
  "detail": {
    "amount": [{ "numeric": [">=", 100000] }],
    "region": [{ "prefix": "KR-" }],
    "channel": [{ "anything-but": ["test"] }]
  }
}
```

### 대상(Target)

- Lambda / Step Functions / SQS / SNS / API Destinations(HTTP) / Kinesis / ECS RunTask 등
- **Input Transformer**로 대상별 페이로드 모양 변환
- **Dead-Letter Queue(DLQ)**, **리트라이**, **최대 동시 호출 제한** 설정 가능

---

## 아키텍처 패턴

### 도메인 내 라우팅(Internal Pub/Sub)

- `orders` 커스텀 버스 ← `OrderCreated` 발행 → 결제/포인트/알림 각각 규칙으로 분기
- **장점**: 팀 간 느슨한 결합, 신규 컨슈머 온보딩 시 프로듀서 무변경

### 도메인 경계/계정 경계 라우팅(Cross-Account)

- **버스 정책(Resource Policy)** 으로 **다른 AWS 계정**에서 PutEvents 허용
- 중앙 관제 계정에 **아카이브/리플레이** 집중 → 컴플라이언스/감사 용이

### EventBridge Pipes

- **소스(예: SQS/Kinesis/DynamoDB Streams)** → **필터/변환(Lambda/StepFn)** → **대상**
- **규칙(Rule)** 은 *버스에 들어온 이벤트*를 필터링,
  **파이프(Pipe)** 는 *소스 스트림*을 **폴링·전달**(백프레셔, 재시도 포함).
- 배치·수율 제어가 필요하면 **Pipes** 고려.

### API Destinations + Connections

- 외부 HTTP(S) 엔드포인트로 이벤트 푸시.
- OAuth2/Basic/ApiKey **Connections** 관리(자격의 중앙화, 로테이션).

### 스케줄러(EventBridge Scheduler)

- **크론/율 표현식**으로 정기 이벤트 발생(전용 서비스).
- 한 번성·반복성 예약을 **원클릭**으로 설정(지연·재시도 정책 포함).

---

## 리트라이·DLQ·백오프·샘플링

### 기본 리트라이

- 대부분 대상은 **지수 백오프**로 재시도. 실패 누적 시 **DLQ(SQS)** 로 투하(옵션).
- 백오프 예(개념):
  $$
  t_k = t_0 \cdot \alpha^k + \mathrm{Jitter}, \quad \alpha>1
  $$
  - 예: $t_0=1\mathrm{s}, \alpha=2$ → 1s, 2s, 4s, … + 랜덤 지터

### DLQ 설계 팁

- DLQ는 **대상(예: Lambda)별**로 분리 → 재처리 파이프라인 분담
- DLQ → EventBridge Pipe → 보정 Lambda → 원 대상을 향한 **보정 재주입**

### 샘플링/쓰로틀

- Rule 타겟의 **MaximumEventAge/MaximumRetryAttempts** 조정
- Lambda 컨슈머는 **Reserved Concurrency**로 폭주 방지
- Pipes는 **Batch size / Parallelization**으로 수율 제어

---

## 스키마 레지스트리와 코드 생성

- 스키마는 **문서/검증/코드생성**의 근거. 이벤트 버전업 시 **하위호환성** 유지 원칙
- **Schema Discovery** ON → 자동 학습(트래픽 기반) → `aws schemas` CLI 조회
- 코드 생성(예: TypeScript)로 **타이핑된 이벤트 DTO**를 배포 파이프라인에 포함

```bash
aws schemas list-registries
aws schemas list-schemas --registry-name default
aws schemas export-schema \
  --registry-name default \
  --schema-name com.myorg#OrderCreated \
  --schema-version 1 \
  --type OpenApi3
```

---

## 보안: 버스 정책·IAM·대상 권한

### 버스 정책(Resource Policy)

- 누가 **PutEvents**(발행)·**PutRule/PutTargets**(관리) 가능한지 정의
- 크로스 계정/조직(Organizations) **신뢰 경계** 설정

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowOtherAccountToPutEvents",
    "Effect": "Allow",
    "Principal": { "AWS": "111122223333" },
    "Action": "events:PutEvents",
    "Resource": "arn:aws:events:ap-northeast-2:123456789012:event-bus/orders"
  }]
}
```

### 대상 실행 역할(Target Role)

- EventBridge가 **대상 리소스 호출** 시 사용할 **IAM Role**(신뢰 주체: `events.amazonaws.com`)
- 최소권한 원칙: 대상 서비스 API에 필요한 작업만 허용

---

## 운영: 관측/아카이브/리플레이/추적

### 메트릭 & 로그

- Bus/Rule/Target 수준 **매칭·전달 실패·지연** 메트릭 감시
- CloudWatch Alarm → SNS/슬랙/온콜
- **X-Ray**: Lambda 타겟에서 Trace 연동, **correlationId**로 상관분석

### 아카이브/리플레이

- 규칙 또는 버스 단위로 **아카이브 필터** 설정 → 저장
- 장애/버그 시 특정 **시간 범위·패턴**으로 **리플레이**
- 데이터 거버넌스: 아카이브 보관주기·암호화(KMS)

---

## 비용 모델 이해와 절감 팁

- **이벤트 수신(퍼블리시)** 기준 과금(프리 티어 월 100,000건)
- **아카이브/리플레이**: 저장 용량·재생량 과금
- 절감:
  - **규칙 통합/정교한 패턴 필터**로 **불필요 이벤트 차단**
  - **버스 분리**(핫/콜드)로 고빈도 경로 최적화
  - S3·DDB 등은 **게이트웨이 엔드포인트** 통해 네트워크 비용 최소화(다운스트림 접근 시)

---

## 실전 시나리오

### “주문 생성 → 결제/포인트/이메일” 파이프라인

- `orders` 커스텀 버스
- 규칙 1: 금액 ≥ 100000 → 결제 서비스 Lambda
- 규칙 2: 모든 주문 → 포인트 적립 SQS
- 규칙 3: 채널=web → 이메일 SNS 토픽

#### CLI: 버스/규칙/타겟

```bash
# 커스텀 이벤트 버스 생성

aws events create-event-bus --name orders

# 규칙(고가 주문 → 결제)

aws events put-rule \
  --name OrderHighValue \
  --event-bus-name orders \
  --event-pattern '{
    "source": ["com.myorg.orders"],
    "detail-type": ["OrderCreated"],
    "detail": { "amount": [{ "numeric": [">=", 100000] }] }
  }'

# 대상(결제 Lambda)

aws events put-targets \
  --event-bus-name orders \
  --rule OrderHighValue \
  --targets '[
    {
      "Id":"PayLambda",
      "Arn":"arn:aws:lambda:ap-northeast-2:123456789012:function:pay-auth",
      "InputPath":"$.detail",
      "DeadLetterConfig":{"Arn":"arn:aws:sqs:ap-northeast-2:123456789012:orders-dlq"},
      "RetryPolicy":{"MaximumRetryAttempts":5,"MaximumEventAgeInSeconds":3600}
    }
  ]'
```

#### 이벤트 발행(boto3)

```python
import boto3, json, time
ev = boto3.client('events', region_name='ap-northeast-2')

for i in range(3):
    detail = {"orderId": f"ORD-{i}", "userId":"U-9f1", "amount": 129000+i*1000, "channel":"web"}
    ev.put_events(Entries=[{
        "EventBusName": "orders",
        "Source": "com.myorg.orders",
        "DetailType": "OrderCreated",
        "Detail": json.dumps(detail),
        "TraceHeader": "00-4bf92...-...-01"
    }])
    time.sleep(0.1)
```

#### — 타입 안정 처리(스키마 반영 가정)

```python
def handler(event, context):
    # event 는 InputPath("$.detail") 로 들어옴
    order_id = event["orderId"]
    amount   = event["amount"]
    # TODO: 결제 승인 로직
    return {"orderId": order_id, "status": "AUTHORIZED", "approvedAmount": amount}
```

### API Destinations로 외부 웹훅 호출

1) Connection(OAuth2) 생성 → 2) API Destination 생성 → 3) Rule Target 지정

```bash
aws events create-connection \
  --name crm-oauth \
  --authorization-type OAUTH_CLIENT_CREDENTIALS \
  --auth-parameters '{
    "ClientParameters":{"ClientID":"xxx","ClientSecret":"***"},
    "OAuthHttpParameters":{"BodyParameters":[{"Key":"scope","Value":"events"}]},
    "AuthorizationEndpoint":"https://auth.crm.example.com/oauth/token",
    "HttpMethod":"POST"
  }'

aws events create-api-destination \
  --name crm-webhook \
  --connection-arn arn:aws:events:ap-northeast-2:123456789012:connection/crm-oauth/xxx \
  --invocation-endpoint https://api.crm.example.com/webhooks/order \
  --http-method POST

aws events put-targets \
  --event-bus-name orders \
  --rule OrderHighValue \
  --targets '[
    {
      "Id":"CRMWebhook",
      "Arn":"arn:aws:events:ap-northeast-2:123456789012:api-destination/crm-webhook/yyy",
      "InputTransformer":{
        "InputPathsMap":{"orderId":"$.detail.orderId","amount":"$.detail.amount"},
        "InputTemplate":"{\"id\":\"<orderId>\",\"value\":<amount>}"
      }
    }
  ]'
```

### Pipes: SQS → 변환Lambda → Step Functions

```bash
aws pipes create-pipe \
  --name sqs-to-orchestration \
  --source arn:aws:sqs:ap-northeast-2:123456789012:orders-queue \
  --enrichment arn:aws:lambda:ap-northeast-2:123456789012:function:normalize-order \
  --target arn:aws:states:ap-northeast-2:123456789012:stateMachine:order-fulfillment \
  --source-parameters '{
    "SqsQueueParameters": {"BatchSize": 10, "MaximumBatchingWindowInSeconds": 5}
  }' \
  --target-parameters '{
    "StepFunctionStateMachineParameters": {"InvocationType":"FIRE_AND_FORGET"}
  }'
```

---

## Input Transformer/Path/Template

- **InputPath**: JSONPath로 일부만 전달
- **InputTransformer**: 명시적 맵핑 + 템플릿(문자열)로 **형상 고정**

```json
{
  "InputPathsMap": { "id": "$.detail.orderId", "amt": "$.detail.amount" },
  "InputTemplate": "{\"order_id\":\"<id>\",\"amount\":<amt>,\"currency\":\"KRW\"}"
}
```

---

## Terraform/SAM으로 표준 스택 템플릿화

### Terraform

```hcl
resource "aws_cloudwatch_event_bus" "orders" { name = "orders" }

resource "aws_cloudwatch_event_rule" "high_value" {
  name           = "OrderHighValue"
  event_bus_name = aws_cloudwatch_event_bus.orders.name
  event_pattern  = jsonencode({
    "source": ["com.myorg.orders"],
    "detail-type": ["OrderCreated"],
    "detail": { "amount": [{ "numeric": [">=", 100000] }] }
  })
}

resource "aws_lambda_function" "pay" {
  function_name = "pay-auth"
  role          = aws_iam_role.lambda_exec.arn
  handler       = "app.handler"
  runtime       = "python3.12"
  filename      = "build/pay.zip"
}

resource "aws_cloudwatch_event_target" "t_pay" {
  rule           = aws_cloudwatch_event_rule.high_value.name
  event_bus_name = aws_cloudwatch_event_bus.orders.name
  target_id      = "PayLambda"
  arn            = aws_lambda_function.pay.arn
  input_path     = "$.detail"
  dead_letter_config { arn = aws_sqs_queue.orders_dlq.arn }
  retry_policy {
    maximum_event_age_in_seconds = 3600
    maximum_retry_attempts       = 5
  }
}

resource "aws_lambda_permission" "allow_events" {
  statement_id  = "AllowExecutionFromEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.pay.arn
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.high_value.arn
}
```

### AWS SAM(간결)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  OrdersBus:
    Type: AWS::Events::EventBus
    Properties: { Name: orders }

  PayFn:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: python3.12
      Handler: app.handler
      CodeUri: pay/
      Policies:
        - AWSLambdaBasicExecutionRole
      Events:
        HighValueOrder:
          Type: EventBridgeRule
          Properties:
            EventBusName: !Ref OrdersBus
            Pattern:
              source: ["com.myorg.orders"]
              detail-type: ["OrderCreated"]
              detail:
                amount:
                  - numeric: [">=", 100000]
            InputPath: "$.detail"
```

---

## 트러블슈팅 체크리스트

- [ ] **이벤트가 규칙에 매칭되지 않음** → Event Pattern(JSONPath/타입/대소문자) 확인
- [ ] **대상 호출 권한 실패** → 대상 실행 역할 신뢰주체 `events.amazonaws.com` 확인
- [ ] **크로스 계정 발행 불가** → **버스 정책**의 Principal/Action/Resource 재확인
- [ ] **리트라이 고갈** → DLQ 메시지 확인, **보정 파이프라인** 설계
- [ ] **외부 API 실패** → API Destinations Connection 만료/스루풋 제한 확인
- [ ] **폭주** → Rule 타겟 동시성 제한, Pipes 배치/동시성, Lambda 예약 동시성

---

## 관측/분석(CloudWatch Logs Insights 예)

**대상 실패 로그 상위 에러메시지 Top-N**

```sql
fields @timestamp, @message
| filter @message like /EventBridge.*Invoke.*Error/
| parse @message /errorMessage":"(?<err>[^"]+)"/
| stats count() as c by err
| sort c desc
| limit 20
```

**지연 이상 탐지(단순 기준)**

```sql
fields @timestamp, latencyMs, target
| filter target = "pay-auth"
| stats avg(latencyMs) as p50, pct(latencyMs,95) as p95 by bin(5m)
```

---

## 도메인 버스 분할과 버전 전략

- **버스 분할**: `orders`, `payments`, `users` … 바운디드 컨텍스트별
- **이벤트 버전**: `detail-type: OrderCreated.v2` 또는 `detail.schemaVersion = "2"`
- **하위호환**: 필드 추가는 허용, **의미 변경/삭제는 새 버전**

---

## 보안·거버넌스 베스트 프랙티스

- **Least Privilege**: PutEvents/PutTargets 최소 권한
- **버스 정책에 조직 단위 허용**(AWS Organizations) → 계정 확장 편의
- **스키마 검증**: 사전 검증(프록시 Lambda) 또는 컨슈머 측 강타입 DTO
- **아카이브 암호화(KMS)**, 보존주기 정의, PII 필드 최소화/마스킹
- **태깅**(비용/데이터계보): `eventbus=orders`, `pii=no`, `env=prod`

---

## 성능·지연·정확성 고려

- EventBridge는 **최소-1회 전달(at-least-once)** → **아이템포턴시 키**로 **중복 처리**
- 순서 보장은 기본적으로 없음(필요 시 **FIFO SQS** → Pipes → 대상)
- 대량 이벤트: **배치형 Pipes** / 대상의 **버퍼링** / 다운스트림 **수평 확장**
- 대기시간 민감 경로: Rule 체인을 최소화, 지역 일치, 네트워크 경로 단축(API Destinations 근접)

---

## 엔드투엔드 예제 — 주문 시스템(요약 코드)

### 프로듀서(Node.js)

```js
import { EventBridgeClient, PutEventsCommand } from "@aws-sdk/client-eventbridge";
const eb = new EventBridgeClient({ region: "ap-northeast-2" });

export async function publishOrderCreated(order) {
  await eb.send(new PutEventsCommand({
    Entries: [{
      EventBusName: "orders",
      Source: "com.myorg.orders",
      DetailType: "OrderCreated",
      Detail: JSON.stringify(order),
      TraceHeader: "00-..."
    }]
  }));
}
```

### — InputTransformer로 정규화된 입력 가정

```python
def handler(event, context):
    # event = {"order_id":"ORD-1","amount":120000,"currency":"KRW"}
    if event["amount"] >= 100000:
        # 고가 주문 처리
        ...
    return {"ok": True}
```

### 재처리(리플레이)

```bash
aws events create-archive \
  --archive-name orders-archive \
  --event-source-arn arn:aws:events:ap-northeast-2:123456789012:event-bus/orders \
  --retention-days 30

aws events start-replay \
  --replay-name replay-highvalue \
  --event-start-time 2025-11-09T00:00:00Z \
  --event-end-time   2025-11-10T00:00:00Z \
  --destination '{"Arn":"arn:aws:events:ap-northeast-2:123456789012:event-bus/orders"}'
```

---

## 요약 체크리스트

- [ ] **커스텀 버스**로 도메인 분리, 규칙은 **도메인 용어**로
- [ ] **Event Pattern**으로 **필터링·분기**를 먼저, 변환은 최소화
- [ ] **아이템포턴시/중복처리** 내재화, 순서 필요 시 **FIFO+Pipes**
- [ ] **리트라이+DLQ** 표준화, **아카이브/리플레이**로 회복력 확보
- [ ] **버스 정책/IAM/연결 권한** 최소화, **스키마 관리** 자동화
- [ ] **관측(메트릭/로그/X-Ray)** 상시, 비용·지연 모니터링
- [ ] **IaC(SAM/Terraform)** 로 재현 가능한 배포

---

## 수학으로 보는 백오프 + 지터

- **Exponential Backoff with Jitter**(추천):
  $$
  t_k = \min(T_{\max}, \mathrm{rand}(0, t_0 \cdot 2^{k}))
  $$
  - 충돌(스파이크) 완화, 꼬리 지연 감소

---

## CloudFormation(리소스 최소 샘플)

```yaml
Resources:
  OrdersBus:
    Type: AWS::Events::EventBus
    Properties: { Name: orders }

  OrderHighValueRule:
    Type: AWS::Events::Rule
    Properties:
      EventBusName: !Ref OrdersBus
      EventPattern:
        source: ["com.myorg.orders"]
        detail-type: ["OrderCreated"]
        detail:
          amount:
            - numeric: [">=", 100000]
      Targets:
        - Id: PayLambda
          Arn: arn:aws:lambda:ap-northeast-2:123456789012:function:pay-auth
          InputPath: "$.detail"
          DeadLetterConfig:
            Arn: arn:aws:sqs:ap-northeast-2:123456789012:orders-dlq
          RetryPolicy:
            MaximumEventAgeInSeconds: 3600
            MaximumRetryAttempts: 5
```

---

## 결론

EventBridge는 **서버리스 이벤트 허브**로, 마이크로서비스·워크플로우·SaaS/외부 시스템을 **정교한 필터·정책·리트라이/리플레이**로 견고하게 묶는다.
본 가이드의 **도메인 버스 분할, 패턴 설계, Pipes/스케줄러 선택, 보안·거버넌스, DLQ/아카이브 운영, IaC 표준**을 적용하면 **대규모·복원력 높은 이벤트 아키텍처**를 일관되게 구축·운영할 수 있다.
