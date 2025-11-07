---
layout: post
title: AspNet - DbContext 정의 및 마이그레이션
date: 2025-03-19 20:20:23 +0900
category: AspNet
---
# EF Core - DbContext 정의 및 마이그레이션

## 0) 큰 그림: EF Core로 보는 애플리케이션-DB 상호작용

1. **엔티티**(모델 클래스) 작성  
2. **DbContext**에 `DbSet<TEntity>` 노출 + `OnModelCreating`에서 구성  
3. **DI 컨테이너**에 `DbContext` 등록 (연결 문자열, 공급자 선택)  
4. **마이그레이션**으로 스키마 버전 관리 (`add` → `update`)  
5. 서비스/핸들러에서 `DbContext` 사용 → LINQ 쿼리/추가/수정/삭제 → `SaveChanges()` 반영

---

## 1) 모델 클래스(Entity) 정의

초안의 `Blog`를 확장해 **제약/탐색 속성/인덱스 후보**를 포함한다.

```csharp
public class Blog
{
    public int Id { get; set; }                          // PK(규칙: <Type> Id 또는 <Type>Id)
    public string Title { get; set; } = default!;
    public string Author { get; set; } = default!;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    // 일대다: Blog 1 - Post N
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}

public class Post
{
    public int Id { get; set; }
    public int BlogId { get; set; }                      // FK
    public string Slug { get; set; } = default!;
    public string Content { get; set; } = default!;
    public DateTime PublishedAt { get; set; }

    // 탐색 속성
    public Blog Blog { get; set; } = default!;
}
```

**포인트**
- 기본 관례(Convention)로도 PK/FK를 대부분 인식.
- 관계형 설계(일대다, 다대다, 일대일)를 탐색 속성으로 표현.

---

## 2) DbContext 클래스 정의

초안에서 **Fluent API**, **인덱스**, **제약**, **고급 구성**까지 확장한다.

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) {}

    public DbSet<Blog> Blogs => Set<Blog>();
    public DbSet<Post> Posts => Set<Post>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 테이블명
        modelBuilder.Entity<Blog>().ToTable("Blogs");
        modelBuilder.Entity<Post>().ToTable("Posts");

        // Blog 구성
        modelBuilder.Entity<Blog>(b =>
        {
            b.HasKey(x => x.Id);
            b.Property(x => x.Title).HasMaxLength(100).IsRequired();
            b.Property(x => x.Author).HasMaxLength(60).IsRequired();
            b.Property(x => x.CreatedAt).HasDefaultValueSql("CURRENT_TIMESTAMP");

            // 인덱스(Author + CreatedAt 복합)
            b.HasIndex(x => new { x.Author, x.CreatedAt });
        });

        // Post 구성
        modelBuilder.Entity<Post>(p =>
        {
            p.HasKey(x => x.Id);
            p.Property(x => x.Slug).HasMaxLength(120).IsRequired();
            p.Property(x => x.Content).IsRequired();

            // 유니크 인덱스(Slug)
            p.HasIndex(x => x.Slug).IsUnique();

            // 관계: Blog 1 - N Post
            p.HasOne(x => x.Blog)
             .WithMany(x => x.Posts)
             .HasForeignKey(x => x.BlogId)
             .OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

**포인트**
- **Fluent API**가 데이터 주석보다 우선한다(충돌 시 Fluent API 적용).
- `HasIndex`, `IsUnique`, `OnDelete` 등의 세부 제어는 운영 품질에 직결.

---

## 3) Program.cs 구성과 연결 문자열

초안의 SQL Server 예시에 더해 **Provider 별 패턴**을 한 번에 보여준다.

```csharp
var builder = WebApplication.CreateBuilder(args);

// 1) SQL Server
// builder.Services.AddDbContext<AppDbContext>(opt =>
//     opt.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// 2) SQLite (개발/데모에 간편)
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlite(builder.Configuration.GetConnectionString("SqliteConnection")));

// 3) PostgreSQL
// builder.Services.AddDbContext<AppDbContext>(opt =>
//     opt.UseNpgsql(builder.Configuration.GetConnectionString("PostgresConnection")));

// 4) MySQL/MariaDB
// builder.Services.AddDbContext<AppDbContext>(opt =>
//     opt.UseMySql(builder.Configuration.GetConnectionString("MySqlConnection"),
//                  ServerVersion.AutoDetect(builder.Configuration.GetConnectionString("MySqlConnection"))));

// 5) 연결 회복/재시도(클라우드 네트워크 불안 대비)
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlite(builder.Configuration.GetConnectionString("SqliteConnection"))
       .EnableSensitiveDataLogging(builder.Environment.IsDevelopment())
       .EnableDetailedErrors());

// 컨텍스트 풀링(고QPS API에서 GC/할당 삭감)
builder.Services.AddDbContextPool<AppDbContext>(opt =>
    opt.UseSqlite(builder.Configuration.GetConnectionString("SqliteConnection")));

var app = builder.Build();
app.MapGet("/", (AppDbContext db) => db.Blogs.Count());
app.Run();
```

`appsettings.json` 예시(다중 Provider 대비):

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=BlogDb;Trusted_Connection=True;MultipleActiveResultSets=True",
    "SqliteConnection": "Data Source=blog.db",
    "PostgresConnection": "Host=localhost;Database=blogdb;Username=postgres;Password=postgres"
  }
}
```

---

## 4) 마이그레이션(Migration) 총정리

### 4.1 dotnet-ef 도구 설치 (최초 1회)
```bash
dotnet tool install --global dotnet-ef
```

### 4.2 초기 마이그레이션 생성
```bash
dotnet ef migrations add InitialCreate
```

생성물:
- `Migrations/<timestamp>_InitialCreate.cs` : `Up/Down` 메서드 포함
- `Migrations/AppDbContextModelSnapshot.cs` : 현재 모델 스냅샷

### 4.3 DB 반영
```bash
dotnet ef database update
```

### 4.4 모델 변경 → 새 마이그레이션 → 업데이트
```bash
# 예) Post에 Summary 속성 추가
dotnet ef migrations add AddSummaryToPost
dotnet ef database update
```

### 4.5 롤백/제거
```bash
# 특정 지점으로 DB 되돌리기
dotnet ef database update InitialCreate

