---
layout: post
title: Docker - Docker 기본 명령어 정리
date: 2024-12-30 19:20:23 +0900
category: Docker
---
# Docker 기본 명령어 정리 — 흐름·옵션·실전 팁까지 확장

본 글은 기존 정리(이미지 다운로드 → 컨테이너 실행 → 상태 확인 → 정지 → 삭제)를 **핵심 유지**하면서,  
**옵션 의미/차이, 실전 예제, 자주 겪는 오류와 해결, 베스트 프랙티스, 관련 서브 명령**까지 한 번에 정리합니다.  
모든 코드는 ``` 코드펜스로, 수식은 필요한 경우만 $$...$$ 로 표기합니다.

---

## 0. 큰 흐름(요약)

```
이미지 다운로드(pull) ──> 컨테이너 실행(run) ──> 상태/로그 확인(ps/logs/exec)
                                      │
                                      └─> 정지(stop) ──> 삭제(rm/rmi)
```

---

# 1. 이미지 다운로드: `docker pull`

```bash
docker pull [레지스트리/저장소/]이미지[:태그]
```

### 예시
```bash
docker pull ubuntu:20.04
docker pull nginx          # 태그 생략 시 latest
docker pull ghcr.io/org/app:1.2.3   # GitHub Container Registry 예
```

### 확인
```bash
docker images
docker image inspect nginx:latest | jq '.[0].RepoDigests'
```
- **tag**(가변) vs **digest**(불변) 구분을 이해하면 운영에서 롤백/재현성을 높일 수 있습니다.

---

# 2. 컨테이너 실행: `docker run`

```bash
docker run [옵션] 이미지[:태그|@digest] [명령/인자...]
```

## 2.1 자주 쓰는 옵션

| 옵션 | 의미 | 비고/팁 |
|---|---|---|
| `-it` | 대화형 터미널(표준입력 + tty) | 셸 접속 등에 사용 |
| `--rm` | 프로세스 종료 시 컨테이너 자동 삭제 | 일회성 잡(job)에 유용 |
| `-d` | 백그라운드(detached) 실행 | 서버형 프로세스에 |
| `--name NAME` | 컨테이너 이름 지정 | 이후 `exec/logs/stop`에 편리 |
| `-p H:C` | 포트 매핑(host:container) | 예: `-p 8080:80` |
| `-v SRC:DST[:옵션]` | 볼륨/마운트 | `:ro`, SELinux `:Z` 등 |
| `--env K=V` | 환경변수 | `-e` 축약 |
| `--cpus N` | CPU 제한 | cgroups v2 |
| `--memory 512m` | 메모리 제한 | OOM 주의 |
| `--restart=always` | 재시작 정책 | 데몬성 컨테이너 |
| `--network NET` | 네트워크 지정 | 사용자 정의 브릿지 등 |
| `--user UID:GID` | 실행 사용자 지정 | 권한 최소화에 유용 |
| `--read-only` | 루트 FS 읽기전용 | 보안성↑, `--tmpfs` 병행 |

## 2.2 기본 실행 예제
```bash
docker run -it --name u1 ubuntu:20.04 bash   # 셸 접속
docker run --rm alpine echo "hello"           # 일회성
docker run -d --name web -p 8080:80 nginx     # 백그라운드 웹
```

## 2.3 digest 고정(재현성 강화)
```bash
docker pull nginx:alpine
docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine
# 예: nginx@sha256:abcd...
docker run --rm -p 8080:80 nginx@sha256:abcd...
```

---

# 3. 컨테이너 상태/리소스/로그 확인

## 3.1 목록: `docker ps`
```bash
docker ps          # 실행 중
docker ps -a       # 종료 포함 전체
```

## 3.2 로그: `docker logs`
```bash
docker logs web
docker logs -f --tail=100 web     # 실시간 + 최근 100줄
docker logs --since=10m web       # 최근 10분
```

## 3.3 내부 접속: `docker exec`
```bash
docker exec -it web sh
# 또는
docker exec -it web bash
```
- `docker attach` 는 **PID 1의 표준 입출력**에 붙습니다(대개 추천 X). 셸은 `exec`로.

## 3.4 리소스 모니터: `docker stats` / 프로세스: `docker top`
```bash
docker stats
docker top web
```

## 3.5 상세 정보: `docker inspect`
```bash
docker inspect web | jq '.[0].NetworkSettings.Ports'
docker inspect web | jq '.[0].State'
```

---

# 4. 정지/삭제/정리

## 4.1 정지/시작/재시작
```bash
docker stop web
docker start web
docker restart web
```

## 4.2 삭제
```bash
docker rm web                 # 정지된 컨테이너 삭제
docker rm -f web              # 강제(실행 중 → kill 후 삭제)
docker rmi nginx:latest       # 이미지 삭제
```

## 4.3 청소(주의)
```bash
docker system df
docker system prune -f        # 미사용 컨테이너/네트워크/빌드 캐시 정리
docker image prune -a -f      # 미사용 이미지 모두 삭제(주의)
```

---

# 5. 실전 흐름 예시(한 번에)

```bash
# 1) 이미지
docker pull nginx

