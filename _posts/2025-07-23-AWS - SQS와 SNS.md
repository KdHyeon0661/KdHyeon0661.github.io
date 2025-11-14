---
layout: post
title: AWS - SQS와 SNS
date: 2025-07-23 23:20:23 +0900
category: AWS
---
# AWS 메시징 시스템: SQS와 SNS

## 한 장 요약

- **SQS**: Poll 기반 **큐**(표준/ FIFO). 백엔드 비동기 처리, 버퍼링, 스로틀 보호, **일시적 장애 흡수**에 최적.
- **SNS**: Push 기반 **토픽**. 이벤트 브로드캐스트, 멀티 구독자(Email/SMS/HTTP/Lambda/SQS), **필터링·팬아웃**에 최적.
- 베스트 프랙티스: **멱등성**, **DLQ + 레드라이브**, **가시성 타임아웃**/롱폴링 튜닝, **메시지 필터**로 팬아웃 비용 감소, **KMS 암호화**와 최소권한 IAM, **CloudWatch 지표** 모니터링.

---

## 개념 모델 — 역할과 데이터 흐름

### 용어 정리

- **Producer(발행자)**: 메시지를 보낸다(SQS로는 SendMessage, SNS로는 Publish).
- **Queue(큐)**: 메시지 임시 저장소(SQS).
- **Consumer(소비자)**: 큐를 Poll → 처리 → **삭제(DeleteMessage)**.
- **Topic(토픽)**: 발행/구독 허브(SNS).
- **Subscriber(구독자)**: SNS에서 알림을 받는 엔드포인트(SQS, Lambda, HTTP 등).
- **Visibility Timeout**: 소비자가 가져간 메시지를 **일시 숨김**. 처리 실패 시 재노출.
- **DLQ(Dead-Letter Queue)**: 재시도에도 실패한 메시지 격리 보관.
- **Redrive**: DLQ에 쌓인 메시지를 **원 큐/람다로 재주입**.

### 선택 기준 (요약)

| 요구사항 | 권장 |
|---|---|
| 백그라운드 처리, 버퍼링, 스로틀 보호 | **SQS** |
| 하나의 이벤트를 다수 시스템에 브로드캐스트 | **SNS** |
| 순서보장 + 중복방지 | **SQS FIFO** |
| 속성 기반 라우팅 | **SNS 필터 정책 + SQS 구독** |
| 정확히 한 번 같은 효과 | **SQS FIFO + 멱등성**(애플리케이션 계층) |

---

## SQS 심화 — 표준 vs FIFO, 동작·튜닝·패턴

### 표준(Standard) vs FIFO

| 항목 | Standard | FIFO |
|---|---|---|
| 순서 | 베스트 에포트(보장 아님) | **메시지그룹 단위** 순서 보장 |
| 중복 | 드문 중복 가능 | **중복 방지**(DeduplicationId) |
| 처리량 | 사실상 무제한 | 1,000 msg/s(배치 시 더↑), 그룹 병렬화로 수평 확장 |
| 지연/비용 | 저지연/저비용 | 약간 더 비쌀 수 있음 |

**FIFO 핵심 필드**:
- `MessageGroupId`: 이 값이 같은 메시지는 **순서 보장** + **동시에 하나만 처리**.
- `MessageDeduplicationId`: 5분 내 **중복 방지**. `ContentBasedDeduplication`을 켜면 본문 해시로 자동.

### 메시지 수명주기

1) **SendMessage** → 큐 저장
2) **ReceiveMessage**(롱폴링 권장) → 메시지와 **ReceiptHandle** 획득
3) **처리**
4) **DeleteMessage**(ReceiptHandle 필요) → 최종 삭제
5) 실패 시 **Visibility Timeout** 만료 후 재노출 → 재시도

### 핵심 속성·튜닝 포인트

- **VisibilityTimeout**: 처리시간 + 여유(네트워크/재시도)를 반영.
  - 처리 중 추가 시간이 필요하면 **ChangeMessageVisibility**로 연장.
- **ReceiveMessageWaitTimeSeconds**: **롱폴링**(0~20초). 빈 응답/과금 감소 + 레이턴시 안정화.
- **MessageRetentionPeriod**: 1분~14일. 비용/복구 요구에 맞춰 설정.
- **DelaySeconds**: 큐 전체 기본 지연(0~15분).
- **ReceiveMessage** `MaxNumberOfMessages`: 배치(최대 10). 처리량↑/요청수↓.
- **RedrivePolicy**: DLQ 연결 + **maxReceiveCount**.

