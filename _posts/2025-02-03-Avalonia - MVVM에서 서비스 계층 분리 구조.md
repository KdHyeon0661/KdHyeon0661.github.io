---
layout: post
title: Avalonia - MVVM에서 서비스 계층 분리 구조
date: 2025-02-03 21:20:23 +0900
category: Avalonia
---
# Avalonia MVVM에서 서비스 계층 분리 구조 (Service / Repository / Unit of Work / Mapper / Caching)

## 0) 설계 원칙 요약

- **SRP / SoC**: UI(상태/이벤트), 도메인 유즈케이스, 데이터 접근을 분리한다.
- **DI**: 모든 경계(서비스/리포지토리/매퍼)를 인터페이스화 → 테스트/교체 용이.
- **계층 경계의 명확화**  
  - ViewModel: 화면/상태/커맨드/네비게이션  
  - Service: 유즈케이스 조합, 트랜잭션/정합성/캐시 정책  
  - Repository: 구체 저장소(API/DB/파일/메모리 등) 접근
- **오프라인/캐시/동기화**: API 실패 시 로컬 데이터 사용, 재시도 정책, 캐시 만료(TTL).
- **검증/에러 처리**: 입력/도메인 검증은 Service, 저장소 오류는 Repository에서 캡슐화.
- **테스트**: VM 테스트는 Service를 모킹, Service 테스트는 Repository를 모킹.

---

## 1) 참조 프로젝트 구조(확장형)

```
MyApp/
├── App.axaml / App.axaml.cs
├── Models/
│   ├── User.cs                   // 도메인 모델
│   ├── PagedResult.cs            // 페이지네이션 모델
│   └── Errors.cs                 // 도메인/인프라 오류 표현
├── Dtos/
│   └── UserDto.cs                // API/DB 전송용 DTO
├── Mapping/
│   └── IUserMapper.cs            // Mapper 인터페이스
│   └── UserMapper.cs             // 수동 매핑 or AutoMapper 대체 가능
├── Repositories/
│   ├── IUserRepository.cs
│   ├── IUnitOfWork.cs
│   ├── api/
│   │   └── UserApiRepository.cs  // 원격
│   └── sqlite/
│       ├── SqliteUnitOfWork.cs   // 트랜잭션/커넥션 수명
│       └── UserSqliteRepository.cs
├── Services/
│   ├── IUserService.cs
│   ├── UserService.cs
│   ├── IClock.cs / SystemClock.cs // 시간 추상화(테스트/TTL)
│   ├── ICache.cs / MemoryCache.cs // 간단 캐시
│   └── Policies.cs                // Polly 재시도/회로차단 설정
├── ViewModels/
│   └── UserViewModel.cs
├── Views/
│   └── UserView.axaml
└── Tests/
    ├── UserViewModelTests.cs
    └── UserServiceTests.cs
```

---

## 2) 모델·DTO·매퍼

### 2.1 도메인 모델

```csharp
// Models/User.cs
public sealed class User
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
    public string Email { get; init; } = "";

    public bool IsValid(out string? reason)
    {
        if (string.IsNullOrWhiteSpace(Name)) { reason = "Name is required."; return false; }
        if (string.IsNullOrWhiteSpace(Email) || !Email.Contains('@')) { reason = "Invalid email."; return false; }
        reason = null; return true;
    }
}
```

### 2.2 DTO

```csharp
// Dtos/UserDto.cs
public sealed class UserDto
{
    public int id { get; set; }
    public string? name { get; set; }
    public string? email { get; set; }
}
```

### 2.3 페이지네이션 공통 모델

```csharp
// Models/PagedResult.cs
public sealed class PagedResult<T>
{
    public IReadOnlyList<T> Items { get; init; } = Array.Empty<T>();
    public int TotalCount { get; init; }
    public int Page { get; init; }
    public int PageSize { get; init; }
}
```

### 2.4 매퍼

```csharp
// Mapping/IUserMapper.cs
public interface IUserMapper
{
    User ToDomain(UserDto dto);
    UserDto ToDto(User model);
}
```

