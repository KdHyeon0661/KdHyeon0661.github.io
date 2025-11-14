---
layout: post
title: Docker - Trivy & Docker Scout
date: 2025-03-10 19:20:23 +0900
category: Docker
---
# Docker 이미지 보안 스캔 도구 완전 정복: Trivy & Docker Scout

## 스캐너가 해결하는 문제와 위협 모델

### 왜 스캔이 필요한가

- **이미지 레이어** 속 OS 패키지/언어 런타임/라이브러리에는 CVE가 누적된다.
- 운영 중 **새 CVE가 공개**되면, 빌드 시점엔 없던 취약점이 런타임에 생긴다(“취약점 드리프트”).
- 규제/감사 대응: SBOM 의무화, 배포 전 취약점 **정책 게이트** 필요.

### 스캐너의 관점

- **패키지 인벤토리**(apk, apt, rpm, pip, npm, gem …)를 추출
- **CVE DB**(NVD 등)와 매칭 → 심각도(Severity), 고정 버전(Fix Version), 영향 범위를 산정
- **정책 엔진**으로 파이프라인 Fail/Pass 결정
- **SBOM**(CycloneDX/SPDX)을 생성해 가시성과 추적성을 제공

---

## Trivy — 범용 오픈소스 스캐너

### 개요

- 오픈소스, 가볍고 빠르며 **이미지·파일시스템·Git 리포·IaC(Dockerfile/Terraform/K8s Manifests)** 등 **멀티 타깃** 지원.
- 실행 모드:
  - `trivy image`: 컨테이너 이미지
  - `trivy fs`: 로컬 디렉터리/워크스페이스
  - `trivy repo`: 원격 Git 리포지토리
  - `trivy config`: IaC 정적 분석(정책 기반)
  - `trivy sbom`: SBOM 생성/소비

### 설치

```bash
# macOS(Homebrew)

brew install aquasecurity/trivy/trivy

# Debian/Ubuntu

sudo apt update && sudo apt install -y trivy

# 컨테이너로 사용(도커 소켓 공유)

docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image nginx:alpine
```

### 이미지 스캔 기초

```bash
# 최신 태그 스캔

trivy image nginx:latest

# 상세 + JSON 결과(머신리더블, CI 아티팩트 보관)

trivy image --format json --output trivy-image.json nginx:latest

# 심각도 필터 + 실패 처리(게이트)

trivy image --severity CRITICAL,HIGH --exit-code 1 nginx:latest
```

출력 해석 포인트
- **Installed** vs **Fixed Version**: 업그레이드하면 사라지는 취약점인지 판단.
- **Layer**: 어느 Dockerfile 레이어에서 유입됐는지 추적.

### 파일/코드/IaC 스캔

```bash
# 워크스페이스의 의존/비밀/민감정보/바이너리 점검

trivy fs .

# Dockerfile 규칙 검사(베이스 이미지 EOL, 루트 실행, 비효율적 레이어 등)

trivy config Dockerfile

# Terraform/K8s 매니페스트의 보안 설정 검증

trivy config infra/
```

### CI/CD 연동(예: GitHub Actions)

{% raw %}
```yaml
# .github/workflows/trivy.yml

name: trivy
on: [push, pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build image
        run: docker build -t ${{ github.repository }}:${{ github.sha }} .

      - name: Trivy Image Scan (fail on HIGH+)
        uses: aquasecurity/trivy-action@v0.19.0
        with:
          image-ref: ${{ github.repository }}:${{ github.sha }}
          format: 'sarif'
          output: 'trivy.sarif'
          severity: 'HIGH,CRITICAL'
          ignore-unfixed: true
          exit-code: '1'

      - name: Upload SARIF to code scanning
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy.sarif
```
{% endraw %}

옵션 요령
- `ignore-unfixed`: 아직 **Fix 미제공 취약점**은 경고만(팀 정책에 맞게).
- `exit-code`: 파이프라인 Fail 기준.
- `format`: `table`, `json`, `sarif` 등.

### 오탐/허용 예외 관리

```bash
# .trivyignore 예시
# 형식: VulnerabilityID [# comment]

CVE-2023-0001
CVE-2024-12345
```
```bash
trivy image --ignorefile .trivyignore my/app:latest
```
- 비즈니스 리스크 평가 후 **만료 기한**을 코멘트로 남겨 주기적으로 재검토.

### 오프라인/사설 레지스트리/프록시

- DB 미러링: 사내 미러를 구성해 인터넷 불가 환경에서 **DB 업데이트** 가능.
- 프락시/자격증명:
```bash
export HTTP_PROXY=http://proxy.local:3128
trivy image --username $REG_USER --password $REG_PASS registry.local/my/app:tag
```

### 성능·정확도 팁

- **빌드 캐시**를 활용해 빈번한 OS 패키지 업데이트 구간을 최소화 → 취약점 수 자체가 줄어든다.
- `--scanners vuln,secret,config,license` 등 필요한 스캐너만 선택해 시간 단축.
- CI에서 **병렬 스캔**(다중 아키텍처/다중 태그)을 지원.

