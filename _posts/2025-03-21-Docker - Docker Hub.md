---
layout: post
title: Docker - Docker Hub
date: 2025-03-21 19:20:23 +0900
category: Docker
---
# Docker Hub

## 0. Docker Hub란?
- Docker Inc.가 운영하는 **컨테이너 이미지 레지스트리**.
- `docker pull` / `docker push`의 **기본 원격 저장소**로 널리 사용.
- **개인/조직(Organization)** 저장소, **공개/비공개(Private)** 리포지토리, **자동 빌드/웹훅/팀 권한** 등을 제공.

---

## 1. 가입·로그인과 인증 토큰

### 1.1 가입
- https://hub.docker.com → ID 생성(고유), 이메일 인증, 2FA 권장.

### 1.2 CLI 로그인
```bash
docker login
# Username: <Docker ID>
# Password: <비밀번호 또는 Personal Access Token>
```
- 최초 로그인 시 `~/.docker/config.json`에 **Base64 인코딩된 자격**이 저장된다.
- **권장**: 비밀번호 대신 **Personal Access Token(PAT)** 을 생성해 사용.

### 1.3 PAT(개인 액세스 토큰) 사용
1) Hub 웹 → Settings → Security → New Access Token  
2) CI/CD나 서버에서는 다음처럼 사용:
```bash
# 비대화식 로그인(예: CI)
echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
```
- 토큰은 **권한/기한**을 세분화하여 발급·폐기할 수 있어 안전하다.

### 1.4 로그아웃
```bash
docker logout
```

---

## 2. 이미지 검색과 공식/검증 이미지 판별

### 2.1 검색
```bash
docker search nginx
```
출력의 주요 열:
- **OFFICIAL**: Docker가 관리하는 **공식 이미지** 여부
- **STARS**: 커뮤니티 신뢰 지표
- **DESCRIPTION**: 요약

### 2.2 레포지토리 네임스페이스
- `nginx` → `library/nginx` (공식 이미지 네임스페이스)
- 사용자/조직 소유: `<namespace>/<repo>:<tag>` (예: `johndoe/myapp:1.0.0`)

---

## 3. Pull: 이미지 내려받기

```bash
docker pull ubuntu:22.04
docker pull nginx                 # 태그 생략 시 :latest
```

### 3.1 다이제스트로 고정(Pin by digest)
```bash
docker pull nginx@sha256:<digest>
```
- **재현성** 확보(같은 해시=같은 콘텐츠) — 운영 배포에 권장.

---

## 4. Build → Tag → Push 의 표준 루틴

### 4.1 로컬 빌드
```bash
docker build -t myapp:dev .
```

### 4.2 태깅(네임스페이스 포함)
```bash
docker tag myapp:dev johndoe/myapp:1.0.0
docker tag myapp:dev johndoe/myapp:latest   # latest는 '가장 최신'이란 보장이 아님. 포인터 개념.
```

### 4.3 푸시
```bash
docker push johndoe/myapp:1.0.0
docker push johndoe/myapp:latest
```

### 4.4 실전 태그 묶음(버전 + 커밋)
```bash
APP=johndoe/myapp
VER=1.4.2
SHA=$(git rev-parse --short=7 HEAD)

docker build -t $APP:$VER -t $APP:$VER-$SHA -t $APP:latest .
docker push $APP:$VER
docker push $APP:$VER-$SHA
docker push $APP:latest
```

---

## 5. 버전 전략(태그 체계)와 운영 팁

| 태그 | 용도 | 비고 |
|---|---|---|
| `vX.Y.Z` | SemVer 릴리스 | 실제 배포 추적 기본 단위 |
| `vX.Y` | 마이너 최신 포인터 | 선택 사항(팀 문화에 따라) |
| `latest` | 가장 최근 빌드 포인터 | 운영에 **직접 핀**으로 쓰지 않는 것을 권장 |
| `commit-sha` | 산출물 추적 | 디버그/롤백 시 유용 |
| `stable` | 검증 통과판 포인터 | CD 단계에서 갱신 |

- **원칙**: 프로덕션은 **명시 태그 또는 다이제스트**로 고정.

---

## 6. 멀티 아키텍처(amd64/arm64) 이미지 푸시

### 6.1 buildx 활성화
```bash
docker buildx create --use
```

### 6.2 멀티플랫폼 빌드·푸시
```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t johndoe/myapp:1.0.0 \
  -t johndoe/myapp:latest \
  --push .
```
- Hub에는 **단일 레포 태그**로 올라가지만 내부적으로 **멀티 아키 매니페스트**가 생성되어, 클라이언트 아키에 맞는 이미지를 받는다.

