---
layout: post
title: Docker - Volumes, 데이터 영속성
date: 2025-01-23 19:20:23 +0900
category: Docker
---
# Docker Volumes: 데이터 영속성 유지하기 — 개념→실전→보안→백업→운영 가이드

본 문서는 기존 글(필요성/정의/종류/기본 명령/Compose 예시)을 바탕으로 **실무 확장**을 덧붙였습니다.
- 볼륨의 **정확한 동작 원리**(Copy-on-Write와 무관한 독립 스토리지)
- **Named/Anonymous/Bind Mount** 비교와 선택 기준
- **권한/SELinux/Windows·macOS·WSL2** 특성
- **백업/복구/마이그레이션** 스크립트
- **성능/보안 베스트 프랙티스**, **트러블슈팅 표**, **Compose 패턴**

---

## 왜 Volume이 필요한가? (요약 보강)

컨테이너의 루트 파일시스템은 **휘발성**입니다. 컨테이너 제거 시 내부 변경사항이 사라집니다.
DB/업로드/로그/캐시 등 **수명 주기와 용량이 컨테이너와 분리**되어야 하는 데이터는 **Volume**으로 관리합니다.

직관식 표기:
$$
\text{Durability}(\text{container FS}) \approx 0,\quad
\text{Durability}(\text{volume}) \gg 0
$$

---

## Volume의 개념과 구조 (정교화)

- Volume은 Docker 엔진이 관리하는 **호스트 측 디렉터리(일반적으로 `/var/lib/docker/volumes/<name>/_data`)** 입니다.
- 컨테이너에서는 단순히 **마운트 지점**으로 보이고, 읽기/쓰기 동작은 호스트의 해당 디렉터리에 반영됩니다.
- 컨테이너를 재생성해도 **볼륨과 그 데이터는 독립적으로 생존**합니다.

구조 예시(개념도):

```
컨테이너
  └─ /app/data (마운트 지점; 컨테이너 내부 경로)
        ⇅
      [volume: myvolume]
  └─ /var/lib/docker/volumes/myvolume/_data (호스트 실제 경로)
```

---

## Volume/Bind Mount/Anonymous 비교(확장)

| 종류 | 생성/관리 | 저장 위치 | 이식성 | 권한/보안 | 주 사용처 |
|---|---|---|---|---|---|
| **Named Volume** | Docker가 이름으로 생성/관리 | `/var/lib/docker/volumes/<name>/_data` | 높음(경로 독립) | 엔진 관리로 비교적 안전 | 운영 DB/로그/공유 데이터 |
| **Anonymous Volume** | 컨테이너 실행 시 자동 생성(이름 없음) | 동일 | 낮음(청소 어려움) | 동일 | 일회성/시험용 |
| **Bind Mount** | 사용자가 호스트 경로 지정 | 임의의 호스트 디렉터리 | 낮음(OS/경로 의존) | 호스트 FS 직접 노출(주의) | 개발(핫리로드)/로컬 검사 |

핵심 요점:
- **운영/장기 데이터** → Named Volume
- **개발 핫리로드** → Bind Mount
- Anonymous는 **의도치 않은 잔여물**이 생기기 쉬우므로 지양

---

## 기본 사용법 (명령/옵션 의미 명확화)

### 볼륨 생성/조회/삭제

```bash
docker volume create myvolume
docker volume ls
docker volume inspect myvolume
docker volume rm myvolume
```

### 컨테이너에 볼륨 마운트

권장 구문: `--mount` (명시적)
```bash
docker run -d \
  --name mycontainer \
  --mount type=volume,source=myvolume,target=/app/data \
  ubuntu sleep infinity
```

단축 구문: `-v` (간편)
```bash
docker run -d --name mycontainer -v myvolume:/app/data ubuntu sleep infinity
```

### Bind Mount와 비교

```bash
# Bind Mount (개발 시 실시간 반영)

docker run -d \
  --name devweb \
  --mount type=bind,source="$(pwd)"/public,target=/usr/share/nginx/html,readonly \
  -p 8080:80 nginx
```

---

## 실전 예제: MySQL/Redis/PostgreSQL (운영 패턴)

### MySQL

```bash
docker volume create mysql-data
docker run -d --name mysql \
  -e MYSQL_ROOT_PASSWORD=1234 \
  --mount type=volume,source=mysql-data,target=/var/lib/mysql \
  -p 3306:3306 mysql:8.0
```

### PostgreSQL

```bash
docker volume create pgdata
docker run -d --name pg \
  -e POSTGRES_PASSWORD=1234 \
  --mount type=volume,source=pgdata,target=/var/lib/postgresql/data \
  -p 5432:5432 postgres:16
```

