---
layout: post
title: Avalonia - 개요
date: 2024-12-26 19:20:23 +0900
category: Avalonia
---
# 🏗️ Avalonia 기본 템플릿 구조 분석 및 확장 방법

## 📦 템플릿 생성

```bash
dotnet new avalonia.app -o MyAvaloniaApp
cd MyAvaloniaApp
```

이 명령을 실행하면 `MyAvaloniaApp`이라는 기본 앱 구조가 생성됩니다.

---

## 📁 프로젝트 구조

생성된 기본 템플릿은 다음과 같은 구조를 가집니다:

```
MyAvaloniaApp/
├── App.axaml
├── App.axaml.cs
├── MainWindow.axaml
├── MainWindow.axaml.cs
├── Program.cs
├── ViewModels/
│   └── MainWindowViewModel.cs
├── obj/
├── bin/
├── MyAvaloniaApp.csproj
```

---

## 🧩 각 파일 설명

### 📄 `Program.cs`

앱의 **진입점**입니다. Avalonia 앱을 실행하는 `AppBuilder`를 설정합니다.

```csharp
public static class Program
{
    public static void Main(string[] args) =>
        BuildAvaloniaApp().StartWithClassicDesktopLifetime(args);

    public static AppBuilder BuildAvaloniaApp()
        => AppBuilder.Configure<App>()
                     .UsePlatformDetect()
                     .LogToTrace();
}
```

- `UsePlatformDetect()`: OS를 자동으로 감지하여 렌더링 백엔드 설정
- `StartWithClassicDesktopLifetime()`: 윈도우 앱처럼 동작 (닫으면 종료)

---

### 📄 `App.axaml` / `App.axaml.cs`

앱 전체의 **루트 구성** 및 **리소스**, **스타일**, **시작 윈도우** 등을 설정하는 곳입니다.

```xml
<Application xmlns="https://github.com/avaloniaui"
             ...
             x:Class="MyAvaloniaApp.App">
    <Application.Styles>
        <FluentTheme Mode="Light"/>
    </Application.Styles>
</Application>
```

- `FluentTheme`: 기본 테마 (Light, Dark 설정 가능)
- 앱 전역에 적용할 스타일을 정의할 수 있음

```csharp
public class App : Application
{
    public override void OnFrameworkInitializationCompleted()
    {
        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            desktop.MainWindow = new MainWindow
            {
                DataContext = new MainWindowViewModel()
            };
        }

        base.OnFrameworkInitializationCompleted();
    }
}
```

- 앱 실행 시 첫 화면(MainWindow)을 띄우는 부분

---

### 📄 `MainWindow.axaml` / `MainWindow.axaml.cs`

메인 윈도우 (UI 레이아웃)와 그 로직을 담고 있습니다.

```xml
<Window ...>
    <StackPanel>
        <TextBlock Text="{Binding Greeting}" />
    </StackPanel>
</Window>
```

- `MainWindow.axaml`: XAML로 UI 정의
- `MainWindow.axaml.cs`: 코드 비하인드 (UI 이벤트 처리 가능)

---

### 📁 `ViewModels/MainWindowViewModel.cs`

MVVM 패턴의 **ViewModel**로, UI와 데이터 바인딩을 담당합니다.

```csharp
public class MainWindowViewModel : ViewModelBase
{
    public string Greeting => "Welcome to Avalonia!";
}
```

- `ViewModelBase`는 `INotifyPropertyChanged` 구현을 상속한 기본 클래스입니다.

---

## 🔧 확장하는 법

프로젝트를 확장하려면 **View - ViewModel 쌍을 추가**하는 방식으로 확장합니다.

---

### ✅ 1. 새로운 View와 ViewModel 추가

예: `SettingsView.axaml`, `SettingsViewModel.cs`

```bash
mkdir Views ViewModels
```

#### 📄 `Views/SettingsView.axaml`

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             x:Class="MyAvaloniaApp.Views.SettingsView">
    <StackPanel>
        <TextBlock Text="설정 화면입니다."/>
    </StackPanel>
</UserControl>
```

#### 📄 `ViewModels/SettingsViewModel.cs`

```csharp
public class SettingsViewModel : ViewModelBase
{
    public string Title => "설정";
}
```

---

### ✅ 2. ViewModel 연결 및 화면 전환

예를 들어, `MainWindowViewModel`에서 버튼 클릭 시 View를 전환하려면 `ContentControl`과 `DataTemplate`을 활용합니다.

#### 📄 App.axaml에 DataTemplate 등록

```xml
<Application.DataTemplates>
    <DataTemplate DataType="{x:Type vm:MainWindowViewModel}">
        <views:MainWindow/>
    </DataTemplate>
    <DataTemplate DataType="{x:Type vm:SettingsViewModel}">
        <views:SettingsView/>
    </DataTemplate>
</Application.DataTemplates>
```

#### 📄 MainWindow.axaml에 ContentControl 추가

```xml
<ContentControl Content="{Binding CurrentViewModel}" />
```

#### 📄 MainWindowViewModel.cs에 ViewModel 전환 로직 추가

```csharp
private ViewModelBase _currentViewModel = new SettingsViewModel();
public ViewModelBase CurrentViewModel
{
    get => _currentViewModel;
    set => this.RaiseAndSetIfChanged(ref _currentViewModel, value);
}
```

---

## 🧪 정리

| 구성 요소 | 역할 |
|-----------|------|
| `Program.cs` | 앱 실행 초기화 |
| `App.axaml` | 전역 스타일, 초기화 설정 |
| `MainWindow` | 메인 UI 화면 |
| `MainWindowViewModel` | 메인 화면 로직과 상태 관리 |
| `ViewModelBase` | 공통 ViewModel 기능 제공 |

---

## 🚀 다음 단계로는?

- 🧩 사용자 정의 컨트롤 만들기
- 🔄 Navigation 구현
- 📦 DI (의존성 주입)과 Service 구조 추가
- 🧪 Unit Test로 ViewModel 테스트

MVVM 구조를 기준으로 기능을 확장하면 대형 앱도 체계적으로 관리할 수 있습니다!