---
layout: post
title: WPF - Dispatcher와 스레드 처리
date: 2025-09-12 14:25:23 +0900
category: WPF
---
# 🧵 WPF **Dispatcher와 스레드 처리** 완전 정복

*(WPF 스레딩 모델 → Dispatcher/우선순위 → `Invoke/BeginInvoke`/`InvokeAsync` → `async/await` 모범 패턴 → `DispatcherTimer` vs `Timer` → 컬렉션 동기화/`EnableCollectionSynchronization` → 진행률/취소 → 백그라운드 이미지 디코딩/Freezable → 다중 UI 스레드(보조 Dispatcher) → 오류/교착 방지 → 퍼포먼스 체크리스트까지)*

> 이 글은 실무에서 “**UI 멈춤 없이** 빠르고 안전한” WPF 앱을 작성하는 데 필요한 **스레드/Dispatcher 지식**을
> 예제와 함께 **누락 없이** 정리했습니다. .NET 6~8 WPF 기준(4.5+도 대부분 동일)입니다.

---

## 큰 그림: WPF 스레딩 모델

- **UI 요소**(`DispatcherObject`/`DependencyObject`)는 **자신이 생성된 스레드**(보통 **메인 UI 스레드**)에서만 접근/변경할 수 있다.
- 각 UI 스레드는 **하나의 `Dispatcher`**를 갖고, **메시지 루프**(priority 큐)를 돌면서 작업을 처리한다.
- 다른 스레드에서 UI를 건드리면:
  > `InvalidOperationException: The calling thread cannot access this object because a different thread owns it.`
- 해결: UI 접근은 항상 `Dispatcher`를 통해 **마샬링(marshalling)** 해야 한다.

---

## Dispatcher 핵심 API & 우선순위

### 대표 메서드

- `Dispatcher.CheckAccess()` / `VerifyAccess()`
- `Dispatcher.Invoke(Action, DispatcherPriority)` — **동기** 실행(호출 스레드가 기다림)
- `Dispatcher.BeginInvoke(Action, DispatcherPriority)` — **비동기** 큐잉(호출 스레드는 즉시 반환)
- `Dispatcher.InvokeAsync(Func<Task>, DispatcherPriority)` — Task 기반 비동기
- `Dispatcher.Yield(DispatcherPriority)` — 현재 await 체인에서 **UI를 잠깐 양보**

### 우선순위(일부)

`Inactive < SystemIdle < ApplicationIdle < ContextIdle < Background < Input < Loaded < Render < DataBind < Normal < Send`

> 보통 **`DispatcherPriority.Background/Normal/Send`** 를 주로 사용.
> - Input/Render보다 낮은 우선순위로 길게 돌면 **UI 렌더/입력 지연**이 발생한다.

---

## Invoke vs BeginInvoke vs InvokeAsync

```csharp
// UI 스레드 여부 확인
if (Dispatcher.CheckAccess())
{
    // UI 접근 OK
    MyText.Text = "Hello";
}
else
{
    // 1) 동기: 호출 스레드는 기다림 (주의: 교착 가능!)
    Dispatcher.Invoke(() => MyText.Text = "Hello", DispatcherPriority.Normal);

    // 2) 비동기: 바로 반환, UI 큐에 등록
    Dispatcher.BeginInvoke(() => MyText.Text = "Hello", DispatcherPriority.Background);

    // 3) Task 기반 비동기: await 가능
    await Dispatcher.InvokeAsync(() => MyText.Text = "Hello", DispatcherPriority.Normal);
}
```

**권장**
- UI 갱신이 **즉시** 필요하지 않다면 `BeginInvoke`/`InvokeAsync` 선호.
- 동기 `Invoke` 남용은 **교착**/프레임 스톨을 부른다(아래 11절 참조).

---

## `async/await`와 SynchronizationContext

- WPF는 UI 스레드를 위한 **`SynchronizationContext`** 를 설치한다.
- **기본**: `await` 후 **캡처된 컨텍스트(UI)** 로 **복귀** → UI 업데이트 쉬움.
- 라이브러리/백엔드 코드에서 **UI 복귀 불필요**하면 `ConfigureAwait(false)`로 컨텍스트 캡처 방지(성능/교착 방지).

