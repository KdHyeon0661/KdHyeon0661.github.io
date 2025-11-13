---
layout: post
title: Docker - Docker Compose에서 볼륨과 네트워크 설정
date: 2025-02-08 20:20:23 +0900
category: Docker
---
# Docker Compose에서 볼륨과 네트워크 설정 완전 정복

## 0. 빠른 개념 지도

- **볼륨(Volumes)**: 컨테이너 수명과 분리된 **데이터 영속화/공유 레이어**
  - 종류: **Named Volume**, **Bind Mount**, (tmpfs 포함)
  - 목적: DB/업로드/캐시 보존, 코드 핫리로드(개발), 로그 수집 등

- **네트워크(Networks)**: 컨테이너 간 통신을 제어하는 **가상 L2/L3**
  - 기본: `bridge` (단일 호스트)
  - 고급: `overlay`(다중 호스트), `macvlan`(실 IP 부여), `host`, `none`
  - Compose가 **내부 DNS**를 제공 → 같은 네트워크 내 **서비스명으로 통신**

---

## 1. 볼륨(Volumes)

### 1.1 볼륨의 3가지 유형

| 유형 | 선언/예시 | 특징/용도 |
|---|---|---|
| **Named Volume** | `volumes: { db-data: {} }` → `- db-data:/var/lib/mysql` | Docker가 관리하는 **이름 있는 저장소**. 경로 불투명하지만 이식성·백업 쉬움. **DB·영구데이터 권장** |
| **Bind Mount** | `- ./src:/app` | **호스트의 실제 경로**를 그대로 마운트. 개발 시 **핫리로드** 최적. 운영에서는 권한/보안 주의 |
| **tmpfs** | `- type: tmpfs, target: /run/cache` (long syntax) | 메모리 기반(비영구). 고성능 임시 캐시, 비밀값 임시 파일 |

> Anonymous volume(익명 볼륨)은 `- /path/in/container` 형태로 생성되며 **이름이 없어** 관리가 번거롭습니다. 재현성 측면에서 **Named Volume**을 권장합니다.

---

### 1.2 Compose에서의 선언 위치와 의미

- **서비스 섹션 내부의 `volumes:`** → “어디를 마운트할지”를 지정
- **루트 레벨의 `volumes:`** → “Named Volume을 정의” (드라이버/옵션 포함)

```yaml
services:
  db:
    image: mysql:8
    volumes:
      - db-data:/var/lib/mysql    # 마운트(서비스 내부)

volumes:
  db-data:                         # 정의(루트 레벨)
```

---

### 1.3 단문/장문 문법 비교 (short vs long syntax)

**Short syntax** *(간단, 빠른)*

```yaml
services:
  app:
    volumes:
      - ./src:/app:rw
      - logs:/var/log/app:ro
```

**Long syntax** *(세부 옵션 제어)*

```yaml
services:
  app:
    volumes:
      - type: bind
        source: ./src
        target: /app
        read_only: false
        consistency: delegated      # Docker Desktop(macOS) 캐시 힌트
      - type: volume
        source: logs
        target: /var/log/app
        read_only: true
        volume:
          nocopy: true              # 기존 컨테이너 파일을 볼륨에 복사하지 않음
          labels:
            owner: "platform"
```

> macOS/Windows에서의 `:cached`/`:delegated`/`consistency` 힌트는 파일 동기화 성능에 영향을 줍니다(Desktop 한정).

---

### 1.4 Bind Mount 운영체제별 주의점

| OS | 경로 표기 | 비고 |
|---|---|---|
| Linux | `./data:/app/data` | SELinux 활성화 시 `:z`/`:Z` 옵션 고려 |
| macOS | `./data:/app/data` | Docker Desktop 파일 I/O 성능 고려, `delegated` 권장(케이스별) |
| Windows | `C:\path\to\data:/app/data` 또는 `./data:/app/data` | 줄바꿈(CRLF)·권한 이슈 유의 |

