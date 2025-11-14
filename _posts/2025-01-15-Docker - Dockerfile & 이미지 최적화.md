---
layout: post
title: Docker - Dockerfile & 이미지 최적화
date: 2025-01-15 20:20:23 +0900
category: Docker
---
# Dockerfile & 이미지 최적화 베스트 프랙티스

## 빠른 개요(핵심 복습)

- **캐시**: Dockerfile **위→아래**로 레이어 캐시. **윗단 변경 = 아랫단 전부 무효화**.
- **.dockerignore**: 빌드 컨텍스트 다이어트(성능·보안·크기).
- **경량 베이스**: `*-slim`, `alpine`, `distroless`/`scratch` 고려.
- **레이어 최소화**: RUN 묶고, 임시파일 삭제, 멀티스테이지로 산출물만 복사.
- **보안**: `USER` 전환, `--read-only`, `--cap-drop ALL`, `no-new-privileges`.
- **재현성**: 태그보다 **다이제스트(sha256)**, 의존성 **잠금 파일** 사용.

---

## 캐시(Cache) 제대로 활용하기

### 원칙

- Dockerfile은 **위에서 아래로** 평가·레이어화.
- 어떤 레이어가 바뀌면 **그 아래 레이어는 모두 재빌드**.
- COPY/ADD는 **컨텍스트의 해시**에 민감. 작은 변화도 캐시 무효화 유발.

### 안 좋은 예 vs 좋은 예

```Dockerfile
# 잘못된 예: 캐시 무효화 유발

COPY . .                       # 변경 잦은 코드 전체 복사
RUN pip install -r requirements.txt
```

```Dockerfile
# 권장 예: 의존성 먼저, 소스 나중

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

### 전략 요약

| 항목 | 설명 |
|---|---|
| 의존성 먼저 COPY | `requirements.txt`, `package*.json`, `go.mod/sum`, `pom.xml` 등 |
| 상단은 안정, 하단은 잦은 변경 | apt 설치·런타임 의존성은 위쪽, 소스는 아래쪽 |
| 레이어 합치기 | RUN 명령 결합 + 캐시/임시파일 정리 |
| `.dockerignore` | 큰 폴더/비밀/로그 제외(컨텍스트 안정화) |

간단 직관(기대 빌드 시간 근사):
$$
\mathbb{E}[T_{\text{build}}] \approx \sum_{i=1}^{n}(1-p_i)\,c_i
$$
여기서 \(p_i\)는 i번째 레이어 캐시 적중률, \(c_i\)는 레이어 비용. 높은 \(p_i\)를 위쪽에서 확보하면 전체 시간이 줄어듭니다.

---

## .dockerignore로 컨텍스트 다이어트

### 예시

```dockerignore
# VCS/캐시/빌드산출물/비밀 제외

.git/
.gitignore
__pycache__/
*.pyc
node_modules/
dist/
build/
*.log
.env
Dockerfile
docker-compose.yml
README.md
tests/
```

### 효과

- 빌드 속도↑(전송·해시 비용↓)
- 보안↑(비밀/토큰 유출 방지)
- 캐시 적중 안정화(불필요 변경 억제)

---

## 경량 베이스 이미지 선택

| 언어/플랫폼 | 경량 추천 | 비고 |
|---|---|---|
| Python | `python:3.x-slim`, `python:3.x-alpine` | `alpine`은 glibc/빌드 이슈 가능, `slim`도 고려 |
| Node.js | `node:XX-slim`, `node:XX-alpine` | 본인 모듈 네이티브 빌드 유무에 따라 선택 |
| Go | `scratch`, `gcr.io/distroless/static` | 정적 링크 빌드 전제 |
| Java | `eclipse-temurin:17-jre-alpine` | jre/jdk 선택, 메모리 옵션 조정 |

> `alpine`은 작지만 musl 기반으로 빌드/런타임 호환 이슈가 있을 수 있습니다. `slim`이 더 안정적인 경우가 많습니다.

---

## 레이어 줄이기와 APT 모범 사례

### RUN 묶기 + 캐시 삭제

```Dockerfile
# 비효율

RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# 개선

RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*
```

### 임시파일 정리

```Dockerfile
RUN wget file.zip \
 && unzip file.zip \
 && rm -f file.zip
