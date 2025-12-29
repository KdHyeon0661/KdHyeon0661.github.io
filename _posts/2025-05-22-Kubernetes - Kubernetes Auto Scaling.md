---
layout: post
title: Kubernetes - Kubernetes Auto Scaling
date: 2025-05-22 22:20:23 +0900
category: Kubernetes
---
# Kubernetes Auto Scaling

## 개요: 레이어별 오토스케일링

Kubernetes에서 오토스케일링은 세 가지 주요 레이어로 구성됩니다:

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

- **HPA(Horizontal Pod Autoscaler)**: _Pod 개수_ 자동 조절 (CPU/메모리/외부 메트릭 기반).
- **VPA(Vertical Pod Autoscaler)**: _컨테이너 requests/limits_ 자동 추천 및 적용.
- **Cluster Autoscaler**: _노드_ 자동 증가 및 감소.

---

## 전제 조건 확인

오토스케일링을 성공적으로 구현하기 위해서는 몇 가지 필수 조건을 확인해야 합니다:

- **Metrics Server 설치**: HPA/VPA의 기본 메트릭 원천으로, `kubectl top nodes/pods` 명령이 정상 작동하는지 확인합니다.
- **리소스 requests 정의**: HPA가 CPU/메모리 사용률(Utilization)을 계산하기 위한 분모로 Pod 스펙에 `resources.requests`가 명시되어 있어야 합니다.
- **시간 동기화**: 메트릭 타임스탬프 왜곡을 방지하기 위해 NTP/Chrony를 통한 시스템 시간 동기화가 필요합니다.
- **PodDisruptionBudget(PDB) 구성**: 축소 또는 노드 제거 시 가용성을 유지하며 HPA/Cluster Autoscaler와의 동작 충돌을 방지합니다.
- **Readiness/Liveness 프로브 설정**: 스케일링 직후 서비스 품질을 보장하기 위해 워밍업과 함께 구성합니다.

---

## 수평 확장(HPA)

### 기본 개념과 동작 원리

HPA는 주기적으로 목표 워크로드의 메트릭을 읽고 `desiredReplicas`를 계산하여 **Replica 수**를 변경합니다. CPU 및 메모리(리소스 메트릭) 외에도 **외부 메트릭 및 객체 메트릭**(Prometheus Adapter, CloudWatch 등)을 기반으로 동작할 수 있습니다. 안정성을 위해 _scaleUp/Down_ 시 **안정화 창(stabilization window)**과 **정책(rate-limit)**이 적용됩니다.

#### HPA 목표값 계산 방식(리소스 메트릭, averageUtilization)

메트릭 정의가 `averageUtilization = U_target(%)`일 경우, 현재 평균 사용률 \(U_{\text{current}}\)과 현재 레플리카 수 \(R_{\text{current}}\)에 대해:

$$
R_{\text{desired}} = R_{\text{current}} \times \frac{U_{\text{current}}}{U_{\text{target}}}
$$

- \(U_{\text{current}} = \frac{\text{각 Pod 사용량 합계}}{\text{각 Pod request 합계}} \times 100\)

> CPU 기준 예시: 현재 평균 사용률이 80%이고 목표가 50%인 경우 `80/50=1.6` → 60% 증가.

### 필수 HPA 스펙 (autoscaling/v2)

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
  behavior:                          # 안정화 및 속도 제어(권장)
    scaleUp:
      stabilizationWindowSeconds: 0  # 빠른 확장
      policies:
      - type: Percent
        value: 100                   # 한 주기 동안 최대 100% 증가
        periodSeconds: 30
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300  # 5분 관성, 과도한 축소 방지
      policies:
      - type: Percent
        value: 20                      # 한 주기 동안 최대 20% 감소
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

### 다중 메트릭 동시 사용

HPA는 각 메트릭별로 desiredReplicas를 계산한 후 **가장 큰 값**을 선택하는 보수적인 접근 방식을 사용합니다. 예를 들어 CPU 50%, 메모리 70%, 큐 길이 100개 기준을 동시에 설정하면 세 가지 모두 계산한 후 가장 큰 값이 적용됩니다.

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

### 커스텀/외부 메트릭: Prometheus Adapter 활용

