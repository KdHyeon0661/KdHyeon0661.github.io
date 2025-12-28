---
layout: post
title: WPF - Visual State Manager
date: 2025-09-07 19:25:23 +0900
category: WPF
---
# WPF Visual State Manager (VSM) 완전 가이드

WPF의 Visual State Manager는 컨트롤의 다양한 상태를 선언적으로 정의하고 상태 간 전환을 애니메이션으로 구현할 수 있는 강력한 시스템입니다. 복잡한 트리거 대신 깔끔한 상태 기반 UI를 구성할 수 있게 해줍니다.

## VSM의 기본 개념 이해하기

Visual State Manager의 핵심 구성 요소는 다음과 같습니다:

- **VisualState**: 특정 UI 상태와 그 상태에서 실행될 애니메이션을 정의합니다
- **VisualStateGroup**: 관련된 상태들을 그룹으로 묶습니다 (예: CommonStates, FocusStates)
- **VisualTransition**: 상태 간 전환 시 적용될 애니메이션과 타이밍을 정의합니다

중요한 원칙은 **한 그룹 내에서는 동시에 하나의 상태만 활성화될 수 있다**는 점입니다. 여러 그룹이 있다면 각 그룹에서 하나씩 상태가 활성화될 수 있습니다.

## 기본적인 VSM 사용법

가장 간단한 VSM 예제로 Button 컨트롤 템플릿을 살펴보겠습니다:

```xml
<Style TargetType="Button" x:Key="ModernButtonStyle">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <Grid x:Name="Root">
                    <Border x:Name="BackgroundBorder"
                            Background="{TemplateBinding Background}"
                            BorderBrush="{TemplateBinding BorderBrush}"
                            BorderThickness="2"
                            CornerRadius="8"/>
                    
                    <ContentPresenter HorizontalAlignment="Center" 
                                      VerticalAlignment="Center"
                                      Margin="{TemplateBinding Padding}"/>
                </Grid>
                
                <ControlTemplate.Triggers>
                    <!-- VSM 상태 정의 -->
                    <VisualStateManager.VisualStateGroups>
                        <VisualStateGroup x:Name="CommonStates">
                            <!-- Normal 상태 (기본) -->
                            <VisualState x:Name="Normal"/>
                            
                            <!-- MouseOver 상태 -->
                            <VisualState x:Name="MouseOver">
                                <Storyboard>
                                    <ColorAnimation Storyboard.TargetName="BackgroundBorder"
                                                    Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                                    To="LightBlue" Duration="0:0:0.1"/>
                                </Storyboard>
                            </VisualState>
                            
                            <!-- Pressed 상태 -->
                            <VisualState x:Name="Pressed">
                                <Storyboard>
                                    <DoubleAnimation Storyboard.TargetName="BackgroundBorder"
                                                     Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleX)"
                                                     To="0.95" Duration="0:0:0.1"/>
                                    <DoubleAnimation Storyboard.TargetName="BackgroundBorder"
                                                     Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleY)"
                                                     To="0.95" Duration="0:0:0.1"/>
                                </Storyboard>
                            </VisualState>
                            
                            <!-- Disabled 상태 -->
                            <VisualState x:Name="Disabled">
                                <Storyboard>
                                    <DoubleAnimation Storyboard.TargetName="Root"
                                                     Storyboard.TargetProperty="Opacity"
                                                     To="0.5" Duration="0:0:0"/>
                                </Storyboard>
                            </VisualState>
                        </VisualStateGroup>
                    </VisualStateManager.VisualStateGroups>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

## 상태 전환 애니메이션 세밀하게 제어하기

VisualTransition을 사용하면 상태 간 전환을 더욱 세밀하게 제어할 수 있습니다:

```xml
<VisualStateManager.VisualStateGroups>
    <VisualStateGroup x:Name="CommonStates">
        <!-- 상태 전환 설정 -->
        <VisualStateGroup.Transitions>
            <!-- 모든 전환에 기본 0.2초 애니메이션 적용 -->
            <VisualTransition GeneratedDuration="0:0:0.2">
                <VisualTransition.EasingFunction>
                    <CubicEase EasingMode="EaseInOut"/>
                </VisualTransition.EasingFunction>
            </VisualTransition>
            
            <!-- Pressed에서 Normal로 돌아올 때는 더 빠르게 -->
            <VisualTransition From="Pressed" To="Normal" GeneratedDuration="0:0:0.1"/>
            
            <!-- Disabled 상태로 전환할 때는 애니메이션 없이 -->
            <VisualTransition To="Disabled" GeneratedDuration="0:0:0"/>
        </VisualStateGroup.Transitions>
        
        <!-- 상태 정의 -->
        <VisualState x:Name="Normal"/>
        <VisualState x:Name="MouseOver">
            <Storyboard>
                <ColorAnimation Storyboard.TargetName="BackgroundBorder"
                                Storyboard.TargetProperty="(Border.BorderBrush).(SolidColorBrush.Color)"
                                To="Blue" Duration="0:0:0"/>
            </Storyboard>
        </VisualState>
        
        <!-- 나머지 상태들... -->
    </VisualStateGroup>
