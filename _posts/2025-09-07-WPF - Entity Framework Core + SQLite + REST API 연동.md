---
layout: post
title: WPF - Entity Framework Core + SQLite + REST API 연동
date: 2025-09-07 20:25:23 +0900
category: WPF
---
# WPF에서 Entity Framework Core + SQLite + REST API 연동

이 글은 WPF 애플리케이션에서 로컬 데이터베이스(SQLite)를 Entity Framework Core(EF Core)로 관리하고, 원격 REST API와 데이터를 동기화하는 방법을 설명합니다. 초중급 개발자를 기준으로 핵심 개념과 실습 위주로 구성했습니다.

---

## 개발 환경과 시나리오

- **프레임워크**: .NET 7/8 WPF
- **도메인**: Todo(할 일) 관리 앱
- **로컬 저장소**: SQLite (파일 기반)
- **ORM**: Entity Framework Core
- **API**: RESTful (CRUD, 페이지네이션, ETag 기반 변경 감지)
- **앱 특성**: 오프라인 작업 가능, 온라인 복구 시 동기화

---

## 프로젝트 구조

간단한 구조로 시작합니다.

```
TodoDemo/
  TodoDemo.App/         # WPF UI (Views, ViewModels)
  TodoDemo.Domain/      # 엔티티, DTO
  TodoDemo.Data/        # DbContext, 리포지토리
  TodoDemo.ApiClient/   # REST API 클라이언트
  TodoDemo.Sync/        # 동기화 서비스
```

---

## 1. 엔티티 설계 (도메인)

동기화를 위한 추가 필드를 포함합니다.

```csharp
// TodoDemo.Domain/Entities/Todo.cs
public class Todo
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Title { get; set; } = "";
    public bool IsDone { get; set; }
    public DateTimeOffset CreatedAt { get; set; } = DateTimeOffset.UtcNow;
    public DateTimeOffset? UpdatedAt { get; set; }

    // 동기화 지원 필드
    public DateTimeOffset LastModifiedUtc { get; set; } = DateTimeOffset.UtcNow;
    public string? ETag { get; set; }               // 서버 버전
    public bool IsDirty { get; set; }               // 로컬 변경 여부
    public bool IsDeleted { get; set; }             // 소프트 삭제
}
```

---

## 2. DbContext와 SQLite 설정

### DbContext 작성

```csharp
// TodoDemo.Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using TodoDemo.Domain.Entities;

public class AppDbContext : DbContext
{
    public DbSet<Todo> Todos => Set<Todo>();

    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Todo>(entity =>
        {
            entity.HasKey(x => x.Id);
            entity.Property(x => x.Title).IsRequired().HasMaxLength(200);
            entity.HasIndex(x => x.IsDirty);
            entity.HasIndex(x => x.LastModifiedUtc);
            entity.HasQueryFilter(x => !x.IsDeleted); // 소프트 삭제 필터
        });
    }
}
```

### 연결 문자열 및 데이터베이스 초기화

WPF 앱에서 `Generic Host`를 사용해 서비스를 등록하고, 시작 시 마이그레이션을 자동 적용합니다.

```csharp
// TodoDemo.App/App.xaml.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

public partial class App : Application
{
    public static IHost HostApplication { get; } = Host.CreateDefaultBuilder()
        .ConfigureServices((context, services) =>
        {
            // 데이터베이스 경로 설정
            string dbFolder = Path.Combine(
                Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
                "TodoDemo");
            Directory.CreateDirectory(dbFolder);
            string dbPath = Path.Combine(dbFolder, "todo.db");

            services.AddDbContext<AppDbContext>(options =>
                options.UseSqlite($"Data Source={dbPath}"));
            // 다른 서비스 등록...
        })
        .Build();

    protected override async void OnStartup(StartupEventArgs e)
    {
        await HostApplication.StartAsync();

        // 마이그레이션 자동 적용
        using var scope = HostApplication.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();

        // 메인 윈도우 표시
        var mainWindow = HostApplication.Services.GetRequiredService<MainWindow>();
        mainWindow.Show();

        base.OnStartup(e);
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        await HostApplication.StopAsync();
        HostApplication.Dispose();
        base.OnExit(e);
    }
}
```

