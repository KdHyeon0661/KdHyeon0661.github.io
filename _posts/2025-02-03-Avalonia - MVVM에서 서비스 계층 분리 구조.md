---
layout: post
title: Avalonia - MVVM에서 서비스 계층 분리 구조
date: 2025-02-03 21:20:23 +0900
category: Avalonia
---
# 🧱 Avalonia MVVM에서 서비스 계층 분리 구조 (Service, Repository 등)

---

## 🎯 계층 분리의 목적

| 계층 | 목적 |
|------|------|
| **ViewModel** | UI 상태 및 사용자 입력 처리 |
| **Service Layer** | 비즈니스 로직, 유즈케이스 처리 |
| **Repository Layer** | DB, API, 파일 등의 실제 데이터 소스 처리 |
| **Model** | 데이터 구조 및 DTO 정의 |

> ✔ SRP(단일 책임 원칙)를 지키고, ViewModel은 UI 로직에만 집중하게 한다는 게 핵심입니다.

---

## 🧱 기본 계층 구조 예시

```
MyApp/
├── ViewModels/
│   └── UserViewModel.cs
├── Views/
│   └── UserView.axaml
├── Services/
│   └── IUserService.cs
│   └── UserService.cs
├── Repositories/
│   └── IUserRepository.cs
│   └── UserApiRepository.cs
├── Models/
│   └── User.cs
```

---

## 1️⃣ 모델 정의

### 📄 Models/User.cs

```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
}
```

---

## 2️⃣ Repository 계층 (데이터 접근)

### 📄 IUserRepository.cs

```csharp
public interface IUserRepository
{
    Task<User?> GetUserByIdAsync(int id);
    Task<IEnumerable<User>> GetAllUsersAsync();
}
```

### 📄 UserApiRepository.cs (API 기반 예시)

```csharp
public class UserApiRepository : IUserRepository
{
    private readonly HttpClient _http;

    public UserApiRepository(HttpClient http)
    {
        _http = http;
    }

    public async Task<User?> GetUserByIdAsync(int id)
    {
        var response = await _http.GetAsync($"https://api.example.com/users/{id}");
        if (!response.IsSuccessStatusCode)
            return null;

        var json = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<User>(json);
    }

    public async Task<IEnumerable<User>> GetAllUsersAsync()
    {
        var json = await _http.GetStringAsync("https://api.example.com/users");
        return JsonSerializer.Deserialize<List<User>>(json) ?? [];
    }
}
```

> ✅ 이 계층은 **실제 데이터의 저장소(DB, API, 파일 등)에만 집중**합니다.

---

## 3️⃣ Service 계층 (비즈니스 로직)

### 📄 IUserService.cs

```csharp
public interface IUserService
{
    Task<User?> GetUserProfileAsync(int userId);
    Task<IEnumerable<User>> SearchUsersAsync(string keyword);
}
```

### 📄 UserService.cs

```csharp
public class UserService : IUserService
{
    private readonly IUserRepository _repository;

    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }

    public async Task<User?> GetUserProfileAsync(int userId)
    {
        var user = await _repository.GetUserByIdAsync(userId);
        // 필요 시 로직 추가 (예: 캐싱, 포맷팅 등)
        return user;
    }

    public async Task<IEnumerable<User>> SearchUsersAsync(string keyword)
    {
        var all = await _repository.GetAllUsersAsync();
        return all.Where(u => u.Name.Contains(keyword, StringComparison.OrdinalIgnoreCase));
    }
}
```

> ✅ 이 계층은 도메인 비즈니스 규칙을 포함합니다. 예: 필터링, 조건 검증, 캐싱 등

---

## 4️⃣ ViewModel 계층

### 📄 UserViewModel.cs

```csharp
public class UserViewModel : ReactiveObject
{
    private readonly IUserService _userService;

    public UserViewModel(IUserService userService)
    {
        _userService = userService;

        LoadUserCommand = ReactiveCommand.CreateFromTask<int>(LoadUserAsync);
    }

    private User? _currentUser;
    public User? CurrentUser
    {
        get => _currentUser;
        set => this.RaiseAndSetIfChanged(ref _currentUser, value);
    }

    public ReactiveCommand<int, Unit> LoadUserCommand { get; }

    private async Task LoadUserAsync(int userId)
    {
        CurrentUser = await _userService.GetUserProfileAsync(userId);
    }
}
```

> ✅ ViewModel은 UI 입력 및 상태 관리를 담당하며, Service를 통해 필요한 데이터 요청

---

## 5️⃣ DI 등록 (App.xaml.cs)

```csharp
public override void OnFrameworkInitializationCompleted()
{
    var services = new ServiceCollection();

    services.AddSingleton<HttpClient>();
    services.AddSingleton<IUserRepository, UserApiRepository>();
    services.AddSingleton<IUserService, UserService>();
    services.AddTransient<UserViewModel>();

    Services = services.BuildServiceProvider();
    ...
}
```

---

## 6️⃣ 테스트 전략 (계층별 Mock 가능)

- ViewModel → Service → Repository 구조이므로
  - **ViewModel 테스트 시**: Service를 Mock
  - **Service 테스트 시**: Repository를 Mock

예: Moq 이용

```csharp
var mock = new Mock<IUserService>();
mock.Setup(x => x.GetUserProfileAsync(It.IsAny<int>()))
    .ReturnsAsync(new User { Id = 1, Name = "Test" });

var vm = new UserViewModel(mock.Object);
await vm.LoadUserCommand.Execute(1);
vm.CurrentUser!.Name.Should().Be("Test");
```

---

## 🧱 요약 계층 구성

| 계층 | 책임 | 예시 |
|------|------|------|
| **Model** | 데이터 정의 | `User` |
| **Repository** | 데이터 소스 접근 | `UserApiRepository` |
| **Service** | 도메인 로직 처리 | `UserService` |
| **ViewModel** | UI 상태 바인딩 | `UserViewModel` |

---

## 🧰 추가 확장 아이디어

- ✅ `IUnitOfWork` 또는 `IDataContext` 등 트랜잭션 관리 도입
- ✅ 파일 저장소(`IFileRepository`), SQLite용 Repository 등도 병렬 사용 가능
- ✅ 로컬/원격 동기화 지원 구조로 확장 가능
- ✅ DTO <-> Model 변환을 위한 Mapper 계층 (`AutoMapper`) 추가 가능

---

## 📘 결론

> Avalonia MVVM에서 서비스 계층을 분리하면 다음과 같은 이점이 있습니다:
>
> - ViewModel의 책임 축소 및 테스트 용이성 확보
> - 각 계층의 역할이 명확하여 유지보수 및 확장이 쉬움
> - 실질적인 비즈니스 로직을 UI 로직과 격리하여 중복 방지