---
layout: post
title: Kubernetes - WASM, eBPF 흐름
date: 2025-06-13 20:20:23 +0900
category: Kubernetes
---
# 쿠버네티스와 WASM, eBPF 흐름 이해하기

Kubernetes(이하 K8s)는 컨테이너 오케스트레이션의 표준이지만, **콜드스타트**, **관측 사각지대**, **L7 정책 한계**, **플랫폼 이식성** 같은 과제가 남아 있습니다.
여기에 **WASM(WebAssembly)** 과 **eBPF(Extended BPF)** 가 더해지면, **실행(Execution)** 과 **관측/제어(Observability/Control)** 를 각각 업그레이드할 수 있습니다.

---

## 기존 K8s의 구조적 한계와 개선 목표

| 영역 | 기존 컨테이너 한계 | 개선 목표(요약) |
|---|---|---|
| 실행(Execution) | 이미지 크기/부팅 시간/아키 종속 | **WASM** 으로 경량·고속·이식성 |
| 관측/보안(Observe/Secure) | 유저 공간 한계, L3/L4 중심 | **eBPF** 로 커널 레벨 가시성·L7 정책 |
| 네트워크 | kube-proxy, iptables 성능 병목 | **eBPF** 기반 LB/정책으로 고성능화 |
| 멀티 아키 | x86/ARM 병행 빌드 부담 | **WASM** 바이트코드로 아키 독립 |

---

## — 컨테이너 이후의 실행 단위

### 핵심 개념

- **바이트코드 포맷** + **안전한 샌드박스 실행**(WASI로 시스템 인터페이스 표준화)
- **수 MB 런타임**, **ms급 콜드스타트**, **아키 독립(이식성)**, **보안 격리 강화**
- 서버리스·플러그인·엣지 런타임에 적합

### K8s 연동 방식(대표 패턴)

1) **containerd WASM shim**: OCI/Pod는 그대로, 컨테이너 자리에 WASM 실행
2) **Krustlet**: WASM 전용 kubelet(학습·PoC에 유용)
3) **서버리스 프레임워크**: Knative/OpenFaaS/Spin 등과 결합
4) **프록시/필터 WASM**: Envoy WASM Filter(네트워크 플러그인), 사이드카 축소

### Rust + WASI “Hello” (기초)

```rust
// src/main.rs
use std::env;
fn main() {
    let who = env::var("TARGET").unwrap_or_else(|_| "WASM".into());
    println!("Hello, {who}");
}
```

**빌드**
```bash
rustup target add wasm32-wasi
cargo build --release --target wasm32-wasi
# 결과물: target/wasm32-wasi/release/hello_wasi.wasm

```

### containerd + wasmtime shim으로 K8s에서 실행

**RuntimeClass 정의**
```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: wasmtime-wasi
handler: wasmtime
overhead: {}
scheduling: {}
```

**Pod 예제 (WASM 실행)**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-wasm
spec:
  runtimeClassName: wasmtime-wasi
  containers:
  - name: app
    image: ghcr.io/you/hello-wasi:wasm  # OCI 아티팩트로 퍼블리시한 wasm
    command: ["/hello_wasi.wasm"]
    env:
    - name: TARGET
      value: "Kubernetes"
```

> 주: 이미지 없이 `.wasm`을 OCI 아티팩트로 푸시할 수 있습니다(oras/oci artifacts).

### Spin(Fermyon)으로 서버리스형 WASM

**Spin.toml**
```toml
spin_version = "1"
name = "hello-spin"
trigger = { type = "http", base = "/" }

[[component]]
id = "hello"
source = "target/wasm32-wasi/release/hello_wasi.wasm"
[component.trigger]
route = "/"
```

로컬:
```bash
spin up
curl http://127.0.0.1:3000/
```

K8s 배포(예: Spin Operator 사용) 시, Pod 없이도 **WASM 함수형 배포**를 단순화할 수 있습니다.

---

## eBPF — 커널 레벨 관측/제어를 쿠버네티스에

### 핵심 개념

- **유저 공간에서 작성한 BPF 프로그램을 커널에 로드**해 특정 hook(XDP, tc, kprobe, tracepoint 등)에서 실행
- **Verifier**가 안전성 검사 → 커널 패닉 방지
- 저오버헤드로 **네트워크·보안·트레이싱·로깅**을 통합

### 대표 스택

- **Cilium**(CNI): eBPF로 kube-proxy 대체, L7 정책(HTTP/gRPC), 고성능 LB
- **Hubble**: eBPF 이벤트 기반 실시간 플로우 관측
- **Falco/Tracee**: 런타임 보안(시스템콜/행위 기반 탐지)
- **Pixie(px.dev)**: 프로브 없이 eBPF로 애플리케이션 가시성 확보

### bpftrace로 특정 컨테이너 sys_open 추적

```bash
# 얻은 뒤:

bpftrace -e 'tracepoint:syscalls:sys_enter_openat /cgroup == C/12345/ / { printf("%s %s\n", comm, str(args->filename)); }'
```
> 디버깅/포렌식 시 특정 Pod의 파일 접근 패턴을 즉시 확인 가능.

---

## Cilium로 eBPF 네트워킹·보안·관측 통합

### 설치(개요)

```bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set kubeProxyReplacement=strict \
  --set k8sServiceHost=<api-server-host> \
  --set k8sServicePort=6443 \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```
> `kubeProxyReplacement=strict` 로 iptables 대신 **eBPF LB** 를 사용.

### L7 HTTP 정책(메서드/Path 기반)

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: web-allow-readonly
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: web
  egress:
  - toEndpoints:
    - matchLabels:
        app: backend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "^/api/v1/items"
```
> 사이드카 없이 **L7 정책**을 적용(Envoy/istio 미도입 환경에서 경량).

### Hubble로 플로우 가시화

```bash
cilium hubble enable
hubble status
hubble observe --from-pod default/web-xxx --protocol http
```

---

## WASM + eBPF를 함께 쓰는 운영 패턴

### 서버리스 API는 WASM, 네트워크·보안은 eBPF

- **WASM(Spin/Knative WASM 런타임/wasmtime)** 으로 API 함수 구성 → 빠른 콜드스타트
- **Cilium** 으로 north-south LB/L7 정책, **Hubble** 로 플로우 관측

### 사이드카 없는 메시·관측

- 사이드카 대신 **Cilium + Hubble** 로 L7 관측/정책
- 트래픽 조절/카나리는 K8s 네이티브(Deployment/Ingress) 혹은 Gateway API 사용

### 엣지·이질 아키텍처 혼합

- ARM/x86 혼재 환경에서 애플리케이션을 **WASM** 으로 통일
- eBPF로 엣지 노드의 네트워크·보안·관측 일관성 보장

---

## 지연·용량 추정 (간단 모델)

서버리스 콜드스타트 비용을 포함한 기대 응답시간(ERT) 근사:
$$
ERT \approx p_{cold}\cdot T_{cold} + (1-p_{cold})\cdot T_{warm}
$$

- WASM은 일반 컨테이너 대비 **\(T_{cold}\)** 가 매우 작고, **\(p_{cold}\)**(콜드 비율)도 minScale, pre-warm 등으로 제어 가능.
- 요청률 \(\lambda\), Pod당 동시성 \(c\), 평균 처리시간 \(S\) 일 때 필요한 Pod 수:
$$
N \approx \left\lceil \frac{\lambda \cdot S}{c} \right\rceil
$$
Knative/OpenFaaS(KEDA) 튜닝 시 위 파라미터(c, target concurrency, window)를 근거로 초기값을 잡고 관측 기반으로 수렴시키면 된다.

---

## 실전 예제 모음

### K8s + WASM (containerd shim)로 간단 Echo

**Pod**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: echo-wasm
  labels: { app: echo }
spec:
  runtimeClassName: wasmtime-wasi
  containers:
  - name: echo
    image: ghcr.io/you/echo-wasm:0.1.0
    args: ["/echo.wasm", "--port", "8080"]
    ports:
    - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: echo-svc
spec:
  selector: { app: echo }
  ports:
  - port: 80
    targetPort: 8080
```

### Cilium L7 정책으로 Echo API 제한

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: echo-readonly
spec:
  endpointSelector:
    matchLabels: { app: echo }
  ingress:
  - toPorts:
    - ports: [{ port: "8080", protocol: TCP }]
      rules:
        http:
        - method: "GET"
          path: "^/echo"
```

### Falco로 의심스런 행동 탐지(컨테이너 내부 쉘 스폰)

```yaml
# /etc/falco/falco_rules.local.yaml

- rule: Terminal shell in container
  desc: Detect a shell spawned by a containerized application
  condition: container and shell_procs and evt.type = execve
  output: "Shell in container (user=%user.name command=%proc.cmdline container=%container.id)"
  priority: WARNING
```

### Pixie로 코드변경 없이 트레이싱

```bash
px deploy        # Pixie 플랫폼 설치
px run px/http_data --pods echo-wasm
px live          # 라이브 뷰/대시보드
```

---

## 보안·서플라이체인·거버넌스

- **WASM 모듈 서명/검증**: Sigstore Cosign으로 OCI 아티팩트 서명 → Admission Policy(OPA/Kyverno)로 검증
- **PSA(파드 보안 허용)**: `pod-security.kubernetes.io/enforce: restricted`
- **SPIFFE/SPIRE**: 워크로드 ID·mTLS로 서비스 간 신뢰도 강화
- **BPF CO-RE**: 커널 버전 차이 완화, bpftool로 디버그/검증

---

## 운영 체크리스트

| 항목 | WASM | eBPF |
|---|---|---|
| 성능 | 런타임 선택(Wasmtime/WasmEdge), 이미지 아님 → OCI 아티팩트 관리 | kube-proxy 제거 시 SNAT/DSR 동작 확인, MTU/ENCAP 튜닝 |
| 관측 | 로그·메트릭 어댑터(예: stdout→OTel) | Hubble/Pixie 대시보드, bpftrace on-demand |
| 보안 | 모듈 서명·정책, WASI 권한 최소화 | Falco/Tracee 룰 정제, BPF Program map/limit 모니터 |
| 배포 | RuntimeClass, GitOps(ArgoCD), Canary | Cilium 업그레이드 롤링, Policy CR 변경은 점진 적용 |
| 트러블슈팅 | `kubectl logs`, 런타임 디버깅 도구(spin/kwasm) | `cilium status`, `hubble observe`, `bpftool prog/show` |

---

## 의사결정 가이드

| 상황 | 권장 조합 |
|---|---|
| 서버리스 API/엣지·다중 아키 | **WASM + Knative/Spin** |
| 사이드카 없이 L7 정책/관측 | **Cilium + Hubble** |
| 런타임 보안(행위 기반) | **Falco/Tracee** |
| 코드수정 없이 트레이싱 | **Pixie** |
| 고성능 LB·kube-proxy 대체 | **Cilium kubeProxyReplacement** |

---

## 자주 만나는 이슈와 해법

- **WASM 네이밍/디스트로**: `.wasm` OCI 푸시 시 레지스트리 호환성 확인(oras, ghcr 권장)
- **RuntimeClass 미설정**: Pod가 일반 컨테이너 런타임으로 스케줄 → `RuntimeClass` 필수
- **Cilium 정책 미스매치**: L7 정책은 Service/Pod 포트·Host 헤더·SNI 매칭 확인
- **Hubble 데이터 빈약**: relay/ui enable, 허용 포트/네임스페이스 라벨링 점검
- **Falco 오탐**: 베이스라인 잡고 룰의 `condition` 을 환경에 맞게 줄이기

---

## 마무리

- **WASM** 은 컨테이너의 자리를 **보완/대체**하는 **경량 실행 단위**로, 서버리스·엣지·멀티아키를 강하게 밀어줍니다.
- **eBPF** 는 커널 레벨 **관측/보안/네트워크 제어**를 제공해 사이드카 부담을 줄이고 고성능 정책을 가능케 합니다.
- K8s 위에서 두 기술을 **상호보완적으로 결합**하면, **더 가볍고(실행)**, **더 잘 보이고(관측)**, **더 안전한(보안)** 플랫폼을 구축할 수 있습니다.

핵심은 **점진적 도입**입니다.
작은 서비스부터 WASM으로 전환, 네트워크부터 Cilium으로 관측·정책을 시작하세요.
관측이 충분해지면, 스케일·보안 정책을 데이터로 설계하고, 사이드카를 걷어내며 단순성을 향해 나아갈 수 있습니다.

---

## 참고 리소스

- WasmEdge: https://github.com/WasmEdge/WasmEdge
- Wasmtime: https://wasmtime.dev/
- Spin: https://www.fermyon.com/spin
- Cilium/Hubble: https://cilium.io/
- Pixie: https://px.dev/
- Falco: https://falco.org/
- Krustlet: https://github.com/krustlet/krustlet
- ORAS(OCI Artifacts): https://oras.land/
