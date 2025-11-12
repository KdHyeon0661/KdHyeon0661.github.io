---
layout: post
title: Docker - BuildKit
date: 2025-03-27 19:20:23 +0900
category: Docker
---
# BuildKit과 Docker 캐시 전략

## 0. BuildKit 한 장 요약 (기존 핵심 + 확장)

| 항목 | Legacy Builder | BuildKit |
|---|---|---|
| 실행 모델 | 단순 순차 실행 | **DAG 기반 병렬 실행**(LLB 그래프) |
| 캐시 | 제한적 (로컬 강조) | **레이어/명령 단위 정교한 캐시** + 원격 캐시 |
| 멀티 플랫폼 | 제약 | **buildx**로 완전 지원 (`--platform`) |
| 보안 비밀 | 불가 | `RUN --mount=type=secret/ssh` |
| 출력 제어 | 이미지 중심 | `--output`(image, tar, local, registry 등) |
| 로깅 | 텍스트 | 구조화 로깅 + 프로그레스 UI |
| 분산/원격 캐시 | - | **type=registry, type=gha, type=local** 등 |

**용어 미리보기**
- **LLB**(Low-Level Build): BuildKit 내부 빌드 그래프 형식  
- **Inline cache**: 이미지 안에 캐시 메타데이터를 함께 푸는 방식  
- **Cache exporter/importer**: `--cache-to/--cache-from`로 캐시 위치를 외부화

---

## 1. BuildKit 활성화 & 기본 사용

### 1.1 활성화
Linux 환경:

```bash
export DOCKER_BUILDKIT=1
```

또는 daemon 전역 설정(`/etc/docker/daemon.json`):

```json
{
  "features": {
    "buildkit": true
  }
}
```

> `docker buildx build ...`를 쓰면 BuildKit이 자동 사용된다.

### 1.2 가장 단순한 실행
```bash
DOCKER_BUILDKIT=1 docker build -t myapp:latest .
```

---

## 2. 캐시 작동 원리 이해: “레이어 해시”와 무효화 규칙

BuildKit은 **각 명령(COPY/RUN/ADD/ENV/ARG 등)** 을 레이어로 기록하고, **명령 자체 + 입력 파일 스냅샷 + 환경**으로 **해시**를 만들며, 이 해시가 같으면 **캐시 히트**가 난다. 아래는 **캐시 무효화**를 유발하는 대표 요인:

1. **명령 텍스트 변경**: `RUN apt-get update && apt-get install curl` → 중간 공백/순서/옵션 변경도 해시 변경
2. **입력 파일 변경**: `COPY package.json`의 내용 바뀜
3. **빌드 인수/환경 변경**: `ARG/ENV` 값 변경
4. **베이스 이미지 다이제스트 변경**: `FROM node:18-alpine`의 내부 digest가 달라짐
5. **시간/비결정성**: timestamp, 비결정적 도구 결과물(불가피하면 고정화)

**정리:** 캐시를 잘 쓰려면 **불변(잘 안 바뀌는 것)** 을 위에, **자주 바뀌는 것**은 아래로 배치하는 **Dockerfile 명령 순서 설계**가 핵심이다.

---

## 3. 캐시 전략 설계 — “순서”와 “컨텍스트”를 지배하라

### 3.1 자주 바뀌지 않는 의존성 먼저
**Node.js 예시**

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
# 의존성 정의만 먼저 복사 → 의존성 설치 레이어 캐시화
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
# 이제 소스 전체를 복사 → 소스 변경 시 아래 레이어만 무효화
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
CMD ["node","dist/index.js"]
```

**Python 예시**

```dockerfile
FROM python:3.11-slim AS base
WORKDIR /app

# 의존성 먼저
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 소스는 나중
COPY . .
CMD ["python","app.py"]
```

### 3.2 `.dockerignore` 로 빌드컨텍스트 최소화
```dockerignore
node_modules
.git
*.log
Dockerfile
README.md
dist
```

> 불필요 파일이 컨텍스트에 포함되면 `COPY . .` 해시가 자주 변해 **캐시 효율이 급락**한다.

---

## 4. Inline Cache & 원격 캐시(Registry/GHA/Local) — CI에서 진짜 빨라지는 법

### 4.1 Inline Cache (단일 레포에서 간단히)
이미지 자체에 캐시 메타데이터를 담는다.

```bash
docker build \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  --tag yourname/myapp:latest \
  --cache-from=yourname/myapp:latest \
  .
```

- **첫 빌드**: 캐시 없음 → 느림.  
- **두 번째부터**: `--cache-from`에 `:latest`를 지정하면, **직전 이미지 레이어**를 캐시 소스로 활용.

> **주의**: 이미지를 **push**해야 다른 머신(CI runner)에서도 캐시 활용 가능.

### 4.2 Buildx + 원격 캐시(추천)
**type=registry** 를 가장 범용적으로 사용.

```bash
docker buildx build \
  --cache-from type=registry,ref=yourname/myapp:buildcache \
  --cache-to   type=registry,ref=yourname/myapp:buildcache,mode=max \
  -t yourname/myapp:latest \
  --push .
