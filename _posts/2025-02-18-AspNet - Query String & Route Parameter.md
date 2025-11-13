---
layout: post
title: AspNet - Query String & Route Parameter
date: 2025-02-18 19:20:23 +0900
category: AspNet
---
# Razor Pages에서 Query String과 Route Parameter 다루기

## 0. 페이지 간 데이터 전달의 두 축

- **Route Parameter**: 경로 일부로 리소스를 식별. RESTful, SEO 친화, 캐싱 용이.
- **Query String**: `?key=value` 추가 파라미터. 검색/필터/정렬/페이지네이션에 적합.

요청 예:
```
GET /products/42              → Route(ID=42)
GET /search?keyword=tv&page=2 → Query(keyword=tv, page=2)
```

실전에서는 **Route = 자원 식별자**, **Query = 뷰 상태(필터/정렬/페이지)**로 역할을 분리하면 URL이 명확해집니다.

---

## 1. Route Parameter — Razor Pages의 파일 기반 라우팅

### 1.1 기본 사용

**Pages/Product/Details.cshtml**
```cshtml
@page "{id:int}"
@model ProductDetailsModel
<h2>상품 ID: @Model.ProductId</h2>
```

**Pages/Product/Details.cshtml.cs**
```csharp
using Microsoft.AspNetCore.Mvc.RazorPages;

public class ProductDetailsModel : PageModel
{
    public int ProductId { get; private set; }

    public void OnGet(int id) => ProductId = id;
}
```

요청:
```
GET /Product/Details/3 → id=3
```

### 1.2 제약(Constraints)과 옵션

| 문법                         | 설명                                      | 예시                                       |
|-----------------------------|-------------------------------------------|--------------------------------------------|
| `{id:int}`                  | 정수만 허용                               | `@page "{id:int}"`                         |
| `{slug:guid}`               | GUID 형식                                 | `@page "{slug:guid}"`                      |
| `{name:alpha}`              | 알파벳만                                  | `@page "{name:alpha}"`                     |
| `{len:length(3,10)}`        | 길이 제약                                 | `@page "{code:length(5)}"`                 |
| `{v:regex(^[a-z0-9-]+$)}`   | 정규식 매칭                               | `@page "{slug:regex(^[a-z0-9-]+$)}"`       |
| `{n:int:min(1):max(9999)}`  | 값 범위                                   | `@page "{id:int:min(1)}"`                  |
| `{id?}`                     | 선택적 파라미터                           | `@page "{id?}"`                            |
| `{*path}`                   | 캐치올(하위 경로 전체 캡처)               | `@page "{*path}"`                          |

> 규칙은 엄격할수록 **잘못된 요청을 라우트 단계에서 거부**할 수 있어 보안/성능상 유리합니다.

### 1.3 다중 라우트(별칭)

```cshtml
@page "/Notice"
@page "/공지사항"
@model NoticeModel
```
두 URL 모두 같은 페이지를 매핑.

또는 Program.cs에서 **Conventions**:
```csharp
builder.Services.AddRazorPages(o =>
{
    o.Conventions.AddPageRoute("/Product/Details", "p/{id:int}");
});
```

---

## 2. Query String — 검색/필터/정렬/페이지의 핵심

### 2.1 기본 바인딩

**Pages/Search.cshtml.cs**
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

public class SearchModel : PageModel
{
    public string? Keyword { get; private set; }
    public int Page { get; private set; } = 1;

    // 단순 바인딩: 파라미터 이름과 쿼리 키가 일치하면 자동 바인딩
    public void OnGet(string? keyword, int page = 1)
    {
        Keyword = keyword;
        Page = page;
    }

