---
layout: post
title: Docker - Macvlan, Overlay, Compose
date: 2025-01-28 19:20:23 +0900
category: Docker
---
# Docker 고급 네트워킹: Macvlan과 Overlay 네트워크 이해

## 개요

Docker는 다양한 네트워킹 요구사항을 충족하기 위해 여러 네트워크 드라이버를 제공합니다. 이 가이드에서는 두 가지 고급 네트워크 드라이버인 Macvlan과 Overlay를 중심으로, 이들의 작동 원리와 실제 적용 방법을 상세히 설명합니다.

## Docker 네트워크 드라이버 비교

| 드라이버 | 주 사용 목적 | 주요 특징 |
|----------|-------------|-----------|
| **bridge** | 단일 호스트 내 가상 네트워크 | 기본 네트워크 드라이버, 포트 매핑 필요 |
| **host** | 최대 성능 요구 환경 | 호스트 네트워크 스택 공유, 포트 충돌 주의 |
| **macvlan** | 컨테이너를 실제 네트워크 장치처럼 노출 | 고유 MAC 주소 할당, 포트 매핑 불필요 |
| **overlay** | 다중 호스트 클러스터 네트워킹 | Swarm 모드 필요, VXLAN 기반 |
| **none** | 네트워크 완전 격리 | 외부 네트워크 접근 차단 |

---

## Macvlan: 컨테이너를 실제 네트워크 장치로

### Macvlan의 개념

Macvlan은 컨테이너에 고유한 MAC 주소를 할당하여 물리 네트워크에 직접 연결하는 네트워크 드라이버입니다. 이 방식을 사용하면 컨테이너가 LAN 상의 독립적인 네트워크 장치처럼 동작하게 되어, 프린터, IoT 장치, NAS와 같이 실제 IP 주소를 필요로 하는 서비스에 적합합니다.

### Macvlan 동작 모드

Linux의 Macvlan은 여러 모드를 지원하지만, Docker 환경에서는 주로 **bridge 모드**를 사용합니다:
- **bridge 모드**: 가장 일반적인 모드로, parent NIC 위에서 여러 Macvlan 인터페이스를 브리지처럼 운용합니다.
- 기타 모드(private/vepa/passthru): 특수한 네트워킹 환경에서 사용됩니다.

### Macvlan 네트워크 생성 및 사용

```bash
# Macvlan 네트워크 생성
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  my-macvlan

# 컨테이너를 Macvlan 네트워크에 연결 (고정 IP 할당)
docker run -d --name printer \
  --network my-macvlan \
  --ip 192.168.1.50 \
  nginx:alpine
```

이제 컨테이너는 LAN 상에서 `192.168.1.50` 주소로 직접 접근 가능합니다. 포트 매핑이 필요하지 않습니다.

### 호스트와 컨테이너 간 통신 문제 해결

Macvlan의 주요 제약점은 동일한 NIC을 사용하는 호스트와 컨테이너 간의 직접 통신이 기본적으로 불가능하다는 점입니다. 이 문제를 해결하는 두 가지 일반적인 방법이 있습니다:

#### 방법 A: 호스트에 별도 Macvlan 인터페이스 생성
```bash
# 호스트에 Macvlan 인터페이스 생성
sudo ip link add macvlan0 link eth0 type macvlan mode bridge
sudo ip addr add 192.168.1.240/24 dev macvlan0
sudo ip link set macvlan0 up

# 이제 호스트에서 컨테이너로 통신 가능
ping -c1 192.168.1.50
```

#### 방법 B: VLAN 서브인터페이스 사용
물리 NIC에 VLAN 태깅을 적용하고, 컨테이너와 호스트를 서로 다른 VLAN에 배치하여 라우터를 통한 통신을 구성합니다.

### Docker Compose에서 Macvlan 사용

Macvlan 네트워크는 일반적으로 Compose 파일 외부에서 미리 생성한 후 참조하는 방식이 권장됩니다:

```yaml
version: "3.9"
services:
  cam:
    image: nginx:alpine
    networks:
      macvlan_lan:
        ipv4_address: 192.168.1.60

networks:
  macvlan_lan:
    external: true
```

네트워크 생성:
```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 --gateway=192.168.1.1 \
  -o parent=eth0 macvlan_lan
```

### Macvlan 운영 시 고려사항

1. **IP 주소 관리**: DHCP 범위와 충돌하지 않도록 정적 IP 할당을 권장합니다.
2. **보안**: 컨테이너가 실제 네트워크 장치처럼 보이므로 방화벽 및 ACL 정책을 적용해야 합니다.
3. **플랫폼 지원**: Macvlan은 Linux 호스트에서 완전히 지원되며, Docker Desktop 환경에서는 제한적입니다.

---

## Overlay: 다중 호스트 가상 네트워크

### Overlay 네트워크 개념

Overlay 네트워크는 여러 Docker 호스트에 걸쳐 컨테이너들이 마치 동일한 네트워크에 있는 것처럼 통신할 수 있게 하는 VXLAN 기반 가상 네트워크입니다. Docker Swarm 모드에서 활성화되며, 데이터센터나 클라우드 환경에서 분산 애플리케이션을 구축하는 데 필수적입니다.

### Overlay 네트워크 기본 사용법

```bash
# Swarm 모드 초기화 (첫 번째 노드에서)
docker swarm init

# Attachable 옵션으로 Overlay 네트워크 생성
docker network create -d overlay --attachable my-overlay

# Swarm 서비스 생성
docker service create --name web --network my-overlay nginx:alpine

# 일반 컨테이너도 네트워크에 연결 가능
docker run -d --name debug --network my-overlay nicolaka/netshoot
```

### 서비스 포트 공개

Overlay 네트워크에서는 서비스 포트를 여러 방식으로 공개할 수 있습니다:

```bash
# Ingress 모드 (라우팅 메시 - 모든 노드에서 접근 가능)
docker service create \
  --name api \
  --network my-overlay \
  --publish published=8080,target=80,mode=ingress \
  nginx:alpine

# Host 모드 (특정 노드에서만 접근 가능)
docker service create \
  --name api \
  --network my-overlay \
  --publish published=8080,target=80,mode=host \
  nginx:alpine
```

### 네트워크 암호화

Overlay 네트워크는 노드 간 통신을 암호화할 수 있습니다:

```bash
# 암호화된 Overlay 네트워크 생성
docker network create -d overlay --attachable --opt encrypted secure-ovl
```

암호화는 보안을 강화하지만 CPU 오버헤드를 발생시킬 수 있으므로 성능 요구사항을 고려해야 합니다.

### MTU 고려사항

Overlay 네트워크는 VXLAN 캡슐화를 사용하므로 약 50바이트의 오버헤드가 발생합니다. 유효 MTU는 다음과 같이 계산할 수 있습니다:

```
유효 MTU ≈ 물리망 MTU - 50
```

예를 들어 물리망 MTU가 1500인 경우, Overlay 네트워크의 유효 MTU는 약 1450입니다. 대용량 패킷을 전송하는 애플리케이션의 경우 MTU 조정이 필요할 수 있습니다.

### Docker Stack을 이용한 배포

Docker Stack은 Compose 파일 형식을 사용하여 Swarm 서비스를 배포하는 방법입니다:

```yaml
# stack-ovl.yml
version: "3.9"
networks:
  appnet:
    driver: overlay
    attachable: true

services:
  db:
    image: postgres:15
    networks: [appnet]
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure

  api:
    image: myorg/api:1.0
    networks: [appnet]
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure

  fe:
    image: nginx:alpine
    networks: [appnet]
    ports:
      - "8080:80"
    deploy:
      replicas: 2
```

