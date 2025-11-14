---
layout: post
title: WPF - MediaElement & 3D
date: 2025-09-07 18:25:23 +0900
category: WPF
---
# 🎬 WPF **MediaElement & 3D (Viewport3D)** 완전 정복

*(예제 중심 · 누락 없는 설명 · 실전 성능/트러블슈팅 포함)*

이 문서는 WPF에서 **미디어(오디오/비디오) 재생**과 **3D 렌더링**을 다루는 핵심 구성요소를 “처음부터 끝까지” 상세히 정리합니다.
핵심 주제는 아래와 같습니다.

- **MediaElement** / **MediaPlayer** / **MediaTimeline** 차이와 선택 기준
- 파일·리소스·스트리밍 소스 연결(URI/pack URI), 재생 제어(Play/Pause/Seek/Loop)
- 이벤트(Opened/MediaEnded/Failed)/볼륨/음소거/속도/자막(기본)/동기화
- **VideoBrush(=VisualBrush)** 로 비디오를 **2D/3D 표면**에 텍스처링
- **Viewport3D** 기반 3D 장면 구성: 카메라, 조명, 메시, 머티리얼, 변환, 애니메이션
- 3D 히트테스트, 마우스 픽킹, 회전/줌/팬, 성능 최적화
- 흔한 함정/제약/성능 팁(가속, 프레임 드롭, Pixel Snapping 등)

> 예제는 .NET 5+ WPF 기준(Framework 4.8도 대부분 동일). XAML은 어디서든 붙여 넣기 가능하도록 작성했습니다.

---

## 한눈에 보는 미디어/3D 구성요소

| 영역 | 클래스 | 용도 | 특징 |
|---|---|---|---|
| **미디어 재생** | `MediaElement` | XAML에 올려 **UI 엘리먼트로 재생** | 간단/바로쓰기, 컨트롤 템플릿 가능, 렌더 트리 내에서 동작 |
|  | `MediaPlayer` + `VideoDrawing` | **코드 주도 재생** + 드로잉 시각화 | 백엔드 객체. UI에 직접 올리지 않고 DrawingContext로 렌더 |
|  | `MediaTimeline`/`MediaClock` | 타임라인 기반 **재생 흐름 제어** | Storyboard/Timeline 모델과 결합, 반복/동기화 용이 |
| **3D** | `Viewport3D` | WPF **3D 씬 컨테이너** | 3D를 2D 시각 트리에 통합 |
|  | `Model3DGroup`/`GeometryModel3D`/`MeshGeometry3D` | 메시·재질·변환 | Freezable(성능에 도움), 애니메이션 가능 |
|  | `DirectionalLight`/`AmbientLight`/`PointLight` | 조명 | Lambert식 기본 라이팅 |
|  | `PerspectiveCamera`/`OrthographicCamera` | 카메라 | 원근/직교 |
|  | `ModelUIElement3D` | 3D 모델에 **입력 이벤트** 연결 | MouseDown 등 |

---

# MediaElement 완전 정복

## 가장 단순한 재생

```xml
<Grid>
  <MediaElement x:Name="Player"
                Source="sample.mp4"
                LoadedBehavior="Manual" UnloadedBehavior="Stop"
                Stretch="Uniform" />
  <StackPanel Orientation="Horizontal" HorizontalAlignment="Center" VerticalAlignment="Bottom" Margin="0,0,0,16">
    <Button Content="▶" Click="Play_Click"/>
    <Button Content="⏸" Click="Pause_Click"/>
    <Button Content="⏹" Click="Stop_Click"/>
  </StackPanel>
</Grid>
```

```csharp
private void Play_Click(object s, RoutedEventArgs e) => Player.Play();
private void Pause_Click(object s, RoutedEventArgs e) => Player.Pause();
private void Stop_Click(object s, RoutedEventArgs e) => Player.Stop();
```

### 속성 요약

