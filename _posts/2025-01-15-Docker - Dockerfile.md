---
layout: post
title: Docker - Dockerfile
date: 2025-01-15 19:20:23 +0900
category: Docker
---
# Dockerfile 완전 정복: 기본 문법과 개념 설명

## Dockerfile이란?

- Docker 이미지를 **선언적으로 정의**하는 스크립트 파일.
- 빌드 시 **위에서 아래로** 순차 실행되며, 각 지시문은 일반적으로 **레이어**를 형성합니다.
- **반복 가능성**(Reproducibility)과 **자동화**(CI/CD)에 최적화.

기본 구조 예시:
```
my-app/
├── Dockerfile
├── app.py
└── requirements.txt
```

---

## 빌드 컨텍스트와 .dockerignore

### 빌드 컨텍스트

`docker build` 시 지정한 디렉터리(예: `.`)의 파일들이 **컨텍스트**로 데몬에 전송됩니다.
컨텍스트가 크면 느려지고 캐시가 자주 무효화됩니다. 반드시 `.dockerignore`로 다이어트하세요.

### .dockerignore 예시

```dockerignore
# VCS/캐시/산출물/비밀

.git/
.gitignore
__pycache__/
*.pyc
node_modules/
dist/
build/
*.log
.env
# Docker 메타파일은 보통 제외

Dockerfile
docker-compose.yml
README.md
tests/
```

효과
- 빌드 속도 향상
- 보안 강화(비밀/토큰 파일 미포함)
- 캐시 안정화(불필요한 변경 억제)

---

## 기본 문법 총정리(확장)

아래 표는 기존 표를 유지하되, 실전 차이를 보강합니다.

| 키워드 | 핵심 | 비고/모범사례 |
|---|---|---|
| `FROM` | 베이스 이미지 지정(필수) | 멀티스테이지에서 반복 사용 가능, 다이제스트 고정 권장 |
| `RUN` | 빌드 시 셸 명령 실행 → 레이어 생성 | `&&`로 묶어 임시/캐시 정리. `apt-get update && install` 한 RUN에 |
| `COPY` | 로컬 → 이미지 복사 | **예측 가능**. 압축 해제/URL 없음. 기본 선택 |
| `ADD` | COPY + 압축해제 + URL | 특별한 경우(신뢰 가능한 tar 자동 해제 등) 외에는 지양 |
| `CMD` | 컨테이너 시작 시 기본 커맨드 | **하나만 존재**. `docker run` 인자로 override 가능 |
| `ENTRYPOINT` | 고정 진입점 | 인자는 CMD 또는 run 인자로 전달. exec form 권장 |
| `WORKDIR` | 이후 명령의 작업 디렉터리 | 누적 가능. 디렉터리 자동 생성 |
| `ENV` | 환경 변수 | 이후 RUN/CMD/ENTRYPOINT에서도 사용 가능 |
| `ARG` | 빌드 시간 변수 | 빌드 과정에서만 유효(이미지 런타임에서 사라짐) |
| `EXPOSE` | 문서용 포트 표시 | 실제 공개는 `-p` 필요 |
| `VOLUME` | 마운트 지점 힌트 | 실행 시 `-v`로 연결. 이미지 안 데이터는 휘발 가능 |
| `LABEL` | 메타데이터 | Label Schema 활용 권장 |
| `HEALTHCHECK` | 런타임 헬스 판단 | 오케스트레이션과 연계(재시작/교체 기준) |
| `STOPSIGNAL` | 종료 시 신호 지정 | SIGTERM 외 필요한 신호 지정 가능 |
| `SHELL` | 기본 셸 지정 | 윈도우/다른 셸 필요 시 |

---

## FROM — 베이스 이미지 전략

```dockerfile
FROM python:3.11-slim
# 또는 다이제스트 고정
# FROM python@sha256:<digest>

```

- `*-slim`, `alpine`, `distroless`, `scratch` 등을 요구에 따라 선택.
- **재현성**을 위해 태그 대신 **다이제스트 고정**을 고려.

---

## RUN — 레이어 최소화와 캐시

좋은 예:
```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```

- `update`와 `install`을 **한 RUN**에 묶어 stale index 방지.
- 임시파일/패키지 캐시를 **즉시 삭제**.

---

## COPY vs ADD — 무엇을 쓸까?

원칙: **COPY가 기본**, ADD는 **정말 필요한 경우**만.

```dockerfile
# 예측 가능한 복사

COPY ./app.py /app/app.py

# ADD는 tar 자동 해제/URL 다운로드가 있지만, 빌드 재현성과 보안상 COPY 권장

ADD https://example.com/app.tar.gz /opt/   # 특별한 경우
```

