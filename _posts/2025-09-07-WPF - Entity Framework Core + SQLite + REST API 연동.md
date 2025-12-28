---
layout: post
title: WPF - Entity Framework Core + SQLite + REST API 연동
date: 2025-09-07 20:25:23 +0900
category: WPF
---
# WPF에서 Entity Framework Core + SQLite + REST API 연동 완전 정복

이 문서는 WPF(MVVM) 애플리케이션에서 로컬 SQLite 데이터베이스를 Entity Framework Core로 관리하고, 원격 REST API와 안정적으로 동기화하는 실전 아키텍처를 설명합니다. .NET 7/8 WPF를 기준으로 하며, .NET 6 또는 .NET Framework 4.8에서도 기본 구조는 동일하게 적용할 수 있습니다.

## 데모 시나리오
*   **도메인**: Todo(할 일) 관리 애플리케이션
*   **로컬 저장소**: SQLite (파일: `%LOCALAPPDATA%\TodoDemo\todo.db`)
*   **ORM**: Entity Framework Core (변경 추적, 마이그레이션, 동시성 처리, 관계형 데이터 관리)
*   **API**: `https://api.example.com/todos` (CRUD, 페이지네이션, ETag 기반 변경 감지)
*   **앱 특성**: 오프라인 상태에서도 편집 가능하며, 온라인 복구 시 양방향 동기화 및 충돌 해결 지원

## 프로젝트 구조
```
TodoDemo/
  TodoDemo.App/                 # WPF UI (Views, ViewModels)
  TodoDemo.Domain/              # 엔티티, 비즈니스 규칙, DTO, 계약
  TodoDemo.Data/                # EF Core DbContext, 마이그레이션, 리포지토리
  TodoDemo.ApiClient/           # REST 클라이언트 (HttpClientFactory, Polly, DTO 매핑)
  TodoDemo.Sync/                # 동기화 서비스 (오프라인 처리, 충돌 해결, 스케줄링)
  TodoDemo.Tests/               # 단위 및 통합 테스트
```

## 핵심 패키지
```bash
# EF Core + SQLite
dotnet add TodoDemo.Data package Microsoft.EntityFrameworkCore
dotnet add TodoDemo.Data package Microsoft.EntityFrameworkCore.Sqlite
dotnet add TodoDemo.Data package Microsoft.EntityFrameworkCore.Design

# 의존성 주입 및 호스팅 (WPF에서 Generic Host 사용 권장)
dotnet add TodoDemo.App package Microsoft.Extensions.Hosting
dotnet add TodoDemo.App package Microsoft.Extensions.DependencyInjection

# HTTP 클라이언트 및 회복력 패턴
dotnet add TodoDemo.ApiClient package Microsoft.Extensions.Http
dotnet add TodoDemo.ApiClient package Polly.Extensions.Http

# 매핑 및 검증
dotnet add TodoDemo.Domain package FluentValidation
dotnet add TodoDemo.App package CommunityToolkit.Mvvm
```

## 도메인 및 엔티티 설계

### 엔티티 (동기화 및 동시성 필드 포함)
```csharp
// TodoDemo.Domain/Entities/Todo.cs
public class Todo
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Title { get; set; } = "";
    public bool IsDone { get; set; }
    public DateTimeOffset CreatedAt { get; set; } = DateTimeOffset.UtcNow;
    public DateTimeOffset? UpdatedAt { get; set; }

    // 동기화 및 충돌 해결을 위한 필드
    public DateTimeOffset LastModifiedUtc { get; set; } = DateTimeOffset.UtcNow;
    public string? ETag { get; set; }                 // 서버 리소스 버전
    public bool IsDirty { get; set; }                 // 로컬 변경 여부
    public bool IsDeleted { get; set; }               // 소프트 삭제 (동기화 시 전파)
}
```

