---
layout: post
title: AWS - Kinesis Data Streams
date: 2025-07-28 19:20:23 +0900
category: AWS
---
# AWS Kinesis Data Streams 실습

## 전체 아키텍처 한눈에

```text
Producers (App, Web, IoT, Logs, ETL)
   └─ PutRecord/PutRecords/KPL/Kinesis Agent
        ▼
  Amazon Kinesis Data Streams (Provisioned/On-Demand)
        ├─ Shards & Enhanced Fan-Out (EFO)
        ├─ Data Retention (24h~7d, 장기보관 옵션)
        ▼
Consumers
  • Lambda Event Source (poller 내장)
  • KCL/Kinesis Client Library (Java/Python)
  • KDA for Apache Flink (상태/윈도우/CEP)
  • Firehose (S3/Redshift/OpenSearch 전송)
  • Custom SDK Pollers
```

---

## 스트림 생성 (콘솔/CLI/IaC)

### 콘솔

- **Stream name**: `my-data-stream`
- **Capacity mode**: Provisioned(샤드 수 지정) 또는 **On-demand**(자동 확장)
- **Shards**: 시작은 1~2, 부하 패턴에 맞춰 탄력적 조정(뒤에서 Resharding)

### CLI

```bash
aws kinesis create-stream --stream-name my-data-stream --shard-count 1
aws kinesis describe-stream-summary --stream-name my-data-stream
```

### Terraform(선택)

```hcl
resource "aws_kinesis_stream" "main" {
  name             = "my-data-stream"
  shard_count      = 1
  retention_period = 72 # hours (24~168)
  stream_mode_details { stream_mode = "PROVISIONED" } # 또는 ON_DEMAND
  encryption_type   = "KMS"
  kms_key_id        = aws_kms_key.kinesis.arn
}
```

---

## Producer: 단일/배치/고성능 전송

### CLI로 단건 전송

```bash
aws kinesis put-record \
  --stream-name my-data-stream \
  --partition-key "user-1" \
  --data "SGVsbG8gS2luZXNpcw=="
```
- `--data`는 **base64** 인코딩(단, SDK는 바이트 직접 전달이 일반적)
- **Partition Key**는 샤드 매핑(해시)을 위한 핵심 값

### Python (boto3) 단건/배치 전송

```python
import boto3, json, time, os, random
kinesis = boto3.client('kinesis', region_name=os.getenv('AWS_REGION','ap-northeast-2'))
stream = 'my-data-stream'

# 단건

def put_one(i):
    payload = {'event_id': i, 'user': f'user-{i%5}', 'action': 'click'}
    kinesis.put_record(StreamName=stream, Data=json.dumps(payload), PartitionKey=payload['user'])

# 배치(최대 500, 5MB)

def put_batch(batch_size=100):
    recs = []
    for i in range(batch_size):
        payload = {'event_id': i, 'user': f'user-{i%50}', 'action': random.choice(['click','view','buy'])}
        recs.append({'Data': json.dumps(payload), 'PartitionKey': payload['user']})
    resp = kinesis.put_records(StreamName=stream, Records=recs)
    print("Failed:", resp['FailedRecordCount'])

if __name__ == "__main__":
    for i in range(10):
        put_one(i)
        time.sleep(0.5)
    put_batch(200)
```

### 고성능: **KPL (Kinesis Producer Library)** 요점

- 배치·집계·압축 자동화, 네트워크 효율 상승
- 레코드 집계(Record Aggregation)로 PUT TPS/요금 절감
- 단, 소비 시 **deaggregation** 필요(KCL/SDK 지원)

---

## Consumer: 3가지 접근법

### CLI(Shard Iterator 기반) – 학습용

```bash
aws kinesis describe-stream --stream-name my-data-stream

aws kinesis get-shard-iterator \
  --stream-name my-data-stream \
  --shard-id shardId-000000000000 \
  --shard-iterator-type TRIM_HORIZON

aws kinesis get-records --shard-iterator <IteratorFromAbove>
```

**ShardIteratorType**:
- `TRIM_HORIZON`: 스트림 시작부터
- `LATEST`: 현재 이후만
- `AT_TIMESTAMP`: 지정 시각 이후

