---
layout: post
title: Kubernetes - Service Mesh
date: 2025-05-11 19:20:23 +0900
category: Kubernetes
---
# Service Mesh란? Istio vs Linkerd 개요

Kubernetes에서 마이크로서비스를 운영하다 보면 다음 요구가 반복된다.

- 서비스 간 트래픽을 **관찰**하고 지연/에러율/호출 경로를 **가시화**
- **Retry, Timeout, Circuit Breaker** 같은 네트워크 회복성 표준화
- **Canary, A/B, 트래픽 미러링** 등 점진 배포
- **mTLS**로 서비스 간 **암호화/상호 인증** 및 정책 기반 제어

이 기능을 **애플리케이션 코드에 넣지 않고**, **인프라 레이어**에서 일괄 적용하는 것이 **Service Mesh**다.

---

## 1) Service Mesh 핵심 개념

### Data Plane vs Control Plane

- **Data Plane**: 각 Pod 옆에 붙는 **사이드카 프록시**가 실데이터 트래픽을 처리  
  - 대표: Envoy(Istio), linkerd-proxy(Linkerd, Rust 기반)
- **Control Plane**: 선언된 정책/구성(메쉬 리소스)을 수집·검증하고, Data Plane으로 **전파/동기화**

### 제공 기능(요약)

| 범주 | 대표 기능 |
|---|---|
| 관측성(Observability) | 요청 지연/성공률/에러율, 히스토그램, 서비스 그래프, 분산 트레이싱 |
| 트래픽 관리 | 라우팅(Host/Path/헤더), Canary/A-B, 서킷브레이커, 제한, 미러링 |
| 보안(Security) | 자동 **mTLS**, ID 기반 정책, Peer/Request 인증, 인가(RBAC) |
| 회복성(Reliability) | **Retry**, **Timeout**, **Outlier Detection**(불량 인스턴스 격리) |
| 정책(Policy) | 서비스 간 통신 허용/차단, 인증서 롤링, Rate limit(확장) |

---

## 2) 대표 Mesh: Istio vs Linkerd 한눈 비교

| 항목 | **Istio** | **Linkerd** |
|---|---|---|
| 프록시 | Envoy (C++), 고기능 L7 | linkerd-proxy (Rust), 경량 |
| 복잡도 | 높음(유연성·기능 풍부) | 낮음(간결·안정성 중점) |
| 설치 | `istioctl`/Helm/Operator | 링크드 전용 CLI/Helm |
| 트래픽 제어 | 세밀(가중 배분, 헤더/쿠키 기반, 미러링 등) | TrafficSplit 중심(간결) |
| 보안 | mTLS 자동화 + 세밀한 AuthZ | 기본 mTLS, 간단 설정 |
| 관측 | Kiali, Prom/Jae/Zipkin 연동 | `linkerd viz`, Prom 연동 |
| 멀티클러스터 | 풍부한 패턴/옵션 | 간결한 멀티클러스터 |
| 학습 곡선 | 가파름 | 완만 |

> 선택 가이드: **엔터프라이즈급 세밀 제어/확장성** → Istio. **경량/간결/빠른 도입** → Linkerd.

---

## 3) 공통 실습 환경 준비

```bash
# 클러스터가 없다면 예: kind
kind create cluster --name mesh-lab

# 네임스페이스 준비
kubectl create ns mesh-demo
kubectl label ns mesh-demo istio-injection=enabled   # Istio 사용 시
```

테스트 워크로드(버전 2개):

```yaml
# app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v1
  namespace: mesh-demo
spec:
  replicas: 2
  selector: { matchLabels: { app: api, version: v1 } }
  template:
    metadata: { labels: { app: api, version: v1 } }
    spec:
      containers:
        - name: api
          image: kennethreitz/httpbin
          ports: [{ containerPort: 80, name: http }]
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: mesh-demo
spec:
  selector: { app: api }      # v1, v2 모두 묶음
  ports: [{ name: http, port: 80, targetPort: http }]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v2
  namespace: mesh-demo
spec:
  replicas: 1
  selector: { matchLabels: { app: api, version: v2 } }
  template:
    metadata: { labels: { app: api, version: v2 } }
    spec:
      containers:
        - name: api
          image: kennethreitz/httpbin
          ports: [{ containerPort: 80, name: http }]
```

```bash
kubectl apply -f app.yaml
```

---

## 4) Istio: 설치, 주입, 트래픽 제어, 보안, 관측

### 4.1 설치와 사이드카 주입

