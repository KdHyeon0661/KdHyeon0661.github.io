---
layout: post
title: AspNet - CRUD 구현
date: 2025-03-23 19:20:23 +0900
category: AspNet
---
# Razor Pages + EF Core 기반 CRUD 구현

- 동기 → **비동기(Async)** 전환, `AsNoTracking`, `CancellationToken`
- **유효성 검사**(Data Annotations) + **오버포스팅 방지**
- **PRG 패턴 + TempData 알림**, **폼 검증 요약/필드 오류 표시**
- **정렬/검색/페이징**(서버 사이드)과 쿼리스트링 유지
- **낙관적 동시성 제어**(RowVersion)
- **연관 로딩**(Include/ThenInclude) + **투영**(Select) + **DTO/ViewModel**
- **트랜잭션/리포지토리 패턴(옵션)**, **시드 데이터**, **소프트 삭제**
- **보안/성능 체크리스트**, **테스트(InMemory Provider)**

---

## 0. 전제: 모델과 DbContext (업데이트)

초안의 `Blog` 엔티티에 **유효성**, **동시성 토큰(RowVersion)**, **소프트 삭제** 필드를 추가한다.

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

public class Blog
{
    public int Id { get; set; }

    [Required, StringLength(120)]
    public string Title { get; set; } = "";

    [Required, StringLength(60)]
    public string Author { get; set; } = "";

    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    // 낙관적 동시성 제어
    [Timestamp]
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();

    // 소프트 삭제
    public bool IsDeleted { get; set; } = false;
}
```

`DbContext`에서 **글로벌 쿼리 필터**로 `IsDeleted == false`만 보이게 한다.

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> opt) : base(opt) { }

    public DbSet<Blog> Blogs => Set<Blog>();

    protected override void OnModelCreating(ModelBuilder mb)
    {
        // 테이블명
        mb.Entity<Blog>().ToTable("Blogs");

        // 글로벌 필터(소프트 삭제 제외)
        mb.Entity<Blog>().HasQueryFilter(b => !b.IsDeleted);

        // 인덱스(검색 최적화)
        mb.Entity<Blog>()
          .HasIndex(b => new { b.Title, b.Author });

        base.OnModelCreating(mb);
    }
}
```

> 마이그레이션 추가:
> `dotnet ef migrations add AddBlogValidationsAndRowVersion`
> `dotnet ef database update`

---

## 1. 폴더 구조 (초안 유지 + 지원 파일 추가)

```
Pages/
 ├── Shared/
 │   ├── _ValidationScriptsPartial.cshtml
 │   └── _StatusMessage.cshtml                ← TempData 알림 Partial
 └── Blogs/
     ├── Index.cshtml                         ← 목록 + 검색/정렬/페이징
     ├── Index.cshtml.cs
     ├── Create.cshtml                        ← 생성
     ├── Create.cshtml.cs
     ├── Edit.cshtml                          ← 수정(동시성)
     ├── Edit.cshtml.cs
     ├── Delete.cshtml                        ← 삭제(확인)
     ├── Delete.cshtml.cs
     ├── Details.cshtml                       ← 상세
     └── Details.cshtml.cs
```

---

## 2. 목록(Read) — 검색/정렬/페이징(서버 사이드)

### 2.1 ViewModel 정의

```csharp
public class BlogListItemDto
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public string Author { get; set; } = "";
    public DateTime CreatedAt { get; set; }
}

// 목록 페이지에서 필요한 상태
public class BlogIndexQuery
{
    public string? Keyword { get; set; }
    public string Sort { get; set; } = "created_desc"; // created_asc|created_desc|title_asc|title_desc
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 10;
}

public class PagedResult<T>
{
    public IReadOnlyList<T> Items { get; init; } = Array.Empty<T>();
    public int Page { get; init; }
    public int PageSize { get; init; }
    public int TotalCount { get; init; }
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
}
```

