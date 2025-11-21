---
layout: post
title: Docker - Docker 경량화 이미지 만들기
date: 2025-03-10 20:20:23 +0900
category: Docker
---
# Docker 경량화 이미지 만들기 (alpine 포함)

본 문서는 사용자가 제시한 초안(Alpine·Slim 비교, 멀티스테이지, distroless)을 **확장**하여,
1) 왜/언제 경량화를 선택해야 하는지, 2) Alpine·Slim·Distroless의 **호환성 트레이드오프**,
3) **언어별(파이썬/Node/Go/Java/.NET)** 최적화 레시피, 4) **캐시·빌드비밀·재현성** 전략,
5) **검증·보안**(취약점 스캔, SBOM, 서명), 6) **디버깅 전술**을 예제와 함께 **빠짐없이** 정리한다.

---

## 경량화의 목적과 한계

### 왜 줄여야 하나

| 목적 | 이유 |
|---|---|
| 배포/스케일 속도 | 작은 레이어는 **푸시/풀** 속도가 빠르다. CI·CD 캐시 적중률도 올라간다. |
| 저장소 비용 절감 | 사설 레지스트리/원격 캐시 비용, 빌드 아티팩트 저장 공간 감소. |
| 공격면 축소 | 패키지·툴을 줄이면 **취약점 개수**가 기하급수적으로 줄어든다. |
| 운영 단순화 | “필요한 것만” 들어있는 이미지는 디버깅 범위를 축소한다. |

### 무조건 Alpine이 해답이 아니다

- **musl vs glibc**: Alpine은 `musl libc` 기반. 일부 네이티브 확장/바이너리(wheel, .so)가 `glibc` 가정일 때 **빌드 실패/런타임 이슈**가 난다.
- 디버깅 난이도: Alpine은 도구가 적다. Distroless는 쉘조차 없다.
- 원칙: **호환성 > 크기**인 경우 `slim`(Debian/Ubuntu) 또는 **언어 전용 경량 런타임**(distroless/java, distroless/python 등)을 우선 검토.

---

## 베이스 이미지 선택 전략 (비교)

| 분류 | 예시 | 장점 | 주의점 |
|---|---|---|---|
| Full | `python:3.11`, `node:20` | 호환성 최고, 디버깅 쉬움 | 이미지 큼 |
| Slim | `python:3.11-slim` | 크기↓, glibc 기반 호환성 유지 | 컴파일러·헤더 등 부족 |
| Alpine | `python:3.11-alpine` | 매우 작음, 시작 빠름 | musl 이슈, 빌드 도구 필요 |
| Distroless | `gcr.io/distroless/python3` | 극소 용량, 공격면 최소 | 쉘 無, 디버깅·확장 난이도↑ |
| Scratch | `scratch` | 완전 빈 베이스(Go 정적링크) | CA 인증서/타임존 등 직접 포함 필요 |

**가이드**
- **네이티브 확장(예: `psycopg2`, `lxml`, `numpy`, `Pillow`)**가 있으면 먼저 `slim`으로 시도.
- 정말 **작은** 런타임이 필요하고 호환성 검증이 끝났다면 Alpine.
- 보안·경량 최우선이면서 런타임만 필요한 경우 Distroless.

---

## 멀티스테이지 빌드 기본 패턴

### — 컴파일러만 분리

```dockerfile
# 빌더: 네이티브 확장 빌드

FROM python:3.11-alpine AS builder
RUN apk add --no-cache gcc musl-dev libffi-dev openssl-dev
WORKDIR /w
COPY requirements.txt .
# 빌드 산출물을 /install에 설치

RUN pip install --prefix=/install --no-cache-dir -r requirements.txt

# 런타임: 가벼운 환경

FROM python:3.11-alpine
WORKDIR /app
# 런타임에 필요한 파일만 복사

COPY --from=builder /install /usr/local
COPY . .
CMD ["python", "app.py"]
```

### — 호환성 + 초경량 런타임

