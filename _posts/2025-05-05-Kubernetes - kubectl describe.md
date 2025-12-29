---
layout: post
title: Kubernetes - kubectl describe
date: 2025-05-05 20:20:23 +0900
category: Kubernetes
---
# Pod 상태 및 이벤트 모니터링: `kubectl describe` 완전 정리

애플리케이션을 배포했을 때 자주 마주치는 문제들이 있습니다.

- Pod가 **Pending** 상태에서 멈춤
- **CrashLoopBackOff**로 반복 재시작
- **ImagePullBackOff** / **ErrImagePull**
- PVC 바인딩/마운트 실패
- Service로 연결이 안 됨

단순 로그(`kubectl logs`)만으로는 원인 파악에 한계가 있습니다. 이때 가장 먼저 사용하는 도구가 **`kubectl describe`**입니다. 특정 리소스(특히 Pod)의 **현재 상태 스냅샷과 이벤트 타임라인**을 한눈에 보여주기 때문입니다.

---

## 기본 사용법과 출력 구조

### 기본 명령

```bash
kubectl describe pod <pod-name>
# 예시
kubectl describe pod myapp-79c6dbcf7c-sntbz
```

### 출력 구조 이해하기

```
Name:           myapp-79c6dbcf7c-sntbz
Namespace:      default
Node:           node-1/192.168.1.10
Start Time:     Thu, 25 Jul 2025 12:00:00 +0900
Labels:         app=myapp
Annotations:    kubernetes.io/limit-ranger=LimitRanger plugin set: cpu request for container myapp
Status:         Running
IP:             10.1.2.5
IPs:            (intra-cluster IPs)
Controlled By:  ReplicaSet/myapp-79c6dbcf7c
QoS Class:      Burstable

Init Containers:
  init-db:
    Image: busybox:1.36
    State: Terminated (ExitCode:0)
    ...

Containers:
  myapp:
    Container ID:   containerd://123456...
    Image:          myorg/myapp:1.2.3
    Image ID:       docker-pullable://...
    Port:           8080/TCP
    State:          Running
    Ready:          True
    Restart Count:  0
    Environment:
      APP_MODE:     production
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access...
    ...

Conditions:
  Type              Status  LastProbeTime   LastTransitionTime
  Initialized       True    ...             ...
  Ready             True    ...             ...
  ContainersReady   True    ...             ...
  PodScheduled      True    ...             ...

Volumes:
  my-pvc:
    Type: PersistentVolumeClaim (a reference to a PVC)
    ClaimName: my-data
    ReadOnly: false
  kube-api-access-xxxxx: ...

Events:
  Type     Reason              Age   From               Message
  ----     ------              ----  ----               -------
  Normal   Scheduled           68s   default-scheduler  Successfully assigned default/myapp... to node-1
  Normal   Pulling             67s   kubelet            Pulling image "myorg/myapp:1.2.3"
  Normal   Pulled              60s   kubelet            Successfully pulled image
  Normal   Created             59s   kubelet            Created container myapp
  Normal   Started             59s   kubelet            Started container myapp
```

### 핵심 필드 요약

| 구역 | 확인 포인트 |
|---|---|
| `Status` | Pod의 전반 상태(Pending/Running/Succeeded/Failed/Unknown) |
| `Node` | 스케줄된 노드. **스케줄 실패나 노드 문제**를 유추하는 단서 |
| `Containers[].State` | 컨테이너 상태(`Running`/`Waiting`/`Terminated`)와 `Reason`, `ExitCode`, `Signal` |
| `Restart Count` | 컨테이너 재시작 횟수(CrashLoopBackOff와 함께 확인) |
| `Conditions` | `Ready`, `PodScheduled`, `ContainersReady`, `Initialized`의 전이 시간과 상태 |
| `Volumes` | PVC 바인딩/마운트 대상 확인 |
| `Events` | 스케줄링, 이미지 Pull, 컨테이너 시작/종료, 프로브 실패 등 **원인을 추적하는 타임라인** |

---

## 상황별 진단 절차

