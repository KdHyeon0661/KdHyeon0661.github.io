---
layout: post
title: Docker - Docker Context 및 Remote Docker
date: 2025-03-27 20:20:23 +0900
category: Docker
---
# Docker Context 및 Remote Docker

## 0. 컨셉 빠른 요약

- **Docker Context** = “CLI가 어떤 Docker 엔진(데몬)과 통신할지”를 **프로파일**로 저장/전환하는 기능
- 기본은 로컬 소켓(`unix:///var/run/docker.sock`)이지만, **SSH** 또는 **TLS TCP**로 **원격 데몬**에 연결 가능
- Context 전환 후의 `docker build/run/compose/ps/logs …`는 **모두 원격**에서 수행
- **Buildx**는 Context를 노드로 사용하여 **멀티 플랫폼 빌드**나 **원격 리소스 활용**이 가능

---

## 1. Context 기초 — 생성/조회/전환

### 1.1 현재 Context 목록 / 사용 중인 Context
```bash
docker context ls
```

예시:
```
NAME        DESCRIPTION                                DOCKER ENDPOINT
default *   Current DOCKER_HOST based configuration    unix:///var/run/docker.sock
myserver    SSH to remote host                         ssh://ubuntu@203.0.113.10
prod        TLS secured remote                          tcp://prod.example.com:2376
```

- `*`가 붙은 항목이 **현재 사용 중** Context.

### 1.2 Context 전환
```bash
docker context use myserver
```
- 이후의 모든 `docker` 명령은 **myserver**의 도커 데몬에서 수행된다.

### 1.3 한 번만 다른 Context로 실행 (`--context`)
```bash
docker --context myserver ps
docker --context prod images
```
- 전역 전환 없이 **단일 명령**에만 특정 Context 적용.

---

## 2. SSH 기반 Remote Context — 가장 안전하고 쉬운 방법

> 전제: 원격 서버에 **SSH 접속 가능**, **docker 설치**, 접속 사용자에게 `sudo 없이` docker 사용 권한이 있어야 편하다(예: `docker` 그룹).

### 2.1 SSH 키/접속 준비
```bash
ssh-keygen -t ed25519 -C "laptop@corp"
ssh-copy-id ubuntu@203.0.113.10
ssh ubuntu@203.0.113.10 "docker version"
```
- `docker version`이 정상 동작해야 한다. 만약 `sudo docker`만 되면, **docker 그룹**에 사용자 추가:
```bash
sudo usermod -aG docker $USER
# 재로그인 필요
```

### 2.2 Context 생성(SSH)
```bash
docker context create myserver \
  --docker "host=ssh://ubuntu@203.0.113.10"
```

### 2.3 전환 & 원격 명령
```bash
docker context use myserver

docker info          # → 원격 서버의 엔진 상태
docker ps            # → 원격 컨테이너 목록
docker images        # → 원격 이미지 목록
docker run -d -p 8080:80 nginx:alpine  # → 원격 서버에서 컨테이너 기동
```

> **중요**: Port `-p 8080:80`은 **원격 서버**에서 노출된다.
> 로컬 브라우저에서 접속하려면 `http://203.0.113.10:8080`.

### 2.4 빌드도 원격에서
```bash
docker build -t myapp:remote .
docker run -d --name myapp -p 8080:80 myapp:remote
```
- **빌드 컨텍스트(소스)** 는 CLI가 SSH 터널로 **원격 데몬에 업로드**한다. 대용량일수록 네트워크 대역폭의 영향을 받는다(아래 성능 섹션 참조).

### 2.5 SSH 고급: 점프호스트(베스천) 경유
`~/.ssh/config`:
```sshconfig
Host bastion
  HostName bastion.corp.net
  User jump

Host priv-node
  HostName 10.0.12.34
  User ubuntu
  ProxyJump bastion
```
Context:
```bash
docker context create dc-a \
  --docker "host=ssh://priv-node"
```
- CLI→베스천→프라이빗노드 경유로 안전하게 연결.

---

## 3. TLS TCP 기반 Remote Context — 데몬을 TCP로 노출(상용/데이터센터)

