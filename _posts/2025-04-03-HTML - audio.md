---
layout: post
title: HTML - audio
date: 2025-04-03 19:20:23 +0900
category: HTML
---
# 🎧 HTML5 `<audio>` 태그 완벽 가이드

HTML5에서 도입된 `<audio>` 태그는 브라우저 상에서 **오디오(음성, 음악, 효과음 등)를 직접 삽입하고 재생**할 수 있도록 지원합니다.  
Flash나 외부 플레이어 없이도 **자체 재생 기능, 제어 버튼, 자바스크립트 연동**이 가능해 매우 유용합니다.

---

## ✅ 기본 문법

```html
<audio src="sound.mp3" controls></audio>
```

또는 다양한 포맷을 지원하는 구조:

```html
<audio controls>
  <source src="sound.mp3" type="audio/mpeg">
  <source src="sound.ogg" type="audio/ogg">
  <p>브라우저가 오디오를 지원하지 않습니다.</p>
</audio>
```

> ✅ 브라우저는 첫 번째로 지원 가능한 오디오 파일을 선택합니다.

---

## 🔧 주요 속성

| 속성 | 설명 |
|------|------|
| `src` | 오디오 파일 경로 |
| `controls` | 기본 재생 컨트롤(재생/일시정지 등)을 표시 |
| `autoplay` | 페이지 로드시 자동 재생 (⚠️ muted 필요 없음) |
| `loop` | 오디오를 반복 재생 |
| `muted` | 음소거 상태로 시작 |
| `preload` | 로드 정책 지정 (`none`, `metadata`, `auto`) |

### 예제

```html
<audio src="bgmusic.mp3" controls autoplay loop preload="auto"></audio>
```

---

## 🎵 지원되는 오디오 포맷

| 포맷 | 확장자 | 설명 | 브라우저 호환성 |
|------|---------|--------|------------------|
| MP3 | `.mp3` | 손실 압축, 가장 널리 사용됨 | ✅ 모든 브라우저 |
| Ogg | `.ogg` | 오픈소스 포맷 | ❌ 일부 Safari 미지원 |
| WAV | `.wav` | 무손실 or 비압축, 고음질 | ✅ 모든 브라우저 |

> 🧠 여러 포맷을 `<source>`로 함께 제공하여 호환성 문제를 피하세요.

```html
<audio controls>
  <source src="effect.ogg" type="audio/ogg">
  <source src="effect.mp3" type="audio/mpeg">
</audio>
```

---

## 🧠 JavaScript 제어 방법

`<audio>`는 JS로 제어가 가능합니다.

```html
<audio id="player" src="sound.mp3" controls></audio>

<button onclick="document.getElementById('player').play()">▶ 재생</button>
<button onclick="document.getElementById('player').pause()">⏸ 일시정지</button>
```

### 주요 JS 속성 및 메서드

| 메서드/속성 | 설명 |
|-------------|------|
| `.play()` | 오디오 재생 시작 |
| `.pause()` | 일시 정지 |
| `.muted = true/false` | 음소거 설정 |
| `.volume = 0~1` | 볼륨 조절 |
| `.currentTime` | 현재 재생 위치 (초 단위) |
| `.duration` | 전체 길이 |
| `.loop = true/false` | 반복 재생 여부 |

---

## 🎧 preload 속성

오디오 데이터를 페이지 로딩 시 어떻게 처리할지 결정합니다.

| 값 | 설명 |
|-----|------|
| `none` | 브라우저가 로드하지 않음 |
| `metadata` | 길이 등 정보만 로드 |
| `auto` | 브라우저가 필요시 자동으로 로드 |

예제:

```html
<audio src="sound.mp3" preload="metadata"></audio>
```

---

## 🧏 접근성과 대체 콘텐츠

오디오를 재생할 수 없는 브라우저나 사용자를 위한 대체 콘텐츠를 제공하세요.

```html
<audio controls>
  <source src="song.mp3" type="audio/mpeg">
  <p>이 브라우저는 오디오를 지원하지 않습니다. <a href="song.mp3">여기에서 다운로드</a>하세요.</p>
</audio>
```

---

## 💡 실전 예제: 배경 음악 자동 재생

```html
<audio src="bg.mp3" autoplay loop></audio>
```

> 📌 자동재생은 모바일 브라우저나 최신 브라우저에서 제약이 있을 수 있습니다. 일반적으로 **사용자 상호작용 이후에만 재생**되도록 제한됩니다.

---

## ✅ `<audio>` 태그 요약 정리

| 기능 | 설명 |
|------|------|
| 🎛 컨트롤 제공 | `controls` 속성으로 기본 재생 버튼 표시 |
| 🔁 반복 재생 | `loop` 속성 사용 |
| 🔊 자동 재생 | `autoplay` 속성 사용 (주의: 일부 브라우저 제한) |
| 📡 포맷 다양화 | `<source>`로 다양한 오디오 포맷 제공 |
| 🎮 JS 제어 | `.play()`, `.pause()` 등으로 동적 제어 가능 |
| 👂 접근성 고려 | `<p>대체 텍스트 제공</p>` 또는 자막 지원 필요 |
