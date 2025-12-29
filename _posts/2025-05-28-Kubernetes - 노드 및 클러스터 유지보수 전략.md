---
layout: post
title: Kubernetes - 노드 및 클러스터 유지보수 전략
date: 2025-05-28 22:20:23 +0900
category: Kubernetes
---
# Kubernetes 노드 및 클러스터 유지보수 전략

## 서론: 체계적인 유지보수의 중요성

Kubernetes 클러스터의 운영은 단순히 애플리케이션 배포에 그치지 않습니다. 지속적인 유지보수 작업을 통해 클러스터의 안정성, 보안, 성능을 유지하는 것이 동등하게 중요합니다. 이 가이드는 OS 패치, Kubernetes 버전 업그레이드, 하드웨어 교체와 같은 다양한 유지보수 작업을 서비스 중단 없이 수행하는 체계적인 접근법을 제공합니다.

효과적인 유지보수 전략은 예측 가능성과 반복 가능성을 기반으로 합니다. 잘 정의된 프로세스, 적절한 도구, 명확한 롤백 계획을 통해 유지보수 작업을 일상적인 운영 활동으로 전환할 수 있습니다.

---

## 표준 유지보수 프로세스

### 1단계: 사전 점검 (Pre-flight Checks)
유지보수 작업을 시작하기 전에 클러스터 상태를 철저히 확인합니다. 이 단계는 잠재적인 문제를 조기에 발견하고 예방하는 데 중점을 둡니다.

### 2단계: 격리 (Isolation)
대상 노드가 새로운 워크로드를 수용하지 못하도록 제한합니다. 이는 기존 워크로드에 대한 영향을 최소화하면서 안전하게 작업할 수 있는 환경을 조성합니다.

### 3단계: 대피 (Evacuation)
노드에서 실행 중인 워크로드를 다른 정상 노드로 안전하게 이동시킵니다. 이 과정에서 애플리케이션의 가용성을 보장하는 것이 핵심입니다.

### 4단계: 작업 수행 (Maintenance Execution)
실제 패치, 업그레이드, 재부팅 등의 작업을 수행합니다. 이 단계에서는 작업의 영향 범위와 소요 시간을 명확히 이해하는 것이 중요합니다.

### 5단계: 복귀 (Restoration)
작업이 완료된 노드를 다시 클러스터에 통합하고, 정상적인 서비스 상태를 확인합니다.

### 6단계: 검증 (Verification)
애플리케이션 성능, 리소스 사용량, 오류율 등 다양한 지표를 통해 유지보수 작업의 성공 여부를 확인합니다.

### 7단계: 문서화 (Documentation)
작업 내용, 발견된 문제, 학습한 교훈을 기록하여 향후 유지보수 작업에 활용합니다.

---

## 핵심 개념: Cordon, Drain, Uncordon

### Cordon (격리)
노드를 "cordon" 상태로 표시하여 새로운 Pod가 해당 노드에 스케줄되지 않도록 합니다. 이는 유지보수 작업을 위해 노드를 준비하는 첫 번째 단계입니다.

```bash
kubectl cordon <노드-이름>
```

### Drain (대피)
노드에서 실행 중인 Pod를 안전하게 제거하고 다른 가용 노드로 재스케줄합니다. 이 과정에서 PodDisruptionBudget(PDB)을 존중하며, DaemonSet Pod는 기본적으로 제외됩니다.

```bash
kubectl drain <노드-이름> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60
```

### Uncordon (복귀)
노드의 cordon 상태를 해제하여 새로운 Pod가 다시 스케줄될 수 있도록 합니다.

```bash
kubectl uncordon <노드-이름>
```

## PodDisruptionBudget(PDB)의 이해와 중요성

PDB는 애플리케이션의 가용성을 보장하기 위한 Kubernetes의 핵심 기능입니다. 이는 자발적 중단(유지보수 작업) 동안 동시에 사용 불가능할 수 있는 Pod 복제본의 최대 수를 제한합니다.

