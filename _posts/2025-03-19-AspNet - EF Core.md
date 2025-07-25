---
layout: post
title: AspNet - EF Core
date: 2025-03-19 20:20:23 +0900
category: AspNet
---
# 🗂️ Entity Framework Core (EF Core) 소개 및 설치

## 📌 EF Core란?

Entity Framework Core는 **Microsoft에서 개발한 ORM(Object-Relational Mapper)**으로,  
객체 지향적인 방식으로 데이터베이스와 상호작용할 수 있게 해주는 도구입니다.

SQL 쿼리를 직접 작성하지 않고도 C# 객체로 데이터 CRUD 작업을 수행할 수 있어,  
생산성을 높이고 유지보수를 용이하게 만들어 줍니다.

---

## ✅ EF Core의 주요 특징

| 기능 | 설명 |
|------|------|
| ORM | 객체 ↔ 테이블 간 매핑 |
| LINQ 지원 | SQL 대신 C# LINQ 사용 가능 |
| 마이그레이션 | DB 스키마 버전 관리 가능 |
| 다중 DB 지원 | SQL Server, SQLite, PostgreSQL, MySQL 등 |
| NoSQL 일부 지원 | Azure Cosmos DB 등 |
| 추적 / 변경 감지 | 객체의 변경 상태 자동 추적 |
| Lazy/Eager Loading | 관계 데이터 불러오기 전략 선택 가능 |

---

## 🔌 지원 데이터베이스

- ✅ Microsoft SQL Server
- ✅ SQLite
- ✅ PostgreSQL (via Npgsql)
- ✅ MySQL (via Pomelo)
- ✅ Oracle (비공식)
- ✅ Azure Cosmos DB

---

## 🛠️ EF Core 설치 (ASP.NET Core 기준)

### 1️⃣ 패키지 설치

#### ✅ SQL Server 기준

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

#### ✅ PostgreSQL 사용 시

```bash
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

#### ✅ SQLite 사용 시

```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
```

#### ✅ 기타 (디자인/마이그레이션 도구)

```bash
dotnet add package Microsoft.EntityFrameworkCore.Design
```

---

## 🧱 프로젝트에 DbContext 등록하기

### 📄 1. 모델 클래스 생성

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }
}
```

---

### 📄 2. DbContext 클래스 정의

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }

    public DbSet<Blog> Blogs { get; set; }
}
```

---

### 📄 3. Program.cs에 DbContext 등록

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

---

### 📄 4. appsettings.json에 연결 문자열 작성

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=MyDb;Trusted_Connection=True;"
  }
}
```

> 로컬 테스트용으로는 `SQLite` 또는 `LocalDB`, 실전에서는 `SQL Server`, `PostgreSQL` 등을 주로 사용함

---

## 🔁 마이그레이션 사용법

### 1. 초기 마이그레이션 생성

```bash
dotnet ef migrations add InitialCreate
```

→ `Migrations` 폴더에 DB 스키마에 해당하는 클래스 파일이 생성됨

---

### 2. 실제 DB 생성

```bash
dotnet ef database update
```

→ 설정한 연결 문자열에 따라 DB가 자동 생성됨

---

## 🧪 LINQ 예시

```csharp
public class IndexModel : PageModel
{
    private readonly AppDbContext _db;

    public IndexModel(AppDbContext db)
    {
        _db = db;
    }

    public List<Blog> Blogs { get; set; }

    public void OnGet()
    {
        Blogs = _db.Blogs
            .Where(b => b.Title.Contains("ASP"))
            .OrderBy(b => b.Id)
            .ToList();
    }
}
```

---

## ✅ EF Core 설치 요약

| 단계 | 설명 |
|------|------|
| 패키지 설치 | `dotnet add package`로 EFCore 및 DB provider 설치 |
| 모델 생성 | POCO 클래스 생성 |
| DbContext 정의 | DbSet으로 테이블 구성 |
| 서비스 등록 | `AddDbContext` 및 `UseSqlServer` |
| 연결 문자열 구성 | `appsettings.json`에 등록 |
| 마이그레이션 수행 | `dotnet ef migrations`, `update` 명령어 |

---

## 🔜 다음 추천 주제

- ✅ 마이그레이션 관리 및 버전 롤백
- ✅ 관계 설정 (`One-to-Many`, `Many-to-Many`, `Foreign Key`)
- ✅ EF Core Lazy Loading vs Eager Loading
- ✅ Seed Data 및 테스트용 데이터 초기화 전략
- ✅ 트랜잭션 처리, Raw SQL 실행