```csharp
// ViewModel 등 UI 계층: 보통 캡처 허용 (기본값)
public async Task LoadAsync()
{
    IsBusy = true;
    try
    {
        var data = await _service.GetAsync(); // UI 컨텍스트 캡처됨
        Items.ReplaceWith(data);              // UI 스레드에서 안전
    }
    finally { IsBusy = false; }
}

// 라이브러리 계층: 되도록 컨텍스트 캡처 금지
public async Task<Data> GetAsync()
{
    using var res = await _http.GetAsync(url, ct).ConfigureAwait(false);
    // … 처리
    return data;
}
```

> **규칙**: **UI 계층**은 캡처 유지(편의), **서비스/라이브러리**는 `ConfigureAwait(false)`(성능/안전).

---

## CPU 작업/IO를 백그라운드로: `Task.Run` + Dispatcher 마샬링

```csharp
// 무거운 CPU 연산을 UI 스레드에서 절대 돌리지 말 것!
var result = await Task.Run(() => HeavyCpuWork(input), ct);
// UI 갱신은 UI 스레드로
MyLabel.Content = result;
```

> **IO(네트워크/디스크)** 는 `Task.Run` 불필요(비동기 API 쓰면 자동으로 스레드 점유 없음).
> CPU 바운드는 `Task.Run`/Parallel 사용.

---

## `DispatcherTimer` vs `System.Timers.Timer` vs `System.Threading.Timer`

- **`DispatcherTimer`**
  - **UI Dispatcher 큐**에서 **지정 우선순위**로 틱 이벤트 발생 → **UI 접근 안전**
  - 렌더/입력이 바쁘면 늦어질 수 있음(프레임 친화)
- **`System.Timers.Timer` / `Threading.Timer`**
  - **스레드풀**에서 콜백 → UI 접근 시 **반드시 Dispatcher로 마샬링** 필요
  - 고정밀/백그라운드 타이밍은 유리

```csharp
// UI 우선 인터벌(프로그레스/애니메이션 보조 등): DispatcherTimer
var dt = new DispatcherTimer(DispatcherPriority.Background)
{
    Interval = TimeSpan.FromMilliseconds(100)
};
dt.Tick += (_,__) => ProgressValue++;
dt.Start();

// 백그라운드 타이머: Threading.Timer
var t = new System.Threading.Timer(_ =>
{
    // UI 접근 금지! 필요하면 Dispatcher로
    Dispatcher.BeginInvoke(() => ProgressValue++);
}, null, 0, 100);
```

---

## 진행률/취소와 UI 갱신 (`IProgress<T>`, `CancellationToken`)

```csharp
public async Task DownloadAsync(IProgress<double> progress, CancellationToken ct)
{
    var total = 1_000_000;
    var read = 0;
    while (read < total)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(10, ct); // IO 시뮬레이션
        read += 16_384;
        progress.Report((double)read / total);
    }
}

// 사용: IProgress는 기본적으로 UI 컨텍스트에서 콜백 실행
var progress = new Progress<double>(p => ProgressBar.Value = p * 100);
var cts = new CancellationTokenSource();
await DownloadAsync(progress, cts.Token);
```

> **팁**: ViewModel에서 진행률 값(예: `double Progress`)을 바인딩해 UI에서 표시.

---

## ObservableCollection 업데이트(스레드 안전)

### 전통적 방법: Dispatcher로 래핑

```csharp
// 백그라운드 스레드
var items = await FetchAsync();
await Dispatcher.InvokeAsync(() =>
{
    MyCollection.Clear();
    foreach (var it in items) MyCollection.Add(it);
});
```

### 다중 스레드에서 동시 업데이트: `BindingOperations.EnableCollectionSynchronization`

```csharp
readonly object _lock = new();
ObservableCollection<Item> Items = new();

public MainWindow()
{
    InitializeComponent();
    BindingOperations.EnableCollectionSynchronization(Items, _lock);
}

// 아무 스레드에서나 안전하게 Add 가능
Task.Run(() =>
{
    for (int i=0;i<1000;i++)
    {
        lock (_lock) Items.Add(new Item{ Id=i });
        Thread.Sleep(1);
    }
});
```

> 이 API는 **컬렉션 수준의 락**을 이용해 **바인딩 엔진**이 안전하게 열람하도록 한다.
> 단, **빈번한 요소 변경**은 여전히 UI 負 → **배치 업데이트**(+ 가상화) 권장.

---

## DataBinding/PropertyChanged를 **UI 스레드**에서