```

- **장점**: CI 러너가 매번 새로 뜨더라도 **레지스트리 캐시**를 가져와 **즉시 가속**  
- `mode=max`로 더 많은 메타데이터 보존 → 히트 확률 상승

**GitHub Actions 캐시 백엔드(type=gha)**

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

**로컬 캐시(type=local)** — 개발자 개인 워크스테이션 최적화용

```bash
docker buildx build \
  --cache-from type=local,src=.buildx-cache \
  --cache-to   type=local,dest=.buildx-cache,mode=max \
  -t yourname/myapp:dev .
```

---

## 5. 멀티 스테이지 + BuildKit 고급 mount (secret/ssh/cache)

### 5.1 Secret mount — 비밀을 레이어에 남기지 않기
```dockerfile
# syntax=docker/dockerfile:1.7
FROM alpine:3.20
RUN --mount=type=secret,id=apikey \
    sh -c 'apk add --no-cache curl && curl -H "X-API-Key: $(cat /run/secrets/apikey)" https://api.example.com'
```

빌드 시:
```bash
docker buildx build \
  --secret id=apikey,src=./apikey.txt \
  -t secret-demo:latest .
```

> **장점**: 비밀값이 이미지 레이어/로그에 남지 않는다.

### 5.2 SSH mount — private git 접근
```dockerfile
# syntax=docker/dockerfile:1.7
FROM alpine:3.20
RUN apk add --no-cache git openssh
RUN --mount=type=ssh git clone git@github.com:org/private-repo.git
```

빌드 시:
```bash
docker buildx build --ssh default -t ssh-demo:latest .
```

### 5.3 캐시 mount — 툴 체인 캐시 디렉토리 고정
```dockerfile
# npm 캐시 예
RUN --mount=type=cache,target=/root/.npm \
    npm ci
```

> 반복 빌드에서 네트워크 비용/설치 시간이 대폭 절감.

---

## 6. 멀티 플랫폼(amd64/arm64) + 캐시 — 빠르고, 어디서나

```bash
docker buildx create --use --name mybuilder
docker buildx inspect --bootstrap
docker run --privileged --rm tonistiigi/binfmt --install all

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=yourname/myapp:buildcache \
  --cache-to   type=registry,ref=yourname/myapp:buildcache,mode=max \
  -t yourname/myapp:latest \
  --push .
```

- **멀티 플랫폼**은 보통 `--push`로 **매니페스트 리스트**를 만들며 배포
- 로컬 적재(`--load`)는 **단일 플랫폼**에서만 가능  
- 성능을 원하면 **아키텍처별 네이티브 러너**(예: ARM64 러너) 도입 검토

---

## 7. 언어별 캐시 최적화 템플릿

### 7.1 Go (크로스 컴파일 쉬움, distroless 런타임)
```dockerfile
# 1. Build
FROM --platform=$BUILDPLATFORM golang:1.23-alpine AS builder
ARG TARGETOS TARGETARCH
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
ENV CGO_ENABLED=0
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /out/app ./cmd/app

# 2. Runtime
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

### 7.2 Python (wheel 선빌드)
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

### 7.3 Java (JLink로 런타임 축소)
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

## 8. CI/CD 통합 (GitHub/GitLab/Jenkins) — 캐시를 파이프라인의 “첫급유소”로

### 8.1 GitHub Actions (레지스트리 캐시)
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

### 8.2 GitLab CI (Docker-in-Docker)
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

### 8.3 Jenkins Pipeline
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

## 9. 수식으로 보는 “캐시가 주는 시간 이득”

이미지 빌드 총 시간 \(T\)는 캐시 미사용 시간 \(T_0\)에 캐시 적중률 \(\gamma \in [0,1]\)을 곱한 실제 실행 시간 \(T\_{\text{exec}}\)의 합으로 근사할 수 있다.

$$
T \approx (1-\gamma)\,T_0 + \gamma\,\epsilon
$$

여기서 \(\epsilon\)은 캐시 검증/메타데이터 다운로드 오버헤드(보통 매우 작음).  
**캐시 적중률 \(\gamma\)를 높이는 설계(정적 의존 먼저, `.dockerignore`, 원격 캐시)는 곧 \(T\)를 선형적으로 줄인다.**

---

## 10. 재현성/결정성(Determinism) 팁

- **버전 고정**: `package-lock.json`/`poetry.lock`/`go.mod` 등 lock 파일 사용  
- **시간 영향 최소화**: 빌드 타임스탬프 삽입 시 `LABEL` 등 별도 레이어 또는 빌드 아규먼트로 제어  
- **베이스 이미지 digest 사용**: `FROM node@sha256:...`  
- **아카이브/압축 도구 옵션 고정**: 순서/mtime 고정 옵션(가능한 경우)으로 결과물 해시 일관성 유지

