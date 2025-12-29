---
layout: post
title: Kubernetes - kubelet, Container Runtime 에러 진단 가이드
date: 2025-06-04 20:20:23 +0900
category: Kubernetes
---
# kubelet, Container Runtime 에러 진단 가이드

`kubectl describe`나 `kubectl logs` 명령만으로 원인을 파악하기 어려울 때는 **노드의 kubelet과 런타임 계층**을 확인해야 합니다.

---

## 한눈에 보는 구조와 책임

```
[API Server] ⇄ kubelet(노드 에이전트)
                ⇵ CRI (gRPC)
            container runtime (containerd / CRI-O / Dockershim(과거))
                ⇵
          runc/crun → cgroups/fs/netns → Linux 커널
```

- **kubelet**: PodSpec 동기화, 프로브 실행, 볼륨 마운트, 이미지 관리 요청을 **런타임에 위임**합니다.
- **container runtime**: 컨테이너 생성/시작/정지, 이미지 풀링/관리, 네임스페이스 및 cgroup 설정을 담당합니다.

> **진단 원칙**: *증상은 파드에서 확인*하되, *근본 원인은 노드에서 찾습니다*.
> `kubectl`로 힌트를 잡은 후, **노드로 내려가 `journalctl`, `crictl`, 파일시스템, cgroup, 네트워크 상태**를 확인하세요.

---

## 현장 3분 초진단

```bash
# 노드 상태와 압력 지표 확인
kubectl get nodes -o wide
kubectl describe node <node> | grep -E -i 'Pressure|taints|Condition|Allocatable'

# 클러스터 및 파드의 최근 이벤트 확인
kubectl get events -A --sort-by='.lastTimestamp' | tail -n 50
kubectl describe pod <pod>

# 노드에 접속해 실시간 로그 모니터링
sudo journalctl -u kubelet -f
sudo journalctl -u containerd -f      # CRI-O라면 -u crio, 도커라면 -u docker
```

---

## kubelet & 런타임 — 역할과 로그 위치

### kubelet의 핵심 책임

- PodSpec을 **PodSandbox와 컨테이너**로 실체화
- 프로브 수행 및 상태 보고, 볼륨 마운트(VolumeManager)
- 이미지 가비지 컬렉션(ImageGC), 컨테이너 GC 실행

### container runtime의 역할

- 샌드박스(pause) 컨테이너 생성
- 컨테이너의 실체 생성/시작/정지
- 이미지 풀링 및 관리
- 네트워크, IPC, PID 네임스페이스 및 cgroup 설정

### 로그와 구성 파일 위치

```bash
# 시스템 서비스 로그 확인
sudo journalctl -u kubelet -f
sudo journalctl -u containerd -f    # 또는 -u crio / -u docker

# kubelet 설정 확인 (배포 환경에 따라 경로 다를 수 있음)
sudo ls /var/lib/kubelet/           # 파드 상태, 플러그인, kubeconfig 등
sudo cat /var/lib/kubelet/config.yaml
sudo cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# containerd 설정 확인
sudo cat /etc/containerd/config.toml
```

---

## 런타임 도구(crictl/ctr/nerdctl)로 내부 상태 관찰

`crictl`은 CRI 레벨에서 kubelet과 런타임 간의 상호작용을 진단하는 표준 도구입니다.

```bash
# 런타임 연결 및 기본 정보 확인 (소켓 경로는 배포판마다 다를 수 있음)
sudo crictl info
sudo crictl ps -a
sudo crictl pods
sudo crictl images

# 특정 컨테이너나 샌드박스의 상세 정보 확인
sudo crictl inspect <container-id> | jq .
sudo crictl inspectp <pod-sandbox-id> | jq .

# 컨테이너 로그 확인
sudo crictl logs <container-id> --tail=200 --timestamps
```

containerd의 원시(native) 인터페이스를 사용하는 경우:
```bash
# 컨테이너 및 태스크 목록 확인
sudo ctr -n k8s.io containers list
sudo ctr -n k8s.io tasks list

# 로그는 일반적으로 CRI 경로(/var/log/pods/...)를 통해 접근
sudo ls -R /var/log/pods
```

