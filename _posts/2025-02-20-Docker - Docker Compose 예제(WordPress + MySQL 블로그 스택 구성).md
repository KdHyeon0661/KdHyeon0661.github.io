---
layout: post
title: Docker - Docker Compose 예제(WordPress + MySQL 블로그 스택 구성)
date: 2025-02-20 19:20:23 +0900
category: Docker
---
# Docker Compose로 WordPress + MySQL 블로그 구성하기

## 개요

이 가이드는 Docker Compose를 사용하여 완전한 WordPress 블로그 환경을 구성하는 방법을 안내합니다. WordPress 웹사이트와 MySQL 데이터베이스를 Docker 컨테이너로 배포하고, 영속성 있는 데이터 저장, 보안 설정, 백업 전략까지 실전 운영에 필요한 모든 요소를 다룹니다.

## 시스템 아키텍처

이 블로그 시스템은 다음과 같은 아키텍처를 가지고 있습니다:

```
[Client Browser]
        │
        ▼
  (선택사항) Nginx Reverse Proxy + SSL
        │
        ▼
  WordPress (PHP + Apache)
        │
        ▼
  MySQL Database
```

## 프로젝트 구조

```plaintext
wordpress-docker/
├── docker-compose.yml
├── .env                    # 환경 변수 설정 (Git에 커밋하지 않음)
├── .env.example           # 환경 변수 템플릿
├── init/
│   ├── 01-create-db.sql   # 초기 데이터베이스 생성 스크립트
│   └── 02-init-user.sql   # 초기 사용자 설정 스크립트
├── nginx/                 # (선택) Nginx 설정 파일
│   └── conf.d/
└── backup/                # 백업 파일 저장 위치
```

## Docker Compose 설정 파일

다음은 완전한 WordPress + MySQL 스택을 정의하는 Docker Compose 파일입니다:

```yaml
version: '3.9'

services:
  wordpress:
    image: wordpress:6.5.5-php8.2-apache   # 명시적 버전 태그 사용
    depends_on:
      db:
        condition: service_healthy         # 데이터베이스가 준비된 후 시작
    ports:
      - "${WEB_PORT:-8080}:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: ${WP_DB_NAME:-wp_db}
      WORDPRESS_DB_USER: ${WP_DB_USER:-wp_user}
      WORDPRESS_DB_PASSWORD: ${WP_DB_PASSWORD:-wp_pass}
    volumes:
      - wordpress_data:/var/www/html
    networks:
      - wp_net
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost/wp-admin/install.php || curl -fsS http://localhost/ || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 10

  db:
    image: mysql:8.0.40
    command: --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_DATABASE: ${WP_DB_NAME:-wp_db}
      MYSQL_USER: ${WP_DB_USER:-wp_user}
      MYSQL_PASSWORD: ${WP_DB_PASSWORD:-wp_pass}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-root_pass}
    volumes:
      - db_data:/var/lib/mysql
      - ./init:/docker-entrypoint-initdb.d:ro   # 초기 SQL 스크립트 자동 실행
    networks:
      - wp_net
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -u${WP_DB_USER:-wp_user} -p${WP_DB_PASSWORD:-wp_pass} --silent"]
      interval: 10s
      timeout: 5s
      retries: 30
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

volumes:
  wordpress_data:
  db_data:

networks:
  wp_net:
    driver: bridge
```

### 주요 구성 요소 설명

1. **WordPress 서비스**: Apache와 PHP를 포함한 WordPress 컨테이너
2. **MySQL 데이터베이스**: WordPress의 데이터 저장소
3. **명명된 볼륨**: `wordpress_data`와 `db_data`로 데이터 영속성 보장
4. **사용자 정의 네트워크**: 서비스 간 통신을 위한 격리된 네트워크
5. **헬스 체크**: 서비스 의존성 관리를 위한 상태 확인 메커니즘

## 환경 변수 설정

`.env` 파일을 생성하여 민감한 정보와 설정 값을 관리합니다:

```env
# WordPress 웹 서버 포트
WEB_PORT=8080

# 데이터베이스 설정
WP_DB_NAME=wp_db
WP_DB_USER=wp_user
WP_DB_PASSWORD=strong_password_here
MYSQL_ROOT_PASSWORD=very_strong_root_password

# WordPress 사이트 설정 (선택사항)
WP_SITE_URL=http://localhost:8080
WP_SITE_TITLE="My Docker Blog"
WP_ADMIN_USER=admin
WP_ADMIN_PASS=admin_secure_password
WP_ADMIN_EMAIL=admin@example.com
```

