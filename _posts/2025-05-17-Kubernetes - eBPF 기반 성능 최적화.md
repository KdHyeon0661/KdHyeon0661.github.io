---
layout: post
title: Kubernetes - eBPF 기반 성능 최적화
date: 2025-05-11 20:20:23 +0900
category: Kubernetes
---
# eBPF 기반 성능 최적화

## 0. 요약: 언제 eBPF를 쓰면 이득인가?

- **네트워크**: iptables 규칙 폭증/Conntrack 병목 → eBPF로 L3/L4 처리(XDP/tc), **Cilium**로 CNI 가속
- **관측/프로파일링**: CPU 핫스팟, 스케줄 지연, I/O 지연 → **bpftrace/libbpf-tools** 로 실시간 관측, **오버헤드 낮음**
- **보안**: 시스템콜/프로세스 행동 정책, CAP 최소화 → **LSM(eBPF)** 기반 정책, 프로세스 원천 추적
- **Kubernetes**: **Hubble**로 흐름 시각화, L7 정책, egress 가속, 대규모 노드 확장성

---

## 1. eBPF 핵심 이해

### 1.1 실행 모델
- **프로그램 타입**: XDP, tc, kprobe/kretprobe, tracepoint, uprobe/uretprobe, perf_event, cgroup/lsm 등
- **맵 타입**: `hash`, `array`, **per-CPU hash/array**, LRU map, ring/perf buffer, stack/map-of-maps, LPM trie
- **로드/검증**: 바이트코드를 `verifier`가 **메모리 안전성·루프 한정성** 검증 → JIT로 네이티브 변환
- **상호작용**: 커널 helper, BPF map을 통한 유저스페이스 통신, perf/ring buffer 이벤트 스트림

### 1.2 CO-RE(BPF Compile-Once, Run-Everywhere)
- **libbpf + BTF** 기반으로 커널 심볼/필드 변화를 런타임에 **재매칭** → 바이너리 하나로 **여러 커널** 호환
- 장점: **패키징/배포 간소화**, **컨테이너**에서의 이식성 향상

---

## 2. 도구 선택 가이드

| 목적 | 권장 도구 | 비고 |
|---|---|---|
| 원인 추적·탐색(수분) | **bpftrace** | 원라이너·DSL, 빠른 가설 검증 |
| 저오버헤드 상시 관측 | **libbpf-tools** | CO-RE, 생산 환경 상주 에이전트 |
| 교육·시작 | **BCC**(bpfcc-tools) | Python 래퍼, 배포/의존성 무거움 |
| 정책/네트워킹 | **Cilium** | eBPF CNI, Hubble 관측 |
| 빌드/디버깅 | **bpftool** | map/pin/verify/dump, 필수 만능툴 |

> 새로 시작한다면 **bpftrace + libbpf-tools + bpftool** 조합이 가장 실전적이다.

---

## 3. 환경 준비

### 3.1 커널·패키지
- 권장 커널: **5.x 이상** (4.18+ 가능, 기능 차이 존재)
- 배포판 패키지:
```bash
# Ubuntu 예시
sudo apt-get update
sudo apt-get install -y linux-headers-$(uname -r) \
  bpfcc-tools bpftrace bpftool clang llvm libbpf-dev libelf-dev zlib1g-dev
```

### 3.2 권한
- root 권장. 컨테이너에서는 CAP_BPF, CAP_PERFMON, CAP_SYS_ADMIN 등이 요구될 수 있다.

---

## 4. 즉시 써먹는 관측 원라이너 (bpftrace)

### 4.1 CPU 핫스팟(함수 레벨)
```bash
sudo bpftrace -e 'profile:hz:99 { @[kstack] = count(); }' | head
```
- 99Hz 주기로 커널 스택을 샘플링 → 상위 스택으로 히트맵.

### 4.2 디스크 I/O 지연
```bash
sudo bpftrace -e 'tracepoint:block:block_rq_complete { @lat[args->dev] = hist(args->nr_sector); }'
```

### 4.3 read() 지연 분포
```bash
sudo bpftrace -e 'kprobe:vfs_read { @start[tid] = nsecs; } kretprobe:vfs_read /@start[tid]/ { @d = hist((nsecs-@start[tid])/1000000); delete(@start[tid]); }'
```

