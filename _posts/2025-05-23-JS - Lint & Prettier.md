---
layout: post
title: JavaScript - Lint / Prettier
date: 2025-05-23 21:20:23 +0900
category: JavaScript
---
# Lint / Prettier 사용법 정리

## 개념 정리 — Lint vs Prettier 역할 구분

- **ESLint(Lint)**: 정적 분석으로 **문법 오류, 잠재 버그, 안티패턴** 탐지 + 일부 자동 수정(`--fix`)
- **Prettier(Formatter)**: **코드 모양(줄바꿈, 들여쓰기, 세미콜론, 따옴표, 줄폭)**을 **일관 규칙으로 자동 정렬**

역할 분담 원칙:
- **ESLint = 품질/버그 규칙** (ex. `no-unused-vars`, `no-undef`, `eqeqeq`, `no-await-in-loop`)
- **Prettier = 포맷 규칙** (ex. `semi`, `quotes`, `max-len`, `indent`)
  → 포맷 관련 ESLint 룰은 **비활성화**하고 Prettier에 맡기는 구성이 일반적입니다.

---

## 프로젝트 세팅 빠른 시작

### 의존성 설치(기본)

```bash
# ESLint 본체

npm i -D eslint

# Prettier 본체

npm i -D prettier

# ESLint ↔ Prettier 충돌 제거/연계

npm i -D eslint-config-prettier eslint-plugin-prettier
```

### Prettier 설정 `.prettierrc`

```json
{
  "singleQuote": true,
  "semi": true,
  "tabWidth": 2,
  "trailingComma": "all",
  "printWidth": 100,
  "arrowParens": "always"
}
```

### ESLint 설정 — 두 가지 방식

#### 신형: **Flat Config** (`eslint.config.js`)

```js
// eslint.config.js (Flat Config)
import eslint from '@eslint/js';
import prettierRecommended from 'eslint-plugin-prettier/recommended';

export default [
  // 기본 JS 권장 규칙
  eslint.configs.recommended,

  // Node/브라우저 공통 환경 설정
  {
    languageOptions: {
      ecmaVersion: 'latest',
      sourceType: 'module',
      globals: {
        window: 'readonly',
        document: 'readonly',
        // Node 런타임을 겸할 경우 필요에 따라 추가
        process: 'readonly',
        __dirname: 'readonly'
      }
    }
  },

  // Prettier와의 연계(포맷 충돌 제거 + 포맷 오류를 ESLint 에러로)
  prettierRecommended,

  // 프로젝트 맞춤 규칙
  {
    rules: {
      'no-unused-vars': ['warn', { argsIgnorePattern: '^_' }],
      'no-console': 'off'
    }
  }
];
```

#### 구형: `.eslintrc.json`

```json
{
  "env": { "browser": true, "es2021": true },
  "extends": ["eslint:recommended", "plugin:prettier/recommended"],
  "parserOptions": { "ecmaVersion": "latest", "sourceType": "module" },
  "rules": {
    "no-unused-vars": ["warn", { "argsIgnorePattern": "^_" }],
    "no-console": "off"
  }
}
```

> 새 프로젝트는 유지보수성을 위해 **Flat Config**를 권장합니다. 기존 레포는 `.eslintrc` 유지해도 무방합니다.

---

## 실행 스크립트 & 기본 사용

`package.json`
```json
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "fmt": "prettier . --check",
    "fmt:write": "prettier . --write"
  }
}
```

실행:
```bash
npm run lint
npm run lint:fix
npm run fmt
npm run fmt:write
```

---

## VS Code 연동(저장 시 자동)

`.vscode/settings.json`
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "eslint.validate": ["javascript", "javascriptreact", "typescript", "typescriptreact"],
  "eslint.run": "onSave",
  "prettier.requireConfig": true
}
```

> 저장 시 Prettier가 포맷을, ESLint가 품질 검사를 수행합니다.

---

## TypeScript 통합

### 패키지

```bash
npm i -D typescript @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

### Flat Config 예시

```js
// eslint.config.js
import tseslint from '@typescript-eslint/eslint-plugin';
import tsParser from '@typescript-eslint/parser';
import prettierRecommended from 'eslint-plugin-prettier/recommended';

export default [
  {
    files: ['**/*.ts', '**/*.tsx'],
    languageOptions: {
      parser: tsParser,
      parserOptions: {
        project: './tsconfig.json',   // 타입 기반 규칙을 원하면 지정(성능 비용 있음)
        ecmaVersion: 'latest',
        sourceType: 'module'
      }
    },
    plugins: { '@typescript-eslint': tseslint },
    rules: {
      ...tseslint.configs['recommended'].rules,
      '@typescript-eslint/no-unused-vars': ['warn', { argsIgnorePattern: '^_' }]
    }
  },
  prettierRecommended
];
```

