---
layout: post
title: Docker - BuildKit
date: 2025-03-27 19:20:23 +0900
category: Docker
---
# BuildKit과 Docker 캐시 전략

## BuildKit 개요

| 항목 | Legacy Builder | BuildKit |
|---|---|---|
| 실행 모델 | 단순 순차 실행 | **DAG 기반 병렬 실행**(LLB 그래프 활용) |
| 캐시 | 제한적 (로컬 캐시 중심) | **레이어 및 명령 단위의 정교한 캐시** + 원격 캐시 지원 |
| 멀티 플랫폼 | 제약 사항 많음 | **buildx**를 통한 완전 지원 (`--platform` 플래그) |
| 보안 비밀 | 지원하지 않음 | `RUN --mount=type=secret/ssh`를 통한 안전한 주입 |
| 출력 제어 | 이미지 생성만 가능 | `--output`을 통한 다양형식(image, tar, local, registry 등) |
| 로깅 | 일반 텍스트 출력 | 구조화된 로깅 및 프로그레스 UI |
| 분산/원격 캐시 | 미지원 | **type=registry, type=gha, type=local** 등 다양한 백엔드 |

**주요 용어**
- **LLB**(Low-Level Build): BuildKit의 내부 빌드 의존성 그래프 표현 형식입니다.
- **Inline cache**: 빌드된 이미지 자체에 캐시 메타데이터를 포함시키는 방식입니다.
- **Cache exporter/importer**: `--cache-to` 및 `--cache-from` 옵션으로 캐시 저장소를 외부화하여 관리합니다.

---

## BuildKit 활성화 및 기본 사용

### 활성화 방법

Linux 환경에서 세션별 활성화:
```bash
export DOCKER_BUILDKIT=1
```

Docker 데몬 전역 설정 (`/etc/docker/daemon.json`):
```json
{
  "features": {
    "buildkit": true
  }
}
```
> `docker buildx build ...` 명령을 사용하면 BuildKit이 자동으로 활성화됩니다.

### 간단한 빌드 실행

```bash
DOCKER_BUILDKIT=1 docker build -t myapp:latest .
```

---

## 캐시 동작 원리: “레이어 해시”와 무효화 조건

BuildKit은 Dockerfile의 **각 명령어(COPY, RUN, ADD, ENV, ARG 등)** 를 개별 레이어로 처리합니다. 각 레이어는 **명령어 자체, 입력 파일의 스냅샷, 빌드 환경**을 기반으로 고유한 **해시(Hash)** 를 생성합니다. 이 해시가 이전 빌드와 동일하면 **캐시 히트**가 발생하여 해당 레이어의 재실행을 건너뜁니다.

**캐시 무효화를 유발하는 주요 요인:**
1.  **명령어 텍스트 변경**: `RUN` 명령어의 내용, 순서, 옵션, 심지어 공백 변화도 해시를 바꿉니다.
2.  **입력 파일 내용 변경**: `COPY`나 `ADD`로 복사하는 파일의 내용이 변경되면 해당 레이어의 해시가 바뀝니다.
3.  **빌드 인수 또는 환경변수 변경**: `ARG`나 `ENV`로 설정된 값이 달라지면 영향을 받는 후속 레이어의 캐시가 무효화됩니다.
4.  **베이스 이미지 변경**: `FROM` 지시어로 지정한 베이스 이미지의 다이제스트(Digest)가 업데이트되면 모든 레이어 캐시가 초기화됩니다.
5.  **비결정적(Non-deterministic) 요소**: 타임스탬프, 랜덤값 생성, 네트워크에서 동적으로 가져오는 파일 등은 캐시 일관성을 해칠 수 있습니다.

**핵심 요약:** 효율적인 캐시 활용을 위해서는 Dockerfile에서 **자주 변경되지 않는 작업(의존성 설치 등)** 을 위쪽에, **자주 변경되는 작업(애플리케이션 소스 복사 등)** 을 아래쪽에 배치하는 설계가 가장 중요합니다.

