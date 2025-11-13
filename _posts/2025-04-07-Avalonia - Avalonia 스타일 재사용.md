---
layout: post
title: Avalonia - Avalonia 스타일 재사용
date: 2025-04-07 22:20:23 +0900
category: Avalonia
---
# Avalonia 스타일 재사용(Themes, Styles)

1) **디자인 토큰화**(색/타이포/간격/컴포넌트 변수화)
2) **리소스 사전 분리와 병합 전략**(ResourceDictionary, MergedDictionaries, StyleInclude)
3) **ThemeVariant 기반 다크/라이트/고대비 전환**(런타임 스위칭 포함)
4) **TemplatedControl로 완전 재정의 + 모듈 단위 배포**(NuGet/Module)

---

## 0. 용어·구성 요약

| 개념 | 설명 |
|------|------|
| `Style` | 컨트롤들에 공통으로 적용할 속성 묶음. 선택자(Selector)로 대상으로 지정 |
| `Theme` | 앱(또는 영역) 전체에 적용되는 **스타일+리소스 묶음** |
| `ResourceDictionary` | 색/브러시/간격/스타일 등 **리소스 컨테이너** |
| `UserControl` | 조합형 사용자 컨트롤(템플릿 전체 재정의는 아님) |
| `TemplatedControl` | 템플릿을 갖는 컨트롤. 시각/구조 완전 재정의(컨트롤 테마) |
| `ThemeVariant` | Avalonia 11+의 Light/Dark/HighContrast 등 **테마 변형 키** |

---

## 1. 디자인 토큰(Design Tokens)으로 **일관성** 확보

대규모 앱은 색상/타이포/간격/반경을 **토큰**으로 모아야 유지보수성이 높아진다.

### 1.1 프로젝트 구조 예시

```
Styles/
├── Tokens/
│   ├── Colors.xaml           # 색 팔레트(역할 기반)
│   ├── Typography.xaml       # 폰트, 크기, 줄간격
│   ├── Spacing.xaml          # 간격 단위(scale 0..n)
│   └── Elevation.xaml        # 그림자/윤곽선 레벨
├── Controls/
│   ├── Buttons.xaml          # Button 변형(Primary, Danger 등)
│   ├── Inputs.xaml           # TextBox, ComboBox 등
│   ├── Lists.xaml            # ListBox, DataGrid 등
│   └── Dialogs.xaml
└── Themes/
    ├── Light.xaml
    ├── Dark.xaml
    └── HighContrast.xaml
```

### 1.2 Colors.xaml — 역할(role) 기반 팔레트

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <!-- Primitive 색상(Brand/Neutral/Support) -->
  <Color x:Key="Color.Brand.500">#2D7FF9</Color>
  <Color x:Key="Color.Brand.600">#1E6AE6</Color>
  <Color x:Key="Color.Neutral.000">#FFFFFF</Color>
  <Color x:Key="Color.Neutral.900">#121212</Color>
  <Color x:Key="Color.Success.500">#2EB872</Color>
  <Color x:Key="Color.Danger.500">#E5534B</Color>

  <!-- 역할(semantic) 추상화 -->
  <SolidColorBrush x:Key="Brush.Surface" Color="{DynamicResource Color.Neutral.000}"/>
  <SolidColorBrush x:Key="Brush.Text.Primary" Color="#1E1E1E"/>
  <SolidColorBrush x:Key="Brush.Text.Inverse" Color="#FFFFFF"/>
  <SolidColorBrush x:Key="Brush.Primary" Color="{DynamicResource Color.Brand.500}"/>
  <SolidColorBrush x:Key="Brush.Primary.Active" Color="{DynamicResource Color.Brand.600}"/>
  <SolidColorBrush x:Key="Brush.Success" Color="{DynamicResource Color.Success.500}"/>
  <SolidColorBrush x:Key="Brush.Danger" Color="{DynamicResource Color.Danger.500}"/>
</ResourceDictionary>
```

### 1.3 Typography.xaml — 폰트/계층화

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <FontFamily x:Key="Font.Primary">Segoe UI, Noto Sans CJK KR, Arial</FontFamily>

  <x:Double x:Key="FontSize.Display">28</x:Double>
  <x:Double x:Key="FontSize.Headline">22</x:Double>
  <x:Double x:Key="FontSize.Body">14</x:Double>
  <x:Double x:Key="FontSize.Caption">12</x:Double>
</ResourceDictionary>
```

