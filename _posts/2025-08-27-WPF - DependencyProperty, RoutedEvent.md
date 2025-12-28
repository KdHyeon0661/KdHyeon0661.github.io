---
layout: post
title: WPF - DependencyProperty, RoutedEvent
date: 2025-08-27 18:25:23 +0900
category: WPF
---
# WPF의 핵심: DependencyProperty와 RoutedEvent 심층 해부

## WPF의 근간을 이루는 두 개의 기둥

WPF가 다른 UI 프레임워크와 구별되는 가장 중요한 특징 중 하나는 바로 **DependencyProperty(의존 속성)**와 **RoutedEvent(라우트된 이벤트)** 시스템입니다. 이 두 가지 메커니즘은 단순한 기술적 구현을 넘어 WPF의 전체 아키텍처 철학을 반영합니다. 그들이 제공하는 강력한 기능 없이는 WPF의 데이터 바인딩, 스타일링, 애니메이션, 템플릿 시스템 모두가 무너집니다.

```csharp
// DependencyProperty의 본질: 속성 시스템의 재정의
public static readonly DependencyProperty ValueProperty = 
    DependencyProperty.Register(
        "Value", 
        typeof(double), 
        typeof(MyControl),
        new FrameworkPropertyMetadata(
            0.0,
            FrameworkPropertyMetadataOptions.AffectsRender,
            OnValueChanged));

// RoutedEvent의 본질: 이벤트 시스템의 재정의  
public static readonly RoutedEvent ValueChangedEvent =
    EventManager.RegisterRoutedEvent(
        "ValueChanged",
        RoutingStrategy.Bubble,
        typeof(RoutedPropertyChangedEventHandler<double>),
        typeof(MyControl));
```

## DependencyProperty: 속성 시스템의 혁명

### 일반 .NET 속성의 한계를 넘어서기

전통적인 .NET 속성은 단순한 getter/setter 쌍에 불과합니다. 하지만 현대적 UI 프레임워크에서는 속성에 대한 더 풍부한 의미와 동작이 필요합니다:

```csharp
// 전통적 속성 - 기능이 제한적
public class TraditionalControl
{
    private double _value;
    
    public double Value
    {
        get => _value;
        set
        {
            if (_value != value)
            {
                _value = value;
                // 변경 알림? 바인딩? 애니메이션? 스타일? 모두 수동 구현 필요
                OnValueChanged();
            }
        }
    }
    
    private void OnValueChanged()
    {
        // 렌더링 갱신, 레이아웃 재계산 등 모두 수동 처리
    }
}

// DependencyProperty - 시스템이 모든 것을 관리
public class ModernControl : FrameworkElement
{
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(
            "Value",
            typeof(double),
            typeof(ModernControl),
            new FrameworkPropertyMetadata(
                0.0,
                FrameworkPropertyMetadataOptions.AffectsRender,  // 자동 렌더 갱신
                OnValueChanged,                                 // 변경 콜백
                CoerceValue));                                 // 값 강제 조정
    
    public double Value
    {
        get => (double)GetValue(ValueProperty);
        set => SetValue(ValueProperty, value);
    }
    
    // 시스템이 호출하는 콜백
    private static void OnValueChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        // WPF가 이미 렌더링을 예약했음
    }
}
```

### DependencyProperty의 마법 같은 특징들

#### 1. 값 우선순위 계층 구조
DependencyProperty는 여러 출처에서 값을 받을 수 있으며, 우선순위에 따라 최종 값을 결정합니다:

```csharp
public class PriorityDemonstration
{
    public void DemonstrateValuePrecedence()
    {
        var control = new MyControl();
        
        // 7. 기본값 (Default Value)
        // 생성 시점: Value = 0.0 (메타데이터에서 정의됨)
        
        // 6. 상속된 값 (Inherited Value)
        // 부모 컨테이너에서 설정된 값이 자식으로 흐름
        control.SetValue(TextElement.FontFamilyProperty, new FontFamily("Segoe UI"));
        
        // 5. 테마 스타일 (Theme Style)
        // 테마에서 정의된 기본 스타일 적용
        
        // 4. 스타일 Setter (Style Setter)
        control.Style = new Style(typeof(MyControl));
        control.Style.Setters.Add(new Setter(MyControl.ValueProperty, 0.5));
        
        // 3. 템플릿 트리거 (Template Trigger)
        // ControlTemplate 내부의 Trigger가 값 설정
        
        // 2. 로컬 값 (Local Value) - 직접 설정
        control.Value = 0.7; // 또는 SetValue(ValueProperty, 0.7)
        
        // 1. 활성 애니메이션 (Active Animation) - 최고 우선순위
        var animation = new DoubleAnimation(0, 1, TimeSpan.FromSeconds(2));
        control.BeginAnimation(MyControl.ValueProperty, animation);
        
        // 이 시점에서 화면에 표시되는 값은 애니메이션에 의해 결정됨
        // 애니메이션이 완료되면 로컬 값(0.7)으로 돌아감
    }
}
```

