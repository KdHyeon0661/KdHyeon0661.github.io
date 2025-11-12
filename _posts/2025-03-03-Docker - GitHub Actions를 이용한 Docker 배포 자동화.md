---
layout: post
title: Docker - GitHub Actions를 이용한 Docker 배포 자동화
date: 2025-03-03 20:20:23 +0900
category: Docker
---
# GitHub Actions를 이용한 Docker 배포 자동화

## 0. 목표/흐름/디렉터리

### 목표
- 푸시/릴리스 시 **자동 빌드·푸시·배포**
- **재현 가능한** 멀티스테이지 빌드와 **빠른 캐시**
- 태그/레이블/라벨 메타데이터 자동화
- **보안 스캔(SBOM/취약점)·서명** 도입
- **승인 기반(prod) 배포**, 실패 시 **즉시 롤백**

### 기본 흐름

```
[코드 Push/Release] → [Actions Runner]
  → (QEMU/Buildx) Docker Build
  → (metadata/cosign/trivy/syft)
  → Registry Push (Docker Hub or GHCR)
  → (옵션) SSH/웹훅/Compose/K8s 배포
```

### 디렉터리 예시

```plaintext
my-app/
├── app/
│   └── main.py
├── Dockerfile
├── docker-compose.yml
└── .github/
    └── workflows/
        ├── docker-publish.yml
        ├── deploy-ssh.yml
        └── rollback.yml
```

---

## 1. Dockerfile — 멀티스테이지 + 런타임 경량화

```dockerfile
# syntax=docker/dockerfile:1.7

# 1. 빌드 스테이지
FROM python:3.11-slim AS builder
WORKDIR /app
ENV PIP_DISABLE_PIP_VERSION_CHECK=1 PIP_NO_CACHE_DIR=1
COPY app/ ./app/
COPY requirements.txt .
RUN python -m pip install --upgrade pip \
 && pip install --prefix=/install -r requirements.txt

# 2. 런타임 스테이지(경량)
FROM gcr.io/distroless/python3-debian12:nonroot
WORKDIR /app
ENV PYTHONUNBUFFERED=1
COPY --from=builder /install /usr/local
COPY app/ ./app/
USER nonroot
EXPOSE 8080
CMD ["app/main.py"]
```

핵심:
- **멀티스테이지**로 최종 이미지 최소화
- **distroless**로 공격면 축소(쉘 없음)
- `USER nonroot`로 기본 권한 최소화

---

## 2. 기본 CI — Docker Hub 로그인/빌드/푸시

### 사전 준비(Secrets)
- `DOCKER_USERNAME`, `DOCKER_PASSWORD` (Docker Hub 토큰 권장)
- (옵션) `IMAGE_NAME` = `yourname/my-app`

### `.github/workflows/docker-publish.yml`

{% raw %}
```yaml
name: docker-publish

on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up QEMU (multi-arch)
        uses: docker/setup-qemu-action@v3

      - name: Set up Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ secrets.DOCKER_USERNAME }}/my-app
          tags: |
            type=raw,value=latest
            type=sha
            type=ref,event=branch
            type=semver,pattern={{version}}
          labels: |
            org.opencontainers.image.source=${{ github.repository }}
            org.opencontainers.image.revision=${{ github.sha }}

      - name: Build and push (multi-arch)
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          labels: ${{ steps.meta.outputs.labels }}
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```
{% endraw %}

포인트:
- **metadata-action**으로 태그/레이블 자동 생성(커밋 SHA, 브랜치, semver)
- **buildx + qemu**로 `amd64/arm64` 동시 빌드
- **GHA 캐시**로 빌드 시간 단축

---

## 3. GHCR로 푸시(조직/프라이빗 운영에 적합)

GHCR는 기본으로 `GITHUB_TOKEN`으로 로그인 가능.

{% raw %}
```yaml
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}/my-app
          tags: |
            type=raw,value=latest
            type=sha
            type=ref,event=branch
```
{% endraw %}

이미지 참조: `ghcr.io/<owner>/<repo>/my-app:latest`

---

## 4. 보안·신뢰도: SBOM/취약점 스캔/서명

### 4.1 SBOM 생성(Syft)

{% raw %}
```yaml
      - name: Generate SBOM (Syft)
        uses: anchore/sbom-action@v0
        with:
          image: ${{ steps.meta.outputs.tags }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Upload SBOM artifact
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.spdx.json
```
{% endraw %}

### 4.2 취약점 스캔(Trivy)

{% raw %}
```yaml
      - name: Scan image (Trivy)
        uses: aquasecurity/trivy-action@0.22.0
        with:
          image-ref: ${{ secrets.DOCKER_USERNAME }}/my-app:latest
          format: table
          severity: CRITICAL,HIGH
          ignore-unfixed: true
```
{% endraw %}

### 4.3 컨테이너 이미지 서명(Cosign)

