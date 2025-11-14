---
layout: post
title: Docker - Macvlan, Overlay, Compose
date: 2025-01-28 19:20:23 +0900
category: Docker
---
# Docker 고급 네트워크 정리 : Macvlan · Overlay · Compose 설정

## 목차

1. 개요 — Docker 네트워크 드라이버 지도
2. **Macvlan**: 물리망처럼 보이게 — 모드, 설계, 호스트↔컨테이너 통신 해결, Compose 예제
3. **Overlay**: 다중 호스트 가상망 — Swarm, 암호화, 서비스/컨테이너 붙이기, 포트 공개, Compose/Stack 예제
4. **Compose에서 네트워크 직접 설정** — 다중 네트워크, 외부 네트워크, 드라이버 옵션, 헬스체크 연계
5. 보안/성능/운영 베스트 프랙티스 — 권한/분리, MTU, 관측/로깅, 네임해결
6. 트러블슈팅 체크리스트 — ARP/DNS/라우팅/포트 충돌, 재현 가능한 진단 명령
7. 부록 — 네트워크 토폴로지 예시, 용어 정리, 치트시트

---

## 개요 — Docker 네트워크 드라이버 지도

| 드라이버     | 주 용도                              | 핵심 특징 |
|--------------|--------------------------------------|-----------|
| **bridge**   | 단일 호스트 내 격리된 가상망         | 포트 매핑 필요, 기본값 |
| **host**     | 네임스페이스 공유(고성능/저격리)     | 포트 매핑 불가, 충돌 유의 |
| **macvlan**  | 컨테이너에 **실제 MAC/IP** 부여      | 외부 장비처럼 취급, 호스트와 직접 통신 제한 |
| **overlay**  | **다중 호스트 간** 가상망            | Swarm 필요, VXLAN, 암호화 가능 |
| **none**     | 네트워크 미사용                      | 완전 격리 |

이 글의 초점은 **macvlan**과 **overlay**입니다. 두 드라이버는 **“물리망과의 통합(혹은 데이터센터/클라우드 전체에 걸친 가상망)”** 이라는 과제를 해결합니다.

---

## Macvlan — 컨테이너가 진짜 장비처럼 보이게

> 컨테이너마다 **고유 MAC** 을 부여하고, 물리 네트워크(L2)에 직접 붙이는 방식.
> 프린터/IoT/NAS처럼 **LAN 상의 실체 IP** 로 보여야 할 때 매우 적합.

### 동작 모드 개요

Linux macvlan은 대표적으로 다음 모드를 가집니다.

- **bridge 모드(가장 흔함)**: parent NIC 위에서 다수의 macvlan 인터페이스를 브리지처럼 운용.
- **private / vepa / passthru**: 특수 환경에서 L2 프레임 전달 정책 상이(대부분 bridge 모드로 충분).

대부분 Docker에서는 **`-o parent=<nic>` + bridge 모드** 를 씁니다.

### 필수 전제/설계 항목

- **parent NIC**: 예) `eth0` (VLAN 태깅 환경이면 `eth0.10` 같은 **VLAN 서브인터페이스**를 parent로 지정 권장)
- **L2/L3 설계**: Subnet, Gateway, IP 할당 방식(DHCP/Static).
  일반적으로 **정적 IP** 를 권장(가정/사내 DHCP 충돌 최소화).
- **호스트 ↔ 컨테이너 통신 제한**: 동일 NIC에서 **호스트와 macvlan 간 L2 통신이 기본 불가** (커널 정책). 아래 해결책 참고.

### 빠른 생성 예제 (수동 CLI)

```bash
# macvlan 네트워크 생성

docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  my-macvlan

# 컨테이너를 macvlan에 연결 (정적 IP 부여)

docker run -d --name printer \
  --network my-macvlan \
  --ip 192.168.1.50 \
  nginx:alpine
```

- 이제 `192.168.1.50:80` 으로 **동일 LAN 상의 다른 장비**에서 접근 가능. 포트 매핑 불필요.

### 호스트 ↔ 컨테이너 직통 불가 문제 해결 (권장 루트)

**문제**: 같은 NIC(예: `eth0`) 위의 macvlan과 호스트는 **서로 보지 못함**.
**해결책(일반적인 두 가지)**:

**A) 호스트 측에 별도 macvlan 인터페이스 생성(bridge 모드) 후 static route**
```bash
# host OS에서 실행 (root 필요)
# 호스트에 macvlan 인터페이스 생성 (ip link)

ip link add macvlan0 link eth0 type macvlan mode bridge
ip addr add 192.168.1.240/24 dev macvlan0
ip link set macvlan0 up

# 이제 호스트 192.168.1.240 과 macvlan 컨테이너(예: 192.168.1.50) 간 통신 가능

ping -c1 192.168.1.50
```
- 라우터/스위치 입장에선 전부 L2 상 동등 장비.

**B) parent를 VLAN 서브인터페이스로 분리**
- 예: `eth0.30` 을 parent로 macvlan, `eth0` 은 호스트 기본.
- 호스트와 컨테이너 간 통신은 **라우터 경유**(VLAN간 라우팅)로 이루어지게 설계.

현실적으로 **A**가 간편하며, VLAN 분리가 필요하면 **B**로 아키텍처를 개정합니다.

### Compose 예제 — macvlan 외부 네트워크 사용

macvlan은 일반적으로 **Compose 내부에서 직접 생성**하기보다는, **외부에서 먼저 만든 네트워크**를 붙여쓰는 패턴이 안전합니다.

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

먼저 외부에서 네트워크를 만들어 둡니다:
```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 --gateway=192.168.1.1 \
  -o parent=eth0 macvlan_lan
```

> Compose에서 macvlan을 **내부 정의**로 만들 수도 있으나, 권한/호스트 종속이 강해 **external 패턴**을 권장합니다.

### 운영 팁/주의

- **DHCP와 충돌**: DHCP 영역과 macvlan의 static IP 영역을 분리.
- **MTU**: 물리망 MTU에 그대로 종속(Overlay와 달리 VXLAN 오버헤드 없음).
- **보안/감사**: 컨테이너가 **LAN 상 실체**로 보이므로, **방화벽/ACL** 정책에 반영.
- **Windows/Mac Docker Desktop**: macvlan은 **Linux 호스트**에서 완전 지원(Desktop은 제한/복잡).

---

## Overlay — 여러 호스트에 걸친 가상 네트워크

> **Swarm 모드**에서 활성화되는 L2.5 가상망(VXLAN).
> 노드 여러 대에 걸친 컨테이너들이 **동일 네트워크** 처럼 통신.

### 빠른 개념

- **Swarm 활성화 필수**: `docker swarm init`
- **VXLAN 터널링**: 데이터센터/클라우드 노드 간 패킷을 터널로 운송.
- **암호화 옵션**: `--opt encrypted` 로 **네트워크 레벨 암호화** 가능(성능 비용 고려).
- **attachable**: 서비스 뿐 아니라 **일반 컨테이너**도 네트워크에 붙을 수 있게.

### 기본 흐름

```bash
# Swarm 초기화 (첫 노드에서)

docker swarm init

# overlay 네트워크 생성(+ attachable)

docker network create -d overlay --attachable my-overlay

# 서비스(스웜 리소스)로 붙이기

docker service create --name web --network my-overlay nginx:alpine

# 일반 컨테이너도 붙일 수 있음

docker run -d --name debug --network my-overlay nicolaka/netshoot
```

노드가 여러 대라면 `docker swarm join` 토큰으로 워커/매니저 추가 후, 같은 overlay에 부착된 서비스/컨테이너끼리 **서로 통신** 가능.

### 포트 공개(ingress/routing mesh)

```bash
docker service create \
  --name api \
  --network my-overlay \
  --publish published=8080,target=80,mode=ingress \
  nginx:alpine
```

- **ingress**: **라우팅 메시**를 통해 모든 노드의 8080에서 API로 분산.
- `mode=host` 로 하면 **해당 태스크가 뜬 노드의 포트만** 바인딩(고성능/고정 매핑 선호 시).

### 암호화(옵션)

```bash
docker network create -d overlay --attachable --opt encrypted secure-ovl
```

- 데이터 플레인 암호화(노드 간 링크 보호). CPU 오버헤드 고려.

### Stack(Compose for Swarm) 예제

Swarm에서는 `docker stack deploy` 를 통해 Compose 유사 문법으로 배포합니다.

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
      restart_policy: { condition: on-failure }

  api:
    image: myorg/api:1.0
    networks: [appnet]
    deploy:
      replicas: 3
      restart_policy: { condition: on-failure }
      placement:
        constraints: [ "node.platform.os == linux" ]

  fe:
    image: nginx:alpine
    networks: [appnet]
    deploy:
      replicas: 2
      labels:
        - traefik.enable=true
    ports:
      - "8080:80"   # 스택 문법에서는 services.fe.ports로 퍼블리시