### PDB의 수학적 표현
애플리케이션에 `r`개의 Pod 복제본이 있고, PDB가 `minAvailable: p%`로 설정된 경우:
```
최소 유지해야 할 Pod 수 = ceil(r × p/100)
동시에 중단 가능한 최대 Pod 수 = r - 최소 유지 Pod 수
```

예를 들어, 20개의 Pod 복제본과 `minAvailable: 90%` PDB가 있는 경우:
- 최소 18개의 Pod가 항상 실행 중이어야 함
- 최대 2개의 Pod만 동시에 중단 가능

이 계산은 유지보수 작업의 속도와 동시성을 결정하는 데 직접적인 영향을 미칩니다.

---

## 사전 점검 체크리스트

유지보수 작업을 시작하기 전에 다음 항목들을 확인해야 합니다:

### 클러스터 상태 확인
- **노드 상태**: 모든 노드가 Ready 상태인지 확인
- **Pod 상태**: 비정상적인 Pod나 지속적인 재시작이 없는지 확인
- **리소스 여유량**: 충분한 리소스 여유 공간이 있는지 확인

### 애플리케이션 보호 메커니즘 검증
- **PodDisruptionBudget**: 모든 중요 애플리케이션에 적절한 PDB가 설정되어 있는지 확인
- **Horizontal Pod Autoscaler**: HPA 설정이 적절한 최소 복제본 수를 보장하는지 확인
- **Readiness/Liveness 프로브**: 애플리케이션 상태 검사가 올바르게 구성되어 있는지 확인

### 백업 및 복구 준비
- **스냅샷**: 중요한 상태 유지 애플리케이션의 스냅샷 생성
- **롤백 계획**: 문제 발생 시 빠르게 이전 상태로 복구할 수 있는 계획 수립

### 모니터링 및 알림 설정
- **경보 시스템**: 유지보수 작업 중 발생할 수 있는 문제를 감지할 수 있는 경보 설정
- **대시보드**: 실시간으로 작업 진행 상황을 모니터링할 수 있는 대시보드 준비

---

## 고급 격리 전략: Taint와 Toleration

### Taint를 활용한 정밀한 격리
Cordon보다 더 강력한 격리가 필요한 경우, Taint를 사용할 수 있습니다:

```bash
# 유지보수용 Taint 추가
kubectl taint nodes <노드-이름> maintenance=planned:NoSchedule

# 작업 완료 후 Taint 제거
kubectl taint nodes <노드-이름> maintenance=planned:NoSchedule-
```

Taint의 장점:
- 특정 Toleration을 가진 Pod만 노드에 스케줄되도록 허용
- cordon보다 더 세분화된 제어 가능
- 시스템 Pod와 사용자 Pod를 다른 방식으로 처리 가능

### 노드 라벨링을 통한 역할 기반 관리
```bash
# 노드에 역할 라벨 추가
kubectl label node <노드-이름> node-role.kubernetes.io/worker=worker
```

라벨을 사용하면:
- 유지보수 작업 순서를 역할별로 관리 가능
- 특정 워크로드 그룹에 대한 유지보수 계획 수립 용이
- 모니터링과 보고에서 노드 그룹화 가능

---

## Drain 작업의 고급 전략과 문제 해결

### 최적화된 Drain 명령어
```bash
kubectl drain <노드-이름> \
  --ignore-daemonsets \          # DaemonSet Pod 무시
  --delete-emptydir-data \       # emptyDir 볼륨 데이터 삭제
  --grace-period=120 \           # Pod 종료 유예 기간(초)
  --timeout=30m \                # 전체 작업 시간 제한
  --disable-eviction \           # 노드 압력이 높을 때 유용
  --skip-wait-for-delete-timeout=10  # 삭제 대기 시간 제한
```

### 일반적인 Drain 실패 시나리오와 해결책

**PDB 제약으로 인한 대기**
- 증상: Drain이 무기한 대기 상태
- 해결: 
  - 배포의 복제본 수를 일시적으로 증가시켜 PDB 요구사항 충족
  - 운영 승인 하에 PDB를 일시적으로 완화 (신중하게 진행)

