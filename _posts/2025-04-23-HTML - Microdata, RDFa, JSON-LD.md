---
layout: post
title: HTML - Microdata, RDFa, JSON-LD
date: 2025-04-23 20:20:23 +0900
category: HTML
---
# 📦 Microdata, RDFa, JSON-LD 완전 가이드

## ✅ 왜 필요한가?

HTML 문서 안에는 사람이 보기에 정보가 있지만, **검색 엔진이나 기계는 의미를 완전히 이해하지 못합니다.**  
예를 들어, `<p>이 제품은 20,000원입니다.</p>`라는 문장이 있어도 기계는 그것이 **상품 가격**이라는 사실을 모릅니다.

이를 해결하기 위해 HTML 문서에 **구조화된 데이터(Semantic Metadata)**를 추가하는 기술이 바로:

- ✅ **Microdata**
- ✅ **RDFa (Resource Description Framework in Attributes)**
- ✅ **JSON-LD (JavaScript Object Notation for Linked Data)**

이 세 가지입니다.

---

## 🌐 공통 기반: [Schema.org](https://schema.org)

세 기술 모두 **[schema.org](https://schema.org)**에서 정의한 어휘(vocabulary)를 사용합니다.

- 상품 (`Product`)
- 사람 (`Person`)
- 리뷰 (`Review`)
- 기사 (`Article`)
- 이벤트 (`Event`)
- 조직 (`Organization`) 등

---

## 1️⃣ Microdata

### 📌 개요

- HTML5에서 등장
- HTML 요소에 `itemscope`, `itemtype`, `itemprop` 속성을 추가해 구조화된 데이터를 정의
- 사람이 보기에 자연스러운 HTML 안에 기계가 읽을 수 있는 의미를 삽입

### ✅ 예제: 상품 정보

```html
<div itemscope itemtype="https://schema.org/Product">
  <h2 itemprop="name">무선 키보드</h2>
  <p itemprop="description">편안한 키감의 무선 키보드입니다.</p>
  <p>가격: <span itemprop="price">20000</span>원</p>
</div>
```

### 🧩 주요 속성

| 속성 | 설명 |
|------|------|
| `itemscope` | 이 요소가 마이크로데이터 블록임을 정의 |
| `itemtype` | schema.org 타입 지정 |
| `itemprop` | 이 항목의 속성(property) 지정 |

---

## 2️⃣ RDFa (Lite)

### 📌 개요

- RDFa는 W3C 표준으로, HTML, XML, SVG 등에 메타데이터를 삽입할 수 있는 속성 기반 기술
- `vocab`, `typeof`, `property`, `resource`, `about` 등 속성 사용
- Microdata보다 더 유연하고 확장성이 높음

### ✅ 예제: 상품 정보

```html
<div vocab="https://schema.org/" typeof="Product">
  <h2 property="name">무선 키보드</h2>
  <p property="description">편안한 키감의 무선 키보드입니다.</p>
  <p>가격: <span property="price">20000</span>원</p>
</div>
```

### 🧩 주요 속성

| 속성 | 설명 |
|------|------|
| `vocab` | 사용할 어휘(schema.org 등) 정의 |
| `typeof` | 객체의 타입 지정 |
| `property` | 해당 속성(property)의 이름 |
| `resource` / `about` | 리소스의 URI 지정 가능 |

---

## 3️⃣ JSON-LD (가장 권장됨)

### 📌 개요

- JSON 포맷으로 작성되어 **HTML과 분리**
- `<script type="application/ld+json">` 안에 구조화된 데이터 작성
- 구글, 빙 등 최신 검색 엔진에서 가장 **권장**하는 방식

### ✅ 예제: 상품 정보

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "무선 키보드",
  "description": "편안한 키감의 무선 키보드입니다.",
  "price": "20000"
}
</script>
```

### 👍 장점

- HTML과 분리되어 코드가 깔끔함
- 유지보수 용이
- 조건부 렌더링에 유리 (SPA 등과 호환성 좋음)

---

## 🔄 Microdata vs RDFa vs JSON-LD

| 항목 | Microdata | RDFa | JSON-LD |
|------|-----------|------|---------|
| 형식 | HTML 속성 | HTML 속성 | JS 내 JSON |
| HTML과 분리 | ❌ 불가능 | ❌ 불가능 | ✅ 가능 |
| 가독성 | 보통 | 낮음 | 높음 |
| 확장성 | 중간 | 높음 | 매우 높음 |
| 표준 | HTML5 | W3C RDF 표준 | W3C JSON-LD 표준 |
| 권장 여부 | ❌ 줄어듦 | ❌ 적음 | ✅ 적극 권장 (Google) |

---

## 📌 언제 사용하나요?

- **SEO 강화를 원할 때** (검색엔진에 의미 전달)
- **Google 리치 스니펫 노출**  
  → 별점, 가격, 리뷰, 저자, 날짜 등
- **지식 패널 / 뉴스 피드 강화**
- **전시회, 기사, 레시피, 제품 상세 페이지 등**

---

## 🧪 테스트 도구

- ✅ [Google 구조화된 데이터 테스트 도구 (Rich Results Test)](https://search.google.com/test/rich-results)
- ✅ [Schema.org Validator](https://validator.schema.org/)
- ✅ [Yandex Structured Data Validator](https://webmaster.yandex.com/tools/microtest/)

---

## 💡 실전 팁

- 구글은 **JSON-LD 방식만 공식 지원**합니다 (Microdata도 해석하긴 함)
- `@context`는 항상 `"https://schema.org"`로 지정
- 복잡한 데이터는 JSON-LD가 훨씬 적합 (중첩, 리스트 등)
- React/Vue 등 SPA에서 SSR로 삽입할 때도 JSON-LD가 유리

---

## 🧠 마무리

| 요약 | 내용 |
|------|------|
| 공통 목적 | 구조화된 데이터를 검색엔진이 이해하게 하기 |
| 주요 사용처 | SEO, 구글 리치 결과, SNS 썸네일 등 |
| 가장 권장 방식 | ✅ JSON-LD |
| 핵심 어휘 | [schema.org](https://schema.org) |

---

## 📚 참고 자료

- [Google Structured Data](https://developers.google.com/search/docs/appearance/structured-data/intro)
- [Schema.org 공식 문서](https://schema.org/)
- [MDN - Microdata](https://developer.mozilla.org/en-US/docs/Web/HTML/Microdata)
- [W3C - RDFa](https://www.w3.org/TR/rdfa-lite/)
- [JSON-LD 소개](https://json-ld.org/)