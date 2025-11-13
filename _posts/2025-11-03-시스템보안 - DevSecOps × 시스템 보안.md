---
layout: post
title: 시스템보안 - DevSecOps × 시스템 보안
date: 2025-11-03 14:30:23 +0900
category: 시스템보안
---
# 17. DevSecOps × 시스템 보안

## 17.1 CI/CD에서 정적·동적·Fuzz·SCA 자동화

### 17.1.1 아키텍처 개요

```
[Developer] --(pre-commit, local)-->
[SCM: GitHub/GitLab] --(PR/MR)-->
  [CI: Build] --> [SAST (Semgrep/Bandit/Gosec)]
                 --> [Secret Scan (gitleaks)]
                 --> [SCA (Syft+Grype/Trivy)]
                 --> [Lint/Format/Test/Coverage]
                 --> [Fuzz (libFuzzer/Atheris + Sanitizers)]
                 --> [DAST (ZAP baseline) / API scan]
                 --> [SBOM (CycloneDX) / License gate]
                 --> [Container build (BuildKit/SBOM attach)]
                 --> [Image scan (Trivy/Grype)]
                 --> [cosign sign/attest (SLSA/in-toto)]
                 --> [Policy gate (OPA/Rego) as CI step]
  [CD: Staging] --> [K8s Admission: Gatekeeper policies]
                 --> [Runtime sensor (Falco/eBPF), log to SIEM]
```

**핵심 원칙**
- **Shift-left**: 가능한 빨리 실패(개발자 로컬·PR 단계).
- **정책을 코드화**: Rego/Sigma/semgrep-rules/SLSA policy 등.
- **증빙 아카이브**: SBOM, 서명, 빌드 메타데이터, 테스트 리포트(아티팩트).
- **품질 게이트**: *“Fail the build”* 기준을 수치화(예: High 취약 ≥1 → 실패).

---

### 17.1.2 로컬 프리훅(pre-commit) — 첫 방어선

**.pre-commit-config.yaml (발췌)**
- 포맷터/린터 + 비밀 스캔 + 기본 Semgrep
```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.8.0
    hooks: [{id: black}]
  - repo: https://github.com/PyCQA/isort
    rev: 5.13.2
    hooks: [{id: isort}]
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
        args: ["detect", "--redact", "--no-banner"]
  - repo: https://github.com/returntocorp/semgrep
    rev: v1.83.0
    hooks:
      - id: semgrep
        args: ["--config", "p/ci","--error","--metrics=off"]
```

> 개발 머신/PR에서 **비밀 유출·고위험 패턴**을 즉시 차단하여 CI 비용을 낮춥니다.

---

### 17.1.3 GitHub Actions: SAST/SCA/Secrets/Fuzz/DAST/SBOM/Sign 일괄 파이프라인

