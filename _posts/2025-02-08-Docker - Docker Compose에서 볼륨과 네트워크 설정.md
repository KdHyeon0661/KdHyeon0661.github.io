---
layout: post
title: Docker - Docker Compose에서 볼륨과 네트워크 설정
date: 2025-02-08 20:20:23 +0900
category: Docker
---
# Docker Compose에서 볼륨과 네트워크 설정 완전 정복

Docker Compose를 효과적으로 사용하려면 볼륨과 네트워크 설정에 대한 깊은 이해가 필수적입니다. 이 가이드에서는 Docker Compose 환경에서 데이터 영속화와 컨테이너 통신을 관리하는 방법을 체계적으로 설명합니다.

## Docker Compose 볼륨 관리의 핵심 개념

볼륨은 컨테이너의 수명 주기와 독립적으로 데이터를 유지하는 메커니즘입니다. 데이터베이스 파일, 업로드된 콘텐츠, 애플리케이션 로그 등 영구적으로 보존해야 하는 데이터를 관리할 때 사용됩니다.

### 주요 볼륨 유형과 활용 시나리오

**네임드 볼륨 (Named Volumes)**
Docker가 관리하는 독립적인 저장 공간으로, 호스트 파일 시스템의 특정 경로와 직접적으로 연결되지 않습니다. 데이터베이스 저장소나 애플리케이션의 영구 데이터를 저장하는 데 가장 적합합니다.

**바인드 마운트 (Bind Mounts)**
호스트 시스템의 특정 디렉토리를 컨테이너에 직접 마운트하는 방식입니다. 개발 환경에서 소스 코드 변경을 실시간으로 반영해야 할 때 특히 유용하지만, 운영 환경에서는 보안과 이식성 측면에서 주의가 필요합니다.

**임시 파일 시스템 (tmpfs)**
메모리 기반의 임시 저장 공간으로, 컨테이너가 종료되면 데이터가 자동으로 삭제됩니다. 고성능이 요구되는 캐시나 일시적인 작업 파일을 저장하는 데 적합합니다.

### Docker Compose에서의 볼륨 정의와 사용

볼륨 설정은 Docker Compose 파일 내에서 두 가지 수준에서 정의됩니다:

```yaml
services:
  database:
    image: postgres:15
    volumes:
      - postgres-data:/var/lib/postgresql/data  # 서비스 수준: 마운트 지점 정의

volumes:
  postgres-data:  # 루트 수준: 볼륨 자체 정의
    driver: local
```

이 구조는 볼륨의 정의와 사용을 명확히 분리하여, 동일한 볼륨을 여러 서비스에서 공유할 수 있게 합니다.

### 단축 문법과 확장 문법 비교

**단축 문법 (Short Syntax)**
간결하고 읽기 쉬운 형식으로 빠른 설정에 적합합니다:

```yaml
volumes:
  - ./config:/app/config:ro
  - app-logs:/var/log/app
```

**확장 문법 (Long Syntax)**
상세한 제어가 필요한 경우 사용합니다:

```yaml
volumes:
  - type: bind
    source: ./config
    target: /app/config
    read_only: true
    bind:
      propagation: rshared
```

확장 문법은 읽기 전용 설정, 바인드 전파 모드, 볼륨 레이블 등 고급 옵션을 지정할 수 있어 복잡한 시나리오에 적합합니다.

### 운영체제별 바인드 마운트 고려사항

각 운영체제는 파일 시스템과 권한 관리에서 고유한 특성을 가지고 있어, 바인드 마운트 사용 시 이를 고려해야 합니다.

**Linux 시스템**
SELinux가 활성화된 환경에서는 보안 컨텍스트 레이블을 적절히 설정해야 합니다:
- `:z` 옵션: 여러 컨테이너가 공유할 수 있는 레이블
- `:Z` 옵션: 단일 컨테이너 전용 레이블

**macOS 시스템**
Docker Desktop의 파일 시스템 가상화로 인한 성능 문제가 발생할 수 있습니다. `:delegated` 옵션을 사용하면 성능을 개선할 수 있습니다.

**Windows 시스템**
경로 구분자와 줄바꿈 문자(CRLF vs LF) 차이로 인한 문제가 발생할 수 있으므로, 특히 스크립트 파일을 다룰 때 주의가 필요합니다.

### 데이터베이스 애플리케이션을 위한 표준 패턴

데이터베이스 컨테이너는 네임드 볼륨을 사용하는 것이 가장 안전하고 효율적인 방법입니다:

```yaml
version: '3.9'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secure_password
      POSTGRES_DB: app_database
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks:
      - database-network

volumes:
  postgres-data:
    driver: local

networks:
  database-network:
    driver: bridge
```

이 패턴은 데이터 영속성, 상태 모니터링, 네트워크 격리를 모두 고려한 완전한 구성을 제공합니다.

