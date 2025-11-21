---
layout: post
title: WPF - PresentationFramework, PresentationCore, milcore
date: 2025-08-27 16:25:23 +0900
category: WPF
---
# WPF 내부 구조 큰 그림: **PresentationFramework / PresentationCore / milcore**

> 이 글은 WPF의 상·하위 계층 구조를 **`PresentationFramework` → `PresentationCore` → `milcore`(Media Integration Layer Core)** 축으로 정리합니다.
> “어떤 네임스페이스/어셈블리에서 무엇을 하고, 어떤 스레드와 채널을 통해 어디까지 내려가며, 최종적으로 화면 픽셀은 어떻게 찍히는가?”를 **예제 중심**으로 설명합니다. 코드 블록은 ` ``` `로 감싸고, 텍스트 전체를 블로그 포맷에 맞춰 정리했습니다.

---

## TL;DR — 한 장 요약

- **PresentationFramework.dll**
  - XAML, 컨트롤, 스타일/템플릿, 데이터 바인딩, 리소스 시스템, 명령/입력 라우팅 등 **UI 프레임워크 레이어**.
  - 개발자가 가장 자주 만나는 영역: `Window`, `Control`, `ItemsControl`, `DataTemplate`, `{Binding}` 등.

- **PresentationCore.dll**
  - **시각적 객체 모델**(Visual, Drawing, Geometry), **미디어/텍스트**(ImageSource, VideoDrawing, TextFormatter), **필수 렌더링 추상화**.
  - WPF의 **장면 그래프(Visual Tree)**를 유지하고, **의존 속성(DependencyProperty)**, **애니메이션**, **Freezable** 등 성능/변경 트래킹의 핵심이 모여 있음.

- **milcore.dll (Media Integration Layer Core)**
  - **네이티브** 구성요소. **Direct3D**와 통신해 실제 **합성(Composition)**과 **그리기**를 수행.
  - **컴포지터 스레드**와 **DUCE 채널**(Dispatcher-UI-Channel-to-Engine)을 통해 **UI 스레드**에서 생성한 장면 그래프를 **렌더 스레드**로 넘겨 GPU/CPU로 합성.

---

## 왜 3계층 구조인가?

WPF는 **보는 것(프레임워크)**과 **그리는 것(코어/미디어)**을 계층화해 **생산성**과 **성능**을 동시에 잡습니다.

- 앱 개발자는 **상위 PresentationFramework**에 집중: XAML, 컨트롤, 바인딩, 트리거, 스타일, 리소스 등을 선언/구성.
- 프레임워크가 내부에서 **PresentationCore** 타입(Visual, DrawingContext 등)을 사용해 **장면 그래프**를 구축.
- 그 장면 그래프는 **milcore**로 전달되어 Direct3D 기반으로 **합성**되어 화면에 출력.

이 구분 덕분에:
- 프레임워크는 **풍부한 UI 개념**(템플릿리, 바인딩, 라우티드 이벤트)을 제공
- **렌더링 엔진**은 **네이티브 최적화**(배치, 클리핑, 타일링, 텍스처 관리)를 집중 수행

---

## 어셈블리별 역할 지도

| 계층 | 대표 어셈블리 | 핵심 네임스페이스 | 하는 일 |
|---|---|---|---|
| 상위 UI 프레임워크 | `PresentationFramework.dll` | `System.Windows`, `System.Windows.Controls`, `System.Windows.Data`, `System.Windows.Documents` | 컨트롤/스타일/템플릿, 리소스, 바인딩, 명령, 문서 흐름, Navigation |
| 시각화 & 코어 | `PresentationCore.dll` | `System.Windows.Media`, `System.Windows.Media.Animation`, `System.Windows.Media.Imaging` | Visual, DrawingContext, Geometry, Brush, Transform, 애니메이션, 이미지 |
| 저수준 미디어/합성 | `milcore.dll` (네이티브) | (공용 CLS 노출 없음) | 컴포지션, D3D 연동, 채널 프로토콜, 렌더 스케줄링 |

추가로 **`WindowsBase.dll`**는 `DependencyObject`, `Freezable`, `Dispatcher` 같은 **기반 인프라**를 제공합니다.

---

## **PresentationFramework** — “프레임워크의 얼굴”

### XAML & 객체화 파이프라인

- XAML을 로딩하면 **BAML**(컴파일된 XAML)→ 객체 그래프로 실체화
- 이때 **스타일/리소스** 머지, **템플릿 확장**, **바인딩** 초기화

```xml
<!-- MainWindow.xaml -->
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        Title="PF/PC/milcore Deep Dive" Width="900" Height="600">
    <Grid>
        <ListBox ItemsSource="{Binding People}">
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <StackPanel Orientation="Horizontal" Margin="4">
                        <Rectangle Width="16" Height="16" Fill="{Binding StatusBrush}" Margin="0,0,8,0"/>
                        <TextBlock Text="{Binding Name}" FontWeight="SemiBold"/>
                        <TextBlock Text="{Binding Title}" Margin="8,0,0,0" Foreground="Gray"/>
                    </StackPanel>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>
    </Grid>
