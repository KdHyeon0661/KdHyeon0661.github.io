---
layout: post
title: Avalonia - Web API 연동
date: 2025-02-09 20:20:23 +0900
category: Avalonia
---
# Avalonia MVVM에서 Web API 연동

Avalonia 애플리케이션에서 백엔드 API와 통신할 때는 MVVM 패턴을 유지하면서도 안정성, 테스트 용이성, 확장성을 확보하는 것이 중요합니다. 이 글에서는 Repository 패턴, HttpClientFactory, Polly를 활용한 회복력 있는 API 클라이언트 구성, 그리고 ViewModel에서의 비동기 처리와 오류 대응까지 단계별로 설명합니다. 초중급 개발자를 기준으로, 실제 프로젝트에서 바로 활용할 수 있는 코드와 함께 핵심 개념을 전달합니다.

---

## 핵심 설계 원칙

1. **Repository 패턴**  
   API 호출을 별도의 클래스(Repository)로 캡슐화하여 ViewModel이 API 세부 사항을 알지 못하게 합니다. 테스트 시 Repository를 가짜(mock)로 교체할 수 있습니다.

2. **HttpClientFactory와 Polly**  
   `IHttpClientFactory`로 HttpClient 인스턴스를 관리하고, Polly 정책(재시도, 타임아웃, 서킷 브레이커)을 적용하여 네트워크 오류에 대한 회복력을 높입니다.

3. **DelegatingHandler**  
   요청/응답 파이프라인에 인증 토큰 추가, 로깅, 상관관계 ID 부여 등의 공통 기능을 핸들러로 분리합니다.

4. **비동기와 취소**  
   모든 API 호출은 비동기로 처리하고, 사용자가 작업을 취소할 수 있도록 `CancellationToken`을 지원합니다.

5. **오류 처리**  
   서버 응답 오류를 공통 형식(ApiError)으로 변환하고, ViewModel에서 사용자에게 알립니다.

---

## 데이터 모델과 결과 래퍼

API와 주고받는 데이터 모델을 정의합니다.

```csharp
// Models/Product.cs
public sealed class Product
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
    public decimal Price { get; init; }
}
```

페이징된 결과를 담을 공통 클래스:

```csharp
// Models/PagedResult.cs
public sealed class PagedResult<T>
{
    public IReadOnlyList<T> Items { get; init; } = Array.Empty<T>();
    public int TotalCount { get; init; }
    public int Page { get; init; }
    public int PageSize { get; init; }
    public string? Sort { get; init; }
    public string? Query { get; init; }
}
```

오류 응답 형식 (ProblemDetails 스타일):

```csharp
// Models/ApiError.cs
public sealed class ApiError
{
    public string? Title { get; init; }
    public string? Detail { get; init; }
    public int? Status { get; init; }
    public string? TraceId { get; init; }
}
```

---

## Repository 인터페이스

CRUD에 페이징, 정렬, 검색, 조건부 요청(ETag)을 포함한 인터페이스를 정의합니다.

```csharp
// Services/IProductRepository.cs
public interface IProductRepository
{
    Task<PagedResult<Product>> GetAllAsync(
        int page = 1,
        int pageSize = 20,
        string? sort = null,     // e.g. "name:asc,price:desc"
        string? query = null,
        CancellationToken ct = default);

    Task<(Product? Item, string? ETag)> GetByIdAsync(
        int id,
        string? ifNoneMatch = null,
        CancellationToken ct = default);

    Task<(int NewId, string? Location)> CreateAsync(
        Product product,
        CancellationToken ct = default);

    Task UpdateAsync(Product product, string? ifMatch = null, CancellationToken ct = default);

    Task DeleteAsync(int id, CancellationToken ct = default);
}
```

---

## HttpClientFactory와 핸들러 파이프라인

### 인증 핸들러

Bearer 토큰을 헤더에 추가합니다.

```csharp
// Services/Http/AuthenticatedHandler.cs
using System.Net.Http.Headers;

public sealed class AuthenticatedHandler : DelegatingHandler
{
    private readonly AppState _state; // 전역 상태 (AuthToken 보관)

    public AuthenticatedHandler(AppState state) => _state = state;

    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        if (!string.IsNullOrWhiteSpace(_state.AuthToken))
        {
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", _state.AuthToken);
        }
        return base.SendAsync(request, cancellationToken);
    }
}
```

