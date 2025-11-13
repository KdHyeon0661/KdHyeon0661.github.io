---
layout: post
title: Git - GitHub Actions 기반 CI/CD 자동화
date: 2025-03-02 20:20:23 +0900
category: Git
---
# GitHub Actions 기반 CI/CD 자동화

## 0. Actions 용어·디렉터리 구조 상기

- 워크플로(Workflow): `.github/workflows/*.yml`
- 잡(Job): 워크플로 내부의 병렬/순차 실행 단위
- 스텝(Step): 잡 내부에서 `uses:` 또는 `run:`으로 실행되는 단계
- 러너(Runner): 잡을 실제로 실행하는 머신. GitHub-Hosted(ubuntu-latest 등) 또는 Self-hosted.

프로젝트의 기본 구조 예:

```
📁 .github/
  └─📁 workflows/
      ├─ ci.yml                 # PR/Push CI
      ├─ deploy.yml             # 배포
      ├─ nightly.yml            # 스케줄 작업
      └─ reusable-test.yml      # 재사용 워크플로
```

---

## 1. 최소 CI — Node.js 테스트 (초안 확장)

기본 예제에 캐시와 타 버전 테스트를 더하면 다음과 같다.

{% raw %}
```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]

jobs:
  test:
    name: Node ${{ matrix.node }} on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest]
        node: [16, 18, 20]

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: npm

      - name: Install
        run: npm ci

      - name: Lint
        run: npm run lint --if-present

      - name: Test
        run: npm test -- --reporter junit --reporter-options "output=reports/junit.xml"

      - name: Upload JUnit Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: junit-${{ matrix.node }}
          path: reports/junit.xml
          retention-days: 7
```
{% endraw %}

핵심 포인트
- `strategy.matrix`로 **멀티 런타임** 테스트
- `setup-node`의 `cache: npm`으로 **의존성 캐시**
- 테스트 결과를 `upload-artifact`로 업로드하여 **PR에서 다운로드 가능**

---

## 2. Python·Java 예제(멀티 언어 저장소/모노레포 대비)

### 2.1 Python(pytest + 캐시)

```yaml
# .github/workflows/ci-python.yml
name: CI (Python)

on:
  pull_request:
    paths:
      - "pyapp/**"
      - ".github/workflows/ci-python.yml"
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: pyapp
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"
      - run: pip install -r requirements.txt
      - run: pytest -q --junitxml=reports/pytest.xml
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: pytest-report
          path: pyapp/reports/pytest.xml
          retention-days: 7
```

### 2.2 Java(Gradle 캐시 + 테스트)

```yaml
# .github/workflows/ci-java.yml
name: CI (Java)

on:
  pull_request:
    paths:
      - "java-app/**"
      - ".github/workflows/ci-java.yml"
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: java-app
    steps:
      - uses: actions/checkout@v4
      - name: Setup Temurin JDK
        uses: actions/setup-java@v4
        with:
          java-version: "21"
          distribution: "temurin"
          cache: "gradle"
      - name: Build & Test
        run: ./gradlew clean test
      - name: Publish Test Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: junit-java
          path: java-app/build/test-results/test/*.xml
```

---

## 3. 캐시 전략 심화 — actions/cache, Docker Layer Cache

### 3.1 Node/PNPM/Yarn 등 수동 캐시 키 제어

{% raw %}
```yaml
- name: Cache node_modules
  uses: actions/cache@v4
  with:
    path: |
      **/node_modules
    key: ${{ runner.os }}-modules-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-modules-
```
{% endraw %}

### 3.2 Docker Buildx + 레이어 캐시

{% raw %}
```yaml
# .github/workflows/docker-build.yml
name: Docker Build

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build & Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
          cache-from: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache
          cache-to: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache,mode=max
```
{% endraw %}

---

## 4. 아티팩트·커버리지·주석(Annotations)

### 4.1 커버리지 업로드(예: Codecov)

{% raw %}
```yaml
- name: Run tests with coverage
  run: npm run test:coverage

- name: Upload coverage to Codecov
  uses: codecov/codecov-action@v4
  with:
    token: ${{ secrets.CODECOV_TOKEN }} # 조직 설정에 따라 불필요할 수 있음
    files: ./coverage/lcov.info
```
{% endraw %}

### 4.2 실패 라인에 주석 달기(ESLint 결과 Annotations)

{% raw %}
```yaml
- name: ESLint
  run: npm run lint:ci
  continue-on-error: true

- name: Annotate ESLint result
  if: failure()
  uses: ataylorme/eslint-annotate-action@v3
  with:
    repo-token: ${{ secrets.GITHUB_TOKEN }}
    report-json: "./reports/eslint.json"
```
{% endraw %}

---

## 5. 브랜치 보호 + Status Checks + 환경(Environments) 보호