#### 2. 효율적인 메모리 관리
DependencyObject는 속성 값을 희소 배열(sparse array)에 저장하여 메모리를 효율적으로 관리합니다:

```csharp
public class MemoryEfficiency
{
    public void ShowMemoryUsage()
    {
        // 1000개의 컨트롤 생성
        var controls = new List<MyControl>();
        for (int i = 0; i < 1000; i++)
        {
            controls.Add(new MyControl());
        }
        
        // 기본값만 사용 시: 추가 메모리 사용 거의 없음
        // 기본값은 메타데이터 테이블에 저장되고 모든 인스턴스가 참조
        
        // 일부 인스턴스에만 로컬 값 설정
        controls[0].Value = 1.0;  // 인덱스 0만 로컬 값 저장
        controls[500].Value = 0.5; // 인덱스 500만 로컬 값 저장
        
        // 나머지 998개 인스턴스는 여전히 기본값 참조
        // 메모리 절약 효과!
    }
}
```

#### 3. 강력한 유효성 검사와 값 강제
```csharp
public class RangeControl : FrameworkElement
{
    // 속성 정의
    public static readonly DependencyProperty MinimumProperty =
        DependencyProperty.Register("Minimum", typeof(double), typeof(RangeControl),
            new FrameworkPropertyMetadata(0.0, OnMinMaxChanged));
    
    public static readonly DependencyProperty MaximumProperty =
        DependencyProperty.Register("Maximum", typeof(double), typeof(RangeControl),
            new FrameworkPropertyMetadata(1.0, OnMinMaxChanged));
    
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register("Value", typeof(double), typeof(RangeControl),
            new FrameworkPropertyMetadata(0.0, 
                FrameworkPropertyMetadataOptions.BindsTwoWayByDefault,
                OnValueChanged, 
                CoerceValue));
    
    // 값 강제(coercion) - 다른 속성에 의존
    private static object CoerceValue(DependencyObject d, object baseValue)
    {
        var control = (RangeControl)d;
        double value = (double)baseValue;
        
        // Minimum과 Maximum 사이로 강제
        return Math.Max(control.Minimum, Math.Min(control.Maximum, value));
    }
    
    // Minimum/Maximum 변경 시 Value 재강제
    private static void OnMinMaxChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        var control = (RangeControl)d;
        control.CoerceValue(ValueProperty); // ValueProperty 재강제
    }
    
    // 유효성 검사
    private static bool ValidateValue(object value)
    {
        double v = (double)value;
        return !double.IsNaN(v) && !double.IsInfinity(v);
    }
}
```

### Attached Property: 다른 객체에 속성을 "붙이는" 마법

```csharp
// Attached Property 정의
public static class LayoutExtensions
{
    public static readonly DependencyProperty SpacingProperty =
        DependencyProperty.RegisterAttached(
            "Spacing",
            typeof(double),
            typeof(LayoutExtensions),
            new FrameworkPropertyMetadata(0.0, FrameworkPropertyMetadataOptions.AffectsArrange));
    
    public static void SetSpacing(UIElement element, double value)
        => element.SetValue(SpacingProperty, value);
    
    public static double GetSpacing(UIElement element)
        => (double)element.GetValue(SpacingProperty);
}

// 사용 예
public class CustomPanel : Panel
{
    protected override Size ArrangeOverride(Size finalSize)
    {
        double spacing = LayoutExtensions.GetSpacing(this);
        // spacing을 사용한 레이아웃 계산
        return finalSize;
    }
}

// XAML에서 사용
<local:CustomPanel local:LayoutExtensions.Spacing="10">
    <Button Content="Button 1"/>
    <Button Content="Button 2"/>
</local:CustomPanel>
```

