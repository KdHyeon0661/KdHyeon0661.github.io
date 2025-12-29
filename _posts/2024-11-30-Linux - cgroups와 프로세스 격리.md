---
layout: post
title: Linux - cgroups와 프로세스 격리
date: 2024-11-30 19:20:23 +0900
category: Linux
---
# cgroups와 프로세스 격리

## 왜 cgroups, namespace, seccomp인가

리눅스에서 프로세스 격리와 리소스 관리는 세 가지 핵심 기술의 조합으로 이루어집니다:

- **cgroups**: CPU, 메모리, IO, PIDs 등의 **리소스 사용량 한계**를 **그룹 단위**로 강제하고 계측
- **namespaces**: PID, Network, Mount, UTS, User, IPC, Cgroup 등의 **시스템 자원 가시성**을 분리
- **seccomp**: **시스템콜 접근**을 필터링하여 프로세스의 실행 가능한 작업 제한

이 세 가지 기술이 결합되어 컨테이너, 샌드박스, 멀티테넌트 서버의 **격리·안전·예측가능성**을 제공합니다.

---

## cgroups 전체 지도 — v1 vs v2, 컨트롤러

### 버전 차이 요약

| 항목 | cgroups v1 | cgroups v2 |
|---|---|---|
| 계층 구조 | 컨트롤러별 개별 계층 | **단일 통합 계층** |
| 컨트롤러 | cpu, cpuacct, cpuset, memory, blkio, pids, freezer, devices 등 | **cpu, io, memory, pids, cpuset, rdma, misc** |
| 인터페이스 | 다양한 파일명 (예: `memory.limit_in_bytes`) | **일관된 명명 규칙** (예: `memory.max`, `cpu.max`) |
| 트리 규칙 | 혼합 가능, 복잡 | 부모-자식 규율 엄격, **위임(delegation)** 설계 용이 |
| 구현 난이도 | 도구/배포판 따라 상이 | 현대적 배포판 기본(컨테이너 런타임도 주류) |

> 새로운 설계와 운영은 **cgroups v2를 권장**합니다. 레거시 시스템(특히 구버전 커널/도구)에서는 v1 문법에 대한 이해도 필요합니다.

### 대표 컨트롤러 및 핵심 파일 (v2 기준)

- **CPU**: `cpu.max`(쿼터/주기), `cpu.weight`(비율 가중치), `cpu.stat`
- **Memory**: `memory.max`(경성 상한), `memory.high`(소프트 압박선), `memory.swap.max`, `memory.min/low`, `memory.current`, `memory.events`
- **IO(blk)**: `io.max`(bps/iops 상한), `io.weight`, `io.stat`
- **PIDs**: `pids.max`, `pids.current`
- **cpuset**: NUMA/CPU 핀닝 (`cpuset.cpus`, `cpuset.mems`)
- **misc/rdma**: 특수 자원 쿼터

> v1에서 자주 보던 `memory.limit_in_bytes`, `blkio.throttle.*`, `cpu.shares` 등은 v2에서 **의미가 대응**되거나 통합되었습니다.

---

## cgroups v2: 실습으로 이해

### 마운트 및 준비

```bash
# (대부분의 최신 배포판은 이미 v2 마운트됨. 확인:)
mount | grep cgroup2 || sudo mount -t cgroup2 none /sys/fs/cgroup

# 루트 cgroup에 하위 그룹을 만들 준비: subtree에 컨트롤러 활성화
echo "+cpu +memory +io +pids" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
```

### 메모리 상한 + CPU 쿼터 설정

```bash
# 워크로드 그룹 생성
sudo mkdir /sys/fs/cgroup/work

# work 하위에서 사용할 컨트롤러를 부모에서 허가
echo "+cpu +memory +io +pids" | sudo tee /sys/fs/cgroup/cgroup.subtree_control

# 제한 설정
echo 200M       | sudo tee /sys/fs/cgroup/work/memory.max        # OOM 발생 상한
echo 100M       | sudo tee /sys/fs/cgroup/work/memory.swap.max   # swap 상한
echo "50000 100000" | sudo tee /sys/fs/cgroup/work/cpu.max       # 50ms/100ms → 50% 쿼터
echo 100        | sudo tee /sys/fs/cgroup/work/pids.max          # 최대 PID 수

# 프로세스 편입(현재 셸 PID)
echo $$ | sudo tee /sys/fs/cgroup/work/cgroup.procs
```

> `cpu.max` 형식: `"<quota> <period>"`. 위 예시는 **주기 100ms 중 50ms만 실행 → 50%**입니다.
> 가중치 기반 공정 배분은 `cpu.weight`(1~10000, 기본 100)를 사용합니다.

