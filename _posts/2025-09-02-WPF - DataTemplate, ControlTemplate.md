---
layout: post
title: WPF - DataTemplate, ControlTemplate
date: 2025-09-02 19:25:23 +0900
category: WPF
---
# DataTemplate, ControlTemplate 활용 완전 가이드

WPF에서 **DataTemplate**는 “데이터 → 표시(뷰)”를, **ControlTemplate**는 “컨트롤의 시각적 트리(스킨)”를 정의합니다.
두 템플릿을 올바르게 쓰면 **MVVM 친화**, **테마 교체 가능**, **재사용성 높은 UI**를 만들 수 있습니다.

---

## 핵심 차이 한눈에

| 항목 | DataTemplate | ControlTemplate |
|---|---|---|
| 목적 | **데이터 개체를 어떻게 그릴지** 정의 | **컨트롤의 외형/구조 자체**를 교체 |
| DataContext | **항목(데이터)** | **TemplatedParent(컨트롤 자신)** |
| 대상 | `ContentControl.ContentTemplate`, `ItemsControl.ItemTemplate`, `HeaderTemplate` 등 | `Control.Template`(Button, ListBox, ScrollViewer 등 모든 Control) |
| 바인딩 키워드 | 일반 `{Binding ...}`(데이터 기준) | `{TemplateBinding ...}`, `RelativeSource TemplatedParent` |
| 전형적 사용 | 리스트 항목 모양, 타입별 뷰 | 버튼/리스트박스의 크롬, VSM 상태(Over/Pressed 등) |

---

## DataTemplate: 데이터 → 뷰

### 기본 사용 (Implicit by DataType)

`DataType`만 지정하면 **해당 타입의 데이터가 표시될 때 자동 적용**됩니다.

```xml
<Window.Resources>
  <DataTemplate DataType="{x:Type vm:Order}">
    <StackPanel Orientation="Horizontal" Spacing="8">
      <TextBlock Text="{Binding Id}" FontWeight="Bold"/>
      <TextBlock Text="{Binding Total, StringFormat={}{0:C}}"/>
    </StackPanel>
  </DataTemplate>
</Window.Resources>

<!-- Orders: ObservableCollection<Order> -->
<ListBox ItemsSource="{Binding Orders}"/>
```

> `x:Key`를 함께 지정하면 **암시적 적용이 꺼지고** 키로만 사용할 수 있습니다.
> **암시적 적용을 원하면 `x:Key`를 생략**하세요.

### ItemsControl / ContentControl 연계

- `ItemsControl.ItemTemplate`: 각 **아이템마다** 적용
- `ContentControl.ContentTemplate`: **단일 콘텐츠**에 적용

```xml
<ContentControl Content="{Binding SelectedOrder}"
                ContentTemplate="{StaticResource SelectedOrderTemplate}"/>
```

### 셀 템플릿 (ListView + GridViewColumn)

```xml
<ListView ItemsSource="{Binding Orders}">
  <ListView.View>
    <GridView>
      <GridViewColumn Header="Id" DisplayMemberBinding="{Binding Id}"/>
      <GridViewColumn Header="Total">
        <GridViewColumn.CellTemplate>
          <DataTemplate>
            <TextBlock Text="{Binding Total, StringFormat={}{0:C}}"
                       Foreground="{Binding Total, Converter={StaticResource PositiveGreen}}"/>
          </DataTemplate>
        </GridViewColumn.CellTemplate>
      </GridViewColumn>
    </GridView>
  </ListView.View>
</ListView>
```

### HierarchicalDataTemplate (트리)

```xml
<TreeView ItemsSource="{Binding Departments}">
  <TreeView.ItemTemplate>
    <HierarchicalDataTemplate ItemsSource="{Binding Members}">
      <TextBlock Text="{Binding Name}"/>
    </HierarchicalDataTemplate>
  </TreeView.ItemTemplate>
</TreeView>
```

### DataTemplateSelector (런타임 선택)

여러 조건(타입·상태)에 따라 **템플릿을 동적으로 선택**합니다.

```csharp
public class OrderTemplateSelector : DataTemplateSelector
{
    public DataTemplate? Normal { get; set; }
    public DataTemplate? Vip { get; set; }

    public override DataTemplate SelectTemplate(object item, DependencyObject container)
    {
        if (item is Order o && o.Total > 1_000_000) return Vip!;
        return Normal!;
    }
}
```

