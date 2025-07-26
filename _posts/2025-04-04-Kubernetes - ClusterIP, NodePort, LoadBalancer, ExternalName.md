---
layout: post
title: Kubernetes - ClusterIP, NodePort, LoadBalancer, ExternalName
date: 2025-04-04 22:20:23 +0900
category: Kubernetes
---
# K8s 핵심 오브젝트 이해하기 : Service (ClusterIP, NodePort, LoadBalancer, ExternalName)

Kubernetes에서 `Pod`는 IP가 동적으로 바뀌고, 클러스터 내부적으로만 접근 가능하기 때문에 **외부 혹은 다른 Pod가 안정적으로 접근**하기 어렵습니다.  
이 문제를 해결하기 위해 사용하는 것이 바로 `Service`입니다.

> Service는 **Pod 앞단에 고정된 접근 포인트를 제공**하여, 안정적인 통신을 가능하게 해줍니다.

---

## ✅ 1. Service란?

- **Pod 집합에 대해 고정된 접근 경로를 제공하는 네트워크 리소스**
- Pod의 생성/삭제와 관계없이 **일관된 IP와 DNS 이름으로 접근 가능**
- 내부 DNS: `서비스명.네임스페이스.svc.cluster.local`

---

## ✅ 2. Service 작동 원리

- Service는 `selector`를 사용하여 특정 `label`을 가진 Pod를 연결
- 연결된 Pod 목록을 `Endpoints` 리소스로 관리
- kube-proxy가 각 노드에 프록시를 구성해 트래픽을 Pod로 라우팅

---

## ✅ 3. Service의 주요 타입

| 타입 | 설명 | 용도 |
|------|------|------|
| **ClusterIP** | 기본값. 클러스터 내부 IP로만 접근 가능 | 내부 통신 |
| **NodePort** | 노드의 포트를 열어 외부에서 접근 가능 | 간단한 외부 노출 |
| **LoadBalancer** | 클라우드의 L4 로드밸런서 연결 | 실서비스 외부 노출 |
| **ExternalName** | 외부 도메인으로 프록시 | 외부 서비스 연결 |

---

## ✅ 4. ClusterIP

클러스터 내부에서만 접근 가능한 가장 기본적인 Service 타입입니다.

### 🔧 예시 YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### ✅ 특징

- 외부에서는 접근 불가
- `kubectl port-forward`를 통해 임시 접근 가능

```bash
kubectl port-forward svc/nginx-clusterip 8080:80
```

---

## ✅ 5. NodePort

모든 노드의 특정 포트를 열어 외부에서 접근 가능하게 합니다.

### 🔧 예시 YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

### ✅ 특징

- `http://<노드IP>:30080` 형태로 접근
- 포트 범위: 30000 ~ 32767
- Minikube에서는 `minikube service nginx-nodeport`로 열람 가능

---

## ✅ 6. LoadBalancer

클라우드 환경(GKE, EKS, AKS 등)에서 외부 L4 Load Balancer를 자동으로 연결합니다.

### 🔧 예시 YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

### ✅ 특징

- 외부 IP가 할당됨
- 퍼블릭 인터넷을 통해 직접 접근 가능
- 로컬 환경에서는 미지원 (별도 LB 구성 필요)

```bash
kubectl get svc nginx-lb
```

---

## ✅ 7. ExternalName

클러스터 외부의 DNS 이름으로 접근할 수 있게 합니다.

### 🔧 예시 YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-google
spec:
  type: ExternalName
  externalName: www.google.com
```

### ✅ 특징

- 내부 DNS 요청을 외부 도메인으로 프록시
- 실제 트래픽은 외부로 나감
- 포트 설정 불가 (`DNS CNAME` 형태로 작동)

---

## ✅ 8. 실습: 간단한 nginx Deployment + Service

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get svc
```

→ NodePort 방식으로 외부에서 접근 가능한 서비스 생성

---

## ✅ 9. 네임스페이스 간 접근 예시

- nginx 서비스가 `default` 네임스페이스에 있을 때, 다른 Pod에서 접근하는 방법:

```bash
curl http://nginx.default.svc.cluster.local
```

---

## ✅ 10. 요약 비교

| 타입 | 외부 접근 가능 | 클라우드 LB | 포트 제어 | 사용처 |
|------|----------------|-------------|-----------|--------|
| ClusterIP | ❌ | ❌ | 내부 포트만 | 내부 서비스 간 통신 |
| NodePort | ✅ (노드IP:포트) | ❌ | 30000~32767 | 개발용/테스트 |
| LoadBalancer | ✅ (외부 IP) | ✅ | 기본 포트 | 클라우드 서비스 |
| ExternalName | ✅ (DNS만) | ❌ | X | 외부 서비스 프록시 |

---

## ✅ 결론

`Service`는 쿠버네티스에서 애플리케이션을 **안정적으로 연결하고 외부에 노출하는 핵심 구성 요소**입니다.  
각 타입은 다음과 같은 상황에 사용됩니다:

- **ClusterIP**: 마이크로서비스 내부 통신
- **NodePort**: 빠르게 외부에 테스트 공개
- **LoadBalancer**: 실제 서비스 외부 노출
- **ExternalName**: 외부 API 연동