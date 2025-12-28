---
layout: post
title: C# - 열거형, 구조체, 튜플
date: 2024-10-27 19:20:23 +0900
category: Csharp
---
# C# 메모리 구조

## C# 메모리 관리의 전체적인 이해

C#의 메모리 관리는 .NET 런타임이 제공하는 강력한 자동 메모리 관리 시스템을 기반으로 합니다. 개발자는 명시적인 메모리 할당과 해제에서 비교적 자유롭지만, 효율적인 메모리 사용을 위해서는 내부 동작 원리를 이해하는 것이 필수적입니다. 값 타입과 참조 타입의 차이, 가비지 컬렉션(GC)의 작동 방식, 스택과 힙의 구분 등은 C# 개발자가 반드시 숙지해야 할 핵심 개념들입니다.

---

## 스택(Stack)과 힙(Heap): 메모리 관리의 두 축

### 스택: 빠르고 체계적인 메모리 관리

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

### 힙: 동적이고 유연한 메모리 관리

힙은 런타임에 동적으로 생성되는 객체들을 저장하는 메모리 영역입니다. 참조 타입(클래스, 배열, 문자열 등)의 인스턴스가 힙에 저장됩니다.

```csharp
void CreateObjects()
{
    // 참조 타입은 힙에 할당됨
    Person person = new Person("Alice", 30);  // Person 객체는 힙에, person 참조는 스택에
    int[] numbers = new int[100];             // 배열은 힙에 할당
    string text = "Hello, World!";           // 문자열은 힙에 할당
}
```

힙의 특징:
- **동적 크기**: 필요에 따라 확장 가능
- **가비지 컬렉션**: 사용되지 않는 객체 자동 회수
- **단편화 가능**: 빈번한 할당/해제로 인한 메모리 단편화 발생 가능
- **상대적 느림**: 할당과 해제가 스택보다 복잡

---

## 값 타입 vs 참조 타입: 근본적인 차이

### 값 타입(Value Types)

값 타입은 데이터 자체를 저장합니다. 구조체(struct), 열거형(enum), 그리고 기본 숫자 타입(int, double 등)이 여기에 속합니다.

```csharp
public struct Point
{
    public int X;
    public int Y;
    
    public Point(int x, int y)
    {
        X = x;
        Y = y;
    }
}

void ValueTypeDemo()
{
    Point p1 = new Point(10, 20);
    Point p2 = p1;  // 값 복사: p2는 p1의 복사본
    
    p2.X = 100;     // p1에는 영향 없음
    
    Console.WriteLine($"p1.X: {p1.X}"); // 10
    Console.WriteLine($"p2.X: {p2.X}"); // 100
}
```

값 타입의 특징:
- 스택에 저장되거나 부모 객체 내에 인라인으로 저장됨
- 복사 시 전체 값이 복사됨
- 일반적으로 가비지 컬렉션의 대상이 아님
- 불변성(immutability) 설계 권장

### 참조 타입(Reference Types)

참조 타입은 데이터에 대한 참조(주소)를 저장합니다. 클래스(class), 인터페이스(interface), 배열, 델리게이트(delegate)가 여기에 속합니다.

```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
    
    public Person(string name, int age)
    {
        Name = name;
        Age = age;
    }
}

void ReferenceTypeDemo()
{
    Person person1 = new Person("Alice", 30);
    Person person2 = person1;  // 참조 복사: 둘 다 같은 객체를 가리킴
    
    person2.Name = "Bob";      // person1도 영향 받음
    
    Console.WriteLine($"person1.Name: {person1.Name}"); // Bob
    Console.WriteLine($"person2.Name: {person2.Name}"); // Bob
}
```

참조 타입의 특징:
- 객체는 힙에 저장되고, 변수는 그 참조를 저장
- 복사 시 참조만 복사됨
- 가비지 컬렉션의 대상
- 상속과 다형성 지원

### 실무에서의 선택 기준

```csharp
// 값 타입이 적합한 경우: 작고, 자주 생성되고, 불변적인 데이터
public readonly struct Money
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    public Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }
}

// 참조 타입이 적합한 경우: 복잡한 객체, 상속이 필요하거나, 큰 데이터
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public List<Order> Orders { get; set; }
    // ...
}
```

일반적인 가이드라인:
- 16바이트 이하의 작은 데이터: 값 타입 고려
- 불변성이 중요한 데이터: 값 타입 또는 불변 참조 타입
- 상속이나 다형성이 필요한 경우: 참조 타입
- 빈번한 복사가 발생하는 경우: 참조 타입이 더 효율적일 수 있음

---

## 박싱과 언박싱: 성능에 영향을 미치는 숨은 비용

### 박싱(Boxing)

