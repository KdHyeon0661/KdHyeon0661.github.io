---
layout: post
title: CSS - Animate.css
date: 2025-05-02 21:20:23 +0900
category: CSS
---
# 🎬 애니메이션 라이브러리 소개: Animate.css 완전 가이드

**Animate.css**는 미리 정의된 CSS 애니메이션을 손쉽게 적용할 수 있게 도와주는 **경량 CSS 라이브러리**입니다.  
단 몇 줄의 클래스만으로도 **부드럽고 강력한 애니메이션 효과**를 웹사이트에 바로 사용할 수 있습니다.

---

## ✅ 1. Animate.css란?

- Daniel Eden이 만든 **오픈소스 CSS 애니메이션 라이브러리**
- 다양한 요소에 **손쉽게 애니메이션 적용 가능**
- **Vanilla HTML + CSS** 환경에서도 바로 사용 가능
- CDN 및 NPM 모두 지원

---

## ✅ 2. 설치 방법

### ▶️ 1) CDN으로 사용 (가장 간단한 방법)

```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/animate.css/4.1.1/animate.min.css" />
```

> `<head>` 안에 삽입하면 바로 사용 가능

---

### ▶️ 2) npm으로 설치 (프로젝트에 통합 시)

```bash
npm install animate.css --save
```

그리고 CSS에 추가:

```css
@import 'animate.css';
```

---

## ✅ 3. 기본 사용 방법

애니메이션을 적용할 HTML 요소에 아래와 같이 클래스를 추가:

```html
<div class="animate__animated animate__bounce">
  Hello!
</div>
```

- `animate__animated`: Animate.css 작동을 위한 **공통 클래스 (필수)**
- `animate__bounce`: 적용할 **애니메이션 이름**

---

## ✅ 4. 자주 사용하는 애니메이션 목록

| 애니메이션 유형 | 클래스 이름                  |
|------------------|-------------------------------|
| 주목/주의 끌기  | `animate__bounce`, `animate__flash`, `animate__pulse`, `animate__shakeX` |
| 페이드 효과     | `animate__fadeIn`, `animate__fadeOut`, `animate__fadeInUp`, `animate__fadeInLeft` |
| 슬라이드        | `animate__slideInDown`, `animate__slideOutRight` |
| 줌               | `animate__zoomIn`, `animate__zoomOut` |
| 회전            | `animate__rotateIn`, `animate__rotateOut` |
| 빙글빙글        | `animate__flip`, `animate__flipInX`, `animate__flipOutY` |

---

## ✅ 5. 추가 옵션 클래스

Animate.css는 몇 가지 유틸리티 클래스를 함께 제공해서 **속도, 반복 등 조절**이 가능합니다.

| 클래스명                   | 기능 설명                          |
|----------------------------|-------------------------------------|
| `animate__slow`            | 애니메이션 속도 느리게 (`2s`)     |
| `animate__slower`          | 더 느리게 (`3s`)                  |
| `animate__fast`            | 빠르게 (`800ms`)                  |
| `animate__faster`          | 더 빠르게 (`500ms`)               |
| `animate__infinite`        | 무한 반복                         |
| `animate__delay-2s`        | 2초 지연                          |
| `animate__repeat-2`        | 두 번 반복                        |

> ex: `animate__animated animate__fadeIn animate__slow animate__delay-1s`

---

## ✅ 6. 실전 예제

### 🎯 요소가 페이드 인 + 아래에서 위로 나타남

```html
<div class="animate__animated animate__fadeInUp animate__delay-1s">
  Welcome to our site!
</div>
```

---

### 🎯 버튼에 hover 시 애니메이션 적용

```html
<button class="btn" onmouseover="this.classList.add('animate__animated','animate__pulse')">
  Hover Me
</button>
```

※ JS로 `hover` 시 애니메이션 클래스 추가 → 애니메이션 실행  
※ 끝나고 `animationend` 이벤트로 제거하면 한 번씩만 재실행 가능

```javascript
const btn = document.querySelector('.btn');
btn.addEventListener('animationend', () => {
  btn.classList.remove('animate__animated', 'animate__pulse');
});
```

---

## ✅ 7. JavaScript 연동 (스크롤 시 애니메이션 등)

Animate.css는 단독으로는 **자동 트리거**를 지원하지 않기 때문에,  
스크롤 등장 시 실행하려면 다음과 같은 라이브러리와 함께 사용해야 함:

- [AOS (Animate On Scroll)](https://michalsnik.github.io/aos/)
- [WOW.js](https://wowjs.uk/)
- [Intersection Observer API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API)

예시 (WOW.js):

```html
<div class="wow animate__animated animate__fadeInUp"></div>

<script src="wow.min.js"></script>
<script>
  new WOW().init();
</script>
```

---

## ✅ 8. 커스터마이징

Animate.css의 소스코드는 SASS로 작성되어 있어 **사용하지 않을 애니메이션만 제거하거나**  
필요한 것만 빌드하는 것도 가능

- GitHub: https://github.com/animate-css/animate.css

---

## ✅ 9. 성능 주의 사항

- 너무 많은 요소에 애니메이션을 동시에 적용하면 **브라우저 성능 저하**
- 가능한 한 `transform`, `opacity` 기반 애니메이션 사용 권장
- `animation-fill-mode: both;`로 잔상 처리 가능

---

## 📌 요약 정리

| 항목            | 내용 요약                                           |
|------------------|----------------------------------------------------|
| 설치 방법        | CDN, npm 모두 지원                                 |
| 기본 구조        | `animate__animated` + `animate__이름` 클래스 조합 |
| 애니메이션 종류  | bounce, fade, zoom, rotate, slide 등 다수 지원     |
| 속도/반복 제어   | `animate__slow`, `animate__infinite` 등으로 설정   |
| 스크롤 연동      | WOW.js, AOS.js, Intersection Observer로 가능      |

---

## 🔗 참고 자료

- [Animate.css 공식 홈페이지](https://animate.style/)
- [GitHub 저장소](https://github.com/animate-css/animate.css)
- [WOW.js (Scroll 연동)](https://wowjs.uk/)
- [AOS.js (Scroll 기반)](https://michalsnik.github.io/aos/)

---

## 💡 마무리 팁

- 애니메이션은 UI에 **생명력과 사용자 경험**을 더해줍니다.
- 단, **의도 없이 과도한 사용은 혼란과 피로감을 유발**할 수 있으니 **사용 목적을 분명히!**
- Animate.css는 빠르게 적용 가능한 도구이지만, 커스터마이징이나 최적화가 필요한 경우에는 직접 `@keyframes`를 사용하는 것도 고려하세요.