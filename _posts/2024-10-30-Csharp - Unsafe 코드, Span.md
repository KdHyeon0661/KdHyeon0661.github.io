---
layout: post
title: C# - Unsafe 코드, Span
date: 2024-10-30 19:20:23 +0900
category: Csharp
---
# C# 포인터와 Unsafe 코드, Span<T> 기초 정리

## 왜 Unsafe 코드와 Span<T>를 알아야 하는가?

C#은 기본적으로 타입 안전성과 자동 메모리 관리를 제공하는 현대적인 프로그래밍 언어입니다. 그러나 특정 상황에서는 이러한 추상화를 넘어서 더 낮은 수준의 메모리 제어가 필요할 때가 있습니다. 네이티브 라이브러리와의 상호작용, 고성능 데이터 처리, 메모리 복사 최소화 등이 대표적인 사례입니다. 이 글은 C#에서 안전하게 저수준 메모리 작업을 수행하는 방법을 실용적인 관점에서 설명합니다.

---

## 프로젝트 설정: Unsafe 코드 활성화

Unsafe 코드를 사용하려면 프로젝트 설정에서 명시적으로 허용해야 합니다.

### 프로젝트 파일 설정

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  </PropertyGroup>
</Project>
```

### 명령줄 빌드

```bash
dotnet build -p:AllowUnsafeBlocks=true
```

이 설정은 프로젝트 전체에 unsafe 코드 사용을 허용합니다. Visual Studio에서도 프로젝트 속성의 빌드 설정에서 "안전하지 않은 코드 허용"을 체크할 수 있습니다.

---

## Unsafe 코드의 기초: 포인터 이해하기

### unsafe 컨텍스트

C#에서 포인터 연산을 사용하려면 `unsafe` 키워드로 블록을 감싸야 합니다:

```csharp
unsafe
{
    int number = 42;
    int* pointer = &number;  // number 변수의 주소를 가져옴
    Console.WriteLine(*pointer);  // 포인터를 역참조하여 값 읽기: 42
    
    *pointer = 100;  // 포인터를 통해 값 변경
    Console.WriteLine(number);  // 100
}
```

### 포인터 연산자

| 연산자 | 설명 | 예제 |
|--------|------|------|
| `*` | 포인터 타입 선언 또는 역참조 | `int* ptr;` 또는 `int value = *ptr;` |
| `&` | 변수의 주소 가져오기 | `int* ptr = &value;` |
| `->` | 구조체 포인터를 통한 멤버 접근 | `point->X = 10;` |
| `[]` | 포인터 인덱싱 | `ptr[2] = 42;` |

### 포인터 산술

포인터는 메모리 주소를 다루므로 산술 연산이 가능합니다:

```csharp
unsafe
{
    int[] numbers = { 10, 20, 30, 40, 50 };
    
    fixed (int* ptr = numbers)
    {
        // 포인터 산술을 통한 배열 접근
        int* current = ptr;
        for (int i = 0; i < numbers.Length; i++)
        {
            Console.WriteLine($"numbers[{i}] = {*current}");
            current++;  // 다음 int 위치로 이동 (4바이트)
        }
        
        // 인덱서를 통한 직접 접근
        Console.WriteLine($"Third element: {ptr[2]}");  // 30
    }
}
```

포인터 산술 시 타입 크기에 맞게 자동으로 조정됩니다. `int*` 포인터에 `+1`을 하면 실제로는 4바이트를 이동합니다.

---

## 메모리 고정: fixed 키워드

.NET의 가비지 컬렉터는 메모리를 효율적으로 관리하기 위해 객체를 이동시킬 수 있습니다. 이는 포인터를 사용할 때 문제가 될 수 있는데, 객체가 이동하면 포인터가 유효하지 않게 되기 때문입니다. `fixed` 키워드는 객체를 고정하여 GC가 이동하지 못하게 합니다.

### 배열 고정 예제

```csharp
unsafe
{
    int[] data = { 1, 2, 3, 4, 5 };
    
    // 배열을 고정하고 포인터 얻기
    fixed (int* ptr = data)
    {
        // 고정된 동안 배열 작업
        for (int i = 0; i < data.Length; i++)
        {
            ptr[i] *= 2;  // 배열 요소 수정
        }
    }  // 고정 해제
    
    // 결과 확인
    foreach (int value in data)
    {
        Console.WriteLine(value);  // 2, 4, 6, 8, 10
    }
}
```

### 다중 포인터 고정

여러 배열을 동시에 고정할 수도 있습니다:

```csharp
unsafe
{
    byte[] source = new byte[100];
    byte[] destination = new byte[100];
    
    // 여러 배열 동시 고정
    fixed (byte* srcPtr = source, dstPtr = destination)
    {
        // 메모리 복사 작업
        Buffer.MemoryCopy(srcPtr, dstPtr, destination.Length, source.Length);
    }
}
```

### 주의사항

고정은 가비지 컬렉터의 효율성을 저하시킬 수 있으므로, 가능한 한 짧은 시간 동안만 사용해야 합니다. 특히 대형 객체를 장시간 고정하면 메모리 단편화를 유발할 수 있습니다.

---

## 스택 할당: stackalloc

`stackalloc` 키워드를 사용하면 스택에 메모리를 할당할 수 있습니다. 이는 힙 할당보다 빠르고 가비지 컬렉션의 영향을 받지 않습니다.

### 기본 사용법

```csharp
unsafe
{
    // 스택에 100개의 int 할당
    int* buffer = stackalloc int[100];
    
    for (int i = 0; i < 100; i++)
    {
        buffer[i] = i * 2;
    }
    
    // 사용 후 자동 해제 (함수 반환 시)
}
```

### Span<T>와 함께 사용

`stackalloc`은 `Span<T>`와 함께 사용하면 더 안전하고 편리합니다:

```csharp
// unsafe 문맥 없이도 사용 가능
Span<int> buffer = stackalloc int[100];

