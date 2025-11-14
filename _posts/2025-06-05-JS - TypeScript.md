---
layout: post
title: JavaScript - TypeScript
date: 2025-06-05 19:20:23 +0900
category: JavaScript
---
# TypeScript 도입하기: JavaScript 프로젝트를 타입 안전하게 전환하는 방법

## TL;DR (요약 체크리스트)

- `npm i -D typescript` → `npx tsc --init` 로 **기본 세팅**
- **엄격 모드(`strict`) + 최소 예외**로 시작, 막히는 곳은 `@ts-expect-error`로 국소 완충
- **점진 도입**: `//@ts-check` → JSDoc 타입 → `.ts/.tsx` 변환 → `any` 제거
- **번들/테스트 통합**: Vite/Webpack + SWC/ESBuild, Vitest/Jest 설정
- **ESLint/Prettier**와 충돌 제거, CI에서 `tsc --noEmit` 로 타입 검증 필수화
- **경로 별칭/모노레포**는 `baseUrl/paths`, Project References로 관리
- **서드파티 타입**: `@types/*`/JSDoc/모듈 선언으로 빈틈 메우기
- **점진적 목표치**: “타입 커버리지/any 잔량/노이즈 룰”을 팀 합의로 수치화

---

## TypeScript란? 왜 지금 도입해야 하는가

> TypeScript는 JavaScript의 **상위 집합**(Superset)으로, 정적 타입 시스템을 제공하여
> **런타임 이전**에 오류를 포착하고, 에디터 지능(Autocomplete/Refactor/Go-to-Definition)을 강화한다.

### 기대 효과 (현실적으로)

- 리팩토링 비용 절감: “용감한 리네임” 가능
- API 계약(Contract) 문서화: 타입이 곧 사양
- 온보딩 속도 향상: 타입 정의를 따라가며 학습
- 테스트 보완: 정적 분석으로 **테스트 사각지대** 축소

---

## 설치와 최소 tsconfig 템플릿

```bash
npm i -D typescript
npx tsc --init
```

`tsconfig.json` (권장 시작 템플릿)

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2020", "DOM"],
    "jsx": "react-jsx",
    "strict": true,
    "noImplicitOverride": true,
    "noUnusedLocals": true,
    "noUnusedParameters": false,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "allowSyntheticDefaultImports": true,
    "useUnknownInCatchVariables": true
  },
  "include": ["src"],
  "exclude": ["dist", "node_modules"]
}
```

> **핵심 옵션**
> - `strict`: 엄격 모드 (가능하면 초반부터 ON)
> - `moduleResolution: "Bundler"`: Vite/SWC/ESBuild와 자연스러운 해석
> - `exactOptionalPropertyTypes`: 선택 속성의 **정확한 의미** 보장
> - `useUnknownInCatchVariables`: `catch (e)`의 타입을 `unknown`으로 안전하게

---

## 점진적 마이그레이션 전략 (중요)

### 가장 쉬운 시작: JS 유지 + 타입 검사

**파일은 그대로 JS**로 두고, 상단에 체크만 켠다.

```js
// @ts-check

/**
 * @param {number} a
 * @param {number} b
 */
export function add(a, b) {
  return a + b;
}
```

- 에디터에서 즉시 타입 검사
- **JSDoc**으로 타입을 표현해 코드 변경 최소화

### 혼합 모드: 일부만 `.ts/.tsx`

- 유틸/순수 함수부터 `.ts` 변환
- React라면 컴포넌트 1~2개만 `.tsx`로 시작
- 막히는 곳은 한시적으로 `any` / `@ts-ignore` → **`@ts-expect-error`** 권장

```ts
// @ts-expect-error - 외부 SDK가 아직 타입 미제공. TODO: replace with official types
const sdk = window.ExternalSdk;
```

### “타입 부채” 관리

- `any` 개수, `@ts-expect-error` 줄 수를 대시보드화
- 스프린트마다 **부채 상환 티켓** 10~20% 유지

---

## 도메인 타입 설계 패턴

### 타입 vs 인터페이스

- **상호 확장/선언 병합**이 필요하면 `interface`, 변형/조합은 `type` 선호
- 팀 룰로 통일하되, 라이브러리 인터페이스 확장에는 `interface`가 편리

```ts
// 인터페이스 확장
interface User {
  id: string;
  email: string;
}
interface Admin extends User {
  roles: string[];
}
```

```ts
// 유니온/유틸 조합이 많으면 type
type OrderState = "pending" | "paid" | "shipped" | "canceled";
type Result<T> = { ok: true; value: T } | { ok: false; error: string };
```

### 유틸리티 타입 적극 사용

```ts
type PartialUser = Partial<User>;
type ReadonlyUser = Readonly<User>;
type Picked = Pick<User, "id" | "email">;
type WithId<T> = T & { id: string };
```

### 좁히기(Narrowing)와 타입 가드

```ts
function isResultOk<T>(r: Result<T>): r is { ok: true; value: T } {
  return r.ok;
}
```

### 브랜드 타입(도메인 안전)

```ts
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<string, "UserId">;
const asUserId = (s: string) => s as UserId;