</Window>
```

### 컨트롤 모델 & 템플릿

- 컨트롤은 시각화를 **ControlTemplate**로 위임 → **Lookless** 디자인
- `ItemsControl` 계열은 가상화, 패널 배치(ItemsPanel), 컨테이너 생성(Container Generator) 등 **구성 요소화**가 핵심

```xml
<ListBox ItemsSource="{Binding Files}">
    <ListBox.ItemsPanel>
        <ItemsPanelTemplate>
            <VirtualizingStackPanel/>
        </ItemsPanelTemplate>
    </ListBox.ItemsPanel>
</ListBox>
```

### 데이터 바인딩

- `{Binding}`은 **DependencyProperty** 시스템과 맞물려 **변경 전파**를 자동화
- **BindingExpression**이 소스→타깃 동기화를 관리, Validation/Converters 지원

```xml
<TextBox Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"/>
```

### 리소스 & 스타일

- **ResourceDictionary** 병합(MergedDictionaries), **DynamicResource** vs `StaticResource` 차이
- **ThemeDictionary**를 통한 OS 테마 대응

```xml
<Window.Resources>
    <SolidColorBrush x:Key="Accent" Color="#4F7CAC"/>
    <Style TargetType="Button">
        <Setter Property="Background" Value="{StaticResource Accent}"/>
    </Style>
</Window.Resources>
```

### 라우티드 이벤트 & 명령

- **터널링(Preview→)** → **버블링** 순으로 트리 경로를 따라 전파
- `ICommand`/`CommandBinding`로 **입력**과 **동작**을 느슨하게 결합

```xml
<Button Command="ApplicationCommands.Copy" Content="Copy"/>
```

### & Navigation

- `FlowDocument`는 리플로우 가능한 문서 레이아웃
- NavigationWindow/Frame은 XAML 페이지 기반 **네비게이션 앱** 구성 허용

---

## **PresentationCore** — “렌더링 추상화와 시각적 트리”

### DependencyProperty & Freezable

- **DependencyObject**는 속성 값의 **우선순위 체계**(로컬 값, 스타일, 트리거, 애니메이션…)와 **변경 알림**을 제공
- **Freezable**: 동결 시 **쓰레드 간 공유** 가능 + **변경 비용** 제거 → 브러시, 변환, 기하 등에서 성능 핵심

```csharp
public class MeterBrush : Freezable
{
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(nameof(Value), typeof(double), typeof(MeterBrush),
            new FrameworkPropertyMetadata(0.0, FrameworkPropertyMetadataOptions.AffectsRender));

    public double Value { get => (double)GetValue(ValueProperty); set => SetValue(ValueProperty, value); }

    protected override Freezable CreateInstanceCore() => new MeterBrush();

    public Brush ToBrush()
    {
        // 값에 따라 GradientBrush 생성
        var b = new LinearGradientBrush(
            new GradientStopCollection {
                new GradientStop(Colors.Lime, 0),
                new GradientStop(Colors.Orange, 0.7),
                new GradientStop(Colors.Red, 1)
            },
            new Point(0,0), new Point(1,0));
        if (CanFreeze) b.Freeze();
        return b;
    }
}
```

### Visual / DrawingContext

- **Visual**은 WPF 렌더링의 최소 단위. `UIElement`는 Visual을 상속해 입력/레이아웃/이벤트를 추가.
- 커스텀 그리기는 `OnRender(DrawingContext dc)`에서 수행 → **장면 그래프 노드**에 드로잉 커맨드를 기록

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

        var w = rect.Width * Math.Max(0, Math.Min(1, Value));
        dc.DrawRectangle(Brushes.LimeGreen, null, new Rect(0, 0, w, rect.Height));
    }
}
```

