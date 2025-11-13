---
layout: post
title: Docker - Host Network vs Bridge Network
date: 2025-01-26 20:20:23 +0900
category: Docker
---
# Docker 네트워크 비교: Host Network vs Bridge Network

## 1. Docker 네트워크 드라이버 개관(정교화)

- **bridge**: 기본값. 호스트 내 가상 브리지(일반적으로 `docker0`)를 중심으로, 컨테이너들이 **프라이빗 IP**(예: `172.17.0.0/16`)를 받아 통신. 외부 접근은 **포트 매핑** 또는 라우팅/프록시로 노출.
- **host**: 컨테이너가 호스트와 **동일 네트워크 네임스페이스**를 공유. 컨테이너의 IP 공간이 따로 없고 호스트의 IP/포트 테이블을 그대로 사용.
- **none**: 네트워크 비활성(완전 격리). 사이드카/보안 격리 실험 등 특별한 경우.
- **overlay**: Swarm(또는 K8s CNI와 유사)의 **다중 호스트 간 가상 네트워크**.
- **macvlan**: 컨테이너에 **개별 MAC**을 부여, L2 레벨에서 호스트와 분리된 엔티티처럼 보이게.

이 글은 **host vs bridge**를 중심으로 실무 차이를 파고듭니다.

---

## 2. Bridge Network: 내부 동작 원리와 실전

### 2.1 구조와 흐름

```
(컨테이너) eth0 —— vethX ===[ docker0(브리지) ]=== 호스트 nic(eth0) —— 외부
                        │
                   iptables(NAT, DNAT/PREROUTING, MASQUERADE)
```

- 컨테이너마다 `veth` 페어가 생성되어 `docker0`에 붙습니다.
- 컨테이너 ↔ 외부 인터넷은 **SNAT(MASQUERADE)**, 외부 ↔ 컨테이너는 **DNAT(포트 매핑)** 로 중개.
- 같은 브리지 내 컨테이너끼리는 **L2 스위칭**으로 직접 통신.

### 2.2 핵심 특징(확장)

- **격리**: 기본적으로 외부에서 컨테이너 IP로 직접 유입 불가. `-p`로 명시적 노출만 허용 → 안전한 디폴트.
- **서비스 디스커버리**: 사용자 정의 브리지에서 **컨테이너 이름으로 DNS**(내장 DNS 127.0.0.11) 해결.
- **헤어핀 NAT**: 호스트에서 `localhost:포트`로 접근 시 DNAT 경로를 타며, 같은 브리지 내에서 **자신의 퍼블릭 포트로 자기 자신 접근**은 설정에 따라 헤어핀 동작이 필요할 수 있음.

### 2.3 빠른 예제

```bash
# 1. 기본 브리지 모드로 웹 노출
docker run -d --name web1 -p 8080:80 nginx

# 2. 사용자 정의 브리지 생성(권장 패턴)
docker network create --driver bridge appnet

# 3. 같은 네트워크로 2개 서비스 배치
docker run -d --name api --network appnet nginx
docker run -d --name fe  --network appnet nginx

# 4. 컨테이너 간 이름으로 접근(내부 DNS)
docker exec -it fe sh -lc "apk add --no-cache curl; curl -s http://api"
```

### 2.4 고정 IP/서브넷 지정(테스트/특정 라우팅 용)

```bash
docker network create \
  --driver bridge \
  --subnet 172.30.0.0/24 \
  --gateway 172.30.0.1 \
  appnet30

docker run -d --name db \
  --network appnet30 \
  --ip 172.30.0.10 \
  postgres:16
```

> 운영에서는 **IP 고정보다 이름 기반 디스커버리**를 권장.

### 2.5 관찰·디버깅

```bash
docker network inspect bridge
ip addr show docker0
iptables -t nat -S | grep -E 'DOCKER|MASQUERADE'
```

패킷 캡처(브리지 관찰):
```bash
sudo tcpdump -i docker0 -n 'tcp port 80'
```

---

## 3. Host Network: 내부 동작 원리와 실전

### 3.1 구조와 흐름

```
(컨테이너 프로세스)   ——[ 호스트 네트워크 네임스페이스 공유 ]——  호스트 nic(eth0) —— 외부
(별도 veth/브리지/NAT 없음, 포트 매핑 무의미)
```

- 컨테이너가 호스트와 **동일 네트워크 네임스페이스**를 사용 → 별도 IP 부여 없음, **호스트 IP/포트 그대로**.
- NAT/포트포워딩 우회 → **오버헤드 최소**.

### 3.2 핵심 특징(확장)

- **포트 충돌**: 동일 포트를 쓰는 다른 프로세스(다른 컨테이너 포함)가 있으면 **실행 실패**.
- **성능**: NAT/브리지 우회를 통한 **낮은 지연**. 프록시/패킷 캡처/로드밸런서에 유리.
- **격리**: 네트워크 격리가 사실상 사라짐. **호스트 방화벽/보안 정책**과 충돌 가능성, 관찰성 낮아질 수 있음.

### 3.3 빠른 예제

```bash
# host 네트워크로 바로 공개(포트 매핑 옵션은 무시됨)
docker run -d --name edge --network host nginx

# 테스트: 호스트에서 바로 접근
curl -s http://127.0.0.1:80 | head
```

> 이미 호스트에서 80 포트를 쓰는 프로세스가 있다면 실패.

### 3.4 관찰·디버깅

- 일반 프로세스와 동일하게 `ss -lntp`, `netstat`, `lsof -i`로 포트 점유 확인.
- iptables는 **호스트 정책**을 그대로 따름. 컨테이너 단위 룰 분리는 어려움.

---

## 4. Host vs Bridge — 차이와 선택 기준(현업 관점)

| 항목 | **Bridge** | **Host** |
|---|---|---|
| IP/포트 공간 | 컨테이너 전용 프라이빗 IP, 포트 매핑 필요 | 호스트와 동일 IP/포트, 매핑 불필요(무시) |
| 성능 경로 | veth/브리지/NAT 경유 | 직접(네이티브) 경로 |
| 격리/보안 | 높음(기본 차단, 명시적 공개) | 낮음(호스트와 동일 경계) |
| 운용 난이도 | 포트 설계/매핑/브리지 관리 | 포트 충돌/관찰성, 호스트 보안 의존 |
| 서비스 디스커버리 | 이름 기반(DNS) 용이(사용자 정의 브리지) | 동일 호스트 내 다른 컨테이너와 이름 접근 불편 |
| 관찰/트레이스 | docker0 / iptables로 구간 추적 가능 | 시스템 전역으로 섞여 추적 더 어려움 |
| 주사용처 | 일반 웹/백엔드/DB, 마이크로서비스 | 고성능 프록시/IDS/테스터/특수 네트워킹 |

**실무 가이드**
- 기본은 **bridge**.
- **패킷 경로 단순화/저지연**이 절대적으로 중요하고 **포트/보안 통제가 가능**한 경우에만 **host**.

---

## 5. Compose로 비교(운영 패턴)

```yaml
version: "3.9"

services:
  api_bridge:
    image: nginx:alpine
    ports:
      - "8080:80"        # bridge: 외부 8080 → 컨테이너 80
    networks:
      - appnet

  edge_host:
    image: nginx:alpine
    network_mode: host   # host: 포트 매핑 불필요, 호스트 80 직접 점유

networks:
  appnet:
    driver: bridge
```

---

## 6. 성능·튜닝

### 6.1 간단 부하 예시(비교 절차)
```bash
# bridge
docker run -d --name bweb -p 8080:80 nginx
# host
docker run -d --name hweb --network host nginx

# ApacheBench 예: 지연/처리량 대조
ab -n 10000 -c 200 http://127.0.0.1:8080/      # bridge
ab -n 10000 -c 200 http://127.0.0.1:80/        # host
```
> 환경에 따라 **host가 낮은 지연**, **bridge는 약간의 NAT 오버헤드**가 보일 수 있음.

### 6.2 Bridge 고급 튜닝
- 사용자 브리지로 **IP/MTU/서브넷** 명시, 충돌 회피.
- 대량 연결 서버는 호스트 **sysctl** 조정(예):
```bash
sudo sysctl -w net.ipv4.ip_local_port_range="1024 65000"
sudo sysctl -w net.ipv4.tcp_fin_timeout=30
sudo sysctl -w net.core.somaxconn=1024
```

---

## 7. 보안·격리·권한

### 7.1 Bridge 모드 권장 수칙
- 외부 노출은 **필요 포트만 -p**로 최소화.
- 컨테이너는 **비루트 USER** + 루트FS **read-only** + **cap-drop**:
```bash
docker run -d \
  --cap-drop ALL \
  --read-only \
  --tmpfs /tmp \
  --user 65532:65532 \
  -p 8080:80 nginx:alpine
```

### 7.2 Host 모드 주의
- 네트워크 경계가 흐려짐 → **호스트 방화벽/IDS 룰** 정비 필수.
- 포트 충돌 방지. 의도치 않은 **서비스 노출** 가능성 점검.

---

## 8. 실전 테스트·디버깅 레시피

### 8.1 포트 충돌 재현(Host)
```bash
docker run -d --network host --name webA nginx
docker run -d --network host --name webB nginx  # (대개 실패: 80 충돌)
```

### 8.2 iptables/NAT 확인(Bridge)
```bash
iptables -t nat -L -n -v | sed -n '1,120p'
```

### 8.3 tcpdump로 흐름 보기
```bash
sudo tcpdump -i docker0 -n 'tcp port 80'
sudo tcpdump -i any -n 'tcp port 8080'
```

### 8.4 컨테이너 내부 라우팅/ARP
```bash
docker exec -it bweb sh -lc "ip route; ip neigh"
```

---

## 9. DNS·서비스 디스커버리·헤어핀 NAT

- **사용자 정의 브리지**에서 **컨테이너 이름으로 DNS 조회** 가능 (`/etc/resolv.conf`의 127.0.0.11).
- **기본 bridge**는 이름 해석 제약이 있으므로, 서비스간 통신은 **사용자 정의 브리지**를 권장.
- **헤어핀 NAT**: 같은 노드에서 자기 퍼블릭 포트로 접속 시 동작이 환경별로 다름. 가장 간단한 해법은 **동일 네트워크 이름 주소로 직접 접근**.

---

## 10. Docker Desktop / WSL2 / IPv6 특성

- **Docker Desktop(macOS/Windows)**: 내부적으로 **VM**. 바인드/포트가 하이퍼바이저 경로를 지나 **I/O 지연**이 커질 수 있음 → 대량 개발 작업은 **볼륨/캐시** 전략 병행.
- **WSL2**: `host.docker.internal`이 동작하며, WSL 네트워크 ↔ Windows 호스트 간 NAT/포트포워딩 규칙 체크.
- **IPv6**: `daemon.json`에서 활성화 후 사용자정의 브리지에 프리픽스 지정:
```json
{
  "ipv6": true,
  "fixed-cidr-v6": "2001:db8:1::/64",
  "experimental": false
}
```
이후 네트워크 생성 시 IPv6 라우팅/방화벽 정책 재검토.

---

## 11. 트러블슈팅 표

