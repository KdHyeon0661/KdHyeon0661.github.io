---
layout: post
title: CSS - display 속성
date: 2025-04-01 19:20:23 +0900
category: CSS
---
# display 속성: block, inline, inline-block

CSS에서 `display` 속성은 HTML 요소가 페이지에 어떻게 보여질지를 결정합니다. 특히 `block`, `inline`, `inline-block`은 자주 사용되는 값으로, 각각의 특성을 이해하면 레이아웃을 제어하는 데 큰 도움이 됩니다.

## 1. block

`block` 요소는 **항상 새로운 줄에서 시작**하며, **가로 전체 너비를 차지**합니다. 대표적인 block 요소에는 `<div>`, `<p>`, `<h1>~<h6>`, `<section>`, `<article>` 등이 있습니다.

```html
<div style="display: block; background: lightblue;">Block 요소</div>
<span style="display: block; background: lightgreen;">Span도 block이 되면 줄바꿈이 생깁니다.</span>
```

특징:
- 줄바꿈이 자동으로 일어남
- `width`, `height`, `margin`, `padding` 모두 적용 가능

## 2. inline

`inline` 요소는 **줄 안에서 다른 요소들과 함께 배치**됩니다. 대표적인 inline 요소에는 `<span>`, `<a>`, `<strong>`, `<em>` 등이 있습니다.

```html
<span style="display: inline; background: yellow;">Inline 요소</span>
<span style="display: inline; background: orange;">옆에 붙습니다.</span>
```

특징:
- 줄바꿈이 일어나지 않음
- `width`, `height`는 적용되지 않음 (내용 크기에 따라 자동 조정)
- `margin`과 `padding`은 **수직 방향**으로는 거의 영향 없음

## 3. inline-block

`inline-block`은 `inline`처럼 줄 안에 배치되지만, `block`처럼 `width`, `height`, `margin`, `padding`을 **자유롭게 설정할 수 있는** 형태입니다. 레이아웃을 조정할 때 매우 유용합니다.

```html
<span style="display: inline-block; width: 100px; height: 50px; background: pink;">inline-block</span>
<span style="display: inline-block; width: 120px; height: 50px; background: lightcoral;">또 다른 박스</span>
```

특징:
- 줄 안에 배치되며, 옆에 다른 요소가 올 수 있음
- `width`, `height` 설정 가능
- 레이아웃 구성 시 `flex`를 쓰기 전에 많이 사용됨

## ✅ 비교 요약표

| 속성         | 줄바꿈 | width/height 사용 | 옆 요소와 나란히 |
|--------------|--------|------------------|------------------|
| `block`      | O      | O                | X                |
| `inline`     | X      | X                | O                |
| `inline-block`| X     | O                | O                |

## 💡 보너스: 기타 display 속성들

아래는 이 주제를 벗어나지만 함께 알아두면 좋은 `display` 속성의 다른 값들입니다.

- `none`: 요소를 화면에서 완전히 제거
- `flex`: 플렉스 박스 레이아웃 (정렬과 분배에 유리)
- `grid`: 그리드 레이아웃 (2차원 배치에 강력)
- `table`: 테이블처럼 동작
- `contents`: 부모는 사라지고 자식만 남음

### 예제 코드 

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>display 속성 예제</title>
<style>
  .none-example {
    display: none; /* 요소가 화면에서 완전히 사라짐 */
  }

  .flex-example {
    display: flex; /* 플렉스 박스 레이아웃 */
    gap: 10px;
    border: 1px solid #333;
    padding: 10px;
  }
  .flex-example > div {
    background: lightblue;
    padding: 10px;
    flex: 1;
    text-align: center;
  }

  .grid-example {
    display: grid; /* 그리드 레이아웃 */
    grid-template-columns: repeat(3, 1fr);
    gap: 10px;
    border: 1px solid #333;
    padding: 10px;
  }
  .grid-example > div {
    background: lightgreen;
    padding: 10px;
    text-align: center;
  }

  .table-example {
    display: table; /* 테이블처럼 동작 */
    border: 1px solid #333;
    width: 100%;
  }
  .table-row {
    display: table-row;
  }
  .table-cell {
    display: table-cell;
    border: 1px solid #999;
    padding: 10px;
  }

  .contents-example {
    display: contents; /* 부모는 사라지고 자식만 남음 */
  }
  .contents-example > div {
    background: pink;
    padding: 10px;
    border: 1px solid #f06;
    margin-bottom: 5px;
  }
</style>
</head>
<body>
  <h2>1. display: none;</h2>
  <div class="none-example">
    이 요소는 화면에서 완전히 사라집니다.
  </div>
  <p>위 문장은 보이지 않습니다.</p>

  <h2>2. display: flex;</h2>
  <div class="flex-example">
    <div>박스 1</div>
    <div>박스 2</div>
    <div>박스 3</div>
  </div>

  <h2>3. display: grid;</h2>
  <div class="grid-example">
    <div>셀 1</div>
    <div>셀 2</div>
    <div>셀 3</div>
    <div>셀 4</div>
    <div>셀 5</div>
    <div>셀 6</div>
  </div>

  <h2>4. display: table;</h2>
  <div class="table-example">
    <div class="table-row">
      <div class="table-cell">행 1, 셀 1</div>
      <div class="table-cell">행 1, 셀 2</div>
    </div>
    <div class="table-row">
      <div class="table-cell">행 2, 셀 1</div>
      <div class="table-cell">행 2, 셀 2</div>
    </div>
  </div>

  <h2>5. display: contents;</h2>
  <div class="contents-example">
    <div>자식 1</div>
    <div>자식 2</div>
  </div>
  <p>위 .contents-example 컨테이너는 렌더링 시 사라지고 자식 요소들만 남아 스타일이 적용됩니다.</p>
</body>
</html>
```

이렇게 `display` 속성을 이해하면, HTML 요소의 배치와 레이아웃에 대해 더 유연하게 대응할 수 있습니다.
