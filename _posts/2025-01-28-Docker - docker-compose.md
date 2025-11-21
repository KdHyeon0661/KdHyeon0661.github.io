---
layout: post
title: Docker - exec vs attach
date: 2025-01-28 21:20:23 +0900
category: Docker
---
# `docker-compose.yml` 문법

## 실행/검증 기본 명령(요약)

```bash
# Compose V2: 'docker compose'  ← 권장

docker compose up -d           # 백그라운드 실행
docker compose down            # 중지 + 네트워크/컨테이너 제거
docker compose ps              # 상태
docker compose logs -f app     # 로그 스트리밍
docker compose exec app sh     # 셸
docker compose restart app     # 재시작
docker compose stop app        # 정지

# 정합성/머지 결과 확인

docker compose config          # 병합·확장·env 적용된 최종 YAML 출력

# — 기능 유사

```

---

## YAML 기본 골격: services / networks / volumes / (secrets, configs)

```yaml
version: "3.9"        # Compose 파일 포맷. 최신 Compose(V2)는 version 생략도 허용.
services:
  web:                # 서비스(컨테이너) 정의
    image: nginx:alpine
    ports: ["8080:80"]
    networks: ["frontnet"]
    depends_on: ["api"]

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
      target: runtime               # 멀티스테이지 타겟
      args:
        APP_ENV: production
    environment:
      DB_HOST: db
      DB_USER: app
      DB_PASS_FILE: /run/secrets/dbpass
    secrets: ["dbpass"]
    networks: ["backnet"]

  db:
    image: postgres:15
    volumes:
      - dbdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/dbpass
    secrets: ["dbpass"]
    networks: ["backnet"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres || exit 1"]
      interval: 5s
      timeout: 2s
      retries: 12

networks:
  frontnet:
    driver: bridge
  backnet:
    driver: bridge

volumes:
  dbdata:

secrets:
  dbpass:
    file: ./secrets/db.password
```

