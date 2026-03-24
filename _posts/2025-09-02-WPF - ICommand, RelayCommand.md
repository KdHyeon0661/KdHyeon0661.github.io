---
layout: post
title: WPF - ICommand, RelayCommand
date: 2025-09-02 15:25:23 +0900
category: WPF
---
# WPF ICommand와 RelayCommand 구현

WPF의 Command 시스템은 UI 상호작용을 체계적으로 구조화하는 핵심 메커니즘입니다. `ICommand` 인터페이스와 그 대표적 구현체인 `RelayCommand`는 MVVM 아키텍처에서 뷰와 뷰모델을 깔끔하게 분리해 줍니다. 이 글에서는 실무에서 바로 사용할 수 있는 수준의 Command 구현 방법을 단계별로 설명합니다.

## ICommand 인터페이스의 역할

`ICommand`는 사용자의 의도를 캡슐화하는 객체 지향 패턴의 구현체입니다. 단순한 메서드 호출과 다른 점은 **상태 기반의 실행 제어**와 **UI 자동 동기화**에 있습니다.

```csharp
public interface ICommand
{
    event EventHandler CanExecuteChanged;
    bool CanExecute(object parameter);
    void Execute(object parameter);
}
```

- `Execute` : 명령의 실제 동작을 수행합니다.
- `CanExecute` : 현재 상태에서 명령을 실행할 수 있는지 여부를 반환합니다. `false`를 반환하면 바인딩된 버튼 등이 자동으로 비활성화됩니다.
- `CanExecuteChanged` : `CanExecute`의 결과가 바뀔 때 발생하는 이벤트입니다. UI는 이 이벤트를 구독하여 명령의 활성화 상태를 갱신합니다.

버튼이 `ICommand` 구현체에 바인딩되면, `CanExecute`가 `false`를 반환할 때 버튼이 자동으로 비활성화됩니다. 이는 단순한 편의 기능이 아니라, UI 상태 관리를 선언적으로 처리할 수 있는 패러다임입니다.

## 기본 RelayCommand 구현

가장 널리 사용되는 `RelayCommand`는 `ICommand`를 간단하게 구현한 헬퍼 클래스입니다.

```csharp
public sealed class RelayCommand : ICommand
{
    private readonly Action<object> _execute;
    private readonly Func<object, bool> _canExecute;

    public RelayCommand(Action execute, Func<bool> canExecute = null)
        : this(_ => execute(), canExecute != null ? (Func<object, bool>)(_ => canExecute()) : null)
    {
    }

    public RelayCommand(Action<object> execute, Func<object, bool> canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }

    public bool CanExecute(object parameter) => _canExecute?.Invoke(parameter) ?? true;
    public void Execute(object parameter) => _execute(parameter);

    public event EventHandler CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }

    public void RaiseCanExecuteChanged()
    {
        CommandManager.InvalidateRequerySuggested();
    }
}
```

- `CommandManager.RequerySuggested`는 WPF가 전역적으로 명령의 실행 가능 상태를 다시 평가할 때 발생하는 이벤트입니다. 이벤트를 이 정적 이벤트에 연결하면 UI 포커스 변경 등 시스템 이벤트에도 자동으로 반응합니다.
- `RaiseCanExecuteChanged` 메서드를 통해 ViewModel에서 명령 상태를 수동으로 갱신할 수 있습니다.

## 강력한 형식의 제네릭 RelayCommand

파라미터를 전달받는 명령에는 제네릭 버전을 사용하면 타입 안정성을 높일 수 있습니다.

```csharp
public sealed class RelayCommand<T> : ICommand
{
    private readonly Action<T> _execute;
    private readonly Predicate<T> _canExecute;

    public RelayCommand(Action<T> execute, Predicate<T> canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }

    public bool CanExecute(object parameter)
    {
        if (_canExecute == null) return true;
        if (parameter is T typedParam) return _canExecute(typedParam);
        return false;
    }

    public void Execute(object parameter)
    {
        if (parameter is T typedParam)
            _execute(typedParam);
    }

    public event EventHandler CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }

    public void RaiseCanExecuteChanged() => CommandManager.InvalidateRequerySuggested();
}
```

이제 ViewModel에서 파라미터 타입에 맞는 명령을 정의할 수 있습니다.

```csharp
public RelayCommand<int> DeleteCommand { get; }
DeleteCommand = new RelayCommand<int>(id => DeleteCustomer(id), id => id > 0);
```

XAML에서는 `CommandParameter`로 데이터를 전달합니다.

```xml
<Button Content="삭제" Command="{Binding DeleteCommand}" CommandParameter="{Binding Id}"/>
```

## 비동기 Command의 안전한 구현

비동기 작업을 Command로 처리할 때는 `async void`를 피하고, 동시 실행을 방지하며 취소를 지원하는 구조가 필요합니다.

