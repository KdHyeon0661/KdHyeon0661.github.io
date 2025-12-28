---
layout: post
title: WPF - Styles & ControlTemplate
date: 2025-09-02 23:25:23 +0900
category: WPF
---
# WPF Styles와 ControlTemplate: 전문가를 위한 완벽 가이드

WPF의 스타일링 시스템은 단순한 CSS 같은 미적 장식을 넘어, 애플리케이션의 시각적 계층 구조와 유지보수성을 결정하는 핵심 아키텍처 요소입니다. 스타일과 컨트롤 템플릿이 어떻게 협력하며, 실제 프로젝트에서 어떤 패턴으로 적용되어야 하는지 깊이 있게 살펴보겠습니다.

## 스타일과 컨트롤 템플릿의 근본적 차이 이해하기

WPF에서 스타일(Style)과 컨트롤 템플릿(ControlTemplate)은 서로 보완적이지만 명확히 구분되는 역할을 합니다. 스타일은 컨트롤의 속성 값을 일괄 설정하는 반면, 컨트롤 템플릿은 컨트롤의 전체 시각적 구조를 재정의합니다. 

이 구분을 이해하는 가장 좋은 방법은 실제 예제를 통해 살펴보는 것입니다:

```xml
<!-- 스타일: Button의 속성 값만 변경 -->
<Style TargetType="Button" x:Key="StyledButton">
    <Setter Property="Background" Value="#2563EB"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="Padding" Value="12,8"/>
    <Setter Property="FontWeight" Value="SemiBold"/>
    <Setter Property="BorderThickness" Value="2"/>
    <Setter Property="BorderBrush" Value="#1E40AF"/>
</Style>

<!-- 컨트롤 템플릿: Button의 시각적 구조 자체를 재정의 -->
<Style TargetType="Button" x:Key="TemplatedButton">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <!-- 완전히 새로운 시각적 구조 -->
                <Grid>
                    <!-- 배경 레이어 -->
                    <Rectangle x:Name="BackgroundLayer" 
                               Fill="{TemplateBinding Background}" 
                               RadiusX="8" RadiusY="8"/>
                    
                    <!-- 그림자 효과 -->
                    <Rectangle x:Name="ShadowLayer" 
                               Fill="#40000000" 
                               Margin="0,2,0,0" 
                               RadiusX="8" RadiusY="8"/>
                    
                    <!-- 컨텐츠 표시 -->
                    <ContentPresenter HorizontalAlignment="Center" 
                                      VerticalAlignment="Center"
                                      Margin="{TemplateBinding Padding}"/>
                    
                    <!-- 강조선 -->
                    <Border x:Name="AccentBorder" 
                            Height="3" 
                            VerticalAlignment="Bottom" 
                            Background="#60FFFFFF"
                            CornerRadius="1,1,8,8"/>
                </Grid>
                
                <!-- 템플릿 트리거: 마우스 인터랙션에 따른 시각적 변화 -->
                <ControlTemplate.Triggers>
                    <Trigger Property="IsMouseOver" Value="True">
                        <Setter TargetName="BackgroundLayer" 
                                Property="Fill" 
                                Value="#1D4ED8"/>
                        <Setter TargetName="AccentBorder" 
                                Property="Background" 
                                Value="#90FFFFFF"/>
                    </Trigger>
                    <Trigger Property="IsPressed" Value="True">
                        <Setter TargetName="BackgroundLayer" 
                                Property="Fill" 
                                Value="#1E40AF"/>
                        <Setter TargetName="ShadowLayer" 
                                Property="Margin" 
                                Value="0,0,0,0"/>
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
    
    <!-- 템플릿과 함께 사용할 속성 값들 -->
    <Setter Property="Background" Value="#2563EB"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="Padding" Value="12,8"/>
</Style>
```

이 예제에서 볼 수 있듯, 스타일은 기존 버튼의 모양을 개선하는 반면, 컨트롤 템플릿은 버튼의 시각적 구현을 완전히 새로 만듭니다. 중요한 것은 컨트롤 템플릿이 버튼의 기능(클릭 이벤트, 명령 실행 등)은 변경하지 않는다는 점입니다.

## 암시적 스타일: 일관된 디자인 시스템의 기초

암시적 스타일은 애플리케이션 전체에 일관된 디자인 언어를 적용하는 강력한 도구입니다. 특정 타입의 모든 컨트롤에 자동으로 적용되는 이 기능은 디자인 시스템 구현의 핵심입니다.

```xml
<!-- 애플리케이션 수준에서 기본 디자인 시스템 정의 -->
<Application.Resources>
    <ResourceDictionary>
        <!-- 기본 색상 팔레트 -->
        <Color x:Key="PrimaryColor">#2563EB</Color>
        <Color x:Key="SecondaryColor">#64748B</Color>
        <Color x:Key="SuccessColor">#10B981</Color>
        <Color x:Key="DangerColor">#EF4444</Color>
        <Color x:Key="WarningColor">#F59E0B</Color>
        
        <!-- 브러시 리소스 -->
        <SolidColorBrush x:Key="PrimaryBrush" Color="{StaticResource PrimaryColor}"/>
        <SolidColorBrush x:Key="SecondaryBrush" Color="{StaticResource SecondaryColor}"/>
        <SolidColorBrush x:Key="SuccessBrush" Color="{StaticResource SuccessColor}"/>
        <SolidColorBrush x:Key="DangerBrush" Color="{StaticResource DangerColor}"/>
        <SolidColorBrush x:Key="WarningBrush" Color="{StaticResource WarningColor}"/>
        
        <!-- 텍스트 색상 -->
        <SolidColorBrush x:Key="TextPrimaryBrush" Color="#1F2937"/>
        <SolidColorBrush x:Key="TextSecondaryBrush" Color="#6B7280"/>
        
        <!-- 배경 색상 -->
        <SolidColorBrush x:Key="BackgroundPrimaryBrush" Color="#FFFFFF"/>
        <SolidColorBrush x:Key="BackgroundSecondaryBrush" Color="#F9FAFB"/>
        
        <!-- 경계선 색상 -->
        <SolidColorBrush x:Key="BorderPrimaryBrush" Color="#E5E7EB"/>
        <SolidColorBrush x:Key="BorderFocusBrush" Color="#93C5FD"/>
        
        <!-- 모든 버튼에 적용되는 기본 스타일 -->
        <Style TargetType="Button">
            <Setter Property="Foreground" Value="{StaticResource TextPrimaryBrush}"/>
            <Setter Property="Background" Value="{StaticResource BackgroundSecondaryBrush}"/>
            <Setter Property="BorderBrush" Value="{StaticResource BorderPrimaryBrush}"/>
            <Setter Property="BorderThickness" Value="1"/>
            <Setter Property="Padding" Value="12,8"/>
            <Setter Property="Margin" Value="4"/>
            <Setter Property="FontSize" Value="14"/>
            <Setter Property="Cursor" Value="Hand"/>
            <Setter Property="FocusVisualStyle">
                <Setter.Value>
                    <Style>
                        <Setter Property="Control.Template">
                            <Setter.Value>
                                <ControlTemplate>
                                    <Rectangle Stroke="{StaticResource BorderFocusBrush}" 
                                               StrokeThickness="2" 
                                               StrokeDashArray="2 2"
                                               Margin="-2"
                                               SnapsToDevicePixels="True"/>
                                </ControlTemplate>
                            </Setter.Value>
                        </Setter>
                    </Style>
                </Setter.Value>
            </Setter>
            
            <!-- 상태에 따른 스타일 변화 -->
            <Style.Triggers>
                <Trigger Property="IsMouseOver" Value="True">
                    <Setter Property="Background" Value="{StaticResource PrimaryBrush}"/>
                    <Setter Property="Foreground" Value="White"/>
                    <Setter Property="BorderBrush" Value="{StaticResource PrimaryBrush}"/>
                </Trigger>
                <Trigger Property="IsPressed" Value="True">
                    <Setter Property="Background" Value="#1D4ED8"/>
                    <Setter Property="RenderTransform">
                        <Setter.Value>
                            <ScaleTransform ScaleX="0.98" ScaleY="0.98"/>
                        </Setter.Value>
                    </Setter>
                    <Setter Property="RenderTransformOrigin" Value="0.5,0.5"/>
                </Trigger>
                <Trigger Property="IsEnabled" Value="False">
                    <Setter Property="Opacity" Value="0.6"/>
                    <Setter Property="Background" Value="#F3F4F6"/>
                    <Setter Property="Foreground" Value="#9CA3AF"/>
                </Trigger>
            </Style.Triggers>
        </Style>
        
        <!-- 모든 TextBox에 적용되는 기본 스타일 -->
        <Style TargetType="TextBox">
            <Setter Property="Background" Value="{StaticResource BackgroundPrimaryBrush}"/>
            <Setter Property="Foreground" Value="{StaticResource TextPrimaryBrush}"/>
            <Setter Property="BorderBrush" Value="{StaticResource BorderPrimaryBrush}"/>
            <Setter Property="BorderThickness" Value="1"/>
            <Setter Property="Padding" Value="8,6"/>
            <Setter Property="FontSize" Value="14"/>
            <Setter Property="SnapsToDevicePixels" Value="True"/>
            <Setter Property="TextWrapping" Value="NoWrap"/>
            <Setter Property="VerticalContentAlignment" Value="Center"/>
            
            <!-- 포커스 상태 -->
            <Style.Triggers>
                <Trigger Property="IsKeyboardFocused" Value="True">
                    <Setter Property="BorderBrush" Value="{StaticResource PrimaryBrush}"/>
                    <Setter Property="BorderThickness" Value="2"/>
                </Trigger>
                
                <!-- 검증 오류 상태 -->
                <Trigger Property="Validation.HasError" Value="True">
                    <Setter Property="BorderBrush" Value="{StaticResource DangerBrush}"/>
                    <Setter Property="BorderThickness" Value="2"/>
                    <Setter Property="ToolTip" 
                            Value="{Binding RelativeSource={RelativeSource Self}, 
                                   Path=(Validation.Errors)[0].ErrorContent}"/>
                </Trigger>
            </Style.Triggers>
        </Style>
    </ResourceDictionary>
</Application.Resources>
```

