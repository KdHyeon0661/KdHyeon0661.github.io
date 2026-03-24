---
layout: post
title: WPF - DataBinding과 ViewModel
date: 2025-09-02 16:25:23 +0900
category: WPF
---
# DataBinding과 ViewModel

## WPF MVVM 패턴의 핵심: DataBinding과 ViewModel

WPF의 DataBinding은 UI와 데이터를 분리하면서도 강력하게 연결해주는 핵심 기술입니다. MVVM(Model-View-ViewModel) 패턴은 이 DataBinding을 극대화하여 유지보수성과 테스트 용이성을 높입니다. 이 글에서는 초중급 개발자를 대상으로 DataBinding의 기본 개념부터 ViewModel 연결까지 실무에 바로 적용할 수 있는 핵심 내용을 다룹니다.

---

## DataContext: 바인딩의 시작점과 상속

### DataContext란?

DataContext는 WPF에서 바인딩이 참조하는 기본 데이터 소스입니다. 마치 "지금 이 요소가 어떤 데이터와 연결되어 있는지"를 알려주는 역할을 합니다. 중요한 특징은 **상속**입니다. 상위 요소에 DataContext를 설정하면 하위 요소들은 자동으로 같은 DataContext를 사용합니다.

```xml
<!-- DataContext 상속 예시 -->
<Window DataContext="{Binding MainViewModel}">
    <StackPanel>
        <!-- Window의 DataContext를 상속받아 MainViewModel.Title에 바인딩 -->
        <TextBlock Text="{Binding Title}"/>
        
        <!-- ListBox의 ItemsSource는 MainViewModel.Orders 컬렉션 -->
        <ListBox ItemsSource="{Binding Orders}">
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <!-- 템플릿 내부에서는 개별 Order 객체가 DataContext -->
                    <TextBlock Text="{Binding OrderId}"/>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>
    </StackPanel>
</Window>
```

이처럼 DataContext는 계층 구조를 따라 흐르기 때문에, 상위에서 한 번 설정해두면 하위 요소에서 반복해서 설정할 필요가 없습니다.

### DataContext의 명시적 참조

때로는 상속된 DataContext가 아닌 다른 요소의 데이터를 참조해야 할 때가 있습니다. 이때는 `ElementName`이나 `RelativeSource`를 사용합니다.

```xml
<Grid>
    <TextBox x:Name="InputBox" Text="검색어"/>
    <Button Content="검색" 
            Command="{Binding SearchCommand}"
            CommandParameter="{Binding Text, ElementName=InputBox}"/>
    
    <!-- 부모 윈도우의 DataContext에 있는 명령 참조 -->
    <Button Content="저장" 
            Command="{Binding DataContext.SaveCommand, 
                      RelativeSource={RelativeSource AncestorType=Window}}"/>
</Grid>
```

---

## ViewModel: 프레젠테이션 로직의 중심

### ViewModel의 역할

ViewModel은 View가 보여줄 데이터와 사용자 명령을 담당합니다. 핵심은 **View에 대한 의존성을 제거**하는 것입니다. ViewModel은 순수 C# 클래스이며, UI 요소(Button, TextBox 등)를 직접 알지 못합니다. 대신 속성과 명령을 통해 상호작용합니다.

### INotifyPropertyChanged 구현

ViewModel에서 가장 중요한 인터페이스는 `INotifyPropertyChanged`입니다. 속성값이 변경되면 이벤트를 발생시켜 View에 알려줍니다.

```csharp
public class MainViewModel : INotifyPropertyChanged
{
    private string _userName;
    public string UserName
    {
        get => _userName;
        set
        {
            if (_userName != value)
            {
                _userName = value;
                OnPropertyChanged();
            }
        }
    }
    
    public event PropertyChangedEventHandler PropertyChanged;
    protected virtual void OnPropertyChanged([CallerMemberName] string propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

`CallerMemberName`을 사용하면 속성 이름을 문자열로 직접 전달하지 않아도 됩니다. 이는 리팩토링에 안전하고 오타를 줄여줍니다.

### ObservableCollection 사용

목록 데이터를 바인딩할 때는 `ObservableCollection<T>`를 사용합니다. 이 컬렉션은 항목이 추가/제거될 때 자동으로 UI를 갱신해줍니다.

```csharp
public ObservableCollection<Product> Products { get; } = new ObservableCollection<Product>();

// 항목 추가 시 UI에 즉시 반영됨
Products.Add(new Product { Name = "새 제품" });
```

### ICommand를 통한 사용자 명령

버튼 클릭 등의 사용자 동작은 `ICommand` 인터페이스로 처리합니다. 가장 간단한 구현은 `RelayCommand`입니다.

```csharp
public class RelayCommand : ICommand
{
    private readonly Action _execute;
    private readonly Func<bool> _canExecute;
    
    public RelayCommand(Action execute, Func<bool> canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }
    
    public bool CanExecute(object parameter) => _canExecute?.Invoke() ?? true;
    public void Execute(object parameter) => _execute();
    
    public event EventHandler CanExecuteChanged
    {
        add { CommandManager.RequerySuggested += value; }
        remove { CommandManager.RequerySuggested -= value; }
    }
}
```

```csharp
// ViewModel에서 명령 정의
public ICommand SaveCommand { get; }

public MainViewModel()
{
    SaveCommand = new RelayCommand(OnSave, CanSave);
}