## RoutedEvent: 이벤트 시스템의 재발명

### 전통적 이벤트의 문제점 해결하기

일반 .NET 이벤트는 직접적인 발행자-구독자 관계를 가집니다. 하지만 복잡한 UI 트리에서는 이 접근 방식이 제한적입니다:

```csharp
// 전통적 접근법 - 각 컨트롤에 개별 핸들러 필요
public class TraditionalUI
{
    public void SetupEventHandlers()
    {
        var button1 = new Button();
        var button2 = new Button();
        var button3 = new Button();
        
        button1.Click += OnButton1Click;  // 각 버튼별 핸들러
        button2.Click += OnButton2Click;
        button3.Click += OnButton3Click;
        
        // 새로운 버튼이 추가될 때마다 핸들러도 추가해야 함
    }
}

// RoutedEvent 접근법 - 한 곳에서 모든 처리 가능
public class ModernUI
{
    public void SetupEventHandlers()
    {
        var container = new StackPanel();
        container.AddHandler(Button.ClickEvent, 
            new RoutedEventHandler(OnAnyButtonClick));  // 모든 버튼 클릭 처리
        
        container.Children.Add(new Button { Content = "Button 1" });
        container.Children.Add(new Button { Content = "Button 2" });
        container.Children.Add(new Button { Content = "Button 3" });
        
        // 새로운 버튼 추가해도 별도 핸들러 불필요
    }
    
    private void OnAnyButtonClick(object sender, RoutedEventArgs e)
    {
        var button = (Button)e.OriginalSource;
        Console.WriteLine($"{button.Content} was clicked!");
    }
}
```

### RoutedEvent의 세 가지 라우팅 전략

#### 1. 터널링(Tunneling) - Preview 이벤트
```csharp
public class TunnelingExample
{
    public void SetupTunnelingHandlers()
    {
        var window = new Window();
        var grid = new Grid();
        var button = new Button { Content = "Click Me" };
        
        grid.Children.Add(button);
        window.Content = grid;
        
        // 터널링: Window → Grid → Button (위에서 아래로)
        window.PreviewMouseDown += (s, e) => 
            Console.WriteLine("Window: PreviewMouseDown (터널링)");
        
        grid.PreviewMouseDown += (s, e) => 
            Console.WriteLine("Grid: PreviewMouseDown (터널링)");
        
        button.PreviewMouseDown += (s, e) => 
            Console.WriteLine("Button: PreviewMouseDown (터널링)");
        
        // 버블링: Button → Grid → Window (아래에서 위로)
        button.MouseDown += (s, e) => 
            Console.WriteLine("Button: MouseDown (버블링)");
        
        grid.MouseDown += (s, e) => 
            Console.WriteLine("Grid: MouseDown (버블링)");
        
        window.MouseDown += (s, e) => 
            Console.WriteLine("Window: MouseDown (버블링)");
        
        // 버튼 클릭 시 출력 순서:
        // 1. Window: PreviewMouseDown (터널링)
        // 2. Grid: PreviewMouseDown (터널링)
        // 3. Button: PreviewMouseDown (터널링)
        // 4. Button: MouseDown (버블링)
        // 5. Grid: MouseDown (버블링)
        // 6. Window: MouseDown (버블링)
    }
}
```

#### 2. 버블링(Bubbling) - 표준 이벤트
```csharp
public class BubblingExample
{
    public void DemonstrateEventBubbling()
    {
        var listBox = new ListBox();
        
        // ListBoxItem 템플릿 내부의 Button 클릭 처리
        listBox.AddHandler(Button.ClickEvent, 
            new RoutedEventHandler((sender, e) =>
            {
                // e.OriginalSource: 실제 클릭된 Button
                // e.Source: 이벤트가 재정의된 소스 (템플릿 경계에서 다를 수 있음)
                
                if (e.OriginalSource is Button button)
                {
                    // 클릭된 버튼을 포함하는 ListBoxItem 찾기
                    var container = FindAncestor<ListBoxItem>(button);
                    if (container != null)
                    {
                        var item = listBox.ItemContainerGenerator
                            .ItemFromContainer(container);
                        Console.WriteLine($"Item clicked: {item}");
                    }
                }
                
                e.Handled = true; // 이벤트 처리 완료 표시
            }));
    }
    
    private T FindAncestor<T>(DependencyObject current) where T : DependencyObject
    {
        while (current != null && !(current is T))
        {
            current = VisualTreeHelper.GetParent(current);
        }
        return current as T;
    }
}
```

