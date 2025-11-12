---
layout: post
title: HTML - XHTML
date: 2025-03-20 20:20:23 +0900
category: HTML
---
# XHTML 완전 정복

## 1. XHTML이란?

- **XHTML (eXtensible HyperText Markup Language)**: **HTML을 XML 문법으로 재정의**한 마크업 언어.
- **W3C 표준**(XHTML 1.0/1.1). HTML의 표현력 + XML의 **엄격/일관/도구친화성**.
- 요약  
  ```
  HTML 4.01 + XML 문법 = XHTML 1.0
  ```

### 왜 쓸까?

| 장점 | 설명 |
|---|---|
| XML 호환 | XSLT, XPath, DOM Level 2+, SVG/MathML 등 XML 생태계와 자연스러운 결합 |
| 엄격한 문법 | **Well-formed** 강제 → 자동 처리/검증/변환 파이프라인에 적합 |
| 예측 가능한 파싱 | 에러 시 **즉시 중단**(fail-fast) → 품질 관리 용이 |
| 표준 지향 | 구조적 일관성, 상호운용성 강화 |

> 현대 웹의 주류는 **HTML5(HTML Living Standard)** 이지만, **전자문서·출판·XML 파이프라인**(예: PDF 변환, 서버사이드 XML 변환, 데이터 교환)에서는 XHTML이 여전히 유용하다.

---

## 2. XHTML 문서 기본 구조와 직렬화(Serialization)

### 2.1 XHTML 1.0 기본 예제(Strict)

```html
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN"
  "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="ko" lang="ko">
  <head>
    <!-- 주의: XML로 서비스할 때 meta http-equiv로 인코딩을 지정해도 효력이 없다 -->
    <meta http-equiv="Content-Type" content="application/xhtml+xml; charset=UTF-8" />
    <title>XHTML 예제</title>
  </head>
  <body>
    <h1>XHTML 문서입니다</h1>
    <p>엄격한 문법을 따릅니다.</p>
  </body>
</html>
```

**구성 포인트**

- `<?xml version="1.0" encoding="UTF-8"?>`: XML 선언(선택이지만 권장).  
- `<!DOCTYPE ...>`: DTD(Strict/Transitional/Frameset).  
- `<html xmlns="http://www.w3.org/1999/xhtml">`: **XML 네임스페이스** 필수.  
- `xml:lang` + `lang` 병행(레거시 UA 호환).

> **MIME/인코딩 규칙 중요:** XML로 서비스(`application/xhtml+xml`)할 경우 **HTTP 헤더의 Content-Type/charset 또는 XML 선언의 encoding**이 권위(source of truth). `meta http-equiv`는 **무시**된다.

---

## 3. XHTML 필수 문법 규칙(잘 틀리는 것들)

| 규칙 | 설명 | 예/비예 |
|---|---|---|
| **소문자 태그** | 태그/속성명은 소문자 | `<P>` ❌ → `<p>` ✅ |
| **속성값 항상 따옴표** | 값 미인용 금지 | `width=100` ❌ → `width="100"` ✅ |
| **모든 요소 닫기** | 명시적 종료 태그 | `<li>항목` ❌ → `<li>항목</li>` ✅ |
| **빈 요소는 self-closing** | 빈 요소는 `/`로 닫음 | `<br>` ❌ → `<br />` ✅ |
| **정확한 중첩** | 교차 중첩 금지 | `<b><i>x</b></i>` ❌ |
| **불린 속성 값 명시** | 속성 존재=참 패턴 금지 | `checked` ❌ → `checked="checked"` ✅ |

### 3.1 빈(Empty) 요소 목록 예
`<br />`, `<hr />`, `<img />`, `<meta />`, `<link />`, `<input />`, `<base />`, `<col />`, `<param />`, `<area />`, `<source />`, `<track />` …

> **주의:** `script`/`style`은 **빈 요소가 아니다**. `<script />`처럼 self-closing 하면 **내용이 없는 스크립트**가 된다.

