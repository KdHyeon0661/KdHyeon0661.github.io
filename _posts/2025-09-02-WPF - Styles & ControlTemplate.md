---
layout: post
title: WPF - Styles & ControlTemplate
date: 2025-09-02 23:25:23 +0900
category: WPF
---
# WPF **Styles & ControlTemplate**

## 큰 그림 요약

- **Style**
  - 특정 타입(예: `Button`) 또는 키가 있는 요소에 **속성 값(Setter)·트리거(Trigger)·이벤트(EventSetter)** 를 일괄 적용.
  - **암시적 스타일(Implicit Style)**: `x:Key`를 생략하고 `TargetType`만 지정하면 **해당 범위의 그 타입 전부**에 적용.
  - **BasedOn**으로 스타일 상속, **ResourceDictionary**로 재사용.
- **ControlTemplate**
  - 컨트롤의 **시각 트리(Visual Tree)** 를 **완전히 교체**. 외형을 바꿔도 **동작(로직)** 은 그대로.
  - 템플릿 내부에서 **`TemplateBinding`** 또는 **`{Binding RelativeSource={RelativeSource TemplatedParent}}`** 로 컨트롤 속성을 반영.
  - **Trigger / VisualStateManager** 로 상태에 따른 시각 효과 제어.
  - **PART_* 이름 관례** 로 템플릿 필수 요소 연결(주로 커스텀 컨트롤에서 사용).

> 기억: **Style = 속성/행동 묶음**, **ControlTemplate = 스킨 교체(시각 트리)**

---

## 스타일의 기본: Setter / TargetType / 암시적 스타일

### 암시적 스타일(Implicit) — 해당 범위의 **모든 버튼**에 적용

```xml
<Window.Resources>
  <!-- x:Key 생략 + TargetType 지정 = 암시적 스타일 -->
  <Style TargetType="Button">
    <Setter Property="FontSize" Value="15"/>
    <Setter Property="Padding" Value="10,6"/>
    <Setter Property="Margin" Value="4"/>
    <Setter Property="Background" Value="#2563EB"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="BorderBrush" Value="#1E40AF"/>
    <Setter Property="BorderThickness" Value="1.5"/>
    <Setter Property="Cursor" Value="Hand"/>
  </Style>
</Window.Resources>

<StackPanel>
  <Button Content="기본 버튼 1"/>
  <Button Content="기본 버튼 2"/>
</StackPanel>
```
- 같은 범위(여기서는 Window) 내부의 **모든 `Button`** 이 스타일을 받습니다.
- **우선순위**: *로컬 값(개별 컨트롤에 직접 지정한 값)*이 **Style Setter**보다 우선합니다.

### 명시적 스타일(Explicit)

```xml
<Window.Resources>
  <Style x:Key="AccentButton" TargetType="Button">
    <Setter Property="Background" Value="#10B981"/>
    <Setter Property="BorderBrush" Value="#059669"/>
  </Style>
</Window.Resources>

<Button Content="OK" Style="{StaticResource AccentButton}"/>
```

### BasedOn으로 스타일 상속

```xml
<Window.Resources>
  <Style x:Key="BaseButton" TargetType="Button">
    <Setter Property="FontSize" Value="14"/>
    <Setter Property="Padding"  Value="10,6"/>
  </Style>

  <Style x:Key="DangerButton" TargetType="Button" BasedOn="{StaticResource BaseButton}">
    <Setter Property="Background" Value="#EF4444"/>
    <Setter Property="Foreground" Value="White"/>
  </Style>
</Window.Resources>
```

> **주의**
> - **Setter는 반드시 DependencyProperty**여야 함(일반 CLR 속성은 Setter로 못 씀).
> - **BasedOn 순환** 금지(런타임 예외).
> - Derived 타입에도 암시적 스타일이 적용되지만, **더 구체적인 TargetType**이 우선.

---

## Style 트리거: Trigger / DataTrigger / MultiTrigger

### Trigger — **의존 속성** 값 변화에 반응

```xml
<Style TargetType="Button">
  <Setter Property="Background" Value="#2563EB"/>
  <Style.Triggers>
    <Trigger Property="IsMouseOver" Value="True">
      <Setter Property="Background" Value="#1D4ED8"/>
    </Trigger>
    <Trigger Property="IsEnabled" Value="False">
      <Setter Property="Opacity" Value="0.6"/>
    </Trigger>
  </Style.Triggers>
</Style>
```