### 개발 환경을 위한 핫 리로드 패턴

개발 과정에서는 소스 코드 변경을 실시간으로 반영하는 것이 중요합니다:

```yaml
services:
  webapp:
    build: .
    volumes:
      - ./src:/app/src
      - ./config:/app/config:ro
    command: python app.py --reload
    environment:
      FLASK_ENV: development
      DEBUG: "true"
```

바인드 마운트를 사용하면 호스트의 소스 코드 변경이 컨테이너 내부에 즉시 반영되어 개발 효율성을 크게 향상시킵니다.

### 고급 볼륨 구성: 외부 스토리지 통합

프로덕션 환경에서는 NFS, GlusterFS, Ceph와 같은 외부 스토리지 시스템과의 통합이 필요할 수 있습니다:

```yaml
volumes:
  shared-storage:
    driver: local
    driver_opts:
      type: "nfs"
      o: "addr=10.0.0.100,rw,nolock,soft,timeo=180"
      device: ":/exports/shared-data"
```

이러한 구성은 스케일 아웃 환경이나 고가용성이 요구되는 시나리오에서 필수적입니다.

## Docker Compose 네트워크 관리

네트워크 구성은 컨테이너 간 통신과 외부 접근을 제어하는 핵심 요소입니다.

### 기본 네트워크 전략: 브리지 네트워크와 DNS

Docker Compose는 자체적인 DNS 서비스를 제공하여, 같은 네트워크 내의 컨테이너들이 서비스 이름으로 서로를 찾을 수 있게 합니다:

```yaml
version: '3.9'

services:
  webserver:
    image: nginx:alpine
    networks:
      - frontend-network
    ports:
      - "80:80"

  application:
    build: ./app
    networks:
      - frontend-network
      - backend-network
    depends_on:
      - database

  database:
    image: postgres:15
    networks:
      - backend-network
    environment:
      POSTGRES_PASSWORD: secret

networks:
  frontend-network:
    driver: bridge
  backend-network:
    driver: bridge
```

이 구성에서:
- `webserver`와 `application`은 `frontend-network`를 통해 통신합니다
- `application`과 `database`는 `backend-network`를 통해 통신합니다
- `webserver`는 `database`와 직접 통신할 수 없어 보안이 강화됩니다

컨테이너 내부에서는 서비스 이름을 사용하여 통신할 수 있습니다:

```bash
# application 컨테이너 내부에서
curl http://webserver
psql -h database -U postgres
```

### 네트워크 드라이버와 IPAM 구성

더 세밀한 네트워크 제어가 필요할 때는 드라이버 옵션과 IPAM(IP Address Management) 설정을 사용할 수 있습니다:

```yaml
networks:
  isolated-network:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: "custom-bridge"
      com.docker.network.bridge.enable_ip_masquerade: "true"
    ipam:
      driver: default
      config:
        - subnet: "172.20.0.0/24"
          gateway: "172.20.0.1"
          ip_range: "172.20.0.128/25"
```

고정 IP 주소와 서브넷을 지정하면 네트워크 구성을 예측 가능하게 만들고, 디버깅과 모니터링을 용이하게 합니다.

### 외부 네트워크 통합

기존의 Docker 네트워크를 Compose 서비스와 통합해야 하는 경우가 있습니다:

```yaml
services:
  reverse-proxy:
    image: nginx:alpine
    networks:
      - default
      - public-network

networks:
  public-network:
    external: true
    name: existing-public-network
```

이 패턴은 리버스 프록시나 모니터링 에이전트와 같은 공유 인프라 컴포넌트와의 통합에 유용합니다.

### 특수 네트워크 모드

특정 시나리오에서는 표준 브리지 네트워크 대신 특수한 네트워크 모드를 사용해야 할 수 있습니다:

```yaml
services:
  network-monitor:
    image: network-tools
    network_mode: "host"  # 호스트 네트워크 스택 사용
    # 포트 매핑 없이 호스트의 모든 네트워크 인터페이스에 직접 접근

  sidecar-container:
    image: log-collector
    network_mode: "service:main-app"  # 다른 서비스와 네트워크 스택 공유
    # main-app 서비스와 동일한 네트워크 네임스페이스 사용
```

이러한 모드들은 성능 최적화나 특수한 디버깅 시나리오에 유용하지만, 보안과 격리 측면에서 신중하게 고려해야 합니다.

## 종합 실전 예제: 완전한 애플리케이션 스택

다음은 WordPress와 MySQL을 포함한 완전한 웹 애플리케이션 스택의 예시입니다:

```yaml
version: '3.9'

services:
  # 데이터베이스 서비스
  mysql:
    image: mysql:8.0
    command: 
      - --default-authentication-plugin=mysql_native_password
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql
      - ./mysql/conf.d:/etc/mysql/conf.d:ro
    networks:
      - backend-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${DB_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s
    restart: unless-stopped

  # 웹 애플리케이션 서비스
  wordpress:
    image: wordpress:php8.2-apache
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      WORDPRESS_DB_HOST: mysql:3306
      WORDPRESS_DB_NAME: ${DB_NAME}
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_CONFIG_EXTRA: |
        define('WP_DEBUG', ${WP_DEBUG});
        define('WP_DEBUG_LOG', ${WP_DEBUG_LOG});
    volumes:
      - wordpress-data:/var/www/html
      - ./wordpress/uploads.ini:/usr/local/etc/php/conf.d/uploads.ini:ro
      - ./wordpress/themes:/var/www/html/wp-content/themes:ro
      - ./wordpress/plugins:/var/www/html/wp-content/plugins:ro
    networks:
      - frontend-network
      - backend-network
    ports:
      - "${HTTP_PORT}:80"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/wp-admin/install.php"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # 캐싱 서비스 (선택적)
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - backend-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    restart: unless-stopped

volumes:
  mysql-data:
    driver: local
    driver_opts:
      type: none
      device: ${VOLUME_PATH}/mysql-data
      o: bind
  wordpress-data:
    driver: local
    driver_opts:
      type: none
      device: ${VOLUME_PATH}/wordpress-data
      o: bind
  redis-data:
    driver: local

networks:
  frontend-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
  backend-network:
    driver: bridge
    internal: true  # 외부 접근 차단
    ipam:
      config:
        - subnet: 172.21.0.0/24
```

이 구성의 주요 특징:
1. **네트워크 분리**: 프론트엔드와 백엔드 네트워크를 분리하여 보안 강화
2. **상태 기반 의존성**: 데이터베이스가 정상 상태일 때만 WordPress 시작
3. **볼륨 영속성**: 모든 중요 데이터는 네임드 볼륨에 저장
4. **상태 모니터링**: 각 서비스에 헬스체크 구성
5. **환경 변수 사용**: 민감한 정보와 환경별 설정 분리

## 개발과 운영 환경 분리

효율적인 Docker Compose 사용의 핵심은 개발 환경과 운영 환경을 명확히 분리하는 것입니다.

### 환경 변수 파일 (.env)

```env
# 애플리케이션 설정
APP_ENV=development
HTTP_PORT=8080

# 데이터베이스 설정
DB_NAME=wordpress
DB_USER=wpuser
DB_PASSWORD=secure_password_123
DB_ROOT_PASSWORD=root_secure_password

# WordPress 설정
WP_DEBUG=true
WP_DEBUG_LOG=true

# 볼륨 경로
VOLUME_PATH=./data
```

### 기본 Docker Compose 구성 (docker-compose.yml)

```yaml
version: '3.9'

services:
  app:
    image: ${APP_IMAGE:-myapp:latest}
    ports:
      - "${HTTP_PORT:-8080}:80"
    environment:
      NODE_ENV: ${APP_ENV:-production}
      DATABASE_URL: ${DATABASE_URL}
    networks:
      - app-network
    restart: unless-stopped

networks:
  app-network:
    driver: bridge

volumes:
  app-data:
    driver: local
```

### 개발 환경 오버라이드 (docker-compose.override.yml)

```yaml
version: '3.9'

services:
  app:
    build: .
    volumes:
      - ./src:/app/src
      - ./config:/app/config:ro
    environment:
      DEBUG: "true"
      HOT_RELOAD: "true"
    command: npm run dev
    ports:
      - "9229:9229"  # 디버거 포트
```

이 구조를 사용하면:
- 운영 환경: `docker-compose up -d` (이미지 기반 배포)
- 개발 환경: `docker-compose up` (소스 코드 마운트와 디버그 설정 적용)

## 보안 모범 사례

### 최소 권한 원칙 적용

```yaml
services:
  webapp:
    image: nginx:alpine
    user: "1000:1000"  # 비루트 사용자로 실행
    read_only: true  # 루트 파일 시스템 읽기 전용
    tmpfs:
      - /tmp
      - /run
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

### 네트워크 격리 강화

```yaml
networks:
  public-network:
    driver: bridge
    # 기본 설정
  
  private-network:
    driver: bridge
    internal: true  # 외부 연결 차단
    ipam:
      config:
        - subnet: "10.10.0.0/24"
  
  database-network:
    driver: bridge
    enable_ipv6: false  # IPv6 비활성화
    driver_opts:
      com.docker.network.bridge.enable_icc: "false"  # 컨테이너 간 통신 차단
```

### 로깅과 모니터링 구성

```yaml
services:
  application:
    image: myapp:latest
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        labels: "production"
        env: "os,customer"
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

## 문제 해결과 진단

### 일반적인 문제와 해결 방법

