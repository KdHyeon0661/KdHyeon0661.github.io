---
layout: post
title: Avalonia - Avalonia with ReactiveUI
date: 2025-04-07 19:20:23 +0900
category: Avalonia
---
# Avalonia with ReactiveUI 심화 — 고급 Rx 연산자(WhenAny, Throttle 등)

## 0. 전제 및 구성

### 프로젝트 의존성

```bash
dotnet add package ReactiveUI
dotnet add package Avalonia.ReactiveUI
dotnet add package DynamicData            # 리스트/컬렉션 Rx에 유리(선택)
dotnet add package ReactiveUI.Validation  # 검증 확장(선택)
```

### 기본 베이스 클래스

```csharp
public abstract class ViewModelBase : ReactiveObject, IActivatableViewModel, IDisposable
{
    public ViewModelActivator Activator { get; } = new();
    private readonly CompositeDisposable _disposables = new();

    protected CompositeDisposable Anchors => _disposables;

    public void Dispose() => _disposables.Dispose();
}
```

> 모든 구독은 `Anchors`에 담아 수명 종료 시 누수 없이 정리한다.

---

## 1. WhenAny / WhenAnyValue — 프로퍼티 반응의 뼈대

### 1.1 로그인 폼의 정석 패턴

```csharp
public sealed class LoginViewModel : ViewModelBase
{
    private string _username = "";
    public string Username { get => _username; set => this.RaiseAndSetIfChanged(ref _username, value); }

    private string _password = "";
    public string Password { get => _password; set => this.RaiseAndSetIfChanged(ref _password, value); }

    public ObservableAsPropertyHelper<bool> CanLogin { get; }

    public ReactiveCommand<Unit, Unit> LoginCommand { get; }

    public LoginViewModel(IAuthApi api)
    {
        var canLoginObs =
            this.WhenAnyValue(x => x.Username, x => x.Password,
                (u, p) => !string.IsNullOrWhiteSpace(u) && !string.IsNullOrWhiteSpace(p))
            .DistinctUntilChanged()
            .Publish()
            .RefCount();

        CanLogin = canLoginObs
            .ToProperty(this, x => x.CanLogin, initialValue: false)
            .DisposeWith(Anchors);

        LoginCommand = ReactiveCommand.CreateFromTask(async () =>
        {
            await api.LoginAsync(Username, Password);
        }, canLoginObs)
        .DisposeWith(Anchors);
    }
}
```

**포인트**
- `Publish().RefCount()`로 다중 구독 비용을 줄인다.
- `ToProperty`는 OneWay 바인딩 최적화(OAPH)로 View에 연결.

### 1.2 WhenAny vs WhenAnyValue

- `WhenAnyValue(x => x.Prop)` : **속성의 값 스트림**을 만든다(단순, 95% 케이스).
- `WhenAny(x => x.Prop, expr => ...)` : **속성의 변경 이벤트**를 보다 유연하게(필요 시).

---

## 2. Throttle/Debounce — 과도한 호출 억제

### 2.1 실시간 검색의 표준 흐름

```csharp
public sealed class SearchViewModel : ViewModelBase
{
    private readonly ISearchApi _api;

    public string Query { get => _query; set => this.RaiseAndSetIfChanged(ref _query, value); }
    private string _query = "";

    public ObservableCollection<string> Results { get; } = new();

    public SearchViewModel(ISearchApi api)
    {
        _api = api;

        this.WhenAnyValue(x => x.Query)
            .Throttle(TimeSpan.FromMilliseconds(400), RxApp.TaskpoolScheduler)
            .DistinctUntilChanged()
            .Select(q => q?.Trim() ?? "")
            .Where(q => q.Length >= 2)                 // 최소 길이 제한
            .Select(q => Observable.FromAsync(ct => _api.SearchAsync(q, ct)))
            .Switch()                                   // 이전 요청 취소
            .ObserveOn(RxApp.MainThreadScheduler)
            .Subscribe(items =>
            {
                Results.Clear();
                foreach (var i in items) Results.Add(i);
            }, ex => { /* 에러 표시/로깅 */ })
            .DisposeWith(Anchors);
    }
}
```

**포인트**
- **스레드**: Throttle은 `TaskpoolScheduler`로, UI 갱신은 `MainThreadScheduler`.
- **Switch**: 최신 입력만 반영(오래 걸리는 이전 호출 취소).

### 2.2 Debounce vs Throttle
- ReactiveUI에서는 보통 `Throttle`을 Debounce처럼 사용(“입력 멈춤 후” 한 번 발화).
- Polling/주기적 업데이트는 `Sample` 또는 `Interval`과 조합.

---

## 3. CombineLatest / Zip — 다중 속성 결합