### Python 간단 Poller

```python
import boto3, json, time
k = boto3.client('kinesis')
stream = 'my-data-stream'
shard_id = k.describe_stream(StreamName=stream)['StreamDescription']['Shards'][0]['ShardId']
it = k.get_shard_iterator(StreamName=stream, ShardId=shard_id, ShardIteratorType='TRIM_HORIZON')['ShardIterator']

while True:
    out = k.get_records(ShardIterator=it, Limit=100)
    for r in out['Records']:
        print(json.loads(r['Data']))
    it = out['NextShardIterator']
    time.sleep(0.5)
```

### Lambda(권장, 관리형 Poller 내장)

- Lambda 트리거: **Kinesis** 선택 → 배치 크기, Starting position 설정
```python
import base64, json

def lambda_handler(event, context):
    for rec in event['Records']:
        data = base64.b64decode(rec['kinesis']['data']).decode('utf-8')
        obj = json.loads(data)
        # 처리 로직...
```
- **동시성/배치/에러**는 Lambda가 관리(최대 10,000 batch/s per shard 제한 고려)

---

## KCL (Kinesis Client Library)로 안정적인 소비

### 왜 KCL?

- 샤드 할당/밸런싱, 체크포인팅, 재시작·리샤딩 대응 자동화
- Java(Python/Node 래퍼 가능), DynamoDB로 체크포인트 관리

### Java KCL v2 스니펫

```java
public class App {
  public static void main(String[] args) {
    ConfigsBuilder cfg = new ConfigsBuilder(
        "my-data-stream", "my-kcl-app",
        new DefaultCredentialsProvider(),
        "worker-1", new RecordProcessorFactory(), Regions.AP_NORTHEAST_2);

    Scheduler scheduler = new Scheduler(
        cfg.checkpointConfig(),
        cfg.coordinatorConfig(),
        cfg.leaseManagementConfig(),
        cfg.lifecycleConfig(),
        cfg.metricsConfig(),
        cfg.processorConfig(),
        cfg.retrievalConfig());

    Runtime.getRuntime().addShutdownHook(new Thread(scheduler::shutdown));
    scheduler.run();
  }
}

class RecordProcessor implements ShardRecordProcessor {
  @Override public void initialize(InitializationInput input) {}
  @Override public void processRecords(ProcessRecordsInput input) {
    input.records().forEach(r -> {
      String data = new String(r.data().array());
      // 처리...
    });
    // 주기적 checkpoint
  }
  @Override public void leaseLost(LeaseLostInput input) {}
  @Override public void shardEnded(ShardEndedInput input) {}
  @Override public void shutdownRequested(ShutdownRequestedInput input) {}
}
```

---

## 샤드·파티션 키·처리량 공식

### 샤드 처리량

- **쓰기(Shard당)**: 최대 **1 MB/s** or **1,000 records/s**
- **읽기(Shard당)**: 최대 **2 MB/s**
- 파티션 키 해시로 샤드가 결정 → 편향(Hot partition) 방지

### 간이 용량 산정

$$
\text{필요 샤드 수} \ge \max\left(
\left\lceil \frac{\text{입력 MB/s}}{1} \right\rceil,
\left\lceil \frac{\text{입력 레코드/s}}{1000} \right\rceil,
\left\lceil \frac{\text{소비 MB/s}}{2} \right\rceil
\right)
$$

### 파티션 키 설계 팁

- `user_id`·`session_id` 등 **자연 분산**되는 키 사용
- 날짜/고정 값 결합은 편향 초래 → 해시/랜덤 접미사 부여

---

## 리샤딩(Scale out/in)

### 수동 Resharding

```bash
# 스플릿(샤드 1→2)

aws kinesis split-shard --stream-name my-data-stream \
  --shard-to-split shardId-000000000000 --new-starting-hash-key 170141183460469231731687303715884105728

# 머지(샤드 2→1)

aws kinesis merge-shards --stream-name my-data-stream \
  --shard-id shardId-000000000001 --adjacent-shard-id shardId-000000000002
```
- 소비자는 KCL이 자동 적응, Lambda는 내부 poller가 반영
- **On-demand** 모드: 자동 용량 조정(버스트 트래픽에 적합)

