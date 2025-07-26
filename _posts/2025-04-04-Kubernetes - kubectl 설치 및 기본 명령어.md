---
layout: post
title: Kubernetes - kubectl 설치 및 기본 명령어
date: 2025-04-04 19:20:23 +0900
category: Kubernetes
---
# kubectl 설치 및 기본 명령어 사용법

쿠버네티스(Kubernetes)를 조작하는 데 가장 중요한 CLI 도구는 바로 `kubectl`입니다. 이 글에서는 `kubectl`을 설치하는 방법과 자주 사용하는 기본 명령어를 예제와 함께 소개합니다.

---

## ✅ kubectl이란?

`kubectl`은 쿠버네티스 클러스터와 상호작용할 수 있는 **명령줄 인터페이스 도구**입니다. 다음과 같은 작업을 수행할 수 있습니다:

- 리소스 생성/삭제
- 상태 조회 및 수정
- YAML 파일로 선언적 관리
- 디버깅 및 로그 확인 등

---

## ✅ kubectl 설치

### 🔧 macOS (Homebrew)

```bash
brew install kubectl
```

### 🔧 Ubuntu / Debian

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubectl
```

### 🔧 Windows (Chocolatey)

```powershell
choco install kubernetes-cli
```

### 설치 확인

```bash
kubectl version --client
```

---

## ✅ kubectl 기본 구조

```bash
kubectl [명령어] [리소스 타입] [이름] [옵션]
```

예시:

```bash
kubectl get pods
kubectl describe service my-service
kubectl delete deployment nginx-deployment
```

---

## ✅ 자주 사용하는 리소스 타입

| 타입 | 설명 |
|------|------|
| pod | 실행 중인 컨테이너 단위 |
| deployment | 애플리케이션 배포 단위 |
| service | 네트워크 서비스 정의 |
| configmap | 설정 데이터 저장 |
| secret | 민감한 데이터 저장 |
| node | 클러스터의 노드 정보 |
| namespace | 리소스의 논리적 분리 공간 |

---

## ✅ kubectl 기본 명령어 모음

### 1. 클러스터 정보 확인

```bash
kubectl cluster-info
kubectl get nodes
```

### 2. 네임스페이스 조회

```bash
kubectl get namespaces
kubectl config set-context --current --namespace=default
```

---

### 3. 리소스 조회 (get)

```bash
kubectl get pods
kubectl get services
kubectl get deployments
kubectl get all
```

YAML 형식으로 출력:

```bash
kubectl get pods -o yaml
```

---

### 4. 리소스 상세 보기 (describe)

```bash
kubectl describe pod [POD_NAME]
kubectl describe service [SERVICE_NAME]
```

---

### 5. 리소스 생성

#### 5.1 Imperative 방식 (즉석 생성)

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=NodePort --port=80
```

#### 5.2 Declarative 방식 (YAML 파일 기반)

```bash
kubectl apply -f nginx-deployment.yaml
```

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

---

### 6. 리소스 수정

```bash
kubectl edit deployment nginx
```

- 텍스트 에디터가 열리며 실시간으로 수정 가능

---

### 7. 리소스 삭제

```bash
kubectl delete pod [POD_NAME]
kubectl delete deployment [DEPLOYMENT_NAME]
kubectl delete -f nginx-deployment.yaml
```

---

### 8. 로그 확인

```bash
kubectl logs [POD_NAME]
kubectl logs -f [POD_NAME]    # 실시간 스트리밍
```

---

### 9. Pod에 접속 (exec)

```bash
kubectl exec -it [POD_NAME] -- /bin/bash
```

→ 컨테이너 내부에 직접 들어가 디버깅 가능

---

### 10. 리소스 자동 완성 (bash/zsh)

```bash
# Bash 기준
source <(kubectl completion bash)

# 영구 적용 (예: .bashrc)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

---

## ✅ 실습: nginx 배포 & 서비스 노출

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=NodePort --port=80
kubectl get all
minikube service nginx
```

---

## ✅ Troubleshooting 팁

| 문제 | 해결 방법 |
|------|------------|
| 클러스터 연결 오류 | `kubectl config view`, `kubectl config use-context` |
| Pod가 Pending 상태 | `kubectl describe pod [이름]`으로 이벤트 확인 |
| 리소스 수정 시 충돌 | `kubectl replace`, `kubectl apply --force` 고려 |

---

## ✅ 결론

`kubectl`은 쿠버네티스를 사용하는 데 있어 가장 중요한 CLI 도구입니다. 기본 명령어만 잘 익혀도 클러스터의 동작을 이해하고, 디버깅 및 운영까지 수행할 수 있습니다.