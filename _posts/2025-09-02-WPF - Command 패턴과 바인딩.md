---
layout: post
title: WPF - Command 패턴과 바인딩
date: 2025-09-02 21:25:23 +0900
category: WPF
---
# Command 패턴과 바인딩

WPF의 **Command**는 GoF의 **Command 패턴**을 UI에 최적화해 구현한 것입니다.
이 글에서는 **개념→WPF 커맨딩 모델→MVVM 바인딩(실전)→다양한 시나리오→베스트 프랙티스** 순서로 정리합니다.

---

## Command 패턴 한눈에 (GoF 관점)

- **목표**: “사용자 액션(요청)”을 **객체(커맨드)**로 캡슐화 → 호출자(버튼/단축키)와 수신자(실행 로직) **분리**
- **효과**: 실행 취소/재실행(undo/redo), 큐잉, 로깅, 권한 제어, 단축키 바인딩 등 **유연성** 확보
- **WPF**: `ICommand` 인터페이스로 표준화, `RoutedCommand`로 **요소 트리 전파**까지 지원

---

## WPF 커맨딩 모델 구성요소

### ICommand (기본)

```csharp
public interface ICommand {
  event EventHandler CanExecuteChanged;
  bool CanExecute(object? parameter);
  void Execute(object? parameter);
}
```
- **CanExecute**=false → 버튼 등 **자동 비활성화**
- **CanExecuteChanged** 발생 시 UI가 상태를 다시 평가

### RoutedCommand / RoutedUICommand (뷰 쪽 전통 패턴)

- **RoutedCommand**: 커맨드 실행 요청이 **요소 트리**를 따라 `CommandBinding`이 있는 곳을 찾음
- **RoutedUICommand**: 표시 문자열·제스처를 추가로 보유

```csharp
public static class MyCommands {
  public static RoutedUICommand Export { get; } =
    new RoutedUICommand("Export", "Export", typeof(MyCommands),
      new InputGestureCollection { new KeyGesture(Key.E, ModifierKeys.Control) });
}
```

XAML/코드 연결:
```xml
<Window.CommandBindings>
  <CommandBinding Command="{x:Static local:MyCommands.Export}"
                  CanExecute="OnCanExport"
                  Executed="OnExport"/>
</Window.CommandBindings>
<KeyBinding Command="{x:Static local:MyCommands.Export}" Key="E" Modifiers="Control"/>
```

```csharp
void OnCanExport(object s, CanExecuteRoutedEventArgs e) => e.CanExecute = CanExportNow();
void OnExport(object s, ExecutedRoutedEventArgs e)      => DoExport();
```

### InputBindings / CommandTarget

- **InputBindings**: 키/마우스 제스처 → 커맨드 연결 (`KeyBinding`, `MouseBinding`)
- **CommandTarget**: 커맨드의 **목표 요소** 지정(포커스가 바뀌어도 고정 대상에 실행)

```xml
<MenuItem Header="_Copy" Command="ApplicationCommands.Copy"
          CommandTarget="{Binding ElementName=EditorBox}"/>
<TextBox x:Name="EditorBox" />
```

---

## MVVM에서의 커맨드 (바인딩 중심)

MVVM에서는 **뷰모델이 ICommand 구현체**(예: `RelayCommand`)를 노출하고, **XAML에서 바인딩**합니다.

### ViewModel

```csharp
public class MainViewModel : INotifyPropertyChanged {
  private string _name = "";
  public string Name { get => _name; set { _name = value; OnChanged(); SaveCommand.RaiseCanExecuteChanged(); } }

  public RelayCommand SaveCommand { get; }
  public RelayCommand<string> DeleteCommand { get; }
  public AsyncRelayCommand LoadCommand { get; }

  public MainViewModel() {
    SaveCommand   = new RelayCommand(Save, CanSave);
    DeleteCommand = new RelayCommand<string>(Delete, id => !string.IsNullOrWhiteSpace(id));
    LoadCommand   = new AsyncRelayCommand(async ct => await LoadAsync(ct), () => true);
  }

  bool CanSave() => !string.IsNullOrWhiteSpace(Name);
  void Save() { /* 저장 */ }
  void Delete(string? id) { /* 삭제 */ }
  Task LoadAsync(CancellationToken ct) => Task.Delay(500, ct);
  // INotifyPropertyChanged 생략
}
```

### XAML

