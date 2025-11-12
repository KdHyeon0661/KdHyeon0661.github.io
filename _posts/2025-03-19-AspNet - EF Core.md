---
layout: post
title: AspNet - EF Core
date: 2025-03-19 20:20:23 +0900
category: AspNet
---
# Entity Framework Core (EF Core) 소개 및 설치

## 1. EF Core란?

**EF Core**는 .NET 애플리케이션에서 **객체-관계 매핑(ORM)**을 제공하여 C# 객체로 데이터베이스를 다룰 수 있게 한다.

- **ORM**: 엔티티 클래스 ↔ 테이블, 속성 ↔ 컬럼 매핑  
- **LINQ**: 타입 안전한 질의(컴파일 타임 검증)  
- **변경 추적(Change Tracking)**: 엔티티 상태 자동 관리  
- **마이그레이션(Migration)**: 스키마 버전 관리  
- **다중 공급자**: SQL Server/SQLite/PostgreSQL/MySQL/Oracle(비공식)/Cosmos DB 등

---

## 2. EF Core의 주요 특징 정리

| 기능 | 요점 |
|---|---|
| ORM | POCO(Entity) 정의만으로 스키마·CRUD 가능 |
| LINQ | 쿼리, 그룹핑, 조인, 프로젝션을 C#으로 작성 |
| 마이그레이션 | `dotnet ef`로 스키마 이력/롤백/스크립트화 |
| 멀티 DB | Provider 교체만으로 타 DB 이식성 확보 |
| 로딩 전략 | Eager(`Include`), Explicit(`Load`), Lazy(옵션) |
| 추적/비추적 | `AsNoTracking()`으로 읽기 성능 최적화 |
| 확장성 | ValueConverter, Owned type, Shadow property |
| 성능 도구 | Compiled queries, Context pooling, SplitQuery |

---

## 3. 지원 데이터베이스와 선택 기준

- **SQL Server**: 기본 선택, 기능·도구 지원이 가장 풍부  
- **SQLite**: 로컬/임베디드/테스트에 적합  
- **PostgreSQL(Npgsql)**: 오픈소스, JSONB/Full Text 등 풍부한 기능  
- **MySQL/MariaDB(Pomelo)**: LAMP/LNMP 환경 연동  
- **Cosmos DB**: NoSQL(문서형) 필요 시

**선택 팁**  
- 클라우드/엔터프라이즈: SQL Server, PostgreSQL  
- 데스크톱/로컬 개발: SQLite  
- NoSQL 문서 저장/글로벌 배포: Cosmos DB

---

## 4. 설치 — 패키지 의존 관계와 권장 조합

### 4.1 필수 패키지(프로바이더 + 도구)

#### SQL Server
```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.EntityFrameworkCore.Design
```

#### PostgreSQL
```bash
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.EntityFrameworkCore.Design
```

#### SQLite
```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.EntityFrameworkCore.Design
```

> `Tools`는 `dotnet ef` 내부 연동, `Design`은 런타임 외 디자인 타임(마이그레이션) 지원.

---

## 5. 프로젝트에 DbContext 등록

### 5.1 엔티티(Entity)와 DbContext

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; } = default!;
    public string Author { get; set; } = default!;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    public ICollection<Post> Posts { get; set; } = new List<Post>();
}

public class Post
{
    public int Id { get; set; }
    public int BlogId { get; set; }
    public string Slug { get; set; } = default!;
    public string Content { get; set; } = default!;
    public DateTime PublishedAt { get; set; }

    public Blog Blog { get; set; } = default!;
}
```

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Blog> Blogs => Set<Blog>();
    public DbSet<Post> Posts => Set<Post>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Blog>(b =>
        {
            b.ToTable("Blogs");
            b.HasKey(x => x.Id);
            b.Property(x => x.Title).HasMaxLength(100).IsRequired();
            b.Property(x => x.Author).HasMaxLength(60).IsRequired();
            b.HasIndex(x => new { x.Author, x.CreatedAt });
        });

        modelBuilder.Entity<Post>(p =>
        {
            p.ToTable("Posts");
            p.HasKey(x => x.Id);
            p.Property(x => x.Slug).HasMaxLength(120).IsRequired();
            p.Property(x => x.Content).IsRequired();
            p.HasIndex(x => x.Slug).IsUnique();

            p.HasOne(x => x.Blog)
             .WithMany(x => x.Posts)
             .HasForeignKey(x => x.BlogId)
             .OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

### 5.2 Program.cs — Provider 구성

```csharp
var builder = WebApplication.CreateBuilder(args);

