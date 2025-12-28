---
layout: post
title: WPF - WPF + MVVM + REST API 클라이언트 구현
date: 2025-09-07 21:25:23 +0900
category: WPF
---
# WPF + MVVM + REST API 클라이언트 구현 완전 정복

> 실전 목표: WPF (MVVM) 애플리케이션에서 REST API를 안정적이고 확장 가능하게 호출하고, 응답 바인딩, 오류 처리, 토큰 관리, 요청 취소, 재시도, 페이징, 검색, 상태 표시를 갖춘 완성된 아키텍처를 구현합니다. .NET 7/8 WPF 기준이며, .NET 6 또는 .NET Framework 4.8도 기본 구조는 동일하게 적용할 수 있습니다.

## 데모 시나리오
*   리소스: Products (목록/단건/검색/페이징/정렬)
*   API 엔드포인트 예시:
    *   GET    /products?page={p}&size={s}&q={query}&sort={name|price}
    *   GET    /products/{id}
    *   POST   /products
    *   PUT    /products/{id}
    *   DELETE /products/{id}
*   인증: Bearer Token (Access/Refresh)
*   UI 구성: 목록, 검색, 페이징, 정렬, 상세, 생성, 수정, 삭제, 에러/진행/오프라인 상태 표시
*   구현 패턴: MVVM + 의존성 주입 (Generic Host) + HttpClientFactory + Polly + 취소 토큰 + 진행률 표시 + 낙관적 업데이트 + 테스트 가능한 설계

## 솔루션 구조
```
Shop/
  Shop.App/               # WPF 애플리케이션 (Views, ViewModels, 부트스트래핑)
  Shop.Domain/            # DTO, 계약, 유효성 검사, 매퍼
  Shop.ApiClient/         # REST 클라이언트 (HttpClientFactory/핸들러/Polly)
  Shop.Core/              # 인프라: 추상화, 유틸리티 (Result, IClock, IDispatcher)
  Shop.Tests/             # 단위 테스트 + 통합 테스트 (모의 HttpMessageHandler)
```

## 핵심 패키지
```bash
# WPF 애플리케이션
dotnet add Shop.App package Microsoft.Extensions.Hosting
dotnet add Shop.App package Microsoft.Extensions.DependencyInjection
dotnet add Shop.App package CommunityToolkit.Mvvm

# API 클라이언트
dotnet add Shop.ApiClient package Microsoft.Extensions.Http
dotnet add Shop.ApiClient package Polly.Extensions.Http
dotnet add Shop.ApiClient package System.Text.Json

# 테스트
dotnet add Shop.Tests package FluentAssertions
dotnet add Shop.Tests package NSubstitute
```

## 도메인 계층: DTO, 결과 객체, 유효성 검사

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

### 1. 인터페이스 정의
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

### 2. 의존성 주입 (DI) + HttpClientFactory + Polly 설정
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
        services.AddTransient<AuthHeaderHandler>(); // 아래 정의된 토큰 자동 주입 핸들러

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

### 3. 인증 메시지 핸들러 (토큰 자동 주입 + 401 처리)
```csharp
// Shop.ApiClient/AuthHeaderHandler.cs
using System.Net.Http.Headers;
using System.Threading;
using System.Threading.Tasks;

public interface ITokenProvider
{
    ValueTask<string?> GetAccessTokenAsync(CancellationToken ct);
    // 필요 시 Refresh 토큰 로직 지원
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

        if (response.StatusCode == System.Net.HttpStatusCode.Unauthorized)
        {
            // Refresh 토큰 시도, 로그아웃, 재인증 UI 트리거 등의 로직 구현
        }
        return response;
    }
}
```

