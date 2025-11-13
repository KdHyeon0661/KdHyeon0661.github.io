---
layout: post
title: AWS - Lambda
date: 2025-07-23 19:20:23 +0900
category: AWS
---
# AWS Lambda: 서버리스 함수 실행

## 0. 한 장 요약

- **서버리스** = 서버가 *없다*가 아니라, **프로비저닝/패치/스케일링을 신경쓰지 않는다**.
- **Lambda 실행 흐름**: **Init**(부트스트랩/핸들러 로딩/네트워크 준비) → **Invoke**(요청 처리) → **(빈번 시) 재사용**.
- **스케일링**: 요청/이벤트에 비례해 **자동 수평 확장**. 동시성=동시에 수행 중인 실행 수.
- **비용**: **요청 수 + (메모리×실행시간)**, 선택 시 **프로비저닝 동시성** 비용 추가.
- **트리거별 차이**: 비동기(S3/SNS/EventBridge) 재시도 정책, SQS/Kinesis **배치/부분실패 보고**, API Gateway 동기 호출 등 **행동이 다르다**.
- **베스트 프랙티스**: 최소권한 IAM, **멱등성**, 짧은 실행/빠른 I/O, **관측성(로그·지표·추적)**, **IaC로 일관 배포**, 필요 시 **Provisioned Concurrency / SnapStart(Java)** 로 콜드스타트 완화.

---

## 1. Lambda 핵심 개념

### 1.1 서버리스란?
- OS/패치/Auto Scaling/고가용성은 **AWS가 관리**. 사용자는 **핸들러 코드**와 **구성(메모리/타임아웃/역할/환경변수)** 에 집중.

### 1.2 Lambda란?
- **FaaS (Function as a Service)**. 이벤트로 **핸들러 함수**를 실행.
- 주요 특징: **이벤트 기반**, **초 단위 과금(100ms 청구 단위)**, **런타임 다수 지원(Node, Python, Java, .NET, Go 등)**, **컨테이너 이미지 기반 배포 지원**.

---

## 2. 실행 수명주기 & 콜드스타트

### 2.1 단계
1) **Init**: 런타임 부팅, 코드/Layers 로딩, `global` 스코프 초기화, 확장: VPC ENI 준비, Extension 시작.
2) **Invoke**: `handler(event, context)` 실행.
3) **재사용**: 동일 **실행 환경**이 **여러 요청** 처리(상태 보존 주의: 캐시/DB 커넥션 재사용은 장점).

### 2.2 콜드스타트 완화
- **메모리 상향**(CPU/네트워크도 비례 증가), **코드/의존성 슬림화**, **VPC 연결 최적화**(엔드포인트/보안그룹 단순화),
- **Provisioned Concurrency**(사전 예열), **SnapStart(Java)**(초기화 스냅샷), **ARM64 런타임**(비용/성능 균형).

---

## 3. 동시성·자동확장·제어

- **동시성(Concurrency)**: 동시에 실행 중인 함수 인스턴스 수. 이벤트가 몰리면 **자동 수평 확장**.
- **예약 동시성(Reserved Concurrency)**: 특정 함수의 **최대 동시성 상한/최소 보장**.
- **프로비저닝 동시성(Provisioned Concurrency)**: 미리 예열된 인스턴스 수. **예측 가능한 지연/요금 추가**.
- **계정 한도**(리전별 동시성 소프트 리밋 존재 → 필요 시 증가 요청).

---

## 4. 비용 모델 (간단 공식)

- **요청 요금** + **컴퓨팅 요금(GB-초)** + (선택) **프로비저닝 동시성**
- 컴퓨팅 비용 추정:
$$
\text{ComputeCost} \approx \left(\sum_i \text{Duration}_i\right) \times \text{Memory(GB)} \times \text{단가(GB\!-\!s)}
$$

- 메모리 증설 시 **CPU/네트워크도 함께 증가**→ 실행 시간이 줄어 **총비용이 오히려 감소**할 수 있다(튜닝 포인트).

---

## 5. 트리거·통합 개요