```

배포:
```bash
docker stack deploy -c stack-ovl.yml appstack
docker stack services appstack
docker stack ps appstack
```

### MTU/오버헤드(간단 수식)

VXLAN 캡슐화 오버헤드는 대략 **50바이트 전후**입니다.
유효 MTU는:
$$
MTU_{\text{effective}} \approx MTU_{\text{underlay}} - 50
$$

예: 물리망 1500이면 overlay 상 컨테이너 MTU는 약 1450.
대용량 UDP/ICMP 시 단편화 가능성 → **애플리케이션/OS MTU 조정** 고려.

---

## Compose에서 네트워크 직접 설정 — 분리/연결/외부 활용

### 다중 네트워크 분리(프런트/백)

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

- **외부 노출**은 `fe`만.
- `api`/`db`는 backnet에서만 통신 → **경계가 명확**.

### 외부 네트워크 붙이기(bridge/macvlan/overlay 등)

```yaml
networks:
  corp_lan:
    external: true
```

- 운영망에 미리 존재하는 네트워크를 **참조만**.

### 드라이버/옵션 지정

```yaml
networks:
  iso:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.enable_icc: "false"   # 컨테이너 간 통신 차단
      com.docker.network.bridge.enable_ip_masquerade: "true"
    ipam:
      driver: default
      config:
        - subnet: 172.30.0.0/16
          gateway: 172.30.0.1
```

- 테스트용 **격리 네트워크** 구현 가능.

### 이름 기반 통신과 헬스체크 연계

- 같은 네트워크에서 `api`는 `db:5432`로 접근.
- Compose v2에서는 `depends_on.condition=service_healthy` 로 **준비 완료 뒤 기동** 조절.

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy
```

---

## 보안/성능/운영 베스트 프랙티스

### 보안

- **네트워크 분리**(프런트/백/관리)로 최소 노출 면적 확보.
- **필요 포트만 공개**: compose의 `ports`는 가급적 프런트 계층에서만.
- **macvlan**: LAN 정책(방화벽/ACL/IDS)에 컨테이너 IP가 **실체**로 보임. 로그/모니터링 포함.
- **overlay**: `--opt encrypted` 고려(링크 암호화) + **노드 보안**(비밀키/매니저 보호).
- 컨테이너 **비루트**(`user`) + **read_only** 루트FS + **cap_drop: [ALL]** + **tmpfs**로 최소 권한.

### 성능

- **host** > macvlan/bridge > overlay 순(대체로).
- overlay는 MTU/캡슐화 오버헤드로 지연 증가 가능.
- 패킷이 큰 워크로드는 **MTU 조정** 또는 **route-mode publishing** 검토.

### 운영/관측

- **DNS**: 도커 내장 레졸버(127.0.0.11) 동작 확인.
- **로깅/메트릭**: `docker events`, `docker logs`, 노드/네트워크 지표 수집(ebpf/pcap).
- **네임스페이스 가시화**: `nsenter`, `ip netns` (root 필요), `nicolaka/netshoot` 컨테이너 적극 활용.

---

## 트러블슈팅 체크리스트

| 증상 | 원인 후보 | 진단/해결 |
|------|-----------|-----------|
| macvlan 컨테이너 ↔ 호스트 통신 불가 | 커널 정책(동일 parent) | **호스트에 macvlan 인터페이스 추가**(2.4-A) 또는 VLAN 분리(2.4-B) |
| 동일망 기기에서 ARP가 비정상 | IP 충돌/DHCP 겹침 | DHCP 범위/정적 IP 재정렬, `arp -a`, `ip neigh` 확인 |
| overlay 사이에서 지연/패킷 유실 | MTU/단편화 | 유효 MTU 계산, 앱/OS MTU 낮추기(예: 1450), 경로 MTU 디스커버리 |
| 서비스 노출이 모든 노드에서 열린다 | routing mesh(ingress) | `mode=host` 로 포트 퍼블리시 변경 |
| 내부 이름해결 실패 | 서로 다른 네트워크 | `docker network inspect`, `getent hosts <svc>` 로 네트워크 가입 여부 확인 |
| 포트가 열리지 않음 | 잘못된 바인딩/방화벽 | `ports`의 바인딩 IP(127.0.0.1 vs 0.0.0.0) 확인, 호스트 방화벽 규칙 점검 |
| Compose 기동 순서 문제 | 단순 depends_on | **healthcheck + condition** 로 현실적 의존 처리 |