```csharp
// Mapping/UserMapper.cs
public sealed class UserMapper : IUserMapper
{
    public User ToDomain(UserDto dto) => new()
    {
        Id = dto.id,
        Name = dto.name ?? "",
        Email = dto.email ?? ""
    };

    public UserDto ToDto(User model) => new()
    {
        id = model.Id,
        name = model.Name,
        email = model.Email
    };
}
```

> AutoMapper를 사용할 수도 있으나, **바운더리 명시성**과 **성능/디버그 용이성** 측면에서 수동 매핑을 권장하는 케이스도 많다.

---

## 3) 저장소 계층(Repository / Unit of Work)

### 3.1 인터페이스

```csharp
// Repositories/IUserRepository.cs
public interface IUserRepository
{
    Task<User?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<PagedResult<User>> GetAllAsync(int page, int pageSize, string? keyword, CancellationToken ct = default);
    Task<int> UpsertAsync(User user, CancellationToken ct = default);   // Insert or Update
    Task<int> DeleteAsync(int id, CancellationToken ct = default);
}
```

```csharp
// Repositories/IUnitOfWork.cs
public interface IUnitOfWork : IAsyncDisposable
{
    Task CommitAsync(CancellationToken ct = default);
    Task RollbackAsync(CancellationToken ct = default);
}
```

### 3.2 API 저장소 구현(원격 데이터 소스)

```csharp
// Repositories/api/UserApiRepository.cs
using System.Net.Http.Json;

public sealed class UserApiRepository : IUserRepository
{
    private readonly HttpClient _http;
    private readonly IUserMapper _mapper;

    public UserApiRepository(HttpClient http, IUserMapper mapper)
    {
        _http = http; _mapper = mapper;
    }

    public async Task<User?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        using var res = await _http.GetAsync($"/api/users/{id}", ct);
        if (!res.IsSuccessStatusCode) return null;
        var dto = await res.Content.ReadFromJsonAsync<UserDto>(cancellationToken: ct);
        return dto is null ? null : _mapper.ToDomain(dto);
    }

    public async Task<PagedResult<User>> GetAllAsync(int page, int pageSize, string? keyword, CancellationToken ct = default)
    {
        var url = $"/api/users?page={page}&pageSize={pageSize}&q={Uri.EscapeDataString(keyword ?? "")}";
        using var res = await _http.GetAsync(url, ct);
        res.EnsureSuccessStatusCode();
        var list = await res.Content.ReadFromJsonAsync<List<UserDto>>(cancellationToken: ct) ?? new();
        // 총수는 헤더나 별도 엔드포인트에서 받을 수 있음(예시로 목록 길이 사용)
        var items = list.Select(_mapper.ToDomain).ToList();
        return new PagedResult<User> { Items = items, TotalCount = items.Count, Page = page, PageSize = pageSize };
    }

    public async Task<int> UpsertAsync(User user, CancellationToken ct = default)
    {
        var dto = _mapper.ToDto(user);
        HttpResponseMessage res;
        if (user.Id == 0)
            res = await _http.PostAsJsonAsync("/api/users", dto, ct);
        else
            res = await _http.PutAsJsonAsync($"/api/users/{user.Id}", dto, ct);

        res.EnsureSuccessStatusCode();
        // 생성된 ID를 응답으로 돌려주는 API라면 파싱해서 반환
        return user.Id == 0 ? int.Parse(await res.Content.ReadAsStringAsync(ct)) : user.Id;
    }

    public async Task<int> DeleteAsync(int id, CancellationToken ct = default)
    {
        using var res = await _http.DeleteAsync($"/api/users/{id}", ct);
        res.EnsureSuccessStatusCode();
        return id;
    }
}
```

> API 저장소는 네트워크 예외/상태코드를 캡슐화한다. 서비스 계층은 “성공/실패” 의미에만 집중하도록 한다.

### 3.3 SQLite + Dapper 저장소(로컬 캐시/오프라인)

