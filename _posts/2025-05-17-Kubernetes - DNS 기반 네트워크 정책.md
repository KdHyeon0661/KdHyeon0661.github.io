---
layout: post
title: Kubernetes - DNS 기반 네트워크 정책
date: 2025-05-11 21:20:23 +0900
category: Kubernetes
---
# DNS 기반 네트워크 정책 구성하기: Calico와 Cilium 비교 가이드

Kubernetes의 기본 `NetworkPolicy`는 IP 주소, 포트, 프로토콜과 같은 L3/L4 수준의 네트워크 제어에 중점을 둡니다. 그러나 현대 애플리케이션은 종종 Stripe, GitHub, Google Cloud Storage와 같은 외부 서비스에 의존하며, 이러한 서비스들은 일반적으로 DNS 이름을 통해 접근됩니다. 문제는 CDN, Anycast, DNS 라운드로빈 등의 기술로 인해 이러한 서비스의 IP 주소가 수시로 변경될 수 있다는 점입니다. 이러한 상황에서 정적 IP 기반 정책은 실용적이지 않으며, 도메인 이름(FQDN) 기반의 아웃바운드 정책이 필요해집니다.

이 가이드는 Calico와 Cilium 두 가지 주요 CNI(Container Network Interface) 솔루션에서 FQDN 기반 네트워크 정책을 구현하는 방법을 비교하고 실습하며, 각 접근법의 원리, 테스트 방법, 운영 모범 사례, 그리고 한계점을 종합적으로 설명합니다.

---

## 기본 개념과 요구사항

### 핵심 요구사항
- **도메인 기반 접근 제어**: 특정 워크로드가 허용된 외부 도메인으로만 아웃바운드 트래픽을 보낼 수 있도록 제한
- **동적 IP 추적**: 외부 서비스의 IP 주소 변경을 자동으로 추적하고 정책에 반영
- **DNS 트래픽 허용**: 도메인 이름 해석을 위한 DNS 트래픽(포트 53)을 필수적으로 허용
- **관측 가능성**: 어떤 워크로드가 어떤 도메인으로 얼마나 많은 트래픽을 보내는지 모니터링 및 감사 가능

### 테스트 환경 설정
실습을 위해 먼저 테스트 환경을 구성합니다:

```bash
# 테스트 네임스페이스 생성
kubectl create ns fqdn-lab

# 테스트 Pod 실행 (대화형 셸)
kubectl -n fqdn-lab run tbox --image=curlimages/curl -it --rm -- bash

# Pod 내부에서 테스트 명령어 예시
curl -sI https://www.google.com | head -n1
```

---

## FQDN 정책의 동작 원리

FQDN 기반 네트워크 정책의 핵심 메커니즘은 DNS 질의-응답 주기를 관찰하고 그 결과를 네트워크 정책 평가에 활용하는 것입니다:

1. **DNS 질의 감지**: Pod가 외부 도메인에 대한 DNS 질의를 CoreDNS나 외부 DNS 서버로 전송합니다.
2. **응답 캐싱**: CNI 컴포넌트가 DNS 응답을 관찰(eBPF 또는 iptables 수준)하고, 도메인-IP 매핑 정보를 내부 캐시에 저장합니다.
3. **트래픽 평가**: Pod에서 나가는 실제 네트워크 트래픽이 평가될 때, 목적지 IP 주소가 허용된 도메인의 캐시된 IP 목록에 포함되어 있는지 확인합니다.
4. **TTL 기반 갱신**: DNS 레코드의 TTL(Time To Live)이 만료되면 캐시를 갱신하여 IP 변경을 추적합니다.

**중요한 주의사항**: 이미 수립된 장기간 TCP 연결은 연결 시점의 IP 주소를 계속 사용할 수 있습니다. IP 주소가 변경된 경우, 새로운 연결부터 업데이트된 IP 매핑이 적용됩니다.

---

## 기본 DNS 트래픽 허용

FQDN 정책을 적용하기 전에 먼저 DNS 트래픽을 허용해야 합니다. DNS 서버에 접근할 수 없으면 도메인 이름 해석 자체가 불가능해집니다.

