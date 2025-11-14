---
layout: post
title: Avalonia - Hot Reload & Live Preview
date: 2025-03-17 19:20:23 +0900
category: Avalonia
---
# Avalonia Hot Reload / Live Preview

## 개념과 적용 시나리오

| 기능 | 핵심 |
|---|---|
| **Hot Reload** | XAML 또는 일부 C# 변경 사항을 **앱 재시작 없이** 런타임에 반영. 보통 `dotnet watch`로 구동 |
| **Live Preview(Previewer)** | IDE 패널/독립 창에서 XAML을 **즉시 렌더링**. 디자인 데이터와 결합하면 복잡한 레이아웃도 즉시 확인 |
| **DevTools(인스펙터)** | 런타임에 시각적 트리/바인딩/리소스/스타일을 검사·수정. `Avalonia.Diagnostics` 필요 |

적용 포인트:
- 스타일/리소스/템플릿/레이아웃 조정
- DataTemplate/ControlTemplate 실험
- MVVM 바인딩 점검(디자인타임 모델 + Previewer)
- 복잡한 화면(대시보드, 위저드, 다크모드) **시각적 회전율** 극대화

---

## 필수 요구사항 및 설치

### 버전 권장치

| 구성 | 권장 |
|---|---|
| Avalonia | 11.x 이상 |
| .NET SDK | 7.0 이상(6.0도 가능) |
| IDE | JetBrains Rider 최신 / Visual Studio 2022 최신 |
| CLI | `dotnet` 최신 + `dotnet watch` 사용 |

### 프로젝트 설정(.csproj)

핵심 속성들(컴파일된 XAML, 트림/싱글파일 시 미리보기 예외 등은 개발 중 비활성화 권장):

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>

    <!-- 컴파일된 XAML: 변경 감지 속도/안정성 향상 -->
    <AvaloniaUseCompiledXaml>true</AvaloniaUseCompiledXaml>

    <!-- 개발 중엔 트리밍/싱글파일 비권장 (Hot Reload/Previewer 영향) -->
    <PublishTrimmed>false</PublishTrimmed>
    <PublishSingleFile>false</PublishSingleFile>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Avalonia" Version="11.*" />
    <PackageReference Include="Avalonia.Desktop" Version="11.*" />
    <PackageReference Include="Avalonia.ReactiveUI" Version="11.*" />
    <PackageReference Include="Avalonia.Diagnostics" Version="11.*" />
  </ItemGroup>
</Project>
```

> `Avalonia.Diagnostics`는 DevTools, 바인딩/리소스/시각 트리 검사를 제공한다(개발 전용으로 두고 배포 시 제외 가능).

---

## 앱 부트스트랩: DevTools/Hot Reload 친화 설정

`Program.cs`(또는 `AppBuilder`)에서 플랫폼 옵션과 DevTools 연결:

```csharp
using Avalonia;
using Avalonia.ReactiveUI;

internal static class Program
{
    // Rider/VS Previewer가 사용할 엔트리
    public static AppBuilder BuildAvaloniaApp()
        => AppBuilder.Configure<App>()
            .UsePlatformDetect()
            .LogToTrace()
            .UseReactiveUI()             // MVVM/Bindings 강화
            .With(new Win32PlatformOptions { EnableMultitouch = true })
            .With(new X11PlatformOptions { UseGpu = true })
            .With(new AvaloniaNativePlatformOptions { UseGpu = true });

