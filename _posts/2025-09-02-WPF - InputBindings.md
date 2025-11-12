---
layout: post
title: WPF - InputBindings
date: 2025-09-02 20:25:23 +0900
category: WPF
---
# InputBindings

## 0. 무엇이 InputBindings 인가?

- **InputBindings**는 **키/마우스 동작(제스처)**을 **Command** 실행으로 **바인딩**하는 WPF의 메커니즘입니다.
- 핵심 클래스
  - **`InputBinding`**: 추상. `Command` + `InputGesture`를 한 쌍으로 묶음
  - **`KeyBinding`**: `InputBinding`의 파생. **키보드** 제스처 처리
  - **`MouseBinding`**: `InputBinding`의 파생. **마우스** 제스처 처리
  - **`InputGesture`**: 추상. 구체적으로 **`KeyGesture`**, **`MouseGesture`**
- 바인딩은 **요소별(FrameworkElement.InputBindings)** 로컬 컬렉션에 추가합니다.  
  라우팅/명령 시스템과 결합되어, **최적의 스코프**에서 키를 묶을 수 있음.

---

## 1. 가장 간단한 예 — Ctrl+S 로 저장

### 1.1 XAML (Window 범위)
```xml
<Window x:Class="InputBindingsDemo.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="InputBindings Demo" Height="300" Width="500">
  <Window.CommandBindings>
    <CommandBinding Command="ApplicationCommands.Save"
                    CanExecute="Save_CanExecute"
                    Executed="Save_Executed"/>
  </Window.CommandBindings>

  <Window.InputBindings>
    <KeyBinding Key="S" Modifiers="Control" Command="ApplicationCommands.Save"/>
  </Window.InputBindings>

  <Grid>
    <TextBox Margin="16" Text="여기서 Ctrl+S를 눌러보세요"/>
  </Grid>
</Window>
```

### 1.2 Code-behind
```csharp
private void Save_CanExecute(object sender, CanExecuteRoutedEventArgs e) => e.CanExecute = true;

private void Save_Executed(object sender, ExecutedRoutedEventArgs e)
{
    MessageBox.Show("저장!", "Save");
}
```

- `Window.InputBindings`에 `KeyBinding`을 등록하면 **Window 트리 전체**에서 단축키가 동작합니다.
- 동일 명령에 대해 **`CommandBinding`**이 **버블 라우팅**으로 검색되어 `CanExecute/Executed`를 호출합니다.

---

## 2. MouseBinding — 마우스 제스처로 명령 실행

```xml
<Border Background="LightSteelBlue" Padding="24">
  <Border.InputBindings>
    <!-- Ctrl + 휠 클릭(중간 버튼) -->
    <MouseBinding MouseAction="MiddleClick"
                  Command="{x:Static local:EditorCommands.ToggleBookmark}"
                  CommandParameter="{Binding SelectedLine}"
                  Modifiers="Control"/>
  </Border.InputBindings>

  <TextBlock Text="Ctrl + MiddleClick = Toggle Bookmark" />
</Border>
```

```csharp
public static class EditorCommands
{
    public static readonly RoutedUICommand ToggleBookmark =
        new("Toggle Bookmark", "ToggleBookmark", typeof(EditorCommands),
            new InputGestureCollection()); // 기본 제스처는 비움
}
```

- `MouseAction`은 `LeftClick`, `RightClick`, `MiddleClick`, `WheelClick` 등.  
- 드래그/더블클릭 같은 복합 제스처는 **직접 커스텀 제스처**로 구현하는 편이 명확(후술).

---

## 3. InputBinding 동작 흐름과 스코프

1. 사용자가 **키/마우스 입력**을 발생
2. WPF 입력 시스템이 **현재 포커스 요소**에서 **Visual Tree 상위**로 탐색하며  
   **가장 가까운** `InputBindings` 컬렉션에서 **제스처 매칭**을 시도
3. 매칭 성공 → **바인딩된 `ICommand` 실행** (`CanExecute` 체크 → `Executed`)
4. `CommandBinding` 탐색은 **버블 라우팅**으로 진행  
   (현재 요소 → 부모 → … → Window → Application 레벨 등록까지)

### 스코프 설계 팁
- 특정 **컨트롤 내부에서만** 동작해야 하면: **그 컨트롤의 InputBindings**에 정의
- 화면 **어디서든** 동작해야 하면: **Window**(또는 **UserControl 루트**)에 정의
- **애플리케이션 공통 단축키**: `Application.Current.MainWindow.InputBindings` 초기화 시 추가  
  또는 **BaseWindow**/Shell에서 통합

