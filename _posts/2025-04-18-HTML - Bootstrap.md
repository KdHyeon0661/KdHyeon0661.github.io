---
layout: post
title: HTML - Bootstrap
date: 2025-04-18 23:20:23 +0900
category: HTML
---
# 🎨 Bootstrap(부트스트랩) 완전 가이드

## ✅ Bootstrap이란?

**Bootstrap**은 웹사이트나 웹 애플리케이션을 빠르게 만들 수 있도록 도와주는 **HTML, CSS, JavaScript 프레임워크**입니다.

> ✅ 반응형 웹 디자인  
> ✅ 다양한 UI 컴포넌트  
> ✅ 일관된 스타일  
> ✅ 빠른 프로토타이핑 가능

---

## 🕰️ 역사

- **2011년**: Twitter 개발자 Mark Otto와 Jacob Thornton이 오픈소스로 공개
- **v3**: 모바일 우선, 반응형 강화
- **v4**: Flexbox 도입, Sass로 전환
- **v5**: jQuery 제거, 유틸리티 API 도입, CSS Grid 지원 강화

> 현재 최신 버전은 **Bootstrap 5.x** (jQuery 미사용, 순수 JavaScript 기반)

---

## 🚀 설치 방법

### 1. CDN 사용 (가장 간편)

```html
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
```

### 2. npm 설치 (모듈 방식)

```bash
npm install bootstrap
```

```js
// main.js
import 'bootstrap/dist/css/bootstrap.min.css';
import 'bootstrap';
```

### 3. 직접 다운로드

- [https://getbootstrap.com](https://getbootstrap.com) 에서 직접 zip 파일 다운로드 가능

---

## 📐 그리드 시스템 (Grid)

Bootstrap의 핵심 기능 중 하나는 **반응형 그리드 시스템**입니다.

```html
<div class="container">
  <div class="row">
    <div class="col-md-6">왼쪽</div>
    <div class="col-md-6">오른쪽</div>
  </div>
</div>
```

### 주요 클래스

| 클래스 | 설명 |
|--------|------|
| `.container` | 고정 폭 컨테이너 |
| `.container-fluid` | 100% 너비 |
| `.row` | 행(Row) |
| `.col-*` | 열(Col), 12칸 기준 |

### 반응형 단위

| 브레이크포인트 | 클래스 접두사 | 기준 너비 |
|----------------|----------------|-----------|
| Extra small | (없음) | <576px |
| Small | `sm` | ≥576px |
| Medium | `md` | ≥768px |
| Large | `lg` | ≥992px |
| Extra large | `xl` | ≥1200px |
| XXL | `xxl` | ≥1400px |

---

## 🧩 주요 컴포넌트

Bootstrap은 다양한 **UI 컴포넌트**를 제공합니다.

| 컴포넌트 | 설명 |
|----------|------|
| 버튼 (`.btn`) | 기본, outline, 크기, 색상 지정 가능 |
| 폼 (`form-control`, `form-group`) | 입력창, 체크박스 등 스타일 제공 |
| 카드 (`.card`) | 제목/본문/푸터를 가진 박스 |
| 내비게이션 (`.navbar`) | 상단 메뉴바 구성 |
| 모달 (`.modal`) | 팝업 창 구현 |
| 드롭다운 (`.dropdown`) | 메뉴, 선택지 |
| 토스트 (`.toast`) | 알림 UI |
| 배지 (`.badge`) | 숫자 표시용 작은 UI |
| 스피너 (`.spinner-border`) | 로딩 애니메이션 |

### 버튼 예제

```html
<button class="btn btn-primary">기본 버튼</button>
<button class="btn btn-outline-success">테두리 버튼</button>
```

---

## 🛠️ 유틸리티 클래스

Bootstrap은 다양한 **헬퍼 클래스**를 제공하여 별도 CSS 없이 빠르게 스타일링할 수 있습니다.

### 레이아웃

- `d-flex`, `justify-content-center`, `align-items-center` 등

### 여백/간격

- `m-1`, `p-3`, `mt-2`, `px-4`  
  (m: margin, p: padding, t/b/l/r/x/y)

### 색상

- `text-primary`, `bg-light`, `border-danger`

### 타이포그래피

- `text-center`, `fw-bold`, `fst-italic`, `lh-lg`

---

## 🎨 커스터마이징

1. **SCSS 파일 수정**
   - Bootstrap은 Sass 기반으로 작성되어 변수 수정이 용이합니다.
   - `$primary`, `$font-family-base`, `$border-radius` 등 커스터마이징 가능

2. **유틸리티 API**
   - Bootstrap 5부터 유틸리티 클래스를 커스터마이징할 수 있는 설정 제공

3. **테마 사용**
   - [https://bootswatch.com](https://bootswatch.com) 등에서 다양한 무료 테마 제공

---

## ⚖️ 장점 vs 단점

| 장점 | 단점 |
|------|------|
| 빠른 개발 속도 | 디자인이 비슷해 보일 수 있음 |
| 반응형 웹 지원 | 처음 배우면 클래스가 많아 복잡함 |
| 다양한 컴포넌트 제공 | 모든 기능을 쓰지 않으면 용량 낭비 가능 |
| 커뮤니티와 예제 풍부 | 고급 사용자 정의는 한계가 있음 |

---

## 📚 참고 사이트

- [🔗 Bootstrap 공식 문서](https://getbootstrap.com)
- [🎨 Bootswatch 테마](https://bootswatch.com)
- [📦 Bootstrap Icons](https://icons.getbootstrap.com)
- [🔧 Bootstrap Build 커스터마이저](https://bootstrap.build/)

---

## 🧠 마무리

Bootstrap은 프론트엔드 개발 입문자부터 프로토타이핑이 필요한 개발자, 디자이너까지  
**빠르게 반응형 웹을 만들고 싶을 때 매우 강력한 도구**입니다.

> 단순히 사용하는 데서 그치지 않고, **Sass 커스터마이징**과 **유틸리티 조합 설계**를 이해하면 Bootstrap을 더욱 효과적으로 활용할 수 있습니다.