    [STAThread]
    public static void Main(string[] args)
    {
#if DEBUG

        // DevTools: 런타임에 F12 또는 Ctrl+Shift+I로 열 수 있음
        BuildAvaloniaApp().StartWithClassicDesktopLifetime(args, ShutdownMode.OnLastWindowClose);
#else

        BuildAvaloniaApp().StartWithClassicDesktopLifetime(args);
#endif

    }
}
```

> Avalonia 11 기준으로 Hot Reload는 **`dotnet watch`**와 **Compiled XAML** 조합이 핵심이며, DevTools는 `Avalonia.Diagnostics` 설치로 활성화된다.
> 일부 템플릿에서는 `AttachDevTools()` 호출이 보일 수 있다. 11.x에서는 DevTools 패키지 참조만으로도 동작한다(IDE/런타임 조합에 따라 다를 수 있으니 프로젝트 템플릿에 맞춰 유지).

---

## Hot Reload 실행(권장 워크플로)

### CLI

```bash
dotnet watch
```

- XAML 저장 즉시 재컴파일 → 실행 중 앱에 반영
- C# 변경은 부분적으로 반영되며, 경우에 따라 재시작이 필요(자세한 제한은 아래 §10).

### IDE

- Rider: 상단 툴바의 **Run with ‘dotnet watch’** 또는 “Hot Reload” 버튼 활성
- Visual Studio: Avalonia 확장 설치 후 **Hot Reload**/Debug 세션에서 XAML 편집→반영

---

## Live Preview(Previewer) 극대화

### XAML 미리보기 패널

- Rider: `.axaml` 열면 우측 **Preview** 탭 활성
- VS: Avalonia Extension 설치 후 Preview 사용 가능(안정성은 Rider가 우수한 편)

### 디자인타임 바인딩 필수 패턴

```xml
<UserControl
    xmlns="https://github.com/avaloniaui"
    xmlns:d="https://github.com/avaloniaui"
    xmlns:vm="clr-namespace:MyApp.ViewModels;assembly=MyApp">

  <!-- 런타임 바인딩 -->
  <UserControl.DataContext>
    <vm:OrderListViewModel />
  </UserControl.DataContext>

  <!-- 디자인타임 바인딩 -->
  <UserControl.d:DataContext>
    <vm:OrderListViewModelDesign />
  </UserControl.d:DataContext>

  <StackPanel Spacing="8" Margin="16">
    <TextBlock Text="{Binding Title}" FontSize="24"/>
    <ListBox Items="{Binding Orders}" />
  </StackPanel>
</UserControl>
```

디자인 전용 ViewModel:

```csharp
public class OrderListViewModelDesign : OrderListViewModel
{
    public OrderListViewModelDesign()
    {
        Title = "디자인 타이틀";
        Orders = new ObservableCollection<string>
        {
            "주문#1001 - 준비 중",
            "주문#1002 - 배송 중",
            "주문#1003 - 완료"
        };
    }
}
```

- Previewer는 **디자인타임 DataContext**만 사용하므로, 서비스/네트워크 접근 없이도 UI 구조 확인 가능
- 복잡한 DataTemplate/ItemsPanel/Virtualization 조합을 안전하게 설계

---

## MVVM과 Hot Reload 결합

### 가벼운 ViewModel 예시

```csharp
public class MainWindowViewModel : ReactiveUI.ReactiveObject
{
    private string _title = "Hot Reload 데모";
    public string Title
    {
        get => _title;
        set => this.RaiseAndSetIfChanged(ref _title, value);
    }

    public ObservableCollection<string> Logs { get; } = new();

    public void Add(string message) => Logs.Add(message);
}
```

XAML:

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="{Binding Title}" Width="800" Height="450">

  <StackPanel Spacing="8" Margin="16">
    <Button Content="로그 추가" Command="{Binding AddLogCommand}"/>
    <ItemsControl Items="{Binding Logs}"/>
  </StackPanel>
</Window>
```

Hot Reload 시나리오:
- `Spacing`, `Margin`, `FontSize` 같은 스타일/레이아웃 값을 바꾸면 **바로 반영**
- `DataTemplate`/`ControlTemplate`도 즉시 적용되어 시각적 실험 속도 ↑

### 스타일/리소스 변경 즉시 반영

`App.axaml`:

```xml
<Application
  xmlns="https://github.com/avaloniaui"
  xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
  x:Class="MyApp.App">

  <Application.Styles>
    <FluentTheme Mode="Light"/>

    <!-- 리소스/스타일: Hot Reload로 즉시 반영됨 -->
    <Style Selector="TextBlock.h1">
      <Setter Property="FontSize" Value="28" />
      <Setter Property="FontWeight" Value="Bold" />
    </Style>
  </Application.Styles>
</Application>
```

View:

```xml
<TextBlock Classes="h1" Text="대제목" />
```

폰트/색상/여백을 미세 조정하며 결과를 즉시 확인할 수 있다.

---

## DevTools(인스펙터)로 런타임 검사

- 실행 중 **F12** 또는 `Ctrl+Shift+I`로 DevTools 오픈(디버그 빌드 + `Avalonia.Diagnostics`)
- 기능: 시각 트리/바인딩 상태/리소스 해석/측정·정렬 박스/실시간 수정
- 바인딩 에러를 즉시 확인하고, 리소스 키 충돌·선언 중복도 빠르게 파악 가능

> Hot Reload와 DevTools를 함께 쓰면 “XAML 수정→반영→DevTools로 즉시 검증” 루프를 만들 수 있다.

---

## 대형 화면/복잡 템플릿에서의 팁

