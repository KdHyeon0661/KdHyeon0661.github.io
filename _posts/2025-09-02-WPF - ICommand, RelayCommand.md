---
layout: post
title: WPF - ICommand, RelayCommand
date: 2025-09-02 15:25:23 +0900
category: WPF
---
# ICommand, RelayCommand 구현 완전 가이드

WPF에서 **ICommand**는 버튼/메뉴/단축키 등의 **사용자 입력을 뷰모델의 동작**으로 연결하는 표준 인터페이스입니다.
그 구현체로 가장 널리 쓰이는 것이 **RelayCommand**(DelegateCommand)이며, 동기/비동기/제네릭 버전을 상황에 맞게 사용합니다.

---

## ICommand 기본

```csharp
public interface ICommand
{
    event EventHandler CanExecuteChanged; // 실행 가능 여부가 바뀔 때 발생
    bool CanExecute(object parameter);     // 지금 실행해도 되는가?
    void Execute(object parameter);        // 실제 실행
}
```

- **CanExecute**가 `false`면, 바인딩된 컨트롤(Button 등)은 **자동으로 비활성화**됩니다.
- **CanExecuteChanged**가 발생하면 WPF는 UI를 갱신하여 (비)활성 상태를 재평가합니다.
- WPF는 키/마우스 포커스 변화 시 자체적으로 **CommandManager.RequerySuggested**를 발생시켜 재평가합니다.
  하지만 **뷰모델 속성 변경** 등 사용자 정의 조건 변화는 자동 인지되지 않을 수 있으므로,
  **직접 CanExecuteChanged를 올리거나 `CommandManager.InvalidateRequerySuggested()`**를 호출해야 합니다.

---

## 가장 기본적인 RelayCommand (동기)

### 비제네릭 버전

```csharp
using System;
using System.Windows;
using System.Windows.Input;

public sealed class RelayCommand : ICommand
{
    private readonly Action<object?> _execute;
    private readonly Predicate<object?>? _canExecute;

    public RelayCommand(Action execute, Func<bool>? canExecute = null)
        : this(execute is null ? throw new ArgumentNullException(nameof(execute))
                               : new Action<object?>(_ => execute()),
               canExecute is null ? null : new Predicate<object?>(_ => canExecute()))
    { }

    public RelayCommand(Action<object?> execute, Predicate<object?>? canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;

        // RequerySuggested를 약한 이벤트로 구독 (메모리 누수 방지)
        WeakEventManager<CommandManager, EventArgs>
            .AddHandler(null, nameof(CommandManager.RequerySuggested), OnRequerySuggested);
    }

    public bool CanExecute(object? parameter) => _canExecute?.Invoke(parameter) ?? true;

    public void Execute(object? parameter) => _execute(parameter);

    public event EventHandler? CanExecuteChanged;

    public void RaiseCanExecuteChanged()
    {
        // UI 스레드에서 이벤트 발생
        if (Application.Current?.Dispatcher?.CheckAccess() == true)
            CanExecuteChanged?.Invoke(this, EventArgs.Empty);
        else
            Application.Current?.Dispatcher?.Invoke(() => CanExecuteChanged?.Invoke(this, EventArgs.Empty));
    }

    private void OnRequerySuggested(object? sender, EventArgs e) => RaiseCanExecuteChanged();
}
```

### 제네릭 버전 (RelayCommand\<T\>)

```csharp
using System;
using System.Windows;
using System.Windows.Input;

public sealed class RelayCommand<T> : ICommand
{
    private readonly Action<T?> _execute;
    private readonly Predicate<T?>? _canExecute;

    public RelayCommand(Action<T?> execute, Predicate<T?>? canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;

        WeakEventManager<CommandManager, EventArgs>
            .AddHandler(null, nameof(CommandManager.RequerySuggested), OnRequerySuggested);
    }

    public bool CanExecute(object? parameter)
    {
        // null 처리: 값 형식 T(예: int)일 때 null이면 default(T)로 처리
        if (parameter is null)
            return _canExecute?.Invoke(default) ?? true;

        // 타입이 다르면 캐스팅 실패 → 실행 불가로 간주
        if (parameter is T t)
            return _canExecute?.Invoke(t) ?? true;

        return false;
    }

    public void Execute(object? parameter)
    {
        T? value = parameter is null ? default : (parameter is T cast ? cast : default);
        _execute(value);
    }

    public event EventHandler? CanExecuteChanged;
    public void RaiseCanExecuteChanged()
    {
        if (Application.Current?.Dispatcher?.CheckAccess() == true)
            CanExecuteChanged?.Invoke(this, EventArgs.Empty);
        else
            Application.Current?.Dispatcher?.Invoke(() => CanExecuteChanged?.Invoke(this, EventArgs.Empty));
    }

    private void OnRequerySuggested(object? sender, EventArgs e) => RaiseCanExecuteChanged();
}
```

---

## 비동기 명령 (AsyncRelayCommand)

비동기는 `async void`가 되기 쉬워 **예외/동시 실행/취소** 관리가 어렵습니다.
아래 구현은 **재진입 방지**, **취소 토큰**, **예외 안전성**을 고려했습니다.

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Input;

