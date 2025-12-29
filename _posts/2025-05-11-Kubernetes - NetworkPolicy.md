---
layout: post
title: Kubernetes - NetworkPolicy
date: 2025-05-11 20:20:23 +0900
category: Kubernetes
---
# Kubernetes NetworkPolicy로 네트워크 제어하기

Kubernetes 클러스터는 기본적으로 정책이 없으면 모든 Pod 간 통신이 허용됩니다. 보안이 중요한 환경에서는 최소 권한 원칙에 따라 명시적으로 허용한 연결만 통과하도록 NetworkPolicy를 적용해야 합니다.

NetworkPolicy는 Pod 간 통신을 제어하는 중요한 도구로, 다음과 같은 기능을 제공합니다:
- Ingress/Egress 트래픽 필터링
- 네임스페이스, Pod, IP 기반 선택자 활용
- 포트 및 프로토콜 제어
- 기본 거부 정책과 허용 목록 패턴 구현

> **중요**: NetworkPolicy가 동작하려면 CNI 플러그인이 이를 지원해야 합니다. Calico, Cilium, Weave Net(일부)은 지원하지만 Flannel은 기본적으로 지원하지 않습니다.

---

## 기본 개념과 동작 원리

NetworkPolicy의 기본 동작 방식을 이해하는 것이 중요합니다:

- **NetworkPolicy가 하나도 없는 경우**: 모든 Pod 간 통신이 허용됩니다.
- **하나 이상의 NetworkPolicy가 적용된 경우**: 정책이 선택한 Pod에 대해 명시된 트래픽만 허용되고, 나머지는 차단됩니다.
- **정책 적용 방식**: NetworkPolicy는 순서가 없으며, 서로 합집합 방식으로 동작합니다. 즉, 하나의 Pod에 여러 정책이 적용될 때, 어떤 정책이라도 트래픽을 허용하면 통과됩니다.

### 기본 NetworkPolicy 구조

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: <policy-name>
  namespace: <target-namespace>
spec:
  podSelector: <이 정책의 대상 Pod 선택자>   # 비어 있으면 네임스페이스의 모든 Pod
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:    # 허용할 수신 연결 원본
        - podSelector: { matchLabels: {...} }
        - namespaceSelector: { matchLabels: {...} }
        - ipBlock: { cidr: 10.0.0.0/8, except: [10.10.0.0/16] }
      ports:
        - protocol: TCP
          port: 80
          endPort: 90      # 포트 범위(선택사항)
  egress:
    - to:      # 허용할 발신 연결 대상
        - podSelector: {...}
        - namespaceSelector: {...}
        - ipBlock: {...}
      ports:
        - protocol: UDP
          port: 53
```

**핵심 요소 설명:**
- `podSelector`: 이 정책이 적용될 대상 Pod를 지정합니다.
- `ingress.from`/`egress.to`: 통신을 허용할 상대방을 지정합니다.
- `policyTypes`: 정책 유형을 명시합니다. 생략할 경우 `ingress` 또는 `egress` 섹션의 존재 여부에 따라 자동 유추됩니다.

---

## 실습 환경 준비

NetworkPolicy를 테스트하기 위한 기본 환경을 구성합니다.

```bash
# 테스트용 네임스페이스 생성
kubectl create ns team-a
kubectl create ns team-b

# 네임스페이스에 라벨 부여
kubectl label ns team-a team=a
kubectl label ns team-b team=b

# team-a에 테스트 애플리케이션 배포
kubectl -n team-a run nginx --image=nginx:1.25 --port=80 --labels app=nginx
kubectl -n team-a expose pod nginx --port=80

