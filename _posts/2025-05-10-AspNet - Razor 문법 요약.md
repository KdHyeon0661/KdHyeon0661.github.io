---
layout: post
title: AspNet - Razor 문법 요약
date: 2025-05-10 21:20:23 +0900
category: AspNet
---
# 🧩 Razor 문법 요약 (ASP.NET Core)

---

## ✅ Razor란?

Razor는 ASP.NET Core에서 사용되는 **서버 사이드 템플릿 엔진**입니다.

- HTML 안에 **C# 코드를 삽입**할 수 있음
- `.cshtml` 확장자 파일에서 사용됨
- MVC와 Razor Pages 모두에서 사용 가능

---

## 📝 기본 문법

### 🔹 `@` 기호

Razor에서 C# 코드를 시작할 때 `@`를 사용해요.

```razor
<p>Hello, @Model.UserName!</p>
```

```razor
@{
    var now = DateTime.Now;
}
<p>현재 시간: @now</p>
```

---

## 🔁 흐름 제어문

### 🔹 if / else

```razor
@if (Model.IsAdmin)
{
    <p>관리자입니다.</p>
}
else
{
    <p>일반 사용자입니다.</p>
}
```

### 🔹 foreach

```razor
<ul>
@foreach (var item in Model.Products)
{
    <li>@item.Name - @item.Price 원</li>
}
</ul>
```

### 🔹 for

```razor
@for (int i = 0; i < 3; i++)
{
    <p>Index: @i</p>
}
```

---

## 📦 변수, 메서드

```razor
@{
    var title = "Razor 예제";
    int Sum(int a, int b) => a + b;
}
<h1>@title</h1>
<p>2 + 3 = @Sum(2, 3)</p>
```

---

## 📄 HTML 인코딩

Razor는 **기본적으로 HTML 인코딩**을 합니다.

```razor
@("<b>Bold</b>")     → 출력: &lt;b&gt;Bold&lt;/b&gt;
@Html.Raw("<b>Bold</b>") → 출력: <b>Bold</b>
```

---

## 📬 Model 바인딩

```razor
@model MyApp.Models.User

<h1>@Model.UserName 님 환영합니다!</h1>
```

- `@model`은 해당 View의 **모델 타입을 지정**
- `Model`은 클래스 인스턴스를 참조함

---

## 📋 폼 바인딩과 입력 처리

### 🔹 입력 바인딩 (Tag Helper)

```razor
<form method="post">
    <input asp-for="Email" />
    <span asp-validation-for="Email"></span>
</form>
```

### 🔹 처리 메서드 (Razor Pages)

```csharp
public class IndexModel : PageModel
{
    [BindProperty]
    public string Email { get; set; }

    public void OnPost()
    {
        // Email 처리
    }
}
```

---

## 🧾 유효성 검사 메시지

```razor
<span asp-validation-for="Email" class="text-danger"></span>
```

→ Data Annotation과 연동되어 자동으로 메시지를 출력

---

## 🌐 Partial View / Layout

### 🔹 Layout 지정

```razor
@{
    Layout = "_Layout";
}
```

### 🔹 Partial 포함

```razor
<partial name="_LoginPartial" />
```

---

## 🔗 링크/페이지 이동

### 🔹 MVC 스타일

```razor
<a asp-controller="Home" asp-action="About">소개</a>
```

### 🔹 Razor Pages 스타일

```razor
<a asp-page="/Contact">문의하기</a>
```

---

## 🔐 조건부 렌더링

### 🔹 `@* 주석 *@`

```razor
@* 이건 Razor 주석입니다 *@
```

---

## 🔤 문자열 출력

```razor
@Model.Title         → HTML 인코딩됨
@Html.Raw(Model.Body) → HTML로 렌더링됨
```

---

## 💡 팁: 복잡한 표현은 괄호로 감싸기

```razor
<p>합계: @(Model.Price * Model.Quantity)</p>
```

---

## 🎯 Razor 문법 예제 종합

```razor
@model MyApp.Models.Product

@{
    var isOnSale = Model.Price < 10000;
}

<h2>@Model.Name</h2>

@if (isOnSale)
{
    <p class="text-success">할인 중!</p>
}

<p>가격: @Model.Price 원</p>

<ul>
@foreach (var tag in Model.Tags)
{
    <li>@tag</li>
}
</ul>
```

---

## ✅ 요약

| 구문 | 역할 |
|------|------|
| `@` | C# 코드 시작 |
| `@{ }` | 코드 블록 |
| `@if`, `@for`, `@foreach` | 흐름 제어 |
| `@model` | 모델 타입 지정 |
| `@Html.Raw` | HTML 직접 출력 |
| `asp-for`, `asp-page` | Tag Helper |
| `@* *@` | Razor 주석 |