#### 3. 직접(Direct) - 전파 없음
```csharp
public class DirectEventExample
{
    // 직접 이벤트 정의
    public static readonly RoutedEvent DirectNotificationEvent =
        EventManager.RegisterRoutedEvent(
            "DirectNotification",
            RoutingStrategy.Direct,  // 전파 없음
            typeof(RoutedEventHandler),
            typeof(DirectEventExample));
    
    public void UseDirectEvent()
    {
        var control = new MyControl();
        
        // 직접 이벤트: 발생한 객체에서만 처리
        control.AddHandler(DirectNotificationEvent, 
            new RoutedEventHandler((s, e) =>
            {
                // 상위 컨테이너로 전파되지 않음
                Console.WriteLine("Direct event handled");
            }));
    }
}
```

### 클래스 핸들러: 모든 인스턴스에 공통 동작 추가

```csharp
public class SmartButton : Button
{
    static SmartButton()
    {
        // 모든 SmartButton 인스턴스에 공통 마우스 오버 효과
        EventManager.RegisterClassHandler(
            typeof(SmartButton),
            Mouse.MouseEnterEvent,
            new MouseEventHandler(OnClassMouseEnter));
        
        EventManager.RegisterClassHandler(
            typeof(SmartButton),
            Mouse.MouseLeaveEvent,
            new MouseEventHandler(OnClassMouseLeave));
    }
    
    private static void OnClassMouseEnter(object sender, MouseEventArgs e)
    {
        var button = (SmartButton)sender;
        button.Background = Brushes.LightBlue;
    }
    
    private static void OnClassMouseLeave(object sender, MouseEventArgs e)
    {
        var button = (SmartButton)sender;
        button.Background = SystemColors.ControlBrush;
    }
}
```

## DependencyProperty와 RoutedEvent의 시너지

### 완전한 사용자 정의 컨트롤 예제

```csharp
public class VolumeSlider : Control
{
    static VolumeSlider()
    {
        DefaultStyleKeyProperty.OverrideMetadata(
            typeof(VolumeSlider),
            new FrameworkPropertyMetadata(typeof(VolumeSlider)));
    }
    
    // DependencyProperty 정의
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(
            "Value",
            typeof(double),
            typeof(VolumeSlider),
            new FrameworkPropertyMetadata(
                0.5,
                FrameworkPropertyMetadataOptions.BindsTwoWayByDefault | 
                FrameworkPropertyMetadataOptions.AffectsRender,
                OnValueChanged,
                CoerceValue));
    
    // RoutedEvent 정의
    public static readonly RoutedEvent ValueChangedEvent =
        EventManager.RegisterRoutedEvent(
            "ValueChanged",
            RoutingStrategy.Bubble,
            typeof(RoutedPropertyChangedEventHandler<double>),
            typeof(VolumeSlider));
    
    // CLR 속성 래퍼
    public double Value
    {
        get => (double)GetValue(ValueProperty);
        set => SetValue(ValueProperty, value);
    }
    
    // 이벤트 접근자
    public event RoutedPropertyChangedEventHandler<double> ValueChanged
    {
        add => AddHandler(ValueChangedEvent, value);
        remove => RemoveHandler(ValueChangedEvent, value);
    }
    
    // 값 강제
    private static object CoerceValue(DependencyObject d, object baseValue)
    {
        double value = (double)baseValue;
        return Math.Max(0.0, Math.Min(1.0, value));
    }
    
    // 속성 변경 처리
    private static void OnValueChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        var slider = (VolumeSlider)d;
        
        // 시각적 갱신 유도 (이미 AffectsRender 플래그로 처리됨)
        slider.InvalidateVisual();
        
        // 라우트된 이벤트 발생
        var args = new RoutedPropertyChangedEventArgs<double>(
            (double)e.OldValue,
            (double)e.NewValue,
            ValueChangedEvent);
        slider.RaiseEvent(args);
    }
    
    // 입력 처리
    protected override void OnMouseLeftButtonDown(MouseButtonEventArgs e)
    {
        base.OnMouseLeftButtonDown(e);
        CaptureMouse();
        UpdateValueFromMouse(e.GetPosition(this));
    }
    
    protected override void OnMouseMove(MouseEventArgs e)
    {
        base.OnMouseMove(e);
        if (IsMouseCaptured)
        {
            UpdateValueFromMouse(e.GetPosition(this));
        }
    }
    
    protected override void OnMouseLeftButtonUp(MouseButtonEventArgs e)
    {
        base.OnMouseLeftButtonUp(e);
        ReleaseMouseCapture();
    }
    
    private void UpdateValueFromMouse(Point position)
    {
        // 마우스 위치를 값으로 변환
        double newValue = position.X / ActualWidth;
        Value = newValue; // DependencyProperty 설정
    }
    
    protected override void OnRender(DrawingContext dc)
    {
        base.OnRender(dc);
        
        // 값에 따라 시각적 표현
        double width = ActualWidth * Value;
        Rect fillRect = new Rect(0, 0, width, ActualHeight);
        Rect backgroundRect = new Rect(0, 0, ActualWidth, ActualHeight);
        
        dc.DrawRectangle(Brushes.LightGray, null, backgroundRect);
        dc.DrawRectangle(Brushes.Green, null, fillRect);
    }
}
```

