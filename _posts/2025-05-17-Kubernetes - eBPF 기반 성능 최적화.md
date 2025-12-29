---
layout: post
title: Kubernetes - eBPF 기반 성능 최적화
date: 2025-05-11 20:20:23 +0900
category: Kubernetes
---
# eBPF 기반 성능 최적화

## eBPF를 활용해야 하는 상황

eBPF(Extended Berkeley Packet Filter)는 커널 수준에서 안전하게 프로그램을 실행할 수 있는 기술로, 시스템 성능 최적화에 효과적으로 활용할 수 있습니다. 다음과 같은 상황에서 eBPF를 사용하면 큰 이점을 얻을 수 있습니다.

- **네트워크 성능**: iptables 규칙이 과도하게 증가하거나 Conntrack 병목이 발생하는 경우 → eBPF를 사용하여 L3/L4 패킷 처리(XDP/tc)를 최적화하고, **Cilium**으로 CNI 가속화
- **관측 및 프로파일링**: CPU 핫스팟, 스케줄링 지연, I/O 지연 문제 해결 → **bpftrace/libbpf-tools**를 사용한 실시간 관측으로 **낮은 오버헤드** 유지
- **보안 강화**: 시스템 콜/프로세스 동작 정책 적용, 권한(CAP) 최소화 → **LSM(eBPF)** 기반 정책 구현, 프로세스 동작 추적
- **Kubernetes 환경**: **Hubble**을 통한 네트워크 흐름 시각화, L7 정책 적용, egress 트래픽 가속화, 대규모 노드 확장성 확보

---

## eBPF 핵심 개념 이해

### 실행 모델

- **프로그램 유형**: XDP, tc, kprobe/kretprobe, tracepoint, uprobe/uretprobe, perf_event, cgroup/lsm 등
- **맵(Maps) 유형**: `hash`, `array`, **per-CPU hash/array**, LRU map, ring/perf buffer, stack/map-of-maps, LPM trie
- **로드 및 검증**: 바이트코드를 `verifier`가 **메모리 안전성과 루프 한정성**을 검증한 후 JIT 컴파일러로 네이티브 코드로 변환
- **상호작용**: 커널 헬퍼 함수, BPF 맵을 통한 사용자 공간 통신, perf/ring buffer 이벤트 스트림

### CO-RE(BPF Compile-Once, Run-Everywhere)

- **libbpf + BTF** 기반으로 커널 심볼과 필드 변화를 런타임에 **재매칭** → 단일 바이너리로 **다양한 커널 버전** 호환
- 장점: **패키징과 배포 간소화**, **컨테이너 환경**에서의 이식성 향상

---

## 도구 선택 가이드

| 목적 | 권장 도구 | 비고 |
|---|---|---|
| 원인 분석 및 탐색(수분 내) | **bpftrace** | 원라이너 및 DSL 지원, 빠른 가설 검증에 적합 |
| 저오버헤드 상시 관측 | **libbpf-tools** | CO-RE 지원, 생산 환경 상주 에이전트용 |
| 학습 및 시작용 | **BCC**(bpfcc-tools) | Python 래퍼 제공, 배포 및 의존성이 다소 무거움 |
| 정책/네트워킹 | **Cilium** | eBPF 기반 CNI, Hubble 관측 기능 제공 |
| 빌드 및 디버깅 | **bpftool** | 맵/프로그램 핀닝/검증/덤프, 필수 유틸리티 |

> 처음 시작한다면 **bpftrace + libbpf-tools + bpftool** 조합이 가장 실용적입니다.

---

## 환경 준비

### 커널 및 패키지 요구사항

- 권장 커널 버전: **5.x 이상** (4.18+도 가능하나 기능 차이 존재)
- 배포판 패키지 설치 예시 (Ubuntu):
```bash
sudo apt-get update
sudo apt-get install -y linux-headers-$(uname -r) \
  bpfcc-tools bpftrace bpftool clang llvm libbpf-dev libelf-dev zlib1g-dev
```

### 권한 설정