### 2.2 Index.cshtml.cs (비동기 + 투영 + AsNoTracking)

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;

public class IndexModel : PageModel
{
    private readonly AppDbContext _db;
    private const int MaxPageSize = 100;

    public IndexModel(AppDbContext db) => _db = db;

    [BindProperty(SupportsGet = true)]
    public BlogIndexQuery Input { get; set; } = new();

    public PagedResult<BlogListItemDto> Result { get; private set; } = new();

    public async Task OnGetAsync(CancellationToken ct)
    {
        // 기본 쿼리(소프트 삭제 제외 필터 자동 적용)
        IQueryable<Blog> q = _db.Blogs.AsNoTracking();

        // 검색
        if (!string.IsNullOrWhiteSpace(Input.Keyword))
        {
            var kw = Input.Keyword.Trim();
            q = q.Where(b => b.Title.Contains(kw) || b.Author.Contains(kw));
        }

        // 정렬
        q = Input.Sort switch
        {
            "created_asc" => q.OrderBy(b => b.CreatedAt).ThenBy(b => b.Id),
            "title_asc"   => q.OrderBy(b => b.Title).ThenBy(b => b.Id),
            "title_desc"  => q.OrderByDescending(b => b.Title).ThenBy(b => b.Id),
            _             => q.OrderByDescending(b => b.CreatedAt).ThenBy(b => b.Id)
        };

        // 페이징
        var pageSize = Math.Clamp(Input.PageSize, 1, MaxPageSize);
        var total = await q.CountAsync(ct);
        var page = Math.Max(1, Input.Page);
        var items = await q.Skip((page - 1) * pageSize)
                           .Take(pageSize)
                           .Select(b => new BlogListItemDto
                           {
                               Id = b.Id,
                               Title = b.Title,
                               Author = b.Author,
                               CreatedAt = b.CreatedAt
                           })
                           .ToListAsync(ct);

        Result = new PagedResult<BlogListItemDto>
        {
            Items = items,
            Page = page,
            PageSize = pageSize,
            TotalCount = total
        };
    }
}
```

### 2.3 Index.cshtml (정렬/검색/페이징 UI + 쿼리 유지)

```razor
@page
@model IndexModel
@{
    ViewData["Title"] = "블로그 목록";
}

<partial name="~/Pages/Shared/_StatusMessage.cshtml" />

<h2>블로그 목록</h2>

<form method="get" class="mb-2">
    <input type="text" name="Keyword" value="@Model.Input.Keyword" placeholder="검색(제목/작성자)" />
    <select name="Sort">
        <option value="created_desc" selected="@(Model.Input.Sort == "created_desc")">최신순</option>
        <option value="created_asc" selected="@(Model.Input.Sort == "created_asc")">오래된순</option>
        <option value="title_asc" selected="@(Model.Input.Sort == "title_asc")">제목↑</option>
        <option value="title_desc" selected="@(Model.Input.Sort == "title_desc")">제목↓</option>
    </select>
    <select name="PageSize">
        <option>10</option><option>20</option><option>50</option>
    </select>
    <button type="submit">적용</button>
    <a asp-page="Create" class="btn">새 블로그</a>
</form>

<table>
    <thead>
        <tr><th>제목</th><th>작성자</th><th>생성일</th><th>관리</th></tr>
    </thead>
    <tbody>
    @foreach (var b in Model.Result.Items)
    {
        <tr>
            <td>@b.Title</td>
            <td>@b.Author</td>
            <td>@b.CreatedAt.ToLocalTime().ToString("yyyy-MM-dd HH:mm")</td>
            <td>
                <a asp-page="Details" asp-route-id="@b.Id">보기</a> |
                <a asp-page="Edit" asp-route-id="@b.Id">수정</a> |
                <a asp-page="Delete" asp-route-id="@b.Id">삭제</a>
            </td>
        </tr>
    }
    </tbody>
</table>

