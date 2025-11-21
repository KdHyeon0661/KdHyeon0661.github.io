---
layout: post
title: Kubernetes - Rolling Update & Rollback
date: 2025-05-28 21:20:23 +0900
category: Kubernetes
---
# Kubernetes 배포 전략: Rolling Update & Rollback

## Rolling Update — 원리와 수학적 직관

### 동작 원리

1. 기존 `ReplicaSet(old)` 유지
2. 새 템플릿으로 `ReplicaSet(new)` 생성
3. **최대 허용 증설량**(`maxSurge`)만큼 새 Pod 추가
4. **최대 허용 가용성 감소**(`maxUnavailable`) 범위에서 구(舊) Pod 제거
5. 새 RS가 목표 `replicas`만큼 차오르면 완료

### 핵심 파라미터

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
      maxSurge: 1           # 정수 | % (기본: 25%)
      maxUnavailable: 1     # 정수 | % (기본: 25%)
  selector:
    matchLabels: { app: my-app }
  template:
    metadata:
      labels: { app: my-app }
    spec:
      containers:
      - name: app
        image: my-app:v2
        ports: [{containerPort: 8080}]
```

- `maxSurge`: **임시로 늘려도 되는** Pod 수
- `maxUnavailable`: 배포 중 **잠시 비워도 되는** Pod 수
- 정수/퍼센트 혼용 가능(반올림 규칙에 유의)

### 가용성 계산 (직관)

배포 중 **동시 운영** 가능한 총 Pod 상한은
$$
\text{maxPods} = \text{replicas} + \text{maxSurge}
$$
동시에 **비활성** 상태일 수 있는 하한은
$$
\text{minAvailable} = \text{replicas} - \text{maxUnavailable}
$$
서비스 지연을 최소화하려면
$$
\text{minAvailable} \approx \text{SLO 요구 QPS 대비 필요 Pod 수}
$$
를 항상 만족시키도록 잡아야 합니다.

> 예: `replicas=4`, `maxSurge=1`, `maxUnavailable=1` → **3~5개** 사이로 이동하며 교체.

---

## 배포 헬스체크: Readiness/Liveness/Startup

**헬스 프로브**는 롤링의 품질을 좌우합니다.

```yaml
readinessProbe:
  httpGet: {path: /healthz, port: 8080}
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
livenessProbe:
  httpGet: {path: /live, port: 8080}
  initialDelaySeconds: 20
  periodSeconds: 10
startupProbe:
  httpGet: {path: /startup, port: 8080}
  failureThreshold: 30
  periodSeconds: 2
```

- **Readiness**: 준비되기 전엔 Service 엔드포인트에 **편입 금지**
- **Liveness**: 비정상 루프 시 **재시작**
- **Startup**: **느린 부팅** 보호(초기화 끝나기 전 liveness 오작동 방지)

> “새 Pod이 **Ready** 되기 전엔 구 Pod을 빼지 않는다” → **무중단**의 핵심.

---

## 배포·롤백 명령 모음(기본기 철저)

```bash
# 배포 상태 모니터링

kubectl rollout status deploy my-app

# 현재/과거 히스토리

kubectl rollout history deploy my-app
kubectl rollout history deploy my-app --revision=3

# 이미지 교체(원클릭 배포)

kubectl set image deploy my-app app=my-app:v2

# 일시정지/재개(Incremental 확인·검증에 유용)

kubectl rollout pause deploy my-app
kubectl rollout resume deploy my-app

# 즉시 롤백(직전)

kubectl rollout undo deploy my-app

# 특정 리비전으로 롤백

kubectl rollout undo deploy my-app --to-revision=3
```

> **팁**: `pause`로 멈춘 뒤 모니터링·검증·성능 테스트 → 문제 없으면 `resume`.

---

## 실전 전략 1 — “보수적” RollingUpdate 프로파일

트래픽 민감·코어 서비스에 추천:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 0              # 불필요한 과증설 금지
    maxUnavailable: 1        # 한번에 1개씩만 비우기 (느리지만 안전)
```

- 장점: SLO 안정, 예측 가능
- 단점: 배포 속도 느림

---

## 실전 전략 2 — “속도 우선” 프로파일

비핵심 서비스·스테이징·대량 마이크로서비스에:

```yaml
rollingUpdate:
  maxSurge: 30%             # 빠른 증설
  maxUnavailable: 30%       # 빠른 교체
```

- 장점: 빠름
- 단점: 짧은 순간 처리량 저하 가능 → **HPA·버퍼링** 고려

---

## Canary 스타일 RollingUpdate(경량 버전)

