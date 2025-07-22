---
layout: post
title: Docker - Docker 이미지
date: 2025-01-11 20:20:23 +0900
category: Docker
---
# 📦 Docker 이미지란? 계층 구조 이해하기

---

## 🐳 Docker 이미지(Image)란?

Docker 이미지란 **컨테이너를 실행하기 위한 불변의 실행 환경(템플릿)**입니다.

### 📌 주요 특징
- **불변(immutable)**: 이미지 자체는 변경되지 않음
- **계층적 구조**: 이미지가 명령어 하나당 하나의 레이어로 구성됨
- **읽기 전용(read-only)**: 컨테이너 실행 시에만 쓰기 가능한 레이어가 추가됨
- **재사용 가능**: 공통 레이어는 여러 이미지가 공유하여 저장 공간을 절약

---

## 🧱 계층 구조(Layered Structure)

Docker 이미지는 **여러 개의 레이어로 쌓인 구조**입니다.  
이 레이어는 **Dockerfile의 각 명령어(Instruction)** 하나하나가 만들어냅니다.

### 🔹 예시: Dockerfile

```Dockerfile
FROM ubuntu:20.04          # [Layer 1]
RUN apt-get update         # [Layer 2]
RUN apt-get install -y nginx  # [Layer 3]
COPY . /app                # [Layer 4]
```

### 📐 결과 구조

```
Layer 1: Ubuntu base image (ubuntu:20.04)
Layer 2: RUN apt-get update
Layer 3: RUN apt-get install -y nginx
Layer 4: COPY . /app
```

- 각 Layer는 읽기 전용(Read-only)입니다.
- 컨테이너가 실행되면 최상단에 쓰기 가능한 **컨테이너 레이어(Write Layer)**가 추가됩니다.

---

## 🔄 이미지의 계층 구조 동작 방식

1. 이미지를 실행하면 컨테이너가 생성됨
2. 컨테이너는 이미지의 모든 레이어를 읽기 전용으로 가져옴
3. 최상단에 쓰기 가능한 레이어가 추가됨 (컨테이너만의 임시 공간)

```
+---------------------------+
| Writable Container Layer |
+---------------------------+
| COPY . /app              | ← Layer 4
+---------------------------+
| RUN install nginx        | ← Layer 3
+---------------------------+
| RUN apt-get update       | ← Layer 2
+---------------------------+
| FROM ubuntu:20.04        | ← Layer 1
+---------------------------+
```

---

## 🔁 레이어의 재사용 & 캐싱

Docker는 빌드시 **기존 레이어를 캐시로 저장**하고,  
**Dockerfile의 명령어가 동일하면 캐시를 재사용**합니다.

> 즉, 위에서 변경이 없는 한 아래 명령은 다시 실행하지 않습니다.

예를 들어:
```Dockerfile
RUN apt-get update        # 변경 없음 → 캐시 사용
RUN apt-get install ...   # 변경 있음 → 이후 레이어부터 다시 빌드
```

---

## 💡 레이어 구조의 장점

| 장점 | 설명 |
|------|------|
| 저장 공간 절약 | 중복 레이어는 한 번만 저장 |
| 빠른 빌드 | 변경된 레이어만 재빌드 (캐싱 덕분) |
| 쉬운 버전 관리 | 계층별로 이미지 관리 가능 |
| 배포 효율 | 변경된 레이어만 네트워크 전송 |

---

## 🧪 이미지 확인 명령어

### 🔍 이미지 목록 보기
```bash
docker images
```

### 🔍 이미지 상세 보기 (계층 확인)
```bash
docker history [이미지이름]
```

예시:
```bash
docker history ubuntu:20.04
```

→ 출력 결과:
```
IMAGE          CREATED       CREATED BY               SIZE
f643c72bc252   3 weeks ago   /bin/sh -c #(nop) CMD    0B
...
```

---

## ✅ 정리

| 개념 | 설명 |
|------|------|
| Docker 이미지 | 컨테이너의 실행을 위한 읽기 전용 실행 환경 |
| 레이어 | Dockerfile 명령어 하나하나가 만드는 계층 |
| 캐시 | 기존 레이어를 재사용해 빠르게 빌드 |
| 저장 최적화 | 공통 레이어는 여러 이미지에서 공유 가능 |

---

## 📌 요약

> Docker 이미지는 **파일 시스템 레이어들의 조합**이며,  
> Dockerfile의 명령어 하나하나가 레이어가 되어  
> **효율적인 빌드, 배포, 저장**을 가능하게 합니다.