---
layout: post
title: WPF - Visual State Manager
date: 2025-09-07 19:25:23 +0900
category: WPF
---
# 🎛️ WPF **Visual State Manager(VSM)** 완전 정복
*(개념 → 문법 → 컨트롤 템플릿 → 코드 제어 → MVVM 패턴 → 사용자 지정 VSM → 성능/트러블슈팅까지, 예제 중심으로 “빠짐없이” 정리)*

> VSM은 컨트롤의 **상태(State)** 를 선언적으로 정의하고, **상태 전환(Transition)** 을 애니메이션으로 기술해
> **복잡한 Trigger 난립 없이** 일관된 UI를 만들게 해 줍니다.  
> WPF 4.0+에서 본격 지원되며, `ControlTemplate` 안에서 가장 빛을 발합니다.

---

## 0. 한눈에 보는 핵심

- **VisualState** = “상태 이름 + Storyboard(선택)”.  
- **VisualStateGroup** = 관련 상태 묶음(예: `CommonStates`: `Normal` / `MouseOver` / `Pressed` / `Disabled`).  
- **한 그룹 내에 동시에 하나의 상태만 활성**. 그룹이 여러 개면 각 그룹에서 하나씩 활성.  
- **전환**: `VisualTransition` 로 상태 간 애니메이션/시간/완화(Easing) 지정.  
- **코드 전환**: `VisualStateManager.GoToState(control, "StateName", useTransitions)`  
- **MVVM**: Behavior/AttachedProperty로 상태를 바인딩하거나, ViewModel 이벤트→상태 전환 연결.  
- **CustomVisualStateManager**: 전환 로직을 커스터마이즈 가능(고급).

---

## 1. 기본 문법: 상태/그룹/전환

### 1.1 Button 템플릿에서 `CommonStates` 정의
```xml
<Style TargetType="Button" x:Key="VsmButton">
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="Button">
        <Grid x:Name="Root" RenderTransformOrigin="0.5,0.5">
          <Border x:Name="Chrome"
                  Background="{TemplateBinding Background}"
                  BorderBrush="{TemplateBinding BorderBrush}"
                  BorderThickness="1.5"
                  CornerRadius="10"/>
          <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"
                            Margin="{TemplateBinding Padding}"/>
          <Grid.RenderTransform>
            <ScaleTransform ScaleX="1" ScaleY="1"/>
          </Grid.RenderTransform>
        </Grid>

        <!-- VSM 구간 -->
        <VisualStateManager.VisualStateGroups>
          <VisualStateGroup x:Name="CommonStates">
            <VisualState x:Name="Normal"/>
            <VisualState x:Name="MouseOver">
              <Storyboard>
                <DoubleAnimation Storyboard.TargetName="Chrome"
                                 Storyboard.TargetProperty="Opacity"
                                 To="0.95" Duration="0:0:0.08"/>
              </Storyboard>
            </VisualState>
            <VisualState x:Name="Pressed">
              <Storyboard>
                <DoubleAnimation Storyboard.TargetName="Root"
                                 Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleY)"
                                 To="0.98" Duration="0:0:0.06"/>
              </Storyboard>
            </VisualState>
            <VisualState x:Name="Disabled">
              <Storyboard>
                <DoubleAnimation Storyboard.TargetName="Root"
                                 Storyboard.TargetProperty="Opacity"
                                 To="0.55" Duration="0:0:0"/>
              </Storyboard>
            </VisualState>
          </VisualStateGroup>
        </VisualStateManager.VisualStateGroups>

        <!-- (선택) 보조 Trigger: 초기 Transform 세팅 등 -->
        <ControlTemplate.Triggers>
          <Trigger Property="IsDefaulted" Value="True">
            <Setter TargetName="Chrome" Property="BorderBrush" Value="#60A5FA"/>
          </Trigger>
        </ControlTemplate.Triggers>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```

