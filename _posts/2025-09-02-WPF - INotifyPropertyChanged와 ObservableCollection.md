---
layout: post
title: WPF - INotifyPropertyChanged와 ObservableCollection
date: 2025-09-02 18:25:23 +0900
category: WPF
---
# INotifyPropertyChanged와 ObservableCollection

## 데이터 동기화의 핵심 메커니즘

WPF의 데이터 바인딩은 UI와 데이터를 자동으로 동기화해줍니다. 이 동기화의 핵심에는 **INotifyPropertyChanged**와 **ObservableCollection**이라는 두 가지 인터페이스가 있습니다. 각각은 개별 속성의 변경과 컬렉션의 구조적 변경을 UI에 알리는 역할을 합니다.

---

## INotifyPropertyChanged: 속성 변경 알림

### 인터페이스의 역할

INotifyPropertyChanged는 속성값이 바뀌었을 때 바인딩된 UI 요소에게 "값이 변경되었으니 다시 읽어오라"고 알려주는 역할을 합니다. 이 인터페이스는 단 하나의 이벤트만 정의합니다.

```csharp
public interface INotifyPropertyChanged
{
    event PropertyChangedEventHandler PropertyChanged;
}
```

### 기본 구현 방법

ViewModel에서 이 인터페이스를 구현하는 가장 기본적인 방법입니다.

```csharp
public class PersonViewModel : INotifyPropertyChanged
{
    private string _name;
    public string Name
    {
        get => _name;
        set
        {
            if (_name != value)
            {
                _name = value;
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

`[CallerMemberName]`을 사용하면 속성 이름을 문자열로 직접 전달하지 않아도 됩니다. 위 예제에서 `OnPropertyChanged()`만 호출하면 자동으로 "Name"이 전달됩니다.

### 계산된 속성 처리

다른 속성에 의존하는 계산된 속성은 의존하는 속성이 변경될 때 함께 알림을 보내야 합니다.

```csharp
public class OrderViewModel : INotifyPropertyChanged
{
    private decimal _price;
    private int _quantity;
    
    public decimal Price
    {
        get => _price;
        set
        {
            if (_price != value)
            {
                _price = value;
                OnPropertyChanged();
                OnPropertyChanged(nameof(Total));
            }
        }
    }
    
    public int Quantity
    {
        get => _quantity;
        set
        {
            if (_quantity != value)
            {
                _quantity = value;
                OnPropertyChanged();
                OnPropertyChanged(nameof(Total));
            }
        }
    }
    
    public decimal Total => Price * Quantity;
    
    // PropertyChanged 이벤트와 OnPropertyChanged 메서드 생략
}
```

### 재사용 가능한 기본 클래스

매번 같은 코드를 반복하지 않기 위해 기본 클래스를 만들어두면 편리합니다.

```csharp
public abstract class ObservableObject : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected virtual void OnPropertyChanged([CallerMemberName] string propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
    
    protected bool SetProperty<T>(ref T field, T value, [CallerMemberName] string propertyName = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value))
            return false;
            
        field = value;
        OnPropertyChanged(propertyName);
        return true;
    }
}
```

이 기본 클래스를 사용하면 ViewModel 코드가 훨씬 간결해집니다.

```csharp
public class ProductViewModel : ObservableObject
{
    private string _productName;
    public string ProductName
    {
        get => _productName;
        set => SetProperty(ref _productName, value);
    }
    
    private decimal _price;
    public decimal Price
    {
        get => _price;
        set => SetProperty(ref _price, value);
    }
}
```

---

## ObservableCollection: 컬렉션 변경 알림

### 컬렉션 변경의 문제점

일반적인 `List<T>`를 사용하면 항목이 추가되거나 제거되어도 UI는 이를 알 수 없습니다. UI를 갱신하려면 매번 새로운 컬렉션을 할당해야 합니다.

```csharp
// 이렇게 하면 UI가 갱신되지 않음
Items.Add(newItem);  // UI에 반영 안 됨

// 이렇게 해야 UI가 갱신됨
Items = new List<Item>(Items) { newItem };  // 전체를 새로 할당
```

### ObservableCollection의 사용

`ObservableCollection<T>`는 `INotifyCollectionChanged`를 구현하여 항목 추가, 제거, 이동 시 자동으로 UI에 알립니다.

```csharp
public class MainViewModel : ObservableObject
{
    public ObservableCollection<string> Items { get; } = new ObservableCollection<string>();
    
    public MainViewModel()
    {
        // 초기 데이터
        Items.Add("첫 번째 항목");
        Items.Add("두 번째 항목");
        
        // 추가 명령
        AddCommand = new RelayCommand(() => Items.Add($"항목 {Items.Count + 1}"));
        
        // 삭제 명령
        RemoveCommand = new RelayCommand(() => Items.RemoveAt(0));
    }
    
    public ICommand AddCommand { get; }
    public ICommand RemoveCommand { get; }
}
```

### 컬렉션 변경 감지 활용

`CollectionChanged` 이벤트를 구독하면 컬렉션 변경 시 추가 작업을 수행할 수 있습니다.

```csharp
public class OrderViewModel : ObservableObject
{
    public ObservableCollection<OrderItem> Items { get; } = new ObservableCollection<OrderItem>();
    
    private decimal _total;
    public decimal Total
    {
        get => _total;
        private set => SetProperty(ref _total, value);
    }
    