---

## 캐시 전략 설계 — “순서”와 “컨텍스트” 최적화

### 의존성 설치를 먼저, 소스 복사를 나중에

**Node.js 애플리케이션 예시**

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
# 1. 의존성 정의 파일만 먼저 복사하여 설치 레이어를 독립적으로 캐시
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
# 2. 소스 코드 전체를 복사. 소스 변경 시 이 아래 레이어만 재빌드
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
CMD ["node","dist/index.js"]
```

**Python 애플리케이션 예시**

```dockerfile
FROM python:3.11-slim AS base
WORKDIR /app

# 1. 요구사항 파일 복사 및 의존성 설치 (변경 빈도 낮음)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 2. 애플리케이션 소스 코드 복사 (변경 빈도 높음)
COPY . .
CMD ["python","app.py"]
```

### `.dockerignore` 파일로 빌드 컨텍스트 최소화

```dockerignore
node_modules
.git
*.log
Dockerfile
README.md
dist
```
> 불필요한 파일이 빌드 컨텍스트에 포함되면 `COPY . .` 명령의 입력 해시가 매번 달라져 **캐시 효율이 크게 떨어집니다.** `.dockerignore`를 꼼꼼히 관리하세요.

---

## CI/CD 환경에서의 실전 캐시 가속

### Inline Cache (단일 저장소에서 간편하게 사용)

이미지를 푸시할 때 캐시 메타데이터도 이미지 내부에 함께 저장하는 방식입니다.

```bash
docker build \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  --tag yourname/myapp:latest \
  --cache-from=yourname/myapp:latest \
  .
```
- **첫 번째 빌드**: 캐시가 없으므로 일반 빌드와 동일한 시간이 소요됩니다.
- **두 번째 이후 빌드**: `--cache-from` 옵션으로 지정한 이전 이미지(`:latest`)에서 캐시 메타데이터를 읽어와 재사용합니다.
> **주의**: 이 캐시를 다른 CI 러너에서도 사용하려면 반드시 이미지를 **레지스트리에 푸시**해야 합니다.

### Buildx와 원격 레지스트리 캐시 (권장 방식)

**`type=registry`** 백엔드를 사용하여 캐시를 레지스트리에 별도로 저장하고 공유합니다.

```bash
docker buildx build \
  --cache-from type=registry,ref=yourname/myapp:buildcache \
  --cache-to   type=registry,ref=yourname/myapp:buildcache,mode=max \
  -t yourname/myapp:latest \
  --push .
```
- **장점**: CI/CD 에이전트가 매번 새로 생성되더라도 **공유 레지스트리에서 캐시를 즉시 가져와** 빌드 속도를 크게 향상시킬 수 있습니다.
- `mode=max` 설정은 가능한 많은 캐시 메타데이터를 저장하여 캐시 히트 확률을 높입니다.

**GitHub Actions 전용 캐시 백엔드(`type=gha`)**

```yaml
- name: Build & Push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: yourname/myapp:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**로컬 디렉토리 캐시(`type=local`) — 개발자 워크스테이션용**

```bash
docker buildx build \
  --cache-from type=local,src=.buildx-cache \
  --cache-to   type=local,dest=.buildx-cache,mode=max \
  -t yourname/myapp:dev .
```

---

## 고급 기능: 멀티 스테이지와 BuildKit 마운트 (secret, ssh, cache)

### Secret 마운트 — 비밀값을 이미지 레이어에 노출하지 않기

```dockerfile
# syntax=docker/dockerfile:1.7

FROM alpine:3.20
RUN --mount=type=secret,id=apikey \
    sh -c 'apk add --no-cache curl && curl -H "X-API-Key: $(cat /run/secrets/apikey)" https://api.example.com'
```

빌드 시 비밀 파일 전달:
```bash
docker buildx build \
  --secret id=apikey,src=./apikey.txt \
  -t secret-demo:latest .
```
> **장점**: API 키 같은 민감정보가 최종 이미지 레이어나 빌드 로그에 절대 남지 않습니다.

