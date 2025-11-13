---
layout: post
title: Avalonia - Web API 연동
date: 2025-02-09 20:20:23 +0900
category: Avalonia
---
# Avalonia MVVM에서 Web API 연동

## 0. 핵심 설계 요약

- **Repository 패턴**으로 API 호출을 캡슐화 → ViewModel 테스트 용이성.
- **DI + HttpClientFactory**로 HttpClient 수명/핸들러 파이프라인 제어.
- **DelegatingHandler**로 인증 토큰, 로깅, 상관관계 ID, 재시도/회로차단을 조립.
- **직렬화 옵션(System.Text.Json)**, **취소 토큰**, **오류 모델(ProblemDetails)** 정립.
- **페이징/정렬/필터**, **ETag/If-None-Match**, **429/503 백오프** 등 운영 내구성.

---

## 1. 데이터 모델과 결과 래퍼

### 1.1 도메인 모델

```csharp
// Models/Product.cs
public sealed class Product
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
    public decimal Price { get; init; }
}
```

### 1.2 페이지네이션/정렬 결과 공통 모델

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

### 1.3 오류 응답(ProblemDetails 등)

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

## 2. Repository 인터페이스(확장형)

초안의 CRUD에서 **페이징/정렬/필터**, **조건적 요청(ETag)**, **취소 토큰**을 포함한다.

```csharp
// Services/IProductRepository.cs
public interface IProductRepository
{
    Task<PagedResult<Product>> GetAllAsync(
        int page = 1,
        int pageSize = 20,
        string? sort = null,     // e.g. "name:asc,price:desc"
        string? query = null,    // search keyword
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

## 3. HttpClientFactory 및 핸들러 파이프라인

### 3.1 인증/상태 핸들러

```csharp
// Services/Http/AuthenticatedHandler.cs
using System.Net.Http.Headers;

public sealed class AuthenticatedHandler : DelegatingHandler
{
    private readonly AppState _state;

    public AuthenticatedHandler(AppState state) => _state = state;

    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        if (!string.IsNullOrWhiteSpace(_state.AuthToken))
        {
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", _state.AuthToken);
        }

        // 상관관계 ID(분산 추적용) 부여 예시
        if (!request.Headers.Contains("X-Correlation-ID"))
            request.Headers.Add("X-Correlation-ID", Guid.NewGuid().ToString("N"));

        return base.SendAsync(request, cancellationToken);
    }
}
```

### 3.2 로깅 핸들러(간단)

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

### 3.3 Polly 기반 회복력(재시도/백오프/서킷)

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
                    Console.WriteLine($"[RETRY] attempt={attempt} delay={delay} status={(int?)outcome.Result?.StatusCode}");
                });

    public static IAsyncPolicy<HttpResponseMessage> TimeoutPolicy =>
        Policy.TimeoutAsync<HttpResponseMessage>(10); // 10s

    public static IAsyncPolicy<HttpResponseMessage> CircuitBreakerPolicy =>
        HttpPolicyExtensions.HandleTransientHttpError()
            .CircuitBreakerAsync(handledEventsAllowedBeforeBreaking: 5, durationOfBreak: TimeSpan.FromSeconds(30));
}
```

> 지터 백오프의 직관적 근사: 평균 지연이 \( d \)일 때, \( n \)번째 재시도 지연의 기대값은 대략
> $$ E[T_n] \approx d \cdot n $$
> 단, jitter 사용 시 분산이 커져서 “떼쓰기(동시 재시도 충돌)”를 완화한다.

---

## 4. JSON 직렬화 옵션

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

## 5. API 구현(ProductApiRepository)

### 5.1 공통 헬퍼

