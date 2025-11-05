---
layout: post
title: Spring - CI/CD 파이프라인
date: 2025-10-21 17:25:23 +0900
category: Spring
---
# 21) CI/CD 파이프라인 — GitHub Actions·Argo CD, 품질 게이트, 버전/릴리스/체인지로그 자동화

> 목표: **“푸시 → 테스트 → 이미지빌드/보안스캔 → 매니페스트 업데이트 → 자동 배포/승인 → 프로덕션 릴리스”**까지의 풀파이프라인을 예제와 함께 뼈대부터 완성한다.  
> 기준: Spring Boot 3.x(Gradle), Docker/BuildKit, Kubernetes(+Helm), **GitHub Actions**로 CI, **Argo CD**로 CD, 품질은 **테스트·커버리지·Lint**를 **게이트**로 삼는다. 버전/릴리스/체인지로그는 **Conventional Commits**를 기반으로 자동화한다.

---

## A. 레포 구조 & 브랜치 전략(추천)

```
acme/
├─ app/                        # 어플리케이션(여러 모듈 가능)
│  ├─ build.gradle.kts
│  └─ src/...
├─ deploy/                     # GitOps(Helm 차트 + 환경별 values)
│  ├─ charts/acme-api/
│  │  ├─ Chart.yaml
│  │  ├─ templates/*.yaml
│  │  └─ values.yaml
│  └─ envs/
│     ├─ dev/values.yaml
│     ├─ stage/values.yaml
│     └─ prod/values.yaml
├─ .github/workflows/          # Actions 워크플로
│  ├─ ci.yml                   # PR/브랜치 CI (빌드/테스트/정적분석/보안스캔)
│  ├─ image-publish.yml        # main에 머지되면 이미지 빌드/푸시
│  ├─ gitops-pr.yml            # 환경 레포에 PR 열기(Argo CD 대상)
│  └─ release.yml              # 태그/릴리스/체인지로그 자동화
└─ version.txt                 # (선택) 버전 소스, 또는 git tag/Conventional Commits 사용
```

브랜치:
- **main**: 보호, 릴리스 태그 생성.
- **release/*** : (선택) 핫픽스/메이저 준비.
- **feat/*** : 기능 브랜치 → PR.

환경:
- **dev**: 자동 머지 시 즉시 배포, 승인 없음.
- **stage**: 자동 PR 생성 → 승인 후 배포.
- **prod**: 수동 승인/승격(“promote”).

---

## B. GitHub Actions — CI 파이프라인(품질 게이트 포함)

### B-1. CI 기본(Gradle 캐시 + JDK + 테스트/커버리지/정적분석/패키징)

`.github/workflows/ci.yml`
{% raw %}
```yaml
name: CI
on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ feat/**, fix/** ]
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-test:
    name: Build & Test (JDK 21)
    runs-on: ubuntu-22.04
    permissions:
      contents: read
      security-events: write    # CodeQL/스캔 리포팅 시
    env:
      GRADLE_OPTS: -Dorg.gradle.daemon=false -Dorg.gradle.jvmargs="-Xmx2g"
      JAVA_TOOL_OPTIONS: -Dfile.encoding=UTF-8 -Duser.timezone=Asia/Seoul
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: gradle

      - name: Validate Gradle Wrapper
        uses: gradle/wrapper-validation-action@v2

      - name: Lint (Spotless/Checkstyle)
        run: ./gradlew spotlessCheck checkstyleMain --scan

      - name: Unit Test with Coverage
        run: ./gradlew test jacocoTestReport

      - name: Upload JUnit & JaCoCo Reports
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: |
            **/build/reports/tests/test/**
            **/build/reports/jacoco/test/html/**
            **/build/jacoco/test.exec

      - name: SonarQube Scan (self-hosted example)
        if: ${{ env.SONAR_HOST_URL != '' }}
        env:
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: ./gradlew sonar --info

      - name: Build JAR
        run: ./gradlew :app:bootJar

  security:
    name: Security Scan (SBOM/Trivy)
    runs-on: ubuntu-22.04
    needs: build-test
    steps:
      - uses: actions/checkout@v4

      - name: Generate SBOM (Syft)
        uses: anchore/sbom-action@v0
        with:
          path: .
          artifact-name: sbom-cyclonedx.json
          format: cyclonedx-json

      - name: Trivy FS Scan
        uses: aquasecurity/trivy-action@0.24.0
        with:
          scan-type: fs
          format: sarif
          output: trivy.sarif
          severity: HIGH,CRITICAL

      - name: Upload Scans
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy.sarif

  gate:
    name: Quality Gate
    needs: [build-test, security]
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/download-artifact@v4
        with: { name: test-reports, path: reports }

      - name: Enforce Coverage ≥ 80%
        run: |
          LINE=$(grep -oP 'line-rate="\K[0-9\.]+' reports/**/jacocoTestReport.xml | head -1)
          RATE=$(python - <<'PY'\nprint(round(float("$LINE")*100))\nPY)
          echo "Coverage=${RATE}%"
          test $RATE -ge 80 || (echo "Coverage gate failed"; exit 1)

      - name: Fail on Critical Vulns (Trivy)
        run: |
          if grep -q "CRITICAL" trivy.sarif; then
            echo "Critical vulnerabilities found"; exit 1
          fi
