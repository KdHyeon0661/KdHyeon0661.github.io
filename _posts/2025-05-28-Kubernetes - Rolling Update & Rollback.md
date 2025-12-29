---
layout: post
title: Kubernetes - Rolling Update & Rollback
date: 2025-05-28 21:20:23 +0900
category: Kubernetes
---
# Kubernetes 배포 전략: Rolling Update & Rollback

## Rolling Update — 동작 원리와 핵심 개념

### 기본 동작 메커니즘

Rolling Update는 애플리케이션의 무중단 업데이트를 가능하게 하는 Kubernetes의 핵심 배포 전략입니다. 이 방식은 다음과 같은 과정으로 진행됩니다:

1. 기존 `ReplicaSet(이전 버전)`을 유지하면서
2. 새 템플릿으로 `ReplicaSet(새 버전)`을 생성합니다
3. **최대 허용 증설량**(`maxSurge`) 범위 내에서 새 Pod을 추가합니다
4. **최대 허용 가용성 감소**(`maxUnavailable`) 범위 내에서 기존 Pod을 제거합니다
5. 새로운 ReplicaSet이 목표 `replicas` 수에 도달하면 배포가 완료됩니다

### 핵심 구성 파라미터

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # 정수 또는 백분율 (기본값: 25%)
      maxUnavailable: 1     # 정수 또는 백분율 (기본값: 25%)
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:v2
        ports:
        - containerPort: 8080
```

- `maxSurge`: 배포 과정에서 **임시로 늘릴 수 있는** Pod 수
- `maxUnavailable`: 배포 중 **잠시 비워둘 수 있는** Pod 수
- 정수와 백분율 값을 혼용할 수 있습니다(반올림 규칙에 주의 필요)

### 가용성 계산 방식

배포 과정 중 **동시에 운영될 수 있는** 최대 Pod 수는 다음과 같습니다:
$$
\text{maxPods} = \text{replicas} + \text{maxSurge}
$$

동시에 **비활성 상태일 수 있는** 최소 Pod 수는 다음과 같습니다:
$$
\text{minAvailable} = \text{replicas} - \text{maxUnavailable}
$$

서비스 지연을 최소화하려면 다음 조건을 만족하도록 설정해야 합니다:
$$
\text{minAvailable} \approx \text{SLO 요구 QPS 대비 필요한 Pod 수}
$$

> 예시: `replicas=4`, `maxSurge=1`, `maxUnavailable=1` 설정 시 **3~5개** 사이에서 Pod이 이동하며 교체됩니다.

---

## 배포 헬스체크: Readiness, Liveness, Startup 프로브

**헬스 프로브** 구성은 롤링 업데이트의 성공 여부와 서비스 품질을 결정짓는 중요한 요소입니다.

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
livenessProbe:
  httpGet:
    path: /live
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 10
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 2
```

- **Readiness 프로브**: Pod이 준비되기 전까지 Service 엔드포인트에 **포함되지 않도록** 합니다
- **Liveness 프로브**: 비정상 상태의 Pod을 감지하여 **자동 재시작**을 수행합니다
- **Startup 프로브**: **느린 부팅 시간**을 가진 애플리케이션을 보호합니다(초기화 완료 전 liveness 프로브 오작동 방지)

> "새 Pod이 **Ready 상태**가 되기 전까지는 기존 Pod을 제거하지 않는다"는 원칙이 **무중단 배포의 핵심**입니다.

---

## 배포 및 롤백 기본 명령어

```bash
# 배포 상태 실시간 모니터링

kubectl rollout status deployment my-app

# 배포 이력 확인

kubectl rollout history deployment my-app
kubectl rollout history deployment my-app --revision=3

# 이미지 변경을 통한 배포(간편한 방법)

kubectl set image deployment my-app app=my-app:v2

# 배포 일시정지 및 재개(점진적 검증에 유용)

kubectl rollout pause deployment my-app
kubectl rollout resume deployment my-app

# 즉시 롤백(직전 버전으로 복귀)

kubectl rollout undo deployment my-app

# 특정 리비전으로 롤백

kubectl rollout undo deployment my-app --to-revision=3
```

> **유용한 팁**: `pause` 명령으로 배포를 멈춘 후 모니터링, 검증, 성능 테스트를 진행하고 문제가 없을 때 `resume` 명령으로 배포를 재개할 수 있습니다.

