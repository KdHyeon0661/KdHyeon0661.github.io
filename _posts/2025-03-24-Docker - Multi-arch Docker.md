---
layout: post
title: Docker - Multi-arch Docker
date: 2025-03-24 21:20:23 +0900
category: Docker
---
# Multi-arch Docker 이미지 만들기

## 0) 왜 Multi-arch 인가? (기존 글의 핵심 + 확장)

| 항목 | 설명 |
|------|------|
| 플랫폼 다양성 | x86_64(amd64), ARM64(Graviton, M1/M2/M3), ARMv7 등 다양한 하드웨어에서 동일 이미지를 제공 |
| Dev/Prod 불일치 해소 | 개발(Mac ARM)·운영(amd64) 간 아키텍처 차이로 인한 바이너리/네이티브 모듈 문제를 사전에 제거 |
| 배포 간소화 | **하나의 레포지토리:여러 아키텍처**를 manifest list 로 제공 → 클라이언트가 자동으로 맞는 이미지를 pull |
| CI 통합 | 빌드 매트릭스(amd64/arm64)로 동일 파이프라인 운영, 캐시·보안 스캔·서명까지 자동화 |

---

## 1) 핵심 도구: BuildKit + `docker buildx`

- **Buildx**: Docker BuildKit 프론트엔드로 **멀티 플랫폼 빌드**와 **manifest list** 생성 지원
- **QEMU/binfmt**: 다른 아키텍처를 에뮬레이션하여 로컬/CI에서 크로스 빌드 가능
- **이미지 결과물**: 단일 태그(`repo:tag`) 아래 `amd64`, `arm64` 등 **여러 아키텍처 매니페스트**가 묶인 형태

---

## 2) 로컬 준비 — buildx/binfmt 초기화

```bash
# 1) buildx 확인
docker buildx version

# 2) 빌더 생성 및 활성화
docker buildx create --use --name mybuilder
docker buildx inspect --bootstrap
```

필요 시 QEMU(binfmt) 설치:

```bash
docker run --privileged --rm tonistiigi/binfmt --install all
```

> `--bootstrap`는 빌더 내부 노드에 QEMU 기반 에뮬 환경을 구성한다.

---

## 3) 첫 실습 — “가장 단순한” multi-arch 빌드

### 3.1 Dockerfile (Alpine 기반)

```dockerfile
# Dockerfile
FROM alpine:3.20
CMD ["echo", "Hello from Multi-arch!"]
```

### 3.2 멀티 플랫폼 빌드 & 푸시

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t yourname/hello-multiarch:latest \
  --push .
```

- `--platform`: 대상 아키텍처 다중 지정
- `--push`: 로컬 대신 레지스트리로 바로 밀어 manifest list 생성
- `--load`: 로컬 도커에 적재. 단, **단일 플랫폼**일 때만 사용 가능

### 3.3 결과 검증

```bash
docker buildx imagetools inspect yourname/hello-multiarch:latest
```

예상 출력: `amd64`, `arm64` 매니페스트가 포함된 list 확인.

---

## 4) 언어/런타임별 설계 패턴

### 4.1 Go (정적 바이너리, 크로스 컴파일 최적)

```dockerfile
# Dockerfile (Go multi-stage, CGO off)
# 1) Build stage
FROM --platform=$BUILDPLATFORM golang:1.23-alpine AS builder
ARG TARGETOS TARGETARCH
WORKDIR /src
COPY . .
ENV CGO_ENABLED=0 GO111MODULE=on
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /out/app ./cmd/app

# 2) Runtime stage (scratch/distroless 권장)
FROM gcr.io/distroless/static:nonroot
WORKDIR /
COPY --from=builder /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

빌드:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t yourname/go-app:1.0.0 \
  --push .
```

> Go는 `CGO_ENABLED=0`로 크로스 빌드가 쉬우며, **distroless**로 극소 이미지를 구성 가능.

---

### 4.2 Node.js (native addon 주의)

```dockerfile
# Dockerfile (Node, native module 캐시 최적화)
FROM --platform=$BUILDPLATFORM node:20-alpine AS builder
ARG TARGETOS TARGETARCH
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
# 네이티브 애드온이 있다면 arch별로 rebuild 필요
RUN npm run build

FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

