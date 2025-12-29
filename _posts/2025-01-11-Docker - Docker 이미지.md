---
layout: post
title: Docker - Docker 이미지
date: 2025-01-11 20:20:23 +0900
category: Docker
---
# Docker 이미지란? 계층 구조 이해하기

## Docker 이미지의 개념

Docker 이미지는 컨테이너 실행을 위한 불변의 실행 환경 템플릿입니다. 이미지 자체는 읽기 전용으로, 컨테이너 실행 시에만 쓰기 가능한 레이어가 추가됩니다. 이러한 구조는 여러 개의 레이어로 구성되어 있으며, 각 레이어는 내용 해시(sha256)로 식별되는 내용 주소화 방식(Content-Addressable Storage)을 사용합니다.

### Docker 이미지의 핵심 특징

- **불변성**: 이미지를 변경하지 않고, 새 레이어를 추가하거나 새로운 이미지를 생성합니다.
- **계층 구조**: Dockerfile의 각 명령어가 하나의 레이어를 생성합니다.
- **공유 가능성**: 동일한 레이어는 여러 이미지와 컨테이너가 공유하여 디스크 공간을 절약합니다.
- **캐시 활용**: 빌드 시 입력이 동일하면 해당 레이어를 캐시에서 재사용합니다.

---

## 레이어 구조의 이해

다음 Dockerfile을 예시로 레이어 구조를 살펴보겠습니다.

```Dockerfile
FROM ubuntu:20.04                  # [Layer 1]
RUN apt-get update                 # [Layer 2]
RUN apt-get install -y nginx       # [Layer 3]
COPY . /app                        # [Layer 4]
```

### 레이어 구조의 시각화

```
Layer 4: COPY . /app
Layer 3: RUN apt-get install -y nginx
Layer 2: RUN apt-get update
Layer 1: FROM ubuntu:20.04
```

각 레이어는 읽기 전용이며, 컨테이너 실행 시 최상단에 쓰기 가능한 컨테이너 레이어가 추가됩니다.

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

## Docker의 작동 메커니즘

Linux 시스템에서 Docker는 일반적으로 OverlayFS를 사용합니다. 이 시스템은 여러 읽기 전용 레이어(lowerdir)와 하나의 쓰기 가능한 레이어(upperdir)를 병합하여 단일 파일 시스템처럼 제공합니다.

**COW(Copy-on-Write)** 원리에 따라, 컨테이너가 파일을 수정하려 할 때 해당 파일이 upperdir로 복사된 후 수정됩니다. 읽기 전용 레이어는 그대로 유지되어 중복 저장을 방지합니다.

이러한 구조의 장점:
- 동일한 베이스 레이어를 여러 컨테이너와 이미지가 공유하여 디스크 공간 절약
- 컨테이너 간 격리와 빠른 생성/삭제 가능

---

## 빌드 캐시 재사용 원리

Docker는 Dockerfile의 각 지시문에 대해 입력이 동일할 때 캐시를 재사용합니다. 입력에는 다음 요소들이 포함됩니다:

- Dockerfile의 해당 명령문 자체
- 빌드 컨텍스트에 포함된 파일들(COPY/ADD 대상)
- 이전 레이어의 난수적 변화(환경변수, ARG, 시간, 네트워크 결과 등)

위쪽 레이어가 변경되면 아래 레이어들도 연쇄적으로 캐시가 무효화됩니다.

빌드 시간 최적화를 위한 수학적 접근:
$$
\mathbb{E}[T_{\text{build}}] \approx \sum_{i=1}^{n}(1-p_i)\,c_i
$$

- $p_i$: i번째 레이어의 캐시 적중률
- $c_i$: i번째 레이어 빌드 비용

변경이 적고 비용이 큰 단계(의존성 설치 등)를 앞쪽에 배치하여 $p_i$를 높이면 전체 빌드 시간을 줄일 수 있습니다.

---

## 빌드 컨텍스트 최적화

빌드 컨텍스트가 크면 해시 계산과 전송 비용으로 인해 빌드 속도가 느려집니다. `.dockerignore` 파일을 사용하여 불필요한 파일을 제외하세요.

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

큰 폴더, 캐시 파일, 비밀 키 파일은 반드시 제외해야 합니다. 컨텍스트가 일관되어야 캐시가 효과적으로 작동합니다.

---

## 효율적인 Dockerfile 작성 패턴

### APT 패키지 관리

```Dockerfile
# 권장 패턴: 한 RUN 명령어에서 업데이트, 설치, 정리 수행
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*
```