# 2) 실행
docker run -d --name web_server -p 8080:80 nginx

# 3) 상태/로그/탐색
docker ps
docker logs -f web_server
docker exec -it web_server sh -lc 'nginx -v; ls -al /usr/share/nginx/html'

# 4) 정지/삭제
docker stop web_server
docker rm web_server
docker rmi nginx
```

---

# 6. 한 단계 더: 자주 쓰는 서브 명령/패턴

## 6.1 `docker cp` — 파일 복사
```bash
# 호스트 → 컨테이너
docker cp ./index.html web:/usr/share/nginx/html/

# 컨테이너 → 호스트
docker cp web:/etc/nginx/nginx.conf ./nginx.conf
```

## 6.2 `docker port` — 포트 확인
```bash
docker port web
# 예: 80/tcp -> 0.0.0.0:8080
```

## 6.3 `docker rename` — 이름 변경
```bash
docker rename web web-prod
```

## 6.4 `docker update` — 실행 중 자원 제한 조정(가능 범위)
```bash
docker update --cpus=1 --memory=512m web
```

## 6.5 `docker create` / `start` — 분리형 생성/시작
```bash
docker create --name web -p 8080:80 nginx
docker start web
```
- `run = create + start (+ attach)` 의 축약입니다.

## 6.6 이미지 태깅/저장/로드
```bash
docker tag nginx:alpine registry.local:5000/nginx:web
docker push registry.local:5000/nginx:web

docker save -o nginx_web.tar nginx:alpine
docker load -i nginx_web.tar
```

## 6.7 컨테이너 내보내기/가져오기(루트FS)
```bash
docker export web -o web_rootfs.tar
cat web_rootfs.tar | docker import - web-rootfs:latest
```

---

# 7. 네트워크/볼륨 — 기초 명령

## 7.1 네트워크
```bash
docker network ls
docker network create appnet
docker run -d --name api --network appnet -p 8081:8080 ghcr.io/acme/api:1.0
docker run --rm --network appnet curlimages/curl http://api:8080/health
docker network inspect appnet | jq '.[0].Containers'
docker network rm appnet   # 사용 중이면 삭제 불가
```

## 7.2 볼륨
```bash
docker volume ls
docker volume create data1
docker run -d --name db -v data1:/var/lib/postgresql/data postgres:16-alpine
docker volume inspect data1 | jq '.[0].Mountpoint'
docker rm -f db && docker volume rm data1
```

---

# 8. 자주 겪는 오류와 즉시 해결

| 증상/오류 | 원인/대응 |
|---|---|
| `permission denied while connecting to Docker daemon` | 리눅스에서 `sudo usermod -aG docker $USER` 후 재로그인 |
| `port is already allocated` | 포트 충돌. `lsof -i :8080`(Linux/macOS) 또는 `netstat -ano | findstr :8080`(Windows)로 점유 프로세스 확인, 포트 변경 |
| `pull` 지연/실패 | 프록시/사설 CA 문제. 데몬 프록시/CA 설정, 컨테이너 내부 프록시 `-e HTTP(S)_PROXY=...` |
| 파일 변경 반영 지연(Windows/WSL2) | 호스트-게스트 경계 I/O 병목. **WSL2 내부 경로**에서 빌드/마운트 |
| SELinux로 마운트 실패(RHEL/Alma/Rocky/Fedora) | `-v /data:/data:Z` 또는 정책 라벨 점검 |
| 컨테이너 즉시 종료 | 정상 동작일 수도(일회성 명령). 로그/ENTRYPOINT/CMD 확인 |

프록시 설정(리눅스 데몬 예):
```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
cat <<'EOF' | sudo tee /etc/systemd/system/docker.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy.local:3128"
Environment="HTTPS_PROXY=http://proxy.local:3128"
Environment="NO_PROXY=localhost,127.0.0.1,.svc,.internal"
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

