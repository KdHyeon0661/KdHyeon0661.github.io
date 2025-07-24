---
layout: post
title: HTML - iframe
date: 2025-03-07 19:20:23 +0900
category: HTML
---
# HTML <iframe> 완벽 가이드 – 사용법부터 보안까지

`<iframe>`은 HTML 문서 내에 **다른 HTML 문서나 웹페이지를 삽입**할 수 있는 태그입니다.  
예를 들어, 유튜브 영상, 외부 사이트, 문서 뷰어 등을 불러오는 데 자주 사용됩니다.

하지만 `<iframe>`은 **보안상 주의가 필요한 태그**이기도 합니다. 이 글에서는  
`<iframe>`의 기본 구조, 실용적인 활용 예시, **보안 리스크**와 **해결책**까지 다뤄봅니다.

---

## ✅ 1. <iframe> 기본 구조

```html
<iframe src="https://example.com" width="600" height="400"></iframe>
```

### 주요 속성

| 속성          | 설명                                  |
|---------------|----------------------------------------|
| `src`         | 삽입할 외부 URL                        |
| `width`/`height` | iframe 크기 지정 (px 또는 %)         |
| `title`       | 접근성용 설명 (스크린리더에 유용)     |
| `allow`       | iframe 내 기능 허용 (카메라, 위치 등) |
| `loading`     | `"lazy"`로 설정 시 지연 로딩 가능      |
| `sandbox`     | iframe 내에서 권한 제한 (보안 필수!)  |

---

## 🧪 2. 활용 예시

### 유튜브 영상 삽입

```html
<iframe 
  width="560" 
  height="315" 
  src="https://www.youtube.com/embed/dQw4w9WgXcQ" 
  title="YouTube video player" 
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" 
  allowfullscreen>
</iframe>
```

### 구글 지도 삽입

```html
<iframe 
  src="https://www.google.com/maps/embed?pb=..." 
  width="600" 
  height="450" 
  style="border:0;" 
  allowfullscreen="" 
  loading="lazy" 
  referrerpolicy="no-referrer-when-downgrade">
</iframe>
```

---

## 🔐 3. 보안상 문제점

`<iframe>`은 외부 콘텐츠를 불러오는 만큼, **보안상 주의가 필요한 요소**입니다.

### 1. Clickjacking (클릭 재킹)

iframe을 투명하게 오버레이해서 사용자의 클릭을 유도하는 공격.

예시:  
사용자가 보이지 않는 로그인 버튼을 클릭하게 유도해 세션 탈취.

### 2. Cross-site scripting (XSS) + iframe

취약한 사이트를 iframe으로 삽입하면 악성 스크립트가 실행될 수 있음.

### 3. 피싱 사이트 내 삽입

정상적인 사이트처럼 보이지만, iframe 내부에서 악성 동작을 수행하는 경우.

### 4. 세션 탈취 / 토큰 리플레이

iframe 내 페이지에서 세션 정보를 훔쳐서 다른 페이지로 전송할 수도 있음.

---

## 🛡️ 4. 보안 대응 방법

### ✅ 1. `sandbox` 속성 사용

`sandbox`는 iframe의 기능을 제한해주는 강력한 보안 수단입니다.

```html
<iframe src="https://example.com" sandbox></iframe>
```

| 옵션                     | 기능 설명                                       |
|--------------------------|------------------------------------------------|
| `sandbox` (단독)         | 거의 모든 기능 제한                           |
| `allow-scripts`          | JavaScript만 허용 (form 제출, 팝업 등은 차단) |
| `allow-same-origin`      | 동일 출처로 취급 (XSS 가능성이 커짐)          |
| `allow-forms`            | 폼 제출 허용                                   |
| `allow-popups`           | 팝업 창 허용                                   |

🔐 **Tip:**  
최소 권한만 열어주고, 절대적으로 필요한 권한만 선택적으로 지정하세요.

---

### ✅ 2. Content-Security-Policy (CSP) 설정

서버단에서 iframe으로 불러올 수 있는 도메인을 제한하는 방식입니다.

**HTTP 응답 헤더 예시:**

```
Content-Security-Policy: frame-ancestors 'self' https://trusted-site.com
```

- `'self'`: 현재 사이트만 iframe 삽입 가능
- `none`: 어떤 곳에서도 iframe 삽입 불가
- `https://domain.com`: 특정 도메인만 허용

> 이 설정은 **iframe을 포함하는 쪽이 아니라 포함당하는 쪽**에서 설정해야 효과가 있습니다.

---

### ✅ 3. X-Frame-Options 설정

과거부터 사용되어온 방법으로, 사이트가 **iframe으로 삽입되는 것 자체를 차단**합니다.

**예시 (응답 헤더):**

```
X-Frame-Options: DENY
```

| 값          | 설명                            |
|-------------|---------------------------------|
| `DENY`      | iframe으로 절대 삽입 불가        |
| `SAMEORIGIN`| 같은 출처에서만 삽입 가능        |
| `ALLOW-FROM uri` | 특정 URI에서만 허용 (비표준) |

> 최신 브라우저에서는 CSP `frame-ancestors`가 더 권장됩니다.

---

### ✅ 4. Referrer Policy 및 Origin 확인

iframe 내에서 민감한 작업을 수행한다면,  
**Referrer 체크** 또는 **postMessage로 출처 검증**을 필수로 해야 합니다.

```javascript
window.addEventListener("message", function(event) {
  if (event.origin !== "https://trusted-origin.com") return;
  // 안전한 메시지 처리
});
```

---

## 🧠 마무리

| 핵심 요소 | 설명 |
|-----------|------|
| `<iframe>` | 외부 콘텐츠 삽입 가능하지만 보안 위험 존재 |
| `sandbox` | iframe 기능 제한, 보안 필수 |
| `CSP` | 어떤 사이트가 iframe으로 포함 가능한지 제어 |
| `X-Frame-Options` | iframe 삽입 자체를 제한 |
| `postMessage` | 출처 기반 메시지 통신, 보안 검증 필요 |

---

## ✅ 결론

`<iframe>`은 다양한 외부 콘텐츠를 손쉽게 불러오는 데 유용하지만,  
**적절한 보안 설정 없이 사용하면 심각한 취약점이 될 수 있습니다.**

**항상 다음을 지켜주세요:**

1. **`sandbox` 속성 필수 사용**
2. **CSP로 iframe 포함 제한**
3. **출처 확인 & 메시지 검증**
4. **iframe 삽입 대상 페이지에도 보안 헤더 적용**

이제 iframe을 자신 있게, **안전하게** 활용하세요!