- `Source` : 재생할 미디어 URI(로컬, pack, http/https)
- `LoadedBehavior`/`UnloadedBehavior` : `Play/Pause/Manual/Stop`
- `Volume`(0~1), `IsMuted`, `SpeedRatio`(재생 속도), `Position`(seek)
- `ScrubbingEnabled` : 프레임 단위 탐색(코스트↑ 가능)
- `Stretch` : 영상 비율 채우기 모드 (`Uniform` 권장)

### 주요 이벤트

- `MediaOpened` : 메타데이터 읽힘(영상 크기, 길이 등 접근 가능)
- `MediaEnded` : 끝 도달 (Loop 시 다시 Play)
- `MediaFailed` : 코덱/파일 문제 등

```csharp
private void Player_MediaOpened(object? sender, RoutedEventArgs e)
{
    Debug.WriteLine($"NaturalVideoWidth={Player.NaturalVideoWidth}, Duration={Player.NaturalDuration}");
}
private void Player_MediaEnded(object? sender, RoutedEventArgs e) => Player.Position = TimeSpan.Zero;
private void Player_MediaFailed(object? sender, ExceptionRoutedEventArgs e) => MessageBox.Show(e.ErrorException?.Message);
```

> 🔎 **코덱 주의**: WPF는 OS 코덱 파이프라인을 사용합니다. 시스템에 코덱이 없거나 DRM/포맷 미지원이면 `MediaFailed`가 발생할 수 있습니다.

---

## 파일 경로와 **pack URI** / 스트리밍

### 로컬/상대/절대 경로

```xml
<MediaElement Source="media/video.mp4"/>
<MediaElement Source="C:\Videos\clip.mp4"/>
```

### **pack URI로 리소스 임베드**

- `video.mp4`를 프로젝트에 추가 → Build Action = Resource
```xml
<MediaElement Source="pack://application:,,,/media/video.mp4"/>
```

### 스트리밍(HTTP/HTTPS)

```xml
<MediaElement Source="https://example.com/video.mp4" LoadedBehavior="Manual"/>
```

> 네트워크 지연 시 `MediaOpened`가 늦게 발생합니다. UI 로딩 스피너를 두고 이벤트에서 해제하세요.

---

## 재생 컨트롤(시크바/볼륨/속도)

```xml
<Grid>
  <MediaElement x:Name="Player" Source="sample.mp4" LoadedBehavior="Manual" />
  <DockPanel VerticalAlignment="Bottom" Background="#80000000" LastChildFill="True" Padding="8">
    <Slider x:Name="Seek" Minimum="0" Maximum="1" ValueChanged="Seek_ValueChanged" Width="200" />
    <Slider x:Name="Vol"  Minimum="0" Maximum="1" Value="0.8" Width="100" Margin="8,0,0,0"/>
    <ComboBox x:Name="Speed" Width="70" Margin="8,0,0,0">
      <ComboBoxItem>0.5</ComboBoxItem><ComboBoxItem>1.0</ComboBoxItem><ComboBoxItem>1.5</ComboBoxItem><ComboBoxItem>2.0</ComboBoxItem>
    </ComboBox>
  </DockPanel>
</Grid>
```

```csharp
DispatcherTimer _tick = new() { Interval = TimeSpan.FromMilliseconds(100) };
public MainWindow()
{
    InitializeComponent();
    _tick.Tick += (_,__) =>
    {
        if (Player.NaturalDuration.HasTimeSpan)
            Seek.Value = Player.Position.TotalSeconds / Player.NaturalDuration.TimeSpan.TotalSeconds;
    };
}
private void Play_Click(object s, RoutedEventArgs e) { Player.Play(); _tick.Start(); }
private void Pause_Click(object s, RoutedEventArgs e) => Player.Pause();
private void Stop_Click(object s, RoutedEventArgs e) { Player.Stop(); _tick.Stop(); Seek.Value = 0; }
private void Seek_ValueChanged(object s, RoutedPropertyChangedEventArgs<double> e)
{
    if (Player.NaturalDuration.HasTimeSpan && (bool)IsLoaded)
        Player.Position = TimeSpan.FromSeconds(e.NewValue * Player.NaturalDuration.TimeSpan.TotalSeconds);
}
private void Vol_ValueChanged(object s, RoutedPropertyChangedEventArgs<double> e) => Player.Volume = e.NewValue;
private void Speed_SelectionChanged(object s, SelectionChangedEventArgs e)
{
    if (Speed.SelectedItem is ComboBoxItem item && double.TryParse((string)item.Content, out var r))
        Player.SpeedRatio = r;
}
```

