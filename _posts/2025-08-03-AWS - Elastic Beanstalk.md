---
layout: post
title: AWS - Elastic Beanstalk
date: 2025-08-03 17:20:23 +0900
category: AWS
---
# Elastic Beanstalk: 앱 배포 자동화

## 0) 왜 Elastic Beanstalk(EB)인가?

Elastic Beanstalk(이하 EB)은 **PaaS 성향의 관리형 배포 프레임워크**입니다.  
코드를 업로드하면 EB가 **EC2 / ALB / Auto Scaling / CloudWatch / IAM** 등을 **템플릿+오케스트레이션**으로 묶어 자동 구성·운영합니다.

- **핵심 가치**
  - 인프라 자동화 + 구성 간소화
  - 멀티 스택 지원(Java/.NET/Node.js/Python/Ruby/Go/PHP/Docker)
  - **배포 전략(롤링/임뮤터블/블루그린)** 내장
  - 헬스체크, 스케일링, 로깅/모니터링 기본 탑재

---

## 1) 아키텍처 구성 요소 상세

### 1.1 개념 계층
- **Application**: 논리적 묶음(여러 Environment, 다수 Version 포함)
- **Application Version**: 업로드된 아티팩트(Zip/WAR/Dockerrun 등)
- **Environment**: 실제 실행 인프라(EC2, ALB, ASG, RDS 등). 타입:
  - **Web Server Environment**: HTTP 트래픽 처리(ALB/NLB/CLB 선택)
  - **Worker Environment**: SQS 기반 비동기 처리(백그라운드 잡)
- **Configuration Template**: 환경 옵션의 스냅샷(재사용 가능)

### 1.2 내부적으로 생성되는 리소스(전형)
- VPC/서브넷(선택), 보안그룹, ALB, Target Group, Auto Scaling Group, Launch Template
- EC2 인스턴스(플랫폼 이미지), CloudWatch Logs, CloudWatch Alarms, IAM Role/Instance Profile
- 선택: RDS(내장 생성 가능하나 **운영은 분리 권장**)

> 팁: 운영에서는 **RDS를 EB 환경 밖에서 별도 생성** → 환경 제거 시 DB 보존.

---

## 2) 플랫폼/런타임 선택과 폴더 구조

### 2.1 표준 플랫폼
- Node.js, Python, Java(Tomcat/Corretto), .NET on Windows/IIS, PHP, Ruby, Go
- Docker (Single / Multi-container on ECS)
- Platform Branch/Version 고정 권장(업데이트 시 릴리스 노트 확인)

### 2.2 애플리케이션 폴더 구조 예시

#### (A) Python Flask
```text
my-flask-app/
├── application.py     # Flask entry
├── requirements.txt
├── .ebextensions/
│   ├── 01_packages.config
│   └── 02_env.config
└── .platform/         # 플랫폼 훅(우분투 기반)
    ├── hooks/
    │   ├── predeploy/00_print_env.sh
    │   └── postdeploy/10_reload_nginx.sh
    └── nginx/conf.d/https-redirect.conf
```

`application.py` 예시:
```python
from flask import Flask
app = Flask(__name__)

@app.route('/health')
def health():
    return 'ok', 200

@app.route('/')
def hello():
    return 'hello beanstalk'
```

#### (B) Node.js Express
```text
my-node-app/
├── app.js
├── package.json
├── .ebextensions/
└── .platform/
```

`app.js` 예시:
```javascript
const express = require('express');
const app = express();
app.get('/health', (_, res) => res.send('ok'));
app.get('/', (_, res) => res.send('hello beanstalk'));
app.listen(process.env.PORT || 8080);
```

#### (C) Docker(Single Container)
```text
my-docker-app/
├── Dockerfile
└── Dockerrun.aws.json        # (대안) EB 구식 단일 컨테이너 정의 파일
```

`Dockerfile` 예시:
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 8080
CMD ["node", "app.js"]
```

---

## 3) EB CLI/콘솔 배포 플로우와 베스트 프랙티스

### 3.1 EB CLI 설치 및 초기화
```bash
pipx install awsebcli  # 또는 pip install awsebcli
eb --version

