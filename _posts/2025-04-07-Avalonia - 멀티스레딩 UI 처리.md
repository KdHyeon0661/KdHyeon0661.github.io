---
layout: post
title: Avalonia - 멀티스레딩 UI 처리
date: 2025-04-07 20:20:23 +0900
category: Avalonia
---
# 🧵 Avalonia에서 멀티스레딩 UI 처리

## Dispatcher 및 UI Thread 제어

---

## 🎯 왜 필요한가?

| 시나리오 | 문제점 | 해결책 |
|----------|--------|--------|
| 백그라운드 API 호출 → UI 반영 | UI 쓰레드 외부에서 UI 변경 시 예외 발생 | UI Thread에서 실행 |
| Timer, Task.Delay 이후 UI 변경 | 동일 | `Dispatcher.UIThread.InvokeAsync()` 사용 |
| 대량 연산 → UI 멈춤 | UI Thread 점유 | 비동기 처리 + UI는 Dispatcher로 |

---

## 1️⃣ Avalonia의 Dispatcher 구조

Avalonia는 UI 스레드 작업을 위해 `Avalonia.Threading.Dispatcher.UIThread`를 사용합니다.

### ✔️ Dispatcher와 RxApp.MainThreadScheduler

| 구조 | 설명 |
|------|------|
| `Dispatcher.UIThread` | Avalonia 내장 UI 쓰레드 제어 도구 |
| `RxApp.MainThreadScheduler` | ReactiveUI에서 Avalonia UI 스케줄러 래핑 |

---

## 2️⃣ 기본 사용 예시

```csharp
// 백그라운드 작업
Task.Run(() =>
{
    var data = LoadLargeData();

    // UI 갱신은 UIThread로!
    Dispatcher.UIThread.InvokeAsync(() =>
    {
        MyList.Clear();
        foreach (var item in data)
            MyList.Add(item);
    });
});
```

- `InvokeAsync`: UI 스레드에 안전하게 액션을 큐잉
- `MyList`는 UI와 연결된 ObservableCollection 등

---

## 3️⃣ async/await 환경에서의 패턴

```csharp
public async Task LoadAsync()
{
    var result = await Task.Run(() => LoadHeavyWork());

    await Dispatcher.UIThread.InvokeAsync(() =>
    {
        StatusMessage = "로딩 완료!";
        Items.Clear();
        foreach (var item in result) Items.Add(item);
    });
}
```

> 비동기 결과를 받은 후에도 UI에 반영하려면 반드시 UI Thread에서 처리해야 함

---

## 4️⃣ UI 상태 바인딩이 있는 경우

예를 들어 스피너 처리:

```csharp
public bool IsLoading { get; set; }

public async Task FetchData()
{
    IsLoading = true;

    var result = await Task.Run(() => Api.GetData());

    await Dispatcher.UIThread.InvokeAsync(() =>
    {
        MyData = result;
        IsLoading = false;
    });
}
```

> 또는 IsLoading도 `ReactiveObject`에 바인딩하면 스피너 UI에 자동 반영됨

---

## 5️⃣ DispatcherTimer 사용

UI 쓰레드에서 실행되는 타이머 (예: 실시간 시계)

```csharp
var timer = new DispatcherTimer
{
    Interval = TimeSpan.FromSeconds(1)
};
timer.Tick += (_, __) => CurrentTime = DateTime.Now.ToString("HH:mm:ss");
timer.Start();
```

> Avalonia의 `DispatcherTimer`는 내부적으로 UIThread에 올라가기 때문에 별도 처리 필요 없음

---

## 6️⃣ Rx + Dispatcher 연동 (ReactiveUI)

```csharp
this.WhenAnyValue(x => x.Query)
    .Throttle(TimeSpan.FromMilliseconds(500))
    .Select(q => Observable.FromAsync(() => SearchAsync(q)))
    .Switch()
    .ObserveOn(RxApp.MainThreadScheduler) // UI 스레드에서 동작
    .Subscribe(results =>
    {
        Results.Clear();
        foreach (var item in results) Results.Add(item);
    });
```

- `ObserveOn`으로 UIThread 전환

---

## 7️⃣ View에서 직접 사용하는 경우

예: 로딩 메시지를 View에 출력

```csharp
Dispatcher.UIThread.Post(() =>
{
    myTextBlock.Text = "데이터 불러오는 중...";
});
```

또는:

```csharp
Dispatcher.UIThread.InvokeAsync(() =>
{
    this.FindControl<Button>("MyButton").IsEnabled = false;
});
```

---

## ✅ 팁: Dispatcher와 성능

| 방식 | 설명 | 언제 사용 |
|------|------|-----------|
| `Dispatcher.UIThread.InvokeAsync()` | UI 쓰레드에서 안전하게 실행 | 대부분의 UI 갱신 |
| `Post()` | InvokeAsync보다 약간 빠름 (즉시 실행 보장 X) | 비중요 UI 작업 |
| `CheckAccess()` | 현재 쓰레드가 UI 쓰레드인지 확인 | 조건 분기용 |

```csharp
if (Dispatcher.UIThread.CheckAccess())
{
    // UI 스레드에서 이미 실행 중
    UpdateUI();
}
else
{
    Dispatcher.UIThread.InvokeAsync(UpdateUI);
}
```

---

## 🔚 결론

Avalonia에서 멀티스레딩 처리를 잘못하면 다음과 같은 문제가 발생합니다:

- ❌ UI freeze
- ❌ InvalidOperationException (UI 요소는 UI 스레드에서만 접근 가능)

하지만 `Dispatcher.UIThread.InvokeAsync()`만 잘 활용하면 다음과 같은 이점:

- ✅ 대용량 연산은 백그라운드에서 처리
- ✅ UI는 끊김 없이 반응
- ✅ Reactive 프로그래밍과 연동 시 자연스럽게 연결됨

---

## 📚 관련 API 정리

| 메서드 | 설명 |
|--------|------|
| `Dispatcher.UIThread.InvokeAsync(Action)` | UI 쓰레드에서 작업 실행 |
| `Dispatcher.UIThread.Post(Action)` | 약간 느슨한 방식으로 작업 큐잉 |
| `Dispatcher.UIThread.CheckAccess()` | 현재 스레드 확인 |
| `DispatcherTimer` | UI 타이머 반복 작업 |

---

## 🧩 실전 활용 예시

- 🧭 Navigation 중 애니메이션과 동시에 백그라운드 API 호출
- 📊 차트 실시간 업데이트
- 💬 WebSocket 수신 → UI 반영
- 📥 다운로드 진행률 UI 갱신
- 🧪 테스트 환경에서 Dispatcher mocking (DI 가능)