### 4.4 TCP RTT/재전송 관찰
```bash
sudo bpftrace -e 'tracepoint:tcp:tcp_retransmit_skb { @retx[comm] = count(); }'
```

---

## 5. libbpf-tools로 상시 관측 (저오버헤드)

### 5.1 설치(소스)
```bash
git clone https://github.com/iovisor/bcc --depth 1
git clone https://github.com/iovisor/bpftrace --depth 1
git clone https://github.com/iovisor/bpf-perf-tools --depth 1
git clone https://github.com/iovisor/bcc/libbpf-tools --depth 1
```

> 배포 패키지에 포함된 경우 `apt install bpftrace bpfcc-tools` 만으로도 충분.

### 5.2 대표 툴

| 툴 | 목적 |
|---|---|
| `runqlat` | 스케줄 큐 대기 지연(latency) |
| `biolatency` | 블록 I/O 지연 히스토그램 |
| `tcptop` | TCP 송수신 상위 프로세스 |
| `softirqs` | 소프트IRQ 시간 분해 |
| `memleak` | 잠재적 메모리 누수 추적(실험적) |

예시:
```bash
sudo /usr/share/bcc/tools/biolatency -m 1
sudo /usr/share/bcc/tools/tcptop -C 1 10
```

---

## 6. 네트워킹 가속: XDP & tc-BPF

### 6.1 XDP로 드롭/리다이렉트
- **NIC 드라이버** 단계에서 BPF 실행 → **패킷을 가장 빠른 경로에서 처리**
- 사용: DDoS 방어, L3/L4 LB, 빠른 드롭/리다이렉트

#### 6.1.1 최소 XDP 드롭 예제(C)
```c
// xdp_drop.c (CO-RE 전용 아님: 데모용)
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_drop_all(struct xdp_md *ctx) {
    return XDP_DROP;
}
char LICENSE[] SEC("license") = "GPL";
```
빌드/적용:
```bash
clang -O2 -target bpf -c xdp_drop.c -o xdp_drop.o
# 인터페이스에 부착
sudo ip link set dev eth0 xdp obj xdp_drop.o sec xdp
# 제거
sudo ip link set dev eth0 xdp off
```

#### 6.1.2 조건부 드롭(예: 특정 포트 차단)
```c
// xdp_block_port.c
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/udp.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_block_udp_12345(struct xdp_md *ctx) {
  void *data = (void *)(long)ctx->data;
  void *data_end = (void *)(long)ctx->data_end;

  struct ethhdr *eth = data;
  if ((void*)(eth + 1) > data_end) return XDP_PASS;
  if (eth->h_proto != __constant_htons(ETH_P_IP)) return XDP_PASS;

  struct iphdr *ip = (void*)(eth + 1);
  if ((void*)(ip + 1) > data_end) return XDP_PASS;
  if (ip->protocol != IPPROTO_UDP) return XDP_PASS;

  struct udphdr *udp = (void*)(ip + 1);
  if ((void*)(udp + 1) > data_end) return XDP_PASS;

  if (udp->dest == __constant_htons(12345)) return XDP_DROP;
  return XDP_PASS;
}
char LICENSE[] SEC("license") = "GPL";
```

### 6.2 tc egress/ingress BPF
- L2 이후, qdisc 레이어에서 TC BPF로 **정교한 분류/리라이트** 가능
- **XDP→tc** 파이프라인을 조합해 단계적 처리

적용 예:
```bash
# tc qdisc 준비
sudo tc qdisc add dev eth0 clsact
# ingress에 BPF 프로그램 부착
sudo tc filter add dev eth0 ingress bpf obj tc_prog.o sec cls_main
```

---

## 7. 시스템콜/스택 관측: kprobe/tracepoint/uprobes

### 7.1 kprobe
커널 함수 진입/복귀 시점 후킹:
```bash
sudo bpftrace -e 'kprobe:__x64_sys_openat { printf("%s %s\n", comm, str(arg1)); }'
```

### 7.2 tracepoint
ABI가 안정적인 커널 이벤트:
```bash
sudo bpftrace -e 'tracepoint:sched:sched_switch { @[prev_comm, next_comm] = count(); }'
```

### 7.3 uprobes
유저 공간 바이너리의 함수 진입/복귀 후킹:
```bash
sudo bpftrace -e 'uprobe:/usr/bin/nginx:ngx_http_process_request_line { @[comm] = count(); }'
```

