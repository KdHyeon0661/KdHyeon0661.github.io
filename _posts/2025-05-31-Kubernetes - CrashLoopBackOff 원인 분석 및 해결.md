---
layout: post
title: Kubernetes - CrashLoopBackOff 원인 분석 및 해결
date: 2025-05-31 22:20:23 +0900
category: Kubernetes
---
# CrashLoopBackOff 원인 분석 및 해결 방법

## 1) CrashLoopBackOff란?

- **Crash**: 컨테이너 프로세스(파드의 각 컨테이너 엔트리포인트)가 **비정상 종료(0이 아닌 코드)**.
- **Loop**: kubelet이 재시작 정책에 따라 **반복 재기동**.
- **BackOff**: 재시도 간격이 **지수적 증가**.

### 백오프 간격(개념식)

재시도 회수 \(k\)에 대한 대기 시간 \(t_k\) (상한 \(t_{\max}\)):

$$
t_k = \min\big(t_{\max},\; t_0 \cdot 2^{k}\big)
$$

일반적으로 수 초에서 시작해 수십 초~수분으로 증가합니다.

### 상태 예시

```bash
kubectl get pod myapp-abc123
# NAME           READY   STATUS             RESTARTS   AGE
# myapp-abc123   0/1     CrashLoopBackOff   7          3m42s
```

---

## 2) 첫 60초 진단 루틴(현장에서 바로 쓰는 순서)

1. **상태 요약**
   ```bash
   kubectl get pod <POD> -o wide
   kubectl describe pod <POD>    # 이벤트/종료코드/Probe 결과/리소스 요약
   ```
2. **직전/이전 로그 모두**
   ```bash
   kubectl logs <POD> -c <CONTAINER> --timestamps
   kubectl logs <POD> -c <CONTAINER> --previous --timestamps
   ```
3. **상태 스냅샷(JSONPath)**
   ```bash
   kubectl get pod <POD> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}'
   kubectl get pod <POD> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.exitCode}{"\n"}'
   ```
4. **클러스터 이벤트**
   ```bash
   kubectl get events --sort-by=.lastTimestamp -A | tail -n 50
   ```
5. **들어가서 보고 싶다면(즉시 종료되어 exec 불가 시)**
   - `startupProbe` 일시 추가 or **ephemeralContainer**로 디버그 셸:
     ```bash
     kubectl debug pod/<POD> -c <CONTAINER> --image=busybox:1.36 --target=<CONTAINER> -it -- bash
     ```

> **팁**: `--previous` 로그는 직전 컨테이너 인스턴스가 남긴 마지막 메시지를 보여줍니다. CrashLoopBackOff에서는 **이전 로그**가 핵심 단서가 되는 경우가 많습니다.

---

## 3) 원인 카테고리 맵(20가지)

| 카테고리 | 대표 로그/증상 | 핵심 조치 |
|---|---|---|
| 1. 앱 런타임 예외 | NPE, uncaught exception | 코드/구성 수정, 의존성 주입 확인 |
| 2. 잘못된 `command/args` | `not found`, `exec format error` | `command/args`·이미지 엔트리포인트 점검 |
| 3. 환경변수/Secret/ConfigMap 누락 | `KeyError`, `file not found` | 참조 리소스 존재/키 이름 일치 확인 |
| 4. Config 파일 경로/권한 | `permission denied`, `no such file` | 마운트 경로/파일 모드/소유자 조정 |
| 5. 포트 바인딩 실패 | `address already in use` | 포트 변경/중복 프로세스 제거 |
| 6. DB/외부 의존 지연 | 부팅 즉시 연결 실패 | `startupProbe` 도입, 리트라이/백오프 |
| 7. Liveness 오탐 | 몇 초 후 kill | `initialDelaySeconds` 상향, 헬스엔드포인트 수정 |
| 8. Readiness 실패로 트래픽 없음 | “정상인데 Serving 되지 않음” | readiness 기준 조정(더 완화/정확) |
| 9. **OOMKilled** | `State: OOMKilled` | 메모리 `limits` 상향/메모리 사용 감소 |
| 10. CPU Throttling 과다 | 느린 시작 → probe 실패 | CPU `requests` 상향, `startupProbe` |
| 11. 파일시스템 read-only | `EROFS` | `readOnlyRootFilesystem` 확인, write 경로 분리 |
| 12. 권한상승/보안컨텍스트 | `Operation not permitted` | `capabilities`/`runAsUser`/PSA 준수 |
| 13. 이미지/엔트리포인트 빌드 문제 | `no main manifest attr` | Dockerfile CMD/ENTRYPOINT/패스 수정 |
| 14. InitContainer 실패 | init 종료코드≠0 | init 로그/마운트 요건 고치기 |
| 15. 사이드카 종속성 | 앱이 사이드카 준비 전 시작 | `startupProbe`/`initContainers`로 순서 보장 |
| 16. Node 리소스/디스크 | `no space left on device` | 로그 로테이션/emptyDir 사용 축소 |
| 17. 커널/런타임 이슈 | 컨테이너d 에러 | 노드 상태/런타임 로그 점검 |
| 18. 지역 설정/시간대 | TLS/서명/만료 오류 | time sync, CA bundle 확인 |
| 19. 경합/락 | `database is locked` | backoff/retry, 락 경합 설계 |
| 20. 잘못된 프로브 경로/포맷 | 404/500/timeout | 올바른 경로/포트/헤더 반영 |

