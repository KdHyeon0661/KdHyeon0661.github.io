---
layout: post
title: WPF - Entity Framework Core + SQLite + REST API 연동
date: 2025-09-07 20:25:23 +0900
category: WPF
---
# 🧩 WPF에서 **Entity Framework Core + SQLite + REST API 연동** 완전 정복
*(예제 중심 · 누락 없이 자세하게 · MVVM/DI/동기화/오프라인/성능/테스트까지 한 번에)*

> 목표: **로컬 SQLite**를 **EF Core**로 다루고, **원격 REST API**와 동기화되는 **WPF(MVVM)** 앱을 만든다.  
> .NET 7/8 WPF 기준이며 .NET 6/Framework 4.8도 큰 틀은 동일합니다.

---

## 0) 데모 시나리오 (끝까지 이어질 예시)

- **도메인**: Todo(할 일)  
- **로컬 저장소**: `SQLite` (파일: `%LOCALAPPDATA%\TodoDemo\todo.db`)  
- **ORM**: `EF Core` (변경 추적/마이그레이션/동시성/관계/쿼리)  
- **API**: `https://api.example.com/todos` (CRUD + 페이지네이션 + 변경 감지(ETag))  
- **앱 특성**: 오프라인에서도 편집 → 온라인 복구 시 **양방향 동기화**, 충돌 해결

---

## 1) 프로젝트 구조

```
TodoDemo/
  TodoDemo.App/                 # WPF UI (Views, ViewModels)
  TodoDemo.Domain/              # 엔티티/규칙/DTO/계약
  TodoDemo.Data/                # EF Core DbContext/마이그레이션/리포지토리
  TodoDemo.ApiClient/           # REST 클라이언트 (HttpClientFactory/Polly/DTO 매핑)
  TodoDemo.Sync/                # 동기화 서비스(오프라인/충돌/스케줄)
  TodoDemo.Tests/               # 단위/통합 테스트
```

---

## 2) 패키지 설치

```bash
# EF Core + SQLite
dotnet add TodoDemo.Data package Microsoft.EntityFrameworkCore
dotnet add TodoDemo.Data package Microsoft.EntityFrameworkCore.Sqlite
dotnet add TodoDemo.Data package Microsoft.EntityFrameworkCore.Design

# DI/호스팅 (WPF에서도 Generic Host 사용 권장)
dotnet add TodoDemo.App package Microsoft.Extensions.Hosting
dotnet add TodoDemo.App package Microsoft.Extensions.DependencyInjection

# Http + Polly(재시도/서킷브레이커)
dotnet add TodoDemo.ApiClient package Microsoft.Extensions.Http
dotnet add TodoDemo.ApiClient package Polly.Extensions.Http

# (선택) 매핑/검증
dotnet add TodoDemo.Domain package FluentValidation
dotnet add TodoDemo.App package CommunityToolkit.Mvvm
```

---

## 3) 도메인/엔티티 설계

### 3.1 엔티티 (동기화/동시성 필드 포함)
```csharp
// TodoDemo.Domain/Entities/Todo.cs
public class Todo
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Title { get; set; } = "";
    public bool IsDone { get; set; }
    public DateTimeOffset CreatedAt { get; set; } = DateTimeOffset.UtcNow;
    public DateTimeOffset? UpdatedAt { get; set; }

    // 동기화/충돌 해결용
    public DateTimeOffset LastModifiedUtc { get; set; } = DateTimeOffset.UtcNow;
    public string? ETag { get; set; }                 // 서버 리소스 버전
    public bool IsDirty { get; set; }                 // 로컬 변경 여부
    public bool IsDeleted { get; set; }               // 소프트 삭제(동기화 시 전파)
}
```

---

## 4) DbContext 구성

