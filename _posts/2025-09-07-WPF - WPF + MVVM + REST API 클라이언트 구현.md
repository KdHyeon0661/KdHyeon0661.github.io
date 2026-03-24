---
layout: post
title: WPF - WPF + MVVM + REST API 클라이언트 구현
date: 2025-09-07 21:25:23 +0900
category: WPF
---
# WPF + MVVM + REST API 클라이언트 구현 완전 정복

이 글에서는 WPF 애플리케이션에서 MVVM 패턴을 적용해 REST API를 호출하는 안정적이고 확장 가능한 클라이언트를 구현하는 방법을 다룹니다. .NET 6/7/8 기준으로 설명하며, .NET Framework 4.8에서도 큰 차이 없이 적용할 수 있습니다.

## 데모 시나리오

- **리소스**: Products (목록, 단건, 검색, 페이징, 정렬)
- **API 엔드포인트 예시**
  - `GET /products?page={p}&size={s}&q={query}&sort={name|price}`
  - `GET /products/{id}`
  - `POST /products`
  - `PUT /products/{id}`
  - `DELETE /products/{id}`
- **인증**: Bearer Token (Access/Refresh)
- **UI 구성**: 목록, 검색, 페이징, 정렬, 상세, 생성, 수정, 삭제, 에러/진행 상태 표시
- **구현 패턴**: MVVM + 의존성 주입 + HttpClientFactory + Polly + 취소 토큰 + 낙관적 업데이트

## 솔루션 구조

```
Shop/
  Shop.App/               # WPF 애플리케이션 (Views, ViewModels, 부트스트래핑)
  Shop.Domain/            # DTO, 계약, 유효성 검사
  Shop.ApiClient/         # REST 클라이언트 (HttpClientFactory/핸들러/Polly)
  Shop.Core/              # 인프라: 추상화, 유틸리티 (Result, IClock, IDispatcher)
  Shop.Tests/             # 단위 테스트
```

## 핵심 패키지

```bash
dotnet add Shop.App package Microsoft.Extensions.Hosting
dotnet add Shop.App package CommunityToolkit.Mvvm
dotnet add Shop.ApiClient package Microsoft.Extensions.Http
dotnet add Shop.ApiClient package Polly.Extensions.Http
dotnet add Shop.Tests package FluentAssertions
dotnet add Shop.Tests package NSubstitute
```

## 도메인 계층: DTO와 결과 객체

```csharp
// Shop.Domain/Products/ProductDto.cs
public sealed record ProductDto(
    Guid Id,
    string Name,
    decimal Price,
    string? Description,
    DateTimeOffset UpdatedAt);

// Shop.Domain/Common/Result.cs
public readonly struct Result<T>
{
    public bool Ok { get; }
    public T? Value { get; }
    public string? Error { get; }
    public int? StatusCode { get; }
    private Result(bool ok, T? value, string? error, int? status)
        => (Ok, Value, Error, StatusCode) = (ok, value, error, status);
    public static Result<T> Success(T value) => new(true, value, null, null);
    public static Result<T> Fail(string message, int? status = null) => new(false, default, message, status);
}
```

## API 클라이언트 설계

### 인터페이스 정의

```csharp
// Shop.ApiClient/IProductsApi.cs
public interface IProductsApi
{
    Task<Result<(IReadOnlyList<ProductDto> items, int total)>> GetAsync(
        int page, int size, string? query, string? sort, CancellationToken ct);

    Task<Result<ProductDto>> GetByIdAsync(Guid id, CancellationToken ct);
    Task<Result<ProductDto>> CreateAsync(ProductDto create, CancellationToken ct);
    Task<Result<ProductDto>> UpdateAsync(Guid id, ProductDto update, string? ifMatch, CancellationToken ct);
    Task<Result<bool>> DeleteAsync(Guid id, string? ifMatch, CancellationToken ct);
}
```

### 의존성 주입 + HttpClientFactory + Polly 설정

```csharp
// Shop.ApiClient/ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddApiClients(this IServiceCollection services, Uri baseAddress)
    {
        services.AddTransient<AuthHeaderHandler>(); // 토큰 자동 주입 핸들러

        services.AddHttpClient<IProductsApi, ProductsApi>(client =>
        {
            client.BaseAddress = baseAddress;
            client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
        })
        .AddHttpMessageHandler<AuthHeaderHandler>()
        .AddPolicyHandler(GetRetryPolicy())
        .AddPolicyHandler(GetCircuitBreaker());

        return services;
    }

    static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
        => HttpPolicyExtensions.HandleTransientHttpError()
           .OrResult(response => response.StatusCode == HttpStatusCode.TooManyRequests)
           .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromMilliseconds(200 * retryAttempt));

    static IAsyncPolicy<HttpResponseMessage> GetCircuitBreaker()
        => HttpPolicyExtensions.HandleTransientHttpError()
           .CircuitBreakerAsync(5, TimeSpan.FromSeconds(15));
}
```

