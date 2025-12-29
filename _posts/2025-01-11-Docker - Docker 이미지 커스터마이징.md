---
layout: post
title: Docker - Docker 이미지 커스터마이징
date: 2025-01-11 21:20:23 +0900
category: Docker
---
# Docker 이미지 생성 및 커스터마이징

Docker 컨테이너의 핵심은 이미지입니다. 이미지는 애플리케이션과 그 실행 환경을 패키징한 읽기 전용 템플릿으로, Dockerfile이라는 정의 파일을 통해 구체적인 구성 방법을 명시합니다. 이번 포스트에서는 효율적이고 안전한 Docker 이미지를 생성하고 관리하는 방법을 단계별로 알아보겠습니다.

## Dockerfile의 기본 구조 이해하기

Dockerfile은 이미지를 구성하기 위한 일련의 지시문으로 이루어진 텍스트 파일입니다. 각 지시문은 이미지에 새로운 레이어를 추가하며, 이 레이어들은 스택처럼 쌓여 최종 이미지를 형성합니다.

간단한 Python Flask 애플리케이션을 Docker 이미지로 만드는 기본 예제부터 시작해보겠습니다.

### 프로젝트 구조
```
my-app/
├── Dockerfile          # 이미지 빌드 명세서
├── app.py             # Flask 애플리케이션
└── requirements.txt   # Python 의존성 목록
```

### 기본 Dockerfile 예제

```Dockerfile
# 베이스 이미지 지정: Python 3.10의 경량 버전
FROM python:3.10-slim

# 작업 디렉토리 설정
WORKDIR /app

# 의존성 파일 복사 및 설치 (캐시 최적화를 위해 먼저 수행)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 애플리케이션 코드 복사
COPY . .

# 컨테이너 실행 시 실행될 명령어
CMD ["python", "app.py"]
```

### 이미지 빌드 및 실행

```bash
# 이미지 빌드
docker build -t my-flask-app .

# 컨테이너 실행
docker run -d -p 5000:5000 --name my-flask-app my-flask-app
```

애플리케이션이 5000번 포트에서 서비스한다면 `http://localhost:5000`으로 접속할 수 있습니다.

## 레이어 캐시의 원리와 최적화 전략

Docker 빌드 과정에서 가장 중요한 개념 중 하나는 레이어 캐시입니다. 각 Dockerfile 지시문은 독립적인 레이어를 생성하며, 이 레이어는 내용이 변경되지 않으면 재사용됩니다. 이 원리를 이해하면 빌드 시간을 크게 단축할 수 있습니다.

### 캐시 최적화 원칙

1. **변경 빈도가 낮은 작업을 앞쪽에 배치**: 시스템 패키지 설치, 런타임 설치, 의존성 다운로드 등은 상대적으로 자주 변경되지 않으므로 Dockerfile 상단에 위치시킵니다.

2. **변경이 잦은 소스 코드는 뒤쪽에 배치**: 애플리케이션 코드는 빈번히 변경되므로 Dockerfile 하단에 위치시켜 상위 레이어의 캐시 무효화를 최소화합니다.

3. **필요한 파일만 복사**: 불필요한 파일을 복사하지 않아야 레이어 크기를 줄이고 캐시 효율을 높일 수 있습니다.

### 캐시 효율성에 대한 직관적 이해

빌드 시간을 수학적으로 표현해보면, 각 레이어의 캐시 적중 확률이 빌드 효율에 직접적인 영향을 미칩니다. 캐시 적중률이 높은 레이어(의존성 설치 등)를 앞쪽에 배치하면 전체 빌드 시간을 크게 줄일 수 있습니다.

## .dockerignore: 빌드 컨텍스트 최적화

Docker 빌드 시 현재 디렉토리(빌드 컨텍스트)의 모든 파일이 Docker 데몬으로 전송됩니다. 불필요한 파일이 많을수록 빌드가 느려지므로 `.dockerignore` 파일을 사용하여 전송할 파일을 필터링해야 합니다.

### .dockerignore 예제

