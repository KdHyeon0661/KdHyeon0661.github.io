---
layout: post
title: WPF - Command 패턴과 바인딩
date: 2025-09-02 21:25:23 +0900
category: WPF
---
# WPF Command 패턴과 바인딩

WPF의 Command 시스템은 사용자 인터랙션을 처리하는 일관된 방법을 제공합니다. 단순히 버튼 클릭 이벤트를 처리하는 것을 넘어, 실행 가능 여부에 따른 UI 상태 자동 관리, 다양한 입력 소스(버튼, 메뉴, 단축키)의 통합 처리, MVVM 패턴에서의 관심사 분리 등 여러 장점이 있습니다. 이 글에서는 Command 패턴의 기본 개념부터 실제 프로젝트에서 활용하는 방법까지 초중급 개발자 관점에서 설명합니다.

## Command 패턴이란?

Command 패턴은 “요청”을 객체로 캡슐화하는 디자인 패턴입니다. 사용자의 액션(버튼 클릭, 메뉴 선택 등)을 명령 객체로 감싸면 호출자(UI)와 실행자(비즈니스 로직) 사이의 결합이 느슨해집니다. 이로 인해 다음과 같은 이점을 얻을 수 있습니다.

- **실행 취소/재실행(Undo/Redo)** 구현이 용이해집니다.
- 명령을 큐에 저장하거나 로깅할 수 있습니다.
- 같은 명령을 여러 UI 요소(버튼, 메뉴, 단축키)에서 재사용할 수 있습니다.

WPF는 이 패턴을 `ICommand` 인터페이스로 구현하여 프레임워크 수준에서 지원합니다.

## ICommand 인터페이스 이해하기

`ICommand`는 단 세 개의 멤버로 구성된 인터페이스입니다.

```csharp
public interface ICommand
{
    event EventHandler CanExecuteChanged;
    bool CanExecute(object parameter);
    void Execute(object parameter);
}
```

- `Execute` : 명령이 실제로 수행할 작업을 정의합니다.
- `CanExecute` : 명령이 현재 실행 가능한 상태인지 반환합니다. `false`를 반환하면 바인딩된 UI 요소(예: 버튼)는 자동으로 비활성화됩니다.
- `CanExecuteChanged` : `CanExecute`의 결과가 바뀔 때 발생하는 이벤트입니다. 이 이벤트가 발생하면 UI는 명령의 실행 가능 상태를 다시 평가합니다.

## 기본적인 Command 구현: RelayCommand

MVVM 패턴에서 가장 널리 사용되는 `RelayCommand`(또는 `DelegateCommand`)는 `ICommand`를 간단하게 구현한 헬퍼 클래스입니다.

```csharp
public class RelayCommand : ICommand
{
    private readonly Action _execute;
    private readonly Func<bool> _canExecute;

    public RelayCommand(Action execute, Func<bool> canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }

    public bool CanExecute(object parameter) => _canExecute?.Invoke() ?? true;
    public void Execute(object parameter) => _execute();

    public event EventHandler CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }
}
```

- `CommandManager.RequerySuggested`는 WPF가 전역적으로 명령의 실행 가능 상태를 다시 평가할 때 발생하는 이벤트입니다. 이벤트를 이 정적 이벤트에 연결하면 UI 상태 변경 시 자동으로 `CanExecute`가 다시 호출됩니다.

## MVVM에서 Command 사용하기

MVVM 패턴에서는 ViewModel이 `ICommand` 타입의 프로퍼티를 노출하고, View는 이 프로퍼티에 바인딩합니다.

### ViewModel 예제

```csharp
public class MainViewModel : INotifyPropertyChanged
{
    private string _userInput;
    public string UserInput
    {
        get => _userInput;
        set
        {
            _userInput = value;
            OnPropertyChanged();
            SaveCommand.RaiseCanExecuteChanged(); // 입력이 바뀌면 명령 상태 갱신
        }
    }

    public RelayCommand SaveCommand { get; }

    public MainViewModel()
    {
        SaveCommand = new RelayCommand(Save, CanSave);
    }

    private bool CanSave() => !string.IsNullOrWhiteSpace(UserInput);
    private void Save() => MessageBox.Show($"저장: {UserInput}");

    public event PropertyChangedEventHandler PropertyChanged;
    protected void OnPropertyChanged([CallerMemberName] string name = null) =>
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
}
```

### View (XAML) 바인딩

```xml
<StackPanel Margin="10">
    <TextBox Text="{Binding UserInput, UpdateSourceTrigger=PropertyChanged}"
             Margin="0,0,0,10"/>
    <Button Content="저장"
            Command="{Binding SaveCommand}"
            Padding="10,5"/>
</StackPanel>
```

- `TextBox`의 `Text`가 변경될 때마다 `UserInput` 프로퍼티가 갱신되고, `SaveCommand.RaiseCanExecuteChanged()`를 호출하여 버튼의 활성화 상태가 자동으로 업데이트됩니다.

## CommandParameter로 데이터 전달하기

Command에 추가 데이터를 전달해야 할 때는 `CommandParameter`를 사용합니다.

