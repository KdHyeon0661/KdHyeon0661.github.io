---
layout: post
title: WPF - UserControl vs CustomControl
date: 2025-09-07 15:25:23 +0900
category: WPF
---
# 🧩 WPF에서 **UserControl vs CustomControl** 완전 정복
*(예제 중심 · 누락 없이 최대한 자세하게 · 실전 선택 기준 + 제작 체크리스트 포함)*

> WPF에서 “컴포넌트화”를 할 때 가장 먼저 부딪히는 질문:
> **UserControl**을 만들까, **CustomControl**을 만들까?
> 두 유형은 **제작 방식·확장성·스타일링·재사용성**이 본질적으로 다릅니다.
> 이 글은 **개념 → 차이점 표 → 언제 어떤 걸 쓰는가 → 코드 예제** 순서로 정리합니다.

---

## 0. 한눈에 보는 차이

| 항목 | **UserControl** | **CustomControl** |
|---|---|---|
| **정의** | XAML로 **내부 UI가 고정된 합성 컨트롤** | **Control** 파생 + `ControlTemplate`로 **외형을 스킨으로 교체 가능한 컨트롤** |
| **템플릿 교체** | 어렵거나 제한적(내부 시각 트리가 고정) | 정상 경로(기본이 템플릿 교체) — `Themes/Generic.xaml` |
| **스타일링 범위** | 내부 요소까지 건드리려면 **컨트롤 수정** 필요 | 외부에서 **Style/Template**로 외형 전부 교체 가능 |
| **의존성 속성** | 만들 수 있음(보통 간단) | **필수**(속성/상태/VSM/명령 라우팅까지 포함) |
| **재사용/브랜딩** | **프로젝트/팀 내부 UI 복제**에 적합 | **컨트롤 라이브러리/SDK** 제작에 적합 |
| **디자인-타임** | XAML 디자이너 즉시 미리보기 쉬움 | Template 분리로 초반 설정 필요(미리보기 지정 권장) |
| **성능/경량** | 내부 트리 고정 → 간단 UI에 유리 | 템플릿 교체/트리거/VSM… 확장에 유리(대규모 앱 적합) |
| **학습 곡선** | 낮음(바로 XAML 합성) | 높음(템플릿·PART·OnApplyTemplate·Generic.xaml) |
| **테마/다크모드** | 특정 컨트롤별로 직접 구현해야 | **ResourceDictionary 교체 + DynamicResource**로 일괄 |
| **익스텐션 포인트** | 보통 **Public 속성 + 이벤트** | **DP + RoutedEvent + Command + PART 계약** |

> 기억:
> - **UserControl** = “내가 쓰려고 만든 작은 화면 조각” (**합성 View**)
> - **CustomControl** = “다른 사람도 스킨 바꿔 쓰라고 만든 **진짜 컨트롤**”

---

## 1. 언제 어떤 걸 써야 하나 — 결정 트리

1) **외형을 소비자가 자유롭게 바꿔야 하나?**
   - 예: 회사별 브랜딩, 다크/라이트/고대비 테마, 둥근/각진 버튼 등
   → **CustomControl** (템플릿 교체가 1급 시민)

2) **내부에서 쓰는 화면 조각을 빠르게 조립하고 싶은가?**
   - 복잡한 페이지를 기능 블록으로 나눠 합성
   → **UserControl** (개발 속도↑)

3) **컨트롤 라이브러리/SDK를 배포할 계획인가?**
   → 99% **CustomControl**

4) **데이터 입력 폼 일부만 캡슐화**(특정 UI 세트) 하고 바깥에서 스킨을 바꿀 필요가 없는가?
   → **UserControl**

---

## 2. UserControl 깊게 이해하기

### 2.1 특징
- **XAML 합성**으로 내부 UI가 **고정**되어 있습니다.
- 외부에 노출되는 건 보통 `public` CLR 속성·`DependencyProperty`·이벤트.
- **템플릿 교체가 목적이 아닌** 화면 파편을 재사용하기에 적합.

