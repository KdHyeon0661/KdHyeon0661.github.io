---
layout: post
title: Docker - Docker 이미지
date: 2025-01-11 20:20:23 +0900
category: Docker
---
# Docker 이미지란? 계층 구조 이해하기

## Docker 이미지(Image)란?

- 컨테이너 실행을 위한 **불변(immutable) 실행 환경 템플릿**입니다.
- 이미지 자체는 **읽기 전용**이며, 실행 시 컨테이너 위에 **쓰기 가능 레이어(컨테이너 레이어)** 가 추가됩니다.
- 이미지는 **여러 개의 레이어(layer)** 로 구성되고, 레이어는 **내용 해시(sha256)** 로 식별됩니다(내용 주소화, Content-Addressable Storage).

### 핵심 특징(요약 복습)

- 불변: 이미지 변경 대신 **새 레이어** 추가 또는 **새 이미지** 생성.
- 계층: Dockerfile의 **명령어 1개 = 레이어 1개**(특정 지시문 제외).
- 공유: 동일 레이어는 여러 이미지/컨테이너가 **공유**(디스크 절약).
- 캐시: 빌드 시 **입력 동일**하면 해당 레이어를 **캐시 재사용**.

---

## 레이어드 구조(Layered Structure)의 직관

다음 Dockerfile을 보겠습니다.

```Dockerfile
FROM ubuntu:20.04                  # [Layer 1]
RUN apt-get update                 # [Layer 2]
RUN apt-get install -y nginx       # [Layer 3]
COPY . /app                        # [Layer 4]
```

### 결과 구조(개념도)

```
Layer 4: COPY . /app
Layer 3: RUN apt-get install -y nginx
Layer 2: RUN apt-get update
Layer 1: FROM ubuntu:20.04
```

- 각 레이어는 **읽기 전용**입니다.
- 컨테이너 실행 시 최상단에 **쓰기 가능한 컨테이너 레이어**가 덧붙습니다.

텍스트 다이어그램(실행 시 파일시스템 스택)

```
+----------------------------+
| Writable Container Layer   |  ← 컨테이너 전용 쓰기 레이어(upperdir)
+----------------------------+
| Layer 4 (COPY . /app)      |
+----------------------------+
| Layer 3 (install nginx)    |
+----------------------------+
| Layer 2 (apt-get update)   |
+----------------------------+
| Layer 1 (ubuntu:20.04)     |  ← 베이스 이미지
+----------------------------+
```

---

## 간단 메커니즘

Linux에서 도커는 보통 **OverlayFS**를 사용합니다(환경에 따라 달라질 수 있음).

- 여러 **읽기 전용 레이어(lowerdir)** 와 **쓰기 가능한 레이어(upperdir)** 를 **merged** 하여 하나의 파일시스템처럼 노출.
- **COW(Copy-on-Write)**: 컨테이너가 파일을 수정하려 하면, 해당 파일이 upperdir로 **복사 후 수정**됩니다.
  읽기 전용 레이어는 그대로 유지되어 중복 저장을 피합니다.

이런 구조 덕분에:
- 동일한 베이스 레이어(예: `ubuntu:20.04`)는 **여러 컨테이너/이미지가 공유** → 디스크 절약.
- 컨테이너 간 격리와 빠른 생성/삭제가 가능.

---

## 빌드 캐시 재사용 규칙(정확하게 이해하기)

Dockerfile의 각 지시문은 입력이 동일하면 캐시를 재사용합니다. “입력”에는 다음이 포함됩니다.

- Dockerfile의 해당 **명령문** 자체(문자열)
- **빌드 컨텍스트에 포함된 파일들**(COPY/ADD 대상)
- 이전 레이어의 **난수적 변화**(환경변수, ARG, 시각, 네트워크 결과 등)

즉, **위쪽 레이어가 달라지면** 아래 레이어들도 **연쇄적으로 캐시 무효화**됩니다.

간단한 직관 수식(기대 빌드 시간 근사):
$$
\mathbb{E}[T_{\text{build}}] \approx \sum_{i=1}^{n}(1-p_i)\,c_i
$$
- \(p_i\): i번째 레이어의 캐시 적중률
- \(c_i\): i번째 레이어 빌드 비용
→ 변경이 적은/비용이 큰 단계(의존성 설치 등)를 **위로** 배치하여 \(p_i\) 를 키우면 전체 빌드 시간이 줄어듭니다.

---

## .dockerignore로 컨텍스트 다이어트

빌드 컨텍스트(도커가 데몬으로 전송하는 파일 집합)가 크면 **해시/전송 비용** 때문에 느려집니다. 불필요한 파일 제외:

```dockerignore
__pycache__/
*.pyc
.git/
node_modules/
dist/
build/
.env
*.log
```

