---
layout: post
title: Kubernetes - CrashLoopBackOff 원인 분석 및 해결
date: 2025-05-31 22:20:23 +0900
category: Kubernetes
---
# CrashLoopBackOff 원인 분석 및 해결 방법

## CrashLoopBackOff란?

CrashLoopBackOff는 Kubernetes에서 자주 발생하는 Pod 상태 중 하나로, 컨테이너가 반복적으로 시작되다가 비정상 종료되는 문제를 나타냅니다.

- **Crash**: 컨테이너의 메인 프로세스가 **비정상 종료**(0이 아닌 종료 코드 반환)됩니다.
- **Loop**: kubelet이 재시작 정책에 따라 컨테이너를 **반복적으로 재시작**합니다.
- **BackOff**: 각 재시도 간격이 **지수적으로 증가**합니다.

### 백오프 간격 메커니즘

재시도 횟수 \(k\)에 따른 대기 시간 \(t_k\) (최대값 \(t_{\max}\) 제한):

$$
t_k = \min\big(t_{\max},\; t_0 \cdot 2^{k}\big)
$$

일반적으로 초기 몇 초에서 시작하여 수십 초, 수분까지 점진적으로 증가합니다.

### 상태 확인 예시

```bash
kubectl get pod myapp-abc123
# NAME           READY   STATUS             RESTARTS   AGE
# myapp-abc123   0/1     CrashLoopBackOff   7          3m42s
```

---

## 즉시 적용 가능한 60초 진단 프로세스

현장에서 즉시 활용할 수 있는 체계적인 진단 순서입니다:

1. **Pod 상태 요약 확인**
   ```bash
   kubectl get pod <POD_NAME> -o wide
   kubectl describe pod <POD_NAME>    # 이벤트, 종료 코드, 프로브 결과, 리소스 정보 확인
   ```
2. **현재 및 이전 로그 분석**
   ```bash
   kubectl logs <POD_NAME> -c <CONTAINER_NAME> --timestamps
   kubectl logs <POD_NAME> -c <CONTAINER_NAME> --previous --timestamps
   ```
3. **상태 정보 상세 추출**
   ```bash
   kubectl get pod <POD_NAME> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}'
   kubectl get pod <POD_NAME> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.exitCode}{"\n"}'
   ```
4. **클러스터 이벤트 확인**
   ```bash
   kubectl get events --sort-by=.lastTimestamp -A | tail -n 50
   ```
5. **컨테이너 내부 직접 확인(필요 시)**
   - `startupProbe` 임시 추가 또는 **ephemeralContainer**를 통한 디버깅:
     ```bash
     kubectl debug pod/<POD_NAME> -c <CONTAINER_NAME> --image=busybox:1.36 --target=<CONTAINER_NAME> -it -- bash
     ```

> **중요 팁**: `--previous` 옵션을 사용하면 직전에 종료된 컨테이너 인스턴스의 마지막 로그를 확인할 수 있습니다. CrashLoopBackOff 문제 해결 시 **이전 로그**가 핵심 단서가 되는 경우가 많습니다.

---

## 원인 분류 및 주요 증상