```csharp
// Repositories/sqlite/SqliteUnitOfWork.cs
using Dapper;
using Microsoft.Data.Sqlite;
using System.Data;

public sealed class SqliteUnitOfWork : IUnitOfWork
{
    private readonly SqliteConnection _conn;
    private SqliteTransaction? _tx;

    public IDbConnection Connection => _conn;
    public IDbTransaction? Transaction => _tx;

    public SqliteUnitOfWork(string dbPath = "app.db")
    {
        _conn = new SqliteConnection($"Data Source={dbPath};Cache=Shared");
        _conn.Open();
        _tx = _conn.BeginTransaction();
    }

    public Task CommitAsync(CancellationToken ct = default)
    {
        _tx?.Commit();
        _tx?.Dispose();
        _tx = _conn.BeginTransaction();
        return Task.CompletedTask;
    }

    public Task RollbackAsync(CancellationToken ct = default)
    {
        _tx?.Rollback();
        _tx?.Dispose();
        _tx = _conn.BeginTransaction();
        return Task.CompletedTask;
    }

    public ValueTask DisposeAsync()
    {
        _tx?.Dispose();
        _conn.Dispose();
        return ValueTask.CompletedTask;
    }
}
```

```csharp
// Repositories/sqlite/UserSqliteRepository.cs
using Dapper;

public sealed class UserSqliteRepository : IUserRepository
{
    private readonly SqliteUnitOfWork _uow;

    public UserSqliteRepository(SqliteUnitOfWork uow)
    {
        _uow = uow;
        _uow.Connection.Execute("""
            CREATE TABLE IF NOT EXISTS Users(
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Name TEXT NOT NULL,
                Email TEXT NOT NULL
            );
        """);
    }

    public async Task<User?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        var row = await _uow.Connection.QuerySingleOrDefaultAsync<(int Id, string Name, string Email)>(
            "SELECT Id,Name,Email FROM Users WHERE Id=@id", new { id }, _uow.Transaction);
        return row.Equals(default((int, string, string))) ? null : new User { Id = row.Id, Name = row.Name, Email = row.Email };
    }

    public async Task<PagedResult<User>> GetAllAsync(int page, int pageSize, string? keyword, CancellationToken ct = default)
    {
        var skip = (page - 1) * pageSize;
        keyword ??= "";
        var where = string.IsNullOrWhiteSpace(keyword) ? "" : "WHERE Name LIKE @kw OR Email LIKE @kw";
        var kw = $"%{keyword}%";

        var items = (await _uow.Connection.QueryAsync<User>(
            $"SELECT Id,Name,Email FROM Users {where} ORDER BY Id DESC LIMIT @pageSize OFFSET @skip",
            new { kw, pageSize, skip }, _uow.Transaction)).ToList();

        var total = await _uow.Connection.ExecuteScalarAsync<int>(
            $"SELECT COUNT(*) FROM Users {where}", new { kw }, _uow.Transaction);

        return new PagedResult<User> { Items = items, TotalCount = total, Page = page, PageSize = pageSize };
    }

    public async Task<int> UpsertAsync(User user, CancellationToken ct = default)
    {
        if (user.Id == 0)
        {
            var id = await _uow.Connection.ExecuteScalarAsync<long>(
                "INSERT INTO Users(Name,Email) VALUES(@Name,@Email); SELECT last_insert_rowid();",
                new { user.Name, user.Email }, _uow.Transaction);
            return (int)id;
        }
        else
        {
            await _uow.Connection.ExecuteAsync(
                "UPDATE Users SET Name=@Name, Email=@Email WHERE Id=@Id",
                new { user.Name, user.Email, user.Id }, _uow.Transaction);
            return user.Id;
        }
    }

    public Task<int> DeleteAsync(int id, CancellationToken ct = default)
        => _uow.Connection.ExecuteAsync("DELETE FROM Users WHERE Id=@id", new { id }, _uow.Transaction)
           .ContinueWith(_ => id, ct);
}
```

---

