---
layout: post
title: WPF - DataBinding과 ViewModel
date: 2025-09-02 16:25:23 +0900
category: WPF
---
# DataBinding과 ViewModel 연결 완전 가이드

WPF에서 **DataBinding**은 View(XAML)와 ViewModel(C#)을 **느슨하게 결합**시켜 UI를 자동 갱신합니다.
핵심은 **DataContext**(바인딩의 기본 원본)와 **INotifyPropertyChanged / ObservableCollection / ICommand**입니다.

---

## DataContext란? (바인딩의 출발점)

- **DataContext**는 해당 요소와 그 **자식 트리**가 바인딩할 **기본 데이터 원본**입니다.
- 어느 컨트롤에도 `DataContext`를 지정하지 않으면 **부모의 DataContext를 상속**받습니다.
- **ItemsControl(DataTemplate 내부)**에 들어가면 DataContext는 **해당 아이템**으로 바뀝니다.

예시:
```xml
<Window ...>
  <!-- 이 아래 자식들은 기본적으로 MainViewModel을 DataContext로 사용 -->
  <Window.DataContext>
    <vm:MainViewModel/>
  </Window.DataContext>

  <StackPanel>
    <TextBlock Text="{Binding Title}"/>
    <ListBox ItemsSource="{Binding Orders}">
      <!-- 여기 내부(DataTemplate) DataContext는 각 Order 객체 -->
      <ListBox.ItemTemplate>
        <DataTemplate>
          <TextBlock Text="{Binding Id}"/>
        </DataTemplate>
      </ListBox.ItemTemplate>
    </ListBox>
  </StackPanel>
</Window>
```

---

## ViewModel 기본 골격

ViewModel은 **프레젠테이션 상태**와 **동작**(ICommand)을 노출합니다.

```csharp
using System.Collections.ObjectModel;
using System.ComponentModel;
using System.Runtime.CompilerServices;
using System.Windows.Input;

public class MainViewModel : INotifyPropertyChanged
{
    private string _title = "주문 목록";
    public string Title
    {
        get => _title;
        set { _title = value; OnPropertyChanged(); }
    }

    public ObservableCollection<Order> Orders { get; } = new();

    public ICommand RefreshCommand { get; }
    public ICommand DeleteCommand { get; }

    public MainViewModel()
    {
        RefreshCommand = new RelayCommand(_ => Load());
        DeleteCommand  = new RelayCommand(o => Delete((Order)o!), o => o is Order);
    }

    private void Load()
    {
        Orders.Clear();
        Orders.Add(new Order { Id = "A-001", Total = 120_000m });
        Orders.Add(new Order { Id = "A-002", Total =  89_000m });
    }

    private void Delete(Order o) => Orders.Remove(o);

    public event PropertyChangedEventHandler? PropertyChanged;
    protected void OnPropertyChanged([CallerMemberName] string? n = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(n));
}

public class Order { public string Id { get; set; } = ""; public decimal Total { get; set; } }
```

> 컬렉션은 **ObservableCollection**으로, 단일 속성은 **INPC**로 변경 알림을 제공해야 UI가 자동 갱신됩니다.

---

## View와 ViewModel 연결 방법

### XAML에서 직접 생성 (가장 간단)

```xml
<Window ...>
  <Window.DataContext>
    <vm:MainViewModel/>
  </Window.DataContext>
  ...
</Window>
```

### 코드비하인드에서 주입/설정

```csharp
public partial class MainWindow : Window
{
    public MainWindow(MainViewModel vm) // DI 생성자 주입 or 수동 생성
    {
        InitializeComponent();
        DataContext = vm; // 또는 new MainViewModel();
    }
}
```

### DI(Generic Host)로 구성(실무 추천)

```csharp
// App.xaml.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

public partial class App : Application
{
    private IHost? _host;
    protected override void OnStartup(StartupEventArgs e)
    {
        _host = Host.CreateDefaultBuilder(e.Args)
            .ConfigureServices(s =>
            {
                s.AddSingleton<MainViewModel>();
                s.AddSingleton<MainWindow>();
            }).Build();

        var win = _host.Services.GetRequiredService<MainWindow>();
        win.DataContext = _host.Services.GetRequiredService<MainViewModel>();
        win.Show();
    }
}
```

### ViewModel-First (DataTemplate 매핑)

뷰는 생성하지 않고 **뷰모델만 Content**로 바인딩하면 **DataTemplate**이 알맞은 뷰를 자동 매핑합니다.
```xml
<!-- App.xaml 전역 DataTemplate -->
<ResourceDictionary ... xmlns:views="clr-namespace:MyApp.Views" xmlns:vm="clr-namespace:MyApp.ViewModels">
  <DataTemplate DataType="{x:Type vm:DashboardViewModel}">
    <views:DashboardView/>
  </DataTemplate>
</ResourceDictionary>
```
```xml
<!-- 어디서든 -->
<ContentControl Content="{Binding CurrentViewModel}"/>
```

---

## 바인딩 핵심 구문 & 옵션

### 기본

```xml
<TextBox Text="{Binding UserName, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}" />
<TextBlock Text="{Binding Total, StringFormat='총액 {0:C}'}" />
```
- **Mode**: OneWay / TwoWay / OneTime / OneWayToSource
- **UpdateSourceTrigger**: PropertyChanged / LostFocus / Explicit
- **StringFormat**: 표시 형식 지정

### Fallback/Null 처리 & 지연 업데이트

```xml
<TextBlock Text="{Binding Title, TargetNullValue='(제목 없음)', FallbackValue='(로딩 중)'}"/>
<TextBox Text="{Binding Query, UpdateSourceTrigger=PropertyChanged, Delay=300}" />
```
- **FallbackValue**: 경로 불일치(바인딩 실패) 시 표시
- **TargetNullValue**: 결과가 null일 때만 대체
- **Delay**: 값 전파를 ms 단위로 지연(타이핑 필터 UX 개선)

### 다른 요소/조상 참조

```xml
<!-- 형제 요소 값 바인딩 -->
<TextBox x:Name="Input"/>
<TextBlock Text="{Binding Text, ElementName=Input}"/>

<!-- 조상(Window)의 DataContext(=MainViewModel) 접근 -->
<TextBlock Text="{Binding DataContext.Title,
  RelativeSource={RelativeSource AncestorType=Window}}"/>
```

### Converter / MultiBinding

```xml
<Window.Resources>
  <local:BoolToVisibilityConverter x:Key="BoolToVisibility"/>
</Window.Resources>

<TextBlock Visibility="{Binding IsBusy, Converter={StaticResource BoolToVisibility}}"/>

<TextBlock>
  <TextBlock.Text>
    <MultiBinding StringFormat="{}{0} / {1}">
      <Binding Path="DoneCount"/>
      <Binding Path="TotalCount"/>
    </MultiBinding>
  </TextBlock.Text>
</TextBlock>
```

### Command와 파라미터

```xml
<Button Content="삭제"
        Command="{Binding DeleteCommand}"
        CommandParameter="{Binding SelectedItem, ElementName=OrderList}"/>

<ListBox x:Name="OrderList" ItemsSource="{Binding Orders}"/>
```

---

## ItemsControl / DataTemplate 안에서의 “컨텍스트 전환”

- `ItemsControl` 내부의 `DataTemplate` **DataContext는 각 아이템**입니다.
- **상위 뷰모델 커맨드**를 호출하려면 **AncestorType** 또는 **x:Reference**를 사용합니다.

```xml
<ListView ItemsSource="{Binding Orders}">
  <ListView.ItemTemplate>
    <DataTemplate>
      <StackPanel Orientation="Horizontal">
        <TextBlock Text="{Binding Id}" Margin="0,0,6,0"/>
        <Button Content="삭제"
                Command="{Binding DataContext.DeleteCommand,
                          RelativeSource={RelativeSource AncestorType=Window}}"
                CommandParameter="{Binding .}"/>
      </StackPanel>
    </DataTemplate>
  </ListView.ItemTemplate>
</ListView>
```

---

## 유효성 검사(Validation)와 바인딩

ViewModel에서 **IDataErrorInfo** 또는 **INotifyDataErrorInfo**를 구현하면, XAML에서 검증 UI를 쉽게 붙일 수 있습니다.

```xml
<TextBox Text="{Binding Age, UpdateSourceTrigger=PropertyChanged,
                ValidatesOnDataErrors=True, NotifyOnValidationError=True}">
  <TextBox.Style>
    <Style TargetType="TextBox">
      <Style.Triggers>
        <Trigger Property="Validation.HasError" Value="True">
          <Setter Property="BorderBrush" Value="Red"/>
          <Setter Property="ToolTip"
                  Value="{Binding RelativeSource={RelativeSource Self},
                          Path=(Validation.Errors)[0].ErrorContent}"/>
        </Trigger>
      </Style.Triggers>
    </Style>
  </TextBox.Style>
</TextBox>
```

---

## 디자인 타임 데이터 (미리보기 개선)

디자이너에서 보기 좋게:
```xml
<Window xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        mc:Ignorable="d">
  <Window.DataContext>
    <vm:MainViewModel/>
  </Window.DataContext>
  <!-- 디자인 전용 DataContext (런타임 무시) -->
  <d:Window.DataContext>
    <vm:Design.DesignMainViewModel/>
  </d:Window.DataContext>
  ...
</Window>
```

---

## 디버깅 & 성능 팁

- **바인딩 오류**는 출력 창에 기록됩니다. 빠르게 찾으려면:
```xml
<TextBlock Text="{Binding Title, PresentationTraceSources.TraceLevel=High}"/>
```
- **Live Visual Tree / Live Property Explorer**로 런타임 바인딩/리소스/시각 트리를 검사.
- **TwoWay + PropertyChanged 남발**은 비용이 큼 → 필요한 곳에만 사용, 나머지는 OneWay/OneTime.
- 긴 작업은 **AsyncRelayCommand**로 비동기화하고 **CanExecute**로 재진입을 막으세요.

---

## 흔한 문제 & 체크리스트

- (문제) 컨트롤 안에서 바인딩이 동작하지 않음
  → (확인) 그 위치의 **DataContext가 무엇인지** Live Visual Tree로 확인
- (문제) 버튼 활성화 갱신이 늦음
  → (해결) 관련 속성 setter에서 `RelayCommand.RaiseCanExecuteChanged()` 호출
- (문제) 리스트가 그려지지 않음
  → (해결) `ObservableCollection`인지 확인(단순 `List<T>`는 변경 알림 없음)
- (문제) 상위 VM 속성을 템플릿 내부에서 못 참조
  → (해결) `RelativeSource AncestorType` 또는 `x:Reference` 사용

---

## 최소 예제(요약)

**MainViewModel.cs**
```csharp
public class MainViewModel : INotifyPropertyChanged
{
    public ObservableCollection<Order> Orders { get; } = new();
    private Order? _selected;
    public Order? Selected { get => _selected; set { _selected = value; OnPropertyChanged(); } }

    public RelayCommand RefreshCommand { get; }
    public RelayCommand DeleteCommand  { get; }

    public MainViewModel()
    {
        RefreshCommand = new RelayCommand(_ => Load());
        DeleteCommand  = new RelayCommand(_ => { if (Selected != null) Orders.Remove(Selected); },
                                          _ => Selected != null);
    }

    private void Load()
    {
        Orders.Clear();
        Orders.Add(new Order{ Id="X-01", Total=1000m });
        Orders.Add(new Order{ Id="X-02", Total=2500m });
    }

    public event PropertyChangedEventHandler? PropertyChanged;
    void OnPropertyChanged([CallerMemberName] string? n=null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(n));
}
public class Order { public string Id { get; set; }=""; public decimal Total { get; set; } }
```

**MainWindow.xaml**
```xml
<Window ...>
  <Window.DataContext>
    <vm:MainViewModel/>
  </Window.DataContext>

  <DockPanel Margin="16">
    <StackPanel Orientation="Horizontal" DockPanel.Dock="Top" Margin="0,0,0,8">
      <Button Content="새로고침" Command="{Binding RefreshCommand}"/>
      <Button Content="삭제" Command="{Binding DeleteCommand}" Margin="8,0,0,0"/>
    </StackPanel>

    <ListView ItemsSource="{Binding Orders}" SelectedItem="{Binding Selected, Mode=TwoWay}">
      <ListView.View>
        <GridView>
          <GridViewColumn Header="Id" DisplayMemberBinding="{Binding Id}"/>
          <GridViewColumn Header="Total" DisplayMemberBinding="{Binding Total, StringFormat={}{0:C}}"/>
        </GridView>
      </ListView.View>
    </ListView>
  </DockPanel>
</Window>
```

---

### 정리

- **DataContext**로 뷰와 뷰모델을 연결하고,
- **INPC / ObservableCollection / ICommand**로 변경·컬렉션·동작을 노출하며,
- **Binding 옵션/Converter/RelativeSource**로 복잡한 UI 요구사항을 선언형으로 해결하세요.

이 구조를 지키면 **테스트 가능한 설계**와 **유지보수 쉬운 UI**를 얻을 수 있습니다.
원하시면 위 코드를 기반으로 하는 **MVVM 스타터 템플릿**도 만들어 드릴게요.
