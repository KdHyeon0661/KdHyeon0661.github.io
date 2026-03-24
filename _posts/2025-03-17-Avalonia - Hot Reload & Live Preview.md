---
layout: post
title: Avalonia - Hot Reload & Live Preview
date: 2025-03-17 19:20:23 +0900
category: Avalonia
---
# Avalonia Hot Reload 및 Live Preview

Avalonia 개발 환경에서는 Hot Reload와 Live Preview를 통해 UI 개발 속도를 크게 향상할 수 있습니다. Hot Reload는 앱 재시작 없이 XAML 및 일부 C# 변경을 즉시 반영하고, Live Preview는 IDE에서 XAML을 실시간으로 렌더링해 줍니다. 이 글에서는 초중급 개발자를 기준으로 Hot Reload와 Live Preview를 설정하고 효과적으로 활용하는 방법을 설명합니다.

---

## 핵심 개념

| 기능 | 설명 |
|------|------|
| **Hot Reload** | XAML 또는 일부 C# 변경 사항을 앱 재시작 없이 런타임에 반영. `dotnet watch`로 구동 |
| **Live Preview** | IDE 패널에서 XAML을 즉시 렌더링. 디자인 타임 데이터와 결합해 복잡한 레이아웃도 미리 확인 |
| **DevTools** | 런타임에 시각적 트리, 바인딩, 리소스, 스타일을 검사·수정. `Avalonia.Diagnostics` 패키지 제공 |

적용 포인트:
- 스타일, 리소스, 템플릿, 레이아웃 조정
- DataTemplate/ControlTemplate 실험
- MVVM 바인딩 점검 (디자인 타임 모델 + Previewer)
- 복잡한 화면(대시보드, 위저드, 다크모드)의 시각적 회전율 극대화

---

## 필수 요구사항 및 설치

### 버전 권장

| 구성 | 권장 |
|------|------|
| Avalonia | 11.x 이상 |
| .NET SDK | 7.0 이상 (6.0도 가능) |
| IDE | JetBrains Rider 최신 / Visual Studio 2022 최신 |
| CLI | `dotnet` 최신 + `dotnet watch` 사용 |

### 프로젝트 설정 (.csproj)

개발 중에는 트리밍이나 단일 파일 배포 옵션을 비활성화해야 Hot Reload와 Previewer가 원활히 동작합니다.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <!-- 컴파일된 XAML: 변경 감지 속도/안정성 향상 -->
    <AvaloniaUseCompiledXaml>true</AvaloniaUseCompiledXaml>
    <!-- 개발 중엔 트리밍/싱글파일 비권장 -->
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

`Avalonia.Diagnostics`는 DevTools(인스펙터)를 제공합니다. 개발 중에만 사용하고 배포 시 제외할 수 있습니다.

---

## 앱 부트스트랩: DevTools 및 Hot Reload 친화 설정

`Program.cs`에서 디버그 빌드 시 DevTools를 활성화합니다.

```csharp
using Avalonia;
using Avalonia.ReactiveUI;

internal static class Program
{
    public static AppBuilder BuildAvaloniaApp()
        => AppBuilder.Configure<App>()
            .UsePlatformDetect()
            .LogToTrace()
            .UseReactiveUI()
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

- Hot Reload는 `dotnet watch`와 Compiled XAML 조합이 핵심입니다.
- DevTools는 `Avalonia.Diagnostics` 패키지 참조만으로 활성화됩니다.

---

## Hot Reload 실행 (권장 워크플로)

### CLI

```bash
dotnet watch
```

- XAML 파일 저장 시 재컴파일되어 실행 중인 앱에 즉시 반영됩니다.
- C# 변경은 부분적으로만 반영되며, 타입 서명이나 생성자 변경 등은 앱 재시작이 필요할 수 있습니다.

### IDE

- **Rider**: 상단 툴바의 **Run with ‘dotnet watch’** 또는 **Hot Reload** 버튼 사용
- **Visual Studio**: Avalonia 확장 설치 후 Debug 세션에서 XAML 편집 시 자동 반영

---

## Live Preview (XAML 미리보기)

### IDE 지원

- **Rider**: `.axaml` 파일을 열면 우측에 **Preview** 탭이 나타납니다.
- **Visual Studio**: Avalonia Extension을 설치하면 미리보기 패널을 사용할 수 있습니다. (Rider가 안정성 면에서 더 우수)

### 디자인 타임 바인딩 필수 패턴

미리보기에서 ViewModel 데이터를 표시하려면 디자인 타임용 DataContext를 별도로 지정합니다.

```xml
<UserControl
    xmlns="https://github.com/avaloniaui"
    xmlns:d="https://github.com/avaloniaui"
    xmlns:vm="clr-namespace:MyApp.ViewModels;assembly=MyApp">

  <!-- 런타임 바인딩 (실제 VM) -->
  <UserControl.DataContext>
    <vm:OrderListViewModel />
  </UserControl.DataContext>

  <!-- 디자인 타임 바인딩 (미리보기 전용) -->
  <UserControl.d:DataContext>
    <vm:OrderListViewModelDesign />
  </UserControl.d:DataContext>

  <StackPanel Spacing="8" Margin="16">
    <TextBlock Text="{Binding Title}" FontSize="24"/>
    <ListBox Items="{Binding Orders}" />
  </StackPanel>