## DbContext 구성
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
            entity.Property(x => x.ETag).HasMaxLength(64);
            entity.HasIndex(x => x.IsDirty);
            entity.HasIndex(x => x.LastModifiedUtc);
            entity.HasQueryFilter(x => !x.IsDeleted); // 소프트 삭제 글로벌 필터
        });
    }
}
```

### SQLite 연결 문자열 및 폴더 준비
```csharp
// TodoDemo.App/App.xaml.cs (Generic Host 구성 포함)
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

public partial class App : Application
{
    public static IHost HostApplication { get; } = Microsoft.Extensions.Hosting.Host
        .CreateDefaultBuilder()
        .ConfigureServices((context, services) =>
        {
            string dbDirectory = Path.Combine(Environment.GetFolderPath(
                Environment.SpecialFolder.LocalApplicationData), "TodoDemo");
            Directory.CreateDirectory(dbDirectory);
            string dbPath = Path.Combine(dbDirectory, "todo.db");

            services.AddDbContext<AppDbContext>(options =>
            {
                options.UseSqlite($"Data Source={dbPath}");
                options.EnableSensitiveDataLogging(false);
            });

            // 기타 서비스 등록 (아래 섹션들에서 설명)
        })
        .Build();

    protected override void OnStartup(StartupEventArgs e)
    {
        HostApplication.Start();
        
        // 마이그레이션 자동 적용 (초기 실행/업데이트)
        using var scope = HostApplication.Services.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        dbContext.Database.Migrate();

        base.OnStartup(e);
        new MainWindow { DataContext = HostApplication.Services.GetRequiredService<MainViewModel>() }.Show();
    }

    protected override void OnExit(ExitEventArgs e)
    {
        HostApplication.Dispose();
        base.OnExit(e);
    }
}
```

## 마이그레이션
```bash
dotnet ef migrations add InitialCreate --project TodoDemo.Data --startup-project TodoDemo.App
dotnet ef database update --project TodoDemo.Data --startup-project TodoDemo.App
```

**SQLite 제약 사항**: SQLite는 일부 `ALTER COLUMN` 작업에 제한이 있습니다. 스키마 변경이 필요한 경우, "새 컬럼 추가 → 데이터 마이그레이션 → 기존 컬럼 삭제" 패턴을 사용하세요.

## 리포지토리 및 유닛 오브 워크 패턴 (선택 사항)

EF Core 자체가 UoW(Unit of Work) 패턴을 제공하지만, 테스트 용이성과 API 인터페이스화를 위해 래핑할 수 있습니다.

```csharp
// TodoDemo.Data/ITodoRepository.cs
using System.Linq.Expressions;

public interface ITodoRepository
{
    Task<Todo?> GetAsync(Guid id, CancellationToken ct = default);
    Task<List<Todo>> GetAllAsync(Expression<Func<Todo, bool>>? predicate = null, CancellationToken ct = default);
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
    private readonly AppDbContext _dbContext;
    public TodoRepository(AppDbContext dbContext) => _dbContext = dbContext;

    public Task<Todo?> GetAsync(Guid id, CancellationToken ct = default)
        => _dbContext.Todos.AsNoTracking().FirstOrDefaultAsync(x => x.Id == id, ct);

    public async Task<List<Todo>> GetAllAsync(Expression<Func<Todo, bool>>? predicate = null, CancellationToken ct = default)
    {
        var query = _dbContext.Todos.AsNoTracking();
        if (predicate != null) query = query.Where(predicate);
        return await query.OrderByDescending(x => x.LastModifiedUtc).ToListAsync(ct);
    }

    public async Task AddAsync(Todo entity, CancellationToken ct = default)
    {
        entity.IsDirty = true;
        entity.LastModifiedUtc = DateTimeOffset.UtcNow;
        await _dbContext.Todos.AddAsync(entity, ct);
    }

    public Task UpdateAsync(Todo entity, CancellationToken ct = default)
    {
        entity.IsDirty = true;
        entity.LastModifiedUtc = DateTimeOffset.UtcNow;
        _dbContext.Todos.Update(entity);
        return Task.CompletedTask;
    }

