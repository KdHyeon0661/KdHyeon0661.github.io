---
layout: post
title: AWS - Glue + Athena 연동
date: 2025-07-28 20:20:23 +0900
category: AWS
---
# AWS Glue + Athena 연동: S3 분석을 위한 완벽 가이드

## 1. AWS Glue란?

AWS Glue는 완전관리형 **ETL(Extract, Transform, Load)** 서비스입니다. 구조화된 데이터나 반구조화된 데이터를 S3, RDS, Redshift 등 다양한 AWS 데이터 저장소에서 추출해 변환하고, 로딩하는 데 사용됩니다.

특징은 다음과 같습니다:

- **서버리스**: 인프라를 직접 관리할 필요 없음
- **크롤러 기능**: S3에 저장된 파일에서 자동으로 메타데이터를 추출
- **데이터 카탈로그**: 테이블 정의, 스키마 정보 등을 저장

---

## 2. Amazon Athena란?

Athena는 S3에 저장된 데이터를 **SQL로 분석**할 수 있는 서버리스 쿼리 서비스입니다.

- **Presto 엔진** 기반
- Glue 데이터 카탈로그를 메타스토어로 사용
- CSV, JSON, Parquet 등 다양한 포맷 지원
- 비용은 쿼리 스캔한 데이터 용량(GB) 기준

---

## 3. Glue + Athena 아키텍처

이 두 서비스를 연동하면 다음과 같은 구조가 만들어집니다:

```plaintext
[S3] → [Glue Crawler] → [Data Catalog] → [Athena SQL 분석]
```

- **S3**: 원시 로그, CSV, JSON 등 다양한 파일 저장소
- **Glue Crawler**: 스키마를 자동 추출
- **Data Catalog**: Athena가 참조하는 메타데이터 저장소
- **Athena**: SQL로 데이터 분석

---

## 4. Glue 데이터 크롤러로 메타데이터 수집하기

Glue Crawler는 지정된 S3 경로를 탐색하여 테이블을 생성합니다.

### 4.1 Crawler 생성 단계

1. **Glue 콘솔** → Crawler 생성
2. **데이터 소스**: S3 버킷 선택
3. **IAM 역할**: S3와 Glue 접근이 가능한 역할 지정
4. **크롤 주기**: 필요 시 주기적으로 실행되도록 설정
5. **데이터베이스 선택**: 새로 만들거나 기존 선택
6. **테이블 이름** 확인 및 마침

### 4.2 스키마 자동 인식

- CSV → 컬럼 이름/타입 자동 분석
- JSON → Nested 구조까지 분석 가능
- Parquet → 컬럼 기반 압축 포맷으로 분석 효율적

---

## 5. Athena로 Glue 테이블 쿼리하기

1. Athena 콘솔 → `Query Editor`
2. 왼쪽 데이터베이스에서 Glue로 생성된 테이블 확인
3. SQL 실행

예:

```sql
SELECT * FROM logs_table
WHERE status_code = 500
LIMIT 10;
```

- Glue에서 등록된 테이블 메타데이터 기반으로 동작
- S3 경로의 데이터를 자동으로 로드해서 분석

---

## 6. 실습: S3에 저장된 로그 분석

### 6.1 S3에 샘플 로그 업로드

```plaintext
access-logs/
  ├── 2025-08-01.log
  ├── 2025-08-02.log
  └── ...
```

예시 로그:

```
192.168.1.1 - - [01/Aug/2025:10:00:00 +0000] "GET /index.html" 200 512
```

### 6.2 Glue Crawler 설정

- 소스: `s3://my-logs-bucket/access-logs/`
- 포맷: 정규 표현식 또는 custom classifier

### 6.3 Athena에서 쿼리

```sql
SELECT COUNT(*) AS error_count
FROM logs_table
WHERE status_code = 500;
```

---

## 7. Athena 결과 저장 위치 설정

Athena는 쿼리 결과를 S3에 저장합니다.

1. 콘솔 상단 → "Settings"
2. Query result location → S3 버킷 경로 지정

```plaintext
s3://my-athena-query-results/
```

> 설정하지 않으면 쿼리 실행 시 오류가 발생할 수 있습니다.

---

## 8. 주의할 점 및 팁

- **크롤 주기적 실행**: 새로운 파일이 추가되면 크롤러를 다시 실행해야 반영됨
- **S3 파티셔닝**을 사용하면 Athena 쿼리 비용과 시간이 줄어듬
  - 예: `s3://bucket/year=2025/month=08/day=07/`
- **Parquet 포맷**이 CSV보다 성능 우수
- Glue ETL 작업으로 데이터 전처리 후 분석하면 더 정밀한 결과 가능
- Athena는 쿼리 단위로 과금되므로 WHERE 조건문 활용이 중요

---

# ✅ 마무리

Glue + Athena 조합은 **S3에 저장된 데이터를 빠르게 분석**하기에 매우 강력한 도구입니다. Glue가 데이터를 정리하고, Athena가 이를 SQL로 조회하게 구성되며, 완전 서버리스이기 때문에 비용/운영 부담이 적습니다.

이 연동은 다음과 같은 상황에서 특히 유용합니다:

- 로그 분석
- 데이터 레이크 구축
- ETL 처리 후의 데이터 웨어하우징