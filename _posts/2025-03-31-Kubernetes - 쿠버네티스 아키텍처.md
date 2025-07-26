---
layout: post
title: Kubernetes - 쿠버네티스 아키텍처
date: 2025-03-31 20:20:23 +0900
category: Kubernetes
---
# 쿠버네티스 아키텍처: Master vs Node 구성

쿠버네티스(Kubernetes)는 **분산 시스템**으로 동작하며, 클러스터(Cluster) 형태로 구성됩니다. 이 클러스터는 크게 두 가지 주요 컴포넌트로 구성됩니다:

- **Control Plane (마스터 노드)**
- **Worker Node (작업자 노드)**

각 컴포넌트는 특정 역할을 담당하며, 함께 작동하여 컨테이너화된 애플리케이션을 배포하고 관리합니다.

---

## 1. 마스터 노드 (Control Plane)

**클러스터 전체의 상태를 제어하고 관리**하는 핵심 부분입니다. 스케줄링, 상태 감시, 정책 결정 등을 수행합니다.

### 주요 구성 요소:

### ✅ 1.1 `kube-apiserver`
- 쿠버네티스의 **중앙 API 게이트웨이**
- 모든 컴포넌트는 이 API를 통해 상호작용
- 외부와 내부 요청의 진입점 역할
- RESTful API를 통해 클러스터 상태를 읽고 쓸 수 있음

```bash
kubectl get pods
```
→ 위 명령은 `kubectl`이 `kube-apiserver`에 REST 요청을 보내는 것과 동일합니다.

---

### ✅ 1.2 `etcd`
- **클러스터의 상태를 저장하는 Key-Value 저장소**
- 모든 구성 정보, 상태 정보가 여기에 저장됨
- 분산 시스템을 위한 고가용성 구성 가능

```bash
# 예: Pod 정의나 ConfigMap 등 YAML의 내부 내용이 etcd에 저장됨
```

---

### ✅ 1.3 `kube-scheduler`
- 새로 생성된 Pod에 대해 **어떤 Node에 배치할지 결정**
- 스케줄링 조건 예:
  - 리소스 가용성 (CPU, Memory)
  - Node Selector, Affinity/Anti-Affinity
  - Taints/Tolerations

---

### ✅ 1.4 `kube-controller-manager`
- 다양한 **컨트롤러(Controller)** 를 관리하여 클러스터 상태를 원하는 상태로 유지
- 대표적인 컨트롤러:
  - **DeploymentController**: 레플리카 수 관리
  - **NodeController**: 노드의 상태 감시
  - **JobController**, **ReplicaSetController** 등

> 실제로는 하나의 프로세스에서 여러 컨트롤러가 동작

---

## 2. 워커 노드 (Worker Node)

**실제 애플리케이션 컨테이너(Pod)가 배포되고 실행되는 머신**입니다. 각 워커 노드는 다음과 같은 컴포넌트를 포함합니다.

### 주요 구성 요소:

### ✅ 2.1 `kubelet`
- 마스터의 명령을 받아 **컨테이너 상태를 관리하는 에이전트**
- Pod의 상태를 주기적으로 체크하고, 정상 동작하지 않으면 복구 시도
- 매니페스트 파일을 기반으로 컨테이너 실행

---

### ✅ 2.2 `kube-proxy`
- **클러스터 내부 네트워크를 구성**하고 서비스 간 통신을 가능하게 함
- 각 노드에서 Pod 간의 통신을 위한 NAT, IP 테이블 등을 관리

---

### ✅ 2.3 컨테이너 런타임
- 실제로 컨테이너를 실행하는 프로그램
- 기본적으로는 **containerd**, **CRI-O**, 또는 Docker (비추천)
- 쿠버네티스는 CRI(Container Runtime Interface)를 통해 런타임과 연결

---

## 3. 전체 아키텍처 구성도 (텍스트 기반)

```
          [ Control Plane (Master) ]
         ┌────────────────────────────┐
         │  kube-apiserver            │
         │  etcd                      │
         │  kube-scheduler            │
         │  kube-controller-manager   │
         └────────────────────────────┘
                     ▲
     ┌───────────────┴───────────────┐
     ▼                               ▼
[ Worker Node 1 ]             [ Worker Node 2 ]
┌──────────────────┐        ┌──────────────────┐
│  kubelet         │        │  kubelet         │
│  kube-proxy      │        │  kube-proxy      │
│  containerd      │        │  containerd      │
│  Pod1, Pod2...   │        │  Pod3, Pod4...   │
└──────────────────┘        └──────────────────┘
```

---

## 4. 요약

| 역할 | Master Node | Worker Node |
|------|-------------|-------------|
| 책임 | 클러스터 상태 관리, 스케줄링 | 실제 컨테이너 실행 |
| 주요 컴포넌트 | kube-apiserver, etcd, scheduler, controller-manager | kubelet, kube-proxy, container runtime |
| 애플리케이션 실행 여부 | ❌ | ✅ Pod 실행 |

---

## 결론

쿠버네티스 아키텍처는 **중앙 집중적인 관리(Control Plane)** 와 **분산된 실행 환경(Worker Node)** 으로 구성되어, 대규모 컨테이너 기반 애플리케이션을 안정적으로 운영할 수 있는 기반을 제공합니다. 이 구조 덕분에 확장성, 장애 복구, 자동화된 운영이 가능해집니다.