### 처리량·동시성 산정 (개념)

- 초당 처리량 목표를 \(R\), 1건 평균 처리시간을 \(T\)라 하면 **필요 동시성**:
$$
\text{필요 동시성} \approx R \times T
$$
- **가시성 타임아웃(V)** 는 \(T\)보다 커야 하며, 네트워크/재시도 여유를 더한다:
$$
V \ge T + \Delta
$$

### 실패/독성 메시지(포이즌 메시지)

- 같은 메시지가 **maxReceiveCount**를 초과하면 **DLQ**로 이동.
- DLQ는 **레드라이브(수동/자동)**로 원 큐나 람다 재처리 파이프라인으로 되돌린다.
- **멱등성**(DynamoDB 조건식/토큰)으로 **중복 재시도**의 부작용 방지.

### 대용량 페이로드

- 메시지 본문 한도: **256KB**.
- 더 크면 **S3 Extended Client 패턴**(본문은 S3, SQS에는 포인터) 사용.

---

## SNS 심화 — 토픽, 구독, 필터링·재시도

### 토픽/구독/전달

- **Publish → Topic → N개의 구독자**(SQS/Lambda/HTTP/Email/SMS 등).
- **푸시 모델**: 지연이 매우 낮고 설계가 단순.
- **구독별 재시도 정책**(HTTP는 지수백오프), 일부 프로토콜은 **DLQ(구독 레벨)** 설정 가능.

### 메시지 속성·필터 정책

- 발행 시 **MessageAttributes**를 넣고, 구독 시 **FilterPolicy**로 조건 매칭.
- **서브셋 라우팅**으로 **불필요한 전송/소비 비용** 절감.

```json
{
  "FilterPolicy": {
    "eventType": ["order.created", "order.cancelled"],
    "region": ["ap-northeast-2", "us-east-1"],
    "priority": [{ "numeric": [">=", 5] }]
  }
}
```

### 팬아웃(팬아웃 + 필터)

- **단일 Publish** → **여러 SQS 큐/람다/HTTP**로 동시 전파.
- 큐별 **필터 정책**으로 서로 다른 서브셋만 수신.

---

## 보안 — IAM/KMS/리소스 정책/VPC Endpoint

### 최소권한 IAM 예시

**SQS 생산자(쓰기 전용)**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect":"Allow",
    "Action":["sqs:SendMessage","sqs:GetQueueAttributes","sqs:GetQueueUrl"],
    "Resource":"arn:aws:sqs:ap-northeast-2:123456789012:my-queue"
  }]
}
```

**SQS 소비자(읽기/삭제)**
```json
{
  "Version":"2012-10-17",
  "Statement":[
    {"Effect":"Allow","Action":["sqs:ReceiveMessage","sqs:DeleteMessage","sqs:ChangeMessageVisibility","sqs:GetQueueAttributes","sqs:GetQueueUrl"],"Resource":"arn:aws:sqs:ap-northeast-2:123456789012:my-queue"}
  ]
}
```

**SNS 발행자**
```json
{
  "Version":"2012-10-17",
  "Statement":[
    {"Effect":"Allow","Action":["sns:Publish"],"Resource":"arn:aws:sns:ap-northeast-2:123456789012:my-topic"}
  ]
}
```

### 리소스 정책(큐에 토픽만 Publish 허용)

```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"AllowOnlyMyTopic",
      "Effect":"Allow",
      "Principal":"*",
      "Action":"sqs:SendMessage",
      "Resource":"arn:aws:sqs:ap-northeast-2:123456789012:orders",
      "Condition":{
        "ArnEquals":{"aws:SourceArn":"arn:aws:sns:ap-northeast-2:123456789012:orders-topic"}
      }
    }
  ]
}
```

### 암호화

- **SQS SSE(KMS)**, **SNS SSE(KMS)** 지원. 키 정책에 **서비스 역할**/프로듀서 권한 고려.

### 사설 접근

- **VPC Endpoint (Interface/Gateway)** 로 인터넷 없이 SQS/SNS 호출. 엔드포인트 정책으로 **더 축소** 가능.

---

## 운영·관측성 — 메트릭·알람·로그

### 주요 메트릭(SQS)

- `ApproximateNumberOfMessagesVisible`: 대기 중 메시지 수(백로그).
- `ApproximateNumberOfMessagesNotVisible`: 처리 중(가시성 타임아웃 내) 메시지 수.
- `NumberOfMessagesReceived/Deleted/Sent`: 처리 흐름 건수.
- FIFO: `SentMessageSize`, 그룹 병목 징후 모니터.

**백로그 알람 예시**
- 임계: **백로그 > 목표 처리율 × 허용 지연 시간**.

### 주요 메트릭(SNS)

- `NumberOfMessagesPublished` / `Delivered` / `Failed`
- **구독별** 전달 실패율/재시도 모니터

### 로그

- SQS/SNS 자체 로그는 적다 → **소비자(Lambda/EC2/ECS) 로그** + **CloudTrail**(API 감시) + **X-Ray**(연쇄 호출 추적)로 보강.

---

## 비용 모델·계산

### SQS

- **요청 수 기준 과금**(Send/Receive/Delete/ChangeVisibility) + **데이터 전송**.
- 롱폴링으로 **빈 Receive**를 줄여 비용 절감.
- FIFO는 표준 대비 단가↑ 가능 → **그룹 병렬화 설계**로 처리량 확보.

### SNS

- **Publish/Delivery 건수** + 프로토콜별 비용.
- **필터 정책**으로 불필요한 전송 억제(=비용 절감).

### 개념 공식

월간 SQS 비용(개략):
$$
\text{Cost} \approx C_{\text{req}}\cdot(N_{\text{send}}+N_{\text{recv}}+N_{\text{del}})+C_{\text{data}}\cdot \text{GB}
$$

SNS 팬아웃:
$$
N_{\text{delivery}} \approx N_{\text{publish}} \times N_{\text{subscribers(matched)}}
$$

---

## SQS — CLI/SDK 빠른 실습

### CLI

```bash
# 큐 생성 (표준)