### Transform/Geometry/Brush

- 모든 그리기 명령은 **불변 구조**(Freezable/Freeze 사용)로 캐시/공유되어 **Overhead 최소화**
- `Geometry`(StreamGeometry)로 파스 경량화 → 경로 렌더 비용 절감

```csharp
var geo = new StreamGeometry();
using (var ctx = geo.Open())
{
    ctx.BeginFigure(new Point(0,10), isFilled:true, isClosed:true);
    ctx.PolyLineTo(new[] { new Point(10,0), new Point(20,10) }, true, true);
}
geo.Freeze();
dc.DrawGeometry(Brushes.CornflowerBlue, null, geo);
```

### 애니메이션 클록 & 타임라인

- `AnimationClock`가 **Property 변경을 시간에 따라 적용**
- 프레임마다 값 강제 설정이 아니라, **애니메이션 시스템**이 DP 수준에서 **변경 레이어**로 합산

```csharp
var da = new DoubleAnimation(0, 1, TimeSpan.FromSeconds(1.2)) { AutoReverse = true, RepeatBehavior = RepeatBehavior.Forever };
this.BeginAnimation(MeterBar.ValueProperty, da);
```

### 텍스트 파이프라인

- `TextFormatter`가 **줄나눔, 폰트/스크립트, 힌팅**을 처리하여 **GlyphRun**을 생성
- 고급 시나리오엔 `FormattedText`/`GlyphRunDrawing`으로 저수준 렌더링

```csharp
var ft = new FormattedText(
    "WPF Text",
    System.Globalization.CultureInfo.CurrentUICulture,
    FlowDirection.LeftToRight,
    new Typeface("Segoe UI"),
    24,
    Brushes.White,
    VisualTreeHelper.GetDpi(this).PixelsPerDip);

dc.DrawText(ft, new Point(8, 8));
```

### Imaging

- `BitmapSource`, `WriteableBitmap`, `RenderTargetBitmap` 등 **이미지 소스** 모델
- DPI=96 기반의 **DIP(Device Independent Pixel)** 좌표: 1DIP = 1/96 인치

---

## **milcore** — “합성과 렌더의 심장”

### 스레드

- **UI 스레드**: 트리/속성 변경 → **DUCE 채널**로 **렌더 명령 배치**
- **컴포지터 스레드**(milcore): 채널 읽어 **장면 그래프(컴포지션 트리)**를 업데이트 후 **D3D**로 합성
- UI 스레드가 잠깐 바빠도, 이미 기록된 장면을 **독립적으로 부드럽게** 재생(애니메이션/미디어)

```
[UI Thread] --(DUCE batch)--> [milcore/Compositor Thread] --(D3D)--> [GPU/SwapChain]
```

### DUCE (Dispatcher-UI-Channel-to-Engine)

- 피연산자 생성/속성 변경/리소스 텍스처 업로드/지오메트리 등 **명령 스트림**
- **배치(Batching)**로 왕복 비용 최소화, **Dirty Region** 기반 부분 갱신

### Direct3D & 렌더링 계층

- 전통적으로 D3D9 기반(운영체제/프레임워크 버전에 따라 상이), **Tier 0/1/2** 하드웨어 가속
- GPU 불가 시 **소프트웨어 패스(렌더 테셀레이션/라스터라이즈)**로 폴백

### HwndTarget & 합성 경계

- `HwndSource/HwndHost`를 통해 **Win32 HWND**와 브리지
- 최종 출력은 윈도우의 **타깃 서피스**로 합성

---

## 스레드 모델 & 디스패처

### STA + Dispatcher

