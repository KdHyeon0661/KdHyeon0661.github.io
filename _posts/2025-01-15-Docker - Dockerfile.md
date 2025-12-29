---
layout: post
title: Docker - Dockerfile
date: 2025-01-15 19:20:23 +0900
category: Docker
---
# Dockerfile 완전 정복: 기본 문법과 개념 설명

## Dockerfile이란?

Dockerfile은 Docker 이미지를 선언적으로 정의하는 스크립트 파일입니다. 빌드 시 위에서 아래로 순차적으로 실행되며, 각 명령어는 일반적으로 새로운 레이어를 생성합니다. 이 구조는 반복 가능한 빌드와 자동화된 CI/CD 파이프라인에 최적화되어 있습니다.

기본적인 프로젝트 구조는 다음과 같습니다:
```
my-app/
├── Dockerfile
├── app.py
└── requirements.txt
```

---

## 빌드 컨텍스트와 .dockerignore 이해

### 빌드 컨텍스트

`docker build` 명령어를 실행할 때 지정한 디렉터리(일반적으로 `.`)의 모든 파일들이 빌드 컨텍스트로 Docker 데몬에 전송됩니다. 컨텍스트가 크면 빌드 속도가 느려지고 캐시 효율성이 떨어지므로, `.dockerignore` 파일을 사용하여 불필요한 파일을 제외하는 것이 중요합니다.

### .dockerignore 파일 예시

```dockerignore
# 버전 관리 시스템 및 캐시 파일
.git/
.gitignore
__pycache__/
*.pyc

# 의존성 및 빌드 산출물
node_modules/
dist/
build/

# 로그 및 환경 파일
*.log
.env

# 문서 및 설정 파일
README.md
tests/
```

`.dockerignore` 파일을 사용하면 다음과 같은 이점이 있습니다:
- 빌드 속도 향상
- 보안 강화(비밀 정보가 이미지에 포함되지 않음)
- 캐시 안정성 개선(불필요한 변경으로 인한 캐시 무효화 방지)

---

## Dockerfile 기본 문법

### FROM: 베이스 이미지 지정

```dockerfile
FROM python:3.11-slim
```

베이스 이미지는 모든 Dockerfile의 첫 번째 명령어로 필수적입니다. 멀티스테이지 빌드에서는 여러 개의 FROM 명령어를 사용할 수 있습니다. 재현성을 위해 태그 대신 다이제스트를 사용하는 것을 고려해보세요.

### RUN: 빌드 시 명령어 실행

```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```

RUN 명령어는 빌드 과정에서 셸 명령어를 실행하고 새로운 레이어를 생성합니다. 관련 명령어들은 `&&`로 연결하여 하나의 RUN 명령어로 묶는 것이 좋으며, 작업 후에는 임시 파일과 캐시를 정리해야 합니다.

### COPY vs ADD: 파일 복사

```dockerfile
# COPY: 예측 가능한 파일 복사
COPY ./app.py /app/app.py

# ADD: 추가 기능 포함 (압축 해제, URL 다운로드)
ADD https://example.com/app.tar.gz /opt/
```

원칙적으로는 COPY를 기본으로 사용하고, ADD는 압축 파일 자동 해제나 URL 다운로드 같은 특별한 경우에만 사용하는 것이 좋습니다.

### CMD와 ENTRYPOINT: 컨테이너 실행 명령

#### exec form vs shell form
- exec form (권장): `["nginx","-g","daemon off;"]`
- shell form: `nginx -g "daemon off;"`

exec form이 신호 전달과 프로세스 종료에서 더 안정적입니다.

#### CMD와 ENTRYPOINT 조합
```dockerfile
ENTRYPOINT ["curl"]
CMD ["--help"]  # 기본값: curl --help
```

이 조합에서 `docker run` 시 추가 인자를 전달하면 CMD 값이 대체됩니다.

### WORKDIR: 작업 디렉터리 설정

```dockerfile
WORKDIR /app
COPY . .         # /app 디렉터리로 복사
RUN ls -al       # /app 디렉터리에서 실행
```

WORKDIR은 이후 명령어들의 작업 디렉터리를 설정합니다. 디렉터리가 존재하지 않으면 자동으로 생성됩니다.

### ENV vs ARG: 환경 변수와 빌드 인자

```dockerfile
ARG BUILD_ENV=prod        # 빌드 시에만 사용
ENV APP_ENV=$BUILD_ENV    # 런타임에도 유지
```

ARG는 빌드 과정에서만 유효하며 이미지에 남지 않습니다. ENV는 런타임에도 환경 변수로 유지됩니다.

### EXPOSE: 포트 문서화

```dockerfile
EXPOSE 8080
```

EXPOSE는 문서화 목적으로만 사용되며, 실제 포트 매핑은 `docker run -p` 옵션으로 수행해야 합니다.

### VOLUME: 데이터 지속성

```dockerfile
VOLUME /var/lib/postgresql/data
```

VOLUME은 컨테이너의 휘발성 데이터를 영구 저장소에 연결하기 위한 힌트를 제공합니다.

### HEALTHCHECK: 상태 확인

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -fsS http://localhost:8080/health || exit 1
```

HEALTHCHECK는 컨테이너의 건강 상태를 주기적으로 확인하여 오케스트레이션 시스템이 자동 복구를 수행할 수 있도록 합니다.

---

## 멀티스테이지 빌드

멀티스테이지 빌드는 빌드 도구와 런타임 환경을 분리하여 최종 이미지 크기를 최소화합니다.

### Node.js 애플리케이션 예시

```dockerfile
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

### Go 애플리케이션 예시

```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -o /out/app ./cmd/app

FROM scratch
COPY --from=build /out/app /app
ENTRYPOINT ["/app"]
```

---

