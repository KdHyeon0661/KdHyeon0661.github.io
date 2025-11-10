---
layout: post
title: AWS - Redshift Spectrum
date: 2025-07-28 23:20:23 +0900
category: AWS
---
# Redshift Spectrum: S3 데이터를 Redshift에서 바로 쿼리하기

## 0) 큰 그림: 왜 Spectrum인가?

- **데이터 레이크(S3)** 에 무한히 쌓이는 원시/정제 데이터를 **DW(Redshift)** 에 **적재(복사) 없이** 곧바로 분석
- **열 지향 포맷(Parquet/ORC)** + **파티셔닝(연/월/일 등)** + **Glue Data Catalog** 로 **스캔 바이트 ↓ → 비용 ↓ / 성능 ↑**
- **Redshift 내부 테이블**과 **외부 테이블**을 **한 쿼리로 조인**해 **레거시+데이터 레이크 하이브리드 분석** 구현

---

## 1) 아키텍처와 동작

```text
S3 (데이터)  ←→  Glue Data Catalog (메타데이터)
                     ▲
                     │ 외부 스키마
Redshift 클러스터 ───┴─────(Redshift Spectrum 엔진)───▶ 읽기(병렬 스캔/필터/컬럼 프루닝)
      │
      └─ 내부 테이블(고속, 빈번쿼리) + 외부 테이블(대용량/비정형/저빈도)
```

**쿼리 흐름**
1. 사용자가 Redshift에서 `SELECT` 실행  
2. **외부 테이블** 포함 시 Spectrum이 **Glue 카탈로그**를 참조해 **S3 경로/스키마/파티션** 파악  
3. **프루닝**(파티션/컬럼/프레디케이트)을 적용, 필요한 S3 오브젝트만 병렬 스캔  
4. 중간 결과를 Redshift 실행계획에 결합 → 결과 반환

---

## 2) 권한과 네트워킹(IAM, Lake Formation, VPC)

### 2.1 IAM Role (Redshift → S3/Glue 접근)
- **신뢰 정책**: `redshift.amazonaws.com`
- **권한 정책**: S3 List/Get, Glue GetDatabase/GetTable, 필요 시 Lake Formation 권한

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:ListBucket"], "Resource": "arn:aws:s3:::my-bucket" },
    { "Effect": "Allow", "Action": ["s3:GetObject"], "Resource": "arn:aws:s3:::my-bucket/*" },
    { "Effect": "Allow", "Action": ["glue:GetDatabase","glue:GetTable","glue:GetPartitions"], "Resource": "*" }
  ]
}
```

**신뢰 정책**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Principal": { "Service": "redshift.amazonaws.com" }, "Action": "sts:AssumeRole" }
  ]
}
```

> Lake Formation을 사용 중이면 **LF 권한(SELECT, DESCRIBE)** 도 부여해야 실제 스캔이 허용된다.

### 2.2 VPC/네트워크
- Spectrum은 **퍼블릭 S3 엔드포인트** 접근 기준으로 동작 (클러스터 서브넷 라우팅 확인)
- **VPC 엔드포인트(S3 Gateway/Interface)** 사용 시 **트래픽 통제/비용 최적화** 가능

---

## 3) 외부 스키마/테이블 생성(Glue 카탈로그 기반)

### 3.1 외부 스키마 만들기
```sql
CREATE EXTERNAL SCHEMA spectrum_sales
FROM data catalog
DATABASE 'sales_db'
IAM_ROLE 'arn:aws:iam::123456789012:role/MySpectrumRole'
CREATE EXTERNAL DATABASE IF NOT EXISTS;
```

### 3.2 CSV 외부 테이블(학습용)
```sql
CREATE EXTERNAL TABLE spectrum_sales.raw_sales_csv (
  sale_id       INT,
  product_name  VARCHAR(200),
  amount        DECIMAL(10,2),
  sale_date     DATE
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION 's3://my-bucket/sales_data/csv/';
```