### 마이그레이션 생성

```bash
dotnet ef migrations add InitialCreate --project TodoDemo.Data --startup-project TodoDemo.App
dotnet ef database update --project TodoDemo.Data --startup-project TodoDemo.App
```

> SQLite는 스키마 변경에 제약이 있습니다. 컬럼 삭제나 타입 변경이 필요하면 새 컬럼 추가 → 데이터 이전 → 기존 컬럼 제거 방식으로 진행합니다.

---

## 3. 리포지토리 (선택 사항)

EF Core 자체가 Unit of Work를 제공하지만, 테스트와 추상화를 위해 간단한 리포지토리를 만들 수 있습니다.

```csharp
// TodoDemo.Data/ITodoRepository.cs
public interface ITodoRepository
{
    Task<Todo?> GetAsync(Guid id);
    Task<List<Todo>> GetAllAsync();
    Task AddAsync(Todo entity);
    Task UpdateAsync(Todo entity);
    Task SoftDeleteAsync(Guid id);
    Task<int> SaveChangesAsync();
}
```

```csharp
// TodoDemo.Data/TodoRepository.cs
public class TodoRepository : ITodoRepository
{
    private readonly AppDbContext _db;
    public TodoRepository(AppDbContext db) => _db = db;

    public async Task<Todo?> GetAsync(Guid id)
        => await _db.Todos.AsNoTracking().FirstOrDefaultAsync(x => x.Id == id);

    public async Task<List<Todo>> GetAllAsync()
        => await _db.Todos.AsNoTracking().OrderByDescending(x => x.LastModifiedUtc).ToListAsync();

    public async Task AddAsync(Todo entity)
    {
        entity.IsDirty = true;
        entity.LastModifiedUtc = DateTimeOffset.UtcNow;
        await _db.Todos.AddAsync(entity);
    }

    public Task UpdateAsync(Todo entity)
    {
        entity.IsDirty = true;
        entity.LastModifiedUtc = DateTimeOffset.UtcNow;
        _db.Todos.Update(entity);
        return Task.CompletedTask;
    }

    public async Task SoftDeleteAsync(Guid id)
    {
        var entity = await _db.Todos.FirstOrDefaultAsync(x => x.Id == id);
        if (entity != null)
        {
            entity.IsDeleted = true;
            entity.IsDirty = true;
            entity.LastModifiedUtc = DateTimeOffset.UtcNow;
        }
    }

    public async Task<int> SaveChangesAsync() => await _db.SaveChangesAsync();
}
```

서비스 등록:

```csharp
services.AddScoped<ITodoRepository, TodoRepository>();
```

---

## 4. ViewModel과 바인딩

MVVM 패턴을 사용합니다. 여기서는 `CommunityToolkit.Mvvm`을 활용합니다.

```csharp
// TodoDemo.App/ViewModels/MainViewModel.cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Collections.ObjectModel;

public partial class MainViewModel : ObservableObject
{
    private readonly ITodoRepository _repository;
    public ObservableCollection<Todo> Items { get; } = new();

    [ObservableProperty]
    private string _newTitle = "";

    public MainViewModel(ITodoRepository repository)
    {
        _repository = repository;
    }

    [RelayCommand]
    private async Task LoadAsync()
    {
        Items.Clear();
        foreach (var todo in await _repository.GetAllAsync())
            Items.Add(todo);
    }

    [RelayCommand]
    private async Task AddAsync()
    {
        if (string.IsNullOrWhiteSpace(NewTitle)) return;

        var todo = new Todo { Title = NewTitle };
        await _repository.AddAsync(todo);
        await _repository.SaveChangesAsync();
        Items.Insert(0, todo);
        NewTitle = "";
    }

    [RelayCommand]
    private async Task ToggleDoneAsync(Todo item)
    {
        item.IsDone = !item.IsDone;
        await _repository.UpdateAsync(item);
        await _repository.SaveChangesAsync();
        // ObservableCollection이 같은 객체를 참조하므로 UI 갱신됨
    }
}
```

View 예제:

```xml
<Window x:Class="TodoDemo.App.MainWindow" ...>
    <DockPanel Margin="16">
        <StackPanel DockPanel.Dock="Top" Orientation="Horizontal" Spacing="8">
            <TextBox Width="320" Text="{Binding NewTitle, UpdateSourceTrigger=PropertyChanged}"/>
            <Button Content="추가" Command="{Binding AddCommand}"/>
            <Button Content="새로고침" Command="{Binding LoadCommand}"/>
        </StackPanel>

        <ListView ItemsSource="{Binding Items}">
            <ListView.ItemTemplate>
                <DataTemplate>
                    <StackPanel Orientation="Horizontal" Spacing="8">
                        <CheckBox IsChecked="{Binding IsDone}" 
                                  Command="{Binding DataContext.ToggleDoneCommand, RelativeSource={RelativeSource AncestorType=ListView}}"
                                  CommandParameter="{Binding}"/>
                        <TextBlock Text="{Binding Title}"/>
                        <TextBlock Text="{Binding LastModifiedUtc, StringFormat='yyyy-MM-dd HH:mm'}" Foreground="#888"/>
                    </StackPanel>
                </DataTemplate>
            </ListView.ItemTemplate>
        </ListView>
    </DockPanel>
</Window>
```

---

## 5. REST API 클라이언트

### DTO와 매핑

```csharp
// TodoDemo.Domain/Dto/TodoDto.cs
public record TodoDto(Guid Id, string Title, bool IsDone, DateTimeOffset LastModifiedUtc, bool IsDeleted);

// 간단한 매핑
public static class TodoMapper
{
    public static TodoDto ToDto(Todo entity) =>
        new(entity.Id, entity.Title, entity.IsDone, entity.LastModifiedUtc, entity.IsDeleted);

    public static void Apply(Todo entity, TodoDto dto)
    {
        entity.Title = dto.Title;
        entity.IsDone = dto.IsDone;
        entity.LastModifiedUtc = dto.LastModifiedUtc;
        entity.IsDeleted = dto.IsDeleted;
    }
}
```

### HttpClientFactory + Polly (재시도)

```csharp
// TodoDemo.ApiClient/ServiceCollectionExtensions.cs
using Polly;
using Polly.Extensions.Http;

public static class ApiClientRegistration
{
    public static IServiceCollection AddTodoApi(this IServiceCollection services, string baseAddress)
    {
        services.AddHttpClient<ITodoApi, TodoApiClient>(client =>
        {
            client.BaseAddress = new Uri(baseAddress);
            client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
        })
        .AddPolicyHandler(GetRetryPolicy())
        .AddPolicyHandler(GetCircuitBreaker());

        return services;
    }

    private static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
        => HttpPolicyExtensions
            .HandleTransientHttpError()
            .OrResult(r => r.StatusCode == System.Net.HttpStatusCode.TooManyRequests)
            .WaitAndRetryAsync(3, retry => TimeSpan.FromMilliseconds(200 * retry));

    private static IAsyncPolicy<HttpResponseMessage> GetCircuitBreaker()
        => HttpPolicyExtensions
            .HandleTransientHttpError()
            .CircuitBreakerAsync(5, TimeSpan.FromSeconds(20));
}
```

### API 인터페이스 및 구현

```csharp
// TodoDemo.ApiClient/ITodoApi.cs
public interface ITodoApi
{
    Task<(IReadOnlyList<TodoDto> Items, string? ETag)> GetPageAsync(int page, int size, string? ifNoneMatch = null);
    Task<(TodoDto Item, string? ETag)> UpsertAsync(TodoDto dto, string? ifMatch);
    Task DeleteAsync(Guid id, string? ifMatch);
}
```

