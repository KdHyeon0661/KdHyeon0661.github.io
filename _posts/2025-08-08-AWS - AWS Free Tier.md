---
layout: post
title: AWS - AWS Free Tier
date: 2025-08-08 15:20:23 +0900
category: AWS
---
# AWS Free Tier 이해와 **실전 활용** 끝장 가이드 (2025 업데이트 반영)

## 빠른 요약

- AWS Free Tier는 크게 **(A) Always Free**, **(B) 신규 고객용 Free Tier(신 모델: 크레딧 + 월별 크레딧)**, **(C) Legacy Free Tier(예: 12개월 무료)** 세 축으로 구성.
- **항상 무료(Always Free)** 예: Lambda 100만 요청/월(지역/정책에 따라 상이 가능), DynamoDB 읽기/쓰기 용량 및 스토리지 일정량 등(세부 한도는 각 서비스 가격/Free Tier 페이지에서 확인).
- **신규 고객**은 Free Tier 크레딧으로 여러 서비스를 폭넓게 체험할 수 있으며, 서비스별 월 크레딧이 **최대 6개월**까지 제공(계정 상태/지역별로 상이). 상세는 Free Tier FAQ/Overview 참조.
- **Legacy Free Tier**는 기존처럼 **12개월/Always Free/Trial** 조합으로 제공되며, 계정 생성일 기준 적용.

---

## Free Tier 구성의 **두 갈래**: 신 모델 vs. 레거시

### 새로운 Free Tier (2025.07.15 도입)

- **핵심**: 신규 고객에게 **시작 크레딧** + **월별 크레딧(최대 6개월)** + **Always Free** 제공.
- **의미**: 과거처럼 “EC2 750시간” 등 **고정 리소스 한도**만으로 체험하던 방식에서 벗어나, **크레딧을 유연하게 배분**해 다양한 서비스를 맛볼 수 있게 됨.
- **무엇이 달라졌나**: 기간/금액/대상 서비스가 **지역과 시점에 따라 다를 수 있음** → **본인 콘솔의 Free Tier**와 **공식 FAQ/Overview**를 반드시 확인.

### Legacy Free Tier (이전 계정)

- **구성**:
  - **12개월 무료**: (예) EC2 t2.micro/t3.micro 급 **월 750시간**, S3 **5GB**, RDS **월 750시간** 등 **고정 한도**(세부 항목과 대상 인스턴스 패밀리는 지역/시점에 따라 상이 가능).
  - **Always Free**: Lambda(예: 100만 요청/월), DynamoDB 스토리지/용량 등 **기간 무제한**의 기본 한도.
  - **Trial**: 특정 서비스의 **한시적 체험**.
- **주의**: 서비스별 정확 한도/대상 인스턴스는 수시로 변동 가능성 → **공식 Free Tier/가격 페이지**에서 확인.

---

## 언제 무엇을 쓸까? (학습/PoC/프로토타입 시나리오 7선)

1) **정적 웹 호스팅**
- **S3** 정적 호스팅 + **CloudFront** 캐시 + **Route 53** 도메인.
- Free Tier 범위: S3 스토리지(Always/Legacy 기준 상이) 및 CloudFront 전송 일부. 상세는 공식 페이지 확인.

2) **서버리스 API**
- **API Gateway + Lambda + DynamoDB** 조합.
- Lambda는 Always Free에 **요청 수**가 포함되어 대개 작은 PoC에 충분(정확 한도는 공식 페이지/콘솔에서 확인).

3) **간단한 백엔드**
- **EC2 t3.micro** 등으로 Flask/Express 서버 1대.
- Legacy Free Tier 계정이라면 “월 750시간” 모델로 한 달 내 1대 상시 가동 가능. 신규 Free Tier는 **크레딧 소진 패턴**을 주시.

4) **관측/로깅 실습**
- **CloudWatch** 메트릭/로그를 작은 볼륨으로 실습. Always Free/Free Tier 크레딧 범위 확인.

5) **데이터 파이프라인 미니 PoC**
- **Kinesis Firehose → S3 → Athena**로 로그 분석 체험(크레딧/Always Free 조합).

