---
layout: post
title: Kubernetes - kubelet, Container Runtime 에러 진단 가이드
date: 2025-06-04 20:20:23 +0900
category: Kubernetes
---
# kubelet, Container Runtime 에러 진단 가이드

`kubectl describe`/`kubectl logs`만으로 원인이 안 보일 때, **노드의 kubelet·런타임 계층**을 확인해야 합니다.  

---

## 0) 한눈에 보는 구조와 책임

```
[API Server] ⇄ kubelet(노드 에이전트)
                ⇵ CRI (gRPC)
            container runtime (containerd / CRI-O / Dockershim(과거))
                ⇵
          runc/crun → cgroups/fs/netns → Linux 커널
```

- **kubelet**: PodSpec 동기화, 프로브, 볼륨 마운트, 이미지 관리 요청을 **런타임에 위임**.
- **container runtime**: 컨테이너 생성/시작/정지, 이미지 풀/관리, 네임스페이스, cgroup 설정.

> **진단 원칙**: *증상은 Pod*, *원인은 노드*.  
> `kubectl`에서 힌트를 잡고, **노드로 내려가 `journalctl`, `crictl`, 파일시스템·cgroup·네트워크**를 확인합니다.

---

## 1) 현장 3분 초진단

```bash
# 1) 노드 상태/압력
kubectl get nodes -o wide
kubectl describe node <node> | egrep -i 'Pressure|taints|Condition|Allocatable'

# 2) 파드 이벤트/사건 기록
kubectl get events -A --sort-by='.lastTimestamp' | tail -n 50
kubectl describe pod <pod>

# 3) 노드로 접속해 실시간 로그
sudo journalctl -u kubelet -f
sudo journalctl -u containerd -f      # cri-o면 -u crio, 도커면 -u docker
```

---

## 2) kubelet & 런타임 — 역할과 로그 위치

### 2.1 kubelet 핵심 책임
- PodSpec → **PodSandbox + Containers**로 실체화
- 프로브 수행 / 상태 보고 / 볼륨 마운트(VolumeManager)
- 이미지 가비지 컬렉션(ImageGC), 컨테이너 GC

### 2.2 container runtime
- 샌드박스(pause) 컨테이너 생성
- 컨테이너 실체 생성/시작/정지
- 이미지 풀/관리
- 네임스페이스(net/ipc/pid), cgroup 설정

### 2.3 로그·구성 파일
```bash
# 로그(시스템드)
sudo journalctl -u kubelet -f
sudo journalctl -u containerd -f    # 또는 -u crio / -u docker

# 설정(배포에 따라 경로 상이)
sudo ls /var/lib/kubelet/           # pod 상태, plugin, kubeconfig 등
sudo cat /var/lib/kubelet/config.yaml
sudo cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# containerd config
sudo cat /etc/containerd/config.toml
```

---

## 3) 런타임 도구(crictl/ctr/nerdctl)로 내부 보기

`crictl`은 CRI 레벨에서 kubelet↔런타임 경계를 진단하는 표준 도구입니다.

```bash
# 연결 확인 (socket 경로는 배포에 따라 다름)
sudo crictl info
sudo crictl ps -a
sudo crictl pods
sudo crictl images

# 특정 컨테이너/샌드박스 상세
sudo crictl inspect <container-id> | jq .
sudo crictl inspectp <pod-sandbox-id> | jq .

# 로그 보기(컨테이너 기준)
sudo crictl logs <container-id> --tail=200 --timestamps
```

containerd 원시 인터페이스:
```bash
# 목록 및 상태
sudo ctr -n k8s.io containers list
sudo ctr -n k8s.io tasks list

# 로그는 CRI 경로로 접근 (보통 /var/log/pods/...)
sudo ls -R /var/log/pods
```

Docker 기반 환경(레거시/온프렘 일부):
```bash
sudo docker ps -a
sudo docker inspect <cid>
sudo docker logs <cid>
```

---