> **포인트**  
> - VSM은 템플릿 내부의 **이름 있는 요소(TargetName)** 에 애니메이션을 적용합니다.  
> - 상태 이름은 관례적으로 `CommonStates`, `FocusStates`, `ValidationStates`, `SelectionStates` 등 그룹으로 나뉩니다.

---

## 2. 상태 전환(VisualTransition) 세밀 제어

### 2.1 `VisualTransition`로 상태 간 공통 전환 지정
```xml
<VisualStateManager.VisualStateGroups>
  <VisualStateGroup x:Name="CommonStates">
    <VisualStateGroup.Transitions>
      <!-- 모든 상태 전환에 기본 120ms Ease 적용 -->
      <VisualTransition GeneratedDuration="0:0:0.12">
        <VisualTransition.EasingFunction>
          <SineEase EasingMode="EaseOut"/>
        </VisualTransition.EasingFunction>
      </VisualTransition>

      <!-- Pressed -> Normal로 돌아갈 때만 더 빠르게 -->
      <VisualTransition From="Pressed" To="Normal" GeneratedDuration="0:0:0.06"/>
    </VisualStateGroup.Transitions>

    <VisualState x:Name="Normal"/>
    <VisualState x:Name="MouseOver">
      <Storyboard>
        <DoubleAnimation Storyboard.TargetName="Chrome" Storyboard.TargetProperty="Opacity"
                         To="0.96" Duration="0:0:0"/>
      </Storyboard>
    </VisualState>
    <VisualState x:Name="Pressed">
      <Storyboard>
        <DoubleAnimation Storyboard.TargetName="Root"
                         Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleY)"
                         To="0.98" Duration="0:0:0"/>
      </Storyboard>
    </VisualState>
    <VisualState x:Name="Disabled">
      <Storyboard>
        <DoubleAnimation Storyboard.TargetName="Root" Storyboard.TargetProperty="Opacity"
                         To="0.55" Duration="0:0:0"/>
      </Storyboard>
    </VisualState>
  </VisualStateGroup>
</VisualStateManager.VisualStateGroups>
```

> **GeneratedDuration**: 상태 Storyboard에 지정된 Duration이 없어도 **그룹 전환 기본 시간**으로 부드럽게 전환.  
> **From/To** 로 특정 쌍 전환만 별도 시간/이징을 지정.

---

## 3. 코드에서 상태 전환: `GoToState`

### 3.1 `OnApplyTemplate`에서 초기 상태 진입
```csharp
public class PillButton : Button
{
    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        // 템플릿이 적용된 후에만 상태 전환 가능
        VisualStateManager.GoToState(this, IsEnabled ? "Normal" : "Disabled", false);
    }

    protected override void OnIsEnabledChanged(DependencyPropertyChangedEventArgs e)
    {
        base.OnIsEnabledChanged(e);
        VisualStateManager.GoToState(this, IsEnabled ? "Normal" : "Disabled", true);
    }
}
```

### 3.2 외부 이벤트로 전환
```csharp
// 예: 로딩 완료 시 "LoadedState"로 전환
private async void LoadDataAsync()
{
    VisualStateManager.GoToState(this, "Loading", true);
    await Task.Delay(600);
    VisualStateManager.GoToState(this, "LoadedState", true);
}
```

> **주의**: `GoToState` 는 **템플릿 루트가 해당 상태를 정의**한 경우에만 성공합니다. 상태 이름 오타/그룹 누락을 조심하세요.

---

## 4. MVVM 친화: XAML에서 상태를 바인딩처럼 쓰는 법

WPF 기본만으로는 **상태 이름 직접 바인딩**은 지원하지 않습니다. 흔한 해법은 두 가지:

### 4.1 **Behaviors** (Microsoft.Xaml.Behaviors.Wpf) 사용
```xml
<!-- 패키지: Microsoft.Xaml.Behaviors.Wpf -->
<Window xmlns:i="http://schemas.microsoft.com/xaml/behaviors"
        xmlns:ei="http://schemas.microsoft.com/expression/2010/interactions">
  <Grid x:Name="Root">
    <i:Interaction.Triggers>
      <!-- ViewModel에서 bool 속성이 True가 되면 상태 전환 -->
      <ei:DataTrigger Binding="{Binding IsBusy}" Value="True">
        <ei:GoToStateAction StateName="Busy" UseTransitions="True"/>
      </ei:DataTrigger>
      <ei:DataTrigger Binding="{Binding IsBusy}" Value="False">
        <ei:GoToStateAction StateName="Idle" UseTransitions="True"/>
      </ei:DataTrigger>
    </i:Interaction.Triggers>
  </Grid>
</Window>
```

### 4.2 **AttachedProperty** 로 래핑
```csharp
public static class VisualStateHelper
{
    public static readonly DependencyProperty StateProperty =
        DependencyProperty.RegisterAttached("State", typeof(string), typeof(VisualStateHelper),
            new PropertyMetadata(null, OnStateChanged));

    public static string GetState(DependencyObject obj) => (string)obj.GetValue(StateProperty);
    public static void SetState(DependencyObject obj, string value) => obj.SetValue(StateProperty, value);

    private static void OnStateChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is FrameworkElement fe && e.NewValue is string s && fe.IsLoaded)
            VisualStateManager.GoToState(fe, s, true);
        else if (d is FrameworkElement fe2)
            fe2.Loaded += (s2, _) => VisualStateManager.GoToState(fe2, (string)e.NewValue, false);
    }
}
```

```xml
<Grid local:VisualStateHelper.State="{Binding CurrentState}">
  <VisualStateManager.VisualStateGroups>
    <VisualStateGroup x:Name="LoadingStates">
      <VisualState x:Name="Idle"/>
      <VisualState x:Name="Busy">
        <Storyboard>
          <DoubleAnimation Storyboard.TargetProperty="Opacity" To="0.5" Duration="0:0:0.1"/>
        </Storyboard>
      </VisualState>
    </VisualStateGroup>
  </VisualStateManager.VisualStateGroups>
</Grid>
```

> 이렇게 하면 ViewModel의 문자열 속성(`CurrentState`)을 바꿔 **상태 전환을 MVVM스럽게** 제어할 수 있습니다.

---

## 5. 템플릿 정석: **Generic.xaml** + 커스텀 컨트롤

### 5.1 커스텀 토글 스위치 예제 (상태: `On`/`Off`/`Disabled`)
**Controls/ToggleSwitch.cs**
```csharp
public class ToggleSwitch : Control
{
    static ToggleSwitch()
    {
        DefaultStyleKeyProperty.OverrideMetadata(typeof(ToggleSwitch),
          new FrameworkPropertyMetadata(typeof(ToggleSwitch)));
    }

    public bool IsOn
    {
        get => (bool)GetValue(IsOnProperty);
        set => SetValue(IsOnProperty, value);
    }
    public static readonly DependencyProperty IsOnProperty =
        DependencyProperty.Register(nameof(IsOn), typeof(bool), typeof(ToggleSwitch),
            new FrameworkPropertyMetadata(false, OnIsOnChanged));

    private static void OnIsOnChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        var t = (ToggleSwitch)d;
        t.UpdateState(true);
    }

    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        UpdateState(false);
        if (GetTemplateChild("PART_Track") is UIElement track)
            track.MouseLeftButtonUp += (_, __) => IsOn = !IsOn;
    }

    private void UpdateState(bool useTransitions)
    {
        var state = IsEnabled ? (IsOn ? "On" : "Off") : "Disabled";
        VisualStateManager.GoToState(this, state, useTransitions);
    }
}
```

