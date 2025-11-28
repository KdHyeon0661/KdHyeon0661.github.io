---
layout: post
title: WPF - DependencyProperty, RoutedEvent
date: 2025-08-27 18:25:23 +0900
category: WPF
---
# WPF 핵심: **DependencyProperty**와 **RoutedEvent** 완전 이해

## 한 장 요약 (TL;DR)

- **DependencyProperty(DP)**:
  - `DependencyObject`에 붙는 **고성능 속성 시스템**. **값 우선순위(애니메이션 > 로컬 값 > 스타일 …)**, **변경 트래킹**, **상속(Inheritance)**, **애니메이션/바인딩/테마** 연동, **레이아웃/렌더 무효화(AffectsMeasure/Arrange/Render)** 등 WPF의 핵심 메커니즘을 제공.
  - 커스텀 컨트롤/프레임워크 요소를 만들 때 **필수**.

- **RoutedEvent(RE)**:
  - **트리 기반 이벤트 전파**(터널링 Preview → 버블링)로 복잡한 UI에서 **중앙집중 처리**와 **관심사의 분리**를 가능케 함.
  - 입력 이벤트(Mouse/Keyboard)와 명령, 스타일/트리거, 클래스 핸들러까지 **광범위하게 결합**.

- **함께 쓰면**:
  - RE로 상태 변화를 “상위에서” 잡고, DP로 상태를 “아래까지” 반영/재렌더링.
  - “입력 → 상태(DP) 변경 → 레이아웃/렌더 → 합성”의 WPF 파이프라인이 완성됨.

---

## 왜 DependencyProperty인가?

### .NET 일반 속성의 한계

- 단순 C# 속성은 **우선순위/애니메이션/바인딩/상속/변경 알림/테마** 개념 없음.
- WPF는 **수만 개 객체/속성**이 동시에 얽히며 바뀜 → **더 강력하고 결정적인 시스템 필요**.

### DP가 제공하는 것

- **값 우선순위 스택**: 애니메이션, 로컬 값, 스타일 Setter, 트리거, 상속값, 기본값.
- **변경 트래킹**: 값 변경시 **콜백**과 **무효화(Affects…)** 자동.
- **리소스/스타일/테마/바인딩/애니메이션**과 자연 통합.
- **메모리 효율**: 기본값은 테이블 참조, 로컬 값만 박아넣는 **희소 저장**.

---

## DependencyProperty의 **기본 사용 패턴**

아래는 `MeterBar`라는 `FrameworkElement`에 DP `Value`를 정의하는 최소 예제입니다.

```csharp
public class MeterBar : FrameworkElement
{
    // 1) 의존 속성 식별자(정적, readonly)
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(
            name: nameof(Value),
            propertyType: typeof(double),
            ownerType: typeof(MeterBar),
            typeMetadata: new FrameworkPropertyMetadata(
                defaultValue: 0.0,
                flags: FrameworkPropertyMetadataOptions.AffectsRender | FrameworkPropertyMetadataOptions.BindsTwoWayByDefault,
                propertyChangedCallback: OnValueChanged,
                coerceValueCallback: CoerceValue),
            // 유효성 검사(등록 시점)
            validateValueCallback: ValidateValue);

    // 2) CLR wrapper (선택적이지만 권장)
    public double Value
    {
        get => (double)GetValue(ValueProperty);
        set => SetValue(ValueProperty, value);
    }

    // 3) Validate: "어떤 스레드/시점에서도" 호출될 수 있음(가벼워야 함)
    private static bool ValidateValue(object v)
    {
        double d = (double)v;
        return !double.IsNaN(d) && !double.IsInfinity(d);
    }

    // 4) Coerce: "현재 객체 상태"를 보고 값을 강제 조정
    private static object CoerceValue(DependencyObject d, object baseValue)
    {
        var val = (double)baseValue;
        if (val < 0) return 0.0;
        if (val > 1) return 1.0;
        return val;
    }

    // 5) PropertyChanged: 값이 확정된 뒤(Validate/Coerce 후) 호출
    private static void OnValueChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        // 필요 시 추가 로직 (로그, 이벤트, 상호 종속 DP Coerce 호출 등)
    }

    // 6) 렌더
    protected override void OnRender(DrawingContext dc)
    {
        var rect = new Rect(0, 0, ActualWidth, ActualHeight);
        dc.DrawRectangle(Brushes.DimGray, null, rect);
        double w = rect.Width * Math.Max(0, Math.Min(1, Value));
        dc.DrawRectangle(Brushes.LimeGreen, null, new Rect(0, 0, w, rect.Height));
    }
}
```

