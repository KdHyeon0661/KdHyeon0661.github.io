---
layout: post
title: WPF - WPF + MVVM + REST API 클라이언트 구현
date: 2025-09-07 21:25:23 +0900
category: WPF
---
# 🔗 WPF + MVVM + REST API 클라이언트 구현 완전 정복
*(예제 중심 · 누락 없이 자세하게 · DI/HttpClientFactory/Polly/에러 처리/페이징/검색/토큰/테스트/오프라인 큐까지)*

> 목표: **WPF(MVVM)** 앱에서 **REST API**를 안전하고 확장성 있게 호출하고,  
> **응답 바인딩 · 오류/토큰/취소/재시도/페이징/검색/상태 표시**를 완비한 “실전 구조”를 만든다.  
> .NET 7/8 WPF 기준이며 .NET 6/Framework 4.8도 큰 틀은 동일합니다.

---

## 0. 데모 시나리오

- 리소스: `Products` (목록/단건/검색/페이지네이션/정렬)
- API 엔드포인트(예시):
  - `GET    /products?page={p}&size={s}&q={query}&sort={name|price}`
  - `GET    /products/{id}`
  - `POST   /products`
  - `PUT    /products/{id}`
  - `DELETE /products/{id}`
- 인증: Bearer Token (Access/Refresh)
- UI: 목록/검색/페이징/정렬/상세/생성/수정/삭제 + 에러/진행/오프라인 표시
- 패턴: **MVVM + DI(Generic Host) + HttpClientFactory + Polly**  
  + **Cancellation** + **Progress** + **Optimistic Update** + **테스트 가능한 설계**

---

## 1. 솔루션 구조

```
Shop/
  Shop.App/               # WPF (Views, ViewModels, Bootstrapping)
  Shop.Domain/            # DTO/Contracts/Validation/Mappers
  Shop.ApiClient/         # REST client (HttpClientFactory/Handlers/Polly)
  Shop.Core/              # Infrastructure: Abstractions, Utils (Result, IClock, IDispatcher)
  Shop.Tests/             # Unit + Integration tests (mocked HttpMessageHandler)
```

---

## 2. 패키지

```bash
# App
dotnet add Shop.App package Microsoft.Extensions.Hosting
dotnet add Shop.App package Microsoft.Extensions.DependencyInjection
dotnet add Shop.App package CommunityToolkit.Mvvm

# API Client
dotnet add Shop.ApiClient package Microsoft.Extensions.Http
dotnet add Shop.ApiClient package Polly.Extensions.Http
dotnet add Shop.ApiClient package System.Text.Json

# Tests
dotnet add Shop.Tests package FluentAssertions
dotnet add Shop.Tests package NSubstitute
```

---

## 3. Domain: DTO & Result & Validation

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

---

## 4. API 클라이언트 설계

### 4.1 인터페이스

```csharp
// Shop.ApiClient/IProductsApi.cs
public interface IProductsApi
{
    Task<Result<(IReadOnlyList<ProductDto> items, int total)>> GetAsync(
        int page, int size, string? query, string? sort, CancellationToken ct);

    Task<Result<ProductDto>> GetByIdAsync(Guid id, CancellationToken ct);
    Task<Result<ProductDto>> CreateAsync(ProductDto create, CancellationToken ct);
    Task<Result<ProductDto>> UpdateAsync(Guid id, ProductDto update, string? ifMatch, CancellationToken ct);
    Task<Result<bool>>       DeleteAsync(Guid id, string? ifMatch, CancellationToken ct);
}
```

### 4.2 DI + HttpClientFactory + Polly

```csharp
// Shop.ApiClient/ServiceCollectionExtensions.cs
using Microsoft.Extensions.DependencyInjection;
using Polly;
using Polly.Extensions.Http;
using System.Net;
using System.Net.Http.Headers;

public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddApiClients(this IServiceCollection services, Uri baseAddress)
    {
        services.AddTransient<AuthHeaderHandler>(); // 아래 정의: 토큰 자동 주입

        services.AddHttpClient<IProductsApi, ProductsApi>(c =>
        {
            c.BaseAddress = baseAddress;
            c.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
        })
        .AddHttpMessageHandler<AuthHeaderHandler>()
        .AddPolicyHandler(GetRetryPolicy())
        .AddPolicyHandler(GetCircuitBreaker());

        return services;
    }

    static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
        => HttpPolicyExtensions.HandleTransientHttpError()
           .OrResult(r => r.StatusCode == HttpStatusCode.TooManyRequests)
           .WaitAndRetryAsync(3, i => TimeSpan.FromMilliseconds(200 * i));

    static IAsyncPolicy<HttpResponseMessage> GetCircuitBreaker()
        => HttpPolicyExtensions.HandleTransientHttpError()
           .CircuitBreakerAsync(5, TimeSpan.FromSeconds(15));
}
```