### SSH 마운트 — 프라이빗 Git 리포지토리 접근

```dockerfile
# syntax=docker/dockerfile:1.7

FROM alpine:3.20
RUN apk add --no-cache git openssh
RUN --mount=type=ssh git clone git@github.com:org/private-repo.git
```

빌드 시 SSH 에이전트 전달:
```bash
docker buildx build --ssh default -t ssh-demo:latest .
```

### 캐시 마운트 — 패키지 매니저 캐시 디렉토리 유지

```dockerfile
# npm 패키지 캐시 예시
RUN --mount=type=cache,target=/root/.npm \
    npm ci
```
> 반복적인 빌드에서 네트워크 다운로드와 패키지 설치 시간을 획기적으로 절약합니다.

---

## 멀티 아키텍처 빌드와 캐시 통합

```bash
# Buildx 빌더 생성 및 설정
docker buildx create --use --name mybuilder
docker buildx inspect --bootstrap
# 멀티 아키텍처 빌드를 위한 에뮬레이터 설치
docker run --privileged --rm tonistiigi/binfmt --install all

# 멀티 플랫폼 빌드 실행 (캐시 통합)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=yourname/myapp:buildcache \
  --cache-to   type=registry,ref=yourname/myapp:buildcache,mode=max \
  -t yourname/myapp:latest \
  --push .
```
- 멀티 플랫폼 빌드는 일반적으로 `--push`와 함께 사용하여 **매니페스트 리스트**를 레지스트리에 직접 푸시합니다.
- 단일 로컬 이미지로 로드(`--load`)하는 작업은 **하나의 플랫폼**에 대해서만 가능합니다.
- 성능이 중요한 경우, ARM64 전용 물리적/가상 머신과 같은 **네이티브 빌드 러너**를 별도로 구성하는 것을 고려하세요.

---

## 언어별 Dockerfile 캐시 최적화 패턴

### Go (크로스 컴파일 용이, Distroless 런타임)

```dockerfile
# Build Stage
FROM --platform=$BUILDPLATFORM golang:1.23-alpine AS builder
ARG TARGETOS TARGETARCH
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
ENV CGO_ENABLED=0
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /out/app ./cmd/app

# Runtime Stage
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

### Python (Wheel 파일 사전 빌드)

```dockerfile
FROM --platform=$BUILDPLATFORM python:3.11-alpine AS wheels
WORKDIR /w
COPY requirements.txt .
RUN apk add --no-cache build-base linux-headers
RUN pip wheel --wheel-dir /w -r requirements.txt

FROM python:3.11-alpine
WORKDIR /app
COPY --from=wheels /w /w
RUN pip install --no-cache-dir --no-index --find-links=/w -r /w/requirements.txt
COPY . .
CMD ["python","app.py"]
```

### Java (JLink를 통한 런타임 최소화)

```dockerfile
FROM --platform=$BUILDPLATFORM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /src
COPY . .
RUN ./gradlew clean build -x test \
 && jlink --add-modules java.base,java.logging \
          --strip-debug --compress=2 --no-header-files --no-man-pages \
          --output /jre

FROM alpine:3.20
ENV JAVA_HOME=/opt/jre PATH="/opt/jre/bin:$PATH"
COPY --from=builder /jre /opt/jre
COPY --from=builder /src/build/libs/app.jar /app/app.jar
CMD ["java","-jar","/app/app.jar"]
```

---

## 주요 CI/CD 플랫폼에서의 BuildKit 캐시 적용

### GitHub Actions (레지스트리 캐시 활용)

{% raw %}
```yaml
name: Build & Push with Remote Cache
on:
  push:
    branches: [ main ]
jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      - name: Set up Buildx
        uses: docker/setup-buildx-action@v3
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USER }}
          password: ${{ secrets.DOCKER_PASS }}
      - name: Build & Push (with registry cache)
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: yourname/myapp:latest
          cache-from: type=registry,ref=yourname/myapp:buildcache
          cache-to:   type=registry,ref=yourname/myapp:buildcache,mode=max
