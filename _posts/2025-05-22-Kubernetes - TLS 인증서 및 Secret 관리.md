---
layout: post
title: Kubernetes - TLS 인증서 및 Secret 관리
date: 2025-05-22 21:20:23 +0900
category: Kubernetes
---
# Kubernetes에서 TLS 인증서 및 Secret 관리

## 배경 지식: X.509, SAN, 체인, SNI

TLS(Transport Layer Security)는 Kubernetes 환경에서 서비스 간 통신의 기밀성과 무결성을 보장하는 핵심 기술입니다. 효과적인 TLS 관리를 이해하기 위해 몇 가지 기본 개념을 살펴보겠습니다.

- TLS 서버 인증서는 **X.509** 형식의 공개키 인증서로 표준화되어 있습니다.
- **SAN(Subject Alternative Name)** 필드에 실제 사용되는 호스트명이 포함되어야 브라우저와 클라이언트가 인증서를 신뢰할 수 있습니다.
- **인증서 체인(leaf → intermediate → root)** 이 완전해야 정상적인 신뢰 관계가 형성됩니다. Ingress 컨트롤러는 일반적으로 `tls.crt`에 **leaf 인증서와 intermediate 인증서**를 연결(concatenate)하여 구성하는 것을 권장합니다.
- TLS 핸드셰이크 과정에서 **SNI(Server Name Indication)** 확장을 통해 호스트별로 적절한 인증서를 선택합니다. 이를 통해 하나의 Ingress에 여러 호스트와 인증서를 구성할 수 있습니다.

---

## Secret의 핵심: `type: kubernetes.io/tls`

### TLS Secret 구성 스펙

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: <BASE64(leaf+intermediate...)>
  tls.key: <BASE64(PEM private key)>
```

- `tls.crt`: **PEM** 형식의 인증서입니다. 가능하면 **중간 CA 인증서**까지 포함하여 구성하세요(leaf 인증서 뒤에 이어서 붙임).
- `tls.key`: **PEM** 형식의 비공개키입니다. 보안을 위해 절대로 유출되어서는 안 됩니다.
- Secret의 데이터 필드는 **base64로 인코딩**되어 저장됩니다.

### 인증서 발급 및 Secret 생성

```bash
# 키 및 인증서 생성 (SAN 포함, 1년 유효기간)

cat > san.cnf <<'EOF'
[req]
distinguished_name=req
x509_extensions = v3_req
prompt = no
[req_distinguished_name]
CN = example.com
[v3_req]
subjectAltName = @alt_names
[alt_names]
DNS.1 = example.com
DNS.2 = www.example.com
EOF

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -config san.cnf
```

```bash
# TLS Secret 생성

kubectl create secret tls my-tls-secret \
  --cert=tls.crt --key=tls.key -n default
```

> 운영 환경에서는 **신뢰할 수 있는 공인 또는 내부 CA**가 서명한 인증서를 사용하는 것을 권장합니다.

---

## Ingress와 TLS 연동 (NGINX/Traefik/ALB, Gateway API)

### NGINX Ingress Controller 구성 예제

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts: ["example.com"]
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

### Traefik 구성 예제

```yaml
metadata:
  annotations:
    traefik.ingress.kubernetes.io/router.tls: "true"
```

### AWS ALB Ingress Controller (AWS Load Balancer Controller)

- 일반적으로 **ACM 인증서 ARN**을 Ingress annotation으로 지정합니다(Secret 대신).

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:123456789012:certificate/xxxx
```

### Gateway API (HTTPRoute + TLS)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gw
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: example.com
    tls:
      certificateRefs:
      - kind: Secret
        name: my-tls-secret
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: web-gw
  hostnames: ["example.com"]
  rules:
  - backendRefs:
    - name: my-service
      port: 80
```

---

## Secret 운영 관리 — 확인/복호화/교체/갱신

### Secret 정보 확인 및 복호화

```bash
kubectl get secret my-tls-secret -n default -o yaml
kubectl get secret my-tls-secret -n default -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
kubectl get secret my-tls-secret -n default -o jsonpath='{.data.tls\.key}' | base64 -d | openssl pkey -noout -text
```

### 무중단 인증서 교체(rollover)

1) 새 인증서와 키로 **동일한 이름**의 Secret을 업데이트하면 Ingress 컨트롤러가 **자동으로 리로드**합니다(컨트롤러마다 반영 타이밍이 다를 수 있음).
2) 더 안전한 방법으로는 **새로운 Secret 이름**으로 생성한 후, Ingress의 `secretName`을 **원자적으로 변경**하고 롤백이 가능하도록 관리하는 것입니다.

---

## cert-manager를 통한 **자동 인증서 발급 및 갱신** (ACME / 내부 CA)

`cert-manager`는 `Issuer`, `ClusterIssuer`, `Certificate` CRD(Custom Resource Definition)를 활용하여 인증서의 라이프사이클을 자동으로 관리합니다.

### 설치 (Helm 사용)

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true
```

