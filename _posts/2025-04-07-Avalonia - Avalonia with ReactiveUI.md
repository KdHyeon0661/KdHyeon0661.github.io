---
layout: post
title: Avalonia - Avalonia with ReactiveUI
date: 2025-04-07 19:20:23 +0900
category: Avalonia
---
# Avalonia with ReactiveUI 심화 — 고급 Rx 연산자(WhenAny, Throttle 등)

ReactiveUI는 Avalonia와 함께 MVVM을 더욱 강력하게 만드는 프레임워크입니다. 특히 **Reactive Extensions(Rx)** 를 기반으로 한 **반응형 프로그래밍**을 통해 복잡한 UI 흐름을 간결하고 선언적으로 표현할 수 있습니다. 이 글에서는 실제 프로젝트에서 자주 사용되는 고급 Rx 연산자와 패턴을 초중급 개발자 관점에서 설명합니다.

---

## 기본 구성

### 패키지 설치

```bash
dotnet add package ReactiveUI
dotnet add package Avalonia.ReactiveUI
dotnet add package DynamicData            # 리스트 반응형 처리 (선택)
dotnet add package ReactiveUI.Validation  # 검증 확장 (선택)
```

### ViewModel 기본 클래스

모든 ViewModel은 `ReactiveObject`를 상속하고, 구독 관리를 위해 `CompositeDisposable`을 사용합니다.

```csharp
public abstract class ViewModelBase : ReactiveObject, IActivatableViewModel, IDisposable
{
    public ViewModelActivator Activator { get; } = new();
    private readonly CompositeDisposable _disposables = new();

    protected CompositeDisposable Anchors => _disposables;

    public void Dispose() => _disposables.Dispose();
}
```

- `IActivatableViewModel`는 뷰가 활성화/비활성화될 때 자동으로 리소스를 정리할 수 있게 합니다.
- 모든 구독은 `Anchors`에 추가하여 ViewModel이 해제될 때 함께 정리됩니다.

---

## WhenAny / WhenAnyValue — 속성 반응의 기본

가장 자주 사용하는 연산자는 `WhenAnyValue`입니다. 특정 속성이 변경될 때마다 관찰 가능한 스트림을 생성합니다.

### 로그인 폼 예제

```csharp
public sealed class LoginViewModel : ViewModelBase
{
    private string _username = "";
    public string Username 
    { 
        get => _username; 
        set => this.RaiseAndSetIfChanged(ref _username, value); 
    }

    private string _password = "";
    public string Password 
    { 
        get => _password; 
        set => this.RaiseAndSetIfChanged(ref _password, value); 
    }

    public ObservableAsPropertyHelper<bool> CanLogin { get; }
    public ReactiveCommand<Unit, Unit> LoginCommand { get; }

    public LoginViewModel(IAuthApi api)
    {
        // 두 속성의 유효성을 조합하여 로그인 가능 여부 판단
        var canLoginObs = this.WhenAnyValue(
                x => x.Username, x => x.Password,
                (u, p) => !string.IsNullOrWhiteSpace(u) && !string.IsNullOrWhiteSpace(p))
            .DistinctUntilChanged()        // 값이 실제로 바뀔 때만 통과
            .Publish()
            .RefCount();                   // 다중 구독 시 하나의 소스 공유

        // ObservableAsPropertyHelper로 UI 바인딩용 속성 생성
        CanLogin = canLoginObs
            .ToProperty(this, x => x.CanLogin, initialValue: false)
            .DisposeWith(Anchors);

        // ReactiveCommand 생성 (CanExecute 소스 연결)
        LoginCommand = ReactiveCommand.CreateFromTask(async () =>
        {
            await api.LoginAsync(Username, Password);
        }, canLoginObs)
        .DisposeWith(Anchors);
    }
}
```

- `WhenAnyValue`는 각 속성의 변경을 감지해 새로운 값을 발행합니다.
- `DistinctUntilChanged`는 중복된 연속 값을 제거해 불필요한 재계산을 막습니다.
- `ToProperty`는 `ObservableAsPropertyHelper`를 생성하여 UI에 단방향 바인딩을 제공합니다.
- `ReactiveCommand`의 `canExecute` 인자에 관찰 가능 스트림을 주면 버튼 활성화가 자동으로 관리됩니다.

