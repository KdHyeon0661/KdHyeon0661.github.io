---
layout: post
title: Docker - 개인용 Docker Registry 설정
date: 2025-03-21 20:20:23 +0900
category: Docker
---
# 개인용 Docker Registry 설정

## 왜 개인 Registry가 필요한가?

| 필요성 | 설명 |
|---|---|
| 속도 | 사내 네트워크나 폐쇄망에서 이미지 **풀(pull) 지연을 최소화** |
| 보안 | 민감한 이미지, 모델, 자산의 **외부 유출 방지** |
| 유연성 | **프로젝트별 네임스페이스**, 태그 규칙, 정책 자유롭게 적용 |
| 오프라인 | 인터넷이 차단된 환경(산업단지, 관제센터 등)에서 필수 인프라 |
| 비용 | 외부 레지스트리의 요율 제한(Rate Limit) 및 전송 비용 회피, **캐시**로 대역폭 절감 |

---

## 옵션 비교: `registry:2` vs Harbor

| 항목 | Docker Registry (registry:2) | Harbor |
|---|---|---|
| UI | 없음 (CLI 및 REST API만) | 통합 웹 UI 제공 |
| 인증/권한 | 기본 Basic Auth, 고급 기능은 프록시 앞단 필요 | 로컬/LDAP/OIDC 지원, **세분화된 RBAC(역할 기반 접근 제어)** |
| 서명/신뢰 | 없음 | Notary/Cosign 통합 아티팩트 서명 관리 |
| 보안 스캔 | 없음 | Trivy/Clair 통합 취약점 스캐너 |
| 프록시 캐시 | **지원**(pull-through cache) | **고급 Proxy Cache Project** 기능 |
| 스토리지 | 파일시스템, S3 등 **다양한 백엔드** | 로컬/오브젝트 스토리지, 외부 DB/Redis 지원 |
| 복제/미러 | 기본 기능 단순 | **다양한 복제 기능**(양방향, 스케줄, 필터) |
| 배포 | 단일 컨테이너 또는 Compose | Compose 또는 **Helm(Kubernetes)** |
| 용도 | 경량 프라이빗 저장소 | 팀 또는 조직 단위의 풀스택 레지스트리 솔루션 |

**선택 가이드**
- 개인 또는 소규모 팀, 간단하고 저비용 운영 → **registry:2**
- 팀 또는 조직 단위, 정책 관리, 감사, 보안 스캔, 서명, 복제 기능 필요 → **Harbor**

---

## Registry 아키텍처 요약

- **Manifests**: 이미지 메타데이터(레이어 목록, 플랫폼 정보). 동일 태그가 여러 아키텍처(amd64/arm64)를 위한 **매니페스트 리스트**를 가질 수 있습니다.
- **Layers(Blobs)**: 압축된 파일시스템 조각. **중복 제거**로 저장 공간을 절약합니다.
- **Name/Tag**: `<호스트>/<네임스페이스>/<리포지토리>:<태그>` 형식. 운영 시에는 **다이제스트(digest) 핀**(`@sha256:...`) 사용을 권장합니다.

---

## 빠른 시작

### 가장 단순한 실행

```bash
docker run -d --name registry -p 5000:5000 registry:2
```
- 기본 저장 경로는 컨테이너 내부의 `/var/lib/registry`입니다. 바인드 마운트나 볼륨을 사용하여 **데이터를 영속화**하는 것을 권장합니다.

```bash
# 영속 볼륨을 사용한 실행
docker run -d --name registry \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  registry:2
```

### 푸시/풀 테스트

```bash
docker pull ubuntu:20.04
docker tag ubuntu:20.04 localhost:5000/my-ubuntu
docker push localhost:5000/my-ubuntu
docker pull localhost:5000/my-ubuntu
```

> 기본 설정은 **HTTP**이므로 로컬 테스트 환경에서만 권장됩니다. 실제 서비스 운영 시에는 반드시 **TLS(HTTPS)** 를 사용해야 합니다.

---

## 구성

### 자체 서명 인증서 생성 (테스트용)

```bash
mkdir -p certs
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout certs/domain.key -x509 -days 365 -out certs/domain.crt \
  -subj "/CN=registry.example.com"
```