## 4) 서비스 계층(유즈케이스/정책/캐시)

### 4.1 기본 인터페이스

```csharp
// Services/IUserService.cs
public interface IUserService
{
    Task<User?> GetUserAsync(int id, CancellationToken ct = default);
    Task<PagedResult<User>> SearchAsync(int page, int pageSize, string? keyword, CancellationToken ct = default);
    Task<int> SaveAsync(User user, CancellationToken ct = default);
    Task<int> RemoveAsync(int id, CancellationToken ct = default);
}
```

### 4.2 시간/캐시 추상화

```csharp
// Services/IClock.cs
public interface IClock { DateTimeOffset Now { get; } }
public sealed class SystemClock : IClock { public DateTimeOffset Now => DateTimeOffset.UtcNow; }
```

```csharp
// Services/ICache.cs
public interface ICache
{
    T? Get<T>(string key);
    void Set<T>(string key, T value, TimeSpan ttl);
    void Remove(string key);
}
```

```csharp
// Services/MemoryCache.cs
public sealed class MemoryCache : ICache
{
    private sealed record Entry(object Value, DateTimeOffset ExpireAt);
    private readonly Dictionary<string, Entry> _store = new();

    public T? Get<T>(string key)
    {
        if (!_store.TryGetValue(key, out var e)) return default;
        if (DateTimeOffset.UtcNow >= e.ExpireAt) { _store.Remove(key); return default; }
        return (T)e.Value;
    }

    public void Set<T>(string key, T value, TimeSpan ttl)
        => _store[key] = new Entry(value!, DateTimeOffset.UtcNow.Add(ttl));

    public void Remove(string key) { _store.Remove(key); }
}
```

> TTL 정책은 **서비스 계층**이 소유한다. 예: 목록 30초, 단건 60초 등.

### 4.3 Polly 정책

```csharp
// Services/Policies.cs
using Polly;
using Polly.Extensions.Http;
using System.Net;

public static class Policies
{
    public static IAsyncPolicy<HttpResponseMessage> TransientHttpPolicy =>
        HttpPolicyExtensions
            .HandleTransientHttpError()
            .OrResult(r => r.StatusCode == HttpStatusCode.TooManyRequests)
            .WaitAndRetryAsync(new[]
            {
                TimeSpan.FromMilliseconds(200),
                TimeSpan.FromMilliseconds(500),
                TimeSpan.FromSeconds(1)
            });
}
```

### 4.4 서비스 구현: API 우선, 실패 시 로컬 폴백 + 캐시

