---
layout: post
title: Avalonia - Avalonia 스타일 재사용
date: 2025-04-07 22:20:23 +0900
category: Avalonia
---
# 🎨 Avalonia 스타일 재사용 (Themes, Styles)

## 🎯 핵심 개념 요약

| 개념 | 설명 |
|------|------|
| `Style` | 컨트롤에 반복 적용되는 속성 묶음 |
| `Theme` | 앱 전역에 적용되는 스타일 모음 |
| `ResourceDictionary` | 스타일/리소스를 담는 컨테이너 |
| `UserControl` | 사용자 정의 컨트롤 |
| `TemplatedControl` | 완전한 스타일 커스터마이징 컨트롤 |

---

## 1️⃣ 기본 스타일 선언 & 적용

### ✔️ Button 스타일 예시

```xml
<Style Selector="Button.primary">
  <Setter Property="Background" Value="#0078D4"/>
  <Setter Property="Foreground" Value="White"/>
  <Setter Property="FontWeight" Value="Bold"/>
  <Setter Property="Padding" Value="10"/>
</Style>
```

```xml
<Button Classes="primary" Content="확인" />
```

- `Selector="Button.primary"`: `Classes="primary"`인 Button에 적용됨

---

## 2️⃣ ResourceDictionary로 스타일 분리

### 📁 프로젝트 구조 예시

```
Styles/
├── Colors.xaml
├── Buttons.xaml
└── Themes/
    ├── Light.xaml
    └── Dark.xaml
```

### 📄 `Buttons.xaml`

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <Style Selector="Button.round">
    <Setter Property="CornerRadius" Value="20"/>
    <Setter Property="Background" Value="#00BFFF"/>
  </Style>
</ResourceDictionary>
```

### 📄 App.axaml에서 등록

```xml
<Application.Styles>
  <FluentTheme Mode="Light" />
  <StyleInclude Source="avares://YourApp/Styles/Buttons.xaml" />
</Application.Styles>
```

---

## 3️⃣ 다크/라이트 테마 전환 구조

```xml
<!-- Themes/Light.xaml -->
<ResourceDictionary xmlns="https://github.com/avaloniaui">
  <Color x:Key="AppBackground">White</Color>
  <SolidColorBrush x:Key="AppBackgroundBrush" Color="{DynamicResource AppBackground}"/>
</ResourceDictionary>
```

```csharp
// App.xaml.cs
public void SetTheme(string theme)
{
    var dict = new StyleInclude(new Uri("resm:Styles?assembly=YourApp"))
    {
        Source = new Uri($"avares://YourApp/Styles/Themes/{theme}.xaml")
    };

    Application.Current.Styles[1] = dict; // 테마만 교체
}
```

---

## 4️⃣ 사용자 정의 컨트롤

### 📄 CustomButton.axaml

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             x:Class="YourApp.Controls.CustomButton">
  <Button Classes="primary" Content="{Binding Content}" />
</UserControl>
```

### 📄 CustomButton.cs

```csharp
public partial class CustomButton : UserControl
{
    public static readonly StyledProperty<string> ContentProperty =
        AvaloniaProperty.Register<CustomButton, string>(nameof(Content));

    public string Content
    {
        get => GetValue(ContentProperty);
        set => SetValue(ContentProperty, value);
    }

    public CustomButton()
    {
        InitializeComponent();
        DataContext = this;
    }
}
```

### 사용

```xml
<controls:CustomButton Content="로그인" />
```

---

## 5️⃣ TemplatedControl로 확장 스타일링

### `MyFancyButton.axaml` (제어 템플릿 분리)

```xml
<Style Selector="local|MyFancyButton">
  <Setter Property="Template">
    <ControlTemplate>
      <Border Background="Gray" CornerRadius="10">
        <ContentPresenter />
      </Border>
    </ControlTemplate>
  </Setter>
</Style>
```

- TemplatedControl은 Button처럼 **기능이 있는 컨트롤의 스타일 구조를 바꿀 때 사용**

---

## 6️⃣ 동적 스타일 변경 (예: 상태에 따라 색 변경)

```xml
<Style Selector="Button:disabled">
  <Setter Property="Opacity" Value="0.4"/>
  <Setter Property="Background" Value="Gray"/>
</Style>
```

- `:disabled`, `:pointerover`, `:pressed` 등 **가상 상태 선택자** 사용 가능

---

## 7️⃣ 글로벌 리소스와 Static/Dynamic 바인딩

```xml
<SolidColorBrush x:Key="PrimaryBrush" Color="#3498db" />
<TextBlock Foreground="{DynamicResource PrimaryBrush}" />
```

- `DynamicResource`: 런타임 테마 변경에 대응 가능  
- `StaticResource`: 앱 시작 시 고정 값

---

## 8️⃣ 템플릿, 스타일, 테마 결합 예시

```xml
<Style Selector="TextBox.search">
  <Setter Property="CornerRadius" Value="5"/>
  <Setter Property="Watermark" Value="검색..."/>
  <Setter Property="Background" Value="{DynamicResource InputBackgroundBrush}"/>
</Style>
```

---

## ✅ 정리

| 항목 | 내용 |
|------|------|
| `Style` | 반복되는 UI 속성을 선언해 재사용 |
| `ResourceDictionary` | 스타일 파일 분리 및 전역 적용 |
| `DynamicResource` | 런타임 테마 변경 대응 |
| `UserControl` | 조합형 사용자 컨트롤 |
| `TemplatedControl` | 완전한 시각적 재정의 |
| `StyleInclude` | App.axaml에 외부 스타일 포함 |

---

## 📚 참고 자료

- [Avalonia Styling 공식 문서](https://docs.avaloniaui.net/docs/styling/styles)
- [XAML 스타일 구조 참고](https://docs.microsoft.com/en-us/dotnet/desktop/wpf/controls/styling-and-templating)

---

## 🔚 결론

Avalonia는 강력한 스타일 시스템을 갖추고 있으며, 다음과 같은 경우에 매우 효과적입니다:

- 🎨 **디자인 일관성 유지**  
- 🌗 **다크/라이트 모드 전환**  
- ♻️ **공통 UI 컴포넌트 재사용**  
- 🧱 **유지보수 쉬운 구조화**

스타일을 분리하고 테마와 리소스를 동적으로 교체하면 대규모 UI 유지 관리에 큰 도움이 됩니다.