    public async Task SoftDeleteAsync(Guid id, CancellationToken ct = default)
    {
        var entity = await _dbContext.Todos.FirstOrDefaultAsync(x => x.Id == id, ct);
        if (entity is null) return;
        
        entity.IsDeleted = true;
        entity.IsDirty = true;
        entity.LastModifiedUtc = DateTimeOffset.UtcNow;
    }

    public Task<int> SaveChangesAsync(CancellationToken ct = default)
        => _dbContext.SaveChangesAsync(ct);
}
```

의존성 주입 등록:
```csharp
services.AddScoped<ITodoRepository, TodoRepository>();
```

## WPF(MVVM)와 EF Core 바인딩

### ViewModel 기본 (CommunityToolkit.Mvvm 활용)
```csharp
// TodoDemo.App/ViewModels/MainViewModel.cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Collections.ObjectModel;

public partial class MainViewModel : ObservableObject
{
    private readonly ITodoRepository _repository;
    public ObservableCollection<Todo> Items { get; } = new ObservableCollection<Todo>();

    [ObservableProperty]
    private string _newTitle = "";

    public MainViewModel(ITodoRepository repository) => _repository = repository;

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
        // ObservableCollection에 동일한 참조가 있으므로 UI가 자동 갱신됩니다 (필요 시 OnPropertyChanged 호출)
    }
}
```

의존성 주입:
```csharp
services.AddSingleton<MainViewModel>();
```

### View 구현
```xml
<!-- TodoDemo.App/Views/MainWindow.xaml -->
<Window x:Class="TodoDemo.App.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:domain="clr-namespace:TodoDemo.Domain.Entities;assembly=TodoDemo.Domain"
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
            <CheckBox IsChecked="{Binding IsDone}" 
                      Command="{Binding DataContext.ToggleDoneCommand, 
                               RelativeSource={RelativeSource AncestorType=ListView}}" 
                      CommandParameter="{Binding}"/>
            <TextBlock Text="{Binding Title}" 
                       TextDecorations="{Binding IsDone, Converter={StaticResource BoolToStrikeConverter}}"/>
            <TextBlock Foreground="#888" Text="{Binding LastModifiedUtc}"/>
          </StackPanel>
        </DataTemplate>
      </ListView.ItemTemplate>
    </ListView>
  </DockPanel>
</Window>
```

**성능 팁**: 대량의 목록을 표시할 때는 가상화를 활성화하세요 (`VirtualizingStackPanel.IsVirtualizing="True"`). EF Core에서 추적된 엔티티를 직접 바인딩할 때는 변경 추적 범위를 관리해야 합니다. 대규모 목록에서는 `AsNoTracking`으로 DTO를 뷰에 바인딩하고, 편집 시 선택된 항목만 Attach/Update하는 방식을 권장합니다.

## REST API 클라이언트

### DTO 및 매퍼
```csharp
// TodoDemo.Domain/Dto/TodoDto.cs
public record TodoDto(Guid Id, string Title, bool IsDone, 
                      DateTimeOffset LastModifiedUtc, bool IsDeleted);

// 매핑 (간단한 수동 매핑 예시)
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

### HttpClientFactory 및 Polly 설정
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
        services.AddHttpClient<ITodoApi, TodoApiClient>(client =>
        {
            client.BaseAddress = new Uri(baseAddress);
            client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
            // 인증 토큰이 있다면 Authorization 헤더 추가
        })
        .AddPolicyHandler(GetRetryPolicy())
        .AddPolicyHandler(GetCircuitBreaker());

        return services;
    }

    private static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
        => HttpPolicyExtensions
            .HandleTransientHttpError()
            .OrResult(response => response.StatusCode == HttpStatusCode.TooManyRequests)
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
    Task<(IReadOnlyList<TodoDto> Items, string? ETag)> GetPageAsync(int page, int size, 
                                                                    string? ifNoneMatch = null, 
                                                                    CancellationToken ct = default);
    Task<(TodoDto Item, string? ETag)> UpsertAsync(TodoDto dto, string? ifMatch, 
                                                   CancellationToken ct = default);
    Task DeleteAsync(Guid id, string? ifMatch, CancellationToken ct = default);
}
```

```csharp
// TodoDemo.ApiClient/TodoApiClient.cs
using System.Net.Http.Json;

