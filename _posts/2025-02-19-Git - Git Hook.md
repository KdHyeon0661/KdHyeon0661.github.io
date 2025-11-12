---
layout: post
title: Git - Git Hook
date: 2025-02-19 20:20:23 +0900
category: Git
---
# Git Hooks 완전 정리 (feat. Husky)

## 0. Git Hooks란?

Git은 커밋, 병합, 푸시, 체크아웃 등 **지정 이벤트 시 자동 실행되는 스크립트**(Hook)를 제공한다. 각 로컬 저장소의 `.git/hooks/` 디렉터리에 위치하며, 파일에 **실행 권한**이 있고 **파일명이 훅 이름과 정확히 일치**하면 동작한다.

- 예: 커밋 전에 포맷/린트, 커밋 메시지 규칙 검사, 푸시 전에 테스트 실행, 병합 후 의존성 설치 등.

훅 실행 순서를 간단히 쓰면 다음처럼 모델링할 수 있다:

$$
\text{Commit Flow}:\quad \texttt{pre-commit}\;\rightarrow\;\texttt{prepare-commit-msg}\;\rightarrow\;\texttt{commit-msg}\;\rightarrow\;\texttt{post-commit}
$$

---

## 1. 내장 Git Hook 종류 — 무엇을 언제 실행하나

`.git/hooks/`에는 샘플 파일(`*.sample`)이 있다. 확장자를 지우고 실행 권한을 주면 즉시 동작한다.

| Hook 이름 | 실행 시점 | 실패 시 동작 | 대표 사용 예 |
|---|---|---|---|
| `pre-commit` | `git commit` 직전, 메시지 입력 전 | **커밋 취소** | 포맷/린트/유닛테스트 초경량 버전, 시크릿 스캔 |
| `prepare-commit-msg` | 메시지 편집기 열기 전 | **커밋 취소** | 메시지 템플릿/자동 prefix 삽입 |
| `commit-msg` | 메시지 확정 직후 | **커밋 취소** | Conventional Commits 규칙 검사 |
| `post-commit` | 커밋 직후 | 진행 | 알림/로컬 태스크 트리거 |
| `pre-push` | `git push` 직전(원격 교신 전에) | **푸시 취소** | 테스트/빌드/타입체크 |
| `pre-rebase` | rebase 시작 전 | **rebase 취소** | 보호 규칙, 특정 브랜치 rebase 방지 |
| `post-merge` | merge 완료 직후 | 진행 | `npm install`, 코드젠, 캐시 재생성 |
| `post-checkout` | 브랜치 전환/파일 checkout 직후 | 진행 | 개발용 .env 동기화, 생성물 정리 |
| `applypatch-msg`, `pre-applypatch`, `post-applypatch` | 패치 적용 시 | 다양 | 패치 워크플로 |
| `pre-merge-commit`, `post-rewrite` 등 | 특수 상황 | 다양 | 고급 시나리오 |

> 서버측 훅(예: `pre-receive`, `update`, `post-receive`)은 GitHub/GitLab 같은 원격 서버에서 실행되는 개념으로, **클라이언트 훅 우회(`--no-verify`)를 막는 최종 보호막**이 된다.

---

## 2. 순정 Git Hook — 바로 써먹는 최소 예제

### 2.1 `pre-commit`에서 ESLint/Prettier 검사

```bash
# .git/hooks/pre-commit
#!/bin/sh
echo "[pre-commit] Running format & lint..."
npx prettier --check .
if [ $? -ne 0 ]; then
  echo "[pre-commit] Prettier check failed."
  exit 1
fi

npx eslint .
if [ $? -ne 0 ]; then
  echo "[pre-commit] ESLint failed."
  exit 1
fi
```

```bash
chmod +x .git/hooks/pre-commit
```

- 실패 시 `exit 1`로 커밋을 중단한다.
- 장점: 깃만 있으면 동작. 단점: **이 디렉터리는 Git이 추적하지 않아 팀 공유가 어렵다.**

### 2.2 `commit-msg`로 메시지 규칙 검사