```dockerfile
# 빌더 (glibc, 호환성↑)

FROM python:3.11-slim AS builder
WORKDIR /w
COPY requirements.txt .
RUN pip install --upgrade pip \
 && pip install --prefix=/install --no-cache-dir -r requirements.txt
COPY . .

# 런타임 (distroless)

FROM gcr.io/distroless/python3-debian12:nonroot
WORKDIR /app
COPY --from=builder /install /usr/local
COPY --from=builder /w /app
USER nonroot
EXPOSE 8080
CMD ["app.py"]
```

**핵심 포인트**
- **컴파일러·헤더는 빌더 단계에만** 있고, 최종 런타임에는 포함하지 않는다.
- `--prefix=/install` → 빌더에서 설치 경로를 분리, 런타임에 **필요한 파일만 COPY**.

---

## 언어별 경량화 레시피

### Python — musl 전용 wheel, 과학 패키지 주의

- Alpine에서 **`musllinux` wheel**이 제공되는지 확인. 없는 경우 **컴파일** 필요.
- **자주 필요한 apk**
  | 목적 | 패키지 |
  |---|---|
  | 빌드 도구 | `build-base`(= gcc g++ make), `musl-dev` |
  | crypto/ffi | `openssl-dev`, `libffi-dev` |
  | Postgres | `postgresql-dev` (런타임: `libpq`) |
  | 이미지 | `jpeg-dev`, `zlib-dev`, `freetype-dev` |
  | XML | `libxml2-dev`, `libxslt-dev` |
- 설치 후 **헤더/캐시 제거**:
  ```dockerfile
  RUN apk add --no-cache --virtual .build-deps \
      build-base musl-dev libffi-dev openssl-dev \
   && pip install --no-cache-dir -r requirements.txt \
   && apk del .build-deps
  ```
- 재현성: `pip-compile`(pip-tools)로 **버전 고정**된 `requirements.txt` 생성.

### Node.js — `npm ci` + alpine 빌드 분리

```dockerfile
# 빌더

FROM node:20-alpine AS build
WORKDIR /w
COPY package*.json ./
RUN npm ci --omit=dev  # 프로덕션 의존성만
COPY . .
RUN npm run build      # 예: React/Vite/Next SSG 산출물

# → Nginx

FROM nginx:alpine
COPY --from=build /w/dist /usr/share/nginx/html
```
- 서버 사이드 런타임(Node)인 경우, 런타임 이미지는 `node:alpine`로 최소화하고 devDeps는 빌더에서만 설치.
- 네이티브 의존성(node-gyp 등)은 빌더에서 해결 후 **프리빌트 산출물만 복사**.

### Go — `CGO_ENABLED=0` + `scratch`/`distroless`

```dockerfile
# 빌드

FROM golang:1.23-alpine AS builder
WORKDIR /src
COPY . .
ENV CGO_ENABLED=0 GOOS=linux
RUN --mount=type=cache,target=/root/.cache/go-build \
    go build -trimpath -ldflags="-s -w" -o app .

# — 주의: CA, tzdata 직접 포함 필요

FROM scratch
WORKDIR /
# CA 인증서만 최소 복사(알파인에서 꺼내오기)

COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
COPY --from=builder /src/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```
- `-ldflags="-s -w"`로 심볼 제거(용량↓).
- DNS/HTTPS 문제 방지를 위해 **CA 인증서** 복사 필수.
- 문제가 있으면 `distroless/static` 또는 `distroless/base` 고려.

### Java — `jlink`로 사용자 정의 JRE