**중요**: `.env` 파일은 Git에 커밋하지 마세요. 대신 `.env.example` 파일을 템플릿으로 제공하세요.

## 시스템 실행

### 초기 실행

```bash
# Docker Compose로 서비스 시작
docker compose up -d

# 서비스 상태 확인
docker compose ps

# 로그 확인
docker compose logs -f

# 브라우저에서 접속
# http://localhost:8080
```

첫 접속 시 WordPress 설치 마법사가 표시됩니다. 환경 변수에 관리자 정보를 설정했다면 설치가 자동으로 완료될 수 있습니다.

### 서비스 관리

```bash
# 서비스 중지
docker compose stop

# 서비스 시작
docker compose start

# 서비스 재시작
docker compose restart

# 서비스 종료 및 제거 (볼륨 유지)
docker compose down

# 서비스 완전 제거 (볼륨 포함)
docker compose down -v
```

## 데이터 관리 및 백업

### 볼륨 구조

| 볼륨 이름 | 컨테이너 경로 | 저장 내용 |
|-----------|---------------|-----------|
| `wordpress_data` | `/var/www/html` | WordPress 코어, 플러그인, 테마, 업로드 파일 |
| `db_data` | `/var/lib/mysql` | MySQL 데이터베이스 파일 |

### 백업 스크립트

```bash
#!/bin/bash
# backup.sh

set -euo pipefail

# 타임스탬프 생성
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="./backup/${TIMESTAMP}"
mkdir -p "${BACKUP_DIR}"

echo "백업 시작: ${BACKUP_DIR}"

# 데이터베이스 덤프
echo "데이터베이스 백업 중..."
docker compose exec -T db \
  mysqldump -u"${WP_DB_USER:-wp_user}" -p"${WP_DB_PASSWORD:-wp_pass}" \
  --single-transaction --routines --triggers \
  "${WP_DB_NAME:-wp_db}" > "${BACKUP_DIR}/database.sql"

# WordPress 파일 백업
echo "WordPress 파일 백업 중..."
docker compose exec -T wordpress \
  tar -czf - -C /var/www/html . > "${BACKUP_DIR}/wordpress-files.tar.gz"

# 백업 정보 기록
echo "백업 완료 시간: $(date)" > "${BACKUP_DIR}/backup-info.txt"
echo "데이터베이스 크기: $(du -h ${BACKUP_DIR}/database.sql | cut -f1)" >> "${BACKUP_DIR}/backup-info.txt"
echo "파일 백업 크기: $(du -h ${BACKUP_DIR}/wordpress-files.tar.gz | cut -f1)" >> "${BACKUP_DIR}/backup-info.txt"

echo "백업 완료: ${BACKUP_DIR}"
```

### 복구 스크립트

```bash
#!/bin/bash
# restore.sh

set -euo pipefail

if [ -z "$1" ]; then
  echo "사용법: $0 <백업디렉토리>"
  echo "예: $0 backup/20240101_120000"
  exit 1
fi

BACKUP_DIR="$1"

if [ ! -d "$BACKUP_DIR" ]; then
  echo "백업 디렉토리를 찾을 수 없습니다: $BACKUP_DIR"
  exit 1
fi

echo "복구 시작: $BACKUP_DIR"

# 서비스 중지
docker compose stop wordpress db

# 데이터베이스 복구
echo "데이터베이스 복구 중..."
cat "${BACKUP_DIR}/database.sql" | docker compose exec -T db \
  mysql -u"${WP_DB_USER:-wp_user}" -p"${WP_DB_PASSWORD:-wp_pass}" "${WP_DB_NAME:-wp_db}"

# WordPress 파일 복구
echo "WordPress 파일 복구 중..."
docker compose run --rm --entrypoint="" wordpress \
  sh -c "rm -rf /var/www/html/*"
cat "${BACKUP_DIR}/wordpress-files.tar.gz" | docker compose exec -T wordpress \
  tar -xzf - -C /var/www/html

# 파일 권한 설정
docker compose exec wordpress \
  chown -R www-data:www-data /var/www/html

# 서비스 재시작
docker compose start db wordpress

echo "복구 완료"
```

## 확장 기능

### phpMyAdmin 추가 (선택사항)

```yaml
phpmyadmin:
  image: phpmyadmin/phpmyadmin:5.2
  restart: unless-stopped
  ports:
    - "8081:80"
  environment:
    PMA_HOST: db
    UPLOAD_LIMIT: 50M
  depends_on:
    db:
      condition: service_healthy
  networks:
    - wp_net
```

phpMyAdmin은 `http://localhost:8081`에서 접근할 수 있습니다. 운영 환경에서는 접근 제한을 설정하는 것이 좋습니다.

