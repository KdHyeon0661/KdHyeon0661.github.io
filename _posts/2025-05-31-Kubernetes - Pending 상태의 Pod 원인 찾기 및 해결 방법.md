---
layout: post
title: Kubernetes - Pending 상태의 Pod 원인 찾기 및 해결 방법
date: 2025-05-31 23:20:23 +0900
category: Kubernetes
---
# Pending 상태의 Pod 원인 찾기 및 해결 방법

`kubectl get pods` 결과의 `STATUS=Pending`은 **스케줄러가 Pod을 아직 어떤 노드에도 배치하지 못했거나**(스케줄링 단계),  
**Pod 실행에 필수 자원이 준비되지 않아**(예: PVC 바인딩) **노드에 올라가지 못한 상태**를 뜻합니다.

아래 가이드는 **원인별 신속한 진단→수정**을 목표로,  
**재현 예제(YAML)**와 **검증 명령, 자동화 스크립트**까지 포함한 **현장형 런북**입니다.

---

## 0) Pending의 정확한 의미: 흐름 상의 위치

> **Pending = 스케줄링 또는 준비 단계에서 멈춘 상태**  
> (이미 Node에 할당된 뒤 컨테이너 이미지를 풀거나, 볼륨을 attach/마운트하는 과정에서도 Pending으로 보일 수 있음)

### 생성→스케줄링→실행 간단 흐름

1. `kubectl apply` / `Deployment` 생성  
2. **스케줄러가 후보 노드를 평가** (요구 리소스, 라벨, 토폴로지, taint, 정책 등)  
3. 조건 만족 노드가 있으면 **바인딩** → Kubelet이 **이미지 pull / 볼륨 attach / mount**  
4. 컨테이너 시작(이후 `ContainerCreating` → `Running`)

> 2~3 단계를 통과하지 못하면 계속 **Pending**입니다.  
> 3~4 단계에서 멈춰도 사용자 눈엔 Pending/ContainerCreating으로 보일 수 있습니다.

---

## 1) 5분 런북: 가장 빠른 진단 절차

| 단계 | 명령 | 확인 포인트 |
|---|---|---|
| 1 | `kubectl get pods -A` | 네임스페이스/Pod/상태 파악(대상 좁히기) |
| 2 | `kubectl describe pod <pod> -n <ns>` | **Events**(맨 아래)에서 스케줄 실패/볼륨 실패 메시지 |
| 3 | `kubectl get events -A --sort-by='.lastTimestamp'` | 최신 이벤트 전체 추적 |
| 4 | `kubectl get nodes -o wide` | 노드 준비 상태/가용 리소스/라벨/존 확인 |
| 5 | `kubectl describe node <node>` | `Taints`, `Allocatable`, `Conditions` (DiskPressure/MemoryPressure 등) |
| 6 | 원인별 툴 | PVC:`kubectl get pvc -n <ns>` / Quota:`kubectl describe quota -n <ns>` / CSINode/StorageClass 등 |

> **핵심은 `describe pod`의 Events**. 스케줄러·볼륨·이미지·보안정책 등 실패 이유가 **문장으로** 기록됩니다.

---

## 2) 빈 패턴별 원인 → 재현 예제 → 해결책

아래는 **실제에서 가장 흔한 Pending 원인**을 “증상 → 예제 YAML → 수정안” 형태로 정리했습니다.

### 2.1 리소스 부족(Insufficient CPU/Memory/EphemeralStorage)

**증상(Event 예시)**  
```
0/3 nodes are available: 2 Insufficient memory, 1 Insufficient cpu.
```

**문제 재현 YAML**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hi-requests
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels: {app: hi-requests}
  template:
    metadata: {labels: {app: hi-requests}}
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: "8"         # 과도한 요청
            memory: "32Gi"   # 과도한 요청
          limits:
            cpu: "8"
            memory: "32Gi"