이러한 암시적 스타일 설정은 애플리케이션 전반에 걸쳐 일관된 디자인을 보장합니다. 새로운 버튼이나 텍스트 상자를 추가할 때마다 별도의 스타일 지정 없이도 통일된 모습을 유지할 수 있습니다.

## BasedOn을 활용한 스타일 상속 시스템

대규모 애플리케이션에서는 스타일의 상속 계층 구조를 체계적으로 설계하는 것이 중요합니다. `BasedOn` 속성을 활용하면 기본 스타일에서 파생된 다양한 변형 스타일을 만들 수 있습니다.

```xml
<!-- 기본 버튼 스타일 (모든 버튼의 토대) -->
<Style x:Key="BaseButtonStyle" TargetType="Button">
    <Setter Property="FontFamily" Value="Segoe UI"/>
    <Setter Property="FontSize" Value="14"/>
    <Setter Property="FontWeight" Value="SemiBold"/>
    <Setter Property="Padding" Value="12,8"/>
    <Setter Property="Margin" Value="4"/>
    <Setter Property="HorizontalContentAlignment" Value="Center"/>
    <Setter Property="VerticalContentAlignment" Value="Center"/>
    <Setter Property="Cursor" Value="Hand"/>
    <Setter Property="BorderThickness" Value="1"/>
    <Setter Property="BorderBrush" Value="Transparent"/>
    <Setter Property="Background" Value="#F3F4F6"/>
    <Setter Property="Foreground" Value="#374151"/>
    
    <!-- 기본 애니메이션 트랜스폼 -->
    <Setter Property="RenderTransform">
        <Setter.Value>
            <ScaleTransform ScaleX="1" ScaleY="1"/>
        </Setter.Value>
    </Setter>
    <Setter Property="RenderTransformOrigin" Value="0.5,0.5"/>
    
    <!-- 호버 효과 -->
    <Style.Triggers>
        <Trigger Property="IsMouseOver" Value="True">
            <Trigger.EnterActions>
                <BeginStoryboard>
                    <Storyboard>
                        <DoubleAnimation Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleX)"
                                         To="1.02" Duration="0:0:0.1"/>
                        <DoubleAnimation Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleY)"
                                         To="1.02" Duration="0:0:0.1"/>
                    </Storyboard>
                </BeginStoryboard>
            </Trigger.EnterActions>
            <Trigger.ExitActions>
                <BeginStoryboard>
                    <Storyboard>
                        <DoubleAnimation Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleX)"
                                         To="1.0" Duration="0:0:0.2"/>
                        <DoubleAnimation Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleY)"
                                         To="1.0" Duration="0:0:0.2"/>
                    </Storyboard>
                </BeginStoryboard>
            </Trigger.ExitActions>
        </Trigger>
        
        <Trigger Property="IsPressed" Value="True">
            <Setter Property="RenderTransform">
                <Setter.Value>
                    <ScaleTransform ScaleX="0.98" ScaleY="0.98"/>
                </Setter.Value>
            </Setter>
        </Trigger>
    </Style.Triggers>
</Style>

<!-- 기본 스타일을 상속한 다양한 버튼 타입들 -->
<Style x:Key="PrimaryButton" TargetType="Button" BasedOn="{StaticResource BaseButtonStyle}">
    <Setter Property="Background" Value="{StaticResource PrimaryBrush}"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="BorderBrush" Value="{StaticResource PrimaryBrush}"/>
    
    <Style.Triggers>
        <Trigger Property="IsMouseOver" Value="True">
            <Setter Property="Background" Value="#1D4ED8"/>
            <Setter Property="BorderBrush" Value="#1D4ED8"/>
        </Trigger>
        <Trigger Property="IsPressed" Value="True">
            <Setter Property="Background" Value="#1E40AF"/>
            <Setter Property="BorderBrush" Value="#1E40AF"/>
        </Trigger>
    </Style.Triggers>
</Style>

<Style x:Key="SecondaryButton" TargetType="Button" BasedOn="{StaticResource BaseButtonStyle}">
    <Setter Property="Background" Value="{StaticResource SecondaryBrush}"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="BorderBrush" Value="{StaticResource SecondaryBrush}"/>
    
    <Style.Triggers>
        <Trigger Property="IsMouseOver" Value="True">
            <Setter Property="Background" Value="#475569"/>
            <Setter Property="BorderBrush" Value="#475569"/>
        </Trigger>
    </Style.Triggers>
</Style>

<Style x:Key="SuccessButton" TargetType="Button" BasedOn="{StaticResource BaseButtonStyle}">
    <Setter Property="Background" Value="{StaticResource SuccessBrush}"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="BorderBrush" Value="{StaticResource SuccessBrush}"/>
    
    <Style.Triggers>
        <Trigger Property="IsMouseOver" Value="True">
            <Setter Property="Background" Value="#0D9488"/>
            <Setter Property="BorderBrush" Value="#0D9488"/>
        </Trigger>
    </Style.Triggers>
</Style>

<Style x:Key="DangerButton" TargetType="Button" BasedOn="{StaticResource BaseButtonStyle}">
    <Setter Property="Background" Value="{StaticResource DangerBrush}"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="BorderBrush" Value="{StaticResource DangerBrush}"/>
    
    <Style.Triggers>
        <Trigger Property="IsMouseOver" Value="True">
            <Setter Property="Background" Value="#DC2626"/>
            <Setter Property="BorderBrush" Value="#DC2626"/>
        </Trigger>
    </Style.Triggers>
</Style>

<Style x:Key="OutlineButton" TargetType="Button" BasedOn="{StaticResource BaseButtonStyle}">
    <Setter Property="Background" Value="Transparent"/>
    <Setter Property="Foreground" Value="{StaticResource PrimaryBrush}"/>
    <Setter Property="BorderBrush" Value="{StaticResource PrimaryBrush}"/>
    <Setter Property="BorderThickness" Value="2"/>
    
    <Style.Triggers>
        <Trigger Property="IsMouseOver" Value="True">
            <Setter Property="Background" Value="{StaticResource PrimaryBrush}"/>
            <Setter Property="Foreground" Value="White"/>
        </Trigger>
    </Style.Triggers>
</Style>

<Style x:Key="GhostButton" TargetType="Button" BasedOn="{StaticResource BaseButtonStyle}">
    <Setter Property="Background" Value="Transparent"/>
    <Setter Property="Foreground" Value="{StaticResource TextSecondaryBrush}"/>
    <Setter Property="BorderBrush" Value="Transparent"/>
    
    <Style.Triggers>
        <Trigger Property="IsMouseOver" Value="True">
            <Setter Property="Background" Value="Transparent"/>
            <Setter Property="Foreground" Value="{StaticResource PrimaryBrush}"/>
        </Trigger>
    </Style.Triggers>
</Style>

<!-- 아이콘 버튼 -->
<Style x:Key="IconButton" TargetType="Button" BasedOn="{StaticResource BaseButtonStyle}">
    <Setter Property="Width" Value="40"/>
    <Setter Property="Height" Value="40"/>
    <Setter Property="Padding" Value="0"/>
    <Setter Property="Background" Value="Transparent"/>
    <Setter Property="BorderBrush" Value="Transparent"/>
    <Setter Property="Foreground" Value="{StaticResource TextSecondaryBrush}"/>
    
    <Style.Triggers>
        <Trigger Property="IsMouseOver" Value="True">
            <Setter Property="Background" Value="#F3F4F6"/>
            <Setter Property="Foreground" Value="{StaticResource PrimaryBrush}"/>
        </Trigger>
    </Style.Triggers>
</Style>
```

