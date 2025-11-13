---
layout: post
title: Docker - Docker Volume vs Bind Mount
date: 2025-01-23 20:20:23 +0900
category: Docker
---
# Docker Volume vs Bind Mount: 완벽 비교

## 1. 공통점(요약 재정리)

- 컨테이너 외부에 데이터 저장(컨테이너 재생성/재시작 후에도 유지).
- 여러 컨테이너가 동일 데이터를 공유 가능.
- `-v` 또는 `--mount`로 연결 가능.
- 읽기/쓰기 모드와 사용자/권한 제어 가능(옵션 조합).

---

## 2. 정의 및 차이점(확장 요약)

| 항목 | Volume | Bind Mount |
|---|---|---|
| 저장 위치 | Docker 관리 디렉터리(보통 `/var/lib/docker/volumes/<name>/_data`) | 사용자가 지정한 호스트 경로(절대경로 권장) |
| 생성/관리 | Docker가 생성·수명관리(이름 기반) | 사용자가 경로를 직접 관리(도커는 경로 전달만) |
| 이식성 | 높음(호스트 경로 독립) | 낮음(호스트 경로 의존·OS별 경로 상이) |
| 보안/격리 | 비교적 안전(엔진이 경로 관리) | 호스트 파일시스템 직접 노출(주의 필요) |
| 운영 가시성 | `docker volume ls/inspect`로 상태 파악 용이 | Docker CLI로 목록/메타 조회 어려움(직접 파일시스템 확인) |
| 성능 | 일반적으로 안정적(엔진 최적화·드라이버 연동) | 호스트 IO 특성/데스크톱 가상화 계층에 따라 크게 변동 |
| 플러그인 | Volume Driver로 원격/NFS/클라우드 스토리지 연동 용이 | 일반적으로 로컬 FS에 한정(별도 FUSE/마운트 조합은 가능) |
| 주요 용도 | DB, 메시지 큐, 영구 데이터, 다중 컨테이너 공유 | 개발 중 코드 핫리로드, 로컬 파일 즉시 반영 |

---

## 3. 명령 구문 비교: `-v` vs `--mount` (권장: `--mount`)

### 3.1 `-v` (단축형)
```bash
# Volume
docker run -d -v myvol:/app/data nginx

# Bind Mount
docker run -d -v "$(pwd)"/data:/app/data nginx
```
- 축약되어 빠르지만, **의미가 응축**되어 가독성/오탈자 확인이 어려울 수 있음.

### 3.2 `--mount` (명시형, 추천)
```bash
# Volume
docker run -d \
  --mount type=volume,source=myvol,target=/app/data \
  nginx

# Bind Mount
docker run -d \
  --mount type=bind,source="$(pwd)"/data,target=/app/data \
  nginx
```
- `type/target/source`가 명시되어 **오류·혼동 감소**.
- 읽기 전용, 선택적 옵션(SELinux/유저권한 등) 표현도 명확.

---

## 4. OS/플랫폼별 경로 주의점

- **Linux**: 일반적으로 경로 일관성 높음. SELinux/AppArmor 정책 고려 필요.
- **Windows**: PowerShell/`cmd`별 경로 문법 차이.
  - `$(pwd)` 대신 `%cd%`(cmd) 또는 `${PWD}`(PowerShell) 사용.
  - 드라이브 공유(Docker Desktop) 설정 필요할 수 있음.
- **macOS**: Docker Desktop 가상화 계층을 거쳐 I/O. 바인드 마운트 성능이 떨어질 수 있음.
- **WSL2**:
  - `/mnt/c/...`(Windows 파일) ↔ WSL2 Linux 파일시스템 성능 차이 존재.
  - 개발 중 바인드 마운트는 **WSL2 내부 경로**를 추천(성능·안정성↑).

예시(Windows PowerShell):
```powershell
docker run -d -v ${PWD}/data:/app/data nginx
```

---

## 5. 권한/보안/SELinux 옵션

### 5.1 읽기 전용(Read-Only)
```bash
docker run -d \
  --mount type=bind,source="$(pwd)"/public,target=/usr/share/nginx/html,readonly \
  nginx
```

### 5.2 사용자 매핑
- 컨테이너 내부 파일 소유자/권한은 **컨테이너의 UID/GID**와 연계.
- 실행 시 `--user` 옵션 혹은 이미지 내 `USER` 지시문으로 비루트 실행 권장.

