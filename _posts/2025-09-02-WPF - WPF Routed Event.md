---
layout: post
title: WPF - WPF Routed Event
date: 2025-09-02 20:25:23 +0900
category: WPF
---
# WPF Routed Event 이해하기

## Routed Event란 무엇인가?

**Routed Event**(라우트된 이벤트)는 WPF의 독특한 이벤트 시스템으로, 이벤트가 발생한 지점에서 멈추지 않고 시각적 트리를 따라 이동하며 전파됩니다. 이는 기존 WinForms의 CLR 이벤트와 구별되는 WPF의 핵심 특징 중 하나입니다.

### 세 가지 라우팅 전략

1. **Direct(직접)**: 이벤트가 발생한 요소에서만 처리됩니다. WinForms의 전통적 이벤트 처리 방식과 유사합니다.
2. **Bubbling(버블링)**: 이벤트가 자식 요소에서 시작하여 부모 요소로 상향 전파됩니다.
3. **Tunneling(터널링)**: 이벤트가 루트에서 시작하여 자식 요소로 하향 전파됩니다. WPF 관례상 이름에 `Preview` 접두사를 사용합니다.

### 왜 라우트된 이벤트가 필요한가?

- **관심사 분리**: 하위 컨트롤의 내부 구현을 알지 못해도 상위 레벨에서 일괄적으로 이벤트를 처리할 수 있습니다.
- **입력 정책 관리**: 윈도우나 페이지 수준에서 Preview 이벤트를 통해 입력을 사전에 필터링하거나 차단할 수 있습니다.
- **템플릿 재사용**: ControlTemplate 내부의 자식 요소들이 발생시킨 이벤트를 외부에서 컨트롤 자체의 이벤트처럼 처리할 수 있습니다.

## 기본 동작 이해하기

### 실전 예제: 버블링과 터널링의 흐름

다음 XAML 구조에서 버튼을 클릭할 때의 이벤트 전파 순서를 살펴보겠습니다:

```xml
<Window x:Class="DemoRoutedEvents.MainWindow"
        Title="Routed Events Demo"
        PreviewMouseDown="Window_PreviewMouseDown"
        MouseDown="Window_MouseDown">
  <Grid PreviewMouseDown="Grid_PreviewMouseDown" MouseDown="Grid_MouseDown">
    <Border PreviewMouseDown="Border_PreviewMouseDown" MouseDown="Border_MouseDown">
      <Button Content="Click Here"
              PreviewMouseDown="Button_PreviewMouseDown"
              MouseDown="Button_MouseDown"
              Click="Button_Click" />
    </Border>
  </Grid>
</Window>
```

버튼 위에서 마우스를 클릭하면 이벤트는 다음과 같은 순서로 처리됩니다:

1. **터널링 단계 (Preview)**: `Window.PreviewMouseDown` → `Grid.PreviewMouseDown` → `Border.PreviewMouseDown` → `Button.PreviewMouseDown`
2. **버블링 단계**: `Button.MouseDown` → `Border.MouseDown` → `Grid.MouseDown` → `Window.MouseDown`
3. **Click 이벤트**: 버튼의 클릭 제스처가 완료되면 `Button.Click` 이벤트가 버블링 방식으로 발생합니다.

### Source vs OriginalSource 이해하기

라우트된 이벤트에는 두 가지 중요한 속성이 있습니다:
- **`Source`**: 이벤트 핸들러 관점에서의 논리적 발신자입니다.
- **`OriginalSource`**: 실제 히트 테스트를 통과한 가장 안쪽의 시각적 요소입니다. 템플릿 내부 요소일 수 있습니다.

예를 들어, 버튼의 템플릿 내부에 있는 `ContentPresenter`를 클릭하면:
- `OriginalSource`는 `ContentPresenter`가 됩니다.
- `Source`는 여전히 `Button`이 됩니다.

## Handled 속성의 중요성

`e.Handled = true`를 설정하면 해당 이벤트는 "처리 완료" 상태가 되어 동일한 라우트의 이후 단계에서는 더 이상 호출되지 않습니다.

### Handled 차단 예시

