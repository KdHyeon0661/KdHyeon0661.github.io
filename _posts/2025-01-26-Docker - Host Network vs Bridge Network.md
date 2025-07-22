---
layout: post
title: Docker - Host Network vs Bridge Network
date: 2025-01-26 20:20:23 +0900
category: Docker
---
# 🌐 Docker 네트워크 비교: Host Network vs Bridge Network

---

## 🧭 1. Docker의 기본 네트워크 이해

Docker는 기본적으로 컨테이너 간 통신을 위해 여러 네트워크 드라이버를 제공합니다:

- `bridge`: 기본값, 가상 네트워크를 구성해 컨테이너 격리
- `host`: 컨테이너가 호스트의 네트워크를 **공유**
- `none`: 네트워크 없음
- `overlay`: 다중 호스트 클러스터 (Swarm, Kubernetes)
- `macvlan`: MAC 주소 수준의 네트워크 분리

---

## 🔍 2. Bridge Network란?

> Docker의 **기본 네트워크 모드**로, 가상 네트워크를 생성하고  
> 컨테이너들이 내부 IP로 통신합니다.

### ✅ 특징
- 각 컨테이너는 **고유한 가상 IP 주소**를 가짐
- 외부와 통신하려면 **포트 매핑 필요 (`-p`)**
- 격리성이 높고, 컨테이너 간 통신도 Docker가 관리
- Docker 데몬이 `iptables`로 NAT, 포트포워딩 설정함

### ✅ 예시

```bash
docker run -d -p 8080:80 nginx
```

- 외부 `8080` → 내부 컨테이너의 `80`포트로 전달
- 네트워크 이름: `bridge` (기본값)

---

## 🌐 3. Host Network란?

> 컨테이너가 **호스트의 네트워크 네임스페이스를 그대로 사용**  
> 즉, 컨테이너가 자신의 IP 없이 **호스트와 동일한 IP/포트 공간** 사용

### ✅ 특징
- **컨테이너가 곧 호스트**처럼 동작
- 포트 매핑이 **불필요**
- 네트워크 오버헤드 최소화 → **성능 우수**
- 그러나 **격리성이 낮고, 포트 충돌 가능성 있음**

### ✅ 예시

```bash
docker run --network host nginx
```

- 외부에서 `localhost:80` 접속 → 곧바로 컨테이너의 80 포트
- `-p` 불필요, 무시됨

---

## ⚙️ 4. 차이점 비교

| 항목 | **Bridge Network** | **Host Network** |
|------|---------------------|------------------|
| IP | 컨테이너 별 내부 IP | 호스트 IP 그대로 사용 |
| 포트 매핑 | 필요 (`-p`) | 불필요 (무시됨) |
| 격리성 | 높음 (가상 네트워크) | 낮음 (호스트와 공유) |
| 성능 | NAT/포워딩 경로 | 직접 연결 → 빠름 |
| 보안 | 비교적 안전 | 주의 필요 (같은 IP/포트) |
| 컨테이너 간 통신 | 동일 네트워크 시 이름으로 접근 | 일반적으로 직접 접근 불가 |
| 포트 충돌 | 없음 (포트 할당 분리됨) | 있음 (호스트 포트 겹치면 에러) |

---

## 📁 5. 실전 예제 비교

### 🟦 Bridge 방식 (격리된 네트워크)

```bash
docker run -d --name web1 -p 8080:80 nginx
```

- 컨테이너는 `172.x.x.x` 같은 내부 IP 가짐
- 외부는 `localhost:8080`으로 접속

### 🟥 Host 방식 (직접 연결)

```bash
docker run -d --name web2 --network host nginx
```

- 컨테이너에서 80 포트를 열면 → 외부에서 `localhost:80`으로 바로 접속
- **다른 컨테이너에서 80 포트를 동시에 사용 불가!**

---

## 📉 6. 언제 사용해야 할까?

| 상황 | 추천 네트워크 |
|------|----------------|
| 일반 웹 서비스, DB 등 | `bridge` |
| 고속 패킷 처리, 프록시 서버, 성능 민감한 서비스 | `host` |
| 여러 컨테이너가 다른 포트로 운영 | `bridge` (포트 충돌 방지) |
| 네트워크 성능이 중요하고, 포트 충돌 통제 가능 | `host` 가능 |

---

## 📋 7. 주의 사항

### ❗ host network 사용 시

- 포트 충돌 방지 필요 (컨테이너 간 동일 포트 사용 금지)
- 보안 경계 모호 → 권한 제한 필수 (`--cap-drop`, `read-only fs` 등)
- 로그/트래픽 추적이 어려울 수 있음

### ❗ bridge network 사용 시

- 포트 노출 실수 방지 (`-p` 없이 실행하면 외부에서 접근 불가)
- 컨테이너 이름으로 통신하려면 동일 네트워크에 있어야 함

---

## 🧪 8. 실습 테스트

### Bridge에서 접속 확인

```bash
docker run -d --name web1 -p 8080:80 nginx
curl localhost:8080  # OK
```

### Host에서 접속 확인

```bash
docker run -d --name web2 --network host nginx
curl localhost:80    # OK
```

### 충돌 테스트

```bash
docker run -d --network host -p 8080:80 nginx  # ❌ -p 무시됨, 중복 사용 시 에러
```

---

## 📚 참고 자료

- [Docker Networking 공식 문서](https://docs.docker.com/network/)
- [Host vs Bridge 설명](https://docs.docker.com/network/host/)
- [Docker Port Mapping](https://docs.docker.com/config/containers/container-networking/)