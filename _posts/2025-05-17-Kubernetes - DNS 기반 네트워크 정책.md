---
layout: post
title: Kubernetes - DNS 기반 네트워크 정책
date: 2025-05-11 21:20:23 +0900
category: Kubernetes
---
# DNS 기반 네트워크 정책 구성하기

Kubernetes 기본 네트워크 정책은 보통 **Pod, IP, Namespace** 기준입니다.  
하지만 실제 서비스에서는 **`example.com`, `api.external.com`과 같은 DNS 기반 제어**가 필요할 때가 많습니다.

이를 위해 Cilium, Calico 등 **고급 CNI 플러그인**을 통해 **FQDN 기반 네트워크 정책**을 구성할 수 있습니다.

---

## ✅ 왜 DNS 기반 정책이 필요한가?

| 기존 방식 | 한계점 |
|-----------|--------|
| IP 기반 | 외부 서비스의 IP가 자주 바뀌는 경우 대응 불가 |
| Pod/Ns 기반 | 외부 도메인 제어 불가 |
| egress: to IP | CDN, DNS 라운드로빈 등에서 한계 |

> DNS 이름으로 제어하면 **도메인 기준 제어**, **자동 IP 업데이트**, **보다 명시적인 정책** 가능

---

## ✅ DNS 기반 네트워크 정책이 가능한 CNI

| CNI | FQDN 지원 여부 | 비고 |
|-----|----------------|------|
| **Calico** | ✅ 지원 (`GlobalNetworkPolicy` with FQDN) | OSS 버전도 가능 |
| **Cilium** | ✅ 지원 (`ToFQDNs`) | DNS 캐싱, L7 정책도 병행 |
| Kube-router / Flannel / 기본 CNI | ❌ 미지원 | 외부 정책 불가 |

---

## ✅ 실습: Calico로 FQDN 제어하기

### 1. Calico 설치 확인

```bash
kubectl get pods -n kube-system | grep calico
```

> 없으면 설치:  
> `https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed/onprem/kubeadm/`

---

### 2. 테스트용 Pod 배포

```bash
kubectl run testpod --image=busybox -it --rm --restart=Never -- sh
```

→ 내부에서 `wget google.com` 시도해볼 수 있게 함

---

### 3. FQDN 기반 GlobalNetworkPolicy 생성

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-to-google
spec:
  selector: all()  # 모든 Pod에 적용
  egress:
  - action: Allow
    protocol: TCP
    to:
    - dns:
        names:
        - "www.google.com"
  - action: Deny
    destination:
      nets:
        - 0.0.0.0/0
  types:
  - Egress
```

```bash
kubectl apply -f allow-google.yaml
```

---

### 4. 테스트

```sh
wget www.google.com  # ✅ 성공
wget www.naver.com   # ❌ 실패 (deny됨)
```

> Calico는 **`dns.names` 필드**로 FQDN 기반 정책 설정 가능  
> DNS → IP 매핑은 Calico가 주기적으로 캐싱하여 유지

---

## ✅ Cilium 기반 FQDN 정책 (ToFQDNs)

### 1. Cilium 설치 (eBPF 기반 CNI)

```bash
minikube start --network-plugin=cni --cni=false
minikube addons enable cilium
```

### 2. 정책 예시: `example.com`만 허용

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-fqdn
spec:
  endpointSelector: {}
  egress:
  - toFQDNs:
    - matchName: "example.com"
```

> `matchPattern`, `matchRegex`도 가능

---

## ✅ 동작 원리 요약

1. Pod에서 DNS 쿼리 발생 (ex. `example.com`)
2. CNI(Calico/Cilium)가 해당 도메인의 IP를 캐싱
3. 이후 실제 IP에 대한 패킷을 **캐싱된 도메인과 비교**
4. 정책에 맞는 경우 허용, 아니면 차단

---

## ✅ 관리 팁

- 도메인 별 TTL 주의: IP 변화 주기 따라 캐시 주기도 설정 필요
- DNS 쿼리가 실패하거나, 도메인 미해석 시 egress 차단될 수 있음
- 외부 도메인 많을 경우 wildcard 사용 (`*.example.com`) 가능 (Cilium)

---

## ✅ DNS 정책의 한계 및 대안

| 한계점 | 설명 |
|--------|------|
| DNS Spoofing 우려 | IP 캐시를 신뢰해야 하므로 보안 위험 존재 |
| 정확한 제어 어려움 | CDN, 라운드 로빈 등 복수 IP의 경우 일관된 제어 어려움 |
| 성능 영향 | 도메인 변화가 많으면 캐싱/업데이트 부하 |

### 대안

- [ ] DNS 기반 정책 + IP allowlist 병행
- [ ] DNS over TLS/HTTPS 사용 권장
- [ ] Istio + Envoy를 통한 외부 도메인 제어

---

## ✅ 결론

| 구성 | 설명 |
|------|------|
| 목적 | 외부 도메인 접근 허용/차단 |
| 대상 CNI | Calico, Cilium 등 |
| 키 필드 | `dns.names`, `toFQDNs.matchName` |
| 장점 | 명시적인 도메인 제어, 유연한 정책 |
| 주의점 | 캐시 TTL, IP 변경, 보안 우려 고려 |

> **DNS 기반 네트워크 정책은 클라우드-네이티브 보안 구성에 매우 유용한 도구**입니다.  
> FQDN 기반 제어로 **외부 API 서버, SaaS, 서드파티 통신을 세밀하게 제한**할 수 있습니다.

---

## ✅ 참고 링크

- [Calico FQDN 정책 문서](https://docs.tigera.io/security/fqdn/fqdn-policy)
- [Cilium FQDN 정책](https://docs.cilium.io/en/latest/policy/language/#dns-based)
- [Kubernetes NetworkPolicy 공식 문서](https://kubernetes.io/docs/concepts/services-networking/network-policies/)