아래 절차는 `describe` 명령으로 현상을 파악한 후 보조 명령으로 원인을 구체화하는 방식으로 진행합니다.

### Pod이 Pending 상태로 멈춤

```bash
kubectl describe pod myapp
```

**주요 확인 사항**

- Events 항목에 `FailedScheduling`이 있는지 확인합니다.
  - `0/3 nodes are available: 3 Insufficient cpu.` → **리소스 부족**
  - `node(s) had taints ...` → **Taint/Toleration이 매칭되지 않음**
  - `node(s) didn't match node selector` → **NodeSelector/NodeAffinity 불일치**
  - `persistentvolumeclaim "my-pvc" not found` → **PVC가 존재하지 않음**
  - `persistentvolumeclaim "my-pvc" is not bound` → **PVC가 바인딩되지 않음**

**추가 진단 명령어**

```bash
# 시간 순으로 스케줄링 이벤트 확인
kubectl get events --sort-by=.lastTimestamp

# 노드 리소스와 테인트 점검
kubectl describe node <node-name>
kubectl get node <node-name> -o jsonpath='{.spec.taints}'

# PVC 상태 확인
kubectl describe pvc my-pvc
kubectl get pvc my-pvc -o yaml
```

**해결 방안**

- **리소스 부족**: Pod의 `requests`/`limits` 조정, 클러스터 노드 증설, HPA 설정, 스케일아웃.
- **Taints/Tolerations**: Pod의 `tolerations`에 해당 테인트를 추가.
- **Affinity/Selector**: Pod과 Node의 라벨 및 선택자 일치 여부 재확인.
- **PVC 문제**: `StorageClass`, `accessModes`, 용량이 PV와 일치하는지 확인, 동적 프로비저닝 상태 점검.

---

### CrashLoopBackOff / Error / OOMKilled

```bash
kubectl describe pod <pod>
```

**주요 확인 사항**

- `Containers[].State: Waiting (Reason: CrashLoopBackOff)` 확인
- `Last State: Terminated (ExitCode: <code>, Reason: OOMKilled | Error | Signal)` 확인
- Events에 `Back-off restarting failed container` 메시지 확인

**추가 진단 명령어**

```bash
# 이전 컨테이너 인스턴스의 로그(크래시 직전 상황)
kubectl logs <pod> -c <container> --previous

# 현재 실행 중인(또는 재시도 중인) 인스턴스 로그
kubectl logs <pod> -c <container>
```

**원인별 진단 팁**

- **ExitCode가 0이 아닌 경우**: 애플리케이션 시작 스크립트, 엔트리포인트, 환경변수, 외부 서비스(예: DB) 연결 오류를 의심.
- **OOMKilled**: 컨테이너 메모리 제한(`limit`) 상향, 애플리케이션 메모리 사용량(Garbage Collection 튜닝, 캐시 크기 조정) 최적화, `request`와 `limit`의 균형 재조정.
- **Signal (예: SIGSEGV, SIGKILL)**: 네이티브 라이브러리/코드 오류, 커널 레벨의 OOM killer에 의한 종료, cgroup 제한 초과를 확인.

**참고: 백오프 지연 계산**
Kubernetes는 컨테이너 재시작 실패 시 지수 백오프를 적용합니다. 간단히 표현하면 다음 공식과 같이 지연 시간이 증가하다가 일정 상한에 도달합니다.

```
next_delay = min(base_delay * 2^(retry_count), max_cap)
```

초기 지연(예: 10초)부터 시작해 최대 지연(예: 5분)까지 증가합니다. CrashLoopBackOff 상태에서는 이 지연 시간을 기다린 후 다시 시작을 시도합니다.

---

### ImagePullBackOff / ErrImagePull

```bash
kubectl describe pod <pod>
```

**Events 예시**

- `Failed to pull image "myorg/myapp:v1": image not found` → 태그 오타 또는 리포지토리 경로 오류
- `pull access denied` → 프라이빗 레지스트리 인증 실패(`ImagePullSecret` 필요)
- `x509: certificate signed by unknown authority` → 사설 CA 또는 인증서 체인 문제