### Calico에서 DNS 허용 정책
```yaml
# calico-dns-allow.yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: fqdn-lab
spec:
  selector: all()  # 네임스페이스 내 모든 Pod에 적용
  types: [Egress]
  egress:
    - action: Allow
      protocol: UDP
      destination:
        selector: 'k8s-app == "kube-dns"'
        ports: [53]
    - action: Allow
      protocol: TCP
      destination:
        selector: 'k8s-app == "kube-dns"'
        ports: [53]
```

**참고**: 클러스터의 CoreDNS 레이블은 환경에 따라 다를 수 있습니다. `kubectl -n kube-system get pods --show-labels` 명령으로 실제 레이블을 확인하고 `selector`를 적절히 조정해야 합니다.

### Cilium에서 DNS 허용 정책
```yaml
# cilium-dns-allow.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-dns
  namespace: fqdn-lab
spec:
  endpointSelector: {}  # 네임스페이스 전체 적용
  egress:
    - toEndpoints:
        - matchLabels:
            k8s:app.kubernetes.io/name: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
```

---

## Calico를 이용한 FQDN 제어

Calico는 `NetworkPolicy` 또는 `GlobalNetworkPolicy` 리소스를 통해 FQDN 기반 아웃바운드 제어를 지원합니다. 주의할 점은 Calico 버전에 따라 와일드카드 지원 범위가 다를 수 있으므로 공식 문서를 참조하는 것이 중요합니다.

### 특정 도메인만 허용하는 네임스페이스 수준 정책
```yaml
# calico-fqdn-allowlist.yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-fqdn-only
  namespace: fqdn-lab
spec:
  selector: all()
  types: [Egress]
  egress:
    # 허용 목록: 특정 도메인으로의 HTTPS 트래픽만 허용
    - action: Allow
      protocol: TCP
      destination:
        ports: [443]
        dns:
          names:
            - "www.google.com"
            - "api.github.com"
    # 기본 차단 규칙: 나머지 모든 트래픽 차단
    - action: Deny
      destination:
        nets:
          - 0.0.0.0/0
          - ::/0
```

**문법 주의**: Calico 버전에 따라 `dns.names` 대신 `domains` 필드를 사용할 수 있습니다. 배포된 Calico 버전의 공식 문서를 확인하여 적절한 필드 이름을 사용하세요.

### 클러스터 전체 적용을 위한 글로벌 정책
여러 네임스페이스에 동일한 정책을 일관되게 적용하려면 `GlobalNetworkPolicy`를 사용할 수 있습니다:

```yaml
# calico-global-fqdn.yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-to-google-global
spec:
  selector: all()
  types: [Egress]
  egress:
    - action: Allow
      protocol: TCP
      destination:
        ports: [443]
        dns:
          names:
            - "www.google.com"
    - action: Deny
      destination:
        nets:
          - 0.0.0.0/0
          - ::/0
```

---

## Cilium을 이용한 FQDN 제어

Cilium은 `CiliumNetworkPolicy` 리소스의 `toFQDNs` 필드를 통해 풍부한 도메인 매칭 기능을 제공합니다. eBPF 기술을 활용하여 효율적인 DNS 트래픽 관찰과 캐싱을 구현합니다.

### 정확한 도메인 이름 매칭
```yaml
# cilium-fqdn-exact.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-specific-fqdns
  namespace: fqdn-lab
spec:
  endpointSelector: {}
  egress:
    - toFQDNs:
        - matchName: "api.github.com"
        - matchName: "www.google.com"
    - toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

### 와일드카드를 이용한 패턴 매칭
Cilium은 와일드카드 패턴을 통한 유연한 도메인 매칭을 지원합니다:

```yaml
# cilium-fqdn-wildcard.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-subdomains
  namespace: fqdn-lab
spec:
  endpointSelector: {}
  egress:
    - toFQDNs:
        - matchPattern: "*.example.com"
    - toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

**보안 고려사항**: `*.example.com`과 같은 광범위한 와일드카드 패턴은 의도하지 않은 하위 도메인까지 허용할 수 있습니다. 가능한 한 구체적인 도메인 목록을 사용하고, 와일드카드는 필수적인 경우에만 제한적으로 적용하세요.