**Local PersistentVolume 사용**
- 증상: Local PV를 사용하는 Pod 이동 불가
- 해결:
  - 애플리케이션 다운타임 계획 수립
  - 스토리지 아키텍처 재검토 (네트워크 스토리지로 마이그레이션)

**DaemonSet 충돌**
- 증상: DaemonSet Pod가 노드 리소스를 점유
- 해결:
  - DaemonSet의 업데이트 전략 조정
  - 노드당 하나만 필요한 DaemonSet인지 확인

**Finalizer 문제**
- 증상: Pod 삭제가 Finalizer에서 멈춤
- 해결:
  - Finalizer를 검토한 후 적절히 제거
  - 애플리케이션의 종료 프로세스 개선

---

## 노드 수준 작업 수행 가이드

### 운영체제 및 커널 업데이트
```bash
# Ubuntu/Debian 기반 시스템
sudo apt update && sudo apt upgrade -y

# RHEL/CentOS 기반 시스템
sudo yum update -y

# 커널 업데이트 후 재부팅 필요 여부 확인
sudo needs-restarting -r
```

### 컨테이너 런타임 업데이트
```bash
# containerd 업데이트
sudo systemctl stop kubelet
sudo apt-get install -y containerd.io
sudo systemctl start containerd
sudo systemctl start kubelet

# 런타임 상태 확인
sudo systemctl status containerd
sudo ctr version
```

### Kubelet 업데이트
```bash
# Kubernetes 버전 호환성 확인
# kubelet 버전은 control plane보다 1 마이너 버전까지 낮을 수 있음

# kubelet 업데이트
sudo systemctl stop kubelet
sudo apt-get install -y kubelet=<버전>
sudo systemctl daemon-reload
sudo systemctl start kubelet
```

### 재부팅 전략
1. **라이브 패치 활용**: Canonical Livepatch 등으로 재부팅 필요성 최소화
2. **예약된 재부팅**: Kured 등의 도구로 자동화된 재부팅 관리
3. **단계적 재부팅**: 클러스터 용량을 고려한 단계적 재부팅 계획

---

## 클러스터 업그레이드 전략

### kubeadm 기반 클러스터 업그레이드
1. **컨트롤 플레인 업그레이드** (HA 환경에서는 한 번에 하나씩)
   ```bash
   kubeadm upgrade plan
   kubeadm upgrade apply v1.30.0
   ```

2. **워커 노드 업그레이드** (한 번에 하나씩)
   ```bash
   kubeadm upgrade node
   kubectl drain <노드> --ignore-daemonsets
   systemctl restart kubelet
   kubectl uncordon <노드>
   ```

### 관리형 Kubernetes 서비스 업그레이드
- **GKE**: Surge 업그레이드, 블루-그린 노드풀
- **EKS**: 노드그룹 교체, 롤링 업데이트
- **AKS**: 노드풀 업그레이드, 자동 업데이트 채널

### 무중단 업그레이드를 위한 설계 고려사항
1. **용량 계획**: 업그레이드 중에도 충분한 용량 유지
2. **트래픽 관리**: Ingress 컨트롤러와 로드 밸런서 구성
3. **상태 유지**: StatefulSet과 PersistentVolume 처리 전략
4. **모니터링**: 업그레이드 진행 상황 실시간 추적

---

## 리소스 압력 관리와 Eviction 정책

### Kubelet Eviction 설정
```yaml
# /var/lib/kubelet/config.yaml
evictionHard:
  memory.available: "200Mi"
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"
  imagefs.inodesFree: "5%"
evictionSoft:
  memory.available: "300Mi"
  nodefs.available: "15%"
evictionSoftGracePeriod:
  memory.available: "1m30s"
  nodefs.available: "1m30s"
```

### 디스크 공간 관리
```bash
# 컨테이너 로그 로테이션 설정
sudo tee /etc/docker/daemon.json <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

# 로그 로테이션 적용
sudo systemctl restart docker
```

### 이미지 가비지 컬렉션
```bash
# 사용하지 않는 컨테이너 이미지 정기적 정리
sudo docker image prune -a --filter "until=24h" --force
```

---

## 자동화 도구와 스크립트

