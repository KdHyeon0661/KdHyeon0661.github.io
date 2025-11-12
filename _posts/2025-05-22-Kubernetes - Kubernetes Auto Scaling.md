---
layout: post
title: Kubernetes - Kubernetes Auto Scaling
date: 2025-05-22 22:20:23 +0900
category: Kubernetes
---
# Kubernetes Auto Scaling

## 0. 큰 그림: 레이어별 오토스케일

```
요청 트래픽/큐 -> 메트릭 수집(Metrics Server/Prometheus/KEDA)
                         │
               ┌─────────┴─────────┐
               │                   │
           HPA (Pod 수)        VPA (Pod 리소스)
               │                   │
               └─────(스케줄 불가 시)──────┐
                                           ▼
                                 Cluster Autoscaler (노드 수)
```

- **HPA**: _Pod 개수_ 자동 조절 (CPU/메모리/외부 메트릭).
- **VPA**: _컨테이너 requests/limits_ 자동 추천/적용.
- **Cluster Autoscaler**: _노드_ 자동 증가/감소.

---

## 1. 전제 조건 체크리스트

| 항목 | 왜 중요한가 | 빠른 확인 |
|---|---|---|
| Metrics Server 설치 | HPA/VPA 기본 메트릭 원천 | `kubectl top nodes/pods` 작동 여부 |
| 리소스 requests 정의 | HPA의 CPU/메모리 **Utilization** 계산의 분모 | Pod 스펙에 `resources.requests` 확인 |
| 타임소스/시계 동기화 | 메트릭 타임스탬프 왜곡 방지 | NTP/Chrony |
| PodDisruptionBudget(PDB) | 축소/노드 제거 시 가용성 유지 | HPA/CA와 동작 충돌 방지 |
| Readiness/Liveness probe | 스케일 직후 품질 보장 | 워밍업과 합치기 |

---

## 2. HPA (Horizontal Pod Autoscaler) — 수평 확장

### 2.1 기본 흐름과 핵심 개념

- HPA는 주기적으로 목표 워크로드의 메트릭을 읽고 `desiredReplicas`를 계산해 **Replica 수**를 변경.
- CPU/메모리(리소스 메트릭) 외에 **외부/객체 메트릭**(Prometheus Adapter, CloudWatch 등) 기반도 가능.
- 안정화(안티 플랩): _scaleUp/Down_ 시 **안정화 창**, **정책**(rate-limit) 적용.

#### HPA 목표값 계산(리소스 메트릭, averageUtilization)
메트릭 정의가 `averageUtilization = U_target(%)`일 때,  
현재 평균 사용률 \(U_{\text{current}}\) 과 현재 레플리카 \(R_{\text{current}}\) 에 대해:

$$
R_{\text{desired}} = R_{\text{current}} \times \frac{U_{\text{current}}}{U_{\text{target}}}
$$

- \(U_{\text{current}} = \frac{\text{sum(각 Pod 사용량)}}{\text{sum(각 Pod request)}} \times 100\)

> CPU 기준 예: 현재 평균 80%, 목표 50%면 `80/50=1.6` → 60% 증설.

### 2.2 필수 스펙 (autoscaling/v2)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
  namespace: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deploy
  minReplicas: 2
  maxReplicas: 10
  behavior:                          # 안정화/속도 제어(권장)
    scaleUp:
      stabilizationWindowSeconds: 0  # 빠른 확장
      policies:
      - type: Percent
        value: 100                   # 1주기 최대 100% 증가
        periodSeconds: 30
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300  # 5분 관성, 과감한 축소 방지
      policies:
      - type: Percent
        value: 20                      # 1주기 최대 20% 감소
        periodSeconds: 60
      selectPolicy: Min
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### 2.3 여러 메트릭 동시 사용

- HPA는 각 메트릭별 desiredReplicas를 계산 후 **가장 큰 값**을 선택(보수적).
- 예: CPU 50%, 메모리 70%, 큐 길이 100개 기준 → 셋 모두 계산 후 **최대치** 적용.

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 60
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 70
- type: External
  external:
    metric:
      name: queue_messages_ready
      selector:
        matchLabels:
          queue: orders
    target:
      type: AverageValue
      averageValue: "100"