```bash
# .git/hooks/commit-msg
#!/bin/sh
MSG_FILE="$1"
echo "[commit-msg] Checking message..."
grep -Eq "^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{1,}$" "$MSG_FILE"
if [ $? -ne 0 ]; then
  echo "Commit message must follow Conventional Commits, e.g. 'feat: add login form'"
  exit 1
fi
```

---

## 3. 팀과 공유하려면? — 세 가지 전략

1) **Husky 사용**  
2) **`core.hooksPath`로 버전 관리 디렉터리 지정**  
3) **언어별 Hook 프레임워크(예: Python `pre-commit`, Ruby `overcommit`, Go `lefthook`)**

### 3.1 `core.hooksPath` (순정 Git만으로 공유)

```bash
# 저장소 루트에 hooks 디렉터리 버전 관리
mkdir -p .githooks
git config core.hooksPath .githooks
```

- 이제 `.githooks/pre-commit` 등은 Git이 추적하므로 **팀에 공유**된다.
- 장점: Node 미의존. 단점: 설치 스텝(위 config)이 필요.

---

## 4. Husky — JavaScript 생태계 표준 도구

Husky는 Git Hook을 **프로젝트 내부 디렉터리(`.husky/`)**로 관리하게 해주고, 설치 자동화(`prepare` 스크립트)로 팀 온보딩을 단순화한다.

### 4.1 설치

```bash
npm init -y
npm install -D husky
npx husky install
```

`package.json`에:

```json
{
  "scripts": {
    "prepare": "husky install"
  }
}
```

> 이제 누군가 이 프로젝트를 clone하고 `npm install`만 해도 `.husky/`가 활성화된다.

### 4.2 Hook 추가

```bash
npx husky add .husky/pre-commit "npm run lint"
```

생성된 파일:

```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

npm run lint
```

### 4.3 `lint-staged`와 함께 “스테이징 파일만” 검사

대용량 리포에서 전체 폴더 검사 대신 **스테이징된 변경만** 검사하면 눈에 띄게 빨라진다.

```bash
npm install -D lint-staged
```

```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --max-warnings=0", "prettier --check"],
    "*.{md,json,yml,yaml}": ["prettier --check"]
  },
  "scripts": {
    "lint": "eslint ."
  }
}
```

```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

npx lint-staged
```

> 성능상 이점이 크며, **개발자 경험(DX)** 을 현저히 개선한다.

### 4.4 `commitlint`로 메시지 규칙 강제

```bash
npm install -D @commitlint/{config-conventional,cli}
```

```bash
# commitlint.config.cjs
module.exports = { extends: ['@commitlint/config-conventional'] };
```

```bash
npx husky add .husky/commit-msg "npx commitlint --edit \$1"
```

> 실패 시 커밋이 취소된다. Conventional Commits를 통해 자동 릴리즈(semantic-release)와도 연계가 쉽다.

### 4.5 Commitizen으로 메시지 가이드

```bash
npm install -D commitizen cz-conventional-changelog
npx commitizen init cz-conventional-changelog --save-dev --save-exact
```

```json
{
  "scripts": {
    "cz": "cz"
  }
}
```

- `npm run cz` 로 프롬프트 기반 메시지 작성.
- `commitlint`와 함께 쓰면 **친절한 입력 + 엄격한 검증** 구성이 된다.

---

## 5. 모노레포/여러 패키지에서의 Hook

- 루트 `.husky/`에서 훅을 만들고, 하위 패키지에 맞게 스크립트를 라우팅한다.
- 예: pnpm 워크스페이스에서 변경된 패키지에만 테스트 실행

```bash
# .husky/pre-push
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "[pre-push] Running tests only for changed packages..."
pnpm -r --filter ...[origin/main] test
```

- `--filter ...[origin/main]`는 메인 대비 변경 범위만 골라 실행(전략은 워크플로에 맞게 조정).

---

## 6. 다양한 언어별 조합 예시

### 6.1 Python 프로젝트