## 4) 대표 에러 패턴 — 원인과 해결

아래는 현장에서 **가장 자주 만나는 에러 15가지**와, **로그 시그널 → 원인 → 복구** 흐름입니다.

### 4.1 ImagePull 실패
```
Failed to pull image "...": rpc error: code = Unknown desc = pull access denied
```
**원인**: 이미지 태그 오타, 퍼블릭/프라이빗 레지스트리 권한, 레지스트리 TLS/미러 설정 문제.  
**확인**
```bash
kubectl describe pod <pod> | sed -n '/Events:/,$p'
sudo crictl pull <image>:<tag>  # 노드에서 직접 풀 시도
```
**해결**
- 이미지 경로/태그 교정, 레지스트리 접속성/방화벽 점검
- imagePullSecret 연결 및 도메인 일치 확인
- 프록시/미러 사용 시 `/etc/containerd/config.toml`의 `registry.mirrors`/`config_path` 확인 후 `systemctl restart containerd`

### 4.2 PodSandbox/Task 생성 실패
```
failed to create shim task / failed to create containerd task
```
**원인**: containerd shim 비정상, 파일시스템(overlayfs), cgroup 설정 불일치, 디스크/ inode 고갈.  
**확인**
```bash
sudo journalctl -u containerd | egrep -i 'shim|overlay|cgroup|No space'
df -h; df -hi
```
**해결**
- 디스크/ inode 확보, `/var/lib/containerd` 퍼미션/소유권 확인
- 커널 모듈/파일시스템 옵션(overlayfs, xfs ftype=1) 점검
- containerd 재시작(장애 중복 시 노드 드레인 후)

### 4.3 cgroup 드라이버/계층 문제
```
cgroup: cannot allocate resource / failed to set cgroup
```
**원인**: kubelet `cgroupDriver`와 런타임(systemd/cgroupfs) 불일치, cgroup v1/v2 혼선.  
**확인**
```bash
sudo cat /var/lib/kubelet/config.yaml | grep cgroupDriver
sudo containerd config dump | grep SystemdCgroup
stat -fc %T /sys/fs/cgroup     # cgroup2fs면 v2
```
**해결**
- 두 곳 모두 **systemd**로 통일 권장
- 변경 후 `systemctl daemon-reload && systemctl restart kubelet containerd`

