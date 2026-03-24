---
layout: post
title: WPF - DynamicResource vs StaticResource
date: 2025-09-07 14:25:23 +0900
category: WPF
---
# WPF DynamicResource와 StaticResource: 심층 가이드

WPF의 리소스 시스템에서 `DynamicResource`와 `StaticResource`의 차이는 단순한 선택 문제를 넘어 애플리케이션의 동작 방식과 성능에 영향을 미치는 근본적인 설계 결정입니다. 이 두 가지 리소스 참조 방식을 깊이 이해하는 것은 복잡한 UI를 구축하고 유지보수하는 데 필수적입니다.

## 기본 개념: 시점과 변경 추적

리소스 참조의 가장 근본적인 차이는 **해석 시점**과 **변경 추적 능력**에 있습니다. `StaticResource`는 로드 시점에 단 한 번 평가되어 고정된 값으로 변환되는 반면, `DynamicResource`는 런타임에 지연 평가되고 변경 사항을 지속적으로 추적합니다.

이 차이를 실제로 이해하기 위해, 테마 시스템을 구현하는 일반적인 시나리오를 살펴보겠습니다.

```xml
<!-- Light 테마 리소스 -->
<ResourceDictionary>
    <SolidColorBrush x:Key="PrimaryBrush" Color="#2563EB"/>
</ResourceDictionary>
```

```xml
<!-- StaticResource로 참조 -->
<Button Background="{StaticResource PrimaryBrush}"/>
```

테마를 런타임에 다크 테마로 교체해도, `StaticResource`로 바인딩된 버튼의 배경은 변하지 않습니다. 반면 `DynamicResource`를 사용하면 자동으로 업데이트됩니다.

```xml
<Button Background="{DynamicResource PrimaryBrush}"/>
```

## 내부 동작 메커니즘

### StaticResource

- **파싱 단계**: XAML 파서가 `StaticResource` 확장을 만나면 즉시 리소스 탐색을 시작합니다.
- **리소스 해석**: 현재 범위(요소 자체 → 부모 → 애플리케이션 → 시스템)에서 키를 찾아 값을 가져옵니다.
- **값 고정**: 찾은 값을 속성에 직접 할당합니다. 이후 변경되지 않습니다.
- **메모리 관리**: 원본 리소스 객체에 대한 참조를 유지하지 않습니다.

### DynamicResource

- **지연 평가**: 파싱 시점에 실제 값을 가져오지 않고, `ResourceReferenceExpression` 객체를 생성합니다.
- **구독 시스템**: 리소스 키에 대한 변경 알림 시스템에 등록합니다.
- **런타임 평가**: 요소가 실제로 로드될 때 리소스 값을 조회합니다.
- **변경 추적**: 리소스 사전에서 해당 키의 값이 변경되면 모든 구독자에게 알립니다.

```csharp
// DynamicResource의 의사코드
public class DynamicResourceExtension
{
    public object ProvideValue(IServiceProvider serviceProvider)
    {
        var resourceKey = this.ResourceKey;
        var targetObject = GetTargetObject(serviceProvider);
        var targetProperty = GetTargetProperty(serviceProvider);
        
        // 리소스 변경 알림 시스템에 등록
        ResourceChangeNotifier.Register(targetObject, targetProperty, resourceKey);
        
        // 현재 값 찾아서 초기화
        var currentValue = FindResource(targetObject, resourceKey);
        if (currentValue != null)
            targetObject.SetValue(targetProperty, currentValue);
        
        return new ResourceReferenceExpression(resourceKey);
    }
}
```

## 실제 사례: 동적 테마 시스템

테마 전환 시 `DynamicResource`가 필수적입니다. 다음은 간단한 테마 관리자 예제입니다.

```csharp
public class ThemeManager
{
    public void SwitchToDarkTheme()
    {
        var app = Application.Current;
        var dict = new ResourceDictionary { Source = new Uri("DarkTheme.xaml", UriKind.Relative) };
        app.Resources.MergedDictionaries.Clear();
        app.Resources.MergedDictionaries.Add(dict);
        // DynamicResource를 사용한 모든 UI가 자동 업데이트됨
    }
}
```

리소스 사전 구조를 계층화하여 유지보수성을 높일 수 있습니다.