---

## 4) 케이스별 **로그→원인→수정 YAML** 레시피

### 4.1 잘못된 엔트리포인트

**로그**
```
/bin/app: no such file or directory
```

**수정 전**
```yaml
containers:
- name: app
  image: my/app:1.0.0
  command: ["/bin/app"]
```

**수정 후(이미지 내 경로 일치/쉘 형태)**
```yaml
containers:
- name: app
  image: my/app:1.0.1
  command: ["/usr/local/bin/app"]
  # 또는
  # command: ["sh","-c","/usr/local/bin/app --config /etc/app/config.yaml"]
```

---

### 4.2 Secret 키 오타

**증상**: 부팅 즉시 `KeyError: PASSWORD`

**수정 전**
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: passwrod   # 오타!
```

**수정 후**
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

**검증**
```bash
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

---

### 4.3 ConfigMap 마운트 경로 상이

**로그**
```
open /etc/app/config.yaml: no such file or directory
```

**수정 전**
```yaml
volumeMounts:
- name: cfg
  mountPath: /etc/appconf
volumes:
- name: cfg
  configMap: { name: app-config }
```

**수정 후**
```yaml
volumeMounts:
- name: cfg
  mountPath: /etc/app
  readOnly: true
```
> 애플리케이션의 실제 기대 경로(`/etc/app/config.yaml`)에 맞춥니다.

---

### 4.4 LivenessProbe가 너무 공격적

**증상**: 부팅이 10초 걸리는데 `initialDelaySeconds: 5`로 kill 반복

**수정 전**
```yaml
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 1
```

**수정 후(StartupProbe 도입 권장)**
```yaml
startupProbe:
  httpGet: { path: /healthz, port: 8080 }
  failureThreshold: 30
  periodSeconds: 2

livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 0   # startupProbe가 끝나면 liveness 개시
  periodSeconds: 10
  timeoutSeconds: 2
```

---

### 4.5 OOMKilled

**확인**
```bash
kubectl describe pod <POD> | grep -A2 "State:"
# State: Terminated (Reason: OOMKilled)
```

**수정 전**
```yaml
resources:
  limits:
    memory: "256Mi"
  requests:
    memory: "128Mi"
```

**수정 후(여유 상향/메모리 릭 점검)**
```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

**운영 팁**: 메모리 릭이 의심되면 **heap dump** 경로를 **writable emptyDir**에 구성하고, **CPU/GC 옵션** 조정.

---

### 4.6 Read-only 루트 FS + 쓰기 경로 혼동

**로그**
```
open /app/logs/app.log: read-only file system
```

**수정(쓰기 경로를 EmptyDir/PVC로 분리)**
```yaml
securityContext:
  readOnlyRootFilesystem: true
volumeMounts:
- name: work
  mountPath: /app/logs
volumes:
- name: work
  emptyDir: {}
```

---

### 4.7 InitContainer 실패

**확인**
```bash
kubectl logs <POD> -c <INIT_NAME>
kubectl get pod <POD> -o jsonpath='{.status.initContainerStatuses[*].state.terminated}'
```

**수정(명령/권한/마운트 조정)**
```yaml
initContainers:
- name: migrate
  image: my/migrator:1.2.0
  command: ["sh","-c","./migrate up || exit 1"]
  envFrom:
  - secretRef: { name: db-secret }
```

---

### 4.8 사이드카 의존(예: Envoy/Jaeger) 준비 전 시작

**해결**: 앱 **StartupProbe**로 의존 채널 연결성까지 검증

```yaml
startupProbe:
  exec:
    command: ["sh","-c","nc -z localhost 15000 && curl -sf http://localhost:15000/server_info"]
  failureThreshold: 60
  periodSeconds: 2
```

---

### 4.9 권한(보안 컨텍스트) 문제

**로그**
```
operation not permitted
```

**수정**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```
> 필요 시 특정 capability만 **추가(ADD)** 하되 최소화.

---

### 4.10 포트 충돌

**로그**
```
bind: address already in use
```

**수정**: 포트 변경 또는 **Sidecar/Daemon** 충돌 제거. `containerPort`와 `Service` 포트를 일치시키고, **중복 리스너** 확인.

---

## 5) 관찰 아키텍처(무엇을 봐야 하나?)

- **컨테이너 로그**: `--previous` 포함
- **Probe 결과**: `describe`의 `Liveness/Readiness` 실패 카운트
- **컨테이너 상태**: `lastState.terminated.reason/exitCode/finishedAt`
- **노드/런타임**: `journalctl -u kubelet`, `containerd`/`dockerd` 로그
- **리소스 지표**: `kubectl top pod`, Prometheus의 `container_memory_working_set_bytes`, `container_cpu_usage_seconds_total`
- **이벤트**: `Warning BackOff`, `Killing`, `Unhealthy`, `OOMKilling`

---

## 6) 디버그 기법 모음

### 6.1 빠르게 “살려서” 들여다보기

```yaml
# 일시적 디버그 오버레이
command: ["sh","-c","sleep 3600"]
```

배포 후 `kubectl exec`로 내부 파일·경로·권한 확인 → 원인 파악 후 본래 명령 복구.

### 6.2 Ephemeral Container

```bash
kubectl debug pod/<POD> -c debug --image=busybox:1.36 -it --target=<CONTAINER>
```

> 크래시하는 컨테이너 네임스페이스에 들어가 **파일/프로세스/퍼미션** 점검.

### 6.3 터미네이션 메시지 활용

```yaml
terminationMessagePolicy: FallbackToLogsOnError
```

**마지막 로그**를 상태에 남겨 재현이 어려운 크래시를 기록.

---

## 7) 프로브 설계 모범사례

- **`startupProbe` 먼저**: 부팅이 긴 앱(언어 런타임/캐시 예열/마이그) → liveness 발동 지연
- **`livenessProbe`는 “자체 복구 불가능” 상태만**: 단순 의존성 장애는 **프로세스 kill보다 재시도**가 낫다.
- **`readinessProbe`는 “트래픽 수용 가능” 기준**: DB 연결 등 핵심 의존이 성립했을 때만 True.
- **시간 매개변수**: `initialDelaySeconds`, `timeoutSeconds`, `periodSeconds`, `failureThreshold`를 **실측 기반**으로 튜닝.

예시(실무 추천 값의 출발점)

```yaml
startupProbe:
  httpGet: { path: /healthz/startup, port: 8080 }
  periodSeconds: 2
  failureThreshold: 60  # 최대 120초 부팅 허용

livenessProbe:
  httpGet: { path: /healthz/live, port: 8080 }
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3

readinessProbe:
  httpGet: { path: /healthz/ready, port: 8080 }
  periodSeconds: 5
  timeoutSeconds: 1
  failureThreshold: 2
```

---

## 8) 리소스 튜닝(Throttle·OOM을 Crash로 오인하지 않기)

- **CPU**: 과도한 throttling은 **느린 시작 → liveness 실패**를 유발
  - 해결: `requests.cpu`를 **현실적**으로 상향, `limits.cpu`를 너무 낮추지 않기
- **Memory**: `limits.memory`를 초과하면 즉시 OOMKilled
  - 해결: 피크 측정 후 **여유 버퍼**(예: 95p + 20~30%)로 재설정

```yaml
resources:
  requests: { cpu: "300m", memory: "512Mi" }
  limits:   { cpu: "1",    memory: "1Gi"   }
```

---

## 9) 배포/롤백 연계(무한 Crash 차단)

- **Deployment 진행 중 Crash**는 **롤링**이 멈추거나 느려짐 → `kubectl rollout status`로 감시
- **빠른 철회**
  ```bash
  kubectl rollout undo deploy/<NAME>
  ```
- **Argo CD/Helm**: **게이트/자동 롤백** 설정(Argo Rollouts, 헬스 체크 기반)

---

## 10) 운영 체크리스트 (변경 전/후)

**변경 전**
- [ ] 프로브 경로를 **로컬에서 검증** (`port-forward` → `curl`)
- [ ] Secret/ConfigMap **키 이름** 정확성
- [ ] `command/args`와 이미지 ENTRYPOINT **일치**
- [ ] 리소스 요청/제한 **실측 기반**
- [ ] 로그 수준/경로/권한 확인

**변경 후**
- [ ] `rollout status` 정상
- [ ] `events`에 BackOff/Killing 없음
- [ ] 재시작 카운트 증가 없음
- [ ] 대시보드(메모리/CPU/프로브 실패) 안정

---

## 11) 재현 가능한 **미니 워크로드** (연습 용)

### 11.1 의도적 Crash(명령 오류)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: crash-sample }
spec:
  replicas: 1
  selector: { matchLabels: { app: crash } }
  template:
    metadata: { labels: { app: crash } }
    spec:
      containers:
      - name: c
        image: alpine:3.20
        command: ["/bin/not-exist"]
```

**관찰 포인트**
- `describe` → `Back-off restarting failed container`
- `logs --previous` → `exec` 에러

**수정**: `command: ["sh","-c","echo OK; sleep 3600"]`

---

### 11.2 Liveness 오탐 시나리오

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: live-tight }
spec:
  replicas: 1
  selector: { matchLabels: { app: live-tight } }
  template:
    metadata: { labels: { app: live-tight } }
    spec:
      containers:
      - name: app
        image: ghcr.io/nginxinc/nginx-unprivileged:stable
        livenessProbe:
          httpGet: { path: /healthz, port: 8080 } # nginx엔 없음 → 404
          initialDelaySeconds: 5
          periodSeconds: 5
```

**결과**: 계속 kill → CrashLoopBackOff

**해결**: 올바른 경로로 수정 또는 `startupProbe` 도입.

---

## 12) FAQ

**Q1. `ImagePullBackOff`와 다른가요?**  
A. 예. 이는 **이미지 풀 실패** 상태이며, 애초에 컨테이너가 시작되지 않습니다. CrashLoopBackOff는 **시작 후 크래시**.

**Q2. Job도 CrashLoopBackOff가 뜨나요?**  
A. Job은 `backoffLimit`로 재시도 제어합니다. 파드 단에서는 CrashLoopBackOff처럼 보일 수 있으나 **Job 컨트롤러**가 실패 처리합니다.
```yaml
spec:
  backoffLimit: 3
```

**Q3. 멀티 컨테이너 파드에서 한 컨테이너만 크래시합니다.**  
A. `containerStatuses`에서 문제 컨테이너만 골라 로그/상태 확인, Sidecar 종속성/공유 볼륨 경합 점검.

---

## 13) 요약(한 장)

- CrashLoopBackOff는 **“시작은 했으나 곧 죽고, 지수 백오프로 재시작”** 상태.
- 60초 루틴: `describe` → `logs --previous` → `events` → 필요 시 `debug`/`startupProbe`.
- 최다 원인: **프로브 오탐, env/config 누락, 엔트리포인트/포트/권한/리소스(OOM)**.
- 해결은 **로그 시그니처 기반**으로: 설정 수정(YAML), 리소스 튜닝, 프로브 재설계, 코드/빌드 정정.
- 배포 파이프라인과 연계해 **빠른 롤백/게이트**를 갖추면 MTTR을 대폭 줄일 수 있다.

---

## 부록: 명령 스니펫 모음

```bash
# 상태 요약
kubectl get pod <POD> -o wide
kubectl describe pod <POD>

# 현재/이전 로그
kubectl logs <POD> -c <CONTAINER>
kubectl logs <POD> -c <CONTAINER> --previous

# 상태 필드 추출
kubectl get pod <POD> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}'
kubectl get pod <POD> -o jsonpath='{.status.containerStatuses[*].restartCount}{"\n"}'

# 이벤트
kubectl get events --sort-by=.lastTimestamp -A | tail -n 50

# 리소스
kubectl top pod <POD>

# 롤아웃/롤백
kubectl rollout status deploy/<NAME>
kubectl rollout undo deploy/<NAME>

# 디버그 진입
kubectl debug pod/<POD> -c debug --image=busybox:1.36 --target=<CONTAINER> -it -- sh
```

---

## 참고 링크

- Kubernetes Pod Lifecycle  
- Probe 설정 가이드  
- `kubectl logs`/`kubectl debug`/`rollout` 명령 사용법