---
layout: post
title: C# - PInvoke, Marshal, NativeMemory
date: 2024-11-01 19:20:23 +0900
category: Csharp
---
# C# Span<T>와 Memory<T> 심화: 고성능 메모리 조작 기법

## 서론: 메모리 접근의 혁명, Span과 Memory

C#에서 고성능 코드를 작성하는 데 있어 가장 중요한 혁신 중 하나가 `Span<T>`와 `Memory<T>`입니다. 이들은 메모리 조작 방식을 근본적으로 변화시켜, 안전성을 유지하면서도 C/C++ 수준의 성능을 달성할 수 있게 해줍니다. 기존의 배열 복사, 문자열 조작, 버퍼 관리에서 발생하던 불필요한 할당과 복사를 제거하여 애플리케이션 성능을 크게 향상시킬 수 있습니다.

이 가이드는 단순한 사용법을 넘어 실제 프로덕션 코드에서 이러한 타입들을 효과적으로 활용하는 고급 기법들을 다룹니다.

---

## 1. Span<T>: 스택 기반의 고속 메모리 뷰

### Span의 본질 이해하기

`Span<T>`는 메모리의 연속된 영역을 나타내는 값 타입입니다. 배열, 문자열, 비관리 메모리 등 다양한 소스의 데이터를 복사 없이 직접 조작할 수 있게 해줍니다.

```csharp
// 다양한 소스에서 Span 생성
int[] array = { 1, 2, 3, 4, 5 };
Span<int> spanFromArray = array;

string text = "Hello, World!";
ReadOnlySpan<char> spanFromString = text.AsSpan();

unsafe
{
    int* pointer = stackalloc int[5];
    Span<int> spanFromPointer = new Span<int>(pointer, 5);
}
```

### Span의 가장 큰 제약: ref struct

`Span<T>`는 `ref struct`이기 때문에 다음과 같은 제약이 있습니다:
- 힙에 저장할 수 없음 (클래스 필드, 제네릭 타입 인수 등)
- 박싱(boxing) 불가
- 비동기 메서드에서 사용 불가 (`async` 메서드의 로컬 변수로 사용 불가)
- 람다 표현식에 캡처 불가

```csharp
class MyClass
{
    // 컴파일 오류: Span<T>는 클래스 필드가 될 수 없음
    // Span<int> _fieldSpan;
    
    void Method()
    {
        int[] array = new int[10];
        Span<int> localSpan = array;  // 지역 변수로는 사용 가능
        
        // 컴파일 오류: 람다에 캡처 불가
        // Action action = () => Console.WriteLine(localSpan[0]);
    }
}
```

### 실용적인 Span 활용: 문자열 처리 최적화

기존의 `Substring` 메서드는 새로운 문자열을 생성하므로 메모리 할당이 발생합니다. `Span`을 사용하면 이를 방지할 수 있습니다.

```csharp
// 기존 방식 (할당 발생)
string ExtractFirstNameOld(string fullName)
{
    int spaceIndex = fullName.IndexOf(' ');
    return spaceIndex >= 0 ? fullName.Substring(0, spaceIndex) : fullName;
}

// Span을 사용한 방식 (할당 없음)
ReadOnlySpan<char> ExtractFirstNameSpan(ReadOnlySpan<char> fullName)
{
    int spaceIndex = fullName.IndexOf(' ');
    return spaceIndex >= 0 ? fullName.Slice(0, spaceIndex) : fullName;
}

// 사용 예제
string fullName = "John Smith";

// 기존 방식
string firstName1 = ExtractFirstNameOld(fullName);  // 새로운 문자열 생성

// Span 방식
ReadOnlySpan<char> firstNameSpan = ExtractFirstNameSpan(fullName.AsSpan());
// Span을 문자열로 변환할 때만 할당 발생
string firstName2 = firstNameSpan.ToString();
```

---

## 2. Memory<T>: 힙에 저장 가능한 메모리 뷰

`Memory<T>`는 `Span<T>`와 유사하지만 `ref struct`가 아니기 때문에 힙에 저장할 수 있습니다. 이는 비동기 코드나 클래스 필드에서 메모리 조각을 유지해야 할 때 필요합니다.