```
{% endraw %}

**설명 포인트**
- **Gradle 캐시/Wrapper 검증**으로 속도/안전.
- **Spotless/Checkstyle** + **JaCoCo** = 기본 품질 게이트.
- **SonarQube**(자체 호스트 or 클라우드) 스캔(코드스멜/버그/보안취약).
- **SBOM**(Syft)과 **Trivy**로 오픈소스 취약점 확인.
- **gate job**에서 커버리지와 취약점 **정책 실패 시 PR Block**.

> 다른 선택지: CodeQL(정적 분석), OWASP Dependency-Check(전이 의존 취약점), Detekt(Kotlin), ESLint(프런트).

---

## C. 이미지 빌드/푸시 & 보안 서명(빌드가드)

### C-1. Docker/BuildKit + 멀티플랫폼 + 캐시 + cosign 서명

`.github/workflows/image-publish.yml`
{% raw %}
```yaml
name: Build & Push Image
on:
  push:
    branches: [ main ]
  workflow_dispatch:

permissions:
  contents: read
  id-token: write   # OIDC (클라우드 레지스트리 연동 시)

jobs:
  image:
    runs-on: ubuntu-22.04
    env:
      IMAGE: ghcr.io/${{ github.repository }}/acme-api
      VERSION: ${{ github.sha }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      - name: Set up Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build (multi-arch) & Push with cache
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ${{ env.IMAGE }}:${{ env.VERSION }}
            ${{ env.IMAGE }}:latest
          cache-from: type=registry,ref=${{ env.IMAGE }}:buildcache
          cache-to: type=registry,ref=${{ env.IMAGE }}:buildcache,mode=max
          provenance: true   # SLSA provenance

      - name: Trivy Image Scan (break on high/critical)
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ env.IMAGE }}:${{ env.VERSION }}
          format: table
          exit-code: 1
          severity: HIGH,CRITICAL

      - name: Cosign sign (keyless)
        uses: sigstore/cosign-installer@v3
      - run: cosign sign --yes ${{ env.IMAGE }}:${{ env.VERSION }}
```
{% endraw %}

**포인트**
- **multi-arch** 빌드로 AMD64/ARM64 동시 푸시(운영 노드 아키텍처 혼재 대비).
- **provenance**(SLSA) & **cosign** 서명으로 공급망 보안.
- 레지스트리는 GHCR 예시(도커허브/ACR/ECR/GCR로 대체 가능). 클라우드라면 **OIDC 페더레이션**으로 키리스 로그인 권장.

---

## D. GitOps — Argo CD로 CD 구성

### D-1. Argo CD App(단일 앱) 또는 ApplicationSet(환경별)

`deploy/charts/acme-api/Chart.yaml`
```yaml
apiVersion: v2
name: acme-api
version: 1.0.0
type: application
appVersion: "0.0.0"     # 자동 덮어쓰기(이미지 태그) 용
```

`deploy/charts/acme-api/values.yaml` (기본값)
```yaml
image:
  repository: ghcr.io/your-org/acme-api
  tag: latest
  pullPolicy: IfNotPresent

replicaCount: 3
service:
  type: ClusterIP
  port: 80

resources:
  requests: { cpu: 300m, memory: 512Mi }
  limits: { cpu: 1500m, memory: 1Gi }

liveness:
  path: /actuator/health/liveness
readiness:
  path: /actuator/health/readiness
```

`deploy/envs/prod/values.yaml` (환경 오버라이드)
```yaml
replicaCount: 6
image:
  tag: "sha-PLACEHOLDER"   # GitHub Actions가 이 값을 커밋으로 대체