public sealed class AsyncRelayCommand : ICommand
{
    private readonly Func<object?, CancellationToken, Task> _executeAsync;
    private readonly Predicate<object?>? _canExecute;
    private CancellationTokenSource? _cts;
    private bool _isExecuting;

    public AsyncRelayCommand(Func<CancellationToken, Task> executeAsync, Func<bool>? canExecute = null)
        : this((_, ct) => executeAsync(ct),
               canExecute is null ? null : new Predicate<object?>(_ => canExecute())) { }

    public AsyncRelayCommand(Func<object?, CancellationToken, Task> executeAsync,
                             Predicate<object?>? canExecute = null)
    {
        _executeAsync = executeAsync ?? throw new ArgumentNullException(nameof(executeAsync));
        _canExecute   = canExecute;

        WeakEventManager<CommandManager, EventArgs>
            .AddHandler(null, nameof(CommandManager.RequerySuggested), OnRequerySuggested);
    }

    public bool CanExecute(object? parameter) =>
        !_isExecuting && (_canExecute?.Invoke(parameter) ?? true);

    public async void Execute(object? parameter)
    {
        if (!CanExecute(parameter)) return;

        _isExecuting = true; RaiseCanExecuteChanged();
        _cts = new CancellationTokenSource();

        try
        {
            await _executeAsync(parameter, _cts.Token).ConfigureAwait(false);
        }
        catch (OperationCanceledException)
        {
            // 취소는 정상 흐름으로 무시
        }
        catch (Exception ex)
        {
            // TODO: 로깅/오류 알림
            System.Diagnostics.Debug.WriteLine(ex);
        }
        finally
        {
            _cts.Dispose(); _cts = null;
            _isExecuting = false; RaiseCanExecuteChanged();
        }
    }

    public void Cancel() => _cts?.Cancel();

    public event EventHandler? CanExecuteChanged;
    public void RaiseCanExecuteChanged()
    {
        if (Application.Current?.Dispatcher?.CheckAccess() == true)
            CanExecuteChanged?.Invoke(this, EventArgs.Empty);
        else
            Application.Current?.Dispatcher?.Invoke(() => CanExecuteChanged?.Invoke(this, EventArgs.Empty));
    }

    private void OnRequerySuggested(object? s, EventArgs e) => RaiseCanExecuteChanged();
}
```

> 팁
> - 긴 작업 중 버튼을 눌러도 **재실행이 막히도록** `_isExecuting`으로 CanExecute를 제어합니다.
> - 작업 취소가 필요하면 **`Cancel()`**을 노출하고, 내부 실행에서 **`ct.ThrowIfCancellationRequested()`**를 적절히 호출하세요.

---

## 뷰모델에서의 사용 예

```csharp
using System.Collections.ObjectModel;
using System.ComponentModel;
using System.Runtime.CompilerServices;
using System.Threading;
using System.Threading.Tasks;
using System.Windows.Input;

public class MainViewModel : INotifyPropertyChanged
{
    private string _name = "";
    private bool _isBusy;
    private int _selectedId;

    public string Name
    {
        get => _name;
        set { _name = value; OnPropertyChanged(); SaveCommand.RaiseCanExecuteChanged(); }
    }

    public bool IsBusy
    {
        get => _isBusy;
        private set { _isBusy = value; OnPropertyChanged(); }
    }

    public int SelectedId
    {
        get => _selectedId;
        set { _selectedId = value; OnPropertyChanged(); DeleteCommand.RaiseCanExecuteChanged(); }
    }

    public RelayCommand SaveCommand { get; }
    public RelayCommand<int> DeleteCommand { get; }
    public AsyncRelayCommand LoadCommand { get; }

    public ObservableCollection<string> Items { get; } = new();

    public MainViewModel()
    {
        SaveCommand   = new RelayCommand(Save, CanSave);
        DeleteCommand = new RelayCommand<int>(Delete, id => id > 0);
        LoadCommand   = new AsyncRelayCommand(async ct => await LoadAsync(ct), () => !IsBusy);
    }

    private bool CanSave() => !string.IsNullOrWhiteSpace(Name);

    private void Save()
    {
        // 저장 로직 …
        Items.Add(Name);
        Name = "";
    }

    private void Delete(int id)
    {
        // id 기반 삭제 …
    }

    private async Task LoadAsync(CancellationToken ct)
    {
        IsBusy = true; LoadCommand.RaiseCanExecuteChanged();
        try
        {
            await Task.Delay(1000, ct); // 예: API 호출
            Items.Clear();
            Items.Add("Alpha");
            Items.Add("Beta");
        }
        finally
        {
            IsBusy = false; LoadCommand.RaiseCanExecuteChanged();
        }
    }