```dockerfile
# 빌더: jlink로 최소 JRE 생성 + 애플리케이션 빌드

FROM eclipse-temurin:21-jdk AS builder
WORKDIR /w
COPY . .
RUN ./mvnw -DskipTests package
RUN $JAVA_HOME/bin/jlink \
    --add-modules java.base,java.logging \
    --strip-debug --no-man-pages --no-header-files \
    --compress=2 --output /opt/jre-min

# 런타임: 경량 JRE + fat-jar

FROM debian:12-slim
WORKDIR /app
COPY --from=builder /opt/jre-min /opt/jre
COPY --from=builder /w/target/app.jar /app/app.jar
ENV PATH="/opt/jre/bin:${PATH}"
ENTRYPOINT ["java","-jar","/app/app.jar"]
```
- `jdeps`로 필요한 모듈 파악 → `jlink`에 반영.
- 또는 `gcr.io/distroless/java17-debian12` 등 **distroless/java** 런타임 사용.

### .NET — Publish Trim + Alpine/Distroless

```dockerfile
# 빌더

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /w
COPY . .
RUN dotnet publish -c Release -o /out \
    /p:PublishTrimmed=true /p:PublishSingleFile=true /p:TieredPGO=true

# 런타임(알파인 또는 distroless/dotnet)

FROM mcr.microsoft.com/dotnet/runtime-deps:8.0-alpine
WORKDIR /app
COPY --from=build /out .
ENTRYPOINT ["./MyApp"]
```
- 트리밍(Trim)으로 **미사용 IL 제거**.
- 네이티브/글리브c 이슈가 있으면 `-slim` 계열 또는 `runtime-deps` Debian 기반으로 전환.

---

## 캐시·재현성·빌드비밀 — BuildKit 고급 기능

### 캐시 마운트로 속도↑

```dockerfile
# syntax=docker/dockerfile:1.7

FROM python:3.11-slim AS build
WORKDIR /w
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --prefix=/install --no-cache-dir -r requirements.txt
```
- `--mount=type=cache`로 **의존성 캐시**. CI·CD에서 Buildx + GHA 캐시와 조합.

### 전달

```dockerfile
RUN --mount=type=secret,id=pip_token \
    pip install --no-cache-dir -r requirements.txt
```
- 빌드 시: `docker build --secret id=pip_token,src=./.pip_token …`
- 소스·레이어에 **비밀 흔적을 남기지 않음**.

### 재현성

- 버전 **고정**(pip-compile, npm `package-lock.json`, Maven/Gradle lock).
- `ARG TARGETOS TARGETARCH` 활용해 멀티아치 reproducible 빌드.
- 다운로드 아카이브에 **SHA256 검증**을 습관화.

---

## 이미지 내부 정리 — 레이어·파일 최소화

### .dockerignore

```
.git
.github
**/__pycache__
*.pyc
*.log
node_modules
tests
docs
dist
build
.env
```

### RUN 병합/캐시 제거

```dockerfile
RUN apk add --no-cache curl && \
    rm -rf /var/cache/apk/*
```
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```

### 불필요한 바이트 제거

- Python: `PYTHONDONTWRITEBYTECODE=1`, `__pycache__` 생성 방지.
- Go: `-ldflags="-s -w"`.
- Java: `jlink --strip-debug`.
- `strip`/`upx` 사용은 기능/라이선스/디버깅 이슈를 **사전에 검증**.

---

## 검증·측정 — 실제로 얼마나 줄었나

### 이미지·히스토리 확인

{% raw %}
```bash
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
docker history your/app:tag
```
{% endraw %}

### 컨텐츠 비교(변경 감지)

```bash
container-diff diff daemon://your/app:slim daemon://your/app:alpine
```

### 런타임 스모크 테스트(헬스체크)

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8080/healthz || exit 1
```

### 예시: 사이즈 비교(개략)

| 전략 | 대략 크기 |
|---|---|
| python:3.11 | 800~900MB |
| python:3.11-slim | 60~120MB (패키지에 따라 상이) |
| python:3.11-alpine | 20~60MB (네이티브 빌드 필요 시 증가) |
| distroless/python3 | 15~40MB (+ 의존성) |
| go + scratch | 5~20MB (정적 링크/기능 따라) |

※ 실제는 애플리케이션·의존성·옵션에 따라 크게 달라진다.

---

## 보안: 취약점 스캔·SBOM·서명

### 취약점 스캔