1) **Partial Reload**를 유도
- 스타일/템플릿/리소스 파일을 **모듈화**하여 변경 범위를 최소화
- 거대한 파일 하나 대신 여러 `.axaml`로 분할

2) **디자인 데이터 정교화**
- 디자인 모델에 **경우의 수**(빈 목록, 긴 텍스트, 에러 메시지)를 포함해 시각적 회귀를 줄인다.

3) **가상화/지연 측정 on/off**
- Previewer에서는 `VirtualizingStackPanel`이나 무거운 애니메이션을 일시적으로 완화하여 그림자 성능 이슈를 분리한다.

---

## Live Preview와 컴포넌트 설계(패턴)

### DataTemplate Playground

```xml
<UserControl ...>
  <UserControl.Resources>
    <DataTemplate x:Key="OrderItemTemplate" DataType="{x:Type vm:OrderItem}">
      <StackPanel Orientation="Horizontal" Spacing="8">
        <TextBlock Text="{Binding Id}"/>
        <TextBlock Text="{Binding Status}"/>
      </StackPanel>
    </DataTemplate>
  </UserControl.Resources>

  <ListBox Items="{Binding Orders}" ItemTemplate="{StaticResource OrderItemTemplate}"/>
</UserControl>
```

템플릿을 빠르게 실험하면서 스타일/간격/아이콘/상태 표시를 반복 적용.

### ControlTemplate 수정

```xml
<Style Selector="Button.theme-primary">
  <Setter Property="Template">
    <ControlTemplate>
      <Border CornerRadius="8" Background="{DynamicResource PrimaryBrush}">
        <ContentPresenter Margin="12,8" HorizontalAlignment="Center"/>
      </Border>
    </ControlTemplate>
  </Setter>
</Style>
```

버튼 모양을 바꾸면 **모든 화면**에서 즉시 확인 가능하다.

---

## Hot Reload의 한계와 우회 전략

| 항목 | 가능한가 | 메모 |
|---|---|---|
| XAML(레이아웃/스타일/템플릿) | 매우 잘 됨 | Hot Reload의 1급 시민 |
| 바인딩 경로/VM 속성 추가/변경 | 상당 부분 가능 | VM 재생성 필요 시가 있음 |
| C# 로직(뷰 코드비하인드/서비스) | 제한적 | 메서드 바디 변경은 일부 반영되나, 타입 서명/제네릭/생성자 변경 등은 **재시작** 필요 |
| 리소스/정적 확장/마크업 확장 | 잘 됨 | 리소스 병합·분리로 Partial Reload 유도 |
| 네이티브/플랫폼 초기화 | 불가 | 앱 재시작 필요 |

**실무 팁**
- C# 변경은 “핵심 루프/핵심 타입”을 건드리지 않고 **작은 단위**로 진행
- VM 교체가 필요한 설계면, “디자인-프렌들리”한 VM 생성자를 유지하고 Previewer로 먼저 검증
- 치명적 변경(타입 이름·제네릭 서명 등)은 즉시 재시작하여 상태 꼬임을 방지

---

## Rider / Visual Studio / CLI 병행 전략

- **Rider**: Previewer 안정/성능이 좋아 XAML 설계에 최적
- **VS**: Avalonia Extension 최신 버전 유지, Previewer 에러 발생 시 “Rebuild → 다시 열기”
- **CLI**: 팀원/CI에서 공통의 “참고 환경”으로 `dotnet watch` 파이프라인을 문서화

---

## 팀/모듈 구조와 핫리로드 최적화

```
MyApp/
├── App.axaml            # 전역 리소스/테마
├── Styles/              # 컴포넌트/도메인별 스타일 묶음
│   ├── Buttons.axaml
│   ├── Lists.axaml
│   └── Charts.axaml
├── Views/               # 화면 XAML
│   ├── Orders/
│   └── Settings/
└── ViewModels/
    ├── Orders/
    └── Settings/
```

- **스타일/템플릿/리소스**를 화면과 분리 → 스타일만 교체하며 회전율 ↑
- 각 모듈에 **Design-ViewModel** 동봉 → Previewer에서 독립적으로 열어 검증

---

## 샘플: 빠른 스타일 실험 루프

### `App.axaml`

```xml
<Application xmlns="https://github.com/avaloniaui"
             x:Class="MyApp.App">
  <Application.Styles>
    <FluentTheme Mode="Light"/>

    <!-- 실험용 팔레트 -->
    <SolidColorBrush x:Key="BrandBrush" Color="#335CFF" />
    <Style Selector="Button.brand">
      <Setter Property="Foreground" Value="White"/>
      <Setter Property="Background" Value="{DynamicResource BrandBrush}"/>
      <Setter Property="CornerRadius" Value="8"/>
      <Setter Property="Padding" Value="12,8"/>
    </Style>
  </Application.Styles>
</Application>
```

