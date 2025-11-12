---
layout: post
title: Docker - Docker 이미지 커스터마이징
date: 2025-01-11 21:20:23 +0900
category: Docker
---
# Docker 이미지 생성 및 커스터마이징

## 0. 빠른 개요(핵심 복습)

- **이미지(Image)**: 컨테이너 실행을 위한 **읽기 전용 템플릿**. 여러 **레이어**의 스택이며 내용 해시로 관리됩니다.
- **Dockerfile**: 이미지를 어떻게 만들지 지시하는 **명령어 모음 파일**.
- **빌드 → 실행**: `docker build -t app:1 .` → `docker run app:1`.
- **레이어/캐시**: `RUN/COPY/ADD` 등 각 지시문이 **레이어**를 만들며, 동일 입력이면 **캐시 재사용**.

---

# 1. Dockerfile 기본

### 1.1 디렉터리 레이아웃(예)
```
my-app/
├── Dockerfile
├── app.py
└── requirements.txt
```

### 1.2 가장 단순한 Flask 예제(기본 버전)
```Dockerfile
# 1. 베이스 이미지
FROM python:3.10-slim

# 2. 작업 디렉토리
WORKDIR /app

# 3. 의존성 설치 (캐시 최대화: 먼저 requirements만 복사)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 4. 앱 코드 복사
COPY . .

# 5. 실행 명령
CMD ["python", "app.py"]
```

### 1.3 이미지 빌드 및 실행
```bash
docker build -t my-flask-app .
docker run -d -p 5000:5000 --name my-flask-app my-flask-app
```

> 앱이 :5000에서 리스닝한다면 `http://localhost:5000`

---

# 2. 레이어와 캐시를 이해하고 이기는 설계

Dockerfile의 각 명령은 **상위에 레이어**를 하나 추가합니다. **입력(파일/환경변수/명령문)이 동일**하면 해당 레이어는 **캐시 히트**로 다시 만들지 않습니다.

- **변경 빈도가 낮은 단계**(런타임 설치, 시스템 패키지, `requirements.txt` 등)를 **위쪽에** 배치
- **변경이 잦은 소스 코드**는 **아래쪽에** 배치
- **.dockerignore** 로 빌드 컨텍스트를 최소화 (큰 파일/불필요 디렉터리 제외)

간단한 직관 수식:
$$
\mathbb{E}[T_{\text{build}}] \approx \sum_{i=1}^{n} (1-p_i)\,c_i
$$
여기서 \(p_i\)는 i번째 레이어 캐시 적중률, \(c_i\)는 그 레이어의 빌드 비용입니다. **\(p_i\)↑** (변경 적은 레이어를 위로) → **총 빌드 시간↓**.

---

# 3. `.dockerignore` — 컨텍스트 다이어트

### 3.1 예시
```dockerignore
__pycache__/
*.pyc
.git/
node_modules/
.env
dist/
build/
*.log
```
- `.git/`, `node_modules/`(언어별 캐시 폴더), 대용량 데이터/로그는 **반드시 제외**.
- 컨텍스트가 클수록 업로드/해시 비용 증가 → 빌드가 느려짐.

---

# 4. 멀티스테이지 빌드 — “빌드는 무겁게, 런타임은 가볍게”

런타임 이미지에서 **컴파일러/툴체인**을 제거하고, **산출물만** 가져옵니다.

## 4.1 Python(Flask) + Gunicorn 멀티스테이지
```Dockerfile
# syntax=docker/dockerfile:1.7

########## build stage ##########
FROM python:3.12-alpine AS build
WORKDIR /app

# 빌드시 필요한 툴/헤더 (wheel 빌드용)
RUN apk add --no-cache build-base libffi-dev

COPY requirements.txt .
# BuildKit 캐시를 활용하면 의존성 설치가 빨라짐(선택)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip wheel --no-cache-dir --no-deps -r requirements.txt -w /wheels

COPY . .

########## runtime stage ##########
FROM python:3.12-alpine
WORKDIR /app

# 보안: 비루트 사용자 생성
RUN addgroup -S app && adduser -S app -G app
USER app

# 휠만 설치 → 빠르고 캐시친화적
COPY --from=build /wheels /wheels
RUN pip install --no-cache-dir /wheels/*

# 앱 복사
COPY --from=build /app /app

# 문서용 포트 선언(실제 노출은 -p 필요)
EXPOSE 5000

# Gunicorn으로 프로덕션 실행 (싱글 워커 예시)
ENTRYPOINT ["gunicorn"]
CMD ["-w","1","-b","0.0.0.0:5000","app:app"]
```