### Nginx 리버스 프록시 및 SSL 설정

프로덕션 환경에서는 Nginx를 리버스 프록시로 사용하고 SSL을 적용하는 것이 좋습니다:

```yaml
nginx:
  image: nginx:alpine
  depends_on:
    wordpress:
      condition: service_healthy
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./nginx/conf.d:/etc/nginx/conf.d:ro
    - ./ssl:/etc/ssl:ro
    - ./certs:/etc/letsencrypt:ro
  networks:
    - wp_net
  restart: unless-stopped
```

Nginx 설정 파일 예시 (`nginx/conf.d/wordpress.conf`):

```nginx
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://wordpress:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Redis 캐시 추가 (성능 향상)

```yaml
redis:
  image: redis:alpine
  command: redis-server --appendonly yes
  volumes:
    - redis_data:/data
  networks:
    - wp_net
  restart: unless-stopped

volumes:
  redis_data:
```

WordPress에서 Redis를 사용하려면 Redis Object Cache 플러그인을 설치하고 `wp-config.php`에 다음 설정을 추가하세요:

```php
define('WP_REDIS_HOST', 'redis');
define('WP_REDIS_PORT', 6379);
define('WP_REDIS_TIMEOUT', 1);
define('WP_REDIS_READ_TIMEOUT', 1);
```

## 운영 모범 사례

### 보안 강화

1. **강력한 비밀번호 사용**: 데이터베이스 및 관리자 계정에 강력한 비밀번호 설정
2. **정기 업데이트**: WordPress 코어, 플러그인, 테마 정기 업데이트
3. **접근 제한**: 관리자 페이지 접근 IP 제한 설정
4. **SSL/TLS 적용**: 모든 통신 암호화
5. **보안 플러그인**: Wordfence 또는 iThemes Security와 같은 보안 플러그인 설치

### 모니터링 및 유지보수

1. **정기 백업**: 일일 또는 주간 백업 스케줄 설정
2. **로그 모니터링**: Docker 로그 및 WordPress 에러 로그 모니터링
3. **성능 모니터링**: 서버 자원 사용량 및 응답 시간 모니터링
4. **정기 점검**: 월간 보안 및 성능 점검 수행

### 문제 해결

| 문제 | 가능한 원인 | 해결 방법 |
|------|------------|-----------|
| 데이터베이스 연결 오류 | DB 서비스가 실행 중이지 않음 | `docker compose logs db`로 로그 확인 |
| WordPress 설치 페이지 반복 | `wp-config.php` 생성 실패 | 볼륨 권한 확인 및 수동 생성 |
| 업로드 실패 | 파일 권한 문제 | `chown -R www-data:www-data /var/www/html` 실행 |
| 느린 응답 속도 | 캐시 미설정 | Redis 캐시 구성 및 최적화 플러그인 설치 |
| SSL 리다이렉트 루프 | 프록시 설정 오류 | `X-Forwarded-Proto` 헤더 설정 확인 |

## 마이그레이션 가이드

기존 WordPress 사이트를 Docker 환경으로 마이그레이션하는 단계:

1. **기존 사이트 백업**: 데이터베이스 덤프 및 파일 아카이브
2. **새 환경 구성**: Docker Compose 파일 설정 및 실행
3. **데이터 이전**: 백업 파일을 새 컨테이너로 복원
4. **도메인 업데이트**: 필요 시 URL 변경 및 퍼머링크 재설정
5. **테스트**: 모든 기능이 정상 작동하는지 확인

## 결론

Docker Compose를 사용한 WordPress + MySQL 환경 구성은 개발, 스테이징, 프로덕션 환경 모두에서 일관된 배포와 관리를 가능하게 합니다. 이 가이드에서 제시한 구성은 단순한 로컬 개발 환경부터 프로덕션급 운영 환경까지 확장할 수 있는 기반을 제공합니다.

주요 장점:
- **일관된 환경**: 개발부터 프로덕션까지 동일한 환경 보장
- **쉬운 배포**: 단일 명령어로 전체 스택 배포 가능
- **확장성**: 필요에 따라 추가 서비스(Nginx, Redis 등) 쉽게 통합
- **이식성**: 다른 서버나 클라우드 환경으로 쉽게 이동 가능
- **관리 용이성**: 설정 파일 기반의 인프라 관리

이 구성을 시작점으로 삼아 조직의 요구사항에 맞게 추가 기능과 최적화를 적용하세요. 정기적인 백업, 보안 업데이트, 성능 모니터링은 안정적인 WordPress 운영의 핵심 요소임을 기억하시기 바랍니다.