env:
  SPRING_PROFILES_ACTIVE: prod
autoscaling:
  enabled: true
  minReplicas: 4
  maxReplicas: 30
  targetCPUUtilizationPercentage: 60
```

**Argo CD Application (클러스터에 배포)**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: acme-api-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/acme
    targetRevision: main
    path: deploy/charts/acme-api
    helm:
      valueFiles:
        - envs/prod/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 5
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 2m
```

> 운영 팁  
> - ApplicationSet를 쓰면 **dev/stage/prod** 3개 앱을 패턴으로 자동 생성 가능.  
> - **automated**(자동 동기화)를 dev/stage에만 두고, prod는 수동 승인(“Sync Wave/Sync Windows”)로 제어.

### D-2. GitOps 업데이트(이미지 태그 갱신 → PR)

`.github/workflows/gitops-pr.yml`
{% raw %}
```yaml
name: GitOps Bump (stage)
on:
  workflow_run:
    workflows: [ "Build & Push Image" ]
    types: [ completed ]
  workflow_dispatch:

jobs:
  bump:
    if: ${{ github.event.workflow_run.conclusion == 'success' || github.event_name == 'workflow_dispatch' }}
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4

      - name: Extract image tag (last SHA)
        run: echo "TAG=$(git rev-parse --short=12 HEAD)" >> $GITHUB_ENV

      - name: Update stage image tag
        run: |
          FILE=deploy/envs/stage/values.yaml
          sed -i "s/tag:.*/tag: \"sha-${TAG}\"/" "$FILE"
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git checkout -b chore/bump-stage-${TAG}
          git commit -am "chore(stage): bump image tag to sha-${TAG}"
          git push -u origin HEAD

      - name: Open PR to main
        uses: peter-evans/create-pull-request@v6
        with:
          title: "chore(stage): bump image to sha-${{ env.TAG }}"
          body: "Auto bump by CI"
          branch: chore/bump-stage-${{ env.TAG }}
          base: main
          draft: false
          labels: bump
```
{% endraw %}

→ 머지되면 Argo CD가 **stage**를 자동 동기화. 승인이 끝나면 **prod**도 같은 방식으로 PR을 열어 **수동 승인 후 배포**.

---

## E. 배포 승인/보호(Environments & Reviewers)

- GitHub **Environments** `stage`/`prod`를 만들고 **required reviewers** 지정 → CD 워크플로에서 `environment: prod`를 요청하면 승인자 승인 후 진행.
- Argo CD 측은 **Sync Windows**로 시간대/권한 제한 가능.

---

## F. 품질 게이트 — 테스트·커버리지·Lint를 “막는” 규칙으로

### F-1. 테스트·커버리지(Gradle+JaCoCo)
`build.gradle.kts` (발췌)
```kotlin
plugins {
  jacoco
  checkstyle
  id("com.diffplug.spotless") version "6.25.0"
}

tasks.test {
  useJUnitPlatform()
  finalizedBy(tasks.jacocoTestReport)
}

tasks.jacocoTestReport {
  dependsOn(tasks.test)
  reports {
    xml.required.set(true)
    html.required.set(true)
  }
}

tasks.register("enforceCoverage") {
  dependsOn(tasks.jacocoTestReport)
  doLast {
    val min = 0.80
    val report = file("$buildDir/reports/jacoco/test/jacocoTestReport.xml")
    val rate = javax.xml.parsers.DocumentBuilderFactory.newInstance().newDocumentBuilder()
      .parse(report).getElementsByTagName("counter").let { nodes ->
        (0 until nodes.length).asSequence().map { nodes.item(it) }
          .first { it.attributes.getNamedItem("type").nodeValue == "LINE" }
          .attributes.getNamedItem("covered").nodeValue.toDouble() /
        (0 until nodes.length).asSequence().map { nodes.item(it) }
          .first { it.attributes.getNamedItem("type").nodeValue == "LINE" }
          .attributes.getNamedItem("missed").nodeValue.toDouble() + 1
      }
    if (rate < min) throw GradleException("Coverage ${rate*100}% < ${min*100}%")
  }
}
```

CI에선 `./gradlew enforceCoverage`를 호출하거나, 위의 **gate job**과 같이 XML 파싱으로 실패 처리.

