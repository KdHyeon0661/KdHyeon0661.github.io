---
layout: post
title: Avalonia - Web API 연동
date: 2025-02-09 20:20:23 +0900
category: Avalonia
---
# 🌐 Avalonia MVVM에서 Web API 연동 (HttpClient 기반)

---

## 🎯 핵심 목표

| 항목 | 설명 |
|------|------|
| HttpClient 사용 | API 서버에 비동기 요청 수행 |
| Repository 패턴 | API 호출을 분리해 테스트 가능하도록 구현 |
| DI로 의존성 주입 | HttpClient와 서비스 구조 분리 |
| 에러/예외 처리 | 실패 처리, 사용자 메시지 출력 등 |
| 인증 처리 | 토큰 포함 및 헤더 설정

---

## 🧱 구성 예시

```
MyApp/
├── Models/
│   └── Product.cs
├── Services/
│   └── IProductRepository.cs
│   └── ProductApiRepository.cs
├── ViewModels/
│   └── ProductListViewModel.cs
├── Views/
│   └── ProductListView.axaml
```

---

## 1️⃣ 데이터 모델 정의

### 📄 Models/Product.cs

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
}
```

---

## 2️⃣ Repository 인터페이스 정의

### 📄 Services/IProductRepository.cs

```csharp
public interface IProductRepository
{
    Task<IEnumerable<Product>> GetAllAsync();
    Task<Product?> GetByIdAsync(int id);
    Task CreateAsync(Product product);
    Task UpdateAsync(Product product);
    Task DeleteAsync(int id);
}
```

---

## 3️⃣ API 구현 클래스 (HttpClient 사용)

### 📄 Services/ProductApiRepository.cs

```csharp
public class ProductApiRepository : IProductRepository
{
    private readonly HttpClient _http;

    public ProductApiRepository(HttpClient http)
    {
        _http = http;
    }

    public async Task<IEnumerable<Product>> GetAllAsync()
    {
        var response = await _http.GetAsync("/api/products");
        response.EnsureSuccessStatusCode();

        var json = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<List<Product>>(json) ?? [];
    }

    public async Task<Product?> GetByIdAsync(int id)
    {
        var response = await _http.GetAsync($"/api/products/{id}");
        if (!response.IsSuccessStatusCode)
            return null;

        var json = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<Product>(json);
    }

    public async Task CreateAsync(Product product)
    {
        var content = new StringContent(JsonSerializer.Serialize(product), Encoding.UTF8, "application/json");
        var response = await _http.PostAsync("/api/products", content);
        response.EnsureSuccessStatusCode();
    }

    public async Task UpdateAsync(Product product)
    {
        var content = new StringContent(JsonSerializer.Serialize(product), Encoding.UTF8, "application/json");
        var response = await _http.PutAsync($"/api/products/{product.Id}", content);
        response.EnsureSuccessStatusCode();
    }

    public async Task DeleteAsync(int id)
    {
        var response = await _http.DeleteAsync($"/api/products/{id}");
        response.EnsureSuccessStatusCode();
    }
}
```

> ✅ `response.EnsureSuccessStatusCode()`는 HTTP 실패 시 예외를 발생시킵니다.

---

## 4️⃣ HttpClient 및 서비스 등록 (App.xaml.cs)

```csharp
private void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton(new HttpClient
    {
        BaseAddress = new Uri("https://api.example.com")
    });

    services.AddSingleton<IProductRepository, ProductApiRepository>();
    services.AddTransient<ProductListViewModel>();
}
```

---

## 5️⃣ ViewModel에서 API 호출 사용

### 📄 ViewModels/ProductListViewModel.cs

```csharp
public class ProductListViewModel : ReactiveObject
{
    private readonly IProductRepository _repository;

    public ObservableCollection<Product> Products { get; } = new();

    public ReactiveCommand<Unit, Unit> LoadCommand { get; }

    public ProductListViewModel(IProductRepository repository)
    {
        _repository = repository;

        LoadCommand = ReactiveCommand.CreateFromTask(LoadProductsAsync);
    }

    private async Task LoadProductsAsync()
    {
        try
        {
            Products.Clear();
            var items = await _repository.GetAllAsync();
            foreach (var item in items)
                Products.Add(item);
        }
        catch (HttpRequestException ex)
        {
            Console.WriteLine($"API 호출 실패: {ex.Message}");
        }
    }
}
```

---

## 6️⃣ 인증 헤더 추가 (Bearer 토큰 등)

```csharp
public class AuthenticatedHandler : DelegatingHandler
{
    private readonly AppState _state;

    public AuthenticatedHandler(AppState state)
    {
        _state = state;
    }

    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        if (!string.IsNullOrWhiteSpace(_state.AuthToken))
        {
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", _state.AuthToken);
        }
        return base.SendAsync(request, cancellationToken);
    }
}
```

DI 등록 시:

```csharp
services.AddSingleton<AppState>();
services.AddTransient<AuthenticatedHandler>();

services.AddSingleton(provider =>
{
    var handler = provider.GetRequiredService<AuthenticatedHandler>();
    return new HttpClient(handler)
    {
        BaseAddress = new Uri("https://api.example.com")
    };
});
```

---

## 7️⃣ HttpClient 팁

| 기능 | 설명 |
|------|------|
| `Timeout` | 기본 100초 → 줄이거나 늘릴 수 있음 |
| `BaseAddress` | 모든 요청에 기본 URL 설정 |
| `DefaultRequestHeaders` | 기본 헤더 설정 |
| `DelegatingHandler` | 인증, 로깅 등 중간 처리기 연결 |

---

## 8️⃣ 테스트 팁

- API 호출 자체를 Mock 하고 싶다면 `IProductRepository`를 테스트용으로 대체합니다.
- 실제 API를 테스트하지 않고 ViewModel만 테스트할 수 있습니다.

```csharp
var mock = new Mock<IProductRepository>();
mock.Setup(repo => repo.GetAllAsync())
    .ReturnsAsync(new[] { new Product { Id = 1, Name = "테스트", Price = 1000 } });

var vm = new ProductListViewModel(mock.Object);
await vm.LoadCommand.Execute();
vm.Products.Count.Should().Be(1);
```

---

## ✅ 정리

| 항목 | 설명 |
|------|------|
| API 호출 | `HttpClient`로 비동기 처리 |
| 구조화 | `Repository`로 캡슐화 후 `ViewModel`에서 주입 |
| 인증 | `Bearer` 토큰은 `AuthorizationHeader` 사용 |
| 테스트 | API를 Mock하고 ViewModel 단위 테스트 가능 |
| DI 통합 | `HttpClient`, `Repository`, `ViewModel` 모두 DI 구성