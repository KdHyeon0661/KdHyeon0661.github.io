---
layout: post
title: Docker - Docker Secrets
date: 2025-03-13 20:20:23 +0900
category: Docker
---
# Docker Secrets

## 0. Docker Secrets가 풀어주는 문제

| 방식 | 장점 | 치명적 단점 |
|---|---|---|
| `.env` 파일 | 단순, 로컬 개발 용이 | 저장소 커밋/백업/협업 경로로 노출 위험 상존 |
| `-e KEY=VALUE` 환경변수 | 배포 편리 | `ps`, `docker inspect`, 프로세스 덤프, 앱 프레임워크 디버그 페이지에서 노출될 수 있음 |
| 파일 바인드 마운트 | 구조 단순 | 파일 권한/백업/로그 연쇄 경로로 유출 가능 |
| **Docker Secrets** | 메모리 기반, 암호화 전달, 자동 마운트(읽기전용) | **Swarm 모드** 필요, 변경시 “교체”가 원칙(수정 불가) |

핵심은 **비가시화**다. Docker Secrets는 민감정보를 **환경변수/로그/레이어** 어디에도 남기지 않도록 설계되었다.

---

## 1. 원리: Swarm, Raft, 전달·마운트 모델

- **Swarm 모드에서만 동작**한다. 매니저 노드는 내부적으로 **Raft 로그에 암호화**하여 Secret 메타데이터를 저장한다.
- Secret은 서비스에 할당될 때 **TLS 채널로 작업 노드에 전달**되고, 컨테이너 내부에는 **`tmpfs`(메모리) 상의 읽기전용 파일**로 마운트된다.
- 컨테이너 내 경로는 기본적으로 `/run/secrets/<name>` 이며, **환경변수로 주입하지 않는다.**
- Secret은 **수정(update) 불가**. 갱신은 “새 Secret을 만들고 서비스를 재배포하여 교체”하는 방식이 권장된다.

간단한 추상 위협 모델을 쓰면, 공격자 관점의 접근 비용 함수 \(C\)는 다음과 같이 근사할 수 있다:

$$
C = C_{\text{호스트}} + C_{\text{컨테이너}} + C_{\text{네트워크}} + C_{\text{로그/백업}}
$$

Secrets는 **네트워크 구간 암호화**와 **컨테이너 내부 비영속(tmpfs)**, **로그/백업 경로 비노출**을 통해 \(C_{\text{네트워크}}, C_{\text{로그/백업}}\)을 크게 증가시킨다.

---

## 2. 빠른 시작: Swarm에서 Secrets 기본 플로우

### 2.1 Swarm 초기화
```bash
docker swarm init
```

### 2.2 Secret 생성
```bash
# 표준입력으로 생성
echo "my-secret-password" | docker secret create db_password -
docker secret ls
```

### 2.3 Secret을 사용하는 서비스 생성
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

## 3. Compose와 Stack: 개발에서 운영까지

### 3.1 Compose 파일로 선언(로컬 선언 → Swarm 배포)
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

주의: 일반 `docker-compose up`은 로컬 Docker Compose 엔진 기준으로 동작하며, **Secrets를 완전한 Swarm 의미로 사용하려면 `docker stack deploy`** 를 이용해야 한다.

### 3.2 기존 네트워크/볼륨과 통합 예시 (PostgreSQL)
```yaml
version: "3.9"

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: app
      # 비밀번호는 환경변수로 주지 않고 secret 파일로 마운트
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

패턴 설명
- DB 컨테이너의 엔트리포인트 앞단에서 Secret을 파일에서 읽어 **프로세스 환경변수로 잠깐 전달**하고, 이후에는 앱이 자체적으로 커넥션 풀을 관리하게 한다.
- 더 안전하게 하려면, 애플리케이션이 **파일 경로에서 직접 읽도록** 구성하는 것이 바람직하다(환경변수 주입 구간 최소화).

---

## 4. 애플리케이션 코드 패턴

### 4.1 Python(Flask) 예
```python
from pathlib import Path
import os

SECRET_PATH = os.getenv("DB_PASSWORD_FILE", "/run/secrets/db_password")

def read_secret(p=SECRET_PATH):
    return Path(p).read_text(encoding="utf-8").strip()