---

## 실전 배포 전략 1 — "보수적" RollingUpdate 프로파일

트래픽에 민감하거나 핵심 서비스에 적합한 안정성 중심의 배포 전략입니다:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 0              # 불필요한 과도한 증설 방지
    maxUnavailable: 1        # 한 번에 1개의 Pod만 비우기 (느리지만 안전한 방식)
```

- 장점: 서비스 수준 목표(SLO) 안정성 보장, 동작 예측 가능
- 단점: 배포 속도가 상대적으로 느림

---

## 실전 배포 전략 2 — "속도 우선" 프로파일

비핵심 서비스, 스테이징 환경, 대량의 마이크로서비스에 적합한 신속 배포 전략입니다:

```yaml
rollingUpdate:
  maxSurge: 30%             # 빠른 증설 허용
  maxUnavailable: 30%       # 빠른 교체 허용
```

- 장점: 배포 속도가 빠름
- 단점: 짧은 순간 처리량 저하 가능성 → **HPA 및 버퍼링 전략**과 함께 고려 필요

---

## Canary 스타일 RollingUpdate (경량 구현)

**전체 트래픽 중 일부만** 새 버전으로 전환하여 문제를 조기 감지하는 방식입니다:

### 두 개의 Deployment를 활용한 단순 Canary 패턴

```yaml
# stable 버전
spec:
  replicas: 8
  template:
    metadata:
      labels:
        track: stable

# canary 버전
spec:
  replicas: 2
  template:
    metadata:
      labels:
        track: canary
```

- Service의 `selector`에 `app=my-app`만 포함 → **stable과 canary가 합산된 트래픽** 분배
- canary 비율을 10~20%로 시작하여 메트릭을 관찰한 후 점진적으로 증가

### 단일 Deployment로 점진적 전환

- **`maxUnavailable: 0`** + 충분한 `maxSurge` 설정: 안정성 유지하며 **추가 Pod로 탐색**
- **readinessGate**(선택적): 외부 컨트롤러가 "품질 승인" 시 Ready 상태로 전환

---

## Blue-Green 배포 전략

**두 개의 환경(Blue=현재 운영, Green=새 버전)**을 병렬로 운영한 후 **한 번에 트래픽을 전환**하는 방식입니다. 다운타임 없이 전환이 가능하며 **즉시 롤백이 간단**합니다.

### 단순한 Service 트래픽 전환

```yaml
# blue와 green은 서로 다른 라벨을 가짐 (예: version: blue/green)
# Service의 selector만 변경하여 트래픽을 일괄 전환

kubectl patch service my-svc -p '{"spec":{"selector":{"app":"my-app","version":"green"}}}'
```

- 장점: 대규모 트래픽 환경에서도 예측 가능한 전환
- 단점: 인프라 이중 운영으로 인한 **비용 증가**

---

## Argo Rollouts를 활용한 Canary/Blue-Green 자동화

**CRD(Custom Resource Definition) 기반**의 고급 롤링 컨트롤러입니다. **메트릭 게이트**, **자동 롤백**, **트래픽 셰이핑** 등을 지원합니다.

### Canary 배포 단계 및 메트릭 게이트 예시

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 10
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:v2
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause:
          duration: 5m
      - analysis:
          templates:
          - templateName: error-rate-check   # Prometheus 등과 연동
      - setWeight: 50
      - pause:
          duration: 10m
      - setWeight: 100
```

- **AnalysisTemplate**을 통해 에러율, 지연 시간 등의 조건을 만족해야 다음 단계 진행
- 조건 위반 시 **자동 롤백** 수행

### Blue-Green 배포 with Preview 서비스

```yaml
strategy:
  blueGreen:
    activeService: my-svc        # 현재 운영 중인 트래픽
    previewService: my-svc-prev  # 검증용 서비스
    autoPromotionEnabled: false  # 수동 승격 방식
```

---

## 배포와 다른 Kubernetes 리소스의 상호작용

### HPA(Horizontal Pod Autoscaler)

- 배포 중 **Pod 증설/감축**이 겹칠 수 있음 → **`maxSurge` 여유 공간**과 **노드 자원/Cluster Autoscaler** 확인 필요
- HPA가 갑자기 다운스케일하면 새 ReplicaSet의 안정화 전 가용성 저하 위험 존재