### 자주 쓰는 TS 규칙 팁

- `@typescript-eslint/no-floating-promises`: 미처리 `Promise` 잡기
- `@typescript-eslint/explicit-function-return-type`: API 경계에서 명시적 반환형
- `no-restricted-imports`: 모놀리포지토리 경로 제한

---

## React/Next.js 통합

### 패키지

```bash
npm i -D eslint-plugin-react eslint-plugin-react-hooks eslint-plugin-jsx-a11y
```

### Flat Config 예시

```js
// eslint.config.js
import react from 'eslint-plugin-react';
import reactHooks from 'eslint-plugin-react-hooks';
import a11y from 'eslint-plugin-jsx-a11y';
import prettierRecommended from 'eslint-plugin-prettier/recommended';

export default [
  {
    files: ['**/*.{jsx,tsx}'],
    plugins: { react, 'react-hooks': reactHooks, 'jsx-a11y': a11y },
    languageOptions: {
      ecmaFeatures: { jsx: true }
    },
    rules: {
      ...react.configs.recommended.rules,
      ...a11y.configs.recommended.rules,
      // Hooks 규칙
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn'
    },
    settings: {
      react: { version: 'detect' }
    }
  },
  prettierRecommended
];
```

### Next.js

- Next는 자체 ESLint 설정을 제공(명령: `next lint`).
- 커스텀 시에도 **Prettier 충돌 제거**와 **react-hooks**, **a11y** 규칙 유지 권장.

---

## 및 테스트

### Node 전용 규칙

```js
// eslint.config.js 일부
{
  files: ['**/*.cjs', '**/*.mjs', '**/*.js'],
  languageOptions: {
    sourceType: 'module',
    globals: { process: 'readonly', __dirname: 'readonly', module: 'readonly' }
  },
  rules: {
    'no-process-exit': 'warn'
  }
}
```

### Jest / Vitest

```bash
npm i -D eslint-plugin-jest
```
```js
// eslint.config.js
import jest from 'eslint-plugin-jest';
export default [
  {
    files: ['**/*.test.{js,ts,jsx,tsx}'],
    plugins: { jest },
    rules: { ...jest.configs.recommended.rules }
  }
];
```

---

## 파일 제외/오버라이드

- **Prettier 제외**: `.prettierignore`
```
dist
coverage
*.min.js
```

- **ESLint 제외**: `.eslintignore`
```
dist
coverage
node_modules
```

- **오버라이드**(Flat Config는 `files` 블록으로 세분화):
  테스트 파일만 규칙 완화, 스크립트 디렉터리만 Node 환경 등.

---

## Husky + lint-staged로 커밋 전 자동 검사

### 설치

```bash
npm i -D husky lint-staged
npx husky install
npm set-script prepare "husky install"
```

### 훅 생성

```bash
npx husky add .husky/pre-commit "npx lint-staged"
```

### `package.json`

```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,yml,yaml,css,scss}": ["prettier --write"]
  }
}
```

> 큰 레포에서는 **lint-staged**가 변경 파일만 처리하므로 속도가 매우 빠릅니다.

---

## CI 통합(GitHub Actions 예시)

`.github/workflows/lint.yml`
```yaml
name: Lint & Format
on:
  pull_request:
  push:
    branches: [ main ]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run fmt           # Prettier check
      - run: npm run lint          # ESLint check
```

> PR에서 실패하면 즉시 피드백 → 코드 리뷰 품질과 속도가 좋아집니다.

---

## 성능 최적화·대규모 적용 노하우

- **ESLint 캐시**: `eslint . --cache --cache-location .eslintcache`
- **TS 타입 규칙 부담 줄이기**: 빌드/CI에만 `parserOptions.project`를 켜고, 로컬은 기본 규칙만으로 빠르게
- **모노레포**: 루트에 공통 Flat Config를 두고 패키지별 `files`로 범위를 분리
- **Prettier 플러그인**:
  - `prettier-plugin-tailwindcss`로 Tailwind 클래스 정렬 자동화
  - 마크다운/JSON/YAML도 포맷 일관성 유지
- **포맷 충돌 제거**: `eslint-config-prettier`가 **포맷성 ESLint 룰을 끕니다**
  (예: `indent`, `quotes`, `semi`, `max-len` 등은 Prettier에서만 관리)

---

## 흔한 오류와 해결

