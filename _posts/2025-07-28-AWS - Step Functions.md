---
layout: post
title: AWS - Step Functions
date: 2025-07-28 15:20:23 +0900
category: AWS
---
# AWS Step Functions: 서버리스 워크플로우 오케스트레이션

## 0) 한눈에 보는 Step Functions

- **무상태 코드를 상태로 감싼다**: 비즈니스 로직을 **상태 기계(State Machine)** 로 선언(ASL: Amazon States Language).
- **서버리스 오케스트레이션**: Lambda, ECS, Batch, Glue, SageMaker, DynamoDB, SQS/SNS, EventBridge, EMR 등과 **코드 없이 연결**.
- **표준 vs 익스프레스**: 장시간/정밀 추적(표준) ↔ 초저비용·초고처리량(익스프레스).
- **실패 내성**: 내장 `Retry`/`Catch`, 지수 백오프, 서킷브레이커 패턴 구성 가능.
- **관측성**: 실행 이력/상태별 입력·출력·로그/X-Ray 트레이싱.

---

## 1) Amazon States Language(ASL) 핵심 문법

### 1.1 최소 예제

```json
{
  "Comment": "초간단 2단계",
  "StartAt": "Step1",
  "States": {
    "Step1": {
      "Type": "Pass",
      "Result": {"hello": "world"},
      "ResultPath": "$.step1",
      "Next": "Done"
    },
    "Done": { "Type": "Succeed" }
  }
}
```

### 1.2 주요 상태 타입 요약

| 타입 | 용도 | 핵심 속성 |
|---|---|---|
| `Task` | 서비스 호출/Lambda/ECS 등 | `Resource`, `Parameters`, `TimeoutSeconds`, `Retry`, `Catch` |
| `Choice` | 분기 | `Choices`(조건), `Default` |
| `Wait` | 대기/타임아웃 | `Seconds`/`Timestamp` |
| `Parallel` | 분기 병렬 | `Branches` |
| `Map` | 리스트 반복 | `ItemsPath`, `Iterator`, `MaxConcurrency` |
| `Succeed`/`Fail` | 종료 | `Cause`, `Error` |
| `Pass` | 변환/전달 | `Parameters`, `ResultPath` |

### 1.3 입력/출력 경로 3총사

- **`InputPath`**: 들어오는 JSON에서 사용할 **부분만** 선택
- **`Parameters`**: 입력을 **새 JSON**으로 **구성**(`.$` 사용 시 JSON 경로 바인딩)
- **`ResultPath`**: Task 결과를 **어디에 병합**할지 결정(`null`이면 결과 버림)
- **`OutputPath`**: 다음 상태로 **내보낼** 서브셋

```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "InputPath": "$.order",
  "Parameters": {
    "FunctionName": "arn:aws:lambda:ap-northeast-2:123:function:ValidateOrder",
    "Payload.$": "$"
  },
  "ResultPath": "$.validation",
  "OutputPath": "$"
}
```

### 1.4 Intrinsic Functions(내장 함수)

- 문자열/JSON 조작: `States.Format`, `States.StringToJson`, `States.JsonToString`
- 배열/수학: `States.Array`, `States.ArrayContains`, `States.MathRandom`
- 시간: `States.Timestamp`, `States.TimestampAdd`, `States.TimestampGreaterThan`
- 예시:

```json
"Parameters": {
  "message.$": "States.Format('주문 {} 건이 접수되었습니다', $.order.id)",
  "when.$": "States.Timestamp()"
}
```

---

## 2) 오류 처리: Retry/Catch, 지수 백오프, 조건부 재시도

### 2.1 Retry with Backoff

```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Retry": [
    {
      "ErrorEquals": ["Lambda.ServiceException","Lambda.TooManyRequestsException"],
      "IntervalSeconds": 2,
      "MaxAttempts": 6,
      "BackoffRate": 2.0
    }
  ],
  "Catch": [
    { "ErrorEquals": ["States.ALL"], "Next": "HandleFailure" }
  ],
  "Next": "NextStep"
}
```