> `node-gyp`·`sharp` 등 **네이티브 의존성**은 각 아키텍처별 바이너리 재빌드가 필요할 수 있다.  
> 필요 패키지(예: `python3`, `make`, `g++`)를 build 단계에서만 설치하는 **multi-stage**가 핵심.

---

### 4.3 Python (C 확장/머신러닝 주의)

```dockerfile
# Dockerfile (Python multi-stage with wheels)
FROM --platform=$BUILDPLATFORM python:3.11-alpine AS builder
WORKDIR /wheels
COPY requirements.txt .
# 네이티브 확장 모듈 대비 도구 설치
RUN apk add --no-cache build-base musl-dev linux-headers \
 && pip wheel --wheel-dir /wheels -r requirements.txt

FROM python:3.11-alpine
WORKDIR /app
COPY --from=builder /wheels /wheels
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r /wheels/requirements.txt
COPY . .
CMD ["python", "app.py"]
```

> **wheel 선빌드**로 런타임에서 컴파일을 피하고, 아키텍처별 빌드 시도 → 실패 원인(헤더/라이브러리) 분리.

---

### 4.4 Java/JVM (플랫폼 무관이지만 네이티브 lib 주의)

```dockerfile
# Dockerfile (JLink로 경량화)
FROM --platform=$BUILDPLATFORM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /src
COPY . .
RUN ./gradlew clean build -x test

# (선택) JLink로 custom JRE 생성
RUN jlink --add-modules java.base,java.logging \
          --strip-debug --compress=2 --no-header-files --no-man-pages \
          --output /customjre

FROM alpine:3.20
ENV JAVA_HOME=/opt/jre
ENV PATH="$JAVA_HOME/bin:$PATH"
COPY --from=builder /customjre /opt/jre
COPY --from=builder /src/build/libs/app.jar /app/app.jar
CMD ["java","-jar","/app/app.jar"]
```

> JVM은 바이트코드로 플랫폼 독립적이나, **JNI** 라이브러리는 아키텍처 별 빌드 필요.

---

## 5) Manifest/Tag 전략 (latest + 버전 + 다이제스트)

**권장**:  
- 가독용(`1.0.0`, `latest`) + **불변 참조용 다이제스트**를 함께 운영
- 릴리스 파이프라인(예: `:1.0.0`, `:stable`, `@sha256:...` 동시 발행)

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t yourname/app:1.0.0 \
  -t yourname/app:latest \
  --push .
```

검증:

```bash
docker buildx imagetools inspect yourname/app:1.0.0
```

---

## 6) 캐시/속도 최적화

### 6.1 레이어/캐시 전략
- `.dockerignore`로 빌드 컨텍스트 최소화
- `package*.json`/`requirements.txt` 먼저 복사 → 의존성 레이어 캐시
- Buildx 캐시 내보내기/가져오기

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=yourname/app:buildcache \
  --cache-to   type=registry,ref=yourname/app:buildcache,mode=max \
  -t yourname/app:1.0.0 --push .
```

### 6.2 CI에서 캐시 복원
- GitHub Actions: `docker/build-push-action@v5` + `cache-from/to`  
- GitLab: 레지스트리 캐시 태그 유지

---

## 7) BuildKit 고급: secrets/ssh/빌드 인수

```dockerfile
# 예: private repo pip install
# syntax=docker/dockerfile:1.7
FROM alpine:3.20
# BuildKit secret mount 예
# docker buildx build --secret id=netrc,src=$HOME/.netrc ...
RUN --mount=type=secret,id=netrc,mode=0400,target=/root/.netrc \
    apk add --no-cache curl
```

```bash
docker buildx build \
  --secret id=netrc,src=$HOME/.netrc \
  -t yourname/secret-demo:latest \
  --push .
```

빌드 인수 예:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --build-arg BUILD_TIME=$(date -u +%FT%TZ) \
  -t yourname/app:1.1.0 \
  --push .
```

---

## 8) CI 파이프라인 (확장) — GitHub / GitLab / Jenkins

### 8.1 GitHub Actions (multi-arch + 캐시 + 서명/리포트 예시)

{% raw %}
```yaml
name: Build & Push Multi-arch

on:
  push:
    branches: [ main ]
  workflow_dispatch:

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

      - name: Build & Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          platforms: linux/amd64,linux/arm64
          tags: yourname/myapp:latest, yourname/myapp:${{ github.sha }}
          cache-from: type=registry,ref=yourname/myapp:buildcache
          cache-to:   type=registry,ref=yourname/myapp:buildcache,mode=max

      # (선택) SBOM 생성/추출 또는 cosign 서명 단계 추가
```
{% endraw %}

### 8.2 GitLab CI

```yaml
stages: [ build ]
variables:
  IMAGE: $CI_REGISTRY_IMAGE
  TAGS: $CI_COMMIT_SHORT_SHA

build:
  stage: build
  image: docker:24
  services: [ docker:dind ]
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker buildx create --use
    - docker run --privileged --rm tonistiigi/binfmt --install all
    - docker buildx build --platform linux/amd64,linux/arm64 \
        -t "$IMAGE:$TAGS" -t "$IMAGE:latest" --push .
```

### 8.3 Jenkins (Pipeline)

```groovy
pipeline {
  agent any
  environment {
    IMAGE = "yourname/myapp:${env.BUILD_NUMBER}"
  }
  stages {
    stage('Buildx Setup') {
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
            docker buildx build --platform linux/amd64,linux/arm64 \
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

## 9) 로컬 테스트 — QEMU 런/바이너리 확인

```bash
# 현재 호스트 아키텍처 확인
uname -m

# 이미지 내부 arch 확인(간단)
docker run --rm -it --platform linux/arm64 alpine:3.20 uname -m
docker run --rm -it --platform linux/amd64 alpine:3.20 uname -m
```

> 에뮬레이션이 잘 동작하면 각기 `aarch64`, `x86_64` 가 출력된다.

---

## 10) 흔한 문제/트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| arm64 빌드 실패 | base 이미지가 multi-arch 미지원 | `alpine`, `debian`, `ubi` 등 다중 아키텍처 지원 베이스로 변경 |
| native 모듈 컴파일 에러 | 빌드 의존 패키지 누락 | `build-base`, `g++`, `python3`, `linux-headers` 등 빌드 단계에만 설치 |
| `--platform` + `--load` 동시 사용 | 스펙 제한 | 여러 플랫폼은 `--push`로 manifest 생성 후 pull 테스트 |
| QEMU 느림 | 에뮬레이션 오버헤드 | 가능하면 **해당 아키텍처 네이티브 러너** 사용(예: arm64 러너) |
| glibc vs musl 차이 | alpine(musl) 바이너리 불일치 | 베이스를 `debian-slim`로 바꾸거나, 동적링크 대상 맞추기 |
| 자바 JNI 실패 | 네이티브 lib 아키텍처 불일치 | arch별 빌드 또는 순수 자바 대체 라이브러리 검토 |

---

## 11) 성능/시간 대략 추정(간이)
병렬 빌드 워커가 \(k\)개, 각 아키텍처 빌드 시간 평균이 \(t\)일 때 전체 시간 \(T\)는 대략:
$$
T \approx \left\lceil \frac{N_{\text{arch}}}{k} \right\rceil \cdot t \quad (\text{캐시 미적용 근사})
$$
캐시 적중률 \(\gamma\) (0–1)일 때:
$$
T_{\text{cache}} \approx T \cdot (1 - \gamma)
$$

- 아키텍처 수 \(N_{\text{arch}}\)가 2(amd64, arm64)일 때, 러너 2개면 이상적으로 \(T \approx t\).

---

## 12) 보안·무결성 — 서명(Cosign) & SBOM

### 12.1 Cosign 서명(요약)

```bash
cosign generate-key-pair
# 서명
cosign sign --key cosign.key yourname/myapp:1.0.0
# 검증
cosign verify --key cosign.pub yourname/myapp:1.0.0
```

> Manifest list에 대해 **각 플랫폼 manifest**가 서명 대상. CI에서 자동화 권장.

### 12.2 SBOM
- `docker sbom` 또는 `syft`로 SBOM 생성 → 아티팩트로 보관/검증
- 취약점 스캔(Trivy/Scout) + 정책 게이트(Gatekeeper/Kyverno) 연계

---

## 13) Buildx Bake — 매트릭스 선언형 빌드

`docker-bake.hcl`:

```hcl
group "default" {
  targets = ["app"]
}

target "app" {
  context = "."
  tags    = ["yourname/app:latest"]
  platforms = ["linux/amd64","linux/arm64"]
  cache-from = ["type=registry,ref=yourname/app:buildcache"]
  cache-to   = ["type=registry,ref=yourname/app:buildcache,mode=max"]
}
```

실행:

```bash
docker buildx bake --push
```

> 여러 이미지/타깃을 한 번에 정의·빌드하는 데 유용.

---

## 14) 실전 템플릿 — 멀티스테이지 + 아키텍처별 스위치

```dockerfile
# syntax=docker/dockerfile:1.7
ARG RUNTIME=node:20-alpine
ARG BUILDER=node:20-alpine

FROM --platform=$BUILDPLATFORM ${BUILDER} AS builder
ARG TARGETOS TARGETARCH
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
# 아키텍처 조건 분기 예: 특정 바이너리 교체
RUN case "$TARGETARCH" in \
      "arm64") echo "ARM64 tweak";; \
      "amd64") echo "AMD64 tweak";; \
      *) echo "other";; \
    esac
RUN npm run build

FROM ${RUNTIME}
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node","dist/index.js"]
```

빌드:

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t yourname/web:1.2.0 --push .
```

---

## 15) 정책/거버넌스 (조직 운영 가이드)

- Base 이미지: 공식/기업 표준만 허용(서명 검증, 정기 갱신)
- Tag 규칙: `MAJOR.MINOR.PATCH`, `latest`는 **사내 규정**에 따라 제한
- Digest 배포: Kubernetes/Helm에는 가급적 **다이제스트 고정** 적용
- 스캔 게이트: PR/CI 단계에서 Trivy/Scout 통과 조건 설정
- 만료 정책: 오래된 태그 정리(레지스트리 수명 정책)
- 재현성: 빌드 시각/의존 고정(lockfile), 캐시 재현 가능한 설정

---

## 16) 종합 “레시피” (체크리스트)

1. 빌더/QEMU 준비: `buildx create --use` + `binfmt --install all`  
2. Dockerfile: 멀티스테이지 + 언어별 네이티브 의존성 해결  
3. `.dockerignore`: 컨텍스트 슬림화  
4. 캐시: `cache-from/to`를 레지스트리에 저장 → CI 간 재사용  
5. buildx: `--platform amd64,arm64 --push` 로 manifest list 생성  
6. 태그: 버전 + latest + (옵션) 다이제스트 고정 운용  
7. 보안: 서명(Cosign), SBOM, 취약점 스캔, 정책 게이트  
8. CI: GitHub/GitLab/Jenkins 템플릿으로 자동화  
9. 테스트: `docker run --platform ... uname -m` 로 아키 확인  
10. 문서화: 트러블슈팅(네이티브 모듈/헤더/링커), 릴리스 노트

---

## 17) 부록 — 로컬 전용(단일 아키) 빌드/검증

```bash
# 현재 호스트 아키만 빌드해 로컬로 적재
docker buildx build --platform linux/arm64 -t test/app:arm64 --load .
docker run --rm test/app:arm64 uname -m
```

> 로컬 디버깅 시 단일 아키 + `--load`로 빠르게 반복.

---

## 참고 명령 요약

```bash
# 빌더 준비
docker buildx create --use --name mybuilder
docker buildx inspect --bootstrap
docker run --privileged --rm tonistiigi/binfmt --install all

# 멀티 아키 빌드/푸시
docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:1.0.0 --push .

# 매니페스트 확인
docker buildx imagetools inspect repo/app:1.0.0

# 캐시 레지스트리 사용
docker buildx build --cache-from type=registry,ref=repo/app:buildcache \
                    --cache-to   type=registry,ref=repo/app:buildcache,mode=max \
                    --platform linux/amd64,linux/arm64 -t repo/app:1.0.1 --push .
```

---

## 마무리

- **핵심**: *멀티스테이지 + buildx + 캐시 + 보안(서명/SBOM/스캔)*  
- **언어별 포인트**: Go(크로스 쉬움), Node/Python(네이티브 확장 주의), Java(JNI 주의)  
- **운영**: 다이제스트 고정, CI 자동화, 정책 게이트로 **재현성과 신뢰성** 확보