---
layout: post
title: Avalonia - 멀티스레딩 UI 처리
date: 2025-04-07 20:20:23 +0900
category: Avalonia
---
# Avalonia에서 멀티스레딩 UI 처리

## 왜 Dispatcher인가?

Avalonia의 모든 `Visual` 은 **UI 스레드 소유**입니다. 다른 스레드에서 접근하면 예외/경합/크래시가 납니다.
따라서 **백그라운드**에서 무거운 일을 처리하고, **UI 갱신만 UI 스레드**로 안전하게 큐잉해야 합니다.

| 시나리오 | 문제 | 권장 해결책 |
|---|---|---|
| 대량 API 호출 → Grid 바인딩 갱신 | InvalidOperationException | 백그라운드 `Task.Run` + `Dispatcher.UIThread.InvokeAsync` 로 UI 업데이트 |
| 실시간 스트림(웹소켓) 수신 | UI 프리징/경합 | 채널/버퍼로 수신 → 배치 단위로 UI 스레드 갱신 |
| 타이머/지연 처리 | 레이아웃 지연/튀는 UI | `DispatcherTimer` 또는 Rx Throttle + MainThreadScheduler |

---

## Avalonia Dispatcher 기초

```csharp
using Avalonia.Threading;

// UI 갱신
await Dispatcher.UIThread.InvokeAsync(() =>
{
    TitleText = "완료";
    Items.Clear();
    foreach (var x in results) Items.Add(x);
});
```

핵심 포인트:

- `InvokeAsync(Action)` / `InvokeAsync(Func<Task>)` : UI 스레드에서 실행
- `Post(Action)` : 큐잉(약간 느슨), 성능 미세개선용
- `CheckAccess()` : 이미 UI 스레드라면 바로 실행
- `DispatcherPriority` : 입력 처리/렌더/바인딩 사이의 우선순위를 조절

```csharp
if (Dispatcher.UIThread.CheckAccess()) UpdateUI();
else await Dispatcher.UIThread.InvokeAsync(UpdateUI, DispatcherPriority.Input);
```

---

## async/await + 백그라운드 작업 기본 패턴

**무거운 연산은 백그라운드**, 결괏값만 **UI 스레드**로:

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

> 팁: `ConfigureAwait(false)`는 라이브러리/서비스 계층에선 유용하지만 **ViewModel → UI 경로**에서는
> UI 스레드 복귀가 필요하므로, 복귀 직전 `Dispatcher`를 써서 의도를 명확히 합니다.

---

## ReactiveUI 스케줄러와 함께 쓰기

ReactiveUI를 사용한다면 **Avalonia 전용 메인 스케줄러**가 제공됩니다.

```csharp
this.WhenAnyValue(x => x.Query)
    .Throttle(TimeSpan.FromMilliseconds(400))
    .DistinctUntilChanged()
    .Select(q => Observable.FromAsync(() => Api.SearchAsync(q)))
    .Switch()
    .ObserveOn(RxApp.MainThreadScheduler) // ← UI 스레드
    .Subscribe(results =>
    {
        Results.Clear();
        foreach (var r in results) Results.Add(r);
    });
```

- `ObserveOn(RxApp.MainThreadScheduler)`: 스트림의 **소비(구독) 위치**를 UI 스레드로 강제
- `Switch()`: 이전 검색 취소
- `Throttle`: 과도한 API 호출 방지(입력 디바운스)

---

## DispatcherTimer vs System.Timers.Timer vs PeriodicTimer

| 타이머 | 실행 스레드 | 용도 | 주의 |
|---|---|---|---|
| `DispatcherTimer` | UI 스레드 | 시계/진척도/작은 애니메이션 | 핸들러 내 무거운 작업 금지 |
| `System.Timers.Timer` | 스레드풀 | 주기적 백그라운드 작업 | UI 접근 시 Dispatcher 필요 |
| `PeriodicTimer` (.NET 6+) | `await` 루프 | 간단·명시적 주기 | 루프 취소토큰으로 종료 |

예시:

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

## 대량 갱신: 배치·스냅샷·가상화

**수천 개 항목**을 한 번에 `ObservableCollection`에 추가하면 **레이아웃/바인딩/Measure** 폭탄이 됩니다. 해결책:

1) **배치 갱신**: 백그라운드에서 리스트를 만든 뒤, UI 스레드에서 한 번에 교체

```csharp
var snapshot = await Task.Run(() => Repository.GetMany());
await Dispatcher.UIThread.InvokeAsync(() =>
{
    Items.ReplaceAll(snapshot); // 커스텀 확장(아래)
});
```

```csharp
public static class CollectionExt
{
    public static void ReplaceAll<T>(this IList<T> target, IEnumerable<T> source)
    {
        target.Clear();
        foreach (var x in source) target.Add(x);
    }
}
```

2) **페이지/청크 단위 추가**:

```csharp
const int CHUNK = 200;
await Dispatcher.UIThread.InvokeAsync(() =>
{
    foreach (var chunk in snapshot.Chunk(CHUNK))
        foreach (var x in chunk) Items.Add(x);
});
```

3) **가상화 컨트롤** 사용: ItemsRepeater/VirtualizingStackPanel 계열 활용

---

## 진행률 보고(Progress<T>)와 취소(CancellationToken)

```csharp
public async Task DownloadAsync(string url, IProgress<double> progress, CancellationToken ct)
{
    using var resp = await _http.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct);
    var total = resp.Content.Headers.ContentLength ?? -1L;
    await using var s = await resp.Content.ReadAsStreamAsync(ct);

    var buf = new byte[81920]; long read = 0;
    int n;
    while ((n = await s.ReadAsync(buf.AsMemory(0, buf.Length), ct)) > 0)
    {
        read += n;
        if (total > 0) progress.Report((double)read / total);
        // ... 파일에 쓰기 ...
    }
}
```

ViewModel:

```csharp
public double Progress { get => _p; set => this.RaiseAndSetIfChanged(ref _p, value); }
double _p;

public async Task StartAsync()
{
    using var cts = new CancellationTokenSource();
    var pr = new Progress<double>(v => Progress = v);
    await Task.Run(() => DownloadAsync(Url, pr, cts.Token));
}
```

> `IProgress<T>`는 UI 스레드 마샬링을 보장하지 않습니다.
> Progress 생성 위치가 UI 컨텍스트라면 UI로 보내주지만, 안전하게 하려면 `Dispatcher.InvokeAsync` 랩핑을 고려하세요.

---

## 스레드-세이프 파이프라인: Channel / BlockingCollection

실시간 로그/센서/소켓 스트림을 **생산자-소비자**로 처리:

```csharp
using System.Threading.Channels;

private readonly Channel<string> _logCh = Channel.CreateUnbounded<string>();

// Producer (BG)
_ = Task.Run(async () =>
{
    await foreach (var line in SocketReader(ct))
        await _logCh.Writer.WriteAsync(line, ct);
});

// Consumer (UI)
_ = Task.Run(async () =>
{
    while (await _logCh.Reader.WaitToReadAsync(ct))
        while (_logCh.Reader.TryRead(out var line))
            await Dispatcher.UIThread.InvokeAsync(() => Logs.Add(line));
});
```

장점: **경합 최소화**, **역류(backpressure)** 제어 가능, 테스트도 수월.

---

## Deadlock/프리징 방지 체크리스트

- UI 스레드에서 **동기 블록 I/O 금지** (`.Result`, `.Wait()` 자제)
- 무거운 바운드 작업은 `Task.Run`
- UI 갱신은 `Dispatcher.InvokeAsync`
- 긴 루프는 **주기적으로 `await Task.Yield()` / `await Task.Delay(0)`**로 협력적 양보

```csharp
for (int i = 0; i < N; i++)
{
    // 무거운 계산 일부
    if (i % 500 == 0) await Task.Yield(); // UI 이벤트 펌프에 기회 제공
}
```

---

## DispatcherPriority 활용

| Priority | 쓰임새 |
|---|---|
| `Render` | 렌더 직전 작업 |
| `Input` | 사용자 입력 처리와 비슷한 타이밍 |
| `Normal` | 기본 |
| `Background` | 낮은 우선순위, 틱 후 밀려도 되는 UI 갱신 |

```csharp
await Dispatcher.UIThread.InvokeAsync(
    () => Status = "정리 중…",
    DispatcherPriority.Background);
```

---

## 예: 다운로드 + 해시 검증 + 리스트 갱신(종합)

```csharp
public ReactiveCommand<Unit, Unit> FetchCmd { get; }

public MyVm()
{
    FetchCmd = ReactiveCommand.CreateFromTask(async () =>
    {
        IsLoading = true;

        var pr = new Progress<double>(v =>
        {
            // 진행률은 가볍게, 우선순위 낮춰 큐잉
            Dispatcher.UIThread.Post(() => Progress = v, DispatcherPriority.Background);
        });

        var data = await Task.Run(() => Downloader.FetchAsync(Url, pr, Ct.Token));
        var isOk = await Task.Run(() => Crypto.VerifySha256(data, ExpectedHash));

        await Dispatcher.UIThread.InvokeAsync(() =>
        {
            if (!isOk) { Error = "해시 불일치"; return; }
            Items.ReplaceAll(Parse(data));
            Status = "완료";
            IsLoading = false;
        }, DispatcherPriority.Input);
    });
}
```

---

## 수학적 관점(프레임 예산)

UI가 **초당 \( f \) FPS**로 부드럽게 보이려면,
프레임당 처리 시간 예산은:

$$
\text{budget\_ms} = \frac{1000}{f}
$$