```csharp
// TodoDemo.Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using TodoDemo.Domain.Entities;

public class AppDbContext : DbContext
{
    public DbSet<Todo> Todos => Set<Todo>();

    public string DbPath { get; }

    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Todo>(e =>
        {
            e.HasKey(x => x.Id);
            e.Property(x => x.Title).IsRequired().HasMaxLength(200);
            e.Property(x => x.ETag).HasMaxLength(64);
            e.HasIndex(x => x.IsDirty);
            e.HasIndex(x => x.LastModifiedUtc);
            e.HasQueryFilter(x => !x.IsDeleted); // 소프트 삭제 글로벌 필터
        });
    }
}
```

### 4.1 SQLite 연결 문자열 & 폴더 준비
```csharp
// TodoDemo.App/App.xaml.cs (또는 Program.cs: WPF에 Generic Host 구성)
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

public partial class App : Application
{
    public static IHost HostApp { get; } = Microsoft.Extensions.Hosting.Host
        .CreateDefaultBuilder()
        .ConfigureServices((ctx, services) =>
        {
            string dbDir = Path.Combine(Environment.GetFolderPath(
                Environment.SpecialFolder.LocalApplicationData), "TodoDemo");
            Directory.CreateDirectory(dbDir);
            string dbPath = Path.Combine(dbDir, "todo.db");

            services.AddDbContext<AppDbContext>(opt =>
            {
                opt.UseSqlite($"Data Source={dbPath}");
                opt.EnableSensitiveDataLogging(false);
            });

            // 기타 서비스 등록 (아래 섹션들에서 채움)
        })
        .Build();

    protected override void OnStartup(StartupEventArgs e)
    {
        HostApp.Start();
        // 마이그레이션 자동 적용 (초기 실행/업데이트)
        using var scope = HostApp.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        db.Database.Migrate();

        base.OnStartup(e);
        new MainWindow { DataContext = HostApp.Services.GetRequiredService<MainViewModel>() }.Show();
    }

    protected override void OnExit(ExitEventArgs e)
    {
        HostApp.Dispose();
        base.OnExit(e);
    }
}
```

---

## 5) 마이그레이션

```bash
dotnet ef migrations add InitialCreate --project TodoDemo.Data --startup-project TodoDemo.App
dotnet ef database update --project TodoDemo.Data --startup-project TodoDemo.App
```

> **SQLite 제약**: 일부 `ALTER COLUMN`이 제한적. 스키마 변경은 **새 컬럼 추가 → 데이터 마이그레이션 → 구 컬럼 삭제** 패턴 사용.

---

## 6) 리포지토리/유닛오브워크 (선택)

EF Core 자체가 UoW를 제공하지만, 테스트 용이성과 API 인터페이스화를 위해 래핑해 봅니다.

```csharp
// TodoDemo.Data/ITodoRepository.cs
using System.Linq.Expressions;
public interface ITodoRepository
{
    Task<Todo?> GetAsync(Guid id, CancellationToken ct = default);
    Task<List<Todo>> GetAllAsync(Expression<Func<Todo, bool>>? pred = null, CancellationToken ct = default);
    Task AddAsync(Todo entity, CancellationToken ct = default);
    Task UpdateAsync(Todo entity, CancellationToken ct = default);
    Task SoftDeleteAsync(Guid id, CancellationToken ct = default);
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

```csharp
// TodoDemo.Data/TodoRepository.cs
public class TodoRepository : ITodoRepository
{
    private readonly AppDbContext _db;
    public TodoRepository(AppDbContext db) => _db = db;

    public Task<Todo?> GetAsync(Guid id, CancellationToken ct = default)
        => _db.Todos.AsNoTracking().FirstOrDefaultAsync(x => x.Id == id, ct);

    public async Task<List<Todo>> GetAllAsync(Expression<Func<Todo, bool>>? pred = null, CancellationToken ct = default)
    {
        var q = _db.Todos.AsNoTracking();
        if (pred != null) q = q.Where(pred);
        return await q.OrderByDescending(x => x.LastModifiedUtc).ToListAsync(ct);
    }

    public async Task AddAsync(Todo e, CancellationToken ct = default)
    {
        e.IsDirty = true; e.LastModifiedUtc = DateTimeOffset.UtcNow;
        await _db.Todos.AddAsync(e, ct);
    }