### DataTrigger — **바인딩 값**에 반응(MVVM 친화)

```xml
<Style TargetType="Button" x:Key="SubmitButtonStyle">
  <Setter Property="IsEnabled" Value="False"/>
  <Style.Triggers>
    <DataTrigger Binding="{Binding CanSubmit}" Value="True">
      <Setter Property="IsEnabled" Value="True"/>
    </DataTrigger>
  </Style.Triggers>
</Style>
```

### MultiTrigger / MultiDataTrigger — **복합 조건**

```xml
<Style TargetType="TextBox">
  <Style.Triggers>
    <MultiTrigger>
      <MultiTrigger.Conditions>
        <Condition Property="IsKeyboardFocused" Value="True"/>
        <Condition Property="IsEnabled"         Value="True"/>
      </MultiTrigger.Conditions>
      <Setter Property="BorderBrush" Value="#22C55E"/>
      <Setter Property="BorderThickness" Value="2"/>
    </MultiTrigger>
  </Style.Triggers>
</Style>
```

> **우선순위 팁**
> - **로컬 값(Local value)** > **(템플릿/스타일) 트리거** > **스타일 Setter**.
> - 즉, 어떤 속성에 **로컬 값을 직접 지정**하면 **트리거 Setter가 덮어쓰지 못합니다**. (필요하면 로컬 값 사용을 피하거나, 템플릿 내 **Visual 상태**로 제어)

---

## EventSetter – XAML에서 이벤트 핸들러 연결

```xml
<Style TargetType="Button" x:Key="LogButtonStyle">
  <EventSetter Event="Click" Handler="OnLogButtonClicked"/>
</Style>
```
```csharp
private void OnLogButtonClicked(object sender, RoutedEventArgs e)
{
    Debug.WriteLine($"Clicked: {((Button)sender).Content}");
}
```
- **주의**: `EventSetter`는 **Code-behind 핸들러**가 필요합니다.
  MVVM에선 **Behaviors / AttachedProperty / ICommand(Interaction)** 등으로 대체 권장.

---

## 리소스/테마/머지 — 유지보수 구조화

### App 레벨 테마 분리

```xml
<!-- App.xaml -->
<Application.Resources>
  <ResourceDictionary>
    <ResourceDictionary.MergedDictionaries>
      <ResourceDictionary Source="Themes/Colors.xaml"/>
      <ResourceDictionary Source="Themes/Buttons.xaml"/>
    </ResourceDictionary.MergedDictionaries>
  </ResourceDictionary>
</Application.Resources>
```

### 라이트/다크 테마 교체 (DynamicResource)

```xml
<!-- Colors.Light.xaml -->
<SolidColorBrush x:Key="Brush.Primary" Color="#2563EB"/>
<SolidColorBrush x:Key="Brush.PrimaryText" Color="White"/>

<!-- Colors.Dark.xaml -->
<SolidColorBrush x:Key="Brush.Primary" Color="#1D4ED8"/>
<SolidColorBrush x:Key="Brush.PrimaryText" Color="#E5E7EB"/>

<!-- Buttons.xaml -->
<Style TargetType="Button">
  <Setter Property="Background" Value="{DynamicResource Brush.Primary}"/>
  <Setter Property="Foreground" Value="{DynamicResource Brush.PrimaryText}"/>
</Style>
```
- 런타임에 **MergedDictionaries 교체**하면 **DynamicResource**가 즉시 반영됩니다.

---

## ControlTemplate — 컨트롤 **외형 전체 교체**

### Button 스킨을 완전히 바꾸기

```xml
<Style TargetType="Button" x:Key="PillButton">
  <Setter Property="Background" Value="#2563EB"/>
  <Setter Property="Foreground" Value="White"/>
  <Setter Property="Padding" Value="12,8"/>
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="Button">
        <Border x:Name="Root"
                Background="{TemplateBinding Background}"
                BorderBrush="{TemplateBinding BorderBrush}"
                BorderThickness="{TemplateBinding BorderThickness}"
                CornerRadius="18"
                SnapsToDevicePixels="True">
          <ContentPresenter
              Padding="{TemplateBinding Padding}"
              HorizontalAlignment="{TemplateBinding HorizontalContentAlignment}"
              VerticalAlignment="{TemplateBinding VerticalContentAlignment}"
              RecognizesAccessKey="True"/>
        </Border>
        <ControlTemplate.Triggers>
          <Trigger Property="IsMouseOver" Value="True">
            <Setter TargetName="Root" Property="Background" Value="#1D4ED8"/>
          </Trigger>
          <Trigger Property="IsPressed" Value="True">
            <Setter TargetName="Root" Property="Background" Value="#1E40AF"/>
          </Trigger>
          <Trigger Property="IsEnabled" Value="False">
            <Setter TargetName="Root" Property="Opacity" Value="0.55"/>
          </Trigger>
        </ControlTemplate.Triggers>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```