**해결 방안**

- 이미지 주소와 태그의 철자를 정확히 확인.
- 프라이빗 레지스트리용 Secret 생성 및 Pod/ServiceAccount에 연결:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=<REGISTRY> \
  --docker-username=<USER> \
  --docker-password=<PASS> \
  --docker-email=<EMAIL>

# 생성된 Secret을 Pod의 spec.imagePullSecrets 또는 ServiceAccount에 추가
```

- 프록시 또는 사설 인증서 환경인 경우, 노드의 도커/컨테이너 런타임 신뢰 체계 설정을 점검.

---

### Readiness/Liveness/Startup Probe 실패

```bash
kubectl describe pod <pod>
```

**Events 예시**

- `Readiness probe failed: HTTP probe failed with statuscode: 500`
- `Liveness probe failed: dial tcp ... i/o timeout`
- `Startup probe failed` (초기화 시간이 긴 애플리케이션에서 발생)

**추가 진단 명령어**

```bash
# Pod의 애플리케이션 포트에 직접 접근하여 응답 확인
kubectl port-forward pod/<pod> 8080:8080
curl -i http://localhost:8080/ready
curl -i http://localhost:8080/health
```

**해결 방안**

- 프로브 설정(`initialDelaySeconds`, `timeoutSeconds`, `periodSeconds`, `failureThreshold`)을 애플리케이션 특성에 맞게 재조정.
- 서버 기동 시간이 매우 길다면 **Startup Probe**를 추가하여 Liveness Probe의 시작을 지연시킴.
- 프로브 경로(`path`)가 인증, 리디렉트, 캐시 없이 정상 상태(예: 200 OK)를 반환하도록 구현.
- HTTP 대신 `tcpSocket` 또는 `exec` 프로브를 상황에 맞게 선택.

---

### PVC 바인딩/마운트 실패

```bash
kubectl describe pod <pod>
kubectl describe pvc <pvc>
# 바인딩된 PV가 있다면
kubectl describe pv  <pv>
```

**자주 보이는 에러 메시지**

- `persistentvolumeclaim "my-data" not found`
- `persistentvolumeclaim "my-data" is not bound`
- `MountVolume.SetUp failed for volume "my-pvc": rpc error: ...`
- `Multi-Attach error for volume` (RWO 볼륨을 여러 노드의 Pod에서 동시에 마운트하려 할 때)

**진단 포인트**

- PVC의 `storageClassName`, `accessModes`, `requests.storage`가 클러스터의 StorageClass 또는 기존 PV와 정확히 매칭되는지 확인.
- `ReadWriteOnce(RWO)` 볼륨을 서로 다른 노드에 스케줄된 여러 Pod가 동시에 사용하려 하지 않는지 확인.
- CSI 드라이버의 로그(`node-plugin`, `controller-plugin`)를 확인하여 드라이버 자체 오류 여부 파악.

---

### DNS/Service 연결 불가

```bash
kubectl describe pod <client-pod>
kubectl describe svc <backend-svc>
```

**주요 확인 사항**

- Service의 `selector` 라벨이 실제 백엔드 Pod의 라벨과 정확히 일치하는지 확인.
- `kubectl get endpoints <svc>` 명령으로 해당 Service의 엔드포인트 목록이 채워져 있는지 확인. (Pod의 `readiness`가 `false`면 엔드포인트에서 제외됨)

**추가 진단 명령어**

```bash
# Pod 내부에서 서비스 이름으로 DNS 조회 테스트
kubectl exec -it <client-pod> -- sh -c 'nslookup my-svc.default.svc.cluster.local'

# 서비스 엔드포인트 상세 확인
kubectl get ep my-svc -o wide
```

---

## 실전 활용 패턴

### 네임스페이스와 라벨로 특정 Pod를 빠르게 찾아 Describe

```bash
# 특정 라벨을 가진 Pod 중 가장 최근 생성된 1개를 Describe
kubectl get pods -n prod -l app=myapp --sort-by=.metadata.creationTimestamp \
  | tail -n 1 \
  | awk '{print $1}' \
  | xargs -I{} kubectl describe pod {} -n prod