```csharp
class DataProcessor
{
    // Memory<T>는 클래스 필드가 될 수 있음
    private Memory<byte> _buffer;
    
    public DataProcessor(int bufferSize)
    {
        _buffer = new byte[bufferSize];
    }
    
    public async Task ProcessAsync(Stream stream)
    {
        // Stream에서 Memory<T>로 직접 읽기
        int bytesRead = await stream.ReadAsync(_buffer);
        
        // 작업을 위해 Span<T>로 변환
        Span<byte> span = _buffer.Span.Slice(0, bytesRead);
        ProcessData(span);
    }
    
    private void ProcessData(Span<byte> data)
    {
        // 데이터 처리 로직
        for (int i = 0; i < data.Length; i++)
        {
            data[i] = (byte)(data[i] ^ 0xFF);  // 비트 반전
        }
    }
}
```

### Memory와 Span의 관계

`Memory<T>`는 `Span<T>`의 "소유자" 역할을 합니다. `Memory<T>`에서 `Span<T>`를 얻어서 실제 작업을 수행합니다.

```csharp
Memory<int> memory = new int[100];
Span<int> span = memory.Span;  // Memory에서 Span 얻기

// 작업 수행
for (int i = 0; i < span.Length; i++)
{
    span[i] = i * 2;
}

// 결과 확인
int[] array = memory.ToArray();
Console.WriteLine($"첫 번째 요소: {array[0]}");  // 0
Console.WriteLine($"마지막 요소: {array[99]}"); // 198
```

---

## 3. stackalloc: 스택에 메모리 할당

`stackalloc`은 스택에 메모리를 할당하는 키워드로, 매우 빠르지만 제한된 크기와 수명을 가집니다.

```csharp
// 스택에 100개의 정수 할당
Span<int> buffer = stackalloc int[100];

// 데이터 채우기
for (int i = 0; i < buffer.Length; i++)
{
    buffer[i] = i * i;
}

// 데이터 사용
int sum = 0;
foreach (int value in buffer)
{
    sum += value;
}

Console.WriteLine($"합계: {sum}");
```

### stackalloc의 주의사항

1. **크기 제한**: 너무 큰 메모리를 할당하면 스택 오버플로우가 발생할 수 있습니다.
2. **수명**: 메서드가 반환되면 자동으로 해제됩니다.
3. **초기화**: `stackalloc`으로 할당된 메모리는 자동으로 초기화되지 않습니다.

```csharp
// 안전한 stackalloc 사용 패턴
const int MaxStackAllocSize = 1024;  // 적절한 최대 크기 정의

static void ProcessData(ReadOnlySpan<byte> data)
{
    // 작은 크기일 때만 stackalloc 사용
    if (data.Length <= MaxStackAllocSize)
    {
        Span<byte> buffer = stackalloc byte[data.Length];
        data.CopyTo(buffer);
        // buffer 사용
    }
    else
    {
        // 크기가 크면 배열 풀 사용
        byte[] rented = ArrayPool<byte>.Shared.Rent(data.Length);
        try
        {
            Span<byte> buffer = rented.AsSpan(0, data.Length);
            data.CopyTo(buffer);
            // buffer 사용
        }
        finally
        {
            ArrayPool<byte>.Shared.Return(rented);
        }
    }
}
```

---

## 4. ArrayPool<T>: 배열 재사용으로 GC 부하 줄이기

`ArrayPool<T>`는 배열을 재사용할 수 있는 풀(pool)을 제공하여 가비지 컬렉터의 부하를 줄여줍니다.

```csharp
using System.Buffers;

class ArrayPoolExample
{
    static void ProcessLargeData(byte[] data)
    {
        // 필요한 크기보다 더 큰 배열을 빌릴 수 있음
        byte[] buffer = ArrayPool<byte>.Shared.Rent(data.Length * 2);
        
        try
        {
            // 데이터 복사
            data.CopyTo(buffer, 0);
            
            // 버퍼 사용 (실제 사용 크기만큼 Span 생성)
            Span<byte> workingSpan = buffer.AsSpan(0, data.Length);
            ProcessBuffer(workingSpan);
        }
        finally
        {
            // 반드시 반환
            ArrayPool<byte>.Shared.Return(buffer);
        }
    }
    
    static void ProcessBuffer(Span<byte> buffer)
    {
        // 버퍼 처리 로직
        for (int i = 0; i < buffer.Length; i++)
        {
            buffer[i] = (byte)(buffer[i] + 1);
        }
    }
}
```

