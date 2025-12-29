---
layout: post
title: Kubernetes - Calico로 L7 정책 구성
date: 2025-05-11 21:20:23 +0900
category: Kubernetes
---
# Calico를 활용한 L7 네트워크 정책 구성 가이드

Kubernetes의 기본 `NetworkPolicy`는 IP 주소, 포트, 프로토콜과 같은 L3/L4 수준의 네트워크 제어만 제공합니다. 그러나 현대적인 애플리케이션 보안 요구사항은 종종 HTTP 메서드, URL 경로, 호스트 헤더, 특정 헤더 값과 같은 L7(애플리케이션 계층) 속성에 기반한 정밀한 제어를 필요로 합니다.

Calico는 이러한 요구를 충족시키기 위해 Application Layer Policy 기능을 제공하며, 이는 HTTP 트래픽을 인식하고 분석할 수 있는 능력을 갖추고 있습니다. 하지만 중요한 점은 Calico 오픈소스 버전과 엔터프라이즈 버전 사이에 L7 기능 지원에 상당한 차이가 존재한다는 것입니다.

## Calico 버전별 기능 비교

- **Calico 오픈소스(OSS)**: 기본적인 L3/L4 네트워크 정책에 초점을 맞추고 있으며, HTTP 트래픽에 대한 정밀한 L7 제어 기능은 제한적입니다.
- **Calico 엔터프라이즈**: `GlobalNetworkPolicy`의 `http:` 섹션이나 전용 `HTTPPolicy` 리소스를 통해 HTTP 메서드, 경로, 호스트, 헤더에 대한 풍부한 매칭 기능을 완전히 지원합니다.

이 가이드는 두 가지 접근 방식을 모두 다룹니다:
1. Calico 엔터프라이즈 환경에서의 정식 L7 정책 구현 방법
2. Calico OSS 환경에서 활용 가능한 우회 전략과 대체 솔루션

---

## 사전 준비 및 환경 설정

### 기능 요구사항 분석

| 요구사항 | Calico OSS | Calico 엔터프라이즈 |
|---------|------------|-------------------|
| L3/L4 기반 정책(Pod, 네임스페이스, IP, 포트) | 완전 지원 | 완전 지원 (기능 향상) |
| HTTP 메서드/경로/호스트/헤더 기반 제어 | 제한적/실질적 불가 | **완전 지원** |
| DNS/FQDN 기반 아웃바운드 트래픽 제어 | 제한적 (우회 방법 필요) | **완전 지원** |
| 정책 시각화, 시뮬레이션, 감사 | 수동 또는 외부 도구 필요 | **내장 기능 제공** |

엔터프라이즈 라이선스 없이 정밀한 L7 제어가 필요한 경우, CNI 수준 대신 인그레스 컨트롤러나 서비스 메시와 같은 L7 프록시 계층에서 해결하는 아키텍처를 고려하는 것이 현실적입니다.

### 테스트 환경 구성

실습을 위해 다음과 같은 환경을 가정합니다:
- Kubernetes 1.24 이상
- Calico CNI (기본 또는 eBPF 데이터플레인)
- 테스트 네임스페이스: `demo`
- HTTP 테스트 서비스: `httpbin` (요청을 그대로 반환하는 테스트 서버)

```bash
# 테스트 네임스페이스 생성
kubectl create ns demo

# 백엔드 애플리케이션 배포 (httpbin)
kubectl -n demo run backend --image=kennethreitz/httpbin --port=80 --labels="app=backend"
kubectl -n demo expose pod backend --port=80 --target-port=80

# 프론트엔드 테스트 클라이언트 배포
kubectl -n demo run frontend --image=busybox:1.36 --restart=Never -it -- sh
```

프론트엔드 Pod에서 연결 테스트:
```bash
# 정책 적용 전 기본 연결 테스트
wget -qO- http://backend.demo.svc.cluster.local/get
```

---

## Calico 엔터프라이즈: L7 정책 구현

Calico 엔터프라이즈에서는 `GlobalNetworkPolicy` 리소스의 `http:` 섹션을 활용하거나 전용 `HTTPPolicy` 리소스를 사용하여 L7 수준의 정책을 정의할 수 있습니다. 운영 환경에서는 일반적으로 `GlobalNetworkPolicy` 하나에 L3/L4/L7 규칙을 모두 통합하는 방식이 선호됩니다.

### GlobalNetworkPolicy를 이용한 L7 제어

다음 정책은 `frontend` Pod에서 `backend` Pod로의 트래픽 중 GET 메서드로 `/get` 엔드포인트 또는 `/api/v1/` 접두사를 가진 경로에 대한 접근만 허용합니다.

