---
layout: post
title: WPF - DynamicResource vs StaticResource
date: 2025-09-07 14:25:23 +0900
category: WPF
---
# 🔁 `DynamicResource` vs `StaticResource` 완전 정복 (WPF)
*(예제 중심 · 누락 없이 최대한 자세하게 · 실무 트러블슈팅 포함)*

> 이 글은 WPF에서 **리소스 해석 시점과 갱신 방식**이 UI에 어떤 영향을 주는지,  
> **`StaticResource`** 와 **`DynamicResource`** 가 **정확히 어떻게 다른지**를  
> “개념 → 내부 동작 → 실전 예제 → 성능/디버깅 → 베스트 프랙티스” 흐름으로 설명합니다.  
> 코드 조각은 .NET 5+ 기준(Framework도 동일 개념)입니다.

---

## 0) 한눈에 요약

| 항목 | **StaticResource** | **DynamicResource** |
|---|---|---|
| **해석 시점** | **로드 시 1회**(파서가 만나는 즉시). 템플릿은 적용 시 한 번. | **런타임 지연 평가**(요청 시). 이후 **리소스 변경을 자동 반영** |
| **종속성** | 값이 **고정**됨(해석 이후는 변경 안 됨) | 리소스 사전의 해당 키가 **다른 인스턴스로 바뀌면 자동 갱신** |
| **선언 순서 의존** | **있음**. 선언 이전 키 참조 불가(파서 에러) | **거의 없음**. 나중에 Merged 시켜도 동작 |
| **성능** | 초기 로드 **빠름** / 런타임 오버헤드 **없음** | 초기 로드 **가벼움** / 런타임에 **조회·구독 비용** |
| **대표 사용처** | 고정 팔레트, 템플릿 내부 고정 값, 변하지 않는 브러시/두께 | **테마/하이콘트라스트/사용자 설정** 즉시 반영, **실시간 스킨 교체** |
| **코드로 대체** | `FindResource` | `SetResourceReference` |

> 기억: **변하지 않을 값 = Static**, **바뀔 수 있는 값(테마/OS 설정/사용자 옵션) = Dynamic**.

---

## 1) 리소스 탐색/해석 **내부 동작**

### 1.1 리소스 탐색 경로(스코프)
리소스를 찾을 때 WPF는 **가까운 곳부터 먼 곳으로** 다음 순서로 탐색합니다.

1) **요소 자신의 `Resources`**  
2) **논리 트리 상위 요소들의 `Resources`**  
3) **해당 컨트롤의 Theme 리소스**(컨트롤 라이브러리 테마)  
4) **`Application.Resources`**  
5) **시스템 리소스**(예: `SystemColors.*Key`)  

> **포인트**: 같은 키가 여러 범위에 있으면 **가장 가까운 것**이 우선.

### 1.2 StaticResource 동작
- XAML 파서가 `StaticResource`를 만나면 **즉시 탐색**해서 **인스턴스를 가져와 값으로 고정**합니다.  
- 이후 **리소스 사전 값이 바뀌어도** UI에 **반영되지 않습니다**.  
- 템플릿/스타일 내부 `StaticResource`는 **템플릿이 적용되는 순간**에 해석되어 고정됩니다.

### 1.3 DynamicResource 동작
- 파서는 **리소스를 곧바로 인스턴스화하지 않고**, **지연 참조 표현식(ResourceReferenceExpression)** 을 저장합니다.  
- 실제로 **UI 요소가 로드되거나 렌더링 시점**에 리소스를 찾습니다.  
- 해당 키의 엔트리가 **다른 객체로 교체**되면(사전에 새 인스턴스를 넣으면) **자동으로 UI가 갱신**됩니다.  
  - **중요**: 브러시 같은 Freezable의 **속성 값 변경**은(Freeze 상태면) 전파되지 않습니다. **사전 엔트리 자체를 새 인스턴스로 교체**해야 합니다.

---

## 2) 기본 예제 — 같은 코드를 `Static`/`Dynamic`으로 비교

### 2.1 팔레트 정의(리소스 파일)
```xml
<!-- Themes/Colors.Light.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation">
  <SolidColorBrush x:Key="Brush.Primary" Color="#2563EB"/>
  <SolidColorBrush x:Key="Brush.PrimaryText" Color="White"/>
</ResourceDictionary>

<!-- Themes/Colors.Dark.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation">
  <SolidColorBrush x:Key="Brush.Primary" Color="#1D4ED8"/>
  <SolidColorBrush x:Key="Brush.PrimaryText" Color="#E5E7EB"/>
</ResourceDictionary>
```

