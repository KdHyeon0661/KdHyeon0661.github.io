---
layout: post
title: WPF - InputBindings
date: 2025-09-02 20:25:23 +0900
category: WPF
---
# InputBindings

## 무엇이 InputBindings 인가?

**InputBindings**는 키보드나 마우스 동작을 특정 명령(Command)에 연결하는 WPF의 선언적 메커니즘입니다. 사용자가 정해진 키 조합이나 마우스 제스처를 수행하면, 바인딩된 명령이 실행되는 방식으로 동작합니다.

핵심 구성 요소는 다음과 같습니다:
- **`InputBinding`**: 명령과 입력 제스처를 묶는 기본 추상 클래스입니다.
- **`KeyBinding`**: 키보드 입력을 처리하는 `InputBinding`의 구체적 구현입니다.
- **`MouseBinding`**: 마우스 입력을 처리하는 `InputBinding`의 구체적 구현입니다.
- **`InputGesture`**: 실제 입력 동작을 표현하는 추상 클래스로, `KeyGesture`와 `MouseGesture`가 대표적입니다.

이 바인딩은 각 UI 요소의 `InputBindings` 컬렉션에 추가되어, 해당 요소의 포커스 범위 내에서 유효하게 동작합니다. WPF의 라우팅 명령 시스템과 결합되어, 애플리케이션 전역부터 특정 컨트롤 내부까지 유연한 스코프 설정이 가능합니다.

---

## 기본 동작 원리와 간단한 예제

InputBindings의 가장 일반적인 사용 예는 `Ctrl+S` 단축키로 저장 기능을 실행하는 것입니다.

**XAML (Window 범위)**
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
    <TextBox Margin="16" Text="Ctrl+S를 눌러 저장 기능을 실행해보세요."/>
  </Grid>
</Window>
```

**Code-behind**
```csharp
private void Save_CanExecute(object sender, CanExecuteRoutedEventArgs e) => e.CanExecute = true;

private void Save_Executed(object sender, ExecutedRoutedEventArgs e)
{
    MessageBox.Show("저장되었습니다!", "Save");
}
```
`Window.InputBindings`에 등록된 단축키는 해당 윈도우의 전체 시각적 트리에서 동작합니다. 명령이 실행될 때, WPF는 버블 라우팅 방식을 통해 `CommandBinding`을 찾아 `CanExecute`와 `Executed` 핸들러를 호출합니다.

---

## MouseBinding을 활용한 마우스 제스처 명령

마우스 클릭이나 휠 동작도 명령에 바인딩할 수 있습니다.

```xml
<Border Background="LightSteelBlue" Padding="24">
  <Border.InputBindings>
    <!-- Ctrl + 마우스 휠 클릭으로 북마크 토글 -->
    <MouseBinding MouseAction="MiddleClick"
                  Command="{x:Static local:EditorCommands.ToggleBookmark}"
                  CommandParameter="{Binding SelectedLine}"
                  Modifiers="Control"/>
  </Border.InputBindings>

  <TextBlock Text="Ctrl + 마우스 휠 클릭으로 북마크를 토글하세요." />
</Border>
```

`MouseAction` 속성으로 `LeftClick`, `RightClick`, `MiddleClick`, `WheelClick` 등의 기본 동작을 지정할 수 있습니다. 더블클릭이나 드래그 같은 복합 제스처는 커스텀 제스처를 구현하여 처리할 수 있습니다.

---

## InputBinding의 동작 흐름과 스코프 관리

InputBindings의 실행 흐름은 다음과 같습니다:
1. 사용자가 키보드나 마우스 입력을 발생시킵니다.
2. WPF 입력 시스템이 현재 포커스가 있는 요소부터 시작해 시각적 트리를 상향으로 탐색하며, 가장 가까운 `InputBindings` 컬렉션에서 매칭되는 제스처를 찾습니다.
3. 제스처가 매칭되면 연결된 `ICommand`의 `Execute` 메서드가 호출됩니다.
4. 실제 명령 실행 로직은 버블 라우팅을 통해 해당 명령에 대한 `CommandBinding`을 찾아 수행됩니다.

**스코프 설계 가이드라인**
- 특정 컨트롤 내부에서만 동작해야 하는 단축키는 해당 컨트롤의 `InputBindings`에 정의합니다.
- 화면 어디서나 동작해야 하는 전역 단축키는 `Window`나 `UserControl`의 루트 수준에 정의합니다.
- 애플리케이션 공통 단축키는 메인 윈도우의 `InputBindings`에 초기화 시 추가하거나 기본 창 클래스에서 통합 관리합니다.

---

## KeyBinding의 상세 활용

**기본 키 조합 정의**
```xml
<KeyBinding Command="{x:Static local:AppCommands.New}"
            Key="N" Modifiers="Control"/>

