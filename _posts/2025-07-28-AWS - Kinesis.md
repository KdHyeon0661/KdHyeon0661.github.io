---
layout: post
title: AWS - Kinesis
date: 2025-07-28 16:20:23 +0900
category: AWS
---
# AWS Kinesis: 실시간 데이터 스트리밍

## 0. 큰 그림: Kinesis 서비스군과 역할

| 서비스 | 핵심 역할 | 주요 목적지/연계 | 대표 시나리오 |
|---|---|---|---|
| **Kinesis Data Streams (KDS)** | 고성능 실시간 수집·분배(샤드 기반) | Lambda, KCL/KDA(Flink), Firehose, Custom | 클릭스트림, IoT, 거래 이벤트, 로그 버스 |
| **Kinesis Data Firehose** | 관리형 ETL 전송(S3/Redshift/OpenSearch/HTTP) | S3/Redshift/OpenSearch/3rd-party | 로그 적재, 준실시간 데이터 레이크 |
| **Kinesis Data Analytics** | SQL/Flink 기반 스트림 처리 | KDS/MSK → KDA → S3/Redshift/Lambda | 실시간 집계/이상탐지/세션 윈도우 |
| **Kinesis Video Streams** | 동영상 프레임 수집/저장/재생 | Rekognition/ML·CV 파이프라인 | CCTV/IoT 카메라, Media 분석 |

> 실무 감각: **고성능 버스(KDS)**, **싱크 자동화(Firehose)**, **실시간 분석(KDA)**를 조합하면, “수집→가공→적재→분석”의 **엔드투엔드 스트리밍 데이터 플랫폼**을 서버리스에 가깝게 구현할 수 있다.

---

## 1. Kinesis Data Streams 정밀 이해

### 1.1 구성 요소
- **Stream**: 레코드의 논리적 파이프.
- **Shard(샤드)**: 처리량 단위.
  - **Write 한도**: 1 MB/s 또는 1,000 records/s
  - **Read 한도**: 2 MB/s
- **Partition Key**: 해시로 샤드에 매핑. **샤드 내 순서 보장** 단위.
- **Retention**: 기본 24시간, 최대 **365일**(장기 보관 옵션). 재처리/리플레이 설계의 핵심.
- **Enhanced Fan-Out(EFO)**: 소비자별 2 MB/s 전용 채널(푸시 기반). 다수 소비자·저지연에 유리.

### 1.2 처리량·샤드 산정 공식

레코드 평균 크기를 \(B\) (bytes), 초당 레코드 수를 \(R\)라 하면:

$$
\text{입력 MB/s} = \frac{B \times R}{1024^2},\quad
\text{필요 샤드(쓰기)} \ge \max\left(\left\lceil \frac{\text{입력MB/s}}{1} \right\rceil,\ \left\lceil \frac{R}{1000} \right\rceil \right)
$$

소비자(읽기) 처리량 기준:

$$
\text{필요 샤드(읽기)} \ge \left\lceil \frac{\text{출력MB/s}}{2} \right\rceil
$$

전체 필요 샤드는 세 조건 중 **최대값**을 사용:

$$
\text{샤드 수} \ge \max\big(\text{쓰기 기준},\ \text{읽기 기준}\big)
$$

> 팁: 초기에 여유있게 시작하기보다 **모니터링 기반 Resharding**(Split/Merge) 또는 **On-Demand 모드**로 진입 장벽을 낮춘다.

### 1.3 Partition Key 설계 핵심
- **분산성** 확보: hot partition(특정 키 집중) 방지.
- 키 후보: `userId`, `sessionId`, `deviceId`(충분히 다양), 필요시 **해시 prefix** 부여.
- 순서 보장 요구 시: **동일 키 → 동일 샤드 → 순서 보장**. 단, 처리량 상한 고려.

---

## 2. 용량 모드 & 비용 구조

### 2.1 용량 모드
- **Provisioned**: 샤드 수 고정/조정. 예측 가능, 세밀 제어.
- **On-Demand**: 초당 처리량 기반 자동 스케일. 예측 어려운 워크로드에 유리(버스트 대응).

### 2.2 비용 항목(요약)
- Provisioned: **샤드 시간 비용** + **PUT/PATCH 요청**(KPL 집계로 절감 가능) + 옵션(EFO, 장기 보관).
- On-Demand: **쓰기/읽기 GB당 요금** + **스트림 시간 요금** + 옵션.

> 실무 체크: 레코드 크기를 **10~50KB**로 묶고(**PutRecords**/KPL 집계), 다운스트림에서 **Parquet** 등 컬럼 포맷화를 병행하면 네트워크/저장/쿼리 비용까지 같이 절감된다.

---

## 3. 생산자(Producer) 구현