---

## Throttle / Debounce — 과도한 호출 방지

사용자가 타이핑할 때마다 API를 호출하면 부하가 큽니다. `Throttle`을 사용해 입력이 멈춘 후 일정 시간이 지나면 한 번만 호출하도록 할 수 있습니다.

### 실시간 검색

```csharp
public sealed class SearchViewModel : ViewModelBase
{
    private readonly ISearchApi _api;

    public string Query 
    { 
        get => _query; 
        set => this.RaiseAndSetIfChanged(ref _query, value); 
    }
    private string _query = "";

    public ObservableCollection<string> Results { get; } = new();

    public SearchViewModel(ISearchApi api)
    {
        _api = api;

        this.WhenAnyValue(x => x.Query)
            .Throttle(TimeSpan.FromMilliseconds(400), RxApp.TaskpoolScheduler)
            .DistinctUntilChanged()
            .Select(q => q?.Trim() ?? "")
            .Where(q => q.Length >= 2)
            .Select(q => Observable.FromAsync(ct => _api.SearchAsync(q, ct)))
            .Switch()                             // 가장 최근 요청만 유지
            .ObserveOn(RxApp.MainThreadScheduler) // UI 갱신은 메인 스레드
            .Subscribe(items =>
            {
                Results.Clear();
                foreach (var i in items) Results.Add(i);
            }, ex => { /* 에러 처리 */ })
            .DisposeWith(Anchors);
    }
}
```

- **Throttle**: 마지막 입력 후 400ms 동안 추가 입력이 없으면 값을 발행합니다.
- **TaskpoolScheduler**: 백그라운드 스레드에서 타이밍 계산을 수행합니다.
- **Switch**: 여러 비동기 요청 중 가장 최신 것만 완료되도록 이전 요청을 취소합니다.
- **ObserveOn**: UI 갱신을 메인 스레드로 전환합니다.

> `Throttle`은 일반적으로 "입력 멈춤 후" 한 번만 발행하는 Debounce 역할을 합니다. ReactiveUI에서는 `Throttle`을 주로 사용합니다.

---

## CombineLatest / Zip — 여러 속성 결합

여러 속성의 최신 값을 조합하여 새로운 값을 생성할 때 `CombineLatest`를 사용합니다.

### 상태 표시줄 예제

```csharp
public sealed class StatusBarViewModel : ViewModelBase
{
    public ObservableAsPropertyHelper<string> Status { get; }

    private string _user = "";
    public string User 
    { 
        get => _user; 
        set => this.RaiseAndSetIfChanged(ref _user, value); 
    }

    private bool _online;
    public bool Online 
    { 
        get => _online; 
        set => this.RaiseAndSetIfChanged(ref _online, value); 
    }

    private bool _busy;
    public bool Busy 
    { 
        get => _busy; 
        set => this.RaiseAndSetIfChanged(ref _busy, value); 
    }

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

- `CombineLatest`는 각 스트림의 최신 값을 조합해 새 값을 발행합니다.
- `Zip`은 각 스트림의 순서쌍을 묶어야 할 때 사용합니다.

---

## Select / Switch / Where / Merge — 비동기 흐름 제어

### API 호출 취소 가능한 파이프라인

```csharp
this.WhenAnyValue(x => x.Query)
    .Throttle(TimeSpan.FromMilliseconds(250))
    .Select(q => q?.Trim() ?? "")
    .Where(q => q.Length > 0)
    .Select(q => Observable.FromAsync(ct => _api.SearchAsync(q, ct)))
    .Switch()
    .ObserveOn(RxApp.MainThreadScheduler)
    .Subscribe(items => UpdateItems(items))
    .DisposeWith(Anchors);
```

- **Select**는 각 입력값을 비동기 작업을 반환하는 `Observable`로 변환합니다.
- **Switch**는 가장 최근에 생성된 `Observable`만 구독하고 이전 것은 취소합니다.
- **Where**로 필터링을, **Merge**로 여러 스트림을 합칠 수 있습니다.

---

## ReactiveCommand — 명령의 완전한 제어

`ReactiveCommand`는 `ICommand` 구현체로, 비동기 작업, 실행 가능 여부, 실행 상태, 예외 처리를 통합합니다.

### 표준 패턴

```csharp
public sealed class SaveViewModel : ViewModelBase
{
    private readonly IRepository _repo;

