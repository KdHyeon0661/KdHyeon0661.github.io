---
layout: post
title: Docker - exec vs attach
date: 2025-01-28 21:20:23 +0900
category: Docker
---
# Docker Compose 문법 완벽 가이드

Docker Compose는 다중 컨테이너 애플리케이션을 정의하고 실행하기 위한 도구입니다. 이 가이드에서는 Compose 파일의 문법을 심층적으로 살펴보고, 실무에서 바로 적용할 수 있는 패턴과 모범 사례를 제공합니다.

## 기본 명령어 요약

```bash
# Compose V2 사용 권장 (docker compose)
docker compose up -d           # 백그라운드 실행
docker compose down            # 중지 및 리소스 정리
docker compose ps              # 서비스 상태 확인
docker compose logs -f [서비스명] # 로그 실시간 확인
docker compose exec [서비스명] sh  # 컨테이너 내부 접속
docker compose restart [서비스명] # 서비스 재시작
docker compose stop [서비스명]    # 서비스 정지

# 구성 검증
docker compose config          # 최종 YAML 구성 확인
```

---

## Compose 파일 구조 이해하기

### 기본 템플릿

```yaml
version: "3.9"  # Compose 파일 형식 버전

services:
  # 서비스 정의들
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    networks:
      - frontend

  api:
    build: ./api
    networks:
      - frontend
      - backend
    depends_on:
      - database

  database:
    image: postgres:15
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - backend

networks:
  frontend:
  backend:

volumes:
  postgres-data:
```

### 핵심 구성 요소

1. **services**: 실행할 컨테이너들 정의
2. **networks**: 서비스 간 통신을 위한 네트워크 정의
3. **volumes**: 데이터 영속성을 위한 볼륨 정의
4. **secrets/configs**: 민감 정보와 설정 파일 관리

---

## 서비스 정의 상세 분석

### 이미지와 빌드

```yaml
services:
  # 레지스트리에서 이미지 사용
  web:
    image: nginx:alpine
    image: myregistry.com/app:1.0.0  # 사설 레지스트리에서도 가능

  # 로컬에서 빌드
  api:
    build:
      context: ./api  # 빌드 컨텍스트 디렉터리
      dockerfile: Dockerfile.prod  # Dockerfile 지정
      target: runtime  # 멀티스테이지 빌드에서 특정 스테이지 선택
      args:
        NODE_ENV: production
        BUILD_NUMBER: ${BUILD_NUMBER}
      cache_from:
        - type=registry,ref=myorg/app:buildcache
      labels:
        maintainer: "devops@company.com"
```

### 포트 매핑

```yaml
ports:
  - "8080:80"                # 기본 TCP
  - "8443:443/tcp"           # 프로토콜 명시적 지정
  - "8081:8081/udp"          # UDP 포트
  - "127.0.0.1:8082:80"      # 특정 IP에만 바인딩
  - "9000-9010:9000-9010"    # 포트 범위 매핑
```

**모범 사례**: 필요한 포트만 최소한으로 노출하고, 내부 통신은 네트워크 이름을 사용하세요.

### 환경 변수 관리

```yaml
environment:
  NODE_ENV: production
  DB_HOST: database
  LOG_LEVEL: ${LOG_LEVEL:-info}  # 기본값 설정
  API_KEY: ${API_KEY}            # 외부 변수 사용

env_file:
  - .env                         # 환경 변수 파일
  - .env.production              # 여러 파일 지정 가능
```

**주의**: 민감한 정보는 `secrets`를 사용하는 것이 더 안전합니다.

### 볼륨 마운트

```yaml
volumes:
  # 바인드 마운트 (개발용)
  - "./src:/app/src:ro"          # 읽기 전용
  - "./logs:/var/log/app:rw"     # 읽기/쓰기
  
  # 명명된 볼륨 (운영용)
  - "app-data:/data"
  
  # 익명 볼륨
  - "/cache"
  
  # tmpfs (메모리 파일시스템)
  - type: tmpfs
    target: /tmp
    tmpfs:
      size: 100000000  # 100MB
```

### 서비스 의존성과 시작 순서

```yaml
depends_on:
  database:
    condition: service_healthy  # 건강 상태 확인 후 시작
  redis:
    condition: service_started  # 시작 후 기다림

# 또는 간단한 형식
depends_on:
  - database
  - redis
```

### 헬스 체크

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 30s          # 체크 간격
  timeout: 10s           # 응답 대기 시간
  retries: 3             # 재시도 횟수
  start_period: 40s      # 시작 후 초기 대기 시간
  start_interval: 5s     # 시작 기간 내 체크 간격
```

### 리소스 제한

```yaml
deploy:  # Swarm 모드에서만 동작
  resources:
    limits:
      cpus: '0.50'
      memory: 512M
    reservations:
      cpus: '0.25'
      memory: 256M