이러한 상속 체계는 여러 가지 이점을 제공합니다:
1. **일관성 유지**: 모든 버튼이 동일한 기본 속성을 공유
2. **유지보수성 향상**: 기본 스타일만 변경하면 모든 파생 스타일이 자동으로 업데이트
3. **확장성**: 새로운 버튼 타입이 필요할 때 쉽게 추가 가능

## 데이터 트리거와 멀티 트리거: MVVM 패턴과의 완벽한 통합

WPF의 강력한 데이터 바인딩 시스템과 스타일 트리거의 결합은 MVVM 아키텍처에서 뛰어난 유연성을 제공합니다.

```xml
<!-- ViewModel 기반의 조건부 스타일링 -->
<Style TargetType="Button" x:Key="ViewModelAwareButton">
    <!-- 기본 상태 -->
    <Setter Property="Background" Value="{StaticResource SecondaryBrush}"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="IsEnabled" Value="False"/>
    
    <!-- 데이터 트리거: ViewModel 속성에 반응 -->
    <Style.Triggers>
        <!-- CanExecute 속성이 true일 때만 활성화 -->
        <DataTrigger Binding="{Binding CanExecuteAction}" Value="True">
            <Setter Property="IsEnabled" Value="True"/>
            <Setter Property="Background" Value="{StaticResource PrimaryBrush}"/>
            <Setter Property="Cursor" Value="Hand"/>
        </DataTrigger>
        
        <!-- 작업 진행 중인 상태 -->
        <DataTrigger Binding="{Binding IsProcessing}" Value="True">
            <Setter Property="Content">
                <Setter.Value>
                    <StackPanel Orientation="Horizontal" Spacing="8">
                        <TextBlock Text="처리 중..."/>
                        <Ellipse Width="12" Height="12">
                            <Ellipse.Fill>
                                <SolidColorBrush Color="{StaticResource PrimaryColor}"/>
                            </Ellipse.Fill>
                            <Ellipse.RenderTransform>
                                <RotateTransform CenterX="6" CenterY="6"/>
                            </Ellipse.RenderTransform>
                            <Ellipse.Triggers>
                                <EventTrigger RoutedEvent="Loaded">
                                    <BeginStoryboard>
                                        <Storyboard RepeatBehavior="Forever">
                                            <DoubleAnimation Storyboard.TargetProperty="(Ellipse.RenderTransform).(RotateTransform.Angle)"
                                                             From="0" To="360" Duration="0:0:1"/>
                                        </Storyboard>
                                    </BeginStoryboard>
                                </EventTrigger>
                            </Ellipse.Triggers>
                        </Ellipse>
                    </StackPanel>
                </Setter.Value>
            </Setter>
            <Setter Property="IsEnabled" Value="False"/>
            <Setter Property="Background" Value="{StaticResource SecondaryBrush}"/>
        </DataTrigger>
        
        <!-- 작업 완료 상태 -->
        <DataTrigger Binding="{Binding IsCompleted}" Value="True">
            <Setter Property="Background" Value="{StaticResource SuccessBrush}"/>
            <Setter Property="Content">
                <Setter.Value>
                    <StackPanel Orientation="Horizontal" Spacing="8">
                        <TextBlock Text="완료됨"/>
                        <Path Data="M9 12l2 2 4-4" Stroke="White" StrokeThickness="2"
                              Width="16" Height="16" Stretch="Uniform"/>
                    </StackPanel>
                </Setter.Value>
            </Setter>
        </DataTrigger>
        
        <!-- 복합 조건: 여러 속성의 조합에 반응 -->
        <MultiDataTrigger>
            <MultiDataTrigger.Conditions>
                <Condition Binding="{Binding IsSelected}" Value="True"/>
                <Condition Binding="{Binding IsValid}" Value="True"/>
                <Condition Binding="{Binding IsEditable}" Value="True"/>
            </MultiDataTrigger.Conditions>
            <Setter Property="Background" Value="#22C55E"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="BorderBrush" Value="#16A34A"/>
            <Setter Property="BorderThickness" Value="2"/>
        </MultiDataTrigger>
        
        <!-- 사용자 권한에 따른 스타일 변화 -->
        <DataTrigger Binding="{Binding CurrentUser.Role}" Value="Admin">
            <Setter Property="Background" Value="#8B5CF6"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="BorderBrush" Value="#7C3AED"/>
        </DataTrigger>
        
        <DataTrigger Binding="{Binding CurrentUser.Role}" Value="Editor">
            <Setter Property="Background" Value="#10B981"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="BorderBrush" Value="#0D9488"/>
        </DataTrigger>
        
        <!-- 우선순위에 따른 색상 변화 -->
        <DataTrigger Binding="{Binding Priority}" Value="High">
            <Setter Property="Background" Value="#EF4444"/>
            <Setter Property="Foreground" Value="White"/>
        </DataTrigger>
        
        <DataTrigger Binding="{Binding Priority}" Value="Medium">
            <Setter Property="Background" Value="#F59E0B"/>
            <Setter Property="Foreground" Value="White"/>
        </DataTrigger>
        
        <DataTrigger Binding="{Binding Priority}" Value="Low">
            <Setter Property="Background" Value="#6B7280"/>
            <Setter Property="Foreground" Value="White"/>
        </DataTrigger>
    </Style.Triggers>
</Style>
```

이 스타일은 ViewModel의 다양한 속성에 반응하여 버튼의 모양과 상태를 동적으로 변경합니다. 이 접근 방식은 UI 로직을 코드 비하인드에서 ViewModel로 완전히 이동시키는 MVVM 패턴의 핵심 원칙을 구현합니다.