```csharp
// Services/Http/HttpExtensions.cs
using System.Net;
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

### 5.2 구현

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
        var q = HttpExtensions.BuildQuery(new Dictionary<string, string?>
        {
            ["page"] = page.ToString(),
            ["pageSize"] = pageSize.ToString(),
            ["sort"] = sort,
            ["q"] = query
        });

        using var res = await _http.GetAsync($"/api/products?{q}", ct);
        if (!res.IsSuccessStatusCode)
        {
            var err = await res.ReadApiErrorAsync(ct);
            throw new HttpRequestException(err?.Detail ?? err?.Title ?? res.ReasonPhrase);
        }

        var items = await res.Content.ReadJsonAsync<List<Product>>(ct) ?? new();
        // 총 개수/정렬/검색어는 헤더나 별도 필드로 받는다고 가정하거나 추정 처리
        // 예제 단순화: TotalCount를 items.Count로 대체
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
        {
            return (null, ifNoneMatch); // 변경 없음 (304)
        }

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
        // 서버가 생성 ID 반환(본문/헤더) 중 한 가지 가정
        if (res.Content.Headers.ContentLength is > 0)
        {
            var created = await res.Content.ReadJsonAsync<Product>(ct);
            return (created?.Id ?? 0, location);
        }
        return (0, location);
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

## 6. DI 구성(App.axaml.cs)

HttpClientFactory + 핸들러 파이프라인 + Polly 정책을 **Named Client**로 등록한다.

```csharp
// App.axaml.cs (중요 부분)
using Microsoft.Extensions.DependencyInjection;
using System.Net.Http;

public class App : Application
{
    public static IServiceProvider Services { get; private set; } = default!;

    public override void OnFrameworkInitializationCompleted()
    {
        var sc = new ServiceCollection();

        sc.AddSingleton<AppState>();

        sc.AddTransient<AuthenticatedHandler>();
        sc.AddTransient<LoggingHandler>();

        sc.AddHttpClient<ProductApiRepository>("product-api", client =>
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

        // IProductRepository -> ProductApiRepository 바인딩
        sc.AddSingleton<IProductRepository>(sp =>
        {
            var factory = sp.GetRequiredService<IHttpClientFactory>();
            var http = factory.CreateClient("product-api");
            return new ProductApiRepository(http);
        });

        // ViewModel
        sc.AddTransient<ProductListViewModel>();

        Services = sc.BuildServiceProvider();

        var vm = Services.GetRequiredService<ProductListViewModel>();
        var win = new Window { Content = new Views.ProductListView(), DataContext = vm };
        win.Show();

        base.OnFrameworkInitializationCompleted();
    }
}
```

> 초기 초안의 “new HttpClient(...)” 생성 대신 **HttpClientFactory**를 사용하면
> 소켓 핸들 누수 방지, 핸들러 체인/Polly 정책 조립, 네임드 클라이언트 관리가 쉬워진다.

---

## 7. ViewModel: 로딩/에러/취소/정렬/검색

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

        // 정렬 변경/페이지 변경 시 자동 로드
        this.WhenAnyValue(x => x.Page, x => x.PageSize, x => x.Sort)
            .Throttle(TimeSpan.FromMilliseconds(150))
            .ObserveOn(RxApp.MainThreadScheduler)
            .Select(_ => Unit.Default)
            .InvokeCommand(LoadCommand);
    }

    public ObservableCollection<Product> Items { get; } = new();

    private int _page = 1;
    public int Page { get => _page; set => this.RaiseAndSetIfChanged(ref _page, value); }

    private int _pageSize = 20;
    public int PageSize { get => _pageSize; set => this.RaiseAndSetIfChanged(ref _pageSize, value); }

    private string? _sort = "name:asc";
    public string? Sort { get => _sort; set => this.RaiseAndSetIfChanged(ref _sort, value); }

    private string? _query = "";
    public string? Query { get => _query; set => this.RaiseAndSetIfChanged(ref _query, value); }

    private bool _isBusy;
    public bool IsBusy { get => _isBusy; set => this.RaiseAndSetIfChanged(ref _isBusy, value); }

    private string? _error;
    public string? Error { get => _error; set => this.RaiseAndSetIfChanged(ref _error, value); }

    public ReactiveCommand<Unit, Unit> LoadCommand { get; }
    public ReactiveCommand<Unit, Unit> SearchCommand { get; }
    public ReactiveCommand<Unit, Unit> CancelCommand { get; }

    private async Task LoadAsync()
    {
        Cancel(); // 기존 요청 취소
        _cts = new CancellationTokenSource();

        IsBusy = true;
        Error = null;

        try
        {
            Items.Clear();
            var result = await _repo.GetAllAsync(Page, PageSize, Sort, Query, _cts.Token);
            foreach (var p in result.Items) Items.Add(p);
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

## 8. View 예시

```xml
<!-- Views/ProductListView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             x:Class="MyApp.Views.ProductListView">
  <StackPanel Margin="16" Spacing="8">
    <StackPanel Orientation="Horizontal" Spacing="8">
      <TextBox Width="200" Watermark="검색어" Text="{Binding Query}"/>
      <ComboBox Width="160" SelectedItem="{Binding Sort}">
        <ComboBoxItem Content="이름 오름차순" Tag="name:asc"/>
        <ComboBoxItem Content="이름 내림차순" Tag="name:desc"/>
        <ComboBoxItem Content="가격 오름차순" Tag="price:asc"/>
        <ComboBoxItem Content="가격 내림차순" Tag="price:desc"/>
      </ComboBox>
      <Button Content="검색" Command="{Binding SearchCommand}"/>
      <Button Content="취소" Command="{Binding CancelCommand}"/>
      <ProgressBar IsIndeterminate="True" IsVisible="{Binding IsBusy}" Width="120" Height="6"/>
    </StackPanel>