eb init -p python-3.11 my-flask-app   # 플랫폼/리전 선택
eb create prod-web --single           # 환경 생성(단일 인스턴스)
eb open                                # URL 오픈
eb status                              # 상태 확인
eb deploy                              # 새 버전 배포
eb logs                                # 로그 가져오기
```

> 권장: **프로덕션은 최소 2 AZ, 2+ 인스턴스**로 고가용성 구성.

### 3.2 CI/CD 연동(예: GitHub Actions)
```yaml
name: deploy
on: [push]
jobs:
  eb-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install awsebcli
      - run: eb init -p python-3.11 my-flask-app --region ap-northeast-2
      - run: zip -r app.zip .
      - run: eb deploy prod-web --staged
```

---

## 4) 환경 구성: .ebextensions / .platform 훅 / nginx 커스텀

### 4.1 .ebextensions (YAML)
- 패키지 설치, 파일 템플릿, 서비스 설정, 환경 옵션 세팅 등
- EC2 프로비저닝 단계에서 **CloudFormation과 유사**하게 실행

`01_packages.config` 예:
```yaml
packages:
  yum:
    git: []
    jq: []
```

`02_env.config` 예(오토스케일/헬스체크/ALB):
```yaml
option_settings:
  aws:autoscaling:asg:
    MinSize: 2
    MaxSize: 6
  aws:elasticbeanstalk:environment:process:default:
    HealthCheckPath: /health
  aws:elasticbeanstalk:environment:
    LoadBalancerType: application
  aws:elbv2:listener:443:
    ListenerEnabled: true
    Protocol: HTTPS
    SSLCertificateArns: arn:aws:acm:ap-northeast-2:123456789012:certificate/xxxx
```

> 참고: EB 플랫폼/브랜치마다 사용 가능한 Namespaces와 옵션 키가 다릅니다. 콘솔에서 내보내기/가져오기 기능을 활용해 키를 역추적하면 안전합니다.

### 4.2 .platform hooks (플랫폼 훅)
- 경로: `.platform/hooks/{prebuild,predeploy,postdeploy,...}`  
- 쉘 스크립트로 배포 전후 동작 정의(캐시 클리어, 마이그레이션 등)

`predeploy/00_db_migrate.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail
source /opt/elasticbeanstalk/support/envvars
# 예: Django
python manage.py migrate --noinput
```

### 4.3 nginx 커스텀(HTTPS 리다이렉트/헤더/버퍼)
경로: `.platform/nginx/conf.d/https-redirect.conf`
```nginx
map $http_x_forwarded_proto $redirect_to_https {
    default 0;
    https   0;
    http    1;
}

server {
    listen 80;
    if ($redirect_to_https) {
        return 301 https://$host$request_uri;
    }
}
```

추가 헤더(보안):
```nginx
add_header X-Frame-Options "DENY";
add_header X-Content-Type-Options "nosniff";
add_header Content-Security-Policy "default-src 'self'";
```

---

## 5) 배포 전략(무중단/안전성)과 Blue/Green

### 5.1 전략 비교
| 전략 | 특성 | 다운타임 | 비용 | 비고 |
|---|---|---|---|---|
| All at once | 즉시 교체 | 가능 | 낮음 | 테스트/개발 전용에 적합 |
| Rolling | 배치 단위 순차 | 낮음 | 중간 | 최소 Healthy 비율 고려 |
| Rolling w/ additional batch | 임시 배치 가세 | 매우 낮음 | 중간+ | 용량 여유 필요 |
| Immutable | 새 ASG로 교체 | 매우 낮음 | 높음 | 실패 시 자동 롤백 강력 |
| Blue/Green | 별도 환경 준비 → CNAME 스왑 | 없음 | 높음 | 가장 안전, 롤백 용이 |

### 5.2 Blue/Green 절차(요약)
1) 기존(Blue) 환경과 동일 설정으로 새(Green) 환경 생성  
2) Green 환경에서 검증(헬스체크/연기 테스트)  
3) 콘솔/CLI로 **CNAME Swap**  
```bash
aws elasticbeanstalk swap-environment-cnames \
  --source-environment-name blue-env \
  --destination-environment-name green-env
```
4) 문제가 있으면 되돌리기(롤백은 CNAME 재스왑)

---

## 6) 오토스케일링/캐패시티 계획/헬스체크

### 6.1 용량 추정(대략치)
- 초당 요청 수를 $$R$$, 인스턴스 1대의 안정 처리량을 $$C$$라 하면 필요한 인스턴스 수:
  $$
  N \approx \left\lceil \frac{R}{C} \times \text{safety\_factor} \right\rceil
  $$
  - `safety_factor`는 1.2~2.0 범위(버스트/장애 여지 고려)

### 6.2 스케일링 정책
- 지표 기반: CPU/Network/ALBRequestCountPerTarget/Latency 등
- 예: ALB 타깃당 RPS 200 기준 스케일 업
```yaml
option_settings:
  aws:autoscaling:trigger:
    MeasureName: RequestCountPerTarget
    Statistic: Average
    Unit: Count
    UpperThreshold: 200
    LowerThreshold: 50
    Period: 60
    BreachDuration: 120
