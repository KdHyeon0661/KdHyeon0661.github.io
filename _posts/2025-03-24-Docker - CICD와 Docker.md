---
layout: post
title: Docker - CI/CD와 Docker
date: 2025-03-24 19:20:23 +0900
category: Docker
---
# 🚀 CI/CD와 Docker: GitHub Actions, GitLab CI, Jenkins 완전 정복

---

## 📌 왜 Docker + CI/CD인가?

| 항목 | 장점 |
|------|------|
| 일관성 | 어느 환경이든 동일한 컨테이너 실행 |
| 이식성 | 빌드 결과를 이미지로 패키징 |
| 속도 | 캐시 활용, 병렬 실행 |
| 배포 자동화 | 빌드 → 테스트 → 이미지 Push → 배포까지 자동화 가능 |

---

## 🔧 공통된 흐름

1. 코드 변경 (Push or PR)
2. 테스트 실행
3. Docker 이미지 빌드
4. 이미지 Push (예: Docker Hub, Harbor)
5. 서버 or 클러스터에 배포

---

## ✅ 1. GitHub Actions

### 📄 `.github/workflows/docker.yml` 예시

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [ "main" ]

jobs:
  docker:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and Push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: yourid/yourapp:latest
```

### 🔐 Secrets 설정

- `DOCKER_USERNAME`, `DOCKER_PASSWORD` → GitHub > Settings > Secrets

### ✅ 장점

- GitHub에 코드만 있으면 바로 실행 가능
- Docker 관련 공식 액션이 매우 잘 갖춰짐

---

## ✅ 2. GitLab CI/CD

### 📄 `.gitlab-ci.yml` 예시

```yaml
stages:
  - build
  - deploy

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
```

### 🔐 자동 변수

| 변수 | 설명 |
|------|------|
| `CI_REGISTRY_IMAGE` | 현재 프로젝트의 이미지 주소 |
| `CI_REGISTRY` | GitLab 컨테이너 레지스트리 |
| `CI_COMMIT_SHORT_SHA` | 커밋 short hash |

### ✅ 장점

- GitLab 자체에 Docker Registry 내장
- `docker:dind` (Docker-in-Docker)로 동작
- Private 환경에서 통합에 용이

---

## ✅ 3. Jenkins + Docker

### 📄 Jenkinsfile 예시 (Pipeline as Code)

```groovy
pipeline {
  agent any

  environment {
    DOCKER_IMAGE = "myrepo/myapp:${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Image') {
      steps {
        sh 'docker build -t $DOCKER_IMAGE .'
      }
    }

    stage('Push Image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push $DOCKER_IMAGE
          '''
        }
      }
    }
  }
}
```

### 🔧 설정 포인트

- Jenkins에서 Docker 설치 필요
- Docker 실행 권한 (`sudo` or 권한 그룹 포함)
- `dockerhub-creds` → Jenkins Credentials로 설정

### ✅ 장점

- 복잡한 커스터마이징 및 멀티 스테이지 지원
- 외부 서버, 온프레미스에 적합
- 많은 플러그인 (Slack, Git, Vault 등)

---

## 🆚 비교 정리

| 항목 | GitHub Actions | GitLab CI | Jenkins |
|------|----------------|-----------|---------|
| 통합도 | GitHub와 완전 통합 | GitLab과 완전 통합 | 외부 통합 필요 |
| 실행 방식 | GitHub 호스팅 | GitLab or Self-hosted | 온프레미스 가능 |
| 레지스트리 | Docker Hub or 외부 | GitLab Container Registry 내장 | 자유롭게 구성 |
| 복잡도 | 낮음 | 중간 | 높음 |
| 확장성 | 중간 | 높음 | 매우 높음 |
| 커뮤니티 | 활발 | 활발 | 오래됨, 튼튼함 |

---

## 🎯 실무 구성 예시

### 예: GitHub Actions → Docker Hub → 클러스터 배포

1. `main` 브랜치에 PR 머지
2. GitHub Actions가 Docker 이미지 빌드
3. Docker Hub에 Push
4. ArgoCD or K8s가 감지하여 새 이미지로 자동 배포

---

## 🛠️ 고급 기능

| 기능 | 설명 |
|------|------|
| Cache Reuse | Docker Layer Caching (`buildx`, `cache-from`) |
| Multi-platform | `platform: linux/amd64, linux/arm64` 빌드 |
| Multi-stage Build | 최적화된 Dockerfile 사용 |
| Docker Compose + CI | 서비스 전체 테스트 가능 |

---

## 📚 참고 링크

- [GitHub Actions 공식 문서](https://docs.github.com/actions)
- [GitLab CI/CD Docs](https://docs.gitlab.com/ee/ci/)
- [Jenkins Pipeline 문서](https://www.jenkins.io/doc/book/pipeline/)
- [Docker Buildx](https://docs.docker.com/build/buildx/)