### 2.2 서킷브레이커 느낌의 Choice 결합

- 실패 누적 카운트를 **상태 입력**에 유지 → 임계 초과 시 **빠른 우회 경로**로 전송.

```json
{
  "StartAt": "TryCall",
  "States": {
    "TryCall": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "IncFail" }],
      "Next": "OK"
    },
    "IncFail": {
      "Type": "Pass",
      "ResultPath": "$.failures",
      "Parameters": { "count.$": "States.MathAdd($.failures.count, 1)" },
      "Next": "MaybeOpenCircuit"
    },
    "MaybeOpenCircuit": {
      "Type": "Choice",
      "Choices": [
        { "Variable": "$.failures.count", "NumericGreaterThanEquals": 3, "Next": "Fallback" }
      ],
      "Default": "WaitAndRetry"
    },
    "WaitAndRetry": { "Type": "Wait", "Seconds": 30, "Next": "TryCall" },
    "Fallback": { "Type": "Task", "Resource": "arn:aws:states:::sns:publish", "End": true },
    "OK": { "Type": "Succeed" }
  }
}
```

---

## 3) 동적 반복: Map, **Distributed Map**, 병렬/팬아웃

### 3.1 Map(일반)

```json
{
  "Type": "Map",
  "ItemsPath": "$.items",
  "MaxConcurrency": 10,
  "Iterator": {
    "StartAt": "HandleItem",
    "States": {
      "HandleItem": {
        "Type": "Task",
        "Resource": "arn:aws:states:::lambda:invoke",
        "Parameters": { "FunctionName": "arn:...:Handle", "Payload.$": "$" },
        "End": true
      }
    }
  },
  "ResultPath": "$.batchResults"
}
```

### 3.2 Distributed Map(대규모 병렬)

- **수십만~수백만** 항목 팬아웃을 **서브워크플로우**로 분산 처리.
- 소스: S3(리스트/매니페스트), Items: 대규모 JSON.

```json
{
  "Type": "Map",
  "ItemProcessor": {
    "ProcessorConfig": { "Mode": "DISTRIBUTED" },
    "StartAt": "Process",
    "States": {
      "Process": {
        "Type": "Task",
        "Resource": "arn:aws:states:::lambda:invoke",
        "Parameters": { "FunctionName": "arn:...:ProcessItem", "Payload.$": "$" },
        "End": true
      }
    }
  },
  "ItemReader": {
    "Resource": "arn:aws:states:::s3:listObjectsV2",
    "Parameters": { "Bucket": "my-bucket", "Prefix": "incoming/" }
  },
  "MaxConcurrency": 1000,
  "ResultPath": "$.distResults"
}
```

---

## 4) 서비스 통합 패턴: `.sync`, 콜백(토큰), Activity, ECS/Batch/Glue

### 4.1 **Request-Response** (Lambda, DynamoDB 등)

```json
{ "Type": "Task", "Resource": "arn:aws:states:::lambda:invoke", "Parameters": { ... } }
```

### 4.2 **`.sync` Job-run** 패턴(Glue/ECS/Batch/SageMaker 등 장시간 잡 대기)

```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::glue:startJobRun.sync",
  "Parameters": {
    "JobName": "etl-job",
    "Arguments": { "--dt.$": "$.dt" }
  },
  "Next": "AfterGlue"
}
```

- `.sync` 를 쓰면 잡 완료까지 **상태가 대기**하며 결과/상태코드 취득.

### 4.3 **콜백(Wait for Callback with Task Token)**

- 외부 시스템/사람이 **토큰으로 완료 신호**를 줄 때까지 대기.

```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
  "Parameters": {
    "FunctionName": "arn:...:RequestApproval",
    "Payload": {
      "taskToken.$": "$$.Task.Token",
      "request.$": "$.request"
    }
  },
  "TimeoutSeconds": 3600,
  "Next": "Continue"
}
```