```

### 2.4 커스텀/외부 메트릭: Prometheus Adapter

1) 어댑터 설치(개념):

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install metrics-server metrics-server/metrics-server -n kube-system

helm repo add prometheus-adapter https://kubernetes-sigs.github.io/prometheus-adapter
helm install prom-adapter prometheus-adapter/prometheus-adapter -n monitoring \
  --set rules.default=true
```

2) 매핑 룰 예(큐 길이 지표를 External로 노출):

```yaml
# values 일부 예시
rules:
  default: false
  custom:
  - seriesQuery: 'rabbitmq_queue_messages_ready{namespace!="",queue!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "rabbitmq_queue_messages_ready"
      as: "queue_messages_ready"
    metricsQuery: 'sum(rate(rabbitmq_queue_messages_ready[1m])) by (namespace,queue)'
```

3) HPA에서 사용:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orders-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orders
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: External
    external:
      metric:
        name: queue_messages_ready
        selector:
          matchLabels:
            queue: orders
      target:
        type: AverageValue
        averageValue: "200"
```

### 2.5 KEDA(대안/보완)

- 이벤트/큐 기반 스케일(제로까지도): Kafka, RabbitMQ, SQS, HTTP 등 60+ 스케일러.
- **HPA를 내부적으로 생성**해줌. **버스트 부하**에 빠르게 대응.

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda -n keda --create-namespace
```

간단 스케일러 예:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: orders-keda
spec:
  scaleTargetRef:
    name: orders
  minReplicaCount: 0
  maxReplicaCount: 50
  triggers:
  - type: rabbitmq
    metadata:
      queueName: orders
      hostFromEnv: RABBITMQ_CONN_STR
      value: "100"     # 큐 길이 목표
```

---

## 3. VPA (Vertical Pod Autoscaler) — 수직 확장

### 3.1 개념/전략
- 컨테이너의 `requests/limits`를 **추천**하고(Off/Initial), **자동 반영**(Auto) 가능.
- 반영 시 **Pod 재시작**이 필요. 무중단 레이어(롤링/레디니스/피어)와 조합 필수.

### 3.2 모드

| updateMode | 의미 | 권장 사용 |
|---|---|---|
| `Off` | 추천만 제공 | HPA와 병행 시 충돌 회피, 대다수 실무 |
| `Initial` | 최초 생성 시 추천값 반영 | 베이스라인 자동화 |
| `Auto` | 동적으로 반영(재시작 수반) | 비상태/탄력적 워크로드, 야간 윈도우 |

### 3.3 설치/예시

```bash
# 최신 릴리즈 경로 확인 후 적용
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/vertical-pod-autoscaler-<ver>/deploy/vpa-crd.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/vertical-pod-autoscaler-<ver>/deploy/recommender.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/vertical-pod-autoscaler-<ver>/deploy/updater.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/vertical-pod-autoscaler-<ver>/deploy/admission-controller.yaml
```

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
  namespace: web
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deploy
  updatePolicy:
    updateMode: "Off"   # 실무 기본: 추천만
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: "200m"
        memory: "256Mi"
      maxAllowed:
        cpu: "4"
        memory: "8Gi"
```

> 실무 패턴: **VPA(Off)로 추천 → HPA는 레플리카 조절**. 야간 점검 윈도우에 추천을 배치에 반영.

---

## 4. Cluster Autoscaler — 노드 자동 확장

### 4.1 동작 개요
- 새 Pod이 **스케줄 불가**(자원 부족) → 노드 증가.
- 노드가 **유휴**(빈/축출 가능) → 노드 감소 (PodDisruptionBudget/PDB 존중).

### 4.2 전제
- 클라우드 매니지드: GKE/EKS/AKS NodeGroup/ASG.
- 온프레: Karpenter/Cluster API 등 대안 고려.

### 4.3 설치(Helm 예시: EKS)

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler -n kube-system \
  --set autoDiscovery.clusterName=my-eks \
  --set awsRegion=ap-northeast-2 \
  --set rbac.serviceAccount.create=true \
  --set rbac.serviceAccount.name=cluster-autoscaler
