---
layout: post
title: Docker - CI/CD와 Docker
date: 2025-03-24 19:20:23 +0900
category: Docker
---
# CI/CD와 Docker: GitHub Actions · GitLab CI · Jenkins

## 공통 아키텍처와 핵심 용어

### 파이프라인 주요 단계

1.  **소스 체크아웃**: 코드 저장소에서 최신 소스를 가져옵니다.
2.  **정적 검사 및 테스트**: 린팅, 단위 테스트, 통합 테스트를 수행합니다.
3.  **보안 점검**: SCA(소프트웨어 구성 분석), SAST(정적 애플리케이션 보안 테스트), 컨테이너 스캔, SBOM(소프트웨어 명세서) 생성을 포함합니다.
4.  **컨테이너 빌드**: 멀티스테이지 Dockerfile과 buildx를 활용한 효율적인 이미지 빌드 및 멀티 아키텍처 지원.
5.  **태깅 및 푸시**: 빌드된 이미지에 태그를 지정하고 Docker Hub, Harbor, ECR, GHCR 등의 레지스트리로 푸시합니다.
6.  **배포**: SSH, Docker Compose, Kubernetes(Helm/ArgoCD) 등을 통해 목표 환경에 애플리케이션을 배포합니다.
7.  **검증 및 헬스체크**: 배포된 애플리케이션의 정상 동작을 확인하고, 실패 시 자동 롤백 조건을 평가합니다.
8.  **아티팩트 보관 및 리포팅**: JUnit 리포트, SARIF(보안 리포트), SBOM, 테스트 커버리지 결과 등을 저장하고 보고합니다.

### 레지스트리 및 태그 설계

- 레지스트리 이름 형식: `<레지스트리>/<조직>/<애플리케이션>:<태그>`
- 권장 태그 전략:
    - **불변 태그 (Immutable)**: `:git-<짧은커밋해시>`, `:build-<빌드번호>` (고유 식별용)
    - **의미적 버전 태그 (Semantic)**: `:vX.Y.Z` (가독성 및 버전 관리용)
    - **환경 채널 태그 (Mutable)**: `:prod`, `:staging`, `:dev` (자동 프로모션에 사용하는 이동 가능한 태그)
- 배포 시 최종적으로는 **다이제스트(Digest)** 로 이미지를 고정하여 사용하는 것을 권장합니다: `image@sha256:<실제해시>`

---

## 모든 CI에 적용할 Dockerfile 최적화

### 멀티스테이지와 캐시 전략 예시 (Python 웹 애플리케이션)

```dockerfile
# syntax=docker/dockerfile:1.7-labs

FROM python:3.11-alpine AS base
WORKDIR /app
RUN adduser -D app && chown -R app:app /app
USER app

FROM base AS deps
# 의존성 설치 단계를 분리하여 캐시 효율성 극대화 (requirements.lock 파일 사용 권장)
COPY --chown=app:app requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt -t /layer

FROM base AS build
COPY --from=deps /layer /usr/local/lib/python3.11/site-packages
COPY --chown=app:app . .
# 선택 사항: 이 단계에서 테스트 또는 린트 실행 가능
# RUN pytest -q

FROM gcr.io/distroless/python3 AS run
WORKDIR /app
COPY --from=build /app /app
USER nonroot
EXPOSE 8080
CMD ["app/main.py"]
```
- `deps` 스테이지는 의존성만 별도로 처리하여 **캐시 히트율을 극대화**합니다.
- 최종 런타임 이미지는 `distroless`를 사용하여 **공격 표면(Attack Surface)을 최소화**합니다.
- CI 파이프라인에서는 buildx와 `cache-from`/`cache-to` 인자를 활용하여 레이어 캐시를 재사용합니다.

### 빌드 매개변수

- buildx를 사용하여 `platform=linux/amd64,linux/arm64`와 같이 멀티 아키텍처 빌드를 설정할 수 있습니다.
- GitHub Actions 및 GitLab CI와 같은 환경은 대부분 QEMU 에뮬레이션을 자동으로 설정해 줍니다.

---

## GitHub Actions 활용