**핵심 포인트**
- `services` = 컨테이너 집합. 이름이 곧 DNS 호스트네임(같은 네트워크에서).
- **네트워크를 명시적으로 분리**해 외부/내부 경계를 강제.
- **secrets**/**configs**(아래 확장절 참조)로 민감정보·환경설정 분리.

---

## 필수 키워드 정리(심화)

### `image` / `build`

```yaml
services:
  app:
    image: myorg/app:1.2.3       # 레지스트리에서 풀
# 또는

  app:
    build:
      context: ./app
      dockerfile: Dockerfile.prod
      target: runtime            # 멀티 스테이지 빌드 타겟
      args:
        NODE_ENV: production
      cache_from:
        - type=registry,ref=myorg/app:buildcache
      labels:
        org.opencontainers.image.source: "https://github.com/myorg/app"
```
- **이미지 vs 빌드**: 빌드가 있으면 `docker compose build` 로 이미지 생성 후 올림.
- `target`으로 멀티 스테이지의 특정 스테이지만 채택 → **경량 런타임** 확보.

### `ports`(포트 매핑)

```yaml
ports:
  - "127.0.0.1:8080:80"  # 로컬에만 바인딩
  - "8443:443/tcp"       # 프로토콜 지정
  - "8081:8081/udp"
```
- **외부 공개 최소화**: 필요 포트만 바인딩. 내부 통신은 네트워크 이름 사용.

### `environment` / `env_file` / 환경변수 치환

```yaml
environment:
  REDIS_HOST: redis
  FEATURE_FLAG: "${FEATURE_FLAG:-off}"
env_file:
  - .env
```
- `${VAR}` 치환 가능. 기본값은 `${VAR:-default}`.
- *민감정보는 `secrets`사용 권장.*

### `volumes`(마운트)

```yaml
# 서비스 측

volumes:
  - "./logs:/var/log/app:rw"     # bind mount (개발용)
  - "namedcache:/cache:rw"       # named volume (운영/공유)
  - "tmpfs:/tmp"                 # tmpfs(메모리) 볼륨

# 루트의 정의

volumes:
  namedcache:
  tmpfs:
    driver: local
```
- 개발은 **bind mount**로 핫리로드, 운영은 **named volume**로 이식성/관리성.

### `depends_on`(기동 순서)

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy   # Compose V2에서 지원(healthcheck 필요)
```
- 단순 문자열 배열도 가능하지만, **건강상태 기반**이 실무적.

### `healthcheck`

```yaml
healthcheck:
  test: ["CMD", "curl", "-fsS", "http://localhost/healthz"]
  interval: 10s
  timeout: 2s
  retries: 6
  start_period: 20s
```

### `command` / `entrypoint`

```yaml
command: ["gunicorn", "-w", "4", "app:app"]
entrypoint: ["/docker-entrypoint.sh"]
```
- `entrypoint`는 “고정 진입점”, `command`는 인자/기본명령.

### `restart`

```yaml
restart: "unless-stopped"   # always / on-failure / unless-stopped
```

---

## 네트워크 심화: 이름 통신·멀티 네트워크·외부 네트워크

### 이름 통신

- 같은 네트워크에서는 `db:5432`처럼 **서비스명**으로 접근.
- 내장 DNS(127.0.0.11)가 서비스 이름을 IP로 해석.

### 멀티 네트워크로 노출 통제

```yaml
services:
  fe:
    image: nginx:alpine
    ports: ["8080:80"]
    networks: [frontnet, backnet]  # 프런트는 외부·백엔드는 내부
  api:
    build: ./api
    networks: [backnet]
networks:
  frontnet: {}
  backnet: {}
```

### 네트워크 사용

```yaml
networks:
  corpnet:
    external: true
```

---

## 보안/리소스/런타임 고급 옵션

```yaml
services:
  app:
    image: myorg/app:1.0
    user: "65532:65532"             # 비루트 UID/GID
    read_only: true                 # 루트FS 읽기전용
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    ulimits:
      nofile: 65536
    sysctls:
      net.core.somaxconn: "4096"
    stop_signal: SIGTERM
    stop_grace_period: 20s
```

**요점**
- 컨테이너를 **비루트**로, 루트FS는 **read-only**, 필요한 **tmpfs**만.
- **capabilities 최소화**(cap_drop), **no-new-privileges**로 권한 상승 차단.

---

## Compose로 “빌드 최적화 + 런타임 경량화”(멀티 스테이지)

### Node.js 프런트엔드(정적 배포)

```yaml
services:
  web:
    build:
      context: ./frontend
      target: builder         # 1단계: 빌더
    image: myorg/frontend:build
  fe:
    image: nginx:alpine
    depends_on: [web]
    volumes:
      - type: bind
        source: ./frontend/dist
        target: /usr/share/nginx/html
    ports: ["8080:80"]
```

**대안**: `COPY --from=builder` 방식으로 최종 런타임 이미지에 **정적 산출물만** 패키징(더 권장).

---

## Secrets / Configs — 민감정보·설정 분리

### secrets

```yaml
services:
  db:
    image: mysql:8
    secrets: [dbpass]
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/dbpass

secrets:
  dbpass:
    file: ./secrets/db.password
```
- 컨테이너에서 `/run/secrets/<name>`로 **파일 형태** 접근(환경변수보다 안전).

### configs (비밀은 아니지만 변경 가능한 설정)

```yaml
services:
  fe:
    image: nginx:alpine
    configs:
      - source: nginx_conf
        target: /etc/nginx/conf.d/default.conf

configs:
  nginx_conf:
    file: ./ops/nginx.default.conf
```

---

## Profiles — 상황별 부분 기동

```yaml
services:
  grafana:
    image: grafana/grafana
    profiles: ["monitoring"]

# 실행 시 지정

docker compose --profile monitoring up -d
```

- 로컬 디버깅만 필요한 서비스, 배치성 잡 등 **상황별 on/off**.

---

## Anchors & Extension Fields — 중복 줄이기

```yaml
x-base: &base
  restart: unless-stopped
  networks: [backnet]
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"

services:
  api:
    <<: *base
    image: myorg/api:1.0
  worker:
    <<: *base
    image: myorg/worker:1.0

networks:
  backnet: {}
```

- YAML 앵커로 **공통 템플릿**을 재사용.

---

## Compose 파일 오버레이(환경별 구성)

```
compose.yaml
compose.prod.yaml
compose.dev.yaml
```

```bash
# prod + 공통 머지

docker compose -f compose.yaml -f compose.prod.yaml up -d

# dev + 공통

docker compose -f compose.yaml -f compose.dev.yaml up -d

# 머지 결과 검사

docker compose -f compose.yaml -f compose.prod.yaml config
```

- **나중 파일이 앞의 키를 덮어씀**(정의 병합 규칙 숙지).

---

## 포트·네트워크 트러블슈팅(체크리스트)

| 증상 | 진단 | 해결 |
|---|---|---|
| `localhost:8080` 접속 불가 | `docker compose ps`, `docker compose logs web` | `ports` 매핑 확인, 서비스 정상기동 확인 |
| 컨테이너 간 접속 실패 | `docker compose exec web getent hosts api` | **같은 네트워크**에 연결, 서비스명으로 접근 |
| 이름해석 불가 | `/etc/resolv.conf`, `nslookup api 127.0.0.11` | 사용자 정의 브리지 사용 권장 |
| 로컬에서만 보이고 외부에서 안 보임 | `ports`에 바인딩 IP 확인 | `0.0.0.0:8080:80`로 확장 |
| 헬스체크 실패 | `docker compose ps`, `health` 상세 | `healthcheck` 조건 수정, 대기시간 늘림 |
| 권한 오류(마운트) | `id -u`, `id -g`, volume 퍼미션 | `user` 지정/권한 동기화, `:delegated`(Mac) |

**관찰 명령**
```bash
docker compose ps
docker compose logs -f web
docker compose config
docker network ls
docker network inspect <proj>_default
docker compose exec web sh -lc "ip addr; ip route; getent hosts api"
```

---

## 실전 예제 1 — 3계층: Nginx(프록시)/API/Postgres

```yaml
version: "3.9"
networks:
  frontnet: {}
  backnet: {}

volumes:
  pgdata: {}

secrets:
  dbpass:
    file: ./secrets/db.password

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/dbpass
    volumes:
      - pgdata:/var/lib/postgresql/data
    secrets: [dbpass]
    networks: [backnet]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres || exit 1"]
      interval: 5s
      timeout: 2s
      retries: 12

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
      target: runtime
    environment:
      DB_HOST: db
      DB_USER: postgres
      DB_PASS_FILE: /run/secrets/dbpass
      PORT: "8080"
    secrets: [dbpass]
    depends_on:
      db:
        condition: service_healthy
    networks: [backnet]
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8080/healthz"]
      interval: 10s
      timeout: 2s
      retries: 6
      start_period: 20s

  fe:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./ops/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - api
    networks: [frontnet, backnet]
```

`./ops/nginx.conf` (간단 프록시)
```nginx
server {
  listen 80;
  location / {
    proxy_pass http://api:8080;
  }
}
```

---

## 실전 예제 2 — 로컬 개발(핫리로드), 운영(세이프)

**dev**
```yaml
services:
  api:
    build: ./api
    volumes:
      - ./api:/app:delegated
    environment:
      APP_ENV: dev
```

**prod**
```yaml
services:
  api:
    image: myorg/api:1.2.3
    read_only: true
    tmpfs: ["/tmp"]
    user: "65532:65532"
    cap_drop: [ "ALL" ]
```

- dev는 **bind mount로 즉시 반영**,
- prod는 **이미지 고정·비루트·read_only**로 안전.

---

## 수식 직관: 공개 포트 최소화의 위험 감소

공격면 \(A\)를 공개 포트 수 \(k\)와 각 포트의 위협 가중치 \(w_i\)로 단순 모델링:
$$
A \approx \sum_{i=1}^{k} w_i
$$
- Compose에서 `ports`를 **정말 필요한 서비스**에만 선언 → \(k\) 최소화 → \(A\) 감소.

---

## 베스트 프랙티스(요약)

1. **사용자 정의 브리지 네트워크**로 서비스 분리(프런트/백).
2. **secrets/configs**로 민감정보/설정 분리, 환경변수 과다 노출 금지.
3. **멀티 스테이지** + `target`으로 런타임 경량화.
4. **비루트 + read_only + tmpfs + cap_drop**로 실행 표준화.
5. `depends_on.condition = service_healthy`로 **현실적인 의존**.
6. `docker compose config`로 **최종 구성 검증** 후 배포.
7. dev는 **bind mount**, prod는 **named volume**.
8. **profiles/오버레이 파일**로 환경별 구성 관리.
9. 컨테이너 간 통신은 **이름 기반**으로, 외부는 **필요 포트만** 공개.
10. 문제 시 **관찰 순서**: ps → logs → config → network inspect → 컨테이너 내 DNS/라우팅 확인.

---

## 빠른 참고 치트시트

```bash
# 올리고 내리고 보기

docker compose up -d
docker compose down
docker compose ps
docker compose logs -f api

# 셸/명령 실행

docker compose exec api sh
docker compose run --rm api pytest

# 스케일(수평확장; stateless 전제)

docker compose up -d --scale api=3

# 구성 검증/머지 결과

docker compose config

# 특정 서비스만 재빌드/재시작

docker compose build api
docker compose up -d api
```

---

## 참고 링크

- 공식 Compose 사양(파일 리퍼런스): https://docs.docker.com/compose/compose-file/
- Compose CLI: https://docs.docker.com/compose/
- YAML 문법: https://yaml.org/spec/