### 3.1 CLI(학습용)
```bash
aws kinesis create-stream --stream-name my-stream --shard-count 1

aws kinesis put-record \
  --stream-name my-stream \
  --partition-key "user-1" \
  --data "SGVsbG8gS2luZXNpcw=="   # "Hello Kinesis" base64

aws kinesis describe-stream-summary --stream-name my-stream
```

### 3.2 Python (boto3) – 단건/배치
```python
import boto3, json, time, random, os
k = boto3.client('kinesis', region_name=os.getenv('AWS_REGION','ap-northeast-2'))
stream = 'my-stream'

def put_one(i):
    d = {'event_id': i, 'user': f'user-{i%100}', 'action': random.choice(['view','click','buy']), 'ts': time.time()}
    k.put_record(StreamName=stream, Data=json.dumps(d), PartitionKey=d['user'])

def put_batch(n=200):
    recs = []
    for _ in range(n):
        d = {'user': f'user-{random.randint(1,10000)}', 'a': 'click', 'ts': time.time()}
        recs.append({'Data': json.dumps(d), 'PartitionKey': d['user']})
    resp = k.put_records(StreamName=stream, Records=recs)
    print("Failed:", resp['FailedRecordCount'])

for i in range(10):
    put_one(i)
put_batch(500)  # 최대 500/요청, 5MB/요청
```

### 3.3 KPL(Kinesis Producer Library) – 고TPS 팁
- 장점: 자동 **집계/압축/재시도**, 네트워크 효율 최적화 → **PUT 요금 절감**.
- 고려: 소비 측 **Deaggregation** 필요(KCL/SDK 지원).
- JVM/네이티브 의존성 운영 고려(컨테이너 이미지에 포함).

---

## 4. 소비자(Consumer) 구현

### 4.1 CLI(Shard Iterator)
```bash
aws kinesis describe-stream --stream-name my-stream
# ShardId 확인

aws kinesis get-shard-iterator \
  --stream-name my-stream \
  --shard-id shardId-000000000000 \
  --shard-iterator-type TRIM_HORIZON

aws kinesis get-records --shard-iterator <Iterator>
```

### 4.2 Python 간단 Poller
```python
import boto3, json, time
k = boto3.client('kinesis'); stream='my-stream'

shards = k.describe_stream(StreamName=stream)['StreamDescription']['Shards']
it = k.get_shard_iterator(StreamName=stream, ShardId=shards[0]['ShardId'],
                          ShardIteratorType='LATEST')['ShardIterator']

while True:
    out = k.get_records(ShardIterator=it, Limit=100)
    for r in out['Records']:
        print(json.loads(r['Data']))
    it = out['NextShardIterator']
    time.sleep(0.2)
```

### 4.3 Lambda(관리형 Poller 내장)
- **Kinesis 트리거** 연결 → 배치 크기/시작 포지션 설정.
```python
import base64, json

def lambda_handler(event, context):
    for rec in event['Records']:
        data = json.loads(base64.b64decode(rec['kinesis']['data']).decode())
        # 멱등 로직/검증/후속 작업...
```
- 실패 시 **bisect on function error** / **DLQ(SQS/SNS)**로 재처리 라인 유지.
- 샤드당 동시성/배치 설정으로 처리량·지연 균형 맞추기.

### 4.4 KCL(Kinesis Client Library) – 프로덕션 등뼈
- 기능: **샤드 할당/리밸런싱/체크포인팅/리샤딩 대응** 자동화.
- 체크포인트 저장: DynamoDB(자동 생성 테이블).
- 언어: Java(정석), Python/Node 래퍼 존재.

---

## 5. 리샤딩(Resharding) & On-Demand

### 5.1 수동 Resharding
```bash
# Split: 1샤드 → 2샤드
aws kinesis split-shard --stream-name my-stream \
  --shard-to-split shardId-000000000000 \
  --new-starting-hash-key 170141183460469231731687303715884105728

# Merge: 인접 2샤드 → 1샤드
aws kinesis merge-shards --stream-name my-stream \
  --shard-id shardId-000000000001 \
  --adjacent-shard-id shardId-000000000002
```
- KCL/Lambda 소비자는 자동 적응.
- 지표: `IncomingBytes/Records`, `GetRecords.IteratorAgeMilliseconds`를 근거로 결정.

### 5.2 On-Demand 모드
- 급격한 버스트/예측 곤란 워크로드에 적합.
- 수동 리샤딩 부담 ↓, 단위 비용 모델을 사전 검토.

---

## 6. 순서 보장·중복·재처리 전략

### 6.1 순서 보장
- **샤드 내** 파티션 키 단위로 순서 보장.
- 순서가 중요한 그룹은 동일 키에 묶고, 처리량 상한 고려해 **key sharding**(prefix+원키)로 확장.