function findUser(id: UserId) {/*...*/}
findUser(asUserId("u_123"));
// findUser("u_123") // 컴파일 에러
```

---

## React/JSX에서 TSX로 전환

### 기본 패턴

```tsx
type ButtonProps = {
  label: string;
  onClick?: () => void;
  as?: "button" | "a";
};

export const Button = ({ label, onClick, as = "button" }: ButtonProps) => {
  if (as === "a") return <a onClick={onClick}>{label}</a>;
  return <button onClick={onClick}>{label}</button>;
};
```

### 이벤트, ref, 제네릭 컴포넌트

```tsx
type InputProps<T extends HTMLInputElement | HTMLTextAreaElement> = {
  as?: "input" | "textarea";
  ref?: React.Ref<T>;
} & React.ComponentPropsWithoutRef<"input">;

export function FancyInput<T extends HTMLInputElement = HTMLInputElement>(
  { as = "input", ...rest }: InputProps<T>
) {
  const Comp = as as any;
  return <Comp {...rest} />;
}
```

### 상태 관리 라이브러리와 타입

- React Query/Zustand/Redux Toolkit: **훅 반환형**/스토어 셀렉터의 타입 명시
- `infer`로 API 응답 자동 파생

```ts
type ApiResponse = Awaited<ReturnType<typeof fetchUser>>;
type User = ApiResponse["user"];
```

---

## 외부 라이브러리 타입

### DefinitelyTyped

```bash
npm i -D @types/lodash
```

### 임시 선언(모듈 보강)

```ts
// src/types/global.d.ts
declare module "some-legacy-sdk" {
  export function connect(apiKey: string): any;
}
```

### JSDoc로 외부 스크립트 난국 완화

{% raw %}
```js
// @ts-check
/** @type {{ track: (ev: string, props?: Record<string, unknown>) => void }} */
const Analytics = window.Analytics;
```
{% endraw %}

---

## 빌드/번들 통합: Vite, Webpack, SWC, ESBuild

### “타입체크”와 “트랜스파일” 분리 원칙

- **빠른 개발 빌드**: SWC/ESBuild로 트랜스파일만
- **타입 검증**: 병렬 `tsc --noEmit` 또는 `vite-plugin-checker`

#### Vite 예시

```bash
npm i -D vite @vitejs/plugin-react vite-plugin-checker
```

```ts
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import checker from "vite-plugin-checker";

export default defineConfig({
  plugins: [
    react(),
    checker({ typescript: true })
  ]
});
```

#### Webpack 예시

```bash
npm i -D ts-loader fork-ts-checker-webpack-plugin
```

```js
// webpack.config.js
const ForkTsCheckerWebpackPlugin = require('fork-ts-checker-webpack-plugin');

module.exports = {
  entry: './src/index.tsx',
  module: {
    rules: [{ test: /\.tsx?$/, use: 'ts-loader', exclude: /node_modules/ }]
  },
  plugins: [new ForkTsCheckerWebpackPlugin()],
  resolve: { extensions: ['.ts', '.tsx', '.js'] }
};
```

---

## 테스트: Vitest/Jest + TS

### Vitest (권장: Vite와 궁합 우수)

```bash
npm i -D vitest @vitest/coverage-v8 ts-node
```

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
export default defineConfig({
  test: {
    environment: "jsdom",
    coverage: { reporter: ["text", "html"] }
  }
});
```

```ts
// src/add.test.ts
import { describe, it, expect } from "vitest";
import { add } from "./add";
describe("add", () => {
  it("1+2=3", () => expect(add(1,2)).toBe(3));
});
```

### Jest

```bash
npm i -D jest ts-jest @types/jest
npx ts-jest config:init
```