설치 확인:

```bash
kubectl get pods -n cert-manager
```

### — HTTP-01(간편) / DNS-01(와일드카드 지원)

#### ClusterIssuer 구성 (Let's Encrypt 프로덕션 환경)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: admin@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

- HTTP-01: `/.well-known/acme-challenge/*` 경로를 Ingress가 처리할 수 있는 환경에서 간편하게 사용 가능합니다.
- 도메인의 DNS 설정이 Ingress의 외부 주소로 **정확히 향하도록** 구성되어 있어야 합니다.

#### DNS-01 (와일드카드 인증서 `*.example.com` 필요 시)

AWS Route53 예시:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: le-dns01
spec:
  acme:
    email: admin@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: le-dns01-key
    solvers:
    - dns01:
        route53:
          region: ap-northeast-2
          hostedZoneID: Z1234567890ABCDE
          accessKeyID: <AWS_ACCESS_KEY_ID>
          secretAccessKeySecretRef:
            name: route53-secret
            key: secret-access-key
```

> DNS-01 방식은 DNS 제공자(CloudDNS/Route53/Cloudflare 등)의 자격증명이 필요합니다.
> **와일드카드 인증서**가 반드시 필요한 경우에 선택하는 방식입니다.

#### Certificate 리소스를 통한 실제 발급

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: site-cert
  namespace: default
spec:
  secretName: mysite-tls
  issuerRef:
    name: letsencrypt-prod   # 또는 le-dns01
    kind: ClusterIssuer
  commonName: example.com
  dnsNames:
  - example.com
  - www.example.com
  renewBefore: 720h          # 만료 30일 전(기본값) 조정 가능
```

`mysite-tls` Secret이 자동으로 생성되고, 만료 기간 전에 자동으로 갱신됩니다.

##### Ingress와의 통합(자동 처리)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: site
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts: ["example.com","www.example.com"]
    secretName: mysite-tls
  rules:
  - host: example.com
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

> cert-manager는 Ingress를 감지하여 `Certificate` 리소스를 생성하고, 도메인 검증, 발급, 갱신까지 전체 과정을 자동으로 처리합니다.

### 내부 CA 및 Vault 통합

- 내부 CA의 루트/중간 CA를 사용하여 서명하는 Issuer 구성 (예: `Issuer` + `ca` 설정).
- 또는 `Vault Issuer` 플러그인을 통해 HashiCorp Vault의 PKI 엔진을 활용할 수 있습니다.

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: corp-ca
  namespace: security
spec:
  ca:
    secretName: corp-root-ca   # ca.crt / tls.key 보관(보안 주의!)
```

---

## 클라이언트 인증서 요구사항 구성 (서버 측)

### Ingress NGINX를 통한 클라이언트 인증서 검증

서버(Ingress)가 **클라이언트 인증서**를 검증하도록 구성할 수 있습니다. 이 경우 신뢰하는 CA 번들을 Secret으로 제공해야 합니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: client-ca
  namespace: default
type: Opaque
data:
  ca.crt: <BASE64(CA bundle)>
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mtls-app
  annotations:
    nginx.ingress.kubernetes.io/auth-tls-secret: "default/client-ca"
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-verify-depth: "2"
spec:
  tls:
  - hosts: ["secure.example.com"]
    secretName: my-tls-secret     # 서버 인증서
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app
            port:
              number: 443
```

- 클라이언트는 `--cert client.crt --key client.key` 옵션을 사용하여 접속해야 합니다.
- Istio/Linkerd 같은 서비스 메시를 사용하는 경우 **DestinationRule/PeerAuthentication** 등의 **메시 정책**을 통해 mTLS를 활성화하는 방식이 일반적입니다.

---

## 운영 보안: RBAC, etcd 암호화, 외부 비밀관리, GitOps

