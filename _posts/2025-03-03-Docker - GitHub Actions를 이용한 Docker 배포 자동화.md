---
layout: post
title: Docker - GitHub Actions를 이용한 Docker 배포 자동화
date: 2025-03-03 20:20:23 +0900
category: Docker
---
# GitHub Actions를 이용한 Docker 배포 자동화 가이드

## 개요

이 가이드는 GitHub Actions를 사용하여 Docker 애플리케이션의 CI/CD 파이프라인을 구축하는 방법을 설명합니다. 코드 변경 시 자동으로 Docker 이미지를 빌드하고, 보안 검사를 수행하며, 컨테이너 레지스트리에 푸시하고, 원격 서버에 배포하는 완전한 워크플로우를 구성할 수 있습니다.

## 시스템 아키텍처와 워크플로우

### 기본 워크플로우 흐름

```
[코드 Push 또는 Release 생성]
        │
        ▼
[GitHub Actions Runner 시작]
        │
        ▼
[Docker 이미지 빌드 (Buildx + QEMU)]
        │
        ▼
[보안 검사 (SBOM 생성, 취약점 스캔, 서명)]
        │
        ▼
[컨테이너 레지스트리 푸시 (Docker Hub 또는 GHCR)]
        │
        ▼
[원격 서버 배포 (SSH, Webhook, 또는 K8s)]
```

### 프로젝트 구조 예시

```plaintext
my-application/
├── app/                    # 애플리케이션 소스 코드
│   └── main.py
├── Dockerfile             # 멀티스테이지 Dockerfile
├── docker-compose.yml     # 배포 구성
├── .dockerignore          # 빌드 컨텍스트 제외 파일
└── .github/
    └── workflows/         # GitHub Actions 워크플로우
        ├── docker-build-push.yml
        └── deploy-production.yml
```

## Dockerfile 구성

효율적이고 안전한 Docker 이미지를 위해 멀티스테이지 빌드를 사용합니다:

```dockerfile
# syntax=docker/dockerfile:1.7

# 빌드 스테이지
FROM python:3.11-slim AS builder
WORKDIR /app
ENV PIP_DISABLE_PIP_VERSION_CHECK=1 PIP_NO_CACHE_DIR=1
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

# 런타임 스테이지 (경량화)
FROM gcr.io/distroless/python3-debian12:nonroot
WORKDIR /app
ENV PYTHONUNBUFFERED=1
COPY --from=builder /install /usr/local
COPY app/ ./app/
USER nonroot
EXPOSE 8080
CMD ["app/main.py"]
```

**주요 특징**:
- 멀티스테이지 빌드로 최종 이미지 크기 최소화
- Distroless 베이스 이미지로 공격 표면적 감소
- 비루트(nonroot) 사용자로 실행하여 보안 강화

## GitHub Actions 워크플로우

### 기본 Docker 빌드 및 푸시 워크플로우

{% raw %}
```yaml
name: Docker Build and Push

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
  workflow_dispatch:  # 수동 실행 허용

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      # 1. 코드 체크아웃
      - name: Checkout repository
        uses: actions/checkout@v4

      # 2. Docker Buildx 설정
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # 3. Docker Hub 로그인
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      # 4. 메타데이터 추출 (태그, 라벨 자동 생성)
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ secrets.DOCKER_USERNAME }}/my-app
          tags: |
            type=raw,value=latest
            type=sha
            type=ref,event=branch
            type=semver,pattern={{version}}

      # 5. Docker 이미지 빌드 및 푸시
      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```
{% endraw %}

### GitHub Container Registry (GHCR) 사용

GitHub Packages의 컨테이너 레지스트리를 사용하는 경우:

{% raw %}
```yaml
- name: Login to GitHub Container Registry
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Extract metadata for GHCR
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: ghcr.io/${{ github.repository }}/my-app
    tags: |
      type=raw,value=latest
      type=sha
      type=ref,event=branch
```
{% endraw %}

### 멀티 아키텍처 빌드 지원

여러 CPU 아키텍처를 지원하는 이미지를 빌드하려면:

{% raw %}
```yaml
- name: Set up QEMU for multi-architecture builds
  uses: docker/setup-qemu-action@v3

- name: Build and push multi-arch images
  uses: docker/build-push-action@v6
  with:
    context: .
    platforms: linux/amd64,linux/arm64
    push: true
    tags: ${{ steps.meta.outputs.tags }}
```
{% endraw %}

## 보안 강화 워크플로우

### Software Bill of Materials (SBOM) 생성

{% raw %}
```yaml
- name: Generate SBOM with Syft
  uses: anchore/sbom-action@v0
  with:
    image: ${{ steps.meta.outputs.tags }}
    format: spdx-json
    output-file: sbom.spdx.json

- name: Upload SBOM as artifact
  uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: sbom.spdx.json
```
{% endraw %}

