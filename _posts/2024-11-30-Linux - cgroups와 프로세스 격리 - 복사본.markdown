---
layout: post
title: Linux - cgroups와 프로세스 격리
date: 2024-11-29 19:20:23 +0900
category: Linux
---
# 리눅스 29편: 🧱 cgroups와 프로세스 격리

리눅스는 `cgroups`, `namespace`, `seccomp` 등 다양한 커널 기능을 통해 **프로세스 리소스를 제어하고 격리**할 수 있습니다.  
이는 컨테이너, 보안 샌드박스, 시스템 리소스 제어의 핵심 기술입니다.

---

## 1. 🔒 cgroups (Control Groups)

`cgroups`는 특정 **프로세스 그룹에 대해 CPU, 메모리, I/O 등의 자원을 제한하거나 모니터링**할 수 있게 해주는 커널 기능입니다.

### ✅ 주요 기능
- CPU, 메모리, 디스크 I/O, 네트워크 대역폭 제한
- 프로세스 그룹별 자원 통계 수집
- 하위 그룹 계층 구조 지원 (v2에서 통합)

### ✅ 버전 구분
- **cgroups v1**: 서브시스템별 디렉토리 구조
- **cgroups v2**: 단일 계층 구조 (보다 일관성 있음)

---

## 2. ⚙️ cgroups 기본 사용 예시 (v1)

### ✅ 디렉토리 생성 및 프로세스 제한
```bash
# 메모리 제한 예시 (cgroups v1 기준)
mkdir -p /sys/fs/cgroup/memory/mygroup
echo 100000000 > /sys/fs/cgroup/memory/mygroup/memory.limit_in_bytes
echo $$ > /sys/fs/cgroup/memory/mygroup/tasks
```

> 위 예시는 현재 쉘의 PID(`$$`)에 100MB 메모리 제한을 설정

### ✅ 주요 경로

| 경로 | 설명 |
|------|------|
| `/sys/fs/cgroup/cpu/...` | CPU 사용량 제한 |
| `/sys/fs/cgroup/memory/...` | 메모리 제한 및 통계 |
| `/sys/fs/cgroup/pids/...` | 생성 가능한 PID 수 제한 |

---

## 3. 🧩 cgroups v2 개요 및 예시

### ✅ 마운트 및 계층 구조 확인
```bash
mount -t cgroup2 none /sys/fs/cgroup
```

### ✅ 메모리 제한 예시 (v2)
```bash
mkdir /sys/fs/cgroup/mygrp
echo "+memory" > /sys/fs/cgroup/cgroup.subtree_control
echo 100000000 > /sys/fs/cgroup/mygrp/memory.max
echo $$ > /sys/fs/cgroup/mygrp/cgroup.procs
```

> v2에서는 `memory.max`, `cpu.max`, `io.max` 등 통합된 속성 사용

---

## 4. 🧱 Namespace를 통한 프로세스 격리

Namespace는 프로세스가 서로 **다른 환경을 보는 것처럼 분리**하는 기능입니다.

| Namespace | 설명 |
|-----------|------|
| `pid`     | PID 공간 분리 (서로 다른 PID 보기) |
| `net`     | 네트워크 인터페이스 분리 |
| `mnt`     | 파일시스템 마운트 분리 |
| `uts`     | 호스트명, 도메인 분리 |
| `ipc`     | 프로세스 간 통신 공간 분리 |
| `user`    | UID/GID 공간 분리 |

### ✅ `unshare` 명령어로 네임스페이스 생성
```bash
sudo unshare --pid --mount --uts --net --ipc --fork bash
```

> 새로운 bash에서 다른 PID 공간, 호스트명, 마운트 환경을 가지게 됨

---

## 5. 🔧 `systemd` 기반 cgroups 관리

시스템이 `systemd`를 사용하면, `cgroups`는 자동으로 서비스 단위로 관리됩니다.

### ✅ 서비스 단위로 리소스 제한
```ini
# /etc/systemd/system/my-limited.service

[Service]
ExecStart=/usr/bin/sleep 1000
MemoryMax=100M
CPUQuota=50%
```

```bash
sudo systemctl daemon-reexec
sudo systemctl start my-limited.service
```

---

## 6. 🧪 실전 예시

### 예시 1: 특정 PID에 메모리 제한 적용 (v1)
```bash
mkdir /sys/fs/cgroup/memory/mytask
echo 50000000 > /sys/fs/cgroup/memory/mytask/memory.limit_in_bytes
echo 12345 > /sys/fs/cgroup/memory/mytask/tasks
```

### 예시 2: `stress` 명령어와 함께 제한 테스트
```bash
# 메모리 100MB 제한
mkdir /sys/fs/cgroup/memory/test
echo 100000000 > /sys/fs/cgroup/memory/test/memory.limit_in_bytes
echo $$ > /sys/fs/cgroup/memory/test/tasks

# 150MB 사용 시도
stress --vm 1 --vm-bytes 150M --vm-hang 0
```

---

## 🧩 cgroups 관련 도구

| 도구 | 설명 |
|------|------|
| `cgcreate`, `cgexec`, `cgclassify` | `libcgroup` 도구군 (v1에서 사용) |
| `systemd-run --property=` | systemd 기반 리소스 제한 |
| `lxc-cgroup` | LXC 컨테이너용 |
| `cgroupfs-mount` | 수동 마운트 도구 (v1/v2 자동 판별) |

---

## 📌 요약

| 기술 | 기능 | 도구/명령 |
|------|------|------------|
| **cgroups** | 리소스 제한 및 분리 | `/sys/fs/cgroup`, `systemd` |
| **namespace** | 프로세스 격리 | `unshare`, `nsenter` |
| **systemd** | 서비스 기반 제어 | `systemd-run`, `.service` |

---

## ✨ 마무리

`cgroups`와 `namespace`는 단순한 커널 기능을 넘어서 **컨테이너, 보안, 리소스 정책, 샌드박싱**의 기초를 이룹니다.

- 프로세스를 격리하고 자원 사용량을 제한할 수 있으며  
- 이는 **Docker, LXC, systemd 서비스 관리**의 근간이 됩니다.