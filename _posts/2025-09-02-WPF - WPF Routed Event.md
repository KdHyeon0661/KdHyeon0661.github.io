---
layout: post
title: WPF - WPF Routed Event
date: 2025-09-02 20:25:23 +0900
category: WPF
---
# WPF Routed Event 완전 정복 — 터널링(Preview*) / 버블링(*Without Preview)

## 0. 한눈 개념: Routed Event란?

- **Routed Event** = “이벤트가 **트리를 따라 이동**하는” WPF 이벤트. 세 가지 라우팅 전략:
  1) **Direct**: 발생한 요소에서만 처리 (WinForms 스타일)
  2) **Bubbling**: **자식 → 부모**(컨트롤에서 Window까지)로 **위로** 올라감
  3) **Tunneling**: **루트 → 자식**(Window에서 컨트롤까지)로 **아래로** 내려감
     - WPF 관례: 터널링 이름은 **`Preview` 접두사** (예: `PreviewMouseDown`)
- 이동 경로: **Visual Tree**(대부분) + 특정 시나리오에서 **ContentElement**/**3D** 등 포함

### 왜 필요한가?
- **관심사의 분리**: 하위 컨트롤 내부 구현을 몰라도 상위에서 일괄 처리 가능
- **입력 정책**: 창/페이지 단에서 통합 필터(Preview) → 조건부 차단 또는 로깅
- **템플릿 재사용**: ControlTemplate 내부 자식들이 발생시킨 이벤트를 Control이 “대표”로 처리

---

## 1. 라우팅 전략 비교 요약

| 전략 | 경로 | 대표 이벤트 | 사용 목적 |
|---|---|---|---|
| **Direct** | 요소 자신 | `Loaded`, `SizeChanged` 등 | 발생 지점에서만 의미가 있을 때 |
| **Bubbling** | **자식 → 부모** | `Click`, `MouseDown`, `KeyDown` | 상위 레벨 공통 로직/명령 실행 |
| **Tunneling** | **부모 → 자식** | `PreviewMouseDown`, `PreviewKeyDown` | **사전 필터링/차단**/로깅/글로벌 핫키 |

---

## 2. “버블링 vs 터널링” 기본 감각 예제

### 2.1 XAML 구조
```xml
<Window x:Class="DemoRoutedEvents.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Routed Events Demo" Height="300" Width="420"
        PreviewMouseDown="Window_PreviewMouseDown"
        MouseDown="Window_MouseDown">

  <Grid Background="Beige" PreviewMouseDown="Grid_PreviewMouseDown" MouseDown="Grid_MouseDown">
    <Border Background="LightSkyBlue" Padding="16" CornerRadius="8"
            PreviewMouseDown="Border_PreviewMouseDown"
            MouseDown="Border_MouseDown">
      <Button Content="Click / MouseDown Here"
              PreviewMouseDown="Button_PreviewMouseDown"
              MouseDown="Button_MouseDown"
              Click="Button_Click" />
    </Border>
  </Grid>
</Window>
```

### 2.2 Code-behind 로깅
```csharp
private void Log(string where, RoutedEventArgs e) =>
    Debug.WriteLine($"{DateTime.Now:HH:mm:ss.fff} [{where}] {e.RoutedEvent.Name}, Source={e.Source}, OriginalSource={e.OriginalSource}, Handled={e.Handled}");

// Tunneling(Preview*)
private void Window_PreviewMouseDown(object s, MouseButtonEventArgs e) => Log("Window(PRE)", e);
private void Grid_PreviewMouseDown(object s, MouseButtonEventArgs e) => Log("Grid(PRE)", e);
private void Border_PreviewMouseDown(object s, MouseButtonEventArgs e) => Log("Border(PRE)", e);
private void Button_PreviewMouseDown(object s, MouseButtonEventArgs e) => Log("Button(PRE)", e);

// Bubbling(*)
private void Button_MouseDown(object s, MouseButtonEventArgs e) => Log("Button(BUB)", e);
private void Border_MouseDown(object s, MouseButtonEventArgs e) => Log("Border(BUB)", e);
private void Grid_MouseDown(object s, MouseButtonEventArgs e) => Log("Grid(BUB)", e);
private void Window_MouseDown(object s, MouseButtonEventArgs e) => Log("Window(BUB)", e);

// Click (Button 전용 Bubbling)
private void Button_Click(object s, RoutedEventArgs e) => Log("Button.Click", e);
```