**전체 트래픽 중 일부만** 새 버전에 흘려 문제를 감지:

### 두 개의 Deployment로 단순 Canary

```yaml
# stable

spec: {replicas: 8, template: {metadata: {labels:{track: stable}}}}

# canary

spec: {replicas: 2, template: {metadata: {labels:{track: canary}}}}
```

- Service의 `selector`에 `app=my-app`만 포함 → **stable+canary 합산 트래픽** 분배
- canary 비율을 10~20%로 시작 → 메트릭 관찰 후 증량

### 단일 Deployment로 점진(서지/언어버블 튜닝)

- **`maxUnavailable: 0`** + 충분한 `maxSurge`: 안정성 유지하며 **추가 Pod로 탐색**
- **readinessGate**(옵션): 외부 컨트롤러가 “품질 승인” 시 Ready로 전환

---

## — Service 전환

**두 환경(Blue=현재, Green=신규)**를 병렬 운영 후 **한 번에 스위치**. 다운타임 없이 전환이 가능하고 **즉시 롤백이 간단**.

### 단순 Service 스위치

```yaml
# blue와 green은 서로 다른 label (e.g., version: blue/green)
# Service selector만 바꿔서 트래픽을 일괄 전환

kubectl patch svc my-svc -p '{"spec":{"selector":{"app":"my-app","version":"green"}}}'
```

- 장점: 대규모 트래픽에도 예측 가능
- 단점: 인프라 이중 운영 → **비용↑**

---

## Argo Rollouts로 Canary/BlueGreen 자동화

**CRD 기반**의 고급 롤링 컨트롤러. **메트릭 게이트**, **자동 롤백**, **트래픽 셰이핑** 지원.

### Canary with steps & metric gate 예시

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: my-app }
spec:
  replicas: 10
  selector: { matchLabels: { app: my-app } }
  template:
    metadata: { labels: { app: my-app } }
    spec:
      containers:
      - name: app
        image: my-app:v2
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 5m}
      - analysis:
          templates:
          - templateName: error-rate-check   # Prometheus 등과 연동
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
```

- **AnalysisTemplate**로 에러율/지연 시간 조건을 만족해야 다음 단계 진행
- 조건 위반 시 **자동 롤백**

### BlueGreen with preview

```yaml
strategy:
  blueGreen:
    activeService: my-svc        # 현재 운영 트래픽
    previewService: my-svc-prev  # 검증용
    autoPromotionEnabled: false  # 수동 승격
```

---

## 배포와 다른 리소스의 상호작용

### HPA(수평 오토스케일러)

- 배포 중 **증설/감축**이 겹칠 수 있음 → **`maxSurge` 여유** vs **노드 자원/CA** 확인
- HPA가 갑자기 다운스케일하면 새 RS 안정화 전 가용성 저하 위험

### PDB(PodDisruptionBudget)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: my-app-pdb }
spec:
  minAvailable: 80%   # or maxUnavailable: 1
  selector: { matchLabels: { app: my-app } }
```
- 롤링+노드 드레인+업그레이드 시 **동시 중단 상한** 보장

### Cluster Autoscaler(노드 자동 확장)

- `maxSurge`가 크면 **임시 노드 확장** 필요 → CA 정책과 코스트 주의

---

## 실전 예제 — “안전한 점진 배포 + 빠른 롤백”

### 매니페스트(배포·프로브·PDB 포함)

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
      maxSurge: 2              # 임시 2개 더 띄움 (안전)
      maxUnavailable: 0        # 가용성 확보
  selector:
    matchLabels: { app: shop-api }
  template:
    metadata:
      labels: { app: shop-api }
    spec:
      containers:
      - name: api
        image: registry/shop-api:v2.3.1
        ports: [{containerPort: 8080}]
        readinessProbe:
          httpGet: {path: /healthz, port: 8080}
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet: {path: /live, port: 8080}
          initialDelaySeconds: 20
          periodSeconds: 10
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: shop-api-pdb
spec:
  minAvailable: 5   # replicas=6, min 5개 유지
  selector:
    matchLabels: { app: shop-api }
```

### 배포·검증·롤백 시나리오

```bash
# 배포

kubectl apply -f deploy.yaml
kubectl rollout status deploy shop-api

# 에러율↑ 감지 → 즉시 롤백