    public bool IsDirty 
    { 
        get => _isDirty; 
        set => this.RaiseAndSetIfChanged(ref _isDirty, value); 
    }
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

        // 실행 중인지 여부를 UI에 바인딩
        SaveCommand.IsExecuting
            .ToProperty(this, x => x.IsSaving, out IsSaving)
            .DisposeWith(Anchors);

        // 예외 처리 (UI에 알림 표시)
        SaveCommand.ThrownExceptions
            .ObserveOn(RxApp.MainThreadScheduler)
            .Subscribe(ex => { /* 에러 메시지 표시 */ })
            .DisposeWith(Anchors);
    }
}
```

- **CreateFromTask**: 비동기 작업을 실행하는 명령을 생성합니다.
- **IsExecuting**: 작업 진행 중 여부 (ProgressBar, 스피너에 바인딩)
- **ThrownExceptions**: 명령 실행 중 발생한 예외를 처리할 수 있는 스트림

---

## 수동 검증 vs ReactiveUI.Validation

### 수동 검증

```csharp
public ObservableAsPropertyHelper<string?> UsernameError { get; }

UsernameError = this.WhenAnyValue(x => x.Username)
    .Select(u => string.IsNullOrWhiteSpace(u) ? "사용자명을 입력하세요." : null)
    .ToProperty(this, x => x.UsernameError)
    .DisposeWith(Anchors);
```

### ReactiveUI.Validation 사용

```csharp
public sealed class ProfileViewModel : ReactiveValidationObject
{
    private string _email = "";
    public string Email 
    { 
        get => _email; 
        set => this.RaiseAndSetIfChanged(ref _email, value); 
    }

    public ProfileViewModel()
    {
        this.ValidationRule(vm => vm.Email,
            email => !string.IsNullOrWhiteSpace(email) && email.Contains("@"),
            "올바른 이메일을 입력하세요.");
    }
}
```

- `ReactiveValidationObject`는 `INotifyDataErrorInfo`를 구현하므로 Avalonia의 `:invalid` 스타일과 연동됩니다.

---

## AutoSave / 지연 저장

사용자 입력 후 일정 시간이 지나면 자동 저장하는 패턴입니다.

```csharp
public sealed class EditorViewModel : ViewModelBase
{
    private string _content = "";
    public string Content 
    { 
        get => _content; 
        set => this.RaiseAndSetIfChanged(ref _content, value); 
    }

    public EditorViewModel(IDocStore store)
    {
        this.WhenAnyValue(x => x.Content)
            .Skip(1)                     // 초기값 무시
            .Throttle(TimeSpan.FromSeconds(2))
            .Select(_ => Observable.FromAsync(ct => store.SaveAsync(Content, ct)))
            .Switch()
            .Subscribe(_ => { /* 자동 저장 완료 표시 */ }, 
                       ex => { /* 오류 표시 */ })
            .DisposeWith(Anchors);
    }
}
```

- `Skip(1)`은 처음 바인딩 시 발생하는 초기 변경을 무시합니다.
- `Throttle` 후 `Select`로 비동기 작업 생성, `Switch`로 중복 요청 취소.

---

## DynamicData를 이용한 리스트 반응형 처리

많은 양의 컬렉션을 필터, 정렬, 그룹화할 때는 **DynamicData**를 사용하면 성능과 편의성이 좋습니다.

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

// 항목 추가/수정/삭제는 _orders.Edit()을 통해 수행
```

- **SourceList**는 변경 가능한 컬렉션입니다.
- **Connect()**로 변경 사항을 관찰하고, 필터/정렬 후 **Bind()**로 읽기 전용 컬렉션에 연결합니다.

---

## 스케줄러와 스레딩

| 작업 | 권장 스케줄러 |
|------|---------------|
| UI 바인딩, 컬렉션 조작 | `RxApp.MainThreadScheduler` |
| 입력 처리, 타이머, API 호출 준비 | `RxApp.TaskpoolScheduler` |
| 단위 테스트 | `TestScheduler` (가상 시간) |