- `INotifyPropertyChanged.PropertyChanged` 는 **구독 스레드(UI)** 에서 처리되길 기대한다.
- 백그라운드에서 모델 속성을 바꾸면, **UI로 이벤트 마샬링**해야 안전.

```csharp
// VM 내부 예시
void Set<T>(ref T field, T value, [CallerMemberName] string? name=null)
{
    if (Equals(field, value)) return;
    field = value;

    if (Application.Current.Dispatcher.CheckAccess())
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
    else
        Application.Current.Dispatcher.BeginInvoke(
            () => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name)));
}
```

---

## 이미지/미디어: 백그라운드 디코딩 + `Freezable.Freeze`

```csharp
static BitmapImage LoadFrozenBitmap(string path)
{
    var bi = new BitmapImage();
    using var fs = File.OpenRead(path);
    bi.BeginInit();
    bi.CacheOption = BitmapCacheOption.OnLoad; // 스트림 닫아도 이미지 유지
    bi.StreamSource = fs;
    bi.EndInit();
    bi.Freeze(); // 핵심: Freeze → 스레드 세이프 & 렌더 최적화
    return bi;
}

var bmp = await Task.Run(() => LoadFrozenBitmap("c:\\img\\hero.png"));
MyImage.Source = bmp; // UI 스레드에서 안전하게 할당
```

> Freezable(Brush/Geometry/Bitmap…)은 **Frozen 상태**일 때 **스레드 경계**를 넘어 **읽기 공유** 가능.

---

## Debounce/Throttle: Dispatcher로 UI 스파이크 완화

### Debounce (마지막 입력 후 N ms 뒤 1회 실행)

```csharp
public sealed class DispatcherDebounce
{
    readonly Dispatcher _dispatcher;
    readonly TimeSpan _delay;
    DispatcherTimer? _timer;
    Action? _action;
    public DispatcherDebounce(Dispatcher dispatcher, TimeSpan delay)
        => (_dispatcher, _delay) = (dispatcher, delay);

    public void Run(Action action)
    {
        _action = action;
        _timer ??= new DispatcherTimer(_delay, DispatcherPriority.Background, OnTick, _dispatcher);
        _timer.Stop();
        _timer.Interval = _delay;
        _timer.Start();
    }
    void OnTick(object? s, EventArgs e)
    {
        _timer?.Stop();
        _action?.Invoke();
    }
}

// 사용: 검색 박스 텍스트 변경 시
private DispatcherDebounce _debounce;
public MainWindow()
{
    InitializeComponent();
    _debounce = new DispatcherDebounce(Dispatcher, TimeSpan.FromMilliseconds(300));
}
private void SearchTextChanged(object s, TextChangedEventArgs e)
{
    _debounce.Run(async () => await ViewModel.SearchAsync());
}
```

### Throttle (최대 N ms마다 1회)

유사하게 `DispatcherTimer.IsEnabled` 체크로 구현.

---

## 회피 패턴

**문제 시나리오**
- UI 스레드가 **동기 `Invoke`** 로 백그라운드 작업 결과를 기다림
- 백그라운드 작업이 다시 UI 스레드로 `Invoke` 하려 함 → **서로 대기** → 교착

**해결 규칙**
- UI → 백그라운드: **항상 비동기 `await`** 로 결과를 받는다.
- 백그라운드 → UI: `BeginInvoke`/`InvokeAsync` 로 비동기 큐잉한다.
- `Result`/`Wait()` 사용 금지(특히 UI 스레드에서).

```csharp
// ❌ 나쁜 예 (UI 스레드에서 동기 대기)
var data = GetAsync().Result; // 교착 위험

// ✅ 좋은 예
var data = await GetAsync();  // 안전
```

---

## `DispatcherUnhandledException` & Task 예외

```csharp
// UI 스레드에서 던져진 예외(Dispatcher 파이프라인) 잡기
Application.DispatcherUnhandledException += (s, e) =>
{
    Log(e.Exception);
    e.Handled = true; // 앱 크래시 방지 (적절히 판단)
};

// 백그라운드 Task 예외(관찰되지 않은)
TaskScheduler.UnobservedTaskException += (s, e) =>
{
    Log(e.Exception);
    e.SetObserved();
};
```

> **주의**: 모든 예외를 삼키면 **이상 상태 은폐**. 기록/알림 후 **안전 종료**나 복구 설계를.

