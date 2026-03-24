---
layout: post
title: Avalonia - Avalonia 스타일 재사용
date: 2025-04-07 22:20:23 +0900
category: Avalonia
---
# Avalonia 스타일 재사용 (Themes, Styles)

Avalonia에서는 스타일을 체계적으로 관리하면 애플리케이션의 디자인 일관성, 유지보수성, 확장성을 크게 향상시킬 수 있습니다. 이 글에서는 **디자인 토큰**을 활용한 일관성 확보, **리소스 사전 분리와 병합**, **테마 전환**(Light/Dark/HighContrast), **재사용 가능한 스타일 구성**, **커스텀 컨트롤 템플릿**까지 초중급 개발자 수준에서 설명합니다.

---

## 1. 디자인 토큰(Design Tokens)으로 일관성 확보

대규모 앱에서는 색상, 타이포그래피, 간격, 그림자 등을 **의미 있는 이름**으로 추상화해야 합니다. 이를 디자인 토큰(Design Tokens)이라고 합니다. 토큰을 변경하면 전체 앱의 모양이 일관되게 갱신됩니다.

### 프로젝트 구조 예시

```
Styles/
├── Tokens/
│   ├── Colors.xaml          # 색상 팔레트, 역할 기반 브러시
│   ├── Typography.xaml      # 폰트, 크기
│   ├── Spacing.xaml         # 간격, 둥근 모서리
│   └── Elevation.xaml       # 그림자, 테두리 두께
├── Controls/
│   ├── Buttons.xaml         # 버튼 변형 (primary, danger)
│   ├── Inputs.xaml          # 텍스트 박스, 콤보박스 등
│   └── Lists.xaml           # 리스트 박스, 데이터 그리드
└── Themes/
    ├── Light.xaml           # 라이트 테마 전용 브러시
    └── Dark.xaml            # 다크 테마 전용 브러시
```

### Colors.xaml (역할 기반)

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <!-- 원색(Primitive) -->
  <Color x:Key="Color.Brand.500">#2D7FF9</Color>
  <Color x:Key="Color.Brand.600">#1E6AE6</Color>
  <Color x:Key="Color.Neutral.000">#FFFFFF</Color>
  <Color x:Key="Color.Neutral.900">#121212</Color>
  <Color x:Key="Color.Success.500">#2EB872</Color>
  <Color x:Key="Color.Danger.500">#E5534B</Color>

  <!-- 역할 기반 브러시 (DynamicResource로 원색 참조) -->
  <SolidColorBrush x:Key="Brush.Surface" Color="{DynamicResource Color.Neutral.000}"/>
  <SolidColorBrush x:Key="Brush.Text.Primary" Color="#1E1E1E"/>
  <SolidColorBrush x:Key="Brush.Text.Inverse" Color="#FFFFFF"/>
  <SolidColorBrush x:Key="Brush.Primary" Color="{DynamicResource Color.Brand.500}"/>
  <SolidColorBrush x:Key="Brush.Primary.Active" Color="{DynamicResource Color.Brand.600}"/>
  <SolidColorBrush x:Key="Brush.Success" Color="{DynamicResource Color.Success.500}"/>
  <SolidColorBrush x:Key="Brush.Danger" Color="{DynamicResource Color.Danger.500}"/>
</ResourceDictionary>
```

### Typography.xaml

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <FontFamily x:Key="Font.Primary">Segoe UI, Noto Sans CJK KR, Arial</FontFamily>

  <x:Double x:Key="FontSize.Display">28</x:Double>
  <x:Double x:Key="FontSize.Headline">22</x:Double>
  <x:Double x:Key="FontSize.Body">14</x:Double>
  <x:Double x:Key="FontSize.Caption">12</x:Double>
</ResourceDictionary>
```

### Spacing.xaml

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <Thickness x:Key="Space.0">0</Thickness>
  <Thickness x:Key="Space.1">4</Thickness>
  <Thickness x:Key="Space.2">8</Thickness>
  <Thickness x:Key="Space.3">12</Thickness>
  <Thickness x:Key="Space.4">16</Thickness>
  <CornerRadius x:Key="Radius.S">4</CornerRadius>
  <CornerRadius x:Key="Radius.M">8</CornerRadius>
  <CornerRadius x:Key="Radius.L">16</CornerRadius>
</ResourceDictionary>
```

---

## 2. 리소스 사전 병합과 스타일 포함

`App.axaml`에서 `StyleInclude`를 사용하여 위에서 만든 파일들을 가져옵니다. `FluentTheme`는 기본 컨트롤 모양을 제공합니다.

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="YourApp.App">
  <Application.Styles>
    <FluentTheme Mode="Light" />

    <!-- 디자인 토큰 -->
    <StyleInclude Source="avares://YourApp/Styles/Tokens/Colors.xaml" />
    <StyleInclude Source="avares://YourApp/Styles/Tokens/Typography.xaml" />
    <StyleInclude Source="avares://YourApp/Styles/Tokens/Spacing.xaml" />

    <!-- 컴포넌트 스타일 -->
    <StyleInclude Source="avares://YourApp/Styles/Controls/Buttons.xaml" />
    <StyleInclude Source="avares://YourApp/Styles/Controls/Inputs.xaml" />
  </Application.Styles>
</Application>
```

