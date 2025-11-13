---
layout: post
title: WPF - XAML–CLR–렌더링 파이프라인
date: 2025-08-27 17:25:23 +0900
category: WPF
---
# XAML–CLR–렌더링 파이프라인 관계 완전 해부

> 이 글은 **XAML → CLR 객체 그래프 → 렌더링(컴포지션)**으로 이어지는 **WPF 파이프라인**을 *빌드 타임*부터 *런타임*, *합성/출력*까지 **끝까지 추적**합니다.
> “XAML이 어떻게 BAML로 바뀌고, `InitializeComponent()`가 무엇을 하고, 바인딩/레이아웃/비주얼 기록이 어떤 스레드를 지나 GPU로 내려가 픽셀로 찍히는가?”를 **코드·도표·체크리스트**로 정리합니다.
> 코드 블록은 모두 \`\`\`로 감쌌고, 전체 글은 블로그 포맷에 맞춰 \~~~markdown 으로 감쌌습니다.

---

## 0. 한 장 요약 (TL;DR)

1. **빌드 타임(Compile)**
   - `*.xaml` → **BAML**(이진 XAML) + `*.g.cs`(부분 클래스) 생성
   - `InitializeComponent()`가 리소스 어셈블리에서 BAML을 읽어 **CLR 객체 그래프**를 만든다.

2. **런타임-UI 스레드**
   - BAML 로더가 **DependencyObject/FrameworkElement**들을 **객체화**(Object Graph Materialization)
   - **리소스/스타일/템플릿** 적용 → **바인딩/트리거/애니메이션** 구동 → **Measure/Arrange** 레이아웃
   - `OnRender(DrawingContext)` 등에서 **비주얼 명령** 기록(장면 그래프).

3. **렌더/컴포지션 스레드(milcore)**
   - UI 스레드가 기록한 장면을 **DUCE 채널**로 배치 전송
   - **컴포지터 스레드**가 이를 읽어 **Direct3D**로 합성(테ク스처/지오메트리/알파 블렌딩)
   - **스왑 체인**이 VSync에 맞춰 화면에 픽셀을 출력.

---

## 1. 큰 그림: 데이터·제어·렌더 플로우

```
[ XAML 파일 ]
   │ (XAML 컴파일: Build Task)
   ↓
[ BAML + .g.cs (InitializeComponent) ]
   │ (런타임: InitializeComponent 실행)
   ↓
[ CLR 객체 그래프 (DO/FE/Controls) ]
   │   ├─ 리소스/스타일/템플릿 병합
   │   ├─ 바인딩/트리거/애니메이션 초기화
   │   └─ 라우티드 이벤트/명령 Hook
   ↓ (Measure/Arrange)
[ 논리 트리 & 비주얼 트리 구축 ]
   │ (OnRender/DrawingContext 등)
   ↓
[ 장면 그래프 업데이트 (UI 스레드) ]
   │ (DUCE 채널로 배치 전송)
   ↓
[ milcore/Compositor 스레드 ]
   │ (D3D 리소스 관리/합성)
   ↓
[ GPU → 프레임버퍼 → 화면 픽셀 ]
```

---

## 2. 빌드 타임: XAML → BAML, 그리고 .g.cs

WPF SDK/MSBuild는 XAML을 **BAML**로 컴파일합니다. BAML은 **이진 토큰화된 XAML**로, 런타임 파싱 비용을 낮추고 타입/멤버 조회를 빠르게 합니다. 동시에 `*.g.cs`가 생성되어 `InitializeComponent()`가 BAML을 로드하게 됩니다.

### 2.1 예시 프로젝트 구조

```
/Views/MainWindow.xaml
/Views/MainWindow.xaml.cs
/obj/Debug/net8.0-windows/Views/MainWindow.g.cs   ← 생성
/obj/.../g/.../MainWindow.baml                    ← 생성(리소스에 포함)
```

### 2.2 생성되는 `MainWindow.g.cs` (요약 예시)

```csharp
// 자동 생성된 코드 (요약 예시)
public partial class MainWindow : System.Windows.Window, System.Windows.Markup.IComponentConnector
{
    public void InitializeComponent()
    {
        if (_contentLoaded) return;
        _contentLoaded = true;
        System.Uri resourceLocater = new System.Uri("/MyApp;component/views/mainwindow.xaml", UriKind.Relative);
        System.Windows.Application.LoadComponent(this, resourceLocater);
    }