- **장점**: 런타임 이미지가 **작고 안전**(컴파일러/헤더 無), 레이어 캐시 효율 ↑  
- **비루트 사용자**로 실행: 컨테이너 탈출 시 리스크 억제

## 4.2 Go(정적 바이너리) 초슬림 예
```Dockerfile
# build
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -o /out/app ./cmd/app

# runtime (distroless scratch)
FROM scratch
COPY --from=build /out/app /app
ENTRYPOINT ["/app"]
```
- `FROM scratch` 로 **수 MB 이하** 런타임 가능
- 단, CA 인증서가 필요한 HTTP 클라이언트라면 **추가 처리** 필요(예: `ca-certificates` 번들)

## 4.3 Node.js (Build → Nginx serve) 예
```Dockerfile
# build stage
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
COPY . .
RUN npm run build  # dist/

# runtime stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

---

# 5. ENTRYPOINT vs CMD — 정확히 알고 쓰기

- **ENTRYPOINT**: “항상 실행되는 고정 바이너리/스크립트” 지정
- **CMD**: 기본 인자(또는 셸 커맨드) 지정. `docker run ... <override>` 로 덮을 수 있음
- 실무 팁:
  - **ENTRYPOINT는 exec form**(JSON 배열)으로, **PID 1**에 올릴 것
  - **신호 전달/종료 처리**가 필요한 애플리케이션은 PID 1에서 **SIGTERM 처리** 확인

### 5.1 안전한 exec form 예
```Dockerfile
ENTRYPOINT ["gunicorn"]
CMD ["-w","1","-b","0.0.0.0:5000","app:app"]
```

### 5.2 쉘 form는 신호 전달 문제 가능
```Dockerfile
# 권장하지 않음(쉘 경유)
ENTRYPOINT gunicorn -w 1 -b 0.0.0.0:5000 app:app
```

---

# 6. 환경 변수/ARG/Label/헬스체크

## 6.1 ENV & ARG
```Dockerfile
ARG BUILD_DATE
ARG VCS_REF
ENV APP_ENV=prod

# Label(메타데이터)
LABEL org.opencontainers.image.created=$BUILD_DATE \
      org.opencontainers.image.revision=$VCS_REF \
      org.opencontainers.image.source="https://example.com/repo"
```
- `ARG` 는 **빌드 시점** 변수(이미지 내부에 남지 않음)
- `ENV` 는 **런타임 환경 변수**

## 6.2 HEALTHCHECK
```Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -fsS http://localhost:5000/health || exit 1
```
- 오케스트레이션(K8s/LB) 전에 **기본 생존성 체크** 가능

---

# 7. 보안·권한·파일시스템 — 기본 방어선

- **비루트(USER)**: root 대신 **일반 사용자** 실행
- **읽기 전용 루트FS**: `--read-only` + 필요한 경로는 `--tmpfs /tmp` 등
- **Capabilities 최소화**: `--cap-drop ALL` + 필요한 것만 `--cap-add`
- **no-new-privileges**: SUID 승격 차단
- **비밀(Secrets)**: 환경변수보다 **런타임 시크릿**(Swarm/K8s/BuildKit Secret mount)

### 7.1 실행 예(기본 방어 조합)
```bash
docker run --rm \
  --read-only \
  --tmpfs /tmp --tmpfs /run \
  --cap-drop ALL --security-opt no-new-privileges \
  --user 65532:65532 \
  --pids-limit 128 --cpus 0.5 --memory 256m \
  my-flask-app