예를 들어 \( f = 60 \)FPS이면, 렌더 + 입력 + 레이아웃 + 바인딩 + 사용자 코드 **합**이 **16.67ms** 이하여야 합니다.
무거운 계산을 UI 스레드에 올리면 이 예산을 쉽게 초과합니다. → **백그라운드로 이동**하고 **UI 갱신 최소화**가 필수입니다.

---

## 흔한 실수와 대안

| 실수 | 증상 | 대안 |
|---|---|---|
| 대량 `Items.Add` | 렌더·측정 폭탄 | 스냅샷 교체/청크 추가 |
| `.Result`/`.Wait()` | 교착/프리징 | `await` + 디스패치 |
| `Task.Run` 없이 CPU 바운드 | 스크롤/클릭 멈춤 | `Task.Run` + 프로그레스 |
| 매 이벤트 UI 대량 갱신 | 끊김/전력낭비 | `Throttle`/`Sample`/배치 |

---

## 테스트 전략

- **Dispatcher 있는 코드**는 가능한 한 **ViewModel에선 최소화**.
  UI 접근은 메소드 경계에서만 → **단위 테스트** 시 대부분 로직은 순수 비동기로 검증 가능.
- 스트림·타이머·채널은 **가짜 타이머/페이크 생산자**로 재현.
- 프로그레스/취소 플로우는 **시간 고정**과 **토큰 주입**으로 재현.

---

## 실전 샘플 — “검색 + 스트림 + 진행률 + 배치 UI”

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
    public ReactiveCommand<Unit, Unit> StopCmd  { get; }
    private CancellationTokenSource? _cts;

    public SearchViewModel()
    {
        Lines = new ReadOnlyObservableCollection<string>(_lines);

        // Query 변경 → API 호출 스트림
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
                while (_lineCh.Reader.TryRead(out var s))
                {
                    batch.Add(s);
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
                if (batch.Count > 0)
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

## 성능/전력 팁

- 모바일/배터리 환경을 염두: **샘플링 주기 늘리기**, `DispatcherPriority.Background` 활용
- 스크롤 중에는 **비싼 계산 중지**(가시 항목만 처리) — `IsScrolling` 플래그와 `Throttle`
- 큰 이미지/비디오/차트는 **가상화**, **지연 로딩**

---

## 요약·결론

- **원칙**: **연산은 백그라운드**, **UI 갱신만 Dispatcher**
- **Reactive 패턴**: `Throttle/Distinct/Switch/ObserveOn(MainThread)`로 깔끔한 흐름
- **대량 UI**: 스냅샷/배치/가상화로 렌더 폭탄 방지
- **타이머**: `DispatcherTimer`(UI), `PeriodicTimer`(BG) 역할 구분
- **프로그레스/취소**: 사용자 피드백과 반응성 유지
- **테스트성**: 인터페이스/채널/가짜 타이머로 단위 테스트 수월

이 패턴만 지키면, **끊김 없는 UI**와 **안정적인 동시성**을 동시에 얻을 수 있습니다.

---
```csharp
// 부록: 최소 유틸 — UI 안전 실행기
public static class Ui
{
    public static ValueTask RunAsync(Action a, DispatcherPriority p = DispatcherPriority.Normal)
    {
        if (Dispatcher.UIThread.CheckAccess()) { a(); return ValueTask.CompletedTask; }
        return new ValueTask(Dispatcher.UIThread.InvokeAsync(a, p));
    }
    public static Task<T> RunAsync<T>(Func<T> f, DispatcherPriority p = DispatcherPriority.Normal)
        => Dispatcher.UIThread.CheckAccess()
            ? Task.FromResult(f())
            : Dispatcher.UIThread.InvokeAsync(f, p).GetTask();
}
```

```csharp
// 부록: Items.ReplaceAll 확장
public static class ItemsExt
{
    public static void ReplaceAll<T>(this IList<T> self, IEnumerable<T> src)
    {
        self.Clear();
        foreach (var x in src) self.Add(x);
    }
}
```

```csharp
// 부록: ConfigureAwait(false) 안전 가이드 (서비스 계층 예)
public async Task<byte[]> GetBlobAsync(string id, CancellationToken ct)
{
    using var resp = await _http.GetAsync($"/blob/{id}", ct).ConfigureAwait(false);
    resp.EnsureSuccessStatusCode();
    return await resp.Content.ReadAsByteArrayAsync(ct).ConfigureAwait(false);
}
// ViewModel 에서 UI 갱신 전 Dispatcher.InvokeAsync 로 명시적으로 복귀
```

```csharp
// 부록: Deadlock 방지 (동기 Wait 금지)
public string Bad()
{
    // 아래는 UI 스레드에서 호출하면 교착 가능
    return Api.GetContentAsync().Result;
}
public async Task<string> Good()
{
    return await Api.GetContentAsync(); // 자연스럽게 비동기
}
```

$$
\boxed{\text{UI Smoothness} \Longleftrightarrow
\text{BG Work Offload} + \text{UI-thread-only Minimal Updates}}
$$
