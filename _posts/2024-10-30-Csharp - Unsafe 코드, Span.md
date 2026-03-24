---
layout: post
title: C# - Unsafe 코드, Span
date: 2024-10-30 19:20:23 +0900
category: Csharp
---
# C# 포인터와 Unsafe 코드, Span<T> 기초 정리

## 왜 Unsafe 코드와 Span<T>를 알아야 하는가

C#은 타입 안전성과 자동 메모리 관리를 제공하는 현대적인 언어입니다. 하지만 네이티브 라이브러리와의 상호작용, 고성능 데이터 처리, 메모리 복사 최소화 등이 필요한 상황에서는 이러한 추상화를 넘어서 더 낮은 수준의 제어가 필요할 때가 있습니다. 이 글은 C#에서 안전하게 저수준 메모리 작업을 수행하는 방법을 실용적인 관점에서 설명합니다.

---

## 프로젝트 설정: Unsafe 코드 활성화

Unsafe 코드를 사용하려면 프로젝트 설정에서 명시적으로 허용해야 합니다. 프로젝트 파일(.csproj)에 다음 설정을 추가합니다.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  </PropertyGroup>
</Project>
```

Visual Studio를 사용한다면 프로젝트 속성의 빌드 설정에서 "안전하지 않은 코드 허용"을 체크하면 됩니다.

---

## Unsafe 코드의 기초: 포인터 이해하기

### unsafe 컨텍스트

C#에서 포인터 연산을 사용하려면 `unsafe` 키워드로 블록을 감싸야 합니다.

```csharp
unsafe
{
    int number = 42;
    int* pointer = &number;      // number 변수의 주소를 가져옴
    Console.WriteLine(*pointer); // 42
    
    *pointer = 100;              // 포인터를 통해 값 변경
    Console.WriteLine(number);   // 100
}
```

`*`는 포인터 타입을 선언하거나 역참조(값을 읽거나 씀)할 때 사용합니다. `&`는 변수의 주소를 가져옵니다.

### 포인터 산술

포인터는 메모리 주소를 다루므로 산술 연산이 가능합니다. `int*`에 1을 더하면 실제로는 4바이트(타입 크기)가 이동합니다.

```csharp
unsafe
{
    int[] numbers = { 10, 20, 30, 40, 50 };
    
    fixed (int* ptr = numbers)
    {
        int* current = ptr;
        for (int i = 0; i < numbers.Length; i++)
        {
            Console.WriteLine(*current);
            current++;   // 다음 int 위치로 이동
        }
        
        Console.WriteLine(ptr[2]);   // 인덱서로도 접근 가능: 30
    }
}
```

---

## 메모리 고정: fixed 키워드

.NET의 가비지 컬렉터는 메모리를 효율적으로 관리하기 위해 객체를 이동시킬 수 있습니다. 이는 포인터를 사용할 때 문제가 되는데, 객체가 이동하면 포인터가 유효하지 않게 되기 때문입니다. `fixed` 키워드는 객체를 고정하여 GC가 이동하지 못하게 합니다.

```csharp
unsafe
{
    int[] data = { 1, 2, 3, 4, 5 };
    
    fixed (int* ptr = data)
    {
        for (int i = 0; i < data.Length; i++)
        {
            ptr[i] *= 2;   // 배열 요소 수정
        }
    }   // 고정 해제
    
    // 결과: 2, 4, 6, 8, 10
}
```

고정은 가비지 컬렉터의 효율성을 저하시킬 수 있으므로, 가능한 한 짧은 시간 동안만 사용해야 합니다.

---

## 스택 할당: stackalloc

`stackalloc` 키워드를 사용하면 힙이 아닌 스택에 메모리를 할당할 수 있습니다. 이는 힙 할당보다 빠르고 가비지 컬렉션의 영향을 받지 않습니다.

```csharp
unsafe
{
    int* buffer = stackalloc int[100];
    for (int i = 0; i < 100; i++)
    {
        buffer[i] = i * 2;
    }
    // 함수 반환 시 자동 해제
}
```

`stackalloc`은 `Span<T>`와 함께 사용하면 더 안전하고 편리합니다. 이 경우 `unsafe` 문맥이 필요하지 않습니다.

```csharp
Span<int> buffer = stackalloc int[100];
for (int i = 0; i < buffer.Length; i++)
{
    buffer[i] = i * 2;
}
```

스택 할당은 함수 반환 시 자동 해제되지만, 스택 크기(기본 1MB)를 초과하면 안 됩니다. 큰 버퍼에는 적합하지 않습니다.

---

## Span<T>: 안전한 메모리 슬라이스

`Span<T>`는 연속적인 메모리 영역을 안전하게 표현하는 구조체입니다. 배열, 네이티브 메모리, 스택 메모리 등 다양한 원본을 동일한 방식으로 다룰 수 있습니다.

### Span<T> 생성하기

```csharp
// 배열로부터 생성
int[] array = { 1, 2, 3, 4, 5 };
Span<int> spanFromArray = array.AsSpan();