---

## 만들기

- WPF 창은 **여러 UI 스레드**로 분산 가능(고급 시나리오).
- 보조 UI 스레드에서 `Window`를 만들고 `Dispatcher.Run()`으로 메시지 루프 시작.

```csharp
Thread _uiThread;
Dispatcher? _subDispatcher;

void StartSecondaryUi()
{
    _uiThread = new Thread(() =>
    {
        // STA 필수 (WPF)
        Thread.CurrentThread.SetApartmentState(ApartmentState.STA);

        // 새 Dispatcher는 자동 생성
        var win = new ToolWindow(); // 보조 창
        _subDispatcher = Dispatcher.CurrentDispatcher;
        win.Show();

        // 메시지 루프
        Dispatcher.Run();
    });
    _uiThread.IsBackground = true;
    _uiThread.Start();
}

void StopSecondaryUi()
{
    _subDispatcher?.BeginInvokeShutdown(DispatcherPriority.Normal);
    _uiThread.Join();
}

// 메인에서 보조 UI에 작업 보내기
_subDispatcher?.BeginInvoke(() => _someControlOnToolWindow.Text = "Hello");
```

> 장점: 무거운 UI(예: 실시간 그래프)를 분리해 메인 UI 지연 감소.
> 단점: 교차 UI 호출 관리가 복잡 → 대부분은 **한 개 UI 스레드 + 백그라운드 작업**으로 충분.

---

## `DispatcherFrame`와 DoEvents 유사 패턴(주의)

- `DispatcherFrame`으로 **임시 메시지 루프**를 돌려 UI를 잠깐 갱신할 수 있다(모달 대기 등).
- 남용 시 **reentrancy(재진입)** 문제로 복잡한 버그 유발 → 되도록 **`await`/대기 화면** 사용.

```csharp
// 예: 매우 제한적으로
var frame = new DispatcherFrame();
Dispatcher.BeginInvoke(new Action(() => frame.Continue = false), DispatcherPriority.Background);
Dispatcher.PushFrame(frame); // 잠깐 메시지 펌프
```

---

## 컬렉션 뷰/필터/정렬 업데이트 성능

- `CollectionViewSource.View` 갱신은 UI 스레드에서 무겁다 → **`DeferRefresh()`** 로 배치
```csharp
using (view.DeferRefresh())
{
    view.Filter = item => /* … */
    view.SortDescriptions.Clear();
    view.SortDescriptions.Add(new SortDescription("Name", ListSortDirection.Ascending));
}
```

---

## 실제 종합 예제 ① — “대용량 파일 해시” (CPU 바운드)

**요구**
- 파일 여러 개 해시 → UI 진행률/취소/결과 표시
- UI 멈춤 금지

```csharp
public async Task ComputeHashesAsync(IEnumerable<string> files, IProgress<(int done, int total)> progress, CancellationToken ct)
{
    int done = 0; int total = files.Count();
    foreach (var path in files)
    {
        ct.ThrowIfCancellationRequested();
        var hash = await Task.Run(() => ComputeSha256(path), ct); // CPU
        await Dispatcher.InvokeAsync(() => Results.Add(new ResultItem(path, hash)));
        progress.Report((++done, total));
    }
}

// 사용
var progress = new Progress<(int done, int total)>(p =>
{
    ProgressBar.Value = (double)p.done / p.total * 100;
});
var cts = new CancellationTokenSource();
await ComputeHashesAsync(FileList, progress, cts.Token);
```

---

## 실제 종합 예제 ② — “검색 디바운스 + 페이징 API” (IO 바운드)

```csharp
private readonly DispatcherDebounce _debounce;
public MainViewModel()
{
    _debounce = new DispatcherDebounce(Application.Current.Dispatcher, TimeSpan.FromMilliseconds(300));
}

private string? _query;
public string? Query
{
    get => _query;
    set
    {
        if (_query == value) return;
        _query = value;
        // 입력 후 300ms 지난 뒤 검색
        _debounce.Run(async () => await SearchAsync());
    }
}

public async Task SearchAsync()
{
    if (string.IsNullOrWhiteSpace(Query)) { Items.Clear(); return; }
    IsBusy = true; Error = null;
    try
    {
        var result = await _api.SearchAsync(Query!); // IO, ConfigureAwait(false) 내부 사용
        Items.ReplaceWith(result);                   // UI 컨텍스트
    }
    catch (Exception ex) { Error = ex.Message; }
    finally { IsBusy = false; }
}
```