## BuildKit 고급 기능

BuildKit은 Docker의 향상된 빌드 시스템으로, 캐시 최적화와 보안 기능을 제공합니다.

### 캐시 마운트

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
```

### 시크릿 관리

```dockerfile
# syntax=docker/dockerfile:1.7

FROM alpine
RUN --mount=type=secret,id=npmrc \
    cp /run/secrets/npmrc /root/.npmrc && npm ci || true
```

빌드 명령:
```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t myapp:secret .
```

### SSH 포워딩

```dockerfile
# syntax=docker/dockerfile:1.7

FROM alpine:3.20
RUN apk add --no-cache git openssh
RUN --mount=type=ssh git clone git@github.com:org/private.git /src
```

빌드 명령:
```bash
docker build --ssh default -t app:git .
```

### Heredoc을 이용한 가독성 향상

```dockerfile
# syntax=docker/dockerfile:1.7

RUN <<'SH'
set -eux
apk add --no-cache curl jq
curl -s https://example.com | jq .
SH
```

---

## 보안 모범 사례

### 최소 권한 원칙 적용

```dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```

### 보안 강화 실행

```bash
docker run --rm \
  --read-only \
  --tmpfs /tmp --tmpfs /run \
  --cap-drop ALL --security-opt no-new-privileges \
  --user 65532:65532 \
  myimg:prod
```

이러한 설정은 컨테이너의 공격 표면을 최소화하고, 잠재적인 보안 위협을 완화합니다.

---

## 재현성 보장

재현 가능한 빌드를 위해 다음과 같은 접근 방식을 고려하세요:

{% raw %}
```bash
# 다이제스트 확인
docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine

# 다이제스트로 고정 실행
docker run --rm nginx@sha256:...
```
{% endraw %}

의존성 잠금 파일(`package-lock.json`, `poetry.lock`, `go.sum` 등)을 사용하고, SBOM(Software Bill of Materials)과 디지털 서명을 통해 공급망 보안을 강화하세요.

---

## 빌드 최적화 원리

레이어별 캐시 적중률을 $p_i$, 레이어 빌드 비용을 $c_i$라고 할 때, 기대 빌드 시간은 다음과 같이 근사할 수 있습니다:

$$
\mathbb{E}[T_{\text{build}}] \approx \sum_{i=1}^{n} (1 - p_i)\, c_i
$$

이 공식은 비용이 큰 단계(의존성 설치 등)를 앞쪽에 배치하고, 변경이 잦은 소스 코드는 뒤쪽에 배치하여 $p_i$를 높이면 전체 빌드 시간을 줄일 수 있음을 보여줍니다.

---

## 실전 예제: 최적화된 Python 애플리케이션

```dockerfile
# syntax=docker/dockerfile:1.7

FROM python:3.11-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app

FROM base AS deps
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt -t /python

FROM base AS runtime
COPY --from=deps /python /python
ENV PYTHONPATH=/python
COPY . .
RUN addgroup --system app && adduser --system --ingroup app app
USER app
EXPOSE 5000
ENTRYPOINT ["python","app.py"]
```

---

## 문제 해결 가이드

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| 빌드 속도 저하 | 컨텍스트 과다, 캐시 미활용 | `.dockerignore` 파일 사용, 의존성 먼저 복사 |
| apt 패키지 설치 실패 | 오래된 패키지 인덱스 | `apt-get update && install`을 하나의 RUN으로 묶기 |
| 이미지 크기 과대 | 개발 의존성 포함, 임시 파일 잔류 | 멀티스테이지 빌드 사용, 임시 파일 정리 |
| 비밀 정보 노출 | 환경 파일이 이미지에 포함 | `.dockerignore`에 추가, BuildKit 시크릿 사용 |
| 신호 전달 문제 | shell form ENTRYPOINT 사용 | exec form으로 변경, `--init` 옵션 고려 |

---

## Dockerfile 작성 모범 사례

1. **레이어 최소화**: 관련 명령어를 `&&`로 연결하여 RUN 명령어 수를 최소화하세요.
2. **컨텍스트 최적화**: `.dockerignore` 파일을 사용하여 불필요한 파일을 제외하세요.
3. **경량 베이스 이미지**: `slim`, `alpine` 또는 `distroless` 이미지를 고려하세요.
4. **멀티스테이지 활용**: 빌드 도구와 런타임 환경을 분리하세요.
5. **exec form 사용**: ENTRYPOINT와 CMD는 exec form을 사용하세요.
6. **최소 권한 실행**: 비루트 사용자로 전환하고 읽기 전용 파일 시스템을 사용하세요.
7. **재현성 보장**: 다이제스트 고정과 의존성 잠금 파일을 활용하세요.

---

## 결론

Dockerfile은 Docker 이미지를 정의하는 선언적 스크립트로, 레이어 기반의 아키텍처를 통해 효율적인 빌드와 배포를 가능하게 합니다. 효과적인 Dockerfile 작성의 핵심은 적절한 명령어 순서, 컨텍스트 최적화, 그리고 보안 고려사항을 이해하는 데 있습니다.

멀티스테이지 빌드를 활용하면 빌드 도구의 무게를 최종 이미지에서 제거하여 보안성을 높이고 이미지 크기를 줄일 수 있습니다. BuildKit의 고급 기능들은 캐시 효율성과 보안을 더욱 향상시킵니다.

가장 중요한 원칙은 "관찰 → 분석 → 개선"의 지속적인 사이클을 유지하는 것입니다. `docker history`, `docker system df`, 그리고 다양한 분석 도구들을 사용하여 빌드 프로세스를 지속적으로 최적화하세요. 이러한 접근 방식은 더 빠르고 안전하며 재현 가능한 Docker 이미지 구축으로 이어질 것입니다.