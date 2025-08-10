---
layout: post
title: AWS - CloudWatch
date: 2025-07-23 22:20:23 +0900
category: AWS
---
# AWS CloudWatch: 모니터링 및 로깅

AWS CloudWatch는 **AWS 리소스와 애플리케이션의 모니터링, 로그 수집 및 분석, 알림** 기능을 제공하는 통합 서비스입니다. 클라우드 인프라가 확장되고 복잡해짐에 따라, 시스템 상태를 실시간으로 파악하고 문제 발생 시 즉각적인 대응을 가능하게 해주는 핵심 도구입니다.

---

## 1. CloudWatch의 주요 기능

| 기능 | 설명 |
|------|------|
| **모니터링(Metrics)** | EC2, RDS, Lambda, ELB 등 다양한 AWS 리소스의 상태와 성능 지표 수집 |
| **로그(Log)** | 애플리케이션 로그, 시스템 로그 등을 수집 및 저장 |
| **지표 대시보드(Dashboard)** | 수집된 지표를 시각화하여 실시간 상태 파악 |
| **경보(Alert)** | 설정한 조건에 따라 SNS, Lambda 등을 통해 알림 전송 |
| **이벤트(Event)** | 리소스 변경, 스케줄링 등 이벤트 기반 자동화 가능 |
| **인사이트(Insights)** | 로그 및 메트릭에 대한 고급 분석 (예: CloudWatch Logs Insights) |

---

## 2. CloudWatch Metrics (지표)

CloudWatch는 AWS 리소스 사용량, 성능 관련 데이터를 **1분 간격**으로 수집합니다. 대부분의 AWS 서비스는 기본적으로 CloudWatch에 메트릭을 제공합니다.

### ✅ 기본 제공 메트릭 예시

- EC2 인스턴스
  - CPUUtilization
  - NetworkIn / NetworkOut
  - DiskReadOps / DiskWriteOps
- RDS
  - CPUUtilization
  - FreeStorageSpace
  - DatabaseConnections
- Lambda
  - Invocations
  - Duration
  - Errors
  - Throttles

### ✅ 사용자 정의(Custom) 메트릭

CloudWatch Agent나 SDK를 이용해 애플리케이션에서 직접 메트릭을 보낼 수 있습니다.

```bash
aws cloudwatch put-metric-data \
  --metric-name ActiveUsers \
  --namespace MyApp \
  --unit Count \
  --value 150
```

---

## 3. CloudWatch Logs

CloudWatch Logs는 **로그 파일을 수집, 저장, 분석, 시각화**할 수 있는 기능을 제공합니다.

### ✅ 로그 수집 대상

- EC2 시스템 로그 (cloud-init, syslog 등)
- Lambda 로그 (자동으로 CloudWatch Logs로 출력됨)
- ECS / Fargate 컨테이너 로그
- VPC Flow Logs (네트워크 트래픽 로그)
- 애플리케이션 로그 (Spring, Node.js 등)

### ✅ 로그 그룹 / 로그 스트림

- **Log Group**: 애플리케이션 단위 또는 기능 단위로 로그를 그룹핑
- **Log Stream**: 시간순, 인스턴스별로 생성되는 세부 로그 스트림

### ✅ CloudWatch Agent

EC2의 시스템 로그, 메모리/디스크 메트릭 등을 수집하려면 CloudWatch Agent 설치 필요

```bash
sudo yum install amazon-cloudwatch-agent
```

CloudWatch Agent는 `/opt/aws/amazon-cloudwatch-agent/bin/config.json`으로 설정

---

## 4. CloudWatch Alarm (경보)

특정 지표가 조건을 만족할 경우 자동으로 알림을 발생시킵니다.

### ✅ 알람 구성 요소

- **Metric**: 감시할 지표 (예: CPUUtilization)
- **Threshold**: 임계값 (예: 80% 이상)
- **Evaluation Period**: 평가 주기 (예: 5분 동안)
- **Action**: 알림 전송, Auto Scaling, Lambda 실행 등

### ✅ 예제: EC2 CPU 경보

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name HighCPU \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:ap-northeast-2:111122223333:MyAlarmTopic \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0
```

### ✅ SNS 연동

CloudWatch Alarm은 **SNS(Simple Notification Service)**와 연동되어 이메일, SMS, Lambda 호출 등이 가능함

---

## 5. CloudWatch Dashboard

여러 지표를 시각화하여 한눈에 볼 수 있는 대시보드 구성

### ✅ JSON 기반 설정

대시보드는 JSON 구조로 정의 가능하며, 수동 생성 또는 IaC (CloudFormation, Terraform)으로 관리 가능

```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 6, "height": 6,
      "properties": {
        "metrics": [
          [ "AWS/EC2", "CPUUtilization", "InstanceId", "i-1234567890abcdef0" ]
        ],
        "period": 300,
        "stat": "Average",
        "region": "ap-northeast-2",
        "title": "EC2 CPU 사용률"
      }
    }
  ]
}
```

---

## 6. CloudWatch Logs Insights

대량의 로그를 SQL 유사 쿼리로 분석하는 기능

### ✅ 예제 쿼리

```sql
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

---

## 7. CloudWatch Events (EventBridge)

CloudWatch Events는 리소스 상태 변경이나 주기적 이벤트를 기반으로 자동화를 구현합니다. 현재는 **Amazon EventBridge**로 이름이 바뀌었습니다.

### ✅ 사용 예

- 매일 정해진 시간에 Lambda 실행
- 특정 EC2 인스턴스 상태 변화 감지
- ECS 태스크 시작 시 알림 전송

---

## 8. CloudWatch Agent 설정 예시

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "cpu": {
        "measurement": [
          "cpu_usage_idle", "cpu_usage_iowait"
        ],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          "used_percent"
        ],
        "metrics_collection_interval": 60
      }
    }
  }
}
```

설정 후에는 다음 명령어로 시작:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a start \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/agent-config.json \
  -s
```

---

## 9. 과금 모델

| 항목 | 과금 기준 |
|------|-----------|
| **Metrics** | 기본 메트릭은 무료 (EC2 등), 사용자 정의 메트릭은 GB 단위 |
| **Logs** | 저장 용량 + 수집 횟수 기준 |
| **Alarm** | 생성 개수 및 평가 주기 |
| **Dashboard** | 3개까지 무료 |
| **Logs Insights** | 쿼리 처리된 데이터 양 기준 |

---

## 10. 베스트 프랙티스

- 필요한 지표만 수집하여 비용 절감
- 알람은 SNS, Lambda와 연계하여 자동 대응
- 사용자 정의 메트릭은 네임스페이스로 구분
- 로그는 보존 기간 설정 (예: 30일 후 삭제)
- 로그 분석은 Logs Insights 또는 Athena 연계

---

## 마무리

CloudWatch는 단순한 모니터링을 넘어서 **자동화, 비용 최적화, 보안 운영**까지 가능한 종합적인 클라우드 운영 도구입니다. 모든 AWS 리소스와 통합되며, DevOps, SRE, 보안 운영팀 모두에게 필수적인 구성 요소입니다.