레거시 Docker 환경(일부 온프레미스 환경):
```bash
sudo docker ps -a
sudo docker inspect <cid>
sudo docker logs <cid>
```

---

## 대표 에러 패턴 — 원인과 해결

아래는 현장에서 가장 자주 접하는 에러와, 로그 신호를 통해 원인을 추적하고 복구하는 흐름입니다.

### ImagePull 실패

```
Failed to pull image "...": rpc error: code = Unknown desc = pull access denied
```
**원인**: 이미지 태그 오타, 퍼블릭/프라이빗 레지스트리 권한 문제, 레지스트리 TLS/미러 설정 오류.
**확인**
```bash
kubectl describe pod <pod> | sed -n '/Events:/,$p'
sudo crictl pull <image>:<tag>  # 노드에서 직접 이미지 풀링 시도
```
**해결**
- 이미지 경로와 태그를 정확히 확인하고, 레지스트리 접속성 및 방화벽을 점검하세요.
- imagePullSecret이 올바른 네임스페이스에 연결되어 있고 도메인이 일치하는지 확인하세요.
- 프록시나 미러를 사용한다면 `/etc/containerd/config.toml`의 `registry.mirrors`와 `config_path` 설정을 확인한 후 `systemctl restart containerd`를 실행하세요.

### PodSandbox/Task 생성 실패

```
failed to create shim task / failed to create containerd task
```
**원인**: containerd shim 프로세스 비정상, 파일시스템(overlayfs) 문제, cgroup 설정 불일치, 디스크 또는 inode 고갈.
**확인**
```bash
sudo journalctl -u containerd | grep -E -i 'shim|overlay|cgroup|No space'
df -h; df -hi
```
**해결**
- 디스크 공간과 inode를 확보하고, `/var/lib/containerd` 디렉터리의 권한과 소유권을 확인하세요.
- 커널 모듈과 파일시스템 옵션(overlayfs, xfs의 ftype=1)을 점검하세요.
- containerd를 재시작하세요. 장애가 지속되면 노드를 드레인(Drain)한 후 진행하세요.

### cgroup 드라이버/계층 문제

```
cgroup: cannot allocate resource / failed to set cgroup
```
**원인**: kubelet의 `cgroupDriver` 설정과 런타임(systemd/cgroupfs)의 드라이버가 불일치하거나, cgroup v1/v2가 혼재되어 있습니다.
**확인**
```bash
sudo cat /var/lib/kubelet/config.yaml | grep cgroupDriver
sudo containerd config dump | grep SystemdCgroup
stat -fc %T /sys/fs/cgroup     # cgroup2fs라면 v2입니다.
```
**해결**
- kubelet과 런타임 모두 **systemd** 드라이버로 통일하는 것을 권장합니다.
- 설정 변경 후 `systemctl daemon-reload && systemctl restart kubelet containerd`를 실행하세요.

### PLEG is not healthy (주기적 발생)

```
PLEG is not healthy: pleg was last seen active...
```
**원인**: kubelet과 런타임 간의 컨테이너 목록 동기화 지연, 런타임 과부하 또는 디스크 병목, 많은 컨테이너의 생성/삭제 반복.
**확인**: kubelet 로그에서 PLEG 경고와 함께 ImageGC/ContainerGC 실패 기록이 있는지 살펴보세요.
**해결**: 불필요한 파드 수를 줄이고, ImageGC 설정을 확인하며, 런타임을 재시작하거나 느린 디스크를 개선하세요.

### Volume 마운트 실패(CSI)

