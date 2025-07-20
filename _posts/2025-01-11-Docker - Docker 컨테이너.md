---
layout: post
title: Docker - Docker 컨테이너
date: 2025-01-11 19:20:23 +0900
category: Docker
---
# 📦 Docker 컨테이너란? 상태와 수명주기 자세히 정리

---

## 🐳 컨테이너(Container)란?

Docker 컨테이너는 **이미지로부터 생성된 실행 중(또는 일시 중지된) 인스턴스**입니다.  
마치 **프로그램의 실행 프로세스**처럼, 이미지를 기반으로 실행되며 독립된 환경을 제공합니다.

### 🔹 핵심 특징
- **이미지 기반**: 컨테이너는 이미지를 기반으로 생성됨
- **격리된 환경**: 각 컨테이너는 파일 시스템, 네트워크, PID 등 격리된 공간에서 실행됨
- **경량화**: 전체 OS가 아닌 **호스트 OS의 커널을 공유**하여 가볍고 빠름
- **일시적**: 컨테이너 자체는 휘발성 (종료 시 데이터 유실, 볼륨 사용 시 제외)

---

## 📐 컨테이너의 구조

```
+-------------------------------+
| Writable Container Layer     | ← 컨테이너에서만 존재 (임시 변경사항 저장)
+-------------------------------+
| Read-only Image Layers       | ← FROM ~ COPY까지의 이미지 계층
+-------------------------------+
```

> 컨테이너는 이미지를 읽기 전용으로 사용하고, 자신만의 변경 사항을 상단 쓰기 레이어에 저장합니다.

---

## 🔄 컨테이너 상태 (State)

컨테이너는 다양한 상태를 가지며, 각 상태에 따라 명령어가 달라집니다.

| 상태 | 설명 | 명령어 예시 |
|------|------|--------------|
| `created` | 이미지로부터 컨테이너를 만들었지만 실행되지 않음 | `docker create` |
| `running` | 현재 실행 중 | `docker run`, `docker start` |
| `paused` | 일시 중지 상태 (시그널, 스케줄러 차단) | `docker pause` |
| `exited` | 종료됨 (정상/비정상 종료 모두 포함) | `docker stop`, `docker kill` |
| `dead` | 비정상적으로 중단됨 (강제 종료 후 복구 실패 등) | 내부 오류 발생 시 |

> 상태 확인:  
```bash
docker ps -a
```

---

## 🔁 컨테이너 수명주기 (Lifecycle)

Docker 컨테이너는 다음과 같은 흐름으로 동작합니다:

```
(이미지)
   ↓
docker create → [created]
   ↓
docker start  → [running]
   ↓
docker stop   → [exited]
   ↓
docker rm     → 삭제됨
```

### 또는:

```
docker run     → [created → running → exited]
```

> `docker run`은 `create + start`를 동시에 실행하는 명령입니다.

---

## ✅ 수명주기 예시 명령어

```bash
# 이미지 기반으로 컨테이너 생성 (하지만 실행 안함)
docker create --name test ubuntu

# 생성된 컨테이너 실행
docker start test

# 컨테이너 상태 확인
docker ps -a

# 컨테이너 일시 중지 / 재개
docker pause test
docker unpause test

# 컨테이너 정지
docker stop test

# 컨테이너 삭제
docker rm test
```

---

## 💬 컨테이너 vs 이미지

| 항목 | 이미지 (Image) | 컨테이너 (Container) |
|------|----------------|------------------------|
| 정적 / 동적 | 정적 (변하지 않음) | 동적 (실행 중 상태 변화) |
| 파일 시스템 | 읽기 전용 | 읽기/쓰기 |
| 상태 | 없음 | 실행 중 / 중지 등 상태 존재 |
| 역할 | 템플릿 | 실행 인스턴스 |
| 생성 방식 | Dockerfile → `docker build` | 이미지 기반 → `docker run` 또는 `docker create` |

---

## 🧼 정리

- **컨테이너는 이미지의 실행 인스턴스**
- 컨테이너는 **임시 쓰기 계층**을 통해 자체적으로 변경된 데이터를 저장
- 상태는 `created`, `running`, `paused`, `exited`, `dead` 등으로 나뉨
- `docker ps`, `docker inspect`, `docker logs` 등을 통해 상태 및 로그 확인 가능
- 컨테이너는 종료 후에도 **볼륨이 없다면 변경 사항은 사라짐**

---

## 📌 관련 명령어 요약

| 명령어 | 설명 |
|--------|------|
| `docker run` | 이미지 기반 컨테이너 생성 + 실행 |
| `docker create` | 실행 없이 컨테이너 생성만 |
| `docker start` | 중지된 컨테이너 실행 |
| `docker stop` | 실행 중인 컨테이너 정지 |
| `docker pause / unpause` | 일시 정지 / 재개 |
| `docker rm` | 컨테이너 삭제 |
| `docker ps -a` | 모든 컨테이너 목록 확인 |
| `docker inspect` | 상세 정보 확인 |
| `docker logs` | 실행 중이거나 종료된 컨테이너 로그 출력 |