```

### 6.3 헬스체크
- **HealthCheckPath**(애플리케이션), HTTP 200 반환 필수
- DB/외부 의존성을 깊게 검사하지 말 것(헬스 엔드포인트는 가볍게)

---

## 7) 보안: 네트워크/IAM/시크릿/HTTPS/WAF

### 7.1 네트워크
- **프라이빗 서브넷**에 인스턴스 배치 + 퍼블릭 서브넷에 ALB
- NAT Gateway 비용 절감: Interface/Gateway **VPC Endpoint** 적극 도입
- 보안그룹: ALB SG → App SG(80/8080), 아웃바운드는 최소화

### 7.2 IAM 권한 경계
- EB 인스턴스 프로파일 최소 권한(CloudWatch Logs/ECR/SSM 필요시)
- S3 배포 버킷 접근 제한(버킷 정책, KMS 암호화)

### 7.3 시크릿 관리
- **AWS Secrets Manager/SSM Parameter Store** → `.ebextensions`나 앱 부팅에서 주입
- 예: 환경변수로 노출하지 말고 런타임 조회/캐시

### 7.4 HTTPS/TLS
- ACM 인증서 연결(ALB 443 Listener 설정), HTTP→HTTPS 리다이렉트
- 최신 TLSPolicy 선택(보안 스코어 향상)

### 7.5 WAF/Shield
- ALB에 **AWS WAF** 연결(공격 패턴/Geo/IP 제한)
- DDoS 완화는 **AWS Shield Standard** 기본 + 고가용성 설계

---

## 8) 로깅/모니터링/트레이싱

### 8.1 애플리케이션 로그
- EB 플랫폼 로그 → **CloudWatch Logs** 자동 스트림
- 로그 보존 기간 설정(예: 7~30일), 지표 필터로 오류율 알람

### 8.2 메트릭/알람
- EC2/ELB/ASG/CPU/메모리(CloudWatch Agent)  
- ALB 5xx 증가/응답지연 증가 알람 → SNS/ChatOps

### 8.3 트레이싱
- **AWS X-Ray** SDK 연동 → 분산 트레이싱 시각화
- 장애/지연 병목 구간 파악

---

## 9) RDS 연동 및 데이터 마이그레이션

### 9.1 권장 패턴
- EB 환경 **외부**에서 RDS 생성(CloudFormation/CDK/Terraform)
- DB 접속 정보는 **Secrets Manager**에 저장 → 앱에서 참조

### 9.2 마이그레이션
- `.platform/hooks/predeploy`에서 `alembic/django manage.py migrate` 등 수행
- 롤백 대비: 마이그레이션 **Backfill/Down 스크립트** 준비

---

## 10) Worker 환경(SQS)과 스케줄링

### 10.1 Worker Environment
- EB가 SQS 폴링 → 메시지를 HTTP로 **/queue** 엔드포인트에 POST
- 백그라운드 작업(이미지 처리/메일 발송/ETL) 분리

Node 예시(간단):
```javascript
const express = require('express');
const app = express();
app.use(express.json());
app.post('/queue', (req, res) => {
  const msg = Buffer.from(req.body, 'base64').toString('utf8');
  console.log('job:', msg);
  // 작업 처리...
  res.status(200).end();
});
app.listen(process.env.PORT || 8080);
```

### 10.2 스케줄링(크론)
- EB **cron.yaml** 지원(플랫폼별 지원 확인)
```yaml
version: 1
cron:
  - name: daily-job
    url: /cron/daily
    schedule: "0 20 * * *"   # UTC 기준
