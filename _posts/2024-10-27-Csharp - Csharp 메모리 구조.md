---
layout: post
title: C# - 열거형, 구조체, 튜플
date: 2024-10-27 19:20:23 +0900
category: Csharp
---
# C# 메모리 구조

C#의 메모리 관리는 .NET 런타임이 제공하는 자동 메모리 관리 시스템을 기반으로 합니다. 개발자는 명시적인 메모리 할당과 해제에서 비교적 자유롭지만, 효율적인 메모리 사용을 위해서는 내부 동작 원리를 이해하는 것이 중요합니다. 값 타입과 참조 타입의 차이, 가비지 컬렉션의 작동 방식, 스택과 힙의 구분 등은 C# 개발자가 반드시 숙지해야 할 핵심 개념들입니다.

---

## 스택과 힙

### 스택

스택은 함수 호출과 지역 변수를 관리하는 데 사용되는 메모리 영역입니다. 후입선출(LIFO) 방식으로 작동하며, 메모리 할당과 해제가 매우 빠르고 예측 가능합니다.

```csharp
void Calculate()
{
    int x = 10;              // 스택에 할당
    double y = 20.5;         // 스택에 할당
    Point p = new Point(3, 4); // 구조체: 값 자체가 스택에 저장
    // 함수 종료 시 x, y, p는 자동으로 해제됨
}
```

스택의 특징:
- **빠른 할당/해제**: 단순한 포인터 이동으로 처리
- **자동 관리**: 변수 범위를 벗어나면 자동 해제
- **크기 제한**: 일반적으로 힙보다 작은 크기(기본적으로 1-4MB)
- **연속 메모리**: 캐시 지역성이 우수

### 힙

힙은 런타임에 동적으로 생성되는 객체들을 저장하는 메모리 영역입니다. 참조 타입(클래스, 배열, 문자열 등)의 인스턴스가 힙에 저장됩니다.

```csharp
void CreateObjects()
{
    Person person = new Person("Alice", 30);  // Person 객체는 힙에, person 참조는 스택에
    int[] numbers = new int[100];             // 배열은 힙에 할당
    string text = "Hello, World!";            // 문자열은 힙에 할당
}
```

힙의 특징:
- **동적 크기**: 필요에 따라 확장 가능
- **가비지 컬렉션**: 사용되지 않는 객체 자동 회수
- **단편화 가능**: 빈번한 할당/해제로 인한 메모리 단편화 발생 가능
- **상대적 느림**: 할당과 해제가 스택보다 복잡

---

## 값 타입과 참조 타입

### 값 타입

값 타입은 데이터 자체를 저장합니다. 구조체(`struct`), 열거형(`enum`), 그리고 기본 숫자 타입(`int`, `double` 등)이 여기에 속합니다.

```csharp
public struct Point
{
    public int X;
    public int Y;
    
    public Point(int x, int y) { X = x; Y = y; }
}

void ValueTypeDemo()
{
    Point p1 = new Point(10, 20);
    Point p2 = p1;  // 값 복사: p2는 p1의 복사본
    
    p2.X = 100;     // p1에는 영향 없음
    
    Console.WriteLine(p1.X); // 10
    Console.WriteLine(p2.X); // 100
}
```

값 타입의 특징:
- 스택에 저장되거나 부모 객체 내에 인라인으로 저장됨
- 복사 시 전체 값이 복사됨
- 일반적으로 가비지 컬렉션의 대상이 아님
- 불변성(immutability) 설계 권장

### 참조 타입

참조 타입은 데이터에 대한 참조(주소)를 저장합니다. 클래스(`class`), 인터페이스(`interface`), 배열, 델리게이트(`delegate`)가 여기에 속합니다.

```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}

void ReferenceTypeDemo()
{
    Person person1 = new Person { Name = "Alice", Age = 30 };
    Person person2 = person1;  // 참조 복사: 둘 다 같은 객체를 가리킴
    
    person2.Name = "Bob";      // person1도 영향 받음
    
    Console.WriteLine(person1.Name); // Bob
    Console.WriteLine(person2.Name); // Bob
}
```

참조 타입의 특징:
- 객체는 힙에 저장되고, 변수는 그 참조를 저장
- 복사 시 참조만 복사됨
- 가비지 컬렉션의 대상
- 상속과 다형성 지원

### 실무에서의 선택 기준