@if (Model.Result.TotalPages > 1)
{
    <nav>
        @for (var i = 1; i <= Model.Result.TotalPages; i++)
        {
            <a asp-page="./Index"
               asp-route-Page="@i"
               asp-route-Keyword="@Model.Input.Keyword"
               asp-route-Sort="@Model.Input.Sort"
               asp-route-PageSize="@Model.Input.PageSize"
               class="@(i == Model.Result.Page ? "active" : "")">@i</a>
        }
    </nav>
}
```

> **포인트**
> - 투영(`Select`)으로 필요한 필드만 가져와 **성능 최적화**
> - `AsNoTracking()`으로 목록 조회 성능 향상
> - 쿼리스트링 유지로 UX 향상

---

## 3. Create(생성) — 오버포스팅 방지 + PRG + TempData 알림

### 3.1 DTO(입력 전용)로 오버포스팅 방지

```csharp
public class BlogCreateDto
{
    [Required, StringLength(120)]
    public string Title { get; set; } = "";

    [Required, StringLength(60)]
    public string Author { get; set; } = "";
}
```

### 3.2 Create.cshtml.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

public class CreateModel : PageModel
{
    private readonly AppDbContext _db;
    public CreateModel(AppDbContext db) => _db = db;

    [BindProperty]
    public BlogCreateDto Input { get; set; } = new();

    [TempData] public string? StatusMessage { get; set; }

    public void OnGet() { }

    public async Task<IActionResult> OnPostAsync(CancellationToken ct)
    {
        if (!ModelState.IsValid) return Page();

        var entity = new Blog
        {
            Title = Input.Title.Trim(),
            Author = Input.Author.Trim(),
            CreatedAt = DateTime.UtcNow
        };

        // 트랜잭션(단순 예, 다중 연산 시 고려)
        await using var tx = await _db.Database.BeginTransactionAsync(ct);
        try
        {
            _db.Blogs.Add(entity);
            await _db.SaveChangesAsync(ct);
            await tx.CommitAsync(ct);

            StatusMessage = "생성되었습니다.";
            return RedirectToPage("Index"); // PRG
        }
        catch
        {
            await tx.RollbackAsync(ct);
            ModelState.AddModelError("", "저장 중 오류가 발생했습니다.");
            return Page();
        }
    }
}
```

### 3.3 Create.cshtml

```razor
@page
@model CreateModel
@{
    ViewData["Title"] = "새 블로그 작성";
}

<h2>새 블로그 작성</h2>

<form method="post">
    <div>
        <label asp-for="Input.Title"></label>
        <input asp-for="Input.Title" />
        <span asp-validation-for="Input.Title"></span>
    </div>
    <div>
        <label asp-for="Input.Author"></label>
        <input asp-for="Input.Author" />
        <span asp-validation-for="Input.Author"></span>
    </div>
    <button type="submit">등록</button>
    <a asp-page="Index">취소</a>
</form>

@section Scripts {
    <partial name="~/Pages/Shared/_ValidationScriptsPartial.cshtml" />
}
```

---

## 4. Details(상세) — 비동기 + NotFound 처리

### Details.cshtml.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;

public class DetailsModel : PageModel
{
    private readonly AppDbContext _db;
    public DetailsModel(AppDbContext db) => _db = db;

    public Blog? Blog { get; private set; }

    public async Task<IActionResult> OnGetAsync(int id, CancellationToken ct)
    {
        Blog = await _db.Blogs.AsNoTracking().FirstOrDefaultAsync(b => b.Id == id, ct);
        if (Blog is null) return NotFound();
        return Page();
    }
}
```

### Details.cshtml

```razor
@page "{id:int}"
@model DetailsModel
@{
    ViewData["Title"] = "블로그 상세";
}

