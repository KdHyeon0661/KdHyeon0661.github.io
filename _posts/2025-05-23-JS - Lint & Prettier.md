---
layout: post
title: JavaScript - Lint / Prettier
date: 2025-05-23 21:20:23 +0900
category: JavaScript
---
# ✅ Lint / Prettier 사용법 정리

현대 JavaScript 개발에서는 **코드 품질 유지와 스타일 통일**이 매우 중요합니다.  
협업 시에도 들쭉날쭉한 코드를 방지하고, 사소한 실수도 빠르게 잡아내는 것이 좋습니다.  
이럴 때 사용하는 대표적인 도구가 바로 **ESLint**와 **Prettier**입니다.

---

## 🧹 Lint란?

> 코드의 문법적 오류, 스타일 오류, 잠재적 버그 등을 **정적 분석(Static Analysis)**으로 찾아주는 도구

### 📌 ESLint란?

- JavaScript에서 가장 널리 쓰이는 Linter 도구
- 변수 선언 누락, 사용하지 않는 변수, 들여쓰기 등 체크 가능
- 룰을 설정하거나, 커스텀 룰 추가 가능
- 플러그인으로 React, TypeScript, Vue 등 지원

---

## 🧽 Prettier란?

> 코드 스타일을 **자동으로 정렬(포맷팅)**해주는 도구

### 특징

- 코드 스타일 논쟁을 줄여줌 (탭/스페이스, 세미콜론 등)
- 린트와 달리 **오류를 잡지는 않음**, 대신 **자동 수정**
- ESLint와 병행해서 사용 가능 (충돌 방지 설정 필요)

---

## ⚙️ ESLint 설치 및 설정

### ✅ 설치

```bash
npm install --save-dev eslint
```

### ✅ 초기화

```bash
npx eslint --init
```

> CLI 질문에 따라 환경(React/TS 등), 코드 스타일, 사용 목적 등을 설정

### ✅ 예시 `.eslintrc.json`

```json
{
  "env": {
    "browser": true,
    "es2021": true
  },
  "extends": ["eslint:recommended"],
  "parserOptions": {
    "ecmaVersion": 12,
    "sourceType": "module"
  },
  "rules": {
    "no-unused-vars": "warn",
    "semi": ["error", "always"],
    "quotes": ["error", "double"]
  }
}
```

### ✅ 예시 명령어

```bash
npx eslint src/
npx eslint src/ --fix  // 자동 수정
```

---

## ✨ Prettier 설치 및 설정

### ✅ 설치

```bash
npm install --save-dev prettier
```

### ✅ 설정 파일 `.prettierrc`

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5"
}
```

### ✅ 예시 명령어

```bash
npx prettier src/**/*.js
npx prettier src/**/*.js --write  // 자동 적용
```

---

## 🤝 ESLint + Prettier 함께 사용하기

둘 다 코드 스타일을 다루므로 **충돌 방지 설정**이 필요합니다.

### ✅ 추가 패키지 설치

```bash
npm install --save-dev eslint-config-prettier eslint-plugin-prettier
```

### ✅ `.eslintrc.json` 수정 예시

```json
{
  "extends": [
    "eslint:recommended",
    "plugin:prettier/recommended"
  ],
  "plugins": ["prettier"],
  "rules": {
    "prettier/prettier": "error"
  }
}
```

이 설정은 Prettier 규칙을 ESLint 내부에서 검사하고, 충돌되는 ESLint 룰을 끔.

---

## 💻 VS Code 연동

1. **ESLint**, **Prettier** 확장 설치
2. `.vscode/settings.json` 설정

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "eslint.validate": ["javascript", "javascriptreact"],
  "eslint.run": "onSave",
  "prettier.requireConfig": true
}
```

> 저장 시 자동으로 ESLint + Prettier 동작

---

## 🧠 실무 적용 팁

| 팁 | 설명 |
|-----|------|
| 코드 리뷰 시간 단축 | 스타일 문제로 인한 리뷰 줄어듦 |
| Git pre-commit hook과 연동 | Husky + lint-staged로 커밋 전 자동 검사 가능 |
| CI/CD 연동 | GitHub Actions에서 lint 실패 시 빌드 중단 가능 |
| 팀 스타일 가이드 공유 | `.eslintrc`, `.prettierrc`를 Git으로 공유 |

---

## 🧪 Husky + lint-staged 예시 (선택)

```bash
npm install --save-dev husky lint-staged
npx husky install
```

`.husky/pre-commit`:

```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

npx lint-staged
```

`package.json`:

```json
"lint-staged": {
  "**/*.js": ["eslint --fix", "prettier --write"]
}
```

> 커밋 전에 자동으로 린트/포맷이 실행됩니다.

---

## 🔗 참고 링크

- [ESLint 공식 사이트](https://eslint.org/)
- [Prettier 공식 사이트](https://prettier.io/)
- [eslint-config-prettier](https://github.com/prettier/eslint-config-prettier)
- [Husky 공식 문서](https://typicode.github.io/husky)

---

## 🧾 마무리 정리

- ESLint는 **오류 잡기와 코드 품질 유지**,  
  Prettier는 **자동 정렬과 스타일 통일**에 초점
- 두 도구는 **함께 사용하면 최적**이며, 충돌 방지 설정 필수
- VSCode, Git Hook, CI 등 실무 환경에 통합하면 효과 극대화