```
MountVolume.MountDevice failed... rpc error: code = Internal
```
**원인**: CSI 드라이버 버그 또는 권한 문제, 노드와 클라우드 API 간 통신 실패, 파일시스템 불일치, multipath 구성 충돌.
**확인**
```bash
kubectl -n kube-system get pods | grep csi
kubectl -n kube-system logs <csi-node-pod> -c node-driver-registrar --tail=200
sudo journalctl -u kubelet | grep -i mount
```
**해결**: CSI 드라이버와 쿠버네티스 버전 호환성을 확인하고, 노드의 IAM/서비스 계정 권한을 점검하세요. 파일시스템 생성 및 포맷을 검토하고, 퍼시스턴트볼륨과 파드가 동일한 가용성 영역(AZ)에 있는지 확인하세요. `WaitForFirstConsumer` 스토리지 클래스를 사용하는 것도 고려하세요.

### Node NotReady / kubelet 등록 실패

```
Node not registered ... failed to get node info
```
**원인**: kubelet과 API 서버 간 TLS/네트워크 통신 장애, `kubelet.conf` 파일 손상, 시스템 시간 차이(스큐).
**확인**
```bash
sudo cat /var/lib/kubelet/kubeconfig
sudo openssl s_client -connect <apiserver>:6443 -showcerts </dev/null 2>/dev/null | openssl x509 -noout -dates
timedatectl status
```
**해결**: NTP 서비스를 이용해 시간을 동기화하고, 방화벽에서 필요한 포트(6443 등)를 허용하세요. 만료된 인증서나 토큰이 있다면 재발급받고, kubelet을 재시작하세요.

### Liveness/Readiness Probe 실패로 인한 컨테이너 반복 재시작

```
Liveness probe failed: ...; Back-off restarting failed container
```
**원인**: 프로브 설정이 너무 까다로움(짧은 timeout 또는 initialDelay), 애플리케이션의 실제 상태 확인 경로나 포트와 불일치.
**해결**: 초기 지연 시간(initialDelaySeconds)과 타임아웃(timeoutSeconds)을 늘려보세요. 애플리케이션 초기화 시간이 길다면 `startupProbe`를 도입하세요. `kubectl port-forward`로 파드에 직접 접속해 프로브 엔드포인트를 수동으로 검증하세요.

### OOMKilled / CPU Throttling

```
State: OOMKilled / cpu throttling high
```
**확인**
```bash
kubectl describe pod <pod> | grep -i OOM
kubectl top pod <pod>
sudo cat /sys/fs/cgroup/memory/.../memory.max
```
**해결**: 메모리 limit 값을 적절히 상향 조정하고, 애플리케이션의 메모리 누수 가능성을 진단하세요. CPU requests와 limits를 재조정하고, 필요하다면 HPA나 VPA를 고려하세요.

### CNI 네트워크 플러그인 실패

```
Failed to create pod sandbox: failed to set up sandbox container "..." network for pod "...": plugin returned error
```
**원인**: CNI 바이너리 또는 구성 파일 누락, IPAM(IP 주소 관리) 충돌, bridge, br_netfilter 등 필요한 커널 모듈이 로드되지 않음.
**확인**
```bash
ls /opt/cni/bin
ls /etc/cni/net.d
sudo lsmod | grep -E 'br_netfilter|overlay'
sudo sysctl net.bridge.bridge-nf-call-iptables
```
**해결**: CNI 플러그인을 설치하거나 버전을 맞추세요. `modprobe br_netfilter`로 커널 모듈을 로드하고, `sysctl -w net.bridge.bridge-nf-call-iptables=1` 명령으로 설정을 적용하세요.

### 파일시스템/디스크 압력

```
NodeHasDiskPressure / overlayfs: no space left on device
```
**확인**
```bash
df -h; df -hi
sudo du -sh /var/lib/containerd /var/lib/docker /var/log
journalctl --disk-usage
```
**해결**: 사용하지 않는 이미지를 정리하고(`crictl rmi`), 오래된 로그 파일을 제거하거나 logrotate를 구성하세요. 필요하다면 파티션을 확장하거나 디스크를 추가하고, inode를 회수하세요.

### SELinux/AppArmor/Seccomp 정책 차단

