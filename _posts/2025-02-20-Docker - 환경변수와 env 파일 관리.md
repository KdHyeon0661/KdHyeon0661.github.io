---
layout: post
title: Docker - 환경변수와 '.env' 파일 관리
date: 2025-02-20 19:20:23 +0900
category: Docker
---
# Docker Compose의 환경변수와 `.env` 파일 관리

## 환경변수란? (핵심 복습)

컨테이너/애플리케이션이 **코드 밖에서 주입받는 설정값**입니다.

| 목적 | 설명 |
|---|---|
| 보안성 | 비밀번호, 토큰을 코드/이미지에서 분리 |
| 유연성 | 환경에 따라 설정만 교체(코드 수정 없음) |
| 재사용성 | dev/stage/prod 간 동일 Compose에 다른 값 적용 |

---

## `.env` 파일 — 로딩 위치, 기본 형식, 자주 틀리는 포인트

- **위치**: 기본은 `docker-compose.yml`(또는 `compose.yaml`)와 **같은 디렉터리**.
- **자동 로드**: `docker compose up` 시 별도 옵션 없이 자동 탐색/로드.
- **대안**: 다른 경로/이름을 쓰고 싶으면 `--env-file path/to/your.env` 사용.

```env
# .env (예시)

MYSQL_ROOT_PASSWORD=root_pass
MYSQL_DATABASE=mydb
MYSQL_USER=myuser
MYSQL_PASSWORD=mypass
WEB_PORT=8080
```

**작성 규칙/주의**
- 한 줄에 `KEY=VALUE` (공백 없이).
- 주석은 `#` 로 시작.
- **따옴표 불필요**(공백/특수문자 포함이면 `"..."` 권장).
- **CRLF(Windows 줄바꿈)** 은 때때로 파싱 이슈를 만듭니다 → 가능하면 **UTF-8 LF** 사용.
- 값에 `$`가 필요하면 치환 방지 위해 **`$$`** 로 이스케이프.

---

## Compose에서 환경변수를 쓰는 3가지 경로 — 개념 분리

| 위치 | 용도 | 변수 치환(인터폴레이션) | 실제 컨테이너에 주입 |
|---|---|---|---|
| **Compose YAML** 내 `${VAR}` | **파일 해석 시** 값 치환 | O (Compose가 해석) | 해석된 결과만 반영 |
| `environment:` | 컨테이너 환경 변수 선언 | O (`${VAR}` 허용) | O |
| `env_file:` | 파일을 컨테이너 ENV로 로드 | X (단순 key=value 전달) | O |

핵심:
- `.env`는 **Compose 파일을 해석**할 때 사용할 **치환 소스**.
- `env_file`은 **컨테이너 안으로 환경변수를 주입**하는 목록 파일.

---

## 변수 **우선순위(Precedence)** — 어떤 값이 최종 적용되나

Compose가 `${VAR}`를 해석할 때의 일반적인 우선순위(높음→낮음):

1. **명령행 `--env-file`** 로 지정한 파일
2. **쉘 환경변수**(현재 터미널의 `export VAR=...`)
3. **Compose 파일과 같은 폴더의 `.env`**
4. **Compose 파일 내 `${VAR:-default}` 의 기본값**
5. **미정의 → 에러(혹은 빈 값)**

컨테이너 내부 ENV의 최종 우선순위(일반적 관측):

1. `environment:`에 **치환 해석된 값**
2. `env_file:` 로 전달된 값(파일에 적힌 그대로)
3. **이미지의 `ENV`** 기본값

> 해석 결과는 `docker compose config`로 미리 확인 가능합니다(아래 §11).

---

## Compose YAML에서의 변수 치환 문법(정석)

```yaml
services:
  web:
    image: wordpress
    ports:
      - "${WEB_PORT}:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
      WORDPRESS_DB_USER: ${MYSQL_USER}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
```

### 기본/확장 문법

```yaml
environment:
  # 기본값 제공
  - DB_HOST=${MYSQL_HOST:-localhost}
  - DB_PORT=${MYSQL_PORT:-3306}

  # 미정의면 에러 (명시적 검증)
  - SECRET_KEY=${SECRET_KEY:?SECRET_KEY is required}

  # 빈 값 허용(정말 비워야 할 때)
  - EMPTY_OK=${CAN_BE_EMPTY-}
```

- `${VAR:-default}`: `VAR` 없으면 `default`.
- `${VAR:?error}`: `VAR` 없으면 **에러로 중단**(CI/CD에서 유용).
- `${VAR-default}`: `VAR`가 **정의되지 않았을 때만** default(빈 문자열은 그대로).

**이스케이프**
- 문자열에 `$`가 필요하다면 `$$` 사용:
  ```yaml
  environment:
    LITERAL_DOLLAR: "$$HOME"   # 컨테이너 안에는 $HOME가 아닌 문자열 "$HOME" 저장
  ```

---

## `.env` vs `env_file` — 완전히 다른 역할

### `.env` (프로젝트 루트에 있는 전역 치환 소스)