### TLS를 사용한 Registry 실행 (443 포트 바인딩)

```bash
docker run -d --name registry \
  -p 443:5000 \
  -v $(pwd)/certs:/certs:ro \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

### 클라이언트 측 인증서 신뢰 구성

```bash
# certs.d 디렉터리명은 "호스트:포트"와 정확히 일치해야 합니다.
sudo mkdir -p /etc/docker/certs.d/registry.example.com:443
sudo cp certs/domain.crt /etc/docker/certs.d/registry.example.com:443/ca.crt
```

> 사설 CA나 회사 내부 CA를 사용하는 경우, **전체 인증서 체인**을 `ca.crt` 파일에 포함시켜야 합니다.

---

## registry:2 기본 인증 설정

### htpasswd 파일 생성

```bash
mkdir -p auth
docker run --rm --entrypoint htpasswd registry:2 \
  -Bbn testuser testpassword > auth/htpasswd
```

### 인증이 활성화된 Registry 실행

```bash
docker run -d --name registry \
  -p 5000:5000 \
  -v $(pwd)/auth:/auth:ro \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2
```

### 레지스트리 로그인

```bash
docker login localhost:5000
```

> 더 고급 인증 요구사항(SSO, 세분화된 RBAC)이 있다면 Harbor를 고려하십시오.

---

## Compose를 이용한 운영형 배포

### 파일시스템 + Basic Auth + TLS 구성 예시

```yaml
version: "3.9"
services:
  registry:
    image: registry:2
    container_name: registry
    ports:
      - "443:5000"
    environment:
      REGISTRY_HTTP_ADDR: :5000
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key
      REGISTRY_STORAGE_DELETE_ENABLED: "true"     # blob 삭제 기능 활성화 (GC 준비)
    volumes:
      - ./data:/var/lib/registry
      - ./auth:/auth:ro
      - ./certs:/certs:ro
    restart: unless-stopped
```

### Pull-through 캐시 설정 (Docker Hub 프록시)

```yaml
environment:
  REGISTRY_PROXY_REMOTEURL: https://registry-1.docker.io
```
- 외부 이미지를 사내 네트워크에 **캐시**하여 **Rate Limit 및 대역폭을 절감**합니다.

### S3/MinIO 백엔드 구성 (대용량, 중복 제거)

```yaml
environment:
  REGISTRY_STORAGE: s3
  REGISTRY_STORAGE_S3_REGION: ap-northeast-2
  REGISTRY_STORAGE_S3_BUCKET: my-registry-bucket
  REGISTRY_STORAGE_S3_ACCESSKEY: ${S3_ACCESS}
  REGISTRY_STORAGE_S3_SECRETKEY: ${S3_SECRET}
  REGISTRY_STORAGE_S3_REGIONENDPOINT: http://minio:9000  # MinIO 예시
  REGISTRY_STORAGE_S3_ENCRYPT: "true"
  REGISTRY_STORAGE_S3_SECURE: "false"                   # http endpoint일 경우
  REGISTRY_STORAGE_S3_ROOTDIRECTORY: /registry
  REGISTRY_STORAGE_REDIRECT: "true"                     # 성능 향상 (오브젝트 직접 다운로드)
```

### 가비지 컬렉션(GC) 실행

- Registry는 **참조되지 않는 blob**을 정기적으로 정리해야 저장 공간을 회수할 수 있습니다.
- 안전한 실행 절차:
```bash
# 1. 푸시 및 삭제 작업 중단 (레지스트리 정지 권장)
docker stop registry

# 2. 오프라인 가비지 컬렉션 실행
docker run --rm \
  -v $(pwd)/data:/var/lib/registry \
  -v $(pwd)/config.yml:/etc/docker/registry/config.yml \
  registry:2 garbage-collect /etc/docker/registry/config.yml --delete-untagged=true

