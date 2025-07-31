---
layout: post
title: Kubernetes - Rolling Update & Rollback
date: 2025-05-28 21:20:23 +0900
category: Kubernetes
---
# Kubernetes 배포 전략: Rolling Update & Rollback 완전 정복

Kubernetes는 기본적으로 **중단 없는 배포(Rolling Update)**를 제공하고,  
문제가 생겼을 때는 빠르게 **롤백(Rollback)**할 수 있는 기능을 내장하고 있습니다.

이 글에서는 다음을 중심으로 설명합니다:

- Rolling Update의 작동 방식  
- 전략 설정 및 튜닝 옵션  
- Rollback 시나리오 및 방법  
- 실무 적용 전략

---

## ✅ Rolling Update란?

**기존 Pod을 점진적으로 교체**하면서 새로운 버전으로 배포하는 방식입니다.

> 모든 Pod을 한꺼번에 교체하지 않기 때문에 서비스 중단 없이 배포 가능

### 🔹 작동 원리

1. 새로운 ReplicaSet 생성
2. 새 Pod을 하나씩 배포
3. 기존 Pod을 하나씩 제거
4. 모든 Pod 교체 완료 시 배포 종료

---

### 🔹 기본 Deployment 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:v2
```

| 옵션 | 설명 |
|------|------|
| `maxSurge` | 새 Pod을 몇 개까지 더 띄울 수 있는가 (기본 25%) |
| `maxUnavailable` | 동시에 중단 가능한 기존 Pod 수 (기본 25%) |

> `replicas: 4`, `maxSurge: 1`, `maxUnavailable: 1` → 최대 5개, 최소 3개까지 유지

---

## ✅ 배포 상태 확인

```bash
kubectl rollout status deployment my-app
```

→ 새로운 버전 배포 중인지 확인

---

## ✅ 배포 이력 확인

```bash
kubectl rollout history deployment my-app
```

→ 과거의 배포 revision을 확인 가능

---

## ✅ Rollback이란?

**이전 버전으로 되돌리는 작업**입니다.  
배포된 버전이 문제를 일으킬 경우, 간단한 명령으로 복구할 수 있습니다.

### 🔹 롤백 명령어

```bash
kubectl rollout undo deployment my-app
```

→ 직전 버전으로 되돌림

---

### 🔹 특정 버전으로 롤백

```bash
kubectl rollout undo deployment my-app --to-revision=2
```

> `history` 명령으로 revision 번호 확인 후 사용

---

## ✅ 실전 전략: Canary 스타일 롤링 업데이트

전체 배포 전에 일부 트래픽만 새 버전에 보내는 전략입니다.

1. `replicas: 10`, `maxSurge: 2`, `maxUnavailable: 0`
2. readinessProbe로 새 Pod이 안정화될 때까지 wait
3. 트래픽 모니터링 (예: Prometheus + Grafana)
4. 이상 시 롤백

→ Helm, Argo Rollouts 등을 통해 정교한 Canary/Blue-Green 배포도 가능

---

## ✅ Rollback 시 유의사항

| 항목 | 설명 |
|------|------|
| Config 변경도 포함 | `Deployment`의 spec 전체가 롤백됨 (환경변수 포함) |
| 상태 저장 앱 주의 | DB 마이그레이션이 수반되는 앱은 롤백 시 역마이그레이션 불가 |
| CrashLoopBackOff 확인 | 이전 버전도 문제였다면 롤백 효과 없음

---

## ✅ 배포 전략 요약 비교

| 전략 | 설명 | 장점 | 단점 |
|------|------|------|------|
| Rolling Update | 기본 전략. 점진적 교체 | 무중단 배포 | 롤백도 점진적 |
| Recreate | 모든 Pod 중지 후 새로 생성 | 깔끔한 재시작 | 잠깐 중단 발생 |
| Canary (확장형) | 일부 트래픽에만 새 버전 적용 | 안전성 ↑ | 복잡도 ↑ |

→ `strategy.type: Recreate`로 설정하면 전체 중지 후 배포

---

## ✅ 실전 운영 팁

| 팁 | 설명 |
|-----|------|
| `readinessProbe` 설정 | 새 버전 Pod이 준비되었는지 확인하고 트래픽 연결 |
| `livenessProbe` 설정 | 잘못된 Pod을 자동으로 재시작 |
| 배포 후 `rollout status` 확인 | 오류가 있는지 즉시 파악 |
| `kubectl set image` 사용 | 실시간으로 이미지 업데이트 가능 |
| CI/CD 연동 | GitOps/Helm 등과 결합해 배포 자동화

---

## ✅ 결론

- **Rolling Update**는 기본이면서도 안전한 배포 전략  
- 문제가 생기면 **Rollout Undo** 명령으로 빠르게 복구 가능  
- 배포 전략은 앱의 특성과 리스크에 따라 유연하게 선택해야 함

---

## ✅ 참고 링크

- [Kubernetes Rolling Update 공식 문서](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)
- [kubectl rollout 명령어](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#rollout)
- [Helm을 통한 Canary 배포 예시](https://helm.sh/docs/howto/charts_tips_and_tricks/#automatically-roll-deployments)
