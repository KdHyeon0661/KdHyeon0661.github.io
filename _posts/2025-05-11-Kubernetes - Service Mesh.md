---
layout: post
title: Kubernetes - Service Mesh
date: 2025-05-11 19:20:23 +0900
category: Kubernetes
---
# Service Mesh 소개: Istio vs Linkerd 개요

Kubernetes에서 마이크로서비스를 운영하다 보면 다음과 같은 요구사항이 반복적으로 발생합니다.

- 서비스 간 트래픽을 **관찰**하고 지연 시간, 에러율, 호출 경로를 **가시화**
- **재시도(Retry), 타임아웃(Timeout), 서킷 브레이커(Circuit Breaker)**와 같은 네트워크 회복성 기능 표준화
- **카나리 배포, A/B 테스트, 트래픽 미러링** 등 점진적 배포 전략 구현
- **상호 TLS(mTLS)**를 통한 서비스 간 **암호화 및 상호 인증**과 정책 기반 제어

이러한 기능을 **애플리케이션 코드에 직접 구현하지 않고**, **인프라 레이어**에서 일괄 적용하는 것이 **서비스 메시(Service Mesh)**의 핵심 개념입니다.

---

## 서비스 메시 핵심 개념

### 데이터 플레인(Data Plane) vs 제어 플레인(Control Plane)

- **데이터 플레인**: 각 Pod에 배치되는 **사이드카 프록시**가 실제 데이터 트래픽을 처리합니다.
  - 대표 예시: Envoy(Istio), linkerd-proxy(Linkerd, Rust 기반)
- **제어 플레인**: 선언된 정책과 구성을 수집하고 검증한 후, 데이터 플레인으로 **전파 및 동기화**합니다.

### 제공 기능 요약

| 범주 | 주요 기능 |
|---|---|
| 관측성(Observability) | 요청 지연 시간, 성공률, 에러율, 히스토그램, 서비스 의존성 그래프, 분산 트레이싱 |
| 트래픽 관리 | 호스트/경로/헤더 기반 라우팅, 카나리/A-B 테스트, 서킷 브레이커, 속도 제한, 트래픽 미러링 |
| 보안(Security) | 자동 **상호 TLS(mTLS)**, ID 기반 정책, 피어/요청 인증, 인가(RBAC) |
| 회복성(Reliability) | **재시도(Retry)**, **타임아웃(Timeout)**, **이상치 탐지(Outlier Detection)**(불량 인스턴스 격리) |
| 정책(Policy) | 서비스 간 통신 허용/차단, 인증서 순환, 속도 제한(확장 기능) |

---

## 주요 서비스 메시 비교: Istio vs Linkerd

| 항목 | **Istio** | **Linkerd** |
|---|---|---|
| 프록시 | Envoy (C++), 고기능 L7 프록시 | linkerd-proxy (Rust), 경량 프록시 |
| 복잡도 | 높음 (유연성과 기능 풍부) | 낮음 (간결성과 안정성 중점) |
| 설치 | `istioctl` / Helm / Operator | Linkerd 전용 CLI / Helm |
| 트래픽 제어 | 세밀함 (가중치 기반 분배, 헤더/쿠키 기반, 미러링 등) | TrafficSplit 중심 (간결함) |
| 보안 | mTLS 자동화 + 세밀한 인가 정책 | 기본 mTLS, 간단한 설정 |
| 관측 | Kiali, Prometheus/Jaeger/Zipkin 연동 | `linkerd viz`, Prometheus 연동 |
| 멀티 클러스터 | 다양한 패턴과 옵션 | 간결한 멀티 클러스터 지원 |
| 학습 곡선 | 가파름 | 완만함 |

> 선택 가이드: **엔터프라이즈 수준의 세밀한 제어와 확장성**이 필요하면 Istio. **경량화, 간결성, 빠른 도입**이 목표라면 Linkerd.

---

## 공통 실습 환경 준비

```bash
# 클러스터 생성 (예: Kind 사용 시)

kind create cluster --name mesh-lab

# 네임스페이스 준비

kubectl create namespace mesh-demo
kubectl label namespace mesh-demo istio-injection=enabled   # Istio 사용 시
```

테스트 워크로드 (두 가지 버전):

