---
layout: post
title: Docker - 환경변수와 `.env` 파일 관리
date: 2025-02-08 19:20:23 +0900
category: Docker
---
# Docker Compose 환경변수와 .env 파일 관리 완벽 가이드

Docker Compose에서 환경변수를 효과적으로 관리하는 것은 애플리케이션 구성의 핵심입니다. 이 가이드에서는 `.env` 파일, `environment`, `env_file`, 빌드 인자 등 다양한 환경변수 관리 방식을 깊이 있게 다루고, 실무에서 바로 적용할 수 있는 패턴을 제공합니다.

## 핵심 개념 이해

Docker Compose에서 환경변수는 여러 수준에서 관리될 수 있습니다:

1. **`.env` 파일**: Compose 파일 내에서 변수 치환(substitution)에 사용
2. **`environment:`**: 컨테이너 런타임 환경변수 설정
3. **`env_file:`**: 외부 파일의 키-값을 컨테이너 환경변수로 주입
4. **`build.args`**: 이미지 빌드 시점에 사용되는 변수

이러한 각각의 방법은 서로 다른 목적과 사용 시점을 가지고 있습니다.

---

## 1. `.env` 파일: 변수 치환의 핵심

### 기본 개념과 사용법

`.env` 파일은 Docker Compose가 YAML 파일을 파싱할 때 변수 치환에 사용하는 파일입니다. 이 파일 자체는 컨테이너 내부로 전달되지 않습니다.

**기본 규칙**:
- 파일 위치: `docker-compose.yml`과 같은 디렉터리 (기본값)
- 자동 로드: `docker compose up` 실행 시 자동으로 읽힘
- 파일 형식: dotenv 포맷 (줄마다 `KEY=VALUE`)

### 예제 `.env` 파일
```env
# 데이터베이스 설정
DB_HOST=database
DB_PORT=5432
DB_NAME=mydatabase
DB_USER=myuser
DB_PASSWORD=securepassword123

# 애플리케이션 설정
APP_PORT=8080
APP_ENVIRONMENT=development
LOG_LEVEL=info

# 네트워크 설정
WEB_PORT=80
API_PORT=3000
```

### 특수한 값 처리
```env
# 공백이 포함된 값은 따옴표로 감싸기
APP_NAME="My Application"

# 특수문자 포함
SECRET_KEY=abc123!@#$%^

# 여러 줄 값 (Docker Compose 2.x 이상)
MULTILINE_VALUE="line1\nline2\nline3"
```

### 다른 `.env` 파일 지정하기
```bash
# 특정 환경 파일 사용
docker compose --env-file .env.production up -d

# 테스트 환경
docker compose --env-file .env.test up -d

# 개발 환경 (기본값)
docker compose --env-file .env.development up -d
```

---

## 2. Compose 파일에서 환경변수 사용하기

### 변수 치환 문법

Compose YAML 파일에서 환경변수를 사용하는 기본 문법은 `${VARIABLE_NAME}`입니다.

```yaml
version: '3.9'

services:
  database:
    image: postgres:15
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "${DB_PORT}:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

  backend:
    build: ./backend
    ports:
      - "${API_PORT}:3000"
    environment:
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@database:5432/${DB_NAME}
      NODE_ENV: ${APP_ENVIRONMENT}
    depends_on:
      - database

  frontend:
    image: nginx:alpine
    ports:
      - "${WEB_PORT}:80"
    volumes:
      - ./frontend:/usr/share/nginx/html
```

### 기본값 설정
변수가 정의되지 않았을 때 기본값을 제공할 수 있습니다.

```yaml
services:
  app:
    image: myapp:latest
    environment:
      # DB_HOST가 정의되지 않았으면 localhost 사용
      DB_HOST: ${DB_HOST:-localhost}
      
      # DB_PORT가 정의되지 않았으면 5432 사용
      DB_PORT: ${DB_PORT:-5432}
      
      # LOG_LEVEL이 정의되지 않았으면 info 사용
      LOG_LEVEL: ${LOG_LEVEL:-info}
      
      # REQUIRED_VAR가 정의되지 않았으면 오류 발생
      REQUIRED_VAR: ${REQUIRED_VAR?REQUIRED_VAR 변수가 필요합니다}
```

