---
layout: post
title: WPF - DataTemplate, ControlTemplate
date: 2025-09-02 19:25:23 +0900
category: WPF
---
# WPF 템플릿 마스터하기: DataTemplate과 ControlTemplate

## 템플릿 시스템의 핵심 철학

WPF의 템플릿 시스템은 UI의 외형과 기능을 분리하는 강력한 패러다임을 제공합니다. DataTemplate은 "데이터를 어떻게 보여줄 것인가"를 정의하고, ControlTemplate은 "컨트롤이 어떻게 생겼는가"를 정의합니다. 이 분리는 MVVM 패턴과 완벽하게 조화를 이루며, 재사용성과 유지보수성을 극대화합니다.

```xml
<!-- DataTemplate: 데이터의 시각적 표현 -->
<DataTemplate DataType="{x:Type models:Product}">
    <Border Background="White" CornerRadius="8" Padding="12">
        <StackPanel>
            <TextBlock Text="{Binding Name}" FontWeight="Bold"/>
            <TextBlock Text="{Binding Price, StringFormat=C}" Foreground="Green"/>
        </StackPanel>
    </Border>
</DataTemplate>

<!-- ControlTemplate: 컨트롤의 시각적 구조 -->
<ControlTemplate TargetType="Button">
    <Border Background="{TemplateBinding Background}" CornerRadius="6">
        <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
    </Border>
</ControlTemplate>
```

## DataTemplate: 데이터의 시각적 변신

### DataTemplate의 본질

DataTemplate은 데이터 객체를 시각적 표현으로 변환하는 변환기입니다. 특정 데이터 타입이 UI에 표시될 때 어떻게 렌더링될지 정의합니다. 예를 들어, `Product` 객체를 화면에 표시할 때 제품명, 가격, 이미지를 배치하는 방식을 DataTemplate으로 지정할 수 있습니다.

```xml
<Window.Resources>
    <DataTemplate DataType="{x:Type local:Product}">
        <Border Background="#F8F9FA" CornerRadius="6" Padding="12">
            <Grid>
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="*"/>
                </Grid.ColumnDefinitions>
                <Image Source="{Binding ImageUrl}" Width="64" Height="64"/>
                <StackPanel Grid.Column="1" Margin="12,0">
                    <TextBlock Text="{Binding Name}" FontWeight="Bold"/>
                    <TextBlock Text="{Binding Price, StringFormat=C}" Foreground="#28A745"/>
                </StackPanel>
            </Grid>
        </Border>
    </DataTemplate>
</Window.Resources>

<ListBox ItemsSource="{Binding Products}"/>
```

DataTemplate은 두 가지 방식으로 사용됩니다. `DataType`만 지정하면 해당 타입이 나타날 때 자동으로 적용되는 암시적 템플릿이 되고, `x:Key`를 함께 지정하면 명시적으로 참조해야 하는 명시적 템플릿이 됩니다.

### DataTemplateSelector: 조건부 템플릿 선택

데이터의 상태나 속성에 따라 다른 템플릿을 적용해야 할 때는 DataTemplateSelector를 사용합니다. 예를 들어, 주문 상태가 "긴급"인 경우와 "완료"인 경우에 서로 다른 시각적 표현을 적용할 수 있습니다.

```csharp
public class OrderTemplateSelector : DataTemplateSelector
{
    public DataTemplate NormalOrderTemplate { get; set; }
    public DataTemplate UrgentOrderTemplate { get; set; }
    
    public override DataTemplate SelectTemplate(object item, DependencyObject container)
    {
        if (item is Order order && order.Status == "Urgent")
            return UrgentOrderTemplate;
        return NormalOrderTemplate;
    }
}
```

```xml
<local:OrderTemplateSelector x:Key="OrderSelector"
    NormalOrderTemplate="{StaticResource NormalTemplate}"
    UrgentOrderTemplate="{StaticResource UrgentTemplate}"/>

<ListBox ItemsSource="{Binding Orders}" ItemTemplateSelector="{StaticResource OrderSelector}"/>
```

## ControlTemplate: 컨트롤의 외형 재정의

### ControlTemplate의 구조 이해