# 3. 레지스트리 재기동
docker start registry
```
- `config.yml` 파일은 실제 Registry 실행 시 사용한 설정과 동일해야 합니다 (스토리지 설정 등 포함).

---

## 앞단 구성 (선택 사항)

### 앞단 구성의 필요성

- **TLS 종료, 리다이렉트, 접근 제어, 레이트 리미팅** 등 고급 정책을 일관되게 처리할 수 있습니다.
- 여러 레지스트리나 서비스를 **도메인별로 라우팅**할 수 있습니다.

### 간단한 Nginx 설정 예시

```nginx
server {
  listen 443 ssl http2;
  server_name registry.example.com;

  ssl_certificate     /etc/nginx/certs/fullchain.pem;
  ssl_certificate_key /etc/nginx/certs/privkey.pem;

  # 대용량 이미지 푸시를 위한 설정
  client_max_body_size 1g;

  location / {
    proxy_pass http://registry:5000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 300;
  }
}
```

---

## Harbor 소개 및 설치

### 요구사항

- Docker & Docker Compose (또는 Kubernetes + Helm)
- 4GB 이상 RAM, FQDN(완전한 도메인 이름), TLS 인증서 (권장), 외부 DB/객체 스토리지 (선택 사항)

### Compose 기반 설치

```bash
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz
tar xvf harbor-online-installer-*.tgz
cd harbor

cp harbor.yml.tmpl harbor.yml
vi harbor.yml
# 주요 설정 항목:
# hostname: registry.example.com
# https:
#   port: 443
#   certificate: /path/to/cert.crt
#   private_key: /path/to/key.key
# 필요에 따라 external_database, external_storage, oidc, proxy-cache 등을 설정

sudo ./install.sh
```
- 접속 URL: `https://registry.example.com`
- 기본 관리자 계정: `admin` / 비밀번호: `Harbor12345` (설치 직후 **즉시 변경** 권장)

### Helm을 이용한 Kubernetes 배포

```bash
helm repo add harbor https://helm.goharbor.io
```
- `values.yaml` 파일에 Ingress, TLS, 스토리지, 외부 DB/Redis, Trivy, Notary, OIDC, RBAC 등의 설정을 한 후 배포합니다.
```bash
helm install harbor harbor/harbor -f values.yaml -n harbor --create-namespace
```

---

## Harbor 핵심 기능 운영 가이드

### 프로젝트 및 RBAC

- 프로젝트 단위로 **퍼블릭/프라이빗** 설정이 가능하며, **역할(관리자, 개발자, 게스트)** 을 부여할 수 있습니다.
- **Robot Account**를 발급하여 CI/CD 파이프라인이 비밀번호 대신 토큰으로 푸시/풀을 수행할 수 있습니다.

### Proxy Cache Project

- 외부 레지스트리(Docker Hub, GHCR)를 캐싱하는 전용 프로젝트를 생성할 수 있습니다.
- 개발 속도를 높이고, 외부 레지스트리의 Rate Limit을 회피하는 데 유용합니다.

### 보안 스캔

- **Trivy** 스캐너가 내장되어 있습니다 (오프라인 DB 동기화 가능).
- 프로젝트 정책을 설정하여, 푸시나 프로모션 시 **특정 취약점 심각도 기준**을 초과하면 차단할 수 있습니다.

### 서명 및 신뢰 (NOTARY / Cosign)

- Harbor는 **서명된 아티팩트**와 관련 정책을 관리합니다.
```bash
# cosign을 사용한 서명 예시 (Harbor 외부에서 서명 수행)
cosign generate-key-pair
cosign sign registry.example.com/demo/app:1.0.0
cosign verify registry.example.com/demo/app:1.0.0
```

### 리텐션 정책 / 쿼터 / 불변 태그

- **Retention Policy**: 오래된 태그나 특정 패턴의 태그를 스케줄에 따라 자동 삭제합니다.
- **Tag Immutability**: 특정 패턴(예: `v*`)의 태그 **덮어쓰기를 방지**합니다.
- **Project Quota**: 프로젝트별 스토리지 용량 또는 태그 개수를 제한합니다.

### 복제 (Replication)

- 원격 Harbor 또는 다른 Registry와 **Push/Pull 방향**의 복제를 설정할 수 있으며, **네임스페이스나 태그 필터** 적용, 스케줄 또는 수동 트리거가 가능합니다.
- 재해 복구(DR), 멀티 리전 배포, 에지 노드 동기화에 활용됩니다.

---

## CI/CD에서 사설 레지스트리 사용