- root 권한 권장. 컨테이너 환경에서는 CAP_BPF, CAP_PERFMON, CAP_SYS_ADMIN 등의 권한이 필요할 수 있습니다.

---

## 즉시 활용 가능한 관측 원라이너 (bpftrace)

### CPU 핫스팟 분석(함수 레벨)

```bash
sudo bpftrace -e 'profile:hz:99 { @[kstack] = count(); }' | head
```
- 99Hz 주기로 커널 스택을 샘플링하여 상위 호출 스택 히트맵 생성.

### 디스크 I/O 지연 분석

```bash
sudo bpftrace -e 'tracepoint:block:block_rq_complete { @lat[args->dev] = hist(args->nr_sector); }'
```

### 파일 읽기 지연 분포 분석

```bash
sudo bpftrace -e 'kprobe:vfs_read { @start[tid] = nsecs; } kretprobe:vfs_read /@start[tid]/ { @d = hist((nsecs-@start[tid])/1000000); delete(@start[tid]); }'
```

### TCP 재전송 관찰

```bash
sudo bpftrace -e 'tracepoint:tcp:tcp_retransmit_skb { @retx[comm] = count(); }'
```

---

## libbpf-tools를 이용한 상시 관측 (저오버헤드)

### 설치 방법(소스 기반)

```bash
git clone https://github.com/iovisor/bcc --depth 1
git clone https://github.com/iovisor/bpftrace --depth 1
git clone https://github.com/iovisor/bpf-perf-tools --depth 1
git clone https://github.com/iovisor/bcc/libbpf-tools --depth 1
```

> 배포판 패키지에 포함된 경우 `apt install bpftrace bpfcc-tools` 명령으로도 충분히 설치할 수 있습니다.

### 주요 도구 소개

| 도구 | 목적 |
|---|---|
| `runqlat` | 스케줄 큐 대기 지연 분석 |
| `biolatency` | 블록 I/O 지연 히스토그램 생성 |
| `tcptop` | TCP 송수신 상위 프로세스 확인 |
| `softirqs` | 소프트IRQ 시간 분해 분석 |
| `memleak` | 잠재적 메모리 누수 추적(실험적) |

사용 예시:
```bash
sudo /usr/share/bcc/tools/biolatency -m 1
sudo /usr/share/bcc/tools/tcptop -C 1 10
```

---

## 네트워킹 가속화: XDP & tc-BPF

### XDP를 통한 패킷 드롭/리다이렉트

- **NIC 드라이버** 단계에서 BPF 프로그램 실행 → **패킷을 가장 빠른 경로에서 처리**
- 사용 사례: DDoS 방어, L3/L4 로드 밸런싱, 빠른 패킷 드롭/리다이렉트

#### 기본 XDP 드롭 프로그램 예제(C 언어)

```c
// xdp_drop.c (CO-RE 전용은 아님: 데모용)
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_drop_all(struct xdp_md *ctx) {
    return XDP_DROP;
}
char LICENSE[] SEC("license") = "GPL";
```
빌드 및 적용:
```bash
clang -O2 -target bpf -c xdp_drop.c -o xdp_drop.o
# 네트워크 인터페이스에 프로그램 부착

sudo ip link set dev eth0 xdp obj xdp_drop.o sec xdp
# 프로그램 제거

sudo ip link set dev eth0 xdp off
```

#### 조건부 패킷 드롭(특정 포트 차단)

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

### tc egress/ingress BPF

- L2 처리 이후, qdisc 레이어에서 TC BPF를 사용한 **정교한 패킷 분류 및 리라이트** 가능
- **XDP → tc** 파이프라인 구성으로 단계적 패킷 처리 구현

적용 예시:
```bash
# tc qdisc 준비

sudo tc qdisc add dev eth0 clsact
# ingress에 BPF 프로그램 부착

sudo tc filter add dev eth0 ingress bpf obj tc_prog.o sec cls_main
```

---

## 시스템 콜/스택 관측: kprobe/tracepoint/uprobes

### kprobe 활용

커널 함수 진입 및 복귀 시점 후킹:
```bash
sudo bpftrace -e 'kprobe:__x64_sys_openat { printf("%s %s\n", comm, str(arg1)); }'
```

