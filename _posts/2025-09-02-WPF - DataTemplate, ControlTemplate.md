---
layout: post
title: WPF - DataTemplate, ControlTemplate
date: 2025-09-02 19:25:23 +0900
category: WPF
---
# WPF 템플릿 마스터하기: DataTemplate과 ControlTemplate의 완벽 이해

## 템플릿 시스템의 핵심 철학

WPF의 템플릿 시스템은 UI의 외형과 기능을 분리하는 강력한 패러다임을 제공합니다. DataTemplate은 "데이터를 어떻게 보여줄 것인가"를 정의하고, ControlTemplate은 "컨트롤이 어떻게 생겼는가"를 정의합니다. 이 분리는 MVVM 패턴과 완벽하게 조화를 이루며, 재사용성과 유지보수성을 극대화합니다.

```xml
<!-- DataTemplate: 데이터의 시각적 표현 -->
<DataTemplate DataType="{x:Type models:Product}">
    <Border Background="White" CornerRadius="8" Padding="12" 
            BorderBrush="#E0E0E0" BorderThickness="1">
        <StackPanel>
            <TextBlock Text="{Binding Name}" FontWeight="Bold"/>
            <TextBlock Text="{Binding Price, StringFormat=C}" 
                       Foreground="Green" Margin="0,4,0,0"/>
        </StackPanel>
    </Border>
</DataTemplate>

<!-- ControlTemplate: 컨트롤의 시각적 구조 -->
<ControlTemplate TargetType="Button">
    <Border Background="{TemplateBinding Background}" 
            CornerRadius="6" Padding="8,4">
        <ContentPresenter HorizontalAlignment="Center" 
                          VerticalAlignment="Center"/>
    </Border>
</ControlTemplate>
```

## DataTemplate: 데이터의 시각적 변신

### DataTemplate의 본질

DataTemplate은 데이터 객체를 시각적 표현으로 변환하는 변환기입니다. 특정 데이터 타입이 UI에 표시될 때 어떻게 렌더링될지 정의합니다.

```xml
<!-- 기본 사용법 -->
<Window.Resources>
    <!-- Product 타입에 대한 DataTemplate -->
    <DataTemplate DataType="{x:Type local:Product}">
        <Border Background="#F8F9FA" CornerRadius="6" Padding="12" 
                Margin="4">
            <Grid>
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="Auto"/>
                </Grid.ColumnDefinitions>
                
                <!-- 제품 이미지 -->
                <Image Source="{Binding ImageUrl}" Width="64" Height="64" 
                       Grid.Column="0" Stretch="Uniform"/>
                
                <!-- 제품 정보 -->
                <StackPanel Grid.Column="1" Margin="12,0">
                    <TextBlock Text="{Binding Name}" FontWeight="Bold"/>
                    <TextBlock Text="{Binding Description}" 
                               TextWrapping="Wrap" Margin="0,4,0,0"/>
                </StackPanel>
                
                <!-- 가격 및 액션 -->
                <StackPanel Grid.Column="2" HorizontalAlignment="Right">
                    <TextBlock Text="{Binding Price, StringFormat=C}" 
                               FontSize="16" FontWeight="Bold" 
                               Foreground="#28A745"/>
                    <Button Content="구매" Margin="0,8,0,0" 
                            Command="{Binding DataContext.BuyCommand, 
                                     RelativeSource={RelativeSource AncestorType=Window}}"
                            CommandParameter="{Binding}"/>
                </StackPanel>
            </Grid>
        </Border>
    </DataTemplate>
</Window.Resources>

<!-- 사용: ListBox에서 Product 객체가 자동으로 템플릿 적용 -->
<ListBox ItemsSource="{Binding Products}"/>
```

### 암시적 vs 명시적 DataTemplate

```xml
<!-- 암시적 DataTemplate: DataType만 지정 -->
<DataTemplate DataType="{x:Type local:Customer}">
    <!-- Customer 타입이 표시될 때마다 자동 적용 -->
</DataTemplate>

<!-- 명시적 DataTemplate: x:Key로 참조 -->
<DataTemplate x:Key="DetailedCustomerTemplate">
    <!-- StaticResource로 명시적 적용 필요 -->
</DataTemplate>

<ContentControl Content="{Binding SelectedCustomer}"
                ContentTemplate="{StaticResource DetailedCustomerTemplate}"/>
```

### 다양한 컨트롤에서의 DataTemplate 활용

