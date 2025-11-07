---
layout: post
title: Docker - 환경변수와 `.env` 파일 관리
date: 2025-02-08 19:20:23 +0900
category: Docker
---
# Docker Compose의 환경변수와 `.env` 파일 관리

## 0. 한눈에 보는 개념 맵

- **`.env`**: Compose가 **파일을 읽어 변수 치환**(substitution)에 사용. (파일 자체가 컨테이너 안으로 들어가진 않음)
- **`environment:`**: **컨테이너 런타임 환경변수** 설정.
- **`env_file:`**: 지정한 파일의 **키=값**을 컨테이너에 주입.
- **빌드 인자(`build.args`)**: **이미지 빌드 시점**에만 쓰이는 변수. Dockerfile의 `ARG`와 매칭.
- **치환 대상**: `image`, `build`, `ports`, `volumes`, `command`, `labels`, `deploy` 등 Compose YAML 안 대부분의 문자열 필드.

---

## 1. `.env` 파일 — 구조, 위치, 로딩

### 1.1 기본 규칙
- 위치: **`docker-compose.yml`와 같은 디렉터리** (기본값).  
- 자동 로드: `docker compose up` 시 자동 읽음.  
- 형식: **dotenv 포맷**(줄별 `KEY=VALUE`, `#` 주석).

```env
# .env
MYSQL_ROOT_PASSWORD=root_pass
MYSQL_DATABASE=mydb
MYSQL_USER=myuser
MYSQL_PASSWORD=mypass
WEB_PORT=8080
```

> 공백이 포함되면 `"값"` 같이 **따옴표를 사용**하세요.  
> CRLF(Windows)·BOM은 문제를 유발할 수 있으니 **UTF-8 / LF** 권장.

### 1.2 CLI로 다른 파일 지정
- v2에서는 `--env-file` 지원:
```bash
docker compose --env-file .env.prod up -d
```
- **여러 파일 동시 지정은 불가**. 필요하면 쉘에서 미리 `export`하거나 Makefile로 조합.

---

## 2. Compose 파일에서 환경변수 쓰는 법(치환)

### 2.1 `${VAR}` 치환(Variable Substitution)

```yaml
version: '3.9'
services:
  web:
    image: wordpress
    ports:
      - "${WEB_PORT}:80"
    environment:
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
      WORDPRESS_DB_USER: ${MYSQL_USER}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
      WORDPRESS_DB_HOST: db:3306

  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
```

- `${VAR}`는 **쉘 환경변수** 또는 **`.env`**에서 읽어 치환됩니다.

### 2.2 기본값/필수값 문법
- 기본값:
```yaml
environment:
  DB_HOST: ${MYSQL_HOST:-localhost}
  DB_PORT: ${MYSQL_PORT:-3306}
```
- 필수값(없으면 에러):
```yaml
image: my/app:${APP_VERSION?APP_VERSION가 필요합니다}
```

### 2.3 달러문자 이스케이프
- 리터럴 `${...}`가 필요하면 `$$` 사용:
```yaml
command: ["sh", "-lc", "echo $$HOME && echo \${NOT_EXPANDED}"]
```

---

## 3. `environment:` vs `env_file:` vs `.env` — 역할과 차이

| 구분 | 목적 | 어디에 쓰이나 | 치환 처리 | 우선순위(컨테이너에 주입될 때) |
|---|---|---|---|---|
| `.env` | **Compose YAML 변수 치환** | Compose 파서 레벨 | O | (치환용) |
| `environment:` | **컨테이너 런타임 환경변수** | 컨테이너 내부 | O(Compose 파서가 미리 치환) | **environment > env_file > Dockerfile `ENV`** |
| `env_file:` | 외부 파일의 키=값을 **컨테이너에 주입** | 컨테이너 내부 | **X**(파일 내용 그대로) | environment가 덮어씀 |

예시:
```yaml
services:
  app:
    build: .
    env_file:
      - .env.runtime        # 이 파일의 내용은 컨테이너로 "그대로" 주입
    environment:
      APP_ENV: ${APP_ENV:-dev}  # 치환됨
```

