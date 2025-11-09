---
layout: post
title: Kubernetes - 노드 및 클러스터 유지보수 전략
date: 2025-05-28 22:20:23 +0900
category: Kubernetes
---
# Kubernetes 노드 및 클러스터 유지보수 전략

## 0) 빠른 개요 — 표준 유지보수 사이클

1. **사전 점검**: PDB·HPA 상태, 노드 용량, 경보 정상 동작
2. **격리**: `cordon`(새 스케줄 금지) → **옵션**: `taint`로 강한 격리
3. **대피**: `drain`(PDB 준수, DaemonSet 제외)
4. **작업**: 커널/OS 패치, 런타임/쿠블릿 업데이트, 재부팅
5. **복귀**: 서비스 확인(ready/metrics) → `uncordon`
6. **검증**: 에러율/지연/포드 재시작/노드 압력·경고 확인
7. **보고/기록**: 변경 이력, 배운 점 개선

---

## 1) 개념 정리: Cordon/Drain/Uncordon + Eviction + PDB 수식

### 1.1 Cordon/Drain/Uncordon

```bash
# 새 파드 스케줄 금지
kubectl cordon NODE

# 파드 대피(DS 제외, emptyDir 삭제 인지)
kubectl drain NODE --ignore-daemonsets --delete-emptydir-data

# 정상 복귀
kubectl uncordon NODE
```

### 1.2 Eviction & PDB(중단 예산)

- Kubelet은 **자원 압력(NodeCondition: DiskPressure/MemoryPressure)** 시 **Eviction**으로 파드를 축출
- **PDB**는 축출/유지보수 중 **동시 중단 허용량**을 규정

**직관 수식**(최소 가용 파드 수):

$$
\text{minAvailable} = 
\begin{cases}
\lceil p \times r \rceil & \text{(백분율 규정)} \\
k & \text{(정수 규정)}
\end{cases}
$$

- \( r \): 대상 파드 총 개수, \( p \): 비율(예: 0.8), \( k \): 정수
- **유지보수로 동시에 비울 수 있는 최대 파드 수**는
$$
\text{maxDisruption} = r - \text{minAvailable}
$$

> Drain이 PDB를 위반하면 대기·실패 → **PDB 설계**가 유지보수 속도를 결정.

---

## 2) 사전 점검 체크리스트 (Preflight)

| 항목 | 확인 포인트 | 명령/방법 |
|---|---|---|
| PDB | minAvailable / maxUnavailable 현실성 | `kubectl get pdb -A` |
| HPA | 다운스케일 제한(예: minReplicas) | `kubectl get hpa -A` |
| CA(Cluster Autoscaler) | 임시 증설 허용?(Surge) | 클라우드 콘솔/파라미터 |
| 리소스 요청 | 파드 `requests/limits` 정의 여부 | `kubectl describe deploy` |
| 노드 상태 | `NotReady`/압력 여부 | `kubectl get nodes -o wide` |
| 스토리지 | Local PV/PVC 존재(이동 불가) | 워크로드 설계 문서 |
| 네트워킹 | CNI/Ingress/DS(로깅/모니터링) 영향 | `kubectl get ds -A` |
| 관측 | 경보/대시보드 정상 | Prometheus/Grafana/Alert |

---

## 3) 유지보수 격리: taint/toleration + label

### 3.1 유지보수 노드 강제 격리

```bash
kubectl taint nodes NODE maintenance=planned:NoSchedule
# 복구
kubectl taint nodes NODE maintenance=planned:NoSchedule-
```

- 일반 파드는 해당 taint를 **toleration** 하지 않으면 스케줄되지 않음
- **cordon**과 병행 시 더욱 안전

### 3.2 역할 라벨링

```bash
kubectl label node NODE role=ingress  # 예: 역할별 드레인 순서 관리
```

---

## 4) Drain 전략 튜닝 & 실패 처리

### 4.1 권장 drain 명령

```bash
kubectl drain NODE \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=20m \
  --force             # 대화형이 아닌 자동화에서 유용 (주의)
```

- `--grace-period`: 종료 유예
- `--timeout`: PDB 대기로 인한 장기 블로킹 방지
- `--force`: Controller가 관리하지 않는 파드도 제거(운영 정책에 맞게)