```xml
<ListBox ItemsSource="{Binding Customers}"
         SelectedItem="{Binding SelectedCustomer}">
    <ListBox.ItemTemplate>
        <DataTemplate>
            <StackPanel Orientation="Horizontal">
                <TextBlock Text="{Binding Name}" Width="100"/>
                <Button Content="삭제"
                        Command="{Binding DataContext.DeleteCommand, RelativeSource={RelativeSource AncestorType=Window}}"
                        CommandParameter="{Binding Id}"/>
            </StackPanel>
        </DataTemplate>
    </ListBox.ItemTemplate>
</ListBox>
```

ViewModel에서는 `RelayCommand<T>`를 사용하여 파라미터의 타입을 안전하게 전달받을 수 있습니다.

```csharp
public class RelayCommand<T> : ICommand
{
    private readonly Action<T> _execute;
    private readonly Predicate<T> _canExecute;

    public RelayCommand(Action<T> execute, Predicate<T> canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }

    public bool CanExecute(object parameter) => _canExecute?.Invoke((T)parameter) ?? true;
    public void Execute(object parameter) => _execute((T)parameter);

    public event EventHandler CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }
}
```

```csharp
// ViewModel에서
public RelayCommand<int> DeleteCommand { get; }
DeleteCommand = new RelayCommand<int>(id => DeleteCustomer(id), id => id > 0);
```

## RoutedCommand와 CommandBinding

WPF는 `RoutedCommand`를 통해 명령이 시각적 트리를 따라 전파되는 라우트 이벤트 방식도 지원합니다. `ApplicationCommands` 클래스에 미리 정의된 복사(Copy), 붙여넣기(Paste) 같은 명령이 대표적입니다.

```xml
<Window.CommandBindings>
    <CommandBinding Command="ApplicationCommands.Copy"
                    Executed="CopyCommand_Executed"
                    CanExecute="CopyCommand_CanExecute"/>
</Window.CommandBindings>

<Button Content="복사" Command="ApplicationCommands.Copy"/>
```

```csharp
private void CopyCommand_CanExecute(object sender, CanExecuteRoutedEventArgs e)
{
    e.CanExecute = textBox.SelectedText.Length > 0;
}

private void CopyCommand_Executed(object sender, ExecutedRoutedEventArgs e)
{
    textBox.Copy();
}
```

- 라우트된 명령은 특정 요소에 바인딩하지 않고도 명령을 처리할 대상을 자동으로 찾습니다. 하지만 MVVM에서는 일반적으로 `RelayCommand`를 더 많이 사용합니다.

## 비동기 Command 간단히 다루기

비동기 작업을 Command로 처리할 때는 `async void`를 사용하지 않도록 주의해야 합니다. 아래는 간단한 `AsyncRelayCommand` 예제입니다.

```csharp
public class AsyncRelayCommand : ICommand
{
    private readonly Func<Task> _execute;
    private readonly Func<bool> _canExecute;
    private bool _isExecuting;

    public AsyncRelayCommand(Func<Task> execute, Func<bool> canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }

    public bool CanExecute(object parameter) => !_isExecuting && (_canExecute?.Invoke() ?? true);
    public async void Execute(object parameter)
    {
        _isExecuting = true;
        RaiseCanExecuteChanged();
        try
        {
            await _execute();
        }
        finally
        {
            _isExecuting = false;
            RaiseCanExecuteChanged();
        }
    }

    public event EventHandler CanExecuteChanged;
    public void RaiseCanExecuteChanged() => CanExecuteChanged?.Invoke(this, EventArgs.Empty);
}
```

ViewModel에서 사용:

```csharp
public AsyncRelayCommand LoadDataCommand { get; }
LoadDataCommand = new AsyncRelayCommand(async () =>
{
    await Task.Delay(2000); // 시뮬레이션
    // 데이터 로드
});
```

## Command 패턴 활용 시 유의점

- **`CanExecute` 성능**: `CanExecute`는 자주 호출될 수 있으므로 가벼운 연산으로 유지합니다. 무거운 검증이 필요하다면 결과를 캐싱하고 적절히 갱신합니다.
- **`CommandManager.InvalidateRequerySuggested`**: 전역적으로 `CanExecute`를 다시 평가하도록 요청합니다. 너무 자주 호출하면 성능에 영향을 줄 수 있습니다.
- **예외 처리**: `Execute` 내부에서 발생하는 예외는 적절히 처리하여 애플리케이션이 중단되지 않도록 합니다.

## 결론

WPF의 Command 패턴은 UI 이벤트 처리를 구조화하고 MVVM 아키텍처에서 View와 ViewModel을 깔끔하게 분리하는 핵심 도구입니다. `ICommand` 인터페이스와 `RelayCommand` 헬퍼 클래스를 사용하면 간단하게 명령을 구현할 수 있으며, `CommandParameter`를 통해 필요한 데이터를 전달할 수 있습니다. 또한 비동기 작업이 필요한 경우 안전한 `AsyncRelayCommand`를 만들어 활용할 수 있습니다.

이 패턴을 이해하고 적절히 활용하면 유지보수성과 테스트 용이성이 높은 WPF 애플리케이션을 개발할 수 있습니다.