```csharp
// Services/UserService.cs
public sealed class UserService : IUserService
{
    private readonly IUserRepository _apiRepo;
    private readonly IUserRepository _localRepo;
    private readonly IUnitOfWork _localUow;
    private readonly ICache _cache;
    private readonly IClock _clock;

    private static readonly TimeSpan SingleTtl = TimeSpan.FromSeconds(60);
    private static readonly TimeSpan ListTtl   = TimeSpan.FromSeconds(30);

    public UserService(
        UserApiRepository apiRepo,    // 명시적으로 타입 주입(인터페이스도 가능)
        UserSqliteRepository localRepo,
        SqliteUnitOfWork localUow,
        ICache cache,
        IClock clock)
    {
        _apiRepo = apiRepo; _localRepo = localRepo; _localUow = localUow; _cache = cache; _clock = clock;
    }

    public async Task<User?> GetUserAsync(int id, CancellationToken ct = default)
    {
        var key = $"user:{id}";
        if (_cache.Get<User>(key) is { } cached) return cached;

        try
        {
            // 1) API 우선
            var u = await _apiRepo.GetByIdAsync(id, ct);
            if (u is not null)
            {
                _cache.Set(key, u, SingleTtl);
                await _localRepo.UpsertAsync(u, ct);
                await _localUow.CommitAsync(ct);
                return u;
            }
        }
        catch { /* 로깅 */ }

        // 2) 폴백: 로컬
        var local = await _localRepo.GetByIdAsync(id, ct);
        if (local is not null) _cache.Set(key, local, SingleTtl);
        return local;
    }

    public async Task<PagedResult<User>> SearchAsync(int page, int pageSize, string? keyword, CancellationToken ct = default)
    {
        var key = $"users:{page}:{pageSize}:{keyword}";
        if (_cache.Get<PagedResult<User>>(key) is { } cached) return cached;

        try
        {
            var list = await _apiRepo.GetAllAsync(page, pageSize, keyword, ct);
            _cache.Set(key, list, ListTtl);

            // 로컬 동기화(단순히 최신 페이지만 반영)
            foreach (var u in list.Items) await _localRepo.UpsertAsync(u, ct);
            await _localUow.CommitAsync(ct);
            return list;
        }
        catch { /* 로깅 */ }

        // 폴백: 로컬
        var local = await _localRepo.GetAllAsync(page, pageSize, keyword, ct);
        _cache.Set(key, local, ListTtl);
        return local;
    }

    public async Task<int> SaveAsync(User user, CancellationToken ct = default)
    {
        if (!user.IsValid(out var why)) throw new InvalidOperationException(why);

        // API 우선
        try
        {
            var id = await _apiRepo.UpsertAsync(user, ct);
            var merged = user with { Id = id };
            await _localRepo.UpsertAsync(merged, ct);
            await _localUow.CommitAsync(ct);

            // 캐시 무효화
            _cache.Remove($"user:{id}");
            // 목록 캐시 키 전략: prefix purge (간단화를 위해 전체 목록 캐시를 비운다)
            // 실제로는 키 인덱스 보관 후 타겟 무효화 구현
            return id;
        }
        catch
        {
            // 오프라인 저장 전략(선택): 로컬 큐/아웃박스에 저장 후 백그라운드 동기화
            await _localRepo.UpsertAsync(user, ct);
            await _localUow.CommitAsync(ct);
            return user.Id;
        }
    }

    public async Task<int> RemoveAsync(int id, CancellationToken ct = default)
    {
        try
        {
            await _apiRepo.DeleteAsync(id, ct);
            await _localRepo.DeleteAsync(id, ct);
            await _localUow.CommitAsync(ct);
            _cache.Remove($"user:{id}");
            return id;
        }
        catch
        {
            // 오프라인 삭제 예약(플래그) 등 처리 가능
            await _localRepo.DeleteAsync(id, ct);
            await _localUow.CommitAsync(ct);
            _cache.Remove($"user:{id}");
            return id;
        }
    }
}
```

> **핵심**  
> - 서비스는 **정책 소유자**: 캐시 TTL/폴백/동기화/검증/트랜잭션.  
> - 저장소는 **구현 상세 캡슐화**: API/SQL 쿼리/파일 I/O.

---

## 5) ViewModel과의 결합

```csharp
// ViewModels/UserViewModel.cs
using ReactiveUI;
using System.Collections.ObjectModel;
using System.Reactive;
using System.Reactive.Linq;

public sealed class UserViewModel : ReactiveObject
{
    private readonly IUserService _svc;

    public UserViewModel(IUserService svc)
    {
        _svc = svc;

        var canSearch = this.WhenAnyValue(x => x.Keyword, k => !string.IsNullOrWhiteSpace(k));
        SearchCommand = ReactiveCommand.CreateFromTask(async () =>
        {
            IsBusy = true;
            try
            {
                var result = await _svc.SearchAsync(Page, PageSize, Keyword);
                Users = new ObservableCollection<User>(result.Items);
                Total = result.TotalCount;
            }
            finally { IsBusy = false; }
        }, canSearch);

        SaveCommand = ReactiveCommand.CreateFromTask(async () =>
        {
            IsBusy = true;
            try
            {
                var model = new User { Id = EditId, Name = EditName, Email = EditEmail };
                var id = await _svc.SaveAsync(model);
                EditId = id;
                await SearchCommand.Execute(); // 목록 갱신
            }
            finally { IsBusy = false; }
        });

        DeleteCommand = ReactiveCommand.CreateFromTask<int>(async id =>
        {
            IsBusy = true;
            try
            {
                await _svc.RemoveAsync(id);
                await SearchCommand.Execute();
            }
            finally { IsBusy = false; }
        });
    }

    // 조회 상태
    private ObservableCollection<User> _users = new();
    public ObservableCollection<User> Users { get => _users; set => this.RaiseAndSetIfChanged(ref _users, value); }

    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 20;
    private int _total;
    public int Total { get => _total; set => this.RaiseAndSetIfChanged(ref _total, value); }

    public string? Keyword { get; set; } = "";

    // 편집 상태
    public int EditId { get; set; }
    public string EditName { get; set; } = "";
    public string EditEmail { get; set; } = "";

    private bool _isBusy;
    public bool IsBusy { get => _isBusy; set => this.RaiseAndSetIfChanged(ref _isBusy, value); }

    public ReactiveCommand<Unit, Unit> SearchCommand { get; }
    public ReactiveCommand<Unit, Unit> SaveCommand { get; }
    public ReactiveCommand<int, Unit> DeleteCommand { get; }
}
```

