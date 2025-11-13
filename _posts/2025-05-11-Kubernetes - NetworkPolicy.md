---
layout: post
title: Kubernetes - NetworkPolicy
date: 2025-05-11 20:20:23 +0900
category: Kubernetes
---
# NetworkPolicy로 네트워크 제어하기

Kubernetes 클러스터는 **정책이 없으면 Pod 간 통신이 모두 허용(allow all)** 된다.
보안이 필요한 환경에서는 서비스 간 최소 권한 원칙(least privilege)에 따라 **명시적으로 허용한 연결만** 지나가도록 **NetworkPolicy**를 적용해야 한다.

- CNI 요구사항과 **기본 동작**(정책이 있으면 차단이 기본)
- **Ingress/Egress** 필터링, **네임스페이스/Pod/IP** 선택자
- **포트/프로토콜**, **포트 범위(endPort)**, **Named port** 활용
- **기본 거부** 템플릿 + **DNS/Egress 화이트리스트** 패턴
- 실전 시나리오(테넌트 격리, 외부 접근 제한, 외부 API만 허용 등)
- **테스트/검증 방법**(busybox/curl/netshoot), **트러블슈팅** 팁

> **주의:** NetworkPolicy가 동작하려면 **CNI 플러그인**이 이를 **지원**해야 한다. 예: **Calico, Cilium, Weave Net(일부)**. **Flannel 단독**은 기본적으로 미지원.

---

## 1. 개념과 동작 원리

- **NetworkPolicy가 하나도 없으면:** 모든 Pod ↔ Pod 통신 **허용**
- **하나라도 적용되면:** 해당 정책이 **선택한 Pod**에 대해 **명시된 트래픽만 허용**, 나머지 **차단**
- 정책은 **순서가 없고** 서로 **합집합(additive)** 으로 동작한다.
  - 같은 Pod를 선택하는 여러 정책이 있을 때, **어느 한 정책이라도 허용**하면 통과
  - 반대로 **하나도 허용 규칙에 걸리지 않으면 차단**

### 기본 스키마(요약)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: <policy-name>
  namespace: <target-namespace>
spec:
  podSelector: <이 정책의 대상 Pod 선택자>   # 비우면 '{}' = 네임스페이스의 모든 Pod
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:    # 누가 들어올 수 있는가?
        - podSelector: { matchLabels: {...} }
        - namespaceSelector: { matchLabels: {...} }
        - ipBlock: { cidr: 10.0.0.0/8, except: [10.10.0.0/16] }
      ports:
        - protocol: TCP
          port: 80
          endPort: 90      # 포트 범위(옵션)
  egress:
    - to:      # 어디로 나갈 수 있는가?
        - podSelector: {...}
        - namespaceSelector: {...}
        - ipBlock: {...}
      ports:
        - protocol: UDP
          port: 53
```

> **포인트**
> - `podSelector`는 **대상 Pod**(= 보호받을 Pod)를 지정한다.
> - `ingress.from`/`egress.to`는 **통신 상대**를 지정한다.
> - `policyTypes`를 생략하면, `ingress`가 있으면 Ingress, `egress`가 있으면 Egress가 자동 유추된다.

---

## 2. 준비: 네임스페이스/라벨 구성

테스트용 네임스페이스와 라벨을 만든다.

```bash
kubectl create ns team-a
kubectl create ns team-b

kubectl label ns team-a team=a
kubectl label ns team-b team=b
```

테스트 Pod:

```bash
# team-a에 nginx 서버와 curl 클라이언트
kubectl -n team-a run nginx --image=nginx:1.25 --port=80 --labels app=nginx
kubectl -n team-a expose pod nginx --port=80

kubectl -n team-a run curl --image=curlimages/curl:8.9.1 -it --restart=Never -- /bin/sh

# team-b에 curl 클라이언트
kubectl -n team-b run curl --image=curlimages/curl:8.9.1 -it --restart=Never -- /bin/sh
```

기본 상태에서는 **어디서나 접근 가능**하다.

```bash
# team-b 쉘에서:
curl -sS http://nginx.team-a.svc.cluster.local
```

---

## 3. 기본 거부(deny)부터: “명시한 것만 허용”

### 3.1 Ingress 기본 거부

```yaml
# deny-all-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: team-a
spec:
  podSelector: {}           # team-a의 모든 Pod 대상
  policyTypes: ["Ingress"]  # Ingress 차단
