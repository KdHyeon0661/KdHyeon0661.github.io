---
layout: post
title: AWS - DynamoDB
date: 2025-07-23 21:20:23 +0900
category: AWS
---
# AWS DynamoDB: NoSQL 데이터베이스 완벽 가이드

## 한 장 요약

- **핵심 철학**: DynamoDB는 **요청 패턴을 중심으로 스키마(=키 설계)**를 만든다. 정규화보다 **조회 키 설계**가 최우선.
- **기본 키**: `PK(Partition Key)`와 `SK(Sort Key)` 조합으로 **1:N 항목 집합(Item Collection)**을 만든다.
- **확장**: 자동 수평 확장, **어댑티브 캐패시티**, 테이블 크기 무제한, 초당 수천~수십만 RPS.
- **성능**: 단일 디지트 ms 레이턴시(캐시 DAX로 서브-ms).
- **유틸리티**: GSI/LSI, TTL, Streams, Global Tables, Transactions, PartiQL, 컨디셔널 라이트.
- **비용**: 저장(GB-월), 읽기/쓰기(온디맨드 또는 RCU/WCU), GSI/백업/글로벌테이블 추가 과금 → **접근 패턴 + 수명주기 + 캐시**로 최적화.

---

## DynamoDB란 무엇인가?

- **완전관리형 NoSQL 키-값/문서형 DB**. 서버 관리 없음. 자동 확장. 고가용성(AZ 분산).
- **유연한 스키마**: JSON 유사 문서. 아이템마다 다른 속성 가능.
- **서버리스 통합**: API Gateway/Lambda와 궁합 우수, Streams로 이벤트를 Lambda로 전달.

### 대표 특성 요약

| 특성 | 설명 |
|---|---|
| 서버리스 | 프로비저닝/운영 자동화 |
| 수평 확장 | 파티션 단위로 분산 저장/처리 |
| 고가용성 | 다중 AZ 자동 복제 |
| API | Get/Put/Update/Delete/Query/Scan/Batch/Transact |
| 인덱스 | GSI(글로벌), LSI(로컬) |
| 이벤트 | Streams, Lambda 트리거 |
| 보안 | VPC Endpoints, KMS 암호화, IAM 미세 권한 |
| 백업 | 온디맨드 스냅샷, Point-in-Time Recovery(PITR) |
| 글로벌 | Global Tables(리전 간 멀티마스터) |

---

## 데이터 모델: 테이블/아이템/속성/키

### 테이블과 기본 키

- **단일 키(Partition Key)**: `PK` 하나로 샤딩. `GetItem/Query` 성능 최고.
- **복합 키(Partition + Sort)**: `PK + SK`로 파티션 내 정렬/범위쿼리 지원.

```json
{
  "PK": "USER#user123",
  "SK": "PROFILE",
  "Name": "Alice",
  "Age": 30,
  "Hobbies": ["hiking", "reading"]
}
```

### 아이템 컬렉션(Item Collection)

- 동일 `PK` 아래 **서브타입**(행 역할)을 `SK`로 구분.

예시(사용자 + 주문 + 주소를 한 테이블에):

```json
{ "PK": "USER#u123", "SK": "PROFILE", "name": "Alice" }
{ "PK": "USER#u123", "SK": "ORDER#2025-01-01T10:00", "orderId": "o1", "amount": 120 }
{ "PK": "USER#u123", "SK": "ORDER#2025-02-03T15:20", "orderId": "o2", "amount": 75 }
{ "PK": "USER#u123", "SK": "ADDR#home", "city": "Seoul" }
```

- `Query PK=USER#u123 and begins_with(SK, 'ORDER#')` → 해당 유저의 주문만 정렬/페이지네이션.

### 단일 테이블 설계(Single-Table Design)

- **여러 엔티티를 한 테이블**에 담고, `PK/SK` 네이밍 규칙으로 관계/조회 경로를 구성.
- 장점: 조인 필요 없는 **O(1)/O(logN)** 접근, **인덱스 줄이기**, **원자적 트랜잭션**.

---

## 읽기/쓰기 API와 패턴

### 쓰기