```bash
# istioctl 다운로드(공식 가이드 참고)
istioctl install --set profile=demo -y
kubectl label ns mesh-demo istio-injection=enabled
# 기존 Pod는 재시작 필요
kubectl -n mesh-demo rollout restart deploy
```

확인:

```bash
kubectl -n istio-system get pods
kubectl -n mesh-demo get pods -o jsonpath='{..containers[*].name}' | tr ' ' '\n' | sort | uniq
# 각 Pod에 'istio-proxy'가 보여야 함
```

### 4.2 기본 라우팅: VirtualService + DestinationRule

**목표:** `api` 서비스로 들어온 트래픽을 `v1:90%`, `v2:10%`로 분배(카나리)

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
      labels: { version: v1 }
    - name: v2
      labels: { version: v2 }
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-vs
  namespace: mesh-demo
spec:
  hosts: ["api.mesh-demo.svc.cluster.local"]
  http:
    - route:
        - destination: { host: api.mesh-demo.svc.cluster.local, subset: v1 }
          weight: 90
        - destination: { host: api.mesh-demo.svc.cluster.local, subset: v2 }
          weight: 10
```

```bash
kubectl apply -f istio-traffic.yaml
```

부하 발생(내부에서):

```bash
kubectl -n mesh-demo run load --image=curlimages/curl -it --rm -- \
  sh -lc 'for i in $(seq 1 50); do curl -s http://api.mesh-demo.svc.cluster.local/status/200 | wc -c; done'
```

> httpbin은 응답 헤더/바디에서 버전 표시가 없으니, 실제 서비스라면 응답에 버전 로그를 추가하거나, 헤더 기반 라우팅 실험을 위한 커스텀 앱을 권장.

### 4.3 회복성: Retry, Timeout, Outlier Detection

```yaml
# istio-resilience.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-resilient
  namespace: mesh-demo
spec:
  hosts: ["api.mesh-demo.svc.cluster.local"]
  http:
    - timeout: 2s
      retries:
        attempts: 3
        perTryTimeout: 500ms
        retryOn: gateway-error,connect-failure,refused-stream,5xx
      route:
        - destination: { host: api.mesh-demo.svc.cluster.local, subset: v1 }
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

적용 후, 장애 인스턴스가 연속 5xx를 반환하면 자동 격리된다.

### 4.4 mTLS와 권한(AuthorizationPolicy)

**전역 mTLS** 엄격(STRICT):

```yaml
# istio-mtls.yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: mesh-demo
spec:
  mtls:
    mode: STRICT
```

**요청 인가(서비스 간 RBAC)**: `backend`로의 요청을 `frontend`만 허용

```yaml
# istio-authz.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-only-from-frontend
  namespace: mesh-demo
spec:
  selector:
    matchLabels: { app: api }   # 보호 대상
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/mesh-demo/sa/frontend-sa"]
```

> `frontend-sa` 서비스어카운트를 만든 뒤, `frontend` Deployment에 적용해야 한다. Istio는 **SPIFFE 기반 ID**(예: `cluster.local/ns/<ns>/sa/<sa>`)로 주체를 식별한다.

### 4.5 Ingress 게이트웨이로 외부 유입

```yaml
# istio-gw.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: meshdemo-gw
  namespace: mesh-demo
spec:
  selector: { istio: ingressgateway }
  servers:
    - port: { number: 80, name: http, protocol: HTTP }
      hosts: ["*"]
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-ingress
  namespace: mesh-demo
spec:
  hosts: ["*"]
  gateways: ["meshdemo-gw"]
  http:
    - match: [{ uri: { prefix: "/api" } }]
      route:
        - destination: { host: api.mesh-demo.svc.cluster.local, port: { number: 80 } }
```

LoadBalancer IP 또는 NodePort로 접근하여 `/api` 경로 요청 라우팅을 확인한다.

### 4.6 관측: Kiali/Jaeger/Grafana

- `profile=demo` 설치 시 일부 컴포넌트 포함.  
- Kiali 대시보드에서 서비스 그래프, 지연/성공률 확인.  
- Jaeger/Zipkin 연동으로 트레이싱 헤더(B3/W3C) 전파·시각화.

```bash
kubectl -n istio-system port-forward svc/kiali 20001:20001
# 브라우저 http://localhost:20001
```

---

## 5) Linkerd: 설치, 주입, 트래픽 분할, 관측

### 5.1 설치와 주입

```bash
curl -sL https://run.linkerd.io/install | sh
linkerd install | kubectl apply -f -
linkerd check
```