- Compose 파일에 있는 `${VAR}` **치환**에 사용.
- 컨테이너 안에는 자동으로 **주입되지 않음**.

### `env_file` (컨테이너 환경변수 주입 목록)

```yaml
services:
  app:
    build: .
    env_file:
      - .env              # 컨테이너로 전달(치환 없음)
      - .env.secret       # 여러 파일 가능 (아래 파일이 우선 적용)
```

- `env_file` 내 값들은 **그대로** 컨테이너 환경에 주입(치환 X).
- 민감값을 **여러 파일로 분리** 가능. 중복 키는 **아래 파일이 우선**.

> 둘을 혼동하면 “왜 컨테이너에 값이 안 들어오지?”라는 혼란이 발생합니다.

---

## 개발/운영 환경 분리 — 파일, 프로필, Makefile 조합

### `.env.dev` / `.env.prod`와 `--env-file`

```bash
docker compose --env-file .env.dev  up -d
docker compose --env-file .env.prod up -d
```

### `profiles`로 서비스 온/오프(예: 캐시, 모니터링)

```yaml
services:
  redis:
    image: redis:7
    profiles: ["cache"]

  loki:
    image: grafana/loki:2.9
    profiles: ["observability"]
```
```bash
# 캐시만 올림

COMPOSE_PROFILES=cache docker compose up -d
# 캐시+관측

COMPOSE_PROFILES=cache,observability docker compose up -d
```

### Makefile로 팀 규칙 표준화

```makefile
up-dev:
	@docker compose --env-file .env.dev up -d

up-prod:
	@docker compose --env-file .env.prod up -d

config-dev:
	@docker compose --env-file .env.dev config
```

---

## 포트/볼륨/네트워크에서도 치환 가능

```yaml
services:
  web:
    ports:
      - "${WEB_PORT:-8080}:80"
    volumes:
      - "${HOST_HTML_DIR:-./html}:/var/www/html:ro"
networks:
  backend:
    driver: ${NET_DRIVER:-bridge}
```

**Windows 경로 주의**: `C:\path\to\data:/app/data` 처럼 `:`가 들어가서 YAML 파싱/포트와 충돌할 수 있음 →
**따옴표로 감싸기**를 습관화:
```yaml
volumes:
  - "C:\data\wp:/var/www/html"
```

---

## 보안 — 비밀번호는 환경변수로만? 더 안전한 선택지

