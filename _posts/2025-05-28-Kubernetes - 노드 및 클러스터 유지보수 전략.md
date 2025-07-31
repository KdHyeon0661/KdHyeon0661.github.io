---
layout: post
title: Kubernetes - 노드 및 클러스터 유지보수 전략
date: 2025-05-28 22:20:23 +0900
category: Kubernetes
---
# Kubernetes 노드 및 클러스터 유지보수 전략

Kubernetes 클러스터는 **지속적으로 운영**되어야 하며, 그 과정에서 노드 장애, 커널 업데이트, 패치 작업, 리소스 고갈 등 다양한 상황이 발생합니다.

안정적인 서비스 운영을 위해 다음과 같은 **노드 및 클러스터 유지보수 전략**이 필요합니다.

---

## ✅ 유지보수 주요 목적

- OS 및 커널 업데이트 (보안 패치)
- Kubelet, Container Runtime, Docker 등 버전 업데이트
- 노드 스케일 인/아웃
- 노드 장애 조치 및 대체
- 클러스터 전체 업그레이드
- 디스크/네트워크 문제 대응

---

## ✅ 노드 유지보수 절차

Kubernetes는 **노드를 드레이닝(Draining)**하여 안전하게 Pod을 다른 노드로 이동시킬 수 있도록 지원합니다.

### 🔹 1. 노드 cordon (스케줄 금지)

```bash
kubectl cordon <노드이름>
```

→ 해당 노드에 **새로운 Pod 배치가 금지됨**

---

### 🔹 2. 노드 drain (Pod 대피)

```bash
kubectl drain <노드이름> --ignore-daemonsets --delete-emptydir-data
```

- 모든 Pod을 안전하게 종료하고 다른 노드로 이동
- DaemonSet은 제외 (`--ignore-daemonsets`)
- `emptyDir`는 삭제됨 주의

> 💡 `PodDisruptionBudget`을 준수하면서 안전하게 처리됨

---

### 🔹 3. 유지보수 작업 수행

- OS 패치
- 커널 업데이트
- 재부팅
- Docker/Kubelet 업그레이드 등

---

### 🔹 4. 노드 복구 및 uncordon

```bash
kubectl uncordon <노드이름>
```

→ 다시 스케줄링 가능 상태로 전환

---

## ✅ drain 실패 시 처리법

- PDB (PodDisruptionBudget)로 인해 일부 Pod이 제거되지 않을 수 있음  
  → `kubectl get pdb`로 확인
- DaemonSet Pod은 자동으로 유지됨
- PVC가 local-path면 migration 불가

---

## ✅ DaemonSet 재배포 예시 (예: 로그 에이전트)

```bash
kubectl rollout restart daemonset <이름> -n <네임스페이스>
```

→ 각 노드에 재배포되며 최신 상태로 유지

---

## ✅ 클러스터 버전 업그레이드 전략

### 🔹 kubeadm 기반

1. Control Plane 노드부터 순차 업그레이드
2. `kubeadm upgrade plan`으로 버전 확인
3. 각 노드에 대해 `kubeadm upgrade node` 실행
4. kubelet 재시작

### 🔹 클라우드 기반(GKE, EKS, AKS)

- UI 또는 CLI로 노드 풀 롤링 업그레이드 제공
- 자동 업그레이드 기능도 사용 가능 (보안 패치 중심)

---

## ✅ OS 및 패키지 유지보수 자동화

- **unattended-upgrades** (Ubuntu)
- **yum-cron** (CentOS/RHEL)
- CRON + `kubectl cordon/drain` 스크립트 자동화
- 시스템 재부팅 후 `kubelet` 자동 복구 여부 확인

---

## ✅ 노드 장애 대처 전략

| 상황 | 대응 |
|------|------|
| 노드 Crash | 자동 감지 + Pod 재스케줄 |
| 노드 Disk Full | 로그 회수 정책 + `emptyDir` 사용 제한 |
| 메모리 누수 | 리소스 제한 + Prometheus Alert |
| Kubelet Down | `node-problem-detector`, `taint` 설정 활용 |

---

## ✅ 운영 팁

| 팁 | 설명 |
|-----|------|
| `taint`로 유지보수 노드 격리 | 수동 또는 자동으로 `NoSchedule` 설정 |
| `PodDisruptionBudget` 적용 | 중요 서비스는 유지보수 시 중단되지 않도록 보호 |
| `readinessProbe`/`livenessProbe` 적극 사용 | 안정적인 상태 감지 및 회복 유도 |
| 노드별 `label`로 역할 구분 | GPU, Worker, Ingress 등 목적별 분리 관리 |
| 로그 백업 및 롤링 설정 | 디스크 고갈 방지 (`logrotate`, `fluentbit` 활용) |

---

## ✅ 노드 자동 복구 예시 (클라우드)

- **EKS**: 노드 그룹에 AutoScaling 설정 → 장애 시 자동 대체  
- **GKE**: 노드 자동 복구 기능 제공 (Node Auto Repair)  
- **AKS**: VMSS 기반 자동 복구 가능

---

## ✅ 자주 쓰는 명령 요약

```bash
kubectl get nodes
kubectl describe node <노드명>
kubectl cordon <노드명>
kubectl drain <노드명> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <노드명>
```

---

## ✅ 결론

| 항목 | 요약 |
|------|------|
| 노드 유지보수 | cordon → drain → patch → uncordon |
| 클러스터 업그레이드 | kubeadm 또는 클라우드 관리 툴 활용 |
| 자동화 | cron, kubelet self-healing, 노드 복구 설정 |
| 안정성 확보 | PDB, 리소스 제한, 적절한 probe 설정 필요 |

정기적인 유지보수와 빠른 장애 대응 전략을 통해 **무중단 운영**이 가능해집니다.

---

## ✅ 참고 링크

- [kubectl drain 공식 문서](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#drain)
- [kubeadm 업그레이드 가이드](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [GKE 자동 복구 설정](https://cloud.google.com/kubernetes-engine/docs/how-to/node-auto-repair)