---

## 4. 빌드 인자(`build.args`)와 Dockerfile `ARG/ENV`

### 4.1 이미지 빌드 시점 변수

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        RUNTIME: "cpython"
        APP_VERSION: ${APP_VERSION:-1.0.0}
```

```Dockerfile
# Dockerfile
ARG RUNTIME
ARG APP_VERSION=0.0.0
ENV APP_VERSION=${APP_VERSION}
RUN echo "runtime=$RUNTIME, version=$APP_VERSION"
```

- `ARG`는 **빌드 시에만** 존재.  
- `ENV`는 **이미지/컨테이너 런타임에도** 남음.
- `build.args`는 Compose 치환 규칙을 그대로 적용(즉, `.env` 영향 받음).

---

## 5. 우선순위와 전파(Precedence)

### 5.1 Compose YAML 치환 우선순위(파일을 해석할 때)
1) **쉘 환경변수**(실행한 터미널/CI에서 `export VAR=value`)  
2) `--env-file`  
3) `.env`  
4) 없으면 기본값(`:-`) 또는 에러(`?msg`)

### 5.2 컨테이너 런타임 환경변수 우선순위
- `environment:` **>** `env_file:` **>** Dockerfile의 `ENV`  
- `env_file`의 동일 키는 `environment`가 **덮어씀**.

---

## 6. 멀티 환경(dev/stage/prod) 운영 패턴

### 6.1 `.env.*` + `--env-file`
```
.env.dev
.env.stage
.env.prod
```
```bash
docker compose --env-file .env.stage up -d
```

### 6.2 `docker-compose.override.yml` 자동 병합
- `docker-compose.yml`(공통) + `docker-compose.override.yml`(개발용 Overwrite)

`docker-compose.yml`:
```yaml
services:
  web:
    image: my/web:${APP_VERSION:-1.0.0}
    ports: ["${WEB_PORT:-8080}:80"]
```

`docker-compose.override.yml`:
```yaml
services:
  web:
    build: .
    volumes:
      - ./src:/app
    environment:
      APP_ENV: dev
```

운영에서는 override 없이:
```bash
docker compose up -d
```

### 6.3 프로파일(Profiles)
- `COMPOSE_PROFILES=debug` 환경변수 또는 `--profile dev` 사용:

```yaml
services:
  api:
    image: my/api
    profiles: ["default"]

  jaeger:
    image: jaegertracing/all-in-one
    profiles: ["debug"]
```

```bash
# 디폴트만
docker compose up -d
# 디버그 서비스 추가
docker compose --profile debug up -d
```

---

## 7. 보안/비밀값 관리 전략

| 항목 | 권장 |
|---|---|
| `.env` Git 업로드 | **금지**, `.gitignore`에 추가 |
| 공개 레포 공유 | `/.env.example` 템플릿 제공만 |
| 운영 비밀값 | CI/CD Secret Store, Vault, SSM Parameter Store 등 외부 비밀관리 연동 |
| 파일로 전달 | 민감값은 **파일로 마운트** 후 경로만 환경변수로 전달 (`ENV_FILE_PATH=/run/secrets/xxx`) |
| 로그 노출 | `docker inspect`로 env가 노출될 수 있음 → 민감값은 가능한 파일/볼륨로 전달 |

예시(파일 마운트):
```yaml
services:
  api:
    image: my/api
    environment:
      DB_PASSWORD_FILE: /run/secret/dbpass
    volumes:
      - ./secrets/dbpass:/run/secret/dbpass:ro
```

컨테이너 코드:
```python
import os, pathlib
p = os.getenv("DB_PASSWORD_FILE", "/run/secret/dbpass")
DB_PASSWORD = pathlib.Path(p).read_text().strip()
```

---

## 8. 실전 예제 — WordPress + MySQL(확장판)

### 8.1 `.env`
```env
WEB_PORT=8080
MYSQL_ROOT_PASSWORD=strong_root
MYSQL_DATABASE=blog
MYSQL_USER=wpuser
MYSQL_PASSWORD=strong_pass
APP_VERSION=1.2.3
```

### 8.2 `docker-compose.yml`
```yaml
version: '3.9'

