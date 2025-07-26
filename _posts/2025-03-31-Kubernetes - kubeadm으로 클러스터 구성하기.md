---
layout: post
title: Kubernetes - kubeadm으로 클러스터 구성하기
date: 2025-03-31 23:20:23 +0900
category: Kubernetes
---
# kubeadm으로 실제 서버에 쿠버네티스 클러스터 구성하기

`kubeadm`은 **쿠버네티스 클러스터를 부트스트랩하는 공식 도구**입니다. Minikube나 Kind는 로컬 개발용이지만, `kubeadm`은 **실제 서버에 쿠버네티스를 직접 구성할 때 사용하는 도구**로, 실무 환경에서도 자주 사용됩니다.

이 글에서는 Ubuntu 서버를 기준으로 `kubeadm`을 사용하여 **1개의 마스터 + N개의 워커 노드로 구성된 클러스터**를 구축하는 방법을 단계별로 설명합니다.

---

## ✅ 1. 클러스터 아키텍처 예시

- master-node (1대): kube-apiserver, etcd, controller-manager, scheduler
- worker-node (1대 이상): 실제 Pod 실행

```
[ master-node ]
     ↑
     ↓ (kubectl, kubeadm, kubelet, etcd)
     ↓
[ worker-node-1 ] 
[ worker-node-2 ] 
```

---

## ✅ 2. 사전 준비 (모든 노드 공통)

- OS: Ubuntu 20.04 이상
- root 또는 sudo 권한 사용자
- 각 노드의 hostname과 IP 정리 필요

### 네트워크 예시

| 노드 | Hostname         | IP            |
|------|------------------|---------------|
| Master | `master-node`    | `192.168.56.101` |
| Worker1 | `worker-node-1`  | `192.168.56.102` |
| Worker2 | `worker-node-2`  | `192.168.56.103` |

---

## ✅ 3. 사전 환경 구성 (공통 설정)

### 3.1 호스트네임 설정

```bash
sudo hostnamectl set-hostname master-node  # 각 노드에 맞게
```

### 3.2 Swap 비활성화

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 3.3 방화벽 열기

```bash
sudo ufw disable  # 테스트 목적
```

### 3.4 커널 파라미터 조정

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

---

## ✅ 4. Docker 또는 Containerd 설치

```bash
sudo apt update
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## ✅ 5. kubeadm, kubelet, kubectl 설치

```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo bash -c 'cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF'

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## ✅ 6. Master 노드 클러스터 초기화

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

→ `--pod-network-cidr`는 사용하는 네트워크 플러그인에 따라 다름 (예: Calico)

### 성공 시 출력 예시:

```
You can now join any number of machines by running the following on each node:
kubeadm join 192.168.56.101:6443 --token abcdef.0123456789abcdef \
--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

이 명령어를 복사해 워커 노드에서 사용합니다.

---

## ✅ 7. kubectl 설정 (Master 노드)

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

클러스터 확인:

```bash
kubectl get nodes
```

---

## ✅ 8. 네트워크 플러그인 설치 (Calico 예)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

Pod 네트워크가 설치되지 않으면 `NotReady` 상태가 계속 유지됩니다.

---

## ✅ 9. 워커 노드 클러스터 조인

각 워커 노드에서 다음 명령 실행:

```bash
sudo kubeadm join 192.168.56.101:6443 --token abcdef.0123456789abcdef \
--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

조인 성공 후 Master 노드에서 확인:

```bash
kubectl get nodes
```

→ 모든 노드가 `Ready` 상태로 나와야 정상

---

## ✅ 10. 간단한 테스트: Nginx 배포

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get all
```

---

## ✅ 11. 클러스터 재시작 시 주의사항

- `kubeadm init`은 한 번만 실행해야 함
- 노드 재부팅 후에도 `kubelet`, `containerd`가 자동 시작되도록 설정되어 있어야 함
- `kubectl`은 `$HOME/.kube/config`가 유지되어야 정상 동작

---

## ✅ 결론

`kubeadm`은 **실제 서버 환경에서 쿠버네티스를 직접 구성할 수 있는 강력한 도구**입니다. 실무에서는 클라우드 매니지드 쿠버네티스(GKE, EKS 등)를 많이 쓰지만, `kubeadm`을 통해 쿠버네티스의 구조와 과정을 직접 경험하면 **운영 및 장애 대응 능력**이 크게 향상됩니다.