### 기본 이미지 빌드 및 푸시 파이프라인

{% raw %}
```yaml
# .github/workflows/docker.yml
name: docker-ci
on:
  push:
    branches: [ "main" ]
  pull_request:

permissions:
  contents: read
  packages: write
  id-token: write  # AWS/GCP 등 OIDC 인증 사용 시 필요

jobs:
  build-push:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Set up QEMU
      uses: docker/setup-qemu-action@v3
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    - name: Build & Push (multi-arch + cache)
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        platforms: linux/amd64,linux/arm64
        tags: ${{ secrets.DOCKER_USERNAME }}/myapp:latest,${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.sha }}
        cache-from: type=registry,ref=${{ secrets.DOCKER_USERNAME }}/myapp:buildcache
        cache-to: type=registry,ref=${{ secrets.DOCKER_USERNAME }}/myapp:buildcache,mode=max
```
{% endraw %}

#### 주요 포인트

- `permissions` 블록에서 **OIDC(OpenID Connect)** 사용을 위한 권한(`id-token: write`)을 부여할 수 있습니다. 이를 통해 AWS ECR, GCP GAR 등에 비밀번호 없이 안전하게 접근할 수 있습니다.
- `cache-from` 및 `cache-to`를 사용하여 **레지스트리에 캐시 이미지**를 저장하고 재사용함으로써 빌드 시간을 단축할 수 있습니다.
- 태그는 이동 가능한 `:latest`와 고유한 `:gitSHA`를 동시에 푸시하여, 이후 환경별 프로모션(예: staging → production)에 활용할 수 있습니다.

### 보안 스캔 및 서명 단계 추가 (Trivy + SBOM + Cosign)

{% raw %}
```yaml
  security:
    runs-on: ubuntu-latest
    needs: build-push
    steps:
    - name: Trivy Scan
      uses: aquasecurity/trivy-action@0.20.0
      with:
        image-ref: ${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.sha }}
        format: 'sarif'
        output: 'trivy.sarif'
        vuln-type: 'os,library'
        severity: 'CRITICAL,HIGH'
        ignore-unfixed: true
    - name: Upload SARIF to Code Scanning
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: trivy.sarif
    - name: Generate SBOM (syft)
      uses: anchore/sbom-action@v0
      with:
        image: ${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.sha }}
        artifact-name: "sbom-${{ github.sha }}.spdx.json"
    - name: Cosign sign (GH OIDC → KMS/Keyless 가능)
      run: |
        cosign sign ${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.sha }} --yes
      env:
        COSIGN_EXPERIMENTAL: 1
```
{% endraw %}

### OIDC를 이용한 AWS ECR 무비밀 로그인

{% raw %}
```yaml
    - name: Configure AWS credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsECR
        aws-region: ap-northeast-2
    - name: Login to ECR
      id: ecr
      uses: aws-actions/amazon-ecr-login@v2
    - name: Build & Push to ECR
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: <ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:${{ github.sha }}
```
{% endraw %}

### SSH를 통한 간단한 배포

{% raw %}
```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: security
    steps:
    - name: SSH deploy
      uses: appleboy/ssh-action@v1.0.0
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USER }}
        key: ${{ secrets.SSH_KEY }}
        script: |
          docker pull $DOCKER_USER/myapp:${GITHUB_SHA}
          docker compose -f /srv/myapp/docker-compose.yml up -d
```
{% endraw %}

### Helm을 이용한 Kubernetes 배포

{% raw %}
```yaml
    - name: Set image in Helm and deploy
      run: |
        helm upgrade --install myapp charts/myapp \
          --set image.repository=${{ secrets.DOCKER_USERNAME }}/myapp \
          --set image.tag=${{ github.sha }}
```
{% endraw %}
- ArgoCD를 사용하는 경우, 이미지 태그 변경을 GitOps 저장소에 반영하면 자동으로 동기화되는 방식으로 전환할 수 있습니다.

---

## GitLab CI/CD 활용

### 기본 `.gitlab-ci.yml` (Docker-in-Docker 방식)