배포 명령:
```bash
docker stack deploy -c stack-ovl.yml appstack
docker stack services appstack
```

---

## Docker Compose에서의 네트워크 구성

### 다중 네트워크를 이용한 아키텍처 분리

```yaml
version: "3.9"
services:
  fe:
    image: nginx:alpine
    ports: ["8080:80"]
    networks: [frontnet, backnet]
    
  api:
    build: ./api
    networks: [backnet]
    
  db:
    image: postgres:15
    networks: [backnet]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres || exit 1"]
      interval: 5s
      retries: 12

networks:
  frontnet: {}
  backnet: {}
```

이 구성에서는 프론트엔드 서비스만 외부에 노출되며, 백엔드 서비스는 내부 네트워크에서만 통신합니다.

### 외부 네트워크 참조

```yaml
networks:
  corp_lan:
    external: true
```

이 방식으로 이미 존재하는 네트워크(Docker 또는 물리 네트워크)를 Compose 서비스에서 사용할 수 있습니다.

### 네트워크 드라이버 옵션 설정

```yaml
networks:
  isolated:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.enable_icc: "false"   # 컨테이너 간 통신 차단
    ipam:
      driver: default
      config:
        - subnet: 172.30.0.0/16
          gateway: 172.30.0.1
```

---

## 보안 및 운영 모범 사례

### 보안 강화

1. **네트워크 분리**: 프론트엔드, 백엔드, 관리 네트워크를 분리하여 공격 표면을 최소화합니다.
2. **최소 권한 원칙**: 컨테이너는 필요한 최소한의 네트워크 권한만 가져야 합니다.
3. **암호화**: 민감한 데이터를 전송하는 Overlay 네트워크는 암호화를 고려하세요.
4. **모니터링**: 네트워크 트래픽을 지속적으로 모니터링하여 이상 활동을 감지하세요.

### 성능 최적화

1. **네트워크 드라이버 선택**: 성능 요구사항에 맞는 드라이버를 선택하세요 (host > macvlan/bridge > overlay).
2. **MTU 조정**: 대용량 패킷을 사용하는 애플리케이션의 경우 적절한 MTU 설정이 중요합니다.
3. **포트 공개 모드 선택**: 성능이 중요한 경우 host 모드 포트 공개를 고려하세요.

### 운영 관찰성

1. **네트워크 진단 도구**: `nicolaka/netshoot` 같은 네트워크 진단 컨테이너를 활용하세요.
2. **모니터링 도구**: 네트워크 트래픽과 성능 메트릭을 수집하는 모니터링 시스템을 구축하세요.
3. **로그 관리**: 네트워크 관련 로그를 중앙에서 관리하고 분석하세요.

---

## 문제 해결 가이드

### 일반적인 문제 및 해결 방법

| 문제 증상 | 가능한 원인 | 해결 방법 |
|-----------|-------------|-----------|
| Macvlan 컨테이너와 호스트 간 통신 불가 | 커널 정책 제한 | 호스트에 별도 Macvlan 인터페이스 생성 또는 VLAN 분리 |
| Overlay 네트워크 성능 저하 | MTU 단편화 | 유효 MTU 계산 및 조정, 애플리케이션 MTU 설정 |
| 서비스 포트 접근 불가 | 라우팅 메시 구성 오류 | 포트 공개 모드 확인 및 조정 |
| DNS 이름 해결 실패 | 네트워크 구성 오류 | 네트워크 연결 상태 확인, DNS 설정 검증 |

### 진단 명령어 모음

```bash
# 네트워크 상태 확인
docker network ls
docker network inspect <네트워크명>

# 컨테이너 네트워크 정보 확인
docker exec <컨테이너> ip addr
docker exec <컨테이너> cat /etc/resolv.conf

# 네트워크 트래픽 분석
sudo tcpdump -ni <인터페이스> host <IP주소>

# DNS 해결 테스트
docker exec <컨테이너> nslookup <서비스명>
```