```xml
<Window.Resources>
  <DataTemplate x:Key="OrderNormal">
    <TextBlock Text="{Binding Id}"/>
  </DataTemplate>
  <DataTemplate x:Key="OrderVip">
    <TextBlock Text="{Binding Id}" FontWeight="Bold" Foreground="Gold"/>
  </DataTemplate>

  <local:OrderTemplateSelector x:Key="OrderSelector"
                               Normal="{StaticResource OrderNormal}"
                               Vip="{StaticResource OrderVip}"/>
</Window.Resources>

<ListBox ItemsSource="{Binding Orders}" ItemTemplateSelector="{StaticResource OrderSelector}"/>
```

### 템플릿 내부 트리거

```xml
<DataTemplate DataType="{x:Type vm:Order}">
  <Border Padding="8">
    <Border.Style>
      <Style TargetType="Border">
        <Setter Property="Background" Value="Transparent"/>
        <Style.Triggers>
          <DataTrigger Binding="{Binding IsOverdue}" Value="True">
            <Setter Property="Background" Value="#FFFDECEC"/>
          </DataTrigger>
        </Style.Triggers>
      </Style>
    </Border.Style>
    <StackPanel Orientation="Horizontal" Spacing="8">
      <TextBlock Text="{Binding Id}"/>
      <TextBlock Text="{Binding DueDate, StringFormat={}{0:yyyy-MM-dd}}"/>
    </StackPanel>
  </Border>
</DataTemplate>
```

### 컨텍스트 주의점

- DataTemplate 내부 `DataContext`는 **항목 자체**입니다.
- 상위 ViewModel 속성이 필요하면:
```xml
<Button Content="삭제"
        Command="{Binding DataContext.DeleteCommand, RelativeSource={RelativeSource AncestorType=Window}}"
        CommandParameter="{Binding .}"/>
```

---

## ControlTemplate: 컨트롤의 스킨

### 기본 구조

- **컨트롤의 시각 트리**를 재정의합니다.
- 내부에서 컨트롤의 속성은 **`{TemplateBinding ...}`** 또는
  **`{Binding RelativeSource={RelativeSource TemplatedParent}, Path=...}`**로 참조합니다.
- **ItemsControl**의 ControlTemplate에는 **반드시 `ItemsPresenter`**가 있어야 아이템이 표시됩니다.

```xml
<Style TargetType="Button" x:Key="RoundedPrimaryButton">
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="Button">
        <Border CornerRadius="12"
                Background="{TemplateBinding Background}"
                Padding="{TemplateBinding Padding}">
          <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
        </Border>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
  <Setter Property="Background" Value="#4F46E5"/>
  <Setter Property="Foreground" Value="White"/>
  <Setter Property="Padding" Value="10,6"/>
</Style>
```

### VisualStateManager (상태별 시각)

**Normal/MouseOver/Pressed/Disabled** 같은 시각 상태를 정의합니다.

```xml
<ControlTemplate TargetType="Button" x:Key="VsmButton">
  <Grid x:Name="Root">
    <VisualStateManager.VisualStateGroups>
      <VisualStateGroup x:Name="CommonStates">
        <VisualState x:Name="Normal"/>
        <VisualState x:Name="MouseOver">
          <Storyboard>
            <DoubleAnimation Storyboard.TargetName="Overlay" Storyboard.TargetProperty="Opacity"
                             To="0.08" Duration="0:0:0.08"/>
          </Storyboard>
        </VisualState>
        <VisualState x:Name="Pressed">
          <Storyboard>
            <DoubleAnimation Storyboard.TargetName="Overlay" Storyboard.TargetProperty="Opacity"
                             To="0.16" Duration="0:0:0.05"/>
          </Storyboard>
        </VisualState>
        <VisualState x:Name="Disabled">
          <Storyboard>
            <DoubleAnimation Storyboard.TargetName="ContentHost" Storyboard.TargetProperty="Opacity"
                             To="0.5" Duration="0:0:0.1"/>
          </Storyboard>
        </VisualState>
      </VisualStateGroup>
    </VisualStateManager.VisualStateGroups>

    <Border Background="{TemplateBinding Background}" CornerRadius="10">
      <Grid>
        <ContentPresenter x:Name="ContentHost"
                          HorizontalAlignment="Center" VerticalAlignment="Center"
                          Margin="{TemplateBinding Padding}"/>
        <Border x:Name="Overlay" Background="Black" Opacity="0" CornerRadius="10"/>
      </Grid>
    </Border>
  </Grid>
</ControlTemplate>
```