    // 명시적으로 FromQuery 사용 가능(선택 사항)
    public void OnGetExplicit([FromQuery] string keyword, [FromQuery] int page = 1) { ... }
}
```

요청:
```
GET /Search?keyword=apple&page=2 → keyword=apple, page=2
```

### 2.2 `[BindProperty(SupportsGet = true)]`와 비교

**SupportsGet = true**를 쓰면 쿼리(또는 라우트) 값을 **속성**에 바로 바인딩:

```csharp
public class GridModel : PageModel
{
    [BindProperty(SupportsGet = true)] public string? Sort { get; set; }
    [BindProperty(SupportsGet = true)] public int Page { get; set; } = 1;
    public void OnGet() { /* Sort, Page 값 사용 */ }
}
```

- **장점**: 속성 접근으로 뷰에서 간결(`@Model.Sort`)
- **주의**: GET 바인딩은 URL 값을 그대로 노출하므로 **화이트리스트 검증**(예: Sort 허용 목록)이 필요

---

## 3. Route vs Query — 역할과 선택 기준

| 항목           | Route Parameter                             | Query String                                  |
|----------------|----------------------------------------------|-----------------------------------------------|
| URL 예         | `/products/3`                                | `/products?id=3`                              |
| 주 용도        | **리소스 식별자**(ID, slug)                  | **필터/옵션/상태**(검색어, 정렬, 페이지, 탭) |
| SEO/가독성     | 우수                                         | 상대적 열세(매개변수 많으면 난잡)             |
| 캐싱/즐겨찾기  | 유리                                         | 비교적 무관                                  |
| 안정성         | 스키마 고정이 바람직                         | 매개변수 추가/변경이 유연                    |

**실전 규칙**:
- **필수 식별자**는 Route.
- **사용자 조작 상태**(검색/정렬/페이지/필터)는 Query.
- 혼합이 필요한 경우 **Route + Query**를 조합(예: `/products/phones?sort=price&page=3`).

---

## 4. 함께 쓰기 — 카테고리(Route) + 검색/페이지(Query)

**Pages/Search/Index.cshtml**
```cshtml
@page "{category?}"
@model SearchModel

<h2>@Model.Category 검색</h2>
<p>키워드: @Model.Keyword, 페이지: @Model.Page</p>

<nav>
  <a asp-page="./Index"
     asp-route-category="@Model.Category"
     asp-route-keyword="@Model.Keyword"
     asp-route-page="@(Model.Page - 1)">이전</a>

  <a asp-page="./Index"
     asp-route-category="@Model.Category"
     asp-route-keyword="@Model.Keyword"
     asp-route-page="@(Model.Page + 1)">다음</a>
</nav>
```

**Pages/Search/Index.cshtml.cs**
```csharp
public class SearchModel : PageModel
{
    public string? Category { get; private set; }
    public string? Keyword { get; private set; }
    public int Page { get; private set; } = 1;

    public void OnGet(string? category, string? keyword, int page = 1)
    {
        Category = category;
        Keyword  = keyword;
        Page     = Math.Max(1, page);
    }
}
```

요청:
```
/Search/electronics?keyword=tv&page=2
```

---

## 5. 링크 생성 — `asp-page`/`asp-route-*` Tag Helper

### 5.1 쿼리스트링 링크
```cshtml
<a asp-page="/Search" asp-route-keyword="camera" asp-route-page="2">
  카메라 검색
</a>
```

### 5.2 라우트 파라미터 링크
```cshtml
<a asp-page="/Product/Details" asp-route-id="10">상품 상세보기</a>
```

### 5.3 현재 쿼리를 유지하며 일부만 변경
```cshtml
@{
    // 현재 Request.Query를 기반으로 RouteValueDictionary 구성(헬퍼 메서드로 추출 권장)
    var route = ViewContext.HttpContext.Request.Query
                  .ToDictionary(kv => kv.Key, kv => (string)kv.Value);
    route["page"] = "2";
}

<a asp-page="./Index"
   asp-all-route-data="route">2페이지</a>
```

> `asp-all-route-data`는 다수의 쿼리/라우트 값을 한 번에 바인딩할 때 유용합니다.

---

## 6. Redirect 시 쿼리스트링 유지 — PRG 패턴

**PRG(Post-Redirect-Get)**: 폼 POST 후 **Redirect**로 새로고침 중복을 방지하고 URL을 공유 가능한 GET으로 통일.

```csharp
public IActionResult OnPost()
{
    // 서버 검증/처리...
    return RedirectToPage("./Index", new { keyword = Form.Keyword, page = 1 });
}
```

또는 현재 쿼리를 거의 유지하고 일부만 바꿀 때:
```csharp
public IActionResult OnPostSetPage(int page)
{
    var route = Request.Query.ToDictionary(kv => kv.Key, kv => kv.Value.ToString());
    route["page"] = page.ToString();
    return RedirectToPage("./Index", route);
}
```

> **주의**: URL 길이 제한(서버/브라우저)이 있으므로 지나치게 긴 필터 상태는 **POST + TempData/세션** 또는 **서버 상태 저장(토큰화)**를 고려.

---

## 7. 페이징/정렬/필터 — 실전 패턴

### 7.1 모델/기본값/화이트리스트 검증
```csharp
public class ListQuery
{
    public int Page { get; set; } = 1;
    public int Size { get; set; } = 20;
    public string? Sort { get; set; } // "name", "price", "-price" 등
    public string? Keyword { get; set; }
}