---

## 실전 예제

### Macvlan을 사용한 실제 장치 에뮬레이션

```yaml
# docker-compose-macvlan.yml
version: "3.9"
services:
  iot-device:
    image: custom-iot-image:latest
    networks:
      lan:
        ipv4_address: 192.168.1.100

networks:
  lan:
    external: true
    name: macvlan_lan
```

### Overlay 네트워크를 이용한 분산 애플리케이션

```yaml
# docker-stack-overlay.yml
version: "3.9"
networks:
  app_network:
    driver: overlay
    attachable: true

services:
  database:
    image: postgres:15
    networks:
      - app_network
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]

  application:
    image: myapp:latest
    networks:
      - app_network
    deploy:
      replicas: 3
    depends_on:
      - database

  loadbalancer:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - app_network
    deploy:
      replicas: 2
```

---

## 네트워크 토폴로지 예시

```
[LAN 192.168.1.0/24]
   |
   +---[Host eth0]------------------------------+
   |                                            |
   | Macvlan 네트워크                           | Overlay 네트워크
   |   +-- 컨테이너 A (192.168.1.50)            |   +-- 노드1
   |   +-- 컨테이너 B (192.168.1.51)            |   |   +-- 서비스 복제본 A
   |                                            |   +-- 노드2
   | 호스트 Macvlan 인터페이스 (192.168.1.240)  |   |   +-- 서비스 복제본 B
   |                                            |   +-- 노드3
   |                                            |       +-- 서비스 복제본 C
   +--------------------------------------------+
```

---

## 주요 명령어 요약

```bash
# Macvlan 네트워크 생성
docker network create -d macvlan \
  --subnet=192.168.1.0/24 --gateway=192.168.1.1 \
  -o parent=eth0 macvlan_lan

# Overlay 네트워크 생성
docker swarm init
docker network create -d overlay --attachable appnet

# 네트워크 정보 확인
docker network ls
docker network inspect <네트워크명>

# 컨테이너 네트워크 연결 관리
docker network connect <네트워크명> <컨테이너명>
docker network disconnect <네트워크명> <컨테이너명>
```

---

## 결론

Docker의 Macvlan과 Overlay 네트워크 드라이버는 각각 특정한 네트워킹 요구사항을 해결하는 강력한 도구입니다.

**Macvlan**은 컨테이너를 실제 네트워크 장치처럼 노출해야 할 때 이상적인 선택입니다. IoT 장치, 프린터, NAS 등 실제 IP 주소가 필요한 서비스를 컨테이너화할 때 특히 유용합니다. 다만 호스트와의 통신 제한을 이해하고 적절히 해결하는 것이 중요합니다.

**Overlay** 네트워크는 다중 호스트 환경에서 분산 애플리케이션을 구축하는 데 필수적입니다. Docker Swarm과 결합하여 데이터센터나 클라우드 전반에 걸친 서비스 네트워킹을 가능하게 합니다. VXLAN 기반의 아키텍처는 확장성과 유연성을 제공하지만, MTU 및 성능 고려사항을 이해해야 합니다.

이 두 네트워크 드라이버는 상호 배타적이지 않으며, 실제 환경에서는 필요에 따라 혼용될 수 있습니다. 예를 들어, 단일 호스트의 실제 장치 에뮬레이션에는 Macvlan을, 클러스터 환경의 마이크로서비스 아키텍처에는 Overlay를 사용할 수 있습니다.

효과적인 Docker 네트워킹 설계의 핵심은 애플리케이션 요구사항을 명확히 이해하고, 각 네트워크 드라이버의 장단점을 고려하여 적절한 아키텍처를 선택하는 데 있습니다. 보안, 성능, 관리 용이성의 균형을 유지하면서 비즈니스 요구를 충족하는 네트워크 설계가 성공적인 컨테이너 환경의 기초가 됩니다.