- 외부에서 완료:
  - `SendTaskSuccess` / `SendTaskFailure`(SDK/CLI) 로 토큰에 응답.

### 4.4 **Activity** (커스텀 워커 Pull)

- Step Functions Activity 생성 → 온프레미스/EC2 워커가 **작업 폴링** 후 결과 회신.  
  (현업에선 **Lambda/ECS + 콜백** 선호, Activity는 레거시 통합에 유용)

---

## 5) 실전 시나리오 1: **주문 처리 SAGA(보상 트랜잭션)**

### 5.1 요구사항
- 결제 승인 → 재고 예약 → 출고 요청  
- 중간 실패 시 **보상(취소/롤백)**

### 5.2 상태 기계(요지)

```json
{
  "StartAt": "Charge",
  "States": {
    "Charge": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "arn:...:Charge", "Payload.$": "$" },
      "ResultPath": "$.charge",
      "Catch": [ { "ErrorEquals": ["States.ALL"], "Next": "FailPayment" } ],
      "Next": "ReserveStock"
    },
    "ReserveStock": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "arn:...:ReserveStock", "Payload.$": "$" },
      "ResultPath": "$.reserve",
      "Catch": [ { "ErrorEquals": ["States.ALL"], "Next": "CompensatePayment" } ],
      "Next": "RequestShipment"
    },
    "RequestShipment": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage",
      "Parameters": {
        "QueueUrl": "https://sqs.ap-northeast-2.amazonaws.com/123/ship",
        "MessageBody.$": "$"
      },
      "Catch": [ { "ErrorEquals": ["States.ALL"], "Next": "CompensateReserve" } ],
      "End": true
    },
    "CompensateReserve": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "arn:...:ReleaseStock", "Payload.$": "$" },
      "Next": "CompensatePayment"
    },
    "CompensatePayment": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "arn:...:Refund", "Payload.$": "$" },
      "End": true
    },
    "FailPayment": { "Type": "Fail", "Cause": "PaymentFailed" }
  }
}
```

- **보상 순서**는 **역순**(결제→재고→출고)로 진행.
- 각 Task에 `Retry`/`Catch` 설정으로 **일시 오류**는 재시도, **영구 오류**는 보상으로 분기.

---

## 6) 실전 시나리오 2: **사람 승인(휴먼 인 더 루프)**

- Step Functions → 승인 요청 Lambda → **SNS/이메일/Slack** 전송  
- 승인자가 클릭/응답 → API → `SendTaskSuccess` 로 **콜백**

```json
{
  "StartAt": "AskApproval",
  "States": {
    "AskApproval": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
      "Parameters": {
        "FunctionName": "arn:...:SendApprovalLink",
        "Payload": { "taskToken.$": "$$.Task.Token", "order.$": "$.order" }
      },
      "TimeoutSeconds": 86400,
      "Next": "OnApproved"
    },
    "OnApproved": {
      "Type": "Succeed"
    }
  }
}
```

- Lambda는 승인 링크에 **토큰** 포함 URL 생성 → API Gateway/Function URL 로 회신 시 토큰 처리.

---

## 7) 실전 시나리오 3: **ETL/데이터 파이프라인**

- EventBridge(스케줄) → Glue `.sync` → Athena Partition 추가 → 결과 Slack 알림

```json
{
  "StartAt": "RunGlue",
  "States": {
    "RunGlue": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": { "JobName": "daily-etl", "Arguments": {"--dt.$": "$.dt"} },
      "Next": "AddPartition"
    },
    "AddPartition": {
      "Type": "Task",
      "Resource": "arn:aws:states:::athena:startQueryExecution.sync",
      "Parameters": {
        "QueryString.$": "States.Format('ALTER TABLE logs ADD IF NOT EXISTS PARTITION (dt=\\'{}\\')', $.dt)",
        "QueryExecutionContext": { "Database": "lake" },
        "ResultConfiguration": { "OutputLocation": "s3://athena-out/" }
      },
      "Next": "Notify"
    },
    "Notify": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": { "TopicArn": "arn:aws:sns:ap-northeast-2:123:ops", "Message.$": "$" },
      "End": true
    }
  }
}
```

