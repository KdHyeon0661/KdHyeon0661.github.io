---
layout: post
title: Docker - GitHub Actions를 이용한 Docker 배포 자동화
date: 2025-03-03 20:20:23 +0900
category: Docker
---
# 🚀 GitHub Actions를 이용한 Docker 배포 자동화

---

## 📌 목표

- GitHub에 Push → 자동으로 Docker 이미지 빌드
- 빌드된 이미지를 Docker Hub 또는 GitHub Container Registry(GHCR)에 Push
- 서버에 자동 배포까지 연결 가능 (SSH 또는 Webhook 등 활용)

---

## ⚙️ 1. 기본 흐름

```
[코드 Push] → [GitHub Actions] → [Docker Build] → [이미지 Push] → [배포 트리거]
```

---

## 📁 프로젝트 구조 예시

```plaintext
my-app/
├── app/
│   └── main.py
├── Dockerfile
├── docker-compose.yml
└── .github/
    └── workflows/
        └── docker-publish.yml
```

---

## 🐳 Dockerfile 예시

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY . .
RUN pip install flask
CMD ["python", "app/main.py"]
```

---

## 🔐 2. Docker Hub 로그인 설정 (Secrets 등록)

### GitHub → Settings → `Secrets and variables` → `Actions`

| 이름 | 값 |
|------|----|
| `DOCKER_USERNAME` | Docker Hub ID |
| `DOCKER_PASSWORD` | Docker Hub Token (or password) |

---

## 🧪 3. GitHub Actions 워크플로우 생성

`.github/workflows/docker-publish.yml`

```yaml
name: 🚀 Docker Image CI/CD

on:
  push:
    branches: [ "main" ]  # main 브랜치에 push될 때 실행

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: 🔄 Checkout Repository
        uses: actions/checkout@v3

      - name: 🐳 Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: 🏗️ Build Docker Image
        run: |
          docker build -t ${{ secrets.DOCKER_USERNAME }}/my-app:latest .

      - name: 📦 Push to Docker Hub
        run: |
          docker push ${{ secrets.DOCKER_USERNAME }}/my-app:latest
```

---

## 🏷️ 태그 버전 자동화 (선택 사항)

```yaml
docker build -t ${{ secrets.DOCKER_USERNAME }}/my-app:${{ github.sha }} .
```

- `github.sha`: 커밋 해시 기반 태그
- 혹은 `github.ref_name`으로 브랜치/버전별 태깅 가능

---

## 🌐 4. 실제 배포 (서버 자동 업데이트)

### 방법 1: SSH 원격 서버 명령어 실행

```yaml
- name: 🚀 Deploy via SSH
  uses: appleboy/ssh-action@v1.0.0
  with:
    host: ${{ secrets.REMOTE_HOST }}
    username: ${{ secrets.REMOTE_USER }}
    key: ${{ secrets.REMOTE_SSH_KEY }}
    script: |
      docker pull yourname/my-app:latest
      docker stop my-app || true
      docker rm my-app || true
      docker run -d --name my-app -p 80:80 yourname/my-app:latest
```

필요한 추가 Secrets:

- `REMOTE_HOST` (서버 IP)
- `REMOTE_USER`
- `REMOTE_SSH_KEY` (비밀번호 대신 **SSH private key 사용**)

---

### 방법 2: Webhook 활용 (ex. Watchtower, Portainer 등)

- GitHub Action에서 `curl`로 webhook 호출
- 서버에 있는 Watchtower가 이미지 변경을 감지하여 자동 재시작

```yaml
- name: 🔔 Notify Webhook
  run: curl -X POST https://yourserver.com/deploy-hook
```

---

## 📚 참고 Action 목록

| Action | 설명 |
|--------|------|
| `actions/checkout` | 코드 체크아웃 |
| `docker/login-action` | Docker Hub 로그인 |
| `appleboy/ssh-action` | 원격 배포 자동화 |
| `actions/upload-artifact` | 로그/결과 파일 저장 |

---

## ✅ 요약

| 단계 | 작업 |
|------|------|
| 1 | Dockerfile로 앱 이미지 정의 |
| 2 | GitHub Actions로 CI/CD 구성 |
| 3 | Docker Hub에 로그인 후 이미지 Push |
| 4 | 서버에 자동으로 Pull 및 실행 (SSH 또는 Watchtower) |

---

## 🚧 고급 구성 예시

- `multi-stage build`로 빌드 경량화
- `docker-compose.yml`로 여러 서비스 조합
- `dev`/`prod` 분리된 배포 전략
- `.env` 파일도 Secrets로 관리
- GitHub Container Registry(GHCR)를 사용하는 보안 배포

---

## 📚 참고 자료

- [GitHub Actions 공식 문서](https://docs.github.com/actions)
- [Docker Hub 로그인 Action](https://github.com/docker/login-action)
- [appleboy/ssh-action](https://github.com/appleboy/ssh-action)
- [Watchtower - 자동 재배포](https://containrrr.dev/watchtower)