### 간단 View

```xml
<!-- Views/UserView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui" x:Class="MyApp.Views.UserView">
  <StackPanel Margin="16" Spacing="8">
    <StackPanel Orientation="Horizontal" Spacing="8">
      <TextBox Width="200" Watermark="검색어" Text="{Binding Keyword}"/>
      <Button Content="검색" Command="{Binding SearchCommand}"/>
      <ProgressBar IsIndeterminate="True" IsVisible="{Binding IsBusy}" Height="6" Width="80"/>
    </StackPanel>

    <DataGrid Items="{Binding Users}" AutoGenerateColumns="False" Height="220">
      <DataGrid.Columns>
        <DataGridTextColumn Header="ID" Binding="{Binding Id}"/>
        <DataGridTextColumn Header="Name" Binding="{Binding Name}"/>
        <DataGridTextColumn Header="Email" Binding="{Binding Email}"/>
      </DataGrid.Columns>
    </DataGrid>

    <StackPanel Orientation="Horizontal" Spacing="8">
      <TextBox Width="150" Watermark="Name" Text="{Binding EditName}"/>
      <TextBox Width="200" Watermark="Email" Text="{Binding EditEmail}"/>
      <Button Content="저장" Command="{Binding SaveCommand}"/>
    </StackPanel>
  </StackPanel>
</UserControl>
```

---

## 6) DI 구성(App.axaml.cs)

```csharp
// App.axaml.cs (중요 부분)
using Microsoft.Extensions.DependencyInjection;
using Polly;

public class App : Application
{
    public static IServiceProvider Services { get; private set; } = default!;

    public override void OnFrameworkInitializationCompleted()
    {
        var sc = new ServiceCollection();

        // 시간/캐시
        sc.AddSingleton<IClock, SystemClock>();
        sc.AddSingleton<ICache, MemoryCache>();

        // 매퍼
        sc.AddSingleton<IUserMapper, UserMapper>();

        // SQLite UnitOfWork & Repo
        sc.AddSingleton<SqliteUnitOfWork>(); // 앱 수명 내 공유(간단 예)
        sc.AddSingleton<UserSqliteRepository>();

        // HttpClient with Polly
        sc.AddHttpClient<UserApiRepository>(client =>
        {
            client.BaseAddress = new Uri("https://api.example.com");
            client.Timeout = TimeSpan.FromSeconds(10);
        }).AddPolicyHandler(Policies.TransientHttpPolicy);

        // 서비스
        sc.AddSingleton<IUserService, UserService>();

        // ViewModel
        sc.AddTransient<UserViewModel>();

        Services = sc.BuildServiceProvider();

        var win = new Window { Content = new Views.UserView(), DataContext = Services.GetRequiredService<UserViewModel>() };
        win.Show();

        base.OnFrameworkInitializationCompleted();
    }
}
```

---

## 7) 고급 설계 포인트