    public Task UpdateAsync(Todo e, CancellationToken ct = default)
    {
        e.IsDirty = true; e.LastModifiedUtc = DateTimeOffset.UtcNow;
        _db.Todos.Update(e);
        return Task.CompletedTask;
    }

    public async Task SoftDeleteAsync(Guid id, CancellationToken ct = default)
    {
        var entity = await _db.Todos.FirstOrDefaultAsync(x => x.Id == id, ct);
        if (entity is null) return;
        entity.IsDeleted = true; entity.IsDirty = true; entity.LastModifiedUtc = DateTimeOffset.UtcNow;
    }

    public Task<int> SaveChangesAsync(CancellationToken ct = default)
        => _db.SaveChangesAsync(ct);
}
```

DI 등록:
```csharp
services.AddScoped<ITodoRepository, TodoRepository>();
```

---

## 7) WPF(MVVM)와 EF Core 바인딩

### 7.1 ViewModel 기본 (CommunityToolkit.Mvvm)
```csharp
// TodoDemo.App/ViewModels/MainViewModel.cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Collections.ObjectModel;

public partial class MainViewModel : ObservableObject
{
    private readonly ITodoRepository _repo;

    public ObservableCollection<Todo> Items { get; } = new();

    [ObservableProperty] private string newTitle = "";

    public MainViewModel(ITodoRepository repo) => _repo = repo;

    [RelayCommand]
    private async Task LoadAsync()
    {
        Items.Clear();
        foreach (var t in await _repo.GetAllAsync())
            Items.Add(t);
    }

    [RelayCommand]
    private async Task AddAsync()
    {
        if (string.IsNullOrWhiteSpace(NewTitle)) return;
        var todo = new Todo { Title = NewTitle };
        await _repo.AddAsync(todo);
        await _repo.SaveChangesAsync();
        Items.Insert(0, todo);
        NewTitle = "";
    }

    [RelayCommand]
    private async Task ToggleDoneAsync(Todo item)
    {
        item.IsDone = !item.IsDone;
        await _repo.UpdateAsync(item);
        await _repo.SaveChangesAsync();
        // ObservableCollection에 같은 참조가 있으므로 UI가 갱신됨 (필요 시 OnPropertyChanged 호출)
    }
}
```

DI:
```csharp
services.AddSingleton<MainViewModel>();
```

### 7.2 View
```xml
<!-- TodoDemo.App/Views/MainWindow.xaml -->
<Window x:Class="TodoDemo.App.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        mc:Ignorable="d"
        Title="Todo" Width="560" Height="420">
  <DockPanel Margin="16">
    <StackPanel Orientation="Horizontal" DockPanel.Dock="Top" Spacing="8">
      <TextBox Width="320" Text="{Binding NewTitle, UpdateSourceTrigger=PropertyChanged}" />
      <Button Content="추가" Command="{Binding AddCommand}"/>
      <Button Content="새로고침" Command="{Binding LoadCommand}"/>
    </StackPanel>

    <ListView ItemsSource="{Binding Items}">
      <ListView.ItemTemplate>
        <DataTemplate DataType="{x:Type domain:Todo}">
          <StackPanel Orientation="Horizontal" Spacing="8">
            <CheckBox IsChecked="{Binding IsDone}" Command="{Binding DataContext.ToggleDoneCommand, RelativeSource={RelativeSource AncestorType=ListView}}" CommandParameter="{Binding}"/>
            <TextBlock Text="{Binding Title}" TextDecorations="{Binding IsDone, Converter={StaticResource BoolToStrikeConverter}}"/>
            <TextBlock Foreground="#888" Text="{Binding LastModifiedUtc}"/>
          </StackPanel>
        </DataTemplate>
      </ListView.ItemTemplate>
    </ListView>
  </DockPanel>
