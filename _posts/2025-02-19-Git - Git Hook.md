---
layout: post
title: Git - Git Hook
date: 2025-02-19 20:20:23 +0900
category: Git
---
# Git Hooks 완벽 가이드 (Husky 포함)

## Git Hooks란 무엇인가?

Git Hooks는 Git 작업의 특정 시점에 자동으로 실행되는 스크립트입니다. 예를 들어 커밋하기 전에 코드 포맷을 검사하거나, 푸시하기 전에 테스트를 실행하는 등의 작업을 자동화할 수 있습니다.

각 Git 저장소의 `.git/hooks/` 디렉토리에 위치하며, 실행 권한이 부여된 스크립트 파일이 특정 이름(예: `pre-commit`, `commit-msg`)을 가지면 Git이 자동으로 실행합니다.

---

## 주요 Git Hooks 종류

Git은 다양한 시점에 실행되는 훅을 제공합니다. 가장 자주 사용되는 것들은 다음과 같습니다:

### 커밋 관련 훅
- **`pre-commit`**: `git commit` 명령 실행 직전, 커밋 메시지 입력 전에 실행됩니다. 여기서 코드 검사(린트, 포맷)를 수행할 수 있습니다.
- **`commit-msg`**: 커밋 메시지가 입력된 후 실행됩니다. 메시지 형식을 검증하는 데 사용합니다.
- **`post-commit`**: 커밋이 완료된 후 실행됩니다. 알림을 보내거나 로깅 등의 작업에 활용할 수 있습니다.

### 푸시 관련 훅
- **`pre-push`**: `git push` 명령 실행 전에 실행됩니다. 테스트 실행이나 빌드 검증에 사용합니다.

### 병합 및 브랜치 관련 훅
- **`post-merge`**: 병합이 완료된 후 실행됩니다. 의존성 설치나 캐시 재생성에 사용할 수 있습니다.
- **`post-checkout`**: 브랜치 전환이나 파일 체크아웃 후 실행됩니다.

---

## 기본 Git Hooks 사용하기

### 간단한 pre-commit 훅 예제
```bash
#!/bin/sh
# .git/hooks/pre-commit

echo "코드 포맷 검사를 시작합니다..."

# Prettier로 코드 포맷 검사
npx prettier --check .
if [ $? -ne 0 ]; then
  echo "포맷 검사 실패: 코드를 정리해주세요."
  exit 1
fi

echo "포맷 검사 통과!"
```

파일을 생성한 후 실행 권한을 부여합니다:
```bash
chmod +x .git/hooks/pre-commit
```

이제 `git commit`을 실행할 때마다 이 스크립트가 실행됩니다. 스크립트가 실패(exit code 1)하면 커밋이 취소됩니다.

### 문제점: 팀 공유의 어려움
`.git/hooks/` 디렉토리는 Git이 추적하지 않기 때문에, 이 방식으로 만든 훅은 팀원들과 공유할 수 없습니다. 각 팀원이 수동으로 설정해야 합니다.

---

## 팀과 공유하는 방법

### 방법 1: core.hooksPath 사용 (순수 Git)
Git 설정을 변경하여 훅 디렉토리를 프로젝트 내부로 이동시킬 수 있습니다:

```bash
# 프로젝트 루트에 hooks 디렉토리 생성
mkdir -p .githooks

# Git 설정 변경
git config core.hooksPath .githooks

# 예제 pre-commit 훅 생성
echo '#!/bin/sh
echo "공유 가능한 훅 실행 중..."
' > .githooks/pre-commit
chmod +x .githooks/pre-commit
```

이제 `.githooks/` 디렉토리를 Git으로 추적할 수 있어 팀원들과 공유가 가능합니다. 단, 각 팀원이 `git config core.hooksPath .githooks`를 실행해야 합니다.

### 방법 2: Husky 사용 (JavaScript 프로젝트)
Husky는 JavaScript 생태계에서 가장 널리 사용되는 Git Hooks 관리 도구입니다.

---

## Husky: 현대적인 Git Hooks 관리

### 설치 및 설정
```bash
npm install --save-dev husky
npx husky install
```

`package.json`에 다음을 추가하여 자동 설치를 설정합니다:
```json
{
  "scripts": {
    "prepare": "husky install"
  }
}
```

이제 프로젝트를 클론하고 `npm install`을 실행하면 Husky가 자동으로 설정됩니다.

### 훅 추가하기
```bash
# pre-commit 훅 추가
npx husky add .husky/pre-commit "npm run lint"

# commit-msg 훅 추가
npx husky add .husky/commit-msg "npx commitlint --edit $1"
```

### lint-staged와 함께 사용하기
대규모 프로젝트에서는 변경된 파일만 검사하는 것이 효율적입니다. `lint-staged`는 스테이징된 파일만 처리합니다:

```bash
npm install --save-dev lint-staged
```

`package.json` 설정:
```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{css,scss}": ["prettier --write"],
    "*.{md,json}": ["prettier --write"]
  }
}
```

`.husky/pre-commit` 파일 수정:
```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

npx lint-staged
```

### 커밋 메시지 검증 (commitlint)
```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

`commitlint.config.js` 파일 생성:
```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      ['feat', 'fix', 'docs', 'style', 'refactor', 'test', 'chore']
    ]
  }
};
```

이제 모든 커밋 메시지는 Conventional Commits 형식을 따라야 합니다 (예: `feat: 새로운 기능 추가`).

---

## 다양한 언어 환경에서의 적용

### Python 프로젝트
```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# Black으로 코드 포맷 검사
black --check .
if [ $? -ne 0 ]; then
  echo "Black 검사 실패"
  exit 1
fi

