---
layout: post
title: JavaScript - 모듈 시스템 (import, export)
date: 2025-05-04 20:20:23 +0900
category: JavaScript
---
# 자바스크립트 모듈 시스템 (import, export) 완전 정복

자바스크립트는 처음에는 모듈 개념이 없었지만, 규모가 커진 애플리케이션을 위해 다양한 **모듈 시스템**이 등장했고,  
최근에는 표준인 **ES Modules (ESM)**이 공식적으로 도입되었습니다.

이 글에서는 자바스크립트의 모듈 개념과 `import`, `export` 사용법을 포함한 **ESM 중심 모듈 시스템**을 정리합니다.

---

## ✅ 1. 모듈이란?

> 모듈은 독립적인 기능 단위의 **자바스크립트 파일**입니다.

모듈을 사용하면 코드를 다음처럼 나눌 수 있습니다:

- 파일 간 **기능 분리 및 재사용성 향상**
- **글로벌 변수 오염 방지**
- **코드 유지보수 용이**

---

## ✅ 2. ES6 모듈 시스템 (ESM: ECMAScript Modules)

### 특징
- 파일 단위로 작동
- `strict mode` 자동 적용
- **정적 구조** → 빌드 타임에 `import`/`export` 파악 가능

---

## ✅ 3. export 문법

### ➤ 1) Named Export (여러 개 가능)

```js
// math.js
export const PI = 3.14159;
export function add(a, b) {
  return a + b;
}
```

- 여러 개를 내보낼 수 있음
- 이름 그대로 가져와야 함

### ➤ 2) Default Export (파일당 하나)

```js
// logger.js
export default function log(message) {
  console.log("LOG:", message);
}
```

- 한 파일에 하나만 가능
- `import` 시 이름을 자유롭게 정할 수 있음

---

## ✅ 4. import 문법

### ➤ 1) Named Import

```js
import { PI, add } from "./math.js";

console.log(PI); // 3.14159
console.log(add(2, 3)); // 5
```

- `{}` 안에 정확한 이름으로 가져와야 함
- 이름 바꾸기:

```js
import { add as sum } from "./math.js";
```

### ➤ 2) Default Import

```js
import log from "./logger.js";

log("Hello"); // LOG: Hello
```

- 이름은 자유롭게 지정 가능

### ➤ 3) 전체 가져오기

```js
import * as math from "./math.js";

console.log(math.PI);
console.log(math.add(1, 2));
```

---

## ✅ 5. export 문법 요약

| 구분 | 문법 |
|------|------|
| Named Export | `export const`, `export function` 등 |
| Default Export | `export default` |
| 여러 개 내보내기 | `export { a, b as c }` |
| 한 줄에서 내보내기 | `export { PI, add };` |

---

## ✅ 6. 모듈은 파일 단위로 동작

- `.js` 파일 자체가 모듈로 취급
- `<script type="module">` 태그로 HTML에서 사용 가능:

```html
<script type="module" src="main.js"></script>
```

- 모듈은 **지연 실행(deferred)** → DOMContentLoaded 이전에 실행되지 않음
- **`this`는 undefined**, **strict mode** 적용됨

---

## ✅ 7. 브라우저에서 모듈 사용 시 주의사항

- `type="module"`이 반드시 필요
- 같은 도메인 또는 CORS 허용된 경로여야 함
- 파일 확장자 `.js` 반드시 명시해야 함 (`./utils.js`, ❌ `./utils`)
- 상대 경로 또는 절대 경로만 허용 (`node_modules`처럼 생략 불가)

---

## ✅ 8. Node.js에서 모듈 사용

Node.js는 다음 두 가지 모듈 시스템을 지원합니다:

| 시스템 | 확장자 | 선언 방식 | 사용 방식 |
|--------|--------|------------|-----------|
| CommonJS | `.js` (기본) | `require()` / `module.exports` | 전통 방식 |
| ESM     | `.mjs` 또는 `package.json`에 `"type": "module"` 설정 | `import` / `export` | 최신 방식 |

### CommonJS 예시

```js
// math.js
module.exports = {
  add: (a, b) => a + b
};

// main.js
const math = require("./math");
console.log(math.add(1, 2));
```

### ES Modules (Node.js >= v12)

```js
// math.mjs
export const add = (a, b) => a + b;

// main.mjs
import { add } from './math.mjs';
console.log(add(1, 2));
```

> `package.json`에 `"type": "module"`을 추가하면 `.js`에서도 ESM 가능

---

## ✅ 9. 동적 import (Dynamic Import)

> 실행 중에 모듈을 불러오는 문법

```js
button.addEventListener("click", async () => {
  const module = await import("./analytics.js");
  module.track();
});
```

- 조건부로 모듈을 불러올 수 있음
- 반환값은 **Promise**

---

## ✅ 10. 모듈 번들러와 모듈 시스템

실제 대규모 애플리케이션에서는 모듈 번들러를 사용하여 모듈을 하나의 파일로 결합합니다.

### 대표적인 번들러
- **Webpack**
- **Vite**
- **Rollup**
- **Parcel**

---

## ✅ 11. 마무리 요약

| 개념           | 설명 |
|----------------|------|
| Named Export   | 여러 개 가능, 이름으로 가져옴 |
| Default Export | 한 개만 가능, 자유 이름으로 가져옴 |
| import         | 필요한 방식에 따라 선택 |
| 모듈 실행 시점 | defer됨, strict mode 적용 |
| 브라우저에서 사용 | `type="module"`, 확장자 명시 필요 |
| Node.js        | CommonJS vs ES Modules 둘 다 지원 |

---

## 📌 보너스: 자주 하는 실수

- `import`/`export`는 **파일 최상단에서만 사용 가능**
- 브라우저에서 `.js` 확장자 생략 시 오류 발생
- `default`는 중괄호 없이 가져와야 함