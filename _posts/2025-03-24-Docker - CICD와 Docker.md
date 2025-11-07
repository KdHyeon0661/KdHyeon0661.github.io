---
layout: post
title: Docker - CI/CD와 Docker
date: 2025-03-24 19:20:23 +0900
category: Docker
---
# CI/CD와 Docker: GitHub Actions · GitLab CI · Jenkins

## 0. 공통 아키텍처와 용어 정리

### 0.1 파이프라인 표준 단계
1) **소스 체크아웃**  
2) **정적 검사/테스트**(단위·통합, 린트)  
3) **보안**(SCA/SAST/Container Scan/SBOM)  
4) **컨테이너 빌드**(멀티스테이지, buildx 캐시, 멀티아키)  
5) **태깅/푸시**(Docker Hub/Harbor/ECR/GHCR)  
6) **배포**(SSH/Compose/K8s/ArgoCD)  
7) **검증/헬스체크**(자동 롤백 조건 포함)  
8) **아티팩트 보관/리포팅**(JUnit, SARIF, SBOM, 커버리지)

### 0.2 레지스트리/태그 설계
- 레지스트리 네임: `<REG>/<ORG>/<APP>:<TAG>`
- 권장 태그 세트:  
  - **불변**: `:git-<shortSHA>`, `:build-<num>`  
  - **가독**: `:vX.Y.Z`(SemVer)  
  - **채널**: `:prod`, `:staging`, `:dev`(움직이는 태그 → 자동 프로모션)
- 배포 시 실제 이미지는 **다이제스트 고정** 사용을 권장: `image@sha256:<digest>`

---

## 1. 공통 Dockerfile 최적화(모든 CI에 적용)

### 1.1 멀티스테이지 + 캐시 전략 예시(파이썬 웹)
```dockerfile
# syntax=docker/dockerfile:1.7-labs
FROM python:3.11-alpine AS base
WORKDIR /app
RUN adduser -D app && chown -R app:app /app
USER app

FROM base AS deps
# 의존성만 별도 캐시 (requirements.lock 권장)
COPY --chown=app:app requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt -t /layer

FROM base AS build
COPY --from=deps /layer /usr/local/lib/python3.11/site-packages
COPY --chown=app:app . .
# 선택: 테스트/린트 수행
# RUN pytest -q

FROM gcr.io/distroless/python3 AS run
WORKDIR /app
COPY --from=build /app /app
USER nonroot
EXPOSE 8080
CMD ["app/main.py"]
```
- `deps` 레이어는 의존성만 담아 **캐시 효율 상승**.
- 최종 런타임은 `distroless`로 **공격 표면 최소화**.
- CI에서 buildx와 `cache-from/cache-to`로 레이어 재사용.

### 1.2 멀티아키(amd64/arm64) 빌드 매개변수
- buildx: `platform=linux/amd64,linux/arm64`
- QEMU 에뮬레이션 자동 세팅(액션/에이전트 제공).

---

## 2. GitHub Actions

### 2.1 최소 파이프라인(빌드+푸시)
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
  id-token: write  # OIDC(예: AWS/GCP/OCI) 사용 시 필요

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

#### 포인트
- `permissions`에서 **OIDC 사용 가능**(ECR/GAR 무비밀 인증).
- `cache-from/to`를 **레지스트리 캐시 이미지**로 구성 → 빌드 시간 단축.
- 태그는 `:latest` + `:gitSHA`를 동시에 푸시(환경 프로모션에 활용).

### 2.2 보안 스캔/품질 게이트 추가(Trivy + SBOM + Cosign)
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

### 2.3 OIDC로 AWS ECR에 로그인(무비밀)
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

### 2.4 배포(단일 서버/Compose) — SSH 액션
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

### 2.5 배포(Kubernetes) — kubectl/Helm/ArgoCD
```yaml
    - name: Set image in Helm and deploy
      run: |
        helm upgrade --install myapp charts/myapp \
          --set image.repository=${{ secrets.DOCKER_USERNAME }}/myapp \
          --set image.tag=${{ github.sha }}
```
- ArgoCD를 쓰면 “이미지 태그 변경 → GitOps 레포 자동 싱크”로 전환 가능.

