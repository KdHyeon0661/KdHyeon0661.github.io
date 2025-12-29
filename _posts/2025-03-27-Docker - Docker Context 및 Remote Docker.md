---
layout: post
title: Docker - Docker Context 및 Remote Docker
date: 2025-03-27 20:20:23 +0900
category: Docker
---
# Docker Context 및 Remote Docker: 원격 엔진 관리 완전 가이드

## 개념 이해

Docker Context는 CLI가 통신할 Docker 엔진(데몬)을 프로파일 형태로 저장하고 전환할 수 있는 기능입니다. 기본적으로 로컬 Unix 소켓(`unix:///var/run/docker.sock`)에 연결되어 있지만, SSH나 TLS TCP를 통해 원격 Docker 데몬에 연결할 수 있습니다. Context를 전환한 후 실행하는 모든 `docker` 명령어는 해당 원격 엔진에서 수행됩니다. Buildx는 이러한 Context를 빌더 노드로 활용하여 멀티 플랫폼 빌드나 원격 리소스 활용을 가능하게 합니다.

---

## Context 기본 사용법

### Context 목록 조회 및 현재 Context 확인

```bash
docker context ls
```

출력 예시:
```
NAME        DESCRIPTION                                DOCKER ENDPOINT
default *   Current DOCKER_HOST based configuration    unix:///var/run/docker.sock
myserver    SSH to remote host                         ssh://ubuntu@203.0.113.10
prod        TLS secured remote                         tcp://prod.example.com:2376
```

`*` 표시는 현재 활성화된 Context를 나타냅니다.

### Context 전환

```bash
docker context use myserver
```

이 명령을 실행한 후 모든 `docker` 명령은 `myserver`에 정의된 원격 Docker 데몬에서 수행됩니다.

### 단일 명령에 특정 Context 적용

Context를 전환하지 않고 단일 명령에만 특정 Context를 적용할 수 있습니다:

```bash
docker --context myserver ps
docker --context prod images
```

---

## SSH 기반 원격 Context 설정

SSH를 통한 원격 연결은 보안성이 우수하면서도 설정이 비교적 간단한 방법입니다. 이 방법을 사용하려면 원격 서버에 SSH 접속이 가능하고 Docker가 설치되어 있어야 하며, 접속 사용자가 `sudo` 없이 Docker 명령을 실행할 수 있는 권한이 있어야 합니다(일반적으로 `docker` 그룹에 속한 사용자).

### 사전 준비: SSH 키 설정 및 접속 테스트

```bash
# SSH 키 생성
ssh-keygen -t ed25519 -C "laptop@corp"

# 공개키를 원격 서버에 복사
ssh-copy-id ubuntu@203.0.113.10

# 원격 서버에서 Docker 명령 테스트
ssh ubuntu@203.0.113.10 "docker version"
```

만약 `sudo docker` 명령만 작동한다면 원격 서버에서 다음 명령을 실행하여 사용자를 docker 그룹에 추가해야 합니다:

```bash
sudo usermod -aG docker $USER
# 변경사항 적용을 위해 재로그인 필요
```

### SSH Context 생성

```bash
docker context create myserver \
  --docker "host=ssh://ubuntu@203.0.113.10"
```

### Context 전환 및 원격 명령 실행

```bash
# Context 전환
docker context use myserver

# 원격 엔진 정보 확인
docker info

# 원격 서버의 컨테이너 목록 조회
docker ps

# 원격 서버의 이미지 목록 조회
docker images

# 원격 서버에서 컨테이너 실행
docker run -d -p 8080:80 nginx:alpine
```

**중요**: `-p 8080:80` 옵션으로 노출된 포트는 원격 서버에서 열립니다. 로컬 브라우저에서 접속하려면 `http://203.0.113.10:8080` 주소를 사용해야 합니다.

### 원격 서버에서 이미지 빌드