| 트리거 | 호출 유형 | 재시도/배치 | 부분실패 | 비고 |
|---|---|---|---|---|
| **API Gateway/ALB/Function URL** | 동기 | 없음 | 없음 | HTTP 엔드포인트 |
| **S3/SNS/EventBridge(비동기)** | 비동기 | 자동 재시도(백오프) | 없음 | 실패 시 DLQ/OnFailure Destinations |
| **SQS** | 폴링 | 배치(1~10, 최대 10분 대기), 재시도 큐 | **지원**(배치 실패 보고) | 처리량↑ 동시성↑ |
| **Kinesis/MSK/DDB Streams** | 폴링 | 샤드당 순서 보장, 체크포인트 | 제한적 | 샤드 동시성=샤드 수 |
| **CloudFront(Lambda@Edge)** | 동기 | 없음 | 없음 | 전 세계 엣지에서 실행 |

---

## 6. 핸들러/언어별 기본 예제

### 6.1 Python 핸들러(기본)
```python
# app.py
import os, json, boto3

dynamodb = boto3.resource("dynamodb")
TABLE = os.environ["TABLE_NAME"]
table = dynamodb.Table(TABLE)

def lambda_handler(event, context):
    # 멱등성의 예: 요청 ID 기반 처리 중복 방지 (간단 예시)
    req_id = event.get("requestId") or context.aws_request_id
    table.put_item(Item={"pk": f"req#{req_id}", "ts": context.get_remaining_time_in_millis()})
    return {"statusCode": 200, "body": json.dumps({"ok": True, "req": req_id})}
```

### 6.2 Node.js (Express 스타일 라우팅은 API GW/Lambda Proxy로)
```js
// index.mjs (Node 18+)
export const handler = async (event) => {
  const name = (event.queryStringParameters && event.queryStringParameters.name) || "world";
  return { statusCode: 200, body: JSON.stringify({ hello: name }) };
};
```

### 6.3 Java – SnapStart 고려 포인트
```java
// Handler.java (Java 17)
public class Handler implements RequestHandler<Map<String,Object>, Map<String,Object>> {
  private final SomeClient client = new SomeClient(); // Init 단계에서 준비 → SnapStart로 스냅샷

  @Override public Map<String,Object> handleRequest(Map<String,Object> event, Context ctx) {
    return Map.of("statusCode", 200, "body", "ok");
  }
}
```

---

## 7. 이벤트 소스(폴링) 상세: SQS / Kinesis / DDB Streams

### 7.1 SQS 소비자 (배치·부분실패)
- **배치 크기**/**가시성 타임아웃**/**동시성**을 조정.
- **부분 실패 보고(ReportBatchItemFailures)** 로 **성공 항목은 삭제**, 실패 항목만 재시도.

```python
# SQS 배치 부분실패 예 (Python)
def lambda_handler(event, _):
    failures = []
    for rec in event["Records"]:
        try:
            process(rec["body"])
        except Exception:
            failures.append({"itemIdentifier": rec["messageId"]})
    return {"batchItemFailures": failures}
```

### 7.2 Kinesis / DynamoDB Streams
- **샤드당 1 동시성**(기본), 순서 보장. 체크포인트 실패 시 같은 배치 재시도.
- **Batch Window/Size** 로 처리량/지연 최적화.

---

## 8. 오류 처리·재시도·DLQ/대상지(Asynchronous Destinations)

| 유형 | 재시도 | 실패 후 |
|---|---|---|
| 동기(API GW/ALB/URL) | 클라이언트 재시도 | 없음 |
| 비동기(S3/SNS/EventBridge) | 자동 재시도(지수 백오프) | **DLQ(SQS/SNS) or OnFailure Destinations** |
| SQS | 폴링/가시성 타임아웃 재처리 | **DLQ(레드라이브 정책)** |
| Kinesis/DDB Streams | 같은 배치 재시도 | 처리 중단(장시간 실패 시 알람 필요) |

- **멱등성**: 재시도/중복전달 대비. **요청ID + 상태저장** 또는 **업서트/조건식** 사용.

---

## 9. 네트워킹: VPC/EFS/엔드포인트

- **VPC 연결**: 사설 RDS/ElastiCache 접근 시 필요. 서브넷/보안그룹 지정.
- 최신 Lambda는 **하이퍼플레인 ENI** 최적화로 콜드스타트 영향이 과거보다 감소.
- **VPC 엔드포인트**(S3/KMS/Secrets Manager 등)로 **인터넷 없이** AWS API 접근.
- **EFS** 연결: 대용량 모델/아카이브 공유. 동시 접근·POSIX 필요 시.

---

## 10. 파일·성능·리소스