### 5.1 브랜치 보호(요지)
- Settings → Branches → Add rule
- Require pull request reviews
- Require status checks to pass → 체크 이름은 워크플로 잡 이름(예: `CI / test`)
- Require linear history(선택)

### 5.2 환경 보호(승인자·비밀 분리·URL)
- Settings → Environments → `production` 생성
- Required reviewers 지정 → 배포 직전 승인 필요
- Secrets를 환경 단위로 분리(`secrets.PROD_*`)

배포 잡에서:

```yaml
environment:
  name: production
  url: https://example.com
```

---

## 6. GitHub Secrets/Variables/Permissions — 보안 기본기

### 6.1 최소 권한(Principle of Least Privilege)

워크플로 최상단:

```yaml
permissions:
  contents: read
```

배포 잡에서만 필요한 권한을 확장:

```yaml
jobs:
  deploy:
    permissions:
      contents: read
      id-token: write   # OIDC에 필요
      packages: write   # 레지스트리 푸시 등
```

### 6.2 환경 변수 계층
- `env:` (워크플로/잡/스텝)
- `secrets.*` (민감 정보)
- `vars.*` (민감하지 않은 상수)

예:

```yaml
env:
  APP_ENV: ci
  API_BASE: https://api.example.com

- run: echo "Using $APP_ENV with $API_BASE"
```

---

## 7. OIDC로 클라우드에 보안 접속(AWS 예시) — 키 없는 배포

### 7.1 AWS IAM 역할 구성(개요)
- GitHub OIDC Provider 등록(Organization/Repository level)
- 역할 트러스트 정책에 `sub` 조건으로 워크플로 제약
  - 예: `repo:org/repo:ref:refs/heads/main`

### 7.2 AWS 로그인 + 배포 예시(S3/CloudFront)

```yaml
# .github/workflows/deploy-aws.yml
name: Deploy AWS (OIDC)

on:
  push:
    tags: [ 'v*.*.*' ]

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    permissions:
      id-token: write       # 필수
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
          aws-region: ap-northeast-2

      - name: Build
        run: |
          npm ci
          npm run build

      - name: Sync to S3
        run: aws s3 sync dist/ s3://my-bucket --delete

      - name: Invalidate CloudFront
        run: aws cloudfront create-invalidation --distribution-id ABCDEFGHIJ --paths "/*"
```

장점
- **AWS 키를 Secrets에 넣지 않음**
- 역할 기반의 단기 토큰(STS) 사용 → 노출 위험 감소

---

## 8. Netlify/Firebase 배포(초안 확장)

### 8.1 Netlify

{% raw %}
```yaml
- name: Deploy to Netlify
  uses: nwtgck/actions-netlify@v2
  with:
    publish-dir: './dist'
    production-branch: 'main'
    netlify-auth-token: ${{ secrets.NETLIFY_AUTH_TOKEN }}
    site-id: ${{ secrets.NETLIFY_SITE_ID }}
```
{% endraw %}

### 8.2 Firebase Hosting

{% raw %}
```yaml
- name: Setup Node
  uses: actions/setup-node@v4
  with:
    node-version: 20

- name: Install Firebase CLI
  run: npm i -g firebase-tools

- name: Deploy to Firebase
  run: firebase deploy --only hosting
  env:
    FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
```
{% endraw %}

---

## 9. Kubernetes 배포(kubectl), Helm

{% raw %}
```yaml
# .github/workflows/deploy-k8s.yml
name: Deploy to Kubernetes

on:
  workflow_dispatch:
    inputs:
      imageTag:
        description: "Image tag to deploy"
        required: true
        default: "latest"

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup kubectl
        uses: azure/setup-kubectl@v4
        with:
          version: 'v1.30.2'

      - name: Kubeconfig from secret
        run: |
          mkdir -p ~/.kube
          echo "${KUBECONFIG_B64}" | base64 -d > ~/.kube/config
        env:
          KUBECONFIG_B64: ${{ secrets.KUBECONFIG_B64 }}

      - name: Set image
        run: |
          kubectl set image deployment/web web=ghcr.io/${{ github.repository }}:${{ inputs.imageTag }} -n prod

      - name: Rollout status
        run: kubectl rollout status deployment/web -n prod
```
{% endaw %}

Helm 이용시:

{% raw %}
```yaml
- name: Helm Upgrade
  run: |
    helm upgrade web charts/web \
      --install \
      --namespace prod \
      --set image.tag=${{ inputs.imageTag }}
```
{% endraw %}

---

## 10. 모노레포 최적화 — paths-filter로 변경 영역만 실행