- CI 단계에서 Trivy/Grype로 **CRITICAL/HIGH** 차단.
```bash
trivy image --severity CRITICAL,HIGH your/app:latest
```

### SBOM(구성요소 목록)

- Syft로 SPDX JSON 생성 → 아티팩트 업로드/감사 추적.

### 서명

- Cosign으로 OIDC **keyless 서명** → 레지스트리/정책 연계로 배포 안전성 강화.
```bash
cosign sign --yes your/app:latest
```

---

## 디버깅 전술 — 경량 이미지의 현실 해법

- Distroless/Alpine 런타임은 **디버깅 도구가 없다**.
- 전술:
  1) 동일 코드·동일 환경변수로 **디버그용 이미지(슬림/풀)** 병행 운영.
  2) 런타임에 **임시 BusyBox**를 Sidecar로 붙이거나, 에페메럴 컨테이너(K8s) 이용.
  3) 애플리케이션 **/debug 엔드포인트**, 구조화 로그, pprof(Go), `faulthandler`(Python) 활성화.
  4) Alpine에서 런타임 세그폴트가 나면 **glibc 기준 바이너리 가정** 여부를 먼저 의심. 필요하면 **slim**으로 전환.

---

## 실전 시나리오 — 동일 앱, 3가지 베이스 비교

### Python API — slim 기반(호환성 최우선)

```dockerfile
FROM python:3.11-slim AS build
WORKDIR /w
ENV PIP_NO_CACHE_DIR=1 PIP_DISABLE_PIP_VERSION_CHECK=1
COPY requirements.txt .
RUN apt-get update && apt-get install -y --no-install-recommends build-essential \
 && pip install --prefix=/install -r requirements.txt \
 && apt-get purge -y build-essential \
 && rm -rf /var/lib/apt/lists/*
COPY . .
# 런타임

FROM python:3.11-slim
WORKDIR /app
COPY --from=build /install /usr/local
COPY . .
EXPOSE 8080
CMD ["python","app.py"]
```

### Python API — alpine 기반(용량 최우선, 호환성 검증 완료)

```dockerfile
FROM python:3.11-alpine AS build
RUN apk add --no-cache build-base musl-dev libffi-dev openssl-dev
WORKDIR /w
COPY requirements.txt .
RUN pip install --prefix=/install --no-cache-dir -r requirements.txt
COPY . .

FROM python:3.11-alpine
WORKDIR /app
COPY --from=build /install /usr/local
COPY . .
HEALTHCHECK CMD wget -qO- http://127.0.0.1:8080/healthz || exit 1
CMD ["python","app.py"]
```

### Python API — distroless 런타임(공격면 최소화)

```dockerfile
FROM python:3.11-slim AS build
WORKDIR /w
COPY requirements.txt .
RUN pip install --prefix=/install --no-cache-dir -r requirements.txt
COPY . .

FROM gcr.io/distroless/python3-debian12:nonroot
WORKDIR /app
COPY --from=build /install /usr/local
COPY --from=build /w /app
USER nonroot
CMD ["app.py"]
```

---

## 성능/크기 최적화를 위한 체크리스트

1. **멀티스테이지**로 빌드 도구 격리.
2. **베이스 선택**: slim → alpine → distroless 순으로 단계적 검증.
3. **.dockerignore**로 불필요 파일 제외.
4. **의존성 캐시**(BuildKit `--mount=type=cache`) 적극 활용.
5. 네이티브 확장 패키지의 **시스템 헤더/라이브러리**를 파악하고, 빌더에만 설치.
6. `pip install --no-cache-dir`, `apt --no-install-recommends`, `apk --no-cache`.
7. **CA 인증서/타임존**이 필요한 언어(Go/scratch 등)는 명시적으로 포함.
8. **헬스체크**로 부팅 실패를 조기에 감지, 롤백 비용↓.
9. **취약점 스캔 + SBOM + 서명**을 CI에 상시 통합.
10. 장애 대응을 위해 **디버그용 풀/슬림 이미지**를 병행 유지.