- **`TemplateBinding`**: **경량 OneWay 바인딩** (성능상 유리).
- **`ContentPresenter`**: 버튼 내용(텍스트/아이콘 등)을 출력.
- **템플릿 트리거**: 템플릿 내부 요소(`TargetName`) 속성 변경.

### TemplateBinding vs TemplatedParent Binding

```xml
<!-- 완전 동일한 의미지만, TemplateBinding이 훨씬 가볍고 빠름 -->
<TextBlock Text="{TemplateBinding FontSize}"/>

<!-- 고급 시나리오(Converter/OneWayToSource 등)가 필요할 때만 -->
<TextBlock Text="{Binding FontSize, RelativeSource={RelativeSource TemplatedParent}}"/>
```

---

## VisualStateManager(VSM) — 상태 기반 템플릿

WPF 4.0+에서 **VSM**를 이용해 상태 전환을 관리할 수 있습니다.
대표 그룹: **CommonStates**(Normal/MouseOver/Pressed/Disabled), **FocusStates**(Focused/Unfocused)

```xml
<Style TargetType="Button" x:Key="VsmButton">
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="Button">
        <Grid x:Name="Root">
          <Border x:Name="Chrome"
                  Background="{TemplateBinding Background}"
                  CornerRadius="8"/>
          <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
        </Grid>
        <VisualStateManager.VisualStateGroups>
          <VisualStateGroup x:Name="CommonStates">
            <VisualState x:Name="Normal"/>
            <VisualState x:Name="MouseOver">
              <Storyboard>
                <ColorAnimation Storyboard.TargetName="Chrome"
                                Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                To="#1D4ED8" Duration="0:0:0.08"/>
              </Storyboard>
            </VisualState>
            <VisualState x:Name="Pressed">
              <Storyboard>
                <ColorAnimation Storyboard.TargetName="Chrome"
                                Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                To="#1E3A8A" Duration="0:0:0.05"/>
                <DoubleAnimation Storyboard.TargetName="Root"
                                 Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleY)"
                                 To="0.98" Duration="0:0:0.05"/>
              </Storyboard>
            </VisualState>
            <VisualState x:Name="Disabled">
              <Storyboard>
                <DoubleAnimation Storyboard.TargetName="Root"
                                 Storyboard.TargetProperty="Opacity"
                                 To="0.55" Duration="0:0:0.0"/>
              </Storyboard>
            </VisualState>
          </VisualStateGroup>
        </VisualStateManager.VisualStateGroups>
        <ControlTemplate.Triggers>
          <!-- 초기 Transform -->
          <Trigger Property="Template" Value="{x:Null}">
            <Setter TargetName="Root" Property="RenderTransform">
              <Setter.Value>
                <ScaleTransform ScaleX="1" ScaleY="1"/>
              </Setter.Value>
            </Setter>
          </Trigger>
        </ControlTemplate.Triggers>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```
- VSM은 **상태 전환 애니메이션**을 다루기 깔끔합니다(트리거 남발 대신).

---

## ControlTemplate의 핵심 요소들

### ContentPresenter / ItemsPresenter

- **ContentPresenter**: `ContentControl`의 콘텐츠 표시(예: `Button`, `Label`).
- **ItemsPresenter**: `ItemsControl`의 아이템 표시(예: `ListBox` 템플릿 안에서 **반드시** `ScrollViewer` 안쪽에 둡니다).

```xml
<ControlTemplate TargetType="ListBox">
  <Border BorderBrush="{TemplateBinding BorderBrush}"
          BorderThickness="{TemplateBinding BorderThickness}">
    <ScrollViewer Focusable="False"
                  Padding="{TemplateBinding Padding}">
      <ItemsPresenter/>
    </ScrollViewer>
  </Border>
</ControlTemplate>
```

### FocusVisualStyle

