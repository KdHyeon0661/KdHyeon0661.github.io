---
layout: post
title: Docker - Docker 경량화 이미지 만들기
date: 2025-03-10 20:20:23 +0900
category: Docker
---
# Docker 경량화 이미지 만들기 (alpine 포함)

본 문서는 컨테이너 이미지를 경량화하는 포괄적인 가이드입니다. Alpine, Slim, Distroless 이미지를 비교하고, 멀티스테이지 빌드 전략을 포함하여 언어별 최적화 방법, 캐시 관리, 보안 검증, 디버깅 전술까지 실제 적용 가능한 내용을 상세히 다룹니다.

---

## 경량화의 목적과 한계

### 왜 이미지를 경량화해야 하는가?

경량화된 이미지를 만드는 것은 단순히 저장 공간을 절약하는 것을 넘어 여러 운영적 이점을 제공합니다:

1. **배포 및 스케일 속도 향상**: 작은 이미지는 네트워크를 통해 더 빠르게 전송되며, 레지스트리에서 풀링하는 시간이 단축됩니다. CI/CD 파이프라인에서 캐시 적중률을 높여 빌드 시간을 줄입니다.

2. **저장소 비용 절감**: 사설 레지스트리 사용 비용은 저장된 이미지 크기에 비례하는 경우가 많습니다. 작은 이미지는 저장 비용을 직접적으로 줄여줍니다.

3. **공격 표면 축소**: 불필요한 패키지와 도구를 제거하면 잠재적 취약점의 수가 기하급수적으로 감소합니다. 공격자가 이용할 수 있는 벡터가 줄어듭니다.

4. **운영 단순화**: 필요한 구성 요소만 포함된 이미지는 이해하기 쉽고, 문제 발생 시 디버깅 범위를 축소시킵니다.

### Alpine 이미지의 함정

Alpine Linux는 가볍고 보안에 중점을 둔 배포판으로 유명하지만, 모든 상황에 적합한 것은 아닙니다:

- **musl vs glibc 호환성 문제**: Alpine은 musl libc를 사용하는 반면, 많은 Linux 배포판은 glibc를 사용합니다. 일부 네이티브 확장이나 사전 컴파일된 바이너리(wheel 파일, .so 라이브러리)가 glibc를 가정하고 빌드된 경우, Alpine에서 실행 시 문제가 발생할 수 있습니다.

- **디버깅 어려움**: Alpine은 기본적으로 최소한의 도구만 포함하므로, 문제 발생 시 진단에 필요한 유틸리티가 부족할 수 있습니다.

- **빌드 복잡성 증가**: 네이티브 의존성이 있는 패키지의 경우, Alpine에서 소스부터 컴파일해야 할 수 있어 빌드 시간이 길어지고 복잡성이 증가합니다.

**핵심 원칙**: 호환성이 크기보다 중요한 경우에는 `slim`(Debian/Ubuntu 기반) 버전이나 언어 전용 경량 런타임을 먼저 고려하세요.

---

## 베이스 이미지 선택 전략

다양한 베이스 이미지 옵션을 비교하고 상황에 맞는 선택 방법을 알아봅니다:

| 분류 | 예시 | 장점 | 주의점 | 적합한 사용 사례 |
|------|------|------|--------|------------------|
| Full | `python:3.11`, `node:20` | 최고의 호환성, 디버깅 용이 | 이미지 크기가 매우 큼 | 초기 개발, 테스트 환경 |
| Slim | `python:3.11-slim` | glibc 기반 호환성 유지, 상대적 경량화 | 컴파일러, 헤더 파일 등 부족 | 대부분의 프로덕션 환경 |
| Alpine | `python:3.11-alpine` | 매우 작은 크기, 빠른 시작 | musl 이슈, 빌드 도구 필요 | 호환성 검증 완료된 환경 |
| Distroless | `gcr.io/distroless/python3` | 극도로 작음, 공격 표면 최소화 | 셸 없음, 디버깅 매우 어려움 | 높은 보안 요구사항 환경 |
| Scratch | `scratch` | 완전히 비어있는 베이스 | 모든 것이 직접 포함되어야 함 | 정적 링크된 Go 애플리케이션 |