```

### APT 스텝을 한 RUN에서

```Dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends build-essential ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```
- `update`와 `install`을 분리하면 **stale index**로 실패/재현성 저해.

---

## 멀티스테이지 빌드 — “빌드는 무겁게, 런타임은 슬림하게”

### Node → Nginx

```Dockerfile
# Stage 1: 빌드

FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build

# Stage 2: 런타임

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

### Python wheel 전략

```Dockerfile
# syntax=docker/dockerfile:1.7

FROM python:3.12-alpine AS build
WORKDIR /app
RUN apk add --no-cache build-base libffi-dev
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip wheel --no-cache-dir --no-deps -r requirements.txt -w /wheels
COPY . .

FROM python:3.12-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
USER app
COPY --from=build /wheels /wheels
RUN pip install --no-cache-dir /wheels/*
COPY --from=build /app /app
ENTRYPOINT ["gunicorn"]
CMD ["-w","2","-b","0.0.0.0:5000","app:app"]
```

### Go → scratch/distroless

```Dockerfile
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

## BuildKit 고급 기능: 캐시·시크릿·SSH

BuildKit 활성화:
```bash
export DOCKER_BUILDKIT=1
```

### 캐시 마운트

```Dockerfile
# Node

RUN --mount=type=cache,target=/root/.npm npm ci

# Go

RUN --mount=type=cache,target=/go/pkg/mod go mod download

# Python

RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
```

### 시크릿 마운트(빌드 시 비밀 노출 금지)

```Dockerfile
# syntax=docker/dockerfile:1.7

FROM alpine
RUN --mount=type=secret,id=npmrc \
    cp /run/secrets/npmrc /root/.npmrc && npm ci || true
```
```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t app:secret .
```

### SSH 포워딩(프라이빗 Git)

```Dockerfile
# syntax=docker/dockerfile:1.7

FROM alpine:3.20
RUN apk add --no-cache git openssh
# 빌드 중 ssh-agent 인증 사용

RUN --mount=type=ssh git clone git@github.com:org/private.git /src
```
```bash
docker build --ssh default -t app:git .
```

---

## 언어별 최적화 패턴

### Python

```Dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt
COPY . .
USER 65532:65532
CMD ["python","app.py"]
```
- `PYTHONDONTWRITEBYTECODE` 로 `.pyc` 쓰기 방지.
- wheel 캐시 사용, dev 의존성 분리(`requirements-prod.txt` 권장).

### Node.js

```Dockerfile
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev
COPY . .
USER node
CMD ["node","server.js"]
```
- `npm ci`로 재현 가능 설치, 프로덕션 모드 `--omit=dev`.

### Java

```Dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/app.jar /app/app.jar
ENTRYPOINT ["java","-XX:+UseZGC","-jar","/app/app.jar"]
```
- JRE 베이스로 경량화, 힙/GC 옵션 조정.

### Go

- 정적 링크 후 `scratch/distroless` 런타임, CA 필요 시 번들 주의.

---

## 보안·권한·실행옵션

### USER 전환

```Dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```

### 런 옵션(최소 권한)

```bash
docker run --rm \
  --read-only \
  --tmpfs /tmp --tmpfs /run \
  --cap-drop ALL --security-opt no-new-privileges \
  --user 65532:65532 \
  myimg:prod
```

### ENTRYPOINT/CMD exec form

```Dockerfile
ENTRYPOINT ["nginx","-g","daemon off;"]
# CMD는 기본 인자(override 용)

```
- exec form은 **신호 전달/종료 처리** 안정적(PID 1 문제 회피).

---

## 재현성·서명·SBOM

### 다이제스트 고정

{% raw %}
```bash
docker pull nginx:alpine
docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine
# nginx@sha256:...