### 인증 메시지 핸들러

```csharp
// Shop.ApiClient/AuthHeaderHandler.cs
public interface ITokenProvider
{
    ValueTask<string?> GetAccessTokenAsync(CancellationToken ct);
}

public sealed class AuthHeaderHandler : DelegatingHandler
{
    private readonly ITokenProvider _tokenProvider;
    public AuthHeaderHandler(ITokenProvider tokenProvider) => _tokenProvider = tokenProvider;

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
    {
        var token = await _tokenProvider.GetAccessTokenAsync(ct);
        if (!string.IsNullOrWhiteSpace(token))
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
        
        var response = await base.SendAsync(request, ct);

        if (response.StatusCode == HttpStatusCode.Unauthorized)
        {
            // Refresh 토큰 시도, 로그아웃, 재인증 UI 트리거 등의 로직 구현
        }
        return response;
    }
}
```

### API 클라이언트 구현 (직렬화 + ETag/If-Match 지원)

```csharp
// Shop.ApiClient/ProductsApi.cs
public sealed class ProductsApi : IProductsApi
{
    private readonly HttpClient _httpClient;
    private static readonly JsonSerializerOptions JsonOptions = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase
    };

    public ProductsApi(HttpClient httpClient) => _httpClient = httpClient;

    public async Task<Result<(IReadOnlyList<ProductDto>, int)>> GetAsync(
        int page, int size, string? query, string? sort, CancellationToken ct)
    {
        var url = $"/products?page={page}&size={size}";
        if (!string.IsNullOrWhiteSpace(query)) url += $"&q={Uri.EscapeDataString(query)}";
        if (!string.IsNullOrWhiteSpace(sort)) url += $"&sort={sort}";

        var response = await _httpClient.GetAsync(url, ct);
        if (!response.IsSuccessStatusCode)
            return Result<(IReadOnlyList<ProductDto>, int)>.Fail(
                $"GET 실패: {(int)response.StatusCode}", (int)response.StatusCode);

        var items = await response.Content.ReadFromJsonAsync<List<ProductDto>>(JsonOptions, ct) ?? new List<ProductDto>();
        int total = 0;
        if (response.Headers.TryGetValues("X-Total-Count", out var values) && int.TryParse(values.FirstOrDefault(), out var parsedTotal))
            total = parsedTotal;

        return Result<(IReadOnlyList<ProductDto>, int)>.Success((items, total));
    }

    public async Task<Result<ProductDto>> CreateAsync(ProductDto create, CancellationToken ct)
    {
        var response = await _httpClient.PostAsJsonAsync("/products", create, JsonOptions, ct);
        if (!response.IsSuccessStatusCode)
            return Result<ProductDto>.Fail($"POST 실패: {(int)response.StatusCode}", (int)response.StatusCode);

        var dto = await response.Content.ReadFromJsonAsync<ProductDto>(JsonOptions, ct);
        return dto is null ? Result<ProductDto>.Fail("잘못된 응답 형식") : Result<ProductDto>.Success(dto);
    }
    // 나머지 메서드 (GetById, Update, Delete) 생략
}
```

## WPF 애플리케이션 부트스트랩 (Generic Host 사용)

```csharp
// Shop.App/App.xaml.cs
public partial class App : Application
{
    public static IHost HostApplication { get; } = Host.CreateDefaultBuilder()
        .ConfigureServices((context, services) =>
        {
            var apiBaseAddress = new Uri("https://api.example.com");
            services.AddSingleton<ITokenProvider, MemoryTokenProvider>();
            services.AddApiClients(apiBaseAddress);
            services.AddSingleton<MainViewModel>();
        })
        .Build();

    protected override void OnStartup(StartupEventArgs e)
    {
        HostApplication.Start();
        base.OnStartup(e);
        var mainViewModel = HostApplication.Services.GetRequiredService<MainViewModel>();
        new MainWindow { DataContext = mainViewModel }.Show();
    }

    protected override void OnExit(ExitEventArgs e)
    {
        HostApplication.Dispose();
        base.OnExit(e);
    }
}

public sealed class MemoryTokenProvider : ITokenProvider
{
    public ValueTask<string?> GetAccessTokenAsync(CancellationToken ct) => new("demo-token");
}
```