```xml
<Grid>
  <Grid.DataContext>
    <vm:MainViewModel/>
  </Grid.DataContext>

  <StackPanel Spacing="8" Margin="16">
    <TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}" Width="240"/>
    <StackPanel Orientation="Horizontal" Spacing="8">
      <Button Content="저장" Command="{Binding SaveCommand}"/>
      <Button Content="삭제" Command="{Binding DeleteCommand}" CommandParameter="{Binding Name}"/>
      <Button Content="불러오기" Command="{Binding LoadCommand}"/>
    </StackPanel>

    <!-- 단축키: Ctrl+S → SaveCommand -->
    <Grid.InputBindings>
      <KeyBinding Command="{Binding SaveCommand}" Key="S" Modifiers="Control"/>
    </Grid.InputBindings>
  </StackPanel>
</Grid>
```

**핵심 포인트**
- **Enable/Disable**는 `CanExecute`로 제어(상태 바뀌면 `RaiseCanExecuteChanged()` 호출)
- **단축키/마우스 제스처**도 동일 커맨드에 연결 가능 → 클릭/키 입력이 **하나의 로직**으로 수렴

---

## CommandParameter 바인딩 패턴

### 강타입 파라미터 (RelayCommand\<T\>)

```xml
<Button Content="삭제"
        Command="{Binding DeleteCommand}"
        CommandParameter="{Binding SelectedItem.Id, ElementName=OrderList}"/>
<ListBox x:Name="OrderList" ItemsSource="{Binding Orders}"/>
```

### 여러 값 전달 (튜플/DTO/MultiBinding)

```xml
<Button Content="이동">
  <Button.CommandParameter>
    <MultiBinding Converter="{StaticResource ToTuple}">
      <Binding Path="SelectedItem.Id" ElementName="OrderList"/>
      <Binding Path="Name"/>
    </MultiBinding>
  </Button.CommandParameter>
</Button>
```
> 또는 `RelayCommand<(int id, string name)>`처럼 **튜플**을 직접 사용할 수도 있습니다.

### DataTemplate 내부에서 상위 VM 커맨드 호출

```xml
<ListBox ItemsSource="{Binding Orders}">
  <ListBox.ItemTemplate>
    <DataTemplate>
      <Button Content="삭제"
              Command="{Binding DataContext.DeleteCommand, RelativeSource={RelativeSource AncestorType=Window}}"
              CommandParameter="{Binding Id}"/>
    </DataTemplate>
  </ListBox.ItemTemplate>
</ListBox>
```

### ContextMenu/Popup(별도 시각 트리)의 Command 바인딩

```xml
<ListBox x:Name="OrderList" ItemsSource="{Binding Orders}">
  <ListBox.ItemContainerStyle>
    <Style TargetType="ListBoxItem">
      <Setter Property="ContextMenu">
        <Setter.Value>
          <ContextMenu>
            <MenuItem Header="삭제"
                      Command="{Binding PlacementTarget.DataContext.DeleteCommand,
                                        RelativeSource={RelativeSource AncestorType=ContextMenu}}"
                      CommandParameter="{Binding PlacementTarget.SelectedItem.Id,
                                        RelativeSource={RelativeSource AncestorType=ContextMenu}}"/>
          </ContextMenu>
        </Setter.Value>
      </Setter>
    </Style>
  </ListBox.ItemContainerStyle>
</ListBox>
```
> `ContextMenu`는 시각 트리가 분리됩니다. **`PlacementTarget`**을 이용해 원래 요소의 `DataContext`/선택 항목에 접근하세요.

---

## RoutedCommand vs MVVM ICommand 비교

| 항목 | RoutedCommand (뷰 중심) | ViewModel ICommand (MVVM) |
|---|---|---|
| 실행 위치 | 요소 트리에서 `CommandBinding`으로 처리 | ViewModel 메서드에서 처리 |
| 바인딩 | `Command="{x:Static ApplicationCommands.Copy}"` | `Command="{Binding SaveCommand}"` |
| 전파 | 터널/버블 라우팅 활용 | 전파 개념 없음 |
| 단축키 | `InputBinding`에 자연스러움 | 동일 |
| 테스트 용이성 | 상대적으로 낮음(뷰 필요) | **높음**(순수 단위 테스트 가능) |
| 권장 용도 | 전역/표준 명령(복사/붙여넣기)·메뉴/도구모음 | 대부분의 앱 로직, MVVM 패턴 |