- `PutItem`: 삽입/덮어쓰기(조건식 가능)
- `UpdateItem`: 부분 업데이트(원자적 증가/배열 조작)
- `DeleteItem`: 삭제(조건식 가능)

### 읽기

- `GetItem`: 단일 키 조회
- `Query`: **PK 필수**, `SK` 범위조건(`=`, `BETWEEN`, `begins_with`) 지원
- `Scan`: 테이블 전수 검색(가능하면 지양)

### 조건식(Optimistic Locking, 유니크 보장)

```json
ConditionExpression 예시:
- attribute_not_exists(PK)  // 존재 안 해야 삽입
- attribute_exists(PK)      // 존재해야 수정/삭제
- version = :expected       // 버전 일치 시 업데이트(낙관적 잠금)
```

---

## 인덱스: LSI vs GSI

| 항목 | LSI(Local Secondary Index) | GSI(Global Secondary Index) |
|---|---|---|
| 정의 시점 | 테이블 생성 시에만 | 언제든 추가/삭제 가능 |
| 키 구성 | PK 동일, **다른 SK** | **다른 PK** (+ 선택적 SK) |
| 용도 | **PK는 유지**, 정렬/필터 관점 추가 | **다른 액세스 패턴**(새 파티션 키) |
| 수 제한(기본) | 테이블당 최대 5개 | 테이블당 최대 20개 |
| 용량/비용 | 테이블 용량 공유 | **별도 용량/비용**(온디맨드/프로비저닝) |

> 원칙: **액세스 패턴을 열거 → 기본 테이블 설계 → 부족한 경로만 GSI로 보강**. LSI는 생성 후 삭제 불가라 초기에 신중.

---

## 용량 모드: 온디맨드 vs 프로비저닝

### 온디맨드(On-Demand)

- 예측 불필요, 요청량 기반 과금. 버스트/급변 트래픽에 유리.
- 시작/초기 서비스/POC 권장.

### 프로비저닝(Provisioned)

- **RCU(읽기 용량), WCU(쓰기 용량)**를 사전에 지정. Auto Scaling 가능.
- 예측 가능한 트래픽, 비용 최적화에 유리.

### RCU/WCU 계산(근사)

- **강한 읽기(Strongly Consistent Read)**: 1 RCU → **최대 4KB**/초
- **최종 일관 읽기(Eventual)**: 1 RCU → **최대 8KB**/초
- **쓰기**: 1 WCU → **최대 1KB**/초

$$
\text{RCU_{strong}}=\left\lceil \frac{\text{ItemSizeBytes}}{4096} \right\rceil \times \text{ReadsPerSec}
$$

$$
\text{RCU_{eventual}}=\left\lceil \frac{\text{ItemSizeBytes}}{8192} \right\rceil \times \text{ReadsPerSec}
$$

$$
\text{WCU}=\left\lceil \frac{\text{ItemSizeBytes}}{1024} \right\rceil \times \text{WritesPerSec}
$$

> 10KB 아이템을 초당 200회 강한 읽기:
> $$\lceil 10/4 \rceil \times 200 = 3 \times 200 = 600 \text{ RCU}$$

---

## 트랜잭션(ACID)와 일관성

- `TransactWriteItems`, `TransactGetItems`로 **원자적 다중 아이템** 처리(최대 25건).
- **비용**: 트랜잭션 오버헤드 존재(읽기/쓰기 2배 과금 기준 참고).
- **일관성**: 트랜잭션은 강한 내구성, 읽기는 강/최종 선택.

---

## DynamoDB Streams & 이벤트 설계

- **변경 스트림**(INSERT/MODIFY/REMOVE)을 **비동기 처리**(Lambda, Kinesis 소비).
- 이벤트 소싱/아웃박스 패턴/비동기 사가에 활용.

```json
{
  "eventName": "INSERT",
  "dynamodb": {
    "Keys": { "PK": {"S":"USER#u123"}, "SK": {"S":"ORDER#o1"} },
    "NewImage": { "amount": {"N":"120"} }
  }
}
```

Lambda 핸들러(파이썬):

```python
import json
def handler(event, context):
    for r in event["Records"]:
        if r["eventName"] == "INSERT":
            img = r["dynamodb"]["NewImage"]
            # 후처리 (예: 포인트 적립, 인덱스 보강 테이블 기록 등)
```