### 2.3 실행 흐름 (버튼 위 클릭)
1. **터널링**: `Window.PreviewMouseDown` → `Grid.PreviewMouseDown` → `Border.PreviewMouseDown` → `Button.PreviewMouseDown`
2. **버블링**: `Button.MouseDown` → `Border.MouseDown` → `Grid.MouseDown` → `Window.MouseDown`
3. (버튼의) `Click`은 `MouseUp` 포함 입력 제스처로 **버블링** 이벤트로 발화

> `e.Source`는 **논리적 발신자**(Button), `e.OriginalSource`는 **히트된 실제 시각 요소**(Template 내부 `ContentPresenter` 등)일 수 있습니다.

---

## 3. `e.Handled`의 의미와 “이벤트가 안 온다” 문제

- **`e.Handled = true`**: “이 이벤트는 처리됨” → **동일 이벤트의 라우트 이후 단계에서 더 이상 호출되지 않음**
  (단, **강제 구독**하는 방법 있음: `AddHandler(RoutedEvent, Handler, handledEventsToo: true)`)

### 3.1 Handled에 막히는 예시
```csharp
private void Button_PreviewMouseDown(object s, MouseButtonEventArgs e)
{
    // 버튼 단계에서 사전 필터 후 차단
    e.Handled = true;
    Log("Button(PRE) - set Handled", e);
}
// 결과: 이 아래로 내려가는 PRE는 호출 안 되고, 이후 Bubbling도 호출되지 않음(동일 라우트에 한해)
```

### 3.2 `handledEventsToo`로 강제 수신
```csharp
public MainWindow()
{
    InitializeComponent();
    // Button.MouseDown이 Handled로 막혀도 Window에서 “강제로” 받기
    AddHandler(UIElement.MouseDownEvent,
               new MouseButtonEventHandler(CatchAll_MouseDown),
               /*handledEventsToo*/ true);
}
private void CatchAll_MouseDown(object s, MouseButtonEventArgs e) =>
    Log("Window(AddHandler, handledEventsToo:true)", e);
```

> **핵심**: 상용 라이브러리 컨트롤이 **내부에서 `Handled=true`로 막아버리는 이벤트**를, 상위에서 **분석/로깅/제스처** 용도로 **여전히 수신**해야 할 때 `handledEventsToo`가 생명줄입니다.

---

## 4. Routed Event의 “라우트(Route)”는 어떻게 구성될까?

- 이벤트 발생 시 프레임워크가 **히트된 시각 요소**(`OriginalSource`)에서 시작해 **Visual Tree**를 따라 “경로”를 만들고,
  전략에 따라 **내려가거나(터널)** **올라오면서(버블)** 각 노드의 클래스 핸들러/인스턴스 핸들러를 호출합니다.
- **ContentElement**나 **3D** 요소도 참여할 수 있으며, **`EventRoute`** 내부적으로 클래스 핸들러 → 인스턴스 핸들러 순서 등 호출 규칙을 가집니다.

> 주의: WPF의 FE(`FrameworkElement`)와 FCE(`FrameworkContentElement`)는 서로 다른 계층이지만 **라우팅에 함께** 포함될 수 있습니다.

---

## 5. RoutedEvent vs CLR 이벤트

| 비교 | Routed Event | CLR Event |
|---|---|---|
| 전파 | 트리를 따라 이동(터널/버블/Direct) | 없음 (발생 지점 한정) |
| 핸들러 등록 | `AddHandler(RoutedEvent, …)` 또는 XAML 속성 | `+=` 문법 |
| 차단 | `e.Handled = true`로 라우트 차단 | 개념 없음 |
| 시나리오 | 입력/명령/컨트롤 상호작용 | 내부 로직 콜백 |

---

## 6. 템플릿 내부 요소가 낸 이벤트를 Control에서 처리하기

컨트롤 템플릿 안 버튼이 낸 클릭을 **컨트롤 외부에서** 처리하고 싶을 때:

### 6.1 ControlTemplate 내부
```xml
<ControlTemplate TargetType="{x:Type local:Card}">
  <Grid>
    <!-- 템플릿 내부 버튼 -->
    <Button x:Name="PART_Action" Content="Action"/>
  </Grid>
</ControlTemplate>
```

