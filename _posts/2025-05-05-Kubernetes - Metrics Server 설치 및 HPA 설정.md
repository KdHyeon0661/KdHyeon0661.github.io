---
layout: post
title: Kubernetes - Metrics Server 설치 및 HPA 설정
date: 2025-05-05 23:20:23 +0900
category: Kubernetes
---
# Kubernetes에서 HPA와 Metrics Server 설정하기

Kubernetes에서 **수요에 따라 Pod 수를 자동으로 조절**하려면 **실시간 리소스 사용량 데이터**가 필요합니다. 이를 제공하는 두 가지 핵심 구성 요소는 다음과 같습니다.

- **Metrics Server**: Pod와 Node의 리소스 사용량(CPU, 메모리)을 경량으로 수집하고 집계하여 `metrics.k8s.io` API로 제공합니다.
- **HPA (Horizontal Pod Autoscaler)**: 수집된 메트릭을 기준으로 대상 리소스(Deployment, StatefulSet 등)의 **replicas 수**를 자동으로 증감시킵니다.

이 문서는 HPA와 Metrics Server의 개념을 설명하고, **운영 환경에서 바로 적용할 수 있는 세부 옵션과 실전 예제**를 제공합니다.

---

## 필수 개념 이해

- HPA는 **컨테이너에 설정된 리소스 request** 값을 기준으로 **사용률(Utilization)**을 계산합니다. CPU request가 없으면 CPU 기반 HPA가 동작하지 않으며, 메모리도 동일한 원리가 적용됩니다.
- `autoscaling/v2`(또는 `v2beta2`) API를 사용하면 **여러 메트릭을 동시에 지정**하고, **behavior(스케일링 속도 및 완충) 정책**을 설정하며, **메모리, 커스텀, 외부 메트릭** 등을 활용할 수 있습니다.
- Metrics Server는 **장기 보관이나 복잡한 쿼리**를 위한 시스템이 아닙니다. 그런 목적에는 Prometheus나 시계열 데이터베이스가 적합합니다. Metrics Server는 HPA와 `kubectl top` 명령을 위한 **저지연 단기 메트릭 엔드포인트**입니다.

---

## Metrics Server 소개

- kubelet의 Summary API를 스크랩하여 **Node와 Pod** 단위의 CPU, 메모리 사용량을 집계합니다.
- `metrics.k8s.io` API 그룹을 제공하여 `kubectl top` 명령과 HPA가 사용할 수 있게 합니다.
- 기본적으로 **몇 분 내의 단기 샘플**만 유지하며 장기 저장을 하지 않습니다.

### 설치 상태 확인

```bash
kubectl api-versions | grep metrics.k8s.io
kubectl top nodes
kubectl top pods -A
```

`metrics.k8s.io` API가 확인되고 `kubectl top` 명령이 정상 작동하면 설치가 완료된 것입니다.

---

## Metrics Server 설치

### 공식 매니페스트로 설치 (권장)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

로컬 개발 환경(Minikube, Kind)이나 자체 서명 TLS 인증서를 사용하는 경우 kubelet 인증 및 주소 문제가 발생할 수 있습니다. 이 경우 **Deployment의 인자(args)**를 조정해야 합니다.

```yaml
# components.yaml 파일 내 metrics-server 컨테이너 args 예시

containers:
- name: metrics-server
  image: registry.k8s.io/metrics-server/metrics-server:v0.7.2
  args:
    - --kubelet-insecure-tls
    - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalIP
    - --metric-resolution=15s   # 샘플링 주기 (기본값: 60초). 너무 낮추면 부하 증가
```

> 운영 환경에서는 `--kubelet-insecure-tls` 옵션 사용을 피하고, **정상적인 Kubelet 인증서**를 구성하는 것을 권장합니다.

### 설치 검증

```bash
kubectl -n kube-system get deployment metrics-server
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top nodes
kubectl top pods -A
```

---