{% raw %}
```yaml
# .github/workflows/ci-monorepo.yml
name: CI Monorepo

on:
  pull_request:
    branches: [ main ]

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      api: ${{ steps.filter.outputs.api }}
      web: ${{ steps.filter.outputs.web }}
    steps:
      - uses: actions/checkout@v4
      - id: filter
        uses: dorny/paths-filter@v3
        with:
          filters: |
            api:
              - 'services/api/**'
            web:
              - 'apps/web/**'

  api:
    needs: changes
    if: needs.changes.outputs.api == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Run API tests only"

  web:
    needs: changes
    if: needs.changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Run Web tests only"
```
{% endraw %}

---

## 11. 동시성·취소·타임아웃 — 불필요한 실행 줄이기

{% raw %}
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test
```
{% endraw %}

---

## 12. 재사용 워크플로(organization-wide 표준화)

### 12.1 호출 당하는 워크플로

{% raw %}
```yaml
# .github/workflows/reusable-test.yml (in org/reusable repo)
name: Reusable Test

on:
  workflow_call:
    inputs:
      node:
        required: true
        type: string

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node }}
      - run: npm ci
      - run: npm test
```
{% endraw %}

### 12.2 호출하는 쪽

```yaml
# .github/workflows/ci.yml
name: CI

on: [pull_request]

jobs:
  call-reusable:
    uses: org/reusable/.github/workflows/reusable-test.yml@v1
    with:
      node: "20"
```

---

## 13. 수동 실행(workflow_dispatch) + 입력 파라미터·승인

{% raw %}
```yaml
on:
  workflow_dispatch:
    inputs:
      env:
        description: "Environment"
        required: true
        default: "staging"
        type: choice
        options: [staging, production]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.env }}
    steps:
      - uses: actions/checkout@v4
      - run: echo "Deploying to ${{ inputs.env }} ..."
```
{% endraw %}

- `environment`가 `production`이면 환경 보호 규칙(승인자)로 배포 전 승인 절차 수행.

---

## 14. 스케줄 작업(schedule) — 야간 빌드·보건 점검

```yaml
on:
  schedule:
    - cron: "0 18 * * *"  # 매일 03:00 KST 기준에 맞춰 조정(UTC 기준)

jobs:
  nightly:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run e2e:headless
```

---

## 15. 실패 알림 — Slack/Discord/Webhook

Slack 예:

{% raw %}
```yaml
- name: Notify Slack (on failure)
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "CI failed on ${{ github.ref }} for ${{ github.sha }}"
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```
{% endraw %}

---

## 16. PR 크기·라벨·자동 병합

### 16.1 PR 라벨링(크기별)

{% raw %}
```yaml
- uses: actions/labeler@v5
  with:
    repo-token: ${{ secrets.GITHUB_TOKEN }}
```
{% endaw %}

`.github/labeler.yml` 예:

```yaml
size/XS:
  - changed-files:
      - any-glob-to-any-file: '**'
      - max-lines-changed: 20
size/S:
  - changed-files:
      - any-glob-to-any-file: '**'
      - min-lines-changed: 21
      - max-lines-changed: 100
```

### 16.2 Dependabot 자동 병합(조건부)

{% raw %}
```yaml
# .github/workflows/auto-merge.yml
name: Auto-merge dependabot

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  automerge:
    if: github.actor == 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - uses: fastify/github-action-merge-dependabot@v3
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```
{% endraw %}

---

## 17. Self-hosted Runner — GPU/사내망/프라이빗 네트워크

- 대규모 빌드·특수 하드웨어(GPU)·내부망 접근이 필요한 경우 Self-hosted Runner 사용
- 보안 수칙
  - 독립 VPC/서브넷, 고정 이미지(불변), 최소 권한 토큰
  - 실행 후 러너 **자동 정리(에페멀)**, 로그/비밀 유출 감시
- 태그 기반 라우팅:
  ```yaml
  runs-on: [self-hosted, linux, gpu]
  ```

---

## 18. 워크플로 간 데이터 전달 — Artifacts·Outputs

### 18.1 스텝/잡 Output

{% raw %}
```yaml
- name: Compute version
  id: ver
  run: echo "VERSION=$(node -p \"require('./package.json').version\")" >> "$GITHUB_OUTPUT"

- name: Use version
  run: echo "Version is ${{ steps.ver.outputs.VERSION }}"
```
{% endraw %}

### 18.2 잡 Output → 다음 잡

{% raw %}
```yaml
jobs:
  build:
    outputs:
      version: ${{ steps.ver.outputs.VERSION }}
    steps:
      - id: ver
        run: echo "VERSION=1.2.3" >> "$GITHUB_OUTPUT"

  release:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Rel version ${{ needs.build.outputs.version }}"
```
{% endraw %}

---

## 19. 릴리스·태깅·체인지로그 자동화

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags: [ 'v*.*.*' ]

jobs:
  gh-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
          files: |
            dist/**.zip
```

