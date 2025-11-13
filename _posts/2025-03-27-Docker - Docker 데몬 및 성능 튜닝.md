---
layout: post
title: Docker - Docker 데몬 및 성능 튜닝
date: 2025-03-27 21:20:23 +0900
category: Docker
---
# Docker 데몬 설정 및 성능 튜닝

## 0. 빠른 개요 — 왜 데몬 튜닝이 중요한가
- 컨테이너 성능은 **애플리케이션 코드** 뿐 아니라, **데몬(엔진) 구성**·**커널 기능(cgroups/네임스페이스)**·**스토리지/네트워크** 설정에 크게 좌우된다.
- 데몬 설정은 **기본값이 무난**하지만, 고부하/대용량 로그/빅이미지/쿠버네티스 연동/프라이빗 레지스트리 등 **특수 워크로드**에선 **명시적 튜닝**이 필수다.

---

## 1. Docker 데몬이 하는 일 (재정의)
- 컨테이너/이미지/볼륨/네트워크 라이프사이클 관리
- CLI/REST API 수신·실행
- 로깅 드라이버, 스토리지 드라이버 관리
- 리소스 격리(cgroups v1/v2), 네임스페이스, seccomp/AppArmor/SELinux 연동
- (현대 배포판) **containerd**와 연동하여 실제 런타임 조정

---

## 2. 표준 설정 파일과 재시작 방법
리눅스 표준 경로: `/etc/docker/daemon.json` (JSON)

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

적용:

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
# 상태 확인
docker info
journalctl -u docker -n 200 --no-pager
```

> 시스템드로 인자 제어가 필요하면 `sudo systemctl edit docker` 로 drop-in 작성(아래 §8.4 예시).

---

## 3. 성능 핵심: 스토리지/로그/자원 상한/데이터 경로

### 3.1 스토리지 드라이버 = `overlay2`
- 대다수 배포판에서 **overlay2** 권장 (안정/성능 균형).
- XFS를 데이터 디스크로 사용할 경우 **ftype=1**로 포맷된 파일시스템 필요.
- 확인:
```bash
docker info | grep -i "Storage Driver"
```

### 3.2 컨테이너 로그 회전은 필수
기본값은 무제한. 고밀도/장수 컨테이너 환경에서 **디스크 고갈**을 초래.

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "20m",
    "max-file": "10"
  }
}
```

총 로그 최대 사용량 근사:
$$
\text{Total} \approx N_{\text{containers}} \times \text{max-file} \times \text{max-size}
$$
예) 200개 컨테이너 × 10 × 20MB = **~40GB** (헤드룸 포함 디스크 설계).

> 중앙집중형 로깅(Fluent Bit, Vector) 사용 시 컨테이너 로컬 로그는 **작게 회전** + 수집 에이전트로 외부 전송.

### 3.3 기본 ulimits (파일 핸들/프로세스)
대량 연결/파일을 여는 애플리케이션(프록시/DB 프런트)에서 상승 권장.

```json
{
  "default-ulimits": {
    "nofile": { "Name": "nofile", "Soft": 65535, "Hard": 65535 },
    "nproc":  { "Name": "nproc",  "Soft": 65535, "Hard": 65535 }
  }
}
```

### 3.4 공유메모리 `/dev/shm` 기본값 상향
브라우저/ML/대형 런타임은 64MB로는 빈번히 부족.

```json
{ "default-shm-size": "512m" }
```

### 3.5 데이터 루트 전용 디스크로 격리
`/var/lib/docker` → **전용 SSD**(NVMe)로 이전하여 I/O 간섭 제거.

```json
{ "data-root": "/mnt/docker" }
```

---

## 4. 네트워크/이미지 풀 최적화

### 4.1 레지스트리 미러
해외 망/사내 프록시 환경에서 이미지 풀 가속:

```json
{
  "registry-mirrors": [
    "https://mirror.gcr.io"
  ]
}
```

### 4.2 브릿지/주소풀 충돌 방지
다중 호스트/사내망 겹침 방지용 **default address pools**:

```json
{
  "default-address-pools": [
    { "base": "10.20.0.0/16", "size": 24 },
    { "base": "10.30.0.0/16", "size": 24 }
  ]
}
```

고정 브릿지/MTU 지정(특수 상황):

```json
{
  "bip": "172.18.0.1/16",
  "mtu": 1450,
  "iptables": true,
  "ip-forward": true
}
```