## HPA 이해하기 (autoscaling/v1 vs v2)

- **대상 리소스**: Deployment, StatefulSet, ReplicaSet 등
- **기준 메트릭**:
  - v1: CPU 및 메모리 **사용률(Utilization)** 중심으로 제한적입니다.
  - v2: **여러 타입의 메트릭(Resource, Pods, Object, External)**을 동시에 지원하며 **behavior 정책**을 통해 스케일링 속도와 완충을 제어할 수 있습니다.

### 사용률(Utilization) 계산 방식 (CPU/메모리)

HPA는 각 Pod 내 컨테이너의 `request` 값을 기준으로 사용률을 계산한 후 모든 Pod에 대한 **평균**을 구합니다.

$$
\text{Utilization} = \frac{\sum\limits_{pods}\sum\limits_{containers} \text{usage}_{container}}{\sum\limits_{pods}\sum\limits_{containers} \text{request}_{container}} \times 100(\%)
$$

> 이 사용률을 기준으로 **현재 replicas 수가 목표(Target) 값에 비해 얼마나 초과하거나 미달인지**를 계산하고, 정책에 따라 replicas 수를 조정합니다.

---

## 실습: 예제 애플리케이션 배포 및 부하 생성

### 예제 Deployment (CPU request 및 limit 포함)

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
            cpu: "100m"   # HPA CPU 사용률 계산에 필수
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

서비스 노출:

```bash
kubectl apply -f cpu-demo.yaml
kubectl expose deployment cpu-demo --name=cpu-demo --port=80
```

### 부하 생성 (간단한 방법)

```bash
kubectl run -it --rm loadgen --image=busybox -- /bin/sh
# Pod 내부에서 다음 명령 실행
while true; do wget -q -O- http://cpu-demo.default.svc.cluster.local > /dev/null; done
```

> 더 현실적인 부하 테스트를 위해서는 `busybox` 대신 `alpine/bombardier`, `rakyll/hey`, `ab`, `wrk` 등의 도구를 사용할 수 있습니다.

---

## HPA 생성

### 간단한 HPA 생성 (CPU 기준, 50% 목표)

```bash
kubectl autoscale deployment cpu-demo \
  --cpu-percent=50 \
  --min=1 \
  --max=5
```

상태 확인:

```bash
kubectl get hpa
kubectl describe hpa cpu-demo
```