services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - db-data:/var/lib/mysql
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -u root -p$${MYSQL_ROOT_PASSWORD} --silent"]
      interval: 5s
      retries: 20

  wordpress:
    image: wordpress:php8.2-apache
    ports:
      - "${WEB_PORT:-8080}:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
      WORDPRESS_DB_USER: ${MYSQL_USER}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
      APP_VERSION: ${APP_VERSION:-dev}
    depends_on:
      db:
        condition: service_healthy

volumes:
  db-data:
```

### 8.3 실행·검증
```bash
docker compose up -d
docker compose ps
curl -I http://localhost:${WEB_PORT}
```

---

## 9. 자주 묻는 질문(FAQ)과 함정

### Q1. `.env`와 `env_file`의 차이?
- `.env`는 **Compose 파일 변수 치환용**.
- `env_file`은 **컨테이너 환경변수 주입용**.
- 둘 다 쓰되, **치환은 `.env`**가, **주입은 `env_file`**이 담당.

### Q2. 왜 `${VAR}`가 안 치환되나?
- 현재 쉘의 `export`/`--env-file`/`.env` 어디에도 값이 없는 경우.
- 오타/대소문자, `--env-file` 경로, 실행 디렉터리 확인.
- 필요한 경우 기본값(`:-`)이나 필수값(`?`) 문법 사용.

### Q3. `env_file` 안의 `${VAR}`는 치환되나?
- **아니오.** `env_file`은 **그대로** 컨테이너에 전달됩니다.
- 치환이 필요하면 Compose YAML의 `environment:`에서 처리하세요.

### Q4. Windows에서 값이 이상하게 들어간다?
- 줄 끝 CRLF, 인코딩, PowerShell 변수 확장 규칙 차이 확인.
- `.env`는 **UTF-8/LF** 권장.  
- PowerShell에서 `${VAR}`는 일반 문자열로 남겨두고 Compose가 치환합니다.

### Q5. 민감값이 `docker inspect`에서 보인다?
- 네. 런타임 env는 메타데이터로 노출될 수 있습니다. **파일 마운트/외부 비밀관리**로 전환을 검토하세요.

---

## 10. 고급 패턴

### 10.1 서비스별 `.env` 분리(런타임 주입 전용)
```
.env             # 치환용 공통 (포트, 버전 등)
.env.db          # DB 컨테이너 런타임 변수
.env.web         # 웹 컨테이너 런타임 변수
```
```yaml
services:
  db:
    image: mysql:8
    env_file: [.env.db]
  web:
    image: my/web:${APP_VERSION}
    env_file: [.env.web]
```

### 10.2 빌드·런 분리
```yaml
services:
  api:
    build:
      context: .
      args:
        BUILD_MODE: ${BUILD_MODE:-release}   # 빌드 전용
    environment:
      RUNTIME_ENV: ${RUNTIME_ENV:-prod}      # 런타임 전용
```

### 10.3 여러 Compose 파일 오버레이
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```
- `prod.yml`에서 `image` 태그나 `environment`를 재정의.

### 10.4 프로젝트 이름 고정
```bash
export COMPOSE_PROJECT_NAME=myblog
docker compose up -d
```
- 네트워크/볼륨 접두사가 고정되어 **자원명이 예측 가능**.

---

## 11. 트러블슈팅 가이드