```

---

# 8. 네트워크/포트·볼륨/영속화 — 빠른 실습

## 8.1 정적 파일 서빙(Nginx 커스터마이징)
```Dockerfile
FROM nginx:alpine
COPY ./html /usr/share/nginx/html
```
```bash
docker build -t custom-nginx .
docker run -d -p 8080:80 --name web custom-nginx
curl http://localhost:8080
```

## 8.2 데이터 영속화(예: Postgres)
```bash
docker volume create pgdata
docker run -d --name pg \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine
```

---

# 9. 재현성·배포: 태그 vs 다이제스트, 저장/로드

- **tag**: 사람이 읽기 쉬움(가변)
- **digest(sha256)**: 내용 기준(불변). 재현 가능 배포에 **권장**

{% raw %}
```bash
docker pull nginx:alpine
docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine
# 예: nginx@sha256:abcd...
docker run --rm -p 8080:80 nginx@sha256:abcd...
```
{% endraw %}

오프라인/에어갭 배포:
```bash
docker save -o custom-nginx.tar custom-nginx
# 원격지에서
docker load -i custom-nginx.tar
```

---

# 10. BuildKit — 캐시/시크릿/SSH/병렬 빌드

환경 변수로 켜기:
```bash
export DOCKER_BUILDKIT=1
```

## 10.1 시크릿 주입(빌드 시간)
```Dockerfile
# syntax=docker/dockerfile:1.7
FROM alpine
RUN --mount=type=secret,id=npmrc cat /run/secrets/npmrc >/dev/null || true
```
```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t app:secret .
```

## 10.2 언어별 캐시 마운트
```Dockerfile
# Go 예: /go/pkg/mod
RUN --mount=type=cache,target=/go/pkg/mod go mod download
```

---

# 11. `.env` / Compose로 개발자 경험(DX) 개선

## 11.1 Compose 예시(Flask + Nginx)
```yaml
# docker-compose.yaml
services:
  api:
    build: ./api
    environment:
      - APP_ENV=dev
    ports:
      - "5000:5000"

  web:
    image: nginx:alpine
    volumes:
      - ./web/html:/usr/share/nginx/html:ro
    ports:
      - "8080:80"
    depends_on:
      - api
```
```bash
docker compose up -d --build
docker compose logs -f
docker compose down -v
```

---

# 12. 트러블슈팅 — 원인별 빠른 표

| 증상/오류 | 가능 원인 | 해결 |
|---|---|---|
| 빌드가 너무 느림 | 컨텍스트 과대, 캐시 미활용 | `.dockerignore` 최적화, 레이어 순서 조정, BuildKit 캐시 |
| `no space left on device` | 디스크/레이어 누적 | `docker system df`, `docker system prune -af`(주의) |
| 컨테이너 즉시 종료 | CMD/ENTRYPOINT가 종료 | `docker logs`, 커맨드 수정, 장기 실행 프로세스 사용 |
| 포트 바인딩 실패 | 포트 충돌 | `lsof -i :8080`(Linux/macOS), `netstat -ano | findstr :8080`(Windows) |
| 권한 오류 | 비루트 + 읽기전용 | `--tmpfs`로 쓰기 경로 제공, `--user` UID/GID 매핑 확인 |
| 프록시/사설CA로 pull 실패 | 네트워크/CA 설정 | 데몬 프록시/CA 등록, 컨테이너 내부 `-e HTTP(S)_PROXY` |
| WSL2에서 파일 I/O 느림 | 호스트↔WSL 경계 병목 | **WSL2 내부 경로**에서 빌드/마운트 |

---

# 13. 언어별 Dockerfile “좋은 습관” 스니펫

## 13.1 Python
```Dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
USER 65532:65532
CMD ["python","app.py"]
```

## 13.2 Node.js
```Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
EXPOSE 3000
USER node
CMD ["npm","start"]
```

## 13.3 Java (JAR 런)
```Dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/app.jar /app/app.jar
ENTRYPOINT ["java","-XX:+UseZGC","-jar","/app/app.jar"]
```

---

# 14. 이미지 크기 줄이기 — 체크리스트

- `alpine`/`slim` 베이스 고려(호환성 이슈 확인)
- 패키지 설치 후 **캐시 삭제**(apk/apt의 캐시 디렉터리)
- 언어별 **프로덕션 의존성만** 설치(npm ci --omit=dev 등)
- 멀티스테이지로 **산출물만** 복사
- 불필요한 도구/셸 스크립트 제거
- **중간 레이어 수** 최소화(단, 가독성과 캐시성의 균형)

---

# 15. 실습: Flask 프로덕션 템플릿(보안/헬스체크 포함)

```Dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-alpine AS base
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app

FROM base AS deps
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt -t /python

FROM base AS runtime
ENV PYTHONPATH=/python
COPY --from=deps /python /python
COPY . .
USER app
EXPOSE 5000

# 헬스엔드포인트가 /health 라고 가정
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python - <<'PY' || exit 1
import urllib.request
try:
    urllib.request.urlopen('http://127.0.0.1:5000/health', timeout=2)
except Exception:
    raise SystemExit(1)
PY

