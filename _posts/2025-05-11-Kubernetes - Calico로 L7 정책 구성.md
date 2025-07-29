---
layout: post
title: Kubernetes - Calico로 L7 정책 구성
date: 2025-05-11 21:20:23 +0900
category: Kubernetes
---
# Calico로 L7 정책 구성하기

Kubernetes의 기본 `NetworkPolicy`는 **L3(네트워크)**와 **L4(전송)** 수준에서만 제어됩니다.

- ✅ IP 기반 (Pod, Namespace)
- ✅ 포트 기반 (TCP/UDP)
- ❌ HTTP 경로, 메서드, Hostname 기반 제어는 불가

**Calico**는 이를 넘어서 **Application Layer (L7)** 수준까지 제어 가능한 **고급 네트워크 정책** 기능을 제공합니다.

---

## ✅ L7 정책이란?

L7 정책은 다음과 같은 HTTP 수준의 제어를 가능하게 합니다:

| 항목 | 설명 |
|------|------|
| `method` | GET, POST 등 요청 메서드 제어 |
| `path` | `/api/v1/*` 같은 URL 경로 기준 필터 |
| `host` | `example.com` 등 도메인 이름 기반 |
| `headers` | 특정 HTTP 헤더 기반 정책 |

---

## ✅ Calico 설치 확인

Calico가 설치되어 있어야 하며, **[Calico Enterprise가 아닌 OSS 버전도 L7 지원 가능](https://docs.tigera.io/try)**입니다.

- CNI로 Calico를 사용하는 클러스터에서 진행해야 합니다.
- `calicoctl` CLI 설치 필요 (선택)

설치 예시 (Kubernetes용):

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

---

## ✅ 1. HTTPApplicationPolicy 리소스 소개

Calico의 **Application Layer Policy**는 `HTTPPolicy` 리소스를 사용합니다.

예를 들어:

```yaml
apiVersion: projectcalico.org/v3
kind: HTTPPolicy
metadata:
  name: allow-get-api
spec:
  selector: app == "backend"
  rules:
    - method: GET
      path: /api/*
```

→ `app=backend`인 Pod는 `/api/`로 시작하는 GET 요청만 허용됨

---

## ✅ 2. 실습 예제: frontend → backend 간 HTTP 제어

### ① 테스트 앱 배포

```bash
kubectl create ns demo

kubectl run backend --image=kennethreitz/httpbin --port=80 -n demo \
  --labels="app=backend"

kubectl expose pod backend --port=80 --target-port=80 -n demo

kubectl run frontend --image=busybox -it --rm --restart=Never -n demo -- sh
```

### ② 기본 curl 테스트

```sh
wget -qO- http://backend.demo.svc.cluster.local/get
```

→ 정상 응답 확인

---

## ✅ 3. L7 정책 적용

### ① HTTPPolicy 생성

```yaml
apiVersion: projectcalico.org/v3
kind: HTTPPolicy
metadata:
  name: restrict-post
spec:
  selector: app == "backend"
  rules:
    - method: GET
      path: /get
```

```bash
kubectl apply -f http-policy.yaml
```

### ② GlobalNetworkPolicy로 L7 정책 연결

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: restrict-backend-access
spec:
  selector: app == "backend"
  types:
    - Ingress
  ingress:
    - action: Allow
      protocol: TCP
      http:
        methods: ["GET"]
        paths: ["/get"]
    - action: Deny
```

```bash
kubectl apply -f gnp-l7.yaml
```

---

## ✅ 4. 테스트 결과

### ① 허용된 요청

```sh
wget -qO- http://backend.demo.svc.cluster.local/get
```

✅ 응답이 잘 옴

### ② 차단된 요청

```sh
wget -qO- --post-data="test" http://backend.demo.svc.cluster.local/post
```

❌ 연결 끊김 또는 HTTP 403 반환

---

## ✅ 5. 기타 고급 L7 제어 예시

### ✅ 특정 헤더 허용

```yaml
http:
  methods: ["GET"]
  paths: ["/secure/*"]
  headers:
    X-Auth-Token: "secret123"
```

### ✅ Hostname 기반 제어

```yaml
http:
  hosts: ["admin.example.com"]
```

---

## ✅ 6. 정책 순서와 우선순위

- Calico는 **GlobalNetworkPolicy**, **NetworkPolicy**, **HTTPPolicy**를 함께 사용 가능
- `Deny`를 마지막에 명시하지 않으면 허용되지 않은 요청도 통과할 수 있음
- 우선순위 조정은 `order` 필드를 통해 설정 가능

```yaml
metadata:
  name: deny-all
spec:
  order: 1000
  ingress:
    - action: Deny
```

---

## ✅ 7. 대시보드와 관측

- Calico Enterprise를 사용하면 **GUI 기반 정책 시각화** 가능
- OSS 환경에서도 **Prometheus + Grafana 연동**으로 메트릭 확인 가능

---

## ✅ 8. Calico 정책 vs K8s NetworkPolicy 차이점

| 항목 | Kubernetes NetworkPolicy | Calico Policy |
|------|--------------------------|---------------|
| 계층 | L3/L4 | L3/L4/L7 (HTTP) |
| 리소스 | `NetworkPolicy` | `GlobalNetworkPolicy`, `HTTPPolicy` |
| 기능 | 기본 네트워크 제어 | 세밀한 트래픽 제어, DNS 기반, FQDN, Host 제한 등 |
| 설치 필요 | 기본 제공 (단, CNI 필요) | Calico 설치 필수 |
| 정책 범위 | Namespace 내 | 전체 클러스터(Global) 가능 |

---

## ✅ 결론

Calico를 사용하면 L7 수준의 정교한 네트워크 정책을 구현할 수 있습니다:

- `/api/v1/*` URL만 허용
- POST 요청 차단
- 특정 헤더 기반 허용
- 도메인 기반 트래픽 제한

이는 서비스 간 보안 및 통제에 매우 유용하며,  
**Zero Trust 네트워크 모델**을 구현하는 데도 중요한 요소입니다.

---

## ✅ 참고 링크

- [Calico 공식 문서 (L7 Policy)](https://docs.tigera.io/security/app-layer-policy/)
- [Calico OSS GitHub](https://github.com/projectcalico/calico)
- [Calico Enterprise Features](https://www.tigera.io/)