```js
// jest.config.js
module.exports = {
  preset: "ts-jest",
  testEnvironment: "jsdom",
};
```

---

## ESLint/Prettier와 충돌 없이 쓰기

```bash
npm i -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin eslint-config-prettier prettier
```

```js
// .eslintrc.cjs
module.exports = {
  root: true,
  parser: "@typescript-eslint/parser",
  plugins: ["@typescript-eslint"],
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "prettier"
  ],
  parserOptions: { ecmaVersion: "latest", sourceType: "module" },
  ignorePatterns: ["dist", "node_modules"]
};
```

```json
// .prettierrc
{ "semi": true, "singleQuote": false, "trailingComma": "all" }
```

> 포맷은 Prettier, 규칙은 ESLint. **eslint-config-prettier**로 충돌 제거.

---

## 경로 별칭과 모노레포

### 경로 별칭

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@src/*": ["src/*"],
      "@components/*": ["src/components/*"]
    }
  }
}
```

Vite/Webpack도 동일 별칭을 plugin/resolve에 반영.

### Project References (대규모/모노레포)

루트 `tsconfig.json`:
```json
{
  "files": [],
  "references": [{ "path": "./packages/ui" }, { "path": "./packages/app" }]
}
```

`packages/ui/tsconfig.json`:
```json
{
  "compilerOptions": { "composite": true, "declaration": true, "outDir": "dist" },
  "include": ["src"]
}
```

`packages/app/tsconfig.json`:
```json
{
  "compilerOptions": { "composite": true },
  "references": [{ "path": "../ui" }],
  "include": ["src"]
}
```

- `composite: true` + `references`로 **증분 빌드**와 의존성 추적
- CI에서 `tsc -b`로 전체/변경분 빌드

---

## 런타임 안전 보강 (Zod/Valibot + TS)

정적 타입만으로 부족한 **런타임 입력 검증**은 스키마로 보강.

```ts
import { z } from "zod";

const CreateUser = z.object({
  email: z.string().email(),
  age: z.number().int().min(0).optional(),
});
type CreateUser = z.infer<typeof CreateUser>;

export function createUser(body: unknown) {
  const input = CreateUser.parse(body); // 런타임 검증
  // input은 타입 안전
}
```

- API 층에서 **스키마 → 타입 자동화**로 안정성과 DX 동시 확보

---

## 빌드/배포 파이프라인 예(웹 + 서버/서버리스)

### 웹 앱 (Vite)

- Dev: Vite HMR + `vite-plugin-checker`
- CI: `npm run typecheck` (`tsc --noEmit`), `vitest run --coverage`, `vite build`
- 배포: Static hosting (Vercel/Netlify/S3+CF)

### Node/서버리스

- 트랜스파일: `tsup`/`esbuild`/`swc`
- 타입 검증: `tsc --noEmit`
- 배포: Vercel Functions/Cloudflare Workers/AWS Lambda

`package.json` 스크립트 예시
```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "typecheck": "tsc --noEmit",
    "test": "vitest",
    "lint": "eslint . --ext .ts,.tsx",
    "format": "prettier -w ."
  }
}
```

---

## 흔한 에러/함정과 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| `Cannot find module` | 경로 별칭 불일치 | tsconfig `paths`와 번들러 alias 동기화 |
| `implicitly any` 폭발 | `strict` 켰는데 타입 미정의 다수 | JSDoc 임시 보완, `any` 국소 허용 후 점진 제거 |
| `esModuleInterop` 관련 | CJS/ESM 혼용 | `esModuleInterop: true`, `allowSyntheticDefaultImports: true` |
| DOM 타입 미인식 | `lib` 누락 | `lib: ["DOM", "ES2020"]` |
| 느린 빌드 | 전 프로젝트 타입체크 | `tsc --noEmit`를 CI로, 로컬은 `vite-plugin-checker`/폴더 단위 |

---

## 팀 규약과 지표로 굳히기

- **규약**: `strict` 유지, `any` 허용 정책, 외부 경계에서만 `unknown→narrowing`
- **지표**:
  - any 개수, `@ts-expect-error` 개수
  - 타입 커버리지(예: `ts-prune`/ESLint 규칙 조합)
  - CI 막대기: **typecheck 실패 → 배포 차단**
- **리뷰 가이드**: 타입 제거/간결화(PR 크기 제한), 도메인 타입은 별도 파일(`types/`)로 공유

---

## 예제: JS → TS 전환(작게 시작해 보기)

### 기존 JS 유틸

```js
// src/price.js
export function toKRW(amount, { currency = "KRW" } = {}) {
  return new Intl.NumberFormat("ko-KR", { style: "currency", currency }).format(amount);
}
```

### JSDoc로 타입 힌트

{% raw %}
```js
// @ts-check

/**
 * @param {number} amount
 * @param {{currency?: string}} [opts]
 */