```xml
<!-- ContentControl에서 사용 -->
<ContentControl Content="{Binding SelectedItem}">
    <ContentControl.ContentTemplate>
        <DataTemplate>
            <Border Background="White" Padding="16">
                <TextBlock Text="{Binding Title}" FontSize="18"/>
            </Border>
        </DataTemplate>
    </ContentControl.ContentTemplate>
</ContentControl>

<!-- ListView의 GridView에서 사용 -->
<ListView ItemsSource="{Binding Orders}">
    <ListView.View>
        <GridView>
            <GridViewColumn Header="주문번호">
                <GridViewColumn.CellTemplate>
                    <DataTemplate>
                        <TextBlock Text="{Binding OrderId}" 
                                   Foreground="Blue" 
                                   FontWeight="Bold"/>
                    </DataTemplate>
                </GridViewColumn.CellTemplate>
            </GridViewColumn>
            
            <GridViewColumn Header="금액">
                <GridViewColumn.CellTemplate>
                    <DataTemplate>
                        <StackPanel Orientation="Horizontal">
                            <TextBlock Text="{Binding Amount, StringFormat=C}"/>
                            <TextBlock Text="{Binding Currency}" 
                                       Margin="4,0,0,0" 
                                       Foreground="Gray"/>
                        </StackPanel>
                    </DataTemplate>
                </GridViewColumn.CellTemplate>
            </GridViewColumn>
        </GridView>
    </ListView.View>
</ListView>
```

### DataTemplateSelector: 조건부 템플릿 선택

```csharp
public class OrderTemplateSelector : DataTemplateSelector
{
    public DataTemplate NormalOrderTemplate { get; set; }
    public DataTemplate UrgentOrderTemplate { get; set; }
    public DataTemplate CompletedOrderTemplate { get; set; }
    
    public override DataTemplate SelectTemplate(object item, 
                                                DependencyObject container)
    {
        if (item is Order order)
        {
            switch (order.Status)
            {
                case OrderStatus.Urgent:
                    return UrgentOrderTemplate;
                case OrderStatus.Completed:
                    return CompletedOrderTemplate;
                default:
                    return NormalOrderTemplate;
            }
        }
        
        return base.SelectTemplate(item, container);
    }
}
```

```xml
<!-- XAML에서 템플릿 셀렉터 설정 -->
<Window.Resources>
    <!-- 다양한 상태별 템플릿 -->
    <DataTemplate x:Key="NormalOrderTemplate">
        <Border Background="White" Padding="8">
            <TextBlock Text="{Binding OrderId}"/>
        </Border>
    </DataTemplate>
    
    <DataTemplate x:Key="UrgentOrderTemplate">
        <Border Background="#FFFCE4E4" Padding="8" 
                BorderBrush="Red" BorderThickness="1">
            <StackPanel Orientation="Horizontal">
                <TextBlock Text="⚠️ " VerticalAlignment="Center"/>
                <TextBlock Text="{Binding OrderId}" FontWeight="Bold"/>
            </StackPanel>
        </Border>
    </DataTemplate>
    
    <DataTemplate x:Key="CompletedOrderTemplate">
        <Border Background="#E8F5E8" Padding="8">
            <StackPanel Orientation="Horizontal">
                <TextBlock Text="✅ " VerticalAlignment="Center"/>
                <TextBlock Text="{Binding OrderId}"/>
            </StackPanel>
        </Border>
    </DataTemplate>
    
    <!-- 템플릿 셀렉터 -->
    <local:OrderTemplateSelector x:Key="OrderTemplateSelector"
        NormalOrderTemplate="{StaticResource NormalOrderTemplate}"
        UrgentOrderTemplate="{StaticResource UrgentOrderTemplate}"
        CompletedOrderTemplate="{StaticResource CompletedOrderTemplate}"/>
</Window.Resources>

<!-- ListBox에서 사용 -->
<ListBox ItemsSource="{Binding Orders}" 
         ItemTemplateSelector="{StaticResource OrderTemplateSelector}"/>
```

## ControlTemplate: 컨트롤의 외형 재정의

### ControlTemplate의 구조 이해

ControlTemplate은 컨트롤의 시각적 모양을 완전히 재정의합니다. 기본 컨트롤의 기능은 유지하면서 외관만 변경할 수 있습니다.