```

```bash
kubectl apply -f deny-all-ingress.yaml
```

이제 **아무도 team-a의 Pod에 들어올 수 없다.**
확인:

```bash
# team-b에서 nginx.team-a로 접근: 타임아웃/거부되어야 함
curl -m 3 -sS http://nginx.team-a.svc.cluster.local || echo "blocked"
```

### 3.2 같은 네임스페이스 내부만 허용

```yaml
# allow-from-same-ns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-same-namespace
  namespace: team-a
spec:
  podSelector: { }          # team-a의 모든 Pod가 보호 대상
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector: {}   # 같은 네임스페이스 내 모든 Pod 허용
```

```bash
kubectl apply -f allow-from-same-ns.yaml
```

검증:

```bash
# team-a의 curl Pod에서 접근: 허용
curl -sS http://nginx.team-a.svc.cluster.local | head -n1

# team-b의 curl Pod에서 접근: 여전히 차단
```

### 3.3 특정 포트만 허용(예: 80/TCP)

```yaml
# allow-port-80-only.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-80-only
  namespace: team-a
spec:
  podSelector: { app: nginx }    # nginx 라벨의 Pod만 보호
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector: {}        # 같은 NS 모두(상황에 따라 좁히기)
      ports:
        - protocol: TCP
          port: 80
```

> ✅ **Named port** 사용 시: `port: http` 처럼 Deployment의 컨테이너 포트 이름과 매칭 가능
> ✅ **포트 범위**는 `endPort`로 지정할 수 있다(동일 프로토콜/연속 범위).

---

## 4. 네임스페이스 간 격리: team-a만 허용

### 4.1 네임스페이스 라벨 기반 허용

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
              team: a     # team=a 라벨 네임스페이스에서 오는 트래픽만 허용
```

검증:

```bash
# team-a -> team-a: 허용
# team-b -> team-a: 차단
```

**중요:** `namespaceSelector`를 사용하려면 대상 네임스페이스에 라벨이 정확히 있어야 한다(이미 앞에서 부여).

---

## 5. Egress 제어: 외부로 나가는 트래픽 제한

### 5.1 Egress 기본 거부

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
  egress: []   # 아무 데도 나가지 못함
```

### 5.2 DNS 허용(필수)

많은 애플리케이션은 DNS가 없으면 외부 연결이 모두 실패한다. `kube-dns(CoreDNS)`가 있는 네임스페이스(kube-system)로의 53/UDP/TCP를 허용한다.

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

### 5.3 특정 외부 대역/포트만 화이트리스트

예: 인터넷(0.0.0.0/0) 중 내부망(10.0.0.0/8)은 제외하고 **443/TCP만** 허용

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

> `ipBlock`은 **클러스터 외부 IP** 매칭에 적합하다. 클러스터 내부 Pod/Service에는 `podSelector/namespaceSelector`가 권장된다.

---

## 6. 고급 예제 모음

### 6.1 특정 앱 라벨만 접근 허용

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
            matchLabels: { app: foo }   # 같은 NS의 foo 라벨 Pod만 허용
      ports:
        - { protocol: TCP, port: 80 }
```

### 6.2 egress 대상이 내부 Service일 때(네임스페이스 분리)

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

### 6.3 포트 범위 허용(endPort)

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
          endPort: 30010   # 30000~30010 허용
```

### 6.4 IPv6 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-ipv6
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
    - to:
        - ipBlock:
            cidr: "2001:db8::/32"
      ports:
        - { protocol: TCP, port: 443 }
```

---

## 7. 테스트/검증 레시피

### 7.1 도구 Pod

- **curl**: `curlimages/curl:latest`
- **busybox**: `busybox:1.36` (`wget` 내장)
- **netshoot**: `nicolaka/netshoot` (dig, tcpdump 등 풍부한 툴)

```bash
kubectl -n team-a run netshoot --image=nicolaka/netshoot -it --rm -- /bin/bash
```

### 7.2 연결 테스트 패턴

```bash
# DNS 확인
dig +short nginx.team-a.svc.cluster.local

# 포트 체크
curl -v telnet://nginx.team-a.svc.cluster.local:80
# 또는
nc -vz nginx.team-a.svc.cluster.local 80

# 외부(예: https)
curl -I https://example.com
```

> **실패가 정상**인 시나리오가 많다(deny-by-default). 항상 **기대 동작**과 비교하라.

---

## 8. 운영 팁 & 베스트 프랙티스