> `ScrubbingEnabled=true`면 프레임 단위로 당겨보기가 가능하지만 비용이 크므로 UI/시스템 상황에 맞춰 토글하세요.

---

## 오디오만 재생(백그라운드 음악)

```xml
<MediaElement x:Name="Bgm" Source="loop.mp3" LoadedBehavior="Manual" UnloadedBehavior="Stop" Visibility="Collapsed" MediaEnded="Bgm_MediaEnded"/>
```
```csharp
private void Bgm_MediaEnded(object s, RoutedEventArgs e) { Bgm.Position = TimeSpan.Zero; Bgm.Play(); }
```

---

## **MediaPlayer + VideoDrawing** (코드 지향 / 시각 요소 분리)

```xml
<DrawingBrush x:Key="VideoBrush">
  <DrawingBrush.Drawing>
    <VideoDrawing x:Name="VideoDraw" Rect="0,0,640,360"/>
  </DrawingBrush.Drawing>
</DrawingBrush>

<Rectangle Width="640" Height="360" Fill="{StaticResource VideoBrush}"/>
```

```csharp
MediaPlayer _mp = new();
public MainWindow()
{
    InitializeComponent();
    _mp.Open(new Uri("sample.mp4", UriKind.Relative));
    // VideoDrawing 찾기
    var brush = (DrawingBrush)Resources["VideoBrush"];
    var vdraw = (VideoDrawing)brush.Drawing;
    vdraw.Player = _mp;
    _mp.Play();
}
```

**언제 쓰나?**
- 여러 곳에 같은 소스를 그릴 때(동일 프레임 공유),
- UIElement 제약 없이 **DrawingContext**로 렌더하고 싶을 때.

---

## **MediaTimeline/MediaClock** (타임라인 기반 반복/동기화)

```xml
<Grid>
  <Grid.Resources>
    <MediaTimeline x:Key="LoopTl" Source="sample.mp4" RepeatBehavior="Forever"/>
  </Grid.Resources>
  <MediaElement x:Name="TLElement" Clock="{StaticResource LoopTl}" />
</Grid>
```

- 타임라인을 **Storyboard**와 동일한 감각으로 반복/동기화할 수 있습니다.
- `MediaTimeline`은 **시간 제어(Seek, SpeedRatio, Pause)** 를 일관된 API로 제공합니다.

---

## Video를 **Brush**로 써서 컨트롤/3D 텍스처링

```xml
<Grid>
  <Grid.Resources>
    <VisualBrush x:Key="VideoVis">
      <VisualBrush.Visual>
        <MediaElement x:Name="VideoVisual" Source="sample.mp4" LoadedBehavior="Play" Stretch="UniformToFill"/>
      </VisualBrush.Visual>
    </VisualBrush>
  </Grid.Resources>

  <Rectangle Fill="{StaticResource VideoVis}" RadiusX="16" RadiusY="16" />
</Grid>
```

> `VisualBrush`로 감싸면 비디오를 **임의의 2D/3D 표면**에 입힐 수 있습니다(아래 3D에서 재사용).

---

## 자막/오디오 트랙에 대하여 (간단 언급)

- WPF **순정 MediaElement**는 **내장 자막/트랙 전환**을 자동 처리하지 않습니다.
  보통 **외부 자막(SRT)** 을 로딩해 별도의 `TextBlock`/Overlay로 동기화하거나,
  다른 미디어 스택(예: Media Foundation 직접, 서드파티 엔진)과 연계합니다.

---

## MediaElement 트러블슈팅

- **검은 화면/소리만**: 코덱 미지원 가능 → 컨테이너/코덱 확인
- **프레임 드롭**: Stretch/레이아웃 변환 최소화, 하드웨어 가속 확인, 다른 애니메이션 과다 회피
- **메모리 증가**: Stop/Close, 참조 해제, MediaPlayer 재사용 정책 점검
- **Scrubbing 느림**: ScrubbingEnabled 필요할 때만, 키프레임 구조 고려