```csharp
public sealed class StatusBarViewModel : ViewModelBase
{
    public ObservableAsPropertyHelper<string> Status { get; }

    private string _user = "";     public string User { get => _user; set => this.RaiseAndSetIfChanged(ref _user, value); }
    private bool _online;          public bool Online { get => _online; set => this.RaiseAndSetIfChanged(ref _online, value); }
    private bool _busy;            public bool Busy { get => _busy; set => this.RaiseAndSetIfChanged(ref _busy, value); }

    public StatusBarViewModel()
    {
        Status = this.WhenAnyValue(x => x.User)
            .CombineLatest(
                this.WhenAnyValue(x => x.Online),
                this.WhenAnyValue(x => x.Busy),
                (u, online, busy) =>
                    !online ? "네트워크 끊김"
                  : busy    ? $"작업 중: {u}"
                  :           $"대기: {u}")
            .DistinctUntilChanged()
            .ToProperty(this, x => x.Status, scheduler: RxApp.MainThreadScheduler)
            .DisposeWith(Anchors);
    }
}
```

**포인트**
- `CombineLatest`는 가장 최근값 조합, `Zip`은 순서쌍(동기적)이 필요할 때.

---

## 4. Select / Switch / Where / Merge — 비동기 흐름 제어의 핵심

### 4.1 API 호출 취소 가능한 파이프라인

```csharp
var resultStream =
    this.WhenAnyValue(x => x.Query)
        .Throttle(TimeSpan.FromMilliseconds(250))
        .Select(q => q?.Trim() ?? "")
        .Where(q => q.Length > 0)
        .Select(q => Observable.FromAsync(ct => _api.SearchAsync(q, ct)))
        .Switch() // 중요: 가장 최근 요청만
        .Catch(Observable.Return(Array.Empty<Item>())); // 실패 시 빈 결과

resultStream
    .ObserveOn(RxApp.MainThreadScheduler)
    .Subscribe(UpdateItems)
    .DisposeWith(Anchors);
```

### 4.2 Merge로 여러 트리거 통합하여 검증

```csharp
var userChanged = this.WhenAnyValue(x => x.Username).Select(_ => Unit.Default);
var passChanged = this.WhenAnyValue(x => x.Password).Select(_ => Unit.Default);

userChanged.Merge(passChanged)
    .Throttle(TimeSpan.FromMilliseconds(150))
    .Subscribe(_ => Validate())
    .DisposeWith(Anchors);
```

---

## 5. ReactiveCommand — CanExecute, 예외, 진행상태

### 5.1 정석 템플릿

```csharp
public sealed class SaveViewModel : ViewModelBase
{
    private readonly IRepository _repo;

    public bool IsDirty { get => _isDirty; set => this.RaiseAndSetIfChanged(ref _isDirty, value); }
    private bool _isDirty;

    public ReactiveCommand<Unit, Unit> SaveCommand { get; }
    public ObservableAsPropertyHelper<bool> IsSaving { get; }

    public SaveViewModel(IRepository repo)
    {
        _repo = repo;

        var canSave = this.WhenAnyValue(x => x.IsDirty);

        SaveCommand = ReactiveCommand.CreateFromTask(async ct =>
        {
            await _repo.SaveAsync(ct);
            IsDirty = false;
        }, canSave);

        // 실행 상태 바인딩
        SaveCommand.IsExecuting
            .ToProperty(this, x => x.IsSaving, out IsSaving)
            .DisposeWith(Anchors);

        // 예외 파이프라인(전역 처리)
        SaveCommand.ThrownExceptions
            .ObserveOn(RxApp.MainThreadScheduler)
            .Subscribe(ex =>
            {
                // UI 알림/로깅
            })
            .DisposeWith(Anchors);
    }
}
```

**포인트**
- `IsExecuting`→ 스피너 UI와 바인딩.
- `ThrownExceptions`→ 중앙화된 에러 처리.
- `CanExecute`→ UI 버튼 활성/비활성 자동 연동.

### 5.2 결과 반환/인수 받는 Command

```csharp
public ReactiveCommand<string, Result> SubmitCommand { get; }

SubmitCommand = ReactiveCommand.CreateFromTask<string, Result>(async text =>
{
    return await _svc.SubmitAsync(text);
});
```

---

## 6. 검증(Validation) — Live Validation, 폼 유효성

### 6.1 간단한 수제 검증

```csharp
public ObservableAsPropertyHelper<string?> UsernameError { get; }

UsernameError = this.WhenAnyValue(x => x.Username)
    .Select(u => string.IsNullOrWhiteSpace(u) ? "사용자명을 입력하세요." : null)
    .ToProperty(this, x => x.UsernameError)
    .DisposeWith(Anchors);
```

### 6.2 ReactiveUI.Validation(선택)

