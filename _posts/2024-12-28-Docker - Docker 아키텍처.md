---
layout: post
title: Docker - Docker 아키텍처
date: 2024-12-28 21:20:23 +0900
category: Docker
---
# Docker 아키텍처: 이미지, 컨테이너, 레지스트리, 데몬

## 개요

Docker는 컨테이너 기반의 애플리케이션 배포 플랫폼으로, 이미지, 컨테이너, 레지스트리, 데몬이라는 네 가지 핵심 구성 요소를 중심으로 동작합니다. 각 요소는 서로 연계되어 효율적인 애플리케이션 패키징, 배포, 실행을 가능하게 합니다.

---

## 이미지(Image)

이미지는 애플리케이션 실행에 필요한 모든 것을 포함한 불변 템플릿입니다. 여러 레이어로 구성되어 있으며, 각 레이어는 파일 시스템의 변경 사항을 나타냅니다.

### 레이어 구조와 캐싱

Docker 이미지는 여러 개의 읽기 전용 레이어로 구성됩니다. 이러한 레이어 구조는 효율적인 캐싱을 가능하게 합니다.

```Dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

이 Dockerfile에서 각 명령어는 새로운 레이어를 생성합니다. `package*.json` 파일을 먼저 복사하고 `npm ci`를 실행함으로써, 패키지 파일이 변경되지 않는 한 이 레이어는 캐시되어 재사용됩니다.

### 이미지 확인 명령어

```bash
# 이미지 빌드 히스토리 확인
docker history demo-web:1

# 이미지 레이어 구조 확인
docker image inspect demo-web:1 | jq '.[0].RootFS'
```

### 태그와 다이제스트

이미지는 태그와 다이제스트로 식별됩니다. 태그는 사람이 읽기 쉬운 이름(예: `nginx:1.27-alpine`)이고, 다이제스트는 이미지 내용의 암호화 해시값입니다. 재현 가능한 배포를 위해서는 다이제스트를 사용하는 것이 권장됩니다.

{% raw %}
```bash
# 이미지 다이제스트 확인
docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine
```
{% endraw %}

### 보안 검사

이미지 보안을 위해 취약점 스캔과 서명 검증을 수행할 수 있습니다.

```bash
# 취약점 스캔
trivy image --severity HIGH,CRITICAL demo-web:1

# 이미지 서명 및 검증
cosign sign ghcr.io/owner/repo/app:1.0
cosign verify ghcr.io/owner/repo/app:1.0
```

---

## 컨테이너(Container)

컨테이너는 이미지를 실행한 인스턴스로, 리눅스 커널의 격리 기능(네임스페이스, cgroups)을 활용하여 독립적인 실행 환경을 제공합니다.

### 기본 실행

```bash
# 상호작용 모드로 컨테이너 실행
docker run --rm -it --name u1 ubuntu:22.04 bash

# 컨테이너 내부에서 시스템 정보 확인
ps -ef
hostname
```

### 데이터 지속성

컨테이너는 일시적이므로 중요한 데이터는 볼륨을 사용하여 지속시켜야 합니다.

```bash
# 볼륨 생성 및 사용
docker volume create data1
docker run -d --name pg -e POSTGRES_PASSWORD=pw \
  -v data1:/var/lib/postgresql/data postgres:16-alpine
```

### 리소스 제한과 보안

```bash
# CPU, 메모리 제한 및 보안 옵션 적용
docker run --rm \
  --cpus=1 --memory=512m \
  --read-only \
  --pids-limit=256 \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  alpine:3.20 sh -c 'touch /tmp/x && echo ok'
```

### 네트워킹

컨테이너 간 네트워크 통신을 위해 사용자 정의 네트워크를 생성할 수 있습니다.

```bash
# 네트워크 생성 및 컨테이너 연결
docker network create appnet
docker run -d --name web --network appnet -p 8080:80 nginx:alpine
docker run --rm --network appnet curlimages/curl http://web
```

---

## 레지스트리(Registry)

레지스트리는 Docker 이미지를 저장하고 공유하는 중앙 저장소입니다.

### 레지스트리 종류

- 공용 레지스트리: Docker Hub, GitHub Container Registry(GHCR), Quay
- 사설 레지스트리: Harbor, JFrog Artifactory, registry:2

### 이미지 푸시와 풀

```bash
# 레지스트리 로그인
docker login ghcr.io -u USER

# 이미지 태깅 및 푸시
docker tag demo-web:1 ghcr.io/USER/demo-web:1
docker push ghcr.io/USER/demo-web:1

# 이미지 풀
docker pull ghcr.io/USER/demo-web:1
```

### 사설 레지스트리 설정

개발 환경에서 사설 레지스트리를 쉽게 구성할 수 있습니다.

```bash
# 로컬 레지스트리 실행
docker run -d --restart=always --name reg -p 5000:5000 registry:2

