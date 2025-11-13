---
layout: post
title: HTML - video
date: 2025-03-30 21:20:23 +0900
category: HTML
---
# HTML5 `<video>` 태그

## 1. 기본 사용

```html
<video src="sample.mp4" controls></video>
```

여러 포맷을 동시 제공:

```html
<video controls>
  <source src="movie.mp4"  type="video/mp4">
  <source src="movie.webm" type="video/webm">
  <p>브라우저가 HTML5 비디오를 지원하지 않습니다. <a href="movie.mp4">다운로드</a></p>
</video>
```

---

## 2. 주요 속성 총정리

| 속성 | 설명 | 사용 팁 |
|---|---|---|
| `src` | 단일 소스 경로 | 다중 포맷이면 `<source>` 권장 |
| `controls` | 기본 컨트롤러 표시 | 커스텀 UI는 JS/CSS로 구현 |
| `autoplay` | 자동 재생 | 대다수 브라우저에서 **음소거(muted) 필요** |
| `muted` | 음소거 시작 | 자동재생 호환을 위해 자주 동반 |
| `loop` | 반복 재생 | 길이 짧은 안내/브랜딩 루프에 유용 |
| `poster` | 로드 전 썸네일 | 가로세로 비율을 영상과 맞추면 레이아웃 점프 방지 |
| `preload` | 로딩 힌트(`auto`/`metadata`/`none`) | 트래픽/TTI 영향, 섣불리 `auto` 남발 금지 |
| `width`/`height` | 렌더링 크기 | **CSS 우선**, HTML 속성은 초기 레이아웃 힌트 |
| `playsinline` | iOS에서 인라인 재생 | 모바일 전체화면 강제 방지 |
| `controlslist` | 컨트롤 제한 | 예: `controlslist="nodownload noplaybackrate"` |
| `disablepictureinpicture` | PiP 비활성 | DRM/저작권 컨텐츠용 |
| `crossorigin` | CORS 제어(`anonymous`/`use-credentials`) | 자막/Canvas 드로잉/미디어키 재생 시 필수적일 수 있음 |

예시:

```html
<video
  width="640" height="360"
  controls autoplay muted loop playsinline
  preload="metadata"
  poster="thumbnail.jpg"
  controlslist="nodownload noplaybackrate"
  crossorigin="anonymous">
  <source src="movie.mp4"  type='video/mp4; codecs="avc1.42E01E, mp4a.40.2"'>
  <source src="movie.webm" type='video/webm; codecs="vp9, opus"'>
</video>
```

---

## 3. `<source>`와 포맷/코덱 선택

### 3.1 컨테이너 vs 코덱

- **컨테이너**: MP4, WebM, Ogg 등 (파일 확장자)
- **비디오 코덱**: H.264/AVC, VP8/VP9, AV1 등
- **오디오 코덱**: AAC, Opus 등

브라우저 호환 관점(일반론):

| 컨테이너/코덱 | 파일 확장자 | 비디오 코덱 | 오디오 코덱 | 비고 |
|---|---|---|---|---|
| MP4 | `.mp4` | H.264(AVC) | AAC | 가장 폭넓은 호환. 필수 제공 권장 |
| WebM | `.webm` | VP8/VP9/AV1 | Opus/Vorbis | 현대 브라우저에서 우수(일부 Safari 구버전 이슈) |
| Ogg | `.ogv` | Theora | Vorbis | 구형 호환용, 요즘은 드묾 |

**권장**: MP4(H.264) + WebM(VP9 또는 AV1)을 동시 제공 → 품질/용량/호환 균형.

### 3.2 `canPlayType()`로 런타임 판단

```html
<script>
  const v = document.createElement('video');
  const mp4OK  = v.canPlayType('video/mp4; codecs="avc1.42E01E, mp4a.40.2"');
  const webmOK = v.canPlayType('video/webm; codecs="vp9, opus"');
  console.log({ mp4OK, webmOK }); // "probably" | "maybe" | ""
</script>
```

---

## 4. 모바일 자동재생 정책 요약

- **자동재생**은 기본적으로 제한. **무음(`muted`)** 상태에서만 허용되는 경우가 많음.
- iOS 인라인 재생: `playsinline` 필요.
- 사용자 제스처(탭/클릭) 후 재생은 대부분 허용.

