---
layout: post
title: Docker - Volumes, 데이터 영속성
date: 2025-01-23 19:20:23 +0900
category: Docker
---
# Docker Volumes: 데이터 영속성 유지하기

## 왜 볼륨이 필요한가?

Docker 컨테이너의 파일 시스템은 기본적으로 휘발성입니다. 컨테이너가 삭제되면 그 안에 저장된 모든 데이터도 함께 사라집니다. 데이터베이스 파일, 애플리케이션 로그, 업로드된 파일 등 장기적으로 보존해야 하는 데이터는 컨테이너의 수명 주기와 분리되어 관리되어야 합니다. Docker 볼륨은 바로 이러한 데이터 영속성 문제를 해결하는 핵심 메커니즘입니다.

---

## Docker 볼륨의 기본 개념

Docker 볼륨은 Docker 엔진이 관리하는 호스트 시스템의 디렉터리입니다. 일반적으로 `/var/lib/docker/volumes/<볼륨이름>/_data` 경로에 위치하며, 컨테이너 내부에서는 특정 마운트 지점으로 연결됩니다. 볼륨에 저장된 데이터는 컨테이너가 삭제되어도 유지되며, 여러 컨테이너 간에 공유될 수도 있습니다.

구조 예시(개념도):

```
컨테이너
  └─ /app/data (마운트 지점; 컨테이너 내부 경로)
        ⇅
      [volume: myvolume]
  └─ /var/lib/docker/volumes/myvolume/_data (호스트 실제 경로)
```

이 구조에서 컨테이너의 `/app/data` 디렉터리에 쓰기 작업이 발생하면, 실제 데이터는 호스트 시스템의 볼륨 디렉터리에 저장됩니다.

---

## 볼륨 종류 비교

Docker는 세 가지 주요 스토리지 마운트 방식을 제공합니다. 각 방식의 특징과 적절한 사용 사례를 이해하는 것이 중요합니다.

### Named Volume (명명된 볼륨)
- **관리 방식**: Docker가 이름으로 생성하고 관리
- **저장 위치**: `/var/lib/docker/volumes/<이름>/_data`
- **이식성**: 높음 (경로에 독립적)
- **주 사용처**: 운영 환경의 데이터베이스, 로그, 공유 데이터

### Anonymous Volume (익명 볼륨)
- **관리 방식**: 컨테이너 실행 시 자동 생성 (이름 없음)
- **저장 위치**: Named Volume과 동일
- **이식성**: 낮음 (관리 및 청소 어려움)
- **주 사용처**: 일회성 테스트 (일반적으로 피해야 함)

### Bind Mount (바인드 마운트)
- **관리 방식**: 사용자가 직접 호스트 경로 지정
- **저장 위치**: 임의의 호스트 디렉터리
- **이식성**: 낮음 (OS 및 경로에 의존적)
- **주 사용처**: 개발 환경의 핫 리로드, 로컬 파일 검사

**핵심 원칙**:
- 운영 환경 및 장기 데이터 저장 → Named Volume 사용
- 개발 중 실시간 코드 반영 → Bind Mount 사용
- Anonymous Volume은 의도치 않은 데이터 잔여물을 생성할 수 있으므로 지양

---

## 기본 사용 방법

### 볼륨 생성 및 관리

```bash
# 볼륨 생성
docker volume create myvolume

# 볼륨 목록 조회
docker volume ls

# 볼륨 상세 정보 확인
docker volume inspect myvolume

# 볼륨 삭제
docker volume rm myvolume
```

### 컨테이너에 볼륨 마운트

권장되는 명시적 구문 (`--mount`):
```bash
docker run -d \
  --name mycontainer \
  --mount type=volume,source=myvolume,target=/app/data \
  ubuntu sleep infinity
```

간편한 구문 (`-v`):
```bash
docker run -d --name mycontainer -v myvolume:/app/data ubuntu sleep infinity
```

### 바인드 마운트 사용 (개발 환경)

```bash
# 개발 시 실시간 코드 반영을 위한 바인드 마운트
docker run -d \
  --name devweb \
  --mount type=bind,source="$(pwd)"/public,target=/usr/share/nginx/html,readonly \
  -p 8080:80 nginx
```

---

## 실전 운영 예제

### MySQL 데이터베이스

```bash
docker volume create mysql-data
docker run -d --name mysql \
  -e MYSQL_ROOT_PASSWORD=1234 \
  --mount type=volume,source=mysql-data,target=/var/lib/mysql \
  -p 3306:3306 mysql:8.0
```

### PostgreSQL 데이터베이스

