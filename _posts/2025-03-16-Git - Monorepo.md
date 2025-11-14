---
layout: post
title: Git - Monorepo
date: 2025-03-16 20:20:23 +0900
category: Git
---
# Monorepo 전략 완벽 정리 (with Git 도구)

> 사용자가 제공한 초안을 바탕으로, 실제 팀에 바로 적용 가능한 **구조 설계 → Git 최적화(sparse-checkout 등) → 패키지/서비스 관리(Lerna/Nx/Turborepo/pnpm) → CI/CD 가속 → 버전·릴리스 전략(Changesets/semver) → 거버넌스(CODEOWNERS/브랜치 보호)**까지 단계별로 확장 정리한다.

---

## Monorepo 한 줄 정의와 사용 맥락

- **정의**: 여러 앱·라이브러리·도구를 **하나의 Git 저장소**에서 관리하는 전략.
- **목표**: 코드 공유·일관 툴링·단일 CI를 통해 **개발/검증/배포 파이프라인의 중복을 제거**.
- **핵심 난점**: 저장소·빌드가 커지면서 성능과 권한/거버넌스 이슈가 함께 커진다. 이를 **Git 기능(Partial/Sparse) + 빌드 캐시 + 변경 영향도 기반 실행**으로 해결한다.

---

## 대표 구조 패턴

### 기본 구조

```
📁 my-org-repo/
├── apps/                     # 배포 단위(웹/앱/서비스)
│   ├── web/
│   └── admin/
├── packages/                 # 공유 라이브러리(도메인 SDK/UI/유틸 등)
│   ├── auth/
│   ├── ui/
│   └── utils/
├── tools/                    # 스크립트/코드젠/CI 헬퍼
│   └── scripts/
├── package.json              # workspace 루트
├── pnpm-workspace.yaml       # 또는 yarn/pnpm workspace
└── nx.json / turbo.json      # Nx 또는 Turborepo 설정
```

### 언어별 변형 예

- **Node/TS**: npm/pnpm/yarn workspace + Lerna/Nx/Turborepo
- **Go**: 루트 `go.work` + 각 모듈 `go.mod`
- **Python**: `pyproject.toml` 여러 개 + Hatch/Poetry/PDM
- **Java**: 멀티 모듈 Gradle (`settings.gradle`) 또는 Maven Aggregator
- **Polyglot**: Nx/Turborepo로 툴링 통합 + 언어별 러너(executor) 연결

---

## 장단점 확장

### 장점

| 장점 | 설명 | 보완책 |
|------|------|--------|
| 코드 공유 용이 | 내부 패키지를 로컬 링크로 신속 반영 | workspace/implicit linking |
| 변경 추적 통일 | 단일 PR/커밋에서 시스템 전반 변화 추적 | Conventional Commits + Changesets |
| 일관된 툴링 | 하나의 ESLint/Prettier/Test/Build 설정 | 루트 config + 패키지별 오버라이드 |
| 단일 CI 파이프라인 | 파이프라인 중복 제거, 공통 캐시 | Affected Only, paths-filter, 캐시 |

### 단점과 대응

| 단점 | 상세 | 대응 |
|------|------|------|
| 저장소 비대 | clone/pull 느림 | Git **partial clone + sparse-checkout** |
| 빌드 느림 | 전체 빌드 과다 | **변경 영향도(affected)** + **원격 캐시** |
| 권한 분리 난이도 | 디렉터리 단위 접근 제어 부재 | **CODEOWNERS**, 브랜치 보호, CI 게이트 |
| 릴리스 복잡 | 버전·체인지로그 난해 | **Changesets** 또는 Lerna publish 전략 |

---

## Git 최적화: Partial Clone, Sparse Checkout, Worktree

### Partial Clone + Sparse Checkout

대규모 저장소에서 **필요 디렉터리만** 빠르게 내려받는다.

```bash
# 히스토리/Blob 최소화

git clone --filter=blob:none --no-checkout https://github.com/your-org/monorepo.git
cd monorepo

# sparse 모드 활성화

git sparse-checkout init --cone

# 필요한 디렉터리만

git sparse-checkout set apps/web packages/ui

# 필요한 시점에만 다른 경로 추가

git sparse-checkout add packages/auth
```