aws sqs create-queue --queue-name std-queue

# FIFO 큐 생성

aws sqs create-queue --queue-name orders.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=true

# 전송

aws sqs send-message --queue-url $URL --message-body 'hello'

# FIFO 전송(그룹/중복방지)

aws sqs send-message --queue-url $FIFO_URL \
  --message-body '{"orderId":123}' \
  --message-group-id orders-apne2 --message-deduplication-id 123

# 수신(롱폴링 20초)

aws sqs receive-message --queue-url $URL --wait-time-seconds 20 --max-number-of-messages 10

# 삭제

aws sqs delete-message --queue-url $URL --receipt-handle $HANDLE
```

### Python(boto3) 소비자 — 부분실패 보고(Lambda 통합과 동일 패턴)

```python
import json, boto3
from botocore.exceptions import ClientError
sqs = boto3.client("sqs")
QUEUE_URL = "https://sqs.ap-northeast-2.amazonaws.com/123456789012/orders"

def handler(event, _):
    # Lambda SQS 이벤트와 동일한 형식이라고 가정
    failures = []
    for rec in event["Records"]:
        try:
            body = json.loads(rec["body"])
            process(body)  # 멱등하게
        except Exception:
            failures.append({"itemIdentifier": rec["messageId"]})
    return {"batchItemFailures": failures}
```

---

## SNS — CLI/필터링/팬아웃

```bash
# 토픽 생성

aws sns create-topic --name orders-topic

# 구독자: SQS 큐 등록

aws sns subscribe --topic-arn $TOPIC_ARN \
  --protocol sqs --notification-endpoint $QUEUE_ARN

# 구독 필터 정책

aws sns set-subscription-attributes \
  --subscription-arn $SUB_ARN \
  --attribute-name FilterPolicy \
  --attribute-value '{"eventType":["order.created"],"priority":[{"numeric":[">=",5]}]}'

# 발행

aws sns publish --topic-arn $TOPIC_ARN \
  --message '{"orderId":123,"amount":5000}' \
  --message-attributes '{"eventType":{"DataType":"String","StringValue":"order.created"},"priority":{"DataType":"Number","StringValue":"5"}}'
```

---

## SQS + SNS 팬아웃 — 아키텍처와 리소스 정책

**시나리오**: `orders-topic` → `orders-processor-queue`, `orders-email-queue`, `orders-analytics-queue`
- 각 큐에 **서로 다른 FilterPolicy** 적용.
- 큐의 리소스 정책으로 **해당 토픽에서만 Send 허용**.

큐 정책(요지)은 4.2의 예시 참고.

---

## Lambda 통합 — SQS Pull / SNS Push

### SQS → Lambda (이벤트 소스 매핑)

- **배치 크기**(1~10), **배치 윈도우**, **부분실패 보고** 지원.
- **가시성 타임아웃 ≥ 함수 타임아웃** + 여유.
- DLQ 연결(큐 레벨) + Lambda **on-failure Destinations**(비동기) 조합 가능.

**Lambda(Python) — 부분실패/멱등성 + DDB 조건식**
```python
import json, boto3, os
ddb = boto3.client("dynamodb")
TABLE = os.environ["TABLE"]

