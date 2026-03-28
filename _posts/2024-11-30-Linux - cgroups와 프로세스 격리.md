---
layout: post
title: Linux - cgroups와 프로세스 격리
date: 2024-11-30 19:20:23 +0900
category: Linux
---
# cgroups와 프로세스 격리

## 왜 cgroups, namespace, seccomp인가

리눅스에서 프로세스 격리와 리소스 관리는 세 가지 핵심 기술의 조합으로 이루어진다.

- **cgroups**: CPU, 메모리, I/O, 프로세스 수 등의 **리소스 사용량**을 그룹 단위로 제한하고 계측한다.
- **namespaces**: PID, 네트워크, 마운트, 호스트명 등 **시스템 자원의 가시성**을 분리한다.
- **seccomp**: 프로세스가 호출할 수 있는 **시스템콜**을 필터링한다.

이 세 기술이 결합되어 컨테이너, 샌드박스, 멀티테넌트 서버의 격리와 안정성을 제공한다.

---

## cgroups 버전 비교

| 항목 | cgroups v1 | cgroups v2 |
|------|------------|------------|
| 계층 구조 | 컨트롤러별 개별 계층 | **단일 통합 계층** |
| 주요 컨트롤러 | cpu, memory, blkio, pids, devices 등 | cpu, memory, io, pids, cpuset 등 |
| 인터페이스 | 컨트롤러마다 파일명 상이 (예: `memory.limit_in_bytes`) | **일관된 명명 규칙** (예: `memory.max`, `cpu.max`) |
| 운영 복잡도 | 혼재 시 관리 어려움 | 구조 단순, 현대 배포판 기본 |

> 새로운 설계는 **cgroups v2**를 기준으로 한다. 구형 시스템에서는 v1에 대한 이해도 필요하다.

---

## cgroups v2 핵심 파일

| 컨트롤러 | 핵심 파일 | 설명 |
|----------|-----------|------|
| CPU | `cpu.max` | 쿼터/주기 (예: `50000 100000` → 50%) |
| | `cpu.weight` | 상대적 가중치 (1~10000, 기본 100) |
| 메모리 | `memory.max` | 하드 상한 (초과 시 OOM) |
| | `memory.high` | 소프트 압박선 (초과 시 회수 시도) |
| | `memory.current` | 현재 사용량 |
| I/O | `io.max` | 디바이스별 bps/iops 상한 |
| | `io.weight` | I/O 가중치 |
| 프로세스 | `pids.max` | 최대 프로세스 수 |
| | `pids.current` | 현재 수 |

---

## cgroups v2 실습

### 1. 제한 그룹 만들기

```bash
sudo mkdir /sys/fs/cgroup/work
```

### 2. 리소스 제한 설정

```bash
# CPU 50% (50ms/100ms)
echo "50000 100000" | sudo tee /sys/fs/cgroup/work/cpu.max

# 메모리 상한 200MB, 스왑 상한 100MB
echo 200M | sudo tee /sys/fs/cgroup/work/memory.max
echo 100M | sudo tee /sys/fs/cgroup/work/memory.swap.max

# 최대 프로세스 100개
echo 100 | sudo tee /sys/fs/cgroup/work/pids.max
```

### 3. 프로세스를 그룹에 추가

```bash
# 현재 셸을 그룹에 포함
echo $$ | sudo tee /sys/fs/cgroup/work/cgroup.procs
```

### 4. 상태 확인

```bash
cat /sys/fs/cgroup/work/memory.current
cat /sys/fs/cgroup/work/memory.events   # OOM 발생 여부
cat /sys/fs/cgroup/work/cpu.stat        # throttle 통계
cat /sys/fs/cgroup/work/pids.current
```

### 메모리 하드 한도와 소프트 압박

- `memory.max`: **하드 상한** – 넘으면 즉시 OOM으로 프로세스 종료
- `memory.high`: **소프트 압박선** – 넘으면 커널이 캐시 등을 회수하려 시도. 그래도 부족하면 결국 OOM

운영 시에는 `memory.high`를 먼저 걸어 캐시부터 줄이고, 최후 방어선으로 `memory.max`를 설정하는 것이 좋다.

---

## systemd와 cgroups 통합

systemd는 모든 유닛(서비스, 스코프, 슬라이스)을 cgroup으로 표현한다.

### 임시 제한 실행

```bash
# CPU 80%, 메모리 200MB로 제한된 셸 실행
systemd-run -t --property=CPUQuota=80% --property=MemoryMax=200M bash
```

### 서비스 유닛에 제한 선언

```ini
# /etc/systemd/system/limited.service
[Service]
ExecStart=/usr/bin/myapp
CPUQuota=50%
MemoryMax=1G
IOReadBandwidthMax=/dev/sda 10M
AllowedCPUs=0-3
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now limited.service
```

### 모니터링

```bash
systemd-cgtop                 # cgroup별 리소스 사용량
systemctl show limited.service -p ControlGroup
cat /sys/fs/cgroup/<경로>/memory.current
```

---

## 네임스페이스: 가시성 분리