**핵심 포인트**
- `FrameworkPropertyMetadataOptions.AffectsRender` 덕분에 `Value` 변경 시 **렌더 무효화** 자동.
- `ValidateValue`는 **정적** 검증(상태 무관), `CoerceValue`는 **인스턴스 상태 기반** 강제.
- `BindsTwoWayByDefault`를 주면 바인딩 기본 모드가 TwoWay.

---

## **값 우선순위(Precedence)** 완전 정리

WPF는 한 속성에 대해 여러 출처가 값을 제시할 수 있으므로 **우선순위 체계**로 최종 유효값을 결정합니다(간략화):

1. **애니메이션(Active Animation/Storyboard)**
2. **로컬 값(Local Value)**: 코드의 `SetValue`, XAML의 직접 할당
3. **템플릿/스타일 트리거 값**
4. **스타일 Setter**
5. **테마 스타일(Theme Style)**
6. **상속된 값(Property Value Inheritance)**
7. **기본값(Default Value)**

> **Tip.** 우선순위가 높은 쪽이 값을 “덮어쓴다”. 예: 로컬 값을 줘도 **애니메이션이 뛰고 있으면** 보이는 값은 애니메이션 결과.

### 코드로 우선순위 체감하기

```csharp
// Local Value 설정
myBar.Value = 0.2;

// 애니메이션이 덮어씀
var da = new DoubleAnimation(0, 1, TimeSpan.FromSeconds(1)) { AutoReverse = true, RepeatBehavior = RepeatBehavior.Forever };
myBar.BeginAnimation(MeterBar.ValueProperty, da);

// 애니메이션 중엔 화면상 값은 애니메이션이 결정.
// 중지하면 Local(0.2)로 복귀:
myBar.BeginAnimation(MeterBar.ValueProperty, null);
```

---

## 메타데이터(FrameworkPropertyMetadata)의 **옵션 플래그**

- **AffectsMeasure / AffectsArrange / AffectsRender**: 값 변경 시 레이아웃/렌더 자동 무효화.
- **Inherits**: **속성값 상속** 허용(예: `FontFamily`, `FlowDirection`).
- **BindsTwoWayByDefault**: 바인딩 기본 모드 TwoWay.
- **Journal**: Navigation journal에 참여.
- **SubPropertiesDoNotAffectRender**: 하위 속성 변경이 렌더에 영향 없음을 힌트.

```csharp
new FrameworkPropertyMetadata(
    defaultValue: Brushes.Black,
    flags: FrameworkPropertyMetadataOptions.AffectsRender | FrameworkPropertyMetadataOptions.SubPropertiesDoNotAffectRender);
```

> **주의**: Brush 같은 **Freezable**은 내부 상태(예: Color) 변경 시에도 렌더 무효화가 필요할 수 있음. Freeze하면 하위 변경 불가 → 안전/성능 상승.

---

## **Attached Property(부착 속성)**

다른 타입의 객체에 값을 “붙일” 수 있는 DP. 레이아웃 패널의 `Grid.Row`, `Canvas.Left`가 대표적.

```csharp
public static class DockExtensions
{
    public static readonly DependencyProperty BadgeProperty =
        DependencyProperty.RegisterAttached(
            "Badge", typeof(string), typeof(DockExtensions),
            new FrameworkPropertyMetadata(default(string), FrameworkPropertyMetadataOptions.AffectsRender));

    public static void SetBadge(UIElement element, string? value) => element.SetValue(BadgeProperty, value);
    public static string? GetBadge(UIElement element) => (string?)element.GetValue(BadgeProperty);
}
```

```xml
<Button Content="Inbox" local:DockExtensions.Badge="99+"/>
```

> **Attached DP**는 **레이아웃/행동을 외부에서 주입**할 수 있어 프레임워크 구성성을 극대화.

---

## **ReadOnly DP**와 **DependencyPropertyKey**

컨트롤 내부 상태는 외부에서 `SetValue` 못하게 막고 싶을 때 **읽기 전용 DP**를 쓴다.

```csharp
public class RangeMeter : FrameworkElement
{
    private static readonly DependencyPropertyKey IsOverloadedPropertyKey =
        DependencyProperty.RegisterReadOnly(
            nameof(IsOverloaded), typeof(bool), typeof(RangeMeter),
            new FrameworkPropertyMetadata(false, FrameworkPropertyMetadataOptions.AffectsRender));

    public static readonly DependencyProperty IsOverloadedProperty = IsOverloadedPropertyKey.DependencyProperty;

    public bool IsOverloaded => (bool)GetValue(IsOverloadedProperty);

    public double Value
    {
        get => (double)GetValue(ValueProperty);
        set => SetValue(ValueProperty, value);
    }
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(nameof(Value), typeof(double), typeof(RangeMeter),
            new FrameworkPropertyMetadata(0.0, OnValueChanged));

    private static void OnValueChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        var self = (RangeMeter)d;
        self.SetValue(IsOverloadedPropertyKey, (double)e.NewValue > 0.9);
    }
}
```