def process(o):
    ddb.put_item(
        TableName=TABLE,
        Item={"pk":{"S":f"order#{o['orderId']}"},"status":{"S":"processed"}},
        ConditionExpression="attribute_not_exists(pk)" # 멱등
    )

def lambda_handler(event, _):
    failures=[]
    for r in event["Records"]:
        try:
            process(json.loads(r["body"]))
        except Exception:
            failures.append({"itemIdentifier": r["messageId"]})
    return {"batchItemFailures": failures}
```

### SNS → Lambda

- 푸시형. 재시도/백오프는 SNS가 처리.
- 처리 실패 시 **SNS 구독 DLQ**(일부 프로토콜) 또는 Lambda Destinations 고려.

---

## IaC — Terraform & SAM 스니펫

### Terraform (SNS + 두 개의 SQS + 필터)

```hcl
resource "aws_sns_topic" "orders" {
  name = "orders-topic"
}

resource "aws_sqs_queue" "high" { name = "orders-high" }
resource "aws_sqs_queue" "low"  { name = "orders-low"  }

resource "aws_sns_topic_subscription" "high_sub" {
  topic_arn = aws_sns_topic.orders.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.high.arn
  filter_policy = jsonencode({
    priority = [{ numeric = [">=", 5] }]
  })
}

resource "aws_sns_topic_subscription" "low_sub" {
  topic_arn = aws_sns_topic.orders.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.low.arn
  filter_policy = jsonencode({
    priority = [{ numeric = ["<", 5] }]
  })
}

# 큐 정책(토픽만 Send 허용)

data "aws_iam_policy_document" "q" {
  statement {
    actions = ["sqs:SendMessage"]
    resources = [aws_sqs_queue.high.arn, aws_sqs_queue.low.arn]
    principals { type = "*", identifiers = ["*"] }
    condition {
      test     = "ArnEquals"
      variable = "aws:SourceArn"
      values   = [aws_sns_topic.orders.arn]
    }
  }
}
resource "aws_sqs_queue_policy" "qp" {
  queue_url = aws_sqs_queue.high.id
  policy    = data.aws_iam_policy_document.q.json
}
resource "aws_sqs_queue_policy" "qp2" {
  queue_url = aws_sqs_queue.low.id
  policy    = data.aws_iam_policy_document.q.json
}
```

### SAM (S3 → SNS → SQS → Lambda)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  Topic:
    Type: AWS::SNS::Topic
    Properties: { TopicName: s3-events }

  Queue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: s3-processor
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt DLQ.Arn
        maxReceiveCount: 5

  DLQ:
    Type: AWS::SQS::Queue
    Properties: { QueueName: s3-processor-dlq }

  Sub:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref Topic
      Protocol: sqs
      Endpoint: !GetAtt Queue.Arn

  QueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues: [!Ref Queue]
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: "*"
            Action: sqs:SendMessage
            Resource: !GetAtt Queue.Arn
            Condition:
              ArnEquals: { aws:SourceArn: !Ref Topic }

  Fn:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: python3.12
      CodeUri: src/
      Handler: app.lambda_handler
      Policies:
        - SQSPollerPolicy:
            QueueName: !GetAtt Queue.QueueName
      Events:
        Q:
          Type: SQS
          Properties:
            Queue: !GetAtt Queue.Arn
            BatchSize: 10
            MaximumBatchingWindowInSeconds: 3
```

---

## 고급 패턴

- **Outbox 패턴**: 트랜잭션 DB에 이벤트 레코드 저장 → 별도 워커가 SNS/SQS로 퍼블리시.
- **Saga/Orchestration**: 단계별 보상 트랜잭션 → **Step Functions + SNS/SQS** 조합.
- **Command vs Event 분리**: SQS는 작업/명령 처리, SNS는 상태변화 알림.
- **FIFO 멀티 그룹 병렬화**: `MessageGroupId`를 엔티티/샤드 키로 설정해 확장.
- **멀티리전**: SNS → SQS(CRR 없음) → 자체 복제/재발행 로직 설계 혹은 EventBridge 글로벌 버스 검토.

---

## 실전 엔드투엔드 — 주문 이벤트 파이프라인

### 요구

- 주문 생성/취소 이벤트를 발행 → **HighPriority**(우선처리), **Billing**, **Audit**에 각각 전달.
- **우선처리**는 **FIFO**로 순서 보장, **멱등성** 필수.
- 장애 시 DLQ, 운영자는 **레드라이브**로 재처리.

