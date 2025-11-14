---
layout: post
title: Kubernetes - kubectl describe
date: 2025-05-05 20:20:23 +0900
category: Kubernetes
---
# Pod 상태 및 이벤트 모니터링: `kubectl describe` 완전 정리

애플리케이션을 배포했는데 다음과 같은 문제가 생길 때가 많다.

- Pod가 **Pending** 상태에서 멈춤
- **CrashLoopBackOff**로 반복 재시작
- **ImagePullBackOff** / **ErrImagePull**
- PVC 바인딩/마운트 실패
- Service로 연결이 안 됨

단순 로그(`kubectl logs`)만으로는 부족하다. 이때 가장 먼저 쓰는 도구가 **`kubectl describe`**다. 특정 리소스(특히 Pod)의 **현재 스냅샷 상태와 이벤트 타임라인**을 한눈에 보여 주기 때문이다.

---

## 기본 사용법과 출력 구조

### 기본 명령

```bash
kubectl describe pod <pod-name>
# 예

kubectl describe pod myapp-79c6dbcf7c-sntbz
```

### 출력 구조 훑어보기

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

### 핵심 필드 표

| 구역 | 확인 포인트 |
|---|---|
| `Status` | Pod 전반 상태(Pending/Running/Succeeded/Failed/Unknown) |
| `Node` | 스케줄된 노드. **스케줄 실패/노드 문제**를 유추 |
| `Containers[].State` | `Running/Waiting/Terminated`, `Reason`, `ExitCode`, `Signal` |
| `Restart Count` | 반복 재시작 감지(CrashLoopBackOff와 세트) |
| `Conditions` | `Ready/PodScheduled/ContainersReady/Initialized` 전이 시간 |
| `Volumes` | PVC 바인딩/마운트 대상 확인 |
| `Events` | 스케줄링→이미지 Pull→컨테이너 시작/종료→프로브 실패 등 **원인 타임라인** |

---

## 상황별 진단 절차(레시피)

아래 절차는 **describe → 보조 명령** 순으로 빠르게 원인을 좁힌다.

### Pod이 Pending 상태로 멈춤

```bash
kubectl describe pod myapp
```

**확인 항목**

- Events의 `FailedScheduling`:
  - `0/3 nodes are available: 3 Insufficient cpu.` → **리소스 부족**
  - `node(s) had taints ...` → **Taint/Toleration 미매칭**
  - `node(s) didn't match node selector` → **NodeSelector/NodeAffinity 불일치**
  - `persistentvolumeclaim "my-pvc" not found` → **PVC 부재**
  - `persistentvolumeclaim "my-pvc" is not bound` → **PVC 미바인딩**

**보조 명령**

```bash
# 스케줄링 이벤트를 시간순으로

kubectl get events --sort-by=.lastTimestamp

# 노드 리소스/테인트 점검

kubectl describe node <node-name>
kubectl get node <node-name> -o jsonpath='{.spec.taints}'

# PVC 상태

kubectl describe pvc my-pvc
kubectl get pvc my-pvc -o yaml
```

**해결 가이드**

- **리소스 부족**: Deployment의 requests/limits 조정, 노드 증설, HPA/스케일아웃.
- **Taints**: 워크로드에 tolerations 추가.
- **Affinity/Selector**: 라벨과 선택자 일치 여부 재검토.
- **PVC**: StorageClass/AccessMode/용량 일치 확인, PV 존재/동적 프로비저닝 점검.

---

### CrashLoopBackOff / Error / OOMKilled

```bash
kubectl describe pod <pod>
```

**확인 항목**

- `Containers[].State: Waiting (Reason: CrashLoopBackOff)`
- `Last State: Terminated (ExitCode: <code>, Reason: OOMKilled | Error | Signal)`
- Events: `Back-off restarting failed container`

**보조 명령**

```bash
# 이전 컨테이너 인스턴스의 로그(크래시 직전)

kubectl logs <pod> -c <container> --previous
# 현재 인스턴스

kubectl logs <pod> -c <container>
```

**원인별 팁**

- **ExitCode≠0**: 시작 스크립트/엔트리포인트/환경변수/DB 연결 오류.
- **OOMKilled**: 메모리 제한 상향, GC 튜닝, 캐시 축소, 리퀘스트/리밋 균형 조정.
- **Signal (e.g., SIGSEGV/SIGKILL)**: 네이티브 크래시/커널 OOM 시그널, cgroup 제한.

**참고: 백오프 계산(개념)**
Backoff 지수 증가를 단순화하면,

$$
\text{next\_delay} = \min(\text{base} \times 2^{\text{retries}}, \text{cap})
$$

일정 횟수 후 cap(예: 5분)까지 증가한다. 크래시 반복 시 **초기 지연을 기다려야** 다시 시작 시도가 된다.

---

### ImagePullBackOff / ErrImagePull

```bash
kubectl describe pod <pod>
```

**Events 예시**

