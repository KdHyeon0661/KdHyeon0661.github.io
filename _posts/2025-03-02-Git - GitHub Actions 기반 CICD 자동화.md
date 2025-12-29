---
layout: post
title: Git - GitHub Actions 기반 CI/CD 자동화
date: 2025-03-02 20:20:23 +0900
category: Git
---
# GitHub Actions CI/CD 완전 가이드

## GitHub Actions 기본 개념 이해하기

### 핵심 구성 요소
GitHub Actions는 자동화 워크플로를 구축하기 위한 강력한 플랫폼입니다. 기본적인 구성 요소들을 이해하는 것이 중요합니다:

**워크플로 (Workflow)**
- 자동화된 프로세스를 정의하는 YAML 파일
- `.github/workflows/` 디렉토리에 저장
- 다양한 이벤트(푸시, PR, 스케줄 등)에 의해 트리거됨

**잡 (Job)**
- 워크플로 내의 실행 단위
- 병렬 또는 순차적으로 실행 가능
- 각 잡은 독립된 러너 환경에서 실행됨

**스텝 (Step)**
- 잡 내부의 개별 작업 단위
- 액션 사용(`uses:`) 또는 스크립트 실행(`run:`)으로 구성

**러너 (Runner)**
- 잡을 실행하는 머신
- GitHub 호스팅(ubuntu-latest 등) 또는 셀프 호스팅

### 프로젝트 구조 예시
```
.github/
└── workflows/
    ├── ci.yml            # 지속적 통합
    ├── deploy.yml        # 배포 자동화
    ├── nightly.yml       # 정기 작업
    └── security.yml      # 보안 검사
```

---

## 기본 CI 설정: Node.js 프로젝트

### 최소한의 CI 워크플로
```yaml
name: CI

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linter
      run: npm run lint --if-present
    
    - name: Run tests
      run: npm test
```

### 고급 CI: 매트릭스 테스트 및 보고서

{% raw %}
```yaml
name: Advanced CI

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]

jobs:
  test:
    name: Test on Node ${{ matrix.node-version }}
    runs-on: ${{ matrix.os }}
    
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [18, 20, 22]
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run tests with coverage
      run: |
        npm test
        npm run test:coverage --if-present
    
    - name: Upload test results
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: test-results-${{ matrix.os }}-${{ matrix.node-version }}
        path: |
          coverage/
          test-results/
        retention-days: 7
```
{% endraw %}

---

## 다양한 프로그래밍 언어 지원

### Python 프로젝트 CI
```yaml
name: Python CI

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'
        cache: 'pip'
    
    - name: Install dependencies
      run: pip install -r requirements.txt
    
    - name: Run tests
      run: pytest --junitxml=junit.xml
    
    - name: Upload test report
      uses: actions/upload-artifact@v4
      with:
        name: pytest-report
        path: junit.xml
```

### Java/Gradle 프로젝트 CI
```yaml
name: Java CI

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Java
      uses: actions/setup-java@v4
      with:
        distribution: 'temurin'
        java-version: '21'
        cache: 'gradle'
    
    - name: Build and test
      run: ./gradlew build test
    
    - name: Upload build artifacts
      uses: actions/upload-artifact@v4
      with:
        name: build-artifacts
        path: build/libs/*.jar
```

---

## 캐시 전략 최적화

### npm/yarn 캐시 설정

{% raw %}
```yaml
- name: Cache node_modules
  uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      **/node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```
{% endraw %}

### Docker 빌드 캐시

{% raw %}
```yaml
name: Docker Build with Cache

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      packages: write
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Login to GitHub Container Registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ghcr.io/${{ github.repository }}:latest
          ghcr.io/${{ github.repository }}:${{ github.sha }}
        cache-from: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache
        cache-to: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache,mode=max
```
{% endraw %}

---

## 코드 품질 및 보안 검사

### 정적 분석 및 보안 검사
```yaml
name: Code Quality and Security

on:
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 2 * * 1'  # 매주 월요일 02:00 UTC

jobs:
  security-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    
    steps:
    - uses: actions/checkout@v4
    
    - name: CodeQL Analysis
      uses: github/codeql-action/init@v3
      with:
        languages: javascript, python
    
    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v3
    
    - name: Run SAST tools
      run: |
        npm audit --audit-level=high
        # 또는 언어별 보안 도구 실행

  quality:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linter
      run: npm run lint
    
    - name: Check code formatting
      run: npm run format:check
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v4
      with:
        files: ./coverage/lcov.info
```

---

## 배포 자동화 전략

### AWS에 OIDC를 이용한 안전한 배포
```yaml
name: Deploy to AWS

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    permissions:
      id-token: write
      contents: read
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
        aws-region: ap-northeast-2
    
    - name: Build application
      run: |
        npm ci
        npm run build
    
    - name: Deploy to S3
      run: |
        aws s3 sync dist/ s3://my-website-bucket/ --delete
    
    - name: Invalidate CloudFront cache
      run: |
        aws cloudfront create-invalidation \
          --distribution-id ABC123DEF456 \
          --paths "/*"
```