</Window>
```

> **팁**: 대량 목록 가상화 유지(`VirtualizingStackPanel.IsVirtualizing="True"`). EF 추적된 엔티티를 직접 바인딩할 때는 **변경 추적 범위** 관리(AsNoTracking로 조회 후 편집 시 Attach/Update).

---

## 8) REST API 클라이언트

### 8.1 DTO & 매퍼
```csharp
// TodoDemo.Domain/Dto/TodoDto.cs
public record TodoDto(Guid Id, string Title, bool IsDone, DateTimeOffset LastModifiedUtc, bool IsDeleted);

// 매핑 (간단 수동)
public static class TodoMapper
{
    public static TodoDto ToDto(Todo e) => new(e.Id, e.Title, e.IsDone, e.LastModifiedUtc, e.IsDeleted);
    public static void Apply(Todo e, TodoDto dto)
    {
        e.Title = dto.Title;
        e.IsDone = dto.IsDone;
        e.LastModifiedUtc = dto.LastModifiedUtc;
        e.IsDeleted = dto.IsDeleted;
    }
}
```

### 8.2 HttpClientFactory + Polly
```csharp
// TodoDemo.ApiClient/ServiceCollectionExtensions.cs
using Microsoft.Extensions.DependencyInjection;
using Polly;
using Polly.Extensions.Http;
using System.Net;
using System.Net.Http.Headers;

public static class ApiClientRegistration
{
    public static IServiceCollection AddTodoApi(this IServiceCollection services, string baseAddress)
    {
        services.AddHttpClient<ITodoApi, TodoApiClient>(c =>
        {
            c.BaseAddress = new Uri(baseAddress);
            c.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
            // 인증 토큰이 있다면 Authorization 헤더 추가
        })
        .AddPolicyHandler(GetRetryPolicy())
        .AddPolicyHandler(GetCircuitBreaker());

        return services;
    }

    private static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
        => HttpPolicyExtensions
            .HandleTransientHttpError()
            .OrResult(r => r.StatusCode == HttpStatusCode.TooManyRequests)
            .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromMilliseconds(200 * retryAttempt));

    private static IAsyncPolicy<HttpResponseMessage> GetCircuitBreaker()
        => HttpPolicyExtensions
            .HandleTransientHttpError()
            .CircuitBreakerAsync(5, TimeSpan.FromSeconds(20));
}
```

```csharp
// TodoDemo.ApiClient/ITodoApi.cs
public interface ITodoApi
{
    Task<(IReadOnlyList<TodoDto> Items, string? ETag)> GetPageAsync(int page, int size, string? ifNoneMatch = null, CancellationToken ct = default);
    Task<(TodoDto Item, string? ETag)> UpsertAsync(TodoDto dto, string? ifMatch, CancellationToken ct = default);
    Task DeleteAsync(Guid id, string? ifMatch, CancellationToken ct = default);
}
```

```csharp
// TodoDemo.ApiClient/TodoApiClient.cs
using System.Net.Http.Json;

public class TodoApiClient : ITodoApi
{
    private readonly HttpClient _http;
    public TodoApiClient(HttpClient http) => _http = http;

    public async Task<(IReadOnlyList<TodoDto>, string?)> GetPageAsync(int page, int size, string? ifNoneMatch = null, CancellationToken ct = default)
    {
        var req = new HttpRequestMessage(HttpMethod.Get, $"/todos?page={page}&size={size}");
        if (!string.IsNullOrEmpty(ifNoneMatch)) req.Headers.IfNoneMatch.ParseAdd(ifNoneMatch);

        var res = await _http.SendAsync(req, ct);
        if (res.StatusCode == System.Net.HttpStatusCode.NotModified)
            return (Array.Empty<TodoDto>(), res.Headers.ETag?.Tag);

        res.EnsureSuccessStatusCode();
        var list = await res.Content.ReadFromJsonAsync<List<TodoDto>>(cancellationToken: ct) ?? new();
        return (list, res.Headers.ETag?.Tag);
    }