```yaml
# gnp-l7-allow-get-path.yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: demo-l7-allow-get-path
spec:
  # 이 정책이 적용될 Pod 선택 (backend 애플리케이션)
  selector: app == "backend"
  order: 100  # 우선순위 (낮은 값이 높은 우선순위)
  types:
  - Ingress  # 수신 트래픽에 적용
  ingress:
  - action: Allow
    protocol: TCP
    source:
      namespaceSelector: projectcalico.org/name == "demo"   # demo 네임스페이스 내에서만
      selector: app == "frontend"                           # frontend Pod만 허용
    destination:
      ports: [80]  # 80 포트 대상
    http:  # L7 수준 규칙
      methods: ["GET"]  # GET 메서드만 허용
      paths:
        - exact: "/get"             # 정확히 /get 경로
        - prefix: "/api/v1/"        # /api/v1/로 시작하는 모든 경로
      hosts:
        - "backend.demo.svc.cluster.local"  # 특정 호스트 헤더 값 매칭
      headers:  # 헤더 기반 조건
        - name: "X-Env"
          exact: "test"  # X-Env 헤더 값이 정확히 "test"여야 함
  - action: Deny  # 위 조건에 맞지 않는 모든 트래픽 차단
```

정책 적용:
```bash
kubectl apply -f gnp-l7-allow-get-path.yaml
```

테스트 시나리오:
```bash
# 프론트엔드 Pod 내에서 테스트

# 허용 케이스: GET /get with correct header
wget -S -qO- --header="X-Env: test" http://backend.demo.svc.cluster.local/get

# 허용 케이스: GET /api/v1/users with correct header
wget -S -qO- --header="X-Env: test" http://backend.demo.svc.cluster.local/api/v1/users

# 차단 케이스: POST /post (메서드 불일치)
wget -S -qO- --post-data="x=1" http://backend.demo.svc.cluster.local/post

# 차단 케이스: 헤더 누락
wget -S -qO- http://backend.demo.svc.cluster.local/get
```

### 고급 헤더 및 호스트 매칭 예제

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: demo-l7-advanced-headers
spec:
  selector: app == "backend"
  order: 90
  types: [Ingress]
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: app == "frontend"
    destination:
      ports: [80]
    http:
      methods: ["GET"]
      hosts:
        - "api.example.com"
        - "backend.demo.svc.cluster.local"
      headers:
        - name: "Authorization"
          prefix: "Bearer "     # "Bearer "로 시작하는 Authorization 헤더
        - name: "X-Role"
          regex: "^admin|ops$"  # 정규식 매칭 (admin 또는 ops 역할)
  - action: Deny  # 기본 차단 규칙
```

헤더 정규식 매칭과 같은 고급 기능은 Calico 버전과 라이선스에 따라 지원 범위가 다를 수 있으므로 공식 문서를 확인하는 것이 중요합니다.

---

## 정책 검증 및 테스트 방법론

효과적인 L7 정책 관리를 위해 다음과 같은 검증 절차를 권장합니다:

1. **허용 케이스 검증**
   - 정책에서 명시적으로 허용한 조합(예: GET 메서드 + 특정 헤더 + 허용된 경로)이 실제로 통과하는지 확인

2. **차단 케이스 검증**
   - 의도적으로 차단되어야 할 트래픽(잘못된 메서드, 부적절한 헤더, 금지된 경로)이 실제로 거부되는지 확인
   - 다른 네임스페이스나 라벨을 가진 Pod에서의 접근 시도 차단 확인

3. **관측 및 모니터링**
   - Calico 엔터프라이즈의 플로우 로그, 정책 히트 카운터, 시뮬레이터 활용
   - OSS 환경에서는 애플리케이션 로그나 인그레스 컨트롤러 로그를 통한 간접 확인

---

## 정책 우선순위 및 설계 패턴

Calico 정책 평가에서는 `order` 필드가 결정적인 역할을 합니다. 숫자가 낮을수록 높은 우선순위를 가지며 먼저 평가됩니다. 운영 환경에서 권장하는 설계 패턴은 다음과 같습니다:

1. **세부 허용 규칙**을 낮은 `order` 값(예: 10-100)으로 정의하여 먼저 평가되도록 구성
2. **기본 차단 규칙**을 높은 `order` 값(예: 1000)으로 정의하여 나머지 모든 트래픽을 차단
3. 명시적 허용 규칙에 매칭되지 않은 모든 트래픽은 기본 차단 규칙에 의해 처리됨

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: demo-default-deny
spec:
  order: 1000  # 높은 값 = 낮은 우선순위
  selector: app == "backend"
  types: [Ingress]
  ingress:
  - action: Deny  # 모든 수신 트래픽 기본 차단
```

이러한 "기본 거부, 명시적 허용" 전략은 제로 트러스트 보안 모델의 핵심 원칙을 구현하며, 의도하지 않은 접근을 효과적으로 방지합니다.

---

## 성능 고려사항 및 설계 권장사항