| 증상 | 점검 포인트 | 해결 |
|---|---|---|
| `${PORT}`가 글자 그대로 보임 | 현재 쉘 `export`, `--env-file`, `.env` 경로, 실행 디렉터리 | `.env`를 compose 파일과 같은 폴더에 두고 다시 실행 |
| `env_file` 안의 `${VAR}`가 확장됨 | 오해 | `env_file`은 literal. 치환은 `environment:`에서 처리 |
| 민감값 Git 유출 | `.gitignore` | `.env` 차단, `.env.example`만 제공 |
| 값에 공백/특수문자 | 인용부호/이스케이프 | `.env`에서 `"값"` 또는 `\` 사용, YAML에선 `'값'` |
| macOS I/O 느림 | Desktop 동기화 | `:cached`/`:delegated` 힌트, Bind Mount 최소화 |
| Windows 경로 문제 | 슬래시/드라이브/CRLF | `C:\path\to\file` 또는 `./path`; LF 권장 |

유용 명령:
```bash
# Compose 레벨에서 어떻게 치환됐는지 보기
docker compose config

# 컨테이너에 실제로 들어간 환경변수 확인
docker compose exec <svc> env | sort
```

---

## 12. 완전 예제 — FastAPI + PostgreSQL (프로파일·오버레이·빌드인자 포함)

### 12.1 `.env`
```env
APP_VERSION=2.0.0
HTTP_PORT=8088
POSTGRES_PASSWORD=secret
POSTGRES_DB=appdb
POSTGRES_USER=appuser
```

### 12.2 `docker-compose.yml`(공통)
```yaml
version: '3.9'

services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
    volumes:
      - db-data:/var/lib/postgresql/data
    networks: [ backend ]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      retries: 20

  api:
    build:
      context: ./api
      args:
        APP_VERSION: ${APP_VERSION}
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: "postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}"
      APP_VERSION: ${APP_VERSION}
    ports:
      - "${HTTP_PORT:-8080}:8080"
    networks: [ frontend, backend ]

networks:
  frontend:
  backend:

volumes:
  db-data:
```

### 12.3 `docker-compose.override.yml`(개발용)
```yaml
services:
  api:
    volumes:
      - ./api/src:/app
    environment:
      APP_ENV: dev
    profiles: ["default", "debug"]

  db:
    profiles: ["default"]
```

### 12.4 `api/Dockerfile`
```Dockerfile
FROM python:3.12-slim
ARG APP_VERSION=0.0.0
ENV APP_VERSION=${APP_VERSION}
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY src/ .
CMD ["uvicorn", "main:app", "--host=0.0.0.0", "--port=8080"]
```

### 12.5 실행
```bash
# 개발(override 자동 병합)
docker compose up -d

# 운영 (override 제외, 또는 prod 파일 오버레이)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 13. 요약 표

### 13.1 무엇을 어디에?

| 목적 | 권장 위치 |
|---|---|
| Compose YAML 치환 | `.env` / `--env-file` |
| 런타임 환경변수 | `environment:` (필요 시 `env_file:` 병행) |
| 빌드 시간 변수 | `build.args` + Dockerfile `ARG` |
| 민감값 | 파일 마운트/비밀관리(Secrets/SSM/Vault), `.env`는 Git 제외 |
| 다중 환경 | `.env.*` + `--env-file`, override, profiles |

### 13.2 우선순위 핵심
- **치환**: Shell 환경 > `--env-file` > `.env`
- **런타임 주입**: `environment:` > `env_file:` > Dockerfile `ENV`

---

## 14. 참고/검증 도구

```bash
# 최종 해석된 Compose 출력(치환 결과 확인)
docker compose config

# 컨테이너 내부 환경변수 리스트
docker compose exec <service> env | sort

# 특정 변수만 대조
docker compose exec <service> sh -lc 'echo "$APP_VERSION"'
```

---

## 15. 마무리

- `.env`는 **치환**, `env_file`은 **주입**이라는 역할 분담을 항상 기억하세요.  
- **빌드/런**의 생명주기 차이를 명확히 나눠 변수 사용을 설계하세요.  
- 보안은 **노출면(ports/env/log/inspect)**을 줄이고, 비밀은 파일/외부 비밀관리로 이관하세요.  
- `docker compose config`로 **치환 결과**를 늘 점검하면 배포 사고를 크게 줄일 수 있습니다.