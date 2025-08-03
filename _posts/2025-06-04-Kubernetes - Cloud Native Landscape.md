---
layout: post
title: Kubernetes - Cloud Native Landscape
date: 2025-06-04 22:20:23 +0900
category: Kubernetes
---
# Cloud Native Landscape 이해하기

Cloud Native 기술은 오늘날 **마이크로서비스 아키텍처**, **컨테이너 기반 개발**, **자동화된 배포 및 운영**, **확장성 있는 시스템 설계**를 실현하기 위해 필수적인 기반입니다.

이 생태계는 **CNCF(Cloud Native Computing Foundation)**에서 정리한 **Cloud Native Landscape**를 통해 정형화되어 있으며, 수많은 오픈소스 기술과 툴이 포함되어 있습니다.

이 문서에서는 Cloud Native Landscape를 구성하는 주요 영역과 대표 프로젝트들을 이해하기 쉽게 설명합니다.

---

## ✅ Cloud Native란?

**Cloud Native**란,  
> 클라우드 환경에서 유연하게 **배포**, **확장**, **관측**, **복구**할 수 있도록 설계된 애플리케이션 및 시스템 아키텍처 접근 방식입니다.

핵심 개념:

- **컨테이너화(Containerization)**: 환경과 무관한 실행 단위
- **마이크로서비스**: 작은 독립적 서비스 단위
- **DevOps & GitOps**: 자동화된 배포 및 운영
- **불변 인프라**: 시스템 상태를 변경하지 않고 교체

---

## ✅ CNCF (Cloud Native Computing Foundation)

- **비영리 오픈소스 재단**
- Kubernetes, Prometheus, Envoy 등 수많은 핵심 프로젝트 주도
- Cloud Native 기술을 **표준화**하고 **확산**하는 역할

[CNCF Landscape 사이트 바로가기](https://landscape.cncf.io/)

---

## 🗺️ Cloud Native Landscape 구조

Cloud Native Landscape는 수백 개의 도구를 다음과 같은 **분야(Category)**로 나눕니다:

### 📌 1. **Provisioning & Infrastructure**
> 클러스터 설치, 인프라 자동화

| 영역 | 도구 |
|------|------|
| IaC (Infrastructure as Code) | Terraform, Pulumi |
| 프로비저닝 | Kubespray, RKE, Kubeadm |
| 서버리스 | Knative, OpenFaaS |

---

### 📌 2. **Orchestration & Management**
> 컨테이너 및 리소스 스케줄링/운영

| 영역 | 도구 |
|------|------|
| 컨테이너 오케스트레이션 | Kubernetes (CNCF Graduated), Nomad |
| 스케줄링/배치 | Volcano, Kube-batch |
| 서비스 메쉬 | Istio, Linkerd, Kuma |

---

### 📌 3. **Runtime**
> 컨테이너 런타임 및 CRI

| 영역 | 도구 |
|------|------|
| 컨테이너 런타임 | containerd, CRI-O, Docker |
| WebAssembly | Wasmtime, WasmEdge |

---

### 📌 4. **Observability & Analysis**
> 모니터링, 로깅, 트레이싱

| 영역 | 도구 |
|------|------|
| 모니터링 | Prometheus, Thanos, Cortex |
| 로그 수집 | Fluentd, Loki |
| 트레이싱 | Jaeger, OpenTelemetry |
| 대시보드 | Grafana, Kibana |

---

### 📌 5. **Security & Compliance**
> 보안, 정책, 취약점 분석

| 영역 | 도구 |
|------|------|
| 이미지 스캐닝 | Trivy, Clair |
| 런타임 보안 | Falco, Cilium, AppArmor |
| 정책 관리 | OPA (Open Policy Agent), Kyverno |
| 인증/권한 | SPIRE, cert-manager |

---

### 📌 6. **App Definition & DevOps**
> 애플리케이션 배포 및 개발 흐름

| 영역 | 도구 |
|------|------|
| 패키징 | Helm, Kustomize |
| CI/CD | ArgoCD, Tekton, Flux |
| 소스 컨트롤 | GitHub, GitLab |
| 테스트 | Litmus, Chaos Mesh |

---

### 📌 7. **Networking**
> 클러스터 네트워킹, 서비스 디스커버리

| 영역 | 도구 |
|------|------|
| CNI | Calico, Cilium, Flannel |
| API 게이트웨이 | Envoy, Kong, Ambassador |
| 서비스 디스커버리 | CoreDNS, Consul |

---

### 📌 8. **Storage**
> 볼륨, 블록/파일/오브젝트 스토리지

| 영역 | 도구 |
|------|------|
| 분산 스토리지 | Rook/Ceph, Longhorn |
| CSI 드라이버 | OpenEBS, TopoLVM |
| 백업/복구 | Velero, Stash |

---

## 🔄 Cloud Native 프로젝트 등급

CNCF는 프로젝트를 다음과 같이 등급화합니다:

| 등급 | 의미 | 예시 |
|------|------|------|
| **Graduated** | 안정적, 대규모 사용 가능 | Kubernetes, Prometheus, Envoy |
| **Incubating** | 성장 중, 많은 채택 | Argo, Vitess, OpenTelemetry |
| **Sandbox** | 초기 실험 단계 | Kubevela, Chaos Mesh |

→ **Graduated**는 실무 투입에 적합한 고성숙도 프로젝트입니다.

---

## 🧭 실무에서의 선택 전략

| 필요 영역 | 추천 도구 |
|-----------|-----------|
| 배포 자동화 | ArgoCD + Helm |
| 모니터링 | Prometheus + Grafana |
| 서비스 메시 | Istio or Linkerd |
| 런타임 보안 | Falco + Trivy |
| 네트워크 정책 | Calico or Cilium |
| 로그 수집 | Loki + Fluent Bit |
| CI 파이프라인 | Tekton, GitHub Actions |

---

## ✅ 정리

| 항목 | 요약 |
|------|------|
| 목적 | 클라우드 환경에 최적화된 시스템 설계 |
| 구조 | 수백 개의 오픈소스가 카테고리별로 정리 |
| 대표 기술 | Kubernetes, Prometheus, Envoy, ArgoCD 등 |
| 실전 접근 | 각 분야별 목적에 맞는 툴 선택 → 통합 운영 체계 구축

---

## 📚 참고 링크

- [CNCF 공식 사이트](https://www.cncf.io/)
- [CNCF Cloud Native Landscape](https://landscape.cncf.io/)
- [CNCF Project Maturity](https://www.cncf.io/projects/)