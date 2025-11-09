---
layout: post
title: Docker - Docker 설치
date: 2024-12-28 20:20:23 +0900
category: Docker
---
# Docker 설치 방법 (Windows / Linux)

본 문서는 기존에 제시된 **Windows / Linux에서 Docker 설치** 가이드를 **핵심을 보강하고 세부 시나리오까지 확장**한 블로그 글입니다.  
**설치 전 준비 → 설치 경로별 절차 → 검증 방법 → 기업/학교 환경(프록시·오프라인) 대응 → 성능·보안·트러블슈팅**을 순서대로 다룹니다.  

---

## 0. 빠른 개요(요약표)

| OS | 권장 설치 방식 | 하이퍼바이저 | 특징 |
|----|----------------|--------------|------|
| Windows 10/11 Pro 이상 | Docker Desktop (Hyper-V 또는 WSL2) | Hyper-V 또는 WSL2 | GUI 관리 도구 포함, Kubernetes 옵션, 자동 업데이트 |
| Windows 10/11 Home | Docker Desktop (WSL2) | WSL2 | Hyper-V 미지원, WSL2 기반 엔진 필수 |
| Ubuntu (권장), Debian, Fedora 등 | 패키지 설치 (공식 리포지토리) | 커널 직접 사용 | 가장 자연스러운 환경, 성능 우수, 서버/CI 표준 |
| RHEL/CentOS/Alma/Rocky | 공식 리포지토리 혹은 Mirantis 패키지 | 커널 직접 사용 | SELinux/AppArmor 고려 필요 |
| Air-gapped/폐쇄망 | 오프라인 번들 + 로컬 레지스트리 | 환경에 따라 다름 | 이미지 미러/사설 레지스트리 운영 필요 |

---

# 1. Windows에서 Docker 설치

## 1.1 설치 전 확인 사항(확장)

| 항목 | 설명 | 확인 방법(예시) |
|------|------|----------------|
| 운영체제 | Windows 10/11 Pro, Enterprise, Education 권장. Home은 WSL2 필수 | 설정 → 시스템 → 정보에서 에디션 확인 |
| 가상화 지원 | BIOS/UEFI에서 Intel VT-x/AMD-V 활성화 | 작업 관리자 → 성능 → CPU → 가상화: 사용 |
| 하이퍼바이저 | Pro 이상은 Hyper-V 사용 가능. Home은 Hyper-V 불가 → WSL2 사용 | `OptionalFeatures.exe`에서 Hyper-V, WSL 체크 가능 |
| 권한/정책 | 회사 PC는 그룹정책/Defender Application Control 영향 가능 | IT 정책 문서, gpedit.msc, 보안 소프트웨어 정책 확인 |
| 디스크/메모리 | 최소 2 vCPU, 4~8 GB RAM 권장. 이미지 캐시 공간(수십 GB) | `wsl --status`, Docker Desktop Settings의 Resources 확인 |

> 핵심 포인트  
> - **Windows Home = WSL2 엔진**을 사용.  
> - **Windows Pro = Hyper-V 또는 WSL2** 중 선택 가능(최근엔 WSL2 권장).  
> - Hyper-V 사용 시 Windows Sandbox/Device Guard/VM 플랫폼 등과의 충돌을 점검.

---

## 1.2 설치 방법 1: Windows Pro 이상 (Hyper-V 또는 WSL2)

### 1.2.1 Docker Desktop 설치

1. Docker Desktop 설치 파일 다운로드  
2. 설치 중 엔진 선택: **Hyper-V** 또는 **WSL2**  
3. 필요 Windows 기능 자동 활성화(재부팅 필요)
4. 설치 완료 후 Docker Desktop 실행

### 1.2.2 기능 확인(명령 예시)

```powershell
# PowerShell 또는 CMD
docker version
docker info

# 간단 동작 검증
docker run --rm hello-world

# 이미지/컨테이너 목록
docker images
docker ps -a
```

### 1.2.3 Hyper-V vs WSL2 선택 기준