네임스페이스 주입:

```bash
kubectl annotate ns mesh-demo linkerd.io/inject=enabled
kubectl -n mesh-demo rollout restart deploy
```

Pod에 `linkerd-proxy`가 붙었는지 확인한다.

### 5.2 기본 mTLS와 가시성

Linkerd는 **기본 mTLS**가 활성화된다. 관측 플러그인 설치:

```bash
linkerd viz install | kubectl apply -f -
linkerd viz dashboard &
# 브라우저 대시보드 확인
```

CLI 기반 실시간 관측:

```bash
linkerd -n mesh-demo stat deploy
linkerd -n mesh-demo top deploy/api-v1
linkerd -n mesh-demo tap deploy/api-v1
```

### 5.3 트래픽 분할: SMI TrafficSplit

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

> `api-v1`, `api-v2`는 Service로도 분리하거나, Headless/EndpointSlice 구성을 사용할 수 있다. 간단 실습에선 `api` 단일 서비스에 버전 라벨을 나누는 대신, 버전별 서비스를 두는 패턴을 권장.

### 5.4 회복성: ServiceProfile로 라우트 별 정책

엔드포인트별 **응답 기대치/타임아웃/재시도**를 선언:

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

적용 후 `linkerd routes`로 라우트 통계를 확인:

```bash
linkerd -n mesh-demo routes deploy/api-v1
```

---

## 6) 공통 시나리오 레시피

### 6.1 Canary → 점진 승격

1) 초기 `90/10`  
2) 적절한 메트릭(지연, 5xx, SLO 위반) 모니터링  
3) `weight`를 50/50 → 100/0으로 늘리며 점진 배포  
4) 에러율 급증 시 즉시 `rollback`(Istio: VirtualService 수정, Linkerd: TrafficSplit 수정)

### 6.2 헤더 기반 라우팅(A/B)

- Istio: `match`의 `headers` 조건으로 사용자 그룹 실험  
- Linkerd: 기본은 TrafficSplit 위주. 헤더 기반 라우팅은 Ingress/애플리케이션 계층과 조합하거나, 별도 통합 컴포넌트 사용

### 6.3 트래픽 미러링(Shadow)

- Istio: `mirror`/`mirrorPercentage`로 실제 트래픽을 v2에 **복제**(응답은 v1으로)  
- Linkerd: 기본 기능으로는 제한적. Ingress 레벨/프록시 확장/옵저버블 라우팅으로 보완

### 6.4 Rate Limit

- Istio: EnvoyFilter/외부 Rate Limiter와 통합(예: envoy rate limit service)  
- Linkerd: 외부 정책 엔진/Ingress/게이트웨이와 조합

---

## 7) 보안 심화: 인증/인가

### 7.1 자동 mTLS

- **Istio**: 인증서 발급/회전/배포 자동화. `PeerAuthentication`으로 모드 설정(STRICT, PERMISSIVE, DISABLE)  
- **Linkerd**: Control Plane가 루트/중간 인증서를 관리, 워크로드 간 자동 mTLS

### 7.2 인가(Authorization)

- **Istio**: `AuthorizationPolicy`로 주체(사이드카가 주입한 ID), 경로, 메서드, 헤더 조합으로 세밀 제어  
- **Linkerd**: 기본 인가는 제한적. 네임스페이스/NetworkPolicy/Ingress 정책과 병행 설계

---

## 8) 관측/모니터링

- 공통: Prometheus로 지표 수집, Grafana 대시보드
- **Istio**: Kiali(서비스 그래프), Envoy 가시화, Jaeger/Zipkin 분산 트레이싱  
- **Linkerd**: `linkerd viz`(Tap/Routes/Top), 기본 지표가 경량·직관적

유용한 CLI:

```bash
# Istio
istioctl proxy-status
istioctl pc clusters <pod>.mesh-demo
istioctl analyze

# Linkerd
linkerd check
linkerd edges deploy
linkerd diagnostics endpoints api
```

---

## 9) 운영 팁과 장애 트러블슈팅

