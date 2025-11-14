---
layout: post
title: Docker - Docker 아키텍처
date: 2024-12-28 21:20:23 +0900
category: Docker
---
# Docker 아키텍처: 이미지, 컨테이너, 레지스트리, 데몬

## 빠른 개요(기존 내용 보강 요약)

- **이미지(Image)**: 불변 템플릿. **레이어**들의 스택이며, **컨텐츠 해시(sha256)** 기반으로 식별됨.
- **컨테이너(Container)**: 이미지 + **쓰기 가능(Overlay) 레이어** + **리눅스 격리(네임스페이스·cgroups)** 로 실행되는 **프로세스**.
- **레지스트리(Registry)**: 이미지를 **push/pull** 하는 원격 저장소. **태그(tag)** 와 **다이제스트(digest)** 로 참조.
- **데몬(dockerd)**: 이미지 빌드/풀/푸시, 컨테이너 생성·실행, 볼륨/네트워크/플러그인 관리. **Docker API(REST)** 로 CLI 요청을 처리.

---

# 이미지(Image)

## 개념(확장)

이미지는 **불변(Immutable)** 이며, **여러 레이어(layer)** 가 **순서대로 합성**된 결과입니다. 각 레이어는 **상위 레이어의 기반이 되는 읽기 전용 스냅샷**이고, 최상단에 컨테이너 실행 시 **쓰기 가능 레이어**(diff)가 더해집니다.

- **Content-Addressable Storage(CAS)**: 실제 레이어 데이터는 **내용 해시(sha256)** 로 주소화되어 **중복 제거**가 자동으로 이루어짐.
- **OCI Image Spec** 준수: 만다토리 구조(매니페스트 → config.json → 레이어 tar)와 **미디어 타입** 표준화.

간단 식(중복이 제거된 유효 크기 추정):
$$
\text{EffectiveImageSize} \approx \sum_{l \in \text{UniqueLayers}} \text{size}(l)
$$

## 레이어와 캐시

### Dockerfile 예제(기존 예의 확장 버전)

```Dockerfile
# 빌드 스테이지(툴 포함)

FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 런타임 스테이지(슬림)

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

- 각 `RUN`, `COPY`, `ADD` 가 **새 레이어**를 만든다고 생각하면 이해가 쉽습니다.
- 캐시 히트 전략:
  - **변경 빈도가 낮은 파일**(의존성 선언)부터 **먼저 COPY** → `npm ci` 레이어 캐시 최대화
  - 대용량/외부 변경이 잦은 파일은 **빨리 COPY 하지 말고** 뒤로 배치

### 레이어 관찰

```bash
docker build -t demo-web:1 .
docker history demo-web:1
docker image inspect demo-web:1 | jq '.[0].RootFS'
```

## 태그(tag)와 다이제스트(digest)

- **태그**: 사람이 읽기 쉬운 별칭. `nginx:1.27-alpine`
- **다이제스트**: **내용 불변 식별자**. `nginx@sha256:...`
- 재현 가능한 배포를 원하면 **다이제스트 고정**을 사용:
{% raw %}
```bash
docker pull nginx:alpine
docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine
# 결과를 Kubernetes/Compose에 고정하면 드리프트 방지

```
{% endraw %}

## SBOM·서명·취약점 스캔(운영 보강)

```bash
# trivy로 취약점 스캔(예: 로컬 이미지)

trivy image --severity HIGH,CRITICAL demo-web:1

# cosign으로 이미지 서명/검증(예시)

cosign sign ghcr.io/owner/repo/app:1.0
cosign verify ghcr.io/owner/repo/app:1.0
```

---

# 컨테이너(Container)

## 개념(보강)

컨테이너는 **이미지 + 쓰기 가능 레이어** 와 **리눅스 커널의 격리 기능**(네임스페이스: pid, mnt, net, ipc, uts, user / cgroups: CPU·메모리·I/O 제한)으로 **프로세스 단위 실행 환경**을 제공합니다.

```bash
# 상호작용 셸

docker run --rm -it --name u1 ubuntu:22.04 bash
# 컨테이너 내부에서 프로세스·네임스페이스 확인

ps -ef
hostname
```

## 일시성(Ephemeral)과 상태 관리

- 컨테이너는 **일시적**이며 종료 시 쓰기 레이어가 사라질 수 있습니다.
- 데이터는 **볼륨(Volume)** 으로 분리/영속화:
```bash
docker volume create data1
docker run -d --name pg -e POSTGRES_PASSWORD=pw \
  -v data1:/var/lib/postgresql/data postgres:16-alpine
