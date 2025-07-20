---
layout: post
title: Docker - 첫 번째 Docker 컨테이너
date: 2025-01-07 19:20:23 +0900
category: Docker
---
# 🐳 첫 번째 Docker 컨테이너 실행하기: Hello World

Docker가 제대로 설치되었는지 확인하기 위한 가장 기본적인 실습입니다.

---

## ✅ 1. Docker 데몬이 실행 중인지 확인

Docker Desktop을 사용 중이라면 **트레이 아이콘이 활성화**되어 있는지 확인합니다.

터미널에서 아래 명령어를 실행해 데몬이 살아 있는지 점검합니다:

```bash
docker version
```

정상 출력 예시:
```
Client: Docker Engine - Community
 Server: Docker Engine - Community
```

---

## ✅ 2. Hello World 이미지 다운로드 및 실행

```bash
docker run hello-world
```

### 🔍 이 명령은 다음 과정을 수행합니다:

1. **`hello-world` 이미지가 로컬에 없다면**,  
   → Docker Hub에서 자동으로 **이미지를 pull (다운로드)** 합니다.
2. 이미지를 기반으로 **새로운 컨테이너를 생성**합니다.
3. 컨테이너가 실행되며, 내부 프로그램이 "Hello from Docker!" 메시지를 출력합니다.
4. 실행 종료 후 컨테이너는 종료 상태로 남아 있게 됩니다.

---

## ✅ 3. 실행 결과 예시

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

이 메시지가 보이면 **Docker 설치 및 컨테이너 실행 환경이 성공적으로 작동 중**이라는 뜻입니다 🎉

---

## 🧱 구조 요약

```text
+------------------------+
|  hello-world 이미지 다운로드  |
+------------------------+
             ↓
+------------------------+
|  컨테이너 생성 및 실행      |
+------------------------+
             ↓
|   "Hello from Docker!" 출력   |
```

---

## ✅ 4. 실행 이후 컨테이너 확인 (종료 상태)

```bash
docker ps -a
```

결과 예시:
```
CONTAINER ID   IMAGE         COMMAND       STATUS                      NAMES
e98dbd4c5cc5   hello-world   "/hello"      Exited (0) 10 seconds ago   dreamy_lalande
```

→ `Exited` 상태: hello-world는 메시지만 출력하고 바로 종료되는 컨테이너입니다.

---

## ✅ 5. 이미지 확인

```bash
docker images
```

결과 예시:
```
REPOSITORY     TAG       IMAGE ID       CREATED        SIZE
hello-world    latest    d1165f221234   2 weeks ago    13.3kB
```

---

## ✅ 6. 정리 명령어 (옵션)

```bash
# 종료된 컨테이너 삭제
docker rm <컨테이너 ID>

# 이미지 삭제
docker rmi hello-world
```

---

## 📌 추가: 컨테이너와 이미지 관계

- **이미지**: 불변 템플릿 (읽기 전용)
- **컨테이너**: 이미지 기반으로 생성된 **실행 인스턴스** (읽기/쓰기 가능)

> 하나의 이미지로 여러 개의 컨테이너를 실행할 수 있습니다.

---

## ✅ 요약 명령어 정리

| 작업 | 명령어 |
|------|--------|
| 컨테이너 실행 | `docker run hello-world` |
| 컨테이너 목록 | `docker ps -a` |
| 이미지 목록 | `docker images` |
| 컨테이너 삭제 | `docker rm <id>` |
| 이미지 삭제 | `docker rmi hello-world` |