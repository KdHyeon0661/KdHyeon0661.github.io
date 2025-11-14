---
layout: post
title: HTML - canvas
date: 2025-04-09 19:20:23 +0900
category: HTML
---
# HTML5 `<canvas>` 태그

## `<canvas>`란 무엇인가

- **픽셀 기반 비트맵 렌더 타깃**. 벡터 DOM이 아닌, **JS가 매 프레임 그려 넣는 화면 버퍼**.
- 자체로는 “빈 사각형”; 실제 그리기는 **CanvasRenderingContext2D** 혹은 **WebGL/WebGPU** 컨텍스트가 수행.
- 상태 기반(Stateful) API: 펜 색, 선 굵기, 변환 행렬 등이 컨텍스트에 저장되어 그 이후 호출에 영향을 준다.

### 최소 문법

```html
<canvas id="c" width="400" height="300">이 브라우저는 Canvas를 지원하지 않습니다.</canvas>
<script>
  const ctx = document.getElementById('c').getContext('2d');
  ctx.fillStyle = '#2b6cb0';
  ctx.fillRect(100, 80, 200, 140);
</script>
```

- `width`/`height`는 **그림 해상도**(픽셀 버퍼 크기).
- CSS `width`/`height`는 화면 배치 크기(레이아웃). 둘의 차이를 이해해야 HiDPI 대응이 가능하다.

---

## 좌표계·경로·상태 — 2D 컨텍스트 핵심 모델

### 좌표계

- 기본 좌표 원점 `(0,0)`은 좌상단, x는 오른쪽, y는 아래로 증가.
- 모든 그리기는 현재 **변환 행렬(CTM)**에 의해 변환된다. `scale/rotate/translate/transform`로 변경.

### 경로(Path)와 상태(State)

- **경로 기반**: `beginPath → moveTo/lineTo/arc/rect/bezierCurveTo/... → fill()/stroke()` 흐름.
- **상태 스택**: `save()`로 푸시, `restore()`로 팝. 스타일/변환/클리핑 등 묶음 관리.

```js
ctx.save();
ctx.translate(cx, cy);
ctx.rotate(Math.PI/6);
ctx.fillStyle = '#4caf50';
ctx.fillRect(-50, -30, 100, 60); // 회전·이동 상태 적용
ctx.restore();
```

---

## 기본 도형 그리기

```js
const ctx = c.getContext('2d');

// 채운 사각형
ctx.fillStyle = '#f6ad55';
ctx.fillRect(20, 20, 120, 80);

// 선 그리기
ctx.beginPath();
ctx.moveTo(20, 120);
ctx.lineTo(200, 160);
ctx.strokeStyle = '#c53030';
ctx.lineWidth = 4;
ctx.stroke();

// 원·호
ctx.beginPath();
ctx.arc(200, 80, 40, 0, Math.PI * 2);
ctx.fillStyle = '#2f855a';
ctx.fill();
```

### 선 스타일

```js
ctx.lineWidth = 8;
ctx.lineCap = 'round';        // butt | round | square
ctx.lineJoin = 'miter';       // bevel | round | miter
ctx.miterLimit = 8;
ctx.setLineDash([12, 6]);     // 대시
ctx.lineDashOffset = 0;
```

### 곡선 API

- `quadraticCurveTo(cpx,cpy,x,y)` — 2차 베지에
- `bezierCurveTo(cp1x,cp1y,cp2x,cp2y,x,y)` — 3차 베지에
- `arc(x,y,r,start,end,ccw?)`, `ellipse(x,y,rx,ry,rot,start,end,ccw?)`
- `roundRect(x,y,w,h,radius)` 최신 사양에서 제공

---

## 채우기·윤곽·그라디언트·패턴·그림자

### 그라디언트

```js
const g = ctx.createLinearGradient(0,0,0,200);
g.addColorStop(0, '#3182ce');
g.addColorStop(1, '#63b3ed');
ctx.fillStyle = g;
ctx.fillRect(240, 20, 120, 160);
```

`createRadialGradient(x0,y0,r0,x1,y1,r1)`로 원형 그라디언트도 가능.

### 패턴

```js
const img = new Image();
img.src = 'tile.png';
img.onload = () => {
  const p = ctx.createPattern(img, 'repeat'); // repeat-x | repeat-y | no-repeat
  ctx.fillStyle = p;
  ctx.fillRect(20, 220, 160, 80);
};
```

