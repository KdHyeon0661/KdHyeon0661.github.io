---
layout: post
title: CSS - Sass와 CSS
date: 2025-05-12 21:20:23 +0900
category: CSS
---
# 🎯 Sass와 CSS의 차이점 완벽 정리

웹 개발에서 CSS는 스타일링의 기본 언어이고, Sass는 CSS를 더 강력하게 만들어 주는 **CSS 전처리기(Preprocessor)**입니다.  
두 언어는 사용 목적과 기능에서 큰 차이가 있으며, 이해하면 더 효율적인 스타일링 작업이 가능합니다.

---

## 1. 기본 개념

| 구분   | CSS                                  | Sass (Syntactically Awesome Stylesheets)              |
|--------|-------------------------------------|-------------------------------------------------------|
| 정의   | 웹 표준 스타일 시트 언어               | CSS를 확장한 전처리기 언어, 컴파일하여 CSS로 변환됨      |
| 실행   | 브라우저가 직접 해석                   | 컴파일 과정 필요, Sass → CSS 변환 후 브라우저 적용       |
| 문법   | 표준 CSS 문법 사용                    | 확장된 문법(SCSS, Sass 스타일) 지원                      |

---

## 2. 주요 기능 차이

| 기능                  | CSS                                      | Sass                                    |
|-----------------------|------------------------------------------|-----------------------------------------|
| 변수 사용             | 불가능 (CSS 변수는 제한적 지원)             | `$변수명: 값;` 형태로 강력한 변수 사용 가능          |
| 중첩(Nesting)          | 불가능                                  | 선택자, 속성 중첩 가능으로 코드 간결화               |
| 믹스인(Mixins)         | 불가능                                  | 재사용 가능한 스타일 블록 정의 가능                  |
| 함수(Functions)        | 제한적 (CSS 함수만 가능)                   | 사용자 정의 함수 작성 가능                             |
| 연산(Operation)        | `calc()` 등의 제한적 지원                   | 산술 연산 가능 (`+`, `-`, `*`, `/`)                   |
| 조건문과 반복문        | 불가능                                  | `@if`, `@for`, `@each`, `@while` 등 제어문 지원     |
| 임포트(import)         | `@import` (브라우저에 직접 호출되어 비효율적) | `@use`, `@forward` 등 모듈화 및 최적화된 import 지원 |
| 출력 형식 설정         | 고정                                     | 컴파일 시 `compressed`, `expanded` 등 선택 가능       |

---

## 3. 예제 비교

### 3.1 변수 사용

**CSS:**

```css
:root {
  --main-color: #3498db;
}
button {
  background-color: var(--main-color);
}
```

**Sass:**

```scss
$main-color: #3498db;

button {
  background-color: $main-color;
}
```

---

### 3.2 중첩(Nesting)

**CSS:**

```css
nav ul {
  list-style: none;
}
nav ul li {
  display: inline-block;
}
nav ul li a {
  color: #333;
}
```

**Sass:**

```scss
nav {
  ul {
    list-style: none;

    li {
      display: inline-block;

      a {
        color: #333;
      }
    }
  }
}
```

---

### 3.3 믹스인(Mixin)과 함수(Function)

```scss
@mixin flex-center {
  display: flex;
  justify-content: center;
  align-items: center;
}

.button {
  @include flex-center;
  padding: 1rem 2rem;
}
```

---

## 4. Sass 사용 이유

- **재사용성 증가**: 변수, 믹스인, 함수로 코드 반복 최소화
- **유지보수 용이**: 중첩과 모듈화로 복잡한 스타일 관리 쉬움
- **생산성 향상**: 제어문과 연산 지원으로 동적 스타일 가능
- **컴파일 최적화**: 다양한 출력 포맷과 임포트 방식 제공

---

## 5. CSS 변수와의 차이점

| 구분                | CSS 변수 (`--var`)                 | Sass 변수 (`$var`)                 |
|---------------------|-----------------------------------|----------------------------------|
| 처리 시점            | 런타임 (브라우저 해석 시)          | 컴파일 타임 (개발 단계)           |
| 재할당               | 가능 (JS, media query 내 등)       | 불가능 (컴파일 후 고정)           |
| 상속 및 범위          | 상속되고 동적으로 변경 가능         | 스코프가 컴파일 시 고정됨          |
| 브라우저 지원         | 최신 브라우저 지원                  | 모든 환경 지원 (컴파일 결과 CSS)  |

---

## 6. 정리

| 구분         | CSS                            | Sass                         |
|--------------|--------------------------------|------------------------------|
| 문법         | 표준 CSS                      | 확장된 문법 (SCSS, Sass)     |
| 변수 지원    | CSS 변수 제한적 지원           | 강력한 변수 지원              |
| 기능 확장    | 제한적                       | 중첩, 믹스인, 함수, 조건문 등 다수 기능 |
| 실행 환경    | 브라우저                      | 컴파일러 필요 (Node, Ruby 등) |
| 유지보수성   | 단순, 규모 커지면 관리 어려움  | 대규모 프로젝트에 적합        |

---

## 7. 참고 링크

- [Sass 공식 사이트](https://sass-lang.com/)
- [MDN - CSS Variables](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties)
- [Sass vs CSS 비교](https://css-tricks.com/sass-vs-css/)