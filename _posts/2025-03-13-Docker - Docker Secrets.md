---
layout: post
title: Docker - Docker Secrets
date: 2025-03-13 20:20:23 +0900
category: Docker
---
# Docker Secrets

## Docker Secrets가 해결하는 문제

| 방식 | 장점 | 단점 |
|---|---|---|
| `.env` 파일 | 단순하고 로컬 개발이 용이함 | 저장소 커밋, 백업, 협업 과정에서 노출 위험 |
| `-e KEY=VALUE` 환경변수 | 배포가 편리함 | `ps`, `docker inspect`, 프로세스 덤프, 디버그 페이지 등에서 노출 가능 |
| 파일 바인드 마운트 | 구조가 단순함 | 파일 권한, 백업, 로그 연쇄 경로를 통한 유출 가능 |
| **Docker Secrets** | 메모리 기반, 암호화 전달, 자동 읽기전용 마운트 | **Swarm 모드**가 필요하며, 변경 시 "교체"가 원칙(수정 불가) |

핵심은 **비가시화**입니다. Docker Secrets는 민감정보를 **환경변수, 로그, 이미지 레이어** 어디에도 남기지 않도록 설계되었습니다.

---

## 원리: Swarm, Raft, 전달·마운트 모델

- **Swarm 모드에서만 동작**합니다. 매니저 노드는 내부 **Raft 로그에 암호화**하여 Secret 메타데이터를 저장합니다.
- Secret은 서비스에 할당될 때 **TLS 채널로 작업 노드에 전달**되고, 컨테이너 내부에는 **`tmpfs`(메모리) 상의 읽기전용 파일**로 마운트됩니다.
- 컨테이너 내 기본 경로는 `/run/secrets/<name>` 이며, **환경변수로 주입하지 않습니다.**
- Secret은 **수정(update)이 불가능**합니다. 갱신은 "새 Secret을 만들고 서비스를 재배포하여 교체"하는 방식이 권장됩니다.

간략한 추상 위협 모델에서 공격자 관점의 접근 비용 함수 \(C\)는 다음과 같이 근사할 수 있습니다:

$$
C = C_{\text{호스트}} + C_{\text{컨테이너}} + C_{\text{네트워크}} + C_{\text{로그/백업}}
$$

Secrets는 **네트워크 구간 암호화**, **컨테이너 내부 비영속(tmpfs) 저장**, **로그/백업 경로 비노출**을 통해 \(C_{\text{네트워크}}\)와 \(C_{\text{로그/백업}}\) 항목의 비용을 크게 증가시킵니다.

---

## 빠른 시작: Swarm에서 Secrets 기본 플로우

### Swarm 초기화

```bash
docker swarm init
```

### Secret 생성

```bash
# 표준입력으로 생성
echo "my-secret-password" | docker secret create db_password -
docker secret ls
```

### Secret을 사용하는 서비스 생성

```bash
docker service create \
  --name myapp \
  --secret db_password \
  myapp-image
```

컨테이너 내부에서:
```bash
# /run/secrets/db_password 로 읽기
cat /run/secrets/db_password
```

---

## Compose와 Stack: 개발에서 운영까지

### Compose 파일로 선언 및 Swarm 배포

```yaml
# docker-compose.yml
version: "3.9"

services:
  web:
    image: myorg/web:1.0
    secrets:
      - db_password
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

배포:
```bash
docker stack deploy -c docker-compose.yml mystack
docker service ls
```

일반 `docker-compose up`은 로컬 Docker Compose 엔진을 사용하며, **Secrets를 완전한 Swarm 방식으로 사용하려면 `docker stack deploy`** 를 이용해야 합니다.

### 기존 네트워크 및 볼륨과 통합 예시 (PostgreSQL)

```yaml
version: "3.9"

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: app
    secrets:
      - db_password
    command: >
      bash -lc 'export PGPASSWORD="$(cat /run/secrets/db_password)" &&
                docker-entrypoint.sh postgres'
    volumes:
      - dbdata:/var/lib/postgresql/data
    networks: [ backend ]

  api:
    image: myorg/api:1.0
    secrets:
      - db_password
    environment:
      DB_HOST: db
      DB_USER: app
    networks: [ backend ]
    depends_on: [ db ]

secrets:
  db_password:
    file: ./secrets/db_password.txt

volumes:
  dbdata:

networks:
  backend:
```

패턴 설명:
- DB 컨테이너의 엔트리포인트 앞단에서 Secret 파일을 읽어 **일시적으로 프로세스 환경변수로 전달**한 후, 애플리케이션이 자체적으로 커넥션 풀을 관리합니다.
- 보다 안전한 방식은 애플리케이션이 **파일 경로에서 직접 Secret을 읽도록** 구성하는 것입니다. 이렇게 하면 환경변수 주입 구간을 최소화할 수 있습니다.

---

## 애플리케이션 코드 패턴

### Python 예시

```python
from pathlib import Path
import os

SECRET_PATH = os.getenv("DB_PASSWORD_FILE", "/run/secrets/db_password")

def read_secret(p=SECRET_PATH):
    return Path(p).read_text(encoding="utf-8").strip()

DB_PASSWORD = read_secret()
# 이후 DB 연결 문자열에 사용
```

### Node.js 예시

```js
const fs = require('fs');
const SECRET_PATH = process.env.DB_PASSWORD_FILE || '/run/secrets/db_password';
const dbPassword = fs.readFileSync(SECRET_PATH, 'utf8').trim();
// DB client 구성 시 dbPassword 사용
```

### Java 예시

```java
String path = System.getenv().getOrDefault("DB_PASSWORD_FILE", "/run/secrets/db_password");
String password = Files.readString(Path.of(path), StandardCharsets.UTF_8).trim();
// DataSource/Properties에 주입
```

권장 사항: **경로 주입 방식**을 12-Factor와 결합하려면, `DB_PASSWORD_FILE=/run/secrets/db_password` 같은 환경변수만 노출하고, 실제 값은 파일에서 읽도록 합니다. 이렇게 하면 로컬 테스트 환경에서 임시 파일로 쉽게 대체할 수 있습니다.

---

## 운영 설계: 회전, 버저닝, 무중단 교체

### Secret 교체 원칙

- "수정"이 아닌 **새 Secret 생성 → 서비스 업데이트로 교체**가 기본입니다.
- 버전명 전략을 사용하세요:
  - `db_password_v1`, `db_password_v2` 와 같이 **명시적 버저닝**
  - 교체 단계에서 두 Secret을 모두 마운트하고, 애플리케이션이 **새 경로를 우선** 사용하도록 한 후, 재기동 시점에 구 Secret을 제거

### 무중단 교체 패턴 (Blue/Green 방식)

1) 새 Secret 생성:
```bash
echo "newpass" | docker secret create db_password_v2 -
```
2) 서비스에 **새 Secret과 기존 Secret을 함께 연결**:
```bash
docker service update \
  --secret-add db_password_v2 \
  mystack_api
```
3) 애플리케이션이 `db_password_v2`를 우선 사용하도록 구성(예: 환경변수 `DB_PASSWORD_FILE=/run/secrets/db_password_v2`).
4) 안정화 확인 후 기존 Secret 제거:
```bash
docker service update \
  --secret-rm db_password \
  mystack_api
docker secret rm db_password
```

### 장애 대응

- Secret 파일 접근 실패 시, 애플리케이션은 **지수 백오프 재시도** 또는 **즉시 종료(오케스트레이션 재시작 유도)** 중 하나를 명확히 선택해야 합니다.
- 컨테이너 시작 시점부터 Secret 파일이 존재하므로, 초기 접근 실패는 배치 타이밍 이슈보다 **권한 문제나 경로 오타**일 가능성이 높습니다.

---

## Nginx Basic Auth, TLS 키/인증서 관리 예시

### Basic Auth

```bash
# htpasswd로 생성한 파일을 secret으로 등록
docker secret create web_htpasswd ./secrets/.htpasswd
```

```yaml
version: "3.9"
services:
  nginx:
    image: nginx:alpine
    secrets:
      - web_htpasswd
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    # nginx.conf 설정:
    # auth_basic "Restricted";
    # auth_basic_user_file /run/secrets/web_htpasswd;
secrets:
  web_htpasswd:
    external: true