### 안전한 유지보수 스크립트 예제
```bash
#!/bin/bash
set -euo pipefail

NODE=$1
TIMEOUT=1800  # 30분 타임아웃
START_TIME=$(date +%s)

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

check_timeout() {
    CURRENT_TIME=$(date +%s)
    ELAPSED=$((CURRENT_TIME - START_TIME))
    if [ $ELAPSED -gt $TIMEOUT ]; then
        log "ERROR: 작업 시간 초과"
        exit 1
    fi
}

# 1. 사전 점검
log "노드 상태 확인 중..."
kubectl get node $NODE -o wide

# 2. 격리
log "노드 격리 중..."
kubectl cordon $NODE

# 3. 대피
log "Pod 대피 중..."
kubectl drain $NODE \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --grace-period=120 \
    --timeout=600

# 4. 유지보수 작업 수행
log "유지보수 작업 수행 중..."
# 실제 유지보수 작업 코드 위치

# 5. 복구 확인
log "노드 복구 확인 중..."
while true; do
    NODE_STATUS=$(kubectl get node $NODE -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')
    if [ "$NODE_STATUS" = "True" ]; then
        break
    fi
    sleep 5
    check_timeout
done

# 6. 격리 해제
log "노드 격리 해제 중..."
kubectl uncordon $NODE

# 7. 검증
log "최종 검증 중..."
kubectl describe node $NODE | grep -i condition

log "유지보수 작업 완료"
```

### Kured: Kubernetes 재부팅 데몬
Kured는 자동화된 노드 재부팅을 관리하는 데몬입니다:
- 커널 업데이트 감지
- 유지보수 윈도우 내 재부팅 스케줄링
- PDB 준수 및 알림 통합

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kured
spec:
  template:
    spec:
      containers:
      - name: kured
        image: docker.io/weaveworks/kured:latest
        env:
        - name: REBOOT_SENTINEL_FILE
          value: /var/run/reboot-required
        - name: REBOOT_TIME_WINDOW
          value: "Mon-Fri 02:00-04:00"
```

---

## 거버넌스와 정책 강제

### Kyverno를 이용한 PDB 정책 강제
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-pdb-for-deployments
spec:
  validationFailureAction: enforce
  rules:
  - name: check-pdb-exists
    match:
      resources:
        kinds:
        - Deployment
    validate:
      message: "프로덕션 네임스페이스의 모든 Deployment는 PodDisruptionBudget이 필요합니다."
      pattern:
        metadata:
          namespace: "prod-*"
        # PDB 존재 여부는 별도의 컨텍스트 룰에서 검증
```

### 리소스 요청/제한 정책
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-requests
spec:
  rules:
  - name: validate-resources
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "모든 컨테이너는 CPU 및 메모리 requests를 지정해야 합니다."
      pattern:
        spec:
          containers:
          - resources:
              requests:
                memory: "?*"
                cpu: "?*"
```

---

## 모니터링 및 관측 가능성

### 핵심 모니터링 지표

**노드 수준 지표**:
- `kube_node_status_condition`: 노드 상태 조건
- `node_memory_MemAvailable_bytes`: 사용 가능 메모리
- `node_filesystem_avail_bytes`: 파일시스템 사용 가능 공간

**Pod 수준 지표**:
- `kube_pod_status_phase`: Pod 상태
- `container_cpu_usage_seconds_total`: CPU 사용량
- `container_memory_working_set_bytes`: 메모리 사용량

**애플리케이션 수준 지표**:
- HTTP 요청률, 에러율, 지연 시간
- 데이터베이스 연결 수, 쿼리 지연

### 경보 규칙 예시
```yaml
groups:
- name: node-maintenance
  rules:
  - alert: NodeDrainStalled
    expr: |
      time() - kube_pod_deletion_timestamp{job="kube-state-metrics"} > 300
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod 삭제가 5분 이상 지연됨"
      description: "Pod {{ $labels.pod }}의 삭제가 5분 이상 지연되고 있습니다."

  - alert: PDBViolationDuringMaintenance
    expr: |
      kube_poddisruptionbudget_status_desired_healthy - kube_poddisruptionbudget_status_current_healthy > 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "PDB 위반 감지"
      description: "{{ $labels.namespace }}/{{ $labels.poddisruptionbudget }}의 PDB가 위반되었습니다."