- `pre-commit` 프레임워크(파이썬 생태계 표준)와 Husky를 함께 써도 된다. 혹은 **Git만**으로 `core.hooksPath`로도 충분.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.8.0
    hooks:
      - id: black
  - repo: https://github.com/PyCQA/flake8
    rev: 7.1.1
    hooks:
      - id: flake8
```

```bash
pip install pre-commit
pre-commit install
```

Husky로 래핑하면:

```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

pre-commit run --all-files
```

### 6.2 .NET/C#

```bash
# .husky/pre-push
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

dotnet build -c Release
if [ $? -ne 0 ]; then
  echo "Build failed."
  exit 1
fi
dotnet test --no-build --configuration Release
```

### 6.3 Go

```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

gofmt -l .
test -z "$(gofmt -l .)"
if [ $? -ne 0 ]; then
  echo "gofmt required."
  exit 1
fi

go vet ./...
go test ./...
```

---

## 7. Windows/WSL/CI 환경 주의점

- **Shebang**: `#!/bin/sh` 권장(크로스 플랫폼). Windows Git Bash/WSL에서 정상 동작.
- **권한**: `chmod +x .husky/*` 가 필요. CI에서는 체크아웃 옵션으로 권한이 깨질 수 있으니, CI 스크립트에서 한 번 더 `chmod` 하거나 Git 속성으로 관리.
- **Node 버전**: nvm/volta 사용 시, 훅 내부에서 **명시적으로** 프레임워크 실행 경로를 잡거나, `corepack` 활성화로 패키지 매니저 버전 고정.

---

## 8. 성능 최적화 체크리스트

- **`lint-staged`** 로 스테이징 파일만 검사
- **작은 훅**: `pre-commit`은 빠르게 끝내고, 무거운 작업(통합 테스트)은 `pre-push`나 CI로 미룬다
- **캐시 활용**: ESLint/TypeScript/테스트 러너의 캐시 옵션(`--cache`)을 켠다
- **병렬 실행**: `concurrently`, `zx`, `make` 등을 사용해 독립 작업 병렬화

```bash
npm install -D concurrently
```

```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

npx concurrently \
  "prettier --check ." \
  "eslint . --max-warnings=0"
```

---

## 9. 보안/정책 및 우회 방지

- 로컬 훅은 `--no-verify` 로 우회 가능:
  ```bash
  git commit -m "msg" --no-verify
  git push --no-verify
  ```
- **근본적 강제**는 서버측 훅(호스팅별 `pre-receive`) 또는 **브랜치 보호 규칙**(필수 상태 체크, 서명 커밋, 선형 이력)으로 만든다.
- **시크릿 스캔**: `trufflehog`, `gitleaks`를 `pre-commit`/`pre-push`에 넣어 유출 방지.

```bash
# .husky/pre-push
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

npx gitleaks detect --no-banner --verbose
```

---

## 10. CI와의 역할 분담