### Firebase 배포
```yaml
name: Deploy to Firebase

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
    
    - name: Install Firebase CLI
      run: npm install -g firebase-tools
    
    - name: Deploy to Firebase
      run: firebase deploy --only hosting
      env:
        FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
```

---

## 모노레포 프로젝트 최적화

### 변경된 부분만 테스트하기

{% raw %}
```yaml
name: Monorepo CI

on:
  pull_request:
    branches: [ main ]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.filter.outputs.frontend }}
      backend: ${{ steps.filter.outputs.backend }}
      shared: ${{ steps.filter.outputs.shared }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Detect changed paths
      id: filter
      uses: dorny/paths-filter@v3
      with:
        filters: |
          frontend:
            - 'packages/frontend/**'
            - 'shared/**'
          backend:
            - 'packages/backend/**'
            - 'shared/**'
          shared:
            - 'shared/**'

  test-frontend:
    needs: detect-changes
    if: needs.detect-changes.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    - run: echo "Testing frontend..."
    # 실제 테스트 스텝 구현

  test-backend:
    needs: detect-changes
    if: needs.detect-changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    - run: echo "Testing backend..."
    # 실제 테스트 스텝 구현
```
{% endraw %}

---

## 고급 기능 활용

### 동시성 제어 및 자동 취소

{% raw %}
```yaml
name: CI with Concurrency Control

on:
  pull_request:
    branches: [ main ]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    
    steps:
    - uses: actions/checkout@v4
    - run: npm ci && npm test
```
{% endraw %}

### 재사용 가능한 워크플로

{% raw %}
```yaml
# .github/workflows/reusable-test.yml
name: Reusable Test Workflow

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
      working-directory:
        required: false
        type: string
        default: '.'

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
      working-directory: ${{ inputs.working-directory }}
    
    - name: Run tests
      run: npm test
      working-directory: ${{ inputs.working-directory }}
```
{% raw %}

### 수동 트리거 및 입력 파라미터

{% raw %}
```yaml
name: Manual Deployment

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      version:
        description: 'Application version'
        required: false
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to ${{ inputs.environment }}
      run: |
        echo "Deploying version ${{ inputs.version || 'latest' }} to ${{ inputs.environment }}"
        # 실제 배포 로직
```
{% endraw %}

---

## 알림 및 모니터링

### 실패 알림 설정

{% raw %}
```yaml
name: CI with Notifications

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    - run: npm ci && npm test

  notify-on-failure:
    needs: test
    if: failure()
    runs-on: ubuntu-latest
    
    steps:
    - name: Send Slack notification
      uses: slackapi/slack-github-action@v2.0.0
      with:
        channel-id: 'C12345678'
        slack-message: |
          CI failed for ${{ github.repository }}:
          - Workflow: ${{ github.workflow }}
          - Branch: ${{ github.ref }}
          - Commit: ${{ github.sha }}
          - Run: ${{ github.run_id }}
      env:
        SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```
{% endraw %}

---

## 셀프 호스팅 러너 설정

### 셀프 호스팅 러너 사용

```yaml
name: Build on Self-hosted Runner

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: [self-hosted, linux, x64]
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Build with custom hardware
      run: |
        # 특수 하드웨어(GPU 등) 활용
        nvidia-smi
        make build
```

**셀프 호스팅 러너 보안 권장사항:**
- 독립된 네트워크 환경 구성
- 정기적인 보안 패치 적용
- 최소 권한 원칙 준수
- 작업 완료 후 환경 자동 정리

---

## 종합적인 CI/CD 파이프라인 예제

{% raw %}
```yaml
name: Complete CI/CD Pipeline

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]
    tags: [ 'v*.*.*' ]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  quality-checks:
    name: Quality Checks
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Lint
      run: npm run lint
    
    - name: Test with coverage
      run: npm run test:coverage
    
    - name: Upload coverage
      uses: codecov/codecov-action@v4
      with:
        files: ./coverage/lcov.info

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Run CodeQL
      uses: github/codeql-action/init@v3
      with:
        languages: javascript
    
    - name: Analyze
      uses: github/codeql-action/analyze@v3

  build-and-push:
    name: Build and Push Container
    needs: [quality-checks, security-scan]
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    permissions:
      packages: write
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=sha,prefix={{branch}}-
    
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: ${{ github.event_name != 'pull_request' }}
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
        cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max

  deploy-staging:
    name: Deploy to Staging
    needs: build-and-push
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
    - name: Deploy to Staging
      run: |
        echo "Deploying to staging environment..."
        # 실제 배포 스크립트

  deploy-production:
    name: Deploy to Production
    needs: deploy-staging
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - name: Create GitHub Release
      uses: softprops/action-gh-release@v2
      with:
        generate_release_notes: true
    
    - name: Deploy to Production
      run: |
        echo "Deploying version ${{ github.ref_name }} to production..."
        # 실제 배포 스크립트
```
{% endraw %}