### 3.2 엔티티(Entities) 주의
- 순수 XML에는 **사전정의 5종**만 존재: `&lt;`, `&gt;`, `&amp;`, `&quot;`, `&apos;`.  
- XHTML 1.x에서는 DTD로 HTML 엔티티 집합을 가져오지만, **네트워크에서 DTD를 못 가져오면 이름 엔티티가 실패**할 수 있다.  
  → **가급적 숫자 엔티티(예: `&#169;`/`&#xA9;`) 사용**을 권장.

---

## 4. HTML과의 차이 — 파서·오류 모델·속성 직렬화

| 항목 | HTML (text/html) | XHTML (application/xhtml+xml) |
|---|---|---|
| 파서 | HTML 토큰 규칙(관대한 복구) | **XML 파서(엄격·fatal error 시 렌더 중단)** |
| 닫힘 규칙 | 일부 생략 허용 | **모두 닫아야 함** |
| 불린 속성 | `checked`만으로 OK | **`checked="checked"`** 필수 |
| 빈 요소 | `<br>` | **`<br />`** |
| 인코딩 결정 | meta/http-equiv 가능 | **HTTP 헤더/ XML 선언**만 유효 |
| 엔티티 | 풍부한 이름 엔티티(HTML 파서 보유) | **DTD 의존/숫자 엔티티 권장** |

**실무 영향**  
- HTML은 **오류 관대** → 살아남지만 의도와 다를 수 있음.  
- XHTML은 **fail-fast** → 품질 보증에 유리하나, **사소한 오류도 전체 파기**.

---

## 5. 스크립트/스타일 포함 — CDATA와 이스케이프

XML 파서는 `<`/`&`에 민감하므로, `script`/`style` 내용에 **문자 데이터**가 들어갈 때 CDATA 또는 이스케이프가 필요하다.

### 5.1 권장 패턴 (CDATA)
```html
<script type="application/javascript">
//<![CDATA[
  if (a < b && c > 0) { alert("XHTML에서 JS 사용"); }
//]]>
</script>
```

```html
<style type="text/css">
/*<![CDATA[*/
  p:before { content: "<x>"; } /* '<' 리터럴 포함 가능 */
/*]]>*/
</style>
```

> 일부 UA/도구 호환을 위해 주석 혼합 패턴(과거 HTML 파서용 `/* */` 또는 `//`)을 덧대던 관습이 있으나, **현대 환경에서는 위와 같은 CDATA 패턴**이 간결하다.

### 5.2 self-closing 금지 예
```html
<!-- 빈 스크립트로 직렬화되어 의도한 코드가 실행되지 않음 -->
<script type="application/javascript" />
```

---

## 6. 네임스페이스와 XML 통합 — SVG/MathML 포함

XHTML의 강점: **다중 XML 네임스페이스 통합**.

### 6.1 SVG 포함
```html
<div>
  <svg xmlns="http://www.w3.org/2000/svg" width="120" height="60" viewBox="0 0 120 60">
    <circle cx="30" cy="30" r="20" fill="#0e7afe" />
    <text x="60" y="35" font-size="16" text-anchor="middle">SVG</text>
  </svg>
</div>
```

### 6.2 MathML 포함
```html
<div xmlns:m="http://www.w3.org/1998/Math/MathML">
  <m:math display="block">
    <m:mi>E</m:mi><m:mo>=</m:mo><m:mi>m</m:mi><m:msup><m:mi>c</m:mi><m:mn>2</m:mn></m:msup>
  </m:math>
</div>
```

> 각 요소의 **네임스페이스 선언**을 잊지 말 것. 상위에 선언하면 하위에서 상속 가능.

---

## 7. MIME 타입과 서버 설정

### 7.1 권장 MIME
- **진짜 XHTML로서 파싱**: `application/xhtml+xml`  
- **호환을 위해 HTML 파서로 처리**: `text/html` (XHTML 문법을 쓰더라도 HTML로 파싱됨)

**주의(레거시 IE):** `application/xhtml+xml` **미지원**. B2B/레거시 환경이면 **콘텐츠 협상** 또는 `text/html` 서비스가 현실적.