### 1.4 Spacing.xaml — 간격 스케일

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

> 이처럼 **역할 기반 키**로 정의하면 다크/라이트 전환이나 디자인 개편 시 토큰만 바꿔도 UI 전체가 일관되게 반응한다.

---

## 2. App.axaml — 리소스 사전 병합과 테마 로딩

### 2.1 App.axaml에 토큰/컨트롤/테마 포함

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="YourApp.App">
  <Application.Styles>
    <!-- Fluent 테마 + 초기 테마변형 -->
    <FluentTheme Mode="Light" />

    <!-- Design Tokens -->
    <StyleInclude Source="avares://YourApp/Styles/Tokens/Colors.xaml" />
    <StyleInclude Source="avares://YourApp/Styles/Tokens/Typography.xaml" />
    <StyleInclude Source="avares://YourApp/Styles/Tokens/Spacing.xaml" />
    <StyleInclude Source="avares://YourApp/Styles/Tokens/Elevation.xaml" />

    <!-- Controls 스타일 -->
    <StyleInclude Source="avares://YourApp/Styles/Controls/Buttons.xaml" />
    <StyleInclude Source="avares://YourApp/Styles/Controls/Inputs.xaml" />
    <StyleInclude Source="avares://YourApp/Styles/Controls/Lists.xaml" />
    <StyleInclude Source="avares://YourApp/Styles/Controls/Dialogs.xaml" />

    <!-- ThemeVariant 리소스(아래 3개 중 하나를 Scope로 교체 가능) -->
    <StyleInclude Source="avares://YourApp/Styles/Themes/Light.xaml" />
  </Application.Styles>
</Application>
```

> `StyleInclude`는 재사용 가능한 스타일 파일을 가져오는 가장 간단한 방법이다. 모듈/패키지에 담아 배포할 때도 동일한 방식으로 포함 가능.

---

## 3. 기본 스타일 선언 & 적용 — 선택자와 변형(variant)

### 3.1 Button 변형(variant) 스타일

```xml
<!-- Buttons.xaml -->
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <!-- Primary -->
  <Style Selector="Button.primary">
    <Setter Property="Background" Value="{DynamicResource Brush.Primary}"/>
    <Setter Property="Foreground" Value="{DynamicResource Brush.Text.Inverse}"/>
    <Setter Property="FontWeight" Value="Bold"/>
    <Setter Property="Padding" Value="{DynamicResource Space.3}"/>
    <Setter Property="CornerRadius" Value="{DynamicResource Radius.M}"/>
  </Style>

  <!-- Danger -->
  <Style Selector="Button.danger">
    <Setter Property="Background" Value="{DynamicResource Brush.Danger}"/>
    <Setter Property="Foreground" Value="{DynamicResource Brush.Text.Inverse}"/>
    <Setter Property="Padding" Value="{DynamicResource Space.3}"/>
    <Setter Property="CornerRadius" Value="{DynamicResource Radius.M}"/>
  </Style>

  <!-- 상태 기반 가상 선택자 -->
  <Style Selector="Button.primary:pointerover">
    <Setter Property="Background" Value="{DynamicResource Brush.Primary.Active}"/>
  </Style>
  <Style Selector="Button:disabled">
    <Setter Property="Opacity" Value="0.5"/>
  </Style>