private void OnSave() { /* 저장 로직 */ }
private bool CanSave() => !string.IsNullOrWhiteSpace(UserName);
```

---

## 뷰와 뷰모델의 연결

### 생성자 주입 방식

의존성 주입(DI)을 사용하면 뷰와 뷰모델을 깔끔하게 연결할 수 있습니다. App.xaml.cs에서 호스트를 구성하고, 뷰의 생성자에서 필요한 뷰모델을 받습니다.

```csharp
// App.xaml.cs (간략)
private IHost _host;

protected override void OnStartup(StartupEventArgs e)
{
    _host = Host.CreateDefaultBuilder()
        .ConfigureServices(services =>
        {
            services.AddSingleton<MainViewModel>();
            services.AddSingleton<MainWindow>();
        })
        .Build();
    
    var mainWindow = _host.Services.GetRequiredService<MainWindow>();
    mainWindow.Show();
}
```

```csharp
// MainWindow.xaml.cs
public MainWindow(MainViewModel viewModel)
{
    InitializeComponent();
    DataContext = viewModel;
}
```

### DataTemplate을 이용한 자동 연결

뷰모델 타입에 따라 자동으로 뷰를 선택하게 할 수도 있습니다. 리소스 사전에 `DataTemplate`을 정의하면 `ContentControl`이 해당 뷰모델에 맞는 뷰를 렌더링합니다.

```xml
<Application.Resources>
    <DataTemplate DataType="{x:Type local:MainViewModel}">
        <local:MainView/>
    </DataTemplate>
</Application.Resources>

<!-- 메인 윈도우에서 -->
<ContentControl Content="{Binding CurrentViewModel}"/>
```

---

## 바인딩의 고급 기법

### 값 변환기(Converter)

바인딩 시 데이터를 표시하기 전에 변환이 필요하면 `IValueConverter`를 사용합니다.

```csharp
public class BoolToVisibilityConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
    {
        return (bool)value ? Visibility.Visible : Visibility.Collapsed;
    }
    
    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
    {
        return (Visibility)value == Visibility.Visible;
    }
}
```

```xml
<Window.Resources>
    <local:BoolToVisibilityConverter x:Key="BoolToVis"/>
</Window.Resources>

<Button Visibility="{Binding IsVisible, Converter={StaticResource BoolToVis}}"/>
```

### 다중 바인딩

여러 속성을 조합하여 하나의 값을 만들 때는 `MultiBinding`을 사용합니다.

```xml
<TextBlock>
    <TextBlock.Text>
        <MultiBinding StringFormat="{}{0} - {1}">
            <Binding Path="FirstName"/>
            <Binding Path="LastName"/>
        </MultiBinding>
    </TextBlock.Text>
</TextBlock>
```

### 비동기 명령

비동기 작업이 필요한 경우 `AsyncRelayCommand`를 구현해 사용자 경험을 개선할 수 있습니다.

```csharp
public class AsyncRelayCommand : ICommand
{
    private readonly Func<Task> _execute;
    private bool _isExecuting;
    
    public AsyncRelayCommand(Func<Task> execute)
    {
        _execute = execute;
    }
    
    public bool CanExecute(object parameter) => !_isExecuting;
    
    public async void Execute(object parameter)
    {
        _isExecuting = true;
        RaiseCanExecuteChanged();
        try
        {
            await _execute();
        }
        finally
        {
            _isExecuting = false;
            RaiseCanExecuteChanged();
        }
    }
    
    public event EventHandler CanExecuteChanged;
    public void RaiseCanExecuteChanged() => CanExecuteChanged?.Invoke(this, EventArgs.Empty);
}
```

---

## 디버깅과 성능

### 바인딩 오류 확인

출력 창(Output)에 바인딩 오류가 표시됩니다. `PresentationTraceSources`를 사용하면 더 자세한 정보를 볼 수 있습니다.

```xml
<TextBlock Text="{Binding UserName, PresentationTraceSources.TraceLevel=High}"/>
```

### 성능 최적화

대용량 리스트는 가상화를 활성화하여 성능을 높입니다.

```xml
<ListBox VirtualizingStackPanel.IsVirtualizing="True"
         VirtualizingStackPanel.VirtualizationMode="Recycling"
         ScrollViewer.CanContentScroll="True">
</ListBox>
```

입력 지연이 필요할 때는 `Delay` 속성을 사용합니다.

```xml
<TextBox Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged, Delay=500}"/>
```

---

## 결론

WPF의 DataBinding과 ViewModel은 **관심사 분리**와 **데이터 중심 설계**라는 중요한 원칙을 구현합니다. 초중급 개발자가 이 개념을 제대로 이해하면 다음과 같은 이점을 얻을 수 있습니다.

| 장점 | 설명 |
|------|------|
| 테스트 용이성 | ViewModel은 UI에 독립적이므로 단위 테스트가 쉽습니다. |
| 유지보수성 | UI와 로직이 분리되어 변경 사항의 영향 범위가 줄어듭니다. |
| 재사용성 | ViewModel을 다른 View에서도 재사용할 수 있습니다. |

처음에는 DataContext와 INotifyPropertyChanged, ICommand만으로 시작해도 충분합니다. 작은 예제부터 적용하면서 점차 복잡한 패턴을 도입하는 것이 좋습니다. WPF의 DataBinding은 현대적인 UI 프레임워크에서도 통용되는 강력한 패턴이므로, 이 기본기를 잘 익혀두면 다른 기술로 확장하기도 수월합니다.