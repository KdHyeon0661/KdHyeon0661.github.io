---
layout: post
title: Kubernetes - Health Check
date: 2025-04-27 21:20:23 +0900
category: Kubernetes
---
# Kubernetes Health Check: 완전한 이해와 실전 구현 가이드

## 헬스 체크(Probe)의 개념과 중요성

쿠버네티스에서 헬스 체크는 컨테이너의 상태를 주기적으로 점검하여 시스템의 안정성과 가용성을 보장하는 핵심 메커니즘입니다. 세 가지 주요 헬스 체크 유형은 각기 다른 목적과 동작 방식을 가지고 있습니다:

| 헬스 체크 유형 | 핵심 질문 | 실패 시 동작 | 주요 목적 |
|---|---|---|---|
| **Liveness Probe** | "컨테이너 프로세스가 정상적으로 실행 중인가?" | 컨테이너 재시작 | 데드락, 응답 불능 상태 감지 및 자가 치유 |
| **Readiness Probe** | "컨테이너가 트래픽을 처리할 준비가 되었는가?" | 서비스 엔드포인트에서 제외 | 초기화 중이거나 의존성 문제 시 트래픽 차단 |
| **Startup Probe** | "컨테이너 초기화가 완료되었는가?" | 성공 전까지 다른 헬스 체크 보류 | 느린 시작 애플리케이션 보호 |

각 헬스 체크는 다음 세 가지 방식 중 하나로 구현될 수 있습니다:
- **`httpGet`**: 지정된 경로로 HTTP 요청을 보내 응답 상태 코드 확인
- **`tcpSocket`**: 특정 포트가 열려 있는지 확인
- **`exec`**: 컨테이너 내부에서 명령어 실행 (종료 코드 0 = 성공)

---

## 헬스 체크의 실제 동작과 영향

### Readiness Probe의 영향
Readiness Probe가 실패하면 해당 Pod는 서비스의 엔드포인트 목록(EndpointSlice)에서 제외됩니다. 이는 로드밸런서, 인그레스, 서비스 디스커버리가 해당 Pod로 트래픽을 전달하지 않음을 의미합니다.

### Liveness Probe의 영향
Liveness Probe가 연속적으로 실패하면 kubelet이 컨테이너를 강제로 재시작합니다. 이는 프로세스가 응답하지 않지만 여전히 실행 중인 상태(예: 데드락)에서 복구하기 위한 메커니즘입니다.

### Startup Probe의 역할
Startup Probe가 성공하기 전까지는 Liveness와 Readiness Probe가 평가되지 않습니다. 이는 초기화에 시간이 오래 걸리는 애플리케이션이 시작 도중에 잘못된 재시작을 당하는 것을 방지합니다.

### 헬스 체크 타이밍 파라미터
각 헬스 체크는 다음과 같은 타이밍 파라미터로 세밀하게 제어됩니다:

- **`initialDelaySeconds`**: 컨테이너 시작 후 첫 번째 헬스 체크까지의 대기 시간
- **`periodSeconds`**: 헬스 체크 실행 간격
- **`timeoutSeconds`**: 각 헬스 체크의 응답 대기 시간 제한
- **`failureThreshold`**: 연속 실패 허용 횟수
- **`successThreshold`**: 연속 성공 필요 횟수 (Readiness Probe에서 특히 중요)

### 실패 판정 시간 계산
헬스 체크 실패 판정까지 걸리는 대략적인 시간은 다음과 같이 계산할 수 있습니다:

```
총 실패 판정 시간 ≈ failureThreshold × periodSeconds
```

---

## 실전 구현 예제: HTTP 기반 애플리케이션