    public event PropertyChangedEventHandler? PropertyChanged;
    private void OnPropertyChanged([CallerMemberName] string? n = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(n));
}
```

---

## XAML에서 바인딩

```xml
<Grid xmlns:i="http://schemas.microsoft.com/xaml/behaviors">
  <Grid.DataContext>
    <local:MainViewModel/>
  </Grid.DataContext>

  <StackPanel Margin="16" Orientation="Vertical" Spacing="8">
    <TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}"
             Width="240" />
    <StackPanel Orientation="Horizontal" Spacing="8">
      <Button Content="저장" Command="{Binding SaveCommand}" />
      <Button Content="삭제"
              Command="{Binding DeleteCommand}"
              CommandParameter="{Binding SelectedId}" />
      <Button Content="불러오기" Command="{Binding LoadCommand}" />
    </StackPanel>

    <!-- 단축키 바인딩 (Ctrl+S → SaveCommand) -->
    <Grid.InputBindings>
      <KeyBinding Command="{Binding SaveCommand}" Key="S" Modifiers="Control"/>
    </Grid.InputBindings>

    <ListBox ItemsSource="{Binding Items}" Height="150"/>
  </StackPanel>
</Grid>
```

> **CommandParameter**
> - `RelayCommand<T>`를 쓰면 **강타입**으로 파라미터를 받을 수 있습니다.
> - 리스트 항목 컨텍스트에서 상위 뷰모델 커맨드를 호출하려면:
>   ```xml
>   <Button Content="삭제"
>           Command="{Binding DataContext.DeleteCommand, RelativeSource={RelativeSource AncestorType=Window}}"
>           CommandParameter="{Binding Id}" />
>   ```

---

## RoutedCommand / ApplicationCommands와의 차이

- **RoutedCommand / RoutedUICommand**: **요소 트리**를 타고 올라가며 `CommandBinding`이 있는 곳에서 처리(뷰 코드비하인드 중심).
- **ICommand(RelayCommand)**: **뷰모델**에 두고 바인딩으로 직접 호출(MVVM 친화).
- 시스템 공용 명령(복사/붙여넣기 등)은 **ApplicationCommands.Copy**처럼 RoutedCommand를 그대로 써도 됩니다.
  MVVM에서는 보통 **뷰모델 커맨드(ICommand)**를 기본으로, 필요 시 RoutedCommand를 혼용합니다.

---

## 흔한 실수 & 베스트 프랙티스

1. **`async void` 남발 금지**
   - 비동기 커맨드는 `AsyncRelayCommand` 같은 **Task 기반**으로 구현하여 예외/취소/재진입을 관리하세요.
2. **CanExecute 갱신 누락**
   - 관련 속성 setter에서 `command.RaiseCanExecuteChanged()`를 호출하세요.
   - 또는 전역으로 `CommandManager.InvalidateRequerySuggested()`를 호출(남용은 성능 저하).
3. **메모리 누수**
   - `CommandManager.RequerySuggested`는 **정적 이벤트**입니다. `WeakEventManager`를 통해 약한 구독을 권장.
4. **UI 타입의 뷰모델 침투**
   - `Visibility/Brush` 등은 **Converter**나 **DataTrigger**로 처리하고, 뷰모델은 **순수 상태**만 노출하세요.
5. **멀티 파라미터 전달**
   - `RelayCommand<(int id, string name)>`처럼 **튜플**을 쓰거나, `MultiBinding + IValueConverter`로 포장하세요.

---

## 프레임워크/툴킷 활용 (선택)

보일러플레이트를 줄이려면 **CommunityToolkit.Mvvm**을 고려하세요.

```csharp
// Install-Package CommunityToolkit.Mvvm
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public partial class OrdersViewModel : ObservableObject
{
    [ObservableProperty] private bool isBusy;

    [RelayCommand(CanExecute = nameof(CanRefresh))]
    private async Task RefreshAsync()
    {
        IsBusy = true;
        try { /* ... */ }
        finally { IsBusy = false; }
    }

    private bool CanRefresh() => !IsBusy;
}
```

- `RelayCommand`, `AsyncRelayCommand`가 **속성 변경을 자동 감지**하도록 특성으로 연결되며,
  `NotifyCanExecuteChangedFor` 등 고급 기능도 지원합니다.

---

## 단위 테스트 포인트

- **CanExecute**: 상태에 따라 `true/false`가 올바르게 변하는지
- **Execute**: 동작이 기대대로 호출되는지
- **Async**: 재진입이 막히는지, 취소가 작동하는지, 예외가 안전하게 처리되는지

```csharp
[Fact]
public void SaveCommand_Disabled_When_Name_Is_Empty()
{
    var vm = new MainViewModel();
    vm.Name = "";
    Assert.False(vm.SaveCommand.CanExecute(null));
    vm.Name = "A";
    Assert.True(vm.SaveCommand.CanExecute(null));
}
```

---

## 결론

- **ICommand**는 WPF 입력을 MVVM으로 연결하는 표준 인터페이스입니다.
- **RelayCommand(동기/제네릭)**로 대부분의 동작을,
  **AsyncRelayCommand(취소/재진입 방지)**로 비동기 시나리오를 안정적으로 처리하세요.
- **CanExecute 갱신**, **WeakEventManager 구독**, **UI 타입 분리**를 습관화하면
  **테스트 가능한 견고한 MVVM 코드베이스**를 유지할 수 있습니다.
