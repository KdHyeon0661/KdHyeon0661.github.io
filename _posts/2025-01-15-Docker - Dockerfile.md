---
layout: post
title: Docker - exec vs attach
date: 2025-01-14 19:20:23 +0900
category: Docker
---
# 🧾 Dockerfile 완전 정복: 기본 문법과 개념 설명

---

## 📌 Dockerfile이란?

**Dockerfile**은 Docker 이미지를 만들기 위한 **스크립트 파일**입니다.  
`이미지를 어떻게 구성할지`를 선언적으로 작성하며,  
이미지 빌드시 `위에서 아래로 순서대로` 실행됩니다.

### ✅ 특징
- **반복 가능성 보장**: 항상 같은 방식으로 이미지를 생성
- **버전 관리 용이**: Git 등으로 기록 가능
- **자동화에 적합**: CI/CD 파이프라인에서 사용

---

## 📁 기본 파일 구조 예시

```
my-app/
├── Dockerfile
├── app.py
└── requirements.txt
```

---

## 🧱 Dockerfile 기본 문법 설명

| 키워드 | 설명 |
|--------|------|
| `FROM` | **기반 이미지 지정 (필수)** |
| `RUN` | 셸 명령어 실행 (이미지 빌드시 수행됨) |
| `COPY` | 파일 또는 디렉토리를 이미지에 복사 |
| `ADD` | `COPY` + 압축 해제 + URL 지원 |
| `CMD` | 컨테이너 시작 시 실행할 **기본 명령어** |
| `ENTRYPOINT` | CMD와 유사하지만 **고정된 진입점** |
| `WORKDIR` | 작업 디렉토리 설정 |
| `ENV` | 환경 변수 설정 |
| `EXPOSE` | 문서용 포트 공개 (명시적 의미, -p 필수) |
| `VOLUME` | 마운트 지점 지정 |
| `LABEL` | 메타데이터 추가 (작성자 등) |

---

## 🔹 FROM

```dockerfile
FROM ubuntu:20.04
```

- 베이스 이미지를 지정합니다. **가장 먼저 나와야 합니다.**
- 대부분의 이미지는 Docker Hub에 존재합니다.
- 여러 단계 빌드를 위해 **멀티 스테이지 빌드**에서도 사용됨

---

## 🔹 RUN

```dockerfile
RUN apt-get update && apt-get install -y nginx
```

- 컨테이너 안에서 셸 명령어를 실행해 **새로운 레이어**를 생성합니다.
- 일반적으로 패키지 설치, 디렉토리 생성 등에 사용합니다.
- 가능한 한 `RUN` 명령은 **합쳐서 작성**해야 이미지가 최적화됩니다.

> 예시 (좋은 방식):
```dockerfile
RUN apt-get update && apt-get install -y curl git
```

---

## 🔹 COPY

```dockerfile
COPY ./app.py /app/app.py
```

- 로컬 파일을 이미지 내 파일 시스템으로 **복사**합니다.
- 기본적으로 상대경로를 기준으로 복사합니다.

> COPY는 단순 복사만 가능하며, 압축 해제/URL 다운로드는 불가

---

## 🔹 ADD

```dockerfile
ADD https://example.com/app.tar.gz /app/
```

- `COPY`와 비슷하지만:
  - URL로부터 다운로드 가능
  - 압축 파일이면 자동으로 압축 해제

> 일반적으로는 예측 가능한 `COPY`를 사용하는 것이 더 안전합니다.

---

## 🔹 CMD

```dockerfile
CMD ["python", "app.py"]
```

- 컨테이너가 시작될 때 **기본 실행 명령어**를 지정합니다.
- `Dockerfile` 안에 **하나만 존재 가능**합니다.
- `docker run` 시 명령어가 주어지면 **CMD는 무시됨**

> `ENTRYPOINT`와 함께 사용하면 **인자(Arguments)** 역할을 할 수 있습니다.

---

## 🔹 ENTRYPOINT

```dockerfile
ENTRYPOINT ["python", "app.py"]
```

- 컨테이너의 **고정된 실행 지점**을 지정합니다.
- `docker run`의 인자(`args`)가 ENTRYPOINT 뒤에 붙습니다.

> CMD보다 더 "강제력 있는 실행 지점"입니다.

예시:
```dockerfile
ENTRYPOINT ["curl"]
CMD ["--help"]
```
→ 실행 결과: `curl --help`

---

## 🔹 WORKDIR

```dockerfile
WORKDIR /app
```

- 이후 명령어(`RUN`, `CMD`, `COPY`, `ADD` 등)가 실행될 **기본 디렉토리**를 지정합니다.
- 중첩 사용 가능하며, 상대 경로를 사용할 수 있습니다.

---

## 🔹 ENV

```dockerfile
ENV DB_HOST=mysql
```

- 환경 변수를 설정하여 실행 시 접근할 수 있습니다.
- `RUN`, `CMD`, `ENTRYPOINT`에서도 사용 가능

> 예:
```dockerfile
RUN echo $DB_HOST
```

---

## 🔹 EXPOSE

```dockerfile
EXPOSE 80
```

- 컨테이너가 사용하는 **포트를 문서상으로 표시**합니다.
- 실제로 외부와 연결하려면 `-p` 옵션이 필요합니다.

```bash
docker run -p 8080:80 image_name
```

---

## 🔹 VOLUME

```dockerfile
VOLUME /data
```

- **외부 데이터 저장 지점**을 지정합니다.
- 실제로는 `docker run -v`로 볼륨을 연결해야 효과가 있습니다.

---

## 🔹 LABEL

```dockerfile
LABEL maintainer="Do Hyun Kim"
LABEL version="1.0"
```

- 이미지에 메타데이터를 부여합니다.
- `docker inspect`로 확인 가능

---

## 🛠 Dockerfile 예제: Python 앱

```Dockerfile
# 베이스 이미지 설정
FROM python:3.11-slim

# 환경변수 설정
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# 작업 디렉토리 생성
WORKDIR /app

# 의존성 복사 및 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 앱 파일 복사
COPY . .

# 포트 명시
EXPOSE 5000

# 기본 실행 명령
CMD ["python", "app.py"]
```

---

## ✅ Dockerfile 빌드 & 실행

```bash
# Dockerfile을 기반으로 이미지 생성
docker build -t my-python-app .

# 이미지 실행
docker run -p 5000:5000 my-python-app
```

---

## 📌 Dockerfile 작성 팁

| 팁 | 설명 |
|-----|------|
| 불필요한 RUN 줄이기 | 여러 명령을 하나의 RUN으로 |
| .dockerignore 활용 | 빌드에 불필요한 파일 제외 |
| 작은 이미지 사용 | `alpine`, `slim` 등 |
| 멀티 스테이지 빌드 | 빌드 단계와 실행 단계 분리 |

---

## 📋 요약 명령어 정리

| 명령어 | 설명 |
|--------|------|
| `FROM` | 베이스 이미지 설정 |
| `RUN` | 빌드시 명령 실행 |
| `COPY` | 파일 복사 |
| `CMD` | 컨테이너 시작 명령 |
| `ENTRYPOINT` | 컨테이너 실행 진입점 |
| `WORKDIR` | 기본 디렉토리 설정 |
| `ENV` | 환경 변수 설정 |
| `EXPOSE` | 포트 문서화 |
| `VOLUME` | 외부 마운트 지점 |
| `LABEL` | 메타데이터 |