### XAML에서의 사용

```xml
<!-- 테마/Generic.xaml -->
<Style TargetType="{x:Type local:VolumeSlider}">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="{x:Type local:VolumeSlider}">
                <Border Background="{TemplateBinding Background}"
                        BorderBrush="{TemplateBinding BorderBrush}"
                        BorderThickness="{TemplateBinding BorderThickness}">
                    <Grid>
                        <!-- 템플릿 내부 구현 -->
                    </Grid>
                </Border>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>

<!-- 사용 -->
<local:VolumeSlider Value="{Binding VolumeLevel}" 
                    ValueChanged="OnVolumeChanged"
                    Width="200" Height="30"/>
```

## 실제 시나리오에서의 활용 패턴

### 시나리오 1: 데이터 그리드의 셀 편집

```csharp
public class DataGridCellEditor : ContentControl
{
    public static readonly DependencyProperty IsEditingProperty =
        DependencyProperty.RegisterAttached(
            "IsEditing",
            typeof(bool),
            typeof(DataGridCellEditor),
            new FrameworkPropertyMetadata(false, OnIsEditingChanged));
    
    public static readonly RoutedEvent BeginEditEvent =
        EventManager.RegisterRoutedEvent(
            "BeginEdit",
            RoutingStrategy.Bubble,
            typeof(RoutedEventHandler),
            typeof(DataGridCellEditor));
    
    public static void SetIsEditing(UIElement element, bool value)
        => element.SetValue(IsEditingProperty, value);
    
    public static bool GetIsEditing(UIElement element)
        => (bool)element.GetValue(IsEditingProperty);
    
    private static void OnIsEditingChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        var element = (UIElement)d;
        bool isEditing = (bool)e.NewValue;
        
        if (isEditing)
        {
            // 편집 모드 시작
            var args = new RoutedEventArgs(BeginEditEvent, element);
            element.RaiseEvent(args);
        }
        else
        {
            // 편집 모드 종료
            // 데이터 검증 및 저장
        }
    }
}
```

### 시나리오 2: 드래그 앤 드롭 구현

```csharp
public class DragDropManager
{
    public static readonly DependencyProperty IsDragSourceProperty =
        DependencyProperty.RegisterAttached(
            "IsDragSource",
            typeof(bool),
            typeof(DragDropManager),
            new PropertyMetadata(false, OnIsDragSourceChanged));
    
    public static readonly DependencyProperty IsDropTargetProperty =
        DependencyProperty.RegisterAttached(
            "IsDropTarget",
            typeof(bool),
            typeof(DragDropManager),
            new PropertyMetadata(false, OnIsDropTargetChanged));
    
    private static void OnIsDragSourceChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is UIElement element)
        {
            if ((bool)e.NewValue)
            {
                // 드래그 소스로 등록
                element.PreviewMouseLeftButtonDown += OnDragSourceMouseDown;
                element.MouseMove += OnDragSourceMouseMove;
            }
            else
            {
                // 드래그 소스 해제
                element.PreviewMouseLeftButtonDown -= OnDragSourceMouseDown;
                element.MouseMove -= OnDragSourceMouseMove;
            }
        }
    }
    
    private static void OnDragSourceMouseDown(object sender, MouseButtonEventArgs e)
    {
        // 드래그 시작 준비
        _dragStartPoint = e.GetPosition(null);
    }
    
    private static void OnDragSourceMouseMove(object sender, MouseEventArgs e)
    {
        if (e.LeftButton == MouseButtonState.Pressed)
        {
            var currentPoint = e.GetPosition(null);
            if (Math.Abs(currentPoint.X - _dragStartPoint.X) > SystemParameters.MinimumHorizontalDragDistance ||
                Math.Abs(currentPoint.Y - _dragStartPoint.Y) > SystemParameters.MinimumVerticalDragDistance)
            {
                // 드래그 시작
                StartDrag(sender as UIElement);
            }
        }
    }
}
```