| 구분 | Hyper-V | WSL2 |
|-----|---------|------|
| 적합한 용도 | 기업 표준 VM 격리, 네이티브 VM 네트워킹 | 개발 편의성, WSL 유닉스 환경, 파일 I/O 개선 |
| 파일시스템 성능 | Windows ↔ VM 경계 I/O 비용 존재 | WSL2 내부 리눅스 FS 사용 시 빠름 |
| 네트워킹 | 별도 가상 스위치 사용 | WSL2 NAT 네트워킹, 포트포워딩 자동 |
| 호환성 | 일부 보안 도구와 충돌 가능 | 점진적으로 개선, 최신 기능 수용 빠름 |

> 최근 개발 환경에서는 **WSL2 엔진**이 권장되는 경향이 많습니다.

---

## 1.3 설치 방법 2: Windows Home (WSL2 기반)

### 1.3.1 WSL2 활성화(신형)

```powershell
wsl --install
```

- 위 명령으로 WSL, Virtual Machine Platform, Ubuntu 기본 설치까지 자동.
- 재부팅 후 배포판 사용자 생성.

### 1.3.2 WSL2 수동 활성화(구형/호환)

```powershell
# WSL, VM 플랫폼 개별 활성화
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# 재부팅 후 커널 업데이트(필요 시)
# https://aka.ms/wsl2kernel

# 기본 버전 2로 설정
wsl --set-default-version 2
```

### 1.3.3 리눅스 배포판 설치

```powershell
# 대표적으로 Ubuntu 설치
wsl --install -d Ubuntu
```

또는 Microsoft Store에서 `Ubuntu`, `Debian`, `openSUSE`, `Alpine` 등 선택.

### 1.3.4 Docker Desktop 설치 및 WSL 연동

- Docker Desktop 설치 시 **Use WSL 2 based engine** 선택
- `Settings > Resources > WSL Integration`에서 사용 중인 배포판(예: Ubuntu) 체크
- 이후 **Windows 터미널 / PowerShell / WSL 쉘** 어디서든 `docker` 사용 가능

#### 동작 확인

```bash
# WSL(Ubuntu) 터미널
docker run --rm hello-world

# 볼륨/바인드 마운트 예시
mkdir -p ~/data
docker run --rm -v ~/data:/data alpine ls -al /data
```

---

## 1.4 Docker Desktop 설치 후 체크리스트

### 1.4.1 일반 확인

```powershell
docker version
docker info
```

- `Server` 섹션에서 `Containerd`/`OS/Kernel` 정보 확인
- `Default Runtime`, `cgroup version` 등 점검

### 1.4.2 Docker Compose v2

```powershell
docker compose version

# 간단 Compose 테스트
mkdir demo && cd demo
@"
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
"@ | Out-File -FilePath docker-compose.yaml -Encoding utf8

docker compose up -d
curl http://localhost:8080
docker compose down
```

### 1.4.3 리소스 제한(WSL2 엔진)

- `%UserProfile%\.wslconfig` 파일로 메모리/CPU 제한

```ini
# %UserProfile%\.wslconfig
[wsl2]
memory=6GB
processors=4
swap=2GB
localhostForwarding=true
```

```powershell
wsl --shutdown
# 재시작 후 적용
```

### 1.4.4 파일 시스템 경로 주의(바인드 마운트)

- **WSL2 내부 경로(예: /home/USER/...)** 를 바인드 마운트하면 성능 우수
- Windows 경로를 마운트할 때는 `C:\...` → `/mnt/c/...` 형식 변환 필요

```bash
# WSL2 쉘에서 Windows 디렉터리 바인드(속도는 WSL 내부 FS보다 느릴 수 있음)
docker run --rm -v /mnt/c/Users/USER/projects:/app alpine ls -al /app
```

---

## 1.5 Windows 문제 해결(대표 시나리오)

### 시나리오 A: Docker가 시작되지 않음

```powershell
# 로그 확인
& "C:\Program Files\Docker\Docker\resources\com.docker.diagnose.exe" gather -upload
```

조치:
- 바이러스 백신/EDR이 `dockerd`/`com.docker.backend` 차단하는지 확인
- Hyper-V/WSL 기능이 비활성화되었는지 확인
- `wsl --shutdown` 후 재시작
- 기업 PC는 Application Control 정책 문의

### 시나리오 B: 포트 충돌(특히 80/443)