| 증상 | 점검 포인트 | 대응 |
|---|---|---|
| 사이드카 미주입 | NS 라벨/Pod 주석, MutatingWebhook 상태 | NS 라벨 `istio-injection=enabled` 또는 `linkerd.io/inject=enabled` 확인, Pod 재시작 |
| 서비스 통신 단절 | mTLS STRICT/정책 차단 | Istio `PeerAuthentication/AuthorizationPolicy` 재검토, Linkerd mTLS 상태 확인 |
| 지연 급증 | Retry 폭주/타임아웃 과도 | 재시도 상한/timeout 튜닝, OutlierDetection/서킷브레이커 활성화 |
| 로그/메트릭 과다 | 샘플링/수집 주기 | 트레이싱 샘플링율 축소, Prom scrape 간격 조정 |
| 포트 안 열림 | 컨테이너 Port name/type | 포트 이름 `http` 등 명시, Service/VS port 일치 |
| Ingress 접근 불가 | Gateway/VirtualService 매칭 | 호스트/경로/게이트웨이 선택자 재확인 |
| CPU/메모리 상승 | 사이드카/프록시 오버헤드 | 리소스 리밋/리퀘스트 조정, 필요 라우팅만 사용 |

성능/비용 고려:

- 사이드카 모델은 Pod당 프록시가 붙어 **오버헤드**가 있다. 고QPS 환경에서는 **프로파일링/부하 테스트**로 컷오프치를 파악.
- Istio의 **Ambient(사이드카리스)** 모드가 존재한다. 네임스페이스/노드 레벨 프록시로 오버헤드를 줄이는 접근이지만, 실제 도입 시 버전/호환성/기능 범위를 문서로 재확인 권장.

---

## 10) 배포 자동화와 GitOps

- 헬름 차트/istioctl 프로필/Linkerd Helm을 **환경별 values**로 관리  
- Argo CD/Flux로 **VirtualService/DestinationRule/TrafficSplit**을 선언적으로 운영  
- 카나리 프로모션 파이프라인: 메트릭 게이팅(에러율/지연/SLO) → 가중치 단계 상승 → 롤백 조건

---

## 11) 실전 예제 번들

### 11.1 Istio: 카나리 + 미러링 + 회복성

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
      labels: { version: v1 }
    - name: v2
      labels: { version: v2 }
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
  hosts: ["api.mesh-demo.svc.cluster.local"]
  http:
    - match:
        - headers:
            x-exp-group:
              exact: "B"
      route:
        - destination: { host: api.mesh-demo.svc.cluster.local, subset: v2 }
    - route:
        - destination: { host: api.mesh-demo.svc.cluster.local, subset: v1 }
          weight: 90
        - destination: { host: api.mesh-demo.svc.cluster.local, subset: v2 }
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
# 그룹 B 헤더로 강제 v2 라우팅
kubectl -n mesh-demo run curler --image=curlimages/curl -it --rm -- \
  sh -lc 'curl -s -H "x-exp-group: B" http://api.mesh-demo.svc.cluster.local/get | jq .headers'
```

### 11.2 Linkerd: TrafficSplit + ServiceProfile

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
      condition: { method: GET, pathRegex: /status/200 }
      isRetryable: true
      timeout: 1s
      retryBudget: { ttl: 10s, retryRatio: 0.2, minRetriesPerSecond: 5 }
```

관측:

```bash
linkerd -n mesh-demo stat deploy
linkerd -n mesh-demo routes deploy/api-v1
```

---

## 12) 결론

- **Service Mesh**는 **코드 수정 없이** 마이크로서비스 네트워킹을 표준화한다.  
- **Istio**는 기능이 가장 풍부하고 세밀한 제어가 가능하며, **Linkerd**는 경량/간결성/안정성에 강점이 있다.  
- 실제 도입은 **요구 기능(보안/트래픽 제어/관측/조직 운영 모델)**, **성능/비용**, **팀의 역량/학습 곡선**을 종합 고려해 결정한다.  
- 본 문서의 **YAML/명령 레시피**를 토대로 **카나리→승격**, **mTLS/RBAC**, **Retry/Timeout/Outlier**, **대시보드 관측**까지 한 흐름으로 실습해보면, 운영 환경에 필요한 설계 포인트가 선명해진다.

---

## 부록: 자주 쓰는 명령 메모

```bash
# Istio
istioctl install --set profile=demo -y
kubectl label ns <ns> istio-injection=enabled
istioctl proxy-status
istioctl analyze

# Linkerd
linkerd install | kubectl apply -f -
linkerd check
kubectl annotate ns <ns> linkerd.io/inject=enabled
linkerd viz install | kubectl apply -f -
linkerd viz dashboard
```

```bash
# 부하 도구 예시
kubectl -n mesh-demo run hey --image=rakyll/hey -it --rm -- \
  hey -z 30s -q 10 http://api.mesh-demo.svc.cluster.local/status/200
```