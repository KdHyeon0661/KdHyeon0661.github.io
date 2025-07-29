---
layout: post
title: Kubernetes - Health Check
date: 2025-04-27 21:20:23 +0900
category: Kubernetes
---
# Kubernetes Health Check 이해하기  
## Liveness Probe vs Readiness Probe

Kubernetes는 애플리케이션의 **상태를 주기적으로 체크**해서  
정상인지 아닌지를 판단하고 자동으로 조치를 취할 수 있습니다.  

이를 위해 사용하는 것이 바로 **Probe(프로브)**입니다.

---

## ✅ 1. Probe란?

**Probe**는 쿠버네티스가 **컨테이너의 상태를 체크**하기 위해 사용하는 메커니즘입니다.

Kubernetes에는 다음 3가지 종류의 프로브가 있습니다:

| 종류 | 설명 |
|------|------|
| **Liveness Probe** | 컨테이너가 **살아있는지(죽었는지)** 확인 |
| **Readiness Probe** | 컨테이너가 **서비스를 받을 준비가 되었는지** 확인 |
| **Startup Probe** | 컨테이너가 **시작되는 데 오래 걸릴 때** 체크 (선택적) |

---

## ✅ 2. Liveness Probe: 살아있는가?

### 💡 목적  
애플리케이션이 **응답은 멈췄지만, 프로세스는 죽지 않은 상태**를 감지하여  
Kubernetes가 **컨테이너를 자동으로 재시작**할 수 있게 합니다.

> 예: 서버가 메모리 누수로 멈췄는데 OS 레벨에서는 살아있을 때

### ✅ 예제: HTTP 체크 방식

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
```

- `initialDelaySeconds`: 시작 후 몇 초 뒤부터 체크할지
- `periodSeconds`: 몇 초마다 체크할지
- `failureThreshold`: 몇 번 연속 실패하면 죽은 것으로 간주할지

→ `/healthz` 경로가 3번 연속 실패하면 `kubectl describe pod`에서 이벤트 확인 가능  
→ 컨테이너 자동 재시작됨

---

## ✅ 3. Readiness Probe: 준비되었는가?

### 💡 목적  
애플리케이션이 **요청을 받을 준비가 되었는지 확인**하고,  
준비되지 않았다면 **Service에서 트래픽을 차단**합니다.

> 즉, **트래픽을 받을지 여부 결정** = Load Balancer 연결 여부

### ✅ 예제: TCP 방식

```yaml
readinessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 5
  periodSeconds: 10
```

→ 컨테이너가 포트 3306에서 응답하지 않으면 해당 Pod으로 트래픽을 보내지 않음

---

## ✅ 4. Startup Probe (선택)

컨테이너가 **기동 속도가 느릴 때** 사용하는 추가적인 Probe입니다.  
`startupProbe`가 성공할 때까지는 `livenessProbe`와 `readinessProbe`가 동작하지 않습니다.

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 5
```

→ 150초(= 30×5초) 동안 `/startup` 성공을 기다림

---

## ✅ 5. 실행 방식

Kubernetes는 아래 3가지 방법 중 하나로 Probe를 실행할 수 있습니다:

| 방식 | 설명 | 예시 |
|------|------|------|
| `httpGet` | HTTP 요청 보냄 | `/health`, `/ready` |
| `tcpSocket` | TCP 포트 오픈 여부 확인 | `3306`, `6379` |
| `exec` | 컨테이너 안에서 명령 실행 | `["cat", "/tmp/ready"]` |

---

## ✅ 6. 전체 YAML 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 2
```

---

## ✅ 7. Liveness vs Readiness 비교

| 항목 | Liveness Probe | Readiness Probe |
|------|----------------|-----------------|
| 목적 | 컨테이너가 **죽었는지** 감지 | 컨테이너가 **요청 받을 준비가 되었는지** 확인 |
| 실패 시 | 컨테이너 **재시작** | Service에서 **제외** (트래픽 차단) |
| 영향 | 프로세스 리셋 | 네트워크 연결만 끊김 |
| 예시 | 메모리 누수, 데드락 | DB 연결 대기, 캐시 로딩 등 |

---

## ✅ 8. 테스트 방법

### Liveness 시뮬레이션

```bash
kubectl describe pod <pod-name>
# "Liveness probe failed" 로그 확인
```

### 동작 확인

```bash
kubectl get endpoints <service-name>
```

→ Readiness가 false면 endpoint에서 제외됨

---

## ✅ 9. 실제 운영 시 팁

- Liveness Probe는 **너무 빠르게 실패 처리하면 위험** (ex: GC 중인 앱이 죽음으로 판단될 수 있음)
- Readiness Probe는 **트래픽 유입 전 설정 필수**
- `exec`는 비용이 높으므로 HTTP 또는 TCP 방식 권장
- Spring Boot, Express 등에서는 `/actuator/health`, `/readyz`, `/livez` 등 엔드포인트 활용 가능
- `failureThreshold`와 `periodSeconds`를 조합해 **시간 버퍼 확보**

---

## ✅ 결론

Kubernetes에서 **Liveness**와 **Readiness**는 애플리케이션의 생존성과 준비 상태를 체크하여  
보다 **탄력적이고 안정적인 배포 환경**을 구축하는 데 핵심적인 역할을 합니다.

| 시나리오 | 사용 프로브 |
|----------|--------------|
| 애플리케이션이 먹통이지만 죽진 않음 | ✅ Liveness |
| DB 연결 등 외부 의존성 대기 중 | ✅ Readiness |
| 시작이 오래 걸리는 앱 | ✅ Startup |