for (int i = 0; i < buffer.Length; i++)
{
    buffer[i] = i * 2;
}

// Span의 다양한 메서드 활용
buffer.Fill(0);  // 모든 요소를 0으로 설정
buffer.Reverse();  // 요소 순서 반전
```

### 제한사항

`stackalloc`으로 할당한 메모리는:
- 함수가 반환될 때 자동으로 해제됨
- 힙에 저장할 수 없음
- 매우 큰 메모리에는 적합하지 않음 (스택 오버플로우 위험)
- 기본적으로 초기화되지 않음 (명시적으로 초기화 필요)

---

## Span<T>: 안전한 메모리 슬라이스

`Span<T>`는 연속적인 메모리 영역을 안전하게 표현하는 구조체입니다. 배열, 네이티브 메모리, 스택 메모리 등 다양한 원본을 동일한 인터페이스로 다룰 수 있습니다.

### Span<T> 생성하기

```csharp
// 배열로부터 생성
int[] array = { 1, 2, 3, 4, 5 };
Span<int> spanFromArray = array.AsSpan();

// 배열의 일부분 슬라이스
Span<int> slice = array.AsSpan(1, 3);  // [2, 3, 4]

// 스택 할당과 함께 사용
Span<byte> stackBuffer = stackalloc byte[1024];

// 네이티브 메모리로부터 생성
unsafe
{
    byte* nativeMemory = (byte*)NativeMemory.Alloc(1024);
    Span<byte> nativeSpan = new Span<byte>(nativeMemory, 1024);
}
```

### ReadOnlySpan<T>

읽기 전용 버전인 `ReadOnlySpan<T>`는 데이터를 수정하지 않고 읽기만 할 때 사용합니다:

```csharp
string text = "Hello, World!";
ReadOnlySpan<char> charSpan = text.AsSpan();

// 문자열 슬라이싱 (부분 문자열 생성 없이)
ReadOnlySpan<char> hello = charSpan.Slice(0, 5);
ReadOnlySpan<char> world = charSpan.Slice(7, 5);