# 9. 짧은 실습 컬렉션(손에 익히기)

## 9.1 일회성 잡 + 자동 삭제
```bash
docker run --rm alpine sh -c 'date && echo done'
```

## 9.2 읽기전용 루트FS + 임시 쓰기(tmpfs)
```bash
docker run --rm --read-only --tmpfs /tmp --tmpfs /run \
  alpine sh -lc 'touch /tmp/x && echo ok'
```

## 9.3 최소 권한 실행
```bash
docker run --rm --cap-drop ALL --security-opt no-new-privileges \
  --user 65532:65532 alpine id
```

## 9.4 Compose 맛보기(단일 서비스)
```yaml
# docker-compose.yaml
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
```
```bash
docker compose up -d
curl http://localhost:8080
docker compose down
```

---

# 10. 명령어 흐름 예시(확장판: 포트/볼륨/네트워크)

```bash
# 네트워크/볼륨 준비
docker network create appnet
docker volume create site

# Nginx에 정적 파일 제공
mkdir -p ./html && echo "hi" > ./html/index.html
docker run -d --name web \
  --network appnet \
  -p 8080:80 \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx:alpine

# 상태/로그
docker ps
docker logs -f web

# 내부 진입
docker exec -it web sh -lc 'ls -al /usr/share/nginx/html && nginx -v'

# 정리
docker stop web
docker rm web
docker network rm appnet
docker volume rm site     # (위 예에선 호스트 바인드사용, 볼륨 미사용)
```

---

# 11. 요약 명령어 표(보강)

| 목적 | 필수 명령 | 보강/심화 |
|---|---|---|
| 이미지 다운로드 | `docker pull 이미지[:태그]` | `docker image inspect`, `docker history` |
| 컨테이너 실행 | `docker run [옵션] 이미지` | `--rm`, `-d`, `-p`, `-v`, `--env`, `--cpus`, `--memory`, `--read-only` |
| 컨테이너 목록 | `docker ps [-a]` | `docker stats`, `docker top` |
| 로그 확인 | `docker logs [--since --tail -f] 컨테이너` | `docker events` |
| 내부 접속 | `docker exec -it 컨테이너 sh|bash` | `docker inspect`, `docker port` |
| 정지/시작 | `docker stop/start/restart 컨테이너` | `docker update` |
| 삭제 | `docker rm [-f] 컨테이너` | `docker rmi 이미지`, `docker system prune` |
| 파일 복사 | `docker cp SRC DST` |  |
| 네트워크/볼륨 | `docker network/volume create/ls/inspect/rm` | Compose/K8s로 확장 |

---

# 12. 간단 수식 메모(선택): 빌드/캐시 체감
레이어별 캐시 적중률을 \(p_i\), 빌드 비용을 \(c_i\) 라고 하면 기대 빌드 시간은
$$
\mathbb{E}[T] \approx \sum_{i=1}^{n} (1-p_i)\,c_i
$$
입니다. 변화가 적은 단계(의존성 설치)를 앞쪽에 두면 \(p_i\) 가 커져 전체 빌드 시간이 줄어듭니다.

---

## 마무리

- **핵심 흐름**(pull → run → 확인 → stop → rm)을 먼저 손에 익히고,  
- **옵션 의미**(네트워크/볼륨/리소스/보안)를 점차 추가하며,  
- **문제 발생 시** 로그(`logs`), 상세(`inspect`), 리소스(`stats`), 이벤트(`events`)로 추적하면 됩니다.  
- 익숙해지면 Compose로 여러 컨테이너를 묶어 관리하고, 운영에서는 digest 고정/권한 최소화/정리 전략을 습관화하세요.