```

---

## 11) Docker로 EB 사용(멀티 컨테이너/ECS 기반)

### 11.1 Dockerrun.aws.json v2 (멀티 컨테이너)
```json
{
  "AWSEBDockerrunVersion": 2,
  "containerDefinitions": [
    {
      "name": "web",
      "image": "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/web:latest",
      "essential": true,
      "memory": 512,
      "portMappings": [{ "hostPort": 80, "containerPort": 8080 }]
    },
    {
      "name": "log-router",
      "image": "public.ecr.aws/aws-observability/aws-for-fluent-bit:latest",
      "essential": false,
      "memory": 128
    }
  ]
}
```
> EB 멀티 컨테이너는 내부적으로 **ECS**를 사용. 복잡한 컨테이너 오케스트레이션이 필요하면 **ECS/EKS**로 전환 고려.

---

## 12) 비용 최적화

- 인스턴스 타입/크기 적정화(프로파일링 → vCPU/메모리 요구 파악)
- **Auto Scaling** 임계치 적절화, 두텁지 않은 배치 전략 선택
- 개발/스테이징은 스케줄러로 야간/주말 중지
- NAT Gateway 요금 절감: VPC Endpoint/S3 직연결 활용
- CloudWatch Logs 보존 일수 단축/필요 스트림만 수집

---

## 13) 트러블슈팅 체크리스트

- **헬스체크 실패**: HealthCheckPath 응답 200 확인, 앱 바인딩 포트(8080) 일치?
- **배포 실패(4xx/5xx)**: EB 이벤트/`/var/log/eb-activity.log`, CloudWatch Logs 확인
- **권한 오류**: 인스턴스 프로파일/EB 서비스 역할의 IAM 정책 점검
- **이미지 Pull 실패(Docker)**: ECR 권한/프라이빗 네트워크 경로/엔드포인트
- **RDS 연결 실패**: SG 상호 참조, 서브넷 가용영역, DNS 해석
- **CPU 100%/메모리 누수**: 프로파일링/리밋 상향/코드 hot path 최적화

---

## 14) 예제: EB + Flask + RDS(Secrets) + HTTPS 리다이렉트

### 14.1 Flask (간략)
```python
import os, boto3, json
from flask import Flask
app = Flask(__name__)

def get_secret(name):
    sm = boto3.client('secretsmanager', region_name='ap-northeast-2')
    return json.loads(sm.get_secret_value(SecretId=name)['SecretString'])

@app.get('/health')
def health():
    return 'ok', 200

@app.get('/')
def home():
    # secret 예: {"host":"...","user":"...","password":"...","db":"..."}
    # db_conn = mysql.connector.connect(**get_secret(os.getenv('DB_SECRET_NAME')))
    return 'hello eb', 200
```

### 14.2 .ebextensions/https.config
```yaml
option_settings:
  aws:elasticbeanstalk:environment:
    LoadBalancerType: application
  aws:elbv2:listener:443:
    ListenerEnabled: true
    Protocol: HTTPS
    SSLCertificateArns: arn:aws:acm:ap-northeast-2:123456789012:certificate/xxxx
```

### 14.3 .platform/nginx/conf.d/https-redirect.conf
```nginx
server {
  listen 80;
  return 301 https://$host$request_uri;
}
```

---

## 15) CloudFormation / CDK로 EB 자동화(요약)

### 15.1 CloudFormation 스니펫(요지)
```yaml
Resources:
  EBApp:
    Type: AWS::ElasticBeanstalk::Application
    Properties:
      ApplicationName: my-eb-app

  EBAppVersion:
    Type: AWS::ElasticBeanstalk::ApplicationVersion
    Properties:
      ApplicationName: !Ref EBApp
      SourceBundle:
        S3Bucket: my-bucket
        S3Key: app.zip

  EBEnv:
    Type: AWS::ElasticBeanstalk::Environment
    Properties:
      ApplicationName: !Ref EBApp
      EnvironmentName: prod-web
      SolutionStackName: "64bit Amazon Linux 2 v5.8.2 running Python 3.11"
      VersionLabel: !Ref EBAppVersion
      OptionSettings:
        - Namespace: aws:autoscaling:asg
          OptionName: MinSize
          Value: '2'
        - Namespace: aws:elasticbeanstalk:environment:process:default
          OptionName: HealthCheckPath
          Value: /health
```

### 15.2 CDK(Cfn 레벨) 예시(TypeScript)
```ts
import * as cdk from 'aws-cdk-lib';
import { CfnApplication, CfnApplicationVersion, CfnEnvironment } from 'aws-cdk-lib/aws-elasticbeanstalk';

export class EbStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string) {
    super(scope, id);

    const app = new CfnApplication(this, 'App', { applicationName: 'my-eb-app' });