// 배열의 일부분 슬라이스 (할당 없음)
Span<int> slice = array.AsSpan(1, 3);   // [2, 3, 4]

// 스택 할당과 함께 사용
Span<byte> stackBuffer = stackalloc byte[1024];
```

### ReadOnlySpan<T>

읽기 전용 버전인 `ReadOnlySpan<T>`는 데이터를 수정하지 않고 읽기만 할 때 사용합니다. 문자열 처리에 특히 유용합니다.

```csharp
string text = "Hello, World!";
ReadOnlySpan<char> charSpan = text.AsSpan();

// 부분 문자열 생성 없이 슬라이싱
ReadOnlySpan<char> hello = charSpan.Slice(0, 5);
ReadOnlySpan<char> world = charSpan.Slice(7, 5);

Console.WriteLine(hello.ToString());   // "Hello"
Console.WriteLine(world.ToString());   // "World"
```

### Span<T>의 주요 기능

```csharp
int[] data = { 10, 20, 30, 40, 50, 60, 70, 80, 90, 100 };
Span<int> span = data.AsSpan();

Span<int> firstHalf = span[..5];      // 처음 5개
Span<int> secondHalf = span[5..];     // 나머지
Span<int> middle = span[2..7];        // 인덱스 2~6

int index = span.IndexOf(50);         // 50의 위치 찾기
span.Slice(2, 5).CopyTo(destination); // 일부를 다른 배열에 복사
```

---

## ref struct와 Memory<T>

`Span<T>`는 `ref struct` 타입입니다. 이는 스택에만 존재할 수 있다는 특별한 제약을 가지며, 이러한 제약은 메모리 안전성을 보장하기 위해 설계되었습니다.

### ref struct의 제약

- 힙에 저장할 수 없음 (클래스의 필드로 선언 불가)
- 박싱할 수 없음
- 비동기 메서드(async)에서 사용할 수 없음
- 이터레이터(yield return)에서 사용할 수 없음

이러한 제약 때문에 `Span<T>`는 비동기 작업이나 클래스 필드로 저장해야 하는 상황에서는 사용할 수 없습니다. 그럴 때는 `Memory<T>`를 대안으로 사용합니다.

```csharp
// Memory<T>는 힙에 저장 가능
class BufferHolder
{
    public Memory<byte> Buffer { get; set; }
}

// 비동기 작업에서 사용 가능
async Task ProcessAsync(Memory<byte> buffer)
{
    await Task.Delay(100);
    Span<byte> span = buffer.Span;   // 필요 시 Span으로 변환
    // 작업 수행
}
```

---

## MemoryMarshal: 저수준 메모리 작업

`MemoryMarshal` 클래스는 `Span<T>`와 관련된 저수준 작업을 제공합니다. 주로 타입 변환과 메모리 레이아웃 관련 작업에 사용됩니다.

### 타입 변환

```csharp
int[] intArray = { 1, 2, 3, 4, 5 };
Span<int> intSpan = intArray.AsSpan();

// int 배열을 byte 배열로 보기 (복사 없음)
Span<byte> byteSpan = MemoryMarshal.AsBytes(intSpan);

// 다시 int 배열로 보기
Span<int> restored = MemoryMarshal.Cast<byte, int>(byteSpan);
```

### 구조체와의 상호작용

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct Point
{
    public int X;
    public int Y;
}

Point[] points = new Point[10];
Span<Point> pointSpan = points.AsSpan();

// 구조체 배열을 byte 배열로 변환
Span<byte> pointBytes = MemoryMarshal.AsBytes(pointSpan);
```

---

## 실전 예제: 고성능 데이터 처리

### 이미지 데이터 처리

이미지 데이터를 처리할 때 `Span<T>`를 사용하면 불필요한 할당 없이 픽셀을 직접 조작할 수 있습니다.

