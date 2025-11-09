---
layout: post
title: Kubernetes - Health Check
date: 2025-04-27 21:20:23 +0900
category: Kubernetes
---
# Kubernetes Health Check 이해하기

## 1. Probe란?

Kubernetes는 컨테이너 단위로 **주기적 상태 점검**(probe)을 수행한다. 세 가지가 있다.

| 종류 | 질문 | 실패 시 동작 | 주 사용 목적 |
|---|---|---|---|
| Liveness | “프로세스가 살아 있나?” | 컨테이너 재시작 | 데드락·핸글(응답 불능) 감지 |
| Readiness | “트래픽 받을 준비 됐나?” | Service 엔드포인트에서 제외 | 초기화/의존성 대기 중 트래픽 차단 |
| Startup | “부팅이 다 끝났나?” | 성공 전까지 다른 probe 보류 | 느린 기동 앱 보호(콜드 스타트) |

Probe는 다음 중 한 방식으로 수행된다:

- `httpGet`: 지정 경로로 HTTP 요청
- `tcpSocket`: 포트 오픈 여부 확인
- `exec`: 컨테이너 내 명령 실행(0 = 성공)

---

## 2. Probe가 실제로 하는 일

- **Readiness 실패**: 해당 Pod는 **Service의 Endpoints(또는 EndpointSlice)** 에서 제외되며 로드밸런서/Ingress 트래픽 대상이 아니다.
- **Liveness 실패**: kubelet이 컨테이너를 **강제 재시작**한다(프로세스 초기화).
- **Startup 성공 전**: liveness/readiness는 **평가되지 않는다**(보호 구간).

Probe 타이밍 파라미터:

- `initialDelaySeconds`: 최초 검사 지연
- `periodSeconds`: 검사 주기
- `timeoutSeconds`: 응답 제한시간
- `failureThreshold`: 연속 실패 허용치
- `successThreshold`: 연속 성공 필요치(주로 Readiness에서 의미 있음)

실패 판단까지 걸리는 시간 예:
- 총 실패 시간 ≈ `failureThreshold × periodSeconds` (HTTP/TCP/exec가 `timeoutSeconds`로 즉시 실패해도, 주기는 period 기준으로 반복)

---

## 3. 최소 구현 예제 (HTTP 앱)

애플리케이션이 `/livez`, `/readyz`, `/startupz` 엔드포인트를 제공한다고 가정한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-probe-demo
spec:
  containers:
  - name: web
    image: ghcr.io/example/web:1.0.0
    ports:
    - containerPort: 8080
    startupProbe:
      httpGet:
        path: /startupz
        port: 8080
      periodSeconds: 5
      failureThreshold: 30     # 최대 150초까지 초기화 허용
    livenessProbe:
      httpGet:
        path: /livez
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 2
      failureThreshold: 3      # ≈ 30초 내 연속 실패 시 재시작
    readinessProbe:
      httpGet:
        path: /readyz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 2
      successThreshold: 1
      failureThreshold: 3
```

핵심 포인트:
- `startupProbe`가 성공하기 전에는 liveness/readiness가 **트리거되지 않는다**. 느린 기동(대규모 캐시 로드, JIT, 마이그레이션) 앱을 보호한다.
- `livenessProbe`는 보수적으로(너무 공격적으로 재시작하지 않도록) 설정한다.
- `readinessProbe`는 “정상 응답 + 의존성(예: DB 커넥션 풀 준비)” 조건을 모두 충족했을 때만 200 반환.

---

## 4. HTTP 서버가 없는 앱 (TCP·exec)

### 4.1 DB, 브로커, 캐시 등 포트 개방 확인(TCP)

```yaml
readinessProbe:
  tcpSocket:
    port: 6379
  periodSeconds: 5
  failureThreshold: 3
livenessProbe:
  tcpSocket:
    port: 6379
  initialDelaySeconds: 15
  periodSeconds: 10
```

- 포트가 열려 있어도 내부 상태가 정상이라고 보장하진 않는다. 가능하면 exec 또는 별도 health 에이전트를 병행.

### 4.2 exec로 내부 상태 점검

```yaml
readinessProbe:
  exec:
    command: ["sh","-c","test -f /tmp/READY && exit 0 || exit 1"]
  periodSeconds: 5
livenessProbe:
  exec:
    command: ["sh","-c","ps aux | grep -q '[m]y-worker'"]
  initialDelaySeconds: 20
  periodSeconds: 10
```

- 비용이 높은 편(프로세스 포크·쉘 실행). 주기는 길게, 명령은 가볍게 유지.

---

## 5. gRPC 애플리케이션

gRPC는 HTTP/2 바이너리 프로토콜로 기본 `httpGet`는 부적합하다. 다음 중 택1:

1) **grpc-health-probe** 바이너리 사용(권장)
```yaml
readinessProbe:
  exec:
    command: ["/bin/grpc_health_probe","-addr=:8080","-service=my.Service"]