# 마지막 마이그레이션 제거(스냅샷/코드 되돌림, DB는 영향 없음)
dotnet ef migrations remove
```

### 4.6 스크립트 추출(운영 배포)
```bash
# 전체 스키마 생성 스크립트
dotnet ef migrations script -o ./sql/000_full.sql

# 특정 버전 ~ 특정 버전 간 증분 스크립트
dotnet ef migrations script 20240101120000_InitialCreate 20240210110000_AddSummaryToPost -o ./sql/010_delta.sql
```

### 4.7 Migration Bundle(.NET 7+)
로컬 SDK 없이 서버에서 실행 가능한 **단일 실행 파일**로 마이그레이션을 번들링
```bash
dotnet ef migrations bundle --configuration Release --self-contained
./efbundle --connection "Data Source=blog.db"
```

---

## 5) 마이그레이션 파일 anatomy

예시(일부):

```csharp
public partial class InitialCreate : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Blogs",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false)
                           .Annotation("Sqlite:Autoincrement", true),
                Title = table.Column<string>(maxLength: 100, nullable: false),
                Author = table.Column<string>(maxLength: 60, nullable: false),
                CreatedAt = table.Column<DateTime>(nullable: false, defaultValueSql: "CURRENT_TIMESTAMP")
            },
            constraints: table => { table.PrimaryKey("PK_Blogs", x => x.Id); });

        migrationBuilder.CreateTable(
            name: "Posts",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false)
                          .Annotation("Sqlite:Autoincrement", true),
                BlogId = table.Column<int>(nullable: false),
                Slug = table.Column<string>(maxLength: 120, nullable: false),
                Content = table.Column<string>(nullable: false),
                PublishedAt = table.Column<DateTime>(nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Posts", x => x.Id);
                table.ForeignKey(
                    name: "FK_Posts_Blogs_BlogId",
                    column: x => x.BlogId,
                    principalTable: "Blogs",
                    principalColumn: "Id",
                    onDelete: ReferentialAction.Cascade);
            });

        migrationBuilder.CreateIndex(
            name: "IX_Posts_Slug",
            table: "Posts",
            column: "Slug",
            unique: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable("Posts");
        migrationBuilder.DropTable("Blogs");
    }
}
```

**포인트**
- `Up`: 적용시 동작, `Down`: 롤백 동작
- 수동 수정으로 정교한 마이그레이션 가능(기존 데이터 마이그레이션/시드 스크립트 등 포함)

---

## 6) 데이터 시드(Seed)와 초기화

### 6.1 Fluent API 시드 (마이그레이션 기반)
```csharp
modelBuilder.Entity<Blog>().HasData(
    new Blog { Id = 1, Title = "EF Core 101", Author = "kim", CreatedAt = DateTime.UtcNow },
    new Blog { Id = 2, Title = "LINQ Tips",  Author = "lee", CreatedAt = DateTime.UtcNow }
);