### 4. API 클라이언트 구현 (직렬화 옵션 + ETag/If-Match 지원)
```csharp
// Shop.ApiClient/ProductsApi.cs
using System.Net;
using System.Net.Http.Json;
using System.Text.Json;

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

    public async Task<Result<ProductDto>> GetByIdAsync(Guid id, CancellationToken ct)
    {
        var response = await _httpClient.GetAsync($"/products/{id}", ct);
        if (response.StatusCode == HttpStatusCode.NotFound)
            return Result<ProductDto>.Fail("찾을 수 없음", 404);
        if (!response.IsSuccessStatusCode)
            return Result<ProductDto>.Fail($"GET 실패: {(int)response.StatusCode}", (int)response.StatusCode);

        var dto = await response.Content.ReadFromJsonAsync<ProductDto>(JsonOptions, ct);
        return dto is null
            ? Result<ProductDto>.Fail("잘못된 응답 형식")
            : Result<ProductDto>.Success(dto);
    }

    public async Task<Result<ProductDto>> CreateAsync(ProductDto create, CancellationToken ct)
    {
        var response = await _httpClient.PostAsJsonAsync("/products", create, JsonOptions, ct);
        if (!response.IsSuccessStatusCode)
            return Result<ProductDto>.Fail($"POST 실패: {(int)response.StatusCode}", (int)response.StatusCode);

        var dto = await response.Content.ReadFromJsonAsync<ProductDto>(JsonOptions, ct);
        return dto is null ? Result<ProductDto>.Fail("잘못된 응답 형식") : Result<ProductDto>.Success(dto);
    }

    public async Task<Result<ProductDto>> UpdateAsync(Guid id, ProductDto update, string? ifMatch, CancellationToken ct)
    {
        var request = new HttpRequestMessage(HttpMethod.Put, $"/products/{id}")
        { Content = JsonContent.Create(update, options: JsonOptions) };
        if (!string.IsNullOrWhiteSpace(ifMatch))
            request.Headers.TryAddWithoutValidation("If-Match", ifMatch);

        var response = await _httpClient.SendAsync(request, ct);
        if (response.StatusCode == HttpStatusCode.PreconditionFailed)
            return Result<ProductDto>.Fail("전제 조건 실패 (ETag 불일치)", 412);
        if (!response.IsSuccessStatusCode)
            return Result<ProductDto>.Fail($"PUT 실패: {(int)response.StatusCode}", (int)response.StatusCode);

        var dto = await response.Content.ReadFromJsonAsync<ProductDto>(JsonOptions, ct);
        return dto is null ? Result<ProductDto>.Fail("잘못된 응답 형식") : Result<ProductDto>.Success(dto);
    }

    public async Task<Result<bool>> DeleteAsync(Guid id, string? ifMatch, CancellationToken ct)
    {
        var request = new HttpRequestMessage(HttpMethod.Delete, $"/products/{id}");
        if (!string.IsNullOrWhiteSpace(ifMatch))
            request.Headers.TryAddWithoutValidation("If-Match", ifMatch);

        var response = await _httpClient.SendAsync(request, ct);
        if (response.StatusCode == HttpStatusCode.PreconditionFailed)
            return Result<bool>.Fail("전제 조건 실패 (ETag 불일치)", 412);
        if (response.StatusCode == HttpStatusCode.NotFound)
            return Result<bool>.Fail("찾을 수 없음", 404);
        if (!response.IsSuccessStatusCode)
            return Result<bool>.Fail($"DELETE 실패: {(int)response.StatusCode}", (int)response.StatusCode);

        return Result<bool>.Success(true);
    }
}
```

## WPF 애플리케이션 부트스트랩 (Generic Host 사용)

