---
layout: post
title: Docker - Multi-arch Docker
date: 2025-03-24 21:20:23 +0900
category: Docker
---
# Multi-arch Docker 이미지 만들기

## 왜 Multi-arch 인가?

현대의 개발 및 배포 환경은 다양한 아키텍처를 포함합니다. MacBook의 Apple Silicon(M1/M2/M3), 클라우드의 AWS Graviton 인스턴스, 전통적인 x86_64 서버 등 다양한 플랫폼에서 동일한 애플리케이션이 실행되어야 합니다. Multi-arch 이미지는 이러한 다양성을 효과적으로 관리하기 위한 핵심 기술입니다.

**Multi-arch의 주요 이점:**

1. **플랫폼 다양성 대응**: 단일 이미지 태그로 x86_64(amd64), ARM64, ARMv7 등 다양한 하드웨어 아키텍처를 지원할 수 있습니다.

2. **개발과 운영 환경의 불일치 해소**: 개발자가 ARM 기반 Mac에서 개발한 애플리케이션이 x86_64 프로덕션 환경에서도 정상적으로 실행되도록 보장합니다.

3. **배포 간소화**: 사용자는 아키텍처를 신경 쓰지 않고 `docker pull myapp:latest`만 실행하면, 도커 엔진이 자동으로 해당 플랫폼에 맞는 이미지를 선택합니다.

4. **CI/CD 통합 효율화**: 단일 빌드 파이프라인으로 모든 아키텍처를 위한 이미지를 생성할 수 있어 관리 부담이 줄어듭니다.

## 핵심 개념 이해

### 매니페스트 리스트(Manifest List)
Multi-arch 이미지의 핵심은 매니페스트 리스트입니다. 이는 단일 태그 아래 여러 아키텍처별 이미지 매니페스트를 그룹화한 메타데이터입니다. 사용자가 이미지를 풀할 때 도커 클라이언트는 호스트의 아키텍처를 확인하고 매니페스트 리스트에서 적절한 이미지를 선택합니다.

### BuildKit과 Buildx
Docker BuildKit은 향상된 빌드 엔진이며, Buildx는 이를 위한 CLI 플러그인입니다. Buildx는 멀티 플랫폼 빌드와 매니페스트 관리를 단순화합니다.

### QEMU 에뮬레이션
로컬 머신이나 CI에서 다른 아키텍처의 이미지를 빌드하려면 QEMU를 통한 에뮬레이션이 필요합니다. binfmt_misc를 통해 다른 아키텍처의 바이너리를 투명하게 실행할 수 있습니다.

---

## 시작하기: 로컬 환경 설정

### Buildx 설치 및 설정

```bash
# Buildx가 이미 설치되어 있는지 확인
docker buildx version

# 새로운 빌더 인스턴스 생성 및 활성화
docker buildx create --use --name multiarch-builder

# 빌더 상태 확인
docker buildx inspect --bootstrap
```

### QEMU 에뮬레이터 설치

```bash
# 모든 지원되는 아키텍처에 대한 QEMU 에뮬레이터 등록
docker run --privileged --rm tonistiigi/binfmt --install all

# 설치된 에뮬레이터 확인
ls /proc/sys/fs/binfmt_misc/ | grep qemu
```

---

## 첫 번째 Multi-arch 이미지 만들기

### 간단한 Dockerfile 작성

```dockerfile
# Dockerfile
FROM alpine:3.20
RUN echo "Building for $(uname -m)" > /arch.txt
CMD ["cat", "/arch.txt"]
```

### 멀티 플랫폼 빌드 및 푸시

```bash
# 여러 아키텍처를 위한 이미지 빌드 및 레지스트리에 푸시
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t yourusername/hello-multiarch:latest \
  --push .
```

**명령어 옵션 설명:**
- `--platform`: 대상 플랫폼 지정 (쉼표로 구분)
- `--push`: 빌드 후 바로 레지스트리에 푸시 (매니페스트 리스트 생성)
- `--load`: 단일 아키텍처 이미지를 로컬 도커에 로드 (멀티플랫폼과 호환되지 않음)

