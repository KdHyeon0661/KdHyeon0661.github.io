---
layout: post
title: WPF - MediaElement & 3D
date: 2025-09-07 18:25:23 +0900
category: WPF
---
# WPF MediaElement와 3D(Viewport3D)

WPF는 멀티미디어 재생과 3D 그래픽을 손쉽게 구현할 수 있는 강력한 기능을 제공합니다. MediaElement를 사용하면 오디오와 비디오를 재생할 수 있고, Viewport3D를 활용하면 하드웨어 가속 3D 장면을 2D UI와 자연스럽게 통합할 수 있습니다. 이 글에서는 초중급 개발자를 대상으로 두 기술의 핵심 개념과 실전 구현 방법을 소개합니다.

## MediaElement: 멀티미디어 재생의 핵심

MediaElement는 WPF에서 오디오와 비디오를 재생하는 가장 기본적인 컨트롤입니다. XAML에서 선언만으로도 미디어 재생 기능을 추가할 수 있습니다.

### 기본 사용법

가장 단순한 미디어 재생 예제는 다음과 같습니다:

```xml
<Grid>
    <MediaElement x:Name="Player"
                  Source="sample.mp4"
                  LoadedBehavior="Manual"
                  UnloadedBehavior="Stop"
                  Stretch="Uniform" />
    
    <StackPanel HorizontalAlignment="Center" VerticalAlignment="Bottom" Margin="20">
        <Button Content="재생" Click="Play_Click" Margin="5"/>
        <Button Content="일시정지" Click="Pause_Click" Margin="5"/>
        <Button Content="정지" Click="Stop_Click" Margin="5"/>
    </StackPanel>
</Grid>
```

```csharp
private void Play_Click(object sender, RoutedEventArgs e) => Player.Play();
private void Pause_Click(object sender, RoutedEventArgs e) => Player.Pause();
private void Stop_Click(object sender, RoutedEventArgs e) => Player.Stop();
```

MediaElement를 선언할 때 `LoadedBehavior`와 `UnloadedBehavior`를 `Manual`로 설정하면 코드에서 직접 재생 제어를 할 수 있습니다. `Stretch` 속성은 영상이 컨트롤 영역에 맞춰지는 방식을 결정합니다.

### 주요 속성과 이벤트

MediaElement를 제대로 활용하려면 핵심 속성과 이벤트를 이해해야 합니다.

| 속성 | 설명 |
|------|------|
| Source | 재생할 미디어 파일의 경로 (로컬, 네트워크, 리소스) |
| Volume | 음량 (0.0 ~ 1.0) |
| SpeedRatio | 재생 속도 (1.0이 기본 속도) |
| Position | 현재 재생 위치 (TimeSpan) |
| NaturalDuration | 미디어의 전체 길이 (Duration) |
| ScrubbingEnabled | 탐색 시 프레임 단위로 즉시 업데이트할지 여부 |

| 이벤트 | 설명 |
|--------|------|
| MediaOpened | 미디어가 성공적으로 열린 후 발생 |
| MediaEnded | 재생이 끝났을 때 발생 |
| MediaFailed | 오류 발생 시 발생 |

### 재생 컨트롤 구현

실제 애플리케이션에서는 재생바, 음량 조절, 속도 변경 등의 기능을 제공하는 것이 일반적입니다.

```xml
<DockPanel VerticalAlignment="Bottom" Background="#22000000" Padding="10">
    <Slider x:Name="SeekSlider" Minimum="0" Maximum="1" 
            ValueChanged="SeekSlider_ValueChanged" DockPanel.Dock="Top"/>
    
    <StackPanel Orientation="Horizontal" HorizontalAlignment="Center">
        <Button Content="▶" Click="Play_Click" Width="40" Margin="5"/>
        <Button Content="⏸" Click="Pause_Click" Width="40" Margin="5"/>
        
        <TextBlock Text="음량:" VerticalAlignment="Center" Margin="10,0,5,0"/>
        <Slider x:Name="VolumeSlider" Minimum="0" Maximum="1" Value="0.8"
                Width="80" ValueChanged="VolumeSlider_ValueChanged"/>
        
        <TextBlock Text="속도:" VerticalAlignment="Center" Margin="10,0,5,0"/>
        <ComboBox x:Name="SpeedComboBox" Width="70" SelectionChanged="SpeedComboBox_SelectionChanged">
            <ComboBoxItem>0.5x</ComboBoxItem>
            <ComboBoxItem>1.0x</ComboBoxItem>
            <ComboBoxItem>1.5x</ComboBoxItem>
            <ComboBoxItem>2.0x</ComboBoxItem>
        </ComboBox>
    </StackPanel>
</DockPanel>
```