```bash
docker run -d --user 1000:1000 \
  --mount type=volume,source=myvol,target=/app/data \
  myimg
```

### 5.3 SELinux(Linux with SELinux)
- 컨텍스트 라벨 필요 시 `:Z`(전용 라벨) 또는 `:z`(공유 라벨) 사용(단축형 `-v` 구문).
```bash
docker run -d -v /host/data:/app/data:Z myimg
```
- `--mount`에서는 `,z` 또는 `,Z` 옵션을 `bind-propagation`와 별도로 지정:
```bash
docker run -d \
  --mount type=bind,source=/host/data,target=/app/data,bind-propagation=rprivate,z \
  myimg
```

---

## 6. 성능 관점: 언제 무엇이 더 빠른가?

- **Linux 네이티브**: Volume이 일반적으로 **일관성 있고 빠름**(엔진 최적화).
- **Docker Desktop(macOS/Windows)**: 바인드 마운트는 **가상화 계층 동기화**로 느려질 수 있음.
  - 코드 핫리로드가 필요하면 바인드 마운트는 불가피하지만, 빌드/런타임 열악해질 수 있음.
  - 대규모 파일 트리(수십만 파일)는 **Volume + `docker cp`** 또는 빌드 단계에 포함하는 방안 고려.

---

## 7. 데이터 일관성과 동시성

- 다중 컨테이너가 같은 디렉터리를 공유하면 **파일 잠금/일관성**에 주의.
- DB 엔진은 **하나의 데이터 디렉터리를 여러 컨테이너가 동시에 쓰기**는 권장하지 않음.
- **로그/업로드** 같은 공유는 Volume으로 이름 기반 공유가 편리.

---

## 8. 백업/복구/마이그레이션

### 8.1 Volume 백업(컨테이너 이용)
```bash
# 백업: myvol → tar
docker run --rm \
  -v myvol:/src \
  -v "$(pwd)":/backup \
  alpine sh -c "cd /src && tar czf /backup/myvol-backup.tgz ."

# 복구: tar → myvol
docker run --rm \
  -v myvol:/dest \
  -v "$(pwd)":/backup \
  alpine sh -c "cd /dest && tar xzf /backup/myvol-backup.tgz"
```

### 8.2 Bind Mount 백업
- 호스트에서 디렉터리를 그대로 압축/스냅샷.
- 자체 버전관리/스냅샷 정책 필요.

---

## 9. 운영 예: DB 스토리지

### 9.1 Volume(권장)
```bash
docker volume create pgdata
docker run -d --name pg \
  -e POSTGRES_PASSWORD=pass \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16
```
- **운영/스테이징**: Volume이 기본 선택(권한/속도/격리/관리성).

### 9.2 Bind Mount(개발·디버깅)
```bash
mkdir -p ./pgdata
docker run -d --name pg \
  -e POSTGRES_PASSWORD=pass \
  -v "$(pwd)"/pgdata:/var/lib/postgresql/data \
  postgres:16
```
- 파일을 직접 보고 싶을 때 유리(개발·분석).

---

## 10. 운영 예: 앱 코드/정적 자산

### 10.1 개발(핫리로드)
```bash
docker run -d \
  -v "$(pwd)"/src:/usr/src/app \
  -p 3000:3000 node:20 \
  bash -lc "cd /usr/src/app && npm ci && npm run dev"
```
- 코드 변경 즉시 반영(프레임워크의 watch 기능 전제).

### 10.2 운영(정적 파일)
- 빌드 산출물을 이미지에 포함하거나, Volume으로 사전 배포 후 `readonly`로 마운트.
```bash
docker run -d \
  --mount type=volume,source=webassets,target=/usr/share/nginx/html,readonly \
  -p 80:80 nginx
```

---

## 11. `docker-compose.yml` 예시(운영/개발 대비)

```yaml
version: "3.9"
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
    volumes:
      - webassets:/usr/share/nginx/html:ro
  api:
    build: ./api
    environment:
      - APP_ENV=prod
    volumes:
      - apilogs:/var/log/myapi

volumes:
  webassets:
  apilogs:
```

개발용(핫리로드):
```yaml
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
    volumes:
      - ./public:/usr/share/nginx/html:ro
  api:
    build: ./api
    environment:
      - APP_ENV=dev
    volumes:
      - ./api/src:/app/src
```