### 2.2 App.xaml에서 병합
```xml
<Application.Resources>
  <ResourceDictionary>
    <ResourceDictionary.MergedDictionaries>
      <!-- 초기 테마: Light -->
      <ResourceDictionary Source="Themes/Colors.Light.xaml"/>
    </ResourceDictionary.MergedDictionaries>
  </ResourceDictionary>
</Application.Resources>
```

### 2.3 버튼 스타일: Static vs Dynamic
```xml
<StackPanel Orientation="Horizontal" Spacing="12" Margin="16">

  <!-- Static: 테마를 바꿔도 즉시 반영되지 않음 -->
  <Button Content="Static"
          Background="{StaticResource Brush.Primary}"
          Foreground="{StaticResource Brush.PrimaryText}"/>

  <!-- Dynamic: 테마 교체 시 즉시 반영 -->
  <Button Content="Dynamic"
          Background="{DynamicResource Brush.Primary}"
          Foreground="{DynamicResource Brush.PrimaryText}"/>

</StackPanel>
```

### 2.4 테마 전환 코드
```csharp
void ApplyTheme(Uri uri) // e.g. new Uri("Themes/Colors.Dark.xaml", UriKind.Relative)
{
    var appRes = Application.Current.Resources.MergedDictionaries;
    // 0번째가 Colors.* 이라고 가정. 실제로는 이름으로 찾아 교체 권장
    appRes[0] = new ResourceDictionary { Source = uri };
}
```

- **Static** 버튼은 색이 **안 바뀜**(기존 브러시 인스턴스가 고정).  
- **Dynamic** 버튼은 **바로 바뀜**(새 브러시 인스턴스로 **재해석**).

---

## 3) “선언 순서”/“해석 시점” 차이

### 3.1 StaticResource는 선언 순서에 민감
```xml
<!-- ❌ 다음은 오류: 아직 정의되지 않은 Key 참조 -->
<SolidColorBrush x:Key="PrimaryForeground" Color="{StaticResource PrimaryColor}"/>

<SolidColorBrush x:Key="PrimaryColor" Color="#2563EB"/>
```
- 파서가 `PrimaryForeground`를 읽을 때 `PrimaryColor`가 **아직 없음** → **파서 에러**.

### 3.2 DynamicResource는 선언 순서 유연
```xml
<!-- ✅ DynamicResource는 나중에 병합/정의되어도 OK -->
<SolidColorBrush x:Key="PrimaryForeground" Color="{DynamicResource PrimaryColor}"/>

<!-- 뒤늦게 MergedDictionaries로 들어와도 추후 해석 -->
```

---

## 4) 시스템/OS 설정과의 연동(High Contrast, SystemColors)

**OS 테마/고대비 전환**을 **자동 반영**하려면 **DynamicResource**를 사용해야 합니다.

```xml
<!-- 시스템 브러시 키를 동적으로 참조 -->
<Border Background="{DynamicResource {x:Static SystemColors.WindowBrushKey}}"
        BorderBrush="{DynamicResource {x:Static SystemColors.WindowFrameBrushKey}}"
        BorderThickness="1" Padding="12">
  <TextBlock Foreground="{DynamicResource {x:Static SystemColors.WindowTextBrushKey}}"
             Text="OS 테마/고대비 변경 즉시 반영"/>
</Border>
```

- **StaticResource**로 참조하면 **앱 실행 중 바뀌는 값**을 잡지 못함.

---

## 5) 스타일/템플릿에서의 차이와 패턴

### 5.1 ControlTemplate 내부: TemplateBinding + (Dynamic/Static)
템플릿 내부에서는 **`TemplateBinding`** 로 **경량 전달**이 기본입니다.  
리소스 참조가 필요할 땐 Dynamic/Static을 **섞어서** 사용합니다.

