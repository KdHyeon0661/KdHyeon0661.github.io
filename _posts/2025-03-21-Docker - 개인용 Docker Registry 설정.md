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
| 속도 | 사내 네트워크/폐쇄망에서 **pull 지연 최소화** |
| 보안 | 민감 이미지/모델/자산 **외부 유출 방지** |
| 유연성 | **프로젝트별 네임스페이스**, 태그 규칙, 정책 적용 |
| 오프라인 | 인터넷 미접속 환경(산단/관제/국방 등)에서 필수 |
| 비용 | 외부 레이트리밋/전송 비용 회피, **캐시**로 대역 절감 |

---

## 옵션 비교: `registry:2` vs Harbor

| 항목 | Docker Registry (registry:2) | Harbor |
|---|---|---|
| UI | 없음 (CLI/REST) | 웹 UI 제공 |
| 인증/권한 | Basic Auth, 프록시 앞단 필요 | 로컬/LDAP/OIDC, **RBAC 프로젝트** |
| 서명/신뢰 | 없음 | Notary/Cosign 아티팩트 관리 |
| 보안 스캔 | 없음 | Trivy/Clair 통합 |
| 프록시 캐시 | **지원**(pull-through cache) | **Proxy Cache Project** 고급화 |
| 스토리지 | FS, S3 등 **다양** | 로컬/오브젝트, 외부 DB/Redis |
| 복제/미러 | 단순 | **Replication**(양방향/스케줄/필터) |
| 배포 | 단일 컨테이너/Compose | Compose 또는 **Helm(K8s)** |
| 용도 | 경량 프라이빗 저장소 | 팀/조직용 풀스택 레지스트리 |

**선택 가이드**
- 개인/소규모, 간단·저비용 → **registry:2**
- 팀·조직, 정책·감사·스캔·서명·복제 → **Harbor**

---

## Registry 아키텍처 요약

- **Manifests**: 이미지 메타(레이어 목록, 플랫폼). 동일 태그가 여러 아키(amd64/arm64) **매니페스트 리스트**를 가질 수 있음.
- **Layers(Blobs)**: 압축된 파일시스템 조각. **중복 제거**로 저장공간 절약.
- **Name/Tag**: `<host>/<namespace>/<repo>:<tag>`; 운영은 **digest 핀** 권장(`@sha256:...`).

---

## 공식 Docker Registry(경량) — 빠른 시작

### 가장 단순한 실행

```bash
docker run -d --name registry -p 5000:5000 registry:2
```
- 저장 경로: 컨테이너 `/var/lib/registry` (바인드/볼륨으로 **영속화** 권장)

```bash
# 영속 볼륨

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

> 기본은 **HTTP**라 로컬에서만 권장. 실제 서비스는 **TLS(HTTPS)** 필수.

---

## TLS(HTTPS) 구성

### 자체 서명(테스트)

```bash
mkdir -p certs
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout certs/domain.key -x509 -days 365 -out certs/domain.crt \
  -subj "/CN=registry.example.com"
```

### TLS로 Registry 실행(443 바인딩)

```bash
docker run -d --name registry \
  -p 443:5000 \
  -v $(pwd)/certs:/certs:ro \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

### 클라이언트 신뢰 구성

```bash
# certs.d 디렉터리명은 "호스트:포트"와 정확히 일치해야 함

sudo mkdir -p /etc/docker/certs.d/registry.example.com:443
sudo cp certs/domain.crt /etc/docker/certs.d/registry.example.com:443/ca.crt
```

> 사설 CA/회사 CA 사용 시 **체인 전체**를 `ca.crt`에 포함.

---

## Basic Auth(아이디/비번) — registry:2

### htpasswd 생성

```bash
mkdir -p auth
docker run --rm --entrypoint htpasswd registry:2 \
  -Bbn testuser testpassword > auth/htpasswd
```

### 인증 활성화 실행

```bash
docker run -d --name registry \
  -p 5000:5000 \
  -v $(pwd)/auth:/auth:ro \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2
```

### 로그인

```bash
docker login localhost:5000
```

> 고급 요구(SSO/RBAC)는 Harbor를 고려.

---

## Compose로 운영형 배치 + 가비지 컬렉션·프록시 캐시·S3