## 고급 컨트롤 템플릿 디자인: 현대적인 UI 컴포넌트 만들기

컨트롤 템플릿을 사용하면 표준 WPF 컨트롤의 시각적 표현을 완전히 재정의할 수 있습니다. 다음은 현대적인 디자인 시스템에 맞는 토글 스위치 컨트롤 템플릿의 예입니다:

```xml
<!-- 현대적인 토글 스위치 템플릿 -->
<Style TargetType="ToggleButton" x:Key="ModernToggleSwitch">
    <!-- 기본 속성 설정 -->
    <Setter Property="Width" Value="52"/>
    <Setter Property="Height" Value="28"/>
    <Setter Property="Background" Value="Transparent"/>
    <Setter Property="BorderBrush" Value="Transparent"/>
    <Setter Property="Foreground" Value="Transparent"/>
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="ToggleButton">
                <!-- 토글 스위치의 전체 구조 -->
                <Grid x:Name="RootGrid">
                    <!-- 애니메이션을 위한 트랜스폼 그룹 -->
                    <Grid.RenderTransform>
                        <TransformGroup>
                            <ScaleTransform/>
                            <TranslateTransform/>
                        </TransformGroup>
                    </Grid.RenderTransform>
                    
                    <!-- 배경 트랙 -->
                    <Border x:Name="TrackBackground"
                            Width="{TemplateBinding Width}"
                            Height="{TemplateBinding Height}"
                            CornerRadius="14"
                            Background="#E5E7EB">
                        <!-- 꺼짐 상태의 그라데이션 -->
                        <Border.Background>
                            <LinearGradientBrush StartPoint="0,0" EndPoint="1,0">
                                <GradientStop Color="#E5E7EB" Offset="0"/>
                                <GradientStop Color="#D1D5DB" Offset="1"/>
                            </LinearGradientBrush>
                        </Border.Background>
                    </Border>
                    
                    <!-- 활성화된 상태의 트랙 -->
                    <Border x:Name="ActiveTrack"
                            Width="{TemplateBinding Width}"
                            Height="{TemplateBinding Height}"
                            CornerRadius="14"
                            Background="{StaticResource SuccessBrush}"
                            Opacity="0">
                        <Border.Background>
                            <LinearGradientBrush StartPoint="0,0" EndPoint="1,0">
                                <GradientStop Color="#10B981" Offset="0"/>
                                <GradientStop Color="#0D9488" Offset="1"/>
                            </LinearGradientBrush>
                        </Border.Background>
                    </Border>
                    
                    <!-- 스위치 핸들 (움직이는 부분) -->
                    <Border x:Name="Thumb"
                            Width="24"
                            Height="24"
                            CornerRadius="12"
                            HorizontalAlignment="Left"
                            Margin="2"
                            Background="White"
                            RenderTransformOrigin="0.5,0.5">
                        <!-- 그림자 효과 -->
                        <Border.Effect>
                            <DropShadowEffect ShadowDepth="1" 
                                              BlurRadius="4" 
                                              Opacity="0.3" 
                                              Color="#000000"/>
                        </Border.Effect>
                        
                        <!-- 클릭 시 리플 효과 -->
                        <Border.Resources>
                            <Storyboard x:Key="RippleEffect">
                                <DoubleAnimation Storyboard.TargetName="RippleEllipse"
                                                 Storyboard.TargetProperty="Opacity"
                                                 From="0.4" To="0" Duration="0:0:0.3"/>
                                <DoubleAnimation Storyboard.TargetName="RippleEllipse"
                                                 Storyboard.TargetProperty="Width"
                                                 From="0" To="48" Duration="0:0:0.3"/>
                                <DoubleAnimation Storyboard.TargetName="RippleEllipse"
                                                 Storyboard.TargetProperty="Height"
                                                 From="0" To="48" Duration="0:0:0.3"/>
                            </Storyboard>
                        </Border.Resources>
                        
                        <Grid>
                            <!-- 리플 효과를 위한 타원 -->
                            <Ellipse x:Name="RippleEllipse"
                                     Width="0" Height="0"
                                     Fill="White"
                                     Opacity="0"
                                     HorizontalAlignment="Center"
                                     VerticalAlignment="Center"/>
                            
                            <!-- 내부 아이콘 (선택적) -->
                            <Viewbox Width="12" Height="12">
                                <Canvas Width="24" Height="24">
                                    <!-- 체크 아이콘 -->
                                    <Path x:Name="CheckIcon" 
                                          Data="M9 16.17L4.83 12l-1.42 1.41L9 19 21 7l-1.41-1.41z"
                                          Fill="{StaticResource SuccessBrush}"
                                          Opacity="0"/>
                                    <!-- 엑스 아이콘 -->
                                    <Path x:Name="CloseIcon"
                                          Data="M19 6.41L17.59 5 12 10.59 6.41 5 5 6.41 10.59 12 5 17.59 6.41 19 12 13.41 17.59 19 19 17.59 13.41 12z"
                                          Fill="#EF4444"
                                          Opacity="0"/>
                                </Canvas>
                            </Viewbox>
                        </Grid>
                    </Border>
                    
                    <!-- 상태 표시 레이블 -->
                    <Grid x:Name="LabelGrid" 
                          HorizontalAlignment="Right" 
                          Margin="0,0,8,0"
                          VerticalAlignment="Center">
                        <TextBlock x:Name="OnLabel" 
                                   Text="ON" 
                                   Foreground="White" 
                                   FontSize="10" 
                                   FontWeight="Bold"
                                   Opacity="0"/>
                        <TextBlock x:Name="OffLabel" 
                                   Text="OFF" 
                                   Foreground="#6B7280" 
                                   FontSize="10" 
                                   FontWeight="Bold"
                                   Margin="8,0,0,0"/>
                    </Grid>
                </Grid>
                
                <!-- 템플릿 트리거 및 Visual State Manager -->
                <ControlTemplate.Triggers>
                    <!-- 초기 상태 설정 -->
                    <Trigger Property="Template" Value="{x:Null}">
                        <Setter TargetName="RootGrid" Property="RenderTransform">
                            <Setter.Value>
                                <TransformGroup>
                                    <ScaleTransform ScaleX="1" ScaleY="1"/>
                                    <TranslateTransform X="0" Y="0"/>
                                </TransformGroup>
                            </Setter.Value>
                        </Setter>
                    </Trigger>
                    
                    <!-- 체크된 상태 -->
                    <Trigger Property="IsChecked" Value="True">
                        <!-- 핸들 위치 이동 -->
                        <Trigger.EnterActions>
                            <BeginStoryboard>
                                <Storyboard>
                                    <!-- 핸들 이동 애니메이션 -->
                                    <DoubleAnimation Storyboard.TargetName="Thumb"
                                                     Storyboard.TargetProperty="(UIElement.RenderTransform).(TranslateTransform.X)"
                                                     To="22" Duration="0:0:0.2"/>
                                    
                                    <!-- 활성 트랙 페이드 인 -->
                                    <DoubleAnimation Storyboard.TargetName="ActiveTrack"
                                                     Storyboard.TargetProperty="Opacity"
                                                     To="1" Duration="0:0:0.2"/>
                                    
                                    <!-- ON 레이블 표시 -->
                                    <DoubleAnimation Storyboard.TargetName="OnLabel"
                                                     Storyboard.TargetProperty="Opacity"
                                                     To="1" Duration="0:0:0.2"/>
                                    
                                    <!-- OFF 레이블 숨기기 -->
                                    <DoubleAnimation Storyboard.TargetName="OffLabel"
                                                     Storyboard.TargetProperty="Opacity"
                                                     To="0" Duration="0:0:0.2"/>
                                    
                                    <!-- 체크 아이콘 표시 -->
                                    <DoubleAnimation Storyboard.TargetName="CheckIcon"
                                                     Storyboard.TargetProperty="Opacity"
                                                     To="1" Duration="0:0:0.3"/>
                                </Storyboard>
                            </BeginStoryboard>
                        </Trigger.EnterActions>
                        
                        <Trigger.ExitActions>
                            <BeginStoryboard>
                                <Storyboard>
                                    <!-- 핸들 원위치 -->
                                    <DoubleAnimation Storyboard.TargetName="Thumb"
                                                     Storyboard.TargetProperty="(UIElement.RenderTransform).(TranslateTransform.X)"
                                                     To="0" Duration="0:0:0.2"/>
                                    
                                    <!-- 활성 트랙 페이드 아웃 -->
                                    <DoubleAnimation Storyboard.TargetName="ActiveTrack"
                                                     Storyboard.TargetProperty="Opacity"
                                                     To="0" Duration="0:0:0.2"/>
                                    
                                    <!-- ON 레이블 숨기기 -->
                                    <DoubleAnimation Storyboard.TargetName="OnLabel"
                                                     Storyboard.TargetProperty="Opacity"
                                                     To="0" Duration="0:0:0.2"/>
                                    
                                    <!-- OFF 레이블 표시 -->
                                    <DoubleAnimation Storyboard.TargetName="OffLabel"
                                                     Storyboard.TargetProperty="Opacity"
                                                     To="1" Duration="0:0:0.2"/>
                                    
                                    <!-- 체크 아이콘 숨기기 -->
                                    <DoubleAnimation Storyboard.TargetName="CheckIcon"
                                                     Storyboard.TargetProperty="Opacity"
                                                     To="0" Duration="0:0:0.2"/>
                                </Storyboard>
                            </BeginStoryboard>
                        </Trigger.ExitActions>
                    </Trigger>
                    
                    <!-- 마우스 오버 상태 -->
                    <Trigger Property="IsMouseOver" Value="True">
                        <Setter TargetName="Thumb" Property="Effect">
                            <Setter.Value>
                                <DropShadowEffect ShadowDepth="2" 
                                                  BlurRadius="6" 
                                                  Opacity="0.4"/>
                            </Setter.Value>
                        </Setter>
                    </Trigger>
                    
                    <!-- 비활성화 상태 -->
                    <Trigger Property="IsEnabled" Value="False">
                        <Setter TargetName="RootGrid" Property="Opacity" Value="0.5"/>
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

이 토글 스위치 템플릿은 여러 가지 고급 기능을 보여줍니다:
1. **계층적 시각적 구조**: 여러 레이어를 겹쳐 풍부한 시각적 효과 구현
2. **부드러운 애니메이션**: 상태 전환 시 자연스러운 모션 효과
3. **상태 피드백**: ON/OFF 상태를 시각적으로 명확하게 표시
4. **상호작용 피드백**: 마우스 오버, 클릭 시 시각적 변화

## Visual State Manager를 활용한 상태 기반 디자인

Visual State Manager(VSM)는 복잡한 상태 기반 UI를 관리하는 데 매우 효과적인 도구입니다. 특히 여러 상태 간의 전환과 애니메이션을 관리할 때 유용합니다.

```xml
<!-- VSM을 사용한 카드 컴포넌트 템플릿 -->
<Style TargetType="ContentControl" x:Key="InteractiveCard">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="ContentControl">
                <!-- 카드의 기본 구조 -->
                <Border x:Name="CardBorder"
                        Background="{TemplateBinding Background}"
                        BorderBrush="{TemplateBinding BorderBrush}"
                        BorderThickness="{TemplateBinding BorderThickness}"
                        CornerRadius="12"
                        Padding="16">
                    
                    <!-- 컨텐츠 표시 -->
                    <ContentPresenter/>
                    
                    <!-- Visual State Manager 설정 -->
                    <VisualStateManager.VisualStateGroups>
                        <!-- 일반 상태 그룹 -->
                        <VisualStateGroup x:Name="CommonStates">
                            <!-- 기본 상태 -->
                            <VisualState x:Name="Normal">
                                <Storyboard>
                                    <!-- 기본 상태로의 애니메이션 -->
                                    <DoubleAnimation Storyboard.TargetName="CardBorder"
                                                     Storyboard.TargetProperty="RenderTransform.ScaleX"
                                                     To="1" Duration="0:0:0.15"/>
                                    <DoubleAnimation Storyboard.TargetName="CardBorder"
                                                     Storyboard.TargetProperty="RenderTransform.ScaleY"
                                                     To="1" Duration="0:0:0.15"/>
                                    <ColorAnimation Storyboard.TargetName="CardBorder"
                                                    Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                                    To="#FFFFFF" Duration="0:0:0.15"/>
                                </Storyboard>
                            </VisualState>
                            
                            <!-- 마우스 오버 상태 -->
                            <VisualState x:Name="MouseOver">
                                <Storyboard>
                                    <DoubleAnimation Storyboard.TargetName="CardBorder"
                                                     Storyboard.TargetProperty="RenderTransform.ScaleX"
                                                     To="1.02" Duration="0:0:0.1"/>
                                    <DoubleAnimation Storyboard.TargetName="CardBorder"
                                                     Storyboard.TargetProperty="RenderTransform.ScaleY"
                                                     To="1.02" Duration="0:0:0.1"/>
                                    <ColorAnimation Storyboard.TargetName="CardBorder"
                                                    Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                                    To="#F8FAFC" Duration="0:0:0.1"/>
                                </Storyboard>
                            </VisualState>
                            
                            <!-- 눌림 상태 -->
                            <VisualState x:Name="Pressed">
                                <Storyboard>
                                    <DoubleAnimation Storyboard.TargetName="CardBorder"
                                                     Storyboard.TargetProperty="RenderTransform.ScaleX"
                                                     To="0.98" Duration="0:0:0.05"/>
                                    <DoubleAnimation Storyboard.TargetName="CardBorder"
                                                     Storyboard.TargetProperty="RenderTransform.ScaleY"
                                                     To="0.98" Duration="0:0:0.05"/>
                                    <ColorAnimation Storyboard.TargetName="CardBorder"
                                                    Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                                    To="#F1F5F9" Duration="0:0:0.05"/>
                                </Storyboard>
                            </VisualState>
                            
                            <!-- 비활성화 상태 -->
                            <VisualState x:Name="Disabled">
                                <Storyboard>
                                    <DoubleAnimation Storyboard.TargetName="CardBorder"
                                                     Storyboard.TargetProperty="Opacity"
                                                     To="0.5" Duration="0:0:0.1"/>
                                </Storyboard>
                            </VisualState>
                        </VisualStateGroup>
                        
                        <!-- 선택 상태 그룹 -->
                        <VisualStateGroup x:Name="SelectionStates">
                            <!-- 선택되지 않음 -->
                            <VisualState x:Name="Unselected"/>
                            
                            <!-- 선택됨 -->
                            <VisualState x:Name="Selected">
                                <Storyboard>
                                    <ColorAnimation Storyboard.TargetName="CardBorder"
                                                    Storyboard.TargetProperty="(Border.BorderBrush).(SolidColorBrush.Color)"
                                                    To="#3B82F6" Duration="0:0:0.2"/>
                                    <DoubleAnimation Storyboard.TargetName="CardBorder"
                                                     Storyboard.TargetProperty="BorderThickness"
                                                     To="3" Duration="0:0:0.2"/>
                                    <ColorAnimation Storyboard.TargetName="CardBorder"
                                                    Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                                    To="#EFF6FF" Duration="0:0:0.2"/>
                                </Storyboard>
                            </VisualState>
                        </VisualStateGroup>
                        
                        <!-- 포커스 상태 그룹 -->
                        <VisualStateGroup x:Name="FocusStates">
                            <!-- 포커스 없음 -->
                            <VisualState x:Name="Unfocused"/>
                            
                            <!-- 포커스 됨 -->
                            <VisualState x:Name="Focused">
                                <Storyboard>
                                    <DoubleAnimation Storyboard.TargetName="CardBorder"
                                                     Storyboard.TargetProperty="(Border.Effect).(DropShadowEffect.BlurRadius)"
                                                     To="12" Duration="0:0:0.2"/>
                                    <DoubleAnimation Storyboard.TargetName="CardBorder"
                                                     Storyboard.TargetProperty="(Border.Effect).(DropShadowEffect.ShadowDepth)"
                                                     To="3" Duration="0:0:0.2"/>
                                </Storyboard>
                            </VisualState>
                        </VisualStateGroup>
                    </VisualStateManager.VisualStateGroups>
                </Border>
                
                <!-- 초기 렌더 트랜스폼 설정 -->
                <ControlTemplate.Triggers>
                    <Trigger Property="Template" Value="{x:Null}">
                        <Setter TargetName="CardBorder" Property="RenderTransform">
                            <Setter.Value>
                                <ScaleTransform ScaleX="1" ScaleY="1"/>
                            </Setter.Value>
                        </Setter>
                        <Setter TargetName="CardBorder" Property="Effect">
                            <Setter.Value>
                                <DropShadowEffect BlurRadius="8" 
                                                  ShadowDepth="1" 
                                                  Opacity="0.1"
                                                  Color="#000000"/>
                            </Setter.Value>
                        </Setter>
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
    
    <!-- 기본 속성 값 -->
    <Setter Property="Background" Value="#FFFFFF"/>
    <Setter Property="BorderBrush" Value="#E5E7EB"/>
    <Setter Property="BorderThickness" Value="1"/>
    <Setter Property="Width" Value="280"/>
    <Setter Property="Height" Value="180"/>
    <Setter Property="Cursor" Value="Hand"/>