6) **이미지 리사이징/큐 기반 처리**
- **S3 업로드 → SQS → Lambda**로 비동기 처리. Lambda/요청 수/실행 시간 내에서 실습.

7) **IaC 학습**
- **CloudFormation/CDK**로 리소스를 **만들고 곧바로 삭제**하는 루틴으로 안전하게 반복 학습.

---

## **비용·한도**를 수식으로 빠르게 감 잡기

Free Tier를 넘기지 않으려면, **월간 사용량**을 대략 계산하고 모니터링하자.

- 예: API 호출 기반 서비스에서의 과금 예상
  $$ \text{예상 청구액} \approx \max(0,\ \text{월간요청수} - F)\times r $$
  - \(F\): 월간 Free Tier 요청 한도(예: Lambda 1,000,000 요청/월 – 정책 확인)
  - \(r\): 초과 **요청당 단가** (해당 서비스 가격표 확인)
- 예: EC2 실행 시간(레거시 750시간 가정)의 잔여 시간
  $$ \text{초과시간} = \max(0,\ \text{실행 인스턴스 수} \times \text{해당월 총시간} - 750) $$

> **경고**: 실제 단가는 **지역별/시점별로 상이**. 수식은 **개념적 가이드**일 뿐이며, 실제 청구는 **공식 가격표**와 **콘솔 사용량**을 반드시 확인.

---

## **모니터링 & 경보** — 초과 과금 방지 필수 루틴

### 콘솔에서 확인하기

- **Billing & Cost Management → Free Tier** 대시보드에서 **서비스별 사용량/남은 한도**를 확인. (계정 유형/시점에 따라 항목이 다름)

### AWS Budgets로 경보 만들기 (월별 비용 알림)

```bash
# 예: 월 1 USD 초과 시 이메일 알림(샘플)

aws budgets create-budget --account-id 123456789012 --budget '{
  "BudgetName": "FreeTierGuard",
  "BudgetLimit": {"Amount": "1", "Unit": "USD"},
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST",
  "CostFilters": {},
  "CostTypes": {"IncludeCredit": true, "IncludeDiscount": true, "UseAmortized": false},
  "TimePeriod": {"Start": "2025-11-01T00:00:00Z", "End": "2026-11-01T00:00:00Z"}
}' --notifications-with-subscribers '[
  {
    "Notification": {
      "NotificationType": "FORECASTED",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 1.0,
      "ThresholdType": "ABSOLUTE_VALUE"
    },
    "Subscribers": [
      {"SubscriptionType": "EMAIL", "Address": "you@example.com"}
    ]
  }
]'
```

### Cost Explorer로 빠른 사용량/비용 조회(CLI)

```bash
aws ce get-cost-and-usage \
  --time-period Start=2025-11-01,End=2025-12-01 \
  --granularity DAILY \
  --metrics "UnblendedCost" "UsageQuantity" \
  --group-by Type=DIMENSION,Key=SERVICE
```

---

## **자동 종료/정리** 스크립트 — 실수 방지

### 유휴 EC2 자동 종료(태그 기반)

```bash
# 태그 "AutoStop=true" 인스턴스 중 상태가 running이고 지난 6시간 내 CPU 평균 < 2% → 종료
# (CloudWatch Metric/Period/Statistics는 실제 운영 환경에 맞게 수정)

INSTANCE_IDS=$(aws ec2 describe-instances \
  --filters "Name=tag:AutoStop,Values=true" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].InstanceId" --output text)

for id in $INSTANCE_IDS; do
  AVG=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/EC2 --metric-name CPUUtilization \
    --dimensions Name=InstanceId,Value=$id \
    --start-time $(date -u -d '6 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
    --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 3600 --statistics Average \
    --query "Datapoints[].Average" --output text | awk '{sum+=$1} END{if(NR>0)print sum/NR;else print 0}')
  if [ "$(printf '%.0f' "$(echo "$AVG < 2" | bc -l)")" -eq 1 ]; then
    aws ec2 stop-instances --instance-ids $id
  fi
done
```

### S3 정리(Lifecycle) — 실습 로그 7일 보관 후 삭제