애플리케이션이 `/livez`, `/readyz`, `/startupz` 엔드포인트를 제공한다고 가정한 예제입니다:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-application-with-probes
spec:
  containers:
  - name: web-server
    image: ghcr.io/example/web-application:1.0.0
    ports:
    - containerPort: 8080
    # Startup Probe: 초기화 보호
    startupProbe:
      httpGet:
        path: /startupz
        port: 8080
      periodSeconds: 5
      failureThreshold: 30     # 최대 150초(5분) 동안 초기화 허용
    # Liveness Probe: 프로세스 생존 확인
    livenessProbe:
      httpGet:
        path: /livez
        port: 8080
      initialDelaySeconds: 15  # 컨테이너 시작 후 15초 대기
      periodSeconds: 10        # 10초마다 확인
      timeoutSeconds: 2        # 2초 내 응답 필요
      failureThreshold: 3      # 연속 3회 실패 시 재시작 (약 30초 내)
    # Readiness Probe: 서비스 준비 상태 확인
    readinessProbe:
      httpGet:
        path: /readyz
        port: 8080
      initialDelaySeconds: 5   # 시작 후 5초 대기
      periodSeconds: 5         # 5초마다 확인
      timeoutSeconds: 2        # 2초 내 응답 필요
      successThreshold: 1      # 1회 성공으로 준비 상태 판정
      failureThreshold: 3      # 연속 3회 실패 시 준비 상태 해제
```

**핵심 설계 원칙**:
1. **Startup Probe**: 느린 초기화가 필요한 애플리케이션을 보호합니다. Startup Probe가 성공하기 전에는 다른 헬스 체크가 실행되지 않습니다.
2. **Liveness Probe**: 보수적으로 설정하여 불필요한 재시작을 방지합니다. 너무 공격적인 설정은 안정성을 해칠 수 있습니다.
3. **Readiness Probe**: 애플리케이션이 실제로 요청을 처리할 준비가 되었을 때만 성공 상태를 반환해야 합니다. 외부 의존성(데이터베이스, 캐시 등) 상태를 포함하여 평가하는 것이 좋습니다.

---

## 다양한 애플리케이션 유형에 대한 헬스 체크 구현

### 데이터베이스, 캐시, 메시지 브로커 (TCP 소켓 검사)
HTTP 엔드포인트가 없는 애플리케이션의 경우 TCP 소켓 검사를 사용할 수 있습니다:

```yaml
readinessProbe:
  tcpSocket:
    port: 6379  # Redis 포트
  periodSeconds: 5
  failureThreshold: 3

livenessProbe:
  tcpSocket:
    port: 6379
  initialDelaySeconds: 15
  periodSeconds: 10
```

**주의사항**: 포트가 열려 있다고 해서 애플리케이션이 완전히 정상 상태라는 보장은 없습니다. 가능한 경우 추가적인 상태 검사를 고려해야 합니다.

### Exec 명령어를 활용한 커스텀 검사
더 복잡한 상태 검사가 필요한 경우 exec 방식을 사용할 수 있습니다:

```yaml
readinessProbe:
  exec:
    command: 
    - sh
    - -c
    - "test -f /var/run/application.ready && exit 0 || exit 1"
  periodSeconds: 5

livenessProbe:
  exec:
    command:
    - sh
    - -c
    - "ps aux | grep -q '[a]pplication-main'"
  initialDelaySeconds: 20
  periodSeconds: 10
```

**성능 고려사항**: exec 방식은 프로세스 포크와 쉘 실행 오버헤드가 발생할 수 있으므로, 검사 주기를 적절히 설정하고 명령어를 가볍게 유지하는 것이 중요합니다.

### gRPC 애플리케이션
gRPC 애플리케이션의 경우 다음과 같은 방법을 고려할 수 있습니다:

1. **grpc-health-probe 도구 활용** (권장):
```yaml
readinessProbe:
  exec:
    command: 
    - /bin/grpc_health_probe
    - -addr=:8080
    - -service=my.Service

livenessProbe:
  exec:
    command:
    - /bin/grpc_health_probe
    - -addr=:8080
```

2. **HTTP 헬스 엔드포인트 추가**: 운영 편의성을 위해 gRPC 서버에 간단한 HTTP 헬스 엔드포인트를 추가하는 방법도 효과적입니다.

---

## Startup Probe를 통한 느린 시작 애플리케이션 보호

느린 초기화가 필요한 애플리케이션(대규모 캐시 로딩, JIT 컴파일, 데이터 마이그레이션 등)은 Startup Probe 없이는 안정적으로 운영하기 어렵습니다:

### 전형적인 Startup Probe 구성
```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  periodSeconds: 5
  failureThreshold: 60   # 최대 5분(300초) 동안 초기화 허용