1) 어댑터 설치:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install metrics-server metrics-server/metrics-server -n kube-system

helm repo add prometheus-adapter https://kubernetes-sigs.github.io/prometheus-adapter
helm install prom-adapter prometheus-adapter/prometheus-adapter -n monitoring \
  --set rules.default=true
```

2) 매핑 규칙 설정(큐 길이 지표를 External 메트릭으로 노출):

```yaml
# values.yaml 예시

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

3) HPA에서 외부 메트릭 사용:

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

### KEDA(대안/보완 솔루션)

KEDA(Kubernetes Event-driven Autoscaling)는 이벤트 및 큐 기반 스케일링을 지원하며, Kafka, RabbitMQ, SQS, HTTP 등 60개 이상의 스케일러를 제공합니다. **내부적으로 HPA를 생성**하며 **버스트 부하**에 빠르게 대응할 수 있습니다.

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda -n keda --create-namespace
```

간단한 KEDA 스케일러 구성 예시:

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
      value: "100"     # 큐 길이 목표값
```

---

## 수직 확장(VPA)

### 개념 및 전략

VPA는 컨테이너의 `requests/limits`를 **추천**하고(Off/Initial 모드), 필요시 **자동 반영**(Auto 모드)할 수 있습니다. 리소스 변경을 반영할 때는 **Pod 재시작**이 필요하므로, 무중단 배포 레이어(롤링 업데이트, 레디니스 프로브, 피어 서비스)와 조합하여 구성해야 합니다.

### 운영 모드

| updateMode | 의미 | 권장 사용 시나리오 |
|---|---|---|
| `Off` | 추천만 제공 | HPA와 병행 사용 시 충돌 회피, 대부분의 실무 환경 |
| `Initial` | 최초 생성 시 추천값 반영 | 베이스라인 자동화 |
| `Auto` | 동적으로 추천값 반영(재시작 수반) | 비상태/탄력적 워크로드, 야간 유지보수 시간대 |

### 설치 및 구성 예시

```bash
# 최신 릴리즈 경로 확인 후 적용 (버전 예시: 0.14.0)

kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/vertical-pod-autoscaler-0.14.0/deploy/vpa-crd.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/vertical-pod-autoscaler-0.14.0/deploy/recommender.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/vertical-pod-autoscaler-0.14.0/deploy/updater.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/vertical-pod-autoscaler-0.14.0/deploy/admission-controller.yaml
```

VPA 구성 예시:

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
    updateMode: "Off"   # 실무 기본: 추천만 제공
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

> 실무 권장 패턴: **VPA(Off 모드)로 추천값 확인 → HPA는 레플리카 수 조절**. 야간 점검 시간대에 추천값을 배치 작업으로 반영합니다.

---

## Cluster Autoscaler — 노드 자동 확장

### 동작 원리

- 새로운 Pod가 **스케줄링 불가능**(자원 부족) 상태일 때 노드 증가.
- 노드가 **유휴 상태**(비어 있거나 축출 가능)일 때 노드 감소 (PodDisruptionBudget/PDB 존중).

### 전제 조건

- 클라우드 관리형 Kubernetes(GKE/EKS/AKS)에서는 NodeGroup/Auto Scaling Group이 필요합니다.
- 온프레미스 환경에서는 Karpenter/Cluster API 등 대안 솔루션을 고려해야 합니다.

### 설치 예시 (Helm을 통한 EKS 환경)

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler -n kube-system \
  --set autoDiscovery.clusterName=my-eks \
  --set awsRegion=ap-northeast-2 \
  --set rbac.serviceAccount.create=true \
  --set rbac.serviceAccount.name=cluster-autoscaler
