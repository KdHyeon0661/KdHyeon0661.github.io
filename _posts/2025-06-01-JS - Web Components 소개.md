---
layout: post
title: JavaScript - Web Components 소개
date: 2025-06-01 22:20:23 +0900
category: JavaScript
---
# 🧩 Web Components 소개: 표준 기반 UI 컴포넌트 개발

---

## 📌 Web Components란?

> **Web Components는 재사용 가능한 UI 컴포넌트를 만들기 위한 브라우저 표준 기술의 모음입니다.**

React, Vue 같은 프레임워크 없이도, 순수 자바스크립트로 **독립적인 HTML 태그를 생성**할 수 있게 해주는 기술입니다.

---

## 🔧 구성 요소 3가지

Web Components는 아래 3가지 핵심 기술로 구성됩니다:

| 기술 | 설명 |
|------|------|
| ✅ **Custom Elements** | 직접 만든 HTML 태그 정의 |
| ✅ **Shadow DOM** | 컴포넌트의 내부 DOM을 캡슐화 |
| ✅ **HTML Templates** | 재사용 가능한 템플릿 정의 |

---

## 1️⃣ Custom Elements

> **사용자 정의 태그**를 만들 수 있는 API입니다.

```js
class MyButton extends HTMLElement {
  connectedCallback() {
    this.innerHTML = `<button>Click me</button>`;
  }
}

customElements.define('my-button', MyButton);
```

```html
<my-button></my-button>
```

- `HTMLElement`를 상속한 클래스를 정의
- `connectedCallback()`은 DOM에 붙을 때 호출
- `customElements.define()`로 등록

---

## 2️⃣ Shadow DOM

> 컴포넌트 내부 DOM을 **외부와 격리**시켜 스타일이나 구조가 충돌하지 않도록 합니다.

```js
class MyCard extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `
      <style>
        p { color: red; }
      </style>
      <p>This is shadow DOM</p>
    `;
  }
}
customElements.define('my-card', MyCard);
```

```html
<my-card></my-card>
```

- 내부 스타일은 외부 CSS에 영향을 받지 않음
- `mode: 'open'`이면 외부 JS에서 접근 가능 (`.shadowRoot`)

---

## 3️⃣ HTML Templates

> 미리 정의된 **템플릿 HTML 조각**을 문서에 저장하고, 필요 시 복사해서 사용할 수 있습니다.

```html
<template id="my-template">
  <style>
    h1 { color: green; }
  </style>
  <h1>Welcome</h1>
</template>
```

```js
class WelcomeBox extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    const template = document.getElementById('my-template');
    const content = template.content.cloneNode(true);
    shadow.appendChild(content);
  }
}
customElements.define('welcome-box', WelcomeBox);
```

```html
<welcome-box></welcome-box>
```

---

## 🧪 실전 예제: 간단한 토글 버튼 만들기

```js
class ToggleSwitch extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' }).innerHTML = `
      <style>
        button { padding: 10px; }
      </style>
      <button>OFF</button>
    `;
  }

  connectedCallback() {
    const btn = this.shadowRoot.querySelector('button');
    btn.addEventListener('click', () => {
      btn.textContent = btn.textContent === 'OFF' ? 'ON' : 'OFF';
    });
  }
}
customElements.define('toggle-switch', ToggleSwitch);
```

```html
<toggle-switch></toggle-switch>
```

---

## ⚖️ 장점과 단점

### ✅ 장점

- ✅ 표준 API → **모든 브라우저** 지원 (Chrome, Edge, Safari, Firefox 등)
- ✅ 캡슐화 → 스타일 충돌 방지
- ✅ 프레임워크 독립적
- ✅ 진입장벽 낮음 (HTML/CSS/JS만 필요)
- ✅ 재사용성 높음

### ❌ 단점

- ❌ React/Vue처럼 선언적 렌더링이나 상태 관리 없음
- ❌ SEO, SSR 지원 미약
- ❌ 복잡한 앱에서는 생산성 부족
- ❌ Shadow DOM의 디버깅이 어려울 수 있음

---

## 🧩 Web Components vs 프레임워크 컴포넌트

| 항목 | Web Components | React/Vue 컴포넌트 |
|------|----------------|---------------------|
| 기반 | 표준 API | 프레임워크 의존 |
| 렌더링 방식 | imperative (명령형) | declarative (선언형) |
| 상태 관리 | 없음 (직접 구현) | 프레임워크 지원 |
| 스타일 격리 | Shadow DOM | CSS Modules, Scoped CSS |
| 생태계 | 작음 | 크고 성숙함 |
| SSR | 제한적 | 공식 지원 |

---

## 🔌 Web Components + 프레임워크도 가능!

React, Vue 등에서도 Web Component를 사용할 수 있습니다.

```jsx
<MyWebComponent some-prop="value" />
```

단, **이벤트 바인딩, 속성 전달** 방식이 다르므로 약간의 호환 작업 필요

---

## 🌍 브라우저 지원 현황

| 브라우저 | 지원 여부 |
|----------|------------|
| Chrome | ✅ 완전 지원 |
| Edge | ✅ 완전 지원 |
| Firefox | ✅ 거의 완전 지원 |
| Safari | ✅ 완전 지원 |
| IE11 | ❌ 미지원 |

---

## 📚 참고 링크

- [Web Components 공식 문서](https://developer.mozilla.org/en-US/docs/Web/Web_Components)
- [customElements - MDN](https://developer.mozilla.org/en-US/docs/Web/API/CustomElementRegistry/define)
- [Shadow DOM - MDN](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_shadow_DOM)
- [HTML Templates - MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/template)

---

## ✅ 마무리 요약

- Web Components는 **재사용 가능한 HTML 요소를 표준으로 만드는 기술**
- **Custom Elements, Shadow DOM, Templates**가 핵심
- 프레임워크 없이 UI를 모듈화할 수 있는 강력한 도구
- 규모가 크지 않은 앱, 디자인 시스템, 위젯 등에 적합