```

### Startup Probe가 해결하는 문제
Startup Probe가 없을 때 발생할 수 있는 일반적인 문제:
- **재시작 루프**: Liveness Probe가 초기화 중인 애플리케이션의 느린 응답을 오작동으로 판단하여 불필요한 재시작을 유발
- **서비스 중단**: Readiness Probe만 있는 경우, 초기화 중 트래픽은 차단되지만 Liveness Probe에 의해 컨테이너가 재시작될 수 있음

---

## 멀티 컨테이너 Pod와 사이드카 패턴

멀티 컨테이너 Pod에서는 각 컨테이너마다 독립적인 헬스 체크를 구성합니다. Pod 전체의 준비 상태는 모든 컨테이너의 Readiness Probe가 성공해야만 결정됩니다:

```yaml
spec:
  containers:
  - name: application
    readinessProbe:
      httpGet:
        path: /readyz
        port: 8080
    livenessProbe:
      httpGet:
        path: /livez
        port: 8080
      
  - name: envoy-sidecar
    readinessProbe:
      httpGet:
        path: /ready
        port: 15021
    livenessProbe:
      tcpSocket:
        port: 15000
```

이 구성에서는 애플리케이션 컨테이너와 Envoy 사이드카 컨테이너 모두 정상 상태여야 Pod가 서비스 트래픽을 받을 준비가 된 것으로 간주됩니다.

---

## 서비스, 배포, 롤링 업데이트와의 통합

헬스 체크는 쿠버네티스의 다른 구성 요소와 밀접하게 통합되어 작동합니다:

### 서비스 트래픽 제어
- Readiness Probe가 실패하면 Pod는 서비스의 엔드포인트 목록에서 즉시 제외됩니다.
- Readiness Probe가 다시 성공하면 Pod는 엔드포인트 목록에 자동으로 재추가됩니다.

### 디플로이먼트 롤링 업데이트
헬스 체크는 무중단 배포의 핵심 요소입니다. 디플로이먼트의 롤링 업데이트 전략과 결합하여 안전한 배포를 보장합니다:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # 최대 1개의 Pod만 동시에 사용 불가능
      maxSurge: 1        # 최대 1개의 Pod만 초과 생성
  minReadySeconds: 10    # 준비 상태로 간주된 후 최소 10초 유지
```

### PodDisruptionBudget(PDB)
PDB와 헬스 체크를 조합하면 유지보수 작업 중에도 최소한의 서비스 가용성을 보장할 수 있습니다:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: application-pdb
spec:
  minAvailable: 2  # 최소 2개의 Pod는 항상 사용 가능해야 함
  selector:
    matchLabels:
      app: web-application
```

---

## 헬스 체크 타이밍 최적화 가이드

### 일반적인 웹 애플리케이션 권장 설정
```yaml
# Startup Probe (느린 초기화 애플리케이션용)
startupProbe:
  periodSeconds: 5
  failureThreshold: 60      # 최대 5분 초기화 허용

# Readiness Probe
readinessProbe:
  initialDelaySeconds: 5    # 컨테이너 시작 후 5초 대기
  periodSeconds: 5          # 5초마다 확인
  timeoutSeconds: 2         # 2초 내 응답 필요
  failureThreshold: 3       # 연속 3회 실패 시 준비 상태 해제

# Liveness Probe
livenessProbe:
  initialDelaySeconds: 30   # 충분한 시작 시간 확보
  periodSeconds: 10         # 10초마다 확인
  timeoutSeconds: 2         # 2초 내 응답 필요
  failureThreshold: 3       # 연속 3회 실패 시 재시작