```

### Events 섹션만 빠르게 확인

```bash
kubectl describe pod <pod> | sed -n '/^Events:/,$p'
# 또는 모든 네임스페이스의 최근 이벤트 확인
kubectl get events --all-namespaces --sort-by=.lastTimestamp | tail -n 50
```

### JSONPath를 이용한 특정 정보 추출

```bash
# 모든 컨테이너의 이름, 상태, 재시작 횟수 출력
kubectl get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}{" "}{.state}{" restarts="}{.restartCount}{"\n"}{end}'

# Pod와 그 Pod가 할당된 Node 매핑 확인 (wide 출력)
kubectl get pod -o wide
```

### RBAC을 통한 읽기 전용 Describe 권한 부여

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: readonly-describe
rules:
- apiGroups: [""]
  resources: ["pods","services","endpoints","events","namespaces","nodes","persistentvolumeclaims"]
  verbs: ["get","list","watch"]
- apiGroups: ["apps"]
  resources: ["deployments","replicasets","statefulsets","daemonsets"]
  verbs: ["get","list","watch"]
```

---

## 문제 유형별 진단 요약표

| 문제 유형 | 대표 증상(`describe`/Events) | 주요 확인 포인트 | 해결 방향 |
|---|---|---|---|
| 스케줄 실패 | `FailedScheduling` | 노드 리소스, 테인트, 어피니티/셀렉터 | 리소스/라벨/어피니티 조정, 노드 증설 |
| 이미지 풀 실패 | `ErrImagePull`, `ImagePullBackOff` | 이미지 주소, 태그, 레지스트리 인증 | ImagePullSecret 생성, 이미지 주소 정정, CA/프록시 설정 |
| 크래시 루프 | `CrashLoopBackOff`, `Back-off restarting` | `ExitCode`, `Signal`, `--previous` 로그 | 애플리케이션 코드/환경 오류 수정, 메모리 제한 조정 |
| 프로브 실패 | `Readiness/Liveness/Startup probe failed` | 프로브 엔드포인트 응답, 네트워크 지연 | 프로브 파라미터 수정, Startup Probe 도입 |
| PVC 바인딩 실패 | `is not bound` | PVC/PV/StorageClass 매칭 상태 | StorageClass, 용량, AccessMode 정합성 확인 |
| 볼륨 마운트 실패 | `MountVolume.SetUp failed` | CSI 드라이버 로그, 권한 | 드라이버 설치/권한/마운트 경로 점검 |
| 서비스 연결 불가 | Service 엔드포인트가 비어 있음 | Service Selector, Pod 라벨 및 Readiness 상태 | Selector 정합성 확인, Pod가 Ready 상태가 되도록 유도 |
| OOM 종료 | `Terminated: OOMKilled` | 컨테이너 메모리 사용량 및 `limits` | 메모리 `limit` 상향, 애플리케이션 메모리 최적화 |

---

## `kubectl describe`와 함께 쓰는 보조 명령어 세트

```bash
# 클러스터 리소스 현황 스냅샷
kubectl get pods -o wide
kubectl top pods
kubectl top nodes

# 이벤트 타임라인 정렬 확인
kubectl get events --all-namespaces --sort-by=.lastTimestamp

# 서비스 및 엔드포인트 상세 정보
kubectl get svc <name> -o yaml
kubectl get ep  <name> -o wide

# 노드 상태 및 테인트 정보
kubectl describe node <node>
kubectl get node <node> -o jsonpath='{.spec.taints}'

# 스토리지 리소스 상태
kubectl get pvc,pv -o wide
kubectl describe pvc <name>
```

---

## 재현용 샘플 매니페스트

### 이미지 태그 오류로 인한 ImagePullBackOff 유발

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-image
spec:
  replicas: 1
  selector:
    matchLabels: { app: bad-image }
  template:
    metadata:
      labels: { app: bad-image }
    spec:
      containers:
        - name: bad
          image: nginx:nonexistent-tag
          ports: [{containerPort: 80}]