- WPF UI는 **STA** 스레드와 **Dispatcher** 메시지 루프에 종속
- 렌더/합성은 **컴포지터 스레드**, 미디어/타이머/애니메이션은 별도 워커가 도울 수 있음

```csharp
[STAThread]
public static void Main()
{
    var app = new Application();
    app.Run(new MainWindow());
}
```

### Freezable와 스레드 안전

- **Freeze**된 리소스는 읽기 전용으로 **크로스 스레드 공유 가능**
- 비프리즈 객체는 **소유 스레드에서만** 접근

---

## 레이아웃 파이프라인(Measure/Arrange)와 렌더 파이프라인의 연결

- **Measure → Arrange → Render**
  - 레이아웃 단계에서 **사이즈/배치** 결정
  - Render 단계에서 **DrawingContext**로 그리기 명령 기록
  - 이 기록이 **milcore**에 전달되어 **합성**

```csharp
public class FixedCellPanel : Panel
{
    protected override Size MeasureOverride(Size constraint)
    {
        var cell = new Size(100, 32);
        foreach (UIElement child in InternalChildren)
            child.Measure(cell);
        var cols = Math.Max(1, (int)(constraint.Width / cell.Width));
        var rows = (int)Math.Ceiling((double)InternalChildren.Count / cols);
        return new Size(cols * cell.Width, rows * cell.Height);
    }

    protected override Size ArrangeOverride(Size arrangeSize)
    {
        var cell = new Size(100, 32);
        var cols = Math.Max(1, (int)(arrangeSize.Width / cell.Width));
        for (int i = 0; i < InternalChildren.Count; i++)
        {
            int r = i / cols;
            int c = i % cols;
            var rect = new Rect(new Point(c * cell.Width, r * cell.Height), cell);
            InternalChildren[i].Arrange(rect);
        }
        return arrangeSize;
    }
}
```

---

## 입력, 명령, 라우팅 이벤트 → 시각화/합성과의 상호작용

- 입력 라우팅(프리뷰→버블)이 **HitTest(Visual Tree)**와 결합
- 시각적 상태/트리거 변경은 DP 변경으로 이어져 **렌더 배치**가 발생

```csharp
protected override void OnMouseEnter(MouseEventArgs e)
{
    this.BeginAnimation(MeterBar.ValueProperty, new DoubleAnimation(1, TimeSpan.FromMilliseconds(300)));
}
```

---

## 리소스 수명주기와 성능 (Freeze/대량 객체/가상화)

### Freeze로 비용 절감

- 브러시/지오메트리/변환 등 **불변화**로 **트랜잭션/클론** 비용 최소화

### 대량 시각화

- **VirtualizingPanel** 사용으로 **실 가시 항목만** 실체화
- DrawingVisual/`AddVisualChild`로 **경량 시각화**를 직접 관리하면 더 큰 스케일 처리 가능

```csharp
public class MillionPoints : FrameworkElement
{
    private readonly DrawingVisual _dv = new DrawingVisual();
    public MillionPoints()
    {
        AddVisualChild(_dv);
        using var dc = _dv.RenderOpen();
        var r = new Random(0);
        for (int i = 0; i < 1_000_00; i++) // 예시: 10만점
        {
            var x = r.NextDouble() * 1000;
            var y = r.NextDouble() * 600;
            dc.DrawRectangle(Brushes.White, null, new Rect(x, y, 1, 1));
        }
    }
    protected override int VisualChildrenCount => 1;
    protected override Visual GetVisualChild(int index) => _dv;
}
```

---

## 미디어/비디오/애니메이션과 컴포지션

- `MediaElement`/`MediaPlayer` 등은 **프레임 스트림**을 텍스처로 올려 **합성**
- 애니메이션 클록은 DP 레벨에서 값 변화를 추적 → 렌더 배치로 반영

---

## D3D 상호운용 (D3DImage, HwndHost)

- **D3DImage**로 Direct3D 텍스처를 WPF 비주얼에 투영
- **HwndHost**로 외부 HWND 콘텐츠(예: Win32/DX 윈도우)를 WPF 트리 안에 포함

