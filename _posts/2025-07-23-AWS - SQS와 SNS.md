---
layout: post
title: AWS - SQS와 SNS
date: 2025-07-23 23:20:23 +0900
category: AWS
---
# AWS 메시징 시스템: SQS와 SNS

AWS는 분산 시스템 간의 통신을 위해 다양한 메시징 서비스를 제공합니다. 그 중 대표적인 서비스가 **SQS (Simple Queue Service)**와 **SNS (Simple Notification Service)**입니다.

이 두 서비스는 비동기 메시징을 가능하게 하며, 마이크로서비스 아키텍처, 이벤트 기반 시스템, 느슨한 결합 구조를 설계할 때 핵심적으로 사용됩니다.

---

## 🔷 SQS (Simple Queue Service)

### ✅ SQS란?

SQS는 **메시지를 큐에 저장하고, 순서에 따라 비동기적으로 처리하는 서비스**입니다. 메시지를 생산하는 쪽(Producer)과 소비하는 쪽(Consumer)을 분리할 수 있어 시스템 간 결합도를 낮출 수 있습니다.

### ✅ 주요 특징

| 특징 | 설명 |
|------|------|
| **완전관리형** | 인프라 운영 없이 큐 생성, 메시지 전송/수신 가능 |
| **높은 내구성** | 메시지는 다중 가용 영역(AZ)에 저장 |
| **확장성** | 초당 수천만 개 메시지 처리 |
| **보안** | IAM 정책, KMS를 이용한 암호화 |
| **가용성** | 메시지는 기본적으로 최대 4일간 저장 (최대 14일까지 연장 가능) |

---

### ✅ SQS 종류

| 종류 | 설명 |
|------|------|
| **표준(Standard)** | 기본 유형, 무제한 처리량, 메시지 중복 가능성 존재, 순서 보장 없음 |
| **FIFO (First-In-First-Out)** | 순서 보장, 중복 방지(ID 중복 체크), 초당 제한된 처리량 |

---

### ✅ 구성 요소

- **Producer**: 메시지를 큐에 전송하는 애플리케이션
- **Queue**: 메시지를 일시적으로 저장하는 공간
- **Consumer**: 큐에서 메시지를 꺼내어 처리하는 애플리케이션
- **Visibility Timeout**: 메시지를 가져온 후 삭제 전까지의 유예 시간 (다른 Consumer가 해당 메시지를 처리하지 않도록 보호)

---

### ✅ 메시지 흐름 예시

1. Producer가 SQS Queue에 메시지 전송
2. Consumer가 메시지를 Poll 방식으로 가져감
3. 처리 완료 시 메시지를 삭제
4. 처리 실패 시 Visibility Timeout 이후 재시도 가능

---

### ✅ SQS 예제 (AWS CLI)

```bash
# 1. 큐 생성
aws sqs create-queue --queue-name myQueue

# 2. 메시지 전송
aws sqs send-message --queue-url <QUEUE_URL> --message-body "Hello, World!"

# 3. 메시지 수신
aws sqs receive-message --queue-url <QUEUE_URL>

# 4. 메시지 삭제
aws sqs delete-message --queue-url <QUEUE_URL> --receipt-handle <RECEIPT_HANDLE>
```

---

### ✅ SQS 사용 예시

- 주문 처리 시스템 (주문 요청 큐)
- 비동기 이메일 전송
- 로그 처리 파이프라인
- 마이크로서비스 간 데이터 전달

---

## 🔷 SNS (Simple Notification Service)

### ✅ SNS란?

SNS는 **게시/구독(Pub/Sub) 방식**의 **알림 서비스**입니다. 하나의 메시지를 여러 구독자에게 동시에 전달할 수 있습니다.

---

### ✅ 주요 특징

