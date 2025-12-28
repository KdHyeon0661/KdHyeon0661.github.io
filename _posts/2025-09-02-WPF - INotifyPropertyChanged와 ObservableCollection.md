---
layout: post
title: WPF - INotifyPropertyChanged와 ObservableCollection
date: 2025-09-02 18:25:23 +0900
category: WPF
---
# WPF 데이터 변경 알림: INotifyPropertyChanged와 ObservableCollection

## 데이터 동기화의 핵심 메커니즘

WPF의 데이터 바인딩 시스템은 UI와 데이터 소스 사이의 자동 동기화를 가능하게 하는 강력한 기능입니다. 이 시스템의 핵심에는 두 가지 중요한 인터페이스가 있습니다: **INotifyPropertyChanged**와 **ObservableCollection**. 이들은 각각 개별 속성과 컬렉션의 변경을 UI에 알리는 역할을 하여, 사용자 인터페이스가 항상 최신 데이터를 반영하도록 보장합니다.

---

## INotifyPropertyChanged: 속성 변경 알림의 근간

### 인터페이스의 본질과 중요성

INotifyPropertyChanged는 .NET의 핵심 인터페이스 중 하나로, 객체의 속성 값이 변경되었을 때 이를 외부에 알리는 표준화된 방법을 제공합니다. 이 인터페이스는 단순한 기술적 구현을 넘어서, 데이터 주도 UI의 철학적 기반을 이루고 있습니다.

```csharp
// INotifyPropertyChanged의 정의
public interface INotifyPropertyChanged
{
    event PropertyChangedEventHandler PropertyChanged;
}
```

### 구현 패턴과 모범 사례

효과적인 INotifyPropertyChanged 구현은 단순한 이벤트 발생을 넘어서 성능 최적화와 코드 가독성을 고려해야 합니다:

```csharp
// 전문적인 INotifyPropertyChanged 구현 예시
public abstract class ObservableObject : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;
    
    // 최적화된 속성 설정 메서드
    protected bool SetProperty<T>(ref T field, T value, 
                                  [CallerMemberName] string propertyName = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value))
            return false;
            
        field = value;
        OnPropertyChanged(propertyName);
        return true;
    }
    
    // 여러 속성을 한 번에 알리는 메서드
    protected void OnPropertiesChanged(params string[] propertyNames)
    {
        foreach (var name in propertyNames)
        {
            OnPropertyChanged(name);
        }
    }
    
    // UI 스레드 안전한 알림
    protected virtual void OnPropertyChanged(string propertyName)
    {
        // UI 스레드 확인
        if (Application.Current?.Dispatcher?.CheckAccess() ?? true)
        {
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
        }
        else
        {
            Application.Current.Dispatcher.Invoke(() =>
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName)));
        }
    }
}

// 실제 ViewModel에서의 사용
public class ProductViewModel : ObservableObject
{
    private string _name;
    private decimal _price;
    private int _quantity;
    
    public string Name
    {
        get => _name;
        set => SetProperty(ref _name, value);
    }
    
    public decimal Price
    {
        get => _price;
        set
        {
            if (SetProperty(ref _price, value))
            {
                // 관련 속성도 함께 알림
                OnPropertyChanged(nameof(TotalPrice));
                OnPropertyChanged(nameof(IsExpensive));
            }
        }
    }
    
    public int Quantity
    {
        get => _quantity;
        set
        {
            if (SetProperty(ref _quantity, value))
            {
                OnPropertyChanged(nameof(TotalPrice));
                OnPropertyChanged(nameof(IsLowStock));
            }
        }
    }
    
    // 계산된 속성들
    public decimal TotalPrice => Price * Quantity;
    public bool IsExpensive => Price > 1000;
    public bool IsLowStock => Quantity < 10;
}
```

### 동작 원리의 깊이 있는 이해

INotifyPropertyChanged의 동작은 단순한 이벤트 발생 이상의 복잡한 프로세스를 포함합니다:

1. **속성 변경 감지**: Setter에서 값이 실제로 변경되었는지 확인
2. **이벤트 발생**: PropertyChanged 이벤트를 통해 변경 사실 통지
3. **WPF 바인딩 엔진 처리**: 
   - 이벤트 구독자에게 알림 전파
   - 바인딩된 UI 요소 식별
   - 새 값으로 UI 업데이트
4. **레이아웃 재계산**: 필요한 경우 UI 레이아웃 재조정

이 프로세스는 완전히 자동화되어 있어, 개발자는 데이터 변경 로직에만 집중할 수 있습니다.

