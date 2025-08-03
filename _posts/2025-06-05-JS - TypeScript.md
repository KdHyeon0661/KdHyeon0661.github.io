---
layout: post
title: JavaScript - TypeScript
date: 2025-06-05 19:20:23 +0900
category: JavaScript
---
# 📘 TypeScript 도입하기: JavaScript 프로젝트를 타입 안전하게 전환하는 방법

---

## 🧩 TypeScript란?

> **TypeScript**는 Microsoft에서 개발한 JavaScript의 **상위 집합(Superset)**으로, 정적 타입을 지원하여 더 안정적이고 유지보수가 쉬운 코드를 작성할 수 있게 해주는 언어입니다.

### 왜 쓰나요?

- 🐞 **런타임 오류를 컴파일 단계에서 잡기**
- 🧠 **자동 완성, 리팩토링 지원 강화**
- 🚦 **더 명확한 API 설계와 문서화**
- 🤝 **대규모 팀 협업에 최적화**

---

## 🚀 도입 절차 개요

1. TypeScript 설치
2. tsconfig.json 설정
3. JS 파일에서 TS 파일로 점진적 변경
4. 타입 선언 적용
5. ESLint/Prettier 설정 조정

---

## 1️⃣ TypeScript 설치

```bash
npm install --save-dev typescript
```

전역 설치 시:
```bash
npm install -g typescript
```

---

## 2️⃣ tsconfig.json 설정

```bash
npx tsc --init
```

기본 설정 외에 자주 사용하는 옵션:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "strict": true,
    "esModuleInterop": true,
    "moduleResolution": "node",
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

### 주요 옵션 설명

| 옵션 | 설명 |
|------|------|
| `strict` | 엄격한 타입 검사 (최대한 켜두는 것이 좋음) |
| `esModuleInterop` | CommonJS 호환성을 높임 |
| `moduleResolution: node` | npm 패키지 찾는 방식 지정 |
| `target` | 컴파일된 JS의 대상 버전 |

---

## 3️⃣ 파일 확장자 변경: .js → .ts (또는 .tsx)

- 우선 단순한 유틸 파일부터 `.ts`로 변경
- React를 사용하는 경우 `.jsx → .tsx`로 변경

```bash
mv utils.js utils.ts
mv App.jsx App.tsx
```

초기에는 모든 파일을 한 번에 바꾸기보단 **점진적으로 진행**하는 것이 중요합니다.

---

## 4️⃣ 타입 선언 추가하기

### 변수 타입

```ts
let age: number = 25;
const name: string = "Kim";
```

### 함수 타입

```ts
function greet(name: string): string {
  return `Hello, ${name}`;
}
```

### 객체 타입

```ts
type User = {
  id: number;
  name: string;
  isAdmin?: boolean; // optional
};
```

### 배열과 튜플

```ts
const scores: number[] = [90, 80, 70];
const tuple: [string, number] = ["Age", 30];
```

---

## 5️⃣ 외부 라이브러리 타입 설정

```bash
npm install --save-dev @types/lodash
```

- 타입스크립트가 타입 정보를 알 수 있도록 `@types/*` 패키지 설치
- 대부분의 유명 라이브러리는 자동 지원되거나 `DefinitelyTyped`에 존재

---

## 6️⃣ ESLint / Prettier 설정 통합

```bash
npm install --save-dev eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

`.eslintrc.js` 예시:

```js
module.exports = {
  parser: '@typescript-eslint/parser',
  plugins: ['@typescript-eslint'],
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended'
  ]
};
```

---

## 7️⃣ 마이그레이션 전략

### ✅ 점진적 전환 전략

1. **기존 JS 파일 유지**하면서 일부 기능만 TS로 변경
2. `.js` 파일에서 `//@ts-check`로 타입 검사 시도
3. 중요 모듈부터 `.ts`로 리팩토링
4. `any` 사용은 **초기에는 허용**, 점점 제거

### 🧪 실전 팁

- `isolatedModules: true`로 모듈 단위 컴파일 가능하게 설정
- 기존 코드에 타입 적용이 어려우면 `type inference`와 `JSDoc` 활용

---

## 8️⃣ React에서의 사용 예 (with JSX/TSX)

```tsx
type ButtonProps = {
  label: string;
  onClick: () => void;
};

const Button = ({ label, onClick }: ButtonProps) => (
  <button onClick={onClick}>{label}</button>
);
```

- Props 정의 시 `interface` 또는 `type` 사용
- 상태/이벤트도 명시적 타입 지정 가능

---

## 📊 도입 장단점 요약

### ✅ 장점

- 타입 안정성 → **런타임 오류 감소**
- 개발자 도구, 자동완성, 리팩토링 향상
- 코드 문서화 효과
- 협업 시 명확한 API 계약 가능

### ❌ 단점

- 러닝 커브 존재
- 초기 전환 시 리팩토링 비용 발생
- build 시간이 JS보다 길 수 있음

---

## 🔗 유용한 도구 & 자료

| 도구/사이트 | 설명 |
|-------------|------|
| [tsconfig.dev](https://tsconfig.dev) | tsconfig 옵션 검색 |
| [TypeScript Handbook](https://www.typescriptlang.org/docs/) | 공식 문서 |
| [Type Challenges](https://github.com/type-challenges/type-challenges) | 고급 타입 학습용 문제집 |
| [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) | @types 저장소 |