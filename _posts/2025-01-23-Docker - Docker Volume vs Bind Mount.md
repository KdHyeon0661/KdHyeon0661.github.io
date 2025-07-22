---
layout: post
title: Docker - Docker Volume vs Bind Mount
date: 2025-01-23 20:20:23 +0900
category: Docker
---
# 🔄 Docker Volume vs Bind Mount: 완벽 비교

---

## 📦 1. 공통점

- 둘 다 **컨테이너 외부에 저장되는 저장소**
- 컨테이너 재시작, 삭제 후에도 데이터 유지 가능
- `docker run -v` 또는 `--mount`로 설정 가능
- 컨테이너 간 데이터 공유에 사용 가능

---

## 📁 2. 정의 및 차이점 요약

| 항목 | Volume | Bind Mount |
|------|--------|------------|
| 저장 위치 | Docker가 관리하는 디렉토리 (`/var/lib/docker/volumes`) | 사용자가 지정한 호스트의 경로 |
| 생성 방식 | Docker가 자동 생성, 관리 | 사용자가 직접 지정 |
| 관리 용이성 | CLI/API로 쉽게 관리 가능 | Docker는 경로만 전달, 관리 책임 없음 |
| 보안 | 컨테이너 격리 유지 | 호스트 파일 직접 접근 가능 (보안 취약 가능성) |
| 사용 용도 | DB, 장기 데이터 저장, 운영 환경 | 개발 환경 (코드 반영, 핫리로드) |
| 백업 | `docker volume inspect`, 백업 도구 사용 쉬움 | 수동으로 디렉토리 백업 필요 |
| 성능 | 일반적으로 더 좋음 (Docker 최적화) | 호스트 I/O에 따라 달라짐 |

---

## 🔧 3. 사용 예시

### ✅ Volume 사용 예시

```bash
# 1. 볼륨 생성
docker volume create myvolume

# 2. 컨테이너 실행
docker run -d -v myvolume:/app/data nginx
```

### ✅ Bind Mount 사용 예시

```bash
# 현재 디렉토리의 data 폴더를 컨테이너 내부에 마운트
docker run -d -v $(pwd)/data:/app/data nginx
```

- 윈도우는 `$(pwd)` 대신 `"%cd%"` 사용 가능
- 바인드 마운트는 호스트의 실제 파일 시스템을 그대로 사용

---

## ⚙️ 4. --mount 구문 비교

```bash
# Volume
docker run --mount type=volume,source=myvolume,target=/app/data nginx

# Bind Mount
docker run --mount type=bind,source="$(pwd)"/data,target=/app/data nginx
```

- `--mount`는 `-v`보다 명시적이며 구조가 명확함 (추천 방식)

---

## 📊 5. 장단점 비교

### 🔵 Volume

**장점**
- Docker가 위치/권한/퍼미션 관리
- Docker CLI로 상태 확인 (`docker volume ls`, `inspect`)
- 호스트에 종속되지 않음 → 이식성 높음
- `volume driver` 사용 가능 (NFS, cloud 등)

**단점**
- 초기 디버깅 불편 (호스트에 어디 있는지 불명확)
- 파일 동기화나 핫리로드에는 적합하지 않음

---

### 🟠 Bind Mount

**장점**
- 개발 중 소스코드 실시간 반영 가능 (Hot reload)
- 파일 편집이 즉시 컨테이너에 반영됨
- VSCode, IDE 등과 함께 사용하기 편함

**단점**
- 경로 설정이 불편 (운영체제별 다름)
- 보안 취약 가능 (루트 경로 등 마운트 시)
- 도커에서 직접 관리하지 않음 (CLI로 조회 불가)
- 호스트 파일 시스템 구조에 종속됨 → 이식성 낮음

---

## 📋 6. 언제 무엇을 써야 할까?

| 상황 | 권장 방식 | 이유 |
|------|-----------|------|
| 🚀 프로덕션 환경 | **Volume** | 안정성, 보안, 관리 용이성 |
| 🧪 개발 환경 (코드 핫리로드 등) | **Bind Mount** | 파일 실시간 동기화 |
| ☁️ 스토리지 통합 (NFS, S3 등) | **Volume + Plugin** | 외부 볼륨 드라이버 사용 |
| 🔁 여러 컨테이너 간 공유 디렉토리 | **Volume** | 이름으로 쉽게 연결 가능 |

---

## 🧼 7. 관리 명령어 요약

```bash
# 볼륨 확인 및 관리
docker volume ls
docker volume inspect <name>
docker volume rm <name>

# 바인드 마운트는 직접 디렉토리 확인
ls /path/to/your/bindmount
```

---

## 🧩 8. 실전 활용 예: MySQL

### 🔵 Volume 방식

```bash
docker run -d \
  -v mysql-volume:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=1234 \
  mysql:8
```

→ 최적화, 관리가 용이하고 프로덕션에 적합

### 🟠 Bind Mount 방식

```bash
docker run -d \
  -v $(pwd)/mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=1234 \
  mysql:8
```

→ 개발 중 데이터 구조 확인이나 직접 접근에 유용

---

## 📚 참고 자료

- [Docker 공식 Volume 문서](https://docs.docker.com/storage/volumes/)
- [Bind Mounts 공식 문서](https://docs.docker.com/storage/bind-mounts/)
- [Docker Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)