---

## 8) 표준 vs 익스프레스, 비용 모델, 대략 계산

### 8.1 비교 요약

| 항목 | 표준(Standard) | 익스프레스(Express) |
|---|---|---|
| 최대 실행 | 1년 | 5분 |
| 처리량 | 중간/배치 | 매우 높음 |
| 과금 | **상태 전이 건수** | **GB-초 + 요청 건수** |
| 추적 | 상세 이력 | 요약/로그 중심 |
| 용도 | 승인/업무플로우/긴 ETL | 클릭스트림/IoT/초고TPS 오케스트레이션 |

### 8.2 비용 근사 수식(표준)

- 상태 전이 비용 단가를 \(p\) (건당)라 하고, 한 실행에 전이 수가 \(t\), 실행 수가 \(n\)이라면:

$$
\text{총비용} \approx n \cdot t \cdot p \quad (+\ \text{서비스 통합 비용, 예: Lambda, Glue 등})
$$

### 8.3 익스프레스(요청+GB-초)

- 실행 시간(초) = \(T\), 메모리(GB) = \(M\), 요청 수 \(n\)  
- GB-초 단가 \(c\), 요청 당 단가 \(r\)

$$
\text{총비용} \approx n \cdot (T \cdot M \cdot c + r)
$$

> **튜닝 포인트**: 과도한 상태 분할은 표준 비용을 올린다. **Map/Parallel의 병렬도**·**상태 수**·**통합 호출 수**를 균형화.

---

## 9) 관측성: 로깅, 메트릭, X-Ray, 입력/출력 샘플링

- **CloudWatch Logs**: 실행별 입력/출력 로깅 레벨(ALL/ERROR) 및 샘플링 비율 설정
- **메트릭**: `ExecutionsStarted/Failed/Succeeded/Throttled/TimedOut`, `ExecutionTime`, 서비스 통합별 지표
- **X-Ray**: Lambda/Step Functions 간 요청 트레이싱(콜드스타트, 다운스트림 지연 가시화)
- **DLQ/보상 루트**: 실패 경로에 SNS/SQS 를 붙여 **사건 알림·재처리 훅** 제공

---

## 10) 테스트 전략: Unit/Local/Simulate

- **Local**: ASL JSON을 **sam local/ASF Local**(시뮬레이터)로 구동(단, 제한적)
- **단위테스트**: **Lambda/Glue 쪽 로직**을 별도 테스트 → 상태 기계는 **통합테스트** 위주
- **Stage 분리**: `dev`/`stg`/`prod` 별 **파라미터 스토어/환경 변수** 분리
- **모의 성공/실패**: Choice/Retry/Catch 경로를 **강제 주입**해 시나리오별 회귀

---

## 11) IaC: CDK / SAM / CloudFormation

### 11.1 AWS CDK (TypeScript) 예시

```ts
import * as cdk from "aws-cdk-lib";
import * as sfn from "aws-cdk-lib/aws-stepfunctions";
import * as tasks from "aws-cdk-lib/aws-stepfunctions-tasks";
import * as lambda from "aws-cdk-lib/aws-lambda";

export class OrderFlowStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const chargeFn = new lambda.Function(this, "ChargeFn", {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: "index.handler",
      code: lambda.Code.fromInline(`
        exports.handler = async (e) => ({ ok: true, txnId: "TX-" + Date.now() });
      `),
      timeout: cdk.Duration.seconds(10)
    });

    const charge = new tasks.LambdaInvoke(this, "Charge", {
      lambdaFunction: chargeFn,
      payloadResponseOnly: true,
    }).addRetry({ maxAttempts: 6, interval: cdk.Duration.seconds(2), backoffRate: 2 })
      .addCatch(new sfn.Fail(this, "FailPayment", { cause: "PaymentFailed" }));

    const reserve = new tasks.LambdaInvoke(this, "ReserveStock", {
      lambdaFunction: chargeFn, // demo 재사용
      payloadResponseOnly: true
    });

    const success = new sfn.Succeed(this, "Done");

    const compensate = new tasks.LambdaInvoke(this, "Refund", {
      lambdaFunction: chargeFn
    });

    const flow = new sfn.StateMachine(this, "OrderSaga", {
      definition: charge.next(reserve.addCatch(compensate).next(success)),
      timeout: cdk.Duration.minutes(5)
    });

    new cdk.CfnOutput(this, "StateMachineArn", { value: flow.stateMachineArn });
  }
}
```