```

**중요 구성 옵션**
- **태인트/토폴로지/스팟 인스턴스** 고려: 스케줄링 불가 원인과 노드 풀 매칭이 가능해야 합니다.
- `--balance-similar-node-groups`, `--expander=least-waste` 등의 정책 설정.

---

## 레이어 조합 전략

| 시나리오 | 권장 조합 | 비고 |
|---|---|---|
| 웹/API 버스트 트래픽 | **HPA + Cluster Autoscaler** | 빠른 Pod 확장 + 노드 보충 |
| 배치/큐 처리 작업 | **KEDA(+HPA) + Cluster Autoscaler**, VPA(Off) | 외부 큐 메트릭 기반 스케일링 |
| 메모리 민감/GC 부하 애플리케이션 | **VPA(Off)**로 사이징 개선 + HPA | 권장치를 주기적으로 반영 |
| 비용 최적화 | HPA + Cluster Autoscaler + 적극적 스케일 다운 | 안정화 창/쿨다운 시간 튜닝 |
| 고가용성 요구사항 | HPA + PDB + 분산 토폴로지 | Cluster Autoscaler 축소 시에도 가용성 확보 |

**HPA와 VPA 동시 사용 시 주의사항**
- HPA: 레플리카 수 조절, VPA: 리소스 requests 조절 → 두 시스템이 동시에 **활성 변경**을 수행하면 **목표 분모가 변동하여 불안정**해질 수 있습니다.
- 일반적인 해결책: **VPA=Off/Initial 모드**, HPA=Active 모드로 운영.

---

## 운영 튜닝: Behavior/쿨다운/워밍업 설정

- **stabilizationWindowSeconds**: 급격한 변동 억제(특히 scaleDown 시 중요).
- **policies**: 분당/퍼센트 상한 설정으로 과도한 확장/축소 방지.
- **워밍업**: 새 Pod가 가용 상태가 될 때까지 평균 메트릭에 반영 지연 → **readinessProbe**, **preStop** 설정으로 안전성 확보.
- **빈번한 GC/메모리 스파이크**: 메모리 목표값은 보수적으로 설정, scaleDown 안정화 창 확대.

---

## 실전 예제: 애플리케이션 스케일링 구성

### 예제 Deployment + HPA + PDB

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: server
        image: ealen/echo-server:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 80
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
    matchLabels:
      app: echo
```

### 부하 생성 테스트

```bash
kubectl run -it --rm load --image=busybox --restart=Never -- /bin/sh -c \
 'while true; do wget -q -O- http://echo.demo.svc.cluster.local; done'
```

### 관측 및 검증 포인트

- `kubectl get hpa -n demo -w` (실시간 HPA 상태 감시)
- `kubectl top pods -n demo` (리소스 사용량 확인)
- Grafana 대시보드에서 CPU 사용률, 레플리카 수, 응답 지연 시간 추적

---

## 알고리즘 및 수학적 기초

### 리소스 메트릭 계산 공식

$$
U_{\text{avg}} = \frac{\sum_{i=1}^{N} \text{usage}_i}{\sum_{i=1}^{N} \text{request}_i} \times 100
\quad,\quad
R_{\text{desired}} = R_{\text{current}} \times \frac{U_{\text{avg}}}{U_{\text{target}}}
$$

### 외부 메트릭 계산 공식

외부 메트릭 목표가 `AverageValue = V_target` 일 경우:

$$
R_{\text{desired}} = \left\lceil \frac{\sum_{i=1}^{N} \text{metric}_i}{V_{\text{target}}} \right\rceil
$$

> 예시: 큐 길이 합계가 2,000이고 목표값이 200인 경우 → 10개의 레플리카 필요.

---

## 테스트 전략 및 시나리오

| 시나리오 | 목표 | 방법 |
|---|---|---|
| 버스트 트래픽 | scaleUp 속도 및 정책 검증 | `hey -z 2m -q 200 -c 100 <서비스-URL>` |
| 장기 트래픽 | scaleDown 안정성 검증 | 안정화 창/정책 튜닝 |
| GC 스파이크 | 메모리 목표값 및 워밍업 검증 | p99 레이턴시/GC 시간 모니터링 |
| 큐 기반 스케일링 | 외부 메트릭 정확도 검증 | 큐 길이와 처리율 상관관계 분석 |
| 노드 축소 | PDB/축출 안전성 검증 | Cluster Autoscaler 로그 및 이벤트 검토 |

---

## 문제 해결 가이드