- `Failed to pull image "myorg/myapp:v1": image not found` → 태그/리포지토리 오타
- `pull access denied` → 프라이빗 레지스트리 인증 실패(ImagePullSecret 필요)
- `x509: certificate signed by unknown authority` → 사설 CA/인증서 체인 문제

**해결 가이드**

- 이미지 주소/태그 정확도 재확인.
- Secret 생성 및 ServiceAccount 연결:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=<REGISTRY> --docker-username=<USER> \
  --docker-password=<PASS> --docker-email=<EMAIL>

# ServiceAccount에 imagePullSecrets 연결 후 Pod/Deployment에 사용

```

- 프록시/사설 CA 환경은 노드 레벨 신뢰체계/CRI 설정을 점검.

---

### Readiness/Liveness/Startup Probe 실패

```bash
kubectl describe pod <pod>
```

**Events 예시**

- `Readiness probe failed: HTTP probe failed with statuscode: 500`
- `Liveness probe failed: dial tcp ... i/o timeout`
- `Startup probe failed` (초기화 오래 걸리는 앱)

**보조 명령**

```bash
kubectl port-forward pod/<pod> 8080:8080
curl -i http://localhost:8080/ready
curl -i http://localhost:8080/health
```

**해결 가이드**

- `initialDelaySeconds`, `timeoutSeconds`, `periodSeconds`, `failureThreshold` 재조정.
- 서버 기동 시간 길면 **Startup Probe** 추가로 Liveness 지연.
- 프로브 경로가 인증/리디렉트/캐시 없이 200 OK를 반환하도록 구현.
- TCP/exec 프로브 선택을 상황에 맞게 적용.

---

### PVC 바인딩/마운트 실패

```bash
kubectl describe pod <pod>
kubectl describe pvc <pvc>
kubectl describe pv  <pv>   # 바인딩된 PV가 있다면
```

**자주 보는 메세지**

- `persistentvolumeclaim "my-data" not found`
- `persistentvolumeclaim "my-data" is not bound`
- `MountVolume.SetUp failed for volume "my-pvc": rpc error: ...`
- `Multi-Attach error for volume` (RWO를 여러 노드에서 동시에 마운트 시도)

**체크리스트**

- PVC의 `storageClassName` / `accessModes` / `requests.storage` 가 StorageClass/PV와 매칭되는가.
- ReadWriteOnce(RWO) 볼륨을 여러 Pod가 서로 다른 노드에서 쓰지 않는가.
- CSI 드라이버 로그(node-plugin/controller-plugin) 확인.

---

### DNS/Service 연결 불가

```bash
kubectl describe pod <client-pod>
kubectl describe svc <backend-svc>
```

**확인 항목**

- Service `selector` 라벨이 Pod 라벨과 일치하는가.
- `kubectl get endpoints <svc>` 에 엔드포인트가 채워지는가.
- Pod `readiness`가 false면 엔드포인트에서 제외된다.

**보조 명령**

```bash
kubectl exec -it <client-pod> -- sh -c 'nslookup my-svc.default.svc.cluster.local'
kubectl get ep my-svc -o wide
```

---

## describe를 더 잘 쓰는 실전 패턴

### 네임스페이스/라벨로 대상을 좁혀 가장 최신 Pod 확인

```bash
# 가장 최근 생성된 Pod 1개 describe

kubectl get pods -n prod -l app=myapp --sort-by=.metadata.creationTimestamp \
  | tail -n 1 \
  | awk '{print $1}' \
  | xargs -I{} kubectl describe pod {} -n prod
```

### Events만 빠르게 추출

```bash
kubectl describe pod <pod> | sed -n '/^Events:/,$p'
# 또는

kubectl get events --sort-by=.lastTimestamp | tail -n 50
```

### JSONPath / wide 보기

```bash
# 컨테이너 상태와 재시작 횟수

kubectl get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}{" "}{.state}{" restarts="}{.restartCount}{"\n"}{end}'

# Pod → Node 매핑

kubectl get pod -o wide
```

### RBAC로 describe 권한 열기(읽기 전용)

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
- apiGroups: [""]
  resources: ["events"]
  verbs: ["watch","list"]
```

---

## 문제 유형별 “원인 → 증상 → 확인 → 해결” 표