### 4.2 실패 공통 원인 & 조치

| 증상 | 원인 | 해결 |
|---|---|---|
| PDB 위반으로 대기 | `minAvailable` 과도 | 배포를 늘려余裕 확보 / PDB 완화(운영 승인 하) |
| Local PV 사용 | 이동 불가 | 워크로드 다운타임 창고지 / 스토리지 설계 변경 |
| DaemonSet 충돌 | DS가 포트를 점유 | DS 롤링 조정 / 용도별 우선순위 |
| Finalizer 걸림 | CR/파드 종료 대기 | Finalizer 해제(검토 후) |

---

## 5) 노드 작업: OS/커널/런타임/Kubelet

### 5.1 런타임(containerd) & kubelet 버전 정합

- **kubelet ≤ Control-Plane**(마이너 버전 차이 1 이내)
- 런타임(containerd/CRI-O) 지원 매트릭스 확인

### 5.2 시스템 서비스 관리 예시 (Ubuntu)

```bash
# 패치
sudo apt-get update && sudo apt-get -y dist-upgrade

# 런타임
sudo systemctl restart containerd

# 쿠블릿
sudo systemctl restart kubelet

# 재부팅
sudo reboot
```

### 5.3 커널/보안 패치 자동화

- Ubuntu: `unattended-upgrades`
- RHEL: `yum-cron` 또는 `dnf-automatic`
- **라이브패치**(Canonical Livepatch 등)로 재부팅 최소화

---

## 6) 복귀(uncordon) 전 검증 포인트

```bash
# 노드 Ready & 조건
kubectl describe node NODE | egrep -i 'Ready|Pressure'

# DS 정상 상태
kubectl get ds -A -o wide | grep NODE

# 파드 스케줄 & Readiness
kubectl get pods -A -o wide | grep NODE
```

**관측 기준**:
- 새 파드 **Ready 전환 시간 p95** ≤ 목표
- 서비스 **에러율/지연** SLO 내
- 노드 압력(메모리/디스크) **없음**

---

## 7) 클러스터 업그레이드 전략

### 7.1 kubeadm(자가 운영)

1. **컨트롤 플레인부터**(HA라면 하나씩)
2. `kubeadm upgrade plan` → `kubeadm upgrade apply vX.Y.Z`
3. 워커 노드: `kubeadm upgrade node`
4. 각 노드 **cordon/drain/작업/uncordon**

### 7.2 매니지드(KaaS)

- **GKE**: Surge Upgrade(동시 교체 파드/노드 수) → 업그레이드 창 지정
- **EKS/AKS**: 노드풀 롤링 업그레이드 → Auto Repair/Auto Upgrade 옵션

> **무중단 목표**라면 PDB·HPA·CA 조합을 먼저 설계하고 업그레이드 파라미터를 일치시켜야 함.

---

## 8) 디스크/메모리 압력과 Eviction 운영

### 8.1 Kubelet Eviction Thresholds (예시)

```yaml
# /var/lib/kubelet/config.yaml
evictionHard:
  memory.available: "200Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
```

- **이미지 GC** & **컨테이너 로그 로테이션**으로 DiskPressure 예방
- 운영 로깅(예: fluent-bit) **배출·보존 정책** 필수

### 8.2 로그 로테이션 예시

```bash
sudo tee /etc/logrotate.d/container-logs <<'EOF'
/var/log/containers/*.log {
  rotate 7
  daily
  compress
  missingok
  copytruncate
}
EOF
sudo logrotate -f /etc/logrotate.d/container-logs
```

---

## 9) HPA/CA/PDB 상호작용 설계

- **HPA**: 부하 중 다운스케일 방지 → `minReplicas` 적절
- **PDB**: 유지보수 중 동시에 비울 수 있는 파드 수를 보장
- **CA**: Drain으로 비는 노드 → 자동 감축 / Surge 필요 시 임시 증설

**권장 패턴**:  
- 중요 API: `PDB minAvailable: 80%`, `HPA minReplicas` 확보  
- 배치성: 느슨한 PDB 또는 미적용, 유지보수 창에서 일괄 처리

---

