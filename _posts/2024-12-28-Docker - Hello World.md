---
layout: post
title: Docker - Hello World
date: 2024-12-28 22:20:23 +0900
category: Docker
---
# 첫 번째 Docker 컨테이너 실행하기: Hello World

## 실행 전 핵심 체크(요약)

- Docker Desktop(Windows/Mac) 또는 Docker Engine(Linux)이 **실행 중**이어야 합니다.
- Windows Home은 **WSL2 엔진**이 필요합니다.
- 회사/학교 망은 **프록시/사설 CA** 영향으로 pull 실패가 날 수 있습니다.
- 첫 실습은 **`docker run hello-world`** 로 진행하고, 이후 **동작 확인/분석/정리**를 순서대로 수행합니다.

---

# Docker 데몬이 실행 중인지 확인

## 공통(Windows PowerShell / WSL2 / Linux)

```bash
docker version
```

정상 출력 예시(발췌):
```
Client: Docker Engine - Community
Server: Docker Engine - Community
```

추가 확인:
```bash
docker info
```
- `Server` 섹션이 보이고, 오류가 없다면 데몬이 살아 있습니다.

## Windows에서 자주 겪는 상황

- Docker Desktop 트레이 아이콘이 **실행 중**이어야 합니다.
- WSL2 엔진 사용 시, 필요하면 다음으로 상태 확인:
```powershell
wsl --status
# 중단 후 재가동

wsl --shutdown
```

---

# Hello World 이미지 다운로드 및 실행

## 가장 기본

```bash
docker run hello-world
```

## 내부에서 실제로 일어나는 일(요약 흐름)

1. **이미지 조회**: 로컬에 `hello-world:latest` 가 있는지 확인
2. **이미지 풀**: 없으면 **Docker Hub**에서 `pull`
3. **컨테이너 생성**: 해당 이미지 기반으로 컨테이너 **Create**
4. **컨테이너 시작**: `Start` 후 **프로세스 실행**
5. **출력 후 종료**: “Hello from Docker!” 메시지 출력 → **Exit(0)**
6. **상태 보존**: 컨테이너는 종료 상태로 남음(`Exited`)

실행 흐름을 한 줄로 요약하면,
```text
run = pull(if needed) + create + start + attach(outputs) + wait(Exit 0)
```

---

# 실행 결과와 검증

## 기대 출력

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```
이 메시지가 보이면 **설치와 기본 실행이 정상**입니다.

## 종료된 컨테이너 확인

```bash
docker ps -a
```
예시:
```
CONTAINER ID   IMAGE         COMMAND   STATUS                      NAMES
e98dbd4c5cc5   hello-world   "/hello"  Exited (0) 10 seconds ago   dreamy_lalande
```
- `Exited (0)` 은 정상 종료를 의미합니다.

## 이미지 확인

```bash
docker images
```
예시:
```
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
hello-world   latest    d1165f2...     2 weeks ago    13.3kB
```

---

# 첫 실습을 다양한 각도에서 반복하기

## 재실행: 캐시 확인과 `--rm`

처음과 달리, 두 번째 실행은 **이미지를 다시 받지 않습니다**(이미 로컬에 존재).
```bash
# 컨테이너 생명주기 후 흔적 없이 정리

docker run --rm hello-world
```
- `--rm` 은 컨테이너가 종료되면 즉시 삭제합니다(이미지에는 영향 없음).

## 명시적 pull → run

```bash
docker pull hello-world:latest
docker run --rm hello-world:latest
```

## digest 고정 실행(정확성/재현성 강화)

{% raw %}
```bash
docker pull hello-world:latest
docker inspect --format='{{index .RepoDigests 0}}' hello-world:latest
# 출력 예: hello-world@sha256:abcdef...

docker run --rm hello-world@sha256:abcdef...   # digest로 고정 실행
```
{% endraw %}
- **태그(tag)** 는 바뀔 수 있으나, **다이제스트(digest)** 는 내용 기반이므로 배포 재현성에 유리합니다.

---

# 컨테이너/이미지 내부 관찰과 이해

## 컨테이너 로그·상태·세부 정보

```bash
# 마지막 실행 컨테이너 이름/ID를 사용

docker logs e98dbd4c5cc5
docker inspect e98dbd4c5cc5 | jq '.[0].State, .[0].Config.Image, .[0].HostConfig.NetworkMode'
```

## 이미지 레이어 구조 관찰

```bash
docker history hello-world:latest
docker image inspect hello-world:latest | jq '.[0].RootFS.Layers'
```
- 레이어(layer) 기반의 **불변 이미지** 개념을 체감할 수 있습니다.

---

# 정리 명령어(옵션)

```bash
# 종료된 컨테이너 삭제(여러 개면 다수 지정 가능)

