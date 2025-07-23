---
layout: post
title: Git - Git Hook
date: 2025-02-19 20:20:23 +0900
category: Git
---
# 🪝 Git Hooks 완전 정리 (feat. Husky)

---

## 🧠 Git Hooks란?

Git은 커밋, 병합, 푸시 등 **특정 이벤트가 발생했을 때** 자동으로 실행할 수 있는 **스크립트(Hook)** 기능을 내장하고 있습니다.

> 예: 커밋 전에 Lint 검사, 푸시 전에 테스트 자동 실행 등

---

## 🔧 Git Hook의 종류

Git 저장소마다 `.git/hooks/` 디렉토리가 있으며, 여기에 여러 Hook 템플릿이 기본으로 제공됩니다.

| Hook 이름 | 실행 시점 | 사용 예시 |
|-----------|-----------|-----------|
| `pre-commit` | `git commit` 전에 | 코드 포맷, lint 검사, 테스트 실행 |
| `prepare-commit-msg` | 커밋 메시지 입력 전에 | 자동 메시지 템플릿 생성 |
| `commit-msg` | 커밋 메시지 작성 후 | 메시지 규칙 검사 (ex. Conventional Commit) |
| `pre-push` | `git push` 전에 | 테스트 통과 여부 확인 |
| `post-merge` | merge 후 | 종속성 자동 설치 (npm install) |
| `post-checkout` | 브랜치 변경 후 | 설정 파일 초기화 등 |

> `.sample` 확장자를 제거하면 실제로 동작함

---

## 📁 Hook 스크립트 기본 구조

예: `pre-commit`에서 ESLint 실행

```bash
#!/bin/sh
echo "🔍 Lint 검사 중..."
npx eslint .
if [ $? -ne 0 ]; then
  echo "❌ Lint 에러가 있습니다. 커밋 중단!"
  exit 1
fi
```

- `exit 1`이면 Git은 커밋을 취소함
- `chmod +x .git/hooks/pre-commit`으로 실행 권한 부여 필요

---

## 🚀 실전 예시

### ✅ 1. pre-commit: 커밋 전에 Lint 검사

```bash
#!/bin/sh
npx prettier --check .
npx eslint .
```

### ✅ 2. pre-push: 푸시 전에 테스트 자동 실행

```bash
#!/bin/sh
npm test
if [ $? -ne 0 ]; then
  echo "❌ 테스트 실패 - 푸시 중단!"
  exit 1
fi
```

---

## 🧪 문제점: Git Hooks는 Git 저장소 내부에서만 동작

- `.git/hooks/`는 **Git에 의해 관리되지 않음**
- 팀원 간 공유 어려움 😥

> 이 문제를 해결하기 위해 등장한 것이 바로 **Husky**

---

# 🐶 Husky: Git Hooks를 깔끔하게 관리하는 툴

---

## 🌟 Husky란?

Husky는 Git Hook을 **프로젝트의 코드 안에서 관리**하고,  
**CI/CD, ESLint, Commit message 규칙 검사 등을 쉽게 설정**할 수 있게 도와주는 도구입니다.

특히 JS 생태계(React, Node.js, Vue 등)에서 자주 사용됩니다.

---

## 🧱 설치 방법

```bash
# 1. 프로젝트 초기화
npm init -y

# 2. Husky 설치
npm install husky --save-dev

# 3. Git Hook 활성화
npx husky install
```

### `package.json`에 추가:

```json
{
  "scripts": {
    "prepare": "husky install"
  }
}
```

> 이후 `npm install` 시 자동으로 `husky install` 실행

---

## 📎 Hook 추가 예시

```bash
npx husky add .husky/pre-commit "npm run lint"
```

`.husky/pre-commit`에 다음과 같은 스크립트가 생깁니다:

```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

npm run lint
```

---

## 🧠 활용 예시: JS 생태계에서의 활용

- `pre-commit`: `eslint`, `prettier`, `tsc` 등 자동 검사
- `commit-msg`: 커밋 메시지 규칙 검증 (`commitlint`)
- `pre-push`: `npm test`, `build` 테스트 등

---

## 🎯 Git Hook & Husky 비교 요약

| 항목 | Git 내장 Hook | Husky |
|------|----------------|-------|
| 설정 위치 | `.git/hooks` | `.husky/` 디렉토리 |
| Git에서 추적 | ❌ 불가 | ✅ Git으로 관리됨 |
| 협업 시 공유 | 불편 | 간편 |
| 실행 환경 | 쉘 스크립트 위주 | JS 환경 친화적 |
| 확장성 | 제한적 | 매우 높음 |

---

## 📦 보완 도구

| 도구 | 설명 |
|------|------|
| Husky | Git Hook 관리 툴 |
| lint-staged | 스테이징된 파일만 검사 |
| commitlint | 커밋 메시지 규칙 검사 (Conventional Commits) |
| commitizen | 커밋 메시지 작성 도우미 (CLI 기반) |

---

## 🔗 참고 링크

- [Git 공식 Hook 설명](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)
- [Husky 공식 사이트](https://typicode.github.io/husky/)
- [lint-staged](https://github.com/okonet/lint-staged)
- [commitlint](https://commitlint.js.org/)