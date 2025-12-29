---
layout: post
title: Docker - Docker 데몬 및 성능 튜닝
date: 2025-03-27 21:20:23 +0900
category: Docker
---
# Docker 데몬 설정 및 성능 튜닝

## 도입: 왜 데몬 튜닝이 중요한가?

Docker 컨테이너의 성능은 애플리케이션 코드 품질만으로 결정되지 않습니다. Docker 데몬(엔진)의 구성, 커널의 cgroups 및 네임스페이스 기능, 스토리지 드라이버 설정, 네트워크 구성 등이 전체 성능에 지대한 영향을 미칩니다.

대부분의 경우 Docker 데몬의 기본 설정은 무난하게 동작하지만, 다음과 같은 특수한 워크로드 환경에서는 명시적인 튜닝이 필수적입니다:

- **고부하 프로덕션 환경**: 초당 수천 건의 요청을 처리하는 웹 서버나 API 서버
- **대용량 로그 생성 애플리케이션**: 실시간 로깅을 하는 마이크로서비스 아키텍처
- **대형 이미지 작업**: 머신러닝 모델이나 대용량 데이터 처리를 위한 컨테이너
- **쿠버네티스 클러스터 통합**: 컨테이너 오케스트레이션 시스템과의 효율적인 연동
- **프라이빗 레지스트리 사용**: 내부 네트워크에서의 이미지 관리 최적화

이 문서는 Docker 데몬의 성능 튜닝과 운영 최적화를 위한 포괄적인 가이드를 제공합니다.

---

## Docker 데몬 이해: 내부 동작과 역할

Docker 데몬(dockerd)은 컨테이너 생태계의 핵심 구성 요소로 다음과 같은 주요 기능을 수행합니다:

1. **컨테이너 라이프사이클 관리**: 컨테이너의 생성, 시작, 정지, 삭제를 관리
2. **이미지 관리**: 이미지 풀링, 빌드, 태깅, 저장소와의 동기화
3. **볼륨 및 네트워크 관리**: 지속적 스토리지와 네트워크 연결성 제공
4. **API 제공**: Docker CLI 및 외부 도구를 위한 REST API 노출
5. **리소스 격리**: cgroups, 네임스페이스, seccomp, AppArmor/SELinux를 통한 격리
6. **런타임 조정**: containerd와 연동하여 실제 컨테이너 런타임 조율

최신 Docker 배포판에서는 데몬이 containerd와 통합되어 더욱 모듈화된 아키텍처를 제공합니다.

---

## 기본 설정 파일과 관리 방법

### 데몬 설정 파일: `/etc/docker/daemon.json`

Docker 데몬의 기본 설정은 JSON 형식의 `/etc/docker/daemon.json` 파일을 통해 관리됩니다. 이 파일이 존재하지 않으면 Docker는 내부 기본값을 사용합니다.

```json
{
  "data-root": "/mnt/docker-data",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "default-shm-size": "256m",
  "insecure-registries": ["my-registry.local:5000"]
}
```

### 설정 적용과 확인

설정 파일을 변경한 후에는 Docker 데몬을 재시작해야 변경사항이 적용됩니다:

```bash
# 설정 파일 변경 후
sudo systemctl daemon-reload
sudo systemctl restart docker

# 데몬 상태 확인
docker info

# 로그 확인 (문제 발생 시)
journalctl -u docker -n 200 --no-pager
```

### systemd Drop-in 파일 사용

특정 상황에서는 systemd drop-in 파일을 사용하여 데몬 인자를 직접 제어해야 할 수 있습니다:

```bash
# systemd 에디터로 Docker 서비스 수정
sudo systemctl edit docker
```

다음과 같은 내용을 추가할 수 있습니다:
```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd \
  --storage-driver=overlay2 \
  --default-shm-size=512m
```

---

## 성능 튜닝 핵심 영역

### 1. 스토리지 드라이버 최적화

대부분의 최신 Linux 배포판에서는 `overlay2` 스토리지 드라이버를 권장합니다. 이는 안정성과 성능의 균형을 잘 맞추고 있습니다.

**확인 방법:**
```bash
docker info | grep -i "Storage Driver"
```

**XFS 파일시스템 사용 시 주의사항:**
Docker 데이터 디스크로 XFS를 사용할 경우 `ftype=1` 옵션으로 포맷되어 있어야 합니다:
```bash
# 마운트된 XFS 파일시스템의 ftype 확인
xfs_info /mnt/docker-data | grep ftype
```