### SBOM 생성·활용

```bash
# CycloneDX SBOM 생성(JSON)

trivy sbom --format cyclonedx --output sbom.json my/app:latest

# SPDX 형식 생성

trivy sbom --format spdx-json --output sbom-spdx.json my/app:latest
```
- SBOM을 아티팩트로 보관 → 감사지원/사고대응(“이 CVE가 어디에 있는가?”) 질의가 쉬워짐.

---

## Docker Scout — Docker 생태계에 긴밀 통합된 분석

### 개요

- Docker CLI/Hub/Desktop과 통합된 스캐너. **SBOM·CVE 분석·이미지 비교·업그레이드 제안**이 장점.
- CI 없이도 개발자가 **로컬에서 빠르게 가시성**을 확보하기 좋다.

### 기본 사용

```bash
# 요약 뷰

docker scout quickview nginx:latest

# 취약점 목록

docker scout cves nginx:latest

# SBOM 출력

docker scout sbom nginx:latest

# 이미지 비교(태그 간 패키지/취약점 차이)

docker scout compare my/app:1.2.0 my/app:1.3.0
```

출력에서 주목할 점
- “**Recommendations**”: 취약점이 줄어드는 **권장 태그/버전** 안내.
- 이미지 간 **변화 레이어**를 추적해 “어디서 악화/개선되었나”를 파악.

### Docker Desktop UI

- 로컬 이미지에서 **Analyze** 실행 → CVE/패키지/레이어 시각화.
- 팀 온보딩/디버깅 단계에서 **학습 곡선이 낮다**는 장점.

---

## Trivy vs Docker Scout — 선택 가이드

| 축 | Trivy | Docker Scout |
|---|---|---|
| 라이선스/유연성 | 오픈소스, CLI·서드파티 CI에 적합 | Docker 사용자 친화, Desktop/Hub와 밀접 |
| 대상 범위 | 이미지·파일·리포·IaC·K8s까지 광범위 | 이미지/SBOM/비교에 특화 |
| 정책 게이트 | CLI 플래그/액션으로 상세 제어 | CLI/GUI 기반, 정책은 비교적 단순 |
| 조직 확장 | 사내 미러, 오프라인, 다양한 포맷 | Docker 중심 워크로드에 최적 |

**결론**
- **광범위 스캔/정책·자동화**가 필요하면 **Trivy**가 중심.
- 개발자 경험/시각화/이미지 비교 기반 **빠른 의사결정**엔 **Scout**를 병행.

---

## 파이프라인 표준 템플릿

### 빌드 직후 스캔(게이트)

```bash
docker build -t registry.example.com/team/app:${GIT_SHA} .
trivy image --severity HIGH,CRITICAL --exit-code 1 registry.example.com/team/app:${GIT_SHA}
```

### 푸시 전 태그 고정 + SBOM 아티팩트

```bash
docker tag registry.example.com/team/app:${GIT_SHA} registry.example.com/team/app:latest
trivy sbom --format cyclonedx --output sbom-${GIT_SHA}.json registry.example.com/team/app:${GIT_SHA}
docker push registry.example.com/team/app:${GIT_SHA}
docker push registry.example.com/team/app:latest
```

### 배포 전 최종 확인(서명/증빙)

```bash
# 예: cosign으로 이미지 서명 및 SBOM 첨부(어테스테이션)

cosign sign --yes registry.example.com/team/app:${GIT_SHA}
cosign attest --yes --predicate sbom-${GIT_SHA}.json --type cyclonedx \
  registry.example.com/team/app:${GIT_SHA}
```

---

## 정책 게이트 고도화(예: 팀 표준)

### 최소 기준