### 취약점 스캔

{% raw %}
```yaml
- name: Scan for vulnerabilities with Trivy
  uses: aquasecurity/trivy-action@0.22.0
  with:
    image-ref: ${{ secrets.DOCKER_USERNAME }}/my-app:latest
    format: table
    severity: CRITICAL,HIGH
    ignore-unfixed: true
```
{% endraw %}

### 컨테이너 이미지 서명

{% raw %}
```yaml
- name: Install Cosign
  uses: sigstore/cosign-installer@v3.6.0

- name: Sign container image with Cosign
  run: |
    cosign sign --yes ${{ secrets.DOCKER_USERNAME }}/my-app:latest
```
{% endraw %}

## 배포 자동화

### SSH를 통한 원격 서버 배포

원격 서버에 SSH로 접속하여 Docker Compose를 사용해 배포하는 예시:

{% raw %}
```yaml
name: Deploy to Production

on:
  workflow_dispatch:
  push:
    branches: [ "main" ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # 환경 보호 규칙 적용
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.REMOTE_HOST }}
          username: ${{ secrets.REMOTE_USER }}
          key: ${{ secrets.REMOTE_SSH_KEY }}
          script: |
            # 최신 이미지 풀
            docker pull ${{ secrets.DOCKER_USERNAME }}/my-app:latest
            
            # 기존 컨테이너 중지
            docker compose -f /srv/my-app/docker-compose.yml down
            
            # 새 컨테이너 시작
            docker compose -f /srv/my-app/docker-compose.yml up -d
            
            # 오래된 이미지 정리
            docker image prune -af --filter "until=168h"
```
{% endraw %}

### 원격 서버 Docker Compose 파일 예시

```yaml
version: '3.9'

services:
  app:
    image: yourname/my-app:latest
    restart: unless-stopped
    ports:
      - "80:8080"
    environment:
      APP_ENV: production
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:8080/health"]
      interval: 30s
      timeout: 3s
      retries: 3
```

## 고급 배포 전략

### 롤백 워크플로우

문제 발생 시 이전 버전으로 롤백하는 워크플로우:

{% raw %}
```yaml
name: Rollback Deployment

on:
  workflow_dispatch:
    inputs:
      tag:
        description: "Rollback to tag (e.g., v1.2.3 or commit SHA)"
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - name: Rollback via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.REMOTE_HOST }}
          username: ${{ secrets.REMOTE_USER }}
          key: ${{ secrets.REMOTE_SSH_KEY }}
          script: |
            set -e
            IMG="${{ secrets.DOCKER_USERNAME }}/my-app:${{ github.event.inputs.tag }}"
            
            # 지정된 태그의 이미지 풀
            docker pull $IMG
            
            # Docker Compose 파일에서 이미지 태그 변경
            sed -i "s|image: .*|image: $IMG|g" /srv/my-app/docker-compose.yml
            
            # 컨테이너 재시작
            docker compose -f /srv/my-app/docker-compose.yml down
            docker compose -f /srv/my-app/docker-compose.yml up -d
```
{% endraw %}

### 자동화된 재배포 도구 통합

#### Watchtower를 이용한 자동 업데이트

서버에서 Watchtower를 실행하면 새 이미지를 자동으로 감지하고 컨테이너를 업데이트합니다:

```bash
docker run -d \
  --name watchtower \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower --cleanup --interval 30
```

#### Portainer 웹훅을 통한 배포

Portainer에서 웹훅을 생성하고 GitHub Actions에서 트리거할 수 있습니다:

{% raw %}
```yaml
- name: Trigger Portainer webhook
  run: |
    curl -X POST https://portainer.example.com/api/webhooks/<webhook-id>
```
{% endraw %}

## 태깅 전략

효과적인 이미지 관리를 위한 태깅 전략:

1. **latest**: 최신 안정 버전 (기본 브랜치 푸시 시)
2. **커밋 SHA**: 특정 커밋에 대한 정확한 버전 (모든 푸시 시)
3. **브랜치 이름**: 개발 중인 기능 버전 (특정 브랜치 푸시 시)
4. **Semantic Version (vX.Y.Z)**: 공식 릴리스 (GitHub Release 생성 시)

## 성능 최적화

### 빌드 캐시 활용

GitHub Actions 캐시를 활용하여 빌드 시간을 단축합니다:

{% raw %}
```yaml
- name: Build with cache optimization
  uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```
{% endraw %}

### .dockerignore 파일

불필요한 파일을 빌드 컨텍스트에서 제외하여 빌드 속도를 향상시킵니다:

```gitignore
.git/
.github/
.env
**/__pycache__/
*.log
node_modules/
dist/
build/
*.pyc
```

## 환경별 배포 전략