<KeyBinding Command="{x:Static local:AppCommands.Close}"
            Key="F4" Modifiers="Alt"/>

<KeyBinding Command="{x:Static local:AppCommands.Find}"
            KeyGesture="Ctrl+F"/>
```

**시스템 예약 키와의 충돌 주의사항**
`Alt+F4`, `Alt+Tab`, `Win` 키 조합 등 운영체제 수준의 예약 단축키는 가로채지 않는 것이 원칙입니다. 특히 보안과 관련된 `Ctrl+Alt+Del` 같은 시퀀스는 완전히 차단됩니다.

**TextBox와의 상호작용 문제 해결**
TextBox는 `Ctrl+C`, `Ctrl+V`, `Ctrl+Z` 같은 기본 편집 키를 내부적으로 처리합니다. 단축키가 동작하지 않는다면:
1. `PreviewKeyDown` 이벤트에서 수동으로 명령을 실행하거나
2. 충돌하지 않는 다른 제스처를 선택하거나
3. `AddHandler`에 `handledEventsToo` 파라미터를 `true`로 설정하여 처리된 이벤트도 수신합니다.

**Enter 키와 Escape 키의 전형적 처리**
```xml
<TextBox x:Name="Input" AcceptsReturn="False">
  <TextBox.InputBindings>
    <KeyBinding Key="Enter" Command="{Binding SubmitCommand}"/>
    <KeyBinding Key="Escape" Command="{Binding CancelCommand}"/>
  </TextBox.InputBindings>
</TextBox>
```
`AcceptsReturn="True"`로 설정된 TextBox에서는 Enter 키가 줄바꿈으로 우선 처리될 수 있습니다. 이 경우 Window 수준에서 단축키를 정의하고 TextBox의 `AcceptsReturn`을 `False`로 유지하는 것이 더 직관적일 수 있습니다.

---

## MouseBinding의 확장 기능

**기본 마우스 동작 정의**
```xml
<MouseBinding MouseAction="LeftClick" Command="{Binding SelectCommand}"/>
<MouseBinding MouseAction="RightClick" Command="{Binding ContextCommand}"/>
<MouseBinding MouseAction="MiddleClick" Command="{Binding MiddleCommand}"/>
```

**수정자(Modifier)와의 조합**
```xml
<MouseBinding MouseAction="LeftClick" Modifiers="Control"
              Command="{Binding AddToSelectionCommand}"/>
```

**복합 제스처 처리**
WPF의 표준 `MouseGesture`는 단일 클릭 기반입니다. 더블클릭이나 드래그 같은 복합 제스처를 처리하려면:
- `MouseDoubleClick` 이벤트를 명령에 연결하거나
- 커스텀 `InputGesture`를 구현하여 `InputBindings`에 추가합니다.

---

## RoutedCommand, CommandBinding, InputBinding의 관계

이 세 요소는 WPF 명령 시스템의 핵심을 이루며 다음과 같이 상호작용합니다:
- **InputBinding**은 물리적 입력을 논리적 명령(`ICommand`)에 매핑합니다.
- **CommandBinding**은 해당 명령의 실행 로직(`Executed`)과 활성화 조건(`CanExecute`)을 제공합니다.
- **RoutedCommand**는 명령 실행 시 버블 라우팅을 통해 적절한 `CommandBinding`을 찾아 실행합니다.

**실행 시퀀스**
1. 사용자 입력 발생 → InputBinding에서 매칭되는 제스처 발견 → 연결된 명령의 `Execute` 호출
2. WPF가 버블 라우팅 방식으로 해당 명령에 대한 `CommandBinding`을 탐색
3. `CanExecute` 메서드로 실행 가능성 확인 → `Executed` 핸들러 실행

---

## 우선순위와 충돌 해결 규칙

InputBindings의 충돌 해결은 다음과 같은 규칙을 따릅니다:
- **근접성 우선**: 현재 포커스가 있는 요소에 가까운 `InputBindings`가 먼저 검사됩니다.
- **처리 상태 차단**: 상위 요소의 `PreviewKeyDown`이나 `PreviewMouseDown` 핸들러에서 `Handled=true`로 설정하면 하위 InputBinding이 실행되지 않을 수 있습니다.
- **기본 동작과의 충돌**: TextBox의 내부 편집 단축키나 메뉴의 AccessKey와 충돌할 경우 예상치 못한 동작이 발생할 수 있습니다.

**충돌 예방 전략**
- 애플리케이션 전역 단축키는 가능한 Window 수준에서 정의합니다.
- 특수 컨트롤의 기본 동작을 이해하고 충돌하지 않는 제스처를 선택합니다.
- Preview 이벤트에서의 `Handled` 설정을 신중하게 관리합니다.

---

## MVVM 패턴에서의 InputBindings 활용

**ViewModel의 명령에 직접 바인딩**
```xml
<Window.InputBindings>
  <KeyBinding Key="N" Modifiers="Control" Command="{Binding NewItemCommand}"/>
  <KeyBinding Key="Delete" Command="{Binding RemoveSelectedCommand}"/>