### 2.2 기본 예제 — `ProfileCard`
```xml
<!-- ProfileCard.xaml -->
<UserControl x:Class="Demo.Controls.ProfileCard"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
  <Border CornerRadius="10" Padding="12" Background="#FFF" BorderBrush="#DDD" BorderThickness="1">
    <StackPanel Orientation="Horizontal" Spacing="12">
      <Ellipse Width="48" Height="48">
        <Ellipse.Fill>
          <ImageBrush ImageSource="{Binding Photo, RelativeSource={RelativeSource AncestorType=UserControl}}"/>
        </Ellipse.Fill>
      </Ellipse>
      <StackPanel>
        <TextBlock Text="{Binding Name, RelativeSource={RelativeSource AncestorType=UserControl}}" FontSize="16" FontWeight="SemiBold"/>
        <TextBlock Text="{Binding Title, RelativeSource={RelativeSource AncestorType=UserControl}}" Foreground="#666"/>
      </StackPanel>
    </StackPanel>
  </Border>
</UserControl>
```

```csharp
// ProfileCard.xaml.cs
public partial class ProfileCard : UserControl
{
    public ProfileCard() => InitializeComponent();

    public ImageSource? Photo { get => (ImageSource?)GetValue(PhotoProperty); set => SetValue(PhotoProperty, value); }
    public static readonly DependencyProperty PhotoProperty =
        DependencyProperty.Register(nameof(Photo), typeof(ImageSource), typeof(ProfileCard));

    public string? Name { get => (string?)GetValue(NameProperty); set => SetValue(NameProperty, value); }
    public static readonly DependencyProperty NameProperty =
        DependencyProperty.Register(nameof(Name), typeof(string), typeof(ProfileCard));

    public string? Title { get => (string?)GetValue(TitleProperty); set => SetValue(TitleProperty, value); }
    public static readonly DependencyProperty TitleProperty =
        DependencyProperty.Register(nameof(Title), typeof(string), typeof(ProfileCard));
}
```

- 내부 레이아웃(둥근 사진/StackPanel)을 **외부에서 스킨 교체**할 수 없습니다.
- 버튼이나 색상 변경 같은 **세부 외형은 코드 수정**이 필요합니다.

### 2.3 장단점
- ✅ 장점: 제작이 **빠름**, 로직이 **단순**, 디자이너/미리보기 편함
- ⚠️ 단점: 외형 커스터마이즈 **한계**, 대규모 스타일·테마 체계에 **부적합**

---

## 3. CustomControl 깊게 이해하기

### 3.1 구조 핵심
- **`public class MyControl : Control`** 로 시작 (또는 `ButtonBase`/`ItemsControl` 등).
- **`Themes/Generic.xaml`** 에 **기본 스타일/템플릿**을 제공.
- **속성은 DP**, 외형은 **ControlTemplate**, 내부 파츠는 **`PART_*` 계약**.

### 3.2 최소 구현 스켈레톤
```
MyControlLibrary/
  Controls/
    Gauge.cs
  Themes/
    Generic.xaml
```

```csharp
// Gauge.cs
public class Gauge : Control
{
    static Gauge()
    {
        DefaultStyleKeyProperty.OverrideMetadata(typeof(Gauge),
            new FrameworkPropertyMetadata(typeof(Gauge)));
    }

    public double Value { get => (double)GetValue(ValueProperty); set => SetValue(ValueProperty, value); }
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(nameof(Value), typeof(double), typeof(Gauge),
            new FrameworkPropertyMetadata(0.0, FrameworkPropertyMetadataOptions.AffectsRender | FrameworkPropertyMetadataOptions.BindsTwoWayByDefault, OnValueChanged, CoerceValue));

    public double Minimum { get => (double)GetValue(MinimumProperty); set => SetValue(MinimumProperty, value); }
    public static readonly DependencyProperty MinimumProperty =
        DependencyProperty.Register(nameof(Minimum), typeof(double), typeof(Gauge),
            new FrameworkPropertyMetadata(0.0, FrameworkPropertyMetadataOptions.AffectsRender));

    public double Maximum { get => (double)GetValue(MaximumProperty); set => SetValue(MaximumProperty, value); }
    public static readonly DependencyProperty MaximumProperty =
        DependencyProperty.Register(nameof(Maximum), typeof(double), typeof(Gauge),
            new FrameworkPropertyMetadata(100.0, FrameworkPropertyMetadataOptions.AffectsRender));

    private static object CoerceValue(DependencyObject d, object baseValue)
    {
        var c = (Gauge)d;
        double v = (double)baseValue;
        return Math.Max(c.Minimum, Math.Min(c.Maximum, v));
    }
    private static void OnValueChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        // 필요 시 RoutedEvent 발생/명령 갱신 등
    }

    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        // var needle = GetTemplateChild("PART_Needle") as FrameworkElement;
        // 템플릿 파트 wiring
    }
}
```

