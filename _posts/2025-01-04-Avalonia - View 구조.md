---
layout: post
title: Avalonia - View 구조
date: 2025-01-04 19:20:23 +0900
category: Avalonia
---
# 🖼️ Avalonia의 View 구조와 확장 방법

## 📌 View란?

**View**는 사용자가 직접 보는 UI를 의미하며, Avalonia에서는 `XAML` 형식인 `.axaml` 파일로 정의됩니다. 각 View는 일반적으로 해당하는 `ViewModel`과 바인딩되어 있으며, **MVVM 구조에서 View는 단순히 표시만** 담당합니다.

---

## 🧱 기본 구조

### 예: MainWindow.axaml

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="https://github.com/avaloniaui"
        xmlns:vm="clr-namespace:MyApp.ViewModels"
        xmlns:local="clr-namespace:MyApp.Views"
        x:Class="MyApp.Views.MainWindow"
        Title="Main Window"
        Width="400" Height="300">

    <Window.DataContext>
        <vm:MainWindowViewModel/>
    </Window.DataContext>

    <StackPanel Margin="20">
        <TextBlock Text="Hello Avalonia!" FontSize="24" />
        <Button Content="클릭" Command="{Binding ClickCommand}" Margin="0,10,0,0"/>
    </StackPanel>
</Window>
```

### 관련 코드: MainWindow.axaml.cs

```csharp
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
    }
}
```

---

## 🪄 새 View 만들기

1. **`Views` 폴더 생성**
2. **`.axaml` + `.axaml.cs` 쌍 생성**
3. **ViewModel 연결 (DataContext 설정)**

---

### 🛠 예: `SettingsView.axaml`

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             x:Class="MyApp.Views.SettingsView"
             xmlns:vm="clr-namespace:MyApp.ViewModels">
    <UserControl.DataContext>
        <vm:SettingsViewModel/>
    </UserControl.DataContext>

    <StackPanel>
        <TextBlock Text="{Binding Title}" FontSize="20"/>
        <Button Content="저장" Command="{Binding SaveCommand}" Margin="0,10,0,0"/>
    </StackPanel>
</UserControl>
```

### 📄 `SettingsView.axaml.cs`

```csharp
public partial class SettingsView : UserControl
{
    public SettingsView()
    {
        InitializeComponent();
    }
}
```

---

## 📎 UserControl vs Window

| View 타입 | 설명 | 예시 |
|-----------|------|------|
| `Window` | 독립된 창. 앱의 MainWindow로 사용 | `MainWindow.axaml` |
| `UserControl` | 다른 View나 Layout에 포함되는 재사용 가능한 UI 조각 | `SettingsView.axaml` |

---

## 🪞 View 재사용 예시

`UserControl`로 만든 View를 다른 곳에 삽입 가능:

```xml
<StackPanel>
    <local:SettingsView/>
</StackPanel>
```

- `xmlns:local="clr-namespace:MyApp.Views"` 지정 필요

---

## 🧠 확장 방법

| 방법 | 설명 |
|------|------|
| 🎨 스타일 분리 | 공통 UI 스타일을 리소스 파일로 분리 (예: `Styles/Colors.axaml`) |
| 📏 데이터 템플릿 | ViewModel에 따라 View 자동 매칭 |
| 🗃️ 다중 View 관리 | ContentControl을 사용해 View 동적으로 변경 |
| 📐 Grid, DockPanel, StackPanel 등 다양한 레이아웃 사용 |
| 🧩 사용자 정의 컨트롤 생성 | 복잡한 UI를 재사용 가능한 컴포넌트로 만듦 |

---

## 🔄 View 전환 예시 (ContentControl)

### 📄 MainView.axaml

```xml
<ContentControl Content="{Binding CurrentViewModel}" />
```

### 📄 App.axaml에 템플릿 등록

```xml
<Application.DataTemplates>
    <DataTemplate DataType="{x:Type vm:SettingsViewModel}">
        <views:SettingsView />
    </DataTemplate>
    <DataTemplate DataType="{x:Type vm:MainViewModel}">
        <views:MainView />
    </DataTemplate>
</Application.DataTemplates>
```

---

## 🧪 View를 깔끔하게 설계하려면?

- UI 구조는 가능한 한 XAML에만 작성
- 로직은 ViewModel로 분리 (예: Command, 상태 관리)
- 공통 UI는 `UserControl`로 분리해서 재사용
- 폴더 구조를 View/ → ViewModels/ → Models/로 구분
- 스타일과 리소스는 Styles/ 폴더로 분리

---

## ✅ 정리

| 요소 | 설명 |
|------|------|
| `.axaml` | XAML 기반 UI 정의 |
| `.axaml.cs` | UI의 코드 비하인드 |
| `UserControl` | 재사용 가능한 View |
| `Window` | 독립 실행 View |
| `ContentControl + DataTemplate` | 동적 화면 전환 |
| `DataContext` | ViewModel 연결 |