### 로깅 핸들러 (간단)

```csharp
// Services/Http/LoggingHandler.cs
public sealed class LoggingHandler : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        Console.WriteLine($"[HTTP] {request.Method} {request.RequestUri}");
        var res = await base.SendAsync(request, cancellationToken);
        Console.WriteLine($"[HTTP] {(int)res.StatusCode} {res.ReasonPhrase}");
        return res;
    }
}
```

### Polly 정책 (재시도, 타임아웃, 서킷 브레이커)

Polly 패키지를 설치합니다:

```bash
dotnet add package Polly
dotnet add package Polly.Extensions.Http
```

```csharp
// Services/Http/Policies.cs
using Polly;
using Polly.Contrib.WaitAndRetry;
using Polly.Extensions.Http;
using System.Net;

public static class Policies
{
    public static IAsyncPolicy<HttpResponseMessage> RetryPolicy =>
        HttpPolicyExtensions
            .HandleTransientHttpError() // 5xx, 408 + HttpRequestException
            .OrResult(r => r.StatusCode == (HttpStatusCode)429) // Rate Limit
            .WaitAndRetryAsync(
                Backoff.DecorrelatedJitterBackoffV2(medianFirstRetryDelay: TimeSpan.FromMilliseconds(200), retryCount: 5),
                onRetry: (outcome, delay, attempt, ctx) =>
                {
                    Console.WriteLine($"[RETRY] attempt={attempt} delay={delay}");
                });

    public static IAsyncPolicy<HttpResponseMessage> TimeoutPolicy =>
        Policy.TimeoutAsync<HttpResponseMessage>(10); // 10초

    public static IAsyncPolicy<HttpResponseMessage> CircuitBreakerPolicy =>
        HttpPolicyExtensions.HandleTransientHttpError()
            .CircuitBreakerAsync(handledEventsAllowedBeforeBreaking: 5, durationOfBreak: TimeSpan.FromSeconds(30));
}
```

**백오프 공식**  
지터 백오프는 재시도 지연을 랜덤화하여 동시에 많은 클라이언트가 동일한 서버에 요청하는 것을 방지합니다. 평균 지연이 \(d\)일 때, \(n\)번째 재시도 지연의 기댓값은 약 \(d \cdot n\) 수준으로 증가합니다.

### DI 구성

`App.axaml.cs`에서 서비스를 등록합니다.

```csharp
// App.axaml.cs
using Microsoft.Extensions.DependencyInjection;

public partial class App : Application
{
    public static IServiceProvider Services { get; private set; } = default!;

    public override void OnFrameworkInitializationCompleted()
    {
        var services = new ServiceCollection();
        ConfigureServices(services);
        Services = services.BuildServiceProvider();

        // 예: MainWindow 표시
        var vm = Services.GetRequiredService<ProductListViewModel>();
        var window = new MainWindow { DataContext = vm };
        window.Show();

        base.OnFrameworkInitializationCompleted();
    }

    private void ConfigureServices(IServiceCollection services)
    {
        // 전역 상태 (토큰 저장용)
        services.AddSingleton<AppState>();

        // 핸들러 등록 (Transient)
        services.AddTransient<AuthenticatedHandler>();
        services.AddTransient<LoggingHandler>();

        // HttpClientFactory + Named Client
        services.AddHttpClient<ProductApiRepository>("product-api", client =>
        {
            client.BaseAddress = new Uri("https://api.example.com");
            client.Timeout = TimeSpan.FromSeconds(15);
            client.DefaultRequestHeaders.Accept.ParseAdd("application/json");
        })
        .AddHttpMessageHandler<AuthenticatedHandler>()
        .AddHttpMessageHandler<LoggingHandler>()
        .AddPolicyHandler(Policies.RetryPolicy)
        .AddPolicyHandler(Policies.TimeoutPolicy)
        .AddPolicyHandler(Policies.CircuitBreakerPolicy);

        // Repository 등록
        services.AddSingleton<IProductRepository>(sp =>
        {
            var factory = sp.GetRequiredService<IHttpClientFactory>();
            var http = factory.CreateClient("product-api");
            return new ProductApiRepository(http);
        });

        // ViewModel 등록
        services.AddTransient<ProductListViewModel>();
    }
}
```