---

## ObservableCollection: 컬렉션 변경 알림의 표준

### 컬렉션 변경 알림의 필요성

단일 속성의 변경 알림이 중요한 것처럼, 컬렉션의 구조적 변경(항목 추가, 삭제, 이동)도 UI에 즉시 반영되어야 합니다. ObservableCollection<T>는 이 요구사항을 완벽하게 충족하는 컬렉션 구현체입니다.

```csharp
// ObservableCollection의 계층 구조
public class ObservableCollection<T> : 
    Collection<T>, 
    INotifyCollectionChanged,    // 컬렉션 구조 변경 알림
    INotifyPropertyChanged       // Count 등 속성 변경 알림
{
    // 내부 구현...
}
```

### ObservableCollection의 실제 활용

```csharp
public class OrderManagementViewModel : ObservableObject
{
    // 주문 목록 - ObservableCollection 사용
    public ObservableCollection<Order> Orders { get; } = new();
    
    // 필터링된 주문 목록
    public ObservableCollection<Order> FilteredOrders { get; } = new();
    
    private OrderStatus _filterStatus;
    public OrderStatus FilterStatus
    {
        get => _filterStatus;
        set
        {
            if (SetProperty(ref _filterStatus, value))
            {
                ApplyOrderFilter();
            }
        }
    }
    
    public OrderManagementViewModel()
    {
        // 초기 데이터 로드
        LoadInitialOrders();
        
        // 컬렉션 변경 이벤트 구독
        Orders.CollectionChanged += OnOrdersCollectionChanged;
    }
    
    private void LoadInitialOrders()
    {
        Orders.Add(new Order { Id = "ORD-001", Status = OrderStatus.Pending });
        Orders.Add(new Order { Id = "ORD-002", Status = OrderStatus.Processing });
        Orders.Add(new Order { Id = "ORD-003", Status = OrderStatus.Completed });
    }
    
    private void OnOrdersCollectionChanged(object sender, NotifyCollectionChangedEventArgs e)
    {
        // 컬렉션 변경 시 추가 작업
        switch (e.Action)
        {
            case NotifyCollectionChangedAction.Add:
                // 새 항목 추가 시 처리
                foreach (Order newOrder in e.NewItems)
                {
                    Console.WriteLine($"Added order: {newOrder.Id}");
                }
                break;
                
            case NotifyCollectionChangedAction.Remove:
                // 항목 삭제 시 처리
                foreach (Order removedOrder in e.OldItems)
                {
                    Console.WriteLine($"Removed order: {removedOrder.Id}");
                }
                break;
                
            case NotifyCollectionChangedAction.Reset:
                // 전체 컬렉션 재설정 시 처리
                Console.WriteLine("Collection was reset");
                break;
        }
        
        // 필터 적용
        ApplyOrderFilter();
    }
    
    private void ApplyOrderFilter()
    {
        FilteredOrders.Clear();
        
        if (FilterStatus == OrderStatus.All)
        {
            foreach (var order in Orders)
            {
                FilteredOrders.Add(order);
            }
        }
        else
        {
            foreach (var order in Orders.Where(o => o.Status == FilterStatus))
            {
                FilteredOrders.Add(order);
            }
        }
    }
    
    public void AddOrder(Order order)
    {
        Orders.Add(order);
    }
    
    public void RemoveOrder(string orderId)
    {
        var order = Orders.FirstOrDefault(o => o.Id == orderId);
        if (order != null)
        {
            Orders.Remove(order);
        }
    }
}
```

### 컬렉션 변경 알림의 세밀한 제어

ObservableCollection은 다양한 수준의 변경 알림을 지원합니다:

```csharp
public class AdvancedCollectionViewModel
{
    public ObservableCollection<DataItem> Items { get; }
    
    public AdvancedCollectionViewModel()
    {
        Items = new ObservableCollection<DataItem>();
        
        // 대량 데이터 추가 시 성능 최적화
        AddItemsInBulk();
    }
    
    private void AddItemsInBulk()
    {
        // 대량 추가를 위한 최적화
        using (Items.DeferRefresh())
        {
            for (int i = 0; i < 1000; i++)
            {
                Items.Add(new DataItem { Id = i, Value = $"Item {i}" });
            }
        }
        // DeferRefresh 블록을 벗어나면 한 번의 CollectionChanged 이벤트 발생
    }
    
    public void UpdateItem(int id, string newValue)
    {
        var item = Items.FirstOrDefault(i => i.Id == id);
        if (item != null)
        {
            // 개별 항목 속성 변경
            item.Value = newValue;
            
            // 항목 자체는 변경되지 않았으므로 CollectionChanged 이벤트가 발생하지 않음
            // 단, DataItem이 INotifyPropertyChanged를 구현해야 UI에 반영됨
        }
    }
    
    public void MoveItemToTop(int id)
    {
        var item = Items.FirstOrDefault(i => i.Id == id);
        if (item != null)
        {
            int oldIndex = Items.IndexOf(item);
            Items.Move(oldIndex, 0); // Move 작업은 CollectionChanged 이벤트 발생
        }
    }
}
```

