---
layout: post
title: Kubernetes - 클러스터 간 통신 제어
date: 2025-05-11 22:20:23 +0900
category: Kubernetes
---
# 클러스터 간 통신 제어: Multi-Cluster Communication Control

현대 클라우드 네이티브 환경에서 단일 Kubernetes 클러스터로는 가용성, 데이터 주권, 조직 분리 등의 요구사항을 충족하기 어렵습니다. 여러 클러스터를 운영하는 멀티클러스터 아키텍처는 필수가 되었으며, 이들 간의 안전하고 효율적인 통신 제어가 중요한 과제로 떠오르고 있습니다.

멀티클러스터 환경에서의 핵심 과제는 단순한 연결을 넘어 일관된 정책, 강력한 보안, 통합된 관측 기능을 제공하는 것입니다.

---

## 아키텍처 개요

```
┌───────────────┐        L3/L7          ┌───────────────┐
│  Cluster A    │  ───────────────▶     │   Cluster B   │
│  (frontend)   │   VPN/Overlay/SubM    │  (backend)    │
│  CNI: Cilium  │   Istio East-West     │  CNI: Calico  │
└───────────────┘   Gateway/API GW      └───────────────┘
        ▲                                         │
        │Observability (Thanos, Loki, Jaeger)     ▼
        └────────────────── Control & Policy ─────┘
```

---

## 멀티클러스터 아키텍처의 필요성

| 목적 | 설명 | 기대 효과 |
|---|---|---|
| 고가용성 | 여러 리전이나 가용성 존에 분산 배포 | 특정 리전 장애 시에도 서비스 지속 운영 |
| 장애 격리 | 영향 범위(Blast radius) 제한 | 문제 발생 시 전체 시스템으로의 확산 방지 |
| 워크로드 분산 | 트래픽, 비용, 데이터 근접성 최적화 | 성능 향상과 비용 효율성 달성 |
| 조직 분리 | 팀, 서비스, 규제 요구사항에 따른 분리 | 권한 관리와 거버넌스의 명확성 확보 |
| 데이터 주권 | 특정 지역 내 데이터 처리 및 보관 | 법적 규제 준수와 운영 유연성 확보 |

---

## 멀티클러스터 통신 아키텍처 유형

| 유형 | 통신 레벨 | 개요 | 강점 | 고려사항 |
|---|---|---|---|---|
| **VPN/Overlay** (WireGuard/IPsec/VXLAN) | L3 | 클러스터 간 직접 IP 라우팅 | 구현이 단순하고 범용적 | L7 정책 제어가 추가로 필요 |
| **Submariner** | L3 | 다중 클러스터 IP 연결 및 서비스 검색 | Kubernetes에 친화적인 설치 | L7 정책 및 관측 기능은 별도 필요 |
| **Cilium ClusterMesh** | L3(+L7) | eBPF 기반 서비스 및 ID 통합 | 고성능, ID 및 정책 일관성 | 동일 CNI 계열 사용이 권장됨 |
| **Service Mesh** (Istio/Linkerd) | L7 | mTLS, 트래픽 정책, 관측 통합 | 보안, 관측, 정책의 일원화 | 배포 복잡도와 오버헤드 고려 |
| **API Gateway Relay** | L7 | 외부 게이트웨이를 통한 라우팅 | 사용자 및 API 관리 단순화 | 내부 서비스 직접성 감소, 지연 증가 |
| **KubeFed** (리소스 동기화) | Control-plane | 리소스 페더레이션 | 배포 정책 일관성 | 데이터 경로 제어와는 별개 |

**선택 기준**: 필요한 보안 수준(mTLS/제로트러스트), L7 트래픽 제어 필요성(카나리, 속도 제한), 관측 통합 요구사항, 팀의 숙련도와 운영 비용, CNI 및 네트워크 제약사항 등을 종합적으로 고려해야 합니다.

---

## 실전 시나리오 예시

**환경 구성:**
- **Cluster A**: 사용자 요청을 처리하는 프론트엔드 서비스
- **Cluster B**: 비즈니스 로직을 처리하는 백엔드 API 서비스