패턴:

```html
<video autoplay muted playsinline src="intro.mp4"></video>
```

---

## 5. 자막/설명/챕터 — `<track>` 고급 활용

`WebVTT(.vtt)` 기반 텍스트 트랙을 추가:

```html
<video controls width="640">
  <source src="movie.mp4" type="video/mp4">
  <!-- 자막 -->
  <track src="subs_ko.vtt" kind="subtitles" srclang="ko" label="Korean" default>
  <track src="subs_en.vtt" kind="subtitles" srclang="en" label="English">
  <!-- 청각장애인용 폐쇄자막(사운드 이벤트 포함) -->
  <track src="captions_en.vtt" kind="captions" srclang="en" label="Captions (EN)">
  <!-- 설명 오디오 텍스트(시각장애 보조) -->
  <track src="descriptions.vtt" kind="descriptions" srclang="en" label="Audio Descriptions">
  <!-- 챕터 -->
  <track src="chapters.vtt" kind="chapters" srclang="en" label="Chapters">
  <p>브라우저가 비디오를 지원하지 않습니다.</p>
</video>
```

### 5.1 WebVTT 예시

`subs_ko.vtt`:

```
WEBVTT

00:00:00.000 --> 00:00:03.000
안녕하세요. HTML5 비디오 가이드입니다.

00:00:03.000 --> 00:00:06.500
이 트랙은 한국어 자막입니다.
```

`chapters.vtt`:

```
WEBVTT

00:00:00.000 --> 00:00:10.000
소개

00:00:10.000 --> 00:01:30.000
기본 사용

00:01:30.000 --> 00:03:00.000
고급 제어
```

### 5.2 TextTrack API로 챕터/큐 제어

```html
<video id="v" controls>
  <source src="movie.mp4" type="video/mp4">
  <track id="chap" src="chapters.vtt" kind="chapters" default>
</video>
<script>
  const v = document.getElementById('v');
  v.addEventListener('loadedmetadata', () => {
    for (const track of v.textTracks) {
      if (track.kind === 'chapters') {
        track.mode = 'hidden';
        for (const cue of track.cues) {
          console.log('Chapter:', cue.text, cue.startTime, cue.endTime);
        }
      }
    }
  });
</script>
```

---

## 6. 자바스크립트로 제어 — 속성/메서드/이벤트

### 6.1 빠른 스타터

```html
<video id="vid" src="movie.mp4" controls preload="metadata"></video>
<button id="play">재생</button>
<button id="pause">일시정지</button>
<input  id="seek" type="range" min="0" max="100" value="0">
<script>
  const v = document.getElementById('vid');
  const play = document.getElementById('play');
  const pause = document.getElementById('pause');
  const seek = document.getElementById('seek');

  play.onclick  = () => v.play();
  pause.onclick = () => v.pause();

  v.addEventListener('loadedmetadata', () => {
    seek.max = Math.floor(v.duration);
  });
  v.addEventListener('timeupdate', () => {
    seek.value = Math.floor(v.currentTime);
  });
  seek.oninput = () => { v.currentTime = seek.valueAsNumber; };
</script>
```

### 6.2 핵심 속성/메서드

| 항목 | 설명 |
|---|---|
| `play()` / `pause()` | 재생/일시정지 |
| `currentTime` | 현재 재생 위치(초) |
| `duration` | 총 길이(초, 알 수 없으면 `NaN`) |
| `paused`, `ended` | 상태 플래그 |
| `muted`, `volume` | 음소거/볼륨(0~1) |
| `playbackRate` | 재생 속도(예: 0.5, 1.0, 2.0) |
| `loop`, `autoplay`, `controls` | 표준 속성 |

### 6.3 자주 쓰는 이벤트

| 이벤트 | 타이밍 |
|---|---|
| `loadedmetadata` | 메타데이터 준비(길이/해상도) |
| `canplay`/`canplaythrough` | 끊김 없이 재생 가능 추정 |
| `timeupdate` | 재생 위치 갱신 |
| `seeking`/`seeked` | 탐색 시작/종료 |
| `ended` | 재생 완료 |
| `error` | 로딩/디코딩 오류 |

오류 핸들링:

```js
v.addEventListener('error', () => {
  const e = v.error;
  console.error('MediaError', e && e.code);
});
```

---

## 7. 전체화면, Picture-in-Picture, Media Session

### 7.1 전체화면 API

```js
const wrap = document.querySelector('.player');
document.getElementById('fs').onclick = () => {
  if (!document.fullscreenElement) wrap.requestFullscreen();
  else document.exitFullscreen();
};
```

```html
<button id="fs">전체화면</button>
```

### 7.2 Picture-in-Picture

```js
const v = document.getElementById('vid');
document.getElementById('pip').onclick = async () => {
  if (document.pictureInPictureElement) {
    await document.exitPictureInPicture();
  } else if (!v.disablePictureInPicture) {
    await v.requestPictureInPicture();
  }
};
```

```html
<button id="pip">PiP</button>
```

### 7.3 Media Session (잠금화면/OS 미디어 컨트롤 연동)

```js
if ('mediaSession' in navigator) {
  navigator.mediaSession.metadata = new MediaMetadata({
    title: '샘플 영상', artist: '제작자', album: '데모', artwork: [{ src: 'cover.png', sizes: '512x512' }]
  });
  navigator.mediaSession.setActionHandler('play',  () => v.play());
  navigator.mediaSession.setActionHandler('pause', () => v.pause());
  navigator.mediaSession.setActionHandler('seekforward',  () => v.currentTime += 10);
  navigator.mediaSession.setActionHandler('seekbackward', () => v.currentTime -= 10);
}
```

---

## 8. 반응형 비디오 — 레이아웃/비율 유지

### 8.1 CSS `aspect-ratio` 권장

```html
<style>
  .video-wrap {
    max-width: 960px;
    margin: 0 auto;
  }
  video.rwd {
    width: 100%;
    height: auto;
    aspect-ratio: 16 / 9; /* 영상 비율 고정 */
    display: block;
    background: #000; /* 로딩 전 블랙 바탕 */
  }
</style>

<div class="video-wrap">
  <video class="rwd" controls poster="preview.jpg">
    <source src="movie.mp4" type="video/mp4">
  </video>
</div>
```

### 8.2 `object-fit`으로 크롭 모드

```css
video.cover { width:100%; height:60vh; object-fit:cover; }
video.contain { width:100%; height:auto; object-fit:contain; }
```

---

## 9. 시나리오별 레시피

### 9.1 히어로 헤더 자동재생(무음/루프/인라인)

```html
<video class="cover" autoplay muted loop playsinline
       preload="none" poster="hero.jpg">
  <source src="hero.mp4" type="video/mp4">
</video>
```

### 9.2 교육 영상: 다국어 자막 + 챕터 + 키보드 접근

```html
<video id="course" controls width="100%" preload="metadata">
  <source src="course.mp4" type="video/mp4">
  <track src="chapters.vtt" kind="chapters" default>
  <track src="subs_ko.vtt" kind="subtitles" srclang="ko" label="Korean">
  <track src="subs_en.vtt" kind="subtitles" srclang="en" label="English">
</video>
<script>
  const v = document.getElementById('course');
  window.addEventListener('keydown', (e) => {
    if (e.code === 'Space') { e.preventDefault(); v.paused ? v.play() : v.pause(); }
    if (e.code === 'ArrowRight') v.currentTime += 5;
    if (e.code === 'ArrowLeft')  v.currentTime -= 5;
  });
</script>
```

### 9.3 지연 로딩(IntersectionObserver로 의사 lazy-load)

```html
<video id="promo" controls preload="none" poster="promo.jpg" width="100%"></video>
<script>
  const el = document.getElementById('promo');
  const load = () => {
    if (el.children.length === 0) {
      const s = document.createElement('source');
      s.src = 'promo.mp4';
      s.type = 'video/mp4';
      el.appendChild(s);
      el.load();
    }
  };
  const io = new IntersectionObserver((entries, obs) => {
    entries.forEach(e => {
      if (e.isIntersecting) { load(); obs.disconnect(); }
    });
  }, { rootMargin: '200px' });
  io.observe(el);
</script>
```

---

## 10. 스트리밍(HLS/DASH)과 MSE/DRM 개요

