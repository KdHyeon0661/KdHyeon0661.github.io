---
layout: post
title: Docker - Multi-arch Docker
date: 2025-03-24 21:20:23 +0900
category: Docker
---
# 🏗️ Multi-arch Docker 이미지 만들기 (ARM/M1 대응)

---

## 📌 왜 Multi-arch 이미지가 필요한가?

| 항목 | 설명 |
|------|------|
| 플랫폼 다양성 | x86_64, ARM64 등 다양한 아키텍처에서 실행 가능 |
| M1/M2 Mac | ARM 기반이므로 기존 x86 이미지와 호환 문제 발생 |
| CI/CD 통합 | 다양한 서버/개발자 환경에서 동일한 이미지 제공 |
| 공식 이미지 | 대부분 Multi-arch로 배포 (e.g., `python`, `nginx`)

---

## ✅ 핵심 도구: `docker buildx`

`buildx`는 `docker build`의 확장 기능으로, **cross-platform 이미지 빌드** 및 **multi-arch manifest** 생성을 지원합니다.

---

## 🔧 1. Buildx 활성화 확인

```bash
docker buildx version
```

없다면 Docker Desktop 최신 버전 설치 필요  
(또는 `docker buildx install` 시도)

---

## 🔨 2. Buildx 빌더 생성

```bash
docker buildx create --use --name mybuilder
docker buildx inspect --bootstrap
```

- `--use`: 생성된 빌더를 기본으로 설정
- `--bootstrap`: QEMU 기반 cross-building 환경 초기화

---

## 🌍 3. 빌드할 플랫폼 지정

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t yourname/myapp:latest \
  --push .
```

### 주요 옵션 설명

| 옵션 | 설명 |
|------|------|
| `--platform` | 타겟 아키텍처 지정 (여러 개 가능) |
| `--push` | 빌드 후 Docker Hub/Registry로 업로드 |
| `--load` | 로컬에만 저장 (단일 플랫폼만 가능) |

---

## 🧪 실습: 간단한 이미지 만들기

```Dockerfile
# Dockerfile
FROM alpine:latest
CMD ["echo", "Hello from Multi-arch!"]
```

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t yourname/hello-multiarch:latest \
  --push .
```

> 푸시된 이미지는 `docker manifest`로 자동 분기됨

---

## 🔍 4. 이미지 아키텍처 확인

```bash
docker buildx imagetools inspect yourname/hello-multiarch:latest
```

출력 예:

```
Name:      docker.io/yourname/hello-multiarch:latest
MediaType: application/vnd.docker.distribution.manifest.list.v2+json
...
  - platform:
      architecture: amd64
      os: linux
  - platform:
      architecture: arm64
      os: linux
```

---

## 🧠 5. QEMU 설치 (로컬 빌드 시 필요한 경우)

```bash
docker run --privileged --rm tonistiigi/binfmt --install all
```

- QEMU를 통해 다른 아키텍처 에뮬레이션 가능

---

## 🌎 6. GitHub Actions에서 Multi-arch 빌드

```yaml
name: Build Multi-arch Image

on:
  push:
    branches: [ main ]

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Set up QEMU
      uses: docker/setup-qemu-action@v3

    - name: Set up Buildx
      uses: docker/setup-buildx-action@v3

    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USER }}
        password: ${{ secrets.DOCKER_PASS }}

    - name: Build & Push Multi-arch
      uses: docker/build-push-action@v5
      with:
        push: true
        platforms: linux/amd64,linux/arm64
        tags: yourname/myapp:latest
```

---

## 🛠️ 7. 빌드 최적화 팁

| 팁 | 설명 |
|-----|------|
| `alpine` | 경량 베이스 이미지로 빌드 속도 향상 |
| 멀티스테이지 빌드 | 빌드 환경과 런타임 분리 |
| 캐시 사용 | `--build-arg BUILDKIT_INLINE_CACHE=1` + `cache-from` 설정 |
| `.dockerignore` | 불필요한 파일 제외하여 빌드 최적화 |

---

## ❗주의사항

| 항목 | 주의 |
|------|------|
| 로컬에서 `--platform`과 `--load` 동시 사용 불가 | `--platform` 여러 개 → 반드시 `--push` 필요 |
| 일부 바이너리/패키지는 아키텍처 제한 존재 | 예: `x86` 전용 패키지 |
| base image가 Multi-arch 아니면 실패 가능 | `python`, `alpine` 등은 대부분 지원 |

---

## 📚 참고 자료

- [Docker Buildx 공식 문서](https://docs.docker.com/buildx/working-with-buildx/)
- [Tonistiigi’s buildx 소개](https://github.com/docker/buildx)
- [Docker multi-arch 예제](https://github.com/docker/build-push-action)

---

## ✅ 요약

| 단계 | 설명 |
|------|------|
| Buildx 생성 | `docker buildx create --use` |
| 빌드 | `docker buildx build --platform ... --push` |
| 검증 | `docker buildx imagetools inspect` |
| 자동화 | GitHub Actions, GitLab CI 등 활용 |