## 10) 운영 자동화 — 안전 스크립트 & Kured

### 10.1 안전 Drain/Uncordon 스크립트

```bash
#!/usr/bin/env bash
set -euo pipefail
NODE="$1"

echo "[+] Preflight"
kubectl get pdb -A
kubectl get node "$NODE" -o wide

echo "[+] Cordon"
kubectl cordon "$NODE"

echo "[+] Drain"
kubectl drain "$NODE" \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=30m

echo "[*] === Perform maintenance on $NODE ==="
# patch/reboot/kubelet/containerd ...

echo "[+] Wait for node to be Ready after reboot..."
until kubectl get node "$NODE" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' | grep -q True; do
  sleep 5
done

echo "[+] Uncordon"
kubectl uncordon "$NODE"

echo "[+] Verify"
kubectl describe node "$NODE" | egrep -i 'Ready|Pressure'
echo "[✓] Done"
```

### 10.2 Kured(자동 재부팅 데몬)

- 커널 업데이트 후 **재부팅 윈도우**에서 **자동 cordon/drain/reboot/uncordon**
- PDB 준수, 슬랙 알림 등 제공 → 야간 보안 패치 자동화

---

## 11) 정책 강제: “모든 배포는 PDB를 가져야 한다”

### 11.1 Kyverno 정책 예시

{% raw %}
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-pdb
spec:
  validationFailureAction: enforce
  rules:
  - name: pdb-exists
    match:
      any:
      - resources:
          kinds: ["Deployment"]
    context:
    - name: pdbs
      apiCall:
        urlPath: "/apis/policy/v1/namespaces/{{request.namespace}}/poddisruptionbudgets"
        jmesPath: "items[?selector.matchLabels.app=='{{ request.object.metadata.labels.app }}']"
    validate:
      message: "Deployment는 연관 PDB가 필요합니다."
      deny:
        conditions:
        - key: "{{ length(pdbs) }}"
          operator: Equals
          value: 0
```
{% endraw %}

> Gatekeeper(OPA)로도 유사한 제약 가능.

---

## 12) 관측/알림 — Prometheus 룰 & 대시보드

### 12.1 알림 룰 예시

```yaml
groups:
- name: k8s-maintenance
  rules:
  - alert: NodeNotReady
    expr: kube_node_status_condition{condition="Ready",status="true"} == 0
    for: 5m
    labels: {severity: critical}
    annotations:
      summary: "노드 NotReady"
      description: "노드가 5분 이상 NotReady 상태입니다."

  - alert: PodDisruptionStuck
    expr: max_over_time(kube_poddisruptionbudget_status_current_healthy[5m])
          < kube_poddisruptionbudget_status_desired_healthy
    for: 10m
    labels: {severity: warning}
    annotations:
      summary: "PDB 충족 불가"
      description: "현재 healthy 파드 수가 PDB 요구를 충족하지 못합니다."