```csharp
public class DxHost : HwndHost
{
    protected override HandleRef BuildWindowCore(HandleRef hwndParent)
    {
        // Win32 CreateWindowEx로 자식 HWND를 만들고 D3D를 어태치
        var hwnd = NativeMethods.CreateChildWindow(hwndParent.Handle);
        return new HandleRef(this, hwnd);
    }
    protected override void DestroyWindowCore(HandleRef hwnd) => NativeMethods.DestroyWindow(hwnd.Handle);
}
```

---

## 텍스트 렌더 품질: DPI, 픽셀 스냅, 힌팅

- DIP 기반 좌표계라 폰트가 **1px 경계**에 어긋나면 흐릿함
- `UseLayoutRounding="True"`/`SnapsToDevicePixels="True"` 등으로 **픽셀 경계 정렬**

```xml
<Grid UseLayoutRounding="True" SnapsToDevicePixels="True">
    <TextBlock Text="Sharp Text" />
</Grid>
```

---

## 하드웨어 가속 Tiers & 폴백

- **Tier 2**: 고급 하드웨어 가속
- **Tier 1**: 제한적 가속
- **Tier 0**: 소프트웨어 렌더
- 상황에 따라 **효과(Blur/DropShadow)**, **비트맵 스케일링** 품질/성능 차이

```csharp
var tier = (RenderCapability.Tier >> 16);
Debug.WriteLine($"Render Tier: {tier}"); // 0, 1, 2
```

---

## 리소스 경합과 배치: 장면 그래프 최적화 관점

- **시각적 계층 평탄화**(불필요한 Panel/Border 제거)
- **공유 브러시/Geometry** + Freeze
- **대형 비트맵** 스트리밍 vs 미리 디코딩
- **RenderOptions.BitmapScalingMode** 조절

```xml
<Image Source="big.png" RenderOptions.BitmapScalingMode="LowQuality"/>
```

---

## 측정 단위, DPI 인식, Per-Monitor DPI

- **DIP**로 선언했더라도 **실제 픽셀**은 모니터 DPI마다 다름
- 최신 Windows에선 **Per-Monitor DPI 인식**이 중요 (WPF .NET 4.6+에서 개선)

---

## 프레임 타이밍과 합성: “왜 초당 60프레임이 안 나올까?”

- UI 스레드가 바쁘면 **배치 밀림** → 합성 스레드는 이전 프레임 반복 출력
- **애니메이션/미디어는** 합성 스레드가 독립적으로 유지하지만, **새 장면** 없이 오래가면 시각 업데이트 한계
- **DispatcherPriority** 관리, 백그라운드 작업 오프로딩(Tasks/BackgroundWorker) 필수

---

## Visual Layer API로 “템플릿 없는” 화면 만들기

- 컨트롤 프레임워크를 거치지 않고 **Visual + DrawingContext**로 장면 구성
- 대규모 데이터 렌더(차트, 맵, 타일러) 시 오버헤드 대폭 절감

```csharp
public sealed class VisualLayerCanvas : FrameworkElement
{
    private readonly VisualCollection _visuals;
    public VisualLayerCanvas() => _visuals = new VisualCollection(this);
    protected override int VisualChildrenCount => _visuals.Count;
    protected override Visual GetVisualChild(int index) => _visuals[index];

    public void Draw(Rect rect, Brush fill)
    {
        var dv = new DrawingVisual();
        using (var dc = dv.RenderOpen())
            dc.DrawRectangle(fill, null, rect);
        _visuals.Add(dv);
    }
}
```

---

## 애니메이션 vs Composition Target

- **Storyboard/Timeline** 기반 애니메이션은 DP와 통합
- `CompositionTarget.Rendering` 이벤트로 **프레임 동기** 루프를 구현 가능 (게임/시뮬레이션)

```csharp
CompositionTarget.Rendering += (s, e) =>
{
    // 프레임별 업데이트
};
```

---

## / ShaderEffect(GPU)

- CPU로 픽셀 조작: `WriteableBitmap`
- GPU로 픽셀 이펙트: `ShaderEffect`(픽셀 셰이더 HLSL)
- 상황에 따라 **대역폭/지연** 트레이드오프