```
permission denied / Operation not permitted / audit: avc: denied
```
**확인**
```bash
getenforce                      # Enforcing/Permissive/Disabled
sudo ausearch -m avc -ts recent # SELinux 거부 로그 확인
sudo dmesg | grep DENIED        # AppArmor 거부 로그 확인
```
**해결**: 컨테이너에 올바른 SELinux 컨텍스트나 AppArmor 프로파일을 할당하세요. 필요한 Linux Capability만 추가로 허용하고, 과도하게 제한적인 정책은 수정하세요. (보안 영향 평가 필수)

### 컨테이너 시간 불일치 또는 인증서 만료

- **증상**: API 호출 시 TLS 오류, 인증 실패, 이미지 레지스트리 접속 TLS 실패.
- **해결**: 노드의 NTP 동기화 상태를 확인하고, 만료된 인증서는 회전(rotate)하거나 재발급받으세요.

### swap 활성화로 인한 kubelet 기동 실패

```
runtime error: swap is enabled ... failSwapOn=true
```
**해결**
```bash
sudo swapoff -a
sudo sed -i.bak '/ swap / s/^/#/' /etc/fstab
# 또는 kubelet config에 failSwapOn: false 설정 (권장하지 않음)
```

### GPU/Device Plugin 실패

```
Failed to allocate device / device plugin unhealthy
```
**확인**: NVIDIA 등 디바이스 드라이버 버전, Device Plugin 파드의 로그, `/dev` 디렉터리 내 디바이스 파일의 권한.
**해결**: 드라이버, 쿠버네티스, 컨테이너 런타임 버전의 호환성을 확인하세요. Device Plugin 파드를 재배포하고, 필요시 노드 커널을 업데이트하세요.

### Pod sandbox changed 에러(재스케줄링 연쇄)

```
pod sandbox has changed: it will be killed and re-created
```
**원인**: 샌드박스가 손상되었거나, 네트워크 설정에 실패했거나, 런타임 업데이트 중에 발생할 수 있습니다.
**해결**: 런타임 서비스가 안정화된 후 파드를 재생성하세요. 문제가 지속되면 노드를 드레인(Drain)하여 정상화한 후 언코돈(Uncordon)하세요.

---

## 원인 매트릭스 (증상 → 유력 원인 → 첫 번째 액션)

| 증상 | 유력 원인 | 첫 번째 액션 |
|---|---|---|
| `ContainerCreating` 장시간 지속 | CNI/CSI/이미지 풀 지연 | `journalctl -u kubelet/containerd` 로그 확인, `crictl pull` 시도, CNI/CSI 파드 로그 확인 |
| `CrashLoopBackOff` | 애플리케이션 오류/프로브 실패/리소스 한계 | `kubectl logs --previous`로 이전 로그 확인, 프로브 설정 또는 리소스 한계 재조정 |
| `ImagePullBackOff` | 레지스트리 인증/태그 오류/네트워크 | 노드에서 `crictl pull` 직접 실행, imagePullSecret/레지스트리 미러/프록시 설정 확인 |
| `Node NotReady` | 네트워크 단절/TLS 문제/시간 불일치 | NTP 동기화, 방화벽 규칙 점검, kubelet 인증서 및 구성 파일 확인 |
| `OOMKilled` | 메모리 한계 과소/애플리케이션 메모리 누수 | 메모리 limit 값 상향 조정, 애플리케이션 메모리 사용 패턴 조사, VPA/HPA 고려 |
| `DiskPressure` | 로그/이미지 축적으로 디스크 가득 참 | 가비지 컬렉션 실행, logrotate 적용, 디스크 공간 확보 |
| `PLEG unhealthy` | 런타임 과부하/디스크 I/O 병목 | 불필요한 파드 정리, 오래된 이미지 제거, 런타임 서비스 재시작 |

---

## 플레이북 — 단계별 진단 절차

### 스케줄링 및 노드 압력 진단

```bash
kubectl describe pod <pod> | sed -n '/Events:/,$p'
kubectl describe node <node> | grep -E -i 'Pressure|Condition|taints'
kubectl top nodes; kubectl top pods -A
```

### 런타임 및 CRI 레벨 상태 관측