# 테스트용 클라이언트 Pod 생성
kubectl -n team-a run curl --image=curlimages/curl:8.9.1 -it --restart=Never -- /bin/sh
kubectl -n team-b run curl --image=curlimages/curl:8.9.1 -it --restart=Never -- /bin/sh
```

초기 상태에서는 모든 통신이 허용됩니다. team-b에서 team-a의 nginx 서비스에 접근해 확인할 수 있습니다:

```bash
# team-b의 curl Pod에서 실행
curl -sS http://nginx.team-a.svc.cluster.local
```

---

## 기본 접근 제어 정책 구현

### 1. 모든 수신 트래픽 차단 (Deny All Ingress)

가장 먼저 모든 수신 연결을 차단하는 기본 정책을 적용합니다.

```yaml
# deny-all-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: team-a
spec:
  podSelector: {}           # team-a의 모든 Pod 대상
  policyTypes: ["Ingress"]  # 수신 트래픽만 제어
```

```bash
kubectl apply -f deny-all-ingress.yaml
```

이제 team-b에서 team-a의 nginx에 접근할 수 없습니다:

```bash
# team-b curl Pod에서 실행
curl -m 3 -sS http://nginx.team-a.svc.cluster.local || echo "접근 차단됨"
```

### 2. 동일 네임스페이스 내 통신 허용

동일 네임스페이스 내에서는 통신이 필요할 수 있습니다. 다음과 같이 정책을 추가합니다.

```yaml
# allow-from-same-ns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-same-namespace
  namespace: team-a
spec:
  podSelector: {}           # team-a의 모든 Pod 대상
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector: {}   # 동일 네임스페이스의 모든 Pod 허용
```

```bash
kubectl apply -f allow-from-same-ns.yaml
```

이제 team-a 내부에서는 통신이 가능하지만, team-b에서의 접근은 여전히 차단됩니다.

### 3. 특정 포트만 허용

특정 애플리케이션에 대해 특정 포트만 허용하는 정책을 적용할 수 있습니다.

```yaml
# allow-port-80-only.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-80-only
  namespace: team-a
spec:
  podSelector: { app: nginx }    # nginx 라벨을 가진 Pod만 대상
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector: {}        # 동일 네임스페이스의 모든 Pod
      ports:
        - protocol: TCP
          port: 80
```

> **참고사항**:
> - Named Port 사용: `port: http`처럼 컨테이너 포트 이름을 사용할 수 있습니다.
> - 포트 범위: `endPort`를 사용하여 연속된 포트 범위를 지정할 수 있습니다.

---

## 네임스페이스 간 통신 제어

특정 네임스페이스에서만 접근을 허용하는 정책을 구성할 수 있습니다.

### 네임스페이스 라벨 기반 접근 허용

```yaml
# allow-ingress-from-namespace-label.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-team-a-ns
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: a     # team=a 라벨이 있는 네임스페이스만 허용
```

이 정책을 적용하면 team-a 라벨이 있는 네임스페이스에서만 team-a의 Pod에 접근할 수 있습니다.

---

## 발신 트래픽 제어 (Egress Control)

### 1. 모든 발신 트래픽 차단

발신 트래픽도 수신 트래픽처럼 기본적으로 차단할 수 있습니다.

```yaml
# deny-all-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress: []   # 모든 발신 트래픽 차단
```

### 2. DNS 서비스 접근 허용

대부분의 애플리케이션은 DNS 서비스가 없으면 외부 연결에 실패합니다. kube-system 네임스페이스의 CoreDNS에 대한 접근을 허용해야 합니다.

```yaml
# allow-egress-dns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-dns
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - { protocol: UDP, port: 53 }
        - { protocol: TCP, port: 53 }
```

### 3. 특정 외부 주소 및 포트 허용

외부 웹 서비스에 대한 접근을 제한할 수 있습니다. 다음 예제는 내부 네트워크(10.0.0.0/8)를 제외한 모든 주소에서 443 포트로의 접근만 허용합니다.

```yaml
# allow-egress-web443.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web443
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
      ports:
        - { protocol: TCP, port: 443 }