---

## 7. 개인/조직 레포지토리, 권한, 팀 협업

### 7.1 레포지토리 생성
- Hub 웹 → Repositories → **Create Repository**
- 이름, 설명, 공개/비공개 선택.

### 7.2 Organization/Team
- Organization을 만들고 멤버 초대 후, **팀 권한**(Read/Write/Admin)을 부여.
- CI에서 사용할 **Org 토큰**을 별도 발급하여 회수/교체를 쉽게 한다.

---

## 8. 자동 빌드/자동 푸시(옵션 다양화)

### 8.1 Docker Hub 내장 자동 빌드(연동형)
- Hub → Repository → **Builds** → GitHub/GitLab 연결
- 브랜치/경로/빌드 컨텍스트/태그 규칙 설정
- 코드 푸시 시 Hub가 자동 빌드·푸시 수행

### 8.2 GitHub Actions(CI)로 직접 빌드·푸시(권장 예시)
```yaml
name: docker-publish

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Set up QEMU (multi-arch)
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push (amd64+arm64)
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            johndoe/myapp:latest
            johndoe/myapp:${{ github.sha }}
          platforms: linux/amd64,linux/arm64
```
- 비밀번호 대신 **토큰**을 Secrets에 저장해 사용.
- 필요 시 `cache-from`/`cache-to`로 빌드 캐시 최적화.

---

## 9. 프라이빗 레포지토리 사용

### 9.1 Pull(인증 필요)
```bash
docker login
docker pull johndoe/private-app:1.2.0
```

### 9.2 팀원 접근
- 해당 레포지토리가 포함된 **Org/team에 읽기 권한**이 있어야 한다.

---

## 10. 이미지 관리(목록, 삭제, 정리)

### 10.1 로컬
```bash
docker images
docker image rm johndoe/myapp:oldtag
docker system prune -f
```

### 10.2 Hub 웹
- Repo → **Tags** 탭에서 태그별 삭제/설명/Immutable(고정) 옵션 등 관리.
- 오래된 태그 **보존 정책**(수동/자동 스크립트)으로 비용·혼잡을 줄인다.

---

## 11. 속도·용량·비용 최적화(실전 체크리스트)

| 주제 | 핵심 팁 |
|---|---|
| 베이스 이미지 | `-slim`, `-alpine`, 또는 **distroless**로 공격면·용량 축소 |
| 멀티스테이지 | 빌더와 런타임 분리 → 최종 이미지 최소화 |
| .dockerignore | 빌드 컨텍스트 축소(`.git`, `node_modules`, `__pycache__` 등 제외) |
| 레이어 최적화 | `RUN` 결합, 불필요 파일 제거, `pip --no-cache-dir` |
| 캐시 | BuildKit 캐시, 레이어 불변성 유지로 **CI 속도 향상** |
| 다이제스트 핀 | 운영에서 태그 대신 **digest** 사용(재현성↑) |

---

## 12. 보안: 토큰·서명·취약점 스캔

### 12.1 비밀 관리
- 비밀번호 대신 **PAT** 사용, **2FA** 활성화.
- CI에서는 `--password-stdin`을 사용, 작업 후 `docker logout` 수행.

### 12.2 이미지 서명(예: cosign)
```bash
cosign sign johndoe/myapp:1.0.0
cosign verify johndoe/myapp:1.0.0
```
- 서명 검증을 배포 파이프라인에 **게이트**로 추가.

### 12.3 취약점 스캔(사전 차단)
- Trivy(로컬/CI), Docker Scout(Hub/CLI/데스크톱 연동).
```bash
trivy image johndoe/myapp:1.0.0
docker scout quickview johndoe/myapp:1.0.0
```
- **CRITICAL/HIGH 발견 시 실패**하도록 CI 정책을 둔다.

---

## 13. Pull Rate Limit/가용성 주의

- Hub는 **풀 레이트 제한**을 적용한다(익명 < 로그인 < 유료 플랜).  
- CI/CD·대규모 배포 환경은 **로그인 상태**로 풀을 수행하고, 캐시/프록시/미러(사설 레지스트리)를 고려한다.

---

## 14. 레지스트리 전환/병행(GHCR 등)

