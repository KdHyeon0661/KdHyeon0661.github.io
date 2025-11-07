---
layout: post
title: Docker - 첫 번째 Docker 컨테이너
date: 2025-01-07 19:20:23 +0900
category: Docker
---
# 첫 번째 Docker 컨테이너 실행하기: Hello World

## 0. 실행 전 체크리스트(요약)

- Docker Desktop(Windows/macOS) 또는 Docker Engine(Linux)이 **실행 중**이어야 합니다.
- Windows Home은 **WSL2 엔진**이 필요합니다.
- 회사/학교 망은 **프록시/사설 CA**로 인해 `pull` 오류가 날 수 있습니다.
- 첫 실습은 `docker run hello-world`, 그다음 `ps / logs / inspect` 순으로 확인합니다.

---

## 1. Docker 데몬이 실행 중인지 확인

### 1.1 공통(Windows PowerShell / Bash / WSL)
```bash
docker version
```

정상 출력의 핵심:
```
Client: Docker Engine - Community
Server: Docker Engine - Community
```

추가 진단:
```bash
docker info
# 문제 시: 에러 메시지(소켓 권한/데몬 미기동/프로시)로 원인 파악
```

### 1.2 Windows에서 자주 쓰는 점검
```powershell
# WSL2 엔진 상태 확인
wsl --status
# WSL2 환경 리셋(캐시/리소스 갱신)
wsl --shutdown
```
- Docker Desktop 트레이 아이콘이 실행 중인지 확인합니다.
- Hyper-V/WSL 기능이 비활성화되어 있으면 Docker Desktop이 자동 활성화/재부팅을 요구합니다.

---

## 2. Hello World 이미지 다운로드 및 실행

### 2.1 한 줄로 실행
```bash
docker run hello-world
```

### 2.2 내부에서 일어나는 일(요약 흐름)
1. **이미지 조회**: 로컬에 `hello-world:latest` 존재 확인  
2. **이미지 pull**: 없으면 Docker Hub에서 다운로드  
3. **컨테이너 create**: 이미지 기반으로 컨테이너 생성  
4. **컨테이너 start**: 실행 후 표준 출력으로 메시지 출력  
5. **종료**: 메시지 출력 후 프로세스가 종료(Exit 0), 컨테이너는 `Exited` 상태로 남음  

실행 과정을 식으로 간단히 적으면,
```text
run = pull(if needed) + create + start + attach + wait(Exit 0)
```

---

## 3. 실행 결과 예시와 의미

### 3.1 기대 출력
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```
- 이 메시지가 보이면 **엔진/네트워크/이미지/컨테이너 경로 전체가 정상**입니다.

### 3.2 종료 상태 확인
```bash
docker ps -a
```
예시:
```
CONTAINER ID   IMAGE         COMMAND     STATUS                      NAMES
e98dbd4c5cc5   hello-world   "/hello"    Exited (0) 10 seconds ago   dreamy_lalande
```
- `Exited (0)` 은 정상 종료를 의미합니다. hello-world는 본질적으로 **일회성** 컨테이너입니다.

---

## 4. 이미지/컨테이너 세부 관찰

### 4.1 이미지 목록
```bash
docker images
```
예시:
```
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
hello-world   latest    d1165f2...     2 weeks ago    13.3kB
```

### 4.2 로그/상세/포트
```bash
docker logs e98dbd4c5cc5
docker inspect e98dbd4c5cc5 | jq '.[0].State, .[0].Config.Image'
```
- `logs` 로 표준 출력 확인  
- `inspect` 로 상태/네트워크/마운트/환경변수 등 **모든 메타데이터** 확인

---

## 5. 정리 명령어(옵션)

```bash
# 종료된 컨테이너 삭제
docker rm <컨테이너_ID_or_NAME>

# 이미지 삭제
docker rmi hello-world
```

주의: 다른 컨테이너가 해당 이미지를 참조 중이면 삭제가 거절됩니다.

---

## 6. Hello World 이후: 한 단계씩 확장 실습

### 6.1 재실행과 `--rm` 옵션
```bash
docker run --rm hello-world
```
- 컨테이너가 종료되면 **자동으로 컨테이너를 삭제**합니다(이미지는 유지).

### 6.2 BusyBox/Alpine로 셸 체험
```bash
docker run --rm busybox echo "ok"
docker run --rm -it alpine:3.20 sh
# 컨테이너 내부에서 몇 가지 명령 실행 후 'exit'
```

### 6.3 포트 매핑(웹 서버 기초)
```bash
docker run -d --name web -p 8080:80 nginx:alpine
curl http://localhost:8080
docker rm -f web
```

### 6.4 볼륨/바인드 마운트(파일 공유)
```bash
mkdir -p ~/demo && echo "hi" > ~/demo/index.html
docker run -d --name web -p 8080:80 \
  -v ~/demo:/usr/share/nginx/html:ro \
  nginx:alpine
curl http://localhost:8080
docker rm -f web
```

### 6.5 네트워크(컨테이너 간 통신)
```bash
docker network create appnet
docker run -d --name web --network appnet nginx:alpine
docker run --rm --network appnet curlimages/curl http://web
docker rm -f web
docker network rm appnet
```

---

## 7. 정확성·재현성 강화: digest 고정

태그는 가변, **digest** 는 이미지 내용 해시로 **불변**입니다. 재현 가능한 실행을 원한다면 digest 고정이 유리합니다.

```bash
docker pull hello-world:latest
docker inspect --format='{{index .RepoDigests 0}}' hello-world:latest
# 출력 예: hello-world@sha256:abcdef...
docker run --rm hello-world@sha256:abcdef...
```

---

## 8. 보안 옵션으로 한 번 더 실행

hello-world는 간단하지만, **보안 습관**을 같이 익혀두면 좋습니다.

```bash
docker run --rm \
  --read-only \
  --pids-limit 64 \
  --cpus 0.5 \
  --memory 64m \
  hello-world