### 4.4 PLEG is not healthy (주기적)
```
PLEG is not healthy: pleg was last seen active...
``>
**원인**: kubelet과 런타임 간 목록 동기화 지연, 런타임 과부하/디스크 병목, 많은 컨테이너 생성/삭제.  
**확인**: kubelet 로그에서 PLEG 경고와 함께 ImageGC/ContainerGC 실패 여부.  
**해결**: 파드 수 축소, ImageGC 동작 확인, 런타임 재시작, 느린 디스크 개선.

### 4.5 Volume 마운트 실패(CSI)
```
MountVolume.MountDevice failed... rpc error: code = Internal
```
**원인**: CSI 드라이버 버그/권한, 노드-클라우드 API 실패, 파일시스템 불일치, multipath 충돌.  
**확인**
```bash
kubectl -n kube-system get pods | grep csi
kubectl -n kube-system logs <csi-node-pod> -c node-driver-registrar --tail=200
sudo journalctl -u kubelet | grep -i mount
```
**해결**: CSI 버전 호환, 노드 IAM/권한, 파일시스템 생성/수정, 같은 AZ/Zone 매칭, `WaitForFirstConsumer` SC 사용.

### 4.6 Node NotReady / kubelet 등록 실패
```
Node not registered ... failed to get node info
```
**원인**: kubelet ↔ API서버 TLS/네트워크, `kubelet.conf` 손상, 시간 스큐.  
**확인**
```bash
sudo cat /var/lib/kubelet/kubeconfig
sudo openssl s_client -connect <apiserver>:6443 -showcerts </dev/null 2>/dev/null | openssl x509 -noout -dates
timedatectl status
```
**해결**: NTP 동기화, 방화벽 허용, 인증서/토큰 재발급, kubelet 재시작.

### 4.7 Liveness/Readiness Probe로 kill 반복
```
Liveness probe failed: ...; Back-off restarting failed container
```
**원인**: 프로브 과격(짧은 timeout/initialDelay), 경로/포트 불일치.  
**해결**: 초기 지연/타임아웃 상향, `startupProbe` 도입, `port-forward`로 직접 검증.

### 4.8 OOMKilled / CPU Throttling
```
State: OOMKilled / cpu throttling high
```
**확인**
```bash
kubectl describe pod <pod> | grep -i OOM
kubectl top pod <pod>
sudo cat /sys/fs/cgroup/memory/.../memory.max
```
**해결**: 메모리 limit 상향/튜닝, 누수 진단, CPU requests/limits 재조정(HPA/VPA 고려).

### 4.9 네트워크 플러그인(CNI) 실패
```
Failed to create pod sandbox: failed to set up sandbox container "..." network for pod "...": plugin returned error
```
**원인**: CNI 바이너리/설정 누락, IPAM 충돌, 커널 모듈(bridge, br_netfilter) 미로딩.  
**확인**
```bash
ls /opt/cni/bin
ls /etc/cni/net.d
sudo lsmod | egrep 'br_netfilter|overlay'
sudo sysctl net.bridge.bridge-nf-call-iptables
```
**해결**: CNI 설치/버전 맞춤, `modprobe br_netfilter`, `sysctl -w net.bridge.bridge-nf-call-iptables=1`.

### 4.10 파일시스템/디스크 압력
```
NodeHasDiskPressure / overlayfs: no space left on device
```
**확인**
```bash
df -h; df -hi
sudo du -sh /var/lib/containerd /var/lib/docker /var/log
journalctl --disk-usage
```
**해결**: 이미지/로그 정리, logrotate, 파티션/디스크 증설, inode 회수.

### 4.11 SELinux/AppArmor/Seccomp 차단
```
permission denied / Operation not permitted / audit: avc: denied
```
**확인**
```bash
getenforce                      # Enforcing/Permissive
sudo ausearch -m avc -ts recent # SELinux denials
sudo dmesg | grep DENIED        # AppArmor
```
**해결**: 올바른 프로파일/컨텍스트 부여, 필요한 capability만 허용, 정책 수정(보안 영향 주의).

### 4.12 컨테이너 시간/인증서 만료
- **증상**: API 호출 TLS 오류, 인증 실패, 이미지 레지스트리 TLS 실패.  
- **해결**: NTP 동기화, 인증서 회전/재발급.

### 4.13 swap 활성화로 kubelet 기동 실패
```
runtime error: swap is enabled ... failSwapOn=true
```
**해결**
```bash
sudo swapoff -a
sudo sed -i.bak '/ swap / s/^/#/' /etc/fstab
# 또는 kubelet config에 failSwapOn: false (권장 X)
```

### 4.14 GPU/Device Plugin 실패
```
Failed to allocate device / device plugin unhealthy
```
**확인**: 디바이스 드라이버 버전, DP Pod 로그, `/dev` 퍼미션.  
**해결**: 드라이버/쿠버/런타임 버전 호환, DP 재배포, 노드 커널 업데이트.

### 4.15 Pod sandbox changed 에러(재스케줄링 연쇄)
```
pod sandbox has changed: it will be killed and re-created
```
**원인**: 샌드박스 파손/네트워크 셋업 실패/런타임 갱신 중.  
**해결**: 런타임 안정화 후 파드 재생성, 노드 드레인→정상화→언코돈.

---

## 5) 원인 매트릭스 (증상 → 유력 원인 → 첫 액션)

| 증상 | 유력 원인 | 첫 액션 |
|---|---|---|
| `ContainerCreating` 장기 | CNI/CSI/이미지 풀 지연 | `journalctl -u kubelet/containerd`, `crictl pull`, CNI/CSI 로그 |
| `CrashLoopBackOff` | 앱 오류/프로브/리밋 | `logs --previous`, 프로브/리소스 재조정 |
| `ImagePullBackOff` | 레지스트리 인증/태그 | `crictl pull`, imagePullSecret/미러/프록시 |
| `Node NotReady` | 네트워크/TLS/시간 | NTP, 방화벽, kubelet 인증서/구성 |
| `OOMKilled` | 리밋 과소/누수 | limit↑, 누수 조사, VPA/HPA |
| `DiskPressure` | 로그/이미지 축적 | GC, logrotate, 디스크 증설 |
| `PLEG unhealthy` | 런타임 과부하/디스크 | 파드 수/이미지 정리, 런타임 재시작 |

---

## 6) 플레이북 — 단계별 절차

### 6.1 스케줄/노드 압력 진단
```bash
kubectl describe pod <pod> | sed -n '/Events:/,$p'
kubectl describe node <node> | egrep -i 'Pressure|Condition|taints'
kubectl top nodes; kubectl top pods -A
```

### 6.2 런타임/CRI 레벨 관측
```bash
sudo journalctl -u containerd -n 200 --no-pager
sudo crictl pods; sudo crictl ps -a
sudo crictl inspect <cid> | jq '.info.pid, .status.state'
```

### 6.3 네트워크(CNI)
```bash
ls /etc/cni/net.d
sudo journalctl -u kubelet | grep -i 'cni\|network setup'
kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=200
```

### 6.4 스토리지(CSI)
```bash
kubectl get pvc -A; kubectl describe pvc <pvc>
kubectl -n kube-system logs -l app=csi-node --tail=200
```

### 6.5 파일시스템/커널
```bash
df -h; df -hi
sudo lsmod | egrep 'overlay|br_netfilter'
sudo dmesg | tail -n 50
```

---

## 7) 설정 체크리스트

### kubelet (일부)
```yaml
# /var/lib/kubelet/config.yaml
kind: KubeletConfiguration
cgroupDriver: systemd
featureGates:
  # 필요 시 기능 플래그