```

> **중요**: `ipBlock`은 주로 클러스터 외부 IP 주소와 매칭할 때 사용합니다. 클러스터 내부 Pod나 Service에 대한 접근은 `podSelector`나 `namespaceSelector`를 사용하는 것이 권장됩니다.

---

## 고급 구성 예제

### 특정 애플리케이션 간 통신 허용

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: only-app-foo-can-access
  namespace: team-a
spec:
  podSelector:
    matchLabels: { app: nginx }
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: foo }   # app=foo 라벨을 가진 Pod만 허용
      ports:
        - { protocol: TCP, port: 80 }
```

### 네임스페이스 간 발신 통신 허용

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-backend
  namespace: team-a
spec:
  podSelector:
    matchLabels: { app: frontend }
  policyTypes: ["Egress"]
  egress:
    - to:
        - namespaceSelector: { matchLabels: { team: b } }
          podSelector: { matchLabels: { app: backend } }
      ports:
        - { protocol: TCP, port: 8080 }
```

### 포트 범위 허용

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-range
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 30000
          endPort: 30010   # 30000~30010 포트 범위 허용
```

---

## 테스트 및 검증 방법

### 테스트용 도구 Pod

다양한 네트워크 테스트 도구를 포함한 Pod를 사용하여 정책을 검증할 수 있습니다:

```bash
# netshoot 이미지 사용 (네트워크 진단 도구 모음)
kubectl -n team-a run netshoot --image=nicolaka/netshoot -it --rm -- /bin/bash

# 또는 간단한 curl/busybox 사용
kubectl -n team-a run test --image=curlimages/curl:latest -it --rm -- /bin/sh
```

### 연결 테스트 패턴

```bash
# DNS 확인
dig +short nginx.team-a.svc.cluster.local

# 포트 연결 테스트
nc -vz nginx.team-a.svc.cluster.local 80

# HTTP 연결 테스트
curl -v http://nginx.team-a.svc.cluster.local

# 외부 연결 테스트
curl -I https://example.com
```

---

## 운영 모범 사례

1. **기본 거부 원칙**: 항상 모든 트래픽을 차단하는 정책부터 시작하여 필요한 통신만 허용하는 화이트리스트 방식을 적용하세요.
2. **DNS 접근 허용**: DNS 서비스(53/UDP 및 53/TCP)에 대한 접근을 허용하지 않으면 외부 연결이 실패합니다.
3. **일관된 라벨링 전략**: `app`, `team`, `environment` 같은 표준 라벨을 사용하여 정책 관리를 단순화하세요.
4. **Liveness/Readiness Probe 고려**: 노드에서 실행되는 헬스 체크 프로브가 차단되지 않도록 적절한 허용 정책을 구성하세요.
5. **점진적 적용**: 먼저 특정 네임스페이스나 애플리케이션에 정책을 적용하여 검증한 후 점진적으로 범위를 확대하세요.
6. **정책 영향 범위 최소화**: 가능한 한 구체적으로 대상을 지정하여 의도하지 않은 통신이 허용되지 않도록 하세요.

---

## 통합 실습: 현실적인 네트워크 정책 구성

다음은 실제 운영 환경에서 사용할 수 있는 종합적인 NetworkPolicy 설정 예제입니다. 이 번들은 기본 거부 정책부터 시작하여 필요한 통신만 허용하는 완전한 접근 제어를 구현합니다.

```yaml
# 01-deny-all-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
---
# 02-allow-same-ns-on-80.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-ns-on-80
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector: {}
      ports:
        - { protocol: TCP, port: 80 }
---
# 03-deny-all-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress: []
---
# 04-allow-dns-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - { protocol: UDP, port: 53 }
        - { protocol: TCP, port: 53 }
---
# 05-allow-web443-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web443
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - { protocol: TCP, port: 443 }
```

적용 방법:

```bash
kubectl apply -f 01-deny-all-ingress.yaml \
              -f 02-allow-same-ns-on-80.yaml \
              -f 03-deny-all-egress.yaml \
              -f 04-allow-dns-egress.yaml \
              -f 05-allow-web443-egress.yaml
```

**검증 포인트:**
- 수신: team-a 내부에서 nginx의 80 포트 접근 가능, team-b에서는 차단
- 발신: 외부 443 포트 접근 가능, 80/22 포트 등은 차단
- DNS: 도메인 이름 해석 가능

---

## 일반적인 문제 해결

| 증상 | 가능한 원인 | 해결 방안 |
|---|---|---|
| 모든 통신이 갑자기 차단됨 | deny-all 정책만 적용되고 허용 정책이 누락됨 | 필요한 허용 정책 추가, 정책이 합집합으로 동작함을 확인 |
| 서비스 이름으로 접근 불가, IP로만 가능 | 정책이 Service가 아닌 Pod 라벨을 기준으로 적용됨 | 대상 Pod의 라벨과 namespaceSelector를 올바르게 사용 |
| 외부 연결 실패 | DNS 발신 접근이 허용되지 않음 | kube-system 네임스페이스의 CoreDNS 53/UDP 및 53/TCP 접근 허용 |
| 헬스 체크 프로브 실패 | 노드 IP에서의 접근이 차단됨 | 노드 대역을 ipBlock으로 허용하거나 서비스 경유 접근 허용 |
| 예상대로 정책이 적용되지 않음 | CNI 플러그인이 NetworkPolicy를 지원하지 않음 | Calico, Cilium 등 지원 CNI로 변경 |
| 특정 포트만 열었는데 다른 포트도 접근 가능 | 다른 정책이 같은 Pod에 허용 규칙을 제공함 | Pod에 적용된 모든 정책 검토(합집합 방식) |

---

## CNI 플러그인 지원 현황

| CNI 플러그인 | NetworkPolicy 지원 |
|---|---|
| Calico | 완전 지원 |
| Cilium | 완전 지원 |
| Weave Net | 부분 지원 |
| Flannel | 기본적으로 미지원 |

> 고급 L7 네트워크 제어가 필요한 경우 CNI별 확장 기능(Calico/Cilium의 L7 정책 등)을 고려할 수 있습니다. 표준 NetworkPolicy는 L3/L4 수준의 제어에 초점을 맞추고 있습니다.

---

## 결론

Kubernetes NetworkPolicy는 클러스터 내 네트워크 보안을 강화하는 핵심 도구입니다. 효과적인 NetworkPolicy 구성을 위해서는 다음 원칙을 준수해야 합니다:

1. **기본 거부 원칙**: 모든 트래픽을 기본적으로 차단하고 필요한 통신만 명시적으로 허용하는 화이트리스트 방식을 적용하세요.
2. **점진적 구현**: 먼저 기본 거부 정책을 적용한 후, 애플리케이션 요구사항에 따라 필요한 허용 정책을 단계적으로 추가하세요.
3. **DNS 접근 보장**: DNS 서비스에 대한 접근을 허용하지 않으면 대부분의 외부 연결이 실패하므로, 항상 DNS 접근 정책을 우선적으로 구성하세요.
4. **일관된 라벨링**: 표준화된 라벨링 전략을 수립하여 정책 관리의 복잡성을 줄이세요.
5. **철저한 테스트**: 모든 정책을 적용하기 전에 테스트 환경에서 충분히 검증하고, 실제 적용 후에도 지속적으로 모니터링하세요.

이 문서에서 제시한 예제와 모범 사례를 참고하여 조직의 보안 요구사항에 맞는 NetworkPolicy를 체계적으로 구성하시기 바랍니다. 올바르게 구성된 NetworkPolicy는 Kubernetes 환경의 보안 수준을 크게 향상시키고, 잠재적인 공격 경로를 효과적으로 차단할 수 있습니다.