### 4.3 인증 메시지 핸들러(토큰 자동 주입 + 401 처리 훅)

```csharp
// Shop.ApiClient/AuthHeaderHandler.cs
using System.Net.Http.Headers;
using System.Threading;
using System.Threading.Tasks;

public interface ITokenProvider
{
    ValueTask<string?> GetAccessTokenAsync(CancellationToken ct);
    // 필요시 Refresh 지원
}

public sealed class AuthHeaderHandler : DelegatingHandler
{
    private readonly ITokenProvider _tokens;
    public AuthHeaderHandler(ITokenProvider tokens) => _tokens = tokens;

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
    {
        var token = await _tokens.GetAccessTokenAsync(ct);
        if (!string.IsNullOrWhiteSpace(token))
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
        var res = await base.SendAsync(request, ct);

        if (res.StatusCode == System.Net.HttpStatusCode.Unauthorized)
        {
            // TODO: Refresh 토큰 시도/로그아웃/재인증 UI 트리거 등
        }
        return res;
    }
}
```

### 4.4 구현(직렬화 옵션 + ETag/If-Match 지원)

```csharp
// Shop.ApiClient/ProductsApi.cs
using System.Net;
using System.Net.Http.Json;
using System.Text.Json;

public sealed class ProductsApi : IProductsApi
{
    private readonly HttpClient _http;
    private static readonly JsonSerializerOptions JsonOpts = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase
    };

    public ProductsApi(HttpClient http) => _http = http;

    public async Task<Result<(IReadOnlyList<ProductDto>, int)>> GetAsync(
        int page, int size, string? query, string? sort, CancellationToken ct)
    {
        var url = $"/products?page={page}&size={size}";
        if (!string.IsNullOrWhiteSpace(query)) url += $"&q={Uri.EscapeDataString(query)}";
        if (!string.IsNullOrWhiteSpace(sort))  url += $"&sort={sort}";

        var res = await _http.GetAsync(url, ct);
        if (!res.IsSuccessStatusCode)
            return Result<(IReadOnlyList<ProductDto>, int)>.Fail($"GET failed: {(int)res.StatusCode}", (int)res.StatusCode);

        var items = await res.Content.ReadFromJsonAsync<List<ProductDto>>(JsonOpts, ct) ?? new();
        int total = 0;
        if (res.Headers.TryGetValues("X-Total-Count", out var vals) && int.TryParse(vals.FirstOrDefault(), out var t))
            total = t;

        return Result<(IReadOnlyList<ProductDto>, int)>.Success((items, total));
    }

    public async Task<Result<ProductDto>> GetByIdAsync(Guid id, CancellationToken ct)
    {
        var res = await _http.GetAsync($"/products/{id}", ct);
        if (res.StatusCode == HttpStatusCode.NotFound)
            return Result<ProductDto>.Fail("Not found", 404);
        if (!res.IsSuccessStatusCode)
            return Result<ProductDto>.Fail($"GET failed: {(int)res.StatusCode}", (int)res.StatusCode);

        var dto = await res.Content.ReadFromJsonAsync<ProductDto>(JsonOpts, ct);
        return dto is null
            ? Result<ProductDto>.Fail("Invalid payload")
            : Result<ProductDto>.Success(dto);
    }

    public async Task<Result<ProductDto>> CreateAsync(ProductDto create, CancellationToken ct)
    {
        var res = await _http.PostAsJsonAsync("/products", create, JsonOpts, ct);
        if (!res.IsSuccessStatusCode)
            return Result<ProductDto>.Fail($"POST failed: {(int)res.StatusCode}", (int)res.StatusCode);

        var dto = await res.Content.ReadFromJsonAsync<ProductDto>(JsonOpts, ct);
        return dto is null ? Result<ProductDto>.Fail("Invalid payload") : Result<ProductDto>.Success(dto);
    }

    public async Task<Result<ProductDto>> UpdateAsync(Guid id, ProductDto update, string? ifMatch, CancellationToken ct)
    {
        var req = new HttpRequestMessage(HttpMethod.Put, $"/products/{id}")
        { Content = JsonContent.Create(update, options: JsonOpts) };
        if (!string.IsNullOrWhiteSpace(ifMatch))
            req.Headers.TryAddWithoutValidation("If-Match", ifMatch);

        var res = await _http.SendAsync(req, ct);
        if (res.StatusCode == HttpStatusCode.PreconditionFailed)
            return Result<ProductDto>.Fail("Precondition failed(ETag mismatch)", 412);
        if (!res.IsSuccessStatusCode)
            return Result<ProductDto>.Fail($"PUT failed: {(int)res.StatusCode}", (int)res.StatusCode);

        var dto = await res.Content.ReadFromJsonAsync<ProductDto>(JsonOpts, ct);
        return dto is null ? Result<ProductDto>.Fail("Invalid payload") : Result<ProductDto>.Success(dto);
    }

    public async Task<Result<bool>> DeleteAsync(Guid id, string? ifMatch, CancellationToken ct)
    {
        var req = new HttpRequestMessage(HttpMethod.Delete, $"/products/{id}");
        if (!string.IsNullOrWhiteSpace(ifMatch))
            req.Headers.TryAddWithoutValidation("If-Match", ifMatch);

        var res = await _http.SendAsync(req, ct);
        if (res.StatusCode == HttpStatusCode.PreconditionFailed)
            return Result<bool>.Fail("Precondition failed(ETag mismatch)", 412);
        if (res.StatusCode == HttpStatusCode.NotFound)
            return Result<bool>.Fail("Not found", 404);
        if (!res.IsSuccessStatusCode)
            return Result<bool>.Fail($"DELETE failed: {(int)res.StatusCode}", (int)res.StatusCode);

        return Result<bool>.Success(true);
    }
}
```