```yaml
# app.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v1
  namespace: mesh-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
      version: v1
  template:
    metadata:
      labels:
        app: api
        version: v1
    spec:
      containers:
        - name: api
          image: kennethreitz/httpbin
          ports:
            - containerPort: 80
              name: http
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: mesh-demo
spec:
  selector:
    app: api      # v1과 v2 모두 포함
  ports:
    - name: http
      port: 80
      targetPort: http
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v2
  namespace: mesh-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
      version: v2
  template:
    metadata:
      labels:
        app: api
        version: v2
    spec:
      containers:
        - name: api
          image: kennethreitz/httpbin
          ports:
            - containerPort: 80
              name: http
```

적용:

```bash
kubectl apply -f app.yaml
```

---

## Istio: 설치, 사이드카 주입, 트래픽 제어, 보안, 관측

### 설치 및 사이드카 주입

```bash
# istioctl 다운로드 (공식 가이드 참조)

istioctl install --set profile=demo -y
kubectl label namespace mesh-demo istio-injection=enabled
# 기존 Pod 재시작 필요

kubectl -n mesh-demo rollout restart deployment
```

확인:

```bash
kubectl -n istio-system get pods
kubectl -n mesh-demo get pods -o jsonpath='{..containers[*].name}' | tr ' ' '\n' | sort | uniq
# 각 Pod에 'istio-proxy' 컨테이너가 있어야 함
```

### 기본 라우팅: VirtualService와 DestinationRule

**목표:** `api` 서비스로 들어오는 트래픽을 `v1:90%`, `v2:10%`로 카나리 배포 방식으로 분배

```yaml
# istio-traffic.yaml

apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-destrule
  namespace: mesh-demo
spec:
  host: api.mesh-demo.svc.cluster.local
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-vs
  namespace: mesh-demo
spec:
  hosts:
    - api.mesh-demo.svc.cluster.local
  http:
    - route:
        - destination:
            host: api.mesh-demo.svc.cluster.local
            subset: v1
          weight: 90
        - destination:
            host: api.mesh-demo.svc.cluster.local
            subset: v2
          weight: 10
```

적용:

```bash
kubectl apply -f istio-traffic.yaml
```

부하 테스트 (클러스터 내부에서):

```bash
kubectl -n mesh-demo run load --image=curlimages/curl -it --rm -- \
  sh -lc 'for i in $(seq 1 50); do curl -s http://api.mesh-demo.svc.cluster.local/status/200 | wc -c; done'
```

> httpbin 응답에는 버전 정보가 포함되어 있지 않습니다. 실제 서비스에서는 응답에 버전 로그를 추가하거나, 헤더 기반 라우팅 실험을 위한 커스텀 애플리케이션을 사용하는 것을 권장합니다.

### 회복성: 재시도, 타임아웃, 이상치 탐지

```yaml
# istio-resilience.yaml

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-resilient
  namespace: mesh-demo
spec:
  hosts:
    - api.mesh-demo.svc.cluster.local
  http:
    - timeout: 2s
      retries:
        attempts: 3
        perTryTimeout: 500ms
        retryOn: gateway-error,connect-failure,refused-stream,5xx
      route:
        - destination:
            host: api.mesh-demo.svc.cluster.local
            subset: v1
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-outlier
  namespace: mesh-demo
spec:
  host: api.mesh-demo.svc.cluster.local
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 5s
      baseEjectionTime: 30s
```

적용 후, 특정 인스턴스가 연속으로 5xx 에러를 반환하면 자동으로 격리됩니다.

### mTLS와 인가 정책

**네임스페이스 전체에 엄격한 mTLS 적용:**

```yaml
# istio-mtls.yaml

apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: mesh-demo
spec:
  mtls:
    mode: STRICT
```

**서비스 간 요청 인가 (RBAC):** `backend` 서비스에 대한 요청을 `frontend` 서비스만 허용

```yaml
# istio-authz.yaml

apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-only-from-frontend
  namespace: mesh-demo
spec:
  selector:
    matchLabels:
      app: api   # 보호 대상 서비스
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/mesh-demo/sa/frontend-sa
```

> `frontend-sa` 서비스 계정을 생성하고, `frontend` Deployment에 적용해야 합니다. Istio는 **SPIFFE 기반 ID** (예: `cluster.local/ns/<네임스페이스>/sa/<서비스계정>`)를 사용하여 주체를 식별합니다.

### Ingress Gateway를 통한 외부 트래픽 유입

