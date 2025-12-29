---
layout: post
title: Kubernetes - Pending 상태의 Pod 원인 찾기 및 해결 방법
date: 2025-05-31 23:20:23 +0900
category: Kubernetes
---
# Pending 상태의 Pod 원인 분석 및 해결 방법

`kubectl get pods` 명령 결과에서 `STATUS=Pending`은 Kubernetes가 Pod을 아직 어떤 노드에도 배치하지 못했거나, Pod 실행에 필요한 자원이 준비되지 않아 노드에 올라가지 못한 상태를 의미합니다.

---

## Pending 상태의 의미와 위치

Pending 상태는 Pod의 라이프사이클에서 **스케줄링 또는 준비 단계에서 멈춘 상태**를 나타냅니다. 이미 노드에 할당된 후 컨테이너 이미지를 다운로드하거나 볼륨을 마운트하는 과정에서도 일시적으로 Pending 상태로 표시될 수 있습니다.

### Pod 생성부터 실행까지의 간단한 흐름

1. `kubectl apply` 실행 또는 `Deployment` 생성
2. **스케줄러가 후보 노드를 평가** (요구 리소스, 라벨, 토폴로지, Taint, 정책 등)
3. 조건을 만족하는 노드가 있으면 **바인딩** → Kubelet이 **이미지 다운로드 / 볼륨 연결 / 마운트**
4. 컨테이너 시작(이후 `ContainerCreating` → `Running` 상태로 전환)

> 2~3 단계를 통과하지 못하면 Pod은 계속 **Pending** 상태로 유지됩니다.
> 3~4 단계에서 멈춰도 사용자에게는 Pending 또는 ContainerCreating 상태로 표시될 수 있습니다.

---

## 5분 안에 해결하는 빠른 진단 절차

| 단계 | 명령 | 확인 포인트 |
|---|---|---|
| 1 | `kubectl get pods -A` | 네임스페이스, Pod, 상태 파악(대상 범위 축소) |
| 2 | `kubectl describe pod <pod> -n <ns>` | **Events 섹션**(맨 아래)에서 스케줄 실패/볼륨 실패 메시지 확인 |
| 3 | `kubectl get events -A --sort-by='.lastTimestamp'` | 최신 이벤트 전체 추적 |
| 4 | `kubectl get nodes -o wide` | 노드 준비 상태, 가용 리소스, 라벨, 가용 영역 확인 |
| 5 | `kubectl describe node <node>` | `Taints`, `Allocatable`, `Conditions` (DiskPressure/MemoryPressure 등) 확인 |
| 6 | 원인별 추가 확인 | PVC: `kubectl get pvc -n <ns>` / Quota: `kubectl describe quota -n <ns>` / CSINode/StorageClass 등 |

> **핵심은 `describe pod`의 Events 섹션입니다.** 스케줄러, 볼륨, 이미지, 보안 정책 등의 실패 이유가 **문장 형태로** 기록되어 있습니다.

---

## 주요 원인별 분석 및 해결 방안

다음은 실제 환경에서 가장 흔하게 발생하는 Pending 원인을 "증상 → 예제 YAML → 해결책" 형태로 정리한 것입니다.

### 리소스 부족 (CPU/메모리/임시 스토리지)

**대표적인 이벤트 메시지**
```
0/3 nodes are available: 2 Insufficient memory, 1 Insufficient cpu.
```

**문제를 재현하는 YAML 예시**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hi-requests
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hi-requests
  template:
    metadata:
      labels:
        app: hi-requests
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: "8"         # 과도한 CPU 요청
            memory: "32Gi"   # 과도한 메모리 요청
          limits:
            cpu: "8"
            memory: "32Gi"
```

**해결 방안**
- 현실적인 `requests` 값으로 **조정** (스케줄링 기준은 `requests`입니다)
- **불필요한 Pod 축소** / **노드 증설** / **Cluster Autoscaler** 활성화
- 사용되지 않는 노드 리소스 회수, 리소스 재배분

> #### 스케줄링 가능성 간단 계산
> Pod들의 총 요청 CPU를 \(\sum r_i\), 노드별 할당 가능 CPU를 \(A_j\)라고 할 때,
> $$
> \sum r_i \le \sum A_j
> $$
> 조건을 만족하더라도 **배치 제약(라벨/가용 영역/Taint)** 때문에 실제 스케줄링이 실패할 수 있습니다. 총합이 충분하다고 해도 항상 스케줄링이 가능한 것은 아닙니다.

**임시 스토리지(Ephemeral Storage) 부족**
이벤트: `Insufficient ephemeral-storage`
→ 컨테이너 로그/캐시가 임시 스토리지를 과도하게 사용합니다. `resources.requests/limits.ephemeral-storage` 지정, 로그 로테이션/사이드카 정책 정비.

---

### PVC 미바인딩 (볼륨 대기 상태)

**대표적인 이벤트 메시지**
```
pod has unbound immediate PersistentVolumeClaims
no persistent volumes available for this claim
```

**문제를 재현하는 YAML 예시**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: default
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-gp3    # 클러스터에 존재하지 않는 StorageClass라고 가정
---
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
```