---

## 글로벌 테이블(Global Tables)

- 멀티리전 **멀티마스터** 복제(충돌은 최근 쓰기 우선 기반).
- 장점: 리전 독립 읽기/쓰기, 재해복구(RTO↓, RPO≈0).
- 주의: **결제/재고** 등 엄격한 직렬화 필요 시 **트랜잭션+업무 규칙** 보완.

---

## TTL/백업/PITR/보안

- **TTL**: 만료 시 자동 삭제(비동기). 임시/세션/로그 데이터 비용 절감.
- **백업**: 온디맨드 스냅샷, **PITR**(최대 35일 시점 복구).
- **보안**: **기본 암호화**(KMS), **IAM 조건부 접근**(예: 사용자 ID 일치), **VPC 엔드포인트**.

IAM 예시(자기 소유 데이터만 접근):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["dynamodb:GetItem","dynamodb:Query","dynamodb:PutItem"],
    "Resource": "arn:aws:dynamodb:ap-northeast-2:123456789012:table/AppTable",
    "Condition": {
      "ForAllValues:StringLike": {
        "dynamodb:LeadingKeys": ["USER#${aws:userid}"]
      }
    }
  }]
}
```

---

## & 캐시 전략

- **DAX**: DynamoDB 전용 인메모리 캐시. **서브-ms 읽기**. 쓰기 스루 방식.
- 읽기 집중/핫키 시 레이턴시 낮추고 RCU 절감.
- 코드 변경 최소: SDK DAX 클라이언트로 교체.

Node.js DAX 클라이언트 예:

```js
const AmazonDaxClient = require('amazon-dax-client');
const AWS = require('aws-sdk');

const dax = new AmazonDaxClient({ endpoints: ['my-dax-cluster:8111'], region: 'ap-northeast-2' });
const docClient = new AWS.DynamoDB.DocumentClient({ service: dax });

await docClient.get({ TableName: 'AppTable', Key: { PK: 'USER#u1', SK: 'PROFILE' } }).promise();
```

---

## 모델링: 단일 테이블 실제 예제

### 요구사항(액세스 패턴)

- 사용자 프로필 조회: `GET /users/{id}`
- 사용자 주문 목록(최신순): `GET /users/{id}/orders?limit=20&cursor=...`
- 주문 상세 조회: `GET /orders/{orderId}`
- 주문을 상태별로 조회: `GET /orders?status=PAID&from=...`
- 이메일로 사용자 찾기: `GET /users?email=...`

### 테이블 스키마(한 테이블)

- **PK**: 다양한 접두어로 파티션 분리
- **SK**: 정렬 및 타입 구분

아이템 예시:

```json
{ "PK":"USER#u123", "SK":"PROFILE", "email":"a@example.com", "name":"Alice" }
{ "PK":"USER#u123", "SK":"ORDER#2025-02-03T15:20#o2", "orderId":"o2", "status":"PAID", "amount":75 }
{ "PK":"ORDER#o2", "SK":"ORDER", "userId":"u123", "status":"PAID", "amount":75 }
{ "PK":"EMAIL#a@example.com", "SK":"USER#u123" }
{ "PK":"STATUS#PAID", "SK":"2025-02-03T15:20#o2" }
```

- 사용자 주문 목록: `PK=USER#u123 AND begins_with(SK,'ORDER#')`
- 주문 상세: `PK=ORDER#o2, SK=ORDER` (혹은 GSI: `orderId`가 PK)
- 이메일 역인덱스(쓰기 시 동기화): `PK=EMAIL#...`

### 보조 인덱스(GSI)

- GSI1: `GPK=orderId`, `GSK=CONST` → **주문 ID 단건 조회**
- GSI2: `GPK=status`, `GSK=eventTime#orderId` → **상태별 페이지네이션**

GSI 투영 속성: 쿼리에 필요한 필드만 **Projected**(ALL/KEYS_ONLY/INCLUDE).

---

## SDK/CLI/PartiQL 예제

### CRUD + 조건식