```yaml
# bucket-lifecycle.yaml (S3 Lifecycle Rule)

Rules:
  - ID: "delete-lab-logs-7days"
    Status: Enabled
    Filter: { Prefix: "lab-logs/" }
    Expiration:
      Days: 7
```

> IaC로 버킷을 만들 때 즉시 넣어두면 **불필요한 저장료**를 막기 쉽다.

---

## **실전 예제**: Always Free 중심 서버리스 API (Lambda + API Gateway + DynamoDB)

### 리소스 아키텍처

```
[Client] → [API Gateway] → [Lambda] ↔ [DynamoDB(Table: items)]
```

- 저/중소량 트래픽의 학습/PoC 용도로 **Always Free 범위**에서 동작하도록 설계.

### CDK(TypeScript) 템플릿

```ts
import * as cdk from 'aws-cdk-lib';
import { Runtime, Code, Function as LambdaFn } from 'aws-cdk-lib/aws-lambda';
import { LambdaRestApi } from 'aws-cdk-lib/aws-apigateway';
import { Table, BillingMode, AttributeType } from 'aws-cdk-lib/aws-dynamodb';

export class FreeTierServerlessStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const table = new Table(this, 'ItemsTable', {
      partitionKey: { name: 'id', type: AttributeType.STRING },
      billingMode: BillingMode.PAY_PER_REQUEST // Free Tier/Always Free에 유리
    });

    const fn = new LambdaFn(this, 'ApiFn', {
      runtime: Runtime.NODEJS_18_X,
      handler: 'index.handler',
      code: Code.fromInline(`
        exports.handler = async (event) => {
          return { statusCode: 200, body: JSON.stringify({ok:true, path:event.path}) };
        };
      `),
      environment: { TABLE_NAME: table.tableName }
    });

    table.grantReadWriteData(fn);

    new LambdaRestApi(this, 'HttpApi', {
      handler: fn,
      proxy: true
    });
  }
}
```

**배포**

```bash
npm i -g aws-cdk
cdk init app --language typescript
# 위 스택 파일을 lib/에 배치 후

npm i aws-cdk-lib constructs
cdk synth
cdk deploy
```

> **팁**: PoC 종료 즉시 `cdk destroy` 로 비용 흔적을 지울 것.

---

## **레거시 Free Tier** 가정: EC2 1대 + S3 정적 사이트 실습

### CloudFormation 스니펫 (EC2 + 보안그룹)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: "Legacy Free Tier style: EC2 micro + SG"
Resources:
  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: allow http
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
  WebInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: ami-xxxxxxxxxxxx # 지역 맞는 최신 AMI로 교체
      SecurityGroupIds:
        - !Ref WebSG
      Tags:
        - Key: AutoStop
          Value: "true"
```

> 레거시 12개월/750시간 모델을 **가정**한 예시. 실제 가능 인스턴스 타입/한도는 **계정/지역/시점**에 따라 다를 수 있다. 항상 공식 페이지와 콘솔 확인.

---

## **보안/운영 모범 사례 12가지**

1) **리소스 태그 표준화**: `Project`, `Owner`, `ExpireAt`(ISO8601) 등으로 정리.
2) **자동 만료**: Lambda/Step Functions로 태그 `ExpireAt` 이 지난 자원 정리.
3) **비공개 기본값**: S3 퍼블릭 액세스 차단, SG 최소 포트 오픈.
4) **자격증명 노출 방지**: IAM Role 사용, 액세스 키는 분리/회전.
5) **최소 권한**: 기능별로 IAM 권한 최소화.
6) **KMS 연계**: PII/민감 데이터는 SSE-KMS/Envelope Encryption.
7) **로그/메트릭**: CloudWatch Alarms 로 **비용 이상치** 감시.
8) **IaC**: 같은 환경을 만들고 지우기 쉽게.
9) **리전 집중**: 여러 리전에 흩어지면 **Free Tier 합산**으로 초과 위험.
10) **데이터 전송 고려**: AZ/리전/인터넷 구간에 따른 전송 비용 체크.
11) **무료 한도 내 테스트**: 부하 테스트는 크레딧/Free Tier 범위를 벗어나기 쉬움.
12) **정기 점검**: 매주 Free Tier/비용 리포트를 메일로 받는 습관.

---

## **자주 하는 실수 & 방지 체크리스트**

- [ ] EC2 두 대를 24x7 가동(레거시 750시간 가정 시 초과).
- [ ] 오브젝트 버저닝/로그 누적으로 S3 스토리지 초과.
- [ ] Lambda/Step Functions **무한 재시도**로 요청 폭증.
- [ ] CloudWatch Logs **보존기간 무제한**으로 비용 증가.
- [ ] RDS를 큰 인스턴스로 생성(Free Tier 비대상).
- [ ] 글로벌 서비스(예: CloudFront) 전송량 과다.
- [ ] 여러 리전에 동일 서비스 생성 → Free Tier **합산 초과**.
- [ ] 종료를 잊은 **Elastic IP/ALB/NAT Gateway**.
- [ ] “Always Free”라고 해서 **무한정 무료**로 오해(한도 존재).

---

## **운영 자동화**: GitHub Actions로 실습 스택 생성→테스트→삭제

```yaml
name: free-tier-lab
on: [workflow_dispatch]
jobs:
  lab:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "18" }
      - run: npm i -g aws-cdk
      - run: npm ci
      - run: cdk synth
      - run: cdk deploy --require-approval never
      - run: npm test
      - run: cdk destroy --force