docker rm <컨테이너ID_OR_NAME>

# 이미지 삭제

docker rmi hello-world:latest
```

주의: 다른 컨테이너가 사용 중이면 이미지 삭제가 거절됩니다.

---

# hello-world 이후: 한 단계씩 확장 실습

Hello World는 1회성 출력으로 끝납니다. **컨테이너 운영의 감을 잡기 위해** 다음 실습을 권장합니다.

## BusyBox/Alpine로 셸 실행

```bash
# BusyBox로 간단 출력

docker run --rm busybox echo "ok"

# Alpine에서 인터랙티브 셸

docker run --rm -it alpine:3.20 sh
# 내부에서

uname -a
ls -al /
exit
```
- **이미지 다양성**과 **인터랙티브 실행**을 경험합니다.

## 포트 매핑(웹 서버 기초)

```bash
docker run -d --name web -p 8080:80 nginx:alpine
curl http://localhost:8080
docker rm -f web
```
- `-p 호스트:컨테이너` 로 **호스트 포트 노출** 원리를 체험합니다.

## 볼륨/바인드 마운트(파일 공유)

```bash
mkdir -p ~/demo-data
echo "hello" > ~/demo-data/index.html

docker run -d --name web \
  -p 8080:80 \
  -v ~/demo-data:/usr/share/nginx/html:ro \
  nginx:alpine

curl http://localhost:8080
docker rm -f web
```
- 호스트 파일을 컨테이너에 제공하여 **상태/콘텐츠 관리**를 학습합니다.

## 네트워크(컨테이너 간 통신)

```bash
docker network create appnet

docker run -d --name web --network appnet nginx:alpine
docker run --rm --network appnet curlimages/curl http://web
docker rm -f web
docker network rm appnet
```
- 동일 네트워크의 컨테이너 간에는 **컨테이너 이름으로 DNS 해석**이 됩니다.

---

# 운영환경/회사망에서 흔한 이슈와 해결

## 프록시/사설 CA로 인한 pull 실패

증상:
```
Error response from daemon: Get "https://registry-1.docker.io/v2/": ...
```
대응:
- Docker Desktop(Settings) 또는 Linux(systemd drop-in)에서 **HTTP(S)_PROXY** 지정
- 사설 인증서를 Docker가 신뢰하도록 **CA 등록**
- 컨테이너 내부에서 프록시를 사용해야 하면 `-e HTTP_PROXY=...` 로 환경변수 전달

예(리눅스 데몬 프록시 설정 스니펫):
```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
cat <<'EOF' | sudo tee /etc/systemd/system/docker.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy.local:3128"
Environment="HTTPS_PROXY=http://proxy.local:3128"
Environment="NO_PROXY=localhost,127.0.0.1,.internal"
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 권한/소켓 접근 문제

증상:
```
Got permission denied while trying to connect to the Docker daemon socket...
```
대응:
```bash
# (리눅스) 유저를 docker 그룹에 추가 후 재로그인

sudo usermod -aG docker $USER
```

## 포트 충돌

- 이미 80/443/8080 등을 점유한 프로세스가 있으면 매핑 실패
```bash
# Linux/macOS

lsof -i :8080
# Windows

netstat -ano | findstr :8080
```
- 다른 포트로 변경하거나 점유 프로세스 종료

## 네임 리졸브/DNS 이슈

```bash
docker run --rm curlimages/curl -I https://example.com
# 실패 시 호스트 DNS/프록시/방화벽 규칙 확인

```

---

# Hello World를 Compose로 실행(로컬 배포 맛보기)

Compose는 복수 컨테이너 정의/실행 도구입니다. Hello World 자체는 단일 컨테이너지만, **도구 사용 흐름**을 빠르게 익힙니다.

```yaml
# docker-compose.yaml

services:
  hello:
    image: hello-world
```

```bash
docker compose up
docker compose ps
docker compose down
```

---

# 정확성·재현성 강화: 태그 vs 다이제스트

- **태그(tag)** 는 사람이 읽기 좋은 이름(예: `latest`, `1.2`). 새 빌드가 같은 태그로 갱신될 수 있습니다.
- **다이제스트(digest)** 는 이미지 내용의 해시(sha256).
  동일 digest는 **내용 동일**을 보장합니다.

실습:
{% raw %}
```bash
docker pull alpine:3.20
docker inspect --format='{{index .RepoDigests 0}}' alpine:3.20
docker run --rm alpine@sha256:... uname -a
```
{% endraw %}
- CI/CD나 보안 민감 환경에서는 **digest 고정**을 권장합니다.