### Python 의존성 관리

```Dockerfile
# 의존성 파일을 먼저 복사하여 캐시 포인트 생성
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
# 그 다음 소스 코드 복사
COPY . .
```

이 패턴을 사용하면 소스 코드만 변경될 때 의존성 레이어 캐시가 유지됩니다.

---

## 멀티스테이지 빌드 활용

빌드 도구와 헤더 파일은 빌드 스테이지에서만 사용하고, 최종 런타임 이미지에는 산출물만 복사합니다.

### Python Flask 애플리케이션 예제

```Dockerfile
# syntax=docker/dockerfile:1.7

########## 빌드 스테이지 ##########
FROM python:3.12-alpine AS build
WORKDIR /app
RUN apk add --no-cache build-base libffi-dev
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip wheel --no-cache-dir --no-deps -r requirements.txt -w /wheels
COPY . .

########## 런타임 스테이지 ##########
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

멀티스테이지 빌드의 장점:
- 최종 런타임 이미지 크기 감소
- 보안 강화(불필요한 빌드 도구 제거)
- 의존성 설치 캐시 효율적 재사용

---

## 이미지 크기 최적화 전략

이미지 크기를 줄이기 위한 효과적인 전략들:

1. **경량 베이스 이미지 사용**: `alpine` 또는 `slim` 태그의 이미지 고려
2. **캐시 정리**: 패키지 설치 후 캐시 디렉터리 삭제
3. **프로덕션 의존성만 설치**: 개발용 패키지 제외
4. **멀티스테이지 빌드**: 최종 산출물만 포함
5. **불필요한 파일 제거**: 도구, 스크립트, 임시 파일 정리

---

## 이미지와 레이어 분석 도구

### 기본 분석 명령어

```bash
# 이미지 목록 확인
docker images

# 이미지 상세 정보 확인
docker image inspect <이미지:태그> | jq '.[0].RootFS.Layers'

# 이미지 빌드 히스토리 확인
docker history <이미지:태그>

# 디스크 사용량 확인
docker system df
```

### 실용적인 사용 예시

```bash
# Ubuntu 이미지의 빌드 히스토리 확인
docker history ubuntu:20.04
```

출력 예시:
```
IMAGE          CREATED       CREATED BY                                      SIZE
f643c72bc252   3 weeks ago   /bin/sh -c #(nop)  CMD ["/bin/bash"]           0B
...
```

---

## 재현성 보장: 태그와 다이제스트

- **태그(tag)**: 사람이 읽기 쉬운 버전 라벨(변경 가능)
- **다이제스트(digest)**: 내용 해시 기반의 불변 식별자(재현성 보장)

{% raw %}
```bash
# 이미지 다운로드
docker pull nginx:alpine

# 다이제스트 확인
docker inspect --format='{{index .RepoDigests 0}}' nginx:alpine

# 다이제스트로 정확한 버전 실행
docker run --rm -p 8080:80 nginx@sha256:abc...
```
{% endraw %}

---

## 실습: 레이어 캐시 체험

### 프로젝트 구조
```
cache-demo/
├── Dockerfile
├── requirements.txt
└── app.py
```

### Dockerfile 예시

```Dockerfile
FROM python:3.12-slim
WORKDIR /app

# 의존성 파일 먼저 복사 (캐시 포인트)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 소스 코드 복사
COPY . .

CMD ["python", "app.py"]
```

### 빌드 실습

```bash
# 첫 번째 빌드 (의존성 설치 수행)
docker build -t cache-demo:1 .

