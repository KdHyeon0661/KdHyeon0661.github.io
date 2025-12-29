---
layout: post
title: Docker - Trivy & Docker Scout
date: 2025-03-10 19:20:23 +0900
category: Docker
---
# Docker 이미지 보안 스캔 도구 완벽 가이드: Trivy와 Docker Scout

## 스캐너가 해결하는 문제와 위협 모델

### 보안 스캔이 필요한 이유

컨테이너 이미지는 레이어 구조로 구성되어 있으며, 각 레이어에 포함된 OS 패키지, 언어 런타임, 라이브러리에는 알려진 취약점(CVE)이 누적될 수 있습니다. 운영 중에 새로운 CVE가 공개되면, 빌드 시점에는 존재하지 않았던 취약점이 런타임 환경에 생기는 "취약점 드리프트" 현상이 발생합니다. 또한 규제 준수와 감사 대응을 위해 SBOM(Software Bill of Materials) 생성 의무화 및 배포 전 취약점에 대한 정책 기반 게이트가 필요합니다.

### 스캐너의 핵심 기능

스캐너는 이미지 내 패키지 인벤토리(apk, apt, rpm, pip, npm, gem 등)를 추출하고, 공개 CVE 데이터베이스(NVD 등)와 매칭하여 취약점의 심각도, 고정 버전, 영향 범위를 산정합니다. 정책 엔진을 통해 파이프라인의 통과/차단을 결정하며, CycloneDX 또는 SPDX 형식의 SBOM을 생성하여 소프트웨어 공급망의 가시성과 추적성을 제공합니다.

---

## Trivy — 범용 오픈소스 스캐너

### 개요

Trivy는 가볍고 빠른 오픈소스 멀티 타깃 스캐너입니다. 컨테이너 이미지, 파일 시스템, Git 리포지토리, IaC 설정 파일(Dockerfile, Terraform, Kubernetes 매니페스트) 등 광범위한 대상에 대한 보안 검사를 지원합니다.

주요 실행 모드는 다음과 같습니다:
- `trivy image`: 컨테이너 이미지 스캔
- `trivy fs`: 로컬 디렉터리 또는 워크스페이스 스캔
- `trivy repo`: 원격 Git 리포지토리 스캔
- `trivy config`: IaC 설정 파일 정적 분석
- `trivy sbom`: SBOM 생성 및 소비

### 설치 방법

```bash
# macOS (Homebrew)
brew install aquasecurity/trivy/trivy

# Debian/Ubuntu
sudo apt update && sudo apt install -y trivy

# Docker 컨테이너로 실행
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image nginx:alpine
```

### 이미지 스캔 기본 사용법

```bash
# 최신 태그 이미지 스캔
trivy image nginx:latest

# JSON 형식으로 상세 결과 출력 및 저장
trivy image --format json --output trivy-image.json nginx:latest

# CRITICAL 및 HIGH 심각도 취약점 필터링, 발견 시 실패 코드 반환
trivy image --severity CRITICAL,HIGH --exit-code 1 nginx:latest
```

결과 해석 시 주목할 점은 **Installed Version**과 **Fixed Version**을 비교하여 업그레이드로 해결 가능한 취약점인지 판단하는 것입니다. 또한 **Layer** 정보를 통해 어느 Dockerfile 레이어에서 취약점이 유입되었는지 추적할 수 있습니다.

### 파일 시스템, 코드, IaC 스캔

```bash
# 현재 워크스페이스의 의존성, 비밀정보, 민감 데이터, 바이너리 점검
trivy fs .

# Dockerfile 보안 규칙 검사 (EOL 베이스 이미지, 루트 권한 실행, 비효율적 레이어 등)
trivy config Dockerfile

# Terraform 및 Kubernetes 매니페스트 보안 설정 검증
trivy config infra/
```

### CI/CD 파이프라인 연동 (GitHub Actions 예시)

