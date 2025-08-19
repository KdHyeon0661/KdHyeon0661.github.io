---
layout: post
title: AWS - AWS Free Tier
date: 2025-08-08 15:20:23 +0900
category: AWS
---
# AWS Free Tier 이해와 활용

AWS Free Tier(프리 티어)는 AWS 서비스를 무료로 체험하고 학습할 수 있도록 제공되는 **제한된 무료 사용량**입니다. 클라우드 환경을 처음 접하는 개발자나 기업이 AWS를 테스트, 프로토타입 개발, PoC(Proof of Concept) 등에 활용할 수 있도록 설계되었습니다.

---

## 1. AWS Free Tier의 구성

AWS Free Tier는 크게 세 가지 유형으로 나눌 수 있습니다.

| 유형 | 설명 | 예시 |
|------|------|------|
| **12개월 무료(12-Month Free)** | 신규 가입 계정 기준 12개월간 무료 사용 가능 | EC2 t2.micro 인스턴스 매월 750시간, S3 스토리지 5GB |
| **항상 무료(Always Free)** | 모든 계정에서 시간 제한 없이 무료 제공 | AWS Lambda 100만 요청/월, DynamoDB 25GB 스토리지 |
| **단기 체험(Trial)** | 특정 서비스의 무료 체험 버전 제공, 기간은 서비스별 상이 | Amazon Inspector 90일 무료 |

---

## 2. 주요 서비스 무료 사용량 예시

| 서비스 | 무료 사용량 (12개월) | 설명 |
|--------|------------------|------|
| **EC2** | 매월 750시간 | Linux, RHEL, SLES, Windows t2.micro 또는 t3.micro 인스턴스 |
| **S3** | 5GB | 표준 스토리지, PUT/GET/LIST 요청 일부 무료 포함 |
| **RDS** | 750시간 | MySQL, PostgreSQL, MariaDB, Oracle BYOL, SQL Server Express |
| **CloudFront** | 50GB 데이터 전송 | 전 세계 CDN 캐시 사용 가능 |
| **Lambda** | 100만 요청 | 항상 무료(Always Free) |
| **DynamoDB** | 25GB 스토리지 | 항상 무료, 25WCU/25RCU 포함 |

---

## 3. Free Tier 활용 전략

### (1) 학습 및 실습 환경 구성
- **EC2**로 개발 서버 구성
- **S3**에 정적 웹사이트 호스팅
- **Lambda + API Gateway**로 서버리스 API 구현

### (2) PoC(Proof of Concept) 진행
- 새로운 아키텍처 테스트
- 데이터 처리 파이프라인 구현 후 성능 검증

### (3) 비용 최적화 연습
- CloudWatch로 사용량 모니터링
- 필요 없는 리소스는 즉시 삭제하여 Free Tier 초과 방지

---

## 4. Free Tier 사용 시 주의사항

1. **무료 한도 초과 시 과금 발생**
   - 예: EC2 t2.micro를 2대 실행하면 750시간 초과 가능
2. **지역(Region)별 사용량 합산**
   - 동일 서비스라도 모든 리전 사용량이 합산되어 한도 계산
3. **12개월 이후 자동 과금**
   - Free Tier 기간이 끝나면 표준 요금 부과
4. **항상 무료 서비스도 한도 있음**
   - 예: Lambda는 100만 요청 이후 과금

---

## 5. Free Tier 사용량 모니터링 방법

### AWS Management Console
- **Billing & Cost Management → Free Tier**
- 서비스별 사용량과 남은 무료 할당량 확인 가능

### AWS CLI 예제
```bash
aws ce get-cost-and-usage \
    --time-period Start=2025-08-01,End=2025-08-10 \
    --granularity MONTHLY \
    --metrics "UsageQuantity" "BlendedCost"
```

---

## 6. 활용 팁

- **리소스 태그(Tag) 관리**  
  태그로 프로젝트별 사용량을 구분해 초과 과금 방지
- **자동 종료 스크립트 작성**  
  일정 시간 사용 후 리소스 자동 종료
- **서버리스 우선 전략**  
  Lambda, S3, API Gateway 등 Always Free 서비스 적극 활용
- **CloudFormation/CDK로 인프라 관리**  
  실습 시 매번 동일 환경을 빠르게 생성하고 종료 가능

---

## 7. 결론

AWS Free Tier는 단순한 "무료 체험"이 아니라, 클라우드 인프라 학습과 아키텍처 실험을 위한 **안전한 테스트 환경**입니다.  
효율적으로 활용하면 **비용 부담 없이** AWS의 핵심 서비스를 깊이 이해하고, 실제 운영 환경에 적용하기 전에 충분히 검증할 수 있습니다.