### tracepoint 활용

안정적인 ABI를 가진 커널 이벤트 추적:
```bash
sudo bpftrace -e 'tracepoint:sched:sched_switch { @[prev_comm, next_comm] = count(); }'
```

### uprobes 활용

사용자 공간 바이너리의 함수 진입 및 복귀 후킹:
```bash
sudo bpftrace -e 'uprobe:/usr/bin/nginx:ngx_http_process_request_line { @[comm] = count(); }'
```

> 프로덕션 환경에서는 **심볼 해석 및 ASLR 고려**가 필요합니다. `nm`, `objdump`, `readelf` 도구를 사용하여 오프셋을 확인하세요.

---

## 맵, 버퍼 및 성능 고려사항

### per-CPU 맵 활용

- 글로벌 락 경합 없이 CPU별 슬롯을 사용 → **통계 수집에 매우 효율적**
- 사용자 공간에서 결과를 합산하여 처리

### perf buffer vs ring buffer

- perf buffer: 성숙한 기술, 다양한 예제 존재
- ring buffer: **낮은 오버헤드**, CO-RE 환경에서 선호되는 선택지

### Tail calls 활용

- 프로그램 체이닝으로 **복잡한 로직 분기** 구현, 인라인 한계 우회
- `BPF_MAP_TYPE_PROG_ARRAY` 맵 타입 활용

---

## bpftool을 통한 내부 구조 분석

```bash
# 로드된 프로그램 및 맵 확인

sudo bpftool prog show
sudo bpftool map show

# 특정 프로그램 디스어셈블

sudo bpftool prog dump xlated id <ID> | less

# 맵 내용 확인

sudo bpftool map dump id <MAP_ID>

# bpffs에 프로그램 고정(pinning)

sudo mount -t bpf bpf /sys/fs/bpf/
sudo bpftool prog pin id <ID> /sys/fs/bpf/myprog
```

---

## Kubernetes에서의 eBPF 활용: Cilium/Hubble

### Cilium을 선택하는 이유

- **iptables 대체** 수준의 L3/L4 처리 능력 + **L7 인식** 기능
- 대규모 환경에서 **규칙 폭증 없이** 선형적 확장 가능
- **Hubble**: 네트워크 흐름 및 정책 이력, 레이턴시 관찰 기능 제공

### Minikube를 활용한 빠른 실습

```bash
minikube start --network-plugin=cni --cni=false
minikube addons enable cilium
cilium status
cilium hubble enable
cilium hubble ui &
```

- **네트워크 정책** 적용 후 Hubble을 통해 허용/거별된 트래픽 시각화 → 정책 안전성 향상
- **eBPF 로드 밸런싱** 경로로 **SNAT/Conntrack 병목 완화**, CPU 사용량 절감

---

## 성능 최적화 워크플로우(실무 레시피)

1) **증상 정의**: 지연/타임아웃/CPU 고사용/패킷 드롭 지표 확인(기존 APM/노드 Exporter 활용)
2) **저오버헤드 스캔**:
   - CPU: `profile`(bpftrace) → 핫스택 식별
   - I/O: `biolatency`, `fileslower`
   - 네트워크: `tcptop`, `softirqs`, XDP-drop 카운트 확인
3) **가설 수립**: 커널/네트워크/애플리케이션 계층 중 어디에서 병목이 발생하는지 분석
4) **상세 관측**: kprobe/tracepoint를 사용한 함수/경로별 분포 데이터 수집
5) **개선 시도**:
   - 네트워크: XDP/tc에서 **조기 패킷 드롭/리다이렉트**, Cilium LB 경로 최적화
   - 스케줄링: 배치/IRQ CPU affinity, `sysctl` 튜닝
   - 애플리케이션: 시스템 콜/락 경합 완화, 비효율적 I/O 배치 개선
6) **회귀 테스트**: before/after **동일 부하** 재현, 분포 비교
7) **상시 관측**: libbpf-tool 측정기를 데몬화, 슬랙/알림 시스템과 연동

