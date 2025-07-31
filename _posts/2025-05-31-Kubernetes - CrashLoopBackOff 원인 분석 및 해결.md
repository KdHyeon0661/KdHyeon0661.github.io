---
layout: post
title: Kubernetes - CrashLoopBackOff 원인 분석 및 해결
date: 2025-05-31 22:20:23 +0900
category: Kubernetes
---
# CrashLoopBackOff 원인 분석 및 해결 방법

Kubernetes에서 Pod 상태를 확인할 때 종종 보게 되는 `CrashLoopBackOff` 상태는 다음과 같은 의미를 갖습니다.

> ❗ **Pod가 시작되었지만, 컨테이너가 반복적으로 비정상 종료되며 재시작 중인 상태**

이 문서에서는 CrashLoopBackOff의 **개념**, **발생 원인**, **분석 방법**, **해결 예시**를 단계별로 소개합니다.

---

## ✅ CrashLoopBackOff란?

- **Crash**: 컨테이너가 비정상 종료됨
- **Loop**: 계속해서 반복적으로 실행됨
- **BackOff**: 재시도 간격이 점점 길어짐 (지수 백오프)

### 📌 예시

```bash
kubectl get pods

NAME           READY   STATUS             RESTARTS   AGE
myapp-abc123   0/1     CrashLoopBackOff   5          2m
```

---

## ✅ 원인별 분석 가이드

CrashLoopBackOff는 다양한 원인으로 발생할 수 있습니다. 주요 케이스를 분류해 보겠습니다.

---

### 🔹 1. 애플리케이션 자체 오류 (코드 오류, 예외 발생)

가장 흔한 원인입니다. 컨테이너는 정상적으로 시작되지만, 내부 애플리케이션이 종료됩니다.

#### 🔍 확인 방법:

```bash
kubectl logs <pod-name>
```

#### ✅ 예시:

```bash
Exception in thread "main" java.lang.NullPointerException
```

#### 💡 해결:

- 코드 디버깅
- 초기화 시 필요한 설정값 확인 (ENV, DB 연결 등)

---

### 🔹 2. `command`/`args` 설정 오류

컨테이너 시작 명령이 잘못되었거나, 잘못된 위치의 실행 파일 지정 시 Crash 발생

#### ✅ 예시 (deployment.yaml):

```yaml
command: ["/bin/app"]
```

→ 실제로 `/bin/app`이 존재하지 않으면 즉시 종료됨

#### 🔍 확인:

```bash
kubectl describe pod <pod-name>
```

→ `Failed to start container` 오류 메시지 확인

---

### 🔹 3. 환경 변수 누락 / 잘못된 설정

애플리케이션이 필요로 하는 환경 변수가 누락된 경우, 초기화에 실패하여 Crash

#### ✅ 예시:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

→ `db-secret`이 없거나 `key`가 오타일 경우 오류 발생

---

### 🔹 4. 파일 또는 볼륨 마운트 오류

`emptyDir`, `hostPath`, `configMap`, `secret` 등의 volume이 올바르게 마운트되지 않으면 앱이 Crash할 수 있습니다.

#### ✅ 예시:

- `/etc/config/config.yaml`을 찾지 못해 앱이 종료
- `secret`에 설정된 key가 누락됨

---

### 🔹 5. 포트 충돌 / 바인딩 실패

컨테이너 내부에서 이미 사용 중인 포트로 바인딩하려 하면 Crash 발생

#### 🔍 로그 예시:

```bash
bind: address already in use
```

---

### 🔹 6. Readiness/Liveness Probe 설정 오류

`livenessProbe`가 너무 타이트하게 설정된 경우, 앱이 정상인데도 K8s가 비정상 종료로 판단해 kill → 재시작 반복

#### ✅ 예시:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

→ 앱이 5초 안에 `/health` 응답을 못 하면 K8s가 재시작함

#### 💡 해결:

- `initialDelaySeconds`, `timeoutSeconds` 적절히 늘리기
- probe 경로 또는 포트가 실제 동작하는지 확인

---

### 🔹 7. 리소스 부족 (OOMKilled)

메모리 제한보다 더 많이 사용하면 K8s가 컨테이너를 종료시킵니다.

#### 확인:

```bash
kubectl describe pod <pod-name>
```

→ `State: OOMKilled`로 표시됨

#### 해결:

- `resources.limits.memory` 값을 증가
- 앱에서 메모리 누수 체크

---

## ✅ 진단 절차 요약

| 단계 | 명령어 | 설명 |
|------|--------|------|
| 1 | `kubectl get pods` | 상태 확인 (CrashLoopBackOff) |
| 2 | `kubectl logs <pod>` | 컨테이너 로그 확인 |
| 3 | `kubectl describe pod <pod>` | 이벤트, 상태, 리소스 정보 확인 |
| 4 | `kubectl get events` | 클러스터 내 이벤트 전체 확인 |
| 5 | `kubectl exec -it <pod> -- /bin/sh` | (가능할 경우) 직접 진입하여 디버깅 |

---

## ✅ 실전 예시: Java 앱 CrashLoopBackOff

```bash
kubectl logs myapp-1234

Error: Could not find or load main class com.example.Main
```

→ Dockerfile에 `ENTRYPOINT`를 잘못 지정했거나 `jar` 파일 경로 오류

**해결**: Dockerfile 수정 및 재배포

---

## ✅ 운영 팁

| 팁 | 설명 |
|-----|------|
| `restartPolicy: Never` | Debug 용도 Pod에 유용 (재시작 방지) |
| probe 설정 검증 | 실제 배포 전 `kubectl port-forward` 등으로 미리 확인 |
| `kubectl rollout status` | 배포 상태 모니터링 |
| `lifecycle.preStop` | 앱 종료 전 처리 로직 추가 가능 |

---

## ✅ 결론

| 요약 항목 | 설명 |
|-----------|------|
| 의미 | CrashLoopBackOff = 컨테이너 반복 실패 + 재시작 대기 |
| 주요 원인 | 코드 오류, 잘못된 CMD/ENV, 리소스 부족, probe 설정 등 |
| 진단 방법 | `logs`, `describe`, `events`, `probe` 확인 |
| 해결 전략 | 로그 기반 원인 파악 후 적절한 리소스/구성 수정

CrashLoopBackOff는 Kubernetes에서 **가장 빈번한 오류 중 하나**이며, 원인을 빠르게 파악하고 해결하는 것이 운영 안정성의 핵심입니다.

---

## ✅ 참고 링크

- [Kubernetes 공식 문서: Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Probe 설정 가이드](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [kubectl logs](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#logs)