## 성능 고려사항과 모범 사례

### 1. Affects 플래그의 올바른 사용

```csharp
public class PerformanceOptimizedControl : FrameworkElement
{
    // 잘못된 예: 모든 변경에 레이아웃 재계산
    public static readonly DependencyProperty BadProperty =
        DependencyProperty.Register("BadProperty", typeof(string), typeof(PerformanceOptimizedControl),
            new FrameworkPropertyMetadata(null, FrameworkPropertyMetadataOptions.AffectsMeasure));
    
    // 올바른 예: 실제로 레이아웃에 영향을 주는 경우에만
    public static readonly DependencyProperty GoodProperty =
        DependencyProperty.Register("GoodProperty", typeof(string), typeof(PerformanceOptimizedControl),
            new FrameworkPropertyMetadata(null));
    
    // 렌더링에만 영향을 주는 경우
    public static readonly DependencyProperty VisualProperty =
        DependencyProperty.Register("VisualProperty", typeof(Brush), typeof(PerformanceOptimizedControl),
            new FrameworkPropertyMetadata(Brushes.Black, FrameworkPropertyMetadataOptions.AffectsRender));
}
```

### 2. Freezable 객체의 효율적 사용

```csharp
public class EfficientResourceUsage
{
    public void UseFreezableObjects()
    {
        // 비효율적: 매번 새로운 브러시 생성
        for (int i = 0; i < 1000; i++)
        {
            var button = new Button();
            button.Background = new SolidColorBrush(Colors.Blue); // 매번 새 인스턴스
        }
        
        // 효율적: 공유 리소스 사용
        var sharedBrush = new SolidColorBrush(Colors.Blue);
        sharedBrush.Freeze(); // 변경 불가능하게 만들어 공유 안전
        
        for (int i = 0; i < 1000; i++)
        {
            var button = new Button();
            button.Background = sharedBrush; // 동일 인스턴스 공유
        }
    }
}
```

### 3. 이벤트 핸들러의 수명 관리

```csharp
public class EventHandlerLifetime : Window
{
    private ListBox _listBox;
    
    public EventHandlerLifetime()
    {
        InitializeComponent();
        
        // 올바른 이벤트 핸들러 등록
        this.Loaded += OnWindowLoaded;
        this.Unloaded += OnWindowUnloaded;
    }
    
    private void OnWindowLoaded(object sender, RoutedEventArgs e)
    {
        _listBox = new ListBox();
        _listBox.AddHandler(Button.ClickEvent, 
            new RoutedEventHandler(OnListItemClick));
        
        // weak event pattern 사용
        CollectionChangedEventManager.AddHandler(
            _dataCollection,
            OnDataCollectionChanged);
    }
    
    private void OnWindowUnloaded(object sender, RoutedEventArgs e)
    {
        // 명시적 이벤트 핸들러 제거
        _listBox.RemoveHandler(Button.ClickEvent, 
            new RoutedEventHandler(OnListItemClick));
        
        CollectionChangedEventManager.RemoveHandler(
            _dataCollection,
            OnDataCollectionChanged);
    }
}
```

## 디버깅과 문제 해결

### DependencyProperty 문제 진단

