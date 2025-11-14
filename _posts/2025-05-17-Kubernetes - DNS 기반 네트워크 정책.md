---
layout: post
title: Kubernetes - DNS 기반 네트워크 정책
date: 2025-05-11 21:20:23 +0900
category: Kubernetes
---
# DNS 기반 네트워크 정책 구성하기

Kubernetes 기본 `NetworkPolicy`는 **Pod/Namespace/IP** 기준의 L3/L4 제어다.
현실 세계의 외부 의존성은 `api.stripe.com`, `storage.googleapis.com` 같은 **도메인(FQDN)** 으로 접근하는 것이 일반적이며, **CDN·Anycast·DNS 라운드로빈으로 IP가 수시로 바뀐다**.
이럴 때 **FQDN(도메인) 기반 Egress 정책**이 필요하다.

본 글은 **Calico**와 **Cilium**에서의 FQDN 정책을 비교·실습하고, 동작 원리/테스트/운영 팁/한계와 대안을 종합 정리한다.
(※ FQDN 제어는 CNI 기능이다. Flannel 등 미지원 CNI에선 동작하지 않는다.)

---

## 요구사항 정리 & 베이스라인

- **요구:** 특정 워크로드가 **허용된 외부 도메인만** 나가게 하라(Allowlist).
- **제약:** 외부 서비스의 **IP 변경**을 자동 추적해야 함.
- **보안:** **DNS 자체로 나가는 트래픽(53/UDP·TCP)** 은 허용해야 이름 해석이 가능.
- **가시성:** “누가 어느 도메인으로 몇 건 나갔나”를 관찰·감사 가능해야 함.

### 네임스페이스 & 테스트 Pod

```bash
kubectl create ns fqdn-lab
kubectl -n fqdn-lab run tbox --image=curlimages/curl -it --rm -- bash
# 셸 예시: curl -sI https://www.google.com | head -n1

```

> 이후 예제는 `fqdn-lab` 네임스페이스를 기준으로 한다.

---

## 동작 원리(공통)

FQDN 정책의 핵심은 **“DNS 해석 결과(IP) 캐시”** 를 정책 평가에 활용하는 것이다.

1. Pod가 `example.com` 질의 → CoreDNS 응답(여러 A/AAAA).
2. **CNI가 이 질의/응답을 관찰해** *(eBPF/iptables 레벨)* **FQDN→IP 매핑을 캐싱**.
3. Pod의 실제 아웃바운드 패킷이 나갈 때 **목적지 IP가 캐시에 포함되는지**로 **허용/차단**.
4. **TTL 만료/갱신** 시 캐시를 업데이트. 새로운 연결은 **최신 매핑 기준**으로 평가.

> 장기 지속 TCP 연결은 **연결 시점의 IP** 로 유지될 수 있다. IP 변경 시 **신규 연결**부터 새 매핑이 적용된다.

---

## 반드시 먼저: DNS(코어DNS) 통신 허용

FQDN 정책을 적용하면 **기본 Egress가 막히는** 상황이 흔하다. DNS가 막히면 FQDN 해석 자체가 안 된다.
다음은 Calico 기준의 예시(네임스페이스 한정). Cilium도 유사한 개념으로 53 포트를 허용해야 한다.

```yaml
# 02-dns-egress-allow.yaml (Calico NetworkPolicy 예시)

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
```

> 클러스터에 따라 CoreDNS 라벨이 다를 수 있다. `kubectl -n kube-system get po --show-labels` 로 확인 후 `selector` 를 맞춰라.
> Cilium은 `CiliumNetworkPolicy` 의 `toEndpoints` 로 CoreDNS를 지정하는 방식이 일반적이다.

---

## Calico로 FQDN 제어

Calico는 **FQDN 기반 egress** 를 **NetworkPolicy** 또는 **GlobalNetworkPolicy** 로 지원한다.
(버전에 따라 **와일드카드(\*.example.com)** 지원 범위가 다를 수 있으므로 **배포 버전 릴리스 노트로 확인** 권장. 안전하게는 **정확한 FQDN 매칭**을 우선 사용하라.)

### 네임스페이스 한정: 특정 도메인만 허용 + 그 외 차단

