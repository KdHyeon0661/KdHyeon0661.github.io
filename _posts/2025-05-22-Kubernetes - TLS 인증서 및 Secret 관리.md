---
layout: post
title: Kubernetes - TLS 인증서 및 Secret 관리
date: 2025-05-22 21:20:23 +0900
category: Kubernetes
---
# Kubernetes에서 TLS 인증서 및 Secret 관리

## 0. 배경 지식: X.509, SAN, 체인, SNI

- TLS 서버 인증서는 **X.509** 형식의 공개키 인증서입니다.
- **SAN(Subject Alternative Name)** 에 호스트명이 포함되어야 브라우저/클라이언트가 신뢰합니다.
- **체인(leaf → intermediate → root)** 이 완전해야 신뢰됩니다. Ingress 컨트롤러는 `tls.crt`에 **leaf + intermediates** 를 연결(concatenate)해 두는 걸 권장합니다.
- TLS 핸드셰이크에서 **SNI(Server Name Indication)** 로 호스트별 인증서를 선택합니다(하나의 Ingress에 여러 호스트/인증서).

---

## 1. Secret의 핵심: `type: kubernetes.io/tls`

### 1.1 TLS Secret 스펙

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

- `tls.crt`: **PEM** 포맷 인증서. 가능하면 **중간 CA**까지 붙여 넣으세요(leaf 뒤에 이어붙임).
- `tls.key`: **PEM** 포맷 비공개키. 절대로 유출 금지.
- Secret 데이터는 **base64 인코딩**입니다.

### 1.2 OpenSSL로 self-signed(테스트용) 발급 → Secret 생성

```bash
# 1. 키/인증서 생성 (SAN 포함, 1년)
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
# 2. Secret 생성
kubectl create secret tls my-tls-secret \
  --cert=tls.crt --key=tls.key -n default
```

> 운영 환경에서는 **신뢰 가능한 공인/내부 CA**가 서명한 인증서를 사용하세요.

---

## 2. Ingress와 TLS 연동 (NGINX/Traefik/ALB, Gateway API까지)

### 2.1 NGINX Ingress Controller 예제

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
  - hosts: [ "example.com" ]
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

### 2.2 Traefik 예제

```yaml
metadata:
  annotations:
    traefik.ingress.kubernetes.io/router.tls: "true"
```

### 2.3 AWS ALB Ingress Controller (AWS Load Balancer Controller)

- 보통 **ACM 인증서 ARN**을 Ingress annotation으로 지정합니다(Secret 대신).

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:123456789012:certificate/xxxx
```

### 2.4 Gateway API (HTTPRoute + TLS)

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

## 3. Secret 운용 — 확인/복호화/교체/회전

### 3.1 확인/복호화

```bash
kubectl get secret my-tls-secret -n default -o yaml
kubectl get secret my-tls-secret -n default -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
kubectl get secret my-tls-secret -n default -o jsonpath='{.data.tls\.key}' | base64 -d | openssl pkey -noout -text
```

### 3.2 무중단 교체(rollover)

1) 새 인증서/키로 **동일 이름** Secret 업데이트 → Ingress 컨트롤러가 **자동 리로드**(컨트롤러마다 반영 타이밍 상이).  
2) 안전하게 하려면 **새 Secret 이름**으로 만들고 Ingress의 `secretName`을 **원자적 변경** 후 롤백 가능하도록 관리.

---

## 4. cert-manager로 **자동 발급/갱신** (ACME / 내부 CA)

`cert-manager`는 `Issuer/ClusterIssuer/Certificate` CRD를 이용해 인증서 라이프사이클을 관리합니다.

### 4.1 설치 (Helm)

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true
```

확인:

```bash
kubectl get pods -n cert-manager
```

### 4.2 ACME(렌츠/공인CA) — HTTP-01(간편) / DNS-01(와일드카드)

#### (A) ClusterIssuer (Let’s Encrypt: 프로덕션)

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

- HTTP-01: `/.well-known/acme-challenge/*` 경로를 Ingress가 처리 가능한 환경에서 간편.
- 도메인 DNS가 Ingress의 외부 주소로 **정확히** 향해야 합니다.

#### (B) DNS-01 (와일드카드 `*.example.com` 필요 시)

Route53 예시:

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

> DNS-01은 DNS 제공자(CloudDNS/Route53/Cloudflare 등) 자격증명이 필요.  
> **와일드카드** 인증서가 반드시 필요할 때 선택.

#### (C) Certificate 리소스로 실제 발급

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

`mysite-tls` Secret이 자동 생성되고, 만료 전에 자동 갱신됩니다.

##### Ingress와 통합(자동)

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
        backend: { service: { name: web, port: { number: 80 } } }
