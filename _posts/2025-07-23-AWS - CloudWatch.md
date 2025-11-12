---
layout: post
title: AWS - CloudWatch
date: 2025-07-23 22:20:23 +0900
category: AWS
---
# AWS CloudWatch: 모니터링 · 로깅 · 알림

## 0. 한 장 요약

- **CloudWatch Metrics**: 표준/상세 메트릭, **고해상도(1초)**, 커스텀 메트릭, **메트릭 수학**과 **이상탐지(Anomaly Detection)**, **합성 알람(Composite Alarms)**.  
- **CloudWatch Logs**: 로그 그룹/스트림, **Logs Insights** 분석, **메트릭 필터**(로그→메트릭), **서브스크립션 필터**(Lambda/Firehose/SIEM 연동), 보존 정책.  
- **알람 & 자동화**: **SNS/Lambda/Systems Manager**와 연동, **Auto Scaling/EC2 복구** 액션, **EventBridge**로 규칙 기반 자동화.  
- **대시보드**: JSON 정의/IaC로 버전관리, 크로스-어카운트/리전 뷰.  
- **에이전트/컨테이너/서버리스**: CloudWatch Agent/OTel, Container Insights, Lambda 기본 로그/메트릭, Synthetics/RUM(프런트엔드)까지 End-to-End 가시성.  
- **비용/보안**: 필요한 것만 수집, 압축/보존기간/샘플링, KMS 암호화, 접근제어, 런북.

---

## 1. CloudWatch의 주요 기능 (보강 요약)

| 기능 | 핵심 포인트 | 실무 팁 |
|---|---|---|
| **Metrics** | 1분(기본)~1초(고해상도) 수집, 차원(Dimension) 기반 | 상세 모니터링(EC2 1분)·커스텀 메트릭 네임스페이스 분리 |
| **Logs** | 로그 그룹/스트림, 보존일, 지연 수집 | **메트릭 필터**로 SLO/ErrorRate 파생 |
| **Dashboards** | JSON 정의, 위젯 단위 구성 | IaC로 PR 리뷰/승인 파이프라인 |
| **Alarms** | 단일/합성, 이상탐지 밴드 | **OK→ALARM→INSUFFICIENT_DATA** 상태 해석 |
| **Events(EventBridge)** | 상태변화·스케줄 트리거 | 재시도/Dead-letter Queue 설계 |
| **Insights** | 로그/컨테이너/네트워크 분석 | 쿼리 템플릿화·저장·태그 공유 |

---

## 2. CloudWatch Metrics — 표준·커스텀·고해상도

### 2.1 메트릭 모델 핵심
- **네임스페이스**(예: `AWS/EC2`, `MyApp`), **메트릭 이름**, **차원**(Dimensions: `InstanceId`, `FunctionName` 등), **통계**(Average/Sum/pNN), **주기(Period)**.
- 고해상도 커스텀 메트릭은 **1초 Period** 가능(비용 유의).

### 2.2 커스텀 메트릭 전송(기본/고해상도)
```bash
# 기본(60s) 커스텀 메트릭
aws cloudwatch put-metric-data \
  --namespace MyApp \
  --metric-name ActiveUsers \
  --value 150 \
  --unit Count \
  --dimensions Service=web,Stage=prod

# 고해상도(1초) 커스텀 메트릭
aws cloudwatch put-metric-data \
  --namespace MyApp \
  --metric-name CheckoutLatency \
  --value 237 \
  --unit Milliseconds \
  --storage-resolution 1 \
  --dimensions Service=checkout,Stage=prod
```

### 2.3 메트릭 수학 & 백분위수
- 여러 메트릭을 결합하는 **메트릭 수학**으로 SLI/SLO 계산:
$$
\text{ErrorRate}(t)=\frac{\text{5xxSum}(t)}{\text{ReqSum}(t)}\times100(\%)
$$
- **p90/p95/p99** 지연시간: 서비스 레벨 목표(sLO)와 직접 연결.

### 2.4 이상탐지(Anomaly Detection)
- 과거 패턴을 바탕으로 **예측 밴드** 형성 → **밴드 이탈 시 알람**.
- 부하 주기가 있는 API에 유용(주간/일중 사이클).

---

## 3. CloudWatch Logs — 수집·분석·전파

### 3.1 로그 그룹/보존/암호화
```bash
# 로그 그룹 생성 + 보존 30일 + KMS 암호화
aws logs create-log-group --log-group-name /myapp/prod
aws logs put-retention-policy --log-group-name /myapp/prod --retention-in-days 30
aws logs associate-kms-key --log-group-name /myapp/prod --kms-key-id arn:aws:kms:ap-northeast-2:123:key/abcd
```