```csharp
// TodoDemo.ApiClient/TodoApiClient.cs
public class TodoApiClient : ITodoApi
{
    private readonly HttpClient _http;
    public TodoApiClient(HttpClient http) => _http = http;

    public async Task<(IReadOnlyList<TodoDto>, string?)> GetPageAsync(int page, int size, string? ifNoneMatch = null)
    {
        var req = new HttpRequestMessage(HttpMethod.Get, $"/todos?page={page}&size={size}");
        if (!string.IsNullOrEmpty(ifNoneMatch))
            req.Headers.IfNoneMatch.ParseAdd(ifNoneMatch);

        var resp = await _http.SendAsync(req);
        if (resp.StatusCode == System.Net.HttpStatusCode.NotModified)
            return (Array.Empty<TodoDto>(), resp.Headers.ETag?.Tag);

        resp.EnsureSuccessStatusCode();
        var items = await resp.Content.ReadFromJsonAsync<List<TodoDto>>() ?? new();
        return (items, resp.Headers.ETag?.Tag);
    }

    public async Task<(TodoDto, string?)> UpsertAsync(TodoDto dto, string? ifMatch)
    {
        var req = new HttpRequestMessage(HttpMethod.Put, $"/todos/{dto.Id}")
        {
            Content = JsonContent.Create(dto)
        };
        if (!string.IsNullOrEmpty(ifMatch))
            req.Headers.TryAddWithoutValidation("If-Match", ifMatch);

        var resp = await _http.SendAsync(req);
        resp.EnsureSuccessStatusCode();
        var body = await resp.Content.ReadFromJsonAsync<TodoDto>() ?? dto;
        return (body, resp.Headers.ETag?.Tag);
    }

    public async Task DeleteAsync(Guid id, string? ifMatch)
    {
        var req = new HttpRequestMessage(HttpMethod.Delete, $"/todos/{id}");
        if (!string.IsNullOrEmpty(ifMatch))
            req.Headers.TryAddWithoutValidation("If-Match", ifMatch);

        var resp = await _http.SendAsync(req);
        resp.EnsureSuccessStatusCode();
    }
}
```

서비스 등록:

```csharp
services.AddTodoApi("https://api.example.com"); // 실제 URL로 변경
```

---

## 6. 동기화 서비스

동기화 서비스는 로컬 변경사항을 서버로 밀어넣고(Push), 서버의 변경사항을 가져와(Pull) 병합합니다.

```csharp
// TodoDemo.Sync/SyncService.cs
public class SyncService
{
    private readonly ITodoRepository _repo;
    private readonly ITodoApi _api;

    public SyncService(ITodoRepository repo, ITodoApi api)
    {
        _repo = repo;
        _api = api;
    }

    public async Task PushAsync()
    {
        var dirty = await _repo.GetAllAsync(); // 실제로는 IsDirty 필터
        foreach (var entity in dirty.Where(x => x.IsDirty || x.IsDeleted))
        {
            try
            {
                if (entity.IsDeleted)
                {
                    await _api.DeleteAsync(entity.Id, entity.ETag);
                    // 필요시 완전 삭제
                }
                else
                {
                    var (remote, etag) = await _api.UpsertAsync(TodoMapper.ToDto(entity), entity.ETag);
                    TodoMapper.Apply(entity, remote);
                    entity.ETag = etag;
                    entity.IsDirty = false;
                }
            }
            catch (HttpRequestException ex) when (ex.StatusCode == System.Net.HttpStatusCode.PreconditionFailed)
            {
                // 충돌 발생: 서버 우선 정책으로 처리
                await ResolveConflict(entity);
            }
        }
        await _repo.SaveChangesAsync();
    }

    public async Task PullAsync()
    {
        string? etag = null;
        int page = 1, size = 50;
        while (true)
        {
            var (items, newTag) = await _api.GetPageAsync(page, size, etag);
            if (items.Count == 0) break;

            foreach (var dto in items)
            {
                var local = await _repo.GetAsync(dto.Id);
                if (local == null)
                {
                    local = new Todo { Id = dto.Id };
                    TodoMapper.Apply(local, dto);
                    local.IsDirty = false;
                    await _repo.AddAsync(local);
                }
                else if (!local.IsDirty || dto.LastModifiedUtc >= local.LastModifiedUtc)
                {
                    TodoMapper.Apply(local, dto);
                    local.IsDirty = false;
                }
            }
            await _repo.SaveChangesAsync();
            page++;
            etag = newTag ?? etag;
        }
    }

    public async Task SyncAllAsync()
    {
        await PushAsync();
        await PullAsync();
    }

    private async Task ResolveConflict(Todo entity)
    {
        // 간단한 충돌 해결: 서버 버전으로 덮어쓰기
        var (serverDto, _) = await _api.UpsertAsync(TodoMapper.ToDto(entity), null);
        TodoMapper.Apply(entity, serverDto);
        entity.IsDirty = false;
        // 사용자에게 알림을 줄 수도 있음
    }
}
```

