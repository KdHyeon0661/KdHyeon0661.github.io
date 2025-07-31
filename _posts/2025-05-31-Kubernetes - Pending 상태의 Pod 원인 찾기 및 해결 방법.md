---
layout: post
title: Kubernetes - Pending 상태의 Pod 원인 찾기 및 해결 방법
date: 2025-05-31 23:20:23 +0900
category: Kubernetes
---
# Pending 상태의 Pod 원인 찾기 및 해결 방법

Kubernetes에서 `kubectl get pods` 명령어를 사용했을 때 `STATUS`가 `Pending`으로 나오는 경우, 이는 **Pod가 아직 Node에 스케줄되지 않았거나, 필요한 리소스를 충족하지 못해 실행 대기 중**이라는 의미입니다.

이 문서에서는 Pending 상태의 주요 원인과 이를 진단하고 해결하는 구체적인 방법을 안내합니다.

---

## ✅ Pending이란?

> `Pending` 상태는 **Pod가 스케줄 되었지만 아직 Node에서 실행되지 않은 상태**를 의미합니다.

### 🔹 일반적인 흐름
1. `kubectl apply` 또는 `Deployment` 생성
2. Scheduler가 적절한 Node를 찾음
3. Node에 리소스가 충분하면 Pod가 실행됨
4. 리소스 부족, PVC 미할당, 노드 taint 등 이슈가 있으면 → `Pending` 상태 유지

---

## ✅ 진단 절차 요약
ㄴ
| 단계 | 명령어 | 설명 |
|------|--------|------|
| 1 | `kubectl get pods` | Pending 상태 확인 |
| 2 | `kubectl describe pod <pod-name>` | 이유, 이벤트 메시지 확인 |
| 3 | `kubectl get nodes` | 노드 상태 확인 |
| 4 | `kubectl get events` | 클러스터 전체 이벤트 보기 |

---

## ✅ 원인별 분석

---

### 🔹 1. **리소스 부족 (CPU/Memory)**

Pod가 요청한 `resources.requests` 값이 현재 클러스터 노드들의 가용 자원을 초과하면 스케줄링되지 못합니다.

#### 예시 로그:
```
0/3 nodes are available: 3 Insufficient memory.
```

#### 해결 방법:
- `resources.requests` 값을 낮추거나
- 더 많은 노드를 추가하거나
- 불필요한 Pod를 축소/종료하여 자원 확보

---

### 🔹 2. **PersistentVolumeClaim (PVC) 대기**

Pod가 사용하는 PVC가 아직 바인딩되지 않았을 경우 `Pending` 상태로 대기합니다.

#### 확인 방법:
```bash
kubectl get pvc
```

→ `STATUS: Pending` 인 PVC가 있을 경우

#### 해결 방법:
- StorageClass 확인
- PV가 수동으로 필요하다면 명시적으로 생성
- 클라우드에서는 해당 볼륨 타입이 존재하는지 확인 (EBS, PD 등)

---

### 🔹 3. **Node에 Taint 설정됨**

Node에 `taint`가 설정되어 있으면, Pod가 `toleration`을 명시하지 않는 한 배치되지 않습니다.

#### 확인 방법:
```bash
kubectl describe node <node-name>
```

→ `Taints:` 항목 확인

#### 해결 방법:
- Pod에 `tolerations` 추가
- 또는 노드에서 `taint` 제거

```bash
kubectl taint node <node-name> key=value:NoSchedule-
```

---

### 🔹 4. **Node Selector / Affinity 미충족**

Pod에 `nodeSelector`, `nodeAffinity`가 설정돼 있는데 조건을 만족하는 노드가 없으면 Pending 됩니다.

#### 예시:
```yaml
spec:
  nodeSelector:
    disktype: ssd
```

→ `disktype=ssd` 레이블이 있는 노드가 없으면 Pending

#### 해결 방법:
- 해당 레이블을 가진 노드가 존재하는지 확인
- 레이블을 추가하거나 조건을 수정

```bash
kubectl label nodes <node-name> disktype=ssd
```

---

### 🔹 5. **이미지 Pull 실패**

`Pending` 상태는 아니지만, Pod가 `ContainerCreating` → `ImagePullBackOff`으로 이어질 수 있음

#### 확인:
```bash
kubectl describe pod <pod-name>
```

→ `Failed to pull image` 메시지

#### 해결:
- 이미지 경로 또는 태그 확인
- 프라이빗 이미지일 경우 Secret 설정 필요

---

### 🔹 6. **Pod 수 제한 (ResourceQuota)**

Namespace에 설정된 `ResourceQuota`로 인해 Pod 생성이 제한될 수 있습니다.

#### 확인:
```bash
kubectl describe quota --namespace=<namespace>
```

→ `used >= hard` 조건이면 제한됨

#### 해결:
- Quota 상향 조정
- 불필요한 리소스 정리

---

### 🔹 7. **PodSecurityPolicy / Admission Controller 차단**

보안 정책에 의해 Pod 스케줄링이 거부될 수 있음 (1.25 이전 버전 또는 PSP를 모방한 정책 사용 시)

#### 해결:
- PSP 정책 조건 충족 여부 확인
- PodSpec 내 `securityContext`, `runAsUser` 등 확인

---

## ✅ 실전 예시: PVC 바인딩 실패

```bash
kubectl describe pod myapp-123

Warning  FailedScheduling  default-scheduler
no persistent volumes available for this claim
```

#### 원인:
- PVC가 요청한 `StorageClass`에 해당하는 PV가 없음

#### 해결:
- 해당 StorageClass에 맞는 PV를 수동 생성하거나
- 자동 프로비저닝이 가능한지 확인

---

## ✅ 운영 팁

| 팁 | 내용 |
|-----|------|
| `kubectl get events --sort-by='.lastTimestamp'` | 최근 이벤트 순으로 확인 가능 |
| `kubectl describe pod` | Scheduling 실패 이유가 상세히 나옴 |
| 클러스터 모니터링 툴 | K9s, Lens 등을 활용해 직관적으로 상태 확인 가능 |
| Pending 오래 유지 시 | `ttlSecondsAfterFinished` 또는 자동 정리 설정 고려 |

---

## ✅ 결론

| 요약 항목 | 설명 |
|-----------|------|
| 의미 | Pending = 아직 Node에 배치되지 않은 Pod |
| 주 원인 | 리소스 부족, PVC 미할당, Taint/Toleration 불일치 등 |
| 진단 명령 | `describe pod`, `get events`, `describe node` |
| 해결 전략 | 상황에 맞는 조건 수정 및 리소스 확보

---

## ✅ 참고 링크

- [Pod Scheduling 공식 문서](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Persistent Volume 가이드](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Taint/Toleration 설명](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