---

## JSON 직렬화 옵션

공통 직렬화 옵션을 정의합니다.

```csharp
// Services/Http/JsonOptions.cs
using System.Text.Json;

public static class JsonOptions
{
    public static readonly JsonSerializerOptions Web = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        DefaultIgnoreCondition = System.Text.Json.Serialization.JsonIgnoreCondition.WhenWritingNull,
        WriteIndented = false
    };
}
```

---

## Repository 구현 (ProductApiRepository)

공통 헬퍼 메서드를 먼저 작성합니다.

```csharp
// Services/Http/HttpExtensions.cs
using System.Text.Json;

public static class HttpExtensions
{
    public static async Task<T?> ReadJsonAsync<T>(this HttpContent content, CancellationToken ct = default)
        => await JsonSerializer.DeserializeAsync<T>(await content.ReadAsStreamAsync(ct), JsonOptions.Web, ct);

    public static async Task<ApiError?> ReadApiErrorAsync(this HttpResponseMessage res, CancellationToken ct = default)
    {
        try { return await res.Content.ReadJsonAsync<ApiError>(ct); }
        catch { return new ApiError { Title = res.ReasonPhrase, Status = (int)res.StatusCode }; }
    }

    public static string BuildQuery(IDictionary<string, string?> pairs)
        => string.Join("&", pairs.Where(kv => !string.IsNullOrWhiteSpace(kv.Value))
                                 .Select(kv => $"{Uri.EscapeDataString(kv.Key)}={Uri.EscapeDataString(kv.Value!)}"));
}
```

이제 `ProductApiRepository`를 구현합니다.

```csharp
// Services/ProductApiRepository.cs
using System.Net.Http.Json;
using System.Text;
using System.Text.Json;

public sealed class ProductApiRepository : IProductRepository
{
    private readonly HttpClient _http;

    public ProductApiRepository(HttpClient http) => _http = http;

    public async Task<PagedResult<Product>> GetAllAsync(
        int page = 1, int pageSize = 20, string? sort = null, string? query = null, CancellationToken ct = default)
    {
        var queryString = HttpExtensions.BuildQuery(new Dictionary<string, string?>
        {
            ["page"] = page.ToString(),
            ["pageSize"] = pageSize.ToString(),
            ["sort"] = sort,
            ["q"] = query
        });

        using var res = await _http.GetAsync($"/api/products?{queryString}", ct);
        if (!res.IsSuccessStatusCode)
        {
            var err = await res.ReadApiErrorAsync(ct);
            throw new HttpRequestException(err?.Detail ?? err?.Title ?? res.ReasonPhrase);
        }

        var items = await res.Content.ReadJsonAsync<List<Product>>(ct) ?? new();
        // 실제 API는 TotalCount를 헤더나 본문에 포함할 수 있음. 여기서는 간단히 items.Count로 처리
        return new PagedResult<Product>
        {
            Items = items,
            TotalCount = items.Count,
            Page = page,
            PageSize = pageSize,
            Sort = sort,
            Query = query
        };
    }

    public async Task<(Product? Item, string? ETag)> GetByIdAsync(
        int id, string? ifNoneMatch = null, CancellationToken ct = default)
    {
        var req = new HttpRequestMessage(HttpMethod.Get, $"/api/products/{id}");
        if (!string.IsNullOrWhiteSpace(ifNoneMatch))
            req.Headers.TryAddWithoutValidation("If-None-Match", ifNoneMatch);

        using var res = await _http.SendAsync(req, ct);
        if (res.StatusCode == System.Net.HttpStatusCode.NotModified)
            return (null, ifNoneMatch); // 304 Not Modified

        if (!res.IsSuccessStatusCode)
        {
            var err = await res.ReadApiErrorAsync(ct);
            throw new HttpRequestException(err?.Detail ?? err?.Title ?? res.ReasonPhrase);
        }

        var etag = res.Headers.ETag?.Tag;
        var product = await res.Content.ReadJsonAsync<Product>(ct);
        return (product, etag);
    }

    public async Task<(int NewId, string? Location)> CreateAsync(Product product, CancellationToken ct = default)
    {
        var json = JsonSerializer.Serialize(product, JsonOptions.Web);
        using var res = await _http.PostAsync("/api/products",
            new StringContent(json, Encoding.UTF8, "application/json"), ct);

        if (!res.IsSuccessStatusCode)
        {
            var err = await res.ReadApiErrorAsync(ct);
            throw new HttpRequestException(err?.Detail ?? err?.Title ?? res.ReasonPhrase);
        }

        var location = res.Headers.Location?.ToString();
        // 서버가 생성된 객체를 응답 본문에 포함한다고 가정
        var created = await res.Content.ReadJsonAsync<Product>(ct);
        return (created?.Id ?? 0, location);
    }

    public async Task UpdateAsync(Product product, string? ifMatch = null, CancellationToken ct = default)
    {
        var req = new HttpRequestMessage(HttpMethod.Put, $"/api/products/{product.Id}")
        {
            Content = new StringContent(JsonSerializer.Serialize(product, JsonOptions.Web), Encoding.UTF8, "application/json")
        };
        if (!string.IsNullOrWhiteSpace(ifMatch))
            req.Headers.TryAddWithoutValidation("If-Match", ifMatch);

        using var res = await _http.SendAsync(req, ct);
        if (!res.IsSuccessStatusCode)
        {
            var err = await res.ReadApiErrorAsync(ct);
            throw new HttpRequestException(err?.Detail ?? err?.Title ?? res.ReasonPhrase);
        }
    }

    public async Task DeleteAsync(int id, CancellationToken ct = default)
    {
        using var res = await _http.DeleteAsync($"/api/products/{id}", ct);
        if (!res.IsSuccessStatusCode)
        {
            var err = await res.ReadApiErrorAsync(ct);
            throw new HttpRequestException(err?.Detail ?? err?.Title ?? res.ReasonPhrase);
        }
    }
}
```