// SQL Server
// builder.Services.AddDbContext<AppDbContext>(opt =>
//     opt.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// SQLite (로컬/개발 권장)
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlite(builder.Configuration.GetConnectionString("SqliteConnection"))
       .EnableDetailedErrors()
       .EnableSensitiveDataLogging(builder.Environment.IsDevelopment()));

// 컨텍스트 풀링(고QPS API에 유리)
builder.Services.AddDbContextPool<AppDbContext>(opt =>
    opt.UseSqlite(builder.Configuration.GetConnectionString("SqliteConnection")));

var app = builder.Build();
app.MapGet("/", (AppDbContext db) => db.Blogs.Count());
app.Run();
```

### 5.3 연결 문자열

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=MyDb;Trusted_Connection=True;",
    "SqliteConnection": "Data Source=blog.db"
  }
}
```

---

## 6. 마이그레이션 도입과 운용

### 6.1 dotnet-ef 도구 설치(최초 1회)
```bash
dotnet tool install --global dotnet-ef
```

### 6.2 초기 마이그레이션 생성
```bash
dotnet ef migrations add InitialCreate
```

생성물
- `Migrations/<timestamp>_InitialCreate.cs` (Up/Down)
- `Migrations/AppDbContextModelSnapshot.cs` (스냅샷)

### 6.3 DB 반영
```bash
dotnet ef database update
```

### 6.4 모델 변경 → 누적 마이그레이션
```bash
# 엔티티에 속성/관계 추가 후
dotnet ef migrations add AddPostSummary
dotnet ef database update
```

### 6.5 롤백/제거
```bash
dotnet ef database update InitialCreate   # 특정 버전으로 롤백
dotnet ef migrations remove               # 마지막 마이그레이션 제거(코드만)
```

### 6.6 SQL 스크립트 추출(운영 반영)
```bash
dotnet ef migrations script -o ./sql/000_full.sql
dotnet ef migrations script PrevMigName NewMigName -o ./sql/010_delta.sql
```

### 6.7 Migration Bundle(.NET 7+)
```bash
dotnet ef migrations bundle --configuration Release --self-contained
./efbundle --connection "Data Source=blog.db"
```

---

## 7. LINQ 사용 예 — 쿼리/정렬/프로젝션

```csharp
public class IndexModel : PageModel
{
    private readonly AppDbContext _db;
    public IndexModel(AppDbContext db) => _db = db;

    public List<BlogVm> Blogs { get; set; } = new();

    public void OnGet()
    {
        Blogs = _db.Blogs
            .Where(b => b.Title.Contains("ASP"))
            .OrderBy(b => b.Id)
            .Select(b => new BlogVm
            {
                Id = b.Id,
                Title = b.Title,
                PostCount = b.Posts.Count
            })
            .AsNoTracking()
            .ToList();
    }

    public record BlogVm(int Id, string Title, int PostCount);
}
```

**키 포인트**
- `AsNoTracking()`은 읽기 전용 쿼리에 성능 이점.
- `Select`로 필요한 필드만 투영해 네트워크/메모리 최적화.

---

## 8. CRUD 패턴과 트랜잭션

### 8.1 추가/수정/삭제

```csharp
public class BlogService
{
    private readonly AppDbContext _db;
    public BlogService(AppDbContext db) => _db = db;

    public async Task<int> AddBlogAsync(string title, string author)
    {
        var blog = new Blog { Title = title, Author = author };
        _db.Blogs.Add(blog);
        await _db.SaveChangesAsync();
        return blog.Id;
    }

    public async Task UpdateTitleAsync(int id, string newTitle)
    {
        var blog = await _db.Blogs.FindAsync(id);
        if (blog is null) return;
        blog.Title = newTitle;
        await _db.SaveChangesAsync();
    }

    public async Task DeleteAsync(int id)
    {
        var blog = await _db.Blogs.FindAsync(id);
        if (blog is null) return;
        _db.Blogs.Remove(blog);
        await _db.SaveChangesAsync();
    }
}
```

### 8.2 트랜잭션(원자성)

```csharp
using var tx = await _db.Database.BeginTransactionAsync();
try
{
    // 여러 엔터티 작업
    await _db.SaveChangesAsync();
    await tx.CommitAsync();
}
catch
{
    await tx.RollbackAsync();
    throw;
}
```

---

## 9. 로딩 전략 — Eager/Explicit/Lazy