### RBAC — Secret 접근 최소화

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["my-tls-secret"]  # 특정 Secret만 접근 허용
  verbs: ["get"]                     # list/watch 권한은 제한 권장
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: bind-secret-reader
  namespace: default
subjects:
- kind: ServiceAccount
  name: web-sa
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

> 애플리케이션이 정말로 Secret을 **읽어야만 하는지** 검토하세요. 가능하면 **Ingress나 Gateway**가 TLS를 처리하고, 애플리케이션은 평문 HTTP로 내부 통신하는 구조를 고려해 보십시오.

### etcd 암호화(Encryption at Rest)

API 서버 설정 예시 (kubeadm 환경):

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <BASE64(32 bytes)>
  - identity: {}
```

`kube-apiserver` 인자에 `--encryption-provider-config=/etc/kubernetes/encryption-config.yaml` 추가.
**키 회전** 전략(새 키 추가 → 리인코딩 → 구 키 제거) 수립이 필수적입니다.

### 외부 비밀관리 및 GitOps 통합

- **Sealed Secrets**: 개발자가 공개 리포지토리에 암호화된 상태로 올려도 복호화는 클러스터에서만 가능합니다.
- **SOPS(+KMS)**: Git에 암호화된 Secret을 저장하고, 배포 시 컨트롤러나 파이프라인에서 복호화합니다.
- **External Secrets Operator(ESO)**: AWS Secrets Manager, GCP Secret Manager, Vault 등에서 Secret을 **자동으로 동기화**합니다.

ESO 구성 예시:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-tls-es
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: prod-secrets
    kind: ClusterSecretStore
  target:
    name: my-tls-secret
    creationPolicy: Owner
  data:
  - secretKey: tls.crt
    remoteRef:
      key: prod/web/tls.crt
  - secretKey: tls.key
    remoteRef:
      key: prod/web/tls.key
```

- **CSI Secrets Store**: Pod에 Secret을 **파일로 마운트**(메모리 기반)하고, Kubernetes Secret 생성 및 갱신 옵션을 지원합니다.

---

## 운영 모범 사례

1. **도메인별 SAN 명시**: `example.com`, `www.example.com` 등 실제 사용되는 모든 호스트를 명확히 나열하세요.
2. **인증서 체인 완전성**: `tls.crt`에 **중간 CA 인증서를 포함**하세요.
3. **자동 갱신 모니터링**: cert-manager의 `Certificate` 상태와 만료 임계값(기본 30일 전)에 대한 알림을 구성하세요.
4. **ACME 스테이징 환경 활용**: Let's Encrypt **스테이징** 환경으로 테스트하여 레이트 리미트를 회피한 후 프로덕션으로 전환하세요.
5. **DNS 전파 및 캐시 고려**: DNS-01 방식은 TTL과 전파 시간이 성공 여부에 영향을 미칩니다.
6. **Secret 접근 범위 최소화**: 네임스페이스 분리와 RBAC 최소 권한 원칙을 준수하세요.
7. **GitOps와 암호화 통합**: Sealed Secrets/SOPS/ESO/CSI를 통해 **리포지토리 유출 위험**을 낮추세요.
8. **정기적인 키 회전**: 일정 주기로 새 키로 교체(Secret 교체 후 롤링 업데이트)하세요.
9. **체계적인 관측성**: `curl -vkI`, `openssl s_client`, Ingress 컨트롤러 로그, cert-manager 이벤트를 통해 **문제 원인을 신속히 파악**하세요.

---

## 문제 해결 시나리오 모음

### 브라우저에서 "인증서를 신뢰할 수 없음" 오류

- 인증서 체인이 누락되었을 가능성이 높습니다. `tls.crt`에 **leaf와 intermediate 인증서가 모두 포함**되었는지 확인하세요.
- 사설 CA를 사용하는 경우, **루트/중간 CA 인증서**를 클라이언트의 신뢰 저장소에 배포해야 합니다.

### `404 Not Found` 또는 HTTP로만 응답

- Ingress가 올바른 **host** 규칙을 가지고 있는지, `ingressClassName`과 annotations을 확인하세요.
- DNS가 올바른 로드 밸런서 주소를 가리키는지 `dig example.com` 명령으로 확인하세요.

### cert-manager 인증서 발급 실패

```bash
kubectl describe certificate site-cert -n default
kubectl describe certificaterequest -n default
kubectl describe order -n default
kubectl describe challenge -n default
kubectl logs -n cert-manager deploy/cert-manager
```