### 3.3 Parquet + 파티셔닝(실무권장)
```sql
CREATE EXTERNAL TABLE spectrum_sales.fact_sales_pq (
  sale_id       BIGINT,
  product_id    BIGINT,
  user_id       BIGINT,
  amount        DECIMAL(12,2),
  event_ts      TIMESTAMP
)
PARTITIONED BY (dt DATE, region VARCHAR(16))
STORED AS PARQUET
LOCATION 's3://my-bucket/datalake/fact_sales_pq/';
```

**파티션 추가**
```sql
ALTER TABLE spectrum_sales.fact_sales_pq
ADD IF NOT EXISTS PARTITION (dt='2025-11-01', region='KR')
LOCATION 's3://my-bucket/datalake/fact_sales_pq/dt=2025-11-01/region=KR/';
```

> Glue **Crawler** 로 파티션을 자동 동기화하거나, 주기적으로 **ALTER TABLE ADD PARTITION** 수행.  
> **파트션 프로젝션**(아래 6.4절)을 쓰면 **메타스토어에 실제 파티션 등록 없이** 프루닝이 가능(운영 편의↑).

### 3.4 JSON/로그(Serde) 예시
```sql
CREATE EXTERNAL TABLE spectrum_sales.events_json (
  user_id      VARCHAR(64),
  action       VARCHAR(64),
  amount       DOUBLE,
  event_time   TIMESTAMP
)
PARTITIONED BY (dt DATE)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES ('ignore.malformed.json'='true')
LOCATION 's3://my-bucket/events/json/';
```

---

## 4) 첫 쿼리 & 조인

### 4.1 기본 집계
```sql
SELECT product_name, SUM(amount) AS revenue
FROM spectrum_sales.raw_sales_csv
GROUP BY product_name
ORDER BY revenue DESC
LIMIT 20;
```

### 4.2 내부 테이블과 조인
```sql
-- 내부 차원 테이블
CREATE TABLE dim_product (
  product_id   BIGINT DISTKEY,
  product_name VARCHAR(200),
  category     VARCHAR(64)
);

-- 외부 사실 테이블(Parquet)
-- spectrum_sales.fact_sales_pq

SELECT d.category, SUM(f.amount) AS revenue
FROM spectrum_sales.fact_sales_pq f
JOIN dim_product d
  ON d.product_id = f.product_id
WHERE f.dt BETWEEN '2025-11-01' AND '2025-11-07'
GROUP BY d.category
ORDER BY revenue DESC;
```

> **하이브리드 조인**: 빈번/소용량 차원은 **내부**, 대용량 사실은 **외부**로 유지하면 **저비용·고유연성**을 얻는다.

---

## 5) 비용 모델 & 수식

- Spectrum 요금은 **스캔 바이트(GB)** 기준. (Region별 단가 상이)
- **열 포맷/압축/파티셔닝/프레디케이트**로 스캔 데이터량을 줄이는 것이 핵심

$$
\text{월 비용} \approx \left(\sum_{q=1}^{Q} \frac{\text{스캔바이트}_q}{\text{GB}}\right) \cdot p_{\text{GB}} \;+\; \text{(Glue/LF 등 부대비용)}
$$

**스캔량 근사**
$$
\text{스캔 바이트} \approx \frac{\text{파일크기(압축 해제 기준)} \times \text{선택 컬럼 비율} \times \text{필터 선택도}}{\text{압축율}}
$$

- **Parquet/ORC**: **컬럼 프루닝** + **압축** → 실제 스캔 바이트 크게 감소  
- **Partition Pruning**: `WHERE dt='2025-11-01'` 처럼 **파티션 키**에 대한 필터는 **파일 오픈 자체를 생략**한다

---

## 6) 성능 최적화: 핵심 7계명

### 6.1 파일 포맷 & 압축
- **Parquet + Snappy** (권장), ORC도 우수
- 컬럼형 + 압축으로 **I/O 최소화 / CPU 효율성↑**

### 6.2 파일 크기
- **128~512 MB**(압축 후) 단위로 **적절히 큰 파일**을 유지  
- 수백만 개 **소형 파일 문제**(small files) 방지 → **압도적인 메타/오브젝트 오버헤드**를 줄임  
- Glue/Spark 작업으로 **Compaction** 배치 수행

