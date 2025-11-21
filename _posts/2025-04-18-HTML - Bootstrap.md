---
layout: post
title: HTML - Bootstrap
date: 2025-04-18 23:20:23 +0900
category: HTML
---
# Bootstrap(부트스트랩)

## Bootstrap 한눈에 보기

- **정의**: 반응형 UI를 빠르게 구성하는 CSS·JS 프레임워크
- **철학**: 모바일 우선, 일관된 디자인 시스템, 컴포넌트 기반
- **v5 하이라이트**
  - jQuery 제거 → **순수 JS**
  - **Sass + CSS Custom Properties**로 테마 유연성↑
  - **Utilities API**로 헬퍼 클래스 확장
  - RTL 지원, 향상된 Grid(Flex/옵션 Grid), Offcanvas 등

CDN 빠른 시작:

```html
<link
  href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css"
  rel="stylesheet"
  integrity="sha384-QWTKZyjpPEjISv5WaRU9OFeRpok6YctnYmDr5pNlyT2bRjXh0JMhjY6hW+ALEwIH"
  crossorigin="anonymous"
/>
<script
  src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"
  integrity="sha384-ENjdO4Dr2bkBIFxQpeoA6LU6TQvYjU9G1rGqFJZ1EM5Q6V5J8mCZf2G6w2vC6k9k"
  crossorigin="anonymous"
></script>
```

NPM:

```bash
npm i bootstrap
```

```js
// main.js
import 'bootstrap/dist/css/bootstrap.min.css';
import 'bootstrap'; // 개별 사용 시: import 'bootstrap/js/dist/modal';
```

---

## — 레이아웃의 뼈대

### 컨테이너와 행/열

```html
<div class="container">            <!-- 고정 폭 (반응형 max-width) -->
  <div class="row">                <!-- 수평 레이아웃 단위 -->
    <div class="col-12 col-md-6">좌</div>
    <div class="col-12 col-md-6">우</div>
  </div>
</div>

<div class="container-fluid">      <!-- 100% 폭 -->
  ...
</div>
```

- `.container`, `.container-{breakpoint}`(예: `.container-lg`), `.container-fluid`
- `.row`는 **gutter(패딩)**를 통해 컬럼 간 간격 제공
- `.col-{breakpoint}-{n}`: 12 그리드 기반

### 브레이크포인트

| 접두 | 기준 |
|---|---|
| (없음) | <576px |
| sm | ≥576px |
| md | ≥768px |
| lg | ≥992px |
| xl | ≥1200px |
| xxl | ≥1400px |

### 제어

```html
<div class="row g-0"> ... </div>         <!-- 간격 0 -->
<div class="row gx-3 gy-5"> ... </div>   <!-- x축/ y축 분리 제어 -->
```

### Offset/Order/Auto Layout

```html
<div class="row">
  <div class="col-md-4 order-2 order-md-1">A</div>
  <div class="col-md-4 offset-md-4 order-1 order-md-2">B</div>
</div>
```

### CSS Grid(선택 사항)

Bootstrap v5는 Flex 기반이 기본이나 **CSS Grid 유틸리티**도 제공됩니다. 레이아웃 복잡도가 높다면 CSS Grid 유틸을 병행하세요.

---

## — “CSS 없이” 빠르게 스타일링

### 여백·정렬·디스플레이

```html
<div class="p-3 m-2 border rounded">패딩/마진/테두리</div>
<div class="d-flex justify-content-between align-items-center">Flex 정렬</div>
<div class="d-none d-md-block">md 이상에서만 보이기</div>
```

- `m{t|b|s|e|x|y}-{0..5}`, `p-{0..5}`
- `d-flex`, `d-grid`, `justify-content-*`, `align-items-*`
- `text-center`, `fw-bold`, `fst-italic`, `lh-lg`

### 색상/타이포/가시성