```csharp
private DispatcherTimer _progressTimer = new DispatcherTimer { Interval = TimeSpan.FromMilliseconds(100) };

private void UpdateSeekPosition()
{
    if (Player.NaturalDuration.HasTimeSpan)
    {
        double total = Player.NaturalDuration.TimeSpan.TotalSeconds;
        double current = Player.Position.TotalSeconds;
        SeekSlider.Value = current / total;
    }
}

private void SeekSlider_ValueChanged(object sender, RoutedPropertyChangedEventArgs<double> e)
{
    if (Player.NaturalDuration.HasTimeSpan && SeekSlider.IsMouseCaptured)
    {
        var target = TimeSpan.FromSeconds(e.NewValue * Player.NaturalDuration.TimeSpan.TotalSeconds);
        Player.Position = target;
    }
}

private void VolumeSlider_ValueChanged(object sender, RoutedPropertyChangedEventArgs<double> e)
    => Player.Volume = e.NewValue;

private void SpeedComboBox_SelectionChanged(object sender, SelectionChangedEventArgs e)
{
    if (SpeedComboBox.SelectedItem is ComboBoxItem item)
    {
        string speedText = item.Content.ToString().Replace("x", "");
        if (double.TryParse(speedText, out double speed))
            Player.SpeedRatio = speed;
    }
}
```

타이머를 이용해 현재 재생 위치를 슬라이더에 반영하고, 슬라이더 조작 시 `Position`을 변경합니다. `IsMouseCaptured`를 확인하여 사용자가 직접 드래그할 때만 위치를 변경하도록 합니다.

### 다양한 소스에서 미디어 로드

MediaElement는 다양한 경로 형식을 지원합니다:

```xml
<!-- 로컬 절대 경로 -->
<MediaElement Source="C:\Videos\sample.mp4"/>

<!-- 상대 경로 (exe 기준) -->
<MediaElement Source="media/video.mp4"/>

<!-- 임베디드 리소스 (Build Action: Resource) -->
<MediaElement Source="pack://application:,,,/Assets/video.mp4"/>

<!-- 네트워크 스트리밍 -->
<MediaElement Source="http://example.com/stream.mp4"/>
```

네트워크 소스를 사용할 때는 미디어가 완전히 로드될 때까지 약간의 지연이 있을 수 있습니다. `MediaOpened` 이벤트에서 재생을 시작하는 것이 안전합니다.

### 비디오를 브러시로 활용하기

WPF의 `VisualBrush`를 사용하면 비디오를 다양한 표면에 입힐 수 있습니다. 예를 들어 원형 영역에 비디오를 재생하거나 텍스트에 비디오를 입힐 수 있습니다.

```xml
<Grid>
    <Grid.Resources>
        <VisualBrush x:Key="VideoBrush" TileMode="None">
            <VisualBrush.Visual>
                <MediaElement Source="sample.mp4" LoadedBehavior="Play" Stretch="UniformToFill"/>
            </VisualBrush.Visual>
        </VisualBrush>
    </Grid.Resources>
    
    <Ellipse Width="300" Height="300" Fill="{StaticResource VideoBrush}"/>
    
    <TextBlock Text="VIDEO" FontSize="72" FontWeight="Bold"
               HorizontalAlignment="Center" VerticalAlignment="Center">
        <TextBlock.Foreground>
            <StaticResource ResourceKey="VideoBrush"/>
        </TextBlock.Foreground>
    </TextBlock>
</Grid>
```

이 기술은 3D 표면에 텍스처로 적용할 때 특히 유용합니다.

## Viewport3D: WPF에서의 3D 그래픽

WPF는 Direct3D 기반의 하드웨어 가속 3D를 지원하며, `Viewport3D` 컨트롤을 통해 3D 장면을 2D 레이아웃에 포함할 수 있습니다.

### 기본 3D 장면 구성

3D 장면은 크게 카메라, 조명, 3D 모델로 구성됩니다.

```xml
<Viewport3D>
    <Viewport3D.Camera>
        <PerspectiveCamera Position="0,0,5" 
                          LookDirection="0,0,-1" 
                          UpDirection="0,1,0" 
                          FieldOfView="45"/>
    </Viewport3D.Camera>
    
    <ModelVisual3D>
        <ModelVisual3D.Content>
            <Model3DGroup>
                <DirectionalLight Color="White" Direction="-1,-1,-2"/>
                <AmbientLight Color="#404040"/>
            </Model3DGroup>
        </ModelVisual3D.Content>
    </ModelVisual3D>
    
    <ModelVisual3D>
        <ModelVisual3D.Content>
            <GeometryModel3D>
                <GeometryModel3D.Geometry>
                    <MeshGeometry3D
                        Positions="-1,-1,0  1,-1,0  1,1,0  -1,1,0"
                        TriangleIndices="0 1 2  0 2 3"
                        TextureCoordinates="0,1 1,1 1,0 0,0"/>
                </GeometryModel3D.Geometry>
                <GeometryModel3D.Material>
                    <DiffuseMaterial Brush="LightBlue"/>
                </GeometryModel3D.Material>
            </GeometryModel3D>
        </ModelVisual3D.Content>
    </ModelVisual3D>
</Viewport3D>
```