**.github/workflows/sec-ci.yml**
```yaml
name: security-ci
on:
  pull_request:
  push:
    branches: [ main ]

jobs:
  sast-secret-sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Python deps 예시(언어별로 교체)
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: pip install -r requirements.txt || true

      # 1) Secrets
      - name: gitleaks
        uses: gitleaks/gitleaks-action@v2
        with:
          args: detect --redact --no-banner --exit-code 1

      # 2) SAST (Semgrep)
      - name: semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/ci
          generateSarif: true
          publishToken: ''   # 사내 서버면 토큰
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with: { sarif_file: semgrep.sarif }

      # 3) SCA + SBOM (Syft+Grype or Trivy)
      - name: SBOM (CycloneDX via Syft)
        run: |
          curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b . v1.0.0
          ./syft packages dir:. -o cyclonedx-json > sbom.cdx.json
      - name: SCA (Grype)
        run: |
          curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b . v0.76.0
          ./grype sbom:sbom.cdx.json --fail-on high --add-cpes-if-none

  build-and-fuzz:
    runs-on: ubuntu-latest
    needs: [sast-secret-sca]
    steps:
      - uses: actions/checkout@v4
      - name: Build app (example)
        run: |
          make build  # 또는 docker buildx bake
      - name: Install clang/llvm for fuzz
        run: sudo apt-get update && sudo apt-get install -y clang lldb lcov
      - name: Build fuzz target (libFuzzer)
        run: |
          clang++ -fsanitize=address,fuzzer -O1 \
            -I include -o fuzz_parser tests/fuzz_parser.cc src/parser.cc
      - name: Run fuzz (short smoke)
        run: ./fuzz_parser -max_total_time=60 -runs=0 || true
      - name: Upload fuzz artifacts
        uses: actions/upload-artifact@v4
        with:
          name: fuzz-crashes
          path: ./crash-*

  dast-zap:
    runs-on: ubuntu-latest
    needs: [build-and-fuzz]
    steps:
      - uses: actions/checkout@v4
      - name: Start service (compose)
        run: docker compose -f docker-compose.yml up -d
      - name: OWASP ZAP baseline
        run: |
          docker run --rm -t --network host \
            ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
            -t http://localhost:8080 -r zap_report.html -m 5 -d
      - uses: actions/upload-artifact@v4
        with: { name: zap-report, path: zap_report.html }

  image-build-scan-sign:
    runs-on: ubuntu-latest
    needs: [dast-zap]
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - name: Set up Buildx
        uses: docker/setup-buildx-action@v3
      - name: Build image + SBOM (BuildKit)
        run: |
          docker buildx build --sbom=true --provenance=true \
            -t ghcr.io/org/app:${{ github.sha }} --load .
      - name: Trivy scan image
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ghcr.io/org/app:${{ github.sha }}
          format: 'table'
          exit-code: '1'
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'
      - name: Cosign sign (keyless, OIDC)
        run: |
          curl -sSfL https://github.com/sigstore/cosign/releases/download/v2.4.0/cosign-linux-amd64 -o cosign && chmod +x cosign
          ./cosign sign ghcr.io/org/app:${{ github.sha }} --yes
      - name: Cosign attest (SBOM/Provenance)
        run: |
          ./cosign attest --predicate sbom.cdx.json --type cyclonedx \
             ghcr.io/org/app:${{ github.sha }} --yes
```

> 포인트:
> - **SAST/Secrets/SCA** → **Fail 기준** 명시.
> - 이미지 **SBOM/Provenance**를 **cosign attest**로 첨부, 레지스트리에서 증빙 가능.
> - DAST는 **Baseline**(패시브/단기)로 MR 단계에 적합, 풀 액티브는 **스테이징**에서 야간/스케줄.

---

### 17.1.4 GitLab CI 변형(요지)

**.gitlab-ci.yml (발췌)**
```yaml
stages: [sast, build, fuzz, dast, package]

sast:
  stage: sast
  image: returntocorp/semgrep
  script:
    - semgrep --config p/ci --error --metrics=off
  artifacts:
    reports:
      sast: semgrep.sarif

sca:
  stage: sast
  image: alpine:3.20
  script:
    - apk add --no-cache curl
    - curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b . v1.0.0
    - ./syft dir:. -o cyclonedx-json > sbom.cdx.json
    - curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b . v0.76.0
    - ./grype sbom:sbom.cdx.json --fail-on high
  artifacts:
    paths: [sbom.cdx.json]

build:
  stage: build
  image: docker:27
  services: [docker:27-dind]
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA --sbom=true --provenance=true .
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - cosign sign $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA --yes
```

---

### 17.1.5 SAST 규칙 커스터마이징(예: Semgrep)

**semgrep-rules/crypto-insecure.yaml**
```yaml
rules:
- id: insecure-rand
  patterns:
    - pattern-either:
      - pattern: random.random(...)
      - pattern: Math.random(...)
  message: "Insecure RNG for crypto use. Use secrets or os.urandom."
  severity: ERROR
  languages: [python, javascript]
```

**CI에서 규칙 번들 사용**
```bash
semgrep --config semgrep-rules/ --error
```

---

### 17.1.6 Fuzzing + Sanitizers (C/C++ libFuzzer, Python Atheris)

**C++ fuzz 타깃**
```cpp
// tests/fuzz_parser.cc
#include <stdint.h>
#include <stddef.h>
#include "parser.hpp"
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
  try { Parser p; p.feed(Data, Size); } catch(...) {}
  return 0;
}
```

**빌드 & 실행**
```bash
clang++ -fsanitize=address,undefined,fuzzer -O1 \
  -I include -o fuzz_parser tests/fuzz_parser.cc src/parser.cc
./fuzz_parser -max_total_time=120
```

