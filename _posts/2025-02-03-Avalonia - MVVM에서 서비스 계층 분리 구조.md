---
layout: post
title: Avalonia - MVVM에서 서비스 계층 분리 구조
date: 2025-02-03 21:20:23 +0900
category: Avalonia
---
# Avalonia MVVM에서 서비스 계층 분리 구조

애플리케이션이 커질수록 ViewModel이 비대해지고, 데이터 접근, 비즈니스 규칙, 캐시 정책 등이 뒤섞이기 쉽습니다. 이를 방지하기 위해 **서비스 계층(Service Layer)** 을 도입해 **관심사 분리(Separation of Concerns)** 를 명확히 하는 구조가 필요합니다. 이 글에서는 ViewModel, Service, Repository, Unit of Work, Mapper, Cache 계층을 어떻게 나누고 조립할지, 초중급 개발자 관점에서 실전 예제와 함께 설명합니다.

---

## 설계 원칙

- **단일 책임 원칙(SRP)**: ViewModel은 UI 상태와 사용자 인터랙션에만 집중합니다. Service는 비즈니스 유즈케이스, 검증, 트랜잭션, 캐시 정책을 담당합니다. Repository는 데이터 소스(API, DB, 파일) 접근만 캡슐화합니다.
- **의존성 역전(DIP)**: 고수준 모듈(Service)이 저수준 모듈(Repository)에 의존하지 않도록 인터페이스로 추상화합니다. DI 컨테이너를 통해 구현체를 주입합니다.
- **테스트 용이성**: 각 계층을 인터페이스로 분리하면 단위 테스트에서 모킹(Mocking)이 쉬워집니다.
- **캐시 및 폴백 전략**: Service 계층이 캐시 TTL, 오프라인 폴백, 재시도 정책 등을 중앙에서 관리합니다.

---

## 프로젝트 구조

```
MyApp/
├── Models/               # 도메인 모델 (불변 또는 단순 POCO)
├── Dtos/                 # 데이터 전송 객체 (API/DB 스키마)
├── Mapping/              # DTO ↔ 도메인 매핑
├── Repositories/         # 데이터 접근 인터페이스 및 구현
├── Services/             # 비즈니스 로직, 캐시, 정책
├── ViewModels/           # UI 상태 및 커맨드
├── Views/                # XAML 뷰
└── App.axaml.cs          # DI 컨테이너 구성
```

---

## 도메인 모델, DTO, 매퍼

### 도메인 모델

도메인 모델은 애플리케이션의 핵심 비즈니스 객체입니다. 여기서는 불변(immutable) 성향을 가지되, UI 바인딩 편의를 위해 가변 프로퍼티를 사용할 수 있습니다. 검증 로직은 모델 내부에 두는 것이 좋습니다.

```csharp
// Models/User.cs
public sealed class User
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
    public string Email { get; init; } = "";

    public bool IsValid(out string? reason)
    {
        if (string.IsNullOrWhiteSpace(Name))
        {
            reason = "이름을 입력하세요.";
            return false;
        }
        if (string.IsNullOrWhiteSpace(Email) || !Email.Contains('@'))
        {
            reason = "올바른 이메일 형식이 아닙니다.";
            return false;
        }
        reason = null;
        return true;
    }
}
```

### DTO (Data Transfer Object)

네트워크 전송이나 DB 저장에 최적화된 구조입니다.

```csharp
// Dtos/UserDto.cs
public sealed class UserDto
{
    public int id { get; set; }
    public string? name { get; set; }
    public string? email { get; set; }
}
```

### 매퍼 (Mapper)

DTO ↔ 도메인 변환을 전담합니다. 수동 매핑을 사용하면 디버깅이 쉽고 성능 저하가 없습니다. AutoMapper 같은 라이브러리를 사용할 수도 있습니다.

