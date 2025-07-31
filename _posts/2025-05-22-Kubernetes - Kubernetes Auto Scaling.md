---
layout: post
title: Kubernetes - Kubernetes Auto Scaling
date: 2025-05-22 22:20:23 +0900
category: Kubernetes
---
# Kubernetes Auto Scaling 완벽 가이드  
## HPA, VPA, Cluster Autoscaler 개념과 실습

Kubernetes는 워크로드의 부하에 따라 **자동으로 확장(Scaling)** 할 수 있는 다양한 메커니즘을 제공합니다.

이 문서에서는 Kubernetes의 세 가지 주요 Auto Scaling 기능인:

- **HPA (Horizontal Pod Autoscaler)**  
- **VPA (Vertical Pod Autoscaler)**  
- **Cluster Autoscaler**

을 개념, 구성 방법, 실습 예제 중심으로 소개합니다.

---

## ✅ 1. HPA (Horizontal Pod Autoscaler) - 수평 확장

**Pod의 개수**를 자동으로 늘리거나 줄입니다.  
CPU 사용률 또는 사용자 정의 메트릭에 따라 작동합니다.

### 🔹 요구사항

- Metrics Server 설치 필수  
- Pod이 리소스를 `requests`로 정의해야 함

### 🔹 기본 사용 예시

```bash
kubectl autoscale deployment nginx-deploy --cpu-percent=50 --min=2 --max=10
```

→ CPU 사용률이 50% 이상이면 Pod 개수를 자동으로 증가시킴 (최대 10개)

### 🔹 YAML로 정의

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deploy
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### 🔹 커스텀 메트릭 기반 확장도 가능

예: QPS, Queue Length, Kafka Lag, 등

---

## ✅ 2. VPA (Vertical Pod Autoscaler) - 수직 확장

**Pod의 CPU/Memory 리소스 요청량 (`requests/limits`)**을 자동으로 조정합니다.

> 일정 시간이 지나면 Pod이 재시작되며 새로운 리소스 값으로 재배포됨

### 🔹 동작 방식

1. 컨트롤러가 Pod의 리소스 사용량을 수집
2. 추천값 계산
3. 재시작 시 반영 또는 수동 적용

---

### 🔹 설치 및 구성

```bash
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-<version>/vpa-crd.yaml
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-<version>/vpa-rbac.yaml
```

### 🔹 예제

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: nginx-deploy
  updatePolicy:
    updateMode: "Auto"
```

> `updateMode`: `"Auto"`, `"Off"`, `"Initial"`

---

## ✅ 3. Cluster Autoscaler - 노드 자동 확장

**클러스터의 노드 수 자체를 자동으로 조절**합니다.  
Pod을 스케줄할 공간이 부족하면 노드를 추가하고,  
불필요한 노드는 제거합니다.

### 🔹 전제 조건

- 클라우드 환경 (GKE, EKS, AKS 등)
- 노드 그룹 또는 Auto Scaling Group 사용

### 🔹 동작 방식

1. Pod 스케줄 불가능 → 노드 추가
2. 노드에 Pod이 없으면 → 노드 제거

---

### 🔹 GKE에서 예시 (자동 설정)

GKE는 Cluster Autoscaler를 UI 또는 명령어로 쉽게 설정 가능:

```bash
gcloud container clusters update my-cluster \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5 \
  --node-pool default-pool
```

---

### 🔹 Helm으로 직접 설치 (kubeadm, EKS 등)

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=ap-northeast-2
```

→ `cloudProvider` 별로 설정 다름

---

## ✅ HPA vs VPA vs Cluster Autoscaler

| 항목 | HPA | VPA | Cluster Autoscaler |
|------|-----|-----|---------------------|
| 대상 | Pod 수 | Pod 리소스 | Node 수 |
| 기준 | CPU, 메모리, 메트릭 | 리소스 사용량 | 스케줄 실패 |
| 즉시 반영 | O | X (재시작 필요) | O |
| 설치 필요 | Metrics Server | VPA 컨트롤러 | 클라우드 설정 or Helm |
| 클라우드 필요 | X | X | O (보통) |

---

## ✅ 조합 예시

- **HPA + Cluster Autoscaler**: 부하 따라 Pod 수 증가, 필요시 Node도 확장  
- **VPA + HPA (주의)**: 충돌 가능성 있음 → `VPA`는 `recommendation`만, `HPA`는 `replica` 조절

→ 실무에서는 HPA + Cluster Autoscaler 조합이 가장 일반적입니다.

---

## ✅ 실전 체크리스트

| 체크 항목 | 설명 |
|-----------|------|
| Metrics Server 설치 여부 | HPA, VPA 모두 필요 |
| Pod에 resource request 설정 | 미설정 시 오토스케일링 미작동 |
| PodDisruptionBudget 고려 | Node 축소 시 서비스 장애 방지 |
| VPA 재시작 동작 이해 | 무중단 서비스는 updateMode: Off 추천 |
| HPA & VPA 병행 여부 | 함께 쓰되 충돌에 주의 (v2beta2 기준 개선됨) |

---

## ✅ 마무리

Kubernetes의 오토스케일링 기능은 다음과 같은 목표를 달성합니다:

- **비용 최적화**: 필요할 때만 리소스 사용  
- **성능 확보**: 부하에 따라 신속한 확장  
- **운영 자동화**: 수작업 개입 최소화

적절한 조합으로 **효율적인 클러스터 운영**을 설계해보세요.

---

## ✅ 참고 링크

- [HPA 공식 문서](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [VPA 프로젝트 GitHub](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Cluster Autoscaler 공식 문서](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