| 카테고리 | 대표적인 로그/증상 | 핵심 조치 방안 |
|---|---|---|
| 1. 애플리케이션 런타임 예외 | NullPointerException, Uncaught Exception | 코드/구성 수정, 의존성 주입 확인 |
| 2. 잘못된 `command/args` 설정 | `not found`, `exec format error` | `command/args`와 이미지 ENTRYPOINT 일치성 점검 |
| 3. 환경변수/Secret/ConfigMap 누락 | `KeyError`, `file not found` | 참조 리소스 존재 여부 및 키 이름 일치 확인 |
| 4. 설정 파일 경로/권한 문제 | `permission denied`, `no such file` | 마운트 경로, 파일 모드, 소유자 설정 조정 |
| 5. 포트 바인딩 실패 | `address already in use` | 포트 변경 또는 중복 프로세스 제거 |
| 6. 데이터베이스/외부 의존성 지연 | 부팅 즉시 연결 실패 | `startupProbe` 도입, 재시도/백오프 로직 구현 |
| 7. Liveness 프로브 오탐지 | 몇 초 후 컨테이너 강제 종료 | `initialDelaySeconds` 증가, 헬스 엔드포인트 수정 |
| 8. Readiness 프로브 실패 | "정상이지만 트래픽 수신 불가" | readiness 기준 완화 또는 정확화 |
| 9. **OOMKilled(메모리 초과)** | `State: OOMKilled` | 메모리 `limits` 상향 또는 메모리 사용 최적화 |
| 10. CPU 스로틀링 과다 | 느린 시작 → 프로브 실패 | CPU `requests` 상향, `startupProbe` 도입 |
| 11. 파일시스템 읽기 전용 | `EROFS` (Read-only file system) | `readOnlyRootFilesystem` 설정 확인, 쓰기 경로 분리 |
| 12. 권한 상승/보안 컨텍스트 문제 | `Operation not permitted` | `capabilities`, `runAsUser`, Pod Security Admission 준수 |
| 13. 이미지/엔트리포인트 빌드 문제 | `no main manifest attribute` | Dockerfile CMD/ENTRYPOINT/경로 수정 |
| 14. InitContainer 실패 | init 종료 코드 ≠ 0 | init 로그 확인, 마운트 요구사항 수정 |
| 15. 사이드카 의존성 문제 | 애플리케이션이 사이드카 준비 전 시작 | `startupProbe` 또는 `initContainers`로 시작 순서 보장 |
| 16. 노드 리소스/디스크 부족 | `no space left on device` | 로그 로테이션 설정, emptyDir 사용량 축소 |
| 17. 커널/런타임 이슈 | 컨테이너 런타임 에러 | 노드 상태/런타임 로그 점검 |
| 18. 지역 설정/시간대 문제 | TLS/서명/만료 관련 오류 | 시간 동기화, CA 번들 확인 |
| 19. 경합/락 조건 | `database is locked` | 백오프/재시도 로직, 락 경합 설계 개선 |
| 20. 잘못된 프로브 경로/포맷 | 404/500/timeout | 올바른 경로, 포트, 헤더 설정 반영 |

---

## 사례별 상세 해결 레시피

### 잘못된 엔트리포인트 설정

**로그 메시지**
```
/bin/app: no such file or directory
```

**문제가 있는 구성**
```yaml
containers:
- name: app
  image: my/app:1.0.0
  command: ["/bin/app"]
```

**수정된 구성(이미지 내 실제 경로 확인)**
```yaml
containers:
- name: app
  image: my/app:1.0.1
  command: ["/usr/local/bin/app"]
  # 또는 쉘 형태로 실행
  # command: ["sh","-c","/usr/local/bin/app --config /etc/app/config.yaml"]
```

---

### Secret 키 이름 오타

**증상**: 애플리케이션 시작 즉시 `KeyError: PASSWORD` 발생

**문제가 있는 구성**
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: passwrod   # 오타!
```

**수정된 구성**
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

**검증 명령어**
```bash
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

---

### ConfigMap 마운트 경로 불일치

**로그 메시지**
```
open /etc/app/config.yaml: no such file or directory
```

**문제가 있는 구성**
```yaml
volumeMounts:
- name: cfg
  mountPath: /etc/appconf
volumes:
- name: cfg
  configMap:
    name: app-config
```

**수정된 구성**
```yaml
volumeMounts:
- name: cfg
  mountPath: /etc/app
  readOnly: true
```
> 애플리케이션이 기대하는 실제 경로(`/etc/app/config.yaml`)와 일치하도록 수정합니다.

---

### 공격적인 Liveness 프로브 설정