- 큰 폴더나 캐시, 비밀키 파일(예: `.env`)은 반드시 제외.
- 컨텍스트가 동일해야 캐시가 잘 맞습니다. 자주 변하는 파일(로그 등)이 포함되면 매번 캐시 무효화가 일어납니다.

---

## 자주 겪는 캐시 함정과 안전한 패턴

### apt 사용 시

```Dockerfile
# 권장 X: update와 install을 분리하면 캐시된 stale index 사용 위험

RUN apt-get update
RUN apt-get install -y curl
```

```Dockerfile
# 권장: 한 RUN에서 update + install + 캐시 삭제

RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*
```

### Python 의존성

```Dockerfile
# 권장 패턴: requirements 먼저 COPY → pip 설치 → 소스 COPY

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```
이러면 **소스만 변경** 시 의존성 레이어 캐시가 유지됩니다.

---

## 멀티스테이지 빌드로 슬림하게

빌드 도구/헤더는 **build stage** 에서만 사용하고, 런타임에는 **산출물만 복사**합니다.

### Python Flask + Gunicorn 예

```Dockerfile
# syntax=docker/dockerfile:1.7

########## build ##########

FROM python:3.12-alpine AS build
WORKDIR /app
RUN apk add --no-cache build-base libffi-dev
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip wheel --no-cache-dir --no-deps -r requirements.txt -w /wheels
COPY . .

########## runtime ##########

FROM python:3.12-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
USER app
COPY --from=build /wheels /wheels
RUN pip install --no-cache-dir /wheels/*

COPY --from=build /app /app
EXPOSE 5000
ENTRYPOINT ["gunicorn"]
CMD ["-w","1","-b","0.0.0.0:5000","app:app"]
```

장점:
- 런타임 이미지가 **작음**(보안/전송 비용/기동 속도에 유리)
- 의존성 설치 캐시를 **효율적으로 재사용**

---

## 이미지 크기 줄이기 체크리스트

- `alpine`/`slim` 베이스 고려(호환성 확인).
- 패키지 설치 후 **캐시 삭제**(apk/apt 캐시 디렉터리).
- **프로덕션 의존성만** 설치(npm ci --omit=dev 등).
- 멀티스테이지로 **산출물만** 복사.
- 불필요한 도구/스크립트 제거.
- 레이어 수 vs 캐시성/가독성의 **균형**.

---

## 이미지·레이어 관찰 명령어

### 목록/상세/히스토리

```bash
docker images
docker image inspect <이미지:태그> | jq '.[0].RootFS.Layers'
docker history <이미지:태그>
```

예:
```bash
docker history ubuntu:20.04
```

일반 출력 예시:
```
IMAGE          CREATED       CREATED BY                                      SIZE
f643c72bc252   3 weeks ago   /bin/sh -c #(nop)  CMD ["/bin/bash"]           0B
...
```

### 레이어/용량 요약

```bash
docker system df
```

---

## 재현성 강화: 태그 vs 다이제스트

- **tag**: 사람이 읽기 쉬운 버전 라벨(가변)
- **digest(sha256)**: **내용 해시(불변)** — 재현 가능한 배포에 권장

{% raw %}
```bash
docker pull nginx:alpine
docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine
# 예: nginx@sha256:abc...

docker run --rm -p 8080:80 nginx@sha256:abc...
```
{% endraw %}

---

## 실습 1 — 레이어 캐시 체감

### 프로젝트(의존성 먼저)

```
cache-demo/
├── Dockerfile
├── requirements.txt
└── app.py
```

```Dockerfile
FROM python:3.12-slim
WORKDIR /app

# 의존성 먼저 복사하여 캐시 포인트로 삼기

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 그 다음 소스 복사

COPY . .

CMD ["python", "app.py"]
```

```bash
# 첫 빌드 — 의존성 설치 수행

docker build -t cache-demo:1 .

# app.py만 살짝 수정 → 재빌드(의존성 레이어는 캐시 히트)

docker build -t cache-demo:2 .
```

빌드 로그를 비교해보면 두 번째 빌드는 **의존성 레이어가 캐시**로 처리되어 빠릅니다.

---

## 실습 2 — COPY 순서에 따른 캐시 차이

```Dockerfile
# 비권장: 소스 전체를 먼저 COPY → 의존성 설치가 매번 무효화될 가능성

FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci
CMD ["npm","start"]
```

```Dockerfile
# 권장: package.json만 먼저 COPY → npm ci → 소스 COPY

FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
CMD ["npm","start"]
```

두 Dockerfile을 각각 빌드 후 작은 소스 변경을 반복해보면, **두 번째 방식**이 **빌드 속도**가 훨씬 안정적으로 빠릅니다.