---

## 모범 사례 및 운영 팁

### 1. 성능 최적화
- **캐시 활용**: 의존성, Docker 레이어, 빌드 아티팩트 캐시
- **병렬 실행**: 독립적인 작업은 병렬로 실행
- **선택적 실행**: 변경된 부분만 테스트(모노레포)
- **얕은 체크아웃**: 필요시만 전체 히스토리 다운로드

### 2. 보안 강화
- **최소 권한 원칙**: 필요한 권한만 부여
- **시크릿 관리**: 환경별 시크릿 분리
- **OIDC 사용**: 장기 인증 키 대신 임시 자격 증명
- **코드 스캔**: 정적 분석 도구 통합

### 3. 유지보수성
- **재사용 가능 워크플로**: 공통 로직 추상화
- **명확한 네이밍**: 작업과 스텝 이름을 의미 있게
- **문서화**: 워크플로 목적과 사용법 주석 추가
- **버전 고정**: 액션 버전 명시적 지정

### 4. 모니터링 및 디버깅
- **상세 로그**: 중요한 정보 로깅
- **아티팩트 저장**: 테스트 결과, 로그, 빌드 산출물
- **알림 설정**: 실패 시 즉시 알림
- **메트릭 수집**: 실행 시간, 성공률 모니터링

### 5. 비용 관리
- **셀프 호스팅 러너**: 대규모 빌드에 경제적
- **캐시 전략**: 반복 작업 비용 절감
- **타임아웃 설정**: 무한 루프 방지
- **리소스 최적화**: 필요 이상의 리소스 사용 방지

---

## 문제 해결 가이드

### 일반적인 문제 및 해결책

**문제 1: 워크플로가 너무 느림**
- **해결**: 캐시 구현, 병렬 실행, 불필요한 단계 제거

**문제 2: 권한 오류**
- **해결**: `permissions` 설정 확인, 필요한 권한 추가

**문제 3: 시크릿 노출**
- **해결**: 로그 마스킹, 적절한 시크릿 관리

**문제 4: 환경 차이로 인한 실패**
- **해결**: 컨테이너 사용, 환경 변수 명시적 설정

**문제 5: 플랫킹 현상**
- **해결**: 동시성 제어, 작업 취소 설정

### 디버깅 도구 및 기법

{% raw %}
```yaml
- name: Debug information
  run: |
    echo "Event: ${{ github.event_name }}"
    echo "Ref: ${{ github.ref }}"
    echo "SHA: ${{ github.sha }}"
    echo "Runner OS: ${{ runner.os }}"
    ls -la
```
{% endraw %}

---

## 결론

GitHub Actions는 현대적인 소프트웨어 개발 라이프사이클을 자동화하는 강력한 도구입니다. 효과적인 CI/CD 파이프라인을 구축하기 위해서는 몇 가지 핵심 원칙을 이해하고 적용하는 것이 중요합니다.

### 성공적인 CI/CD 구현을 위한 핵심 원칙

1. **점진적 개선**: 완벽한 파이프라인을 한 번에 구축하려 하지 말고, 작은 단계부터 시작하여 점진적으로 개선하세요.

2. **팀 협업**: CI/CD는 개인의 작업이 아닌 팀의 워크플로우입니다. 팀원들과 협력하여 표준과 모범 사례를 정립하세요.

3. **보안 우선**: 자동화의 편리함이 보안을 희생시키지 않도록 하세요. 최소 권한 원칙을 준수하고, 민감 정보를 안전하게 관리하세요.

4. **모니터링과 피드백**: 파이프라인을 구축한 후 방치하지 마세요. 지속적으로 모니터링하고, 문제점을 개선하며, 팀의 피드백을 반영하세요.

5. **유연성 유지**: 프로젝트의 성장과 변화에 따라 파이프라인도 진화해야 합니다. 과도하게 복잡한 설정은 유지보수를 어렵게 만듭니다.

### 마지막 조언

GitHub Actions를 효과적으로 사용하는 비결은 단순함과 일관성에 있습니다. 처음부터 모든 고급 기능을 구현하려 하지 말고, 프로젝트의 실제 필요에 맞는 최소한의 기능부터 시작하세요. 시간이 지나면서 프로젝트가 성장하고 팀의 요구사항이 변화함에 따라 파이프라인도 함께 발전시켜 나가면 됩니다.

기억하세요, 가장 좋은 CI/CD 파이프라인은 팀의 생산성을 높이고, 코드 품질을 보장하며, 배포 과정을 신뢰할 수 있게 만드는 파이프라인입니다. 기술적 완벽함보다 실제 비즈니스 가치를 창출하는 데 집중하세요.

GitHub Actions는 단순한 자동화 도구를 넘어 팀의 개발 문화와 프로세스를 개선하는 기회입니다. 이 기회를 활용하여 더 나은 소프트웨어를 더 빠르고 안전하게 제공하는 여정을 시작해 보세요.