```

### TLS Private Key/Certificate

```bash
docker secret create tls_key ./secrets/tls.key
docker secret create tls_crt ./secrets/tls.crt
```

Nginx 설정에서:
```
ssl_certificate     /run/secrets/tls_crt;
ssl_certificate_key /run/secrets/tls_key;
```

---

## Compose 개발 환경과의 간극 해소

Swarm 없이 로컬 개발만 할 때는 Docker Secrets의 **완전한 보안 이점을 누리기 어렵습니다.** 대안은 다음과 같습니다:

1) **BuildKit 빌드-타임 시크릿**을 활용하여 레지스트리 자격증명이나 프라이빗 종속성 설치 시 안전하게 사용합니다.
2) 개발 환경에서는 바인드 마운트로 대체하되, **프로덕션과 동일한 경로**(`/run/secrets/...`)를 유지하여 코드 변경 없이 배포할 수 있도록 심볼릭 링크나 환경변수로 치환합니다.
3) 통합 테스트 파이프라인은 반드시 Swarm 또는 스테이징 클러스터 상에서 수행합니다.

---

## BuildKit의 빌드 타임 시크릿

런타임 Secret과 별개로, **빌드 과정에서만** 필요한 자격증명(예: 프라이빗 레포지토리 토큰)은 BuildKit의 `RUN --mount=type=secret`으로 안전하게 전달합니다. 이 값은 최종 이미지 레이어에 남지 않습니다.

### 사용 예시 (Dockerfile)

```dockerfile
# syntax=docker/dockerfile:1.7

FROM alpine:3.20
RUN --mount=type=secret,id=npmrc,dst=/root/.npmrc \
    apk add --no-cache nodejs npm && npm ci
```

빌드 시:
```bash
DOCKER_BUILDKIT=1 docker build \
  --secret id=npmrc,src=$HOME/.npmrc \
  -t my/app:build .