| 특징 | 설명 |
|------|------|
| **다중 구독자 지원** | 이메일, SMS, Lambda, SQS, HTTP 등 다양한 구독 방식 |
| **실시간 푸시** | 이벤트 발생 시 즉시 전달 |
| **유연한 연동** | Lambda, SQS, EC2 등과 쉽게 연동 가능 |
| **낮은 지연 시간** | 거의 실시간으로 메시지 전파 |

---

### ✅ 구성 요소

- **Topic**: 메시지를 발행하는 대상
- **Publisher**: 메시지를 전송하는 주체
- **Subscriber**: 메시지를 수신하는 주체 (다양한 엔드포인트 가능)

---

### ✅ SNS 메시지 흐름

1. Publisher가 Topic에 메시지 게시
2. Topic은 모든 구독자에게 메시지를 전달
3. 각 Subscriber는 지정된 방식(SQS, Lambda 등)으로 메시지 수신

---

### ✅ SNS 예제 (AWS CLI)

```bash
# 1. 토픽 생성
aws sns create-topic --name myTopic

# 2. 구독자 등록 (예: 이메일)
aws sns subscribe --topic-arn <TOPIC_ARN> --protocol email --notification-endpoint you@example.com

# 3. 메시지 게시
aws sns publish --topic-arn <TOPIC_ARN> --message "Hello Subscribers!"
```

---

### ✅ SNS 구독 유형

| 프로토콜 | 설명 |
|----------|------|
| Email | 이메일 주소로 알림 전송 |
| SMS | 핸드폰 문자 메시지 |
| HTTP/HTTPS | 웹훅 형태로 POST 요청 전송 |
| Lambda | 지정된 Lambda 함수 실행 |
| SQS | SQS 큐에 메시지 전달 (Fan-Out 구조 가능) |

---

## 🔁 SQS + SNS 연동 (Fan-Out 패턴)

SNS에서 SQS를 구독하게 하면, 하나의 메시지를 여러 SQS 큐로 전달할 수 있습니다.

📌 예시: 주문 접수 → SNS → [주문 처리 큐], [이메일 알림 큐], [로깅 큐] 등

---

## 🔐 보안

- IAM 정책으로 주체별 액세스 제어
- 메시지 암호화 (SSE, KMS)
- 액세스 제한 (VPC 엔드포인트, IP 제한)

---

## 💸 비용

| 항목 | SQS | SNS |
|------|-----|-----|
| 전송 요청 | 요청 수 기준 과금 | 전송 건당 과금 |
| 저장 메시지 | 저장 기간 기반 과금 | 없음 |
| 무료 사용량 | 월 100만 건 무료 | 월 100만 건 무료 |

---

## ✅ SQS vs SNS 요약

| 항목 | SQS | SNS |
|------|-----|-----|
| 구조 | Poll 방식 큐 | Push 방식 토픽 |
| 메시지 수신자 | 단일 또는 다수의 Consumer | 다수의 Subscriber |
| 순서 보장 | FIFO 큐만 가능 | 불가능 |
| 메시지 저장 | 메시지를 큐에 저장 | 저장하지 않음 |
| 용도 | 백엔드 처리용 | 이벤트 브로드캐스트 |

---

## 🧠 실전 예시

- **SNS → Lambda**: 이미지 업로드 시 썸네일 생성 자동화
- **SNS → SQS (Fan-Out)**: 하나의 이벤트로 여러 시스템 동시 알림
- **SQS → Lambda Polling**: 비동기 주문 처리

---

## ✅ 마무리

SQS와 SNS는 AWS에서 가장 핵심적인 메시징 도구입니다. 이들을 적절히 활용하면 서비스 간 결합도를 낮추고, 이벤트 기반의 확장 가능한 아키텍처를 구현할 수 있습니다.

- 비동기 처리 → SQS
- 다중 대상 알림 → SNS
- 복합 구조 → SNS + SQS (Fan-Out)

앞으로 Lambda, EventBridge 등과 함께 사용하면 더욱 강력한 분산 이벤트 기반 시스템을 구축할 수 있습니다.