```

**해결 방안**
- 현실적 `requests`로 **다운사이징** (스케줄 기준은 `requests`)  
- **불필요 Pod 축소** / **노드 증설** / **Cluster Autoscaler** 활성화  
- 미사용 노드 리소스 회수, 리밸런싱

> #### 스케줄 가능성 간단 판별(개념)
> Pod들의 총 요청 CPU를 \(\sum r_i\), 노드별 할당 가능 CPU를 \(A_j\)라 할 때,
> $$
> \sum r_i \le \sum A_j
> $$
> 를 만족해도 **배치 제약(라벨/존/taint)** 때문에 실제로는 실패할 수 있습니다. 총합이 된다고 끝이 아님!

**Ephemeral Storage 부족**  
Event: `Insufficient ephemeral-storage`  
→ 컨테이너 로그/캐시가 ephemeral-storage를 잠식. `resources.requests/limits.ephemeral-storage` 지정, 로그 로테이션/사이드카 정책 정비.

---

### 2.2 PVC 미바인딩(볼륨 대기)

**증상(Event 예시)**  
```
pod has unbound immediate PersistentVolumeClaims
no persistent volumes available for this claim
```

**문제 재현 YAML**
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
  storageClassName: fast-gp3    # 클러스터에 없는 SC라고 가정
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
- `kubectl get storageclass`로 **SC 존재 여부** 확인/수정  
- 동적 프로비저닝 미지원 환경이면 **PV를 수동 생성**  
- ReadWriteMany 필요 시 NFS/CSI RWX 가능 스토리지로 전환  
- **Topology(Zone) 불일치** 시: PVC/Pod/노드 **존 일치** 확인(특히 클라우드 블록스토리지)

---

### 2.3 Node Taint → Toleration 누락

**증상(Event 예시)**  
```
node(s) had taint {key: dedicated, value: gpu, effect: NoSchedule}, that the pod didn't tolerate
```

**진단**
```bash
kubectl describe node <node> | grep -i taint -A2
```

**수정 예시(Pod tolerations 추가)**
```yaml
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule
```

또는 관리 측에서 유지보수용 taint 해제:
```bash
kubectl taint node <node> dedicated=gpu:NoSchedule-
```

---

### 2.4 NodeSelector/NodeAffinity 조건 불만족

**증상(Event 예시)**  
```
0/5 nodes are available: 5 node(s) didn't match node selector.
```

**문제 예시**
```yaml
spec:
  nodeSelector:
    disktype: ssd   # 해당 라벨을 가진 노드가 없음
```

**해결**
- 노드 라벨링:
  ```bash
  kubectl label node <node> disktype=ssd
  ```
- 조건 완화: `preferredDuringSchedulingIgnoredDuringExecution`로 시작해 점진 강화

---

### 2.5 Pod(안티)어피니티/Topology Spread 제약

**증상(Event 예시)**  
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
          matchLabels: {app: frontend}
        topologyKey: "kubernetes.io/hostname"
```
→ 모든 노드에 이미 `app=frontend`가 깔려 있으면 **완전 금지**가 되어 Pending.

**해결**
- `required` → `preferred`로 완화하거나 topology 범위를 `zone`으로 확장
- **topologySpreadConstraints** 조합 시, 분산 정책과 충돌하지 않도록 **maxSkew**/selector 재점검

---

### 2.6 ResourceQuota/LimitRange 초과

**증상(Event 예시)**  
```
exceeded quota: compute-resources, requested: requests.cpu=1, used: requests.cpu=4, limited: 4
```

**진단**
```bash
kubectl describe quota -n <ns>
kubectl get limitrange -n <ns>
```

**해결**
- 불필요 리소스 제거 또는 Quota 상향  
- LimitRange에 의해 **자동 기본값**이 과도해지는 경우 조정

**LimitRange 예시(기본값 주입)**
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

### 2.7 이미지 풀 문제(ImagePullBackOff 전 단계)

**증상**
- `Pending` 또는 `ContainerCreating`에 머물다 `ImagePullBackOff`로 전환

**진단**
```bash
kubectl describe pod <pod> | sed -n '/Events/,$p'
# Failed to pull image / back-off pulling image ...
```

**해결**
- **이미지 경로/태그** 확인  
- 프라이빗 레지스트리면 **imagePullSecrets** 설정
```yaml
spec:
  imagePullSecrets:
  - name: regcred
```

---

### 2.8 GPU 요청/드라이버/RuntimeClass 불일치

**증상**
```
0/4 nodes are available: 3 node(s) had no available GPU, 1 node(s) didn't match node selector.
```

**문제 예시**
```yaml
resources:
  requests:
    nvidia.com/gpu: "1"
```

**해결**
- GPU 노드에 **NVIDIA Device Plugin** 배포 확인  
- GPU 수량/파티셔닝(MIG) 정책 부합 확인  
- **RuntimeClass** 필요한 경우 지정:
```yaml
spec:
  runtimeClassName: nvidia
```

---

### 2.9 PSA/OPA/Gatekeeper/Kyverno 등 정책 위반

**증상**
- `Forbidden` 또는 Admission webhook 거부  
- `describe pod` 이벤트에 정책 명시됨

