---
layout: post
title: HTML - video
date: 2025-03-30 21:20:23 +0900
category: HTML
---
# 🎬 HTML5 `<video>` 태그 완벽 가이드

HTML5에서는 웹 페이지에 **플러그인 없이 비디오를 직접 삽입**할 수 있도록 `<video>` 태그를 도입했습니다.  
이제 YouTube나 Flash 없이도 동영상을 웹에 삽입하고 제어할 수 있습니다.

이 글에서는 `<video>` 태그의 **기본 문법부터 고급 사용법까지** 자세히 설명합니다.

---

## ✅ 기본 문법

```html
<video src="https://www.w3schools.com/html/mov_bbb.mp4" controls></video>
```

혹은 여러 포맷을 지원하는 방식:

```html
<video controls>
  <source src="https://www.w3schools.com/html/mov_bbb.mp4" type="video/mp4">
  <source src="./../assets/video/movie.webm" type="video/webm">
  <p>지원되지 않는 브라우저입니다.</p>
</video>
```

---

## 🔧 주요 속성 정리

| 속성 | 설명 |
|------|------|
| `src` | 비디오 파일 경로 (단일 파일일 때) |
| `controls` | 재생/일시정지 등의 컨트롤러 표시 |
| `autoplay` | 페이지 로드시 자동 재생 (⚠️ muted 필요) |
| `muted` | 음소거 상태로 시작 |
| `loop` | 반복 재생 |
| `poster` | 영상 로드 전 보여줄 이미지 (썸네일) |
| `preload` | 비디오를 미리 불러올지 지정 (`auto`, `metadata`, `none`) |
| `width`, `height` | 비디오 크기 지정 |
| `playsinline` | iOS에서 전체화면 없이 재생 (모바일 대응) |

### 예제

```html
<video src="sample.mp4" width="640" height="360" controls autoplay muted loop poster="thumbnail.jpg"></video>
```

---

## 🎥 `<source>` 태그 사용 이유

- 브라우저마다 지원하는 비디오 포맷이 다르기 때문에 `<source>` 태그를 **여러 개 중첩**하여 다양한 포맷을 지원하는 것이 일반적입니다.

```html
<video controls>
  <source src="https://www.w3schools.com/html/mov_bbb.mp4" type="video/mp4">
  <source src="./../assets/video/movie.webm" type="video/webm">
  <p>해당 브라우저는 비디오를 지원하지 않습니다.</p>
</video>
```

> ✅ 브라우저는 지원 가능한 형식을 자동으로 선택합니다.

---

## 🌍 브라우저 호환성과 포맷

| 포맷 | 확장자 | 브라우저 지원 |
|------|---------|----------------|
| MP4 (H.264) | `.mp4` | ✅ 모든 브라우저 |
| WebM (VP8/VP9) | `.webm` | ✅ 대부분 (Safari 제외 일부) |
| Ogg (Theora) | `.ogv` | ⚠️ 일부만 지원 |

> ⚠️ Safari의 경우 WebM 지원이 제한적일 수 있으므로 **MP4는 필수**로 제공해야 합니다.

---

## 🧏 자막 추가 – `<track>` 태그

HTML5에서는 자막도 `<video>` 내부에 추가할 수 있습니다.

```html
<video controls>
  <source src="https://www.w3schools.com/html/mov_bbb.mp4" type="video/mp4">
  <track src="subtitles.vtt" kind="subtitles" srclang="en" label="English">
</video>
```

| 속성 | 설명 |
|------|------|
| `src` | 자막 파일 (VTT 형식) |
| `kind` | `subtitles`, `captions`, `descriptions` 등 |
| `srclang` | 언어 코드 (`en`, `ko`, `jp` 등) |
| `label` | 사용자 선택 시 표시될 라벨 이름 |

---

## 🧠 JavaScript로 제어하기

비디오는 DOM 요소로 접근 가능하여 JS로 제어할 수 있습니다.

```html
<video id="myVideo" src="sample.mp4" controls></video>

<button onclick="document.getElementById('myVideo').play()">▶ 재생</button>
<button onclick="document.getElementById('myVideo').pause()">⏸ 일시정지</button>
```

### 주요 메서드/속성

| 항목 | 설명 |
|------|------|
| `.play()` | 재생 시작 |
| `.pause()` | 재생 멈춤 |
| `.currentTime` | 현재 재생 위치 (초 단위) |
| `.duration` | 전체 길이 |
| `.muted = true/false` | 음소거 설정 |
| `.volume = 0~1` | 볼륨 조절 |
| `.loop = true/false` | 반복 설정 |

---

## 🖼 `poster` 속성으로 썸네일 지정

```html
<video src="https://www.w3schools.com/html/mov_bbb.mp4" controls poster="thumbnail.png"></video>
```

- `poster`는 영상이 **로드되기 전 보여줄 이미지**입니다.
- 사용자에게 콘텐츠의 미리보기를 제공할 수 있습니다.

---

## ⚙️ preload 속성

```html
<video src="sample.mp4" controls preload="metadata"></video>
```

| 값 | 설명 |
|-----|------|
| `none` | 로딩하지 않음 |
| `metadata` | 길이 등 메타 정보만 로드 |
| `auto` | 브라우저가 필요하다고 판단하면 로드 |

---

## 🛡 접근성과 UX 팁

- `<video>`는 자막(`<track>`)을 제공하면 **접근성(Accessibility)** 향상에 도움이 됩니다.
- 자막 없이 청각장애인이 이해하기 어려우므로 필수 영상에는 제공을 권장합니다.
- `autoplay`는 **muted** 없이 작동하지 않는 브라우저가 많으므로 함께 사용합니다.
```html
<video autoplay muted>
```

---

## 📌 실전 예제: 반응형 비디오 플레이어

```html
<video controls width="100%" poster="preview.jpg">
  <source src="https://www.w3schools.com/html/mov_bbb.mp4" type="video/mp4">
  <source src="./../assets/video/movie.webm" type="video/webm">
  <track src="subs_ko.vtt" kind="subtitles" srclang="ko" label="Korean">
  <p>브라우저가 비디오를 지원하지 않습니다.</p>
</video>
```

---

## ✅ 마무리

HTML5의 `<video>` 태그는 간단한 속성만으로도 **강력한 동영상 플레이어 기능**을 제공하며,  
자막, 썸네일, JS 제어, 다양한 포맷 지원 등으로 **웹 콘텐츠의 멀티미디어 경험을 향상**시킵니다.

> 🎯 잘 구성된 `<video>` 태그는 UX 향상, 접근성 보장, 반응형 디자인에 모두 기여할 수 있습니다.