---

## 순서 보장·중복·재처리

### 순서

- **샤드 내** 파티션 키 단위 순서 보장
- 순서 중요 시 **동일 파티션 키** 사용 + 샤드 과포화 주의

### 중복·멱등성

- 네트워크/재시도로 **중복 발생 가능**
- **idempotent sink** 설계: 레코드 키 기반 upsert, Exactly-once는 다운스트림(예: Flink state)에서 달성

### 재처리

- **Retention**(24h~7d, 장기보관 옵션) 내에서 `AT_TIMESTAMP`/`TRIM_HORIZON`으로 재읽기
- Lambda: 실패 배치 → **bisect on function error** 옵션, DLQ(SQS/SNS) 연계

---

## 보안: IAM/KMS/VPC

### IAM(생산자/소비자 최소권한)

```json
{
  "Version":"2012-10-17",
  "Statement":[
    {"Effect":"Allow","Action":["kinesis:PutRecord","kinesis:PutRecords"],"Resource":"arn:aws:kinesis:ap-northeast-2:111122223333:stream/my-data-stream"}
  ]
}
```

### 암호화

- **서버측 암호화**(SSE-KMS) 활성화
- 전송구간 TLS 1.2+, 프라이빗 접근은 **VPC 인터페이스 엔드포인트** 활용

---

## 모니터링/알람/로그

### CloudWatch 지표

- Producers: `PutRecord.Success`, `PutRecords.ThrottledRecords`
- Stream: `IncomingBytes`, `IncomingRecords`
- Consumers: `GetRecords.IteratorAgeMilliseconds`(백로그 지표)

### 알람 예시

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name KDS-IteratorAge-High \
  --namespace AWS/Kinesis \
  --metric-name GetRecords.IteratorAgeMilliseconds \
  --dimensions Name=StreamName,Value=my-data-stream \
  --statistic Maximum --period 60 --threshold 60000 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:ap-northeast-2:111122223333:Ops
```

---

## 비용·성능 최적화

### 비용 구성

- **Provisioned**: 샤드/시간, PUT 요청 수
- **On-demand**: 데이터 처리량 기반
- KMS, VPC 엔드포인트, Lambda/KCL/KDA 등 **연계 비용** 포함

### 간이 비용 모델

$$
\text{월 비용} \approx (\text{샤드 수} \times \text{시간} \times p_{\text{shard-hour}})
+ \text{PUT 요청} \times p_{\text{req}} + \text{옵션}(KMS/VPC/Lambda)
$$

### 성능 팁

- Producer: **PutRecords(배치)** 또는 **KPL** 사용
- 레코드 크기 10–50KB 수준으로 묶기(네트워크 효율↑)
- Hot partition 발생 시 파티션 키 설계 재검토

---

## 운영 체크리스트

- [ ] **용량 모드**(Provisioned/On-demand)와 초기 샤드 수 결정
- [ ] **파티션 키** 분산성 테스트(샘플 해시 분포)
- [ ] **Retention** 기간/재처리 전략 문서화
- [ ] Lambda/KCL **배치/동시성/오류** 정책 설정
- [ ] **IteratorAge** 알람 + 자동 대응 런북
- [ ] **SSE-KMS**/VPC 엔드포인트/정밀 IAM
- [ ] IaC(Terraform/CFn)로 **선형 재현** 가능하게 관리

---

## 고급: Enhanced Fan-Out(EFO) & KDA/Flink

### EFO

- 소비자당 **전용 2MB/s** 푸시 기반 전송(지연 수 ms)
- 소비자 수가 많고 지연이 중요한 경우 고려(비용 증가)

### KDA for Apache Flink

- 상태/윈도우/CEP/Exactly-once(시맨틱)
- 예: 5분 슬라이딩 윈도우 사용자별 집계 후 S3/Redshift 전송
```java
events
  .keyBy(e -> e.userId)
  .window(SlidingEventTimeWindows.of(Time.minutes(5), Time.minutes(1)))
  .aggregate(new CountAgg())
  .addSink(new FlinkKinesisProducer<>(...));
