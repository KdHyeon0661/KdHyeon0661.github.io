---
layout: post
title: JavaScript - Vanilla JS vs 프레임워크
date: 2025-06-01 19:20:23 +0900
category: JavaScript
---
# 🥞 Vanilla JS vs 프레임워크: 어떤 걸 써야 할까?

---

## 1️⃣ Vanilla JS란?

> **Vanilla JS**는 프레임워크나 라이브러리를 **전혀 사용하지 않은 순수 자바스크립트 코드**를 말합니다.

### ✔ 특징

- 브라우저에서 기본 제공되는 기능만 사용
- HTML, CSS, JS를 직접 연결해서 UI 및 로직 구현
- 외부 의존성이 없어 학습 비용이 낮음
- 작은 프로젝트나 실습용에 적합

### 예시: 클릭 시 텍스트 변경

```html
<p id="text">Hello</p>
<button onclick="changeText()">변경</button>

<script>
  function changeText() {
    document.getElementById("text").innerText = "Hi";
  }
</script>
```

---

## 2️⃣ 프레임워크란?

> **프레임워크는 UI 구성, 상태 관리, 라우팅 등 웹 개발 전반을 효율적으로 처리할 수 있도록 제공하는 일종의 ‘틀’**입니다.

### 대표 프레임워크

| 이름 | 특징 |
|------|------|
| **React** | 컴포넌트 기반, Virtual DOM, 유연함 |
| **Vue** | 가볍고 직관적, 양방향 바인딩 |
| **Angular** | 대규모 앱에 적합, TypeScript 기반 |
| **Svelte** | 컴파일 타임에 최적화, 빠른 렌더링 |

### React 예시:

```jsx
import { useState } from 'react';

function App() {
  const [text, setText] = useState("Hello");
  return (
    <>
      <p>{text}</p>
      <button onClick={() => setText("Hi")}>변경</button>
    </>
  );
}
```

---

## 3️⃣ 핵심 비교

| 항목 | Vanilla JS | 프레임워크 |
|------|-------------|-------------|
| 정의 | 순수 자바스크립트 | JS 기반 UI 구성 도구 |
| 학습 곡선 | 낮음 | 높음 (개념과 구조 이해 필요) |
| 개발 속도 | 작거나 단순한 앱에 빠름 | 복잡한 앱에서 유리 |
| 유지보수 | 구조적 가이드 부족 | 구조화되어 유지보수 편리 |
| 성능 최적화 | 직접 DOM 최적화 필요 | Virtual DOM, Reactive 등 제공 |
| 생태계 | 기본 브라우저 기능만 사용 | 라우팅, 상태 관리 등 도구 풍부 |
| 코드 구조 | 자유롭고 유연 | 컴포넌트 기반으로 체계적 |
| 협업 | 어렵고 일관성 없음 | 명확한 패턴으로 협업 용이 |

---

## 4️⃣ 어떤 상황에 어떤 걸 쓸까?

| 상황 | 추천 |
|------|------|
| 빠른 프로토타입, 학습용 프로젝트 | ✅ Vanilla JS |
| DOM 조작만 필요한 간단한 UI | ✅ Vanilla JS |
| SPA, 상태 관리 필요한 앱 | ✅ React / Vue |
| 대규모 팀 프로젝트 | ✅ React / Angular |
| SEO와 SSR이 필요한 경우 | ✅ Next.js / Nuxt |
| 초기 로딩이 중요한 모바일 앱 | ✅ Svelte / Vue (가벼움) |

---

## 5️⃣ 실전 예시 비교

### ✅ Vanilla JS

```html
<input id="input" />
<p id="result"></p>
<script>
  document.getElementById("input").addEventListener("input", (e) => {
    document.getElementById("result").textContent = e.target.value;
  });
</script>
```

### ✅ React 버전

```jsx
import { useState } from 'react';

function App() {
  const [text, setText] = useState("");

  return (
    <>
      <input onChange={(e) => setText(e.target.value)} />
      <p>{text}</p>
    </>
  );
}
```

👉 Vanilla JS는 빠르고 직관적이지만, DOM 접근이 반복되고 코드가 길어지기 쉽습니다.  
👉 React는 구조화가 잘 되어 있고, 복잡한 상태 변화 관리에 유리합니다.

---

## 6️⃣ 성능 비교 (기초)

| 항목 | Vanilla JS | 프레임워크 |
|------|------------|------------|
| 초기 로딩 속도 | 빠름 | JS 번들에 따라 느려질 수 있음 |
| 런타임 성능 | 최적화 가능 | 프레임워크 자체 오버헤드 존재 |
| 메모리 사용량 | 적음 | 상대적으로 많음 |
| 렌더링 최적화 | 수동 처리 필요 | Virtual DOM, Reactive 등 자동 |

---

## 7️⃣ 유지보수와 확장성

| 항목 | Vanilla JS | 프레임워크 |
|------|------------|------------|
| 코드 구조 | 자유로움 → 혼란 가능 | 명확한 구조, 디렉토리 패턴 |
| 테스트 | 테스트 프레임워크와 직접 연동 | 라이브러리와 연동 쉬움 (Jest 등) |
| 협업 | 스타일 편차 큼 | 일관된 규칙 제공 (ESLint, Prettier, Hooks 등) |

---

## 8️⃣ 결론: 언제 무엇을 선택할까?

| 선택 기준 | 추천 도구 |
|-----------|-----------|
| 단순함, 학습용 | ✅ Vanilla JS |
| 재사용 가능한 UI, 상태 중심 | ✅ React, Vue |
| 구조화된 앱, 협업 | ✅ React, Angular |
| 로직보다 화면 중심, 빠른 구현 | ✅ Vue, Svelte |
| 복잡한 로직, 커스텀 많음 | ✅ Vanilla + 라이브러리 조합 가능 |

---

## 📌 마무리 요약

| 항목 | Vanilla JS | 프레임워크 |
|------|------------|-------------|
| 개발 방식 | 직접 모든 걸 구현 | 구성요소 기반 |
| 사용 범위 | 소규모 앱 | 중대형 앱 |
| 의존성 | 없음 | 있음 (설치 필요) |
| 초보자 진입 | 쉬움 | 개념 학습 필요 |
| 유연성 | 매우 높음 | 명확한 제약 속 유연함 |
| 유지보수 | 어렵지만 자유 | 쉽고 체계적 |

---

## 🔗 참고 링크

- [vanilla-js.com](http://vanilla-js.com/)
- [React 공식 문서](https://reactjs.org/)
- [Vue 공식 문서](https://vuejs.org/)
- [Angular 공식 문서](https://angular.io/)
- [Svelte 공식 문서](https://svelte.dev/)