### F-2. Lint/정적 분석
- **Spotless**: 포매팅/라이선스 헤더.
- **Checkstyle/PMD**: 코드 규약 위반 차단.
- **SonarQube**: 신규 코드 품질 게이트(커버리지, 버그 0, 취약 0).
- **CodeQL/Trivy**: 보안 취약점 차단.

> 규칙은 **점진 강화**: 초기에 너무 엄격하면 개발 속도 저하. **“신규 코드”**에 우선 적용하고 오래된 코드는 단계적 리팩터.

---

## G. 버전·릴리스·체인지로그 자동화

### G-1. Conventional Commits(권장 규약)
- 형식: `type(scope)!: subject`  
  예) `feat(api)!: add pagination to /orders`  
- 주요 타입: `feat`, `fix`, `perf`, `refactor`, `docs`, `test`, `chore`, `build`, `ci`.  
- **BREAKING CHANGE**는 `!` 또는 바디에 `BREAKING CHANGE:`로 표시 → **메이저** 승격.

### G-2. semantic-release (Node 기반, 언어 독립)
`.releaserc.json`
```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md"]
    }],
    "@semantic-release/github"
  ]
}
```

`.github/workflows/release.yml`
```yaml
name: Release (semantic)
on:
  push:
    branches: [ main ]

jobs:
  release:
    runs-on: ubuntu-22.04
    permissions:
      contents: write
      issues: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }   # 태그 기반 분석
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm i -g semantic-release @semantic-release/changelog @semantic-release/git
      - run: semantic-release
```

→ 자동으로 **버전 결정(semver)**, **Git 태그**, **GitHub Release** 생성, **CHANGELOG.md 업데이트**.

### G-3. Gradle 버전 동기화
- semantic-release가 만든 태그를 **Gradle**이 참조하여 **앱 버전/이미지 태그**에 반영:
```kotlin
val ver = System.getenv("GIT_TAG") ?: "0.0.0-SNAPSHOT"
version = ver
tasks.register("printVersion") { doLast { println(project.version) } }
```
Actions에서:
```yaml
- name: Resolve version from tag
  run: echo "GIT_TAG=${GITHUB_REF_NAME}" >> $GITHUB_ENV
```

### G-4. release-please(대안, Google)
- 언어 독립, PR 기반 릴리스 관리.  
- GitHub App/Action로 **릴리스 PR**을 자동 생성 → 리뷰 후 머지하면 태그/릴리스/체인지로그 생성.

### G-5. 체인지로그 템플릿 커스터마이즈
- 팀에서 중요 섹션만: **Features**, **Fixes**, **Breaking**, **Chores/Docs** 숨김.  
- 이슈/PR 번호 자동 링크(semantic-release 기본 제공).

---

## H. 프로모션 플로우(Dev → Stage → Prod)

1) **main 병합** → CI 통과 → **이미지 빌드/서명** → **stage values.yaml** **자동 PR**  
2) 승인 후 머지 → Argo CD가 stage 배포 → **검증**(헬스/메트릭/트래픽)  
3) “Promote to prod” 워크플로(수동/ChatOps) → **prod values.yaml** PR → 승인 머지 → prod 동기화  
4) semantic-release가 **버전/릴리스/체인지로그** 생성 → 게시

**프로모션 워크플로 예**
{% raw %}
```yaml
name: Promote to Prod
on:
  workflow_dispatch:
    inputs:
      tag:
        description: "Image tag (sha-xxxx)"
        required: true

jobs:
  promote:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      - run: |
          sed -i "s/tag:.*/tag: \"${{ github.event.inputs.tag }}\"/" deploy/envs/prod/values.yaml
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git checkout -b chore/bump-prod-${{ github.event.inputs.tag }}
          git commit -am "chore(prod): bump image to ${{ github.event.inputs.tag }}"
          git push -u origin HEAD
      - uses: peter-evans/create-pull-request@v6
        with:
          title: "promote(prod): ${{ github.event.inputs.tag }}"
          body: "Promote stage artifact to production"
          branch: chore/bump-prod-${{ github.event.inputs.tag }}
          base: main
          labels: promote
          reviewers: team-leads
```
{% endraw %}

---

## I. 보안/컴플라이언스 내장

- **서명/검증**: cosign sign/verify, policy-controller(입력 시 서명 검증).
- **비밀**: Actions에서 클라우드에 **OIDC**로 접속(장기 키 제거).  
- **SBOM 저장**: 릴리스 자산/아티팩트로 **CycloneDX** 보관.  
- **정책**: **Branch Protection**(필수 리뷰/체크), **환경 승인**, **Require signed commits**.  
- **감사**: 배포마다 commit/이미지 digest/ArgoCD Sync Result 링크를 릴리스 노트에 남김.