```xml
<!-- Themes/Generic.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                    xmlns:local="clr-namespace:MyControlLibrary.Controls">
  <Style TargetType="{x:Type local:Gauge}">
    <Setter Property="Template">
      <Setter.Value>
        <ControlTemplate TargetType="{x:Type local:Gauge}">
          <Grid>
            <Ellipse Fill="{DynamicResource GaugeBackgroundBrush}" Stroke="#DDD"/>
            <!-- PART_Needle 등 템플릿 파트 -->
            <Line x:Name="PART_Needle" X1="0" Y1="0" X2="0" Y2="-40" Stroke="Red" StrokeThickness="3"
                  RenderTransformOrigin="0.5,0.9">
              <Line.RenderTransform>
                <RotateTransform Angle="0"/>
              </Line.RenderTransform>
            </Line>
          </Grid>
          <ControlTemplate.Triggers>
            <!-- VSM/트리거로 상태/값 표현 -->
          </ControlTemplate.Triggers>
        </ControlTemplate>
      </Setter.Value>
    </Setter>
  </Style>
</ResourceDictionary>
```

- 소비자는 **Style/Template만 바꿔서** 완전히 다른 외형을 만들 수 있습니다.
- 라이브러리 관점에서 **확장성·테마화**가 뛰어납니다.

### 3.3 `PART_*` 파트 계약
- 템플릿이 바뀌어도 **필수 요소를 찾을 수 있게** 관례적으로 `PART_` 접두사를 사용.
- 문서화: “템플릿에는 `PART_Needle`이 있어야 합니다.”
- `OnApplyTemplate()`에서 `GetTemplateChild("PART_Needle")`로 조회.

### 3.4 VisualStateManager(VSM)
- `CommonStates`(Normal/Disabled), `FocusStates` 등 상태를 템플릿에서 정의.
- **스타일/트리거 남발 대신 상태 전환**으로 관리해 가독성↑.

---

## 4. **같은 기능**을 UserControl ↔ CustomControl로 구현 비교

### 요구: “Avatar + Name + Busy 상태 표시” 컨트롤

#### 4.1 UserControl 버전(빠른 합성)
```xml
<UserControl ... x:Class="Demo.Controls.UserAvatar">
  <Grid>
    <Ellipse Width="36" Height="36">
      <Ellipse.Fill>
        <ImageBrush ImageSource="{Binding Photo, RelativeSource={RelativeSource AncestorType=UserControl}}"/>
      </Ellipse.Fill>
    </Ellipse>
    <Ellipse Width="10" Height="10" Fill="Lime" HorizontalAlignment="Right" VerticalAlignment="Bottom"
             Visibility="{Binding IsBusy, RelativeSource={RelativeSource AncestorType=UserControl}, Converter={StaticResource BoolToVisibility}}"/>
  </Grid>
</UserControl>
```

```csharp
public partial class UserAvatar : UserControl
{
    public ImageSource? Photo { get => (ImageSource?)GetValue(PhotoProperty); set => SetValue(PhotoProperty, value); }
    public static readonly DependencyProperty PhotoProperty =
        DependencyProperty.Register(nameof(Photo), typeof(ImageSource), typeof(UserAvatar));

    public bool IsBusy { get => (bool)GetValue(IsBusyProperty); set => SetValue(IsBusyProperty, value); }
    public static readonly DependencyProperty IsBusyProperty =
        DependencyProperty.Register(nameof(IsBusy), typeof(bool), typeof(UserAvatar));
}
```

- 외형을 둥근 대신 **사각형**으로 하고 싶다면? → **컨트롤 XAML 수정 필요**.