```bash
docker volume create pgdata
docker run -d --name pg \
  -e POSTGRES_PASSWORD=1234 \
  --mount type=volume,source=pgdata,target=/var/lib/postgresql/data \
  -p 5432:5432 postgres:16
```

### Redis 데이터 저장

```bash
docker volume create redisdata
docker run -d --name redis \
  --mount type=volume,source=redisdata,target=/data \
  -p 6379:6379 redis:7 --appendonly yes
```

**운영 팁**: 데이터베이스의 데이터 디렉터리는 여러 컨테이너가 동시에 쓰기 작업을 해서는 안 됩니다. 백업이나 점검 작업은 읽기 전용 모드의 별도 컨테이너를 사용하여 안전하게 수행하세요.

---

## 권한 및 보안 고려사항

### 파일 권한과 사용자 ID

컨테이너 내부의 파일 소유권은 컨테이너의 UID/GID를 따릅니다. 보안을 강화하기 위해 비루트 사용자를 사용하는 것이 좋습니다.

```bash
docker run -d --name app \
  --user 1000:1000 \
  --mount type=volume,source=appdata,target=/app/data \
  myimg:latest
```

### 읽기 전용 마운트

설정 파일이나 정적 자산은 읽기 전용으로 마운트하는 것이 안전합니다.

```bash
docker run -d \
  --mount type=volume,source=cfg,target=/app/cfg,readonly \
  myimg
```

### SELinux 환경 (RHEL/CentOS/Fedora)

SELinux가 활성화된 환경에서는 적절한 보안 컨텍스트 라벨을 지정해야 합니다.

```bash
# -v 구문에서 라벨 지정
docker run -d -v /host/data:/app/data:Z myimg

# --mount 구문에서 라벨 지정
docker run -d --mount type=bind,source=/host/data,target=/app/data,z myimg
```

### 플랫폼별 특성

- **Windows**: PowerShell에서는 `${PWD}`, Command Prompt에서는 `%cd%` 사용
- **macOS**: 바인드 마운트는 가상화 계층을 거치므로 성능 저하가 있을 수 있음
- **WSL2**: `/mnt/c/` 경로보다 WSL2 내부 경로(예: `~/project`) 사용 시 I/O 성능이 더 좋음

---

## 백업 및 복구 절차

### 볼륨 데이터 백업

```bash
# myvolume의 데이터를 현재 디렉터리에 백업
docker run --rm \
  -v myvolume:/src \
  -v "$(pwd)":/backup \
  alpine sh -c "cd /src && tar czf /backup/myvolume-$(date +%F).tgz ."
```

### 백업 데이터 복구

```bash
# 백업 파일에서 myvolume으로 데이터 복구
docker run --rm \
  -v myvolume:/dest \
  -v "$(pwd)":/backup \
  alpine sh -c "cd /dest && tar xzf /backup/myvolume-2025-11-06.tgz"
```

### 데이터 검증

```bash
docker run --rm -v myvolume:/data alpine ls -l /data
```

**운영 팁**: 데이터베이스 볼륨을 백업할 때는 먼저 일관성 있는 상태를 확보하세요. MySQL의 경우 `FLUSH TABLES WITH READ LOCK`, PostgreSQL의 경우 `pg_dump`를 사용하는 등 데이터베이스별 적절한 방법을 적용하세요.

---

## Docker Compose에서의 볼륨 사용

### 운영 환경 구성

```yaml
version: "3.9"
services:
  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=1234
    volumes:
      - dbdata:/var/lib/mysql
    ports:
      - "3306:3306"

volumes:
  dbdata:
```

### 개발 환경 구성

```yaml
version: "3.9"
services:
  api:
    build: ./api
    environment:
      - APP_ENV=dev
    volumes:
      - ./api/src:/app/src        # 바인드 마운트: 핫 리로드
      - apilogs:/var/log/myapi    # 명명된 볼륨: 로그 저장
    ports:
      - "5000:5000"

volumes:
  apilogs:
```

**명령어**:
```bash
docker compose up -d              # 서비스 시작
docker compose down               # 컨테이너만 제거
docker compose down -v            # 컨테이너와 볼륨 모두 제거 (주의)
```

---

## 정리 및 유지보수

### 사용하지 않는 자원 정리

```bash
# 사용하지 않는 볼륨 정리 (주의: 데이터 영구 삭제)
docker system prune --volumes

# 특정 볼륨 삭제 (사용 중인 컨테이너가 있으면 실패)
docker volume rm <볼륨이름>
```