```xml
<Style TargetType="Button" x:Key="ThemedButton">
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="Button">
        <Border x:Name="Chrome"
                Background="{DynamicResource Brush.Primary}"   <!-- 테마 즉시 반영 -->
                BorderBrush="{StaticResource Brush.Border}"    <!-- 고정 경계색 -->
                BorderThickness="1.5" CornerRadius="10">
          <ContentPresenter
             Margin="{TemplateBinding Padding}"
             HorizontalAlignment="{TemplateBinding HorizontalContentAlignment}"
             VerticalAlignment="{TemplateBinding VerticalContentAlignment}"/>
        </Border>
        <ControlTemplate.Triggers>
          <Trigger Property="IsMouseOver" Value="True">
            <Setter TargetName="Chrome" Property="Background" Value="{DynamicResource Brush.Primary.Hover}"/>
          </Trigger>
          <Trigger Property="IsPressed" Value="True">
            <Setter TargetName="Chrome" Property="Background" Value="{DynamicResource Brush.Primary.Pressed}"/>
          </Trigger>
        </ControlTemplate.Triggers>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```

- **테마 색**처럼 **바뀔 가능성**이 있는 값은 **Dynamic**으로,  
- 변하지 않을(또는 바뀌어도 즉시 반영이 굳이 필요 없는) 값은 **Static**으로.

### 5.2 ItemsControl/복잡 트리 — 가상화 영향 없음
- Dynamic/Static 선택은 **가상화 여부**에 직접 영향은 없습니다.  
- 다만 **리소스 교체 빈번** + **가상화 재사용**이 겹치면 **재측정/재배치** 부담이 늘 수 있습니다.

---

## 6) 코드 비하인드에서의 사용

### 6.1 Static: `FindResource` / `TryFindResource`
```csharp
var brush = (Brush)FindResource("Brush.Primary"); // 없으면 예외
var brush2 = (Brush)TryFindResource("Brush.Primary"); // 없으면 null
MyButton.Background = brush; // 고정
```

### 6.2 Dynamic: `SetResourceReference`
```csharp
// MyButton.Background에 “Brush.Primary”의 DynamicResource 참조를 연결
MyButton.SetResourceReference(Control.BackgroundProperty, "Brush.Primary");
// 이후 사전에서 Brush.Primary 엔트리를 다른 Brush 인스턴스로 교체하면 자동 반영
```

---

## 7) Freezable(Brush/Geometry)과 DynamicResource

- **Freezable**(예: `SolidColorBrush`)는 `Freeze()` 되면 **속성 변경 불가**.  
- DynamicResource는 **엔트리 교체**를 감지해 업데이트합니다.  
  - **브러시 객체의 Color만 바꾸는 방식**은, 해당 브러시가 **여러 곳에서 공유**되면 **모두가 함께 변**합니다.
  - 컨트롤별로 독립 애니메이션을 적용하려면 **`x:Shared="False"`** 또는 **개별 인스턴스** 사용이 필요.

```xml
<SolidColorBrush x:Key="AccentBrush" Color="#2563EB" x:Shared="False"/>
```

- **애니메이션 대상**인 브러시는 공유하지 않는 편이 안전합니다(원치 않는 동시 변화 방지).

---

## 8) 성능 고려

- **StaticResource**: 파싱 시 한 번만 해석 → **초기 비용↑**(큰 딕셔너리도 한 번에) but 런타임 오버헤드 없음.  
- **DynamicResource**: 초기 가벼움 → 런타임에 **구독/재해석/무효화**가 일어남.  
- **권장**: 바뀔 가능성이 **현실적으로 없는** 값은 **Static**.  
  테마/OS/사용자 설정으로 **바뀌어야만** 하는 값만 **Dynamic**.

> 흔한 패턴:  
> - **팔레트(색/폰트/간격)** = Dynamic (테마 전환)  
> - **템플릿 구조/고정 치수** = Static (변경 불필요)

---

## 9) MergedDictionaries 교체와 즉시 반영(테마 스위처)

### 9.1 전형적인 스위처 구현
```csharp
public static class ThemeService
{
    public static void Apply(Uri palette)
    {
        // App.Resources.MergedDictionaries 중 팔레트 사전을 찾아 교체
        var appRes = Application.Current.Resources;
        var md = appRes.MergedDictionaries;

        var index = md.Select((d,i) => (d,i))
                      .FirstOrDefault(t => t.d.Source?.OriginalString.Contains("Colors.") == true).i;

        var rd = new ResourceDictionary { Source = palette }; // e.g. "Themes/Colors.Dark.xaml"

        if (index >= 0) md[index] = rd; else md.Insert(0, rd);
        // DynamicResource로 묶여 있는 요소들은 자동으로 갱신됨
    }
}
```