---

## 기본 차단 전략과 정책 레이어링

보안 모범 사례는 "기본적으로 모든 것을 차단하고, 필요한 것만 명시적으로 허용"하는 것입니다. 이 접근법을 FQDN 정책에 적용해 보겠습니다.

### Calico 기본 차단 정책
```yaml
# calico-default-deny.yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: fqdn-lab
spec:
  selector: all()
  types: [Egress]
  egress:
    - action: Deny
```

이 기본 차단 정책을 적용한 후, DNS 허용 정책과 특정 FQDN 허용 정책을 별도로 적용하여 필요한 트래픽만 허용할 수 있습니다.

### Cilium 기본 차단 정책
```yaml
# cilium-default-deny.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-deny
  namespace: fqdn-lab
spec:
  endpointSelector: {}
  egress:
    - toEntities: []  # 명시적 허용이 없는 모든 트래픽 차단
```

**정책 적용 순서**: 정책이 평가되는 순서가 중요합니다. 일반적으로 DNS 허용 → FQDN 허용 → 기본 차단의 순서로 정책을 적용해야 합니다.

---

## 실전 시나리오 구현 패턴

### SaaS API 접근 제한 시나리오
결제 처리 애플리케이션이 GitHub API, Stripe API, Slack API에만 접근할 수 있도록 제한하는 예제입니다.

**Calico 구현**:
```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-saas-only
  namespace: fqdn-lab
spec:
  selector: app == "payments"
  types: [Egress]
  egress:
    - action: Allow
      protocol: TCP
      destination:
        ports: [443]
        dns:
          names:
            - "api.github.com"
            - "api.stripe.com"
            - "slack.com"
    - action: Deny
      destination:
        nets:
          - 0.0.0.0/0
          - ::/0
```

**Cilium 구현**:
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-saas-only
  namespace: fqdn-lab
spec:
  endpointSelector:
    matchLabels:
      app: payments
  egress:
    - toFQDNs:
        - matchName: "api.github.com"
        - matchName: "api.stripe.com"
        - matchName: "slack.com"
    - toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

### 조직 내부 도메인 패턴 허용
특정 조직의 모든 서브도메인에 대한 접근을 허용하는 경우:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-organization-domains
  namespace: fqdn-lab
spec:
  endpointSelector: {}
  egress:
    - toFQDNs:
        - matchPattern: "*.corp.example.com"
    - toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

### 고급 제어 요구사항
FQDN 정책만으로는 HTTP 경로나 메서드 수준의 제어가 어렵습니다. 이러한 고급 요구사항의 경우:
- **Service Mesh 통합**: Istio나 Linkerd와 같은 서비스 메시를 도입하여 L7 수준의 정책 적용
- **API 게이트웨이 활용**: Kong, Traefik과 같은 API 게이트웨이를 통해 경로 기반 라우팅 및 제어
- **프록시 계층 추가**: 애플리케이션 프록시를 통해 세부적인 HTTP 수준 제어 구현

---

## 정책 테스트 및 검증 방법론

### 기본 연결 테스트
```bash
# 허용된 도메인 테스트
curl -sI https://www.google.com | head -n1
curl -sI https://api.github.com | head -n1

# 차단되어야 할 도메인 테스트
curl -sI https://www.naver.com | head -n1

# DNS 확인
dig +short api.github.com
```

### Cilium 관측 도구 활용
Cilium은 풍부한 관측 도구를 제공하여 정책 동작을 실시간으로 모니터링할 수 있습니다:

```bash
# Cilium 상태 확인
cilium status

# 정책 드롭 이벤트 모니터링
cilium monitor --type drop

# Hubble을 통한 네트워크 흐름 가시화
hubble observe --from-pod fqdn-lab/tbox --http
```

### 네트워크 트래픽 분석
필요한 경우 노드나 Pod 레벨에서 패킷 캡처를 통해 트래픽 흐름을 분석할 수 있습니다:

```bash
# DNS 트래픽 모니터링
tcpdump -i any port 53 -nn

# 특정 IP와 포트의 트래픽 분석
tcpdump -i any host 140.82.112.5 and port 443 -nn
```

---

## 운영 모범 사례 및 고려사항

### DNS 캐시 및 TTL 관리
- **CoreDNS 캐시 설정**: CoreDNS의 `cache` 플러그인 설정을 통해 DNS 응답 캐싱 동작을 최적화
- **CNI 캐시 TTL**: Calico나 Cilium의 내부 FQDN 캐시 TTL을 적절히 설정하여 성능과 정확성 간 균형 유지
- **Too Short TTL**: 너무 짧은 TTL은 빈번한 DNS 재질의로 인한 오버헤드 발생
- **Too Long TTL**: 너무 긴 TTL은 IP 변경 추적 지연으로 인한 연결 문제 발생

### 장기 연결 처리
- **IP 변경 영향**: 도메인의 IP가 변경되어도 이미 수립된 TCP 연결은 계속 유지될 수 있음
- **재연결 전략**: 민감한 트래픽의 경우 주기적인 재연결이나 짧은 keep-alive 시간 설정 고려
- **프록시 계층**: 중요한 트래픽은 L7 프록시나 서비스 메시를 통해 라우팅하여 연결 관리

### 이중 스택(IPv4/IPv6) 환경
- **IPv6 고려**: 정책에 IPv6 차단(`::/0`)을 명시하지 않으면 IPv6 트래픽이 우회될 수 있음
- **일관된 정책**: IPv4와 IPv6 모두에 대해 동일한 정책 원칙 적용
- **테스트**: 이중 스택 환경에서는 두 프로토콜 모두에서 정책이 의도대로 동작하는지 검증

### 보안 및 감사
- **최소 권한 원칙**: 필요한 최소한의 도메인만 허용하고, 와일드카드는 신중하게 사용
- **GitOps 통합**: 정책을 Git 리포지토리에서 관리하고 Argo CD나 Flux를 통해 배포
- **감사 로깅**: 허용/차단된 도메인 접근에 대한 상세 로깅 구현
- **정기 검토**: 허용된 도메인 목록을 정기적으로 검토하고 불필요한 항목 제거

---

## 일반적인 문제 해결

| 증상 | 가능한 원인 | 해결 방안 |
|------|-------------|-----------|
| 모든 외부 연결 실패 | DNS 트래픽이 차단됨 | DNS 허용 정책 추가 및 CoreDNS 서비스 레이블 확인 |
| 특정 도메인만 간헐적 실패 | CDN, 다중 IP, IPv6 문제 | IPv6 정책 확인, TTL 조정, 특정 DNS 레코드 타입 확인 |
| 와일드카드 패턴이 작동하지 않음 | 버전별 지원 차이 | CNI 버전 확인, 정확한 도메인 매칭으로 우선 구현 |
| 정책이 적용되지 않음 | 우선순위 또는 범위 문제 | 정책 적용 순서 확인, 네임스페이스/글로벌 정책 구분 확인 |
| 성능 저하 | 과도한 도메인 패턴 또는 짧은 TTL | 도메인 목록 최적화, TTL 적정화, 관측 데이터 기반 튜닝 |

---

## 대체 아키텍처 및 고급 패턴

### 이그레스 게이트웨이 패턴
서비스 메시의 이그레스 게이트웨이를 활용하면 중앙 집중식 아웃바운드 제어와 향상된 가시성을 얻을 수 있습니다:

- **Istio Egress Gateway**: SNI(Server Name Indication) 기반 라우팅 및 정책 적용
- **중앙 집중식 로깅**: 모든 아웃바운드 트래픽에 대한 통합 로깅 및 감사
- **IP 고정**: NAT를 통해 내부 Pod IP를 일관된 출발지 IP로 마스킹

### L7 수준의 세부 제어
FQDN 기반 제어에 더해 HTTP 수준의 세부 제어가 필요한 경우:

- **Calico Application Layer Policy**: HTTP 메서드, 경로, 헤더 기반 제어 (엔터프라이즈 기능)
- **EnvoyFilter**: Istio 환경에서 HTTP 트래픽에 대한 세부적인 필터링
- **API 게이트웨이**: Kong, Traefik 등을 통한 경로, 메서드, 인증 기반 제어