```csharp
public static void ApplyGrayscale(Span<byte> imageData, int width, int height)
{
    int bytesPerPixel = 4;   // BGRA 형식 가정
    int stride = width * bytesPerPixel;
    
    for (int y = 0; y < height; y++)
    {
        Span<byte> row = imageData.Slice(y * stride, stride);
        
        for (int x = 0; x < width; x++)
        {
            int idx = x * bytesPerPixel;
            byte blue = row[idx];
            byte green = row[idx + 1];
            byte red = row[idx + 2];
            
            byte gray = (byte)((red * 0.299) + (green * 0.587) + (blue * 0.114));
            
            row[idx] = gray;     // Blue
            row[idx + 1] = gray; // Green
            row[idx + 2] = gray; // Red
            // Alpha는 변경하지 않음
        }
    }
}
```

### 네트워크 패킷 파싱

```csharp
public static bool TryParsePacket(ReadOnlySpan<byte> data, out Packet packet)
{
    packet = default;
    
    if (data.Length < 8) return false;
    
    // 마지막 4바이트는 체크섬
    ReadOnlySpan<byte> payload = data[..^4];
    uint expectedChecksum = BinaryPrimitives.ReadUInt32LittleEndian(data[^4..]);
    
    if (CalculateChecksum(payload) != expectedChecksum) return false;
    
    ushort packetId = BinaryPrimitives.ReadUInt16LittleEndian(payload);
    ushort dataLength = BinaryPrimitives.ReadUInt16LittleEndian(payload[2..]);
    
    if (payload.Length < 4 + dataLength) return false;
    
    packet = new Packet(packetId, payload.Slice(4, dataLength).ToArray());
    return true;
}
```

---

## 안전한 Unsafe 코드 작성 지침

### 가능하면 Span<T> 사용

포인터 대신 `Span<T>`를 사용하면 경계 검사가 자동으로 이루어져 메모리 안전성이 보장됩니다.

```csharp
// 권장: Span<T> 사용
void ProcessWithSpan(Span<int> data)
{
    for (int i = 0; i < data.Length; i++)
        data[i] *= 2;
}

// 비권장: 직접 포인터 사용 (경계 검사 없음)
unsafe void ProcessWithPointer(int* data, int length)
{
    for (int i = 0; i < length; i++)
        data[i] *= 2;
}
```

### 고정 범위 최소화

`fixed` 블록은 가능한 한 짧게 유지합니다.

```csharp
unsafe
{
    fixed (byte* ptr = data)
    {
        // 빠른 작업만 수행
        NativeLibrary.Process(ptr, data.Length);
    }
    // 고정 해제 후 다른 작업 수행
}
```

### 스택 할당 크기 제한

`stackalloc`은 적절한 크기로 제한합니다. 1MB에 가까운 할당은 스택 오버플로우를 유발할 수 있습니다.

```csharp
Span<byte> buffer = stackalloc byte[1024];   // 적절한 크기
```

---

## 결론

C#의 Unsafe 코드와 `Span<T>`는 고성능 시나리오에서 강력한 도구이지만, 신중하게 사용해야 합니다.

**안전성 우선**: 가능하면 `Span<T>`와 `Memory<T>`를 사용하여 안전한 추상화를 활용하세요. 이들은 경계 검사와 메모리 안전성을 제공합니다.

**적절한 추상화 선택**: 일반적인 작업은 안전한 관리 코드로, 고성능 메모리 작업은 `Span<T>`로, 네이티브 상호작용이 필요한 경우에만 포인터를 사용하는 식으로 계층을 나누세요.

**메모리 수명 관리**: `fixed`, `stackalloc`, `Span<T>`의 수명을 이해하고 관리하세요. 특히 `ref struct`의 제약을 반드시 숙지해야 합니다.

**성능 vs 안전성 균형**: 성능 최적화가 정말 필요한지 확인하세요. 프로파일링을 통해 실제 병목이 확인된 경우에만 Unsafe 코드를 도입하는 것이 바람직합니다.

C#은 안전한 코드와 고성능 코드 사이의 균형을 잘 잡을 수 있는 언어입니다. Unsafe 코드와 `Span<T>`는 이 균형을 유지하면서도 필요한 경우 낮은 수준의 제어를 가능하게 하는 도구입니다. 올바른 상황에서 적절하게 사용한다면, C#으로도 시스템 수준의 성능을 요구하는 애플리케이션을 충분히 구축할 수 있습니다.