---

## ViewModel (ProductListViewModel)

이제 Repository를 사용하는 ViewModel을 작성합니다. ReactiveUI를 활용하여 명령, 로딩 상태, 오류 메시지를 처리합니다.

```csharp
// ViewModels/ProductListViewModel.cs
using ReactiveUI;
using System.Collections.ObjectModel;
using System.Reactive;
using System.Reactive.Linq;

public sealed class ProductListViewModel : ReactiveObject
{
    private readonly IProductRepository _repo;
    private CancellationTokenSource? _cts;

    public ProductListViewModel(IProductRepository repo)
    {
        _repo = repo;

        LoadCommand = ReactiveCommand.CreateFromTask(LoadAsync);
        SearchCommand = ReactiveCommand.CreateFromTask(LoadAsync);
        CancelCommand = ReactiveCommand.Create(Cancel);

        // 페이지, 페이지 크기, 정렬이 변경되면 자동으로 로드 (150ms 디바운스)
        this.WhenAnyValue(x => x.Page, x => x.PageSize, x => x.Sort)
            .Throttle(TimeSpan.FromMilliseconds(150))
            .ObserveOn(RxApp.MainThreadScheduler)
            .Select(_ => Unit.Default)
            .InvokeCommand(LoadCommand);
    }

    public ObservableCollection<Product> Items { get; } = new();

    private int _page = 1;
    public int Page
    {
        get => _page;
        set => this.RaiseAndSetIfChanged(ref _page, value);
    }

    private int _pageSize = 20;
    public int PageSize
    {
        get => _pageSize;
        set => this.RaiseAndSetIfChanged(ref _pageSize, value);
    }

    private string? _sort = "name:asc";
    public string? Sort
    {
        get => _sort;
        set => this.RaiseAndSetIfChanged(ref _sort, value);
    }

    private string? _query = "";
    public string? Query
    {
        get => _query;
        set => this.RaiseAndSetIfChanged(ref _query, value);
    }

    private bool _isBusy;
    public bool IsBusy
    {
        get => _isBusy;
        set => this.RaiseAndSetIfChanged(ref _isBusy, value);
    }

    private string? _error;
    public string? Error
    {
        get => _error;
        set => this.RaiseAndSetIfChanged(ref _error, value);
    }

    public ReactiveCommand<Unit, Unit> LoadCommand { get; }
    public ReactiveCommand<Unit, Unit> SearchCommand { get; }
    public ReactiveCommand<Unit, Unit> CancelCommand { get; }

    private async Task LoadAsync()
    {
        Cancel(); // 이전 요청 취소
        _cts = new CancellationTokenSource();

        IsBusy = true;
        Error = null;

        try
        {
            Items.Clear();
            var result = await _repo.GetAllAsync(Page, PageSize, Sort, Query, _cts.Token);
            foreach (var p in result.Items)
                Items.Add(p);
        }
        catch (OperationCanceledException)
        {
            // 사용자가 취소
        }
        catch (HttpRequestException ex)
        {
            Error = ex.Message;
        }
        finally
        {
            IsBusy = false;
        }
    }

    private void Cancel()
    {
        if (_cts is { IsCancellationRequested: false })
            _cts.Cancel();
    }
}
```