### 3.2 에이전트(EC2/온프레)
- CloudWatch Agent로 **OS 메트릭 + 파일 로그** 동시 수집.
```bash
sudo yum install -y amazon-cloudwatch-agent
# 설정 배치 후 시작
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a start -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/agent-config.json -s
```
**예시 설정(보강)**:
```json
{
  "agent": { "metrics_collection_interval": 60, "run_as_user": "root" },
  "metrics": {
    "append_dimensions": { "InstanceId":"${aws:InstanceId}", "AutoScalingGroupName":"${aws:AutoScalingGroupName}" },
    "metrics_collected": {
      "cpu": { "resources": ["*"], "measurement": ["usage_idle","usage_iowait","usage_system"], "totalcpu": true },
      "disk": { "measurement": ["used_percent"], "resources": ["*"] },
      "mem": { "measurement": ["mem_used_percent"] }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          { "file_path": "/var/log/messages", "log_group_name": "/os/messages", "log_stream_name": "{instance_id}" },
          { "file_path": "/var/log/myapp/app.log", "log_group_name": "/myapp/prod", "log_stream_name": "{instance_id}" }
        ]
      }
    }
  }
}
```

### 3.3 Logs Insights — 빠른 분석
```sql
-- 최근 15분 5xx 비율 Top 엔드포인트 (ALB 로그 가정)
fields @timestamp, elb_status_code, target_status_code, request_url
| filter elb_status_code like /^5\d\d$/
| stats count() as errors by request_url
| sort errors desc
| limit 20
```

```sql
-- Lambda 에러율 추이 (기본 로그)
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() as errorCount by bin(5m)
```

```sql
-- VPC Flow Logs: 특정 서브넷 지연/드랍 탐지
fields @timestamp, srcAddr, dstAddr, action, bytes, dstPort
| filter action = "REJECT"
| stats count() as rejects, sum(bytes) as bytes by dstPort, bin(5m)
| sort rejects desc
```

### 3.4 메트릭 필터(로그→메트릭 파생)
```bash
aws logs put-metric-filter \
  --log-group-name /myapp/prod \
  --filter-name ErrorCount \
  --filter-pattern '"ERROR"' \
  --metric-transformations \
    metricName=AppErrorCount,metricNamespace=MyApp,metricValue=1,defaultValue=0
```
→ **알람**과 대시보드에서 **에러율**을 직접 감시 가능.

### 3.5 서브스크립션 필터(실시간 전파)
- 로그를 **Lambda**(정규화/PII 마스킹), **Kinesis/Firehose**(S3/Redshift/SIEM)로 스트리밍.
```bash
aws logs put-subscription-filter \
  --log-group-name /myapp/prod \
  --filter-name ship-to-firehose \
  --filter-pattern "" \
  --destination-arn arn:aws:firehose:ap-northeast-2:123:deliverystream/cwl-raw
```

---

## 4. 알람(Alarm) — 단일/합성·이상탐지·액션

### 4.1 단일 알람(예: EC2 CPU)
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu-i-abc \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --alarm-actions arn:aws:sns:ap-northeast-2:111122223333:MyAlarmTopic
```

### 4.2 이상탐지 알람(지연시간 밴드)
```bash
aws cloudwatch put-anomaly-detector \
  --namespace MyApp --metric-name CheckoutLatency \
  --stat Mean \
  --dimensions Name=Service,Value=checkout Name=Stage,Value=prod

aws cloudwatch put-metric-alarm \
  --alarm-name checkout-latency-anomaly \
  --comparison-operator GreaterThanUpperThreshold \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --metrics '[{"Id":"m1","MetricStat":{"Metric":{"Namespace":"MyApp","MetricName":"CheckoutLatency","Dimensions":[{"Name":"Service","Value":"checkout"},{"Name":"Stage","Value":"prod"}]},"Period":60,"Stat":"Average"}},{"Id":"ad1","Expression":"ANOMALY_DETECTION_BAND(m1, 2)","Label":"anomaly-band","ReturnData":true}]' \
  --threshold-metric-id ad1 \
  --alarm-actions arn:aws:sns:ap-northeast-2:111122223333:MyAlarmTopic
```

### 4.3 합성(Composite) 알람
- 여러 알람을 **AND/OR**로 묶어 **노이즈 감소**.
```bash
aws cloudwatch put-composite-alarm \
  --alarm-name web-critical \
  --alarm-rule 'ALARM("high-5xx") AND ALARM("high-latency")' \
  --actions-enabled \
  --alarm-actions arn:aws:sns:ap-northeast-2:111122223333:OnCallCritical