```csharp
var wb = new WriteableBitmap(640, 480, 96, 96, PixelFormats.Bgra32, null);
wb.Lock();
// unsafe로 back buffer 접근 후 처리
wb.AddDirtyRect(new Int32Rect(0, 0, wb.PixelWidth, wb.PixelHeight));
wb.Unlock();
```

---

## 문서/인쇄 파이프라인

- `FixedDocument`/`FlowDocument`에서 **Paginator**로 페이지 생산
- `PrintDialog` / `XpsDocumentWriter`로 출력 → 화면과 유사한 **벡터 품질**

---

## 리소스 로딩과 이미지 캐시

- Image/BitmapCacheOption, DecodePixelWidth/Height 등으로 **메모리/시간 최적화**
- 같은 URI의 BitmapSource는 **디코더 캐시 공유**(조건부)로 메모리 절감

```xml
<Image Source="/Assets/Huge.png" DecodePixelWidth="600"/>
```

---

## 성능 문제 체크리스트

1) **Layout thrash**: 빈번한 `Measure/Arrange` 유발
2) **대형 Visual 트리**: 불필요한 Panel/Decorator 제거
3) **Binding 폭주**: `INotifyPropertyChanged` 과다, 바인딩 에러 로그
4) **Bitmap 스케일링**: 고품질 스케일은 비용 큼
5) **프리징 미사용**: 공유 리소스는 Freeze
6) **너무 많은 Effects/Opacity**: 합성 경로 비싸짐
7) **DPI/픽셀 스냅** 미적용으로 흐릿하거나 불필요한 반픽셀 렌더
8) **UI 스레드 블로킹**: 파일 IO/CPU 작업은 비동기로

---

## milcore와 “그 아래”를 굳이 더 알면 좋은 포인트

- **Dirty Region/Clip/Transform**을 활용한 합성 비용 절감
- **Z-Order/Overdraw** 최소화
- **RenderTarget 공유/재사용** 전략
- **Text/Glyph Cache** 재사용

> WPF 개발자는 milcore를 직접 호출할 수 없지만, **상위 계층의 설계 방식**이 **컴포지션 비용**에 큰 영향을 준다는 점을 이해하면 튜닝 포인트가 보입니다.

---

## 종단 간 흐름을 하나의 예로

1. 개발자: XAML로 `ListBox` + `DataTemplate` 구성
2. PresentationFramework: ItemsControl이 컨테이너/패널/가상화 설정
3. PresentationCore: Visual로 드로잉 명령(텍스트/도형/이미지)을 기록
4. milcore: 명령 배치를 수신해 컴포지션 트리 갱신
5. D3D: 텍스처/지오메트리/셰이더를 통해 화면 합성
6. 모니터: VSync에 맞춰 스왑 → 픽셀 출력

---

## 실전 예제 — “컨트롤 없이” 60FPS 막대 차트

> 컨트롤 프레임워크 대신 Visual Layer로 그려 **오버헤드 최소화**, 애니메이션은 CompositionTarget에 맞춰 값 갱신.

```csharp
public sealed class Bars : FrameworkElement
{
    private readonly DrawingVisual _dv = new DrawingVisual();
    private readonly Random _rand = new Random(1);
    private double[] _values;
    private Stopwatch _sw;

    public Bars()
    {
        _values = Enumerable.Range(0, 2000).Select(_ => _rand.NextDouble()).ToArray();
        AddVisualChild(_dv);
        _sw = Stopwatch.StartNew();
        CompositionTarget.Rendering += OnRendering;
    }

    private void OnRendering(object? s, EventArgs e)
    {
        // 시간에 따라 값 변화
        var t = _sw.Elapsed.TotalSeconds;
        for (int i = 0; i < _values.Length; i++)
            _values[i] = 0.5 + 0.5 * Math.Sin(t + i * 0.03);

        using var dc = _dv.RenderOpen();
        var w = ActualWidth;
        var h = ActualHeight;
        var bw = Math.Max(1, w / _values.Length);

        dc.DrawRectangle(Brushes.Black, null, new Rect(0, 0, w, h));
        for (int i = 0; i < _values.Length; i++)
        {
            var val = _values[i];
            var bh = val * (h - 2);
            dc.DrawRectangle(Brushes.Lime, null, new Rect(i * bw, h - bh, bw - 1, bh));
        }
    }

    protected override int VisualChildrenCount => 1;
    protected override Visual GetVisualChild(int index) => _dv;
}
```

