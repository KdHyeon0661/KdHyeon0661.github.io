---
layout: post
title: HTML - audio
date: 2025-04-03 19:20:23 +0900
category: HTML
---
# HTML5 `<audio>` 태그

## 1. `<audio>` 한눈에 보기

- HTML5의 네이티브 오디오 플레이어.
- **플러그인 없이** 재생/일시정지/탐색/볼륨 등 제공.
- **여러 포맷을 동시 제공**하면 브라우저가 재생 가능한 첫 번째 소스를 선택.
- JS DOM API로 **정교한 제어/시각화** 가능.

### 기본 문법

```html
<audio src="sound.mp3" controls></audio>
```

여러 포맷 제공(권장):

```html
<audio controls>
  <source src="sound.mp3" type="audio/mpeg">
  <source src="sound.ogg" type="audio/ogg">
  브라우저가 오디오를 지원하지 않습니다. <a href="sound.mp3">다운로드</a>
</audio>
```

---

## 2. 핵심 속성 총정리

| 속성 | 의미/기능 | 비고 |
|---|---|---|
| `src` | 단일 소스 파일 경로 | `<source>` 여러 개 쓰면 `src` 생략 가능 |
| `controls` | 기본 UI 표시 | 네이티브 컨트롤 |
| `autoplay` | 자동 재생 시도 | **대부분의 브라우저는 사용자 제스처 없이 소리 나는 자동재생을 차단** |
| `muted` | 음소거 시작 | 자동재생 정책 회피에 유리(그러나 오디오는 음소거 자동재생의 UX가 제한적) |
| `loop` | 반복 재생 | 끝 → 처음 자동 루프 |
| `preload` | 로드 힌트 (`none`/`metadata`/`auto`) | 다수의 플레이어가 있을 땐 `metadata` 또는 `none` 권장 |
| `crossorigin` | 교차 출처 로딩 정책 (`anonymous`/`use-credentials`) | Web Audio/Canvas로 파이프라인 연결 시 필요 |
| `controlslist` | UI 제어 힌트(예: `nodownload noplaybackrate`) | 브라우저마다 지원 상이, **완전한 다운로드 방지는 아님** |
| `playsinline` | (주로 비디오용) 인라인 재생 힌트 | 오디오에는 의미 적음 |

### 예시(정석형)

```html
<audio controls preload="metadata" crossorigin="anonymous">
  <source src="track.opus" type='audio/webm; codecs="opus"'>
  <source src="track.mp3"  type="audio/mpeg">
  브라우저가 오디오를 지원하지 않습니다. <a href="track.mp3">다운로드</a>
</audio>
```

---

## 3. 브라우저 자동재생 정책 정확히 이해하기

- **기본 원칙**: 사용자의 명시적 상호작용(클릭/탭) **이전에는** “소리가 나는” 미디어의 자동재생을 차단.
- **음소거(`muted`)** 상태는 자동재생 허용 범위가 넓지만, **오디오를 음소거 재생하는 UX는 실효성 낮음**.
- **사용자 제스처 이후**(버튼 클릭 등)에는 `.play()`가 정상 동작.
- **모바일 Safari** 등은 자동재생 정책이 특히 엄격. 배경 음악은 **사용자 동작으로 시작**하도록 설계.

권장 패턴:

```html
<button id="start">재생 시작</button>
<audio id="bgm" src="bg.mp3" loop preload="none"></audio>
<script>
  document.getElementById('start').addEventListener('click', async () => {
    const a = document.getElementById('bgm');
    try { await a.play(); }
    catch (e) { console.warn('재생 실패(정책/네트워크):', e); }
  });
</script>
```

---

## 4. 지원 포맷과 MIME

| 포맷 | 확장자 | MIME | 특성/비고 |
|---|---|---|---|
| MP3 | `.mp3` | `audio/mpeg` | 범용 호환, 기본 제공 권장 |
| Ogg Vorbis | `.ogg` | `audio/ogg` | 오픈, Safari 옛 버전 주의 |
| Opus | `.opus` or `.webm` | `audio/ogg` 또는 `audio/webm` | 저비트 우수, 보조 제공 추천 |
| WAV | `.wav` | `audio/wav` | 무손실/무압축, 대용량 |

서버(MIME) 예시(Nginx):

```nginx
types {
  audio/mpeg  mp3;
  audio/ogg   ogg opus;
  audio/wav   wav;
}
```

---

## 5. `preload` 전략과 성능

