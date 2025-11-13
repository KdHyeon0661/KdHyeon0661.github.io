---
layout: post
title: HTML - Microdata, RDFa, JSON-LD
date: 2025-04-23 20:20:23 +0900
category: HTML
---
# Microdata, RDFa, JSON-LD

## 0. 공통 기반 — Schema.org란?

- **schema.org**는 웹의 대표 검색엔진들이 합의한 **공통 어휘(vocabulary) 집합**입니다.
- 대표 타입: `Product`, `Article`, `Event`, `FAQPage`, `HowTo`, `Organization`, `Person`, `BreadcrumbList`, `VideoObject` 등.
- **세 방식(Microdata/RDFa/JSON-LD)은 모두 schema.org 어휘를 공유**합니다.

---

## 1. 세 방식 한눈에 비교

| 구분 | Microdata | RDFa (Lite) | JSON-LD |
|---|---|---|---|
| 표기 방식 | HTML 속성(`itemscope`, `itemprop`) | HTML 속성(`typeof`, `property`) | `<script type="application/ld+json">` 내부 JSON |
| HTML과의 분리 | 어려움 | 어려움 | **완전 분리(권장)** |
| 가독성/유지보수 | 텍스트 섞여 복잡 | 속성 다양, 복잡 | **깨끗함**, 템플릿·SSR 친화 |
| 복잡 데이터 모델링 | 보통 | 유연 | **매우 우수** |
| 검색엔진 권장(2025) | 제한적 | 제한적 | **강력 권장** |

> 실무에서는 **JSON-LD가 1순위**입니다. (Microdata/RDFa도 해석되지만 관리 난이도가 상승)

---

## 2. “같은 상품 페이지”를 세 방식으로 — 축약 기본 예제

아래는 동일한 `Product` 정보를 Microdata/RDFa/JSON-LD로 각각 마크업하는 최소 예시입니다.

### 2.1 Microdata
```html
<div itemscope itemtype="https://schema.org/Product">
  <h1 itemprop="name">무선 키보드</h1>
  <img src="/img/keyboard.jpg" alt="무선 키보드" itemprop="image">
  <p itemprop="description">편안한 키감의 초경량 무선 키보드</p>
  <span itemprop="brand">KeyWave</span>
  <div itemprop="offers" itemscope itemtype="https://schema.org/Offer">
    <link itemprop="availability" href="https://schema.org/InStock">
    <meta itemprop="priceCurrency" content="KRW">
    <span itemprop="price">20000</span>원
  </div>
</div>
```

### 2.2 RDFa (Lite)
```html
<div vocab="https://schema.org/" typeof="Product">
  <h1 property="name">무선 키보드</h1>
  <img src="/img/keyboard.jpg" alt="무선 키보드" property="image">
  <p property="description">편안한 키감의 초경량 무선 키보드</p>
  <span property="brand">KeyWave</span>
  <div typeof="Offer" property="offers">
    <link property="availability" href="https://schema.org/InStock">
    <meta property="priceCurrency" content="KRW">
    <span property="price">20000</span>원
  </div>
</div>
```

### 2.3 JSON-LD (권장)
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@type":"Product",
  "@id":"https://example.com/product/keyboard#product",
  "name":"무선 키보드",
  "image":"https://example.com/img/keyboard.jpg",
  "description":"편안한 키감의 초경량 무선 키보드",
  "brand":{"@type":"Brand","name":"KeyWave"},
  "offers":{
    "@type":"Offer",
    "priceCurrency":"KRW",
    "price":"20000",
    "availability":"https://schema.org/InStock",
    "url":"https://example.com/product/keyboard"
  }
}
</script>
```

---

## 3. JSON-LD 실무 패턴 — 꼭 알아야 할 10가지

JSON-LD는 구조가 자유롭고 **@graph**, **@id** 링크, **sameAs** 등 확장 기능이 강력합니다.

### 3.1 `@id`와 `mainEntityOfPage`로 ID 고정
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@type":"Article",
  "@id":"https://example.com/blog/semantic-web#article",
  "headline":"구조화 데이터 가이드",
  "mainEntityOfPage":"https://example.com/blog/semantic-web",
  "author":{"@type":"Person","name":"Do Hyun Kim"},
  "datePublished":"2025-11-01T09:00:00+09:00",
  "dateModified":"2025-11-09T12:00:00+09:00",
  "image":"https://example.com/og/semantic.jpg"
}
</script>
```
- **@id**는 컨텐츠의 **영속 식별자**(fragment 포함)를 고정합니다. 캐시·중복 색인 방지에 도움.