```

> **철칙**: “만들었으면 반드시 지운다.”

---

## **FAQ**

**Q1. 내 계정이 신규 Free Tier인지, Legacy인지 어떻게 알 수 있나?**
A. **Free Tier 페이지/FAQ**에서 계정 상태를 확인. **신규 고객**에게는 크레딧 기반 Free Tier가 안내되며, **2025-07-15 이전** 생성 계정은 Legacy 모델이 적용된다.

**Q2. Always Free는 진짜 영구 무료인가?**
A. 특정 **항목/한도**에 한해 영구 제공. 한도 초과분은 과금. 상세 한도는 서비스 가격/Free Tier 문서를 확인.

**Q3. 공식 숫자(예: EC2 750시간, S3 5GB)를 그대로 믿어도 되나?**
A. **시점/지역/계정 유형에 따라 달라질 수 있음**. 본문 예시는 이해를 돕기 위한 일반적 관례이며, **반드시 콘솔/공식 페이지를 확인**해야 한다.

---

## **현장용 체크리스트** (복붙해서 쓰기)

- [ ] 콘솔 **Free Tier 대시보드** 즐겨찾기(매주 확인).
- [ ] **Budgets** 월 1~5 USD 알림 설정.
- [ ] IaC로 생성 → 테스트 → **파괴(destroy)** 루틴.
- [ ] S3 Lifecycle, CloudWatch Logs 보존기간 7~30일.
- [ ] EC2/NAT/ALB/EIP 유휴자원 제거 스크립트.
- [ ] 태그: `Project`, `Owner`, `ExpireAt`.
- [ ] 리전 1곳 집중.
- [ ] KMS/SSE로 민감데이터 보호.
- [ ] 매월 첫 주 **비용 리뷰** & 아키텍처 다이어트.

---

## 마무리

AWS Free Tier는 **단순 무료 쿠폰**이 아니라, **비용 위험을 통제**하면서도 **실전 학습/PoC**를 반복할 수 있게 만든 **안전 그물**이다.
2025년 도입된 **크레딧 기반 Free Tier** 덕분에 초기 실험의 폭은 넓어졌지만, **모니터링/자동정리/태깅/경보**가 없으면 초과 과금 위험은 여전하다.

- **핵심 요약**
  1) 본인 계정이 **신규**인지 **Legacy**인지 먼저 확인.
  2) **Always Free** 범위에 맞춘 **서버리스/저비용 아키텍처**로 시작.
  3) **Budgets/Cost Explorer**와 **정리 스크립트**로 **초과 과금 방지**.
  4) **IaC로 생성-검증-삭제**를 습관화.

---

### 참고 & 원문

- AWS Free Tier FAQ (2025): 새 크레딧 기반 모델, Legacy Free Tier 안내, 지역/계정별 차이 명시.
- AWS Free Tier Overview (2025): 프로그램 개요/구성요소/신규 고객 혜택 설명.