<h2>블로그 상세</h2>
@if (Model.Blog is not null)
{
    <p><strong>제목:</strong> @Model.Blog.Title</p>
    <p><strong>작성자:</strong> @Model.Blog.Author</p>
    <p><strong>생성일:</strong> @Model.Blog.CreatedAt.ToLocalTime()</p>
}
<a asp-page="Index">← 목록</a>
```

---

## 5. Edit(수정) — 낙관적 동시성 제어 (RowVersion)

### 5.1 DTO(수정 전용) + RowVersion 포함

```csharp
public class BlogEditDto
{
    public int Id { get; set; }

    [Required, StringLength(120)]
    public string Title { get; set; } = "";

    [Required, StringLength(60)]
    public string Author { get; set; } = "";

    // 동시성 토큰
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}
```

### 5.2 Edit.cshtml.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;

public class EditModel : PageModel
{
    private readonly AppDbContext _db;
    public EditModel(AppDbContext db) => _db = db;

    [BindProperty]
    public BlogEditDto Input { get; set; } = new();

    [TempData] public string? StatusMessage { get; set; }

    public async Task<IActionResult> OnGetAsync(int id, CancellationToken ct)
    {
        var entity = await _db.Blogs.AsNoTracking().FirstOrDefaultAsync(b => b.Id == id, ct);
        if (entity is null) return NotFound();

        Input = new BlogEditDto
        {
            Id = entity.Id,
            Title = entity.Title,
            Author = entity.Author,
            RowVersion = entity.RowVersion
        };
        return Page();
    }

    public async Task<IActionResult> OnPostAsync(CancellationToken ct)
    {
        if (!ModelState.IsValid) return Page();

        // 추적 엔티티 생성 후 동시성 토큰 포함 갱신
        var entity = new Blog
        {
            Id = Input.Id,
            Title = Input.Title.Trim(),
            Author = Input.Author.Trim(),
            RowVersion = Input.RowVersion
        };

        _db.Attach(entity);
        _db.Entry(entity).Property(x => x.Title).IsModified = true;
        _db.Entry(entity).Property(x => x.Author).IsModified = true;
        // CreatedAt은 수정하지 않음(오버포스팅 방지)
        _db.Entry(entity).Property(x => x.CreatedAt).IsModified = false;
        // RowVersion은 동시성 토큰
        _db.Entry(entity).Property(x => x.RowVersion).OriginalValue = Input.RowVersion;

        try
        {
            await _db.SaveChangesAsync(ct);
            StatusMessage = "수정되었습니다.";
            return RedirectToPage("Index");
        }
        catch (DbUpdateConcurrencyException)
        {
            // 현재 DB의 최신 값을 읽어 사용자에게 충돌 메시지 제공
            var dbEntity = await _db.Blogs.AsNoTracking().FirstOrDefaultAsync(b => b.Id == Input.Id, ct);
            if (dbEntity is null) return NotFound();

            ModelState.AddModelError("", "다른 사용자가 먼저 수정했습니다. 현재 DB 값을 확인 후 다시 저장하세요.");

            // DB 값으로 폼에 최신 표시
            Input.RowVersion = dbEntity.RowVersion;
            Input.Title = dbEntity.Title;
            Input.Author = dbEntity.Author;

            return Page();
        }
    }
}
```

### 5.3 Edit.cshtml

```razor
@page "{id:int}"
@model EditModel
@{
    ViewData["Title"] = "블로그 수정";
}

<h2>블로그 수정</h2>

<form method="post">
    <input type="hidden" asp-for="Input.Id" />
    <input type="hidden" asp-for="Input.RowVersion" />

    <div>
        <label asp-for="Input.Title"></label>
        <input asp-for="Input.Title" />
        <span asp-validation-for="Input.Title"></span>
    </div>
    <div>
        <label asp-for="Input.Author"></label>
        <input asp-for="Input.Author" />
        <span asp-validation-for="Input.Author"></span>
    </div>

    <button type="submit">저장</button>
    <a asp-page="Index">취소</a>
</form>

@section Scripts {
    <partial name="~/Pages/Shared/_ValidationScriptsPartial.cshtml" />
}
```

---