### Redis (AOF/로그 저장)

```bash
docker volume create redisdata
docker run -d --name redis \
  --mount type=volume,source=redisdata,target=/data \
  -p 6379:6379 redis:7 --appendonly yes
```

운영 팁:
- DB 데이터 디렉터리를 **복수 컨테이너가 동시에 쓰기**하지 않기
- 백업/점검은 **읽기 가능한 별도 컨테이너**로 안전하게 수행

---

## 권한/보안/SELinux/Windows·macOS·WSL2 특성

### 권한/UID·GID

- 컨테이너 내부에서의 파일 소유자는 컨테이너의 UID/GID에 따릅니다.
- 이미지에서 `USER` 지시문을 사용하거나, `docker run --user 1000:1000`으로 실행 권장.

```bash
docker run -d --name app \
  --user 1000:1000 \
  --mount type=volume,source=appdata,target=/app/data \
  myimg:latest
```

### 읽기 전용 마운트

```bash
docker run -d \
  --mount type=volume,source=cfg,target=/app/cfg,readonly \
  myimg
```

### SELinux (RHEL/CentOS/Fedora 등)

- 라벨 지정이 필요할 수 있습니다. `-v` 구문에서는 `:Z`(전용), `:z`(공유) 옵션 사용.

```bash
docker run -d \
  -v /host/data:/app/data:Z \
  myimg
```

`--mount`에서도 `,z` 또는 `,Z` 추가가 가능합니다.
```bash
docker run -d \
  --mount type=bind,source=/host/data,target=/app/data,z \
  myimg
```

### Windows/macOS/WSL2

- **Windows**: PowerShell은 `${PWD}`, `cmd`는 `%cd%`. 드라이브 공유 설정 필요할 수 있음.
- **macOS**: 바인드 마운트는 가상화 계층으로 느려질 수 있음 → 볼륨 활용/이미지 내 포함 고려.
- **WSL2**: `/mnt/c/...` 경로보다 **WSL2 내부 경로**(예: `~/proj`)가 I/O 성능과 안정성에서 유리.

---

## 백업/복구/마이그레이션 스크립트 (안전 절차)

### Volume → tar 백업

```bash
# myvolume의 내용물을 현재 디렉터리에 백업

docker run --rm \
  -v myvolume:/src \
  -v "$(pwd)":/backup \
  alpine sh -c "cd /src && tar czf /backup/myvolume-$(date +%F).tgz ."
```

### tar → Volume 복구

```bash
docker run --rm \
  -v myvolume:/dest \
  -v "$(pwd)":/backup \
  alpine sh -c "cd /dest && tar xzf /backup/myvolume-2025-11-06.tgz"
```

### 데이터 검증

```bash
docker run --rm -v myvolume:/data alpine ls -l /data
```

운영 팁:
- 백업은 **일관 시점** 확보(예: DB `FLUSH TABLES WITH READ LOCK`, `pg_dump` 등) 후 수행.
- 암호화/무결성 체크섬(SHA256)을 함께 관리.

---

## Compose에서 Volume 사용 (운영/개발 대비)

### 운영(명명된 볼륨)

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

### 개발(핫리로드 + 로그는 볼륨)

```yaml
version: "3.9"
services:
  api:
    build: ./api
    environment:
      - APP_ENV=dev
    volumes:
      - ./api/src:/app/src
      - apilogs:/var/log/myapi
    ports:
      - "5000:5000"

volumes:
  apilogs:
```

명령:
```bash
docker compose up -d
docker compose down            # 컨테이너만 제거
docker compose down -v         # 컨테이너 + 볼륨 제거(주의)
```

---

## 정리/청소 명령과 주의점

```bash
# 사용 안하는 데이터 정리(주의: 실제 데이터 삭제됨)

docker system prune --volumes

# 개별 볼륨 삭제 (컨테이너가 사용 중이면 실패)

docker volume rm <name>
```

안전 팁:
- 운영 환경에서 `prune --volumes`는 **백업 후** 제한적으로 사용.
- **명명된 볼륨**을 표준으로 하여 정리/이전/백업을 일관 처리.

---

## 성능/운영/보안 베스트 프랙티스