```dockerignore
# Python 관련
__pycache__/
*.pyc
*.pyo
*.pyd
.Python

# 환경 설정 파일
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# 의존성 디렉터리
node_modules/
venv/
env/
.venv/

# 버전 관리 시스템
.git/
.gitignore
.gitattributes

# 빌드 아티팩트
dist/
build/
*.egg-info/

# 운영체제 관련
.DS_Store
Thumbs.db

# 로그 파일
*.log
logs/

# 테스트 관련
coverage/
.pytest_cache/
.tox/
```

이러한 설정으로 빌드 컨텍스트 크기를 최소화하면 빌드 속도가 향상되고 불필요한 파일이 이미지에 포함되는 것을 방지할 수 있습니다.

## 멀티스테이지 빌드: 빌드 도구와 런타임 분리

멀티스테이지 빌드는 빌드 시간 의존성(컴파일러, 빌드 도구 등)과 런타임 의존성을 분리하여 최종 이미지 크기를 최소화하는 강력한 기술입니다. "빌드는 무겁게, 런타임은 가볍게"라는 철학을 구현합니다.

### Python 애플리케이션 멀티스테이지 예제

```Dockerfile
# syntax=docker/dockerfile:1.7

########## 빌드 스테이지 ##########
FROM python:3.12-alpine AS build
WORKDIR /app

# 빌드에 필요한 도구 설치
RUN apk add --no-cache build-base libffi-dev

# 의존성 파일 복사 및 wheel 패키지 생성
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip wheel --no-cache-dir --no-deps -r requirements.txt -w /wheels

# 애플리케이션 코드 복사
COPY . .

########## 런타임 스테이지 ##########
FROM python:3.12-alpine
WORKDIR /app

# 보안을 위한 비루트 사용자 생성
RUN addgroup -S app && adduser -S app -G app
USER app

# 빌드 스테이지에서 생성된 wheel 패키지 설치
COPY --from=build /wheels /wheels
RUN pip install --no-cache-dir /wheels/*

# 애플리케이션 코드 복사
COPY --from=build /app /app

# 서비스 포트 선언
EXPOSE 5000

# Gunicorn을 통한 프로덕션 실행
ENTRYPOINT ["gunicorn"]
CMD ["-w", "1", "-b", "0.0.0.0:5000", "app:app"]
```

### 멀티스테이지 빌드의 장점

1. **이미지 크기 최소화**: 컴파일러, 빌드 도구, 헤더 파일 등이 최종 이미지에 포함되지 않습니다.
2. **보안 강화**: 불필요한 도구가 제거되어 공격 표면이 줄어듭니다.
3. **레이어 캐시 효율성**: 의존성 설치와 애플리케이션 빌드가 별도의 스테이지에서 이루어져 캐시 재사용률이 높아집니다.

### 초경량 스크래치 이미지 예제 (Go 언어)

```Dockerfile
# 빌드 스테이지
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -o /out/app ./cmd/app

# 런타임 스테이지 (스크래치: 완전히 빈 베이스)
FROM scratch
COPY --from=build /out/app /app
ENTRYPOINT ["/app"]
```

`FROM scratch`는 완전히 빈 베이스 이미지로, 실행 파일만 포함하기 때문에 이미지 크기가 수 MB 이하로 유지될 수 있습니다. 단, HTTP 클라이언트 기능이 필요한 경우 CA 인증서 번들 추가가 필요할 수 있습니다.

### Node.js 애플리케이션 예제