```yaml
stages: [lint, test, build, security, deploy]
variables:
  DOCKER_DRIVER: overlay2
  IMAGE: $CI_REGISTRY_IMAGE
  TAG: $CI_COMMIT_SHORT_SHA

lint:
  stage: lint
  image: python:3.11-alpine
  script:
    - pip install ruff
    - ruff check .

build:
  stage: build
  image: docker:27-dind
  services: [docker:27-dind]
  variables:
    DOCKER_TLS_CERTDIR: ""
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - docker buildx create --use
    - docker buildx build \
        --platform linux/amd64,linux/arm64 \
        -t $IMAGE:latest -t $IMAGE:$TAG \
        --cache-from=type=registry,ref=$IMAGE:buildcache \
        --cache-to=type=registry,ref=$IMAGE:buildcache,mode=max \
        --push .

security:
  stage: security
  image: alpine:3.20
  before_script:
    - apk add --no-cache trivy
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE:$TAG

deploy:
  stage: deploy
  image: alpine:3.20
  script:
    - apk add --no-cache openssh-client
    - ssh -i /root/.ssh/id_rsa -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST \
        "docker pull $IMAGE:$TAG && docker compose -f /srv/myapp/docker-compose.yml up -d"
  when: manual   # 수동 승인 설정
```

#### 주요 포인트

- GitLab은 **내장 컨테이너 레지스트리**를 제공하며, `$CI_REGISTRY_IMAGE` 변수를 통해 자동으로 접근할 수 있습니다.
- `dind`(Docker-in-Docker) 방식은 설정이 간편하나, 성능과 보안 관점에서 **Kaniko**나 **Buildah**를 대안으로 고려할 수 있습니다.

### Kaniko를 이용한 루트리스(Rootless) 빌드

```yaml
build_kaniko:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:latest
    entrypoint: [""]
  script:
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"username\":\"$CI_REGISTRY_USER\",\"password\":\"$CI_REGISTRY_PASSWORD\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor --context $CI_PROJECT_DIR \
        --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA \
        --destination $CI_REGISTRY_IMAGE:latest \
        --snapshotMode=redo \
        --cache=true --cache-repo=$CI_REGISTRY_IMAGE/cache
```

### 환경, 승인 및 보호 브랜치 관리

- `environments:` 키를 사용하여 `staging`, `production` 등의 배포 환경을 정의할 수 있습니다.
- `only`/`except` 또는 `rules:` 구문을 활용하여 **보호된 브랜치**(예: `main`, `production`)에 대해서만 배포(deploy) 단계가 실행되도록 제한할 수 있습니다.
- `when: manual` 또는 `when: delayed`와 `needs:`를 조합하여 수동 승인이나 지연 배포를 구현할 수 있습니다.

---

## Jenkins 활용

### Jenkinsfile (선언적 파이프라인)

```groovy
pipeline {
  agent any
  environment {
    REGISTRY = "registry.example.com"
    APP      = "demo/myapp"
    IMAGE    = "${REGISTRY}/${APP}:${env.BUILD_NUMBER}"
    LATEST   = "${REGISTRY}/${APP}:latest"
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Build') {
      steps {
        sh '''
          docker build -t $IMAGE -t $LATEST .
        '''
      }
    }
    stage('Login & Push') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'reg-creds',
          usernameVariable: 'REG_USER',
          passwordVariable: 'REG_PASS'
        )]) {
          sh '''
            echo "$REG_PASS" | docker login $REGISTRY -u "$REG_USER" --password-stdin
            docker push $IMAGE
            docker push $LATEST
          '''
        }
      }
    }
    stage('Deploy') {
      when { branch 'main' }
      steps {
        sh '''
          ssh -o StrictHostKeyChecking=no deploy@server \
            "docker pull $IMAGE && docker compose -f /srv/myapp/docker-compose.yml up -d"
        '''
      }
    }
  }
  post {
    always { archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true }
    failure { mail to: 'devops@company.com', subject: "Build #${env.BUILD_NUMBER} failed", body: "Check Jenkins." }
  }
}
```

#### 주요 포인트