### 4.3 IPv6 활성화(선택)
사내 IPv6 라우팅 존재 시:

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "2001:db8:1::/64"
}
```

---

## 5. 쿠버네티스/컨테이너 오케스트레이션과의 정합

### 5.1 cgroup driver 일치
kubelet/CRI가 `systemd`를 사용하면 Docker도 동일:

```json
{ "exec-opts": ["native.cgroupdriver=systemd"] }
```

일치하지 않으면 **Pod 스케줄 실패/자원 격리 문제**.

### 5.2 live-restore (데몬 재시작 중 컨테이너 유지)
데몬 재시작으로 서비스 중단을 피함:

```json
{ "live-restore": true }
```

> dockerd 프로세스가 재시작되어도 **기존 컨테이너는 계속 동작**.

---

## 6. 빌드 성능/안전: BuildKit & 비밀 주입

### 6.1 BuildKit 기본 활성화
```json
{
  "features": { "buildkit": true }
}
```
또는 환경변수:
```bash
export DOCKER_BUILDKIT=1
```

### 6.2 캐시 전략 (레지스트리 캐시)
CI/CD에서 레이어 재사용:

```bash
docker buildx build \
  --cache-from type=registry,ref=yourname/app:buildcache \
  --cache-to   type=registry,ref=yourname/app:buildcache,mode=max \
  -t yourname/app:latest --push .
```

### 6.3 시크릿/SSH 프록시로 안전한 빌드
```dockerfile
# Dockerfile (BuildKit 문법)
# syntax=docker/dockerfile:1.6

RUN --mount=type=secret,id=npm_token \
    bash -lc 'echo "//registry.npmjs.org/:_authToken=$(cat /run/secrets/npm_token)" > ~/.npmrc && npm ci'
```

빌드 시:
```bash
docker build --secret id=npm_token,src=./.secrets/npm_token .
```

---

## 7. 보안/격리: userns-remap, rootless, TLS

### 7.1 사용자 네임스페이스(UID 매핑)
호스트 UID와 컨테이너 UID를 매핑해 **호스트 보호**:

```json
{ "userns-remap": "default" }
```

### 7.2 Rootless Docker (강력 추천 시나리오)
- 일반 사용자 권한에서 dockerd 실행 → 호스트 보호.
- 셋업:
```bash
dockerd-rootless-setuptool.sh install
systemctl --user enable docker --now
```
CLI:
```bash
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
docker info | grep -i rootless
```

### 7.3 원격 접근 보호 (TLS)
**2375/plain 금지**. 2376/TLS + mTLS:
- 서버: `--tlsverify --tlscacert=... --tlscert=... --tlskey=...`
- 클라 Context:
```bash
docker context create prod \
  --docker "host=tcp://prod.example.com:2376,ca=/path/ca.pem,cert=/path/cert.pem,key=/path/key.pem"
```

---

## 8. systemd Drop-in / 커맨드라인 인자

### 8.1 직접 인자
```bash
dockerd \
  --storage-driver=overlay2 \
  --log-driver=json-file \
  --default-shm-size=512m
```

### 8.2 systemd drop-in 파일로 영구화
```bash
sudo systemctl edit docker
```

아래 내용 입력:
```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd \
  --storage-driver=overlay2 \
  --default-shm-size=512m