</Window.InputBindings>
```
이 방식은 Code-behind를 최소화하고 테스트 용이성을 높입니다. 단, TextBox 등의 기본 키 동작과 충돌하지 않도록 주의가 필요합니다.

**Attached Behavior 활용**
Blend SDK의 Behavior를 사용하면 XAML 선언만으로 입력을 명령에 연결할 수 있습니다.
```xml
<i:Interaction.Triggers>
  <i:KeyTrigger Key="Enter">
    <i:InvokeCommandAction Command="{Binding SubmitCommand}" />
  </i:KeyTrigger>
</i:Interaction.Triggers>
```

---

## 커스텀 InputGesture 구현

**더블클릭 제스처 구현**
```csharp
public sealed class DoubleClickGesture : MouseGesture
{
    public DoubleClickGesture(ModifierKeys modifiers = ModifierKeys.None)
        : base(MouseAction.LeftDoubleClick, modifiers) {}
}
```

**마우스 휠 업/다운 제스처 구현**
```csharp
public sealed class WheelUpGesture : InputGesture
{
    public ModifierKeys Modifiers { get; }
    public WheelUpGesture(ModifierKeys modifiers = ModifierKeys.Control) => Modifiers = modifiers;

    public override bool Matches(object targetElement, InputEventArgs inputEventArgs)
    {
        return inputEventArgs is MouseWheelEventArgs mw &&
               mw.Delta > 0 &&
               (Keyboard.Modifiers & Modifiers) == Modifiers;
    }
}
```
이러한 커스텀 제스처를 통해 복잡한 입력 패턴도 명령 시스템에 통합할 수 있습니다.

---

## 디버깅과 문제 해결 가이드

InputBindings 관련 문제 발생 시 다음 사항을 점검해보세요:

1. **포커스 위치 확인**: 현재 포커스가 의도한 요소에 있는지 Snoop이나 Live Visual Tree 도구로 확인합니다.
2. **이벤트 처리 상태 검사**: Preview 이벤트 핸들러에서 `Handled=true`가 설정되지 않았는지 확인합니다.
3. **기본 동작 충돌 분석**: TextBox, RichTextBox 등의 내부 단축키와 충돌하는지 검토합니다.
4. **명령 바인딩 존재 확인**: 해당 명령에 대한 `CommandBinding`이 버블 라우팅 경로 상에 존재하는지 확인합니다.
5. **시스템 예약 키 확인**: 사용한 제스처가 운영체제 수준에서 예약된 키는 아닌지 검토합니다.

---

## 실전 적용 패턴

**대규모 애플리케이션에서의 관리 전략**
- **중앙 등록 서비스**: 공통 단축키를 Shell/메인 윈도우에서 통합 관리합니다.
- **모듈별 등록**: 각 기능 모듈이 로드될 때 필요한 InputBindings만 동적으로 추가/제거합니다.
- **조건부 활성화**: 특정 컨텍스트에서만 단축키가 활성화되도록 `CanExecute` 로직을 구현합니다.
- **표준화**: 팀 내 단축키 규칙을 문서화하여 일관된 사용자 경험을 제공합니다.

**재사용 가능한 스타일 정의**
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

---

## 결론

InputBindings는 WPF에서 사용자 입력을 애플리케이션 명령으로 변환하는 강력한 메커니즘입니다. 선언적 접근 방식을 통해 XAML에서 직관적으로 단축키를 정의할 수 있으며, WPF의 라우팅 시스템과 결합되어 유연한 스코프 관리가 가능합니다.

효과적인 InputBindings 설계를 위해서는:
- **적절한 스코프 선택**: 컨트롤 수준, 윈도우 수준, 애플리케이션 수준의 적절한 범위를 선택합니다.
- **충돌 관리**: 기본 컨트롤 동작, 메뉴 시스템, 운영체제 예약 키와의 충돌을 사전에 방지합니다.
- **MVVM 패턴 통합**: ViewModel의 명령에 직접 바인딩하여 테스트 가능하고 유지보수성 높은 코드를 작성합니다.
- **사용자 경험 고려**: 일관된 단축키 체계를 설계하고, 필요시 커스텀 제스처를 구현하여 풍부한 상호작용을 제공합니다.

InputBindings를 효과적으로 활용하면 키보드 중심 사용자의 생산성을 크게 향상시킬 수 있으며, 전반적인 애플리케이션의 전문적인 인상을 강화할 수 있습니다.