### 결과 검증

```bash
# 매니페스트 리스트 확인
docker buildx imagetools inspect yourusername/hello-multiarch:latest

# 특정 아키텍처의 이미지 실행 테스트
docker run --rm --platform linux/amd64 yourusername/hello-multiarch:latest
docker run --rm --platform linux/arm64 yourusername/hello-multiarch:latest
```

---

## 언어별 Multi-arch 빌드 패턴

### Go: 정적 링크로 크로스 컴파일

Go는 정적 링크에 우수한 지원을 제공하여 Multi-arch 빌드가 비교적 간단합니다.

```dockerfile
# 빌더 스테이지
FROM --platform=$BUILDPLATFORM golang:1.23-alpine AS builder

# 대상 아키텍처 정보를 빌드 인자로 전달
ARG TARGETOS
ARG TARGETARCH

WORKDIR /src
COPY . .

# 크로스 컴파일 설정
ENV CGO_ENABLED=0 GO111MODULE=on
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build \
    -trimpath \
    -ldflags="-s -w" \
    -o /out/app ./cmd/app

# 런타임 스테이지 (scratch 사용)
FROM scratch
WORKDIR /

# CA 인증서 복사 (HTTPS 통신에 필요)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# 애플리케이션 바이너리 복사
COPY --from=builder /out/app /app

# 비루트 사용자로 실행
USER 65532:65532
ENTRYPOINT ["/app"]
```

### Node.js: 네이티브 애드온 관리

Node.js 애플리케이션은 네이티브 애드온(C++ 확장)이 있을 때 주의가 필요합니다.

```dockerfile
# 빌더 스테이지
FROM --platform=$BUILDPLATFORM node:20-alpine AS builder

ARG TARGETOS
ARG TARGETARCH

WORKDIR /app

# 패키지 파일 복사 및 설치 (레이어 캐싱 최적화)
COPY package*.json ./
RUN npm ci

# 소스 코드 복사 및 빌드
COPY . .
RUN npm run build

# 런타임 스테이지
FROM node:20-alpine
WORKDIR /app

ENV NODE_ENV=production

# 빌더에서 필요한 파일만 복사
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

CMD ["node", "dist/index.js"]
```

**네이티브 애드온이 있는 경우 추가 작업:**
```dockerfile
# 네이티브 애드온 빌드를 위한 의존성 설치
RUN apk add --no-cache python3 make g++ linux-headers

# 아키텍처별 재빌드 필요 시
RUN npm rebuild --arch=$TARGETARCH
```

### Python: C 확장 모듈 처리

Python의 C 확장 모듈은 각 아키텍처별로 컴파일되어야 합니다.

```dockerfile
# 빌더 스테이지 (wheel 미리 빌드)
FROM --platform=$BUILDPLATFORM python:3.11-alpine AS builder

WORKDIR /wheels

# 빌드 의존성 설치
RUN apk add --no-cache \
    build-base \
    musl-dev \
    libffi-dev \
    openssl-dev

# 요구사항 파일 복사 및 wheel 빌드
COPY requirements.txt .
RUN pip wheel --wheel-dir /wheels -r requirements.txt

# 런타임 스테이지
FROM python:3.11-alpine
WORKDIR /app

# 미리 빌드된 wheel 파일 복사
COPY --from=builder /wheels /wheels

# wheel에서 패키지 설치 (컴파일 불필요)
RUN pip install --no-cache-dir \
    --no-index \
    --find-links=/wheels \
    -r /wheels/requirements.txt

# 애플리케이션 코드 복사
COPY . .

CMD ["python", "app.py"]
```

### Java/JVM: 플랫폼 독립적이지만 JNI 주의

Java 바이트코드는 플랫폼 독립적이지만, JNI(Java Native Interface)를 사용하는 경우 주의가 필요합니다.

