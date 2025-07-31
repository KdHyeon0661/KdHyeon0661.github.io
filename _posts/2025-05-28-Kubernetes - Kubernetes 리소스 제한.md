---
layout: post
title: Kubernetes - Kubernetes 리소스 제한
date: 2025-05-28 19:20:23 +0900
category: Kubernetes
---
# Kubernetes 리소스 제한 (CPU, Memory Requests/Limits) 완전 정복

Kubernetes는 컨테이너 기반으로 작동하기 때문에, **각 Pod/Container가 얼마나 많은 리소스를 사용할 수 있는지 명시적으로 지정**해주는 것이 매우 중요합니다.

이를 통해 다음과 같은 목적을 달성할 수 있습니다:

- **클러스터 자원 낭비 방지**
- **서비스 안정성 확보**
- **오토스케일링 기반 마련**
- **스케줄러의 올바른 판단 유도**

---

## ✅ 핵심 용어 정리

| 용어 | 설명 |
|------|------|
| `requests` | Pod이 **최소한으로 보장받는 자원량**<br>(스케줄링 기준) |
| `limits` | Pod이 **최대로 사용할 수 있는 자원량**<br>(초과 시 제한 or 종료) |

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

---

## ✅ CPU 단위 해설

- `1` CPU = 1 vCPU = 1 코어
- `500m` = 0.5 core
- `100m` = 0.1 core

> `cpu`는 공유 가능 자원 → `limit` 초과 시 Throttling 발생

---

## ✅ Memory 단위 해설

- `128Mi` = 128 Mebibyte
- `1Gi` = 1024 Mi

> `memory`는 비공유 자원 → `limit` 초과 시 **OOMKilled (Out Of Memory)** 발생

---

## ✅ 리소스 제한 예제 (Deployment)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample
  template:
    metadata:
      labels:
        app: sample
    spec:
      containers:
      - name: app-container
        image: nginx
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

→ Pod 1개당 CPU 0.1 core를 보장, 최대 0.5 core까지 사용 가능  
→ 메모리는 최소 128Mi를 확보하고, 256Mi 초과 시 OOM

---

## ✅ 왜 requests/limits를 명시해야 하는가?

| 이유 | 설명 |
|------|------|
| **스케줄링 정확도** | 노드에 Pod을 적절히 분산하기 위해 `requests` 필요 |
| **리소스 보호** | 과도한 사용 시 다른 Pod에 영향 주지 않도록 `limits` 설정 |
| **오토스케일링 작동** | HPA, VPA는 `requests` 기반으로 판단 |
| **비용 최적화** | 적절한 설정으로 리소스 낭비 방지

---

## ✅ 실전 운영 팁

### 1. 너무 작게 잡으면?

- 서비스 느려짐, OutOfMemory, 과도한 Throttling
- 예: `requests.cpu: 10m` → Throttled 지속

### 2. 너무 크게 잡으면?

- 스케줄 불가, 노드 낭비, 오토스케일러 작동 지연
- 예: `requests.memory: 2Gi` → 2Gi 남은 노드에서만 스케줄됨

---

## ✅ 추천 설정 전략

| 전략 | 설명 |
|------|------|
| **리소스 프로파일링** | 프로메테우스, Datadog 등으로 사용량 분석 후 설정 |
| **CPU는 낮게, limit만 제한** | HPA 활용 시 유리 (ex: `requests: 200m`, `limits: 1`) |
| **Memory는 여유 있게 설정** | OOM 방지 목적, `limits`는 반드시 필요 |
| **초기엔 동일값 사용** | `requests = limits` 로 시작하고 후에 조정

---

## ✅ 리소스 설정 확인 명령

```bash
kubectl describe pod <pod-name>
kubectl top pod
kubectl get pod <pod-name> -o=jsonpath='{.spec.containers[*].resources}'
```

---

## ✅ LimitRange로 기본값 강제하기 (네임스페이스 단위)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
```

→ 해당 네임스페이스에서 리소스 미지정 시 위 값을 자동 적용

---

## ✅ 리소스 초과 시 행동 요약

| 자원 | 초과 시 결과 |
|------|--------------|
| CPU Limit 초과 | 사용량 Throttling (성능 저하) |
| Memory Limit 초과 | OOMKilled (Pod 재시작) |
| CPU/Memory Request 초과 | 스케줄링 불가 (Pending) |

---

## ✅ 결론

| 설정 | 의미 | 초과 시 동작 |
|------|------|---------------|
| `requests` | 최소 보장 (스케줄 기준) | 없으면 스케줄 안 됨 |
| `limits` | 최대 허용량 (초과 금지) | CPU: 제한 / Memory: 종료 |

**최소한의 `requests`, 적절한 `limits`는 클러스터의 안정성과 확장성을 보장하는 핵심입니다.**

---

## ✅ 참고 링크

- [Kubernetes 공식 리소스 제한 문서](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [OOMKilled 이유 분석](https://kubernetes.io/docs/tasks/debug/debug-application/determine-reason-pod-failure/)
- [LimitRange 공식 문서](https://kubernetes.io/docs/concepts/policy/limit-range/)