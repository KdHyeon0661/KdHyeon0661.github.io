---
layout: post
title: WPF - UserControl vs CustomControl
date: 2025-09-07 15:25:23 +0900
category: WPF
---
# UserControl vs CustomControl

WPF에서 재사용 가능한 UI 컴포넌트를 만들 때 가장 먼저 마주하는 선택은 **UserControl**을 만들 것인가, **CustomControl**을 만들 것인가입니다. 이 글에서는 두 방식의 차이점을 이해하고, 각각을 어떤 상황에서 사용해야 하는지 초중급 개발자의 시각에서 정리합니다.

---

## UserControl: 기존 컨트롤을 조립하기

UserControl은 기존의 WPF 컨트롤(TextBox, Button, ComboBox 등)을 XAML에서 조합하여 만드는 컴포넌트입니다. 마치 레고 블록을 조립하듯, 여러 컨트롤을 하나로 묶어 새로운 기능을 제공합니다.

### 간단한 예: 주소 입력 컴포넌트

주소 입력을 위한 컴포넌트를 UserControl로 만들어보겠습니다. 우편번호 검색 버튼, 주소 자동완성 콤보박스, 상세 주소 입력란을 하나로 묶습니다.

```xml
<!-- AddressInput.xaml -->
<UserControl x:Class="MyApp.AddressInput"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <StackPanel>
        <Grid>
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="*"/>
                <ColumnDefinition Width="Auto"/>
            </Grid.ColumnDefinitions>
            <TextBox x:Name="ZipCodeBox" Text="{Binding ZipCode, RelativeSource={RelativeSource AncestorType=UserControl}}"/>
            <Button Grid.Column="1" Content="검색" Click="SearchButton_Click"/>
        </Grid>
        <ComboBox ItemsSource="{Binding AddressSuggestions, RelativeSource={RelativeSource AncestorType=UserControl}}"
                  DisplayMemberPath="FullAddress"
                  SelectedItem="{Binding SelectedAddress, RelativeSource={RelativeSource AncestorType=UserControl}}"/>
        <TextBox Text="{Binding StreetAddress, RelativeSource={RelativeSource AncestorType=UserControl}}"/>
        <TextBox Text="{Binding DetailAddress, RelativeSource={RelativeSource AncestorType=UserControl}}"/>
    </StackPanel>
</UserControl>
```

```csharp
// AddressInput.xaml.cs
public partial class AddressInput : UserControl
{
    public AddressInput()
    {
        InitializeComponent();
    }

    // 의존 속성 정의
    public static readonly DependencyProperty ZipCodeProperty =
        DependencyProperty.Register(nameof(ZipCode), typeof(string), typeof(AddressInput));
    public static readonly DependencyProperty StreetAddressProperty =
        DependencyProperty.Register(nameof(StreetAddress), typeof(string), typeof(AddressInput));
    public static readonly DependencyProperty DetailAddressProperty =
        DependencyProperty.Register(nameof(DetailAddress), typeof(string), typeof(AddressInput));
    public static readonly DependencyProperty AddressSuggestionsProperty =
        DependencyProperty.Register(nameof(AddressSuggestions), typeof(IEnumerable<AddressSuggestion>), typeof(AddressInput));
    public static readonly DependencyProperty SelectedAddressProperty =
        DependencyProperty.Register(nameof(SelectedAddress), typeof(AddressSuggestion), typeof(AddressInput));

    public string ZipCode
    {
        get => (string)GetValue(ZipCodeProperty);
        set => SetValue(ZipCodeProperty, value);
    }
    // ... 나머지 속성들도 동일 패턴

    private void SearchButton_Click(object sender, RoutedEventArgs e)
    {
        // 우편번호 검색 로직 (예: 서비스 호출)
        var suggestions = AddressService.SearchByZipCode(ZipCode);
        AddressSuggestions = suggestions;
    }
}
```

**UserControl의 특징**
- UI와 코드가 한 파일(또는 XAML + 코드비하인드)에 밀접하게 연결됩니다.
- 디자이너에서 바로 미리보기 가능하고, 이벤트 처리도 직관적입니다.
- 외부에서 이 컴포넌트의 모양을 바꾸려면 내부 XAML을 직접 수정해야 합니다. (스타일을 통한 재정의가 어렵습니다)

---

## CustomControl: 확장 가능한 컨트롤 만들기

CustomControl은 WPF 컨트롤 시스템 자체를 확장합니다. **시각적 표현(템플릿)**과 **로직(코드)**이 완전히 분리되어 있어, 사용자가 컨트롤의 외형을 자유롭게 재정의할 수 있습니다.

### 같은 예제를 CustomControl로 구현

먼저 컨트롤 클래스를 만듭니다.