### PDB(PodDisruptionBudget)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 80%   # 또는 maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```
- 롤링 업데이트, 노드 드레인, 클러스터 업그레이드 시 **동시 중단 가능한 Pod 상한** 보장

### Cluster Autoscaler (노드 자동 확장)

- 큰 `maxSurge` 값은 **임시 노드 확장** 필요 → Cluster Autoscaler 정책과 비용 고려 필요

---

## 실전 예제 — "안전한 점진 배포와 빠른 롤백"

### 통합 매니페스트(배포, 프로브, PDB 포함)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-api
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2              # 임시로 2개 더 배포 (안전성 확보)
      maxUnavailable: 0        # 가용성 완전 보장
  selector:
    matchLabels:
      app: shop-api
  template:
    metadata:
      labels:
        app: shop-api
    spec:
      containers:
      - name: api
        image: registry/shop-api:v2.3.1
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /live
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 10
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: shop-api-pdb
spec:
  minAvailable: 5   # replicas=6, 최소 5개 유지
  selector:
    matchLabels:
      app: shop-api
```

### 배포, 검증, 롤백 실습 시나리오

```bash
# 배포 실행

kubectl apply -f deploy.yaml
kubectl rollout status deployment shop-api

# 에러율 급증 감지 → 즉시 롤백

kubectl rollout undo deployment shop-api
kubectl rollout status deployment shop-api
```

---

## 문제 해결 가이드

**새 Pod이 Ready 상태가 되지 않음**
- **가능한 원인**: 헬스 엔드포인트 구성 오류, 타임아웃 설정 부적절
- **해결 방안**: `startupProbe` 추가, `readinessProbe` 설정 조정

**배포 진행이 멈춤**
- **가능한 원인**: `maxUnavailable=0` 설정 + Pod Ready 상태 전환 실패
- **해결 방안**: 배포 일시정지(`pause`) 후 원인 조사, 임시로 `maxUnavailable` 증가, 코드 롤백

**롤백 후에도 서비스 실패**
- **가능한 원인**: 환경 변수/ConfigMap 롤백 불일치, 데이터베이스 마이그레이션 역호환성 문제
- **해결 방안**: **Config/스키마 호환성** 전략(Expand/Contract 패턴), 기능 플래그 활용

**배포 중 순간적 5xx 에러 증가**
- **가능한 원인**: `maxUnavailable` 값 과도, `readinessProbe` 간격 너무 짧음
- **해결 방안**: `maxUnavailable` 값 감소, 네그레이션 기간 및 웜업 도입

**노드 부족으로 Pod Pending 상태**
- **가능한 원인**: `maxSurge` 값 과도 + Cluster Autoscaler 설정 미비
- **해결 방안**: `maxSurge` 값 감소 또는 Cluster Autoscaler/노드 풀 용량 증가

---

## 데이터베이스/스키마 변경과 롤백 전략

**스키마 변경**이 수반되는 배포의 경우 단순한 롤백이 파괴적일 수 있습니다. 다음과 같은 패턴을 권장합니다:

**Expand → Code Switch → Contract 패턴**
1. **Expand 단계**: 새 컬럼/인덱스 추가(기존 코드와 호환)
2. **Code Switch 단계**: 애플리케이션이 새 스키마 사용 시작
3. **Contract 단계**: 충분한 기간 후 구 스키마 제거

기능 플래그를 활용한 **Dual-Write/Read 기간** 설정을 통해 비가역적 리스크를 감소시킬 수 있습니다.

---

## Helm, GitOps 및 CI/CD 파이프라인 통합

### Helm 값을 통한 배포 전략 주입

```yaml
# values.yaml

rollingUpdate:
  maxSurge: 1
  maxUnavailable: 0
```
{% raw %}
```yaml
# templates/deploy.yaml

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: {{ .Values.rollingUpdate.maxSurge }}
    maxUnavailable: {{ .Values.rollingUpdate.maxUnavailable }}