### 관측 포인트

```bash
cat /sys/fs/cgroup/work/memory.current
cat /sys/fs/cgroup/work/memory.events     # oom, oom_kill 카운트
cat /sys/fs/cgroup/work/cpu.stat          # usage_usec, throttled_usec, nr_throttled
cat /sys/fs/cgroup/work/io.stat
cat /sys/fs/cgroup/work/pids.current
```
- **OOM** 발생 여부, CPU **throttle** 여부, IO 사용 상한 도달 여부 등을 숫자로 확인할 수 있습니다.

### memory.high vs memory.max

- `memory.max`: **하드 상한**. 넘으면 즉시 OOM 대상
- `memory.high`: **소프트 압박선**. 넘어가면 **메모리 회수 압박**(쓰로틀링/리클레임) 후에도 안 줄면 결국 OOM 가능
- **튜닝 팁**: 배경작업/캐시가 많은 워크로드에 `memory.high`를 먼저 설정하여 캐시부터 줄이게 하고, 최후 방어선으로 `memory.max` 설정

---

## cgroups v1: 레거시 인터페이스 빠르게 익히기

### 메모리 제한 (v1)

```bash
sudo mkdir -p /sys/fs/cgroup/memory/mygroup
echo 100000000 | sudo tee /sys/fs/cgroup/memory/mygroup/memory.limit_in_bytes
echo $$          | sudo tee /sys/fs/cgroup/memory/mygroup/tasks
```

### CPU 공유/쿼터 (v1)

```bash
sudo mkdir -p /sys/fs/cgroup/cpu/mycg
echo 1024 | sudo tee /sys/fs/cgroup/cpu/mycg/cpu.shares     # 비율
echo 50000 | sudo tee /sys/fs/cgroup/cpu/mycg/cpu.cfs_quota_us
echo 100000 | sudo tee /sys/fs/cgroup/cpu/mycg/cpu.cfs_period_us
```

### 블록 IO (v1, blkio)

```bash
# 특정 디바이스(메이저:마이너)에 iops/bps 상한
echo "8:0 10485760" | sudo tee /sys/fs/cgroup/blkio/mycg/blkio.throttle.read_bps_device
```

> v1/v2 혼재 환경은 **컨테이너 런타임**이 알아서 처리하지만, 수동 튜닝 시 파일명이 다름에 유의하세요.

---

## systemd와 cgroups — services/slices/scopes의 세계

systemd는 **모든 유닛을 cgroup으로 표현**합니다.

### 빠른 시작: 제한이 걸린 임시 유닛 실행

```bash
# 메모리 200M, CPU 80%로 제한된 쉘 실행
systemd-run -t --property=MemoryMax=200M --property=CPUQuota=80% bash
```

### 서비스 유닛에 자원정책 선언

```ini
# /etc/systemd/system/my-limited.service

[Unit]
Description=Limited Worker

[Service]
ExecStart=/usr/bin/sleep 10000
MemoryMax=150M
CPUQuota=50%
IOReadBandwidthMax=/dev/vda 10M
IOWriteBandwidthMax=/dev/vda 5M
# cpuset 예시(systemd ≥ 247, 커널 지원 필요)
AllowedCPUs=0-3
AllowedMemoryNodes=0

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now my-limited.service
systemctl status my-limited.service
```

### 슬라이스/스코프

- **slices**: 대역 클래스(예: `system.slice`, `user.slice`, `machine.slice`)로 **예산** 나누기
- **scopes**: 외부에서 이미 시작된 프로세스 묶기(예: `systemd-run --scope -p MemoryMax=... <cmd>`)

### 어디에 생성되었는지 확인

```bash
systemd-cgls                  # cgroup 트리 요약
systemd-cgtop                 # 유닛별 소비량 top
systemctl show my-limited.service -p ControlGroup
cat /sys/fs/cgroup/<unit path>/memory.current
```

---

## 네임스페이스: 무엇을 보게 할 것인가

### 종류와 용도

| 네임스페이스 | 격리 객체 | 예시 도구 |
|---|---|---|
| **pid** | PID 트리 | `unshare --pid` |
| **net** | NIC/라우팅/포트 | `ip netns`, `unshare --net` |
| **mnt** | 마운트 트리, 전파 | `unshare --mount` |
| **uts** | hostname/domain | `unshare --uts` |
| **ipc** | shm/msg/sem | `unshare --ipc` |
| **user** | UID/GID 맵핑 | `unshare --user` |
| **cgroup** | cgroup 계층 가시성 | `unshare --cgroup` |

### 실습: 완전 격리된 미니 컨테이너 셸

