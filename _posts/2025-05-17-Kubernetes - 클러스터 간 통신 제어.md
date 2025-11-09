---
layout: post
title: Kubernetes - 클러스터 간 통신 제어
date: 2025-05-11 22:20:23 +0900
category: Kubernetes
---
# 클러스터 간 통신 제어 (Multi-Cluster Communication Control)

현대 인프라는 한 개의 Kubernetes로는 부족하다. **리전/존 장애 격리**, **데이터 주권**, **조직·제품 분리**, **성능/비용 최적화**를 위해 **여러 클러스터**를 병행한다.  
문제는 단순 연결이 아니라, **어떻게 일관된 정책/보안/관측**으로 **안전하고 효율적**인 **서비스-to-서비스 통신**을 제공하느냐다.

---

## 0) 개요 다이어그램

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

## 1) 왜 Multi-Cluster인가?

| 이유 | 설명 | 직접 효과 |
|---|---|---|
| 고가용성(HA) | 다중 리전/존 분산 | 리전 장애에도 서비스 지속 |
| 장애 격리 | Blast radius 축소 | 문제 영향 최소화 |
| 워크로드 분산 | 트래픽/비용/데이터 근접성 | 성능/비용 최적 |
| 조직 분리 | 팀/서비스·규제 단위 분리 | 권한/거버넌스 명확 |
| 데이터 주권 | 지역 내 처리·보관 | 규제 준수 및 운영 유연성 |

---

## 2) 아키텍처 유형과 선택 기준

| 유형 | 통신 레벨 | 요약 | 강점 | 주의 |
|---|---|---|---|---|
| **VPN/Overlay** (WireGuard/IPsec/VXLAN) | L3 | 클러스터 간 IP 라우팅 직접 연결 | 단순·보편, 메쉬 토대 | IP 충돌, 정책/L7 제어는 별도 |
| **Submariner** | L3 | 다중 클러스터 IP 연결·서비스 검색 | 설치 간단, K8s 친화 | L7 정책/관측은 추가 필요 |
| **Cilium ClusterMesh** | L3(+L7) | eBPF 기반 서비스/ID 정합 | 고성능, ID/Policy 일관 | 동일 CNI 계열 선호 |
| **Service Mesh(Istio/Linkerd)** | L7 | mTLS/트래픽 정책/관측 | 보안·관측·정책 일원화 | 배포 복잡도/오버헤드 |
| **API Gateway Relay** | L7 | 외부 GW 통해 라우팅 | 사용자/API 단순화 | 내부 SVC 직접성↓, latency |
| **KubeFed(동기화)** | Ctrl-plane | 리소스 페더레이션 | 배포 정책 일관 | 데이터경로 제어와 별개 |

> **판단 팩터**: _필요한 보안 레벨(mTLS/제로트러스트)_, _L7 트래픽 제어 필요성(Canary, RateLimit)_, _관측 일원화_, _팀 숙련도/운영비용_, _CNI/네트워크 제약_.

---

## 3) 실전 시나리오(예시)

- **cluster-a**: `frontend` 사용자를 수용
- **cluster-b**: `backend` API를 제공
- 목표: a→b **안전(mTLS)** 하고 **정책 기반**으로 **효율적 라우팅**. 장애 시 페일오버.

---

## 4) L3 기반 연결

### 4.1 VPN/WireGuard(개념)
- 노드/게이트웨이 간 터널로 **사설 라우팅**.
- 장점: 단순·범용. 단점: L7 정책/가시성은 별도 구성 필요.

### 4.2 Submariner로 간단 연결
- **서로 다른 Pod CIDR**을 가진 클러스터를 L3로 연결하고, **ServiceExport/Import** 제공.

> 최소 예시(개념 흐름):
```bash
# 두 클러스터 컨텍스트 준비: kctx a, kctx b 가정
# (1) operator 설치
kctx a && kubectl apply -f https://get.submariner.io/operator.yaml
kctx b && kubectl apply -f https://get.submariner.io/operator.yaml
# (2) broker 생성/참여(간단 흐름; 실제는 join 명령/토큰 사용)
# 참조: 공식 문서 워크플로에 따라 broker -> join 적용
```

