---
layout: post
title: Avalonia - Avalonia with ReactiveUI
date: 2025-04-07 19:20:23 +0900
category: Avalonia
---
# ⚙️ Avalonia with ReactiveUI 심화  

## 고급 Rx 연산자 (WhenAny, Throttle 등) 활용

---

## 🎯 목표

| 항목 | 설명 |
|------|------|
| 💡 WhenAnyValue | 속성 변경 감지 및 실시간 반응 |
| ⏱ Throttle | 입력 지연 처리 (ex. 실시간 검색) |
| 🧪 CombineLatest | 다중 속성 동시 처리 |
| 🧮 Select / Switch | 조건 연산 및 흐름 분기 |
| ⚙️ ReactiveCommand 개선 | CanExecute 동적 처리 |
| 📦 응용 | Live Validation, 검색어 필터링, AutoSave 등

---

## 📌 전제: ReactiveUI + Avalonia 구성

```bash
dotnet add package ReactiveUI
dotnet add package Avalonia.ReactiveUI
```

---

## 1️⃣ `WhenAnyValue` - 속성 반응 감지

```csharp
public class LoginViewModel : ReactiveObject
{
    public string Username { get => _username; set => this.RaiseAndSetIfChanged(ref _username, value); }
    public string Password { get => _password; set => this.RaiseAndSetIfChanged(ref _password, value); }

    private string _username;
    private string _password;

    public ObservableAsPropertyHelper<bool> CanLogin { get; }

    public LoginViewModel()
    {
        CanLogin = this.WhenAnyValue(
            x => x.Username,
            x => x.Password,
            (u, p) => !string.IsNullOrWhiteSpace(u) && !string.IsNullOrWhiteSpace(p)
        ).ToProperty(this, x => x.CanLogin);
    }
}
```

- `WhenAnyValue`: 프로퍼티 변경을 감지해 즉시 반응
- `ToProperty`: `ObservableAsPropertyHelper`로 OneWay 바인딩

---

## 2️⃣ `Throttle` - 입력 지연 처리 (AutoSearch, Debounce)

```csharp
public string Query { get => _query; set => this.RaiseAndSetIfChanged(ref _query, value); }
private string _query;

public ObservableCollection<string> SearchResults { get; } = new();

public SearchViewModel()
{
    this.WhenAnyValue(x => x.Query)
        .Throttle(TimeSpan.FromMilliseconds(500)) // 0.5초 입력 지연
        .DistinctUntilChanged()
        .ObserveOn(RxApp.MainThreadScheduler)
        .Subscribe(async q =>
        {
            var results = await _api.SearchAsync(q);
            SearchResults.Clear();
            foreach (var r in results)
                SearchResults.Add(r);
        });
}
```

- `Throttle`: 입력이 멈춘 후 500ms 뒤에 실행
- `ObserveOn`: UI Thread에서 Collection 조작

---

## 3️⃣ `CombineLatest` - 여러 값 동시에 추적

```csharp
public ObservableAsPropertyHelper<string> StatusMessage { get; }

public MyViewModel()
{
    StatusMessage = this.WhenAnyValue(x => x.Username, x => x.Password)
        .CombineLatest(this.WhenAnyValue(x => x.IsConnected), 
          (up, connected) =>
          {
              var (username, password) = up;
              return connected
                  ? $"입력완료: {username}, {password}"
                  : "네트워크 연결 없음";
          })
        .ToProperty(this, x => x.StatusMessage);
}
```

---

## 4️⃣ `Select`, `Switch`, `Where`, `Merge` 등 활용 예

### ✅ `Select` + `Switch`로 API 호출 흐름 제어

```csharp
var searchResults = this.WhenAnyValue(x => x.Query)
    .Throttle(TimeSpan.FromMilliseconds(300))
    .Where(q => !string.IsNullOrWhiteSpace(q))
    .Select(query => Observable.FromAsync(() => _api.SearchAsync(query)))
    .Switch() // 이전 요청 취소
    .ObserveOn(RxApp.MainThreadScheduler);

searchResults.Subscribe(results =>
{
    SearchResults.Clear();
    foreach (var r in results) SearchResults.Add(r);
});
```

### ✅ `Merge`: 여러 이벤트를 하나로 통합

```csharp
var trigger1 = this.WhenAnyValue(x => x.Username).Select(_ => Unit.Default);
var trigger2 = this.WhenAnyValue(x => x.Password).Select(_ => Unit.Default);

trigger1.Merge(trigger2)
    .Throttle(TimeSpan.FromMilliseconds(200))
    .Subscribe(_ => Validate());
```

---

## 5️⃣ ReactiveCommand + CanExecute 연동

```csharp
public ReactiveCommand<Unit, Unit> LoginCommand { get; }

public LoginViewModel()
{
    var canLogin = this.WhenAnyValue(x => x.Username, x => x.Password,
        (u, p) => !string.IsNullOrWhiteSpace(u) && !string.IsNullOrWhiteSpace(p));

    LoginCommand = ReactiveCommand.CreateFromTask(LoginAsync, canLogin);
}
```

- `CanExecute`는 자동으로 버튼 바인딩에 반영됨

---

## 6️⃣ 응용: 자동 저장 (AutoSave), 폼 유효성, Live Preview

```csharp
this.WhenAnyValue(x => x.FormData)
    .Throttle(TimeSpan.FromSeconds(2))
    .Where(_ => IsValid)
    .Select(_ => Observable.FromAsync(() => SaveAsync()))
    .Switch()
    .Subscribe();
```

---

## ✅ 정리

| 연산자 | 설명 | 사용 예 |
|--------|------|---------|
| `WhenAnyValue` | 속성 변화 감지 | 로그인 버튼 활성화 |
| `Throttle` | 지연 실행 | 실시간 검색, AutoSave |
| `CombineLatest` | 복수 속성 결합 | 메시지, 유효성 |
| `Select`, `Switch` | 흐름 전환 | 비동기 API 호출 |
| `Merge`, `Where` | 조건 반응 | 이벤트 결합, 검증 트리거 |
| `ToProperty` | OneWay 바인딩 | 상태 메시지, 결과 |

---

## 📚 참고 자료

- [ReactiveUI 공식 문서](https://www.reactiveui.net/)
- [Rx.NET 연산자 목록](https://github.com/dotnet/reactive/wiki/Query-Operators)
- [Avalonia + ReactiveUI 템플릿](https://github.com/AvaloniaCommunity/ReactiveUI.Avalonia.Templates)

---

## 🔚 결론

ReactiveUI와 Avalonia를 함께 사용하면 단순한 MVVM을 넘어 **진짜 반응형(Rx 기반)** UI를 구성할 수 있습니다.

이 구조를 활용하면 다음과 같은 앱을 쉽게 개발할 수 있습니다:

- ✅ 실시간 필터/검색 기반 UI
- ✅ 다중 조건에 따른 상태 처리
- ✅ AutoSave, Live Validation 등 UX 중심 기능