> `StyleInclude`는 다른 어셈블리(모듈, NuGet 패키지)에 있는 스타일도 가져올 수 있습니다.

---

## 3. 기본 스타일 선언: 선택자와 클래스

### Buttons.xaml – 클래스 기반 변형

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <!-- 기본 버튼 스타일 (모든 Button에 적용) -->
  <Style Selector="Button">
    <Setter Property="FontFamily" Value="{DynamicResource Font.Primary}"/>
    <Setter Property="Padding" Value="{DynamicResource Space.2}"/>
    <Setter Property="CornerRadius" Value="{DynamicResource Radius.S}"/>
  </Style>

  <!-- primary 변형 -->
  <Style Selector="Button.primary">
    <Setter Property="Background" Value="{DynamicResource Brush.Primary}"/>
    <Setter Property="Foreground" Value="{DynamicResource Brush.Text.Inverse}"/>
  </Style>

  <!-- danger 변형 -->
  <Style Selector="Button.danger">
    <Setter Property="Background" Value="{DynamicResource Brush.Danger}"/>
    <Setter Property="Foreground" Value="{DynamicResource Brush.Text.Inverse}"/>
  </Style>

  <!-- 가상 선택자: 호버, 비활성화 -->
  <Style Selector="Button:pointerover">
    <Setter Property="Opacity" Value="0.9"/>
  </Style>
  <Style Selector="Button:disabled">
    <Setter Property="Opacity" Value="0.5"/>
  </Style>
</ResourceDictionary>
```

사용 예:

```xml
<Button Content="확인" Classes="primary"/>
<Button Content="삭제" Classes="danger"/>
```

---

## 4. 스타일 상속(BasedOn)과 키 스타일

공통 속성을 기본 스타일로 정의하고, 이를 상속받아 변형을 만들 수 있습니다.

```xml
<!-- 기본 스타일 (키 있음) -->
<Style x:Key="BaseButtonStyle" Selector="Button">
  <Setter Property="FontFamily" Value="{DynamicResource Font.Primary}"/>
  <Setter Property="Padding" Value="{DynamicResource Space.2}"/>
  <Setter Property="CornerRadius" Value="{DynamicResource Radius.S}"/>
</Style>

<!-- 상속하여 확장 -->
<Style Selector="Button.primary" BasedOn="{StaticResource BaseButtonStyle}">
  <Setter Property="Background" Value="{DynamicResource Brush.Primary}"/>
</Style>
```

키 스타일은 특정 컨트롤에만 명시적으로 적용할 때 유용합니다.

```xml
<Button Styles="{StaticResource BaseButtonStyle}" Content="명시 적용"/>
```

---

## 5. 테마 전환 (Light / Dark / HighContrast)

Avalonia 11+는 `ThemeVariant`를 지원합니다. 런타임에 테마를 바꾸려면 `Application.RequestedThemeVariant`를 설정하면 됩니다.

### 테마 리소스 파일 (Light.xaml / Dark.xaml)

테마 파일에서는 주로 브러시를 재정의합니다.

```xml
<!-- Light.xaml -->
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <SolidColorBrush x:Key="Brush.Surface" Color="#FFFFFF"/>
  <SolidColorBrush x:Key="Brush.Text.Primary" Color="#1E1E1E"/>
</ResourceDictionary>

<!-- Dark.xaml -->
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <SolidColorBrush x:Key="Brush.Surface" Color="#121212"/>
  <SolidColorBrush x:Key="Brush.Text.Primary" Color="#EEEEEE"/>
</ResourceDictionary>
```

### ThemeService (C#)

```csharp
public interface IThemeService
{
    void SetTheme(ThemeVariant variant);
    ThemeVariant Current { get; }
}

public sealed class ThemeService : IThemeService
{
    public ThemeVariant Current { get; private set; } = ThemeVariant.Light;

    public void SetTheme(ThemeVariant variant)
    {
        Current = variant;
        if (Application.Current is { } app)
            app.RequestedThemeVariant = variant;
    }
}
```

### App.axaml에서 테마 파일 포함

기본 테마(예: Light)는 `Application.Styles`에 포함합니다. 런타임에 `RequestedThemeVariant`를 바꾸면 Avalonia가 자동으로 적절한 리소스를 선택합니다.

```xml
<Application.Styles>
  <FluentTheme Mode="Light" />
  <StyleInclude Source="avares://YourApp/Styles/Themes/Light.xaml" />