```

**중요 옵션**
- **태인트/토폴로지/스팟** 고려: 스케줄 불가 원인과 노드풀 매칭이 가능해야 함.
- `--balance-similar-node-groups`, `--expander=least-waste` 등 정책.

---

## 5. 레이어 조합 전략

| 시나리오 | 권장 조합 | 메모 |
|---|---|---|
| 웹/API 버스트 | **HPA + CA** | 빠른 Pod 확장 + 노드 보충 |
| 배치/큐 처리 | **KEDA(+HPA) + CA**, VPA(Off) | 외부 큐 메트릭 기반 |
| 메모리 민감/GC 압박 | **VPA(Off)**로 sizing 개선 + HPA | 권장치 반영 주기적 |
| 비용 최적화 | HPA + CA + 스케일다운 적극화 | 안정화 창/쿨다운 튜닝 |
| 고가용성 | HPA + PDB + 분산 토폴로지 | CA 축소 시에도 가용성 확보 |

**HPA & VPA 충돌 주의**  
- HPA: replicas 조절, VPA: requests 조절 → 둘 다 동시에 **Active 변경** 시 **목표 분모가 변해 불안정**.  
- 보편 해법: **VPA=Off/Initial**, HPA=Active.

---

## 6. 운영 튜닝: Behavior/쿨다운/워밍업

- **stabilizationWindowSeconds**: 급격한 변동 억제(특히 scaleDown).
- **policies**: 분당/퍼센트 상한으로 과잉 확장/축소 방지.
- **워밍업**: 새 Pod 가용까지 평균 지표에 반영 지연 → **readinessProbe**, **preStop**로 안전.
- **빈번한 GC/스파이크**: 메모리 목표는 보수적, scaleDown 안정화 창 늘리기.

---

## 7. 실전: 샘플 애플리케이션 스케일링

### 7.1 예제 디플로이 + HPA + PDB

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels: { app: echo }
  template:
    metadata:
      labels: { app: echo }
    spec:
      containers:
      - name: server
        image: ealen/echo-server:latest
        ports: [{containerPort: 80}]
        resources:
          requests: { cpu: "200m", memory: "256Mi" }
          limits:   { cpu: "1",    memory: "512Mi" }
        readinessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 5
          periodSeconds: 3
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: echo-hpa
  namespace: demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: echo
  minReplicas: 2
  maxReplicas: 20
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 20
        periodSeconds: 60
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: echo-pdb
  namespace: demo
spec:
  minAvailable: 1
  selector:
    matchLabels: { app: echo }
```

### 7.2 부하 유발(간단)

```bash
kubectl run -it --rm load --image=busybox --restart=Never -- /bin/sh -c \
 'while true; do wget -q -O- http://echo.demo.svc.cluster.local; done'
```

### 7.3 관측/검증 포인트

- `kubectl get hpa -n demo -w`
- `kubectl top pods -n demo`
- Grafana 대시보드에서 CPU/레플리카/응답지연 추적.

---

## 8. 수학/알고리즘 디테일

### 8.1 리소스 메트릭(평균 활용률) 공식

$$
U_{\text{avg}} = \frac{\sum_{i=1}^{N} \text{usage}_i}{\sum_{i=1}^{N} \text{request}_i} \times 100
\quad,\quad
R_{\text{desired}} = R_{\text{current}} \times \frac{U_{\text{avg}}}{U_{\text{target}}}
$$

### 8.2 외부 메트릭(평균 값) 공식

외부 메트릭 목표가 `AverageValue = V_target` 일 때:

$$
R_{\text{desired}} = \left\lceil \frac{\sum_{i=1}^{N} \text{metric}_i}{V_{\text{target}}} \right\rceil
$$

> 예: 큐 길이 합이 2,000, 타겟 200 → 10개 레플리카.

---

## 9. 테스트 전략/시나리오

| 시나리오 | 목표 | 방법 |
|---|---|---|
| 버스트 트래픽 | scaleUp 속도/정책 확인 | `hey -z 2m -q 200 -c 100 <svc-url>` |
| 롱테일 트래픽 | scaleDown 안정성 | 안정화 창/정책 튜닝 |
| GC 스파이크 | 메모리 목표/워밍업 검증 | p99 레이턴시/GC 시간 모니터링 |
| 큐 기반 | 외부 메트릭 정확도 | 큐 길이/처리율 상관 검증 |
| 노드 축소 | PDB/축출 안전성 | CA 로그/이벤트 검토 |

---

## 10. 트러블슈팅 표