---

## 11. 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| “항상 풀빌드” | `COPY . .`가 너무 일찍 실행 | 의존→소스 순서 재구성, `.dockerignore` 보강 |
| 멀티 플랫폼에서 느림 | 에뮬레이션(QEMU) 과부하 | 아키텍처 네이티브 러너 도입 or CI 분리 |
| inline cache가 안 먹힘 | 푸시/`--build-arg` 미사용 | `BUILDKIT_INLINE_CACHE=1` + 이미지 push |
| private registry 캐시 실패 | 권한/경로 문제 | `ref`를 정확히, 로그인 상태 확인 |
| 비밀이 레이어에 남음 | ENV/ARG로 주입 | `RUN --mount=type=secret` 로 전환 |
| 네이티브 모듈 컴파일 실패 | 빌드 도구/헤더 부재 | `build-base`/`g++`/`python3` 등 스테이지에 설치 |

---

## 12. 고급: Bake 파일로 선언형 멀티 타깃 빌드

`docker-bake.hcl`:

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

> 다수 이미지/타깃을 한 번에, **동일 캐시 정책**으로 관리.

---

## 13. 보안/거버넌스와 BuildKit

- **비밀 주입은 mount만**: `--mount=type=secret/ssh` (ENV로 넣지 않기)  
- **정책 게이트**: PR 단계에서 Dockerfile 린트(Dockle), 취약점 스캔(Trivy/Scout), OPA/Gatekeeper  
- **서명/SBOM**: cosign 서명, `docker sbom`/`syft`로 SBOM 생성 → 아티팩트 업로드  
- **기반 이미지 규정**: 조직 표준 베이스 + 정기 갱신

---

## 14. 종합 예제 — “Node 앱” 완전체 (캐시/시크릿/멀티플랫폼)

```dockerfile
# syntax=docker/dockerfile:1.7

########################
# 1. deps (캐시 최대화)
########################
FROM --platform=$BUILDPLATFORM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
# npm 캐시 mount로 반복 속도 향상
RUN --mount=type=cache,target=/root/.npm npm ci

########################
# 2. build (소스 변화만 빌드)
########################
FROM --platform=$BUILDPLATFORM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

########################
# 3. runtime (슬림)
########################
FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/dist ./dist
# 비밀이 필요하면 runtime이 아닌 build에서만 사용하도록 설계
CMD ["node","dist/index.js"]
```

멀티 플랫폼 + 원격 캐시 빌드 & 푸시:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=yourname/app:buildcache \
  --cache-to   type=registry,ref=yourname/app:buildcache,mode=max \
  -t yourname/app:1.0.0 -t yourname/app:latest \
  --push .
```

---

## 15. 참고 명령 스니펫 모음

```bash
# BuildKit on
export DOCKER_BUILDKIT=1

# Buildx builder
docker buildx create --use --name mybuilder
docker buildx inspect --bootstrap

# 레지스트리 캐시
docker buildx build \
  --cache-from type=registry,ref=repo/app:buildcache \
  --cache-to   type=registry,ref=repo/app:buildcache,mode=max \
  -t repo/app:latest --push .

# Inline cache
docker build \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  -t repo/app:latest \
  --cache-from=repo/app:latest .

# Secret
docker buildx build --secret id=apikey,src=./apikey.txt -t secret-demo .

# 멀티 플랫폼
docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:latest --push .
```

---

## 16. 결론 요약

| 축 | 핵심 실천 |
|---|---|
| Dockerfile 순서 | **의존 → 소스** 순으로 배치, `COPY . .`는 최대한 뒤로 |
| 컨텍스트 관리 | `.dockerignore`로 노이즈 제거 |
| 캐시 백엔드 | **registry/GHA/local** 조합, CI에서 지속 재사용 |
| 멀티 스테이지 | 빌드 산출물만 런타임에 포함하여 캐시·용량 최적화 |
| 보안 빌드 | `--mount=type=secret/ssh` 로 비밀/키 관리 |
| 멀티 플랫폼 | buildx + (가능하면) 네이티브 러너 |
| 재현성 | lockfile, digest, 시간/비결정성 제거 |
| 품질 게이트 | 린트/스캔/서명/SBOM을 파이프라인에 내장 |

위 원칙들을 일관되게 적용하면, **첫 빌드 이후**에는 “수 초~수십 초” 단위로 **신뢰 가능한 빠른 빌드**가 반복 가능한 생산 체계가 된다.

---

## 참고 문서
- BuildKit: https://docs.docker.com/build/buildkit/
- Dockerfile Best Practices: https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
- docker/build-push-action: https://github.com/docker/build-push-action
- buildx: https://github.com/docker/buildx