```Dockerfile
# 빌드 스테이지
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
COPY . .
RUN npm run build  # dist/ 디렉토리 생성

# 런타임 스테이지
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

## ENTRYPOINT와 CMD: 컨테이너 실행 방식 제어

Dockerfile에서 컨테이너의 실행 방식을 정의하는 두 가지 주요 지시문은 ENTRYPOINT와 CMD입니다. 이 둘의 차이와 적절한 사용법을 이해하는 것이 중요합니다.

### ENTRYPOINT와 CMD의 관계

- **ENTRYPOINT**: 컨테이너가 시작될 때 항상 실행되는 기본 실행 파일이나 스크립트를 정의합니다. 변경되지 않아야 하는 핵심 실행 로직에 사용합니다.
- **CMD**: ENTRYPOINT에 전달될 기본 인자나 대체 실행 명령을 정의합니다. `docker run` 명령어로 쉽게 재정의할 수 있습니다.

### 실행 형식: Exec Form vs Shell Form

**Exec Form (JSON 배열) - 권장**
```Dockerfile
ENTRYPOINT ["gunicorn"]
CMD ["-w", "1", "-b", "0.0.0.0:5000", "app:app"]
```
- 장점: PID 1로 직접 실행되어 신호(SIGTERM 등)가 정확히 전달됨
- 장점: 셸 파싱 오버헤드 없음

**Shell Form - 권장하지 않음**
```Dockerfile
# PID 1이 셸이 되어 신호 전달에 문제 발생 가능
ENTRYPOINT gunicorn -w 1 -b 0.0.0.0:5000 app:app
```

### 실제 사용 예시

```bash
# Dockerfile에 정의된 기본 명령 실행
docker run my-app

# CMD 재정의
docker run my-app -w 4 -b 0.0.0.0:8000 app:app

# ENTRYPOINT와 CMD 모두 재정의
docker run --entrypoint python my-app -c "print('Hello')"
```

## 고급 Dockerfile 기능 활용

### 환경 변수와 빌드 인자

```Dockerfile
# 빌드 시점 변수 (이미지 내부에 남지 않음)
ARG BUILD_DATE
ARG VCS_REF
ARG APP_VERSION=1.0.0

# 런타임 환경 변수
ENV APP_ENV=production
ENV LOG_LEVEL=info
ENV PORT=5000

# 메타데이터 레이블
LABEL org.opencontainers.image.created=$BUILD_DATE \
      org.opencontainers.image.version=$APP_VERSION \
      org.opencontainers.image.revision=$VCS_REF \
      org.opencontainers.image.source="https://github.com/user/repo"
```

### 헬스체크 구성

```Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:5000/health || exit 1
```

헬스체크는 컨테이너의 실행 상태를 주기적으로 점검하여 오케스트레이션 시스템(Kubernetes, Docker Swarm 등)에 정상 여부를 보고합니다.

## 보안 모범 사례 구현

Docker 컨테이너 보안은 여러 층위에서 접근해야 합니다. 이미지 빌드 단계부터 보안을 고려한 설계가 필요합니다.

### 기본 보안 조치

1. **비루트 사용자로 실행**: 컨테이너 내부에서 root 권한을 필요로 하지 않는 경우, 일반 사용자로 실행합니다.

```Dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```

2. **의존성 신뢰성**: 공식 이미지 사용, 패키지 서명 검증, 취약점 스캔 도구 활용

3. **이미지 서명**: 컨테이너 이미지 무결성 검증을 위한 디지털 서명 적용

### 실행 시 보안 설정 예시

```bash
docker run --rm \
  --read-only \                    # 루트 파일 시스템 읽기 전용
  --tmpfs /tmp --tmpfs /run \     # 쓰기가 필요한 임시 파일시스템
  --cap-drop ALL \                # 모든 권한 제거
  --cap-add NET_BIND_SERVICE \    # 필요한 권한만 추가
  --security-opt no-new-privileges \  # 권한 상승 차단
  --user 1000:1000 \              # 비루트 사용자 지정
  --pids-limit 100 \              # 최대 프로세스 수 제한
  --cpus 1.0 \                    # CPU 사용 제한
  --memory 512m \                 # 메모리 사용 제한
  my-app
```

## 실용적인 예제: 프로덕션 준비 Flask 애플리케이션

이제 지금까지 배운 개념들을 종합하여 실제 프로덕션 환경에 적용 가능한 Flask 애플리케이션 Dockerfile을 작성해보겠습니다.

```Dockerfile
# syntax=docker/dockerfile:1.7

# 베이스 스테이지: 공통 설정
FROM python:3.12-alpine AS base
WORKDIR /app