## MVVM: 메인 ViewModel (목록/검색/페이징/정렬/상태/에러/취소)

```csharp
// Shop.App/ViewModels/MainViewModel.cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Collections.ObjectModel;

public partial class MainViewModel : ObservableObject
{
    private readonly IProductsApi _api;
    private CancellationTokenSource? _cts;

    public ObservableCollection<ProductDto> Items { get; } = new();

    [ObservableProperty]
    private int _page = 1;
    [ObservableProperty]
    private int _size = 20;
    [ObservableProperty]
    private string? _query;
    [ObservableProperty]
    private string? _sort;
    [ObservableProperty]
    private int _total;
    [ObservableProperty]
    private bool _isBusy;
    [ObservableProperty]
    private string? _error;

    public MainViewModel(IProductsApi api) => _api = api;

    [RelayCommand]
    private async Task LoadAsync()
    {
        Cancel();
        _cts = new CancellationTokenSource();
        try
        {
            IsBusy = true;
            Error = null;
            var result = await _api.GetAsync(Page, Size, Query, Sort, _cts.Token);
            if (!result.Ok)
            {
                Error = result.Error;
                return;
            }

            Items.Clear();
            foreach (var p in result.Value.items) Items.Add(p);
            Total = result.Value.total;
        }
        catch (OperationCanceledException) { }
        catch (Exception ex)
        {
            Error = ex.Message;
        }
        finally
        {
            IsBusy = false;
        }
    }

    [RelayCommand]
    private async Task NextPageAsync()
    {
        if (Page * Size >= Total) return;
        Page++;
        await LoadAsync();
    }

    [RelayCommand]
    private async Task PrevPageAsync()
    {
        if (Page <= 1) return;
        Page--;
        await LoadAsync();
    }

    [RelayCommand]
    private async Task SearchAsync()
    {
        Page = 1;
        await LoadAsync();
    }

    [RelayCommand]
    private void Cancel()
    {
        _cts?.Cancel();
        _cts?.Dispose();
        _cts = null;
    }
}
```

## View: 상태/진행/에러/검색/페이징 바인딩

```xml
<!-- Shop.App/MainWindow.xaml -->
<Window x:Class="Shop.App.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        Title="Products" Width="800" Height="500"
        Loaded="{Binding LoadCommand}">
    <DockPanel Margin="12">
        <!-- 상단 검색 도구 -->
        <StackPanel Orientation="Horizontal" DockPanel.Dock="Top">
            <TextBox Width="200" Text="{Binding Query, UpdateSourceTrigger=PropertyChanged}"/>
            <Button Content="검색" Command="{Binding SearchCommand}"/>
            <Button Content="새로고침" Command="{Binding LoadCommand}"/>
            <Button Content="취소" Command="{Binding CancelCommand}" IsEnabled="{Binding IsBusy}"/>
            <TextBlock Margin="16,0,0,0" Text="{Binding Total, StringFormat=총 {0} 개}"/>
            <ProgressBar Width="100" Height="16" IsIndeterminate="True" 
                         Visibility="{Binding IsBusy, Converter={StaticResource BoolToVis}}"/>
        </StackPanel>

        <!-- 제품 목록 -->
        <DataGrid ItemsSource="{Binding Items}" AutoGenerateColumns="False" IsReadOnly="True">
            <DataGrid.Columns>
                <DataGridTextColumn Header="이름" Binding="{Binding Name}"/>
                <DataGridTextColumn Header="가격" Binding="{Binding Price, StringFormat=C}"/>
                <DataGridTextColumn Header="수정일" Binding="{Binding UpdatedAt}"/>
            </DataGrid.Columns>
        </DataGrid>

        <!-- 하단 페이징 -->
        <StackPanel Orientation="Horizontal" DockPanel.Dock="Bottom" HorizontalAlignment="Center">
            <Button Content="이전" Command="{Binding PrevPageCommand}"/>
            <TextBlock Margin="8,0" Text="{Binding Page}"/>
            <Button Content="다음" Command="{Binding NextPageCommand}"/>
        </StackPanel>

        <!-- 오류 메시지 -->
        <Border DockPanel.Dock="Bottom" Background="Red" Padding="8"
                Visibility="{Binding Error, Converter={StaticResource NullToVis}}">
            <TextBlock Foreground="White" Text="{Binding Error}"/>
        </Border>
    </DockPanel>
</Window>
```

## 고급 UX 기능: 낙관적 업데이트

