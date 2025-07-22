---
layout: post
title: Docker - exec vs attach
date: 2025-01-14 19:20:23 +0900
category: Docker
---
# 🧠 Dockerfile & 이미지 최적화 베스트 프랙티스

---

## 🧊 1. Docker 캐싱(Cache) 활용

Docker는 `Dockerfile` 명령어 단위로 레이어를 캐시하여  
**변경되지 않은 명령은 재실행하지 않고 재사용**합니다.

### ✅ 원칙
- 위에서 아래로 캐시 적용
- **이전 레이어가 변경되면 이후 레이어는 전부 무효화**

### 🔍 예시

```Dockerfile
# 잘못된 예 (캐시 무효화)
COPY . .                 # 소스코드 전체 복사 → 자주 바뀜
RUN pip install -r requirements.txt

# 좋은 예
COPY requirements.txt .  # 먼저 복사 (변경 적음)
RUN pip install -r requirements.txt
COPY . .
```

### 🛠️ 전략
| 항목 | 설명 |
|------|------|
| 의존성 먼저 COPY | requirements.txt, package.json 등을 먼저 COPY |
| 빌드 도구 최소화 | `RUN apt-get clean && rm -rf /var/lib/apt/lists/*` |
| 파일 분리 | 자주 바뀌는 코드와 자주 안 바뀌는 코드 분리

---

## 📁 2. .dockerignore 활용

`.dockerignore`는 Docker 빌드 시 제외할 파일/폴더를 지정하는 파일입니다.  
**불필요한 파일이 이미지에 포함되지 않도록 필수적으로 사용해야 합니다.**

### ✅ 예시

```bash
# .dockerignore
.git/
__pycache__/
*.pyc
node_modules/
.env
Dockerfile
docker-compose.yml
README.md
```

### 🔥 효과
- 빌드 성능 향상 (불필요한 복사 제거)
- 보안 강화 (비밀번호 포함 파일 제외)
- 이미지 크기 감소

> `.dockerignore` 없이는 `.git`, `.env`까지 이미지에 포함될 수 있음!

---

## 🪶 3. 경량 이미지 사용

### ✅ 공식 경량 베이스 이미지

| 언어/플랫폼 | 경량 이미지 추천 |
|-------------|-------------------|
| Python | `python:3.X-slim`, `python:3.X-alpine` |
| Node.js | `node:XX-slim`, `node:XX-alpine` |
| Go | `scratch` or `distroless` |
| Java | `eclipse-temurin:XX-jre-alpine` |

> `alpine`은 매우 작지만 일부 패키지 호환성 이슈가 있을 수 있으니 `slim`도 고려

---

## 🧼 4. 불필요한 레이어 제거

- 여러 RUN 명령어를 **하나로 합치기**
```Dockerfile
# 비효율
RUN apt update
RUN apt install -y curl
RUN apt clean

# 개선
RUN apt update && apt install -y curl && rm -rf /var/lib/apt/lists/*
```

- 임시 파일 제거
```Dockerfile
RUN wget file.zip && unzip file.zip && rm file.zip
```

- 이미지 빌드 후 `multi-stage`로 결과만 추출

---

## 🧩 5. 멀티 스테이지 빌드 활용

**빌드 도구와 실행 환경을 분리**하여 최종 이미지를 경량화합니다.

```Dockerfile
# Stage 1: 빌드
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

# Stage 2: 실행
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

- 이미지 크기 수십 MB 감소
- 보안 위험도 감소

---

## 📌 6. 최소 권한/보안 고려

### ✅ 사용자 권한 변경

기본적으로 Docker는 `root`로 실행됩니다.  
보안을 위해 일반 사용자로 실행하는 것이 좋습니다.

```Dockerfile
RUN adduser --disabled-password myuser
USER myuser
```

---

## 🔧 7. 기타 팁 요약

| 항목 | 설명 |
|------|------|
| `CMD`는 JSON 배열로 | `"CMD ["node", "app.js"]"` (쉘 레벨 문제 회피) |
| 레이어 캐시 유지를 위해 파일명 고정 | `COPY requirements.txt`, `not requirements-dev.txt` |
| 빌드 인자 사용 | `ARG NODE_ENV`, `ENV` 변수 설정 |
| 적절한 이미지 태깅 | `myapp:1.0.0`, `myapp:latest` |
| 문서화 | `LABEL description="..." version="..."` |

---

## ✅ 결론 요약

| 목표 | 실전 전략 |
|------|-----------|
| 캐시 효율 | 의존성 먼저 COPY, RUN 합치기 |
| 용량 최소화 | 슬림 이미지, 멀티 스테이지 빌드 |
| 보안 강화 | `.dockerignore`, USER 설정 |
| 유지보수 | 명확한 태깅, LABEL 메타데이터 |
| 빌드 속도 | 캐시와 ignore를 잘 조합 |

---

## 🔎 참고 명령어

```bash
# 이미지 용량 확인
docker images

# 이미지 레이어 확인
docker history <image_id>

# 이미지 내 파일 구조 확인 (Dive 도구 활용 추천)
dive <image_name>

# 이미지 실행 파일 확인
docker run -it --rm <image> sh
```