public class ProductsModel : PageModel
{
    [BindProperty(SupportsGet = true)] public ListQuery Q { get; set; } = new();

    private static readonly HashSet<string> AllowedSort =
        new(StringComparer.OrdinalIgnoreCase) { "name", "price", "-price" };

    public void OnGet()
    {
        Q.Page = Math.Max(1, Q.Page);
        Q.Size = Math.Clamp(Q.Size, 1, 100);
        if (!string.IsNullOrEmpty(Q.Sort) && !AllowedSort.Contains(Q.Sort))
            Q.Sort = "name"; // 기본 정렬로 강제
    }
}
```

### 7.2 링크(정렬 토글)
```cshtml
<a asp-page="./Index" asp-route-sort="name"   asp-route-page="@Model.Q.Page">이름↑</a>
<a asp-page="./Index" asp-route-sort="-name"  asp-route-page="@Model.Q.Page">이름↓</a>
<a asp-page="./Index" asp-route-sort="price"  asp-route-page="@Model.Q.Page">가격↑</a>
<a asp-page="./Index" asp-route-sort="-price" asp-route-page="@Model.Q.Page">가격↓</a>
```

> **화이트리스트**를 통해 SQL 인젝션/열거형 오류를 예방하세요(정렬 필드/방향을 엄격히 제한).

---

## 8. 컬렉션/복합 타입 바인딩 — 체크박스·다중 선택

### 8.1 다중 체크박스 → `int[]`
```cshtml
@page
@model FilterModel

<form method="get">
  <label><input type="checkbox" name="brands" value="1"
         @(Model.Brands.Contains(1) ? "checked" : "") /> A사</label>
  <label><input type="checkbox" name="brands" value="2"
         @(Model.Brands.Contains(2) ? "checked" : "") /> B사</label>
  <button type="submit">적용</button>
</form>
```

```csharp
public class FilterModel : PageModel
{
    [BindProperty(SupportsGet = true)]
    public int[] Brands { get; set; } = Array.Empty<int>();

    public void OnGet() { /* Brands 이용 */ }
}
```

요청:
```
/?brands=1&brands=2
```

### 8.2 복합 타입(`FilterDto`) — 키 접두사로 하위 매핑
```cshtml
<input name="filter.minPrice" />
<input name="filter.maxPrice" />
```
```csharp
public class FilterDto { public int? MinPrice { get; set; } public int? MaxPrice { get; set; } }

public void OnGet([FromQuery] FilterDto filter) { /* ... */ }
```

---

## 9. 문화권/인코딩/형식 — 날짜·숫자 파싱 이슈

- 쿼리/라우트 값 파싱은 **현재 Culture**에 의존.
  예: `1,234` vs `1.234` (천 단위/소수점).
- 서버/클라이언트 문화권이 다르면 **형식 불일치** 발생 가능 →
  **표준 포맷**(ISO-8601 날짜 등)을 권장: `yyyy-MM-dd`, `O`(Round-trip).
- 라우트 템플릿에서 날짜는 문자열로 받고 내부에서 `DateTime.TryParseExact`로 안전 파싱.

예:
```csharp
public void OnGet(string date)
{
    if (DateTime.TryParseExact(date, "yyyy-MM-dd", CultureInfo.InvariantCulture,
                               DateTimeStyles.None, out var d))
    {
        // 사용
    }
}
```

---

## 10. 보안 — 신뢰 불가 입력과 화이트리스트

- **모든 URL 파라미터는 신뢰하지 않는다**: 정렬/필터 이름, 페이지/사이즈는 범위 제한과 허용 목록 필요.
- **경로 순회 방지**: 파일 경로/슬러그를 받아 파일 접근 시 `Path.GetFileName` 등으로 정규화.
- **인증/권한**: 라우트만으로 접근을 제어하지 말고 `[Authorize]`/폴더 정책을 결합.
- **공격적 쿼리 길이**: 서버/역프록시의 최대 URL 길이 제한 설정(DoS 방어).

---

## 11. 고급: `Url.Page`/`LinkGenerator`로 코드에서 링크 생성

PageModel/서비스:
```csharp
public class LinksModel : PageModel
{
    private readonly LinkGenerator _link;
    public LinksModel(LinkGenerator link) => _link = link;