</VisualStateManager.VisualStateGroups>
```

## 코드에서 상태 전환하기

코드 비하인드에서 Visual State를 제어하려면 `VisualStateManager.GoToState` 메서드를 사용합니다:

```csharp
public class CustomButton : Button
{
    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        
        // 템플릿이 적용된 후 초기 상태 설정
        VisualStateManager.GoToState(this, IsEnabled ? "Normal" : "Disabled", false);
    }
    
    protected override void OnMouseEnter(MouseEventArgs e)
    {
        base.OnMouseEnter(e);
        VisualStateManager.GoToState(this, "MouseOver", true);
    }
    
    protected override void OnMouseLeave(MouseEventArgs e)
    {
        base.OnMouseLeave(e);
        VisualStateManager.GoToState(this, "Normal", true);
    }
    
    protected override void OnIsEnabledChanged(DependencyPropertyChangedEventArgs e)
    {
        base.OnIsEnabledChanged(e);
        VisualStateManager.GoToState(this, IsEnabled ? "Normal" : "Disabled", true);
    }
}
```

## MVVM 패턴과 VSM 통합하기

MVVM 패턴에서 VSM을 사용하려면 몇 가지 접근 방법이 있습니다:

### Attached Property를 활용한 방법

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
        if (d is FrameworkElement element && e.NewValue is string stateName)
        {
            if (element.IsLoaded)
            {
                VisualStateManager.GoToState(element, stateName, true);
            }
            else
            {
                element.Loaded += (sender, args) => 
                    VisualStateManager.GoToState(element, stateName, true);
            }
        }
    }
}
```

XAML에서 사용:
```xml
<Grid local:VisualStateHelper.State="{Binding CurrentState}">
    <VisualStateManager.VisualStateGroups>
        <VisualStateGroup x:Name="AppStates">
            <VisualState x:Name="Loading"/>
            <VisualState x:Name="Loaded"/>
            <VisualState x:Name="Error"/>
        </VisualStateGroup>
    </VisualStateManager.VisualStateGroups>
</Grid>
```

## 실전 예제: 토글 스위치 컨트롤 만들기

커스텀 컨트롤과 VSM을 함께 사용하는 완전한 예제입니다:

### ToggleSwitch.cs
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
        var toggleSwitch = (ToggleSwitch)d;
        toggleSwitch.UpdateVisualState(true);
    }

    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        
        // 템플릿 요소에 이벤트 연결
        if (GetTemplateChild("PART_Track") is FrameworkElement track)
        {
            track.MouseLeftButtonDown += (sender, e) => IsOn = !IsOn;
        }
        
        // 초기 상태 설정
        UpdateVisualState(false);
    }

    private void UpdateVisualState(bool useTransitions)
    {
        string stateName = IsEnabled ? (IsOn ? "On" : "Off") : "Disabled";
        VisualStateManager.GoToState(this, stateName, useTransitions);
    }
}
```

### Generic.xaml
```xml
<Style TargetType="{x:Type local:ToggleSwitch}">
    <Setter Property="Width" Value="60"/>
    <Setter Property="Height" Value="34"/>
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="{x:Type local:ToggleSwitch}">
                <Grid x:Name="Root">
                    <!-- 배경 트랙 -->
                    <Border x:Name="PART_Track" 
                            Background="#E5E7EB" 
                            CornerRadius="17"
                            Width="60" Height="34"/>
                    
                    <!-- 토글 버튼 -->
                    <Ellipse x:Name="Thumb" 
                             Width="26" Height="26" 
                             Fill="White" 
                             HorizontalAlignment="Left"
                             Margin="4"/>
                </Grid>
                
                <ControlTemplate.Triggers>
                    <VisualStateManager.VisualStateGroups>
                        <VisualStateGroup x:Name="ToggleStates">
                            <!-- 상태 전환 애니메이션 -->
                            <VisualStateGroup.Transitions>
                                <VisualTransition GeneratedDuration="0:0:0.2">
                                    <VisualTransition.EasingFunction>
                                        <CubicEase EasingMode="EaseOut"/>
                                    </VisualTransition.EasingFunction>
                                </VisualTransition>
                            </VisualStateGroup.Transitions>
                            
                            <!-- Off 상태 -->
                            <VisualState x:Name="Off">
                                <Storyboard>
                                    <ThicknessAnimation Storyboard.TargetName="Thumb"
                                                       Storyboard.TargetProperty="Margin"
                                                       To="4,4,30,4" Duration="0:0:0"/>
                                </Storyboard>
                            </VisualState>
                            
                            <!-- On 상태 -->
                            <VisualState x:Name="On">
                                <Storyboard>
                                    <ThicknessAnimation Storyboard.TargetName="Thumb"
                                                       Storyboard.TargetProperty="Margin"
                                                       To="30,4,4,4" Duration="0:0:0"/>
                                    <ColorAnimation Storyboard.TargetName="PART_Track"
                                                   Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                                   To="#10B981" Duration="0:0:0"/>
                                </Storyboard>
                            </VisualState>
                            
                            <!-- Disabled 상태 -->
                            <VisualState x:Name="Disabled">
                                <Storyboard>
                                    <DoubleAnimation Storyboard.TargetName="Root"
                                                   Storyboard.TargetProperty="Opacity"
                                                   To="0.5" Duration="0:0:0"/>
                                </Storyboard>
                            </VisualState>
                        </VisualStateGroup>
                    </VisualStateManager.VisualStateGroups>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

## 여러 상태 그룹을 함께 사용하기

복잡한 컨트롤에서는 여러 상태 그룹을 함께 사용할 수 있습니다:

```xml
<VisualStateManager.VisualStateGroups>
    <!-- 상호작용 상태 -->
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
                <DoubleAnimation Storyboard.TargetName="FocusVisual"
                                 Storyboard.TargetProperty="Opacity"
                                 To="1" Duration="0:0:0.1"/>
            </Storyboard>
        </VisualState>
    </VisualStateGroup>
    
    <!-- 검증 상태 -->
    <VisualStateGroup x:Name="ValidationStates">
        <VisualState x:Name="Valid"/>
        <VisualState x:Name="Invalid">
            <Storyboard>
                <ColorAnimation Storyboard.TargetName="Border"
                                Storyboard.TargetProperty="(Border.BorderBrush).(SolidColorBrush.Color)"
                                To="#EF4444" Duration="0:0:0.1"/>
            </Storyboard>
        </VisualState>
    </VisualStateGroup>
</VisualStateManager.VisualStateGroups>
```

## 페이지 레벨에서 VSM 사용하기

VSM은 컨트롤 뿐만 아니라 페이지나 사용자 정의 뷰에서도 유용하게 사용할 수 있습니다:

```xml
<Grid x:Name="MainGrid">
    <VisualStateManager.VisualStateGroups>
        <VisualStateGroup x:Name="ViewStates">
            <VisualState x:Name="List">
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetName="DetailView"
                                     Storyboard.TargetProperty="Opacity"
                                     To="0" Duration="0:0:0.2"/>
                    <DoubleAnimation Storyboard.TargetName="ListView"
                                     Storyboard.TargetProperty="Opacity"
                                     To="1" Duration="0:0:0.2"/>
                </Storyboard>
            </VisualState>
            
            <VisualState x:Name="Detail">
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetName="DetailView"
                                     Storyboard.TargetProperty="Opacity"
                                     To="1" Duration="0:0:0.2"/>
                    <DoubleAnimation Storyboard.TargetName="ListView"
                                     Storyboard.TargetProperty="Opacity"
                                     To="0" Duration="0:0:0.2"/>
                </Storyboard>
            </VisualState>
        </VisualStateGroup>
    </VisualStateManager.VisualStateGroups>
    
    <!-- 리스트 뷰 -->
    <Grid x:Name="ListView" Opacity="1">
        <!-- 리스트 콘텐츠 -->
    </Grid>
    
    <!-- 상세 뷰 -->
    <Grid x:Name="DetailView" Opacity="0">
        <!-- 상세 콘텐츠 -->
    </Grid>
</Grid>
```

코드에서 상태 전환:
```csharp
// 리스트 뷰로 전환
VisualStateManager.GoToState(MainGrid, "List", true);

// 상세 뷰로 전환
VisualStateManager.GoToState(MainGrid, "Detail", true);
```

## 성능 최적화를 위한 팁

1. **RenderTransform 사용**: LayoutTransform 대신 RenderTransform을 사용하여 레이아웃 재계산을 피하세요
2. **애니메이션 최적화**: 불필요한 애니메이션을 피하고, 짧은 지속시간을 사용하세요
3. **상태 그룹 분리**: 서로 관련 없는 상태는 별도의 그룹으로 분리하세요
4. **초기 상태 설정**: OnApplyTemplate에서 초기 상태를 명시적으로 설정하세요
5. **공유 리소스 주의**: 애니메이션 대상이 되는 Brush는 공유하지 마세요

## 일반적인 문제 해결

### 문제 1: 상태 전환이 작동하지 않음
**원인**: 상태 이름 오타, 템플릿이 적용되지 않음
**해결**: 상태 이름 확인, OnApplyTemplate 이후에 GoToState 호출

### 문제 2: 애니메이션이 부드럽지 않음
**원인**: 너무 짧은 지속시간, 적절하지 않은 EasingFunction
**해결**: 지속시간 조정, 적절한 EasingFunction 사용

### 문제 3: 여러 애니메이션이 충돌
**원인**: 같은 속성을 여러 상태에서 애니메이션
**해결**: 상태 그룹을 적절히 분리하거나 애니메이션 대상 변경

## 결론

WPF의 Visual State Manager는 상태 기반 UI를 구성하는 강력하고 유연한 도구입니다. 복잡한 트리거 대신 직관적인 상태 정의를 통해 가독성 높은 XAML 코드를 작성할 수 있으며, 상태 전환 애니메이션을 통해 풍부한 사용자 경험을 제공할 수 있습니다.

VSM을 효과적으로 사용하기 위해서는:
1. 상태와 상태 그룹을 논리적으로 구성하기
2. VisualTransition을 활용하여 부드러운 상태 전환 구현하기
3. 코드 비하인드나 MVVM 패턴과의 적절한 통합 방법 선택하기
4. 성능을 고려한 애니메이션 설계하기

이러한 원칙들을 준수하면 일관되고 반응성이 뛰어난 WPF 애플리케이션을 개발할 수 있습니다. VSM은 특히 커스텀 컨트롤 개발이나 복잡한 UI 상태 관리가 필요한 프로젝트에서 그 진가를 발휘합니다.