    // IComponentConnector.Connect … (NameScope 연결, x:Name 필드 바인딩 등)
}
```

- 핵심은 **`Application.LoadComponent(this, uri)`**가 리소스에서 **BAML**을 찾아 로딩한다는 점입니다.

---

## 3. 런타임 1: `InitializeComponent()`가 하는 일

`InitializeComponent()` 호출 시(대개 생성자에서), 다음이 일어납니다.

1. **BAML 스트림 로드** → 파서가 토큰을 순회하며 **객체 생성**
2. 생성된 객체에 **속성 주입**(의존 속성 포함), **이벤트 연결**, **x:Name → NameScope 등록**
3. **ResourceDictionary 병합**, **StaticResource/DynamicResource** 해석
4. **스타일/템플릿**을 요소에 적용(Style Hierarchy)
5. **마크업 확장**(예: `{Binding}`, `{StaticResource}`)을 해석 → **실제 객체/식**으로 치환
6. `Loaded` 이전에 바인딩/트리거 등 **초기 평가** 수행 → 초기 레이아웃 트리 준비

### 3.1 간단 XAML 예시

```xml
<Window x:Class="MyApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="XAML→CLR→Render" Width="900" Height="600">
    <Window.Resources>
        <SolidColorBrush x:Key="Accent" Color="#4F7CAC"/>
    </Window.Resources>

    <Grid>
        <TextBox x:Name="SearchBox" Margin="8"
                 Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"/>
        <ListBox Margin="8,48,8,8" ItemsSource="{Binding People}">
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <StackPanel Orientation="Horizontal">
                        <Ellipse Width="8" Height="8" Fill="{StaticResource Accent}" Margin="0,0,6,0"/>
                        <TextBlock Text="{Binding Name}" FontWeight="SemiBold"/>
                        <TextBlock Text="{Binding Title}" Foreground="Gray" Margin="8,0,0,0"/>
                    </StackPanel>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>
    </Grid>
</Window>
```

### 3.2 BAML 파서가 만드는 것(개념적)

- `Window` → `Grid` → `TextBox`, `ListBox` → `DataTemplate` 등 **CLR 객체**가 생성
- 의존 속성 세팅 (`TextBox.Text`, `FrameworkElement.Margin`, `Control.Template` 등)
- `{Binding ...}` → **Binding 객체**/BindingExpression 생성 대기
- `x:Name="SearchBox"` → **NameScope**에 등록 → `.g.cs`의 `IComponentConnector`로 필드 연결

---

## 4. 런타임 2: 데이터 바인딩/트리거/애니메이션 초기화

### 4.1 바인딩 초기화

BAML 로딩 과정에서 `{Binding}` 마크업 확장은 **`Binding` 객체**로 인스턴스화되고, **타깃 DP(의존 속성)**에 **BindingExpression**이 부착됩니다. 이후 **DataContext** 전파에 따라 **소스 해석 → 값 전파**가 일어나죠.

```csharp
public sealed class MainViewModel : INotifyPropertyChanged
{
    private string _searchText = "";
    public string SearchText { get => _searchText; set { _searchText = value; OnPropertyChanged(); } }

    public ObservableCollection<Person> People { get; } = new();

    // INotifyPropertyChanged 구현 생략
}
```

```csharp
// 코드비하인드(또는 App 시작 시점)에서 DataContext 주입
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        DataContext = new MainViewModel(); // ← 여기서 바인딩이 활성화
    }
}
```

**포인트**
- `{Binding SearchText}`는 `DataContext`가 설정되는 순간 **소스 경로를 확인**하고 `TextBox.TextProperty`로 값 반영.
- 값 변경 시 **INPC** → 바인딩 엔진 → DP 값 갱신 → **Measure/Arrange/Render 무효화**까지 이어질 수 있음.

### 4.2 트리거/애니메이션

스타일 트리거나 Storyboard는 **DP 레벨의 값 오버레이**로 동작합니다. 런타임 초기화 후, 상태 변화(예: MouseOver) 시 **애니메이션 클록**이 DP에 **시간 의존적 값**을 공급합니다.

```xml
<Button Content="Play" Width="140" Height="36">
    <Button.Style>
        <Style TargetType="Button">
            <Setter Property="Background" Value="SlateBlue"/>
            <Style.Triggers>
                <Trigger Property="IsMouseOver" Value="True">
                    <Setter Property="Background" Value="RoyalBlue"/>
                </Trigger>
            </Style.Triggers>
        </Style>
    </Button.Style>