- **CRITICAL/HIGH** 존재하면 Fail
- **Fix Version 존재**하는 이슈는 반드시 해결(유지보수 가능한 범위)
- **EOL 베이스 이미지** 사용 금지

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed=false --exit-code 1 my/app:tag
```

### 환경별 차등

- Dev: 경고(통과), Staging: HIGH 이상 차단, Prod: MEDIUM 이상도 차단
- `.trivyignore`는 **만료 주석**을 강제해 영구 무시 방지

### 수학적 리스크 총합(간단 모델)

취약점 \(i\)의 심각도를 가중치 \(w_i\)로 두고, 총 위험도 \(R\)를
$$
R = \sum_{i=1}^{N} w_i \cdot \mathbb{1}\{\text{exploitable}(i)\}
$$
로 근사한다.
- \(\mathbb{1}\{\cdot\}\): 악용 가능성이 있거나 네트워크 경계 밖에서 도달 가능한 경우 1
- 기준값 \(R_{\text{max}}\)를 정하여 \(R>R_{\text{max}}\)면 배포 차단.

---

## 문제 원인별 개선 접근

| 유형 | 원인 | 개선 |
|---|---|---|
| OS 패키지 기인 | 오래된 베이스 이미지 | 베이스 업그레이드(예: debian:bookworm-slim 최신), OS 패치 주기화 |
| 언어 라이브러리 | 구버전 pip/npm/gem | 최소 버전 상향, SemVer 제한 재검토, 잠금파일 재생성 |
| 불필요 레이어 | dev 툴 포함 | 멀티스테이지 빌드, runtime-only 이미지로 전환 |
| 루트 실행 | 기본 `USER root` | `USER nonroot`, FS read-only, seccomp/cap-drop |
| IaC 취약 | 잘못된 보안 설정 | `trivy config` + 정책 문서화/코드리뷰 |

---

## 실제 데모: 취약 이미지 만들고, 고치고, 재스캔

### 취약한 Dockerfile(예시)

```dockerfile
FROM python:3.11
RUN pip install flask==2.0.0  # 의도적으로 구버전
COPY app.py /app/app.py
WORKDIR /app
CMD ["python","app.py"]
```

```bash
docker build -t demo/vuln:bad .
trivy image demo/vuln:bad
```

### 개선 버전(베이스/패키지 업)

```dockerfile
FROM python:3.11-slim
ENV PIP_NO_CACHE_DIR=1
RUN pip install --no-cache-dir flask==2.3.3
COPY app.py /app/app.py
WORKDIR /app
USER 65532:65532
CMD ["python","app.py"]
```

```bash
docker build -t demo/vuln:good .
trivy image --severity HIGH,CRITICAL --exit-code 1 demo/vuln:good
docker scout quickview demo/vuln:good
docker scout compare demo/vuln:bad demo/vuln:good
```
- 결과: 취약점 수 감소/심각도 완화/권장 업그레이드가 반영되는지 확인.

---

## 조직 운영 체크리스트

1. **기준선(Base image) 카탈로그**: 승인된 베이스 이미지 목록/버전/EOL 관리.
2. **주기 스캔**: 배포 시점 + **정기 리스캔**(새 CVE 반영).
3. **Artifact 관리**: 스캔 리포트(SARIF/JSON), SBOM 보관.
4. **정책 게이트**: 환경별 심각도 기준/예외 승인 프로세스/만료 정책.
5. **개선 루프**: Scout/Trivy 리포트에 따라 업그레이드 PR 자동 생성(봇).
6. **교육**: CVE 읽는 법, 오탐/미해결 취약점 처리 가이드.
7. **감사/증빙**: 서명/어테스테이션(cosign) + 릴리즈 노트 기록.

---

## 성능 최적화 팁

- **멀티플랫폼 빌드**(`linux/amd64,arm64`) 시 아키텍처별 스캔을 병렬화.
- **캐시 재사용**: 레이어 변경을 최소화(의존 설치 전에 `COPY requirements*`).
- **스캐너 옵션 축소**: 파이프라인 단계별로 스캐너 범위를 나눠 실행(빠른 피드백 → 정밀 스캔).

---

## 자주 묻는 질문(FAQ)

**Q1. “Unfixed” 취약점은 어떻게 할까?**
A. `ignore-unfixed=true`로 게이트는 완화하되, 리포트에는 남겨 추적한다. 공급사 패치 일정 공유/교체 대안 검토.

**Q2. Alpine이 취약점이 더 적게 나오는데 무조건 좋은가?**
A. DB/매칭 기준 차이와 패키지 세분성 차이의 영향도 있다. **호환성/운영성**을 함께 고려해야 한다.

**Q3. 이미지가 너무 커서 스캔이 느리다.**
A. 슬림/멀티스테이지로 크기 자체를 줄이고, **정밀 스캔은 야간 배치**, PR 단계는 **요약 스캔**으로 나눈다.

---

## 명령어 치트시트

```bash
# Trivy: 이미지 스캔(게이트)

trivy image --severity HIGH,CRITICAL --exit-code 1 my/app:tag

# Trivy: IaC(Dockerfile/K8s/Terraform) 점검

trivy config .

# Trivy: SBOM 생성

trivy sbom --format cyclonedx --output sbom.json my/app:tag

# Trivy: 오탐/예외

trivy image --ignorefile .trivyignore my/app:tag

# Docker Scout: 요약/비교/SBOM

docker scout quickview my/app:tag
docker scout compare my/app:old my/app:new
docker scout sbom my/app:tag
```

---

## 결론

- **Trivy**는 “넓게·깊게” 스캔하는 오픈소스 표준 도구로, CI/CD 정책 게이트·IaC·SBOM까지 포괄한다.
- **Docker Scout**는 Docker 개발자 워크플로우에 긴밀히 녹아 있어 **빠른 피드백·이미지 비교·권장 업그레이드**가 강점이다.
- 실무에서는 두 도구를 **보완적으로 결합**하고, “**기준 베이스 → 자동 스캔 → 정책 게이트 → 증빙(SBOM/서명) → 주기 리스캔**”의 루프를 자동화하라.
- 중요한 것은 **크게 보는 관성**이 아니라, **매 릴리즈마다 취약점 수를 꾸준히 줄여가는 습관**이다.
