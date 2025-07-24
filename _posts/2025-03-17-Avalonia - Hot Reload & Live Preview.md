---
layout: post
title: Avalonia - Hot Reload & Live Preview
date: 2025-03-17 19:20:23 +0900
category: Avalonia
---
# 🔁 Avalonia Hot Reload / Live Preview 사용법

Avalonia는 실시간 UI 변경 미리보기를 지원하여, 개발자가 **XAML/C# 코드 변경을 즉시 UI에서 확인**할 수 있게 해줍니다.  
이 기능을 통해 생산성이 크게 향상되며, UI/UX를 실시간으로 조정할 수 있습니다.

---

## 🎯 목표

| 항목 | 설명 |
|------|------|
| 🔁 Hot Reload | XAML/C# 코드 변경 시 앱 재시작 없이 반영 |
| 🔍 Live Preview | IDE 또는 별도 창에서 UI 실시간 확인 |
| 🧪 실시간 변경 | 레이아웃/스타일/바인딩 즉시 확인 가능 |

---

## 1️⃣ Hot Reload (핫 리로드)

### 🔧 설치 요구사항

| 항목 | 버전 |
|------|------|
| Avalonia | `11.0.0-preview5` 이상 권장 |
| .NET SDK | `6.0` 또는 `7.0` |
| 개발 환경 | Rider, VS 2022+, CLI |

---

### ✅ 기본 설정

프로젝트의 `.csproj` 파일에 다음을 추가:

```xml
<PropertyGroup>
  <AvaloniaUseCompiledXaml>true</AvaloniaUseCompiledXaml>
</PropertyGroup>
```

그리고 패키지 확인:

```bash
dotnet add package Avalonia.Desktop
dotnet add package Avalonia.Diagnostics
```

> `Avalonia.Diagnostics`는 핫리로드 및 인스펙터 도구 제공

---

### ▶️ 실행 방법 (CLI 기준)

```bash
dotnet watch
```

> `dotnet watch` 명령은 변경 감지를 통해 자동으로 앱을 재실행합니다.

---

### 🔥 작동 방식

- XAML 파일 저장 시 자동 컴파일 + UI에 즉시 반영
- ViewModel 바인딩도 동적으로 적용됨
- C# 코드도 부분 적용 가능 (일부 한계 있음)

---

## 2️⃣ Live Preview (XAML Previewer)

### 🔧 Rider Previewer (권장)

JetBrains Rider는 Avalonia 전용 Previewer를 내장 제공:

- `.axaml` 파일 열기 → 우측에 "Preview" 탭 자동 활성화
- ViewModel이 설정된 경우 `d:DataContext`에 따라 바인딩 표시

---

### ✅ XAML 미리보기 시 준비사항

1. View 파일에 `d:DataContext` 추가:

```xml
<UserControl
  xmlns="https://github.com/avaloniaui"
  xmlns:d="https://github.com/avaloniaui"
  xmlns:vm="clr-namespace:MyApp.ViewModels;assembly=MyApp">

  <UserControl.d:DataContext>
    <vm:MyViewModelDesign />
  </UserControl.d:DataContext>
</UserControl>
```

2. Previewer가 자동으로 `MyViewModelDesign` 인스턴스를 사용해 렌더링

---

### ⚠️ Visual Studio에서는?

- Avalonia Extension 설치 후 Preview 기능 제공
- 단, Rider에 비해 안정성이 떨어질 수 있음

설치 링크: [Avalonia VS 확장](https://marketplace.visualstudio.com/items?itemName=AvaloniaTeam.AvaloniaforVisualStudio)

---

## 3️⃣ 인스펙터 및 Hot Reload UI

핫리로드 중 앱을 오른쪽 클릭하거나 단축키로 UI 인스펙터 열기:

```csharp
AppBuilder.Configure<App>()
    .UsePlatformDetect()
    .LogToTrace()
    .With(new AvaloniaNativePlatformOptions { UseGpu = true })
    .With(new Win32PlatformOptions { EnableMultitouch = true })
    .With(new X11PlatformOptions { UseGpu = true })
    .WithApplicationLifetime(new ClassicDesktopStyleApplicationLifetime())
    .UseReactiveUI()
    .SetupWithHotReload(); // 여기에 HotReload 연결
```

---

## 4️⃣ HotReload와 MVVM 구조 함께 쓰기

ViewModel 바인딩 구조도 변경이 즉시 반영됩니다:

```csharp
public class MainWindowViewModel : ViewModelBase
{
    public string Title => "🔁 실시간 반영되는 타이틀";
}
```

```xml
<TextBlock Text="{Binding Title}" FontSize="20"/>
```

→ 저장하면 바로 Preview/실행 앱에 반영됨

---

## 5️⃣ 사용 시 주의사항

| 항목 | 내용 |
|------|------|
| ✅ 대부분 XAML 즉시 반영 | 컨트롤/스타일/바인딩 등 |
| ⚠️ C# 코드 변경은 제한적 | 종종 앱 재시작 필요 |
| ⚠️ 앱이 크면 느려질 수 있음 | 대규모 프로젝트에서는 Previewer 최적화 필요 |

---

## 6️⃣ 사용 팁

- `dotnet watch` + `d:DataContext`로 설계 시 반복 속도 대폭 향상
- ViewModel-First 설계 시에도 미리보기용 ViewModel을 따로 둘 것
- 커스텀 컨트롤 설계 시에도 Previewer가 반응함

---

## ✅ 결론

| 기능 | 효과 |
|------|------|
| 🔁 Hot Reload | 코드 저장 시 자동 반영, 재시작 필요 없음 |
| 🔍 Live Preview | XAML 기반 UI 설계 가속 |
| 🎯 생산성 향상 | UI 구현 속도 최대 2~3배 향상 |

→ **Avalonia의 핫리로드/프리뷰는 MVVM 패턴과 강력히 결합되어 생산성 극대화에 유리합니다.**