### 7.1 정렬/필터/페이지네이션 기준의 서비스 책임
- Repository는 **기술적 쿼리 능력**(WHERE/ORDER/LIMIT)을 제공.
- Service는 **도메인 규칙**(권한별 필터, 기본 정렬, 클리닝)과 **페이징 UI 정책**(기본 page=1, pageSize=20)을 소유.

### 7.2 입력/도메인 검증 위치
- **ViewModel**: 사용자 피드백용 **UI 레벨** 검증(빈 값/포맷).
- **Service**: 시스템 일관성을 보장하는 **도메인 검증**(User.IsValid 등)을 반드시 재검증.
- **Repository**: 무결성 제약(UNIQUE 등) 실패 시 **인프라 오류**로 승격 → Service에서 적절히 메시지 변환.

### 7.3 캐시 만료/동기화 수식
만료 시각을 \( T_\text{exp} \), 현재 시각을 \( t \), TTL을 \( \tau \)라 하면  
캐시 유효 조건은
$$
t - T_\text{set} < \tau
$$
이며, API 성공 시 **쓰기 직후 캐시 갱신**으로 일관성을 유지한다.

### 7.4 오프라인 전략
- 모든 쓰기(Upsert/Delete)는 로컬에 반영 후, 백그라운드 동기화 큐(Outbox)로 서버 반영 가능.
- 충돌 정책: “서버 우선”, “최신 타임스탬프 우선”, “필드 병합” 등 결정.

### 7.5 사양 패턴/쿼리 오브젝트(선택)
- 복잡한 검색/정렬 조건을 `UserQuery` 객체로 캡슐화 → Repository가 Query를 해석.
- Service는 Query 빌더를 제공하여 UI 요구를 단순화.

---

## 8) 테스트 전략

### 8.1 ViewModel 테스트: Service 모킹

```csharp
// Tests/UserViewModelTests.cs
using Moq;
using FluentAssertions;
using Xunit;

public class UserViewModelTests
{
    [Fact]
    public async Task SearchCommand_FillsUsers_FromService()
    {
        var svc = new Mock<IUserService>();
        svc.Setup(s => s.SearchAsync(1, 20, "kim", default))
           .ReturnsAsync(new PagedResult<User>
           {
               Items = new[] { new User { Id=1, Name="Kim", Email="k@a.com" } },
               Page = 1, PageSize = 20, TotalCount = 1
           });

        var vm = new UserViewModel(svc.Object) { Keyword = "kim", Page=1, PageSize=20 };
        await vm.SearchCommand.Execute();

        vm.Users.Count.Should().Be(1);
        vm.Users[0].Name.Should().Be("Kim");
    }
}
```

### 8.2 Service 테스트: Repository 모킹

```csharp
// Tests/UserServiceTests.cs
public class UserServiceTests
{
    [Fact]
    public async Task GetUserAsync_UsesApiThenCaches_AndSyncsLocal()
    {
        var api = new Mock<IUserRepository>();
        var local = new Mock<IUserRepository>();
        var uow = new Mock<IUnitOfWork>();
        var cache = new MemoryCache();
        var clock = new SystemClock();

        api.Setup(a => a.GetByIdAsync(1, default))
           .ReturnsAsync(new User { Id=1, Name="A", Email="a@a.com" });

        var svc = new UserService(
            (UserApiRepository)Activator.CreateInstance(typeof(UserApiRepository), true)!,
            (UserSqliteRepository)Activator.CreateInstance(typeof(UserSqliteRepository), true)!,
            (SqliteUnitOfWork)Activator.CreateInstance(typeof(SqliteUnitOfWork), "test.db")!,
            cache, clock);

        // 위 코드는 실제 타입 주입용이므로, 실무에선 생성자 오버로드를 두거나
        // 인터페이스 기반 주입 + 테스트 더블 구현을 권장한다.
        // (여기서는 개념을 보여주기 위한 단순 예시)

        // 개념상 검증 포인트:
        // 1) API 호출 후 결과 캐시 존재
        cache.Get<User>("user:1").Should().NotBeNull();
    }
}
```