---

## 5. App 부트스트랩 (Generic Host in WPF)

```csharp
// Shop.App/App.xaml.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

public partial class App : Application
{
    public static IHost HostApp { get; } = Host.CreateDefaultBuilder()
        .ConfigureServices((ctx, s) =>
        {
            var apiBase = new Uri("https://api.example.com");
            s.AddSingleton<ITokenProvider, MemoryTokenProvider>(); // 데모용
            s.AddApiClients(apiBase);
            s.AddSingleton<MainViewModel>();
        })
        .Build();

    protected override void OnStartup(StartupEventArgs e)
    {
        HostApp.Start();
        base.OnStartup(e);
        var vm = HostApp.Services.GetRequiredService<MainViewModel>();
        new MainWindow { DataContext = vm }.Show();
    }

    protected override void OnExit(ExitEventArgs e)
    {
        HostApp.Dispose();
        base.OnExit(e);
    }
}

public sealed class MemoryTokenProvider : ITokenProvider
{
    public ValueTask<string?> GetAccessTokenAsync(CancellationToken ct) => new("demo-access-token");
}
```

---

## 6. MVVM: 메인 ViewModel (목록/검색/페이징/정렬/상태/에러/취소)

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

    [ObservableProperty] private int page = 1;
    [ObservableProperty] private int size = 20;
    [ObservableProperty] private string? query;
    [ObservableProperty] private string? sort; // "name" | "price"
    [ObservableProperty] private int total;
    [ObservableProperty] private bool isBusy;
    [ObservableProperty] private string? error;

    public MainViewModel(IProductsApi api) => _api = api;

    [RelayCommand]
    private async Task LoadAsync()
    {
        CancelInFlight();
        _cts = new CancellationTokenSource();
        try
        {
            IsBusy = true; Error = null;
            var result = await _api.GetAsync(Page, Size, Query, Sort, _cts.Token);
            if (!result.Ok) { Error = result.Error; return; }

            Items.Clear();
            foreach (var p in result.Value!.items)
                Items.Add(p);
            Total = result.Value.Value.total;
        }
        catch (OperationCanceledException) { /* 무시 */ }
        catch (Exception ex) { Error = ex.Message; }
        finally { IsBusy = false; }
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
        Page = 1; // 검색은 첫 페이지로
        await LoadAsync();
    }

    [RelayCommand]
    private async Task RefreshAsync() => await LoadAsync();

    [RelayCommand]
    private void Cancel() => CancelInFlight();

    private void CancelInFlight()
    {
        try { _cts?.Cancel(); } catch { }
        _cts?.Dispose(); _cts = null;
    }
}
```

---

## 7. View: 상태/진행/에러/검색/페이징 바인딩

```xml
<!-- Shop.App/MainWindow.xaml -->
<Window x:Class="Shop.App.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Products" Width="820" Height="540"
        Loaded="{Binding LoadCommand}">
  <DockPanel Margin="12">
    <!-- 상단 바 -->
    <StackPanel Orientation="Horizontal" DockPanel.Dock="Top" Spacing="8">
      <TextBox Width="240" x:Name="SearchBox" Text="{Binding Query, UpdateSourceTrigger=PropertyChanged}" />
      <ComboBox Width="120" SelectedItem="{Binding Sort}" >
        <ComboBoxItem Content="name"/>
        <ComboBoxItem Content="price"/>
      </ComboBox>
      <Button Content="검색" Command="{Binding SearchCommand}"/>
      <Button Content="새로고침" Command="{Binding RefreshCommand}"/>
      <Button Content="취소" Command="{Binding CancelCommand}" IsEnabled="{Binding IsBusy}"/>
      <TextBlock Margin="16,0,0,0">
        <Run Text="총 "/><Run Text="{Binding Total}"/><Run Text=" 개"/>
      </TextBlock>
      <ProgressBar Width="120" Height="8" IsIndeterminate="True" Visibility="{Binding IsBusy, Converter={StaticResource BoolToVisibility}}"/>
    </StackPanel>

    <!-- 리스트 -->
    <DataGrid ItemsSource="{Binding Items}" AutoGenerateColumns="False" IsReadOnly="True">
      <DataGrid.Columns>
        <DataGridTextColumn Header="Id" Binding="{Binding Id}"/>
        <DataGridTextColumn Header="Name" Binding="{Binding Name}"/>
        <DataGridTextColumn Header="Price" Binding="{Binding Price, StringFormat={}{0:C}}"/>
        <DataGridTextColumn Header="Updated" Binding="{Binding UpdatedAt}"/>
      </DataGrid.Columns>
    </DataGrid>

    <!-- 페이징 -->
    <StackPanel Orientation="Horizontal" DockPanel.Dock="Bottom" HorizontalAlignment="Center" Margin="0,8,0,0">
      <Button Content="◀" Command="{Binding PrevPageCommand}"/>
      <TextBlock Margin="8,0" VerticalAlignment="Center">
        <Run Text="{Binding Page}"/><Run Text=" / "/>
        <Run Text="{Binding Total, Converter={StaticResource TotalToPages}, ConverterParameter={Binding Size}}"/>
      </TextBlock>
      <Button Content="▶" Command="{Binding NextPageCommand}"/>
    </StackPanel>

    <!-- 에러 -->
    <Border DockPanel.Dock="Bottom" Background="#FFEF4444" CornerRadius="6" Padding="8"
            Visibility="{Binding Error, Converter={StaticResource NullToCollapsed}}">
      <TextBlock Foreground="White" Text="{Binding Error}"/>
    </Border>
  </DockPanel>
