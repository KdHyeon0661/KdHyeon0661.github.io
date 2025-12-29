---
layout: post
title: Docker - Dockerfile & 이미지 최적화
date: 2025-01-15 20:20:23 +0900
category: Docker
---
# Dockerfile과 이미지 최적화 완벽 가이드

## 왜 Dockerfile 최적화가 중요한가?

Dockerfile은 이미지를 빌드하는 청사진입니다. 잘 작성된 Dockerfile은 빠른 빌드 시간, 작은 이미지 크기, 향상된 보안, 그리고 더 나은 개발자 경험을 제공합니다. 이 가이드에서는 Dockerfile 작성의 핵심 원칙과 실무에서 바로 적용할 수 있는 최적화 기술들을 소개합니다.

---

## Dockerfile 빌드 원리 이해하기

### 레이어 기반 캐시 시스템

Docker는 Dockerfile을 **위에서 아래로** 실행하며, 각 명령어마다 새로운 레이어를 생성합니다. 중요한 점은 한 레이어가 변경되면 **그 아래의 모든 레이어가 캐시에서 무효화**된다는 것입니다.

### 캐시 효율성: 나쁜 예 vs 좋은 예

```dockerfile
# 나쁜 예: 캐시 효율성이 떨어짐
COPY . .  # 소스 코드 전체를 먼저 복사
RUN pip install -r requirements.txt
```

```dockerfile
# 좋은 예: 캐시 효율성 최적화
COPY requirements.txt .  # 자주 변경되지 않는 의존성 파일만 먼저 복사
RUN pip install --no-cache-dir -r requirements.txt
COPY . .  # 자주 변경되는 소스 코드는 마지막에 복사
```

**핵심 전략**: 변경 빈도가 낮은 파일(의존성 정의 파일)을 위쪽에, 변경 빈도가 높은 파일(소스 코드)을 아래쪽에 위치시킵니다.

---

## 필수 최적화 기술

### 1. .dockerignore 파일로 빌드 컨텍스트 최소화

`.dockerignore` 파일은 Git의 `.gitignore`와 유사하게 동작하며, 불필요한 파일이 빌드 컨텍스트에 포함되지 않도록 합니다.

```dockerignore
# 일반적인 .dockerignore 예시
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
coverage/
*.md
```

**효과**:
- 빌드 속도 향상 (전송할 데이터 양 감소)
- 보안 강화 (비밀 키나 환경 파일 유출 방지)
- 캐시 안정성 향상 (불필요한 변경으로 인한 캐시 무효화 방지)

### 2. 경량 베이스 이미지 선택

애플리케이션에 맞는 최적의 베이스 이미지를 선택하는 것은 이미지 크기와 보안에 큰 영향을 미칩니다.

| 언어/플랫폼 | 경량 옵션 | 고려사항 |
|---|---|---|
| **Python** | `python:3.x-slim`, `python:3.x-alpine` | `alpine`은 작지만 호환성 이슈가 있을 수 있음 |
| **Node.js** | `node:XX-slim`, `node:XX-alpine` | 네이티브 모듈 빌드 필요 시 `slim` 권장 |
| **Go** | `scratch`, `distroless/static` | 정적 링크 빌드 필요 |
| **Java** | `eclipse-temurin:17-jre-alpine` | JRE만 필요한 경우 선택 |

### 3. 레이어 통합과 정리

여러 개의 `RUN` 명령어를 하나로 통합하고, 임시 파일을 정리하는 것이 중요합니다.

```dockerfile
# 비효율적인 예
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# 효율적인 예
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```

**중요**: `apt-get update`와 `apt-get install`은 항상 같은 `RUN` 명령어에서 실행해야 합니다. 그렇지 않으면 오래된 패키지 목록을 사용하게 될 수 있습니다.

### 4. 멀티스테이지 빌드

빌드 도구와 런타임 환경을 분리하여 최종 이미지 크기를 크게 줄일 수 있습니다.