### 3.2 여러 엔티티를 **@graph**로 묶기
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@graph":[
    {
      "@type":"Organization",
      "@id":"https://example.com/#org",
      "name":"Example Inc.",
      "url":"https://example.com",
      "logo":{"@type":"ImageObject","url":"https://example.com/img/logo.png"},
      "sameAs":["https://twitter.com/example","https://www.youtube.com/@example"]
    },
    {
      "@type":"WebSite",
      "@id":"https://example.com/#website",
      "url":"https://example.com",
      "name":"Example Blog",
      "publisher":{"@id":"https://example.com/#org"},
      "potentialAction":{
        "@type":"SearchAction",
        "target":"https://example.com/search?q={search_term_string}",
        "query-input":"required name=search_term_string"
      }
    }
  ]
}
</script>
```

### 3.3 `BreadcrumbList`(빵크럼) — 탐색 컨텍스트 강화
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@type":"BreadcrumbList",
  "itemListElement":[
    {"@type":"ListItem","position":1,"name":"홈","item":"https://example.com/"},
    {"@type":"ListItem","position":2,"name":"블로그","item":"https://example.com/blog"},
    {"@type":"ListItem","position":3,"name":"구조화 데이터 가이드","item":"https://example.com/blog/semantic-web"}
  ]
}
</script>
```

### 3.4 `Product` 심화 — 가격·재고·평점·SKU/GTIN
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@type":"Product",
  "@id":"https://example.com/product/keyboard#product",
  "name":"무선 키보드",
  "image":["https://example.com/img/keyboard-1.jpg","https://example.com/img/keyboard-2.jpg"],
  "description":"저소음, 멀티페어링, 1년 배터리",
  "sku":"KB-2025-BLACK",
  "gtin13":"8801234567890",
  "brand":{"@type":"Brand","name":"KeyWave"},
  "aggregateRating":{"@type":"AggregateRating","ratingValue":"4.6","reviewCount":"128"},
  "offers":{
    "@type":"Offer",
    "url":"https://example.com/product/keyboard",
    "priceCurrency":"KRW",
    "price":"20000",
    "priceValidUntil":"2026-12-31",
    "availability":"https://schema.org/InStock",
    "itemCondition":"https://schema.org/NewCondition",
    "shippingDetails":{
      "@type":"OfferShippingDetails",
      "shippingRate":{"@type":"MonetaryAmount","value":"0","currency":"KRW"},
      "shippingDestination":{"@type":"DefinedRegion","addressCountry":"KR"}
    }
  }
}
</script>
```
- **가격/통화/유효기간**을 ISO 포맷으로 명확히.
- **식별자**: `sku`, `gtin8/12/13/14`, `mpn` 등 하나 이상 제공 권장.

### 3.5 `Article/NewsArticle/BlogPosting` — 이미지·헤드라인 규칙
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@type":"BlogPosting",
  "headline":"JSON-LD로 SEO 리치결과 얻는 법",
  "image":["https://example.com/og/jsonld-1200x630.jpg"],
  "author":{"@type":"Person","name":"Do Hyun Kim"},
  "datePublished":"2025-11-09T10:00:00+09:00",
  "dateModified":"2025-11-09T10:30:00+09:00",
  "publisher":{
    "@type":"Organization",
    "name":"Example Inc.",
    "logo":{"@type":"ImageObject","url":"https://example.com/img/logo-600x60.png","width":600,"height":60}
  },
  "mainEntityOfPage":"https://example.com/blog/jsonld-rich-results"
}
</script>
```
- 헤드라인은 **간결**(과도한 반복/키워드 나열 금지).
- 이미지 1200×630 권장(큰 썸네일 카드 대응).

