---
layout: post
title: Kubernetes - eBPF 기반 성능 최적화
date: 2025-05-11 20:20:23 +0900
category: Kubernetes
---
# eBPF 기반 성능 최적화

최근 클라우드 네이티브 인프라에서는 **eBPF(extended Berkeley Packet Filter)**를 통해  
시스템의 성능을 비약적으로 향상시키고 있습니다.

eBPF는 기존 리눅스 커널을 **수정 없이** 확장할 수 있는 기술로,  
**네트워킹, 보안, 트레이싱, 모니터링** 등 다양한 분야에서 활용되고 있습니다.

---

## ✅ eBPF란?

- **커널 공간에서 유저 정의 프로그램을 실행**할 수 있게 해주는 기술
- 원래는 `tcpdump`나 `iptables` 등에 쓰이던 BPF에서 발전
- **커널 내부 함수나 이벤트에 직접 후킹**하여 데이터를 추적하고 처리 가능

> eBPF 프로그램은 "가상 머신" 형태로 커널에서 실행되며,  
> 안전성 검증(Verifier)을 거쳐 로드됩니다.

---

## ✅ eBPF 특징

| 특징 | 설명 |
|------|------|
| 성능 | 커널 모드에서 실행되어 오버헤드 최소화 |
| 안전성 | 검증된 바이트코드만 실행되므로 시스템 안정성 보장 |
| 범용성 | 네트워크, 파일 I/O, 스케줄링 등 다양한 영역 추적 가능 |
| 동적 삽입 | 커널 재빌드나 리부팅 없이 동작 |

---

## ✅ eBPF가 성능 최적화에 기여하는 영역

1. **네트워크 트래픽 가속**
   - L3/L4 Load Balancing (ex: Cilium)
   - DDoS 방어, 패킷 필터링

2. **시스템 호출 분석**
   - `syscall`, `block I/O`, `TCP` 등 모니터링을 통한 병목 탐지

3. **CPU/메모리 트레이싱**
   - 어떤 프로세스가 CPU를 많이 사용하는지 확인
   - 캐시 미스, context switch 분석

4. **파일/디스크 I/O 분석**
   - 자주 호출되는 파일/디스크 경로, 시간 측정

5. **Kubernetes 네트워크 최적화**
   - CNI 플러그인 대체 (Cilium)
   - Pod 간 네트워크 경로 최적화

---

## ✅ 실전: BCC와 bpftrace 도구 사용

eBPF의 활용은 아래 도구들로 쉽게 접근 가능합니다.

### 1. BCC (BPF Compiler Collection)

```bash
sudo apt install bpfcc-tools linux-headers-$(uname -r)
```

#### 예제: 어떤 파일이 가장 많이 읽히는지 확인

```bash
sudo opensnoop-bpfcc
```

#### 예제: 실행 중인 프로세스의 TCP 연결 추적

```bash
sudo tcplife-bpfcc
```

---

### 2. bpftrace (고급 추적 스크립트)

```bash
sudo apt install bpftrace
```

#### 예제: 특정 syscall 호출 횟수

```bash
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { @[comm] = count(); }'
```

#### 예제: read() syscall의 지연 시간 측정

```bash
sudo bpftrace -e 'kprobe:vfs_read { @latency = hist(nsecs); }'
```

---

## ✅ Kubernetes와 eBPF: Cilium

[Cilium](https://cilium.io)은 eBPF를 기반으로 한 Kubernetes CNI (Container Network Interface)입니다.

| 항목 | 설명 |
|------|------|
| 트래픽 가시화 | Hubble로 흐름 추적 가능 |
| 고성능 네트워킹 | iptables 대신 eBPF로 처리 |
| L7 정책 지원 | HTTP, gRPC 기반 제어 |
| 클러스터 보안 | 암호화, 정책 기반 통제 |

### 예: 기존 iptables 기반의 네트워크 문제

- 성능 저하 (rule 수가 많아질수록 느려짐)
- 가시성 부족 (누가 누구에게 요청하는지 모호함)

→ Cilium은 **eBPF를 이용해 커널에서 직접 처리**, rule-less하고 빠름

---

## ✅ 실전 예제: Cilium 설치 및 간단 테스트

### 1. Cilium 설치 (Minikube 예시)

```bash
minikube start --network-plugin=cni --cni=false
minikube addons enable cilium
```

### 2. 흐름 가시화 도구: Hubble

```bash
cilium hubble enable
cilium hubble ui
```

### 3. 흐름 확인

```bash
cilium hubble observe
```

→ Pod 간 트래픽, L7 요청 내용 확인 가능

---

## ✅ 성능 최적화 적용 사례

| 분야 | 적용 사례 |
|------|-----------|
| Netflix | CPU 사용량 추적 및 최적화 |
| Facebook | TCP retransmission 분석 |
| Google | GKE 네트워크 모니터링 |
| Cloudflare | L4-L7 방화벽 및 DDoS 대응 |
| Isovalent (Cilium) | K8s 네트워크 전환으로 수천 노드까지 확장 가능 |

---

## ✅ 주의사항

- 최신 커널 필요 (`4.18+`, `5.x` 권장)
- 일부 커널 기능이 비활성화되면 eBPF 로딩 실패
- 보안 정책에 따라 eBPF 기능 제한 가능 (`seccomp`, `AppArmor`)

---

## ✅ 결론

| 항목 | 설명 |
|------|------|
| eBPF | 커널에서 유저 정의 로직 실행 |
| 성능 장점 | 커널 모드 직접 실행으로 오버헤드 없음 |
| 대표 도구 | bpftrace, BCC, Cilium |
| Kubernetes 활용 | 고성능 CNI, L7 정책, 흐름 추적 |
| 사용 조건 | 최신 커널, 검증된 코드, 보안 정책 주의 |

eBPF는 단순한 모니터링 도구를 넘어,  
**운영 효율성과 성능 개선, 보안 강화까지 아우르는 핵심 기술**입니다.

---

## ✅ 참고 링크

- [eBPF 공식 소개](https://ebpf.io/)
- [BCC GitHub](https://github.com/iovisor/bcc)
- [bpftrace 공식 사이트](https://bpftrace.org/)
- [Cilium 공식 사이트](https://cilium.io/)
- [Hubble observability](https://docs.cilium.io/en/latest/gettingstarted/hubble/)