```
Themes/
├── Base/
│   ├── Colors.xaml      # 색상 키 정의
│   └── Brushes.xaml     # 브러시 정의 (DynamicResource로 색상 참조)
├── Light/
│   └── LightColors.xaml # 실제 색상 값
└── Dark/
    └── DarkColors.xaml
```

```xml
<!-- Base/Brushes.xaml -->
<SolidColorBrush x:Key="PrimaryBrush" Color="{DynamicResource Color.Primary}"/>
```

```xml
<!-- Light/LightColors.xaml -->
<Color x:Key="Color.Primary">#2563EB</Color>
```

## 성능 고려사항

`DynamicResource`는 유연하지만 성능 비용이 있습니다. 다음 표는 성능 특성을 비교한 것입니다.

| 항목 | StaticResource | DynamicResource |
|------|----------------|-----------------|
| 초기 로드 속도 | 빠름 | 느림 (리소스 탐색 지연) |
| 변경 감지 오버헤드 | 없음 | 있음 (구독/알림) |
| 메모리 사용량 | 낮음 | 중간 (식별자 및 구독 정보) |
| 런타임 업데이트 | 불가능 | 가능 |

**최적화 전략**
- 자주 변경되지 않는 리소스는 `StaticResource`를 사용합니다.
- 컨트롤 템플릿 내부에서는 `StaticResource`를 우선 고려합니다.
- 리소스 키에 접두사를 붙여 사용 목적을 명확히 합니다(예: `Brush.Primary`는 동적, `Static.Primary`는 정적).

```xml
<!-- 효율적인 하이브리드 접근 -->
<Style TargetType="Button">
    <Setter Property="Padding" Value="{StaticResource Thickness.Padding}"/>
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <Border Background="{TemplateBinding Background}">
                    <ContentPresenter/>
                </Border>
                <ControlTemplate.Triggers>
                    <Trigger Property="IsMouseOver" Value="True">
                        <!-- 동적 변경이 필요한 부분만 DynamicResource -->
                        <Setter Property="Background" Value="{DynamicResource Brush.Hover}"/>
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

## 일반적인 문제와 해결책

### 문제: 리소스가 변경되지 않음

```csharp
// ❌ 잘못된 방법: 기존 브러시 속성 변경
var brush = Application.Current.Resources["PrimaryBrush"] as SolidColorBrush;
brush.Color = Colors.Red;  // DynamicResource가 감지하지 못함

// ✅ 올바른 방법: 새 인스턴스로 교체
var newBrush = new SolidColorBrush(Colors.Red);
Application.Current.Resources["PrimaryBrush"] = newBrush;
```

### 문제: 리소스 범위 오류

디버깅 시 리소스 탐색 경로를 추적하려면 `FindResource` 호출 스택을 확인하거나 `ResourceScopeDebugger` 같은 도우미를 사용할 수 있습니다.

```csharp
public static void TraceResourceLookup(string key, DependencyObject element)
{
    var current = element;
    while (current != null)
    {
        if (current is FrameworkElement fe && fe.Resources?.Contains(key) == true)
            Debug.WriteLine($"Found in {fe.GetType().Name}");
        current = VisualTreeHelper.GetParent(current) ?? LogicalTreeHelper.GetParent(current);
    }
}
```

## 결론: 상황에 맞는 최적의 선택

`DynamicResource`와 `StaticResource`의 선택은 **변경 가능성**에 기반해야 합니다.

- **런타임에 변경될 수 있는 값**(테마 색상, 사용자 설정)에는 `DynamicResource`를 사용합니다.
- **정적 값**(여백, 폰트 크기, 애플리케이션 구조적 상수)에는 `StaticResource`를 사용하여 성능을 최적화합니다.
- **대규모 애플리케이션**에서는 리소스 키 네이밍 규칙을 정하고, 변경이 예상되는 리소스와 그렇지 않은 리소스를 구분하여 관리합니다.
- **프로파일링**을 통해 실제 성능 영향을 측정하고, 필요 시 `DynamicResource` 사용 범위를 조정합니다.

적절히 조화된 리소스 관리 전략은 WPF 애플리케이션의 품질과 유지보수성을 결정하는 중요한 요소입니다.