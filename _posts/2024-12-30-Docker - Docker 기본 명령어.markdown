---
layout: post
title: 알고리즘 - 알고리즘이란
date: 2024-12-26 19:20:23 +0900
category: 알고리즘
---
# 🐳 Docker 기본 명령어 정리

Docker의 주요 작업 흐름은 다음과 같이 정리할 수 있습니다:

> `이미지 다운로드 → 컨테이너 실행 → 상태 확인 → 정지 → 삭제`

---

## 📥 1. 이미지 다운로드: `docker pull`

```bash
docker pull [이미지이름][:태그]
```

### 예시:
```bash
docker pull ubuntu:20.04
docker pull nginx
```

> `:태그`를 생략하면 `latest` 버전을 가져옵니다.

---

## 🚀 2. 컨테이너 실행: `docker run`

```bash
docker run [옵션] [이미지이름] [명령어]
```

### 자주 쓰는 옵션:
| 옵션 | 설명 |
|------|------|
| `-it` | 터미널 연결 (interactive + tty) |
| `--rm` | 실행 후 자동 삭제 |
| `-d` | 백그라운드 실행 (detached) |
| `--name` | 컨테이너 이름 지정 |
| `-p` | 포트 매핑 (예: `-p 8080:80`) |

### 예시:
```bash
docker run -it ubuntu bash           # Ubuntu 컨테이너에서 bash 실행
docker run --rm alpine echo hello    # 일회성 실행 후 자동 삭제
docker run -d -p 8080:80 nginx       # Nginx 서버 백그라운드 실행
```

---

## 🔍 3. 컨테이너 목록 확인: `docker ps`

```bash
docker ps              # 실행 중인 컨테이너만 표시
docker ps -a           # 종료된 컨테이너 포함 전체 목록
```

---

## ⏹️ 4. 컨테이너 정지: `docker stop`

```bash
docker stop [컨테이너ID or 이름]
```

### 예시:
```bash
docker stop web_server
```

> 종료되면 `docker ps` 목록에서는 사라지고, `docker ps -a`에 나타납니다.

---

## ❌ 5. 컨테이너 삭제: `docker rm`

```bash
docker rm [컨테이너ID or 이름]
```

### 예시:
```bash
docker rm sleepy_snyder
docker rm -f web_server     # 강제 종료 후 삭제
```

---

## 🧼 6. 이미지 삭제: `docker rmi`

```bash
docker rmi [이미지ID or 이름]
```

### 예시:
```bash
docker rmi ubuntu
```

---

## 🔁 7. 실행 중인 컨테이너에 접속: `docker exec`

```bash
docker exec -it [컨테이너이름] [명령어]
```

### 예시:
```bash
docker exec -it web_server bash
```

---

## 📋 보너스: 컨테이너 로그 보기

```bash
docker logs [컨테이너이름]
```

---

## ✅ 명령어 흐름 예시

```bash
docker pull nginx
docker run -d --name web_server -p 8080:80 nginx
docker ps
docker stop web_server
docker rm web_server
docker rmi nginx
```

---

## 🧾 요약 명령어 표

| 목적 | 명령어 |
|------|--------|
| 이미지 다운로드 | `docker pull 이미지` |
| 컨테이너 실행 | `docker run 옵션 이미지` |
| 컨테이너 목록 | `docker ps [-a]` |
| 컨테이너 정지 | `docker stop 이름` |
| 컨테이너 삭제 | `docker rm 이름` |
| 이미지 삭제 | `docker rmi 이미지` |
| 컨테이너 접속 | `docker exec -it 이름 bash` |
| 로그 확인 | `docker logs 이름` |