> 프로덕션에서는 **심볼 해석/ASLR 고려**가 필요. `nm`, `objdump`, `readelf`로 오프셋 확인.

---

## 8. 맵·버퍼·성능 고려

### 8.1 per-CPU 맵
- 글로벌 락 경합 없이 CPU별 슬롯을 사용 → **통계 수집에 탁월**
- 유저스페이스에서 합산

### 8.2 perf buffer vs ring buffer
- perf buffer: 성숙, 다양한 예제 존재
- ring buffer: **낮은 오버헤드**, CO-RE 환경에서 선호

### 8.3 Tail calls
- 프로그램 체이닝으로 **복잡한 로직 분기** 구현, 인라인 한계 우회
- `BPF_MAP_TYPE_PROG_ARRAY` 이용

---

## 9. bpftool로 내부 들여다보기

```bash
# 로드된 프로그램/맵
sudo bpftool prog show
sudo bpftool map show

# 특정 프로그램 disasm
sudo bpftool prog dump xlated id <ID> | less

# 맵 내용
sudo bpftool map dump id <MAP_ID>

# bpffs 고정(pinning)
sudo mount -t bpf bpf /sys/fs/bpf/
sudo bpftool prog pin id <ID> /sys/fs/bpf/myprog
```

---

## 10. Kubernetes에서 eBPF: Cilium/Hubble

### 10.1 Cilium을 쓰는 이유
- **iptables 대체** 수준의 L3/L4 처리 + **L7 인식**
- 대규모 환경에서 **규칙 포크폭** 없이 선형 확장
- **Hubble**: 플로우/정책 이력, 레이턴시 관찰

### 10.2 Minikube 빠른 실습
```bash
minikube start --network-plugin=cni --cni=false
minikube addons enable cilium
cilium status
cilium hubble enable
cilium hubble ui &
```

- **네트워크 정책** 적용 후 Hubble로 허용/거부를 시각화 → 정책 안전성 향상
- **eBPF LB** 경로로 **SNAT/Conntrack 병목 완화**, CPU 절감

---

## 11. 성능 최적화 워크플로우(현업 레시피)

1) **증상 정의**: 지연/타임아웃/CPU 고사용/패킷 드롭 지표 확인(기존 APM/노드 Exporter)
2) **저오버헤드 스캔**:
   - CPU: `profile`(bpftrace) → 핫스택
   - I/O: `biolatency`, `fileslower`
   - 네트워크: `tcptop`, `softirqs`, XDP-drop 카운트
3) **가설 수립**: 커널/네트워크/애플리케이션 층 어디서 병목?
4) **미세 관측**: kprobe/tracepoint로 함수/경로별 분포 수집
5) **개선 시도**:
   - 네트워크: XDP/tc에서 **얼리 드롭/리다이렉트**, Cilium LB 경로
   - 스케줄: 배치/IRQ CPU affinity, `sysctl` 튜닝
   - 앱: syscalls/lock 경합 완화, 비효율 I/O 배치
6) **회귀 테스트**: before/after **동일 부하** 재현, 분포 비교
7) **상시 관측**: libbpf-tool 측정기를 데몬화, 슬랙/알람 연동

---

## 12. 실제 튜닝 예시 시나리오

### 12.1 고QPS API 서버의 짧은 타임아웃
- 현상: P99 지연 폭증, 재전송 증가
- 진단:
  - `tcptop` 으로 소켓 단위 상위 커넥션 확인
  - `softirqs` 로 NET_RX softirq 타임비용 급증
  - XDP 카운터/드롭 없음 → 커널 상단 아님
  - `runqlat` 에서 런큐 지연↑ → CPU 부족 + IRQ 편중
- 조치:
  - **RPS/RFS** 튜닝, IRQ affinity 재분배
  - Cilium **DSR or Maglev LB** 경로 확인
  - 애플리케이션 **Nagle/알고리즘 옵션** 및 타임아웃 재조정
- 결과: P95/99 **30~45% 단축**, 재전송 **50% 감소**

### 12.2 파일 서버의 꼬리 지연
- `biolatency` → 꼬리만 길다(헤드 OK)
- `fileslower` → 특정 경로/워크로드 집중
- 조치: IO 스케줄러/배치 크기, readahead 조정 → tail **40% 감소**

---

## 13. 안전성 & 보안