---

## 4. KeyBinding의 모든 것

### 4.1 Key + Modifiers
```xml
<KeyBinding Command="{x:Static local:AppCommands.New}"
            Key="N" Modifiers="Control"/>

<KeyBinding Command="{x:Static local:AppCommands.Close}"
            Key="F4" Modifiers="Alt"/>

<KeyBinding Command="{x:Static local:AppCommands.Find}"
            Modifiers="Control"
            KeyGesture="Ctrl+F"/> <!-- 문자열 파서 사용 -->
```

- `KeyGesture` 문자열 파서는 **로케일의 키 이름**을 따르므로,  
  코드에서는 **`new KeyGesture(Key.F, ModifierKeys.Control)`**를 권장(다국어 안정성).

### 4.2 시스템 예약 키와 충돌
- `Alt+F4`, `Alt+Tab`, `Win` 키 조합 등 **OS 레벨** 단축키는 가로채기 불가(또는 권장하지 않음).
- `Ctrl+Alt+Del` 등 보안 시퀀스는 불가.

### 4.3 텍스트 입력 컨트롤(TextBox/RichTextBox)과의 상호작용
- TextBox는 **기본 편집 키**(예: `Ctrl+C`, `Ctrl+V`, `Ctrl+Z`, `Ctrl+X`, `Ctrl+A`, `Del`, `Backspace`, `Enter`, `Esc`, `Tab`)에 대해  
  내부에서 **미리 처리**하거나 `Handled=true`로 소비하는 경우가 많습니다.
- **내 단축키가 안 먹는다?**
  1) **Preview 단계**에서 잡기 (`PreviewKeyDown` / `PreviewTextInput` + 수동 실행), 또는
  2) **InputBindings를 상위(예: Window)에 두되**, TextBox의 기본 동작과 **충돌하지 않는 제스처** 선택, 또는
  3) `AddHandler(CommandManager.PreviewCanExecuteEvent, ..., true)`로 **handledEventsToo** 활용(고급)

### 4.4 Enter 키/ESC 키의 전형적인 처리
```xml
<TextBox x:Name="Input" AcceptsReturn="False">
  <TextBox.InputBindings>
    <KeyBinding Key="Enter" Command="{Binding SubmitCommand}"/>
    <KeyBinding Key="Escape" Command="{Binding CancelCommand}"/>
  </TextBox.InputBindings>
</TextBox>
```
- `AcceptsReturn="True"`이면 `Enter`는 **줄바꿈**이 되어, **InputBinding보다 텍스트 입력이 우선**될 수 있습니다.  
  제출은 보통 **Window 단** `KeyBinding`으로 두고, TextBox는 `AcceptsReturn=False`를 유지하는 방법이 직관적.

---

## 5. MouseBinding의 모든 것

### 5.1 MouseAction
```xml
<MouseBinding MouseAction="LeftClick" Command="{Binding SelectCommand}"/>
<MouseBinding MouseAction="RightClick" Command="{Binding ContextCommand}"/>
<MouseBinding MouseAction="WheelClick" Command="{Binding MiddleCommand}"/>
```

### 5.2 Modifiers와 조합
```xml
<MouseBinding MouseAction="LeftClick" Modifiers="Control"
              Command="{Binding AddToSelectionCommand}"/>
```

### 5.3 더블클릭/드래그 등 복합 제스처
- WPF 표준 `MouseGesture`는 **단일 클릭** 기반.  
- 더블클릭은 `MouseDoubleClick` 이벤트를 **RoutedEvent → Command**로 연결하거나  
  **커스텀 InputGesture**를 만들어 **InputBindings**에 넣는 방식(후술 “커스텀 제스처”).

---

## 6. RoutedCommand, CommandBinding, InputBinding의 관계

- **InputBinding**은 **`ICommand`**(대개 **`RoutedCommand`/`RoutedUICommand`**)와 **제스처**를 매핑.
- **CommandBinding**은 해당 명령의 **실행 로직(Executed)**과 **활성화 판단(CanExecute)**을 제공.
- 실행 시퀀스:
  1) 입력 발생 → InputBinding 매칭 → `command.Execute(parameter)`
  2) 프레임워크가 **CommandBinding**을 **버블 라우팅**으로 찾음
  3) `CanExecute` → `Executed` 호출