**해결**
- 네임스페이스 라벨(PSA `baseline/restricted`)과 Pod의 `securityContext` 합치  
- Gatekeeper Constraint/Template 로그 확인, 룰 예외 조건 설정(만료일 포함 권장)

---

### 2.10 Cluster Autoscaler와의 상호작용

**특징**
- 스케줄 불가 상태가 지속되면 **CA가 새 노드 증설**  
- 단, **노드 풀 한도**/태인트/존/인스턴스 타입 제약으로 못 늘릴 수도

**점검**
- CA 로그/이벤트: **왜 증설하지 못했는지** 사유가 나옵니다(인스턴스 타입 불가, 제한 초과 등)

---

## 3) 자동 진단 스니펫 모음

### 3.1 Pending 원인 퀵리포트(쉘)

```bash
#!/usr/bin/env bash
NS=${1:-default}
echo "== Pending pods in ns:${NS} =="
kubectl get pods -n "$NS" --field-selector=status.phase=Pending -o name | while read -r P; do
  echo "---- $P ----"
  kubectl describe -n "$NS" "$P" | awk '/Events:/,0'
  echo
done
```

### 3.2 PVC 상태 일괄 점검

```bash
kubectl get pvc -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,STATUS:.status.phase,SC:.spec.storageClassName,SIZE:.spec.resources.requests.storage
```

### 3.3 노드 가용 리소스 요약

```bash
kubectl get nodes -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.cpu}{"\t"}{.status.allocatable.memory}{"\n"}{end}'
```

---

## 4) “원인→수정” 시나리오 7선

### 시나리오 A: 과도한 requests로 스케줄 실패
- **원인**: `requests.cpu=4`인데 2코어 노드만 존재
- **수정**: `requests.cpu=500m`로 조정 또는 4코어 노드풀 추가

### 시나리오 B: PVC Pending
- **원인**: StorageClass 오타
- **수정**: 올바른 SC로 변경, 필요 시 PV 수동 생성

### 시나리오 C: Taint로 격리된 노드
- **원인**: `NoSchedule` Taint, Toleration 누락
- **수정**: Pod에 `tolerations` 추가 또는 Taint 제거

### 시나리오 D: Anti-Affinity가 너무 강함
- **원인**: 모든 노드에 대상 라벨 Pod 배치 → 완전 금지
- **수정**: `required`→`preferred`, topology 범위 조정

### 시나리오 E: Quota 초과
- **원인**: `requests.memory` Oversubscription
- **수정**: 오래된 워크로드 제거, Quota 상향, requests 조정

### 시나리오 F: Ephemeral Storage 부족
- **원인**: 로그 폭증 / 큰 임시파일
- **수정**: `ephemeral-storage` requests/limits 지정, 로그 로테이션, 사이드카로 분리

### 시나리오 G: GPU 요청
- **원인**: device plugin 미배포
- **수정**: NVIDIA plugin 설치, MIG/타입 일치 확인, `runtimeClassName` 세팅

---

## 5) 재현 가능한 데모: 4가지 실패와 해결

> 아래는 하나씩 켰다 껐다 해보며 Pending→해결까지 체험하는 **학습용 세트**입니다.

### 5.1 PVC 미바인딩 데모

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: demo-pvc}
spec:
  accessModes: ["ReadWriteOnce"]
  resources: {requests: {storage: 1Gi}}
  storageClassName: doesnt-exist
---
apiVersion: v1
kind: Pod
metadata: {name: demo-pvc-pod}
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim: {claimName: demo-pvc}
```

**해결**: 존재하는 SC명으로 교체(예: `gp2`/`standard`), 동적 프로비저닝 확인.

---

### 5.2 Taint 미허용 데모

1) 노드에 Taint 추가:
```bash
kubectl taint node <node> dedicated=qa:NoSchedule
```

2) 일반 Pod 배포(대기):
```yaml
apiVersion: v1
kind: Pod
metadata: {name: demo-taint}
spec:
  containers:
  - name: c
    image: busybox
    command: ["sh","-c","sleep 3600"]
```

**해결**: tolerations 추가:
```yaml
spec:
  tolerations:
  - key: dedicated
    value: qa
    operator: Equal
    effect: NoSchedule
```

---

### 5.3 NodeSelector 불일치 데모

```yaml
apiVersion: v1
kind: Pod
metadata: {name: demo-ns}
spec:
  nodeSelector: {env: prod}
  containers:
  - name: c
    image: busybox
    command: ["sh","-c","sleep 3600"]
```

**해결**: 실제 노드 라벨 확인 후 일치시키기:
```bash
kubectl label node <node> env=prod
```

---

### 5.4 Quota 초과 데모

**Quota**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata: {name: rq, namespace: demo}
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    pods: "2"
```