### ArrayPool의 고급 사용법

```csharp
using System.Buffers;

class AdvancedArrayPoolUsage
{
    // 여러 메서드에서 배열 공유 시 주의사항
    public static void ProcessWithSharedBuffer()
    {
        const int bufferSize = 4096;
        byte[] buffer = ArrayPool<byte>.Shared.Rent(bufferSize);
        
        try
        {
            // 첫 번째 작업
            FillBuffer(buffer, 0, 100);
            ProcessChunk(buffer.AsSpan(0, 100));
            
            // 두 번째 작업 (같은 버퍼 재사용)
            FillBuffer(buffer, 100, 200);
            ProcessChunk(buffer.AsSpan(100, 200));
            
            // 버퍼 초기화가 필요한 경우
            buffer.AsSpan(0, 300).Clear();
        }
        finally
        {
            ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
        }
    }
    
    private static void FillBuffer(byte[] buffer, int offset, int length)
    {
        var random = new Random();
        for (int i = 0; i < length; i++)
        {
            buffer[offset + i] = (byte)random.Next(256);
        }
    }
    
    private static void ProcessChunk(Span<byte> chunk)
    {
        // 청크 처리 로직
        Console.WriteLine($"처리 중: {chunk.Length}바이트");
    }
}
```

---

## 5. MemoryPool<T>와 IMemoryOwner<T>: 더 안전한 메모리 관리

`MemoryPool<T>`는 `IMemoryOwner<T>`를 통해 메모리 수명을 명시적으로 관리할 수 있는 더 안전한 패턴을 제공합니다.

```csharp
using System.Buffers;

class MemoryPoolExample
{
    public static async Task ProcessStreamAsync(Stream stream)
    {
        // 메모리 임대 (using으로 자동 반환)
        using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(4096);
        
        // Memory<T> 얻기
        Memory<byte> memory = owner.Memory;
        
        // 스트림에서 읽기
        int bytesRead = await stream.ReadAsync(memory);
        
        // 읽은 데이터 처리
        ProcessData(memory.Span.Slice(0, bytesRead));
        
        // using 블록이 끝나면 자동으로 반환됨
    }
    
    private static void ProcessData(Span<byte> data)
    {
        // 데이터 처리
        Console.WriteLine($"처리된 데이터 크기: {data.Length}바이트");
    }
}
```

### IMemoryOwner의 장점

1. **명시적 수명 관리**: `using` 문과 함께 사용하면 자동으로 메모리가 반환됩니다.
2. **실수 방지**: 반환을 잊어버리는 실수를 방지할 수 있습니다.
3. **API 설계**: 메서드가 메모리 소유권을 가져갈 때 명확하게 표현할 수 있습니다.

---

## 6. 고급 문자열 처리: string.Create와 TryFormat

### string.Create: 효율적인 문자열 생성

`string.Create` 메서드를 사용하면 문자열을 생성할 때 중간 할당을 피할 수 있습니다.

```csharp
class StringCreateExample
{
    public static string CreateFullName(string firstName, string lastName)
    {
        // 기존 방식
        // return firstName + " " + lastName;  // 중간 할당 발생
        
        // string.Create 사용
        return string.Create(
            firstName.Length + lastName.Length + 1,  // 전체 길이
            (firstName, lastName),                   // 상태
            (span, state) =>                         // 생성자
            {
                state.firstName.AsSpan().CopyTo(span);
                span[state.firstName.Length] = ' ';
                state.lastName.AsSpan().CopyTo(span.Slice(state.firstName.Length + 1));
            });
    }
    
    // 사용 예제
    static void Demo()
    {
        string fullName = CreateFullName("John", "Doe");
        Console.WriteLine(fullName);  // "John Doe"
    }
}
```

### TryFormat: 할당 없는 포맷팅

값 타입에 `TryFormat` 메서드를 구현하면 할당 없이 문자열로 포맷팅할 수 있습니다.