docker run --rm nginx@sha256:...
```
{% endraw %}

### 의존성 잠금

- Python: `pip-tools`, `poetry.lock`
- Node: `package-lock.json`/`pnpm-lock.yaml`
- Go: `go.sum`
- Java: `mvn dependency:go-offline` + reproducible builds 설정

### SBOM/취약점/서명(예시 도구)

- SBOM: `syft`
- 취약점 스캔: `trivy`
- 서명/검증: `cosign`
- 공급망: SLSA/Provenance(요구 시)

---

## 이미지 크기 줄이기 체크리스트

- `*-slim`/`alpine`/`distroless`/`scratch` 검토.
- 멀티스테이지로 **산출물만** 복사.
- RUN 묶기 + 캐시/임시파일 삭제.
- dev 의존성 제외(`--omit=dev` 등).
- `.dockerignore`로 컨텍스트 축소.
- 불필요 바이너리/스크립트 제거.

---

## 측정·검증 도구

```bash
docker images                       # 크기
docker history <image:tag>          # 레이어별 크기/명령
docker system df                    # 디스크 사용량 요약
dive <image:tag>                    # 레이어별 파일 분석(권장 도구)
docker run -it --rm <image> sh      # 실제 파일 시스템 확인
```

---

## 트러블슈팅 표

| 증상 | 원인 | 진단 | 해결 |
|---|---|---|---|
| 빌드가 느림 | 컨텍스트 과대, 캐시 미활용 | `docker system df`, BuildKit 로깅 | `.dockerignore`, 레이어 순서 조정, cache mount |
| apt 404/stale | update/install 분리 | Dockerfile/로그 | **한 RUN**에 `update && install` |
| 이미지가 비대 | dev deps 포함, 캐시 미정리 | `docker history`, `dive` | 멀티스테이지, `--omit=dev`, 임시파일 제거 |
| 비밀 유출 | `.env`/키 포함 COPY | `dive`, 컨텍스트 검사 | `.dockerignore`, BuildKit secret mount |
| Alpine 호환 이슈 | musl/glibc 차이 | 런타임 에러 | `slim` 사용 고려 또는 적절한 패키지 |
| 캐시가 매번 무효화 | 변동 파일 상단 COPY | 로그 비교 | 의존성 먼저 COPY, 변동 파일은 하단 COPY |

---

## 예제: 실전 Flask 서비스 최적화(종합)

```
my-app/
├── Dockerfile
├── requirements.txt
└── app.py
```

```Dockerfile
# syntax=docker/dockerfile:1.7

FROM python:3.12-slim AS base
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

`.dockerignore`:
```dockerignore
.git/
__pycache__/
*.pyc
.env
dist/
build/
*.log
```

빌드/실행:
```bash
docker build -t flask-slim:1 .
docker run -d --name api -p 5000:5000 \
  --read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges \
  flask-slim:1
```

---

## 예제: 프런트엔드(React/Vite) → Nginx 배포

```Dockerfile
FROM node:20-slim AS build
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
# 헬스엔드포인트

RUN printf "events {}\nhttp { server { listen 80; location /health { return 200 'ok'; } } }" > /etc/nginx/nginx.conf
```

---

## 베이스 선택 의사결정 간단 가이드

1) 네이티브 확장/빌드 필요한가? → 그렇다면 `slim` 우선.
2) 순수 정적 산출물? → `alpine`/`distroless`/`scratch` 고려.
3) 런타임 디버깅 필요? → 셸/툴 유무 확인(`alpine`은 busybox, distroless엔 없음).

---

## 수학 메모(캐시 적중률과 빌드 비용 직관)

레이어별 캐시 적중률 \(p_i\), 레이어 비용 \(c_i\)일 때:
$$
\mathbb{E}[T_{\text{build}}] \approx \sum_{i=1}^{n}(1-p_i)\,c_i
$$
- **변경 빈도 낮고 비용 큰 단계**(의존성 설치)를 **상단**으로 배치 → \(p_i \uparrow\) → 빌드 시간 감소.

---

## 결론 요약

| 목표 | 전략 |
|---|---|
| 캐시 효율 | 의존성 먼저 COPY, 레이어 순서 최적화, BuildKit cache mount |
| 이미지 크기 | 경량 베이스 + 멀티스테이지 + 임시파일 제거 |
| 보안 | `USER` 전환, `--read-only`, `--cap-drop ALL`, secrets mount |
| 재현성 | 다이제스트 고정, 의존성 잠금, reproducible build, SBOM/서명 |
| DX/속도 | `.dockerignore`, RUN 묶기, 언어별 최적 패턴, 측정 도구(dive) |

---

## 참고 명령 모음

{% raw %}
```bash
# 이미지/레이어/디스크

docker images
docker history <image:tag>
docker system df

# 실행 확인

docker run -it --rm <image> sh
dive <image:tag>

# BuildKit on

export DOCKER_BUILDKIT=1

# 캐시/시크릿/SSH 사용 빌드 예

docker build --secret id=npmrc,src=$HOME/.npmrc --ssh default -t app:opt .

# 재현 가능한 실행(다이제스트)

docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine
docker run --rm nginx@sha256:...
```
{% endraw %}
