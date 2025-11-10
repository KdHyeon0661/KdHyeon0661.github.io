---
layout: post
title: AWS - CloudFormation
date: 2025-08-03 15:20:23 +0900
category: AWS
---
# CloudFormation: 인프라의 코드화(Infrastructure as Code)

## 0) CloudFormation 한눈에 보기

- **템플릿(Template)**: YAML/JSON으로 AWS 리소스를 선언
- **스택(Stack)**: 템플릿의 실행 인스턴스(리소스 묶음)
- **체인지셋(Change Set)**: **배포 전 변경 미리보기**
- **드리프트(Drift) 탐지**: 실환경이 템플릿과 달라졌는지 검사
- **롤백/스택 정책/롤백 트리거**: **안전 장치**로 배포 실패/오작동 방지

---

## 1) 템플릿의 표준 구조와 내장 함수

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 최소 예제
Metadata:
  Owners: PlatformTeam
Parameters:
  EnvType:
    Type: String
    AllowedValues: [dev, stg, prod]
    Default: dev
Mappings:
  RegionMap:
    ap-northeast-2: { NginxImage: 'public.ecr.aws/nginx/nginx:stable' }
Conditions:
  IsProd: !Equals [!Ref EnvType, 'prod']
Resources:
  Bucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
Outputs:
  BucketName:
    Value: !Ref Bucket
    Export:
      Name: !Sub '${AWS::StackName}-BucketName'
```

### 1.1 자주 쓰는 **Intrinsic Functions**

- `!Ref`, `!GetAtt`, `!Sub`, `!Join`, `!Split`, `!Select`, `!If`, `!Equals`, `!FindInMap`, `!ImportValue`, `!Sub`의 변수/매핑 혼합

```yaml
Value: !Sub 'https://${Bucket}.s3.${AWS::Region}.amazonaws.com/'
```

### 1.2 **동적 참조(Dynamic References)** — 민감정보 주입

```yaml
Parameters:
  DbPassword:
    Type: String
    NoEcho: true
Resources:
  SecretString:
    Type: AWS::SSM::Parameter
    Properties:
      Type: String
      Value: '{{resolve:ssm-secure:/prod/db/password:1}}' # 예시
```

- SSM/Secrets Manager의 값을 **평문 없이 참조**(템플릿/이벤트 로그에 노출 최소화)

---

## 2) 스택 생명주기와 안전장치

### 2.1 기본 CLI

```bash
aws cloudformation validate-template --template-body file://main.yml
aws cloudformation create-stack --stack-name app-dev --template-body file://main.yml --parameters ParameterKey=EnvType,ParameterValue=dev --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
aws cloudformation create-change-set --stack-name app-dev --change-set-name next --template-body file://main.yml
aws cloudformation describe-change-set --stack-name app-dev --change-set-name next
aws cloudformation execute-change-set --stack-name app-dev --change-set-name next
aws cloudformation delete-stack --stack-name app-dev
```

### 2.2 **스택 정책(Stack Policy)** — **중요 리소스 보호**

```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "Update:*",
      "Principal": "*",
      "Resource": "*",
      "Condition": { "StringEquals": { "ResourceType": ["AWS::KMS::Key", "AWS::S3::Bucket"] } }
    }
  ]
}
```

- 실수로 핵심 리소스 교체/삭제를 막음. 필요한 경우 **임시로 Overwrite** 후 다시 보호.

### 2.3 **롤백 트리거(Rollback Triggers)** — 배포 중 모니터링 실패 시 자동 롤백

```yaml
RollbackConfiguration:
  RollbackTriggers:
    - Arn: arn:aws:cloudwatch:ap-northeast-2:123456789012:alarm:High5XX
      Type: AWS::CloudWatch::Alarm
  MonitoringTimeInMinutes: 15
```

### 2.4 **드리프트 탐지(Drift Detection)**

```bash
aws cloudformation detect-stack-drift --stack-name app-prod
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id <id>
aws cloudformation describe-stack-resource-drifts --stack-name app-prod
```

---

## 3) **DeletionPolicy/UpdateReplacePolicy**와 데이터 보호

```yaml
Resources:
  LogsBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain          # 스택 삭제해도 데이터 유지
    UpdateReplacePolicy: Retain     # 교체 시에도 유지
  DbTable:
    Type: AWS::DynamoDB::Table
    Properties: { ... }
    DeletionPolicy: Snapshot        # RDS 등은 Snapshot을 자주 사용