| 방식 | 장점 | 주의점 |
|---|---|---|
| `.env`/`environment` | 간단/빠름 | `docker inspect`, `ps` 출력 등 **노출 위험** |
| `env_file` | 파일로 분리 | 여전히 컨테이너 ENV(노출 가능) |
| **secrets**(Compose/Swarm) | 파일로 **/run/secrets/** 에 마운트, ENV에 안 뜸 | 앱이 **파일을 읽도록** 구현 필요 |

### Compose secrets(로컬에서도 사용 가능)

```yaml
services:
  db:
    image: mysql:8
    secrets:
      - db_root_pw
    environment:
      # MySQL은 파일 경로 방식도 지원
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_pw

secrets:
  db_root_pw:
    file: ./secrets/mysql_root_password.txt
```

- 장점: `docker inspect`로 ENV에서 비번이 노출되지 않음.
- 애플리케이션이 **파일 경로**에서 비밀을 읽도록 설계해야 함.
- CI/CD에서는 Vault/Parameter Store와 연동해 secrets 파일을 생성 후 배포.

**반드시** `.gitignore`에 민감 파일 추가:
```gitignore
.env
.env.*
secrets/
```

---

## 실전 예시 — WordPress + MySQL, dev/prod 분리

### `.env.dev`

```env
WEB_PORT=8080
MYSQL_ROOT_PASSWORD=devroot
MYSQL_DATABASE=blog
MYSQL_USER=devuser
MYSQL_PASSWORD=devpass
```

### `.env.prod`

```env
WEB_PORT=80
MYSQL_ROOT_PASSWORD=please_change_me_strong
MYSQL_DATABASE=blog
MYSQL_USER=wpuser
MYSQL_PASSWORD=strong_random_64
```

### `docker-compose.yml`

```yaml
version: '3.9'

services:
  wordpress:
    image: wordpress:6.5
    ports:
      - "${WEB_PORT}:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
      WORDPRESS_DB_USER: ${MYSQL_USER}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
    depends_on:
      - db

  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:?ROOT password required}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
```

**실행**
```bash
# 개발

docker compose --env-file .env.dev up -d

# 운영

docker compose --env-file .env.prod up -d
```

---

## 검증/디버깅 — 실제로 어떤 값이 적용됐나

### Compose가 해석한 최종 YAML 보기

```bash
docker compose config
# 또는

docker compose --env-file .env.prod config
```
- 이 출력에는 `${VAR}`가 **전부 치환된 상태**로 나타납니다.
- CI 파이프라인에서 **사전 검증** 단계로 강력 추천.

### 컨테이너 내부에서 ENV 확인

```bash
docker compose exec web env | sort
docker compose exec db  printenv | sort
```

### 빠른 건강검진

```bash
docker compose ps
docker compose logs -f db
docker compose logs -f wordpress
```

---

## 트러블슈팅(자주 겪는 함정과 해결책)

| 증상 | 원인 | 해결 |
|---|---|---|
| `${VAR}`가 문자열 그대로 남음 | `.env`/쉘 환경에 값 없음 | `docker compose config`로 치환 상태 확인, `--env-file` 사용 여부 체크 |
| 값에 `$`가 들어가면 깨짐 | 치환/셸 해석 | Compose YAML에서 `$$` 이스케이프, 또는 따옴표로 감싸기 |
| Windows 바인드 마운트 실패 | 경로/콜론 충돌 | `"C:\path\to\dir:/container/path"`처럼 **따옴표** 사용 |
| `depends_on`인데도 DB 연결 실패 | `depends_on`은 **준비완료 대기**가 아님 | **healthcheck**로 준비 상태 보장, 재시도 로직 사용 |
| 비밀번호가 `docker inspect`에 보임 | ENV 주입 방식 | **secrets** 사용, 애플리케이션이 파일 경로에서 읽도록 |
| `.env` 무시됨 | 경로 다름 | Compose 파일과 **같은 폴더**인지, 또는 `--env-file`로 정확히 지정 |
| `.env` 변수의 따옴표 포함됨 | 값 포맷 문제 | 값에 공백/특수문자가 없다면 **따옴표 제거**, 필요 시 `"..."`로 정확히 묶기 |

---

## CI/CD 파이프라인에서의 베스트 프랙티스

- **Secrets/Variables**: CI 시스템의 **보안 저장소**(GitHub Actions Secrets, GitLab CI Variables 등)에 보관.
- **템플릿 파일**: `.env.example`를 리포에 포함(키 목록만 제공).
- **사전 검증**: 배포 전 `docker compose --env-file <env> config` 실행,
  `${VAR:?missing}`로 **필수값 검증**(누락 시 즉시 실패).
- **감사/추적**: 변경 이력은 `.env`가 아닌 **CI 변수/시크릿 관리 이력**을 통해 추적.

---

## 심화 예제 — `env_file` + `environment` 혼합, 기본값, 필수값

```yaml
services:
  api:
    image: myorg/api:1.2.3
    env_file:
      - ./.env.base     # 공통 키/값 (치환 X)
      - ./.env.secret   # 민감값(치환 X, Git에 올리지 않음)
    environment:
      # 치환/검증: 정의 없으면 기본값, 또는 에러
      APP_PORT:  "${APP_PORT:-8080}"
      APP_MODE:  "${APP_MODE:-production}"
      JWT_ISS:   "${JWT_ISS:?JWT_ISS required}"
      # env_file과 충돌 시 여기 값이 **우선**
    ports:
      - "${APP_PORT:-8080}:8080"
```

- 운영에서 공통은 `env_file`로, 서비스별로 다른 값/필수값은 `environment`로 **오버라이드 + 검증**.

---

## 요약 정리 — 한눈에 보는 체크리스트

1. `.env`는 **치환 소스**, `env_file`은 **컨테이너 주입**(치환 X).
2. 우선순위는 **`--env-file` > 쉘 ENV > .env > 기본값**.
3. `${VAR:-default}`, `${VAR:?error}` 로 **기본값/필수값**을 엄격히 관리.
4. 민감값은 ENV 대신 **secrets(/run/secrets)** 또는 외부 비밀 저장소 사용.
5. **profiles/--env-file**/Makefile로 환경 분리 자동화.
6. 릴리즈 전 `docker compose config`로 **치환 결과**를 항상 확인.
7. 운영 노출 경로(`docker inspect`, 로그, 히스토리)를 고려해 **비밀/경로/포맷**을 설계.

---

## 부록 A) 미니 실습 — 60초 설정 검증 스크립트

```bash
#!/usr/bin/env bash

set -euo pipefail

ENV_FILE="${1:-.env}"

echo "[*] Using env-file: ${ENV_FILE}"
docker compose --env-file "${ENV_FILE}" config > /tmp/compose.out

echo "[*] Resolved compose preview:"
sed -n '1,80p' /tmp/compose.out

echo "[*] Sanity check: required envs"
grep -E "MYSQL_ROOT_PASSWORD|MYSQL_DATABASE|MYSQL_USER|MYSQL_PASSWORD" /tmp/compose.out >/dev/null \
  && echo "  - MySQL envs present" \
  || { echo "  - Missing MySQL envs"; exit 1; }

echo "[OK] Validation passed."
```

---

## 부록 B) `.gitignore` 템플릿

```gitignore
# env & secrets

.env
.env.*
secrets/
# local artifacts

*.log
*.tmp
```

---

## 참고 문서(추가 탐독 권장)

- Docker Compose: Environment variables / Variable substitution
- 12-Factor: Config
- Docker secrets & Swarm / Compose secrets

위 가이드를 적용하면 **개발부터 운영까지** 환경변수를 **예측 가능하고 안전**하게 운용할 수 있습니다.
값이 어떻게 최종 적용되는지 의심이 들면 **항상 `docker compose config`로 미리 확인**하세요.