```

### 4.4 자동 조치 예
- **EC2 복구(Recover)**, **Auto Scaling 정책 트리거**, **Lambda 런북 실행**, **Systems Manager Automation 문서** 호출.

---

## 5. 대시보드(Dashboard) — JSON · IaC

### 5.1 단일 위젯 예
```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 12, "height": 6,
      "properties": {
        "metrics": [
          [ "AWS/EC2", "CPUUtilization", "InstanceId", "i-1234567890abcdef0", { "stat": "Average" } ],
          [ ".", "NetworkIn", ".", ".", { "yAxis": "right" } ]
        ],
        "period": 300,
        "region": "ap-northeast-2",
        "title": "EC2 CPU vs NetworkIn"
      }
    }
  ]
}
```

### 5.2 대시보드 생성
```bash
aws cloudwatch put-dashboard \
  --dashboard-name ops-main \
  --dashboard-body file://dashboard.json
```

> 팁: **크로스-어카운트/리전** 위젯을 한 보드에 모아 NOC 뷰 구성.

---

## 6. EventBridge(구 CloudWatch Events) — 이벤트 자동화

### 6.1 스케줄러(크론/레이트)
```bash
aws events put-rule \
  --name nightly-maintenance \
  --schedule-expression "cron(0 16 * * ? *)"  # UTC 16:00
aws events put-targets \
  --rule nightly-maintenance \
  --targets "Id"="t1","Arn"="arn:aws:lambda:ap-northeast-2:123:function:nightly"
```

### 6.2 상태 변화 트리거(예: RDS 이벤트)
```bash
aws events put-rule \
  --name rds-failover \
  --event-pattern '{"source":["aws.rds"],"detail-type":["RDS DB Instance Event"]}'
```

---

## 7. 서버리스/컨테이너/프런트엔드

### 7.1 Lambda
- 기본 메트릭: `Invocations/Duration/Errors/Throttles/IteratorAge`.  
- 구조화 로그(EMF) → **메트릭 자동 파생**(고급).

```python
# Python: 구조화 로그 → EMF
import json, time
def lambda_handler(event, context):
    print(json.dumps({
        "_aws": {"Timestamp": int(time.time()*1000),
                 "CloudWatchMetrics":[{"Namespace":"MyApp","Dimensions":[["Function"]],"Metrics":[{"Name":"Processed","Unit":"Count"}]}]},
        "Function":"checkout",
        "Processed":1
    }))
    return {"statusCode":200}