</Window>
```

보조 컨버터(간단 구현 예):
```csharp
public class BoolToVisibility : IValueConverter
{
    public object Convert(object v, Type t, object p, CultureInfo c)
        => (v is bool b && b) ? Visibility.Visible : Visibility.Collapsed;
    public object ConvertBack(object v, Type t, object p, CultureInfo c) => Binding.DoNothing;
}
public class NullToCollapsed : IValueConverter
{
    public object Convert(object v, Type t, object p, CultureInfo c)
        => v is null ? Visibility.Collapsed : Visibility.Visible;
    public object ConvertBack(object v, Type t, object p, CultureInfo c) => Binding.DoNothing;
}
public class TotalToPages : IValueConverter
{
    public object Convert(object total, Type t, object param, CultureInfo c)
    {
        if (total is int T && param is BindingExpression be && be.DataItem is MainViewModel vm)
            return Math.Max(1, (int)Math.Ceiling((double)T / vm.Size));
        return 1;
    }
    public object ConvertBack(object v, Type t, object p, CultureInfo c) => Binding.DoNothing;
}
```

> 팁: **가상화 활성화**  
> `<DataGrid VirtualizingPanel.IsVirtualizing="True" VirtualizingPanel.VirtualizationMode="Recycling" />`

---

## 8. 상세/등록/수정/삭제 with Optimistic Update

### 8.1 커맨드 (등록/수정/삭제, 처리 중 상태, 낙관적 갱신)

```csharp
public partial class MainViewModel : ObservableObject
{
    [ObservableProperty] private ProductDto? selected;