### 6.3 파티셔닝 전략
- `dt=YYYY-MM-DD/region=KR` 등 **고카디널리티 키를 상위에 두지 말 것**  
- 조회 패턴에 맞춰 **날짜 → 지역/채널** 순, 과도 분할 방지  
- **균형**: 프루닝 효과 vs 파티션 수(메타/리스트 비용)

### 6.4 **Partition Projection**(파티션 메타 없이 프루닝)
```sql
ALTER TABLE spectrum_sales.fact_sales_pq
SET TABLE PROPERTIES (
  'projection.enabled'='true',
  'projection.dt.type'='date',
  'projection.dt.range'='2024-01-01,2026-12-31',
  'projection.dt.format'='yyyy-MM-dd',
  'projection.region.type'='enum',
  'projection.region.values'='KR,US,JP',
  'storage.location.template'='s3://my-bucket/datalake/fact_sales_pq/dt=${dt}/region=${region}/'
);
```
- **대량 파티션 등록 없이**도 `WHERE dt BETWEEN …` 프루닝 가능 → **크롤/ALTER 부담↓**

### 6.5 프레디케이트/컬럼 프루닝
- `SELECT *` 금지, 필요한 컬럼만 선택  
- `WHERE` 는 **파티션 키 + 컬럼**을 함께 활용  
- 문자열 필터는 **정규식보다 등호/범위**가 효율적

### 6.6 조인/집계
- 외부 테이블은 **스캔 우선 비용**. 조인 시 **작은 테이블을 내부로**(Sort/Distkey 최적화), 외부는 큰 사실

### 6.7 캐시/리소스
- Redshift WLM/큐 구성으로 **동시성 관리**, Spectrum 스캔과 내부 작업이 서로 **자원 경합**하지 않게 QoS 설계

---

## 7) 머티리얼라이즈: UNLOAD/CTAS/외부 테이블 변환

### 7.1 외부 → 내부로 스냅샷(고속 BI 필요 구간)
```sql
CREATE TABLE fact_sales_hot AS
SELECT *
FROM spectrum_sales.fact_sales_pq
WHERE dt >= '2025-11-01';
```
- **빈번/저지연** 요구를 내부에 **부분 적재**하여 **혼합 전략** 채택

### 7.2 내부 → 레이크로 역방향(장기 보관/비용↓)
```sql
UNLOAD ($$ SELECT * FROM fact_sales_hot $$)
TO 's3://my-bucket/exports/fact_sales_hot_'
IAM_ROLE 'arn:aws:iam::123456789012:role/MySpectrumRole'
FORMAT PARQUET PARTITION BY (dt);
```

---

## 8) 실전 랩: 엔드투엔드 구축

### 8.1 S3 예시 데이터
```
s3://my-bucket/datalake/fact_sales_pq/
  dt=2025-11-01/region=KR/part-0001.parquet
  dt=2025-11-01/region=US/part-0002.parquet
  dt=2025-11-02/region=KR/part-0003.parquet
```

### 8.2 Glue 크롤러(선택) 또는 파티션 프로젝션
- 크롤러: 스키마/파티션 자동 수집(주기 실행)
- 프로젝션: 위 6.4 설정으로 ALTER 없이 프루닝

### 8.3 외부 스키마/테이블 생성(위 3절 참고)

### 8.4 쿼리 예시
```sql
-- 파티션 프루닝 + 컬럼 프루닝
SELECT region, SUM(amount) AS revenue
FROM spectrum_sales.fact_sales_pq
WHERE dt BETWEEN '2025-11-01' AND '2025-11-02'
GROUP BY region
ORDER BY revenue DESC;

-- 내부 조인
SELECT d.category, SUM(f.amount)
FROM spectrum_sales.fact_sales_pq f
JOIN dim_product d ON d.product_id = f.product_id
WHERE f.dt = '2025-11-01' AND f.region = 'KR'
GROUP BY d.category
ORDER BY 2 DESC;
```

### 8.5 성능 검증 체크리스트
- `EXPLAIN` / `STL_SCAN` / `SVL_S3LOG` 뷰로 **스캔 바이트/파일 수/필터 적용** 확인  
- **CloudWatch S3 요청** / Glue 호출량(크롤) 모니터링

---

## 9) 데이터 품질/스키마 진화