- Jenkins의 **Credentials** 시스템(`reg-creds`)을 사용하여 레지스트리 비밀번호와 같은 민감정보를 안전하게 관리합니다.
- 빌드 워크로드가 많을 경우, **에이전트 풀**을 구성하거나 **Kubernetes 플러그인**을 사용하여 동적으로 파드(Pod) 에이전트를 생성할 수 있습니다. 예: `agent { kubernetes { yaml """ ...pod spec... """ } }`

### Jenkins와 Kubernetes 에이전트 조합 예시

```groovy
pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: dind
    image: docker:27-dind
    securityContext:
      privileged: true
    args: ["--insecure-registry=registry.example.com"]
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
      """
    }
  }
  stages {
    stage('Build with Kaniko') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor --context $WORKSPACE \
              --destination registry.example.com/demo/myapp:${BUILD_NUMBER}
          '''
        }
      }
    }
  }
}
```

---

## 배포 전략 (단일 서버, Compose, Kubernetes)

### 단일 서버/Docker Compose 배포 예시

```yaml
# /srv/myapp/docker-compose.yml
version: "3.9"
services:
  app:
    image: registry.example.com/demo/myapp:git-abcdef1
    ports: ["80:8080"]
    restart: unless-stopped
    env_file: /srv/myapp/.env
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
```
- CI 파이프라인에서 배포 시 `image:` 필드의 값을 태그 대신 **다이제스트(Digest)** 로 명시적으로 교체하는 방식을 권장합니다:
    - `image: registry.example.com/demo/myapp@sha256:...`

### Kubernetes 배포 전략 (롤링, 블루-그린, 카나리)

- **롤링 업데이트**: `Deployment`의 기본 전략입니다. `maxSurge`와 `maxUnavailable` 값을 조정하여 업데이트 속도와 가용성을 제어할 수 있습니다.
- **블루-그린 배포**: `blue`와 `green` 두 개의 독립된 Deployment와 Service를 준비한 후, 트래픽을 전환하는 방식입니다.
- **카나리 배포**: Ingress Controller(예: NGINX Ingress)나 Service Mesh(예: Istio)의 트래픽 가중치(`weight`) 기능을 활용하여 새 버전으로의 트래픽을 점진적으로 이전합니다.

#### 다이제스트 고정 Kubernetes Deployment 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector: { matchLabels: { app: myapp } }
  template:
    metadata: { labels: { app: myapp } }
    spec:
      containers:
      - name: app
        image: registry.example.com/demo/myapp@sha256:abcd...   # 다이제스트로 고정!
        ports: [{containerPort: 8080}]
        readinessProbe:
          httpGet: { path: /health, port: 8080 }
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

## 테스트, 품질, 보안 통합

### 테스트 및 커버리지

- PR 단계에서 `pytest --junitxml=...`, `go test -json`, `jest --ci --reporters` 등을 실행하고 그 결과 리포트를 CI 시스템에 업로드합니다.
- 테스트 커버리지가 사전에 정의된 기준치를 충족하지 못하면 파이프라인을 실패 처리하는 게이트를 설정할 수 있습니다.

### 린트, 포맷, 정적 분석

- Python: ruff, flake8, black
- JS/TS: eslint, prettier
- 인프라 코드(IaC): `tflint`, `checkov`, `kics`
- Dockerfile: `hadolint`, `dockle`

### 컨테이너 보안: 스캔, 서명, 정책

- **스캔**: `trivy image --exit-code 1 --severity HIGH,CRITICAL` 명령으로 심각한 취약점이 발견되면 빌드를 실패시킵니다.
- **SBOM 생성**: Anchore Syft나 CycloneDX를 사용하여 소프트웨어 명세서를 생성하고 아티팩트로 보관합니다.
- **서명**: Cosign을 이용한 Keyless(OpenID Connect 기반) 서명이나 KMS 키를 이용한 서명을 수행합니다.
- **정책 적용**: OPA/Gatekeeper/Kyverno와 같은 도구를 사용하여 "반드시 서명된 이미지만 배포 허용"과 같은 정책을 클러스터에 적용합니다.

---

## 비밀 관리 (Secrets)