### 9.2 사용
```csharp
private void ToggleTheme_Click(object sender, RoutedEventArgs e)
{
    var dark = new Uri("Themes/Colors.Dark.xaml", UriKind.Relative);
    var light = new Uri("Themes/Colors.Light.xaml", UriKind.Relative);

    // 간단 토글
    var current = Application.Current.Resources.MergedDictionaries.First().Source.OriginalString;
    ThemeService.Apply(current.Contains(".Dark.") ? light : dark);
}
```

---

## 10) StaticResource가 필요한 순간들

- **스타일/템플릿 내 성능 최적화**: 자주 참조되는 고정 리소스(두께, 코너 라운드, 고정 색 등)  
- **리소스 선언 순서가 확실**하고 **변경 의도가 전혀 없는 값**  
- **디자인 시스템에서 변동 대상이 아닌 토큰**(예: 1px BorderThickness 상수)

---

## 11) DynamicResource가 필요한 순간들

- **테마 전환**(Light/Dark/Brand Color Switch)  
- **OS 설정 변경 반영**(고대비, 시스템 색/폰트 크기)  
- **사용자 환경 설정**(가령 “기본 폰트 크기” 슬라이더 → 전체 UI 즉시 반영)

### 11.1 폰트 스케일 예제
```xml
<!-- App.xaml: 전역 폰트 스케일까지 Dynamic -->
<sys:Double x:Key="FontScale">1.0</sys:Double>

<TextBlock FontSize="{Binding Source={StaticResource FontScale}, Converter={StaticResource Multiply}, ConverterParameter=14}" />
<!-- ↑ StaticResource를 Binding Source로 쓰면 변경 반영이 안됩니다.
     폰트 스케일 같은 값은 DynamicResource를 직접 폰트에 매핑하거나, 스타일 레벨에서 반영하세요. -->

<!-- 올바른 방법 1: 스타일에 DynamicResource -->
<Style TargetType="TextBlock">
  <Setter Property="FontSize" Value="{DynamicResource BaseFontSize}"/>
</Style>
```

> **실전 팁**: “수식 기반” 글로벌 스케일은 **AttachedProperty**나 **효과(Effect)** 로 처리하되, **기준 값 자체는 Dynamic**으로 보관하세요.

---

## 12) 리소스 키/네임 충돌 & 탐색 우선순위 함정

- 같은 키를 **여러 사전**에서 정의하면 **가까운 스코프**가 우선.  
- 템플릿 내부에 같은 키가 있어 **의도치 않은 값**을 잡을 수 있음 → **접두사** 전략으로 키를 구분하세요.  
  - 예: `Btn.Primary`, `Card.Border`, `Palette.Foreground`

---

## 13) DataTemplate/ControlTemplate 내부의 해석 타이밍

- **StaticResource**: 템플릿이 **적용될 때 1회** 해석되어 **고정**됩니다.  
- **DynamicResource**: 템플릿 적용 후에도 **리소스 교체**를 **추적**하여 **갱신**됩니다.  
- **DataTemplate** 내에서 **Binding** + **DynamicResource**를 혼합할 때는 **성능**과 **변경 범위**를 염두에 두세요.

---

## 14) 애니메이션과 리소스(공유주의!)

- **공유 Brush**(리소스 브러시)를 애니메이션하면 **모든 참조자**가 함께 바뀝니다.  
  - 개별 컨트롤만 애니메이션하려면 **로컬 브러시**를 생성하거나 **`x:Shared="False"`**.  
- DynamicResource를 통해 브러시를 공급받는 경우, **애니메이션 도중 테마 스위치**가 오면 **전환 타이밍**을 고려해야 합니다.

---

## 15) 디버깅/트러블슈팅

### 15.1 “리소스가 안 잡혀요”
- **StaticResource + 선언 순서** 문제인지 확인(파서 에러/무시).  
- **스코프** 확인: 해당 요소의 상위에 원하는 키가 있는지, MergedDictionaries 순서.  
- **오탈자**: 키 이름, Source 경로(팩 URI), 빌드 액션(Resource) 체크.

### 15.2 “Dynamic인데 안 바뀌어요”
- **사전 엔트리 교체**를 하지 않고 **브러시의 Color만 변경**했는지? (Freeze 상태면 불가)  
  → **새 브러시 인스턴스를 만들어 사전 엔트리를 통째로 교체**하세요.  