---

## 3. GitLab CI/CD

### 3.1 기본 `.gitlab-ci.yml`(Docker-in-Docker)
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
  when: manual   # 수동 승인
```

#### 포인트
- GitLab은 **컨테이너 레지스트리 내장** → `$CI_REGISTRY_IMAGE` 자동 할당.
- `dind`는 간편하나, 고성능/보안 관점에선 **Kaniko/Buildah** 대안 고려.

### 3.2 Kaniko로 루트리스/고성능 빌드
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

### 3.3 환경/승인/보호 브랜치
- `environments:` 키로 `staging`·`production` 정의.
- `only/except` 혹은 `rules:`로 **보호 브랜치**에만 deploy stage 허용.
- `manual`/`when: delayed`/`needs:`로 승인/스케줄링 구현.

---

## 4. Jenkins

### 4.1 Jenkinsfile(Declarative Pipeline)
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

#### 포인트
- Jenkins Credentials(`reg-creds`)로 비밀 관리.
- 워크로드가 많다면 **에이전트 풀** 또는 **Kubernetes 플러그인**으로 **동적 Pod 에이전트** 활용:
  - `agent { kubernetes { yaml """ ...pod spec... """ } }`

### 4.2 Jenkins + K8s 에이전트 스니펫
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

## 5. 배포 전략(서버·Compose·Kubernetes)

### 5.1 단일 서버/Compose 표준 예시
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
- CI에서 `image:`만 **다이제스트**로 바꾸는 방식 권장:
  - `image: registry.example.com/demo/myapp@sha256:...`

### 5.2 Kubernetes 롤링/블루-그린/카나리
- **롤링**: `Deployment` 기본; `maxSurge/maxUnavailable` 조정
- **블루-그린**: `blue`와 `green` 디플로이먼트/서비스 분리 → 라우팅 스위치
- **카나리**: `weight` 기반 Ingress/ServiceMesh 이관(예: NGINX Ingress, Istio)

#### Deployment 예시(이미지 다이제스트 고정)
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
        image: registry.example.com/demo/myapp@sha256:abcd...   # 고정!
        ports: [{containerPort: 8080}]
        readinessProbe:
          httpGet: { path: /health, port: 8080 }
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

## 6. 테스트/품질/보안 통합

### 6.1 테스트/커버리지
- PR에서 `pytest --junitxml=...`/`go test -json`/`jest --ci --reporters` 등으로 **리포트 업로드**.
- 커버리지 기준 미달 시 실패 게이트.

### 6.2 린트/포맷/정적분석
- Python: ruff/flake8/black  
- JS/TS: eslint/prettier  
- IaC: `tflint`, `checkov`, `kics`  
- Dockerfile: `hadolint`, `dockle`

### 6.3 컨테이너 스캔/서명/정책
- **Trivy**: `trivy image --exit-code 1 --severity HIGH,CRITICAL`
- **SBOM**: Anchore Syft/CycloneDX 생성 → 아티팩트 보관
- **서명**: Cosign keyless(oidc) 또는 KMS 키 → **배포 전 검증**
- **정책**: OPA/Gatekeeper/Kyverno로 “서명된 이미지만 배포” 규칙

---

## 7. 비밀관리(Secrets)

- CI: GitHub Actions Secrets/GitLab Variables/Jenkins Credentials
- 런타임: **Docker Secrets(Swarm)**, **K8s Secrets**, 외부 Vault(예: HashiCorp Vault)/Cloud Secret Manager
- OIDC(클라우드)로 **무자격증명 접근**(임시 토큰) → 장기 키 제거

---

## 8. 성능/비용 최적화

### 8.1 빌드 가속
- buildx + registry 캐시
- 의존성 레이어 분리(멀티스테이지)
- `.dockerignore` 정리
- 사설 **프록시 캐시 레지스트리/Harbor Proxy Cache**로 외부 pull 절감

### 8.2 테스팅 가속
- **서비스 컨테이너**로 DB/브로커 붙여 통합테스트(깃허브액션/깃랩 지원)
- **매트릭스 전략**(OS/파이썬 버전/플랫폼) 병렬화  
  예) `strategy.matrix.python: [3.10, 3.11]`

### 8.3 단순 모델로 통신량 절감의 대략 계산
빌드 캐시/프록시 캐시로 절감되는 네트워크 바이트 총량 \(S\)는 간단히
$$
S \approx \sum_{l=1}^{L} (B_l \cdot (n_l - 1))
$$
- \(L\): 공유 레이어 개수, \(B_l\): 레이어 \(l\)의 바이트 크기, \(n_l\): 해당 레이어 다운로드 횟수.
- 최초 1회 이후는 캐시 적중으로 외부 전송이 줄어든다.

---

## 9. 배포 후 검증/롤백

### 9.1 헬스체크/스모크 테스트
- 서버/클러스터에서 `/health` 확인, 2xx/준수시간 내 응답
- E2E 테스트(간단한 기능 동작 검사)를 **배포 직후** 자동화

### 9.2 롤백 방법
- Compose: 직전 다이제스트로 `image` 변경 → `up -d`
- K8s: `kubectl rollout undo deploy/myapp` 또는 GitOps에서 이전 리비전

---

## 10. 관측성(Observability)

- 로그: Loki/ELK/CloudWatch → CI 단계·릴리스태그 메타 포함
- 지표: Prometheus + Grafana → 배포 전후 **에러율/지연/자원** 비교
- 트레이싱: OpenTelemetry/Jaeger → 특정 릴리스 이상징후 추적
- 알림: Slack/Teams/Webhook(빌드 성공/실패/배포 완료/롤백)

---

## 11. 레지스트리/배포 환경 통합 팁

- **Harbor**: RBAC, 스캔, 리텐션, 불변 태그, 복제; Robot 계정으로 CI 접근
- **ECR/GAR/GHCR**: OIDC로 무비밀 로그인 → 보안/회계 간편화
- 이미지 참조는 **다이제스트**로 잠금; 태그는 가독/프로모션 용도

---

## 12. 샘플 리포지터리 구조(권장)

```
repo/
├─ app/                      # 애플리케이션
├─ charts/myapp/             # Helm 차트(배포 스펙)
├─ deploy/compose/           # Compose 파일
├─ Dockerfile
├─ .dockerignore
├─ .github/workflows/        # GH Actions
│   ├─ ci.yml
│   └─ deploy.yml
├─ .gitlab-ci.yml            # GitLab 선택 시
├─ Jenkinsfile               # Jenkins 선택 시
├─ Makefile                  # 로컬 개발/도커 명령 모음
└─ security/                 # 정책/서명/스캔 설정
```

---

## 13. 트러블슈팅

| 증상 | 원인/해결 |
|---|---|
| `x509: certificate signed by unknown authority` | 사설 레지스트리 CA 미신뢰 → `/etc/docker/certs.d/<host:port>/ca.crt` 배포 |
| `denied: requested access to the resource is denied` | 로그인/권한/리포지토리 프라이빗/네임 오타 확인 |
| buildx 캐시 안 먹음 | `cache-from/to` 참조 이미지 권한/태그/레지스트리 상주 여부 확인 |
| DinD 속도 저하 | Kaniko/Buildah로 전환, 레이어/캐시 최적화 |
| 배포 후 502/CrashLoop | 헬스체크 지연/리소스 제한/환경변수/시크릿 누락, 롤백 후 원인 분석 |
| Trivy CRITICAL 다수 | 베이스 이미지 업데이트, 패키지 버전 고정, 불필요 패키지 제거(알파인/슬림) |

---

## 14. 고급: 모노레포/멀티서비스 매트릭스

### 14.1 서비스별 디렉터리 감지로 조건부 빌드
```yaml
# GitHub Actions 예시: path filter
on:
  push:
    branches: [main]
    paths:
      - 'services/api/**'
      - '.github/workflows/**'