```yaml
# istio-gw.yaml

apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: meshdemo-gw
  namespace: mesh-demo
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-ingress
  namespace: mesh-demo
spec:
  hosts:
    - "*"
  gateways:
    - meshdemo-gw
  http:
    - match:
        - uri:
            prefix: /api
      route:
        - destination:
            host: api.mesh-demo.svc.cluster.local
            port:
              number: 80
```

LoadBalancer IP 또는 NodePort를 통해 `/api` 경로로 요청을 보내 라우팅을 확인합니다.

### 관측: Kiali, Jaeger, Grafana

- `profile=demo`로 설치 시 일부 관측용 컴포넌트가 포함됩니다.
- Kiali 대시보드에서 서비스 의존성 그래프, 지연 시간, 성공률 등을 확인할 수 있습니다.
- Jaeger 또는 Zipkin과 연동하여 분산 트레이싱 헤더(B3/W3C)의 전파를 시각화합니다.

```bash
kubectl -n istio-system port-forward service/kiali 20001:20001
# 브라우저에서 http://localhost:20001 접속
```

---

## Linkerd: 설치, 사이드카 주입, 트래픽 분할, 관측

### 설치 및 사이드카 주입

```bash
# Linkerd CLI 설치 (공식 문서 참조)
curl -sL https://run.linkerd.io/install | sh

# Linkerd 설치
linkerd install | kubectl apply -f -
linkerd check

# 네임스페이스에 사이드카 주입 활성화
kubectl annotate namespace mesh-demo linkerd.io/inject=enabled
kubectl -n mesh-demo rollout restart deployment
```

Pod에 `linkerd-proxy` 컨테이너가 추가되었는지 확인합니다.

### 기본 mTLS와 관측 기능

Linkerd는 **기본적으로 mTLS가 활성화**됩니다. 관측용 플러그인 설치:

```bash
linkerd viz install | kubectl apply -f -
linkerd viz dashboard &
# 브라우저에서 대시보드 확인
```

CLI 기반 실시간 관측:

```bash
linkerd -n mesh-demo stat deployment
linkerd -n mesh-demo top deployment/api-v1
linkerd -n mesh-demo tap deployment/api-v1
```

### 트래픽 분할: SMI TrafficSplit

```yaml
# linkerd-split.yaml

apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: api-split
  namespace: mesh-demo
spec:
  service: api
  backends:
    - service: api-v1
      weight: 90
    - service: api-v2
      weight: 10
```

> `api-v1`, `api-v2`는 별도의 Service로 분리하거나, Headless Service/EndpointSlice 구성을 사용할 수 있습니다. 간단한 실습에서는 `api`라는 단일 서비스에 버전 라벨을 구분하는 대신, 버전별로 서비스를 분리하는 패턴을 권장합니다.

### 회복성: ServiceProfile을 통한 경로별 정책

엔드포인트별 **응답 기대치, 타임아웃, 재시도** 정책을 선언합니다.

```yaml
# linkerd-sp.yaml

apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: api.mesh-demo.svc.cluster.local
  namespace: mesh-demo
spec:
  routes:
    - name: GET_/delay/{sec}
      condition:
        method: GET
        pathRegex: /delay/.*    # httpbin 경로 예시
      isRetryable: true
      timeout: 2s
      retryBudget:
        ttl: 10s
        retryRatio: 0.2
        minRetriesPerSecond: 10
```

적용 후 `linkerd routes` 명령으로 경로별 통계를 확인합니다.

```bash
linkerd -n mesh-demo routes deployment/api-v1
```

---

## 공통 시나리오 구현 패턴

### 카나리 배포 → 점진적 승격

1) 초기 배포: `90/10` 가중치로 시작
2) 적절한 메트릭(지연 시간, 5xx 에러율, SLO 준수율) 모니터링
3) `weight` 값을 `50/50` → `100/0`으로 점진적으로 조정하며 배포 진행
4) 에러율이 급증할 경우 즉시 `rollback` (Istio: VirtualService 수정, Linkerd: TrafficSplit 수정)

### 헤더 기반 라우팅 (A/B 테스트)

- Istio: `match` 섹션의 `headers` 조건을 사용하여 사용자 그룹별 실험 구성
- Linkerd: 기본적으로 TrafficSplit 중심입니다. 헤더 기반 라우팅은 Ingress 컨트롤러나 애플리케이션 계층과 조합하거나, 별도의 통합 컴포넌트를 사용합니다.

### 트래픽 미러링 (Shadowing)