</Button>
```

---

## 5. 레이아웃 파이프라인: **Measure → Arrange → Render**

### 5.1 트리거: 무엇이 레이아웃을 다시 돌릴까?

- **사이즈/마진/폰트** 등 레이아웃 관련 DP 변경
- **바인딩 값 변경**으로 콘텐츠 크기가 달라짐
- **템플릿/스타일** 변화

### 5.2 Measure/Arrange 개요

- **Measure**: 자식이 필요한 **크기 요청** 계산(Available Size 입력 → Desired Size 출력)
- **Arrange**: 자식의 **최종 배치** 확정(Rect)
- 두 단계 후에 렌더 무효화 영역이 파생되어 **렌더 파이프라인**에 반영된다.

```csharp
public class FixedGrid : Panel
{
    public int Columns { get; set; } = 3;
    public double CellWidth { get; set; } = 150;
    public double CellHeight { get; set; } = 40;

    protected override Size MeasureOverride(Size constraint)
    {
        var cell = new Size(CellWidth, CellHeight);
        foreach (UIElement child in InternalChildren) child.Measure(cell);
        int rows = (int)Math.Ceiling((double)InternalChildren.Count / Columns);
        return new Size(Columns * CellWidth, rows * CellHeight);
    }

    protected override Size ArrangeOverride(Size arrangeSize)
    {
        for (int i = 0; i < InternalChildren.Count; i++)
        {
            int r = i / Columns, c = i % Columns;
            var rect = new Rect(c * CellWidth, r * CellHeight, CellWidth, CellHeight);
            InternalChildren[i].Arrange(rect);
        }
        return arrangeSize;
    }
}
```

---

## 6. 비주얼 트리와 렌더 레코딩(Recording)

### 6.1 Visual/DrawingContext

**`UIElement`**는 **`Visual`**을 상속합니다. 실제 그리기는 `OnRender(DrawingContext)`에서 수행하며, 이는 **장면 그래프**에 **드로잉 명령**을 기록합니다(보존(retained) 모드).

```csharp
public class MeterBar : FrameworkElement
{
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(nameof(Value), typeof(double), typeof(MeterBar),
           new FrameworkPropertyMetadata(0.0, FrameworkPropertyMetadataOptions.AffectsRender));

    public double Value { get => (double)GetValue(ValueProperty); set => SetValue(ValueProperty, value); }