```bash
sudo journalctl -u containerd -n 200 --no-pager
sudo crictl pods; sudo crictl ps -a
sudo crictl inspect <cid> | jq '.info.pid, .status.state'
```

### 네트워크(CNI) 상태 확인

```bash
ls /etc/cni/net.d
sudo journalctl -u kubelet | grep -i 'cni\|network setup'
kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=200
```

### 스토리지(CSI) 상태 확인

```bash
kubectl get pvc -A; kubectl describe pvc <pvc>
kubectl -n kube-system logs -l app=csi-node --tail=200
```

### 파일시스템 및 커널 상태 점검

```bash
df -h; df -hi
sudo lsmod | grep -E 'overlay|br_netfilter'
sudo dmesg | tail -n 50
```

---

## 현장 스니펫 — 빠른 도우미 명령어

### 최근 경고 및 오류 로그만 모아보기

```bash
sudo journalctl -u kubelet -p warning --since "30 min ago"
sudo journalctl -u containerd -p warning --since "30 min ago"
```

### 파드의 실제 PID와 네트워크 네임스페이스를 이용한 진입

```bash
# 컨테이너 PID 확인
CID=$(sudo crictl ps | awk '/myapp/{print $1; exit}')
PID=$(sudo crictl inspect $CID | jq -r '.info.pid')

# 해당 네트워크 네임스페이스에서 curl 실행 (예: 헬스 체크)
sudo nsenter -t $PID -n curl -sS http://127.0.0.1:8080/healthz
```

### 디스크 사용량과 inode 사용량을 동시에 확인

```bash
echo "[disk usage]"; df -h; echo; echo "[inode usage]"; df -hi
```

### 레지스트리 TLS 인증서 체인 확인

```bash
echo | openssl s_client -connect registry.example.com:443 -servername registry.example.com 2>/dev/null | openssl x509 -noout -issuer -subject -dates
```

---

## 자주 묻는 질문(FAQ)

**Q. containerd와 Docker 중 어떤 런타임을 사용해야 하나요?**
A. 쿠버네티스는 CRI(Container Runtime Interface)로 런타임을 표준화했습니다. 현재는 containerd나 CRI-O 사용을 권장합니다. Docker는 Moby 프로젝트와 도커 데몬에 의존하며, 과거에는 Dockershim을 통해 사용되었지만 쿠버네티스 v1.24부터 제거되었습니다.

**Q. cgroup v2 환경에서는 무엇을 주의해야 하나요?**
A. 최신 리눅스 배포판은 기본적으로 cgroup v2를 사용합니다. kubelet과 컨테이너 런타임 모두 `SystemdCgroup` 옵션을 활성화하여 일관성을 유지해야 합니다.

**Q. XFS 파일시스템에서 overlayfs 관련 오류가 발생합니다.**
A. XFS를 overlayfs 백엔드로 사용하려면 `ftype=1` 옵션으로 포맷되어야 합니다. `xfs_info` 명령으로 확인하고, 필요하다면 파일시스템을 재포맷하는 것을 고려하세요.

**Q. 시스템 시간 차이가 왜 심각한 문제를 일으키나요?**
A. TLS 인증서 유효성 검사와 JWT 토큰의 만료 로직은 시스템 시간에 의존합니다. NTP 동기화가 되지 않으면 API 서버 인증, 레지스트리 이미지 풀링 등이 실패할 수 있습니다.

---

## 운영 수칙 요약

- **관측 흐름**: 파드 `describe/events/logs` → **노드 로그**(`journalctl`) → **CRI 관측**(`crictl`) 순으로 추적하세요.
- **자원 점검**: Disk, INode, CPU, Memory Pressure를 항상 먼저 확인하세요.
- **환경 정합성**: cgroup 드라이버를 통일하고, CNI/CSI 버전 호환성을 유지하며, NTP 동기화를 유지하세요.
- **안전한 변경**: 노드 수준의 재시작이나 업데이트는 반드시 `drain` 명령을 먼저 실행하세요.
- **보안 정책 이해**: SELinux, AppArmor, Seccomp 프로파일이 거부하는 로그를 보고 원인을 정확히 판단하세요.