```html
<p class="text-primary bg-light">색상</p>
<p class="text-truncate">오버플로우 생략</p>
<p class="visually-hidden">스크린리더만</p>
```

### Responsive utilities

```html
<div class="d-none d-lg-block">큰 화면에서만</div>
```

---

## — 입력 UI와 검증

### 기본 폼과 그리드 조합

```html
<form class="row g-3">
  <div class="col-md-6">
    <label class="form-label">이메일</label>
    <input type="email" class="form-control" placeholder="name@example.com">
  </div>
  <div class="col-md-6">
    <label class="form-label">비밀번호</label>
    <input type="password" class="form-control">
  </div>
  <div class="col-12">
    <button class="btn btn-primary w-100">가입</button>
  </div>
</form>
```

### 체크/라디오/스위치/파일

```html
<div class="form-check">
  <input class="form-check-input" type="checkbox" id="agree">
  <label class="form-check-label" for="agree">약관 동의</label>
</div>

<div class="form-switch">
  <input class="form-check-input" type="checkbox" id="darkmode">
  <label class="form-check-label" for="darkmode">다크모드</label>
</div>

<input class="form-control" type="file" />
```

### Floating label

```html
<div class="form-floating">
  <input type="email" class="form-control" id="fEmail" placeholder="name@example.com">
  <label for="fEmail">이메일 주소</label>
</div>
```

### HTML5 검증 + 시각적 피드백

```html
<form class="needs-validation" novalidate>
  <div class="mb-3">
    <input type="text" class="form-control" required>
    <div class="invalid-feedback">필수 입력입니다.</div>
  </div>
  <button class="btn btn-primary">전송</button>
</form>

<script>
(() => {
  const forms = document.querySelectorAll('.needs-validation');
  forms.forEach(f => f.addEventListener('submit', e => {
    if (!f.checkValidity()) { e.preventDefault(); e.stopPropagation(); }
    f.classList.add('was-validated');
  }));
})();
</script>
```

---

## — 생산성의 핵심

### 버튼/버튼 그룹

```html
<button class="btn btn-primary">Primary</button>
<div class="btn-group">
  <button class="btn btn-outline-secondary">왼쪽</button>
  <button class="btn btn-outline-secondary">오른쪽</button>
</div>
```

### 카드(Card)

```html
<div class="card" style="max-width: 22rem;">
  <img src="https://picsum.photos/600/300" class="card-img-top" alt="">
  <div class="card-body">
    <h5 class="card-title">카드 제목</h5>
    <p class="card-text">간단 설명</p>
    <a href="#" class="btn btn-primary">자세히</a>
  </div>
</div>
```

### 네비게이션바(Navbar)

```html
<nav class="navbar navbar-expand-lg bg-body-tertiary">
  <div class="container">
    <a class="navbar-brand" href="#">Brand</a>
    <button class="navbar-toggler" data-bs-toggle="collapse" data-bs-target="#nav">
      <span class="navbar-toggler-icon"></span>
    </button>
    <div class="collapse navbar-collapse" id="nav">
      <ul class="navbar-nav ms-auto">
        <li class="nav-item"><a class="nav-link active">Home</a></li>
        <li class="nav-item"><a class="nav-link">Docs</a></li>
      </ul>
    </div>
  </div>
</nav>
```

### 드롭다운/드롭다운 내 메뉴

```html
<div class="dropdown">
  <button class="btn btn-secondary dropdown-toggle" data-bs-toggle="dropdown">메뉴</button>
  <ul class="dropdown-menu">
    <li><a class="dropdown-item" href="#">액션</a></li>
    <li><hr class="dropdown-divider"></li>
    <li><a class="dropdown-item disabled">비활성</a></li>
  </ul>
</div>
```

> 드롭다운·툴팁·팝오버는 Popper.js가 번들에 포함된 `bootstrap.bundle.js` 필요.

### 모달(Modal)