    public async Task<(TodoDto, string?)> UpsertAsync(TodoDto dto, string? ifMatch, CancellationToken ct = default)
    {
        var req = new HttpRequestMessage(HttpMethod.Put, $"/todos/{dto.Id}")
        {
            Content = JsonContent.Create(dto)
        };
        if (!string.IsNullOrWhiteSpace(ifMatch))
            req.Headers.TryAddWithoutValidation("If-Match", ifMatch);

        var res = await _http.SendAsync(req, ct);
        res.EnsureSuccessStatusCode();
        var body = await res.Content.ReadFromJsonAsync<TodoDto>(cancellationToken: ct) ?? dto;
        return (body, res.Headers.ETag?.Tag);
    }

    public async Task DeleteAsync(Guid id, string? ifMatch, CancellationToken ct = default)
    {
        var req = new HttpRequestMessage(HttpMethod.Delete, $"/todos/{id}");
        if (!string.IsNullOrWhiteSpace(ifMatch))
            req.Headers.TryAddWithoutValidation("If-Match", ifMatch);

        var res = await _http.SendAsync(req, ct);
        res.EnsureSuccessStatusCode();
    }
}
```

DI:
```csharp
services.AddTodoApi("https://api.example.com");  // 앱 구성에서 BaseAddress 가져오기
```

---

## 9) 오프라인·동기화 서비스

### 9.1 Sync 전략(핵심 아이디어)
1) **푸시**: 로컬 `IsDirty==true` 항목 → 서버 Upsert/Delete (If-Match: ETag)  
2) **풀**: 서버 변경분 페이지 조회 (If-None-Match: ETag) → 로컬 병합  
3) **충돌**: 서버 ETag 불일치(412 Precondition Failed) → **정책**:  
   - *서버 우선*: 서버 버전을 받아 로컬에 덮어쓰기  
   - *클라이언트 우선*: 로컬 버전 재시도(Force) 또는 별도 충돌 컬렉션으로 사용자에게 선택 유도  
4) **삭제**: 소프트 삭제를 우선(복구 용이), 최종 정리 배치에서 하드 삭제

### 9.2 구현
```csharp
// TodoDemo.Sync/SyncService.cs
public class SyncService
{
    private readonly ITodoRepository _repo;
    private readonly ITodoApi _api;

    public SyncService(ITodoRepository repo, ITodoApi api)
    {
        _repo = repo; _api = api;
    }