failSwapOn: true
authentication:
  anonymous:
    enabled: false
```

### containerd (발췌)
```toml
# /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true

[plugins."io.containerd.grpc.v1.cri".registry.mirrors."your-registry.example.com"]
  endpoint = ["https://mirror1.your-registry.example.com"]
```

변경 후:
```bash
sudo systemctl daemon-reload
sudo systemctl restart containerd kubelet
```

---

## 8) 현장 스니펫 — 빠른 도우미

### 8.1 최근 경고/오류만 모아보기
```bash
sudo journalctl -u kubelet -p warning --since "30 min ago"
sudo journalctl -u containerd -p warning --since "30 min ago"
```

### 8.2 파드의 실제 PID/네임스페이스로 진입
```bash
# 컨테이너 PID 확인
CID=$(sudo crictl ps | awk '/myapp/{print $1; exit}')
PID=$(sudo crictl inspect $CID | jq -r '.info.pid')
# 해당 netns에서 curl
sudo nsenter -t $PID -n curl -sS http://127.0.0.1:8080/healthz
```

### 8.3 inode/디스크 동시에 확인
```bash
echo "[disk]"; df -h; echo; echo "[inode]"; df -hi
```

### 8.4 레지스트리 TLS 체인 확인
```bash
echo | openssl s_client -connect registry.example.com:443 -servername registry.example.com 2>/dev/null | openssl x509 -noout -issuer -subject -dates
```

---

## 9) 자주 묻는 질문(FAQ)

**Q. containerd vs Docker?**  
A. Kubernetes는 CRI로 런타임을 표준화. containerd/CRI-O 권장. Docker는 Moby/도커데몬 의존으로 과거엔 Dockershim을 사용했으나 v1.24+에서 제거.

**Q. cgroup v2 환경?**  
A. Modern 배포는 v2 디폴트. kubelet, 런타임 모두 systemd cgroup 옵션으로 정합성 유지.

**Q. XFS에서 overlayfs 오류?**  
A. XFS는 `ftype=1` 필수. `xfs_info`로 확인하고, 필요시 재포맷 고려.

**Q. 시간 스큐가 왜 치명적?**  
A. TLS 검증/토큰 만료 로직이 시간 의존. NTP 불일치로 인증·레지스트리 풀이 실패할 수 있음.

---

## 10) 운영 수칙 요약

- **관측**: `describe/events/logs` → **노드 로그**(`journalctl`) → **CRI 관측**(`crictl`)
- **자원**: Disk/INode/CPU/Memory/Pressure 먼저 확인
- **정합성**: cgroup 드라이버 통일, CNI/CSI 버전 호환, NTP 동기화
- **변경은 안전하게**: `drain` 후 노드 수준 재시작/업데이트
- **보안/정책**: SELinux/AppArmor/Seccomp 프로파일 인지하고 원인과 분리

---

## 11) 실전 시나리오 3종 — 끝까지 해결하기

### 11.1 `ContainerCreating` 20분 지속
1) `describe pod` → `Failed to create pod sandbox` + CNI 에러  
2) 노드: `journalctl -u kubelet -f`에서 `cni` 키워드 검색  
3) `/etc/cni/net.d` 파일 누락 확인 → CNI 재설치  
4) `modprobe br_netfilter` + `sysctl -w net.bridge.bridge-nf-call-iptables=1`  
5) 파드 재생성, 이벤트 정상화 확인

### 11.2 `ImagePullBackOff` (프라이빗 레지스트리)
1) `crictl pull <image>` → TLS 에러  
2) 레지스트리 루트/중간 인증서 체인 확인(위 openssl)  
3) `/etc/containerd/certs.d/<registry>/hosts.toml` 구성 후 containerd 재시작  
4) imagePullSecret 네임스페이스/도메인 매칭 재확인 → 성공

### 11.3 `PLEG is not healthy` + DiskPressure
1) 노드 `df -h/df -hi` → `/var/lib/containerd` 가득 참  
2) `crictl images` 오래된 이미지 정리, 로그 정리, logrotate 적용  
3) containerd 재시작 → kubelet 경고 해소, 파드 안정

---

## 12) 명령 레퍼런스(요약)

```bash
# kubelet/containerd 로그
journalctl -u kubelet -f
journalctl -u containerd -f