- **Verifier 제약**: 무한루프 불가, 경계검사 필수 → 프로그램은 **짧고 결정적**이어야 한다.
- **권한 관리**: BPF CAP 최소화, 컨테이너에서 BPF 사용은 보안 모델 검토
- **LSM(eBPF)**: 보안 정책을 eBPF로 구성 시, **정책 검증/롤백** 전략 필수
- **커널 호환성**: CO-RE 전제하되, **CI에 커널 매트릭스 테스트** 도입 권장

---

## 14. 실습: bpftrace 스크립트 3종

### 14.1 특정 바이너리의 디스크 읽기 지연
```bash
sudo bpftrace -e '
kprobe:vfs_read /comm == "myservice"/ { @start[tid] = nsecs; }
kretprobe:vfs_read /@start[tid]/ {
  $d = (nsecs - @start[tid]) / 1000000;
  @lat = hist($d);
  delete(@start[tid]);
}'
```

### 14.2 TCP 연결 수 상위 프로세스
```bash
sudo bpftrace -e '
tracepoint:inet:inet_sock_set_state /args->newstate == 1/ { @[comm] = count(); }'
```

### 14.3 스케줄 지연 히스토그램
```bash
sudo bpftrace -e '
tracepoint:sched:sched_wakeup { @ts[args->pid] = nsecs; }
tracepoint:sched:sched_switch /@ts[prev->pid]/ {
  $d = (nsecs - @ts[prev->pid]) / 1000000;
  @runq = hist($d);
  delete(@ts[prev->pid]);
}'
```

---

## 15. libbpf CO-RE 최소 예제(프로세스 fork 카운트)

### 15.1 BPF(C) — `fork_count.bpf.c`
```c
// fork_count.bpf.c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, u32);
    __type(value, u64);
    __uint(max_entries, 10240);
} counts SEC(".maps");

SEC("tracepoint:sched:sched_process_fork")
int TP_fork(struct trace_event_raw_sched_process_fork *ctx) {
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    u64 init = 1, *v = bpf_map_lookup_elem(&counts, &pid);
    if (!v) bpf_map_update_elem(&counts, &pid, &init, BPF_ANY);
    else __sync_fetch_and_add(v, 1);
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

### 15.2 유저(C) — `fork_count.c`
```c
// fork_count.c
#include <stdio.h>
#include <bpf/libbpf.h>
#include <unistd.h>
#include <signal.h>

static volatile int exiting;

static void on_sigint(int sig) { exiting = 1; }