Console.WriteLine(hello.ToString());  // "Hello"
Console.WriteLine(world.ToString());  // "World"
```

### Span<T>의 주요 기능

```csharp
int[] data = { 10, 20, 30, 40, 50, 60, 70, 80, 90, 100 };
Span<int> span = data.AsSpan();

// 슬라이싱
Span<int> firstHalf = span[..5];      // 처음 5개 요소
Span<int> secondHalf = span[5..];     // 나머지 요소
Span<int> middle = span[2..7];        // 인덱스 2부터 7까지 (7 제외)

// 변환
Span<byte> asBytes = MemoryMarshal.AsBytes(span);

// 검색
int index = span.IndexOf(50);  // 50이 있는 인덱스 찾기
bool contains = span.Contains(30);  // 30 포함 여부

// 복사
int[] destination = new int[5];
span.Slice(2, 5).CopyTo(destination);
```

---

## ref struct와 메모리 안전성

`Span<T>`와 `ReadOnlySpan<T>`는 `ref struct` 타입입니다. 이는 특별한 제약을 가지는데, 이러한 제약은 메모리 안전성을 보장하기 위해 설계되었습니다.

### ref struct의 제약

1. **힙에 저장할 수 없음**
   ```csharp
   // 컴파일 오류: ref struct는 클래스 필드가 될 수 없음
   class Container
   {
       // Span<int> field;  // 오류!
   }
   ```

2. **박싱 불가**
   ```csharp
   // 컴파일 오류: ref struct는 object로 박싱할 수 없음
   Span<int> span = stackalloc int[10];
   // object obj = span;  // 오류!
   ```

3. **비동기 메서드에서 사용 제한**
   ```csharp
   // 컴파일 오류: ref struct는 async 메서드에서 사용할 수 없음
   async Task ProcessAsync()
   {
       // Span<int> span = stackalloc int[10];  // 오류!
   }
   ```

4. **이터레이터에서 사용 제한**
   ```csharp
   // 컴파일 오류: ref struct는 yield return과 함께 사용할 수 없음
   IEnumerable<int> GetNumbers()
   {
       // Span<int> span = stackalloc int[10];  // 오류!
       // yield return span[0];  // 오류!
   }
   ```

### 왜 이런 제약이 필요한가?

이러한 제약은 `Span<T>`가 스택에 할당된 메모리나 고정된 메모리를 참조할 수 있기 때문입니다. 만약 `Span<T>`가 힙에 저장되거나 비동기 컨텍스트에서 사용된다면, 참조하는 메모리가 유효하지 않게 될 위험이 있습니다.

### 대안: Memory<T>

힙에 저장해야 하거나 비동기 작업에서 메모리를 참조해야 할 때는 `Memory<T>`를 사용합니다:

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
    
    // Span<T>로 변환하여 작업
    Span<byte> span = buffer.Span;
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
Span<int> restoredIntSpan = MemoryMarshal.Cast<byte, int>(byteSpan);
```

### 구조체와의 상호작용

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct Point
{
    public int X;
    public int Y;
}

// 구조체 배열을 byte 배열로 보기
Point[] points = new Point[10];
Span<Point> pointSpan = points.AsSpan();
Span<byte> pointBytes = MemoryMarshal.AsBytes(pointSpan);

// 개별 구조체 접근
ref Point firstPoint = ref MemoryMarshal.GetReference(pointSpan);
firstPoint.X = 100;
firstPoint.Y = 200;
```

### 메모리 읽기/쓰기

```csharp
byte[] buffer = new byte[100];
Span<byte> span = buffer.AsSpan();

// 특정 위치에서 값 읽기
int value = MemoryMarshal.Read<int>(span.Slice(10));

// 특정 위치에 값 쓰기
MemoryMarshal.Write(span.Slice(20), ref value);