public class TodoApiClient : ITodoApi
{
    private readonly HttpClient _httpClient;
    public TodoApiClient(HttpClient httpClient) => _httpClient = httpClient;

    public async Task<(IReadOnlyList<TodoDto>, string?)> GetPageAsync(int page, int size, 
                                                                      string? ifNoneMatch = null, 
                                                                      CancellationToken ct = default)
    {
        var request = new HttpRequestMessage(HttpMethod.Get, $"/todos?page={page}&size={size}");
        if (!string.IsNullOrEmpty(ifNoneMatch)) 
            request.Headers.IfNoneMatch.ParseAdd(ifNoneMatch);

        var response = await _httpClient.SendAsync(request, ct);
        if (response.StatusCode == System.Net.HttpStatusCode.NotModified)
            return (Array.Empty<TodoDto>(), response.Headers.ETag?.Tag);

        response.EnsureSuccessStatusCode();
        var list = await response.Content.ReadFromJsonAsync<List<TodoDto>>(cancellationToken: ct) ?? new();
        return (list, response.Headers.ETag?.Tag);
    }

    public async Task<(TodoDto, string?)> UpsertAsync(TodoDto dto, string? ifMatch, 
                                                      CancellationToken ct = default)
    {
        var request = new HttpRequestMessage(HttpMethod.Put, $"/todos/{dto.Id}")
        {
            Content = JsonContent.Create(dto)
        };
        
        if (!string.IsNullOrWhiteSpace(ifMatch))
            request.Headers.TryAddWithoutValidation("If-Match", ifMatch);

        var response = await _httpClient.SendAsync(request, ct);
        response.EnsureSuccessStatusCode();
        
        var body = await response.Content.ReadFromJsonAsync<TodoDto>(cancellationToken: ct) ?? dto;
        return (body, response.Headers.ETag?.Tag);
    }

    public async Task DeleteAsync(Guid id, string? ifMatch, CancellationToken ct = default)
    {
        var request = new HttpRequestMessage(HttpMethod.Delete, $"/todos/{id}");
        if (!string.IsNullOrWhiteSpace(ifMatch))
            request.Headers.TryAddWithoutValidation("If-Match", ifMatch);

        var response = await _httpClient.SendAsync(request, ct);
        response.EnsureSuccessStatusCode();
    }
}
```

의존성 주입:
```csharp
services.AddTodoApi("https://api.example.com");  // 앱 구성에서 BaseAddress 가져오기
```

## 오프라인 및 동기화 서비스

### 동기화 전략 (핵심 아이디어)

1.  **푸시 (Push)**: 로컬에서 `IsDirty == true`인 항목을 서버로 전송 (Upsert/Delete, If-Match: ETag 사용)
2.  **풀 (Pull)**: 서버의 변경분을 페이지 단위로 조회 (If-None-Match: ETag 사용) → 로컬에 병합
3.  **충돌 해결**: 서버 ETag 불일치 (412 Precondition Failed) 발생 시 정책 적용:
    *   **서버 우선**: 서버 버전을 받아 로컬에 덮어쓰기
    *   **클라이언트 우선**: 로컬 버전으로 재시도 (Force) 또는 별도 충돌 컬렉션으로 사용자에게 선택 유도
4.  **삭제 처리**: 소프트 삭제를 우선 적용 (복구 용이), 최종 정리 배치에서 하드 삭제

### 구현
```csharp
// TodoDemo.Sync/SyncService.cs
public class SyncService
{
    private readonly ITodoRepository _repository;
    private readonly ITodoApi _api;