### 선택 가이드라인

1. **네이티브 확장 패키지 사용 시**: `psycopg2`, `lxml`, `numpy`, `Pillow`와 같은 패키지를 사용한다면 먼저 `slim` 버전으로 시도하세요.

2. **확실한 호환성 검증 후**: Alpine의 호환성을 철저히 테스트한 후, 크기 절감이 중요한 경우에만 Alpine으로 전환하세요.

3. **보안 최우선 환경**: 보안이 최우선이며 런타임만 필요한 경우 Distroless를 고려하세요. 단, 디버깅 전략을 반드시 마련해야 합니다.

4. **정적 링크 가능한 언어**: Go와 같은 언어로 정적 링크된 바이너리를 만들 수 있다면 `scratch`를 고려할 수 있습니다.

---

## 멀티스테이지 빌드: 기본부터 고급까지

### 기본 패턴: 컴파일러 분리

```dockerfile
# 빌더 스테이지: 모든 빌드 도구와 의존성 포함
FROM python:3.11-alpine AS builder
RUN apk add --no-cache gcc musl-dev libffi-dev openssl-dev
WORKDIR /w
COPY requirements.txt .

# 별도의 설치 경로에 패키지 설치
RUN pip install --prefix=/install --no-cache-dir -r requirements.txt

# 런타임 스테이지: 최소한의 환경만 유지
FROM python:3.11-alpine
WORKDIR /app

# 빌더에서 생성된 패키지만 복사
COPY --from=builder /install /usr/local
COPY . .
CMD ["python", "app.py"]
```

### 고급 패턴: 호환성과 경량화의 균형

```dockerfile
# 빌더: 호환성 우선 (glibc 기반)
FROM python:3.11-slim AS builder
WORKDIR /w
COPY requirements.txt .
RUN pip install --upgrade pip \
 && pip install --prefix=/install --no-cache-dir -r requirements.txt
COPY . .

# 런타임: 보안 강화 (Distroless)
FROM gcr.io/distroless/python3-debian12:nonroot
WORKDIR /app
COPY --from=builder /install /usr/local
COPY --from=builder /w /app
USER nonroot
EXPOSE 8080
CMD ["app.py"]
```

### 멀티스테이지 빌드의 핵심 원칙

1. **분리된 책임**: 빌드 도구와 컴파일러는 빌더 스테이지에만 존재해야 합니다.
2. **선택적 복사**: 최종 이미지에는 애플리케이션 실행에 필요한 파일만 복사합니다.
3. **계층화된 접근**: 다른 베이스 이미지를 각 스테이지에 사용하여 호환성과 경량화를 동시에 달성합니다.

---

## 언어별 최적화 레시피

### Python: musl 호환성과 과학 계산 패키지 대응

Alpine에서 Python 애플리케이션을 빌드할 때 주의해야 할 점:

```dockerfile
FROM python:3.11-alpine

# 필요한 빌드 의존성 추가 (목적별)
RUN apk add --no-cache \
    build-base \          # 기본 컴파일러 도구
    musl-dev \           # musl C 라이브러리 개발 파일
    libffi-dev \         # Foreign Function Interface
    openssl-dev \        # 암호화 라이브러리
    postgresql-dev \     # PostgreSQL 클라이언트 라이브러리
    jpeg-dev \           # 이미지 처리
    zlib-dev \           # 압축 라이브러리
    freetype-dev \       # 폰트 렌더링
    libxml2-dev \        # XML 처리
    libxslt-dev          # XSLT 변환

# 임시 빌드 의존성 패턴 (정리 용이)
RUN apk add --no-cache --virtual .build-deps \
    build-base musl-dev libffi-dev openssl-dev \
 && pip install --no-cache-dir -r requirements.txt \
 && apk del .build-deps
```