### 그림자

```js
ctx.shadowColor = 'rgba(0,0,0,0.35)';
ctx.shadowBlur = 10;
ctx.shadowOffsetX = 0;
ctx.shadowOffsetY = 6;
ctx.fillStyle = '#fff';
ctx.fillRect(200, 220, 160, 80);
```

### 합성(Compositing)

```js
ctx.globalAlpha = 0.8;
ctx.globalCompositeOperation = 'source-over';
// destination-over, multiply, screen, overlay, darken, lighten 등 지원
```

---

## 텍스트 렌더링과 측정

```js
ctx.font = '600 24px system-ui, sans-serif'; // weight size family
ctx.textAlign = 'left';       // start | end | left | right | center
ctx.textBaseline = 'alphabetic'; // top | hanging | middle | ideographic | bottom
ctx.fillStyle = '#1a202c';
ctx.fillText('Hello Canvas', 30, 340);
```

텍스트 폭 측정:

```js
const m = ctx.measureText('Hello Canvas');
console.log(m.width);
```

윤곽선 텍스트:

```js
ctx.strokeStyle = '#2b6cb0';
ctx.lineWidth = 2;
ctx.strokeText('Outlined', 30, 370);
```

---

## 이미지·비디오 그리기와 고급 옵션

### 이미지 그리기

```js
const img = new Image();
img.crossOrigin = 'anonymous'; // CORS 설정(taint 방지)
img.src = 'https://example-cdn.com/pic.jpg';
img.onload = () => {
  ctx.drawImage(img, 20, 400, 160, 120);
  // 소스 영역 일부를 잘라서 그리기
  // ctx.drawImage(img, sx,sy,sw,sh, dx,dy,dw,dh);
};
```

### 비디오 프레임 그리기

```js
const v = document.createElement('video');
v.src = 'movie.mp4';
v.muted = true; v.autoplay = true; v.loop = true;
v.addEventListener('play', drawFrame);
function drawFrame() {
  if (!v.paused && !v.ended) {
    ctx.drawImage(v, 200, 400, 200, 112);
    requestAnimationFrame(drawFrame);
  }
}
```

### 이미지 스무딩·필터

```js
ctx.imageSmoothingEnabled = true;
ctx.imageSmoothingQuality = 'high'; // low | medium | high

// CSS와 유사한 필터(브라우저 지원 확인)
ctx.filter = 'blur(2px) brightness(1.1) contrast(1.05)';
ctx.drawImage(img, 420, 20, 160, 120);
ctx.filter = 'none';
```

---

## 클리핑과 Path2D

### 클리핑

```js
ctx.save();
ctx.beginPath();
ctx.arc(500, 220, 60, 0, Math.PI*2);
ctx.clip(); // 이후 그리기는 원 내부로 제한
ctx.drawImage(img, 440, 160, 120, 120);
ctx.restore();
```

### Path2D 재사용

```js
const p = new Path2D();
p.moveTo(0,0); p.lineTo(80,0); p.lineTo(40,70); p.closePath();
ctx.save();
ctx.translate(420, 220);
ctx.fillStyle = '#ed8936';
for (let i=0;i<6;i++){ ctx.rotate(Math.PI/3); ctx.fill(p); }
ctx.restore();
```

---

## 픽셀 조작과 이펙트

### 픽셀 읽기/쓰기

```js
const { width, height } = c;
const imgData = ctx.getImageData(0, 0, width, height); // 비용 큼
const data = imgData.data; // [r,g,b,a,...]

for (let i = 0; i < data.length; i += 4) {
  // 간단한 그레이스케일
  const y = 0.2126*data[i] + 0.7152*data[i+1] + 0.0722*data[i+2];
  data[i] = data[i+1] = data[i+2] = y;
}
ctx.putImageData(imgData, 0, 0);
```

성능 팁:
- 빈번한 `getImageData`는 피하고, **작은 영역만** 처리.
- `ctx.getContext('2d', { willReadFrequently: true })` 힌트를 줄 수 있다.
- 무거운 필터링은 **Web Worker + OffscreenCanvas**로 분리.

---

## 애니메이션 루프와 물리 기초

### 기본 루프

```js
let t0 = performance.now();
function loop(t1) {
  const dt = (t1 - t0) / 1000; t0 = t1;
  update(dt); draw();
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);

function update(dt) {
  // 위치, 속도, 가속 업데이트 등
}
function draw() {
  ctx.clearRect(0,0,c.width,c.height);
  // 상태 기반 그리기
}
```