- `--filter=blob:none`은 **partial clone**(서버를 promisor로) 하여 blob 지연 다운로드.
- `--cone` 패턴은 트리 성능 최적화.

### 부분 히스토리만 받기(shallow)

```bash
git fetch --depth=1 origin main
git checkout main
```

- 빌드/테스트만 필요한 CI에서 유용.

### Worktree로 병렬 개발

하나의 저장소에 **여러 작업 트리**를 연결해 브랜치별 코드를 동시에 열 수 있다.

```bash
git worktree add ../feature-login feature/login
git worktree list
```

- 여러 앱/패키지를 동시에 수정해야 할 때 유용.

---

## JavaScript/TypeScript Monorepo 도구

### npm/pnpm/yarn workspaces

- 루트에서 의존성 설치/호이스팅 및 패키지 간 **로컬 링크** 자동.

`pnpm-workspace.yaml` 예:

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

루트 `package.json`:

```json
{
  "private": true,
  "packageManager": "pnpm@9.12.0",
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "build": "pnpm -r build",
    "test": "pnpm -r test"
  }
}
```

### Lerna

설치 및 초기화:

```bash
npm i -D lerna
npx lerna init
```

핵심 명령:

```bash
npx lerna bootstrap         # 패키지 설치+링크
npx lerna run build         # 전 패키지 빌드
npx lerna changed           # 변경 패키지만
npx lerna publish           # 버전/배포 (independent/fixed)
```

`lerna.json` 예:

```json
{
  "version": "independent",
  "npmClient": "pnpm",
  "packages": ["packages/*", "apps/*"],
  "command": {
    "publish": {
      "conventionalCommits": true
    }
  }
}
```

- `version: "independent"`: 패키지별 버전을 독립 관리.
- `conventionalCommits`: 커밋 메시지에서 자동 버전 결정 가능.

### Nx

의존성 그래프, 영향도 기반 실행, 캐시.

설치:

```bash
npx create-nx-workspace@latest
```

대표 명령:

```bash
nx graph            # 의존성 시각화
nx affected:test    # 변경 영향 패키지만 테스트
nx run-many --target=build --all
```

캐시 저장소(Remote Cache) 연결(예: Nx Cloud)로 CI 속도 향상.

### Turborepo

파이프라인 선언형 + 캐시/병렬 최적화.

`turbo.json` 예:

```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": []
    },
    "lint": {
      "outputs": []
    }
  }
}
```

실행:

```bash
pnpm dlx turbo run build test lint --filter=...
```

- 상위 의존 빌드를 선행(`^build`)
- 캐시가 동일 입력/환경에서 **결과 재사용**

### pnpm workspace

의존성 저장 방식이 효율적(중복 제거). **대형 Monorepo**에서 디스크 절약.

명령:

```bash
pnpm -r build            # 모든 패키지
pnpm --filter ./apps/web build
pnpm --filter "@org/ui" build
```

---

## CI/CD 최적화 (GitHub Actions 예시)

### 변경 경로 기반 실행(paths-filter)

변경된 영역만 테스트/빌드:

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
              - 'apps/api/**'
              - 'packages/**'
            web:
              - 'apps/web/**'
              - 'packages/**'

  test-api:
    needs: changes
    if: needs.changes.outputs.api == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm i --frozen-lockfile
      - run: pnpm --filter ./apps/api test

  test-web:
    needs: changes
    if: needs.changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm i --frozen-lockfile
      - run: pnpm --filter ./apps/web test
```
{% endraw %}

### Nx/Turbo 기반 Affected Only

Nx:

```yaml
- run: pnpm i --frozen-lockfile
- run: npx nx affected --target=test --parallel=3
```

Turborepo:

```yaml
- run: pnpm dlx turbo run test --filter=...[HEAD^1]
```

### Git 최적화 활용

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0                   # Nx/Turbo 영향도 분석 시 필요
    sparse-checkout: |
      apps/web
      packages/ui
```

### 캐시

{% raw %}
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: pnpm

- uses: actions/cache@v4
  with:
    path: |
      .turbo
      node_modules/.cache/nx
    key: ${{ runner.os }}-mono-${{ hashFiles('**/pnpm-lock.yaml', '**/turbo.json', '**/nx.json') }}