```dockerfile
# 빌더 스테이지
FROM --platform=$BUILDPLATFORM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /src
COPY . .

# 애플리케이션 빌드
RUN ./gradlew clean build -x test

# 최소 JRE 생성 (jlink 사용)
RUN $JAVA_HOME/bin/jlink \
    --add-modules java.base,java.logging,java.sql \
    --strip-debug \
    --compress=2 \
    --no-header-files \
    --no-man-pages \
    --output /customjre

# 런타임 스테이지
FROM alpine:3.20

# JRE 환경 설정
ENV JAVA_HOME=/opt/jre
ENV PATH="$JAVA_HOME/bin:$PATH"

# 커스텀 JRE 및 애플리케이션 복사
COPY --from=builder /customjre /opt/jre
COPY --from=builder /src/build/libs/app.jar /app/app.jar

CMD ["java", "-jar", "/app/app.jar"]
```

---

## 태그 및 버전 관리 전략

효과적인 Multi-arch 배포를 위한 태그 전략:

### 권장 태그 패턴

```bash
# 버전 태그와 latest 태그 동시 푸시
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t yourusername/app:1.0.0 \
  -t yourusername/app:latest \
  --push .

# 다이제스트(Digest) 기반 불변 참조
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t yourusername/app:1.0.0 \
  -t yourusername/app:latest \
  -t yourusername/app@$(date +%Y%m%d-%H%M%S) \
  --push .
```

### 매니페스트 검증

{% raw %}
```bash
# 태그에 포함된 모든 아키텍처 확인
docker buildx imagetools inspect yourusername/app:1.0.0

# 특정 아키텍처의 다이제스트 확인
docker buildx imagetools inspect yourusername/app:1.0.0 \
  --format "{{ json . }}"
```
{% endraw %}

---

## 빌드 성능 최적화

### 캐시 전략

```bash
# 레지스트리 기반 캐시 사용
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=yourusername/app:buildcache \
  --cache-to type=registry,ref=yourusername/app:buildcache,mode=max \
  -t yourusername/app:1.1.0 \
  --push .
```

### 로컬 캐시 활용

```bash
# 로컬 캐시와 레지스트리 캐시 조합
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=local,src=/tmp/.buildx-cache \
  --cache-from type=registry,ref=yourusername/app:buildcache \
  --cache-to type=local,dest=/tmp/.buildx-cache.new \
  --cache-to type=registry,ref=yourusername/app:buildcache,mode=max \
  -t yourusername/app:1.2.0 \
  --push .
```

---

## CI/CD 파이프라인 통합

### GitHub Actions 워크플로우

{% raw %}
```yaml
name: Build and Push Multi-arch Docker Image

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          install: true

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata for Docker
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

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          platforms: linux/amd64,linux/arm64
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```
{% endraw %}

### GitLab CI 설정

```yaml
variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""

stages:
  - build

build-multiarch:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker buildx create --use --name builder
    - docker run --privileged --rm tonistiigi/binfmt --install all
  script:
    - |
      docker buildx build \
        --platform linux/amd64,linux/arm64 \
        -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA" \
        -t "$CI_REGISTRY_IMAGE:latest" \
        --push .
  after_script:
    - docker buildx rm builder
```

---

## 테스트 및 검증

### 로컬 테스트

```bash
# 호스트 아키텍처 확인
uname -m

# 특정 아키텍처로 컨테이너 실행 테스트
docker run --rm --platform linux/arm64 alpine:3.20 uname -m
# 출력: aarch64

docker run --rm --platform linux/amd64 alpine:3.20 uname -m
# 출력: x86_64

# 멀티아키 이미지의 모든 아키텍처 테스트
for arch in amd64 arm64; do
  echo "Testing $arch..."
  docker run --rm --platform linux/$arch yourusername/app:latest echo "Hello from $arch"
done
```

### 통합 테스트