# 이미지 태깅 및 푸시
docker tag nginx:alpine localhost:5000/nginx:alpine
docker push localhost:5000/nginx:alpine
```

---

## 데몬(Docker Daemon)

Docker 데몬(dockerd)은 Docker API 요청을 처리하고 컨테이너 런타임을 관리하는 백그라운드 서비스입니다.

### Docker API 사용

```bash
# Docker 소켓을 통한 API 호출
curl --unix-socket /var/run/docker.sock http://localhost/info | jq '.ServerVersion,.Driver'
```

운영 환경에서는 보안을 위해 소켓 접근 권한을 엄격히 제어하거나, TCP API와 mTLS를 사용하는 것이 좋습니다.

### BuildKit 활용

BuildKit은 향상된 빌드 기능을 제공하는 빌드 시스템입니다.

```bash
# BuildKit 활성화
export DOCKER_BUILDKIT=1
docker build -t demo-buildkit:1 .
```

### 빌드 시 시크릿 사용

BuildKit을 사용하면 빌드 시 안전하게 시크릿을 전달할 수 있습니다.

```Dockerfile
# syntax=docker/dockerfile:1.7

FROM alpine
RUN --mount=type=secret,id=npmrc \
    cat /run/secrets/npmrc >/dev/null || true
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t demo-secret:1 .
```

---

## 스토리드 드라이버와 파일 시스템

### OverlayFS 원리

리눅스에서 Docker는 주로 OverlayFS를 사용합니다. 이는 여러 읽기 전용 레이어(lowerdir)와 하나의 쓰기 가능 레이어(upperdir)를 병합(merged)하여 단일 파일 시스템으로 제공합니다.

```bash
# 컨테이너 파일 시스템 관찰
docker run --name ov -d alpine:3.20 sleep 9999
docker exec ov sh -c 'echo hi > /etc/hi.txt'
docker inspect ov | jq '.[0].GraphDriver'
```

---

## 실전 시나리오

### Docker Compose를 이용한 다중 컨테이너 애플리케이션

```yaml
# docker-compose.yaml
services:
  web:
    image: nginx:alpine
    volumes: ["./site:/usr/share/nginx/html:ro"]
    ports: ["8080:80"]
    depends_on: ["api"]

  api:
    build: ./api
    environment:
      - DB_HOST=db
    ports: ["8081:8080"]
    depends_on: ["db"]

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_PASSWORD=demo
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```bash
# 애플리케이션 실행
docker compose up -d --build
curl http://localhost:8080
curl http://localhost:8081/health
```

### 이미지 배포와 전달

```bash
# 다이제스트를 사용한 정확한 버전 배포
IMG="nginx@sha256:xxxxxxxx"
docker pull "$IMG"
docker run -d -p 8080:80 --name w "$IMG"

# 이미지 저장 및 로드
docker save ghcr.io/user/app:1.0 -o app_1_0.tar
docker load -i app_1_0.tar
```

---

## 빌드 최적화 패턴

### .dockerignore 파일 활용

빌드 컨텍스트를 최소화하여 빌드 속도를 향상시킵니다.

```gitignore
node_modules
.git
dist
*.log
```

### 멀티스테이지 빌드

빌드 도구와 런타임 환경을 분리하여 최종 이미지 크기를 줄입니다.

```Dockerfile
# syntax=docker/dockerfile:1.7

FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build \
    go build -o /out/app ./cmd/app

FROM alpine
COPY --from=build /out/app /app
CMD ["/app"]
```

---

## 보안 모범 사례

Docker 컨테이너 보안을 강화하기 위한 주요 원칙:

1. **최소 권한 원칙**: 필요한 권한만 부여
2. **읽기 전용 파일 시스템**: 불필요한 쓰기 방지
3. **리소스 제한**: CPU, 메모리, 프로세스 수 제한
4. **정기적인 취약점 검사**: 이미지 보안 유지
5. **네트워크 격리**: 불필요한 네트워크 접근 차단

```bash
# 보안 강화된 컨테이너 실행 예시
docker run --rm \
  --read-only --tmpfs /tmp --tmpfs /run \
  --cap-drop ALL --pids-limit 128 \
  --cpus=0.5 --memory=256m \
  nginx:alpine
```

---

## 결론

Docker 아키텍처는 이미지, 컨테이너, 레지스트리, 데몬이라는 네 가지 핵심 요소로 구성되어 있습니다. 이미지는 불변의 애플리케이션 템플릿으로, 레이어 구조를 통해 효율적인 빌드와 배포를 가능하게 합니다. 컨테이너는 이 이미지를 격리된 환경에서 실행하는 인스턴스이며, 레지스트리는 이미지를 저장하고 공유하는 중앙 저장소 역할을 합니다. 데몬은 이러한 모든 요소를 관리하고 조율하는 핵심 서비스입니다.

효율적인 Docker 사용을 위해서는 레이어 캐싱 원리를 이해하고 적절히 활용해야 합니다. 또한 보안을 위해 최소 권한 원칙을 준수하고, 재현 가능한 배포를 위해 다이제스트를 사용하는 것이 중요합니다. 현대적인 애플리케이션 개발에서는 멀티스테이지 빌드를 통해 이미지 크기를 최소화하고, BuildKit을 활용하여 빌드 프로세스를 최적화하는 것이 필수적입니다.

Docker는 지속적으로 발전하고 있으며, OCI 표준을 준수함으로써 다양한 런타임과의 호환성을 유지하고 있습니다. 이러한 표준화는 컨테이너 생태계의 성장과 발전을 촉진하는 중요한 요소입니다.