    [RelayCommand]
    private async Task CreateAsync()
    {
        var draft = new ProductDto(Guid.NewGuid(), "New", 0m, null, DateTimeOffset.UtcNow);
        IsBusy = true; Error = null;
        try
        {
            var res = await _api.CreateAsync(draft, CancellationToken.None);
            if (!res.Ok) { Error = res.Error; return; }
            Items.Insert(0, res.Value!);
            Total++;
        }
        catch (Exception ex) { Error = ex.Message; }
        finally { IsBusy = false; }
    }

    [RelayCommand(CanExecute = nameof(CanEdit))]
    private async Task SaveAsync()
    {
        if (Selected is null) return;
        IsBusy = true; Error = null;
        try
        {
            // 낙관적 갱신: UI 먼저 반영 (필요 시 별도 복제 후 실패 시 롤백)
            var idx = Items.IndexOf(Selected);
            var original = Items[idx];

            var updated = Selected with { UpdatedAt = DateTimeOffset.UtcNow };
            Items[idx] = updated;

            var res = await _api.UpdateAsync(updated.Id, updated, ifMatch: null, CancellationToken.None);
            if (!res.Ok)
            {
                Error = res.Error;
                Items[idx] = original; // 롤백
                return;
            }
            Items[idx] = res.Value!;
        }
        catch (Exception ex) { Error = ex.Message; }
        finally { IsBusy = false; }
    }

    [RelayCommand(CanExecute = nameof(CanEdit))]
    private async Task DeleteAsync()
    {
        if (Selected is null) return;
        IsBusy = true; Error = null;
        try
        {
            var idx = Items.IndexOf(Selected);
            var remove = Selected;
            Items.RemoveAt(idx); // 낙관적 삭제

            var res = await _api.DeleteAsync(remove.Id, ifMatch: null, CancellationToken.None);
            if (!res.Ok)
            {
                Error = res.Error;
                Items.Insert(idx, remove); // 롤백
                return;
            }
            Total--;
        }
        catch (Exception ex) { Error = ex.Message; }
        finally { IsBusy = false; }
    }