### 11.2 SAM(템플릿)로 정의 일부

```yaml
Resources:
  ChargeFn:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: nodejs20.x
      Handler: index.handler
      InlineCode: |
        exports.handler = async (e) => ({ ok: true });
  StateMachine:
    Type: AWS::Serverless::StateMachine
    Properties:
      DefinitionUri: statemachine.asl.json
      Policies:
        - LambdaInvokePolicy:
            FunctionName: !Ref ChargeFn
```

---

## 12) 보안/거버넌스

- **IAM 최소 권한**: 상태 기계가 호출하는 리소스만 허용 (정확한 ARN 스코프)
- **입출력 민감정보 마스킹**: CloudWatch Logs 로깅 레벨/필드 마스킹
- **서비스 쿼터**: 동시 실행/상태 전이 TPS → 필요한 경우 사전 상향 요청
- **Idempotency**: 외부 시스템 호출 시 **중복 안전 키**(주문ID/리퀘스트ID)로 **멱등성** 확보

---

## 13) 모범 패턴 카탈로그

1. **Job-run `.sync`**: Batch/Glue/ECS/SageMaker 학습/변환 완료까지 기다림  
2. **콜백 토큰**: 외부/사람 승인까지 대기(Wait for Callback)  
3. **SAGA 보상**: 분산 트랜잭션 대체(정방향 성공/역방향 취소)  
4. **분산 Map**: 대량 S3 오브젝트/레코드 병렬 처리  
5. **장애 분기**: Catch → SNS/SQS 알림/재처리 큐  
6. **표준↔익스프레스 혼합**: 장시간 플로우는 표준, 고TPS 소단위는 익스프레스로 **조합**  
7. **EventBridge 트리거**: 스케줄/비즈 이벤트로 상태기계 구동

---

## 14) 확장 예제: **Kinesis → Step Functions(익스프레스) → Lambda → Firehose/S3**

- Kinesis 레코드 배치 → Express State Machine(Map with concurrency) → 변환 → Firehose PutRecordBatch
- 수식상 **GB-초**가 저렴, 대량 이벤트 오케스트레이션에 적합

```json
{
  "StartAt": "ForEachRecord",
  "States": {
    "ForEachRecord": {
      "Type": "Map",
      "ItemsPath": "$.Records",
      "MaxConcurrency": 256,
      "Iterator": {
        "StartAt": "Transform",
        "States": {
          "Transform": {
            "Type": "Task",
            "Resource": "arn:aws:states:::lambda:invoke",
            "Parameters": { "FunctionName": "arn:...:Transform", "Payload.$": "$" },
            "End": true
          }
        }
      },
      "End": true
    }
  }
}
```

---

## 15) 실무 체크리스트(요약)

- 입력/출력 경로: `InputPath/Parameters/ResultPath/OutputPath` **정확히** 설계해 **불필요 페이로드 축소**
- Retry/Catch: 외부 의존성 에러만 선택적으로 재시도(5xx/429), **영구 실패**는 빠른 Catch
- 병렬도: Map/Distributed Map **MaxConcurrency** 안전 설정(다운스트림 쿼터 고려)
- 비용: 표준=상태전이, 익스프레스=GB-초/요청 … **수식 기반 사전 산정**
- 보안: IAM 최소권한, PII 마스킹, 서비스 쿼터 상향/알림
- 관측: Logs(샘플링), 메트릭 알람, X-Ray, 실패시 DLQ