</Style>
```

VSM을 사용하면 상태 관리가 훨씬 체계적이고 유지보수하기 쉬워집니다. 각 상태 그룹은 독립적으로 관리되며, 상태 전환 애니메이션도 일관된 방식으로 정의할 수 있습니다.

## 성능 최적화를 위한 고급 기법

대규모 애플리케이션에서 스타일과 템플릿의 성능은 매우 중요합니다. 다음은 성능을 최적화하기 위한 실용적인 기법들입니다:

### 1. 리소스 공유와 Freezable 최적화

```xml
<!-- 성능 최적화를 위한 리소스 정의 -->
<ResourceDictionary>
    <!-- 정적 리소스는 Freeze하여 성능 향상 -->
    <SolidColorBrush x:Key="OptimizedPrimaryBrush" Color="#2563EB" x:Shared="False">
        <SolidColorBrush.Resources>
            <Style TargetType="Brush">
                <Setter Property="Freezable.IsFrozen" Value="True"/>
            </Style>
        </SolidColorBrush.Resources>
    </SolidColorBrush>
    
    <!-- Geometry 리소스 캐싱 -->
    <Geometry x:Key="CheckIconGeometry" x:Shared="False">
        M9 16.17L4.83 12l-1.42 1.41L9 19 21 7l-1.41-1.41z
        <Geometry.Resources>
            <Style TargetType="Geometry">
                <Setter Property="Freezable.IsFrozen" Value="True"/>
            </Style>
        </Geometry.Resources>
    </Geometry>
    
    <!-- 반복 사용되는 LinearGradientBrush 캐싱 -->
    <LinearGradientBrush x:Key="OptimizedGradient" 
                         StartPoint="0,0" 
                         EndPoint="1,0"
                         x:Shared="False">
        <GradientStop Color="#2563EB" Offset="0"/>
        <GradientStop Color="#1D4ED8" Offset="1"/>
        <LinearGradientBrush.Resources>
            <Style TargetType="LinearGradientBrush">
                <Setter Property="Freezable.IsFrozen" Value="True"/>
            </Style>
        </LinearGradientBrush.Resources>
    </LinearGradientBrush>