## 6. Delete(삭제) — 소프트 삭제 + 확인 페이지

### Delete.cshtml.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;

public class DeleteModel : PageModel
{
    private readonly AppDbContext _db;
    public DeleteModel(AppDbContext db) => _db = db;

    [BindProperty]
    public Blog? Blog { get; set; }

    [TempData] public string? StatusMessage { get; set; }

    public async Task<IActionResult> OnGetAsync(int id, CancellationToken ct)
    {
        Blog = await _db.Blogs.AsNoTracking().FirstOrDefaultAsync(b => b.Id == id, ct);
        if (Blog is null) return NotFound();
        return Page();
    }

    public async Task<IActionResult> OnPostAsync(int id, CancellationToken ct)
    {
        var blog = await _db.Blogs.FirstOrDefaultAsync(b => b.Id == id, ct);
        if (blog is null) return NotFound();

        // 소프트 삭제
        blog.IsDeleted = true;
        await _db.SaveChangesAsync(ct);

        StatusMessage = "삭제되었습니다.";
        return RedirectToPage("Index");
    }
}
```

### Delete.cshtml

```razor
@page "{id:int}"
@model DeleteModel
@{
    ViewData["Title"] = "삭제 확인";
}

<h2>삭제 확인</h2>
@if (Model.Blog is not null)
{
    <p><strong>@Model.Blog.Title</strong> 글을 삭제하시겠습니까?</p>
    <form method="post">
        <button type="submit">삭제</button>
        <a asp-page="Index">취소</a>
    </form>
}
```

---

## 7. 알림 Partial(TempData) — `_StatusMessage.cshtml`

```razor
@* Pages/Shared/_StatusMessage.cshtml *@
@{
    var msg = TempData["StatusMessage"] as string;
}
@if (!string.IsNullOrEmpty(msg))
{
    <div class="alert alert-success">@msg</div>
}
```

> `Create/Edit/Delete`의 `StatusMessage`가 PRG 패턴으로 표시된다.

---

## 8. 검증/보안 모범 사례

- **오버포스팅 방지**: 엔티티를 직접 바인딩하지 말고 **DTO** 사용 또는 `TryUpdateModelAsync(entity, includeProperties...)`로 허용 필드만 갱신
- **[ValidateAntiForgeryToken]**: Razor Pages는 기본 활성. `<form method="post">`에 자동 삽입
- **ModelState** 오류 요약/필드 오류 UI 제공
- **AsNoTracking**: 읽기 전용 쿼리 성능 향상
- **RowVersion**: 낙관적 동시성으로 충돌 방지
- **쿼리 제한**: 키워드 길이/페이지 크기 상한으로 과도한 부하 방지
- **Index/Include**: 검색/정렬 열에 인덱스, 필요한 경우만 Include
- **Raw SQL**: 필요 시 `FromSqlInterpolated` 사용(파라미터 바인딩 필수)

---

## 9. 연관 로딩/투영(예시)

`Blog`가 `Posts`를 갖는다고 가정:

```csharp
var withPosts = await _db.Blogs
    .Include(b => b.Posts.Where(p => !p.IsDeleted))
    .ThenInclude(p => p.Tags)
    .AsNoTracking()
    .FirstOrDefaultAsync(b => b.Id == id, ct);
```

대신 전송량 절약을 위해 **투영**:

```csharp
var dto = await _db.Blogs
    .Where(b => b.Id == id)
    .Select(b => new BlogDetailsDto
    {
        Id = b.Id,
        Title = b.Title,
        Author = b.Author,
        CreatedAt = b.CreatedAt,
        PostCount = b.Posts.Count(p => !p.IsDeleted)
    })
    .AsNoTracking()
    .FirstOrDefaultAsync(ct);