    <TextBlock Text="{Binding Error}" Foreground="Red" TextWrapping="Wrap" IsVisible="{Binding Error, Converter={x:Static StringConverters.IsNotNullOrEmpty}}"/>

    <DataGrid Items="{Binding Items}" AutoGenerateColumns="False" Height="300">
      <DataGrid.Columns>
        <DataGridTextColumn Header="ID" Binding="{Binding Id}"/>
        <DataGridTextColumn Header="상품명" Binding="{Binding Name}"/>
        <DataGridTextColumn Header="가격" Binding="{Binding Price, StringFormat={}{0:N0}}"/>
      </DataGrid.Columns>
    </DataGrid>
  </StackPanel>
</UserControl>
```

> ComboBox에서 `SelectedItem` 대신 `SelectedValue` + `SelectedValuePath=Tag`를 써서 Tag 문자열을 직접 바인딩하는 패턴도 좋다.

---

## 9. 개별 항목 읽기/동시성 제어(ETag)

리소스 버전 충돌 방지: 서버가 `ETag` 제공 → 수정 시 `If-Match` 헤더로 낙관적 동시성 제어.

```csharp
// 읽기
var (item, etag) = await _repo.GetByIdAsync(42);
// 수정
await _repo.UpdateAsync(item! with { Price = 19900m }, ifMatch: etag);
```

서버가 412(Precondition Failed) 반환 시, ViewModel에서 “다른 사용자가 먼저 수정” 메시지를 안내하고 재로딩/머지 UI를 제공한다.

---

## 10. 429(레이트 리밋)/백오프 처리

Polly로 재시도하되, 서버가 `Retry-After` 헤더를 줄 경우 해당 시간을 우선한다.
지터 백오프의 직관적 기대 지연 합은 \( \sum_{k=1}^n E[T_k] \)이며, 단순 선형 증가 근사로
$$
\sum_{k=1}^n d \cdot k = d \cdot \frac{n(n+1)}{2}
$$
이므로 재시도 횟수 \( n \)이 커질수록 지연 총량이 급증한다. → 재시도 상한 필수.

---

## 11. 단위 테스트

### 11.1 ViewModel: Repository 모킹

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
        repo.Setup(r => r.GetAllAsync(1, 20, "name:asc", "", It.IsAny<CancellationToken>()))
            .ReturnsAsync(new PagedResult<Product>
            {
                Items = new[] { new Product { Id=1, Name="A", Price=1000m } },
                TotalCount = 1, Page = 1, PageSize = 20, Sort = "name:asc", Query = ""
            });

        var vm = new ProductListViewModel(repo.Object)
        {
            Page = 1, PageSize = 20, Sort = "name:asc", Query = ""
        };

        await vm.LoadCommand.Execute();

        vm.Items.Should().HaveCount(1);
        vm.Items[0].Name.Should().Be("A");
        vm.Error.Should().BeNull();
    }
}
```

### 11.2 Repository: HttpMessageHandler 스텁

```csharp
// Tests/HttpMessageHandlerStub.cs
using System.Net;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

public sealed class HandlerStub : HttpMessageHandler
{
    private readonly Func<HttpRequestMessage, HttpResponseMessage> _res;
    public HandlerStub(Func<HttpRequestMessage, HttpResponseMessage> res) => _res = res;

    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
        => Task.FromResult(_res(request));
}
```

```csharp
// Tests/ProductApiRepositoryTests.cs
using System.Text.Json;
using FluentAssertions;
using Xunit;

public sealed class ProductApiRepositoryTests
{
    [Fact]
    public async Task GetAllAsync_ParsesJson()
    {
        var payload = JsonSerializer.Serialize(new[] { new Product { Id=10, Name="N", Price=10m } }, JsonOptions.Web);
        var handler = new HandlerStub(_ => new HttpResponseMessage(System.Net.HttpStatusCode.OK)
        {
            Content = new StringContent(payload, System.Text.Encoding.UTF8, "application/json")
        });

        var http = new HttpClient(handler) { BaseAddress = new Uri("https://dummy/") };
        var repo = new ProductApiRepository(http);

        var page = await repo.GetAllAsync();
        page.Items.Should().HaveCount(1);
        page.Items[0].Id.Should().Be(10);
    }
}
```

---

## 12. 보안·운영 팁

- **비밀번호 입력**: Avalonia `TextBox` 대신 PasswordBox/Masking 사용(별도 컨트롤/커스텀).
- **토큰 저장**: 메모리 우선, 자동 로그인 필요 시 OS별 안전 저장소(예: DPAPI/Mac Keychain/SecretService) 고려. JSON 파일 평문 저장 지양.
- **TLS 설정/인증서 고정(Pinning)**: 고보안 환경에서 DelegatingHandler로 구현 가능.
- **요청/응답 크기 제한**: 서버가 `Content-Length` 제한, 클라이언트는 `MaxResponseContentBufferSize`나 스트리밍 처리.
- **진단**: OpenTelemetry/ActivitySource로 상관관계 추적, Serilog로 요청/응답 요약 로깅.

---

## 13. 전체 흐름 도식

1) View → ViewModel: 사용자 액션(검색/정렬/페이지)
2) ViewModel → Repository: 쿼리 파라미터와 함께 API 요청
3) HttpClientFactory 파이프라인: 인증/로깅/Polly 정책 적용
4) Repository: 응답 파싱 → 도메인 모델 반환
5) ViewModel: 상태 업데이트(Items/IsBusy/Error)
6) View: DataGrid/ProgressBar/오류 TextBlock 반영