livenessProbe:
  exec:
    command: ["/bin/grpc_health_probe","-addr=:8080"]
```

2) 앱 내부에 **HTTP 헬스 엔드포인트**를 추가(운영팀에 친화적).

---

## 6. 느린 기동 앱을 보호하는 Startup Probe

### 6.1 전형적 패턴
- 초기화(마이그레이션, 캐시워밍, 모델 로딩 등) 동안 **Startup만 성공/실패**를 판단.
- Startup 성공 이후부터 Liveness/Readiness가 활성화.

```yaml
startupProbe:
  httpGet: { path: /startup, port: 8080 }
  periodSeconds: 5
  failureThreshold: 60   # 최대 5분까지 허용
```

### 6.2 Startup 미사용 시의 문제
- Liveness가 초기화 중의 느린 응답을 “오작동”으로 오인 → 재시작 루프.
- Readiness만 둘 경우, 준비는 막지만 Liveness가 때때로 죽여버릴 수 있다.

---

## 7. 사이드카·멀티컨테이너 Pod

컨테이너별로 **독립된 Probe**를 둔다.

- 앱 컨테이너: `/livez`, `/readyz`
- 사이드카(예: Envoy, Log shipper): 자체 health 또는 TCP 포트 테스트
- 전체 Ready 판정은 **모든 컨테이너의 readiness = true** 여야 함

```yaml
spec:
  containers:
  - name: app
    # ... probes ...
  - name: envoy
    readinessProbe:
      httpGet: { path: /ready, port: 15021 }
```

---

## 8. Service·배포·롤링과의 상호작용

- **Readiness = false** → **EndpointSlice에서 제외** → 로드밸런싱 대상 아님.
- **Deployment 롤링 업데이트**에서 `maxUnavailable`, `maxSurge`, `minReadySeconds`와 결합하여 **무중단 배포**를 설계한다.
- **PodDisruptionBudget(PDB)**로 유지해야 할 최소 가용 복제수를 지정한다.

```yaml
spec:
  minReadySeconds: 5  # Ready 후 최소 유지시간(빠른 플랩 방지)
```

---

## 9. 시간 파라미터 튜닝 공식

대략적 가이드(HTTP 기준):

- **Readiness 안정 판정 시간**  
  ≈ `successThreshold × periodSeconds` (성공 연속 필요)
- **Readiness 실패 판정 시간**  
  ≈ `failureThreshold × periodSeconds`
- **Liveness 재시작까지의 시간**  
  ≈ `initialDelaySeconds + failureThreshold × periodSeconds`

권장 시작점(웹/백엔드 일반):

```text
startup:  period=5,   failure=60 (최대 5분 초기화 허용)
readiness:initialDelay=5, period=5, timeout=2, failure=3
liveness: initialDelay=30, period=10, timeout=2, failure=3
```

장시간 GC/Stop-the-world가 발생 가능한 런타임(자바 등)은 liveness를 더 느슨하게.

---

## 10. 애플리케이션 코드 샘플

### 10.1 Express(Node.js)

```javascript
// server.js
const express = require('express');
const app = express();

let ready = false;
let live = true;

// 간단 초기화 시뮬레이션 (예: 캐시 프리로드)
setTimeout(() => { ready = true; }, 15000);

// Liveness: 내부 상태 점검 (예: 이벤트루프 헬스, 큐 길이 등 추가 가능)
app.get('/livez', (_, res) => {
  if (live) return res.sendStatus(200);
  return res.sendStatus(500);
});

// Readiness: 외부 의존성(예: DB 커넥션 풀) 만족 시 200
app.get('/readyz', async (_, res) => {
  if (!ready) return res.sendStatus(503);
  // 예: DB ping 추가
  return res.sendStatus(200);
});

// Startup: 초기화 완료 전 200 금지
app.get('/startupz', (_, res) => ready ? res.sendStatus(200) : res.sendStatus(503));

app.listen(8080, () => console.log('listening on 8080'));
```

Dockerfile 예시:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY server.js ./
EXPOSE 8080
CMD ["node","server.js"]
```

### 10.2 Spring Boot (관리자 엔드포인트)

`application.yaml`:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      probes:
        enabled: true     # /actuator/health/liveness, /actuator/health/readiness
server:
  port: 8080
```

Probe:
```yaml
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 8080 }
startupProbe:
  httpGet: { path: /actuator/health, port: 8080 }
  failureThreshold: 60
  periodSeconds: 5
```

---

## 11. 데이터베이스·외부 의존성과의 연계

- Readiness는 **외부 의존성(예: DB, 캐시, 메시지 브로커, API)** 이 준비되었을 때만 200을 반환하라.
- Liveness는 외부 의존성 실패로 **즉시 죽이지 말 것**(잠시 장애일 수 있음).  
  외부 실패를 Liveness 실패로 삼으면 **재시작 폭풍**을 부르는 안티패턴.

권장:  
- Readiness: DB ping 성공 시 200, 실패 시 503  
- Liveness: 내부 처리 루프/스레드/큐 상태 위주

---

## 12. 멱등 롤링을 위한 프로브 + 종료 시그널

정상 종료에도 섬세함이 필요하다.

- `preStop` 훅에서 **Readiness=false**로 만들 시간을 확보 → **드레인 후 종료**
- `terminationGracePeriodSeconds`를 충분히 부여

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh","-c","curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8080/quit && sleep 5"]
terminationGracePeriodSeconds: 30
```

