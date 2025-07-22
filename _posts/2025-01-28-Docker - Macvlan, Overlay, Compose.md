---
layout: post
title: Docker - Macvlan, Overlay, Compose
date: 2025-01-14 19:20:23 +0900
category: Docker
---
# 🧠 Docker 고급 네트워크 정리: Macvlan, Overlay, Compose 설정

---

## 📌 목차

1. 🔌 macvlan: 물리 네트워크처럼 동작
2. 🌍 overlay: 여러 호스트 간 통신 (Swarm 기반)
3. 🧱 docker-compose에서 네트워크 직접 설정하기

---

## 1️⃣ macvlan 네트워크: 컨테이너에 MAC 주소를 부여한다?

> 컨테이너가 **호스트의 네트워크 인터페이스를 분할하여**  
> 물리 네트워크 상의 **고유한 MAC/IP 주소**를 가진 독립 장비처럼 동작

### ✅ 특징

| 항목 | 설명 |
|------|------|
| IP / MAC | 물리 네트워크의 진짜 IP처럼 동작 |
| 목적 | IoT, 프린터, 로컬 장비처럼 인식되어야 할 때 |
| 제한 | 호스트와 컨테이너 간 통신은 **불가능** (같은 NIC를 사용할 경우) |
| 외부 장비 통신 | 가능 (라우터, 스위치와 직접 통신) |

### ✅ 구성 예시

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  my-macvlan
```

- `parent=eth0`: 물리 NIC를 기반으로 생성
- `192.168.1.X` 대역의 **고유 IP를 컨테이너가 직접 부여받음**

### ✅ 컨테이너 실행 예

```bash
docker run -d --name printer \
  --network my-macvlan \
  --ip 192.168.1.50 \
  nginx
```

→ 외부에서 `192.168.1.50:80`으로 직접 접근 가능 (포트 포워딩 불필요)

---

## ❗ macvlan 주의 사항

- 호스트와 동일 NIC를 사용하면, **컨테이너와 호스트 간 통신 안 됨**
  - 해결: **bridge + veth pair** 혹은 **macvlan bridge 모드** 필요
- 라우터 DHCP 설정과 충돌하지 않도록 정적 IP 사용 권장
- Linux에서만 완전 지원됨 (Mac/Windows는 Docker Desktop에서 제한)

---

## 2️⃣ overlay 네트워크: 다중 Docker 호스트 통신

> 여러 Docker **노드(서버)** 간에 가상 네트워크를 구성할 때 사용하는 방식  
> Docker Swarm 모드 기반으로 동작

### ✅ 특징

| 항목 | 설명 |
|------|------|
| 네트워크 범위 | 여러 물리적 호스트 (노드)에 걸쳐 있음 |
| 컨테이너 통신 | 같은 overlay 네트워크에 있으면 서로 통신 가능 |
| 암호화 | VXLAN 기반으로 **데이터 암호화 가능** |
| 필요 조건 | Swarm 모드 활성화 (`docker swarm init`) |

---

### ✅ 구성 흐름

```bash
# 1. Swarm 초기화
docker swarm init

# 2. Overlay 네트워크 생성
docker network create \
  -d overlay \
  --attachable \
  my-overlay
```

- `--attachable`: 일반 컨테이너도 네트워크에 붙일 수 있음

### ✅ 컨테이너 실행

```bash
docker service create \
  --name web \
  --network my-overlay \
  nginx
```

또는 Swarm이 아닌 일반 컨테이너도 붙일 수 있음:

```bash
docker run -d --network my-overlay nginx
```

> 여러 호스트에 설치된 Docker들이 이 overlay 네트워크를 공유함

---

## 3️⃣ Docker Compose의 네트워크 설정

### ✅ 기본 동작

Compose는 자동으로 네트워크를 만들어 서비스끼리 통신 가능하게 합니다.

```yaml
services:
  web:
    image: nginx
  db:
    image: mysql
```

→ `web`은 `db:3306`으로 접근 가능  
→ 자동 생성된 네트워크 이름은 `디렉토리명_default`

---

### ✅ 사용자 정의 네트워크 사용 예

```yaml
version: '3.9'
services:
  web:
    image: nginx
    networks:
      - frontend
  db:
    image: mysql
    networks:
      - backend

networks:
  frontend:
  backend:
```

- `web`과 `db`는 서로 **통신 불가** (네트워크가 다름)
- 추가로 통신하게 하려면 두 네트워크를 모두 연결해야 함

```yaml
  web:
    networks:
      - frontend
      - backend
```

---

### ✅ external 네트워크 (외부에서 만든 네트워크 사용)

```yaml
networks:
  mynet:
    external: true
```

> `docker network create mynet`으로 먼저 생성 필요

---

### ✅ 네트워크 드라이버 지정

```yaml
networks:
  frontend:
    driver: bridge
  overlaynet:
    driver: overlay
```

---

## 📋 정리 요약

| 네트워크 | 용도 | 특징 |
|----------|------|------|
| **bridge** | 기본값 | 격리, 포트 매핑 필요 |
| **host** | 성능 중시 | 호스트 IP 직접 사용, 격리 없음 |
| **macvlan** | 외부 장비처럼 동작 | 컨테이너에 실제 IP 부여, 호스트와 통신 불가 |
| **overlay** | 다중 노드 구성 | Swarm 기반, 암호화 가능, 클러스터 환경 필수 |
| **docker-compose** | 멀티 컨테이너 구성 | 이름 기반 통신, 네트워크 명시 가능 |

---

## 📚 참고 자료

- [Docker macvlan docs](https://docs.docker.com/network/macvlan/)
- [Docker overlay network docs](https://docs.docker.com/network/overlay/)
- [Docker Compose networks](https://docs.docker.com/compose/networking/)