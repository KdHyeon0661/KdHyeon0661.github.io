---
layout: post
title: Kubernetes - NetworkPolicy
date: 2025-05-11 20:20:23 +0900
category: Kubernetes
---
# NetworkPolicy로 네트워크 제어하기

Kubernetes 클러스터에서 Pod 간 통신은 기본적으로 **모두 허용**되어 있습니다.  
하지만 보안이 중요한 환경에서는 서비스 간 **네트워크 접근을 제한**해야 합니다.

이를 위한 Kubernetes의 기본 기능이 바로 **`NetworkPolicy`**입니다.

---

## ✅ NetworkPolicy란?

`NetworkPolicy`는 **Pod 레벨에서 네트워크 트래픽을 제어**하는 Kubernetes 리소스입니다.

- **Ingress**: 외부에서 들어오는 트래픽 제어
- **Egress**: Pod에서 나가는 트래픽 제어
- **Pod Selector**, **Namespace Selector**, **IPBlock** 등을 기준으로 제어

> 단, `NetworkPolicy`가 적용되려면 **네트워크 플러그인(CNI)**이 이를 **지원**해야 합니다.  
> (예: Calico, Cilium, Weave Net 등)

---

## ✅ 기본 동작 원리

1. **NetworkPolicy가 없으면** 모든 Pod 간 통신이 허용됨
2. **NetworkPolicy가 존재하면**, 명시된 정책 외에는 차단됨 (**기본 차단**)

---

## ✅ 예제 시나리오

- `namespace: team-a`의 Pod는 서로 통신 가능
- `namespace: team-b`에서는 team-a에 접근 불가능
- 특정 IP나 포트만 허용할 수도 있음

---

## ✅ 1. NetworkPolicy 기본 구조

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-team-a
  namespace: team-a
spec:
  podSelector: {}  # 이 네임스페이스의 모든 Pod 대상으로
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: a
```

- `podSelector`: 정책이 적용될 대상
- `from`: 접근을 허용할 대상 (namespace / pod / IP)
- `to`: egress 정책에서 사용
- `ports`: 포트를 기반으로 필터링 가능

---

## ✅ 2. 실습 예제: team-a는 허용, 나머지는 차단

### 네임스페이스 생성

```bash
kubectl create ns team-a
kubectl create ns team-b

kubectl label namespace team-a team=a
kubectl label namespace team-b team=b
```

### 테스트용 Pod 배포

```bash
kubectl run nginx --image=nginx -n team-a
kubectl run curl --image=appropriate/curl -it --restart=Never -n team-b -- /bin/sh
```

### 기본 상태 확인 (모두 접근 가능)

```bash
curl http://nginx.team-a.svc.cluster.local
```

→ team-b에서 접근 가능

---

## ✅ 3. NetworkPolicy 적용: team-a 간 통신만 허용

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-only-team-a
  namespace: team-a
spec:
  podSelector: {}  # team-a의 모든 Pod 대상
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: a
```

```bash
kubectl apply -f allow-only-team-a.yaml
```

### 다시 curl 시도

```bash
curl http://nginx.team-a.svc.cluster.local
```

- team-a 내부에서: 성공
- team-b에서: 연결 실패

---

## ✅ 4. 특정 포트만 허용 (예: 80)

```yaml
ingress:
  - from:
      - podSelector: {}
    ports:
      - protocol: TCP
        port: 80
```

→ TCP 80번 포트만 허용, 나머지는 차단됨

---

## ✅ 5. Egress 제어: 외부로 나가는 트래픽 제한

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-external
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
```

→ `team-a`의 Pod는 **10.0.0.0/24** 대역으로만 나갈 수 있고, 외부 인터넷은 차단됨

---

## ✅ 6. 모든 Ingress 차단 (기본 정책)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: secure-ns
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

→ 아무것도 명시하지 않으면 **모든 Ingress 트래픽 차단**

---

## ✅ 7. 동작 확인 팁

```bash
# 현재 NetworkPolicy 목록
kubectl get networkpolicy -A

# 특정 Pod의 IP 확인
kubectl get pod -o wide

# curl 로 연결 시도
kubectl run curl --image=appropriate/curl -it --restart=Never -- sh
```

→ 연결 실패 시 timeout 발생

---

## ✅ 8. 네트워크 플러그인 주의사항

| CNI 플러그인 | NetworkPolicy 지원 여부 |
|--------------|--------------------------|
| Calico | ✅ 완전 지원 |
| Cilium | ✅ 완전 지원 |
| Weave Net | ✅ 일부 지원 |
| Flannel | ❌ 미지원 (기본 CNI) |

Flannel만 사용하는 경우 **NetworkPolicy는 작동하지 않습니다**.

→ 해결 방법: Calico 또는 Cilium으로 CNI 변경 필요

---

## ✅ 결론

| 항목 | 설명 |
|------|------|
| NetworkPolicy | Pod 간 네트워크 트래픽 제어 |
| 기본 동작 | 정책 없으면 모두 허용, 있으면 명시된 것 외 차단 |
| ingress/egress | 들어오고 나가는 트래픽 각각 설정 가능 |
| 라벨 기반 제어 | Namespace, Pod, IP 블록 단위로 세분화 가능 |

> 보안이 중요한 클러스터에서는 최소 권한 원칙에 따라  
> 서비스 간 통신을 명시적으로 허용하고, 그 외는 차단하는 것이 권장됩니다.

---

## ✅ 참고 링크

- [Kubernetes 공식 NetworkPolicy 문서](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Calico CNI](https://docs.tigera.io/)
- [Cilium CNI](https://docs.cilium.io/)