- **DynamicResource를 다른 리소스의 StaticResource가 참조** 중인지?  
  - 예: A는 Dynamic인데 B를 Static이 붙잡고 있으면 B는 안 바뀜.  
  - **동적으로 바뀌는 경로는 끝까지 Dynamic**으로 설계.

### 15.3 “성능 이슈”
- DynamicResource 남용 → **무효화 폭증** 가능.  
- 대량 컨트롤 트리에서 자주 바뀌는 리소스는 **상위 레벨에서만** Dynamic로 두고, 하위는 **템플릿/바인딩 최소화**.

---

## 16) 고급: 코드에서 “전역 팔레트 값” 교체

```csharp
public static class Palette
{
    public static void SetPrimary(Color c)
    {
        var rd = Application.Current.Resources;
        // 기존 키 제거 후 새 인스턴스로 등록
        rd["Brush.Primary"] = new SolidColorBrush(c);
        // DynamicResource 참조들 자동 갱신
    }
}
```

- **주의**: 기존 브러시 인스턴스를 **수정**하는 것이 아니라, **사전 엔트리 교체**를 해야 Dynamic이 반응.

---

## 17) 실전 템플릿: 라이트/다크/하이콘트라스트 3종

### 17.1 팔레트 키 표준화
```xml
<!-- 공통 키 이름만 고정하고, 실제 색은 테마별로 다르게 -->
<SolidColorBrush x:Key="Palette.Background" Color="#FFFFFF"/>
<SolidColorBrush x:Key="Palette.Foreground" Color="#0F172A"/>
<SolidColorBrush x:Key="Palette.Accent"     Color="#2563EB"/>
<SolidColorBrush x:Key="Palette.Border"     Color="#E5E7EB"/>
```

### 17.2 컨트롤 스타일
```xml
<Style TargetType="Button" x:Key="AccentButton">
  <Setter Property="Foreground" Value="{DynamicResource Palette.Background}"/>
  <Setter Property="Background" Value="{DynamicResource Palette.Accent}"/>
  <Setter Property="BorderBrush" Value="{DynamicResource Palette.Border}"/>
  <Setter Property="Padding" Value="12,6"/>
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="Button">
        <Border x:Name="Chrome"
                Background="{TemplateBinding Background}"
                BorderBrush="{TemplateBinding BorderBrush}"
                BorderThickness="1.4" CornerRadius="10">
          <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"
                            Margin="{TemplateBinding Padding}"/>
        </Border>
        <ControlTemplate.Triggers>
          <Trigger Property="IsMouseOver" Value="True">
            <Setter TargetName="Chrome" Property="Background" Value="{DynamicResource Palette.AccentHover}"/>
          </Trigger>
          <Trigger Property="IsPressed" Value="True">
            <Setter TargetName="Chrome" Property="Background" Value="{DynamicResource Palette.AccentPressed}"/>
          </Trigger>
        </ControlTemplate.Triggers>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```

### 17.3 테마 전환(하이콘트라스트 포함)
- `Themes/Colors.Light.xaml`, `Themes/Colors.Dark.xaml`, `Themes/Colors.HighContrast.xaml` 준비  
- **DynamicResource**만 사용하면 **전환 즉시 반영**  
- OS 고대비 감지 시 자동 전환 트리거(시스템 이벤트 구독)도 가능

---

## 18) 체크리스트(베스트 프랙티스)

- [ ] **변할 수 있는 값**(테마/OS/사용자 설정)은 **DynamicResource**  
- [ ] **절대값/상수**는 **StaticResource** (성능·예측성↑)  
- [ ] **팔레트 키를 추상화**하고, **테마 사전 교체**로 스킨 전환  
- [ ] **공유 리소스 애니메이션 금지**(필요시 `x:Shared="False"` 또는 로컬 인스턴스)  
- [ ] **선언 순서 오류**는 Static에서만 발생 → 템플릿/모듈화 시 **Dynamic** 고려  
- [ ] **코드에서 전환**은 `SetResourceReference`(동적 부착) / **사전 엔트리 교체**  
- [ ] **과도한 Dynamic 남용 금지**: 성능 이슈 → 필요한 경로만 동적화

---

## 19) 미니 Q&A