### 6.1 RoutedUICommand + 기본 제스처
```csharp
public static class AppCommands
{
    public static readonly RoutedUICommand Open =
        new("Open", "Open", typeof(AppCommands),
            new InputGestureCollection { new KeyGesture(Key.O, ModifierKeys.Control) });

    public static readonly RoutedUICommand SaveAs =
        new("Save As", "SaveAs", typeof(AppCommands),
            new InputGestureCollection { new KeyGesture(Key.S, ModifierKeys.Control | ModifierKeys.Shift) });
}
```

```xml
<Window.InputBindings>
  <!-- 명령이 기본 제스처를 이미 가지고 있더라도
       원하는 스코프에 다시 선언해주는 것이 명확 -->
  <KeyBinding Command="{x:Static local:AppCommands.Open}"   Key="O" Modifiers="Control"/>
  <KeyBinding Command="{x:Static local:AppCommands.SaveAs}" Key="S" Modifiers="Control,Shift"/>
</Window.InputBindings>

<Window.CommandBindings>
  <CommandBinding Command="{x:Static local:AppCommands.Open}"   Executed="Open_Executed"   CanExecute="AlwaysTrue"/>
  <CommandBinding Command="{x:Static local:AppCommands.SaveAs}" Executed="SaveAs_Executed" CanExecute="AlwaysTrue"/>
</Window.CommandBindings>
```

---

## 7. 우선순위/충돌 규칙

- **가장 가까운 요소**의 `InputBindings`가 우선(포커스 기준).  
  예) TextBox 내부 `InputBindings` > Grid의 `InputBindings` > Window의 `InputBindings`.
- 한 제스처에 **여러 InputBinding**이 매칭되면: **가까운 요소의 첫 매칭**이 승리.
- `Handled=true`를 일찍 설정하는 코드(특히 Preview 단계 핸들러)는 **InputBinding을 차단**할 수 있습니다.
- **메뉴(AccessKey)**, **TextBox 기본 단축키**, **스핀/스마트 컨트롤**의 내부 처리와의 **충돌**에 주의.

---

## 8. Preview 단계와 InputBindings

- 표준 `InputBindings` 매칭은 **KeyDown/MouseDown** 처리의 **일반(버블)** 라우트에서 이뤄지는 것으로 이해하면 쉽습니다.
- 상위 요소의 `PreviewKeyDown/PreviewMouseDown`에서 `Handled=true`를 설정하면 **InputBinding이 실행되지 않을 수 있음**.
- 반대로, 내 단축키를 **무조건** 먼저 처리하려면 `PreviewKeyDown`에서 직접 `Command.Execute`를 호출하거나,  
  **커스텀 InputProvider** 수준의 확장을 고려(매우 고급).

예: 글로벌 ESC로 다이얼로그 닫기(Preview에서 흡수)
```csharp
protected override void OnPreviewKeyDown(KeyEventArgs e)
{
    if (e.Key == Key.Escape && IsDialogOpen)
    {
        CloseDialog();
        e.Handled = true; // 다른 키바인딩/입력으로 내려가지 않음
        return;
    }
    base.OnPreviewKeyDown(e);
}
```

---

## 9. MVVM에서의 “깨끗한” InputBindings

### 9.1 ViewModel에 ICommand 공개 + Window에서 바인딩
```xml
<Window.InputBindings>
  <KeyBinding Key="N" Modifiers="Control" Command="{Binding NewItemCommand}"/>
  <KeyBinding Key="Delete" Command="{Binding RemoveSelectedCommand}"/>
</Window.InputBindings>
```

- **좋은 점**: Code-behind가 필요 없음, 테스트 용이.
- **주의**: TextBox의 기본 키와 겹치지 않도록, 또는 Preview 정책을 명확히.

### 9.2 Attached Behavior/Blend Behavior로 선언적 구성
```xml
<i:Interaction.Triggers>
  <i:KeyTrigger Key="Enter">
    <i:InvokeCommandAction Command="{Binding SubmitCommand}" />
  </i:KeyTrigger>
</i:Interaction.Triggers>
```
- Blend SDK(Behaviors) 사용 시 **XAML만으로** 입력→명령 연결 가능.  
- 순수 WPF 내장만으로도 충분하지만, Behaviors는 **복합 제스처/조건** 조합이 용이.

---