```dockerfile
# Dockerfile에 헬스체크 추가
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

---

## 일반적인 문제와 해결 방법

### 문제 1: 베이스 이미지가 멀티아키텍처를 지원하지 않음

**증상**: 특정 아키텍처에서 빌드 실패, "no matching manifest" 오류

**해결**:
- 공식 이미지 사용: `alpine`, `debian`, `ubuntu` 등 공식 이미지는 일반적으로 멀티아키텍처를 지원합니다.
- 지원 여부 확인:
  ```bash
  docker manifest inspect alpine:3.20 | grep architecture
  ```

### 문제 2: 네이티브 모듈 컴파일 실패

**증상**: C/C++ 확장 모듈 빌드 실패, 헤더 파일 누락 오류

**해결**:
- 빌드 의존성 명시적 설치:
  ```dockerfile
  # Alpine
  RUN apk add --no-cache build-base musl-dev linux-headers
  
  # Debian/Ubuntu
  RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential \
      pkg-config
  ```

### 문제 3: QEMU 에뮬레이션 속도 저하

**증상**: ARM 아키텍처 빌드가 매우 느림

**해결**:
- 네이티브 러너 사용: CI 파이프라인에서 아키텍처별 네이티브 러너 활용
- 캐시 최적화: 빌드 캐시를 레지스트리에 저장하여 재사용
- 필요한 아키텍처만 빌드: 개발 단계에서는 호스트 아키텍처만 빌드

### 문제 4: glibc vs musl 호환성 문제

**증상**: Alpine(musl)에서 빌드한 바이너리가 다른 배포판에서 실행되지 않음

**해결**:
- 일관된 베이스 이미지 사용: 모든 스테이지에서 동일한 libc 사용
- 정적 링크: 가능한 경우 정적 링크 사용 (Go의 `CGO_ENABLED=0`)

---

## 보안 고려사항

### 이미지 서명 (Cosign)

```bash
# 키페어 생성
cosign generate-key-pair

# 멀티아키 이미지 서명
cosign sign --key cosign.key yourusername/app:1.0.0

# 서명 검증
cosign verify --key cosign.pub yourusername/app:1.0.0
```

### SBOM(Software Bill of Materials) 생성

```bash
# Syft로 SBOM 생성
syft yourusername/app:1.0.0 -o spdx-json > sbom.json

# Docker Scout 활용
docker scout sbom yourusername/app:1.0.0
```

### 취약점 스캔 통합

```bash
# Trivy로 멀티아키 이미지 스캔
trivy image --scanners vuln \
  --platform linux/amd64 \
  yourusername/app:1.0.0

trivy image --scanners vuln \
  --platform linux/arm64 \
  yourusername/app:1.0.0
```

---

## 고급 기법: Bake 파일을 통한 선언적 빌드

`docker-bake.hcl` 파일을 사용하여 빌드 구성을 선언적으로 관리할 수 있습니다.

```hcl
# docker-bake.hcl
variable "TAG" {
  default = "latest"
}

variable "REGISTRY" {
  default = "yourusername"
}

group "default" {
  targets = ["app"]
}

target "app" {
  context = "."
  dockerfile = "Dockerfile"
  tags = [
    "${REGISTRY}/app:${TAG}",
    "${REGISTRY}/app:latest"
  ]
  platforms = [
    "linux/amd64",
    "linux/arm64"
  ]
  cache-from = ["type=registry,ref=${REGISTRY}/app:buildcache"]
  cache-to = ["type=registry,ref=${REGISTRY}/app:buildcache,mode=max"]
  args = {
    BUILDKIT_INLINE_CACHE = "1"
  }
}

target "test" {
  inherits = ["app"]
  platforms = ["linux/amd64"]  # 테스트는 단일 아키텍처로
  output = ["type=docker"]
}
```

사용 방법:
```bash
# 모든 타겟 빌드 및 푸시
docker buildx bake --push

# 특정 타겟만 빌드
docker buildx bake app --push

# 변수 오버라이드
docker buildx bake --set app.tags=yourusername/app:2.0.0 --push
```

---

## 운영 모범 사례

### 조직적 규칙

1. **베이스 이미지 표준화**: 조직 내에서 승인된 베이스 이미지 목록 유지
2. **태그 정책**: 의미 있는 버전 태그 사용, `latest` 태그의 신중한 활용
3. **다이제스트 고정**: 프로덕션 배포 시에는 다이제스트를 고정하여 불변성 보장
4. **정기 점검**: 오래된 이미지 태그 정리, 보안 업데이트 적용

### 모니터링과 알림

{% raw %}
```bash
# 이미지 업데이트 확인
docker buildx imagetools inspect yourusername/app:latest