    public OrderViewModel()
    {
        Items.CollectionChanged += (s, e) => CalculateTotal();
        
        // 항목 속성 변경도 감지하기 위해 각 항목의 PropertyChanged 구독
        Items.CollectionChanged += OnItemsChanged;
    }
    
    private void OnItemsChanged(object sender, NotifyCollectionChangedEventArgs e)
    {
        if (e.NewItems != null)
        {
            foreach (OrderItem item in e.NewItems)
            {
                item.PropertyChanged += OnItemPropertyChanged;
            }
        }
        
        if (e.OldItems != null)
        {
            foreach (OrderItem item in e.OldItems)
            {
                item.PropertyChanged -= OnItemPropertyChanged;
            }
        }
        
        CalculateTotal();
    }
    
    private void OnItemPropertyChanged(object sender, PropertyChangedEventArgs e)
    {
        if (e.PropertyName == nameof(OrderItem.Price) || 
            e.PropertyName == nameof(OrderItem.Quantity))
        {
            CalculateTotal();
        }
    }
    
    private void CalculateTotal()
    {
        Total = Items.Sum(item => item.Price * item.Quantity);
    }
}
```

### 대량 업데이트 최적화

많은 항목을 한 번에 추가할 때는 각 추가마다 UI 갱신이 발생하여 성능이 저하될 수 있습니다. `DeferRefresh`를 사용하면 여러 변경을 모아서 한 번에 처리할 수 있습니다.

```csharp
public void LoadLargeData(List<Item> newItems)
{
    using (Items.DeferRefresh())  // 블록이 끝날 때까지 갱신 보류
    {
        Items.Clear();
        foreach (var item in newItems)
        {
            Items.Add(item);
        }
    }  // 여기서 한 번에 갱신됨
}
```

---

## 두 메커니즘의 통합

### 항목 속성 변경까지 감지하는 컬렉션

컬렉션 내부 항목의 속성 변경까지 UI에 반영하려면 ViewModel이 항목의 변경도 구독해야 합니다.

```csharp
public class TodoListViewModel : ObservableObject
{
    public ObservableCollection<TodoItem> Items { get; } = new ObservableCollection<TodoItem>();
    
    private int _completedCount;
    public int CompletedCount
    {
        get => _completedCount;
        private set => SetProperty(ref _completedCount, value);
    }
    
    public TodoListViewModel()
    {
        Items.CollectionChanged += OnItemsChanged;
    }
    
    private void OnItemsChanged(object sender, NotifyCollectionChangedEventArgs e)
    {
        if (e.NewItems != null)
        {
            foreach (TodoItem item in e.NewItems)
            {
                item.PropertyChanged += OnTodoItemChanged;
            }
        }
        
        if (e.OldItems != null)
        {
            foreach (TodoItem item in e.OldItems)
            {
                item.PropertyChanged -= OnTodoItemChanged;
            }
        }
        
        UpdateCompletedCount();
    }
    
    private void OnTodoItemChanged(object sender, PropertyChangedEventArgs e)
    {
        if (e.PropertyName == nameof(TodoItem.IsCompleted))
        {
            UpdateCompletedCount();
        }
    }
    
    private void UpdateCompletedCount()
    {
        CompletedCount = Items.Count(item => item.IsCompleted);
    }
}
```

### 간단한 구조 다이어그램

```
[View] <-- 바인딩 --> [ViewModel]
                         |
                         v
                INotifyPropertyChanged
                (속성 변경 알림)
                         |
                         v
                ObservableCollection<T>
                (컬렉션 변경 알림)
                         |
                         v
                [Model] (개별 항목도 INotifyPropertyChanged 구현)
```

---

## 성능과 메모리 고려사항

### 불필요한 알림 방지

값이 실제로 변경되지 않았는데도 알림을 보내는 것은 불필요한 UI 작업을 유발합니다. SetProperty 패턴에서는 값이 같으면 알림을 보내지 않습니다.

```csharp
// 좋은 예: 값이 실제로 변경될 때만 알림
public string Name
{
    get => _name;
    set => SetProperty(ref _name, value);
}

// 나쁜 예: 항상 알림 발생
public string Name
{
    get => _name;
    set
    {
        _name = value;
        OnPropertyChanged();  // 같은 값이어도 알림 발생
    }
}
```

### 이벤트 구독 해제

컬렉션 항목의 PropertyChanged 이벤트를 구독할 때는 항목이 제거될 때 반드시 구독을 해제해야 메모리 누수를 방지할 수 있습니다.

```csharp
private void OnItemsChanged(object sender, NotifyCollectionChangedEventArgs e)
{
    // 이전 코드 참조: 반드시 OldItems에서 구독 해제
}
```

---

## 요약

| 인터페이스 | 목적 | 주요 사용처 |
|-----------|------|------------|
| INotifyPropertyChanged | 개별 속성 변경 알림 | ViewModel의 모든 속성 |
| ObservableCollection\<T\> | 컬렉션 구조 변경 알림 | 리스트, 그리드 등 목록 데이터 |

이 두 메커니즘을 올바르게 사용하면 UI와 데이터가 항상 동기화 상태를 유지합니다. 개발자는 UI 갱신 코드를 직접 작성할 필요 없이 데이터 변경에만 집중할 수 있습니다.

초보자라면 먼저 `ObservableObject` 기본 클래스를 만들어두고, 컬렉션은 `ObservableCollection`을 기본으로 사용하는 습관을 들이는 것이 좋습니다. 점차 복잡한 시나리오(중첩 컬렉션, 실시간 데이터 등)에 맞춰 구독 관리와 성능 최적화를 적용해 나가면 됩니다.