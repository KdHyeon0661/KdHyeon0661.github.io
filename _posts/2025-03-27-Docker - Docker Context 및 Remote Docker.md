---
layout: post
title: Docker - Docker Context 및 Remote Docker
date: 2025-03-27 20:20:23 +0900
category: Docker
---
# 🌐 Docker Context 및 Remote Docker 사용법

---

## 📌 1. Docker Context란?

> "다양한 Docker 데몬 환경을 손쉽게 전환하며 제어할 수 있게 해주는 기능"

Docker는 기본적으로 **로컬 Docker 데몬**에 명령을 내립니다.  
하지만 context 기능을 활용하면 아래처럼 전환할 수 있습니다:

- ✅ 로컬 Docker
- ✅ 리눅스 서버의 Docker 데몬 (SSH 접속)
- ✅ Docker in Docker (CI/CD에서)
- ✅ ECS, ACI, Kubernetes 등 클라우드 백엔드

---

## ✅ 2. 현재 Context 확인 및 전환

```bash
docker context ls
```

출력 예시:

```
NAME                DESCRIPTION                               DOCKER ENDPOINT               ...
default *           Current DOCKER_HOST based configuration   unix:///var/run/docker.sock
myserver            SSH to remote host                       ssh://user@192.168.0.100
```

```bash
docker context use myserver
```

→ 이후 모든 `docker build`, `docker run`, `docker ps` 등은 **myserver에서 실행됨**

---

## 🛠️ 3. 원격 Docker Context 생성 (SSH 방식)

```bash
docker context create myserver  \
  --docker "host=ssh://user@192.168.0.100"
```

옵션 설명:

| 옵션 | 의미 |
|------|------|
| `myserver` | context 이름 |
| `host=ssh://...` | SSH를 통해 해당 서버의 Docker 데몬 제어 |

> 🔐 서버에 미리 SSH 접속 가능해야 하고, `docker`가 설치되어 있어야 함

---

## 🐳 4. 원격에서 Docker 명령 실행하기

```bash
docker context use myserver

docker ps        # → myserver의 컨테이너 목록 출력
docker build .   # → 원격 서버에서 빌드 실행
docker run ...   # → 원격 서버에서 컨테이너 실행
```

---

## 🧼 5. Context 삭제

```bash
docker context rm myserver
```

---

## 🔐 6. Remote Docker 보안: TLS 방식 (고급)

### 1️⃣ 서버 측: TLS 인증서 기반 설정

```bash
dockerd \
  --host tcp://0.0.0.0:2376 \
  --tlsverify \
  --tlscacert=/etc/docker/ca.pem \
  --tlscert=/etc/docker/server-cert.pem \
  --tlskey=/etc/docker/server-key.pem
```

### 2️⃣ 클라이언트 측: Docker Context 생성

```bash
docker context create secure-remote \
  --docker "host=tcp://REMOTE_IP:2376,ca=/path/to/ca.pem,cert=/path/to/cert.pem,key=/path/to/key.pem"
```

> 🔐 **주의**: TLS 인증서를 생성 및 배포해야 하며, TCP 포트는 방화벽에서 열어야 함

---

## 🧪 7. 실습 예제: 원격 서버에서 빌드 실행하기

```bash
docker context create mybuild \
  --docker "host=ssh://ubuntu@123.45.67.89"

docker context use mybuild

docker build -t myapp:remote .
docker run -d -p 8080:80 myapp:remote
```

→ 내 로컬 PC에서 명령을 내리지만, 실제 컨테이너는 **123.45.67.89**에서 돌아감

---

## 🧰 8. 응용: `docker buildx`에서 remote context 사용

```bash
docker buildx create --name remote-builder --use --platform linux/arm64 \
  --driver docker-container \
  --context mybuild
```

→ ARM64 이미지 빌드를 **리눅스 ARM 서버에서 실행**

---

## 🌐 9. Cloud Backend Context (선택)

```bash
docker context create ecs my-ecs \
  --from-env
```

> Amazon ECS, Azure ACI 등도 Context로 통합 가능

---

## 📚 참고 자료

- [Docker Context 공식 문서](https://docs.docker.com/engine/context/working-with-contexts/)
- [Buildx + Remote Context 사용](https://docs.docker.com/build/building/context/)
- [TLS 기반 원격 Docker 접근](https://docs.docker.com/engine/security/protect-access/)

---

## ✅ 요약

| 기능 | 명령 |
|------|------|
| Context 목록 보기 | `docker context ls` |
| Context 전환 | `docker context use [이름]` |
| SSH로 원격 연결 | `docker context create --docker host=ssh://user@ip` |
| 원격 명령 실행 | `docker ps`, `docker build`, `docker run` 등 |
| 보안 연결 (TLS) | `--host tcp:// --tlsverify` 방식 사용 |