```dockerfile
# Node.js 애플리케이션 예시
# Stage 1: 빌드
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
RUN npm run build

# Stage 2: 런타임
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

멀티스테이지 빌드를 사용하면:
- 빌드 도구와 의존성이 최종 이미지에 포함되지 않음
- 이미지 크기가 크게 감소
- 보안성이 향상 (불필요한 패키지 감소)

---

## BuildKit 고급 기능 활용

BuildKit은 Docker의 최신 빌드 엔진으로, 다양한 고급 기능을 제공합니다.

### 캐시 마운트
```dockerfile
# 캐시 디렉터리를 빌드 간에 유지
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

### 보안 시크릿
```dockerfile
# 빌드 시 비밀 정보 안전하게 사용
RUN --mount=type=secret,id=npmrc \
    cp /run/secrets/npmrc /root/.npmrc && npm ci
```
```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t app:secret .
```

### SSH 에이전트
```dockerfile
# 프라이빗 Git 저장소 접근
RUN --mount=type=ssh git clone git@github.com:org/private.git
```
```bash
docker build --ssh default -t app:git .
```

---

## 언어별 최적화 패턴

### Python 애플리케이션
```dockerfile
FROM python:3.12-slim

# Python 최적화 환경 변수 설정
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# 의존성 먼저 설치 (캐시 효율성)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 애플리케이션 코드 복사
COPY . .

# 비권한 사용자로 전환
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

CMD ["python", "app.py"]
```

### Node.js 애플리케이션
```dockerfile
FROM node:20-slim

WORKDIR /app

# 패키지 파일 복사 및 의존성 설치
COPY package*.json ./
RUN npm ci --omit=dev

# 애플리케이션 코드 복사
COPY . .

# 비권한 사용자로 실행
USER node

CMD ["node", "server.js"]
```

### Go 애플리케이션
```dockerfile
# Stage 1: 빌드
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app ./cmd/main

# Stage 2: 최소한의 런타임
FROM scratch
COPY --from=builder /app /app
ENTRYPOINT ["/app"]
```

---

## 보안 최적화

### 최소 권한 원칙 적용
```dockerfile
# 비권한 사용자 생성 및 전환
RUN addgroup -g 1000 appgroup && \
    adduser -u 1000 -G appgroup -D appuser
USER appuser
```

### 실행 시 보안 옵션 추가
```bash
docker run --rm \
  --read-only \                      # 루트 파일시스템 읽기 전용
  --tmpfs /tmp \                     # 필요한 임시 디렉터리만 쓰기 가능
  --cap-drop ALL \                   # 모든 특권 제거
  --security-opt no-new-privileges \ # 새 권한 획득 방지
  --user 1000:1000 \                 # 특정 사용자로 실행
  myapp:latest
```

### ENTRYPOINT exec 형식 사용
```dockerfile
# 신호 처리를 위해 exec 형식 사용
ENTRYPOINT ["python", "app.py"]
# CMD ["--help"]  # 기본 인자
```

---

## 이미지 분석과 측정

### 이미지 크기와 레이어 분석
```bash
# 이미지 목록과 크기 확인
docker images

# 이미지 레이어 상세 정보
docker history <이미지명:태그>

# 디스크 사용량 확인
docker system df

# 레이어별 파일 분석 (dive 도구)
dive <이미지명:태그>
```

### 재현성 보장
```bash
# 이미지 다이제스트 확인
docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine

# 다이제스트로 컨테이너 실행 (항상 동일한 이미지 보장)
docker run --rm nginx@sha256:abc123...
```

---

## 일반적인 문제와 해결 방법

### 문제 1: 빌드가 너무 느림
**원인**: 큰 빌드 컨텍스트, 캐시 미활용
**해결**: `.dockerignore` 파일을 사용하고, Dockerfile 레이어 순서를 최적화하세요.

