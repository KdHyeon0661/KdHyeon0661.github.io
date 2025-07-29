---
layout: post
title: AspNet - Tag Helper
date: 2025-05-10 19:20:23 +0900
category: AspNet
---
# 🏷 자주 쓰는 Tag Helper 정리 (ASP.NET Core)

---

## ✅ Tag Helper란?

**HTML 태그와 C# 코드의 연결고리**로, Razor에서 더 선언적이고 직관적으로 C# 로직을 작성하게 해주는 기능이에요.

- HTML처럼 보이지만 C# 코드로 동작함
- HTML과 IntelliSense를 함께 사용 가능
- `asp-` 접두사를 통해 서버 사이드 속성을 바인딩

---

## 🧱 1. `asp-for`

📌 **Model 속성과 양방향 바인딩**  
`name`, `id`, `value` 등을 자동으로 생성합니다.

```razor
<input asp-for="UserName" class="form-control" />
<span asp-validation-for="UserName" class="text-danger"></span>
```

- `asp-for="UserName"` → `<input name="UserName" value="@Model.UserName">`
- 폼 바인딩, 유효성 검사 시 필수!

---

## 📋 2. `asp-action`, `asp-controller`

📌 **링크나 폼의 이동 경로 지정**

```razor
<a asp-controller="Home" asp-action="About">About Page</a>

<form asp-action="Login" method="post">
  <input asp-for="Email" />
</form>
```

- `asp-action` : 이동할 액션 메서드 이름
- `asp-controller` : 대상 컨트롤러 이름
- MVC에서도 Razor Pages에서도 사용 가능

---

## 🌍 3. `asp-page`, `asp-page-handler`

📌 **Razor Pages에서 사용되는 경로 기반 속성**

```razor
<a asp-page="/Contact">Contact</a>
<form asp-page-handler="Submit" method="post">
```

- `asp-page="/경로"`: Razor Page 지정
- `asp-page-handler="Submit"`: `OnPostSubmit` 핸들러 호출

---

## 🧾 4. `asp-route`, `asp-route-{param}`

📌 **라우트 파라미터 전달**

```razor
<a asp-page="/Product" asp-route-id="123">View Product</a>
```

- 결과: `/Product?id=123`

```razor
<a asp-controller="Product" asp-action="Detail" asp-route-slug="phone">...</a>
```

- 결과: `/Product/Detail?slug=phone`

---

## 🔁 5. `asp-items` (Select 목록 바인딩)

📌 **드롭다운 구성 시 List 바인딩**

```razor
<select asp-for="CategoryId" asp-items="Model.Categories"></select>
```

> `Model.Categories`는 `List<SelectListItem>` 형태

```csharp
Model.Categories = new List<SelectListItem> {
  new("Phone", "1"),
  new("Tablet", "2")
};
```

---

## 🛡 6. `asp-validation-for`, `asp-validation-summary`

📌 **서버/클라이언트 유효성 검사 메시지 출력**

```razor
<span asp-validation-for="UserName" class="text-danger"></span>

<partial name="_ValidationScriptsPartial" />
```

- `asp-validation-for`: 개별 필드 메시지
- `asp-validation-summary="All"`: 전체 오류 출력

> 클라이언트 측 validation을 위해 `_ValidationScriptsPartial.cshtml` 포함 필수

---

## 🔐 7. `form` + `asp-antiforgery`

📌 CSRF 방지용 토큰 자동 삽입

```razor
<form asp-antiforgery="true" method="post">
  <input asp-for="Email" />
</form>
```

> 기본값이 `true`이므로 생략 가능  
> `@Html.AntiForgeryToken()`과 동일한 역할 수행

---

## 📷 8. `input type="file"` + `asp-for`

📌 파일 업로드 시에도 `asp-for` 사용 가능

```razor
<input type="file" asp-for="ProfileImage" />
```

- `IFormFile` 속성과 바인딩됨

---

## 📄 9. `partial`, `view-component`

📌 UI 컴포넌트 분할

```razor
<partial name="_LoginPartial" />
```

> Razor Partial View를 포함

---

## 🌐 10. `environment` (환경별 분기)

📌 `Development`, `Production` 등 환경 구분

```razor
<environment include="Development">
    <script src="lib/jquery.js"></script>
</environment>

<environment exclude="Development">
    <script src="lib/jquery.min.js"></script>
</environment>
```

---

## 🎯 11. `cache` (Razor Partial 캐싱)

📌 부분 UI를 서버에서 일정 시간 캐싱

```razor
<cache expires-after="00:01:00">
    <partial name="_ExpensiveComponent" />
</cache>
```

- `expires-after`: 지정된 시간 동안 HTML 결과를 캐싱
- Output Caching (.NET 8+)과 함께 사용 가능

---

## 🧪 12. `input`, `label`, `textarea`의 Tag Helper

- `<input asp-for="Title" />`
- `<label asp-for="Title" />`
- `<textarea asp-for="Description"></textarea>`

→ 자동으로 `id`, `name`, `value`, `for` 속성까지 바인딩됨  
→ HTML/Form 구조에서 필수적인 구성 요소

---

## 📦 13. 종합 예시 (폼)

```razor
<form asp-page-handler="Submit" method="post" enctype="multipart/form-data">
    <label asp-for="UserName"></label>
    <input asp-for="UserName" class="form-control" />

    <label asp-for="ProfileImage"></label>
    <input type="file" asp-for="ProfileImage" />

    <select asp-for="RoleId" asp-items="Model.RoleList"></select>

    <button type="submit">Register</button>
</form>
```

---

## ✅ 요약

| Tag Helper | 설명 |
|------------|------|
| `asp-for` | 모델 바인딩, 입력 필드 연결 |
| `asp-action`, `asp-controller` | MVC 라우팅 |
| `asp-page`, `asp-page-handler` | Razor Pages 라우팅 |
| `asp-items` | Select 드롭다운 구성 |
| `asp-validation-for` | 필드별 유효성 메시지 |
| `asp-route`, `asp-route-id` | 쿼리 파라미터 |
| `asp-antiforgery` | CSRF 보호 |
| `partial`, `cache`, `environment` | UI 분리, 환경 제어 |