```js
const AWS = require('aws-sdk');
AWS.config.update({ region: 'ap-northeast-2' });
const dc = new AWS.DynamoDB.DocumentClient();
const Table = 'AppTable';

// Create with uniqueness on PK/SK
await dc.put({
  TableName: Table,
  Item: { PK: 'USER#u123', SK: 'PROFILE', email: 'a@example.com', name: 'Alice', version: 1 },
  ConditionExpression: 'attribute_not_exists(PK) AND attribute_not_exists(SK)'
}).promise();

// Optimistic locking update
await dc.update({
  TableName: Table,
  Key: { PK: 'USER#u123', SK: 'PROFILE' },
  UpdateExpression: 'SET #n=:n, version=version+:one',
  ConditionExpression: 'version = :v',
  ExpressionAttributeNames: { '#n': 'name' },
  ExpressionAttributeValues: { ':n': 'Alice K', ':v': 1, ':one': 1 }
}).promise();

// Query user's recent orders
const r = await dc.query({
  TableName: Table,
  KeyConditionExpression: 'PK=:pk AND begins_with(SK,:sk)',
  ExpressionAttributeValues: { ':pk': 'USER#u123', ':sk': 'ORDER#' },
  ScanIndexForward: false, // 최신 우선
  Limit: 20,
  ExclusiveStartKey: /* 페이지 토큰 */
}).promise();
```

### 트랜잭션 예

```python
import boto3
from boto3.dynamodb.conditions import Key
d = boto3.client('dynamodb', region_name='ap-northeast-2')
tbl = 'AppTable'

# 주문 생성 + 상태 인덱스 기록을 원자적으로

d.transact_write_items(
  TransactItems=[
    {
      'Put': {
        'TableName': tbl,
        'Item': {
          'PK': {'S': 'USER#u123'},
          'SK': {'S': 'ORDER#2025-02-03T15:20#o2'},
          'orderId': {'S':'o2'}, 'status': {'S':'PAID'}, 'amount': {'N':'75'}
        },
        'ConditionExpression': 'attribute_not_exists(PK)'
      }
    },
    {
      'Put': {
        'TableName': tbl,
        'Item': {
          'PK': {'S':'ORDER#o2'}, 'SK': {'S':'ORDER'},
          'userId': {'S':'u123'}, 'status': {'S':'PAID'}, 'amount': {'N':'75'}
        }
      }
    },
    {
      'Put': {
        'TableName': tbl,
        'Item': {
          'PK': {'S':'STATUS#PAID'},
          'SK': {'S':'2025-02-03T15:20#o2'}
        }
      }
    }
  ])
```

### PartiQL(표준 SQL 유사)

```sql
-- Insert
INSERT INTO "AppTable" VALUE {'PK':'USER#u123','SK':'PROFILE','email':'a@example.com','name':'Alice'};

-- Update
UPDATE "AppTable" SET name='Alice K' WHERE PK='USER#u123' AND SK='PROFILE';

-- Select (주의: PartiQL의 Scan/Query 의미 구분)
SELECT * FROM "AppTable" WHERE PK='USER#u123' AND begins_with(SK,'ORDER#')
```

### CLI 스니펫

```bash
aws dynamodb put-item \
  --table-name AppTable \
  --item '{"PK":{"S":"USER#u1"},"SK":{"S":"PROFILE"},"email":{"S":"x@y.com"}}' \
  --condition-expression "attribute_not_exists(PK)"

aws dynamodb query \
  --table-name AppTable \
  --key-condition-expression "PK=:pk AND begins_with(SK,:sk)" \
  --expression-attribute-values '{":pk":{"S":"USER#u1"},":sk":{"S":"ORDER#"}}' \
  --scan-index-forward false \
  --limit 20
```

---

## 페이지네이션/정렬/패턴

- `Query` 기본 정렬: `SK` 오름차순. `ScanIndexForward=false`로 내림차순(최신 우선).
- 페이지네이션: `LastEvaluatedKey`/`ExclusiveStartKey` 사용. **토큰 보존**.
- 범위: `between`, `begins_with`, `>`, `<` 조합. SK 프리픽스 설계가 중요.

---

## 성능/확장/핫 파티션