---

## 크기 비교·시간 단축 실험 절차(예시)

```bash
# 여러 Dockerfile로 빌드

docker build -t myapp:full   -f Dockerfile.full .
docker build -t myapp:slim   -f Dockerfile.slim .
docker build -t myapp:alpine -f Dockerfile.alpine .
docker build -t myapp:dist   -f Dockerfile.distroless .

# 크기 비교

docker images | grep myapp

# 히스토리 및 레이어 분석

docker history myapp:alpine

# 컨테이너 가동 및 스모크 테스트

docker run --rm -p 8080:8080 myapp:alpine &
curl -fsS http://127.0.0.1:8080/healthz
```

---

## 자주 겪는 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| Alpine에서 `segfault` | glibc 기대 바이너리/wheel | slim으로 전환, 또는 소스에서 재컴파일 |
| `psycopg2` 빌드 실패 | libpq 헤더 없음 | `apk add postgresql-dev` or `apt-get install libpq-dev` |
| Go + scratch 에서 HTTPS 실패 | CA 인증서 미포함 | `ca-certificates` 복사 |
| Distroless에서 쉘 명령 불가 | 쉘 없음 | 엔드포인트/로그/메트릭으로 진단, 디버그 이미지를 병행 |
| 용량이 안 줄어듦 | 캐시 레이어에 dev 파일 포함 | 멀티스테이지로 dev툴 제거 후 **필요 파일만 COPY** |

---

## 수학적 관점(간단한 모델)

경량화 의사결정 시, 이미지 전송 시간 \(T\)를 **네트워크 대역폭 \(B\)** 와 **이미지 크기 \(S\)** 로 근사하면
$$
T \approx \frac{S}{B}
$$
레이어 공유/캐시로 **효과적 크기 \(S_{\text{eff}}\)** 가 줄면,
$$
T_{\text{eff}} \approx \frac{S_{\text{eff}}}{B} \ll \frac{S}{B}
$$
→ 팀 전체가 **동일 베이스/레이어**를 표준화할수록 배포 시간이 체감상 크게 감소.

---

## 결론

- **경량화의 핵심은 “필요한 것만 담는 것”** 이며, 기술적 레버는 `멀티스테이지`, **올바른 베이스 선택**(slim/alpine/distroless), **캐시·재현성·보안**의 삼박자다.
- Alpine은 강력하지만 **musl 호환성**을 반드시 검증해야 하며, 문제 시 **slim**으로 전환한다.
- 운영 관점에서는 **스캔·SBOM·서명·헬스체크**가 크기만큼 중요하다.
- 위 레시피(언어별 예시 및 체크리스트)를 템플릿으로 삼아, 프로젝트마다 “호환성 → 최적화 → 하드닝” 순으로 점진적으로 적용하라.

---

## 부록: 패키지 치트시트

### Python (alpine)

- 빌드: `build-base musl-dev libffi-dev openssl-dev`
- DB: `postgresql-dev` (런타임: `libpq`)
- 이미지: `jpeg-dev zlib-dev freetype-dev`
- XML: `libxml2-dev libxslt-dev`

### Node (alpine)

- `npm ci --omit=dev`, `node-gyp` 필요 시 `python3 make g++`
- 프론트엔드 정적 사이트는 **Nginx 런타임**로 이관

### Go

- `CGO_ENABLED=0` 정적 링크, CA 인증서 복사
- 문제 시 `distroless/static` 고려

### Java

- `jdeps`로 모듈 파악 → `jlink`로 최소 JRE
- 또는 `distroless/javaXX` 런타임

### .NET

- `PublishTrimmed`/`SingleFile`/`TieredPGO`
- alpine에서 네이티브 의존 시 slim 고려

---

## 참고

- Dockerfile Best Practices
- Distroless Images
- Alpine Linux 문서

(공식 URL은 프로젝트/정책에 맞게 내부 위키나 북마크에 정리해 두길 권장)