```bash
docker build -t myapp:remote .
docker run -d --name myapp -p 8080:80 myapp:remote
```

빌드 컨텍스트(소스 코드)는 CLI가 SSH 터널을 통해 원격 데몬으로 업로드합니다. 대용량 프로젝트의 경우 네트워크 대역폭 영향을 고려해야 합니다.

### 베스천 서버를 경유하는 연결

SSH Config 파일(`~/.ssh/config`)에 다음과 같이 설정하여 베스천 서버를 경유할 수 있습니다:

```sshconfig
Host bastion
  HostName bastion.corp.net
  User jump

Host priv-node
  HostName 10.0.12.34
  User ubuntu
  ProxyJump bastion
```

Context 생성:
```bash
docker context create dc-a \
  --docker "host=ssh://priv-node"
```

이렇게 설정하면 CLI → 베스천 → 프라이빗 노드 경로로 안전하게 연결됩니다.

---

## TLS TCP 기반 원격 Context 설정

TLS TCP 연결은 데이터센터나 상용 환경에서 주로 사용됩니다. **보안을 위해 절대 비암호화 TCP(2375 포트)를 사용하지 마세요.** 반드시 TLS 암호화(2376 포트)와 양방향 인증(mTLS)을 구현해야 합니다.

### Docker 데몬 TLS 설정

systemd drop-in 파일을 생성하여 Docker 데몬을 TLS로 설정할 수 있습니다:

`/etc/systemd/system/docker.service.d/10-tls-remote.conf`:
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
sudo ss -lntp | grep 2376  # 포트 리스닝 확인
```

### 클라이언트 Context 생성

```bash
docker context create prod \
  --docker "host=tcp://prod.example.com:2376,ca=/path/ca.pem,cert=/path/client-cert.pem,key=/path/client-key.pem"
```

방화벽에서 2376/TCP 포트를 열어야 하며, CA, 서버, 클라이언트 인증서는 올바른 CN(Common Name)과 SAN(Subject Alternative Name)으로 발급되어야 합니다.

---

## Docker Compose와 원격 Context 통합

Context를 전환한 후 `docker compose` 명령도 원격에서 실행됩니다. 이때 **볼륨이나 바인드 마운트 경로는 원격 서버의 파일시스템을 기준으로 해석된다는 점**을 반드시 기억해야 합니다.

### 예제 프로젝트 구조

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

### 원격에서 Compose 실행

```bash
docker context use myserver
docker compose up -d
docker compose ps
```

`./public` 경로는 원격 서버 기준으로 존재해야 합니다. 일반적으로 로컬 프로젝트 디렉터리를 rsync나 sshfs로 원격 서버에 동기화하거나, 정적 파일을 이미지에 내장하여 바인드 마운트 의존성을 제거합니다.

---

## Buildx와 원격 Context를 활용한 고급 빌드

### 원격 빌더 노드 구성

```bash
docker context use myserver
docker buildx create --name rbuilder --use --driver docker-container
docker buildx inspect --bootstrap
```

`docker-container` 드라이버는 원격 서버에 격리된 빌더 컨테이너를 생성합니다. 멀티 아키텍처 빌드를 위해 QEMU를 설치해야 할 수 있습니다:

```bash
docker run --privileged --rm tonistiigi/binfmt --install all
```

### 멀티 아키텍처 빌드 및 레지스트리 캐시 활용

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=yourname/app:buildcache \
  --cache-to   type=registry,ref=yourname/app:buildcache,mode=max \
  -t yourname/app:1.0.0 -t yourname/app:latest \
  --push .
```

---

## 성능 최적화 전략

### 빌드 컨텍스트 최소화

`.dockerignore` 파일을 활용하여 불필요한 파일 전송을 방지합니다:

```dockerignore
.git
node_modules
dist
*.log
*.tmp
```

Dockerfile 작성 시 캐시 효율성을 고려한 레이어 순서를 적용합니다:

```dockerfile
# 의존성 파일 먼저 복사 및 설치
COPY package.json package-lock.json ./
RUN npm install

# 나머지 소스 코드 복사
COPY . .
```

### 원격 캐시 활용

Buildx의 `--cache-from` 및 `--cache-to` 옵션을 사용하여 CI 환경이나 팀원 간 빌드 캐시를 공유할 수 있습니다.

### 성능 분석 모델

원격 빌드의 총 소요 시간 \(T\)은 다음과 같이 근사할 수 있습니다:

$$
T \approx \frac{S_{\text{ctx}}}{B} + T_{\text{handshake}} + T_{\text{cache\_miss}}
$$

여기서:
- \(S_{\text{ctx}}\): 전송되는 빌드 컨텍스트 크기
- \(B\): 실효 네트워크 대역폭
- \(T_{\text{handshake}}\): 연결 핸드셰이크 시간
- \(T_{\text{cache\_miss}}\): 캐시 미스로 인한 추가 빌드 시간

이 모델에서 알 수 있듯이, 빌드 컨텍스트 크기를 최소화하고 캐시 적중률을 높이는 것이 성능 향상의 핵심입니다.

빈번한 빌드가 필요한 경우 원격 서버에서 직접 Git 클론을 수행하고 CI 러너에서 빌드하는 패턴을 고려할 수 있습니다. 이렇게 하면 빌드 컨텍스트 전송 자체를 없앨 수 있습니다.

---

## 보안 및 권한 관리

### SSH 기반 보안

- SSH 키 페어와 `known_hosts` 파일을 체계적으로 관리합니다.
- 베스천 서버와 MFA(Multi-Factor Authentication)를 조합하여 보안을 강화합니다.
- 서버 측 SSH 설정(`AllowTcpForwarding`, `PermitUserEnvironment` 등)을 적절히 구성합니다.

### TLS mTLS 보안

- CA(Certificate Authority), 서버 인증서, 클라이언트 인증서를 분리하여 관리합니다.
- 방화벽에서 2376 포트만 개방하고, 보안 그룹 규칙을 최소화합니다.
- 정기적인 인증서 로테이션 계획을 수립합니다.

### Rootless Docker

원격 서버에서 Rootless 모드로 Docker를 실행하면 권한 상승에 대한 보안 위협을 줄일 수 있습니다:

```bash
dockerd-rootless-setuptool.sh install
systemctl --user start docker
systemctl --user enable docker
```

Context는 해당 사용자의 rootless 소켓에 자동으로 연결됩니다.

### 감사 및 로깅

- Docker 데몬 로깅 드라이버(syslog 또는 journald)를 구성합니다.
- `docker events` 명령으로 API 이벤트를 추적합니다.
- 원격 서버 측에서 방화벽, Fail2ban, 감사 규칙을 적용합니다.

---

## 실전 운영 시 고려사항

원격 Docker Context를 사용할 때는 "원격은 원격이다"라는 기본 원칙을 명심해야 합니다:

1. **네트워킹**: `-p` 옵션으로 노출되는 포트는 원격 호스트에서 열립니다.
2. **파일 경로**: 바인드 마운트 경로는 원격 서버의 파일시스템을 기준으로 해석됩니다.
3. **리소스**: CPU, 메모리, 디스크 등 리소스는 원격 서버에서 소모됩니다.
4. **이미지 및 볼륨**: `docker images`나 `docker volume ls` 명령은 원격 서버의 상태를 조회합니다.
5. **Compose**: `.env` 파일과 상대 경로는 원격 서버 기준으로 해석됩니다.
6. **CI 통합**: CI 러너가 원격 서버에 접근 가능한지 확인하고 적절한 캐시 전략을 수립합니다.
7. **이름 충돌**: 다중 사용자 환경에서는 서비스나 컨테이너 이름, 포트 번호 충돌에 주의합니다.

