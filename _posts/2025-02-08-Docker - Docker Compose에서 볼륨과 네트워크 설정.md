---
layout: post
title: Docker - Docker Compose에서 볼륨과 네트워크 설정
date: 2025-02-08 20:20:23 +0900
category: Docker
---
# 🧱 Docker Compose에서 볼륨과 네트워크 설정 완전 정복

---

## 📦 1. 볼륨(Volumes) 설정

---

### 🔹 볼륨이란?

- **컨테이너와 호스트 간 데이터 공유/보존**을 위한 디렉터리
- 컨테이너 삭제 시에도 유지 가능 (영속성 보장)
- Compose에서는 **2가지 방식** 지원:
  1. **이름 있는(named) 볼륨**
  2. **바인드 마운트(bind mount)**

---

### ✅ Named Volume 예시

```yaml
services:
  db:
    image: mysql
    volumes:
      - db-data:/var/lib/mysql

volumes:
  db-data:
```

- `db-data`는 Compose가 관리하는 **독립된 Docker 볼륨**  
- `/var/lib/mysql`은 MySQL 데이터 디렉터리

→ `docker volume ls`로 확인 가능

---

### ✅ Bind Mount 예시

```yaml
services:
  app:
    build: .
    volumes:
      - ./src:/app
```

- **호스트 디렉터리(`./src`)를 컨테이너 내 `/app`에 마운트**
- 개발 중 실시간 코드 변경 반영에 유리

---

### 📁 볼륨 선언 위치 비교

| 위치 | 예시 | 의미 |
|------|------|------|
| 서비스 안 | `volumes:` | 컨테이너 내부 마운트 설정 |
| 루트 레벨 | `volumes:` | Docker 볼륨 정의 (Named) |

---

### 🛠️ 볼륨 사용 팁

| 상황 | 전략 |
|------|------|
| 데이터베이스 | named volume 사용 (`db-data`) |
| 앱 개발 중 | bind mount로 코드 실시간 반영 (`./src:/app`) |
| 환경 분리 | `docker-compose.override.yml`에서 볼륨 다르게 설정 가능 |

---

## 🌐 2. 네트워크(Networks) 설정

---

### 🔹 Docker Compose 네트워크란?

- **서비스 간 통신을 위한 가상 네트워크**
- 서비스 이름이 곧 호스트명이 되므로 **DNS 기반 통신 가능**
- 기본적으로 `bridge` 드라이버 사용

---

### ✅ 사용자 정의 네트워크 예시

```yaml
services:
  web:
    image: nginx
    networks:
      - frontend

  app:
    build: .
    networks:
      - frontend
      - backend

  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
  backend:
```

- `web` ↔ `app` 은 `frontend` 네트워크에서 통신
- `app` ↔ `db` 는 `backend` 네트워크에서 통신
- `web` ↔ `db` 는 통신 불가 (네트워크 분리 효과)

---

### ✅ 네트워크에 driver 지정하기

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
```

- 기본은 `bridge`
- 고급 설정 시 `overlay`, `macvlan`, `host` 등도 사용 가능

---

### 🔎 서비스 이름으로 통신 가능

```yaml
# app 컨테이너 내부에서
ping db       # db 서비스에 ping
curl http://web  # web 서비스에 HTTP 요청
```

- Compose는 **자동 DNS 등록**을 해주기 때문에  
  같은 네트워크 내에서는 **서비스명**으로 통신 가능

---

### 📡 외부 네트워크 연결 (예: Nginx Proxy Network 공유)

```yaml
networks:
  nginx-net:
    external: true
```

- 기존의 공유 네트워크(`docker network create nginx-net`)에 컨테이너 연결

---

## 📋 실전 예시

```yaml
version: '3.9'

services:
  wordpress:
    image: wordpress
    ports:
      - "8080:80"
    volumes:
      - wp-data:/var/www/html
    networks:
      - frontend
    depends_on:
      - db

  db:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
    volumes:
      - db-data:/var/lib/mysql
    networks:
      - backend

networks:
  frontend:
  backend:

volumes:
  wp-data:
  db-data:
```

- **네트워크를 나눠서 통신 범위 최소화**
- **각 서비스에 대한 볼륨을 분리**하여 책임 명확화

---

## ✅ 요약 비교

### 🔸 볼륨

| 구분 | Named Volume | Bind Mount |
|------|--------------|------------|
| 생성 위치 | Compose 내부 관리 | 호스트 디렉토리 |
| 영속성 | O | O |
| 목적 | 데이터베이스, 업로드 파일 | 코드 동기화 |
| 유연성 | 덜 유연 (고정 경로) | 매우 유연 (자유로운 경로 설정) |

---

### 🔸 네트워크

| 구분 | 내용 |
|------|------|
| 기본 드라이버 | bridge |
| 서비스 이름 해석 | 가능 (내부 DNS) |
| 분리 목적 | 보안, 책임 분리 |
| 고급 네트워크 | overlay, macvlan 등 사용 가능 |

---

## 📚 참고 자료

- [Docker Compose 네트워크 공식 문서](https://docs.docker.com/compose/networking/)
- [Docker Volumes 공식 문서](https://docs.docker.com/storage/volumes/)
- [Docker Bind Mount 공식 문서](https://docs.docker.com/storage/bind-mounts/)
- [Docker External Networks](https://docs.docker.com/compose/networking/#use-a-pre-existing-network)