```csharp
private void Button_PreviewMouseDown(object sender, MouseButtonEventArgs e)
{
    // 버튼 단계에서 이벤트 처리 완료 선언
    e.Handled = true;
    Debug.WriteLine("Button PreviewMouseDown - 이벤트 차단됨");
}
// 결과: 이후의 모든 MouseDown 이벤트 핸들러는 호출되지 않음
```

### Handled 이벤트 강제 수신

가끔은 다른 코드에서 `Handled=true`로 설정한 이벤트도 수신해야 할 때가 있습니다. 이럴 때는 `AddHandler` 메서드에 `handledEventsToo` 매개변수를 사용합니다:

```csharp
public MainWindow()
{
    InitializeComponent();
    // Handled로 처리된 이벤트도 강제로 수신
    AddHandler(UIElement.MouseDownEvent,
               new MouseButtonEventHandler(CatchAll_MouseDown),
               handledEventsToo: true);
}

private void CatchAll_MouseDown(object sender, MouseButtonEventArgs e)
{
    Debug.WriteLine("모든 MouseDown 이벤트를 수신합니다 (Handled 여부와 관계없이)");
}
```

이 기법은 특히 서드파티 컨트롤이 내부적으로 이벤트를 처리한 경우에도 로깅이나 분석을 위해 이벤트를 수신해야 할 때 유용합니다.

## 템플릿 내부 이벤트를 컨트롤 이벤트로 변환하기

컨트롤 템플릿 내부의 요소가 발생시킨 이벤트를 컨트롤 외부에서 처리하고 싶을 때는 클래스 핸들러를 사용합니다:

```csharp
public class Card : Control
{
    static Card()
    {
        // Card 컨트롤 내부에서 발생하는 모든 Button.Click 이벤트를 가로챔
        EventManager.RegisterClassHandler(typeof(Card),
                                          Button.ClickEvent,
                                          new RoutedEventHandler(OnAnyButtonClickInside));
    }

    private static void OnAnyButtonClickInside(object sender, RoutedEventArgs e)
    {
        if (sender is Card card && e.OriginalSource is Button button)
        {
            if (button.Name == "PART_Action")
            {
                // 외부에 노출할 커스텀 이벤트 발생
                card.RaiseEvent(new RoutedEventArgs(ActionClickedEvent, card));
                e.Handled = true; // 이 컨트롤에서 소비 완료
            }
        }
    }

    // 커스텀 라우트된 이벤트 정의
    public static readonly RoutedEvent ActionClickedEvent =
        EventManager.RegisterRoutedEvent("ActionClicked", 
                                         RoutingStrategy.Bubble,
                                         typeof(RoutedEventHandler), 
                                         typeof(Card));

    public event RoutedEventHandler ActionClicked
    {
        add => AddHandler(ActionClickedEvent, value);
        remove => RemoveHandler(ActionClickedEvent, value);
    }
}
```

이 방식의 장점은 템플릿이 변경되더라도 컨트롤의 외부 인터페이스가 일관되게 유지된다는 점입니다.

## 커스텀 라우트된 이벤트 만들기

직접 라우트된 이벤트를 정의하고 사용하는 방법은 다음과 같습니다:

```csharp
public class Stepper : Control
{
    // 1. 라우트된 이벤트 등록
    public static readonly RoutedEvent StepChangedEvent =
        EventManager.RegisterRoutedEvent(
            name: "StepChanged",
            routingStrategy: RoutingStrategy.Bubble,
            handlerType: typeof(RoutedPropertyChangedEventHandler<int>),
            ownerType: typeof(Stepper));

    // 2. CLR 이벤트 래퍼 제공
    public event RoutedPropertyChangedEventHandler<int> StepChanged
    {
        add => AddHandler(StepChangedEvent, value);
        remove => RemoveHandler(StepChangedEvent, value);
    }

    private int _step;
    public int Step
    {
        get => _step;
        set
        {
            if (_step == value) return;
            
            int oldValue = _step;
            _step = value;

            // 3. 이벤트 발생
            var args = new RoutedPropertyChangedEventArgs<int>(oldValue, value, StepChangedEvent);
            RaiseEvent(args);
        }
    }
}
```