**Python 경량화 팁**:
- `PYTHONDONTWRITEBYTECODE=1` 환경 변수를 설정하여 .pyc 파일 생성을 방지하세요.
- `pip install` 시 `--no-cache-dir` 옵션을 사용하여 pip 캐시를 이미지에 포함하지 마세요.
- `pip-compile`(pip-tools)을 사용하여 정확한 버전 고정이 된 requirements.txt를 생성하세요.

### Node.js: 프로덕션 의존성 최소화

```dockerfile
# 빌더 스테이지: 모든 의존성 설치 및 빌드
FROM node:20-alpine AS build
WORKDIR /w
COPY package*.json ./

# 개발 의존성을 제외하고 정확한 버전으로 설치
RUN npm ci --omit=dev --ignore-scripts
COPY . .

# 필요한 경우 빌드 실행 (React, Vue, Next.js 등)
RUN npm run build

# 런타임 스테이지: 빌드 결과물만 복사
FROM nginx:alpine
COPY --from=build /w/dist /usr/share/nginx/html

# 또는 Node.js 런타임 사용 시
FROM node:20-alpine
WORKDIR /app
COPY --from=build /w/node_modules /app/node_modules
COPY --from=build /w /app
CMD ["node", "server.js"]
```

### Go: 정적 링크와 scratch 베이스

```dockerfile
# 빌더 스테이지
FROM golang:1.23-alpine AS builder
WORKDIR /src
COPY . .

# Go 빌드 최적화
ENV CGO_ENABLED=0 GOOS=linux
RUN --mount=type=cache,target=/root/.cache/go-build \
    go build -trimpath -ldflags="-s -w" -o app .

# 런타임 스테이지: scratch (완전 빈 베이스)
FROM scratch
WORKDIR /

# CA 인증서 복사 (HTTPS 통신을 위해 필수)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# 타임존 데이터 복사 (선택사항)
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# 애플리케이션 바이너리 복사
COPY --from=builder /src/app /app

# 비루트 사용자로 실행 (보안 강화)
USER 65532:65532
ENTRYPOINT ["/app"]
```

**Go 최적화 팁**:
- `-ldflags="-s -w"`로 디버그 심볼을 제거하여 바이너리 크기를 줄이세요.
- `CGO_ENABLED=0`로 설정하여 정적 링크를 보장하세요.
- scratch 사용 시 CA 인증서 복사를 잊지 마세요.

### Java: 모듈화 JRE와 jlink 활용

```dockerfile
# 빌더: JDK와 애플리케이션 빌드
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /w
COPY . .

# 애플리케이션 빌드
RUN ./mvnw -DskipTests package

# 최소 JRE 생성
RUN $JAVA_HOME/bin/jlink \
    --add-modules java.base,java.logging,java.sql \  # 필요한 모듈만
    --strip-debug \
    --no-man-pages \
    --no-header-files \
    --compress=2 \
    --output /opt/jre-min

# 런타임: 최소 JRE + 애플리케이션
FROM debian:12-slim
WORKDIR /app
COPY --from=builder /opt/jre-min /opt/jre
COPY --from=builder /w/target/app.jar /app/app.jar
ENV PATH="/opt/jre/bin:${PATH}"
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

**Java 최적화 팁**:
- `jdeps` 명령어로 애플리케이션이 실제로 사용하는 모듈을 분석하세요.
- 또는 `gcr.io/distroless/java17-debian12` 같은 Distroless Java 런타임을 사용하세요.
- Spring Boot의 경우 `spring-boot-thin-launcher`를 고려하세요.

### .NET: 트리밍과 단일 파일 배포

```dockerfile
# 빌더 스테이지
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /w
COPY . .

# 최적화된 발행
RUN dotnet publish -c Release -o /out \
    /p:PublishTrimmed=true \
    /p:PublishSingleFile=true \
    /p:TieredPGO=true \
    /p:EnableCompressionInSingleFile=true

# 런타임 스테이지
FROM mcr.microsoft.com/dotnet/runtime-deps:8.0-alpine
WORKDIR /app
COPY --from=build /out .
ENTRYPOINT ["./MyApp"]
```

---

## 빌드 성능 최적화: 캐시와 재현성

### BuildKit 고급 기능 활용

```dockerfile
# syntax=docker/dockerfile:1.7