- Istio: `mirror` 및 `mirrorPercentage` 설정을 사용하여 실제 트래픽을 v2 버전에 **복제**합니다. (응답은 v1 버전으로부터 받음)
- Linkerd: 기본 기능으로는 제한적입니다. Ingress 레벨, 프록시 확장, 옵저버블 라우팅을 통해 보완할 수 있습니다.

### 속도 제한 (Rate Limiting)

- Istio: EnvoyFilter 또는 외부 Rate Limiter 서비스와 통합 (예: Envoy Rate Limit Service)
- Linkerd: 외부 정책 엔진, Ingress 컨트롤러, 게이트웨이와 조합하여 구현합니다.

---

## 보안 심화: 인증 및 인가

### 자동 mTLS

- **Istio**: 인증서 발급, 회전, 배포를 자동화합니다. `PeerAuthentication` 리소스로 모드(STRICT, PERMISSIVE, DISABLE)를 설정합니다.
- **Linkerd**: 제어 플레인이 루트 및 중간 인증서를 관리하며, 워크로드 간 자동 mTLS를 제공합니다.

### 인가 (Authorization)

- **Istio**: `AuthorizationPolicy` 리소스를 사용하여 주체(사이드카가 주입한 ID), 경로, 메서드, 헤더 조합으로 세밀한 제어가 가능합니다.
- **Linkerd**: 기본 인가 기능은 제한적입니다. 네임스페이스, NetworkPolicy, Ingress 정책과 병행하여 설계합니다.

---

## 관측 및 모니터링

- 공통점: Prometheus를 통한 지표 수집, Grafana 대시보드
- **Istio**: Kiali(서비스 그래프), Envoy 메트릭 시각화, Jaeger/Zipkin 분산 트레이싱
- **Linkerd**: `linkerd viz`(Tap/Routes/Top), 기본 메트릭이 경량화되고 직관적입니다.

유용한 CLI 명령:

```bash
# Istio

istioctl proxy-status
istioctl pc clusters <pod-name>.mesh-demo
istioctl analyze

# Linkerd

linkerd check
linkerd edges deployment
linkerd diagnostics endpoints api
```

---

## 운영 팁 및 장애 해결

**사이드카가 주입되지 않음**
- **점검 포인트**: 네임스페이스 라벨/Pod 어노테이션, MutatingWebhook 상태
- **대응**: 네임스페이스 라벨 `istio-injection=enabled` 또는 `linkerd.io/inject=enabled` 확인, Pod 재시작

**서비스 간 통신 단절**
- **점검 포인트**: mTLS STRICT 모드 적용, 정책 차단 여부
- **대응**: Istio `PeerAuthentication`/`AuthorizationPolicy` 재검토, Linkerd mTLS 상태 확인

**지연 시간 급증**
- **점검 포인트**: 재시도 폭주, 타임아웃 설정 과도
- **대응**: 재시도 상한/타임아웃 값 튜닝, 이상치 탐지/서킷 브레이커 활성화

**로그/메트릭 데이터 과다**
- **점검 포인트**: 샘플링율, 수집 주기
- **대응**: 트레이싱 샘플링율 축소, Prometheus scrape 간격 조정

**포트 미개방**
- **점검 포인트**: 컨테이너 Port name/type
- **대응**: 포트 이름 `http` 등 명시, Service/VirtualService 포트 일치 확인

**Ingress 접근 불가**
- **점검 포인트**: Gateway/VirtualService 호스트/경로 매칭
- **대응**: 호스트, 경로, 게이트웨이 선택자 재확인

**CPU/메모리 사용량 증가**
- **점검 포인트**: 사이드카/프록시 오버헤드
- **대응**: 리소스 limit/request 조정, 필요한 라우팅만 사용

성능 및 비용 고려사항:
- 사이드카 모델은 Pod당 프록시가 추가되어 **오버헤드**가 발생합니다. 높은 QPS 환경에서는 **프로파일링 및 부하 테스트**를 통해 적절한 컷오프 값을 파악해야 합니다.
- Istio의 **Ambient 모드(사이드카리스)**가 존재합니다. 이는 네임스페이스 또는 노드 레벨 프록시를 통해 오버헤드를 줄이는 접근 방식이지만, 실제 도입 시 버전, 호환성, 기능 범위를 문서를 통해 재확인하는 것을 권장합니다.

---

## 배포 자동화 및 GitOps