| 항목 | 권장 사항 |
|---|---|
| 운영 데이터 저장소 | Named Volume 사용(이름 기반, 경로 독립) |
| 개발 핫리로드 | Bind Mount(단, 대규모 파일은 Volume로 대체/내부 빌드) |
| 읽기 전용 | 설정/정적 자산은 `:ro` 또는 `readonly` |
| 권한 | 컨테이너 `USER` 비루트, 필요 시 `--user UID:GID` |
| SELinux | `:Z/:z` 또는 `--mount ...,z` 적용 |
| 백업 | tar 압축 + 체크섬/암호화, 정기 스냅샷 |
| 마이그레이션 | `volume -> tar -> volume` 경로 표준화 |
| 모니터링 | `docker system df`, `docker volume inspect`, 스토리지 사용량 추적 |
| 로그 분리 | 애플리케이션 로그 디렉터리를 별도 Volume로 분리(회전/보관 용이) |

---

## 트러블슈팅 표

| 증상 | 가능 원인 | 진단 | 해결 |
|---|---|---|---|
| 권한 에러(EACCES) | UID/GID 불일치 | `docker exec id`, `ls -l` | `--user`/`USER` 맞추기, 퍼미션 조정 |
| SELinux 차단 | 라벨 미설정 | `journalctl -t setroubleshoot` 등 | `:Z/:z` 라벨 부여 |
| mac/Windows 느린 I/O | Desktop 가상화 동기화 | 대용량 파일 접근 시 | Volume로 이전/WSL2 내부 경로 사용 |
| 데이터 사라짐 | Anonymous/익명 볼륨 사용 | `docker volume ls` | Named Volume로 표준화 |
| DB 손상 | 복수 컨테이너 동시 쓰기 | DB 로그 | 데이터 디렉터리 단일 컨테이너 전담 |
| 경로 오타 | 상대/공백/권한 | `docker inspect` `Mounts` | 절대경로/따옴표/권한 점검 |

---

## 실습: Volume vs Bind Mount 성격 차이 체감

### Volume로 정적 사이트 제공

```bash
docker volume create webdata
docker run --rm \
  -v webdata:/dst \
  -v "$(pwd)"/public:/src:ro \
  alpine sh -c "cp -r /src/* /dst/"

docker run -d --name webv \
  --mount type=volume,source=webdata,target=/usr/share/nginx/html,readonly \
  -p 8080:80 nginx

curl -s localhost:8080 | head
```

### Bind Mount로 개발 핫리로드

```bash
mkdir -p ./public && echo "<h1>Hello</h1>" > ./public/index.html
docker run -d --name webb \
  --mount type=bind,source="$(pwd)"/public,target=/usr/share/nginx/html,readonly \
  -p 8081:80 nginx
# public/index.html 수정 → 즉시 반영 확인

```

---

## 수학적 직관(용량/성능 균형)

볼륨/바인드 선택에 따른 “체감 효용” \(U\)를 성능 \(P\), 이식성 \(I\), 운영성 \(O\)의 가중 합으로 보면:
$$
U \approx \alpha P + \beta I + \gamma O
$$
- 개발: 바인드의 \(O\)(즉시성)가 커서 \(U_{\text{dev}}\) ↑
- 운영: 볼륨의 \(I/O\)(이식성·운영성)가 커서 \(U_{\text{prod}}\) ↑

---

## 최종 체크리스트

- [ ] 운영 DB/영구 데이터는 **Named Volume**
- [ ] 개발 핫리로드 디렉터리는 **Bind Mount**
- [ ] `--mount` 구문으로 명시적 정의
- [ ] `:ro`/`readonly` + 비루트 `USER` + 필요 시 `--read-only` 루트FS
- [ ] SELinux 환경 라벨(`:Z/:z`) 설정
- [ ] 백업 스크립트 확보(정기 수행/체크섬)
- [ ] `docker system df`로 스토리지 사용량 점검
- [ ] Compose 파일로 스토리지 정의를 **버전 관리**
- [ ] 익명 볼륨 남용 금지 → 이름 붙이기
- [ ] DB 데이터 디렉터리는 **단일 컨테이너 전담**

---

## 참고 명령 요약

```bash
# 생성/조회/삭제

docker volume create <name>
docker volume ls
docker volume inspect <name>
docker volume rm <name>

# 실행(권장 --mount)

docker run --mount type=volume,source=myvol,target=/data myimg
docker run --mount type=bind,source="$(pwd)"/data,target=/data myimg

# 읽기 전용

docker run --mount type=volume,source=myvol,target=/data,readonly myimg

# 볼륨 내용 확인(임시 컨테이너)

docker run --rm -v myvol:/data alpine ls -l /data

# 정리(주의)

docker system prune --volumes
```

---

## 참고

- Docker 공식 문서의 Storage(Volumes/Bind Mounts), Compose 스펙을 함께 확인하면 운영 정책 설계에 유익합니다.