```csharp
readonly struct Point : ISpanFormattable
{
    public int X { get; }
    public int Y { get; }
    
    public Point(int x, int y) => (X, Y) = (x, y);
    
    // ToString() 대신 사용할 수 있는 효율적인 메서드
    public bool TryFormat(Span<char> destination, out int charsWritten, 
                         ReadOnlySpan<char> format, IFormatProvider? provider)
    {
        // 포맷팅 로직
        if (destination.Length < 10)  // "(X, Y)" 형식에 충분한 공간인지 확인
        {
            charsWritten = 0;
            return false;
        }
        
        destination[0] = '(';
        if (!X.TryFormat(destination.Slice(1), out int xWritten, default, provider))
        {
            charsWritten = 0;
            return false;
        }
        
        destination[1 + xWritten] = ',';
        destination[2 + xWritten] = ' ';
        
        if (!Y.TryFormat(destination.Slice(3 + xWritten), out int yWritten, default, provider))
        {
            charsWritten = 0;
            return false;
        }
        
        destination[3 + xWritten + yWritten] = ')';
        charsWritten = 4 + xWritten + yWritten;
        return true;
    }
    
    // 기존 ToString() 호출을 위해 필요
    public override string ToString()
    {
        // 내부적으로 TryFormat 사용
        Span<char> buffer = stackalloc char[32];
        return TryFormat(buffer, out int charsWritten, default, null) 
            ? new string(buffer.Slice(0, charsWritten))
            : base.ToString();
    }
}

// 사용 예제
static void DemoPointFormatting()
{
    Point point = new Point(10, 20);
    
    // 할당 없는 포맷팅
    Span<char> buffer = stackalloc char[32];
    if (point.TryFormat(buffer, out int charsWritten, default, null))
    {
        ReadOnlySpan<char> formatted = buffer.Slice(0, charsWritten);
        Console.WriteLine(formatted.ToString());  // "(10, 20)"
    }
}
```

---

## 7. 이진 데이터 처리: BinaryPrimitives와 MemoryMarshal

### BinaryPrimitives: 엔디언 안전한 이진 처리

`BinaryPrimitives` 클래스는 엔디언(endianness)을 고려한 안전한 이진 데이터 처리를 제공합니다.

```csharp
using System.Buffers.Binary;

class BinaryPrimitivesExample
{
    public static void ProcessBinaryData(ReadOnlySpan<byte> data)
    {
        if (data.Length < 8) return;
        
        // 리틀 엔디언으로 정수 읽기
        uint littleEndianValue = BinaryPrimitives.ReadUInt32LittleEndian(data);
        
        // 빅 엔디언으로 정수 읽기
        uint bigEndianValue = BinaryPrimitives.ReadUInt32BigEndian(data.Slice(4));
        
        Console.WriteLine($"리틀 엔디언: {littleEndianValue}");
        Console.WriteLine($"빅 엔디언: {bigEndianValue}");
        
        // 데이터 쓰기
        Span<byte> buffer = stackalloc byte[8];
        BinaryPrimitives.WriteUInt32LittleEndian(buffer, 0x12345678);
        BinaryPrimitives.WriteUInt32BigEndian(buffer.Slice(4), 0x87654321);
    }
}
```

### MemoryMarshal: 메모리 재해석

`MemoryMarshal` 클래스는 메모리를 다른 타입으로 재해석할 수 있는 고급 기능을 제공합니다.

```csharp
using System.Runtime.InteropServices;

class MemoryMarshalExample
{
    public static void ReinterpretMemory()
    {
        // 정수 배열을 바이트 배열로 재해석
        int[] intArray = { 0x11223344, 0x55667788, 0x99AABBCC };
        Span<int> intSpan = intArray.AsSpan();
        
        // int 배열을 byte 배열로 재해석 (복사 없음)
        Span<byte> byteSpan = MemoryMarshal.AsBytes(intSpan);
        
        Console.WriteLine($"바이트 개수: {byteSpan.Length}");  // 12 (3 * 4)
        Console.WriteLine($"첫 번째 바이트: 0x{byteSpan[0]:X2}");  // 플랫폼에 따라 다름
        
        // 구조체로 재해석
        if (byteSpan.Length >= Marshal.SizeOf<Point>())
        {
            ref Point point = ref MemoryMarshal.AsRef<Point>(byteSpan);
            Console.WriteLine($"구조체 X: {point.X}");
        }
    }
    
    [StructLayout(LayoutKind.Sequential)]
    struct Point
    {
        public int X;
        public int Y;
    }
}
```