### YAML을 통한 HPA 생성 (CPU + 메모리 동시 기준)

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
        averageUtilization: 50     # CPU 사용률 50% 목표
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70     # 메모리 사용률 70% 목표
```

> 여러 메트릭을 지정하면 **가장 높은 replicas 수를 요구하는 메트릭**이 적용됩니다.

적용:

```bash
kubectl apply -f hpa-v2.yaml
```

---

## HPA 동작 관찰

```bash
kubectl get hpa -w  # 실시간 상태 감시
kubectl describe hpa cpu-mem-demo
kubectl get deployment cpu-demo
kubectl top pods -l app=cpu-demo
```

`kubectl describe hpa` 명령으로 확인할 수 있는 주요 정보:

- `Metrics`: 현재 및 목표 메트릭 값
- `Min/Max replicas`: 설정된 최소/최대 replicas 수
- `Replicas`: 현재 replicas 수
- `Conditions`: `ScalingActive`, `AbleToScale`, `ScalingLimited` 등의 상태
- 최근 스케일링 이벤트 메시지

---

## autoscaling/v2 고급 기능: behavior (스케일링 속도 및 완충 정책)

트래픽 스파이크나 노이즈로 인한 **과도한 스케일링**을 방지하려면 `behavior` 설정을 활용합니다.

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
        value: 100     # 한 번에 최대 100% 증가 (예: 3 → 6)
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300  # 5분 동안의 평균 추세를 확인 후 스케일 다운
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

- **stabilizationWindowSeconds**: 급격한 변화를 완충하기 위해 최근 N초 동안의 권장 값을 관찰하여 완만하게 적용합니다.
- **policies**: `Pods`(절대값) 또는 `Percent`(비율) 단위로 증가 및 감소를 제한합니다.
- **selectPolicy**: `Max`(가장 공격적), `Min`(가장 보수적), `Disabled` 중 선택합니다.

---

## 메모리 기반 HPA (autoscaling/v2)

메모리는 CPU와 달리 스로틀링(throttling)이 없으므로 **임계치 설정에 주의**가 필요합니다.

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

## 다중 컨테이너 Pod에서의 HPA

HPA는 **Pod 전체**의 CPU 및 메모리 사용률을 계산합니다. Pod 내 모든 컨테이너의 `requests` 합계와 `usage` 합계로 계산되므로, **주요 컨테이너**에 적절한 `requests` 값을 설정하는 것이 중요합니다.

Sidecar(예: Envoy, Istio-proxy)가 많은 리소스를 사용하는 경우, **별도의 request 설정**과 **리소스 재배분**이 필요할 수 있습니다.

---

## HPA와 Cluster Autoscaler 연동

HPA는 **Pod 수**만 조절할 뿐, **Node 수**는 증가시키지 않습니다. Pod가 증가하여 스케줄링이 불가능해지면 **Cluster Autoscaler**가 노드를 증설해야 전체적인 스케일링이 완성됩니다.

- HPA: Pod 수 조절
- Cluster Autoscaler: Node 수 조절

**두 시스템이 모두 적절히 구성**되어야 **수평 스케일링이 실제 처리량 향상으로 이어집니다.**

---

## 실전 문제 해결 가이드

**`kubectl top` 명령에 결과가 표시되지 않음**
- **원인**: Metrics Server 미설치, CrashLoop, TLS 문제
- **점검**: `kubectl -n kube-system logs deployment/metrics-server`, `kubectl get apiservices`
- **해결**: 올바른 이미지와 인자로 재배포, TLS 및 주소 옵션 확인

**HPA가 항상 `minReplicas`를 유지함**
- **원인**: 실제 사용량이 목표치 미만
- **점검**: `kubectl top pods`, `kubectl describe hpa`
- **해결**: 부하 생성 확인, request 값 및 target 목표치 재조정

**HPA가 전혀 작동하지 않음**
- **원인**: 컨테이너에 `resources.requests` 설정 없음
- **점검**: `kubectl get deployment -o yaml`
- **해결**: 대상 컨테이너에 CPU 및 메모리 request 설정 추가

**빈번한 스케일링 진동(토글링) 발생**
- **원인**: 급격한 트래픽 변동 또는 메트릭 노이즈
- **점검**: `kubectl describe hpa`로 behavior 설정 확인
- **해결**: `behavior` 설정으로 완충, stabilization 시간 확대, 정책 보수화

**Scale Out이 발생하지 않고 대기 상태**
- **원인**: PDB(Pod Disruption Budget) 제한, 최대 처리량 도달, 리소스 부족
- **점검**: 이벤트 로그, 스케줄러 로그 확인
- **해결**: Cluster Autoscaler 구성, 노드 자원 상태, 리소스 쿼터 점검

**메모리 기준 과도한 스케일링 발생**
- **원인**: GC(Garbage Collection) 일시적 증가, 캐시 버스트
- **점검**: 메모리 사용 프로파일 분석
- **해결**: 목표치 상향 조정, 컨테이너 메모리 request 값 재평가

---

## HPA 상태 상세 분석

```bash
kubectl describe hpa cpu-demo-behavior
```

주요 분석 포인트:

- **Metrics 섹션**: 현재 메트릭 값과 목표값 비교
- **Conditions**:
  - `ScalingActive`: HPA가 정상적으로 작동 중인지 여부
  - `AbleToScale`: 최근 스케일링 작업이 가능했는지 여부
  - `ScalingLimited`: 정책에 의해 스케일링이 제한되었는지 여부
- **Events**: 최근 스케일링 이벤트 기록 (예: `SuccessfulRescale from 3 to 5`)

---

## 예제 모음

### 멀티 메트릭 HPA 예제

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

### Pods 메트릭 기반 HPA 예제 (참고용)

Prometheus Adapter 등을 통해 `messages_inflight` 메트릭이 노출되었다고 가정:

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
        averageValue: "30"  # Pod 당 30개의 메시지를 목표로 함
```

