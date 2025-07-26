---
layout: post
title: Kubernetes - Kind로 클러스터 구성하기
date: 2025-03-31 22:20:23 +0900
category: Kubernetes
---
# Kind(Kubernetes in Docker)로 클러스터 구성하기

**Kind**는 **Kubernetes IN Docker**의 약자로, **Docker 컨테이너 위에 쿠버네티스 클러스터를 구성할 수 있도록 도와주는 도구**입니다. 주로 CI/CD 테스트나 로컬 개발 환경 구축에 활용됩니다.

> "가상 머신 없이", "가볍게", "빠르게" 쿠버네티스 클러스터를 띄우고 싶다면 **Kind**가 최고의 선택입니다.

---

## ✅ 1. Kind란 무엇인가?

- Docker 컨테이너를 노드처럼 사용하여 **Kubernetes 클러스터를 구성**
- 별도의 하이퍼바이저(VM) 필요 없음
- Kubernetes 공식 프로젝트 중 하나로, 빠른 테스트에 최적화
- `kind` 명령어 하나로 클러스터 생성, 삭제 가능

---

## ✅ 2. 사전 요구사항

- Docker 설치 및 실행 중이어야 함
- Go 언어 환경이 필요 없음 (바이너리 실행)

---

## ✅ 3. Kind 설치

### macOS (Homebrew)

```bash
brew install kind
```

### Linux

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Windows (Chocolatey)

```powershell
choco install kind
```

설치 확인:

```bash
kind version
```

---

## ✅ 4. Kind로 클러스터 생성하기

### 가장 간단한 방법 (기본 설정)

```bash
kind create cluster
```

- 이름: 기본값은 `kind`
- 노드: 1개 (control-plane)
- 쿠버네티스 버전: 최신 안정 버전

### 커스텀 구성 (Config 파일 사용)

```yaml
# cluster-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --name my-cluster --config cluster-config.yaml
```

### 생성된 클러스터 확인

```bash
kubectl cluster-info --context kind-my-cluster
kubectl get nodes
```

---

## ✅ 5. 컨테이너로 구성된 클러스터 확인

```bash
docker ps
```

→ 컨트롤 플레인과 워커 노드가 각각 Docker 컨테이너로 뜨는 것을 볼 수 있습니다.

예:
```
kind-control-plane
kind-worker
kind-worker2
```

---

## ✅ 6. Pod 배포 테스트

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get all
```

### 클러스터 내에서 서비스 접근 테스트

```bash
kubectl run curl --image=radial/busyboxplus:curl -it --restart=Never -- curl http://nginx
```

---

## ✅ 7. Docker 이미지 로딩하기

Kind는 일반 Docker 컨테이너와 격리되어 있으므로, 로컬 이미지 사용 시 별도로 로딩이 필요합니다.

```bash
# Docker 이미지 빌드
docker build -t my-app:latest .

# Kind 클러스터에 이미지 로딩
kind load docker-image my-app:latest
```

---

## ✅ 8. 클러스터 삭제

```bash
kind delete cluster
```

혹은 이름 지정 클러스터 삭제:

```bash
kind delete cluster --name my-cluster
```

---

## ✅ 9. Kind vs Minikube 비교

| 항목 | Kind | Minikube |
|------|------|----------|
| 기반 | Docker 컨테이너 | VM 또는 Docker |
| 설치 속도 | 매우 빠름 | 상대적으로 느림 |
| 리소스 소비 | 낮음 | 높음 |
| 다중 노드 구성 | 지원 | 제한적 |
| 실습 난이도 | 약간 높음 | 쉬움 |
| 실제 네트워크 접근 | 제한 있음 | NodePort/Ingress 가능 |

---

## ✅ 10. 실전 팁

- CI 환경(GitHub Actions, GitLab CI 등)에서 E2E 테스트 용도로 매우 유용
- 로컬 쿠버네티스 환경을 구성할 때, VM 리소스가 부족하거나 Docker만 있는 경우 최적
- `kind` + `kubectl` + `kustomize` + `helm` 조합으로 로컬 테스트 자동화 가능

---

## ✅ 결론

**Kind**는 쿠버네티스를 빠르고 가볍게 실행할 수 있는 훌륭한 도구입니다. 특히 DevOps, 백엔드 개발자, 테스트 환경 구축자에게 매우 적합합니다.