- **카메라**: `PerspectiveCamera`는 원근감 있는 시점을 제공합니다. `Position`은 카메라 위치, `LookDirection`은 바라보는 방향입니다.
- **조명**: `DirectionalLight`는 특정 방향에서 오는 빛, `AmbientLight`는 전체적으로 균일한 빛을 제공합니다.
- **모델**: `MeshGeometry3D`로 정점(Positions)과 삼각형 인덱스(TriangleIndices)를 정의하고, `Material`로 표면 재질을 지정합니다.

### 3D 정육면체 만들기

정육면체는 6개의 면으로 구성되며, 각 면을 두 개의 삼각형으로 표현합니다.

```xml
<MeshGeometry3D x:Key="CubeMesh"
    Positions="
    -1,-1,-1  1,-1,-1  1,1,-1  -1,1,-1   <!-- 뒤면 -->
    -1,-1, 1  1,-1, 1  1,1, 1  -1,1, 1"  <!-- 앞면 -->
    TriangleIndices="
    0 1 2  0 2 3   <!-- 뒤면 -->
    4 5 6  4 6 7   <!-- 앞면 -->
    0 4 5  0 5 1   <!-- 아래면 -->
    2 6 7  2 7 3   <!-- 윗면 -->
    0 3 7  0 7 4   <!-- 왼쪽면 -->
    1 5 6  1 6 2"  <!-- 오른쪽면 -->
    TextureCoordinates="..."/>
```

`TextureCoordinates`는 각 정점에 대응하는 텍스처 좌표를 지정하여 이미지가 어떻게 매핑될지 결정합니다.

### 3D 모델에 애니메이션 적용하기

`RotateTransform3D`와 `DoubleAnimation`을 사용하여 모델을 회전시킬 수 있습니다.

```xml
<GeometryModel3D Geometry="{StaticResource CubeMesh}">
    <GeometryModel3D.Material>
        <DiffuseMaterial Brush="#4066CCFF"/>
    </GeometryModel3D.Material>
    <GeometryModel3D.Transform>
        <RotateTransform3D>
            <RotateTransform3D.Rotation>
                <AxisAngleRotation3D x:Name="CubeRotation" Axis="0,1,0" Angle="0"/>
            </RotateTransform3D.Rotation>
        </RotateTransform3D>
    </GeometryModel3D.Transform>
</GeometryModel3D>

<Viewport3D.Triggers>
    <EventTrigger RoutedEvent="Loaded">
        <BeginStoryboard>
            <Storyboard RepeatBehavior="Forever">
                <DoubleAnimation Storyboard.TargetName="CubeRotation"
                                Storyboard.TargetProperty="Angle"
                                From="0" To="360" Duration="0:0:6"/>
            </Storyboard>
        </BeginStoryboard>
    </EventTrigger>
</Viewport3D.Triggers>
```

### 3D 표면에 비디오 텍스처 입히기

앞서 만든 `VisualBrush`를 3D 모델의 재질로 사용하면 비디오를 입체 표면에 재생할 수 있습니다.

```xml
<Grid.Resources>
    <VisualBrush x:Key="VideoTexture" TileMode="None">
        <VisualBrush.Visual>
            <MediaElement Source="sample.mp4" LoadedBehavior="Play" Stretch="UniformToFill"/>
        </VisualBrush.Visual>
    </VisualBrush>
</Grid.Resources>

<Viewport3D>
    <ModelVisual3D>
        <ModelVisual3D.Content>
            <GeometryModel3D>
                <GeometryModel3D.Geometry>
                    <MeshGeometry3D
                        Positions="-1.6,-0.9,0  1.6,-0.9,0  1.6,0.9,0  -1.6,0.9,0"
                        TriangleIndices="0 1 2  0 2 3"
                        TextureCoordinates="0,1 1,1 1,0 0,0"/>
                </GeometryModel3D.Geometry>
                <GeometryModel3D.Material>
                    <DiffuseMaterial Brush="{StaticResource VideoTexture}"/>
                </GeometryModel3D.Material>
                <GeometryModel3D.Transform>
                    <RotateTransform3D>
                        <RotateTransform3D.Rotation>
                            <AxisAngleRotation3D Axis="0,1,0" Angle="15"/>
                        </RotateTransform3D.Rotation>
                    </RotateTransform3D>
                </GeometryModel3D.Transform>
            </GeometryModel3D>
        </ModelVisual3D.Content>
    </ModelVisual3D>
</Viewport3D>
```

