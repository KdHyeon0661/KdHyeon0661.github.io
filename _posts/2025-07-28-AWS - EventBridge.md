---
layout: post
title: AWS - EventBridge
date: 2025-07-28 14:20:23 +0900
category: AWS
---
# 🌉 AWS EventBridge란 무엇인가?

Amazon EventBridge는 **AWS 서비스, 사용자 애플리케이션, 외부 SaaS 애플리케이션 간에 이벤트 기반 아키텍처를 구성**할 수 있는 **서버리스 이벤트 버스(event bus)** 서비스입니다. 다양한 이벤트 소스를 수신하고 이를 이벤트 대상으로 라우팅함으로써, 마이크로서비스 간의 연결, 자동화된 워크플로우 구성, 분산 시스템 설계를 간단하게 만들어줍니다.

---

## 📌 핵심 개념 정리

### 1. 이벤트(Event)
- 시스템 내에서 발생하는 **상태 변화나 활동**을 의미함.
- 예: S3에 파일 업로드, EC2 인스턴스 시작, 주문 생성 등

이벤트 예시 (JSON 형태):
```json
{
  "source": "aws.ec2",
  "detail-type": "EC2 Instance State-change Notification",
  "detail": {
    "instance-id": "i-0123456789abcdef0",
    "state": "running"
  }
}
```

---

### 2. 이벤트 버스(Event Bus)
- 이벤트를 수신하고 처리하는 **이벤트의 통로**
- 기본적으로 세 가지 유형:
  - `default`: AWS 서비스 이벤트를 수신하는 기본 버스
  - `Partner Event Bus`: 외부 SaaS 파트너 이벤트 (예: Zendesk, Datadog)
  - `Custom Event Bus`: 사용자가 직접 생성해 커스텀 애플리케이션 이벤트를 보낼 수 있음

---

### 3. 이벤트 패턴(Event Pattern)
- 어떤 이벤트를 **필터링하여 수신**할지를 정의하는 조건
- 특정 소스, 상태 변화, 서비스에 대해 **정교한 조건** 설정 가능

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["stopped"]
  }
}
```

---

### 4. 대상(Target)
- 수신한 이벤트를 **전달할 대상**
- 예: Lambda, SQS, SNS, Step Functions, EC2, CodePipeline, Kinesis 등

---

## 📐 EventBridge의 구조

```text
+----------------+        +-----------------+        +------------------+
| Event Producer | -----> | EventBridge Bus | -----> | Event Target(s)  |
+----------------+        +-----------------+        +------------------+
                              ▲         │
                              │         ▼
                      +---------------+--------------+
                      | Event Rule (패턴, 필터 등)   |
                      +-----------------------------+
```

---

## 🛠️ 사용 사례

### 1. AWS 서비스 이벤트 감지 및 처리
- 예: EC2 인스턴스가 종료되었을 때 자동으로 알람 발송

### 2. 서버리스 워크플로우 구성
- 다양한 서비스 이벤트들을 Lambda, Step Functions, SNS/SQS 등으로 연결

### 3. 애플리케이션 통합
- 마이크로서비스 간의 decoupling 및 이벤트 기반 메시징

### 4. SaaS 통합
- Zendesk, Datadog, Auth0 같은 SaaS 제품과 AWS 리소스를 직접 연결

---

## ⚙️ EventBridge 규칙 생성 예시 (AWS CLI)

```bash
aws events put-rule \
    --name "EC2StoppedRule" \
    --event-pattern '{
      "source": ["aws.ec2"],
      "detail-type": ["EC2 Instance State-change Notification"],
      "detail": { "state": ["stopped"] }
    }' \
    --description "EC2 중지 이벤트 감지"
```

이후 대상 추가:
```bash
aws events put-targets \
    --rule "EC2StoppedRule" \
    --targets '[
      {
        "Id": "SendToLambda",
        "Arn": "arn:aws:lambda:ap-northeast-2:123456789012:function:handleEC2Stopped"
      }
    ]'
```

---

## 💡 EventBridge vs CloudWatch Events

| 항목                    | CloudWatch Events | EventBridge         |
|------------------------|-------------------|---------------------|
| 기능 범위               | 제한적            | 확장됨              |
| SaaS 통합              | 지원 안 함        | 지원                |
| 커스텀 이벤트           | 제한적            | 매우 유연           |
| JSON 스키마 레지스트리 | 없음              | **있음**            |
| 필터링 능력            | 기본              | **정교한 패턴 필터**|

> ※ EventBridge는 CloudWatch Events의 발전된 버전으로, CloudWatch Events는 EventBridge로 통합됨

---

## 📦 스키마 레지스트리 (Schema Registry)

- EventBridge는 **이벤트 형식을 등록 및 문서화**할 수 있는 스키마 레지스트리를 제공
- JSON 기반 이벤트 구조를 자동으로 탐지하고, 코드 생성 (Java, Python, TypeScript 등) 지원

예:
```bash
aws schemas list-registries
aws schemas list-schemas --registry-name default
```

---

## 💸 요금 구조

| 항목               | 비용 |
|--------------------|------|
| 이벤트 수신         | $1.00 / 1,000,000 이벤트 |
| 이벤트 아카이빙      | 저장량 기준 청구 (GB/월) |
| 리플레이 기능 사용   | 사용량 기준 청구        |

> ※ AWS 프리 티어는 월 100,000건의 이벤트를 무료로 제공

---

## 🧪 리플레이 및 아카이브 기능

- **리플레이(Replay)**: 과거에 발생한 이벤트를 다시 재생할 수 있음
- **아카이브(Archive)**: 이벤트를 저장하고 나중에 분석 및 리플레이에 활용

---

## 🔐 보안과 권한

- 이벤트 버스에 이벤트를 전송하려면 **PutEvents 권한**이 필요
- 대상 리소스에는 **EventBridge가 실행할 수 있는 IAM 역할** 부여 필요

---

## ✅ 마무리

Amazon EventBridge는 복잡한 분산 시스템을 이벤트 기반으로 연결할 수 있는 강력한 도구입니다. 서버리스 기반으로 유지보수 없이 확장성 높은 아키텍처를 구성할 수 있으며, 다른 AWS 서비스 또는 외부 애플리케이션과 통합할 때 매우 유용합니다.

---

## 📎 참고 링크

- [EventBridge 공식 문서](https://docs.aws.amazon.com/eventbridge/latest/userguide/what-is-amazon-eventbridge.html)
- [EventBridge Pricing](https://aws.amazon.com/eventbridge/pricing/)