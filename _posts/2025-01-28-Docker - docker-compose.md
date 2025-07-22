---
layout: post
title: Docker - exec vs attach
date: 2025-01-28 21:20:23 +0900
category: Docker
---
# 📄 `docker-compose.yml` 문법 완전 정복

---

## 📌 기본 개념

`docker-compose.yml` 파일은 여러 개의 **컨테이너 설정을 YAML 형식으로 구성**합니다.

### ▶️ 실행 명령어 요약

```bash
docker-compose up -d      # 백그라운드 실행
docker-compose down       # 종료 및 네트워크/컨테이너 삭제
docker-compose ps         # 상태 확인
docker-compose logs       # 로그 보기
```

---

## 🧱 YAML 기본 구조

```yaml
version: '3.9'       # 파일 형식 버전 (필수)
services:            # 실행할 컨테이너 정의
  <서비스명>:
    image: <이미지명>
    build: .
    ports:
      - "8080:80"
    volumes:
      - ./app:/app
    environment:
      - ENV_VAR=value
networks:            # (선택) 사용자 정의 네트워크
volumes:             # (선택) 사용자 정의 볼륨
```

---

## 🔍 주요 키워드 및 설명

### ✅ version

```yaml
version: '3.9'
```

- `2`, `3`, `3.9` 등 여러 버전 존재 (Docker 버전 호환에 따라)
- 권장: 최신 Compose 사용 시 `3.9` 또는 `3`

---

### ✅ services

컨테이너를 의미하며, 여러 개 정의 가능

```yaml
services:
  web:
    image: nginx
```

---

### ✅ image

- 이미지를 사용할 때

```yaml
image: nginx:latest
```

---

### ✅ build

- 이미지를 직접 빌드할 때 사용

```yaml
build:
  context: ./app
  dockerfile: Dockerfile.dev
```

또는 간단히

```yaml
build: .
```

---

### ✅ ports

- 외부 ↔ 컨테이너 포트 매핑

```yaml
ports:
  - "8080:80"
  - "443:443"
```

---

### ✅ volumes

- 호스트 ↔ 컨테이너 디렉토리 마운트

```yaml
volumes:
  - ./data:/var/lib/mysql
  - logs:/var/log/nginx     # 사용자 정의 볼륨
```

---

### ✅ environment

- 환경 변수 지정

```yaml
environment:
  - MYSQL_ROOT_PASSWORD=secret
  - MYSQL_DATABASE=mydb
```

또는 키-값 방식

```yaml
environment:
  MYSQL_ROOT_PASSWORD: secret
```

---

### ✅ env_file

```yaml
env_file:
  - .env
```

`.env` 파일의 환경 변수를 자동으로 가져옴

---

### ✅ depends_on

- 의존성 설정 (순서 보장 X, 단순 실행 순서)

```yaml
depends_on:
  - db
```

---

### ✅ networks

```yaml
networks:
  - backend
```

→ 아래에서 `backend` 정의 필요

---

### ✅ restart

- 재시작 정책

```yaml
restart: always        # 항상 재시작
restart: on-failure    # 실패 시 재시작
restart: unless-stopped
```

---

### ✅ command / entrypoint

- 컨테이너 시작 시 명령어 지정

```yaml
command: gunicorn app:app
entrypoint: /start.sh
```

---

### ✅ extra_hosts

- 컨테이너 내 `/etc/hosts` 추가 설정

```yaml
extra_hosts:
  - "local.dev:127.0.0.1"
```

---

### ✅ healthcheck

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 30s
  timeout: 10s
  retries: 3
```

---

### ✅ logging

```yaml
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"
```

---

## 🌐 네트워크 정의

```yaml
networks:
  backend:
    driver: bridge
```

- 기본은 `bridge`
- `overlay`, `macvlan` 등도 설정 가능

---

## 💾 볼륨 정의

```yaml
volumes:
  logs:
  data:
    driver: local
```

- `logs`, `data`는 이름이 지정된 Docker volume으로 관리됨

---

## 📁 전체 예제

```yaml
version: '3.9'

services:
  app:
    build: .
    ports:
      - "8080:80"
    volumes:
      - .:/app
    environment:
      - DEBUG=true
    networks:
      - frontend
    depends_on:
      - db

  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - db-data:/var/lib/mysql
    networks:
      - backend

networks:
  frontend:
  backend:

volumes:
  db-data:
```

---

## ✅ 유용한 옵션 요약

| 옵션 | 설명 |
|------|------|
| `build` | 이미지 직접 빌드 |
| `image` | 사용할 이미지 지정 |
| `ports` | 호스트 ↔ 컨테이너 포트 매핑 |
| `volumes` | 데이터 마운트 |
| `environment` | 환경 변수 |
| `depends_on` | 의존 컨테이너 설정 |
| `restart` | 재시작 정책 |
| `networks` | 사용자 정의 네트워크 연결 |
| `command` / `entrypoint` | 시작 명령 설정 |
| `logging` / `healthcheck` | 로깅 / 상태 체크 |

---

## 📚 참고 링크

- [Compose 파일 공식 문서](https://docs.docker.com/compose/compose-file/)
- [YAML 문법 공식 가이드](https://yaml.org/spec/)