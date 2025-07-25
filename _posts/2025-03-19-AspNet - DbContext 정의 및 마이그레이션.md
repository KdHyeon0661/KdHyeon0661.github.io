---
layout: post
title: AspNet - DbContext 정의 및 마이그레이션
date: 2025-03-19 20:20:23 +0900
category: AspNet
---
# 🧱 EF Core - DbContext 정의 및 마이그레이션 완전 정복

---

## 📌 DbContext란?

`DbContext`는 **EF Core에서 데이터베이스와 애플리케이션 간의 연결 고리**입니다.

- 테이블은 `DbSet<TEntity>`로 표현됨
- LINQ로 쿼리하고, `SaveChanges()`로 DB에 반영
- 실제 DB 연결과 ORM 내부 동작을 담당하는 중추 클래스

---

## 📄 1. 모델 클래스 정의 (Entity)

```csharp
public class Blog
{
    public int Id { get; set; }              // PK
    public string Title { get; set; }        // 문자열
    public string Author { get; set; }       // 작성자
    public DateTime CreatedAt { get; set; }  // 생성일
}
```

→ 이 클래스 하나가 곧 **데이터베이스의 테이블**이 됨

---

## 📄 2. DbContext 클래스 정의

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options)
    {
    }

    public DbSet<Blog> Blogs { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 테이블 이름 지정 (선택)
        modelBuilder.Entity<Blog>().ToTable("Blogs");
        
        // Fluent API로 속성 제약 설정 (선택)
        modelBuilder.Entity<Blog>().Property(b => b.Title).HasMaxLength(100);
    }
}
```

- `DbSet<Blog>` → `Blogs` 테이블을 의미
- `OnModelCreating()`에서 세부 설정 가능

---

## ⚙️ 3. Program.cs에 DbContext 등록

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

- 연결 문자열은 `appsettings.json`에 설정

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=BlogDb;Trusted_Connection=True;"
  }
}
```

---

## 🔁 마이그레이션(Migration)이란?

> 마이그레이션은 **EF Core가 모델 변경을 감지하고** 이에 맞춰 **DB 스키마를 자동 생성/변경하는 기능**입니다.

### 마이그레이션 흐름

```text
모델 클래스 변경 → dotnet ef migrations add → 마이그레이션 파일 생성
     ↓
dotnet ef database update → 실제 DB에 반영
```

---

## 🧪 4. 마이그레이션 명령어

### 📦 EF CLI 도구 설치 (최초 1회)

```bash
dotnet tool install --global dotnet-ef
```

> 이미 설치했으면 건너뛰어도 됨

---

### 🔹 마이그레이션 생성

```bash
dotnet ef migrations add InitialCreate
```

- `Migrations` 폴더가 생성되고, 클래스 파일이 생김
- `InitialCreate`는 마이그레이션 이름 (자유롭게 지정 가능)

---

### 🔹 DB 생성 및 적용

```bash
dotnet ef database update
```

- 실제로 **BlogDb**라는 데이터베이스가 생성됨
- 모델에 기반한 테이블이 자동 생성됨

---

## 📄 마이그레이션 폴더 구조

| 파일명 | 역할 |
|--------|------|
| `YYYYMMDD_HHMM_InitialCreate.cs` | 마이그레이션 본문 (Up/Down 메서드 포함) |
| `ModelSnapshot.cs` | 현재 DB 모델 상태 스냅샷 (EF가 변경 감지에 사용) |

---

## 🔄 마이그레이션 수정 흐름 예시

### 1. 모델 수정

```csharp
public string Content { get; set; }  // 속성 추가
```

### 2. 새 마이그레이션 생성

```bash
dotnet ef migrations add AddContentToBlog
```

### 3. 변경 DB 반영

```bash
dotnet ef database update
```

---

## 🗑 마이그레이션 삭제 / 롤백

- 마지막 마이그레이션 삭제:

```bash
dotnet ef migrations remove
```

- 특정 마이그레이션으로 롤백:

```bash
dotnet ef database update [MigrationName]
```

> 예: `dotnet ef database update InitialCreate`

---

## 🧪 예제: Blog 생성 및 저장

```csharp
public class BlogService
{
    private readonly AppDbContext _context;

    public BlogService(AppDbContext context)
    {
        _context = context;
    }

    public void AddBlog(string title, string author)
    {
        var blog = new Blog
        {
            Title = title,
            Author = author,
            CreatedAt = DateTime.Now
        };

        _context.Blogs.Add(blog);
        _context.SaveChanges(); // DB에 반영
    }
}
```

---

## ✅ 마무리 요약

| 항목 | 설명 |
|------|------|
| `DbContext` | EF Core의 핵심 클래스, 테이블과 매핑 |
| `DbSet<T>` | 특정 엔티티와 연결된 테이블 |
| `OnModelCreating` | Fluent API 구성 (제약조건, 이름 등) |
| `dotnet ef migrations add` | 마이그레이션 생성 |
| `dotnet ef database update` | 실제 DB 반영 |
| 마이그레이션 파일 | 변경 기록, 롤백 기능 제공 |