### 3.6 `FAQPage` — 리치결과에 자주 쓰는 패턴
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@type":"FAQPage",
  "mainEntity":[
    {
      "@type":"Question",
      "name":"반품은 어떻게 하나요?",
      "acceptedAnswer":{"@type":"Answer","text":"마이페이지 > 주문내역에서 반품 신청을 해주세요."}
    },
    {
      "@type":"Question",
      "name":"무상보증 기간은?",
      "acceptedAnswer":{"@type":"Answer","text":"구매일로부터 1년입니다."}
    }
  ]
}
</script>
```
- **실제 페이지에 보이는 Q/A와 동일**해야 하며, 과도한 마크업 남용 금지.

### 3.7 `HowTo` — 단계형 가이드(이미지·시간·재료)
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@type":"HowTo",
  "name":"무선 키보드 페어링 방법",
  "totalTime":"PT5M",
  "tool":[{"@type":"HowToTool","name":"블루투스 지원 PC/모바일"}],
  "step":[
    {"@type":"HowToStep","text":"전원 스위치를 켭니다."},
    {"@type":"HowToStep","text":"블루투스 페어링 버튼을 3초간 누릅니다."},
    {"@type":"HowToStep","text":"PC/모바일에서 'KeyWave Keyboard'를 선택합니다."}
  ]
}
</script>
```

### 3.8 `Event` — 시간/장소/티켓
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@type":"Event",
  "name":"프론트엔드 컨퍼런스 2026",
  "startDate":"2026-03-12T10:00:00+09:00",
  "endDate":"2026-03-12T18:00:00+09:00",
  "eventAttendanceMode":"https://schema.org/OfflineEventAttendanceMode",
  "eventStatus":"https://schema.org/EventScheduled",
  "location":{
    "@type":"Place",
    "name":"COEX",
    "address":{"@type":"PostalAddress","addressLocality":"서울","addressCountry":"KR"}
  },
  "offers":{"@type":"Offer","price":"99000","priceCurrency":"KRW","availability":"https://schema.org/InStock"},
  "organizer":{"@type":"Organization","name":"FE Korea"}
}
</script>
```

### 3.9 `VideoObject` — 재생/썸네일/길이
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@type":"VideoObject",
  "name":"Canvas로 게임 만들기",
  "description":"HTML5 Canvas 기초와 애니메이션",
  "thumbnailUrl":["https://example.com/thumbs/canvas.jpg"],
  "uploadDate":"2025-10-10T09:00:00+09:00",
  "duration":"PT12M33S",
  "contentUrl":"https://example.com/video/canvas.mp4",
  "embedUrl":"https://player.example.com/embed/abc123"
}
</script>
```

### 3.10 `Organization`/`WebSite` — 기본 신뢰 신호
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@graph":[
    {
      "@type":"Organization",
      "@id":"https://example.com/#org",
      "name":"Example Inc.",
      "url":"https://example.com",
      "logo":"https://example.com/img/logo.png",
      "sameAs":[
        "https://www.linkedin.com/company/example",
        "https://github.com/example"
      ]
    },
    {
      "@type":"WebSite",
      "@id":"https://example.com/#website",
      "name":"Example",
      "url":"https://example.com",
      "publisher":{"@id":"https://example.com/#org"},
      "potentialAction":{
        "@type":"SearchAction",
        "target":"https://example.com/search?q={search_term_string}",
        "query-input":"required name=search_term_string"
      }
    }
  ]
}
</script>
```

---

## 4. Microdata/RDFa 심화 포인트

### 4.1 Microdata 중첩(offers/review 등)
```html
<div itemscope itemtype="https://schema.org/Product">
  <span itemprop="name">무선 키보드</span>
  <div itemprop="aggregateRating" itemscope itemtype="https://schema.org/AggregateRating">
    <meta itemprop="ratingValue" content="4.6">
    <meta itemprop="reviewCount" content="128">
    평점 4.6 (128)
  </div>
