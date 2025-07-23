---
layout: post
title: Docker - 환경변수와 `.env` 파일 관리
date: 2025-02-08 19:20:23 +0900
category: Docker
---
# ⚙️ Docker Compose의 환경변수와 `.env` 파일 관리

---

## 📌 1. 환경변수란?

환경변수(environment variable)는 **애플리케이션 외부에서 설정되는 값**으로,  
Docker Compose에서는 다음과 같은 이유로 사용됩니다:

| 목적 | 설명 |
|------|------|
| 보안성 | 비밀번호, 토큰 등 민감 정보 분리 |
| 유연성 | 설정 값 변경 시 코드 수정 없이 대응 가능 |
| 재사용성 | 다양한 환경(dev/stage/prod)에서 설정만 바꿔 사용 가능 |

---

## 📁 2. `.env` 파일 구조

- 기본 위치: `docker-compose.yml`과 **같은 디렉터리**
- 자동으로 로드됨 (별도 설정 필요 없음)

```env
# .env 파일 예시
MYSQL_ROOT_PASSWORD=root_pass
MYSQL_DATABASE=mydb
MYSQL_USER=myuser
MYSQL_PASSWORD=mypass
WEB_PORT=8080
```

- 주석은 `#`
- 따옴표(`"`)는 필요 없음 (공백 포함 시만 사용)

---

## 🧱 3. `docker-compose.yml`에서 환경변수 사용하는 방법

### ✅ 방법 1: `${변수명}` 문법으로 삽입

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

- `docker-compose up` 시 `.env` 파일을 자동으로 읽어 사용

---

### ✅ 방법 2: `env_file` 키워드로 명시

```yaml
services:
  app:
    build: .
    env_file:
      - .env
```

- `env_file`은 **컨테이너 내부의 환경변수로 전달**됨
- `.env`는 **Compose 자체에서 변수 치환용**  
→ 둘은 역할이 다르며 **동시에 사용 가능**

---

## 📎 4. 변수 기본값 지정

```yaml
environment:
  - DB_HOST=${MYSQL_HOST:-localhost}
  - DB_PORT=${MYSQL_PORT:-3306}
```

- `${VAR:-default}` 문법으로 **없을 경우 기본값 지정**

---

## 🔐 5. 보안에 대한 주의사항

| 위험 요소 | 해결책 |
|-----------|--------|
| `.env` 유출 | `.gitignore`에 추가 (`.env` 파일은 Git에 올리지 말 것) |
| 하드코딩된 비밀번호 | 반드시 환경변수 사용 |
| 비밀번호 노출 로그 | `docker inspect` 또는 `ps` 명령어로 확인 가능 → 주의 |

```bash
# .gitignore 예시
.env
```

---

## 🧪 6. 실습 예시

### `.env`

```env
WEB_PORT=8080
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=blog
MYSQL_USER=admin
MYSQL_PASSWORD=1234
```

### `docker-compose.yml`

```yaml
services:
  wordpress:
    image: wordpress
    ports:
      - "${WEB_PORT}:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
      WORDPRESS_DB_USER: ${MYSQL_USER}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}

  db:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
```

---

## 🛠️ 7. 실전 팁

| 상황 | 팁 |
|------|----|
| dev/prod 구분 | `.env.dev`, `.env.prod` 등으로 나누고 `env_file`로 명시 |
| GitHub 업로드 | `.env.example` 제공 (변수 이름만 보여주는 템플릿) |
| CI/CD 연동 | `.env`를 Secrets 환경에 저장하거나 Vault 연동 |

---

## 📚 참고 자료

- [Docker Compose 공식 문서 - 환경변수](https://docs.docker.com/compose/environment-variables/)
- [YAML 변수 치환 문법](https://docs.docker.com/compose/compose-file/compose-file-v3/#variable-substitution)
- [12 Factor App - 설정 분리 원칙](https://12factor.net/config)

---

## ✅ 요약 정리

| 구분 | 용도 | 변수 치환 여부 |
|------|------|----------------|
| `.env` | Compose 전역 설정 | O (`${VAR}` 문법으로 사용) |
| `env_file` | 컨테이너 환경 전달 | X (Compose 변수 치환 X) |
| `environment` | 직접 설정 | O (YAML 내부에 `${VAR}` 사용 시) |