```xml
<Style TargetType="Button">
  <Setter Property="FocusVisualStyle">
    <Setter.Value>
      <Style TargetType="Control">
        <Setter Property="Template">
          <Setter.Value>
            <ControlTemplate TargetType="Control">
              <Rectangle StrokeDashArray="1 2" StrokeThickness="1" Stroke="#93C5FD"/>
            </ControlTemplate>
          </Setter.Value>
        </Setter>
      </Style>
    </Setter.Value>
  </Setter>
</Style>
```

### OverridesDefaultStyle

```xml
<Style TargetType="CheckBox">
  <Setter Property="OverridesDefaultStyle" Value="True"/>
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="CheckBox">
        <!-- 완전 커스텀 렌더링 -->
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```
- 기본 스타일을 완전히 무시하고 **제로부터** 씁니다. (보통 커스텀 토글 스위치 등)

---

## **UserControl** vs **CustomControl** (템플릿 관점)

- **UserControl**
  - 내부 XAML이 고정. 스킨을 **템플릿으로 교체**하긴 어려움(재정의 지점 제한).
- **CustomControl**
  - **`Themes/Generic.xaml`** 에 **기본 스타일 + ControlTemplate** 제공.
  - 외부에서 언제든 스타일/템플릿을 **교체** 가능 → “컨트롤 라이브러리” 제작에 적합.

### Generic.xaml에 기본 템플릿 제공

```
MyControls/
  Themes/
    Generic.xaml
```

```xml
<!-- Themes/Generic.xaml -->
<ResourceDictionary
  xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
  xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
  xmlns:local="clr-namespace:MyControls">

  <Style TargetType="{x:Type local:TagChip}">
    <Setter Property="Template">
      <Setter.Value>
        <ControlTemplate TargetType="{x:Type local:TagChip}">
          <Border x:Name="Root" Background="{TemplateBinding Background}" CornerRadius="12" Padding="8,4">
            <StackPanel Orientation="Horizontal" Spacing="6">
              <TextBlock Text="{TemplateBinding Text}"/>
              <Button x:Name="PART_Close" Content="×" Padding="4,0" />
            </StackPanel>
          </Border>
          <ControlTemplate.Triggers>
            <Trigger Property="IsMouseOver" Value="True">
              <Setter TargetName="Root" Property="Background" Value="#E5E7EB"/>
            </Trigger>
          </ControlTemplate.Triggers>
        </ControlTemplate>
      </Setter.Value>
    </Setter>
  </Style>
</ResourceDictionary>
```

```csharp
public class TagChip : Control
{
    static TagChip()
    {
        DefaultStyleKeyProperty.OverrideMetadata(typeof(TagChip),
            new FrameworkPropertyMetadata(typeof(TagChip)));
    }

    public static readonly DependencyProperty TextProperty =
        DependencyProperty.Register(nameof(Text), typeof(string), typeof(TagChip), new PropertyMetadata("Tag"));

    public string Text
    {
        get => (string)GetValue(TextProperty);
        set => SetValue(TextProperty, value);
    }

    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        if (GetTemplateChild("PART_Close") is Button btn)
            btn.Click += (_, __) => RaiseEvent(new RoutedEventArgs(ClosedEvent, this));
    }

    // 커스텀 라우티드 이벤트 예시
    public static readonly RoutedEvent ClosedEvent =
        EventManager.RegisterRoutedEvent("Closed", RoutingStrategy.Bubble, typeof(RoutedEventHandler), typeof(TagChip));
    public event RoutedEventHandler Closed { add => AddHandler(ClosedEvent, value); remove => RemoveHandler(ClosedEvent, value); }
}
```
- 템플릿에서 사용할 필수 요소는 **`PART_*`** 이름 관례로 노출하고 **`OnApplyTemplate()`** 에서 연결.

---

## 애니메이션/트리거 — Storyboard, EventTrigger, Enter/Exit Actions

### Style 트리거 + Storyboard

```xml
<Style TargetType="Button">
  <Style.Triggers>
    <Trigger Property="IsMouseOver" Value="True">
      <Trigger.EnterActions>
        <BeginStoryboard>
          <Storyboard>
            <DoubleAnimation Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleX)"
                             To="1.05" Duration="0:0:0.08"/>
            <DoubleAnimation Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleY)"
                             To="1.05" Duration="0:0:0.08"/>
          </Storyboard>
        </BeginStoryboard>
      </Trigger.EnterActions>
      <Trigger.ExitActions>
        <BeginStoryboard>
          <Storyboard>
            <DoubleAnimation Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleX)"
                             To="1.0" Duration="0:0:0.08"/>
            <DoubleAnimation Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleY)"
                             To="1.0" Duration="0:0:0.08"/>
          </Storyboard>
        </BeginStoryboard>
      </Trigger.ExitActions>
    </Trigger>
  </Style.Triggers>
  <Setter Property="RenderTransform">
    <Setter.Value><ScaleTransform ScaleX="1" ScaleY="1"/></Setter.Value>
  </Setter>
</Style>
```