- **기본 거부(deny-all)** 부터 시작해 **필요한 것만 허용**(화이트리스트)
- **DNS egress**를 잊지 말 것(53/UDP/TCP). DNS가 막히면 모든 외부 접근이 무너진다.
- **라벨 표준화**: `app`, `role`, `team` 같은 공통 라벨을 정책에 활용
- **Service 접근을 열고 싶다면** 대상 Pod 라벨을 기준으로 허용(서비스 이름 자체가 아닌 **Pod**이 매칭됨)
- **Liveness/Readiness Probe**가 막히는 경우
  - 프로브는 **노드 IP**에서 들어올 수 있다(특히 hostNetwork/구성에 따라). 필요 시 노드 대역을 `ipBlock`으로 허용.
- **정책 영향 범위를 최소화**: 처음에는 특정 앱/네임스페이스에만 적용해 검증 후 범위 확대
- **정책은 합집합**: 같은 Pod에 여러 정책이 적용될 수 있으며, 허용 규칙이 **하나라도** 매칭되면 통과
- **CNI별 세부 차이**: iptables/eBPF 엔진에 따른 처리 방식 차이는 있으나 **스펙 해석은 동일**해야 한다. 확장 기능(L7 등)은 CNI 고유 리소스를 사용(Calico/Cilium).

---

## 9. 시나리오 통합 실습(한 번에 적용)

아래 번들은 **team-a를 기본 차단**, **동일 NS 허용**, **포트80만**, **DNS & 외부 443만 허용**을 묶은 **현실적인 스타터 정책**이다.

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

적용:

```bash
kubectl apply -f 01-deny-all-ingress.yaml \
              -f 02-allow-same-ns-on-80.yaml \
              -f 03-deny-all-egress.yaml \
              -f 04-allow-dns-egress.yaml \
              -f 05-allow-web443-egress.yaml
```

검증 포인트:

- **Ingress**: team-a 내부에서 nginx:80 **OK**, team-b에서 **차단**
- **Egress**: 외부 **443만 허용**, 80/22 등은 **차단**
- **DNS**: 도메인 이름 접근 가능(53 허용 덕분)

---

## 10. 트러블슈팅 가이드

| 증상 | 원인 후보 | 해결 방안 |
|---|---|---|
| 갑자기 통신 전부 차단 | deny-all를 먼저 적용, 허용 정책 누락 | 허용 정책(ingress/egress) 추가, 정책 순서 무관/합집합임을 유념 |
| 서비스 이름으로는 안 열리고 IP로만 되거나 반대 | Service가 아니라 Pod 라벨 기준으로 매칭 | 대상 **Pod 라벨**과 **namespaceSelector** 사용, Service는 선택자가 아니다 |
| 외부 연결이 전부 실패 | DNS egress 미허용 | `kube-system`의 CoreDNS 53/UDP+TCP 허용 |
| Liveness/Readiness Probes 실패 | Prober 소스가 노드 IP | 노드 대역 `ipBlock` 허용 or 서비스 경유 탐색 시 네임스페이스/포트 허용 |
| team-b가 team-a 접근 가능 | 라벨 오타/namespaceSelector 불일치 | 네임스페이스 라벨 재확인(`kubectl get ns --show-labels`) |
| 규칙 적용된 것 같지 않음 | CNI가 NetworkPolicy 미지원 | **Calico/Cilium** 등 지원 CNI 사용 여부 확인 |
| 특정 포트만 열었는데 다른 포트도 열림 | 다른 정책이 같은 Pod에 허용 중 | 같은 Pod에 적용되는 **모든 정책** 점검(합집합) |
| IPv6 트래픽이 이상 | ipBlock IPv6 누락 | 별도 IPv6 `cidr`와 포트 허용 추가 |

---

## 11. CNI 지원 현황(요지)

| CNI | NetworkPolicy |
|---|---|
| Calico | ✅ (완전 지원) |
| Cilium | ✅ (완전 지원) |
| Weave Net | ✅ (일부) |
| Flannel | ❌ (기본 미지원) |

> 고급 L7 제어가 필요하면 CNI 고유 확장(예: Calico/Cilium의 L7 정책)을 검토하자. **표준 NetworkPolicy**는 L3/L4에 초점을 둔다.

---

## 마무리

- **정책이 없으면 모두 허용** → **정책이 생기면 명시 허용 외 차단**
- 기본 전략은 **deny-all부터 시작**해 필요한 **Ingress/Egress만 화이트리스트**
- **DNS egress**, **네임스페이스/Pod 라벨 표준화**, **테스트 도구**를 습관화
- 정책은 **합집합(additive)**. 같은 Pod에 여러 정책이 동시에 적용될 수 있으니 충돌/중복을 이해하고 설계할 것
- CNI 지원을 **반드시 확인**하라

위 템플릿과 절차대로 적용·검증하면, 운영 환경에서도 **예측 가능**하고 **안전한 네트워크 경계**를 빠르게 구축할 수 있다.
