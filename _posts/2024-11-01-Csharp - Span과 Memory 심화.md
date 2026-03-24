---
layout: post
title: C# - PInvoke, Marshal, NativeMemory
date: 2024-11-01 19:20:23 +0900
category: Csharp
---
# C# Span<T>와 Memory<T> 심화: 고성능 메모리 조작 기법

고성능 C# 애플리케이션을 작성하다 보면 불필요한 메모리 할당과 복사를 줄여야 하는 상황이 자주 발생합니다. `Span<T>`와 `Memory<T>`는 이러한 문제를 해결하기 위해 등장한 핵심 도구로, 안전성을 유지하면서도 C/C++ 수준의 메모리 제어 능력을 제공합니다.

---

## Span<T>의 본질과 제약

### Span<T>란 무엇인가

`Span<T>`는 연속된 메모리 영역을 나타내는 값 타입입니다. 배열, 문자열, 비관리 메모리, 스택 메모리 등 다양한 소스의 데이터를 복사 없이 직접 참조할 수 있게 해줍니다. `Span<T>`의 핵심은 **할당 없는 메모리 뷰**를 제공한다는 점입니다.

```csharp
int[] array = new int[] { 1, 2, 3, 4, 5 };
Span<int> spanFromArray = array;               // 배열 전체
Span<int> slice = spanFromArray.Slice(1, 3);   // {2, 3, 4} (복사 없음)

string text = "Hello";
ReadOnlySpan<char> spanFromString = text.AsSpan();  // 문자열을 Span으로

unsafe
{
    int* ptr = stackalloc int[10];
    Span<int> fromStack = new Span<int>(ptr, 10);    // 스택 메모리
}
```

### ref struct의 제약

`Span<T>`는 `ref struct`이므로 다음과 같은 제약이 있습니다.
- 힙에 저장할 수 없음 (클래스 필드, 배열 요소 등 불가)
- 박싱(Boxing) 불가
- 비동기 메서드(`async`)에서 사용 불가 (await 지점을 넘길 수 없음)
- 람다 표현식이나 로컬 함수에 캡처 불가

```csharp
class Example
{
    // 컴파일 오류: Span<int>는 클래스 필드가 될 수 없음
    // private Span<int> _field;
    
    async Task AsyncMethod()
    {
        int[] arr = new int[10];
        Span<int> span = arr;
        
        // 컴파일 오류: async 메서드에서 Span 사용 불가
        // await Task.Delay(1);
        // Console.WriteLine(span[0]);
    }
}
```

따라서 `Span<T>`는 동기 메서드 내에서만 사용해야 하며, 스택 프레임을 벗어나지 않는 수명을 가집니다.

---

## Memory<T>: 힙에 저장할 수 있는 Span

`Memory<T>`는 `Span<T>`와 유사하지만 `ref struct`가 아니므로 힙에 저장할 수 있습니다. 이는 비동기 코드, 클래스 필드, 제네릭 컬렉션 등에서 메모리 조각을 유지해야 할 때 사용합니다.

```csharp
class DataBuffer
{
    private Memory<byte> _buffer;  // 클래스 필드로 저장 가능
    
    public DataBuffer(int size)
    {
        _buffer = new byte[size];
    }
    
    public async Task ProcessAsync(Stream stream)
    {
        // 비동기 읽기: Memory<T>를 직접 전달
        int read = await stream.ReadAsync(_buffer);
        
        // 실제 작업은 Span<T>로 변환
        Span<byte> span = _buffer.Span.Slice(0, read);
        for (int i = 0; i < span.Length; i++)
        {
            span[i] = (byte)(span[i] ^ 0xFF);
        }
    }
}
```

`Memory<T>`에서 `Span<T>`를 얻을 때는 `Span` 속성을 사용합니다. 이 변환은 즉시 이루어지며 비용이 거의 없습니다.

---

## stackalloc: 스택 메모리 할당

작은 임시 버퍼가 필요할 때 `stackalloc`을 사용하면 힙 할당 없이 스택에 메모리를 확보할 수 있습니다. 반드시 `unsafe` 컨텍스트에서 사용하거나 `Span<T>`와 함께 사용해야 합니다.

```csharp
Span<int> numbers = stackalloc int[100];  // 스택에 100개 정수 할당
for (int i = 0; i < numbers.Length; i++)
    numbers[i] = i * i;
    
// 혼합 방식: 작은 크기는 stackalloc, 큰 크기는 ArrayPool
const int threshold = 1024;
int length = GetLength();
Span<byte> buffer = length <= threshold
    ? stackalloc byte[length]
    : ArrayPool<byte>.Shared.Rent(length);
```