- Parquet 스키마 변경(**컬럼 추가/nullable**)은 비교적 유연하나 **타입 변경**은 주의  
- **Glue 스키마 버저닝**과 **백필(Backfill)** 전략 병행  
- **스키마 온 리드** 환경이므로 **계약(Contract)** 문서화/검증 파이프라인 준비(예: Deequ/Great Expectations)

---

## 10) 보안·컴플라이언스

- **SSE-S3/SSE-KMS** 로 S3 암호화, IAM Role에 KMS 권한 포함
- 민감 컬럼은 **Tokenization/Hashing** 후 저장, 또는 **LF Cell-Level 권한**(Lake Formation 고급)
- 감사: **CloudTrail**(Redshift API/STS), **S3 서버 액세스 로그**, **LF Audit**(사용 시)

---

## 11) 트러블슈팅 FAQ

**Q. `Permission denied` / `Access Denied`**  
A. IAM Role/S3 정책/Glue 권한/Lake Formation 권한 모두 점검. LF 사용 시 Redshift 프린시펄에 **SELECT/Describe** 부여 필요.

**Q. 쿼리가 느리다(스캔 바이트 과다)**  
A. 포맷=Parquet/ORC, 파일 크기 128~512MB, 파티션 프루닝/프로젝션 적용, `SELECT *` 지양, 필터 적용 재검토.

**Q. 파티션이 인식되지 않는다**  
A. 크롤러 재실행 또는 `ALTER TABLE ADD PARTITION …`, 프로젝션 설정/템플릿/포맷 확인.

**Q. 소형 파일 수백만 개**  
A. Glue/Spark(EMR)로 **Compaction** 배치 수행 → 큰 파일로 합치기.

---

## 12) IaC 스니펫(CloudFormation/Terraform)

### 12.1 CloudFormation(외부 스키마는 SQL로 생성)
```yaml
Resources:
  SpectrumRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
          - Effect: Allow
            Principal: { Service: redshift.amazonaws.com }
            Action: sts:AssumeRole
      Policies:
        - PolicyName: SpectrumAccess
          PolicyDocument:
            Statement:
              - Effect: Allow
                Action: [ "s3:GetObject", "s3:ListBucket" ]
                Resource: [ "arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*" ]
              - Effect: Allow
                Action: [ "glue:GetDatabase", "glue:GetTable", "glue:GetPartitions" ]
                Resource: "*"
```

### 12.2 Terraform(역할)
```hcl
resource "aws_iam_role" "spectrum" {
  name = "MySpectrumRole"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{ Effect = "Allow", Principal = { Service = "redshift.amazonaws.com" }, Action = "sts:AssumeRole" }]
  })
}

resource "aws_iam_role_policy" "spectrum_policy" {
  role = aws_iam_role.spectrum.id
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      { Effect = "Allow", Action = ["s3:ListBucket"], Resource = "arn:aws:s3:::my-bucket" },
      { Effect = "Allow", Action = ["s3:GetObject"],  Resource = "arn:aws:s3:::my-bucket/*" },
      { Effect = "Allow", Action = ["glue:GetDatabase","glue:GetTable","glue:GetPartitions"], Resource="*" }
    ]
  })
}
```

---

## 13) 고급: Federated Query/Athena/LF와의 공존

- **Athena** 와 **Redshift Spectrum** 은 **Glue 카탈로그**를 **공유**할 수 있다 → 동일 S3/스키마를 두 엔진에서 소비
- **Federated Query**(RDS/Aurora 등 외부 DB) + Spectrum를 조합하면 **데이터 레이크/소스 DB/Redshift 내부**를 아우르는 **전체 SQL 계층** 구축
- **Lake Formation** 으로 **세분 권한/감사**를 중앙화

---

## 14) 요약(치트시트)

| 주제 | 권장안 |
|---|---|
| 포맷 | **Parquet+Snappy** |
| 파일 크기 | **128~512MB** |
| 파티션 | 날짜(dt) 우선 + 2차 키(지역/채널) 신중히 |
| 파티션 관리 | **프로젝션** 또는 **크롤러+ALTER** |
| 보안 | IAM 최소권한 + KMS + (LF 사용 시 권한 부여) |
| 비용 | 스캔 바이트 절감(포맷/프루닝/압축) |
| 하이브리드 | 내부(핫)+외부(콜드) 레이어링, 필요 시 UNLOAD/CTAS |
| 관측 | STL/SVL 시스템 뷰 + CloudWatch + S3 로그 |