```csharp
// 백그라운드에서 계산, 결과를 UI에 반영
this.WhenAnyValue(x => x.SearchTerm)
    .Throttle(TimeSpan.FromMilliseconds(300), RxApp.TaskpoolScheduler)
    .Select(term => ComputeExpensive(term))
    .ObserveOn(RxApp.MainThreadScheduler)
    .Subscribe(result => UpdateUI(result));
```

---

## 수명 관리: DisposeWith와 WhenActivated

모든 구독은 ViewModel이 해제될 때 함께 정리되어야 합니다.

```csharp
// ViewModel 생성 시 Anchors에 추가
this.WhenAnyValue(x => x.Query)
    .Subscribe(...)
    .DisposeWith(Anchors);
```

**WhenActivated**는 뷰가 활성화/비활성화될 때 구독을 자동으로 관리합니다.

```csharp
public sealed class MyViewModel : ViewModelBase
{
    public MyViewModel(IService svc)
    {
        this.WhenActivated(disposables =>
        {
            // 뷰가 활성화될 때 실행
            this.WhenAnyValue(x => x.Query)
                .Subscribe(...)
                .DisposeWith(disposables);
        });
    }
}
```

- `disposables`는 뷰가 비활성화될 때 함께 해제됩니다.

---

## 통합 예제: 검색 페이지

```csharp
public sealed class SearchPageViewModel : ViewModelBase
{
    private readonly ISearchApi _api;

    public string Query 
    { 
        get => _query; 
        set => this.RaiseAndSetIfChanged(ref _query, value); 
    }
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

        // 자동 검색 (Throttle + Switch)
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
            }, ex => { /* 오류 처리 */ })
            .DisposeWith(Anchors);

        // 수동 새로고침
        RefreshCommand = ReactiveCommand.CreateFromTask(async ct =>
        {
            var list = await _api.SearchAsync(Query.Trim(), ct);
            _items.Clear();
            foreach (var i in list) _items.Add(i);
        }, queryValid);

        RefreshCommand.IsExecuting
            .ToProperty(this, x => x.Busy, out var busyProp)
            .DisposeWith(Anchors);
        Busy = busyProp;
    }
}
```

XAML 바인딩:

```xml
<StackPanel>
  <TextBox Text="{Binding Query, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"/>
  <TextBlock Text="{Binding Error}" Foreground="Red"/>
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

## 요약표

| 주제 | 핵심 연산자/기술 | 설명 |
|------|------------------|------|
| 속성 반응 | `WhenAnyValue`, `ToProperty` | 속성 변화 감지, 계산된 속성 생성 |
| 입력 지연 | `Throttle`, `DistinctUntilChanged` | 불필요한 호출 방지, 중복 제거 |
| 다중 속성 결합 | `CombineLatest`, `Zip` | 여러 상태 조합 |
| 비동기 제어 | `Select`, `Switch`, `Merge` | 최신 요청만 반영, 병렬 처리 |
| 명령 | `ReactiveCommand` | 실행 조건, 상태, 예외 통합 |
| 검증 | 수동 검증 / `ReactiveUI.Validation` | UI 피드백 연동 |
| 스레딩 | `ObserveOn`, `SubscribeOn` | UI/백그라운드 분리 |
| 수명 관리 | `DisposeWith`, `WhenActivated` | 메모리 누수 방지 |
| 대규모 컬렉션 | DynamicData | 고성능 필터/정렬/그룹화 |

---

## 결론

ReactiveUI는 Avalonia MVVM 개발에 **반응형 프로그래밍의 힘**을 제공합니다.  
- **WhenAnyValue**와 **ToProperty**로 속성 간 의존성을 선언적으로 표현합니다.  
- **Throttle + Switch**로 과도한 API 호출을 막고 최신 요청만 반영합니다.  
- **ReactiveCommand**로 명령의 실행 가능 여부, 진행 상태, 예외를 일관되게 관리합니다.  
- **DisposeWith / WhenActivated**로 리소스 누수를 방지합니다.  

이러한 패턴을 익히면 복잡한 상호작용도 안정적이고 유지보수하기 쉬운 코드로 작성할 수 있습니다.