# 비루트 사용자 생성
RUN addgroup -S app && adduser -S app -G app

# 환경 변수 설정
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

# 의존성 스테이지
FROM base AS deps
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt -t /python

# 런타임 스테이지
FROM base AS runtime
ENV PYTHONPATH=/python:$PYTHONPATH

# 의존성 복사
COPY --from=deps /python /python

# 애플리케이션 코드 복사
COPY . .

# 사용자 변경
USER app

# 서비스 포트 노출
EXPOSE 5000

# 헬스체크 설정
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request; \
      urllib.request.urlopen('http://127.0.0.1:5000/health', timeout=2)" || exit 1

# 실행 명령
ENTRYPOINT ["gunicorn"]
CMD ["--workers", "4", "--bind", "0.0.0.0:5000", "--access-logfile", "-", "app:app"]
```

## Nginx 정적 파일 서버 커스터마이징

웹 애플리케이션에서 정적 파일 서빙을 위해 Nginx를 커스터마이징하는 예제입니다.

### Dockerfile

```Dockerfile
FROM nginx:alpine

# 정적 파일 복사
COPY ./html /usr/share/nginx/html

# Nginx 설정 파일 복사
COPY ./nginx.conf /etc/nginx/nginx.conf

# 비루트 사용자로 전환 (nginx 이미지는 이미 nginx 사용자가 있음)
USER nginx
```

### Nginx 설정 파일 (nginx.conf)

```nginx
# ./nginx.conf

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 로그 형식
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    # Gzip 압축
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript 
               application/javascript application/xml+rss application/json;

    server {
        listen 80;
        server_name localhost;
        root /usr/share/nginx/html;

        # 보안 헤더
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;

        # 정적 파일 캐싱
        location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }

        # 헬스체크 엔드포인트
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }

        # 기본 인덱스 파일
        location / {
            try_files $uri $uri/ /index.html;
        }

        # 404 에러 페이지
        error_page 404 /404.html;
        location = /404.html {
            internal;
        }
    }
}
```

### 빌드 및 실행

```bash
# 이미지 빌드
docker build -t custom-nginx:1.0 .

# 컨테이너 실행
docker run -d -p 8080:80 --name web custom-nginx:1.0

# 헬스체크 확인
curl http://localhost:8080/health
```

## 데이터 영속화: 데이터베이스 예제

상태를 유지해야 하는 애플리케이션(데이터베이스 등)의 경우 볼륨을 사용하여 데이터를 영속화합니다.

```bash
# 볼륨 생성
docker volume create postgres_data

# PostgreSQL 컨테이너 실행
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secure_password \
  -e POSTGRES_USER=app_user \
  -e POSTGRES_DB=app_db \
  -v postgres_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16-alpine

# 볼륨 정보 확인
docker volume inspect postgres_data
```

## BuildKit: 현대적인 빌드 시스템

Docker 18.09 이상에서는 BuildKit을 새로운 빌드 엔진으로 사용할 수 있습니다. BuildKit은 향상된 캐시 메커니즘, 병렬 빌드, 시크릿 관리 등의 고급 기능을 제공합니다.

### BuildKit 활성화

```bash
# 환경 변수로 활성화
export DOCKER_BUILDKIT=1

# 또는 Docker 데몬 설정으로 영구 활성화
# /etc/docker/daemon.json 에서 "features": {"buildkit": true}
```

### BuildKit 고급 기능

**시크릿 주입 (빌드 시)**
```Dockerfile
# syntax=docker/dockerfile:1.7
FROM alpine
RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret
```

```bash
# 빌드 시 시크릿 전달
docker build --secret id=mysecret,src=./secret.txt -t app:secret .
```

**캐시 마운트 (언어별 의존성 캐시)**
```Dockerfile
# Go 언어 예시
RUN --mount=type=cache,target=/go/pkg/mod go mod download

# Node.js 예시
RUN --mount=type=cache,target=/root/.npm npm ci

# Python 예시
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
```

## Docker Compose를 활용한 멀티컨테이너 개발 환경

여러 서비스로 구성된 애플리케이션은 Docker Compose를 사용하여 통합 관리할 수 있습니다.

### docker-compose.yml 예제

```yaml
version: '3.8'

