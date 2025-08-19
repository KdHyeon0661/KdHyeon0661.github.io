---
layout: post
title: AWS - Elastic Beanstalk
date: 2025-08-03 17:20:23 +0900
category: AWS
---
# 🌱 Elastic Beanstalk: 앱 배포 자동화

**Elastic Beanstalk**는 AWS에서 제공하는 *PaaS (Platform as a Service)* 솔루션으로, 애플리케이션을 빠르게 배포하고 관리할 수 있게 해주는 서비스입니다. 사용자는 인프라 설정에 대한 복잡성을 줄이고 코드에 집중할 수 있습니다.

## 🧩 Elastic Beanstalk의 주요 특징

| 기능 | 설명 |
|------|------|
| **자동 프로비저닝** | EC2, ELB, Auto Scaling, RDS 등 필요한 리소스를 자동 생성 |
| **자동 배포 및 롤백** | 애플리케이션 업데이트 시 자동 배포 및 실패 시 롤백 기능 제공 |
| **모니터링** | CloudWatch 연동으로 로그 및 성능 지표 확인 |
| **다양한 언어 지원** | Java, .NET, Node.js, Python, Ruby, Go, Docker 등 |
| **환경 구성 관리** | 구성 템플릿 및 설정 스냅샷 기능 제공 |

## 🛠️ Elastic Beanstalk의 구성 요소

Elastic Beanstalk는 내부적으로 아래와 같은 구성 요소들을 조합하여 애플리케이션을 실행합니다:

### 1. **애플리케이션(Application)**  
- 논리적 그룹: 여러 버전과 환경을 가질 수 있음.

### 2. **애플리케이션 버전(Application Version)**  
- 업로드된 소스 코드(Zip, WAR, Docker 이미지 등)
- Git이나 CI/CD 도구에서 자동 푸시 가능

### 3. **환경(Environment)**  
- 애플리케이션을 실행하는 인프라 (EC2, ELB, RDS 등 포함)
- 환경 종류:
  - **Web Server Environment**
  - **Worker Environment**

### 4. **환경 구성(Environment Configuration)**  
- EC2 인스턴스 유형, Auto Scaling 설정, RDS, 보안 그룹 등
- `.ebextensions`로 커스터마이징 가능

---

## 🚀 배포 흐름

```plaintext
1. 코드 작성 및 압축(zip)
2. Elastic Beanstalk 애플리케이션 및 환경 생성
3. 코드 업로드 및 애플리케이션 버전 생성
4. 배포 및 실행 (Elastic Beanstalk가 자동으로 리소스 구성)
5. 모니터링 및 스케일링
```

예시 CLI:

```bash
# 애플리케이션 생성
eb init -p python-3.8 my-flask-app

# 환경 생성
eb create flask-env

# 배포
eb deploy

# 상태 확인
eb status
```

---

## 📁 폴더 구조 예시

```plaintext
my-flask-app/
├── application.py
├── requirements.txt
├── .elasticbeanstalk/
│   └── config.yml
├── .ebextensions/
│   └── 01_packages.config
```

`.ebextensions`는 인스턴스 설정이나 설치 명령 등을 정의하는 곳입니다.

```yaml
# .ebextensions/01_packages.config
packages:
  yum:
    git: []
```

---

## 📊 모니터링과 로깅

Elastic Beanstalk는 **CloudWatch**와 연동되어 자동으로 애플리케이션의 성능 및 상태를 모니터링합니다.

- CPU 사용률, 네트워크 IO, 디스크 사용량
- 로그 스트리밍 및 검색 (`eb logs` 또는 콘솔 사용)
- 상태 변경 이벤트 기록

---

## ⚙️ 자동 스케일링

Elastic Beanstalk는 Auto Scaling Group을 활용하여 자동으로 인스턴스 수를 조절할 수 있습니다.

- 최소/최대 인스턴스 수 설정
- 트래픽 기반 스케일링
- 헬스 체크 실패 시 인스턴스 자동 교체

```yaml
# .ebextensions/scaling.config
option_settings:
  aws:autoscaling:asg:
    MinSize: 2
    MaxSize: 5
```

---

## 🔄 배포 전략

Elastic Beanstalk는 다양한 배포 방식을 제공합니다:

| 전략 | 설명 |
|------|------|
| All at once | 모든 인스턴스에 동시에 배포 |
| Rolling | 일부 인스턴스씩 순차적으로 배포 |
| Rolling with additional batch | 임시 인스턴스를 추가해 무중단 배포 |
| Immutable | 새 인스턴스 그룹 생성 후 트래픽 전환 |
| Blue/Green | 새로운 환경을 배포한 후 트래픽 전환 (수동 또는 Route 53 전환)

---

## ☁️ RDS 연동

Elastic Beanstalk 환경 생성 시 RDS를 함께 생성하거나 외부에서 별도로 연결할 수 있습니다.

- 환경 제거 시 자동 삭제 여부 주의
- `.ebextensions`에서 DB 초기화 스크립트 가능

---

## 💰 비용 관리 전략

- Auto Scaling으로 리소스 최적화
- EC2 스팟 인스턴스 활용 가능 (약간의 설정 필요)
- 필요하지 않은 환경 종료 후 보존 정책 설정
- 로깅/모니터링 비용 주의 (CloudWatch Logs)

---

## ✅ 사용 사례

- 스타트업이나 중소기업의 빠른 MVP 배포
- CI/CD와 연계하여 자동화된 테스트/배포 파이프라인 구축
- 정적 웹사이트, 백엔드 API 서버, 워커 프로세스 처리 등

---

## 🧪 CI/CD 통합

GitHub Actions, CodePipeline 등과 연계하여 자동 배포 가능:

```yaml
# GitHub Actions 예시
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: eb deploy flask-env
```

---

## 🧠 정리

Elastic Beanstalk는 복잡한 인프라 구성 없이도 빠르게 AWS 위에서 애플리케이션을 배포하고 운영할 수 있게 해주는 강력한 PaaS 서비스입니다. 특히 다음과 같은 장점이 있습니다:

- 인프라 자동화 + 코드 중심 운영
- 다양한 언어 및 프레임워크 지원
- 확장성과 고가용성 내장
- 로그/모니터링 연계

복잡한 설정이 필요 없는 배포를 원하거나 빠르게 AWS 기반 서비스를 시작하려는 개발자에게 매우 적합한 서비스입니다.

---

> 📚 **참고**: Elastic Beanstalk은 내부적으로 EC2, Auto Scaling, ELB, CloudWatch 등 여러 AWS 리소스를 통합하여 관리합니다. 복잡한 설정이 필요할 경우 `.ebextensions` 또는 Terraform, CDK 등 IaC 도구를 병행해서 사용하는 것도 좋은 전략입니다.