### 변수 치환의 다양한 활용
```yaml
services:
  app:
    # 이미지 태그에 변수 사용
    image: myregistry.com/myapp:${APP_VERSION:-latest}
    
    # 빌드 컨텍스트에 변수 사용
    build:
      context: ./${APP_DIR:-app}
      args:
        BUILD_VERSION: ${BUILD_VERSION}
    
    # 포트 번호에 변수 사용
    ports:
      - "${HOST_PORT:-8080}:${CONTAINER_PORT:-80}"
    
    # 볼륨 경로에 변수 사용
    volumes:
      - "${LOG_PATH:-./logs}:/app/logs"
    
    # 레이블에 변수 사용
    labels:
      - "com.example.version=${APP_VERSION}"
      - "com.example.environment=${ENVIRONMENT}"
```

---

## 3. `environment:` vs `env_file:` 비교

### `environment:` 직접 정의
```yaml
services:
  app:
    image: myapp:latest
    environment:
      # 직접 값 정의
      APP_NAME: "My Application"
      DEBUG: "false"
      
      # 환경변수에서 값 가져오기
      API_KEY: ${API_KEY}
      
      # 기본값 설정
      LOG_LEVEL: ${LOG_LEVEL:-info}
```

### `env_file:` 외부 파일 사용
```yaml
services:
  app:
    image: myapp:latest
    env_file:
      # 단일 파일
      - .env.app
      
      # 여러 파일 (나중 파일이 우선순위)
      - .env.common
      - .env.secrets
      
      # 경로 지정
      - ./config/env/production.env
```

### `env_file` 예제
```env
# .env.app
APP_NAME=My Application
APP_VERSION=1.0.0
DEBUG=false
LOG_LEVEL=info

# .env.secrets
API_KEY=sk_test_1234567890abcdef
DATABASE_URL=postgresql://user:pass@host:5432/db
```

### 주요 차이점 요약

| 특성 | `environment:` | `env_file:` |
|------|---------------|-------------|
| **정의 방식** | YAML 파일 내 직접 정의 | 외부 파일에서 로드 |
| **변수 치환** | 지원 (${VAR} 사용 가능) | 지원 안 함 (원본 그대로) |
| **우선순위** | 높음 (env_file을 덮어씀) | 낮음 |
| **관리 편의성** | 소규모 설정에 적합 | 대규모 설정에 적합 |
| **보안** | YAML 파일에 노출 | 분리된 파일로 관리 가능 |

### 혼합 사용 예제
```yaml
services:
  app:
    image: myapp:latest
    env_file:
      - .env.common      # 공통 설정
      - .env.secrets     # 비밀 정보
    environment:
      # env_file의 값을 덮어씀
      DEBUG: ${DEBUG_OVERRIDE:-false}
      
      # 추가 설정
      EXTRA_CONFIG: "value"
```

---

## 4. 빌드 인자와 Dockerfile 연동

### 빌드 인자 정의
```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        # 기본값 없는 빌드 인자
        NODE_ENV: ${BUILD_NODE_ENV}
        
        # 기본값 있는 빌드 인자
        APP_VERSION: ${APP_VERSION:-1.0.0}
        
        # 직접 값 지정
        BUILD_TIMESTAMP: "2024-01-01"
```

### Dockerfile에서 사용
```dockerfile
# Dockerfile
ARG NODE_ENV
ARG APP_VERSION=0.0.0
ARG BUILD_TIMESTAMP

# 빌드 인자를 환경변수로 전환
ENV NODE_ENV=${NODE_ENV}
ENV APP_VERSION=${APP_VERSION}
ENV BUILD_TIMESTAMP=${BUILD_TIMESTAMP}

# 빌드 중 사용
RUN echo "Building version ${APP_VERSION} for ${NODE_ENV} environment"
```