| 유형 | 대표 증상(`describe`/Events) | 확인 포인트 | 해결 |
|---|---|---|---|
| 스케줄 실패 | `FailedScheduling` | 노드 리소스, 테인트, 어피니티 | 리소스/라벨/어피니티 조정, 노드 증설 |
| 이미지 풀 실패 | `ErrImagePull`, `ImagePullBackOff` | 이미지 주소/태그/인증 | ImagePullSecret, 주소 교정, CA/프록시 |
| 크래시 루프 | `CrashLoopBackOff` / `Back-off restarting` | `ExitCode`, 이전 로그 | 코드/ENV/포트 충돌 수정, 메모리/리밋 조정 |
| 프로브 실패 | `Readiness/Liveness/Startup probe failed` | 엔드포인트 응답/지연 | 프로브 파라미터 수정, Startup Probe 도입 |
| PVC 바인딩 | `is not bound` | PVC/PV/SC 매칭 | StorageClass/용량/AccessMode 정합 |
| 볼륨 마운트 | `MountVolume.SetUp failed` | CSI 로그/권한 | 드라이버/권한/경로 재확인 |
| 엔드포인트 없음 | Service 엔드포인트 비어 있음 | selector/라벨/Ready | selector 정합, Readiness 성공 유도 |
| OOM | `Terminated: OOMKilled` | 메모리 사용량/리밋 | 리밋 상향, 메모리 최적화 |

---

## `kubectl describe`와 함께 쓰는 필수 명령 세트

```bash
# 리소스 스냅샷

kubectl get pods -o wide
kubectl top pods
kubectl top nodes

# 이벤트 타임라인

kubectl get events --sort-by=.lastTimestamp

# 엔드포인트/서비스

kubectl get svc <name> -o yaml
kubectl get ep  <name> -o wide

# 스케줄러 관점에서 보기(스케줄 실패시)

kubectl get pod <pod> -o yaml | sed -n '/schedulerName/p;/tolerations:/,/^$/{p}'

# 노드 상태/테인트

kubectl describe node <node>
kubectl get node <node> -o jsonpath='{.spec.taints}'

# 스토리지

kubectl get pvc,pv -o wide
kubectl describe pvc <name>
```

---

## 재현용 샘플 — 일부러 실패나 경보를 만들어 보자

### 이미지 오타로 ImagePullBackOff 유발

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

### Readiness 실패 유발(잘못된 경로)

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
      httpGet: { path: /readyz, port: 80 }   # nginx는 404
      initialDelaySeconds: 3
      periodSeconds: 5
```

```bash
kubectl apply -f bad-ready.yaml
kubectl describe pod bad-ready
kubectl get endpoints # 서비스에 연결 안 됨 확인에 활용
```

### PVC 미바인딩 유발(없는 StorageClass)

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

## 추가 고급 주제

### Init Containers와 스타트업 경로

`Init Containers`가 실패하면 애플리케이션 컨테이너는 시작되지 않는다. `describe`에서 **Init 컨테이너의 State/ExitCode**를 확인한다.

### SecurityContext/권한 문제

`permission denied`, `read-only file system` 류는 보통 다음을 확인:

- `securityContext.runAsUser`, `runAsGroup`, `fsGroup`
- 마운트 경로 권한/SELinux/AppArmor
- rootless 이미지 사용 여부

### 네트워크 정책

`NetworkPolicy`로 인해 통신이 차단될 수 있다. `describe`만으로는 직접 보이지 않으므로 네임스페이스의 네트폴을 점검:

```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy <np> -n <ns>
```

---

## 운영 자동화 스니펫 모음

### 최근 10분간 Warning 이벤트

```bash
since="$(date -u -d '-10 min' +%Y-%m-%dT%H:%M:%SZ)"
kubectl get events --all-namespaces --field-selector type=Warning \
  --sort-by=.lastTimestamp \
  | tail -n 100
```

### 특정 네임스페이스에서 CrashLoopBackOff Pod 나열

```bash
kubectl get pods -n prod \
  --field-selector=status.phase!=Succeeded \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{range .status.containerStatuses[*]}{.state.waiting.reason}{" "}{end}{"\n"}{end}' \
| awk '/CrashLoopBackOff/'
```

### describe + logs 원샷(가장 최근 Pod)

```bash
ns=default; app=myapp
pod=$(kubectl get pod -n "$ns" -l app="$app" --sort-by=.metadata.creationTimestamp | tail -n1 | awk '{print $1}')
kubectl describe pod "$pod" -n "$ns"
kubectl logs "$pod" -n "$ns" --all-containers --tail=200
```

---

## 요약 결론

- `kubectl describe`는 **현재 상태와 이벤트 타임라인을 한 화면**에 제공한다.
- **Pending**은 스케줄링/리소스/PVC/어피니티/테인트를 먼저 의심한다.
- **CrashLoopBackOff**는 `ExitCode/Signal`과 `--previous` 로그로 원인을 특정한다.
- **ImagePullBackOff**는 이미지 주소/인증/CA/프록시를 확인한다.
- **Readiness/Liveness/Startup** 실패는 프로브 파라미터/엔드포인트/기동 시간을 조정한다.
- **PVC/PV** 이슈는 StorageClass/AccessMode/용량/CSI 드라이버를 점검한다.
- describe와 함께 `get events / get endpoints / describe node / logs`를 병행하여 **빠르게 가설을 세우고 검증**하라.

이 문서의 절차/스니펫을 그대로 적용하면, 대부분의 초기 장애는 **분 단위로 원인**을 좁힐 수 있다.