### GitOps 기반 정책 관리
정책을 코드로 관리하고 GitOps 워크플로우에 통합하는 예제:

```yaml
# Helm 템플릿을 통한 Cilium 정책 관리
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: {{ include "app.name" . }}-fqdn-policy
  namespace: {{ .Values.namespace }}
spec:
  endpointSelector:
    matchLabels:
      app: {{ .Values.appName }}
  egress:
    - toFQDNs:
        {{- range .Values.allowedDomains }}
        - matchName: "{{ . }}"
        {{- end }}
    - toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

---

## 종합 실습 예제

### Calico 전체 구성 예제
```yaml
# calico-complete-example.yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: fqdn-lab
spec:
  selector: all()
  types: [Egress]
  egress:
    - action: Deny

---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: fqdn-lab
spec:
  selector: all()
  types: [Egress]
  egress:
    - action: Allow
      protocol: UDP
      destination:
        selector: 'k8s-app == "kube-dns"'
        ports: [53]
    - action: Allow
      protocol: TCP
      destination:
        selector: 'k8s-app == "kube-dns"'
        ports: [53]

---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-fqdn
  namespace: fqdn-lab
spec:
  selector: all()
  types: [Egress]
  egress:
    - action: Allow
      protocol: TCP
      destination:
        ports: [443]
        dns:
          names:
            - "www.google.com"
            - "api.github.com"
    - action: Deny
      destination:
        nets: [0.0.0.0/0, ::/0]
```

### 테스트 실행
```bash
# 정책 적용
kubectl -n fqdn-lab apply -f calico-complete-example.yaml

# 테스트 Pod 실행 및 검증
kubectl -n fqdn-lab run test-pod --image=curlimages/curl -it --rm -- \
  sh -lc 'curl -sI https://api.github.com | head -n1; echo "---"; curl -sI https://www.naver.com | head -n1'
```

---

## 결론

DNS 기반 네트워크 정책은 IP 주소가 자주 변경되는 현대 클라우드 서비스 환경에서 필수적인 보안 제어 메커니즘입니다. Calico와 Cilium은 각각의 접근법으로 FQDN 기반 아웃바운드 제어를 제공하며, 조직의 기술 스택과 요구사항에 맞는 솔루션을 선택할 수 있습니다.

**핵심 구현 원칙**:
1. **DNS 우선 허용**: 도메인 이름 해석을 위한 DNS 트래픽을 먼저 허용
2. **명시적 허용 목록**: 필요한 외부 도메인만 정확하게 지정하여 허용
3. **기본 차단**: 명시적으로 허용되지 않은 모든 트래픽 차단
4. **정책 레이어링**: DNS 허용 → FQDN 허용 → 기본 차단의 순차적 적용

**운영 고려사항**:
- **TTL 관리**: DNS 캐시 TTL을 적절히 설정하여 성능과 정확성 간 균형 유지
- **이중 스택 지원**: IPv4와 IPv6 모두에 대한 정책 고려
- **장기 연결**: IP 변경 시 영향을 받을 수 있는 장기 연결에 대한 대비
- **관측 가능성**: 정책 적용 현황과 트래픽 패턴에 대한 지속적인 모니터링

**고급 보안을 위한 확장**:
FQDN 기반 제어는 강력한 첫 번째 방어선이지만, HTTP 메서드, 경로, 헤더 수준의 세부 제어가 필요한 경우 서비스 메시, API 게이트웨이, L7 프록시와 같은 추가 계층과의 통합을 고려해야 합니다. 이러한 다층 방어 전략을 통해 진정한 제로 트러스트 네트워크 보안을 구현할 수 있습니다.

조직의 특정 요구사항, 기술 역량, 예산을 고려하여 적절한 CNI 솔루션과 정책 전략을 선택하고, 이 가이드의 패턴과 모범 사례를 참고하여 안전하고 관리 가능한 Kubernetes 네트워크 보안 환경을 구축하시기 바랍니다.