---

## INotifyPropertyChanged와 ObservableCollection의 통합 활용

### 실제 시나리오에서의 협력

이 두 인터페이스는 함께 사용될 때 가장 강력한 시너지를 발휘합니다:

```csharp
public class DashboardViewModel : ObservableObject
{
    private string _dashboardTitle = "실시간 대시보드";
    public string DashboardTitle
    {
        get => _dashboardTitle;
        set => SetProperty(ref _dashboardTitle, value);
    }
    
    private DateTime _lastUpdated;
    public DateTime LastUpdated
    {
        get => _lastUpdated;
        private set => SetProperty(ref _lastUpdated, value);
    }
    
    // 실시간 데이터 스트림
    public ObservableCollection<Metric> LiveMetrics { get; } = new();
    
    // 역사적 데이터
    public ObservableCollection<HistoricalData> HistoricalData { get; } = new();
    
    // 선택된 메트릭
    private Metric _selectedMetric;
    public Metric SelectedMetric
    {
        get => _selectedMetric;
        set
        {
            if (SetProperty(ref _selectedMetric, value))
            {
                // 선택 변경 시 관련 데이터 새로고침
                LoadMetricDetails(value);
                UpdateRelatedCharts(value);
            }
        }
    }
    
    // 데이터 로딩 상태
    private bool _isLoading;
    public bool IsLoading
    {
        get => _isLoading;
        set
        {
            if (SetProperty(ref _isLoading, value))
            {
                // 로딩 상태 변경 시 명령 가용성 업데이트
                RefreshCommand.RaiseCanExecuteChanged();
            }
        }
    }
    
    // 명령들
    public ICommand RefreshCommand { get; }
    public ICommand AddMetricCommand { get; }
    public ICommand RemoveMetricCommand { get; }
    
    public DashboardViewModel()
    {
        // 명령 초기화
        RefreshCommand = new RelayCommand(async () => await RefreshDataAsync());
        AddMetricCommand = new RelayCommand(AddNewMetric);
        RemoveMetricCommand = new RelayCommand(
            () => RemoveSelectedMetric(),
            () => SelectedMetric != null);
        
        // 초기 데이터 로드
        InitializeAsync();
    }
    
    private async Task InitializeAsync()
    {
        IsLoading = true;
        try
        {
            await LoadInitialDataAsync();
            StartLiveDataStream();
        }
        finally
        {
            IsLoading = false;
        }
    }
    
    private async Task LoadInitialDataAsync()
    {
        var metrics = await DataService.GetMetricsAsync();
        
        // ObservableCollection에 데이터 추가
        LiveMetrics.Clear();
        foreach (var metric in metrics)
        {
            LiveMetrics.Add(metric);
        }
        
        LastUpdated = DateTime.Now;
    }
    
    private void StartLiveDataStream()
    {
        // 실시간 데이터 구독
        DataService.MetricUpdated += OnMetricUpdated;
    }
    
    private void OnMetricUpdated(object sender, Metric updatedMetric)
    {
        // UI 스레드에서 컬렉션 업데이트
        Application.Current.Dispatcher.Invoke(() =>
        {
            var existingMetric = LiveMetrics.FirstOrDefault(m => m.Id == updatedMetric.Id);
            if (existingMetric != null)
            {
                // 항목 업데이트
                int index = LiveMetrics.IndexOf(existingMetric);
                LiveMetrics[index] = updatedMetric;
            }
            else
            {
                // 새 항목 추가
                LiveMetrics.Add(updatedMetric);
            }
            
            LastUpdated = DateTime.Now;
        });
    }
    
    private void AddNewMetric()
    {
        var newMetric = new Metric
        {
            Id = Guid.NewGuid(),
            Name = "새 메트릭",
            Value = 0,
            Timestamp = DateTime.Now
        };
        
        LiveMetrics.Add(newMetric);
        SelectedMetric = newMetric;
    }
    
    private void RemoveSelectedMetric()
    {
        if (SelectedMetric != null)
        {
            LiveMetrics.Remove(SelectedMetric);
            SelectedMetric = null;
        }
    }
}
```

