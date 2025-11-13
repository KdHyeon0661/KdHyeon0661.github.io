---
layout: post
title: Docker - Docker Compose 예제(WordPress + MySQL 블로그 스택 구성)
date: 2025-02-20 19:20:23 +0900
category: Docker
---
# Docker Compose 예제 : WordPress + MySQL 블로그 스택 구성하기

## 1. 목표 & 아키텍처 개요

- **목표**
  - WordPress 웹사이트와 MySQL DB를 **Docker Compose**로 손쉽게 구성
  - **Named Volumes**로 영속성 보장
  - 브라우저에서 설치 페이지 접근 → 초기셋업 완료
  - 실전 운영까지 고려한 보안/백업/모니터링/확장 패턴 제공

- **아키텍처 요약**
  ```
  [Client Browser]
        │
        ▼
  (Optional) Nginx Reverse Proxy + Certbot (HTTPS)
        │            │
        │            └─> ACME/Let's Encrypt
        ▼
  WordPress (PHP+Apache)
        │
        ▼
  MySQL (InnoDB)
  ```

---

## 2. 폴더 구조

```plaintext
wordpress-docker/
├── docker-compose.yml
├── .env                     # (선택) Compose 변수 치환/포트/비밀번호 등
├── init/
│   ├── 01-create-db.sql     # (선택) 초기 SQL
│   └── 02-init-user.sql     # (선택) 초기 사용자/권한
└── backup/
    └── (백업 스크립트/덤프가 생성될 위치)
```

> `init/*.sql`은 MySQL 컨테이너 최초 기동 시 자동 실행(이미 데이터가 있으면 실행 안 됨).

---

## 3. 기본 docker-compose.yml (확장/강화 버전)

> 초안의 스택을 기반으로 **healthcheck**, **restart 정책**, **명시적 버전 태그**, **리소스 제한(옵션)**, **로깅 제한** 등을 추가했습니다.

```yaml
version: '3.9'

services:
  wordpress:
    image: wordpress:6.5.5-php8.2-apache   # 가급적 명시적 태그
    depends_on:
      db:
        condition: service_healthy         # MySQL healthcheck와 연동
    ports:
      - "${WEB_PORT:-8080}:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: ${WP_DB_NAME:-wp_db}
      WORDPRESS_DB_USER: ${WP_DB_USER:-wp_user}
      WORDPRESS_DB_PASSWORD: ${WP_DB_PASSWORD:-wp_pass}
      # Prod에서는 아래 값도 검토(멀티사이트/업로드 제한 등)
      # WORDPRESS_CONFIG_EXTRA: |
      #   define('WP_MEMORY_LIMIT', '256M');
    volumes:
      - wordpress_data:/var/www/html
    networks:
      - wp_net
    restart: unless-stopped
    # (선택) 리소스/로깅 정책
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
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
      - ./init:/docker-entrypoint-initdb.d:ro   # 초기 SQL 자동 적용(선택)
    networks:
      - wp_net
    restart: unless-stopped
    # MySQL 상태가 실제로 준비될 때까지 헬스체크
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

### 핵심 포인트
- **`depends_on.condition: service_healthy`**: 단순 순서가 아니라 DB **헬스체크 통과 후** WordPress가 의존하도록 함.
- **`docker-entrypoint-initdb.d`**: 초기 스키마/유저를 자동 세팅.
- **로깅/리소스**: json-file 로그 롤링, 메모리/CPU 제한은 운영 환경 안정성에 도움.
- **명시 태그**: `latest` 대신 **명시 버전**으로 재현성 확보.

---

## 4. .env 예시(민감값/포트 분리)

```env
WEB_PORT=8080
WP_DB_NAME=wp_db
WP_DB_USER=wp_user
WP_DB_PASSWORD=wp_pass
MYSQL_ROOT_PASSWORD=very_strong_root
```

> `.env`는 **Compose 변수 치환용**입니다. Git에는 올리지 말고 `.env.example`만 배포하세요.

---

## 5. 실행 & 초기 접근

```bash
# 1. 실행
docker compose up -d --build

# 2. 상태 확인
docker compose ps