---

## 5. 환경별 구성 관리 패턴

### 패턴 1: 환경별 `.env` 파일
```
project/
├── .env.development
├── .env.staging
├── .env.production
└── docker-compose.yml
```

```bash
# 개발 환경
docker compose --env-file .env.development up -d

# 스테이징 환경  
docker compose --env-file .env.staging up -d

# 프로덕션 환경
docker compose --env-file .env.production up -d
```

### 패턴 2: Compose 오버레이 파일
```
project/
├── docker-compose.yml
├── docker-compose.override.yml
├── docker-compose.prod.yml
└── .env
```

**기본 구성 (docker-compose.yml)**:
```yaml
version: '3.9'

services:
  app:
    image: ${APP_IMAGE:-myapp}:${APP_VERSION:-latest}
    environment:
      NODE_ENV: ${NODE_ENV:-development}
    ports:
      - "${APP_PORT:-3000}:3000"
```

**개발 오버레이 (docker-compose.override.yml)**:
```yaml
services:
  app:
    build: .
    volumes:
      - ./src:/app/src
    environment:
      DEBUG: "true"
      HOT_RELOAD: "true"
```

**프로덕션 오버레이 (docker-compose.prod.yml)**:
```yaml
services:
  app:
    image: myregistry.com/myapp:${APP_VERSION}
    environment:
      NODE_ENV: production
      DEBUG: "false"
    deploy:
      replicas: 3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
```

### 패턴 3: 프로파일 사용
```yaml
version: '3.9'

services:
  app:
    image: myapp:latest
    profiles: ["default", "app"]
    
  database:
    image: postgres:15
    profiles: ["default", "database"]
    
  redis:
    image: redis:alpine
    profiles: ["cache"]
    
  monitoring:
    image: grafana/grafana
    profiles: ["monitoring"]
```

```bash
# 기본 서비스만 실행
docker compose up -d

# 캐시 서비스 추가
docker compose --profile cache up -d

# 모든 서비스 실행
docker compose --profile cache --profile monitoring up -d
```

---

## 6. 보안 모범 사례

### 민감 정보 관리
```yaml
# 안전하지 않은 방법 (비추천)
environment:
  DB_PASSWORD: "supersecret123"
  API_KEY: "sk_test_abcdef123456"

# 안전한 방법 1: env_file 사용
env_file:
  - .secrets.env  # .gitignore에 추가

# 안전한 방법 2: Docker Secrets 사용 (Swarm 모드)
secrets:
  db_password:
    external: true
  api_key:
    file: ./secrets/api_key.txt
```

### `.env` 파일 보안
```bash
# .gitignore에 추가
/.env
/.env.*
/secrets/
*.secret
```

```env
# .env.example (템플릿 파일)
DB_HOST=localhost
DB_PORT=5432
DB_NAME=your_database
DB_USER=your_username
DB_PASSWORD=your_password
API_KEY=your_api_key
```

### 파일 기반 비밀 관리
```yaml
services:
  app:
    image: myapp:latest
    environment:
      # 비밀 파일 경로만 환경변수로 전달
      DB_PASSWORD_FILE: /run/secrets/db_password
      API_KEY_FILE: /run/secrets/api_key
    volumes:
      # 읽기 전용으로 마운트
      - ./secrets/db_password:/run/secrets/db_password:ro
      - ./secrets/api_key:/run/secrets/api_key:ro
```

애플리케이션 코드에서 파일 읽기:
```python
import os

def read_secret(file_path_env_var):
    file_path = os.getenv(file_path_env_var)
    if file_path and os.path.exists(file_path):
        with open(file_path, 'r') as f:
            return f.read().strip()
    return None

db_password = read_secret('DB_PASSWORD_FILE')
api_key = read_secret('API_KEY_FILE')
```