# 단일 호스트에서는 runtime 옵션 사용
mem_limit: 512m
mem_reservation: 256m
cpus: 0.5
```

---

## 네트워킹 구성

### 다중 네트워크를 활용한 서비스 분리

```yaml
services:
  frontend:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - public-network
      - internal-network

  backend:
    build: ./backend
    networks:
      - internal-network

  database:
    image: postgres:15
    networks:
      - database-network

networks:
  public-network:
    driver: bridge
    # ipam:
    #   config:
    #     - subnet: 172.20.0.0/16

  internal-network:
    driver: bridge
    internal: true  # 외부 접근 차단

  database-network:
    driver: bridge
    internal: true
```

### 외부 네트워크 사용

```yaml
networks:
  default:
    external: true
    name: my-existing-network
```

---

## 보안 강화 설정

```yaml
services:
  secure-app:
    image: myapp:latest
    user: "1000:1000"           # 비루트 사용자로 실행
    read_only: true             # 루트 파일시스템 읽기 전용
    tmpfs:
      - /tmp                    # 필요한 쓰기 권한 경로
      - /run
    
    cap_drop:
      - ALL                     # 모든 권한 제거
    
    cap_add:
      - CHOWN                   # 필요한 권한만 추가
      - SETGID
      - SETUID
    
    security_opt:
      - no-new-privileges:true  # 새 권한 획득 방지
      - seccomp:unconfined
    
    sysctls:
      net.core.somaxconn: "1024"
    
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
```

---

## Secrets와 Configs로 민감 정보 관리

### Secrets (비밀 정보)

```yaml
services:
  database:
    image: postgres:15
    secrets:
      - db-password
      - ssl-certificate
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db-password

secrets:
  db-password:
    file: ./secrets/db_password.txt
  
  ssl-certificate:
    external: true  # 외부에서 생성된 시크릿 사용
```

### Configs (설정 파일)

```yaml
services:
  web-server:
    image: nginx:alpine
    configs:
      - source: nginx-config
        target: /etc/nginx/nginx.conf
        mode: 0440  # 파일 권한 설정

configs:
  nginx-config:
    file: ./configs/nginx.conf
  
  app-settings:
    content: |
      debug: false
      log_level: info
```

---

## 프로파일을 활용한 환경별 구성

```yaml
services:
  web:
    image: nginx:alpine
    profiles: ["web", "production"]

  api:
    build: ./api
    profiles: ["api"]

  database:
    image: postgres:15
    profiles: ["database", "production"]

  redis:
    image: redis:alpine
    profiles: ["cache"]

  monitoring:
    image: grafana/grafana
    profiles: ["monitoring"]
```

사용 방법:
```bash
# 특정 프로파일만 실행
docker compose --profile monitoring up -d

# 여러 프로파일 조합
docker compose --profile web --profile api up -d
```

---

## YAML 재사용을 위한 앵커와 별칭

```yaml
# 공통 설정 정의
x-common: &common
  restart: unless-stopped
  networks:
    - app-network
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"

x-database: &database
  <<: *common
  volumes:
    - /data
  environment:
    MAX_CONNECTIONS: 100

# 서비스 정의에서 재사용
services:
  api:
    <<: *common
    image: myapp/api:latest
    depends_on:
      - postgres
      - redis

  postgres:
    <<: *database
    image: postgres:15
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser

  redis:
    <<: *common
    image: redis:alpine
    command: redis-server --appendonly yes
```

---

## 환경별 구성 파일 오버레이

파일 구조:
```
docker-compose.yml      # 기본 구성
docker-compose.override.yml  # 개발 환경 오버레이
docker-compose.prod.yml      # 프로덕션 환경 오버레이
```

### 기본 구성 (docker-compose.yml)
```yaml
version: "3.9"

services:
  app:
    image: myapp:${TAG:-latest}
    networks:
      - app-network

networks:
  app-network:
```

### 개발 오버레이 (docker-compose.override.yml)
```yaml
services:
  app:
    build: .
    volumes:
      - ./src:/app/src
    environment:
      NODE_ENV: development
      DEBUG: "true"
    ports:
      - "3000:3000"
```

### 프로덕션 오버레이 (docker-compose.prod.yml)
```yaml
services:
  app:
    image: myregistry.com/myapp:${BUILD_NUMBER}
    environment:
      NODE_ENV: production
    deploy:
      replicas: 3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
```

사용 방법:
```bash
# 개발 환경 (기본 + 오버레이)
docker compose up -d

# 프로덕션 환경
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 구성 확인
docker compose -f docker-compose.yml -f docker-compose.prod.yml config
```

---

## 실전 예제: 완전한 3계층 애플리케이션

```yaml
version: "3.9"