services:
  # Flask API 서비스
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    environment:
      - DATABASE_URL=postgresql://app_user:password@db:5432/app_db
      - REDIS_URL=redis://redis:6379/0
    ports:
      - "5000:5000"
    depends_on:
      - db
      - redis
    volumes:
      - ./api:/app
    networks:
      - app-network

  # PostgreSQL 데이터베이스
  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=app_user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=app_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

  # Redis 캐시
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - app-network

  # Nginx 프록시
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./static:/usr/share/nginx/html:ro
    depends_on:
      - api
    networks:
      - app-network

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

### Compose 명령어

```bash
# 서비스 시작
docker compose up -d

# 빌드 후 시작
docker compose up -d --build

# 로그 확인
docker compose logs -f api

# 서비스 상태 확인
docker compose ps

# 서비스 중지 (볼륨 유지)
docker compose down

# 서비스 중지 및 볼륨 삭제
docker compose down -v

# 특정 서비스만 재시작
docker compose restart api
```

## 이미지 관리와 배포 전략

### 태그 전략

{% raw %}
```bash
# 버전 태그
docker build -t myapp:1.0.0 .
docker build -t myapp:latest .

# 환경별 태그
docker build -t myapp:staging .
docker build -t myapp:production .

# 다이제스트 고정 (재현성 보장)
docker pull nginx:alpine
docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine
# 출력: nginx@sha256:abcdef123456...

docker run --rm nginx@sha256:abcdef123456...
```
{% endraw %}

### 이미지 저장 및 전송

```bash
# 이미지 저장 (오프라인 배포용)
docker save -o myapp-1.0.0.tar myapp:1.0.0

# 이미지 로드
docker load -i myapp-1.0.0.tar

# 레지스트리 푸시
docker tag myapp:1.0.0 myregistry.com/myapp:1.0.0
docker push myregistry.com/myapp:1.0.0

# 레지스트리에서 풀
docker pull myregistry.com/myapp:1.0.0
```

## 언어별 최적화 팁

### Python

```Dockerfile
FROM python:3.12-slim

# 바이트코드 생성 방지 및 버퍼링 비활성화
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# 의존성 파일 분리 복사 (캐시 최적화)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 애플리케이션 코드 복사
COPY . .

# 비루트 사용자 실행
USER 65534:65534  # nobody 사용자

CMD ["python", "app.py"]
```

### Node.js

```Dockerfile
FROM node:20-alpine

WORKDIR /app

# 패키지 파일 복사 및 의존성 설치 (캐시 활용)
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# 애플리케이션 코드 복사
COPY . .

# 포트 노출
EXPOSE 3000

# 비루트 사용자 실행
USER node

CMD ["npm", "start"]
```

### Java

```Dockerfile
# 빌드 스테이지
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# 런타임 스테이지
FROM eclipse-temurin:21-jre
WORKDIR /app

# JAR 파일 복사
COPY --from=build /app/target/*.jar app.jar

# 최적화된 JVM 옵션
ENTRYPOINT ["java", \
    "-XX:+UseZGC", \
    "-XX:+ZGenerational", \
    "-Xmx512m", \
    "-jar", \
    "/app/app.jar"]
```

## 문제 해결 가이드

Docker 이미지 빌드와 실행 과정에서 발생하는 일반적인 문제들에 대한 해결 방법입니다.

### 빌드 속도 문제

**증상**: 빌드가 매우 느림
**원인**: 
- 과도한 빌드 컨텍스트
- 캐시 미활용
- 네트워크 지연

**해결**:
```bash
# .dockerignore 파일 최적화
# 레이어 순서 재조정 (자주 변경되는 부분을 아래로)
# BuildKit 캐시 마운트 사용
# 프록시 설정 확인
```

### 디스크 공간 부족

**증상**: `no space left on device` 오류
**원인**: 사용하지 않는 이미지, 컨테이너, 볼륨 누적