    public SyncService(ITodoRepository repository, ITodoApi api)
    {
        _repository = repository;
        _api = api;
    }

    public async Task PushAsync(CancellationToken ct = default)
    {
        var dirtyItems = await _repository.GetAllAsync(x => x.IsDirty || x.IsDeleted, ct);
        foreach (var entity in dirtyItems)
        {
            try
            {
                if (entity.IsDeleted)
                {
                    await _api.DeleteAsync(entity.Id, entity.ETag, ct);
                    // 로컬에서도 완전 삭제(옵션) 또는 Tombstone 유지
                }
                else
                {
                    var (remote, etag) = await _api.UpsertAsync(TodoMapper.ToDto(entity), entity.ETag, ct);
                    TodoMapper.Apply(entity, remote);
                    entity.ETag = etag;
                    entity.IsDirty = false;
                }
            }
            catch (HttpRequestException) { throw; }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                // 412 등 충돌 처리 정책
                // 간단한 구현: 서버 우선
                // 실제 구현에서는 에러 유형별로 분기 처리
            }
        }
        await _repository.SaveChangesAsync(ct);
    }

    public async Task PullAsync(CancellationToken ct = default)
    {
        string? etag = null; // 앱 설정 또는 로컬에 마지막 ETag 저장
        int page = 1, size = 100;
        
        while (true)
        {
            var (items, newTag) = await _api.GetPageAsync(page, size, etag, ct);
            if (items.Count == 0) { /* NotModified 또는 끝 */ break; }

            foreach (var dto in items)
            {
                var localEntity = await _repository.GetAsync(dto.Id, ct);
                if (localEntity is null)
                {
                    localEntity = new Todo { Id = dto.Id };
                    TodoMapper.Apply(localEntity, dto);
                    localEntity.ETag = newTag ?? localEntity.ETag;
                    localEntity.IsDirty = false;
                    await _repository.AddAsync(localEntity, ct);
                }
                else
                {
                    // 로컬이 Dirty 상태면 충돌 정책 필요. 여기서는 서버 우선 덮어쓰기
                    if (!localEntity.IsDirty || dto.LastModifiedUtc >= localEntity.LastModifiedUtc)
                    {
                        TodoMapper.Apply(localEntity, dto);
                        localEntity.ETag = newTag ?? localEntity.ETag;
                        localEntity.IsDirty = false;
                        await _repository.UpdateAsync(localEntity, ct);
                    }
                }
            }
            await _repository.SaveChangesAsync(ct);
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

의존성 주입:
```csharp
services.AddScoped<SyncService>();
```

**주기적 동기화**: `System.Threading.PeriodicTimer`를 사용하여 30~60초마다, 또는 "온라인 전환/앱 포커스 복귀/수동 버튼" 이벤트에 트리거를 설정할 수 있습니다.

## 동시성 및 트랜잭션 처리
*   **EF Core**: `DbContext`는 **단일 스레드/스코프 단위**로 사용합니다.
*   여러 저장 작업을 하나의 트랜잭션에 묶기:
    ```csharp
    using var transaction = await _dbContext.Database.BeginTransactionAsync(ct);
    await _repository.AddAsync(todo1, ct);
    await _repository.AddAsync(todo2, ct);
    await _repository.SaveChangesAsync(ct);
    await transaction.CommitAsync(ct);
    ```
*   **낙관적 동시성**: 서버 API는 ETag/If-Match로 보호됩니다. 로컬 SQLite에서도 `RowVersion(BLOB)` 컬럼을 사용할 수 있으나, SQLite에는 기본 rowversion 타입이 없으므로 **수동 관리**(LastModifiedUtc 비교/해시 사용)가 필요합니다.

## 성능 최적화 지침
*   **쿼리 최적화**: 필요한 컬럼만 선택 (ProjectTo DTO), `AsNoTracking` 활용
*   **배치 작업**: 대량 삽입/업데이트 시 `SaveChanges` 호출 횟수 최소화
*   **인덱스 활용**: 자주 필터링되는 컬럼(`IsDirty`, `LastModifiedUtc`)에 인덱스 생성
*   **WAL 모드**: 쓰기 병행성 향상을 위해 SQLite WAL 모드 활성화
    ```csharp
    await _dbContext.Database.ExecuteSqlRawAsync("PRAGMA journal_mode=WAL;");
    ```
*   **UI 가상화**: `ListView`/`DataGrid` 가상화 활성화
*   **UI 스레드 관리**: 디스크/네트워크 I/O는 `await` + `ConfigureAwait(false)` 사용, UI 업데이트는 `Dispatcher`를 통해 최소화

## 오류 처리 및 네트워크 회복력
*   **Polly 재시도 정책**: 지수 백오프(Exponential Backoff) 적용, 429/5xx 응답 처리
*   **타임아웃 설정**: HttpClient.Timeout 또는 개별 요청 `CancellationToken` 활용
*   **오프라인 감지**: 네트워크 연결 체크 (핑/소켓/Connect), 실패 시 로컬만 사용하고 **IsDirty** 플래그 유지
*   **사용자 피드백**: 상태 표시줄(온라인/오프라인/동기화 중), 충돌 알림 UI 제공

## 보안 및 토큰 관리
*   **인증**: Bearer Token(Access/Refresh) 사용, HttpClientFactory에서 메시지 핸들러로 자동 주입
*   **토큰 보관**: Windows DPAPI/ProtectedData, MSAL(기업 환경), 또는 OS 보안 저장소 활용
*   **전송 보안**: HTTPS 필수 적용, 인증 실패 시 `401` → 재인증 UI 표시

## 테스트 전략
*   **도메인 테스트**: 순수 단위 테스트 (로직/유효성 검사)
*   **데이터 계층 테스트**: **SQLite In-Memory**를 통한 통합 테스트 (주의: 관계/제약 반영)
    ```csharp
    var connection = new SqliteConnection("DataSource=:memory:");
    await connection.OpenAsync();
    var options = new DbContextOptionsBuilder<AppDbContext>().UseSqlite(connection).Options;
    using var dbContext = new AppDbContext(options);
    dbContext.Database.EnsureCreated();
    // 테스트 진행
    ```
*   **EFCore.InMemory**는 실제 SQLite와 동작 차이가 있으므로 **스키마/제약 검증에는 SQLite In-Memory 권장**
*   **API 클라이언트 테스트**: `HttpMessageHandler` 스텁으로 응답 시뮬레이션
*   **동기화 테스트**: 푸시/풀/충돌 케이스별 시나리오 테스트

## 실제 운영 팁
*   **마이그레이션 버전 관리**: 앱 시작 시 자동 적용하되, 실패 시 백업/롤백 전략 준비
*   **데이터베이스 백업**: 앱 종료 시 또는 주기적으로 `.db` 파일 백업 (파일 잠금 주의)
*   **로깅**: EF `LogTo`, HttpClient `LoggingHandler`, 동기화 이벤트 로그
*   **데이터 정리**: tombstone(삭제표시) 주기적 정리/아카이빙
*   **국제화**: 서버/로컬 모두 **UTC** 저장, 표시 시 로컬 타임존 변환

## 전체 흐름 요약
1.  **애플리케이션 시작** → 데이터베이스 폴더 생성 → **마이그레이션 적용**
2.  **MainViewModel.Load** → 로컬 데이터베이스에서 목록 로드
3.  **사용자 조작** (추가/수정/삭제) → 엔티티에 `IsDirty=true` 설정 및 저장
4.  **SyncService 트리거** (주기적/버튼/온라인 전환)
    *   **푸시(Push)**: Dirty/Tombstone 항목 → API 전송 → 성공 시 `IsDirty=false`, `ETag` 갱신
    *   **풀(Pull)**: ETag 기반 변경분 페치 → 로컬 병합 (정책 적용)
5.  **UI**: `ObservableCollection<Todo>` 바인딩 → 자동 업데이트
6.  **오프라인 상태**: 로컬 작업 계속 → 온라인 상태 시 **자동 동기화**

## 추가 유틸리티 코드

### ValueConverter: 완료 시 취소선 표시
```csharp
public class BoolToStrikeConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
        => (value is bool boolValue && boolValue) ? TextDecorations.Strikethrough : null!;
    
    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture) 
        => Binding.DoNothing;
}
```

### Dispatcher를 통한 안전한 UI 업데이트
```csharp
Application.Current.Dispatcher.Invoke(() => Items.Insert(0, entity));
```

### EF Core 변경 감지 최적화
```csharp
_dbContext.ChangeTracker.AutoDetectChangesEnabled = false;
// 대량 작업 전/후에만 DetectChanges 호출
```

## 자주 묻는 질문 (FAQ)

**Q. EF Core 추적 엔티티를 그대로 바인딩해도 되나요?**
A. 소규모 애플리케이션에서는 가능합니다. 그러나 대규모 목록에서는 `AsNoTracking`으로 DTO를 뷰에 바인딩하고, 편집 시 선택된 항목만 Attach/Update하는 방식을 권장합니다.

**Q. SQLite에서 동시 쓰기 충돌은 어떻게 처리하나요?**
A. WAL 모드 활성화, 짧은 트랜잭션 사용, 재시도 전략 구현을 통해 처리합니다. UI에서 대량 쓰기 작업을 분할하는 것도 좋은 방법입니다.

**Q. 마이그레이션이 잦을 때 운영 배포는 어떻게 하나요?**
A. 앱 시작 시 자동 마이그레이션 적용과 실패 시 백업 복구 전략을 준비하세요. 스키마 변경은 호환성을 고려하여 (새 컬럼 추가 → 데이터 이전 → 기존 컬럼 제거) 방식으로 진행하세요.

**Q. API 충돌 정책을 다르게 구현하고 싶습니다.**
A. SyncService에서 412/409 응답을 분기 처리하여 "서버 우선/클라이언트 우선/사용자 선택" 정책을 구현하고, 충돌 레코드를 별도 테이블로 보관하여 UI에 표시할 수 있습니다.

---

## 결론

이 구현 가이드는 WPF 애플리케이션에서 다음과 같은 실전 아키텍처를 구축하는 방법을 제시합니다:

1. **오프라인 친화적인 로컬 저장소**: Entity Framework Core와 SQLite를 조합하여 강력한 로컬 데이터 관리 체계를 구축합니다.
2. **회복력 있는 API 연동**: HttpClientFactory와 Polly를 활용하여 네트워크 장애에 강건한 REST API 통신 계층을 구현합니다.
3. **안전한 동기화 메커니즘**: ETag, IsDirty, LastModifiedUtc 등의 메타데이터를 활용하여 데이터 충돌을 방지하고 안전한 양방향 동기화를 보장합니다.
4. **테스트 가능한 구조**: MVVM 패턴, 의존성 주입(DI), Generic Host를 조합하여 유닛 테스트와 통합 테스트가 용이한 아키텍처를 설계합니다.
5. **실전 품질 보장**: 성능 최적화, 오류 처리, 보안 고려사항을 포함하여 상용 애플리케이션에 필요한 모든 요소를 다룹니다.

이러한 접근 방식은 단순한 데이터 조회를 넘어 **페이징, 검색, 정렬, 낙관적 업데이트, 오류 처리, 작업 취소** 등 현실적인 요구사항을 모두 아우르는 확장 가능한 기반을 제공하며, 장기적인 유지보수성과 코드 품질을 크게 향상시킵니다.