| 증상 | 가능 원인 | 진단 | 해결 |
|---|---|---|---|
| host 모드 포트 바인딩 실패 | 포트 충돌 | `ss -lntp`, `lsof -i` | 포트 변경/프로세스 정리 |
| bridge 노출 안됨 | `-p` 미지정 / 방화벽 | `docker ps`, `iptables -t nat -S` | `-p` 지정, 방화벽 허용 |
| 컨테이너 간 통신 불가 | 서로 다른 네트워크 | `docker network inspect` | 동일 사용자 브리지 연결 |
| DNS 이름 해석 실패 | 기본 bridge 사용 | `/etc/resolv.conf` | 사용자 정의 브리지로 이전 |
| 성능 저하 | NAT/가상화/MTU | `ab`, `iperf3`, `ip -d link` | host 모드/MTU 조정/직접 LB |
| 헤어핀 실패 | NAT 규칙 부재 | iptables nat 테이블 | 내부 DNS/서비스 이름 사용 |

---

## 12. 체크리스트(선택 가이드)

- [ ] 기본은 **bridge**. 공개는 필요한 포트만 `-p`.
- [ ] 컨테이너 간 통신 필요 → **사용자 정의 브리지** + 이름 기반.
- [ ] 초저지연/프록시/패킷 도구 → **host**, 단 보안/충돌/관찰성 각오.
- [ ] Docker Desktop/WSL2 → I/O/포트포워딩 경로 이해 후 설계.
- [ ] IPv6 사용 시 데몬/네트워크/보안 정책 **동시 구성**.
- [ ] 성능/장애 조사: `tcpdump`, `iptables`, `ss`, `docker network inspect`.

---

## 13. 추가 실습: Nginx + 앱 2단 구성(Bridge vs Host)

### 13.1 Bridge(권장)
```bash
docker network create appnet

docker run -d --name app --network appnet hashicorp/http-echo \
  -text="hello from app"

docker run -d --name fe --network appnet -p 8080:80 nginx:alpine
# 프록시 설정 주입(간단 예)
docker exec -it fe sh -lc 'printf "server { listen 80;
  location / { proxy_pass http://app:5678; } }" > /etc/nginx/conf.d/default.conf && nginx -s reload'

curl -s localhost:8080
```

### 13.2 Host(고성능 경로)
```bash
# app는 host 포트 5678 사용
docker run -d --name apph --network host hashicorp/http-echo -text="hello host" -listen=:5678

# fe도 host, 80 사용(충돌 유의)
docker run -d --name feh --network host nginx:alpine
docker exec -it feh sh -lc 'printf "server { listen 80;
  location / { proxy_pass http://127.0.0.1:5678; } }" > /etc/nginx/conf.d/default.conf && nginx -s reload'

curl -s http://127.0.0.1/
```

---

## 14. 수식으로 보는 의사결정(직관)

네트워크 모드 선택 효용 \(U\)를 **성능 \(P\)**, **보안/격리 \(S\)**, **운영성 \(O\)**의 가중합으로 단순화:
$$
U = \alpha P + \beta S + \gamma O
$$
- 마이크로서비스/일반 웹: \( \beta,\gamma \) 비중 ↑ → **bridge**
- L4 프록시/패킷 수집: \( \alpha \) 비중 ↑ → **host**

---

## 15. 요약

- **Bridge**: 격리·명시적 공개·내장 DNS·운영 친화. 대부분 워크로드의 기본값.
- **Host**: NAT/브리지 우회로 저지연. 포트 충돌/보안 경계/관찰성에 주의.
- 실무에서는 **사용자 정의 브리지 + 이름 기반 통신 + 필요한 포트만 공개**가 표준.
- 성능이 가장 핵심이면 **host**를 고려하되, **포트/보안/관찰성**을 먼저 설계.

---
```bash
# 핵심 명령 요약

# 브리지 노출
docker run -d -p 8080:80 nginx

# 사용자 정의 브리지
docker network create --driver bridge appnet
docker run -d --network appnet --name s1 nginx
docker run -it --network appnet --rm alpine sh -lc "apk add curl; curl -s http://s1"

# 호스트 네트워크
docker run -d --network host nginx
# 포트 매핑 옵션(-p)은 host에서 무시됨
```