**목표:**
- Cluster A에서 Cluster B로의 안전한 통신(mTLS) 보장
- 정책 기반 효율적 라우팅 구현
- 장애 발생 시 자동 페일오버 지원

---

## L3 기반 연결 방식

### VPN/WireGuard 연결

VPN이나 WireGuard를 사용하여 클러스터 간 터널을 구성하는 방식으로, 노드나 게이트웨이 간의 직접 라우팅을 제공합니다.

**장점**: 구현이 단순하고 범용적으로 적용 가능합니다.
**단점**: L7 수준의 정책 제어와 가시성은 별도로 구성해야 합니다.

### Submariner를 활용한 연결

Submariner는 서로 다른 Pod CIDR을 가진 클러스터를 L3 수준에서 연결하며, ServiceExport/Import 기능을 통해 서비스 검색을 제공합니다.

**서비스 내보내기 예시 (Cluster B):**
```yaml
apiVersion: lighthouse.submariner.io/v2alpha1
kind: ServiceExport
metadata:
  name: backend
  namespace: app
```

Cluster A에서는 자동으로 ServiceImport가 생성되며, `backend.app.svc.supercluster.local` 같은 도메인 이름을 통해 서비스에 접근할 수 있습니다.

**장점**: IP 라우팅과 서비스 검색이 자동화됩니다.
**고려사항**: L7 수준의 보안과 정책 제어는 추가 구성이 필요합니다.

---

## 서비스 메시 기반 접근법

### Istio 멀티클러스터 구성

Istio를 사용하면 여러 클러스터 간의 통신에 대해 일관된 정책, 보안, 관측 기능을 제공할 수 있습니다.

**설치 개요:**
```bash
# Istio 설치 (데모 프로필 기준)
istioctl install --set profile=demo -y

# 네임스페이스에 사이드카 주입 활성화
kubectl label namespace app istio-injection=enabled

# East-West 게이트웨이 설치 (클러스터 간 트래픽 처리)
# 공식 문서의 멀티클러스터 게이트웨이 설정 참조
```

**클러스터 간 서비스 진입점 구성:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: backend-api
  namespace: app
spec:
  hosts:
  - backend.app.global
  location: MESH_INTERNAL
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
```

**트래픽 라우팅 및 보안 정책:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: dr-backend
  namespace: app
spec:
  host: backend.app.global
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL  # mTLS 적용
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: vs-backend
  namespace: app
spec:
  hosts:
  - backend.app.global
  http:
  - route:
    - destination:
        host: backend.app.global
        subset: v1
      weight: 90
    - destination:
        host: backend.app.global
        subset: v2
      weight: 10  # 카나리 배포
```

**장점**: mTLS, 인증/인가 정책, 트레이싱/메트릭, 트래픽 제어를 일원화할 수 있습니다.
**고려사항**: 리소스 소비와 운영 복잡성이 증가할 수 있습니다.

### Linkerd 멀티클러스터 구성

Linkerd는 경량 서비스 메시로, 게이트웨이와 서비스 미러링을 통해 간단한 멀티클러스터 구성을 제공합니다.

**개념적 설치 흐름:**
```bash
# Linkerd 코어 설치
linkerd install | kubectl apply -f -
linkerd check

# 멀티클러스터 확장 설치
linkerd multicluster install | kubectl apply -f -
linkerd multicluster link --cluster-name cluster-b | kubectl apply -f -
```

**장점**: 설치와 운영이 간단하며, 기본적으로 강력한 mTLS를 제공합니다.
**고려사항**: 고급 트래픽 정책과 에코시스템은 Istio에 비해 제한적일 수 있습니다.

### Cilium ClusterMesh 구성

Cilium ClusterMesh는 eBPF와 ID 기반으로 다중 클러스터를 통합하며, Hubble을 통한 뛰어난 가시성을 제공합니다.

**네트워크 정책 예시:**
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: app
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
  - toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/.*"