## 10. 커스텀 제스처(InputGesture) 만들기

### 10.1 예: 더블클릭을 명령으로
```csharp
public sealed class DoubleClickGesture : MouseGesture
{
    public DoubleClickGesture(ModifierKeys modifiers = ModifierKeys.None)
        : base(MouseAction.LeftDoubleClick, modifiers) {}
}
```

```xml
<Window.InputBindings>
  <MouseBinding Command="{Binding OpenDetailsCommand}">
    <MouseBinding.Gesture>
      <local:DoubleClickGesture />
    </MouseBinding.Gesture>
  </MouseBinding>
</Window.InputBindings>
```

- 사실 WPF의 `MouseAction` enum에는 **LeftDoubleClick**가 **내장**되어 있습니다.  
  (버전에 따라 다를 수 있으니, 없는 경우 위와 유사한 커스텀 구현으로 대체합니다)

### 10.2 예: “Ctrl+휠 위로” → 확대
```csharp
public sealed class WheelUpGesture : InputGesture
{
    public ModifierKeys Modifiers { get; }
    public WheelUpGesture(ModifierKeys modifiers = ModifierKeys.Control) => Modifiers = modifiers;

    public override bool Matches(object targetElement, InputEventArgs inputEventArgs)
    {
        if (inputEventArgs is MouseWheelEventArgs mw &&
            mw.Delta > 0 &&
            (Keyboard.Modifiers & Modifiers) == Modifiers)
            return true;
        return false;
    }
}
```

```xml
<Window.InputBindings>
  <InputBinding Command="{Binding ZoomInCommand}">
    <InputBinding.Gesture>
      <local:WheelUpGesture />
    </InputBinding.Gesture>
  </InputBinding>
</Window.InputBindings>
```

### 10.3 예: “멀티스텝” 제스처(Up → Up → Down 같은 시퀀스)
- `InputGesture` 내부에서 **상태 머신**을 갖고, 일정 시간 내 순서를 만족하면 `true`.  
- 또는 **PreviewKeyDown**에서 상태를 관리하고, 완성 시 `Command.Execute`를 호출.  
- 실전에서는 **복잡 제스처는 전용 핸들러/서비스**로 관리하고, **View에서는 단일 명령**에 연결하는 것이 유지보수에 유리.

---

## 11. 메뉴/툴바와 InputBindings의 관계

- **MenuItem.InputGestureText** 속성으로 **단축키 안내 텍스트**만 표시할 수 있습니다.
- 실제 동작은 **InputBindings** 또는 **Command(기본 제스처)** 가 처리합니다.

```xml
<MenuItem Header="_Open"
          Command="{x:Static local:AppCommands.Open}"
          InputGestureText="Ctrl+O"/>
```

- **권장**: “메뉴의 명령”과 “InputBindings의 제스처”를 **같은 RoutedCommand**로 통일.  
  → UI 어디서나 **동일 로직** 실행, 메뉴엔 **표시만**.

---

## 12. 포커스/탭 순서/액세스 키(AccessText)와의 상호작용

- **Alt+문자** 액세스 키는 **메뉴/버튼**에 우선 적용될 수 있습니다.  
- **TextBox**에서 Alt 조합은 보통 사용되지 않으므로 충돌 적음.  
- **Tab**/`Shift+Tab`은 포커스 이동의 기본키 → 단축키로 쓰고 싶다면 **명확한 컨텍스트**에서만.

---

## 13. “왜 단축키가 안 먹죠?” 트러블슈팅 체크리스트

1. **포커스가 어디에 있나?**  
   - 포커스 요소의 **InputBindings**가 우선.  
   - 포커스가 **Popup/Adorner** 내부 등 **별도 트리**에 있을 수도 있음.
2. **Preview 단계에서 `Handled=true` 되었나?**  
   - 상위/형제 코드가 **차단**했을 수 있음.
3. **TextBox 등에서 기본 동작과 충돌?**  
   - `Ctrl+C/V/Z/A` 등은 내부가 선점.  
   - 제스처 바꾸거나 **PreviewKeyDown**에서 직접 처리.
4. **메뉴/AccessKey와 충돌?**  
   - Alt 조합, F 키 등 메뉴가 먼저 가져갈 수 있음.
5. **OS 예약키?**  
   - Alt+Tab, Win 키 등은 불가.
6. **CommandBinding이 실제로 발견되나?**  
   - `Executed`가 안 불리면 **버블 상위에 CommandBinding**이 없는 것.  
   - Window/루트에 선언했는지 확인.
