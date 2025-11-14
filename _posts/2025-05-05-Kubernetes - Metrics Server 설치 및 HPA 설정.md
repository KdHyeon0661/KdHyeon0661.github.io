---
layout: post
title: Kubernetes - Metrics Server 설치 및 HPA 설정
date: 2025-05-05 23:20:23 +0900
category: Kubernetes
---
# Metrics Server 설치 및 HPA(Horizontal Pod Autoscaler) 설정하기

Kubernetes에서 **수요에 따라 Pod 수를 자동 조절**하려면 **실시간 리소스 사용량**이 필요하다.
이를 담당하는 구성 요소가 다음 두 가지다.

- **Metrics Server**: Pod/Node의 리소스 사용량(CPU/메모리)을 경량으로 수집·집계하여 `metrics.k8s.io` API로 제공
- **HPA (Horizontal Pod Autoscaler)**: 해당 메트릭을 기준으로 대상 리소스(Deployment/StatefulSet 등)의 **replicas**를 자동 증감

본 글은 기존 요약본을 기반으로, **운영에서 바로 쓸 수 있는 세부 옵션과 실전 예제**를 대폭 확장했다.

---

## 필수 개념 요약

- HPA는 **컨테이너 리소스 request**를 기준으로 **사용률(Utilization)**을 계산한다. request가 없으면 CPU 기준 HPA가 동작하지 않는다(메모리 Utilization도 동일).
- `autoscaling/v2`(또는 `v2beta2`)를 사용하면 **여러 메트릭 동시 지정**, **behavior(스케일 속도/완충) 정책**, **메모리/컨테이너 포트/외부/커스텀 메트릭** 등을 활용할 수 있다.
- Metrics Server는 **장기 보관/복잡한 질의** 목적이 아니다. 그건 Prometheus/LTS가 맡는다. Metrics Server는 HPA/`kubectl top`을 위한 **저지연 단기 메트릭 엔드포인트**다.

---

## Metrics Server란?

- kubelet Summary API를 스크레이프 → **Node/Pod** 단위로 CPU/메모리 사용량을 집계
- API 리소스(Group: `metrics.k8s.io`)를 제공하여 `kubectl top`과 HPA가 사용
- 기본적으로 **몇 분 내 단기 샘플**만 유지(장기 저장 아님)

### 확인 명령

```bash
kubectl api-versions | grep metrics.k8s.io
kubectl top nodes
kubectl top pods -A
```

`metrics.k8s.io`가 보이고 `kubectl top`이 동작하면 설치가 정상이다.

---

## Metrics Server 설치

### 공식 매니페스트(권장)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

일부 로컬/개발 환경(Minikube/Kind/자체서명 TLS)에서는 kubelet 인증/주소 이슈가 발생할 수 있으므로 **Deployment args**를 조정한다.

```yaml
# components.yaml의 metrics-server 컨테이너 args 예시

containers:
- name: metrics-server
  image: k8s.gcr.io/metrics-server/metrics-server:v0.7.2
  args:
    - --kubelet-insecure-tls
    - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalIP
    - --metric-resolution=15s   # 샘플 주기(기본 60s). 너무 낮추면 부하↑
```

> 운영환경에서는 `--kubelet-insecure-tls` 사용을 가급적 피하고, **정상적인 Kubelet 인증서**를 구성하는 것을 권장한다.

### 설치 검증

```bash
kubectl -n kube-system get deploy metrics-server
kubectl get apiservices | grep metrics
kubectl top nodes
kubectl top pods -A
```

---

## HPA란? (autoscaling/v1 vs v2)

- **대상**: Deployment / StatefulSet / ReplicaSet 등
- **기준 메트릭**
  - v1: CPU/메모리 **Utilization**(퍼센트) 중심(제한적)
  - v2: **여러 타입의 메트릭 동시** 지원 (Resource, Pods, Object, External) + **behavior**(스케일 속도/완충) 정책

### Utilization 계산식(CPU/메모리)

HPA는 각 Pod의 컨테이너별 `request`를 기준으로 사용률을 구한 뒤 **평균**을 낸다.

$$
\text{Utilization} = \frac{\sum\limits_{pods}\sum\limits_{containers} \text{usage}_{container}}{\sum\limits_{pods}\sum\limits_{containers} \text{request}_{container}} \times 100(\%)
$$

> 사용률을 기준으로 **현재 replicas가 목표(Target) 대비 얼마나 초과/미달인지**를 계산, 정책에 따라 증감한다.

---

## 실습: 예제 앱 배포 및 부하 유발

### 예제 Deployment (CPU request/limit 포함)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-demo
  labels:
    app: cpu-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-demo
  template:
    metadata:
      labels:
        app: cpu-demo
    spec:
      containers:
      - name: web
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"   # HPA CPU Utilization 계산에 필수
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