**Themes/Generic.xaml**
```xml
<ResourceDictionary
  xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
  xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
  xmlns:local="clr-namespace:MyControls">

  <Style TargetType="{x:Type local:ToggleSwitch}">
    <Setter Property="Width" Value="56"/>
    <Setter Property="Height" Value="32"/>
    <Setter Property="Template">
      <Setter.Value>
        <ControlTemplate TargetType="{x:Type local:ToggleSwitch}">
          <Grid x:Name="Root">
            <Border x:Name="PART_Track" Background="#CBD5E1" CornerRadius="16"/>
            <Border x:Name="Thumb" Width="28" Height="28" Background="White" CornerRadius="14" Margin="2"
                    HorizontalAlignment="Left"/>
          </Grid>

          <VisualStateManager.VisualStateGroups>
            <VisualStateGroup x:Name="ToggleStates">
              <VisualStateGroup.Transitions>
                <VisualTransition GeneratedDuration="0:0:0.12">
                  <VisualTransition.EasingFunction>
                    <CubicEase EasingMode="EaseInOut"/>
                  </VisualTransition.EasingFunction>
                </VisualTransition>
              </VisualStateGroup.Transitions>

              <VisualState x:Name="Off">
                <Storyboard>
                  <DoubleAnimation Storyboard.TargetName="Thumb"
                                   Storyboard.TargetProperty="(FrameworkElement.Margin).(Thickness.Left)"
                                   To="2" Duration="0:0:0"/>
                  <ColorAnimation Storyboard.TargetName="PART_Track"
                                  Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                  To="#CBD5E1" Duration="0:0:0"/>
                </Storyboard>
              </VisualState>

              <VisualState x:Name="On">
                <Storyboard>
                  <DoubleAnimation Storyboard.TargetName="Thumb"
                                   Storyboard.TargetProperty="(FrameworkElement.Margin).(Thickness.Left)"
                                   To="26" Duration="0:0:0"/>
                  <ColorAnimation Storyboard.TargetName="PART_Track"
                                  Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                  To="#22C55E" Duration="0:0:0"/>
                </Storyboard>
              </VisualState>

              <VisualState x:Name="Disabled">
                <Storyboard>
                  <DoubleAnimation Storyboard.TargetName="Root" Storyboard.TargetProperty="Opacity"
                                   To="0.55" Duration="0:0:0"/>
                </Storyboard>
              </VisualState>
            </VisualStateGroup>
          </VisualStateManager.VisualStateGroups>
        </ControlTemplate>
      </Setter.Value>
    </Setter>
  </Style>
</ResourceDictionary>
```

> `UpdateState` 에서 **현재 속성(IsOn/IsEnabled)** 기반으로 상태 이름을 만들고 `GoToState` 호출 → **상태-기반 템플릿** 완성.

---

## 6. 폼/검증/선택 등 **여러 그룹** 동시 사용

한 컨트롤 템플릿에 **여러 그룹**을 넣어 각각 독립 상태를 운용할 수 있습니다.

```xml
<VisualStateManager.VisualStateGroups>
  <!-- 상호작용 공통 상태 -->
  <VisualStateGroup x:Name="CommonStates">
    <VisualState x:Name="Normal"/>
    <VisualState x:Name="MouseOver"/>
    <VisualState x:Name="Pressed"/>
    <VisualState x:Name="Disabled"/>
  </VisualStateGroup>

  <!-- 포커스 상태 -->
  <VisualStateGroup x:Name="FocusStates">
    <VisualState x:Name="Unfocused"/>
    <VisualState x:Name="Focused">
      <Storyboard>
        <ColorAnimation Storyboard.TargetName="Chrome"
                        Storyboard.TargetProperty="(Border.BorderBrush).(SolidColorBrush.Color)"
                        To="#60A5FA" Duration="0:0:0"/>
      </Storyboard>
    </VisualState>
  </VisualStateGroup>

  <!-- 검증 상태 -->
  <VisualStateGroup x:Name="ValidationStates">
    <VisualState x:Name="Valid"/>
    <VisualState x:Name="Invalid">
      <Storyboard>
        <ColorAnimation Storyboard.TargetName="Chrome"
                        Storyboard.TargetProperty="(Border.BorderBrush).(SolidColorBrush.Color)"
                        To="#EF4444" Duration="0:0:0"/>
      </Storyboard>
    </VisualState>
  </VisualStateGroup>
</VisualStateManager.VisualStateGroups>
```