**과도 요청 Pod**
```yaml
apiVersion: v1
kind: Pod
metadata: {name: demo-quota, namespace: demo}
spec:
  containers:
  - name: c
    image: nginx
    resources:
      requests:
        cpu: "2"
        memory: 2Gi
```

**해결**: 요청 축소 또는 Quota 상향.

---

## 6) 운영 자동화: “Pending 알람 → 원인 요약 → 제안”

프로메테우스 + 알러팅(예: Alertmanager)으로 **`kube_pod_status_phase{phase="Pending"}`** 비율 임계 초과 시 알림 →  
아래 스크립트로 **이벤트 요약** 후 Slack에 첨부.

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
# curl -X POST <SLACK_WEBHOOK> -H 'Content-type: text/plain' --data-binary @"$OUT"
```

---

## 7) 체크리스트(현장용)

- [ ] `describe pod` Events에서 **문장형 원인** 확인  
- [ ] PVC/PV/SC/CSI 상태 점검 (Zone/AccessModes 일치)  
- [ ] 노드 `Taints`와 Pod `Tolerations` 대응  
- [ ] `nodeSelector/affinity`와 실제 노드 라벨 일치  
- [ ] `topologySpreadConstraints` / Pod(안티)어피니티 충돌 여부  
- [ ] `ResourceQuota`/`LimitRange` 한도 확인  
- [ ] CPU/Memory/**ephemeral-storage** requests 합리적 설정  
- [ ] 이미지 풀 권한/경로/태그 확인 (`imagePullSecrets`)  
- [ ] GPU/RuntimeClass/Device Plugin 배포 상태  
- [ ] Cluster Autoscaler 증설 불가 사유(한도, 템플릿) 확인  
- [ ] PSA/OPA/Gatekeeper/Kyverno 등 정책 거부 로그 확인

---

## 8) 부록: 예제 Manifest(수정 전/후)

### 8.1 수정 전(문제)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: shop}
spec:
  replicas: 3
  selector: {matchLabels: {app: shop}}
  template:
    metadata: {labels: {app: shop}}
    spec:
      nodeSelector: {disktype: ssd}
      containers:
      - name: api
        image: private.repo.local/shop:latest
        ports: [{containerPort: 8080}]
        resources:
          requests: {cpu: "4", memory: "8Gi"}
          limits:   {cpu: "4", memory: "8Gi"}
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim: {claimName: shop-pvc}
```

**문제점**
- `disktype=ssd` 라벨 노드 부재
- 과도한 requests
- 프라이빗 이미지 pull 시크릿 없음
- PVC 준비 상태 미검증

### 8.2 수정 후(해결)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: shop}
spec:
  replicas: 3
  selector: {matchLabels: {app: shop}}
  template:
    metadata: {labels: {app: shop}}
    spec:
      imagePullSecrets:
      - name: regcred
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
      containers:
      - name: api
        image: private.repo.local/shop:1.4.7
        ports: [{containerPort: 8080}]
        resources:
          requests: {cpu: "500m", memory: "512Mi", ephemeral-storage: "1Gi"}
          limits:   {cpu: "2",    memory: "2Gi",   ephemeral-storage: "4Gi"}
        volumeMounts:
        - name: data
          mountPath: /data
        readinessProbe:
          httpGet: {path: /healthz/ready, port: 8080}
          periodSeconds: 5
      volumes:
      - name: data
        persistentVolumeClaim: {claimName: shop-pvc}
```

---

## 결론

- **Pending은 “스케줄 불가 또는 준비 미완료”의 신호**입니다.  
- **`describe pod`의 이벤트**로 **정확한 문장형 원인을 먼저 확인**하세요.  
- 리소스/볼륨/태인트/라벨/정책/토폴로지/Quota/이미지/GPU/Autoscaler 등 **각 계층별 체크포인트**를 체계적으로 점검하면 **대부분 수분 내 해결**이 가능합니다.  
- 재발 방지를 위해 **리소스 프로파일링, 기본값 정책(LimitRange), 분산/토폴로지 설계, 볼륨/스토리지 클래스 표준화, 정책 관측(OPA/PSA), Autoscaler 설정**을 지속적으로 개선하세요.

---

## 참고 링크

- Kubernetes Pod 스케줄링 개요  
- PersistentVolume / StorageClass  
- Taint & Toleration  
- Pod(안티)어피니티 & Topology Spread  
- ResourceQuota / LimitRange  
- Cluster Autoscaler / Device Plugin / PSA / OPA(Gatekeeper)