```

## 리소스 격리와 보안 옵션

```bash
# CPU/메모리 제한, 읽기 전용 루트 FS, 권한 최소화

docker run --rm \
  --cpus=1 --memory=512m \
  --read-only \
  --pids-limit=256 \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  alpine:3.20 sh -c 'touch /tmp/x && echo ok'
```

- **User 네임스페이스/Rootless 모드**: 호스트 root와 분리된 UID/GID 매핑으로 보안성 향상.

## 네트워킹 기초

```bash
docker network ls
docker network create appnet
docker run -d --name web --network appnet -p 8080:80 nginx:alpine
docker run --rm --network appnet curlimages/curl http://web
```
- 기본 `bridge`, 사용자 정의 브릿지, overlay(스웜/오케스트레이션) 등.

---

# 레지스트리(Registry)

## 개념·종류(보강)

- 공용: **Docker Hub**, **GHCR**, **Quay**
- 사설: **Harbor**, **JFrog Artifactory**, `registry:2`(오픈소스)

## 로그인/푸시/풀

```bash
docker login ghcr.io -u USER
docker tag demo-web:1 ghcr.io/USER/demo-web:1
docker push ghcr.io/USER/demo-web:1

docker pull ghcr.io/USER/demo-web:1
```

## 사설 레지스트리 빠르게 띄우기(개발용)

```bash
docker run -d --restart=always --name reg -p 5000:5000 registry:2
docker tag nginx:alpine localhost:5000/nginx:alpine
docker push localhost:5000/nginx:alpine
docker pull localhost:5000/nginx:alpine
```

## 사설 CA/프록시/미러(운영 시 필수)

```bash
# 데몬 레벨 미러 설정(예: Linux)

sudo mkdir -p /etc/docker
cat <<'JSON' | sudo tee /etc/docker/daemon.json
{
  "registry-mirrors": ["https://mirror.gcr.io"],
  "insecure-registries": ["registry.internal:5000"]
}
JSON
sudo systemctl restart docker
```

- 기업 프록시/사설 CA 사용 시 Docker Desktop(Windows) 또는 systemd drop-in(Linux)로 **HTTP(S)_PROXY**, **신뢰 CA** 설정 필요.

---

# 데몬(Docker Daemon)

## 역할(보강)

- **Docker API(REST)** 요청을 받아 **컨테이너 런타임(containerd/runc)** 를 통해 컨테이너 생성·실행을 조율.
- 네트워크/볼륨/플러그인/이미지·빌드 파이프라인(BuildKit) 관리.

## Docker API 호출 예시(unix socket)

```bash
# 리눅스에서 도커 소켓 직접 호출(정보 조회)

curl --unix-socket /var/run/docker.sock http://localhost/info | jq '.ServerVersion,.Driver'

# 컨테이너 목록 조회

curl --unix-socket /var/run/docker.sock http://localhost/containers/json | jq '.[].Names'
```

> 운영환경에서는 소켓 접근 권한을 엄격히 제한하고, 필요 시 **TCP API + mTLS** 로 보호.

## BuildKit 활성화 및 사용

```bash
export DOCKER_BUILDKIT=1
docker build -t demo-buildkit:1 .
```
- 병렬 빌드, 정교한 캐시(원격 캐시, inline cache), 시크릿/SSH 포워딩 지원.

### BuildKit 시크릿 예시

```Dockerfile
# syntax=docker/dockerfile:1.7

FROM alpine
# --mount=type=secret,id=npmrc 로 빌드타임 비밀 주입

RUN --mount=type=secret,id=npmrc \
    cat /run/secrets/npmrc >/dev/null || true
```
```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t demo-secret:1 .
```

---

# 전체 플로우(보강 다이어그램)

```
+---------------- Registry ------------------+
|   (Docker Hub / GHCR / Harbor / Private)  |
|    ^                     |                |
|    |   pull (GET)        |   push (PUT)   |
+----|--------------------- | --------------+
     |                      |
     v                      v
+-----------------+   docker build   +-----------------+
|  Local Image    | <--------------- |   BuildKit      |
|  Store (CAS)    | ---------------->+  / containerd   |
+--------|--------+                  +--------|--------+
         |                                    |
         | docker run (create+start)          |
         v                                    v
+-----------------+    OCI runtime     +-----------------+
|   Container     | <----------------- |   runc/crun     |
| (namespaces,    |  (create/start)    +-----------------+
|  cgroups, FS)   |
+-----------------+
     ^       |
     |       +-- network/volume mgmt --+
     |                                  |
     +-- Docker CLI/API --> dockerd <---+