// 시퀀스 읽기
Span<int> intValues = MemoryMarshal.Cast<byte, int>(span.Slice(30));
```

---

## 실전 예제: 고성능 데이터 처리

### 이미지 데이터 처리

```csharp
public unsafe class ImageProcessor
{
    public static void ApplyGrayscale(Span<byte> imageData, int width, int height)
    {
        // 각 픽셀은 4바이트 (BGRA 형식 가정)
        int bytesPerPixel = 4;
        int stride = width * bytesPerPixel;
        
        for (int y = 0; y < height; y++)
        {
            Span<byte> row = imageData.Slice(y * stride, stride);
            
            for (int x = 0; x < width; x++)
            {
                int pixelIndex = x * bytesPerPixel;
                
                // BGR 구성 요소 추출
                byte blue = row[pixelIndex];
                byte green = row[pixelIndex + 1];
                byte red = row[pixelIndex + 2];
                
                // 그레이스케일 계산
                byte gray = (byte)((red * 0.299) + (green * 0.587) + (blue * 0.114));
                
                // 모든 채널에 동일한 값 설정
                row[pixelIndex] = gray;     // Blue
                row[pixelIndex + 1] = gray; // Green
                row[pixelIndex + 2] = gray; // Red
                // Alpha 채널은 변경하지 않음
            }
        }
    }
}
```

### 네트워크 패킷 파싱

```csharp
public static class PacketParser
{
    public static bool TryParsePacket(ReadOnlySpan<byte> data, out Packet packet)
    {
        packet = default;
        
        // 최소 패킷 크기 확인
        if (data.Length < 8) return false;
        
        // 마직막 4바이트는 체크섬
        ReadOnlySpan<byte> payload = data[..^4];
        uint expectedChecksum = BinaryPrimitives.ReadUInt32LittleEndian(data[^4..]);
        
        // 체크섬 검증
        if (CalculateChecksum(payload) != expectedChecksum)
            return false;
        
        // 패킷 헤더 파싱
        ushort packetId = BinaryPrimitives.ReadUInt16LittleEndian(payload);
        ushort dataLength = BinaryPrimitives.ReadUInt16LittleEndian(payload[2..]);
        
        // 데이터 길이 검증
        if (payload.Length < 4 + dataLength) return false;
        
        // 데이터 추출
        ReadOnlySpan<byte> packetData = payload.Slice(4, dataLength);
        
        packet = new Packet(packetId, packetData.ToArray());
        return true;
    }
    
    private static uint CalculateChecksum(ReadOnlySpan<byte> data)
    {
        uint checksum = 0;
        for (int i = 0; i < data.Length; i++)
        {
            checksum += data[i];
        }
        return checksum;
    }
}
```

### 성능 비교: 전통적 방식 vs Span<T> 방식

```csharp
public class PerformanceBenchmark
{
    // 전통적인 방식: 여러 번의 배열 할당
    public static byte[] ProcessDataTraditional(byte[] input)
    {
        // 첫 번째 변환
        byte[] temp1 = new byte[input.Length];
        Array.Copy(input, temp1, input.Length);
        Transform1(temp1);
        
        // 두 번째 변환
        byte[] temp2 = new byte[temp1.Length];
        Array.Copy(temp1, temp2, temp1.Length);
        Transform2(temp2);
        
        // 세 번째 변환
        byte[] result = new byte[temp2.Length];
        Array.Copy(temp2, result, temp2.Length);
        Transform3(result);
        
        return result;
    }
    
    // Span<T> 방식: 할당 최소화
    public static byte[] ProcessDataWithSpan(byte[] input)
    {
        byte[] result = new byte[input.Length];
        Span<byte> buffer = result;
        
        // 입력 데이터 복사
        input.AsSpan().CopyTo(buffer);
        
        // 동일한 버퍼에서 변환 수행
        Transform1(buffer);
        Transform2(buffer);
        Transform3(buffer);
        
        return result;
    }
    