```

### 시간 파라미터 계산 공식
- **Readiness 성공 판정 시간**: `successThreshold × periodSeconds`
- **Readiness 실패 판정 시간**: `failureThreshold × periodSeconds`
- **Liveness 재시작까지 시간**: `initialDelaySeconds + (failureThreshold × periodSeconds)`

### 장시간 GC(가비지 컬렉션)가 발생하는 런타임
자바 가상 머신(JVM)과 같이 장시간 Stop-the-world GC가 발생할 수 있는 런타임의 경우, Liveness Probe를 더 관대하게 설정해야 합니다:

```yaml
livenessProbe:
  initialDelaySeconds: 60     # 더 긴 초기 대기 시간
  periodSeconds: 15           # 더 긴 검사 간격
  timeoutSeconds: 5           # 더 긴 응답 대기 시간
  failureThreshold: 3         # 여러 번 기회 부여
```

---

## 애플리케이션 구현 예시

### Node.js (Express) 애플리케이션
```javascript
const express = require('express');
const app = express();

// 상태 플래그
let isReady = false;
let isLive = true;

// 초기화 시뮬레이션 (예: 캐시 프리로드)
setTimeout(() => {
  isReady = true;
  console.log('애플리케이션 초기화 완료');
}, 15000); // 15초 초기화 시간

// Liveness 엔드포인트: 기본 프로세스 상태 확인
app.get('/livez', (req, res) => {
  if (isLive) {
    return res.status(200).json({ status: 'healthy', timestamp: new Date().toISOString() });
  }
  return res.status(503).json({ status: 'unhealthy', timestamp: new Date().toISOString() });
});

// Readiness 엔드포인트: 서비스 준비 상태 확인
app.get('/readyz', async (req, res) => {
  if (!isReady) {
    return res.status(503).json({ 
      status: 'not ready', 
      reason: 'initializing',
      timestamp: new Date().toISOString() 
    });
  }
  
  // 추가 의존성 검사 (예: 데이터베이스 연결)
  try {
    // 데이터베이스 연결 상태 확인 로직
    const dbHealthy = await checkDatabaseHealth();
    
    if (dbHealthy) {
      return res.status(200).json({ 
        status: 'ready', 
        dependencies: { database: 'healthy' },
        timestamp: new Date().toISOString() 
      });
    } else {
      return res.status(503).json({ 
        status: 'not ready', 
        reason: 'database unavailable',
        timestamp: new Date().toISOString() 
      });
    }
  } catch (error) {
    return res.status(503).json({ 
      status: 'not ready', 
      reason: 'health check failed',
      error: error.message,
      timestamp: new Date().toISOString() 
    });
  }
});

// Startup 엔드포인트
app.get('/startupz', (req, res) => {
  if (isReady) {
    return res.status(200).json({ 
      status: 'startup complete',
      timestamp: new Date().toISOString() 
    });
  }
  return res.status(503).json({ 
    status: 'starting up',
    timestamp: new Date().toISOString() 
  });
});

app.listen(8080, () => {
  console.log('애플리케이션 8080 포트에서 실행 중');
});
```

### Spring Boot 애플리케이션
Spring Boot는 Actuator를 통해 내장된 헬스 체크 기능을 제공합니다:

```yaml
# application.yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      probes:
        enabled: true  # /actuator/health/liveness, /actuator/health/readiness 자동 활성화
      show-details: always
server:
  port: 8080
```

쿠버네티스 매니페스트:
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5

startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  periodSeconds: 5
  failureThreshold: 60
```

---

## 외부 의존성과의 통합 전략

### 데이터베이스 및 외부 서비스 연계
헬스 체크 설계 시 외부 의존성에 대한 올바른 접근 방식:

1. **Readiness Probe**: 외부 의존성(데이터베이스, 캐시, 외부 API) 상태를 포함하여 평가
   - 데이터베이스 연결 풀 준비 상태 확인
   - 핵심 외부 서비스 응답 시간 모니터링
   - 캐시 연결 상태 검증

2. **Liveness Probe**: 주로 내부 상태에 집중
   - 외부 의존성 실패를 Liveness 실패로 처리하지 않음
   - 내부 스레드 풀, 메모리 상태, 큐 길이 등 내부 메트릭 모니터링