```

---

## 실전 시나리오 (End-to-End)

### 생산자(웹/모바일) → KDS

```javascript
// 브라우저 예: API Gateway→Lambda→KDS 프록시(서명/보안)
fetch('/ingest', { method:'POST', body: JSON.stringify({ user, action, ts: Date.now() }) });
```

### 소비자(옵션 3가지)

1) Lambda: 간단 로직/후속 이벤트(SNS/SQS/Firehose)
2) KCL: 정확한 체크포인팅·대량 처리·복잡 로직
3) KDA(Flink): 준실시간 분석·상태 머신·세션/윈도우

### 실패/재처리

- Lambda DLQ(SQS) 연결 → 재처리 워커
- Retention 내 재검토: `AT_TIMESTAMP`로 복원 처리

---

## 예제: Lambda Consumer + DynamoDB 멱등 Upsert

```python
import base64, json, boto3, os, hashlib
ddb = boto3.resource('dynamodb').Table(os.getenv('TABLE','events'))

def key_of(rec):
    j = json.dumps(rec, sort_keys=True)
    return hashlib.sha256(j.encode()).hexdigest()[:32]

def lambda_handler(event, context):
    with ddb.batch_writer() as bw:
        for r in event['Records']:
            obj = json.loads(base64.b64decode(r['kinesis']['data']))
            pk = key_of(obj)
            # 멱등 upsert (조건식/버전 관리 적용 가능)
            bw.put_item(Item={'pk': pk, 'ts': obj.get('ts'), 'user': obj.get('user'), 'action': obj.get('action')})
```

---

## 실습 확장: PutRecords 스트레스 테스트

```python
import boto3, json, random, time
k = boto3.client('kinesis'); stream='my-data-stream'

def load_test(rate=2000, seconds=60):
    start = time.time()
    sent = 0
    while time.time() - start < seconds:
        batch = []
        for _ in range(min(500, rate)):  # PutRecords limit 500
            u = f"user-{random.randint(1,10000)}"
            batch.append({'Data': json.dumps({'u':u,'a':'click','ts':time.time()}), 'PartitionKey': u})
        k.put_records(StreamName=stream, Records=batch)
        sent += len(batch)
        time.sleep(1)
    print("Sent:", sent)

if __name__ == "__main__":
    load_test(2000, 30)
```

- 실행 중 CloudWatch **IncomingBytes/Records**, 소비 측 **IteratorAge**를 동시 관찰
- 샤드 포화 시 **Resharding** 또는 **On-demand** 전환 고려

---

## 자주 묻는 질문(FAQ)

**Q. Record 최대 크기?**
A. 1MB/레코드(압축은 앱 레벨). 요청(배치) 최대 5MB, 500레코드.

**Q. 순서 보장이 필요한데 Throughput도 높아야 한다면?**
A. 파티션 키 설계로 **유사 순서 그룹**을 분할(예: userId의 해시 prefix+원래 키)하고 소비 측에서 **partial ordering** 허용.

**Q. KPL을 쓰면 좋은가?**
A. 고TPS·저비용에 유리. 단, Deaggregation 필요·라이브러리 운영 부담 고려.

**Q. On-demand vs Provisioned 선택?**
A. 트래픽 가변/초기 예측 어려움 → **On-demand**. 안정적·예측 가능·비용 통제 → **Provisioned**.

---

## 마무리 요약

| 주제 | 요점 |
|---|---|
| 용량 | 샤드 처리량 한계(1MB/s write, 2MB/s read)를 기준으로 산정 |
| 키 설계 | 파티션 키 분산성 확보로 Hot shard 방지 |
| 소비 | Lambda(간단/관리형) vs KCL(정교/체크포인트) vs KDA(Flink/고급분석) |
| 운영 | IteratorAge/Throttling 알람, Resharding/On-demand |
| 보안 | SSE-KMS, VPC 엔드포인트, 최소권한 IAM |
| 비용/성능 | 배치·집계(KPL), Parquet/후속 파이프라인(필요 시) |