| 값 | 동작 | 권장 상황 |
|---|---|---|
| `none` | 즉시 로드 안 함 | 목록형 플레이어 다수, 데이터 절약 |
| `metadata` | 길이·코덱 등 메타만 | **기본 추천** |
| `auto` | 브라우저 재량 로드 | 짧고 곧바로 들려줄 가능성이 높을 때만 |

추가 최적화:
- **많은 오디오**를 한 페이지에 둘 경우: 모두 `preload="none"` + 첫 상호작용에 `src` 주입/`load()` 호출.
- **캐시/범위 요청**(Range) 활성화: 긴 트랙 탐색 성능 향상.
- `<link rel="preload" as="audio" href="...">`로 선제 로드 힌트(주요 브라우저에서 시험적으로 활용).

---

## 6. 이벤트/상태 흐름 이해 (디버깅 핵심)

주요 이벤트:

| 이벤트 | 의미 |
|---|---|
| `loadedmetadata` | 길이/트랙 등 메타 획득 |
| `loadeddata` | 첫 프레임(버퍼) 사용 가능 |
| `canplay` | 재생 시작 가능(짧은 버퍼) |
| `canplaythrough` | 중단 없이 재생할 만큼 충분 |
| `play` / `playing` | 재생 시작/실행 중 |
| `pause` | 일시정지 |
| `timeupdate` | 재생 위치 변경 |
| `ended` | 재생 끝 |
| `waiting`/`stalled` | 버퍼 부족/중단 |
| `error` | 네트워크/디코딩 오류 |

로깅 예시:

```html
<audio id="p" controls preload="metadata">
  <source src="song.mp3" type="audio/mpeg">
</audio>
<script>
  const p = document.getElementById('p');
  [
    'loadedmetadata','loadeddata','canplay','canplaythrough',
    'play','playing','pause','timeupdate','waiting','stalled','ended','error'
  ].forEach(ev => p.addEventListener(ev, () => console.log(ev, p.currentTime)));
</script>
```

---

## 7. JS 제어: 필수/확장 API

핵심:

```html
<audio id="player" src="sound.mp3" controls></audio>
<button onclick="player.play()">재생</button>
<button onclick="player.pause()">일시정지</button>
<button onclick="player.currentTime += 10">+10s</button>
<button onclick="player.currentTime = 0">처음으로</button>
<script>
  const player = document.getElementById('player');
  // 볼륨/음소거
  player.volume = 0.7;
  player.muted = false;
  // 속도
  player.playbackRate = 1.25;
  // 루프
  player.loop = true;
</script>
```

고급(오디오 스프라이트: 하나의 파일에서 구간별 재생):

```html
<audio id="fx" src="sfx.mp3" preload="metadata"></audio>
<button data-start="0" data-end="1.2" class="shot">샷</button>
<button data-start="1.2" data-end="2.1" class="ding">딩</button>
<script>
const fx = document.getElementById('fx');
let timer;
function playSegment(start, end) {
  clearInterval(timer);
  fx.currentTime = start;
  fx.play();
  timer = setInterval(() => {
    if (fx.currentTime >= end) { fx.pause(); clearInterval(timer); }
  }, 20);
}
document.querySelectorAll('button').forEach(b=>{
  b.addEventListener('click', () => playSegment(+b.dataset.start, +b.dataset.end));
});
</script>
```

---

## 8. 커스텀 컨트롤(UI) 만들기

네이티브 `controls`는 일관성이 높지만 스타일 제어가 제한. 직접 UI를 만들 수 있다.

```html
<div class="player">
  <audio id="a" src="song.mp3" preload="metadata"></audio>
  <button id="toggle">재생</button>
  <input id="seek" type="range" min="0" value="0" step="0.01">
  <input id="vol" type="range" min="0" max="1" step="0.01" value="0.8">
  <span id="t">0:00 / 0:00</span>
</div>

<script>
const a = document.getElementById('a');
const toggle = document.getElementById('toggle');
const seek = document.getElementById('seek');
const vol = document.getElementById('vol');
const t = document.getElementById('t');

function fmt(s){ const m = Math.floor(s/60)|0; const ss = Math.floor(s%60).toString().padStart(2,'0'); return `${m}:${ss}`; }

a.addEventListener('loadedmetadata', () => {
  seek.max = a.duration;
  t.textContent = `0:00 / ${fmt(a.duration)}`;
});
a.addEventListener('timeupdate', () => {
  seek.value = a.currentTime;
  t.textContent = `${fmt(a.currentTime)} / ${fmt(a.duration)}`;
});
seek.addEventListener('input', () => a.currentTime = seek.value);
vol.addEventListener('input', () => a.volume = vol.value);

toggle.addEventListener('click', async () => {
  if (a.paused) { try { await a.play(); toggle.textContent='일시정지'; } catch(e){ console.warn(e); } }
  else { a.pause(); toggle.textContent='재생'; }
});
</script>
```