**권장 패턴**:
- Readiness: 데이터베이스 ping 성공 시 200, 실패 시 503 반환
- Liveness: 내부 처리 루프/스레드/메모리 상태 위주로 평가

### 외부 의존성 실패 처리 안티패턴
외부 의존성 실패를 Liveness Probe 실패로 처리하면 다음과 같은 문제가 발생할 수 있습니다:
- **재시작 폭풍**: 일시적인 외부 서비스 장애 시 모든 Pod가 연쇄적으로 재시작
- **상황 악화**: 재시작 중인 Pod는 의존성 복구에 기여할 수 없음
- **불필요한 복잡성**: 외부 문제를 내부 재시작으로 해결하려는 잘못된 접근

---

## 정상 종료(Graceful Shutdown)와의 통합

안전한 종료를 보장하기 위해 헬스 체크와 라이프사이클 훅을 조합할 수 있습니다:

```yaml
spec:
  containers:
  - name: application
    lifecycle:
      preStop:
        exec:
          command: 
          - sh
          - -c
          - |
            # Readiness 상태를 false로 변경
            curl -s -o /dev/null -X POST http://127.0.0.1:8080/prepare-shutdown
            
            # 연결 드레인을 위한 대기 시간
            sleep 10
    terminationGracePeriodSeconds: 30  # 종료 유예 시간
```

애플리케이션 측 구현:
```javascript
// 정상 종료 준비 엔드포인트
app.post('/prepare-shutdown', (req, res) => {
  isReady = false;  // Readiness 상태 변경
  console.log('종료 준비 중... 새 연결 수락 중지');
  
  // 진행 중인 요청 완료 대기 로직
  // 연결 풀 정리 등
  
  res.status(200).json({ message: '종료 준비 시작됨' });
});
```

---

## 일반적인 문제 진단 및 해결

### 문제 상황과 해결 방안

| 증상 | 가능한 원인 | 해결 방법 |
|---|---|---|
| 재시작 루프 발생 | Liveness Probe가 너무 공격적 | Startup Probe 도입, Liveness Probe 완화, 내부 상태 메트릭 기반 조건 개선 |
| 롤링 업데이트 중 503 오류 | Readiness Probe가 너무 일찍 성공 상태로 전환 | 캐시/연결 풀 완전 준비 후 성공 반환, `successThreshold` 증가 |
| 초기화 중 컨테이너 재시작 | Startup Probe 미사용 | Startup Probe 추가, `failureThreshold` × `periodSeconds`로 초기화 시간 충분히 확보 |
| 높은 CPU 사용량 | exec Probe 비용 과다 | HTTP/TCP Probe로 전환 또는 검사 주기 증가 |
| Pod 준비 상태지만 트래픽 없음 | 서비스 셀렉터 불일치 | `kubectl get endpoints`로 실제 엔드포인트 확인, 라벨 일치 검증 |
| 간헐적 5xx 오류 | Readiness 상태 변동(플래핑) | `minReadySeconds` 사용, `successThreshold` 증가 |

### 진단 명령어
```bash
# Pod 상태 상세 확인
kubectl describe pod <pod-name>

# 서비스 엔드포인트 확인
kubectl get endpoints <service-name> -o wide

# 엔드포인트슬라이스 확인
kubectl get endpointslices -l kubernetes.io/service-name=<service-name>

# 컨테이너 로그 확인
kubectl logs <pod-name> -c <container-name>

# 실시간 이벤트 모니터링
kubectl get events --field-selector involvedObject.name=<pod-name> --sort-by=.lastTimestamp -w
```

---

## 피해야 할 안티패턴

효과적인 헬스 체크 구현을 위해 피해야 할 일반적인 안티패턴:

1. **모든 실패를 Liveness 실패로 처리**: 외부 의존성 장애까지 재시작으로 처리하면 상황을 악화시킬 수 있습니다.

2. **과도하게 짧은 검사 주기**: 1초 주기와 같은 과도한 검사는 불필요한 부하를 생성하고 시스템 노이즈를 증가시킵니다.

