---
layout: post
title: Docker - Docker 이미지 커스터마이징
date: 2025-01-11 21:20:23 +0900
category: Docker
---
# 🏗️ Docker 이미지 생성 및 커스터마이징

---

## 📦 Docker 이미지란?

- 컨테이너 실행을 위한 **읽기 전용 템플릿**
- 이미지 = 여러 개의 레이어(layer)로 구성
- Dockerfile을 통해 이미지를 생성 및 커스터마이징 가능

---

## 🧾 Dockerfile이란?

> 이미지를 생성하기 위한 **명령어 모음 파일**

```bash
# 기본 파일명은 Dockerfile
# 같은 디렉토리에 있는 파일들을 기준으로 build
```

### 예시 구조

```
my-app/
├── Dockerfile
├── app.py
└── requirements.txt
```

---

## 🧱 Dockerfile 기본 예제 (Python Flask)

```Dockerfile
# 1. 베이스 이미지 지정
FROM python:3.10-slim

# 2. 작업 디렉토리 생성
WORKDIR /app

# 3. 필요한 파일 복사
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 4. 앱 코드 복사
COPY . .

# 5. 실행 명령 지정
CMD ["python", "app.py"]
```

---

## 🏗️ 이미지 빌드하기: `docker build`

```bash
docker build -t my-flask-app .
```

| 옵션 | 설명 |
|------|------|
| `-t` | 이미지에 태그(이름) 지정 |
| `.`  | 현재 디렉토리를 컨텍스트로 사용 |

> 빌드 결과는 `docker images` 명령어로 확인 가능

---

## 🚀 이미지로 컨테이너 실행

```bash
docker run -d -p 5000:5000 my-flask-app
```

> 앱이 포트 5000에서 실행된다면 브라우저로 `http://localhost:5000` 접속 가능

---

## 🧩 Dockerfile 주요 명령어 정리

| 명령어 | 설명 |
|--------|------|
| `FROM` | 베이스 이미지 지정 (필수) |
| `WORKDIR` | 작업 디렉토리 설정 |
| `COPY` | 로컬 파일을 이미지로 복사 |
| `RUN` | 쉘 명령어 실행 (빌드시 실행됨) |
| `CMD` | 컨테이너 시작 시 실행 명령 (하나만 존재) |
| `ENTRYPOINT` | 고정된 실행 명령 설정 |
| `ENV` | 환경 변수 설정 |
| `EXPOSE` | 사용할 포트 명시 (문서용, 실제 포트는 -p 필요) |

---

## 🛠️ 커스터마이징 전략

| 전략 | 설명 |
|------|------|
| 최소 베이스 이미지 사용 | 예: `alpine`, `slim` 버전 |
| 멀티 스테이지 빌드 | 빌드 이미지와 런타임 이미지를 분리하여 최적화 |
| `.dockerignore` | 빌드 시 제외할 파일 설정 (`.git`, `node_modules` 등) |
| 환경 변수 설정 | `ENV`, `.env` 파일 활용 |

---

## 📁 .dockerignore 예시

```dockerignore
__pycache__/
*.pyc
.git/
node_modules/
.env
```

---

## 🧪 실습: Nginx 커스터마이징 예제

```Dockerfile
FROM nginx:latest
COPY ./html /usr/share/nginx/html
```

```bash
docker build -t custom-nginx .
docker run -d -p 8080:80 custom-nginx
```

> 커스터마이징한 HTML을 포함한 Nginx 서버 실행

---

## 🔁 이미지 업데이트 & 재빌드

- `Dockerfile`, 소스코드 변경 후 다시 빌드:
```bash
docker build -t my-flask-app .
```

- 기존 컨테이너 중지 및 삭제:
```bash
docker stop my-flask-app
docker rm my-flask-app
```

- 새로운 이미지로 컨테이너 재실행:
```bash
docker run -d --name my-flask-app -p 5000:5000 my-flask-app
```

---

## ✅ 정리

| 단계 | 설명 |
|------|------|
| 1. `Dockerfile` 작성 | 명령어 기반으로 이미지 커스터마이징 |
| 2. `docker build` | 이미지 생성 |
| 3. `docker run` | 이미지로 컨테이너 실행 |
| 4. 반복 | 이미지 수정 → 재빌드 → 재배포 |

---

## 📌 참고

- `Docker Hub`에 이미지를 푸시하고 배포할 수도 있음 (`docker push`)
- 보안과 성능을 위해 항상 **최소한의 레이어**와 **최신 패키지**를 유지하세요
- 실무에서는 **CI/CD 파이프라인에서 자동 빌드**를 구성하기도 합니다