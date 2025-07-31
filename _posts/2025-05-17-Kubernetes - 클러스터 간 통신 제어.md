---
layout: post
title: Kubernetes - 클러스터 간 통신 제어
date: 2025-05-11 22:20:23 +0900
category: Kubernetes
---
# 클러스터 간 통신 제어 (Multi-Cluster Communication Control)

현대적인 인프라에서는 **여러 개의 Kubernetes 클러스터**를 사용하는 경우가 많습니다.  
이때 중요한 것은 클러스터 간의 **서비스 통신을 어떻게 안전하고 효율적으로 제어**할 것인가입니다.

---

## ✅ 왜 Multi-Cluster 환경이 필요한가?

| 이유 | 설명 |
|------|------|
| 고가용성 | 여러 리전에 클러스터를 분산 운영 |
| 장애 격리 | 한 클러스터 장애 시 다른 클러스터로 전환 |
| 워크로드 분산 | 트래픽이나 리소스에 따라 작업 분배 |
| 조직 분리 | 팀/제품 단위로 클러스터 분리 |
| 데이터 주권 | 규제 상 특정 지역 내 데이터 운영 필요

---

## ✅ Multi-Cluster 아키텍처 유형

| 유형 | 설명 | 통신 방식 |
|------|------|------------|
| **Federation (KubeFed)** | 클러스터 간 리소스 동기화 | Control-plane 중심 |
| **API Gateway Relay** | 클러스터 외부에 API Gateway 배치 | L7 레벨 |
| **VPN/Overlay 네트워크** | 클러스터 간 네트워크 연결 (WireGuard, VXLAN 등) | L3 레벨 |
| **Service Mesh (Istio, Linkerd)** | 클러스터 간 mTLS, 정책 통제 | L7 + 보안 |
| **Direct Pod-to-Pod (CNI 확장)** | Cilium, Submariner 등으로 IP 간 라우팅 | L3 직접 연결 |

---

## ✅ 실전 구성 예시: 2개 클러스터 간 통신

### 예시 시나리오

- `cluster-a`: 사용자 요청을 받는 프론트엔드
- `cluster-b`: 백엔드 API 서버

→ 클러스터 간 **service-to-service 통신 필요**

---

## ✅ 1. 클러스터 간 네트워크 연결

### 옵션 1: VPN / WireGuard

```bash
# 두 클러스터 노드 간의 사설 네트워크 구성
```

- 직접 IP 통신 가능 (단, IP 중복 피해야 함)
- Calico, Cilium 같은 CNI가 라우팅 테이블 통합 필요

### 옵션 2: Submariner 사용

[Submariner](https://submariner.io/)는 서로 다른 클러스터 간 L3 연결을 위한 CNCF 프로젝트입니다.

```bash
# Cluster A와 B에 각각 submariner operator 설치
```

- 자동 라우팅, 서비스 검색, 격리 가능

---

## ✅ 2. 클러스터 간 서비스 디스커버리

| 방법 | 설명 |
|------|------|
| ExternalName | DNS alias로 외부 서비스 매핑 |
| Headless Service + CoreDNS | 수동 DNS 라우팅 |
| ServiceExport / Import | Submariner, Istio에서 공식 지원 |

### 예시: ExternalName으로 cross-cluster 요청

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ExternalName
  externalName: api.cluster-b.internal
```

→ 클러스터 A에서 `backend-svc`로 호출 시, cluster B의 API로 리디렉션

---

## ✅ 3. 인증 및 보안 통제

클러스터 간 트래픽은 외부망을 지날 수 있어 **mTLS**, **정책 제어**, **네트워크 필터링**이 중요합니다.

### mTLS를 통한 안전한 통신

- Istio / Linkerd 등의 **Service Mesh**를 통해 자동으로 mTLS 적용
- 인증서 기반 트래픽 암호화 + 인증

### NetworkPolicy로 통신 제어

각 클러스터 내에서 egress, ingress 제어 가능:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-to-clusterb
  namespace: default
spec:
  podSelector: {}
  egress:
  - to:
    - ipBlock:
        cidr: 10.100.0.0/16  # cluster B 대역
    ports:
    - protocol: TCP
      port: 8080
  policyTypes:
  - Egress
```

---

## ✅ 4. Istio를 활용한 클러스터 간 통신

Istio는 클러스터 간 통신에서 **인증/라우팅/정책**을 통합 관리합니다.

### 구성 방식

- **Primary-Remote 모델** (control-plane 공유)
- **Replicated control-plane 모델** (서로 별도)

### 필수 구성 요소

- `EastWest Gateway` 설치
- `ServiceEntry`, `VirtualService` 구성
- `ServiceExport`, `DestinationRule` 구성

### 예시: cluster-a에서 cluster-b로 API 라우팅

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: backend-api
spec:
  hosts:
  - api.clusterb.svc.global
  location: MESH_INTERNAL
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
```

---

## ✅ 5. 관측 및 디버깅

| 도구 | 역할 |
|------|------|
| Prometheus + Thanos | 멀티 클러스터 모니터링 |
| Grafana | 클러스터별 메트릭 시각화 |
| Loki | 클러스터별 로그 통합 |
| Hubble (Cilium) | 흐름 추적 |
| Jaeger / Tempo | 클러스터 간 트레이싱 (Istio) |

---

## ✅ 보안 고려 사항

- [x] 클러스터 간 인증서 교환 여부
- [x] 외부 노출 API의 Access 제어 (OAuth, mTLS 등)
- [x] DNS 스푸핑 및 IP spoofing 방지
- [x] 노드/Pod 간 통신 암호화
- [x] 정책 기반 허용 리스트 사용 (allowlist)

---

## ✅ 결론

| 항목 | 설명 |
|------|------|
| 연결 방식 | VPN, Overlay, Submariner, Service Mesh 등 |
| 통신 대상 | Service / Pod / ExternalName |
| 보안 수단 | mTLS, 인증서, 네트워크 정책 |
| 서비스 검색 | DNS, Headless, Mesh DNS |
| 운영 도구 | Istio, Submariner, Cilium, Prometheus |

> 클러스터 간 통신은 고가용성과 멀티리전에 필수적입니다.  
> 통신은 단순 연결보다 **정책, 인증, 모니터링**까지 포함되어야 안전한 구조입니다.

---

## ✅ 참고 링크

- [Istio Multi-Cluster 공식 문서](https://istio.io/latest/docs/setup/install/multicluster/)
- [Cilium ClusterMesh 소개](https://docs.cilium.io/en/stable/network/clustermesh/)
- [Submariner 공식 문서](https://submariner.io/)
- [Kubernetes Federation v2](https://github.com/kubernetes-sigs/kubefed)