export function toKRW(amount, opts = {}) {
  const currency = opts.currency ?? "KRW";
  return new Intl.NumberFormat("ko-KR", { style: "currency", currency }).format(amount);
}
```
{% endraw %}

### `.ts` 변환

```ts
// src/price.ts
type FormatOpts = { currency?: string };

export function toKRW(amount: number, opts: FormatOpts = {}) {
  const currency = opts.currency ?? "KRW";
  return new Intl.NumberFormat("ko-KR", { style: "currency", currency }).format(amount);
}
```

### 사용처에서 안전한 오용 차단

```ts
toKRW(1000);               // OK
// toKRW("1000");          // 컴파일 에러
toKRW(1000, { currency: "USD" }); // OK
```

---

## 고급: 제네릭, 조건부 타입, 매핑 타입 한 방에

```ts
// 엔드포인트 응답 매핑
type Endpoints = {
  "/me": { id: string; name: string };
  "/posts": { id: string; title: string }[];
};

type Fetcher = <P extends keyof Endpoints>(path: P) => Promise<Endpoints[P]>;

const fetcher: Fetcher = async (path) => {
  const res = await fetch(path);
  return res.json();
};

async function demo() {
  const me = await fetcher("/me");       // { id: string; name: string }
  const posts = await fetcher("/posts"); // { id; title }[]
}
```

```ts
// API 입력 검증 + 출력 타입 연결
import { z } from "zod";
const CreatePost = z.object({ title: z.string().min(1) });
type CreatePost = z.infer<typeof CreatePost>;
```

---

## 마이그레이션 로드맵(예시)

1. **주 1~2일**: JSDoc + `//@ts-check`로 경고 줄이기
2. 1차 전환: **공통 유틸/퓨어 함수**를 `.ts`로 변경
3. 2차 전환: **도메인 모델 타입** 정의 → 컴포넌트/서비스 레이어에 적용
4. 3차 전환: **API 경계에 스키마**(Zod) 도입, `unknown` → 안전 변환
5. 4차 전환: **경로 별칭/프로젝트 레퍼런스**로 구조화
6. 5차: **CI/PR Gate**로 typecheck 강제, any/예외 카운트 지표화
7. 안정화: any 제거/리팩토링 스프린트 주기적으로 편성

---

## 결론

TypeScript 도입은 “한 번에”가 아니라 **지속 가능한 속도로 부채를 상환**하며 **DX와 안정성을 동반 상승**시키는 장기 전략이다.
**작게 시작**하되, CI 게이트/팀 규약/지표로 **일관성**을 확보하면, 전환 비용보다 **리팩토링·협업·품질 향상 이득**이 더 크다.

---

## 부록 A. 자주 쓰는 스니펫

```ts
// 안전한 JSON 파서
export function safeJson<T = unknown>(s: string): { ok: true; value: T } | { ok: false; error: string } {
  try { return { ok: true, value: JSON.parse(s) as T }; }
  catch (e) { return { ok: false, error: (e as Error).message }; }
}
```

```ts
// fetch 래퍼 (입출력 타입 바인딩)
export async function getJson<T>(url: string, init?: RequestInit): Promise<T> {
  const r = await fetch(url, init);
  if (!r.ok) throw new Error(`HTTP ${r.status}`);
  return r.json() as Promise<T>;
}
```

```ts
// 불변 업데이트 유틸
export const update = <T, K extends keyof T>(obj: T, key: K, val: T[K]): T =>
  ({ ...obj, [key]: val });
```

---

## 참고 자료

- 공식 문서: https://www.typescriptlang.org/docs/
- tsconfig 옵션 안내: https://tsconfig.dev
- DefinitelyTyped: https://github.com/DefinitelyTyped/DefinitelyTyped
- Type Challenges: https://github.com/type-challenges/type-challenges
- Zod: https://github.com/colinhacks/zod
- Vitest: https://vitest.dev/
- Vite: https://vitejs.dev/