3. **항상 200을 반환하는 /healthz 엔드포인트**: 아무런 상태 정보도 제공하지 않는 헬스 체크는 무의미합니다.

4. **느린 초기화 애플리케이션에 Startup Probe 미사용**: 롤링 업데이트 시 무한 재시작 루프를 유발할 수 있습니다.

5. **헬스 체크 없이 HPA/롤링 업데이트에 의존**: 트래픽 전환 및 드레인 작업이 실패할 수 있습니다.

---

## 테스트 및 시뮬레이션 방법

### Liveness 실패 시뮬레이션
애플리케이션에 디버깅용 엔드포인트를 추가하여 테스트할 수 있습니다:

```bash
# 애플리케이션의 /simulate-failure 엔드포인트 호출
kubectl exec -it deploy/web-application -- curl -X POST http://127.0.0.1:8080/simulate-failure

# Liveness 실패 이벤트 관찰
kubectl describe pod <pod-name> | grep -A5 "Liveness probe failed"
```

### Readiness 상태 변동 모니터링
```bash
# 엔드포인트 IP 주소 실시간 모니터링
watch -n1 'kubectl get endpoints <service-name> -o jsonpath="{.subsets[0].addresses[*].ip}"'
```

### 헬스 체크 실패 임계값 테스트
헬스 체크 파라미터를 점진적으로 조정하면서 시스템 동작을 관찰하는 것이 중요합니다:
1. `failureThreshold`를 낮춰 민감도 테스트
2. `periodSeconds`를 조정하여 부하 영향 확인
3. `timeoutSeconds`를 조정하여 네트워크 지연 영향 평가

---

## 보안 및 성능 고려사항

### 보안 모범 사례
- 헬스 체크 엔드포인트는 내부 네트워크(127.0.0.1)에만 바인딩하거나 NetworkPolicy로 외부 접근을 제한
- 민감한 정보가 헬스 체크 응답에 포함되지 않도록 주의
- 인증/인가가 필요한 경우 간단한 토큰 기반 인증 구현

### 성능 최적화
- 헬스 체크 핸들러는 최소한의 비용으로 구현 (무거운 데이터베이스 쿼리 회피)
- 자주 액세스하는 데이터는 메모리 캐시 활용
- 헬스 체크 요청 로깅은 샘플링하여 로그 볼륨 제어
- `periodSeconds`를 워크로드 특성에 맞게 최적화

---

## 종합 실전 예제: 완전한 프로덕션 구성

다음은 프로덕션 환경을 고려한 완전한 헬스 체크 구성 예제입니다:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-api
  labels:
    app: production-api
    environment: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  minReadySeconds: 10  # 준비 상태로 간주된 후 최소 10초 유지 (플래핑 방지)
  selector:
    matchLabels:
      app: production-api
  template:
    metadata:
      labels:
        app: production-api
        environment: production
    spec:
      containers:
      - name: api-server
        image: ghcr.io/organization/production-api:v1.5.2
        ports:
        - name: http
          containerPort: 8080
        # Startup Probe: 5분 동안 초기화 허용
        startupProbe:
          httpGet:
            path: /internal/startup
            port: http
          periodSeconds: 5
          failureThreshold: 60
        # Liveness Probe: 보수적인 설정
        livenessProbe:
          httpGet:
            path: /internal/live
            port: http
          initialDelaySeconds: 45  # JVM 등 느린 런타임 고려
          periodSeconds: 15
          timeoutSeconds: 3
          failureThreshold: 4      # 총 60초 내 연속 실패 시 재시작
        # Readiness Probe: 정확한 준비 상태 판단
        readinessProbe:
          httpGet:
            path: /internal/ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 2
          successThreshold: 2      # 2회 연속 성공 필요 (플래핑 방지)
          failureThreshold: 3      # 15초 내 연속 실패 시 준비 상태 해제
        # 정상 종료 지원
        lifecycle:
          preStop:
            exec:
              command:
              - sh
              - -c
              - |
                # Readiness 상태 false로 설정
                curl -sf -X POST http://127.0.0.1:8080/internal/prepare-shutdown || true
                # 진행 중 요청 완료 대기
                sleep 15
        # 리소스 제한
        resources:
          requests:
            cpu: "200m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
        # 환경 변수
        env:
        - name: LOG_LEVEL
          value: "info"
        - name: JAVA_OPTS
          value: "-Xms256m -Xmx512m"
      # Pod 수준 보안 컨텍스트
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      # 종료 유예 시간
      terminationGracePeriodSeconds: 60