{% raw %}
```yaml
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

주요 옵션 설명:
- `ignore-unfixed`: 아직 수정 버전이 제공되지 않은 취약점은 경고만 출력합니다. 팀 정책에 따라 조정하세요.
- `exit-code`: 스캔 결과에 따라 파이프라인을 실패시킬 기준을 설정합니다.
- `format`: 결과 출력 형식을 지정합니다 (`table`, `json`, `sarif` 등).

### 오탐 및 허용 예외 관리

`.trivyignore` 파일을 생성하여 특정 취약점을 무시할 수 있습니다.
```bash
# .trivyignore 파일 예시
CVE-2023-0001
CVE-2024-12345
```
```bash
trivy image --ignorefile .trivyignore my/app:latest
```
비즈니스 리스크 평가 후 예외를 등록할 때는 만료 기한을 코멘트로 명시하여 주기적으로 재검토하는 것이 좋습니다.

### 오프라인 환경 및 사설 레지스트리 대응

Trivy는 데이터베이스 미러링을 지원하여 인터넷이 차단된 환경에서도 취약점 DB를 업데이트할 수 있습니다. 프록시나 인증이 필요한 사설 레지스트리의 경우 다음과 같이 사용합니다.
```bash
export HTTP_PROXY=http://proxy.local:3128
trivy image --username $REG_USER --password $REG_PASS registry.local/my/app:tag
```

### 성능 및 정확도 향상 팁

- **빌드 캐시 활용**: 빈번한 OS 패키지 업데이트가 발생하는 구간을 최소화하여 이미지 레이어를 효율적으로 구성하면 취약점 수를 줄일 수 있습니다.
- **필요한 스캐너만 선택**: `--scanners vuln,secret,config,license`와 같이 필요한 검사 유형만 지정하여 실행 시간을 단축합니다.
- **병렬 스캔**: CI 환경에서 여러 아키텍처나 태그에 대한 스캔을 병렬로 실행합니다.

### SBOM 생성 및 활용

```bash
# CycloneDX 형식의 SBOM 생성
trivy sbom --format cyclonedx --output sbom.json my/app:latest

# SPDX 형식의 SBOM 생성
trivy sbom --format spdx-json --output sbom-spdx.json my/app:latest
```
생성된 SBOM을 아티팩트로 보관하면 향후 감사 지원이나 사고 대응 시 특정 CVE가 어떤 컴포넌트에 존재하는지 빠르게 질의할 수 있습니다.

---

## Docker Scout — Docker 생태계 통합 분석 도구

### 개요

Docker Scout는 Docker CLI, Hub, Desktop과 긴밀하게 통합된 분석 도구입니다. SBOM 관리, CVE 분석, 이미지 비교, 업그레이드 제안 등의 기능을 강점으로 하며, CI 환경 없이도 개발자가 로컬에서 빠르게 보안 가시성을 확보하기에 적합합니다.

### 기본 사용법

```bash
# 이미지 보안 상태 요약 보기
docker scout quickview nginx:latest

# 취약점 목록 조회
docker scout cves nginx:latest

# SBOM 출력
docker scout sbom nginx:latest

# 두 이미지 간 패키지 및 취약점 차이 비교
docker scout compare my/app:1.2.0 my/app:1.3.0
```

출력 결과에서 **Recommendations** 섹션은 취약점 수를 줄일 수 있는 권장 태그나 버전 업그레이드 안내를 제공합니다. 또한 이미지 간 변화된 레이어를 추적하여 보안 상태가 악화되거나 개선된 지점을 파악할 수 있습니다.

### Docker Desktop 통합

Docker Desktop UI에서는 로컬 이미지에 대한 **Analyze** 기능을 제공하여 CVE, 패키지, 레이어 정보를 시각적으로 확인할 수 있습니다. 이는 팀 온보딩이나 디버깅 단계에서 학습 곡선을 낮추는 데 유용합니다.

---

## Trivy와 Docker Scout 비교 및 선택 가이드

| 기준 | Trivy | Docker Scout |
|------|-------|--------------|
| 라이선스 및 유연성 | 오픈소스, CLI 및 서드파티 CI/CD 도구와 통합에 적합 | Docker 생태계에 최적화, Desktop 및 Hub와 밀접한 통합 |
| 지원 범위 | 이미지, 파일 시스템, 리포지토리, IaC, K8s 등 광범위 | 이미지, SBOM, 비교 분석에 특화 |
| 정책 게이트 | CLI 플래그 및 GitHub Action을 통한 세밀한 제어 가능 | CLI 및 GUI 기반 비교적 단순한 정책 적용 |
| 조직 확장성 | 사내 미러, 오프라인 DB, 다양한 출력 포맷 지원 | Docker 중심 워크로드에 최적화된 환경 |

**결론적으로**, 광범위한 대상에 대한 스캔과 복잡한 정책 기반 자동화가 필요하다면 Trivy를 중심으로 구성하고, 개발자 경험과 시각화, 이미지 비교를 통한 빠른 의사결정이 중요하다면 Docker Scout를 병행하는 것이 효과적입니다.

---

## 파이프라인 구현 표준 템플릿

### 빌드 직후 보안 스캔 (정책 게이트)

```bash
docker build -t registry.example.com/team/app:${GIT_SHA} .
trivy image --severity HIGH,CRITICAL --exit-code 1 registry.example.com/team/app:${GIT_SHA}
```

### 레지스트리 푸시 전 태그 고정 및 SBOM 생성

```bash
docker tag registry.example.com/team/app:${GIT_SHA} registry.example.com/team/app:latest
trivy sbom --format cyclonedx --output sbom-${GIT_SHA}.json registry.example.com/team/app:${GIT_SHA}
docker push registry.example.com/team/app:${GIT_SHA}
docker push registry.example.com/team/app:latest
```

### 배포 전 최종 검증 (서명 및 증명)

```bash
# Cosign을 사용한 이미지 서명 및 SBOM 어테스테이션 첨부
cosign sign --yes registry.example.com/team/app:${GIT_SHA}
cosign attest --yes --predicate sbom-${GIT_SHA}.json --type cyclonedx \
  registry.example.com/team/app:${GIT_SHA}