---

## 부록 A) 주문 처리 예제 ― 전체 ASL 샘플

```json
{
  "Comment": "주문 처리 + 보상 + 알림",
  "StartAt": "Validate",
  "States": {
    "Validate": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "arn:...:ValidateOrder", "Payload.$": "$" },
      "ResultPath": "$.validate",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "NotifyInvalid" }],
      "Next": "Charge"
    },
    "Charge": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "arn:...:Charge", "Payload.$": "$" },
      "ResultPath": "$.charge",
      "Retry": [{ "ErrorEquals": ["Lambda.TooManyRequestsException"], "IntervalSeconds": 2, "BackoffRate": 2, "MaxAttempts": 5 }],
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "NotifyPaymentFail" }],
      "Next": "Reserve"
    },
    "Reserve": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "inventory",
        "Key": { "sku": { "S.$": "$.order.sku" } },
        "UpdateExpression": "SET reserved = reserved + :one",
        "ExpressionAttributeValues": { ":one": { "N": "1" } },
        "ReturnValues": "ALL_NEW"
      },
      "ResultPath": "$.reserve",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "CompensatePayment" }],
      "Next": "ShipRequest"
    },
    "ShipRequest": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage",
      "Parameters": {
        "QueueUrl": "https://sqs.ap-northeast-2.amazonaws.com/123/ship",
        "MessageBody.$": "States.JsonToString($)"
      },
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "CompensateReserve" }],
      "Next": "NotifySuccess"
    },
    "CompensateReserve": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "arn:...:ReleaseStock", "Payload.$": "$" },
      "Next": "CompensatePayment"
    },
    "CompensatePayment": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "arn:...:Refund", "Payload.$": "$" },
      "Next": "NotifyCompensated"
    },
    "NotifyInvalid": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": { "TopicArn": "arn:aws:sns:ap-northeast-2:123:ops", "Message": "Invalid order." },
      "End": true
    },
    "NotifyPaymentFail": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": { "TopicArn": "arn:aws:sns:ap-northeast-2:123:ops", "Message": "Payment failed." },
      "End": true
    },
    "NotifyCompensated": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": { "TopicArn": "arn:aws:sns:ap-northeast-2:123:ops", "Message": "Compensation complete." },
      "End": true
    },
    "NotifySuccess": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": { "TopicArn": "arn:aws:sns:ap-northeast-2:123:ops", "Message": "Order succeeded." },
      "End": true
    }
  }
}
```

---

## 부록 B) Lambda 핸들러(예시: Node.js)

```js
// ValidateOrder
exports.handler = async (event) => {
  if (!event.order || !event.order.id) {
    throw new Error("InvalidOrder");
  }
  return { ok: true, id: event.order.id };
};

// Charge
exports.handler = async (event) => {
  // 멱등성 키: event.order.id 를 결제 게이트웨이 요청 키로 사용
  return { ok: true, txnId: "TX-" + Date.now() };
};

// ReleaseStock / Refund 도 유사한 형태로 구현
```

---

## 부록 C) 비용 튜닝 체크

- **표준**: 상태 수(특히 Map 내부 상태) 최소화, 공통 변환은 `Pass`/`Parameters`로 합치기
- **익스프레스**: 실행 시간/메모리 최소화, 불필요한 대기/외부 호출 제거
- **다운스트림 쿼터**: Lambda 동시성, DynamoDB WCU/RCU, SQS TPS 를 Map 병렬도에 맞춰 **역산**

---

## 결론

Step Functions는 **“코드로 분산 시스템을 직접 짜지 않고도”** 안정적인 워크플로우를 만드는 **선언적 오케스트레이터**다.  
본 가이드의 **ASL 문법/경로/재시도/동적 Map/토큰 콜백/.sync/보상 트랜잭션** 패턴과 **관측·보안·비용** 체크리스트를 기반으로,  
**주문·ETL·ML·실시간 처리**까지 **운영 가능한** 서버리스 파이프라인을 빠르고 안전하게 설계하자.