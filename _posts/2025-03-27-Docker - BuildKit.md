---
layout: post
title: Docker - BuildKit
date: 2025-03-27 19:20:23 +0900
category: Docker
---
# ⚙️ BuildKit과 Docker 캐시 전략 완전 정복

---

## 🧱 BuildKit이란?

BuildKit은 Docker의 차세대 빌드 엔진으로, 기존 `legacy builder`에 비해 다음과 같은 장점을 갖습니다:

| 항목 | Legacy Builder | BuildKit |
|------|----------------|----------|
| 병렬 처리 | ❌ 순차적 실행 | ✅ 병렬 단계 실행 |
| 캐시 제어 | ❌ 제한적 | ✅ 단계별/레이어별 정교한 캐시 사용 |
| 보안 비밀 | ❌ 불가 | ✅ `--secret` 지원 |
| 빌드 출력 | 단일 이미지 | ✅ 여러 출력 지원 (image, tar 등) |
| 멀티 플랫폼 | ❌ 제한적 | ✅ `--platform` 완벽 지원 |
| 로깅 | 단순 텍스트 | ✅ 구조화된 로깅 지원 |

---

## 🔧 BuildKit 활성화 방법

### 🖥️ Docker Desktop (Mac/Windows)

- 기본으로 활성화되어 있음

### 🐧 Linux 환경

```bash
export DOCKER_BUILDKIT=1
```

또는 system-wide 설정 (e.g. `/etc/docker/daemon.json`):

```json
{
  "features": {
    "buildkit": true
  }
}
```

---

## 🛠️ BuildKit 사용 예시

```bash
DOCKER_BUILDKIT=1 docker build -t myapp:latest .
```

또는 Buildx 사용 시 BuildKit은 자동 활성화됨:

```bash
docker buildx build ...
```

---

## 📦 BuildKit 주요 기능

### ✅ 1. 고급 캐시 제어

```dockerfile
# Dockerfile 예시
FROM node:18 AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci            # 여기서 캐시 생성됨
COPY . ./
RUN npm run build     # 소스 코드 변경 시에만 재실행
```

> `RUN`, `COPY`, `ADD` 등의 명령은 Dockerfile 순서에 따라 **캐시 레이어**를 생성함

---

### ✅ 2. `--build-arg BUILDKIT_INLINE_CACHE=1` 사용

```bash
docker build \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  --tag yourname/myapp:latest \
  --cache-from=yourname/myapp:latest \
  .
```

| 항목 | 설명 |
|------|------|
| `BUILDKIT_INLINE_CACHE` | 이전 이미지에 캐시 메타데이터 포함 |
| `--cache-from` | 기존 이미지를 캐시 소스로 활용 |

※ `--push` 할 때 캐시 정보도 포함시켜야, CI 등에서 재사용 가능

---

## 🎯 캐시 전략 Best Practice

| 전략 | 설명 |
|------|------|
| 불변의 의존성 먼저 COPY | `COPY requirements.txt` → `RUN pip install` |
| 자주 변경되는 파일은 나중에 COPY | `COPY . .`는 마지막에 |
| `.dockerignore` 적극 활용 | 빌드 캐시 오염 방지 (e.g., `node_modules`, `.git`) |
| 멀티 스테이지 빌드 | 빌드 결과만 최종 이미지에 포함하여 캐시 적게 사용 |
| `buildx + inline cache` | CI/CD 환경에서 캐시 재사용 가능 |

---

## 🧪 예시: Node.js 앱 빌드 캐시 전략

```Dockerfile
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:18-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
CMD ["node", "dist/index.js"]
```

→ 의존성과 소스를 분리하여, **의존성 캐시 재사용**이 극대화됨

---

## 📁 `.dockerignore` 예시

```dockerignore
node_modules
.git
*.log
Dockerfile
README.md
```

→ 불필요한 파일은 COPY 대상에서 제외하여 캐시 효율성 상승

---

## 🧑‍💻 CI/CD에서 BuildKit + 캐시 활용 (GitHub Actions)

```yaml
- name: Login to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_USER }}
    password: ${{ secrets.DOCKER_PASS }}

- name: Build & Push with Cache
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: yourname/myapp:latest
    cache-from: type=registry,ref=yourname/myapp:latest
    cache-to: type=inline
```

| cache-from | 이전 이미지에서 캐시 추출 |
|------------|----------------------------|
| cache-to   | 새로 만든 이미지에 캐시 저장 |

---

## 🔐 BuildKit의 고급 기능

| 기능 | 설명 |
|------|------|
| Secret mount | `--secret id=mykey,src=key.txt` → `RUN --mount=type=secret` |
| SSH agent | `RUN --mount=type=ssh` → Git 접근 |
| Output control | `--output type=tar`, `type=local` 등 다양한 출력 지원 |

---

## 📚 참고 문서

- [BuildKit 공식 문서](https://docs.docker.com/build/buildkit/)
- [Docker build-push-action v5](https://github.com/docker/build-push-action)
- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

---

## ✅ 요약

| 항목 | 설명 |
|------|------|
| BuildKit | 병렬성, 캐시, 보안 강화된 빌드 엔진 |
| 캐시 전략 | 순서를 고려해 Dockerfile 구성 |
| CI 활용 | `buildx`, `inline cache`, `--cache-from` 적극 사용 |
| 최적화 | `.dockerignore`, 멀티 스테이지 빌드 활용 |