---

# WPF 3D(Viewport3D) 완전 정복

## 최소 구성 예: 카메라 + 조명 + 메시

```xml
<Viewport3D x:Name="View" ClipToBounds="True">
  <!-- 카메라 -->
  <Viewport3D.Camera>
    <PerspectiveCamera Position="0,0,5" LookDirection="0,0,-5" UpDirection="0,1,0" FieldOfView="45"/>
  </Viewport3D.Camera>

  <!-- 씬 모델 -->
  <ModelVisual3D>
    <ModelVisual3D.Content>
      <Model3DGroup>
        <!-- 조명 -->
        <DirectionalLight Color="White" Direction="-1,-1,-2"/>
        <AmbientLight Color="#404040"/>

        <!-- 메시 -->
        <GeometryModel3D>
          <GeometryModel3D.Geometry>
            <MeshGeometry3D
              Positions="-1,-1,0  1,-1,0  1,1,0  -1,1,0"
              TriangleIndices="0 1 2  0 2 3"
              TextureCoordinates="0,1 1,1 1,0 0,0"/>
          </GeometryModel3D.Geometry>
          <GeometryModel3D.Material>
            <DiffuseMaterial>
              <DiffuseMaterial.Brush>
                <ImageBrush ImageSource="checker.png"/>
              </DiffuseMaterial.Brush>
            </DiffuseMaterial>
          </GeometryModel3D.Material>
        </GeometryModel3D>

      </Model3DGroup>
    </ModelVisual3D.Content>
  </ModelVisual3D>
</Viewport3D>
```

- **카메라**: `PerspectiveCamera`(원근) / `OrthographicCamera`(직교)
- **조명**: `DirectionalLight`, `AmbientLight`, `PointLight`, `SpotLight`
- **메시**: `MeshGeometry3D`(정점/인덱스/UV/Normals), `GeometryModel3D`
- **재질(Material)**: `DiffuseMaterial`, `SpecularMaterial`, `EmissiveMaterial` (혼합 가능, `MaterialGroup`)

---

## **정육면체** 만들기(Positions/Indices)

```xml
<MeshGeometry3D x:Key="CubeMesh"
  Positions="
   -1,-1,-1  1,-1,-1  1,1,-1  -1,1,-1
   -1,-1, 1  1,-1, 1  1,1, 1  -1,1, 1"
  TriangleIndices="
   0 1 2  0 2 3   <!-- Back -->
   4 6 5  4 7 6   <!-- Front -->
   0 4 5  0 5 1   <!-- Bottom -->
   2 6 7  2 7 3   <!-- Top -->
   0 7 4  0 3 7   <!-- Left -->
   1 5 6  1 6 2"  /><!-- Right -->
```

```xml
<Viewport3D>
  <Viewport3D.Camera>
    <PerspectiveCamera Position="3,3,6" LookDirection="-3,-3,-6" UpDirection="0,1,0" FieldOfView="45"/>
  </Viewport3D.Camera>
  <ModelVisual3D>
    <ModelVisual3D.Content>
      <Model3DGroup>
        <DirectionalLight Color="White" Direction="-1,-1,-2"/>
        <GeometryModel3D Geometry="{StaticResource CubeMesh}">
          <GeometryModel3D.Material>
            <MaterialGroup>
              <DiffuseMaterial Brush="#4066CCFF"/>
              <SpecularMaterial SpecularPower="30" Brush="#80FFFFFF"/>
            </MaterialGroup>
          </GeometryModel3D.Material>
          <GeometryModel3D.Transform>
            <RotateTransform3D>
              <RotateTransform3D.Rotation>
                <AxisAngleRotation3D x:Name="Rot" Axis="0,1,0" Angle="0"/>
              </RotateTransform3D.Rotation>
            </RotateTransform3D>
          </GeometryModel3D.Transform>
        </GeometryModel3D>
      </Model3DGroup>
    </ModelVisual3D.Content>
  </ModelVisual3D>

  <!-- 회전 애니메이션 -->
  <Viewport3D.Triggers>
    <EventTrigger RoutedEvent="Loaded">
      <BeginStoryboard>
        <Storyboard RepeatBehavior="Forever">
          <DoubleAnimation Storyboard.TargetName="Rot" Storyboard.TargetProperty="Angle" From="0" To="360" Duration="0:0:6"/>
        </Storyboard>
      </BeginStoryboard>
    </EventTrigger>
  </Viewport3D.Triggers>
</Viewport3D>
```