---

## View (ProductListView.axaml)

검색, 정렬, 데이터 그리드, 로딩 상태 등을 표시합니다.

```xml
<!-- Views/ProductListView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyApp.Views.ProductListView">
    <StackPanel Margin="16" Spacing="8">
        <!-- 검색 및 정렬 도구 -->
        <StackPanel Orientation="Horizontal" Spacing="8">
            <TextBox Width="200" Watermark="검색어" Text="{Binding Query, Mode=TwoWay}" />
            <ComboBox Width="160" SelectedItem="{Binding Sort}">
                <ComboBoxItem Content="이름 오름차순" Tag="name:asc" />
                <ComboBoxItem Content="이름 내림차순" Tag="name:desc" />
                <ComboBoxItem Content="가격 오름차순" Tag="price:asc" />
                <ComboBoxItem Content="가격 내림차순" Tag="price:desc" />
            </ComboBox>
            <Button Content="검색" Command="{Binding SearchCommand}" />
            <Button Content="취소" Command="{Binding CancelCommand}" />
            <ProgressBar IsIndeterminate="True" IsVisible="{Binding IsBusy}" Width="120" Height="6" />
        </StackPanel>

        <!-- 오류 표시 -->
        <TextBlock Text="{Binding Error}" Foreground="Red" TextWrapping="Wrap"
                   IsVisible="{Binding Error, Converter={x:Static StringConverters.IsNotNullOrEmpty}}" />

        <!-- 데이터 그리드 -->
        <DataGrid Items="{Binding Items}" AutoGenerateColumns="False" Height="300">
            <DataGrid.Columns>
                <DataGridTextColumn Header="ID" Binding="{Binding Id}" />
                <DataGridTextColumn Header="상품명" Binding="{Binding Name}" />
                <DataGridTextColumn Header="가격" Binding="{Binding Price, StringFormat={}{0:N0}}" />
            </DataGrid.Columns>
        </DataGrid>
    </StackPanel>
</UserControl>
```

---

## 단위 테스트

### ViewModel 테스트 (Repository 모킹)

Moq를 사용하여 Repository를 가짜로 교체합니다.

```csharp
// Tests/ProductListViewModelTests.cs
using Moq;
using FluentAssertions;
using Xunit;

public sealed class ProductListViewModelTests
{
    [Fact]
    public async Task LoadCommand_FillsItems_FromRepository()
    {
        var repo = new Mock<IProductRepository>();
        var products = new List<Product> { new Product { Id = 1, Name = "A", Price = 1000m } };
        repo.Setup(r => r.GetAllAsync(1, 20, "name:asc", "", It.IsAny<CancellationToken>()))
            .ReturnsAsync(new PagedResult<Product>
            {
                Items = products,
                TotalCount = 1,
                Page = 1,
                PageSize = 20,
                Sort = "name:asc",
                Query = ""
            });

        var vm = new ProductListViewModel(repo.Object)
        {
            Page = 1,
            PageSize = 20,
            Sort = "name:asc",
            Query = ""
        };

        await vm.LoadCommand.Execute();

        vm.Items.Should().HaveCount(1);
        vm.Items[0].Name.Should().Be("A");
        vm.Error.Should().BeNull();
    }
}
```