```
{% endraw %}

### GitLab CI (Docker-in-Docker 방식)

```yaml
stages: [ build ]
variables:
  IMAGE: $CI_REGISTRY_IMAGE
build:
  stage: build
  image: docker:24
  services: [ docker:dind ]
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker buildx create --use
    - docker run --privileged --rm tonistiigi/binfmt --install all
    - docker buildx build \
        --platform linux/amd64,linux/arm64 \
        --cache-from type=registry,ref=$IMAGE:buildcache \
        --cache-to   type=registry,ref=$IMAGE:buildcache,mode=max \
        -t $IMAGE:latest --push .
```

### Jenkins Pipeline

```groovy
pipeline {
  agent any
  environment {
    IMAGE = "yourname/myapp:${env.BUILD_NUMBER}"
  }
  stages {
    stage('Setup Buildx') {
      steps {
        sh 'docker buildx create --use || true'
        sh 'docker run --privileged --rm tonistiigi/binfmt --install all'
      }
    }
    stage('Build & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DU', passwordVariable: 'DP')]) {
          sh '''
            echo "$DP" | docker login -u "$DU" --password-stdin
            docker buildx build \
              --platform linux/amd64,linux/arm64 \
              --cache-from type=registry,ref=yourname/myapp:buildcache \
              --cache-to   type=registry,ref=yourname/myapp:buildcache,mode=max \
              -t yourname/myapp:${BUILD_NUMBER} -t yourname/myapp:latest \
              --push .
          '''
        }
      }
    }
  }
}
```

---

## BuildKit 보안 및 거버넌스 통합

- **비밀 정보 주입**: 반드시 `--mount=type=secret/ssh` 방식을 사용하세요. `ENV`나 `ARG`로 전달하면 이미지 레이어에 남습니다.
- **정적 분석 게이트**: PR 단계에서 Dockerfile 린터(`hadolint`), 컨테이너 취약점 스캔(`Trivy`), OPA/`Gatekeeper` 정책 검사를 통합하세요.
- **이미지 서명 및 SBOM**: `cosign`을 이용한 이미지 서명과 `syft` 또는 `docker sbom`을 통한 소프트웨어 명세서(SBOM) 생성을 파이프라인에 포함시키세요.
- **베이스 이미지 관리**: 조직에서 승인한 표준 베이스 이미지를 사용하고, 정기적으로 업데이트하는 정책을 수립하세요.

---

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| “항상 전체 빌드가 실행됨” | `COPY . .` 명령이 너무 이른 단계에 위치 | Dockerfile 구조를 의존성→소스 순서로 재구성하고, `.dockerignore` 파일을 점검하세요. |
| 멀티 플랫폼 빌드가 매우 느림 | QEMU 에뮬레이션에 의한 CPU 오버헤드 | 해당 아키텍처의 네이티브 빌드 러너를 도입하거나, 아키텍처별 CI 파이프라인을 분리하세요. |
| Inline cache가 효과가 없음 | `BUILDKIT_INLINE_CACHE=1` 빌드 인수가 없거나 이미지가 푸시되지 않음 | 빌드 인수를 설정하고, 빌드 후 반드시 이미지를 레지스트리에 푸시하세요. |
| 프라이빗 레지스트리 캐시 실패 | 인증 문제 또는 `ref` 경로 오류 | 레지스트리 로그인 상태를 확인하고, `ref` 값을 정확한 이미지 참조로 지정하세요. |
| 비밀값이 이미지 레이어에 노출됨 | `ENV`나 `ARG`를 통해 비밀값을 전달 | `RUN --mount=type=secret` 방식을 사용하여 비밀값을 안전하게 주입하세요. |
| 네이티브 모듈 컴파일 실패 | 빌드 스테이지에 필요한 컴파일 도구 누락 | `build-base`, `g++`, `python3-dev` 등의 패키지를 빌드 스테이지에 설치하세요. |

---

## 고급 활용: Bake 파일을 이용한 선언형 멀티 타겟 빌드

`docker-bake.hcl` 파일을 사용하여 여러 이미지 대상을 한 번에 정의하고 관리할 수 있습니다.

```hcl
group "default" { targets = ["app"] }