---

## 9. 접근성(A11y)과 대체 콘텐츠

- **스크린 리더**: 오디오 컨트롤에 레이블을 명확히(버튼에 텍스트 제공).
- **대체 수단**: 자막/가사 파일(WebVTT)이나 **텍스트 대본** 링크 제공.
- 키보드 접근(탭/스페이스/엔터)으로 조작 가능하게 구성.

### WebVTT 캡션(가사) 예시

```html
<audio controls>
  <source src="song.mp3" type="audio/mpeg">
  <track src="lyrics_ko.vtt" kind="captions" srclang="ko" label="가사(한국어)" default>
  이 브라우저는 오디오를 지원하지 않습니다.
</audio>
```

`lyrics_ko.vtt`:

```
WEBVTT

00:00:00.000 --> 00:00:04.000
아침 햇살 속에 노래를 시작해요

00:00:04.000 --> 00:00:08.500
함께 부르면 마음이 가벼워져요
```

> 자막은 시각적 보조뿐 아니라 **청각장애 접근성**에 중요.

---

## 10. Media Session API (잠금 화면/시스템 UI 통합)

모바일/데스크톱에서 **미디어 메타데이터**와 **플랫폼 컨트롤** 통합.

```html
<audio id="m" src="podcast.mp3" preload="metadata"></audio>
<button id="play">재생</button>
<script>
const m = document.getElementById('m');
document.getElementById('play').onclick = () => m.play();

if ('mediaSession' in navigator) {
  navigator.mediaSession.metadata = new MediaMetadata({
    title: '에피소드 42: HTML 오디오',
    artist: 'DevCast',
    album: '웹개발',
    artwork: [
      { src: 'cover-96.png',  sizes:'96x96',  type:'image/png' },
      { src: 'cover-512.png', sizes:'512x512',type:'image/png' }
    ]
  });
  navigator.mediaSession.setActionHandler('play',  () => m.play());
  navigator.mediaSession.setActionHandler('pause', () => m.pause());
  navigator.mediaSession.setActionHandler('seekforward',  () => m.currentTime += 10);
  navigator.mediaSession.setActionHandler('seekbackward', () => m.currentTime -= 10);
}
</script>
```

---

## 11. Web Audio API 연동(비주얼라이저/이펙트)

`<audio>` → `AudioContext`로 라우팅하여 FFT 시각화/필터 적용 가능.

```html
<canvas id="viz" width="600" height="120"></canvas>
<audio id="s" src="song.mp3" controls crossorigin="anonymous"></audio>
<script>
const a = document.getElementById('s');
const ctx = new (window.AudioContext || window.webkitAudioContext)();
const src = ctx.createMediaElementSource(a);
const analyser = ctx.createAnalyser();
analyser.fftSize = 256;
src.connect(analyser).connect(ctx.destination);

const canvas = document.getElementById('viz');
const g = canvas.getContext('2d');
const data = new Uint8Array(analyser.frequencyBinCount);

function draw(){
  requestAnimationFrame(draw);
  analyser.getByteFrequencyData(data);
  g.clearRect(0,0,canvas.width,canvas.height);
  const w = canvas.width / data.length;
  data.forEach((v,i) => {
    const h = v / 255 * canvas.height;
    g.fillRect(i*w, canvas.height - h, w-1, h);
  });
}
draw();

// iOS/Safari: 사용자 제스처 후 resume 필요
document.addEventListener('click', () => { if (ctx.state !== 'running') ctx.resume(); }, { once: true });
</script>
```

> 교차 출처 리소스는 `crossorigin`과 서버 CORS 헤더가 필요(아래 참조).

---

## 12. CORS와 Canvas/Web Audio

- 다른 도메인에서 오디오를 불러오려면:
  - 서버: `Access-Control-Allow-Origin: *` (또는 특정 오리진)
  - 클라이언트: `<audio crossorigin="anonymous">`
- 그렇지 않으면 **보안 정책**으로 인해 Web Audio/Canvas로의 파이프라인에서 제약 발생(tainted).

```html
<audio controls crossorigin="anonymous">
  <source src="https://cdn.example.com/bgm.mp3" type="audio/mpeg">
</audio>
```