### View

```xml
<StackPanel Spacing="10" Margin="20">
  <TextBlock Text="색/모서리/폰트 실험" FontSize="18" />
  <Button Classes="brand" Content="확인"/>
</StackPanel>
```

- 색상/CornerRadius/폰트를 Hot Reload로 바꿔가며 즉시 확인
- DevTools로 런타임 측정/정렬/리소스 확인

---

## 디버깅/트러블슈팅

### Previewer 빈 화면/예외

- **디자인타임 DataContext**가 없는 경우: `d:DataContext` 추가
- 정적 생성자/서비스 초기화에서 예외 발생: `Design.IsDesignMode`로 분기

```csharp
using Avalonia.Controls;
using Avalonia.Controls.Platform;

public class OrdersViewModel
{
    public OrdersViewModel()
    {
        if (Design.IsDesignMode)
        {
            // 디자인 모드: 더미 데이터/서비스 사용
            return;
        }
        // 런타임 초기화(파일/네트워크/DI)
    }
}
```

### Hot Reload가 반영되지 않는 경우

- 파일 저장이 IDE에서 실제 디스크로 내려가는지 확인
- `AvaloniaUseCompiledXaml`가 `true`인지 확인
- 큰 변경(C# 타입 서명 등)은 앱 재시작

### DevTools 미표시

- `Avalonia.Diagnostics` 패키지 확인
- 디버그 빌드인지 확인(릴리즈/싱글파일/트리밍 환경은 DevTools가 비활성일 수 있음)

---

## CI/팀 온보딩을 위한 스크립트

`Makefile` 또는 PowerShell로 **공통 실행** 정의:

```bash
# 개발 서버(Hot Reload)

dev:
	dotnet watch --project src/MyApp.Desktop

# 빠른 클린/빌드

re:
	dotnet clean && dotnet build -c Debug
```

팀 가이드:
- Rider/VS에서 `.axaml` 우측 Previewer를 사용
- 복잡 화면은 디자인 뷰모델을 우선 설계 → Previewer로 완성 → 런타임 서비스 결합

---

## 고급: 리소스 테마 스위치(다크/라이트)도 실시간

```xml
<!-- App.axaml에 두 테마를 ResourceDictionary로 분리 -->
<Application.Styles>
  <FluentTheme Mode="{Binding ThemeMode, Source={x:Static vm:ThemeManager.Instance}}"/>

  <!-- 커스텀 팔레트 Light -->
  <ResourceDictionary x:Key="LightPalette">
    <SolidColorBrush x:Key="BrandBrush" Color="#335CFF" />
  </ResourceDictionary>

  <!-- 커스텀 팔레트 Dark -->
  <ResourceDictionary x:Key="DarkPalette">
    <SolidColorBrush x:Key="BrandBrush" Color="#7BA7FF" />
  </ResourceDictionary>
</Application.Styles>
```

테마 매니저가 `ResourceInclude`를 갈아끼우는 방식으로 테마를 바꾸면, Hot Reload와 결합해 **테마 개발 속도**가 급격히 빨라진다.

---

## 체크리스트(핵심 요약)

- `.csproj`에 `AvaloniaUseCompiledXaml=true`
- `Avalonia.Diagnostics` 설치 + 디버그에서 DevTools 사용
- `dotnet watch`로 Hot Reload 루프 가동
- **디자인타임 ViewModel**(`d:DataContext`)로 Previewer 품질 확보
- 리소스/스타일/템플릿 **모듈화**로 Partial Reload 효율 ↑
- C# 변경은 “작게”/“빈번히 저장”하고, 타입 서명 변경 시 재시작
- Previewer/DevTools/Hot Reload 3종을 **동시에** 활용

---

## 결론

Hot Reload/Live Preview는 Avalonia에서 **UI 회전율**을 극대화하는 핵심 도구다.
- XAML·리소스·템플릿 중심의 설계를 채택하면, 저장 즉시 반영되는 “짧은 피드백 루프”를 만들 수 있다.
- 디자인타임 데이터와 DevTools를 결합하면 **시각적 버그/바인딩 오류**를 개발 초기에 제거할 수 있다.
- 대형 화면/복잡 템플릿/테마 실험도 모듈화와 Partial Reload로 충분히 쾌적하게 작업 가능하다.