</ResourceDictionary>
```

사용:

```xml
<Button Classes="primary" Content="확인"/>
<Button Classes="danger"  Content="삭제"/>
```

> 클래스 기반 변형은 **간단·명시적**이고 MVVM에도 적합하다.

---

## 4. 리소스 분리와 병합 — 모듈/패키지로 확장

### 4.1 모듈에서 Styles를 제공하는 NuGet 패키지

- 모듈 `YourApp.Modules.Reports` 내 `Styles/Reports.xaml` 생성
- `.csproj`에 `AvaloniaResource`로 포함
- **호스트 앱**의 `App.axaml`에 다음 추가:

```xml
<StyleInclude Source="avares://YourApp.Modules.Reports/Styles/Reports.xaml" />
```

> 팀/조직 내 공통 컴포넌트를 **NuGet 패키지화**하면 앱마다 쉽게 테마와 컨트롤을 공유 가능.

---

## 5. 다크/라이트/고대비 — ThemeVariant 및 런타임 전환

Avalonia 11부터는 `ThemeVariant`를 통해 테마 변형을 자연스럽게 관리할 수 있다.
대표 전략 두 가지:

1) **전역 교체형**: `Application.Styles[i]`에 들어있는 Theme 사전을 통째로 교체
2) **범위 지정형**: `ThemeVariantScope`로 **영역별** 테마 지정

### 5.1 Theme 리소스 예시(Light.xaml / Dark.xaml)

```xml
<!-- Light.xaml -->
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <SolidColorBrush x:Key="Brush.Surface" Color="#FFFFFF"/>
  <SolidColorBrush x:Key="Brush.Text.Primary" Color="#1E1E1E"/>
  <!-- Primary 등은 Colors.xaml의 Brand 계열을 재지정할 수도 있음 -->
</ResourceDictionary>
```

```xml
<!-- Dark.xaml -->
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <SolidColorBrush x:Key="Brush.Surface" Color="#121212"/>
  <SolidColorBrush x:Key="Brush.Text.Primary" Color="#EEEEEE"/>
</ResourceDictionary>
```

### 5.2 런타임 전환 서비스(ThemeService)

```csharp
public interface IThemeService
{
    void SetThemeVariant(ThemeVariant variant); // Light/Dark/HighContrast
    ThemeVariant Current { get; }
}

public sealed class ThemeService : IThemeService
{
    public ThemeVariant Current { get; private set; } = ThemeVariant.Light;

    public void SetThemeVariant(ThemeVariant variant)
    {
        Current = variant;

        // 전역 ThemeVariant (FluentTheme에도 반영 가능)
        if (Application.Current is { } app)
        {
            app.RequestedThemeVariant = variant;

            // 리소스 사전 교체형(옵션):
            // app.Styles[index] = new StyleInclude { Source = new Uri("avares://YourApp/Styles/Themes/Dark.xaml") };
        }
    }
}
```

호출:

```csharp
_themeService.SetThemeVariant(ThemeVariant.Dark);
```

### 5.3 영역별 테마(ThemeVariantScope)

```xml
<ThemeVariantScope RequestedThemeVariant="Dark">
  <Border Background="{DynamicResource Brush.Surface}">
    <TextBlock Text="이 영역만 다크 모드" Foreground="{DynamicResource Brush.Text.Primary}"/>
  </Border>
</ThemeVariantScope>
```

> 고대비(HighContrast) 전용 사전도 따로 두어 접근성 대응을 강화할 수 있다.

---

## 6. 스타일 상속(BasedOn)과 키 스타일(Keyed Style)

### 6.1 BasedOn으로 변형만 다른 스타일

```xml
<Style x:Key="Button.Base" Selector="Button">
  <Setter Property="FontFamily" Value="{DynamicResource Font.Primary}"/>
  <Setter Property="Padding" Value="{DynamicResource Space.3}"/>
  <Setter Property="CornerRadius" Value="{DynamicResource Radius.S}"/>
</Style>

<Style Selector="Button.primary" BasedOn="{StaticResource Button.Base}">
  <Setter Property="Background" Value="{DynamicResource Brush.Primary}"/>
  <Setter Property="Foreground" Value="{DynamicResource Brush.Text.Inverse}"/>
</Style>
```

### 6.2 키 지정 스타일을 특정 컨트롤에만 적용

```xml
<Style x:Key="SearchTextBoxStyle" Selector="TextBox">
  <Setter Property="Watermark" Value="검색..."/>
  <Setter Property="CornerRadius" Value="{DynamicResource Radius.S}"/>
</Style>

<!-- 사용 -->
<TextBox Styles="{StaticResource SearchTextBoxStyle}" />
```

> **Selector 스타일**은 대상 자동 적용, **키 스타일**은 명시 적용. 상황에 맞게 혼용.

---

## 7. DataTrigger/가상 선택자 — 상태 기반 스타일

### 7.1 가상 선택자

```xml
<Style Selector="Button:pointerover">
  <Setter Property="Opacity" Value="0.9"/>
</Style>
<Style Selector="Button:pressed">
  <Setter Property="RenderTransform">
    <ScaleTransform ScaleX="0.98" ScaleY="0.98"/>
  </Setter>