```

---

## 정책 게이트 고도화: 팀 표준 예시

### 최소 보안 기준

- **CRITICAL** 또는 **HIGH** 심각도 취약점이 존재하면 빌드 실패
- **수정 버전이 존재하는(Fixed Version)** 취약점은 반드시 해결
- **지원 종료(EOL)된 베이스 이미지** 사용 금지

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed=false --exit-code 1 my/app:tag
```

### 환경별 차등 정책

- 개발 환경: 경고만 출력, 빌드 통과
- 스테이징 환경: HIGH 이상 심각도 취약점 발견 시 차단
- 프로덕션 환경: MEDIUM 이상 심각도 취약점 발견 시 차단
- `.trivyignore` 파일을 사용할 경우 만료 일자를 주석으로 명시하여 영구 예외 방지

### 리스크 기반 평가 모델

취약점의 위험도를 정량적으로 평가하기 위해 간단한 수학적 모델을 적용할 수 있습니다. 각 취약점 \(i\)에 심각도 가중치 \(w_i\)를 부여하고, 악용 가능성 또는 네트워크 경계 외부에서의 접근 가능성을 고려한 지표 \(\mathbb{1}\{\text{exploitable}(i)\}\)를 정의합니다. 전체 위험도 \(R\)는 다음과 같이 계산됩니다.

$$
R = \sum_{i=1}^{N} w_i \cdot \mathbb{1}\{\text{exploitable}(i)\}
$$

팀에서 정의한 최대 허용 위험도 \(R_{\text{max}}\)를 기준으로 \(R > R_{\text{max}}\)인 경우 배포를 차단하는 정책을 수립할 수 있습니다.

---

## 문제 유형별 근본 원인 분석 및 개선 방안

| 유형 | 근본 원인 | 개선 방안 |
|------|-----------|-----------|
| OS 패키지 취약점 | 오래된 베이스 이미지 사용 | 최신 베이스 이미지로 업그레이드 (예: `debian:bookworm-slim`), OS 패치 주기화 |
| 언어 라이브러리 취약점 | 구버전 패키지 고정 | 패키지 최소 버전 상향, Semantic Versioning 제약 재검토, 잠금 파일(lock file) 재생성 |
| 불필요한 레이어 | 개발 도구가 프로덕션 이미지에 포함 | 멀티스테이지 빌드 채택, 런타임 전용 이미지로 전환 |
| 과도한 권한 | 기본 `USER root`로 실행 | `USER nonroot` 지정, 파일 시스템 읽기 전용 마운트, seccomp/권한 제한 적용 |
| IaC 설정 오류 | 잘못된 보안 설정 | `trivy config`를 통한 정적 분석, 정책 문서화 및 코드 리뷰 프로세스 강화 |

---

## 실습 데모: 취약한 이미지 생성, 수정, 재스캔

### 취약점을 포함한 Dockerfile 예시

```dockerfile
FROM python:3.11
RUN pip install flask==2.0.0  # 의도적으로 구버전 설치
COPY app.py /app/app.py
WORKDIR /app
CMD ["python","app.py"]
```