```powershell
netstat -ano | findstr :80
tasklist /FI "PID eq <PID>"
```

조치:
- IIS/Skype/내장 HTTP.SYS 사용 중인지 확인
- Compose 포트 변경 또는 점유 프로세스 종료

### 시나리오 C: 프록시/사설 CA 환경

```powershell
# Docker Desktop Settings > Resources > Proxies
# 시스템 프록시 사용 또는 수동 프록시 설정
```

- 사설 CA 사용 시 **Docker Desktop 신뢰 저장소** 또는 WSL 리눅스의 `/usr/local/share/ca-certificates`에 CA 추가 후 `update-ca-certificates` 수행

---

# 2. Linux에서 Docker 설치(확장)

여기서는 **Ubuntu**를 기준으로 하되, 다른 배포판 변형 포인트도 설명합니다.  

## 2.1 기존 패키지 제거(필요 시)

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc
```

> 이전에 `docker.io`(Debian/Ubuntu 커뮤니티 패키지)를 설치했다면, **공식 리포지토리(docker-ce)** 로 교체 권장.

---

## 2.2 공식 리포지토리 설정(Ubuntu)

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# GPG 키
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 저장소 등록
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
```

---

## 2.3 Docker 설치

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

버전 고정이 필요하면 `apt-cache madison docker-ce` 로 버전 확인 후 특정 버전 설치:

```bash
apt-cache madison docker-ce | head -n 10
sudo apt install -y docker-ce=<VERSION> docker-ce-cli=<VERSION> containerd.io
```

---

## 2.4 권한 설정(비 sudo 실행)

```bash
sudo usermod -aG docker $USER
# 세션 재로그인 또는 재부팅 필요
```

검증:

```bash
docker run --rm hello-world
```

---

## 2.5 systemd와 서비스 관리

```bash
# 부팅시 자동 시작
sudo systemctl enable docker
sudo systemctl start docker

# 상태 확인
systemctl status docker
journalctl -u docker --no-pager | tail
```

---

## 2.6 cgroup v2 / AppArmor / SELinux 고려

- Ubuntu/Fedora 등 최신 배포판은 기본 **cgroup v2** 사용.
- 보안 프로파일(AppArmor/SELinux)로 인해 특정 기능이 제한될 수 있음.

예: SELinux(Enforcing)에서 볼륨 마운트 문제 발생 시:

```bash
# (RHEL/CentOS/Alma/Rocky/Fedora 등)
# :z 또는 :Z 옵션으로 컨텍스트 라벨 조정
docker run --rm -v /data:/data:Z alpine ls -al /data
```

---

## 2.7 프록시/미러/사설 레지스트리 설정

### 2.7.1 Docker 데몬 프록시

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
cat <<'EOF' | sudo tee /etc/systemd/system/docker.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://user:pass@proxy.local:3128"
Environment="HTTPS_PROXY=http://user:pass@proxy.local:3128"
Environment="NO_PROXY=localhost,127.0.0.1,.internal,.svc"
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 2.7.2 레지스트리 미러

```bash
sudo mkdir -p /etc/docker
cat <<'EOF' | sudo tee /etc/docker/daemon.json
{
  "registry-mirrors": ["https://mirror.gcr.io", "https://<your-mirror>"]
}
EOF

sudo systemctl restart docker
```

### 2.7.3 사설 레지스트리(인증서)

```bash
# 예) registry.internal:5000 이 사설 CA를 사용한다면
sudo mkdir -p /etc/docker/certs.d/registry.internal:5000
sudo cp rootCA.crt /etc/docker/certs.d/registry.internal:5000/ca.crt
sudo systemctl restart docker
```

---

## 2.8 NVIDIA GPU 사용(선택)

1. 호스트에 NVIDIA 드라이버 설치
2. `nvidia-container-toolkit` 설치

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -fsSL https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list \
 | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
 | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

검증:

```bash
docker run --rm --gpus all nvidia/cuda:12.5.0-base-ubuntu22.04 nvidia-smi
```

---

## 2.9 Compose/Buildx/BuildKit 실전 예시

### 2.9.1 Buildx로 멀티아키 이미지 빌드

