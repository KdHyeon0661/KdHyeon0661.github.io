---
layout: post
title: AWS - Aurora, Auto Scaling, 장애 복구 테스트, RDS 비용 절감 전략
date: 2025-07-23 16:20:23 +0900
category: AWS
---
# 📘 Amazon Aurora, Auto Scaling, 장애 복구 테스트, RDS 비용 절감 전략

---

## 1. Amazon Aurora란 무엇인가?

### ✅ 개요

Amazon Aurora는 Amazon RDS(Relational Database Service)에서 제공하는 고성능의 관계형 데이터베이스 엔진입니다. Aurora는 MySQL 및 PostgreSQL과 호환되며, 해당 오픈소스 데이터베이스의 **속도, 안정성, 보안성**을 크게 개선한 **Amazon의 자체 관리형 데이터베이스 서비스**입니다.

### ✅ 주요 특징

| 항목 | 설명 |
|------|------|
| 고성능 | MySQL 대비 최대 5배, PostgreSQL 대비 최대 3배 빠름 |
| 고가용성 | 6개 복제본을 3개 AZ에 자동 분산 저장 |
| 자동 장애 복구 | 기본 인스턴스 장애 시 자동 failover |
| 복제 기능 | 리드 리플리카 최대 15개까지 생성 가능 |
| 보안 | VPC, KMS 암호화, IAM 통합 가능 |
| 백업 | 지속적인 자동 백업 및 스냅샷 지원 |
| 확장성 | 스토리지는 최대 128TB까지 자동 확장 |

### ✅ Aurora의 두 가지 버전

- **Aurora MySQL-Compatible Edition**
- **Aurora PostgreSQL-Compatible Edition**

SQL 문법, 커넥션 드라이버, ORM 등은 기존의 MySQL/PostgreSQL 도구를 그대로 사용할 수 있습니다.

---

## 2. RDS Auto Scaling

### ✅ 개요

Auto Scaling은 애플리케이션의 워크로드에 따라 **DB 인스턴스 용량을 자동으로 조절**하는 기능입니다. Amazon RDS는 기본적으로 EC2 기반으로 동작하므로 EC2 Auto Scaling처럼 명시적인 Auto Scaling 그룹이 존재하지 않지만, Aurora는 다음과 같은 기능을 통해 Auto Scaling을 제공합니다.

---

### ✅ Aurora의 Auto Scaling 구성 요소

| 구성 요소 | 설명 |
|------------|--------|
| Aurora Replica Auto Scaling | 리드 리플리카 개수를 트래픽에 따라 자동 조절 |
| Aurora Serverless v2 | 프로비저닝 없이 필요에 따라 용량 조정 (초 단위) |
| Aurora Global Database | 전 세계 리전 간 읽기 지연 최소화, 리전 자동 전환 |
| Compute Auto Scaling | (Aurora v2 전용) vCPU/메모리를 초 단위로 조정 가능 |

---

### ✅ Aurora Serverless v2 예시

```bash
# Aurora Serverless v2 활성화 시 트래픽 증가에 따라 다음 단계가 자동으로 수행됨:
# 1. vCPU, 메모리 증가
# 2. 필요 시 자동 스토리지 확장
# 3. 워크로드 감소 시 자동 축소
```

---

## 3. 장애 복구 테스트

### ✅ 개요

장애 복구 테스트는 **고가용성을 보장하는 아키텍처가 실제로 실패 시 자동으로 복구되는지 검증**하는 과정입니다. RDS 및 Aurora는 다음과 같은 장애 복구 전략을 제공합니다.

---

### ✅ 장애 시 처리 흐름

#### [단일 AZ 구성]
- 장애 발생 시: 수동 복구 필요 → 가용성 낮음

#### [다중 AZ 구성]
- RDS: 자동 장애 감지 → 스탠바이 인스턴스로 failover
- Aurora: 기본 인스턴스가 장애 나면 리플리카가 자동 승격

---

### ✅ 테스트 방법

```bash
# RDS 인스턴스를 강제로 재시작
aws rds reboot-db-instance --db-instance-identifier mydbinstance --force-failover

# Aurora 기본 인스턴스 재시작
aws rds reboot-db-instance --db-instance-identifier myaurora-instance --force-failover
```

→ 장애 발생 후 **CloudWatch 및 EventBridge**로 알림을 받을 수 있음  
→ **Application Layer에서 Connection Pool 재연결**이 중요

---

### ✅ 복구 시간 비교

| 엔진 | 복구 시간 |
|------|-----------|
| RDS (Multi-AZ) | 약 1~2분 |
| Aurora | 수초 ~ 1분 미만 |

---

## 4. RDS 비용 절감 전략

### ✅ 주요 과금 항목

| 항목 | 과금 기준 |
|------|------------|
| 인스턴스 사용 | 인스턴스 유형 및 시간 단위 |
| 스토리지 사용 | 스토리지 크기, IOPS 수, 백업 저장 |
| 데이터 전송 | 리전 간, 퍼블릭 인터넷 전송 |
| 백업 | 기본 백업 무료, 추가 보관 비용 발생 |

---

### ✅ 절감 전략 1: 예약 인스턴스 활용

- **Reserved Instance (RI)**: 1년 혹은 3년 약정 시 최대 70% 할인
- 추천 상황:
  - 트래픽 예측 가능
  - 장기 서비스 운영
- CLI 예시:

```bash
aws rds purchase-reserved-db-instances-offering \
  --reserved-db-instances-offering-id rixxx \
  --db-instance-count 1
```

---

### ✅ 절감 전략 2: Aurora Serverless 도입

- 유휴 시간 과금 없음
- vCPU, 메모리를 초 단위로 조정
- 부하가 일정하지 않은 서비스에 적합

---

### ✅ 절감 전략 3: 백업 스토리지 관리

- 기본 백업 보존 기간: 무료
- 스냅샷은 수동 삭제 필요
- 오래된 수동 스냅샷 → 비용 증가 유발

```bash
aws rds delete-db-snapshot --db-snapshot-identifier old-snapshot
```

---

### ✅ 절감 전략 4: 모니터링을 통한 리소스 최적화

- CloudWatch로 CPU/메모리/스토리지 사용률 모니터링
- 평균 CPU가 20% 미만일 경우 다운사이징 고려
- 예시:

```bash
# 현재 인스턴스 유형 변경
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.medium \
  --apply-immediately
```

---

### ✅ 절감 전략 5: 자동 스토리지 축소 불가 주의

- RDS는 스토리지를 자동으로 줄일 수 없음
- 필요 시 스냅샷 → 새 인스턴스로 리스토어 후 축소

---

### ✅ 절감 전략 6: Multi-AZ 불필요 시 해제

- 개발 환경, 테스트 서버 등은 단일 AZ로 구성

---

## ✅ 마무리 요약

| 전략 | 요약 |
|------|------|
| 예약 인스턴스 | 최대 70% 할인 가능 |
| Aurora Serverless | 트래픽 변동 큰 서비스에 적합 |
| 스냅샷 정리 | 스토리지 비용 절감 핵심 |
| 인스턴스 최적화 | 리소스 낭비 방지 |
| 스토리지 관리 | 용량 증가는 자동, 축소는 수동 |
| AZ 구성 관리 | 환경별 구성 분리 필수 |

---

## ✅ 참고 문서

- [Amazon Aurora 공식 문서](https://docs.aws.amazon.com/aurora/)
- [RDS 가격](https://aws.amazon.com/rds/pricing/)
- [Aurora Serverless v2 개요](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless.html)
