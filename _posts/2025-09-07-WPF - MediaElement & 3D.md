---
layout: post
title: WPF - MediaElement & 3D
date: 2025-09-07 18:25:23 +0900
category: WPF
---
# WPF MediaElement와 3D(Viewport3D) 완전 가이드

이 문서는 WPF에서 멀티미디어 재생과 3D 그래픽을 효과적으로 구현하는 방법을 다룹니다. MediaElement를 통한 오디오/비디오 재생부터 Viewport3D를 활용한 3D 장면 구성까지, 실제 프로젝트에 바로 적용할 수 있는 실용적인 내용을 제공합니다.

## MediaElement: 멀티미디어 재생의 핵심

MediaElement는 WPF에서 오디오와 비디오를 재생하는 가장 간단하면서도 강력한 컨트롤입니다. UI 요소로 직접 사용할 수 있어 XAML에서 쉽게 통합할 수 있습니다.

### 기본 사용법

가장 기본적인 미디어 재생 구현은 다음과 같습니다:

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

### 주요 속성과 이벤트

MediaElement의 핵심적인 속성과 이벤트들을 이해하는 것이 중요합니다:

**주요 속성:**
- `Source`: 재생할 미디어 파일의 경로 (로컬, 네트워크, 임베디드 리소스)
- `LoadedBehavior`/`UnloadedBehavior`: 미디어 로드 시와 언로드 시의 기본 동작 설정
- `Volume`: 음량 (0.0 ~ 1.0)
- `SpeedRatio`: 재생 속도 조절 (1.0이 일반 속도)
- `Position`: 현재 재생 위치
- `Stretch`: 영상 크기 조정 방식

**중요 이벤트:**
- `MediaOpened`: 미디어 파일이 성공적으로 열렸을 때 발생
- `MediaEnded`: 미디어 재생이 끝났을 때 발생
- `MediaFailed`: 미디어 재생 중 오류가 발생했을 때 발생

### 미디어 컨트롤 구현

실제 사용자 인터페이스에서는 재생바, 음량 조절, 재생 속도 선택 등의 컨트롤을 제공하는 것이 일반적입니다:

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
// 재생 위치 업데이트를 위한 타이머
private DispatcherTimer _progressTimer;

public MainWindow()
{
    InitializeComponent();
    
    // 100ms 간격으로 재생 위치 업데이트
    _progressTimer = new DispatcherTimer { Interval = TimeSpan.FromMilliseconds(100) };
    _progressTimer.Tick += (s, e) => UpdateSeekPosition();
}

private void UpdateSeekPosition()
{
    if (Player.NaturalDuration.HasTimeSpan)
    {
        var totalSeconds = Player.NaturalDuration.TimeSpan.TotalSeconds;
        var currentSeconds = Player.Position.TotalSeconds;
        SeekSlider.Value = currentSeconds / totalSeconds;
    }
}

private void SeekSlider_ValueChanged(object sender, RoutedPropertyChangedEventArgs<double> e)
{
    if (Player.NaturalDuration.HasTimeSpan && SeekSlider.IsMouseCaptured)
    {
        var targetTime = TimeSpan.FromSeconds(e.NewValue * Player.NaturalDuration.TimeSpan.TotalSeconds);
        Player.Position = targetTime;
    }
}

private void VolumeSlider_ValueChanged(object sender, RoutedPropertyChangedEventArgs<double> e)
{
    Player.Volume = e.NewValue;
}

private void SpeedComboBox_SelectionChanged(object sender, SelectionChangedEventArgs e)
{
    if (SpeedComboBox.SelectedItem is ComboBoxItem item)
    {
        var speedText = item.Content.ToString().Replace("x", "");
        if (double.TryParse(speedText, out double speed))
        {
            Player.SpeedRatio = speed;
        }
    }
}
```

### 다양한 소스에서 미디어 로드하기

MediaElement는 여러 종류의 소스에서 미디어를 로드할 수 있습니다:

```xml
<!-- 로컬 파일 -->
<MediaElement Source="C:\Videos\sample.mp4"/>

