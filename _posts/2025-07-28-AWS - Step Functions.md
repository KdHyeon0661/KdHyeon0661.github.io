---
layout: post
title: AWS - Step Functions
date: 2025-07-28 15:20:23 +0900
category: AWS
---
# 🧩 AWS Step Functions: 서버리스 워크플로우 오케스트레이션

---

## ✅ Step Functions란?

**AWS Step Functions**는 여러 AWS 서비스들을 **순차적/병렬적/조건적**으로 연결하여 **워크플로우(workflow)**로 구성할 수 있도록 해주는 서비스다.

즉, 하나의 작업이 끝나고 다음 작업을 자동으로 실행하거나, 오류 발생 시 예외 처리를 하는 등의 **복잡한 로직 제어**를 **시각적이면서 선언적으로** 구성할 수 있게 해준다.

### 🔧 주요 특징

| 기능 | 설명 |
|------|------|
| 서버리스 | 인프라 관리 불필요, AWS가 자동 스케일링 |
| 상태 기계 | 작업 흐름을 상태(state) 기반으로 구성 |
| 시각적 디자인 | 콘솔에서 시각적 플로우 차트로 워크플로우 구성 가능 |
| 자동 재시도 | 실패한 작업에 대한 재시도 로직 내장 |
| AWS 서비스 통합 | Lambda, ECS, SQS, DynamoDB, SNS, SageMaker 등과 통합 |
| 고가용성 | AWS가 내장 제공하는 안정적인 서비스 |
| 상태 추적 | 각 단계별 실행 기록 보존 |

---

## 🧠 Step Functions의 구조

Step Functions는 워크플로우를 **상태 기계(State Machine)**라고 부른다.

### 📘 정의 파일(JSON)

Step Functions는 **Amazon States Language (ASL)**이라는 JSON 기반 DSL로 상태를 정의한다.

```json
{
  "Comment": "Example state machine",
  "StartAt": "Step1",
  "States": {
    "Step1": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-northeast-2:123456789012:function:MyLambdaFunction",
      "Next": "Step2"
    },
    "Step2": {
      "Type": "Succeed"
    }
  }
}
```

---

## 🧱 상태(State) 타입 종류

| 상태 타입 | 설명 |
|-----------|------|
| `Task` | Lambda, ECS 등 실제 작업 수행 |
| `Choice` | 조건 분기 |
| `Wait` | 일정 시간 대기 |
| `Parallel` | 병렬 실행 |
| `Map` | 배열 항목에 반복 적용 |
| `Pass` | 입력 데이터를 그대로 넘김 |
| `Succeed` | 성공 종료 |
| `Fail` | 실패 종료 |

---

## 🔗 통합 가능한 AWS 서비스 예시

| 서비스 | 사용 예시 |
|--------|-----------|
| Lambda | 서버리스 함수 실행 |
| DynamoDB | 데이터 조회/저장 |
| ECS/Fargate | 컨테이너 작업 실행 |
| SNS/SQS | 메시지 발행/큐 대기 |
| Glue | ETL 파이프라인 실행 |
| SageMaker | ML 모델 학습/예측 |
| API Gateway | 외부 트리거 호출 지원 |
| EventBridge | 이벤트 기반 실행 트리거 |

---

## 🧪 예시 시나리오

### 📦 상품 주문 처리 예시

```text
[주문 수신] → [결제 처리] → [재고 확인]
                  ↓ 실패 시
            [오류 처리] → [알림 전송]
```

```json
{
  "StartAt": "ReceiveOrder",
  "States": {
    "ReceiveOrder": {
      "Type": "Task",
      "Resource": "Lambda:ReceiveOrder",
      "Next": "ProcessPayment"
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "Lambda:Payment",
      "Next": "CheckInventory",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "HandleError"
      }]
    },
    "CheckInventory": {
      "Type": "Task",
      "Resource": "Lambda:InventoryCheck",
      "End": true
    },
    "HandleError": {
      "Type": "Task",
      "Resource": "Lambda:SendAlert",
      "End": true
    }
  }
}
```

---

## 📊 상태 추적 및 디버깅

- AWS 콘솔에서 **각 단계별 실행 내역**, **입출력 JSON**을 **시각적으로 확인 가능**
- 실패한 작업은 이유까지 함께 기록되어 **디버깅에 용이**

---

## 💰 비용

- 실행 횟수 기반 요금
  - 첫 4000건은 매월 무료
  - 이후 **$0.025/1000건** (표준 워크플로우 기준)

- 워크플로우가 복잡하고 단계가 많을수록 요금 증가

---

## 🔄 표준 vs 익스프레스 워크플로우

| 항목 | 표준(Standard) | 익스프레스(Express) |
|------|----------------|----------------------|
| 실행 시간 | 최대 1년 | 최대 5분 |
| 호출 빈도 | 낮음 (Batch) | 높음 (Realtime) |
| 사용 목적 | 신중한 상태 추적이 필요한 복잡한 프로세스 | 대량의 빠른 이벤트 처리 |
| 비용 | 상대적으로 높음 | 매우 저렴함 |
| 상태 추적 | 예 | 간단한 로그만 지원 |

---

## 💡 사용 사례

- 전자상거래 주문 처리 플로우
- 서버리스 데이터 처리 파이프라인
- 배치 데이터 처리 자동화
- ML 모델 추론 파이프라인
- 백오피스 승인 절차 자동화

---

## 🧩 정리

AWS Step Functions는 여러 AWS 서비스들을 연결해주는 **워크플로우 조립 툴**이다.  
복잡한 비즈니스 로직을 **코드 없이 시각적으로 선언**할 수 있고, 자동화/재시도/에러처리 등을 내장 제공한다.

**실시간 시스템**, **백엔드 자동화**, **ETL 파이프라인**, **ML 파이프라인**, **DevOps 자동화** 등 다양한 환경에서 폭넓게 쓰인다.