**안전 팁**: 운영 환경에서 `prune --volumes` 명령어는 반드시 백업을 확인한 후 신중하게 사용하세요. 명명된 볼륨을 표준으로 사용하면 정리, 이전, 백업 작업을 일관되게 관리할 수 있습니다.

---

## 성능 및 운영 모범 사례

1. **운영 데이터 저장**: Named Volume을 표준으로 사용하세요.
2. **개발 효율성**: 코드 변경의 실시간 반영이 필요하면 Bind Mount를 사용하세요.
3. **보안 강화**: 설정 파일은 읽기 전용(`:ro`)으로 마운트하고, 비루트 사용자로 컨테이너를 실행하세요.
4. **백업 전략**: 정기적인 백업 스케줄을 수립하고 체크섬 검증을 포함하세요.
5. **모니터링**: `docker system df` 명령어로 스토리지 사용량을 정기적으로 모니터링하세요.
6. **로그 관리**: 애플리케이션 로그는 별도의 볼륨에 저장하여 로그 회전과 보관을 용이하게 하세요.

---

## 문제 해결 가이드

| 증상 | 가능한 원인 | 해결 방법 |
|---|---|---|
| 권한 오류 (EACCES) | 컨테이너와 호스트의 UID/GID 불일치 | `--user` 옵션으로 사용자 맞추기, 파일 권한 조정 |
| SELinux 차단 | 적절한 보안 컨텍스트 라벨 없음 | `:Z`(전용) 또는 `:z`(공유) 라벨 적용 |
| macOS/Windows I/O 느림 | Docker Desktop 가상화 계층 오버헤드 | 볼륨 사용 또는 WSL2 내부 경로로 전환 |
| 데이터 손실 | 익명 볼륨 사용 또는 잘못된 마운트 | Named Volume으로 표준화, 마운트 설정 검증 |
| 데이터베이스 손상 | 여러 컨테이너의 동시 쓰기 접근 | 데이터 디렉터리는 단일 컨테이너만 접근 |

---

## 실습: 볼륨 vs 바인드 마운트 차이 체험

### 볼륨을 사용한 정적 웹사이트 호스팅

```bash
# 볼륨 생성 및 데이터 복사
docker volume create webdata
docker run --rm \
  -v webdata:/dst \
  -v "$(pwd)"/public:/src:ro \
  alpine sh -c "cp -r /src/* /dst/"

# 볼륨을 사용한 웹서버 실행
docker run -d --name webv \
  --mount type=volume,source=webdata,target=/usr/share/nginx/html,readonly \
  -p 8080:80 nginx

# 접속 테스트
curl -s localhost:8080 | head
```

### 바인드 마운트를 사용한 개발 환경

```bash
# 개발 파일 준비
mkdir -p ./public && echo "<h1>Hello Developer</h1>" > ./public/index.html

# 바인드 마운트로 웹서버 실행
docker run -d --name webb \
  --mount type=bind,source="$(pwd)"/public,target=/usr/share/nginx/html,readonly \
  -p 8081:80 nginx

# 파일 수정 후 즉시 반영 확인
echo "<h1>Updated Content</h1>" > ./public/index.html
```

---

## 주요 명령어 요약

```bash
# 볼륨 관리
docker volume create <이름>
docker volume ls
docker volume inspect <이름>
docker volume rm <이름>

# 볼륨 마운트 (권장 방식)
docker run --mount type=volume,source=myvol,target=/data myimg

# 바인드 마운트
docker run --mount type=bind,source="$(pwd)"/data,target=/data myimg

# 읽기 전용 마운트
docker run --mount type=volume,source=myvol,target=/data,readonly myimg

# 볼륨 내용 확인
docker run --rm -v myvol:/data alpine ls -l /data
```

---

## 결론

Docker 볼륨은 컨테이너 기반 애플리케이션에서 데이터 영속성을 보장하는 핵심 메커니즘입니다. Named Volume, Anonymous Volume, Bind Mount 각각의 특징을 이해하고 상황에 맞게 적절히 선택하는 것이 중요합니다.

운영 환경에서는 데이터 안전성과 이식성을 위해 Named Volume을 표준으로 사용하고, 개발 환경에서는 생산성 향상을 위해 Bind Mount를 활용하세요. 권한 관리, 보안 설정, 정기적인 백업은 데이터 무결성과 시스템 안정성을 보장하는 필수 요소입니다.

Docker 볼륨을 효과적으로 관리하려면 일관된 명명 규칙, 체계적인 백업 전략, 그리고 지속적인 모니터링이 필요합니다. 이러한 모범 사례들을 준수하면 컨테이너 환경에서도 안정적이고 신뢰할 수 있는 데이터 관리를 구현할 수 있습니다.