</ResourceDictionary>
```

### 2. 템플릿 바인딩 최적화

```xml
<!-- 최적화된 컨트롤 템플릿 -->
<ControlTemplate TargetType="Button">
    <!-- TemplateBinding: 가장 빠른 바인딩 -->
    <Border Background="{TemplateBinding Background}"
            BorderBrush="{TemplateBinding BorderBrush}"
            BorderThickness="{TemplateBinding BorderThickness}"
            CornerRadius="{TemplateBinding Tag, Converter={StaticResource CornerRadiusConverter}}">
        
        <!-- ContentPresenter 최적화 -->
        <ContentPresenter x:Name="ContentHost"
                          HorizontalAlignment="{TemplateBinding HorizontalContentAlignment}"
                          VerticalAlignment="{TemplateBinding VerticalContentAlignment}"
                          Margin="{TemplateBinding Padding}"
                          RecognizesAccessKey="True"
                          SnapsToDevicePixels="{TemplateBinding SnapsToDevicePixels}"/>
        
        <!-- 최소한의 트리거만 사용 -->
        <ControlTemplate.Triggers>
            <!-- 복잡한 조건은 MultiDataTrigger 대신 단순 Trigger 사용 -->
            <Trigger Property="IsMouseOver" Value="True">
                <!-- 가능하면 Storyboard보다 Setter 사용 -->
                <Setter TargetName="ContentHost" Property="TextElement.Foreground" 
                        Value="{DynamicResource PrimaryBrush}"/>
            </Trigger>
            
            <!-- IsEnabled 대신 UIElement.IsEnabled 사용 (더 가벼움) -->
            <Trigger Property="UIElement.IsEnabled" Value="False">
                <Setter Property="Opacity" Value="0.6"/>
            </Trigger>
        </ControlTemplate.Triggers>
    </Border>
</ControlTemplate>
```

### 3. 가상화 지원 ItemsControl 템플릿

```xml
<!-- 가상화를 지원하는 ListBox 템플릿 -->
<Style TargetType="ListBox" x:Key="VirtualizedListBox">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="ListBox">
                <!-- 최소한의 요소로 구성 -->
                <Border x:Name="Bd"
                        BorderBrush="{TemplateBinding BorderBrush}"
                        BorderThickness="{TemplateBinding BorderThickness}"
                        Background="{TemplateBinding Background}"
                        SnapsToDevicePixels="True">
                    
                    <!-- 가상화를 유지하는 ScrollViewer -->
                    <ScrollViewer x:Name="ScrollViewer"
                                  Focusable="False"
                                  Padding="{TemplateBinding Padding}"
                                  HorizontalScrollBarVisibility="{TemplateBinding ScrollViewer.HorizontalScrollBarVisibility}"
                                  VerticalScrollBarVisibility="{TemplateBinding ScrollViewer.VerticalScrollBarVisibility}"
                                  CanContentScroll="{TemplateBinding ScrollViewer.CanContentScroll}">
                        
                        <!-- 가상화 패널 사용 -->
                        <ItemsPresenter x:Name="ItemsPresenter"
                                        SnapsToDevicePixels="{TemplateBinding SnapsToDevicePixels}"/>
                    </ScrollViewer>
                </Border>
                
                <!-- 최소한의 트리거 -->
                <ControlTemplate.Triggers>
                    <Trigger Property="IsEnabled" Value="False">
                        <Setter TargetName="Bd" Property="Background" 
                                Value="{DynamicResource {x:Static SystemColors.ControlBrushKey}}"/>
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
    
    <!-- 가상화 설정 -->
    <Setter Property="VirtualizingPanel.IsVirtualizing" Value="True"/>
    <Setter Property="VirtualizingPanel.VirtualizationMode" Value="Recycling"/>
    <Setter Property="ScrollViewer.CanContentScroll" Value="True"/>
    
    <!-- 성능 최적화 -->
    <Setter Property="UIElement.IsEnabled" Value="True"/>
    <Setter Property="TextElement.FontSize" Value="12"/>