target "app" {
  context = "."
  tags = ["yourname/app:latest"]
  platforms = ["linux/amd64","linux/arm64"]
  cache-from = ["type=registry,ref=yourname/app:buildcache"]
  cache-to   = ["type=registry,ref=yourname/app:buildcache,mode=max"]
}
```

실행:
```bash
docker buildx bake --push
```
> 여러 이미지나 빌드 타겟을 **동일한 캐시 정책** 하에 한 번에 관리하고 빌드할 때 매우 유용합니다.

---

## 핵심 명령어 요약

```bash
# BuildKit 활성화
export DOCKER_BUILDKIT=1

# Buildx 빌더 설정
docker buildx create --use --name mybuilder
docker buildx inspect --bootstrap

# 레지스트리 캐시를 활용한 빌드
docker buildx build \
  --cache-from type=registry,ref=repo/app:buildcache \
  --cache-to   type=registry,ref=repo/app:buildcache,mode=max \
  -t repo/app:latest --push .

# Inline Cache 활용
docker build \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  -t repo/app:latest \
  --cache-from=repo/app:latest .

# Secret을 사용한 빌드
docker buildx build --secret id=apikey,src=./apikey.txt -t secret-demo .

# 멀티 플랫폼 빌드
docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:latest --push .
```

---

## 결론

효율적인 Docker 이미지 빌드와 캐시 관리는 개발 생산성과 CI/CD 파이프라인의 속도에 직접적인 영향을 미칩니다. BuildKit과 buildx는 이를 위한 강력한 도구를 제공합니다. 성공적인 전략을 수립하기 위한 핵심 원칙은 다음과 같습니다:

1.  **Dockerfile 설계 최적화**: 변경 빈도가 낮은 의존성 설치 단계를 앞쪽에, 자주 변경되는 애플리케이션 소스 복사 단계를 뒤쪽에 배치하세요. 이는 캐시 효율의 근간이 됩니다.
2.  **빌드 컨텍스트 관리**: `.dockerignore` 파일을 통해 불필요한 파일이 빌드 과정에 포함되지 않도록 철저히 관리하세요.
3.  **적절한 캐시 백엔드 선택**: CI/CD 환경에서는 **레지스트리 캐시(`type=registry`)** 나 **GitHub Actions 캐시(`type=gha`)** 를 활용하여 러너 간 캐시를 공유하고, 로컬 개발 환경에서는 **로컬 캐시(`type=local`)** 를 사용하세요.
4.  **멀티 스테이지 빌드 적극 활용**: 빌드 도구와 런타임을 분리하여 최종 이미지 크기를 줄이고, 빌드 단계에서만 필요한 의존성(예: 컴파일러)이 런타임 이미지에 포함되지 않도록 하세요.
5.  **보안 빌드 관행 수립**: 민감정보는 `--mount=type=secret/ssh`를 통해서만 전달하여 이미지 레이어나 로그에 노출되는 것을 근본적으로 방지하세요.
6.  **재현성(Reproducibility) 보장**: `package-lock.json`, `go.sum` 같은 Lock 파일을 사용하고, 가능한 경우 베이스 이미지를 다이제스트로 고정하여 빌드 결과의 예측 가능성을 높이세요.

이러한 원칙을 체계적으로 적용하면, 첫 번째 빌드 이후의 반복적인 빌드 작업을 **수초에서 수십 초** 내에 완료할 수 있는 빠르고 신뢰할 수 있는 CI/CD 파이프라인을 구축할 수 있습니다. BuildKit의 고급 기능을 숙지하고 프로젝트와 팀의 워크플로에 맞게 조합하는 것이 지속 가능한 개발 운영의 핵심입니다.