    protected override void OnRender(DrawingContext dc)
    {
        var rect = new Rect(0, 0, ActualWidth, ActualHeight);
        dc.DrawRectangle(Brushes.DimGray, null, rect);

        double w = rect.Width * Math.Max(0, Math.Min(1, Value));
        dc.DrawRectangle(Brushes.LimeGreen, null, new Rect(0, 0, w, rect.Height));
    }
}
```

- `AffectsRender` 메타 옵션 덕분에 `Value` 변경 시 **렌더 무효화**가 발생 → 다음 합성 프레임에 반영.

### 6.2 Retained vs Immediate

- **Retained Mode**(WPF): 개체 그래프/드로잉 명령을 유지하여 **부분 갱신**, **시스템 주도 합성**
- **Immediate Mode**(GDI+/Direct2D 단독): 호출자가 매 프레임 직접 모든 픽셀을 갱신

---

## 7. UI 스레드 → 컴포지터(milcore) 스레드: DUCE 채널

### 7.1 왜 채널이 필요한가

- UI 스레드는 **객체 그래프/DP/레이아웃**을 관리
- **렌더링/합성**은 **네이티브 milcore + D3D**에서 **별도 스레드**로 수행
- 두 세계를 **저비용 배치 스트림(DUCE)**으로 연결

### 7.2 흐름

1. UI 스레드에서 **장면 그래프 변경**(비주얼, 지오메트리, 브러시, 텍스처 등)
2. 변경점이 **명령 버퍼**로 효율적으로 직렬화(DUCE)
3. 컴포지터 스레드가 버퍼를 읽어 **D3D 리소스/상태**를 업데이트
4. 다음 **합성 틱**에서 화면에 반영

---

## 8. 텍스트 파이프라인: TextFormatter → GlyphRun → 합성

- **TextFormatter**가 줄바꿈/스크립트/폰트 폴백/힌팅을 처리해 **GlyphRun**을 생성
- `DrawText`/`DrawGlyphRun` 호출로 비주얼에 텍스트가 기록
- 컴포지터는 글리프 비트맵/서브픽셀 포지셔닝을 고려해 합성

```csharp
protected override void OnRender(DrawingContext dc)
{
    var ft = new FormattedText(
        "XAML→CLR→Render", System.Globalization.CultureInfo.CurrentUICulture,
        FlowDirection.LeftToRight, new Typeface("Segoe UI"),
        24, Brushes.White, VisualTreeHelper.GetDpi(this).PixelsPerDip);

    dc.DrawText(ft, new Point(8, 8));
}
```

---

## 9. 애니메이션: 타임라인/클록 → DP 오버레이 → 합성

애니메이션은 DP에 **시간 함수**를 겹쳐 씌웁니다(값 우선순위 스택에서 애니메이션 레벨이 존재). 프레임마다 **클록**이 진행되어 DP의 유효 값이 바뀌고, **렌더 무효화**를 유발합니다.

```csharp
var da = new DoubleAnimation(0, 1, TimeSpan.FromSeconds(1.2))
{
    AutoReverse = true,
    RepeatBehavior = RepeatBehavior.Forever
};
myMeterBar.BeginAnimation(MeterBar.ValueProperty, da);
```

---

## 10. DP 값 우선순위와 무효화

**DependencyProperty**의 유효 값은 다음의 우선순위 스택에서 결정됩니다(요약):

1. **Animation**
2. **Local Value** (코드로 SetValue, XAML 속성 할당)
3. **Style/Template Setter**
4. **Theme Style**
5. **Inherited** (예: `FontFamily` 상속)
6. **Default Value**

값이 바뀌면 **PropertyMetadata**의 플래그에 따라 **AffectsMeasure/AffectsArrange/AffectsRender** 가동 → 알맞은 파이프라인 무효화가 발생.

---

## 11. 입력/라우티드 이벤트 → 비주얼 HitTest → 상태 갱신

입력은 **프리뷰(터널링)** → **버블링** 경로로 라우팅되며, **Visual Tree HitTest** 결과를 이용합니다. 이벤트 핸들러에서 DP를 바꾸면 **레이아웃/렌더**가 이어집니다.

```csharp
protected override void OnMouseEnter(MouseEventArgs e)
{
    BeginAnimation(MeterBar.ValueProperty, new DoubleAnimation(1, TimeSpan.FromMilliseconds(200)));
}
```

---

## 12. Dispatcher & 프레임 타이밍

- WPF UI는 **STA Dispatcher 루프** 위에서 동작
- **DispatcherPriority**로 작업 순서를 제어(입력/렌더 우선)
- **CompositionTarget.Rendering** 이벤트는 **합성과 동기**된 콜백(프레임별 업데이트에 유용)

```csharp
CompositionTarget.Rendering += (_, __) =>
{
    // 프레임 동기 루프(게임/시뮬레이션/차트)에서 값 갱신
};
```

---

## 13. Per-Monitor DPI, DIP 좌표, 픽셀 스냅

- 좌표계는 **DIP(1/96")** 기준. 모니터 DPI에 따라 실제 픽셀 수가 다름
- 텍스트/선명도를 위해 `UseLayoutRounding="True"`, `SnapsToDevicePixels="True"` 권장

```xml
<Grid UseLayoutRounding="True" SnapsToDevicePixels="True">
    <TextBlock Text="Sharp Text" />
