---
layout: post
title: Avalonia - MVVM
date: 2025-01-03 20:20:23 +0900
category: Avalonia
---
# 🧭 MVVM 패턴이란? (Model-View-ViewModel)

## 📌 MVVM의 정의

**MVVM(Model-View-ViewModel)**은 UI 애플리케이션에서 **관심사 분리(Separation of Concerns)**를 실현하는 디자인 패턴입니다.  
특히 XAML 기반 UI 프레임워크(WPF, Avalonia, Xamarin, MAUI 등)에서 많이 사용됩니다.

MVVM은 UI 로직(View)과 비즈니스 로직(Model)을 명확히 분리하며, **ViewModel**을 중간 매개체로 둬 두 영역이 직접적으로 연결되지 않도록 합니다.

---

## 🧱 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| **Model** | 실제 데이터 구조 및 비즈니스 로직을 담당 (예: DB, API 등) |
| **View** | 사용자에게 보여지는 UI. XAML로 작성됨 |
| **ViewModel** | View와 Model 사이에서 데이터를 중계하고 상태를 관리 |

---

## 🔄 흐름 구조

```
[사용자] → View ↔ ViewModel ↔ Model
                         ↑
                  (Command, Binding)
```

- **View ↔ ViewModel**: 바인딩을 통해 데이터와 명령을 연결 (양방향 가능)
- **ViewModel ↔ Model**: 데이터 요청, 가공, 저장 등의 로직 처리

---

## 🧠 왜 MVVM을 사용할까?

| 이유 | 설명 |
|------|------|
| ✅ **UI와 로직 분리** | 유지보수가 쉬워지고 테스트가 용이 |
| ✅ **재사용성** | ViewModel만 재사용해 다양한 UI에 적용 가능 |
| ✅ **테스트 용이성** | View를 제외하고 단위 테스트 가능 |
| ✅ **XAML Binding 최적화** | View와 ViewModel 간 데이터 자동 연결 (INotifyPropertyChanged 기반) |

---

## 💡 Avalonia에서 MVVM 사용하는 예시

### 1. Model: 단순 데이터 클래스

```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}
```

---

### 2. ViewModel: 데이터와 로직 관리

```csharp
using ReactiveUI;

public class MainWindowViewModel : ReactiveObject
{
    private string _name = "홍길동";

    public string Name
    {
        get => _name;
        set => this.RaiseAndSetIfChanged(ref _name, value);
    }

    public ReactiveCommand<Unit, Unit> GreetCommand { get; }

    public MainWindowViewModel()
    {
        GreetCommand = ReactiveCommand.Create(() =>
        {
            Name = $"안녕하세요, {Name}!";
        });
    }
}
```

---

### 3. View (MainWindow.axaml)

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="https://github.com/avaloniaui"
        xmlns:vm="clr-namespace:MyApp.ViewModels"
        x:Class="MyApp.Views.MainWindow"
        Title="MVVM Demo"
        Width="400" Height="200">

    <Window.DataContext>
        <vm:MainWindowViewModel/>
    </Window.DataContext>

    <StackPanel HorizontalAlignment="Center" VerticalAlignment="Center">
        <TextBox Text="{Binding Name}" Width="200"/>
        <Button Content="인사하기" Command="{Binding GreetCommand}" Margin="0,10,0,0"/>
    </StackPanel>
</Window>
```

---

### 4. View 연결 코드 (MainWindow.axaml.cs)

```csharp
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        // DataContext는 XAML에서 설정했기 때문에 생략 가능
    }
}
```

---

## 🛠 MVVM 구현 팁

- `INotifyPropertyChanged`를 반드시 ViewModel에 구현해야 합니다.
- Avalonia에서는 `ReactiveUI`를 많이 활용합니다 (`ReactiveObject`, `ReactiveCommand` 등).
- ViewModel은 View에 의존하지 않도록 설계해야 합니다 (완전한 테스트 가능).

---

## 📚 마무리

MVVM은 UI 애플리케이션을 **구조적으로 설계**하고 **유지보수를 쉽게** 만드는 데 큰 도움이 되는 디자인 패턴입니다. Avalonia는 MVVM을 자연스럽게 지원하는 구조이므로, 이 패턴에 익숙해진다면 복잡한 앱도 체계적으로 개발할 수 있습니다.