L7 네트워크 정책 도입 시 고려해야 할 성능 및 설계 요소:

1. **프록시 오버헤드**: L7 트래픽 검사는 순수 L3/L4 패킷 필터링보다 더 많은 CPU 및 메모리 리소스를 소비합니다. 높은 QPS(초당 쿼리 수)를 처리해야 하는 경로에서는 정책을 단순화하거나 캐싱 계층을 도입하는 것이 좋습니다.

2. **eBPF 데이터플레인**: Calico의 eBPF 데이터플레인은 L3/L4 성능을 크게 향상시킬 수 있지만, L7 검사에는 여전히 추가적인 처리가 필요할 수 있습니다. 실제 워크로드에서 성능 측정이 필수적입니다.

3. **TLS 트래픽 처리**: L7 정책은 일반적으로 평문 HTTP 헤더와 경로를 검사합니다. 엔드투엔드 TLS가 적용된 트래픽의 경우, 중간에서 TLS 종료를 수행하거나 특별한 처리가 필요합니다.

4. **정규식 성능**: 복잡한 정규식 패턴은 처리 비용이 높을 수 있습니다. 가능한 경우 `exact`(정확 일치)나 `prefix`(접두사 일치)를 우선 사용하고, 정규식은 필요한 경우에만 제한적으로 적용하세요.

---

## Calico OSS 환경에서의 대안적 접근법

Calico 오픈소스 버전만 사용 가능한 환경에서는 L7 기능이 제한적이므로, 다음과 같은 대체 솔루션을 고려할 수 있습니다:

### 1. 인그레스 컨트롤러를 활용한 L7 제어
NGINX, HAProxy, Contour 등의 인그레스 컨트롤러는 경로, 호스트, 메서드 기반 라우팅과 함께 기본적인 L7 필터링 기능을 제공합니다.

```yaml
# NGINX 인그레스 컨트롤러 예제
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      # GET 또는 HEAD 메서드만 허용
      if ($request_method !~ ^(GET|HEAD)$) { return 403; }
      # 특정 헤더 검증
      if ($http_x_env != "test") { return 403; }
spec:
  rules:
  - host: backend.demo.local
    http:
      paths:
      - path: /api/?(.*)
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 80
```

### 2. 서비스 메시 통합
Istio, Linkerd와 같은 서비스 메시 솔루션은 풍부한 L7 트래픽 제어 기능을 제공하며, Calico와 함께 사용하여 다층 보안 전략을 구현할 수 있습니다.

### 3. API 게이트웨이 활용
Kong, Traefik, NGINX Plus와 같은 API 게이트웨이는 고급 L7 기능(인증, 속도 제한, 회로 차단기 등)을 제공하며, 북-남 및 동-서 트래픽 모두에 적용 가능합니다.

### 4. 애플리케이션 수준 제어
애플리케이션 프레임워크 자체의 미들웨어나 필터를 활용하여 L7 수준의 접근 제어를 구현할 수 있습니다. 이 접근법은 애플리케이션 코드와 긴밀하게 통합되지만, 분산 환경에서의 일관성 유지가 어려울 수 있습니다.

**Calico OSS와의 통합 패턴**: OSS Calico는 위의 L7 솔루션 앞뒤에서 L3/L4 수준의 최소 권한 원칙을 강제하는 데 탁월합니다. 예를 들어, 인그레스 컨트롤러 Pod만 백엔드 서비스에 접근할 수 있도록 제한하는 정책을 적용할 수 있습니다.

```yaml
# Calico OSS 정책: 인그레스 컨트롤러만 백엔드 접근 허용
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: oss-backend-access
spec:
  selector: app == "backend"
  order: 100
  types: [Ingress]
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: app == "ingress-nginx"  # 인그레스 컨트롤러 Pod 선택
    destination:
      ports: [80]
  - action: Deny  # 나머지 모든 접근 차단
```

---

## 운영 모범 사례

Calico L7 정책 운영 시 다음 사항을 고려하세요:

1. **정책 우선순위 관리**: 모든 정책에 명시적인 `order` 값을 부여하고, 의도한 평가 순서를 정기적으로 검증하세요.

2. **TLS 처리 전략**: L7 정책이 적용될 트래픽의 TLS 종단 지점을 명확히 이해하고, 필요시 TLS 종료/재암호화 계층을 도입하세요.

3. **네임스페이스 및 서비스 아이덴티티 활용**: 정책 소스 범위를 가능한 한 좁히기 위해 네임스페이스 셀렉터와 Pod 라벨을 적극적으로 활용하세요.

4. **단계적 배포 및 테스트**: 새로운 정책을 프로덕션에 적용하기 전에 스테이징 환경에서 철저히 테스트하세요. 허용 및 차단 시나리오를 모두 검증하는 것이 중요합니다.