- **작고, 자주 생성되고, 불변적인 데이터**: 값 타입이 적합합니다. 예를 들어 좌표, 통화 등이 있습니다.
- **복잡한 객체, 상속이 필요한 경우, 큰 데이터**: 참조 타입을 사용합니다.
- 일반적으로 16바이트 이하의 데이터는 값 타입을 고려할 수 있습니다.
- 빈번한 복사가 발생하는 경우 참조 타입이 더 효율적일 수 있습니다.

---

## 박싱과 언박싱

### 박싱

값 타입을 참조 타입으로 변환하는 과정입니다. 이 과정에서 힙에 새로운 객체가 생성됩니다.

```csharp
int number = 42;
object boxed = number;  // 박싱: 힙에 새로운 객체 생성
```

### 언박싱

박싱된 객체를 다시 값 타입으로 변환하는 과정입니다.

```csharp
object boxed = 42;
int unboxed = (int)boxed;  // 언박싱: 값 복사
```

### 성능 영향

박싱은 힙 할당을 유발하므로 빈번하게 발생하면 성능 저하로 이어집니다.

```csharp
// 좋지 않은 예: ArrayList는 object를 저장하므로 값 타입을 넣을 때마다 박싱
ArrayList list = new ArrayList();
for (int i = 0; i < 10000; i++) list.Add(i);  // 매번 박싱

// 개선: 제네릭 컬렉션 사용
List<int> list = new List<int>();
for (int i = 0; i < 10000; i++) list.Add(i);  // 박싱 없음
```

박싱을 피하는 전략:
- 제네릭 컬렉션(`List<T>`, `Dictionary<TKey, TValue>`) 사용
- 값 타입에 적합한 인터페이스 설계
- `Span<T>`나 `Memory<T>` 활용

---

## 가비지 컬렉션

### 세대별 가비지 컬렉션

.NET GC는 객체의 수명에 따라 세 개의 세대(Generation)로 구분하여 관리합니다.

```csharp
// Gen 0: 새로 생성된 객체
var shortLived = new byte[1024];

// 살아남은 객체는 상위 세대로 승격
// Gen 0 → Gen 1 → Gen 2
```

세대별 특징:
- **Gen 0**: 가장 젊은 객체들, 빈번하게 수집됨
- **Gen 1**: Gen 0에서 살아남은 객체들, 중간 빈도로 수집됨
- **Gen 2**: 가장 오래된 객체들, 드물게 수집됨

### LOH(Large Object Heap)

85,000바이트 이상의 큰 객체는 LOH에 할당됩니다. LOH는 압축(Compaction)이 제한적이므로 단편화 가능성이 있습니다.

```csharp
byte[] largeBuffer = new byte[100_000];  // LOH에 할당
byte[] smallBuffer = new byte[10_000];   // 일반 힙에 할당
```

### 최적의 메모리 사용을 위한 가이드라인

- 객체 수명을 최소화하세요.
- 대형 객체는 재사용하세요 (`ArrayPool<T>` 활용).
- 세분화된 할당보다 큰 덩어리로 할당하세요.
- `Finalizer`를 최소화하고 `IDisposable` 패턴을 우선 사용하세요.

---

## 메모리 누수와 관리되지 않는 리소스

### IDisposable 패턴

관리되지 않는 리소스(파일, 네트워크 연결 등)는 명시적으로 해제해야 합니다.

```csharp
public class DatabaseConnection : IDisposable
{
    private SqlConnection _connection;
    private bool _disposed = false;
    
    public DatabaseConnection(string connStr)
    {
        _connection = new SqlConnection(connStr);
        _connection.Open();
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            _connection?.Close();
            _connection?.Dispose();
            _disposed = true;
        }
        GC.SuppressFinalize(this);
    }
}

// 사용 예
using (var db = new DatabaseConnection(connStr))
{
    // 작업 수행
} // 자동으로 Dispose 호출
```

### 일반적인 메모리 누수 원인

- **이벤트 핸들러 미해제**: 구독 해제하지 않으면 객체가 GC되지 않음
- **정적 컬렉션 누적**: 정적 컬렉션에 객체를 계속 추가하면 영구 참조
- **타이머 참조 유지**: 타이머가 콜백을 통해 객체를 참조

---

## 고성능 메모리 관리 기법

### ArrayPool을 이용한 버퍼 재사용

```csharp
public void ProcessData(Stream stream)
{
    var pool = ArrayPool<byte>.Shared;
    byte[] buffer = pool.Rent(8192);  // 풀에서 버퍼 대여
    try
    {
        int bytesRead;
        while ((bytesRead = stream.Read(buffer, 0, buffer.Length)) > 0)
        {
            ProcessBuffer(buffer, bytesRead);
        }
    }
    finally
    {
        pool.Return(buffer);  // 버퍼 반환
    }
}
```

