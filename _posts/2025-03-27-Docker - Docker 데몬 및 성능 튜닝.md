---
layout: post
title: Docker - Docker 데몬 및 성능 튜닝
date: 2025-03-27 21:20:23 +0900
category: Docker
---
# ⚙️ Docker 데몬 설정 및 성능 튜닝

---

## 🧠 1. Docker 데몬이란?

`dockerd`는 Docker의 핵심 프로세스로, 다음 기능을 담당합니다:

- 컨테이너/이미지/볼륨/네트워크 관리
- CLI 명령 수신 및 실행
- 로그 기록 및 시스템 상태 모니터링
- 저장소 드라이버와 통신
- 리눅스 커널 기능(cgroups, namespace)과 상호작용

---

## 📄 2. 데몬 설정 파일 (`/etc/docker/daemon.json`)

Linux에서는 일반적으로 이 JSON 파일을 통해 설정합니다.

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

| 항목 | 설명 |
|------|------|
| `data-root` | 이미지/컨테이너가 저장되는 디렉터리 변경 |
| `log-driver` | 로그 드라이버 종류 설정 (`json-file`, `journald`, `syslog`, `none`, `gelf` 등) |
| `log-opts` | 로그 회전 설정 (최대 크기, 파일 수 등) |
| `storage-driver` | 저장소 드라이버 설정 (보통 `overlay2`) |
| `default-shm-size` | 컨테이너 공유 메모리(/dev/shm) 기본 크기 |
| `insecure-registries` | HTTPS 없는 Registry 허용 (테스트용) |

변경 후 반드시 데몬 재시작:

```bash
sudo systemctl restart docker
```

---

## 🚀 3. 성능 관련 주요 설정

### ✅ A. 저장소 드라이버 튜닝

- 대부분의 리눅스에서는 `overlay2` 사용 권장
- `aufs`, `devicemapper`, `btrfs`, `zfs`는 구버전 혹은 특정 목적

```bash
docker info | grep "Storage Driver"
```

> `overlay2`는 Copy-on-write 기반으로 성능과 안정성이 우수

---

### ✅ B. 컨테이너 로그 회전(log rotation)

기본 설정으로는 로그 파일이 무한정 커짐

```json
"log-driver": "json-file",
"log-opts": {
  "max-size": "10m",
  "max-file": "5"
}
```

→ 컨테이너가 많은 서버에서는 반드시 설정해야 로그 폭주 방지

---

### ✅ C. `default-ulimits` 설정 (파일 디스크립터 등)

```json
"default-ulimits": {
  "nofile": {
    "Name": "nofile",
    "Hard": 65535,
    "Soft": 65535
  }
}
```

- `nofile`: 파일 핸들 개수 제한
- `nproc`: 프로세스 개수 제한

> 고부하 서버에서는 반드시 증가시켜야 함

---

### ✅ D. `default-shm-size` 설정

```json
"default-shm-size": "512m"
```

- 기본은 64MB로 적음
- 브라우저, ML, 대형 애플리케이션 등은 `/dev/shm` 부족으로 오류 발생

---

### ✅ E. `data-root` 경로 변경

```json
"data-root": "/mnt/docker"
```

- SSD 디스크나 전용 디스크로 이동시 성능 향상
- `/var/lib/docker`는 기본 경로 → 크기 제한으로 문제 발생 가능

---

## 🔧 4. 데몬 실행 시 커맨드라인 설정

직접 명령어로 실행할 경우:

```bash
dockerd \
  --log-driver=json-file \
  --storage-driver=overlay2 \
  --default-shm-size=512m
```

하지만 대부분의 운영 시스템에서는 systemd를 통해 실행됨:

```bash
sudo systemctl edit docker
```

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --storage-driver=overlay2 --default-shm-size=512m
```

→ 이후 `sudo systemctl daemon-reexec && systemctl restart docker`

---

## 📊 5. 데몬 상태 확인 명령

```bash
docker info
```

출력 항목:

- `Storage Driver`
- `Logging Driver`
- `Cgroup Driver`
- `Kernel Version`
- `Registry Mirrors`
- `Server Version`

```bash
ps aux | grep dockerd
```

→ 현재 실행 중인 옵션 확인 가능

---

## 🧠 6. 고급 튜닝: cgroup driver 일치

쿠버네티스와 연동 시 중요!

- `systemd` 또는 `cgroupfs` 중 일치 필요

```json
"exec-opts": ["native.cgroupdriver=systemd"]
```

> `kubelet`의 cgroup 설정과 일치하지 않으면 pod 실행 실패

---

## 📁 7. Registry 미러 설정 (속도 개선)

```json
"registry-mirrors": ["https://mirror.gcr.io", "https://docker.mirrors.ustc.edu.cn"]
```

- Docker Hub가 느릴 때 미러를 통해 빠르게 이미지 다운로드 가능

---

## 🔐 8. 보안 관련 데몬 설정

| 항목 | 설정 |
|------|------|
| TLS 통신 | `--tlsverify`, `--tlscert`, `--tlskey` |
| Rootless Docker | 일반 사용자 계정에서 Docker 실행 |
| AppArmor/SELinux | 추가 컨테이너 격리 레벨 적용 |
| userns-remap | 사용자 네임스페이스 격리 (`dockremap` 등)

```json
"userns-remap": "default"
```

---

## 🧪 9. 실전: 서버 최적화 설정 예시

```json
{
  "data-root": "/mnt/docker",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "20m",
    "max-file": "10"
  },
  "storage-driver": "overlay2",
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Soft": 65535,
      "Hard": 65535
    }
  },
  "default-shm-size": "512m",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://mirror.gcr.io"]
}
```

---

## 🧰 관련 명령 요약

| 명령어 | 설명 |
|--------|------|
| `docker info` | 현재 데몬 설정 및 상태 확인 |
| `ps aux | grep dockerd` | 데몬 실행 인자 확인 |
| `systemctl restart docker` | 설정 변경 후 재시작 |
| `journalctl -u docker` | 데몬 로그 확인 |

---

## 📚 참고 자료

- [Docker 데몬 공식 문서](https://docs.docker.com/engine/reference/commandline/dockerd/)
- [Docker 성능 튜닝 가이드](https://docs.docker.com/config/containers/resource_constraints/)
- [Rootless Docker](https://docs.docker.com/engine/security/rootless/)

---

## ✅ 요약

| 항목 | 핵심 포인트 |
|------|-------------|
| 로그 관리 | 로그 회전 설정 필수 (`max-size`, `max-file`) |
| 저장소 드라이버 | `overlay2` 권장 |
| 메모리 설정 | `default-shm-size`, `ulimits` 조정 |
| 데이터 경로 | `data-root`로 디스크 I/O 분리 |
| 보안 | `userns-remap`, TLS, rootless 등 고려 |
| 성능 향상 | Registry mirror, SSD 사용, 캐시 전략 등 병행