**SELinux**:
- `:z` → 여러 컨테이너가 공유 가능한 레이블
- `:Z` → 단일 컨테이너 전용 레이블
예) `- ./data:/var/lib/mysql:Z`

---

### 1.5 읽기전용/전파/권한

- 읽기전용: `- ./config:/app/config:ro` 또는 `read_only: true`
- 마운트 전파(propagation): 고급 시나리오(호스트-컨테이너 중첩 마운트). 필요 시 long syntax의 `bind.propagation: rshared` 등.

---

### 1.6 데이터베이스 표준 패턴 (Named Volume 권장)

```yaml
version: '3.9'
services:
  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 20
      start_period: 10s

volumes:
  db-data:
```

- 컨테이너 재생성에도 데이터 유지
- 백업: `docker run --rm -v db-data:/var/lib/postgresql/data busybox tar czf - /var/lib/postgresql/data > backup.tgz`

---

### 1.7 개발용 핫리로드 패턴 (Bind Mount 권장)

```yaml
services:
  web:
    build: .
    volumes:
      - ./src:/app
    command: flask run --host=0.0.0.0 --reload
```

- 코드 변경 → 즉시 반영
- 운영 전환 시 **Bind Mount 제거**하고 이미지로만 배포

---

### 1.8 tmpfs (메모리) 마운트

```yaml
services:
  api:
    image: my/api
    volumes:
      - type: tmpfs
        target: /run/cache
        tmpfs:
          size: 268435456   # 256MiB
```

- 민감 정보/고성능 임시 캐시 용도에 유용
- **컨테이너 종료 시 내용 소멸**

---

### 1.9 Volume Driver/옵션(외부 스토리지)

```yaml
volumes:
  nfs-share:
    driver: local
    driver_opts:
      type: "nfs"
      o: "addr=10.0.0.1,rw,nolock,soft"
      device: ":/exports/share"
```

- NFS/GlusterFS/ceph 등과 연계 가능
- Docker Desktop/호스트 네트워크/방화벽 설정과 함께 테스트 필수

---

### 1.10 운영 팁

- **이름 있는 볼륨**을 선호(백업/이식성)
- 개발/운영 **오버라이드**: `docker-compose.override.yml`에서 bind mount만 추가
- **권한/소유자** 문제는 컨테이너의 `UID/GID`와 볼륨 파일의 소유권 매칭으로 해소
- 불필요 볼륨 정리: `docker volume prune` (신중히)

---

## 2. 네트워크(Networks)

### 2.1 기본 — bridge + 서비스명 DNS

```yaml
version: '3.9'
services:
  web:
    image: nginx
    networks: [ frontend ]
  app:
    build: .
    networks: [ frontend, backend ]
  db:
    image: mysql:8
    networks: [ backend ]

networks:
  frontend:
  backend:
```

- `web ↔ app`(frontend), `app ↔ db`(backend) 통신 가능
- `web ↔ db`는 **격리** (보안·책임분리)

컨테이너 내부 통신 예:
```bash
# app 컨테이너 쉘에서
curl http://web
mysql -h db -u root -p
```

> Compose는 같은 네트워크에 있는 서비스들을 **내부 DNS**에 등록합니다. → **IP 대신 서비스명** 사용 권장.

---

### 2.2 드라이버 지정/고급 옵션

```yaml
networks:
  frontend:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.enable_icc: "true"   # 컨테이너 간 통신 허용
      com.docker.network.bridge.enable_ip_masquerade: "true"
    ipam:
      driver: default
      config:
        - subnet: 172.22.0.0/24
          gateway: 172.22.0.1
```

- **IPAM**으로 서브넷/게이트웨이를 고정하면 IP 충돌을 피하고, 디버깅이 쉬워집니다.
- `enable_icc=false` 등으로 컨테이너 간 통신 제한도 가능(고립 강화).

---

### 2.3 외부 네트워크 사용

공유 리버스 프록시(예: `nginx-proxy`) 네트워크를 재사용:

```yaml
services:
  web:
    image: my/web
    networks:
      - default
      - proxy-net

networks:
  proxy-net:
    external: true
```