```xml
<!-- 기본 Button을 둥근 모양으로 재정의 -->
<Style TargetType="Button" x:Key="RoundedButton">
    <Setter Property="Background" Value="#4F46E5"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="Padding" Value="12,8"/>
    <Setter Property="Cursor" Value="Hand"/>
    
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <Grid>
                    <!-- 배경 레이어 -->
                    <Border x:Name="BackgroundBorder"
                            Background="{TemplateBinding Background}"
                            CornerRadius="8"/>
                    
                    <!-- 그림자 효과 -->
                    <Border x:Name="ShadowBorder" 
                            Background="Transparent"
                            CornerRadius="8"
                            Effect="{StaticResource DropShadowEffect}"/>
                    
                    <!-- 콘텐츠 -->
                    <ContentPresenter x:Name="ContentHost"
                        HorizontalAlignment="{TemplateBinding HorizontalContentAlignment}"
                        VerticalAlignment="{TemplateBinding VerticalContentAlignment}"
                        Margin="{TemplateBinding Padding}"/>
                    
                    <!-- 호버 오버레이 -->
                    <Border x:Name="HoverOverlay"
                            Background="White"
                            Opacity="0"
                            CornerRadius="8"/>
                </Grid>
                
                <!-- 트리거로 상태 변화 처리 -->
                <ControlTemplate.Triggers>
                    <Trigger Property="IsMouseOver" Value="True">
                        <Setter TargetName="HoverOverlay" 
                                Property="Opacity" Value="0.1"/>
                        <Setter TargetName="BackgroundBorder" 
                                Property="Background" 
                                Value="{Binding Background, 
                                        RelativeSource={RelativeSource TemplatedParent},
                                        Converter={StaticResource DarkenBrushConverter}}"/>
                    </Trigger>
                    
                    <Trigger Property="IsPressed" Value="True">
                        <Setter TargetName="HoverOverlay" 
                                Property="Opacity" Value="0.2"/>
                        <Setter TargetName="ContentHost" 
                                Property="RenderTransform">
                            <Setter.Value>
                                <TranslateTransform Y="1"/>
                            </Setter.Value>
                        </Setter>
                    </Trigger>
                    
                    <Trigger Property="IsEnabled" Value="False">
                        <Setter TargetName="BackgroundBorder" 
                                Property="Opacity" Value="0.5"/>
                        <Setter TargetName="ContentHost" 
                                Property="Opacity" Value="0.7"/>
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

### VisualStateManager를 활용한 모던 스타일링

```xml
<!-- VisualStateManager를 사용한 버튼 템플릿 -->
<ControlTemplate TargetType="Button" x:Key="ModernButtonTemplate">
    <Grid x:Name="Root">
        <VisualStateManager.VisualStateGroups>
            <VisualStateGroup x:Name="CommonStates">
                <VisualState x:Name="Normal"/>
                
                <VisualState x:Name="MouseOver">
                    <Storyboard>
                        <DoubleAnimation Storyboard.TargetName="HoverEffect"
                                         Storyboard.TargetProperty="Opacity"
                                         To="0.1" Duration="0:0:0.15"/>
                        <ColorAnimation Storyboard.TargetName="BackgroundBrush"
                                        Storyboard.TargetProperty="Color"
                                        To="#3B82F6" Duration="0:0:0.15"/>
                    </Storyboard>
                </VisualState>
                
                <VisualState x:Name="Pressed">
                    <Storyboard>
                        <DoubleAnimation Storyboard.TargetName="HoverEffect"
                                         Storyboard.TargetProperty="Opacity"
                                         To="0.2" Duration="0:0:0.1"/>
                        <ColorAnimation Storyboard.TargetName="BackgroundBrush"
                                        Storyboard.TargetProperty="Color"
                                        To="#2563EB" Duration="0:0:0.1"/>
                        <DoubleAnimation Storyboard.TargetName="ContentPresenter"
                                         Storyboard.TargetProperty="RenderTransform.ScaleX"
                                         To="0.98" Duration="0:0:0.1"/>
                        <DoubleAnimation Storyboard.TargetName="ContentPresenter"
                                         Storyboard.TargetProperty="RenderTransform.ScaleY"
                                         To="0.98" Duration="0:0:0.1"/>
                    </Storyboard>
                </VisualState>
                
                <VisualState x:Name="Disabled">
                    <Storyboard>
                        <DoubleAnimation Storyboard.TargetName="ContentPresenter"
                                         Storyboard.TargetProperty="Opacity"
                                         To="0.5" Duration="0:0:0.2"/>
                    </Storyboard>
                </VisualState>
            </VisualStateGroup>
            
            <VisualStateGroup x:Name="FocusStates">
                <VisualState x:Name="Focused">
                    <Storyboard>
                        <DoubleAnimation Storyboard.TargetName="FocusBorder"
                                         Storyboard.TargetProperty="Opacity"
                                         To="1" Duration="0:0:0.15"/>
                    </Storyboard>
                </VisualState>
                <VisualState x:Name="Unfocused"/>
            </VisualStateGroup>
        </VisualStateManager.VisualStateGroups>
        
        <!-- 배경 -->
        <Border CornerRadius="6" Margin="2">
            <Border.Background>
                <SolidColorBrush x:Name="BackgroundBrush" Color="#4F46E5"/>
            </Border.Background>
            
            <!-- 호버 효과 -->
            <Border x:Name="HoverEffect" Background="White" 
                    Opacity="0" CornerRadius="6"/>
        </Border>
        
        <!-- 포커스 표시 -->
        <Border x:Name="FocusBorder" BorderBrush="#60A5FA" 
                BorderThickness="2" CornerRadius="8" Opacity="0"/>
        
        <!-- 콘텐츠 -->
        <ContentPresenter x:Name="ContentPresenter"
            HorizontalAlignment="Center" 
            VerticalAlignment="Center"
            Margin="12,8">
            <ContentPresenter.RenderTransform>
                <ScaleTransform ScaleX="1" ScaleY="1"/>
            </ContentPresenter.RenderTransform>
        </ContentPresenter>
    </Grid>