```
{% endraw %}

---

## 버전·릴리스 전략

### 전략 옵션

| 전략 | 설명 | 장단점 |
|------|------|--------|
| Fixed | 모든 패키지 동일 버전 | 관리 단순 vs 변경 작은 패키지도 강제 상승 |
| Independent | 패키지별 버전 | 현실적/정확 vs 관리 복잡 |
| 앱은 태그, 라이브러리는 패키지 버전 | 배포 단위에 맞춤 | 이중 전략 관리 필요 |

### Changesets로 체인지로그·버전 자동화

설치:

```bash
pnpm add -D @changesets/cli
pnpm changeset init
```

PR에서 변경 소개:

```bash
pnpm changeset                  # 마법사로 변경 요약 입력
pnpm changeset version          # 버전/체인지로그 반영
pnpm changeset publish          # 레지스트리로 배포
```

CI 자동화 예(태그 푸시 시):

{% raw %}
```yaml
- name: Version packages
  run: pnpm changeset version
- name: Publish
  run: pnpm changeset publish
  env:
    NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```
{% endraw %}

### Conventional Commits + 자동 버전

`feat:`, `fix:`, `chore:` 규칙으로 릴리스 판단. Lerna `conventionalCommits` 사용 또는 semantic-release.

---

## 거버넌스: CODEOWNERS, 브랜치 보호, 경로 제한

### CODEOWNERS

`.github/CODEOWNERS`:

```
/apps/web/          @web-team
/apps/admin/        @admin-team
/packages/ui/       @design-team
/tools/             @devexp-team
```

- PR 생성 시 자동 리뷰어 지정.
- 브랜치 보호에서 **Require review from Code Owners** 활성화.

### 브랜치 보호

- Require status checks to pass
- Require pull request reviews
- Require linear history (선택)
- Restrict who can push(필요 시)

### 경로 기반 정책

- GitHub는 **디렉터리 권한**을 직접 제공하지 않으므로, **CI에서 경로 검증**하여 정책 위반 PR 실패 처리.
- 예: `apps/banking/**`는 특정 팀만 수정 허용 → CI에서 작성자/팀 검증.

---

## Submodule vs Subtree vs Monorepo

| 항목 | Submodule | Subtree | Monorepo |
|------|-----------|---------|----------|
| 저장 형태 | 외부 저장소를 링크 | 외부 저장소를 통합 병합 | 단일 저장소 |
| 업데이트 | 수동(커밋 고정) | 병합/동기화 필요 | 내부 패키지 링크 |
| 협업 난이도 | 높음 | 중간 | 낮음(내부 공유 용이) |
| 대규모 개발 | 불편 | 관리 복잡 | 권장(도구 적합) |

- Submodule/Subtree는 외부 종속을 **엄격히 고정**하고 싶을 때만 고려.

---

## 언어별 모듈·빌드 특이점

### Go

루트 `go.work`:

```
go 1.22

use (
  ./apps/api
  ./packages/sdk
)
```

- 각 디렉터리 `go.mod`에서 모듈 선언. `go work sync`로 go.work 반영.
- 변경 영향도 기반 빌드: `go list -deps` + CI paths-filter.

### Python

- `pyproject.toml` 기반. Poetry/Hatch로 관리.
- 공통 툴링(ruff/black/pytest) 루트 설정 + 패키지별 env 관리.
- wheel 생성 및 내부 index(예: Nexus/PyPI private)로 배포.

### Java

- Gradle 멀티 모듈: 루트 `settings.gradle`에서 포함 프로젝트 선언.
- 빌드 캐시, configuration cache로 속도 개선.

---

## 마이그레이션 로드맵(Polyrepo → Monorepo)

1) **목록 정리**: 서비스/라이브러리/공통 구성요소 인벤토리
2) **합치는 순서**: 공통 라이브러리 → 소비 앱 순으로
3) **패키지 이름/버전 정책 확정**: scope(`@org/*`), semver
4) **워크스페이스 도입**: pnpm/yarn workspace 설정
5) **빌드/테스트 통합**: Nx/Turbo/Lerna로 공통 스크립트
6) **CI 전환**: paths-filter → affected only, 캐시
7) **거버넌스**: CODEOWNERS, 브랜치 보호, 리뷰 규칙
8) **릴리스 자동화**: Changesets/semantic-release
9) **문서화**: 기여 가이드(CONTRIBUTING.md), 린트/포맷 규칙, 브랜치 전략

---

## 실전 레시피 모음

### 루트 스크립트 예

`package.json`:

```json
{
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx,.js",
    "typecheck": "tsc -b",
    "build": "pnpm -r build",
    "test": "pnpm -r test",
    "affected:test": "nx affected --target=test",
    "affected:build": "nx affected --target=build"
  }
}
```

### Apps/Web 빌드 스크립트 예

`apps/web/package.json`:

```json
{
  "name": "web",
  "scripts": {
    "build": "vite build",
    "test": "vitest run",
    "lint": "eslint src --ext .ts,.tsx"
  },
  "dependencies": {
    "@org/ui": "workspace:*",
    "@org/auth": "workspace:*"
  }
}
```

### Turborepo 파이프라인 예

`turbo.json`:

```json
{
  "pipeline": {
    "typecheck": {
      "outputs": []
    },
    "build": {
      "dependsOn": ["^build", "typecheck"],
      "outputs": ["dist/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": []
    }
  }
}
```

### GitHub Actions 전체 예시(변경 영향 + 캐시 + 릴리스)

{% raw %}
```yaml
# .github/workflows/ci.yml

name: Monorepo CI

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  detect:
    runs-on: ubuntu-latest
    outputs:
      api: ${{ steps.filter.outputs.api }}
      web: ${{ steps.filter.outputs.web }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - id: filter
        uses: dorny/paths-filter@v3
        with:
          filters: |
            api:
              - "apps/api/**"
              - "packages/**"
            web:
              - "apps/web/**"
              - "packages/**"

  build-web:
    needs: detect
    if: needs.detect.outputs.web == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm i --frozen-lockfile
      - run: pnpm --filter ./apps/web build
      - uses: actions/upload-artifact@v4
        with:
          name: web-dist
          path: apps/web/dist

  build-api:
    needs: detect
    if: needs.detect.outputs.api == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm i --frozen-lockfile
      - run: pnpm --filter ./apps/api build

  release:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    needs: [build-web, build-api]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: pnpm i --frozen-lockfile
      - run: pnpm changeset version
      - run: pnpm changeset publish
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```
{% endraw %}

---

## 운영 체크리스트

- Git
  - partial clone `--filter=blob:none`, sparse-checkout로 필요한 디렉터리만
  - worktree로 병렬 브랜치 작업
- 빌드
  - Nx/Turbo/Lerna + workspace
  - Affected Only, 원격 캐시
- CI
  - paths-filter, 캐시, concurrency cancel, timeout
  - 실패 로그·리포트 아티팩트 업로드
- 릴리스
  - Changesets/semantic-release, semver 표준
  - 태그/릴리스 노트 자동화
- 보안/거버넌스
  - CODEOWNERS, 브랜치 보호, Required reviews/status checks
  - 환경 보호(승인자), 최소 권한 `permissions`
- 문서화
  - CONTRIBUTING.md, 패키지 네이밍/버전 규칙, PR 템플릿, 커밋 규칙

---

## 도입 여부 판단 가이드

- 다음에 해당하면 Monorepo를 고려
  - 공통 코드/디자인 시스템을 여러 앱이 공유
  - 린트/테스트/빌드/릴리스 규칙을 일원화하고 싶다
  - 팀 간 PR 동기화를 쉽게 하고 싶다
- 다음에 해당하면 Polyrepo가 더 적합
  - 강력한 권한 격리가 필수(규제 준수, 비밀 소스)
  - 각 서비스의 릴리스 주기가 완전히 독립
  - 레거시 도메인 간 결합이 크고 이동 비용이 과다

---

## 참고 링크

- Git sparse-checkout: https://git-scm.com/docs/git-sparse-checkout
- Lerna: https://lerna.js.org/
- Nx: https://nx.dev/
- Turborepo: https://turbo.build/repo
- pnpm Workspaces: https://pnpm.io/workspaces
- Changesets: https://github.com/changesets/changesets
