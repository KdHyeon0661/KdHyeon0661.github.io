---
layout: post
title: Docker - Host Network vs Bridge Network
date: 2025-01-26 20:20:23 +0900
category: Docker
---
# Docker 네트워크 비교: Host Network vs Bridge Network

Docker에서 컨테이너의 네트워킹 방식을 선택할 때 가장 많이 고려되는 두 가지 옵션은 Host Network와 Bridge Network입니다. 각각의 방식은 고유한 특징과 장단점을 가지고 있으며, 애플리케이션의 요구사항과 운영 환경에 따라 적절한 선택이 필요합니다.

## Docker 네트워크 드라이버 개요

Docker는 다양한 네트워크 드라이버를 제공하여 컨테이너의 네트워킹 방식을 유연하게 구성할 수 있습니다:

- **bridge**: 기본값으로, 가상 네트워크 브리지를 통해 컨테이너 간 통신을 제공합니다.
- **host**: 컨테이너가 호스트의 네트워크 스택을 직접 사용합니다.
- **none**: 네트워크를 완전히 비활성화합니다.
- **overlay**: 다중 호스트 환경에서 컨테이너 간 통신을 가능하게 합니다.
- **macvlan**: 컨테이너에 실제 MAC 주소를 할당하여 물리적 네트워크에 직접 연결합니다.

이 글에서는 가장 일반적으로 사용되는 **bridge**와 **host** 네트워크 모드의 차이점을 심층적으로 분석합니다.

---

## Bridge Network: 격리된 가상 네트워크

### 작동 원리와 구조

Bridge 네트워크는 Docker의 기본 네트워크 모드로, 각 컨테이너에 가상 네트워크 인터페이스를 제공합니다. 구조는 다음과 같습니다:

```
[컨테이너] eth0 ←→ vethX ←→ [docker0 브리지] ←→ 호스트 NIC ←→ 외부 네트워크
                ↑
         NAT/포트 포워딩(iptables)
```

### 주요 특징

1. **네트워크 격리**: 각 컨테이너는 독립된 IP 주소를 가지며, 기본적으로 외부에서 직접 접근할 수 없습니다.
2. **포트 매핑 필요**: 컨테이너 서비스를 외부에 노출하려면 `-p` 옵션으로 명시적 포트 매핑이 필요합니다.
3. **내부 DNS**: 사용자 정의 브리지 네트워크에서는 컨테이너 이름으로 서로 통신할 수 있습니다.
4. **보안**: 기본적으로 격리되어 있어 보안성이 상대적으로 높습니다.

### 실무 사용 예제

```bash
# 기본 브리지 네트워크로 컨테이너 실행
docker run -d --name web-server -p 8080:80 nginx:alpine

# 사용자 정의 브리지 네트워크 생성
docker network create app-network

# 동일 네트워크의 컨테이너들은 이름으로 서로 접근 가능
docker run -d --name api --network app-network my-api:latest
docker run -d --name frontend --network app-network -p 3000:3000 my-frontend:latest

# API 컨테이너에서 frontend 컨테이너 접근 테스트
docker exec api curl http://frontend:3000
```

### 네트워크 구성 커스터마이징

```bash
# 서브넷과 게이트웨이를 지정한 네트워크 생성
docker network create \
  --driver bridge \
  --subnet 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  custom-network

# 고정 IP로 컨테이너 실행
docker run -d --name database \
  --network custom-network \
  --ip 192.168.100.10 \
  postgres:latest
```

### 모니터링과 디버깅

```bash
# 네트워크 정보 확인
docker network inspect bridge

# Docker 브리지 인터페이스 확인
ip addr show docker0

# NAT 규칙 확인
sudo iptables -t nat -L -n

# 네트워크 트래픽 모니터링
sudo tcpdump -i docker0 -n 'tcp port 80'
```

---

## Host Network: 호스트 네트워크 공유

### 작동 원리와 구조

Host 네트워크 모드에서는 컨테이너가 호스트의 네트워크 스택을 직접 사용합니다. 이 방식은 추가적인 네트워크 계층 없이 직접 통신합니다:

```
[컨테이너 프로세스] ←→ [호스트 네트워크 스택 공유] ←→ 호스트 NIC ←→ 외부 네트워크
```

### 주요 특징

1. **직접적인 네트워크 접근**: 컨테이너는 호스트의 IP 주소와 포트를 직접 사용합니다.
2. **포트 충돌 가능성**: 동일한 포트를 사용하는 다른 프로세스와 충돌할 수 있습니다.
3. **성능 최적화**: NAT 오버헤드가 없어 네트워크 성능이 향상됩니다.
4. **보안 고려사항**: 네트워크 격리가 없으므로 보안 설정에 주의가 필요합니다.

### 실무 사용 예제

```bash
# Host 네트워크로 컨테이너 실행
docker run -d --name proxy --network host nginx:alpine

# 호스트에서 직접 접근 (포트 매핑 없음)
curl http://localhost:80

# 포트 충돌 예시 (이미 80포트 사용 중일 경우)
docker run -d --name web2 --network host nginx:alpine  # 실패 가능성 있음
```

### Host 네트워크의 모니터링

```bash
# 호스트의 모든 네트워크 연결 확인
ss -lntp

# 특정 포트 사용 프로세스 확인
sudo lsof -i :80

# 호스트의 네트워크 인터페이스 확인
ip addr show
```

---

## Host Network vs Bridge Network: 비교 분석

### 성능 측면

| 항목 | Bridge Network | Host Network |
|------|---------------|--------------|
| **네트워크 오버헤드** | NAT와 브리징으로 인한 약간의 오버헤드 | 최소한의 오버헤드 |
| **처리량** | 일반적으로 충분하나 약간의 감소 가능 | 최대 처리량 가능 |
| **지연 시간** | 약간 증가 | 최소 지연 시간 |

### 보안과 격리

| 항목 | Bridge Network | Host Network |
|------|---------------|--------------|
| **네트워크 격리** | 우수 (기본적으로 격리) | 없음 (호스트와 공유) |
| **포트 노출** | 명시적 포트 매핑 필요 | 모든 포트 자동 노출 |
| **보안 정책 적용** | 컨테이너별 정책 적용 가능 | 호스트 수준 정책만 적용 |

### 운영과 관리

| 항목 | Bridge Network | Host Network |
|------|---------------|--------------|
| **포트 관리** | 포트 매핑으로 유연한 관리 | 포트 충돌 관리 필요 |
| **서비스 디스커버리** | 내장 DNS로 컨테이너 이름 기반 접근 | 직접 IP/호스트 이름 사용 |
| **모니터링** | Docker 네트워크 명령어로 관리 | 호스트 네트워크 도구 사용 |
| **이식성** | 높음 (호스트 독립적) | 낮음 (호스트 환경 의존) |

### 성능 테스트 비교

```bash
# Bridge 네트워크 테스트
docker run -d --name web-bridge -p 8080:80 nginx:alpine
ab -n 10000 -c 100 http://localhost:8080/

# Host 네트워크 테스트
docker run -d --name web-host --network host nginx:alpine
ab -n 10000 -c 100 http://localhost:80/
```

---

## 실무 선택 가이드: 언제 무엇을 사용할까?

### Bridge Network를 선택해야 하는 경우

1. **웹 애플리케이션 서비스**: 대부분의 웹 서비스에 적합합니다.
2. **마이크로서비스 아키텍처**: 서비스 간 통신이 필요한 환경.
3. **다중 컨테이너 환경**: 여러 컨테이너가 협업하는 시스템.
4. **보안이 중요한 환경**: 네트워크 격리가 필요한 경우.
5. **개발 및 테스트 환경**: 포트 충돌을 피하고 격리된 환경이 필요할 때.

```bash
# 일반적인 웹 서비스 배포 패턴
docker network create web-network

# 데이터베이스
docker run -d --name db --network web-network \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:latest

# 백엔드 API
docker run -d --name api --network web-network \
  -p 3000:3000 \
  my-backend:latest

# 프론트엔드
docker run -d --name frontend --network web-network \
  -p 80:80 \
  nginx:alpine
```

### Host Network를 선택해야 하는 경우

1. **고성능 네트워킹 요구**: 네트워크 성능이 가장 중요한 경우.
2. **네트워크 모니터링 도구**: 패킷 분석이나 네트워크 모니터링 도구.
3. **특정 포트 번호 필수**: 특정 포트 번호를 반드시 사용해야 하는 경우.
4. **단일 서비스 배포**: 단일 컨테이너만 실행하는 간단한 환경.

```bash
# 고성능 프록시 서버
docker run -d --name haproxy --network host \
  -v /etc/haproxy:/usr/local/etc/haproxy:ro \
  haproxy:alpine

# 네트워크 모니터링 도구
docker run -d --name packet-capture --network host \
  -v /captures:/captures \
  nicolaka/netshoot \
  tcpdump -i any -w /captures/capture.pcap
```

### 보안 모범 사례

#### Bridge Network 보안 강화
```bash
docker run -d --name secure-app \
  --cap-drop ALL \                      # 불필요한 권한 제거
  --read-only \                         # 루트 파일시스템 읽기 전용
  --tmpfs /tmp \                        # 임시 파일시스템
  --user 1000:1000 \                    # 비권한 사용자로 실행
  -p 8080:80 \
  nginx:alpine
```

#### Host Network 보안 고려사항
```bash
# 호스트 방화벽 설정 (iptables 예시)
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -A INPUT -j DROP  # 명시적 허용 외 차단

# 호스트에서 실행 중인 서비스 확인
sudo netstat -tlnp
```

---

## Docker Compose를 활용한 네트워크 구성

### Bridge 네트워크를 사용한 구성
```yaml
version: '3.8'

services:
  database:
    image: postgres:latest
    networks:
      - app-network
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: securepassword

  backend:
    build: ./backend
    networks:
      - app-network
    ports:
      - "3000:3000"
    depends_on:
      - database

  frontend:
    build: ./frontend
    networks:
      - app-network
    ports:
      - "80:80"
    depends_on:
      - backend

networks:
  app-network:
    driver: bridge

volumes:
  postgres-data:
```