#### 4.2 CustomControl 버전(템플릿 교체 가능)
```csharp
public class Avatar : Control
{
    static Avatar()
    {
        DefaultStyleKeyProperty.OverrideMetadata(typeof(Avatar),
            new FrameworkPropertyMetadata(typeof(Avatar)));
    }

    public ImageSource? Photo { get => (ImageSource?)GetValue(PhotoProperty); set => SetValue(PhotoProperty, value); }
    public static readonly DependencyProperty PhotoProperty =
        DependencyProperty.Register(nameof(Photo), typeof(ImageSource), typeof(Avatar));

    public bool IsBusy { get => (bool)GetValue(IsBusyProperty); set => SetValue(IsBusyProperty, value); }
    public static readonly DependencyProperty IsBusyProperty =
        DependencyProperty.Register(nameof(IsBusy), typeof(bool), typeof(Avatar));

    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        // 필요 시 PART 연결
    }
}
```

```xml
<!-- Generic.xaml -->
<Style TargetType="{x:Type local:Avatar}">
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="{x:Type local:Avatar}">
        <Grid>
          <Ellipse Width="36" Height="36">
            <Ellipse.Fill>
              <ImageBrush ImageSource="{TemplateBinding Photo}"/>
            </Ellipse.Fill>
          </Ellipse>
          <Ellipse Width="10" Height="10" Fill="Lime" HorizontalAlignment="Right" VerticalAlignment="Bottom"
                   Visibility="{Binding IsBusy, RelativeSource={RelativeSource TemplatedParent}, Converter={StaticResource BoolToVisibility}}"/>
        </Grid>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```

- 소비자는 다른 프로젝트에서 **템플릿만 교체**해 사각·네온·라운드 등 **브랜딩** 자유.

---

## 5. 의존성 속성(DP)·라우티드 이벤트(RE)·명령(Command) 차이

| 항목 | UserControl | CustomControl |
|---|---|---|
| **DP** | 사용 가능(보통 바인딩 노출용) | **핵심 수단**(AffectsRender/Measure/Arrange 옵션, Coerce/Validate 포함) |
| **RoutedEvent** | 필요 시 사용 | **자주 사용**(템플릿 내부 이벤트를 외부로 승격, `AddOwner` 등) |
| **ICommand/RoutedCommand** | 내부 버튼에 직접 바인딩 | **CommandBinding**·키 제스처·InputBindings와 자연스럽게 결합 |

### 5.1 DP 고급 옵션(메타데이터)
```csharp
FrameworkPropertyMetadata meta = new(
    defaultValue: 0.0,
    flags: FrameworkPropertyMetadataOptions.AffectsRender | FrameworkPropertyMetadataOptions.BindsTwoWayByDefault,
    propertyChangedCallback: OnChanged,
    coerceValueCallback: Coerce
);
```
- **CustomControl** 쪽에서 자주 사용: **렌더 영향**, **강제 범위**, **기본 바인딩 모드** 등.

---

## 6. 테마/다크모드·리소스 구조

- **UserControl**: 컨트롤 내부 색·여백을 **직접 바꿔야** 함 → 대규모 테마 전환이 힘듦.
- **CustomControl**: **DynamicResource + MergedDictionaries** 로 테마 스와핑.

```xml
<!-- Generic.xaml에서 DynamicResource 사용 -->
<Border Background="{DynamicResource Palette.Card.Background}"
        BorderBrush="{DynamicResource Palette.Card.Border}"/>
```

```csharp
// 전역 테마 전환
Application.Current.Resources.MergedDictionaries[0] = new ResourceDictionary { Source = new Uri("Themes/Colors.Dark.xaml", UriKind.Relative) };
```

---

## 7. 성능 관점

- **UserControl**: 내부 시각 트리가 고정 → 작은 합성에 **경량**.
- **CustomControl**: 템플릿/트리거/VSM/바인딩 많아질수록 **유연성 대가**가 듦.
- 공통 최적화:
  - 템플릿 내부는 **`TemplateBinding`** 우선(경량 바인딩)
  - **Freezable 리소스** 사용(Brush/Geometry는 Freeze)
  - 가상화 컨트롤(`ItemsControl`)은 **`ItemsPresenter`** 유지
  - 애니메이션 최소화/단일 브러시 공유 주의(`x:Shared="False"` 고려)

---

## 8. 디자인-타임(Design-time) 지원