# CRI
crictl ps -a; crictl pods; crictl images
crictl inspect <cid>; crictl logs <cid>

# 노드 리소스
df -h; df -hi; free -h; top
lsmod | egrep 'overlay|br_netfilter'
timedatectl status

# CNI/네트워크
ls /etc/cni/net.d; ls /opt/cni/bin
nsenter -t <pid> -n <cmd>
sysctl net.bridge.bridge-nf-call-iptables

# CSI/스토리지
kubectl get pvc -A; kubectl describe pvc <pvc>
/var/log/pods/*csi* / kube-system csi-*

# 보안
getenforce; ausearch -m avc -ts recent; dmesg | grep DENIED
```

---

## 13) 결론

- **증상(Pod)** 은 표면, **원인(노드)** 은 심층.  
- `kubelet`·`container runtime` 로그와 **CRI 관측 도구**로 경계를 열고,  
- **자원·정합성·보안** 3축을 점검하면 대부분의 장애를 **재현→관측→복구**로 닫을 수 있습니다.

---

## 참고 링크

- Kubelet Reference: <https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/>
- containerd Troubleshooting: <https://github.com/containerd/containerd/blob/main/docs/troubleshooting.md>
- crictl Guide: <https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/>
- CNI Spec: <https://www.cni.dev/docs/spec/>
- Kubernetes Node Problem Detector: <https://github.com/kubernetes/node-problem-detector>