### EventTrigger (주의)

```xml
<Button Content="Pulse">
  <Button.Triggers>
    <!-- RoutedEvent를 트리거로 Storyboard 시작 -->
    <EventTrigger RoutedEvent="Button.Click">
      <BeginStoryboard>
        <Storyboard>
          <DoubleAnimation Storyboard.TargetProperty="Opacity" To="0.3" AutoReverse="True" Duration="0:0:0.15"/>
        </Storyboard>
      </BeginStoryboard>
    </EventTrigger>
  </Button.Triggers>
</Button>
```
- **WPF의 `EventTrigger`는 Storyboard 전용**입니다(코드 실행을 직접 호출하는 트리거가 아님).
  동작/명령은 **InputBindings/Behaviors**를 사용하세요.

---

## ItemsControl 재템플릿 — 가상화/스크롤 주의

```xml
<Style TargetType="ListBox" x:Key="ChipList">
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="ListBox">
        <Border BorderBrush="{TemplateBinding BorderBrush}" BorderThickness="{TemplateBinding BorderThickness}">
          <ScrollViewer CanContentScroll="True">
            <ItemsPresenter/>
          </ScrollViewer>
        </Border>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
  <Setter Property="ItemsPanel">
    <Setter.Value>
      <ItemsPanelTemplate>
        <VirtualizingStackPanel/>
      </ItemsPanelTemplate>
    </Setter.Value>
  </Setter>
</Style>
```
- `ItemsPresenter`는 반드시 템플릿 내부에 존재해야 **아이템이 그려집니다**.
- 가상화 유지하려면 **외부 `ScrollViewer` 추가**나 **`WrapPanel`로 교체**를 신중히 하세요(가상화 꺼질 수 있음).

---

## StaticResource vs DynamicResource (스타일/템플릿에서)

- **StaticResource**: 로드 시 **한 번만 해석**. 빠르고 안전.
- **DynamicResource**: 런타임에 리소스 **변경 감지**. 테마 전환/사용자 커스터마이즈에 유용.
- 템플릿/스타일에서 **테마 색** 같이 바뀔 가능성이 있는 것은 **DynamicResource**로,
  그렇지 않으면 **StaticResource**로 성능 최적화.

```xml
<Setter Property="Background" Value="{DynamicResource Brush.Primary}"/>
<Setter Property="BorderBrush" Value="{StaticResource Brush.Border}"/>
```

---

## 성능/안정성 체크리스트

- **TemplateBinding** 우선(경량) → 복잡 바인딩 필요 시 `TemplatedParent` 바인딩.
- **Freezable(Brush/Geometry)** 리소스는 **동적 변경 없으면 Freeze**(WPF 내부가 자동 Freeze).
- **Trigger 남발 주의**: 상태가 많으면 **VSM**로 단순화.
- **로컬 값 설정 지양**: 스타일/트리거가 **덮어쓰기 어려움**. 가능하면 **Style 관리**.
- **BasedOn 체인**은 짧게: 순환/해석 비용/가독성.

---

## 실전: **토글 스위치**(CheckBox → 스위치 UI) 템플릿