</Application.Styles>
```

**런타임 전환**:

```csharp
var theme = new ThemeService();
theme.SetTheme(ThemeVariant.Dark);
```

### 영역별 테마 (ThemeVariantScope)

특정 영역만 다른 테마를 적용하고 싶다면 `ThemeVariantScope`를 사용합니다.

```xml
<ThemeVariantScope RequestedThemeVariant="Dark">
  <Border Background="{DynamicResource Brush.Surface}">
    <TextBlock Text="이 영역만 다크 모드" Foreground="{DynamicResource Brush.Text.Primary}"/>
  </Border>
</ThemeVariantScope>
```

---

## 6. 모듈/패키지로 스타일 공유

스타일을 별도의 어셈블리(NuGet 패키지)로 만들면 여러 프로젝트에서 재사용할 수 있습니다.

1. 클래스 라이브러리 프로젝트에 `Styles` 폴더 생성
2. `.csproj`에 `AvaloniaResource` 포함
3. 호스트 앱에서 `StyleInclude`로 참조

```xml
<StyleInclude Source="avares://MyCompany.Controls/Styles/Buttons.xaml" />
```

---

## 7. 컨트롤 템플릿 완전 재정의 (TemplatedControl)

완전히 새로운 모양의 컨트롤을 만들고 싶다면 `TemplatedControl`을 상속받아 컨트롤 템플릿을 정의합니다.

```csharp
public class MyFancyButton : TemplatedControl
{
    public static readonly StyledProperty<object?> ContentProperty =
        AvaloniaProperty.Register<MyFancyButton, object?>(nameof(Content));

    public object? Content
    {
        get => GetValue(ContentProperty);
        set => SetValue(ContentProperty, value);
    }
}
```

### 컨트롤 템플릿 (XAML)

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui"
                    xmlns:local="clr-namespace:YourApp.Controls">
  <Style Selector="local|MyFancyButton">
    <Setter Property="Template">
      <ControlTemplate>
        <Border Background="{DynamicResource Brush.Primary}"
                CornerRadius="{DynamicResource Radius.M}">
          <ContentPresenter Content="{TemplateBinding Content}"
                            HorizontalAlignment="Center"
                            VerticalAlignment="Center"
                            Margin="{DynamicResource Space.3}"/>
        </Border>
      </ControlTemplate>
    </Setter>
  </Style>

  <Style Selector="local|MyFancyButton:pointerover">
    <Setter Property="Opacity" Value="0.95"/>
  </Style>
</ResourceDictionary>
```

사용:

```xml
<controls:MyFancyButton Content="클릭"/>
```

---

## 8. 성능 및 유지보수 팁

- **DynamicResource vs StaticResource**  
  `DynamicResource`는 런타임에 리소스 변경을 반영하므로 테마 전환이 필요한 곳에 사용합니다. 빈번히 바뀌지 않는 리소스는 `StaticResource`를 사용해 성능을 높입니다.

- **우선순위**  
  로컬 값 > 트리거 > 스타일 > 테마 리소스 순으로 적용됩니다. 의도치 않은 덮어쓰기를 피하려면 선택자를 구체적으로 작성하세요.

- **리소스 검색 범위**  
  `Control.Resources` → 상위 요소 → `Application.Resources` 순으로 탐색합니다. 특정 화면에만 적용할 리소스는 해당 범위에 배치하세요.

- **모듈화**  
  스타일을 작은 단위로 나누고 `StyleInclude`로 필요한 것만 가져오세요. 모든 스타일을 한 파일에 넣으면 유지보수가 어렵습니다.

- **디자인 타임 데이터**  
  디자인 타임에도 테마가 제대로 보이도록 `d:DataContext`와 함께 더미 데이터를 활용하세요.

---

## 9. 요약 표

| 구성 요소 | 역할 | 구현 방법 |
|-----------|------|-----------|
| 디자인 토큰 | 색상, 간격, 폰트 등 기본값을 추상화 | ResourceDictionary에 키 정의 |
| `StyleInclude` | 외부 스타일 파일 가져오기 | App.axaml에서 소스 지정 |
| 선택자(Selector) | 스타일을 적용할 대상 지정 | `Button.primary`, `Button:pointerover` 등 |
| 클래스 기반 변형 | 명시적 변형 적용 | `Classes="primary"` |
| `ThemeVariant` | 라이트/다크/고대비 전환 | `RequestedThemeVariant` 설정 |
| `TemplatedControl` | 완전한 사용자 정의 컨트롤 | ControlTemplate 정의 |

---

## 결론

Avalonia에서 스타일을 체계적으로 관리하려면 **디자인 토큰을 먼저 정의**하고, **리소스 사전을 역할별로 분리**한 후, **필요한 곳에만 포함**하는 방식이 가장 효과적입니다. `ThemeVariant`를 활용하면 런타임에 테마 전환도 손쉽게 할 수 있습니다. 모듈화된 스타일은 여러 프로젝트에서 재사용할 수 있어 생산성을 크게 높여줍니다.

이 가이드를 바탕으로 자신의 프로젝트에 맞는 스타일 구조를 설계하고, 재사용 가능한 컴포넌트 라이브러리를 만들어 보세요.