```csharp
public sealed class AsyncRelayCommand : ICommand
{
    private readonly Func<CancellationToken, Task> _asyncExecute;
    private readonly Func<bool> _canExecute;
    private CancellationTokenSource _cts;
    private bool _isExecuting;

    public AsyncRelayCommand(Func<CancellationToken, Task> asyncExecute, Func<bool> canExecute = null)
    {
        _asyncExecute = asyncExecute ?? throw new ArgumentNullException(nameof(asyncExecute));
        _canExecute = canExecute ?? (() => true);
    }

    public bool IsExecuting => _isExecuting;

    public bool CanExecute(object parameter) => !_isExecuting && _canExecute();

    public async void Execute(object parameter)
    {
        if (!CanExecute(parameter)) return;

        _isExecuting = true;
        RaiseCanExecuteChanged();
        _cts = new CancellationTokenSource();

        try
        {
            await _asyncExecute(_cts.Token);
        }
        catch (OperationCanceledException)
        {
            // 취소는 정상 처리
        }
        finally
        {
            _cts.Dispose();
            _cts = null;
            _isExecuting = false;
            RaiseCanExecuteChanged();
        }
    }

    public void Cancel() => _cts?.Cancel();

    public event EventHandler CanExecuteChanged;
    public void RaiseCanExecuteChanged() => CanExecuteChanged?.Invoke(this, EventArgs.Empty);
}
```

이 구현체는 실행 중에는 `CanExecute`가 `false`를 반환하여 중복 실행을 방지합니다. ViewModel에서 다음과 같이 사용합니다.

```csharp
public AsyncRelayCommand LoadDataCommand { get; }
LoadDataCommand = new AsyncRelayCommand(async ct =>
{
    await Task.Delay(2000, ct); // 실제 작업
    // 데이터 로드
});
```

## MVVM에서 Command 적용하기

ViewModel은 `INotifyPropertyChanged`를 구현하고, `RelayCommand` 타입의 프로퍼티를 노출합니다.

```csharp
public class CustomerViewModel : INotifyPropertyChanged
{
    private string _customerName;
    public string CustomerName
    {
        get => _customerName;
        set
        {
            _customerName = value;
            OnPropertyChanged();
            SaveCommand.RaiseCanExecuteChanged();
        }
    }

    public RelayCommand SaveCommand { get; }

    public CustomerViewModel()
    {
        SaveCommand = new RelayCommand(Save, CanSave);
    }

    private bool CanSave() => !string.IsNullOrWhiteSpace(CustomerName);
    private void Save() => MessageBox.Show($"저장: {CustomerName}");

    public event PropertyChangedEventHandler PropertyChanged;
    protected void OnPropertyChanged([CallerMemberName] string name = null) =>
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
}
```

XAML에서는 바인딩만 하면 됩니다.

```xml
<TextBox Text="{Binding CustomerName, UpdateSourceTrigger=PropertyChanged}" />
<Button Content="저장" Command="{Binding SaveCommand}" />
```

## Command 팩토리 패턴

대규모 애플리케이션에서는 Command 생성을 일관되게 관리하기 위해 팩토리 패턴을 도입할 수 있습니다.

```csharp
public interface ICommandFactory
{
    ICommand CreateCommand(Action execute, Func<bool> canExecute = null);
    ICommand CreateCommand<T>(Action<T> execute, Predicate<T> canExecute = null);
    IAsyncCommand CreateAsyncCommand(Func<CancellationToken, Task> execute, Func<bool> canExecute = null);
}
```

```csharp
public class CommandFactory : ICommandFactory
{
    public ICommand CreateCommand(Action execute, Func<bool> canExecute = null)
        => new RelayCommand(execute, canExecute);

    public ICommand CreateCommand<T>(Action<T> execute, Predicate<T> canExecute = null)
        => new RelayCommand<T>(execute, canExecute);

    public IAsyncCommand CreateAsyncCommand(Func<CancellationToken, Task> execute, Func<bool> canExecute = null)
        => new AsyncRelayCommand(execute, canExecute);
}
```

이렇게 하면 ViewModel에서 `ICommandFactory`를 주입받아 Command를 생성할 수 있어 테스트와 유지보수가 쉬워집니다.

## 성능과 디버깅 고려사항

| 항목 | 권장 사항 |
|------|-----------|
| `CanExecute` | 가벼운 연산으로 유지하고, 결과를 캐싱하세요. |
| `RaiseCanExecuteChanged` | 너무 자주 호출하지 마세요. 불필요한 UI 갱신을 유발할 수 있습니다. |
| 예외 처리 | `Execute` 내부에서 발생하는 예외는 적절히 처리하여 애플리케이션이 중단되지 않도록 합니다. |
| 디버깅 | Command 이름을 지정하면 로그에서 어떤 명령이 실행되었는지 추적하기 쉽습니다. |

## 결론

`ICommand`와 `RelayCommand`는 WPF에서 MVVM 패턴을 실현하는 핵심 도구입니다. 기본적인 동기 Command부터 제네릭, 비동기 Command까지 다양한 구현체를 이해하고 적절히 활용하면 UI와 비즈니스 로직을 명확히 분리할 수 있습니다. 또한 팩토리 패턴을 도입하면 대규모 프로젝트에서도 일관된 Command 관리를 할 수 있습니다.

이 패턴들을 제대로 활용하면 테스트 가능성, 유지보수성, 확장성이 뛰어난 WPF 애플리케이션을 개발할 수 있습니다.