- 동일 파이프라인에서 **여러 레지스트리**에 푸시 가능:
```bash
# Docker Hub
docker tag myapp:1.0.0 johndoe/myapp:1.0.0
docker push johndoe/myapp:1.0.0

# GHCR
docker tag myapp:1.0.0 ghcr.io/johndoe/myapp:1.0.0
echo "$GHCR_TOKEN" | docker login ghcr.io -u johndoe --password-stdin
docker push ghcr.io/johndoe/myapp:1.0.0
```
- 멀티 레지스트리 전략으로 **가용성/속도/비용**을 최적화할 수 있다.

---

## 15. 웹훅/자동 재배포(옵션)

- Hub 레포지토리 **Webhooks** → 새 이미지 푸시 시 URL 호출.
- 서버 측에서는 Watchtower/Portainer/자체 스크립트가 훅을 받아 `docker pull && restart` 수행.

예: 간단 Webhook 수신 스크립트(의사 코드)
```bash
#!/usr/bin/env bash
# /opt/hooks/deploy-myapp.sh
set -e
IMAGE="johndoe/myapp:latest"

docker pull "$IMAGE"
docker stop myapp || true
docker rm myapp || true
docker run -d --name myapp -p 80:8080 "$IMAGE"
```

---

## 16. 실전 시나리오: 작은 Flask 앱을 Hub로 배포

### 16.1 프로젝트
```
myapp/
├─ app.py
├─ requirements.txt
└─ Dockerfile
```

`app.py`
```python
from flask import Flask
app = Flask(__name__)
@app.get("/")
def hello():
    return {"message": "Hello Hub"}
```

`requirements.txt`
```
Flask==2.3.3
```

`Dockerfile`
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["python", "app.py"]
```

### 16.2 빌드·푸시
```bash
APP=johndoe/myapp
docker build -t $APP:1.0.0 -t $APP:latest .
docker login
docker push $APP:1.0.0
docker push $APP:latest
```

### 16.3 다른 호스트에서 실행
```bash
docker login
docker pull johndoe/myapp:latest
docker run -d --name myapp -p 80:8080 johndoe/myapp:latest
curl localhost
# {"message":"Hello Hub"}
```

---

## 17. 운영 체크리스트(요약)

- 태그 전략: **SemVer + latest 포인터 + 커밋 해시**(선택).
- 운영 고정: 태그 대신 **다이제스트** 사용 권장.
- 빌드: 멀티스테이지 + `.dockerignore` + 캐시 최적화.
- 멀티아키: `buildx`로 amd64/arm64 **동시 배포**.
- 보안: PAT, 2FA, 서명(cosign), 스캔(Trivy/Scout), 최소 이미지.
- 팀 협업: Org/Team 권한, 웹훅·CI로 **자동화**.
- 정리: 오래된 태그/이미지 **주기 삭제**, 저장소 비용 관리.
- 레이트 리밋: **로그인 상태** 풀, 캐시/미러/사설 레지스트리 고려.

---

## 부록 A. 자주 쓰는 명령 모음

```bash
# 로그인/로그아웃
docker login
docker logout

# 빌드/태그/푸시
docker build -t johndoe/myapp:1.0.0 .
docker tag johndoe/myapp:1.0.0 johndoe/myapp:latest
docker push johndoe/myapp:1.0.0
docker push johndoe/myapp:latest

# 멀티아키(amd64+arm64)
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 \
  -t johndoe/myapp:1.0.0 --push .

# 취약점 스캔
trivy image johndoe/myapp:1.0.0
docker scout quickview johndoe/myapp:1.0.0

# 다이제스트 확인
docker inspect --format='{{index .RepoDigests 0}}' johndoe/myapp:1.0.0
```

---

## 부록 B. 수학적 관점의 태그 안정성 메모(선택 읽기)

운영환경에서 재현성 \(R\)을 높인다는 것은, 동일 태그가 **다른 시점**에 다른 이미지를 가리킬 확률 \(p\)를 낮추는 문제와 같다.  
다이제스트 핀을 쓰면 \(p \approx 0\)로 수렴한다. 이를 간단히 쓰면:

$$
R = 1 - p(\text{태그 변동}) \quad\text{이며}\quad
\text{digest pin} \Rightarrow p \to 0
$$

즉, **다이제스트로 고정**하면 같은 해시=같은 바이트스트림이므로 재현성이 최대화된다.

---

## 참고 링크
- Docker Hub: https://hub.docker.com/
- Docker Hub Docs: https://docs.docker.com/docker-hub/
- Docker CLI: https://docs.docker.com/engine/reference/commandline/cli/
- Docker Buildx: https://docs.docker.com/build/buildx/
- Docker Scout: https://docs.docker.com/scout/
- Trivy: https://aquasecurity.github.io/trivy/
- Cosign: https://github.com/sigstore/cosign