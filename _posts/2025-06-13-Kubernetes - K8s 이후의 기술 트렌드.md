---
layout: post
title: Kubernetes - K8s 이후의 기술 트렌드
date: 2025-06-13 21:20:23 +0900
category: Kubernetes
---
# K8s 이후의 기술 트렌드: K3s, MicroK8s, Edge K8s 등

Kubernetes(K8s)는 현대 클라우드 인프라의 핵심 플랫폼이지만, **모든 환경에서 K8s가 최적은 아닙니다.**

특히 **IoT, 엣지 컴퓨팅, 개발용 환경, 경량 서버** 등에서는 K8s의 무거운 구조가 오히려 **복잡성과 리소스 낭비**를 초래할 수 있습니다.

이에 따라 등장한 기술이 **경량화된 K8s 배포판**들입니다.  
대표적으로 **K3s, MicroK8s**, 그리고 다양한 **Edge K8s 플랫폼**이 있습니다.

---

## 📌 왜 경량 Kubernetes가 필요한가?

| 문제 | 설명 |
|------|------|
| 리소스 과다 | 기본 K8s는 컨트롤 플레인과 여러 컴포넌트로 수백 MB ~ GB 사용 |
| 복잡한 설치 | kubeadm 등으로 설치 시 많은 설정, 구성 필요 |
| 엣지 환경 부적합 | 저사양 기기, 불안정 네트워크에서는 과도한 운영 부담 |
| Dev 환경 무거움 | 개발용 로컬 테스트에 과한 구조

---

## 🚀 K3s: Rancher Labs가 만든 경량 K8s

### ✅ 특징

- **완전한 CNCF 호환 K8s API**
- **50MB 미만 바이너리**, ARM 지원
- SQLite 기본 내장 (etcd 대체 가능)
- systemd 없이도 동작 가능
- `kubectl`과 동일하게 사용 가능

### 🏗 구성 요소

- `k3s` 단일 바이너리 → `kube-apiserver`, `controller`, `scheduler` 포함
- **컨테이너 런타임: containerd 내장**
- **클러스터 통신: Flannel or Traefik 기본 포함**
- 인증서 자동 갱신 및 로테이션 지원

### 🛠 설치 방법 (단일 노드)

```bash
curl -sfL https://get.k3s.io | sh -
kubectl get nodes
```

### 🌍 활용 사례

- IoT 게이트웨이 및 엣지 디바이스
- 로컬 테스트 및 학습용 클러스터
- K8s 기반 경량 클러스터로 Rancher에서 통합 관리

---

## 🧪 MicroK8s: Canonical이 만든 개발 중심 경량 K8s

### ✅ 특징

- **Snap 패키지로 설치**, 우분투 최적화
- **애드온 방식 구성 (DNS, Ingress, Prometheus 등)**  
- GPU 지원, AI/ML 개발 친화적
- **멀티 노드 클러스터 구성 가능**
- 자체 컨테이너 런타임 (`containerd`)

### 🛠 설치 및 실행

```bash
sudo snap install microk8s --classic
microk8s status --wait-ready
microk8s enable dns ingress dashboard
microk8s kubectl get nodes
```

### 🌍 활용 사례

- **로컬 개발 및 테스트 환경** (Kubeflow, ML 플랫폼)
- CI 파이프라인 내 K8s 테스트 환경
- GPU가 포함된 AI/ML 엣지 환경

---

## 🛰️ Edge K8s: 엣지 전용 K8s 플랫폼들

엣지 컴퓨팅 환경에서는 일반적인 클라우드와 다른 **요구사항**이 존재합니다:

- 네트워크 불안정 (간헐적 연결)
- 낮은 하드웨어 스펙
- 보안 정책 강화 필요
- 멀티노드 구성 불가한 경우도 있음

이를 해결하기 위한 다양한 플랫폼들이 등장하고 있습니다.

### ✅ 대표 Edge 플랫폼

| 플랫폼 | 설명 |
|--------|------|
| **KubeEdge** | K8s와 엣지 장치를 연결하는 플랫폼 (Huawei 주도) |
| **SuperEdge** | 대규모 엣지 노드 지원, 분산된 제어 평면 구성 |
| **OpenYurt** | 클라우드-엣지 하이브리드 애플리케이션 구현 |
| **Rafay Edge** | SaaS 기반 엣지 K8s 관리 플랫폼 |

### 📦 KubeEdge 구성 예

- Cloud: K8s Master + CloudHub
- Edge: EdgeCore + MQTT → K8s API 연결
- 디바이스 직접 제어 (GPIO, Serial 등)

---

## 📊 K3s vs MicroK8s vs KubeEdge 비교

| 항목 | K3s | MicroK8s | KubeEdge |
|------|-----|----------|----------|
| 목적 | 경량 K8s 클러스터 | 로컬 개발/AI용 K8s | 엣지 장치 연동 |
| 설치 | 스크립트로 단순 | snap 기반 간편 설치 | 복잡함 (다중 컴포넌트) |
| 리소스 | 매우 적음 (~100MB RAM) | 중간 (~300MB RAM) | 장치 스펙 따라 상이 |
| 관리 | Rancher와 통합 용이 | 명령형 UI 구성 | 엣지-클라우드 연동 |
| 기능 확장 | 기본 내장 (traefik 등) | `enable` 명령으로 활성화 | MQTT, EdgeBus 지원 |
| CPU 아키텍처 | x86/ARM 모두 지원 | x86/ARM | x86/ARM |

---

## 🧭 언제 어떤 것을 선택할까?

| 상황 | 추천 플랫폼 |
|------|-------------|
| 저사양 라즈베리파이에서 K8s 운영 | **K3s** |
| 로컬에서 간편하게 개발/실험 | **MicroK8s** |
| 엣지 디바이스 + 클라우드 연동 | **KubeEdge**, **OpenYurt** |
| GPU 기반 AI 모델 테스트 | **MicroK8s** with GPU |
| 단일 파일 배포, 빠른 부팅 | **K3s** |

---

## 🧩 확장 트렌드: 경량화 → 분산 엣지 제어

미래의 K8s 트렌드는 다음과 같은 방향으로 전개되고 있습니다:

1. **경량화된 쿠버네티스 (Single binary, No etcd)**
2. **멀티 클러스터 엣지 분산 처리**
3. **보안 강화 (SPIFFE, mTLS, Zero Trust)**
4. **eBPF + WASM으로 커널/애플리케이션 제어 통합**
5. **AI/ML 모델 배포를 위한 엣지 오케스트레이션**

---

## 📚 참고 링크

- [K3s 공식 사이트](https://k3s.io/)
- [MicroK8s 공식 문서](https://microk8s.io/)
- [KubeEdge 공식 프로젝트](https://kubeedge.io/)
- [OpenYurt (Alibaba)](https://openyurt.io/)
- [Edge Native Working Group](https://edgenative.cncf.io/)

---

## ✅ 마무리 요약

| 키포인트 | 설명 |
|----------|------|
| K8s는 점점 더 가볍고 유연해지고 있음 |
| K3s, MicroK8s는 **로컬/경량 환경**에 최적 |
| KubeEdge, OpenYurt는 **엣지와 클라우드 연계**에 강점 |
| 선택 기준은 **리소스 제약, 배포 시나리오, 연동 대상**에 따라 다름 |