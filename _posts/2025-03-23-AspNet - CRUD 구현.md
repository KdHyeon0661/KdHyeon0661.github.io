---
layout: post
title: AspNet - CRUD 구현
date: 2025-03-23 19:20:23 +0900
category: AspNet
---
# 🛠️ Razor Pages + EF Core 기반 CRUD 구현

---

## 📌 전제 조건

- ASP.NET Core Razor Pages 프로젝트
- `EF Core`, `DbContext`, `Migrations`까지 완료된 상태
- `Blog` 모델 예시 사용

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Author { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

---

## 🏗️ Pages 폴더 구조 (CRUD 예시)

```
Pages/
 ├── Blogs/
 │   ├── Index.cshtml         => 목록 (Read)
 │   ├── Create.cshtml        => 생성 (Create)
 │   ├── Edit.cshtml          => 수정 (Update)
 │   ├── Delete.cshtml        => 삭제 (Delete)
 │   └── Details.cshtml       => 상세 보기
```

---

## 📚 Read - 목록 조회 (`Index.cshtml`)

### ✅ Index.cshtml.cs

```csharp
public class IndexModel : PageModel
{
    private readonly AppDbContext _context;

    public IndexModel(AppDbContext context) => _context = context;

    public List<Blog> Blogs { get; set; }

    public void OnGet()
    {
        Blogs = _context.Blogs.OrderByDescending(b => b.CreatedAt).ToList();
    }
}
```

---

### ✅ Index.cshtml

```html
@page
@model IndexModel

<h2>📄 블로그 목록</h2>
<a asp-page="Create">+ 새 블로그</a>

<table>
    <thead><tr><th>제목</th><th>작성자</th><th>생성일</th><th>관리</th></tr></thead>
    <tbody>
    @foreach (var blog in Model.Blogs)
    {
        <tr>
            <td>@blog.Title</td>
            <td>@blog.Author</td>
            <td>@blog.CreatedAt.ToShortDateString()</td>
            <td>
                <a asp-page="Edit" asp-route-id="@blog.Id">수정</a> |
                <a asp-page="Delete" asp-route-id="@blog.Id">삭제</a> |
                <a asp-page="Details" asp-route-id="@blog.Id">보기</a>
            </td>
        </tr>
    }
    </tbody>
</table>
```

---

## 🆕 Create - 새 블로그 생성

### ✅ Create.cshtml.cs

```csharp
public class CreateModel : PageModel
{
    private readonly AppDbContext _context;
    [BindProperty] public Blog Blog { get; set; }

    public CreateModel(AppDbContext context) => _context = context;

    public void OnGet() => Blog = new Blog();

    public IActionResult OnPost()
    {
        if (!ModelState.IsValid) return Page();

        Blog.CreatedAt = DateTime.Now;
        _context.Blogs.Add(Blog);
        _context.SaveChanges();

        return RedirectToPage("Index");
    }
}
```

---

### ✅ Create.cshtml

```html
@page
@model CreateModel
<h2>📝 새 블로그 작성</h2>

<form method="post">
    <label>제목</label>
    <input asp-for="Blog.Title" />
    <label>작성자</label>
    <input asp-for="Blog.Author" />
    <button type="submit">등록</button>
</form>
```

---

## ✏️ Update - 블로그 수정

### ✅ Edit.cshtml.cs

```csharp
public class EditModel : PageModel
{
    private readonly AppDbContext _context;
    [BindProperty] public Blog Blog { get; set; }

    public EditModel(AppDbContext context) => _context = context;

    public IActionResult OnGet(int id)
    {
        Blog = _context.Blogs.Find(id);
        if (Blog == null) return NotFound();
        return Page();
    }

    public IActionResult OnPost()
    {
        if (!ModelState.IsValid) return Page();

        _context.Attach(Blog).State = EntityState.Modified;
        _context.SaveChanges();

        return RedirectToPage("Index");
    }
}
```

---

### ✅ Edit.cshtml

```html
@page "{id:int}"
@model EditModel

<h2>✏️ 블로그 수정</h2>

<form method="post">
    <input type="hidden" asp-for="Blog.Id" />
    <label>제목</label>
    <input asp-for="Blog.Title" />
    <label>작성자</label>
    <input asp-for="Blog.Author" />
    <button type="submit">수정</button>
</form>
```

---

## 🗑️ Delete - 삭제 처리

### ✅ Delete.cshtml.cs

```csharp
public class DeleteModel : PageModel
{
    private readonly AppDbContext _context;
    [BindProperty] public Blog Blog { get; set; }

    public DeleteModel(AppDbContext context) => _context = context;

    public IActionResult OnGet(int id)
    {
        Blog = _context.Blogs.Find(id);
        if (Blog == null) return NotFound();
        return Page();
    }

    public IActionResult OnPost()
    {
        var blog = _context.Blogs.Find(Blog.Id);
        if (blog == null) return NotFound();

        _context.Blogs.Remove(blog);
        _context.SaveChanges();
        return RedirectToPage("Index");
    }
}
```

---

### ✅ Delete.cshtml

```html
@page "{id:int}"
@model DeleteModel

<h2>🗑️ 삭제 확인</h2>
<p>@Model.Blog.Title 을 정말 삭제할까요?</p>

<form method="post">
    <input type="hidden" asp-for="Blog.Id" />
    <button type="submit">삭제</button>
    <a asp-page="Index">취소</a>
</form>
```

---

## 🔍 Details - 상세 보기

### ✅ Details.cshtml.cs

```csharp
public class DetailsModel : PageModel
{
    private readonly AppDbContext _context;
    public Blog Blog { get; set; }

    public DetailsModel(AppDbContext context) => _context = context;

    public IActionResult OnGet(int id)
    {
        Blog = _context.Blogs.FirstOrDefault(b => b.Id == id);
        if (Blog == null) return NotFound();
        return Page();
    }
}
```

---

### ✅ Details.cshtml

```html
@page "{id:int}"
@model DetailsModel

<h2>🔎 블로그 상세</h2>
<p><strong>제목:</strong> @Model.Blog.Title</p>
<p><strong>작성자:</strong> @Model.Blog.Author</p>
<p><strong>생성일:</strong> @Model.Blog.CreatedAt</p>
<a asp-page="Index">← 목록</a>
```

---

## ✅ 마무리 요약

| 작업 | 설명 |
|------|------|
| **Create** | 폼 제출 후 `OnPost`에서 `Add()` & `SaveChanges()` |
| **Read**   | `Index.cshtml.cs`에서 리스트 출력 |
| **Update** | `Find(id)` → 바인딩 후 수정 |
| **Delete** | 삭제 전 확인 페이지 → `Remove()` 호출 |
| **Details**| 단일 항목 표시용 페이지 |