```bash
# QEMU 기반 멀티아키 빌드 세팅
docker buildx create --name multi --use
docker buildx inspect --bootstrap

# 예시 Dockerfile 빌드 → amd64 + arm64
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t registry.internal:5000/demo/web:1.0 \
  --push .
```

### 2.9.2 Compose로 로컬 개발 스택

```yaml
# docker-compose.yaml
services:
  api:
    build: ./api
    environment:
      - DB_HOST=db
    ports: ["8081:8080"]
    depends_on: [db]

  web:
    image: nginx:alpine
    volumes:
      - ./site:/usr/share/nginx/html:ro
    ports: ["8080:80"]
    depends_on: [api]

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_PASSWORD=devpass
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```bash
docker compose up -d
curl http://localhost:8080
docker compose down -v
```

---

## 2.10 Linux 문제 해결(대표 시나리오)

### 시나리오 L1: `permission denied`(소켓/볼륨)

- `docker` 그룹 미가입 → `sudo usermod -aG docker $USER` 후 재로그인
- SELinux/AppArmor 라벨 문제 → `:Z`/`:z` 옵션, 또는 정책 확인

### 시나리오 L2: pull 속도 지연

- 레지스트리 미러 설정(`daemon.json`)
- 프록시 환경 변수 점검
- DNS 지연 시 `/etc/resolv.conf` 최적화

### 시나리오 L3: 서비스 기동 실패

```bash
systemctl status docker
journalctl -u docker -b --no-pager
cat /etc/docker/daemon.json
```

- JSON 문법 오류, 포트 충돌, 잘못된 플래그 확인 후 수정

---

# 3. 이미지·컨테이너 기본 예제(설치 검증용)

## 3.1 Hello-World, BusyBox, Alpine

```bash
docker run --rm hello-world
docker run --rm busybox echo "ok"
docker run --rm alpine uname -a
```

## 3.2 바인드 마운트/볼륨

```bash
mkdir -p ~/demo-data
docker run --rm -v ~/demo-data:/data alpine sh -c 'date > /data/now.txt && cat /data/now.txt'

docker volume create d1
docker run --rm -v d1:/var/log alpine sh -c 'touch /var/log/app.log && ls -al /var/log'
```

## 3.3 네트워크/포트

```bash
docker run -d --name web -p 8080:80 nginx:alpine
curl http://localhost:8080
docker rm -f web
```

---

# 4. 기업/학교 환경 실전: 프록시, 사설 CA, 오프라인

## 4.1 시스템 프록시와 Docker 통합

- Windows: Docker Desktop → Settings → Resources → Proxies
- Linux: systemd drop-in(위 2.7.1)으로 `HTTP_PROXY/HTTPS_PROXY/NO_PROXY` 설정

컨테이너 내부 프록시 필요 시:

```bash
docker run -e HTTP_PROXY=http://proxy.local:3128 -e HTTPS_PROXY=http://proxy.local:3128 \
  -e NO_PROXY=localhost,127.0.0.1,.svc \
  curlimages/curl -I https://example.com