```csharp
// AddressControl.cs
public class AddressControl : Control
{
    static AddressControl()
    {
        DefaultStyleKeyProperty.OverrideMetadata(typeof(AddressControl),
            new FrameworkPropertyMetadata(typeof(AddressControl)));
    }

    public AddressControl()
    {
        SearchCommand = new RelayCommand(ExecuteSearch, CanExecuteSearch);
    }

    // 의존 속성 (UserControl 예제와 동일)
    public static readonly DependencyProperty ZipCodeProperty = ...;
    public static readonly DependencyProperty StreetAddressProperty = ...;
    public static readonly DependencyProperty DetailAddressProperty = ...;
    public static readonly DependencyProperty AddressSuggestionsProperty = ...;
    public static readonly DependencyProperty SelectedAddressProperty = ...;

    public string ZipCode { get; set; }
    // ... 나머지 속성

    public ICommand SearchCommand { get; }

    private bool CanExecuteSearch() => !string.IsNullOrWhiteSpace(ZipCode);
    private async void ExecuteSearch()
    {
        var suggestions = await AddressService.SearchByZipCodeAsync(ZipCode);
        AddressSuggestions = suggestions;
    }

    // 템플릿 파트 연결
    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        var searchButton = GetTemplateChild("PART_SearchButton") as Button;
        if (searchButton != null)
            searchButton.Click += (s, e) => SearchCommand.Execute(null);
    }
}
```

이제 이 컨트롤의 기본 모양을 정의하는 **템플릿**을 `Themes/Generic.xaml`에 작성합니다.

```xml
<!-- Themes/Generic.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                    xmlns:local="clr-namespace:MyApp">
    <Style TargetType="{x:Type local:AddressControl}">
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="{x:Type local:AddressControl}">
                    <StackPanel>
                        <Grid>
                            <TextBox x:Name="PART_ZipCodeBox" Text="{TemplateBinding ZipCode}"/>
                            <Button x:Name="PART_SearchButton" Content="검색"/>
                        </Grid>
                        <ComboBox ItemsSource="{TemplateBinding AddressSuggestions}"
                                  SelectedItem="{TemplateBinding SelectedAddress}"/>
                        <TextBox Text="{TemplateBinding StreetAddress}"/>
                        <TextBox Text="{TemplateBinding DetailAddress}"/>
                    </StackPanel>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>
</ResourceDictionary>
```

**CustomControl의 특징**
- 컨트롤의 시각적 구조는 템플릿에 정의되므로, 외부에서 이 템플릿을 교체하면 완전히 다른 디자인을 적용할 수 있습니다.
- 로직(코드)과 모양이 분리되어 있어 유지보수와 확장이 용이합니다.
- 재사용 가능한 컨트롤 라이브러리를 만들 때 표준적인 방식입니다.

---

## 비교: 한눈에 보는 차이

| 항목 | UserControl | CustomControl |
|------|-------------|---------------|
| **구성 방식** | 기존 컨트롤을 XAML로 조합 | Control 클래스를 상속받아 새 컨트롤 정의 |
| **시각적 표현** | XAML 파일에 고정됨 | 템플릿으로 분리되어 재정의 가능 |
| **스타일링/테마 지원** | 제한적 (템플릿 없음) | 완전 지원 (템플릿 기반) |
| **개발 속도** | 빠름 (드래그 앤 드롭, 코드 비하인드) | 상대적으로 느림 (템플릿 작성 필요) |
| **사용 예** | 특정 화면 전용 컴포넌트, 내부 도구 | 재사용 가능한 라이브러리, 테마 지원 컴포넌트 |

---

## 선택 기준: 언제 무엇을 써야 할까?

### UserControl을 선택하는 상황
- 빠르게 화면을 만들어야 할 때 (프로토타입, 내부 관리 도구)
- 컴포넌트의 디자인이 변경될 가능성이 낮을 때
- 복잡한 비즈니스 로직이 UI와 강하게 결합되어 있을 때
- 프로젝트 내부에서만 사용될 때

### CustomControl을 선택하는 상황
- 여러 프로젝트에서 재사용되는 컨트롤 라이브러리를 만들 때
- 다크 테마 등 다양한 테마를 지원해야 할 때
- 사용자가 컨트롤의 외형을 자유롭게 변경할 수 있어야 할 때
- 성능 최적화가 중요할 때 (템플릿 시스템이 더 효율적)

---

## 실제 개발 팁

1. **처음에는 UserControl로 시작하라**  
   기능이 복잡해지고 여러 곳에서 재사용되기 시작하면 CustomControl로 리팩터링하는 것도 좋은 방법입니다.

2. **CustomControl은 템플릿 파트 네이밍 규칙을 지키자**  
   템플릿 내에서 참조할 요소에는 `PART_` 접두어를 붙이고 `[TemplatePart]` 특성을 명시하면 인텔리센스와 디자이너 지원이 향상됩니다.

3. **UserControl도 의존 속성을 활용하라**  
   UserControl 내부에서 `{Binding ... RelativeSource={RelativeSource AncestorType=UserControl}}` 대신 의존 속성을 정의하면 바인딩이 더 명확해집니다.

4. **성능이 중요한 대규모 리스트에는 CustomControl을 고려하라**  
   CustomControl은 템플릿을 통해 시각적 트리가 가볍게 유지될 수 있어 대량의 항목을 표시할 때 유리합니다.

---

## 결론

UserControl과 CustomControl은 모두 WPF에서 재사용 가능한 UI 컴포넌트를 만드는 강력한 도구입니다. UserControl은 빠른 개발과 단순한 캡슐화에, CustomControl은 확장성과 디자인 자유도에 강점을 가집니다.

프로젝트의 요구사항과 미래 확장성을 고려하여 적절한 방식을 선택하세요. 작은 규모에서는 UserControl로 시작해도 충분하지만, 컴포넌트가 여러 프로젝트에서 공유되거나 디자인 커스터마이징이 중요해진다면 CustomControl로 전환하는 것이 유지보수성을 높이는 길입니다.