</ControlTemplate>
```

### TemplateBinding vs RelativeSource TemplatedParent

```xml
<ControlTemplate TargetType="Button">
    <!-- TemplateBinding: 단순한 OneWay 바인딩 (권장) -->
    <Border Background="{TemplateBinding Background}"
            BorderBrush="{TemplateBinding BorderBrush}"
            BorderThickness="{TemplateBinding BorderThickness}">
        <ContentPresenter Margin="{TemplateBinding Padding}"/>
    </Border>
    
    <!-- RelativeSource TemplatedParent: 복잡한 바인딩 필요시 -->
    <TextBlock Text="{Binding Path=Content, 
                     RelativeSource={RelativeSource TemplatedParent},
                     Converter={StaticResource StringToUpperConverter}}"/>
</ControlTemplate>
```

### ItemsControl의 ControlTemplate

```xml
<!-- ListBox의 ControlTemplate 재정의 -->
<Style TargetType="ListBox">
    <Setter Property="Background" Value="Transparent"/>
    <Setter Property="BorderBrush" Value="#E5E7EB"/>
    <Setter Property="BorderThickness" Value="1"/>
    <Setter Property="ScrollViewer.HorizontalScrollBarVisibility" Value="Disabled"/>
    <Setter Property="ScrollViewer.VerticalScrollBarVisibility" Value="Auto"/>
    
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="ListBox">
                <Border Background="{TemplateBinding Background}"
                        BorderBrush="{TemplateBinding BorderBrush}"
                        BorderThickness="{TemplateBinding BorderThickness}"
                        CornerRadius="6">
                    
                    <!-- ItemsPresenter 필수: 아이템들이 렌더링되는 위치 -->
                    <ItemsPresenter Margin="{TemplateBinding Padding}"/>
                </Border>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
    
    <!-- ItemContainerStyle도 함께 설정 -->
    <Setter Property="ItemContainerStyle">
        <Setter.Value>
            <Style TargetType="ListBoxItem">
                <Setter Property="Background" Value="Transparent"/>
                <Setter Property="BorderBrush" Value="Transparent"/>
                <Setter Property="BorderThickness" Value="0"/>
                <Setter Property="Template">
                    <Setter.Value>
                        <ControlTemplate TargetType="ListBoxItem">
                            <Border x:Name="ItemBorder"
                                    Background="{TemplateBinding Background}"
                                    BorderBrush="{TemplateBinding BorderBrush}"
                                    BorderThickness="{TemplateBinding BorderThickness}"
                                    Padding="8">
                                <ContentPresenter/>
                            </Border>
                            
                            <ControlTemplate.Triggers>
                                <Trigger Property="IsSelected" Value="True">
                                    <Setter TargetName="ItemBorder" 
                                            Property="Background" 
                                            Value="#EFF6FF"/>
                                    <Setter TargetName="ItemBorder" 
                                            Property="BorderBrush" 
                                            Value="#3B82F6"/>
                                </Trigger>
                                
                                <Trigger Property="IsMouseOver" Value="True">
                                    <Setter TargetName="ItemBorder" 
                                            Property="Background" 
                                            Value="#F9FAFB"/>
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