int main() {
    struct bpf_object *obj;
    struct bpf_program *prog;
    int err;

    signal(SIGINT, on_sigint);

    obj = bpf_object__open_file("fork_count.bpf.o", NULL);
    if (libbpf_get_error(obj)) { perror("open"); return 1; }
    err = bpf_object__load(obj);
    if (err) { perror("load"); return 1; }

    prog = bpf_object__find_program_by_title(obj, "tracepoint/sched/sched_process_fork");
    int link_fd = bpf_program__attach_tracepoint(prog, "sched", "sched_process_fork");
    if (link_fd < 0) { perror("attach"); return 1; }

    printf("Collecting... Ctrl-C to stop\n");
    while (!exiting) sleep(1);

    // 맵 덤프(간단 표시)
    struct bpf_map *map = bpf_object__find_map_by_name(obj, "counts");
    int map_fd = bpf_map__fd(map);
    u32 key = 0, next_key; u64 val;
    while (bpf_map_get_next_key(map_fd, &key, &next_key) == 0) {
        if (bpf_map_lookup_elem(map_fd, &next_key, &val) == 0) {
            printf("pid=%u forks=%llu\n", next_key, (unsigned long long)val);
        }
        key = next_key;
    }
    return 0;
}
```

### 15.3 빌드
```bash
clang -O2 -g -target bpf -c fork_count.bpf.c -o fork_count.bpf.o
clang -O2 -g fork_count.c -o fork_count -lelf -lz -lbpf
sudo ./fork_count
```

> 실제 배포 시에는 `bpftool gen skeleton`, `libbpf-bootstrap` 템플릿을 활용해 스켈레톤 코드를 자동 생성하는 방식을 추천.

---

## 16. 수학적 관점: 꼬리 지연 관리

성능 최적화 목표는 평균이 아니라 **꼬리(Tail) 지연**이다. P99를 $$T_{99}$$ 라고 할 때,
$$
T_{99} \approx T_{net} + T_{sched} + T_{io} + T_{app}
$$
각 항의 분해 관측을 eBPF로 수행하고, **최대 항**을 우선 줄인다.
- `T_net`: XDP/tc 드롭·리다이렉트, IRQ 분배
- `T_sched`: run-queue, CPU 핫스팟 제거
- `T_io`: I/O path 지연, readahead/queue 깊이
- `T_app`: syscalls/lock 경합, 블로킹 외부 호출

---

## 17. 운영 팁 & 베스트프랙티스

- **한 번에 하나의 가설**만 검증: 관측→변경→검증 사이클 짧게
- **오버헤드 측정**: 관측 도구 자체의 CPU·메모리·latency 비용 검토
- **코드 리뷰/정책**: eBPF는 커널 권한에 준한다. **리뷰/서명/롤백 플랜** 필수
- **Helm/Ansible**로 배포 자동화, **GitOps**로 이력 관리
- **커널/도구 버전 매트릭스 테스트**: CI에서 CO-RE 호환성 확보
- **SLO 기반**: P95/99/99.9 타깃을 명시하고, 대시보드·알람과 연결

---

## 18. Kubernetes와의 결합: Cilium Playbook

1) Cilium + Hubble로 **흐름/정책** 가시화
2) eBPF LB 경로 활성화(기본) → **iptables 경합 제거**
3) **L7 정책**(HTTP/gRPC)로 외부 호출 제어
4) Hubble 지표로 **실제 서비스 경로** 기준 최적화(핫스팟 네임스페이스/서비스 식별)
5) Node/Pod별 **softirq, runqlat** 관측으로 핀포인트 튜닝

---

## 19. 자주 겪는 문제와 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| eBPF 로드 실패 `permission denied` | CAP 부족/LSM 제한 | root/CAP_BPF, seccomp/AppArmor 예외 |
| verifier reject | 경계 검사 불충분/루프 | 포인터 체이닝 검증, loop bound 명시 |
| 성능 저하 | 과도한 이벤트/버퍼 백프레셔 | 샘플링률 낮추기, ring buffer 크기 조정 |
| 커널 버전 호환 | 심볼/필드 차이 | CO-RE로 빌드, BTF 유효성 체크, 커널 매트릭스 CI |
| XDP 부착 실패 | 드라이버 미지원 | generic/native 모드 전환, 드라이버 업데이트 |

---

## 20. 마무리

- eBPF는 **모니터링 도구**를 넘어 **네트워크/보안/관측/정책**을 **저오버헤드로 커널 경로에 직접 적용**하게 한다.
- **bpftrace**로 빠르게 가설을 검증하고, **libbpf-tools/CO-RE**로 상시 관측을 구축하라.
- **XDP/tc**로 불필요 패킷을 빨리 걸러내고, **Cilium**으로 Kubernetes 네트워크를 선형 확장하라.
- 안전성(Verifier), 운영(권한/롤백/관측)을 체계화하면, **평균이 아닌 꼬리 지연**을 실질적으로 줄일 수 있다.

---

## 부록: 명령어 치트시트

```bash
# bpftrace: 빠른 가설 검증
sudo bpftrace -l 'tracepoint:*' | head
sudo bpftrace -e 'profile:hz:99 { @[kstack] = count(); }'

# BCC: 상용구 툴
sudo /usr/share/bcc/tools/biolatency -m 1
sudo /usr/share/bcc/tools/tcptop -C 1 10
sudo /usr/share/bcc/tools/runqlat 1 5

# bpftool: 내부 들여다보기
sudo bpftool prog show
sudo bpftool map show
sudo bpftool prog dump xlated id <ID> | less

# XDP: 드롭 예제 장착/해제
sudo ip link set dev eth0 xdp obj xdp_drop.o sec xdp
sudo ip link set dev eth0 xdp off

# Cilium/Hubble
cilium status
cilium hubble enable
hubble observe --from-pod ns/app
```

---

## 참고 링크

- eBPF: https://ebpf.io
- bpftrace: https://bpftrace.org
- BCC: https://github.com/iovisor/bcc
- libbpf-bootstrap: https://github.com/libbpf/libbpf-bootstrap
- bpftool: https://www.kernel.org/doc/html/latest/bpf/bpftool.html
- Cilium: https://cilium.io
- Hubble: https://docs.cilium.io/en/stable/gettingstarted/hubble/