```

---

## DNS 및 서비스 검색 전략

| 방법 | 사용 사례 | 장점 | 고려사항 |
|---|---|---|---|
| **ExternalName** | 간단한 별칭(alias) 구성 | 구현이 매우 간단함 | L7 보안 및 정책 제어 부족 |
| **CoreDNS 조건부 포워딩** | 온프레미스 및 내부 도메인 관리 | 제어가 용이하고 유연함 | 관리 오버헤드 발생 |
| **Submariner ServiceExport/Import** | 멀티클러스터 서비스 검색 | 자동화된 서비스 검색 제공 | Submariner에 의존적 |
| **Istio Mesh DNS** | 메시 내 서비스 이름 통일 | 일관된 서비스 검색 체계 | 서비스 메시 의존성 |
| **클라우드 프라이빗 DNS** | 하이브리드 클라우드 환경 | 관리형 서비스의 안정성 | 클라우드 공급자 의존성 |

**ExternalName 서비스 예시:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: app
spec:
  type: ExternalName
  externalName: api.cluster-b.internal
```

---

## 네트워크 정책 및 방화벽 구성

멀티클러스터 환경에서도 각 클러스터 내부의 네트워크 정책은 기본적으로 적용되어야 합니다.

**Cluster A에서 Cluster B로의 발신 트래픽 허용 예시:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-to-clusterb
  namespace: app
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - ipBlock:
        cidr: 10.100.0.0/16  # Cluster B의 Pod CIDR
    ports:
    - protocol: TCP
      port: 8080
```

> 외부 도메인에 대한 접근 제어는 Calico나 Cilium의 FQDN 정책을 활용하면 IP 주소 변경에도 유연하게 대응할 수 있습니다.

---

## 보안: mTLS, ID, 제로트러스트

멀티클러스터 환경에서의 보안은 여러 수준에서 고려되어야 합니다:

1. **서비스 메시 기반 mTLS**: Istio나 Linkerd는 자동 mTLS를 제공하여 서비스 간 통신을 암호화합니다.
2. **SPIFFE/SPIRE**: 워크로드 식별 기반 인증서(`spiffe://trust-domain/ns/app/sa/default`)를 통한 정밀한 접근 제어.
3. **동서(East-West) 트래픽 보안**: 경계 보안만으로는 부족하며, 클러스터 내부 트래픽도 암호화와 식별이 필요합니다.

**Istio 인가 정책 예시:**
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-allow-only-frontend
  namespace: app