- **Eager**: `Include`로 즉시 합류
```csharp
var blogs = await _db.Blogs
    .Include(b => b.Posts)
    .OrderByDescending(b => b.CreatedAt)
    .ToListAsync();
```

- **Explicit**: 필요 시점에 명시적으로 로드
```csharp
var blog = await _db.Blogs.FirstAsync();
await _db.Entry(blog).Collection(b => b.Posts).LoadAsync();
```

- **Lazy**: 별도 설정 필요(프록시) — 예측 어려움과 N+1 리스크 주의

---

## 10. 고급 매핑 — ValueConverter/Owned/Shadow/동시성

### 10.1 ValueConverter
```csharp
public enum Visibility { Private, Public }

public class Note
{
    public int Id { get; set; }
    public Visibility Visibility { get; set; }
}

protected override void OnModelCreating(ModelBuilder mb)
{
    var conv = new ValueConverter<Visibility, string>(
        v => v.ToString(),
        s => Enum.Parse<Visibility>(s));

    mb.Entity<Note>()
      .Property(n => n.Visibility)
      .HasConversion(conv)
      .HasMaxLength(16);
}
```

### 10.2 Owned Type
```csharp
public class AuditInfo { public string CreatedBy { get; set; } = default!; public DateTime CreatedAt { get; set; } }
public class Comment { public int Id { get; set; } public int PostId { get; set; } public AuditInfo Audit { get; set; } = new(); public string Body { get; set; } = default!; }

protected override void OnModelCreating(ModelBuilder mb)
{
    mb.Entity<Comment>().OwnsOne(c => c.Audit, a => a.Property(x => x.CreatedBy).HasMaxLength(40));
}
```

### 10.3 Shadow Property
```csharp
protected override void OnModelCreating(ModelBuilder mb)
{
    mb.Entity<Post>().Property<DateTime>("LastModified").HasDefaultValueSql("CURRENT_TIMESTAMP");
}
```
```csharp
_db.Entry(post).Property("LastModified").CurrentValue = DateTime.UtcNow;
```

### 10.4 동시성 토큰
```csharp
public class Inventory
{
    public int Id { get; set; }
    public string Sku { get; set; } = default!;
    [Timestamp] public byte[] RowVersion { get; set; } = default!;
}
```
```csharp
try { await _db.SaveChangesAsync(); }
catch (DbUpdateConcurrencyException) { /* 재시도/사용자 병합 로직 */ }
```

---

## 11. Raw SQL 과 안전한 파라미터화

```csharp
// 엔티티 반환
var recent = await _db.Posts
    .FromSqlInterpolated($"SELECT * FROM Posts WHERE PublishedAt > {DateTime.UtcNow.AddDays(-7)}")
    .AsNoTracking()
    .ToListAsync();

// 비-엔티티 프로젝션(.NET 7+ Database.SqlQuery)
public record PostSummary(int Id, string Slug);

var rows = await _db.Database
    .SqlQuery<PostSummary>($"SELECT Id, Slug FROM Posts WHERE BlogId = {blogId}")
    .ToListAsync();
```

> `FromSqlRaw`에 문자열 결합 금지. 반드시 파라미터화 사용.

---

## 12. 시드(Seed) 데이터

### 12.1 Fluent API 기반(마이그레이션 포함)
```csharp
modelBuilder.Entity<Blog>().HasData(
    new Blog { Id = 1, Title = "EF Core Guide", Author = "kim", CreatedAt = DateTime.UtcNow }
);
```

### 12.2 런타임 초기화
```csharp
public static class DbInit
{
    public static async Task EnsureSeedAsync(IServiceProvider sp)
    {
        using var scope = sp.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();

        if (!db.Blogs.Any())
        {
            db.Blogs.Add(new Blog { Title = "Hello", Author = "admin" });
            await db.SaveChangesAsync();
        }
    }
}
```

---

## 13. 성능 최적화 체크리스트

- **AsNoTracking**: 읽기 전용 쿼리  
- **Include 최소화** + 필요한 속성만 `Select`  
- **SplitQuery** vs SingleQuery 튜닝
```csharp
_db.Blogs.Include(b => b.Posts).AsSplitQuery();
```
- **Compiled Queries**
```csharp
private static readonly Func<AppDbContext,int,Task<Post?>> GetPostById =
    EF.CompileAsyncQuery((AppDbContext db, int id) => db.Posts.FirstOrDefault(p => p.Id == id));
```
- **Context Pooling**: `AddDbContextPool`  
- 적절한 **인덱스 설계**, 배치 쓰기(수백~수천 단위로 SaveChanges)