- CI 시스템 내부: GitHub Actions Secrets, GitLab CI Variables, Jenkins Credentials를 활용합니다.
- 런타임 환경: **Docker Secrets**(Swarm 모드), **Kubernetes Secrets**, 혹은 HashiCorp Vault나 클라우드 제공 Secret Manager와 같은 외부 시스템을 연동합니다.
- 클라우드 환경: **OIDC(OpenID Connect)** 를 활용하여 장기적인 자격증명(비밀번호/액세스 키) 없이 임시 보안 토큰으로 리소스에 접근하는 방식을 적극 권장합니다.

---

## 성능 및 비용 최적화

### 빌드 가속화

- Docker buildx와 레지스트리 캐시(`cache-from`/`cache-to`)를 활용합니다.
- Dockerfile을 멀티스테이지로 설계하고, 의존성 설치 레이어를 분리합니다.
- 불필요한 파일이 빌드 컨텍스트에 포함되지 않도록 `.dockerignore` 파일을 정기적으로 점검합니다.
- Harbor의 Proxy Cache Project와 같은 **사설 프록시 캐시 레지스트리**를 구축하여 외부 레지스트리(Docker Hub 등)로의 반복적인 풀(pull) 트래픽과 Rate Limit을 회피합니다.

### 테스트 가속화

- GitHub Actions나 GitLab CI가 제공하는 **서비스 컨테이너** 기능을 활용하여 통합 테스트에 필요한 DB나 메시지 브로커를 쉽게 연결할 수 있습니다.
- **매트릭스 전략**을 사용하여 OS, 언어 버전, 아키텍처 등 다양한 조합의 테스트를 병렬로 실행합니다.
    - 예: `strategy.matrix.python: [3.10, 3.11]`

---

## 배포 후 검증 및 롤백

### 헬스체크 및 스모크 테스트

- 배포 직후 애플리케이션의 `/health` 엔드포인트를 주기적으로 호출하여 정상 응답(2xx)과 응답 시간을 확인합니다.
- 배포된 서비스의 핵심 기능이 정상 동작하는지 확인하는 간단한 **E2E(End-to-End) 스모크 테스트**를 자동화하여 배포 후 즉시 실행합니다.

### 롤백 방법

- Docker Compose 환경: Compose 파일의 `image`를 직전에 성공한 다이제스트로 변경한 후 `docker compose up -d`를 실행합니다.
- Kubernetes 환경: `kubectl rollout undo deployment/myapp` 명령을 실행하거나, GitOps(ArgoCD) 도구를 사용하여 이전 정상 리비전으로 되돌립니다.

---

## 관측성 (Observability)

- **로그**: Loki, ELK 스택, CloudWatch 등을 연동하고, CI 빌드 번호나 릴리스 태그와 같은 메타데이터를 로그에 포함시켜 추적성을 높입니다.
- **지표**: Prometheus와 Grafana를 설정하여 배포 전후의 에러율, 응답 지연 시간, 자원 사용량 변화를 모니터링하고 비교합니다.
- **트레이싱**: OpenTelemetry, Jaeger 등을 도입하여 특정 릴리스 버전에서 발생하는 성능 이슈나 오류를 분산 추적 시스템으로 분석합니다.
- **알림**: Slack, Microsoft Teams 웹훅 또는 일반 웹훅을 설정하여 빌드 성공/실패, 배포 완료, 롤백 발생 등 중요한 이벤트에 대한 알림을 받습니다.

---

## 레지스트리 및 배포 환경 통합 팁

- **Harbor**: RBAC, 자동 보안 스캔, 리텐션 정책, 불변 태그, 복제 기능을 제공합니다. CI 파이프라인에서는 Robot 계정을 사용하여 접근하는 것이 안전하고 편리합니다.
- **ECR/GAR/GHCR**: OIDC를 지원하므로, 장기 비밀번호 대신 임시 토큰으로 안전하게 로그인할 수 있어 보안과 회계 관리가 간편해집니다.
- 이미지 참조는 최종 배포 시 **다이제스트(Digest)** 로 고정하여 사용하고, 태그는 가독성이나 환경 간 프로모션을 위한 목적으로만 사용합니다.

---

## 권장 프로젝트 구조