</Grid>
```

---

## 14. 사례 연구 A: “XAML 한 줄 → 픽셀 한 줄”을 끝까지 따라가기

### 14.1 XAML

```xml
<TextBlock Text="Hello WPF" FontSize="24" Foreground="White" Margin="8"/>
```

### 14.2 BAML 로드

- BAML 파서 → `TextBlock` 인스턴스 생성 → DP 할당(FontSize/Foreground/Margin/Text)

### 14.3 레이아웃

- 부모 `Panel`이 Measure 호출 → `TextBlock`이 `TextFormatter`를 통해 `DesiredSize` 계산
- Arrange에서 최종 배치 확정(Rect)

### 14.4 렌더 기록

- `OnRender` 내부적으로 `DrawText(FormattedText)` 호출 → **GlyphRun** 생성/기록

### 14.5 채널 전송

- 렌더 변경점이 DUCE로 배치 → 컴포지터가 D3D 텍스처/버퍼에 글리프 배치

### 14.6 화면 출력

- VSync 시 스왑체인 Present → 모니터 픽셀로 “Hello WPF”가 나타남

---

## 15. 사례 연구 B: 바인딩이 렌더에 미치는 연쇄

1. 사용자가 `TextBox`에 입력 → `Text` DP 변경
2. 바인딩이 ViewModel의 `SearchText` 변경 → INPC 발생
3. `People` 필터/정렬로 `ItemsSource` 변화 → 항목 가시성/수량 변경
4. `ItemsControl` 레이아웃 무효화(Measure/Arrange) → 비주얼 변경
5. 렌더 레코딩 업데이트 → DUCE → 합성 → 화면

**핵심:** 바인딩은 **데이터 변화**를 **시각 변화**로 연결하는 **촉매**이며, 그 결과 **레이아웃/렌더 파이프라인**이 다시 돈다.

---

## 16. 스레드 규칙: UI 스레드 · 컴포지터 스레드 · Freezable

- **UI 스레드**: DO/FE 소유, 대부분의 DP 접근은 **UI 스레드 한정**
- **컴포지터 스레드**: milcore/D3D 합성 담당
- **Freezable**: `Freeze()`하면 **읽기 전용 + 스레드 간 공유 가능** (브러시/지오메트리/트랜스폼 등)

```csharp
var brush = new LinearGradientBrush(Colors.CadetBlue, Colors.DarkSlateBlue, 0);
if (brush.CanFreeze) brush.Freeze(); // 렌더/스레드에 유리
```

---

## 17. 이미지/미디어: BitmapSource/MediaElement의 파이프라인

- `BitmapImage`/`WriteableBitmap` → 디코딩/픽셀 버퍼 → 텍스처 업로드
- `MediaElement`/`MediaPlayer` → 프레임 디코딩 → 텍스처 형태로 합성

```xml
<Image Source="pack://application:,,,/Assets/Hero.jpg"
       DecodePixelWidth="1200" Stretch="UniformToFill"/>
```

> `DecodePixelWidth/Height`로 디코딩 단계에서 다운샘플 → **메모리/대역폭 절감** → 합성 비용 감소.

---

## 18. 고급: VisualLayer(컨트롤 패스 우회)로 초경량 렌더

컨트롤/바인딩/템플릿 오버헤드 없이 **DrawingVisual**만으로 장면을 구성하면 **대규모 데이터 시각화**가 가능.

```csharp
public sealed class VisualLayerCanvas : FrameworkElement
{
    private readonly VisualCollection _visuals;
    public VisualLayerCanvas() => _visuals = new VisualCollection(this);

    protected override int VisualChildrenCount => _visuals.Count;
    protected override Visual GetVisualChild(int index) => _visuals[index];