---

## 문제 해결 가이드

| 증상 | 가능한 원인 | 해결 방안 |
|---|---|---|
| `Got permission denied while trying to connect to the Docker daemon socket` | 원격 사용자에게 docker 그룹 권한 없음 | 원격 서버에서 `sudo usermod -aG docker $USER` 실행 후 재로그인 |
| `ssh: handshake failed` | SSH 키 또는 known_hosts 문제 | `ssh -v user@host` 명령으로 디버깅, SSH 키 재배포 |
| 빌드 속도가 매우 느림 | 빌드 컨텍스트가 너무 크거나 캐시 미스 | `.dockerignore` 강화, 의존성→소스 순서 적용, 원격 캐시 도입 |
| 바인드 마운트 실패 | 원격 서버에 해당 경로 존재하지 않음 | rsync/sshfs로 파일 동기화 또는 이미지 내장 방식으로 전환 |
| 2376 포트 접속 실패 | 방화벽 차단 또는 인증서 불일치 | 방화벽 규칙 확인, 인증서 재발급, SAN에 FQDN/IP 포함 |
| Buildx `--load` 실패 | 멀티플랫폼 이미지는 `--load` 옵션 지원 안 함 | 멀티플랫폼 이미지는 `--push` 사용 후 레지스트리에서 풀 |
| 베스천 경유 연결 실패 | ProxyJump 설정 누락 | `~/.ssh/config` 파일의 ProxyJump 설정 확인 |

---

## 실습 시나리오: 원격 Ubuntu 서버에 Nginx 배포

1. Context 생성
```bash
docker context create edge-a --docker "host=ssh://ubuntu@198.51.100.20"
docker context use edge-a
```

2. 컨테이너 실행
```bash
docker run -d --name site -p 8080:80 nginx:alpine
```

3. 상태 확인
```bash
docker ps
curl http://198.51.100.20:8080
```

4. Docker Compose로 전환
```yaml
# docker-compose.yml (원격 서버의 프로젝트 디렉터리에 저장)

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

## 개발 워크플로우 패턴

### 패턴 A: 중앙 집중식 빌드 (권장)
로컬에서 개발 → Git 저장소에 푸시 → 원격 CI 러너/서버에서 빌드 → 레지스트리에 푸시 → 원격 배포

**장점**: 빌드 컨텍스트 전송 불필요, 캐시와 빌드 환경 일관성 유지, 비밀 정보 안전하게 관리

### 패턴 B: 직접 원격 빌드
로컬에서 `docker build` 명령을 SSH Context로 원격 실행

**장점**: 추가 장비 없이 단일 개발 머신으로 가능
**단점**: 대용량 프로젝트에서 네트워크 대역폭 영향, 대규모 프로젝트에 부적합

---

## 핵심 명령어 참조

```bash
# Context 목록 및 전환
docker context ls
docker context use <name>
docker --context <name> ps

# SSH Context 생성
docker context create myserver --docker "host=ssh://user@host"

# TLS Context 생성
docker context create prod \
  --docker "host=tcp://host:2376,ca=/path/ca.pem,cert=/path/cert.pem,key=/path/key.pem"

# Context 관리
docker context rm myserver
docker context update myserver --description "Seoul rack A"

# 원격에서 Docker Compose 실행
docker --context myserver compose up -d
docker --context myserver compose logs -f