### GitHub Actions 워크플로 예시

{% raw %}
```yaml
name: push-to-private-registry
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Trust self-hosted CA
        run: |
          sudo mkdir -p /etc/docker/certs.d/registry.example.com:443
          echo "${{ secrets.REGISTRY_CA_PEM }}" | sudo tee /etc/docker/certs.d/registry.example.com:443/ca.crt

      - name: Login to Private Registry
        run: echo "${{ secrets.REGISTRY_PASSWORD }}" | \
          docker login registry.example.com -u "${{ secrets.REGISTRY_USERNAME }}" --password-stdin

      - name: Build & Push Image
        run: |
          IMAGE=registry.example.com/demo/app
          TAG=${GITHUB_SHA::7}
          docker build -t $IMAGE:$TAG -t $IMAGE:latest .
          docker push $IMAGE:$TAG
          docker push $IMAGE:latest
```
{% endraw %}
- 사설 CA 인증서는 GitHub Secrets에 PEM 문자열 형태로 저장합니다.

---

## 백업 및 복구

### registry:2 (파일시스템 또는 S3 백엔드)

- **파일시스템**: `/var/lib/registry` 디렉터리 전체를 스냅샷 백업합니다 (레지스트리 정지 후 권장).
- **S3**: 객체 스토리지의 버킷 버저닝 또는 라이프사이클 정책에 의존합니다.

### Harbor

- 주요 구성 요소: **DB(PostgreSQL)**, **Registry Storage**, **Redis**(메타데이터 캐시).
- **필수 백업 대상**: DB 덤프 + Registry 스토리지 스냅샷 + Harbor 구성 파일.
- 백업 예시:
```bash
# PostgreSQL DB 덤프
docker exec -t harbor-db pg_dump -U postgres > harbor_$(date +%F).sql

# 로컬 파일 스토리지 백업
rsync -a /data/harbor/ /backup/harbor-$(date +%F)/
```
- 복구는 동일한 Harbor 버전으로 재배포한 후, 백업된 DB와 스토리지 데이터를 복원하는 과정을 거칩니다.

---

## 성능 및 운영 최적화

- **레이어 재사용**: 공통 베이스 이미지 사용, 빌드 캐시 최적화, `.dockerignore` 파일을 통해 불필요한 컨텍스트 제외.
- **S3 리다이렉트**: `REGISTRY_STORAGE_REDIRECT=true` 설정으로 클라이언트가 오브젝트 스토리지에서 레이어를 직접 다운로드하도록 하여 성능을 향상시킵니다.
- **프록시 캐시 활용**: 자주 사용하는 외부 이미지에 대해 프록시 캐시를 구성하여 내부 네트워크에서 빠르게 풀(pull)하고, 외부 Rate Limit을 회피합니다.
- **멀티 아키텍처 이미지**: `docker buildx`를 사용하여 단일 태그에 amd64, arm64 등 여러 아키텍처 이미지를 포함시킵니다. 클라이언트는 자동으로 적합한 이미지를 선택합니다.
- **정기적인 가비지 컬렉션**: 레이어 리텐션 정책과 함께 정기적인 오프라인 GC를 실행하여 저장 공간을 관리합니다.
- **CDN 연동**: 대규모 트래픽이 예상되는 경우, S3 백엔드와 CloudFront/Cloudflare 같은 CDN을 연동하여 엣지 네트워크에서 이미지를 제공할 수 있습니다.
- **TLS 튜닝**: HTTP/2 활성화, 적절한 `client_max_body_size` 설정, 네트워크 RTT에 맞는 `read_timeout` 값을 구성합니다.

---

## 보안 모범 사례