    const version = new CfnApplicationVersion(this, 'Ver', {
      applicationName: app.applicationName!,
      sourceBundle: { s3Bucket: 'my-bucket', s3Key: 'app.zip' }
    });

    new CfnEnvironment(this, 'Env', {
      applicationName: app.applicationName!,
      environmentName: 'prod-web',
      solutionStackName: '64bit Amazon Linux 2 v5.8.2 running Python 3.11',
      versionLabel: version.ref,
      optionSettings: [
        { namespace: 'aws:autoscaling:asg', optionName: 'MinSize', value: '2' },
        { namespace: 'aws:elasticbeanstalk:environment:process:default', optionName: 'HealthCheckPath', value: '/health' }
      ]
    }).addDependency(version);
  }
}
```

> 주의: EB는 CDK의 L2 고수준 Construct가 제한적입니다. **Cfn*** 리소스(CFN 직결)로 정의하거나, EB CLI/파이프라인과 혼용합니다.

---

## 16) 체크리스트(운영/보안/신뢰성)

- [ ] 최소 2AZ, MinSize ≥ 2
- [ ] 헬스체크 경로 분리(/health), DB 등 외부의존성 배제
- [ ] HTTPS(ACM) + 보안 헤더 + 최신 TLS 정책
- [ ] 시크릿은 Secrets Manager/SSM으로 관리
- [ ] CloudWatch Logs 보존, 오류율/지연 알람
- [ ] 스케일링 임계치 튜닝, 배포 전략은 Immutable/Blue-Green
- [ ] RDS/Redis 등은 환경 외부에서 관리
- [ ] VPC Endpoint 구성으로 NAT 비용 최적화
- [ ] .platform hooks로 마이그레이션/캐시 워밍/검증 자동화

---

## 17) 요약

- **Elastic Beanstalk = “코드에 집중”**을 가능케 하는 AWS의 PaaS 스타일 배포 자동화.  
- 인프라 생성/운영(EC2/ALB/ASG/CloudWatch/IAM)을 추상화하고, **배포 전략/스케일링/모니터링**까지 일괄 제공.  
- 복잡도가 낮은 **웹/백오피스/API/워커** 워크로드에 특히 유용.  
- **보안/관측/스케일/데이터 분리** 원칙을 적용하면, EB만으로도 **프로덕션 품질**의 운영이 충분히 가능.

---

## 부록 A) 자주 쓰는 EB CLI 명령 요약

```bash
eb init                         # 앱 초기화
eb create <env>                 # 환경 생성
eb deploy [env]                 # 배포
eb status [env]                 # 상태
eb health [env]                 # 헬스 상세
eb logs [env] --all             # 로그
eb config [env]                 # 옵션 조회/수정
eb clone <env>                  # 환경 복제
eb swap [env1] [env2]           # CNAME 스왑(Blue/Green)
eb terminate <env>              # 환경 종료
```

---

## 부록 B) 샘플 .ebextensions 모음

### B.1 CloudWatch Logs 보존 기간
```yaml
Resources:
  CWLogsGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /aws/elasticbeanstalk/my-flask-app
      RetentionInDays: 14
```

### B.2 환경 변수 주입
```yaml
option_settings:
  aws:elasticbeanstalk:application:environment:
    APP_ENV: prod
    DB_SECRET_NAME: myapp/db/prod
```

### B.3 ALB 속성(Idle Timeout 등)
```yaml
option_settings:
  aws:elbv2:loadbalancer:
    IdleTimeout: 120
```

---

## 부록 C) 성능·용량 산식 예시

1) 요청 처리량 기반 인스턴스 수:
$$
N \approx \left\lceil \frac{R_{peak}}{C_{inst}} \cdot \text{SF} \right\rceil
$$
- \( R_{peak} \): 피크 초당 요청
- \( C_{inst} \): 인스턴스당 안정 처리량(RPS)
- \( \text{SF} \): 안전계수(1.2~2.0)

2) 응답시간 목표 기반(리틀의 법칙 근사):
$$
L \approx \lambda \cdot W \quad \Rightarrow \quad
\text{평균 동시 처리 수 } L \le N \cdot K
$$
- \( \lambda \): 초당 도착률
- \( W \): 평균 응답시간
- \( K \): 인스턴스 1대의 동시 처리 한계

위 관계를 바탕으로 **부하 테스트(Locust/k6)**로 \( C_{inst}, K \)를 추정한 뒤, 오토스케일 임계치를 보정합니다.