### 커스텀 컨트롤 템플릿

```csharp
// 커스텀 컨트롤 정의
[TemplatePart(Name = "PART_TextBlock", Type = typeof(TextBlock))]
[TemplatePart(Name = "PART_ValueBox", Type = typeof(TextBox))]
public class LabeledTextBox : Control
{
    static LabeledTextBox()
    {
        DefaultStyleKeyProperty.OverrideMetadata(typeof(LabeledTextBox),
            new FrameworkPropertyMetadata(typeof(LabeledTextBox)));
    }
    
    // 의존 속성 정의
    public static readonly DependencyProperty LabelProperty =
        DependencyProperty.Register("Label", typeof(string), typeof(LabeledTextBox));
    
    public static readonly DependencyProperty TextProperty =
        DependencyProperty.Register("Text", typeof(string), typeof(LabeledTextBox));
    
    public string Label
    {
        get => (string)GetValue(LabelProperty);
        set => SetValue(LabelProperty, value);
    }
    
    public string Text
    {
        get => (string)GetValue(TextProperty);
        set => SetValue(TextProperty, value);
    }
    
    private TextBlock _labelElement;
    private TextBox _valueElement;
    
    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        
        // 템플릿 파트 가져오기
        _labelElement = GetTemplateChild("PART_TextBlock") as TextBlock;
        _valueElement = GetTemplateChild("PART_ValueBox") as TextBox;
        
        if (_valueElement != null)
        {
            _valueElement.TextChanged += (s, e) =>
            {
                Text = _valueElement.Text;
            };
        }
    }
}
```