값 타입을 참조 타입으로 변환하는 과정입니다. 이 과정에서 힙에 새로운 객체가 생성됩니다.

```csharp
int number = 42;
object boxed = number;  // 박싱: 힙에 새로운 객체 생성
```

### 언박싱(Unboxing)

박싱된 객체를 다시 값 타입으로 변환하는 과정입니다.

```csharp
object boxed = 42;
int unboxed = (int)boxed;  // 언박싱: 값 복사
```

### 박싱의 성능 영향

```csharp
// 성능에 좋지 않은 예: 빈번한 박싱
public void ProcessNumbers(ArrayList list)  // ArrayList는 object 타입 저장
{
    int sum = 0;
    for (int i = 0; i < 100000; i++)
    {
        list.Add(i);  // 매번 박싱 발생!
    }
}

// 개선된 예: 제네릭 컬렉션 사용
public void ProcessNumbers(List<int> list)  // List<int>는 제네릭
{
    int sum = 0;
    for (int i = 0; i < 100000; i++)
    {
        list.Add(i);  // 박싱 없음
    }
}
```

박싱을 피하기 위한 전략:
1. 제네릭 컬렉션(`List<T>`, `Dictionary<TKey, TValue>` 등) 사용
2. 값 타입에 적합한 인터페이스 설계
3. `Span<T>`나 `Memory<T>` 활용

---

## 가비지 컬렉션(GC): .NET의 자동 메모리 관리자

### 세대별 가비지 컬렉션

.NET GC는 객체의 수명에 따라 세 개의 세대(Generation)로 구분하여 관리합니다:

```csharp
public class GarbageCollectionDemo
{
    public void DemonstrateGenerations()
    {
        // Gen 0: 새로 생성된 객체
        var shortLived = new byte[1024];
        
        // GC.Collect() 호출하지 말 것! (예제용 설명만)
        // 실제로는 런타임이 자동으로 관리
        
        // 살아남은 객체는 상위 세대로 승격
        // Gen 0 → Gen 1 → Gen 2
    }
}
```

**세대별 특징**:
- **Gen 0**: 가장 젊은 객체들, 빈번하게 수집됨
- **Gen 1**: Gen 0에서 살아남은 객체들, 중간 빈도로 수집됨
- **Gen 2**: 가장 오래된 객체들, 드물게 수집됨

### LOH(Large Object Heap): 대형 객체의 특별 관리

85,000바이트 이상의 큰 객체는 LOH에 할당됩니다:

```csharp
// LOH에 할당되는 예
byte[] largeBuffer = new byte[100_000];  // LOH에 할당

// LOH에 할당되지 않는 예
byte[] smallBuffer = new byte[10_000];   // 일반 힙에 할당
```

LOH의 특징:
- 압축(Compaction)이 제한적 → 단편화 가능성
- 수집 빈도가 낮음
- 할당/해제 비용이 큼

### 최적의 메모리 사용을 위한 가이드라인

1. **객체 수명 최소화**: 가능한 한 빨리 참조 해제
2. **대형 객체 재사용**: `ArrayPool<T>` 활용
3. **세분화된 할당 피하기**: 큰 덩어리로 할당
4. **Finalizer 최소화**: `IDisposable` 패턴 우선 사용

---

## 메모리 누수와 관리되지 않는 리소스

### IDisposable 패턴과 using 문

관리되지 않는 리소스(파일, 네트워크 연결, 데이터베이스 연결 등)는 명시적으로 해제해야 합니다:

```csharp
public class DatabaseConnection : IDisposable
{
    private SqlConnection _connection;
    private bool _disposed = false;
    
    public DatabaseConnection(string connectionString)
    {
        _connection = new SqlConnection(connectionString);
        _connection.Open();
    }
    
    public void ExecuteQuery(string query)
    {
        if (_disposed)
            throw new ObjectDisposedException(nameof(DatabaseConnection));
        
        // 쿼리 실행
    }
    
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                // 관리 리소스 정리
                _connection?.Close();
                _connection?.Dispose();
            }
            
            // 비관리 리소스 정리
            _disposed = true;
        }
    }
    
    ~DatabaseConnection()
    {
        Dispose(false);
    }
}

// 사용 예
using (var db = new DatabaseConnection(connectionString))
{
    db.ExecuteQuery("SELECT * FROM Users");
} // 자동으로 Dispose 호출
```

### 메모리 누수의 일반적인 원인

1. **이벤트 핸들러 미해제**:
```csharp
public class Publisher
{
    public event EventHandler? SomethingHappened;
}

public class Subscriber
{
    public Subscriber(Publisher publisher)
    {
        publisher.SomethingHappened += OnSomethingHappened;
    }
    
    private void OnSomethingHappened(object? sender, EventArgs e)
    {
        // ...
    }
    
    // 만약 구독 해제하지 않으면 Subscriber가 GC되지 않음
}
```