services:
  # 프론트엔드 레이어
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - static-files:/var/www/static
    networks:
      - frontend-network
    depends_on:
      - web-app
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s

  # 애플리케이션 레이어
  web-app:
    build:
      context: ./app
      dockerfile: Dockerfile
      target: production
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://appuser:${DB_PASSWORD}@postgres/appdb
      REDIS_URL: redis://redis:6379
    volumes:
      - app-logs:/app/logs
    networks:
      - frontend-network
      - backend-network
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s

  # 데이터 레이어
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD} --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - backend-network
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s

  # 모니터링 레이어 (선택적)
  prometheus:
    image: prom/prometheus:latest
    profiles: ["monitoring"]
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    networks:
      - monitoring-network
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'

networks:
  frontend-network:
  backend-network:
    internal: true
  monitoring-network:

volumes:
  postgres-data:
  redis-data:
  app-logs:
  static-files:
  prometheus-data:
```

---

## 문제 해결 가이드

### 일반적인 문제와 해결 방법

#### 포트 충돌
```bash
# 사용 중인 포트 확인
sudo lsof -i :8080
sudo ss -lntp | grep :8080

# 다른 포트 사용
docker compose down
# compose.yml에서 포트 번호 수정 후
docker compose up -d
```

#### 컨테이너 간 통신 문제
```bash
# 네트워크 확인
docker network ls
docker network inspect [프로젝트명]_default

# DNS 확인
docker compose exec [서비스명] nslookup [다른서비스명]
docker compose exec [서비스명] cat /etc/resolv.conf
```

#### 볼륨 권한 문제
```bash
# 호스트 디렉토리 권한 확인
ls -la ./data

# 컨테이너 내부 사용자 확인
docker compose exec [서비스명] id

# 볼륨 소유자 변경
sudo chown -R 1000:1000 ./data
```

#### 로그 분석
```bash
# 모든 서비스 로그 확인
docker compose logs

# 특정 서비스 로그
docker compose logs [서비스명]

# 실시간 로그 모니터링
docker compose logs -f [서비스명]

# 로그 필터링
docker compose logs [서비스명] | grep -i error
```

### 디버깅 명령어 모음
```bash
# 구성 확인
docker compose config

# 서비스 상태
docker compose ps
docker compose top

# 네트워크 연결 테스트
docker compose run --rm curlimages/curl http://web:80

# 컨테이너 내부 진입
docker compose exec [서비스명] sh
docker compose exec [서비스명] bash

# 일시적 테스트 컨테이너 실행
docker compose run --rm [서비스명] [명령어]
```

---

## 모범 사례 요약

1. **의미 있는 네트워크 분리**: 프론트엔드, 백엔드, 데이터베이스를 별도의 네트워크로 분리하세요.
2. **비밀 정보 관리**: 환경 변수 대신 Docker Secrets를 사용하세요.
3. **헬스 체크 구현**: 모든 서비스에 적절한 헬스 체크를 추가하세요.
4. **리소스 제한 설정**: 메모리와 CPU 사용량을 제한하여 호스트 시스템을 보호하세요.
5. **볼륨 전략**: 개발 환경에서는 바인드 마운트, 프로덕션 환경에서는 명명된 볼륨을 사용하세요.
6. **로그 관리**: JSON 파일 드라이버를 사용하여 로그 회전을 설정하세요.
7. **보안 강화**: 비루트 사용자로 실행하고, 불필요한 권한을 제거하세요.
8. **의존성 관리**: `depends_on`에 `condition: service_healthy`를 사용하세요.
9. **구성 검증**: 배포 전 `docker compose config`로 구성 파일을 검증하세요.
10. **환경 분리**: 프로파일이나 오버레이 파일을 사용하여 환경별 구성을 관리하세요.

---

## 결론

Docker Compose는 다중 컨테이너 애플리케이션을 효과적으로 관리하기 위한 강력한 도구입니다. 올바르게 사용하면 개발, 테스트, 프로덕션 환경을 일관되게 유지하면서도 각 환경의 특수한 요구사항을 충족할 수 있습니다.

가장 중요한 것은 Compose 파일을 단순한 설정 파일이 아니라 애플리케이션 인프라의 코드화된 표현으로 여기는 것입니다. 버전 관리 시스템에 Compose 파일을 포함시키고, 코드 리뷰 과정에서 인프라 구성도 함께 검토하는 습관을 들이세요.

초기에는 간단한 구성으로 시작하여 점진적으로 고급 기능을 추가하는 것이 좋습니다. 각 옵션이 왜 필요한지 이해하고, 팀의 워크플로우와 운영 요구사항에 맞게 조정하세요. 잘 구성된 Compose 파일은 애플리케이션의 신뢰성, 보안성, 유지보수성을 크게 향상시킬 수 있습니다.

마지막으로, Docker Compose는 지속적으로 발전하는 도구입니다. 새로운 기능과 모범 사례를 계속 학습하고, 자신의 프로젝트에 적용해 보는 것이 중요합니다.