```html
<button class="btn btn-primary" data-bs-toggle="modal" data-bs-target="#exampleModal">열기</button>

<div class="modal fade" id="exampleModal" tabindex="-1">
  <div class="modal-dialog"><div class="modal-content">
    <div class="modal-header">
      <h5 class="modal-title">타이틀</h5>
      <button class="btn-close" data-bs-dismiss="modal"></button>
    </div>
    <div class="modal-body">본문 내용</div>
    <div class="modal-footer">
      <button class="btn btn-secondary" data-bs-dismiss="modal">닫기</button>
      <button class="btn btn-primary">저장</button>
    </div>
  </div></div>
</div>
```

JS 제어:

```js
import Modal from 'bootstrap/js/dist/modal';
const modal = new Modal('#exampleModal');
modal.show();
```

### 오프캔버스(Offcanvas)

```html
<button class="btn btn-outline-primary" data-bs-toggle="offcanvas" data-bs-target="#side">
  메뉴
</button>
<div class="offcanvas offcanvas-start" id="side">
  <div class="offcanvas-header">
    <h5>사이드</h5>
    <button class="btn-close" data-bs-dismiss="offcanvas"></button>
  </div>
  <div class="offcanvas-body">콘텐츠</div>
</div>
```

### 탭/아코디언/토스트/스피너

```html
<ul class="nav nav-tabs">
  <li class="nav-item"><a class="nav-link active" data-bs-toggle="tab" href="#t1">탭1</a></li>
  <li class="nav-item"><a class="nav-link" data-bs-toggle="tab" href="#t2">탭2</a></li>
</ul>
<div class="tab-content p-3 border border-top-0">
  <div class="tab-pane fade show active" id="t1">내용1</div>
  <div class="tab-pane fade" id="t2">내용2</div>
</div>
```

```html
<div class="accordion" id="acc">
  <div class="accordion-item">
    <h2 class="accordion-header">
      <button class="accordion-button" data-bs-toggle="collapse" data-bs-target="#c1">섹션 1</button>
    </h2>
    <div id="c1" class="accordion-collapse collapse show" data-bs-parent="#acc">
      <div class="accordion-body">본문</div>
    </div>
  </div>
</div>
```

```html
<div class="toast align-items-center text-bg-primary" id="t" role="alert">
  <div class="d-flex">
    <div class="toast-body">알림 메시지</div>
    <button class="btn-close btn-close-white me-2 m-auto" data-bs-dismiss="toast"></button>
  </div>
</div>

<script>
import Toast from 'bootstrap/js/dist/toast';
new Toast(document.getElementById('t')).show();
</script>
```

---

## 테이블/이미지/미디어

### 반응형 테이블

```html
<div class="table-responsive">
  <table class="table table-striped align-middle">
    <thead><tr><th>#</th><th>이름</th><th>상태</th></tr></thead>
    <tbody>
      <tr><td>1</td><td>Alpha</td><td><span class="badge bg-success">OK</span></td></tr>
      <tr><td>2</td><td>Beta</td><td><span class="badge bg-warning">Pending</span></td></tr>
    </tbody>
  </table>
</div>
```

### 이미지/Ratio(비율박스)

```html
<img src="..." class="img-fluid rounded" alt="">
<div class="ratio ratio-16x9"><iframe src="..." allowfullscreen></iframe></div>
```

---

## 아이콘과 브랜딩

