---
layout: post
title: AWS - Kinesis Data Streams
date: 2025-07-28 19:20:23 +0900
category: AWS
---
# 🌀 AWS Kinesis Data Streams 실습 가이드

Amazon Kinesis Data Streams(KDS)는 실시간으로 데이터를 수집하고 처리할 수 있는 AWS의 스트리밍 데이터 플랫폼입니다. 이 실습에서는 Kinesis 스트림 생성부터 데이터를 전송하고 소비하는 프로세스를 단계별로 구성해보겠습니다.

---

## 1️⃣ Kinesis Stream 생성

### 🔸 콘솔에서 생성

1. AWS 콘솔 > Kinesis > **Data Streams** > [Create data stream]
2. **Stream name**: `my-data-stream`
3. **Number of open shards**: 1 (기본값)
4. [Create Stream] 클릭

### 🔸 CLI로 생성

```bash
aws kinesis create-stream \
  --stream-name my-data-stream \
  --shard-count 1
```

### 🔸 확인

```bash
aws kinesis describe-stream-summary \
  --stream-name my-data-stream
```

---

## 2️⃣ Producer: 데이터 전송

### 🔸 AWS CLI를 이용한 간단한 데이터 전송

```bash
aws kinesis put-record \
  --stream-name my-data-stream \
  --partition-key "user-1" \
  --data "SGVsbG8gS2luZXNpcw=="  # "Hello Kinesis" base64
```

- `--data`는 base64 인코딩된 문자열이어야 합니다.
- `partition-key`는 데이터를 샤드에 분배할 때 사용됩니다.

### 🔸 Python으로 Producer 작성 (boto3)

```python
import boto3
import json
import time

kinesis = boto3.client('kinesis')

stream_name = 'my-data-stream'

for i in range(10):
    data = {'event_id': i, 'message': 'Kinesis test'}
    kinesis.put_record(
        StreamName=stream_name,
        Data=json.dumps(data),
        PartitionKey='partition-{}'.format(i % 2)
    )
    time.sleep(1)
```

---

## 3️⃣ Consumer: 데이터 읽기

### 방법 1. CLI를 통한 수동 소비 (Shard Iterator 사용)

1. Shard ID 조회

```bash
aws kinesis describe-stream \
  --stream-name my-data-stream
```

2. Shard Iterator 가져오기

```bash
aws kinesis get-shard-iterator \
  --stream-name my-data-stream \
  --shard-id shardId-000000000000 \
  --shard-iterator-type TRIM_HORIZON
```

3. 데이터 가져오기

```bash
aws kinesis get-records \
  --shard-iterator <IteratorReturnedAbove>
```

---

### 방법 2. Python 소비자 예제 (Shard Iterator 사용)

```python
import boto3
import json

kinesis = boto3.client('kinesis')
stream_name = 'my-data-stream'

# Shard ID 얻기
stream = kinesis.describe_stream(StreamName=stream_name)
shard_id = stream['StreamDescription']['Shards'][0]['ShardId']

# Iterator 얻기
response = kinesis.get_shard_iterator(
    StreamName=stream_name,
    ShardId=shard_id,
    ShardIteratorType='TRIM_HORIZON'
)
shard_iterator = response['ShardIterator']

# 데이터 읽기
while True:
    out = kinesis.get_records(ShardIterator=shard_iterator, Limit=10)
    records = out['Records']
    for record in records:
        print(json.loads(record['Data']))
    shard_iterator = out['NextShardIterator']
```

---

### 방법 3. Lambda를 Consumer로 연결

1. Lambda 함수 생성
2. 트리거 추가 > **Kinesis** > `my-data-stream` 선택
3. Batch size, Starting position 설정

```python
def lambda_handler(event, context):
    for record in event['Records']:
        payload = record['kinesis']['data']
        print(f"Decoded data: {base64.b64decode(payload).decode('utf-8')}")
```

---

## 4️⃣ CloudWatch를 통한 모니터링

- AWS 콘솔 > CloudWatch > Metrics > Kinesis
- `IncomingBytes`, `GetRecords.IteratorAgeMilliseconds`, `PutRecord.Success` 등 주요 지표 확인 가능

---

## 5️⃣ 스트림 삭제

### 🔸 CLI

```bash
aws kinesis delete-stream \
  --stream-name my-data-stream
```

---

## 📌 보충 개념: 샤드란?

- 각 샤드는 최대 **1MB/s 쓰기**, **2MB/s 읽기**, **최대 1000 PUT/s**를 지원합니다.
- **샤드 수를 늘리면 처리량이 증가**하지만 비용도 증가합니다.
- 샤드를 기준으로 병렬 처리가 가능합니다.

---

## 📘 정리

| 구성 요소     | 설명 |
|--------------|------|
| Stream        | 데이터의 논리적인 흐름 단위 |
| Shard         | 병렬 처리를 위한 단위, 처리량 기준 |
| Producer      | 데이터를 Kinesis에 전송 |
| Consumer      | 데이터를 읽는 주체 (Lambda, CLI, App 등) |
| Record        | 실제 데이터 단위 (Data + Partition Key + Sequence Number) |
| Iterator      | 샤드에서 데이터를 읽기 위한 커서 |

---

## 🧠 참고

- 실시간 분석을 위해 Kinesis Data Analytics, Firehose와도 연동 가능
- PutRecord와 PutRecords (배치) 비교
- Lambda와 결합한 이벤트 기반 아키텍처 가능

---

## ✅ 마무리

이번 실습을 통해 Kinesis Data Stream의 기본적인 구조와 활용 방법을 익혔습니다. 실시간 로그 처리, IoT 이벤트 수집, 스트리밍 ETL 등 다양한 실시간 처리 아키텍처에 적용해볼 수 있습니다.