> 마우스/포커스/검증 상태가 **동시에** 달라질 수 있으므로, 그룹을 나눠 **독립적으로** 상태를 유지하세요.

---

## 7. VSM vs Triggers: 언제 무엇을?

| 항목 | **VSM** | **(Style/Template) Trigger** |
|---|---|---|
| 선언 위치 | 주로 `ControlTemplate` 안 `VisualStateGroups` | Style/Template 내 Triggers 섹션 |
| 목적 | **상태의 집합**과 **전환**을 구조화 | **단일 조건**에 따른 Setter/Storyboard 실행 |
| 애니메이션 | `VisualTransition`/`Storyboard` 내장 | `BeginStoryboard`/`Enter/ExitActions` |
| 복잡도 | 상태가 많아질수록 **가독성 ↑** | 상태 폭증 시 **난립/충돌** 우려 |
| MVVM | `GoToState`/Behavior/AttachedProperty로 용이 | DataTrigger 등 간단 바인딩은 쉬움 |

> 컨트롤 수준(버튼/입력 등)에서는 **VSM 권장**. 단순한 속성 반응만 필요하면 Trigger도 OK.

---

## 8. 고급: **CustomVisualStateManager** 로 전환 로직 커스터마이즈

특정 상태 전환을 **특수 규칙**으로 처리하고 싶을 때 VSM을 상속해 사용합니다.

```csharp
public class InstantPressStateManager : VisualStateManager
{
    protected override bool GoToStateCore(FrameworkElement control, FrameworkElement templateRoot,
        string stateName, VisualStateGroup group, VisualState state, bool useTransitions)
    {
        // Pressed로 갈 때는 전환 무시(즉시 반응), 그 외는 기본 처리
        if (group.Name == "CommonStates" && stateName == "Pressed")
            return base.GoToStateCore(control, templateRoot, stateName, group, state, false);

        return base.GoToStateCore(control, templateRoot, stateName, group, state, useTransitions);
    }
}
```

템플릿에서 **CustomVisualStateManager** 지정:
```xml
<ControlTemplate TargetType="Button">
  <VisualStateManager.CustomVisualStateManager>
    <local:InstantPressStateManager/>
  </VisualStateManager.CustomVisualStateManager>

  <!-- VisualStateGroups ... -->
</ControlTemplate>
```

> 드물지만 “특정 전환만 시간 없이”, “다른 그룹 전환과 동기화” 같은 특수 규칙이 필요한 경우 유용합니다.

---

## 9. 페이지 전환/레이아웃 상태에도 VSM 적용

> 단일 컨트롤이 아니라 **페이지/뷰 자체**에 상태를 정의하고 전환에 활용할 수 있습니다.

```xml
<Grid x:Name="Root">
  <VisualStateManager.VisualStateGroups>
    <VisualStateGroup x:Name="PageStates">
      <VisualState x:Name="List">
        <Storyboard>
          <DoubleAnimation Storyboard.TargetName="DetailPanel" Storyboard.TargetProperty="Opacity"
                           To="0" Duration="0:0:0.15"/>
        </Storyboard>
      </VisualState>
      <VisualState x:Name="Detail">
        <Storyboard>
          <DoubleAnimation Storyboard.TargetName="DetailPanel" Storyboard.TargetProperty="Opacity"
                           To="1" Duration="0:0:0.18"/>
          <DoubleAnimation Storyboard.TargetName="DetailPanel"
                           Storyboard.TargetProperty="(UIElement.RenderTransform).(TranslateTransform.X)"
                           From="20" To="0" Duration="0:0:0.18"/>
        </Storyboard>
      </VisualState>
    </VisualStateGroup>
  </VisualStateManager.VisualStateGroups>

  <Grid x:Name="DetailPanel" Opacity="0">
    <Grid.RenderTransform><TranslateTransform X="20"/></Grid.RenderTransform>
    <!-- 상세 뷰 -->
  </Grid>
</Grid>
```