# 3. 브라우저 접속
# http://localhost:8080
```

- 첫 접속 시 WordPress 설치 마법사가 진행됩니다(사이트 제목/관리자 계정 생성).

---

## 6. 볼륨 설계 & 백업/복구 전략

### 6.1 볼륨 역할
| 볼륨 | 컨테이너 경로 | 설명 |
|---|---|---|
| `wordpress_data` | `/var/www/html` | WP 코어/플러그인/테마/업로드 |
| `db_data` | `/var/lib/mysql` | InnoDB 데이터 파일(영속성 핵심) |

### 6.2 간단 백업 스크립트(덤프 + 웹 파일 아카이브)
```bash
#!/usr/bin/env bash
set -euo pipefail
TS="$(date +%Y%m%d_%H%M%S)"
BACKUP_DIR="./backup/${TS}"
mkdir -p "${BACKUP_DIR}"

# DB 덤프
docker compose exec -T db \
  mysqldump -u"${WP_DB_USER:-wp_user}" -p"${WP_DB_PASSWORD:-wp_pass}" \
  "${WP_DB_NAME:-wp_db}" > "${BACKUP_DIR}/db.sql"

# WordPress 파일 아카이브
docker compose exec -T wordpress \
  sh -lc "tar -C /var/www -czf - html" > "${BACKUP_DIR}/wordpress.tgz"

echo "Backup done: ${BACKUP_DIR}"
```

### 6.3 복구(주의: 운영 중 덮어쓰기 전에 스냅샷/정지)
```bash
# DB 복구(테이블 drop/생성 포함 시 주의)
cat backup/20240101_120000/db.sql | docker compose exec -T db \
  mysql -u"${WP_DB_USER:-wp_user}" -p"${WP_DB_PASSWORD:-wp_pass}" "${WP_DB_NAME:-wp_db}"

# 파일 복구(필요 시 기존 파일 백업 후)
cat backup/20240101_120000/wordpress.tgz | \
  docker compose exec -T wordpress sh -lc "tar -C /var/www -xzf -"
```

> 운영에서는 **정기 스냅샷/오브젝트 스토리지(R2/S3)/오프사이트** 백업도 반드시 설계하세요.

---

## 7. phpMyAdmin 추가(선택)

```yaml
  phpmyadmin:
    image: phpmyadmin/phpmyadmin:5.2
    restart: unless-stopped
    ports:
      - "8081:80"
    environment:
      PMA_HOST: db
      # PMA_USER/PMA_PASSWORD를 지정하지 않으면 로그인 페이지에서 직접 입력
    depends_on:
      db:
        condition: service_healthy
    networks:
      - wp_net
```

- 접속: `http://localhost:8081`
- 운영에서는 **접속 제한(IP allow)** 또는 **VPN 뒤에 배치** 권장.

---

## 8. wp-cli 자동화(설치/플러그인/테마/유저)

### 8.1 ad-hoc 실행
```bash
docker compose exec wordpress bash -lc \
  'curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar && \
   php wp-cli.phar --info && \
   php wp-cli.phar core version'
```

### 8.2 컨테이너 빌드에 내장(선택)
`wordpress`를 직접 `Dockerfile`로 래핑해 `wp` 바이너리 포함, 초기 설치 스크립트까지 자동화 가능:
```Dockerfile
FROM wordpress:6.5.5-php8.2-apache
RUN curl -sSLO https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
 && chmod +x wp-cli.phar \
 && mv wp-cli.phar /usr/local/bin/wp
COPY docker-entrypoint.d/ /docker-entrypoint.d/
```

`docker-entrypoint.d/01-wp-setup.sh` (예: 첫 부팅시 자동 설치 — 멱등성 주의)
```bash
#!/usr/bin/env bash
set -e
cd /var/www/html

if ! wp core is-installed --allow-root; then
  wp core install \
    --url="${WP_SITE_URL:-http://localhost}" \
    --title="${WP_SITE_TITLE:-MyBlog}" \
    --admin_user="${WP_ADMIN_USER:-admin}" \
    --admin_password="${WP_ADMIN_PASS:-change_me}" \
    --admin_email="${WP_ADMIN_EMAIL:-admin@example.com}" \
    --skip-email --allow-root

  wp plugin install classic-editor --activate --allow-root
  wp theme install twentytwentythree --activate --allow-root
fi
```

> 운영에서 자동 설치 스크립트는 **재기동 시 멱등성**(중복 실행 방지)을 확실히 하세요.