### ItemsControl의 ControlTemplate (ItemsPresenter 필수)

```xml
<Style TargetType="ListBox">
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="ListBox">
        <Border BorderBrush="{TemplateBinding BorderBrush}" BorderThickness="{TemplateBinding BorderThickness}">
          <ScrollViewer Focusable="False">
            <!-- 아이템이 여기서 렌더됨 -->
            <ItemsPresenter/>
          </ScrollViewer>
        </Border>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```

### TemplateBinding vs TemplatedParent

- **`{TemplateBinding Property=...}`**: **가볍고 빠른 OneWay** 바인딩 (ControlTemplate 전용)
- **`RelativeSource TemplatedParent`**: 일반 바인딩 문법(Converter, MultiBinding 등 확장 가능)

```xml
<!-- 단순 전달 → TemplateBinding -->
Background="{TemplateBinding Background}"

<!-- 변환/복합 필요 → TemplatedParent -->
{Binding Path=Background, RelativeSource={RelativeSource TemplatedParent}, Converter={StaticResource ...}}
```

### 커스텀 컨트롤 & PART 규약

일부 컨트롤은 템플릿에 **필수 파트**가 필요합니다(예: `PART_Editor`).
커스텀 컨트롤을 만들 때 `[TemplatePart]`로 계약을 표시하고, `OnApplyTemplate`에서 찾아 사용합니다.

```csharp
[TemplatePart(Name = "PART_Box", Type = typeof(TextBox))]
public class SearchBox : Control
{
    private TextBox? _box;
    static SearchBox() => DefaultStyleKeyProperty.OverrideMetadata(
        typeof(SearchBox), new FrameworkPropertyMetadata(typeof(SearchBox)));

    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        _box = GetTemplateChild("PART_Box") as TextBox;
    }
}
```

```xml
<!-- Themes/Generic.xaml -->
<Style TargetType="{x:Type local:SearchBox}">
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="{x:Type local:SearchBox}">
        <Border>
          <Grid>
            <TextBox x:Name="PART_Box" />
            <Button Content="⟲" HorizontalAlignment="Right" VerticalAlignment="Center"/>
          </Grid>
        </Border>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```

> 커스텀 컨트롤의 기본 템플릿은 **`Themes/Generic.xaml`**에 둡니다.

---

## ItemsTemplate vs ItemContainerStyle vs ItemsPanel

- **ItemTemplate / ItemTemplateSelector**: **각 데이터 항목의 모양**
- **ItemContainerStyle**: **컨테이너(ListBoxItem, TreeViewItem)의 속성/상태**
- **ItemsPanel**: 아이템을 배치하는 **패널(Grid/WrapPanel/VirtualizingStackPanel 등)**

```xml
<ListBox ItemsSource="{Binding Orders}">
  <ListBox.ItemTemplate>
    <!-- 데이터 모양 -->
    <DataTemplate>
      <TextBlock Text="{Binding Id}"/>
    </DataTemplate>
  </ListBox.ItemTemplate>

  <ListBox.ItemContainerStyle>
    <!-- 컨테이너 스타일(선택/포커스 등) -->
    <Style TargetType="ListBoxItem">
      <Setter Property="Padding" Value="8"/>
      <Setter Property="HorizontalContentAlignment" Value="Stretch"/>
    </Style>
  </ListBox.ItemContainerStyle>

  <ListBox.ItemsPanel>
    <!-- 배치 패널 -->
    <ItemsPanelTemplate>
      <VirtualizingStackPanel/>
    </ItemsPanelTemplate>
  </ListBox.ItemsPanel>
</ListBox>
```

---

## 성능 & 구조 팁

- **가상화**: 대량 목록은 `VirtualizingStackPanel.IsVirtualizing="True"`(기본) 유지, `ScrollUnit="Pixel"` 검토
- **Reusable 리소스**: 브러시/그라디언트/Geometry는 리소스로 올려 **공유** (가능하면 Freezable Freeze)
- **Trigger 남발 지양**: 복잡한 상태는 VSM 또는 간결한 스타일로
- **DataTemplate 내부의 무거운 컨트롤**(예: WebBrowser) 최소화
- **StaticResource 우선**, 런타임 테마 전환만 **DynamicResource**