### 개발, 스테이징, 프로덕션 분리

```yaml
name: Multi-Environment Deployment

on:
  push:
    branches:
      - develop    # 스테이징 환경
      - main       # 프로덕션 환경

jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: 
          - ${{ github.ref == 'refs/heads/develop' && 'staging' || 'production' }}
    steps:
      - name: Deploy to ${{ matrix.environment }}
        run: |
          echo "Deploying to ${{ matrix.environment }} environment"
          # 환경별 배포 로직
```

## 종합 워크플로우 예시

{% raw %}
```yaml
name: Complete CI/CD Pipeline

on:
  push:
    branches: [ "main", "develop" ]
  release:
    types: [ published ]
  workflow_dispatch:

jobs:
  build-and-test:
    runs-on: ubuntu-latest
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
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}/my-app
          tags: |
            type=raw,value=latest
            type=ref,event=branch
            type=sha
            type=semver,pattern={{version}}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Security scan
        uses: aquasecurity/trivy-action@0.22.0
        with:
          image-ref: ${{ fromJSON(steps.meta.outputs.json).tags[0] }}
          ignore-unfixed: true
          severity: CRITICAL,HIGH

  deploy-staging:
    needs: build-and-test
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to staging
        run: |
          echo "Deploying to staging environment"
          # 스테이징 배포 로직

  deploy-production:
    needs: build-and-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.PRODUCTION_HOST }}
          username: ${{ secrets.PRODUCTION_USER }}
          key: ${{ secrets.PRODUCTION_SSH_KEY }}
          script: |
            docker pull ghcr.io/${{ github.repository }}/my-app:latest
            docker compose -f /srv/my-app/docker-compose.yml up -d
```
{% endraw %}

## 보안 모범 사례

1. **비밀 정보 관리**: Docker 자격 증명, SSH 키, API 토큰은 GitHub Secrets에 저장
2. **최소 권한 원칙**: 서비스 계정에 필요한 최소한의 권한만 부여
3. **정기적인 보안 검사**: 취약점 스캔을 정기적으로 실행
4. **이미지 서명**: 배포된 이미지의 무결성 검증
5. **환경 보호**: 프로덕션 배포 시 승인 요구사항 설정

## 문제 해결

### 일반적인 문제 및 해결 방법

| 문제 | 원인 | 해결 방법 |
|------|------|-----------|
| Docker Hub 로그인 실패 | 잘못된 자격 증명 | GitHub Secrets 값 확인 및 업데이트 |
| 빌드 시간 과다 | 캐시 미사용 | Buildx 캐시 설정 확인 |
| 배포 실패 | SSH 연결 문제 | SSH 키 권한 및 서버 접근 확인 |
| 메모리 부족 | 빌드 리소스 부족 | 더 큰 GitHub Actions Runner 사용 |
| 취약점 검출 | 이미지에 취약한 패키지 포함 | 베이스 이미지 업데이트 또는 패키지 교체 |

### 디버깅 명령어

워크플로우 실행 중 문제를 진단하기 위해:

{% raw %}
```yaml
- name: Debug information
  run: |
    echo "Branch: ${{ github.ref }}"
    echo "SHA: ${{ github.sha }}"
    echo "Runner OS: ${{ runner.os }}"
    echo "Image tags: ${{ steps.meta.outputs.tags }}"
```
{% endraw %}

## 결론

GitHub Actions를 활용한 Docker CI/CD 파이프라인 구축은 개발부터 배포까지의 전체 프로세스를 자동화하고 표준화하는 강력한 방법입니다. 이 가이드에서 제시한 워크플로우는 다음과 같은 이점을 제공합니다:

1. **자동화된 빌드 및 배포**: 코드 변경 시 자동으로 이미지 빌드 및 배포
2. **멀티 아키텍처 지원**: 다양한 플랫폼에서 동작하는 이미지 생성
3. **보안 강화**: SBOM 생성, 취약점 스캔, 이미지 서명을 통한 보안 검증
4. **롤백 기능**: 문제 발생 시 이전 버전으로 빠르게 복구
5. **환경 분리**: 개발, 스테이징, 프로덕션 환경을 명확히 분리

이 구성을 시작점으로 삼아 조직의 특정 요구사항에 맞게 워크플로우를 조정하고 확장할 수 있습니다. 지속적인 통합과 지속적인 배포는 소프트웨어 개발 생명주기의 품질과 안정성을 크게 향상시킵니다. 초기에는 단순한 워크플로우로 시작하고, 필요에 따라 점진적으로 고급 기능을 추가해 나가는 접근 방식을 권장합니다.

GitHub Actions의 강력한 에코시스템과 Docker의 유연한 컨테이너 기술을 결합하면 효율적이고 안정적인 CI/CD 파이프라인을 구축할 수 있습니다.