### 7.2 Apache 설정 예
```apache
# .xhtml을 XHTML로 서비스
AddType application/xhtml+xml .xhtml

# 강제 헤더(사이트 정책에 맞게)
<FilesMatch "\.xhtml$">
  Header set Content-Type "application/xhtml+xml; charset=UTF-8"
</FilesMatch>
```

### 7.3 Nginx 설정 예
```nginx
types {
  application/xhtml+xml  xhtml;
}
# 또는 location 기반
location ~ \.xhtml$ {
  add_header Content-Type "application/xhtml+xml; charset=UTF-8";
}
```

> **XML로 서비스할 때** 인코딩 지정은 **HTTP 헤더 또는 XML 선언**으로만 해야 하며, `meta http-equiv`는 효력이 없다.

---

## 8. DTO/폼/불린 속성 직렬화 — 자주 하는 실수

### 8.1 폼 요소와 불린 속성
```html
<form action="/submit" method="post" enctype="application/x-www-form-urlencoded">
  <label for="email">이메일</label>
  <input type="text" id="email" name="email" required="required" />
  <input type="checkbox" name="agree" value="yes" checked="checked" />
  <input type="submit" value="보내기" />
</form>
```
- `required="required"`, `checked="checked"`처럼 **명시적 값** 필요.

### 8.2 링크/이미지
```html
<a href="https://example.com" title="예제">링크</a>
<img src="/img/logo.png" alt="로고" width="120" height="32" />
```
- `img`는 **빈 요소 self-closing**.
- `alt` 필수(접근성 + 문법).

---

## 9. 오류 모델 — 디버깅/검증

- HTML: 에러를 **가능한 복구**. 표시되지만 의도와 다를 수 있다.  
- XHTML(XML): **치명적인 오류 시 렌더 중단**(노출되지 않음).  

**검증 팁**
- XML 파서/유효성 검사기로 **well-formedness** 확인(태그 닫힘, 엔티티, 중첩).  
- DTD 유효성 검사(옵션)로 엔티티/속성/콘텐츠 모델 확인.  
- CI에 **lint + 검증** 투입 → 품질 담보.

---

## 10. Polyglot & XHTML5 — 현대적 전략

### 10.1 Polyglot Markup(양쪽 파서 호환)
- 문법을 **HTML 파서·XML 파서 모두에서 합법**이 되도록 제약.  
- 규칙 예: **소문자 태그/속성**, **속성값 모두 인용**, **빈 요소에 `/>`**, **엔티티는 숫자 권장**, **불린 속성에 값 지정**, `id` 유일성 등.

### 10.2 XHTML5(HTML5 어휘의 XML 직렬화)
- HTML Living Standard의 **요소/속성 어휘** + **XML 직렬화 규칙**.  
- Doctype은 단순화 가능(일부는 omit), **네임스페이스는 동일**.  
- 서비스는 `application/xhtml+xml`. (레거시 대응 필요 시 **콘텐츠 협상**)

**간단 예 (XHTML5 풍)**
```html
<?xml version="1.0" encoding="UTF-8"?>
<html xmlns="http://www.w3.org/1999/xhtml" lang="ko" xml:lang="ko">
  <head>
    <meta charset="UTF-8" />
    <title>XHTML5 스타일</title>
    <link rel="stylesheet" href="/app.css" />
  </head>
  <body>
    <header><h1>헤더</h1></header>
    <main><article><h2>본문</h2><p>XML 직렬화 규칙 준수</p></article></main>
    <footer>&#169; 2025</footer>
  </body>
</html>
```

---

## 11. 접근성·SEO·국제화

- `lang` + `xml:lang`: 언어 인식 향상(스크린 리더, 검색엔진).  
- 제목 구조 `<h1>`~`<h6>` 엄격 중첩, 리스트/테이블 시맨틱 태그 준수.  
- 문자셋은 **UTF-8** 권장. **XML 선언/HTTP 헤더**에서 일치.

---

## 12. 실전 종합 예제(보안·SVG·스크립트 포함)