</div>
```

### 4.2 RDFa 어휘 스위칭
```html
<!-- 기본 어휘를 schema.org로 선언 -->
<div vocab="https://schema.org/" typeof="Article">
  <span property="headline">제목</span>
  <!-- 필요 시 특정 property만 Dublin Core 등 외부 어휘로 표기 가능 -->
  <span property="dc:creator" xmlns:dc="http://purl.org/dc/elements/1.1/">Do Hyun Kim</span>
</div>
```

> Microdata/RDFa는 **HTML과 텍스트가 뒤섞여 가독성·유지보수가 어려워지기 쉬움**. 리팩토링/템플릿 분리가 힘들다면 JSON-LD 전환 검토.

---

## 5. SPA/SSR 환경: 삽입·동기화 전략

### 5.1 Next.js에서 JSON-LD 주입

{% raw %}
```jsx
import Head from "next/head";

export default function ProductSEO({ product }) {
  const jsonLd = {
    "@context":"https://schema.org",
    "@type":"Product",
    "@id":`${product.url}#product`,
    "name":product.name,
    "image":product.images,
    "description":product.description,
    "brand":{"@type":"Brand","name":product.brand},
    "offers":{
      "@type":"Offer",
      "priceCurrency":product.currency,
      "price":product.price,
      "availability":product.inStock ? "https://schema.org/InStock" : "https://schema.org/OutOfStock",
      "url":product.url
    }
  };
  return (
    <Head>
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }} />
    </Head>
  );
}
```
{% endraw %}

- **SSR 시점**에 JSON-LD가 HTML에 포함되도록 하세요(크롤러 호환성 향상).
- 클라이언트 전용 렌더 시, **초기 페인트 직후 삽입**하되 DOMContentLoaded 이전 가시성이 좋음.

### 5.2 다국어/멀티도메인
- `inLanguage` 사용 `("ko-KR", "en-US")`.
- 도메인별 **정규 URL/캐노니컬** 일관 유지.
- `@id`를 로케일별로 고정(언어 페이지마다 별도 @id 필요).

---

## 6. 리치결과 대상별 필수/권장 필드 베스트 프랙티스

| 대상 | 필수/권장 포인트 |
|---|---|
| Product | `name`, `image`, `description`, `offers.priceCurrency`, `offers.price`, `availability`, **식별자(sku/gtin/mpn)** |
| Article/BlogPosting | `headline`, `image`, `datePublished`, `dateModified`, `author`, `mainEntityOfPage` |
| FAQPage | 실제 노출 Q/A와 **동일**해야 하며 광고/홍보 문구 과다 금지 |
| Event | `name`, `startDate`, `location`, `offers`(유료 시), `eventStatus` |
| VideoObject | `name`, `description`, `thumbnailUrl`, `uploadDate`, `contentUrl`/`embedUrl`, `duration` |
| BreadcrumbList | 순서가 있는 `ListItem` with `position`, `name`, `item` |

> **경고**: “구조화 데이터 스팸” 판정(보이는 내용과 불일치, 과장, 숨김)은 리치결과 제외 및 신뢰도 하락으로 이어질 수 있습니다.

---

## 7. 검증/모니터링 — 배포 전·후 체크리스트

1. **스키마 문법 검사**
   - Google **Rich Results Test**
   - **Schema.org Validator**
2. **서치 콘솔**
   - 리치결과 보고서(유효/경고/오류 필드)
   - URL 검사 → 라이브 테스트
3. **상태 관찰**
   - 제품 가격/재고 자동 변경 시 **JSON-LD 동기**(크론·훅)
   - 이미지 404/리디렉트 점검
4. **일관성**
   - 페이지 본문·메타와 구조화 데이터 내용 **동일성 유지**
   - **@id**/`og:url`/`canonical` **정합성**
5. **성능/캐시**
   - SSR로 초기 HTML에 포함
   - 변동 필드는 캐시 무효화 루틴 구축

---

## 8. 자주 발생하는 오류와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| “Missing field ‘price’” | Product/Offer에 가격 누락 | `offers.price`와 `priceCurrency` 추가 |
| “Invalid enum value” | 잘못된 상수값 사용 | `availability` 등은 **공식 IRI** 사용(예: `https://schema.org/InStock`) |
| “Inconsistent URL” | `@id`, `og:url`, canonical 불일치 | 정규 URL 1개로 통일 |
| “Image too small” | 썸네일 규격 미달 | 권장 크기(예: 1200×630) 이상 이미지 사용 |
| “Content mismatch” | 본문과 구조화 데이터 내용 차이 | 페이지 실제 내용과 **동일**하게 유지 |