ControlTemplate은 컨트롤의 시각적 모양을 완전히 재정의합니다. 버튼의 기본 사각형 모양을 둥근 모서리의 그라데이션 버튼으로 바꾸거나, 체크박스를 토글 스위치 형태로 변경하는 것이 가능합니다. 중요한 것은 컨트롤의 기능(클릭 이벤트, 상태 변화 등)은 그대로 유지된다는 점입니다.

```xml
<Style TargetType="Button" x:Key="RoundedButton">
    <Setter Property="Background" Value="#4F46E5"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <Border x:Name="border" 
                        Background="{TemplateBinding Background}"
                        CornerRadius="8">
                    <ContentPresenter HorizontalAlignment="Center" 
                                      VerticalAlignment="Center"
                                      Margin="12,8"/>
                </Border>
                <ControlTemplate.Triggers>
                    <Trigger Property="IsMouseOver" Value="True">
                        <Setter TargetName="border" Property="Background" Value="#6366F1"/>
                    </Trigger>
                    <Trigger Property="IsPressed" Value="True">
                        <Setter TargetName="border" Property="Background" Value="#4338CA"/>
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

템플릿 내에서 컨트롤의 속성을 참조할 때는 `{TemplateBinding}`을 사용합니다. 이는 템플릿이 적용된 컨트롤의 속성 값을 가져오는 간단한 바인딩 방식입니다. 더 복잡한 바인딩이 필요할 때는 `{RelativeSource TemplatedParent}`를 사용할 수 있습니다.

### VisualStateManager를 활용한 상태 관리

VisualStateManager는 컨트롤의 다양한 시각적 상태(마우스 오버, 눌림, 비활성화 등)를 애니메이션과 함께 정의하는 현대적인 방식입니다. ControlTemplate의 트리거보다 더 풍부한 상호작용을 구현할 수 있습니다.

```xml
<ControlTemplate TargetType="Button">
    <Grid>
        <VisualStateManager.VisualStateGroups>
            <VisualStateGroup x:Name="CommonStates">
                <VisualState x:Name="Normal"/>
                <VisualState x:Name="MouseOver">
                    <Storyboard>
                        <ColorAnimation Storyboard.TargetName="backgroundBrush"
                                        Storyboard.TargetProperty="Color"
                                        To="#3B82F6" Duration="0:0:0.15"/>
                    </Storyboard>
                </VisualState>
                <VisualState x:Name="Pressed">
                    <Storyboard>
                        <ColorAnimation Storyboard.TargetName="backgroundBrush"
                                        Storyboard.TargetProperty="Color"
                                        To="#2563EB" Duration="0:0:0.1"/>
                    </Storyboard>
                </VisualState>
            </VisualStateGroup>
        </VisualStateManager.VisualStateGroups>
        
        <Border x:Name="border" CornerRadius="6">
            <Border.Background>
                <SolidColorBrush x:Name="backgroundBrush" Color="#4F46E5"/>
            </Border.Background>
            <ContentPresenter/>
        </Border>
    </Grid>
</ControlTemplate>
```

### ItemsControl의 ControlTemplate 재정의

ListBox나 ListView 같은 ItemsControl은 전체 컨테이너의 모양뿐만 아니라 각 항목의 모양도 함께 제어할 수 있습니다. ItemsPanel로 항목 배치 방식을 변경하고, ItemContainerStyle로 각 항목 컨테이너의 스타일을 지정할 수 있습니다.

```xml
<Style TargetType="ListBox">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="ListBox">
                <Border BorderBrush="#E5E7EB" BorderThickness="1" CornerRadius="6">
                    <ItemsPresenter Margin="4"/>
                </Border>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
    <Setter Property="ItemContainerStyle">
        <Setter.Value>
            <Style TargetType="ListBoxItem">
                <Setter Property="Template">
                    <Setter.Value>
                        <ControlTemplate TargetType="ListBoxItem">
                            <Border x:Name="border" Background="Transparent" Padding="8">
                                <ContentPresenter/>
                            </Border>
                            <ControlTemplate.Triggers>
                                <Trigger Property="IsSelected" Value="True">
                                    <Setter TargetName="border" Property="Background" Value="#EFF6FF"/>
                                </Trigger>
                            </ControlTemplate.Triggers>
                        </ControlTemplate>
                    </Setter.Value>
                </Setter>
            </Style>
        </Setter.Value>
    </Setter>