---

## 9. Nginx 리버스 프록시 & HTTPS(선택)

### 9.1 목적
- WordPress 컨테이너의 포트를 **내부로만 노출**하고, 외부는 Nginx가 SSL/TLS 종료.
- 도메인: `blog.example.com`

### 9.2 compose 예시(간단 reverse proxy)
```yaml
  nginx:
    image: nginx:1.27
    depends_on:
      wordpress:
        condition: service_healthy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/letsencrypt:ro
    networks:
      - wp_net
    restart: unless-stopped
```

`nginx/conf.d/blog.conf`
```nginx
server {
  listen 80;
  server_name blog.example.com;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl http2;
  server_name blog.example.com;

  ssl_certificate     /etc/letsencrypt/live/blog.example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/blog.example.com/privkey.pem;

  location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_pass http://wordpress:80;
  }
}
```

> 인증서는 **certbot 컨테이너**를 별도 운영하거나, 호스트에서 발급 후 `./certs`에 반영.
> 자동 갱신 시 **nginx reload**가 필요합니다.

---

## 10. 운영 보안 체크리스트

- **비밀번호/Secret 분리**: `.env`는 반드시 Git 제외. 운영은 Secret Manager/Vault로 관리.
- **최소 권한**: DB 계정은 `wp_db`만 접근 권한. 루트 사용 금지.
- **파일 권한**: `wp-content/uploads`는 쓰기 필요, 그 외 디렉토리는 읽기 위주.
  (가능하면 `DISALLOW_FILE_EDIT` 사용으로 관리자 편집기 차단)
- **업데이트 전략**: 코어/플러그인/테마는 정기 업데이트 & 스테이징 검증 후 반영.
- **WAF/CDN**: Cloudflare/WAF로 1차 보호, Rate-limit/캡차 고려.
- **로깅/모니터링**: Nginx/Apache/MySQL 슬로우 로그/에러 로그 롤링, Fail2ban(호스트), 보안 플러그인.

---

## 11. 성능 & 확장

- **OPcache/오브젝트 캐시**: Redis 캐시 연동(별도 컨테이너)로 DB 부하 완화.
- **업로드 공유 문제**: WordPress 복수 인스턴스(Scale-out) 시 업로드를 **공유 볼륨/S3 호환**으로 외부화.
- **DB 튜닝**: InnoDB 버퍼 풀, 연결 수, 슬로우쿼리 확인.
- **이미지 최적화**: WebP, 이미지 크기 제한.
- **페이지 캐시**: 정적 캐싱(플러그인) + Nginx microcache(고급) 조합.

---

## 12. 기존 사이트 마이그레이션(핵심 단계)

1. 원본 서버에서 DB 덤프:
   ```bash
   mysqldump -uUSER -p DBNAME > db.sql
   ```
2. 업로드/테마/플러그인 파일 아카이브:
   ```bash
   tar -czf wp-files.tgz /var/www/html
   ```
3. 새 스택 기동 후 **wp-files.tgz** 복원:
   ```bash
   cat wp-files.tgz | docker compose exec -T wordpress sh -lc "tar -C / -xzf -"
   ```
4. DB 복원:
   ```bash
   cat db.sql | docker compose exec -T db mysql -u${WP_DB_USER} -p${WP_DB_PASSWORD} ${WP_DB_NAME}
   ```
5. 도메인 변경 시 **URL Search/Replace**:
   ```bash
   docker compose exec wordpress wp search-replace \
     'https://old.example.com' 'https://blog.example.com' --all-tables --allow-root
   ```
6. 퍼머링크 재저장, 캐시 비우기, 미디어 경로 검증.

---

## 13. 트러블슈팅(증상→원인→조치)