### `docker-compose.yml` 예시(파일시스템 + Basic Auth + TLS)

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
      REGISTRY_STORAGE_DELETE_ENABLED: "true"     # blob 삭제 기능 (GC 준비)
    volumes:
      - ./data:/var/lib/registry
      - ./auth:/auth:ro
      - ./certs:/certs:ro
    restart: unless-stopped
```

### Pull-through 캐시(도커 허브 프록시)

```yaml
environment:
  REGISTRY_PROXY_REMOTEURL: https://registry-1.docker.io
```
- 외부 이미지를 사내에 **캐시**하여 **레이트리밋/대역 절감**.

### S3/MinIO 백엔드(대용량·중복제거·리전 복제)

```yaml
environment:
  REGISTRY_STORAGE: s3
  REGISTRY_STORAGE_S3_REGION: ap-northeast-2
  REGISTRY_STORAGE_S3_BUCKET: my-registry-bucket
  REGISTRY_STORAGE_S3_ACCESSKEY: ${S3_ACCESS}
  REGISTRY_STORAGE_S3_SECRETKEY: ${S3_SECRET}
  REGISTRY_STORAGE_S3_REGIONENDPOINT: http://minio:9000  # MinIO 예시
  REGISTRY_STORAGE_S3_ENCRYPT: "true"
  REGISTRY_STORAGE_S3_SECURE: "false"                   # http endpoint시
  REGISTRY_STORAGE_S3_ROOTDIRECTORY: /registry
  REGISTRY_STORAGE_REDIRECT: "true"                     # 성능↑(오브젝트 직접 다운로드)
```

### 가비지 컬렉션(GC)

- Registry는 **참조되지 않는 blob**을 정리해야 용량 회수.
- 안전한 절차:
```bash
# 푸시/삭제 중단(정지 권장)

docker stop registry

# 오프라인 GC 실행

docker run --rm \
  -v $(pwd)/data:/var/lib/registry \
  -v $(pwd)/config.yml:/etc/docker/registry/config.yml \
  registry:2 garbage-collect /etc/docker/registry/config.yml --delete-untagged=true

# 재기동

