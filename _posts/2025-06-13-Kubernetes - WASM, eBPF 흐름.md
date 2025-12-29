---
layout: post
title: Kubernetes - WASM, eBPF 흐름
date: 2025-06-13 20:20:23 +0900
category: Kubernetes
---
# 쿠버네티스와 WASM, eBPF 흐름 이해하기

Kubernetes(이하 K8s)는 컨테이너 오케스트레이션의 표준이 되었지만, **콜드스타트**, **관측 사각지대**, **L7 정책의 한계**, **플랫폼 이식성**과 같은 근본적인 과제는 여전히 남아 있습니다.
이러한 과제를 해결하기 위해 **WASM(WebAssembly)** 과 **eBPF(Extended BPF)** 기술이 주목받고 있으며, 각각 **애플리케이션 실행**과 **시스템 관측/제어** 영역을 혁신적으로 업그레이드할 수 있는 가능성을 제시합니다.

---

## 기존 K8s의 구조적 한계와 개선 목표

| 영역 | 기존 컨테이너 한계 | 개선 목표 |
|---|---|---|
| 실행(Execution) | 이미지 크기/부팅 시간/아키텍처 종속성 | **WASM**을 통한 경량화, 고속 실행, 뛰어난 이식성 |
| 관측/보안(Observe/Secure) | 유저 공간 모니터링의 한계, L3/L4 중심 정책 | **eBPF**를 활용한 커널 레벨 가시성 확보 및 L7 정책 적용 |
| 네트워크 | kube-proxy, iptables 기반의 성능 병목 | **eBPF** 기반 로드밸런싱과 정책 적용으로 고성능 구현 |
| 멀티 아키텍처 | x86/ARM 병행 빌드 및 관리 부담 | **WASM** 바이트코드를 통한 아키텍처 독립성 확보 |

---

## WASM — 컨테이너 이후의 실행 단위

### 핵심 개념
WASM은 **표준화된 바이트코드 포맷**과 **안전한 샌드박스 실행 환경**(WASI를 통해 시스템 인터페이스 표준화)을 제공합니다. 수 MB 크기의 경량 런타임, ms 단위의 콜드스타트, 아키텍처 독립성, 강화된 보안 격리 등의 특징으로 서버리스, 플러그인 시스템, 엣지 컴퓨팅 등의 시나리오에 매우 적합합니다.

### K8s 연동 방식(대표 패턴)
1. **containerd WASM shim**: 기존 OCI/Pod 스펙을 유지한 채, 컨테이너 자리에 WASM 모듈을 실행합니다.
2. **Krustlet**: WASM 워크로드 전용 kubelet으로, 학습이나 개념 검증(PoC)에 유용합니다.
3. **서버리스 프레임워크 통합**: Knative, OpenFaaS, Fermyon Spin 등과 결합하여 함수 단위 배포를 단순화합니다.
4. **프록시/필터 플러그인**: Envoy WASM Filter처럼 네트워크 플러그인으로 사용하거나, 경량화된 사이드카 역할을 할 수 있습니다.

### Rust + WASI 기반 간단한 예제
```rust
// src/main.rs
use std::env;
fn main() {
    let who = env::var("TARGET").unwrap_or_else(|_| "WASM".into());
    println!("Hello, {who}");
}
```

**빌드 명령**
```bash
rustup target add wasm32-wasi
cargo build --release --target wasm32-wasi
# 결과물: target/wasm32-wasi/release/hello_wasi.wasm
```

### containerd + wasmtime shim으로 K8s에서 실행하기
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

**WASM을 실행하는 Pod 예제**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-wasm
spec:
  runtimeClassName: wasmtime-wasi
  containers:
  - name: app
    image: ghcr.io/you/hello-wasi:wasm  # OCI 아티팩트로 퍼블리시한 WASM 모듈
    command: ["/hello_wasi.wasm"]
    env:
    - name: TARGET
      value: "Kubernetes"
```
> 참고: 별도의 컨테이너 이미지 없이 `.wasm` 파일 자체를 OCI 아티팩트로 관리하고 배포할 수 있습니다(oras 도구 활용).

---

## eBPF — 커널 레벨 관측과 제어를 쿠버네티스에 통합

### 핵심 개념
eBPF는 **유저 공간에서 작성한 안전한 프로그램을 커널에 로드**하여 특정 Hook(예: XDP, tc, kprobe)에서 실행하는 기술입니다. 커널의 Verifier에 의해 안전성이 검증되기 때문에 커널 패닉 위험 없이, 매우 낮은 오버헤드로 **네트워크 처리, 보안 모니터링, 트레이싱, 성능 분석** 등을 통합적으로 수행할 수 있습니다.

### 주요 도구 및 프로젝트
- **Cilium(CNI)**: eBPF 기반으로 kube-proxy를 대체하고, L7(HTTP/gRPC) 수준의 네트워크 정책과 고성능 로드밸런싱을 제공합니다.
- **Hubble**: eBPF에서 수집한 네트워크 플로우 이벤트를 기반으로 실시간 가시성을 제공하는 관측 도구입니다.
- **Falco/Tracee**: 시스템 콜 및 커널 이벤트를 모니터링하여 런타임 보안 위협을 탐지합니다.
- **Pixie(px.dev)**: 애플리케이션 코드 변경 없이 eBPF를 통해 자동으로 텔레메트리 데이터를 수집하고 가시성을 제공합니다.

### bpftrace를 이용한 실시간 모니터링 예시
특정 컨테이너의 파일 접근을 추적하려면 다음과 같이 실행할 수 있습니다.
```bash
# 컨테이너의 cgroup ID를 확인한 후
bpftrace -e 'tracepoint:syscalls:sys_enter_openat /cgroup == C/12345/ / { printf("%s %s\n", comm, str(args->filename)); }'
```
이를 통해 특정 Pod의 파일 시스템 접근 패턴을 즉시 확인할 수 있어 디버깅이나 보안 포렌식에 유용합니다.

---

## Cilium을 통한 eBPF 기반 네트워킹, 보안, 관측 통합

### 설치 개요
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
`kubeProxyReplacement=strict` 설정을 통해 기존의 iptables 기반 로드밸런싱 대신 **eBPF 기반의 고성능 로드밸런서**를 사용하게 됩니다.

### L7 HTTP 정책 적용 예시
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
이렇게 설정하면, Envoy나 Istio와 같은 사이드카 프록시를 도입하지 않아도 **L7 수준의 세밀한 트래픽 제어 정책**을 적용할 수 있습니다.

### Hubble를 통한 네트워크 플로우 가시화
```bash
cilium hubble enable
hubble status
hubble observe --from-pod default/web-xxx --protocol http
```

---

## WASM과 eBPF의 협업 운영 패턴

### 패턴 1: 실행과 관측/제어의 분리
- **애플리케이션 실행(WASM)**: Spin, Knative의 WASM 런타임 등을 활용해 API 함수를 구성합니다. 콜드스타트가 빠르고 리소스 소비가 적습니다.
- **네트워킹 및 보안(eBPF)**: Cilium을 통해 north-south/east-west 트래픽의 로드밸런싱, L7 정책 적용, Hubble를 통한 실시간 관측을 담당합니다.

### 패턴 2: 사이드카 없는 서비스 메시 및 관측
- 서비스 메시의 복잡한 사이드카 프록시 대신 **Cilium의 eBPF 기반 L7 관측과 정책**을 활용합니다.
- 트래픽 라우팅, 카나리 배포 등은 K8s 네이티브 리소스(Deployment, Service, Ingress)나 Gateway API를 사용하여 관리합니다.

### 패턴 3: 이기종 엣지 환경 통합
- ARM, x86 등 아키텍처가 혼재된 엣지 환경에서 애플리케이션을 **WASM 바이트코드로 통일**하여 빌드 및 배포 복잡성을 해소합니다.
- 모든 노드에는 **eBPF 기반의 통일된 네트워크 스택, 보안 정책, 관측 도구**를 배치하여 일관된 운영 체계를 유지합니다.

---

## 성능 및 용량 계획 고려사항

서버리스 워크로드를 운영할 때는 콜드스타트를 고려한 응답시간 예측이 중요합니다. 기대 응답시간(ERT)은 다음과 같이 근사할 수 있습니다.

$$
ERT \approx p_{cold}\cdot T_{cold} + (1-p_{cold})\cdot T_{warm}
$$

- **WASM**은 일반 컨테이너에 비해 콜드스타트 시간(`T_cold`)이 훨씬 짧습니다. 또한 `minScale`, pre-warming 등을 통해 콜드스타트가 발생할 확률(`p_cold`)을 관리할 수 있습니다.
- 필요한 Pod 수를 예측하려면 요청률(`λ`), Pod당 동시 처리 수(`c`), 평균 요청 처리 시간(`S`)을 고려합니다.
$$
N \approx \left\lceil \frac{\lambda \cdot S}{c} \right\rceil
$$
Knative나 OpenFaaS(KEDA)를 튜닝할 때는 위 파라미터들을 기반으로 초기 값을 설정하고, 실제 관측 데이터를 바탕으로 점진적으로 조정해 나가는 것이 좋습니다.

---

## 실전 적용 예시

### 예시 1: K8s에서 WASM 기반 Echo 서버 실행
**Pod 및 Service 정의**
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

### 예시 2: Cilium L7 정책으로 API 접근 제한
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

### 예시 3: Falco를 이용한 컨테이너 내 이상 행위 탐지
```yaml
# /etc/falco/falco_rules.local.yaml
- rule: Terminal shell in container
  desc: Detect a shell spawned by a containerized application
  condition: container and shell_procs and evt.type = execve
  output: "Shell in container (user=%user.name command=%proc.cmdline container=%container.id)"
  priority: WARNING
```

### 예시 4: Pixie를 통한 코드 변경 없는 애플리케이션 트레이싱
```bash
px deploy        # Pixie 플랫폼 설치
px run px/http_data --pods echo-wasm
px live          # 실시간 트래픽 대시보드 확인
```

---

## 보안 및 거버넌스 고려사항

- **WASM 모듈 무결성**: Sigstore의 Cosign을 사용해 OCI 아티팩트에 서명하고, Kyverno나 OPA 같은 정책 엔진을 Admission Controller에 활용하여 검증하는 패턴을 적용합니다.
- **파드 보안 표준**: Pod Security Admission(PSA)을 `restricted` 등급으로 적용하여 기본 보안 수준을 강화합니다.
- **워크로드 신원**: SPIFFE/SPIRE를 도입해 워크로드 간 mTLS를 구현하고 신뢰 경계를 명확히 합니다.
- **eBPF 프로그램 관리**: BPF CO-RE(Compile Once – Run Everywhere)를 활용해 커널 버전 차이를 해소하고, bpftool 등을 통해 로드된 프로그램을 검증 및 모니터링합니다.

---

## 기술 선택 가이드라인

| 시나리오 | 권장 기술 조합 |
|---|---|
| 서버리스 API 또는 이기종 엣지 환경 | **WASM + Knative/Spin** |
| 사이드카 없이 L7 트래픽 제어 및 관측 필요 | **Cilium + Hubble** |
| 런타임 보안(시스템 콜/행위 기반 탐지) | **Falco** 또는 **Tracee** |
| 애플리케이션 코드 수정 없이 심층 트레이싱 | **Pixie** |
| 고성능 로드밸런싱 및 kube-proxy 대체 | **Cilium (kubeProxyReplacement)** |

---

## 자주 발생하는 이슈와 해결 방안

- **WASM 모듈 배포 실패**: `.wasm` 파일을 OCI 레지스트리에 푸시할 때 레지스트리 호환성을 확인하세요. ORAS 도구와 GitHub Container Registry(ghcr.io)를 사용하는 것을 권장합니다.
- **Pod가 WASM 런타임으로 스케줄되지 않음**: 반드시 올바른 `RuntimeClass`를 정의하고 Pod 스펙에서 참조했는지 확인하세요.
- **Cilium L7 정책이 동작하지 않음**: 정책의 `endpointSelector` 라벨, Service/Port 설정, HTTP 호스트 헤더 또는 TLS SNI 매칭 조건을 재점검하세요.
- **Hubble에서 데이터가 보이지 않음**: Hubble relay와 UI 컴포넌트가 활성화되었는지, 필요한 네트워크 포트가 열려 있는지, 네임스페이스 필터링 설정을 확인하세요.
- **Falco에서 많은 오탐지 발생**: 환경에 대한 정상적인 활동 베이스라인을 수집한 후, 탐지 룰의 `condition`을 조정하여 점진적으로 정확도를 높이세요.

---

## 결론

Kubernetes 생태계는 컨테이너 이상의 진화를 모색하고 있습니다. **WASM**은 기존 컨테이너의 무게와 아키텍처 종속성을 해결하는 차세대 경량 실행 단위로 부상하고 있으며, 특히 서버리스와 엣지 컴퓨팅 시나리오에서 강력한 장점을 발휘합니다. **eBPF**는 커널 수준에서의 관측, 보안, 네트워킹 기능을 제공함으로써 사이드카 프록시의 복잡성과 오버헤드를 줄이고, 고성능의 정교한 제어를 가능하게 합니다.

이 두 기술을 Kubernetes 플랫폼 위에 **상호 보완적으로 결합**할 때, 우리는 더 빠르고 가벼운 실행 환경, 더 깊고 넓은 가시성, 그리고 더 견고한 보안성을 동시에 얻을 수 있습니다. 성공적인 도입의 핵심은 **점진적 접근**에 있습니다. 작은 서비스나 특정 기능부터 WASM으로 전환을 시작하고, 클러스터 네트워킹 레이어부터 eBPF 기반 도구를 도입하여 관측 데이터를 축적하세요. 충분한 데이터와 경험이 쌓이면, 이를 바탕으로 아키텍처를 단순화하고(예: 불필요한 사이드카 제거), 더 효율적이고 안정적인 플랫폼을 구축해 나갈 수 있을 것입니다.
