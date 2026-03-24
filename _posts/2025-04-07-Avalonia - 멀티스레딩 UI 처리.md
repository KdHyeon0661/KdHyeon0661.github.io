---
layout: post
title: Avalonia - 멀티스레딩 UI 처리
date: 2025-04-07 20:20:23 +0900
category: Avalonia
---
# Avalonia에서 멀티스레딩 UI 처리

Avalonia의 모든 UI 요소(`Visual`)는 **UI 스레드** 소유입니다. 다른 스레드에서 UI에 접근하면 예외나 크래시가 발생합니다. 따라서 무거운 작업은 백그라운드에서 처리하고, UI 갱신만 UI 스레드로 전달해야 합니다. 이 글에서는 초중급 개발자가 실제로 적용할 수 있는 멀티스레딩 패턴을 정리합니다.

---

## Dispatcher 기초

Avalonia의 `Dispatcher`는 UI 스레드에 작업을 큐잉하는 역할을 합니다.

```csharp
using Avalonia.Threading;

// UI 갱신 예
await Dispatcher.UIThread.InvokeAsync(() =>
{
    TitleText = "완료";
    Items.Clear();
    foreach (var x in results) Items.Add(x);
});
```

**주요 메서드**
- `InvokeAsync(Action)` / `InvokeAsync(Func<Task>)` : UI 스레드에서 실행
- `Post(Action)` : 큐잉 (약간 느슨)
- `CheckAccess()` : 이미 UI 스레드인지 확인

```csharp
if (Dispatcher.UIThread.CheckAccess()) UpdateUI();
else await Dispatcher.UIThread.InvokeAsync(UpdateUI);
```

---

## async/await + 백그라운드 작업 기본 패턴

CPU 바운드 작업은 `Task.Run`으로 백그라운드로 보내고, 결과만 UI에 바인딩합니다.

```csharp
public async Task LoadAsync(CancellationToken ct)
{
    IsLoading = true;
    var result = await Task.Run(() => Repository.LoadHeavy(), ct);

    await Dispatcher.UIThread.InvokeAsync(() =>
    {
        Items.Clear();
        foreach (var r in result) Items.Add(r);
        IsLoading = false;
    });
}
```

> **주의**: 라이브러리 코드에서 `ConfigureAwait(false)`를 사용할 수 있지만, UI 갱신 직전에는 반드시 UI 스레드로 복귀해야 합니다.

---

## ReactiveUI 스케줄러와 함께 사용

ReactiveUI를 사용하면 `RxApp.MainThreadScheduler`로 UI 스레드로의 전환을 쉽게 처리할 수 있습니다.

```csharp
this.WhenAnyValue(x => x.Query)
    .Throttle(TimeSpan.FromMilliseconds(400))
    .Select(q => Observable.FromAsync(() => Api.SearchAsync(q)))
    .Switch()
    .ObserveOn(RxApp.MainThreadScheduler)  // ← UI 스레드
    .Subscribe(results =>
    {
        Results.Clear();
        foreach (var r in results) Results.Add(r);
    });
```

- `Throttle`: 입력 디바운스
- `Switch`: 이전 요청 취소
- `ObserveOn`: 소비 위치를 UI 스레드로 강제

---

## 타이머 종류와 선택

| 타이머 | 실행 스레드 | 용도 | 주의 |
|--------|------------|------|------|
| `DispatcherTimer` | UI 스레드 | 시계, 작은 애니메이션 | 핸들러 내 무거운 작업 금지 |
| `System.Timers.Timer` | 스레드풀 | 주기적 백그라운드 작업 | UI 접근 시 `Dispatcher` 필요 |
| `PeriodicTimer` (.NET 6+) | `await` 루프 | 명시적 주기 작업 | 취소 토큰으로 종료 가능 |

**예제**

```csharp
// UI 전용
var uiTimer = new DispatcherTimer { Interval = TimeSpan.FromSeconds(1) };
uiTimer.Tick += (_, __) => NowText = DateTime.Now.ToString("HH:mm:ss");
uiTimer.Start();

// 백그라운드
using var pt = new PeriodicTimer(TimeSpan.FromMilliseconds(250));
_ = Task.Run(async () =>
{
    while (await pt.WaitForNextTickAsync(ct))
    {
        var sample = Sensor.Read();
        await Dispatcher.UIThread.InvokeAsync(() => AppendSample(sample));
    }
}, ct);
```

---