---

## 12. 접근 제어와 무결성

- **읽기 전용**(`:ro`/`readonly`)으로 공격 면 최소화.
- 컨테이너를 **비루트**로 실행(`USER`, `--user`) + 루트FS 읽기 전용(`--read-only`) + `tmpfs`로 임시 경로 제공:
```bash
docker run -d --read-only \
  --tmpfs /tmp --tmpfs /run \
  --cap-drop ALL --security-opt no-new-privileges \
  --user 65532:65532 \
  --mount type=volume,source=cfg,target=/app/cfg,readonly \
  myimg
```

---

## 13. 고급: Volume Driver로 외부 스토리지 연계

- NFS/GlusterFS/클라우드(예: EBS, Azure Disk, GCE PD) 등과 연계하는 **볼륨 드라이버**.
- 팀/다수 호스트 간 공유, 스냅샷/백업 자동화 가능.
- 도입 전 성능/락/일관성 정책 검토 필수.

---

## 14. tmpfs(메모리 기반) 마운트

- 민감 데이터/캐시/임시파일을 디스크에 남기지 않도록 메모리에 마운트:
```bash
docker run -d \
  --mount type=tmpfs,target=/run/tmp \
  myimg
```
- 재시작 시 휘발.

---

## 15. 파일 동기화/변경 감지(특히 데스크톱)

- macOS/Windows에서 바인드 마운트 시 변경 감지가 **느리거나 누락**될 수 있음.
  - 프레임워크 옵션(폴링 기반 watcher) 활성화
  - 파일 수가 매우 많으면, 개발 중에도 가능한 파일 수/깊이를 제한
  - 대규모 빌드는 컨테이너 내부에서 수행하거나 **Volume에 복사 후 작업**

---

## 16. 실전 시나리오별 추천

| 시나리오 | 추천 | 설명 |
|---|---|---|
| 운영 DB/메시지 큐 | **Volume** | 안정성/권한/드라이버/백업 수월 |
| 개발용 DB(스키마 확인) | Bind Mount | 파일 직접 접근·분석 유용 |
| FE 정적 사이트 운영 | 이미지 내 포함 or Volume:ro | 불변 배포, CDN/캐시와 연계 |
| BE 로그 | Volume | 호스트 경로 의존 낮추고 수집기(cont.)와 공유 |
| 로컬 개발(핫리로드) | Bind Mount | 코드 변경 즉시 반영(프레임워크 watch) |
| 팀 공유 데이터셋 | Volume Driver(NFS 등) | 중앙 스토리지로 일관성/백업 관리 |

---

## 17. 트러블슈팅 표

| 증상 | 원인 | 진단 | 해결 |
|---|---|---|---|
| 권한 에러(EACCES) | UID/GID 불일치, 읽기전용 | `docker exec ls -l`, `id` | `--user`/`USER`로 맞추거나 퍼미션 조정 |
| SELinux 차단 | 라벨 미설정 | 호스트 `audit.log`/denials | `:Z`/`:z` 또는 `--mount ...,z` 지정 |
| mac/Windows 느린 IO | 가상화 동기화 오버헤드 | 대규모 파일 접근 시 | Volume 사용/WSL2 내부 경로/빌드 내부화 |
| 변경 감지 안됨 | watcher/FS 이벤트 미지원 | 앱 로그/워처 설정 | 폴링 watcher 사용/간단 경로로 제한 |
| DB 손상 | 다중 컨테이너 동시 쓰기 | DB 로그 | DB 인스턴스는 단일 컨테이너가 전담 |
| 경로가 이상 | 상대경로/스페이스/권한 | `docker inspect`, `pwd` | 절대경로 사용·경로 인용·권한 점검 |
| 데이터 사라짐 | 익명 볼륨 생성/정리 | `docker volume ls` | **명명된 볼륨** 사용, 백업 정책 수립 |

---

## 18. 실습: 같은 구조를 Volume/Bind Mount로 번갈아 실행

### 18.1 앱(예: Nginx) + 데이터 폴더

#### Volume
```bash
docker volume create webdata
docker run -d --name webv \
  --mount type=volume,source=webdata,target=/usr/share/nginx/html \
  -p 8080:80 nginx

# 컨텐츠 주입(임시 컨테이너로 복사)
docker run --rm \
  --mount type=volume,source=webdata,target=/dst \
  -v "$(pwd)"/public:/src:ro \
  alpine sh -c "cp -r /src/* /dst/"

# 확인
curl -s localhost:8080 | head
```