kubectl rollout undo deploy shop-api
kubectl rollout status deploy shop-api
```

---

## 트러블슈팅 체크리스트

| 증상 | 원인 후보 | 해결 |
|---|---|---|
| 새 Pod이 Ready 안 됨 | 헬스 엔드포인트/타임아웃 부적절 | `startupProbe` 추가, `readinessProbe` 조정 |
| 배포가 멈춤 | `maxUnavailable=0` + Ready 불가 | `pause`→조사, 임시로 `maxUnavailable`↑, 코드 롤백 |
| 롤백했는데도 실패 | 환경 변수/Config 롤백 안 맞음, DB 마이그레이션 역호환X | **Config/Schema 호환성** 전략(Expand/Contract), 피처 플래그 |
| 순간 5xx 증가 | `maxUnavailable` 과대, `readinessProbe` 빠름 | `maxUnavailable`↓, 네그레이스 기간·웜업 도입 |
| 노드 부족으로 Pending | `maxSurge` 과대 + CA 설정 미흡 | `maxSurge`↓ 또는 CA/노드풀 용량↑ |

---

## 데이터베이스/스키마와 롤백

**스키마 변경**이 수반되면 단순 롤백이 파괴적일 수 있습니다.
권장 패턴: **Expand → Code Switch → Contract**
1. **Expand**: 새 컬럼/인덱스 추가(구 코드와 호환)
2. **Code Switch**: 앱이 새 스키마 사용 시작
3. **Contract**: 충분한 기간 후 구 스키마 제거

피처 플래그로 **Dual-Write/Read** 기간을 두면 비가역 리스크 감소.

---

## Helm·GitOps·파이프라인 팁

### Helm 값으로 전략 주입

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

### GitOps(Argo CD/Flux)

- `rollout pause/resume`를 PR 리뷰·승인과 연동
- **메트릭 게이트(프로메테우스 쿼리)** → **자동 승격/자동 롤백** 파이프라인

---

## 모니터링 메트릭과 SLO 게이트

**배포 단계별**로 다음을 감시:
- **에러율**(HTTP 5xx RPS, 비율)
- **지연 시간**(p95/p99)
- **Saturation**(CPU Throttle, 컨테이너 OOM, GC 시간)
- **Readiness 전환 시간**(새 Pod이 Ready까지 걸린 시간 분포)

임계치(예):
- 에러율 상승률 \(\Delta \text{error\_rate} > 1\%\)
- p99 지연 \(> 800\text{ ms}\)
- 새 Pod Ready 시간 p95 \(> 60\text{ s}\) → **자동 중단/롤백**

---

## 운영 레시피 요약

1. **헬스 프로브**를 먼저 튜닝(Startup → Readiness → Liveness)
2. **보수적 시작**: `maxUnavailable: 0`, 작은 `maxSurge`
3. **점진 상향**: 배포속도 필요 시 `maxSurge`/`maxUnavailable` 비율 조정
4. **PDB/HPA/CA**와 충돌 없는지 사전 점검
5. **Canary/BlueGreen**은 **중요 릴리스**에서 사용
6. **메트릭 게이트**로 자동화(Argo Rollouts 등)
7. **롤백 시나리오**를 **사전에** 문서화·리허설(Chaos/게임데이)
8. **DB 스키마**는 Expand/Contract 전개, 피처 플래그로 가드
9. 장애 시 `pause`하고 **원인규명 → 롤백**을 망설이지 말 것

---

## Recreate 전략(완전 재시작)

```yaml
strategy:
  type: Recreate     # 모든 Pod 종료 후 새 버전 기동
```
- 장점: 상태 불일치 최소화(단일 리더 등)
- 단점: **잠깐 중단** 발생 → 게이트웨이 레벨 버퍼링/재시도 필요

---

## 배포 검증용 Job/Probe 패턴

새 버전이 뜬 뒤 **사내 Synthetic 테스트**를 자동 수행:

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: post-deploy-check }
spec:
  template:
    spec:
      containers:
      - name: smoke
        image: curlimages/curl
        args: ["sh","-c","curl -sf http://my-app/health && curl -sf http://my-app/smoke"]
      restartPolicy: Never
```

- 실패 시 알람 → `rollout undo`

---

## 결론

- **Rolling Update**는 안전한 기본값이지만, **파라미터·헬스체크·SLO 게이트**가 품질을 결정합니다.
- **Rollback**은 “옵션”이 아니라 **전략의 일부**입니다. *항상* 되돌릴 길을 남겨두세요.
- 중요 릴리스는 **Canary/Blue-Green** + **메트릭 기반 자동화**로 **위험을 금전화(金轉化)** 하십시오.

---

## 참고 명령·링크(요약)

- `kubectl rollout status|history|undo|pause|resume`
- `kubectl set image deploy X app=image:tag`
- 공식 문서: Deployment RollingUpdate, Rollout, Probes, PDB, HPA, Argo Rollouts