```xml
<!-- Themes/Generic.xaml -->
<Style TargetType="{x:Type local:LabeledTextBox}">
    <Setter Property="Background" Value="White"/>
    <Setter Property="BorderBrush" Value="#D1D5DB"/>
    <Setter Property="BorderThickness" Value="1"/>
    <Setter Property="Padding" Value="8"/>
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="{x:Type local:LabeledTextBox}">
                <Border Background="{TemplateBinding Background}"
                        BorderBrush="{TemplateBinding BorderBrush}"
                        BorderThickness="{TemplateBinding BorderThickness}"
                        CornerRadius="4"
                        Padding="{TemplateBinding Padding}">
                    <Grid>
                        <Grid.ColumnDefinitions>
                            <ColumnDefinition Width="Auto"/>
                            <ColumnDefinition Width="*"/>
                        </Grid.ColumnDefinitions>
                        
                        <!-- 라벨 -->
                        <TextBlock x:Name="PART_TextBlock"
                                   Text="{TemplateBinding Label}"
                                   VerticalAlignment="Center"
                                   Margin="0,0,8,0"/>
                        
                        <!-- 입력 필드 -->
                        <TextBox x:Name="PART_ValueBox"
                                 Grid.Column="1"
                                 Text="{TemplateBinding Text, Mode=TwoWay}"
                                 BorderThickness="0"
                                 Background="Transparent"/>
                    </Grid>
                </Border>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

## DataTemplate과 ControlTemplate의 조화로운 사용

### 완전한 예제: 제품 목록 애플리케이션

```xml
<!-- App.xaml - 전역 리소스 -->
<Application.Resources>
    <!-- Product 타입에 대한 DataTemplate -->
    <DataTemplate DataType="{x:Type models:Product}">
        <Border Background="White" CornerRadius="8" 
                BorderBrush="#E5E7EB" BorderThickness="1"
                Margin="4">
            <Grid>
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                    <RowDefinition Height="Auto"/>
                </Grid.RowDefinitions>
                
                <!-- 제품 이미지 -->
                <Border Grid.Row="0" CornerRadius="6,6,0,0" 
                        Background="#F9FAFB" Height="120">
                    <Image Source="{Binding ImageUrl}" 
                           Stretch="Uniform" 
                           HorizontalAlignment="Center"
                           VerticalAlignment="Center"/>
                </Border>
                
                <!-- 제품 정보 -->
                <StackPanel Grid.Row="1" Margin="12">
                    <TextBlock Text="{Binding Name}" 
                               FontWeight="Bold" 
                               TextWrapping="Wrap"/>
                    <TextBlock Text="{Binding Category}" 
                               Foreground="#6B7280" 
                               Margin="0,4,0,0"/>
                    <TextBlock Text="{Binding Description}" 
                               TextWrapping="Wrap" 
                               Margin="0,8,0,0"/>
                </StackPanel>
                
                <!-- 가격 및 액션 -->
                <Border Grid.Row="2" Background="#F9FAFB" 
                        CornerRadius="0,0,8,8" Padding="12">
                    <Grid>
                        <Grid.ColumnDefinitions>
                            <ColumnDefinition Width="*"/>
                            <ColumnDefinition Width="Auto"/>
                        </Grid.ColumnDefinitions>
                        
                        <!-- 가격 -->
                        <StackPanel Grid.Column="0">
                            <TextBlock Text="{Binding Price, StringFormat=C}" 
                                       FontSize="16" 
                                       FontWeight="Bold"
                                       Foreground="#10B981"/>
                            <TextBlock Text="재고: {Binding Stock}" 
                                       FontSize="12"
                                       Foreground="#6B7280"/>
                        </StackPanel>
                        
                        <!-- 버튼 -->
                        <Button Grid.Column="1" 
                                Content="장바구니에 추가"
                                Style="{StaticResource PrimaryButton}"
                                Command="{Binding DataContext.AddToCartCommand, 
                                         RelativeSource={RelativeSource AncestorType=Window}}"
                                CommandParameter="{Binding}"/>
                    </Grid>
                </Border>
            </Grid>
            
            <!-- 상태에 따른 트리거 -->
            <DataTemplate.Triggers>
                <DataTrigger Binding="{Binding Stock}" Value="0">
                    <Setter Property="Opacity" Value="0.6"/>
                </DataTrigger>
            </DataTemplate.Triggers>
        </Border>
    </DataTemplate>
    
    <!-- PrimaryButton 스타일 -->
    <Style x:Key="PrimaryButton" TargetType="Button">
        <Setter Property="Background" Value="#10B981"/>
        <Setter Property="Foreground" Value="White"/>
        <Setter Property="Padding" Value="12,8"/>
        <Setter Property="Cursor" Value="Hand"/>
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Grid>
                        <Border x:Name="BackgroundBorder"
                                Background="{TemplateBinding Background}"
                                CornerRadius="6"/>
                        <ContentPresenter HorizontalAlignment="Center" 
                                          VerticalAlignment="Center"
                                          Margin="{TemplateBinding Padding}"/>
                        <Border x:Name="HoverOverlay" Background="White" 
                                Opacity="0" CornerRadius="6"/>
                    </Grid>
                    
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsMouseOver" Value="True">
                            <Setter TargetName="HoverOverlay" 
                                    Property="Opacity" Value="0.1"/>
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>
</Application.Resources>
```

```xml
<!-- MainWindow.xaml -->
<Window>
    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        
        <!-- 헤더 -->
        <Border Grid.Row="0" Background="White" 
                Padding="16" CornerRadius="8"
                BorderBrush="#E5E7EB" BorderThickness="1">
            <StackPanel>
                <TextBlock Text="제품 카탈로그" 
                           FontSize="24" 
                           FontWeight="Bold"/>
                <TextBlock Text="다양한 제품을 탐색해보세요" 
                           Foreground="#6B7280" 
                           Margin="0,4,0,0"/>
            </StackPanel>
        </Border>
        
        <!-- 제품 목록 -->
        <ItemsControl Grid.Row="1" 
                      ItemsSource="{Binding Products}"
                      Margin="0,16,0,0">
            <ItemsControl.ItemsPanel>
                <ItemsPanelTemplate>
                    <!-- WrapPanel로 반응형 그리드 생성 -->
                    <WrapPanel Orientation="Horizontal"/>
                </ItemsPanelTemplate>
            </ItemsControl.ItemsPanel>
            
            <ItemsControl.ItemTemplate>
                <!-- Product 타입의 DataTemplate 자동 적용 -->
                <!-- DataContext는 각 Product 객체 -->
            </ItemsControl.ItemTemplate>
        </ItemsControl>
        
        <!-- 푸터 -->
        <Border Grid.Row="2" Background="#F9FAFB" 
                Margin="0,16,0,0" Padding="12"
                CornerRadius="8">
            <StackPanel Orientation="Horizontal" 
                        HorizontalAlignment="Right">
                <Button Content="이전" 
                        Style="{StaticResource SecondaryButton}"
                        Margin="0,0,8,0"/>
                <Button Content="다음" 
                        Style="{StaticResource PrimaryButton}"/>
            </StackPanel>
        </Border>
    </Grid>