modelBuilder.Entity<Post>().HasData(
    new Post { Id = 1, BlogId = 1, Slug = "hello-ef", Content = "Start EF Core", PublishedAt = DateTime.UtcNow }
);
```
- `HasData`는 마이그레이션에 삽입 SQL이 포함된다.

### 6.2 런타임 시드(앱 시작 시)
```csharp
public static class DbInitializer
{
    public static async Task SeedAsync(IServiceProvider sp)
    {
        using var scope = sp.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();

        if (!db.Blogs.Any())
        {
            db.Blogs.Add(new Blog { Title = "Getting Started", Author = "admin" });
            await db.SaveChangesAsync();
        }
    }
}

// Program.cs
await DbInitializer.SeedAsync(app.Services);
```
- 마이그레이션/시드 동시 처리로 로컬 개발 생산성 향상.

---

## 7) 실제 사용: CRUD, 관계, 로딩 전략

### 7.1 추가/수정/삭제
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

### 7.2 로딩 전략
- **즉시 로딩(Eager)**: `Include`  
- **명시적 로딩(Explicit)**: `Entry(...).Collection(...).LoadAsync()`  
- **지연 로딩(Lazy)**: 프록시 사용(성능/예상치 못한 N+1 주의)

```csharp
var blogs = await _db.Blogs
    .Include(b => b.Posts) // Eager
    .OrderByDescending(b => b.CreatedAt)
    .ToListAsync();
```

명시적 로딩:
```csharp
var blog = await _db.Blogs.FirstAsync();
await _db.Entry(blog).Collection(b => b.Posts).LoadAsync();
```

### 7.3 추적 전략
- 기본은 **추적(Tracking)**. 읽기 전용 쿼리는 `AsNoTracking()`으로 **변경 추적 오버헤드 제거**.
```csharp
var list = await _db.Blogs.AsNoTracking().ToListAsync();
```

---

## 8) 고급 구성: 소유 타입(Owned), 값 변환기(ValueConverter), 그림자 속성(Shadow), 동시성 토큰

### 8.1 Owned 타입(복합 값 객체)
```csharp
public class Audit
{
    public string CreatedBy { get; set; } = default!;
    public DateTime CreatedAt { get; set; }
}

public class Comment
{
    public int Id { get; set; }
    public int PostId { get; set; }
    public Audit Audit { get; set; } = new();
    public string Body { get; set; } = default!;
}

protected override void OnModelCreating(ModelBuilder mb)
{
    mb.Entity<Comment>(e =>
    {
        e.OwnsOne(c => c.Audit, a =>
        {
            a.Property(x => x.CreatedBy).HasMaxLength(40);
        });
    });
}
```

### 8.2 ValueConverter(예: enum/flags/string 압축 저장)
```csharp
public enum Visibility { Private, Unlisted, Public }

public class Note
{
    public int Id { get; set; }
    public Visibility Visibility { get; set; }
}