서비스 등록:

```csharp
services.AddScoped<SyncService>();
```

### 주기적 동기화

`PeriodicTimer`를 사용해 30초마다 동기화를 실행합니다.

```csharp
// 예: App.xaml.cs의 OnStartup에서 시작
_ = Task.Run(async () =>
{
    var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));
    while (await timer.WaitForNextTickAsync())
    {
        using var scope = HostApplication.Services.CreateScope();
        var sync = scope.ServiceProvider.GetRequiredService<SyncService>();
        await sync.SyncAllAsync();
    }
});
```

---

## 7. 오류 처리와 성능 팁

### EF Core 성능

- 대량 조회 시 `AsNoTracking()` 사용
- 변경 감지가 필요 없는 읽기 전용 작업에는 `AsNoTracking()` 권장
- `SaveChanges` 호출 횟수 최소화 (배치 처리)

```csharp
// 대량 추가 예제
using (var scope = _dbContext.Database.BeginTransaction())
{
    foreach (var item in manyItems)
        _dbContext.Todos.Add(item);
    await _dbContext.SaveChangesAsync();
    await scope.CommitAsync();
}
```

### SQLite 최적화

- WAL 모드 활성화: `PRAGMA journal_mode=WAL;`
- 인덱스: `IsDirty`, `LastModifiedUtc`에 인덱스 생성

```csharp
// DbContext 생성 직후 실행
await dbContext.Database.ExecuteSqlRawAsync("PRAGMA journal_mode=WAL;");
```

### 네트워크 회복력

- Polly 정책으로 재시도, 회로 차단기 적용
- 취소 토큰을 사용하여 사용자 취소 가능하게
- 오프라인 감지 시 동기화 건너뛰고 로컬 작업만

---

## 8. 전체 흐름 요약

1. **앱 시작** → 데이터베이스 폴더 생성 → 마이그레이션 적용
2. **MainViewModel 로드** → 로컬 데이터 표시
3. **사용자 조작** (추가/수정/삭제) → 엔티티에 `IsDirty = true` 설정 및 저장
4. **주기적 동기화** 또는 버튼 클릭 → `SyncService.SyncAllAsync()` 호출
   - Push: `IsDirty` 항목을 서버로 전송, 성공 시 `IsDirty = false`, ETag 갱신
   - Pull: 서버 변경분 가져와 로컬에 병합 (충돌 시 서버 우선)
5. UI는 `ObservableCollection`을 통해 자동 갱신

---

## 자주 묻는 질문

**Q. EF Core 추적 엔티티를 그대로 바인딩해도 되나요?**  
A. 작은 앱에서는 괜찮습니다. 대규모 목록에서는 `AsNoTracking`으로 읽고, 편집 시 별도로 `Update`하는 것이 좋습니다.

**Q. SQLite에서 동시 쓰기 충돌은 어떻게 하나요?**  
A. WAL 모드 활성화, 트랜잭션 짧게 유지, 재시도 로직을 구현합니다.

**Q. API 충돌 정책을 바꾸고 싶어요.**  
A. `SyncService.ResolveConflict`에서 "서버 우선" 대신 "클라이언트 우선"이나 사용자 선택 UI를 구현할 수 있습니다.

---

## 결론

이 구조를 통해 WPF 앱에서 로컬 SQLite와 원격 REST API를 안정적으로 연동할 수 있습니다. EF Core가 변경 추적과 마이그레이션을 관리하고, Polly가 네트워크 장애를 회복하며, 동기화 서비스가 데이터 일관성을 유지합니다. 초중급 개발자라면 먼저 간단한 CRUD부터 구현하고 점차 동기화 기능을 추가하는 방식으로 접근하시기 바랍니다.