- **메모리**: 128MB~10GB. **CPU/네트워크 비례 증가**.
- **에페메럴 스토리지(`/tmp`)**: 기본 512MB, 최대 10GB까지 확장 가능. 대용량 임시 작업(Gzip, 이미지 처리)에 유용.
- **아키텍처**: `x86_64` vs `arm64`. ARM64는 비용/성능 우수한 경우 많음(테스트 권장).
- **패키징**:
  - **ZIP 배포**(빠르고 단순, 50MB 압축/250MB 압축해제 한도), **Layers** 로 의존성 공유.
  - **컨테이너 이미지**(ECR, 최대 10GB) – 머신러닝/네이티브 라이브러리 편리.

---

## 11. 보안/IAM/비밀

- **실행 역할(Execution Role)**: 최소권한으로 S3/DDB/… 접근.
- **환경변수**: 민감 값은 **KMS 암호화**.
- **Secrets Manager/SSM Parameter**: **캐싱** 라이브러리 활용(네트워크 호출 최소화).
- **정적 아웃바운드 차단 시** VPC 엔드포인트 구성.
- **코드 서명(Code Signing)**: 서명된 배포만 허용(규정 준수 환경).

---

## 12. 관측성: 로그/지표/추적

- **CloudWatch Logs**: 구조화(JSON) 로깅, 상관ID(correlation id) 포함.
- **지표**: `Invocations`, `Duration`, `Errors`, `Throttles`, `IteratorAge`(Streams), `ConcurrentExecutions`.
- **경보**: 오류율↑, 지연↑, 스로틀↑, 처리지연(IteratorAge)↑ 시 알림.
- **X-Ray**: 분산 추적. 외부 호출/다운스트림 서비스 병목 파악.
- **AWS Lambda Powertools**(Python/TypeScript/Java): 로깅/트레이싱/지표/멱등성/유효성검사 **표준화**.

---

## 13. HTTP 통합: API Gateway / ALB / Function URL

- **API Gateway(REST/HTTP)**: 인증/인가·쿼터·변환·캐시 등 API 관리 필요 시.
- **ALB**: 경량 HTTP 트리거, 경로/호스트 기반 라우팅.
- **Function URL**: 함수 전용 퍼블릭 엔드포인트(간단, 기능 제한).

### Lambda Proxy 응답 포맷(예: API GW HTTP API)
```json
{
  "statusCode": 200,
  "headers": {"content-type": "application/json"},
  "body": "{\"ok\":true}"
}
```

---

## 14. 엔드투엔드 예제 1 — 이미지 썸네일 파이프라인(S3→Lambda→S3)

### 14.1 IAM 역할 요약(최소)
```json
{
  "Version":"2012-10-17",
  "Statement":[
    {"Effect":"Allow","Action":["s3:GetObject"],"Resource":"arn:aws:s3:::origin-bucket/*"},
    {"Effect":"Allow","Action":["s3:PutObject"],"Resource":"arn:aws:s3:::thumb-bucket/*"},
    {"Effect":"Allow","Action":["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"],"Resource":"*"}
  ]
}
```

### 14.2 Lambda 코드(Python, Pillow 사용)
```python
import os, io, boto3
from PIL import Image

s3 = boto3.client("s3")
OUT_BUCKET = os.environ["OUT_BUCKET"]
SIZE = int(os.environ.get("SIZE", "256"))

def lambda_handler(event, _):
    rec = event["Records"][0]
    bucket = rec["s3"]["bucket"]["name"]
    key = rec["s3"]["object"]["key"]

    img_obj = s3.get_object(Bucket=bucket, Key=key)
    image = Image.open(img_obj["Body"])
    image.thumbnail((SIZE, SIZE))

    buf = io.BytesIO()
    image.save(buf, "JPEG", quality=85)
    buf.seek(0)

    dst_key = f"thumb/{key.rsplit('/',1)[-1]}"
    s3.put_object(Bucket=OUT_BUCKET, Key=dst_key, Body=buf, ContentType="image/jpeg")
    return {"statusCode": 200, "body": f"ok:{dst_key}"}
```

### 14.3 S3 알림 설정
- 원본 버킷 `origin-bucket` → **객체 생성(접두사 images/)** → 대상: Lambda.

---

## 15. 엔드투엔드 예제 2 — SQS 워커(배치·부분실패·멱등성)