```

> cert-manager가 Ingress를 감지해 `Certificate`를 생성하고, 검증/발급/갱신까지 자동 처리.

### 4.3 내부 CA(사설 PKI) 연동

- 사내 CA의 루트/중간 CA로 서명하는 Issuer 구성(예: `Issuer` + `ca` 설정).
- 또는 `Vault Issuer` 플러그인으로 HashiCorp Vault의 PKI 엔진 사용.

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: corp-ca
  namespace: security
spec:
  ca:
    secretName: corp-root-ca   # ca.crt / tls.key 보관(보안!)
```

---

## 5. mTLS(서버/클라이언트 양방향 인증) — Ingress NGINX 예제

### 5.1 클라이언트 인증서 요구(서버 입장)

- 서버(Ingress)가 **클라이언트 인증서**를 검사.
- 신뢰하는 CA 번들을 Secret로 제공.

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
  - hosts: [ "secure.example.com" ]
    secretName: my-tls-secret     # 서버 인증서
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: app, port: { number: 443 } } }
```

- 클라이언트는 `--cert client.crt --key client.key`로 접속해야 성공.
- Istio/Linkerd 사용 시에는 **DestinationRule/PeerAuthentication** 등 **메쉬 정책**으로 mTLS를 켜는 편이 일반적.

---

## 6. 운영 보안: RBAC, etcd 암호화, 외부 비밀관리, GitOps

### 6.1 RBAC — Secret 접근 최소화

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["my-tls-secret"]  # 특정 Secret만
  verbs: ["get"]                     # list/watch 금지 권장
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

> 앱이 정말로 Secret을 **읽어야만** 하는지 점검하세요. 가능하면 **Ingress/Gateway**가 TLS를 처리하고, 앱은 평문 HTTP로 내부 통신.

### 6.2 etcd 암호화(Encryption at Rest)

API 서버 설정(예: kubeadm 환경):

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

`kube-apiserver` 인자에 `--encryption-provider-config=/etc/kubernetes/encryption-config.yaml`.  
**키 회전**(새 key 추가 → 리인코드 → 구 key 제거) 전략 수립 필수.

### 6.3 외부 비밀관리 / GitOps

- **Sealed Secrets**: 개발자가 공개 리포에 올려도 복호화는 클러스터에서만 가능.
- **SOPS(+KMS)**: Git에 암호화된 Secret 저장, 배포 시 컨트롤러/파이프라인에서 복호화.
- **External Secrets Operator(ESO)**: AWS Secrets Manager, GCP Secret Manager, Vault 등에서 Secret을 **자동 동기화**.

ESO 예:

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
    remoteRef: { key: prod/web/tls.crt }
  - secretKey: tls.key
    remoteRef: { key: prod/web/tls.key }
```

- **CSI Secrets Store**: Pod에 Secret을 **파일로 마운트**(메모리 기반)하고, K8s Secret 생성/갱신 옵션 지원.

---

## 7. 운영 베스트 프랙티스

1. **도메인 별로 SAN 확실히**: `example.com`, `www.example.com` 등 실제 사용 호스트 모두 나열.  
2. **체인 완전성**: `tls.crt`에 **intermediate 포함**.  
3. **자동갱신 모니터링**: cert-manager의 `Certificate` 상태, 만료 임계(기본 30일 전) 알람.  
4. **스테이징 ACME로 검증**: Let’s Encrypt **스테이징**으로 레이트리밋 회피 후 프로덕션 전환.  
5. **DNS 전파/캐시 고려**: DNS-01은 TTL/전파시간이 관건.  
6. **Secret 스코프 최소화**: 네임스페이스 분리, RBAC 최소권한.  
7. **GitOps + 암호화**: Sealed Secrets/SOPS로 **리포 유출 위험** 낮추기.  
8. **키 회전**: 정기적으로 새 키로 교체(Secret 교체 후 롤링).  
9. **관측성**: `curl -vkI`, `openssl s_client`, Ingress 컨트롤러 로그, cert-manager events로 **원인 가시화**.

---

## 8. 트러블슈팅 시나리오 모음

### 8.1 브라우저 “인증서 신뢰 안 됨”

- 체인 누락 가능성↑. `tls.crt`에 **leaf + intermediate** 붙였는지 확인.  
- 사설 CA 사용 시, **루트/중간 CA** 를 클라이언트 신뢰 저장소에 배포.

### 8.2 `404 Not Found` 혹은 HTTP로만 응답

- Ingress가 올바른 **host** 룰을 가졌는지, `ingressClassName`/annotations 확인.  
- DNS가 올바른 LB 주소를 가리키는지 `dig example.com` 확인.

### 8.3 cert-manager 발급 실패