</Style>
```

## DataTemplate과 ControlTemplate의 조화

두 템플릿은 함께 사용될 때 더 큰 시너지를 발휘합니다. 예를 들어, 버튼의 ControlTemplate으로 모양을 정의하고, 버튼에 표시될 데이터는 DataTemplate으로 정의할 수 있습니다. ContentPresenter가 이 둘을 연결하는 역할을 합니다.

```xml
<Button>
    <Button.ContentTemplate>
        <DataTemplate>
            <StackPanel Orientation="Horizontal">
                <Image Source="{Binding Icon}" Width="16" Height="16"/>
                <TextBlock Text="{Binding Text}" Margin="4,0,0,0"/>
            </StackPanel>
        </DataTemplate>
    </Button.ContentTemplate>
</Button>
```

## 실전 패턴

### 뷰-뷰모델 자동 매핑

MVVM 패턴에서 ViewModel에 따라 View를 자동으로 선택하는 패턴입니다. ContentControl에 ViewModel을 바인딩하면, 해당 ViewModel 타입에 맞는 DataTemplate이 자동으로 View를 렌더링합니다.

```xml
<Window.Resources>
    <DataTemplate DataType="{x:Type viewmodels:DashboardViewModel}">
        <views:DashboardView/>
    </DataTemplate>
    <DataTemplate DataType="{x:Type viewmodels:ProductsViewModel}">
        <views:ProductsView/>
    </DataTemplate>
</Window.Resources>

<ContentControl Content="{Binding CurrentViewModel}"/>
```

### 템플릿 리소스 구성

템플릿을 리소스 사전으로 분리하면 여러 창에서 재사용할 수 있습니다. 테마를 지원하는 경우, 리소스 사전을 교체하는 방식으로 전체 UI 스타일을 변경할 수 있습니다.

```xml
<!-- Themes/LightTheme.xaml -->
<ResourceDictionary>
    <SolidColorBrush x:Key="PrimaryBrush" Color="#3B82F6"/>
    <ControlTemplate x:Key="ButtonTemplate" TargetType="Button">
        <Border Background="{StaticResource PrimaryBrush}" CornerRadius="6">
            <ContentPresenter/>
        </Border>
    </ControlTemplate>
</ResourceDictionary>
```

## 성능 고려사항

템플릿을 사용할 때는 성능에 유의해야 합니다. 많은 항목을 표시하는 ListBox에서는 가상화를 활성화하는 것이 중요합니다. 복잡한 템플릿은 렌더링 비용이 높을 수 있으므로, 불필요한 중첩을 피하고 Freezable 객체(브러시, 애니메이션 등)는 가능한 공유하여 사용하는 것이 좋습니다.

```xml
<ListBox VirtualizingPanel.IsVirtualizing="True"
         VirtualizingPanel.VirtualizationMode="Recycling">
```

## 결론

WPF의 템플릿 시스템은 UI 개발에서 관심사의 분리를 실현합니다. DataTemplate은 데이터 표현에 집중하고, ControlTemplate은 컨트롤의 외형에 집중하여 코드의 재사용성과 유지보수성을 높입니다. 초기 학습 곡선은 다소 가파를 수 있지만, 이 개념을 마스터하면 WPF의 진정한 힘을 발휘할 수 있습니다.

실무에서는 다음과 같은 원칙을 기억하면 좋습니다:

- 단순한 스타일 변경은 Template보다 Style로 충분합니다
- 템플릿은 재사용 가능한 단위로 분리하여 리소스 사전에 관리합니다
- 복잡한 템플릿은 VisualStateManager를 활용하여 상태 전환을 명확히 합니다
- 많은 항목을 다루는 컨트롤에서는 가상화와 템플릿 최적화를 고려합니다

템플릿을 효과적으로 활용하면 디자인 시스템을 일관되게 적용하면서도 비즈니스 로직과 UI를 깔끔하게 분리한 애플리케이션을 구축할 수 있습니다.