    public void DrawRect(Rect rect, Brush fill)
    {
        var dv = new DrawingVisual();
        using (var dc = dv.RenderOpen())
            dc.DrawRectangle(fill, null, rect);
        if (fill is Freezable f && f.CanFreeze) f.Freeze();
        _visuals.Add(dv);
    }
}
```

- UI 스레드에서 **명령 수 최소화** → DUCE 전송량도 감소 → 합성 비용 안정화.

---

## 19. 진단과 최적화 체크리스트

1. **바인딩 에러 로그**(출력 창) 제거: 오버헤드/Null 경로
2. **Freeze 가능한 리소스 Freeze**: 공유/불변
3. **Layout Thrash 방지**: 빈번한 Measure/Arrange 루프 회피
4. **효과/Opacity 남용 금지**: 합성 경로 비용 큼(특히 중첩 반투명)
5. **대형 비트맵 스케일** 줄이기: `DecodePixelWidth/Height`
6. **VirtualizingPanel** 활성화: 항목 가상화
7. **Pixel Snap/LayoutRounding**로 텍스트 선명도 확보
8. **CompositionTarget.Rendering** 오용 주의: 프레임당 과도한 작업 금지
9. **RenderCapability.Tier** 확인 후 전략 분기

```csharp
var tier = (RenderCapability.Tier >> 16); // 0/1/2
Debug.WriteLine($"Render Tier: {tier}");
```

---

## 20. 미세 타이밍: 한 프레임에서 실제로 벌어지는 일

**프레임 n**에서, 대략 다음 순서(단순화):

1. **입력 처리**(Dispatcher Input) → 라우티드 이벤트
2. **App 코드 실행**(INPC/바인딩 리액션 포함)
3. **레이아웃**(Measure/Arrange, Dirty 영역 계산)
4. **렌더 업데이트**(OnRender/드로잉 명령 기록)
5. **DUCE 배치 플러시**(장면 변경점 전송)
6. **컴포지터 틱**(milcore, D3D 상태/리소스 업데이트)
7. **Present**(VSync/스왑)

프레임 예산(예: 16.6ms @60Hz) 내에 1~5가 끝나야 6에서 부드럽게 합성됩니다.

---

## 21. FAQ (자주 받는 질문)

**Q1. XAML을 런타임에 직접 파싱할 수도 있나요?**
A1. 네, `XamlReader.Load`로 문자열 XAML을 동적 로드 가능. 하지만 **BAML(컴파일)** 경로가 훨씬 빠릅니다.

**Q2. `OnRender` vs `ControlTemplate` 중 무엇이 좋나요?**
A2. **스킨/Lookless**가 목적이면 Template, **대량·경량 렌더**가 목적이면 OnRender/VisualLayer. 용도에 따라 병행.

**Q3. 애니메이션이 끊기는 이유는?**
A3. 대개 **UI 스레드 과부하**(레이아웃 스톰/비트맵 디코딩/동기 IO). 클록은 따라가지만 **새 장면 기록**이 늦으면 끊깁니다.

**Q4. 텍스트가 흐릿해요.**
A4. DIP/픽셀 경계 정렬(정수 좌표), `UseLayoutRounding`, `SnapsToDevicePixels`. 확대 변환(ScaleTransform) 최소화.

---

## 22. 실전 종합 예제 — “End-to-End 미니 앱”

### 22.1 ViewModel

```csharp
public sealed class Person { public string Name { get; init; } = ""; public string Title { get; init; } = ""; }

public sealed class MainViewModel : INotifyPropertyChanged
{
    private string _searchText = "";
    public string SearchText { get => _searchText; set { _searchText = value; OnPropertyChanged(); ApplyFilter(); } }

    public ObservableCollection<Person> All { get; } = new();
    public ObservableCollection<Person> View { get; } = new();

    public MainViewModel()
    {
        All.Add(new Person { Name="Alice", Title="Engineer" });
        All.Add(new Person { Name="Bob", Title="Designer" });
        All.Add(new Person { Name="Carol", Title="Manager" });
        ApplyFilter();
    }

    void ApplyFilter()
    {
        View.Clear();
        foreach (var p in All.Where(p => p.Name.Contains(SearchText, StringComparison.OrdinalIgnoreCase)))
            View.Add(p);
    }

    // INPC 구현 생략
}
```

### 22.2 View(XAML)

```xml
<Window x:Class="MyApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:local="clr-namespace:MyApp.Views"
        Title="Pipeline Demo" Width="900" Height="600"
        UseLayoutRounding="True" SnapsToDevicePixels="True">
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition/>
        </Grid.RowDefinitions>

        <TextBox Grid.Row="0" Margin="8"
                 Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"
                 PlaceholderText="Type name to filter..."/>

        <ListBox Grid.Row="1" Margin="8" ItemsSource="{Binding View}">
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <DockPanel LastChildFill="True" Margin="0,2">
                        <local:MeterBar Width="80" Height="10" Value="{Binding Name.Length, Converter={StaticResource NormalizeLength}}"
                                        Margin="0,0,12,0"/>
                        <TextBlock Text="{Binding Name}" FontWeight="SemiBold"/>
                        <TextBlock Text="{Binding Title}" Foreground="Gray" Margin="8,0,0,0"/>
                    </DockPanel>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>
    </Grid>