```

---

## 4) 조건/매핑으로 **환경별 분기**와 **리전별 값 관리**

```yaml
Parameters:
  EnvType:
    Type: String
    AllowedValues: [dev, stg, prod]
Mappings:
  AmiMap:
    ap-northeast-2: { AL2023: '/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-arm64' }
Conditions:
  IsProd: !Equals [!Ref EnvType, 'prod']
Resources:
  NatGw:
    Type: AWS::EC2::NatGateway
    Condition: IsProd
  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: !Sub '{{resolve:ssm:${AmiParam}}}'
Parameters:
  AmiParam:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64
```

---

## 5) **실전 네트워킹 템플릿** (VPC + 서브넷 + IGW + NAT + RT)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC with 2AZ public/private subnets, NAT in prod only.

Parameters:
  EnvType:
    Type: String
    AllowedValues: [dev, stg, prod]
    Default: dev
  Cidr:
    Type: String
    Default: 10.0.0.0/16

Conditions:
  IsProd: !Equals [!Ref EnvType, 'prod']

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref Cidr
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags: [{ Key: Name, Value: !Sub '${AWS::StackName}-vpc' }]

  IGW:
    Type: AWS::EC2::InternetGateway
  IGWAttach:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties: { VpcId: !Ref VPC, InternetGatewayId: !Ref IGW }

  PublicA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      CidrBlock: 10.0.1.0/24
  PublicB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      MapPublicIpOnLaunch: true
      CidrBlock: 10.0.2.0/24
  PrivateA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: 10.0.101.0/24
  PrivateB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: 10.0.102.0/24

  EIPA:
    Type: AWS::EC2::EIP
    Condition: IsProd
    Properties: { Domain: vpc }
  NAT:
    Type: AWS::EC2::NatGateway
    Condition: IsProd
    Properties:
      AllocationId: !GetAtt EIPA.AllocationId
      SubnetId: !Ref PublicA

  RtPublic:
    Type: AWS::EC2::RouteTable
    Properties: { VpcId: !Ref VPC }
  RtPublicRoute:
    Type: AWS::EC2::Route
    Properties: { RouteTableId: !Ref RtPublic, DestinationCidrBlock: 0.0.0.0/0, GatewayId: !Ref IGW }
  RtPubAssocA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: { SubnetId: !Ref PublicA, RouteTableId: !Ref RtPublic }
  RtPubAssocB:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: { SubnetId: !Ref PublicB, RouteTableId: !Ref RtPublic }

  RtPrivateA:
    Type: AWS::EC2::RouteTable
    Properties: { VpcId: !Ref VPC }
  RtPrivateB:
    Type: AWS::EC2::RouteTable
    Properties: { VpcId: !Ref VPC }

  RtPrvRouteA:
    Type: AWS::EC2::Route
    Condition: IsProd
    Properties: { RouteTableId: !Ref RtPrivateA, DestinationCidrBlock: 0.0.0.0/0, NatGatewayId: !Ref NAT }
  RtPrvAssocA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: { SubnetId: !Ref PrivateA, RouteTableId: !Ref RtPrivateA }

  RtPrvRouteB:
    Type: AWS::EC2::Route
    Condition: IsProd
    Properties: { RouteTableId: !Ref RtPrivateB, DestinationCidrBlock: 0.0.0.0/0, NatGatewayId: !Ref NAT }
  RtPrvAssocB:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: { SubnetId: !Ref PrivateB, RouteTableId: !Ref RtPrivateB }

Outputs:
  VpcId: { Value: !Ref VPC, Export: { Name: !Sub '${AWS::StackName}-VpcId' } }
  PublicSubnets: { Value: !Join [',', [!Ref PublicA, !Ref PublicB]] }
  PrivateSubnets: { Value: !Join [',', [!Ref PrivateA, !Ref PrivateB]] }
```

- **dev/stg**: NAT 제거로 비용 절감, **prod**: NAT 활성화  
- **Export**로 VPC ID 공유(다른 스택에서 `!ImportValue`)

---

## 6) 애플리케이션 계층 예시 — ALB + ASG(LaunchTemplate) + UserData + 신호

**권장 패턴:** `CreationPolicy` + **`cfn-signal`**로 **정상 구동을 확인**하고 AutoRollback.