| 증상 | 원인 후보 | 해결 가이드 |
|---|---|---|
| `kubectl top` 불가 | Metrics Server 미설치/TLS | 재설치/args(`--kubelet-insecure-tls`) |
| HPA 작동 없음 | requests 미설정/네임스페이스 불일치 | Pod 스펙/권한/타깃 ref 확인 |
| 확장 느림 | 안정화 창/정책 너무 보수적 | `behavior.scaleUp` 완화 |
| 축소 과격 | stabilizationWindow 부족 | scaleDown 창 확대/percent 완화 |
| 외부 메트릭 0 | 어댑터 쿼리/라벨 불일치 | PromQL/selector/네임스페이스 매핑 수정 |
| CA 미확장 | 스케줄 불가가 아님(리소스 외 원인) | 토폴로지/taint/affinity 점검 |
| CA 축소 실패 | PDB/DaemonSet/바인딩 Pod | 축출가능성/분산 확인 |

---

## 11. 보안/거버넌스 포인트

- **리소스 요청 표준화**(팀별 가이드/최소값).
- **네임스페이스/라벨링** 일관성: HPA/어댑터 셀렉터가 명확히 매칭되도록.
- **권한**: HPA/어댑터/메트릭스 접근에 필요한 RBAC 최소화.
- **감사/변경관리**: HPA/VPA/CA 파라미터는 변경 시 영향도가 큼 → PR 게이트/승인.

---

## 12. 운영 예시 플레이북(요약)

1) 메트릭 소스 준비(Metrics Server/Prometheus).  
2) 워크로드에 **requests/limits** 정의.  
3) **HPA(v2)** 배치 + **behavior** 설정.  
4) 큐/외부 메트릭이면 **Prometheus Adapter**(또는 **KEDA**)로 노출.  
5) **PDB**/**RollingUpdate** 전략 조율.  
6) **Cluster Autoscaler** 정책/노드풀 용량 확인.  
7) Grafana/Alerting으로 **p95/레플리카/큐 길이** 경보.  
8) 부하 테스트 → 정책 미세조정.

---

## 13. 부록: 실무형 템플릿 모음

### 13.1 CPU+메모리+큐 혼합 HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: svc-hpa
  namespace: prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: svc
  minReplicas: 3
  maxReplicas: 60
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 200
        periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 600
      policies:
      - type: Percent
        value: 15
        periodSeconds: 60
      selectPolicy: Min
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 60 }
  - type: Resource
    resource:
      name: memory
      target: { type: Utilization, averageUtilization: 70 }
  - type: External
    external:
      metric:
        name: http_requests_inflight
      target:
        type: AverageValue
        averageValue: "120"
```

### 13.2 VPA 추천 + 범위 제한

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: svc-vpa
  namespace: prod
spec:
  targetRef: { apiVersion: apps/v1, kind: Deployment, name: svc }
  updatePolicy: { updateMode: "Off" }
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      controlledResources: ["cpu", "memory"]
      minAllowed: { cpu: "300m", memory: "512Mi" }
      maxAllowed: { cpu: "6", memory: "12Gi" }
```

### 13.3 KEDA HTTP(게이트웨이) 스케일 아웃(예시)

```yaml
apiVersion: keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: web-keda
  namespace: prod
spec:
  host: web.prod.example.com
  scaleTargetRef:
    deployment: web
  replicas:
    min: 0
    max: 100
  targetPendingRequests: 100
```

---

## 14. 마무리

- **HPA**는 레플리카, **VPA**는 리소스 사이징, **Cluster Autoscaler**는 노드 용량을 맡는다.
- 기본은 **HPA + Cluster Autoscaler**, 사이징 개선은 **VPA(Off/Initial)** 로 **추천→반영 루프**.
- 큐/이벤트 중심이면 **KEDA**가 생산적.
- 안정적 스케일링은 **behavior(정책/안정화 창)**, **PDB/프로브/롤링 전략**, **관측/알림**이 좌우한다.
- 수학/정책/운영·테스트를 결합해 **비용·성능·안정성**의 균형점을 찾자.

---

## 참고 링크

- HPA: <https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/>
- VPA: <https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler>
- Cluster Autoscaler: <https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler>
- metrics-server: <https://github.com/kubernetes-sigs/metrics-server>
- prometheus-adapter: <https://github.com/kubernetes-sigs/prometheus-adapter>
- KEDA: <https://keda.sh/>