```bash
docker build -t demo/vuln:bad .
trivy image demo/vuln:bad
```

### 개선된 Dockerfile (베이스 이미지 및 패키지 업그레이드)

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
스캔 결과를 비교하여 취약점 수 감소, 심각도 완화, 권장 업그레이드가 제대로 반영되었는지 확인합니다.

---

## 성능 최적화를 위한 실용적 팁

- **멀티플랫폼 빌드 대응**: `linux/amd64` 및 `linux/arm64`와 같은 다중 아키텍처 이미지의 경우, 아키텍처별 스캔 작업을 병렬화합니다.
- **빌드 캐시 전략**: 레이어 변경을 최소화하기 위해 애플리케이션 코드(`COPY . .`)보다 의존성 설치(`COPY requirements.txt`, `RUN pip install`)를 먼저 수행하는 것이 좋습니다.
- **단계적 스캔 전략**: 파이프라인 단계별로 스캐너 범위를 구분하여 실행합니다. PR 검토 시에는 빠른 피드백을 위한 요약 스캔을, 머지 전에는 정밀 스캔을 수행합니다.

---

## 자주 묻는 질문 (FAQ)

**Q1. 아직 수정 버전이 제공되지 않은 "Unfixed" 취약점은 어떻게 처리해야 하나요?**
A. `ignore-unfixed=true` 옵션을 사용하여 게이트에서는 완화하되, 리포트에는 계속 추적 정보로 남겨둡니다. 동시에 공급사의 패치 일정을 모니터링하거나 대체 라이브러리 검토를 진행합니다.

**Q2. Alpine Linux 기반 이미지가 취약점이 적게 보고되는데 항상 더 안전한 선택인가요?**
A. 취약점 데이터베이스의 커버리지와 매칭 기준, 패키지의 세분화 정도에 따라 차이가 있을 수 있습니다. 이미지 크기와 보안성 뿐만 아니라 애플리케이션 호환성과 운영성을 종합적으로 고려하여 선택해야 합니다.

**Q3. 이미지 크기가 너무 커서 스캔 속도가 느립니다.**
A. 슬림 베이스 이미지 사용과 멀티스테이지 빌드를 통해 이미지 크기 자체를 줄이는 것이 근본적인 해결책입니다. 또한 PR 단계에서는 핵심 취약점만 검사하는 요약 스캔을, 야간 배치 작업에서는 전체 정밀 스캔을 수행하는 전략을 고려해보세요.

---

## 핵심 명령어 치트시트

```bash
# Trivy: 이미지 스캔 및 정책 게이트
trivy image --severity HIGH,CRITICAL --exit-code 1 my/app:tag

# Trivy: IaC 구성 점검
trivy config .

# Trivy: SBOM 생성
trivy sbom --format cyclonedx --output sbom.json my/app:tag

# Trivy: 예외 파일을 사용한 스캔
trivy image --ignorefile .trivyignore my/app:tag

# Docker Scout: 빠른 상태 점검
docker scout quickview my/app:tag

# Docker Scout: 이미지 비교
docker scout compare my/app:old my/app:new

# Docker Scout: SBOM 출력
docker scout sbom my/app:tag
```

---

## 마무리: 효과적인 컨테이너 보안을 위한 접근법

Trivy는 오픈소스 도구로서 컨테이너 이미지부터 IaC 설정까지 광범위한 대상에 대한 심층적인 보안 스캔과 정책 기반 자동화를 가능하게 합니다. 반면 Docker Scout는 Docker 생태계에 특화되어 개발자 친화적인 인터페이스와 직관적인 비교 분석을 제공합니다.

실제 운영 환경에서는 두 도구를 상호 보완적으로 활용하는 것이 효과적입니다. "승인된 베이스 이미지 사용 → 자동화된 스캔 → 정책 게이트 적용 → SBOM 및 서명으로 증빙 → 주기적인 재평가"라는 선순환 프로세스를 구축하는 것이 핵심입니다.

가장 중요한 것은 한 번에 모든 취약점을 제거하려는 압박보다는, **각 릴리즈마다 지속적으로 취약점 수를 줄여나가는 체계적인 습관**을 문화로 정착시키는 것입니다. 도구는 이 문화를 지원하는 수단일 뿐, 궁극적인 목표는 더 안전한 소프트웨어 공급망을 구축하는 데 있습니다.