### 6.2 컨트롤 클래스에서 클래스 핸들러 등록 (정적 생성자)
```csharp
public class Card : Control
{
    static Card()
    {
        // 모든 Button.Click이 아님! "Card의 시각 트리 내부에서 발생한" Click 라우트에만 반응
        EventManager.RegisterClassHandler(typeof(Card),
                                          Button.ClickEvent,
                                          new RoutedEventHandler(OnAnyButtonClickInside));
    }
    private static void OnAnyButtonClickInside(object sender, RoutedEventArgs e)
    {
        if (sender is Card card)
        {
            // 템플릿의 PART_Action인지 확인(OriginalSource)
            if (e.OriginalSource is Button b && b.Name == "PART_Action")
            {
                card.OnActionClicked(e);
                e.Handled = true; // 이 컨트롤에서 소비
            }
        }
    }

    protected virtual void OnActionClicked(RoutedEventArgs e)
    {
        // 외부로 노출할 별도의 라우티드 이벤트를 Raise해도 좋다
        RaiseEvent(new RoutedEventArgs(ActionClickedEvent, this));
    }

    public static readonly RoutedEvent ActionClickedEvent =
        EventManager.RegisterRoutedEvent("ActionClicked", RoutingStrategy.Bubble,
            typeof(RoutedEventHandler), typeof(Card));
    public event RoutedEventHandler ActionClicked
    {
        add => AddHandler(ActionClickedEvent, value);
        remove => RemoveHandler(ActionClickedEvent, value);
    }
}
```

> **포인트**: `RegisterClassHandler`는 **특정 형식(Type)**에 대해 **전역적으로** “해당 형식 인스턴스의 라우트에 올라오는 이벤트”를 가로챕니다.
> 템플릿 구조가 바뀌어도 Control은 안정적으로 내부 이벤트를 수신할 수 있어 **캡슐화**가 좋아집니다.

---

## 7. 커스텀 Routed Event 정의 (직접 발화/버블/터널)

### 7.1 선언/등록
```csharp
public class Stepper : Control
{
    // 1) 등록
    public static readonly RoutedEvent StepChangedEvent =
        EventManager.RegisterRoutedEvent(
            name: "StepChanged",
            routingStrategy: RoutingStrategy.Bubble, // Bubble / Tunnel / Direct
            handlerType: typeof(RoutedPropertyChangedEventHandler<int>),
            ownerType: typeof(Stepper));

    // 2) CLR 이벤트 래퍼
    public event RoutedPropertyChangedEventHandler<int> StepChanged
    {
        add    => AddHandler(StepChangedEvent, value);
        remove => RemoveHandler(StepChangedEvent, value);
    }

    private int _step;
    public int Step
    {
        get => _step;
        set
        {
            if (_step == value) return;
            int old = _step; _step = value;

            // 3) 발화(Raise)
            var args = new RoutedPropertyChangedEventArgs<int>(old, _step, StepChangedEvent);
            RaiseEvent(args);
        }
    }
}
```

### 7.2 사용
```xml
<local:Stepper StepChanged="Stepper_StepChanged"/>
```

```csharp
private void Stepper_StepChanged(object s, RoutedPropertyChangedEventArgs<int> e)
{
    Debug.WriteLine($"Step changed: {e.OldValue} -> {e.NewValue}");
}
```

> **`RoutedPropertyChangedEventArgs<T>`**를 이용하면 **이전/새 값**을 자연스럽게 전달.
> `RoutingStrategy`를 `Tunnel`로 두면 **`PreviewStepChanged`**처럼 이름을 짓는 관례를 따르는 게 좋습니다.

---

## 8. “소유자 추가(AddOwner)”로 이벤트 재사용

- WPF 기본 컨트롤이 가진 RoutedEvent를 **내 컨트롤도 동일 의미로** 노출하고 싶다면:

```csharp
public class HyperLabel : Label
{
    public static readonly RoutedEvent ClickEvent =
        ButtonBase.ClickEvent.AddOwner(typeof(HyperLabel));

    public event RoutedEventHandler Click
    {
        add    => AddHandler(ClickEvent, value);
        remove => RemoveHandler(ClickEvent, value);
    }

    protected override void OnMouseLeftButtonUp(MouseButtonEventArgs e)
    {
        base.OnMouseLeftButtonUp(e);
        if (!e.Handled)
            RaiseEvent(new RoutedEventArgs(ClickEvent, this));
    }
}
```