```
- `--read-only`: 루트 파일 시스템을 읽기 전용  
- `--pids-limit`: 생성 가능한 프로세스 수 제한  
- `--cpus`, `--memory`: cgroups 기반 자원 제한

---

## 9. 자주 겪는 문제와 해결

| 증상/오류 | 원인 | 해결 |
|---|---|---|
| `permission denied while trying to connect to the Docker daemon socket` | 리눅스에서 도커 소켓 권한 부족 | `sudo usermod -aG docker $USER` 후 재로그인 |
| `Get https://registry-1.docker.io/v2/: ...` | 프록시/사설 CA로 pull 실패 | 데몬 프록시/CA 설정(아래 예), 컨테이너 내부 프록시 `-e HTTP(S)_PROXY` |
| 포트 바인딩 실패 | 호스트 포트 충돌 | `lsof -i :8080` 또는 `netstat -ano | findstr :8080` 로 점유 확인 후 포트 변경/종료 |
| Windows에서 파일 변경 반영 느림 | 호스트↔WSL 경계 I/O 병목 | 프로젝트를 **WSL2 내부 경로**에서 빌드/마운트 |
| 컨테이너 즉시 종료 | hello-world 특성(일회성) | 정상. 장기 실행 테스트는 `nginx:alpine` 등 사용 |

프록시/사설 CA(리눅스 데몬 예):
```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
cat <<'EOF' | sudo tee /etc/systemd/system/docker.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy.local:3128"
Environment="HTTPS_PROXY=http://proxy.local:3128"
Environment="NO_PROXY=localhost,127.0.0.1,.internal,.svc"
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## 10. Compose로 “Hello” 맛보기(도구 사용 흐름 익히기)

Hello World 자체는 단일 컨테이너지만, Compose 명령 흐름을 **빠르게 경험**합니다.

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

## 11. 내부 동작을 한눈에(텍스트 다이어그램)

```text
+----------------------- Docker CLI -----------------------+
| docker run hello-world                                  |
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

## 12. 수학적 메모(선택): 캐시 적중 직관

Dockerfile 레이어별 캐시 적중률을 \(p_i\), 빌드 비용을 \(c_i\) 라고 하면 기대 빌드 시간의 근사는
$$
\mathbb{E}[T] \approx \sum_{i=1}^{n}(1-p_i)\,c_i
$$
입니다. 변경이 적은 레이어(의존성 설치)를 앞쪽에 배치해 \(p_i\) 를 크게 만들면 전체 빌드 시간이 줄어듭니다.

---

## 13. 확장 실습 시나리오 3선

### 13.1 “조금 더” 복습: busybox/alpine/ubuntu 비교
```bash
docker run --rm busybox sh -c 'echo BB && uname -a'
docker run --rm alpine  sh -c 'echo AL && uname -a'
docker run --rm ubuntu  bash -lc 'echo UB && uname -a'
```
- 기본 셸/패키지 관리자 차이, 이미지 크기 차이를 체감합니다.

### 13.2 Nginx 정적 페이지 제공
```bash
mkdir -p site && echo "hello" > site/index.html
docker run -d --name web -p 8080:80 \
  -v $(pwd)/site:/usr/share/nginx/html:ro \
  nginx:alpine
curl http://localhost:8080
docker rm -f web
```

### 13.3 내부 셸과 패키지 설치
```bash
docker run -it --name a1 alpine:3.20 sh
# 컨테이너 내부에서
apk add --no-cache curl
curl -I https://example.com
exit
docker rm a1
```

---

## 14. 요약 명령어 정리(보강판)

| 작업 | 기본 | 보강 |
|---|---|---|
| 컨테이너 실행 | `docker run hello-world` | `--rm`, digest 고정 `image@sha256:...` |
| 컨테이너 목록 | `docker ps -a` | `docker inspect`, `docker stats`, `docker top` |
| 이미지 목록 | `docker images` | `docker history`, `docker image inspect` |
| 로그 확인 | `docker logs <name>` | `-f`, `--since 10m`, `--tail 100` |
| 내부 접속 | `docker exec -it <name> sh|bash` | `docker cp`, `docker port` |
| 정지/삭제 | `docker stop <name>` / `docker rm <name>` | `docker rm -f`, `docker rmi <image>` |
| 청소 | — | `docker system df`, `docker system prune -f` |
| Compose | `docker compose up/down` | `docker compose ps/logs` |

---

## 15. 결론 및 다음 단계

- `docker run hello-world` 는 **설치·네트워크·이미지·컨테이너** 경로가 정상임을 입증하는 **가장 작은 실습**입니다.
- 뒤이어 `ps / logs / inspect` 로 내부 정보를 확인하고, `nginx`·`alpine` 등으로 **포트/볼륨/네트워크**를 체험하면 운영에 가까운 감을 빠르게 얻습니다.
- 재현성(다이제스트), 보안(읽기 전용·권한 최소화), 문제 해결(프록시/권한/포트/DNS) 습관을 초기부터 같이 들이면 이후 확장(Compose → Kubernetes)도 자연스럽습니다.