<!-- 상대 경로 -->
<MediaElement Source="media/video.mp4"/>

<!-- 임베디드 리소스 (Build Action을 Resource로 설정) -->
<MediaElement Source="pack://application:,,,/Assets/video.mp4"/>

<!-- 네트워크 스트리밍 -->
<MediaElement Source="http://example.com/stream.mp4"/>
```

## 비디오를 브러시로 활용하기

WPF의 강력한 기능 중 하나는 비디오를 VisualBrush로 변환하여 다양한 표면에 적용할 수 있다는 점입니다:

```xml
<Grid>
    <Grid.Resources>
        <!-- 비디오를 VisualBrush로 변환 -->
        <VisualBrush x:Key="VideoBrush" TileMode="None">
            <VisualBrush.Visual>
                <MediaElement Source="sample.mp4" LoadedBehavior="Play" Stretch="UniformToFill"/>
            </VisualBrush.Visual>
        </VisualBrush>
    </Grid.Resources>
    
    <!-- 원형 영역에 비디오 적용 -->
    <Ellipse Width="300" Height="300" Fill="{StaticResource VideoBrush}"/>
    
    <!-- 텍스트에 비디오 적용 -->
    <TextBlock Text="VIDEO" FontSize="72" FontWeight="Bold" HorizontalAlignment="Center" VerticalAlignment="Center">
        <TextBlock.Foreground>
            <StaticResource ResourceKey="VideoBrush"/>
        </TextBlock.Foreground>
    </TextBlock>
</Grid>
```

이 기술은 3D 표면에 비디오를 텍스처로 입히는 데 특히 유용합니다.

## Viewport3D: WPF에서의 3D 그래픽

WPF는 하드웨어 가속 3D 그래픽을 지원하며, Viewport3D 컨트롤을 통해 2D UI와 자연스럽게 통합할 수 있습니다.

### 기본 3D 장면 구성

간단한 3D 장면은 다음과 같은 요소들로 구성됩니다:

```xml
<Viewport3D>
    <!-- 카메라 설정 -->
    <Viewport3D.Camera>
        <PerspectiveCamera Position="0,0,5" 
                          LookDirection="0,0,-1" 
                          UpDirection="0,1,0" 
                          FieldOfView="45"/>
    </Viewport3D.Camera>
    
    <!-- 조명 -->
    <ModelVisual3D>
        <ModelVisual3D.Content>
            <Model3DGroup>
                <DirectionalLight Color="White" Direction="-1,-1,-2"/>
                <AmbientLight Color="#404040"/>
            </Model3DGroup>
        </ModelVisual3D.Content>
    </ModelVisual3D>
    
    <!-- 3D 모델 -->
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
                    <DiffuseMaterial>
                        <DiffuseMaterial.Brush>
                            <SolidColorBrush Color="LightBlue"/>
                        </DiffuseMaterial.Brush>
                    </DiffuseMaterial>
                </GeometryModel3D.Material>
            </GeometryModel3D>
        </ModelVisual3D.Content>
    </ModelVisual3D>
</Viewport3D>
```

### 3D 정육면체 만들기

더 복잡한 3D 모델을 생성하려면 정점(vertex)와 삼각형 인덱스를 정의해야 합니다:

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
    TextureCoordinates="
    0,1 1,1 1,0 0,0   <!-- 뒤면 UV -->
    0,1 1,1 1,0 0,0   <!-- 앞면 UV -->
    0,1 1,1 1,0 0,0   <!-- 아래면 UV -->
    0,1 1,1 1,0 0,0   <!-- 윗면 UV -->
    0,1 1,1 1,0 0,0   <!-- 왼쪽면 UV -->
    0,1 1,1 1,0 0,0"  <!-- 오른쪽면 UV -->
/>
```

### 3D 모델에 애니메이션 적용하기

3D 모델에 애니메이션을 적용하여 생동감을 줄 수 있습니다:

```xml
<Viewport3D>
    <Viewport3D.Camera>
        <PerspectiveCamera Position="3,3,6" LookDirection="-3,-3,-6" FieldOfView="45"/>
    </Viewport3D.Camera>
    
    <ModelVisual3D>
        <ModelVisual3D.Content>
            <Model3DGroup>
                <DirectionalLight Color="White" Direction="-1,-1,-2"/>
                
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
            </Model3DGroup>
        </ModelVisual3D.Content>
    </ModelVisual3D>
    
    <!-- 회전 애니메이션 -->
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
</Viewport3D>
```

## 3D 표면에 비디오 텍스처 입히기

MediaElement와 Viewport3D를 결합하면 3D 표면에 비디오를 재생하는 효과적인 시각적 경험을 만들 수 있습니다:

```xml
<Grid>
    <Grid.Resources>
        <!-- 비디오를 VisualBrush로 변환 -->
        <VisualBrush x:Key="VideoTexture" TileMode="None">
            <VisualBrush.Visual>
                <MediaElement Source="sample.mp4" 
                            LoadedBehavior="Play" 
                            Stretch="UniformToFill"/>
            </VisualBrush.Visual>
        </VisualBrush>
    </Grid.Resources>
    
    <Viewport3D>
        <Viewport3D.Camera>
            <PerspectiveCamera Position="0,0,4" LookDirection="0,0,-1" FieldOfView="60"/>
        </Viewport3D.Camera>
        
        <ModelVisual3D>
            <ModelVisual3D.Content>
                <Model3DGroup>
                    <DirectionalLight Color="White" Direction="-0.5,-0.5,-1"/>
                    
                    <!-- 비디오 텍스처가 적용된 평면 -->
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
                </Model3DGroup>
            </ModelVisual3D.Content>
        </ModelVisual3D>
    </Viewport3D>
</Grid>
```

## 카메라 컨트롤 구현

사용자가 3D 장면을 인터랙티브하게 탐색할 수 있도록 카메라 컨트롤을 구현하는 방법:

```csharp
public partial class MainWindow : Window
{
    private PerspectiveCamera _camera;
    private Point _lastMousePosition;
    private double _yaw = 0, _pitch = 0, _distance = 8;
    private const double RotationSpeed = 0.01;
    private const double ZoomSpeed = 0.001;
    
    public MainWindow()
    {
        InitializeComponent();
        _camera = (PerspectiveCamera)Viewport3D.Camera;
        
        // 실시간 카메라 업데이트
        CompositionTarget.Rendering += (s, e) => UpdateCamera();
    }
    
    private void UpdateCamera()
    {
        // 카메라 위치 계산
        var direction = new Vector3D(
            Math.Cos(_pitch) * Math.Sin(_yaw),
            Math.Sin(_pitch),
            Math.Cos(_pitch) * Math.Cos(_yaw));
        
        var position = -direction * _distance;
        _camera.Position = new Point3D(position.X, position.Y, position.Z);
        _camera.LookDirection = direction;
        _camera.UpDirection = new Vector3D(0, 1, 0);
    }
    
    private void Viewport3D_MouseDown(object sender, MouseButtonEventArgs e)
    {
        _lastMousePosition = e.GetPosition(Viewport3D);
        Mouse.Capture(Viewport3D);
    }
    
    private void Viewport3D_MouseMove(object sender, MouseEventArgs e)
    {
        if (!Viewport3D.IsMouseCaptured) return;
        
        var currentPosition = e.GetPosition(Viewport3D);
        var deltaX = currentPosition.X - _lastMousePosition.X;
        var deltaY = currentPosition.Y - _lastMousePosition.Y;
        
        _yaw += deltaX * RotationSpeed;
        _pitch += deltaY * RotationSpeed;
        
        // 각도 제한
        _pitch = Math.Clamp(_pitch, -Math.PI / 2 + 0.1, Math.PI / 2 - 0.1);
        
        _lastMousePosition = currentPosition;
    }
    
    private void Viewport3D_MouseUp(object sender, MouseButtonEventArgs e)
    {
        Mouse.Capture(null);
    }
    
    private void Viewport3D_MouseWheel(object sender, MouseWheelEventArgs e)
    {
        _distance = Math.Clamp(_distance - e.Delta * ZoomSpeed, 2, 20);
    }
}
```