> 3D 회전은 `RotateTransform3D + AxisAngleRotation3D` 조합이 간편합니다.

---

## 3D 표면에 **비디오 텍스처** 입히기 (MediaElement + VisualBrush)

```xml
<Grid>
  <Grid.Resources>
    <!-- 비디오를 Visual로 감싸 브러시화 -->
    <VisualBrush x:Key="VideoBrush">
      <VisualBrush.Visual>
        <MediaElement Source="sample.mp4" LoadedBehavior="Play" Stretch="UniformToFill"/>
      </VisualBrush.Visual>
    </VisualBrush>
  </Grid.Resources>

  <Viewport3D>
    <Viewport3D.Camera>
      <PerspectiveCamera Position="0,0,4" LookDirection="0,0,-4" UpDirection="0,1,0" FieldOfView="40"/>
    </Viewport3D.Camera>

    <ModelVisual3D>
      <ModelVisual3D.Content>
        <Model3DGroup>
          <DirectionalLight Color="White" Direction="-0.5,-0.5,-1"/>
          <GeometryModel3D>
            <GeometryModel3D.Geometry>
              <MeshGeometry3D
                Positions="-1,-1,0  1,-1,0  1,1,0  -1,1,0"
                TriangleIndices="0 1 2  0 2 3"
                TextureCoordinates="0,1 1,1 1,0 0,0"/>
            </GeometryModel3D.Geometry>
            <GeometryModel3D.Material>
              <DiffuseMaterial Brush="{StaticResource VideoBrush}"/>
            </GeometryModel3D.Material>
          </GeometryModel3D>
        </Model3DGroup>
      </ModelVisual3D.Content>
    </ModelVisual3D>
  </Viewport3D>
</Grid>
```

> 이렇게 하면 **3D 스크린**(예: TV/프로젝션 월)에 **실제 동영상**을 재생할 수 있습니다.
> 복수 표면에서 동일 비디오를 공유하려면 `MediaPlayer + VideoDrawing` 조합이 유리합니다.

---

## 카메라 조작(마우스 회전/줌/팬)

```csharp
// MainWindow.xaml.cs
private PerspectiveCamera? _cam;
private Point _last;
private double _yaw, _pitch, _dist = 6;

private void View_Loaded(object s, RoutedEventArgs e)
{
    _cam = (PerspectiveCamera)View.Camera;
    CompositionTarget.Rendering += (_,__) => UpdateCamera();
}
private void UpdateCamera()
{
    if (_cam == null) return;
    var dir = new Vector3D(
        Math.Cos(_pitch) * Math.Sin(_yaw),
        Math.Sin(_pitch),
        Math.Cos(_pitch) * Math.Cos(_yaw));
    var pos = -dir; pos *= _dist;
    _cam.Position     = new Point3D(pos.X, pos.Y, pos.Z);
    _cam.LookDirection= dir;
    _cam.UpDirection  = new Vector3D(0,1,0);
}

private void View_MouseDown(object s, MouseButtonEventArgs e) { _last = e.GetPosition(View); Mouse.Capture(View); }
private void View_MouseMove(object s, MouseEventArgs e)
{
    if (!View.IsMouseCaptured) return;
    var p = e.GetPosition(View);
    _yaw   += (p.X - _last.X) * 0.01;
    _pitch += (p.Y - _last.Y) * 0.01;
    _pitch = Math.Clamp(_pitch, -1.2, 1.2);
    _last = p;
}
private void View_MouseUp(object s, MouseButtonEventArgs e) => Mouse.Capture(null);
private void View_MouseWheel(object s, MouseWheelEventArgs e) => _dist = Math.Clamp(_dist - e.Delta * 0.001, 2, 20);
```