```csharp
public class DependencyPropertyDebugger
{
    public static void DebugPropertyValues(DependencyObject obj)
    {
        var enumerator = obj.GetLocalValueEnumerator();
        
        Console.WriteLine($"Debugging {obj.GetType().Name}:");
        while (enumerator.MoveNext())
        {
            var entry = enumerator.Current;
            Console.WriteLine($"  {entry.Property.Name}: {entry.Value} " +
                             $"(IsExpression: {DependencyPropertyHelper.GetValueSource(obj, entry.Property).IsExpression})");
        }
    }
    
    public static void TraceBindingErrors()
    {
        // 바인딩 오류 추적 활성화
        PresentationTraceSources.DataBindingSource.Switch.Level = SourceLevels.Warning;
        PresentationTraceSources.DataBindingSource.Listeners.Add(
            new ConsoleTraceListener());
    }
}
```

### RoutedEvent 흐름 추적

```csharp
public class EventTracer
{
    public static void TraceEventFlow(UIElement element, RoutedEvent routedEvent)
    {
        element.AddHandler(routedEvent, new RoutedEventHandler((sender, e) =>
        {
            Console.WriteLine($"{routedEvent.Name} handled by {sender.GetType().Name}");
            Console.WriteLine($"  OriginalSource: {e.OriginalSource.GetType().Name}");
            Console.WriteLine($"  Source: {e.Source.GetType().Name}");
            Console.WriteLine($"  Handled: {e.Handled}");
        }), true); // handledEventsToo: true
    }
}
```

## 결론: WPF의 정수를 이해하기

DependencyProperty와 RoutedEvent는 단순한 기술적 구현체가 아닙니다. 그들은 WPF의 핵심 철학을 구현한 시스템입니다. 이 두 시스템을 통해 WPF는 다음과 같은 목표를 달성합니다:

### 1. 선언적 프로그래밍의 실현
XAML과 같은 선언적 언어가 가능한 것은 DependencyProperty 시스템 덕분입니다. 속성 값의 계산, 상속, 애니메이션, 데이터 바인딩 모두가 런타임에 자동으로 처리됩니다.

### 2. 관심사의 분리
RoutedEvent는 이벤트 처리 로직을 이벤트가 발생한 객체에서 분리시켜 상위 컨테이너에서 일관된 처리를 가능하게 합니다. 이는 코드 재사용성과 유지보수성을 크게 향상시킵니다.

### 3. 성능과 유연성의 균형
DependencyProperty의 효율적인 메모리 관리와 값 캐싱, RoutedEvent의 전파 제어(`Handled` 플래그)는 복잡한 UI에서도 뛰어난 성능을 보장합니다.

### 4. 확장성과 재사용성
Attached Property와 Attached Event는 기존 컨트롤의 동작을 수정하지 않고도 새로운 기능을 추가할 수 있게 합니다. 이는 OCP(Open-Closed Principle)의 훌륭한 구현입니다.

### 마스터하기 위한 실천적 조언

1. **기존 컨트롤을 먼저 이해하세요**: Button, TextBox 같은 표준 컨트롤이 어떻게 DependencyProperty와 RoutedEvent를 사용하는지 분석하세요.

2. **필요할 때만 새로 만드세요**: 간단한 요구사항은 기존 속성과 이벤트로 충분히 해결할 수 있습니다. 진짜 필요할 때만 새로운 DependencyProperty나 RoutedEvent를 만드세요.

3. **프레임워크와 협력하세요**: WPF의 시스템을 거스르지 말고 함께 작동하도록 설계하세요. `Affects` 플래그, `CoerceValue` 콜백, `Handled` 플래그는 모두 시스템과의 협약입니다.

4. **성능을 고려하세요**: 모든 DependencyProperty 변경이 레이아웃을 무효화하지 않도록 주의하세요. `AffectsMeasure`는 정말 필요할 때만 사용하세요.

5. **테스트 가능성을 설계하세요**: DependencyProperty와 RoutedEvent는 테스트하기 어려울 수 있습니다. ViewModel과의 명확한 분리를 유지하고, 가능한 한 비즈니스 로직을 ViewModel에 두세요.

WPF를 진정으로 마스터하는 것은 이 두 시스템을 완전히 이해하고, 그들의 힘을 활용하면서도 그들의 제약 안에서 창의적으로 문제를 해결하는 방법을 배우는 것입니다. DependencyProperty와 RoutedEvent는 단순한 도구가 아니라, WPF의 세계를 보는 새로운 눈입니다. 이 눈을 통해 여러분은 더 우아하고, 더 효율적이며, 더 유지보수하기 쉬운 애플리케이션을 만들 수 있을 것입니다.