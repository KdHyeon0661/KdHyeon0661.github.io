---
layout: post
title: AWS - Kinesis
date: 2025-07-28 16:20:23 +0900
category: AWS
---
# AWS Kinesis: 실시간 데이터 스트리밍 완전 정복

AWS Kinesis는 대규모 실시간 데이터 스트리밍을 수집하고 처리할 수 있는 완전관리형 서비스군입니다.  
로그, 클릭스트림, IoT, 트랜잭션 등 다양한 소스로부터 데이터를 실시간으로 수집하고 분석, 저장, 시각화할 수 있습니다.

---

## 1. Kinesis 서비스 구성 종류

| 서비스 | 설명 |
|--------|------|
| **Kinesis Data Streams (KDS)** | 고성능 실시간 데이터 수집 및 처리 (Producer → Stream → Consumers) :contentReference[oaicite:0]{index=0} |
| **Kinesis Data Firehose** | 실시간 스트리밍 데이터를 S3, Redshift, Elasticsearch 등으로 자동 전송 :contentReference[oaicite:1]{index=1} |
| **Kinesis Data Analytics** | SQL 또는 Apache Flink 기반 실시간 스트림 분석 :contentReference[oaicite:2]{index=2} |
| **Kinesis Video Streams** | 실시간 또는 저장된 영상 스트리밍 및 분석용 처리 서비스 :contentReference[oaicite:3]{index=3} |

---

## 2. Kinesis Data Streams 상세

### 주요 기능 및 사례

- **실시간 처리**: IT 로그, 클릭스트림, 금융 거래 등 초당 기가바이트 단위 데이터 수집 및 처리 :contentReference[oaicite:4]{index=4}
- **저지연**: data‑put 후 거의 1초 이내 처리 가능 :contentReference[oaicite:5]{index=5}
- **다중 소비자 구조**: 하나의 스트림 데이터를 여러 소비자로 병렬 처리 가능 :contentReference[oaicite:6]{index=6}

### 구성 요소

- **Shard (샤드)**: 처리 용량 단위, 1 MB/sec 쓰기 또는 2 MB/sec 읽기 :contentReference[oaicite:7]{index=7}
- **파티션 키**: 메시지를 특정 샤드에 분배하는 키 (순서 보장용)
- **Producer & Consumer**: SDK, KPL, KCL, Lambda 등 다양한 방식 지원 :contentReference[oaicite:8]{index=8}
- **Enhanced Fan‑Out**: 각 소비자에게 별도 2 MB/sec 읽기 경로 제공 (저지연 병렬 처리 가능) :contentReference[oaicite:9]{index=9}
- **Retention (보존 기간)**: 기본 24시간, 최대 365일까지 확장 가능 :contentReference[oaicite:10]{index=10}

---

## 3. Kinesis 가격 구조 (Data Streams 기준)

### 용량 모드 비교

| 모드 | 설명 | 비용 특성 |
|------|------|-----------|
| **Provisioned** | 샤드 수 직접 지정 | 샤드당 시간 요금 + PUT 단가, 예측 가능 :contentReference[oaicite:11]{index=11} |
| **On‑Demand** | 사용량 기반 자동 확장 | 쓰기/읽기 GB 단위 요금 + 시간당 스트림 요금 :contentReference[oaicite:12]{index=12} |

### 비용 항목 개요 (On‑Demand 예시)

- 데이터 수신(In): GB당 약 $0.08  
- 데이터 읽기(Out): GB당 약 $0.04  
- 스트림 가동 시간당 비용 추가  
- 확장 보관 및 Enhanced Fan‑Out 등 옵션 요금 적용 :contentReference[oaicite:13]{index=13}

### 요금 예시

- 1,000건/sec, 각 3 KB → 월 약 $918 소요(데이터 수신 + 읽기 + 스트림 시간 기준) :contentReference[oaicite:14]{index=14}
- Provisioned 모드 비용 분석도 참고 가능 :contentReference[oaicite:15]{index=15}

---

## 4. Kinesis의 활용 구조 (예시)

```text
Producer → Kinesis Data Streams → (Lambda / Kinesis Analytics / Firehose) → 데이터 처리 또는 저장
```

- **실시간 분석**: Analytics를 통한 SQL 기반 실시간 집계  
- **데이터 적재**: Firehose로 S3, Redshift 등 저장소 자동 적재  
- **실시간 응답**: Lambda와 연동해 즉각적인 처리 가능

---

## 5. 장단점 및 선택 팁

### 장점
- 완전관리형으로 인프라 운영 부담 최소화  
- 고성능, 확장성 높은 실시간 처리 가능  
- AWS 생태계 서비스와 깊은 통합

### 고려사항
- 샤드 과/초과 관리 필요 (Provisioned 모드)  
- 비용 구조를 정확히 이해하고 설계 필요  
- 데이터 보존 기간 확장 시 추가 요금 발생

---

## 6. 요약 정리

| 항목 | Kinesis Data Streams 요약 |
|------|----------------------------|
| 적합 대상 | 실시간 데이터 수집 및 처리 필요 시스템 |
| 구조 | 샤드 기반 스트림 + 여러 소비자 병렬 처리 |
| 용량 모드 | Provisioned vs On‑Demand 선택 가능 |
| 비용 요소 | 쓰기/읽기 GB, 샤드 시간, 옵션 요금 |
| 시나리오 | 실시간 로그 분석, 클릭스트림 처리, IoT 데이터 |