`stackalloc`으로 할당된 메모리는 메서드가 반환되면 자동 해제되며, 너무 큰 크기를 요청하면 `StackOverflowException`이 발생할 수 있습니다.

---

## ArrayPool<T>: 배열 재사용

`ArrayPool<T>`는 자주 사용되는 배열을 풀링하여 GC 부하를 줄여줍니다. 특히 대용량 버퍼를 반복적으로 생성/해제하는 패턴에서 효과적입니다.

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(8192);  // 풀에서 가져옴
try
{
    // 버퍼 사용
    int bytesRead = stream.Read(buffer, 0, 8192);
    Process(buffer.AsSpan(0, bytesRead));
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);  // 반드시 반환
}
```

`Rent`는 요청한 크기 이상의 배열을 반환할 수 있으므로 실제 사용 길이를 항상 추적해야 합니다. `Return` 시 `clearArray: true`를 지정하면 배열을 초기화한 후 반환하여 데이터 잔존 문제를 방지할 수 있습니다.

---

## MemoryPool<T>와 IMemoryOwner<T>

`MemoryPool<T>`는 `ArrayPool<T>`와 유사하지만 `IMemoryOwner<T>`를 반환합니다. `IMemoryOwner<T>`는 `IDisposable`을 구현하여 using 문과 함께 사용하기 좋습니다.

```csharp
using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(4096);
Memory<byte> memory = owner.Memory;

int read = await stream.ReadAsync(memory);
Process(memory.Span.Slice(0, read));

// using 블록이 끝나면 자동으로 반환
```

이 방식은 메모리 소유권을 명시적으로 표현할 수 있어 코드가 더 안전해집니다.

---

## 고급 문자열 처리: string.Create와 TryFormat

### string.Create

문자열을 생성할 때 중간 할당을 피하려면 `string.Create`를 사용합니다. 이 메서드는 대상 문자열의 버퍼를 직접 채울 수 있는 델리게이트를 받습니다.

```csharp
string fullName = string.Create(
    firstName.Length + lastName.Length + 1,  // 총 길이
    (firstName, lastName),                   // 상태
    (span, state) =>
    {
        state.firstName.AsSpan().CopyTo(span);
        span[state.firstName.Length] = ' ';
        state.lastName.AsSpan().CopyTo(span.Slice(state.firstName.Length + 1));
    });