### 6.2 중복/멱등성
- 네트워크 재시도/중복 Put 가능 → **멱등 소비자** 필수.
- 예: DynamoDB upsert 시 **조건식**/버전키 또는 해시(pk)로 **중복 무해화**.

```python
# 멱등 Upsert 예시 (Lambda → DynamoDB)
import base64, json, boto3, hashlib, os
ddb = boto3.resource('dynamodb').Table(os.getenv('TABLE','events'))

def key_of(obj):
    j = json.dumps(obj, sort_keys=True)
    return hashlib.sha256(j.encode()).hexdigest()[:32]

def lambda_handler(event, context):
    with ddb.batch_writer() as bw:
        for r in event['Records']:
            obj = json.loads(base64.b64decode(r['kinesis']['data']).decode())
            bw.put_item(Item={'pk': key_of(obj), 'ts': obj.get('ts'), 'user': obj.get('user'), 'a': obj.get('a')})
```

### 6.3 재처리/리플레이
- Retention 윈도 내 **AT_TIMESTAMP/TRIM_HORIZON**으로 재소비.
- Lambda: 실패 재시도 정책 + DLQ/오프라인 재주입(백필).

---

## 7. 보안·네트워킹

### 7.1 IAM 최소 권한(예: Producer)
```json
{
  "Version":"2012-10-17",
  "Statement":[
    {"Effect":"Allow","Action":["kinesis:PutRecord","kinesis:PutRecords"],
     "Resource":"arn:aws:kinesis:ap-northeast-2:111122223333:stream/my-stream"}
  ]
}
```

### 7.2 암호화/네트워크
- **SSE-KMS**(서버측 암호화) 활성화.
- 전송구간 TLS 1.2+.
- 사설 통신: **VPC 인터페이스 엔드포인트**로 프라이빗 경로 확보.

---

## 8. 모니터링·알람·운영 관측성

### 8.1 핵심 지표
- **Producer**: `PutRecord.Success`, `PutRecords.ThrottledRecords`, `FailedRecordCount`
- **Stream**: `IncomingBytes/Records`
- **Consumer**: `GetRecords.IteratorAgeMilliseconds`(백로그/지연의 대표 지표), `ReadProvisionedThroughputExceeded`

### 8.2 알람 예시(IteratorAge 상승)
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name KDS-IteratorAge-High \
  --namespace AWS/Kinesis --metric-name GetRecords.IteratorAgeMilliseconds \
  --dimensions Name=StreamName,Value=my-stream \
  --statistic Maximum --period 60 --threshold 60000 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:ap-northeast-2:111122223333:Ops
```

> 런북: 알람 발생 시 **(1) 샤드 포화 확인 → (2) Resharding/On-Demand 전환 → (3) Hot key/네트워크 오류 점검 → (4) 소비자 병렬도↑/배치 설정 조정**.

---

## 9. 운영 비용·성능 최적화

### 9.1 비용 모델(개념식)
$$
\text{월 비용} \approx (\text{샤드 수} \times \text{시간} \times p_{\text{shard-hour}})
+ (\text{PUT 요청 수} \times p_{\text{req}})
+ \text{옵션}(EFO/KMS/보관)
$$

On-Demand는 **GB 처리량**과 **스트림 시간**을 가중.

### 9.2 실무 팁
- Producer **배치/집계**(PutRecords/KPL)로 요청/요금 감소.
- **Hot partition** 방지: 파티션 키 다양화·해시 prefix.
- 다운스트림(데이터 레이크)에서 **Parquet**·압축으로 저장/쿼리 비용↓.
- **테스트→관측→리샤딩/튜닝** 반복(블루/그린 소비자 배포로 무중단 실험).

---

## 10. 패턴별 아키텍처 예시

### 10.1 KDS → Lambda → (SQS DLQ) → DynamoDB
- 단순 이벤트 처리/Enrichment/알림.
- DLQ로 **데이터 손실 0** 설계.

### 10.2 KDS → KDA(Flink/SQL) → Firehose → S3/Redshift
- 실시간 집계/세션 윈도/CEP → 저비용 장기 저장/BI.

### 10.3 KDS → 다중 소비자(EFO)
- Fraud/LTV/알림팀 등 **각 팀 전용 2MB/s**로 분리, 상호 간섭 최소화.

---

## 11. IaC(예: Terraform) 스캐폴딩

```hcl
resource "aws_kinesis_stream" "main" {
  name               = "my-stream"
  shard_count        = 2
  retention_period   = 72
  encryption_type    = "KMS"
  kms_key_id         = aws_kms_key.kinesis.arn
  stream_mode_details { stream_mode = "PROVISIONED" }
}