> **절대 무방비로 2375(plain TCP)를 열지 말 것.** **2376/TLS** + **양방향 인증(mTLS)** 필수.

### 3.1 서버 측(cert 준비 후) dockerd 설정
systemd drop-in 예시: `/etc/systemd/system/docker.service.d/10-tls-remote.conf`
```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd \
  --host=unix:///var/run/docker.sock \
  --host=tcp://0.0.0.0:2376 \
  --tlsverify \
  --tlscacert=/etc/docker/ssl/ca.pem \
  --tlscert=/etc/docker/ssl/server-cert.pem \
  --tlskey=/etc/docker/ssl/server-key.pem
```
적용:
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo ss -lntp | grep 2376
```

### 3.2 클라이언트 Context 생성
```bash
docker context create prod \
  --docker "host=tcp://prod.example.com:2376,ca=/path/ca.pem,cert=/path/client-cert.pem,key=/path/client-key.pem"
```
- 방화벽에서 **2376/TCP** 오픈
- CA/서버/클라이언트 인증서 **CN/SAN** 올바르게 발급(예: `openssl` or `cfssl` 사용)

---

## 4. Compose × Remote Context — 팀 개발/운영 루틴

> Context 전환 후 `docker compose`도 **원격에서** 실행된다.
> **볼륨/바인드 경로는 “원격 서버의 파일시스템 기준”** 임을 반드시 기억.

### 4.1 예시 프로젝트
```
app/
├─ docker-compose.yml
└─ .env
```

`docker-compose.yml`:
```yaml
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
    volumes:
      - ./public:/usr/share/nginx/html:ro
```

### 4.2 원격 실행
```bash
docker context use myserver
docker compose up -d
docker compose ps
```
- `./public` **경로는 원격 서버 기준**으로 존재해야 한다.
  - 로컬 프로젝트 디렉터리를 **rsync/sshfs** 등으로 원격에 동기화하는 패턴이 흔하다.
  - 또는 **이미지에 정적 파일을 내장**해 바인드 의존 제거.

---

## 5. Buildx × Remote Context — 멀티 아키텍처/원격 리소스 활용

### 5.1 원격 빌더 노드 구성
```bash
docker context use myserver
docker buildx create --name rbuilder --use --driver docker-container
docker buildx inspect --bootstrap
```
- `docker-container` 드라이버는 원격 서버에 빌더 컨테이너를 띄워 격리된 환경에서 빌드.
- 필요 시 QEMU 설치:
```bash
docker run --privileged --rm tonistiigi/binfmt --install all
```

### 5.2 멀티아치 빌드 + 레지스트리 캐시
```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=yourname/app:buildcache \
  --cache-to   type=registry,ref=yourname/app:buildcache,mode=max \
  -t yourname/app:1.0.0 -t yourname/app:latest \
  --push .