docker start registry
```
- `config.yml`은 실제 실행과 동일해야 함(스토리지 설정 포함).

---

## Reverse Proxy(Nginx) 앞단 구성(선택)

### 이유

- **TLS/리다이렉트/접근제어/레이트리밋** 등 고급 정책을 일관 처리.
- 다중 레지스트리/서비스를 **도메인별 라우팅**.

### 간단 Nginx 설정 예

```nginx
server {
  listen 443 ssl http2;
  server_name registry.example.com;

  ssl_certificate     /etc/nginx/certs/fullchain.pem;
  ssl_certificate_key /etc/nginx/certs/privkey.pem;

  # 대용량 푸시 튜닝
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

## Harbor(풀스택) 설치

### 요구

- Docker & Docker Compose(또는 K8s + Helm)
- 4GB+ RAM, FQDN, TLS(권장), 외부 DB/객체스토리지 선택 가능

### 설치(Compose 기반)

```bash
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz
tar xvf harbor-online-installer-*.tgz
cd harbor

cp harbor.yml.tmpl harbor.yml
vi harbor.yml
# 핵심:
# hostname: registry.example.com
# https:
#   port: 443
#   certificate: /path/to/cert.crt
#   private_key: /path/to/key.key
# external_database, external_storage, oidc, proxy-cache 등 필요시 설정

sudo ./install.sh
```
- 접속: `https://registry.example.com`
- 기본 관리자: `admin / Harbor12345`(설치 직후 **즉시 변경**)

### Helm(Kubernetes) 배포 개요

- `helm repo add harbor https://helm.goharbor.io`
- `values.yaml`에 Ingress/TLS/스토리지/DB/Redis/Trivy/Notary/OIDC/RBAC 설정 후 배포:
```bash
helm install harbor harbor/harbor -f values.yaml -n harbor --create-namespace
```

---

## Harbor 핵심 기능 운영 가이드

### 프로젝트/RBAC

- 프로젝트 단위로 **퍼블릭/프라이빗** 설정, **역할(관리자/개발/게스트)** 부여.
- **Robot Account** 발급 → CI/CD가 비밀번호 대신 토큰으로 푸시/풀.

### Proxy Cache Project

- 외부 레지스트리(도커 허브/ghcr) 캐시 전용 프로젝트.
- 개발 속도↑, 레이트리밋 회피.

### 보안 스캔

- **Trivy** 내장 선택(오프라인 DB 동기화 가능).
- 프로젝트 정책: 푸시/프로모션 시 **취약점 임계치** 기준 차단.

### 서명/신뢰(NOTARY/Cosign)

- Harbor는 **서명 아티팩트**와 **정책**을 관리(아티팩트 탭).
```bash
# cosign 예시(외부에서 서명 → Harbor는 결과/메타 관리)

cosign generate-key-pair
cosign sign registry.example.com/demo/app:1.0.0
cosign verify registry.example.com/demo/app:1.0.0
```

### 리텐션/쿼터/불변 태그

- **Retention Policy**: 오래된 태그/패턴 자동 삭제(스케줄).
- **Tag Immutability**: 특정 패턴(예: `v*`) 태그 **덮어쓰기 방지**.
- **Project Quota**: 스토리지/태그 개수 제한.

### 복제(Replication)

- 원격 Harbor/Registry와 **push/pull 방향** 복제, **네임스페이스/태그 필터**, 스케줄/수동 트리거.
- DR/멀티리전/에지노드 동기화에 활용.

---

## CI/CD에서 사설 레지스트리 사용

### GitHub Actions 예

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

      - name: Login
        run: echo "${{ secrets.REGISTRY_PASSWORD }}" | \
          docker login registry.example.com -u "${{ secrets.REGISTRY_USERNAME }}" --password-stdin

      - name: Build & Push
        run: |
          IMAGE=registry.example.com/demo/app
          TAG=${GITHUB_SHA::7}
          docker build -t $IMAGE:$TAG -t $IMAGE:latest .
          docker push $IMAGE:$TAG
          docker push $IMAGE:latest
```
{% endraw %}
- 사설 CA는 **Secrets**에 PEM 문자열로 저장.

---

## 백업·복구

### registry:2 (FS/S3)

- **FS**: `/var/lib/registry` 디렉터리 스냅샷(정지 후).
- **S3**: 스토리지 백업 정책에 따름(버킷 버저닝/라이프사이클).

### Harbor

- 구성요소: **DB(PostgreSQL)**, **Registry Storage**, **Redis**(메타 캐시).
- **필수 백업**: DB dump + Registry 스토리지 스냅샷 + 설정 파일.
- 예시:
```bash
# PostgreSQL

docker exec -t harbor-db pg_dump -U postgres > harbor_$(date +%F).sql

# 파일 스토리지(로컬일 때)

rsync -a /data/harbor/ /backup/harbor-$(date +%F)/

# 복구는 같은 버전으로 재배포 후 DB/스토리지 복원

```

---

## 성능·용량 최적화 체크리스트

| 항목 | 요령 |
|---|---|
| 레이어 재사용 | Base image/빌드 캐시 최적화, `.dockerignore` 정리 |
| S3 redirect | `REGISTRY_STORAGE_REDIRECT=true` 로 **오브젝트 직접 다운로드** |
| 프록시 캐시 | 빈번한 외부 pull은 Proxy Cache로 내부화 |
| 멀티아키 | `buildx`로 단일 태그에 amd64/arm64 합치기(클라이언트 적합 자동 선택) |
| GC 주기 | 리텐션 정책 + 정기 **오프라인 GC** |
| CDN | 대규모 트래픽은 S3 + CloudFront/Cloudflare로 엣지 가속 |
| TLS 튜닝 | HTTP/2, 적절한 `client_max_body_size`, keepalive, 대역폭/RTT에 맞춘 read_timeout |

---

## 보안 모범사례

- **TLS 강제**, 사설 CA 체인 배포 표준화.
- `docker login`은 **PAT/로봇 계정**(Harbor) 사용.
- Harbor: **프로젝트 프라이빗 + RBAC + Webhook**(푸시/삭제/스캔 이벤트).
- 푸시 허용 전 **스캔 정책**(Critical/High 차단), **불변 태그**로 덮어쓰기 방지.
- **서명 검증 게이트**(cosign verify 실패 시 배포 차단).
- 레지스트리 서버는 **비루트 실행** 컨테이너, **읽기전용 FS**(가능 시), 호스트와 최소 권한.

---

## 네임 해석·포트·인프라 팁

- Docker는 `certs.d/<호스트:포트>/ca.crt` 규칙으로 CA를 찾는다.
  예) `registry.example.com:443` 디렉터리명에 **포트까지 포함**.
- 가능한 443 사용(SSL/TLS 표준). 5000/8443 등 비표준 포트는 방화벽/프록시 고려.
- 내부 DNS에 `A/CNAME` 등록, 클라이언트는 **FQDN으로 pull/push**.

---

## 트러블슈팅

| 증상 | 원인/해결 |
|---|---|
| `x509: certificate signed by unknown authority` | 클라이언트 `certs.d/<host:port>/ca.crt` 누락/체인 불완전 |
| `denied: requested access to the resource is denied` | 로그인/권한/프로젝트 프라이빗 상태 점검 |
| 푸시 느림/중단 | 프록시/IPS/방화벽 타임아웃, Nginx `proxy_read_timeout`/`client_max_body_size` 조정 |
| GC 후에도 용량 비슷 | 실행 중 푸시/삭제 → 오프라인 GC로 재시도, `--delete-untagged` 확인 |
| Harbor 스캔 실패 | 오프라인 DB 동기화/네트워크 프록시, Trivy 업데이트 설정 재확인 |

---

## 대역 절감 추정(간단 계산 메모)

한 기간 동안 외부에서 당겨 오던 N개의 이미지를 Proxy Cache로 내부화하면 총 트래픽 절감량 \(S\)는 대략

$$
S \approx \sum_{i=1}^{N} \left( D_i \cdot (k_i - 1) \right)
$$

여기서 \(D_i\)는 이미지 \(i\)의 평균 다운로드 바이트, \(k_i\)는 동일 이미지의 총 요청 횟수. 최초 1회는 외부 다운로드이므로 \(k_i-1\) 회가 내부 캐시 적중으로 절감된다. 실제로는 레이어 중복/공유로 \(D_i\)가 더 작아지므로 절감폭은 **위 식 이상**이 되는 경향이 있다.

---

## 부록 A. registry:2 전체 설정 파일 예시(`config.yml`)

```yaml
version: 0.1
log:
  fields:
    service: registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
# TLS는 환경변수로 지정했으면 생략 가능

storage:
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true
# 프록시 캐시(선택)

proxy:
  remoteurl: https://registry-1.docker.io
```

GC 시에는 **실행 중인 설정과 동일한 파일**을 사용해야 한다.

---

## 부록 B. Harbor values.yaml(Helm) 핵심 발췌

```yaml
expose:
  type: ingress
  tls:
    enabled: true
  ingress:
    hosts:
      core: registry.example.com
externalURL: https://registry.example.com

harborAdminPassword: "ChangeThisNow"

database:
  type: external
  external:
    host: pg.example.com
    port: 5432
    username: harbor
    password: "secret"
    coreDatabase: registry_core

registry:
  replicas: 2
  storage:
    type: s3
    s3:
      region: ap-northeast-2
      bucket: harbor-registry
      accesskey: AKIA...
      secretkey: xxx...
      regionendpoint: https://s3.ap-northeast-2.amazonaws.com
      rootdirectory: /artifacts
trivy:
  enabled: true
notary:
  enabled: false
chartmuseum:
  enabled: true
jobservice:
  replicas: 2
```

---

## 요약

- **간단/개인**: `registry:2` + TLS + Basic Auth + 프록시 캐시 + 주기적 **GC**.
- **팀/조직**: **Harbor** + RBAC + 스캔 + 서명 + 복제 + 리텐션/쿼터 + Webhook.
- 공통: **TLS/CA 배포**, **digest 고정**, **CI 로봇 계정/토큰**, **백업/복구 시나리오**.
- 성장 경로: 단일 컨테이너 → Compose 운영 → K8s/Helm(Harbor) + 오브젝트 스토리지.

> 이 가이드를 따라가면 “설치 → 보안(HTTPS/인증) → 프록시 캐시/S3 → 스캔/서명 → 리텐션/GC → CI/CD → 백업/복구”까지 **온전한 개인/팀 레지스트리**가 완성된다.