---

## 실제 튜닝 시나리오 예시

### 고QPS API 서버의 짧은 타임아웃 문제

- 현상: P99 지연 급증, 재전송 증가
- 진단:
  - `tcptop`으로 소켓 단위 상위 연결 확인
  - `softirqs`로 NET_RX softirq 시간 비용 급증 확인
  - XDP 카운터/드롭 없음 → 커널 상단 문제 아님
  - `runqlat`에서 런큐 지연 증가 → CPU 부족 + IRQ 편중
- 조치:
  - **RPS/RFS** 튜닝, IRQ affinity 재분배
  - Cilium **DSR 또는 Maglev LB** 경로 확인
  - 애플리케이션 **Nagle 알고리즘/옵션** 및 타임아웃 재조정
- 결과: P95/99 **30~45% 단축**, 재전송 **50% 감소**

### 파일 서버의 꼬리 지연 문제

- `biolatency` → 꼬리 지연만 길게 나타남(헤드는 정상)
- `fileslower` → 특정 경로/워크로드에 집중된 문제 확인
- 조치: I/O 스케줄러/배치 크기, readahead 설정 조정 → 꼬리 지연 **40% 감소**

---

## 안전성 및 보안 고려사항

- **검증기(Verifier) 제약**: 무한 루프 불가, 경계 검사 필수 → 프로그램은 **짧고 결정적**이어야 합니다.
- **권한 관리**: BPF CAP 최소화, 컨테이너 환경에서 BPF 사용 시 보안 모델 검토 필수
- **LSM(eBPF)**: eBPF 기반 보안 정책 구성 시, **정책 검증 및 롤백** 전략 필수
- **커널 호환성**: CO-RE 기반으로 구축하되, **CI 파이프라인에 커널 매트릭스 테스트** 도입 권장

---

## 실습: bpftrace 스크립트 예시

### 특정 프로세스의 디스크 읽기 지연 분석

```bash
sudo bpftrace -e '
kprobe:vfs_read /comm == "myservice"/ { @start[tid] = nsecs; }
kretprobe:vfs_read /@start[tid]/ {
  $d = (nsecs - @start[tid]) / 1000000;
  @lat = hist($d);
  delete(@start[tid]);
}'
```

### TCP 연결 수 상위 프로세스 확인

```bash
sudo bpftrace -e '
tracepoint:inet:inet_sock_set_state /args->newstate == 1/ { @[comm] = count(); }'
```

### 스케줄링 지연 히스토그램 생성

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

## libbpf CO-RE 최소 예제 (프로세스 fork 카운트)

### — `fork_count.bpf.c`

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

### — `fork_count.c`

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

    // 맵 데이터 덤프(간단한 표시)
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

### 빌드 및 실행

```bash
clang -O2 -g -target bpf -c fork_count.bpf.c -o fork_count.bpf.o
clang -O2 -g fork_count.c -o fork_count -lelf -lz -lbpf
sudo ./fork_count
```

> 실제 배포 환경에서는 `bpftool gen skeleton`, `libbpf-bootstrap` 템플릿을 활용하여 스켈레톤 코드를 자동 생성하는 방식을 권장합니다.

---

## 수학적 관점: 꼬리 지연 관리

성능 최적화의 주요 목표는 평균이 아닌 **꼬리(Tail) 지연** 관리입니다. P99 지연 시간을 $$T_{99}$$ 라고 할 때, 다음과 같은 구성 요소로 분해할 수 있습니다.
$$
T_{99} \approx T_{net} + T_{sched} + T_{io} + T_{app}
$$
각 구성 요소의 분해 관측을 eBPF로 수행하고, **가장 큰 영향을 미치는 요소**부터 우선적으로 줄이는 것이 효과적입니다.
- `T_net`: XDP/tc 패킷 드롭·리다이렉트, IRQ 분배 최적화
- `T_sched`: 런큐(run-queue), CPU 핫스팟 제거
- `T_io`: I/O 경로 지연, readahead/큐 깊이 최적화
- `T_app`: 시스템 콜/락 경합, 블로킹 외부 호출 최소화