**해결 방안**
- `kubectl get storageclass`로 **StorageClass 존재 여부** 확인 및 수정
- 동적 프로비저닝이 지원되지 않는 환경에서는 **PV를 수동 생성**
- ReadWriteMany 접근 모드가 필요하면 NFS/CSI RWX 지원 스토리지로 전환
- **토폴로지(가용 영역) 불일치**: PVC, Pod, 노드의 **가용 영역 일치** 확인(특히 클라우드 블록 스토리지 사용 시)

---

### Node Taint → Toleration 누락

**대표적인 이벤트 메시지**
```
node(s) had taint {key: dedicated, value: gpu, effect: NoSchedule}, that the pod didn't tolerate
```

**진단 명령**
```bash
kubectl describe node <node> | grep -i taint -A2
```

**수정 예시 (Pod에 tolerations 추가)**
```yaml
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule
```

또는 운영자 측에서 유지보수용 Taint 해제:
```bash
kubectl taint node <node> dedicated=gpu:NoSchedule-
```

---

### NodeSelector/NodeAffinity 조건 불만족

**대표적인 이벤트 메시지**
```
0/5 nodes are available: 5 node(s) didn't match node selector.
```

**문제 예시**
```yaml
spec:
  nodeSelector:
    disktype: ssd   # 해당 라벨을 가진 노드가 없음
```

**해결 방법**
- 노드에 라벨 추가:
  ```bash
  kubectl label node <node> disktype=ssd
  ```
- 조건 완화: `preferredDuringSchedulingIgnoredDuringExecution`로 시작하여 점진적으로 강화

---

### Pod (Anti-)Affinity/Topology Spread 제약

**대표적인 이벤트 메시지**
```
0/6 nodes are available: 6 node(s) didn't satisfy existing pod anti-affinity rules.
```

**문제 예시**
```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: frontend
        topologyKey: "kubernetes.io/hostname"
```
→ 모든 노드에 이미 `app=frontend` Pod이 배치되어 있으면 **완전히 배치 금지** 상태가 되어 Pending이 됩니다.

**해결 방법**
- `required` → `preferred`로 완화하거나 topology 범위를 `zone`으로 확장
- **topologySpreadConstraints** 조합 시, 분산 정책과 충돌하지 않도록 **maxSkew**/selector 재점검

---

### ResourceQuota/LimitRange 초과

**대표적인 이벤트 메시지**
```
exceeded quota: compute-resources, requested: requests.cpu=1, used: requests.cpu=4, limited: 4
```

**진단 명령**
```bash
kubectl describe quota -n <ns>
kubectl get limitrange -n <ns>
```

**해결 방법**
- 불필요한 리소스 제거 또는 Quota 상향 조정
- LimitRange에 의해 **자동 적용되는 기본값**이 과도한 경우 조정

**LimitRange 예시 (기본값 자동 주입)**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: dev
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: 200m
      memory: 256Mi
    default:
      cpu: "1"
      memory: 512Mi
```

---

### 이미지 다운로드 문제 (ImagePullBackOff 전 단계)

**증상**
- `Pending` 또는 `ContainerCreating` 상태에 머물다가 `ImagePullBackOff`로 전환

**진단 명령**
```bash
kubectl describe pod <pod> | sed -n '/Events/,$p'
# Failed to pull image / back-off pulling image ...
```

**해결 방법**
- **이미지 경로/태그** 확인
- 프라이빗 레지스트리 사용 시 **imagePullSecrets** 설정
```yaml
spec:
  imagePullSecrets:
  - name: regcred
```

---

### GPU 요청/드라이버/RuntimeClass 불일치

**대표적인 이벤트 메시지**
```
0/4 nodes are available: 3 node(s) had no available GPU, 1 node(s) didn't match node selector.
```

**문제 예시**
```yaml
resources:
  requests:
    nvidia.com/gpu: "1"