```

---

## 10. 트랜잭션/리포지토리(옵션)

단일 SaveChanges면 EF가 내부 트랜잭션을 쓴다. **여러 Aggregate**를 한 번에 처리하면 명시적으로 트랜잭션.

```csharp
await using var tx = await _db.Database.BeginTransactionAsync(ct);
try
{
    _db.Blogs.Add(blog);
    // 다른 엔티티 변경...
    await _db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
}
catch
{
    await tx.RollbackAsync(ct);
    throw;
}
```

리포지토리를 도입할 경우 `IRepository<T>` 인터페이스로 쿼리/명령을 래핑하되, **과도한 추상화**는 지양(단순 CRUD에 EF가 이미 리포지토리 역할).

---

## 11. Seed 데이터 (개발/테스트 편의)

`Program.cs`에서 마이그레이션 후 시드:

```csharp
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();

    if (!await db.Blogs.AnyAsync())
    {
        db.Blogs.AddRange(
            new Blog { Title = "첫 글", Author = "kim", CreatedAt = DateTime.UtcNow },
            new Blog { Title = "두 번째", Author = "lee", CreatedAt = DateTime.UtcNow }
        );
        await db.SaveChangesAsync();
    }
}
```

---

## 12. 테스트(InMemory Provider)

```csharp
using Microsoft.EntityFrameworkCore;
using Xunit;

public class BlogTests
{
    [Fact]
    public async Task Create_Blog_Success()
    {
        var opt = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase("blogs")
            .Options;

        await using var db = new AppDbContext(opt);
        db.Blogs.Add(new Blog { Title = "Hello", Author = "kim" });
        await db.SaveChangesAsync();

        var count = await db.Blogs.CountAsync();
        Assert.Equal(1, count);
    }
}
```

> 주의: InMemory는 **관계/트랜잭션** 동작이 실제 RDB와 달라질 수 있다.
> 통합 테스트는 SQLite In-Memory 모드 사용을 고려.

---

## 13. 성능/운영 체크리스트

- [ ] **페이지네이션**: `Skip/Take` + 인덱스
- [ ] **투영**: 필요한 필드만 `Select`
- [ ] **NoTracking**: 읽기 전용
- [ ] **쿼리 로그**: 개발에만 `EnableSensitiveDataLogging`
- [ ] **캔슬 토큰**: 긴 쿼리 취소 경로 제공
- [ ] **적절한 컬럼 인덱스**: 정렬/검색 칼럼
- [ ] **배치 저장**: 한 번에 `SaveChanges`(단, 트랜잭션 고려)
- [ ] **N+1** 회피: Include 또는 별도 `IN` 질의로 일괄 조회

---

## 14. 전체 요약

| 작업 | 핵심 포인트 |
|---|---|
| Read | `AsNoTracking` + 검색/정렬/페이징 + 투영 |
| Create | DTO로 오버포스팅 방지 + PRG + TempData |
| Details | NotFound/에러 처리 + 비동기 |
| Edit | `RowVersion` 기반 낙관적 동시성 + 부분 필드만 수정 |
| Delete | 소프트 삭제 + 글로벌 필터 |
| 공통 | 유효성 검사/안티포저리/로깅/트랜잭션/테스트 전략 |

---

## 15. 부록: _ValidationScriptsPartial.cshtml (기본)

```razor
@* 보통 템플릿에 포함됨: jQuery Validation + Unobtrusive *@
<script src="~/lib/jquery/dist/jquery.min.js"></script>
<script src="~/lib/jquery-validation/dist/jquery.validate.min.js"></script>
<script src="~/lib/jquery-validation-unobtrusive/jquery.validate.unobtrusive.min.js"></script>
```

실서비스에서는 번들링/무결성(SRI), CDN/폴백 전략을 적용한다.

---

# 결론

초안의 CRUD를 **실무 수준**으로 확장했다.
- 안정성: 유효성/오버포스팅 방지/동시성/소프트 삭제
- UX: PRG + TempData, 검색/정렬/페이징과 쿼리 유지
- 성능: NoTracking/투영/인덱스
- 확장: DTO/ViewModel/트랜잭션/시드/테스트