---

## J. 모니터링/피드백을 파이프라인에 통합

- CI 요약: 테스트/커버리지/취약점 결과를 **Job Summary**에 표로 출력.
- CD 요약: 배포 성공/실패, 롤아웃 상태(Argo events)를 댓글(Bot)로 PR에 반영.
- **슬랙/팀즈 알림**: 빌드 실패/배포 성공/품질 게이트 실패를 채널로.

예) Job Summary 출력
```bash
echo "### Test Summary" >> $GITHUB_STEP_SUMMARY
echo "- Passed: $PASSED" >> $GITHUB_STEP_SUMMARY
echo "- Coverage: ${RATE}%" >> $GITHUB_STEP_SUMMARY
```

---

## K. 트러블슈팅(빠른 표)

| 문제 | 원인 | 해결 |
|---|---|---|
| PR이 계속 빨간불 | 품질 게이트(커버리지/린트/취약점) | 최소치 낮춰 점진 적용 or 테스트 추가, 취약점 패치 |
| 이미지 스캔 실패 | High/Critical 존재 | 베이스 이미지 업데이트, 종속 버전 상향, 임시 Ignore는 만료일 설정 |
| Argo CD 동기화 안됨 | 권한/경로/차트 경로 오류 | Application `path`, `targetRevision` 확인, RBAC 점검 |
| stage만 배포됨 | prod는 승인 대기 | Environments reviewers 승인, Sync Window 확인 |
| semantic-release 버전 안 오름 | 커밋 메시지 규약 위반 | Conventional Commits 강제(커밋 린트 pre-commit) |
| 커버리지 0%로 표기 | jacoco xml 경로/멀티모듈 | 루트에서 리포트 집계 or `reports.xml.required=true` 확인 |

---

## L. 한 장 요약(런북)

1) **CI**: `checkout → JDK → lint → test+coverage → sonar → sbom+trivy → gate`.  
2) **이미지**: Buildx multi-arch + cache + **Trivy** + **cosign**.  
3) **CD(GitOps)**: Helm `values.yaml` 이미지만 바꿔 **PR→머지→Argo 동기화**.  
4) **승격**: stage 검증 후 **Promote to prod** 워크플로로 PR/승인/배포.  
5) **버전/릴리스/체인지로그**: Conventional Commits → **semantic-release** 자동화.  
6) **보안**: OIDC 키리스, SBOM 보관, 정책/서명 검증, Branch Protection.  
7) **가시성**: Job Summary/Slack, Argo 이벤트, 에러율·p95 대시보드로 **즉시 피드백**.

---

## M. 부록 — pre-commit 훅(커밋 규약·포맷팅 자동화)

`.husky/commit-msg` (Node 환경 예)
```bash
#!/usr/bin/env bash
npx --yes commitlint --edit "$1"
```

`.pre-commit-config.yaml` (Python pre-commit 사용 예)
```yaml
repos:
  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v3.3.3
    hooks: [ { id: prettier } ]
  - repo: https://github.com/jumanjihouse/pre-commit-hooks
    rev: 3.0.0
    hooks:
      - id: git-check
```

`package.json` (commitlint)
```json
{
  "devDependencies": {
    "@commitlint/cli": "^19.4.0",
    "@commitlint/config-conventional": "^19.4.0",
    "husky": "^9.1.6"
  },
  "commitlint": { "extends": [ "@commitlint/config-conventional" ] },
  "scripts": {
    "prepare": "husky install"
  }
}
```

→ 커밋 단계에서 규약을 강제해 **릴리스 자동화와 품질 게이트**가 부드럽게 연결된다.

---

## N. 결론
- Actions로 **빠르고 표준화된 CI**, Argo CD로 **안전하고 가시적인 CD**를 만든다.  
- 품질 게이트는 “블로킹 규칙”으로, 릴리스는 “규약 기반 자동화”로 **사람의 실수를 시스템이 보완**한다.  
- 모든 산출물(이미지·SBOM·서명·차트·릴리스노트)을 **Git & 레지스트리**에 남겨 추적 가능성을 극대화하라.  
- 이렇게 구성하면 팀은 “코드 작성과 가설 검증”에 집중하고, **배포는 평온한 일상**이 된다.