```csharp
// Mapping/IUserMapper.cs
public interface IUserMapper
{
    User ToDomain(UserDto dto);
    UserDto ToDto(User domain);
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

    public UserDto ToDto(User domain) => new()
    {
        id = domain.Id,
        name = domain.Name,
        email = domain.Email
    };
}
```

---

## Repository 계층

Repository는 데이터 소스에 대한 CRUD 작업을 캡슐화합니다. 여기서는 API와 SQLite 두 가지 구현을 예로 듭니다.

### 공통 인터페이스

```csharp
// Repositories/IUserRepository.cs
public interface IUserRepository
{
    Task<User?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<PagedResult<User>> GetAllAsync(int page, int pageSize, string? keyword, CancellationToken ct = default);
    Task<int> UpsertAsync(User user, CancellationToken ct = default);
    Task<int> DeleteAsync(int id, CancellationToken ct = default);
}
```

`PagedResult<T>`는 페이지네이션 정보를 담는 공통 모델입니다.

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

### API Repository (원격)

```csharp
// Repositories/Api/UserApiRepository.cs
public sealed class UserApiRepository : IUserRepository
{
    private readonly HttpClient _http;
    private readonly IUserMapper _mapper;

    public UserApiRepository(HttpClient http, IUserMapper mapper)
    {
        _http = http;
        _mapper = mapper;
    }

    public async Task<User?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        var response = await _http.GetAsync($"/api/users/{id}", ct);
        if (!response.IsSuccessStatusCode) return null;
        var dto = await response.Content.ReadFromJsonAsync<UserDto>(ct);
        return dto is null ? null : _mapper.ToDomain(dto);
    }

    // GetAllAsync, UpsertAsync, DeleteAsync 생략 (유사한 패턴)
}
```

### SQLite Repository (로컬)

SQLite를 사용할 때는 Dapper 같은 마이크로 ORM을 활용합니다. Unit of Work로 트랜잭션을 관리합니다.

```csharp
// Repositories/Sqlite/SqliteUnitOfWork.cs
public sealed class SqliteUnitOfWork : IUnitOfWork, IAsyncDisposable
{
    private readonly SqliteConnection _connection;
    private SqliteTransaction? _transaction;

    public SqliteUnitOfWork(string dbPath = "app.db")
    {
        _connection = new SqliteConnection($"Data Source={dbPath};Cache=Shared");
        _connection.Open();
        _transaction = _connection.BeginTransaction();
    }

    public IDbConnection Connection => _connection;
    public IDbTransaction? Transaction => _transaction;

    public Task CommitAsync(CancellationToken ct = default)
    {
        _transaction?.Commit();
        _transaction?.Dispose();
        _transaction = _connection.BeginTransaction();
        return Task.CompletedTask;
    }

    public Task RollbackAsync(CancellationToken ct = default)
    {
        _transaction?.Rollback();
        _transaction?.Dispose();
        _transaction = _connection.BeginTransaction();
        return Task.CompletedTask;
    }

    public async ValueTask DisposeAsync()
    {
        _transaction?.Dispose();
        await _connection.DisposeAsync();
    }
}
```

```csharp
// Repositories/Sqlite/UserSqliteRepository.cs
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
            "SELECT Id, Name, Email FROM Users WHERE Id = @id",
            new { id }, _uow.Transaction);
        if (row == default) return null;
        return new User { Id = row.Id, Name = row.Name, Email = row.Email };
    }

    // GetAllAsync, UpsertAsync, DeleteAsync 구현 (유사)
}
```

---

## 서비스 계층 (Service)

Service는 **유즈케이스**를 조합하고, 캐시, 재시도, 오프라인 폴백, 트랜잭션 등의 **정책**을 적용합니다.

### 시간 추상화 (테스트 용이)

```csharp
// Services/IClock.cs
public interface IClock
{
    DateTimeOffset Now { get; }
}

public sealed class SystemClock : IClock
{
    public DateTimeOffset Now => DateTimeOffset.UtcNow;
}
```

### 캐시 추상화