---

## **PropertyChangedCallback / Coerce / Validate** 패턴 심화

- **Validate**: 값 자체의 기본 제약(스레드 안전, 빠르게).
- **Coerce**: **객체 상태 기반** 보정. 다른 DP 변화 시 **재강제(CoerceValue)** 호출권장.
- **PropertyChanged**: 확정된 값에 대한 후처리(다른 DP 영향, 이벤트 발생, 캐시 무효화).

### 서로 의존하는 DP 간 Coerce 예시

```csharp
public static readonly DependencyProperty MinimumProperty =
    DependencyProperty.Register(nameof(Minimum), typeof(double), typeof(RangeMeter),
        new FrameworkPropertyMetadata(0.0, OnMinMaxChanged));

public static readonly DependencyProperty MaximumProperty =
    DependencyProperty.Register(nameof(Maximum), typeof(double), typeof(RangeMeter),
        new FrameworkPropertyMetadata(1.0, OnMinMaxChanged));

private static void OnMinMaxChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
{
    var self = (RangeMeter)d;
    // Min/Max 바뀌면 Value를 재강제
    self.CoerceValue(ValueProperty);
}

private static object CoerceValue(DependencyObject d, object baseValue)
{
    var self = (RangeMeter)d;
    double v = (double)baseValue;
    return Math.Max(self.Minimum, Math.Min(self.Maximum, v));
}
```

---

## DP와 **INotifyPropertyChanged**의 관계

- DP는 자체적으로 변경 트래킹/전파를 제공 → **일반적으로 DP에 INPC 불필요**.
- **ViewModel** 계층에서 POCO 속성 바인딩 시에는 INPC가 **필수**.
- **Control 내부**: 공개 DP + 내부 캐시 필드는 **DP Change에서 UI 반영**, VM 값은 INPC로 UI 반영.

---

## DP **도구·디버깅**

- **Output 창 바인딩 오류** 확인: 경로/타입 불일치
- `DependencyPropertyDescriptor.FromProperty(...).AddValueChanged(...)`로 변화 훅
- `GetLocalValueEnumerator()`로 로컬 값 나열
- `ClearValue(DP)`로 로컬 값 해제(우선순위 하향)

```csharp
var enumerator = myElt.GetLocalValueEnumerator();
while (enumerator.MoveNext())
{
    var entry = enumerator.Current;
    Debug.WriteLine($"{entry.Property.Name} = {entry.Value}");
}
```

---

## **RoutedEvent** 개요

### 왜 Routed?

- 복잡한 트리(논리/비주얼)에서 **개별 요소마다 핸들러**를 다는 건 비효율.
- **터널링(Preview…)**으로 상위에서 **사전 제어**, **버블링**으로 상위에서 **사후 처리**.
- 상위 컨테이너가 **하위의 세부 구현을 몰라도** 공통 로직을 수신/차단 가능(`e.Handled = true`).

### 전파 방식

- **Direct**: 전파 없음(일반 .NET 이벤트와 유사).
- **Tunneling**: 루트→소스까지 내려가며 전달. 이름 규칙상 **Preview***.
- **Bubbling**: 소스→루트로 올라오며 전달(기본 MouseDown 등).

---

## RoutedEvent 정의·등록·발생(raise)

### 사용자 정의 RoutedEvent 등록

```csharp
public class BadgeButton : Button
{
    // 1) 이벤트 식별자 등록
    public static readonly RoutedEvent BadgeClickedEvent =
        EventManager.RegisterRoutedEvent(
            name: "BadgeClicked",
            routingStrategy: RoutingStrategy.Bubble,
            handlerType: typeof(RoutedEventHandler),
            ownerType: typeof(BadgeButton));

    // 2) .NET 이벤트 래퍼
    public event RoutedEventHandler BadgeClicked
    {
        add => AddHandler(BadgeClickedEvent, value);
        remove => RemoveHandler(BadgeClickedEvent, value);
    }

    protected override void OnClick()
    {
        base.OnClick();
        // 3) 라우티드 이벤트 발생
        RaiseEvent(new RoutedEventArgs(BadgeClickedEvent, this));
    }
}
```

### XAML에서 핸들러 연결

```xml
<local:BadgeButton Content="Inbox" BadgeClicked="OnBadgeClicked"/>
```

```csharp
private void OnBadgeClicked(object sender, RoutedEventArgs e)
{
    // 버블링 중간에서 잡을 수도 있고
    // e.Handled = true; // 상위 전파 차단 가능
}
```