ENTRYPOINT ["python","app.py"]
```

---

# 16. 실습: Nginx 커스터마이징(확장)

### 16.1 사용자 정의 conf 포함
```Dockerfile
FROM nginx:alpine
COPY ./html /usr/share/nginx/html
COPY ./nginx.conf /etc/nginx/nginx.conf
```

```nginx
# ./nginx.conf (간단 예)
events {}
http {
  server {
    listen 80;
    root /usr/share/nginx/html;
    location /health { return 200 "ok\n"; }
  }
}
```

```bash
docker build -t custom-nginx:1 .
docker run -d -p 8080:80 --name web custom-nginx:1
curl http://localhost:8080/health
```

---

# 17. 이미지 업데이트 & 롤링 재배포(수동 프로세스)

```bash
# 1. 코드/ Dockerfile 변경 → 재빌드
docker build -t my-flask-app:2 .

# 2. 구버전 정지/삭제
docker stop my-flask-app || true
docker rm my-flask-app || true

# 3. 신버전 실행
docker run -d --name my-flask-app -p 5000:5000 my-flask-app:2
```

**힌트**: 실제 운영에서는 **태그 전략**(`:1.0.1`/`:prod`) & **다이제스트 고정** & **오케스트레이터**(Compose/K8s)를 함께 사용.

---

# 18. 고급 팁 — 레지스트리/메타데이터/서명/스캔

- 레지스트리 로그인/푸시:
  ```bash
  docker tag my-flask-app:2 ghcr.io/owner/my-flask-app:2
  docker push ghcr.io/owner/my-flask-app:2
  ```
- 메타데이터(Label)로 **생성일/커밋 해시/빌드번호** 남기기
- **서명/검증**(cosign/Notary), **취약점 스캔**(trivy) 습관화

---

# 19. 문제해결 레시피(원인→진단→처방)

| 상황 | 진단 명령 | 처방 |
|---|---|---|
| 빌드 실패(의존성) | `docker build --no-cache`, 로그 | 빌드 툴/헤더 추가, 버전 고정, 캐시 마운트 |
| 컨테이너 즉시 종료 | `docker logs`, `docker inspect` | CMD/ENTRYPOINT 교정, 장기 실행 커맨드 |
| 포트 접근 불가 | `docker ps`, `docker port`, `curl` | `-p` 확인, 방화벽/SELinux 라벨, 포트 충돌 점검 |
| 파일 권한 문제 | `docker exec -it ... sh`, `ls -al` | `--user`/UID 매핑, 마운트 옵션, 읽기전용 FS 조정 |
| 메모리 OOM 재시작 | `docker stats`, 로그 | `--memory` 상향/GC 튜닝, 메모리 누수 확인 |
| 느린 빌드 | `docker system df`, 컨텍스트 크기 | `.dockerignore` 업데이트, 멀티스테이지, BuildKit 캐시 |

---

# 20. 요약

1. **레이어/캐시** 원리를 이해하고 **.dockerignore + 레이어 순서**로 빌드를 빠르게.  
2. **멀티스테이지**로 런타임을 **작고 안전하게**.  
3. **ENTRYPOINT/CMD, 비루트, 읽기전용 FS, 최소 권한**으로 보안 기본기를 확보.  
4. **다이제스트 고정**으로 재현성, **save/load**로 이동성, **Compose**로 개발자 경험 향상.  
5. 문제는 **관찰(로그/inspect/stats) → 가설 → 수정** 루프로 빠르게 해결.

---

## 부록 A) 빠른 치트시트

{% raw %}
```bash
# 빌드/실행
docker build -t app:1 .
docker run -d --name app -p 8080:8080 app:1

# 내부접속/로그/상태
docker exec -it app sh
docker logs -f app
docker inspect app | jq '.[0].State'

# 정리
docker stop app && docker rm app
docker rmi app:1
docker system prune -f   # 주의

# 재현성
docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine
docker run --rm nginx@sha256:...

# 저장/로드
docker save -o app.tar app:1
docker load -i app.tar
```
{% endraw %}

## 부록 B) Python requirements 잠금/최적화 팁

```Dockerfile
# poetry/uv/pip-tools 등으로 잠금 파일 생성 후 사용 권장
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt
```

## 부록 C) Windows/WSL2 팁(요약)

- 프로젝트를 **WSL2 내부 경로**에 두고 마운트하면 I/O가 훨씬 빠름.
- 포트 충돌 진단:
  ```powershell
  netstat -ano | findstr :8080
  ```
- 파일 권한/줄바꿈 차이(CRLF/LF)로 셸 스크립트 실행 실패 시 `dos2unix` 고려.