---
layout: post
title: Docker - Docker 기본 명령어 정리
date: 2024-12-30 19:20:23 +0900
category: Docker
---
# Docker 기본 명령어 정리 — 흐름, 옵션, 실전 팁

이 문서는 Docker를 사용하는 핵심 흐름을 중심으로 기본 명령어와 필수 옵션, 실전 팁을 정리합니다. 이미지를 다운로드하고 컨테이너를 실행하며 상태를 확인하는 전 과정을 쉽게 따라할 수 있도록 구성했습니다.

## Docker 핵심 흐름 이해

Docker를 사용하는 기본적인 흐름은 다음과 같습니다.

1.  **이미지 가져오기**: 애플리케이션 실행에 필요한 패키지를 레지스트리에서 다운로드합니다.
2.  **컨테이너 실행**: 이미지를 기반으로 격리된 환경(컨테이너)을 생성하고 실행합니다.
3.  **상태 관리**: 실행 중인 컨테이너의 상태를 확인하고 로그를 조회합니다.
4.  **생명주기 관리**: 컨테이너를 정지하거나 삭제하여 정리합니다.

이 흐름에 따라 각 단계별로 가장 자주 사용하는 명령어와 옵션을 알아보겠습니다.

---

## 1. 이미지 다운로드: `docker pull`

레지스트리(Docker Hub 등)에서 이미지를 로컬로 가져옵니다.

### 기본 사용법
```bash
docker pull [레지스트리/저장소/]이미지[:태그]
```

### 예시
```bash
docker pull ubuntu:20.04                # 특정 버전의 Ubuntu 이미지
docker pull nginx                       # 태그 생략 시 'latest' 태그 자동 적용
docker pull ghcr.io/myorg/myapp:1.2.3   # GitHub Container Registry에서 가져오기
```

### 확인 및 팁
```bash
# 다운로드된 이미지 목록 확인
docker images

# 특정 이미지의 상세 정보 확인 (예: 다이제스트)
docker image inspect nginx:latest | jq '.[0].RepoDigests'
```
**팁**: 태그(예: `latest`)는 변경될 수 있지만, 다이제스트(예: `sha256:...`)는 이미지 내용의 고유 해시값으로 항상 동일합니다. 운영 환경에서는 특정 다이제스트를 사용하여 재현성을 보장할 수 있습니다.

---

## 2. 컨테이너 실행: `docker run`

이미지를 기반으로 새로운 컨테이너를 생성하고 실행합니다. 가장 다양하고 중요한 옵션을 갖는 명령어입니다.

### 기본 형식
```bash
docker run [옵션] 이미지[:태그|@다이제스트] [명령어]
```

### 필수 옵션 분류 및 예제

#### 실행 모드 관련
- **`-it`**: 대화형 모드로 실행 (터미널 입력 가능 + TTY 할당)
  ```bash
  docker run -it ubuntu:20.04 bash  # 컨테이너 내부의 bash 셸에 접속
  ```
- **`-d`**: 백그라운드(데몬) 모드로 실행
  ```bash
  docker run -d --name webserver nginx  # 백그라운드에서 Nginx 실행
  ```
- **`--rm`**: 컨테이너 종료 시 자동 삭제
  ```bash
  docker run --rm alpine echo "Hello, Docker!"  # 명령 실행 후 컨테이너 자동 정리
  ```

#### 네트워킹 관련
- **`-p HOST_PORT:CONTAINER_PORT`**: 포트 매핑
  ```bash
  docker run -d -p 8080:80 nginx  # 호스트의 8080포트를 컨테이너의 80포트로 연결
  ```
- **`--network NETWORK_NAME`**: 사용자 정의 네트워크 사용
  ```bash
  docker network create mynetwork
  docker run -d --network mynetwork --name app1 myapp:latest
  ```

#### 저장소 관련
- **`-v HOST_PATH:CONTAINER_PATH[:옵션]`**: 볼륨 마운트
  ```bash
  # 호스트 디렉터리를 컨테이너에 마운트
  docker run -v /home/user/data:/app/data:ro myapp  # 읽기 전용(:ro) 옵션
  ```

