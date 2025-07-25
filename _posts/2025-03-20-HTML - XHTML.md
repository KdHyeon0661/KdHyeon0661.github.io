---
layout: post
title: HTML - XHTML
date: 2025-03-20 20:20:23 +0900
category: HTML
---
# 📘 XHTML 완전 정복 – HTML보다 엄격한 문법의 마크업 언어

HTML은 유연하고 자유롭지만 때론 문법이 너무 느슨해 문제를 일으키곤 합니다. 이를 보완하기 위해 등장한 것이 바로 **XHTML (eXtensible HyperText Markup Language)**입니다. 이 글에서는 XHTML이 무엇인지, 어떻게 작성하고 사용하는지, 그리고 HTML과의 차이점을 중심으로 자세히 설명합니다.

---

## 📌 XHTML이란?

- **XHTML**은 HTML을 XML 문법으로 재작성한 마크업 언어입니다.
- **W3C**에서 제안한 표준이며, **엄격한 문법**을 따릅니다.
- HTML의 구조적 장점 + XML의 일관성과 확장성을 결합한 형태입니다.

```text
HTML 4.01 + XML 문법 = XHTML 1.0
```

---

## ✅ XHTML을 사용하는 이유

| 장점 | 설명 |
|------|------|
| XML과 호환 | 다른 XML 기반 기술(SVG, MathML 등)과 결합 쉬움 |
| 엄격한 문법 | 오류를 줄이고, 구조적으로 정돈된 문서 생성 |
| 미래 확장성 | XML 파서나 웹 서비스와 쉽게 연동 가능 |
| 표준 지향 개발 | 잘 정의된 문서 구조 보장 |

---

## 🔧 XHTML 문서 구조

```html
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN"
  "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">

<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="ko" lang="ko">
  <head>
    <meta http-equiv="Content-Type" content="application/xhtml+xml; charset=UTF-8" />
    <title>XHTML 예제</title>
  </head>
  <body>
    <h1>XHTML 문서입니다</h1>
    <p>엄격한 문법을 따릅니다.</p>
  </body>
</html>
```

### 설명:
- `<?xml version="1.0"?>`: XML 문서임을 명시
- `<!DOCTYPE ...>`: 문서 유형 정의(DTD)
- `<html>` 태그에 `xmlns` 속성 필수
- 모든 태그는 **소문자 사용**
- 모든 태그는 **반드시 닫아야 함**
- 빈 요소는 **셀프 클로징 `<br />`, `<img />`** 형태 사용

---

## ⚠️ XHTML 작성 시 주요 규칙

| 규칙 | 설명 | 예시 |
|------|------|------|
| 태그는 소문자 | `<P>` → ❌, `<p>` → ✅ |
| 속성값은 반드시 따옴표 | `width=100` → ❌, `width="100"` → ✅ |
| 빈 태그는 닫아야 함 | `<br>` → ❌, `<br />` → ✅ |
| 모든 태그는 닫아야 함 | `<li>Item` → ❌, `<li>Item</li>` → ✅ |
| 중첩은 정확하게 | `<b><i>text</b></i>` → ❌ |

### ✅ 예시: 올바른 XHTML

```html
<img src="logo.png" alt="로고 이미지" />
<input type="text" name="email" />
<br />
```

---

## 📎 XHTML과 HTML의 차이

| 항목 | HTML | XHTML |
|------|------|--------|
| 문법 | 느슨함 | 엄격함 |
| 파서 | 브라우저 중심 | XML 파서 기반 |
| 태그 닫기 | 일부 생략 가능 | 반드시 닫아야 함 |
| 대소문자 | 대소문자 구분 없음 | **소문자만 허용** |
| 속성 | 따옴표 없어도 동작 | **반드시 따옴표** |
| 빈 태그 | `<br>` 가능 | `<br />` 필수 |

---

## 🚀 실전 XHTML 예제

```xhtml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
  "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="ko" lang="ko">
  <head>
    <meta http-equiv="Content-Type" content="application/xhtml+xml; charset=UTF-8" />
    <title>XHTML 실전</title>
  </head>
  <body>
    <h1>환영합니다!</h1>
    <p>이 페이지는 <strong>XHTML</strong>로 작성되었습니다.</p>
    <img src="welcome.jpg" alt="환영 이미지" width="300" height="200" />
    <hr />
    <a href="https://www.example.com">예제 사이트</a>
  </body>
</html>
```

---

## ⚙️ 웹 서버 설정과 MIME 타입 주의

브라우저가 XHTML을 올바르게 해석하려면 올바른 **MIME 타입**이 필요합니다.

| MIME Type | 설명 |
|-----------|------|
| `application/xhtml+xml` | 진짜 XHTML 문서 (권장, 단 IE는 지원 안함) |
| `text/html` | HTML처럼 처리 (IE 호환을 위해 사용됨) |

> 대부분의 사이트에서는 XHTML을 HTML처럼 처리 (`text/html`)하지만, 진정한 XHTML 사용을 위해선 MIME 설정이 필요합니다.

---

## ❗ XHTML 사용 시 주의사항

- JavaScript는 CDATA로 감싸야 XML 파서 오류 방지

```html
<script type="text/javascript">
  //<![CDATA[
  alert("XHTML에서 JS 사용");
  //]]>
</script>
```

- form 태그 내부에 `<input />`, `<label />` 등도 전부 **셀프 클로징 형태**로 작성

---

## ✅ XHTML은 지금도 사용할까?

- ✅ 교육용, 문서 기반 시스템에서는 사용됨
- ⚠️ 현대 웹(HTML5 중심)에서는 거의 사용하지 않음
- ❌ `application/xhtml+xml`은 IE에서 미지원
- ✅ HTML5 + XML 구조 사용을 원할 경우, **HTML5 문법 + strict 스타일**로 개발하는 것이 대세

---

## ✅ 정리

| 특징 | XHTML |
|------|--------|
| 기반 | XML 기반 HTML |
| 목적 | 문법 엄격 + 구조 정돈 |
| 태그 규칙 | 닫기 필수, 속성값 따옴표 필수 |
| 사용법 | XML 선언, DTD 정의, MIME 설정 |
| 현대성 | HTML5 등장 이후 감소 추세 |
| 사용처 | XML 기반 시스템, 디지털 문서, 교육 등 |

---

**XHTML**은 HTML보다 더 엄격하고 정돈된 문서를 만들 수 있지만, 현대 웹에서는 **HTML5**가 주류입니다.  
그러나 XML 기반 시스템이나 PDF 변환, 전자 문서 표준화 등에서는 여전히 XHTML이 사용되며,  
**웹 문서의 기본 구조와 문법에 대한 깊은 이해를 돕는 좋은 학습 수단**이기도 합니다.