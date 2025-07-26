---
layout: post
title: CSS - position 속성
date: 2025-04-01 20:20:23 +0900
category: CSS
---
# position 속성 (static, relative, absolute, fixed, sticky)

CSS의 `position` 속성은 HTML 요소의 위치를 **어떻게 배치할 것인지**를 정의합니다. `static`, `relative`, `absolute`, `fixed`, `sticky` 다섯 가지 주요 값이 있으며, 각각의 의미와 동작을 이해하면 UI를 세밀하게 조정할 수 있습니다.

---

## 1. static (기본값)

`static`은 **기본 위치 지정 방식**이며, 요소는 일반적인 문서 흐름에 따라 배치됩니다.  
`top`, `left`, `right`, `bottom` 속성이 **무시**됩니다.

```html
<div style="position: static; top: 20px;">나는 static이야 (top 무시됨)</div>
```

특징:
- 기본 배치 방식
- 위치 조정 불가 (top, left 등 무시)

---

## 2. relative

`relative`는 **자기 자신(static 기준)의 위치를 기준으로 이동**합니다.  
원래 위치는 **보존**되며, 문서 흐름에서 자리를 차지합니다.

```html
<div style="position: relative; top: 20px; left: 10px;">조금 이동한 relative</div>
```

특징:
- 원래 위치 기준으로 이동
- top/left/right/bottom 사용 가능
- 부모 요소가 absolute 자식의 기준점이 될 수도 있음

---

## 3. absolute

`absolute`는 **문서 흐름에서 제외**되며, 가장 가까운 `position: relative | absolute | fixed` 조상 요소를 기준으로 위치합니다.  
해당 기준 요소가 없으면 `body` 기준이 됩니다.

```html
<div style="position: relative;">
  <div style="position: absolute; top: 0; left: 0; background: yellow;">
    부모의 좌상단 기준으로 위치한 absolute
  </div>
</div>
```

특징:
- 문서 흐름에서 빠짐 (공간 차지 안 함)
- 기준 조상 요소를 기준으로 위치
- 레이어처럼 위에 올리기 좋음

---

## 4. fixed

`fixed`는 **화면(뷰포트)을 기준**으로 고정되어 스크롤에도 움직이지 않습니다.  
주로 상단 고정 헤더, 사이드바 등에 사용됩니다.

```html
<div style="position: fixed; top: 0; left: 0; width: 100%; background: lightblue;">
  항상 화면 상단에 고정된 fixed
</div>
```

특징:
- 브라우저 뷰포트를 기준
- 스크롤해도 고정
- 문서 흐름에서 제외됨

---

## 5. sticky

`sticky`는 `relative`처럼 동작하다가, 특정 스크롤 위치에 도달하면 `fixed`처럼 **고정**됩니다.  
스크롤이 특정 기준을 넘을 때까지는 `relative`, 넘으면 `fixed`로 바뀝니다.

```html
<div style="position: sticky; top: 0; background: lightgreen;">
  나는 sticky! 스크롤하면 고정됨
</div>
```

특징:
- 부모 요소 영역 내에서만 sticky 작동
- top 또는 bottom 등 기준 위치 필수
- 최신 브라우저에서 지원

---

## 6. 전체 예제 코드

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>position 속성 예제</title>
<style>
  body {
    height: 200vh; /* 스크롤을 위한 충분한 높이 확보 */
    margin: 20px;
    font-family: Arial, sans-serif;
  }

  .container {
    position: relative;
    margin-bottom: 60px;
    border: 2px solid #333;
    padding: 20px;
    height: 150px;
    overflow: visible;
  }

  .box {
    width: 120px;
    height: 80px;
    color: white;
    line-height: 80px;
    text-align: center;
    font-weight: bold;
    margin-bottom: 10px;
  }

  /* 1. static (기본값) */
  .static {
    position: static; /* 기본 위치 지정 */
    background-color: #777;
  }

  /* 2. relative (자기 위치 기준으로 이동) */
  .relative {
    position: relative;
    top: 10px;
    left: 30px;
    background-color: #1e90ff;
  }

  /* 3. absolute (가장 가까운 위치 지정된 조상 기준) */
  .absolute-container {
    position: relative; /* 위치 기준 조상 */
    background-color: #eee;
    height: 150px;
    border: 1px dashed #999;
  }
  .absolute {
    position: absolute;
    top: 20px;
    right: 20px;
    background-color: #28a745;
  }

  /* 4. fixed (뷰포트 기준 고정) */
  .fixed {
    position: fixed;
    bottom: 20px;
    left: 20px;
    background-color: #dc3545;
  }

  /* 5. sticky (스크롤 위치에 따라 고정/흐름) */
  .sticky-container {
    height: 200px;
    border: 1px solid #666;
    overflow: auto;
    margin-top: 30px;
    padding: 5px;
  }
  .sticky {
    position: sticky;
    top: 0;
    background-color: #ff9800;
    padding: 10px;
    font-weight: bold;
  }
</style>
</head>
<body>

<h2>1. position: static (기본값)</h2>
<div class="container">
  <div class="box static">Static</div>
  다른 요소들과 일반적인 문서 흐름에 따라 배치됩니다.
</div>

<h2>2. position: relative (자기 위치에서 이동)</h2>
<div class="container">
  <div class="box relative">Relative</div>
  원래 자리에서 top:10px, left:30px 만큼 이동했지만, 다른 요소 레이아웃에는 영향이 없습니다.
</div>

<h2>3. position: absolute (조상 중 위치 지정자 기준)</h2>
<div class="absolute-container">
  <p>이 컨테이너는 position: relative 상태입니다.</p>
  <div class="box absolute">Absolute</div>
</div>
<p>Absolute 박스는 부모 컨테이너의 위쪽 오른쪽에서 20px 떨어진 위치에 고정됩니다.</p>

<h2>4. position: fixed (뷰포트 기준 고정)</h2>
<div class="container" style="height: 300px;">
  <div class="box fixed">Fixed</div>
  화면을 스크롤해도 이 요소는 항상 뷰포트 왼쪽 아래에 고정됩니다.
</div>

<h2>5. position: sticky (스크롤 중 특정 위치에 고정)</h2>
<div class="sticky-container">
  <div class="sticky">Sticky Header</div>
  <p>스크롤을 내려서 이 헤더가 상단에 닿으면 고정됩니다.</p>
  <p>내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용</p>
  <p>내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용</p>
  <p>내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용 내용</p>
</div>

</body>
</html>
```

## 7. 비교 요약표

| 속성       | 문서 흐름 포함 | 위치 기준               | 스크롤 영향 |
|------------|----------------|--------------------------|--------------|
| `static`   | O              | 없음 (기본 흐름)         | O            |
| `relative` | O              | 자기 자신 위치            | O            |
| `absolute` | X              | 가장 가까운 positioned 조상 | X            |
| `fixed`    | X              | 브라우저 뷰포트           | X            |
| `sticky`   | O              | 뷰포트 + 부모 컨텍스트     | 조건부 고정   |

---

## 8. 활용 팁

- `absolute`를 사용할 때는 반드시 부모에 `position: relative`를 지정해야 제어가 쉬움
- `sticky`는 `overflow: hidden`이나 `overflow: auto`가 적용된 부모에서 동작 안 될 수 있음
- `z-index`와 조합하면 레이어 순서도 제어 가능

---

이처럼 `position` 속성은 **레이아웃과 인터랙션을 결정짓는 핵심 도구**입니다. 각 속성의 특성을 정확히 이해하고 활용해 보세요.