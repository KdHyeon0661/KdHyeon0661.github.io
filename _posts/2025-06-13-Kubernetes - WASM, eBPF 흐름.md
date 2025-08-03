---
layout: post
title: Kubernetes - WASM, eBPF 흐름
date: 2025-06-13 20:20:23 +0900
category: Kubernetes
---
# 쿠버네티스와 WASM, eBPF 흐름 이해하기

최근 Kubernetes 기반 인프라에서 주목받는 기술 두 가지가 있습니다:

- **WASM (WebAssembly)**: 경량 고성능 실행 환경
- **eBPF (Extended Berkeley Packet Filter)**: 커널 레벨의 관측, 보안, 네트워크 제어

이 두 기술은 쿠버네티스와 결합되며, **컨테이너 한계를 넘어서고**, **운영과 보안의 새로운 패러다임**을 열고 있습니다. 이번 글에서는 이 흐름을 **실전 중심의 인프라 관점**에서 설명합니다.

---

## 🧱 기존 Kubernetes의 한계

Kubernetes는 컨테이너 기반 오케스트레이션에 뛰어나지만, 다음과 같은 한계도 존재합니다:

| 영역 | 기존 방식의 한계 |
|------|------------------|
| 성능 | 컨테이너 자체 오버헤드, 느린 cold start |
| 네트워크 | IP 기반 정책 한계, 복잡한 L3/L4 구성 |
| 보안 | 커널 수준에서의 동작 추적 어려움 |
| 이식성 | x86/ARM 등 아키텍처 종속적인 이미지 |

이러한 한계를 극복하기 위한 기술로 **WASM과 eBPF**가 등장했습니다.

---

## 🧪 WASM (WebAssembly) 개요

### 📌 정의

> **WASM(WebAssembly)**는 브라우저를 넘어, **경량 실행 환경으로 확장**된 바이트코드 기반의 실행 포맷입니다.  
> 다양한 언어(C, Rust, Go 등)로 작성된 코드를 안전하게 실행할 수 있으며, **컨테이너보다 훨씬 가볍고 빠릅니다.**

### ✅ 특징

- **초경량 런타임** (수 MB, 실행 시간 < ms)
- 높은 **이식성** (CPU 아키텍처 무관)
- 강력한 **보안 격리** (sandboxed 실행)
- **빠른 cold start** (서버리스에 적합)

### 🚀 Kubernetes + WASM 활용

- **서버리스 함수 런타임**: scale-to-zero & 빠른 응답 (→ [Wasmtime](https://wasmtime.dev/), [WasmEdge](https://wasmedge.org/))
- **마이크로 VM 대체**: 경량 실행 환경으로 사용
- **Sidecar 없이 plugin 실행**: network/storage filter 등을 런타임에 삽입

### 🛠 대표 프로젝트

| 프로젝트 | 설명 |
|----------|------|
| **Krustlet** | WASM 전용 K8s kubelet |
| **WasmEdge** | CNCF Sandbox 등록 WASM 런타임 |
| **Spin (Fermyon)** | WASM 기반의 서버리스 프레임워크 |
| **containerd + Wasm shim** | 컨테이너 런타임 내 WASM 지원 |

---

## 🔍 eBPF (Extended Berkeley Packet Filter) 개요

### 📌 정의

> **eBPF**는 Linux 커널 내부에 **유저 공간에서 작성한 프로그램을 주입하여 실행**할 수 있는 기술입니다.  
> 관측, 트레이싱, 네트워크 제어, 보안 정책 등 다양하게 활용됩니다.

### ✅ 특징

- **런타임 관찰 및 제어 가능** (트레이싱/보안)
- 커널 수정 없이 **안정성 보장**
- **낮은 오버헤드**, 고성능 실시간 실행
- **필터링/감시/변경 가능** (네트워크 패킷, 시스템콜 등)

### 🔧 Kubernetes + eBPF 활용

| 분야 | 활용 예 |
|------|----------|
| **네트워크 정책** | L3/L4 + L7 제어 (Cilium) |
| **보안** | Pod 단위 시스템콜 감시 (Falco, Tracee) |
| **관측** | eBPF 기반 메트릭/트레이싱 (Pixie, Hubble) |
| **로드 밸런싱** | kube-proxy 제거, BPF 기반 직접 라우팅

### 🛠 대표 프로젝트

| 프로젝트 | 설명 |
|----------|------|
| **Cilium** | eBPF 기반 CNI 플러그인 (Service Mesh + NetworkPolicy) |
| **Hubble** | 실시간 eBPF 트래픽 관측 도구 |
| **Falco** | eBPF 기반 런타임 보안 감지 |
| **BPFtrace** | eBPF 기반 DSL 트레이싱 언어 |
| **Pixie** | 전체 Pod 수준 eBPF 기반 관측 솔루션

---

## 🧬 WASM + eBPF + Kubernetes 조합의 흐름

이제 두 기술은 **서로 다른 역할**을 수행하며, Kubernetes 생태계에서 보완적인 관계를 형성합니다.

| 항목 | WASM | eBPF |
|------|------|------|
| 목적 | 앱 실행 환경 대체 (컨테이너 대안) | 커널 수준 제어 및 관측 |
| 위치 | 유저 공간에서 실행 | 커널 공간에서 실행 |
| 대표 사례 | 서버리스, MicroVM 대체 | 네트워크, 보안, 성능 분석 |
| K8s 연계 | Krustlet, Spin, WasmEdge | Cilium, Falco, Pixie |

결국, **WASM은 ‘실행’의 새로운 단위**, **eBPF는 ‘관측 및 제어’의 확장된 시야**를 Kubernetes에 제공합니다.

---

## 💡 실무에서 기대할 수 있는 변화

| 분야 | 기대 효과 |
|------|------------|
| **서버리스/경량 런타임** | WASM을 통한 cold start 극복, 빠른 배포 |
| **서비스 메시 단순화** | Cilium + eBPF → Envoy 없이 메시 구현 가능 |
| **보안 & 추적** | eBPF로 시스템콜 추적, 행위 기반 공격 탐지 |
| **디버깅 및 관측** | eBPF 기반 실시간 metrics/tracing 도입 |
| **다중 플랫폼 지원** | WASM으로 아키텍처 이식성 확보 (x86/ARM 등) |

---

## 📚 참고 리소스

- [WasmEdge (CNCF)](https://github.com/WasmEdge/WasmEdge)
- [Cilium 공식 사이트](https://cilium.io/)
- [Pixie - eBPF Observability](https://px.dev/)
- [Krustlet 프로젝트](https://github.com/krustlet/krustlet)
- [OpenFaaS + WASM 예제](https://docs.openfaas.com/cli/templates/wasm/)

---

## ✅ 마무리 요약

| 키워드 | 설명 |
|--------|------|
| WASM | 초경량 런타임 → 서버리스, 마이크로VM 대체 가능 |
| eBPF | 커널 수준 보안/관측 도구 → 네트워크/보안 강화 |
| Kubernetes 통합 | 더 가볍고, 안전하며, 실시간 제어 가능한 인프라 구현 가능 |
| 미래 지향점 | "컨테이너 그 이후의 실행 환경"과 "가시성 높은 운영체계" 구축 |