코드 또는 Behavior로:
```csharp
VisualStateManager.GoToState(this, "Detail", true);
```

---

## 10. 상태 이름 컨벤션(권장)

- **CommonStates**: `Normal`, `MouseOver`, `Pressed`, `Disabled`
- **FocusStates**: `Focused`, `Unfocused`
- **SelectionStates**: `Selected`, `Unselected`
- **ValidationStates**: `Valid`, `InvalidFocused`, `InvalidUnfocused`
- **ExpansionStates**: `Expanded`, `Collapsed`
- **CheckStates**: `Checked`, `Unchecked`, `Indeterminate`
- **OrientationStates**: `Horizontal`, `Vertical`
- **ActiveStates**: `Active`, `Inactive`
- **LoadingStates**: `Idle`, `Busy`, `Loaded`

> 컨트롤 라이브러리를 배포한다면 **문서화**하세요(어떤 그룹/상태가 있는지, 언제 전환되는지).

---

## 11. 성능/안정성 체크리스트

- [ ] **Storyboard 최소화**: 상태별 Storyboard는 짧고 단순하게, **RenderTransform 우선**(LayoutTransform 지양)  
- [ ] **공유 Brush 애니메이션 금지**: 리소스 Brush는 Freeze/공유됨 → **로컬 Brush** 또는 `x:Shared="False"`  
- [ ] **GeneratedDuration** 로 공통 전환시간 정의 → 개별 Storyboard Duration 생략 가능  
- [ ] **상태 충돌 방지**: 같은 속성을 여러 그룹에서 동시에 애니메이션하지 않도록 설계  
- [ ] **초기 상태 확정**: `OnApplyTemplate` 후 `GoToState(..., false)`로 초기 상태 강제  
- [ ] **대규모 리스트**: 각 항목 템플릿의 상태 애니메이션을 **짧게**(가상화 영향 최소화)

---

## 12. 트러블슈팅

**Q1. `GoToState` 가 항상 `false` 를 반환합니다.**  
- 템플릿 안에 해당 **상태 이름이 존재**하는지, 그룹/이름 오타 여부 확인.  
- `OnApplyTemplate` 이전 호출인지 확인(템플릿 미적용이면 실패).

**Q2. 전환 애니메이션이 일부만 적용됩니다.**  
- 다른 그룹/Trigger가 **같은 속성**을 동시에 조작하고 있는지 점검.  
- **FillBehavior**/`HoldEnd` vs 다른 애니메이션이 값을 덮어쓰는지 확인.

**Q3. 마우스 상태 전환이 느리거나 튑니다.**  
- `GeneratedDuration` 을 더 짧게, **Pressed→Normal** 등 빠른 복귀 전환 별도 지정.  
- RenderTransform 사용으로 레이아웃 재계산 최소화.

**Q4. MVVM에서 상태 관리가 번거롭습니다.**  
- Behavior(`GoToStateAction`) 또는 AttachedProperty를 사용해 바인딩처럼 추상화.

---

## 13. “붙여 넣어 바로 쓰는” 스니펫