> **장점**: 소비자 입장에서는 **Button의 Click과 똑같이** 다룰 수 있어 API 일관성이 좋아집니다.

---

## 9. 명령(Command) 라우팅과의 관계

- `ICommand`/`RoutedCommand`는 내부적으로 **Input(키/마우스) RoutedEvent**와 결합됩니다.
- 예: `Button` 클릭 → `Click`(RoutedEvent) → `Command` 실행
  또는 키 제스처 → `CommandBinding` 탐색(버블 라우팅) → `CanExecute/Executed` 호출

### 9.1 예제
```xml
<Window.CommandBindings>
  <CommandBinding Command="ApplicationCommands.Copy"
                  CanExecute="Copy_CanExecute" Executed="Copy_Executed"/>
</Window.CommandBindings>

<Button Command="ApplicationCommands.Copy" Content="Copy"/>
```

```csharp
private void Copy_CanExecute(object s, CanExecuteRoutedEventArgs e) => e.CanExecute = true;
private void Copy_Executed(object s, ExecutedRoutedEventArgs e) { /* 수행 */ }
```

> **핵심**: 명령 바인딩 탐색도 **라우트(버블)**의 일종.
> “왜 내 `Executed`가 안 불리지?” → **상위에 `CommandBinding`이 있는지**, `Handled`로 막힌 이벤트는 아닌지 점검.

---

## 10. `OriginalSource` vs `Source`

- **`OriginalSource`**: 히트 테스트(실제 클릭)로 가장 **안쪽**의 요소 (템플릿 자식, Run, Border 등)
- **`Source`**: 라우팅 과정에서 “현재 핸들러 관점의 소스”(상황에 따라 상위로 치환될 수 있음)
- 템플릿 내부 요소에만 있는 이름/특성 판별 시 **`OriginalSource`**를 주로 확인합니다.

```csharp
private void Any_MouseDown(object s, MouseButtonEventArgs e)
{
    var original = e.OriginalSource as DependencyObject;
    var templatedButton = original?.FindAncestor<Button>(); // 시각 트리 보조 메서드 사용 가정
}
```

---

## 11. 템플릿/컨트롤 경계에서의 라우팅 팁

- **ControlTemplate 내부**의 요소가 일으킨 버블 이벤트는 **컨트롤 인스턴스**까지 쭉 올라옵니다.
  → 외부에서는 “Control이 낸 것처럼” 다룰 수 있어 **구현 은닉**이 됩니다.
- 반대로 **Preview 이벤트(터널링)**는 **Window → … → Control → Template Child** 순으로 내려오므로,
  **상위에서 선제 차단**하거나 **컨트롤에서 필터**가 가능합니다.

---

## 12. 입력 이벤트 모음 (대표)

| 카테고리 | 터널링(Preview*) | 버블링(*) | 메모 |
|---|---|---|---|
| 마우스 | `PreviewMouseDown`, `PreviewMouseUp`, `PreviewMouseWheel`, `PreviewMouseMove` | `MouseDown`, `MouseUp`, `MouseWheel`, `MouseMove` | Wheel도 라우티드 |
| 키보드 | `PreviewKeyDown`, `PreviewKeyUp`, `PreviewTextInput` | `KeyDown`, `KeyUp`, `TextInput` | IME 입력은 TextInput 경유 |
| 스타일러스/터치 | `PreviewStylusDown`, `PreviewTouchDown` … | `StylusDown`, `TouchDown` … | 멀티 터치 |
| 포커스 | `PreviewGotKeyboardFocus`, `PreviewLostKeyboardFocus` | `GotKeyboardFocus`, `LostKeyboardFocus` | `Handled` 주의 |
| Drag&Drop | 내부적으로 라우팅 + 히트테스트 | | `AllowDrop` 필요 |

---

## 13. 고급: `OnPreview*` / `On*` 오버라이드 순서

컨트롤 파생 시 보통 다음 순서로 호출됩니다.

1. **터널링 클래스 핸들러** → **인스턴스 `OnPreview…` 오버라이드** → 인스턴스 XAML 핸들러
2. (핸들링/차단이 없으면) **버블링 인스턴스 `On…`** → XAML 핸들러 → 상위로 버블 계속