**Python (Atheris)**
```python
# tests/fuzz_json.py
import atheris, json, sys
def TestOneInput(data: bytes):
    try:
        s = data.decode("utf-8", "ignore")
        json.loads(s)
    except Exception:
        pass
def main():
    atheris.Setup(sys.argv, TestOneInput)
    atheris.Fuzz()
if __name__ == "__main__":
    main()
```

---

### 17.1.7 DAST (OWASP ZAP Baseline) + API 스캔 힌트

**Docker 명령**
```bash
docker run --rm -t --network host ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t http://app:8080 -r zap.html -m 5 -d
```

**OpenAPI 기반**
```bash
zap-api-scan.py -t http://app:8080/openapi.json -f openapi -r api_zap.html -d
```

> Baseline은 **패시브** 중심. 액티브 스캔은 **스테이징**에서 시간 제한·허용리스트로 운용.

---

### 17.1.8 SBOM/CycloneDX + 라이선스 게이트

**CycloneDX 검사 예(license allowlist)**
```bash
cyclonedx-cli analyze -o license-report.json sbom.cdx.json
jq '.components[] | select(.licenses[]?.license.id | IN("MIT","Apache-2.0") | not)' license-report.json
# 결과가 비어있지 않으면 Fail
```

---

### 17.1.9 공급망: 서명·증명(SLSA, cosign, in-toto)

- **Build provenance**: 빌드 시점, 빌더 ID, 소스 커밋, 파라미터 → **attestation**
- **Keyless(OIDC) 서명**: CI 러너 OIDC 토큰으로 cosign sign/attest
- **정책 게이트**: “서명된 이미지만 배포”, “Provenance 없는 이미지는 거부”

**검증(배포 전)**
```bash
cosign verify ghcr.io/org/app:TAG --certificate-identity-regexp "https://github.com/org/.+"
cosign verify-attestation --type cyclonedx ghcr.io/org/app:TAG
```

---

## 17.2 컨테이너 이미지·런타임 스캔, 정책(OPA/Gatekeeper)

### 17.2.1 Dockerfile 하드닝 베스트 프랙티스

**예시**
```dockerfile
# syntax=docker/dockerfile:1.7
FROM gcr.io/distroless/python3-debian12@sha256:...
# 또는 python:3.11-slim + multi-stage

# (멀티스테이지) 빌드 단계
FROM python:3.11-slim AS build
WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN pip install --no-cache-dir poetry && poetry export -f requirements.txt --output reqs.txt --without-hashes
RUN --mount=type=cache,target=/root/.cache/pip pip wheel -r reqs.txt -w wheels
COPY src/ ./src/
RUN pip wheel ./src -w wheels

# 런타임 단계
FROM gcr.io/distroless/python3-debian12
WORKDIR /app
COPY --from=build /app/wheels /wheels
RUN python -m pip install --no-warn-script-location --no-index --find-links=/wheels app && rm -rf /wheels
USER 65532:65532           # 비루트
ENV PYTHONUNBUFFERED=1
EXPOSE 8080
CMD ["app"]
```

**체크리스트**
- 비루트 `USER`
- 불필요한 툴/셸 제거(디스트롤레스/슬림)
- **읽기 전용 루트** + **tmpfs**(K8s에서)
- healthcheck는 K8s Probe로 대체
- **SBOM/서명** 첨부

---

### 17.2.2 이미지 스캔(Trivy/Grype) — CI와 레지스트리 훅

**CI**: 위 17.1.3 참조 (Fail on HIGH/CRITICAL)
**레지스트리 훅**: Harbor/ACR/ECR 이미지 취약점 스캔 결과를 **정책 게이트**에 반영(예: Gatekeeper가 어노테이션/라벨 기반으로 허용/거부)

---

### 17.2.3 런타임 스캔/탐지 (Falco/eBPF)

**Falco 규칙(발췌)** — *“컨테이너 내 쉘 스폰 금지”*
```yaml
- rule: Terminal shell in container
  desc: Detect shell spawned in a container
  condition: container and proc.name in (bash, sh, zsh)
  output: "Shell spawned in container (user=%user.name container=%container.name proc=%proc.name cmd=%proc.cmdline)"
  priority: WARNING
  tags: [container, mitre_execution]
```