### 13.1 공용 버튼 템플릿 (라이트/다크 팔레트와 조합 가정)
```xml
<Style TargetType="Button" x:Key="DsButton">
  <Setter Property="Padding" Value="12,7"/>
  <Setter Property="Background" Value="{DynamicResource Palette.Accent}"/>
  <Setter Property="BorderBrush" Value="{DynamicResource Palette.Border}"/>
  <Setter Property="Foreground" Value="{DynamicResource Palette.Background}"/>
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="Button">
        <Grid x:Name="Root" RenderTransformOrigin="0.5,0.5">
          <Border x:Name="Chrome" CornerRadius="10"
                  Background="{TemplateBinding Background}"
                  BorderBrush="{TemplateBinding BorderBrush}"
                  BorderThickness="1.4"/>
          <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"
                            Margin="{TemplateBinding Padding}"/>
          <Grid.RenderTransform><ScaleTransform/></Grid.RenderTransform>

          <VisualStateManager.VisualStateGroups>
            <VisualStateGroup x:Name="CommonStates">
              <VisualStateGroup.Transitions>
                <VisualTransition GeneratedDuration="0:0:0.12">
                  <VisualTransition.EasingFunction>
                    <CubicEase EasingMode="EaseOut"/>
                  </VisualTransition.EasingFunction>
                </VisualTransition>
                <VisualTransition From="Pressed" To="Normal" GeneratedDuration="0:0:0.07"/>
              </VisualStateGroup.Transitions>

              <VisualState x:Name="Normal"/>
              <VisualState x:Name="MouseOver">
                <Storyboard>
                  <DoubleAnimation Storyboard.TargetName="Chrome"
                                   Storyboard.TargetProperty="Opacity"
                                   To="0.95" Duration="0:0:0"/>
                </Storyboard>
              </VisualState>
              <VisualState x:Name="Pressed">
                <Storyboard>
                  <DoubleAnimation Storyboard.TargetName="Root"
                                   Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleY)"
                                   To="0.98" Duration="0:0:0"/>
                </Storyboard>
              </VisualState>
              <VisualState x:Name="Disabled">
                <Storyboard>
                  <DoubleAnimation Storyboard.TargetName="Root" Storyboard.TargetProperty="Opacity"
                                   To="0.55" Duration="0:0:0"/>
                </Storyboard>
              </VisualState>
            </VisualStateGroup>

            <VisualStateGroup x:Name="FocusStates">
              <VisualState x:Name="Unfocused"/>
              <VisualState x:Name="Focused">
                <Storyboard>
                  <ColorAnimation Storyboard.TargetName="Chrome"
                                  Storyboard.TargetProperty="(Border.BorderBrush).(SolidColorBrush.Color)"
                                  To="#93C5FD" Duration="0:0:0"/>
                </Storyboard>
              </VisualState>
            </VisualStateGroup>
          </VisualStateManager.VisualStateGroups>
        </Grid>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```

### 13.2 페이지 상태 전환 Behavior 없이 바인딩 (AttachedProperty)
```xml
<Grid local:VisualStateHelper.State="{Binding IsDetailVisible, Converter={StaticResource BoolToState}, ConverterParameter='Detail|List'}">
  <VisualStateManager.VisualStateGroups>
    <VisualStateGroup x:Name="PageStates">
      <VisualState x:Name="List"/>
      <VisualState x:Name="Detail">
        <Storyboard>
          <DoubleAnimation Storyboard.TargetName="DetailPanel" Storyboard.TargetProperty="Opacity"
                           To="1" Duration="0:0:0.2"/>
        </Storyboard>
      </VisualState>
    </VisualStateGroup>
  </VisualStateManager.VisualStateGroups>
  <Grid x:Name="DetailPanel" Opacity="0"/>
</Grid>
```

---

## 14. 요약

- **VSM는 “상태-중심 설계”**: 상태 이름과 전환만 정리하면, 복잡한 트리거 덩어리 대신 **읽기 쉬운 템플릿**이 됩니다.  
- **`VisualTransition`** 으로 공통 전환 시간을 지정해 **부드럽고 일관된** 모션을 확보하세요.  
- **코드/MVVM와의 연결**은 `GoToState`, Behavior, AttachedProperty 등으로 유연합니다.  
- **성능**은 RenderTransform·간단한 Storyboard·충돌 없는 속성 지배로 확보하시고,  
- 필요 시 **CustomVisualStateManager**로 전환 규칙을 커스터마이즈하세요.