---

## 운영 팁 및 모범 사례

- **한 번에 하나의 가설만 검증**: 관측 → 변경 → 검증 사이클을 짧게 유지
- **오버헤드 측정**: 관측 도구 자체의 CPU·메모리·지연 비용 검토
- **코드 리뷰 및 정책**: eBPF는 커널 권한에 준하므로 **코드 리뷰/서명/롤백 계획** 필수
- **Helm/Ansible을 통한 배포 자동화**, **GitOps를 통한 이력 관리**
- **커널 및 도구 버전 매트릭스 테스트**: CI 파이프라인에서 CO-RE 호환성 보장
- **SLO 기반 접근**: P95/99/99.9 목표치를 명시하고, 대시보드 및 알림 시스템과 연동

---

## Kubernetes와의 통합: Cilium 운영 가이드

1) Cilium + Hubble로 **네트워크 흐름 및 정책** 가시화
2) eBPF 로드 밸런싱 경로 활성화(기본값) → **iptables 경합 제거**
3) **L7 정책**(HTTP/gRPC)으로 외부 호출 제어
4) Hubble 메트릭을 활용한 **실제 서비스 경로 기반 최적화**(핫스팟 네임스페이스/서비스 식별)
5) Node/Pod별 **softirq, runqlat** 관측으로 정밀 튜닝

---

## 자주 발생하는 문제 및 해결 방법

| 문제 | 원인 | 해결 방법 |
|---|---|---|
| eBPF 로드 실패 `permission denied` | 권한(CAP) 부족/LSM 제한 | root/CAP_BPF 권한 부여, seccomp/AppArmor 예외 추가 |
| 검증기(verifier) 거부 | 경계 검사 불충분/루프 문제 | 포인터 체이닝 검증 강화, loop bound 명시적 설정 |
| 성능 저하 | 과도한 이벤트/버퍼 백프레셔 | 샘플링율 낮추기, ring buffer 크기 조정 |
| 커널 버전 호환 문제 | 심볼/필드 차이 | CO-RE 기반 빌드, BTF 유효성 확인, 커널 매트릭스 CI 구축 |
| XDP 부착 실패 | 드라이버 미지원 | generic/native 모드 전환, 드라이버 업데이트 |

---

## 결론

eBPF는 단순한 **모니터링 도구**를 넘어 **네트워크, 보안, 관측, 정책**을 **낮은 오버헤드로 커널 경로에 직접 적용**할 수 있는 강력한 기술입니다.

**bpftrace**를 활용하여 빠르게 가설을 검증하고, **libbpf-tools/CO-RE**를 통해 상시 관측 시스템을 구축하세요. **XDP/tc**를 사용하여 불필요한 패킷을 조기에 필터링하고, **Cilium**을 통해 Kubernetes 네트워크를 선형적으로 확장할 수 있습니다.

안전성(검증기), 운영(권한/롤백/관측)을 체계적으로 관리하면, **평균이 아닌 꼬리 지연**을 실질적으로 줄일 수 있으며, 이는 최종 사용자 경험과 시스템 안정성에 직접적인 영향을 미칩니다.

---

## 부록: 명령어 치트시트

```bash
# bpftrace: 빠른 가설 검증

sudo bpftrace -l 'tracepoint:*' | head
sudo bpftrace -e 'profile:hz:99 { @[kstack] = count(); }'

# BCC: 표준 도구 활용

sudo /usr/share/bcc/tools/biolatency -m 1
sudo /usr/share/bcc/tools/tcptop -C 1 10
sudo /usr/share/bcc/tools/runqlat 1 5

# bpftool: 내부 구조 분석

sudo bpftool prog show
sudo bpftool map show
sudo bpftool prog dump xlated id <ID> | less

# XDP: 드롭 프로그램 장착/제거

sudo ip link set dev eth0 xdp obj xdp_drop.o sec xdp
sudo ip link set dev eth0 xdp off

# Cilium/Hubble

cilium status
cilium hubble enable
hubble observe --from-pod ns/app
```