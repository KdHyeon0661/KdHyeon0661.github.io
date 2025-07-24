---
layout: post
title: Docker - Docker 경량화 이미지 만들기
date: 2025-03-10 20:20:23 +0900
category: Docker
---
# 🏋️ Docker 경량화 이미지 만들기 (alpine 등)

---

## 📌 왜 이미지 경량화가 중요한가?

| 항목 | 이유 |
|------|------|
| 🕒 빌드/배포 속도 | 이미지가 작을수록 빌드·전송 속도가 빠름 |
| 💾 저장소 절약 | Docker Hub, CI/CD 캐시, 레지스트리 비용 감소 |
| 📦 공격 표면 감소 | 패키지 수 줄여 보안 취약점 최소화 |
| 📉 운영 리소스 감소 | 불필요한 OS 구성 제거로 컨테이너 경량화 |

---

## 🧱 일반 vs 경량 이미지 예시

| 이미지 | 용량 | 설명 |
|--------|------|------|
| `python:3.11` | 약 900MB | 풀 데비안 기반, 전체 라이브러리 포함 |
| `python:3.11-slim` | 약 60~80MB | 불필요한 것 제거 |
| `python:3.11-alpine` | 약 20~40MB | 초경량 Alpine Linux 기반 |

---

## 🐧 Alpine Linux란?

- 5MB 수준의 초경량 리눅스 배포판
- `musl libc`와 `busybox` 기반
- 보안에 강하고, 빠르게 시작됨
- 패키지 매니저는 `apk`

---

## 🧪 실전 예제: Flask 앱 경량화 (alpine 사용)

### ✅ 일반 Dockerfile (용량 큼)

```dockerfile
FROM python:3.11

WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

📦 약 **900MB** 이미지 생성됨

---

### ✅ 경량화 Dockerfile (alpine)

```dockerfile
FROM python:3.11-alpine

# 시스템 패키지 설치 (예: gcc 등)
RUN apk add --no-cache gcc musl-dev libffi-dev

WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
CMD ["python", "app.py"]
```

📦 약 **40MB** 수준으로 줄어듦  
단, C 확장 모듈 컴파일 필요시 `build-deps`를 반드시 설치해야 함

---

## 📁 requirements.txt 예시

```
Flask==2.3.3
requests
```

> 이 정도는 Alpine에서 바로 설치 가능하지만,  
> `Pillow`, `lxml`, `psycopg2` 등은 추가 패키지 필요

---

## 🧰 Multi-stage Build로 경량화 + 의존성 제거

```dockerfile
FROM python:3.11-alpine AS builder

RUN apk add --no-cache gcc musl-dev libffi-dev

WORKDIR /install
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt


FROM python:3.11-alpine

COPY --from=builder /install /usr/local
WORKDIR /app
COPY . .

CMD ["python", "app.py"]
```

### ✅ 이점

| 항목 | 효과 |
|------|------|
| 컴파일 환경 제거 | 최종 이미지에 build 도구 없음 |
| 캐시 사용 최적화 | 단계별 분리로 레이어 재사용 가능 |
| 결과 용량 감소 | 필요한 파일만 포함됨 (이미지 30MB 이하 가능) |

---

## 🛠️ 추가 팁: 이미지 최적화 Best Practices

| 전략 | 설명 |
|------|------|
| ✅ `.dockerignore` 설정 | `.git`, `__pycache__`, `*.md`, `node_modules` 등 제외 |
| ✅ `--no-cache-dir` | `pip install` 시 캐시 폴더 제거 |
| ✅ 하나의 `RUN`으로 묶기 | 레이어 수 줄이기 (`&&` 활용) |
| ✅ 불필요한 `apt`, `apk` 삭제 | `rm -rf /var/cache/...` 등으로 정리 |
| ✅ 필요 시 `distroless` 사용 | 운영체제 자체 제거 (단, 디버깅 어려움) |

---

## 🧪 비교: 이미지 사이즈 측정

```bash
docker build -t my-app:default -f Dockerfile.default .
docker build -t my-app:alpine -f Dockerfile.alpine .

docker images | grep my-app
```

---

## 🛡️ 경량화 + 보안 고려 시 distroless 도입도 가능

```dockerfile
# 빌더 이미지
FROM python:3.11-slim as builder
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt

# distroless (실행만 가능한 베이스)
FROM gcr.io/distroless/python3
COPY --from=builder /app /app
WORKDIR /app
CMD ["app.py"]
```

- `gcr.io/distroless/python3`는 shell도 없는 ultra-minimal 이미지
- 단점: 디버깅/확장 매우 어려움

---

## ✅ 정리 요약

| 전략 | 설명 |
|------|------|
| Alpine 사용 | 가장 기본적이고 안전한 경량화 방식 |
| Multi-stage Build | 빌드와 런타임 환경 분리 |
| `.dockerignore` | 불필요한 파일 제외 |
| `--no-cache-dir`, `--no-install-recommends` | 캐시, 불필요한 의존성 제거 |
| distroless | 궁극의 경량 + 보안 (단, 디버깅 어려움) |

---

## 📚 참고 자료

- [Alpine Linux 공식 문서](https://alpinelinux.org/)
- [Docker Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Distroless 이미지란?](https://github.com/GoogleContainerTools/distroless)