---

## 14. 요약 표

| 항목 | 구현 포인트 | 테스트 포인트 |
|------|-------------|---------------|
| HttpClient 구성 | HttpClientFactory, Named Client, 핸들러 파이프라인 | 핸들러 스텁으로 Repository 단위 테스트 |
| Repository | 직렬화/오류 캡슐화, 페이징/정렬/검색, ETag | 성공/실패/304/412 케이스 |
| ViewModel | Load/Cancel, Busy/Error, ReactiveCommand | Repository 모킹으로 상태 전이 검증 |
| 회복력 | Polly 재시도/타임아웃/서킷, 429 백오프 | 재시도 횟수/지연/실패 최종 메시지 |
| 인증 | DelegatingHandler에서 Bearer | 토큰 부재/만료 시 동작 |

---

## 15. 결론

- 초안의 기본 CRUD 예시를 **HttpClientFactory/Polly/DelegatingHandler**로 **운영 내구성**있게 확장했다.
- **Repository 캡슐화** 덕분에 ViewModel 테스트는 간단하며, API 변화에도 UI는 안정적이다.
- **ETag/If-Match**, **취소 토큰**, **지터 백오프** 등은 실서비스의 “체력”을 좌우한다.
  본 템플릿을 바탕으로 도메인별 DTO/Mapper, 캐시·오프라인 전략, 오류 UX(ProblemDetails 매핑)를 더해 **현업 품질의 Avalonia MVVM + Web API** 스택을 구축하자.