---

## CMD vs ENTRYPOINT — exec form을 기본으로

### exec form vs shell form

- exec form(권장): `["nginx","-g","daemon off;"]`
  신호 전달/프로세스 종료가 안정적(PID 1 문제 감소)
- shell form: `nginx -g "daemon off;"`
  셸을 거쳐 신호 전달이 왜곡될 수 있음

### 조합 패턴

```dockerfile
ENTRYPOINT ["curl"]
CMD ["--help"]         # => 기본은 curl --help
# docker run 이미지 -I https://...  => curl -I https://...

```

### 흔한 실수 교정

나쁜 예:
```dockerfile
ENTRYPOINT service nginx start
```
좋은 예:
```dockerfile
ENTRYPOINT ["nginx","-g","daemon off;"]
```

---

## WORKDIR — 상대경로를 안전하게

```dockerfile
WORKDIR /app
COPY . .         # /app 기준으로 복사
RUN ls -al       # /app에서 실행
```
- 디렉터리가 없으면 자동 생성.
- 반복 지정 시 누적.

---

## ENV vs ARG — 수명과 용도

```dockerfile
ARG BUILD_ENV=prod        # 빌드 시간 변수
ENV APP_ENV=$BUILD_ENV    # 런타임에도 남김
```

- `ARG`는 **빌드 시**에만 존재 → 이미지 메타에 남지 않음(ENV로 승격하지 않으면).
- `ENV`는 **런타임**에도 남음.

---

## EXPOSE — 문서, 실제 매핑은 -p

```dockerfile
EXPOSE 8080
# 실행: docker run -p 8080:8080 이미지

```

- 문서용이므로 **네트워크 공개 효과는 없음**.

---

## VOLUME — 상태 데이터 분리

```dockerfile
VOLUME /var/lib/postgresql/data
# 실행 시: -v pgdata:/var/lib/postgresql/data

```

- 컨테이너 레이어는 휘발. 볼륨/바인드 마운트로 영속화.

---

## LABEL — 메타데이터

```dockerfile
LABEL org.opencontainers.image.title="myapp" \
      org.opencontainers.image.version="1.2.3" \
      org.opencontainers.image.source="https://github.com/acme/myapp"
```

- OCI 라벨 키 사용을 권장.

---