```csharp
// 설치: ReactiveUI.Validation
public sealed class ProfileViewModel : ReactiveValidationObject
{
    private string _email = "";
    public string Email { get => _email; set => this.RaiseAndSetIfChanged(ref _email, value); }

    public ProfileViewModel()
    {
        this.ValidationRule(vm => vm.Email,
            email => !string.IsNullOrWhiteSpace(email) && email.Contains("@"),
            "올바른 이메일을 입력하세요.");
    }
}
```

---

## 7. AutoSave / 지연 저장 / 오프라인 큐

### 7.1 AutoSave(수정 2초 후 저장, 중복 요청 취소)

```csharp
public sealed class EditorViewModel : ViewModelBase
{
    private string _content = "";
    public string Content { get => _content; set => this.RaiseAndSetIfChanged(ref _content, value); }

    public EditorViewModel(IDocStore store)
    {
        this.WhenAnyValue(x => x.Content)
            .Skip(1)
            .Throttle(TimeSpan.FromSeconds(2))
            .Select(_ => Observable.FromAsync(ct => store.SaveAsync(Content, ct)))
            .Switch()
            .Subscribe(_ => { /* 표시: 자동 저장됨 */ }, ex => { /* 표시/로깅 */ })
            .DisposeWith(Anchors);
    }
}
```

### 7.2 네트워크 불가 시 로컬 큐에 저장 후 재시도

- `Select(… FromAsync()) + Catch()`로 실패 시 로컬 큐에 저장.
- 별도 `ConnectivityViewModel`의 `IsOnline` 스트림과 `CombineLatest`하여 온라인일 때 Drain.

---

## 8. 리스트/그리드 Rx — DynamicData 활용(선택)

```csharp
private readonly SourceList<Order> _orders = new();

public ReadOnlyObservableCollection<Order> View { get; }

public OrdersViewModel()
{
    _orders.Connect()
        .Filter(o => o.State != OrderState.Deleted)
        .Sort(SortExpressionComparer<Order>.Ascending(x => x.CreatedAt))
        .ObserveOn(RxApp.MainThreadScheduler)
        .Bind(out var view)
        .Subscribe()
        .DisposeWith(Anchors);

    View = view;
}
```

> 대규모 컬렉션 반응형 정렬/필터/그룹이 필요한 경우 DynamicData가 매우 유용.

---

## 9. Scheduler / 스레드 — 어디서 무엇을?

| 위치 | 권장 Scheduler |
|------|----------------|
| 입력 처리, API 호출 준비 | `RxApp.TaskpoolScheduler` |
| UI 바인딩/컬렉션 조작 | `RxApp.MainThreadScheduler` |
| 타이머/간헐적 작업 | `RxApp.TaskpoolScheduler` 또는 `NewThreadScheduler` |

**규칙**: 비UI 작업은 **백스레드**, 뷰/컬렉션 업데이트는 **메인**으로 `ObserveOn`.

---

## 10. 수명/메모리 — Dispose, Activation, View와 엮기

### 10.1 IActivatableViewModel 패턴

```csharp
this.WhenActivated(disposables =>
{
    this.WhenAnyValue(x => x.Query)
        .Throttle(TimeSpan.FromMilliseconds(300))
        .Subscribe(_ => { /* ... */ })
        .DisposeWith(disposables);
});
```

> 뷰가 나타날 때 구독 시작, 사라지면 자동 Dispose. **누수 방지 핵심**.

---

## 11. 예시 통합: 검색 + 검증 + 명령 + 상태

```csharp
public sealed class SearchPageViewModel : ViewModelBase
{
    private readonly ISearchApi _api;

    public string Query { get => _query; set => this.RaiseAndSetIfChanged(ref _query, value); }
    private string _query = "";

    public ReadOnlyObservableCollection<Item> Items => _items;
    private readonly ObservableCollection<Item> _items = new();

    public ObservableAsPropertyHelper<string?> Error { get; }
    public ReactiveCommand<Unit, Unit> RefreshCommand { get; }
    public ObservableAsPropertyHelper<bool> Busy { get; }

    public SearchPageViewModel(ISearchApi api)
    {
        _api = api;

        // 검증
        var queryValid = this.WhenAnyValue(x => x.Query)
            .Select(q => !string.IsNullOrWhiteSpace(q) && q.Trim().Length >= 2);

        Error = queryValid.Select(ok => ok ? null : "2글자 이상 입력하세요.")
            .ToProperty(this, x => x.Error)
            .DisposeWith(Anchors);

        // 자동 검색
        this.WhenAnyValue(x => x.Query)
            .Throttle(TimeSpan.FromMilliseconds(350), RxApp.TaskpoolScheduler)
            .Select(q => q?.Trim() ?? "")
            .DistinctUntilChanged()
            .Where(q => q.Length >= 2)
            .Select(q => Observable.FromAsync(ct => _api.SearchAsync(q, ct)))
            .Switch()
            .ObserveOn(RxApp.MainThreadScheduler)
            .Subscribe(list =>
            {
                _items.Clear();
                foreach (var i in list) _items.Add(i);
            }, ex => { /* 오류 표시 */ })
            .DisposeWith(Anchors);

        // 수동 새로고침
        var canRefresh = queryValid;
        RefreshCommand = ReactiveCommand.CreateFromTask(async ct =>
        {
            var list = await _api.SearchAsync(Query.Trim(), ct);
            _items.Clear();
            foreach (var i in list) _items.Add(i);
        }, canRefresh);

        RefreshCommand.IsExecuting
            .ToProperty(this, x => x.Busy, out var exec, scheduler: RxApp.MainThreadScheduler)
            .DisposeWith(Anchors);

        Busy = exec;
    }
}
```