---

# 내부 메커니즘을 한눈에(텍스트 다이어그램)

```text
+----------------------- Docker CLI -----------------------+
|  docker run hello-world                                 |
+---------------------+-------------------+----------------+
                      |                   |
                      v                   |
                 Docker API               |
                      |                   |
                 +----v-------------------v-----+
                 |           dockerd            |
                 |  (이미지 관리, 컨테이너 수명)|
                 +----+-------------------+-----+
                      |                   |
      pull(if needed) |                   | create/start
                      v                   v
            +---------+---------+   +-----+----------------+
            | Local Image Store |   |   containerd/runc   |
            +---------+---------+   +-----+----------------+
                      |                   |
                      | merged FS         | 프로세스 격리
                      v                   v
                +-----+-------------------+-----+
                |       Container (PID,net,fs)   |
                +--------------------------------+
                            stdout/exit
```

---

# Hello World 이후 반드시 해볼 추가 과제

## 과제 A: 컨테이너 수명과 로그 파이프라인

```bash
docker run --name short --rm busybox sh -c 'echo A; echo B 1>&2; exit 0'
# stdout/stderr 분리 수집 연습

docker logs short  # --rm이므로 실행과 동시에 삭제되었을 수 있음
```
- 로그 수집기의 stdout/stderr 처리 차이를 이해합니다.

## 과제 B: 리소스 제한 기초

```bash
docker run --rm --cpus=0.5 --memory=128m alpine:3.20 sh -c 'dd if=/dev/zero of=/dev/null count=1000000'
```
- cgroups 기반의 자원 제어 개념을 체감합니다.

## 과제 C: 읽기 전용 루트 파일시스템

```bash
docker run --rm --read-only alpine:3.20 sh -c 'touch /tmp/x || echo "fail"; mkdir -p /run && touch /run/y || true'
```
- 애플리케이션이 쓰기 필요한 경로만 **tmpfs** 로 제공하는 습관을 익힙니다.

---

# 수학적 메모: 캐시 적중 직관(선택)

간단한 직관을 수식으로 적어 보면, Dockerfile 레이어 캐시 적중률을 $$p_i$$ 로, 각 레이어 빌드 비용을 $$c_i$$ 라고 할 때, 기대 빌드 시간의 1차 근사는
$$
\mathbb{E}[T] \approx \sum_{i=1}^{n} (1-p_i)\,c_i
$$
입니다.
변경 가능성이 낮은 레이어(의존성 설치)를 앞에 배치해 $$p_i$$ 를 크게 만들면, 전체 $$\mathbb{E}[T]$$ 가 줄어드는 효과를 얻습니다.

---

# 자주 묻는 질문(FAQ)

**Q1. `docker run hello-world` 가 아무 출력 없이 멈춥니다.**
A. 레지스트리 접속이 지연되거나 프록시/방화벽 문제일 수 있습니다. `docker pull hello-world` 로 먼저 시도하고, 프록시/CA 설정을 점검합니다.

**Q2. Windows인데 파일 변경이 컨테이너에 천천히 반영됩니다.**
A. WSL2 환경에서는 프로젝트를 **WSL2 내부 경로**(예: `/home/<user>/project`)에 두고 바인드 마운트하면 I/O 속도가 훨씬 낫습니다.

**Q3. 컨테이너가 바로 종료됩니다.**
A. 정상입니다. hello-world는 메시지 출력 후 종료하도록 설계되어 있습니다. 장기 실행 테스트는 `nginx:alpine` 같은 이미지를 사용하세요.

**Q4. 이미지가 너무 많습니다. 용량을 줄이고 싶습니다.**
A. 미사용 자원 정리에 유의하여 아래처럼 실행하세요:
```bash
docker system df
docker system prune -f      # 주의: 필요 자원이 삭제될 수 있음
```

---

# 요약 명령어 모음(필수)

| 작업 | 명령어 |
|------|--------|
| 컨테이너 실행 | `docker run hello-world` |
| 컨테이너 목록(종료 포함) | `docker ps -a` |
| 이미지 목록 | `docker images` |
| 컨테이너 삭제 | `docker rm <id>` |
| 이미지 삭제 | `docker rmi hello-world` |

보강:
{% raw %}
```bash
# 정확성: digest 고정

docker inspect --format='{{index .RepoDigests 0}}' hello-world:latest

# 내부 관찰

docker history hello-world:latest
docker image inspect hello-world:latest | jq '.[0].RootFS.Layers'
```
{% endraw %}
