---
layout: post
title: CSS - CSS Reset vs Normalize
date: 2025-05-06 22:20:23 +0900
category: CSS
---
# 🧼 CSS Reset vs Normalize.css

CSS를 작성할 때, **브라우저마다 기본 스타일이 달라** 동일한 코드라도 결과가 다르게 나타나는 경우가 많습니다.  
이런 문제를 방지하기 위해 사용하는 것이 바로 **CSS Reset**과 **Normalize.css**입니다.

---

## ✅ 공통 목적

| 항목        | 설명 |
|-------------|------|
| 🎯 목적       | 브라우저 간의 기본 스타일 차이를 없애거나 일관되게 맞추는 것 |
| 📦 사용 시기   | 프로젝트 시작 시 CSS 최상단에 포함 |
| 🧩 방식 차이   | Reset은 **기본 스타일을 제거**, Normalize는 **일관되게 맞춤** |

---

## 🔄 CSS Reset이란?

**모든 브라우저의 기본 스타일을 초기화**하는 기법입니다.

### 📌 대표적인 Reset 코드 (Eric Meyer’s Reset)

```css
/* Eric Meyer's Reset CSS v2.0 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
html, body {
  height: 100%;
}
ul, ol {
  list-style: none;
}
a {
  text-decoration: none;
  color: inherit;
}
button {
  background: none;
  border: none;
}
```

### ✅ 장점

- 기본 스타일 제거로 **완전한 제어권 확보**
- 디자인이 완전히 커스터마이징된 프로젝트에 유리

### ❌ 단점

- 유용한 브라우저 기본 스타일까지 제거됨 (ex. `button`, `h1`, `input`)
- 일부 접근성 문제 발생 가능성 있음

---

## ⚖ Normalize.css란?

브라우저 기본 스타일을 제거하지 않고, **일관성 있게 유지하고 보정하는 방식**입니다.  
[Normalize.css 공식 깃허브](https://github.com/necolas/normalize.css)

### 📌 사용 방법

```html
<link rel="stylesheet" href="https://necolas.github.io/normalize.css/8.0.1/normalize.css">
```

### ✅ 특징

- HTML 요소의 **표준 스타일을 보정하여 통일**
- `h1`~`h6`, `form`, `input`, `button` 등 **사용성 있는 기본 스타일 유지**
- 접근성과 가독성 유지

### ✅ 장점

- **시맨틱한 태그 구조 유지**에 유리
- **접근성(A11Y)**을 해치지 않음
- 브라우저마다 미묘한 차이를 최소화

### ❌ 단점

- Reset처럼 완전히 초기화되진 않음
- 디자인을 완전 커스터마이징하려면 오히려 수정할 코드가 더 많을 수 있음

---

## 🧪 비교 정리

| 항목            | CSS Reset                          | Normalize.css                            |
|-----------------|-------------------------------------|-------------------------------------------|
| 목적            | 기본 스타일 **완전 초기화**         | 스타일을 **보정 및 일관성 유지**         |
| 접근성          | 기본 스타일 제거로 손상 가능성 있음 | 접근성 유지                                |
| UI 요소 스타일  | 전부 제거 (form, h1 등)             | 최대한 유지                               |
| 컨트롤 정도     | 매우 높음                            | 중간 정도                                  |
| 적합한 상황     | 완전 커스텀 UI                      | 기존 HTML 요소 활용 중심 프로젝트         |

---

## 🧠 실무 팁: 둘을 혼합한 Reset + Normalize 전략

많은 프레임워크는 **reset과 normalize를 적절히 섞어서** 사용합니다.  
예: Tailwind CSS, Bootstrap도 자체 리셋 및 정규화를 제공

### 예시: Custom Reset + Normalize 스타일

```css
/* 기본 Reset */
*, *::before, *::after {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

/* Normalize 일부 */
button {
  font: inherit;
  border: none;
  background: none;
  cursor: pointer;
}

input, select, textarea {
  font: inherit;
  color: inherit;
}
```

---

## ✅ 실제 프레임워크 사용 예

| 프레임워크     | 초기화 방식                      |
|----------------|----------------------------------|
| Tailwind CSS   | `preflight` (Normalize 기반 커스텀 리셋) |
| Bootstrap      | Reboot (Reset + Normalize 혼합)  |
| Foundation     | Normalize.css 사용                |

---

## 🔚 결론

| 질문                           | 선택 기준 |
|--------------------------------|------------|
| UI를 완전히 내 손으로 만들 것인가? | ✔ CSS Reset |
| 브라우저 스타일도 활용할 것인가? | ✔ Normalize.css |
| 중간 경로를 원한다면?           | ✔ 둘을 혼합한 커스텀 방식 |

---

## 🔗 참고 자료

- [Normalize.css 공식 문서](https://necolas.github.io/normalize.css/)
- [Eric Meyer’s Reset CSS](https://meyerweb.com/eric/tools/css/reset/)
- [MDN - 브라우저 스타일 차이](https://developer.mozilla.org/ko/docs/Web/CSS/Universal_selectors)