---

## RoutedEvent **핵심 속성**: `Source` vs `OriginalSource`, `Handled`

- **OriginalSource**: **최초** 이벤트가 발생한 요소(보통 가장 깊은 하위).
- **Source**: 전파 중 **현재 컨텍스트에서 보정된 소스**(템플릿 경계에서 바뀔 수 있음).
- **Handled**: `true`로 설정되면 이후 “기본” 핸들러(스타일 첨부 핸들러 포함)가 **건너뛰어짐**.
  - 단, **AddHandler(routedEvent, handler, handledEventsToo:true)**로 **이미 Handled된 이벤트도 수신** 가능.

```csharp
myListBox.AddHandler(UIElement.MouseDownEvent,
    new MouseButtonEventHandler((s, e) =>
    {
        Debug.WriteLine($"OriginalSource: {e.OriginalSource}, Source: {e.Source}");
    }), handledEventsToo: true);
```

---

## **클래스 핸들러(Class Handler)**

컨트롤 소유 타입에 **정적 등록**하여, 인스턴스에 별도 코드 없이 공통 동작을 주입.

```csharp
public class MeterBar : FrameworkElement
{
    static MeterBar()
    {
        EventManager.RegisterClassHandler(
            typeof(MeterBar),
            UIElement.MouseDownEvent,
            new MouseButtonEventHandler(OnMeterMouseDown),
            handledEventsToo: false);
    }

    private static void OnMeterMouseDown(object sender, MouseButtonEventArgs e)
    {
        var self = (MeterBar)sender;
        // 모든 MeterBar 인스턴스에 공통 적용
        e.Handled = true;
    }
}
```

---

## vs 버블링** 실전 이해

예: `PreviewMouseDown`(터널링)에서 상위 컨테이너가 **해당 입력을 차단**하면, 하위 버튼의 `Click`까지 **막을 수 있음**.

```csharp
this.PreviewMouseDown += (s, e) =>
{
    if (ShouldBlock(e)) e.Handled = true; // 하위로 내려갈 이벤트 차단
};
```

> 입력 필터/제스처 프레임워크 구축 시 유용. 반대로 글로벌 로깅/측정은 **버블링**에서 수집.

---

## 라우팅**

- `Button.Command` 등 **Command 바인딩**은 내부적으로 **라우팅**과 결합.
- `CommandBinding`을 **상위 컨테이너**에 걸어두면 하위 컨트롤에서 발생한 명령도 처리 가능.

```xml
<Window.CommandBindings>
    <CommandBinding Command="ApplicationCommands.Copy"
                    Executed="OnCopyExecuted"
                    CanExecute="OnCopyCanExecute"/>
</Window.CommandBindings>

<Button Command="ApplicationCommands.Copy" Content="Copy"/>
```

```csharp
private void OnCopyCanExecute(object sender, CanExecuteRoutedEventArgs e) => e.CanExecute = true;
private void OnCopyExecuted(object sender, ExecutedRoutedEventArgs e) { /* 도메인 로직 */ }
```

---

## 스타일/트리거에서 RoutedEvent 활용

XAML 스타일에서 라우티드 이벤트를 **선언적으로** 연결 가능.

```xml
<Style TargetType="local:BadgeButton">
    <EventSetter Event="BadgeClicked" Handler="OnBadgeClicked"/>
</Style>
```

또는 **EventTrigger**로 시각 상태 전환:

```xml
<Button Content="Hover Me">
    <Button.Style>
        <Style TargetType="Button">
            <Setter Property="Background" Value="SteelBlue"/>
            <Style.Triggers>
                <EventTrigger RoutedEvent="Mouse.MouseEnter">
                    <BeginStoryboard>
                        <Storyboard>
                            <ColorAnimation Storyboard.TargetProperty="(Button.Background).(SolidColorBrush.Color)"
                                            To="RoyalBlue" Duration="0:0:0.2"/>
                        </Storyboard>
                    </BeginStoryboard>
                </EventTrigger>
            </Style.Triggers>
        </Style>
    </Button.Style>
</Button>
```

---

## **RoutedEvent 등록 시그니처** 심화

```csharp
public static readonly RoutedEvent MyEvent =
    EventManager.RegisterRoutedEvent(
        name: "My",
        routingStrategy: RoutingStrategy.Direct | RoutingStrategy.Bubble | RoutingStrategy.Tunnel,
        handlerType: typeof(MyRoutedEventHandler),
        ownerType: typeof(MyControl));
```