</Style>
<Style Selector="Button:disabled">
  <Setter Property="Opacity" Value="0.5"/>
</Style>
```

### 7.2 DataTrigger(속성 값에 따라)

```xml
<Style Selector="TextBlock">
  <Style.Triggers>
    <DataTrigger Binding="{Binding HasError}" Value="True">
      <Setter Property="Foreground" Value="{DynamicResource Brush.Danger}"/>
      <Setter Property="FontWeight" Value="Bold"/>
    </DataTrigger>
  </Style.Triggers>
</Style>
```

---

## 8. 컨트롤 템플릿 완전 재정의 — TemplatedControl

### 8.1 커스텀 컨트롤(TemplatedControl)

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

### 8.2 컨트롤 테마(Style)로 Template 정의

```xml
<!-- Controls/FancyButton.xaml -->
<ResourceDictionary xmlns="https://github.com/avaloniaui"
                    xmlns:local="clr-namespace:YourApp.Controls">
  <Style Selector="local|MyFancyButton">
    <Setter Property="Template">
      <ControlTemplate>
        <Border Background="{DynamicResource Brush.Primary}" CornerRadius="{DynamicResource Radius.M}">
          <ContentPresenter Content="{TemplateBinding Content}"
                            HorizontalAlignment="Center" VerticalAlignment="Center"
                            Margin="{DynamicResource Space.3}"
                            TextElement.Foreground="{DynamicResource Brush.Text.Inverse}"/>
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
<controls:MyFancyButton Content="커스텀"/>
```

> `TemplateBinding`으로 템플릿 내부에서 컨트롤의 속성을 매핑한다.

---

## 9. 애니메이션을 스타일에 조합(선택)

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui"
                    xmlns:ani="clr-namespace:Avalonia.Animation;assembly=Avalonia.Animation">
  <Style Selector="Button.primary:pointerover">
    <Style.Animations>
      <ani:Animations>
        <ani:Scale Duration="0:0:0.15" Easing="CubicOut" From="1" To="1.03"/>
      </ani:Animations>
    </Style.Animations>
  </Style>
</ResourceDictionary>
```

---

## 10. 런타임 테마 전환 UI 예시

### 10.1 ThemeSelectorViewModel

```csharp
public sealed class ThemeSelectorViewModel : ReactiveObject
{
    private readonly IThemeService _theme;

    public ThemeSelectorViewModel(IThemeService theme) => _theme = theme;

    public void SetLight()        => _theme.SetThemeVariant(ThemeVariant.Light);
    public void SetDark()         => _theme.SetThemeVariant(ThemeVariant.Dark);
    public void SetHighContrast() => _theme.SetThemeVariant(ThemeVariant.HighContrast);
}
```

### 10.2 XAML

```xml
<StackPanel Orientation="Horizontal" Spacing="8">
  <Button Content="Light" Click="(vm) => vm.SetLight()"/>
  <Button Content="Dark" Click="(vm) => vm.SetDark()"/>
  <Button Content="HC"   Click="(vm) => vm.SetHighContrast()"/>
</StackPanel>
```

> MVVM 순수성을 지키려면 Command 바인딩으로 전환.

---

## 11. 성능·우선순위·모듈화 팁

- **우선순위**: 로컬 값 > 트리거 > 스타일 > 테마/리소스.
  의도치 않은 덮어쓰기를 피하려면 변형마다 **명확한 Selector**를 사용.
- **DynamicResource vs StaticResource**: 테마 전환/실행 중 변경이 필요하면 **Dynamic**을 사용.
- **리소스 탐색 범위**: Control.Resources → 상위 → Application.Resources.
  **스코프 리소스**를 활용하면 특정 화면/모듈만 별도 테마 적용 가능.
- **대규모 앱 최적화**: Styles 파일을 **여러 개로 분할**하고 App.axaml에서 **필요한 것만** Include.
- **모듈(NuGet)화**: Controls/Styles/Themes를 **모듈 패키지**로 묶고, 호스트에서 Include만 하도록 설계.

---

## 12. 예시 확장: 검색 상자 스타일 일괄 적용