2. **정적 컬렉션 누적**:
```csharp
public static class Cache
{
    private static Dictionary<string, object> _cache = new();
    
    public static void Add(string key, object value)
    {
        _cache[key] = value;  // 영구적으로 참조 유지
    }
    
    // 해결책: 약한 참조(WeakReference) 사용 또는 만료 정책 추가
}
```

3. **타이머 참조 유지**:
```csharp
public class Service
{
    private Timer _timer;
    
    public Service()
    {
        _timer = new Timer(Callback, null, 1000, 1000);
    }
    
    private void Callback(object? state)
    {
        // 타이머가 콜백을 통해 Service를 참조함
    }
}
```

---

## 고성능 메모리 관리 기법

### ArrayPool을 이용한 버퍼 재사용

```csharp
public class BufferProcessor
{
    public void ProcessData(Stream stream)
    {
        var pool = ArrayPool<byte>.Shared;
        byte[] buffer = pool.Rent(8192);  // 풀에서 버퍼 대여
        
        try
        {
            int bytesRead;
            while ((bytesRead = stream.Read(buffer, 0, buffer.Length)) > 0)
            {
                // 버퍼 처리
                ProcessBuffer(buffer, bytesRead);
            }
        }
        finally
        {
            pool.Return(buffer, clearArray: true);  // 버퍼 반환
        }
    }
    
    private void ProcessBuffer(byte[] buffer, int length)
    {
        // 처리 로직
    }
}
```

### Span<T>와 Memory<T>를 이용한 메모리 효율성

```csharp
public class SpanExample
{
    public int SumArray(ReadOnlySpan<int> numbers)
    {
        int sum = 0;
        for (int i = 0; i < numbers.Length; i++)
        {
            sum += numbers[i];
        }
        return sum;
    }
    
    public void ProcessData()
    {
        int[] array = new int[1000];
        // 배열 채우기...
        
        // 전체 배열 슬라이스
        var slice = array.AsSpan();
        int total = SumArray(slice);
        
        // 부분 슬라이스
        var middlePart = array.AsSpan(100, 200);
        int middleSum = SumArray(middlePart);
    }
}
```

### 스택 할당 배열(stackalloc)

스택에 배열을 할당하여 힙 할당을 피할 수 있습니다:

```csharp
public unsafe void StackAllocExample()
{
    // 스택에 256바이트 배열 할당
    Span<byte> buffer = stackalloc byte[256];
    
    // 사용
    for (int i = 0; i < buffer.Length; i++)
    {
        buffer[i] = (byte)i;
    }
}
```

주의사항: `stackalloc`은 스택 메모리를 사용하므로 큰 배열에는 적합하지 않습니다.

---

## 메모리 진단과 프로파일링

### 기본적인 메모리 사용량 확인

```csharp
public class MemoryDiagnostics
{
    public void ShowMemoryInfo()
    {
        // 현재 프로세스의 메모리 사용량
        var process = Process.GetCurrentProcess();
        Console.WriteLine($"Working Set: {process.WorkingSet64:N0} bytes");
        Console.WriteLine($"Private Memory: {process.PrivateMemorySize64:N0} bytes");
        
        // GC 정보
        Console.WriteLine($"Total Memory: {GC.GetTotalMemory(false):N0} bytes");
        
        for (int i = 0; i <= 2; i++)
        {
            Console.WriteLine($"Gen {i} Collections: {GC.CollectionCount(i)}");
        }
    }
    
    public void MeasureAllocation(Action action)
    {
        long before = GC.GetAllocatedBytesForCurrentThread();
        action();
        long after = GC.GetAllocatedBytesForCurrentThread();
        
        Console.WriteLine($"Allocated: {after - before:N0} bytes");
    }
}
```

### dotnet 진단 도구 활용

```bash
# 메모리 카운터 모니터링
dotnet-counters monitor --process-id [PID] System.Runtime

# 힙 덤프 생성
dotnet-gcdump collect --process-id [PID]

# 메모리 프로파일링
dotnet-trace collect --process-id [PID] --providers Microsoft-DotNETRuntime-SampleProfiler
```

---

## 실전 메모리 최적화 패턴

### 객체 풀링 패턴