- `handlerType`: 반드시 **델리게이트 타입**(예: `RoutedEventHandler`, `MouseButtonEventHandler`, 커스텀 델리게이트).
- **Direct** 이벤트는 전파 안하지만 **라우티드 이벤트 인프라** 장점(스타일에서 EventSetter 사용 등)을 활용 가능.

---

## **RoutedEventArgs** 파생과 커스텀 데이터

```csharp
public delegate void BadgeChangedEventHandler(object sender, BadgeChangedEventArgs e);

public class BadgeChangedEventArgs : RoutedEventArgs
{
    public int OldCount { get; }
    public int NewCount { get; }

    public BadgeChangedEventArgs(RoutedEvent routedEvent, object source, int oldCount, int newCount)
        : base(routedEvent, source)
    {
        OldCount = oldCount;
        NewCount = newCount;
    }
}
```

이벤트 발생:

```csharp
RaiseEvent(new BadgeChangedEventArgs(BadgeChangedEvent, this, oldCount, newCount));
```

---

## **RoutedEvent와 성능·안정성 체크리스트**

1. **불필요한 전파 방지**: 적절히 `e.Handled = true` 또는 Direct 이벤트 사용.
2. **handledEventsToo** 남용주의: 이미 처리된 이벤트까지 수신하면 빈번/중복 호출 증가.
3. **OriginalSource 캐스팅** 주의: 템플릿 변형 시 타입 가정 금지. `FindAncestor`(VisualTreeHelper)로 탐색.
4. **클래스 핸들러**는 강력하지만 전역 영향 → 명확한 문서화/테스트 필수.

---

## **DP·RE를 함께 쓰는 패턴** (입력→상태→렌더)

예: 마우스 드래그로 `Value`를 업데이트하는 게이지 컨트롤.

```csharp
public class DragMeter : FrameworkElement
{
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(nameof(Value), typeof(double), typeof(DragMeter),
            new FrameworkPropertyMetadata(0.0, FrameworkPropertyMetadataOptions.AffectsRender, null, CoerceValue));

    public double Value { get => (double)GetValue(ValueProperty); set => SetValue(ValueProperty, value); }

    private static object CoerceValue(DependencyObject d, object baseValue)
    {
        double v = (double)baseValue;
        return Math.Max(0, Math.Min(1, v));
    }

    private bool _dragging;

    static DragMeter()
    {
        // 미리 클래스 핸들러로 입력 묶기
        EventManager.RegisterClassHandler(typeof(DragMeter), Mouse.PreviewMouseDownEvent,
            new MouseButtonEventHandler((s, e) =>
            {
                var self = (DragMeter)s;
                self._dragging = true;
                self.CaptureMouse();
                e.Handled = true;
            }));

        EventManager.RegisterClassHandler(typeof(DragMeter), Mouse.PreviewMouseUpEvent,
            new MouseButtonEventHandler((s, e) =>
            {
                var self = (DragMeter)s;
                self._dragging = false;
                self.ReleaseMouseCapture();
                e.Handled = true;
            }));

        EventManager.RegisterClassHandler(typeof(DragMeter), Mouse.MouseMoveEvent,
            new MouseEventHandler((s, e) =>
            {
                var self = (DragMeter)s;
                if (!self._dragging) return;
                var p = e.GetPosition(self);
                self.Value = p.X / Math.Max(1.0, self.ActualWidth); // DP 갱신 → 렌더 무효화
            }), handledEventsToo: false);
    }

    protected override void OnRender(DrawingContext dc)
    {
        var rect = new Rect(0, 0, ActualWidth, ActualHeight);
        dc.DrawRectangle(Brushes.DimGray, null, rect);
        dc.DrawRectangle(Brushes.LimeGreen, null,
            new Rect(0, 0, rect.Width * Value, rect.Height));
    }
}
```

**설명**
- 입력은 **RoutedEvent(PreviewMouseDown/Move/Up)**로 잡음.
- 상태는 **DP(Value)**로 보관 → 메타데이터 `AffectsRender`로 **자동 렌더 무효화**.
- 결과: **UI 스레드는 코드 최소**, 파이프라인과 자연스럽게 결합.

---

## DP **상속(Inherits)**

부모에서 설정한 값이 자식으로 **흘러 내려감**. 대표적으로 `FontFamily`, `FlowDirection`.

```csharp
public static readonly DependencyProperty AccentBrushProperty =
    DependencyProperty.Register(nameof(AccentBrush), typeof(Brush), typeof(AccentScope),
        new FrameworkPropertyMetadata(Brushes.SteelBlue, FrameworkPropertyMetadataOptions.Inherits | FrameworkPropertyMetadataOptions.AffectsRender));

public Brush AccentBrush
{
    get => (Brush)GetValue(AccentBrushProperty);
    set => SetValue(AccentBrushProperty, value);
}
```