**해결**:
```bash
# 디스크 사용량 확인
docker system df

# 사용하지 않는 리소스 정리
docker system prune -af

# 특정 리소스만 정리
docker image prune -af
docker container prune -f
docker volume prune -f
```

### 컨테이너 즉시 종료

**증상**: 컨테이너가 시작 후 바로 종료됨
**원인**: 
- CMD/ENTRYPOINT 명령이 종료
- 애플리케이션 오류
- 포트 충돌

**해결**:
```bash
# 로그 확인
docker logs <container_name>

# 상세 정보 확인
docker inspect <container_name>

# 인터랙티브 모드로 테스트
docker run -it --rm <image_name> sh
```

### 포트 접근 불가

**증상**: `curl: (7) Failed to connect to localhost port 8080`
**원인**: 
- 포트 매핑 오류
- 애플리케이션 포트 미리스닝
- 방화벽/SELinux

**해결**:
```bash
# 컨테이너 포트 확인
docker port <container_name>

# 컨테이너 내부에서 테스트
docker exec <container_name> curl http://localhost:5000

# 호스트 포트 사용 확인
# Linux/Mac
lsof -i :8080

# Windows
netstat -ano | findstr :8080
```

## Windows/WSL2 환경 특별 고려사항

### 파일 시스템 성능

Windows에서 WSL2를 사용할 때, 호스트 Windows 파일 시스템을 Docker 컨테이너에 마운트하면 성능 저하가 발생할 수 있습니다.

**권장 접근법**:
```bash
# Windows 경로 (느림)
docker run -v /mnt/c/Users/name/project:/app ...

# WSL2 내부 경로 (빠름)
docker run -v /home/name/project:/app ...
```

프로젝트를 WSL2 파일 시스템 내에 위치시켜 작업하면 파일 I/O 성능이 크게 향상됩니다.

### 줄바꿈 문자 문제

Windows와 Linux의 줄바꿈 문자 차이(CRLF vs LF)로 인해 스크립트 실행이 실패할 수 있습니다.

**해결**:
```Dockerfile
# Dockerfile 내에서 변환
RUN sed -i 's/\r$//' entrypoint.sh && chmod +x entrypoint.sh
```

또는 Git 설정에서 자동 변환을 비활성화:
```bash
git config --global core.autocrlf false
```

## 결론

효율적이고 안전한 Docker 이미지를 생성하는 것은 현대적인 소프트웨어 개발과 배포 파이프라인의 핵심 요소입니다. 이번 포스트에서 다룬 주요 개념들을 정리해보면:

1. **레이어 캐시 원리 이해**: Dockerfile 작성 시 변경 빈도에 따른 레이어 배치로 빌드 시간 최소화
2. **멀티스테이지 빌드 활용**: 빌드 도구와 런타임 환경 분리로 이미지 크기 최소화 및 보안 강화
3. **보안 모범 사례 적용**: 비루트 사용자 실행, 최소 권한 원칙, 읽기 전용 파일시스템 등 기본적인 보안 조치 구현
4. **.dockerignore 파일 관리**: 빌드 컨텍스트 최소화로 빌드 효율성 향상
5. **BuildKit 고급 기능 활용**: 캐시 마운트, 시크릿 관리 등 현대적인 빌드 기능 적극 사용
6. **재현성 보장**: 다이제스트 기반 이미지 참조로 동일한 빌드 결과 보장

이러한 원칙들을 실제 프로젝트에 적용하면 더 빠르고 안전하며 관리하기 쉬운 Docker 이미지를 생성할 수 있습니다. Docker 이미지 설계는 단순한 기술 작업이 아닌, 애플리케이션의 배포, 확장, 보안을 고려한 종합적인 아키텍처 설계의 일환으로 접근해야 합니다.

각 프로젝트의 특성에 맞게 이러한 패턴들을 조정하고 적응시키는 과정에서 Docker 활용 능력이 점차 성장할 것입니다. 처음에는 기본 패턴을 따르다가 점차 프로젝트 요구사항에 맞게 최적화해나가는 접근법을 권장합니다.