```csharp
// Shop.App/App.xaml.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

public partial class App : Application
{
    public static IHost HostApplication { get; } = Host.CreateDefaultBuilder()
        .ConfigureServices((context, services) =>
        {
            var apiBaseAddress = new Uri("https://api.example.com");
            services.AddSingleton<ITokenProvider, MemoryTokenProvider>(); // 데모용 구현
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
    public ValueTask<string?> GetAccessTokenAsync(CancellationToken ct) => new("demo-access-token");
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
    private CancellationTokenSource? _currentCancellationTokenSource;

    public ObservableCollection<ProductDto> Items { get; } = new ObservableCollection<ProductDto>();

    [ObservableProperty]
    private int _page = 1;
    [ObservableProperty]
    private int _size = 20;
    [ObservableProperty]
    private string? _query;
    [ObservableProperty]
    private string? _sort; // "name" | "price"
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
        CancelPendingRequest();
        _currentCancellationTokenSource = new CancellationTokenSource();
        try
        {
            IsBusy = true;
            Error = null;
            var result = await _api.GetAsync(Page, Size, Query, Sort, _currentCancellationTokenSource.Token);
            if (!result.Ok)
            {
                Error = result.Error;
                return;
            }

            Items.Clear();
            foreach (var product in result.Value.items)
                Items.Add(product);
            Total = result.Value.total;
        }
        catch (OperationCanceledException)
        {
            // 취소된 작업은 무시
        }
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
        Page = 1; // 검색 시 첫 페이지로 이동
        await LoadAsync();
    }

    [RelayCommand]
    private async Task RefreshAsync() => await LoadAsync();

    [RelayCommand]
    private void Cancel() => CancelPendingRequest();

    private void CancelPendingRequest()
    {
        try
        {
            _currentCancellationTokenSource?.Cancel();
        }
        catch { }
        _currentCancellationTokenSource?.Dispose();
        _currentCancellationTokenSource = null;
    }
}
```

## View: 상태/진행/에러/검색/페이징 바인딩

```xml
<!-- Shop.App/MainWindow.xaml -->
<Window x:Class="Shop.App.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Products" Width="820" Height="540"
        Loaded="{Binding LoadCommand}">
  <DockPanel Margin="12">
    <!-- 상단 도구 모음 -->
    <StackPanel Orientation="Horizontal" DockPanel.Dock="Top" Spacing="8">
      <TextBox Width="240" x:Name="SearchBox" Text="{Binding Query, UpdateSourceTrigger=PropertyChanged}" />
      <ComboBox Width="120" SelectedItem="{Binding Sort}" >
        <ComboBoxItem Content="이름"/>
        <ComboBoxItem Content="가격"/>
      </ComboBox>
      <Button Content="검색" Command="{Binding SearchCommand}"/>
      <Button Content="새로고침" Command="{Binding RefreshCommand}"/>
      <Button Content="취소" Command="{Binding CancelCommand}" IsEnabled="{Binding IsBusy}"/>
      <TextBlock Margin="16,0,0,0">
        <Run Text="총 "/><Run Text="{Binding Total}"/><Run Text=" 개"/>
      </TextBlock>
      <ProgressBar Width="120" Height="8" IsIndeterminate="True" 
                   Visibility="{Binding IsBusy, Converter={StaticResource BoolToVisibility}}"/>
    </StackPanel>

    <!-- 제품 목록 -->
    <DataGrid ItemsSource="{Binding Items}" AutoGenerateColumns="False" IsReadOnly="True">
      <DataGrid.Columns>
        <DataGridTextColumn Header="ID" Binding="{Binding Id}"/>
        <DataGridTextColumn Header="이름" Binding="{Binding Name}"/>
        <DataGridTextColumn Header="가격" Binding="{Binding Price, StringFormat={}{0:C}}"/>
        <DataGridTextColumn Header="수정일" Binding="{Binding UpdatedAt}"/>
      </DataGrid.Columns>
    </DataGrid>

    <!-- 하단 페이징 -->
    <StackPanel Orientation="Horizontal" DockPanel.Dock="Bottom" HorizontalAlignment="Center" Margin="0,8,0,0">
      <Button Content="이전" Command="{Binding PrevPageCommand}"/>
      <TextBlock Margin="8,0" VerticalAlignment="Center">
        <Run Text="{Binding Page}"/><Run Text=" / "/>
        <Run Text="{Binding Total, Converter={StaticResource TotalToPages}, ConverterParameter={Binding Size}}"/>
      </TextBlock>
      <Button Content="다음" Command="{Binding NextPageCommand}"/>
    </StackPanel>

    <!-- 오류 메시지 표시 -->
    <Border DockPanel.Dock="Bottom" Background="#FFEF4444" CornerRadius="6" Padding="8"
            Visibility="{Binding Error, Converter={StaticResource NullToCollapsed}}">
      <TextBlock Foreground="White" Text="{Binding Error}"/>
    </Border>
  </DockPanel>
</Window>
```

보조 값 변환기 (간단한 구현 예시):
```csharp
public class BoolToVisibilityConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
        => (value is bool b && b) ? Visibility.Visible : Visibility.Collapsed;
    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture) => Binding.DoNothing;
}

public class NullToCollapsedConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
        => value is null ? Visibility.Collapsed : Visibility.Visible;
    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture) => Binding.DoNothing;
}

public class TotalToPagesConverter : IValueConverter
{
    public object Convert(object total, Type targetType, object parameter, CultureInfo culture)
    {
        if (total is int totalCount && parameter is BindingExpression be && be.DataItem is MainViewModel vm)
            return Math.Max(1, (int)Math.Ceiling((double)totalCount / vm.Size));
        return 1;
    }
    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture) => Binding.DoNothing;
}
```

성능 팁: 대량의 데이터를 표시할 때는 가상화를 활성화하세요.
```xml
<DataGrid VirtualizingPanel.IsVirtualizing="True" VirtualizingPanel.VirtualizationMode="Recycling" />
```

## 상세/등록/수정/삭제 with 낙관적 업데이트

```csharp
public partial class MainViewModel : ObservableObject
{
    [ObservableProperty]
    private ProductDto? _selectedProduct;

    [RelayCommand]
    private async Task CreateAsync()
    {
        var draft = new ProductDto(Guid.NewGuid(), "새 제품", 0m, null, DateTimeOffset.UtcNow);
        IsBusy = true;
        Error = null;
        try
        {
            var result = await _api.CreateAsync(draft, CancellationToken.None);
            if (!result.Ok)
            {
                Error = result.Error;
                return;
            }
            Items.Insert(0, result.Value);
            Total++;
        }
        catch (Exception ex)
        {
            Error = ex.Message;
        }
        finally
        {
            IsBusy = false;
        }
    }

    [RelayCommand(CanExecute = nameof(CanEdit))]
    private async Task SaveAsync()
    {
        if (SelectedProduct is null) return;
        IsBusy = true;
        Error = null;
        try
        {
            // 낙관적 업데이트: UI를 먼저 반영 (실패 시 롤백)
            var index = Items.IndexOf(SelectedProduct);
            var original = Items[index];

            var updated = SelectedProduct with { UpdatedAt = DateTimeOffset.UtcNow };
            Items[index] = updated;

            var result = await _api.UpdateAsync(updated.Id, updated, ifMatch: null, CancellationToken.None);
            if (!result.Ok)
            {
                Error = result.Error;
                Items[index] = original; // 롤백
                return;
            }
            Items[index] = result.Value;
        }
        catch (Exception ex)
        {
            Error = ex.Message;
        }
        finally
        {
            IsBusy = false;
        }
    }

    [RelayCommand(CanExecute = nameof(CanEdit))]
    private async Task DeleteAsync()
    {
        if (SelectedProduct is null) return;
        IsBusy = true;
        Error = null;
        try
        {
            var index = Items.IndexOf(SelectedProduct);
            var itemToRemove = SelectedProduct;
            Items.RemoveAt(index); // 낙관적 삭제

            var result = await _api.DeleteAsync(itemToRemove.Id, ifMatch: null, CancellationToken.None);
            if (!result.Ok)
            {
                Error = result.Error;
                Items.Insert(index, itemToRemove); // 롤백
                return;
            }
            Total--;
        }
        catch (Exception ex)
        {
            Error = ex.Message;
        }
        finally
        {
            IsBusy = false;
        }
    }

    private bool CanEdit() => SelectedProduct is not null && !IsBusy;
}
```

실제 운영 환경에서는 **ETag/If-Match**를 사용한 동시성 제어를 적용하세요. 서버에서 ETag를 제공하고, 뷰모델에 저장한 후 수정/삭제 시 해당 값을 헤더로 반환해야 합니다.

## 고급 UX 기능 구현

### 디바운스 검색
TextBox의 `TextChanged` 이벤트에서 연속 입력을 300ms 후에 처리하도록 구현:
```csharp
private CancellationTokenSource? _searchCancellationTokenSource;

partial void OnQueryChanged(string? value)
{
    _searchCancellationTokenSource?.Cancel();
    _searchCancellationTokenSource?.Dispose();
    _searchCancellationTokenSource = new CancellationTokenSource();
    var token = _searchCancellationTokenSource.Token;
    
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

### 무한 스크롤
`ScrollViewer.ScrollChanged` 이벤트에서 스크롤이 끝에 근접하면 다음 페이지를 로드:
```csharp
// XAML: ScrollViewer.CanContentScroll="True" ScrollChanged="OnScrollChanged"
private async void OnScrollChanged(object sender, ScrollChangedEventArgs e)
{
    if (e.VerticalOffset + e.ViewportHeight >= e.ExtentHeight - 48)
        await (DataContext as MainViewModel).NextPageAsync();
}
```

### 다운로드/업로드 진행률
`HttpClient` 전송 시 `ProgressMessageHandler`를 사용하거나 스트림 복사를 통해 진행률을 추적할 수 있습니다.

## 에러 처리 및 표시 패턴
*   **서버 오류 (4xx/5xx)**: `Result.Fail(status, message)`로 ViewModel에 전달
*   **네트워크 오류**: Polly 재시도 후 실패 시 사용자 친화적인 메시지 표시
*   **작업 취소**: `OperationCanceledException`은 무시
*   **UI 표시**: 상단 또는 하단에 Alert Bar 또는 Toast 메시지로 표시

공통 에러 메시지 포맷터와 로거를 도입하여 일관된 에러 처리를 유지하세요.

## 인증 흐름 관리
*   `AuthHeaderHandler`에서 `401` 상태 코드 감지 시 Refresh 흐름 트리거
*   Refresh 성공 시 원래 요청 재시도
*   Refresh 실패 시 로그아웃 또는 재인증 UI로 전환

토큰 저장/로드:
*   Windows DPAPI(`ProtectedData`) 또는 OS 보안 저장소 활용
*   애플리케이션 시작 시 `TokenProvider`가 메모리에 로드

## 오프라인/작업 큐잉 (선택 사항)
*   요청(Create/Update/Delete)을 명령 큐에 저장
*   네트워크 연결 상태 확인
*   온라인 상태로 전환 시 큐 처리
*   실패 또는 충돌 발생 시 사용자에게 알리고 수동 해결 옵션 제공

간단한 큐 인터페이스 예시:
```csharp
public interface IOutbox
{
    Task EnqueueAsync(HttpRequestMessage request, CancellationToken ct);
    Task FlushAsync(CancellationToken ct);
}
```

## 로깅 및 추적
*   `HttpClientFactory` 로깅 (핸들러에 `ILogger` 주입)
*   ViewModel에서 핵심 상태 변화 로깅
*   에러 및 성능 이벤트 수집 (진단 및 분석용)

## 테스트 전략

### HttpMessageHandler 모의 객체를 사용한 API 테스트
```csharp
// Shop.Tests/HttpTestHandler.cs
public sealed class HttpTestHandler : HttpMessageHandler
{
    public Func<HttpRequestMessage, HttpResponseMessage>? Responder { get; set; }
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
        => Task.FromResult(Responder?.Invoke(request)
           ?? new HttpResponseMessage(System.Net.HttpStatusCode.NotFound));
}

// Shop.Tests/ProductsApiTests.cs
[Fact]
public async Task GetAsync_Returns_List_And_Total()
{
    var handler = new HttpTestHandler
    {
        Responder = request =>
        {
            var response = new HttpResponseMessage(HttpStatusCode.OK);
            response.Content = JsonContent.Create(new[]
            {
                new ProductDto(Guid.NewGuid(), "제품 A", 10m, null, DateTimeOffset.UtcNow)
            });
            response.Headers.Add("X-Total-Count", "42");
            return response;
        }
    };
    var httpClient = new HttpClient(handler) { BaseAddress = new Uri("https://fake-api/") };
    var api = new ProductsApi(httpClient);

    var result = await api.GetAsync(1, 20, null, null, CancellationToken.None);

    result.Ok.Should().BeTrue();
    result.Value.total.Should().Be(42);
    result.Value.items.Should().HaveCount(1);
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
               (new List<ProductDto>{ new(Guid.NewGuid(), "테스트 제품", 5m, null, DateTimeOffset.UtcNow) }, 10)));

    var viewModel = new MainViewModel(mockApi);
    await viewModel.LoadAsync();

    viewModel.Items.Should().HaveCount(1);
    viewModel.Total.Should().Be(10);
    viewModel.Error.Should().BeNull();
}
```

## 성능 및 안정성 고려사항
*   **비동기 프로그래밍 규칙**: 모든 I/O 작업은 `async/await` 사용, UI 업데이트는 Dispatcher 통해 실행
*   **취소 토큰 전파**: 시간이 오래 걸리는 호출마다 `CancellationToken` 지원
*   **UI 가상화**: 리스트/그리드 컨트롤에 Virtualization 활성화
*   **적절한 페이지 크기**: 20~50개 항목이 적정, 서버 측 정렬/필터링 활용
*   **Polly 정책 설정**: 과도한 재시도 방지 (백오프 적용), 429(Too Many Requests) 응답 처리
*   **메모리 관리**: 이미지/대용량 페이로드는 스트리밍 방식으로 처리 고려
*   **예외 처리**: 예외가 무음 처리되지 않도록 로깅 및 사용자 친화적 메시지 제공

## 부가 기능: 다크모드/접근성/국제화
*   **다크모드**: 리소스 토큰화 + `DynamicResource` 사용 (테마 팔레트 교체)
*   **접근성**: 키보드 탐색, 스크린리더 친화적 텍스트, `AutomationProperties` 설정
*   **국제화**: 서버/클라이언트 모두 UTC 시간 저장, 표시 시 지역화 및 문자열 리소스 활용

---

## 결론: 실전 적용을 위한 핵심 원칙

이 구현은 WPF 애플리케이션에서 MVVM 패턴을 준수하면서 RESTful API와 통신하는 견고한 클라이언트 아키텍처를 제공합니다. 핵심은 다음과 같습니다:

1.  **관심사 분리**: 네트워킹, 비즈니스 로직, UI 렌더링을 명확한 계층으로 분리하여 유지보수성을 높입니다.
2.  **회복력 있는 통신**: `HttpClientFactory`와 `Polly`를 통해 일시적 네트워크 오류, 타임아웃, 서버 과부하에 대한 표준화된 대응 체계를 구축합니다.
3.  **반응형 UI 상태 관리**: `IsBusy`, `Error`와 같은 중앙 집중식 상태 속성을 정의하고 이를 기준으로 UI(로딩 표시기, 에러 메시지, 컨트롤 비활성화)가 반응하도록 설계하여 사용자 경험을 개선합니다.
4.  **테스트 가능성**: 의존성 주입을 철저히 적용하고 외부 의존성(특히 `HttpClient`)을 인터페이스 뒤에 숨겨 단위 테스트를 통한 코드 신뢰도를 보장합니다.
5.  **현대적 .NET 생태계 활용**: WPF 프로젝트에서도 `.NET 6/7/8`을 타겟팅하고, `Generic Host`, `CommunityToolkit.Mvvm` 같은 현대적 라이브러리를 활용하여 생산성을 높입니다.

이 구조는 단순한 데이터 조회를 넘어 **페이징, 검색, 정렬, 낙관적 업데이트, 오류 처리, 작업 취소** 등 현실적인 요구사항을 아우르는 확장 가능한 기반을 마련합니다.