### 간단한 탄성 공 예제

```js
const W = c.width, H = c.height;
let x=80, y=80, vx=160, vy=60, r=20, g=400; // 중력
function update(dt){
  vy += g*dt;
  x += vx*dt; y += vy*dt;
  if (x<r){ x=r; vx*=-0.9; } if (x>W-r){ x=W-r; vx*=-0.9; }
  if (y>H-r){ y=H-r; vy*=-0.8; } if (y<r){ y=r; vy*=-0.8; }
}
function draw(){
  ctx.clearRect(0,0,W,H);
  ctx.fillStyle='#3182ce';
  ctx.beginPath(); ctx.arc(x,y,r,0,Math.PI*2); ctx.fill();
}
```

---

## 반응형·HiDPI(레티나) 스케일링

### 기본 원리

- CSS 크기와 실제 픽셀 버퍼 크기를 **디바이스 픽셀 비율(dpr)**에 맞춰 동기화.
- `devicePixelRatio`를 곱해 캔버스 버퍼를 크게 잡고 컨텍스트를 스케일링.

```js
function resizeCanvas(canvas){
  const dpr = window.devicePixelRatio || 1;
  const rect = canvas.getBoundingClientRect();
  canvas.width  = Math.round(rect.width  * dpr);
  canvas.height = Math.round(rect.height * dpr);
  const ctx = canvas.getContext('2d');
  ctx.setTransform(dpr,0,0,dpr,0,0); // 이후 좌표는 CSS 픽셀 기준
  return ctx;
}
const ctx2 = resizeCanvas(c);

// 창/컨테이너 리사이즈에 대응
new ResizeObserver(()=>resizeCanvas(c)).observe(c);
```

---

## 성능 최적화 체크리스트

- **리드백 최소화**: `getImageData`/`measureText` 등 CPU↔GPU 왕복 줄이기.
- **드로우 콜 묶기**: 동일 스타일은 연속 호출, `beginPath`/`fill` 최소화.
- **오브젝트 풀**: GC 압력 줄이기(새 배열·Path2D 남발 지양).
- **레벨-오브-디테일(LOD)**: 원거리·작은 요소는 단순화.
- **OffscreenCanvas**: 백그라운드 렌더링, 멀티스레드.
- **캔버스 레이어링**: 불변 배경/빈번 갱신 요소를 다른 캔버스로 분리.
- **이미지 프리로드**: onload 이후 그리기, GPU 업로드 캐시 활용.
- **요청 타이밍**: `requestAnimationFrame` 사용, setInterval 지양.

---

## 보안·CORS·Tainted Canvas

- 교차 출처 이미지/비디오를 인증 없이 그리면 캔버스가 **tainted** 상태가 되어 `toDataURL`/`getImageData`가 차단.
- 해결:
  1) 이미지 측 **CORS 헤더**: `Access-Control-Allow-Origin: https://your-origin`
  2) JS 측 이미지에 `crossOrigin='anonymous'` 설정.
- 민감 데이터 렌더링 시 스크린샷 유출에 유의. 렌더 결과의 **다운로드 링크** 제공은 정책적으로 검토.

```js
const img = new Image();
img.crossOrigin = 'anonymous'; // 서버가 ACAO 허용해야 유효
img.src = 'https://cdn.example.com/a.png';
```

---

## 내보내기와 파일 다운로드

```js
// DataURL
const url = c.toDataURL('image/png'); // 'image/jpeg', 품질 인자 가능
// Blob이 더 메모리 효율적
c.toBlob(blob => {
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = 'canvas.png';
  a.click();
  URL.revokeObjectURL(a.href);
}, 'image/png');
```

---

## 접근성 전략

- 캔버스 내용은 스크린리더에 **직접 노출되지 않는다**.
- 접근성 보조층:
  - **대체 텍스트**를 `<canvas>` 내용으로 제공.
  - 인터랙티브 컨트롤은 **관찰 가능한 DOM** 위젯을 **오버레이** 레이어로 제공하고 캔버스는 시각화만 담당.
  - 키보드 포커스는 오버레이 요소에 두고, 캔버스는 `role="img"`와 `aria-label`로 요약.