| 증상 | 가능한 원인 | 해결 방안 |
|---|---|---|
| `kubectl top` 명령 실패 | Metrics Server 미설치/TLS 문제 | 재설치 또는 인자(`--kubelet-insecure-tls`) 추가 |
| HPA 작동하지 않음 | requests 미설정/네임스페이스 불일치 | Pod 스펙/권한/타깃 ref 확인 |
| 확장 속도 느림 | 안정화 창/정책이 너무 보수적 | `behavior.scaleUp` 설정 완화 |
| 과도한 축소 | stabilizationWindow 부족 | scaleDown 창 확대/percent 값 완화 |
| 외부 메트릭 값 0 | 어댑터 쿼리/라벨 불일치 | PromQL/selector/네임스페이스 매핑 수정 |
| Cluster Autoscaler 미확장 | 스케줄링 불가가 아님(리소스 외 원인) | 토폴로지/taint/affinity 설정 점검 |
| Cluster Autoscaler 축소 실패 | PDB/DaemonSet/바인딩된 Pod 존재 | 축출 가능성/분산 배치 확인 |

---

## 보안 및 거버넌스 고려사항

- **리소스 요청 표준화**: 팀별 가이드라인과 최소값 기준 수립.
- **네임스페이스 및 라벨링 일관성**: HPA/어댑터 셀렉터가 명확히 매칭되도록 구성.
- **최소 권한 원칙**: HPA/어댑터/메트릭스 접근에 필요한 RBAC 최소화.
- **변경 관리 및 감사**: HPA/VPA/Cluster Autoscaler 파라미터 변경 시 영향도가 크므로 PR 승인 프로세스 및 게이트 구현.

---

## 운영 플레이북 요약

1) 메트릭 소스 준비(Metrics Server/Prometheus 설치).
2) 워크로드에 **requests/limits** 정의.
3) **HPA(v2)** 배치 및 **behavior** 설정 구성.
4) 큐/외부 메트릭 사용 시 **Prometheus Adapter** 또는 **KEDA** 설정.
5) **PDB** 및 **RollingUpdate** 전략 조율.
6) **Cluster Autoscaler** 정책 및 노드 풀 용량 확인.
7) Grafana/Alerting으로 **p95/레플리카 수/큐 길이** 모니터링 및 경보 설정.
8) 부하 테스트 수행 → 정책 미세 조정.

---

## 부록: 실무형 템플릿 모음

### CPU+메모리+큐 혼합 HPA 템플릿

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
        name: http_requests_inflight
      target:
        type: AverageValue
        averageValue: "120"
```

### VPA 추천 + 범위 제한 템플릿

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: svc-vpa
  namespace: prod
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: svc
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      controlledResources: ["cpu", "memory"]
      minAllowed:
        cpu: "300m"
        memory: "512Mi"
      maxAllowed:
        cpu: "6"
        memory: "12Gi"
```

### KEDA HTTP 스케일링 템플릿

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

## 결론

Kubernetes 오토스케일링은 **HPA**, **VPA**, **Cluster Autoscaler** 세 가지 핵심 구성 요소로 이루어진 다층적 시스템입니다. 각 구성 요소는 명확한 책임 범위를 가지며, 효과적인 조합을 통해 비용, 성능, 안정성의 최적 균형을 달성할 수 있습니다.

**HPA**는 워크로드의 레플리카 수를 동적으로 조절하여 트래픽 변화에 대응합니다. **VPA**는 컨테이너의 리소스 요구사항을 분석하여 적절한 사이징을 추천하며, **Cluster Autoscaler**는 인프라 수준에서 노드 용량을 관리합니다.

기본적인 조합으로는 **HPA와 Cluster Autoscaler**를 함께 사용하고, 리소스 사이징 최적화를 위해 **VPA(Off/Initial 모드)**로 추천값을 확인하는 접근 방식이 실무에서 효과적입니다. 이벤트 기반 스케일링이 필요한 경우 **KEDA**를 고려할 수 있습니다.

성공적인 오토스케일링 운영을 위해서는 **behavior 정책과 안정화 창 설정**, **PDB와 프로브 구성**, **체계적인 관측 및 알림 시스템**이 필수적입니다. 각 구성 요소의 수학적 원리와 알고리즘을 이해하고, 실제 부하 테스트를 통해 정책을 검증 및 튜닝하는 과정이 중요합니다.

이러한 접근 방식을 통해 애플리케이션의 성능 요구사항을 충족하면서도 인프라 비용을 최적화하고 시스템의 전반적인 안정성을 유지할 수 있습니다.