**서비스 내보내기/가져오기**
```yaml
# cluster-b (backend)에서 export
apiVersion: lighthouse.submariner.io/v2alpha1
kind: ServiceExport
metadata:
  name: backend
  namespace: app
---
# cluster-a에서 자동 ServiceImport 반영되어 DNS 제공 (backend.app.svc.supercluster.local 등)
```

**장점**: IP 라우팅·서비스 검색 자동화. **단점**: L7 보안/정책은 추가(예: 메쉬/프록시) 필요.

---

## 5) L7 기반(서비스 메쉬) — Istio 예제

### 5.1 토폴로지
- **Primary/Remote** 또는 **Replicated Control Plane**.
- 클러스터마다 **East-West Gateway**(크로스 트래픽 진입점) 설치.
- 사이드카 **Envoy**가 **mTLS**, **정책**, **Telemetry** 책임.

### 5.2 설치 요령(요약)
```bash
# istioctl profile demo 기준(개념)
istioctl install --set profile=demo -y
kubectl label namespace app istio-injection=enabled
# EastWest Gateway 설치 템플릿은 공식 예제 참조(클러스터별 hostname 노출)
```

### 5.3 Cross-Cluster 서비스 진입(ServiceEntry)
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

### 5.4 트래픽 라우팅(VirtualService/DestinationRule)
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
      mode: ISTIO_MUTUAL  # mTLS
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
      weight: 10  # 카나리 분배
```

> **장점**: mTLS·정책(인증/인가)·관측(Tracing/Metric)·트래픽 제어 일원화.  
> **주의**: 리소스/운영 복잡도, 사이드카 오버헤드.

---

## 6) Linkerd 멀티클러스터(경량)

- **Linkerd Multicluster**는 **gateway + service mirror**로 단순 연결.
- 기본 **mTLS** 강력, 경량·쉬운 배포. 고급 L7 정책은 Istio 대비 제한적.

**개념 워크플로**
```bash
# 설치(개념)
linkerd install | kubectl apply -f -
linkerd check

# multicluster extension
linkerd multicluster install | kubectl apply -f -
linkerd multicluster link --cluster-name cluster-b | kubectl apply -f -
```

**장점**: 쉬움·경량·기본 보안 강함. **주의**: 고급 트래픽 정책·에코시스템은 Istio에 비해 단순.

---

## 7) Cilium ClusterMesh (eBPF/ID 기반)

- 다중 클러스터를 **ID/정책**으로 정합, **Hubble**로 가시성 탁월.
- **CNI=Cilium** 권장. 레벨3 라우팅 + L7 정책(HTTP/gRPC)도 병행 가능.

**정책 예(클러스터 레이블/서비스 ID 활용, 개념)**
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

## 8) DNS/서비스 디스커버리 전략

| 방법 | 사용처 | 장점 | 주의 |
|---|---|---|---|
| **ExternalName** | 간단 alias | 쉬움 | L7 보안/정책 없음 |
| **CoreDNS Stub/Conditional** | 온프레/내부 도메인 | 제어 용이 | 관리 오버헤드 |
| **Submariner ServiceExport/Import** | MC디스커버리 | 자동화 | Submariner 종속 |
| **Istio Mesh DNS** | Mesh 내 서비스명 통일 | 일원화 | 메쉬 의존 |
| **클라우드 DNS(Private)** | 하이브리드 | 관리형 안정 | 클라우드 종속성 |

**ExternalName 예**
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

## 9) 네트워크 정책(egress/ingress)와 방화벽

MC 환경에서도 **각 클러스터 내부**의 **네트워크 정책**은 기본이다.

**egress allowlist 예(Cluster-A → Cluster-B 대역)**
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
        cidr: 10.100.0.0/16
    ports:
    - protocol: TCP
      port: 8080
```

> 외부 도메인은 **FQDN 정책(Calico/Cilium)**로 제어하면 변경 IP에도 대응 가능.