**주의**: `MemoryMarshal`을 사용할 때는 메모리 정렬(alignment)과 엔디언을 항상 고려해야 합니다.

---

## 8. 실전 예제: 고성능 CSV 파서

실제 프로덕션 환경에서 Span과 Memory를 활용하는 방법을 알아보기 위해 고성능 CSV 파서를 구현해 보겠습니다.

```csharp
using System;
using System.Buffers;

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
        
        if (_position >= _data.Length)
            return false;
        
        int fieldStart = _position;
        bool inQuotes = false;
        
        while (_position < _data.Length)
        {
            char current = _data[_position];
            
            if (current == '"')
            {
                inQuotes = !inQuotes;
            }
            else if (!inQuotes)
            {
                if (current == ',')
                {
                    // 필드 종료
                    if (fieldCount >= fields.Length)
                        return false;  // 필드 버퍼가 부족함
                    
                    fields[fieldCount++] = new Range(fieldStart, _position);
                    fieldStart = _position + 1;
                }
                else if (current == '\n' || current == '\r')
                {
                    // 행 종료 처리
                    if (_position > 0 && _data[_position - 1] == '\r' && current == '\n')
                    {
                        // \r\n 처리
                        if (fieldCount >= fields.Length)
                            return false;
                        
                        fields[fieldCount++] = new Range(fieldStart, _position - 1);
                    }
                    else
                    {
                        if (fieldCount >= fields.Length)
                            return false;
                        
                        fields[fieldCount++] = new Range(fieldStart, _position);
                    }
                    
                    // 줄바꿈 문자 건너뛰기
                    _position++;
                    if (current == '\r' && _position < _data.Length && _data[_position] == '\n')
                        _position++;
                    
                    return true;
                }
            }
            
            _position++;
        }
        
        // 마지막 필드 (파일 끝)
        if (fieldStart <= _data.Length && fieldCount < fields.Length)
        {
            fields[fieldCount++] = new Range(fieldStart, _data.Length);
            return true;
        }
        
        return false;
    }
    
    public ReadOnlySpan<char> GetField(ReadOnlySpan<char> row, Range range)
    {
        var field = row[range];
        
        // 따옴표 제거
        if (field.Length >= 2 && field[0] == '"' && field[^1] == '"')
        {
            return field.Slice(1, field.Length - 2);
        }
        
        return field;
    }
}

class Program
{
    static void ProcessCsv(ReadOnlySpan<char> csvData)
    {
        var parser = new CsvParser(csvData);
        Span<Range> fieldRanges = stackalloc Range[20];  // 최대 20개 필드
        
        int rowCount = 0;
        while (parser.TryReadRow(fieldRanges, out int fieldCount))
        {
            rowCount++;
            Console.WriteLine($"행 {rowCount}, 필드 {fieldCount}개:");
            
            for (int i = 0; i < fieldCount; i++)
            {
                var field = parser.GetField(csvData, fieldRanges[i]);
                Console.WriteLine($"  필드 {i + 1}: '{field.ToString()}'");
            }
            
            Console.WriteLine();
        }
        
        Console.WriteLine($"총 {rowCount}행 처리 완료");
    }
    
    static void Main()
    {
        // 테스트 CSV 데이터
        string csvData = """
            Name,Age,City
            "John Doe",30,"New York, NY"
            "Jane Smith",25,London
            Bob Johnson,35,"San Francisco, CA"
            """;
        
        ProcessCsv(csvData.AsSpan());
    }
}
```

이 CSV 파서의 주요 장점:
1. **할당 없음**: 문자열 복사 없이 원본 데이터를 직접 처리
2. **고성능**: 루프 내부에서 힙 할당이 발생하지 않음
3. **메모리 효율**: 큰 CSV 파일도 적은 메모리로 처리 가능
4. **안전성**: 버퍼 오버런 방지를 위한 경계 검사 포함

---

## 9. 성능 팁과 모범 사례

### 성능 최적화 팁

1. **적절한 도구 선택**:
   - 작은 임시 버퍼: `stackalloc`
   - 큰 재사용 버퍼: `ArrayPool<T>`
   - 비동기/장기 저장: `Memory<T>` + `MemoryPool<T>`