**네트워크 연결 문제**
컨테이너 간 통신이 실패하는 경우:
```bash
# 네트워크 상태 확인
docker network ls
docker network inspect <network-name>

# 컨테이너 네트워크 설정 확인
docker inspect <container-name> --format '{{json .NetworkSettings.Networks}}'

# DNS 해결 테스트
docker exec <container-name> nslookup <service-name>
```

**볼륨 권한 문제**
파일 시스템 권한 오류가 발생하는 경우:
```bash
# 볼륨 내용 확인
docker run --rm -v <volume-name>:/data alpine ls -la /data

# 호스트와 컨테이너의 UID/GID 비교
id  # 호스트 사용자
docker exec <container-name> id  # 컨테이너 사용자

# 소유권 변경 (필요 시)
docker run --rm -v <volume-name>:/data alpine chown -R 1000:1000 /data
```

**포트 충돌 문제**
포트 바인딩이 실패하는 경우:
```bash
# 포트 사용 확인
sudo lsof -i :<port-number>
sudo netstat -tulpn | grep :<port-number>

# Docker 데몬 재시작 (극단적인 경우)
sudo systemctl restart docker
```

### 진단을 위한 유틸리티 컨테이너

복잡한 문제 해결을 위한 다목적 진단 컨테이너를 사용할 수 있습니다:

```yaml
services:
  diagnostics:
    image: nicolaka/netshoot
    container_name: diagnostics-tool
    networks:
      - app-network
    command: sleep infinity
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

이 컨테이너 내에서 네트워크 진단, 트래픽 분석, 시스템 모니터링 등을 수행할 수 있습니다.

## 고급 구성 패턴

### 다중 환경 구성 관리

대규모 애플리케이션의 경우 환경별 구성 파일을 체계적으로 관리해야 합니다:

```
project/
├── docker-compose.yml
├── docker-compose.dev.yml
├── docker-compose.staging.yml
├── docker-compose.prod.yml
├── .env.dev
├── .env.staging
└── .env.prod
```

환경별 실행:
```bash
# 개발 환경
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up

# 스테이징 환경
docker-compose -f docker-compose.yml -f docker-compose.staging.yml up

# 프로덕션 환경
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
```

### 확장 가능한 서비스 패턴

마이크로서비스 아키텍처를 위한 서비스 확장 패턴:

```yaml
services:
  api-gateway:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - gateway-network
  
  user-service:
    image: user-service:${TAG:-latest}
    deploy:
      replicas: ${USER_SERVICE_REPLICAS:-2}
    networks:
      - gateway-network
      - user-network
    environment:
      SERVICE_NAME: user-service
  
  order-service:
    image: order-service:${TAG:-latest}
    deploy:
      replicas: ${ORDER_SERVICE_REPLICAS:-3}
    networks:
      - gateway-network
      - order-network
    environment:
      SERVICE_NAME: order-service

networks:
  gateway-network:
    driver: bridge
  user-network:
    driver: bridge
    internal: true
  order-network:
    driver: bridge
    internal: true
```

## 결론

Docker Compose에서의 볼륨과 네트워크 관리는 컨테이너 기반 애플리케이션의 성공적인 운영을 위한 핵심 기술입니다. 이 가이드에서 다룬 주요 원칙과 패턴을 정리하면 다음과 같습니다:

**데이터 관리의 분리 원칙**: 영구 데이터, 개발 소스 코드, 임시 캐시는 각각 네임드 볼륨, 바인드 마운트, tmpfs로 구분하여 관리해야 합니다. 이렇게 함으로써 데이터의 수명 주기와 사용 목적에 맞는 최적의 저장 전략을 구현할 수 있습니다.

**네트워크 설계의 보안 우선 접근**: 서비스 간 통신은 최소 권한 원칙에 따라 구성해야 합니다. 프론트엔드와 백엔드 네트워크를 분리하고, 내부 전용 네트워크를 사용하여 불필요한 노출을 방지하는 것이 중요합니다.

**환경별 구성의 체계적 관리**: 개발, 테스트, 프로덕션 환경의 차이를 Docker Compose 파일과 환경 변수를 통해 명확히 분리하면, 일관성 있는 배포 프로세스와 효율적인 문제 해결이 가능해집니다.

**모니터링과 진단의 사전 준비**: 헬스체크, 로깅 설정, 진단 도구를 사전에 구성하면 운영 중 발생할 수 있는 문제를 빠르게 식별하고 해결할 수 있습니다.

이러한 원칙들을 프로젝트에 적용할 때는 애플리케이션의 특성과 팀의 요구사항을 고려하여 유연하게 조정하는 것이 중요합니다. Docker Compose의 강력한 선언적 구문을 활용하여 재현 가능하고 문서화된 인프라 구성을 만들면, 개발부터 운영까지의 전체 라이프사이클에서 일관성과 효율성을 확보할 수 있습니다.