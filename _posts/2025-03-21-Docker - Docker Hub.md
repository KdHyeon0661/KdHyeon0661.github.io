---
layout: post
title: Docker - Docker Hub
date: 2025-03-21 19:20:23 +0900
category: Docker
---
# 🐳 Docker Hub 사용법 완전 정복

---

## 📌 Docker Hub란?

- Docker Inc.에서 운영하는 **공식 이미지 저장소**
- 컨테이너 이미지를 **저장, 공유, 검색, 배포**할 수 있음
- 기본적으로 `docker pull`, `docker push` 명령어에 사용됨

---

## 1️⃣ 가입 및 로그인

### 🔐 가입하기

- URL: [https://hub.docker.com](https://hub.docker.com)
- 이메일, 사용자 이름(고유), 비밀번호 설정

### 🔑 로그인

```bash
docker login
```

- 명령어 실행 후 Docker ID, 비밀번호 입력
- 로그인 후 `~/.docker/config.json`에 인증 정보 저장

---

## 2️⃣ Docker Hub에서 이미지 검색

```bash
docker search nginx
```

출력 예시:

```
NAME                DESCRIPTION                                     STARS     OFFICIAL
nginx               Official build of Nginx.                        17000+     [OK]
bitnami/nginx       Bitnami nginx Docker Image                      600+
```

---

## 3️⃣ 이미지 다운로드 (pull)

```bash
docker pull nginx
docker pull ubuntu:22.04
```

- `docker pull <이미지>:<태그>` 형식
- 생략 시 `latest`가 기본

---

## 4️⃣ Docker Hub에 이미지 업로드 (push)

### 🛠️ 1. 태그 지정

```bash
docker tag myapp:latest <DockerID>/myapp:latest
```

예:

```bash
docker tag myapp:latest johndoe/myapp:latest
```

### 🚀 2. 푸시 (push)

```bash
docker push johndoe/myapp:latest
```

> Docker ID와 일치하는 리포지토리 이름만 푸시 가능

---

## 5️⃣ 개인/공용 리포지토리 만들기

1. [https://hub.docker.com/repositories](https://hub.docker.com/repositories) 접속
2. `Create Repository` 클릭
3. 이름, 설명, Visibility(Public/Private) 선택
4. 생성 후 `docker push`로 업로드

---

## 6️⃣ 자동 빌드 (CI/CD)

Docker Hub는 GitHub/GitLab과 연결하여 **자동으로 이미지 빌드** 가능

### ⚙️ 설정 방법

1. Docker Hub → 리포지토리 → `Builds` 탭
2. GitHub 계정 연결
3. Git 브랜치/디렉토리 기준 자동 빌드 설정
4. `Dockerfile` 위치 지정
5. 코드 푸시 시 자동 빌드 & 배포

---

## 7️⃣ 팀 협업 기능

- 조직(Organization) 생성 가능
- 팀 별로 `Read`, `Write`, `Admin` 권한 부여
- 유료 요금제로 Private 이미지/팀 기능 확장 가능

---

## 8️⃣ 이미지 관리 (Tag 삭제, 보기)

### 🔍 이미지 목록 확인

```bash
docker images
```

### 🧹 로컬 이미지 삭제

```bash
docker rmi nginx
```

### 🔄 태그 목록 확인 (Web)

- Docker Hub 웹 → 리포지토리 클릭 → `Tags` 탭에서 확인

---

## 9️⃣ 이미지 버전 전략 예시

| 버전 태그 | 설명 |
|-----------|------|
| `latest` | 최신 개발 버전 (주의: 불안정할 수 있음) |
| `v1.0.0` | SemVer 기반 버전 |
| `stable` | 배포용 안정 버전 |
| `dev`, `test` | 개발·테스트용 이미지 |

---

## 🔐 10. 프라이빗 이미지 사용 예시

```bash
docker login
docker pull johndoe/private-app:latest
```

- 로그인 상태에서만 private 이미지 접근 가능
- 팀원도 같은 조직에 속해 있어야 접근 가능

---

## 🧪 실전 요약 명령어

```bash
# 로그인
docker login

# 이미지 빌드
docker build -t myapp .

# 태그 설정
docker tag myapp johndoe/myapp:latest

# Docker Hub에 업로드
docker push johndoe/myapp:latest

# 다른 곳에서 다운로드
docker pull johndoe/myapp:latest
```

---

## 💸 요금제 (2025 기준)

| 플랜 | 공용 Repo | 개인 Repo | 자동 빌드 | 팀 권한 |
|------|-----------|-----------|------------|----------|
| Free | 무제한    | 1개       | 200 빌드/월 | ❌ 없음 |
| Pro  | 무제한    | 무제한    | 더 많음     | ✅ 가능 |
| Team | 무제한    | 무제한    | CI/CD 무제한 | ✅ 가능 |

> 요금은 [https://www.docker.com/pricing/](https://www.docker.com/pricing/) 참조

---

## 📚 참고 링크

- [Docker Hub 공식 사이트](https://hub.docker.com/)
- [Docker Hub 사용 문서](https://docs.docker.com/docker-hub/)
- [자동 빌드 공식 문서](https://docs.docker.com/docker-hub/builds/)