```

## 4.2 사설 CA 신뢰

- Windows: Docker Desktop 신뢰 저장소 또는 WSL 배포판에 CA 추가
- Linux: `/usr/local/share/ca-certificates/*.crt` 배치 후 `update-ca-certificates`

## 4.3 오프라인(air-gapped) 설치 시나리오

1. 외부망에서 필요한 **패키지/이미지** 미리 다운로드
2. 오프라인 환경으로 옮겨 설치
3. **사설 레지스트리** 또는 **로컬 타르볼**로 이미지 공급

```bash
# 온라인에서
docker pull nginx:alpine
docker save nginx:alpine -o nginx_alpine.tar

# 오프라인으로 tar 전달 후
docker load -i nginx_alpine.tar
docker run -d -p 8080:80 nginx:alpine
```

사설 레지스트리:

```bash
docker run -d --name reg -p 5000:5000 registry:2
docker tag nginx:alpine localhost:5000/nginx:alpine
docker push localhost:5000/nginx:alpine
```

---

# 5. 성능 최적화 팁(Windows/WSL2 포함)

| 항목 | 권장 설정/설명 |
|------|----------------|
| 파일 I/O | **WSL2 내부 FS 사용**(예: `/home/USER/project`)이 Windows 경로 마운트보다 빠름 |
| 메모리/CPU | `.wslconfig`로 합리적 제한(예: 6~8GB, 4 vCPU) 설정 |
| 이미지 캐시 | 멀티스테이지 빌드, 정확한 `.dockerignore`, 빈번한 레이어 캐시 타격 |
| 네트워킹 | 불필요한 포트포워딩 삭제, 프록시/미러로 pull 가속 |
| 로깅 | 방대한 stdout 로깅 지양, 로테이션 설정 |
| 빌드 | BuildKit/Buildx 사용, 빌드 컨텍스트 최소화 |
| 볼륨 | 빈번한 I/O는 **볼륨** 사용(바인드보다 성능 유리) |

---

# 6. 보안 기본 설정

- 최신 Docker/컨테이너 런타임 유지
- 최소 권한 원칙: `--cap-drop ALL` 뒤 필요한 capability만 `--cap-add`
- 루트리스 모드(리눅스): 필요 시 `rootless` 설치 가이드 참고
- 이미지 서명/검증(Notary v2, cosign 등) 도입 검토
- 비밀정보는 환경변수보다 **외부 시크릿 스토어**(Vault/Parameter Store) 사용 고려
- **네임스페이스/네트워크 분리**, 내부 전용 레지스트리 접근을 네트워크 정책으로 제한

예: 최소 권한 실행 예시

```bash
docker run --rm --read-only --pids-limit=256 --memory=256m --cpus=0.5 \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  -v d1:/app/data:ro \
  nginx:alpine
```

---

# 7. 트러블슈팅 종합 레시피

## 7.1 컨테이너가 즉시 종료됨

{% raw %}
```bash
docker logs <CONTAINER>
docker inspect <CONTAINER> --format '{{.State.ExitCode}}'
```
{% endraw %}

- Entrypoint/Command 오타, 실행 파일 권한 문제, 환경변수 누락 점검

## 7.2 네트워크 연결 불가

```bash
docker network ls
docker inspect <NETWORK>
docker exec -it <CONTAINER> sh -c "ip addr; ip route; nslookup example.com"
```

- DNS, 프록시, 방화벽 정책 확인
- Windows의 경우 Hyper-V/WSL2 NAT 포워딩 상태 점검

## 7.3 볼륨 권한/SELinux 오류

- `:Z`/`:z` 옵션 사용
- 컨테이너 사용자(`--user`)와 볼륨 디렉터리 UID/GID 맞춤

## 7.4 빌드가 느리거나 실패

- `.dockerignore` 확인(대용량 파일/디렉터리 제외)
- BuildKit 활성화

```bash
export DOCKER_BUILDKIT=1
docker build -t demo .
```

---

# 8. 설치 검증을 겸한 실전 예제(미니 프로젝트)

목표: **Nginx 정적 웹 + Python API + PostgreSQL**을 Windows(WSL2) 또는 Linux에서 동일하게 실행.

## 8.1 디렉터리 구조

```
stack/
  web/
    site/index.html
  api/
    app.py
    requirements.txt
    Dockerfile
  docker-compose.yaml
```

## 8.2 파일 내용

### 8.2.1 `web/site/index.html`

```html
<!doctype html>
<html>
  <head><meta charset="utf-8"><title>Docker Demo</title></head>
  <body><h1>OK</h1><p>Static served by nginx.</p></body>
</html>
```

### 8.2.2 `api/app.py`

```python
from flask import Flask, jsonify
import os
app = Flask(__name__)

@app.get("/health")
def health():
    return jsonify(ok=True)

@app.get("/echo")
def echo():
    return jsonify(env=dict(os.environ))

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

### 8.2.3 `api/requirements.txt`

```
flask==3.0.3
```

### 8.2.4 `api/Dockerfile`

```dockerfile
FROM python:3.12-alpine
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 8080
CMD ["python", "app.py"]
```

### 8.2.5 `docker-compose.yaml`

```yaml
services:
  web:
    image: nginx:alpine
    volumes:
      - ./web/site:/usr/share/nginx/html:ro
    ports: ["8080:80"]
    depends_on: [api]

  api:
    build: ./api
    environment:
      - DB_HOST=db
      - DB_USER=demo
      - DB_PASSWORD=demo
    ports: ["8081:8080"]
    depends_on: [db]

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_PASSWORD=demo
      - POSTGRES_USER=demo
      - POSTGRES_DB=demo
    volumes:
      - dbdata:/var/lib/postgresql/data

volumes:
  dbdata:
```

## 8.3 실행/검증

```bash
docker compose up -d --build
curl http://localhost:8080
curl http://localhost:8081/health
docker compose logs -f
docker compose down -v
```

> Windows(WSL2)에서는 프로젝트를 **WSL2 리눅스 홈 디렉터리**에 두고 실행하면 파일 I/O 성능이 좋습니다.

---

# 9. 자주 묻는 질문(FAQ)

**Q1. Windows Home인데 Hyper-V가 없어도 되나요?**  
A. 네. **WSL2 기반 엔진**으로 충분합니다. Docker Desktop 설치 시 WSL2를 선택하세요.

**Q2. 회사 PC에서 이미지 pull이 안 됩니다.**  
A. 프록시/사설 CA 환경일 수 있습니다. **Docker 프록시 설정**과 **CA 신뢰**를 구성하세요.

**Q3. Windows에서 파일 변경이 컨테이너 내에 늦게 반영됩니다.**  
A. Windows 파일시스템을 바인드하면 I/O가 느릴 수 있습니다. **WSL2 내부 FS**로 프로젝트를 옮기세요.

**Q4. 오프라인 서버에 설치하려면?**  
A. 외부망에서 **이미지 tar**로 저장해 가져오거나, **사설 레지스트리**를 구축하세요.

**Q5. GPU를 쓰고 싶습니다.**  
A. 리눅스는 `nvidia-container-toolkit`를 설치, Windows/WSL2는 NVIDIA CUDA on WSL 가이드를 따르세요.

---

# 10. 결론

- **Windows**: Home은 WSL2, Pro는 Hyper-V와 WSL2 중 **WSL2 권장**  
- **Linux(Ubuntu)**: 공식 리포지토리에서 설치, `docker` 그룹 권한, systemd 관리  
- **기업·프록시·오프라인** 환경까지 고려한 **프록시/CA/레지스트리/미러** 설정 제공  
- **성능·보안·트러블슈팅** 체크리스트로 실무 대응력 강화

---

# 부록 A. 명령어 치트시트

```bash
# 시스템
docker version
docker info
docker system df
docker system prune -f

# 이미지
docker images
docker pull <image>
docker rmi <image>
docker save -o img.tar <image>
docker load -i img.tar

# 컨테이너
docker ps -a
docker run -d --name c1 -p 8080:80 nginx:alpine
docker logs -f c1
docker exec -it c1 sh
docker cp c1:/etc/nginx/nginx.conf .
docker rm -f c1

# 네트워크/볼륨
docker network ls
docker volume ls
docker volume inspect <vol>
docker network inspect <net>

# Compose
docker compose up -d
docker compose ps
docker compose logs -f
docker compose down -v
```

---

# 부록 B. 리눅스 배포판별 설치 힌트(요약)

- **Debian**: Ubuntu와 유사. codename(`bookworm`, `bullseye`) 주의  
- **Fedora**: `dnf config-manager`로 리포 등록 후 `dnf install docker-ce ...`  
- **RHEL/Alma/Rocky**: SELinux 라벨, 방화벽, FIPS 모드 호환성 고려  
- **Amazon Linux**: `amazon-linux-extras` 또는 공식 리포지토리 선택  
- **Arch**: `pacman -S docker docker-compose` (Rolling, 문서 확인)

---

# 부록 C. 설치 후 유틸리티

```bash
# dive: 이미지 레이어 분석(빌드 최적화)
# https://github.com/wagoodman/dive
dive <image>

# trivy: 이미지 취약점 스캐너
# https://github.com/aquasecurity/trivy
trivy image nginx:alpine
```

---

# 참고 링크

- 공식 설치 가이드: https://docs.docker.com/get-docker/  
- Docker Desktop (Windows): https://www.docker.com/products/docker-desktop/  
- NVIDIA Container Toolkit: https://docs.nvidia.com/datacenter/cloud-native/