#### 리소스 관리 관련
- **`--memory`**: 메모리 사용량 제한
  ```bash
  docker run --memory="512m" myapp  # 512MB로 제한
  ```
- **`--cpus`**: CPU 사용량 제한
  ```bash
  docker run --cpus="1.5" myapp  # 1.5개 코어 사용 제한
  ```

#### 보안 관련
- **`--user UID:GID`**: 특정 사용자로 실행
  ```bash
  docker run --user 1000:1000 myapp  # UID 1000, GID 1000 사용자로 실행
  ```
- **`--read-only`**: 루트 파일시스템을 읽기 전용으로 마운트
  ```bash
  docker run --read-only --tmpfs /tmp alpine  # /tmp만 쓰기 가능
  ```

### 다이제스트를 사용한 정확한 버전 실행
```bash
# 다이제스트 확인
docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine
# 출력 예: nginx@sha256:abcdef123456...

# 다이제스트로 컨테이너 실행 (항상 동일한 이미지 보장)
docker run --rm nginx@sha256:abcdef123456...
```

---

## 3. 컨테이너 상태 확인 및 관리

### 실행 중인 컨테이너 목록 확인
```bash
docker ps        # 실행 중인 컨테이너만 표시
docker ps -a     # 모든 컨테이너 표시 (중지된 것 포함)
docker ps -aq    # 모든 컨테이너의 ID만 표시
```

### 컨테이너 로그 확인
```bash
docker logs [컨테이너 이름 또는 ID]          # 기본 로그 출력
docker logs -f [컨테이너]                   # 실시간 로그 스트림 확인
docker logs --tail=50 [컨테이너]            # 최근 50줄만 확인
docker logs --since=10m [컨테이너]          # 최근 10분간의 로그 확인
```

### 실행 중인 컨테이너에 접속하기
```bash
docker exec -it [컨테이너 이름] sh    # 또는 bash
```
**참고**: `docker attach`는 주 프로세스(PID 1)의 입출력에 연결하므로, 대부분의 경우 `docker exec`가 더 안전하고 유연합니다.

### 컨테이너 리소스 사용량 확인
```bash
docker stats                   # 모든 컨테이너의 실시간 리소스 사용량
docker stats [컨테이너 이름]    # 특정 컨테이너만 확인
docker top [컨테이너 이름]      # 컨테이너 내부의 프로세스 목록
```

### 컨테이너 상세 정보 조회
```bash
docker inspect [컨테이너 이름]
# JSON 형식의 상세 정보 출력. jq와 함께 사용하면 특정 정보만 추출 가능
docker inspect [컨테이너] | jq '.[0].NetworkSettings.IPAddress'
```

---

## 4. 컨테이너 정지 및 삭제

### 컨테이너 정지, 시작, 재시작
```bash
docker stop [컨테이너 이름]     # 정상 종료 요청 (SIGTERM)
docker kill [컨테이너 이름]     # 강제 종료 (SIGKILL)
docker start [컨테이너 이름]    # 중지된 컨테이너 시작
docker restart [컨테이너 이름]  # 컨테이너 재시작
```

### 컨테이너 삭제
```bash
docker rm [컨테이너 이름]          # 중지된 컨테이너 삭제
docker rm -f [컨테이너 이름]       # 실행 중인 컨테이너 강제 삭제
docker rm $(docker ps -aq)        # 모든 컨테이너 삭제 (주의)
```

### 이미지 삭제
```bash
docker rmi [이미지 이름:태그]      # 특정 이미지 삭제
docker rmi $(docker images -q)   # 모든 이미지 삭제 (주의)
```

### 시스템 정리
```bash
docker system df                  # Docker 디스크 사용량 확인
docker system prune               # 사용하지 않는 모든 Docker 객체 정리
docker system prune -a            # 사용하지 않는 이미지까지 모두 정리 (주의)
```