</UserControl>
```

디자인 전용 ViewModel은 더미 데이터를 제공합니다.

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

- Previewer는 `d:DataContext`만 사용하므로, 서비스나 네트워크 접근 없이 UI 구조를 확인할 수 있습니다.
- 복잡한 DataTemplate이나 ItemsPanel 구성도 미리 검증 가능합니다.

---

## Hot Reload와 MVVM 결합

### 간단한 ViewModel 예시

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
- `Spacing`, `Margin`, `FontSize` 같은 레이아웃/스타일 값 변경 → 즉시 반영
- `DataTemplate` / `ControlTemplate`도 즉시 적용

### 스타일/리소스 변경 즉시 반영

`App.axaml`에 전역 스타일을 정의하면, 스타일 수정 시 모든 화면에 동시에 반영됩니다.

```xml
<Application ...>
  <Application.Styles>
    <FluentTheme Mode="Light"/>

    <Style Selector="TextBlock.h1">
      <Setter Property="FontSize" Value="28" />
      <Setter Property="FontWeight" Value="Bold" />
    </Style>
  </Application.Styles>
</Application>
```

```xml
<TextBlock Classes="h1" Text="대제목" />
```

폰트 크기, 색상, 여백 등을 저장할 때마다 미리보기와 실행 중인 앱 모두에서 확인할 수 있습니다.

---

## DevTools(인스펙터)로 런타임 검사

- 실행 중에 **F12** 또는 `Ctrl+Shift+I`를 누르면 DevTools 창이 열립니다 (디버그 빌드 + `Avalonia.Diagnostics` 필요)
- 기능: 시각 트리, 바인딩 상태, 리소스 해석, 측정·정렬 박스, 실시간 값 수정
- 바인딩 에러를 즉시 확인하고, 리소스 키 충돌도 빠르게 파악 가능

Hot Reload와 DevTools를 함께 사용하면 “XAML 수정 → 반영 → DevTools로 검증” 루프를 매우 짧게 유지할 수 있습니다.

---

## 대형 화면 및 복잡 템플릿에서의 팁

1. **Partial Reload 유도**
   - 스타일/템플릿/리소스 파일을 모듈화하여 변경 범위를 최소화합니다.
   - 거대한 파일 하나보다 여러 `.axaml`로 분할하는 것이 좋습니다.

2. **디자인 데이터 정교화**
   - 디자인 모델에 빈 목록, 긴 텍스트, 에러 메시지 등 다양한 경우의 수를 포함해 시각적 회귀를 줄입니다.

3. **가상화/지연 측정 on/off**
   - Previewer에서는 `VirtualizingStackPanel`이나 무거운 애니메이션을 일시적으로 완화하여 성능 이슈를 분리합니다.

---

## Live Preview를 활용한 컴포넌트 설계

### DataTemplate Playground

템플릿을 빠르게 실험하며 스타일, 간격, 아이콘, 상태 표시를 반복 적용할 수 있습니다.

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

### ControlTemplate 수정

버튼 스타일을 템플릿으로 정의하면, 수정 시 모든 버튼에 즉시 반영됩니다.

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

---

## Hot Reload의 한계와 우회 전략

| 항목 | 가능 여부 | 비고 |
|------|-----------|------|
| XAML (레이아웃/스타일/템플릿) | 매우 잘 됨 | Hot Reload의 주요 대상 |
| 바인딩 경로/VM 속성 추가/변경 | 상당 부분 가능 | VM 재생성 필요 시 일부 제한 |
| C# 로직 (뷰 코드비하인드/서비스) | 제한적 | 메서드 바디 변경은 반영되나, 타입 서명/생성자 변경은 재시작 필요 |
| 리소스/정적 확장/마크업 확장 | 잘 됨 | 리소스 병합·분리로 Partial Reload 유도 |
| 네이티브/플랫폼 초기화 | 불가 | 앱 재시작 필요 |

**실무 팁**
- C# 변경은 핵심 타입을 건드리지 않고 작은 단위로 진행합니다.
- VM 교체가 필요한 설계는 디자인-프렌들리한 생성자를 유지하고 Previewer로 먼저 검증합니다.
- 타입 이름이나 제네릭 서명 변경 시 즉시 재시작하여 상태 꼬임을 방지합니다.

---

## IDE별 병행 전략

- **Rider**: Previewer 안정성과 성능이 좋아 XAML 설계에 최적
- **Visual Studio**: Avalonia Extension 최신 버전 유지, Previewer 에러 발생 시 “Rebuild → 다시 열기”
- **CLI**: 팀 공통 환경으로 `dotnet watch` 파이프라인을 문서화

---

## 모듈 구조와 핫리로드 최적화

```
MyApp/
├── App.axaml            # 전역 리소스/테마
├── Styles/              # 컴포넌트별 스타일 묶음
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