### 성능 고려사항과 최적화 전략

대규모 애플리케이션에서는 변경 알림의 성능 영향을 고려해야 합니다:

```csharp
public class OptimizedViewModel : ObservableObject
{
    // 대용량 컬렉션 처리를 위한 최적화
    private readonly object _collectionLock = new object();
    
    // 가상화를 위한 컬렉션
    public VirtualizingObservableCollection<LargeItem> LargeCollection { get; }
    
    // 일괄 업데이트를 위한 버퍼
    private readonly List<DataUpdate> _updateBuffer = new();
    private readonly System.Timers.Timer _updateTimer;
    
    public OptimizedViewModel()
    {
        LargeCollection = new VirtualizingObservableCollection<LargeItem>();
        
        // 타이머 기반 일괄 업데이트
        _updateTimer = new System.Timers.Timer(100); // 100ms 간격
        _updateTimer.Elapsed += ProcessUpdateBuffer;
        _updateTimer.Start();
        
        // 초기 데이터 로드
        LoadDataInBackground();
    }
    
    private async void LoadDataInBackground()
    {
        // 백그라운드 스레드에서 데이터 로드
        await Task.Run(() =>
        {
            var data = DataService.GetLargeDataSet();
            
            // UI 스레드로 마샬링하여 컬렉션 업데이트
            Application.Current.Dispatcher.Invoke(() =>
            {
                using (LargeCollection.DeferRefresh())
                {
                    foreach (var item in data)
                    {
                        LargeCollection.Add(item);
                    }
                }
            });
        });
    }
    
    public void QueueUpdate(DataUpdate update)
    {
        lock (_updateBuffer)
        {
            _updateBuffer.Add(update);
        }
    }
    
    private void ProcessUpdateBuffer(object sender, System.Timers.ElapsedEventArgs e)
    {
        List<DataUpdate> bufferCopy;
        
        lock (_updateBuffer)
        {
            if (_updateBuffer.Count == 0) return;
            
            bufferCopy = new List<DataUpdate>(_updateBuffer);
            _updateBuffer.Clear();
        }
        
        // UI 스레드에서 일괄 처리
        Application.Current.Dispatcher.Invoke(() =>
        {
            using (LargeCollection.DeferRefresh())
            {
                foreach (var update in bufferCopy)
                {
                    ProcessSingleUpdate(update);
                }
            }
        });
    }
    
    private void ProcessSingleUpdate(DataUpdate update)
    {
        // 개별 업데이트 처리 로직
    }
}
```

---

## 고급 패턴과 실제 문제 해결

### 복합 속성과 컬렉션의 변경 알림

복잡한 데이터 구조에서의 변경 알림 처리:

```csharp
public class ComplexViewModel : ObservableObject
{
    private Order _selectedOrder;
    public Order SelectedOrder
    {
        get => _selectedOrder;
        set
        {
            if (SetProperty(ref _selectedOrder, value))
            {
                // 이전 주문의 변경 알림 구독 해제
                if (_selectedOrder != null)
                {
                    _selectedOrder.PropertyChanged -= OnOrderPropertyChanged;
                }
                
                // 새 주문의 변경 알림 구독
                if (value != null)
                {
                    value.PropertyChanged += OnOrderPropertyChanged;
                }
                
                OnPropertyChanged(nameof(CanEditOrder));
                OnPropertyChanged(nameof(CanDeleteOrder));
            }
        }
    }
    
    public ObservableCollection<OrderLine> OrderLines { get; } = new();
    
    // 중첩된 컬렉션의 변경 추적
    private void OnOrderPropertyChanged(object sender, PropertyChangedEventArgs e)
    {
        // 주문 속성 변경 시 관련 UI 업데이트
        switch (e.PropertyName)
        {
            case nameof(Order.TotalAmount):
                OnPropertyChanged(nameof(OrderSummary));
                break;
                
            case nameof(Order.Status):
                OnPropertyChanged(nameof(CanEditOrder));
                OnPropertyChanged(nameof(CanDeleteOrder));
                break;
        }
    }
    
    // 컬렉션 내부 항목의 변경 추적
    public ComplexViewModel()
    {
        OrderLines.CollectionChanged += OnOrderLinesChanged;
    }
    
    private void OnOrderLinesChanged(object sender, NotifyCollectionChangedEventArgs e)
    {
        // 새 항목의 변경 알림 구독
        if (e.NewItems != null)
        {
            foreach (OrderLine newLine in e.NewItems)
            {
                newLine.PropertyChanged += OnOrderLinePropertyChanged;
            }
        }
        
        // 제거된 항목의 변경 알림 구독 해제
        if (e.OldItems != null)
        {
            foreach (OrderLine oldLine in e.OldItems)
            {
                oldLine.PropertyChanged -= OnOrderLinePropertyChanged;
            }
        }
        
        // 컬렉션 변경 시 관련 속성 업데이트
        OnPropertyChanged(nameof(TotalLineCount));
        OnPropertyChanged(nameof(OrderLinesTotal));
    }
    
    private void OnOrderLinePropertyChanged(object sender, PropertyChangedEventArgs e)
    {
        // 주문 라인 속성 변경 처리
        if (e.PropertyName == nameof(OrderLine.Quantity) || 
            e.PropertyName == nameof(OrderLine.UnitPrice))
        {
            OnPropertyChanged(nameof(OrderLinesTotal));
        }
    }
    
    // 계산된 속성들
    public int TotalLineCount => OrderLines.Count;
    public decimal OrderLinesTotal => OrderLines.Sum(line => line.Total);
    public string OrderSummary => $"총 {TotalLineCount}개 항목, 합계: {OrderLinesTotal:C}";
    
    public bool CanEditOrder => SelectedOrder?.Status != OrderStatus.Completed;
    public bool CanDeleteOrder => SelectedOrder?.Status == OrderStatus.Pending;
}
```

---

## 결론

INotifyPropertyChanged와 ObservableCollection은 WPF의 데이터 바인딩 시스템을 이루는 두 기둥입니다. 이들의 올바른 이해와 적용은 현대적이고 반응형인 사용자 인터페이스를 구축하는 데 필수적입니다.

### 핵심 통찰

1. **추상화의 힘**: 이 두 인터페이스는 데이터와 UI 사이의 추상화 계층을 제공하여, 비즈니스 로직과 프레젠테이션 로직의 명확한 분리를 가능하게 합니다.

2. **선언적 프로그래밍**: 데이터 변경 알림 시스템은 개발자가 "어떻게" UI를 업데이트할지 명령적으로 코딩하는 대신, "무엇이" 변경되었는지 선언적으로 표현할 수 있게 합니다.

3. **일관성과 신뢰성**: 자동화된 변경 알림은 UI와 데이터 모델 사이의 일관성을 보장하며, 동기화 오류의 가능성을 크게 줄입니다.

### 실무적 권장사항

- **적절한 추상화 수준 선택**: 작은 프로젝트에는 단순한 구현으로 시작하고, 프로젝트가 성장함에 따라 점진적으로 고급 패턴을 도입하세요.

- **성능과 기능의 균형**: 모든 속성에 변경 알림을 적용하는 것이 항상 최선은 아닙니다. 실제로 UI에 반영되어야 하는 속성에만 적용하세요.

- **메모리 관리**: 이벤트 핸들러 구독은 메모리 누수의 일반적인 원인입니다. 적절한 시점에 구독을 해제하세요.

- **테스트 용이성**: 변경 알림 시스템은 ViewModel의 단위 테스트를 용이하게 합니다. 이 점을 최대한 활용하세요.

### 미래를 위한 준비

INotifyPropertyChanged와 ObservableCollection은 시간이 지나도 변하지 않는 견고한 패턴이지만, 새로운 기술과의 통합을 고려하는 것도 중요합니다:

- **리액티브 확장(Reactive Extensions)**: 복잡한 데이터 흐름 시나리오에는 Rx를 고려하세요.
- **소스 제너레이터**: C# 9.0의 소스 제너레이터를 활용하여 보일러플레이트 코드를 자동 생성하세요.
- **불변성(Immutability)**: 가능한 경우 불변 데이터 구조를 사용하여 예측 가능성을 높이세요.

이러한 인터페이스와 패턴을 깊이 이해하고 적절히 적용할 때, 개발자는 데이터 변경에 반응하는 동적이고 견고한 애플리케이션을 구축할 수 있습니다. 이는 단순한 기술적 숙련도를 넘어서, 사용자 경험을 고려한 소프트웨어 설계 철학의 실천입니다. 이러한 이해는 WPF에 국한되지 않고, 모든 데이터 주도 UI 개발에 적용할 수 있는 근본적인 원칙을 제공합니다.