> 사전 생성: `docker network create proxy-net`

---

### 2.4 네트워크 앨리어스(alias)와 고정 IP(테스트 한정)

```yaml
services:
  svc:
    image: my/svc
    networks:
      backend:
        aliases: [ users, accounts ]

networks:
  backend:
    ipam:
      config:
        - subnet: 172.30.0.0/24
```

> 고정 IP는 **재현성/충돌** 이슈로 일반적으로 **권장되지 않으며**, 테스트/레거시 요구에서만 제한적으로 사용하세요.

---

### 2.5 network_mode 특수값

```yaml
services:
  fastgw:
    image: my/gateway
    network_mode: host   # 호스트 네트워크(포트매핑 무시, 성능 우선)
```

- `host`: NAT/포워딩 오버헤드 없음(리눅스 한정)
- `service:<name>`/`container:<id>`: 동일 네트 스택 공유(디버깅/사이드카 패턴)
- `none`: 네트워크 격리 최대화

---

### 2.6 Overlay/macvlan 요약(Compose 관점)

- `overlay`: 다중 호스트 간 네트워크(Swarm/Orchestrator), Compose 단독으로는 **제약**
- `macvlan`: 컨테이너에 **실 IP** 부여(스위치/라우터에서 호스트처럼 보임). 호스트↔컨테이너 통신 제약/보안정책 고려.

---

## 3. 종합 실전 예시: WordPress + MySQL (네트워크 분리, 볼륨/헬스체크/보안)

초안 예시를 보강해, **네트워크 분리**·**영속화**·**헬스체크**·**기동 순서 안정화**를 반영합니다.

```yaml
version: '3.9'

services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: wp
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    volumes:
      - db-data:/var/lib/mysql
    networks: [ backend ]
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -u root -p$${MYSQL_ROOT_PASSWORD} --silent"]
      interval: 5s
      retries: 20
      start_period: 10s

  wordpress:
    image: wordpress:php8.2-apache
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: wp
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
    volumes:
      - wp-data:/var/www/html
    depends_on:
      db:
        condition: service_healthy
    networks: [ frontend, backend ]

networks:
  frontend:
  backend:

volumes:
  wp-data:
  db-data:
```

포인트:
- **두 네트워크(frontend/backend) 분리**로 외부 노출 경로 최소화
- DB는 **Named Volume**으로 영속화
- `healthcheck` + `depends_on.condition` 으로 **안정적 기동 순서**

---

## 4. 개발/운영 분리: `.env` + `docker-compose.override.yml`

`.env` (공통/개발 변수만)
```env
APP_ENV=dev
HTTP_PORT=8080
```

`docker-compose.yml` (운영 기본)
```yaml
services:
  web:
    image: my/web:1.0.0
    ports: ["${HTTP_PORT:-8080}:80"]
    networks: [ frontend ]
```

`docker-compose.override.yml` (개발 전용 자동 병합)
```yaml
services:
  web:
    build: .
    volumes:
      - ./src:/app
    environment:
      APP_ENV: ${APP_ENV}
```

> 운영에서는 override 없이 `compose up -d` (이미지 고정), 개발에서는 override로 **Bind Mount** 적용.

---

## 5. 보안·운영 체크리스트

- **불필요한 포트 노출 금지**: DB 컨테이너의 `ports:` 제거(내부 네트워크만)
- **비밀값 관리**: `.env` 최소화, CI/CD 비밀관리(Secrets Manager/Vault/Parameter Store)
- **권한/사용자**: 컨테이너를 **비루트 사용자**로 실행 (`USER`), 볼륨 소유권 정합 유지
- **네트워크 분리**: 프런트/백엔드/DB 네트워크 분리로 blast radius 축소
- **로그 회전/수집**: json-file 옵션(`max-size`, `max-file`) 또는 로깅 드라이버
- **백업/복구**: Named Volume 백업 파이프라인 + 주기적 리스토어 리허설