---

## 10) 보안: mTLS·ID·제로트러스트

- **Service Mesh**: 자동 **mTLS**(TLS 인증서/회전 관리), 정책(RBAC) 적용.
- **SPIFFE/SPIRE**: ID 기반 워크로드 인증서(`spiffe://trust-domain/ns/app/sa/default`).
- **경계 보안**만으론 부족 → **동서(East-West)** 트래픽도 암호화/식별.

**Istio AuthorizationPolicy 예**
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

## 11) 트래픽 전략: 카나리/스플릿/페일오버

### 11.1 Istio(카나리)
앞서 `VirtualService`의 `weight`로 비율 제어.  
**Failover** 예(지역/클러스터 셋업에 따라)
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

### 11.2 Linkerd(Traffic Split)
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

## 12) 관측: 메트릭/로그/트레이싱 일원화

| 영역 | 선택지 | 포인트 |
|---|---|---|
| 메트릭 | **Prometheus + Thanos** | 멀티클러스터 집계/장기보관 |
| 로그 | **Loki** | 테넌트/클러스터 라벨로 구분 |
| 트레이싱 | **Jaeger/Tempo** | Cross-Cluster trace propagation |
| 네트워크 흐름 | **Hubble(Cilium)** | L3/L7 플로우 실시간 |

**Prometheus → Thanos 스케치**
```
Cluster A/B: Prometheus Sidecar ─→ Thanos Store/Query ─→ Grafana
```

---

## 13) SLO/에러 예산 간단 수식

월간 SLO 99.9%의 허용 다운타임(초) 계산:

$$
\text{ErrorBudget(s)} = T_{\text{month}} \times (1 - \text{SLO})
$$

예: 30일 기준 \(T=30\times24\times3600=2{,}592{,}000\)초,  
99.9%면 \(2592000\times0.001=2{,}592\)초 ≈ **43.2분**.

멀티클러스터 **활성-활성**이면 각 클러스터 MTTR/MTBF, 분산 비율을 반영해  
**총 가용성**을 평가하고 **페일오버 목표 RTO/RPO**를 잡는다.

---

## 14) 실습: “Cluster-A에서 Cluster-B API 호출” 3가지 경로

### 14.1 ExternalName(+TLS) — 최소 구성
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-proxy
  namespace: app
spec:
  type: ExternalName
  externalName: api.cluster-b.internal
---
# egress 정책/방화벽으로 해당 FQDN/IP만 허용
```
- 간단하지만, **L7 정책/관측** 미약 → GW/메쉬로 보완 권장.

### 14.2 Submariner ServiceExport/Import — L3 네이티브
```yaml
# cluster-b
apiVersion: lighthouse.submariner.io/v2alpha1
kind: ServiceExport
metadata:
  name: backend
  namespace: app
```
- Cluster-A에서 `backend.app.svc.supercluster.local`로 호출.
- **mTLS/정책은 별도**(메쉬/프록시/방화벽).

### 14.3 Istio East-West — L7 보안/정책 일원화
```yaml
# EastWest Gateway 배치 후
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
- **권장**: L7 제어, mTLS, 관측, 카나리/페일오버까지 일관.

---

## 15) 운영 체크리스트

**네트워크/주소**
- [ ] Pod/Service CIDR 충돌 없는가?
- [ ] 라우팅(overlay/BGP) 경로 정상? MTU 조정 필요?
- [ ] DNS 경로(조건부 포워딩/메쉬 DNS) 설계/검증 완료?

**보안**
- [ ] 사이드카/mTLS 기본값 enforcing?
- [ ] 경계/동서 트래픽 모두 암호화?
- [ ] egress allowlist, FQDN 정책, 방화벽 이중화?

**정책/거버넌스**
- [ ] 네임스페이스·RBAC·네트폴 표준 템플릿?
- [ ] 서비스 간 통신 승인 절차? 변경 감시/감사(audit)?

**관측/알림**
- [ ] Thanos/Loki/Tracing 통합?
- [ ] SLO/에러 예산 정의·대시보드?
- [ ] Cross-Cluster 지연/실패율 알림?