```

---

# 스토리지 드라이버와 파일 시스템

## OverlayFS 원리(리눅스)

- **lowerdir**(읽기 전용 레이어들) + **upperdir**(쓰기 레이어) → **merged** 로 노출
- Copy-on-Write: 파일 변경 시 upperdir에 복사본 생성

관찰 예시(컨테이너 내부 파일 변경 확인):
```bash
docker run --name ov -d alpine:3.20 sleep 9999
docker exec ov sh -c 'echo hi > /etc/hi.txt'
# 호스트 경로는 드라이버 구현에 따라 달라짐(관찰용)

docker inspect ov | jq '.[0].GraphDriver'
docker rm -f ov
```

## Windows/WSL2

- Windows native: **windowsfilter**
- WSL2: 내부는 리눅스 커널 + ext4 기반 가상 디스크, 바인드 마운트 시 **WSL2 내부 경로** 권장(성능)

---

# 네트워킹 상세

## 기본 브릿지/사용자 브릿지

```bash
docker network create appnet
docker run -d --name api --network appnet -p 8081:8080 ghcr.io/user/api:1
docker run --rm --network appnet curlimages/curl http://api:8080/health
```

## 포트 포워딩과 충돌 진단

```bash
docker run -d -p 80:80 --name web nginx:alpine
# 충돌 시 포트 점유 프로세스 확인(Windows 예)

netstat -ano | findstr :80
```

## DNS·서비스 디스커버리

- 동일 네트워크의 컨테이너는 **컨테이너명**으로 서로 접근 가능(`api:8080`)

---

# 운영/개발 실전 시나리오

## 로컬 풀스택(Compose) — 이미지·컨테이너·네트워크·볼륨 종합

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
docker compose up -d --build
curl http://localhost:8080
curl http://localhost:8081/health
docker compose down -v
```

## 레지스트리 고정(digest pinning) 배포

```bash
IMG="nginx@sha256:xxxxxxxx"
docker pull "$IMG"
docker run -d -p 8080:80 --name w "$IMG"
```

## 오프라인(Air-gapped) 전달

```bash
docker pull ghcr.io/user/app:1.0
docker save ghcr.io/user/app:1.0 -o app_1_0.tar
# 이동 후

docker load -i app_1_0.tar
docker run --rm ghcr.io/user/app:1.0
```

---

# 빌드·캐시·최적화 패턴

## .dockerignore

```gitignore
node_modules
.git
dist
*.log
```
- 빌드 컨텍스트를 최소화해 빌드 속도↑, 캐시 효율↑.

## 멀티스테이지 최적화

- **빌드 도구 포함 이미지**(용량 큼)에서 산출물만 **슬림 런타임**으로 복사.
- 보안상 공격면 축소, 런타임 이미지 크기↓.

## 레이어 재사용 극대화

- 패키지 설치/의존성 설치를 **변경 빈도 낮은 단계에 배치**.
- `--mount=type=cache`(BuildKit)로 언어별 캐시 활용.

### BuildKit 캐시 예시(Go)

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

# 보안 베스트 프랙티스

- **최소 권한**: `--cap-drop ALL` + 필요한 것만 `--cap-add`.
- **read-only rootfs**, `tmpfs` 활용로 임시 쓰기만 허용.
- **비밀(Secret)** 은 환경변수 대신 **런타임 주입**(예: Swarm/K8s 시크릿, BuildKit secret mount).
- **서명/검증**(cosign/Notary), **취약점 스캔**(trivy), **SBOM** 관리.
- **네트워크 분리/정책**: 민감 컨테이너는 별도 네트워크, Egress 제한.
- **이미지 출처 검증**: 공식/신뢰 레지스트리, digest 고정.

예시:
```bash
docker run --rm \
  --read-only --tmpfs /tmp --tmpfs /run \
  --cap-drop ALL --pids-limit 128 \
  --cpus=0.5 --memory=256m \
  nginx:alpine
```

---

# 트러블슈팅 가이드

## 이미지/레이어 문제

```bash
docker image ls
docker system df
docker system prune -af  # 주의: 필요 자원 삭제
```
- 디스크 압박 시 **중복 레이어 제거/미사용 자원 정리**.

## 컨테이너 즉시 종료

{% raw %}
```bash
docker logs <NAME>
docker inspect <NAME> --format '{{.State.ExitCode}}'
```
{% endraw %}
- Entrypoint/Command 오류, 권한/경로 문제 확인.