```

**해결 방법**
- GPU 노드에 **NVIDIA Device Plugin** 배포 상태 확인
- GPU 수량/파티셔닝(MIG) 정책 부합 확인
- **RuntimeClass** 필요한 경우 지정:
```yaml
spec:
  runtimeClassName: nvidia
```

---

### PSA/OPA/Gatekeeper/Kyverno 정책 위반

**증상**
- `Forbidden` 또는 Admission webhook 거부 메시지
- `describe pod` 이벤트에 정책 위반 내용 명시됨

**해결 방법**
- 네임스페이스 라벨(PSA `baseline/restricted`)과 Pod의 `securityContext` 일치
- Gatekeeper Constraint/Template 로그 확인, 규칙 예외 조건 설정(만료일 포함 권장)

---

### Cluster Autoscaler와의 상호작용

**특징**
- 스케줄링 불가 상태가 지속되면 **Cluster Autoscaler가 새 노드 증설** 시도
- 단, **노드 풀 한도**, Taint, 가용 영역, 인스턴스 타입 제약으로 증설하지 못할 수 있음

**점검 방법**
- Cluster Autoscaler 로그/이벤트: **증설 실패 원인**이 기록됩니다(인스턴스 타입 불가, 제한 초과 등)

---

## 자동화된 진단 스니펫 모음

### Pending 원인 퀵 리포트 (쉘 스크립트)

```bash
#!/usr/bin/env bash

NS=${1:-default}
echo "== Pending pods in namespace:${NS} =="
kubectl get pods -n "$NS" --field-selector=status.phase=Pending -o name | while read -r P; do
  echo "---- $P ----"
  kubectl describe -n "$NS" "$P" | awk '/Events:/,0'
  echo
done
```

### PVC 상태 일괄 점검

```bash
kubectl get pvc -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,STATUS:.status.phase,SC:.spec.storageClassName,SIZE:.spec.resources.requests.storage
```

### 노드 가용 리소스 요약

```bash
kubectl get nodes -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.cpu}{"\t"}{.status.allocatable.memory}{"\n"}{end}'
```

---

## "원인 → 수정" 실전 시나리오

### 시나리오 A: 과도한 리소스 요청으로 스케줄링 실패

- **원인**: `requests.cpu=4`로 설정했지만 2코어 노드만 존재
- **수정**: `requests.cpu=500m`로 조정 또는 4코어 노드 풀 추가

### 시나리오 B: PVC Pending 상태

- **원인**: StorageClass 이름 오타
- **수정**: 올바른 StorageClass 이름으로 변경, 필요 시 PV 수동 생성

### 시나리오 C: Taint로 격리된 노드

- **원인**: `NoSchedule` Taint가 설정되었으나 Toleration 누락
- **수정**: Pod에 `tolerations` 추가 또는 Taint 제거

### 시나리오 D: Anti-Affinity가 너무 강한 제약

- **원인**: 모든 노드에 대상 라벨 Pod이 배치되어 완전 배치 금지 상태
- **수정**: `required` → `preferred`로 완화, topology 범위 조정

### 시나리오 E: Quota 초과

- **원인**: `requests.memory` 과다 사용으로 한도 초과
- **수정**: 오래된 워크로드 제거, Quota 상향, requests 조정

### 시나리오 F: 임시 스토리지 부족

- **원인**: 로그 폭증 / 큰 임시 파일 생성
- **수정**: `ephemeral-storage` requests/limits 지정, 로그 로테이션, 사이드카로 분리

### 시나리오 G: GPU 요청 문제

- **원인**: device plugin 미배포
- **수정**: NVIDIA plugin 설치, MIG/타입 일치 확인, `runtimeClassName` 설정

---

## 재현 가능한 학습용 데모

아래 예제들을 하나씩 실행해보며 Pending 상태에서 해결까지의 과정을 체험할 수 있는 학습용 세트입니다.

### PVC 미바인딩 데모

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
  storageClassName: doesnt-exist
---
apiVersion: v1
kind: Pod
metadata:
  name: demo-pvc-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: demo-pvc
```

**해결 방법**: 존재하는 StorageClass 이름으로 교체(예: `gp2`/`standard`), 동적 프로비저닝 지원 확인.

### Taint 미허용 데모

1) 노드에 Taint 추가:
```bash
kubectl taint node <node> dedicated=qa:NoSchedule
```

2) 일반 Pod 배포(대기 상태):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-taint
spec:
  containers:
  - name: c
    image: busybox
    command: ["sh","-c","sleep 3600"]
```

**해결 방법**: tolerations 추가:
```yaml
spec:
  tolerations:
  - key: dedicated
    value: qa
    operator: Equal
    effect: NoSchedule
