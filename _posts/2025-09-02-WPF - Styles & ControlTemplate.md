---
layout: post
title: WPF - Styles & ControlTemplate
date: 2025-09-02 23:25:23 +0900
category: WPF
---
# WPF 스타일과 컨트롤 템플릿

WPF에서 스타일(Style)과 컨트롤 템플릿(ControlTemplate)은 UI의 재사용성과 일관성을 높이는 핵심 도구입니다. 이 글에서는 초중급 개발자를 대상으로 두 개념의 차이부터 실무에서 자주 사용하는 패턴까지 핵심만 정리합니다.

---

## 스타일(Style)의 기본

### 스타일이란?

스타일은 여러 컨트롤에 동일한 속성값을 일괄 적용할 수 있는 집합입니다. CSS와 비슷하지만, WPF에서는 속성 외에도 트리거, 애니메이션 등을 포함할 수 있습니다.

```xml
<!-- 버튼에 공통 스타일 적용 -->
<Style TargetType="Button" x:Key="PrimaryButton">
    <Setter Property="Background" Value="#2563EB"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="FontSize" Value="14"/>
    <Setter Property="Padding" Value="12,8"/>
    <Setter Property="BorderThickness" Value="0"/>
</Style>

<!-- 사용 예시 -->
<Button Style="{StaticResource PrimaryButton}" Content="클릭"/>
```

### 암시적 스타일

`x:Key` 없이 `TargetType`만 지정하면 해당 타입의 모든 컨트롤에 자동으로 적용됩니다.

```xml
<Window.Resources>
    <!-- 이 윈도우 안의 모든 버튼에 적용 -->
    <Style TargetType="Button">
        <Setter Property="Background" Value="#E5E7EB"/>
        <Setter Property="Margin" Value="4"/>
        <Setter Property="Padding" Value="8,4"/>
    </Style>
</Window.Resources>
```

### BasedOn을 이용한 스타일 상속

기본 스타일을 정의하고 이를 확장하여 다양한 변형을 만들 수 있습니다.

```xml
<Style x:Key="BaseButton" TargetType="Button">
    <Setter Property="FontSize" Value="14"/>
    <Setter Property="Padding" Value="8,4"/>
    <Setter Property="Margin" Value="4"/>
</Style>

<Style x:Key="SuccessButton" TargetType="Button" BasedOn="{StaticResource BaseButton}">
    <Setter Property="Background" Value="#10B981"/>
    <Setter Property="Foreground" Value="White"/>
</Style>

<Style x:Key="DangerButton" TargetType="Button" BasedOn="{StaticResource BaseButton}">
    <Setter Property="Background" Value="#EF4444"/>
    <Setter Property="Foreground" Value="White"/>
</Style>
```

---

## 컨트롤 템플릿(ControlTemplate)의 이해

### 템플릿이 필요한 이유

스타일은 기존 컨트롤의 속성만 변경할 수 있습니다. 컨트롤의 전체 구조를 바꾸려면 **ControlTemplate**을 사용해야 합니다. 예를 들어, 버튼을 동그란 원형으로 만들거나 토글 스위치처럼 완전히 다른 모양으로 재정의할 수 있습니다.

```xml
<!-- 원형 버튼 템플릿 -->
<Style TargetType="Button" x:Key="CircleButton">
    <Setter Property="Width" Value="48"/>
    <Setter Property="Height" Value="48"/>
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <Border x:Name="border"
                        Background="{TemplateBinding Background}"
                        CornerRadius="24"
                        BorderThickness="0">
                    <ContentPresenter HorizontalAlignment="Center"
                                      VerticalAlignment="Center"/>
                </Border>
                <ControlTemplate.Triggers>
                    <Trigger Property="IsMouseOver" Value="True">
                        <Setter TargetName="border" Property="Background" Value="#3B82F6"/>
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
    <Setter Property="Background" Value="#2563EB"/>
    <Setter Property="Foreground" Value="White"/>
</Style>
```

### TemplateBinding으로 속성 연결

템플릿 내부에서 컨트롤의 원래 속성을 참조할 때는 `TemplateBinding`을 사용합니다. 이렇게 하면 스타일에서 설정한 Background, Padding 등이 템플릿에 전달됩니다.