```xml
<Viewport3D x:Name="Viewport3D"
            MouseDown="Viewport3D_MouseDown"
            MouseMove="Viewport3D_MouseMove"
            MouseUp="Viewport3D_MouseUp"
            MouseWheel="Viewport3D_MouseWheel">
    <!-- 3D 장면 구성 -->
</Viewport3D>
```

## 3D 모델 선택 및 상호작용

3D 모델과의 상호작용을 위해 히트 테스트를 구현하는 방법:

```csharp
private void Viewport3D_MouseLeftButtonDown(object sender, MouseButtonEventArgs e)
{
    var mousePosition = e.GetPosition(Viewport3D);
    
    // 히트 테스트 수행
    var hitTestResult = VisualTreeHelper.HitTest(Viewport3D, mousePosition);
    
    if (hitTestResult is RayMeshGeometry3DHitTestResult meshHit)
    {
        var selectedModel = meshHit.ModelHit as GeometryModel3D;
        
        if (selectedModel != null)
        {
            // 선택된 모델 강조
            HighlightModel(selectedModel);
            
            // 선택 정보 표시
            ShowModelInfo(selectedModel);
        }
    }
}

private void HighlightModel(GeometryModel3D model)
{
    // 기존 강조 제거
    ResetAllHighlights();
    
    // 새로운 모델 강조
    var highlightMaterial = new DiffuseMaterial(new SolidColorBrush(Colors.Yellow));
    var originalMaterial = model.Material;
    
    // MaterialGroup으로 원본 재질과 강조 효과 결합
    var materialGroup = new MaterialGroup();
    materialGroup.Children.Add(originalMaterial);
    materialGroup.Children.Add(highlightMaterial);
    
    model.Material = materialGroup;
    
    // 강조 상태 저장
    _highlightedModel = model;
    _originalMaterial = originalMaterial;
}
```

## 성능 최적화 팁

WPF에서 미디어와 3D 그래픽을 최적화하는 몇 가지 중요한 팁:

1. **MediaElement 성능 최적화**
   - `ScrubbingEnabled` 속성을 필요한 경우에만 true로 설정
   - 네트워크 스트리밍 시 버퍼링 적절히 구성
   - 불필요한 이벤트 핸들러 최소화

2. **3D 그래픽 성능 최적화**
   - 정적 메시와 브러시는 Freeze() 메서드로 고정
   - 동일한 지오메트리 인스턴스 재사용
   - 불필요한 애니메이션 최소화
   - 텍스처 크기를 적절하게 조정

3. **메모리 관리**
   - 미디어 재생이 끝나면 적절히 정리
   - 3D 모델 참조를 적시에 해제
   - 큰 미디어 파일은 스트리밍 방식으로 처리

## 결론

WPF의 MediaElement와 Viewport3D는 각각 멀티미디어 재생과 3D 그래픽을 구현하는 강력한 도구입니다. MediaElement는 다양한 형식의 오디오와 비디오를 쉽게 재생할 수 있게 해주며, Viewport3D는 하드웨어 가속 3D 그래픽을 2D UI와 자연스럽게 통합할 수 있는 프레임워크를 제공합니다.

이 두 기술을 결합하면 비디오 텍스처가 적용된 3D 모델, 인터랙티브한 3D 미디어 플레이어, 혹은 동적인 3D 데이터 시각화와 같은 고급 사용자 경험을 만들 수 있습니다. 실제 프로젝트에서는 사용자의 하드웨어 성능과 요구사항에 맞게 적절한 최적화를 적용하는 것이 중요합니다.

이 가이드에서 제시한 기본 패턴과 기술들을 활용하면 WPF 기반 애플리케이션에서 풍부한 멀티미디어와 3D 그래픽 기능을 효과적으로 구현할 수 있을 것입니다.