```csharp
[RelayCommand(CanExecute = nameof(CanEdit))]
private async Task DeleteAsync()
{
    if (SelectedProduct is null) return;
    IsBusy = true;
    try
    {
        var index = Items.IndexOf(SelectedProduct);
        var itemToRemove = SelectedProduct;
        Items.RemoveAt(index); // 낙관적 삭제

        var result = await _api.DeleteAsync(itemToRemove.Id, null, CancellationToken.None);
        if (!result.Ok)
        {
            Error = result.Error;
            Items.Insert(index, itemToRemove); // 롤백
            return;
        }
        Total--;
    }
    finally
    {
        IsBusy = false;
    }
}
```

## 에러 처리 및 표시 패턴

| 상황 | 처리 |
|------|------|
| 서버 오류 (4xx/5xx) | `Result.Fail`로 전달, UI에 메시지 표시 |
| 네트워크 오류 | Polly 재시도 후 실패 시 사용자 메시지 |
| 작업 취소 | `OperationCanceledException` 무시 |
| 인증 오류 (401) | AuthHeaderHandler에서 refresh 또는 재인증 트리거 |

## 테스트 전략

### HttpMessageHandler 모의 객체를 사용한 API 테스트

```csharp
// Shop.Tests/HttpTestHandler.cs
public sealed class HttpTestHandler : HttpMessageHandler
{
    public Func<HttpRequestMessage, HttpResponseMessage>? Responder { get; set; }
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
        => Task.FromResult(Responder?.Invoke(request) ?? new HttpResponseMessage(HttpStatusCode.NotFound));
}

[Fact]
public async Task GetAsync_Returns_List_And_Total()
{
    var handler = new HttpTestHandler
    {
        Responder = request =>
        {
            var response = new HttpResponseMessage(HttpStatusCode.OK);
            response.Content = JsonContent.Create(new[] { new ProductDto(...) });
            response.Headers.Add("X-Total-Count", "42");
            return response;
        }
    };
    var httpClient = new HttpClient(handler) { BaseAddress = new Uri("https://fake/") };
    var api = new ProductsApi(httpClient);

    var result = await api.GetAsync(1, 20, null, null, CancellationToken.None);

    Assert.True(result.Ok);
    Assert.Equal(42, result.Value.total);
}
```

### ViewModel 테스트

```csharp
[Fact]
public async Task Load_Sets_Items_And_Total()
{
    var mockApi = Substitute.For<IProductsApi>();
    mockApi.GetAsync(1, 20, null, null, Arg.Any<CancellationToken>())
           .Returns(Result<(IReadOnlyList<ProductDto>, int)>.Success(
               (new List<ProductDto> { new(...) }, 10)));

    var vm = new MainViewModel(mockApi);
    await vm.LoadAsync();

    Assert.Single(vm.Items);
    Assert.Equal(10, vm.Total);
}
```

## 성능 및 안정성 고려사항

- **비동기**: 모든 I/O 작업은 `async/await`, UI 업데이트는 Dispatcher 통해 처리
- **취소 토큰**: 장시간 호출에 `CancellationToken` 전파
- **UI 가상화**: `VirtualizingPanel.IsVirtualizing="True"` 설정
- **적절한 페이지 크기**: 20~50개 항목
- **Polly 정책**: 지수 백오프 적용, 429 처리
- **메모리 관리**: 이미지/대용량 페이로드 스트리밍 고려

---

## 결론

이 구현은 WPF + MVVM + REST API 클라이언트를 구성하는 견고한 아키텍처를 제시합니다. 핵심 원칙은 다음과 같습니다.

- **관심사 분리**: 네트워킹, 비즈니스 로직, UI를 계층화하여 유지보수성을 높입니다.
- **회복력 있는 통신**: `HttpClientFactory`와 `Polly`로 일시적 오류에 대한 표준 대응 체계를 구축합니다.
- **반응형 UI 상태 관리**: `IsBusy`, `Error` 같은 상태 속성을 중심으로 UI가 자동으로 반응합니다.
- **테스트 가능성**: 의존성 주입을 통해 외부 의존성을 모의 객체로 대체하여 단위 테스트를 용이하게 합니다.
- **현대적 .NET 생태계 활용**: Generic Host, CommunityToolkit.Mvvm, System.Text.Json 등 최신 라이브러리를 활용해 생산성을 높입니다.

이 구조는 페이징, 검색, 정렬, 낙관적 업데이트, 오류 처리, 작업 취소 등 현실적인 요구사항을 아우르는 확장 가능한 기반을 제공합니다.