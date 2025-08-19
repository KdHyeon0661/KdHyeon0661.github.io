---
layout: post
title: AWS - CloudFormation
date: 2025-08-03 15:20:23 +0900
category: AWS
---
# ☁️ CloudFormation: 인프라의 코드화 (Infrastructure as Code)

## 🔸 개요: CloudFormation이란?

**CloudFormation**은 AWS에서 제공하는 **인프라 자동화 도구**로, 클라우드 리소스를 **템플릿 기반 코드**로 정의하고 배포할 수 있게 해줍니다.  
즉, EC2 인스턴스, VPC, S3 버킷, IAM 역할, RDS 인스턴스 등 AWS 자원을 **코드로 구성하고 자동으로 생성 및 관리**할 수 있습니다.

### ✅ 왜 CloudFormation을 사용하는가?

| 장점 | 설명 |
|------|------|
| **버전 관리** | YAML/JSON 파일로 Git에서 추적 가능 |
| **자동화** | 수동 구성 필요 없이 반복 가능한 배포 |
| **재사용성** | 매개변수(parameter)를 통한 템플릿 재사용 |
| **안정성** | 실패한 경우 롤백(rollback) 지원 |
| **비용 추적** | 생성된 리소스를 스택 단위로 삭제/비용 추적 가능 |

---

## 🔸 기본 구성 요소

### 1. 템플릿 (Template)

CloudFormation의 핵심은 템플릿입니다. 이는 **YAML 또는 JSON** 형식으로 작성되며, 아래와 같은 구조를 가집니다:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: S3 버킷 생성 예제
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-sample-bucket-1234
```

### 2. 스택 (Stack)

템플릿을 실행하면 **스택(stack)** 이 생성됩니다.  
스택은 **하나의 논리적 단위로 묶인 리소스 집합**입니다.  
스택을 생성, 업데이트, 삭제함으로써 관련된 모든 리소스를 한 번에 관리할 수 있습니다.

### 3. 매개변수 (Parameters)

템플릿에서 입력값을 받아 **유연한 배포**가 가능하게 해줍니다.

```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues:
      - t3.micro
      - t3.small
      - t3.medium
```

### 4. 출력 (Outputs)

스택 생성 후 결과값(예: S3 버킷 URL, EC2 Public IP)을 제공할 수 있습니다.

```yaml
Outputs:
  BucketName:
    Description: "생성된 S3 버킷 이름"
    Value: !Ref MyS3Bucket
```

---

## 🔸 자주 사용하는 리소스 예시

- **EC2 인스턴스 생성**
- **VPC 및 서브넷 구성**
- **RDS 인스턴스 자동 생성**
- **ALB + Auto Scaling 연동**
- **IAM 역할, 정책 자동 배포**
- **S3 + CloudFront 연동**

---

## 🔸 고급 기능

### 1. 조건문 (`Conditions`)

환경(예: dev vs prod)에 따라 리소스 생성 여부를 제어할 수 있습니다.

```yaml
Conditions:
  IsProd: !Equals [ !Ref EnvType, "prod" ]

Resources:
  MyDB:
    Type: AWS::RDS::DBInstance
    Condition: IsProd
```

### 2. 매핑 (`Mappings`)

리전이나 환경에 따라 값을 바꿔주는 정적 매핑입니다.

```yaml
Mappings:
  RegionMap:
    us-east-1:
      AMI: ami-0ff8a91507f77f867
    ap-northeast-2:
      AMI: ami-0abcdef1234567890
```

### 3. 내장 함수 (`Intrinsic Functions`)

- `!Ref`, `!GetAtt`, `!Join`, `!Sub`, `!If`, `!Equals`, `!FindInMap` 등

예:

```yaml
Value: !Sub "https://${MyS3Bucket}.s3.amazonaws.com/"
```

---

## 🔸 스택 생명 주기 관리

| 단계 | 설명 |
|------|------|
| **생성 (create-stack)** | 최초 리소스 생성 |
| **업데이트 (update-stack)** | 템플릿 변경 사항 반영 |
| **삭제 (delete-stack)** | 스택과 관련된 모든 리소스 삭제 |
| **변경 세트 (change set)** | 변경 내용 사전 미리보기 지원 |

```bash
aws cloudformation create-stack --stack-name my-stack --template-body file://template.yaml
aws cloudformation update-stack --stack-name my-stack --template-body file://template.yaml
aws cloudformation delete-stack --stack-name my-stack
```

---

## 🔸 실습 예제: EC2 + S3 배포 템플릿

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: EC2와 S3를 생성하는 템플릿

Parameters:
  KeyName:
    Description: EC2 인스턴스에 사용할 키 페어 이름
    Type: AWS::EC2::KeyPair::KeyName

Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket

  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: ami-0c55b159cbfafe1f0 # 예시 AMI
      KeyName: !Ref KeyName
      Tags:
        - Key: Name
          Value: MyEC2
```

---

## 🔸 CloudFormation StackSets: 멀티 계정/리전 배포

**StackSets**는 하나의 템플릿을 여러 계정/리전에 동시에 배포할 수 있게 해줍니다.  
예를 들어, 보안 정책이나 공통 태그 정책을 전사적으로 일괄 배포하는 데 활용합니다.

---

## 🔸 CloudFormation vs 기타 IaC 도구

| 도구 | 특징 |
|------|------|
| **CloudFormation** | AWS 네이티브, 관리형, 가장 널리 사용됨 |
| **Terraform** | 멀티 클라우드 지원, 플러그인 생태계 풍부 |
| **CDK** | 코드(Python/TS/Java 등) 기반 IaC, CloudFormation을 추상화 |

---

## 🧠 마무리

CloudFormation은 AWS 인프라를 **정형화된 템플릿**으로 구성하고 자동으로 관리할 수 있게 해줍니다.  
여러 환경(dev/staging/prod)에서 일관된 인프라를 유지하고, 수동 실수를 줄이며, 변경 사항을 추적하는 데 매우 유용합니다.

CloudFormation은 초기 진입장벽이 있을 수 있지만, 제대로 익히면 AWS 리소스를 안정적이고 확장 가능하게 운영할 수 있는 강력한 무기가 됩니다.

---

## 📚 참고 문서

- [CloudFormation 공식 문서](https://docs.aws.amazon.com/cloudformation/)
- [리소스 유형 참조](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)
- [YAML vs JSON 템플릿 비교](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-formats.html)