```csharp
protected override void OnPreviewMouseDown(MouseButtonEventArgs e)
{
    // 사전 필터/포커스 정책
    base.OnPreviewMouseDown(e);
}

protected override void OnMouseDown(MouseButtonEventArgs e)
{
    // 클릭 반응
    base.OnMouseDown(e);
}
```

> **무분별한 `Handled = true`**는 상위 로직/명령 라우팅을 깨뜨릴 수 있습니다. 꼭 필요한 곳만!

---

## 14. 라우트를 “가로세로” 모두 활용하는 실전 패턴

### 14.1 글로벌 키 필터(ESC로 다이얼로그 닫기)
```csharp
// App 전역(예: Window)
private void Window_PreviewKeyDown(object s, KeyEventArgs e)
{
    if (e.Key == Key.Escape && DialogIsOpen())
    {
        CloseDialog();
        e.Handled = true; // 더 이상 하위가 키를 받지 않음
    }
}
```

### 14.2 목록 항목 내부 컨트롤 클릭을 상위 명령으로 승격
```xml
<ListBox ItemsSource="{Binding Items}">
  <ListBox.ItemTemplate>
    <DataTemplate>
      <StackPanel>
        <TextBlock Text="{Binding Title}"/>
        <Button Content="Remove"
                Command="{Binding DataContext.RemoveItemCommand,
                          RelativeSource={RelativeSource AncestorType=ListBox}}"
                CommandParameter="{Binding}" />
      </StackPanel>
    </DataTemplate>
  </ListBox.ItemTemplate>
</ListBox>
```
> 클릭 자체는 하위에서 발생하지만, **커맨드 바인딩은 상위(리스트/Window)** 에서 처리 → **라우팅 + CommandBinding** 협업.

---

## 15. `AddHandler` 고급: 특정 타입만 필터하기

모든 `MouseDown`을 다 받으면 노이즈가 큼. **원본 타입**으로 필터:

```csharp
AddHandler(UIElement.MouseDownEvent, new MouseButtonEventHandler((s, e) =>
{
    if (e.OriginalSource is DependencyObject d)
    {
        var button = d as Button ?? d.FindAncestor<Button>();
        if (button != null)
        {
            // Button에서 올라온 클릭만 로깅
            Log("Window(MouseDown from Button)", e);
        }
    }
}), true); // handledEventsToo: 내부에서 Handled 해도 받음
```

---

## 16. 성능과 안정성 체크리스트

- **핸들러 최소화**: 최상위(Window)에 광범위한 핸들러를 많이 달면 불필요한 호출 증가
  → **정말 필요한 이벤트만** 구독, 내부에서 **조기 필터** (예: 타입/Name)
- **`Handled` 남용 금지**: 명령/단축키 등 다른 기능이 막힐 수 있음
- **템플릿 변경 영향**: `OriginalSource`가 바뀌어도 견고한 탐색 로직(`FindAncestor<T>`) 사용
- **디버깅 도구**: **Snoop / Live Visual Tree**로 `OriginalSource`/Route 시각화, Output 바인딩 오류 확인

---

## 17. “왜 내 이벤트가 안 오지?” 체크리스트

1. **다른 곳에서 `Handled = true` 했나?**
   → `AddHandler(ev, handler, handledEventsToo:true)`로 확인/우회
2. **템플릿 자식이 포커스를 먹는가?**
   → 포커스 이동 정책 조정, `Focusable="False"` 고려
3. **Route가 해당 컨테이너로 안 올라오나?**
   → Visual Tree 상의 부모가 맞는지(팝업/Adorner 등은 별 트리), `StaysOpen`/Popup Placement 영향 확인
4. **Drag&Drop/ContextMenu**는 다른 루프?
   → 별도 Mouse capture/Popup Window로 인해 루트가 다를 수 있음

---

## 18. 실습: 간단한 “이벤트 스파이” 컨트롤

### 18.1 XAML
```xml
<Grid>
  <local:Spy x:Name="Spy" />
  <StackPanel Margin="16" Background="#22FFFFFF">
    <Button Content="A"/>
    <Border Background="LightGreen" Padding="10">
      <TextBox Width="160" Height="26"/>
    </Border>
  </StackPanel>
</Grid>
```