---

## 언제 어떤 템플릿을 쓸까?

- **데이터가 다르면 모양도 달라야** → **DataTemplate / Selector**
- **같은 컨트롤인데 스킨을 바꾸고 싶다** → **ControlTemplate**
- **리스트 항목의 외형 vs 선택/포커스 스타일** → **ItemTemplate vs ItemContainerStyle**
- **배치 방식 변경(그리드/타일/워터폴)** → **ItemsPanel(…Template)**

---

## 흔한 실수 체크리스트

- **ItemsPresenter 누락**(ItemsControl 템플릿) → 아이템이 안 보임
- ControlTemplate 안에서 **일반 `{Binding}`으로 부모 속성 접근** → **`TemplatedParent`**로 바인딩
- DataTemplate 내부에서 **상위 ViewModel 속성 접근 실패** → `RelativeSource AncestorType` 사용
- DataType 템플릿에 **`x:Key` 함께 지정** → 암시적 적용이 안 됨
- `TemplateBinding`으로 **Converter/MultiBinding** 쓰려다 실패 → TemplatedParent 바인딩 사용

---

## 종합 예: 타입별 DataTemplate + 스킨 교체 Button

```xml
<Window.Resources>
  <!-- 타입별 DataTemplate (암시적) -->
  <DataTemplate DataType="{x:Type vm:ErrorLog}">
    <StackPanel Orientation="Horizontal" Spacing="6">
      <TextBlock Text="❗" />
      <TextBlock Text="{Binding Message}" Foreground="Tomato"/>
    </StackPanel>
  </DataTemplate>
  <DataTemplate DataType="{x:Type vm:InfoLog}">
    <StackPanel Orientation="Horizontal" Spacing="6">
      <TextBlock Text="ℹ" />
      <TextBlock Text="{Binding Message}" Foreground="SteelBlue"/>
    </StackPanel>
  </DataTemplate>

  <!-- 버튼 스킨 -->
  <Style TargetType="Button" x:Key="PillButton">
    <Setter Property="Background" Value="#0EA5E9"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="Padding" Value="10,6"/>
    <Setter Property="Template">
      <Setter.Value>
        <ControlTemplate TargetType="Button">
          <Border x:Name="Chrome" Background="{TemplateBinding Background}" CornerRadius="999">
            <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"
                              Margin="{TemplateBinding Padding}"/>
          </Border>
          <ControlTemplate.Triggers>
            <Trigger Property="IsMouseOver" Value="True">
              <Setter TargetName="Chrome" Property="Opacity" Value="0.9"/>
            </Trigger>
            <Trigger Property="IsPressed" Value="True">
              <Setter TargetName="Chrome" Property="RenderTransform">
                <Setter.Value>
                  <ScaleTransform ScaleX="0.98" ScaleY="0.98"/>
                </Setter.Value>
              </Setter>
            </Trigger>
            <Trigger Property="IsEnabled" Value="False">
              <Setter TargetName="Chrome" Property="Opacity" Value="0.5"/>
            </Trigger>
          </ControlTemplate.Triggers>
        </ControlTemplate>
      </Setter.Value>
    </Setter>
  </Style>
</Window.Resources>

<Grid Margin="16">
  <Grid.RowDefinitions>
    <RowDefinition Height="Auto"/>
    <RowDefinition/>
  </Grid.RowDefinitions>

  <Button Content="새 로그" Style="{StaticResource PillButton}" Command="{Binding AddLogCommand}"/>

  <!-- 서로 다른 타입(ErrorLog/InfoLog)이 한 리스트에 섞여 있어도
       각 타입의 DataTemplate가 자동 적용됨 -->
  <ListBox Grid.Row="1" ItemsSource="{Binding Logs}"/>
</Grid>
```

---

### 결론

- **DataTemplate**: 데이터의 “표현”을 선언하고, **ControlTemplate**: 컨트롤의 “스킨/구조”를 교체합니다.
- 템플릿, 스타일, ItemsPanel을 **역할별로 분리**하면 **테마 교체**, **확장성**, **성능**까지 챙길 수 있습니다.
- 위 원칙과 예제를 베이스로 프로젝트 템플릿(Styles/ControlTemplates/DataTemplates/Selectors)을 구성하면
  **MVVM 친화적인 프로 UI**를 안정적으로 구축할 수 있습니다.