# 아키텍처별 크기 모니터링
docker buildx imagetools inspect yourusername/app:latest \
  --format '{{range .Manifests}}{{.Digest}} {{.Platform}} {{.Size}} bytes{{"\n"}}{{end}}'
```
{% endraw %}

### 재현 가능한 빌드

```dockerfile
# 빌드 타임스탬프 및 아키텍처 정보 주입
ARG BUILD_DATE
ARG TARGETARCH
ARG TARGETOS

LABEL org.opencontainers.image.created=$BUILD_DATE \
      org.opencontainers.image.authors="your-team" \
      org.opencontainers.image.url="https://example.com/app" \
      org.opencontainers.image.source="https://github.com/yourorg/app" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.revision=$GIT_COMMIT \
      org.opencontainers.image.arch=$TARGETARCH \
      org.opencontainers.image.os=$TARGETOS
```

---

## 성능 고려사항

멀티아키텍처 빌드의 성능을 이해하는 것이 중요합니다.

### 병렬 빌드 효율성

멀티아키텍처 빌드는 기본적으로 병렬로 실행됩니다. 빌드 시간은 다음 요소에 영향을 받습니다:

- **아키텍처 수**: 아키텍처가 많을수록 총 빌드 시간 증가
- **캐시 효율성**: 레이어 재사용 가능성이 높을수록 빌드 시간 감소
- **네이티브 vs 에뮬레이션**: 네이티브 아키텍처 빌드가 에뮬레이션보다 빠름

### 리소스 요구사항

멀티아키텍처 빌드는 단일 아키텍처 빌드보다 더 많은 리소스를 사용합니다:
- **메모리**: 각 아키텍처 빌드에 대한 별도의 빌드 컨텍스트
- **디스크 공간**: 중간 이미지 레이어 저장
- **네트워크 대역폭**: 레지스트리 캐시 업로드/다운로드

---

## 결론

멀티아키텍처 Docker 이미지 빌드는 현대적인 개발 및 배포 워크플로우의 필수 요소가 되었습니다. 다양한 하드웨어 플랫폼이 공존하는 환경에서 일관된 배포 경험을 제공하기 위해 반드시 마스터해야 할 기술입니다.

효과적인 멀티아키텍처 전략을 구현하기 위한 핵심 원칙을 정리하면:

1. **적절한 도구 활용**: Buildx와 BuildKit을 활용하여 멀티 플랫폼 빌드를 단순화하세요.
2. **언어별 특성 이해**: Go, Node.js, Python, Java 등 각 언어의 크로스 컴파일 특성을 이해하고 적절히 대응하세요.
3. **캐시 전략 수립**: 레지스트리 기반 캐시를 활용하여 빌드 성능을 최적화하세요.
4. **보안 통합**: 이미지 서명, SBOM 생성, 취약점 스캔을 빌드 파이프라인에 통합하세요.
5. **테스트 자동화**: 모든 아키텍처에 대한 테스트를 자동화하여 호환성 문제를 조기에 발견하세요.
6. **운영 규칙 수립**: 태그 정책, 베이스 이미지 표준, 다이제스트 고정 등 조직적 규칙을 수립하세요.

멀티아키텍처 지원은 처음에는 복잡해 보일 수 있지만, 일단 파이프라인이 구축되면 다양한 플랫폼에서의 배포 관리가 훨씬 간소화됩니다. 점진적으로 접근하여 먼저 주요 아키텍처(예: amd64와 arm64)부터 지원을 시작하고, 필요에 따라 추가 아키텍처로 확장하는 것이 좋은 접근법입니다.

최종적으로 멀티아키텍처 이미지는 기술적 구현을 넘어 팀의 생산성, 애플리케이션의 접근성, 그리고 조직의 기술적 민첩성을 높이는 전략적 투자입니다.