- **UserControl**: 바로 미리보기 OK.
- **CustomControl**:
  - `DesignInstance`, `DesignWidth/Height`
  - 샘플 데이터(`d:DataContext`)
  - `DefaultStyleKey` 설정 + **디자인 전용 템플릿** 제공 가능.

```xml
<!-- Generic.xaml에서 디자인용 기본 컨텐츠를 넣어 시각 확인 -->
<TextBlock Text="Avatar" d:Text="Preview Avatar"/>
```

---

## 9. 접근성(A11y)·자동화(UIA)

- **CustomControl** 제작 시 **`AutomationPeer`** 를 제공하면 스크린리더·UI 테스트 자동화 대응이 좋습니다.
- **UserControl** 은 내부 기본 컨트롤들의 Automation을 **상속**.

```csharp
protected override AutomationPeer OnCreateAutomationPeer()
    => new GaugeAutomationPeer(this);
```

---

## 10. 단위 테스트/리그레션

- **UserControl**: ViewModel과 분리해 **UI 없는 단위 테스트**는 쉽지 않음(대개 상호작용 테스트는 UI 자동화 도구).
- **CustomControl**: **DP/상태/코어 로직**을 별도 클래스로 분리하면 **순수 단위 테스트** 가능.
- Snapshot(픽셀) 테스트로 템플릿 변화 감시.

---

## 11. 배포·재사용

- **UserControl**: 보통 **앱 프로젝트 내부**에서만 사용. 다른 앱으로 복사/붙여넣기.
- **CustomControl**: **별도 어셈블리**(NuGet 배포)에 최적, 테마/로컬라이제이션/리소스 포함.

---

## 12. 실무 선택 가이드(요약)

- **빠르게 합성하고 팀 내부에서만** 쓰는 **화면 조각** → **UserControl**
- **브랜딩/테마/리디자인** 요구가 있고 **외부에도 배포**할 **재사용 가능한 컨트롤** → **CustomControl**
- 초기에 UserControl로 시작했다가 요구가 커지면 **CustomControl로 리팩터링**(아래 예시)

---

## 13. 마이그레이션: UserControl → CustomControl

### 13.1 리팩터링 전략
1) UserControl의 **public DP/이벤트** 목록 정리
2) 내부 XAML에서 **“외형”만 Template로 이식**
3) **`Control` 파생** 클래스로 속성/로직 이동
4) 내부 요소 접근은 **`PART_*`** 로 치환 + `OnApplyTemplate()`에서 wiring
5) **Generic.xaml**에 기본 템플릿 배치
6) 외부 사용처는 `<local:OldUserControl .../>` → `<lib:NewCustomControl .../>` 로 교체
7) 테마/스타일 문서화

### 13.2 간단 예
- `ProfileCard`(UserControl) → `Profile`(CustomControl)로 바꾸면서 템플릿 교체 가능하도록.

```csharp
public class Profile : Control
{
    static Profile()
    {
        DefaultStyleKeyProperty.OverrideMetadata(typeof(Profile), new FrameworkPropertyMetadata(typeof(Profile)));
    }

    public ImageSource? Photo { get => (ImageSource?)GetValue(PhotoProperty); set => SetValue(PhotoProperty, value); }
    public static readonly DependencyProperty PhotoProperty =
        DependencyProperty.Register(nameof(Photo), typeof(ImageSource), typeof(Profile));

    public string? Name { get => (string?)GetValue(NameProperty); set => SetValue(NameProperty, value); }
    public static readonly DependencyProperty NameProperty =
        DependencyProperty.Register(nameof(Name), typeof(string), typeof(Profile));

    public string? Title { get => (string?)GetValue(TitleProperty); set => SetValue(TitleProperty, value); }
    public static readonly DependencyProperty TitleProperty =
        DependencyProperty.Register(nameof(Title), typeof(string), typeof(Profile));
}
```