```xml
<Viewport3D x:Name="View" Loaded="View_Loaded" MouseDown="View_MouseDown" MouseMove="View_MouseMove" MouseUp="View_MouseUp" MouseWheel="View_MouseWheel">
  <!-- 카메라/씬 … (앞 예제 참고) -->
</Viewport3D>
```

---

## 3D **히트테스트**(모델 클릭/선택)

```csharp
private void View_MouseLeftButtonDown(object sender, MouseButtonEventArgs e)
{
    var pt = e.GetPosition(View);
    var hit = VisualTreeHelper.HitTest(View, pt) as RayHitTestResult;
    if (hit is RayMeshGeometry3DHitTestResult meshHit)
    {
        var model = (GeometryModel3D)meshHit.ModelHit;
        // 예: 선택된 모델 강조
        if (model.Material is DiffuseMaterial dm)
            dm.Brush = new SolidColorBrush(Colors.Orange);
    }
}
```

```xml
<Viewport3D x:Name="View" MouseLeftButtonDown="View_MouseLeftButtonDown">
  <!-- 씬 -->
</Viewport3D>
```

> 보다 편한 입력 바인딩은 `ModelUIElement3D`를 사용해 `MouseDown` 등을 직접 연결할 수 있습니다.

---

## 3D 애니메이션(회전/이동/스케일)

```xml
<GeometryModel3D x:Name="Obj" Geometry="{StaticResource CubeMesh}">
  <GeometryModel3D.Transform>
    <Transform3DGroup>
      <ScaleTransform3D x:Name="Sc" ScaleX="1" ScaleY="1" ScaleZ="1"/>
      <RotateTransform3D>
        <RotateTransform3D.Rotation>
          <AxisAngleRotation3D x:Name="RotY" Axis="0,1,0" Angle="0"/>
        </RotateTransform3D.Rotation>
      </RotateTransform3D>
      <TranslateTransform3D x:Name="T" OffsetX="0" OffsetY="0" OffsetZ="0"/>
    </Transform3DGroup>
  </GeometryModel3D.Transform>
</GeometryModel3D>

<!-- Storyboard -->
<Viewport3D.Triggers>
  <EventTrigger RoutedEvent="Loaded">
    <BeginStoryboard>
      <Storyboard RepeatBehavior="Forever">
        <DoubleAnimation Storyboard.TargetName="RotY" Storyboard.TargetProperty="Angle" From="0" To="360" Duration="0:0:4"/>
        <DoubleAnimation Storyboard.TargetName="Sc" Storyboard.TargetProperty="ScaleX" From="1" To="1.2" AutoReverse="True" Duration="0:0:1.2"/>
        <DoubleAnimation Storyboard.TargetName="Sc" Storyboard.TargetProperty="ScaleY" From="1" To="1.2" AutoReverse="True" Duration="0:0:1.2"/>
      </Storyboard>
    </BeginStoryboard>
  </EventTrigger>
</Viewport3D.Triggers>
```

---

## 2D UI를 3D에 투영(빌보드/대시보드)

```xml
<Grid>
  <Grid.Resources>
    <VisualBrush x:Key="UiBrush">
      <VisualBrush.Visual>
        <Border Background="#CC111827" CornerRadius="8" Padding="12">
          <StackPanel>
            <TextBlock Text="3D Overlay" Foreground="White" FontSize="16" FontWeight="SemiBold"/>
            <Button Content="Click" Margin="0,8,0,0"/>
          </StackPanel>
        </Border>
      </VisualBrush.Visual>
    </VisualBrush>
  </Grid.Resources>

  <Viewport3D>
    <!-- 카메라/조명 생략 -->
    <ModelVisual3D>
      <ModelVisual3D.Content>
        <GeometryModel3D>
          <GeometryModel3D.Geometry>
            <MeshGeometry3D Positions="-1,-0.6,0  1,-0.6,0  1,0.6,0 -1,0.6,0"
                            TriangleIndices="0 1 2  0 2 3"
                            TextureCoordinates="0,1 1,1 1,0 0,0"/>
          </GeometryModel3D.Geometry>
          <GeometryModel3D.Material>
            <DiffuseMaterial Brush="{StaticResource UiBrush}"/>
          </GeometryModel3D.Material>
        </GeometryModel3D>
      </ModelVisual3D.Content>
    </ModelVisual3D>
  </Viewport3D>
</Grid>
```