```xml
<Style TargetType="CheckBox" x:Key="ToggleSwitch">
  <Setter Property="Width"  Value="56"/>
  <Setter Property="Height" Value="32"/>
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="CheckBox">
        <Grid x:Name="Root" SnapsToDevicePixels="True">
          <Grid.ColumnDefinitions>
            <ColumnDefinition Width="*"/>
          </Grid.ColumnDefinitions>

          <!-- Track -->
          <Border x:Name="Track" CornerRadius="16"
                  Background="#CBD5E1" Height="32" Width="56"/>

          <!-- Thumb -->
          <Border x:Name="Thumb" Width="28" Height="28" CornerRadius="14"
                  Background="White" Margin="2"
                  HorizontalAlignment="Left"/>

        </Grid>

        <ControlTemplate.Triggers>
          <Trigger Property="IsChecked" Value="True">
            <Setter TargetName="Track" Property="Background" Value="#22C55E"/>
            <Setter TargetName="Thumb" Property="HorizontalAlignment" Value="Right"/>
          </Trigger>
          <Trigger Property="IsEnabled" Value="False">
            <Setter TargetName="Track" Property="Opacity" Value="0.6"/>
            <Setter TargetName="Thumb" Property="Opacity" Value="0.8"/>
          </Trigger>
          <!-- MouseOver/Pressed 애니메이션 (간단 버전) -->
          <Trigger Property="IsMouseOver" Value="True">
            <Setter TargetName="Thumb" Property="Effect">
              <Setter.Value>
                <DropShadowEffect Opacity="0.35" BlurRadius="8" ShadowDepth="0"/>
              </Setter.Value>
            </Setter>
          </Trigger>
        </ControlTemplate.Triggers>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>

<!-- 사용 -->
<CheckBox Style="{StaticResource ToggleSwitch}" Content="알림"/>
```
- **기능(체크/언체크)** 은 그대로, **시각만 전환**.
- 고급: Thumb 이동을 **DoubleAnimation**으로 부드럽게 만들 수 있음.

---

## 실전: **TextBox 오류 상태 스타일**(검증 UI)

```xml
<Style TargetType="TextBox" x:Key="ErrorTextBox">
  <Setter Property="BorderBrush" Value="#CBD5E1"/>
  <Setter Property="BorderThickness" Value="1.5"/>
  <Style.Triggers>
    <!-- Validation.HasError는 의존 속성 -->
    <Trigger Property="Validation.HasError" Value="True">
      <Setter Property="BorderBrush" Value="#EF4444"/>
      <Setter Property="ToolTip"
              Value="{Binding RelativeSource={RelativeSource Self},
                              Path=(Validation.Errors)[0].ErrorContent}"/>
    </Trigger>
  </Style.Triggers>
</Style>

<TextBox Style="{StaticResource ErrorTextBox}"
         Text="{Binding Name, UpdateSourceTrigger=PropertyChanged, ValidatesOnDataErrors=True}"/>
```
- MVVM의 `INotifyDataErrorInfo`/`IDataErrorInfo` 기반 검증과 조합.

---

## 고급 팁 모음

### 스타일에서 **AttachedProperty** Setter

```xml
<Style TargetType="ListBox">
  <Setter Property="ScrollViewer.CanContentScroll" Value="True"/>
  <Setter Property="ScrollViewer.HorizontalScrollBarVisibility" Value="Disabled"/>
</Style>
```

### `x:Shared`로 리소스 인스턴스 제어

```xml
<SolidColorBrush x:Key="AccentBrush" Color="#2563EB" x:Shared="False"/>
```
- **`x:Shared="False"`**: 리소스를 **참조할 때마다 새 인스턴스**(애니메이션 대상 Brush가 개별 인스턴스가 필요할 때).

### 파편화 방지 — “기본 타입 스타일 + 역할별 서브 스타일”

```xml
<!-- 기본 Button 스타일(암시적) -->
<Style TargetType="Button">
  <Setter Property="FontWeight" Value="SemiBold"/>
  <Setter Property="Padding" Value="10,6"/>
</Style>
<!-- 역할별 -->
<Style x:Key="PrimaryButton" TargetType="Button" BasedOn="{StaticResource {x:Type Button}}">
  <Setter Property="Background" Value="#2563EB"/>
  <Setter Property="Foreground" Value="White"/>
</Style>
<Style x:Key="DangerButton" TargetType="Button" BasedOn="{StaticResource {x:Type Button}}">
  <Setter Property="Background" Value="#EF4444"/>
  <Setter Property="Foreground" Value="White"/>
</Style>
```

### TemplateBinding으로 **성능 최적화**

- 템플릿 내 **단순 전달**은 **TemplateBinding**을 최우선 사용.
- 복잡 바인딩(Converter, Fallback 등)이 필요할 때만 `TemplatedParent` 바인딩.

### Trigger보다 **VSM 우선**(복잡 상태)

- 상태 개수가 많아질수록 VSM로 그룹화(가독성/유지보수).

---

## “왜 적용이 안 되지?” 트러블슈팅

1. **암시적 스타일 범위**가 다름?
   - 그 컨트롤이 다른 Resource 스코프에 있지 않은지(예: DataTemplate 내부 ItemsControl 등).
2. **로컬 값이 우선**?
   - XAML에 직접 속성 값을 박아두면 스타일 Setter/Trigger가 못 덮음.