| 증상 | 원인 | 조치 |
|---|---|---|
| `Error establishing a database connection` | DB 기동 지연/비번 오타/포트 불일치 | `depends_on.condition=healthy` 확인, `.env` 값 일치, `docker compose logs db` |
| WordPress가 설치 페이지 반복 | `wp-config.php` 권한/볼륨 이슈, DB 접근 실패 | `wordpress_data` 권한/소유자 확인, DB 연결 재검증 |
| 업로드 실패/권한 에러 | 컨테이너 사용자/퍼미션 | `chown -R www-data:www-data /var/www/html/wp-content` |
| 느린 응답 | 캐시 부재/이미지 비대/DB 튜닝 미흡 | Redis 캐시, 썸네일 전략, 슬로우로그 분석 |
| https 리다이렉션 루프 | 리버스 프록시 헤더 미전달 | `proxy_set_header X-Forwarded-Proto https;` 추가 및 WP_SITEURL/WP_HOME 재설정 |
| 플러그인/테마 충돌 | 버전 불일치/호환성 | 스테이징에서 검증, 문제 플러그인 비활성화 |

유용 명령:
```bash
docker compose logs -f wordpress
docker compose logs -f db
docker compose exec db mysql -u${WP_DB_USER} -p${WP_DB_PASSWORD} -e "SHOW DATABASES;"
docker compose exec wordpress wp core version --allow-root
```

---

## 14. Compose 파일(최소/기본/전체) 비교

### 14.1 최소 구성(학습/로컬)
```yaml
version: '3.9'
services:
  wordpress:
    image: wordpress:6.5
    ports: ["8080:80"]
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: wp_db
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD: wp_pass
    volumes: ["wordpress_data:/var/www/html"]
    depends_on: [db]
  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: wp_db
      MYSQL_USER: wp_user
      MYSQL_PASSWORD: wp_pass
      MYSQL_ROOT_PASSWORD: root_pass
    volumes: ["db_data:/var/lib/mysql"]
volumes: {wordpress_data:, db_data:}
```

### 14.2 본문 기본(헬스체크/로깅/리소스)
→ **3장 compose** 참조

### 14.3 운영 확장(Reverse Proxy/HTTPS/캐시/백업)
- Nginx+Certbot, Redis, 백업 자동화 크론 컨테이너 등을 추가.

---

## 15. 마무리 요약

- **핵심**: *Compose로 WordPress + MySQL를 빠르게 올리고*, **Named Volume**으로 영속성 확보, **헬스체크/로그/리소스**까지 챙긴다.
- **운영 필수**: 비밀값 분리(.env 제한), 정기 백업(덤프+파일), 보안(WAF/업데이트/권한), 모니터링.
- **확장**: Reverse Proxy + TLS, 캐시/오브젝트 스토리지, 스테이징-프로덕션 이원화, 오케스트레이션(K8s) 이전 대비.

---

## 부록 A) 초기 SQL 예시(`init/*.sql`)

```sql
-- 01-create-db.sql
CREATE DATABASE IF NOT EXISTS `wp_db` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 02-init-user.sql
CREATE USER IF NOT EXISTS 'wp_user'@'%' IDENTIFIED BY 'wp_pass';
GRANT ALL PRIVILEGES ON `wp_db`.* TO 'wp_user'@'%';
FLUSH PRIVILEGES;
```

> `.env`의 실제 값과 일치시키세요(운영에서는 강력한 비밀번호 사용).

---

## 부록 B) 운영 체크스크립트(예)

```bash
#!/usr/bin/env bash
set -euo pipefail

# 헬스체크 워드프레스 페이지
curl -fsS http://localhost:${WEB_PORT:-8080}/wp-login.php >/dev/null \
  && echo "[OK] WordPress HTTP reachable" \
  || { echo "[FAIL] WordPress not reachable"; exit 1; }

# DB 단순 쿼리
docker compose exec -T db \
  mysql -u"${WP_DB_USER:-wp_user}" -p"${WP_DB_PASSWORD:-wp_pass}" \
  -e "SELECT 1" "${WP_DB_NAME:-wp_db}" >/dev/null \
  && echo "[OK] DB query" \
  || { echo "[FAIL] DB query"; exit 2; }
```

---

## 참고(공식 문서 권장 탐독)
- WordPress Docker Hub / MySQL Docker Hub / phpMyAdmin Docker Hub
- Docker Compose 문서(환경변수/볼륨/네트워크/헬스체크)
- WordPress 운영 가이드(보안/성능/캐시/업데이트 수칙)

이 가이드에 따라 **로컬 학습 → 스테이징 → 프로덕션**까지 자연스럽게 확장할 수 있습니다.
필요 시 Nginx+TLS, Redis 캐시, 백업 자동화, 모니터링을 순차적으로 추가하세요.