```xml
<ControlTemplate TargetType="Button">
    <Border Background="{TemplateBinding Background}"
            BorderBrush="{TemplateBinding BorderBrush}"
            BorderThickness="{TemplateBinding BorderThickness}"
            CornerRadius="8">
        <ContentPresenter Margin="{TemplateBinding Padding}"/>
    </Border>
</ControlTemplate>
```

### 템플릿과 스타일의 분리

템플릿을 따로 정의하고 스타일에서 참조하면 재사용성이 높아집니다.

```xml
<!-- 템플릿 정의 -->
<ControlTemplate x:Key="RoundButtonTemplate" TargetType="Button">
    <Border Background="{TemplateBinding Background}"
            CornerRadius="20"
            Padding="12,8">
        <ContentPresenter/>
    </Border>
</ControlTemplate>

<!-- 스타일에서 템플릿 지정 -->
<Style TargetType="Button" x:Key="RoundButton">
    <Setter Property="Template" Value="{StaticResource RoundButtonTemplate}"/>
    <Setter Property="Background" Value="#2563EB"/>
    <Setter Property="Foreground" Value="White"/>
</Style>
```

---

## 트리거를 이용한 상태 변화

### 속성 트리거

컨트롤의 속성(IsMouseOver, IsPressed 등)에 따라 스타일을 동적으로 변경합니다.

```xml
<Style TargetType="Button">
    <Setter Property="Background" Value="#E5E7EB"/>
    <Style.Triggers>
        <Trigger Property="IsMouseOver" Value="True">
            <Setter Property="Background" Value="#3B82F6"/>
            <Setter Property="Foreground" Value="White"/>
        </Trigger>
        <Trigger Property="IsPressed" Value="True">
            <Setter Property="Background" Value="#1E40AF"/>
        </Trigger>
        <Trigger Property="IsEnabled" Value="False">
            <Setter Property="Opacity" Value="0.5"/>
        </Trigger>
    </Style.Triggers>
</Style>
```

### 데이터 트리거

MVVM 패턴에서 ViewModel의 속성에 따라 UI 모양을 바꿀 때 유용합니다.

```xml
<Style TargetType="Button" x:Key="SaveButton">
    <Setter Property="Background" Value="#10B981"/>
    <Style.Triggers>
        <DataTrigger Binding="{Binding IsProcessing}" Value="True">
            <Setter Property="IsEnabled" Value="False"/>
            <Setter Property="Content" Value="처리 중..."/>
        </DataTrigger>
        <DataTrigger Binding="{Binding IsSaved}" Value="True">
            <Setter Property="Background" Value="#6B7280"/>
            <Setter Property="Content" Value="저장됨"/>
        </DataTrigger>
    </Style.Triggers>
</Style>
```

### 멀티 트리거

여러 조건을 동시에 만족할 때 스타일을 적용합니다.

```xml
<MultiTrigger>
    <MultiTrigger.Conditions>
        <Condition Property="IsMouseOver" Value="True"/>
        <Condition Property="IsEnabled" Value="True"/>
    </MultiTrigger.Conditions>
    <Setter Property="Background" Value="#2563EB"/>
</MultiTrigger>
```

---

## 리소스 관리와 테마 전환

### 리소스 사전 분리

프로젝트가 커지면 리소스를 파일로 분리하는 것이 좋습니다.

```
Themes/
├── LightTheme.xaml
├── DarkTheme.xaml
└── Controls/
    ├── ButtonStyles.xaml
    └── TextBoxStyles.xaml
```

### 동적 테마 전환

`DynamicResource`를 사용하면 런타임에 리소스를 교체할 수 있습니다.

```xml
<!-- App.xaml -->
<Application.Resources>
    <ResourceDictionary>
        <ResourceDictionary.MergedDictionaries>
            <ResourceDictionary Source="Themes/LightTheme.xaml"/>
        </ResourceDictionary.MergedDictionaries>
    </ResourceDictionary>
</Application.Resources>
```

```csharp
// 코드에서 테마 변경
var newTheme = new ResourceDictionary { Source = new Uri("Themes/DarkTheme.xaml", UriKind.Relative) };
Application.Current.Resources.MergedDictionaries[0] = newTheme;
```

---

## 성능 고려사항

### Freezable 리소스 사용

자주 사용하는 브러시, 기하 도형 등은 `Freezable`을 활용하여 성능을 향상시킬 수 있습니다.

