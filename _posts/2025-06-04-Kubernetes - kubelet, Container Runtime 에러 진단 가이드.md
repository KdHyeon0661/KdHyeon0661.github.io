---
layout: post
title: Kubernetes - kubelet, Container Runtime 에러 진단 가이드
date: 2025-06-04 20:20:23 +0900
category: Kubernetes
---
# kubelet, Container Runtime 에러 진단 가이드

Kubernetes 클러스터에서 Pod가 제대로 동작하지 않는 경우, `kubectl describe pod`와 `kubectl logs`로 파악할 수 있는 정보가 한계일 수 있습니다.  
이럴 땐 **노드에서 직접 실행되는 kubelet과 컨테이너 런타임(containerd, Docker 등)의 로그를 확인**하여 원인을 파악해야 합니다.

이 글에서는:

- kubelet, container runtime 역할
- 로그 확인 방법
- 주요 에러 사례 및 진단 팁
을 정리합니다.

---

## ✅ kubelet이란?

> kubelet은 **각 노드에서 실행되는 Kubernetes 에이전트**로, Control Plane의 지시에 따라:
>
> - Pod를 생성
> - 상태를 모니터링
> - 컨테이너 런타임과 상호작용  
> 등을 수행합니다.

### 주요 기능:

- **PodSpec → 실제 컨테이너로 변환**
- **liveness/readiness probe 수행**
- **노드 상태 report**

---

## ✅ Container Runtime이란?

> 컨테이너를 실제로 생성/실행/중지하는 역할을 하는 엔진입니다.

대표적인 종류:

- `containerd` (기본, K8s 공식 런타임)
- `Docker` (초창기 많이 사용)
- `CRI-O`, `rktd`, `Mirantis Container Runtime`

Kubernetes는 [CRI(Container Runtime Interface)]를 통해 kubelet과 런타임을 연결합니다.

---

## ✅ kubelet 로그 확인 방법

노드에 SSH로 접속한 후, 시스템 로그로 확인합니다.

### 📌 Systemd 기반 시스템:

```bash
sudo journalctl -u kubelet -f
```

- `-u kubelet`: kubelet 유닛 로그 필터
- `-f`: 실시간 로그 tail

### 📌 로그 검색:

```bash
sudo journalctl -u kubelet | grep -i 'error'
```

---

## ✅ Container Runtime 로그 확인

### 🔸 containerd 사용 시:

```bash
sudo journalctl -u containerd -f
```

### 🔸 Docker 사용 시:

```bash
sudo journalctl -u docker -f
```

또는 특정 컨테이너의 상태:

```bash
sudo crictl ps -a       # containerd 기준 모든 컨테이너 확인
sudo crictl inspect <container-id>
```

※ `crictl`은 `kubelet` ↔ `container runtime` 진단에 특화된 도구이며, 설치 추천

---

## ✅ 에러 사례별 진단 가이드

---

### 🔹 1. ImagePull 관련 오류

```bash
Failed to pull image "myrepo/myapp:v1.0": rpc error: code = Unknown desc = Error response from daemon: pull access denied
```

#### 원인:
- 이미지 존재하지 않음
- 레지스트리 인증 실패

#### 해결:
- 이미지 경로 및 태그 확인
- `imagePullSecrets` 설정 여부 점검

---

### 🔹 2. containerd gRPC 통신 오류

```bash
Failed to create container: rpc error: code = Unknown desc = failed to create shim task
```

#### 원인:
- containerd의 shim 관리 프로세스 문제
- 파일 시스템, cgroup 관련 문제

#### 해결:
- containerd 재시작
- `/var/lib/containerd/` 디렉토리 권한/공간 확인

---

### 🔹 3. cgroup 관련 오류

```bash
failed to reserve memory: cgroup: cannot allocate resource
```

#### 원인:
- 노드 자원 부족
- kubelet이 리소스 제한 설정에 실패

#### 해결:
- 노드 자원 모니터링 (`top`, `free -h`)
- `system-reserved`, `kube-reserved` 설정 확인

---

### 🔹 4. Volume 마운트 실패

```bash
MountVolume.MountDevice failed for volume "pvc-xxxxx": rpc error: code = Internal desc = ...
```

#### 원인:
- PVC 바인딩은 되었으나, 노드에서 디바이스 마운트 실패
- 권한, 드라이버 문제 등

#### 해결:
- CSI 로그 (`/var/log/pods` 또는 `journalctl -u kubelet`)
- cloud-provider plugin 동작 확인

---

### 🔹 5. kubelet 등록 실패 / Node NotReady

```bash
Node not registered because of: failed to get node info from kubelet
```

#### 원인:
- kubelet이 API 서버와 통신 불가
- 인증서, kubeconfig 문제

#### 해결:
- `kubelet.conf` 경로 확인: `/var/lib/kubelet/kubeconfig`
- API 서버와의 연결성 점검 (방화벽, DNS 등)

---

## ✅ 유용한 진단 도구

| 도구 | 설명 |
|------|------|
| `journalctl` | kubelet, containerd, docker 로그 확인 |
| `crictl` | container runtime 직접 제어 및 확인 |
| `kubelet --version` | 버전 확인 및 설정 파악 |
| `df -h`, `free -h` | 디스크, 메모리 상태 확인 |
| `kubectl describe node <node>` | 전체 리소스 상태 및 taint 확인 |

---

## ✅ 고급 설정 (참고)

kubelet 설정은 보통 `/var/lib/kubelet/config.yaml` 또는 `/etc/kubernetes/kubelet.conf`에서 확인합니다.

```yaml
kind: KubeletConfiguration
cgroupDriver: systemd
failSwapOn: true
authentication:
  anonymous:
    enabled: false
```

---

## ✅ 정리

| 항목 | 설명 |
|------|------|
| 문제 상황 | Pod 생성 지연, CrashLoopBackOff, ContainerCreating |
| 진단 대상 | kubelet, container runtime(containerd/docker) |
| 접근 방법 | `journalctl`, `crictl`, `/var/lib/kubelet` 내 설정 확인 |
| 공통 원인 | 이미지 Pull 실패, cgroup 자원 오류, volume mount 실패 |
| 해결 전략 | 로그 분석 → 자원 상태 점검 → 설정 수정/재시작

---

## ✅ 참고 링크

- [Kubelet 공식 문서](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
- [Containerd troubleshooting](https://github.com/containerd/containerd/blob/main/docs/troubleshooting.md)
- [crictl 사용법](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)