```xml
<local:AccentScope AccentBrush="Tomato">
    <!-- 내부 모든 하위 요소는 별도 로컬 값이 없으면 AccentBrush를 상속 -->
</local:AccentScope>
```

> **주의**: 상속은 편리하지만 트리 규모가 클수록 전파 비용이 커질 수 있음. 변경 빈도 높은 속성엔 신중하게.

---

## DP **애니메이션**과의 상호작용

애니메이션은 **DP에 덮어씌우는 레이어**. Local/Style 값보다 **우선**.

```csharp
var brush = new SolidColorBrush(Colors.SteelBlue);
if (brush.CanFreeze) brush.Freeze(); // 애니메이션할 Color는 Freeze 금지(동적 변경 필요) 주의!

var animBrush = new SolidColorBrush(Colors.SteelBlue);
var colorAnim = new ColorAnimation(Colors.SteelBlue, Colors.RoyalBlue, TimeSpan.FromSeconds(0.3))
{
    AutoReverse = true, RepeatBehavior = RepeatBehavior.Forever
};
animBrush.BeginAnimation(SolidColorBrush.ColorProperty, colorAnim);
myShape.Fill = animBrush; // DP 값으로 설정
```

---

## 고급: **DependencyPropertyDescriptor**와 디자인 타임

디자이너/도구·복잡 시나리오에서 DP 변화를 **관찰**.

```csharp
var dpd = DependencyPropertyDescriptor.FromProperty(MeterBar.ValueProperty, typeof(MeterBar));
dpd.AddValueChanged(myBar, (_, __) => Debug.WriteLine($"Value changed: {myBar.Value}"));
```

> 내부적으로는 DP 변경에 연결된 **약참조(weak)** 기반 콜백을 사용.

---

## **메모리/성능** 관점에서의 DP

- **희소 저장**: 기본값은 메타테이블 참조, **로컬 값만** 엔트리 보유 → 많은 객체에도 메모리 절감.
- **Freeze 가능한 Freezable** 사용: 변경 불가로 만들어 **스냅샷·공유** 성능 향상.
- **빈번한 Coerce/PropertyChanged 비용**을 의식: 짧고 빠르게, 필요한 경우에만.

---

## RoutedEvent **디버깅**과 흔한 함정

- **핸들러가 안 불린다**: 중간에서 `e.Handled = true`로 차단됐을 수 있음. `handledEventsToo:true`로 검사.
- **OriginalSource 캐스팅 예외**: 템플릿 내부 요소로 들어갈 수 있음. 시각 트리를 따라 조심스럽게 탐색.
- **중복 핸들링**: 같은 라우트에 여러 위치에서 로직 중복. **핸들링 책임**을 명확히.

---

## 예제: **템플릿 내부 요소 이벤트를 상위에서 처리**

`ListBoxItem` 템플릿 안의 `Button` 클릭을 **상위 ListBox**에서 모두 처리하고 싶다.

```xml
<ListBox x:Name="Files" ItemsSource="{Binding Items}">
    <ListBox.ItemTemplate>
        <DataTemplate>
            <StackPanel Orientation="Horizontal">
                <TextBlock Text="{Binding Name}" Margin="0,0,8,0"/>
                <Button Content="Delete" />
            </StackPanel>
        </DataTemplate>
    </ListBox.ItemTemplate>
</ListBox>
```

```csharp
Files.AddHandler(Button.ClickEvent, new RoutedEventHandler((s, e) =>
{
    if (e.OriginalSource is Button btn)
    {
        var container = ItemsControl.ContainerFromElement(Files, btn) as ListBoxItem;
        var item = (MyItem)Files.ItemContainerGenerator.ItemFromContainer(container);
        Delete(item);
        e.Handled = true;
    }
}));
```

**포인트**
- **버블링**으로 상위에서 한 번에 처리.
- 템플릿 구조 변경에도 상위 로직은 변경 최소화.

---

## 예제: **Preview 키 입력 차단·필터**

숫자만 입력 가능한 TextBox를 **상위 Grid**에서 한 번에 통제.

```csharp
this.PreviewTextInput += (s, e) =>
{
    if (!e.Text.All(char.IsDigit))
        e.Handled = true; // 터널링에서 차단 → 하위 TextBox까지 전달되지 않음
};
```

> IME/붙여넣기 등 복합 입력은 `DataObject.Pasting`, `PreviewKeyDown` 등 추가 고려.

---

## **이벤트 트리거** vs **스타일 트리거** vs **코드** 선택 기준

- **순수 시각 변화**(색/Opacity/Transform 등) → **EventTrigger/Storyboard** 권장.
- **도메인 로직**(모델 변경/IO/명령) → **CommandBinding/코드 비하인드**.
- **구성 가능성**(테마/제어 확장) → **Style EventSetter / Class Handler**.