```csharp
// Services/ICache.cs
public interface ICache
{
    T? Get<T>(string key);
    void Set<T>(string key, T value, TimeSpan ttl);
    void Remove(string key);
}

// Services/MemoryCache.cs
public sealed class MemoryCache : ICache
{
    private readonly Dictionary<string, (object Value, DateTimeOffset ExpireAt)> _store = new();

    public T? Get<T>(string key)
    {
        if (_store.TryGetValue(key, out var entry) && entry.ExpireAt > DateTimeOffset.UtcNow)
            return (T)entry.Value;
        _store.Remove(key);
        return default;
    }

    public void Set<T>(string key, T value, TimeSpan ttl)
        => _store[key] = (value!, DateTimeOffset.UtcNow.Add(ttl));

    public void Remove(string key) => _store.Remove(key);
}
```

### 폴백 정책 (예: Polly 재시도)

```csharp
// Services/Policies.cs
using Polly;
using Polly.Extensions.Http;

public static class Policies
{
    public static IAsyncPolicy<HttpResponseMessage> TransientHttpPolicy =>
        HttpPolicyExtensions
            .HandleTransientHttpError()
            .WaitAndRetryAsync(new[]
            {
                TimeSpan.FromMilliseconds(200),
                TimeSpan.FromMilliseconds(500),
                TimeSpan.FromSeconds(1)
            });
}
```

### 서비스 구현 (API 우선 + 로컬 폴백 + 캐시)

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
    private static readonly TimeSpan ListTtl = TimeSpan.FromSeconds(30);

    public UserService(
        UserApiRepository apiRepo,
        UserSqliteRepository localRepo,
        SqliteUnitOfWork localUow,
        ICache cache,
        IClock clock)
    {
        _apiRepo = apiRepo;
        _localRepo = localRepo;
        _localUow = localUow;
        _cache = cache;
        _clock = clock;
    }

    public async Task<User?> GetUserAsync(int id, CancellationToken ct = default)
    {
        var cacheKey = $"user:{id}";
        if (_cache.Get<User>(cacheKey) is { } cached)
            return cached;

        try
        {
            var user = await _apiRepo.GetByIdAsync(id, ct);
            if (user != null)
            {
                _cache.Set(cacheKey, user, SingleTtl);
                await _localRepo.UpsertAsync(user, ct);
                await _localUow.CommitAsync(ct);
                return user;
            }
        }
        catch { /* 로깅 */ }

        // API 실패 시 로컬 폴백
        var local = await _localRepo.GetByIdAsync(id, ct);
        if (local != null)
            _cache.Set(cacheKey, local, SingleTtl);
        return local;
    }

    public async Task<PagedResult<User>> SearchAsync(int page, int pageSize, string? keyword, CancellationToken ct = default)
    {
        var cacheKey = $"users:{page}:{pageSize}:{keyword}";
        if (_cache.Get<PagedResult<User>>(cacheKey) is { } cached)
            return cached;

        try
        {
            var result = await _apiRepo.GetAllAsync(page, pageSize, keyword, ct);
            _cache.Set(cacheKey, result, ListTtl);

            // 백그라운드 로컬 동기화 (단순 예)
            foreach (var u in result.Items)
                await _localRepo.UpsertAsync(u, ct);
            await _localUow.CommitAsync(ct);
            return result;
        }
        catch
        {
            var local = await _localRepo.GetAllAsync(page, pageSize, keyword, ct);
            _cache.Set(cacheKey, local, ListTtl);
            return local;
        }
    }

    public async Task<int> SaveAsync(User user, CancellationToken ct = default)
    {
        if (!user.IsValid(out var why))
            throw new InvalidOperationException(why);

        try
        {
            var id = await _apiRepo.UpsertAsync(user, ct);
            var merged = user with { Id = id };
            await _localRepo.UpsertAsync(merged, ct);
            await _localUow.CommitAsync(ct);
            _cache.Remove($"user:{id}");
            // 목록 캐시는 전체 무효화 (간략화)
            return id;
        }
        catch
        {
            // 오프라인 저장: 로컬에만 반영
            var localId = await _localRepo.UpsertAsync(user, ct);
            await _localUow.CommitAsync(ct);
            return localId;
        }
    }

    public async Task<int> DeleteAsync(int id, CancellationToken ct = default)
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
            await _localRepo.DeleteAsync(id, ct);
            await _localUow.CommitAsync(ct);
            _cache.Remove($"user:{id}");
            return id;
        }
    }
}
```

> **핵심**: Service는 API 호출, 로컬 저장, 캐시, 예외 처리, 트랜잭션을 모두 책임집니다. Repository는 단순 데이터 접근에만 집중합니다.

---

## ViewModel과의 결합

ViewModel은 Service 인터페이스에만 의존합니다. UI 상태와 커맨드만 관리합니다.

```csharp
// ViewModels/UserViewModel.cs
public sealed class UserViewModel : ReactiveObject
{
    private readonly IUserService _userService;