> 실제 테스트에서는 `UserService` 생성자를 **인터페이스 기반**으로 구성하고, **Mock<IUserRepository>**를 직접 주입하는 구조가 이상적이다.

---

## 9) CQRS/유즈케이스 분해(선택)

읽기/쓰기 파이프라인을 분리하여 **읽기 최적화(캐시/인덱스)**와 **쓰기 검증/트랜잭션**을 독립적으로 확장할 수 있다.  
예) `IUserQueries`, `IUserCommands`로 인터페이스 분리.

---

## 10) 오류 모델

```csharp
// Models/Errors.cs
public abstract record AppError(string Code, string Message);

public sealed record NotFoundError(string Resource, string Key)
    : AppError("not_found", $"{Resource}({Key}) not found");

public sealed record ValidationError(string Field, string Detail)
    : AppError("validation", $"{Field}: {Detail}");

public sealed record InfraError(string Operation, string Detail)
    : AppError("infra", $"{Operation}: {Detail}");
```

Service는 저장소에서 발생한 예외를 `InfraError` 등으로 **의미화**하여 ViewModel이 사용자가 이해 가능한 메시지로 매핑하도록 돕는다.

---

## 11) 성능·운영 팁

- **HttpClient 재사용**: DI 컨테이너로 관리, 소켓 고갈 방지
- **Polly**: 429/5xx 재시도, 서킷브레이커로 폭주 방지
- **로깅**: Serilog/NLog로 **계층별**(Repo/Service/VM) 로그 분리
- **측정**: Stopwatch/Activities(OpenTelemetry)로 API/DB 지연 파악
- **캐시 키 전략**: prefix+인덱스 키로 부분 무효화 구현

---

## 12) 요약 표

| 계층 | 책임 | 구현 포인트 | 테스트 포인트 |
|------|------|------------|---------------|
| ViewModel | UI 상태/커맨드/바인딩 | ReactiveCommand, IsBusy, 오류 표시 | Service 모킹으로 플로우 검증 |
| Service | 유즈케이스, 검증, 캐시, 폴백 | TTL, 트랜잭션, 오프라인 동기화 | Repository 모킹, 정책 검증 |
| Repository | 데이터 소스 접근(API/DB/파일) | 예외 캡슐화, 쿼리 최적화 | 쿼리 단위 테스트(로컬 DB) |
| Mapper | DTO↔모델 | 명시 매핑(성능/가독) | 필드 매핑 누락 검증 |

---

## 13) 부록: 간단 수학(캐시 히트율 근사)

요청 빈도를 \( \lambda \), 캐시 TTL을 \( \tau \), 원천 적중 확률을 \( p \)라 할 때, 단순 포아송 근사에서 캐시 적중 확률 \( H \)는

$$
H \approx 1 - e^{-\lambda \tau} (1 - p)
$$

으로 근사할 수 있다. \( \lambda \tau \)가 클수록(요청이 잦고 TTL이 길수록) 캐시 히트율은 상승한다.  
서비스 계층에서 TTL을 조정할 때, 대략적인 영향도를 가늠하는 직관으로 활용할 수 있다.

---

## 14) 결론

- **ViewModel–Service–Repository** 분리는 UI/도메인/인프라의 콘크리트 결합을 끊고, 테스트성·유지보수성·확장성을 극대화한다.
- **Service**는 캐시·오프라인·동기화·검증·트랜잭션·정책의 주체이며, **Repository**는 데이터 접근 구현에만 집중한다.
- DI·Polly·로깅·매퍼·UoW를 적절히 조합하면 **실서비스 품질**의 Avalonia MVVM 아키텍처를 수립할 수 있다.

> 다음 단계: Outbox 패턴(쓰기 동기화), Change Tracking(로컬 편집 이력), Role/Policy 기반 읽기 필터, AutoMapper/Mapster 도입 시 장단점 비교, OpenTelemetry 연동으로 성능/에러 관측.