```yaml
# 03-calico-fqdn-allowlist.yaml

apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-fqdn-only
  namespace: fqdn-lab
spec:
  selector: all()
  types: [Egress]
  egress:
    # 1) 허용 목록
    - action: Allow
      protocol: TCP
      destination:
        ports: [443]
        domains:
          # Calico v3.26+는 domains 필드(별칭) 또는 dns.names 필드를 지원한다.
          # 배포 버전에 따라 필드명이 'dns: { names: [...] }' 인 경우도 있다.
          - "www.google.com"
          - "api.github.com"
    # 2) 기본 차단
    - action: Deny
      destination:
        nets:
          - 0.0.0.0/0
          - ::/0
```

> `domains:` 대신 `dns: { names: [...] }` 를 쓰는 문법도 있다. **설치한 Calico 버전 문서**의 스키마를 확인해 일치시켜라.

테스트:

```bash
# 허용

curl -sI https://www.google.com | head -n1
curl -sI https://api.github.com | head -n1

# 차단

curl -sI https://www.naver.com
```

### 클러스터 전역: GlobalNetworkPolicy

여러 네임스페이스에 동일 정책을 깔끔히 적용하려면 글로벌로:

```yaml
# 03b-calico-global-fqdn-allow-google.yaml

apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-to-google
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

### 패턴/와일드카드 주의

- 버전에 따라 `*.example.com` 과 같은 **와일드카드** 또는 `suffix` 형식 지원이 **제한**될 수 있다.
- CDN·다중 레코드 환경에서 **IPv4/IPv6** 혼재 여부도 고려하라. 필요 시 `::/0` 차단과 별도 IPv6 허용을 명시.

---

## Cilium로 FQDN 제어

Cilium은 `CiliumNetworkPolicy` 의 `toFQDNs` (및 `matchName`, `matchPattern`) 로 풍부한 도메인 매칭을 제공한다.
또한 eBPF를 활용하여 DNS 관찰/캐시를 효율적으로 처리한다.

### 필수: DNS 허용(Cilium)

```yaml
# 04-cilium-allow-dns.yaml

apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-dns
  namespace: fqdn-lab
spec:
  endpointSelector: {}
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

> CoreDNS 라벨은 환경마다 다르다. `kubectl -n kube-system get svc kube-dns -o yaml` 로 라벨을 확인하고 `matchLabels` 를 맞춰라.

### FQDN 허용(정확 매칭)

```yaml
# 04b-cilium-fqdn-allow.yaml

apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-github-google
  namespace: fqdn-lab
spec:
  endpointSelector: {}   # 네임스페이스 전체 적용(필요 시 레이블로 좁혀라)
  egress:
    - toFQDNs:
        - matchName: "api.github.com"
        - matchName: "www.google.com"
    - toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

### 와일드카드/패턴 허용

```yaml
# 04c-cilium-fqdn-wildcard.yaml

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

> `matchPattern` 은 와일드카드(`*`) 패턴을 지원한다. 너무 광범위한 패턴은 **오탐 허용** 위험이 있으므로 최소화하라.

---

## **베이스라인 Deny** + **정책 레이어링** (권장)

**원칙:** “기본 차단(Default Deny) → 필요한 것만 허용(Allowlist)”.

Calico 예:

```yaml
# 05-calico-default-deny-egress.yaml

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

이후 순서·우선순위를 고려하여 `allow-dns`, `allow-fqdn` 정책을 **별도 리소스**로 겹쳐 적용한다.
(GlobalNetworkPolicy는 **order** 필드로 명시적 우선순위 설정 가능.)

Cilium 예:

```yaml
# 05b-cilium-default-deny.yaml

apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-deny
  namespace: fqdn-lab
spec:
  endpointSelector: {}
  egress:
    - toEntities: []  # 명시적 허용이 없으면 사실상 차단