앱은 `/quit` 호출 시 내부 플래그를 내려 Readiness=false로 전환 후 연결을 정리한다.

---

## 13. 자주 발생하는 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| 재시작 루프 | Liveness가 과도하게 공격적 | Startup 도입, liveness 완화, 내부 지표 기반 조건 정교화 |
| 롤링 중 트래픽 503 | Readiness가 너무 일찍 true | 캐시/풀 준비 완료 후 true, `successThreshold` 조정 |
| 초기화 중 계속 죽음 | Startup 미사용 | Startup 도입, 초기화 시간만큼 failureThreshold×period 늘림 |
| exec Probe 고비용 | 쉘·프로세스 포크 오버헤드 | HTTP/TCP로 전환 또는 주기 완화 |
| 포드 준비인데 트래픽 없음 | 네임스페이스·라벨 미스매치, Service 셀렉터 오류 | `kubectl get endpoints/endpointslices`로 실제 대상 확인 |
| 간헐 5xx | Readiness 변동(플랩) | `minReadySeconds` 사용, successThreshold 상향 |

확인 명령:

```bash
kubectl describe pod <pod>
kubectl get endpoints <svc> -o wide
kubectl get endpointslices -l kubernetes.io/service-name=<svc>
kubectl logs <pod> -c <container>
```

---

## 14. Anti-Patterns

- **모든 실패를 Liveness로 처리**: 외부 의존 장애까지 재시작으로 덮으면 상황 악화.
- **하드코딩 1초 주기 검사**: 불필요한 부하와 노이즈. 합리적 주기·임계치를 두자.
- **/healthz가 항상 200**: 아무 것도 보장하지 않는다. 준비·생존 조건을 명확히 분리.
- **초기화 시간이 긴데 Startup 미사용**: 롤아웃 시 대참사(무한 재시작).
- **Probe 없이 HPA/롤링 의존**: 트래픽 절체·드레인 실패.

---

## 15. 테스트·시뮬레이션

### Liveness 실패 유도(Express 예시)
```bash
# 앱에 /panic 같은 엔드포인트가 있다면
kubectl exec -it deploy/web -- curl -sS http://127.0.0.1:8080/panic
kubectl describe pod <pod> | grep -A2 "Liveness probe failed"
```

### Readiness 플립 관찰
```bash
watch -n1 'kubectl get endpoints <svc> -o jsonpath="{.subsets[*].addresses[*].ip}"'
```

---

## 16. 보안·성능 고려

- 헬스 엔드포인트는 **내부 바인딩(127.0.0.1)** 또는 **NetworkPolicy**로 외부 노출을 제한.
- 헬스 핸들러는 **최소 비용**으로 구현(무거운 DB 쿼리 금지, 캐시 히트만 확인).
- 로그 지저분해지지 않도록 헬스 요청은 **샘플링/필터링**.

---

## 17. 종합 예시(Deployment + Service)

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  minReadySeconds: 5
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      containers:
      - name: api
        image: ghcr.io/example/api:1.2.3
        ports: [{ containerPort: 8080, name: http }]
        startupProbe:
          httpGet: { path: /startupz, port: http }
          periodSeconds: 5
          failureThreshold: 60
        livenessProbe:
          httpGet: { path: /livez, port: http }
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 3
        readinessProbe:
          httpGet: { path: /readyz, port: http }
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["sh","-c","curl -sf http://127.0.0.1:8080/quit || true; sleep 5"]
        resources:
          requests: { cpu: "200m", memory: "256Mi" }
          limits:   { cpu: "1",    memory: "1Gi"  }
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector: { app: api }
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

---

## 18. 요약

- **Readiness**는 **트래픽 라우팅**의 스위치, **Liveness**는 **프로세스 생존 판정**, **Startup**은 **부팅 보호막**이다.
- 느린 기동·외부 의존성·GC 등 현실을 반영해 **시간/임계치**를 설계하라.
- 헬스 엔드포인트는 “**실제 준비 조건**”을 반영해야 한다(캐시·풀 준비, 큐 정상, 내부 루프 정상).
- 롤링/Service/EndpointSlice/PDB와 결합해 **무중단 배포**를 완성하라.
- 공격적 Liveness, 무의미한 200, 과도한 exec는 대표적인 **안티패턴**이다.

이 가이드를 바탕으로, 현재 서비스의 프로브를 “왜, 어떻게” 설계했는지 점검하고 **재시작은 적게, 절체는 빠르게**라는 목표를 달성하라.