```

중요한 점:
- 빌드 시크릿은 최종 이미지에 남지 않습니다.
- **런타임 시크릿(Docker Secrets)** 과 목적이 다릅니다: 빌드 과정 대 실행 환경.

---

## 외부 시크릿 매니저와의 통합 전략

Docker Secrets는 Swarm 내장형 솔루션입니다. 조직에서 이미 **HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault** 등을 사용하고 있다면 다음 전략 중 하나를 선택할 수 있습니다.

- **애플리케이션 직접 조회**: 시작 시 Vault에서 비밀을 가져와 메모리에 보관합니다. 파일에 기록하지 않으며, 회전도 Vault 정책으로 처리합니다.
- **사이드카/에이전트 패턴**: 사이드카 컨테이너가 주기적으로 Secret을 동기화하여 `/run/secrets/...` 경로에 반영합니다. 애플리케이션은 기존 파일 기반 방식 유지. 파일 변경 감지 및 핫 리로드 설계가 필요합니다.
- **사전-주입(Init 컨테이너)**: 시작 시 한 번 주입하고, 변경 시에는 롤링 업데이트로 전체 서비스를 교체합니다.

외부 Vault를 사용하더라도, **컨테이너 내부 경로/권한 관리 및 로그 노출 방지**라는 기본 원칙은 동일하게 적용됩니다.

---

## 운영 및 보안 고려사항

1.  **최소 권한 원칙**: Secret은 필요한 서비스에만 연결하고, 서로 다른 용도로 재사용하지 마십시오.
2.  **로그 방지**: Secret 내용이 애플리케이션 로그, 프레임워크 에러 리포트, ORM 쿼리 로그 등에 기록되지 않도록 설정을 검토하십시오.
3.  **메모리 관리**: Secret 값을 문자열 상수나 장기 생존하는 전역 변수에 보관하기보다는, 필요 시점에 읽고 가능한 한 빨리 참조를 해제하도록 설계하십시오.
4.  **백업 범위 분리**: Secret 원본 파일이 저장된 경로(예: `./secrets/*.txt`)가 일반 백업/스토리지에 포함되지 않도록 운영 환경과 개발 환경을 명확히 구분하십시오.
5.  **감사 추적**: Secret의 버전, 생성자, 배포 릴리스, 회전 이력을 문서화하여 변경 내역을 추적할 수 있도록 합니다.
6.  **종단간 암호화**: Swarm 클러스터의 TLS 통신을 기본으로 유지하고, 매니저 노드를 물리적으로 안전하게 보호하십시오.

---

## 자주 발생하는 오류와 해결 방법

| 증상 | 원인 | 해결 |
|---|---|---|
| 컨테이너에 `/run/secrets/...` 파일 없음 | Swarm 모드가 아닌 `docker run` 사용 | `docker swarm init` 후 `docker service` 또는 `docker stack deploy` 사용 |
| Permission denied 오류 | 컨테이너 내 사용자(USER)가 파일 읽기 권한 없음 | 컨테이너 `USER`의 권한과 Secret 파일의 읽기 가능 여부 확인(읽기전용) |
| DB 접속 실패(비밀번호 무시됨) | 애플리케이션이 환경변수만 참조하고 파일은 읽지 않음 | `*_FILE` 패턴 지원 여부 확인 또는 Secret 파일을 직접 읽도록 코드 수정 |
| Secret 회전 후 일부 인스턴스가 예전 값 사용 | 롤링 업데이트 중 경로 혼합 상태 | 이중 마운트 패턴(신규/기존)과 우선순위 전략 적용, 안정화 후 기존 Secret 제거 |
| CI/CD 파이프라인 로그에 비밀 노출 | 테스트 로그나 스크린샷에 민감정보 포함 | 비밀번호/토큰 자동 마스킹 규칙 적용, 비밀 내용 출력 금지 |

---

## 스택 예시

### 디렉터리 구조

```
stack/
├── docker-compose.yml
└── secrets/
    ├── db_password.txt
    ├── tls.crt
    └── tls.key
```

### docker-compose.yml 파일

```yaml
version: "3.9"

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: app
    secrets:
      - db_password
    command: >
      bash -lc 'export PGPASSWORD="$(cat /run/secrets/db_password)" &&
                docker-entrypoint.sh postgres'
    volumes:
      - dbdata:/var/lib/postgresql/data
    networks: [ backend ]

  api:
    image: myorg/api:1.0
    secrets: [ db_password ]
    environment:
      DB_HOST: db
      DB_USER: app
      DB_PASSWORD_FILE: /run/secrets/db_password
    depends_on: [ db ]
    networks: [ backend ]

  web:
    image: nginx:alpine
    depends_on: [ api ]
    ports:
      - "443:443"
    secrets:
      - tls_crt
      - tls_key
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks: [ backend ]
    # nginx.conf에서 ssl_certificate/key 경로를 /run/secrets/tls_crt|tls_key 로 지정

secrets:
  db_password:
    file: ./secrets/db_password.txt
  tls_crt:
    file: ./secrets/tls.crt
  tls_key:
    file: ./secrets/tls.key

volumes:
  dbdata:

networks:
  backend:
```

배포:
```bash
docker swarm init           # 최초 1회 실행
docker stack deploy -c docker-compose.yml mystack
```

---

## 성능 및 운영 팁

- Secret 파일은 일반적으로 몇 KB 수준의 작은 텍스트이므로 성능에 미치는 영향은 미미합니다.
- Secret 개수가 많다면 **명명 규칙**을 정의하고, **서비스 단위로 논리적으로 묶어** 관리 복잡도를 낮추십시오.
- 애플리케이션이 Secret 파일을 **런타임에 반복해서 읽을 필요는 없습니다**. 시작 시 한 번 읽고 커넥션을 유지하며, Secret 회전 이벤트가 발생하는 경우에만 재로딩하는 방식을 고려하십시오.

---

## 결론

Docker Secrets는 Swarm 오케스트레이션 환경에서 **안전하고 관리하기 쉬운 런타임 비밀 전달 수단**을 제공합니다. 그 핵심 가치는 민감정보가 **환경변수, 애플리케이션 로그, 또는 도커 이미지 레이어에 노출되는 것을 방지**하고, **네트워크를 통해 암호화된 상태로 전달**된다는 점에 있습니다.

운영 측면에서는 **버저닝 전략, 무중단 교체 패턴, 그리고 감사 및 백업 경로에 대한 주의 깊은 관리**가 성공적인 도입의 핵심 요소입니다. 또한, 빌드 단계의 비밀은 **BuildKit 시크릿**으로, 대규모 멀티 클라우드 환경에서는 **외부 시크릿 매니저와의 하이브리드 구성**이 실용적인 접근법이 될 수 있습니다.

---

## 부록 A. 주요 명령어 요약

```bash
# Swarm 모드 초기화
docker swarm init

# Secret 생성/조회/삭제
echo "secret" | docker secret create mysecret -
docker secret ls
docker secret rm mysecret

# Secret을 사용하는 서비스 생성
docker service create --name app --secret mysecret myimg:tag

# Compose 파일을 이용한 스택 배포
docker stack deploy -c docker-compose.yml mystack
```