```xml
<SolidColorBrush x:Key="PrimaryBrush" Color="#2563EB" 
                 PresentationOptions:Freeze="True"/>
```

### 템플릿 최소화

복잡한 템플릿은 성능에 영향을 줍니다. 필요한 요소만 포함하고, 불필요한 중첩은 피하세요.

```xml
<!-- 불필요하게 복잡한 템플릿 -->
<ControlTemplate>
    <Grid>
        <Border>
            <StackPanel>
                <ContentPresenter/>
            </StackPanel>
        </Border>
    </Grid>
</ControlTemplate>

<!-- 간결하게 -->
<ControlTemplate>
    <ContentPresenter/>
</ControlTemplate>
```

### 가상화와 함께 사용

`ItemsControl`의 템플릿을 만들 때 가상화를 고려해야 합니다.

```xml
<ListBox VirtualizingStackPanel.IsVirtualizing="True"
         VirtualizingStackPanel.VirtualizationMode="Recycling">
    <ListBox.ItemTemplate>
        <DataTemplate>
            <!-- 템플릿 내용 -->
        </DataTemplate>
    </ListBox.ItemTemplate>
</ListBox>
```

---

## 실제 패턴: 버튼 스타일 계층 구조

다음은 실무에서 자주 사용하는 버튼 스타일 계층 구조의 예입니다.

```xml
<ResourceDictionary>
    <!-- 기본 버튼 스타일 (암시적) -->
    <Style TargetType="Button">
        <Setter Property="FontSize" Value="14"/>
        <Setter Property="Padding" Value="12,8"/>
        <Setter Property="Margin" Value="4"/>
        <Setter Property="Cursor" Value="Hand"/>
        <Setter Property="Background" Value="#F3F4F6"/>
        <Setter Property="Foreground" Value="#1F2937"/>
        <Setter Property="BorderBrush" Value="#E5E7EB"/>
        <Setter Property="BorderThickness" Value="1"/>
        
        <Style.Triggers>
            <Trigger Property="IsMouseOver" Value="True">
                <Setter Property="Background" Value="#E5E7EB"/>
            </Trigger>
            <Trigger Property="IsPressed" Value="True">
                <Setter Property="Background" Value="#D1D5DB"/>
            </Trigger>
            <Trigger Property="IsEnabled" Value="False">
                <Setter Property="Opacity" Value="0.5"/>
            </Trigger>
        </Style.Triggers>
    </Style>
    
    <!-- 주요 버튼 -->
    <Style x:Key="PrimaryButton" TargetType="Button" BasedOn="{StaticResource {x:Type Button}}">
        <Setter Property="Background" Value="#2563EB"/>
        <Setter Property="Foreground" Value="White"/>
        <Setter Property="BorderBrush" Value="#2563EB"/>
        <Style.Triggers>
            <Trigger Property="IsMouseOver" Value="True">
                <Setter Property="Background" Value="#1D4ED8"/>
            </Trigger>
        </Style.Triggers>
    </Style>
    
    <!-- 위험 버튼 -->
    <Style x:Key="DangerButton" TargetType="Button" BasedOn="{StaticResource {x:Type Button}}">
        <Setter Property="Background" Value="#EF4444"/>
        <Setter Property="Foreground" Value="White"/>
        <Setter Property="BorderBrush" Value="#EF4444"/>
    </Style>
</ResourceDictionary>
```

---

## 요약

| 개념 | 역할 | 예시 |
|------|------|------|
| **Style** | 속성 값의 집합, 재사용 | 버튼의 배경색, 글꼴, 여백 통일 |
| **ControlTemplate** | 컨트롤의 시각적 구조 재정의 | 버튼을 원형이나 토글 스위치로 변경 |
| **Trigger** | 상태 변화에 따른 스타일 변경 | 마우스 오버 시 색상 변화 |
| **DataTrigger** | ViewModel 속성에 따른 변경 | 처리 중일 때 버튼 비활성화 |
| **ResourceDictionary** | 리소스의 물리적 분리 | 테마별 리소스 파일 분리 |

WPF의 스타일과 템플릿을 잘 활용하면 UI 코드의 중복을 크게 줄이고 유지보수성이 높은 애플리케이션을 만들 수 있습니다. 처음에는 간단한 스타일부터 적용해보고, 점차 템플릿을 활용한 고급 디자인으로 확장해 나가기를 권장합니다.