</Style>
```

## 다크/라이트 테마 지원 시스템

프로페셔널한 애플리케이션은 사용자 선호도에 따른 테마 전환을 지원해야 합니다. 다음은 완전한 테마 시스템의 구현 예입니다:

```xml
<!-- ThemeManager.cs -->
public static class ThemeManager
{
    public enum Theme { Light, Dark, HighContrast }
    
    private static Theme _currentTheme = Theme.Light;
    
    public static Theme CurrentTheme
    {
        get => _currentTheme;
        set
        {
            if (_currentTheme != value)
            {
                _currentTheme = value;
                ApplyTheme(value);
                ThemeChanged?.Invoke(null, EventArgs.Empty);
            }
        }
    }
    
    public static event EventHandler ThemeChanged;
    
    private static void ApplyTheme(Theme theme)
    {
        var app = Application.Current;
        if (app == null) return;
        
        // 기존 테마 리소스 제거
        var mergedDictionaries = app.Resources.MergedDictionaries;
        var existingTheme = mergedDictionaries
            .FirstOrDefault(d => d.Source?.OriginalString?.Contains("Theme.") == true);
        
        if (existingTheme != null)
            mergedDictionaries.Remove(existingTheme);
        
        // 새 테마 리소스 추가
        var themeUri = theme switch
        {
            Theme.Dark => new Uri("Themes/Theme.Dark.xaml", UriKind.Relative),
            Theme.HighContrast => new Uri("Themes/Theme.HighContrast.xaml", UriKind.Relative),
            _ => new Uri("Themes/Theme.Light.xaml", UriKind.Relative)
        };
        
        var themeDictionary = new ResourceDictionary { Source = themeUri };
        mergedDictionaries.Add(themeDictionary);
        
        // 시스템 테마 감지 (선택적)
        if (theme == Theme.Light)
        {
            DetectSystemTheme();
        }
    }
    
    private static void DetectSystemTheme()
    {
        // Windows 시스템 테마 감지
        try
        {
            var key = Microsoft.Win32.Registry.CurrentUser.OpenSubKey(
                @"Software\Microsoft\Windows\CurrentVersion\Themes\Personalize");
            
            if (key?.GetValue("AppsUseLightTheme") is int useLightTheme)
            {
                if (useLightTheme == 0)
                {
                    CurrentTheme = Theme.Dark;
                }
            }
        }
        catch
        {
            // 레지스트리 접근 실패 시 기본값 유지
        }
    }
    
    public static void Initialize()
    {
        // 초기 테마 적용
        ApplyTheme(CurrentTheme);
        
        // 시스템 테마 변경 감지
        SystemEvents.UserPreferenceChanged += (s, e) =>
        {
            if (e.Category == UserPreferenceCategory.General)
            {
                DetectSystemTheme();
            }
        };
    }
}

<!-- Theme.Light.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    
    <!-- 라이트 테마 색상 팔레트 -->
    <Color x:Key="PrimaryColor">#2563EB</Color>
    <Color x:Key="SecondaryColor">#64748B</Color>
    <Color x:Key="BackgroundColor">#FFFFFF</Color>
    <Color x:Key="SurfaceColor">#F9FAFB</Color>
    <Color x:Key="TextPrimaryColor">#1F2937</Color>
    <Color x:Key="TextSecondaryColor">#6B7280</Color>
    <Color x:Key="BorderColor">#E5E7EB</Color>
    <Color x:Key="SuccessColor">#10B981</Color>
    <Color x:Key="WarningColor">#F59E0B</Color>
    <Color x:Key="ErrorColor">#EF4444</Color>
    
    <!-- 동적 브러시 리소스 -->
    <SolidColorBrush x:Key="PrimaryBrush" Color="{DynamicResource PrimaryColor}"/>
    <SolidColorBrush x:Key="SecondaryBrush" Color="{DynamicResource SecondaryColor}"/>
    <SolidColorBrush x:Key="BackgroundBrush" Color="{DynamicResource BackgroundColor}"/>
    <SolidColorBrush x:Key="SurfaceBrush" Color="{DynamicResource SurfaceColor}"/>
    <SolidColorBrush x:Key="TextPrimaryBrush" Color="{DynamicResource TextPrimaryColor}"/>
    <SolidColorBrush x:Key="TextSecondaryBrush" Color="{DynamicResource TextSecondaryColor}"/>
    <SolidColorBrush x:Key="BorderBrush" Color="{DynamicResource BorderColor}"/>
    <SolidColorBrush x:Key="SuccessBrush" Color="{DynamicResource SuccessColor}"/>
    <SolidColorBrush x:Key="WarningBrush" Color="{DynamicResource WarningColor}"/>
    <SolidColorBrush x:Key="ErrorBrush" Color="{DynamicResource ErrorColor}"/>
    
    <!-- 그림자 효과 -->
    <Style x:Key="CardShadow" TargetType="Border">
        <Setter Property="Effect">
            <Setter.Value>
                <DropShadowEffect BlurRadius="12" 
                                  ShadowDepth="2" 
                                  Opacity="0.08"
                                  Color="#000000"/>
            </Setter.Value>
        </Setter>
    </Style>
    
</ResourceDictionary>

<!-- Theme.Dark.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    
    <!-- 다크 테마 색상 팔레트 -->
    <Color x:Key="PrimaryColor">#3B82F6</Color>
    <Color x:Key="SecondaryColor">#94A3B8</Color>
    <Color x:Key="BackgroundColor">#0F172A</Color>
    <Color x:Key="SurfaceColor">#1E293B</Color>
    <Color x:Key="TextPrimaryColor">#F1F5F9</Color>
    <Color x:Key="TextSecondaryColor">#CBD5E1</Color>
    <Color x:Key="BorderColor">#334155</Color>
    <Color x:Key="SuccessColor">#34D399</Color>
    <Color x:Key="WarningColor">#FBBF24</Color>
    <Color x:Key="ErrorColor">#F87171</Color>
    
    <!-- 동일 키로 브러시 재정의 -->
    <SolidColorBrush x:Key="PrimaryBrush" Color="{DynamicResource PrimaryColor}"/>
    <SolidColorBrush x:Key="SecondaryBrush" Color="{DynamicResource SecondaryColor}"/>
    <SolidColorBrush x:Key="BackgroundBrush" Color="{DynamicResource BackgroundColor}"/>
    <SolidColorBrush x:Key="SurfaceBrush" Color="{DynamicResource SurfaceColor}"/>
    <SolidColorBrush x:Key="TextPrimaryBrush" Color="{DynamicResource TextPrimaryColor}"/>
    <SolidColorBrush x:Key="TextSecondaryBrush" Color="{DynamicResource TextSecondaryColor}"/>
    <SolidColorBrush x:Key="BorderBrush" Color="{DynamicResource BorderColor}"/>
    <SolidColorBrush x:Key="SuccessBrush" Color="{DynamicResource SuccessColor}"/>
    <SolidColorBrush x:Key="WarningBrush" Color="{DynamicResource WarningColor}"/>
    <SolidColorBrush x:Key="ErrorBrush" Color="{DynamicResource ErrorColor}"/>
    
    <!-- 다크 테마에 맞는 그림자 효과 -->
    <Style x:Key="CardShadow" TargetType="Border">
        <Setter Property="Effect">
            <Setter.Value>
                <DropShadowEffect BlurRadius="16" 
                                  ShadowDepth="3" 
                                  Opacity="0.15"
                                  Color="#000000"/>
            </Setter.Value>
        </Setter>
    </Style>
    