```

---

## 6. 성능 최적화 — 업로드·캐시·네트워크

### 6.1 빌드 컨텍스트 최소화
- `.dockerignore`로 **컨텍스트 축소** (거대한 `.git`, `node_modules`, `venv`, `dist` 제외)
- `COPY package.json` → deps 설치 → `COPY . .` 순서로 **캐시 히트** 극대화

`.dockerignore` 예:
```dockerignore
.git
node_modules
dist
*.log
*.tmp
```

### 6.2 원격 캐시 / Inline 캐시
- Buildx의 `--cache-from/--cache-to`(type=registry or gha)로 **CI/팀 간 캐시 공유**
- Inline cache도 함께 사용 가능

### 6.3 대역폭/지연을 수식으로 감 잡기
원격 빌드 총 전송 시간 \(T\) 근사:
$$
T \approx \frac{S_{\text{ctx}}}{B} + T_{\text{handshake}} + T_{\text{cache\_miss}}
$$
- \(S_{\text{ctx}}\): 전송되는 빌드 컨텍스트 크기
- \(B\): 실효 대역폭
- **따라서** \(S_{\text{ctx}}\)를 작게, **캐시 미스**를 줄이는 것이 지배적.

> 빈번 빌드라면 **원격 Git clone + CI 러너에서 직접 빌드**가 유리할 수 있다. (컨텍스트 전송 자체를 없앰)

---

## 7. 보안/권한 — SSH·TLS·rootless·감사

### 7.1 SSH 권장
- SSH 키 페어 + `known_hosts` 관리
- 베스천(ProxyJump)·MFA 조합
- 서버 측 `AllowTcpForwarding`, `PermitUserEnvironment` 등 정책 점검

### 7.2 TLS mTLS
- CA, 서버/클라 인증서 분리
- 포트 2376만 개방, 보안 그룹 최소화
- 인증서 로테이션 계획

### 7.3 Rootless Docker(원격 사용자 권한 최소화)
- 원격 서버에서 **rootless 모드**로 Docker 실행:
```bash
dockerd-rootless-setuptool.sh install
systemctl --user start docker
systemctl --user enable docker
```
- Context는 해당 사용자의 **rootless 소켓**으로 연결(SSH면 자동).

### 7.4 감사/로깅
- 데몬 로깅 드라이버(syslog/journald) 설정
- `docker events`로 API 이벤트 추적
- 원격 서버 측 방화벽/Fail2ban/감사 규칙

---

## 8. 운영 실전 팁 — “원격은 원격이다”

| 주제 | 체크리스트 |
|---|---|
| 네트워킹 | `-p`로 노출되는 포트는 **원격 호스트**에서 열린다 |
| 파일경로 | 바인드 마운트 경로는 **원격 파일시스템 기준** |
| 리소스 | CPU/RAM/디스크는 **원격 서버 자원**이 소모 |
| 이미지/볼륨 | `docker images/volume ls`는 **원격 상태** 조회 |
| Compose | `.env`, 파일 경로 해석도 원격 기준(경로 존재 확인) |
| CI 통합 | 런너가 원격에 접근 가능? 캐시 전달 전략은? |
| Name 충돌 | 동일 서비스/컨테이너 이름, Port 충돌 주의(멀티 사용자 환경) |

---

## 9. 트러블슈팅 표

| 증상 | 원인 | 해결 |
|---|---|---|
| `Got permission denied while trying to connect to the Docker daemon socket` | 원격 사용자에 docker 그룹 권한 없음 | 원격에서 `sudo usermod -aG docker $USER`, 재로그인 |
| `ssh: handshake failed` | SSH 키/known_hosts 문제 | `ssh -v user@host`로 진단, 키 재배포 |
| Build가 너무 느림 | 컨텍스트 과대, 캐시 미스 | `.dockerignore` 강화, deps→src 순서, 원격 캐시 도입 |
| 바인드 마운트 파일 없음 | 경로가 “원격 기준”인데 원격에 파일 없음 | rsync/sshfs로 동기화 or 이미지 내장 |
| 2376 접속 실패 | 방화벽/인증서 CN/SAN 불일치 | FW 오픈, cert 재발급, SAN에 FQDN/IP 포함 |
| Buildx `--load` 실패(멀티플랫폼) | `--load`는 단일 플랫폼만 | 멀티플랫폼은 `--push` 사용 |
| 베스천 경유 실패 | ProxyJump 설정 누락 | `~/.ssh/config`의 ProxyJump 확인 |

---

## 10. 실습 시나리오: “원격 우분투에서 Nginx 띄우기”

1) Context 만들기
```bash
docker context create edge-a --docker "host=ssh://ubuntu@198.51.100.20"
docker context use edge-a
```

2) 컨테이너 실행
```bash
docker run -d --name site -p 8080:80 nginx:alpine
```

3) 상태 확인
```bash
docker ps
curl http://198.51.100.20:8080
```

4) Compose로 교체(정적 html)
```yaml
# docker-compose.yml (원격 서버의 프로젝트 폴더에)
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
    volumes:
      - ./public:/usr/share/nginx/html:ro
