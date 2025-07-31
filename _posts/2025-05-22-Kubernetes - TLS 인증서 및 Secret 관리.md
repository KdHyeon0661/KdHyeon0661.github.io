---
layout: post
title: Kubernetes - TLS 인증서 및 Secret 관리
date: 2025-05-22 21:20:23 +0900
category: Kubernetes
---
# Kubernetes에서 TLS 인증서 및 Secret 관리

HTTPS 통신은 보안의 핵심입니다. Kubernetes에서는 TLS 인증서를 `Secret`으로 관리하고,  
Ingress 혹은 애플리케이션과 연동하여 **보안 연결**을 제공합니다.

이 글에서는 TLS 인증서와 Secret의 개념부터,  
**직접 생성하는 방법**, **자동화(Cert-Manager)**까지 자세히 알아봅니다.

---

## ✅ Secret이란?

Kubernetes의 Secret은 **비밀번호, 토큰, 인증서 등의 민감한 정보를 저장하는 객체**입니다.

### Secret의 종류

| 타입 | 설명 |
|------|------|
| Opaque | 일반적인 Key-Value 비밀값 |
| kubernetes.io/dockerconfigjson | Docker 이미지 pull용 인증정보 |
| kubernetes.io/tls | TLS 인증서 + 키 전용 타입 |

---

## ✅ TLS Secret 구성 예시

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: BASE64_ENCODED_CERT
  tls.key: BASE64_ENCODED_KEY
```

→ `tls.crt`와 `tls.key`는 각각 **인증서**와 **개인키**를 base64 인코딩한 값입니다.

---

## ✅ 인증서 생성 및 Secret 생성

### 1. OpenSSL로 인증서 + 키 생성

```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=example.com/O=example"
```

### 2. Kubernetes Secret으로 등록

```bash
kubectl create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  -n default
```

→ 위 명령은 자동으로 `kubernetes.io/tls` 타입으로 생성

---

## ✅ Ingress와 TLS 연동하기

Ingress는 Secret으로 저장된 인증서를 참조하여 HTTPS를 제공합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - example.com
    secretName: my-tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

→ `https://example.com` 접속 시 TLS 암호화 적용

---

## ✅ Secret 내용 확인 및 복호화

### Secret 확인

```bash
kubectl get secret my-tls-secret -o yaml
```

### base64 복호화

```bash
echo "BASE64_ENCODED_CERT" | base64 -d
```

---

## ✅ Secret 보안 팁

| 항목 | 설명 |
|------|------|
| 권한 제한 | Secret에 접근 가능한 사용자/ServiceAccount를 최소화 |
| 저장소 암호화 | etcd에 저장된 Secret 암호화 (Encryption at Rest) |
| 감사 로그 설정 | Secret 접근 이력 확인을 위한 Audit 설정 |
| 외부 Vault 연동 | HashiCorp Vault, AWS Secrets Manager와 연동 가능 |

---

## ✅ TLS 인증서 자동화: Cert-Manager

[`cert-manager`](https://cert-manager.io)는 Let's Encrypt 또는 내부 CA와 연동하여  
**자동으로 TLS 인증서를 생성, 관리, 갱신**하는 컨트롤러입니다.

### 1. cert-manager 설치 (Helm)

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

---

### 2. ClusterIssuer 생성 (Let’s Encrypt)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

---

### 3. Ingress에서 자동 TLS 사용

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - mysite.com
    secretName: mysite-tls
  rules:
  - host: mysite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

→ cert-manager가 인증서를 자동 생성하고 `mysite-tls` Secret에 저장  
→ 만료 전 자동 갱신까지 처리됨

---

## ✅ 기타 TLS 사용 사례

| 사용처 | 설명 |
|--------|------|
| Ingress TLS | 외부 HTTPS 트래픽 암호화 |
| gRPC 통신 | 내부 서비스 간 mTLS 통신 (Sidecar 포함) |
| Kafka / RabbitMQ | TLS 기반 인증 및 암호화 |
| 웹앱 환경변수 | Secret에서 TLS cert를 마운트해 앱 내 사용 |

---

## ✅ 결론

| 항목 | 설명 |
|------|------|
| Secret | 민감정보 저장용 리소스 (`type: kubernetes.io/tls`) |
| 인증서 생성 | OpenSSL 또는 cert-manager 활용 |
| 연동 | Ingress → TLS Secret → HTTPS 적용 |
| 자동화 | cert-manager로 발급 및 갱신 관리 |
| 보안 팁 | 접근 제한, 저장 암호화, Vault 연동 고려 |

> Kubernetes에서 TLS 인증서를 안전하게 관리하고 자동화하는 것은  
> 서비스의 보안과 신뢰성을 높이는 핵심 작업입니다.

---

## ✅ 참고 링크

- [Kubernetes TLS Secret 공식 문서](https://kubernetes.io/docs/concepts/configuration/secret/#tls-secrets)
- [cert-manager 공식 사이트](https://cert-manager.io/)
- [Let's Encrypt API](https://letsencrypt.org/docs/acme-protocol/)