```python
import json, boto3, os
from botocore.exceptions import ClientError

ddb = boto3.client("dynamodb")
TABLE = os.environ["TABLE"]

def process_one(msg):
    body = json.loads(msg["body"])
    order_id = body["orderId"]
    # 멱등 처리: 조건부 Put(이미 처리된 orderId면 스킵)
    ddb.put_item(
        TableName=TABLE,
        Item={"pk":{"S":f"order#{order_id}"}, "status":{"S":"processed"}},
        ConditionExpression="attribute_not_exists(pk)"
    )

def lambda_handler(event, _):
    failures = []
    for r in event["Records"]:
        try:
            process_one(r)
        except ClientError as e:
            if e.response["Error"]["Code"] == "ConditionalCheckFailedException":
                # 이미 처리됨 → 성공 취급
                continue
            failures.append({"itemIdentifier": r["messageId"]})
        except Exception:
            failures.append({"itemIdentifier": r["messageId"]})
    return {"batchItemFailures": failures}
```

- **배치 크기**: 10(권장 시작점), 가시성 타임아웃: 함수 최대 실행시간 이상.
- 실패 항목만 재시도 → **중복 처리 최소화**.

---

## 16. IaC 배포 스니펫

### 16.1 SAM 템플릿(HTTP API + Lambda + 권한)
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  Api:
    Type: AWS::Serverless::HttpApi

  Func:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: app.lambda_handler
      Runtime: python3.12
      MemorySize: 1024
      Timeout: 15
      Architectures: [arm64]
      Policies:
        - CloudWatchLogsFullAccess
      Environment:
        Variables:
          TABLE_NAME: Users
      Events:
        Route:
          Type: HttpApi
          Properties:
            ApiId: !Ref Api
            Path: /hello
            Method: GET
```

### 16.2 Serverless Framework (Node)
```yaml
service: api
provider:
  name: aws
  runtime: nodejs20.x
  region: ap-northeast-2
functions:
  hello:
    handler: index.handler
    memorySize: 1024
    timeout: 10
    events:
      - httpApi:
          path: /hello
          method: get
```

### 16.3 Terraform(컨테이너 이미지 배포)
```hcl
resource "aws_lambda_function" "img" {
  function_name = "img-worker"
  package_type  = "Image"
  image_uri     = "${aws_ecr_repository.repo.repository_url}:v1"
  role          = aws_iam_role.exec.arn
  timeout       = 30
  memory_size   = 1024
  architectures = ["arm64"]
}
```

---

## 17. 고급 기능

- **Provisioned Concurrency**: 예열된 실행환경 유지 → 지연 안정화(비용 추가).
- **SnapStart for Java**: Init 완료 상태를 스냅샷, **콜드스타트 단축**.
- **Lambda Extensions**: 모니터링/보안 에이전트 사이드카 패턴.
- **Function URL**: 간단 퍼블릭 HTTP 엔드포인트. **권한(일반/iam)** 설정 주의.
- **Step Functions**: **다중 Lambda** 오케스트레이션(재시도/분기/보상 트랜잭션).
- **Lambda@Edge / CloudFront Functions**: 엣지 위치에서 헤더 수정/인증/리다이렉트.

---

## 18. 성능·비용 최적화 체크리스트

- [ ] **의존성 슬림/트리셰이킹**, 지연 로딩(필요 시 import).
- [ ] **메모리 상향 후 실행시간 변화** 측정 → **총비용 최솟값 탐색**.
- [ ] 네트워크: **HTTP Keep-Alive/커넥션 재사용**, SDK 클라이언트 **글로벌 초기화**.
- [ ] I/O: S3 Select/압축/열지향(Parquet)로 **전송/처리 감소**.
- [ ] **ARM64** 평가.
- [ ] **프로비저닝 동시성**은 API p99 SLA 등 **명확한 근거** 있을 때만.
- [ ] **관측성 대시보드**: 오류율·p95/p99 지연·스로틀·IteratorAge·동시성.

---

## 19. 보안·규정 준수

- **최소권한 IAM**(액션/리소스 구체화).
- **비밀은 Secrets Manager/SSM** 사용 + **캐시**.
- **KMS 암호화**(환경변수/애플리케이션 데이터).
- **VPC 엔드포인트**로 사설 호출 강제, **보안그룹 최소화**.
- **Code Signing**와 **정책 시뮬레이터/Access Analyzer**로 사전 검증.
- **S3/Object Lock + 버전관리 + 수명주기** 조합(백업/감사).

---

## 20. 트러블슈팅 빠른 가이드

- **TimeOut**: 외부 API 지연? VPC/NAT/DNS? 라이브러리 콜드로드?
- **Throttle(429)**: 동시성 부족 → 예약 동시성/프로비저닝/큐잉 도입.
- **SQS 재시도 폭발**: 배치 실패 원인 로깅, **부분실패 보고** 적용, DLQ/레드라이브 설정.
- **Kinesis 정체(IteratorAge↑)**: 샤드 부족/처리시간 과다/배치 크기 재조정.
- **메모리 부족**: OOM 로그 확인, 메모리/에페메럴 스토리지 상향.

---

## 21. 엔드투엔드 예제 3 — API Gateway + Lambda + DynamoDB (서버리스 CRUD)

### 21.1 테이블
- PK: `id` (문자열)

### 21.2 Lambda 코드(Node.js)
```js
// crud.mjs
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, GetCommand, PutCommand, DeleteCommand, ScanCommand } from "@aws-sdk/lib-dynamodb";