protected override void OnModelCreating(ModelBuilder mb)
{
    var conv = new ValueConverter<Visibility, string>(
        v => v.ToString(), s => Enum.Parse<Visibility>(s));

    mb.Entity<Note>()
      .Property(n => n.Visibility)
      .HasConversion(conv)
      .HasMaxLength(12);
}
```

### 8.3 Shadow Property(모델에 없는 컬럼)
```csharp
protected override void OnModelCreating(ModelBuilder mb)
{
    mb.Entity<Post>().Property<DateTime>("LastModified").HasDefaultValueSql("CURRENT_TIMESTAMP");
}
```
사용:
```csharp
_db.Entry(post).Property("LastModified").CurrentValue = DateTime.UtcNow;
```

### 8.4 동시성 제어(낙관적 Lock)
```csharp
public class Inventory
{
    public int Id { get; set; }
    public string Sku { get; set; } = default!;
    [Timestamp] public byte[] RowVersion { get; set; } = default!;
}
```
갱신 시:
```csharp
try
{
    await _db.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    // 충돌 감지 → 병합/재시도 전략
}
```

---

## 9) 트랜잭션/유닛오브워크/원자성

기본적으로 `SaveChanges()`는 트랜잭션을 포함한다. 여러 `DbContext` 또는 외부 리소스 동시 작업 시 명시적 트랜잭션 사용.

```csharp
using var tx = await _db.Database.BeginTransactionAsync();
try
{
    // 작업 1
    // 작업 2
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

## 10) Raw SQL 과 안전한 파라미터화

### 10.1 읽기 쿼리
```csharp
var recent = await _db.Posts
    .FromSqlInterpolated($"SELECT * FROM Posts WHERE PublishedAt > {DateTime.UtcNow.AddDays(-7)}")
    .AsNoTracking()
    .ToListAsync();
```

### 10.2 비-엔티티 결과(프로젝션)
```csharp
public record PostSummary(int Id, string Slug);

var rows = await _db.Database
    .SqlQuery<PostSummary>($"SELECT Id, Slug FROM Posts WHERE BlogId = {blogId}")
    .ToListAsync();
```

**주의**: `FromSqlRaw` 사용 시 문자열 결합 금지. **반드시 파라미터화** 사용.

---

## 11) 성능 체크리스트

- **컨텍스트 풀링**(`AddDbContextPool`)로 할당/GC 감축  
- **읽기 쿼리 `AsNoTracking()`**  
- 필요한 경우 **SplitQuery** vs **SingleQuery** 밸런스 조정
```csharp
_db.Blogs.Include(b => b.Posts).AsSplitQuery();
```
- **Compiled Query**로 고정 패턴 성능 최적화
```csharp
private static readonly Func<AppDbContext, int, Task<Post?>> _getPostById =
    EF.CompileAsyncQuery((AppDbContext db, int id) => db.Posts.FirstOrDefault(p => p.Id == id));
```
- 적절한 **인덱스 설계**, **로딩 전략**(N+1 방지)
- 대용량 쓰기: **Batch 단위**로 저장(예: 500~2000개씩)

---

## 12) 설계/운영: 다중 DbContext, 다중 마이그레이션, 스키마 분리

- **다중 컨텍스트**: Bounded Context 별 분리(`AppDbContext`, `ReportingDbContext` 등)
- **마이그레이션 세트 분리**: `--context` 옵션으로 컨텍스트별 마이그레이션 폴더 관리
```bash
dotnet ef migrations add InitReporting --context ReportingDbContext --output-dir Migrations/Reporting
```
- 멀티테넌시/스키마: `ToTable("Blogs", schema: "tenant1")`

---

## 13) 테스트: InMemory/Sqlite/컨테이너

### 13.1 InMemory 기능 테스트(관계/제약이 약함)
```csharp
builder.Services.AddDbContext<AppDbContext>(opt => opt.UseInMemoryDatabase("test"));
```

### 13.2 Sqlite In-Memory(제약/인덱스 검증에 유리)
```csharp
var keepAlive = new SqliteConnection("DataSource=:memory:");
keepAlive.Open();
builder.Services.AddDbContext<AppDbContext>(opt => opt.UseSqlite(keepAlive));
```

### 13.3 Testcontainers로 진짜 DB 통합 테스트
- PostgreSQL/MSSQL Docker 컨테이너 기동 → 실제 동작 검증

---

## 14) 보안/탄력성

- **연결 재시도**, **명령 타임아웃** 설정
```csharp
opt.UseSqlServer(conn, sql => sql.EnableRetryOnFailure(5, TimeSpan.FromSeconds(10), null)
                                 .CommandTimeout(30));
```
- **민감 데이터 로깅 금지**(개발 외 비활성)
- **마이그레이션 스크립트 검토** 후 운영 반영(변경 영향/락 시간 고려)
- 대규모 스키마 변경 시 **Online Index**, **Zero-downtime** 전략(청크/뷰 교체/드리프트 방지)

---

## 15) 종합 예제: API + EF Core

### 15.1 DTO/요청 검증
```csharp
public record CreatePostRequest(int BlogId, string Slug, string Content);

public static class PostEndpoints
{
    public static void MapPostEndpoints(this IEndpointRouteBuilder app)
    {
        app.MapPost("/posts", async (CreatePostRequest req, AppDbContext db) =>
        {
            var exists = await db.Blogs.AnyAsync(b => b.Id == req.BlogId);
            if (!exists) return Results.NotFound("Blog not found");

            db.Posts.Add(new Post
            {
                BlogId = req.BlogId, Slug = req.Slug, Content = req.Content, PublishedAt = DateTime.UtcNow
            });
            await db.SaveChangesAsync();
            return Results.Created($"/posts/{req.Slug}", req);
        });

        app.MapGet("/blogs/{id:int}", async (int id, AppDbContext db) =>
        {
            var blog = await db.Blogs
                .AsNoTracking()
                .Include(b => b.Posts.OrderByDescending(p => p.PublishedAt))
                .FirstOrDefaultAsync(b => b.Id == id);
            return blog is null ? Results.NotFound() : Results.Ok(blog);
        });
    }
}
```

### 15.2 Program.cs
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContextPool<AppDbContext>(opt => opt.UseSqlite("Data Source=blog.db"));
var app = builder.Build();

app.MapGet("/", () => "EF Core Sample");
app.MapPostEndpoints();

await using (var scope = app.Services.CreateAsyncScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();
}
app.Run();
```

---

## 16) 트러블슈팅 FAQ

- **마이그레이션 생성 안 됨**: 올바른 `Startup/Program` 구성/`DbContext` 생성자/Provider 체크  
- **스냅샷 충돌**: 수동 편집 시 주의. 필요 시 `remove` 후 재생성  
- **인덱스/외래키 이름 충돌**: Fluent API에서 명시적 이름 지정  
- **N+1 느림**: `Include`/프로젝션/쿼리 재구성  
- **잠금 경합**: 인덱스 튜닝/트랜잭션 범위 최소화/쓰기 배치화  
- **타임아웃**: `.CommandTimeout()`/쿼리 계획 점검

---

## 17) 요약 표

| 항목 | 키 포인트 |
|------|-----------|
| DbContext | `DbSet<TEntity>` 노출 + `OnModelCreating`에서 Fluent 구성 |
| 등록/Provider | SqlServer/Sqlite/Postgres/MySQL 등 선택, 재시도/풀링 |
| 마이그레이션 | `add` → `update` → `script`/`bundle` 로 운영 반영 |
| 관계/로딩 | `Include`/명시적/지연 로딩, N+1 방지, 인덱스 설계 |
| 고급 구성 | Owned, ValueConverter, Shadow, 동시성 토큰 |
| 성능 | AsNoTracking, SplitQuery, Compiled Query, 풀링 |
| 테스트 | InMemory/Sqlite/컨테이너, 시드/마이그레이션 자동화 |
| 배포 | SQL 스크립트/번들, 제로 다운타임 고려 |

---

## 부록 A. 수학적 관점의 낙관적 동시성(간단 표기)

동시성 충돌은 간단히 **버전 벡터 비교**로 생각할 수 있다. 레코드의 버전을 $$ v \in \mathbb{N} $$ 로 두고, 트랜잭션 시나리오를

- 읽기 시점 버전: $$ v_r $$
- 커밋 시점 현재 버전: $$ v_c $$

이라고 할 때, **성공 조건**은 $$ v_r = v_c $$. 실패 시 $$ v_c \ne v_r $$ 이므로 충돌 예외를 발생시키고 병합 또는 재시도를 수행한다. EF Core의 `[Timestamp]`/RowVersion이 바로 이 $$ v $$를 담당한다.

---

# 마무리

- 초안의 **핵심(모델 → DbContext → Program.cs → 마이그레이션 명령)**을 기반으로, 관계/인덱스/시드/동시성/성능/운영 배포까지 실무에 필요한 부분을 총망라했다.  
- 여기의 패턴을 토대로 **테스트 가능한 구조**와 **안전한 마이그레이션 파이프라인**을 갖추면, 개발-운영 전 과정이 수월해진다.