**증상**: 부팅에 10초가 걸리지만 `initialDelaySeconds: 5`로 인해 반복적 강제 종료

**문제가 있는 구성**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 1
```

**수정된 구성(StartupProbe 도입 권장)**
```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 2

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 0   # startupProbe 완료 후 liveness 시작
  periodSeconds: 10
  timeoutSeconds: 2
```

---

### OOMKilled(메모리 초과)

**상태 확인**
```bash
kubectl describe pod <POD_NAME> | grep -A2 "State:"
# State: Terminated (Reason: OOMKilled)
```

**문제가 있는 구성**
```yaml
resources:
  limits:
    memory: "256Mi"
  requests:
    memory: "128Mi"
```

**수정된 구성(메모리 여유 확보 또는 누수 점검)**
```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

**운영 팁**: 메모리 누수가 의심될 경우 **heap dump** 경로를 **쓰기 가능한 emptyDir**에 구성하고, **JVM/GC 옵션**을 조정하세요.

---

### 읽기 전용 루트 파일시스템과 쓰기 경로 충돌

**로그 메시지**
```
open /app/logs/app.log: read-only file system
```

**수정된 구성(쓰기 경로를 EmptyDir 또는 PVC로 분리)**
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

### InitContainer 실패

**진단 명령어**
```bash
kubectl logs <POD_NAME> -c <INIT_CONTAINER_NAME>
kubectl get pod <POD_NAME> -o jsonpath='{.status.initContainerStatuses[*].state.terminated}'
```

**수정된 구성(명령어, 권한, 마운트 조정)**
```yaml
initContainers:
- name: migrate
  image: my/migrator:1.2.0
  command: ["sh","-c","./migrate up || exit 1"]
  envFrom:
  - secretRef:
      name: db-secret
```

---

### 의존성 준비 전 애플리케이션 시작 문제

**해결 방안**: **StartupProbe**를 활용하여 모든 의존성 연결성 검증

```yaml
startupProbe:
  exec:
    command: ["sh","-c","nc -z localhost 15000 && curl -sf http://localhost:15000/server_info"]
  failureThreshold: 60
  periodSeconds: 2
```

---

### 권한 문제

**로그 메시지**
```
operation not permitted
```

**수정된 구성**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```
> 필요 시 특정 capability만 **추가(ADD)** 하되 최소한으로 제한하세요.

---

### 포트 충돌 문제

**로그 메시지**
```
bind: address already in use
```

**해결 방안**: 포트 변경 또는 **Sidecar/DaemonSet** 충돌 제거. `containerPort`와 `Service` 포트를 일치시키고, **중복 리스너**를 확인하세요.

---

## 체계적인 관찰 아키텍처

문제 해결을 위해 다음 정보를 체계적으로 확인하세요:

- **컨테이너 로그**: `--previous` 옵션 포함하여 전체 로그 확인
- **프로브 결과**: `describe` 명령의 `Liveness/Readiness` 실패 횟수 확인
- **컨테이너 상태**: `lastState.terminated.reason`, `exitCode`, `finishedAt` 정보 분석
- **노드/런타임 로그**: `journalctl -u kubelet`, `containerd`/`dockerd` 로그 확인
- **리소스 메트릭**: `kubectl top pod`, Prometheus의 `container_memory_working_set_bytes`, `container_cpu_usage_seconds_total` 확인
- **이벤트**: `Warning BackOff`, `Killing`, `Unhealthy`, `OOMKilling` 이벤트 모니터링

---

## 고급 디버깅 기법 모음

### 컨테이너 "살리기" 기법

```yaml
# 일시적인 디버그 목적 오버라이드