- HTTP-01: `/.well-known/acme-challenge/*` 경로가 외부에서 접근 가능한지 `curl -vkL http://example.com/.well-known/acme-challenge/TEST`로 확인하세요.
- DNS-01: TXT 레코드가 정확히 생성되고 전파되었는지 확인하세요.

### 만료 임박인데 자동 갱신이 안 되는 경우

- `Certificate` 리소스의 `renewBefore` 설정을 확인하세요(기본값 720시간).
- cert-manager 컨트롤러 로그에 오류가 없는지 확인하세요.
- 권한 및 솔버 설정(Route53 권한 등)을 재점검하세요.

### mTLS 연결 거부

- 클라이언트가 **올바른 인증서와 키**로 접속하는지 확인하세요.
- Ingress annotation의 `auth-tls-secret` CA 번들 정확성과 verify-depth 설정을 조정하세요.

---

## 예제 모음: 운영 환경에서 바로 사용 가능한 템플릿

### Let's Encrypt 스테이징 + 자동 TLS Ingress

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: admin@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: le-staging-key
    solvers:
    - http01:
        ingress:
          class: nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts: ["stage.example.com"]
    secretName: stage-tls
  rules:
  - host: stage.example.com
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

### 내부 CA와 mTLS 혼합 구성

- 서버 인증서는 `corp-ca` Issuer로 발급.
- 클라이언트 CA 번들은 별도 Secret으로 관리.

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: corp-ca
  namespace: default
spec:
  ca:
    secretName: corp-root-ca
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-api-cert
  namespace: default
spec:
  secretName: internal-api-tls
  issuerRef:
    name: corp-ca
    kind: Issuer
  dnsNames: ["internal-api.svc.cluster.local"]
---
apiVersion: v1
kind: Secret
metadata:
  name: mtlsca
  namespace: default
type: Opaque
data:
  ca.crt: <BASE64(CLIENT-CA-BUNDLE)>
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: internal-api
  annotations:
    nginx.ingress.kubernetes.io/auth-tls-secret: "default/mtlsca"
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
spec:
  tls:
  - hosts: ["internal.example.com"]
    secretName: internal-api-tls
  rules:
  - host: internal.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 8443
```

---

## 유용한 검증 명령어 레퍼런스

```bash
# 인증서 상세 정보 확인

kubectl get secret mysite-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout

# 원격 서버의 인증서 체인 확인

openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null 2>/dev/null | openssl x509 -noout -issuer -subject -dates

# HTTP-01 챌린지 경로 접근 테스트

curl -vkL http://example.com/.well-known/acme-challenge/TEST

# cert-manager 리소스 상태 확인

kubectl get certificate,certificaterequest,order,challenge -A
kubectl describe certificate <name> -n <ns>
```

---

## 결론

Kubernetes 환경에서 효과적인 TLS 인증서 및 Secret 관리는 시스템의 신뢰성, 보안성, 운영 효율성을 결정하는 중요한 요소입니다.

| 주제 | 핵심 포인트 |
|------|-----------|
| Secret/TLS 기본 | `kubernetes.io/tls` 타입 Secret에 `tls.crt`(체인 포함)와 `tls.key` 저장 |
| Ingress/Gateway 연동 | `spec.tls[].secretName`을 통해 HTTPS 활성화 |
| 자동화 도구 | cert-manager의 Issuer/Certificate 리소스를 활용한 발급·갱신 **코드 기반 관리** |
| 와일드카드/다중 도메인 | **DNS-01** 방식 선호, 공급자 자격증명 및 권한 필요 |
| 보안 운영 | RBAC 최소 권한, etcd 암호화, 외부 비밀관리(ESO/Vault/SOPS/Sealed) 통합 |
| 관측성 및 대응 | `describe`/`logs`/`openssl`/`curl` 도구를 통한 **신속한 문제 진단** |

TLS 관리는 단순한 기술적 구현을 넘어 조직의 **보안 문화와 운영 체계**를 반영합니다. 수동 인증서 관리부터 cert-manager를 통한 자동화, mTLS 구성, 다양한 비밀 관리 솔루션 통합까지, 본 문서에서 소개한 패턴과 모범 사례를 참고하여 **안전하고 자동화된 HTTPS 인프라**를 구축하시기 바랍니다. 지속적인 모니터링, 정기적인 보안 검토, 체계적인 키 관리 프로세스를 통해 프로덕션 환경의 보안과 안정성을 극대화할 수 있습니다.