```yaml
Resources:
  Lt:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        InstanceType: t3.micro
        ImageId: !Ref AmiId
        SecurityGroupIds: [!Ref Sg]
        UserData: !Base64 |
          #!/bin/bash -xe
          yum update -y
          amazon-linux-extras install nginx1 -y
          systemctl enable nginx && systemctl start nginx
          /opt/aws/bin/cfn-signal --success true --region ${AWS::Region} --stack ${AWS::StackName} --resource Asg

  Asg:
    Type: AWS::AutoScaling::AutoScalingGroup
    CreationPolicy:
      ResourceSignal:
        Timeout: PT10M
        Count: 2
    UpdatePolicy:
      AutoScalingRollingUpdate:
        PauseTime: PT10M
        WaitOnResourceSignals: true
        MinInstancesInService: 1
    Properties:
      MinSize: '2'
      MaxSize: '4'
      DesiredCapacity: '2'
      VPCZoneIdentifier: [!Ref PrivateA, !Ref PrivateB]
      LaunchTemplate: { LaunchTemplateId: !Ref Lt, Version: !GetAtt Lt.LatestVersionNumber }
```

> 포인트  
> - 인스턴스 **부팅 완료 후** `cfn-signal`을 보내야 스택이 성공 처리  
> - 롤링 업데이트 시에도 `WaitOnResourceSignals`로 안정성 확보

---

## 7) **Exports/Imports**로 스택 간 느슨한 결합

**네트워크 스택**에서 `VpcId`/`Subnets`를 Export → **앱 스택**에서 Import:

```yaml
Parameters:
  VpcId:
    Type: String
    Default: !ImportValue base-network-VpcId
```

- **교차계정/교차리전**은 `Export/Import` 불가 → **RAM(Resource Access Manager)**, **StackSets** 또는 **파이프라인 배포**로 설계

---

## 8) **Nested Stacks**로 대형 템플릿 분해

루트 템플릿 → `AWS::CloudFormation::Stack` 리소스로 **하위 템플릿 포함**

```yaml
Resources:
  NetworkNested:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/nested/network.yml
      Parameters: { EnvType: !Ref EnvType }
```

- 장점: **템플릿 크기/복잡도 분산**, 재사용, 업데이트 단위 축소

---

## 9) **StackSets** — 멀티계정·멀티리전 일괄 배포

- 조직 OU 대상으로 **IAM/가드레일/표준 로깅 버킷** 등 전사 공통 배포
- 배포 모드: Self-managed vs Service-managed(Organizations 연동)
- 운영 팁: **승인 워크플로우** + **드리프트 감시** + **버전 태깅**

---

## 10) 거버넌스: **CloudFormation Guard**와 **cfn-lint**

### 10.1 Guard 규칙(예)

```bash
# 설치
pip install cfn-lint
cargo install cfn-guard
```

```hcl
# guard.rules
AWS::S3::Bucket Encrypted is true
let sse = Resources.*[ Type == "AWS::S3::Bucket" ].Properties.BucketEncryption.ServerSideEncryptionConfiguration
rule Encrypted when %sse != null { %sse[*].ServerSideEncryptionByDefault.SSEAlgorithm == "aws:kms" }
```

```bash
cfn-guard validate -r guard.rules -d main.yml
cfn-lint main.yml
```

- PR 게이트에 **cfn-lint/Guard**를 붙여 **정책 위반 사전 차단**

---

## 11) 보안 베스트 프랙티스

- S3: **Block Public Access**, 버전·수명주기, **KMS 암호화**
- KMS KeyPolicy 최소화(루트 풀권한 지양), **Key Rotation**
- IAM: **최소 권한(리소스 수준)**, **Boundary/Permissions Policy**로 배포 역할 제한
- VPC: **보안그룹은 허용 규칙만**, NACL은 억제적으로, VPC Flow Logs
- 로깅: **CloudTrail, Config, GuardDuty**, 주요 리소스에 **CloudWatch Alarms**
- 파라미터/시크릿은 **동적 참조**로 주입(로그/이벤트 노출 방지)

---

## 12) 서버리스/매크로/확장

- **AWS SAM(Transform: AWS::Serverless-2016-10-31)**: 간결한 서버리스 템플릿로 컴파일 → CFN  
- **매크로(Macros)**: 템플릿을 배포 전 **커스텀 변환**(예: 공통 태그/보안 자동 주입)
- **Language Extensions**: 반복/조건 간소화 기능(선택적)

SAM 예:

```yaml
Transform: AWS::Serverless-2016-10-31
Resources:
  ApiFn:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: app.handler
      Runtime: nodejs20.x
      Events:
        Api:
          Type: Api
          Properties:
            Path: /items
            Method: get
```

---

## 13) 비용 근사와 최적화 감각

- NAT 게이트웨이, ALB, 데이터 전송, EBS/RDS 스토리지, Lambda GB-초, KMS API 등 **누적 비용**에 주의
- 예) NAT GW 월 비용 근사:  
  $$ \text{NAT 비용} \approx h \cdot c_h + GB_{\text{out}} \cdot c_{gb} $$
  - \(h\): 월 가동 시간, \(c_h\): 시간당 비용, \(GB_{\text{out}}\): 전송량, \(c_{gb}\): GB당 전송 단가
- dev/stg에선 **NAT 삭제 + VPC 엔드포인트** 고려, ALB 공유/스팟 혼합, S3 Infrequent Access 수명주기

---

## 14) 운영 전 점검 체크리스트

- [ ] `cfn-lint`/`Guard` 통과  
- [ ] 변경은 **Change Set**로 검토, **스택 정책** 설정  
- [ ] **RollbackTrigger**로 배포 품질 가드  
- [ ] **DeletionPolicy/UpdateReplacePolicy**로 데이터 보호  
- [ ] **Drift Detection** 주기 실행  
- [ ] **Exports/Imports** 의존도 최소화(버전 계획)  
- [ ] 민감정보 **동적 참조** 사용, 파라미터는 `NoEcho`  
- [ ] 로그/모니터링/알람/LifeCycle 모두 코드화

---

## 15) 통합 예제: **VPC + ALB + ASG + RDS + S3 + CloudWatch 경보 + 스택정책 + 롤백트리거**

> 실제 운영에 쓰는 **축약판**입니다(핵심 패턴 위주). RDS는 Snapshot 보호, ASG는 cfn-signal, ALB 헬스체크, 경보를 롤백 트리거로 연동.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Web tier with ALB/ASG and RDS; guarded by alarms & stack policy.

Parameters:
  EnvType: { Type: String, AllowedValues: [dev, stg, prod], Default: prod }
  DBUser:  { Type: String, Default: appuser }
  DBPass:  { Type: String, NoEcho: true }
  AmiParam:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64

Conditions:
  IsProd: !Equals [!Ref EnvType, 'prod']

Resources:
  LogsBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault: { SSEAlgorithm: aws:kms }

  VPC:
    Type: AWS::EC2::VPC
    Properties: { CidrBlock: 10.10.0.0/16 }

  PubA:  Type: AWS::EC2::Subnet
  PubB:  Type: AWS::EC2::Subnet
  PrvA:  Type: AWS::EC2::Subnet
  PrvB:  Type: AWS::EC2::Subnet
  # ... (서브넷/IGW/NAT/RT는 앞 장 예제와 동일 패턴으로 구성)

  AlbSg:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ALB
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - { IpProtocol: tcp, FromPort: 80, ToPort: 80, CidrIp: 0.0.0.0/0 }

  AsgSg:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: App
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - { IpProtocol: tcp, FromPort: 80, ToPort: 80, SourceSecurityGroupId: !Ref AlbSg }

  Alb:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internet-facing
      Subnets: [!Ref PubA, !Ref PubB]
      SecurityGroups: [!Ref AlbSg]

  Tgt:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      VpcId: !Ref VPC
      Port: 80
      Protocol: HTTP
      HealthCheckPath: /
      TargetType: instance

  Lsn:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref Alb
      Port: 80
      Protocol: HTTP
      DefaultActions: [{ Type: forward, TargetGroupArn: !Ref Tgt }]

  Lt:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: !Ref AmiParam
        InstanceType: t3.micro
        SecurityGroupIds: [!Ref AsgSg]
        UserData: !Base64 |
          #!/bin/bash -xe
          dnf -y install nginx
          systemctl enable nginx && systemctl start nginx
          echo "OK" > /usr/share/nginx/html/index.html
          /opt/aws/bin/cfn-signal --success true --stack ${AWS::StackName} --region ${AWS::Region} --resource Asg

  Asg:
    Type: AWS::AutoScaling::AutoScalingGroup
    CreationPolicy:
      ResourceSignal: { Timeout: PT10M, Count: 2 }
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: 1
        PauseTime: PT10M
        WaitOnResourceSignals: true
    Properties:
      MinSize: '2'
      MaxSize: '4'
      DesiredCapacity: '2'
      VPCZoneIdentifier: [!Ref PrvA, !Ref PrvB]
      TargetGroupARNs: [!Ref Tgt]
      LaunchTemplate: { LaunchTemplateId: !Ref Lt, Version: !GetAtt Lt.LatestVersionNumber }

  DbSg:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: DB
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - { IpProtocol: tcp, FromPort: 5432, ToPort: 5432, SourceSecurityGroupId: !Ref AsgSg }

  DbSubnet:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: app-db
      SubnetIds: [!Ref PrvA, !Ref PrvB]

  Rds:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot
    Properties:
      Engine: postgres
      DBInstanceClass: db.t3.micro
      AllocatedStorage: 20
      MasterUsername: !Ref DBUser
      MasterUserPassword: !Ref DBPass
      DBSubnetGroupName: !Ref DbSubnet
      VPCSecurityGroups: [!Ref DbSg]
      MultiAZ: !If [IsProd, true, false]
      PubliclyAccessible: false

  High5xx:
    Type: AWS::CloudWatch::Alarm
    Properties:
      ActionsEnabled: false
      ComparisonOperator: GreaterThanThreshold
      EvaluationPeriods: 1
      MetricName: HTTPCode_Target_5XX_Count
      Namespace: AWS/ApplicationELB
      Period: 60
      Statistic: Sum
      Threshold: 10
      Dimensions:
        - Name: LoadBalancer
          Value: !Select [1, !Split ['loadbalancer/', !Ref Alb]]
      TreatMissingData: notBreaching

  Stack:
    Type: AWS::CloudFormation::WaitConditionHandle
    Metadata:
      StackPolicy:
        Statement:
          - Effect: Deny
            Action: 'Update:*'
            Principal: '*'
            Resource: '*'
            Condition:
              StringEquals: { ResourceType: ['AWS::S3::Bucket','AWS::RDS::DBInstance'] }