- **Bootstrap Icons**(별도 패키지): `https://icons.getbootstrap.com/`
- SVG inline 또는 아이콘 폰트 사용 가능

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.css" />
<i class="bi bi-alarm"></i>
```

---

## — 시맨틱 + ARIA + 포커스

- 컴포넌트는 기본적으로 **ARIA 속성**과 키보드 탐색을 지원
- **스크린 리더 텍스트**: `.visually-hidden`
- **모션 감소**: `@media (prefers-reduced-motion: reduce)` 대응
- **컬러 대비**: `text-bg-*` / 변수 조정으로 명암비 확보
- **포커스 링**: `.focus-visible` 전략 고려

예: 시맨틱 버튼과 label 연결

```html
<label for="search" class="form-label">검색</label>
<input id="search" class="form-control" type="search" />
```

---

## 성능/번들/빌드 — 필요한 것만, 빠르게

### 개별 JS 임포트

```js
import Collapse from 'bootstrap/js/dist/collapse';
import Dropdown from 'bootstrap/js/dist/dropdown';
```

- 사용하지 않는 컴포넌트는 제외 → 번들 크기↓
- Vite/webpack/Rollup으로 트리셰이킹

### CSS 최적화

- **Sass 빌드**(아래 커스터마이징과 함께) 후 **Purge/Content-aware 제거**(예: `@fullhuman/postcss-purgecss`, Tailwind JIT 유사 도구 등)
- 정적 사이트는 빌드시 사용 클래스만 남기는 전략으로 크기↓
  단, **동적 클래스명**은 safelist에 추가 필요

### HTTP/캐싱

- CDN 사용 + SRI + 긴 `max-age`/`immutable`
- 앱 CSS/JS는 해시 기반 파일명으로 캐싱 전략 수립

---

## Sass 커스터마이징 — 디자인 시스템 내 것으로

> Bootstrap 5는 Dart Sass 기준의 `@use` 문법 권장.

### 변수 오버라이드 → Bootstrap 로드

```scss
// styles.scss
// 1) 먼저 변수 설정
$primary: #5b8def;
$body-font-family: 'Inter', system-ui, -apple-system, Segoe UI, Roboto, sans-serif;
$border-radius: .75rem;
$enable-rounded: true;

// 2) Bootstrap 로드
@use "bootstrap/scss/bootstrap";
```

### 부분 임포트(선택 로딩)

```scss
// 필요한 파트만
@use "bootstrap/scss/functions";
@use "bootstrap/scss/variables" with (
  $primary: #5b8def,
  $danger: #ff5c5c
);
@use "bootstrap/scss/maps";
@use "bootstrap/scss/mixins";
@use "bootstrap/scss/root";
@use "bootstrap/scss/reboot";
@use "bootstrap/scss/type";
@use "bootstrap/scss/buttons";
// ... 필요한 컴포넌트만 추가
```

### 유틸리티 API로 커스텀 유틸 만들기

```scss
@use "bootstrap/scss/bootstrap" with (
  $utilities: (
    "rotate": (
      property: transform,
      class: rotate,
      values: (
        0: rotate(0),
        90: rotate(90deg),
        180: rotate(180deg)
      )
    )
  )
);
```

```html
<div class="rotate-90">회전된 박스</div>
```

### CSS Custom Properties 활용

- 일부 색상/스페이싱은 런타임에도 CSS 변수로 조정 가능
- 다크모드 전환 시 루트 변수만 교체하는 테크닉 활용

---

## 다크 모드/테마 토글(실전)

```html
<button id="themeToggle" class="btn btn-sm btn-outline-secondary">Theme</button>