spec:
  selector:
    matchLabels:
      app: backend
  rules:
  - from:
    - source:
        principals: ["spiffe://mesh.local/ns/app/sa/frontend-sa"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/api/*"]
```

---

## 트래픽 관리 전략

### 카나리 배포 및 트래픽 분할

Istio의 VirtualService를 활용하면 특정 버전의 서비스로 트래픽을 가중치 기반으로 분배할 수 있습니다.

### 페일오버 구성

**Istio DestinationRule을 통한 페일오버 예시:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: dr-backend-failover
  namespace: app
spec:
  host: backend.app.global
  trafficPolicy:
    tls: { mode: ISTIO_MUTUAL }
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 5s
      baseEjectionTime: 30s
    loadBalancer:
      simple: LEAST_CONN
```

### Linkerd 트래픽 분할

```yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: backend-split
  namespace: app
spec:
  service: backend
  backends:
  - service: backend-v1
    weight: 90
  - service: backend-v2
    weight: 10
```

---

## 통합 관측성

멀티클러스터 환경에서의 통합 관측성은 운영 효율성을 결정하는 중요한 요소입니다.

| 관측 영역 | 솔루션 선택 | 핵심 포인트 |
|---|---|---|
| 메트릭 | **Prometheus + Thanos** | 멀티클러스터 메트릭 집계 및 장기 보관 |
| 로그 | **Loki** | 테넌트 및 클러스터 라벨 기반 로그 관리 |
| 분산 추적 | **Jaeger/Tempo** | 클러스터 간 트레이스 전파 및 통합 분석 |
| 네트워크 흐름 | **Hubble(Cilium)** | L3/L7 네트워크 플로우 실시간 모니터링 |

**Thanos 아키텍처 개요:**
```
Cluster A/B: Prometheus Sidecar ─→ Thanos Store/Query ─→ Grafana
```

---

## 서비스 수준 목표(SLO) 관리

서비스의 가용성 목표를 수치화하여 운영하는 것이 중요합니다. 월간 SLO 99.9%의 허용 다운타임을 계산하는 공식은 다음과 같습니다:

```
ErrorBudget(초) = 월간 총 시간(초) × (1 - SLO)
```

예를 들어, 30일 기준 월간 총 시간은 2,592,000초(30일 × 24시간 × 3600초)입니다.
99.9% SLO의 경우: 2,592,000 × 0.001 = 2,592초 ≈ **43.2분**의 허용 다운타임이 있습니다.

멀티클러스터 활성-활성 구성에서는 각 클러스터의 평균 수리 시간(MTTR)과 평균 고장 간격(MTBF)을 고려하여 총 가용성을 평가하고, 페일오버 목표인 복구 시간 목표(RTO)와 복구 지점 목표(RPO)를 설정해야 합니다.

---

## 실전 구현 패턴

### 패턴 1: 최소 구성 (ExternalName 서비스)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-proxy
  namespace: app
spec:
  type: ExternalName
  externalName: api.cluster-b.internal
```

**특징**: 구현이 간단하지만, L7 정책과 관측 기능이 제한적입니다.

### 패턴 2: Submariner 기반 L3 네이티브 연결

Cluster B에서 ServiceExport를 생성하면 Cluster A에서 `backend.app.svc.supercluster.local` 도메인을 통해 접근할 수 있습니다.

**장점**: IP 라우팅과 서비스 검색이 자동화됩니다.
**고려사항**: mTLS와 L7 정책은 별도로 구성해야 합니다.

### 패턴 3: Istio East-West 게이트웨이 기반 L7 통합

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: backend-ew
  namespace: app
spec:
  hosts: [ "backend.app.global" ]
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend-ew-dr
  namespace: app
spec:
  host: backend.app.global
  trafficPolicy:
    tls: { mode: ISTIO_MUTUAL }
```

**권장사항**: L7 제어, mTLS, 관측, 카나리/페일오버를 일관되게 관리할 수 있는 가장 포괄적인 접근 방식입니다.

---

## 운영 모범 사례

### 네트워크 및 주소 관리
- Pod 및 Service CIDR이 충돌하지 않는지 확인
- 라우팅 경로(overlay/BGP)가 정상적으로 구성되었는지 검증
- DNS 경로(조건부 포워딩/메시 DNS)가 적절히 설계되고 검증되었는지 확인

### 보안 구성
- 사이드카 주입과 mTLS가 기본적으로 적용되는지 확인
- 경계 트래픽과 동서 트래픽 모두 암호화되어 있는지 검증
- 발신 트래픽 허용 목록, FQDN 정책, 방화벽 이중화가 구성되었는지 확인

### 정책 및 거버넌스
- 네임스페이스, RBAC, 네트워크 정책에 대한 표준 템플릿을 보유
- 서비스 간 통신에 대한 승인 절차를 수립
- 변경 사항에 대한 감사(audit) 로깅 구성

### 관측 및 알림
- Thanos, Loki, 분산 추적 시스템이 통합되어 있는지 확인
- SLO와 에러 예산이 정의되고 대시보드로 시각화되었는지 확인
- 클러스터 간 지연 시간과 실패율에 대한 알림이 구성되었는지 확인

### 배포 및 릴리스 관리
- 카나리 배포와 트래픽 분할을 위한 표준 파이프라인 구축
- 롤백 및 페일오버 절차에 대한 플레이북 작성 및 정기적인 훈련 수행

---

## 문제 해결 가이드

| 증상 | 가능한 원인 | 확인 및 조치 방법 |
|---|---|---|
| 클러스터 간 연결 타임아웃 | 라우팅, 방화벽, DNS 문제 | `traceroute`, `dig`, CNI 로그 확인, MTU 1450/1400으로 조정 |
| 5xx 오류 급증 | 백엔드 과부하 또는 정책 차단 | 서비스 메시 메트릭(Envoy), Pod HPA, Istio OutlierDetection 확인 |
| TLS 핸드셰이크 실패 | 인증서 도메인 또는 루트 불일치 | 메시 CA, SPIFFE ID, SNI 설정 점검 |
| 간헐적 지연 증가 | 리전 간 경로 문제, Nagle/Keepalive 설정 | 로드 밸런서 정책, HTTP2/gRPC 설정, 지역 기반 로드 밸런싱 확인 |
| DNS 불안정 | 외부 DNS TTL/캐시 문제 | CoreDNS stub 설정, FQDN 정책 캐시 조정 |
| 발신 트래픽 차단 | 네트워크 정책 또는 FQDN 규칙 문제 | `kubectl describe netpol` 실행, Cilium/Calico 로그 확인 |

---

## 비용 및 성능 최적화

1. **데이터 경로 최적화**: 리전 간 왕복 시간을 최소화하기 위해 지리적 라우팅 적용
2. **L7 정책의 선택적 적용**: 사이드카 오버헤드를 고려하여 필요한 곳에만 L7 정책 적용 (Istio Ambient 모드 검토)
3. **관측 데이터 관리**: Thanos 보존 정책, 트레이싱 샘플링률 조정으로 스토리지 비용 최적화
4. **eBPF 활용**: Cilium과 같은 eBPF 기반 솔루션으로 iptables 규칙 폭증 문제 해결 및 고성능 달성

---

## 참조 구성 예시

### 클러스터 A 발신 트래픽 허용 목록 + mTLS

```yaml
# 네트워크 정책
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-backend-8080
  namespace: app
spec:
  podSelector:
    matchLabels: { app: frontend }
  policyTypes: [Egress]
  egress:
  - to:
    - ipBlock: { cidr: 10.100.0.0/16 }  # Cluster B CIDR
    ports:
    - { protocol: TCP, port: 8080 }
---
# Istio PeerAuthentication
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: app
spec:
  mtls:
    mode: STRICT
```

### Linkerd 트래픽 분할 (멀티클러스터)

```yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: backend-split
  namespace: app
spec:
  service: backend
  backends:
  - { service: backend-remote, weight: 50 }
  - { service: backend-local,  weight: 50 }
```

---

## 결론

멀티클러스터 Kubernetes 환경을 효과적으로 운영하기 위해서는 다음과 같은 원칙을 준수해야 합니다:

1. **적절한 연결 방식 선택**: 조직의 요구사항과 성숙도에 따라 VPN/Submariner(단순 연결) → 서비스 메시(L7 제어) → 통합 관측성(Thanos/Loki/추적) 순으로 진화하세요.

2. **보안 기본값 설정**: 모든 트래픽에 mTLS를 기본으로 적용하고, 발신 트래픽은 허용 목록 방식으로 제어하며, FQDN 정책으로 유연한 접근 제어를 구현하세요.

3. **일관된 관측 체계 구축**: 메트릭, 로그, 트레이싱을 통합하여 클러스터 간 문제를 빠르게 진단하고 해결할 수 있는 환경을 조성하세요.

4. **운영 자동화 및 훈련**: SLO 기반 운영 체계를 수립하고, 재해 복구 및 페일오버 절차를 정기적으로 훈련하여 실제 장애 시 신속히 대응할 수 있도록 준비하세요.

멀티클러스터 환경은 복잡성을 증가시키지만, 적절한 아키텍처와 도구를 선택하고 체계적으로 운영한다면 뛰어난 가용성, 보안성, 유연성을 제공할 수 있습니다. 이 문서에서 제시한 패턴과 모범 사례를 참고하여 조직에 맞는 멀티클러스터 통신 제어 전략을 수립하시기 바랍니다.