---

## 컨테이너 레이어(쓰기 레이어)와 데이터 보존

컨테이너 실행 시 생성되는 **쓰기 레이어**는 컨테이너 수명과 함께 합니다.
데이터 보존이 필요한 경우 **볼륨(volume)** 을 사용해야 합니다.

```bash
# 예: PostgreSQL 데이터 디렉터리를 볼륨에 유지

docker volume create pgdata
docker run -d --name pg \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine
```

- 컨테이너를 삭제해도 볼륨을 삭제하지 않는 한 데이터는 유지됩니다.

---

## 보안 실행 스니펫(이미지와 별개지만 습관화 권장)

```bash
docker run --rm \
  --read-only \
  --tmpfs /tmp --tmpfs /run \
  --cap-drop ALL --security-opt no-new-privileges \
  --user 65532:65532 \
  --pids-limit 128 --cpus 0.5 --memory 256m \
  <이미지:태그>
```

- 읽기 전용 루트FS, 임시 쓰기 경로 최소, 권한 최소화로 **기본 방어선** 형성.
- 이미지 자체는 불변이지만, **실행 시 옵션**으로 안전성을 크게 높일 수 있습니다.

---

## Nginx 커스터마이징(간단 예제)

```Dockerfile
FROM nginx:alpine
COPY ./html /usr/share/nginx/html
```

```bash
docker build -t custom-nginx .
docker run -d -p 8080:80 --name web custom-nginx
curl http://localhost:8080
```

추가로 설정파일 포함(확장):

```Dockerfile
FROM nginx:alpine
COPY ./html /usr/share/nginx/html
COPY ./nginx.conf /etc/nginx/nginx.conf
```

---

## 트러블슈팅 표

| 현상 | 가능 원인 | 해결 |
|---|---|---|
| 빌드가 매우 느림 | 컨텍스트 과대, 캐시 미활용 | `.dockerignore` 추가, 레이어 순서 개선, BuildKit 활성화 |
| `no space left on device` | 레이어/이미지/컨테이너 누적 | `docker system df`, `docker system prune -af`(주의) |
| apt 패키지 404 | 오래된 인덱스/미러 | `apt-get update && apt-get install` 을 한 RUN에서 수행 |
| 포트 접근 불가 | `-p` 누락/방화벽/SELinux 라벨 | `docker ps`, `docker port`, 방화벽/SELinux 확인 |
| 컨테이너 즉시 종료 | CMD/ENTRYPOINT가 종료 | `docker logs`, 장기 실행 프로세스/엔트리 조정 |
| 파일 권한 오류 | 비루트 + 읽기전용 FS | `--tmpfs` 제공, `--user` UID/GID/마운트 권한 확인 |

---

## 관찰·점검에 유용한 명령어 묶음

```bash
# 이미지/레이어

docker images
docker image inspect <이미지:태그> | jq '.[0].RootFS.Layers'
docker history <이미지:태그>

# 디스크/캐시

docker system df
docker builder prune   # 빌더 캐시만 정리(주의)

# 컨테이너 상태

docker ps -a
docker logs -f <컨테이너>
docker inspect <컨테이너> | jq '.[0].State, .[0].Mounts'
```

---

## 요약

- Docker 이미지는 **읽기 전용 레이어의 스택**으로 구성되며, 컨테이너 실행 시 **쓰기 레이어**가 추가됩니다.
- **OverlayFS + COW** 구조 덕분에 **저장공간 절약**과 **고속 생성/삭제**가 가능합니다.
- **캐시 재사용**을 극대화하려면 `.dockerignore` 와 **레이어 순서**(의존성 먼저, 소스 나중)를 설계하세요.
- **멀티스테이지 빌드**로 런타임 이미지를 **슬림/안전**하게 유지하세요.
- 운영 재현성을 위해 **digest 고정**(sha256) 사용을 고려하세요.
- 문제는 **관찰(로그/inspect/system df) → 가설 → 수정** 루프로 빠르게 해결합니다.

---

## 실전형 Python 슬림 Dockerfile(요약)

```Dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
USER 65532:65532
EXPOSE 5000
CMD ["python","app.py"]
```

## Node.js 캐시 최적화 스니펫

```Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
CMD ["npm","start"]
```

## 수식 메모(캐시 적중 직관)

레이어별 캐시 적중률을 \(p_i\), 빌드비용을 \(c_i\) 라 하면,
$$
\mathbb{E}[T_{\text{build}}] \approx \sum_{i=1}^{n}(1-p_i)\,c_i
$$
변경이 잦은 소스는 **아래**, 변경이 드문 의존성은 **위**로 배치하여 \(p_i\) 를 높이면 전체 빌드 시간이 줄어듭니다.