---

## **접근성(A11y)** 관점의 RE/DP

- RE: **터널링** 단계에서 키보드 포커스 이동/차단, 스크린리더 이벤트 디스패치에 영향.
- DP: `AutomationProperties.Name/HelpText` 등 **Attached DP**로 UIA 속성 주입.

```xml
<Button Content="Buy"
        AutomationProperties.Name="Buy Button"
        AutomationProperties.HelpText="Adds the item to the cart"/>
```

---

## **테스트/진단** 팁

- **UIAutomation**으로 라우트/포커스 흐름 점검.
- **PresentationTraceSources.TraceLevel=High**로 바인딩/DP 트레이스.
- **Snoop/Live Visual Tree** 등 도구로 **OriginalSource/Source** 확인.
- 프레임 드랍 시 **핵심 의심**: 과도한 DP 변경(특히 AffectsMeasure), 과도한 RE 수신.

---

## 미니 종합 데모: **Badge List** (RE로 입력, DP로 상태, 자동 렌더)

### Badge 모델·VM

```csharp
public class BadgeItem : INotifyPropertyChanged
{
    private int _count;
    public string Name { get; }
    public int Count { get => _count; set { _count = value; OnPropertyChanged(); } }
    // INPC 구현 생략
    public BadgeItem(string name, int count) { Name = name; Count = count; }
}
```

### BadgeButton(커스텀 컨트롤): RE + DP

```csharp
public class BadgeButton : Button
{
    public static readonly DependencyProperty BadgeCountProperty =
        DependencyProperty.Register(nameof(BadgeCount), typeof(int), typeof(BadgeButton),
            new FrameworkPropertyMetadata(0, FrameworkPropertyMetadataOptions.AffectsRender));

    public int BadgeCount
    {
        get => (int)GetValue(BadgeCountProperty);
        set => SetValue(BadgeCountProperty, value);
    }

    public static readonly RoutedEvent BadgeClickedEvent =
        EventManager.RegisterRoutedEvent("BadgeClicked", RoutingStrategy.Bubble,
            typeof(RoutedEventHandler), typeof(BadgeButton));

    public event RoutedEventHandler BadgeClicked
    {
        add => AddHandler(BadgeClickedEvent, value);
        remove => RemoveHandler(BadgeClickedEvent, value);
    }

    protected override void OnClick()
    {
        RaiseEvent(new RoutedEventArgs(BadgeClickedEvent, this));
        base.OnClick();
    }

    protected override void OnRender(DrawingContext dc)
    {
        base.OnRender(dc);
        if (BadgeCount <= 0) return;
        var r = 10.0;
        var x = ActualWidth - r - 4;
        var y = r + 4;
        dc.DrawEllipse(Brushes.Crimson, null, new Point(x, y), r, r);
        var ft = new FormattedText(
            BadgeCount.ToString(), System.Globalization.CultureInfo.CurrentUICulture,
            FlowDirection.LeftToRight, new Typeface("Segoe UI"), 10, Brushes.White, VisualTreeHelper.GetDpi(this).PixelsPerDip);
        dc.DrawText(ft, new Point(x - ft.Width / 2, y - ft.Height / 2));
    }
}
```

### 상위 창에서 버블링 처리

```csharp
// 상위 컨테이너에서 모든 BadgeButton 클릭 처리
root.AddHandler(BadgeButton.BadgeClickedEvent, new RoutedEventHandler((s, e) =>
{
    if (e.OriginalSource is BadgeButton bb && bb.DataContext is BadgeItem item)
    {
        item.Count = Math.Max(0, item.Count - 1); // DP 렌더 → VM 값 변경로직 연계
        e.Handled = true;
    }
}));
```

### XAML

```xml
<ItemsControl ItemsSource="{Binding Badges}">
    <ItemsControl.ItemTemplate>
        <DataTemplate>
            <local:BadgeButton Content="{Binding Name}" BadgeCount="{Binding Count}" Margin="4"/>
        </DataTemplate>
    </ItemsControl.ItemTemplate>
</ItemsControl>
```

**동작 흐름**
- 사용자가 BadgeButton 클릭(RE 발생) → 상위에서 한 곳에서 처리 → VM Count 감소 → 바인딩으로 DP 갱신 → `AffectsRender` → 뱃지 재그림.

---

## FAQ

**Q1. 일반 C# 속성 vs DP를 언제 고를까?**
- 템플릿/스타일/바인딩/애니메이션/상속이 필요 없고, 내부 로직용이면 **일반 속성**.
- 외부에 공개되어 **WPF 시스템과 상호작용**해야 하면 **DP**.