---

## 13. 스트리밍(라이브/롱폼) 개요: HLS/DASH

긴 오디오/라이브 라디오/팟캐스트에는 **적응형 스트리밍**이 유리:
- HLS (`.m3u8`), DASH (`.mpd`)
- **Safari는 HLS 네이티브 재생**, 그 외 브라우저는 hls.js 같은 **MSE(Media Source Extensions)** 플레이어 필요.

HLS 예시(hls.js):

```html
<audio id="live" controls></audio>
<script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
<script>
const audio = document.getElementById('live');
const src = 'https://example.com/live/stream.m3u8';

if (audio.canPlayType('application/vnd.apple.mpegurl')) {
  audio.src = src; // Safari
} else if (Hls.isSupported()) {
  const hls = new Hls();
  hls.loadSource(src);
  hls.attachMedia(audio);
} else {
  audio.outerHTML = '<p>브라우저가 HLS 오디오를 지원하지 않습니다.</p>';
}
</script>
```

---

## 14. 서버·배포 팁

- **Range 요청(부분 전송)** 활성화 → 탐색/스트리밍 반응성 개선.
- `Cache-Control`/`ETag`로 캐시 히트 유도.
- 대역폭 최적화를 위해 **CDN** 사용.
- 파일명/경로에 공백·특수문자 지양(인코딩 문제 방지).

---

## 15. 실전 예제 모음

### 15.1 팟캐스트 플레이어(자동재생 정책 친화)

```html
<section>
  <h2>팟캐스트 에피소드</h2>
  <audio id="pod" preload="metadata" controls crossorigin="anonymous">
    <source src="ep42.opus" type='audio/webm; codecs="opus"'>
    <source src="ep42.mp3"  type="audio/mpeg">
  </audio>
  <button id="startPod">재생</button>
  <button onclick="pod.currentTime -= 15">⟵ 15초</button>
  <button onclick="pod.currentTime += 30">30초 ⟶</button>
</section>
<script>
  document.getElementById('startPod').addEventListener('click', () => pod.play());
</script>
```

### 15.2 효과음(낮은 지연)

```html
<button id="hit">효과음</button>
<audio id="fx" src="hit.mp3" preload="auto"></audio>
<script>
  const fx = document.getElementById('fx');
  document.getElementById('hit').addEventListener('click', () => {
    // 연타 시도도 빠르게
    fx.currentTime = 0;
    fx.play();
  });
</script>
```

### 15.3 다수 플레이어 성능 패턴

```html
<ul id="list">
  <li><button data-src="a1.mp3">샘플 1</button></li>
  <li><button data-src="a2.mp3">샘플 2</button></li>
  <li><button data-src="a3.mp3">샘플 3</button></li>
</ul>
<audio id="one" preload="none"></audio>
<script>
const audio = document.getElementById('one');
document.getElementById('list').addEventListener('click', e => {
  const btn = e.target.closest('button[data-src]'); if (!btn) return;
  if (audio.src !== new URL(btn.dataset.src, location).href) {
    audio.src = btn.dataset.src; audio.load();
  }
  audio.play();
});
</script>
```

---

## 16. 체크리스트(프로덕션)

- [ ] **포맷**: MP3 필수 + Opus(Ogg/WebM) 보조(선택)
- [ ] **자동재생**: 사용자 제스처 이후 `.play()` 호출
- [ ] **preload**: `metadata` 또는 `none` (대량 페이지)
- [ ] **접근성**: 가사/대본(WebVTT 또는 텍스트 링크) 제공
- [ ] **CORS**: 교차 출처 시 `crossorigin` + 서버 CORS 헤더
- [ ] **성능**: Range/캐시/CDN, 짧은 효과음은 `preload="auto"`
- [ ] **UI**: 네이티브 컨트롤 또는 커스텀 컨트롤(키보드 조작 가능)
- [ ] **오류 처리**: `error` 이벤트 핸들링/폴백 제공
- [ ] **메타데이터**: Media Session으로 플랫폼 통합

---

## 17. 요약

- `<audio>`는 **호환성·성능·접근성**을 균형 있게 챙기면 가장 안정적인 오디오 전달 수단이다.
- 자동재생 정책을 존중하고, 포맷 폴백(MP3 + Opus)과 적절한 `preload` 전략을 사용하라.
- Media Session/Web Audio/CORS/스트리밍까지 이해하면, **팟캐스트·음악·효과음·교육 서비스**까지 확장 가능한 **완성도 높은 오디오 경험**을 제공할 수 있다.