# flake8로 린트 검사
flake8 .
```

### Go 프로젝트
```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# 코드 포맷 검사
go fmt ./...
go vet ./...
```

### .NET 프로젝트
```bash
# .husky/pre-push
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# 빌드 및 테스트
dotnet build
dotnet test
```

---

## 성능 최적화 팁

1. **변경된 파일만 검사**: `lint-staged`를 사용하면 스테이징된 파일만 검사하여 속도를 크게 향상시킬 수 있습니다.

2. **캐시 활용**: ESLint, TypeScript 등의 도구는 캐시 기능을 지원합니다. 설정에서 캐시를 활성화하세요.

3. **작업 분리**: 가벼운 검사(포맷, 린트)는 `pre-commit`에서, 무거운 작업(테스트, 빌드)는 `pre-push`나 CI에서 실행하세요.

4. **병렬 실행**: 독립적인 작업은 병렬로 실행할 수 있습니다:
```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# 병렬 실행
npx concurrently "npm run lint" "npm run type-check"
```

---

## 일반적인 문제 해결

### 훅이 실행되지 않을 때
1. 실행 권한 확인: `chmod +x .husky/*`
2. 파일 경로 확인: 훅 파일이 정확한 위치에 있는지 확인
3. Husky 설치 확인: `npx husky install` 실행

### Windows에서의 문제
Windows에서는 다음과 같이 `bash`를 명시적으로 지정할 수 있습니다:
```bash
# .husky/pre-commit
#!/usr/bin/env bash

# 나머지 스크립트...
```

### 특정 커밋에서 훅 건너뛰기
긴급한 상황에서 훅을 건너뛰어야 할 때:
```bash
git commit -m "메시지" --no-verify
git push --no-verify
```

**주의**: 이 방법은 팀 규칙을 위반할 수 있으니 신중히 사용하세요.

---

## 보안 고려사항

로컬 Git Hooks는 `--no-verify` 옵션으로 우회할 수 있기 때문에 보안 강제 수단으로 사용해서는 안 됩니다. 중요한 검사는 다음과 같이 구현하세요:

1. **CI/CD 파이프라인**: GitHub Actions, GitLab CI 등의 자동화 도구에서 필수 검사 실행
2. **보호된 브랜치**: 메인 브랜치에 대한 강제 규칙 설정
3. **프리-리시브 훅**: 서버 측에서 실행되는 훅 사용 (GitHub, GitLab 엔터프라이즈에서 지원)

예를 들어, GitHub에서 비밀 키가 커밋되는 것을 방지하려면:
```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# 비밀 키 검사
if grep -r "SECRET_KEY=" --include="*.env" .; then
  echo "경고: 비밀 키가 발견되었습니다!"
  exit 1
fi
```

---

## 실전 예제: 완전한 웹 프로젝트 설정

### 1. 의존성 설치
```bash
npm install --save-dev husky lint-staged @commitlint/cli @commitlint/config-conventional prettier eslint
```

### 2. package.json 설정
```json
{
  "scripts": {
    "prepare": "husky install",
    "lint": "eslint . --ext .js,.jsx,.ts,.tsx",
    "format": "prettier --check .",
    "test": "vitest run"
  },
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{css,scss}": ["prettier --write"],
    "*.{md,json}": ["prettier --write"]
  }
}
```

### 3. Husky 훅 설정
```bash
# pre-commit 훅
npx husky add .husky/pre-commit "npx lint-staged"

# commit-msg 훅
npx husky add .husky/commit-msg "npx commitlint --edit $1"

# pre-push 훅 (선택사항)
npx husky add .husky/pre-push "npm test"
```

### 4. commitlint 설정
`commitlint.config.js` 파일 생성:
```javascript
module.exports = {
  extends: ['@commitlint/config-conventional']
};
```

---

## 마무리

Git Hooks는 개발 워크플로우를 자동화하고 코드 품질을 유지하는 강력한 도구입니다. 올바르게 사용하면 다음과 같은 이점을 얻을 수 있습니다:

### 주요 장점
1. **일관된 코드 품질**: 모든 팀원이 동일한 코드 표준을 따르도록 강제합니다.
2. **시간 절약**: 반복적인 작업을 자동화하여 개발자들이 중요한 작업에 집중할 수 있습니다.
3. **실수 방지**: 커밋 전에 버그나 형식 오류를 미리 발견할 수 있습니다.
4. **표준화된 커뮤니케이션**: Conventional Commits를 통해 변경 사항을 명확하게 기록할 수 있습니다.

### 선택 가이드
- **소규모 프로젝트/다양한 언어**: `core.hooksPath`를 사용한 순수 Git 방식
- **JavaScript/TypeScript 프로젝트**: Husky + lint-staged + commitlint 조합
- **Python 프로젝트**: `pre-commit` 프레임워크
- **엔터프라이즈 환경**: 서버 측 훅과 CI/CD 파이프라인 조합

### 기억할 점
Git Hooks는 도구일 뿐입니다. 진정한 가치는 팀의 협업 문화와 코드 품질에 대한 공유된 가치관에서 나옵니다. 너무 엄격한 규칙은 생산성을 저하시킬 수 있으므로, 팀의 상황과 필요에 맞는 적절한 수준의 자동화를 찾는 것이 중요합니다.

훅을 처음 도입한다면, 간단한 것부터 시작하여 점진적으로 추가해나가는 것을 권장합니다. 예를 들어 먼저 코드 포맷 검사부터 시작하고, 팀이 적응하면 린트 검사, 테스트 실행 순으로 확장해나가세요.

올바르게 구성된 Git Hooks는 눈에 띄지 않게 조용히 작동하면서 개발 팀의 생산성과 코드 품질을 지속적으로 향상시켜줄 것입니다.