```

### 12.2 대시보드 핵심 위젯
- **노드 Ready/압력** 타임라인
- **Drain 진행률**(노드별 파드 수 변동)
- **PDB 여유도** = `desired - current`
- **에러율/지연 p95/p99**(서비스 SLO)
- **Eviction 수/사유**(메모리/디스크 압력)

---

## 13) 사고 수습 런북(시나리오 기반)

### 13.1 Drain 중 PDB로 멈춤
1. `kubectl describe pdb`로 대상 파드·요구치 확인  
2. 배포 **replicas↑** 또는 canary 축소·트래픽 전환  
3. 운영 승인 하 **임시 PDB 완화** → 종료 후 원복

### 13.2 DiskPressure로 Eviction 급증
1. 노드 디스크 사용률, 이미지 캐시, 컨테이너 로그 확인  
2. **이미지 GC 임계** 낮춤, 로그 로테이션 적용  
3. 과도한 `emptyDir`/`hostPath` 점검 → 설계 개선

### 13.3 재부팅 후 NotReady 지속
1. `journalctl -u kubelet` / `containerd` 로그  
2. CNI/모듈/네트워크 링크 상태 점검  
3. 클라우드 메타데이터/라우팅 테이블 확인 → 필요 시 노드 교체

---

## 14) 무중단 업그레이드 플랜(예시)

- **윈도우**: 주 2회 02:00–04:00 (현지 시간)  
- **동시성**: AZ별 **Max 1 노드** (PDB 기반 계산)  
- **순서**: Ingress → Stateless API → Stateful(리더 선출 지원) → 배치  
- **검증 단계**: 각 단계마다 10분 관찰(에러율/p95/p99, Ready 전환 p95)  
- **롤백 기준**: 에러율 +1%p 또는 p99 > 800ms 5분 지속 시 중단·원복  
- **보고**: 변경 이력 PR + Grafana 스냅샷 첨부

---

## 15) 계산 예 — PDB와 유지보수 속도

서비스 파드 수 \( r=20 \), PDB가 `minAvailable: 90%`라면
$$
\text{minAvailable} = \lceil 0.9 \times 20 \rceil = 18,\quad
\text{maxDisruption} = 20 - 18 = 2
$$
→ **동시에 2개 파드**까지 축출 가능. 노드당 해당 파드 분포를 고려해 **동시 드레인 노드 수**를 결정.

---

## 16) 예제: “안전한 노드 롤링 유지보수” 파이프라인 YAML

### 16.1 크론잡으로 순차 유지보수 트리거(태그된 노드)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-node-maintenance
spec:
  schedule: "0 2 * * 2,5"  # 화/금 02:00
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: ops-bot
          containers:
          - name: maint
            image: bitnami/kubectl:1.30
            command: ["bash","-lc"]
            args:
            - |
              set -euo pipefail
              for NODE in $(kubectl get nodes -l maintenance=window -o name | cut -d/ -f2); do
                echo "=== $NODE ==="
                kubectl cordon $NODE
                kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --timeout=30m
                # 재부팅 예시(SSH/SSM 등 외부 채널과 연계)
                # ssh $NODE 'sudo reboot'
                echo "Waiting node Ready..."
                until kubectl get node $NODE -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' | grep -q True; do sleep 5; done
                kubectl uncordon $NODE
              done
          restartPolicy: OnFailure
```

> 실제 재부팅은 클라우드 SSM·이니트 스크립트·Kured와 연계.

---

## 17) 운영 팁 모음

- **Readiness/Liveness/Startup** 철저히: 유지보수 중 **트래픽 오작배분 방지**
- **Ingress/Gateway Pod 분리 배치**: 노드 교체 시 프런트 도메인 가용성 확보
- **노드당 파드 밀도** 상한 설정: Eviction/조정 비용↓
- **DS 최소화**: 드레인 저항 요소(로깅/모니터링/보안)는 경량·빠른 재기동 설계
- **Chaos/GameDay**: 분기별 “노드 드레인·자원 압력” 훈련
- **문서화**: “누가/언제/무엇을/왜” 남기는 **변경 이력**은 사고시 생명줄

---

## 18) 자주 쓰는 명령 요약

```bash
# 노드/상태
kubectl get nodes -o wide
kubectl describe node <NODE>

# 격리/대피/복귀
kubectl cordon <NODE>
kubectl drain <NODE> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <NODE>

# PDB/HPA
kubectl get pdb -A
kubectl get hpa -A

# Eviction/이벤트
kubectl get events -A --sort-by=.lastTimestamp
```

---

## 결론

- **표준 절차**(cordon → drain → patch/reboot → uncordon)와 **PDB·Eviction·HPA/CA**의 **정합**이 **무중단 유지보수**의 핵심입니다.
- 자동화(스크립트/Kured)와 정책(Kyverno/Gatekeeper), 관측(Prometheus/Grafana/Alert)을 결합하여 **예측 가능한 작업**으로 만드세요.
- 사전 설계(스토리지/DS/라벨·taint)와 **검증 가능한 롤백 기준**이 있으면, 야간 패치도 “일상 작업”이 됩니다.

---

## 참고(원칙·문헌·명령)
- `kubectl drain/cordon/uncordon`, PDB(policy/v1), Kubelet Eviction, Cluster Autoscaler, Kured, Kyverno/Gatekeeper, Prometheus Alerting
- 클라우드 제공 Surge/Auto Repair 옵션(GKE/EKS/AKS)