command: ["sh","-c","sleep 3600"]
```

배포 후 `kubectl exec`로 컨테이너 내부의 파일, 경로, 권한을 확인한 후 원인을 파악하고 본래 명령으로 복구하세요.

### Ephemeral Container 활용

```bash
kubectl debug pod/<POD_NAME> -c debug --image=busybox:1.36 -it --target=<CONTAINER_NAME>
```

> 크래시하는 컨테이너의 네임스페이스에 진입하여 **파일, 프로세스, 권한**을 직접 점검합니다.

### 종료 메시지 활용

```yaml
terminationMessagePolicy: FallbackToLogsOnError
```

**마지막 로그**를 상태에 저장하여 재현이 어려운 크래시 문제를 기록합니다.

---

## 프로브 설계 모범 사례

- **`startupProbe` 우선 적용**: 긴 부팅 시간이 필요한 애플리케이션(런타임 초기화, 캐시 예열, 마이그레이션) → liveness 프로브 발동 지연
- **`livenessProbe`는 "자가 복구 불가능" 상태만 감지**: 단순 의존성 장애는 **프로세스 강제 종료보다 재시도**가 더 적합합니다.
- **`readinessProbe`는 "트래픽 수용 가능" 기준 적용**: 데이터베이스 연결 등 핵심 의존성이 정상일 때만 True 반환.
- **시간 매개변수 조정**: `initialDelaySeconds`, `timeoutSeconds`, `periodSeconds`, `failureThreshold`를 **실제 측정 데이터 기반**으로 튜닝하세요.

실무 권장값 예시(시작점으로 활용):

```yaml
startupProbe:
  httpGet:
    path: /healthz/startup
    port: 8080
  periodSeconds: 2
  failureThreshold: 60  # 최대 120초 부팅 시간 허용

livenessProbe:
  httpGet:
    path: /healthz/live
    port: 8080
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /healthz/ready
    port: 8080
  periodSeconds: 5
  timeoutSeconds: 1
  failureThreshold: 2
```

---

## 리소스 튜닝 가이드(스로틀링·OOM 오류 방지)

- **CPU 스로틀링**: 과도한 스로틀링은 **느린 시작 → liveness 프로브 실패**를 유발합니다.
  - 해결책: `requests.cpu`를 **현실적인 값**으로 상향, `limits.cpu`를 지나치게 낮추지 마세요.
- **메모리 제한**: `limits.memory`를 초과하면 즉시 OOMKilled됩니다.
  - 해결책: 실제 사용 피크를 측정한 후 **여유 버퍼**(예: 95퍼센타일 + 20~30%)로 재설정하세요.

```yaml
resources:
  requests:
    cpu: "300m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

---

## 배포/롤백 전략과 통합(무한 Crash 방지)

- **배포 중 Crash 발생**: Deployment 진행이 멈추거나 느려질 수 있습니다 → `kubectl rollout status`로 지속 모니터링
- **신속한 롤백 실행**
  ```bash
  kubectl rollout undo deployment/<DEPLOYMENT_NAME>
  ```
- **Argo CD/Helm 통합**: **게이트 조건/자동 롤백** 설정(Argo Rollouts, 헬스 체크 기반)

---

## 재현 가능한 실습 워크로드

### 의도적인 Crash 시나리오(명령어 오류)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crash-sample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crash
  template:
    metadata:
      labels:
        app: crash
    spec:
      containers:
      - name: c
        image: alpine:3.20
        command: ["/bin/not-exist"]
```

**관찰 포인트**
- `describe` → `Back-off restarting failed container`
- `logs --previous` → `exec` 에러 확인

**수정 방법**: `command: ["sh","-c","echo OK; sleep 3600"]`

---

### Liveness 프로브 오탐지 시나리오

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: live-tight
spec:
  replicas: 1
  selector:
    matchLabels:
      app: live-tight
  template:
    metadata:
      labels:
        app: live-tight
    spec:
      containers:
      - name: app
        image: ghcr.io/nginxinc/nginx-unprivileged:stable
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080  # nginx에는 해당 경로가 없음 → 404 응답
          initialDelaySeconds: 5
          periodSeconds: 5
```