### 18.2 Spy 구현
```csharp
public class Spy : Decorator
{
    private readonly TextBox _log = new() { IsReadOnly = true, VerticalScrollBarVisibility = ScrollBarVisibility.Auto };

    public Spy()
    {
        Child = _log;
        // 자주 쓰는 입력 이벤트를 전부 감시
        Hook(UIElement.MouseDownEvent);
        Hook(UIElement.MouseUpEvent);
        Hook(UIElement.MouseMoveEvent);
        Hook(UIElement.KeyDownEvent);
        Hook(UIElement.KeyUpEvent);
        Hook(UIElement.GotKeyboardFocusEvent);
        Hook(UIElement.LostKeyboardFocusEvent);
        Hook(ButtonBase.ClickEvent);
    }

    void Hook(RoutedEvent ev)
    {
        AddHandler(ev, new RoutedEventHandler((s, e) =>
        {
            var origin = e.OriginalSource as DependencyObject;
            var name = (origin as FrameworkElement)?.Name ?? origin?.GetType().Name ?? "?";
            Append($"{DateTime.Now:HH:mm:ss.fff} {ev.Name}  From={name}  Handled={e.Handled}");
        }), true); // handledEventsToo:true
    }

    void Append(string line)
    {
        _log.AppendText(line + Environment.NewLine);
        _log.ScrollToEnd();
    }
}
```

> 이 Spy를 이용하면, **어떤 이벤트가 어떤 순서로, 무엇을 원본으로** 올라오는지 바로 확인할 수 있습니다.

---

## 19. RoutedEvent와 “가상화 컨테이너” (ItemsControl)

- `ListBox`/`ListView`의 **가상화**가 켜져 있으면(기본) **화면에 보이는 아이템만** 컨테이너가 생성됩니다.
  → 스크롤로 “새로운 컨테이너”가 만들어지면 **클래스 핸들러**가 자동 적용되(도록 설계),
  라우팅도 정상 작동합니다.
- 단, `ItemContainerStyle`에서 **무거운 트리거/이벤트 작업**은 컨테이너 재사용(Recycling) 시 **상태 꼬임**을 유발할 수 있습니다.

---

## 20. 실전 Q&A (면접/코드 리뷰에서 자주 나오는 포인트)

**Q1. 터널링과 버블링을 각각 언제 쓰나요?**
- **터널링**: **사전 필터**(예: 특정 영역 클릭 금지, 글로벌 단축키 선점, 모달 오버레이)
- **버블링**: **상위 정책/명령 집행**(예: 리스트 안의 모든 “삭제” 버튼을 상위에서 하나의 커맨드로 처리)

**Q2. `Handled=true`의 부작용은?**
- 이후 라우트가 차단 → **명령 실행/바인딩/다른 핸들러**가 동작하지 않을 수 있음

**Q3. 템플릿 변경에 안전한 코드는?**
- `OriginalSource`에서 **조상 탐색**(유틸 메서드) → `Name`/`Tag`/`TemplatedParent`로 판별

**Q4. 클래스 핸들러와 인스턴스 핸들러 우선순위?**
- **클래스 핸들러**가 먼저 개입할 수 있습니다(등록 시점/라우트 단계에 따라).
  공통 행동/차단은 클래스 핸들러, 개별 행동은 인스턴스 핸들러로 나누세요.

---

## 21. 유틸: 조상 탐색 도우미

```csharp
public static class VisualTreeHelpers
{
    public static T? FindAncestor<T>(this DependencyObject? d) where T : DependencyObject
    {
        while (d != null)
        {
            if (d is T t) return t;
            d = VisualTreeHelper.GetParent(d);
        }
        return null;
    }
}
```

---

## 22. Drag&Drop와 RoutedEvent

- Drag 시작은 보통 `MouseMove`+`LeftButtonDown` 조합 → `DoDragDrop` 호출
- Drop 대상은 **`AllowDrop=true`** 필요 + `DragEnter/Over/Leave/Drop`(Routed)
- Preview 단계에서 **파일 확장자/데이터 포맷 필터** 후 `Handled=true`로 조기 차단 가능

---

## 23. Popup/Adorner/Topmost 경계

- `Popup`은 **별도 윈도우**로 떠서 **Visual Tree가 분리**됩니다.
  → 라우팅이 **부모 창으로 버블되지 않음**. 필요하면 **`PlacementTarget`**로 상호작용 설계.