    public async Task PushAsync(CancellationToken ct = default)
    {
        var dirty = await _repo.GetAllAsync(x => x.IsDirty || x.IsDeleted, ct);
        foreach (var e in dirty)
        {
            try
            {
                if (e.IsDeleted)
                {
                    await _api.DeleteAsync(e.Id, e.ETag, ct);
                    // 로컬에서도 완전 삭제(옵션) 또는 Tombstone 유지
                }
                else
                {
                    var (remote, etag) = await _api.UpsertAsync(TodoMapper.ToDto(e), e.ETag, ct);
                    TodoMapper.Apply(e, remote);
                    e.ETag = etag;
                    e.IsDirty = false;
                }
            }
            catch (HttpRequestException) { throw; }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                // 412 등 충돌 처리 정책
                // 간단: 서버 우선
                // 실제로는 에러 유형별로 분기
            }
        }
        await _repo.SaveChangesAsync(ct);
    }

    public async Task PullAsync(CancellationToken ct = default)
    {
        string? etag = null; // 앱 설정/로컬에 마지막 ETag 저장
        int page = 1, size = 100;
        while (true)
        {
            var (items, newTag) = await _api.GetPageAsync(page, size, etag, ct);
            if (items.Count == 0) { /* NotModified or end */ break; }

            foreach (var dto in items)
            {
                var local = await _repo.GetAsync(dto.Id, ct);
                if (local is null)
                {
                    local = new Todo { Id = dto.Id };
                    TodoMapper.Apply(local, dto);
                    local.ETag = newTag ?? local.ETag;
                    local.IsDirty = false;
                    await _repo.AddAsync(local, ct);
                }
                else
                {
                    // 로컬이 Dirty면 충돌 정책 필요. 여기서는 서버 우선 덮어쓰기
                    if (!local.IsDirty || dto.LastModifiedUtc >= local.LastModifiedUtc)
                    {
                        TodoMapper.Apply(local, dto);
                        local.ETag = newTag ?? local.ETag;
                        local.IsDirty = false;
                        await _repo.UpdateAsync(local, ct);
                    }
                }
            }
            await _repo.SaveChangesAsync(ct);
            page++;
            etag = newTag ?? etag;
        }
        // etag 저장 (사용자 설정, 로컬 파일 등)
    }

    public async Task SyncAllAsync(CancellationToken ct = default)
    {
        await PushAsync(ct);
        await PullAsync(ct);
    }
}
```

DI:
```csharp
services.AddScoped<SyncService>();
```

> **주기 동기화**: `System.Threading.PeriodicTimer`로 30~60초마다, 또는 “온라인 전환/앱 포커스 복귀/수동 버튼”에 트리거.

---

## 10) 동시성/트랜잭션

- **EF Core**: `DbContext`는 **단일 스레드/스코프 단위** 사용.  
- 여러 저장 작업을 **하나의 트랜잭션**에 묶기:
```csharp
using var tx = await _db.Database.BeginTransactionAsync(ct);
await _repo.AddAsync(todo1, ct);
await _repo.AddAsync(todo2, ct);
await _repo.SaveChangesAsync(ct);
await tx.CommitAsync(ct);
```

- **낙관적 동시성**: 서버 API는 ETag/If-Match로 보호. 로컬 SQLite에서도 `RowVersion(BLOB)` 컬럼을 둘 수 있으나 SQLite는 rowversion 타입이 없으므로 **수동 관리**(LastModifiedUtc 비교/해시 사용).

---

## 11) 성능 최적화 체크리스트

- **쿼리**: 필요한 컬럼만(ProjectTo DTO) / `AsNoTracking` 활용  
- **배치**: 대량 삽입/업데이트 시 `SaveChanges` 호출 횟수 최소화  
- **인덱스**: 자주 필터링되는 칼럼(`IsDirty`, `LastModifiedUtc`) 인덱스  
- **WAL 모드**: 쓰기 병행성 향상
```csharp
await _db.Database.ExecuteSqlRawAsync("PRAGMA journal_mode=WAL;");
```
- **가상화**: `ListView`/`DataGrid` 가상화 켜기  
- **UI 스레드**: 디스크/네트워크 IO는 `await` + `ConfigureAwait(false)`(라이브러리) / UI 업데이트는 `Dispatcher`를 통해 최소화

---

## 12) 오류/네트워크 회복력

- **Polly** 재시도(지수 백오프), 429/5xx 처리  
- **타임아웃**: HttpClient.Timeout / 개별 요청 `CancellationToken`  
- **오프라인 판별**: 네트워크 체크(핑/소켓/Connect), 실패 시 로컬만 사용하고 **IsDirty** 플래그 유지  
- **사용자 피드백**: 상태 바(온라인/오프라인/동기화 중), 충돌 알림 UI

---

## 13) 보안/토큰 관리

- **Auth**: Bearer Token(Access/Refresh), HttpClientFactory에서 메시지 핸들러로 자동 주입  
- **보존**: Windows DPAPI/ProtectedData, MSAL(기업 환경), 또는 OS 보안 저장소  
- **전송**: HTTPS 필수, 인증 실패 시 `401` → 재인증 UI

---

## 14) 테스트 전략

- **도메인**: 순수 단위 테스트(로직/밸리데이션)  
- **데이터**: **SQLite In-Memory**(주의: 관계/제약 반영)로 통합 테스트
```csharp
var conn = new SqliteConnection("DataSource=:memory:");
await conn.OpenAsync();
var options = new DbContextOptionsBuilder<AppDbContext>().UseSqlite(conn).Options;
using var db = new AppDbContext(options);
db.Database.EnsureCreated();
// 테스트 진행
```
- EFCore.InMemory는 실제 SQLite와 동작 차이가 있으므로 **스키마/제약 검증에는 SQLite In-Memory 추천**  
- **API 클라이언트**: `HttpMessageHandler` 스텁으로 응답 시뮬레이션  
- **동기화**: 푸시/풀/충돌 케이스별 시나리오 테스트

---

## 15) 실제 운영 팁

- **마이그레이션 버전 관리**: 앱 시작 시 자동 적용하되, 실패 시 백업/롤백 전략  
- **DB 백업**: 앱 종료 시/주기적으로 `.db` 백업(파일 잠금 주의)  
- **로깅**: EF `LogTo`, HttpClient `LoggingHandler`, Sync 이벤트 로그  
- **데이터 정리**: tombstone(삭제표시) 주기 정리/아카이빙  
- **국제화**: 서버/로컬 모두 **UTC** 저장, 표시 시 로컬 타임존 변환

---

## 16) 끝까지 이어지는 **전체 흐름 요약**

1. **App 시작** → Db 폴더 생성 → **Migrate**  
2. **MainViewModel.Load** → 로컬 DB에서 목록 로드  
3. **사용자 조작**(추가/수정/삭제) → 엔티티에 `IsDirty=true` & 저장  
4. **SyncService** 트리거(주기/버튼/온라인 전환)  
   - **Push**(Dirty/Tombstone → API) → 성공 시 `IsDirty=false`, `ETag` 갱신  
   - **Pull**(ETag 기반 변경분 페치) → 로컬 병합(정책 적용)  
5. **UI**는 `ObservableCollection<Todo>` 바인딩 → 자동 업데이트  
6. **오프라인** 시에도 로컬 작업 계속 → 온라인 시 **자동 동기화**

---

## 17) 추가 코드 조각 (필요할 때 가져다 쓰기)

### 17.1 ValueConverter: 완료 시 취소선
```csharp
public class BoolToStrikeConverter : IValueConverter
{
    public object Convert(object value, Type t, object p, CultureInfo c)
        => (value is bool b && b) ? TextDecorations.Strikethrough : null!;
    public object ConvertBack(object v, Type t, object p, CultureInfo c) => Binding.DoNothing;
}
```

### 17.2 Dispatcher 안전 업데이트
```csharp
Application.Current.Dispatcher.Invoke(() => Items.Insert(0, entity));
```

### 17.3 EF Core 변경 감지 최적화
```csharp
_db.ChangeTracker.AutoDetectChangesEnabled = false;
// 대량 작업 전/후에만 DetectChanges 호출
```

---

## 18) 자주 묻는 질문(FAQ)

**Q. EF Core 추적 엔티티를 그대로 바인딩해도 되나요?**  
A. 소규모는 OK. 대규모 목록에서는 `AsNoTracking`으로 DTO를 뷰에 바인딩 → 편집 시 선택 항목만 Attach/Update를 권장.

**Q. SQLite에서 동시 쓰기 충돌은?**  
A. WAL 모드/짧은 트랜잭션/재시도 전략. UI에서 대량 쓰기를 분할.

**Q. 마이그레이션이 잦을 때 운영 배포는?**  
A. 앱 시작 시 자동 마이그레이션 + 실패 시 백업 복구. 스키마 변경은 호환성 고려(새 컬럼 추가 → 데이터 이전 → 구 컬럼 제거).

**Q. API 충돌 정책을 다르게 하고 싶습니다.**  
A. SyncService에서 412/409 응답을 분기하여 “서버 우선/클라 우선/사용자 선택”을 구현하고, 충돌 레코드를 별도 테이블로 보존하여 UI에 표기하세요.

---

## 19) 결론

- **EF Core + SQLite**로 **오프라인 친화적인 로컬 저장소**를 만들고,  
- **HttpClientFactory + Polly**로 **회복력 있는 API 연동**을 구성하며,  
- **ETag/IsDirty/LastModifiedUtc**를 이용한 **안전한 동기화**로 실전 품질을 확보할 수 있습니다.  
- MVVM + DI + Generic Host를 활용하면 **테스트 가능한 구조**와 **장기 유지보수성**이 크게 향상됩니다.
