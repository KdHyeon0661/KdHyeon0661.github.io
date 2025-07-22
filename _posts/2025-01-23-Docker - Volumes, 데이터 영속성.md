---
layout: post
title: Docker - Volumes, 데이터 영속성
date: 2025-01-23 19:20:23 +0900
category: Docker
---
# 📦 Docker Volumes: 데이터 영속성 유지하기

---

## 🔍 1. 왜 Volume이 필요한가?

Docker 컨테이너는 **휘발성**입니다.

- 컨테이너를 중지하거나 삭제하면 내부 데이터도 함께 사라집니다.
- 예: MySQL 컨테이너를 삭제하면 DB도 사라짐!

### ✅ 해결책: **Volume**
> 컨테이너와는 분리된 **독립적인 저장소**를 제공하여, 데이터를 보존할 수 있게 함

---

## 📦 2. Volume이란?

- Docker가 관리하는 **호스트의 디렉토리/파일 시스템 공간**
- 컨테이너에서 데이터를 저장/공유할 때 사용
- 컨테이너 삭제와 무관하게 **데이터는 유지**

### 📁 구조

```
컨테이너
   └─ /app/data (컨테이너 내부 경로)
        ⇅
       volume
   └─ /var/lib/docker/volumes/myvolume/_data (호스트 경로)
```

---

## 🧬 3. Volume 종류

| 종류 | 설명 | 사용 예시 |
|------|------|-----------|
| **Named Volume** | Docker가 자동으로 이름 붙이고 관리 | `docker volume create` |
| **Anonymous Volume** | 이름 없이 자동 생성 (일시적) | `-v /data` |
| **Bind Mount** | 호스트 디렉토리를 직접 연결 | `-v $(pwd)/data:/app/data` |

---

## 🛠️ 4. 기본 사용법

### ✅ 1. 볼륨 생성

```bash
docker volume create myvolume
```

### ✅ 2. 볼륨 연결하여 컨테이너 실행

```bash
docker run -d -v myvolume:/app/data --name mycontainer ubuntu
```

- `/app/data`는 컨테이너 내부 경로
- `myvolume`은 호스트에서 유지되는 실제 데이터 저장 위치

### ✅ 3. 볼륨 확인

```bash
docker volume ls
docker volume inspect myvolume
```

### ✅ 4. 볼륨 삭제

```bash
docker volume rm myvolume
```

> **주의**: 컨테이너가 해당 볼륨을 사용 중이면 삭제 불가

---

## 🔄 5. 볼륨과 Bind Mount 차이점

| 항목 | Volume | Bind Mount |
|------|--------|------------|
| 정의 | Docker가 관리하는 저장소 | 호스트 디렉토리 직접 연결 |
| 생성 | Docker 명령어로 생성 | 직접 경로 지정 |
| 이식성 | 이식성이 높음 (경로 없음) | 호스트 의존성 있음 |
| 권한 관리 | 보안/권한 분리 가능 | 호스트 권한 영향 받음 |
| 권장 대상 | 데이터베이스, 장기 저장소 | 개발 중 코드 실시간 반영 |

### 예시

```bash
# Named Volume
-v myvolume:/app/data

# Bind Mount
-v $(pwd)/data:/app/data
```

---

## 💡 6. 실전 예제: MySQL 데이터 유지

```bash
# 1. 볼륨 생성
docker volume create mysql-data

# 2. MySQL 컨테이너 실행 (볼륨 연결)
docker run -d \
  --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=1234 \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0
```

### 👉 결과
- MySQL의 DB 파일이 `/var/lib/mysql`에 저장됨
- 컨테이너를 삭제해도 `mysql-data` 볼륨은 **데이터를 유지**

---

## 🧹 7. 볼륨 정리

```bash
# 중지된 컨테이너 + 안 쓰는 볼륨 자동 삭제
docker system prune --volumes
```

- 디스크 낭비 방지에 유용
- **주의**: 실제 데이터가 지워질 수 있으므로 꼭 확인 후 실행

---

## 📌 8. 도커 컴포즈에서 볼륨 사용

```yaml
version: '3.9'
services:
  db:
    image: mysql:8
    volumes:
      - dbdata:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=1234

volumes:
  dbdata:
```

### 🔍 결과
- `dbdata`라는 이름의 **Named Volume**이 자동 생성됨
- `docker-compose down -v` 명령어로 컨테이너와 볼륨 같이 제거 가능

---

## 📋 9. 베스트 프랙티스

| 항목 | 권장 사항 |
|------|------------|
| ❌ 데이터 저장에 컨테이너 내부 경로만 쓰지 말 것 | 휘발성 문제 발생 |
| ✅ Volume 사용 | 데이터 영속성 확보 |
| 📁 민감한 데이터 | 별도의 Volume 관리 권장 |
| 🧪 개발 시 Bind Mount | 코드 핫 리로드 등 개발 편의성 |
| 🔒 운영 시 Volume 고정 | 보안/일관성 유지

---

## 🧪 10. 상태 확인 & 디버깅

```bash
# 볼륨이 실제 어디에 저장되는지 보기
docker volume inspect myvolume

# 볼륨 내부 파일 확인
docker run --rm -v myvolume:/data alpine ls /data
```

---

## ✅ 결론 정리

| 목적 | 방법 |
|------|------|
| 데이터 보존 | Named Volume 사용 |
| 실시간 코드 반영 | Bind Mount 사용 |
| 다중 컨테이너 간 공유 | 같은 Volume 사용 |
| 불필요한 데이터 정리 | `docker volume rm`, `system prune` |

---

## 📚 참고 자료

- [Docker 공식 볼륨 문서](https://docs.docker.com/storage/volumes/)
- [Docker Compose - Named Volumes](https://docs.docker.com/compose/compose-file/#volumes)