---

## 부록 A) 실습 전체 SQL 묶음

```sql
-- 1) 외부 스키마
CREATE EXTERNAL SCHEMA spectrum_sales
FROM data catalog
DATABASE 'sales_db'
IAM_ROLE 'arn:aws:iam::123456789012:role/MySpectrumRole'
CREATE EXTERNAL DATABASE IF NOT EXISTS;

-- 2) Parquet 외부 테이블 (파티션)
CREATE EXTERNAL TABLE spectrum_sales.fact_sales_pq (
  sale_id    BIGINT,
  product_id BIGINT,
  user_id    BIGINT,
  amount     DECIMAL(12,2),
  event_ts   TIMESTAMP
)
PARTITIONED BY (dt DATE, region VARCHAR(8))
STORED AS PARQUET
LOCATION 's3://my-bucket/datalake/fact_sales_pq/';

-- 3) 파티션 추가(예시)
ALTER TABLE spectrum_sales.fact_sales_pq
ADD IF NOT EXISTS PARTITION (dt='2025-11-01', region='KR')
LOCATION 's3://my-bucket/datalake/fact_sales_pq/dt=2025-11-01/region=KR/';

-- 4) 파티션 프로젝션(선택)
ALTER TABLE spectrum_sales.fact_sales_pq
SET TABLE PROPERTIES (
  'projection.enabled'='true',
  'projection.dt.type'='date',
  'projection.dt.range'='2024-01-01,2026-12-31',
  'projection.dt.format'='yyyy-MM-dd',
  'projection.region.type'='enum',
  'projection.region.values'='KR,US,JP',
  'storage.location.template'='s3://my-bucket/datalake/fact_sales_pq/dt=${dt}/region=${region}/'
);

-- 5) 조인/집계
CREATE TABLE dim_product (
  product_id   BIGINT DISTKEY,
  product_name VARCHAR(200),
  category     VARCHAR(64)
);

SELECT d.category, SUM(f.amount) AS revenue
FROM spectrum_sales.fact_sales_pq f
JOIN dim_product d ON d.product_id = f.product_id
WHERE f.dt BETWEEN '2025-11-01' AND '2025-11-07'
GROUP BY d.category
ORDER BY revenue DESC;
```

---

## 부록 B) 스캔 바이트·비용 추정 예시(수식)

- 평균 파일 압축 해제 크기: \(S\) (GB)  
- 선택 컬럼 비율: \(c\) (0~1)  
- 프레디케이트 선택도: \(s\) (0~1)  
- 압축율(압축 후/전): \(r\) (0~1, 예: Snappy 0.3~0.6)  
- 쿼리 수: \(Q\), GB 단가: \(p_{\text{GB}}\)

$$
\text{스캔GB}_{\text{쿼리}} \approx S \cdot c \cdot s \cdot r
\quad\Rightarrow\quad
\text{월비용} \approx \left(\sum_{q=1}^{Q} \text{스캔GB}_q \right) \cdot p_{\text{GB}}
$$

- **튜닝 목표**: \(c\downarrow\) (컬럼 프루닝), \(s\downarrow\) (프레디케이트/파티션 프루닝), \(r\downarrow\) (압축↑), \(S\downarrow\) (컴팩션/정제)

---

## 결론

**Redshift Spectrum** 은 **데이터 레이크(S3)** 와 **데이터 웨어하우스(Redshift)** 를 **한 쿼리 계층으로 통합**한다.  
핵심은 **Parquet+압축**, **적정 파일 크기**, **파티션 프루닝(또는 프로젝션)**, **하이브리드 레이어링**(핫=내부, 콜드=외부)이다.  
본 가이드의 스키마/권한/튜닝 체크리스트를 템플릿으로 삼아, **낮은 비용**으로 **대규모 데이터**를 **즉시 분석** 가능한 플랫폼을 구축하자.