resource "aws_lambda_event_source_mapping" "kinesis" {
  event_source_arn  = aws_kinesis_stream.main.arn
  function_name     = aws_lambda_function.consumer.arn
  starting_position = "LATEST"
  batch_size        = 500
  maximum_batching_window_in_seconds = 1
  parallelization_factor = 1
}
```

---

## 12. 종단간 실습 시나리오 (End-to-End)

1) **Stream 생성**: `my-stream`, PROVISIONED shard=1
2) **Producer 배치 주입**(위 Python `put_batch` 500/초)
3) **Consumer: Lambda** 배치=500, 윈도=1s, DLQ=SQS 연결
4) **알람**: IteratorAge>60s 시 SNS 통보
5) **부하 상승** → `IncomingBytes`/`IteratorAge` 확인
6) **Split-shard** 실행 또는 On-Demand 전환
7) **Hot key** 발견 시 producer 파티션 키 설계 변경 롤아웃
8) **비용 검토**: KPL 적용 → PutRequests 감소, 다운스트림 Parquet 변환

---

## 13. 자주 묻는 질문(FAQ)

**Q1. 레코드 최대 크기/요청 제한은?**
A. 레코드 **최대 1MB**. `PutRecords`는 **500개/5MB** 제한.

**Q2. 순서 보장과 처리량을 동시에 높이려면?**
A. “완전 전역 순서”는 비용↑병목↑. **순서가 필요한 단위로 파티셔닝**(키 샤딩)하고, 소비자에서 **부분 순서** 허용/보상 로직 설계.

**Q3. 중복 방지는 어떻게?**
A. 멱등 처리: 고유키 기반 upsert/조건식, 해시키, **Exactly-once**는 Flink 상태/싱크에서 달성.

**Q4. Lambda vs KCL 선택 기준?**
A. **간단·관리형**(운영 단순성/자동 스케일) → Lambda. **정교한 체크포인트·세밀 제어** → KCL.

**Q5. Firehose와 차이는?**
A. Firehose는 **목적지 전송 중심(ETL/버퍼/압축/포맷)**, KDS는 **고성능 스트림 버스**. 필요 시 **KDS→Firehose** 연동.

---

## 14. 요약 정리 (Cheat Sheet)

| 항목 | 핵심 포인트 |
|---|---|
| 샤드 처리량 | Write 1 MB/s·1000 rec/s, Read 2 MB/s |
| 키 설계 | 분산성(Hot key 방지) + 순서 요구 범위에 맞춘 파티셔닝 |
| 소비자 | Lambda(간편), KCL(정밀), KDA(Flink 고급) |
| 리샤딩 | CloudWatch 지표 기반 Split/Merge, 또는 On-Demand |
| 보안 | SSE-KMS, VPC 엔드포인트, 최소권한 IAM |
| 관측 | IteratorAge·Throttling 알람 + 런북 |
| 비용 | 배치/집계(KPL), Parquet/압축, 적정 모드 선택 |

---

## 부록 A) 스트레스 로더 & 관측(테스트 코드)

```python
# PutRecords 압박 테스트 (관측: IncomingBytes/IteratorAge)
import boto3, json, random, time
k = boto3.client('kinesis', region_name='ap-northeast-2')
stream = 'my-stream'

def load(rate_per_sec=2000, sec=60):
    end = time.time()+sec
    sent = 0
    while time.time() < end:
        batch = []
        for _ in range(min(500, rate_per_sec)):
            u = f"user-{random.randint(1,20000)}"
            batch.append({'Data': json.dumps({'u':u,'a':'click','ts':time.time()}),
                          'PartitionKey': u})
        k.put_records(StreamName=stream, Records=batch)
        sent += len(batch)
        time.sleep(1)
    print("Sent:", sent)

if __name__ == "__main__":
    load(2000, 30)
```

---

## 부록 B) 수식 모음

### B.1 샤드 산정 요약
$$
S_w = \max\left(\left\lceil\frac{B\cdot R}{2^{20}}\right\rceil,\ \left\lceil\frac{R}{1000}\right\rceil\right),\quad
S_r = \left\lceil\frac{\text{출력MB/s}}{2}\right\rceil,\quad
S = \max(S_w, S_r)
$$

### B.2 비용 개념식(Provisioned)
$$
\text{Cost} \approx S \cdot H \cdot p_{\text{shard-hour}} + \text{PUTs} \cdot p_{\text{req}} + \text{옵션}
$$
- \(S\): 평균 샤드 수, \(H\): 시간(월 환산), \(p\): 단가

---

## 결론

Kinesis는 **수집(Streams)**, **전송(Firehose)**, **분석(Analytics)**을 관통하는 **실시간 데이터 백본**이다.
핵심은 **파티션 키 설계/샤드 산정/멱등 소비/관측·알람/리샤딩 운영**이다.
본 가이드를 **템플릿** 삼아, 작은 트래픽부터 **관측→튜닝→확장** 루프를 돌리면 **안정적이고 비용 효율적인** 스트리밍 플랫폼을 구축할 수 있다.