---

## 14. DbContext 수명과 DI

- DbContext는 **Scoped**가 표준(요청당 1개)  
- Singleton에서 DbContext 직접 주입 금지(스코프 불일치). 필요 시 `IServiceScopeFactory`로 스코프 생성  
- 백그라운드 작업(HostedService)에서도 스코프 분리 후 사용

```csharp
public class MySingleton
{
    private readonly IServiceScopeFactory _scopeFactory;
    public MySingleton(IServiceScopeFactory scopeFactory) => _scopeFactory = scopeFactory;

    public async Task WorkAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // ...
        await db.SaveChangesAsync();
    }
}
```

---

## 15. 테스트 전략 — InMemory/Sqlite/Testcontainers

- **InMemory**: 빠르지만 관계/제약 검증 한계
```csharp
builder.Services.AddDbContext<AppDbContext>(o => o.UseInMemoryDatabase("test"));
```

- **SQLite InMemory**: 제약/인덱스 검증에 유리
```csharp
var keep = new SqliteConnection("DataSource=:memory:");
keep.Open();
builder.Services.AddDbContext<AppDbContext>(o => o.UseSqlite(keep));
```

- **실 DB 컨테이너(Postgres/SQL Server)**: 통합 테스트 신뢰도 최고

---

## 16. 트러블슈팅 FAQ

- **마이그레이션이 안 생김**: `DbContext`가 DI에 등록됐는지, 생성자/Provider 확인  
- **스냅샷 충돌**: 수동 편집 주의. 필요 시 마지막 마이그레이션 제거 후 재생성  
- **N+1 성능저하**: `Include`/프로젝션/로딩 전략 재점검  
- **잠금/타임아웃**: 인덱스 튜닝, 트랜잭션 범위 축소, `.CommandTimeout()` 조정  
- **민감데이터 로깅**: 운영 환경에서 `EnableSensitiveDataLogging(false)`

---

## 17. 실전 예제 — Minimal API + EF Core

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContextPool<AppDbContext>(o => o.UseSqlite("Data Source=blog.db"));
var app = builder.Build();

app.MapGet("/blogs", async (AppDbContext db) =>
    await db.Blogs.AsNoTracking()
        .Select(b => new { b.Id, b.Title, Count = b.Posts.Count })
        .ToListAsync());

app.MapPost("/blogs", async (AppDbContext db, Blog input) =>
{
    db.Blogs.Add(input);
    await db.SaveChangesAsync();
    return Results.Created($"/blogs/{input.Id}", input);
});

await using (var scope = app.Services.CreateAsyncScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();
}
app.Run();
```

---

## 18. 요약

| 단계 | 핵심 |
|---|---|
| 설치 | Provider + Tools + Design 패키지 |
| 모델 | POCO(Entity) + Fluent API 구성 |
| DbContext | `DbSet<T>` 노출, 관계/인덱스/제약 |
| 등록 | `AddDbContext`/`AddDbContextPool` + `UseXxx` |
| 마이그레이션 | `add` → `update` → `script`/`bundle` |
| LINQ | 타입 안전 질의, `AsNoTracking` 적극 활용 |
| 성능 | Include 최소화/프로젝션/Compiled/Pooling |
| 테스트 | InMemory/SQLite/실 DB 컨테이너 조합 |

---

## 부록: 간단한 수학적 직관 — 변경 추적(Delta)

엔티티의 변경은 **현재 스냅샷**과 **원본 스냅샷**의 차이(델타)로 판단할 수 있다.  
속성 벡터를 $$ \mathbf{x} = (x_1,\dots,x_n) $$, 원본을 $$ \mathbf{x}_0 $$이라 하면  
변경량은 $$ \Delta \mathbf{x} = \mathbf{x} - \mathbf{x}_0 $$.  
EF Core는 내부적으로 이 $$ \Delta \mathbf{x} $$가 0인지 여부로 수정/삽입/삭제를 결정한다(개념적 직관).

---

# 다음 추천 주제
- 관계 매핑 심화(일대일/다대다/고급 키 매핑)
- Lazy vs Eager vs Explicit 로딩 사례 비교
- Seed 전략(고정 시드 vs 런타임 시드)와 환경별 분기
- 마이그레이션 운영 전략(증분 스크립트/번들/제로 다운타임)
- Raw SQL과 저장 프로시저, 뷰 통합 전략