FROM python:3.11-slim AS build
WORKDIR /w
COPY requirements.txt .

# 캐시 마운트로 pip 캐시 재사용
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --prefix=/install --no-cache-dir -r requirements.txt

# 비밀 정보 안전하게 전달
RUN --mount=type=secret,id=pip_token \
    PIP_EXTRA_INDEX_URL="https://__token__:$(cat /run/secrets/pip_token)@gitlab.example.com/api/v4/projects/123/packages/pypi/simple" \
    pip install --prefix=/install --no-cache-dir -r requirements.txt
```

빌드 시 비밀 정보 전달:
```bash
docker build --secret id=pip_token,src=./.pip_token .
```

### 재현 가능한 빌드 보장

1. **의존성 버전 고정**:
   - Python: `pip-compile`로 정확한 버전 관리
   - Node.js: `package-lock.json` 또는 `yarn.lock` 사용
   - Java: Maven/Gradle 버전 락 파일 사용

2. **멀티 아키텍처 지원**:
```dockerfile
ARG TARGETARCH
RUN if [ "$TARGETARCH" = "arm64" ]; then \
      echo "Building for ARM64"; \
    else \
      echo "Building for AMD64"; \
    fi
```

3. **다운로드 검증**:
```dockerfile
# SHA256 체크섬 검증 예시
ADD --checksum=sha256:abc123... https://example.com/file.tar.gz /tmp/
```

---

## 이미지 내부 정리와 최적화

### 효과적인 .dockerignore 파일

```
# 버전 관리
.git
.gitignore
.github/

# 런타임 생성 파일
**/__pycache__/
*.py[cod]
*.so
.Python

# 의존성
node_modules/
vendor/
*.egg-info/

# 빌드 아티팩트
dist/
build/
*.egg
target/

# 환경 설정
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# 문서
docs/
*.md