**배포/릴리즈**
- [ ] 카나리/트래픽 스플릿 표준 파이프라인?
- [ ] 롤백/페일오버 플레이북(훈련)?

---

## 16) 트러블슈팅 가이드

| 증상 | 원인 후보 | 확인/조치 |
|---|---|---|
| A→B 연결 타임아웃 | 라우팅/방화벽/DNS | `traceroute`/`dig`/CNI 로그, MTU 1450/1400 조정 |
| 5xx 급증 | 백엔드 과부하/정책 차단 | Mesh metrics(Envoy), Pod HPA/istio OutlierDetection |
| TLS Handshake 실패 | 인증서 도메인/루트 불일치 | Mesh CA/Spiffe ID·SNI 점검 |
| 간헐 지연 증가 | Cross-region path, Nagle/Keepalive | LB 정책/HTTP2/gRPC 설정, locality LB |
| DNS 플랩 | 외부 DNS TTL/캐시 | CoreDNS stub, FQDN 정책 캐시 조정 |
| egress 차단 | NetPolicy/FQDN 룰 | `kubectl describe netpol`, Cilium/Calico 로그 |

---

## 17) 비용/성능 힌트

- **데이터 경로 짧게**: 리전간 왕복 최소화(Geo-routing).
- **L7 정책은 필요한 곳에만**: 사이드카 오버헤드 고려(istio-ambient도 검토).
- **메트릭 보존/샘플링**: Thanos rule/retention, tracing 샘플링률 하향.
- **eBPF(Cilium)**: iptables rule 폭증 회피, 고성능·가시성 확보.

---

## 18) 참조 YAML 모음

### 18.1 A 클러스터 egress allowlist + 메쉬 mTLS
```yaml
# netpol
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
    - ipBlock: { cidr: 10.100.0.0/16 }
    ports:
    - { protocol: TCP, port: 8080 }
---
# istio authz
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: app
spec:
  mtls:
    mode: STRICT
```

### 18.2 Linkerd TrafficSplit(멀티클러스터 import 서비스에 적용)
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

## 19) 결론

- **연결 방식**은 다양하지만, **정책/보안/관측**을 **일관**되게 관리해야 **안전한 멀티클러스터**가 된다.
- 단순 연결(VPN/Submariner) → **L7 보안/정책(메쉬)** → **관측 일원화(Thanos/Loki/Trace)** 순으로 **성숙도**를 올려라.
- **Istio**는 **정책·가시성·트래픽 엔지니어링**이 풍부, **Linkerd**는 **경량·mTLS 기본**. **Cilium ClusterMesh**는 **eBPF/ID 기반 고성능**. 팀 역량과 요구에 맞춰 선택·혼용 가능.
- 성공 키워드: **mTLS 기본값**, **egress allowlist**, **FQDN 정책**, **SLO 기반 운영**, **DR/페일오버 리허설**.

---

## 20) 빠른 실행 요약(메모)

```bash
# 1) 네트워크 연결(택1): VPN/Submariner/Cilium ClusterMesh/메쉬 게이트웨이
# 2) 서비스 디스커버리: ExternalName or Export/Import or Mesh DNS
# 3) 보안: mTLS(메쉬), egress allowlist, FQDN 정책
# 4) 트래픽: 카나리/스플릿/페일오버(메쉬)
# 5) 관측: Thanos/Loki/Tracing/Hubble
# 6) 운영: SLO·플레이북·정기 DR 연습
```

---

## 21) 참고(심화 읽기)
- Istio Multi-Cluster 설치/토폴로지
- Linkerd Multicluster & SMI TrafficSplit
- Cilium ClusterMesh & Hubble
- Submariner ServiceExport/Import
- SRE SLO·에러예산 디자인

본 문서는 실전 위주의 결정포인트·예제·운영 항목을 **한 번에** 담았다.  
이제 당신의 환경/조직 성숙도에 맞춰 **L3→L7까지 단계적으로** 구축해 보라.