**배포**
```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco -n falco --create-namespace
```

> Falco 이벤트는 **SIEM**으로 수집하여 **정책 위반/침해 지표**와 상관 분석.

---

### 17.2.4 OPA/Gatekeeper로 “진입 시” 정책 강제

**(A) 기본 구성**
```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper gatekeeper/gatekeeper -n gatekeeper-system --create-namespace
```

**(B) CapDrop/Root 금지/ReadOnlyRootFS 요구 — Rego 템플릿**

1) **ConstraintTemplate (Rego 포함)** — `k8spsp-secure-pod.yaml`
```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8spspsecurepod
spec:
  crd:
    spec:
      names:
        kind: K8sPspSecurePod
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spspsecurepod
        violation[{"msg": msg}] {
          input.review.kind.kind == "Pod"
          c := input.review.object.spec.containers[_]

          # 1) root 금지
          not c.securityContext.runAsNonRoot
          msg := sprintf("container %s must set runAsNonRoot: true", [c.name])
        }
        violation[{"msg": msg}] {
          input.review.kind.kind == "Pod"
          c := input.review.object.spec.containers[_]
          sc := c.securityContext
          not sc.readOnlyRootFilesystem
          msg := sprintf("container %s must set readOnlyRootFilesystem: true", [c.name])
        }
        violation[{"msg": msg}] {
          input.review.kind.kind == "Pod"
          c := input.review.object.spec.containers[_]
          # 3) Capabilities drop: ALL (또는 허용 리스트)
          not c.securityContext.capabilities.drop[_] == "ALL"
          msg := sprintf("container %s must drop ALL capabilities", [c.name])
        }
```

2) **Constraint** — 네임스페이스별 적용
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPspSecurePod
metadata:
  name: ns-default-secure-pod
spec:
  match:
    namespaces: ["default","staging"]
```

> 추가로 **hostNetwork/hostPID/privileged** 금지, **SELinux/AppArmor profile**, **seccompProfile** 요구 정책을 단계적으로 확장.

---

### 17.2.5 이미지만 허용(서명·프로비넌스) — OPA 정책

**(A) 이미지 서명 검증 (예: cosign 어노테이션 요구)**
- Admission 시 이미지 메타데이터 또는 **증명 어노테이션** 존재 여부 확인(실전에서는 **policy-controller**/cosign-policy-controller 고려).

**예시 ConstraintTemplate**
```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8simagerequireannotations
spec:
  crd:
    spec:
      names:
        kind: K8sImageRequireAnnotations
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8simagerequireannotations
      allowed := {"cosign.sigstore.dev/status": "verified"}
      violation[{"msg": msg}] {
        input.review.kind.kind == "Pod"
        ann := input.review.object.metadata.annotations
        not ann["cosign.sigstore.dev/status"] == "verified"
        msg := "image must be cosign verified (annotation missing)"
      }
```

**Constraint**
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sImageRequireAnnotations
metadata:
  name: require-cosign-verified
spec:
  match:
    kinds: [{apiGroups: [""], kinds: ["Pod"]}]
```

> 실전에서는 **Sigstore policy-controller** 또는 **Kyverno verifyImages** 같이 서명과 정책을 **자동 검증**하는 컨트롤러 사용을 권장.

---

### 17.2.6 배포 전 CI 수준의 OPA 검증(shift-left)

**Conftest로 쿠버네티스 매니페스트 사전 검사**
```bash
# 정책 디렉터리(policy/)에 Rego 작성 후:
conftest test k8s-manifests/ -p policy/ --output table
```

**GitHub Actions 단계**
```yaml
- name: Conftest policy check
  uses: instrumenta/conftest-action@v0.3.0
  with:
    files: k8s-manifests
    policy: policy
```

> “Admission에서 거절될 리소스는 **CI에서 미리 실패**”하게 만들어 **배포 속도/안정성**을 높입니다.

---

### 17.2.7 읽기 전용 루트/임시 디렉터리 마운트 등 PodSecurity 강화 예