---

## 성능/안정성 체크리스트

- [ ] **UI 스레드**에서 **무거운 작업 금지**(CPU/대량 LINQ/대용량 변환 등)
- [ ] IO는 **순수 비동기**(await) + 라이브러리는 `ConfigureAwait(false)`
- [ ] UI 갱신은 `BeginInvoke`/`InvokeAsync` 로 **비동기 마샬링**
- [ ] `ObservableCollection` 대량 변경은 **배치**(Clear+AddRange) & **가상화**
- [ ] 정기 작업은 `DispatcherTimer`, 고정밀/백그라운드는 `Threading.Timer`
- [ ] `EnableCollectionSynchronization` 로 다중 스레드 컬렉션 접근 보호
- [ ] 이미지/브러시/지오메트리 등 `Freezable`은 **Freeze** 해서 스레드/렌더 최적화
- [ ] `DispatcherUnhandledException`/`TaskScheduler.UnobservedTaskException` 로깅
- [ ] `Invoke`(동기) 남용 금지 — 교착 주의, 가능하면 `await` 기반
- [ ] UI 반응성: `Dispatcher.Yield()` 로 장시간 루프 분절, 또는 **chunking**

---

## 흔한 오류 & 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `The calling thread cannot access this object…` | 다른 스레드에서 UI 접근 | `Dispatcher.BeginInvoke`/`InvokeAsync` 사용 |
| 스크롤/입력 버벅임 | UI 스레드에서 무거운 작업 | `Task.Run`/비동기화 + 가상화/배치 |
| 랜덤 크래시/상태 꼬임 | 컬렉션을 백그라운드에서 직접 변경 | `EnableCollectionSynchronization` 또는 UI로 마샬링 |
| 교착(응답 없음) | UI에서 `Result/Wait()` | 전부 `await`, 동기 대기 금지 |
| 타이머 틱이 끊김 | DispatcherTimer 우선순위/부하 | 우선순위 조정 또는 Threading.Timer 사용 |
| 이미지 할당 시 예외 | Freezable Freeze 미적용 | 백그라운드에서 `BitmapImage.Freeze()` 후 전달 |

---

## “붙여 넣어 바로 쓰는” 스니펫

### UI 스레드 안전 호출 헬퍼

```csharp
public static class DispatcherEx
{
    public static void SafeInvoke(this Dispatcher d, Action action, DispatcherPriority p = DispatcherPriority.Normal)
    {
        if (d.CheckAccess()) action();
        else d.Invoke(action, p);
    }
    public static Task SafeInvokeAsync(this Dispatcher d, Action action, DispatcherPriority p = DispatcherPriority.Normal)
        => d.CheckAccess() ? Task.Run(action) : d.InvokeAsync(action, p).Task;
}
```

### UI 대량 갱신 배치(간단 AddRange)

```csharp
public static class ObservableCollectionExtensions
{
    public static void ReplaceWith<T>(this ObservableCollection<T> coll, IEnumerable<T> items)
    {
        coll.Clear();
        foreach (var it in items) coll.Add(it);
    }
    public static void AddRange<T>(this ObservableCollection<T> coll, IEnumerable<T> items)
    {
        foreach (var it in items) coll.Add(it);
    }
}
```

### 장시간 루프 분절(프레임 양보)

```csharp
public async Task ProcessBigListAsync(IReadOnlyList<Item> list)
{
    const int chunk = 100;
    for (int i=0; i<list.Count; i+=chunk)
    {
        var slice = list.Skip(i).Take(chunk);
        await Task.Run(() => ProcessSlice(slice)); // CPU
        await Dispatcher.Yield(DispatcherPriority.Background); // 입력/렌더에 양보
    }
}
```

---

## 마무리

- **원칙**: “**UI는 가볍게, 무거운 건 밖에서**” + “**UI 접근은 Dispatcher 통해**”
- `async/await` + 올바른 컨텍스트 관리가 **교착/프리즈**를 없애고
- `DispatcherTimer`/컬렉션 동기화/`Freezable`/다중 UI 스레드 등 도구들을 **정확히** 쓰면
  복잡한 앱에서도 **부드러운 UX**를 유지할 수 있습니다.