XAML에서 이 이벤트를 사용하는 방법:
```xml
<local:Stepper StepChanged="Stepper_StepChanged"/>
```

## 실전 활용 패턴

### 글로벌 키 필터링

윈도우 수준에서 Preview 이벤트를 사용하여 애플리케이션 전역의 키 입력을 처리할 수 있습니다:

```csharp
private void Window_PreviewKeyDown(object sender, KeyEventArgs e)
{
    if (e.Key == Key.Escape && IsDialogOpen)
    {
        CloseDialog();
        e.Handled = true; // 하위 컨트롤들이 이 키 입력을 받지 않음
    }
    
    if (e.Key == Key.F1)
    {
        ShowHelp();
        e.Handled = true;
    }
}
```

### 모달 오버레이 구현

터널링 이벤트를 활용하여 모달 다이얼로그의 배경 클릭을 차단하는 패턴:

```xml
<Grid>
  <!-- 메인 콘텐츠 -->
  <ContentPresenter x:Name="MainContent"/>
  
  <!-- 모달 오버레이 -->
  <Border x:Name="ModalOverlay" Background="#80000000" Visibility="Collapsed"
          PreviewMouseDown="ModalOverlay_PreviewMouseDown"/>
  
  <!-- 모달 다이얼로그 -->
  <Border x:Name="ModalDialog" Visibility="Collapsed"/>
</Grid>
```

```csharp
private void ModalOverlay_PreviewMouseDown(object sender, MouseButtonEventArgs e)
{
    // 오버레이 클릭 시 이벤트 차단 (다이얼로그 외부 클릭 방지)
    e.Handled = true;
}
```

## 문제 해결 가이드

라우트된 이벤트 관련 문제가 발생했을 때 확인해야 할 사항들:

1. **이벤트가 전혀 도착하지 않는 경우**
   - 다른 핸들러에서 `e.Handled = true`를 설정했는지 확인
   - `AddHandler(..., handledEventsToo: true)`로 강제 수신 시도
   - 포커스가 올바른 요소에 있는지 확인

2. **이벤트 순서가 예상과 다른 경우**
   - 터널링 → 버블링의 기본 순서 이해
   - 클래스 핸들러와 인스턴스 핸들러의 실행 순서 확인

3. **템플릿 내부 요소의 이벤트를 잡지 못하는 경우**
   - `OriginalSource`를 사용하여 실제 발신자 확인
   - `FindAncestor<T>()` 메서드로 원하는 타입까지 트리를 탐색

## 성능 고려사항

라우트된 이벤트 시스템은 강력하지만 잘못 사용하면 성능 문제를 일으킬 수 있습니다:

- **과도한 이벤트 핸들러 등록 피하기**: 윈도우 수준에 너무 많은 핸들러를 등록하면 모든 입력에 대해 불필요한 호출이 발생합니다.
- **적절한 이벤트 선택**: 모든 `MouseMove`를 수신하는 대신 실제로 필요한 이벤트만 구독하세요.
- **조기 필터링**: 상위 핸들러에서 가능한 한 빨리 불필요한 이벤트를 필터링하세요.

## 결론

WPF의 라우트된 이벤트 시스템은 단순한 이벤트 전달 메커니즘을 넘어, 애플리케이션의 입력 정책을 체계적으로 관리할 수 있는 강력한 도구입니다. 터널링과 버블링의 이중 구조를 이해하고, `Handled` 속성을 적절히 활용하며, 클래스 핸들러를 통해 템플릿과의 결합도를 낮추는 것이 효과적인 WPF 개발의 핵심입니다.

실제 프로젝트에서는 다음과 같은 원칙을 기억하세요:
- **Preview(터널링)은 필터링과 차단에**, **일반 이벤트(버블링)은 실제 동작 수행에** 사용하세요.
- 다른 컨트롤이나 라이브러리에서 처리한 이벤트도 필요하면 `handledEventsToo` 옵션으로 수신하세요.
- 템플릿 기반 컨트롤을 설계할 때는 클래스 핸들러를 활용하여 내부 구현을 캡슐화하세요.

이러한 개념들을 잘 이해하고 적용하면 보다 유연하고 견고한 WPF 애플리케이션을 구축할 수 있을 것입니다.