{% raw %}
```yaml
      - name: Install Cosign
        uses: sigstore/cosign-installer@v3.6.0

      - name: Cosign sign (keyless OIDC)
        run: |
          cosign sign --yes ${{ secrets.DOCKER_USERNAME }}/my-app:latest
```
{% endraw %}

> OIDC 기반 **keyless** 서명을 쓰면 별도 키 보관 없이 서명 가능(레지스트리/정책과 연계해 배포 신뢰성 강화).

---

## 5. 배포 — 원격 서버에 SSH로 롤링 교체

### 사전 준비(Secrets)
- `REMOTE_HOST`, `REMOTE_USER`, `REMOTE_SSH_KEY`(private key)
- 서버에 Docker 설치 및 레지스트리 접근 가능해야 함.

`.github/workflows/deploy-ssh.yml`:

{% raw %}
```yaml
name: deploy-ssh
on:
  workflow_dispatch:
  push:
    branches: [ "main" ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production    # 환경 승인 사용 시
    steps:
      - uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.REMOTE_HOST }}
          username: ${{ secrets.REMOTE_USER }}
          key: ${{ secrets.REMOTE_SSH_KEY }}
          script: |
            set -e
            docker pull ${{ secrets.DOCKER_USERNAME }}/my-app:latest
            docker compose -f /srv/my-app/docker-compose.yml down || true
            docker compose -f /srv/my-app/docker-compose.yml up -d
            docker image prune -af --filter "until=168h"
```
{% endraw %}

서버 측 `docker-compose.yml`(예시):

```yaml
services:
  app:
    image: yourname/my-app:latest
    restart: unless-stopped
    ports:
      - "80:8080"
    environment:
      APP_ENV: production
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:8080/healthz"]
      interval: 30s
      timeout: 3s
      retries: 3
```

> `environment: production`을 사용하면 GitHub 환경 보호 규칙을 통해 **배포 승인(Approvals)**을 강제할 수 있다.

---

## 6. Watchtower/Portainer/Webhook 기반 자동 재배포

### 6.1 Watchtower로 최신 태그 자동 갱신

서버에서 워치타워 실행:

```bash
docker run -d \
  --name watchtower \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower --cleanup --interval 30
```

**Actions에서 웹훅 호출**(예: 사내 엔드포인트):

```yaml
- name: Notify deploy webhook
  run: curl -fsS -X POST https://yourserver.example.com/deploy-hook
```

워치타워는 최신 이미지 감지 시 자동 재시작.

### 6.2 Portainer Webhook

Portainer에서 Stack에 Webhook URL 발급 → Actions에서 `curl` 호출로 업데이트 트리거.

---

## 7. 태깅 전략 — latest + semver + sha

권장:
- `latest`는 최신 안정 배포용
- `vX.Y.Z`는 릴리스(Release) 트리거에서만
- 커밋 트레이스 용도로 `sha` 태그

릴리스 트리거 예시:

```yaml
on:
  release:
    types: [published]

# metadata-action에서 type=semver 활성화
```

---

## 8. 성능 최적화 — 빌드 캐시/레이어 관리/.dockerignore

### 8.1 캐시
- `cache-from: type=gha` / `cache-to: type=gha,mode=max`
- 빈번 변경 파일(앱 코드)은 **Dockerfile 하단**에 COPY → 캐시 활용

### 8.2 .dockerignore

```
.git
.github
.env
**/__pycache__
*.log
node_modules
public
```

### 8.3 멀티스테이지
- 런타임에 **필요한 산출물만** COPY
- **distroless/alpine** 기반 런타임으로 용량/공격면 최소화

---

## 9. 보호/가시성 — 환경 승인/병행 제어/아티팩트

### 9.1 환경 승인(Approvals)
- `jobs.<job>.environment: production` → 환경 보호 규칙에서 승인자 지정

### 9.2 병행 제어(동시 배포 방지)

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