</Window>
```

> 여기서 `MeterBar`는 **OnRender 기반 커스텀**으로, 바인딩 값 변화가 DP→렌더 무효화→DUCE→합성으로 이어짐.
> `SearchText` 변경은 `View` 갱신 → `ItemsControl` 레이아웃/렌더 재평가.

### 22.3 코드비하인드 시작

```csharp
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        DataContext = new MainViewModel();
    }
}
```

**이 한 앱 속에서 발생하는 파이프라인**
- XAML(BAML) → 객체화 → 바인딩 연결 → INPC로 데이터/뷰 갱신 → 레이아웃 → 렌더 기록 → 채널 전송 → 합성 → 픽셀.

---

## 23. “끊김 없이” 만들려면? (성능 전략 요약)

- **Freeze** 가능한 건 Freeze(브러시/지오메트리/트랜스폼)
- **가상화**: `VirtualizingStackPanel`, 지연 로딩
- **디코딩 크기** 지정: `DecodePixelWidth/Height`
- **바인딩/INPC 폭주** 제어: 배치 업데이트, 스로틀/디바운스
- **UI 스레드 작업 최소화**: 파일 IO/CPU는 `Task.Run`/`async`로
- **투명/이펙트 최소화**: 오버드로/블렌딩 비용 큼
- **픽셀 스냅**: 텍스트 선명도/과도한 재컴포지션 방지

---

## 24. 마무리: XAML–CLR–렌더링은 “계약”이다

- **XAML**은 **선언**(무엇을 보여줄지)
- **CLR**은 **행동/상태**(어떻게 변할지)
- **렌더링**은 **효율적 합성**(얼마나 부드럽게 그릴지)

이 셋의 **계약을 지키는 설계**(Freeze, 가상화, 적절한 바인딩, 알맞은 레이아웃)는 곧 **프레임 안정성**과 **선명한 출력**으로 이어집니다.
“XAML 한 줄 → 픽셀 한 줄”의 연결고리를 이해하면, WPF는 여전히 **가장 강력한 데스크톱 UI 파이프라인** 중 하나입니다.

---

### 부록 A) `InitializeComponent()` 디버깅 팁

- **출력 창**의 바인딩 에러 확인
- `PresentationTraceSources.TraceLevel=High`로 바인딩 추적
- `Loaded` 시점에 `VisualTreeHelper`로 트리 검사
- `RenderOptions.ProcessRenderMode`로 강제 소프트웨어 경로(비교 진단)

```xml
<TextBlock Text="{Binding Name, PresentationTraceSources.TraceLevel=High}"/>
```

---

### 부록 B) 텍스트 선명도 체크 스니펫

```csharp
var dpi = VisualTreeHelper.GetDpi(this);
Debug.WriteLine($"DPI: {dpi.PixelsPerDip}, Scale: {dpi.DpiScaleX}x{dpi.DpiScaleY}");
```

---

### 부록 C) Frame-Sync 루프에서의 주의

```csharp
CompositionTarget.Rendering += (_, __) =>
{
    // (1) 작은 상태 업데이트만
    // (2) 무거운 연산은 Task.Run + 결과만 UI에 반영
    // (3) 새 Geometry/Brush는 Freeze 후 재사용
};
```

---

### 부록 D) 값 우선순위 데모

```csharp
// Local value vs Animation vs Style
btn.SetValue(Button.BackgroundProperty, Brushes.Orange); // Local
var anim = new ColorAnimation(Colors.Orange, Colors.Red, TimeSpan.FromSeconds(2));
var brush = new SolidColorBrush(Colors.Orange);
btn.Background = brush;
brush.BeginAnimation(SolidColorBrush.ColorProperty, anim); // 애니메이션 레벨이 Local 위에 올라탐
```

> 애니메이션이 끝나면 DP의 유효값은 다시 Local로 복귀(보통 HoldEnd가 아니면).

---

## 끝
“XAML–CLR–렌더링”의 흐름을 몸으로 익히면, 성능·유지보수·확장성이라는 세 마리 토끼를 동시에 잡을 수 있습니다.
