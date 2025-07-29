---
layout: post
title: Kubernetes - Service Mesh
date: 2025-05-05 19:20:23 +0900
category: Kubernetes
---
# Service Mesh란? Istio vs Linkerd 개요

Kubernetes에서 마이크로서비스를 운영하다 보면 다음과 같은 문제가 발생합니다:

- 서비스 간 통신을 **암호화하거나 모니터링**하고 싶다
- **Retry, Timeout, Circuit Breaker** 같은 네트워크 정책을 일관성 있게 적용하고 싶다
- 트래픽을 **버전별로 라우팅 (A/B 테스트, Canary 배포)**하고 싶다
- 서비스 간 **통신 보안 (mTLS)**을 설정하고 싶다

이러한 문제를 **애플리케이션 코드에 직접 구현하지 않고**,  
**인프라 계층에서 일괄적으로 처리**하게 해주는 것이 바로 **Service Mesh**입니다.

---

## ✅ Service Mesh란?

Service Mesh는 마이크로서비스 간 통신을 제어하고 관찰할 수 있는 **인프라 레이어**입니다.  
기능은 보통 **사이드카(Proxy)**로 동작하며, 각 Pod에 주입되어 트래픽을 가로채고 관리합니다.

### 핵심 개념

- **Data Plane**: 실제 트래픽을 처리하는 프록시 (ex. Envoy, Linkerd-proxy)
- **Control Plane**: 정책을 정의하고 프록시들을 제어하는 중앙 컴포넌트

---

## ✅ Service Mesh가 제공하는 주요 기능

| 기능 | 설명 |
|------|------|
| **Observability (가시성)** | 트래픽 흐름, 요청 지연 시간, 성공률, 로그, 트레이싱 등 시각화 |
| **Traffic Management** | 라우팅, A/B 테스트, Canary 배포, Rate Limit 등 |
| **Security** | mTLS(서비스 간 암호화), 인증/인가 정책 |
| **Reliability** | Retry, Timeout, Circuit Breaker 등 네트워크 회복성 |
| **Policy Control** | RBAC, IP 기반 제한, 서비스 간 통신 허용 제어 등 |

---

## ✅ 대표적인 Service Mesh: Istio vs Linkerd

### 📌 Istio

- **Google, IBM, Lyft** 등에서 공동 개발
- 가장 기능이 풍부한 서비스 메쉬
- 프록시로 **Envoy** 사용
- 플러그인 방식으로 다양한 기능 확장 가능

**특징**

- 강력한 **트래픽 제어 기능** (A/B, Canar, Mirror 등)
- **mTLS 자동 설정**, 정책 기반 보안 구성
- Jaeger, Prometheus, Kiali 등과 연동
- 복잡하지만 기능은 가장 풍부

**구성 요소**

| 컴포넌트 | 역할 |
|----------|------|
| istiod | Control Plane 역할, 프록시 구성 관리 |
| Envoy Proxy | 각 Pod에 사이드카로 주입되는 L7 프록시 |
| Kiali | 대시보드 (옵션) |
| Prometheus / Grafana / Jaeger | 모니터링 & 트레이싱 (옵션) |

**설치 예시 (istioctl)**

```bash
istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled
```

---

### 📌 Linkerd

- CNCF 공식 프로젝트
- 사용하기 쉬운 경량형 서비스 메쉬
- 프록시로 **Rust 기반의 Linkerd-proxy** 사용

**특징**

- 설치와 구성 매우 간단
- 기본적으로 **mTLS** 및 **트래픽 가시성 제공**
- Helm, CLI 모두 지원
- 복잡한 라우팅은 다소 제한적이지만, **경량성과 안정성에 집중**

**구성 요소**

| 컴포넌트 | 역할 |
|----------|------|
| control-plane | 트래픽 정책, 보안 설정 |
| data-plane | 사이드카 프록시 (Rust 기반) |

**설치 예시 (CLI)**

```bash
curl -sL https://run.linkerd.io/install | sh
linkerd install | kubectl apply -f -
linkerd check
```

---

## ✅ Istio vs Linkerd 비교

| 항목 | Istio | Linkerd |
|------|-------|---------|
| 복잡도 | 높음 | 낮음 |
| 프록시 | Envoy | Rust 기반 linkerd-proxy |
| 설치 용이성 | Helm / istioctl (설정 많음) | CLI 기반 설치 간단 |
| 트래픽 제어 | 고급 (Canary, Mirror 등) | 기본 (Traffic Split 정도) |
| 성능 | 다소 무거움 | 매우 경량 |
| 보안 | 강력한 정책 기반 제어 | 기본 mTLS 제공 |
| 확장성 | 매우 높음 | 제한적 |
| 관측 도구 | Jaeger, Kiali, Prometheus 등 연동 | 자체 대시보드 및 Prometheus 연동 |

---

## ✅ 어떤 Service Mesh를 선택해야 할까?

| 상황 | 추천 |
|------|------|
| **기능이 풍부한 엔터프라이즈 환경** | Istio |
| **간단한 서비스 + 빠른 설치/성능 우선** | Linkerd |
| **Canary / AB 테스트 필요** | Istio |
| **Dev/Test 클러스터에서 가볍게 시작** | Linkerd |

---

## ✅ 참고 자료

- [Istio 공식 문서](https://istio.io/latest/docs/)
- [Linkerd 공식 문서](https://linkerd.io/2.14/)
- [CNCF Service Mesh Landscape](https://landscape.cncf.io/category=service-mesh)

---

## ✅ 결론

Service Mesh는 **코드 변경 없이** 마이크로서비스의 네트워크 통신을 제어하고 관측할 수 있는 혁신적인 기술입니다.

- 운영 복잡성은 증가할 수 있지만,  
  보안, 안정성, 트래픽 제어, 모니터링 등에서 막대한 이점을 제공합니다.
- Kubernetes와 함께 도입할 경우 특히 효과가 큽니다.