7. **다국어 키보드/IME 상태**  
   - Key 값이 바뀌기도 함. Gesture를 **KeyGesture(Key.X, …)** 식으로 지정.

---

## 14. 대규모 화면에서의 관리 전략

- **중앙 등록 서비스**: Shell/Window가 시작 시 공통 제스처를 등록.  
- **기능 모듈별 레지스트리**: 각 모듈 View가 로드될 때 **필요한 InputBindings만 추가/제거**.  
- **조건부 활성화**: 특정 상태에서만 `CanExecute=true` 되도록 분기 → **사용자 혼동 감소**.  
- **명령 이름/제스처 표준화**: 팀 규칙 문서화 (“검색은 항상 Ctrl+F” 등).

---

## 15. 디버깅/프로파일링 팁

- **Snoop / Live Visual Tree**로 포커스/트리 확인.  
- `InputManager.PreProcessInput` 이벤트 구독하여 **입력 파이프라인** 로깅:
```csharp
InputManager.Current.PreProcessInput += (s, e) =>
{
    if (e.StagingItem?.Input is KeyEventArgs ke)
        Debug.WriteLine($"PreProcess: {ke.RoutedEvent.Name} Key={ke.Key} IsDown={ke.IsDown}");
};
```
- `CommandManager.PreviewCanExecute`/`PreviewExecuted` 후킹도 가능(고급).  
- XAML 해석 오류나 Gesture 구문 오류는 **Output 창**에 경고/예외로 나타남.

---

## 16. 예제: “편집 화면” 종합 샘플 (MVVM, TextBox 충돌 회피)

### 16.1 ViewModel
```csharp
public class EditorViewModel
{
    public ICommand SaveCommand { get; }
    public ICommand SaveAsCommand { get; }
    public ICommand FindCommand { get; }
    public ICommand CloseCommand { get; }
    public ICommand ToggleCommentCommand { get; }
    public ICommand ZoomInCommand { get; }
    public ICommand ZoomOutCommand { get; }

    public EditorViewModel()
    {
        SaveCommand           = new RelayCommand(_ => Save(), _ => CanSave());
        SaveAsCommand         = new RelayCommand(_ => SaveAs());
        FindCommand           = new RelayCommand(_ => Find());
        CloseCommand          = new RelayCommand(_ => Close());
        ToggleCommentCommand  = new RelayCommand(_ => ToggleComment(), _ => !IsReadOnly);
        ZoomInCommand         = new RelayCommand(_ => Zoom(+1));
        ZoomOutCommand        = new RelayCommand(_ => Zoom(-1));
    }

    bool CanSave() => true;
    void Save() {} void SaveAs() {} void Find() {} void Close() {}
    bool IsReadOnly => false;
    void ToggleComment() {}
    void Zoom(int delta) {}
}
```

### 16.2 View (Window)
```xml
<Window x:Class="InputBindingsDemo.EditorWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:local="clr-namespace:InputBindingsDemo"
        Title="Editor" Width="800" Height="600">

  <Window.InputBindings>
    <!-- 앱 전역/편집 공통 -->
    <KeyBinding Key="S" Modifiers="Control" Command="{Binding SaveCommand}"/>
    <KeyBinding Key="S" Modifiers="Control,Shift" Command="{Binding SaveAsCommand}"/>
    <KeyBinding Key="F" Modifiers="Control" Command="{Binding FindCommand}"/>
    <KeyBinding Key="W" Modifiers="Control" Command="{Binding CloseCommand}"/>
    <!-- 텍스트 충돌 회피: ToggleComment는 Ctrl+/ 처럼 충돌 적은 키로 -->
    <KeyBinding Modifiers="Control" KeyGesture="Ctrl+Divide" Command="{Binding ToggleCommentCommand}"/>
  </Window.InputBindings>

  <Grid>
    <Grid.RowDefinitions>
      <RowDefinition Height="Auto"/>
      <RowDefinition/>
      <RowDefinition Height="Auto"/>
    </Grid.RowDefinitions>

    <ToolBarTray Grid.Row="0">
      <ToolBar>
        <Button Command="{Binding SaveCommand}" Content="Save (Ctrl+S)"/>
        <Button Command="{Binding SaveAsCommand}" Content="Save As (Ctrl+Shift+S)"/>
        <Button Command="{Binding FindCommand}" Content="Find (Ctrl+F)"/>
      </ToolBar>
    </ToolBarTray>

    <ScrollViewer Grid.Row="1">
      <TextBox x:Name="Editor"
               AcceptsReturn="True"
               AcceptsTab="True"
               FontFamily="Consolas" FontSize="14"
               TextWrapping="NoWrap" VerticalScrollBarVisibility="Auto"
               HorizontalScrollBarVisibility="Auto"
               Text="{Binding DocumentText, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}">
        <TextBox.InputBindings>
          <!-- 편집 전용: Ctrl+Wheel = Zoom -->
          <InputBinding Command="{Binding ZoomInCommand}">
            <InputBinding.Gesture>
              <local:WheelUpGesture />
            </InputBinding.Gesture>
          </InputBinding>
          <InputBinding Command="{Binding ZoomOutCommand}">
            <InputBinding.Gesture>
              <local:WheelUpGesture Modifiers="Control"/> <!-- 예시: Up만 + Ctrl -->
            </InputBinding.Gesture>
          </InputBinding>
        </TextBox.InputBindings>
      </TextBox>
    </ScrollViewer>

    <StatusBar Grid.Row="2">
      <StatusBarItem Content="Ctrl+S Save, Ctrl+F Find, Ctrl+/ Toggle Comment"/>
    </StatusBar>
  </Grid>
</Window>
```