```html
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN"
  "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="ko" lang="ko">
  <head>
    <meta http-equiv="Content-Type" content="application/xhtml+xml; charset=UTF-8" />
    <title>종합 예제</title>

    <!-- 외부 CSS -->
    <link rel="stylesheet" href="/assets/style.css" type="text/css" />

    <!-- 간단 스타일 (주의: XML에서는 <style> 내용에 '<' 등 포함 시 CDATA 고려) -->
    <style type="text/css">
      /*<![CDATA[*/
      body { font-family: system-ui, sans-serif; line-height: 1.7; }
      .btn { background: #0e7afe; color: #fff; padding: .5em 1em; border-radius: .5em; border: 0; }
      /*]]>*/
    </style>
  </head>
  <body>
    <h1>XHTML 종합</h1>

    <p>빈 요소: 라인<br />분리</p>

    <form action="/submit" method="post">
      <label for="q">검색</label>
      <input type="text" id="q" name="q" required="required" />
      <input type="submit" value="찾기" class="btn" />
    </form>

    <!-- SVG 네임스페이스 포함 -->
    <div>
      <svg xmlns="http://www.w3.org/2000/svg" width="160" height="60" viewBox="0 0 160 60">
        <rect x="10" y="10" width="140" height="40" rx="8" fill="#0e7afe" />
        <text x="80" y="36" font-size="16" text-anchor="middle" fill="#fff">SVG OK</text>
      </svg>
    </div>

    <!-- 스크립트: CDATA 블록 -->
    <script type="application/javascript">
    //<![CDATA[
      (function () {
        var btns = document.getElementsByClassName('btn');
        if (btns.length > 0) {
          btns[0].addEventListener('click', function (e) {
            // XHTML에서는 이벤트 연결도 동일하지만, 스크립트 직렬화에 주의
            // 예: '<' 나 '&'가 소스에 그대로 들어가면 XML 파서 오류 → CDATA로 보호
            console.log('버튼 클릭');
          }, false);
        }
      })();
    //]]>
    </script>
  </body>
</html>
```

---

## 13. 체크리스트(요약)

- [ ] 태그/속성 **소문자**  
- [ ] **모든 속성값 인용**  
- [ ] **모든 요소 닫기**, 빈 요소는 **`/>`**  
- [ ] **불린 속성 값 명시**(`disabled="disabled"`)  
- [ ] **정확한 중첩**  
- [ ] **네임스페이스** 명시(`xmlns="..."`)  
- [ ] **엔티티**는 가급적 **숫자 엔티티** 사용  
- [ ] `script`/`style`의 `<`/`&`는 **CDATA**로 보호  
- [ ] **MIME/인코딩**: XML은 **HTTP 헤더/선언**로 결정  
- [ ] 레거시 브라우저 필요 시 **text/html** 또는 **Polyglot** 전략

---

## 14. 자주 묻는 질문(FAQ)

**Q. .xhtml로 저장만 하면 XHTML인가요?**  
A. **아니오.** **문법(Well-formed) + 올바른 MIME/인코딩**이 만족되어야 XML로서 XHTML이다.

**Q. `meta charset`이면 충분한가요?**  
A. **XML 컨텍스트에서는 불충분.** `application/xhtml+xml`로 서비스 시 **HTTP 헤더/ XML 선언**으로 인코딩을 지정해야 한다.

**Q. self-closing을 모든 태그에 써도 되나요?**  
A. **콘텐츠가 없는 경우에만.** `<script />`처럼 쓰면 **내용이 없는 스크립트**가 되어 코드가 사라진다.

**Q. HTML5 기능을 쓰면서 XHTML처럼 엄격하게?**  
A. **XHTML5(HTML 어휘의 XML 직렬화)** 또는 **Polyglot**을 고려. 레거시 호환은 콘텐츠 협상.

---

## 15. 결론

- XHTML은 **엄격한 문법 + XML 생태계 통합**이 강점.  
- 현대 웹앱은 대부분 **HTML5**로 충분하지만, **전자문서/출판/정적 변환 파이프라인**에서는 XHTML이 여전히 가치가 크다.  
- 실무에서는 **문법·MIME·인코딩·엔티티·스크립트 직렬화**가 핵심 함정. 본 가이드의 체크리스트와 예제를 그대로 적용하면 **깨지지 않는 XHTML**을 안정적으로 운용할 수 있다.