## HEALTHCHECK — 헬스 판단과 운영

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -fsS http://localhost:8080/health || exit 1
```

- 컨테이너 상태를 `healthy`/`unhealthy`로 표기.
- 오케스트레이션(K8s 등)과 조합해 교체/재시작 기준으로 활용.

---

## STOPSIGNAL — 종료 시그널 지정

```dockerfile
STOPSIGNAL SIGQUIT
```

- 앱 특성에 맞는 신호로 종료 절차를 안정화.

---

## SHELL — 기본 셸 변경(비일반)

윈도우 컨테이너나 특별한 환경에서 셸 명령 해석기 지정:
```dockerfile
SHELL ["powershell", "-Command"]
```

---

## 멀티스테이지 빌드 — 빌드는 무겁게, 런타임은 슬림하게

### Node → Nginx

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

### Python wheel 전략

```dockerfile
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
COPY --from=build /wheels /wheels
RUN pip install --no-cache-dir /wheels/*
COPY --from=build /app /app
ENTRYPOINT ["gunicorn"]
CMD ["-w","2","-b","0.0.0.0:5000","app:app"]
```

### Go → scratch

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

## BuildKit 고급 기능 — 캐시/시크릿/SSH/허리독

BuildKit 활성화:
```bash
export DOCKER_BUILDKIT=1
```

### 캐시 마운트

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
RUN --mount=type=cache,target=/root/.npm npm ci
RUN --mount=type=cache,target=/go/pkg/mod go mod download
```

### 시크릿(비밀 파일을 이미지에 남기지 않기)

```dockerfile
# syntax=docker/dockerfile:1.7

FROM alpine
RUN --mount=type=secret,id=npmrc \
    cp /run/secrets/npmrc /root/.npmrc && npm ci || true
```
빌드:
```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t myapp:secret .
```

### SSH 포워딩(프라이빗 Git)

```dockerfile
# syntax=docker/dockerfile:1.7

FROM alpine:3.20
RUN apk add --no-cache git openssh
RUN --mount=type=ssh git clone git@github.com:org/private.git /src
```
빌드:
```bash
docker build --ssh default -t app:git .
```

### 허리독(Heredoc)로 가독성 향상

```dockerfile
# syntax=docker/dockerfile:1.7

RUN <<'SH'
set -eux
apk add --no-cache curl jq
curl -s https://example.com | jq .
SH
```

---

## 보안과 최소 권한 실행

```dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```
실행 시:
```bash
docker run --rm \
  --read-only \
  --tmpfs /tmp --tmpfs /run \
  --cap-drop ALL --security-opt no-new-privileges \
  --user 65532:65532 \
  myimg:prod
```

- exec form ENTRYPOINT로 PID 1 신호/종료 처리 안정화.
- 읽기 전용 루트FS + tmpfs로 쓰기 최소화.

---

## 재현성과 태깅

- 태그보다 **다이제스트 고정** 권장:
{% raw %}
```bash
docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine
docker run --rm nginx@sha256:...
```
{% endraw %}
- 의존성 잠금: `package-lock.json`/`poetry.lock`/`go.sum`/`pom.xml` 재현 빌드 설정.
- SBOM/서명: `syft`/`trivy`/`cosign` 등 도구로 공급망 신뢰 강화.

---

## 빌드 시간 직관(레이어 캐시 적중률)

레이어별 캐시 적중률 \(p_i\), 레이어 비용 \(c_i\)일 때, 기대 빌드 시간 근사는:
$$
\mathbb{E}[T_{\text{build}}] \approx \sum_{i=1}^{n} (1 - p_i)\, c_i
$$
의존성 설치처럼 **비용 큰 단계**를 위로 배치하고, **변동 많은 소스**는 아래로 두어 \(p_i\)를 높이면 전체 시간이 줄어듭니다.

---

## 실전 예제: Python 앱(최적화 버전)

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

`.dockerignore`
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
docker build -t my-python-app .
docker run -p 5000:5000 my-python-app
```

---

## 실전 예제: ENTRYPOINT/CMD 조합

```dockerfile
FROM alpine
RUN apk add --no-cache curl
ENTRYPOINT ["curl"]
CMD ["--help"]
# docker run 이미지 -I https://example.com → curl -I https://example.com

```

---

## 트러블슈팅 표

| 증상 | 원인 | 진단 | 해결 |
|---|---|---|---|
| 빌드 느림 | 컨텍스트 과대, 캐시 미활용 | `docker system df`, 로그 | `.dockerignore`, 의존성 먼저 COPY, 캐시 마운트 |
| apt 404 | update/install 분리 | Dockerfile 확인 | **한 RUN**에 `apt-get update && install` |
| 이미지 비대 | dev deps 포함, 임시파일 잔존 | `docker history`, `dive` | 멀티스테이지, `--omit=dev`, 캐시/임시 삭제 |
| 비밀 포함 | `.env`/키 COPY됨 | `dive`, 컨텍스트 확인 | `.dockerignore`, BuildKit secrets |
| 신호 전달 불안 | shell form ENTRYPOINT | 컨테이너 종료 로그 | exec form으로 변경, `--init` 고려 |
| 로그가 뒤섞임 | `attach`로 관찰 | 사용 방식 점검 | 로그는 `docker logs -f` 사용 |

---

## 작성 팁 재정리

| 팁 | 설명 |
|---|---|
| RUN 줄이기 | 명령 묶고 임시/캐시 삭제 |
| .dockerignore | 컨텍스트 축소로 속도/보안/캐시 안정 |
| 경량 베이스 | `slim`/`alpine`/`distroless`/`scratch` |
| 멀티스테이지 | 빌드는 무겁게, 런타임은 슬림하게 |
| exec form | ENTRYPOINT/CMD는 exec form 권장 |
| USER 전환 | 비루트 + 읽기전용 루트FS + tmpfs |
| 재현성 | 잠금 파일 + 다이제스트 고정 + SBOM/서명 |
| BuildKit | cache/secret/ssh/heredoc 적극 활용 |

---

## 명령어 요약

| 명령 | 설명 |
|---|---|
| `docker build -t name .` | 이미지 빌드 |
| `docker run -p H:C image` | 실행/포트 매핑 |
| `docker images` | 이미지 목록 |
| `docker history image` | 레이어/명령 히스토리 |
| `docker system df` | 디스크 사용량 요약 |
| `dive image` | 레이어/파일 구성 분석(외부 도구) |

---

## 결론

- Dockerfile은 **선언적**이고 **레이어 기반**이므로, **순서**와 **컨텍스트**가 성능/재현성/보안에 직결됩니다.
- **COPY 우선**, **exec form**, **의존성 먼저**, **멀티스테이지**, **.dockerignore**, **BuildKit**이 실무의 기본기입니다.
- 문제는 **관찰(로그/히스토리/디스크/레이어) → 가설 → 교정** 루프로 신속히 해결하세요.