```
{% endraw %}

### GitOps(Argo CD/Flux) 환경에서의 배포 관리

- `rollout pause/resume` 명령을 PR 리뷰 및 승인 프로세스와 연동
- **메트릭 게이트(Prometheus 쿼리 기반)** → **자동 승격/자동 롤백** 파이프라인 구현

---

## 모니터링 메트릭과 SLO 게이트

**배포 단계별로** 다음과 같은 메트릭을 지속적으로 모니터링해야 합니다:
- **에러율**(HTTP 5xx RPS, 에러 비율)
- **지연 시간**(p95/p99 백분위수)
- **리소스 포화도**(CPU 스로틀링, 컨테이너 OOM, GC 시간)
- **Readiness 전환 시간**(새 Pod이 Ready 상태까지 걸리는 시간 분포)

권장 임계치 예시:
- 에러율 상승률 \(\Delta \text{error\_rate} > 1\%\)
- p99 지연 시간 \(> 800\text{ ms}\)
- 새 Pod Ready 시간 p95 \(> 60\text{ s}\) → **자동 중단/롤백** 트리거

---

## 운영 모범 사례 요약

1. **헬스 프로브 우선 튜닝**: Startup → Readiness → Liveness 순서로 설정 검토
2. **보수적 시작**: `maxUnavailable: 0`, 적절한 `maxSurge` 값으로 시작
3. **점진적 조정**: 배포 속도 필요 시 `maxSurge`/`maxUnavailable` 비율 점진적 조정
4. **다른 리소스와의 충돌 사전 확인**: PDB, HPA, Cluster Autoscaler와의 상호작용 검토
5. **중요 릴리스에는 Canary/Blue-Green 전략 활용**
6. **메트릭 기반 자동화**: Argo Rollouts 등의 도구를 통한 자동 승격/롤백 구현
7. **롤백 시나리오 사전 준비**: 문서화 및 리허설(Chaos Engineering/게임데이)
8. **데이터베이스 스키마 관리**: Expand/Contract 패턴 적용, 기능 플래그 활용
9. **장애 발생 시 신속한 대응**: 배포 일시정지(`pause`), 원인 분석, 필요시 신속한 롤백

---

## Recreate 전략(완전 재시작)

```yaml
strategy:
  type: Recreate     # 모든 기존 Pod 종료 후 새 버전 기동
```
- 장점: 상태 불일치 최소화(단일 리더 등 특수한 경우 적합)
- 단점: **일시적 서비스 중단** 발생 → 게이트웨이 수준의 버퍼링/재시도 전략 필요

---

## 배포 후 검증을 위한 Job/Probe 패턴

새 버전 배포 후 **자동화된 스모크 테스트** 수행:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: post-deploy-check
spec:
  template:
    spec:
      containers:
      - name: smoke
        image: curlimages/curl
        args: ["sh","-c","curl -sf http://my-app/health && curl -sf http://my-app/smoke"]
      restartPolicy: Never
```

- 테스트 실패 시 알림 발생 → `rollout undo` 명령 실행

---

## 결론

Kubernetes의 Rolling Update는 안전한 무중단 배포를 위한 강력한 기본 전략이지만, 그 효과는 **파라미터 설정, 헬스체크 구성, SLO 게이트 적용**에 크게 의존합니다. 단순한 기술적 구현을 넘어 운영 관점에서의 철저한 계획과 검증이 필요합니다.

**롤백** 기능은 단순한 "비상용 옵션"이 아니라 **배포 전략의 필수 구성 요소**로 인식해야 합니다. 언제든 안전하게 되돌아갈 수 있는 경로를 항상 확보하는 것이 현명한 운영의 시작점입니다.

중요한 릴리스의 경우 **Canary/Blue-Green 배포 전략**과 **메트릭 기반 자동화**를 결합하여 위험을 최소화하면서 신속한 배포를 실현할 수 있습니다. 이러한 다층적 접근 방식을 통해 비즈니스 요구사항과 기술적 제약 사이의 최적 균형점을 찾을 수 있습니다.

배포 프로세스는 단순한 기술적 작업이 아닌 조직의 **신뢰성 문화와 운영 성숙도**를 반영하는 거울입니다. 체계적인 계획, 철저한 테스트, 지속적인 개선을 통해 안정적이고 효율적인 배포 문화를 구축해 나가시기 바랍니다.

---

## 참고 자료 및 명령어 요약

- 주요 명령어: `kubectl rollout status|history|undo|pause|resume`
- 이미지 업데이트: `kubectl set image deployment <디플로이먼트명> <컨테이너명>=<이미지>:<태그>`
- 공식 문서: Deployment RollingUpdate, Rollout, Probes, PDB, HPA, Argo Rollouts