```xml
<!-- Inputs.xaml -->
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <Style Selector="TextBox.search">
    <Setter Property="CornerRadius" Value="{DynamicResource Radius.S}"/>
    <Setter Property="Background" Value="{DynamicResource Brush.Surface}"/>
    <Setter Property="Watermark" Value="검색..."/>
    <Setter Property="Padding" Value="{DynamicResource Space.2}"/>
  </Style>

  <Style Selector="TextBox.search:pointerover">
    <Setter Property="BorderBrush" Value="{DynamicResource Brush.Primary}"/>
    <Setter Property="BorderThickness" Value="1.5"/>
  </Style>
</ResourceDictionary>
```

사용:

```xml
<TextBox Classes="search" Text="{Binding Query, Mode=TwoWay}"/>
```

---

## 13. 접근성(HighContrast)와 사용자 설정

- HighContrast.xaml에서 텍스트 대비, 포커스 링 두께 등을 **강조**한다.
- 폰트 크기를 사용자 설정과 연결하여 `FontSize` 토큰을 **확대/축소**할 수 있게 한다.
- 키보드 포커스 시각화(예: `:focus` 선택자)와 접근성 지표를 스타일로 명확히 반영한다.

---

## 14. 테스트 전략

- **시각 회귀 테스트**: 샷 비교(Playwright/스크린샷)로 테마 변경 시 파손 여부 확인.
- **리소스 참조 검증**: 주요 컨트롤 템플릿의 리소스 키 사용 여부 단위 테스트(정적 분석 수준).
- **다크/라이트 스위치 테스트**: 전환 후 컬러/브러시 실 값이 바뀌었는지 간단한 UI 테스트.

---

## 15. 요약 표

| 주제 | 핵심 |
|------|------|
| 디자인 토큰 | 색/간격/폰트/그림자를 **역할 기반 리소스**로 추상화 |
| 스타일 분리 | Tokens/Controls/Themes로 구조화, `StyleInclude`로 로드 |
| 변형/상태 | `Classes="primary"` + `:pointerover/:disabled` 등 **Selector** |
| 테마 전환 | `ThemeVariant`(Light/Dark/HC) + `ThemeService` 런타임 스위치 |
| 완전 재정의 | `TemplatedControl` + ControlTemplate |
| 성능/유지보수 | 분할·모듈화·우선순위 관리·DynamicResource 남용 주의 |

---

## 부록 A. App.axaml에서 **한 번에** 모든 스타일 결선 예

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="YourApp.App">
  <Application.Styles>
    <FluentTheme Mode="Light"/>

    <!-- Tokens -->
    <StyleInclude Source="avares://YourApp/Styles/Tokens/Colors.xaml"/>
    <StyleInclude Source="avares://YourApp/Styles/Tokens/Typography.xaml"/>
    <StyleInclude Source="avares://YourApp/Styles/Tokens/Spacing.xaml"/>
    <StyleInclude Source="avares://YourApp/Styles/Tokens/Elevation.xaml"/>

    <!-- Components -->
    <StyleInclude Source="avares://YourApp/Styles/Controls/Buttons.xaml"/>
    <StyleInclude Source="avares://YourApp/Styles/Controls/Inputs.xaml"/>
    <StyleInclude Source="avares://YourApp/Styles/Controls/Lists.xaml"/>
    <StyleInclude Source="avares://YourApp/Styles/Controls/Dialogs.xaml"/>

    <!-- ThemeVariant (기본: Light) -->
    <StyleInclude Source="avares://YourApp/Styles/Themes/Light.xaml"/>
  </Application.Styles>
</Application>
```

---

## 부록 B. 코드에서 ThemeVariant 전환

```csharp
public partial class App : Application
{
    public override void OnFrameworkInitializationCompleted()
    {
        base.OnFrameworkInitializationCompleted();

        RequestedThemeVariant = ThemeVariant.Light; // 기본
        // 나중에 ThemeService 통해 Dark/HC로 변경 가능.
    }
}
```

---

## 결론

- **토큰화**로 디자인 일관성을 유지하고,
- **리소스/스타일을 모듈화**하여 팀·앱 간 **재사용**을 극대화하며,
- **ThemeVariant + DynamicResource**로 **런타임 테마 전환**을 안전하게 지원하고,
- 고급 커스터마이징이 필요하면 **TemplatedControl**로 완전 재정의하자.