```xml
<Window ...>
    <Grid>
        <local:Bars/>
    </Grid>
</Window>
```

**핵심 포인트**
- `DrawingVisual` 하나에 매 프레임 재그림 → **노드 수 최소화**
- 컨트롤/바인딩/템플릿 오버헤드 제거 → milcore까지 **명령 배치가 간결**
- 수천 개 사각형도 빠르게 합성

---

## 실전 예제 — `D3DImage`를 통한 네이티브 텍스처 합성

> Direct3D에서 만든 텍스처를 WPF 트리에 올려 **고가의 영상/과학 시각화**를 WPF UI와 결합

```csharp
public class D3DImageHost : Image
{
    private D3DImage _d3dImage;
    private IntPtr _surface;

    public D3DImageHost()
    {
        _d3dImage = new D3DImage();
        this.Source = _d3dImage;
        CompositionTarget.Rendering += (s,e) =>
        {
            if (_surface != IntPtr.Zero)
            {
                _d3dImage.Lock();
                _d3dImage.SetBackBuffer(D3DResourceType.IDirect3DSurface9, _surface);
                _d3dImage.AddDirtyRect(new Int32Rect(0,0,_d3dImage.PixelWidth,_d3dImage.PixelHeight));
                _d3dImage.Unlock();
            }
        };
    }

    public void AttachSurface(IntPtr surface, int width, int height)
    {
        _surface = surface;
        _d3dImage.Lock();
        _d3dImage.SetBackBuffer(D3DResourceType.IDirect3DSurface9, surface);
        _d3dImage.SetPixelSize(width, height);
        _d3dImage.Unlock();
    }
}
```

> 네이티브 측(D3D9/11 → 9Ex 공유 등) 준비가 필요하지만, 아이디어는 **WPF 합성 경로에 텍스처를 편입**하는 것.

---

## 흔한 난관과 처방

- **텍스트 흐릿함**: LayoutRounding/PixelSnap + 정수 좌표
- **스케일 변환(Zoom) 시 성능 저하**: 전체 트리 ScaleTransform 남용 주의, 레이어링/타일링
- **대량 알파 블렌딩**: Overdraw 줄이기, 반투명 레이어 최소화
- **애니메이션 끊김**: UI 스레드 블로킹 제거, 애니메이션 DP로 위임
- **메모리 급증**: 비트맵 복사/디코딩 전략 점검(DecodePixelWidth/Height), 캐시 공유

---

## 체크리스트: PresentationFramework → PresentationCore → milcore **정렬 맞추기**

1. **프레임워크 오버헤드 최소화**
   - 불필요한 컨트롤/패널/데코레이터 제거
   - 바인딩 에러/빈번한 Notify 폭주를 로그로 잡기
2. **시각화 경량화**
   - VisualLayer(DrawingVisual)로 대량 렌더
   - Freezable Freeze, 공유 리소스 재사용
3. **합성 비용 최적화**
   - 픽셀 스냅/클리핑/오버드로 최소
   - 비트맵 스케일/이펙트 최소화, 필요한 곳에만
4. **스레드/디스패처**
   - 백그라운드 작업 분리, UI 스레드 유휴 확보
   - CompositionTarget/애니메이션을 올바르게 사용

---

## FAQ

- **Q. milcore를 직접 건드릴 수 있나요?**
  - A. 아니요. 공개 API가 아닙니다. 하지만 **상위 계층에서 올바른 설계**로 milcore의 작업 부담을 크게 줄일 수 있습니다.

- **Q. Freezable은 꼭 써야 하나요?**
  - A. 브러시/지오메트리/변환처럼 **공유·정적 리소스**는 Freeze가 성능상 매우 유리합니다.

- **Q. 120Hz 모니터에서 120FPS가 가능한가요?**
  - A. 합성/디스패처 타이밍과 GPU/CPU 여건에 달렸습니다. **작업량이 충분히 가벼울 때** 가능합니다. 병목은 주로 **UI 스레드**와 **대량 오버드로**입니다.

---

## 마무리