Outputs:
  AlbDNS: { Value: !GetAtt Alb.DNSName }
```

- **RDS**: `DeletionPolicy: Snapshot`로 파괴 안전
- **ASG**: `cfn-signal` + RollingUpdate로 **점진 배포**
- **알람**: `High5xx`를 **RollbackTrigger**에 연결(상단 2.3 예시)
- **스택정책**: S3/RDS 업데이트 제한(메타데이터 또는 별도 적용)

---

## 16) 문제 해결(트러블슈팅) 모음

- **CREATE_FAILED**: 이벤트 탭에서 원인 리소스 확인 → **DependsOn** 또는 IAM 권한/서브넷/SG 상호 참조 검토
- **UPDATE_ROLLBACK_FAILED**: 리커버리 가이드에 따라 수동 정리 후 `continue-update-rollback`
- **템플릿 1MB/리소스 한도**: **Nested Stack**로 분할
- **드리프트 많음**: 수동 변경 지양, **IaC-only 원칙** 확립

---

## 17) CloudFormation vs CDK vs Terraform 간 빠른 선택 가이드

| 항목 | CloudFormation | CDK | Terraform |
|---|---|---|---|
| 형식 | YAML/JSON 선언 | 언어( TS/Py/Java/C# ) | HCL |
| 추상화 | 원형(정확) | 고급 추상화(L2/L3) | Provider 다양 |
| 학습 곡선 | 낮음 | 언어/런타임 필요 | 중간 |
| 멀티클라우드 | X | X | O |
| 추천 | 네이티브/보수적 | 개발자 생산성 | 멀티 환경 |

> CDK는 **결국 CloudFormation 템플릿**으로 컴파일되어 배포됩니다. 조직 정책/거버넌스에 맞게 선택하세요.

---

## 18) 마무리 — 운영형 CloudFormation의 핵심 10계명

1) **Change Set**으로 배포 전 변경 검토  
2) **Stack Policy**로 핵심 리소스 보호  
3) **Rollback Triggers**로 품질 보장  
4) **Deletion/UpdateReplacePolicy**로 데이터 보호  
5) **Drift Detection** 정기 수행  
6) **Dynamic References**로 비밀 관리  
7) **Guard/cfn-lint**를 PR 게이트에  
8) **Nested/StackSets**로 대규모 구조화  
9) **CreationPolicy+cfn-signal**로 실구동 확인  
10) **비용 민감 리소스(NAT/ALB/RDS)**는 **조건/환경 분기**로 제어