| 네임스페이스 | 격리 대상 |
|--------------|-----------|
| `pid` | 프로세스 ID 트리 |
| `net` | 네트워크 인터페이스, 포트 |
| `mnt` | 마운트 포인트 |
| `uts` | 호스트명 |
| `user` | UID/GID 매핑 |
| `ipc` | 공유 메모리, 세마포어 |

### 직접 실험

```bash
# 격리된 환경에서 셸 실행
sudo unshare --pid --net --mount --uts --ipc --user --map-root-user bash

# 새 셸 내부에서
hostname isolated
ip link
```

### 다른 프로세스 네임스페이스 진입

```bash
sudo nsenter --target <PID> --pid --net --mount bash
```

---

## seccomp: 시스템콜 필터링

컨테이너 런타임(Docker, Podman)은 기본적으로 seccomp 프로파일을 적용하여 불필요한 시스템콜을 차단한다.

```bash
# seccomp 프로파일을 지정해 컨테이너 실행
docker run --security-opt seccomp=/path/to/profile.json alpine
```

운영 환경에서는 기본 프로파일로도 충분한 경우가 많지만, 보안이 중요한 서비스는 더 엄격한 프로파일을 적용할 수 있다.

---

## 실전 시나리오

### 시나리오 1: 빌드 작업 제한

```bash
systemd-run --scope -p CPUQuota=200% -p MemoryMax=2G make -j
```

### 시나리오 2: 로그 서비스 I/O 제한

```bash
systemd-run --scope -p IOWriteBandwidthMax=/dev/nvme0n1 5M ./loggy
```

### 시나리오 3: cgroups v2 수동으로 빌드 그룹 생성

```bash
sudo mkdir /sys/fs/cgroup/build
echo "+cpu +memory +pids" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
echo "200000 100000" | sudo tee /sys/fs/cgroup/build/cpu.max   # 200%
echo 2G | sudo tee /sys/fs/cgroup/build/memory.max
echo $$ | sudo tee /sys/fs/cgroup/build/cgroup.procs
make -j
```

---

## CPU 쿼터 계산

CPU 쿼터가 $T$ 주기 동안 $Q$만큼 허용될 때 최대 사용률은:
$$
\text{CPU 사용률}_{\max} = \frac{Q}{T} \times 100\ \%
$$
예: `cpu.max = "50000 100000"` → 50%.

---

## 흔한 함정과 해결

| 문제 | 해결 |
|------|------|
| 권한 없음 | `/sys/fs/cgroup` 쓰기는 root 필요. delegation이 필요한 경우 적절히 구성 |
| v1/v2 혼용 | 현대 시스템은 v2로 통일. 혼용은 피할 것 |
| cpuset 설정 안 됨 | 부모 그룹에 먼저 `cpuset.cpus`를 설정해야 자식에 반영 |
| I/O 제한 적용 안 됨 | 디바이스 메이저:마이너 번호 정확히 확인 (`lsblk`) |
| OOM 발생 시 전체 그룹 종료 | v2에서 `memory.oom.group=1`로 설정하면 그룹 전체 종료 가능 |

---

## 관측 도구

```bash
# PSI (Pressure Stall Information)
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io

# cgroup 이벤트
cat /sys/fs/cgroup/<그룹>/memory.events
cat /sys/fs/cgroup/<그룹>/cgroup.events
```

---

## 핵심 명령어 요약

| 작업 | 명령어 / 파일 |
|------|----------------|
| cgroup v2 그룹 생성 | `mkdir /sys/fs/cgroup/그룹명` |
| CPU 제한 | `echo "50000 100000" > cpu.max` |
| 메모리 제한 | `echo 200M > memory.max` |
| 프로세스 추가 | `echo $$ > cgroup.procs` |
| systemd 임시 제한 | `systemd-run -t --property=CPUQuota=... --property=MemoryMax=... bash` |
| 서비스 제한 설정 | `/etc/systemd/system/서비스.service`에 `CPUQuota=`, `MemoryMax=` 등 추가 |
| 네임스페이스 격리 | `unshare --pid --net --mount bash` |
| nsenter로 진입 | `nsenter --target PID --pid --net bash` |
| seccomp 확인 | `docker run --security-opt seccomp=...` |

---

## 결론

cgroups, 네임스페이스, seccomp는 리눅스에서 프로세스를 격리하고 자원을 제어하는 핵심 기술이다.

- **cgroups**는 리소스 사용량을 그룹 단위로 제한한다. v2 + systemd 조합이 현대적인 표준이다.
- **네임스페이스**는 프로세스가 보는 시스템 자원의 뷰를 분리한다.
- **seccomp**는 시스템콜 접근을 제한하여 공격 표면을 줄인다.

운영 시에는 다음을 기억하자:
- `memory.high`로 캐시부터 압박, `memory.max`로 최종 상한 설정
- CPU는 쿼터(`cpu.max`)로 할당량을, 가중치(`cpu.weight`)로 비율을 조정
- I/O 제한은 디바이스별로 bps/iops를 지정
- systemd 유닛에 제한을 선언하면 cgroup이 자동으로 구성된다

이러한 기술을 적절히 활용하면 멀티테넌트 환경에서의 공정한 자원 분배, 컨테이너의 안정성, 장애 격리를 달성할 수 있다.