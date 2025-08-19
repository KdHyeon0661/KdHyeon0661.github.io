---
layout: post
title: AWS - Redshift Spectrum
date: 2025-07-28 23:20:23 +0900
category: AWS
---
# 🧠 Redshift Spectrum: S3 데이터를 Redshift에서 바로 쿼리하기

## 1. ✅ 개요

**Amazon Redshift Spectrum**은 Amazon Redshift 클러스터 외부에 있는 **Amazon S3의 데이터를 직접 쿼리**할 수 있도록 해주는 기능입니다. Redshift에 로딩하지 않고도 S3에 저장된 구조화된 데이터를 SQL로 분석할 수 있습니다.

### 주요 특징

| 항목 | 설명 |
|------|------|
| 🔗 데이터 위치 | Redshift 외부, Amazon S3 |
| 🗃️ 포맷 지원 | CSV, JSON, Parquet, ORC, Avro 등 |
| 📊 쿼리 엔진 | Redshift 내부 쿼리 엔진과 동일 |
| 📁 저장 형식 | 압축, 열 형식 지원 |
| 🧾 요금 | 처리한 **스캔 바이트 수(GB)** 기준 |

---

## 2. 🧱 구성 요소

Redshift Spectrum은 다음 구성요소로 이루어집니다:

- **Amazon S3**: 데이터를 저장하는 장소
- **Amazon Redshift**: SQL 쿼리를 실행하는 클러스터
- **AWS Glue Data Catalog**: S3 데이터의 스키마 메타데이터를 관리 (외부 테이블 정의)
- **Redshift Spectrum 쿼리 엔진**: S3에서 데이터를 검색하여 Redshift로 전달

---

## 3. 🔄 동작 흐름

1. 사용자가 Redshift에서 `SELECT` 쿼리를 실행
2. 쿼리에 포함된 **외부 테이블**을 Redshift Spectrum이 감지
3. **AWS Glue** 또는 **Redshift 자체의 외부 스키마**를 통해 S3의 파일을 탐색
4. 필요한 블록만 **병렬로 읽기**
5. S3에서 데이터를 읽어 Redshift로 스트리밍
6. Redshift는 결과를 반환

> 🚀 S3 데이터를 Redshift로 **복사하지 않고** 분석 가능!

---

## 4. 🛠️ 실습 예시

### 4-1. S3에 샘플 데이터 업로드

예를 들어 `s3://my-bucket/sales_data/` 경로에 `CSV` 데이터를 저장했다고 가정합니다.

### 4-2. Glue Data Catalog에 테이블 생성

```sql
CREATE EXTERNAL SCHEMA spectrum_schema
FROM data catalog
DATABASE 'my_glue_database'
IAM_ROLE 'arn:aws:iam::123456789012:role/MySpectrumRole'
CREATE EXTERNAL DATABASE IF NOT EXISTS;
```

```sql
CREATE EXTERNAL TABLE spectrum_schema.sales_data (
    sale_id INT,
    product_name STRING,
    amount DECIMAL(10,2),
    sale_date DATE
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION 's3://my-bucket/sales_data/';
```

### 4-3. Redshift에서 Spectrum 쿼리

```sql
SELECT product_name, SUM(amount)
FROM spectrum_schema.sales_data
GROUP BY product_name
ORDER BY SUM(amount) DESC;
```

---

## 5. 📦 Redshift Spectrum vs COPY

| 항목 | Redshift Spectrum | COPY |
|------|-------------------|------|
| 데이터 이동 | 없음 (S3 직접 읽기) | Redshift 내부로 로딩 |
| 속도 | 빠르지만 I/O 많음 | 내부 최적화로 빠름 |
| 비용 | 스캔 바이트 기준 | Redshift 저장 공간 기준 |
| 사용 예 | Ad-hoc 분석, 외부 연동 | 정제된 데이터 적재 |

---

## 6. 💰 요금

- **쿼리된 데이터 양(스캔한 바이트 수 기준)**으로 요금 발생
- 압축, 열 형식(예: Parquet, ORC)을 사용하면 **스캔 양 ↓ → 비용 ↓**
- Glue Data Catalog 사용 시에도 추가 요금 발생 가능

---

## 7. 🔒 권한 구성

- S3 객체 접근 권한
- Glue Catalog 접근 권한
- Redshift에서 사용할 IAM Role이 있어야 함

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:ListBucket",
    "glue:GetTable",
    "glue:GetDatabase"
  ],
  "Resource": "*"
}
```

---

## 8. ✅ 활용 팁

- 💡 **데이터 웨어하우스 + 데이터 레이크** 통합 분석이 가능
- 💡 **Hot / Cold 데이터 분리 전략**에서 유용
- 💡 S3 + Parquet + Glue + Spectrum 조합은 **비용/성능 최적화에 유리**

---

## 9. ⚠️ 주의사항

- 외부 테이블은 `INSERT`, `UPDATE`, `DELETE`가 **불가능** (Read-only)
- 쿼리 성능은 **데이터 포맷, 압축률, 파티셔닝**에 크게 의존
- 비용은 쿼리 최적화 없을 경우 **폭증**할 수 있음

---

## 🔚 결론

Redshift Spectrum은 기존 Redshift를 보완하는 **외부 데이터 레이크 분석** 솔루션으로, **S3 데이터를 분석에 직접 활용**하고자 할 때 매우 유용합니다. 특히 Glue와 함께 쓰면 메타데이터 관리도 간편해지고, Athena와의 스키마 공유도 가능해집니다.

> 💬 S3에 저장된 빅데이터를 빠르게 분석하고 싶은가요? 그럼 Spectrum이 답입니다!