5. **성능 최적화**: 고성능이 요구되는 경로에서는 정책을 단순화하고, 가능한 경우 접두사 일치를 사용하며, 정규식은 필수적인 경우에만 제한적으로 적용하세요.

6. **관측 가능성 확보**: 플로우 로그, 정책 히트 카운터, 애플리케이션 로그를 통해 정책 효과를 지속적으로 모니터링하세요.

---

## 종합 예제: 통합 배포 매니페스트

다음은 데모 환경을 한 번에 구성하는 완전한 예제입니다:

```yaml
# 1. 네임스페이스 생성
apiVersion: v1
kind: Namespace
metadata:
  name: demo

---
# 2. 백엔드 서비스
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: demo
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80

---
# 3. L7 허용 정책 (프론트엔드 → 백엔드)
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: demo-l7-allow
spec:
  selector: app == "backend"
  order: 100
  types: [Ingress]
  ingress:
  - action: Allow
    protocol: TCP
    source:
      namespaceSelector: projectcalico.org/name == "demo"
      selector: app == "frontend"
    destination:
      ports: [80]
    http:
      methods: ["GET"]
      paths:
        - exact: "/get"
        - prefix: "/api/v1/"
      headers:
        - name: "X-Env"
          exact: "test"
  - action: Deny  # 나머지 모든 트래픽 차단

---
# 4. 기본 차단 정책 (안전망)
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: demo-default-deny
spec:
  selector: app == "backend"
  order: 1000  # 가장 낮은 우선순위
  types: [Ingress]
  ingress:
  - action: Deny  # 명시적 허용 규칙에 매칭되지 않은 모든 트래픽 차단
```

테스트 명령어:
```bash
# 허용되는 요청
wget -S -qO- --header="X-Env: test" http://backend.demo.svc.cluster.local/get
wget -S -qO- --header="X-Env: test" http://backend.demo.svc.cluster.local/api/v1/users

# 차단되는 요청
wget -S -qO- --post-data="x=1" http://backend.demo.svc.cluster.local/post
wget -S -qO- http://backend.demo.svc.cluster.local/get  # 헤더 없음
```

---

## 문제 해결 가이드

| 증상 | 가능한 원인 | 해결 방안 |
|------|-------------|-----------|
| L7 정책이 예상대로 동작하지 않음 | Calico OSS에서 L7 기능 미지원 | 엔터프라이즈 기능 여부 확인 또는 대체 솔루션(인그레스/서비스 메시) 도입 |
| 허용 규칙이 있음에도 트래픽이 차단됨 | 정책 우선순위(`order`) 문제 | 낮은 `order` 값을 가진 차단 규칙이 먼저 적용되고 있을 수 있음. `calicoctl get policy -o wide`로 정책 순서 확인 |
| 헤더/호스트 매칭 실패 | TLS 종단 지점 문제 | 평문 HTTP 헤더 검사를 위해 TLS 종료 지점을 인그레스나 프록시 계층으로 이동 |
| 일부 트래픽만 통과/차단됨 | 경로 또는 정규식 조건 불일치 | 실제 요청 값 로깅 및 정책 조건 점검. 규칙을 단순화하여 테스트 |
| 성능 저하 관찰 | L7 검사 및 정규식 처리 부하 | 고성능 경로에서는 규칙 단순화, 접두사 일치 우선 사용, 캐싱 계층 도입 |

---

## 결론

Calico를 활용한 L7 네트워크 정책 구성은 Kubernetes 환경에서 정교한 애플리케이션 수준 보안을 구현하는 강력한 방법을 제공합니다.

**Calico 엔터프라이즈** 사용자는 `GlobalNetworkPolicy`의 `http:` 섹션을 통해 HTTP 메서드, 경로, 호스트, 헤더에 대한 정밀한 제어를 네트워크 계층에서 직접 구현할 수 있습니다. 이는 제로 트러스트 보안 모델을 네트워크 수준으로 확장하는 데 매우 효과적입니다.

**Calico OSS** 환경에서는 기본적인 L7 기능이 제한적이므로, 인그레스 컨트롤러, 서비스 메시, API 게이트웨이와 같은 L7 프록시 계층에서 정책을 구현하고, Calico는 L3/L4 수준의 최소 권한 원칙을 강제하는 이중 레이어 전략을 채택하는 것이 현실적입니다.

효과적인 운영을 위해서는 명확한 정책 우선순위 관리, 기본 거부 전략 채택, TLS 처리 고려, 성능 측정 및 관측 가능성 확보가 필수적입니다. 조직의 요구사항, 예산, 기술 스택을 종합적으로 고려하여 가장 적합한 접근 방식을 선택하시기 바랍니다.

이 가이드의 예제와 패턴을 참고하여 조직의 특정 요구사항에 맞는 L7 네트워크 보안 전략을 설계하고 구현할 수 있을 것입니다.