### 2. 컨테이너 로그 관리

기본적으로 Docker는 컨테이너 로그의 크기 제한을 설정하지 않습니다. 이는 장시간 실행되는 컨테이너나 로그를 많이 생성하는 애플리케이션에서 디스크 공간 고갈로 이어질 수 있습니다.

**로그 회전 설정 예시:**
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "20m",
    "max-file": "10"
  }
}
```

**디스크 사용량 추정:**
전체 로그 사용량은 다음 공식으로 근사적으로 계산할 수 있습니다:
```
총 로그 사용량 ≈ 컨테이너 수 × max-file × max-size
```

예를 들어, 200개의 컨테이너가 각각 10개의 로그 파일(각 20MB)을 유지한다면:
```
200 × 10 × 20MB = 약 40GB
```

**중앙 집중형 로깅과의 통합:**
프로덕션 환경에서는 Fluent Bit, Vector, Logstash와 같은 중앙 로깅 솔루션과 통합하고, 로컬 로그는 최소한으로 유지하는 것이 좋습니다.

### 3. 리소스 제한 설정

#### 파일 디스크립터와 프로세스 제한
대량의 연결이나 파일을 처리하는 애플리케이션(예: 리버스 프록시, 데이터베이스 프론트엔드)에서는 기본 ulimits를 증가시켜야 합니다:

```json
{
  "default-ulimits": {
    "nofile": { "Name": "nofile", "Soft": 65535, "Hard": 65535 },
    "nproc":  { "Name": "nproc",  "Soft": 65535, "Hard": 65535 }
  }
}
```

#### 공유 메모리 크기
브라우저 기반 테스트, 머신러닝 모델, 대형 런타임을 사용하는 컨테이너는 기본 64MB 공유 메모리로는 부족할 수 있습니다:

```json
{ "default-shm-size": "512m" }
```

개별 컨테이너에서도 필요시 오버라이드 가능:
```bash
docker run --shm-size=1g myapp:latest
```

### 4. 데이터 저장소 최적화

Docker의 기본 데이터 경로(`/var/lib/docker`)를 전용 고성능 스토리지로 분리하면 I/O 경합을 줄이고 성능을 향상시킬 수 있습니다:

```json
{ "data-root": "/mnt/docker" }
```

**권장 사항:**
- NVMe SSD와 같은 고성능 스토리지 사용
- `/var` 파티션과 분리하여 독립적인 디스크 사용
- 충분한 inode 수를 가진 파일시스템 선택

---

## 네트워크 성능 최적화

### 1. 레지스트리 미러 설정
해외 레지스트리 접속이 느리거나 프록시 환경에서는 미러를 설정하여 이미지 풀 속도를 향상시킬 수 있습니다:

```json
{
  "registry-mirrors": [
    "https://mirror.gcr.io",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
```

### 2. 네트워크 주소 풀 관리
다중 호스트 환경이나 사내 네트워크와의 주소 충돌을 방지하기 위해 기본 주소 풀을 지정할 수 있습니다:

```json
{
  "default-address-pools": [
    { "base": "10.20.0.0/16", "size": 24 },
    { "base": "10.30.0.0/16", "size": 24 }
  ]
}
```

### 3. 브리지 네트워크 설정
특수한 네트워크 구성이 필요한 경우 브리지 IP와 MTU를 명시적으로 설정할 수 있습니다:

```json
{
  "bip": "172.18.0.1/16",
  "mtu": 1450,
  "iptables": true,
  "ip-forward": true
}
```

### 4. IPv6 지원
IPv6 네트워크 환경에서는 다음과 같이 IPv6를 활성화할 수 있습니다:

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "2001:db8:1::/64"
}
```

---

## 쿠버네티스 통합 최적화

### cgroup 드라이버 일치
쿠버네티스와 Docker가 동일한 cgroup 드라이버를 사용하도록 설정해야 합니다. 대부분의 최신 시스템에서는 `systemd`를 사용합니다:

```json
{ "exec-opts": ["native.cgroupdriver=systemd"] }
```

드라이버가 일치하지 않으면 Pod 스케줄링 실패나 리소스 격리 문제가 발생할 수 있습니다.

### Live Restore 기능
데몬 재시작 중에도 실행 중인 컨테이너를 유지하려면 live-restore 기능을 활성화합니다:

```json
{ "live-restore": true }
```

이 기능을 사용하면 Docker 데몬이 재시작되는 동안에도 기존 컨테이너는 중단 없이 계속 실행됩니다.

---

## 빌드 성능 최적화

### BuildKit 활성화
BuildKit은 Docker의 현대적 빌드 엔진으로, 빌드 성능과 기능을 크게 향상시킵니다:

```json
{
  "features": { "buildkit": true }
}
```

또는 환경 변수를 통해 활성화할 수 있습니다:
```bash
export DOCKER_BUILDKIT=1
```

### 빌드 캐시 전략
CI/CD 파이프라인에서 빌드 성능을 향상시키기 위해 레지스트리 기반 캐시를 사용할 수 있습니다:

```bash
docker buildx build \
  --cache-from type=registry,ref=yourname/app:buildcache \
  --cache-to type=registry,ref=yourname/app:buildcache,mode=max \
  -t yourname/app:latest --push .
```

### 안전한 빌드를 위한 시크릿 관리
비밀 정보를 안전하게 빌드 프로세스에 주입할 수 있습니다:

```dockerfile
# Dockerfile (BuildKit 문법)
# syntax=docker/dockerfile:1.6

RUN --mount=type=secret,id=npm_token \
    bash -lc 'echo "//registry.npmjs.org/:_authToken=$(cat /run/secrets/npm_token)" > ~/.npmrc && npm ci'
```

빌드 시 비밀 정보 전달:
```bash
docker build --secret id=npm_token,src=./.secrets/npm_token .
```

---

## 보안 강화 설정

### 사용자 네임스페이스 재매핑
컨테이너 내부의 root 사용자를 호스트의 비-root 사용자로 매핑하여 보안을 강화할 수 있습니다:

```json
{ "userns-remap": "default" }
```

이 설정은 컨테이너 탈출 공격으로 인한 호스트 시스템 침해 위험을 줄여줍니다.

### Rootless Docker
일반 사용자 권한으로 Docker 데몬을 실행하여 보안을 극대화할 수 있습니다:

```bash
# Rootless Docker 설치
dockerd-rootless-setuptool.sh install

# 사용자 서비스로 Docker 시작
systemctl --user enable docker --now

# 환경 변수 설정
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
```

Rootless Docker는 특히 개발자 워크스테이션이나 CI 환경에 적합합니다.

### 원격 접근 보안
프로덕션 환경에서는 TLS를 통한 암호화된 연결을 사용해야 합니다:

```bash
# TLS 인증서 생성 (간단한 예시)
openssl genrsa -aes256 -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem

# 서버 인증서 생성
openssl genrsa -out server-key.pem 4096
openssl req -subj "/CN=your-server" -sha256 -new -key server-key.pem -out server.csr

# 클라이언트 인증서 생성
openssl genrsa -out key.pem 4096
openssl req -subj "/CN=client" -new -key key.pem -out client.csr
```

Docker Context를 사용한 원격 연결 관리:
```bash
docker context create prod \
  --docker "host=tcp://prod.example.com:2376,ca=/path/ca.pem,cert=/path/cert.pem,key=/path/key.pem"
```

---

## 모니터링과 진단

### 기본 상태 확인
```bash
# Docker 데몬 정보
docker info

# Docker 버전 확인
docker version

# 데몬 프로세스 확인
ps aux | grep dockerd
```

### 로그와 이벤트 모니터링
```bash
# 시스템 로그에서 Docker 관련 메시지 확인
journalctl -u docker -f

# 실시간 Docker 이벤트 모니터링
docker events --since 1h
```

### 디스크 사용량 분석
```bash
# Docker 시스템 디스크 사용량 상세 정보
docker system df -v

# Docker 데이터 디렉토리 크기 확인
du -sh /var/lib/docker/* 2>/dev/null | sort -hr
```

### 성능 벤치마킹
- **네트워크 성능**: `iperf3`, `qperf`를 사용한 컨테이너 간 네트워크 성능 측정
- **I/O 성능**: `fio`를 사용한 컨테이너 내부와 외부의 디스크 성능 비교
- **응답 시간**: `hey`, `wrk`를 사용한 웹 서비스 latency/throughput 측정

---

## 워크로드별 최적화 프로파일

### 고부하 웹/프록시 서버
```json
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": { "max-size": "20m", "max-file": "10" },
  "default-ulimits": { 
    "nofile": { "Name": "nofile", "Soft": 65535, "Hard": 65535 }
  },
  "default-shm-size": "256m",
  "registry-mirrors": ["https://mirror.gcr.io"],
  "live-restore": true,
  "features": { "buildkit": true }
}
```

### 데이터베이스 및 메시징 시스템
```json
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "5" },
  "default-shm-size": "1g",
  "default-ulimits": {
    "nofile": { "Name": "nofile", "Soft": 131072, "Hard": 131072 }
  },
  "exec-opts": ["native.cgroupdriver=systemd"],
  "data-root": "/mnt/docker",
  "live-restore": true
}
```

### 쿠버네티스 워커 노드
```json
{
  "storage-driver": "overlay2",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" },
  "features": { "buildkit": true },
  "live-restore": true,
  "default-address-pools": [
    {"base": "10.123.0.0/16", "size": 24}
  ]
}
```

---

## 일반적인 문제 시나리오와 해결책

### 시나리오 1: 로그 폭주로 인한 디스크 공간 고갈

**증상**: 디스크 사용량이 100%에 도달하여 새로운 컨테이너 생성 실패

**긴급 조치**:

{% raw %}
```bash
# 문제 컨테이너의 로그 파일 확인
docker inspect <container_id> --format '{{.LogPath}}'

# 로그 파일 내용 비우기
sudo truncate -s 0 /var/lib/docker/containers/<container_id>/<container_id>-json.log
```
{% endraw %}

**장기적 해결**:
1. `daemon.json`에 적절한 로그 회전 정책 설정
2. 중앙 집중형 로깅 시스템 도입 (Fluent Bit, Vector 등)
3. 디스크 사용량 모니터링 및 경고 설정

### 시나리오 2: 이미지 풀링 속도 저하 또는 타임아웃

**해결책**:
1. 레지스트리 미러 설정 추가
2. DNS 확인 및 네트워크 프록시 설정 검토
3. 프라이빗 레지스트리의 경우 캐시 프록시(Harbor, registry mirror) 도입
4. 대역폭이 제한된 환경에서는 이미지 압축 옵션 고려

### 시나리오 3: Elasticsearch, Chromium 등 대용량 메모리 애플리케이션 실패

**증상**: OOM(Out Of Memory) 킬러에 의해 컨테이너 종료 또는 공유 메모리 부족 오류

**해결책**:
1. `default-shm-size` 증가 또는 컨테이너별 `--shm-size` 옵션 사용
2. cgroups 메모리 및 스왑 제한 적절히 설정
3. 호스트 시스템의 메모리 압박 확인 및 필요시 증설

### 시나리오 4: 쿠버네티스에서 Pod가 빈번하게 CrashLoopBackOff

**해결책**:
1. Docker와 kubelet의 cgroup 드라이버 일치 확인
2. 노드 디스크 공간 및 inode 사용량 확인
3. iptables 규칙 과부하 검토
4. 컨테이너 로그 회전 설정 검증

---

## 시스템 정리와 유지보수

### 정기적인 시스템 정리
```bash
# 사용하지 않는 모든 Docker 객체 제거 (주의: 파괴적 작업)
docker system prune -af --volumes
```

운영 환경에서는 이 명령을 신중하게 사용해야 하며, 일반적으로는 예약된 유지보수 기간에 실행하는 것이 좋습니다.

### 이미지 레이어 관리 정책
1. **태그 규칙 수립**: Semantic Versioning, 날짜, Git 커밋 SHA 등을 활용한 일관된 태그 전략
2. **오래된 태그 자동 정리**: 레지스트리 보존 정책과 노드 정리 스케줄 조합
3. **베이스 이미지 정기 업데이트**: 보안 패치와 성능 향상을 위한 주기적 업데이트

### 파일시스템 최적화
- **전용 파티션 사용**: Docker 데이터를 독립적인 파티션에 저장
- **적절한 파일시스템 선택**: XFS(ftype=1) 또는 충분한 inode를 가진 ext4
- **마운트 옵션 최적화**: 읽기 위주 워크로드의 경우 `noatime` 옵션 고려

---

## 커널 파라미터 튜닝

애플리케이션 특성에 따라 일부 커널 파라미터를 조정해야 할 수 있습니다. 모든 변경은 측정-조정-검증의 순서로 접근해야 합니다.

### 일반적인 튜닝 파라미터

**inotify 제한 증가** (파일 시스템 감시를 많이 사용하는 애플리케이션):
```bash
sudo sysctl -w fs.inotify.max_user_watches=1048576
```

**가상 메모리 맵 카운트 증가** (Elasticsearch 등):
```bash
sudo sysctl -w vm.max_map_count=262144
```

**네트워크 백로그 및 버퍼** (고연결/고처리량 프록시):
```bash
sudo sysctl -w net.core.somaxconn=65535
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=65535
```

### 영구적 적용
`/etc/sysctl.d/docker-tuning.conf` 파일에 설정을 추가하여 재부팅 후에도 유지되도록 할 수 있습니다:
```bash
# 설정 파일 생성
echo "fs.inotify.max_user_watches=1048576" | sudo tee /etc/sysctl.d/docker-tuning.conf
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.d/docker-tuning.conf

# 설정 적용
sudo sysctl --system
```

---

## 운영 검증 계획

효과적인 튜닝을 위해 다음과 같은 검증 계획을 수립하는 것이 좋습니다:

1. **기준선 확립**: 튜닝 전후의 성능 메트릭 수집
   - `docker system df -v`로 디스크 사용량 측정
   - `fio`로 I/O 성능 벤치마킹
   - `hey`/`wrk`로 네트워크 응답 시간 측정

2. **점진적 변경**: 한 번에 하나의 설정만 변경하고 효과를 측정

3. **부하 및 장기 테스트**: 24-72시간의 지속적 부하 테스트로 안정성 검증
   - 로그 회전 기능 검증
   - 메모리 누수 확인
   - 디스크 사용량 추이 모니터링

4. **모니터링 및 알림 설정**:
   - 디스크 사용률 모니터링
   - inode 사용량 추적
   - 로그 파일 수 감시
   - 컨테이너 재시작 횟수 모니터링
   - OOM 이벤트 알림

---

## 종합 운영 설정 예시

다음은 프로덕션 서버를 위한 종합적인 `daemon.json` 설정 예시입니다:

```json
{
  "data-root": "/mnt/docker",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": { 
    "max-size": "20m", 
    "max-file": "10" 
  },
  "default-ulimits": {
    "nofile": { "Name": "nofile", "Soft": 65535, "Hard": 65535 }
  },
  "default-shm-size": "512m",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "live-restore": true,
  "registry-mirrors": ["https://mirror.gcr.io"],
  "features": { "buildkit": true },
  "default-address-pools": [
    { "base": "10.40.0.0/16", "size": 24 }
  ]
}
```

---

## 결론

Docker 데몬 설정과 성능 튜닝은 컨테이너 기반 인프라의 안정성, 보안, 성능을 결정하는 핵심 요소입니다. 효과적인 튜닝을 위해 다음 원칙을 기억하세요:

1. **가장 중요한 기본 튜닝**: 스토리지 드라이버(overlay2), 로그 회전, 자원 상한(nofile/shm), 데이터 루트 분리, cgroup 드라이버 일치, live-restore는 투자 대비 효과가 가장 큰 핵심 설정입니다.

2. **보안 강화**: 사용자 네임스페이스 재매핑, Rootless Docker, TLS 보호를 통해 보안을 강화하면서도 성능 저하를 최소화할 수 있습니다.

3. **워크로드 특성 고려**: 애플리케이션의 특성에 맞춰 네트워크 설정, 빌드 최적화, 커널 파라미터를 조정해야 합니다.

4. **측정 기반 접근**: 모든 튜닝은 실제 메트릭을 기반으로 해야 합니다. 변경 전후의 성능을 정량적으로 측정하고 비교하세요.

5. **장기적 관점**: 단기 성능뿐만 아니라 로그 누적, 디스크 사용량 추이, 장기 실행 안정성까지 고려한 종합적인 접근이 필요합니다.

6. **자동화와 모니터링**: 설정 변경을 자동화하고, 모든 변경사항을 모니터링하여 예상치 못한 문제를 조기에 발견하세요.

성공적인 Docker 데몬 튜닝은 한 번에 완성되는 것이 아닌, 지속적인 측정, 조정, 검증의 순환 과정을 통해 점진적으로 이루어집니다. 조직의 특정 요구사항과 워크로드 패턴을 이해하고, 이 가이드를 출발점으로 삼아 최적의 구성을 찾아나가시기 바랍니다.