```
repo/
├─ app/                      # 애플리케이션 소스 코드
├─ charts/myapp/             # Helm 차트 (Kubernetes 배포 명세)
├─ deploy/compose/           # Docker Compose 파일
├─ Dockerfile
├─ .dockerignore
├─ .github/workflows/        # GitHub Actions 워크플로 정의
│   ├─ ci.yml
│   └─ deploy.yml
├─ .gitlab-ci.yml            # GitLab CI 설정 파일 (해당 시)
├─ Jenkinsfile               # Jenkins 파이프라인 정의 (해당 시)
├─ Makefile                  # 로컬 개발 및 Docker 관련 명령어 모음
└─ security/                 # 보안 관련 설정 (정책, 서명 키 등)
```

---

## 문제 해결

| 증상 | 원인 및 해결 방법 |
|---|---|
| `x509: certificate signed by unknown authority` | 사설 레지스트리의 CA 인증서가 클라이언트에 신뢰되지 않음. `/etc/docker/certs.d/<호스트:포트>/ca.crt` 경로에 인증서를 배포하십시오. |
| `denied: requested access to the resource is denied` | 레지스트리 로그인 상태, 사용자 권한, 리포지토리의 프라이빗 설정, 또는 이미지 이름 오타를 확인하십시오. |
| buildx 캐시가 효과가 없음 | `cache-from`/`cache-to`가 참조하는 캐시 이미지의 접근 권한, 태그 존재 여부, 레지스트리 주소를 확인하십시오. |
| DinD(Docker-in-Docker) 빌드 속도가 느림 | Kaniko나 Buildah와 같은 루트리스 빌드 도구로 전환을 고려하십시오. Dockerfile 레이어와 캐시 구성을 최적화하십시오. |
| 배포 후 502 에러 또는 CrashLoopBackOff | 애플리케이션 헬스체크 경로/지연 설정, 컨테이너 리소스 제한, 필요한 환경변수 또는 시크릿 누락을 확인하십시오. 우선 롤백한 후 원인을 분석하십시오. |
| Trivy에서 CRITICAL 취약점이 다수 발견됨 | 베이스 이미지를 최신 버전으로 업데이트하고, 애플리케이션 패키지 버전을 고정하며, Alpine 또는 Slim 베이스 이미지를 사용하여 불필요한 패키지를 제거하십시오. |

---

## 결론

GitHub Actions, GitLab CI, Jenkins는 각각 다른 장점을 가진 강력한 CI/CD 플랫폼입니다. **GitHub Actions**는 통합성과 풍부한 마켓플레이스, OIDC 지원으로 편리함을 제공합니다. **GitLab CI**는 단일 플랫폼 내에서 소스, 레지스트리, 파이프라인을 완벽하게 통합 관리할 수 있는 장점이 있습니다. **Jenkins**는 오랜 역사와 방대한 플러그인 생태계를 바탕으로 높은 수준의 커스터마이제이션과 온프레미스 제어가 가능합니다.

어떤 플랫폼을 선택하든, 성공적인 Docker 기반 CI/CD를 구축하기 위해서는 몇 가지 공통된 핵심 원칙을 적용해야 합니다:
1.  **최적화된 Docker 이미지**: 멀티스테이지 빌드, 캐시 활용, 경량 베이스 이미지로 빌드 시간과 보안을 개선합니다.
2.  **강화된 보안 게이트**: 자동화된 보안 스캔, SBOM 생성, 컨테이너 서명을 파이프라인에 필수 단계로 포함시킵니다.
3.  **안정적인 배포**: 이미지 참조는 불변의 다이제스트로 고정하고, 명확한 롤백 절차와 헬스체크를 통해 배포 안정성을 확보합니다.
4.  **투명한 관측성**: 로그, 메트릭, 트레이싱을 연동하여 배포 전후의 애플리케이션 상태를 지속적으로 모니터링하고 문제를 신속하게 진단합니다.

이러한 원칙을 준수하며 각 플랫폼의 장점을 살린 파이프라인을 설계한다면, 개발 생산성, 배포 신뢰성, 그리고 애플리케이션 보안을 동시에 높일 수 있을 것입니다.