### Span<T>와 Memory<T>

스택 할당이나 배열의 일부를 힙 할당 없이 다룰 수 있습니다.

```csharp
public int SumSpan(ReadOnlySpan<int> numbers)
{
    int sum = 0;
    for (int i = 0; i < numbers.Length; i++) sum += numbers[i];
    return sum;
}

int[] array = new int[1000];
int total = SumSpan(array);                    // 전체
int part = SumSpan(array.AsSpan(100, 200));    // 일부
```

### 스택 할당 배열 (stackalloc)

스택에 배열을 할당하여 힙 할당을 피할 수 있습니다. 큰 배열에는 적합하지 않습니다.

```csharp
Span<byte> buffer = stackalloc byte[256];
for (int i = 0; i < buffer.Length; i++) buffer[i] = (byte)i;
```

---

## 메모리 진단과 프로파일링

### 기본적인 메모리 사용량 확인

```csharp
var process = Process.GetCurrentProcess();
Console.WriteLine($"Working Set: {process.WorkingSet64:N0} bytes");
Console.WriteLine($"Total Memory: {GC.GetTotalMemory(false):N0} bytes");
for (int i = 0; i <= 2; i++)
    Console.WriteLine($"Gen {i} Collections: {GC.CollectionCount(i)}");
```

### dotnet 진단 도구

- `dotnet-counters`로 메모리 카운터 모니터링
- `dotnet-gcdump`으로 힙 덤프 생성
- `dotnet-trace`로 프로파일링

---

## 실전 메모리 최적화 패턴

### 객체 풀링

자주 생성되고 소멸되는 객체를 풀에 보관하여 재사용합니다.

```csharp
public class ObjectPool<T> where T : class, new()
{
    private readonly ConcurrentBag<T> _objects = new();
    
    public T Get() => _objects.TryTake(out T item) ? item : new T();
    public void Return(T item) => _objects.Add(item);
}

// StringBuilder 풀 예
var pool = new ObjectPool<StringBuilder>();
var sb = pool.Get();
try { sb.Append("..."); }
finally { sb.Clear(); pool.Return(sb); }
```

### 불변성 활용

객체가 생성 후 변경되지 않도록 하면 스레드 안전성과 예측 가능성이 높아집니다.

```csharp
public readonly struct ImmutablePoint
{
    public int X { get; }
    public int Y { get; }
    public ImmutablePoint(int x, int y) { X = x; Y = y; }
    public ImmutablePoint WithX(int newX) => new ImmutablePoint(newX, Y);
}
```

### 메모리 효율적인 데이터 구조

인덱스가 연속적인 경우 배열을 사용하여 Dictionary보다 메모리 효율을 높일 수 있습니다.

```csharp
private readonly Item[] _items;
public Item GetItem(int id) => id >= 0 && id < _items.Length ? _items[id] : null;
```

---

## 결론

C# 메모리 관리는 개발자의 생산성과 애플리케이션 성능 사이의 균형을 잘 잡아줍니다. 고성능 애플리케이션을 개발하기 위해서는 내부 동작 원리를 이해하고 적절한 패턴을 적용하는 것이 중요합니다.

**핵심 원칙 요약:**

- **의도적인 타입 선택**: 작고 불변한 데이터는 값 타입이, 복잡하고 상속이 필요한 객체는 참조 타입이 적합합니다.
- **메모리 할당 최소화**: 특히 핫 패스에서는 불필요한 할당을 피하고 객체 풀링, 배열 재사용, 스택 할당을 고려하세요.
- **박싱 회피**: 제네릭 컬렉션과 메서드를 사용하여 불필요한 박싱을 피하세요.
- **리소스 관리**: 관리되지 않는 리소스는 `IDisposable`과 `using` 문으로 명시적으로 해제하세요.
- **메모리 누수 예방**: 이벤트 핸들러, 정적 컬렉션, 타이머 등의 순환 참조에 주의하세요.
- **도구 활용**: 메모리 프로파일링 도구를 사용하여 실제 사용 패턴을 분석하고 최적화하세요.
- **최신 기능 활용**: `Span<T>`, `Memory<T>`, `ArrayPool<T>` 등 최신 .NET 기능을 적극 활용하세요.

메모리 최적화는 종합적인 접근이 필요합니다. 성능이 중요한 부분에서는 할당량을 측정하고, 다양한 접근법을 실험하며, 실제 프로파일링 데이터를 기반으로 결정을 내리세요. C#의 강력한 메모리 관리 시스템은 잘 이해하고 사용할 때 최대의 효과를 발휘합니다.