```
```bash
docker compose up -d
```

---

## 11. 원격에서의 “개발자 루프” 패턴

- **패턴 A (권장)**: 로컬에서 개발 → git push → **원격 CI 러너/서버에서 build** → 레지스트리 push → 원격 배포
  - 장점: 컨텍스트 전송이 없다. 캐시/비밀/빌드 환경이 일관.
- **패턴 B (간편)**: 로컬에서 `docker build`를 **SSH Context**로 → 원격에서 실행
  - 장점: 장비 하나로 충분.
  - 단점: 컨텍스트 전송/대역폭, 대규모 프로젝트에 불리.

---

## 12. 명령어 치트시트

```bash
# 목록/전환
docker context ls
docker context use <name>
docker --context <name> ps

# 생성(SSH)
docker context create myserver --docker "host=ssh://user@host"

# 생성(TLS)
docker context create prod \
  --docker "host=tcp://host:2376,ca=/path/ca.pem,cert=/path/cert.pem,key=/path/key.pem"

# 삭제/수정
docker context rm myserver
docker context update myserver --description "Seoul rack A"

# Compose(원격)
docker --context myserver compose up -d
docker --context myserver compose logs -f

# Buildx(원격 빌더)
docker context use myserver
docker buildx create --name rbuilder --use --driver docker-container
docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:latest --push .
```

---

## 13. 구성 템플릿 모음

### 13.1 SSH Config (bastion 포함)
```sshconfig
Host bastion
  HostName bastion.company.com
  User ops
  IdentityFile ~/.ssh/ops_ed25519

Host node-sea
  HostName 10.10.20.30
  User ubuntu
  IdentityFile ~/.ssh/ops_ed25519
  ProxyJump bastion
```

### 13.2 systemd drop-in (TLS 원격)
`/etc/systemd/system/docker.service.d/10-tls-remote.conf`
```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd \
  --host=unix:///var/run/docker.sock \
  --host=tcp://0.0.0.0:2376 \
  --tlsverify \
  --tlscacert=/etc/docker/ssl/ca.pem \
  --tlscert=/etc/docker/ssl/server-cert.pem \
  --tlskey=/etc/docker/ssl/server-key.pem
```

### 13.3 `.dockerignore`
```dockerignore
.git
node_modules
dist
__pycache__
*.log
*.tmp
```

---

## 14. FAQ

**Q1. 로컬에서 `-p 8080:80` 했는데 접속이 안 됩니다.**
A. 원격에서 포트를 열었습니다. **원격 서버 IP**로 접속해야 하며, 원격 서버 방화벽/보안 그룹 확인.

**Q2. 원격 바인드 마운트가 실패합니다.**
A. 경로는 **원격 기준**입니다. 그 경로가 원격에 존재해야 하며, 권한/SELinux/AppArmor 정책도 확인.

**Q3. Build가 너무 느립니다.**
A. `.dockerignore`, deps→src 순서, 레지스트리 캐시, 원격 CI 러너를 고려하세요.

**Q4. SSH가 아닌 HTTP 프록시 뒤입니다.**
A. SSH/TLS 모두 프록시 정책과 방화벽을 조율해야 합니다. 필요한 포트/도메인 화이트리스트 협의.

**Q5. 여러 사용자가 한 서버를 씁니다. 충돌이 납니다.**
A. 네임스페이스 구분(프로젝트별 접두사), 포트 충돌 회피, Compose 프로젝트 이름(`-p`) 사용.

---

## 15. 결론

- **Docker Context**로 **“로컬 조작, 원격 실행”**이 쉬워진다.
- **SSH Context**는 보안/운영 난이도 대비 생산성이 뛰어나 **가장 실용적**.
- **TLS mTLS**는 데이터센터/상용 환경에서 엄격한 접근 통제로 적합.
- **Compose/Buildx/CI**와 결합하면, **멀티 아키텍처**, **원격 캐시**, **무중단 배포 루프**까지 유연하게 확장된다.
- 핵심: **경로는 원격 기준**, **포트는 원격에서 열린다**, **캐시와 컨텍스트를 다스려라**.

---

## 참고 자료
- Docker Context: https://docs.docker.com/engine/context/working-with-contexts/
- Protect access to the Docker daemon (TLS): https://docs.docker.com/engine/security/protect-access/
- Buildx & remote contexts: https://docs.docker.com/build/building/context/
- Compose CLI: https://docs.docker.com/compose/
- Rootless Docker: https://docs.docker.com/engine/security/rootless/