### 흐름

`Order Service → SNS(orders-topic)`
→ `SQS FIFO (high-priority.fifo)` [필터: priority>=5]
→ `SQS (billing)` [필터: eventType in {created,cancelled}]
→ `SQS (audit)` [필터: 모두]

**발행 코드(Node.js v3)**
```js
import { SNSClient, PublishCommand } from "@aws-sdk/client-sns";
const sns = new SNSClient({ region: "ap-northeast-2" });
export const publishOrder = async (evt) => {
  await sns.send(new PublishCommand({
    TopicArn: process.env.TOPIC_ARN,
    Message: JSON.stringify(evt),
    MessageAttributes: {
      eventType: { DataType: "String", StringValue: evt.type },          // e.g., order.created
      priority:  { DataType: "Number", StringValue: String(evt.priority) }
    }
  }));
};
```

**HighPriority 소비자(Python, FIFO 멱등 + DDB 조건)**
```python
import json, boto3, os
ddb = boto3.client("dynamodb")
TABLE = os.environ["TABLE"]

def lambda_handler(event, _):
    failures=[]
    for r in event["Records"]:
        body = json.loads(r["body"])
        msg = json.loads(body["Message"]) if "Message" in body else body
        key = f"order#{msg['orderId']}"
        try:
            ddb.put_item(
                TableName=TABLE,
                Item={"pk":{"S":key},"status":{"S":"processed"}},
                ConditionExpression="attribute_not_exists(pk)"  # 멱등
            )
            # ... 실제 우선 처리 로직 ...
        except Exception:
            failures.append({"itemIdentifier": r["messageId"]})
    return {"batchItemFailures": failures}
```

**운영**
- 백로그 급증 시 **BatchSize/Window/동시성** 상향.
- DLQ 적재 이벤트에 **CloudWatch 알람** → 오퍼레이터가 레드라이브.
- 지표 대시보드: `Visible`, `NotVisible`, `AgeOfOldestMessage`, Lambda `Errors/Duration/Throttles`.

---

## 체크리스트

- [ ] **롱폴링** 활성화(ReceiveMessageWaitTimeSeconds).
- [ ] **가시성 타임아웃 ≥ 처리시간 + 여유**.
- [ ] **DLQ + maxReceiveCount** 설정, 레드라이브 경로 마련.
- [ ] **멱등성**(조건식/토큰/해시) 구현.
- [ ] **필터 정책**으로 불필요한 전송 줄이기.
- [ ] **암호화(KMS)** + 최소권한 IAM + **리소스 정책**으로 출처 제한.
- [ ] **CloudWatch 대시보드/알람**: 백로그/실패율/지연.
- [ ] **테스트: 장애 주입**(처리 실패/지연/네트워크 단절)으로 재시도·DLQ 동작 검증.
- [ ] **비용 리포트**로 요청 수/데이터 전송/스토리지 확인.

---

## 부록 — 자주 쓰는 스니펫

### 큐 속성 변경(롱폴링/가시성)

```bash
aws sqs set-queue-attributes --queue-url $URL \
  --attributes ReceiveMessageWaitTimeSeconds=20,VisibilityTimeout=120
```

### 메시지 가시성 연장

```bash
aws sqs change-message-visibility --queue-url $URL \
  --receipt-handle $HANDLE --visibility-timeout 300
```

### DLQ 연결

```bash
aws sqs set-queue-attributes --queue-url $URL \
  --attributes RedrivePolicy='{"deadLetterTargetArn":"ARN_OF_DLQ","maxReceiveCount":"5"}'
```

### SNS 구독 확인(HTTP/Email)

```bash
# 이메일은 수신함에서 Confirm 필요

aws sns list-subscriptions-by-topic --topic-arn $TOPIC_ARN
```

---

## 마무리

SQS와 SNS는 **비동기·이벤트 기반 아키텍처의 핵심 빌딩 블록**이다.
- **SQS**로 백엔드 처리의 **탄력·복원력**을 높이고,
- **SNS**로 이벤트를 **필터링·팬아웃**하여 여러 다운스트림을 느슨하게 연결하라.

운영 성공의 비결은 **멱등성 + DLQ/레드라이브 + 관측성 + 보안 최소권한 + 비용 인지**이다. 본 가이드의 패턴과 IaC·코드 스니펫을 템플릿화하면, **재사용 가능한 표준 메시징 플랫폼**을 빠르게 구축할 수 있다.