| 상황 | 원인 | 해결 |
|------|------|------|
| ESLint와 Prettier가 줄바꿈/따옴표로 계속 충돌 | 포맷 룰이 둘 다 켜짐 | `eslint-config-prettier` 추가, `plugin:prettier/recommended` 적용 |
| 저장 시 포맷이 안 됨 | VS Code 기본 포매터가 Prettier 아님 | `editor.defaultFormatter`를 Prettier 확장으로 지정 |
| TS에서 `no-undef` 오탐 | TS는 타입 체크로 undefined 추적, ESLint `no-undef` 불필요 | TS 파일에선 `no-undef` 비활성화 또는 TS 추천 구성을 사용 |
| 타입 규칙이 너무 느림 | `parserOptions.project`가 전체 타입 체크 | 로컬은 끄고 CI에서만 켜거나, 포함 경로를 축소 |
| Next.js에서 import 순서 정리 필요 | 정렬 일관성 미흡 | `eslint-plugin-import`와 `import/order` 규칙 또는 Prettier import 정렬 플러그인 사용 |

---

## 실전 템플릿 모음

### JS + React + Prettier(Flat Config)

```js
// eslint.config.js
import eslint from '@eslint/js';
import react from 'eslint-plugin-react';
import reactHooks from 'eslint-plugin-react-hooks';
import a11y from 'eslint-plugin-jsx-a11y';
import prettierRecommended from 'eslint-plugin-prettier/recommended';

export default [
  eslint.configs.recommended,
  {
    files: ['**/*.{js,jsx}'],
    languageOptions: {
      ecmaVersion: 'latest',
      sourceType: 'module',
      ecmaFeatures: { jsx: true }
    },
    plugins: { react, 'react-hooks': reactHooks, 'jsx-a11y': a11y },
    rules: {
      ...react.configs.recommended.rules,
      ...a11y.configs.recommended.rules,
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn'
    },
    settings: { react: { version: 'detect' } }
  },
  prettierRecommended
];
```

### TS + React + Prettier(Flat Config)

```js
// eslint.config.js
import tsParser from '@typescript-eslint/parser';
import ts from '@typescript-eslint/eslint-plugin';
import react from 'eslint-plugin-react';
import reactHooks from 'eslint-plugin-react-hooks';
import prettierRecommended from 'eslint-plugin-prettier/recommended';

export default [
  {
    files: ['**/*.{ts,tsx}'],
    languageOptions: {
      parser: tsParser,
      parserOptions: { ecmaVersion: 'latest', sourceType: 'module' }
    },
    plugins: { '@typescript-eslint': ts, react, 'react-hooks': reactHooks },
    rules: {
      ...ts.configs.recommended.rules,
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn',
      '@typescript-eslint/no-unused-vars': ['warn', { argsIgnorePattern: '^_' }]
    },
    settings: { react: { version: 'detect' } }
  },
  prettierRecommended
];
```

### + Prettier(Flat Config)

```js
// eslint.config.js
import eslint from '@eslint/js';
import prettierRecommended from 'eslint-plugin-prettier/recommended';

export default [
  eslint.configs.recommended,
  {
    languageOptions: {
      ecmaVersion: 'latest',
      sourceType: 'module',
      globals: { process: 'readonly' }
    },
    rules: {
      'no-process-exit': 'warn'
    }
  },
  prettierRecommended
];
```

---

## 마이그레이션 가이드(요약)

- 기존 `.eslintrc` → 점진적 이전:
  1) Flat Config 파일 `eslint.config.js` 생성
  2) `extends` 기반 구성을 **배열 병합** 방식으로 재현
  3) 파일 글로브별 `files` 블록으로 세분화
- 팀 합의:
  - **포맷 전부 Prettier 관리**, ESLint는 **품질 규칙**에 집중
  - 커밋 훅(Lint-Staged)과 CI에서 **일관 검증**
  - VS Code 설정을 레포에 포함해 **개발자별 환경 차이 최소화**

---

## 마무리

- **ESLint**는 버그/품질, **Prettier**는 포맷. **역할을 분리**하면 충돌 없이 깔끔합니다.
- 신형 **Flat Config**는 구성 파일을 **표준 JS로 명시**하고, **오버라이드와 모듈화**가 간결합니다.
- **Husky + lint-staged + CI**로 “변경분만 빠르게 체크 → PR에서 자동 검증” 흐름을 만들면 **코드리뷰가 본질(설계/로직)에 집중**됩니다.
- 팀/프로젝트에 맞는 **규칙 최소 집합**으로 시작해, 점진적으로 강화하세요.
