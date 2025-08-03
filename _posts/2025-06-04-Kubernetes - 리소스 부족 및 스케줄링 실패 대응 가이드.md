---
layout: post
title: Kubernetes - 리소스 부족 및 스케줄링 실패 대응 가이드
date: 2025-06-04 19:20:23 +0900
category: Kubernetes
---
# 리소스 부족 및 스케줄링 실패 대응 가이드 (Kubernetes)

Kubernetes 환경에서 `Pod`가 실행되지 않고 `Pending`, `FailedScheduling`, 또는 `Insufficient CPU/Memory`와 같은 메시지가 나타난다면, 이는 대부분 **리소스 부족**이나 **스케줄링 실패**로 인한 문제입니다.

이 문서에서는 다음을 설명합니다:

- 리소스 부족의 증상 및 원인
- 스케줄링 실패 원인과 진단 방법
- 실전 대응 전략 (Node 증설, 리소스 튜닝 등)

---

## ✅ 증상 요약

### 📌 Pod 상태 예시:

```bash
kubectl get pods

NAME            READY   STATUS    RESTARTS   AGE
myapp-xyz       0/1     Pending   0          2m
```

### 📌 `describe` 출력:

```bash
kubectl describe pod myapp-xyz
```

```
Warning  FailedScheduling  default-scheduler  0/3 nodes are available: 
  3 Insufficient memory.
```

---

## ✅ 주요 원인

| 유형 | 설명 |
|------|------|
| **리소스 부족 (CPU/Memory)** | Pod가 요청한 리소스를 충족할 수 있는 Node가 없음 |
| **Node Affinity 불일치** | 지정한 조건을 만족하는 Node가 없음 |
| **Taint / Toleration 미일치** | Node가 taint 되어 있고, Pod가 이를 toleration하지 않음 |
| **Pod 수 제한 (Quota)** | Namespace에 할당된 Pod 수 또는 리소스를 초과 |
| **PVC 바인딩 실패** | 스토리지가 아직 할당되지 않음 |

---

## ✅ 리소스 부족 (CPU/Memory)

Pod의 `resources.requests` 또는 `limits`가 현재 클러스터에서 사용 가능한 리소스를 초과할 경우, 해당 Pod는 Pending 상태로 유지됩니다.

### 🔍 진단 방법

```bash
kubectl describe pod <pod-name>
```

```
0/3 nodes are available: 
  2 Insufficient cpu, 
  1 Insufficient memory
```

### 📌 확인 명령어

```bash
kubectl top nodes
```

→ 현재 각 노드의 CPU/Memory 사용량 확인 가능

---

## ✅ 스케줄링 실패 유형별 대응

### 🔹 1. 리소스 부족 시 대응

| 방법 | 설명 |
|------|------|
| **Node 수 확장** | 클러스터에 노드 추가 (클라우드에서는 Auto Scaling도 가능) |
| **Pod 리소스 요청 조정** | `resources.requests` 값을 낮춰 작은 노드에도 스케줄 가능하도록 |
| **HPA 도입** | CPU 기반 AutoScaler를 도입해 부하에 따라 Pod 수 조절 |
| **VPA 도입** | 리소스 요청 자동 조정으로 낭비 줄이기 |

### 📌 Deployment 예시

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

→ 요청(requests)은 스케줄링 기준, 제한(limits)은 실제 사용 한도

---

### 🔹 2. Taint/Toleration 미일치

```bash
kubectl describe node <node-name>
```

→ `Taints: dedicated=infra:NoSchedule` 와 같은 항목 확인

#### 해결:

- Pod에 tolerations 추가

```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "infra"
  effect: "NoSchedule"
```

---

### 🔹 3. Node Affinity 조건 불충족

PodSpec에 정의된 조건이 클러스터 내 어떤 Node도 만족하지 않으면 스케줄링 실패

#### 예시:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: disktype
              operator: In
              values:
                - ssd
```

→ `disktype=ssd` 라벨이 없는 노드는 스케줄링 제외됨

#### 해결:

- Node에 해당 label 추가

```bash
kubectl label nodes <node-name> disktype=ssd
```

- 조건을 완화하거나 optional 처리

---

### 🔹 4. ResourceQuota 초과

```bash
kubectl describe quota -n <namespace>
```

→ Pod 수, CPU/Memory 사용량이 quota의 hard limit을 초과했는지 확인

#### 해결:

- Quota 상향 조정
- 리소스를 덜 사용하는 Pod로 대체
- 불필요한 리소스 정리

---

### 🔹 5. PVC 바인딩 실패

스토리지 리소스가 아직 할당되지 않으면 Pod는 Pending 상태로 대기

```bash
kubectl get pvc
```

→ `STATUS: Pending`

#### 해결:

- StorageClass 확인
- 자동 프로비저닝 가능한지 점검
- 수동 PV 바인딩

---

## ✅ 실전 대응 전략 요약

| 전략 | 설명 |
|------|------|
| 리소스 튜닝 | 필요 이상으로 설정된 requests/limits 재검토 |
| Node 확장 | 수평 확장 (HPA + Cluster Autoscaler) 고려 |
| Preemption 정책 | 낮은 우선순위 Pod를 제거하고 고우선 Pod 스케줄링 |
| 모니터링 도입 | Grafana + Metrics Server로 자원 사용량 지속 추적 |
| 우선순위 클래스 | `priorityClassName`으로 중요 Pod 보호 가능

---

## ✅ 모니터링 명령어 모음

| 목적 | 명령어 |
|------|--------|
| Pod 상태 확인 | `kubectl get pods` |
| 스케줄 실패 로그 | `kubectl describe pod <pod>` |
| 노드 사용량 | `kubectl top nodes` |
| 전체 이벤트 확인 | `kubectl get events --sort-by='.lastTimestamp'` |
| PVC 상태 확인 | `kubectl get pvc` |

---

## ✅ 결론

| 요약 항목 | 내용 |
|-----------|------|
| 원인 | CPU/Memory 부족, taint/toleration 미일치, node 조건 불일치 |
| 진단 도구 | `kubectl describe pod`, `top`, `get events`, `describe node` |
| 대응 전략 | 리소스 요청 튜닝, 노드 확장, 우선순위 설정, 스케일링 도입 |
| 권장 | Autoscaler(HPA, VPA, Cluster Autoscaler), 모니터링 도입 필수

---

## ✅ 참고 링크

- [Kubernetes Scheduling 공식 문서](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Resource Management 문서](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Cluster Autoscaler 사용법](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