- 파티션은 **키 해시** 기반 분배. **PK 카디널리티**를 충분히 높여 **균등 분산**.
- **핫 파티션** 방지: `PK`에 **접두 랜덤화**(예: `USER#u123#shard#07`), 쓰기 샤딩, 타임버킷.
- **어댑티브 캐패시티**: 불균형을 자동 보정하지만 **근본적으로 균등 키 설계**가 최선.

---

## 운영/관측/테스트

- **메트릭**: `ConsumedRead/WriteCapacityUnits`, `ThrottledRequests`, `SystemErrors`, `ReturnedItemCount`.
- **경보**: 스로틀 발생률, 지연 증가, 온디맨드 급증 비용.
- **부하 테스트**: k6/Locust + 백오프/지수재시도/아이템 크기 변화.
- **재시도 정책**: SDK 기본 재시도 + **지수 백오프 + 지터**.
- **아이템 크기**: 400KB 제한. 대용량은 S3 링크+메타만 저장.

Node.js 재시도 제어(예):

```js
const { DynamoDBClient, RetryStrategy } = require('@aws-sdk/client-dynamodb');
const client = new DynamoDBClient({
  region: 'ap-northeast-2',
  maxAttempts: 5
});
```

---

## 비용 구조와 최적화

### 근사 비용 수식

$$
\text{Monthly} \approx C_{\text{storage}} + C_{\text{read/write}} + C_{\text{GSI}} + C_{\text{backup/PITR}} + C_{\text{global}}
$$

- **저장**: 데이터 + 인덱스 사본. 압축 X(문자열/맵 최적화 필요).
- **읽기/쓰기**: 온디맨드(요청 기반) 또는 RCU/WCU(프로비저닝).
- **GSI**: **쓰기 시 GSI도 동시 갱신** → 쓰기 비용 증가.
- **백업/PITR/글로벌**: 사용량/리전 수에 따라 추가.

### 절감 팁

- **온디맨드 → 프로비저닝 전환**(패턴이 안정화되면 Auto Scaling 포함).
- **GSI 최소화**: 쓰기 증폭 방지. Projection을 **INCLUDE**로 최소 필드.
- **TTL**로 만료 데이터 제거.
- **DAX/어플리케이션 캐시**로 RCU 절감.
- **아이템 크기 줄이기**: 중복 필드/큰 배열/문자열 압축(애플리케이션 측) 또는 S3 오프로딩.
- **배치 쓰기/읽기**로 네트워크 왕복 감소.

---

## 보안/거버넌스

- **KMS 기본 암호화**. 고객관리키(CMK)로 접근 제어.
- **IAM 미세 권한**: 파티션 키 기반 조건(`dynamodb:LeadingKeys`).
- **VPC 엔드포인트**: 내부 경로 고정, 데이터 출구 통제.
- **감사**: CloudTrail로 테이블/인덱스/Backup 이벤트 추적.

---

## 실습: 풀스택 미니 프로젝트(서버리스)

### 요구

- API: `POST /users/{id}/orders`, `GET /users/{id}/orders`, `GET /orders/{orderId}`
- 서버리스(HTTP API + Lambda + DynamoDB), 온디맨드 모드

### 테이블 생성(CloudFormation 요약)

```yaml
Resources:
  AppTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: AppTable
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: PK  ; AttributeType: S
        - AttributeName: SK  ; AttributeType: S
      KeySchema:
        - AttributeName: PK  ; KeyType: HASH
        - AttributeName: SK  ; KeyType: RANGE
      SSESpecification:
        SSEEnabled: true
      TimeToLiveSpecification:
        AttributeName: ttl
        Enabled: true
```

### Lambda(주문 생성)