const ddb = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const TABLE = process.env.TABLE;

export const handler = async (event) => {
  const method = event.requestContext.http.method;
  const path = event.requestContext.http.path;
  const body = event.body ? JSON.parse(event.body) : {};

  try {
    if (method === "GET" && path === "/items") {
      const res = await ddb.send(new ScanCommand({ TableName: TABLE, Limit: 20 }));
      return ok(res.Items);
    }
    if (method === "GET" && path.startsWith("/items/")) {
      const id = path.split("/").pop();
      const res = await ddb.send(new GetCommand({ TableName: TABLE, Key: { id } }));
      return ok(res.Item || {});
    }
    if (method === "POST" && path === "/items") {
      const item = { id: body.id, ...body };
      await ddb.send(new PutCommand({ TableName: TABLE, Item: item, ConditionExpression: "attribute_not_exists(id)" }));
      return created(item);
    }
    if (method === "DELETE" && path.startsWith("/items/")) {
      const id = path.split("/").pop();
      await ddb.send(new DeleteCommand({ TableName: TABLE, Key: { id } }));
      return noContent();
    }
    return notFound();
  } catch (e) { return error(e); }
};

const ok = (data) => ({ statusCode: 200, headers: {"content-type":"application/json"}, body: JSON.stringify(data) });
const created = (data) => ({ statusCode: 201, headers: {"content-type":"application/json"}, body: JSON.stringify(data) });
const noContent = () => ({ statusCode: 204, body: "" });
const notFound = () => ({ statusCode: 404, body: "not found" });
const error = (e) => ({ statusCode: 500, body: JSON.stringify({ error: e.message }) });
```

### 21.3 API GW(HTTP API) 라우팅
- `GET /items`, `GET /items/{id}`, `POST /items`, `DELETE /items/{id}` → 단일 Lambda(프록시 통합).

---

## 22. 비용 시뮬레이션(개념)

- 월 **N** 요청, 평균 실행시간 **T(ms)**, 메모리 **M(GB)**, 요청요금 단가 **C_req**, 컴퓨팅 단가 **C_gbs** 일 때:
$$
\text{MonthlyCost} \approx N \cdot C_{\text{req}} + \left(\frac{T}{1000}\cdot N \cdot M\right)\cdot C_{\text{gbs}}
$$
- 메모리 상향으로 **T 감소 효과**를 반영해 **최적점**을 탐색하라(벤치마크 필수).

---

## 23. 체크리스트 (배포 전)

- [ ] 타임아웃/메모리/에페메럴 스토리지 적정 설정
- [ ] 최소 권한 IAM + KMS/비밀관리 + 환경변수 관리
- [ ] 트리거별 **재시도/DLQ/대상지** 정책 적용
- [ ] 멱등성(요청ID/조건식/Upsert) 구현
- [ ] 관측성: 구조화 로그 + 지표 경보 + X-Ray
- [ ] IaC(SAM/Serverless/Terraform)로 재현 가능
- [ ] 필요 시 Provisioned Concurrency/SnapStart

---

## 결론

Lambda는 **이벤트 기반 마이크로 서비스/ETL/웹훅/API/배치**까지 폭넓게 적용 가능한 **서버리스 실행 엔진**이다.
핵심은 **트리거 특성에 맞춘 재시도·배치·부분실패 처리**, **멱등성**, **관측성**, **보안 최소권한**, **비용/성능 튜닝**이다.
본 가이드의 패턴과 스니펫을 **템플릿화**하여 팀 표준으로 삼으면, 빠르게 안정적이고 비용 효율적인 워크로드를 구축할 수 있다.