DB_PASSWORD = read_secret()
# 이후 DB 연결 문자열에 사용
```

### 4.2 Node.js 예
```js
const fs = require('fs');
const SECRET_PATH = process.env.DB_PASSWORD_FILE || '/run/secrets/db_password';
const dbPassword = fs.readFileSync(SECRET_PATH, 'utf8').trim();
// DB client 구성 시 dbPassword 사용
```

### 4.3 Java(Spring) 예
```java
String path = System.getenv().getOrDefault("DB_PASSWORD_FILE", "/run/secrets/db_password");
String password = Files.readString(Path.of(path), StandardCharsets.UTF_8).trim();
// DataSource/Properties에 주입
```

권장: **경로 주입 방식**을 12-Factor와 결합하려면, `DB_PASSWORD_FILE=/run/secrets/db_password` 같은 환경변수만 노출하고, 값은 파일에서 읽도록 하라. 그러면 테스트(로컬)에서는 임시 파일로 쉽게 대체 가능하다.

---

## 5. 운영 설계: 회전(로테이션), 버저닝, 무중단 교체

### 5.1 Secret 교체의 원칙
- “수정”이 아니라 **새 Secret 생성 → 서비스 업데이트로 교체**다.
- 버전명 전략을 사용하라:
  - `db_password_v1`, `db_password_v2` 처럼 **명시적 버저닝**
  - 교체 단계에서 두 Secret을 모두 마운트하고 앱이 **새 경로 우선**, 구 경로 fallback 후 재기동 시 구 Secret 제거

### 5.2 무중단 교체 패턴(Blue/Green-like)
1) 새 Secret 생성:
```bash
echo "newpass" | docker secret create db_password_v2 -
```
2) 서비스에 **두 Secret**을 함께 연결:
```bash
docker service update \
  --secret-add db_password_v2 \
  mystack_api
```
3) 애플리케이션이 `db_password_v2` 우선 사용하도록 구성(환경변수 `DB_PASSWORD_FILE=/run/secrets/db_password_v2` 등).
4) 안정화 확인 후 구 Secret 제거:
```bash
docker service update \
  --secret-rm db_password \
  mystack_api

docker secret rm db_password
```

### 5.3 장애 대응
- Secret 파일 접근 실패 시, 앱은 **지수 백오프 재시도** 또는 **읽기 실패시 종료**(오케스트레이션 재시작) 중 하나를 명확히 택하라.
- 컨테이너 시작 시점부터 Secret 파일이 존재하므로, 일반적으로 초기 접근 실패는 배치 타이밍 이슈보다 **권한·오타** 문제일 확률이 높다.

---

## 6. Nginx Basic Auth, TLS 키/인증서 관리 예시

### 6.1 Basic Auth
```bash
# htpasswd로 만든 파일을 secret으로 등록
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
    # nginx.conf에서:
    # auth_basic "Restricted";
    # auth_basic_user_file /run/secrets/web_htpasswd;

secrets:
  web_htpasswd:
    external: true