> **주의**: 이렇게 입힌 버튼은 3D 표면이라 **실제 클릭 포커스/탭 이동**은 작동하지 않습니다(별도 히트테스트/입력 라우팅 필요). 단순 디스플레이나 히트테스트 기반 인터랙션을 구현하세요.

---

## 성능 튜닝(필수 체크리스트)

- **Freezable Freeze**: 변하지 않는 `Brush/Geometry/Transform`는 `Freeze()`(XAML은 내부적으로 Freeze 최적화 수행)
- **모델 병합**: 너무 많은 `GeometryModel3D`는 드로우콜 증가 → 적절히 병합
- **텍스처 크기**: 과도한 해상도/비정형 비율 회피, `BitmapCacheOption` 조정
- **하드웨어 가속**: 시스템 그래픽 가속 상태 확인
- **애니메이션 수**: 동시에 돌아가는 Storyboard 최소화
- **RenderOptions.EdgeMode**/`BitmapScalingMode` 조정(텍스트/이미지 품질 vs 성능)
- **HitTest 범위**: 필요할 때만 수행(마우스 다운 등 이벤트 트리거 기반)

---

# Media & 3D **통합 사례** — “3D TV 벽 + 컨트롤 오버레이”

**요구**
- 뒤쪽 3D 벽에는 **여러 TV 패널**에 서로 다른 동영상 재생
- 앞쪽 오버레이 카드에는 **현재 선택 비디오 정보** 표시
- 마우스로 TV 패널을 클릭하면 해당 패널 **강조/확대**

### 비디오 브러시 리소스

```xml
<Grid.Resources>
  <VisualBrush x:Key="Vid1">
    <VisualBrush.Visual>
      <MediaElement Source="media/a.mp4" LoadedBehavior="Play" Stretch="UniformToFill"/>
    </VisualBrush.Visual>
  </VisualBrush>
  <VisualBrush x:Key="Vid2">
    <VisualBrush.Visual>
      <MediaElement Source="media/b.mp4" LoadedBehavior="Play" Stretch="UniformToFill"/>
    </VisualBrush.Visual>
  </VisualBrush>
</Grid.Resources>
```

### 3D 패널 메시/머티리얼

```xml
<Viewport3D x:Name="View" MouseLeftButtonDown="View_MouseLeftButtonDown">
  <Viewport3D.Camera>
    <PerspectiveCamera Position="0,1.5,6" LookDirection="0,-0.2,-6" UpDirection="0,1,0" FieldOfView="45"/>
  </Viewport3D.Camera>
  <ModelVisual3D>
    <ModelVisual3D.Content>
      <Model3DGroup>
        <DirectionalLight Color="White" Direction="-0.6,-0.5,-1"/>

        <GeometryModel3D x:Name="Panel1">
          <GeometryModel3D.Geometry>
            <MeshGeometry3D Positions="-2,1,0  -0.2,1,0  -0.2,-1,0  -2,-1,0"
                            TriangleIndices="0 1 2  0 2 3"
                            TextureCoordinates="0,0 1,0 1,1 0,1"/>
          </GeometryModel3D.Geometry>
          <GeometryModel3D.Material>
            <DiffuseMaterial Brush="{StaticResource Vid1}"/>
          </GeometryModel3D.Material>
        </GeometryModel3D>

        <GeometryModel3D x:Name="Panel2">
          <GeometryModel3D.Geometry>
            <MeshGeometry3D Positions="0.2,1,0  2,1,0  2,-1,0  0.2,-1,0"
                            TriangleIndices="0 1 2  0 2 3"
                            TextureCoordinates="0,0 1,0 1,1 0,1"/>
          </GeometryModel3D.Geometry>
          <GeometryModel3D.Material>
            <DiffuseMaterial Brush="{StaticResource Vid2}"/>
          </GeometryModel3D.Material>
        </GeometryModel3D>

      </Model3DGroup>
    </ModelVisual3D.Content>
  </ModelVisual3D>
</Viewport3D>
```