---

## 9. Microdata/RDFa ↔ JSON-LD 전환 요령

- **전략**: 우선 JSON-LD로 핵심 스니펫(Organization/WebSite/Product/Article/FAQ)을 구축 →
  Microdata/RDFa 잔존 시 **중복·충돌**을 피하도록 **동일 사실을 두 번 마크업하지 않기**(특히 Product 가격 등).
- **이관 순서**: Organization → WebSite(SearchAction) → Breadcrumb → Article/Product/Event/FAQ → VideoObject.

---

## 10. 예제 모음 — “한 페이지에 필요한 전형 조합”

### 10.1 블로그 포스트 + 빵크럼 + 사이트 정보
```html
<!-- Organization & Website & Breadcrumb & BlogPosting -->
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@graph":[
    {
      "@type":"Organization",
      "@id":"https://example.com/#org",
      "name":"Example Dev Blog",
      "url":"https://example.com",
      "logo":"https://example.com/img/logo.png",
      "sameAs":["https://github.com/example","https://x.com/example"]
    },
    {
      "@type":"WebSite",
      "@id":"https://example.com/#website",
      "url":"https://example.com",
      "name":"Example Dev Blog",
      "publisher":{"@id":"https://example.com/#org"},
      "potentialAction":{"@type":"SearchAction","target":"https://example.com/search?q={search_term_string}","query-input":"required name=search_term_string"}
    },
    {
      "@type":"BreadcrumbList",
      "@id":"https://example.com/blog/jsonld#breadcrumb",
      "itemListElement":[
        {"@type":"ListItem","position":1,"name":"홈","item":"https://example.com/"},
        {"@type":"ListItem","position":2,"name":"블로그","item":"https://example.com/blog"},
        {"@type":"ListItem","position":3,"name":"JSON-LD 가이드","item":"https://example.com/blog/jsonld"}
      ]
    },
    {
      "@type":"BlogPosting",
      "@id":"https://example.com/blog/jsonld#article",
      "headline":"JSON-LD 완전 가이드",
      "image":["https://example.com/og/jsonld.jpg"],
      "author":{"@type":"Person","name":"Do Hyun Kim"},
      "datePublished":"2025-11-08T10:00:00+09:00",
      "dateModified":"2025-11-09T09:20:00+09:00",
      "mainEntityOfPage":"https://example.com/blog/jsonld",
      "publisher":{"@id":"https://example.com/#org"}
    }
  ]
}
</script>
```