# app.py만 수정 후 재빌드 (의존성 레이어 캐시 히트)
docker build -t cache-demo:2 .
```

두 번째 빌드에서는 의존성 레이어가 캐시되어 더 빠르게 완료됩니다.

---

## 실습: COPY 순서에 따른 캐시 차이

### 비효율적인 패턴

```Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .  # 모든 파일 먼저 복사
RUN npm ci  # 소스 변경 시 항상 재실행
CMD ["npm","start"]
```

### 효율적인 패턴

```Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./  # 패키지 파일만 먼저 복사
RUN --mount=type=cache,target=/root/.npm npm ci  # 캐시 활용
COPY . .  # 나머지 소스 복사
CMD ["npm","start"]
```

두 Dockerfile을 비교해보면, 패키지 파일만 먼저 복사하는 방식이 소스 변경 시 빌드 속도를 크게 향상시킵니다.

---

## 데이터 지속성과 볼륨

컨테이너 실행 시 생성되는 쓰기 레이어는 컨테이너 수명과 함께합니다. 데이터를 영구적으로 보존하려면 볼륨을 사용해야 합니다.

```bash
# PostgreSQL 데이터 볼륨 생성 및 사용
docker volume create pgdata
docker run -d --name pg \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine
```

볼륨을 사용하면 컨테이너를 삭제해도 데이터가 유지됩니다.

---

## 보안 강화 실행 예시

```bash
docker run --rm \
  --read-only \  # 읽기 전용 루트 파일시스템
  --tmpfs /tmp --tmpfs /run \  # 임시 쓰기 경로
  --cap-drop ALL \  # 불필요한 권한 제거
  --security-opt no-new-privileges \  # 권한 상승 방지
  --user 65532:65532 \  # 비루트 사용자
  --pids-limit 128 --cpus 0.5 --memory 256m \  # 리소스 제한
  <이미지:태그>
```

이러한 옵션들을 통해 이미지 자체는 불변이더라도 실행 시 보안성을 크게 향상시킬 수 있습니다.

---

## Nginx 커스터마이징 예제

```Dockerfile
FROM nginx:alpine
COPY ./html /usr/share/nginx/html
COPY ./nginx.conf /etc/nginx/nginx.conf
```

```bash
# 이미지 빌드 및 실행
docker build -t custom-nginx .
docker run -d -p 8080:80 --name web custom-nginx
curl http://localhost:8080
```

---

## 문제 해결 가이드

| 문제 현상 | 가능한 원인 | 해결 방안 |
|---|---|---|
| 빌드 속도가 매우 느림 | 빌드 컨텍스트가 큼, 캐시 미활용 | `.dockerignore` 추가, 레이어 순서 개선, BuildKit 활성화 |
| 디스크 공간 부족 | 레이어/이미지/컨테이너 누적 | `docker system df` 확인, `docker system prune` 실행 |
| apt 패키지 설치 실패 | 오래된 패키지 인덱스 | `apt-get update && apt-get install`을 한 RUN에서 수행 |
| 포트 접근 불가 | 포트 매핑 누락 또는 방화벽 | `docker ps`로 포트 확인, 방화벽/SELinux 설정 점검 |
| 컨테이너 즉시 종료 | CMD/ENTRYPOINT 오류 | `docker logs`로 로그 확인, 프로세스 설정 조정 |

---

## 유용한 관찰 명령어

```bash
# 이미지와 레이어 분석
docker images
docker image inspect <이미지:태그> | jq '.[0].RootFS.Layers'
docker history <이미지:태그>

# 디스크 사용량 확인
docker system df

# 컨테이너 상태 확인
docker ps -a
docker logs -f <컨테이너>
docker inspect <컨테이너> | jq '.[0].State, .[0].Mounts'
```

---

## 실전 Dockerfile 예제

### Python 애플리케이션

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

### Node.js 애플리케이션

```Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
CMD ["npm","start"]
```

---

## 결론

Docker 이미지는 읽기 전용 레이어의 계층적 스택으로 구성되며, 이러한 구조는 효율적인 저장공간 활용과 빠른 컨테이너 생성을 가능하게 합니다. OverlayFS와 Copy-on-Write 메커니즘은 동일한 베이스 레이어를 여러 컨테이너가 공유할 수 있도록 하여 리소스 사용을 최적화합니다.

빌드 성능을 극대화하기 위해서는 `.dockerignore` 파일을 활용하고, 의존성 파일을 먼저 복사하는 레이어 순서를 설계하는 것이 중요합니다. 멀티스테이지 빌드는 최종 이미지 크기를 줄이고 보안성을 향상시키는 효과적인 방법입니다.

운영 환경에서 재현성을 보장하려면 태그 대신 다이제스트를 사용하는 것이 권장됩니다. 또한, 컨테이너 실행 시 보안 옵션을 적용하면 이미지의 불변성과 결합되어 강력한 보안 방어선을 구성할 수 있습니다.

Docker의 레이어 캐시 시스템을 이해하고 적절히 활용하면 개발 생산성을 크게 향상시킬 수 있습니다. 변경 빈도가 낮은 레이어를 앞쪽에 배치하고, 자주 변경되는 소스 코드는 뒤쪽에 위치시키는 것이 기본 원칙입니다. 이러한 이해를 바탕으로 효율적이고 안전한 Docker 이미지를 설계하고 관리할 수 있습니다.