```

적용:
```bash
sudo systemctl daemon-reexec
sudo systemctl restart docker
```

> 일반적으로는 `daemon.json`로 관리하고, **특정 배포판 버그/옵션**만 drop-in으로 보강.

---

## 9. 모니터링/진단: 실무 루틴

### 9.1 상태/환경
```bash
docker info
docker version
ps aux | grep dockerd
```

### 9.2 로그/이벤트
```bash
journalctl -u docker -f
docker events --since 1h
```

### 9.3 디스크·레이어 용량
```bash
docker system df -v
du -sh /var/lib/docker/* 2>/dev/null
```

### 9.4 성능 계측(간단 부하)
- HTTP 서비스라면 `hey`/`wrk`로 latency/throughput 체크
- I/O: `fio`로 데이터 디스크 성능 확인 (컨테이너 내/외부 비교)

---

## 10. 워크로드별 추천 프로파일 (템플릿)

### 10.1 고부하 웹/프록시 노드
```json
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": { "max-size": "20m", "max-file": "10" },
  "default-ulimits": { "nofile": { "Name": "nofile", "Soft": 65535, "Hard": 65535 } },
  "default-shm-size": "256m",
  "registry-mirrors": ["https://mirror.gcr.io"],
  "live-restore": true,
  "features": { "buildkit": true }
}
```

### 10.2 데이터/분석(브라우저/ML) 노드
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

### 10.3 쿠버네티스 연동 노드
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

## 11. 시나리오 기반 실전 예제

### 11.1 “로그 폭주로 디스크 100%” 대처
1. 긴급: 문제 컨테이너 로그 truncate/회전
{% raw %}
```bash
docker inspect <cid> --format '{{.LogPath}}'
sudo truncate -s 0 /var/lib/docker/containers/<cid>/<cid>-json.log
```
{% endraw %}
2. `daemon.json`에 회전 정책 입력(§3.2) → 재시작
3. 중앙 로깅 경로 구축(Fluent Bit/Vector) + 알람

### 11.2 “이미지 풀 느림/타임아웃”
- 레지스트리 미러 추가(§4.1)
- DNS/프록시/방화벽 확인
- 프라이빗 레지스트리라면 **캐시 프록시**(Harbor/registry mirror) 도입

### 11.3 “Elasticsearch/Chromium 컨테이너가 OOM/Shared Memory 부족”
- `default-shm-size` 상향(§3.4) 또는 **컨테이너별 `--shm-size`**
- cgroups 메모리/스왑 제한도 점검

### 11.4 “쿠버네티스 노드에서 파드가 자주 CrashLoopBackOff”
- `exec-opts`의 cgroupdriver 일치(§5.1)
- 노드 디스크/ inode, iptables 규칙 과부하, 컨테이너 로그 과다 확인

---

## 12. 운영 관리: 청소/프룬/쿼터

### 12.1 시스템 정리
```bash
docker system prune -af --volumes
```
> 파괴적(clean sweep). 운영 노드에선 **예약 점검 윈도우**에만.

### 12.2 이미지/레이어 관리 정책
- 태그 규칙(semver, 날짜, git SHA)
- 오래된 태그 자동 정리(레지스트리 측 Retention + 노드 `prune` 스케줄)

### 12.3 파티션/파일시스템
- **전용 파티션** + **XFS ftype=1** or **ext4** with 충분 inode
- `noatime` 마운트 옵션 고려(읽기 위주 워크로드)

---

## 13. 커널/시스템 파라미터(상황별)
> 애플리케이션 특성에 따라 조정. 무분별한 상향 금지, **측정-조정-검증** 순서.

- inotify:
```bash
sudo sysctl fs.inotify.max_user_watches=1048576
```
- 맵카운트(ES 등):
```bash
sudo sysctl vm.max_map_count=262144
```
- 네트워크 backlog/buffers: 고연결/고치환 프록시에서 조정

`/etc/sysctl.d/docker-tuning.conf` 로 영구화 후 `sysctl --system`.

---

## 14. 검증 플랜(권장 체크리스트)
1. **기준선 확보**: 튜닝 전후 `docker system df -v`, `fio`, `hey/wrk`로 수치 확보
2. **한 번에 한 가지** 변경 → 유의미한 차이인지 확인
3. **부하·장기 테스트**: 24~72시간 soak (로그 회전/메모리 누수/디스크 사용 추세)
4. **알람**: 디스크 사용률/ inode/ 로그 파일 수/ 컨테이너 재시작 횟수/ OOM 이벤트

---

## 15. 종합 예시: 운영 서버용 `daemon.json`
```json
{
  "data-root": "/mnt/docker",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": { "max-size": "20m", "max-file": "10" },
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

## 16. 명령 요약
| 목적 | 명령 |
|---|---|
| 설정 반영 | `systemctl restart docker` |
| 데몬 상태 | `docker info`, `journalctl -u docker` |
| 디스크 현황 | `docker system df -v` |
| 이벤트 추적 | `docker events` |
| 강제 청소 | `docker system prune -af --volumes` |

---

## 17. 참고
- Docker Daemon: `man dockerd`, https://docs.docker.com/engine/reference/commandline/dockerd/
- 리소스 제약: https://docs.docker.com/config/containers/resource_constraints/
- Rootless: https://docs.docker.com/engine/security/rootless/
- BuildKit: https://docs.docker.com/build/buildkit/

---

## 결론
- **스토리지(overlay2)**, **로그 회전**, **자원 상한(nofile/shm)**, **데이터 루트 분리**, **cgroupdriver 일치**, **live-restore**가 **가치/위험 대비 효율이 가장 높은** 기본 튜닝 세트다.
- 그 다음 단계로 **네트워크 주소풀/미러**, **BuildKit 캐시**, **userns-remap/rootless**, **TLS 보호**를 도입하면 **성능·안정·보안**이 고르게 향상된다.
- 모든 튜닝은 **측정→적용→검증** 루프 안에서, **장기 동작(로그/디스크 누적)**까지 관측해야 비로소 실효를 갖는다.