```

### 6.2 TLS Private Key/Cert
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

## 7. Compose 개발 환경과의 간극 메우기

Swarm 없이 로컬 개발만 할 때는 Docker Secrets의 **보안 이점이 완벽히 동일하지 않다.** 대안:

1) **BuildKit 빌드-타임 시크릿**(아래 8장)으로 레지스트리 자격증명/프라이빗 종속성 설치 시 안전사용.
2) 개발은 바인드 마운트로 대체하되, **프로덕션 동일 경로**(`/run/secrets/...`)를 지켜서 코드 변경 없이 배포할 수 있도록 심볼릭 링크/환경변수로 치환.
3) 통합 테스트 파이프라인은 반드시 Swarm/스테이징 클러스터 상에서 수행.

---

## 8. BuildKit의 빌드 타임 시크릿(Compose/Swarm 외 영역)

런타임 Secret과 별개로, **빌드 과정에서만** 필요한 자격증명(프라이빗 레포 토큰 등)은 BuildKit의 `RUN --mount=type=secret`으로 안전하게 전달한다. 이 값은 이미지 레이어에 남지 않는다.

### 8.1 사용 예(Dockerfile)
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

포인트
- 빌드 시크릿은 최종 이미지에 남지 않는다.
- **런타임 시크릿(Docker Secrets)** 과 목적이 다르다: 빌드 vs 실행.

---

## 9. 외부 시크릿 매니저와의 통합 전략

Docker Secrets는 Swarm 내장형이다. 조직에서 이미 **HashiCorp Vault / AWS Secrets Manager / GCP Secret Manager / Azure Key Vault** 를 쓰고 있다면 다음 중 하나를 고른다.

- **앱이 직접 조회**: 시작 시 Vault에서 가져오고, 파일에 기록하지 않으며 메모리 내 보관. 회전도 Vault 정책으로 처리.
- **사이드카/에이전트 패턴**: 사이드카가 주기적으로 Secret을 동기화해 `/run/secrets/...` 경로에 반영(애플리케이션은 파일 기반 유지). 단, 파일 변경 감지/핫 리로드 설계 필요.
- **사전-주입(Init 컨테이너)**: 시작 시 한 번 주입하고 변경 시 롤링 업데이트로 교체.

Vault를 쓰더라도, **컨테이너 내부 경로/권한/로그 노출 금지** 원칙은 동일하다.

---

## 10. 보안/감사 관점 체크리스트

1) **권한 최소화**: Secret은 해당 서비스에만 연결. 재사용 금지.
2) **로그 금지**: Secret 내용을 로깅하지 않도록 프레임워크 설정(ORM/에러 리포터).
3) **코어 덤프 방지**: 비밀번호를 문자열 상수/전역에 오래 보관하지 말고, 가능한 한 **짧은 수명**으로 다룬다.
4) **백업 범위 점검**: Secret 원본이 저장된 경로(예: `./secrets/*.txt`)가 백업/스토리지에 들어가면 무력화. 운영/개발 분리.
5) **감사 추적**: Secret 버전/생성자/배포 릴리즈/회전 이력을 문서화.
6) **E2E 암호화**: Swarm TLS를 기본 유지, 매니저 노드 보호.

---

## 11. 자주 겪는 오류와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| 컨테이너에 `/run/secrets/...` 가 없음 | Swarm 아닌 `docker run` 사용 | `docker swarm init` 후 `docker service` 또는 `docker stack deploy` 사용 |
| Permission denied | 사용자 권한/UID mismatch | 컨테이너 `USER`가 읽기 가능한지 확인(읽기전용 파일) |
| DB 접속 실패(비밀번호 무시) | 앱이 환경변수 경로만 참조 | `*_FILE` 패턴 또는 Secret 파일 직접 읽도록 수정 |
| 회전 후 일부 인스턴스가 구 값 사용 | 롤링 중 경로 혼합 | 이중 마운트 패턴(신/구) + 우선순위 전략, 종료 후 구 Secret 제거 |
| CI 아티팩트에 비밀 노출 | 테스트 로그/스크린샷 | 비밀번호/토큰을 마스킹, 비밀 내용 출력 금지 규칙 적용 |

---

## 12. 종합 예제: API + PostgreSQL + Nginx(TLS) 스택

### 12.1 디렉터리
```
stack/
├── docker-compose.yml
└── secrets/
    ├── db_password.txt
    ├── tls.crt
    └── tls.key
```

### 12.2 Compose(스택) 파일
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
    # nginx.conf에서 ssl_certificate/key 경로를 /run/secrets/tls_crt|tls_key 로 사용

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
docker swarm init           # 최초 1회
docker stack deploy -c docker-compose.yml mystack
```

---

## 13. 성능/운영 팁

- Secret 파일은 작은 텍스트(몇 KB 수준)가 일반적이므로 성능 부담은 미미하다.
- Secret 갯수가 많다면 **명명 규칙**과 **서비스 단위 묶음**으로 관리 복잡도 최소화.
- 앱이 Secret 파일을 **런타임에 반복해서 읽을 필요는 없다**. 시작 시 읽고 커넥션을 유지하되, 회전 이벤트를 감지하는 경우에만 재로딩.

---

## 14. 결론

- Docker Secrets는 Swarm 환경에서 **간결하고 안전한 런타임 비밀 전달**을 제공한다.
- 핵심 가치는 **환경변수/로그/레이어 비노출**과 **암호화된 전달**에 있다.
- 운영 관점에서는 **버저닝·무중단 교체 패턴·감사/백업 경로 관리**가 성공의 열쇠다.
- 빌드 단계는 **BuildKit 시크릿**, 멀티 클라우드/엔터프라이즈는 **외부 시크릿 매니저**와의 하이브리드 구성이 실용적이다.

---

## 부록 A. 명령어 요약

```bash
# Swarm
docker swarm init

# 생성/조회/삭제
echo "secret" | docker secret create mysecret -
docker secret ls
docker secret rm mysecret

# 서비스에 연결
docker service create --name app --secret mysecret myimg:tag

# 스택 배포(Compose v3+)
docker stack deploy -c docker-compose.yml mystack
```

---

## 부록 B. 위험도 계산의 간단 모델

관리 기준을 정량화하려면 취약점/노출 경로/감사 요구사항을 반영한 임계치 \(R_{\text{max}}\)를 두고, 런타임 비밀의 노출 위험 \(R\)이 이를 넘으면 배포를 차단한다:

$$
R = \alpha \cdot \mathbb{1}\{\text{환경변수로 비밀 주입}\}
+ \beta \cdot \mathbb{1}\{\text{파일 영구 저장}\}
+ \gamma \cdot \mathbb{1}\{\text{로그/백업 경로 포함}\}
$$

Docker Secrets 도입의 목표는 각 항목의 **표시함수**(indicator)를 0으로 만드는 것이다.

---

## 참고 문서

- Docker Engine Swarm Secrets
- Compose v3 Secrets
- Secrets 보안 팁(공식 모범사례)
- BuildKit Secrets(`RUN --mount=type=secret`)

(공식 문서는 배포 환경과 버전에 따라 세부 옵션이 다를 수 있으므로 실제 배포 전에 로컬/스테이징에서 반드시 검증할 것.)