---

## 실전 시나리오 3가지 — 끝까지 해결하기

### 시나리오 1: `ContainerCreating` 상태가 20분 이상 지속되는 경우

1) `kubectl describe pod`를 실행해 `Failed to create pod sandbox`와 함께 CNI 에러 메시지를 확인합니다.
2) 노드에 접속해 `journalctl -u kubelet -f` 로그에서 `cni` 키워드로 검색합니다.
3) `/etc/cni/net.d` 디렉터리에 CNI 구성 파일이 존재하는지 확인하고, 없다면 CNI 플러그인을 재설치합니다.
4) `modprobe br_netfilter` 명령으로 커널 모듈을 로드하고, `sysctl -w net.bridge.bridge-nf-call-iptables=1`로 설정을 적용합니다.
5) 문제 파드를 삭제하고 재생성한 후, 이벤트가 정상적으로 흘러가는지 확인합니다.

### 시나리오 2: 프라이빗 레지스트리에서 `ImagePullBackOff` 발생

1) 노드에서 `crictl pull <이미지:태그>`를 실행해 TLS 에러 등 구체적인 메시지를 확인합니다.
2) `openssl s_client` 명령으로 레지스트리의 루트 및 중간 인증서 체인을 점검합니다.
3) `/etc/containerd/certs.d/<레지스트리_도메인>/hosts.toml` 파일을 올바르게 구성한 후 `systemctl restart containerd`를 실행합니다.
4) 파드의 `imagePullSecret`이 현재 네임스페이스에 존재하고, Secret 내의 도메인 정보가 일치하는지 다시 확인합니다.

### 시나리오 3: `PLEG is not healthy`와 함께 `DiskPressure` 경고 발생

1) 노드에서 `df -h`와 `df -hi` 명령으로 `/var/lib/containerd` 등 주요 디렉터리의 디스크와 inode 사용량을 확인합니다.
2) `crictl images`로 오래되고 사용되지 않는 이미지를 정리하고, `/var/log` 디렉터리의 오래된 로그 파일을 삭제하거나 logrotate를 적용합니다.
3) `systemctl restart containerd`로 런타임을 재시작한 후, kubelet 로그에서 PLEG 경고가 사라지고 파드가 안정화되는지 관찰합니다.

---

## 명령 레퍼런스(요약)

```bash
# kubelet 및 containerd 로그 확인
journalctl -u kubelet -f
journalctl -u containerd -f

# CRI 도구를 통한 상태 확인
crictl ps -a; crictl pods; crictl images
crictl inspect <cid>; crictl logs <cid>

# 노드 기본 리소스 상태 확인
df -h; df -hi; free -h; top
lsmod | grep -E 'overlay|br_netfilter'
timedatectl status

# CNI 및 네트워크 관련 확인
ls /etc/cni/net.d; ls /opt/cni/bin
nsenter -t <pid> -n <cmd>
sysctl net.bridge.bridge-nf-call-iptables

# CSI 및 스토리지 관련 확인
kubectl get pvc -A; kubectl describe pvc <pvc>
# CSI 드라이버 파드 로그는 /var/log/pods/... 또는 kube-system 네임스페이스에서 확인

# 보안 정책 관련 확인
getenforce; ausearch -m avc -ts recent; dmesg | grep DENIED
```

---

## 결론

쿠버네티스 노드 에러 진단의 핵심은 **증상과 원인의 계층을 구분**하는 데 있습니다. 파드에 나타난 증상은 표면일 뿐이며, 근본 원인은 대부분 노드의 kubelet과 컨테이너 런타임 계층에 있습니다. `journalctl`로 시스템 서비스 로그를 확인하고, `crictl` 같은 CRI 관측 도구를 활용해 런타임 내부 상태를 들여다보면 문제의 실체에 가까워질 수 있습니다. 디스크, 메모리, 네트워크, 보안 정책이라는 주요 축을 체계적으로 점검하고, 환경의 정합성을 유지하는 것이 장애를 신속하게 재현, 관측, 복구하는 지름길입니다.