**Q2. RE와 일반 .NET 이벤트 차이는?**
- RE는 **트리 전파**와 스타일/클래스 핸들러/명령 라우팅 통합.
- 일반 이벤트는 **발생한 곳에서만** 수신.

**Q3. `e.Handled = true`를 남발해도 되나요?**
- 아니요. 너무 일찍/광범위하게 차단하면 **재사용성·디버깅성** 저하. 꼭 필요한 지점에서만.

**Q4. DP 기본값으로 참조 타입(예: Brush)을 넣어도 되나요?**
- 가능하지만 **공유 인스턴스**가 됨. 변경 위험 → **Freezable은 Freeze**, 변형 필요한 경우 **콜백에서 새로 생성**.

---

## 체크리스트 (실전 적용 전)

- [ ] 컨트롤 공개 속성은 **DP**로 만들었는가? (바인딩/스타일/애니메이션 필요 여부)
- [ ] **AffectsMeasure/Arrange/Render** 플래그가 올바른가?
- [ ] **Validate/Coerce/Changed**에서 **무거운 작업** 하지 않는가?
- [ ] **Freeze** 가능한 리소스는 Freeze 했는가?
- [ ] RoutedEvent 핸들러가 **올바른 전파 단계**(Preview vs Bubble)에 위치했는가?
- [ ] `handledEventsToo`를 꼭 필요한 곳에서만 썼는가?
- [ ] `OriginalSource` 의존 코드는 템플릿 변경에 안전한가?

---

## 마무리

- **DependencyProperty**는 “속성의 생애주기 전체(값 계산 → 무효화 → 전파)”를 시스템 차원에서 다룹니다.
- **RoutedEvent**는 “이벤트의 생애주기 전체(발생 → 전파 → 핸들링 우선순위)”를 트리 차원에서 다룹니다.
- 둘을 올바르게 설계하면, **입력 → 상태 → 렌더**의 파이프라인이 최소 비용으로 깔끔하게 돌아갑니다.

> 오늘 당장 할 일:
> 1) 컨트롤의 공개 속성 중 DP로 바꿔야 할 것 체크.
> 2) 자주 바뀌는 속성에 **Affects…** 플래그 재검토.
> 3) 전역 입력 로직을 **Preview** 단계에서 통제할지, **Bubble**에서 수집할지 결정.
> 4) 바인딩 에러 로그를 반드시 **0**으로.

---

### **Get/SetCurrentValue**: Local Value를 덮지 않고 값 갱신

`SetCurrentValue`는 **로컬 값 우선순위를 깨지 않고** 현재 유효값만 갱신.

```csharp
// 템플릿/스타일 값 위에 "일시적으로" 값만 바꾸고 싶을 때
myControl.SetCurrentValue(Control.BackgroundProperty, Brushes.LightYellow);
```

---

### **ClearValue**와 `ReadLocalValue`

```csharp
// 로컬 값 제거 → 우선순위 아래(스타일/상속/기본값)로 복귀
myControl.ClearValue(Control.BackgroundProperty);

// 로컬 값이 있는지 점검
var lv = myControl.ReadLocalValue(Control.BackgroundProperty);
if (lv != DependencyProperty.UnsetValue)
    Debug.WriteLine($"Local value exists: {lv}");
```

---

### 템플릿 경계에서 `Source` 보정

`OriginalSource`는 템플릿 내부 요소일 수 있으므로, **항목 컨테이너**를 역추적:

```csharp
DependencyObject FindContainer(ItemsControl ic, DependencyObject start)
{
    while (start != null && !(start is ListBoxItem))
        start = VisualTreeHelper.GetParent(start);
    return start;
}
```

---

### XAML에서 **Attached Event** (RE의 부착 버전)

```xml
<Grid Mouse.MouseDown="OnAnyMouseDown"/>
```

```csharp
private void OnAnyMouseDown(object sender, MouseButtonEventArgs e)
{
    // MouseDownEvent는 버블링. Grid에서 한 번에 잡는다.
}
```

---

### 스타일 기반 EventSetter의 “클래스 핸들러 대용”

```xml
<Style TargetType="TextBox">
    <EventSetter Event="PreviewKeyDown" Handler="TextBox_PreviewKeyDown"/>
</Style>
```

> 모든 TextBox에 공통 로직 적용. 라이브러리 수준에선 **RegisterClassHandler**가 더 강력.

---

## 끝

이제 **DependencyProperty**와 **RoutedEvent**를 “API”가 아닌 “파이프라인의 계약”으로 보세요.
그 계약을 지키면, 복잡한 UI도 **적은 코드**로 **빠르고 부드럽게** 유지할 수 있습니다.