```

### TryFormat

값 타입에 `TryFormat` 메서드를 구현하면 문자열 변환 시 할당을 없앨 수 있습니다.

```csharp
readonly struct Point : ISpanFormattable
{
    public int X, Y;
    public bool TryFormat(Span<char> destination, out int charsWritten,
                         ReadOnlySpan<char> format, IFormatProvider? provider)
    {
        // 형식: (X, Y)
        if (destination.Length < 10) { charsWritten = 0; return false; }
        destination[0] = '(';
        X.TryFormat(destination.Slice(1), out int xLen, default, provider);
        destination[1 + xLen] = ',';
        destination[2 + xLen] = ' ';
        Y.TryFormat(destination.Slice(3 + xLen), out int yLen, default, provider);
        destination[3 + xLen + yLen] = ')';
        charsWritten = 4 + xLen + yLen;
        return true;
    }
}
```

이를 사용하면 `Span<char>`에 직접 포맷팅할 수 있어 힙 할당 없이 문자열 표현을 생성할 수 있습니다.

---

## 이진 데이터 처리: BinaryPrimitives와 MemoryMarshal

### BinaryPrimitives

엔디언(endianness)을 고려한 이진 데이터 변환을 제공합니다.

```csharp
ReadOnlySpan<byte> data = ...;
uint value = BinaryPrimitives.ReadUInt32LittleEndian(data);  // 리틀 엔디언
BinaryPrimitives.WriteUInt32BigEndian(buffer, 0x12345678);    // 빅 엔디언
```

### MemoryMarshal

메모리를 다른 타입으로 재해석(reinterpret)할 수 있습니다.

```csharp
int[] intArray = new int[] { 1, 2, 3 };
Span<byte> byteSpan = MemoryMarshal.AsBytes(intArray.AsSpan());  // int → byte
ref int first = ref MemoryMarshal.GetReference(byteSpan);        // 바이트 스팬의 첫 번째 요소를 int로 참조
```

이러한 재해석은 복사를 수반하지 않으므로 매우 빠르지만, 메모리 정렬(alignment)과 엔디언을 항상 고려해야 합니다.

---

## 실전 예제: 고성능 CSV 파서

Span을 활용하여 할당 없이 CSV 데이터를 파싱하는 예제입니다. 이 파서는 전체 데이터를 한 번에 읽어 들이고, 각 필드를 `Range`로 표현하여 원본 데이터를 직접 참조합니다.

```csharp
public ref struct CsvParser
{
    private readonly ReadOnlySpan<char> _data;
    private int _position;
    
    public CsvParser(ReadOnlySpan<char> data)
    {
        _data = data;
        _position = 0;
    }
    
    public bool TryReadRow(Span<Range> fields, out int fieldCount)
    {
        fieldCount = 0;
        if (_position >= _data.Length) return false;
        
        int start = _position;
        bool inQuotes = false;
        
        while (_position < _data.Length)
        {
            char c = _data[_position];
            if (c == '"')
                inQuotes = !inQuotes;
            else if (!inQuotes)
            {
                if (c == ',')
                {
                    fields[fieldCount++] = new Range(start, _position);
                    start = _position + 1;
                }
                else if (c == '\n')
                {
                    fields[fieldCount++] = new Range(start, _position);
                    _position++;
                    return true;
                }
                else if (c == '\r')
                {
                    // \r\n 처리
                    if (_position + 1 < _data.Length && _data[_position + 1] == '\n')
                        _position++;
                    fields[fieldCount++] = new Range(start, _position);
                    _position++;
                    return true;
                }
            }
            _position++;
        }
        
        // 마지막 필드
        if (start < _data.Length)
            fields[fieldCount++] = new Range(start, _data.Length);
        return true;
    }
    
    public ReadOnlySpan<char> GetField(ReadOnlySpan<char> row, Range range)
    {
        var field = row[range];
        if (field.Length >= 2 && field[0] == '"' && field[^1] == '"')
            return field.Slice(1, field.Length - 2);
        return field;
    }
}
```

사용 예:

```csharp
string csv = "Name,Age\nJohn,30\n\"Jane, Smith\",25";
var parser = new CsvParser(csv.AsSpan());
Span<Range> ranges = stackalloc Range[10];

while (parser.TryReadRow(ranges, out int count))
{
    for (int i = 0; i < count; i++)
    {
        var field = parser.GetField(csv.AsSpan(), ranges[i]);
        Console.Write($"{field.ToString()} ");
    }
    Console.WriteLine();
}
```

이 파서는 원본 데이터를 복사하지 않으며, 필드 단위로도 추가 할당을 최소화합니다.

---

## 성능 팁과 모범 사례

- **적절한 도구 선택**:  
  - 작은 임시 버퍼 → `stackalloc`  
  - 대용량 재사용 버퍼 → `ArrayPool<T>`  
  - 비동기/장기 저장 → `Memory<T>` + `MemoryPool<T>`

- **불필요한 복사 제거**:  
  `Substring` 대신 `AsSpan().Slice()` 사용, 배열 복사 대신 `CopyTo` 활용.

- **범위 검사 최적화**:  
  JIT가 인덱스 범위 검사를 최적화할 수 있도록 단순한 루프 작성.

- **소유권 명시**:  
  `IMemoryOwner<T>`를 반환하는 메서드는 호출자가 메모리를 반환해야 함을 명확히 표현.

- **안전성 우선**:  
  `Span` 사용 시 항상 길이를 검사하고, 예외가 발생해도 리소스 누수가 없도록 `try-finally` 또는 `using` 적용.

---

## 결론

`Span<T>`와 `Memory<T>`는 C#에서 고성능 메모리 관리를 가능하게 하는 핵심 요소입니다. 이들은 불필요한 할당과 복사를 제거하여 GC 부하를 줄이고 CPU 캐시 효율성을 높여줍니다. 초중급 개발자라면 먼저 `Span<T>`의 기본 사용법과 제약을 숙지하고, 점진적으로 `Memory<T>`, `ArrayPool<T>`, `MemoryMarshal` 등의 고급 기능을 적용해 나가는 것이 좋습니다.

항상 성능 측정을 통해 최적화 효과를 확인하고, 코드의 가독성과 유지보수성을 희생하지 않는 범위 내에서 이러한 기법들을 활용하세요. 적절한 상황에 적절한 도구를 선택하는 것이 안정적이고 빠른 애플리케이션을 만드는 지름길입니다.