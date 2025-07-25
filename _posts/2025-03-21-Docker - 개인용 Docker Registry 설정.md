---
layout: post
title: Docker - 개인용 Docker Registry 설정
date: 2025-03-21 20:20:23 +0900
category: Docker
---
# 🏠 개인용 Docker Registry 설정
> Harbor 및 Docker Registry(Self-hosted) 비교 & 구축 가이드

---

## 1️⃣ 왜 개인 Registry가 필요한가?

| 필요성 | 설명 |
|--------|------|
| 속도 | 사내 네트워크에서 Pull 속도 향상 |
| 보안 | 민감한 이미지 유출 방지 |
| 유연성 | 태그 관리, 권한 제어, 미러 설정 가능 |
| 오프라인 | 폐쇄망 환경에서 사용 가능 |

---

## 2️⃣ 옵션 비교: Harbor vs 공식 Docker Registry

| 항목 | Docker Registry | Harbor |
|------|-----------------|--------|
| UI | ❌ 없음 (CLI only) | ✅ 웹 UI |
| 인증 | ❌ 기본 없음 | ✅ LDAP, OIDC 지원 |
| 권한 제어 | ❌ 없음 | ✅ 프로젝트, 사용자별 권한 |
| 이미지 서명 | ❌ 없음 | ✅ Notary 지원 |
| 보안 스캔 | ❌ 없음 | ✅ Trivy, Clair 내장 가능 |
| 배포 | Docker 컨테이너 | Docker or Helm |
| 용도 | 간단한 프라이빗 저장소 | 완전한 기업용 Registry |

---

## 🔧 3️⃣ Docker Registry 설치 (공식 self-hosted)

### 📦 설치

```bash
docker run -d \
  -p 5000:5000 \
  --name registry \
  registry:2
```

- 기본 포트: `5000`
- 저장소 경로: `/var/lib/registry`

---

### 🐳 이미지 Push 예제

```bash
# 태그 변경 (로컬 Registry용)
docker tag ubuntu:20.04 localhost:5000/my-ubuntu

# 푸시
docker push localhost:5000/my-ubuntu

# Pull 테스트
docker pull localhost:5000/my-ubuntu
```

> 기본 설정에서는 TLS 없이 HTTP로 동작  
> 실제 서비스 시에는 HTTPS 설정 필수 (see below)

---

## 🔒 4️⃣ TLS 인증서 설정

### 📂 디렉토리 구성

```bash
mkdir -p certs
openssl req -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key \
  -x509 -days 365 -out certs/domain.crt
```

### 🐳 HTTPS Registry 실행

```bash
docker run -d \
  -p 443:5000 \
  --name registry \
  -v $(pwd)/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

### 🧩 Docker 클라이언트에 신뢰 설정

```bash
# Ubuntu 예시
sudo mkdir -p /etc/docker/certs.d/<your-domain>:443
sudo cp certs/domain.crt /etc/docker/certs.d/<your-domain>:443/ca.crt
```

---

## 👥 5️⃣ 사용자 인증 (basic auth)

```bash
mkdir auth
docker run --entrypoint htpasswd registry:2 -Bbn testuser testpassword > auth/htpasswd
```

```bash
docker run -d \
  -p 5000:5000 \
  -v $(pwd)/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  registry:2
```

→ Push 시 다음 명령 사용:

```bash
docker login localhost:5000
```

---

## 🧭 6️⃣ Harbor 설치 (기업용 풀기능 Registry)

### 🐋 Harbor란?

- CNCF 프로젝트, Docker Registry 기반 고급 관리 플랫폼
- UI, 사용자 관리, RBAC, 스캔, 서명, Helm Repo 등 지원

---

### 🧱 요구사항

| 항목 | 내용 |
|------|------|
| Docker & Docker Compose | 필수 |
| 최소 메모리 | 4GB 이상 |
| FQDN or IP | TLS 인증서 구성용 |

---

### 📦 설치 절차

1. [Harbor GitHub Release](https://github.com/goharbor/harbor/releases)에서 최신 버전 다운로드

```bash
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz
tar xvf harbor-online-installer-*.tgz
cd harbor
```

2. 설정 파일 수정

```bash
cp harbor.yml.tmpl harbor.yml
vim harbor.yml
```

설정 항목:

```yaml
hostname: registry.example.com
https:
  port: 443
  certificate: /your/cert/path.crt
  private_key: /your/key/path.key
```

3. 설치 실행

```bash
sudo ./install.sh
```

---

### 🌐 접속

- `https://registry.example.com`
- 기본 관리자 ID: `admin / Harbor12345`

---

## 🔐 Harbor 주요 기능 요약

| 기능 | 설명 |
|------|------|
| UI 기반 관리 | 웹에서 이미지, 사용자, 권한 설정 |
| RBAC | 팀, 프로젝트, 권한 세분화 |
| 이미지 서명 | Notary or Cosign |
| 보안 스캔 | Trivy, Clair 통합 가능 |
| Webhook, Replication | 이미지 복제, 이벤트 알림 |
| Helm Chart 저장소 | Helm 저장소도 지원 |

---

## 📚 참고 문서

- [Docker Registry 공식 문서](https://docs.docker.com/registry/)
- [Harbor 공식 사이트](https://goharbor.io/)
- [Harbor 설치 가이드](https://goharbor.io/docs/)
- [Self-hosted TLS 인증서 설정](https://docs.docker.com/registry/insecure/)

---

## ✅ 요약

| 방식 | 특징 | 적합한 용도 |
|------|------|-------------|
| `registry:2` | 경량, CLI 기반 | 개인 실습용, 내부 빌드 서버 |
| `Harbor` | UI + 보안 + 관리 기능 | 사내 배포, 조직/팀 단위 운영 |