- Helm 차트, istioctl 프로필, Linkerd Helm을 **환경별 values 파일**로 관리합니다.
- Argo CD 또는 Flux를 사용하여 **VirtualService, DestinationRule, TrafficSplit**을 선언적으로 운영합니다.
- 카나리 승격 파이프라인: 메트릭 게이팅(에러율/지연/SLO) → 가중치 단계적 상승 → 롤백 조건 설정

---

## 실전 예제 번들

### Istio: 카나리 배포 + 미러링 + 회복성

```yaml
# istio-bundle.yaml

apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api
  namespace: mesh-demo
spec:
  host: api.mesh-demo.svc.cluster.local
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 5s
      baseEjectionTime: 30s
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
  namespace: mesh-demo
spec:
  hosts:
    - api.mesh-demo.svc.cluster.local
  http:
    - match:
        - headers:
            x-exp-group:
              exact: "B"
      route:
        - destination:
            host: api.mesh-demo.svc.cluster.local
            subset: v2
    - route:
        - destination:
            host: api.mesh-demo.svc.cluster.local
            subset: v1
          weight: 90
        - destination:
            host: api.mesh-demo.svc.cluster.local
            subset: v2
          weight: 10
      mirror:
        host: api.mesh-demo.svc.cluster.local
        subset: v2
      timeout: 2s
      retries:
        attempts: 2
        perTryTimeout: 500ms
        retryOn: 5xx,connect-failure,refused-stream
```

테스트:

```bash
# 그룹 B 헤더를 포함하여 v2 버전으로 강제 라우팅

kubectl -n mesh-demo run curler --image=curlimages/curl -it --rm -- \
  sh -lc 'curl -s -H "x-exp-group: B" http://api.mesh-demo.svc.cluster.local/get | jq .headers'
```

### Linkerd: TrafficSplit + ServiceProfile

```yaml
# linkerd-bundle.yaml

apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: api
  namespace: mesh-demo
spec:
  service: api
  backends:
    - service: api-v1
      weight: 90
    - service: api-v2
      weight: 10
---
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: api.mesh-demo.svc.cluster.local
  namespace: mesh-demo
spec:
  routes:
    - name: GET_/status/200
      condition:
        method: GET
        pathRegex: /status/200
      isRetryable: true
      timeout: 1s
      retryBudget:
        ttl: 10s
        retryRatio: 0.2
        minRetriesPerSecond: 5
```

관측:

```bash
linkerd -n mesh-demo stat deployment
linkerd -n mesh-demo routes deployment/api-v1
```

---

## 결론

**서비스 메시**는 **애플리케이션 코드를 수정하지 않고도** 마이크로서비스 네트워킹의 복잡성을 표준화하고 관리하는 강력한 도구입니다.

**Istio**는 가장 풍부한 기능과 세밀한 제어를 제공하며, 엔터프라이즈 수준의 복잡한 요구사항을 충족시킬 수 있습니다. 반면 **Linkerd**는 경량화, 간결성, 안정성에 중점을 두어 빠른 도입과 유지보수가 용이합니다.

실제 환경에 도입할 때는 **요구되는 기능(보안, 트래픽 제어, 관측성, 조직 운영 모델)**, **성능 및 비용 고려사항**, **팀의 기술 역량과 학습 곡선**을 종합적으로 평가하여 결정해야 합니다.

이 문서의 **YAML 구성 파일과 명령어 레시피**를 통해 **카나리 배포 → 점진적 승격**, **mTLS/RBAC 구성**, **재시도/타임아웃/이상치 탐지**, **대시보드 관측**까지의 전체 흐름을 실습해 보면, 운영 환경에 필요한 설계 요소와 고려사항을 명확히 이해할 수 있을 것입니다.

---

## 부록: 자주 사용하는 명령어 메모

```bash
# Istio

istioctl install --set profile=demo -y
kubectl label namespace <네임스페이스> istio-injection=enabled
istioctl proxy-status
istioctl analyze

# Linkerd

linkerd install | kubectl apply -f -
linkerd check
kubectl annotate namespace <네임스페이스> linkerd.io/inject=enabled
linkerd viz install | kubectl apply -f -
linkerd viz dashboard
```

```bash
# 부하 테스트 도구 예시

kubectl -n mesh-demo run hey --image=rakyll/hey -it --rm -- \
  hey -z 30s -q 10 http://api.mesh-demo.svc.cluster.local/status/200
```