<script>
const html = document.documentElement;
document.getElementById('themeToggle').addEventListener('click', () => {
  html.dataset.bsTheme = html.dataset.bsTheme === 'dark' ? 'light' : 'dark';
});
</script>
```

```css
:root { --bs-primary: #5b8def; }
:root[data-bs-theme="dark"] {
  --bs-body-bg: #121212;
  --bs-body-color: #e6e6e6;
  --bs-primary: #8ab4f8;
}
```

---

## 실전 레이아웃 패턴

### 대시보드(고정 사이드바 + 컨텐츠)

```html
<div class="d-flex">
  <aside class="d-none d-md-block bg-light border-end" style="width: 260px;">
    <div class="p-3">
      <h5 class="mb-3">메뉴</h5>
      <ul class="nav nav-pills flex-column gap-1">
        <li class="nav-item"><a class="nav-link active">Overview</a></li>
        <li class="nav-item"><a class="nav-link">Reports</a></li>
      </ul>
    </div>
  </aside>
  <main class="flex-fill">
    <nav class="navbar bg-body-tertiary border-bottom">
      <div class="container-fluid">
        <span class="navbar-brand">Admin</span>
        <button class="btn btn-sm btn-outline-secondary">Export</button>
      </div>
    </nav>
    <div class="container-fluid py-4">
      <div class="row g-3">
        <div class="col-md-4">
          <div class="card h-100">
            <div class="card-body">
              <h6 class="text-muted">Users</h6>
              <div class="fs-2 fw-semibold">12,930</div>
            </div>
          </div>
        </div>
        <!-- ... -->
      </div>
    </div>
  </main>
</div>
```

### 랜딩 페이지 히어로

```html
<header class="bg-light py-5">
  <div class="container text-center">
    <h1 class="display-5 fw-bold">빠르게, 일관되게, 아름답게</h1>
    <p class="lead text-muted">Bootstrap으로 반응형 제품을 더 빨리 출시하세요.</p>
    <a class="btn btn-primary btn-lg" href="#get-started">시작하기</a>
    <a class="btn btn-outline-secondary btn-lg ms-2" href="#docs">문서 보기</a>
  </div>
</header>
```

---

## 프레임워크와 결합

- **React**: React-Bootstrap, Reactstrap 또는 기본 Bootstrap + data-bs-* / 직접 JS
- **Vue**: BootstrapVueNext(Community), 기본 적용
- **Svelte/Angular**: 래퍼 컴포넌트 또는 기본 CSS + data 속성
- SPA에선 **라우트 전환시 포커스 관리**, 컴포넌트 언마운트 시 이벤트 정리 필요

---

## 요약

- jQuery 의존성 제거 → `data-bs-*` 속성명으로 변경
- `.form-group` → **공백 유틸리티**로 대체, 폼 구조 단순화
- Card/Badge 일부 클래스 변경(색 명명 체계 업데이트)
- 아이콘은 별도 패키지(“Glyphicons” 미포함)
- 유틸리티 이름/스케일 일부 변경(문서의 Migration 가이드 필수 확인)

---

## 테스트/디버깅/접근성 검증

- DevTools **Rendering** 패널로 모바일/색 약/모션 감소 체크
- Lighthouse/axe-core로 접근성 자동 점검
- **Tab** 키로 인터랙션 가능한지 수동 테스트
- 적절한 **aria-label/title**/`<label for>`/대체 텍스트 제공 필수

---

## 자주 겪는 문제와 해결

| 문제 | 원인/대응 |
|---|---|
| 드롭다운/툴팁 동작 안 함 | `bootstrap.bundle.js`(Popper 포함)로 로드했는지 확인 |
| 모달이 뒤에 깔림 | z-index 스택 컨텍스트 문제. 부모에 `position`/`transform` 확인 |
| 모바일에서 Navbar 토글 안 됨 | `data-bs-toggle="collapse"`/`data-bs-target` ID 일치 확인 |
| 폰트/아이콘 안 보임 | CDN 경로/SRI/crossorigin 확인 |
| 유틸 겹침 | 사용자 CSS가 Bootstrap보다 뒤에서 강한 선택자로 덮어썼는지 확인 |

---

## 보안/정책/엔터프라이즈 운영

- CDN 사용 시 **SRI + crossorigin** 지정
- 사내망은 사설 NPM/사내 CDN 구성
- 컴포넌트 동작은 전부 클라이언트 → **입력 검증**/XSS/CSRF 서버단 필수
- 콘텐츠 보안 정책(CSP) 설정 시 inline script 회피 또는 nonce/hash 사용

---

## 체크리스트: 실전 프로젝트 시작 전

- 디자인 토큰(색/타이포/간격/둥글기) → Sass 변수 정의
- **Utilities API**로 팀 표준 헬퍼 세트 정의
- 컴포넌트 사용 범위 합의(“카드만 쓰자/모달 금지” 등)
- 접근성 기준(WCAG 레벨)과 검수 루틴 명시
- 번들 전략(개별 import/트리셰이킹)과 Purge 구성
- 브라우저 지원(IE 미지원) 범위 합의

---

## 최소 템플릿(실행 가능한 스타터)

```html
<!doctype html>
<html lang="ko" data-bs-theme="light">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Bootstrap Starter</title>
  <link
    href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css"
    rel="stylesheet"
    integrity="sha384-QWTKZyjpPEjISv5WaRU9OFeRpok6YctnYmDr5pNlyT2bRjXh0JMhjY6hW+ALEwIH"
    crossorigin="anonymous"
  >
</head>
<body>
  <nav class="navbar navbar-expand-lg bg-body-tertiary">
    <div class="container">
      <a class="navbar-brand" href="#">Brand</a>
      <button class="navbar-toggler" data-bs-toggle="collapse" data-bs-target="#nav">
        <span class="navbar-toggler-icon"></span>
      </button>
      <div id="nav" class="collapse navbar-collapse">
        <ul class="navbar-nav ms-auto">
          <li class="nav-item"><a class="nav-link active">Home</a></li>
          <li class="nav-item"><a class="nav-link">Docs</a></li>
        </ul>
      </div>
    </div>
  </nav>

  <header class="py-5 text-center">
    <div class="container">
      <h1 class="display-5 fw-bold">Bootstrap v5.x</h1>
      <p class="lead text-muted">빠른 반응형 UI, 커스터마이징 가능한 디자인 시스템</p>
      <button class="btn btn-primary btn-lg" data-bs-toggle="modal" data-bs-target="#info">자세히</button>
    </div>
  </header>

  <main class="container py-4">
    <div class="row g-3">
      <div class="col-md-4">
        <div class="card h-100"><div class="card-body">
          <h5 class="card-title">그리드</h5>
          <p class="card-text">12단 그리드로 반응형 레이아웃을 설계합니다.</p>
        </div></div>
      </div>
      <div class="col-md-4">
        <div class="card h-100"><div class="card-body">
          <h5 class="card-title">유틸리티</h5>
          <p class="card-text">여백/정렬/색상을 클래스만으로 조합.</p>
        </div></div>
      </div>
      <div class="col-md-4">
        <div class="card h-100"><div class="card-body">
          <h5 class="card-title">컴포넌트</h5>
          <p class="card-text">모달/드롭다운/오프캔버스 등 공통 패턴 제공.</p>
        </div></div>
      </div>
    </div>
  </main>

  <div class="modal fade" id="info" tabindex="-1">
    <div class="modal-dialog"><div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title">정보</h5>
        <button class="btn-close" data-bs-dismiss="modal"></button>
      </div>
      <div class="modal-body">여기에 프로젝트 안내를 넣으세요.</div>
      <div class="modal-footer">
        <button class="btn btn-secondary" data-bs-dismiss="modal">닫기</button>
        <button class="btn btn-primary">확인</button>
      </div>
    </div></div>
  </div>

  <script
    src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"
    integrity="sha384-ENjdO4Dr2bkBIFxQpeoA6LU6TQvYjU9G1rGqFJZ1EM5Q6V5J8mCZf2G6w2vC6k9k"
    crossorigin="anonymous"
  ></script>
</body>
</html>
```

---

## 결론

- **빠른 프로토타이핑**과 **안정적인 디자인 시스템** 구축에 탁월
- v5의 **jQuery 제거/유틸리티 API/Sass 변수/다크모드/RTL**로 **현대적 요구** 충족
- 실무에서는 **접근성/성능/빌드/커스터마이징 전략**을 함께 설계해야 진가 발휘