```html
<canvas id="chart" role="img" aria-label="2025년 월별 매출 꺾은선 그래프">
  2025년 월별 매출 그래프 요약: 1월 12억, 2월 14억, ...
</canvas>
```

---

## 실전 미니 프로젝트

### 반응형 막대 차트(데이터 변화 애니메이션 포함)

```html
<style>
#wrap { width: 100%; max-width: 720px; margin: 24px auto; }
#wrap canvas { width: 100%; height: 360px; display: block; border: 1px solid #ddd; }

</style>
<div id="wrap">
  <canvas id="bar"></canvas>
</div>
<script>
const canvas = document.getElementById('bar');
const ctx = canvas.getContext('2d', { alpha: false });
let DPR=1, W=0, H=0;

const series0 = [12, 18, 9, 22, 15, 30];
let anim = { t: 0, dur: 600, from: series0.map(()=>0), to: series0.slice() };

function easeOutCubic(x){ return 1 - Math.pow(1-x, 3); }

function resize(){
  const r = canvas.getBoundingClientRect();
  DPR = window.devicePixelRatio || 1;
  canvas.width = Math.round(r.width*DPR);
  canvas.height= Math.round(r.height*DPR);
  ctx.setTransform(DPR,0,0,DPR,0,0);
  W = r.width; H = r.height;
}
new ResizeObserver(resize).observe(canvas);

function drawBars(vals){
  ctx.clearRect(0,0,W,H);
  const pad = 32, bw = (W - pad*2) / vals.length * 0.6;
  const gap = (W - pad*2) / vals.length * 0.4;
  const maxV = Math.max(...vals, 1);
  // 축
  ctx.strokeStyle='#999'; ctx.lineWidth=1;
  ctx.beginPath(); ctx.moveTo(pad, H-pad); ctx.lineTo(W-pad, H-pad); ctx.stroke();

  for(let i=0;i<vals.length;i++){
    const x = pad + i*(bw+gap) + gap/2;
    const h = (H - pad*2) * (vals[i]/maxV);
    ctx.fillStyle = '#3182ce';
    ctx.fillRect(x, H-pad-h, bw, h);
    ctx.fillStyle = '#222';
    ctx.font = '12px system-ui';
    ctx.fillText(String(vals[i]), x, H-pad-h-6);
  }
}

let last=0;
function tick(t){
  if(!last) last=t;
  const dt = t - last; last = t;
  anim.t = Math.min(anim.t + dt, anim.dur);
  const k = easeOutCubic(anim.t/anim.dur);
  const vals = anim.to.map((v,i)=>anim.from[i] + (v-anim.from[i])*k);
  drawBars(vals);
  if (anim.t < anim.dur) requestAnimationFrame(tick);
}

function randomize(){
  anim.from = anim.to.slice();
  anim.to = anim.to.map(()=> Math.round(5 + Math.random()*30));
  anim.t = 0;
  requestAnimationFrame(tick);
}

resize();
requestAnimationFrame(tick);
setInterval(randomize, 2000);
</script>
```

포인트:
- **HiDPI 대응**: `devicePixelRatio` 스케일.
- **애니메이션**: 시간 보간과 이징.
- **단일 캔버스**지만, 축/막대/라벨을 **드로우콜 최소화**로 구성.

---

## OffscreenCanvas·Web Worker로 병렬 처리

메인 스레드 UI 가로막힘 없이 렌더:

```js
// main.js
const worker = new Worker('worker.js', { type: 'module' });
const off = new OffscreenCanvas(800, 600);
const offCtx = off.getContext('2d');
worker.postMessage({ canvas: off }, [off]); // 소유권 이전

// worker.js
self.onmessage = e => {
  const ctx = e.data.canvas.getContext('2d');
  function loop(){
    ctx.clearRect(0,0,800,600);
    // 무거운 드로잉
    ctx.fillRect(Math.random()*800, Math.random()*600, 10, 10);
    requestAnimationFrame(loop);
  }
  loop();
};
```

주의:
- 브라우저 지원 확인 필요.
- 동일 출처 정책과 파일 로딩 경로 주의.

---

## WebGL과의 연계

- `<canvas>`는 **2D** 뿐 아니라 **WebGL/WebGL2** 컨텍스트를 제공한다.
- 3D 장면, 대규모 파티클, GPGPU 등 고성능 요구 작업 가능.
- 2D에서도 WebGL 기반 엔진(Pixi.js 등)을 쓰면 대규모 스프라이트에 유리.