2. **불필요한 복사 피하기**:
   ```csharp
   // 나쁜 예: 불필요한 복사
   byte[] data = GetData();
   ProcessData(data.ToArray());  // 복사 발생
   
   // 좋은 예: 복사 없음
   byte[] data = GetData();
   ProcessData(data.AsSpan());
   ```

3. **범위 검사 최적화**:
   ```csharp
   // JIT가 범위 검사를 제거할 수 있도록 작성
   static void ProcessSpan(Span<int> span)
   {
       // 나쁜 예: 매번 범위 검사
       for (int i = 0; i < span.Length; i++)
       {
           if (i < span.Length)  // 불필요한 검사
               span[i] = i * 2;
       }
       
       // 좋은 예: JIT 최적화 가능
       for (int i = 0; i < span.Length; i++)
       {
           span[i] = i * 2;
       }
   }
   ```

### 안전성 모범 사례

1. **Span 수명 관리**:
   - `Span<T>`는 스택에만 저장
   - 비동기 코드에는 `Memory<T>` 사용
   - 람다에 캡처하지 않기

2. **Memory 소유권 명시**:
   ```csharp
   // 명확한 소유권 전달
   void ProcessData(IMemoryOwner<byte> dataOwner)
   {
       using (dataOwner)  // 소유권 가져옴
       {
           ProcessSpan(dataOwner.Memory.Span);
       }  // 자동으로 반환
   }
   ```

3. **버퍼 오버런 방지**:
   ```csharp
   // 항상 길이 검사
   static void SafeCopy(ReadOnlySpan<byte> source, Span<byte> destination)
   {
       if (source.Length > destination.Length)
           throw new ArgumentException("대상 버퍼가 너무 작습니다.");
       
       source.CopyTo(destination);
   }
   ```

---

## 결론: 현대적인 C#의 고성능 메모리 관리

`Span<T>`, `Memory<T>`, 그리고 관련 도구들은 C#에서 고성능 코드를 작성하는 방식을 근본적으로 변화시켰습니다. 이러한 도구들을 효과적으로 사용하면 다음과 같은 이점을 얻을 수 있습니다:

### 핵심 이점 정리

1. **메모리 효율성**: 불필요한 할당과 복사를 제거하여 메모리 사용량을 크게 줄일 수 있습니다.
2. **성능 향상**: 가비지 컬렉터의 부하를 줄이고 CPU 캐시 효율성을 향상시킵니다.
3. **안전성**: 메모리 안전성을 유지하면서 저수준 최적화를 수행할 수 있습니다.
4. **표준화**: 일관된 API로 다양한 메모리 소스(배열, 문자열, 비관리 메모리)를 처리할 수 있습니다.

### 적절한 사용 시나리오

- **Span<T>**: 동기 메서드 내부의 임시 작업, 문자열/바이너리 파싱, 고성능 알고리즘
- **Memory<T>**: 비동기 작업, 클래스 필드 저장, 장기간 메모리 참조 필요 시
- **stackalloc**: 작은 임시 버퍼, 로컬 계산, 수명이 짧은 작업
- **ArrayPool/MemoryPool**: 대용량 버퍼 재사용, 성능이 중요한 핵심 경로

### 마지막 조언

이러한 고급 기능들을 사용할 때는 항상 다음과 같은 원칙을 기억하세요:

1. **점진적 적용**: 기존 코드를 한 번에 모두 변경하지 말고, 성능 병목이 확인된 부분부터 적용하세요.
2. **측정 기반 최적화**: 최적화 전후로 성능을 측정하여 실제 개선 효과를 확인하세요.
3. **가독성 유지**: 지나치게 복잡한 최적화는 코드 유지보수를 어렵게 만듭니다.
4. **안전성 우선**: 성능보다 안전성이 더 중요합니다. 항상 경계 검사와 예외 처리를 포함하세요.

Span과 Memory는 강력한 도구이지만, 모든 문제에 대한 만능 해결책은 아닙니다. 적절한 상황에 적절한 도구를 선택하는 것이 중요합니다. 이러한 기술들을 올바르게 활용하면 C#으로도 시스템 수준의 고성능 애플리케이션을 구축할 수 있습니다.