### 클릭 시 강조/확대

```csharp
private GeometryModel3D? _selected;
private void View_MouseLeftButtonDown(object sender, MouseButtonEventArgs e)
{
    var pt = e.GetPosition(View);
    var result = VisualTreeHelper.HitTest(View, pt) as RayHitTestResult;
    if (result is RayMeshGeometry3DHitTestResult r)
    {
        if (_selected != null) ResetHighlight(_selected);
        _selected = (GeometryModel3D)r.ModelHit;
        Highlight(_selected);
    }
}
private void Highlight(GeometryModel3D m)
{
    var t = new ScaleTransform3D(1.05, 1.05, 1);
    m.Transform = t;
    if (m.Material is DiffuseMaterial dm) dm.Brush = new SolidColorBrush(Color.FromArgb(200, 255, 255, 255));
}
private void ResetHighlight(GeometryModel3D m)
{
    m.Transform = Transform3D.Identity;
    // 원래 브러시로 복구 (실전에서는 상태 보관)
}
```

---

# 자주 묻는 질문(FAQ)

### Q1. MediaElement로 **H.265/HEVC**가 재생이 안 돼요.

A. OS/코덱 지원 범위를 따릅니다. Windows에 해당 코덱이 없으면 실패합니다. 일반적으로 **H.264(AAC)** 컨테이너(MP4)가 호환성이 높습니다.

### Q2. 자막(SRT)을 **미디어 위에 표시**하려면?

A. 타이머로 `Position`을 읽어 SRT 파싱 결과에서 현재 줄을 찾아 **Overlay TextBlock**에 표시하세요. 자막 렌더/스타일은 XAML로 자유롭게.

### Q3. 3D에서 **스펙큘러 하이라이트**를 더 강하게?

A. `SpecularMaterial`의 `Brush`(밝기)와 `SpecularPower`를 조정하세요. `MaterialGroup`으로 `Diffuse + Specular` 혼합.

### Q4. **퍼포먼스 튜닝** 우선순위는?

A. (1) 애니메이션/모델 수 줄이기 (2) 텍스처/브러시 Freeze/공유 관리 (3) RenderTransform 우선 (4) 히트테스트 최소화 (5) 레이아웃 변화 억제.

---

# 체크리스트(현업용 요약)

- **MediaElement**
  - [ ] `LoadedBehavior="Manual"` + Play/Pause/Stop 제어
  - [ ] `MediaOpened/Ended/Failed` 핸들링
  - [ ] Seek/Volume/SpeedRatio UI 제공
  - [ ] 네트워크 지연/코덱 실패 대비
  - [ ] Video를 **VisualBrush**로 2D/3D 텍스처 활용

- **3D(Viewport3D)**
  - [ ] 카메라/조명/메시/재질 구분 설계
  - [ ] `MeshGeometry3D` UV/Normals 정확히
  - [ ] `Rotate/Scale/Translate` Transform 구성
  - [ ] Storyboard 애니메이션/VSM 필요 시 사용
  - [ ] **히트테스트**로 선택/인터랙션 구현
  - [ ] Freeze/모델 병합/텍스처 최적화

---

## 마무리

이 문서는 WPF에서 **미디어 재생**과 **3D 렌더링**을 안정적으로 구축하는 **실전 가이드**입니다.
여기 있는 스니펫만 조합해도 **비디오 플레이어**, **3D 대시보드/갤러리**, **미디어 파사드** 등 다양한 UI를 빠르게 만들 수 있습니다.

원하시면:
- 프로젝트 구조에 맞춘 **미디어/3D 유틸리티 클래스**,
- **MVVM**과 결합된 **플레이어 컨트롤**,
- **3D 카메라 컨트롤러(Orbit/Pan/Zoom)**,
- **비디오 텍스처 파이프라인**(여러 표면 동기 재생)