```

### NodeSelector 불일치 데모

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-ns
spec:
  nodeSelector:
    env: prod
  containers:
  - name: c
    image: busybox
    command: ["sh","-c","sleep 3600"]
```

**해결 방법**: 실제 노드 라벨 확인 후 일치시키기:
```bash
kubectl label node <node> env=prod
```

### Quota 초과 데모

**Quota 설정**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq
  namespace: demo
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    pods: "2"
```

**과도한 요청을 하는 Pod**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-quota
  namespace: demo
spec:
  containers:
  - name: c
    image: nginx
    resources:
      requests:
        cpu: "2"
        memory: 2Gi
```

**해결 방법**: 요청 축소 또는 Quota 상향 조정.

---

## 운영 자동화: "Pending 알림 → 원인 요약 → 해결 제안"

Prometheus + Alertmanager를 활용하여 **`kube_pod_status_phase{phase="Pending"}`** 메트릭 비율 임계값 초과 시 알림을 생성하고, 아래 스크립트로 **이벤트 요약** 후 Slack 등의 채널에 전송할 수 있습니다.

```bash
#!/usr/bin/env bash

NS=${1:-default}
OUT=/tmp/pending_report.txt
echo "[Pending Report - $(date)]" > "$OUT"
for P in $(kubectl get pods -n $NS --field-selector=status.phase=Pending -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'); do
  echo "---- $P ----" >> "$OUT"
  kubectl describe pod "$P" -n "$NS" | awk '/Events:/,0' >> "$OUT"
done
cat "$OUT"
# curl -X POST <SLACK_WEBHOOK_URL> -H 'Content-type: text/plain' --data-binary @"$OUT"
```

---

## 현장 적용을 위한 점검 항목

- [ ] `describe pod`의 Events 섹션에서 **문장형 원인** 확인
- [ ] PVC/PV/StorageClass/CSI 상태 점검 (가용 영역/접근 모드 일치)
- [ ] 노드 `Taints`와 Pod `Tolerations`의 대응 관계 확인
- [ ] `nodeSelector`/`affinity`와 실제 노드 라벨 일치 여부
- [ ] `topologySpreadConstraints` / Pod (Anti-)Affinity 충돌 여부
- [ ] `ResourceQuota`/`LimitRange` 한도 확인
- [ ] CPU/메모리/임시 스토리지 requests 합리적 설정 여부
- [ ] 이미지 다운로드 권한/경로/태그 확인 (`imagePullSecrets`)
- [ ] GPU/RuntimeClass/Device Plugin 배포 상태 확인
- [ ] Cluster Autoscaler 증설 불가 사유(한도, 템플릿) 확인
- [ ] PSA/OPA/Gatekeeper/Kyverno 등 정책 거부 로그 확인

---

## 결론

Kubernetes에서 Pod의 Pending 상태는 단순한 "대기 중" 표시를 넘어 다양한 원인과 해결책이 존재하는 복합적인 문제입니다. 효과적인 문제 해결을 위해서는 체계적인 접근 방식이 필요합니다.

**첫 번째 단계**는 항상 `kubectl describe pod` 명령을 통해 Events 섹션을 확인하는 것입니다. 여기에 스케줄러가 남긴 실패 메시지가 명확한 문장 형태로 기록되어 있어 빠른 원인 파악이 가능합니다.

**두 번째 단계**는 문제의 계층을 식별하는 것입니다. 리소스 부족, 볼륨 문제, Taint/Toleration 불일치, 라벨/어피니티 제약, Quota 한도, 이미지 다운로드 문제, GPU/특수 하드웨어 요구사항, 정책 위반 등 다양한 계층에서 문제가 발생할 수 있습니다.

**예방적 차원**에서는 리소스 프로파일링을 통한 현실적인 요청 값 설정, 기본값 정책(LimitRange) 관리, 분산/토폴로지 설계의 신중한 계획, 볼륨 및 스토리지 클래스의 표준화, 정책 관측(OPA/PSA) 체계 구축, Cluster Autoscaler의 적절한 구성 등이 중요합니다.

Pending 문제는 단순한 기술적 문제를 넘어 **클러스터의 리소스 관리, 정책 준수, 운영 체계의 성숙도**를 반영하는 지표이기도 합니다. 체계적인 모니터링, 자동화된 알림, 명확한 문제 해결 프로세스를 통해 Pending 상태로 인한 서비스 영향도를 최소화하고, 지속적인 개선을 통해 더 견고한 Kubernetes 환경을 구축할 수 있습니다.