```csharp
public class ObjectPool<T> where T : class, new()
{
    private readonly ConcurrentBag<T> _objects;
    private readonly Func<T> _objectGenerator;
    
    public ObjectPool(Func<T> objectGenerator)
    {
        _objects = new ConcurrentBag<T>();
        _objectGenerator = objectGenerator ?? new Func<T>(() => new T());
    }
    
    public T Get()
    {
        if (_objects.TryTake(out T item))
            return item;
        
        return _objectGenerator();
    }
    
    public void Return(T item)
    {
        _objects.Add(item);
    }
}

// 사용 예
var pool = new ObjectPool<StringBuilder>(() => new StringBuilder());
var sb = pool.Get();
try
{
    sb.Append("Hello");
    sb.Append(" World");
    string result = sb.ToString();
}
finally
{
    sb.Clear();
    pool.Return(sb);
}
```

### 불변성(Immutability)을 통한 메모리 안전성

```csharp
public readonly struct ImmutablePoint
{
    public readonly int X;
    public readonly int Y;
    
    public ImmutablePoint(int x, int y)
    {
        X = x;
        Y = y;
    }
    
    // 변경 메서드는 새 인스턴스 반환
    public ImmutablePoint WithX(int newX) => new ImmutablePoint(newX, Y);
    public ImmutablePoint WithY(int newY) => new ImmutablePoint(X, newY);
}
```

### 메모리 효율적인 데이터 구조

```csharp
public class MemoryEfficientLookup
{
    // Dictionary 대신 배열 사용 (인덱스가 연속적일 때)
    private readonly Item[] _items;
    
    public MemoryEfficientLookup(int capacity)
    {
        _items = new Item[capacity];
    }
    
    public Item GetItem(int id)
    {
        if (id >= 0 && id < _items.Length)
            return _items[id];
        
        return null;
    }
}

// 압축된 데이터 저장
public class CompressedData
{
    private readonly byte[] _data;
    
    public CompressedData(byte[] originalData)
    {
        using var compressedStream = new MemoryStream();
        using var gzipStream = new GZipStream(compressedStream, CompressionMode.Compress);
        gzipStream.Write(originalData, 0, originalData.Length);
        gzipStream.Flush();
        _data = compressedStream.ToArray();
    }
    
    public byte[] Decompress()
    {
        using var compressedStream = new MemoryStream(_data);
        using var gzipStream = new GZipStream(compressedStream, CompressionMode.Decompress);
        using var resultStream = new MemoryStream();
        gzipStream.CopyTo(resultStream);
        return resultStream.ToArray();
    }
}
```

---

## 결론

C# 메모리 관리는 개발자의 생산성과 애플리케이션 성능 사이의 균형을 잘 잡아줍니다. 자동 가비지 컬렉션은 메모리 관리의 부담을 줄여주지만, 고성능 애플리케이션을 개발하기 위해서는 내부 동작 원리를 이해하고 적절한 패턴을 적용하는 것이 중요합니다.

### 핵심 원칙 요약:

1. **의도적인 타입 선택**: 값 타입과 참조 타입의 차이를 이해하고 상황에 맞게 선택하세요. 작고 불변한 데이터는 값 타입이, 복잡하고 상속이 필요한 객체는 참조 타입이 적합합니다.

2. **메모리 할당 최소화**: 특히 핫 패스(hot path)에서는 불필요한 할당을 피하세요. 객체 풀링, 배열 재사용, 스택 할당 등을 고려하세요.

3. **박싱 회피**: 제네릭 컬렉션과 메서드를 사용하여 불필요한 박싱을 피하세요. 값 타입을 object로 다루는 것은 성능에 큰 영향을 미칩니다.

4. **리소스 관리**: 관리되지 않는 리소스는 반드시 명시적으로 해제하세요. `IDisposable` 패턴과 `using` 문을 일관되게 사용하세요.

5. **메모리 누수 예방**: 이벤트 핸들러, 정적 컬렉션, 타이머 등의 순환 참조로 인한 메모리 누수를 주의하세요.

6. **도구 활용**: 메모리 프로파일링 도구를 사용하여 실제 메모리 사용 패턴을 분석하고 최적화하세요.

7. **최신 기능 활용**: `Span<T>`, `Memory<T>`, `ArrayPool<T>` 등 최신 .NET의 메모리 최적화 기능을 적극 활용하세요.

8. **세대별 GC 이해**: 객체의 수명에 따른 GC 동작을 이해하고, 장수 객체와 단수 객체를 구분하여 관리하세요.

메모리 최적화는 종합적인 접근이 필요합니다. 성능이 중요한 부분에서는 할당량을 측정하고, 다양한 접근법을 실험하며, 실제 프로파일링 데이터를 기반으로 결정을 내리세요. C#의 강력한 메모리 관리 시스템은 잘 이해하고 사용할 때 최대의 효과를 발휘합니다. 이러한 이해는 단순한 성능 최적화를 넘어서 더 견고하고 예측 가능한 소프트웨어를 구축하는 데 기여합니다.