- **TLS 필수 적용**: 모든 통신에 TLS를 적용하고, 사설 CA 체인은 모든 클라이언트에 표준화된 방법으로 배포합니다.
- **안전한 인증**: `docker login` 시에는 가능한 한 개인 액세스 토큰(PAT)이나 Harbor의 **로봇 계정**을 사용합니다.
- **접근 제어 강화 (Harbor)**: 프로젝트를 프라이빗으로 설정하고, RBAC를 통해 역할을 세분화하며, 푸시/삭제/스캔 이벤트를 감지하는 **Webhook**을 설정합니다.
- **보안 정책 적용**: 푸시 시 자동 보안 스캔을 수행하고, Critical/High 수준의 취약점이 발견되면 차단하는 정책을 구성합니다. 중요한 태그에는 **불변성(Immutable)** 을 설정하여 덮어쓰기를 방지합니다.
- **서명 검증**: CI/CD 파이프라인에 cosign verify 단계를 추가하여 서명되지 않은 이미지의 배포를 차단합니다.
- **컨테이너 보안**: 레지스트리 서버 컨테이너는 **비루트(non-root) 사용자**로 실행하고, 가능하다면 **읽기전용 파일시스템**을 사용하며, 호스트 시스템에 대한 권한을 최소화합니다.

---

## 네트워크 및 인프라 팁

- Docker 데몬은 `/etc/docker/certs.d/<호스트:포트>/ca.crt` 경로 규칙을 따라 CA 인증서를 찾습니다. 예를 들어 `registry.example.com:443`에 접근한다면, 디렉터리 이름에 **포트 번호까지 정확히 포함**해야 합니다.
- 가능하면 표준 HTTPS 포트인 443을 사용하십시오. 5000이나 8443 같은 비표준 포트를 사용할 경우, 클라이언트의 방화벽이나 프록시 설정을 추가로 고려해야 합니다.
- 내부 DNS에 레지스트리 호스트명(`A` 또는 `CNAME` 레코드)을 등록하고, 클라이언트는 **FQDN(정규화된 도메인 이름)** 을 사용하여 풀(pull) 및 푸시(push)를 수행하도록 합니다.

---

## 문제 해결

| 증상 | 원인 및 해결 방법 |
|---|---|
| `x509: certificate signed by unknown authority` | 클라이언트 측 `certs.d/<host:port>/ca.crt` 경로에 인증서가 없거나, 인증서 체인이 불완전함. |
| `denied: requested access to the resource is denied` | 로그인 상태, 사용자 권한, 또는 해당 프로젝트의 프라이빗 설정을 확인하십시오. |
| 푸시 속도 느림 또는 중단 | 프록시 서버, IPS, 방화벽의 타임아웃 설정 확인. Nginx의 `proxy_read_timeout` 및 `client_max_body_size` 값을 조정하십시오. |
| GC 실행 후에도 저장 공간이 줄지 않음 | GC 실행 중 푸시/삭제 작업이 발생했을 수 있습니다. 레지스트리를 완전히 정지시킨 상태에서 오프라인 GC를 다시 실행하고, `--delete-untagged` 옵션을 확인하십시오. |
| Harbor 보안 스캔 실패 | Trivy의 오프라인 DB 동기화 설정 또는 네트워크 프록시 설정을 확인하십시오. Trivy 컴포넌트의 로그를 확인하여 업데이트 문제가 있는지 점검합니다. |

---

## 결론

개인용 Docker Registry는 개발 및 배포 파이프라인의 속도, 보안, 유연성, 비용 효율성을 크게 향상시킬 수 있는 핵심 인프라입니다. **간단한 개인 사용**에는 `registry:2`에 TLS와 기본 인증을 적용하는 것으로 충분히 시작할 수 있으며, 정기적인 가비지 컬렉션과 프록시 캐시 설정으로 운영 효율을 높일 수 있습니다.

반면, **팀 또는 조직 차원**에서 운영하기 위해서는 Harbor와 같은 엔터프라이즈급 레지스트리 솔루션을 고려해야 합니다. Harbor는 RBAC, 보안 스캔, 아티팩트 서명, 복제, 리텐션 정책 등 풍부한 기능을 제공하여 컨테이너 이미지의 수명 주기를 안전하고 체계적으로 관리할 수 있도록 돕습니다.

공통적으로 중요한 것은 **TLS를 통한 암호화 통신의 적용**, 안전한 인증 방식을 위한 **CA 인증서의 표준화된 배포**, 그리고 **주기적인 백업과 복구 절차의 확립**입니다. 이 가이드를 따라 단계적으로 레지스트리를 구축하고 운영한다면, 안정적이고 효율적인 개인/사내 컨테이너 이미지 관리 체계를 구축할 수 있을 것입니다.