</Window>
```

## 템플릿 시스템의 실전 패턴

### 패턴 1: 뷰-뷰모델 자동 매핑

```xml
<!-- DataTemplate을 이용한 뷰-뷰모델 자동 연결 -->
<Window.Resources>
    <DataTemplate DataType="{x:Type viewmodels:DashboardViewModel}">
        <views:DashboardView/>
    </DataTemplate>
    
    <DataTemplate DataType="{x:Type viewmodels:ProductsViewModel}">
        <views:ProductsView/>
    </DataTemplate>
    
    <DataTemplate DataType="{x:Type viewmodels:SettingsViewModel}">
        <views:SettingsView/>
    </DataTemplate>
</Window.Resources>

<!-- 메인 컨텐츠 영역 -->
<ContentControl Content="{Binding CurrentViewModel}"/>
<!-- CurrentViewModel이 변경되면 해당 뷰모델에 맞는 뷰가 자동 표시 -->
```

### 패턴 2: 테마 시스템 구현

```xml
<!-- 라이트 테마 -->
<ResourceDictionary x:Key="LightTheme">
    <Color x:Key="PrimaryColor">#3B82F6</Color>
    <Color x:Key="BackgroundColor">#FFFFFF</Color>
    <Color x:Key="TextColor">#111827</Color>
    
    <!-- 버튼 템플릿 -->
    <ControlTemplate x:Key="ButtonTemplate" TargetType="Button">
        <Border Background="{DynamicResource PrimaryColor}" 
                CornerRadius="6">
            <ContentPresenter/>
        </Border>
    </ControlTemplate>
</ResourceDictionary>

<!-- 다크 테마 -->
<ResourceDictionary x:Key="DarkTheme">
    <Color x:Key="PrimaryColor">#60A5FA</Color>
    <Color x:Key="BackgroundColor">#111827</Color>
    <Color x:Key="TextColor">#F9FAFB</Color>
    
    <!-- 동일한 키의 버튼 템플릿 -->
    <ControlTemplate x:Key="ButtonTemplate" TargetType="Button">
        <Border Background="{DynamicResource PrimaryColor}" 
                CornerRadius="6">
            <ContentPresenter/>
        </Border>
    </ControlTemplate>
</ResourceDictionary>
```

### 패턴 3: 로딩 상태 템플릿

```xml
<!-- 로딩 상태 DataTemplate -->
<DataTemplate x:Key="LoadingTemplate">
    <Grid HorizontalAlignment="Center" 
          VerticalAlignment="Center">
        <StackPanel Orientation="Horizontal">
            <ProgressBar IsIndeterminate="True" 
                         Width="100" 
                         Margin="0,0,12,0"/>
            <TextBlock Text="로딩 중..." 
                       VerticalAlignment="Center"/>
        </StackPanel>
    </Grid>
</DataTemplate>

<!-- 오류 상태 DataTemplate -->
<DataTemplate x:Key="ErrorTemplate">
    <Border Background="#FEF2F2" 
            BorderBrush="#FCA5A5" 
            BorderThickness="1"
            CornerRadius="6" 
            Padding="16">
        <StackPanel HorizontalAlignment="Center">
            <TextBlock Text="⚠️ 오류 발생" 
                       FontWeight="Bold" 
                       Foreground="#DC2626"/>
            <TextBlock Text="{Binding ErrorMessage}" 
                       Margin="0,8,0,0" 
                       TextWrapping="Wrap"/>
            <Button Content="재시도" 
                    Margin="0,12,0,0"
                    Command="{Binding RetryCommand}"/>
        </StackPanel>
    </Border>
</DataTemplate>
```

## 성능 최적화와 모범 사례

### 템플릿 성능 팁

```xml
<!-- 1. 가상화 활용 -->
<ListBox VirtualizingPanel.IsVirtualizing="True"
         VirtualizingPanel.VirtualizationMode="Recycling"
         VirtualizingPanel.ScrollUnit="Pixel">
    <!-- 많은 항목도 효율적으로 처리 -->
</ListBox>