    public string? NextUrl { get; private set; }

    public void OnGet(int page = 1)
    {
        NextUrl = _link.GetPathByPage(HttpContext,
                     page: "/Search/Index",
                     values: new { page = page + 1, keyword = Request.Query["keyword"] });
    }
}
```

> 하드코딩 URL 대신 **라우트 기반 링크 생성**을 쓰면 리팩터링에 강합니다.

---

## 12. 유틸: 현재 쿼리 유지/병합 헬퍼

**확장 메서드**로 자주 쓰는 병합 로직을 캡슐화:
```csharp
public static class QueryExtensions
{
    public static RouteValueDictionary Merge(this IQueryCollection query, object overrides)
    {
        var dict = query.ToDictionary(kv => kv.Key, kv => (string)kv.Value);
        foreach (var prop in overrides.GetType().GetProperties())
        {
            dict[prop.Name] = prop.GetValue(overrides)?.ToString() ?? "";
        }
        return new RouteValueDictionary(dict);
    }
}
```

사용:
```csharp
var merged = Request.Query.Merge(new { page = 2 });
return RedirectToPage("./Index", merged);
```

---

## 13. 실전 예제 — 제품 목록: 카테고리(Route) + 필터(Query) + PRG

**Pages/Products/Index.cshtml**
```cshtml
@page "{category?}"
@model ProductsIndexModel
@{
    var q = Model.Q;
}
<h1>@(Model.Category ?? "전체")</h1>

<form method="get">
  <input name="keyword" value="@q.Keyword" placeholder="검색어" />
  <select name="sort">
    <option value="">정렬 없음</option>
    <option value="name"   @(q.Sort=="name"   ? "selected" : "")>이름↑</option>
    <option value="-name"  @(q.Sort=="-name"  ? "selected" : "")>이름↓</option>
    <option value="price"  @(q.Sort=="price"  ? "selected" : "")>가격↑</option>
    <option value="-price" @(q.Sort=="-price" ? "selected" : "")>가격↓</option>
  </select>
  <button type="submit">검색</button>
</form>

<table>
  <thead><tr><th>이름</th><th>가격</th></tr></thead>
  <tbody>
  @foreach (var p in Model.Items)
  {
      <tr>
        <td>@p.Name</td>
        <td>@p.Price</td>
      </tr>
  }
  </tbody>
</table>

<nav>
  <a asp-page="./Index"
     asp-route-category="@Model.Category"
     asp-route-keyword="@q.Keyword"
     asp-route-sort="@q.Sort"
     asp-route-page="@(q.Page-1)">이전</a>

  <a asp-page="./Index"
     asp-route-category="@Model.Category"
     asp-route-keyword="@q.Keyword"
     asp-route-sort="@q.Sort"
     asp-route-page="@(q.Page+1)">다음</a>
</nav>
```

**Pages/Products/Index.cshtml.cs**
```csharp
using Microsoft.AspNetCore.Mvc.RazorPages;

public record Product(int Id, string Name, int Price);

public class ListQuery
{
    public string? Keyword { get; set; }
    public string? Sort { get; set; }
    public int Page { get; set; } = 1;
    public int Size { get; set; } = 10;
}

public class ProductsIndexModel : PageModel
{
    public string? Category { get; private set; }
    public ListQuery Q { get; private set; } = new();
    public IReadOnlyList<Product> Items { get; private set; } = Array.Empty<Product>();

    private static readonly HashSet<string> AllowedSort =
        new(StringComparer.OrdinalIgnoreCase) { "name", "-name", "price", "-price" };