```bash
kubectl describe certificate site-cert -n default
kubectl describe certificaterequest -n default
kubectl describe order -n default
kubectl describe challenge -n default
kubectl logs -n cert-manager deploy/cert-manager
```

- HTTP-01: `/.well-known/acme-challenge/*`가 외부에서 접근 가능한지 `curl -vkL http://example.com/.well-known/acme-challenge/TEST`  
- DNS-01: TXT 레코드가 정확히 생성/전파됐는지 확인.

### 8.4 만료 임박인데 갱신 안 됨

- `Certificate`의 `renewBefore` 확인(기본 720h).  
- cert-manager 컨트롤러 로그에 에러가 없는지 확인.  
- 퍼미션/솔버 설정(Route53 권한 등) 재점검.

### 8.5 mTLS 연결 거부

- 클라이언트가 **올바른 cert/key** 로 접속하는지.  
- Ingress annotation의 `auth-tls-secret` CA 번들 정확성, verify-depth 조정.

---

## 9. 예제 모음: “운영에서 바로 쓰는” 템플릿

### 9.1 Let’s Encrypt 스테이징 + 자동 TLS Ingress

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: admin@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef: { name: le-staging-key }
    solvers:
    - http01:
        ingress: { class: nginx }
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
        backend: { service: { name: web, port: { number: 80 } } }
```

### 9.2 내부 CA + mTLS(서버/클라이언트) 혼합

- 서버 인증서는 `corp-ca` Issuer로 발급.  
- 클라이언트 CA 번들은 별도 Secret.

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
  issuerRef: { name: corp-ca, kind: Issuer }
  dnsNames: [ "internal-api.svc.cluster.local" ]
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
        backend: { service: { name: api, port: { number: 8443 } } }
```

---

## 10. 간단한 검증 명령 레퍼런스

```bash
# 인증서 정보
kubectl get secret mysite-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout

# 원격 서버의 체인 확인
openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null 2>/dev/null | openssl x509 -noout -issuer -subject -dates

# HTTP-01 챌린지 경로 확인
curl -vkL http://example.com/.well-known/acme-challenge/TEST

# cert-manager 리소스 상태
kubectl get certificate,certificaterequest,order,challenge -A
kubectl describe certificate <name> -n <ns>
```

---

## 11. 체크리스트 요약

- [ ] Ingress/Gateway에 **올바른 Secret 참조**(체인 포함).  
- [ ] cert-manager: Issuer/ClusterIssuer/Certificate **상태 녹색**.  
- [ ] ACME 스테이징 → 프로덕션 **단계적 전환**.  
- [ ] 만료 알람/`renewBefore`/대시보드 구성.  
- [ ] RBAC 최소권한, Secret **네임스페이스 경계** 준수.  
- [ ] **etcd 암호화** 활성화 + **키 회전** 절차.  
- [ ] GitOps 시 **Sealed Secrets/SOPS/ESO/CSI** 고려.  
- [ ] mTLS 필요 시 CA 번들 관리 및 깊이(verify-depth) 조정.  
- [ ] 정기 보안점검: 누가 어떤 Secret에 접근하는지 **감사**.

---

## 결론

| 주제 | 핵심 요점 |
|------|-----------|
| Secret/TLS 기본 | `kubernetes.io/tls`에 `tls.crt`(체인 포함)/`tls.key` 저장 |
| Ingress/Gateway 연동 | `spec.tls[].secretName`로 HTTPS 활성화 |
| 자동화 | cert-manager의 Issuer/Certificate로 발급·갱신을 **코드로** 관리 |
| 와일드카드/다중 도메인 | **DNS-01** 선호, 프로바이더 자격/권한 필요 |
| 보안 운영 | RBAC 최소권한, etcd 암호화, 외부 비밀관리(ESO/Vault/SOPS/Sealed) |
| 가시성/대응 | `describe`/`logs`/`openssl`/`curl`로 **빠른 원인 파악** |

Kubernetes에서 TLS를 제대로 다루는 것은 **신뢰·보안·운영 효율**의 교차점입니다.  
수동 발급부터 cert-manager 자동화, mTLS, 비밀관리 보안까지 본문 레시피를 참고해 **안전하고 자동화된 HTTPS 스택**을 구축하세요.

---

## 참고
- Kubernetes TLS Secret: <https://kubernetes.io/docs/concepts/configuration/secret/#tls-secrets>  
- cert-manager: <https://cert-manager.io/>  
- Let’s Encrypt ACME: <https://letsencrypt.org/docs/acme-protocol/>  
- External Secrets Operator: <https://external-secrets.io/>  
- Sealed Secrets: <https://github.com/bitnami-labs/sealed-secrets>  
- SOPS: <https://github.com/getsops/sops>  
- CSI Secrets Store: <https://secrets-store-csi-driver.sigs.k8s.io/>