#### Bind Mount
```bash
mkdir -p ./public
echo "<h1>Hello</h1>" > ./public/index.html

docker run -d --name webb \
  --mount type=bind,source="$(pwd)"/public,target=/usr/share/nginx/html,readonly \
  -p 8081:80 nginx

curl -s localhost:8081 | head
```

비교 포인트:
- 개발 중에는 `./public` 편집 → 즉시 반영(바인드).
- 운영에서는 `webdata` 볼륨 → 경로 독립, 읽기전용 제공 쉬움.

---

## 19. Compose로 관리 명령 간소화

```yaml
version: "3.9"
services:
  nginxv:
    image: nginx:alpine
    ports: ["8080:80"]
    volumes:
      - webdata:/usr/share/nginx/html:ro
  nginxb:
    image: nginx:alpine
    ports: ["8081:80"]
    volumes:
      - ./public:/usr/share/nginx/html:ro

volumes:
  webdata:
```

- `docker compose up -d` 한 번으로 두 방식 비교 운영.
- 배포 환경에서는 **named volume**이 이식성과 관리 용이성에서 유리.

---

## 20. 명령 요약

```bash
# 볼륨
docker volume ls
docker volume inspect <name>
docker volume create <name>
docker volume rm <name>

# 실행(권장: --mount)
docker run --mount type=volume,source=myvol,target=/data myimg
docker run --mount type=bind,source="$(pwd)"/data,target=/data myimg

# 읽기 전용
docker run --mount type=volume,source=myvol,target=/data,readonly myimg

# tmpfs
docker run --mount type=tmpfs,target=/run/tmp myimg
```

---

## 21. 수학적 직관(성능·이식성·운영성 간의 균형)

성능 지표를 \(P\), 이식성을 \(I\), 운영성(관리/보안/관찰)을 \(O\)라 하자.
개발(핫리로드)에서의 종합 효용 \(U_{\text{dev}}\), 운영에서의 종합 효용 \(U_{\text{prod}}\)의 직관은 다음과 같이 생각할 수 있다.
$$
U_{\text{dev}} \approx \alpha P_{\text{bind}} + \beta I_{\text{bind}} + \gamma O_{\text{bind}}, \quad
U_{\text{prod}} \approx \alpha' P_{\text{vol}} + \beta' I_{\text{vol}} + \gamma' O_{\text{vol}}
$$
일반적으로,
- 개발에서는 **바인드**가 \(P\)는 떨어질 수 있어도(데스크톱 가상화) \(O\)의 “즉시성”이 커서 \(U_{\text{dev}}\)가 큼.
- 운영에서는 **볼륨**의 \(I\), \(O\)(관리/보안/드라이버) 가중치가 높아 \(U_{\text{prod}}\)가 큼.

---

## 22. 결론 요약

- **운영/장기 데이터**: Volume(이름 기반·경로 독립·드라이버·백업 수월·보안 유리).
- **개발/핫리로드**: Bind Mount(코드 변경 즉시 반영·IDE 친화).
- **보안/권한**: 읽기 전용, 비루트, SELinux 라벨, read-only 루트FS + tmpfs.
- **성능**: 데스크톱 환경의 바인드 마운트는 느릴 수 있음. 대규모 파일·빌드는 Volume/컨테이너 내부 작업 고려.
- **운영성**: Compose/볼륨 드라이버/백업 절차로 **가시성·재해복구** 확보.

---

## 23. 체크리스트

- [ ] 운영 DB/영구 데이터는 **Volume** 사용
- [ ] 개발 핫리로드 디렉토리는 **Bind Mount**
- [ ] `--mount` 구문 사용(명시적)
- [ ] 읽기 전용(`readonly`/`:ro`) + 비루트(`--user`/`USER`)
- [ ] SELinux 환경에서 `:Z/:z` 또는 `--mount ...,z`
- [ ] mac/Windows 바인드 성능 고려(WSL2 내부 경로/Volume 전환)
- [ ] Volume **이름**으로 공유/백업/이전 용이하게 설계
- [ ] 데이터 백업/복구 스크립트 준비(Volume tar/압축)
- [ ] Compose로 서비스·스토리지 정의 버전관리