## 대량 UI 갱신 최적화

`ObservableCollection`에 수천 개 항목을 한 번에 추가하면 Measure/Layout 폭탄이 발생합니다. 해결책:

### 스냅샷 교체

```csharp
var snapshot = await Task.Run(() => Repository.GetMany());
await Dispatcher.UIThread.InvokeAsync(() =>
{
    Items.Clear();
    foreach (var x in snapshot) Items.Add(x);
});
```

### 청크 단위 추가

```csharp
const int CHUNK = 200;
await Dispatcher.UIThread.InvokeAsync(() =>
{
    foreach (var chunk in snapshot.Chunk(CHUNK))
        foreach (var x in chunk) Items.Add(x);
});
```

### 가상화 컨트롤 사용
`ItemsRepeater`, `VirtualizingStackPanel` 등 가상화를 지원하는 컨트롤을 활용합니다.

---

## 진행률 보고와 취소

```csharp
public async Task DownloadAsync(string url, IProgress<double> progress, CancellationToken ct)
{
    using var resp = await _http.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct);
    var total = resp.Content.Headers.ContentLength ?? -1L;
    await using var s = await resp.Content.ReadAsStreamAsync(ct);

    var buf = new byte[81920];
    long read = 0;
    int n;
    while ((n = await s.ReadAsync(buf.AsMemory(0, buf.Length), ct)) > 0)
    {
        read += n;
        if (total > 0) progress.Report((double)read / total);
        // 파일에 쓰기 ...
    }
}
```

ViewModel:

```csharp
public double Progress { get => _p; set => this.RaiseAndSetIfChanged(ref _p, value); }

public async Task StartAsync()
{
    using var cts = new CancellationTokenSource();
    var pr = new Progress<double>(v => Progress = v);
    await Task.Run(() => DownloadAsync(Url, pr, cts.Token));
}
```

> `IProgress<T>`는 생성한 스레드의 동기화 컨텍스트를 캡처합니다. UI 스레드에서 생성하면 UI 스레드로 콜백이 전달됩니다.

---

## 생산자-소비자 패턴 (Channel)

실시간 스트림(웹소켓, 로그)을 안전하게 UI로 전달할 때는 `System.Threading.Channels`를 사용합니다.

```csharp
private readonly Channel<string> _logCh = Channel.CreateUnbounded<string>();

// 생산자 (백그라운드)
_ = Task.Run(async () =>
{
    await foreach (var line in SocketReader(ct))
        await _logCh.Writer.WriteAsync(line, ct);
});

// 소비자 (UI로 배치 전달)
_ = Task.Run(async () =>
{
    var batch = new List<string>(200);
    while (await _logCh.Reader.WaitToReadAsync(ct))
    {
        while (_logCh.Reader.TryRead(out var line)) batch.Add(line);
        if (batch.Count > 0)
        {
            var snapshot = batch.ToArray();
            batch.Clear();
            await Dispatcher.UIThread.InvokeAsync(() =>
            {
                foreach (var s in snapshot) Logs.Add(s);
            }, DispatcherPriority.Background);
        }
    }
});
```

---

## DispatcherPriority 활용

| Priority | 용도 |
|----------|------|
| `Render` | 렌더 직전 작업 |
| `Input` | 사용자 입력 처리와 유사한 타이밍 |
| `Normal` | 기본값 |
| `Background` | 낮은 우선순위 (여유 있을 때) |

```csharp
await Dispatcher.UIThread.InvokeAsync(
    () => Status = "정리 중…",
    DispatcherPriority.Background);
```

---

## 프레임 예산 (수학적 관점)

UI가 초당 \( f \) 프레임(FPS)으로 부드럽게 보이려면 프레임당 처리 시간 예산은

$$
\text{budget\_ms} = \frac{1000}{f}
$$

예를 들어 \( f = 60 \)FPS이면 **16.67ms** 이내에 렌더링 + 사용자 코드가 완료되어야 합니다.  
무거운 작업을 UI 스레드에 두면 이 예산을 초과하여 끊김이 발생합니다. → **백그라운드로 이동**하고 **UI 갱신 최소화**가 필수입니다.

---

## 흔한 실수와 대안