서비스 노출:

```bash
kubectl apply -f cpu-demo.yaml
kubectl expose deploy cpu-demo --name=cpu-demo --port=80
```

### 부하 유발(간단)

```bash
kubectl run -it --rm loadgen --image=busybox -- /bin/sh
# pod 내부에서

while true; do wget -q -O- http://cpu-demo.default.svc.cluster.local > /dev/null; done
```

> 더 현실적인 부하는 `busybox` 대신 `alpine/bombardier`, `rakyll/hey`, `ab`, `wrk` 등을 사용.

---

## HPA 생성

### CLI(v1) 간단 생성 (CPU 기준, 50% 목표)

```bash
kubectl autoscale deployment cpu-demo \
  --cpu-percent=50 \
  --min=1 \
  --max=5
```

확인:

```bash
kubectl get hpa
kubectl describe hpa cpu-demo
```

### YAML(v2) — CPU + 메모리 동시

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-mem-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-demo
  minReplicas: 1
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50     # CPU 50%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70     # 메모리 70%
```

> 여러 메트릭을 지정하면 **가장 높은 스케일 요구치**가 적용된다(더 큰 replicas를 요구하는 메트릭 우선).

적용:

```bash
kubectl apply -f hpa-v2.yaml
```

---

## HPA 동작 관찰

```bash
kubectl get hpa -w
kubectl describe hpa cpu-mem-demo
kubectl get deploy cpu-demo
kubectl top pods -l app=cpu-demo
```

`describe`에서 확인할 필드:

- `Metrics` / `Min/Max replicas` / `Replicas`
- `Conditions`: `ScalingActive`, `AbleToScale`, `ScalingLimited` 등
- 최근 스케일 이벤트 메시지

---

## autoscaling/v2 고급: behavior(스케일 속도/완충)

트래픽 스파이크/노이즈로 **과도한 스케일링**을 방지하려면 `behavior`를 설정한다.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-demo-behavior
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-demo
  minReplicas: 1
  maxReplicas: 20
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 100     # 한 번에 최대 100% 증가(예: 3→6)
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300  # 5분 평균 추세 보고 내림
      policies:
      - type: Percent
        value: 20      # 한 번에 최대 20% 감소
        periodSeconds: 60
      selectPolicy: Min
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

- **stabilizationWindowSeconds**: 급격한 변화를 완충(최근 N초 동안의 권장치를 관찰해 완만하게 적용)
- **policies**: `Pods`/`Percent` 단위 증가·감소 제한
- **selectPolicy**: `Max`(가장 공격적), `Min`(가장 보수적), `Disabled` 중 선택

---

## 메모리 기반 HPA(autoscaling/v2)

메모리는 스로틀링이 없으므로 **임계치 설정에 주의**. 예: 70% 목표.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mem-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-demo
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

---

## 다중 컨테이너 Pod의 HPA

HPA는 **Pod 전체**의 CPU(또는 메모리) 사용률을 계산한다.
Pod 내 컨테이너의 `requests` 합과 `usage` 합으로 계산되므로 **필수 컨테이너**에 `requests`를 명확히 설정하자.
Sidecar(예: Envoy, Istio-proxy)가 큰 사용량을 갖는다면 **별도 request**와 **리밸런싱**이 필요하다.

---

## HPA + Cluster Autoscaler 연동

HPA는 **Pod 개수**를 조절할 뿐, **Node 수**는 늘리지 않는다.
Pod가 늘며 스케줄링 불가가 발생하면 **Cluster Autoscaler**가 노드를 증설해야 스케일이 완성된다.

- HPA: Pods
- Cluster Autoscaler: Nodes

**둘 다** 적절히 구성되어야 **수평 스케일 → 실제 처리량 향상**이 이루어진다.

---

## 실전 트러블슈팅

| 현상 | 원인 | 점검 명령 | 해결책 |
|------|------|-----------|--------|
| `kubectl top` 결과 없음 | Metrics Server 미설치/CrashLoop/TLS | `kubectl -n kube-system logs deploy/metrics-server`, `kubectl get apiservices` | 올바른 이미지/args로 재배포, TLS/주소 옵션 확인 |
| HPA가 항상 `minReplicas` 유지 | 실제 사용량이 목표 미만 | `kubectl top pods`, `kubectl describe hpa` | 부하 생성 확인, request·target 재조정 |
| HPA가 작동 안 함(에러) | 컨테이너에 `resources.requests` 없음 | `kubectl get deploy -o yaml` | 대상 컨테이너에 CPU/메모리 request 설정 |
| 잦은 진동(토글링) | 급격한 트래픽 변동/노이즈 | `kubectl describe hpa` behavior 확인 | `behavior`로 완충, stabilization 확대, 정책 보수화 |
| ScaleOut 안 됨(대기) | PDB/최대 스루풋 도달/리소스 부족 | 이벤트·스케줄러 로그 | Cluster Autoscaler/노드 자원/리소스쿼터 점검 |
| 메모리 기준 과도 스케일 | GC 일시 증가/캐시 버스트 | 메모리 프로파일 | 목표치 상향, 컨테이너 메모리 request 재평가 |

---

## 상태 관찰을 위한 describe 예시 해석

```bash
kubectl describe hpa cpu-demo-behavior
```

주요 섹션:

- **Metrics**: 현재/목표, 소스(Resource/Pods/Object/External)
- **Conditions**:
  - `ScalingActive`: HPA가 유효하게 작동 중인지
  - `AbleToScale`: 최근 스케일 동작 가능 여부
  - `ScalingLimited`: 정책으로 제한되었는지(과도 증감 방지로 제한되면 True)
- **Events**: 최근 스케일 이벤트(예: `SuccessfulRescale from 3 to 5`)

---

## 예제 모음

### v2 — 다중 메트릭 + OR(최대값) 스케일

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: multi-metric
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 20
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
          averageUtilization: 75
```