---
apiVersion: v1
kind: Service
metadata:
  name: production-api-service
spec:
  selector:
    app: production-api
  ports:
  - name: http
    port: 80
    targetPort: http
  type: ClusterIP
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: production-api-pdb
spec:
  minAvailable: 2  # 최소 2개 Pod는 항상 사용 가능
  selector:
    matchLabels:
      app: production-api
```

---

## 결론: 효과적인 헬스 체크 설계 원칙

쿠버네티스 헬스 체크는 단순한 기술적 기능을 넘어 애플리케이션의 안정성, 가용성, 복원력을 결정하는 핵심 설계 요소입니다. 효과적인 헬스 체크 구현을 위한 핵심 원칙을 정리합니다:

### 1. 명확한 책임 분리
- **Startup Probe**: 초기화 완료 확인 - "시스템이 시작되었는가?"
- **Readiness Probe**: 서비스 준비 상태 확인 - "요청을 처리할 준비가 되었는가?"
- **Liveness Probe**: 프로세스 생존 상태 확인 - "시스템이 여전히 살아 있는가?"

### 2. 현실적인 타이밍 설정
- 애플리케이션의 실제 동작 특성(초기화 시간, GC 패턴, 외부 의존성 응답 시간)을 반영한 파라미터 설정
- 너무 공격적인 설정(짧은 주기, 낮은 실패 임계값)은 불필요한 재시작을 유발
- 너무 보수적인 설정은 문제 감지 및 복구를 지연

### 3. 의미 있는 상태 검사
- Readiness Probe는 실제 서비스 처리 능력을 정확히 반영해야 함
- Liveness Probe는 진정한 프로세스 장애를 감지해야 함
- 단순한 "동작 중" 표시가 아닌 실제 상태 정보 제공

### 4. 외부 의존성과의 적절한 분리
- 외부 의존성(DB, 캐시, 외부 API) 상태는 Readiness Probe에 반영
- 외부 의존성 실패를 Liveness 실패로 처리하지 않음 (재시작 폭풍 방지)
- 외부 복구 기다림 vs 내부 재시작의 적절한 균형 유지

### 5. 통합적 접근
- 헬스 체크를 서비스 디스커버리, 로드밸런싱, 롤링 업데이트, 자동 스케일링과 통합
- PodDisruptionBudget, 리소스 제한, 네트워크 정책과 조화로운 구성
- 모니터링, 로깅, 알림 시스템과의 연계

### 6. 지속적인 개선과 테스트
- 실제 장애 시나리오에서 헬스 체크 동작 검증
- 부하 테스트를 통한 헬스 체크 파라미터 최적화
- 운영 경험을 반영한 지속적인 튜닝

효과적인 헬스 체크는 "재시작은 최소화하되, 트래픽 전환은 신속하게"라는 목표를 실현하는 도구입니다. 이 가이드의 원칙을 바탕으로 조직의 특정 요구사항과 애플리케이션 특성에 맞는 최적의 헬스 체크 전략을 수립하고 구현한다면, 더욱 견고하고 신뢰할 수 있는 쿠버네티스 기반 서비스를 구축할 수 있을 것입니다.

헬스 체크 설계는 단순한 기술 구성이 아닌 서비스 운영 철학의 반영입니다. 애플리케이션의 실제 동작 방식을 깊이 이해하고, 시스템의 복잡성을 존중하며, 실패를 예상하고 대비하는 마음가짐으로 접근할 때, 진정으로 효과적인 헬스 체크 전략을 구축할 수 있습니다.