**Deployment (발췌)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: app }
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        fsGroup: 2000
      containers:
        - name: web
          image: ghcr.io/org/app:sha-...
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
            seccompProfile: { type: RuntimeDefault }
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: { medium: Memory }
```

**Gatekeeper로 강제**: 위 항목들이 없으면 **거절**.

---

### 17.2.8 운영 대시보드/게이트 기준(추천)

- **SAST**: ERROR ≥ 1 → Fail, WARNING는 PR 코멘트/TODO
- **Secrets**: 발견 시 **무조건 Fail**(허위양성은 allowlist, 만료일)
- **SCA**: **High/CRITICAL ≥ 1 Fail**, OS/라이브러리 분리 기준
- **Fuzz/Sanitizers**: 크래시 발견 → Fail, 커버리지 수치 추적
- **DAST**: High/Confirmed ≥ 1 Fail, Low/Medium은 티켓
- **SBOM**: 반드시 첨부, **금지 라이선스 발견** → Fail
- **Image Scan**: 배포 태그는 HIGH/CRITICAL 0 조건
- **Sign/Provenance**: 미첨부 → 배포 차단
- **OPA/Gatekeeper**: *“보안 프로필 미설정/루트/권능 미삭제/서명 미검증”* → 거절

---

## 17.3 (옵션) 예제 저장소 구조 제안

```
repo/
 ├─ src/                    # 앱 코드
 ├─ tests/                  # 단위/통합/퍼즈 타깃
 ├─ docker/                 # Dockerfile, compose
 ├─ k8s-manifests/          # 배포 YAML
 ├─ policy/                 # OPA Rego, Gatekeeper 템플릿
 ├─ sigma/                  # 탐지 규칙(참고)
 ├─ .github/workflows/      # CI 파이프라인
 ├─ semgrep-rules/          # SAST 커스텀 룰
 ├─ .pre-commit-config.yaml
 └─ docs/SECURITY.md        # 품질 게이트, 예외/만료 정책
```

---

## 17.4 트러블슈팅 & FAQ

**Q. 오탐 때문에 빌드가 자주 깨져요.**
A. “**PR 코멘트(정보)** → **Warning(점수)** → **Fail(치명)** 3단계로 점진. 허용 리스트에는 **만료일**을 부여하세요.

**Q. 취약점 CVSS 점수 기준만으로 Fail 하기 어려워요.**
A. *실제 경로 노출/공격 표면* (인터넷 노출, 권한, 데이터 민감도)을 가중치로 반영하여 **Risk Score** 기반으로 Fail.

**Q. Gatekeeper 정책이 지나치게 엄격해요.**
A. `dryrun: true`로 시범 운용 → 위반 리포트 수집 → 예외/템플릿 조정 후 enforce.

**Q. 서명/증명 검증이 복잡합니다.**
A. Sigstore **policy-controller** 또는 Kyverno **verifyImages** 사용으로 단순화. **OIDC**를 통한 **Keyless**가 비밀 관리 부담을 낮춥니다.

---

## 17.5 체크리스트(요약)

- [ ] **pre-commit**(gitleaks+semgrep)
- [ ] **CI: SAST/Secrets/SCA** + **Fail 기준**
- [ ] **Fuzz + Sanitizers** Smoke
- [ ] **DAST Baseline**(PR/스테이징)
- [ ] **SBOM(CycloneDX)** 생성 & **라이선스 게이트**
- [ ] **이미지 스캔**(High/Cr critical=0)
- [ ] **cosign sign/attest**(Provenance/SBOM)
- [ ] **OPA/Conftest**로 매니페스트 사전 검증
- [ ] **Gatekeeper**: runAsNonRoot/CapDrop/ROFS/verifyImage 정책
- [ ] **Falco** 런타임 이상 탐지 + **SIEM 연계**

---

# 결론

- DevSecOps의 핵심은 **보안 활동을 코드/정책/파이프라인**으로 구조화해 **자동으로 가드레일을 넘지 못하게** 만드는 것입니다.
- **Shift-left → 게이트 → 런타임 감시**의 3단계 설계를 적용하면, 개발 속도를 유지하면서도 **가시성·재현성·감사 가능성**을 확보할 수 있습니다.
- 본문 예제(YAML/Rego/Dockerfile/코드)를 템플릿으로 삼아, 조직 표준의 **Fail 기준/예외 만료/증빙 아카이브**를 명문화하면 **지속 가능한 보안 품질**을 확보할 수 있습니다.