- **AdornerLayer**는 시각 트리 레이어이지만 라우팅은 기본적으로 **원본 트리**를 따릅니다.

---

## 24. 입력 제스처와 터널링 조합 (핫키 처리 예)

```csharp
// 상위에서 ESC/Enter 핫키 공통 처리
protected override void OnPreviewKeyDown(KeyEventArgs e)
{
    if (e.Key == Key.Escape && CanCancel()) { Cancel(); e.Handled = true; return; }
    if (e.Key == Key.Enter && CanSubmit())  { Submit(); e.Handled = true; return; }
    base.OnPreviewKeyDown(e);
}
```

---

## 25. 테스트 전략

- **단위 테스트**: ViewModel/Command 로직은 테스트 쉽지만, 이벤트 라우팅은 **UI 런타임 필요**
  → **UI 테스트 프레임워크**(Playwright-WPF 브릿지, TestStack.White 등) 또는 **시뮬레이터** 사용
- **시각 검사**: Spy/로거로 사건 순서/Handled 여부 규칙 검증

---

## 26. 미니 캡스톤: “클릭 차단 레이어” 구현

### 요구
- 특정 영역을 “모달”로 막고, 내부 Dialog 외의 클릭은 **모두 흡수**(작동 금지)

### 구현
```xml
<Grid>
  <!-- 본문 -->
  <ContentPresenter x:Name="Body"/>

  <!-- 오버레이 -->
  <Border x:Name="Overlay" Background="#66000000" Visibility="Collapsed"
          PreviewMouseDown="Overlay_PreviewMouseDown"
          PreviewKeyDown="Overlay_PreviewKeyDown"/>

  <!-- 모달 -->
  <Border x:Name="Dialog" Width="360" Height="200" Background="White"
          Visibility="Collapsed" />
</Grid>
```

```csharp
private void Overlay_PreviewMouseDown(object s, MouseButtonEventArgs e)
{
    // 오버레이를 누르면 모달 외부 클릭 → 흡수
    e.Handled = true;
}
private void Overlay_PreviewKeyDown(object s, KeyEventArgs e)
{
    if (e.Key == Key.Escape) CloseModal();
    e.Handled = true; // 키도 흡수
}
```

> **터널링** 덕분에 부모 레이어에서 **아래 자식들에게 내려가기 전에** 이벤트를 **흡수**하여
> 뒤쪽 컨트롤이 실수로 클릭되는 것을 막을 수 있습니다.

---

## 27. 요약 체크리스트

- [ ] 터널링 = **Preview** 하향식, 버블링 = 상향식
- [ ] `Handled`는 라우트 **차단** / `AddHandler(…, true)`로 **무시** 가능
- [ ] 템플릿 내부 이벤트는 **클래스 핸들러**로 캡슐화
- [ ] `OriginalSource` 기준으로 **정확한 발신자 판별**
- [ ] 명령 라우팅과 이벤트 라우팅의 **협업** 이해
- [ ] Popup/Adorner/가상화 등 **특수 트리**에서 라우팅 경계 인지
- [ ] 성능: 상위에 **광범위 핸들러 남발 금지**, 조기 필터 적용

---

## 28. 마무리

Routed Event는 단순 “이벤트가 위아래로 움직인다”가 아니라,
**입력 정책의 중앙집중, 템플릿 캡슐화, 상위 명령 승격, 모달/오버레이, 고급 단축키 처리**까지
WPF의 **표현력 있는 상호작용**을 가능케 하는 핵심입니다.

현업에서는 아래 세 가지를 특히 기억하세요.

1) **Preview에서 필터, 일반에서 행위** (필요 시 `Handled=true`)
2) **타인의 Handled를 뚫어야 하면 `handledEventsToo:true`**
3) **템플릿 내부를 외부 계약으로 끌어올릴 땐 `RegisterClassHandler` + 커스텀 RoutedEvent**

이 가이드를 토대로 팀 코드에서 “왜 이 이벤트가 여기서 잡히고/안 잡히는지”를 명쾌하게 설명하고,
안정적이고 예측 가능한 입력/명령 시스템을 설계해 보세요. 🚀

---
### 부록 A) 전체 예제 코드(요약)

> 본문 예제를 하나의 프로젝트에 모은 버전(핵심 부분만).
> 필요하시면 **완전 실행 가능한 솔루션** 형태로도 정리해 드릴게요.