```

### 14.2 매트릭스(서비스 × 플랫폼)
```yaml
strategy:
  matrix:
    service: [api, web, worker]
    platform: [linux/amd64, linux/arm64]
```

---

## 15. 종합 예: “PR → 스캔/테스트 → 이미지 빌드/푸시 → 스테이징 자동배포 → 승인 후 프로덕션”

1) PR 시: Lint/Unit/Trivy(SBOM)/컨테이너 빌드(푸시 X)  
2) main 머지: 빌드 → 서명 → `:gitSHA` 푸시  
3) 스테이징: Helm로 다이제스트 배포 + 스모크 테스트  
4) 수동 승인: 프로덕션로 동일 다이제스트 롤아웃  
5) 실패 시 자동 롤백 + 알림 + 이슈 트래킹(릴리스 태그)

---

## 16. 체크리스트

- [ ] Dockerfile 멀티스테이지/캐시/최소화(슬림/알파인/디스트로리스)  
- [ ] buildx 멀티아키 + 레지스트리 캐시  
- [ ] SBOM/Trivy/서명(Cosign) + 정책 게이트  
- [ ] 다이제스트 고정 배포, 태그는 가독/프로모션  
- [ ] 비밀은 Secrets/Variables/Credentials, OIDC 적극 사용  
- [ ] 스테이징 자동/프로덕션 승인, 롤백 자동화  
- [ ] 관측성(로그/지표/트레이싱)과 알림 연결  
- [ ] 문서화(README/Runbook)와 백업/DR 시나리오

---

## 참고 문서
- GitHub Actions: https://docs.github.com/actions  
- GitLab CI/CD: https://docs.gitlab.com/ee/ci/  
- Jenkins Pipeline: https://www.jenkins.io/doc/book/pipeline/  
- Docker Buildx: https://docs.docker.com/build/buildx/  
- Trivy: https://aquasecurity.github.io/trivy/  
- Cosign: https://docs.sigstore.dev/cosign/  
- ArgoCD: https://argo-cd.readthedocs.io/  
- Harbor: https://goharbor.io/

---

## 부록 A. GitHub Actions “재사용 가능한 워크플로”로 표준화
```yaml
# .github/workflows/reusable-docker-build.yml
name: reusable-docker-build
on:
  workflow_call:
    inputs:
      image:
        required: true
        type: string

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: docker/setup-buildx-action@v3
    - uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    - uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ inputs.image }}:latest,${{ inputs.image }}:${{ github.sha }}