실무에서는 **MVVM ICommand가 기본**이고, **표준 편집 명령**(Copy/Paste/Undo 등)이나 **클래식 메뉴 패턴**에선 `RoutedCommand`를 혼용합니다.

---

## Event → Command (이벤트를 커맨드로)

버튼 외에도 **아무 이벤트**를 커맨드로 연결하고 싶다면 **Behaviors**를 사용합니다.

```xml
<Grid xmlns:i="http://schemas.microsoft.com/xaml/behaviors">
  <i:Interaction.Triggers>
    <i:EventTrigger EventName="SelectionChanged">
      <i:InvokeCommandAction Command="{Binding SelectionChangedCommand}"
                             PassEventArgsToCommand="True"/>
    </i:EventTrigger>
  </i:Interaction.Triggers>
  <ListBox ItemsSource="{Binding Orders}"/>
</Grid>
```

> `PassEventArgsToCommand=True`로 `CommandParameter`에 `EventArgs`가 전달됩니다(필요 시 Converter로 추출).

---

## 비동기 커맨드(Async)와 재진입 방지

긴 작업은 **UI 프리징 방지**와 **중복 실행 차단**이 필요합니다.

```csharp
public sealed class AsyncRelayCommand : ICommand {
  // (요약) _isExecuting 으로 재진입 방지, CancellationToken 지원
}
```

사용:
```csharp
public AsyncRelayCommand LoadCommand { get; }
LoadCommand = new AsyncRelayCommand(async ct => await LoadAsync(ct), () => !IsBusy);
```

> 실행 중에는 `CanExecute=false`가 되어 버튼 비활성화 → UX 안정화

---

## 베스트 프랙티스 & 체크리스트

1. **Click 핸들러 대신 Command**: 테스트/재사용/단축키 연계가 쉬움
2. **CanExecute 즉시 갱신**: 관련 속성 Setter에서 `RaiseCanExecuteChanged()` 호출
3. **메모리 누수 방지**: `CommandManager.RequerySuggested` 구독은 `WeakEventManager`로
4. **UI 타입 분리**: ViewModel 커맨드 파라미터로 **UI 요소**를 넘기지 말 것(필요 정보만 DTO/Id로)
5. **다중 파라미터**: 튜플/DTO/`MultiBinding`+Converter 활용
6. **ContextMenu/Popup**: `PlacementTarget` 패턴 숙지
7. **Undo/Redo**: Command 패턴의 장점 활용(스택으로 기록)
8. **툴킷 고려**: *CommunityToolkit.Mvvm*의 `[RelayCommand]`, `AsyncRelayCommand`로 보일러플레이트 최소화

---

## 미니 예제 (동작하는 전형 패턴)

**ViewModel**
```csharp
public class TinyVm : INotifyPropertyChanged {
  private string _text = "";
  public string Text { get => _text; set { _text = value; OnChanged(); Save.RaiseCanExecuteChanged(); } }
  public RelayCommand Save { get; }
  public TinyVm() => Save = new RelayCommand(() => Saved = $"[{Text}] saved", () => !string.IsNullOrWhiteSpace(Text));
  public string? Saved { get; private set; }
  public event PropertyChangedEventHandler? PropertyChanged;
  void OnChanged([System.Runtime.CompilerServices.CallerMemberName] string? n = null)
    => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(n));
}
```

**XAML**
```xml
<Grid>
  <Grid.DataContext><local:TinyVm/></Grid.DataContext>
  <StackPanel Margin="16" Spacing="8">
    <TextBox Text="{Binding Text, UpdateSourceTrigger=PropertyChanged}" Width="220"/>
    <Button Content="Save" Command="{Binding Save}"/>
    <TextBlock Text="{Binding Saved}"/>
    <Grid.InputBindings>
      <KeyBinding Command="{Binding Save}" Key="S" Modifiers="Control"/>
    </Grid.InputBindings>
  </StackPanel>
</Grid>
```

---

### 결론

- **Command 패턴**은 UI 트리거와 실행 로직을 느슨하게 결합해 **테스트 가능**하고 **유연한** 구조를 제공합니다.
- WPF에서는 **ICommand 바인딩**(MVVM)과 **RoutedCommand**(전통/전역 명령)를 상황에 맞게 조합하세요.
- `CommandParameter`, `InputBindings`, `ContextMenu`, `DataTemplate` 등 **다양한 바인딩 패턴**을 익히면
  클릭·키보드·마우스 제스처가 **하나의 일관된 커맨드 흐름**으로 통합되어 유지보수가 쉬워집니다.