# 테스트
tests/
test/
spec/
coverage/
```

### 레이어 최적화 패턴

**Alpine 패키지 관리**:
```dockerfile
# 좋은 예: 한 RUN 명령으로 설치 및 정리
RUN apk add --no-cache curl jq \
 && apk add --no-cache --virtual .build-deps gcc musl-dev \
 && pip install --no-cache-dir some-package \
 && apk del .build-deps \
 && rm -rf /var/cache/apk/*
```

**Debian/Ubuntu 패키지 관리**:
```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
 && rm -rf /var/lib/apt/lists/*
```

---

## 검증과 측정: 얼마나 개선되었는가?

### 이미지 크기 분석

{% raw %}
```bash
# 이미지 크기 비교
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | grep myapp

# 레이어별 크기 분석
docker history myapp:latest --no-trunc --format "{{.Size}}\t{{.CreatedBy}}"

# 이미지 내부 탐색
docker run --rm -it myapp:latest du -sh /* 2>/dev/null | sort -hr

# 두 이미지 비교
container-diff diff daemon://myapp:slim daemon://myapp:alpine --type=size --type=file
```
{% endraw %}

### 런타임 검증

```dockerfile
# 헬스체크 설정
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8080/healthz || exit 1
```

### 실전 크기 비교 (Python 예시)

| 전략 | 예상 크기 | 특징 |
|------|-----------|------|
| `python:3.11` | 800-900MB | 모든 도구 포함, 호환성 최고 |
| `python:3.11-slim` | 60-120MB | glibc 기반, 상당한 경량화 |
| `python:3.11-alpine` | 20-60MB | 매우 작음, musl 호환성 확인 필요 |
| `distroless/python3` | 15-40MB | 보안 최적화, 디버깅 어려움 |

---

## 보안 검증 통합

### 취약점 스캔 자동화

```bash
# CI 파이프라인에 통합
trivy image --severity CRITICAL,HIGH --ignore-unfixed --exit-code 1 \
  myapp:latest

# 상세 보고서 생성
trivy image --format json --output trivy-report.json myapp:latest
```

### SBOM(Software Bill of Materials) 생성

```bash
# Syft로 SBOM 생성
syft myapp:latest -o spdx-json > sbom.json

# Docker Scout 활용
docker scout sbom myapp:latest --format cyclonedx-json --output sbom.cdx.json
```

### 이미지 서명과 검증

```bash
# Cosign으로 서명
cosign sign --yes myapp:latest

# 서명 검증
cosign verify myapp:latest

# 정책 기반 배포 차단 (Kubernetes Admission Controller 등)
```

---

## 경량 이미지 디버깅 전략

Distroless나 Alpine 같은 경량 이미지는 디버깅 도구가 부족할 수 있습니다. 다음과 같은 전략으로 대응하세요:

### 1. 디버그용 이미지 병행 운영

```dockerfile
# 프로덕션용 (경량)
FROM gcr.io/distroless/python3-debian12:nonroot
# ...

# 디버그용 (풀 버전)
FROM python:3.11-slim
# 동일한 애플리케이션 코드
# 추가 디버그 도구 포함
```

### 2. 런타임 디버깅 접근법

**Kubernetes 에피메럴 컨테이너**:
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: myapp:distroless
  ephemeralContainers:
  - name: debugger
    image: busybox:latest
    command: ["/bin/sh"]
    stdin: true
    tty: true
```

**Sidecar 패턴**:
```yaml
containers:
- name: app
  image: myapp:alpine
- name: debug-sidecar
  image: nicolaka/netshoot:latest
  command: ["sleep", "infinity"]
```

### 3. 애플리케이션 내장 진단

- **Python**: `faulthandler` 활성화, `/debug` 엔드포인트
- **Go**: `net/http/pprof`, 런타임 메트릭 노출
- **Java**: JMX, Actuator 엔드포인트
- 모든 언어: 구조화된 로깅, 헬스체크 엔드포인트

---

## 실전 적용 사례

### 시나리오 1: Python API - 호환성 중시 (Slim 기반)

```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /w

ENV PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PYTHONDONTWRITEBYTECODE=1

COPY requirements.txt .
RUN apt-get update \
 && apt-get install -y --no-install-recommends build-essential \
 && pip install --prefix=/install -r requirements.txt \
 && apt-get purge -y build-essential \
 && apt-get autoremove -y \
 && rm -rf /var/lib/apt/lists/*

COPY . .

FROM python:3.11-slim
WORKDIR /app

COPY --from=builder /install /usr/local
COPY --from=builder /w .

USER 1000:1000
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')"
CMD ["python", "app.py"]
```

### 시나리오 2: Python API - 크기 최적화 (Alpine 기반)

```dockerfile
FROM python:3.11-alpine AS builder

RUN apk add --no-cache --virtual .build-deps \
    build-base \
    musl-dev \
    libffi-dev \
    openssl-dev \
    postgresql-dev

WORKDIR /w
COPY requirements.txt .
RUN pip install --prefix=/install --no-cache-dir -r requirements.txt
COPY . .

FROM python:3.11-alpine
WORKDIR /app

COPY --from=builder /install /usr/local
COPY --from=builder /w .

RUN apk del .build-deps || true

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8080/healthz || exit 1

CMD ["python", "app.py"]
```

### 시나리오 3: 최대 보안 (Distroless 런타임)

```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /w
COPY requirements.txt .
RUN pip install --prefix=/install --no-cache-dir -r requirements.txt
COPY . .

FROM gcr.io/distroless/python3-debian12:nonroot
WORKDIR /app
COPY --from=builder /install /usr/local
COPY --from=builder /w /app
USER nonroot
CMD ["app.py"]
```

---

## 자주 발생하는 문제와 해결책

### 문제 1: Alpine에서 Segmentation Fault

**증상**: Alpine 기반 이미지에서 애플리케이션이 예기치 않게 종료되며 segfault 메시지 출력

**원인**: glibc를 기대하는 사전 컴파일된 바이너리나 wheel 파일 사용

**해결**:
1. `slim` 버전으로 전환 (가장 쉬운 해결책)
2. 소스에서 직접 컴파일: `pip install --no-binary :all: 패키지명`
3. musl 호환 wheel 찾기: `pip download 패키지명 --platform musllinux`

### 문제 2: psycopg2 빌드 실패

**증상**: `pg_config executable not found` 또는 비슷한 에러

**원인**: PostgreSQL 개발 헤더 파일 누락

**해결**:
```dockerfile
# Alpine
RUN apk add --no-cache postgresql-dev

# Debian/Ubuntu
RUN apt-get update && apt-get install -y --no-install-recommends libpq-dev
```

### 문제 3: Go + scratch에서 HTTPS 실패

**증상**: HTTPS 요청 시 `x509: certificate signed by unknown authority`

**원인**: CA 인증서 누락

**해결**:
```dockerfile
FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
# 또는
COPY --from=alpine:latest /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
```

### 문제 4: Distroless에서 디버깅 불가

**증상**: 셸이 없어 컨테이너 내부 진단 불가

**해결**:
1. 디버그용 이미지 병행 빌드 및 운영
2. 애플리케이션에 진단 엔드포인트 추가
3. Kubernetes 에피메럴 컨테이너 사용
4. 로깅과 메트릭 강화

### 문제 5: 이미지 크기 예상보다 큼

**증상**: 경량화 전략 적용했는데도 이미지가 예상보다 큼

**원인**: 캐시 레이어에 개발 파일 포함, 불필요한 레이어

**해결**:
1. 멀티스테이지 빌드로 개발 도구 완전 분리
2. `.dockerignore` 파일 검토
3. `docker history`로 레이어별 크기 분석
4. BuildKit 캐시 최적화 적용

---

## 수학적 관점: 경량화의 경제성

경량화의 이점을 수학적으로 모델링해보면:

**이미지 전송 시간**:
$$
T = \frac{S}{B} + L
$$
여기서 $T$는 총 전송 시간, $S$는 이미지 크기, $B$는 네트워크 대역폭, $L$은 레이턴시입니다.

**캐시 효율성**:
레이어 재사용을 고려한 효과적 크기:
$$
S_{\text{eff}} = S_{\text{unique}} + \alpha \cdot S_{\text{shared}}
$$
$\alpha$는 캐시 적중률입니다.

**팀 전체 영향**:
$n$개의 서비스가 동일 베이스 레이어를 공유할 때:
$$
T_{\text{total}} \approx \frac{S_{\text{base}}}{B} + n \cdot \frac{S_{\text{app}}}{B}
$$
베이스 레이어를 최적화하면 모든 서비스에 걸쳐 누적 효과가 발생합니다.

---

## 결론

Docker 이미지 경량화는 단순한 기술적 최적화를 넘어 운영 효율성, 보안, 비용 관리까지 영향을 미치는 전략적 작업입니다. 효과적인 경량화를 위해 다음 원칙을 기억하세요:

1. **점진적 접근**: Full → Slim → Alpine → Distroless 순으로 단계적 검증과 적용
2. **맥락에 맞는 선택**: 호환성, 보안 요구사항, 운영 환경을 고려한 베이스 이미지 선택
3. **멀티스테이지 필수**: 빌드 도구와 런타임 환경을 철저히 분리
4. **지속적 검증**: 취약점 스캔, SBOM 생성, 이미지 서명을 개발 워크플로우에 통합
5. **디버깅 준비**: 경량 이미지의 도전 과제를 인지하고 적절한 진단 전략 마련
6. **측정과 개선**: 크기, 빌드 시간, 보안 상태를 지속적으로 모니터링하고 개선

가장 중요한 것은 "한 번에 완벽하게"하려는 시도보다는, 지속적인 개선 문화를 정착시키는 것입니다. 작은 변경이라도 꾸준히 적용하면 시간이 지남에 따라 상당한 운영 이점을 누릴 수 있습니다.

경량화는 목적이 아니라 수단임을 기억하세요. 최종 목표는 더 안전하고, 빠르고, 효율적인 애플리케이션 제공입니다. 이 가이드의 내용을 출발점으로 삼아 조직의 특정 요구사항과 제약 조건에 맞게 조정하고 발전시켜 나가시기 바랍니다.