---

## 5. 실전 활용 예제

### 전체 흐름 한번에 따라하기
```bash
# 1. 이미지 다운로드
docker pull nginx:alpine

# 2. 컨테이너 실행
docker run -d --name myweb -p 8080:80 nginx:alpine

# 3. 상태 확인
docker ps
curl http://localhost:8080

# 4. 로그 확인 및 컨테이너 내부 탐색
docker logs myweb
docker exec -it myweb sh -c "nginx -v; ls -la /usr/share/nginx/html"

# 5. 정리
docker stop myweb
docker rm myweb
docker rmi nginx:alpine
```

### 파일 복사하기
```bash
# 호스트 → 컨테이너
docker cp ./index.html myweb:/usr/share/nginx/html/

# 컨테이너 → 호스트  
docker cp myweb:/etc/nginx/nginx.conf ./nginx-backup.conf
```

### 네트워크 및 볼륨 관리
```bash
# 네트워크 생성 및 사용
docker network create app-network
docker run -d --name api --network app-network myapi:latest

# 볼륨 생성 및 사용
docker volume create app-data
docker run -d --name db -v app-data:/var/lib/data postgres:latest
```

---

## 6. 자주 발생하는 문제와 해결 방법

### 권한 오류
**문제**: `permission denied while connecting to Docker daemon`
**해결**: 사용자를 docker 그룹에 추가
```bash
sudo usermod -aG docker $USER
# 이후 로그아웃 후 다시 로그인 필요
```

### 포트 충돌
**문제**: `port is already allocated`
**해결**: 다른 포트 사용 또는 기존 프로세스 확인
```bash
# Linux/macOS
lsof -i :8080

# Windows
netstat -ano | findstr :8080
```

### 이미지 다운로드 실패
**문제**: 프록시 환경에서 `pull` 실패
**해결**: Docker 데몬 프록시 설정
```bash
# 시스템에 맞게 프록시 설정
sudo mkdir -p /etc/systemd/system/docker.service.d
echo '[Service]
Environment="HTTP_PROXY=http://proxy.example.com:3128"
Environment="HTTPS_PROXY=http://proxy.example.com:3128"' | sudo tee /etc/systemd/system/docker.service.d/proxy.conf
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## 7. 실전 모범 사례

### 1. 일회성 작업에 `--rm` 사용
```bash
docker run --rm alpine sh -c "date && echo 작업 완료"
```

### 2. 보안 강화를 위한 최소 권한 실행
```bash
docker run --rm \
  --read-only \
  --tmpfs /tmp \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --user 1001:1001 \
  alpine id
```

### 3. Docker Compose로 여러 서비스 관리
```yaml
# docker-compose.yml
version: '3.8'
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: example
```
```bash
# 서비스 시작
docker compose up -d

# 서비스 중지
docker compose down
```

## 마무리

Docker 명령어는 처음에는 많아 보이지만, 실제로 자주 사용하는 명령어는 한정되어 있습니다. **pull → run → ps/logs → stop → rm**의 기본 흐름을 먼저 익히고, 점차적으로 네트워크, 볼륨, 리소스 제한 등의 옵션을 추가해 나가는 것이 좋습니다.

가장 중요한 것은 직접 실습해 보는 것입니다. 간단한 Nginx나 Alpine 이미지로 시작하여 컨테이너를 실행하고, 로그를 확인하고, 내부를 탐색해 보세요. 문제가 발생하면 `docker logs`와 `docker inspect` 명령어로 원인을 파악하는 습관을 기르면 Docker를 효과적으로 활용할 수 있게 될 것입니다.

실제 운영 환경에서는 항상 특정 버전의 이미지를 사용하고, 최소 권한 원칙을 적용하며, 리소스 제한을 설정하는 등 보안과 안정성을 고려한 설정이 필요합니다. 이러한 모범 사례들을 하나씩 적용해 나간다면 Docker를 더욱 효과적이고 안전하게 사용할 수 있을 것입니다.