---
layout: post
title: Asp - Query String & Route Parameter
date: 2025-02-10 19:20:23 +0900
category: AspNet
---
# 🌐 Razor Pages에서 Query String과 Route Parameter 다루기

ASP.NET Core Razor Pages에서는 **페이지 간 데이터 전달**을 위해 주로 두 가지 방식을 사용합니다:

1. 🔢 Route Parameter (경로 기반)
2. ❓ Query String (URL 뒤 ? 로 전달되는 파라미터)

각 방식의 사용법, 장단점, 예제 등을 자세히 살펴보겠습니다.

---

## 🛣️ 1. Route Parameter (경로 기반 파라미터)

### 📌 기본 개념

Route Parameter는 URL 경로 일부에 데이터를 포함하는 방식입니다.

예:  
```
/Product/Details/3
```

`3`이라는 숫자가 경로에 포함된 Route Parameter입니다.

---

### ✅ Razor Pages에서 사용 방법

#### ① `@page` 지시어에 경로 템플릿 작성

**Pages/Product/Details.cshtml**

```razor
@page "{id:int}"
@model ProductDetailsModel

<h2>상품 ID: @Model.ProductId</h2>
```

#### ② PageModel에서 파라미터 수신

```csharp
public class ProductDetailsModel : PageModel
{
    public int ProductId { get; set; }

    public void OnGet(int id)
    {
        ProductId = id;
    }
}
```

#### ✅ 요청 예시

```
GET /Product/Details/3
→ id = 3 전달됨
```

---

### 💡 지원하는 라우트 제약조건

| 제약조건 | 예시 | 설명 |
|----------|------|------|
| `{id:int}` | 정수만 허용 |
| `{name:alpha}` | 영문자만 허용 |
| `{slug:guid}` | GUID 형식만 허용 |
| `{name?}` | 선택적 파라미터 |

---

## ❓ 2. Query String (쿼리 문자열 파라미터)

### 📌 기본 개념

Query String은 `?` 이후에 key-value 형태로 전달되는 데이터입니다.

예:
```
/Search?keyword=notebook&page=2
```

---

### ✅ Razor Pages에서 사용 방법

#### ① PageModel에서 `[FromQuery]` 혹은 일반 파라미터로 수신

```csharp
public class SearchModel : PageModel
{
    public string Keyword { get; set; }
    public int Page { get; set; }

    public void OnGet(string keyword, int page = 1)
    {
        Keyword = keyword;
        Page = page;
    }
}
```

또는 더 명시적으로:

```csharp
public void OnGet([FromQuery] string keyword, [FromQuery] int page = 1)
```

#### ✅ 요청 예시

```
GET /Search?keyword=apple&page=2
→ keyword = "apple", page = 2
```

---

## 🔁 Route vs QueryString 비교

| 항목 | Route Parameter | Query String |
|------|------------------|---------------|
| URL 예시 | `/Product/3` | `/Product?id=3` |
| 선언 위치 | `@page "{id}"` | 없음 (`OnGet(string id)` 등) |
| 주 용도 | 리소스 식별 (RESTful) | 검색, 필터, 옵션 등 |
| 검색엔진 친화성 | ✅ 좋음 | ❌ 상대적 불리 |
| 가독성 | ✅ 깔끔함 | 🔸 데이터 많을수록 지저분 |

---

## 🧭 함께 사용하는 예시

**Search.cshtml**

```razor
@page "{category?}"
@model SearchModel

<h2>@Model.Category 카테고리 검색</h2>
<p>검색어: @Model.Keyword, 페이지: @Model.Page</p>
```

**Search.cshtml.cs**

```csharp
public class SearchModel : PageModel
{
    public string? Category { get; set; }
    public string? Keyword { get; set; }
    public int Page { get; set; } = 1;

    public void OnGet(string? category, string? keyword, int page = 1)
    {
        Category = category;
        Keyword = keyword;
        Page = page;
    }
}
```

**요청 예시**:
```
/Search/electronics?keyword=tv&page=2
→ category = "electronics", keyword = "tv", page = 2
```

---

## 🔧 파라미터 전송 링크 만들기

### ✅ 쿼리스트링 링크

```razor
<a asp-page="/Search" asp-route-keyword="camera" asp-route-page="2">카메라 검색</a>
```

### ✅ 라우트 파라미터 링크

```razor
<a asp-page="/Product/Details" asp-route-id="10">상품 상세보기</a>
```

- `asp-route-` 접두사는 자동으로 쿼리나 라우트 값 삽입을 도와줌

---

## ✅ 마무리 요약

| 구분 | Route Parameter | Query String |
|------|------------------|---------------|
| 선언 방식 | `@page "{id}"` | 없음 (`OnGet(...)`) |
| 전송 방식 | `/Product/3` | `/Product?id=3` |
| 의미 | 리소스 식별 | 옵션, 필터링 등 |
| 코드에서 처리 | `OnGet(int id)` | `OnGet(string keyword)` |
| 링크 만들기 | `asp-route-id="3"` | `asp-route-keyword="notebook"` |

---

## 🔜 다음 추천 주제

- ✅ **라우트 제약조건 상세 정리 (`{id:int}`, `{slug:alpha}` 등)**
- ✅ **`asp-page`, `asp-route` Tag Helper 정리**
- ✅ **폼 데이터와 QueryString 조합 처리**
- ✅ **Redirect 시 쿼리 스트링 유지하는 방법**