    private static void Transform1(Span<byte> data)
    {
        for (int i = 0; i < data.Length; i++)
        {
            data[i] = (byte)(data[i] ^ 0xFF);  // 비트 반전
        }
    }
    
    private static void Transform2(Span<byte> data)
    {
        // 다른 변환 작업
    }
    
    private static void Transform3(Span<byte> data)
    {
        // 또 다른 변환 작업
    }
}
```

---

## 안전한 Unsafe 코드 작성 지침

### 1. 가능하면 Span<T> 사용

포인터 대신 `Span<T>`를 사용하면 경계 검사가 자동으로 이루어져 메모리 안전성이 보장됩니다:

```csharp
// 권장: Span<T> 사용
void ProcessWithSpan(Span<int> data)
{
    for (int i = 0; i < data.Length; i++)
    {
        data[i] *= 2;  // 안전한 접근
    }
}

// 비권장: 직접 포인터 사용
unsafe void ProcessWithPointer(int* data, int length)
{
    for (int i = 0; i < length; i++)
    {
        data[i] *= 2;  // 경계 검사 없음
    }
}
```

### 2. 고정 범위 최소화

`fixed` 블록은 가능한 한 작은 범위로 제한하세요:

```csharp
// 좋은 예: 최소한의 범위만 고정
void ProcessData(byte[] data)
{
    unsafe
    {
        fixed (byte* ptr = data)
        {
            // 빠른 작업만 수행
            NativeLibrary.Process(ptr, data.Length);
        }
    }
    // 고정 해제 후 추가 작업
}

// 나쁜 예: 불필요하게 긴 고정
void ProcessDataBad(byte[] data)
{
    unsafe
    {
        fixed (byte* ptr = data)
        {
            // 긴 작업 수행 (고정 상태 유지)
            Thread.Sleep(1000);  // 나쁜 예!
            NativeLibrary.Process(ptr, data.Length);
        }
    }
}
```

### 3. 스택 할당 크기 제한

`stackalloc`은 적절한 크기로 제한하세요:

```csharp
// 좋은 예: 적절한 크기
void ProcessSmallData()
{
    Span<byte> buffer = stackalloc byte[1024];  // 적절한 크기
    // 작업 수행
}

// 나쁜 예: 너무 큰 할당
void ProcessLargeData()
{
    // 위험: 스택 오버플로우 가능성
    Span<byte> buffer = stackalloc byte[1024 * 1024];  // 너무 큼!
}
```

### 4. 예외 처리

Unsafe 코드에서는 특히 예외 처리가 중요합니다:

```csharp
unsafe void ProcessWithSafety(byte[] data)
{
    if (data == null) throw new ArgumentNullException(nameof(data));
    if (data.Length == 0) return;
    
    fixed (byte* ptr = data)
    {
        try
        {
            NativeLibrary.Process(ptr, data.Length);
        }
        catch (Exception ex)
        {
            // 적절한 예외 처리
            Console.WriteLine($"처리 실패: {ex.Message}");
            throw;
        }
    }
}
```

---

## 일반적인 함정과 해결책

### 1. 댕글링 포인터(Dangling Pointer)

```csharp
// 위험한 코드
unsafe int* GetDanglingPointer()
{
    int value = 42;
    return &value;  // 지역 변수의 주소 반환 - 함수 종료 후 무효!
}

// 안전한 대안
unsafe void ProcessWithLocal()
{
    int value = 42;
    int* ptr = &value;
    // ptr은 현재 블록 내에서만 유효
    Console.WriteLine(*ptr);
}
```

### 2. 정렬 문제

```csharp
// 정렬되지 않은 접근은 플랫폼에 따라 문제를 일으킬 수 있음
unsafe void MisalignedAccess()
{
    byte[] data = new byte[10];
    fixed (byte* ptr = data)
    {
        // 잘못된 정렬로 접근 (플랫폼에 따라 크래시 가능)
        int* intPtr = (int*)(ptr + 1);  // 1바이트 오프셋
        *intPtr = 42;  // 정렬되지 않은 접근
    }
}