### 10.1 HLS(HTTP Live Streaming), DASH
- **어댑티브 비트레이트**로 네트워크 환경에 맞게 품질 조정.
- Safari는 **HLS를 네이티브** 지원. 그 외 브라우저는 MSE(Media Source Extensions) 기반 라이브러리 사용.

### 10.2 hls.js를 이용한 HLS 로드(비-Safari)

```html
<video id="live" controls playsinline></video>
<script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
<script>
  const v = document.getElementById('live');
  const src = 'stream.m3u8';
  if (v.canPlayType('application/vnd.apple.mpegurl')) {
    v.src = src; // Safari
  } else if (Hls.isSupported()) {
    const hls = new Hls();
    hls.loadSource(src);
    hls.attachMedia(v);
    hls.on(Hls.Events.MANIFEST_PARSED, () => v.play());
  } else {
    v.outerHTML = '<p>스트리밍을 지원하지 않는 브라우저입니다.</p>';
  }
</script>
```

### 10.3 DRM(Encrypted Media Extensions, EME)
- **EME** API를 통해 Widevine/PlayReady/FairPlay 등과 통신.
- `<video>` 자체는 동일, 라이선스 서버와 키 교환 로직이 추가됨(플레이어 라이브러리 사용 권장).

---

## 11. CORS/보안/정책

| 주제 | 요약 | 실무 팁 |
|---|---|---|
| CORS | 다른 출처의 영상/자막/포스터 로드 시 CORS 필요 | 서버에 `Access-Control-Allow-Origin` 설정, `crossorigin` 속성 맞춤 |
| 캔버스 드로잉 | `<video>` 프레임을 `<canvas>`에 그리면 교차 출처 제한 | CORS 통과 및 `crossorigin="anonymous"` 필수 |
| 자동재생 | 사용자 경험 보호 정책 존재 | `autoplay muted playsinline` 조합, 실패 시 `play()` Promise 캐치 |
| 다운로드 억제 | 완전 차단 불가 | `controlslist="nodownload"`은 UI 힌트일 뿐, 네트워크로는 항상 내려받음 |
| 개인정보 | 리퍼러/쿠키 전송 | 필요 시 `referrerpolicy`, CORS credentials 정책 정합성 확인 |
| 접근성 | 자막/대본 제공 | 필수 콘텐츠는 `captions`/텍스트 대본 제공 권장 |

---

## 12. 성능 최적화 체크리스트

- [ ] `preload="metadata"` 기본, 히어로/Above-the-fold가 아니면 **지연 로딩**(IO)
- [ ] 포맷/코덱을 **화질 대비 용량 균형**으로 선택(MP4+WebM/AV1)
- [ ] 모바일 데이터 절약: 짧은 프리롤, **저해상도 프리뷰**(LQIP) + 인터랙션 후 고해상도 로딩
- [ ] `poster` 제공으로 **레이아웃 점프 방지**
- [ ] `aspect-ratio`로 CLS 최소화
- [ ] 스트리밍은 HLS/DASH 적용(긴/라이브/대규모 트래픽)
- [ ] 캐싱 헤더와 범위 요청 지원(서버)
- [ ] 동일 출처·CORS 세팅을 사전에 점검(자막/포스터/썸네일)

---

## 13. 자주 하는 실수와 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| 자동재생 실패 | 음소거 없음 | `autoplay muted playsinline` |
| 포스터가 안 보임 | 이미 `autoplay`로 즉시 재생 | 시작 전만 표시. 필요 시 JS로 `pause()` 후 수동 재생 |
| 자막이 안 뜸 | 파일 경로/`srclang`/MIME 미스 | `.vtt` 경로·헤더·언어코드 확인, CORS 허용 |
| WebM이 Safari에서 재생 안 됨 | 코덱/브라우저 미지원 | **MP4(H.264)** 동시 제공 |
| Canvas에 그리면 오염 상태 | 교차 출처 미허용 | 서버 CORS + `crossorigin="anonymous"` |

---

## 14. 완성형 예제: 반응형, 다국어 자막, PiP/전체화면, Media Session