3. **키/리소스 해석 시점**
   - `StaticResource` 는 선언 순서 영향. 필요하면 `DynamicResource`.
4. **템플릿 내부 이름/TargetName 오타**
   - `TargetName` 못 찾으면 트리거가 적용 안 됨.
5. **ItemsControl 가상화**
   - 외부 ScrollViewer로 감싸서 **가상화가 꺼지면** 성능/스타일 동작이 달라질 수 있음.
6. **BasedOn 순환**
   - 예외 발생. 스타일 상속 트리를 단순화.

---

## 실습: “카드 카드” 컨트롤 – 스타일 & 템플릿 결합 예

### Card 스타일

```xml
<Style x:Key="Card" TargetType="ContentControl">
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="ContentControl">
        <Border x:Name="Root" Background="#FFFFFF" CornerRadius="10"
                BorderBrush="#E5E7EB" BorderThickness="1" Padding="14"
                SnapsToDevicePixels="True">
          <ContentPresenter/>
        </Border>
        <ControlTemplate.Triggers>
          <Trigger Property="IsMouseOver" Value="True">
            <Setter TargetName="Root" Property="BorderBrush" Value="#93C5FD"/>
            <Setter TargetName="Root" Property="Effect">
              <Setter.Value>
                <DropShadowEffect BlurRadius="8" ShadowDepth="1" Opacity="0.2"/>
              </Setter.Value>
            </Setter>
          </Trigger>
        </ControlTemplate.Triggers>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```

### 카드 내부 버튼 스타일 재사용

```xml
<Style x:Key="CardAction" TargetType="Button" BasedOn="{StaticResource {x:Type Button}}">
  <Setter Property="Padding" Value="8,4"/>
  <Setter Property="Background" Value="#F1F5F9"/>
  <Setter Property="BorderBrush" Value="#CBD5E1"/>
  <Setter Property="Foreground" Value="#0F172A"/>
  <Style.Triggers>
    <Trigger Property="IsMouseOver" Value="True">
      <Setter Property="Background" Value="#E2E8F0"/>
    </Trigger>
  </Style.Triggers>
</Style>
```

### 사용

```xml
<ContentControl Style="{StaticResource Card}">
  <StackPanel>
    <TextBlock Text="Report" FontSize="18" FontWeight="SemiBold"/>
    <TextBlock Text="2025 Q4 Metrics" Opacity="0.7" Margin="0,2,0,10"/>
    <StackPanel Orientation="Horizontal" Spacing="8">
      <Button Content="Open" Style="{StaticResource CardAction}"/>
      <Button Content="Export" Style="{StaticResource CardAction}"/>
    </StackPanel>
  </StackPanel>
</ContentControl>
```

---

## 스타일/템플릿의 테스트 전략

- **시각 스냅샷 테스트**: 동일 렌더링 보장(픽셀 비교).
- **시각 트리 규칙 검사**: `Template.FindName` / UI 자동화 트리로 필수 요소(PART) 존재 검증.
- **상태 전환 테스트**: VSM 상태 호출(`VisualStateManager.GoToState`) 후 속성 값 확인.
- **성능**: 스크롤/리스트 가상화 유지, 템플릿 바인딩 수 최소화.

---

## 마무리 요약

- **Style**
  - 속성/트리거/이벤트를 범위 단위로 묶어 **일관성**과 **재사용성**을 확보.
  - **암시적 스타일**로 기본값을 정의하고, **BasedOn**으로 역할별 커스터마이즈.
  - 데이터/상태 변화는 **(Multi)DataTrigger/Trigger**.
- **ControlTemplate**
  - 컨트롤 시각 트리를 교체; **동작은 그대로**.
  - **TemplateBinding**으로 경량 연결, **VSM**으로 상태 전환을 깔끔히.
  - 커스텀 컨트롤은 **Generic.xaml**에 기본 템플릿 제공 + **PART** 관례.

이제 여기까지의 설계를 **리소스 딕셔너리**로 모듈화하고,
라이트/다크 테마로 교체 가능한 **DynamicResource** 기반 팔레트를 더하면,
**유지보수 쉬운 디자인 시스템**을 완성할 수 있습니다. 🎯

---

## 부록 A) 통합 샘플 — 버튼 디자인 시스템 (라이트/다크 지원)