```

> Cilium은 `toEntities: [world]` 등 **엔티티 단위** 허용도 제공한다. FQDN 제어와 혼용 시 **정확한 스코프**를 설계해라.

---

## 실전 시나리오 레시피

### SaaS 3곳만 허용

- GitHub API, Stripe API, Slack만 허용. 나머지 인터넷 차단.

Calico:

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-saas
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

Cilium:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-saas
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

### 와일드카드: 특정 조직의 서브도메인 전체 허용(Cilium)

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-org
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

### 도메인 + 특정 HTTP 경로 제한(대안)

FQDN 정책만으로는 **HTTP 경로/메서드** 까지 통제하기 어렵다.
**Service Mesh(Istio/Envoy)** 나 **Calico L7(HTTP) 정책**을 병행하여 **SNI·Host+Path 수준**으로 좁혀라.

---

## 테스트·검증·관측

### 빠른 검증 명령

```bash
# 정상/차단 확인

curl -sI https://www.google.com | head -n1
curl -sI https://api.github.com | head -n1
curl -sI https://www.naver.com | head -n1

# DNS 확인

apt-get update && apt-get install -y dnsutils 2>/dev/null || true
dig +short api.github.com
```

### 패킷 관측(필요 시)

노드/Pod에서 `tcpdump` 로 53/443 흐름을 본다. (컨테이너 런타임·권한에 주의)

```bash
tcpdump -i any port 53 -nn
tcpdump -i any host 140.82.112.5 and port 443 -nn
```

### 정책 히트/미스 가시화

- **Calico:** Felix/typha 로그, 정책 엔트리 카운터(엔터프라이즈는 GUI·메트릭 강화).
- **Cilium:** `cilium monitor`, Hubble(가시성)로 정책 매치/드롭 이벤트를 시각화.

```bash
# Cilium

cilium status
cilium monitor --type drop
hubble observe --from-pod fqdn-lab/tbox --http
```

---

## 운영 팁(성능·신뢰·안전)

### DNS 캐시 & TTL

- CoreDNS `cache` 플러그인 TTL, CNI의 FQDN 캐시 TTL이 **너무 짧으면 빈번한 갱신으로 오버헤드**.
- 반대로 너무 길면 **IP 변경 추적 지연**. 서비스 특성에 맞춰 균형점을 찾자.

### 장기 커넥션

- 도메인 IP가 바뀌어도 **이미 맺은 TCP 세션은 살아있을 수 있다**.
- 민감 트래픽은 **짧은 keep-alive**, **주기적 재연결**, **L7 프록시(메쉬/프록시) 경유**로 안전하게.

### IPv6·Dual Stack

- 정책에 `::/0` 차단을 넣지 않으면 **IPv6로 우회 허용**될 수 있다. 듀얼스택 환경에서는 **v4/v6 모두 고려**하라.

### 와일드카드 범위 최소화

- `*.example.com` 은 자칫 **예상치 못한 하위 도메인**까지 허용한다.
- 가능하면 **정확 FQDN 목록**으로 관리하고, 불가피하면 패턴을 최소화해라.

### 감사·컴플라이언스

- 네임스페이스/팀별로 **정책 리포지토리(values/Helm/Kustomize)** 를 분리.
- **GitOps(Argo CD/Flux)** 로 리뷰/승인을 거쳐 변경 이력을 남긴다.
- Cilium Hubble이나 Calico 엔터프라이즈 지표로 **도메인별 요청량·거부율**을 대시보드화.

---

## 자주 만나는 오류 & 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| 모든 외부 접속 실패 | DNS 자체가 막힘 | **DNS 허용 정책** 추가(53/TCP·UDP). CoreDNS 라벨·포트 확인. |
| 어떤 도메인만 간헐 실패 | CDN·다중 IP·IPv6 혼재 | IPv6 허용 포함, TTL/캐시 튜닝, 특정 레코드 유형(A/AAAA) 점검. |
| 와일드카드가 안 먹음 | 버전·스키마 차이 | CNI/버전 문서 확인. **정확 FQDN** 매칭으로 우선 구현. |
| 정책 적용했는데 영향 없음 | 우선순위/범위 불일치 | Calico Global vs Namespaced, Cilium 다중 CNP 병합 규칙 재검토. |
| 성능 저하 | 과도한 패턴/도메인·짧은 TTL | 도메인 집합 통합·정리, TTL 적정화, 관찰 결과로 다이어트. |

---

## 대안·보완 아키텍처

- **Egress Gateway(서비스 메쉬) 경유:** Istio EgressGateway/Envoy로 **SNI/Host** 기준 제어 + 중앙 IP 고정(NAT) + 로깅 일원화.
- **HTTP L7 정책:** Calico Application Layer(HTTP)나 EnvoyFilter로 **메서드/경로/헤더** 기반 통제.
- **프록시(HTTP CONNECT/미러):** 트래픽을 명시 프록시로 보내 도메인·User-Agent·토큰 레벨 정책 추가.

---

## Helm·GitOps로 구성 관리(샘플)

### Cilium FQDN Chart 템플릿 스니펫

{% raw %}
```yaml
# templates/fqdn.yaml

apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: {{ include "app.name" . }}-fqdn
  namespace: {{ .Values.namespace }}
spec:
  endpointSelector: {{ toYaml .Values.selector | nindent 2 }}
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
{% endraw %}

```yaml
# values.yaml

namespace: fqdn-lab
selector:
  matchLabels:
    app: payments
allowedDomains:
  - api.github.com
  - api.stripe.com
  - slack.com
```

### Calico Global 정책 Kustomize 오버레이

```yaml
# base/gnp.yaml

apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-fqdn-global
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
            - "api.github.com"
            - "api.stripe.com"
    - action: Deny
      destination:
        nets: [0.0.0.0/0, ::/0]
```

```yaml
# overlays/prod/kustomization.yaml

resources:
  - ../../base/gnp.yaml
patches:
  - target:
      kind: GlobalNetworkPolicy
      name: allow-fqdn-global
    patch: |-
      - op: add
        path: /spec/egress/0/destination/dns/names/-
        value: "slack.com"
```

---

## 요약(치트시트)

- **필수:** 먼저 **DNS 허용** → 그 다음 **FQDN Allowlist** → 마지막 **기본 차단**.
- **Calico:** `dns.names`/`domains`(버전별 차이). **와일드카드 가용성은 버전 확인**.
- **Cilium:** `toFQDNs.matchName/matchPattern` 으로 유연한 도메인 매칭.
- **Dual Stack:** IPv4/IPv6 모두 고려.
- **장기연결:** IP 변경 시 신규 연결부터 적용. 민감 트래픽엔 프록시/메쉬 고려.
- **운영:** GitOps, 대시보드(허용·거부 추세), TTL 튜닝으로 안정화.

---

## 부록 A — 전체 실습 번들(빠른 재현)

```yaml
# 99-bundle-calico.yaml (Calico)

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

```yaml
# 99-bundle-cilium.yaml (Cilium)

apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-deny
  namespace: fqdn-lab
spec:
  endpointSelector: {}
  egress:
    - toEntities: []   # 명시적 허용만 통과
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-dns
  namespace: fqdn-lab
spec:
  endpointSelector: {}
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
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-fqdn
  namespace: fqdn-lab
spec:
  endpointSelector: {}
  egress:
    - toFQDNs:
        - matchName: "www.google.com"
        - matchName: "api.github.com"
    - toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

적용 후:

```bash
kubectl -n fqdn-lab apply -f 99-bundle-<calico|cilium>.yaml
kubectl -n fqdn-lab run tbox --image=curlimages/curl -it --rm -- \
  sh -lc 'curl -sI https://api.github.com | head -n1; curl -sI https://www.naver.com | head -n1'
```

---

## 결론

**DNS 기반 네트워크 정책**은 **IP 변동이 잦은 외부 서비스**를 안전히 제어하는 데 핵심이다.
- **Calico** 와 **Cilium** 모두 DNS(FQDN) egress 제어를 제공하지만, **문법·와일드카드 지원** 등이 다르다.
- 실무에선 **DNS 허용 → FQDN Allowlist → 기본 차단**의 레이어링과 **TTL/IPv6/장기연결**을 함께 설계하라.
- 더 세밀한 제어(메서드/경로/헤더)는 **L7(메쉬/프록시/Calico L7)** 와 결합해 **제로트러스트**에 가까운 통제를 완성하라.