<!-- 2. Freezable 객체 재사용 -->
<Window.Resources>
    <!-- Freeze 가능한 객체는 Freeze하여 공유 -->
    <LinearGradientBrush x:Key="SharedGradient" 
                         StartPoint="0,0" EndPoint="1,1">
        <GradientStop Color="#3B82F6" Offset="0"/>
        <GradientStop Color="#8B5CF6" Offset="1"/>
    </LinearGradientBrush>
</Window.Resources>

<!-- 3. 캐싱 활용 -->
<Border CacheMode="BitmapCache">
    <!-- 복잡한 콘텐츠 캐싱 -->
</Border>
```

### 디버깅과 문제 해결

```xml
<!-- 템플릿 디버깅을 위한 트레이싱 -->
<ContentControl Content="{Binding SelectedItem}"
                ContentTemplate="{Binding SelectedTemplate}">
    <!-- 바인딩 오류 추적 -->
    <ContentControl.Style>
        <Style TargetType="ContentControl">
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="ContentControl">
                        <Border BorderBrush="Red" BorderThickness="1"
                                Background="Transparent">
                            <ContentPresenter/>
                        </Border>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>
    </ContentControl.Style>
</ContentControl>

<!-- DataTemplate 내부에서 상위 DataContext 접근 -->
<Button Command="{Binding DataContext.ParentCommand, 
                 RelativeSource={RelativeSource AncestorType=Window}}"/>
```

## 결론: WPF 템플릿의 예술적 완성

WPF의 템플릿 시스템은 단순한 기술적 기능을 넘어 UI 개발의 새로운 패러다임을 제시합니다. DataTemplate과 ControlTemplate을 효과적으로 활용하면 다음과 같은 이점을 얻을 수 있습니다:

### 1. 관심사의 완벽한 분리
- **DataTemplate**: 데이터 표현 로직만 담당
- **ControlTemplate**: 컨트롤 외형만 담당
- **ViewModel**: 비즈니스 로직만 담당

이 분리는 코드의 재사용성과 테스트 가능성을 크게 향상시킵니다.

### 2. 무한한 커스터마이제이션
기본 컨트롤의 기능을 유지하면서 외형을 완전히 재정의할 수 있습니다. 이는 브랜딩 요구사항이나 특수한 UX 요구를 충족시키는 데 필수적입니다.

### 3. 동적 적응성
DataTemplateSelector를 활용하면 런타임에 데이터 상태에 따라 적절한 템플릿을 선택할 수 있습니다. 이는 복잡한 비즈니스 규칙을 우아하게 구현하는 데 도움이 됩니다.

### 4. 일관된 디자인 시스템
템플릿을 통해 디자인 시스템을 구축하면 애플리케이션 전체에 일관된 UX를 제공할 수 있습니다. 버튼, 입력 필드, 카드 등 공통 컴포넌트의 재사용이 용이해집니다.

### 실천적 지혜

1. **적절한 추상화 수준 선택**: 모든 것을 템플릿으로 만들 필요는 없습니다. 간단한 스타일로 충분한 경우도 많습니다.

2. **성능 고려**: 템플릿은 강력하지만 비용이 따릅니다. 가상화, 캐싱, 리소스 재사용을 통해 성능을 최적화하세요.

3. **유지보수성 우선**: 복잡한 템플릿은 문서화와 함께 제공하세요. 다른 개발자들이 이해하고 수정할 수 있도록 설계하세요.

4. **점진적 개선**: 처음부터 완벽한 템플릿을 만들기보다, 기본 동작을 구현하고 점진적으로 개선해 나가세요.

5. **표준 준수**: 플랫폼의 시각적 표준과 접근성 가이드라인을 준수하세요. 템플릿은 기능뿐만 아니라 사용성도 책임집니다.

WPF의 템플릿 시스템을 마스터하는 것은 단순한 기술 습득을 넘어, 사용자 인터페이스 설계에 대한 깊은 통찰력을 얻는 과정입니다. 데이터와 컨트롤, 외형과 기능의 관계를 이해하고, 이들을 조화롭게 통합하는 능력이 바로 전문 WPF 개발자의 핵심 역량입니다.

여러분의 다음 프로젝트에서 이 지식을 활용하여 더 아름답고, 더 기능적이며, 더 유지보수하기 쉬운 애플리케이션을 만들어 보세요. 템플릿의 힘은 여러분의 창의성만큼 큽니다.