    public UserViewModel(IUserService userService)
    {
        _userService = userService;

        SearchCommand = ReactiveCommand.CreateFromTask(SearchAsync);
        SaveCommand = ReactiveCommand.CreateFromTask(SaveAsync);
        DeleteCommand = ReactiveCommand.CreateFromTask<int>(DeleteAsync);
    }

    private ObservableCollection<User> _users = new();
    public ObservableCollection<User> Users
    {
        get => _users;
        set => this.RaiseAndSetIfChanged(ref _users, value);
    }

    private string _keyword = "";
    public string Keyword
    {
        get => _keyword;
        set => this.RaiseAndSetIfChanged(ref _keyword, value);
    }

    private int _page = 1;
    private int _pageSize = 20;
    private int _total;
    // ... 기타 상태 (IsBusy, EditName 등)

    public ReactiveCommand<Unit, Unit> SearchCommand { get; }
    public ReactiveCommand<Unit, Unit> SaveCommand { get; }
    public ReactiveCommand<int, Unit> DeleteCommand { get; }

    private async Task SearchAsync()
    {
        IsBusy = true;
        try
        {
            var result = await _userService.SearchAsync(_page, _pageSize, _keyword);
            Users = new ObservableCollection<User>(result.Items);
            Total = result.TotalCount;
        }
        finally
        {
            IsBusy = false;
        }
    }

    private async Task SaveAsync()
    {
        IsBusy = true;
        try
        {
            var user = new User { Id = EditId, Name = EditName, Email = EditEmail };
            await _userService.SaveAsync(user);
            await SearchAsync();
        }
        finally { IsBusy = false; }
    }

    // DeleteAsync 유사
}
```

---

## DI 구성 (App.axaml.cs)

```csharp
public partial class App : Application
{
    public static IServiceProvider Services { get; private set; } = default!;

    public override void OnFrameworkInitializationCompleted()
    {
        var services = new ServiceCollection();

        // 기본 서비스
        services.AddSingleton<IClock, SystemClock>();
        services.AddSingleton<ICache, MemoryCache>();

        // 매퍼
        services.AddSingleton<IUserMapper, UserMapper>();

        // SQLite (Unit of Work는 Singleton으로 공유)
        services.AddSingleton<SqliteUnitOfWork>();
        services.AddSingleton<UserSqliteRepository>();

        // HttpClient with Polly
        services.AddHttpClient<UserApiRepository>(client =>
        {
            client.BaseAddress = new Uri("https://api.example.com");
        }).AddPolicyHandler(Policies.TransientHttpPolicy);

        // 서비스
        services.AddSingleton<IUserService, UserService>();

        // ViewModel
        services.AddTransient<UserViewModel>();

        Services = services.BuildServiceProvider();

        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            var mainWindow = new MainWindow
            {
                DataContext = Services.GetRequiredService<UserViewModel>()
            };
            desktop.MainWindow = mainWindow;
        }