---

## 7. 실전 예제: 완전한 애플리케이션 스택

### 프로젝트 구조
```
myapp/
├── .env
├── .env.production
├── docker-compose.yml
├── docker-compose.prod.yml
├── backend/
│   ├── Dockerfile
│   └── src/
├── frontend/
│   └── build/
└── secrets/
    ├── db_password
    └── api_key
```

### `.env` (개발용)
```env
# 애플리케이션 설정
APP_NAME=MyApp
APP_ENV=development
APP_VERSION=1.0.0-dev

# 데이터베이스 설정
DB_HOST=localhost
DB_PORT=5432
DB_NAME=mydb_development
DB_USER=devuser

# 서비스 포트
BACKEND_PORT=3000
FRONTEND_PORT=8080
DATABASE_PORT=5432

# 기능 플래그
ENABLE_FEATURE_X=true
ENABLE_FEATURE_Y=false
```

### `.env.production` (프로덕션용)
```env
# 애플리케이션 설정
APP_NAME=MyApp
APP_ENV=production
APP_VERSION=1.2.3

# 데이터베이스 설정
DB_HOST=production-db
DB_PORT=5432
DB_NAME=mydb_production
DB_USER=produser

# 서비스 포트
BACKEND_PORT=3000
FRONTEND_PORT=80
DATABASE_PORT=5432

# 기능 플래그
ENABLE_FEATURE_X=false
ENABLE_FEATURE_Y=true
```

### `docker-compose.yml`
```yaml
version: '3.9'

services:
  # 데이터베이스 서비스
  database:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    ports:
      - "${DATABASE_PORT}:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./secrets/db_password:/run/secrets/db_password:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # 백엔드 API 서비스
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
      args:
        NODE_ENV: ${APP_ENV}
        APP_VERSION: ${APP_VERSION}
    ports:
      - "${BACKEND_PORT}:3000"
    environment:
      NODE_ENV: ${APP_ENV}
      DATABASE_URL: postgresql://${DB_USER}@database:5432/${DB_NAME}
      DATABASE_PASSWORD_FILE: /run/secrets/db_password
      API_KEY_FILE: /run/secrets/api_key
      APP_NAME: ${APP_NAME}
      APP_VERSION: ${APP_VERSION}
      ENABLE_FEATURE_X: ${ENABLE_FEATURE_X}
      ENABLE_FEATURE_Y: ${ENABLE_FEATURE_Y}
    volumes:
      - ./secrets/db_password:/run/secrets/db_password:ro
      - ./secrets/api_key:/run/secrets/api_key:ro
      - backend-logs:/app/logs
    depends_on:
      database:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s

  # 프론트엔드 서비스
  frontend:
    image: nginx:alpine
    ports:
      - "${FRONTEND_PORT}:80"
    volumes:
      - ./frontend/build:/usr/share/nginx/html:ro
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - backend
    environment:
      BACKEND_URL: http://backend:3000

networks:
  default:
    name: ${APP_NAME:-myapp}_network

volumes:
  postgres-data:
  backend-logs:
```

### `docker-compose.prod.yml` (프로덕션 오버레이)
```yaml
services:
  backend:
    image: myregistry.com/myapp-backend:${APP_VERSION}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        max_attempts: 3
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  frontend:
    deploy:
      replicas: 2
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

---

## 8. 문제 해결과 디버깅

### 일반적인 문제 해결

#### 문제 1: 환경변수가 치환되지 않음
```bash
# 현재 환경변수 확인
printenv | grep -i "변수명"

# Compose 구성 확인
docker compose config

# .env 파일 확인
cat .env

# 특정 서비스 환경변수 확인
docker compose exec [서비스명] env
```

#### 문제 2: 변수 충돌
```yaml
# 명시적 우선순위 설정
environment:
  # env_file의 값을 명시적으로 덮어씀
  LOG_LEVEL: ${LOG_LEVEL_OVERRIDE:-${LOG_LEVEL:-info}}