### 10.2 상품 상세 + FAQ(선택)
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@graph":[
    {
      "@type":"Product",
      "@id":"https://example.com/product/keyboard#product",
      "name":"무선 키보드",
      "image":["https://example.com/img/keyboard.jpg"],
      "description":"저소음·장시간 배터리·멀티페어링",
      "sku":"KB-2025",
      "brand":{"@type":"Brand","name":"KeyWave"},
      "aggregateRating":{"@type":"AggregateRating","ratingValue":"4.7","reviewCount":"372"},
      "offers":{
        "@type":"Offer",
        "url":"https://example.com/product/keyboard",
        "priceCurrency":"KRW",
        "price":"20000",
        "availability":"https://schema.org/InStock",
        "priceValidUntil":"2026-12-31"
      }
    },
    {
      "@type":"FAQPage",
      "mainEntity":[
        {"@type":"Question","name":"배터리는 얼마나 가나요?","acceptedAnswer":{"@type":"Answer","text":"일 평균 4시간 사용 기준 약 12개월입니다."}},
        {"@type":"Question","name":"운영체제 호환은?","acceptedAnswer":{"@type":"Answer","text":"Windows, macOS, Android, iOS를 지원합니다."}}
      ]
    }
  ]
}
</script>
```

---

## 11. 운영 관점 체크리스트(팀 적용)

- [ ] **스키마 카탈로그** 작성(페이지 유형별: 제품/기사/이벤트/FAQ/동영상)
- [ ] 필수/권장 필드 표준 정의(가격/통화/날짜 형식/이미지 규격)
- [ ] **생성 파이프라인**(SSR 템플릿·Head 주입 컴포넌트) 통일
- [ ] **데이터 소스**(DB/Headless CMS)와 JSON-LD 키 매핑 테이블 유지
- [ ] 변경 감지(가격/재고/일정) → 마크업 자동 갱신/캐시 무효화
- [ ] 배포 전 자동 **Rich Results Test**(CI 파이프라인에 통합)
- [ ] 서치 콘솔 경고/오류 알림 모니터링(주간 리포트)

---

## 12. Microdata/RDFa 예제 확장(참고)

### 12.1 Microdata로 FAQ
```html
<div itemscope itemtype="https://schema.org/FAQPage">
  <div itemscope itemprop="mainEntity" itemtype="https://schema.org/Question">
    <h3 itemprop="name">반품은 어떻게 하나요?</h3>
    <div itemscope itemprop="acceptedAnswer" itemtype="https://schema.org/Answer">
      <div itemprop="text">마이페이지 > 주문내역에서 반품 신청을 해주세요.</div>
    </div>
  </div>
</div>
```

### 12.2 RDFa로 Article
```html
<article vocab="https://schema.org/" typeof="Article">
  <h1 property="headline">시맨틱 마크업 가이드</h1>
  <img property="image" src="/og/semantic.jpg" alt="">
  <span property="author" typeof="Person"><span property="name">Do Hyun Kim</span></span>
  <time property="datePublished" datetime="2025-11-08T10:00:00+09:00">2025-11-08</time>
</article>
```

---

## 13. 테스트 도구 & 참고 링크

- Google **Rich Results Test** / **Search Console**(리치결과 보고서)
- **Schema.org Validator**
- **구현 가이드**: schema.org 타입 문서, Google Search Central의 각 리치결과 가이드

---

## 결론

- **목표**: 기계가 이해하는 **의미**를 부여해 **검색 노출 품질·클릭률·탐색성**을 높인다.
- **방법**: Microdata/RDFa도 가능하지만, **JSON-LD를 1순위**로 도입하고 `@id`/`@graph`/`sameAs`/`mainEntityOfPage` 등 **베스트 프랙티스**를 따른다.
- **운영**: 템플릿화·자동화·검증 파이프라인으로 유지보수 비용을 최소화한다.

_핵심 한 줄_: **“페이지에 보이는 사실을, JSON-LD로 정확히·일관되게 기술하고, 검증과 모니터링을 지속하라.”**