**결과**: 지속적인 강제 종료 → CrashLoopBackOff 발생

**해결책**: 올바른 경로로 수정 또는 `startupProbe` 도입.

---

## 자주 묻는 질문(FAQ)

**Q1. `ImagePullBackOff`와 다른 상태인가요?**
A. 예, 다릅니다. `ImagePullBackOff`는 **이미지 다운로드 실패** 상태로, 컨테이너가 시작조차 되지 않습니다. CrashLoopBackOff는 **컨테이너가 시작되었으나 크래시**된 상태입니다.

**Q2. Job에서도 CrashLoopBackOff가 발생하나요?**
A. Job은 `backoffLimit` 설정으로 재시도 횟수를 제어합니다. Pod 수준에서는 CrashLoopBackOff처럼 보일 수 있지만 **Job 컨트롤러**가 실패 처리를 관리합니다.
```yaml
spec:
  backoffLimit: 3
```

**Q3. 멀티 컨테이너 Pod에서 한 컨테이너만 크래시합니다.**
A. `containerStatuses`에서 문제가 있는 컨테이너를 특정하여 로그/상태를 확인하세요. Sidecar 의존성 또는 공유 볼륨의 경합 조건을 점검하세요.

---

## 결론

CrashLoopBackOff는 Kubernetes 운영에서 자주 마주치는 문제이지만, 체계적인 접근 방식을 통해 효과적으로 해결할 수 있습니다.

문제 해결의 핵심은 **로그 기반의 분석과 단계적인 진단 프로세스**에 있습니다. 컨테이너 로그(특히 `--previous` 옵션), 프로브 구성, 리소스 설정, 환경 변수 및 의존성 관계를 꼼꼼히 검토하세요.

예방 차원에서 **적절한 프로브 설정**, **현실적인 리소스 할당**, **철저한 구성 검증**이 중요합니다. StartupProbe를 활용하여 부팅 시간이 긴 애플리케이션을 보호하고, 메트릭 기반의 모니터링을 통해 문제를 조기에 감지하세요.

운영 환경에서는 **자동 롤백 메커니즘**과 **체계적인 배포 파이프라인**을 구축하여 CrashLoopBackOff로 인한 서비스 중단 시간을 최소화하는 것이 바람직합니다. 문제 발생 시 신속한 롤백 경로를 확보하고, 근본 원인 분석을 통해 재발을 방지하는 문화를 정착시키는 것이 장기적인 운영 안정성으로 이어집니다.

---

## 부록: 유용한 명령어 모음

```bash
# Pod 상태 요약

kubectl get pod <POD_NAME> -o wide
kubectl describe pod <POD_NAME>

# 현재 및 이전 로그 확인

kubectl logs <POD_NAME> -c <CONTAINER_NAME>
kubectl logs <POD_NAME> -c <CONTAINER_NAME> --previous

# 상태 정보 상세 추출

kubectl get pod <POD_NAME> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}'
kubectl get pod <POD_NAME> -o jsonpath='{.status.containerStatuses[*].restartCount}{"\n"}'

# 클러스터 이벤트 확인

kubectl get events --sort-by=.lastTimestamp -A | tail -n 50

# 리소스 사용량 확인

kubectl top pod <POD_NAME>

# 배포 상태 및 롤백

kubectl rollout status deployment/<DEPLOYMENT_NAME>
kubectl rollout undo deployment/<DEPLOYMENT_NAME>

# 디버그 컨테이너로 진입

kubectl debug pod/<POD_NAME> -c debug --image=busybox:1.36 --target=<CONTAINER_NAME> -it -- sh
```

---

## 참고 자료

- Kubernetes Pod Lifecycle 공식 문서
- Probe 설정 및 모범 사례 가이드
- `kubectl logs`, `kubectl debug`, `rollout` 명령어 상세 사용법
- Pod Security Admission(PSA) 및 보안 컨텍스트 설정 가이드