| 실수 | 증상 | 대안 |
|------|------|------|
| 대량 `Items.Add` | 렌더·측정 폭탄 | 스냅샷 교체/청크 추가 |
| `.Result` / `.Wait()` | 교착/프리징 | `await` 사용, UI 스레드에서는 동기 블록 금지 |
| `Task.Run` 없이 CPU 바운드 | 스크롤/클릭 멈춤 | `Task.Run` + 진행률 표시 |
| 매 이벤트마다 UI 대량 갱신 | 끊김/전력 낭비 | `Throttle`/`Sample`/배치 처리 |

---

## 실전 종합 예제: 검색 + 스트림 + 진행률 + 배치 UI

```csharp
public sealed class SearchViewModel : ReactiveObject
{
    private readonly Channel<string> _lineCh = Channel.CreateUnbounded<string>();
    private readonly ObservableCollection<string> _lines = new();
    public ReadOnlyObservableCollection<string> Lines { get; }

    public string Query { get => _q; set => this.RaiseAndSetIfChanged(ref _q, value); }
    private string _q = "";

    public double Progress { get => _p; set => this.RaiseAndSetIfChanged(ref _p, value); }
    private double _p;

    public ReactiveCommand<Unit, Unit> StartCmd { get; }
    public ReactiveCommand<Unit, Unit> StopCmd { get; }
    private CancellationTokenSource? _cts;

    public SearchViewModel()
    {
        Lines = new ReadOnlyObservableCollection<string>(_lines);

        // 쿼리 변경 → API 호출 스트림
        this.WhenAnyValue(x => x.Query)
            .Throttle(TimeSpan.FromMilliseconds(350))
            .DistinctUntilChanged()
            .Select(q => Observable.FromAsync(() => StartSearchAsync(q)))
            .Switch()
            .Subscribe();

        // 채널 소비자: 배치로 UI 갱신
        _ = Task.Run(async () =>
        {
            const int BATCH = 200;
            var batch = new List<string>(BATCH);
            while (await _lineCh.Reader.WaitToReadAsync())
            {
                while (_lineCh.Reader.TryRead(out var s)) batch.Add(s);
                if (batch.Count >= BATCH)
                {
                    var snapshot = batch.ToArray();
                    batch.Clear();
                    await Dispatcher.UIThread.InvokeAsync(() =>
                    {
                        foreach (var x in snapshot) _lines.Add(x);
                    }, DispatcherPriority.Background);
                }
            }
        });

        StartCmd = ReactiveCommand.CreateFromTask(async () =>
        {
            _cts?.Cancel();
            _cts = new CancellationTokenSource();
            await StartSearchAsync(Query);
        });

        StopCmd = ReactiveCommand.Create(() => _cts?.Cancel());
    }

    private async Task StartSearchAsync(string q)
    {
        _cts?.Cancel();
        _cts = new CancellationTokenSource();
        var ct = _cts.Token;

        _lines.Clear();
        Progress = 0;

        var pr = new Progress<double>(v =>
            Dispatcher.UIThread.Post(() => Progress = v, DispatcherPriority.Background));

        await Task.Run(async () =>
        {
            await foreach (var line in Api.StreamSearchAsync(q, pr, ct))
                await _lineCh.Writer.WriteAsync(line, ct);
        }, ct);
    }
}
```

---

## 유틸리티 확장

```csharp
public static class Ui
{
    public static ValueTask RunAsync(Action a, DispatcherPriority p = DispatcherPriority.Normal)
    {
        if (Dispatcher.UIThread.CheckAccess()) { a(); return ValueTask.CompletedTask; }
        return new ValueTask(Dispatcher.UIThread.InvokeAsync(a, p));
    }
}

public static class ItemsExt
{
    public static void ReplaceAll<T>(this IList<T> self, IEnumerable<T> src)
    {
        self.Clear();
        foreach (var x in src) self.Add(x);
    }
}
```

---

## 결론

Avalonia에서 멀티스레딩을 안전하게 다루기 위한 원칙은 다음과 같습니다.

- **연산은 백그라운드** (`Task.Run`, 채널 등)
- **UI 갱신만 Dispatcher** (`InvokeAsync`)
- **ReactiveUI의 스케줄러**로 선언적 스레드 전환
- **대량 데이터는 배치/스냅샷/가상화**로 처리
- **취소 토큰**과 **진행률 보고**로 사용자 경험 향상

$$
\boxed{\text{UI Smoothness} \Longleftrightarrow \text{BG Work Offload} + \text{UI-thread-only Minimal Updates}}
$$

이 패턴을 지키면 끊김 없는 UI와 안정적인 동시성을 동시에 얻을 수 있습니다.