```bash
sudo unshare --fork --pid --mount --uts --ipc --net --user --map-root-user bash -l
# 새 셸 내부:
hostname mini
mount -t proc proc /proc
ip link add veth0 type veth peer name veth1   # netns 단독이라면 더 설정 필요
```

### nsenter: 다른 프로세스의 네임스페이스로 진입

```bash
# pid 1234가 가진 네임스페이스로 들어가 bash 실행
sudo nsenter --target 1234 --mount --uts --ipc --net --pid bash
```

### 마운트 전파(propagation)

- `shared`/`slave`/`private`/`rshared` 등 전파 플래그로 **호스트↔컨테이너** 마운트 영향 범위 결정
```bash
mount --make-rshared /
```

---

## seccomp: 시스템콜 필터로 공격면 축소

### 개념

- BPF(또는 eBPF) 기반 **시스템콜 허용/거부** 정책
- 컨테이너 런타임(Docker/K8s)은 기본 프로파일로 광범위 차단

### 간단한 실행 예(Docker)

```bash
# Allowlist 기반 최소 syscalls(예시 json)를 지정해 컨테이너 실행
docker run --security-opt seccomp=/path/seccomp-min.json alpine:3.20
```

> 네이티브로는 `seccomp()`(C) 또는 `libseccomp`로 규칙 설치
> 실무에서는 런타임이 제공하는 **프로파일** 튜닝이 일반적

---

## 실전 시나리오

### "빌드 작업은 CPU 200%·메모리 2GiB 이내로 제한"
#### cgroups v2 수동 설정

```bash
sudo mkdir /sys/fs/cgroup/build
echo "+cpu +memory +pids" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
echo "200000 100000" | sudo tee /sys/fs/cgroup/build/cpu.max  # 200%
echo $((2*1024*1024*1024)) | sudo tee /sys/fs/cgroup/build/memory.max
echo $$ | sudo tee /sys/fs/cgroup/build/cgroup.procs
make -j
```
#### systemd-run 사용

```bash
systemd-run --scope -p CPUQuota=200% -p MemoryMax=2G make -j
```

### "백그라운드 캐시 데몬은 메모리 압박 우선 적용"

```bash
systemd-run --scope -p MemoryHigh=1G -p MemoryMax=2G -p CPUWeight=25 ./cached
```
- 캐시층부터 리클레임 압박(`memory.high`) → 전체 한도(`memory.max`)

### "IO 가디언: 로그 폭주 서비스의 디스크 쓰기 제한"

```bash
systemd-run --scope -p IOWriteBandwidthMax=/dev/nvme0n1 5M ./loggy
```
- v2 io 컨트롤러를 systemd 속성으로 간결히 설정

---

## 컨테이너 런타임과의 연결

### Docker

```bash
docker run --cpus=1.5 --memory=512m --pids-limit=128 --memory-swap=512m \
  --cpuset-cpus=0-3 --device-read-bps /dev/sda:10mb \
  -it ubuntu:24.04 bash
```
- 옵션이 내부적으로 cgroup v2 파일에 매핑됩니다
- `--security-opt seccomp=…`, `--cap-drop=…`로 커널 표면 축소

### Kubernetes(참고)

- `resources.requests/limits` → 노드의 cgroup 정책으로 투영
- Burstable/Guaranteed QoS 클래스가 cgroup weight/limit에 영향

---

## 관측·디버깅·안정성

### PSI(Pressure Stall Information)

리소스 압박을 퍼센트로 노출하여 **스로틀/경합**을 수치화합니다
```bash
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
```

### cgroup 이벤트 파일

```bash
cat /sys/fs/cgroup/<grp>/memory.events
cat /sys/fs/cgroup/<grp>/cgroup.events      # frozen, populated 등
```

### OOM의 이해

- **cgroup OOM**: 그룹 내부에서만 프로세스 종료
- **시스템 OOM**: 전역 메모리 부족
- `memory.oom.group=1`(v2)로 **그룹 단위 종료**를 유도해 **부분적 누수** 방지

### 흔한 함정·해결

- **권한**: `/sys/fs/cgroup` 쓰기는 root 필요. delegation 시 `cgroup.procs`/`cgroup.subtree_control` 권한 설계
- **혼합 계층**: v1과 v2 혼용 피하기
- **cpuset**: 부모가 먼저 `cpuset.cpus/mems` 설정해야 자식에 반영 가능
- **IO 제한**: 디바이스 메이저:마이너 번호 정확히 지정
- **스왑**: v2의 `memory.swap.max`로 별도 상한 명시, 예측성 확보

---

## 자동화 스니펫

### 워크로드를 묶어 실행하는 스크립트(v2)