# 원격 Buildx 구성 및 사용
docker context use myserver
docker buildx create --name rbuilder --use --driver docker-container
docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:latest --push .
```

---

## 구성 템플릿

### SSH Config (베스천 포함)
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

### systemd Drop-in (TLS 원격 연결)
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

### 효율적인 .dockerignore
```dockerignore
.git
node_modules
dist
__pycache__
*.log
*.tmp
```

---

## 자주 묻는 질문

**Q1: 로컬에서 `-p 8080:80`으로 컨테이너를 실행했는데 접속이 안 됩니다.**

A: 포트는 원격 서버에서 열립니다. 원격 서버의 IP 주소로 접속해야 하며, 원격 서버의 방화벽이나 보안 그룹 설정을 확인하세요.

**Q2: 원격 바인드 마운트가 실패합니다.**

A: 바인드 마운트 경로는 원격 서버를 기준으로 합니다. 해당 경로가 원격 서버에 존재하는지 확인하고, 필요한 경우 SELinux나 AppArmor 정책을 점검하세요.

**Q3: 원격 빌드가 너무 느립니다.**

A: `.dockerignore` 파일로 불필요한 파일을 제외하고, 의존성 설치를 먼저 수행하여 캐시 효율을 높이세요. 레지스트리 캐시를 활용하거나 원격 CI 러너 사용을 고려하세요.

**Q4: HTTP 프록시 뒤에 있는 환경에서 사용할 수 있나요?**

A: SSH와 TLS 모두 프록시 설정이 필요할 수 있습니다. 필요한 포트와 도메인을 화이트리스트에 추가하고 프록시 정책을 조정하세요.

**Q5: 여러 사용자가 동일한 원격 서버를 사용할 때 충돌이 발생합니다.**

A: 프로젝트별 접두사 사용, 포트 번호 분배, Docker Compose 프로젝트 이름(`-p` 옵션) 지정으로 충돌을 방지하세요.

---

## 결론: 원격 Docker 컨텍스트의 효과적 활용

Docker Context와 원격 Docker 기능은 현대적인 컨테이너 기반 개발 및 운영 워크플로우에서 필수적인 도구입니다. 이 기술을 효과적으로 활용하기 위한 핵심 원칙을 정리합니다:

1. **적절한 연결 방식 선택**: SSH 연결은 보안성과 사용 편의성 면에서 대부분의 경우 최적의 선택입니다. TLS 연결은 엄격한 접근 제어가 필요한 기업 환경에 적합합니다.

2. **원격 환경의 특성 이해**: 원격에서 실행되는 모든 작업은 해당 서버의 리소스를 사용하며, 파일 경로와 네트워크 설정은 원격 서버를 기준으로 해석됩니다. 이 기본 원칙을 명심하지 않으면 예기치 않은 문제에 직면할 수 있습니다.

3. **성능 최적화 전략 수립**: 빌드 컨텍스트 크기 최소화, 효율적인 캐싱 전략, 적절한 네트워크 구성은 원격 작업의 성능을 결정하는 핵심 요소입니다.

4. **보안 체계 구축**: SSH 키 관리, TLS 인증서 로테이션, 최소 권한 원칙 적용, 정기적인 감사 로깅을 통해 안전한 원격 운영 환경을 구축합니다.

5. **자동화와 통합**: CI/CD 파이프라인, GitOps 워크플로우, 모니터링 시스템과 통합하여 일관되고 효율적인 운영 체계를 수립합니다.

6. **점진적 도입 접근**: 처음부터 모든 기능을 구현하려 하기보다 핵심 시나리오부터 시작하여 점진적으로 확장하는 접근 방식을 채택합니다.

Docker Context는 단순한 기술적 기능을 넘어 팀의 협업 방식을 변화시킬 수 있는 강력한 도구입니다. 로컬 개발 환경과 원격 운영 환경의 경계를 넘나들며 일관된 개발 경험을 제공함으로써, 더 빠르고 안정적인 소프트웨어 제공 주기를 실현할 수 있습니다.

이 가이드의 원칙과 모범 사례를 적용하면 복잡한 인프라 환경에서도 Docker의 강력한 기능을 효과적으로 활용할 수 있을 것입니다. 기술이 발전함에 따라 새로운 기능과 개선 사항이 지속적으로 도입되겠지만, 여기서 소개한 기본 원칙은 앞으로도 유효한 지침이 될 것입니다.