## 네트워크 이슈

```bash
docker network inspect <NET>
docker exec -it <NAME> sh -lc 'ip a; ip r; nslookup example.com'
```
- DNS/프록시/포트 충돌 점검.

## 프록시/사설 CA

- 도커 데몬 프록시/CA 설정과 **컨테이너 내부 프록시** 설정을 구분.
- Windows Desktop: Settings → Resources → Proxies/Certificates.

---

# 데몬/런타임 내부 심화

## containerd/runc 파이프라인

- **dockerd** → **containerd**(컨테이너 수명주기/이미지 관리) → **runc**(OCI 런타임, `create/start` 로 실제 프로세스 포크)
- Cgroups v2에서 **리소스 제어**가 일원화.

## 이벤트/로그 스트림

```bash
docker events --filter type=container
docker logs -f <NAME>
```
- CI/CD에서 이벤트 기반 자동화(컨테이너 상태 감시)가 가능.

---

# 실전 연습: 아키텍처 체감 미션

## 미션 1) 캐시 최적화 체감

1. `COPY package*.json` → `npm ci` → `COPY .` 순서로 Dockerfile 구성
2. 소스만 미세 수정 후 빌드 시간을 비교

```bash
time docker build -t app:1 .
# 코드 몇 줄 수정

time docker build -t app:2 .
```

## 미션 2) digest 고정 배포

1. `docker inspect` 로 RepoDigest 추출
2. Compose/K8s 매니페스트에 digest로 이미지 지정 → 재현성 확보

## 미션 3) 보안 옵션 적용

- `--read-only`, `--cap-drop ALL`, `--pids-limit` 조합의 동작 확인

---

# 결론

- **이미지**는 **레이어 + 해시 기반(CAS)** 의 불변 템플릿, **컨테이너**는 이를 **격리된 프로세스**로 실행.
- **레지스트리**는 태그/다이제스트로 이미지 배포 재현성을 보장하며, **데몬**은 BuildKit·containerd·runc와 연동해 전 과정을 자동화.
- 운영에서는 **캐시 최적화, digest 고정, 프록시/사설 CA, 네트워크/볼륨 분리, 최소 권한 실행, 서명/스캔**이 핵심.
- 본 문서의 예제(빌드·런·네트워크·볼륨·API 호출)를 통해 **설치 이후의 아키텍처 이해와 실전 운영 감각**을 강화할 수 있다.

---

## 부록 A. 명령 치트시트(아키텍처 관점)

```bash
# 이미지/레이어

docker history <IMG>
docker image inspect <IMG> | jq '.[0].RootFS.Layers'
docker save -o img.tar <IMG>; docker load -i img.tar

# 컨테이너/리소스

docker run --rm --cpus=1 --memory=512m --pids-limit=128 alpine:3.20 true
docker exec -it <NAME> sh
docker stats

# 네트워크/볼륨

docker network create n1
docker volume create v1
docker network inspect n1
docker volume inspect v1

# 레지스트리

docker login
docker tag app:1 ghcr.io/user/app:1
docker push ghcr.io/user/app:1
docker pull ghcr.io/user/app@sha256:...

# 데몬/API

curl --unix-socket /var/run/docker.sock http://localhost/info | jq .
docker events --since 10m
```

## 부록 B. Windows/WSL2 팁(요약)

- 프로젝트를 **WSL2 내부 경로**(`/home/<user>/...`)에 두면 바인드 마운트 I/O가 빠름.
- `%UserProfile%\.wslconfig` 로 CPU/RAM/SWAP 조정.
- 포트 충돌은 `netstat -ano` 로 점검 후 점유 프로세스 종료/포트 변경.

## 부록 C. 문제 풀이 템플릿(현장 적용)

- 증상: “pull 느림” → 레지스트리 미러 설정, 프록시/CA 확인, DNS 캐시 점검
- 증상: “컨테이너 즉시 종료” → `docker logs`, Entrypoint/Command, 권한/경로 확인
- 증상: “권한/SELinux 오류” → 볼륨에 `:Z/:z` 옵션, 컨테이너 `--user` 매핑
- 증상: “파일 변경 반영 지연” → WSL2 내부 FS로 코드 이동, `docker sync`류 도구 고려

---

# 참고

- OCI Image/Runtime Spec
- Docker Docs: 이미지/레이어/BuildKit/네트워크/볼륨
- cosign/trivy/SBOM 등 공급망 보안 도구