```

```bash
kubectl apply -f bad-image.yaml
kubectl describe pod -l app=bad-image
```

### 잘못된 경로로 인한 Readiness Probe 실패 유발

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-ready
spec:
  containers:
  - name: web
    image: nginx:1.25
    ports: [{containerPort: 80}]
    readinessProbe:
      httpGet: { path: /readyz, port: 80 }   # Nginx 기본 설정에는 없는 경로
      initialDelaySeconds: 3
      periodSeconds: 5
```

```bash
kubectl apply -f bad-ready.yaml
kubectl describe pod bad-ready
# 엔드포인트가 비어있는지 확인
kubectl get endpoints
```

### 존재하지 않는 StorageClass를 지정한 PVC 바인딩 실패 유발

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-missing-sc
spec:
  storageClassName: sc-not-exist
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f pvc-missing-sc.yaml
kubectl describe pvc pvc-missing-sc
```

---

## 고급 주제

### Init Containers와 애플리케이션 시작

`Init Containers`가 실패하면 메인 애플리케이션 컨테이너는 시작되지 않습니다. `describe` 출력의 **Init Containers** 섹션에서 State와 ExitCode를 확인해야 합니다.

### SecurityContext 및 권한 문제

`permission denied`, `read-only file system` 오류는 다음과 같은 점을 확인하세요.

- Pod/Container의 `securityContext` (`runAsUser`, `runAsGroup`, `fsGroup`)
- 볼륨 마운트 경로의 파일 시스템 권한
- SELinux 또는 AppArmor 프로필
- rootless 컨테이너 이미지 사용 여부

### NetworkPolicy 영향

`NetworkPolicy`로 인해 Pod 간 통신이 차단될 수 있습니다. `describe`로는 직접 확인이 어려우므로 관련 네트워크 정책을 점검하세요.

```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy <np-name> -n <namespace>
```

---

## 운영 자동화 스니펫

### 최근 10분간 발생한 Warning 이벤트 모니터링

```bash
since="$(date -u -d '-10 min' +%Y-%m-%dT%H:%M:%SZ)"
kubectl get events --all-namespaces --field-selector type=Warning \
  --sort-by=.lastTimestamp \
  | tail -n 100
```

### 특정 네임스페이스에서 CrashLoopBackOff 상태인 Pod 찾기

```bash
kubectl get pods -n prod \
  --field-selector=status.phase!=Succeeded \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{range .status.containerStatuses[*]}{.state.waiting.reason}{" "}{end}{"\n"}{end}' \
| awk '/CrashLoopBackOff/'
```

### Describe와 Logs를 한 번에 확인 (가장 최근 Pod 대상)

```bash
ns=default; app=myapp
pod=$(kubectl get pod -n "$ns" -l app="$app" --sort-by=.metadata.creationTimestamp | tail -n1 | awk '{print $1}')
kubectl describe pod "$pod" -n "$ns"
kubectl logs "$pod" -n "$ns" --all-containers --tail=200
```

---

## 결론

`kubectl describe`는 Kubernetes 리소스, 특히 Pod의 상태를 진단하는 데 있어 가장 강력하고 근본적인 도구입니다. 단순한 로그 확인을 넘어 **리소스의 현재 상태 스냅샷과 시간 순으로 정리된 이벤트 히스토리를 한눈에 제공**함으로써 문제의 근본 원인을 체계적으로 추적할 수 있게 합니다.

주요 장애 시나리오별로 `describe`의 출력에서 집중해야 할 포인트를 숙지하고, 필요에 따라 `kubectl logs`, `kubectl get events`, `kubectl get endpoints` 등의 보조 명령어와 연계하여 사용한다면, 복잡해 보이는 클러스터 장애도 신속하게 해결의 실마리를 찾을 수 있을 것입니다. 이 문서의 절차와 패턴을 참고하여 체계적인 문제 해결 습관을 기르시기 바랍니다.