        base.OnFrameworkInitializationCompleted();
    }
}
```

---

## 테스트 전략

### ViewModel 테스트 (Service 모킹)

```csharp
[Fact]
public async Task SearchCommand_LoadsUsers()
{
    var mockService = new Mock<IUserService>();
    mockService.Setup(s => s.SearchAsync(1, 20, "test", default))
               .ReturnsAsync(new PagedResult<User>
               {
                   Items = new[] { new User { Id = 1, Name = "Test" } },
                   TotalCount = 1
               });

    var vm = new UserViewModel(mockService.Object);
    vm.Keyword = "test";
    await vm.SearchCommand.Execute();

    Assert.Single(vm.Users);
    Assert.Equal("Test", vm.Users[0].Name);
}
```

### Service 테스트 (Repository 모킹)

```csharp
[Fact]
public async Task GetUserAsync_ReturnsFromCache_AfterFirstCall()
{
    var apiMock = new Mock<IUserRepository>();
    var localMock = new Mock<IUserRepository>();
    var uowMock = new Mock<IUnitOfWork>();
    var cache = new MemoryCache();
    var clock = new SystemClock();

    var user = new User { Id = 1, Name = "Alice" };
    apiMock.Setup(r => r.GetByIdAsync(1, default)).ReturnsAsync(user);

    var service = new UserService(apiMock.Object, localMock.Object, uowMock.Object, cache, clock);

    var result1 = await service.GetUserAsync(1);
    var result2 = await service.GetUserAsync(1);

    Assert.Same(result1, result2); // 캐시에서 반환되었는지 확인
    apiMock.Verify(r => r.GetByIdAsync(1, default), Times.Once);
}
```

---

## 성능과 확장

### 캐시 히트율 근사

캐시 TTL을 \(\tau\), 요청 도착률을 \(\lambda\)라 할 때, 단순 포아송 근사에서 캐시 적중 확률 \(H\)는

$$
H \approx 1 - e^{-\lambda \tau} (1 - p)
$$

여기서 \(p\)는 원본 데이터가 캐시에 없을 때 실제 원천에서 성공할 확률입니다. TTL이 클수록 히트율이 높아지지만, 데이터 신선도와의 트레이드오프를 고려해야 합니다.

### 추가 고려 사항

- **Outbox 패턴**: 오프라인 상태에서 생성된 변경사항을 큐에 저장했다가 네트워크 복구 시 서버와 동기화합니다.
- **CQRS 분리**: 읽기와 쓰기 모델을 분리하여 읽기 전용 쿼리를 최적화합니다.
- **로깅 및 모니터링**: Serilog, OpenTelemetry로 각 계층의 호출을 추적합니다.

---

## 계층별 책임 요약

| 계층         | 책임                                                                 |
|--------------|----------------------------------------------------------------------|
| **ViewModel**| UI 상태 관리, 사용자 커맨드, 바인딩. Service 호출만 수행.            |
| **Service**  | 유즈케이스 조합, 검증, 캐시, 재시도, 오프라인 폴백, 트랜잭션 경계.   |
| **Repository**| 데이터 소스(API, DB, 파일)에 대한 CRUD 캡슐화. 예외를 도메인 오류로 변환. |
| **Unit of Work** | 트랜잭션 관리 (Commit/Rollback).                                   |
| **Mapper**   | DTO ↔ 도메인 모델 변환.                                              |

---

## 결론

ViewModel에서 Service, Repository, Unit of Work, Cache, Mapper를 분리하면 각 계층의 책임이 명확해지고, 테스트 용이성과 유지보수성이 크게 향상됩니다. Service 계층이 정책(캐시 TTL, 오프라인 폴백, 재시도)을 중앙에서 관리하므로, 전체 애플리케이션의 동작을 일관되게 제어할 수 있습니다. 초기에는 코드량이 다소 늘어날 수 있지만, 장기적으로 견고한 크로스 플랫폼 애플리케이션을 만드는 데 필수적인 구조입니다.