비디오를 3D 평면에 입혀 벽면 TV처럼 표시하거나, 더 복잡한 형태의 모델에도 적용할 수 있습니다.

### 카메라 컨트롤 구현

사용자가 마우스로 3D 장면을 회전하고 확대/축소할 수 있도록 카메라를 제어하는 코드입니다.

```csharp
private PerspectiveCamera _camera;
private Point _lastMousePos;
private double _yaw = 0, _pitch = 0, _distance = 8;
private const double RotSpeed = 0.01, ZoomSpeed = 0.001;

private void Viewport3D_MouseDown(object sender, MouseButtonEventArgs e)
{
    _lastMousePos = e.GetPosition(Viewport3D);
    Mouse.Capture(Viewport3D);
}

private void Viewport3D_MouseMove(object sender, MouseEventArgs e)
{
    if (!Viewport3D.IsMouseCaptured) return;
    var delta = e.GetPosition(Viewport3D) - _lastMousePos;
    _yaw += delta.X * RotSpeed;
    _pitch += delta.Y * RotSpeed;
    _pitch = Math.Clamp(_pitch, -Math.PI / 2 + 0.1, Math.PI / 2 - 0.1);
    _lastMousePos = e.GetPosition(Viewport3D);
}

private void Viewport3D_MouseUp(object sender, MouseButtonEventArgs e)
    => Mouse.Capture(null);

private void Viewport3D_MouseWheel(object sender, MouseWheelEventArgs e)
    => _distance = Math.Clamp(_distance - e.Delta * ZoomSpeed, 2, 20);

private void UpdateCamera()
{
    var dir = new Vector3D(Math.Cos(_pitch) * Math.Sin(_yaw),
                           Math.Sin(_pitch),
                           Math.Cos(_pitch) * Math.Cos(_yaw));
    _camera.Position = -dir * _distance;
    _camera.LookDirection = dir;
}
```

`CompositionTarget.Rendering` 이벤트에서 `UpdateCamera()`를 호출하여 매 프레임 카메라 위치를 갱신합니다.

### 3D 모델 선택 및 상호작용

히트 테스트를 통해 사용자가 클릭한 3D 모델을 식별하고 강조 표시할 수 있습니다.

```csharp
private void Viewport3D_MouseLeftButtonDown(object sender, MouseButtonEventArgs e)
{
    var hit = VisualTreeHelper.HitTest(Viewport3D, e.GetPosition(Viewport3D));
    if (hit is RayMeshGeometry3DHitTestResult meshHit)
    {
        var model = meshHit.ModelHit as GeometryModel3D;
        if (model != null)
            HighlightModel(model);
    }
}
```

선택된 모델에 대해 재질을 변경하거나, 강조 효과를 주는 등의 상호작용을 추가할 수 있습니다.

## 성능 최적화 팁

미디어와 3D를 함께 사용할 때는 성능에 유의해야 합니다.

- **MediaElement 최적화**: `ScrubbingEnabled`는 필요한 경우에만 true로 설정합니다. 네트워크 스트리밍 시 버퍼링 설정을 고려하고, 불필요한 이벤트 핸들러를 제거합니다.
- **3D 그래픽 최적화**: 정적 메시와 브러시는 `Freeze()` 메서드로 고정하여 성능을 향상시킬 수 있습니다. 동일한 지오메트리 인스턴스를 재사용하고, 불필요한 애니메이션을 최소화합니다.
- **메모리 관리**: 미디어 재생이 끝나면 `MediaElement.Close()`를 호출하여 리소스를 해제합니다. 3D 모델 참조를 적시에 정리합니다.

## 결론

WPF의 MediaElement와 Viewport3D는 각각 멀티미디어 재생과 3D 그래픽을 구현하는 강력한 도구입니다. MediaElement는 다양한 소스의 오디오/비디오를 손쉽게 재생할 수 있게 해주며, Viewport3D는 하드웨어 가속 3D 장면을 2D UI와 자연스럽게 통합할 수 있는 프레임워크를 제공합니다.

이 두 기술을 결합하면 비디오 텍스처가 적용된 3D 모델, 인터랙티브한 3D 미디어 플레이어, 동적인 3D 데이터 시각화 등 다양한 고급 시나리오를 구현할 수 있습니다. 실제 프로젝트에서는 하드웨어 성능과 요구사항에 맞게 적절한 최적화를 적용하는 것이 중요합니다.

이 가이드에서 소개한 기본 패턴과 기술들을 활용하면 WPF 기반 애플리케이션에서 풍부한 멀티미디어와 3D 그래픽 기능을 효과적으로 구현할 수 있을 것입니다.