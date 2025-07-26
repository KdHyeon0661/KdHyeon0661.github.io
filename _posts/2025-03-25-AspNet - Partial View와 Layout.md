---
layout: post
title: AspNet - Partial View와 Layout
date: 2025-03-25 20:20:23 +0900
category: AspNet
---
# 🧩 Partial View와 Layout 적용 완전 정리 (ASP.NET Core MVC 기준)

---

## 📌 1. View의 구성 계층

ASP.NET Core MVC의 View는 다음과 같은 **계층 구조**를 이룹니다:

```
_Layout.cshtml  ← 전체 페이지 레이아웃
 └ View (.cshtml) ← 컨트롤러에서 반환되는 페이지
    └ Partial View ← 구성 요소 재사용 (헤더, 카드 등)
```

---

## 🟣 2. Layout (_Layout.cshtml)

### ✅ 역할

- 사이트 전체에 공통 적용되는 **마스터 페이지**
- HTML `head`, `nav`, `footer` 등 **공통 요소 유지**
- 본문은 `@RenderBody()`로 구성

### ✅ 위치

`Views/Shared/_Layout.cshtml` (일반적인 위치)

---

### ✅ 기본 구조 예시

```html
<!DOCTYPE html>
<html>
<head>
    <title>@ViewData["Title"] - MyApp</title>
    <link rel="stylesheet" href="~/css/site.css" />
</head>
<body>
    <header>
        <h1>MyApp</h1>
        <nav>...</nav>
    </header>

    <main role="main" class="container">
        @RenderBody()
    </main>

    <footer>
        <p>&copy; 2025 MyApp</p>
    </footer>
</body>
</html>
```

---

### ✅ View에서 Layout 지정

```csharp
@{
    Layout = "_Layout";
}
```

혹은 `_ViewStart.cshtml`을 통해 전체 View에 기본 Layout 지정:

```csharp
// Views/_ViewStart.cshtml
@{
    Layout = "_Layout";
}
```

---

## 🔹 3. Partial View

### ✅ 역할

- **재사용 가능한 View 조각**
- 여러 View에서 중복 없이 사용 가능
- 대표 예시: 사용자 카드, 댓글 블록, 리스트 항목 등

### ✅ 위치

- `/Views/Shared/` 또는
- `/Views/{Controller}/` 아래에 둬도 됨

### ✅ Partial View 파일 명명 규칙

- 일반적으로 `_` 접두어 사용  
  예: `_ProductCard.cshtml`, `_Comment.cshtml`

---

### ✅ Partial View 예시

#### 📄 `_ProductCard.cshtml`

```html
@model Product

<div class="card">
    <h3>@Model.Name</h3>
    <p>@Model.Description</p>
    <span>가격: @Model.Price.ToString("C")</span>
</div>
```

#### 📄 View에서 사용

```csharp
@model IEnumerable<Product>

@foreach (var product in Model)
{
    @Html.Partial("_ProductCard", product)
}
```

혹은 `await` 방식:

```csharp
@await Html.PartialAsync("_ProductCard", product)
```

---

## 🧪 4. Partial vs Layout 차이점

| 항목 | Layout | Partial View |
|------|--------|--------------|
| 목적 | 전체 뼈대 | 재사용 가능한 조각 |
| 위치 | Views/Shared/_Layout.cshtml | Views/Shared 또는 각 폴더 |
| 렌더링 위치 | `@RenderBody()` | `@Html.Partial()` |
| 사용 대상 | 전체 View | 여러 View 중 일부에서 반복 사용 |
| 데이터 전달 | ViewModel 또는 ViewData | 모델 전달 필요 (`@model`) |

---

## ✨ 5. 추가 기능: `RenderSection`

Layout에서 특정 View가 내용을 채워야 하는 **지정된 영역**을 만들 수 있어요.

### ✅ Layout.cshtml

```csharp
<body>
    @RenderBody()
    @RenderSection("Scripts", required: false)
</body>
```

### ✅ View.cshtml

```csharp
@section Scripts {
    <script>
        console.log("페이지 전용 스크립트");
    </script>
}
```

---

## 🧰 6. Component 구조 제안

| 기능 | 구현 방법 |
|------|-----------|
| 네비게이션 바 | Partial View: `_Navbar.cshtml` |
| 사용자 카드 | Partial View: `_UserCard.cshtml` |
| 기본 레이아웃 | Layout: `_Layout.cshtml` |
| 관리자 Layout | 다른 Layout: `_AdminLayout.cshtml` |
| 페이지별 스크립트 | `@section Scripts` |

---

## 🛠️ 7. 추천 실전 구조

```
Views/
 ├── Shared/
 │    ├── _Layout.cshtml
 │    ├── _Navbar.cshtml
 │    ├── _Footer.cshtml
 │    └── _ProductCard.cshtml
 ├── Home/
 │    ├── Index.cshtml
 │    └── About.cshtml
 └── Products/
      ├── Index.cshtml
      └── Details.cshtml
```

`_Layout.cshtml`에 `_Navbar.cshtml`, `_Footer.cshtml`을 `Partial`로 포함하고  
각 페이지에서 `@RenderBody()`를 통해 뷰 내용 삽입

---

## ✅ 요약

| 개념 | 설명 |
|------|------|
| **Layout** | 전체 페이지 공통 뼈대 |
| **Partial View** | 반복 가능한 View 조각 |
| **@RenderBody** | Layout 내 View 본문 삽입 위치 |
| **@RenderSection** | 특정 섹션만 뷰에서 삽입 |
| **@Html.Partial / PartialAsync** | Partial View 호출 방식 |

---

## 🔜 추천 다음 주제

- ✅ ViewComponent (Partial View의 확장판)
- ✅ Layout을 동적으로 변경하는 방법
- ✅ 다국어 Razor Layout 적용
- ✅ _ViewImports.cshtml의 역할
- ✅ Component 기반 설계 전략