- 저장/찾기 등 **전역 단축키는 Window 레벨**에 두고,  
- **편집 전용 제스처(휠 확대/축소)**는 **TextBox 스코프**에 둬서 **문맥성**을 반영.

---

## 17. 스타일/리소스로 재사용 가능한 InputBindings 패턴

### 17.1 스타일에 포함(특정 Control 공통 키)
```xml
<Style TargetType="DataGrid" x:Key="EditableGridKeys">
  <Setter Property="InputBindings">
    <Setter.Value>
      <InputBindingCollection>
        <KeyBinding Key="F2" Command="{Binding EditCellCommand}"/>
        <KeyBinding Key="Delete" Command="{Binding DeleteRowCommand}"/>
      </InputBindingCollection>
    </Setter.Value>
  </Setter>
</Style>
```

### 17.2 리소스 딕셔너리 분리
- `Keys.xaml`에 InputBinding 컬렉션을 정의한 뒤, 필요한 화면에서 **머지**.  
- 팀 단축키 가이드에 맞춰 **일관성 유지**.

---

## 18. 보안/접근성/국제화 관점

- **접근성(Accessibility)**: 키보드만으로도 모든 기능 접근 가능하도록 **단축키 제공**.  
  화면 리더 사용자는 `Alt` 메뉴/AccessKey를 주로 사용.
- **국제화**: `KeyGesture` 문자열은 로케일 영향을 받음 → 코드에서 `KeyGesture(Key, Modifiers)` 권장.  
- **보안**: 시스템 예약키 가로채기는 사용자 기대를 깨뜨림. **표준 관례** 유지.

---

## 19. 테스트 자동화

- **UI 자동화**: Appium/WinAppDriver, FlaUI 등으로 키 입력 시뮬레이션.  
- **단위 테스트**: `ICommand.CanExecute/Execute` 로직을 **ViewModel 단위**로 테스트.  
- **스모크 테스트**: 주요 화면에 대해 **핫키 목록**을 순회하며 `CanExecute`가 true인지 검증.

---

## 20. 요약 체크리스트

- [ ] **InputBindings**는 **요소 스코프**로 작동. **포커스**가 중요  
- [ ] **KeyBinding/MouseBinding** → **RoutedCommand/CommandBinding**과 결합  
- [ ] **TextBox 기본 단축키**와 충돌 주의(필요 시 Preview/제스처 교체)  
- [ ] **스코프 설계**: 컨트롤-로컬 vs Window-전역  
- [ ] **충돌/우선순위**: 가까운 요소가 우선, `Handled` 주의  
- [ ] **커스텀 제스처**: `InputGesture.Matches`로 무엇이든 가능(더블클릭/휠/시퀀스)  
- [ ] **MVVM 친화**: ViewModel의 ICommand에 직접 바인딩  
- [ ] **디버깅**: `InputManager.PreProcessInput`/Snoop/Live Visual Tree

---

## 21. 부록: 미니 레퍼런스