### Repository 테스트 (HttpMessageHandler 스텁)

HTTP 응답을 가로채는 핸들러를 만들어 테스트합니다.

```csharp
// Tests/HttpMessageHandlerStub.cs
public sealed class HandlerStub : HttpMessageHandler
{
    private readonly Func<HttpRequestMessage, HttpResponseMessage> _response;

    public HandlerStub(Func<HttpRequestMessage, HttpResponseMessage> response) => _response = response;

    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
        => Task.FromResult(_response(request));
}
```

```csharp
// Tests/ProductApiRepositoryTests.cs
using FluentAssertions;
using System.Text.Json;
using Xunit;

public sealed class ProductApiRepositoryTests
{
    [Fact]
    public async Task GetAllAsync_ParsesJson()
    {
        var product = new Product { Id = 10, Name = "Test", Price = 99.99m };
        var json = JsonSerializer.Serialize(new[] { product }, JsonOptions.Web);
        var handler = new HandlerStub(_ => new HttpResponseMessage(System.Net.HttpStatusCode.OK)
        {
            Content = new StringContent(json, Encoding.UTF8, "application/json")
        });

        var http = new HttpClient(handler) { BaseAddress = new Uri("https://dummy/") };
        var repo = new ProductApiRepository(http);

        var result = await repo.GetAllAsync();

        result.Items.Should().HaveCount(1);
        result.Items[0].Id.Should().Be(10);
        result.Items[0].Name.Should().Be("Test");
    }
}
```

---

## 보안 및 운영 팁

- **토큰 저장**  
  `AppState.AuthToken`을 메모리에 보관합니다. 자동 로그인이 필요하다면 OS별 안전 저장소(Windows DPAPI, macOS Keychain, Linux SecretService)를 사용하세요.

- **민감 정보**  
  암호나 토큰을 JSON 파일에 평문으로 저장하지 마십시오. 암호화 서비스를 활용하거나 앞서 설명한 안전 저장소를 사용합니다.

- **레이트 리밋(429)**  
  Polly의 재시도 정책에서 `Retry-After` 헤더를 읽어 해당 시간만큼 대기하는 로직을 추가할 수 있습니다.

- **TLS/인증서 고정**  
  높은 보안이 필요한 경우 `DelegatingHandler`에서 인증서 검증을 강화할 수 있습니다.

- **로깅과 추적**  
  `LoggingHandler`에 상관관계 ID(`X-Correlation-ID`)를 부여하면 서버 로그와 연동하여 디버깅이 쉬워집니다.

---

## 요약

| 계층 | 역할 | 주요 기술 |
|------|------|-----------|
| View | UI 표시 | XAML, DataBinding |
| ViewModel | 상태 관리, 명령 | ReactiveUI, ReactiveCommand, 취소 토큰 |
| Repository | API 호출 캡슐화 | HttpClientFactory, DelegatingHandler |
| HTTP 파이프라인 | 인증, 로깅, 회복력 | Polly, DelegatingHandler |
| 모델 | 데이터 구조 | POCO, 직렬화 옵션 |

**전체 흐름**

1. 사용자가 View에서 검색/정렬을 선택 → ViewModel 속성 변경  
2. ViewModel이 `LoadCommand` 실행 → Repository 호출  
3. Repository가 HttpClient를 통해 API 요청 (핸들러 파이프라인 적용)  
4. 응답 수신 → JSON 파싱 → 도메인 객체 반환  
5. ViewModel이 Items 컬렉션 업데이트 → View 자동 갱신  

---

## 결론

이 글에서 소개한 구조를 따르면 Avalonia 애플리케이션에서 Web API를 안정적이고 테스트 가능하게 연동할 수 있습니다. Repository 패턴으로 API 호출을 캡슐화하고, HttpClientFactory와 Polly로 네트워크 오류에 강한 클라이언트를 구성하며, ReactiveUI를 활용해 반응형 ViewModel을 작성하는 것이 핵심입니다. 이 기반 위에 페이징, 정렬, 검색, ETag 기반 동시성 제어 등을 추가하면 실제 제품 수준의 API 연동을 완성할 수 있습니다.