### Host 네트워크를 사용한 구성
```yaml
version: '3.8'

services:
  proxy:
    image: nginx:alpine
    network_mode: host  # Host 네트워크 사용
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    # ports:  # Host 네트워크에서는 포트 매핑 무시됨

  monitor:
    image: prom/node-exporter
    network_mode: host  # 호스트 메트릭 수집
```

---

## 문제 해결 가이드

### 일반적인 문제와 해결 방법

#### 문제 1: 포트 충돌
**증상**: `Bind for 0.0.0.0:80 failed: port is already allocated`
**해결**:
```bash
# 사용 중인 포트 확인
sudo lsof -i :80
sudo ss -lntp | grep :80

# 충돌하는 프로세스 중지 또는 다른 포트 사용
```

#### 문제 2: 컨테이너 간 통신 실패
**증상**: 컨테이너가 서로 통신할 수 없음
**해결**:
```bash
# 네트워크 연결 확인
docker network inspect [네트워크이름]

# 동일 네트워크에 컨테이너 연결
docker network connect [네트워크이름] [컨테이너이름]

# DNS 확인
docker exec [컨테이너] nslookup [다른컨테이너]
```

#### 문제 3: 외부에서 컨테이너 접근 불가
**증상**: 호스트에서는 접근 가능하지만 외부에서는 불가능
**해결**:
```bash
# 방화벽 규칙 확인
sudo iptables -L -n

# Docker NAT 규칙 확인
sudo iptables -t nat -L -n

# 호스트 라우팅 확인
ip route show
```

#### 문제 4: 네트워크 성능 저하
**증상**: 네트워크 성능이 예상보다 느림
**해결**:
```bash
# 네트워크 인터페이스 통계 확인
ip -s link show docker0

# 패킷 드롭 확인
netstat -s | grep -i drop

# MTU 설정 확인
ip link show docker0 | grep mtu
```

### 디버깅 도구 활용

```bash
# 네트워크 연결 테스트
docker run --rm -it --network app-network alpine ping api

# HTTP 연결 테스트
docker run --rm --network app-network curlimages/curl http://api:3000/health

# 네트워크 라우팅 확인
docker exec container-name ip route show

# DNS 해석 테스트
docker exec container-name nslookup google.com
```

---

## 고급 네트워크 구성

### IPv6 지원 활성화
```bash
# Docker 데몬 설정 수정
sudo tee /etc/docker/daemon.json << EOF
{
  "ipv6": true,
  "fixed-cidr-v6": "2001:db8:1::/64"
}
EOF

# Docker 재시작
sudo systemctl restart docker

# IPv6 네트워크 생성
docker network create --ipv6 --subnet=2001:db8:1::/64 ipv6-network
```

### 네트워크 대역폭 제한
```bash
# 네트워크 대역폭 제한이 있는 네트워크 생성
docker network create \
  --driver bridge \
  --opt com.docker.network.bridge.name=mybridge \
  --opt com.docker.network.bridge.enable_icc=true \
  --opt com.docker.network.bridge.enable_ip_masquerade=true \
  --opt com.docker.network.bridge.host_binding_ipv4=0.0.0.0 \
  limited-network
```

---

## 결론

Docker 네트워크 모드 선택은 애플리케이션의 요구사항, 성능 요구치, 보안 정책, 운영 환경을 종합적으로 고려해야 합니다.

### Bridge Network가 적합한 경우
- 대부분의 웹 애플리케이션과 마이크로서비스
- 다중 컨테이너 환경에서의 서비스 간 통신
- 네트워크 격리와 보안이 중요한 환경
- 개발 및 테스트 환경

### Host Network가 적합한 경우
- 네트워크 성능이 가장 중요한 고성능 애플리케이션
- 네트워크 모니터링 및 패킷 분석 도구
- 특정 포트 번호를 반드시 사용해야 하는 경우
- 단일 서비스로 구성된 간단한 환경

### 실무 권장 사항

1. **기본적으로 Bridge Network 사용**: 대부분의 경우 Bridge 네트워크가 최적의 선택입니다.
2. **사용자 정의 네트워크 활용**: 프로덕션 환경에서는 항상 사용자 정의 브리지 네트워크를 생성하고 사용하세요.
3. **보안 강화**: 컨테이너는 최소한의 권한으로 실행하고, 필요한 포트만 노출하세요.
4. **성능 테스트**: 고성능 요구사항이 있는 경우 실제 환경에서 두 모드를 테스트해보세요.
5. **모니터링과 로깅**: 네트워크 문제를 진단할 수 있는 모니터링 도구를 설정하세요.

최종적으로, 네트워크 구성은 애플리케이션의 특성과 팀의 운영 역량에 맞게 결정되어야 합니다. 시작은 Bridge 네트워크로 시작하고, 성능이나 특수 요구사항이 발생할 때 Host 네트워크로의 전환을 고려하는 접근법이 현명한 선택일 수 있습니다.