로그 예시:
```yaml
services:
  web:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

---

## 6. 트러블슈팅

| 현상 | 가능 원인 | 해결 |
|---|---|---|
| 컨테이너 간 접속 실패 | 같은 네트워크가 아님 | `networks:` 확인, `network inspect`로 연결 상태 점검 |
| 외부 접속 불가 | 포트 매핑 누락 | `ports:` 추가, 방화벽/보안그룹 확인 |
| 파일 권한 오류 | 호스트 UID/GID ≠ 컨테이너 UID | 볼륨 소유 변경(`chown`), 컨테이너 `USER` 맞춤 |
| DB 초기화 스크립트 재실행 안 됨 | entrypoint-init은 최초 1회 | 새 볼륨으로 재기동 혹은 수동 적용 |
| macOS에서 느린 I/O | Desktop 파일 동기화 오버헤드 | `:delegated`/`:cached` 힌트, Bind Mount 최소화 |
| SELinux로 마운트 실패 | 컨텍스트 불일치 | `:z`/`:Z` 옵션 적용 |

유용 명령:
{% raw %}
```bash
# 네트워크/볼륨 현황
docker network ls
docker network inspect <net>
docker volume ls
docker volume inspect <vol>

# 컨테이너별 네트워크/마운트 바인딩 확인
docker inspect <container> --format '{{json .NetworkSettings.Networks}}'
docker inspect <container> --format '{{json .Mounts}}' | jq
```
{% endraw %}

---

## 7. 고급 토픽 모음

- **서비스별 read-only 루트 FS**:
  ```yaml
  services:
    api:
      image: my/api
      read_only: true
      tmpfs: ["/tmp"]
  ```
- **프라이빗 레지스트리/프록시 네트워크**: 외부 `proxy-net` 공유
- **IPAM/고정 IP(테스트)**: IP 계획이 필요한 레거시 상호연동 환경에서만 제한 사용
- **overlay/macvlan**: Compose 단독보다는 Swarm/K8s와 함께 고려

---

## 8. 최소/확장 예제 요약

### 8.1 최소(초안 스타일 유지)

```yaml
version: '3.9'

services:
  app:
    build: .
    volumes:
      - ./src:/app
    networks: [ app-net ]
    ports:
      - "8080:80"

  db:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data
    networks: [ app-net ]

networks:
  app-net:

volumes:
  db-data:
```

### 8.2 확장(분리/보안/헬스체크)

```yaml
version: '3.9'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes: [ "db-data:/var/lib/postgresql/data" ]
    networks: [ backend ]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 20

  api:
    image: my/api:1.0.0
    depends_on:
      db:
        condition: service_healthy
    networks: [ frontend, backend ]
    ports: [ "8080:8080" ]
    read_only: true
    tmpfs: [ "/tmp" ]
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

networks:
  frontend:
  backend:

volumes:
  db-data:
```

---

## 9. 핵심 정리표

### 9.1 볼륨

| 목적 | 권장 |
|---|---|
| DB/영구데이터 | **Named Volume** |
| 개발 핫리로드 | **Bind Mount** |
| 비영구 고성능 임시 | **tmpfs** |
| 외부 스토리지 | volume driver(NFS/ceph 등) |

### 9.2 네트워크

| 시나리오 | 권장 |
|---|---|
| 단일 호스트, 표준 | `bridge` + 서비스명 DNS |
| 외부 공개 최소화 | 네트워크 분리(frontend/backend) |
| 고성능 게이트웨이 | `host`(리눅스/주의) |
| 다중 호스트 | `overlay`(Swarm/Orchestrator) |
| 실 IP 필요 | `macvlan`(제약/보안 검토) |

---

## 10. 마무리

- **Compose는 선언형**입니다. 재현 가능·문서화 가능한 방식으로 **볼륨/네트워크**를 정의하세요.
- **개발/운영 분리**(Bind Mount ↔ Named Volume, 포트/로그/비밀)로 수명주기 전환을 쉽게.
- **네트워크 분리**와 **권한 최소화**로 기본 보안을 잡으면, 이후 확장은 훨씬 수월합니다.