```xml
<!-- Generic.xaml -->
<Style TargetType="{x:Type local:Profile}">
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="{x:Type local:Profile}">
        <Border CornerRadius="10" Padding="12" Background="{DynamicResource CardBg}" BorderBrush="{DynamicResource CardBorder}" BorderThickness="1">
          <StackPanel Orientation="Horizontal" Spacing="12">
            <Ellipse Width="48" Height="48">
              <Ellipse.Fill>
                <ImageBrush ImageSource="{TemplateBinding Photo}"/>
              </Ellipse.Fill>
            </Ellipse>
            <StackPanel>
              <TextBlock Text="{TemplateBinding Name}" FontSize="16" FontWeight="SemiBold"/>
              <TextBlock Text="{TemplateBinding Title}" Foreground="#666"/>
            </StackPanel>
          </StackPanel>
        </Border>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```

- 이후 소비자는 **사각형 카드 템플릿**·**네온 스타일** 등 마음대로 교체 가능.

---

## 14. 흔한 함정·트러블슈팅

1) **Generic.xaml 누락** → CustomControl가 흰 사각형만 보임
   - `Themes/Generic.xaml` 파일명/빌드 액션(Resource) 확인, `DefaultStyleKey` 메타데이터 확인.
2) **`PART_*` 이름 불일치** → 기능 미동작
   - 템플릿 문서화/테스트 필수.
3) **UserControl에서 외형 커스터마이즈 요구 증가**
   - 스타일/템플릿으로는 한계 → **CustomControl로 전환** 고려.
4) **DP 기본값/Coerce/Validate 누락**
   - 값 범위/기본 동작 깨짐 → 메타데이터/콜백 추가.
5) **StaticResource로 테마 키 박음**
   - 테마 전환 미반영 → **DynamicResource**로 변경.
6) **ItemsControl 재템플릿에서 `ItemsPresenter` 누락**
   - 아이템 출력 안 됨/가상화 깨짐 → 템플릿 규칙 지키기.
7) **디자인-타임 Null Data**
   - `d:` 네임스페이스 샘플 값 제공.

---

## 15. 제작 체크리스트(현업용)

### UserControl
- [ ] 외형 고정으로 충분한가? (브랜딩 요구 없음)
- [ ] 공개 DP/이벤트만으로 조작 가능한가?
- [ ] 내부에서 다른 컨트롤/뷰모델 합성이 핵심인가?

### CustomControl
- [ ] `Control` 파생 + `DefaultStyleKey` 메타데이터
- [ ] **Generic.xaml** 기본 Style/Template
- [ ] **DP**: AffectsMeasure/Render/Arrange, Coerce/Validate
- [ ] **RoutedEvent/Command** 필요 여부
- [ ] **PART** 문서화 + OnApplyTemplate wiring
- [ ] **VSM** 상태 설계(Disabled/Focused/Pressed 등)
- [ ] **DynamicResource**로 테마 대응
- [ ] **AutomationPeer** (접근성)
- [ ] 성능: TemplateBinding·Freeze·가상화·바인딩 최소화

---

## 16. FAQ

**Q. CustomControl이 항상 더 “좋은”가요?**
A. 목적이 다릅니다. **빠른 화면 합성/내부 재사용**은 UserControl이 생산적입니다.
**브랜딩/재배포/스킨 교체**가 핵심이면 CustomControl이 정답입니다.

**Q. UserControl에서 외형 바꾸려면?**
A. 대부분 **컨트롤 소스 수정**이 필요합니다. 외부 Style로는 내부 구조를 갈아엎기 어렵습니다.

**Q. 라이브러리로 배포하려는데 UserControl로 가능?**
A. 기술적으로 가능하지만, **소비자 측 커스터마이즈 범위가 협소**합니다. 라이브러리는 CustomControl 권장.

**Q. 디자인-타임 미리보기는?**
A. UserControl이 더 단순하지만, CustomControl도 샘플 템플릿/디자인 데이터로 충분히 좋게 만들 수 있습니다.

---

## 17. 결론

- **UserControl**: “**합성 View**” — 빠르고 직관적. 내부 UI가 바뀌지 않는 **사내 화면 조각**에 최적.
- **CustomControl**: “**스킨 가능한 컨트롤**” — 템플릿/스타일/테마/브랜딩/배포에 최적.
- 초기에 **요구 불확실**하면 UserControl로 시작해도 되지만,
  **외형 커스터마이즈·브랜딩** 요구가 생기는 순간 **CustomControl로 리팩터링**하는 것이 장기 유지보수에 유리합니다.
