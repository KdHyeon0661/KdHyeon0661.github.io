---
layout: post
title: Docker - exec vs attach
date: 2025-01-14 19:20:23 +0900
category: Docker
---
# 🔍 `docker exec` vs `docker attach` 차이점 정리

---

## 📌 공통점

| 항목 | 설명 |
|------|------|
| 목적 | 실행 중인 컨테이너에 **접속하거나 명령을 실행**함 |
| 전제 조건 | 컨테이너가 `running` 상태여야 함 |
| 사용 용도 | 디버깅, 쉘 진입, 로그 확인 등 |

---

## 🧨 차이점 요약

| 항목 | `docker exec` | `docker attach` |
|------|----------------|------------------|
| 주요 기능 | **새로운 프로세스** 실행 | 기존 컨테이너의 **기본 터미널**에 연결 |
| 병렬 실행 | O (여러 개 가능) | X (한 번에 하나만 연결) |
| 분리 가능 | O (`exit`로 종료) | X (`exit` 하면 컨테이너도 종료될 수 있음) |
| 출력 동기화 | 독립적인 출력 | 기본 STDOUT/STDIN 연결 |
| 사용 용도 | 쉘 접속, 디버깅, 명령 실행 | 기본 프로그램 직접 관찰 및 제어 |

---

## 📦 `docker exec` 자세히 보기

```bash
docker exec -it [컨테이너이름] bash
```

- **컨테이너 내에 새로운 프로세스 생성**
- `-it`: interactive + pseudo-TTY (쉘처럼 사용)
- 여러 명령 실행 가능 (bash, top, tail 등)

### 예시:

```bash
docker exec -it mycontainer bash
```

→ 컨테이너 내 bash 쉘 진입  
(기존 앱은 그대로 실행 중, bash는 별도)

---

## 📺 `docker attach` 자세히 보기

```bash
docker attach [컨테이너이름]
```

- 컨테이너의 **기본 프로세스의 STDIN/STDOUT**에 연결
- 예: nginx, flask 등 실행 중인 서버 로그나 입력 제어에 연결됨
- **종료 시 컨테이너가 멈출 수 있음** (기본 PID 1 프로세스가 종료됨)

### 예시:

```bash
docker run -it ubuntu
# 컨테이너가 실행 중인 상태에서
docker attach [컨테이너ID]
```

> `Ctrl+C` 또는 `exit` 시 컨테이너까지 종료될 수 있음  
> 안전하게 분리하려면 `Ctrl + P + Q` 사용

---

## 🛠 실전 팁

| 상황 | 추천 명령어 |
|------|-------------|
| 디버깅 또는 쉘 접속 | `docker exec -it [컨테이너] bash/sh` |
| 컨테이너 실행 로그 실시간 확인 | `docker logs -f [컨테이너]` |
| 단순 연결 및 입력 제어 | `docker attach` (주의 필요) |

---

## 🔐 보안 관점 요약

- `exec`는 **제한된 단일 작업 실행**이라 더 안전함
- `attach`는 **전체 STDIN/STDOUT 공유**라 위험할 수 있음
- 운영환경에서는 `attach`는 가급적 사용하지 않음

---

## ✅ 결론 요약

| 구분 | exec | attach |
|------|------|--------|
| 별도 쉘 제공 | ✅ | ❌ |
| 여러 번 연결 가능 | ✅ | ❌ |
| 종료해도 컨테이너 유지 | ✅ | ❌ (종료될 수 있음) |
| 디버깅/유지보수 | 최적 | 제한적 |
| 실무 권장도 | 매우 높음 | 낮음 (주의 필요) |

---

## 📌 안전하게 빠져나오는 법

- `exec` 쉘에서는 `exit` 입력
- `attach` 시에는 `Ctrl + P` → `Ctrl + Q` 순서로 분리 (detach)