// 해결책: 적절한 정렬 보장
unsafe void AlignedAccess()
{
    // 정렬된 메모리 할당
    byte* aligned = (byte*)NativeMemory.AlignedAlloc(100, 16);
    try
    {
        // 16바이트 정렬된 메모리 사용
        // ...
    }
    finally
    {
        NativeMemory.AlignedFree(aligned);
    }
}
```

### 3. 엔디안 문제

```csharp
// 엔디안 문제 해결
public static ushort ReadUInt16LittleEndian(ReadOnlySpan<byte> buffer)
{
    if (buffer.Length < 2)
        throw new ArgumentException("버퍼가 너무 짧습니다", nameof(buffer));
    
    if (BitConverter.IsLittleEndian)
    {
        // 리틀 엔디안 시스템
        return (ushort)(buffer[0] | (buffer[1] << 8));
    }
    else
    {
        // 빅 엔디안 시스템
        return (ushort)((buffer[0] << 8) | buffer[1]);
    }
}
```

---

## 결론

C#의 Unsafe 코드와 `Span<T>`는 고성능 시나리오에서 강력한 도구이지만, 신중하게 사용해야 합니다. 다음 원칙을 기억하세요:

### 핵심 원칙 요약

1. **안전성 우선**: 가능하면 `Span<T>`와 `Memory<T>`를 사용하여 안전한 추상화를 활용하세요. 이들은 경계 검사와 메모리 안전성을 제공합니다.

2. **적절한 추상화 선택**: 문제에 맞는 적절한 수준의 추상화를 선택하세요:
   - 일반적인 작업: 안전한 관리 코드
   - 고성능 메모리 작업: `Span<T>`, `Memory<T>`
   - 네이티브 상호작용: 안전한 P/Invoke
   - 최후의 수단: 포인터와 Unsafe 코드

3. **메모리 수명 관리**: `fixed`, `stackalloc`, `Span<T>`의 수명을 이해하고 관리하세요. 특히 `ref struct`의 제약을 이해하고 준수하세요.

4. **성능 vs 안전성 균형**: 성능 최적화가 정말 필요한지 확인하세요. 대부분의 경우 안전한 코드로도 충분한 성능을 얻을 수 있습니다.

5. **테스트와 검증**: Unsafe 코드는 철저한 테스트와 검증이 필요합니다. 단위 테스트, 통합 테스트, 그리고 가능하다면 정적 분석 도구를 활용하세요.

6. **문서화**: Unsafe 코드는 명확한 문서화가 필요합니다. 왜 Unsafe 코드가 필요한지, 어떤 위험이 있는지, 어떻게 안전하게 사용하는지 문서로 남기세요.

### 실용적인 조언

- **점진적 접근**: 처음에는 안전한 코드로 시작하고, 프로파일링을 통해 실제 병목이 확인된 경우에만 Unsafe 코드를 도입하세요.
- **격리**: Unsafe 코드는 가능한 한 작은 모듈로 격리하고, 잘 정의된 인터페이스 뒤에 숨기세요.
- **코드 리뷰**: Unsafe 코드는 특히 신중한 코드 리뷰가 필요합니다.
- **현대적 도구 활용**: 최신 C#과 .NET의 기능(`Span<T>`, `Memory<T>`, `ref` 구조체 등)을 최대한 활용하세요.

C#은 안전한 코드와 고성능 코드 사이의 균형을 잘 잡을 수 있는 언어입니다. Unsafe 코드와 `Span<T>`는 이 균형을 유지하면서도 필요한 경우 낮은 수준의 제어를 가능하게 하는 강력한 도구입니다. 그러나 이러한 도구들은 권한과 같아서, 책임감 있게 사용할 때만 그 가치를 발휘합니다. 올바른 상황에서 적절하게 사용한다면, C#으로도 시스템 수준의 성능을 요구하는 애플리케이션을 구축하는 데 부족함이 없을 것입니다.