    private bool CanEdit() => Selected is not null && !IsBusy;
}
```

> 실제 운영에서는 **ETag/If-Match**로 동시성 보호를 적용하세요(서버에서 ETag 제공, 뷰모델에 저장 → 수정/삭제 시 헤더로 반환).

---

## 9. 고급 UX: 디바운스 검색 / 무한 스크롤 / 진행률

### 9.1 디바운스 검색
- TextBox의 `TextChanged`에서 **연속 입력 300ms** 후 검색 실행:
```csharp
private CancellationTokenSource? _searchCts;
partial void OnQueryChanged(string? value)
{
    _searchCts?.Cancel(); _searchCts?.Dispose();
    _searchCts = new CancellationTokenSource();
    var token = _searchCts.Token;
    _ = Task.Run(async () =>
    {
        try
        {
            await Task.Delay(300, token);
            if (!token.IsCancellationRequested)
                await Application.Current.Dispatcher.InvokeAsync(async () => await SearchAsync());
        }
        catch { }
    });
}
```

### 9.2 무한 스크롤
- `ScrollViewer.ScrollChanged`에서 **끝 근처**이면 `NextPageCommand`:
```csharp
// XAML: ScrollViewer.CanContentScroll="True" ScrollChanged="OnScroll"
private async void OnScroll(object s, ScrollChangedEventArgs e)
{
    if (e.VerticalOffset + e.ViewportHeight >= e.ExtentHeight - 48)
        await (DataContext as MainViewModel)!.NextPageAsync();
}
```

### 9.3 다운로드/업로드 진행률
- `HttpClient` 전송에 `ProgressMessageHandler` 또는 스트림 복사 래퍼 사용(생략 가능).

---

## 10. 에러 처리/표시 패턴

- **서버 오류(4xx/5xx)** → `Result.Fail(status, message)` 로 VM에 전달  
- **네트워크 오류** → Polly 재시도 후 실패 시 **친절한 메시지**  
- **취소** → `OperationCanceledException` 무시  
- UI: 상단/하단 **Alert Bar** 또는 Toast

> 공통 에러 메시지 포맷터/로거를 도입해 일관성 유지.

---

## 11. 인증(Access/Refresh) 흐름

- `AuthHeaderHandler` 에서 `401` 감지 → **Refresh 흐름** 트리거
- Refresh 성공 → 원 요청 재시도
- 실패 → **로그아웃/재인증 UI**로 전환

토큰 저장/로드:
- Windows DPAPI(ProtectedData) 또는 OS 보안 저장소 이용
- 앱 시작 시 TokenProvider가 메모리 로드

---

## 12. 오프라인/큐잉(선택)

- 요청을 **명령 큐**에 적재(예: `Create/Update/Delete` DTO)  
- 온라인 상태 확인(네트워크 체크) → 온라인 전환 시 **큐 플러시**  
- 실패/충돌 → 사용자 알림 및 수동 해결

간단한 큐 인터페이스:
```csharp
public interface IOutbox
{
    Task EnqueueAsync(HttpRequestMessage req, CancellationToken ct);
    Task FlushAsync(CancellationToken ct);
}
```

---

## 13. 로그/추적

- `HttpClientFactory` 로깅(Handler에서 `ILogger` 주입)  
- ViewModel에서 핵심 상태 변화 로그  
- 에러/성능 이벤트 수집(진단/분석)

---

## 14. 테스트 (핵심: HttpMessageHandler 스텁)

### 14.1 핸들러 목킹
```csharp
// Shop.Tests/HttpTestHandler.cs
public sealed class HttpTestHandler : HttpMessageHandler
{
    public Func<HttpRequestMessage, HttpResponseMessage>? Responder { get; set; }
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
        => Task.FromResult(Responder?.Invoke(request)
           ?? new HttpResponseMessage(System.Net.HttpStatusCode.NotFound));
}
```

### 14.2 API 테스트
```csharp
[Fact]
public async Task GetAsync_Returns_List_And_Total()
{
    var handler = new HttpTestHandler
    {
        Responder = req =>
        {
            var res = new HttpResponseMessage(HttpStatusCode.OK);
            res.Content = JsonContent.Create(new[]
            {
                new ProductDto(Guid.NewGuid(), "A", 10m, null, DateTimeOffset.UtcNow)
            });
            res.Headers.Add("X-Total-Count", "42");
            return res;
        }
    };
    var http = new HttpClient(handler) { BaseAddress = new Uri("https://fake/") };
    var api = new ProductsApi(http);

    var r = await api.GetAsync(1, 20, null, null, CancellationToken.None);

    r.Ok.Should().BeTrue();
    r.Value!.total.Should().Be(42);
    r.Value!.items.Should().HaveCount(1);
}
```

### 14.3 ViewModel 테스트(간단)
```csharp
[Fact]
public async Task Load_Sets_Items_And_Total()
{
    var api = Substitute.For<IProductsApi>();
    api.GetAsync(1, 20, null, null, Arg.Any<CancellationToken>())
       .Returns(Result<(IReadOnlyList<ProductDto>, int)>.Success(
           (new List<ProductDto>{ new(Guid.NewGuid(), "X", 5m, null, DateTimeOffset.UtcNow) }, 10)));

    var vm = new MainViewModel(api);
    await vm.LoadAsync();

    vm.Items.Should().HaveCount(1);
    vm.Total.Should().Be(10);
    vm.Error.Should().BeNull();
}
```

---

## 15. 성능/안정성 체크리스트

- **비동기 규율**: 모든 I/O `async/await` + UI 업데이트는 **Dispatcher**  
- **취소 전파**: 긴 호출마다 `CancellationToken` 지원  
- **가상화**: 리스트/그리드는 Virtualization 켜기  
- **페이지 크기**: 20~50 적정, 서버 정렬/필터 적극 활용  
- **Polly**: 과도한 재시도 금지(백오프), 429 처리  
- **메모리**: 이미지/대형 페이로드 스트리밍 처리 고려  
- **예외**: 삼켜지지 않게 로깅 + 사용자 친화 메시지

---

## 16. 보너스: 다크모드/접근성/국제화

- 다크모드: 리소스 토큰화 + `DynamicResource` (팔레트 교체)  
- 접근성: 키보드 탐색/스크린리더 Friendly Text/AutomationProperties  
- 국제화: 서버/클라 모두 UTC 저장, 표시 시 지역화/문자열 리소스

---

## 17. 마무리

- **MVVM**으로 뷰-로직 분리 → 테스트 용이  
- **HttpClientFactory + Polly**로 회복력 있는 통신  
- **상태/로딩/에러/취소/검색/페이징/정렬**을 표준화  
- **Optimistic Update + ETag 동시성**으로 UX와 무결성 균형  
- **테스트**는 Handler 스텁/대체 가능한 API 인터페이스로 가볍게