### v2 — 특정 포트의 큐 길이를 Pods 메트릭으로(참고용)

Prometheus Adapter 등을 통해 `Pods` 메트릭으로 노출했다 가정:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: queue-based
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker
  minReplicas: 1
  maxReplicas: 50
  metrics:
  - type: Pods
    pods:
      metric:
        name: messages_inflight
      target:
        type: AverageValue
        averageValue: "30"  # Pod 하나당 30 메시지 목표
```

> 커스텀/외부 메트릭은 **Prometheus Adapter** 같은 어댑터가 필요하다.

---

## 운영 체크리스트

- 컨테이너별 `resources.requests`와 실제 워크로드의 **사용량 프로파일**이 일치하는가?
- HPA **target**은 경험적/실측 기반으로 설정했는가? (CPU 50~70% 권장 출발점)
- `behavior`로 **스파이크 완충**과 **과잉 스케일 다운** 방지가 적용되어 있는가?
- PodDisruptionBudget, PodTopologySpread, Affinity/Anti-Affinity가 **스케일을 가로막지** 않는가?
- Cluster Autoscaler와 **동시에** 구성되었는가?
- Metrics Server의 **샘플링 주기**와 **자원 요청/제한**이 적절한가?

---

## 전체 예시 묶음(YAML 일괄)

```yaml
# Deployment + Service

apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-demo
spec:
  replicas: 1
  selector:
    matchLabels: { app: cpu-demo }
  template:
    metadata:
      labels: { app: cpu-demo }
    spec:
      containers:
      - name: web
        image: nginx:1.25
        ports: [{ containerPort: 80 }]
        resources:
          requests: { cpu: "100m", memory: "128Mi" }
          limits:   { cpu: "500m", memory: "256Mi" }
---
apiVersion: v1
kind: Service
metadata:
  name: cpu-demo
spec:
  selector: { app: cpu-demo }
  ports:
  - port: 80
    targetPort: 80
---
# HPA v2 with behavior

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-demo-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-demo
  minReplicas: 1
  maxReplicas: 15
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 20
        periodSeconds: 60
      selectPolicy: Min
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

적용:

```bash
kubectl apply -f demo-all.yaml
```

---

## 부록 — 로컬/클라우드 특이사항

- **Minikube/Kind**: Kubelet 인증서 이슈 시 `--kubelet-insecure-tls`로 우회 가능(운영 비권장)
- **EKS/AKS/GKE**: 대부분 Metrics Server를 add-on/설치로 제공. 클러스터 버전에 맞춰 릴리스 사용
- **권한(RBAC)**: `metrics-server`에 필요한 ClusterRole/Binding이 매니페스트에 포함되어야 함
- **샘플링 해상도**: 너무 낮추면 API 부하·리소스 비용 증가. 기본 60s 권장 시작

---

## 결론

- **Metrics Server**는 HPA/`kubectl top`을 위한 **경량 실시간 메트릭 공급원**이다.
- **autoscaling/v2 HPA**를 사용하면 CPU/메모리 외에도 다양한 메트릭과 **behavior 정책**으로 **안정적이며 예측 가능한 오토스케일링**을 구현할 수 있다.
- 실서비스에서는 **requests 정합성**, **스파이크 완충**, **Cluster Autoscaler 연동**, **운영 모니터링**을 포함한 전반적 튜닝이 필수다.

이 가이드의 YAML과 명령을 순서대로 적용하면, **설치 → 검증 → 부하 → 스케일 관찰 → 정책 튜닝**까지 일관된 플로우로 HPA를 손에 익힐 수 있다.
