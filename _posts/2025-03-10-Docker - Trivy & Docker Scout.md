---
layout: post
title: Docker - Trivy & Docker Scout
date: 2025-03-10 19:20:23 +0900
category: Docker
---
# 🔍 Docker 이미지 보안 스캔 도구: Trivy & Docker Scout

---

## 1️⃣ Trivy

### 📌 개요

- **오픈소스** 이미지 스캐너 (by Aqua Security)
- 이미지뿐 아니라 소스코드, Git 리포지토리, IaC (Terraform, Dockerfile 등)도 스캔 가능
- 빠르고 가벼우며, **CI/CD 연동에 매우 적합**

---

### ✅ 설치

```bash
# macOS
brew install aquasecurity/trivy/trivy

# Linux (APT 기반)
sudo apt install trivy

# Docker로 실행
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image nginx:alpine
```

---

### 🔍 이미지 스캔 예제

```bash
trivy image nginx:latest
```

출력 예시:

```
nginx:latest (debian 12)
=========================
Total: 5 (UNKNOWN: 0, LOW: 2, MEDIUM: 1, HIGH: 1, CRITICAL: 1)

┌────────────┬────────────────────────────┬───────────┬────────────┬──────────────┐
│  Package   │        Vulnerability ID     │ Severity  │ Installed  │ Fixed Version │
├────────────┼────────────────────────────┼───────────┼────────────┼──────────────┤
│ openssl    │ CVE-2023-3817               │ CRITICAL  │ 3.0.2      │ 3.0.8         │
└────────────┴────────────────────────────┴───────────┴────────────┴──────────────┘
```

---

### 🔐 코드/파일 스캔 (Dockerfile, git 등)

```bash
# Dockerfile 검사
trivy config Dockerfile

# GitHub 레포지토리 검사
trivy repo https://github.com/flaskcwg/flask

# S3/Secrets 노출 여부
trivy fs .
```

---

### 🔁 CI/CD 연동 예시 (GitHub Actions)

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@v0.15.0
  with:
    image-ref: yourname/your-image:latest
```

---

## 2️⃣ Docker Scout (이전: Docker Scan)

### 📌 개요

- Docker Desktop에 내장된 **GUI 기반 보안 스캐너**
- CLI에서도 사용 가능 (`docker scout` 명령어)
- **SBOM(소프트웨어 구성 요소 목록)** 및 **CVE 분석**, **이미지 비교** 기능 제공
- Snyk 엔진 기반이었으나 현재 자체 분석 엔진 보유

---

### ✅ CLI 사용 예시

```bash
docker scout quickview nginx:latest
```

출력 예시:

```
Security issues: 3 Critical, 2 High, 5 Medium
Recommendations: Upgrade to nginx:1.25.2 to fix 2 critical issues
```

---

### 🔎 주요 기능

| 기능 | 설명 |
|------|------|
| Vulnerability scan | 이미지에 포함된 CVE 목록 제공 |
| SBOM 생성 | 어떤 라이브러리가 포함됐는지 확인 |
| 이미지 비교 | 태그 간 차이점 (패키지, 취약점 등) 분석 |
| 정책 설정 | 조직 기준에 맞춘 보안 정책 정의 |

---

### 🧰 사용 예 (SBOM 생성)

```bash
docker scout sbom nginx:latest
```

---

### 🖥️ Docker Desktop UI

1. 이미지 목록에서 → "Analyze" 클릭
2. 취약점(CVE), 패키지, 레이어 별 분석 제공
3. 추천 업그레이드 버전까지 안내

---

## 🆚 Trivy vs Docker Scout 비교

| 항목 | Trivy | Docker Scout |
|------|-------|---------------|
| 오픈소스 여부 | ✅ 완전 오픈 | ❌ 일부 비공개 |
| CLI 지원 | ✅ | ✅ |
| GUI 지원 | ❌ | ✅ (Docker Desktop) |
| 대상 범위 | 이미지, 코드, IaC, Git 등 | 이미지 중심 |
| 확장성 | 매우 넓음 | Docker 사용자 중심 |
| CI/CD 통합 | 우수 | 중간 |
| 속도 | 매우 빠름 | 빠름 |
| SBOM 지원 | ✅ | ✅ |

---

## 🧪 실제 적용 예: 이미지 푸시 전 검사

```bash
docker build -t myapp .
trivy image myapp
```

문제 없을 때에만 push:

```bash
docker push myname/myapp:latest
```

---

## 🛡️ 권장 워크플로우

1. Dockerfile 작성 → `trivy config Dockerfile`
2. 이미지 빌드 → `trivy image`
3. 코드 검사 → `trivy fs .`
4. GitHub Action에 통합
5. 프로덕션에선 **Scout UI 또는 Watchtower 연동**도 고려

---

## 📚 참고 링크

- [Trivy 공식 문서](https://aquasecurity.github.io/trivy/)
- [Docker Scout 공식 페이지](https://docs.docker.com/scout/)
- [GitHub Action for Trivy](https://github.com/aquasecurity/trivy-action)
- [SBOM이란?](https://cyclonedx.org/docs/about/)