이 글의 핵심은 **“프레임워크(편의)와 렌더링(성능) 사이의 계약”**을 정확히 이해하는 것입니다.
- **PresentationFramework**에선 **바인딩/템플릿/스타일**을 사용하되, 과유불급을 경계합니다.
- **PresentationCore**에선 **Visual/Drawing/Freezable**을 이해해 **장면 그래프**를 깔끔하게 유지합니다.
- **milcore**는 보이지 않지만, **합성 비용**은 언제나 여기서 청구됩니다.

WPF가 “느린가?”라는 질문은 종종 **계약 위반**에서 시작합니다.
계약을 지키면, WPF는 여전히 **강력하고 부드러운 데스크톱 UI 엔진**입니다.

---

### 부록 A — 샘플: Freezable 기반 그래디언트 브러시 리소스 팩토리

```csharp
public class GradientFactory : Freezable
{
    public static readonly DependencyProperty StartProperty =
        DependencyProperty.Register(nameof(Start), typeof(Color), typeof(GradientFactory),
            new FrameworkPropertyMetadata(Colors.LightSkyBlue, OnChanged));
    public static readonly DependencyProperty EndProperty =
        DependencyProperty.Register(nameof(End), typeof(Color), typeof(GradientFactory),
            new FrameworkPropertyMetadata(Colors.SteelBlue, OnChanged));

    public Color Start { get => (Color)GetValue(StartProperty); set => SetValue(StartProperty, value); }
    public Color End   { get => (Color)GetValue(EndProperty);   set => SetValue(EndProperty, value); }

    private LinearGradientBrush? _cache;

    protected override Freezable CreateInstanceCore() => new GradientFactory();

    private static void OnChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
        => ((GradientFactory)d).Invalidate();

    private void Invalidate() => _cache = null;

    public Brush Get()
    {
        if (_cache is null)
        {
            _cache = new LinearGradientBrush(Start, End, 0);
            if (_cache.CanFreeze) _cache.Freeze();
        }
        return _cache;
    }
}
```

```xml
<Window.Resources>
    <local:GradientFactory x:Key="HeaderGrad" Start="#5FA8D3" End="#1B4965"/>
</Window.Resources>

<Border Background="{Binding Source={StaticResource HeaderGrad}, Path=Get}"/>
```

---

### 부록 B — Visual Tree HitTest로 커스텀 인터랙션

```csharp
protected override void OnMouseDown(MouseButtonEventArgs e)
{
    var p = e.GetPosition(this);
    HitTestResult hit = VisualTreeHelper.HitTest(this, p);
    if (hit?.VisualHit is DrawingVisual dv)
    {
        // dv에 태그/메타데이터를 붙여놓고 식별
        // 선택/하이라이트 로직
    }
}
```

---

### 부록 C — 텍스트 고급: GlyphRun 수동 렌더

```csharp
var typeface = new Typeface(new FontFamily("Segoe UI"), FontStyles.Normal, FontWeights.Normal, FontStretches.Normal);
if (typeface.TryGetGlyphTypeface(out var gtf))
{
    var indices = new ushort[] { /* 문자의 glyph index들 */ };
    var advances = Enumerable.Repeat(12.0, indices.Length).ToArray();
    var gr = new GlyphRun(gtf, 0, false, 24, indices, new Point(20, 40), advances, null, null, null, null, null, null);
    dc.DrawGlyphRun(Brushes.White, gr);
}
```

---

### 부록 D — BitmapCache/CacheMode로 벡터 일괄 캐시

```xml
<Canvas>
    <Path Data="M0,0 L100,0 100,100 0,100 z"
          Fill="DarkCyan">
        <Path.CacheMode>
            <BitmapCache RenderAtScale="2"/>
        </Path.CacheMode>
    </Path>
</Canvas>
```

> 변형이 잦고 복잡한 벡터를 **비트맵화**하여 **합성 비용**을 줄이는 테크닉입니다(상황 의존적).

---

## 끝

이제 WPF를 설계할 때, “이 레이어의 결정이 저 레이어에 어떤 비용을 남기는가?”를 항상 떠올려 보세요. 그 습관이 성능과 유지보수성을 동시에 끌어올립니다.