```bash
#!/usr/bin/env bash

set -euo pipefail
GRP="/sys/fs/cgroup/jobs/${1:?group}"
shift
sudo mkdir -p "$GRP"
# 부모 subtree 활성화 보장(필요 시 상위에서 echo)
echo "+cpu +memory +io +pids" | sudo tee /sys/fs/cgroup/cgroup.subtree_control >/dev/null
echo "100000 100000" | sudo tee "$GRP/cpu.max"       >/dev/null   # 100% (=1 vCPU)
echo 1G               | sudo tee "$GRP/memory.max"   >/dev/null
echo 512M             | sudo tee "$GRP/memory.high"  >/dev/null
echo 256              | sudo tee "$GRP/pids.max"     >/dev/null
sudo bash -c "echo $$ > '$GRP/cgroup.procs'"
exec "$@"
```

### systemd 서비스 템플릿

```ini
# /etc/systemd/system/worker@.service

[Unit]
Description=Worker %i with limits

[Service]
ExecStart=/usr/local/bin/worker --team=%i
CPUQuota=150%
CPUWeight=200
MemoryHigh=1G
MemoryMax=2G
IOReadBandwidthMax=/dev/nvme0n1 20M
IOWriteBandwidthMax=/dev/nvme0n1 10M
TasksMax=512
AmbientCapabilities=
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
LockPersonality=yes

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now worker@alpha worker@beta
```

---

## CPU 쿼터 계산

CPU 쿼터가 `T` 주기 동안 `Q`만큼 허용될 때 **최대 사용률**은:
$$
\text{CPU Utilization}_{\max} = \frac{Q}{T}\times 100\ \%.
$$
예: `cpu.max = "50000 100000"`인 경우:
$$
\frac{50\text{ ms}}{100\text{ ms}} \times 100 = 50\%.
$$

---

## 명령·파일 요약표

| 범주 | v2 파일/명령 | 요지 |
|---|---|---|
| CPU | `cpu.max`, `cpu.weight`, `cpu.stat` | 쿼터/주기, 가중치, 쓰로틀 통계 |
| Memory | `memory.max/high/low/min`, `memory.swap.max`, `memory.events/current` | 상·중·하한, 스왑, OOM 이벤트 |
| IO | `io.max`, `io.weight`, `io.stat` | 디바이스별 bps/iops 상한, 가중치 |
| PIDs | `pids.max/current` | 포크 폭주 차단 |
| cpuset | `cpuset.cpus/mems` | NUMA/CPU 고정 |
| PSI | `/proc/pressure/*` | 압박 관측 |
| systemd | `systemd-run`, unit props | Quota/Max/Weight/IO* |
| 네임스페이스 | `unshare`, `nsenter` | 보이는 세계 분리 |
| seccomp | 런타임 프로파일 | 시스템콜 표면 축소 |

---

## 결론

리눅스의 cgroups, 네임스페이스, seccomp는 현대적인 프로세스 격리와 리소스 관리를 위한 강력한 기술 조합입니다. 효과적인 운영을 위해 다음 원칙들을 기억하세요:

**기술적 조합의 이해**:
- **cgroups v2 + systemd**는 현대 리눅스의 표준 리소스 정책 프레임워크입니다
- **네임스페이스**로 "보이는 세계"를 분리하고, **cgroups**로 "사용 가능한 양"을 규정하며, **seccomp**로 "수행 가능한 행위"를 제한하여 컨테이너 수준의 격리와 예측 가능성을 달성합니다

**체계적인 운영 접근법**:
1. **관측 우선**: PSI, cgroup.events, cpu.stat 등을 활용한 수치 기반 모니터링
2. **점진적 정책 적용**: 새로운 워크로드는 기본 상한(최소한의 Max/Quota/Tasks/PIDs)부터 시작
3. **서비스 SLO 기반 튜닝**: 하향식으로 완화하며 성능과 안정성 균형 찾기
4. **자동화 통합**: systemd 유닛과 스크립트를 활용한 일관된 정책 적용

**실무 중심의 문제 해결**:
- 메모리 문제는 `memory.high`와 `memory.max`의 차이를 이해하고 적절히 활용
- CPU 할당은 쿼터(`cpu.max`)와 가중치(`cpu.weight`)를 상황에 맞게 조합
- IO 제한은 디바이스별 특성을 고려한 bps/iops 설정
- 네임스페이스 격리는 보안 경계와 성능 요구사항의 균형에서 설계

이러한 기술들을 효과적으로 활용하면 멀티테넌트 환경에서의 리소스 분배, 컨테이너 기반 배포, 서비스 안정성 향상 등 다양한 운영 요구사항을 충족시킬 수 있습니다. 각 환경의 특성에 맞게 적절히 조정하고 지속적으로 최적화하는 것이 장기적인 성공의 핵심입니다.