```csharp
// App.xaml.cs
public partial class App : Application { }

// MainWindow.xaml.cs
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        // handledEventsToo 예시
        AddHandler(UIElement.MouseDownEvent,
            new MouseButtonEventHandler((s, e) => Log("Window(AddHandler,true)", e)), true);
    }
    void Log(string where, RoutedEventArgs e) =>
        Debug.WriteLine($"{where} {e.RoutedEvent.Name} Source={e.Source} Original={e.OriginalSource} Handled={e.Handled}");

    private void Window_PreviewMouseDown(object s, MouseButtonEventArgs e) => Log("Window(PRE)", e);
    private void Window_MouseDown(object s, MouseButtonEventArgs e) => Log("Window(BUB)", e);
    private void Grid_PreviewMouseDown(object s, MouseButtonEventArgs e) => Log("Grid(PRE)", e);
    private void Grid_MouseDown(object s, MouseButtonEventArgs e) => Log("Grid(BUB)", e);
    private void Border_PreviewMouseDown(object s, MouseButtonEventArgs e) => Log("Border(PRE)", e);
    private void Border_MouseDown(object s, MouseButtonEventArgs e) => Log("Border(BUB)", e);
    private void Button_PreviewMouseDown(object s, MouseButtonEventArgs e) { Log("Button(PRE)", e); /* e.Handled = true; */ }
    private void Button_MouseDown(object s, MouseButtonEventArgs e) => Log("Button(BUB)", e);
    private void Button_Click(object s, RoutedEventArgs e) => Log("Button.Click", e);
}
```

```xml
<!-- MainWindow.xaml -->
<Window x:Class="DemoRoutedEvents.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Routed Events Demo" Height="300" Width="420"
        PreviewMouseDown="Window_PreviewMouseDown"
        MouseDown="Window_MouseDown">
  <Grid Background="Beige" PreviewMouseDown="Grid_PreviewMouseDown" MouseDown="Grid_MouseDown">
    <Border Background="LightSkyBlue" Padding="16" CornerRadius="8"
            PreviewMouseDown="Border_PreviewMouseDown"
            MouseDown="Border_MouseDown">
      <Button Content="Click / MouseDown Here"
              PreviewMouseDown="Button_PreviewMouseDown"
              MouseDown="Button_MouseDown"
              Click="Button_Click" />
    </Border>
  </Grid>
</Window>
```

```csharp
// 카드 컨트롤 + 클래스 핸들러 예시
public class Card : Control
{
    static Card()
    {
        DefaultStyleKeyProperty.OverrideMetadata(typeof(Card),
            new FrameworkPropertyMetadata(typeof(Card)));

        EventManager.RegisterClassHandler(typeof(Card),
            ButtonBase.ClickEvent,
            new RoutedEventHandler(OnAnyButtonClickInside));
    }

    private static void OnAnyButtonClickInside(object sender, RoutedEventArgs e)
    {
        if (sender is Card card && e.OriginalSource is Button b && b.Name == "PART_Action")
        {
            card.RaiseEvent(new RoutedEventArgs(ActionClickedEvent, card));
            e.Handled = true;
        }
    }

    public static readonly RoutedEvent ActionClickedEvent =
        EventManager.RegisterRoutedEvent("ActionClicked", RoutingStrategy.Bubble,
            typeof(RoutedEventHandler), typeof(Card));
    public event RoutedEventHandler ActionClicked
    {
        add => AddHandler(ActionClickedEvent, value);
        remove => RemoveHandler(ActionClickedEvent, value);
    }
}
```

```xml
<!-- Generic.xaml -->
<ResourceDictionary
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:local="clr-namespace:DemoRoutedEvents">
  <Style TargetType="{x:Type local:Card}">
    <Setter Property="Template">
      <Setter.Value>
        <ControlTemplate TargetType="{x:Type local:Card}">
          <Border Background="White" Padding="8" CornerRadius="6" BorderBrush="#DDD" BorderThickness="1">
            <StackPanel>
              <ContentPresenter/>
              <Button x:Name="PART_Action" Content="Action"/>
            </StackPanel>
          </Border>
        </ControlTemplate>
      </Setter.Value>
    </Setter>
  </Style>
</ResourceDictionary>
```