---

## 흔한 문제와 해결

| 문제 | 원인 | 대처 |
|---|---|---|
| 글자가 흐릿함 | HiDPI 미대응 | DPR 스케일링, 정수 좌표 스냅 |
| 이미지 그릴 때 CORS 에러 | 교차 출처 + CORS 미설정 | 서버 ACAO + `img.crossOrigin='anonymous'` |
| 애니메이션이 끊김 | 과도한 드로우콜/픽셀 리드백 | 레이어 분리, 오브젝트 풀, getImageData 최소화 |
| 클릭 판정 어려움 | Canvas는 자체 DOM 없음 | 투명 오버레이 DOM, 또는 **픽셀 피킹** 구현 |
| 리사이즈 시 일그러짐 | CSS/버퍼 불일치 | DPR 기반 동기화 함수 사용 |

---

## 라이브러리·엔진

- 2D 엔진: **Pixi.js**, **Konva.js**, **Fabric.js**, **Paper.js**
- 차트: **Chart.js**, **uPlot**, **ECharts**
- 게임: **Phaser**, **Babylon.js(WebGL)**, **Three.js(WebGL)**
- 물리: **matter-js**, **planck.js**
- 필터/이미지: **glfx.js**, WebGL 기반 자체 구현 권장

---

## 체크리스트 요약

- [ ] HiDPI 스케일링과 리사이즈 처리
- [ ] 상태 스택(`save/restore`)으로 지역 스타일 관리
- [ ] 드로우콜/경로 묶기, 리드백 최소화
- [ ] 필요 시 OffscreenCanvas/Workers 분리
- [ ] CORS·taint 규칙 이해 후 이미지 그리기
- [ ] 접근성 대체 텍스트/DOM 오버레이 제공
- [ ] 내보내기 시 `toBlob` 사용, 메모리 관리

---

## 부록 A. API 치트시트

```js
// 상태
ctx.save(); ctx.restore();
ctx.globalAlpha = 1;
ctx.globalCompositeOperation = 'source-over';

// 선/채우기
ctx.strokeStyle = '#000'; ctx.fillStyle = '#000';
ctx.lineWidth = 1; ctx.lineCap='butt'; ctx.lineJoin='miter';
ctx.setLineDash([]);

// 변환
ctx.translate(x, y); ctx.rotate(rad); ctx.scale(sx, sy);
ctx.transform(a,b,c,d,e,f); ctx.setTransform(1,0,0,1,0,0); // 리셋

// 경로
ctx.beginPath(); ctx.moveTo(x,y); ctx.lineTo(x,y);
ctx.rect(x,y,w,h); ctx.arc(x,y,r,sa,ea,true?);
ctx.ellipse(x,y,rx,ry,rot,sa,ea);
ctx.quadraticCurveTo(cpx,cpy,x,y); ctx.bezierCurveTo(...);
ctx.closePath(); ctx.stroke(); ctx.fill('nonzero'); // 'evenodd'

// 텍스트
ctx.font = '16px sans-serif';
ctx.textAlign = 'start'; ctx.textBaseline = 'alphabetic';
ctx.fillText('text', x, y); ctx.strokeText('text', x, y);
ctx.measureText('text');

// 이미지
ctx.drawImage(img, dx,dy);
ctx.drawImage(img, sx,sy,sw,sh, dx,dy,dw,dh);

// 픽셀
ctx.getImageData(sx,sy,sw,sh); ctx.putImageData(imgData, dx,dy);

// 스타일 확장
ctx.createLinearGradient(...); ctx.createRadialGradient(...);
ctx.createPattern(img, 'repeat');

// 필터/스무딩
ctx.filter = 'none';
ctx.imageSmoothingEnabled = true;
ctx.imageSmoothingQuality = 'low';
```

---

## 결론

`<canvas>`는 **픽셀 버퍼에 직접 그리는 저수준 그래픽스 API**로, 게임·시뮬레이션·데이터 시각화·이미지 처리 등 광범위한 분야를 포괄한다. 핵심은 **상태 기반 모델과 좌표·변환·경로의 이해**, 그리고 **성능·반응형·보안·접근성**에 대한 일관된 전략이다. 본문 예제와 체크리스트를 기반으로, 필요 시 WebGL/OffscreenCanvas/Workers를 결합하여 **부드럽고 선명하며 안전한** 캔버스 애플리케이션을 구축하라.