    public void OnGet(string? category, string? keyword, string? sort, int page = 1, int size = 10)
    {
        Category = category;
        Q = new ListQuery
        {
            Keyword = keyword,
            Sort = AllowedSort.Contains(sort ?? "") ? sort : null,
            Page = Math.Max(1, page),
            Size = Math.Clamp(size, 1, 100)
        };

        // 샘플 데이터
        var all = new List<Product> {
            new(1, "Alpha", 100), new(2,"Beta", 200), new(3,"Gamma", 150)
        };

        if (!string.IsNullOrWhiteSpace(Q.Keyword))
            all = all.Where(p => p.Name.Contains(Q.Keyword, StringComparison.OrdinalIgnoreCase)).ToList();

        // 카테고리별 필터는 Category 값으로 추가 필터링 (예시)
        if (!string.IsNullOrEmpty(Category))
            all = all.Where(p => p.Name.StartsWith(Category, StringComparison.OrdinalIgnoreCase)).ToList();

        all = Q.Sort switch
        {
            "name"   => all.OrderBy(p => p.Name).ToList(),
            "-name"  => all.OrderByDescending(p => p.Name).ToList(),
            "price"  => all.OrderBy(p => p.Price).ToList(),
            "-price" => all.OrderByDescending(p => p.Price).ToList(),
            _        => all
        };

        Items = all.Skip((Q.Page - 1) * Q.Size).Take(Q.Size).ToList();
    }
}
```

---

## 14. 에지 케이스/함정 정리

1) **중복 키**: `/search?tag=a&tag=b` → `string[]` 또는 `IEnumerable<string>`로 바인딩
2) **잘못된 형식**: `?page=abc`는 int 변환 실패 → 0/기본값. 필요 시 `TryParse`로 명시적 처리
3) **공백/인코딩**: `keyword=4k tv` → `%20` 등 URL 인코딩. `asp-route-*`는 자동 인코딩 처리
4) **긴 URL**: 필터가 많아질수록 URL이 길어짐 → 서버/프록시 최대 길이 고려
5) **대소문자/정규화**: 슬러그는 소문자/`-`로 정규화하여 중복 방지
6) **`SupportsGet` 남용**: GET으로 민감한 값 수신 지양(노출/북마크/로그). 필요한 값만 허용
7) **라우트 충돌**: 같은 경로에 여러 `@page` 템플릿 사용 금지. Conventions로 별칭 부여 시 중복 확인

---

## 15. 테스트 아이디어

- **라우팅 테스트**: `/products/42`가 원하는 PageModel로 매핑되는지 통합 테스트
- **바인딩 테스트**: `?page=2&size=2000` → 클램핑 적용 여부
- **정렬 화이트리스트**: 허용되지 않은 정렬값 입력 시 기본값으로 대체되는지
- **링크 생성**: `asp-page`/`asp-route-*`가 예상 URL을 생성하는지

---

## 16. 요약 표

| 주제                        | 핵심 요점 |
|-----------------------------|-----------|
| Route Parameter             | 리소스 식별자, 제약으로 강하게 제한, SEO/캐싱 우수 |
| Query String                | 뷰 상태(검색/필터/정렬/페이지), 유연·조합 용이 |
| 함께 쓰기                   | `/products/{category}?sort=price&page=2` 패턴 추천 |
| 링크 생성                   | `asp-page` + `asp-route-*`, `asp-all-route-data`로 병합 |
| Redirect에서 쿼리 유지      | PRG 패턴 + 현재 쿼리 병합(헬퍼/사전) |
| 바인딩 고급                 | `[BindProperty(SupportsGet=true)]`, 배열/복합 타입, 접두사 |
| 문화권/형식                 | ISO-8601/정규화, `TryParseExact`로 안정 파싱 |
| 보안                        | 화이트리스트/범위 제한/경로 정규화/권한 정책 결합 |
| 디버깅                      | 개발 모드 라우트 로깅, `HttpContext.GetEndpoint()` 확인 |

---

# 결론

- **Route**로 **리소스 식별**을, **Query**로 **보기 상태**를 표현하면 URL이 명료해지고 유지보수가 쉬워집니다.
- Razor Pages에서는 `@page` 템플릿과 제약으로 안전한 라우팅을 설계하고, `asp-page`/`asp-route-*`로 **강타입 링크 생성**을 사용하세요.
- PRG 패턴, 화이트리스트, 문화권/형식 정규화, URL 길이/인코딩에 유의하면 **견고한 페이징/필터 UI**를 만들 수 있습니다.
- 본문 예제들을 템플릿화하여 팀 표준(헬퍼/확장/Conventions)로 재사용하면, **일관된 URL 설계**와 **개발 속도**를 동시에 확보할 수 있습니다.