- 스타일/템플릿/리소스를 화면과 분리하면 스타일만 교체하며 빠르게 실험 가능
- 각 모듈에 **Design-ViewModel**을 동봉해 Previewer에서 독립적으로 열어 검증

---

## 디버깅 및 트러블슈팅

### Previewer 빈 화면 / 예외

- 디자인 타임 DataContext가 없으면 빈 화면이 나올 수 있습니다. `d:DataContext`를 반드시 추가하세요.
- 정적 생성자나 서비스 초기화에서 예외가 발생하면 디자인 모드에서 분기 처리합니다.

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
        // 런타임 초기화 (파일/네트워크/DI)
    }
}
```

### Hot Reload가 반영되지 않는 경우

- 파일 저장이 실제 디스크에 반영되었는지 확인
- `AvaloniaUseCompiledXaml`이 `true`인지 확인
- 큰 변경(C# 타입 서명 등)은 앱 재시작 필요

### DevTools 미표시

- `Avalonia.Diagnostics` 패키지 설치 확인
- 디버그 빌드인지 확인 (릴리즈/트리밍 환경에서는 비활성)

---

## CI/팀 온보딩을 위한 스크립트

```bash
# 개발 서버 (Hot Reload)
dev:
	dotnet watch --project src/MyApp.Desktop

# 빠른 클린/빌드
re:
	dotnet clean && dotnet build -c Debug
```

팀 가이드:
- Rider/VS에서 `.axaml` 우측 Previewer 사용
- 복잡 화면은 디자인 뷰모델을 우선 설계 → Previewer로 완성 → 런타임 서비스 결합

---

## 고급: 다크/라이트 테마 실시간 전환

리소스를 테마별로 분리하고, 테마 매니저를 통해 동적으로 교체하면 Hot Reload와 결합해 테마 개발 속도를 높일 수 있습니다.

```xml
<!-- App.axaml -->
<Application.Styles>
  <FluentTheme Mode="{Binding ThemeMode, Source={x:Static vm:ThemeManager.Instance}}"/>

  <ResourceDictionary x:Key="LightPalette">
    <SolidColorBrush x:Key="BrandBrush" Color="#335CFF" />
  </ResourceDictionary>

  <ResourceDictionary x:Key="DarkPalette">
    <SolidColorBrush x:Key="BrandBrush" Color="#7BA7FF" />
  </ResourceDictionary>
</Application.Styles>
```

---

## 핵심 체크리스트

- [ ] `.csproj`에 `AvaloniaUseCompiledXaml=true` 설정
- [ ] `Avalonia.Diagnostics` 패키지 설치, 디버그 빌드에서 DevTools 사용
- [ ] `dotnet watch`로 Hot Reload 루프 가동
- [ ] **디자인 타임 ViewModel**(`d:DataContext`)로 Previewer 품질 확보
- [ ] 리소스/스타일/템플릿 모듈화로 Partial Reload 효율 향상
- [ ] C# 변경은 작게 빈번히 저장하고, 타입 서명 변경 시 재시작
- [ ] Previewer / DevTools / Hot Reload 세 가지를 동시에 활용

---

## 결론

Hot Reload와 Live Preview는 Avalonia 개발에서 **UI 피드백 루프**를 획기적으로 줄여줍니다. XAML, 리소스, 템플릿 중심의 설계를 채택하면 저장 즉시 반영되는 “짧은 피드백 루프”를 만들 수 있고, 디자인 타임 데이터와 DevTools를 결합하면 시각적 버그와 바인딩 오류를 개발 초기에 제거할 수 있습니다. 대형 화면, 복잡 템플릿, 테마 실험도 모듈화와 Partial Reload로 충분히 쾌적하게 작업할 수 있습니다.

이러한 도구들을 적극 활용해 보다 빠르고 안정적인 Avalonia 애플리케이션을 개발하시기 바랍니다.