> 커스텀 또는 외부 메트릭을 사용하려면 **Prometheus Adapter**와 같은 어댑터가 필요합니다.

---

## 운영 환경 구성 시 확인 사항

- 컨테이너의 `resources.requests` 값이 실제 워크로드의 **사용량 프로파일과 일치**하는지 확인합니다.
- HPA의 **목표(target) 값**이 경험적 데이터나 실제 측정값을 기반으로 설정되었는지 확인합니다. (CPU의 경우 50~70%를 시작점으로 권장)
- `behavior` 설정을 통해 **트래픽 스파이크 완충**과 **과도한 스케일 다운 방지**가 적용되었는지 확인합니다.
- PodDisruptionBudget, PodTopologySpread, Affinity/Anti-Affinity 설정이 **스케일링을 방해하지 않는지** 확인합니다.
- Cluster Autoscaler가 HPA와 **함께 구성**되었는지 확인합니다.
- Metrics Server의 **샘플링 주기**와 **자원 요청/제한**이 환경에 적절한지 확인합니다.

---

## 전체 예제 묶음 (YAML 파일)

```yaml
# Deployment + Service + HPA 통합 예제

apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-demo
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
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: cpu-demo
spec:
  selector:
    app: cpu-demo
  ports:
  - port: 80
    targetPort: 80
---
# HPA v2 with behavior 설정

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

## 로컬 및 클라우드 환경별 특이사항

- **Minikube/Kind**: Kubelet 인증서 문제가 발생할 경우 `--kubelet-insecure-tls` 옵션으로 우회할 수 있습니다. (운영 환경에서는 비권장)
- **EKS/AKS/GKE**: 대부분의 클라우드 관리형 Kubernetes는 Metrics Server를 add-on이나 기본 설치로 제공합니다. 클러스터 버전에 맞는 릴리스를 사용하세요.
- **권한(RBAC)**: `metrics-server`에 필요한 ClusterRole 및 Binding이 매니페스트에 포함되어 있어야 합니다.
- **샘플링 해상도**: 너무 낮은 샘플링 주기는 API 부하와 리소스 비용을 증가시킬 수 있습니다. 기본값인 60초로 시작하는 것을 권장합니다.

---

## 결론

Kubernetes에서 효과적인 자동 스케일링을 구현하려면 Metrics Server와 HPA의 조화로운 구성이 필수적입니다.

**Metrics Server**는 HPA와 `kubectl top` 명령을 위한 경량의 실시간 메트릭 공급원 역할을 합니다. 간단한 설치로 클러스터의 리소스 사용 현황을 즉시 파악할 수 있게 해줍니다.

**autoscaling/v2 HPA**를 사용하면 CPU와 메모리 사용률뿐만 아니라 다양한 메트릭을 동시에 모니터링할 수 있으며, `behavior` 정책을 통해 안정적이고 예측 가능한 오토스케일링을 구현할 수 있습니다.

운영 환경에서는 단순한 설정을 넘어, 컨테이너의 `requests` 값 정합성 확인, 트래픽 스파이크에 대한 완충 정책 구성, Cluster Autoscaler와의 연동, 지속적인 모니터링과 튜닝이 필요합니다. 이 가이드의 YAML 파일과 명령어를 순서대로 따라가면, **설치 → 검증 → 부하 생성 → 스케일링 관찰 → 정책 튜닝**까지의 전체 흐름을 경험하며 HPA 운영에 대한 이해를 깊이있게 쌓을 수 있습니다.