**Q. DynamicResource는 값 변경 이벤트를 어떻게 감지하나요?**  
A. 리소스 엔트리(딕셔너리의 해당 키)가 **새 인스턴스로 교체되는 것**을 추적합니다. 객체 내부 속성 변경은 보장하지 않습니다(특히 `Freeze`된 `Freezable`).  

**Q. StaticResource인데도 테마가 바뀌길 원하면?**  
A. Static은 설계상 고정입니다. **Binding**이나 **DynamicResource**로 바꾸거나, **해당 속성의 값을 재설정**해야 합니다.  

**Q. DataTemplate 안에서 Static/Dynamic 혼용해도 되나요?**  
A. 가능합니다. 다만 **템플릿 적용 시점**과 **실시간 갱신 범위**를 고려하세요(많은 Dynamic은 성능에 영향).

---

## 20) 전체 샘플(요약 · 붙여 넣어 실행 가능)

### 20.1 App.xaml
```xml
<Application x:Class="ResDemo.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             StartupUri="MainWindow.xaml">
  <Application.Resources>
    <ResourceDictionary>
      <ResourceDictionary.MergedDictionaries>
        <ResourceDictionary Source="Themes/Colors.Light.xaml"/>
        <ResourceDictionary Source="Themes/Controls.xaml"/>
      </ResourceDictionary.MergedDictionaries>
    </ResourceDictionary>
  </Application.Resources>
</Application>
```

### 20.2 Themes/Controls.xaml
```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation">
  <Style TargetType="Button" x:Key="DynamicAccent">
    <Setter Property="Foreground" Value="{DynamicResource Brush.PrimaryText}"/>
    <Setter Property="Background" Value="{DynamicResource Brush.Primary}"/>
    <Setter Property="Padding" Value="12,7"/>
  </Style>

  <Style TargetType="Button" x:Key="StaticAccent">
    <Setter Property="Foreground" Value="{StaticResource Brush.PrimaryText}"/>
    <Setter Property="Background" Value="{StaticResource Brush.Primary}"/>
    <Setter Property="Padding" Value="12,7"/>
  </Style>
</ResourceDictionary>
```

### 20.3 MainWindow.xaml
```xml
<Window x:Class="ResDemo.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Dynamic vs Static" Width="520" Height="240">
  <StackPanel Margin="16" Spacing="10">
    <StackPanel Orientation="Horizontal" Spacing="10">
      <Button Content="Static Button" Style="{StaticResource StaticAccent}"/>
      <Button Content="Dynamic Button" Style="{StaticResource DynamicAccent}"/>
    </StackPanel>
    <StackPanel Orientation="Horizontal" Spacing="10">
      <Button Content="Light"  Click="Light_Click"/>
      <Button Content="Dark"   Click="Dark_Click"/>
    </StackPanel>
    <TextBlock Text="System Window Brush"
               Background="{DynamicResource {x:Static SystemColors.WindowBrushKey}}"
               Foreground="{DynamicResource {x:Static SystemColors.WindowTextBrushKey}}"
               Padding="8"/>
  </StackPanel>
</Window>
```

### 20.4 MainWindow.xaml.cs
```csharp
public partial class MainWindow : Window
{
    public MainWindow() => InitializeComponent();

    void Switch(Uri uri)
    {
        var md = Application.Current.Resources.MergedDictionaries;
        md[0] = new ResourceDictionary { Source = uri };
    }

    private void Light_Click(object s, RoutedEventArgs e) =>
        Switch(new Uri("Themes/Colors.Light.xaml", UriKind.Relative));

    private void Dark_Click(object s, RoutedEventArgs e) =>
        Switch(new Uri("Themes/Colors.Dark.xaml", UriKind.Relative));
}
```

실행 후 `Dark` 버튼을 누르면 **Dynamic Button**만 즉시 색이 바뀌고,  
`Static Button`은 **바뀌지 않음**을 확인할 수 있습니다.

---

## 결론

- **StaticResource**는 **빠르고 단순**합니다. **변하지 않을 값**에 적합합니다.  
- **DynamicResource**는 **유연**합니다. **테마/OS/사용자 설정** 같은 **런타임 변경**에 반응해야 할 때 필수입니다.  
- 성능·예측성·유지보수를 고려해 **“필요한 경로만 Dynamic”** 으로 설계하는 것이 최적의 전략입니다.