```html
<!doctype html>
<html lang="ko">
<head>
<meta charset="utf-8">
<title>완전한 HTML5 비디오 플레이어</title>
<style>
  .player { max-width: 960px; margin: 2rem auto; }
  .player video { width: 100%; aspect-ratio: 16/9; background:#000; display:block; }
  .toolbar { display:flex; gap:.5rem; margin-top:.5rem; flex-wrap:wrap; }
  .toolbar button, .toolbar input[type="range"] { padding:.4rem .6rem; }
  .spacer { flex:1 }
</style>
</head>
<body>
  <div class="player">
    <video id="v"
      preload="metadata" controls playsinline
      poster="preview.jpg" crossorigin="anonymous"
      controlslist="nodownload">
      <source src="movie.mp4"  type='video/mp4; codecs="avc1.42E01E, mp4a.40.2"'>
      <source src="movie.webm" type='video/webm; codecs="vp9, opus"'>
      <track src="subs_ko.vtt" kind="subtitles" srclang="ko" label="Korean" default>
      <track src="subs_en.vtt" kind="subtitles" srclang="en" label="English">
      <p>비디오를 지원하지 않습니다. <a href="movie.mp4">다운로드</a></p>
    </video>

    <div class="toolbar">
      <button id="play">재생/일시정지</button>
      <label>재생위치 <input id="seek" type="range" min="0" max="100" value="0"></label>
      <label>속도
        <select id="rate">
          <option>0.5</option><option selected>1</option><option>1.25</option><option>1.5</option><option>2</option>
        </select>
      </label>
      <button id="mute">음소거</button>
      <div class="spacer"></div>
      <button id="pip">PiP</button>
      <button id="fs">전체화면</button>
    </div>
  </div>

<script>
const v = document.getElementById('v');
const play = document.getElementById('play');
const seek = document.getElementById('seek');
const rate = document.getElementById('rate');
const mute = document.getElementById('mute');

play.onclick = async () => { v.paused ? await v.play() : v.pause(); };
mute.onclick = () => { v.muted = !v.muted; };

v.addEventListener('loadedmetadata', () => { seek.max = Math.floor(v.duration || 0); });
v.addEventListener('timeupdate', () => { seek.value = Math.floor(v.currentTime || 0); });
seek.oninput = () => { v.currentTime = seek.valueAsNumber; };
rate.onchange = () => { v.playbackRate = parseFloat(rate.value); };

// Picture-in-Picture
document.getElementById('pip').onclick = async () => {
  try {
    if (document.pictureInPictureElement) await document.exitPictureInPicture();
    else if (!v.disablePictureInPicture) await v.requestPictureInPicture();
  } catch (e) { console.warn('PiP 실패', e); }
};

// Fullscreen
document.getElementById('fs').onclick = async () => {
  try {
    if (!document.fullscreenElement) await v.requestFullscreen();
    else await document.exitFullscreen();
  } catch(e){ console.warn('FS 실패', e); }
};

// Media Session
if ('mediaSession' in navigator) {
  navigator.mediaSession.metadata = new MediaMetadata({
    title: '데모 영상', artist: '제작자', album: 'HTML5 Guide',
    artwork: [{ src: 'cover.png', sizes: '512x512', type: 'image/png' }]
  });
  navigator.mediaSession.setActionHandler('play',  () => v.play());
  navigator.mediaSession.setActionHandler('pause', () => v.pause());
  navigator.mediaSession.setActionHandler('seekforward',  () => v.currentTime += 10);
  navigator.mediaSession.setActionHandler('seekbackward', () => v.currentTime -= 10);
}
</script>
</body>
</html>
```

---

## 15. 요약

- **필수 포맷**: MP4(H.264) + 보조로 WebM(VP9/AV1) 병행 제공.
- **모바일 자동재생**: `autoplay muted playsinline`.
- **자막/챕터**: `<track>` + WebVTT, TextTrack API로 확장.
- **제어/UX**: 기본 컨트롤 + 전체화면 + PiP + Media Session.
- **스트리밍**: HLS/DASH 필요 시 MSE 플레이어(hls.js 등) 사용.
- **보안/CORS**: 교차 출처 로딩은 서버 CORS와 `crossorigin` 정합성 확보.
- **성능**: `preload="metadata"` 기본, IO로 지연 로딩, `aspect-ratio`로 CLS 방지.

위의 원칙과 레시피를 조합하면, 플러그인 없이도 **고품질·접근성·성능**을 모두 충족하는 HTML5 비디오 경험을 제공할 수 있다.