```python
import json, time, os
import boto3
d = boto3.resource('dynamodb').Table(os.environ['TABLE'])

def handler(event, context):
    user = event["pathParameters"]["id"]
    body = json.loads(event["body"])
    order_id = body["orderId"]
    ts = body.get("ts", int(time.time()))
    item_user_order = {
        "PK": f"USER#{user}",
        "SK": f"ORDER#{ts}#{order_id}",
        "orderId": order_id,
        "status": "PAID",
        "amount": body.get("amount", 0),
        "ttl": int(time.time()) + 3600*24*90
    }
    item_order = {
        "PK": f"ORDER#{order_id}",
        "SK": "ORDER",
        "userId": user,
        "status": "PAID",
        "amount": body.get("amount", 0)
    }
    d.meta.client.transact_write_items(
      TransactItems=[
        {"Put": {"TableName": d.name, "Item": {k: {'S':str(v)} if isinstance(v,str) else {'N':str(v)} for k,v in item_user_order.items()}, "ConditionExpression":"attribute_not_exists(PK)"}},
        {"Put": {"TableName": d.name, "Item": {k: {'S':str(v)} if isinstance(v,str) else {'N':str(v)} for k,v in item_order.items()}}}
      ]
    )
    return {"statusCode": 201, "body": json.dumps({"ok": True, "orderId": order_id})}
```

### Lambda(주문 목록)

```python
def handler(event, context):
    user = event["pathParameters"]["id"]
    params = {
      "TableName": os.environ["TABLE"],
      "KeyConditionExpression": "PK = :pk AND begins_with(SK,:sk)",
      "ExpressionAttributeValues": {":pk": {"S": f"USER#{user}"}, ":sk": {"S":"ORDER#"}},
      "ScanIndexForward": False,
      "Limit": 20
    }
    if "queryStringParameters" in event and event["queryStringParameters"] and "cursor" in event["queryStringParameters"]:
        params["ExclusiveStartKey"] = json.loads(event["queryStringParameters"]["cursor"])
    r = d.meta.client.query(**params)
    return {
      "statusCode": 200,
      "body": json.dumps({
        "items": r.get("Items", []),
        "cursor": r.get("LastEvaluatedKey")
      })
    }
```

> 이 구성만으로 **API + 데이터 + 만료(TTL) + 트랜잭션 + 페이지네이션**을 모두 체험 가능.

---

## 고급 토픽

### 조건부 유니크 제약

- 이메일 유니크: `PK=EMAIL#addr`, `SK=USER#id` 항목을 **조건식**으로 확보(`attribute_not_exists(PK)`).

### 감사 로그 아웃박스 패턴

- 쓰기 트랜잭션에 **이벤트 레코드**를 함께 기록 → Streams/Lambda가 외부 시스템 발행(카프카/이메일/슬랙).

### 부분 업데이트 표현식

```js
await dc.update({
  TableName: 'AppTable',
  Key: { PK:'USER#u1', SK:'PROFILE' },
  UpdateExpression: 'SET #n=:n, lastLogin=:now ADD loginCount :one',
  ExpressionAttributeNames: { '#n':'name' },
  ExpressionAttributeValues: { ':n':'Bob', ':now': Date.now(), ':one': 1 }
}).promise();
```

### 에러/재시도/아이템포턴시

- Put/Update/Delete는 **자연스럽게 멱등화** 가능(같은 키/버전).
- 트랜잭션 충돌 시 **재시도 + 랜덤 지터**.

---

## 체크리스트

- [ ] 액세스 패턴을 글머리표로 **먼저** 나열했는가?
- [ ] `PK/SK` 네이밍이 쿼리 요구를 모두 만족하는가?
- [ ] GSI가 **정말 필요한가**(쓰기 증폭/비용)?
- [ ] 온디맨드 vs 프로비저닝(오토스케일) 적절 선택?
- [ ] TTL/백업/PITR/Streams/Global 필요여부 검토?
- [ ] 핫 파티션 방지 설계(샤딩/버킷)?
- [ ] SDK 재시도/지수백오프/타임아웃 설정?
- [ ] CloudWatch 경보(스로틀/지연/소비용량/에러율)?
- [ ] 대용량 속성은 S3로 오프로딩?

---

## 마무리

DynamoDB는 **액세스 패턴 중심 설계**를 전제로 하면, 초고성능·초확장·고가용성을 **낮은 운영 부담**으로 달성하게 해준다.
본 가이드의 단일 테이블 예제, 조건식/트랜잭션, Streams, TTL·PITR·Global Tables, DAX, 비용 최적화까지 조합하면, **실전 서비스**에서 요구하는 성능·안정성·비용 균형을 빠르게 확보할 수 있다.