```xml
<!-- Colors.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
  <!-- 팔레트 -->
  <SolidColorBrush x:Key="Btn.Primary" Color="#2563EB"/>
  <SolidColorBrush x:Key="Btn.Primary.Hover" Color="#1D4ED8"/>
  <SolidColorBrush x:Key="Btn.Primary.Pressed" Color="#1E40AF"/>
  <SolidColorBrush x:Key="Btn.Text" Color="White"/>
  <SolidColorBrush x:Key="Btn.Border" Color="#1E40AF"/>
</ResourceDictionary>
```

```xml
<!-- Buttons.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
  <Style TargetType="Button" x:Key="Btn.Primary">
    <Setter Property="Foreground" Value="{DynamicResource Btn.Text}"/>
    <Setter Property="Background" Value="{DynamicResource Btn.Primary}"/>
    <Setter Property="BorderBrush" Value="{DynamicResource Btn.Border}"/>
    <Setter Property="BorderThickness" Value="1.4"/>
    <Setter Property="Padding" Value="12,7"/>
    <Setter Property="Cursor" Value="Hand"/>
    <Setter Property="Template">
      <Setter.Value>
        <ControlTemplate TargetType="Button">
          <Border x:Name="Chrome"
                  Background="{TemplateBinding Background}"
                  BorderBrush="{TemplateBinding BorderBrush}"
                  BorderThickness="{TemplateBinding BorderThickness}"
                  CornerRadius="10">
            <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"
                              Margin="{TemplateBinding Padding}"/>
          </Border>
          <ControlTemplate.Triggers>
            <Trigger Property="IsMouseOver" Value="True">
              <Setter TargetName="Chrome" Property="Background" Value="{DynamicResource Btn.Primary.Hover}"/>
            </Trigger>
            <Trigger Property="IsPressed" Value="True">
              <Setter TargetName="Chrome" Property="Background" Value="{DynamicResource Btn.Primary.Pressed}"/>
            </Trigger>
            <Trigger Property="IsEnabled" Value="False">
              <Setter TargetName="Chrome" Property="Opacity" Value="0.6"/>
            </Trigger>
          </ControlTemplate.Triggers>
        </ControlTemplate>
      </Setter.Value>
    </Setter>
  </Style>
</ResourceDictionary>
```

```xml
<!-- App.xaml -->
<Application ...>
  <Application.Resources>
    <ResourceDictionary>
      <ResourceDictionary.MergedDictionaries>
        <ResourceDictionary Source="Themes/Colors.xaml"/>
        <ResourceDictionary Source="Themes/Buttons.xaml"/>
      </ResourceDictionary.MergedDictionaries>
    </ResourceDictionary>
  </Application.Resources>
</Application>
```

```xml
<!-- 사용 -->
<Button Content="확인" Style="{StaticResource Btn.Primary}" />
<Button Content="삭제" Style="{StaticResource Btn.Primary}" />
```

**다크 테마 전환**은 `Colors.xaml`을 `Colors.Dark.xaml`로 교체(동일 키 유지)만 하면 끝.
**DynamicResource** 덕분에 **재시작 없이 즉시 반영**됩니다.

---

## 부록 B) 코드로 테마 전환(런타임 교체 예)

```csharp
void ApplyTheme(Uri themeUri)
{
    var appRes = Application.Current.Resources;
    // 첫 번째 머지 딕셔너리가 팔레트라고 가정
    var merged = appRes.MergedDictionaries;
    var idx = merged.ToList().FindIndex(d => d.Source != null && d.Source.OriginalString.Contains("Colors"));
    if (idx >= 0) merged[idx] = new ResourceDictionary { Source = themeUri };
    else merged.Insert(0, new ResourceDictionary { Source = themeUri });
}

// 라이트 → 다크
ApplyTheme(new Uri("Themes/Colors.Dark.xaml", UriKind.Relative));
```

---

## 부록 C) 스타일/템플릿 체크리스트 (현업용)

- [ ] **암시적 기본 스타일** 정의(타입별 디자인 토대)
- [ ] 역할/상태 별 **서브 스타일**(Primary/Danger/Outline/Link…)
- [ ] **ControlTemplate로 스킨 통일**, 상태는 **VSM**로
- [ ] **DynamicResource 팔레트**(라이트/다크/브랜드 색)
- [ ] ItemsControl 템플릿은 **ItemsPresenter + 가상화 유지**
- [ ] **로컬 값 최소화** / 스타일로 통제
- [ ] **PART 요소**/OnApplyTemplate 계약(커스텀 컨트롤)
- [ ] 성능: **TemplateBinding 우선**, 트리거/바인딩 최소화