```

#### 문제 3: 빌드 인자 전달 실패
```bash
# 빌드 인자 확인
docker compose build --no-cache --progress=plain

# Dockerfile 디버깅
docker build --build-arg NODE_ENV=production -t test .
```

### 디버깅 명령어 모음
```bash
# 최종 Compose 구성 확인
docker compose config

# 환경변수 치환 결과 확인
docker compose config | grep -A5 -B5 "변수명"

# 컨테이너 환경변수 목록
docker compose exec [서비스명] printenv

# 특정 환경변수 값 확인
docker compose exec [サービス名] sh -c 'echo $변수명'

# 빌드 인자 확인
docker inspect [이미지명] | jq '.[0].Config.Labels'
```

---

## 9. 고급 패턴과 팁

### 동적 변수 생성
```yaml
services:
  app:
    image: myapp:latest
    environment:
      # 타임스탬프 생성
      BUILD_TIMESTAMP: ${BUILD_TIMESTAMP:-$(date -u +"%Y-%m-%dT%H:%M:%SZ")}
      
      # 호스트명 포함
      INSTANCE_ID: ${HOSTNAME:-unknown}_${RANDOM}
```

### 조건부 변수 설정
```yaml
services:
  app:
    image: myapp:latest
    environment:
      # APP_ENV가 production이면 특정 값 사용
      DEBUG: ${DEBUG:-$([ "${APP_ENV}" = "production" ] && echo "false" || echo "true")}
      
      # 다른 변수 기반 값 설정
      LOG_LEVEL: ${LOG_LEVEL:-$([ "${APP_ENV}" = "development" ] && echo "debug" || echo "info")}
```

### 변수 체이닝
```yaml
services:
  app:
    image: myapp:latest
    environment:
      BASE_URL: https://${DOMAIN:-example.com}
      API_URL: ${BASE_URL}/api
      GRAPHQL_URL: ${API_URL}/graphql
      WS_URL: ws://${DOMAIN:-example.com}/ws
```

---

## 결론

Docker Compose에서 환경변수를 효과적으로 관리하는 것은 안정적이고 유지보수 가능한 애플리케이션 배포의 핵심입니다. 올바른 환경변수 관리 전략을 수립하면 다음과 같은 이점을 얻을 수 있습니다:

1. **환경 간 일관성**: 개발, 스테이징, 프로덕션 환경을 동일한 Compose 파일로 관리
2. **보안 강화**: 민감 정보를 코드베이스에서 분리
3. **유연성**: 환경별 구성을 쉽게 변경 가능
4. **재사용성**: 동일한 인프라를 다양한 환경에 적용 가능

### 핵심 원칙 요약

1. **역할 분리**: `.env`는 치환용, `env_file`은 주입용으로 명확히 구분하세요.
2. **보안 우선**: 민감 정보는 파일이나 외부 비밀 관리 시스템으로 관리하세요.
3. **환경 분리**: 환경별 `.env` 파일과 Compose 오버레이를 활용하세요.
4. **검증 습관**: 배포 전 `docker compose config`로 구성을 항상 확인하세요.
5. **문서화**: `.env.example` 파일을 제공하여 필요한 환경변수를 문서화하세요.

### 시작점 제안

초기 프로젝트에서는 간단하게 시작하세요:

1. 기본 `.env` 파일로 시작
2. `environment:`를 사용해 직접 변수 정의
3. 프로젝트 성장에 따라 `env_file:`과 환경별 구성 도입
4. 프로덕션 환경에서는 보안 강화 패턴 적용

환경변수 관리는 한 번에 완벽하게 구현하기보다는 점진적으로 개선해 나가는 것이 중요합니다. 프로젝트의 요구사항과 팀의 워크플로우에 맞게 적절한 패턴을 선택하고, 지속적으로 개선해 나가시기 바랍니다.