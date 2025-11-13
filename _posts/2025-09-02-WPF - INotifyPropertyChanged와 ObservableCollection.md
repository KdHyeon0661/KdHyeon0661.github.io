---
layout: post
title: WPF - INotifyPropertyChanged와 ObservableCollection
date: 2025-09-02 18:25:23 +0900
category: WPF
---
# WPF 데이터 변경 알림: `INotifyPropertyChanged`와 `ObservableCollection`

WPF의 데이터 바인딩은 **데이터 소스(ViewModel)**의 값이 변경될 때 자동으로 UI에 반영되도록 설계되어 있습니다.
이를 가능하게 해주는 핵심 인터페이스가 바로 **`INotifyPropertyChanged`**와 **`ObservableCollection<T>`**입니다.

---

## `INotifyPropertyChanged`란?

### 개념
- .NET에서 **객체의 속성(Property)이 변경되었음을 알리는 인터페이스**.
- ViewModel에서 UI에 데이터 변경 사실을 알릴 때 사용합니다.
- UI는 이 알림을 받고 바인딩된 값을 자동으로 갱신합니다.

### 정의
```csharp
public interface INotifyPropertyChanged
{
    event PropertyChangedEventHandler PropertyChanged;
}
```

### ⚡ 구현 방법
- `PropertyChanged` 이벤트를 호출해 UI에 알립니다.
- 보통 ViewModel의 속성 Setter에서 `OnPropertyChanged()`를 호출합니다.

```csharp
using System.ComponentModel;

public class PersonViewModel : INotifyPropertyChanged
{
    private string name;
    public string Name
    {
        get => name;
        set
        {
            if (name != value)
            {
                name = value;
                OnPropertyChanged(nameof(Name)); // 변경 알림 발생
            }
        }
    }

    public event PropertyChangedEventHandler PropertyChanged;

    protected void OnPropertyChanged(string propertyName)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

### 동작 원리
1. `Name` 속성이 변경됨
2. `OnPropertyChanged("Name")` 실행 → `PropertyChanged` 이벤트 발생
3. WPF 바인딩 엔진이 이벤트를 감지
4. 바인딩된 UI(`TextBox`, `TextBlock` 등)가 자동 갱신

---

## `ObservableCollection<T>`란?

### 개념
- **컬렉션(리스트) 데이터 변경을 UI에 알릴 수 있는 컬렉션 클래스**.
- `Add`, `Remove`, `Clear` 등 요소의 추가/삭제를 감지해 UI에 반영합니다.
- 일반 `List<T>`는 데이터 추가/삭제를 UI에 알리지 못하기 때문에, WPF에서는 `ObservableCollection<T>`를 사용해야 합니다.

### 정의
```csharp
public class ObservableCollection<T> : Collection<T>, INotifyCollectionChanged, INotifyPropertyChanged
```
- `INotifyCollectionChanged` 구현 → 컬렉션의 구조적 변경 알림 (`CollectionChanged` 이벤트).
- `INotifyPropertyChanged` 구현 → `Count` 등 속성 변경 알림.

### 예제
```csharp
using System.Collections.ObjectModel;

public class PeopleViewModel
{
    public ObservableCollection<string> People { get; set; }

    public PeopleViewModel()
    {
        People = new ObservableCollection<string>
        {
            "Alice",
            "Bob",
            "Charlie"
        };
    }
}
```

```xml
<ListBox ItemsSource="{Binding People}" />
```

- UI의 `ListBox`는 `People` 컬렉션과 바인딩됩니다.
- `People.Add("David")` 실행 시 자동으로 `ListBox`에 새로운 아이템이 표시됩니다.
- `People.Remove("Alice")` 실행 시 `ListBox`에서 해당 항목이 제거됩니다.

---

## `INotifyPropertyChanged` vs `ObservableCollection<T>`

| 기능 | INotifyPropertyChanged | ObservableCollection<T> |
|------|-----------------------|--------------------------|
| 대상 | 단일 속성(Property) | 컬렉션(Collection) 전체 |
| 알림 | 속성 값 변경 시 | 요소 추가/삭제 시 |
| 이벤트 | PropertyChanged | CollectionChanged |
| 예시 | `Name` 속성 변경 | `People.Add("Tom")` |

---

## 두 가지를 함께 사용하는 예제

보통 `INotifyPropertyChanged`와 `ObservableCollection<T>`는 **동시에 사용**됩니다.

```csharp
public class MainViewModel : INotifyPropertyChanged
{
    private string title;
    public string Title
    {
        get => title;
        set
        {
            if (title != value)
            {
                title = value;
                OnPropertyChanged(nameof(Title));
            }
        }
    }

    public ObservableCollection<string> Items { get; set; }

    public MainViewModel()
    {
        Items = new ObservableCollection<string> { "Item1", "Item2" };
    }

    public event PropertyChangedEventHandler PropertyChanged;
    protected void OnPropertyChanged(string propertyName) =>
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
}
```

```xml
<StackPanel>
    <TextBox Text="{Binding Title, Mode=TwoWay}" />
    <ListBox ItemsSource="{Binding Items}" />
    <Button Content="Add" Command="{Binding AddItemCommand}" />
</StackPanel>
```

- `Title` 속성은 `INotifyPropertyChanged`로 변경 사항을 UI에 알립니다.
- `Items`는 `ObservableCollection<T>`라서 항목 추가/삭제가 자동 반영됩니다.

---

# 결론
- **속성 변경 알림 → `INotifyPropertyChanged`**
- **컬렉션 변경 알림 → `ObservableCollection<T>`**
- MVVM 패턴에서 UI와 ViewModel을 동기화하려면 이 두 가지는 사실상 필수적입니다.