**진단 명령 모음**
```bash
# 컨테이너 내부에서 네트워크 관찰

docker run --rm -it --network <net> nicolaka/netshoot sh
ip addr; ip route; cat /etc/resolv.conf; getent hosts api

# 네트워크 상세

docker network ls
docker network inspect <net>

# ARP/이웃 캐시

ip neigh

# 패킷 캡처(호스트)

sudo tcpdump -ni eth0 host 192.168.1.50

# Compose 전체 구성 확인

docker compose config
```

---

## 부록

### 토폴로지 예시 (문자 다이어그램)

```
[LAN 192.168.1.0/24]
   |                  (Router/GW .1)
   +---[Host eth0 .10]------------------------------+
   |                                               |
   | (A) macvlan bridge mode                       | (B) overlay (다중 호스트)
   |   +-- docker macvlan net: macvlan_lan         |   Swarm
   |   |     +-- cam(.60)  +-- sensor(.61)         |   +-- node1, node2, node3
   |   |                                            \-- overlay: appnet
   |   +-- host macvlan0(.240) ↔ 컨테이너 직통           +-- services: fe/api/db
   |
   +-- 일반 bridge: backnet (fe/api/db 내부 통신)
```

### 용어 빠른 정리

- **parent NIC**: macvlan이 매달리는 실물(혹은 VLAN 서브인터페이스).
- **attachable**(overlay): 일반 컨테이너도 overlay에 붙일 수 있게.
- **routing mesh**: ingress 포트가 **클러스터 전체 노드로 공개**되는 Swarm 기능.
- **healthcheck**: 의존/기동 순서 제어의 사실상 표준.

### 치트시트

```bash
# macvlan(외부 생성)

docker network create -d macvlan \
  --subnet=192.168.1.0/24 --gateway=192.168.1.1 \
  -o parent=eth0 macvlan_lan

# host 측 macvlan0 만들기(직통 해결)

ip link add macvlan0 link eth0 type macvlan mode bridge
ip addr add 192.168.1.240/24 dev macvlan0
ip link set macvlan0 up

# overlay

docker swarm init
docker network create -d overlay --attachable appnet
docker service create --name fe --network appnet --publish 8080:80 nginx:alpine
docker run -d --name dbg --network appnet nicolaka/netshoot

# compose 검증

docker compose config
```

---

## 전체 예제 세트

### A) Macvlan + Compose(외부 네트워크 참조)

```yaml
version: "3.9"
services:
  cam:
    image: nginx:alpine
    networks:
      lan:
        ipv4_address: 192.168.1.60
    # 포트 매핑 불필요: LAN에서 cam:80 직접 접근

networks:
  lan:
    external: true
```

**외부 생성**
```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 --gateway=192.168.1.1 \
  -o parent=eth0 lan
```

### B) Overlay + Stack(스웜)

```yaml
# stack.yml

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

  api:
    image: ghcr.io/myorg/api:1.0
    networks: [appnet]
    deploy:
      replicas: 3

  fe:
    image: nginx:alpine
    networks: [appnet]
    deploy:
      replicas: 2
    ports:
      - "8080:80"   # ingress
```

배포:
```bash
docker swarm init
docker stack deploy -c stack.yml app
```

### C) Bridge(내부) + Front 공개, Back 보호

```yaml
version: "3.9"
services:
  fe:
    image: nginx:alpine
    ports: ["8080:80"]
    networks: [front, back]
    depends_on: [api]
  api:
    build: ./api
    networks: [back]
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:15
    networks: [back]
    healthcheck:
      test: ["CMD-SHELL","pg_isready -U postgres || exit 1"]
      interval: 5s
      retries: 12

networks:
  front: {}
  back: {}
```

---

## 결론

- **macvlan**: 컨테이너를 **LAN 상 실체 장비**처럼 노출해야 할 때 정답. 호스트↔컨테이너 통신은 **별도 macvlan0** 또는 **VLAN 분리**로 해결.
- **overlay**: **여러 호스트**에 걸친 서비스 연결의 표준. Swarm, attachable, ingress/host publish 모드, 암호화 등 **아키텍처 선택지** 풍부.
- **Compose/Stack**: **네트워크 경계**를 선언으로 고정하고, 이름 기반 통신과 헬스체크로 **기동 안정성**을 확보.

두 기술은 상호 배타적이 아니라 **환경 목적**에 맞춰 **혼용**됩니다. 단일 호스트에서는 macvlan/bridge를, 클러스터/멀티노드에서는 overlay를 채택하는 식으로 **설계 의도**를 명확히 하십시오.
