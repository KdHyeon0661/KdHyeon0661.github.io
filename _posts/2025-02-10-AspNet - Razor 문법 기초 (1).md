---
layout: post
title: AspNet - Razor 문법 기초 (1)
date: 2025-02-10 19:20:23 +0900
category: AspNet
---
# 🖋️ Razor 문법 기초 완전 정복 (ASP.NET Core 기준)

**Razor 문법**은 C# 코드를 HTML 안에 자연스럽게 삽입하여 **동적인 웹 페이지**를 작성할 수 있게 해줍니다. ASP.NET Core의 Razor Pages, MVC View 등에서 모두 사용됩니다.

이 글에서는 Razor 문법 중 **기초적인 것들만 골라** 자세히 설명합니다.

---

## ⚡️ Razor 문법의 핵심 기호: `@`

Razor 문법은 `@` 기호를 사용하여 **HTML 안에 C# 코드를 삽입**합니다.

### ✅ 기본 출력 예시

```cshtml
@DateTime.Now
```

현재 시간을 HTML 페이지에 출력합니다.

```cshtml
@("안녕하세요! 오늘은 " + DateTime.Today.ToString("yyyy-MM-dd") + "입니다.")
```

---

## 🧱 1. 코드 블록: `@{ ... }`

`@{}` 안에서는 **여러 줄의 C# 코드**를 작성할 수 있습니다. 변수 선언, 조건문 등도 가능해요.

```cshtml
@{
    var name = "홍길동";
    var age = 30;
}
<p>@name (@age세)</p>
```

---

## 📄 2. @model: 뷰에서 사용하는 데이터 정의

Razor 파일에서 사용할 **C# 모델 클래스**를 지정합니다. View 또는 Razor Page에서 가장 위에 선언합니다.

```cshtml
@model MyApp.Models.Product
```

그리고 아래에서 해당 모델의 속성들을 사용할 수 있어요:

```cshtml
<h2>@Model.Name</h2>
<p>가격: @Model.Price 원</p>
```

- `@Model`은 지정한 모델 객체를 의미함
- 속성 접근은 `@Model.속성명`

---

## 🔁 3. 반복문: `@foreach`

데이터 목록을 화면에 출력할 때 사용합니다.

```cshtml
@model List<string>

<ul>
@foreach (var item in Model)
{
    <li>@item</li>
}
</ul>
```

예:
```csharp
new List<string> { "사과", "바나나", "포도" }
```

출력:
```html
<li>사과</li>
<li>바나나</li>
<li>포도</li>
```

---

## 🧪 4. 조건문: `@if`, `@else`

조건에 따라 내용을 다르게 출력할 수 있습니다.

```cshtml
@{
    var score = 85;
}

@if (score >= 90)
{
    <p>우수</p>
}
else if (score >= 70)
{
    <p>보통</p>
}
else
{
    <p>미달</p>
}
```

---

## 🎯 5. 표현식 출력: `@( ... )`

복잡한 식이나 계산 결과를 출력할 때 사용합니다.

```cshtml
<p>총합: @(100 + 200)원</p>
<p>결과: @(Model.Score >= 60 ? "합격" : "불합격")</p>
```

---

## 📜 6. 주석

### ✅ Razor 전용 주석 (`브라우저에 보이지 않음`)

```razor
@* 이건 Razor 주석입니다. HTML에는 표시되지 않음 *@
```

### ✅ HTML 주석 (`브라우저 소스 보기에서 보임`)

```html
<!-- 이건 HTML 주석입니다 -->
```

---

## 🧩 7. HTML 안에서 C# 변수 사용

```cshtml
@{
    var title = "Razor 문법 배우기";
}
<h1>@title</h1>
```

또는 더 간단히:

```cshtml
<h1>@("안녕하세요 " + title + " 페이지입니다.")</h1>
```

---

## 📝 요약 표: Razor 문법 기초

| 문법 | 설명 | 예시 |
|------|------|------|
| `@` | C# 표현식 삽입 | `@DateTime.Now` |
| `@{}` | 코드 블록 | `@{ var x = 1; }` |
| `@model` | 모델 클래스 지정 | `@model Product` |
| `@Model` | 모델 객체 접근 | `@Model.Name` |
| `@foreach` | 반복 출력 | `@foreach (var x in ...)` |
| `@if`, `@else` | 조건문 | `@if (조건) { ... }` |
| `@("문자열" + 변수)` | 문자열 출력 | `@("안녕 " + name)` |
| `@* *@` | Razor 주석 | `@* 주석 *@` |

---

# ✅ 마무리

Razor 문법은 복잡하지 않으며, HTML과 C#을 섞어서 **가독성 좋고 생산성 높은 페이지**를 만들 수 있게 해줍니다.