### 문제 2: 이미지가 너무 큼
**원인**: 개발 도구 포함, 임시 파일 미삭제
**해결**: 멀티스테이지 빌드를 사용하고, 빌드 후 임시 파일을 정리하세요.

### 문제 3: APT 패키지 설치 실패
**원인**: `apt-get update`와 `install` 분리
**해결**: 두 명령을 같은 RUN 명령어에서 실행하세요.

### 문제 4: 보안 정보 노출
**원인**: 비밀 키가 이미지에 포함
**해결**: `.dockerignore`에 비밀 파일을 추가하고, BuildKit 시크릿 기능을 사용하세요.

---

## 실전 예제: Flask 애플리케이션 최적화

### 프로젝트 구조
```
my-flask-app/
├── Dockerfile
├── requirements.txt
├── .dockerignore
└── app.py
```

### 최적화된 Dockerfile
```dockerfile
# syntax=docker/dockerfile:1.7

# Stage 1: 빌드 및 의존성 설치
FROM python:3.12-slim AS builder

WORKDIR /app

# Python 환경 변수 설정
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# 의존성 설치 (캐시 활용)
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --user --no-cache-dir -r requirements.txt

# 애플리케이션 코드 복사
COPY . .

# Stage 2: 최종 이미지
FROM python:3.12-slim

WORKDIR /app

# 비권한 사용자 생성
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# 빌드 스테이지에서 설치된 패키지 복사
COPY --from=builder /root/.local /root/.local
COPY --chown=appuser:appgroup . .

# PATH에 사용자 패키지 경로 추가
ENV PATH=/root/.local/bin:$PATH

# 사용자 전환
USER appuser

# 애플리케이션 실행
CMD ["python", "app.py"]
```

### .dockerignore 파일
```dockerignore
.git/
__pycache__/
*.pyc
.env
venv/
*.log
tests/
coverage/
```

---

## 베이스 이미지 선택 가이드라인

베이스 이미지 선택은 애플리케이션의 특성에 따라 결정해야 합니다:

1. **네이티브 확장 모듈이 필요한가?**
   - 예: `slim` 버전 사용 (glibc 호환성)
   - 아니오: `alpine` 고려

2. **디버깅 도구가 필요한가?**
   - 예: `slim` 또는 표준 버전
   - 아니오: `distroless` 또는 `scratch` 고려

3. **이미지 크기가 중요한가?**
   - 예: `alpine` → `distroless` → `scratch` 순으로 평가
   - 아니오: 편의성을 고려하여 선택

---

## 결론

효율적인 Dockerfile 작성은 여러 측면에서 이점을 제공합니다:

1. **개발 생산성 향상**: 빠른 빌드 시간은 개발자의 반복 작업 시간을 크게 단축합니다.
2. **배포 효율성 개선**: 작은 이미지 크기는 네트워크 전송 시간과 저장소 공간을 절약합니다.
3. **보안 강화**: 최소 권한 원칙과 불필요 패키지 제거는 공격 표면을 줄입니다.
4. **운영 안정성 확보**: 재현 가능한 빌드와 명확한 의존성 관리는 운영 환경의 예측 가능성을 높입니다.

Dockerfile 최적화는 한 번에 완벽하게 달성할 수 있는 목표가 아니라 지속적인 개선 과정입니다. 애플리케이션이 성장하고 변화함에 따라 Dockerfile도 함께 진화해야 합니다. 정기적으로 이미지 크기를 측정하고, 빌드 시간을 모니터링하며, 새로운 최적화 기법을 적용하는 습관을 기르는 것이 중요합니다.

가장 효과적인 최적화 전략은 애플리케이션의 특성과 팀의 요구사항에 맞게 조정하는 것입니다. 이 가이드에서 소개한 원칙과 기법을 출발점으로 삼아, 자신의 프로젝트에 최적화된 Dockerfile 작성 방식을 개발해 나가시기 바랍니다.