### 21.1 Key enum 예시
- 문자/숫자: `A`~`Z`, `D0`~`D9` (상단 숫자), `NumPad0`~`NumPad9`
- 기능키: `F1`~`F24`
- 편집키: `Enter`, `Escape`, `Back`, `Delete`, `Insert`, `Tab`, `Space`
- 방향키: `Left`, `Right`, `Up`, `Down`
- 특수: `Home`, `End`, `PageUp`, `PageDown`, `Scroll`, `Pause`, `PrintScreen`

### 21.2 Modifiers
- `Control`, `Alt`, `Shift`, `Windows`  
  *(Windows 키는 조합 동작이 제한적이며, 사용자 기대와 충돌하므로 권장하지 않음)*

### 21.3 MouseAction
- `LeftClick`, `RightClick`, `MiddleClick`, `WheelClick`  
- (버전에 따라) `LeftDoubleClick` 등 제공. 없으면 커스텀 구현.

---

## 22. 자주 하는 실수와 해법

- **실수**: “Window에 키를 묶었는데 TextBox 포커스일 땐 안 된다”  
  **해법**: TextBox 기본 편집 키와 충돌하는지 확인. 다른 제스처 사용, PreviewKeyDown에서 수동 처리, 또는 포커스 이동(Enter 시 `MoveFocus` 등).
- **실수**: “Executed가 호출되지 않는다”  
  **해법**: **CommandBinding이 버블 경로** 어딘가에 있는지 확인. UserControl 내부에만 두면 상위 Window 키에서 못 찾음.
- **실수**: “두 개 화면에 같은 Ctrl+S를 배치했더니 서로 간섭”  
  **해법**: **스코프 분리**. 각각 해당 뷰가 활성일 때만 InputBindings 추가/제거.

---

## 23. 실전 템플릿(요약) — 프로젝트에 바로 붙여 쓰는 코드

### 23.1 공통 명령 정의
```csharp
public static class CommonCommands
{
    public static readonly RoutedUICommand Save   = new("Save", "Save", typeof(CommonCommands),
        new InputGestureCollection { new KeyGesture(Key.S, ModifierKeys.Control) });

    public static readonly RoutedUICommand Find   = new("Find", "Find", typeof(CommonCommands),
        new InputGestureCollection { new KeyGesture(Key.F, ModifierKeys.Control) });

    public static readonly RoutedUICommand Close  = new("Close", "Close", typeof(CommonCommands),
        new InputGestureCollection { new KeyGesture(Key.W, ModifierKeys.Control) });
}
```

### 23.2 Shell(메인 윈도우)
```xml
<Window ...>
  <Window.InputBindings>
    <KeyBinding Command="{x:Static local:CommonCommands.Save}"  Key="S" Modifiers="Control"/>
    <KeyBinding Command="{x:Static local:CommonCommands.Find}"  Key="F" Modifiers="Control"/>
    <KeyBinding Command="{x:Static local:CommonCommands.Close}" Key="W" Modifiers="Control"/>
  </Window.InputBindings>

  <Window.CommandBindings>
    <CommandBinding Command="{x:Static local:CommonCommands.Save}"  Executed="Save_Executed"  CanExecute="AlwaysTrue"/>
    <CommandBinding Command="{x:Static local:CommonCommands.Find}"  Executed="Find_Executed"  CanExecute="AlwaysTrue"/>
    <CommandBinding Command="{x:Static local:CommonCommands.Close}" Executed="Close_Executed" CanExecute="AlwaysTrue"/>
  </Window.CommandBindings>
  ...
</Window>
```

### 23.3 사용자 컨트롤(로컬 제스처)
```xml
<UserControl ...>
  <UserControl.InputBindings>
    <KeyBinding Key="F2" Command="{Binding RenameCommand}"/>
    <MouseBinding MouseAction="RightClick" Command="{Binding ShowContextCommand}"/>
  </UserControl.InputBindings>
  ...
</UserControl>
```

---

## 24. 마무리

**InputBindings**는 “**사용자 입력 → 명령 실행**”을 **선언적**으로 연결하는 WPF의 핵심 도구입니다.  
적절한 **스코프 설계**, **텍스트 입력 컨트롤과의 충돌 회피**, **커스텀 제스처**로 복잡한 요구도 깔끔하게 처리할 수 있습니다.  
MVVM과 결합하면 **테스트 가능한 입력 설계**가 가능하며, 대규모 앱에서도 **일관된 단축키 경험**을 제공할 수 있습니다.