</ResourceDictionary>

<!-- App.xaml.cs -->
public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);
        
        // 테마 관리자 초기화
        ThemeManager.Initialize();
        
        // 기본 테마 설정 (사용자 설정에서 불러올 수 있음)
        ThemeManager.CurrentTheme = Properties.Settings.Default.Theme;
        
        // 테마 변경 시 설정 저장
        ThemeManager.ThemeChanged += (s, args) =>
        {
            Properties.Settings.Default.Theme = ThemeManager.CurrentTheme;
            Properties.Settings.Default.Save();
        };
    }
}

<!-- 테마 전환 메뉴 예시 -->
<Menu>
    <MenuItem Header="테마">
        <MenuItem Header="라이트 테마" 
                  IsChecked="{Binding CurrentTheme, 
                           Converter={StaticResource ThemeToBoolConverter}, 
                           ConverterParameter=Light}"
                  Command="{x:Static local:ThemeCommands.ChangeToLightThemeCommand}"/>
        
        <MenuItem Header="다크 테마" 
                  IsChecked="{Binding CurrentTheme, 
                           Converter={StaticResource ThemeToBoolConverter}, 
                           ConverterParameter=Dark}"
                  Command="{x:Static local:ThemeCommands.ChangeToDarkThemeCommand}"/>
        
        <Separator/>
        
        <MenuItem Header="시스템 테마 따르기"
                  IsCheckable="True"
                  IsChecked="{Binding FollowSystemTheme, Mode=TwoWay}"/>
    </MenuItem>
</Menu>
```

## 실제 프로젝트 통합: 디자인 시스템 컴포넌트 라이브러리

실제 프로젝트에서는 이러한 스타일과 템플릿을 체계적으로 조직화하여 재사용 가능한 컴포넌트 라이브러리를 구축합니다:

```
DesignSystem/
├── Assets/
│   ├── Icons.xaml
│   └── Fonts.xaml
├── Colors/
│   ├── Light.xaml
│   ├── Dark.xaml
│   └── HighContrast.xaml
├── Components/
│   ├── Buttons.xaml
│   ├── Inputs.xaml
│   ├── Cards.xaml
│   ├── Navigation.xaml
│   └── DataGrid.xaml
├── Themes/
│   └── Generic.xaml
└── DesignSystem.xaml
```

```xml
<!-- DesignSystem.xaml - 메인 리소스 딕셔너리 -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    
    <ResourceDictionary.MergedDictionaries>
        <!-- 자산 -->
        <ResourceDictionary Source="Assets/Icons.xaml"/>
        <ResourceDictionary Source="Assets/Fonts.xaml"/>
        
        <!-- 현재 테마 색상 -->
        <ResourceDictionary Source="Colors/Light.xaml"/>
        
        <!-- 컴포넌트 스타일 -->
        <ResourceDictionary Source="Components/Buttons.xaml"/>
        <ResourceDictionary Source="Components/Inputs.xaml"/>
        <ResourceDictionary Source="Components/Cards.xaml"/>
        <ResourceDictionary Source="Components/Navigation.xaml"/>
        <ResourceDictionary Source="Components/DataGrid.xaml"/>
        
        <!-- 테마 오버라이드 -->
        <ResourceDictionary Source="Themes/Generic.xaml"/>
    </ResourceDictionary.MergedDictionaries>
    
    <!-- 디자인 시스템 토큰 -->
    <system:Double x:Key="BorderRadius.Small">4</system:Double>
    <system:Double x:Key="BorderRadius.Medium">8</system:Double>
    <system:Double x:Key="BorderRadius.Large">12</system:Double>
    <system:Double x:Key="BorderRadius.XLarge">16</system:Double>
    
    <system:Double x:Key="Spacing.XSmall">4</system:Double>
    <system:Double x:Key="Spacing.Small">8</system:Double>
    <system:Double x:Key="Spacing.Medium">16</system:Double>
    <system:Double x:Key="Spacing.Large">24</system:Double>
    <system:Double x:Key="Spacing.XLarge">32</system:Double>
    
    <system:Double x:Key="Elevation.Small">2</system:Double>
    <system:Double x:Key="Elevation.Medium">8</system:Double>
    <system:Double x:Key="Elevation.Large">16</system:Double>
    
    <!-- 타이포그래피 스케일 -->
    <system:Double x:Key="Typography.XSmall">12</system:Double>
    <system:Double x:Key="Typography.Small">14</system:Double>
    <system:Double x:Key="Typography.Medium">16</system:Double>
    <system:Double x:Key="Typography.Large">20</system:Double>
    <system:Double x:Key="Typography.XLarge">24</system:Double>
    <system:Double x:Key="Typography.XXLarge">32</system:Double>
    
</ResourceDictionary>
```

## 결론: 전문가 수준의 스타일링 철학

WPF의 스타일과 컨트롤 템플릿 시스템은 단순한 미적 장식을 넘어, 애플리케이션 아키텍처의 핵심 요소입니다. 이 시스템을 효과적으로 활용하기 위한 몇 가지 핵심 원칙을 정리해 보겠습니다:

**첫째, 추상화의 적절한 수준을 유지하세요.** 스타일은 속성 값의 재사용을, 템플릿은 시각적 구조의 재사용을 담당합니다. 이 두 가지를 명확히 구분하고 적절히 조합해야 합니다. 너무 많은 로직을 스타일에 넣거나, 너무 제한적인 템플릿을 만들지 않도록 주의해야 합니다.

**둘째, 일관성과 유연성의 균형을 찾으세요.** 암시적 스타일은 일관성을 보장하지만, 지나치게 엄격하면 사용자 정의가 어려워집니다. BasedOn 상속 체계를 통해 기본적인 일관성을 유지하면서도 특정 상황에 맞는 변형을 쉽게 만들 수 있어야 합니다.

**셋째, 성능을 고려한 설계를 하세요.** TemplateBinding을 우선 사용하고, Freezable 리소스를 적극적으로 활용하며, 불필요한 트리거와 애니메이션을 피해야 합니다. 대규모 데이터를 다룰 때는 가상화를 고려한 템플릿 설계가 필수적입니다.

**넷째, 테마와 접근성을 고려하세요.** 현대적인 애플리케이션은 다양한 사용자 환경을 지원해야 합니다. 다크 모드, 고대비 모드, 다양한 디스플레이 크기 등을 고려한 유연한 디자인 시스템을 구축해야 합니다. DynamicResource와 리소스 딕셔너리 병합을 효과적으로 활용하면 런타임 테마 전환도 쉽게 구현할 수 있습니다.

**다섯째, 컴포넌트 기반 사고를 하세요.** 재사용 가능한 스타일과 템플릿을 컴포넌트로 패키징하면, 여러 프로젝트에서 일관된 디자인 시스템을 공유할 수 있습니다. 이는 유지보수성을 크게 향상시키고, 디자인과 개발 간의 협업을 원활하게 합니다.

WPF의 스타일링 시스템은 배우기에는 다소 복잡할 수 있지만, 일단 숙달하면 강력한 디자인 시스템을 구축할 수 있는 탁월한 도구가 됩니다. 이 시스템을 효과적으로 활용하면 단순히 보기 좋은 UI를 넘어, 확장성 있고 유지보수하기 쉬우며 접근성 높은 전문적인 애플리케이션을 만들 수 있습니다.

스타일과 템플릿은 단순한 기술이 아니라, 사용자 경험을 설계하는 도구입니다. 이러한 도구를 효과적으로 활용하는 것은 기술적 능력뿐만 아니라 디자인적 통찰력도 요구합니다. 올바르게 구현된 스타일링 시스템은 애플리케이션의 품질을 결정하는 중요한 요소가 될 것입니다.