---

## 12. 단위 테스트 전략(RxTest)

- **스케줄러 제어**: `TestScheduler`로 가상 시간 전진 → Throttle/Switch 타이밍 검증.
- **명령 테스트**: `CanExecute` 변화, `ThrownExceptions` 구독.
- **검증 테스트**: 입력 조합에 따른 에러 문자열 스트림 확인.

예시(개념):

```csharp
[Fact]
public void Throttle_Search_Emits_After_350ms()
{
    var ts = new TestScheduler();
    RxApp.MainThreadScheduler = ts;
    RxApp.TaskpoolScheduler   = ts;

    var vm = new SearchViewModel(new FakeApi());
    vm.Query = "a";
    ts.AdvanceByMs(100);   // 아직 발화 안됨
    vm.Query = "ab";
    ts.AdvanceByMs(400);   // 발화 지점

    // vm.Results 검사 등
}
```

---

## 13. 성능·품질 팁

- **DistinctUntilChanged**를 적극 사용: 불필요한 재계산 방지.
- 반복 호출이 비싼 API는 **공유 캐시**(e.g., `Replay(1).RefCount()` 또는 결과 캐싱) 고려.
- 긴 파이프라인은 **중간 결과 로그**로 디버깅 용이성 확보.
- UI 갱신은 **배치 갱신**(가능하면)으로 과도한 ObservableCollection 변경 최소화.
- 모든 `Subscribe`는 **DisposeWith**로 수명 관리.

---

## 14. XAML 바인딩 예시

```xml
<StackPanel>
  <TextBox Text="{Binding Query, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"/>
  <TextBlock Foreground="Red" Text="{Binding Error}"/>
  <Button Content="검색" Command="{Binding RefreshCommand}" IsEnabled="{Binding RefreshCommand.CanExecute}"/>
  <ListBox Items="{Binding Items}">
    <ListBox.ItemTemplate>
      <DataTemplate>
        <TextBlock Text="{Binding Name}"/>
      </DataTemplate>
    </ListBox.ItemTemplate>
  </ListBox>
  <TextBlock Text="{Binding Busy, StringFormat=로딩: {0}}"/>
</StackPanel>
```

---

## 15. 요약표

| 주제 | 핵심 포인트 | 대표 연산자/기술 |
|------|-------------|------------------|
| 속성 반응 | 값 변화에 즉각 반응 | `WhenAnyValue`, `ToProperty` |
| 입력 억제 | 검색·자동저장 지연 호출 | `Throttle`, `DistinctUntilChanged` |
| 조합 | 다중 상태 합성 | `CombineLatest`, `Zip` |
| 비동기 제어 | 최신 요청만 반영 | `Select(FromAsync)`, `Switch` |
| 명령 | 실행 가능 조건/상태/예외 | `ReactiveCommand`, `IsExecuting`, `ThrownExceptions` |
| 검증 | Live Validation | 수제 검증 또는 `ReactiveUI.Validation` |
| 스레딩 | UI vs 백그라운드 | `ObserveOn(Main)`, `Throttle(Taskpool)` |
| 수명 | 누수 방지 | `DisposeWith`, `WhenActivated` |
| 리스트 Rx | 대규모 목록 반응형 | DynamicData |

---

## 결론

ReactiveUI와 Avalonia를 결합하면, **입력 → 검증 → 비동기 호출 → 결과 반영**의 전 과정을 **명시적·선언적 스트림**으로 구성할 수 있다.  
핵심은 다음 네 가지다.

1) **Throttle + Switch**로 과도 호출과 오래된 요청을 차단한다.  
2) **WhenAnyValue + ToProperty**로 계산/상태를 OAPH로 일급화한다.  
3) **ReactiveCommand**로 UI 이벤트와 비동기 흐름을 안전하게 캡슐화한다.  
4) **Scheduler/Dispose/Activation** 규칙을 일관되게 적용해 성능과 안정성을 확보한다.