```

### 7.2 ECS/EKS (Container Insights)
- 노드/파드/서비스 레벨 메트릭/로그 자동 수집(에이전트/OTel).  
- **애드온**: Fluent Bit로 로그 라우팅.

### 7.3 Synthetics & RUM(선택)
- **Synthetics Canaries**: 브라우저/HTTP 시나리오 헬스체크, TLS/만료/상태/스크립트 검증 → 알람 연계.  
- **RUM**: 실제 유저 브라우저 지표(LCP/CLS/TBT 등) 수집.

---

## 8. 비용 모델 & 최적화

### 8.1 근사 수식
$$
\text{Monthly Cost} \approx
C_\text{metrics}+C_\text{logs}+C_\text{alarms}+C_\text{insights}
$$
- **Metrics**: 표준은 서비스 포함, **커스텀/고해상도** 유료.  
- **Logs**: 수집 요청 + 저장 GB·월 + 조회/Insights 스캔 데이터.  
- **Alarms**: 개수×평가주기, 컴포지트 상대적으로 저렴한 노이즈 감축.  
- **Insights**: 스캔 바이트당.

### 8.2 절감 체크리스트
- [x] **로그 보존일** 설정(예: 30~90일), 장기보관은 **Firehose→S3→Athena**  
- [x] **필요한 필드만** 수집(구조화/샘플링/압축)  
- [x] **커스텀 메트릭 최소화**(집계 후 전송, 1초 해상도는 정말 필요한 곳에만)  
- [x] Logs Insights 쿼리 범위/시간/필드 제한으로 스캔 비용 절감  
- [x] 대시보드/알람 **IaC 관리**로 중복/유휴 자원 정리

---

## 9. 보안/거버넌스

- **KMS 암호화**(로그 그룹, 대시보드 데이터는 메타 수준), **전송구간 TLS**.  
- **IAM 최소권한**: `logs:PutLogEvents`, `cloudwatch:PutMetricData` 최소 스코프.  
- **리소스 정책**: 서브스크립션 대상 제한(교차 계정 송신 시).  
- **PII/비밀**: Lambda 서브스크립션으로 **마스킹** 후 저장.

---

## 10. 런북(운영 절차) 예시

### 10.1 알람: API 5xx 급증
1. 대시보드에서 **Req/5xx/ErrorRate/p95 Latency** 동시 확인  
2. Logs Insights: 최근 5~15분, **경로/고객/버전** 차원으로 분해  
3. 배포 타임라인 매칭(릴리스/플래그 on/off)  
4. 롤백/플래그 오프 → 알람 해소 확인  
5. 포스트모템: 근본원인/테스트갭/경보 튜닝

### 10.2 인프라: EC2 CPU 90% 지속
1. 프로세스/스레드/GC/IO Wait 확인(에이전트 메트릭)  
2. 오토스케일 조건 및 ALB 타겟 상태 점검  
3. **임시 Scale-out** + 근본 쿼리/캐시/락 분석

---

## 11. 트러블슈팅 Q&A

| 증상 | 원인 | 해결 |
|---|---|---|
| 알람 플래핑 | 임계/평가주기 과민 | 히스테리시스(메트릭 수학), 합성 알람, 이상탐지 사용 |
| Logs 지연 표시 | 에이전트 버퍼링/네트워크 | 버퍼/플러시/재시도 설정, 리전 근접성 |
| 커스텀 메트릭 미표시 | 차원 미스매치/네임스페이스 오타 | 동일 차원 세트 유지, 대시보드 쿼리 확인 |
| Insights 비용 높음 | 광범위 쿼리 | 시간/필드 제한, 저장 쿼리 템플릿, Athena로 이관 |

---

## 12. 실전 시나리오 모음

### 12.1 백엔드 API SLO(99% 가용, p95<300ms)
- **메트릭**: `Req`, `5xx`, `Latency(p95)`  
- **수식**:
$$
\text{SLO\_Availability} = 1 - \frac{\text{5xx}}{\text{Req}}
$$
- **알람**: 5분 평균 ErrorRate>1% AND p95>300ms → **합성 알람**  
- **대시보드**: 시간대별 트래픽/에러/지연, 배포 마커(주석 위젯)

### 12.2 배치 장애 탐지
- **커스텀 메트릭**: `BatchSuccess/Failure` 이벤트 기반 Sum  
- **알람**: 스케줄 창 내 Success==0 이면 경보, EventBridge 재시도.

### 12.3 데이터 파이프라인 품질
- **메트릭 필터**: “record_parsed”·“record_dropped” 카운트  
- **알람**: `dropped/parsed > 0.05` over 10m

---

## 13. IaC 스니펫(요약)

### 13.1 CloudFormation: 대시보드 리소스
```yaml
Resources:
  OpsDashboard:
    Type: AWS::CloudWatch::Dashboard
    Properties:
      DashboardName: ops-main
      DashboardBody: !Sub |
        {
          "widgets":[
            {"type":"metric","x":0,"y":0,"width":12,"height":6,
             "properties":{"title":"API Error Rate","region":"${AWS::Region}",
               "metrics":[
                 ["MyApp","5xx","Service","api","Stage","prod",{"stat":"Sum"}],
                 [".","Req",".",".",".",".",{"stat":"Sum"}],
                 [{"expression":"(m1/m2)*100","label":"Error%","id":"e1"}]
               ],
               "view":"timeSeries","period":60}}
          ]
        }
```

### 13.2 Terraform: 알람 리소스
```hcl
resource "aws_cloudwatch_metric_alarm" "error_rate" {
  alarm_name          = "api-error-rate"
  comparison_operator = "GreaterThanThreshold"
  threshold           = 1
  evaluation_periods  = 2
  datapoints_to_alarm = 2
  metric_query {
    id          = "m1"
    return_data = false
    metric {
      namespace  = "MyApp"
      metric_name= "5xx"
      period     = 60
      stat       = "Sum"
      dimensions = { Service="api", Stage="prod" }
    }
  }
  metric_query {
    id          = "m2"
    return_data = false
    metric {
      namespace  = "MyApp"
      metric_name= "Req"
      period     = 60
      stat       = "Sum"
      dimensions = { Service="api", Stage="prod" }
    }
  }
  metric_query {
    id          = "e1"
    expression  = "(m1/m2)*100"
    label       = "Error%"
    return_data = true
  }
  alarm_actions = [aws_sns_topic.oncall.arn]
}
```

---

## 14. 마무리 요약(초안 + 보강)

| 주제 | 핵심 |
|---|---|
| 메트릭 | 표준+커스텀, 고해상도, **수학/이상탐지**로 스마트 알람 |
| 로그 | 보존/암호화/서브스크립션, **메트릭 필터**로 SLI 파생 |
| 알람 | 단일/합성, 자동조치(SNS/Lambda/ASG/SSM) |
| 대시보드 | JSON·IaC, 크로스-어카운트/리전 한 눈에 |
| 이벤트 | EventBridge로 스케줄/상태변화 자동화 |
| 서버리스/컨테이너 | Lambda 기본 메트릭, Container Insights, OTel |
| 비용/보안 | 보존·압축·샘플링, KMS/IAM 최소권한 |
| 운영 | 런북/포스트모템, PR로 모니터링 코드화(Observability as Code) |

이 글의 예제들을 **당신의 리전/계정/서비스 이름**에 맞춰 치환하면, **CloudWatch 중심의 관측성 플랫폼**을 빠르게 표준화할 수 있다.