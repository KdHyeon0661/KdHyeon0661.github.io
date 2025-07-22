---
layout: post
title: Docker - Docker Compose 예제: WordPress + MySQL 블로그 스택 구성
date: 2025-02-05 19:20:23 +0900
category: Docker
---
# 📝 Docker Compose 예제 : WordPress + MySQL 블로그 스택 구성하기

---

## 📌 목표

- WordPress 웹사이트와 MySQL 데이터베이스를 Docker로 구성
- Docker Compose로 두 서비스를 하나의 스택으로 관리
- 볼륨을 통해 데이터 영속성 확보
- 웹브라우저에서 직접 WordPress 블로그 설치 페이지 접근

---

## 📁 디렉터리 구조

```plaintext
wordpress-docker/
├── docker-compose.yml
└── (필요 시) .env
```

---

## 🐳 docker-compose.yml 구성

```yaml
version: '3.9'

services:
  wordpress:
    image: wordpress:6.5
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: wp_db
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD: wp_pass
    volumes:
      - wordpress_data:/var/www/html
    depends_on:
      - db
    networks:
      - wp_net

  db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_DATABASE: wp_db
      MYSQL_USER: wp_user
      MYSQL_PASSWORD: wp_pass
      MYSQL_ROOT_PASSWORD: root_pass
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - wp_net

volumes:
  wordpress_data:
  db_data:

networks:
  wp_net:
```

---

## 🔍 서비스 설명

| 서비스 | 역할 | 이미지 | 특징 |
|--------|------|--------|------|
| `wordpress` | 웹 프론트엔드 | `wordpress:6.5` | PHP + Apache 포함 |
| `db` | 데이터베이스 | `mysql:8.0` | WordPress 백엔드 저장소 |

---

## 🌐 환경변수 설명

### `wordpress` 서비스

| 변수 | 설명 |
|------|------|
| `WORDPRESS_DB_HOST` | MySQL 컨테이너의 호스트 이름 |
| `WORDPRESS_DB_NAME` | 사용할 데이터베이스 이름 |
| `WORDPRESS_DB_USER` | 접속 계정 |
| `WORDPRESS_DB_PASSWORD` | 계정 비밀번호 |

### `db` 서비스

| 변수 | 설명 |
|------|------|
| `MYSQL_DATABASE` | 최초 생성할 DB명 |
| `MYSQL_USER` | 사용자 계정 |
| `MYSQL_PASSWORD` | 사용자 계정 비밀번호 |
| `MYSQL_ROOT_PASSWORD` | 루트 계정 비밀번호 |

---

## 🗂️ 데이터 영속성 (Volumes)

| 볼륨 이름 | 위치 | 설명 |
|-----------|------|------|
| `wordpress_data` | `/var/www/html` | WordPress의 소스 및 업로드 데이터 |
| `db_data` | `/var/lib/mysql` | MySQL 데이터베이스 파일 |

---

## 🚀 실행 방법

```bash
# 1. 디렉터리로 이동
cd wordpress-docker

# 2. 컨테이너 실행
docker-compose up -d

# 3. 상태 확인
docker-compose ps
```

---

## 🌐 브라우저 접속

- 웹브라우저에서 [http://localhost:8080](http://localhost:8080) 접속
- WordPress 설치 마법사가 자동 실행됩니다.

---

## 🔐 보안 및 운영 팁

| 항목 | 권장사항 |
|------|----------|
| `MYSQL_ROOT_PASSWORD` | 반드시 강력하게 설정 |
| `.env` 파일로 비밀번호 관리 | 민감한 정보는 분리 |
| `docker-compose.override.yml` | dev/prod 환경 분리 |
| nginx + certbot | HTTPS 설정 (추가 구성) |

---

## ✅ 주의사항

- `depends_on`은 **순서만 보장**하고 MySQL이 완전히 실행되었는지는 보장하지 않음  
  → 해결: WordPress 내 재시도 로직 내장되어 있어 실무에선 대부분 문제 없음

---

## 📚 확장 아이디어

| 추가 구성 | 설명 |
|-----------|------|
| phpMyAdmin | MySQL UI를 위한 컨테이너 추가 |
| nginx + certbot | HTTPS TLS 암호화 적용 |
| wp-cli | WordPress 명령어 기반 운영 도구 |
| S3 백업 | WordPress/DB 데이터를 클라우드로 주기적 백업 |
| Cloudflare | DNS 및 CDN 설정 연동 |

---

## 🧩 phpMyAdmin 추가 구성 (선택)

```yaml
  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    restart: always
    ports:
      - "8081:80"
    environment:
      PMA_HOST: db
    depends_on:
      - db
    networks:
      - wp_net
```

- 브라우저에서 [http://localhost:8081](http://localhost:8081) 접속 가능

---

## 📚 참고 자료

- [WordPress Docker Hub](https://hub.docker.com/_/wordpress)
- [MySQL Docker Hub](https://hub.docker.com/_/mysql)
- [phpMyAdmin Docker Hub](https://hub.docker.com/r/phpmyadmin/phpmyadmin)
- [Docker Compose 공식 문서](https://docs.docker.com/compose/)