- 훅은 **개발자 머신에서 빠른 피드백** 제공
- CI는 **권위(authoritative) 검사**로, 반드시 통과해야 병합되도록 보호 규칙과 연결
- GitHub Actions의 예(훅과 동일 검사를 서버에서 재검증):

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --ci
```

> PR 보호 규칙에서 **Required status checks** 로 `CI`를 필수화하면, 로컬 우회가 무력화된다.

---

## 11. 트러블슈팅 모음

| 증상 | 원인 | 해결 |
|---|---|---|
| 훅이 안 돈다 | 실행 권한 없음, 파일명/경로 오타 | `chmod +x`, 경로 확인, `git config core.hooksPath` 여부 확인 |
| Windows에서 npx 명령 못 찾음 | PATH/쉘 이슈 | `bash`로 강제 실행, 절대경로 사용, `corepack enable` |
| 너무 느리다 | 전체 폴더 검사 | `lint-staged` 도입, 캐시, 병렬 |
| 커밋이 안 된다 | 린트/테스트 실패 | 로그 확인 후 규칙 완화 또는 스킵 조건(예: docs 커밋 스킵) |
| `--no-verify` 남용 | 정책 미비 | 서버측 훅/보호 규칙/필수 CI 도입 |
| Husky가 설치 안 됨 | `prepare` 미실행 | `npm run prepare` 수동 실행, CI 권한 확인 |

---

## 12. 실전 레시피 — “웹 프론트 단일 리포” 표준 세트

1) 의존성
```bash
npm i -D husky lint-staged @commitlint/{cli,config-conventional} prettier eslint typescript
```

2) 설정 파일
```json
// package.json (발췌)
{
  "scripts": {
    "prepare": "husky install",
    "lint": "eslint . --max-warnings=0",
    "format": "prettier --check .",
    "test": "vitest run"
  },
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --max-warnings=0", "prettier --check"],
    "*.{md,json,yml,yaml}": ["prettier --check"]
  }
}
```

```bash
# commitlint.config.cjs
module.exports = { extends: ['@commitlint/config-conventional'] };
```

3) 훅
```bash
npx husky add .husky/pre-commit "npx lint-staged"
npx husky add .husky/commit-msg "npx commitlint --edit \$1"
npx husky add .husky/pre-push "npm test"
```

---

## 13. 대안/변형: Node 의존 없이 공유하려면

- `core.hooksPath` + 순정 쉘 스크립트로 버전관리
- 다국어 팀이면 언어별 프레임워크를 존중: Python(`pre-commit`), Go(`lefthook`), Ruby(`overcommit`)

예: `core.hooksPath` 하나로 통합

```bash
mkdir -p tools/hooks
git config core.hooksPath tools/hooks
```

```bash
# tools/hooks/pre-commit
#!/bin/sh
set -e
printf "[pre-commit] fmt+lint...\n"
prettier --check .
eslint . --max-warnings=0
```

---

## 14. 수학적 요약 — 커밋 파이프라인에서의 훅 실행 모델

커밋 과정에서 훅의 실패가 파이프라인을 중단하는 것을 다음처럼 쓸 수 있다:

$$
\text{Commit} =
\begin{cases}
\text{fail}, & \text{if any of}\ \{\texttt{pre-commit},\ \texttt{prepare-commit-msg},\ \texttt{commit-msg}\}\ \text{fails} \\
\text{success}, & \text{otherwise}
\end{cases}
$$

푸시도 유사하게:

$$
\text{Push} =
\begin{cases}
\text{fail}, & \text{if}\ \texttt{pre-push}\ \text{fails (client)}\\
\text{fail}, & \text{if}\ \texttt{pre-receive}\ \text{fails (server)}\\
\text{success}, & \text{otherwise}
\end{cases}
$$

---

## 15. 요약 표 — 내장 Hook vs Husky vs core.hooksPath

| 항목 | 내장 Hook(.git/hooks) | Husky(.husky) | core.hooksPath(예: .githooks) |
|---|---|---|---|
| 팀 공유 | 기본 불가 | 가능(코드로 추적) | 가능(코드로 추적) |
| 설치 편의 | 수동 복사/권한 | `prepare` 자동 | `git config` 필요 |
| 생태계 연계 | 쉘 중심 | JS 도구와 밀접 | 언어 불문 |
| 러닝커브 | 낮음 | 낮음 | 낮음 |
| Node 필요 | 불필요 | 필요 | 불필요 |
| 모노레포 | 수동 구성 | 쉽게 구성 | 수동 구성 |

---

## 16. 명령 치트시트

```bash
# 훅 활성화(순정)
chmod +x .git/hooks/pre-commit

# Husky 설치/활성화
npm i -D husky
npx husky install
npx husky add .husky/pre-commit "npx lint-staged"

# lint-staged
npm i -D lint-staged
# package.json에 "lint-staged": { ... }

# commitlint
npm i -D @commitlint/{cli,config-conventional}
# .husky/commit-msg에 "npx commitlint --edit $1"

# 훅 우회(로컬만, 정책상 지양)
git commit -m "msg" --no-verify
git push --no-verify

# 순정 Git로 공유
git config core.hooksPath .githooks
```

---

## 참고 링크

- Git 공식 훅 설명: https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks  
- Husky: https://typicode.github.io/husky/  
- lint-staged: https://github.com/okonet/lint-staged  
- commitlint: https://commitlint.js.org/