### 9.3 빌드 산출물/로그 보관

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-logs
    path: /tmp/build-logs/*
```

---

## 10. 롤백 전략 — 이전 태그로 빠르게 복귀

`.github/workflows/rollback.yml`:

{% raw %}
```yaml
name: rollback
on:
  workflow_dispatch:
    inputs:
      tag:
        description: "Rollback to tag (e.g., v1.2.3 or sha)"
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - name: SSH rollback
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.REMOTE_HOST }}
          username: ${{ secrets.REMOTE_USER }}
          key: ${{ secrets.REMOTE_SSH_KEY }}
          script: |
            set -e
            IMG="${{ secrets.DOCKER_USERNAME }}/my-app:${{ github.event.inputs.tag }}"
            docker pull $IMG
            docker compose -f /srv/my-app/docker-compose.yml down || true
            sed -i "s#image: .*#image: $IMG#g" /srv/my-app/docker-compose.yml
            docker compose -f /srv/my-app/docker-compose.yml up -d
```
{% endraw %}

---

## 11. 블루-그린/카나리(Compose로 단순화)

### 블루-그린
- `app-blue`, `app-green` 두 서비스를 올리고 **Nginx 프록시**에서 업스트림 전환
- 전환 전 **헬스체크/사전 트래픽 테스트**

Nginx 업스트림 예시:

```nginx
upstream app {
  server app-blue:8080 backup;
  server app-green:8080;
}
```

### 카나리
- `latest`와 `sha` 태그 이미지를 **서로 다른 서비스**로 띄우고 라우팅 비율 조정

---

## 12. 개발/스테이징/운영 분리

- 브랜치/태그 규칙:
  - `feat/*` → 임시 이미지 `pr-<num>` 태그
  - `develop` → `dev` 환경 자동 배포
  - `main` → `staging` 자동, 승인 후 `production`

예) 이벤트 조건 분기:

```yaml
on:
  push:
    branches: [ "develop", "main" ]
```

`if: github.ref == 'refs/heads/main'` 조건으로 prod 단계 분리.

---

## 13. 종합 예제 — 확장형 워크플로(요약)

{% raw %}
```yaml
name: ci-cd

on:
  push:
    branches: [ "main", "develop" ]
  release:
    types: [ published ]
  workflow_dispatch:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write   # cosign keyless
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3

      - name: Login GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}/my-app
          tags: |
            type=raw,value=latest
            type=ref,event=branch
            type=sha
            type=semver,pattern={{version}}

      - name: Build & Push
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ${{ fromJSON(steps.meta.outputs.json).tags[0] }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.spdx.json

      - name: Trivy scan
        uses: aquasecurity/trivy-action@0.22.0
        with:
          image-ref: ${{ fromJSON(steps.meta.outputs.json).tags[0] }}
          ignore-unfixed: true
          severity: CRITICAL,HIGH

      - name: Cosign
        uses: sigstore/cosign-installer@v3.6.0
      - run: cosign sign --yes ${{ fromJSON(steps.meta.outputs.json).tags[0] }}

  deploy-prod:
    needs: [build-push]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.REMOTE_HOST }}
          username: ${{ secrets.REMOTE_USER }}
          key: ${{ secrets.REMOTE_SSH_KEY }}
          script: |
            docker pull ghcr.io/${{ github.repository }}/my-app:latest
            docker compose -f /srv/my-app/docker-compose.yml up -d
```
{% endraw %}

---

## 14. 운영 팁 요약

| 항목 | 권장 사항 |
|---|---|
| 이미지 크기/속도 | 멀티스테이지 + distroless + 캐시 GHA |
| 보안 | nonroot, SBOM, Trivy, Cosign |
| 태그 | latest + semver + sha |
| 배포 | SSH/Watchtower/Portainer/K8s(Argo CD·Flux) |
| 승인 | Environments + 보호 규칙 |
| 롤백 | `workflow_dispatch` 입력으로 태그 핀 고정 |
| 비용/속도 | matrix 단일화, 캐시 TTL 관리, 불필요 아키텍처 빼기 |

---

## 15. 검증/문서화 체크리스트

- 빌드 로그와 SBOM 아티팩트 보관
- 취약점 임계값 정책 세우기(예: CRITICAL 발견 시 실패)
- 릴리스 노트 자동 생성/첨부(옵션)
- `.dockerignore`/`compose`의 민감정보 누락 확인
- **비밀 정보는 반드시 Secrets** 사용(절대 커밋 금지)

---

## 16. 빠른 시작(최소 버전)

{% raw %}
```yaml
# .github/workflows/docker-publish.yml
name: docker-publish
on: { push: { branches: [ "main" ] } }

jobs:
  build-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/my-app:latest
```
{% endraw %}

---

## 17. 참고 액션/문서 모음

- `actions/checkout`, `docker/login-action`, `docker/build-push-action`, `docker/metadata-action`
- `docker/setup-qemu-action`, `docker/setup-buildx-action`
- `anchore/sbom-action`, `aquasecurity/trivy-action`, `sigstore/cosign-installer`
- GitHub Actions: https://docs.github.com/actions
- Docker Hub: https://hub.docker.com/
- GHCR: https://docs.github.com/packages/working-with-a-github-packages-registry/working-with-the-container-registry
- Watchtower: https://containrrr.dev/watchtower

---

## 결론

초안에서 제시한 **Docker Hub 로그인 → 빌드 → 푸시 → 서버 배포** 흐름을, 위와 같이 **멀티 아키텍처, 메타데이터/태깅 자동화, 캐시 최적화, SBOM/취약점/서명, 승인 기반 배포, 롤백**까지 확장하면, **재현 가능한 안전한 파이프라인**으로 성장시킬 수 있다. 작은 프로젝트는 최소 워크플로로 시작하고, 운영에서 요구되는 요소를 **단계적으로** 추가해 나가면 됩니다.