```

---

## 부록 B. GitLab CI “템플릿 include”
```yaml
# .gitlab-ci-templates/docker-build.yml
.build_docker:
  stage: build
  image: docker:27-dind
  services: [docker:27-dind]
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```
```yaml
# .gitlab-ci.yml
include:
  - local: '.gitlab-ci-templates/docker-build.yml'

build:
  extends: .build_docker
```

---

## 부록 C. Jenkins 공유 라이브러리로 파이프라인 재사용
```groovy
// vars/dockerBuild.groovy
def call(String image) {
  sh "docker build -t ${image}:${env.BUILD_NUMBER} -t ${image}:latest ."
  sh "docker push ${image}:${env.BUILD_NUMBER}"
  sh "docker push ${image}:latest"
}
```
```groovy
// Jenkinsfile
@Library('company-lib') _
pipeline {
  agent any
  stages {
    stage('Build&Push'){ steps { dockerBuild("registry.example.com/demo/myapp") } }
  }
}
```

---

### 결론
- **GitHub Actions**: 간결·강력한 마켓플레이스, OIDC·buildx 친화적  
- **GitLab CI**: 레지스트리·권한·프로젝트와 **완전 통합**  
- **Jenkins**: 온프레미스·커스터마이즈·에코시스템 극대화  

세 플랫폼 어디서든 **동일한 원칙**(최소 이미지·캐시·보안게이트·다이제스트 배포·관측성·자동 롤백)을 적용하면, Docker 중심 CI/CD는 **신뢰·속도·보안**을 동시에 달성한다.