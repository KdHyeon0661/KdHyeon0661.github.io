---
layout: post
title: Kubernetes - Minikube로 로컬 환경 구축
date: 2025-03-31 21:20:23 +0900
category: Kubernetes
---
# Minikube로 로컬 환경 구축하기

쿠버네티스(Kubernetes)는 기본적으로 분산 시스템 환경에서 동작하는 복잡한 구조를 가지고 있지만, **Minikube**를 이용하면 로컬 PC에서도 쉽게 쿠버네티스를 학습하고 실습할 수 있습니다.

이 글에서는 **Minikube를 설치하고, 쿠버네티스 클러스터를 로컬에서 실행**하는 방법을 자세히 설명합니다.

---

## ✅ Minikube란?

Minikube는 로컬 머신에서 쿠버네티스 클러스터를 실행할 수 있도록 도와주는 **경량 개발 도구**입니다. 개발자들이 실습 및 테스트 목적의 K8s 클러스터를 빠르게 구성할 수 있게 해줍니다.

- 단일 노드 클러스터 생성
- 다양한 하이퍼바이저 및 드라이버 지원
- `kubectl`을 통한 일반 쿠버네티스 클러스터와 동일한 명령어 사용 가능

---

## ✅ 1. 사전 준비

### 요구 사항

- Windows / macOS / Linux
- CPU 가상화 지원
- Hypervisor (아래 중 하나)
  - Windows: Hyper-V, VirtualBox, WSL2
  - macOS: HyperKit, VirtualBox
  - Linux: KVM, VirtualBox, Docker 등

---

## ✅ 2. Minikube 설치

### macOS (Homebrew 사용)

```bash
brew install minikube
```

### Ubuntu / Debian

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```

### Windows (Chocolatey 사용)

```powershell
choco install minikube
```

---

## ✅ 3. kubectl 설치

Minikube는 kubectl도 함께 설치할 수 있지만, 수동으로 설치하려면:

### macOS

```bash
brew install kubectl
```

### Linux

```bash
sudo apt install -y kubectl
```

설치 확인:

```bash
kubectl version --client
```

---

## ✅ 4. Minikube 클러스터 시작

### 기본 실행 (가상 머신 방식)

```bash
minikube start
```

### Docker 드라이버 사용 (가상화 없이 실행)

```bash
minikube start --driver=docker
```

→ 이 명령은 Minikube가 백그라운드에서 단일 노드 쿠버네티스 클러스터를 자동 구성합니다.

### 클러스터 상태 확인

```bash
minikube status
```

---

## ✅ 5. 기본 사용법

### 쿠버네티스 대시보드 실행

```bash
minikube dashboard
```

→ 브라우저가 자동으로 열리고 GUI 환경에서 클러스터 상태를 확인할 수 있습니다.

---

### Pod 배포 예제 (nginx)

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=NodePort --port=80
```

### 서비스 URL 확인

```bash
minikube service nginx
```

→ 브라우저로 nginx 페이지에 접속 가능

---

## ✅ 6. 클러스터 중지 및 삭제

```bash
# 일시 중지
minikube stop

# 클러스터 완전 삭제
minikube delete
```

---

## ✅ 7. 유용한 명령어 모음

```bash
minikube start                 # 클러스터 시작
minikube status                # 클러스터 상태 확인
minikube dashboard             # 웹 UI 실행
minikube service [서비스명]   # 서비스 접속 URL 오픈
minikube ssh                  # 가상 머신 내부로 접속
minikube addons list          # 추가 기능 확인
```

---

## ✅ 8. 자주 사용하는 애드온 (addons)

Minikube는 여러 가지 애드온을 제공합니다. 예:

```bash
minikube addons enable ingress
minikube addons enable metrics-server
```

- `ingress`: Ingress Controller 설치
- `metrics-server`: HPA(Horizontal Pod Autoscaler)를 위한 메트릭 수집

---

## ✅ 9. 자주 발생하는 문제 해결

| 문제 | 해결 방법 |
|------|------------|
| Docker 드라이버 사용 시 클러스터가 안 뜸 | `minikube delete && minikube start --driver=docker` 재시작 |
| `kubectl`이 Minikube와 연결 안 됨 | `kubectl config use-context minikube` |
| 가상화 오류 발생 | BIOS에서 VT-x/AMD-V 가상화 옵션 활성화 필요 |

---

## ✅ 결론

Minikube는 **쿠버네티스를 배우고 실습하기에 가장 간편한 방법**입니다. 실제 운영 환경은 복잡할 수 있지만, Minikube를 통해 핵심 개념과 명령어를 익힌다면 이후 GKE, EKS 같은 클라우드 환경으로 자연스럽게 확장할 수 있습니다.

→ 다음 단계로는 `Kind`, `Helm`, 또는 `Ingress`, `ConfigMap`, `PersistentVolume` 등을 실습해보는 것이 좋습니다.