```

---

## 사고 대응 및 문제 해결

### 일반적인 문제 해결 시나리오

**Drain 작업 중단**
1. 현재 Drain 상태 확인: `kubectl get pods --all-namespaces --field-selector spec.nodeName=<노드>`
2. PDB 제약 확인: `kubectl describe pdb`
3. 문제 Pod 조사: `kubectl describe pod <문제-포드>`
4. 필요 시 강제 삭제: `kubectl delete pod <포드> --grace-period=0 --force`

**노드 재부팅 후 NotReady 상태**
1. Kubelet 로그 확인: `journalctl -u kubelet -n 50`
2. 컨테이너 런타임 상태 확인: `systemctl status containerd`
3. 네트워크 연결 확인: `ip addr show`, `ping <컨트롤플레인-IP>`
4. 인증서 갱신 필요 여부 확인

**리소스 부족으로 인한 Eviction**
1. 노드 리소스 사용량 확인: `kubectl describe node`
2. 큰 리소스를 사용하는 Pod 식별: `kubectl top pods --all-namespaces`
3. 리소스 요청/제한 조정 또는 노드 확장 검토

---

## 운영 모범 사례 종합

### 계획 수립 단계
1. **유지보수 일정 통신**: 관련 팀에 사전 통지 및 동의 획득
2. **영향 분석**: 영향을 받는 애플리케이션 및 사용자 식별
3. **롤백 계획**: 문제 발생 시 빠른 복구를 위한 명확한 계획 수립

### 실행 단계
1. **단계적 접근**: 한 번에 너무 많은 변경을 시도하지 않기
2. **모니터링 강화**: 작업 중 실시간 모니터링 강화
3. **문서화**: 모든 작업 단계와 결과 기록

### 사후 단계
1. **검증**: 모든 서비스가 정상적으로 작동하는지 확인
2. **성능 분석**: 작업 전후 성능 비교
3. **회고**: 성공 요인과 개선점 도출

### 지속적인 개선
1. **자동화**: 반복적인 작업 자동화
2. **테스트**: 스테이징 환경에서 유지보수 절차 테스트
3. **교육**: 팀원들과 지식 공유 및 교육

---

## 결론

Kubernetes 노드 및 클러스터 유지보수는 단순한 기술적 작업을 넘어서 체계적인 프로세스, 적절한 도구 활용, 명확한 커뮤니케이션의 조화가 필요한 종합적인 운영 활동입니다. 효과적인 유지보수 전략을 수립하고 실행하기 위해 다음과 같은 원칙을 준수하는 것이 중요합니다:

1. **예측 가능성**: 표준화된 프로세스를 통해 모든 유지보수 작업이 예측 가능하도록 합니다.
2. **자동화**: 반복적이고 오류가 발생하기 쉬운 작업은 자동화하여 일관성과 안정성을 보장합니다.
3. **안전성**: PodDisruptionBudget, 적절한 리소스 요청/제한, 견고한 상태 검사를 통해 애플리케이션 가용성을 보호합니다.
4. **가시성**: 종합적인 모니터링과 경보 시스템을 통해 작업 진행 상황과 영향을 실시간으로 파악합니다.
5. **학습 문화**: 각 유지보수 작업에서 얻은 교훈을 문서화하고 공유하여 지속적으로 개선합니다.

이 가이드에 제시된 접근법과 모범 사례를 적용하면, Kubernetes 클러스터 유지보수가 단순한 "필요한 악"이 아닌, 안정적이고 효율적인 클라우드 네이티브 인프라를 구축하는 핵심 역량으로 자리 잡을 수 있을 것입니다. 유지보수 작업을 두려워하기보다는, 잘 계획되고 실행된 유지보수가 시스템의 장기적인 건강과 성능에 기여하는 기회로 인식하는 문화를 조성하는 것이 무엇보다 중요합니다.