태그 생성 파이프라인(버전 증가)을 별도 워크플로에서 수행하고, 방금 예제를 통해 릴리스 노트를 자동 생성할 수 있다.

---

## 20. 품질·보안 내재화: CodeQL, Secret Scanning, 권한

### 20.1 CodeQL

```yaml
# .github/workflows/codeql.yml
name: CodeQL

on:
  schedule:
    - cron: '0 2 * * 1'
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]

jobs:
  analyze:
    permissions:
      contents: read
      security-events: write
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false

    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: 'javascript,python'
      - uses: github/codeql-action/analyze@v3
```

### 20.2 Secret Scanning
- GitHub Advanced Security가 켜져 있으면 자동 검사
- 서드파티 CI 로그·아티팩트에 **비밀이 노출되지 않게** `::add-mask::` 또는 `secrets` 사용.

---

## 21. 운영 팁 모음 — 현업에서 가장 자주 겪는 이슈

1) **러너 시간 절약**: paths-filter로 변경 없는 영역 CI 생략, `concurrency`로 중복 취소
2) **긴 잡 분해**: “빌드 → 테스트 → 린트”를 병렬 잡으로 쪼개 전체 시간을 단축
3) **PR 드리프트 방지**: `pull_request` + `merge_group`(대규모 저장소) 사용
4) **환경별 동행**: `staging`은 자동, `production`은 환경 보호(승인자)
5) **속도 병목**: Docker 캐시, 언어별 캐시, shallow fetch (`fetch-depth: 0` 필요한 경우만)
6) **로그 가독성**: step 이름을 의미 있게, 실패시 아티팩트로 리포트/스크린샷 첨부
7) **권한 최소화**: 워크플로 루트 `permissions: contents: read`, 필요한 잡에서만 확장
8) **태그/릴리스 표준**: SemVer + 릴리스 노트 자동 생성, 배포 아티팩트 첨부
9) **러너 안정성**: Self-hosted라면 자동 패치·이미지 롤링·격리 네트워크·비밀 주입 표준화
10) **문제 재현**: 실패 잡의 아티팩트/캐시 키/환경 변수를 기록해 로컬 재현 스크립트 제공

---

## 22. 끝에서 정리 — 실무형 체크리스트

- 트리거: `pull_request`, `push(main)`, `workflow_dispatch`, `schedule`, `release`
- 품질: Lint, Test(JUnit/coverage), CodeQL, Secret Scanning
- 속도: matrix, cache(actions/setup-*/cache), docker buildx cache
- 결과물: artifacts(리포트), GHCR 이미지, 릴리스
- 정책: Branch Protection(Status checks), Environments(승인자)
- 보안: 최소 권한 `permissions`, OIDC(id-token: write), Secrets 분리
- 운영: concurrency cancel, timeout, paths-filter, reusable workflows
- 배포: Netlify/Firebase/AWS(OIDC)/K8s(Helm/kubectl)

---

## 부록 A) 하나로 묶은 “PR → CI → 배포(스테이징) → 릴리스” 샘플

{% raw %}
```yaml
# .github/workflows/full-pipeline.yml
name: Full Pipeline

on:
  pull_request:
    branches: [ main ]
  push:
    tags: [ 'v*.*.*' ]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: 20

jobs:
  ci:
    name: CI(Lint/Test)
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm

      - run: npm ci
      - run: npm run lint --if-present
      - run: npm test -- --reporter junit --reporter-options "output=reports/junit.xml"
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: junit
          path: reports/junit.xml

  build-image:
    name: Build & Push Image (GHCR)
    needs: ci
    if: github.event_name == 'push' || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache
          cache-to: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache,mode=max

  deploy-staging:
    name: Deploy Staging (Netlify)
    needs: [ci]
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: ${{ steps.deploy.outputs.deploy-url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm
      - run: npm ci && npm run build
      - name: Deploy to Netlify
        id: deploy
        uses: nwtgck/actions-netlify@v2
        with:
          publish-dir: './dist'
          production-branch: 'main'
          netlify-auth-token: ${{ secrets.NETLIFY_AUTH_TOKEN }}
          site-id: ${{ secrets.NETLIFY_SITE_ID }}

  release:
    name: Create GitHub Release
    needs: [build-image]
    if: startsWith(github.ref, 'refs/tags/')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Create Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
          files: |
            dist/**.zip
```
{% endraw %}

---

## 참고 링크

- GitHub Actions 공식 문서: https://docs.github.com/en/actions
- actions/checkout: https://github.com/actions/checkout
- actions/setup-node: https://github.com/actions/setup-node
- actions/cache: https://github.com/actions/cache
- docker/build-push-action: https://github.com/docker/build-push-action
- Netlify Action: https://github.com/nwtgck/actions-netlify
- Codecov Action: https://github.com/codecov/codecov-action
- Branch protection rules: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches
