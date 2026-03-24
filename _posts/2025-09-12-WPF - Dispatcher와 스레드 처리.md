---
layout: post
title: WPF - Dispatcher와 스레드 처리
date: 2025-09-12 14:25:23 +0900
category: WPF
---
# WPF Dispatcher와 스레드 처리 완전 가이드

WPF의 스레딩 모델과 Dispatcher는 응답성 높은 애플리케이션을 개발하는 데 필수적인 개념입니다. 이 글에서는 UI 멈춤 없이 안전하고 효율적으로 백그라운드 작업을 처리하는 방법을 단계적으로 설명합니다.

## WPF 스레딩 모델의 기본 원리

WPF는 단일 스레드 UI 모델을 따릅니다. 즉, 모든 UI 요소(DispatcherObject/DependencyObject)는 생성된 스레드(일반적으로 메인 UI 스레드)에서만 접근하고 수정할 수 있습니다. 각 UI 스레드는 자체 Dispatcher를 가지며, 이를 통해 작업의 우선순위를 관리하고 실행합니다. 다른 스레드에서 UI 요소를 직접 조작하려고 하면 “The calling thread cannot access this object because a different thread owns it” 예외가 발생합니다. 이러한 제약을 해결하기 위해 Dispatcher를 사용하여 작업을 UI 스레드로 마샬링해야 합니다.

## Dispatcher의 주요 메서드와 우선순위

### 기본 메서드

- **CheckAccess() / VerifyAccess()**: 현재 스레드가 UI 요소에 접근할 수 있는지 확인합니다.
- **Invoke()**: 작업을 동기적으로 실행합니다. 호출 스레드는 작업이 완료될 때까지 대기합니다.
- **BeginInvoke()**: 작업을 비동기적으로 큐에 추가합니다. 호출 스레드는 즉시 반환됩니다.
- **InvokeAsync()**: Task 기반 비동기 작업을 실행합니다. `await`와 함께 사용할 수 있습니다.

### 작업 우선순위

Dispatcher는 작업을 다양한 우선순위로 실행할 수 있습니다. 주요 우선순위는 다음과 같습니다.

| 우선순위 | 설명 |
|----------|------|
| `Background` | 낮은 우선순위, 백그라운드 작업에 적합 |
| `Input` | 입력 처리 전에 실행 |
| `Loaded` | 레이아웃 로드 후 실행 |
| `Render` | 렌더링 직전 실행 |
| `Normal` | 일반적인 작업 (기본값) |
| `Send` | 가장 높은 우선순위, 즉시 실행 |

일반적으로 `Background`나 `Normal` 우선순위를 많이 사용하며, UI 응답성을 위해 긴 작업은 낮은 우선순위로 실행하는 것이 좋습니다.

## 다양한 호출 방식 비교

```csharp
// 현재 스레드가 UI 스레드인지 확인
if (Dispatcher.CheckAccess())
{
    MyTextBox.Text = "Hello";
}
else
{
    // 1. 동기 호출 - 호출 스레드 대기
    Dispatcher.Invoke(() => MyTextBox.Text = "Hello");
    
    // 2. 비동기 호출 - 즉시 반환
    Dispatcher.BeginInvoke(() => MyTextBox.Text = "Hello");
    
    // 3. Task 기반 비동기 호출 - await 가능
    await Dispatcher.InvokeAsync(() => MyTextBox.Text = "Hello");
}
```

**권장사항**: UI 업데이트가 즉시 필요하지 않다면 `BeginInvoke()`나 `InvokeAsync()`를 사용하세요. 동기적인 `Invoke()`는 교착 상태나 UI 정지를 초래할 수 있습니다.

## async/await와 동기화 컨텍스트

WPF는 UI 스레드를 위한 `SynchronizationContext`를 제공합니다. 기본적으로 `await` 이후의 코드는 원래의 동기화 컨텍스트(UI 스레드)에서 실행되므로 UI 업데이트가 쉽습니다.

```csharp
public async Task LoadDataAsync()
{
    IsLoading = true;
    try
    {
        var data = await dataService.GetDataAsync(); // UI 컨텍스트 유지
        Items.Clear();
        foreach (var item in data) Items.Add(item); // UI 스레드에서 안전
    }
    finally
    {
        IsLoading = false;
    }
}
```

반면, 서비스나 라이브러리 계층에서는 `ConfigureAwait(false)`를 사용하여 불필요한 컨텍스트 캡처를 방지합니다.

```csharp
public async Task<Data> GetDataAsync()
{
    var response = await httpClient.GetAsync(url).ConfigureAwait(false);
    return await response.Content.ReadAsAsync<Data>();
}
```

**기본 규칙**: UI 계층에서는 컨텍스트 캡처를 유지하고, 백엔드 계층에서는 `ConfigureAwait(false)`를 사용하여 성능을 최적화하세요.

## CPU 집약적 작업 처리하기

무거운 CPU 연산은 UI 스레드에서 실행하면 응답성이 떨어집니다. `Task.Run()`을 사용하여 백그라운드 스레드에서 처리하세요.

```csharp
public async Task ProcessHeavyComputationAsync()
{
    StatusText = "처리 중...";
    var result = await Task.Run(() => HeavyComputationAlgorithm());
    StatusText = $"완료: {result}";
}
```

**주의**: I/O 작업(네트워크, 디스크)은 `Task.Run()`으로 감싸지 마세요. 비동기 I/O API를 직접 사용하면 스레드 점유 없이 효율적으로 처리됩니다.

## 다양한 타이머 비교

WPF에는 여러 타이머가 있습니다.

```csharp
// DispatcherTimer - UI 스레드에서 안전
var uiTimer = new DispatcherTimer(DispatcherPriority.Background)
{
    Interval = TimeSpan.FromMilliseconds(100)
};
uiTimer.Tick += (s, e) => UpdateProgress();
uiTimer.Start();

// System.Threading.Timer - 백그라운드 타이머
var bgTimer = new System.Threading.Timer(state =>
{
    Application.Current.Dispatcher.BeginInvoke(() => UpdateProgress());
}, null, 0, 100);
```

- **DispatcherTimer**: UI 업데이트에 적합하며, UI 스레드에서 실행됩니다.
- **System.Threading.Timer**: 정확한 타이밍이 필요한 백그라운드 작업에 적합합니다.

## 진행률 표시와 작업 취소

`IProgress<T>`와 `CancellationToken`을 함께 사용하면 진행률을 표시하고 작업을 취소할 수 있습니다.

```csharp
public async Task DownloadFileAsync(
    string url,
    string destination,
    IProgress<double> progress,
    CancellationToken cancellationToken)
{
    using var response = await httpClient.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, cancellationToken);
    using var stream = await response.Content.ReadAsStreamAsync(cancellationToken);
    using var fileStream = new FileStream(destination, FileMode.Create);

    var totalBytes = response.Content.Headers.ContentLength ?? 0;
    var buffer = new byte[8192];
    long totalRead = 0;

    int bytesRead;
    while ((bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length, cancellationToken)) > 0)
    {
        await fileStream.WriteAsync(buffer, 0, bytesRead, cancellationToken);
        totalRead += bytesRead;
        if (totalBytes > 0)
            progress.Report((double)totalRead / totalBytes);
    }
}
```

사용 예시:

```csharp
var progress = new Progress<double>(p => ProgressBar.Value = p * 100);
var cts = new CancellationTokenSource();

await DownloadFileAsync(url, destination, progress, cts.Token);
```

## ObservableCollection의 스레드 안전한 업데이트

여러 스레드에서 ObservableCollection을 업데이트할 때는 주의가 필요합니다.

### 전통적인 방법

```csharp
var newItems = await dataService.GetItemsAsync();

await Application.Current.Dispatcher.InvokeAsync(() =>
{
    Items.Clear();
    foreach (var item in newItems)
        Items.Add(item);
});
```

### EnableCollectionSynchronization 사용

```csharp
public class MainViewModel
{
    private readonly object _collectionLock = new object();
    public ObservableCollection<Item> Items { get; } = new ObservableCollection<Item>();

    public MainViewModel()
    {
        BindingOperations.EnableCollectionSynchronization(Items, _collectionLock);
    }

    public void AddItemFromBackground(Item item)
    {
        lock (_collectionLock)
        {
            Items.Add(item);
        }
    }
}
```

`EnableCollectionSynchronization`은 컬렉션 수준의 락을 사용하여 바인딩 엔진이 안전하게 컬렉션에 접근할 수 있도록 합니다. 빈번한 변경은 여전히 성능에 영향을 줄 수 있으므로 배치 업데이트를 고려하세요.

## INotifyPropertyChanged와 스레드 안전성

`INotifyPropertyChanged` 이벤트는 구독 스레드(일반적으로 UI 스레드)에서 처리될 것으로 기대됩니다. 따라서 이벤트 발생 시 UI 스레드로 마샬링하는 것이 안전합니다.

```csharp
protected virtual void OnPropertyChanged([CallerMemberName] string? propertyName = null)
{
    var handler = PropertyChanged;
    if (handler != null)
    {
        if (Application.Current.Dispatcher.CheckAccess())
            handler(this, new PropertyChangedEventArgs(propertyName));
        else
            Application.Current.Dispatcher.BeginInvoke(() =>
                handler(this, new PropertyChangedEventArgs(propertyName)));
    }
}
```

## 이미지와 Freezable 객체 처리

Freezable 객체(BitmapImage, Brush 등)는 `Freeze()` 메서드를 호출하여 읽기 전용으로 만들면 여러 스레드에서 안전하게 공유할 수 있습니다.

```csharp
public static BitmapImage LoadAndFreezeImage(string filePath)
{
    var bitmap = new BitmapImage();
    using (var stream = File.OpenRead(filePath))
    {
        bitmap.BeginInit();
        bitmap.CacheOption = BitmapCacheOption.OnLoad;
        bitmap.StreamSource = stream;
        bitmap.EndInit();
        bitmap.Freeze(); // 스레드 안전하게 만듦
    }
    return bitmap;
}
```

## Debounce 패턴 구현

사용자 입력에 대한 반응을 최적화하기 위해 Debounce 패턴을 구현할 수 있습니다.

```csharp
public class DebounceDispatcher
{
    private readonly Dispatcher _dispatcher;
    private readonly TimeSpan _delay;
    private DispatcherTimer? _timer;
    private Action? _pendingAction;

    public DebounceDispatcher(Dispatcher dispatcher, TimeSpan delay)
    {
        _dispatcher = dispatcher;
        _delay = delay;
    }

    public void Execute(Action action)
    {
        _pendingAction = action;

        _timer ??= new DispatcherTimer(_delay, DispatcherPriority.Background, OnTimerTick, _dispatcher);
        _timer.Stop();
        _timer.Interval = _delay;
        _timer.Start();
    }

    private void OnTimerTick(object? sender, EventArgs e)
    {
        _timer?.Stop();
        _pendingAction?.Invoke();
        _pendingAction = null;
    }
}
```

사용 예시:

```csharp
private readonly DebounceDispatcher _searchDebounce = new DebounceDispatcher(Dispatcher, TimeSpan.FromMilliseconds(300));

private void SearchTextBox_TextChanged(object sender, TextChangedEventArgs e)
{
    _searchDebounce.Execute(async () => await PerformSearchAsync());
}
```

## 일반적인 문제와 해결 방법

| 문제 | 원인 | 해결 |
|------|------|------|
| “The calling thread cannot access this object” | 다른 스레드에서 UI 요소 직접 접근 | `Dispatcher.BeginInvoke()` 또는 `InvokeAsync()` 사용 |
| UI 응답성 저하 | UI 스레드에서 무거운 작업 실행 | `Task.Run()`으로 백그라운드 이동, 가상화 사용 |
| ObservableCollection 업데이트 예외 | 백그라운드 스레드에서 직접 수정 | `EnableCollectionSynchronization` 또는 Dispatcher로 마샬링 |
| 교착 상태(Deadlock) | UI 스레드에서 `Task.Result` / `Task.Wait()` 사용 | 항상 `await` 사용, 동기 대기 피하기 |

## 성능 최적화 팁

- **UI 스레드 가볍게 유지**: CPU 집약적 작업은 백그라운드로 이동
- **비동기 I/O 활용**: 네트워크/디스크 작업은 비동기 API 직접 사용
- **배치 업데이트**: 여러 UI 변경을 하나의 Dispatcher 호출로 그룹화
- **가상화 사용**: 대용량 목록은 UI 가상화 적용
- **Freezable 활용**: 재사용 가능한 리소스는 `Freeze()`하여 성능 향상
- **적절한 우선순위**: 중요도에 따라 `DispatcherPriority` 조정

## 결론

WPF의 Dispatcher와 스레드 처리는 응답성 높은 애플리케이션 개발의 핵심입니다. 기본 원칙을 이해하고 적절한 패턴을 적용하면 복잡한 작업에서도 부드러운 사용자 경험을 제공할 수 있습니다.

- UI 스레드는 가볍게 유지하고, 무거운 작업은 백그라운드로 이동
- UI 요소 접근은 항상 Dispatcher를 통해 마샬링
- 비동기 프로그래밍 패턴을 일관되게 적용
- 성능 최적화 기법들을 상황에 맞게 활용

이러한 원칙들을 준수하면 대용량 데이터 처리, 복잡한 계산, 네트워크 통신 등 다양한 시나리오에서도 안정적이고 반응성이 뛰어난 WPF 애플리케이션을 개발할 수 있습니다.