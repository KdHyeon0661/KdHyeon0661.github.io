---
layout: post
title: C# - Unsafe 코드, Span
date: 2024-10-30 19:20:23 +0900
category: Csharp
---
# C# 포인터와 Unsafe 코드, Span<T> 기초 정리

C#은 기본적으로 **타입 안전성과 GC 기반 메모리 관리**를 제공합니다. 그럼에도 다음과 같은 상황에서는 **네이티브 수준의 메모리 접근**이 필요합니다.

- P/Invoke로 **네이티브 API**와 상호작용
- **복사/할당 비용**을 최소화해야 하는 고성능 버퍼 처리
- **이미지/바이너리** 포맷을 직접 파싱/생성
- **대량 연산**에서 GC 압력을 줄이기 위한 **스택 기반 임시 버퍼**

이 글은 다음 순서로 **실전 위주**로 정리합니다.

- `unsafe`/포인터 문법과 안전 수칙
- `fixed`로 GC 이동 방지(핀 고정)
- `stackalloc` + `Span<T>`로 **무할당 임시 버퍼**
- `Span<T>`/`ReadOnlySpan<T>`/`MemoryMarshal` 기본기
- `ref struct` 제약 및 올바른 사용 패턴
- P/Invoke/버퍼 조작에서의 **실전 레시피**, 성능/안전 체크리스트

---

## 0) 빌드/프로젝트 설정

### `/unsafe` 활성화
- **CLI**: `dotnet build -p:AllowUnsafeBlocks=true`
- **.csproj** 예시:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  </PropertyGroup>
</Project>
```

---

## 1) `unsafe` 키워드와 포인터 기초

`unsafe` 블록에서만 **포인터**, `*`, `&`, `->` 같은 연산을 사용할 수 있습니다.

```csharp
unsafe
{
    int x = 10;
    int* p = &x;             // 주소 취득
    Console.WriteLine(*p);   // 역참조: 10
    *p = 42;                 // 쓰기
    Console.WriteLine(x);    // 42
}
```

### 핵심 문법 요약

| 연산자 | 의미 | 예시 |
|---|---|---|
| `*T` | T에 대한 포인터 타입 | `int* p` |
| `&x` | x의 주소 | `int* p = &x` |
| `*p` | 역참조 | `int v = *p` |
| `p + n` | 포인터 산술(요소 단위) | `*(p + 2)` |
| `->` | 구조체 멤버 접근 | `pt->X` |

> 포인터는 **타입 안전을 우회**합니다. 크래시/메모리 손상 가능 → **최소 범위**, **검증 철저**, **테스트 필수**.

---

## 2) `fixed`—GC로부터 메모리 고정(Pinning)

GC는 힙 객체를 **이동(compact)**할 수 있습니다. 포인터로 **관리 객체**(배열/문자열/관리 구조체)에 접근하려면 그 주소를 **고정**해야 합니다.

```csharp
unsafe
{
    int[] nums = { 1, 2, 3, 4 };
    fixed (int* p = nums)           // nums 핀 고정
    {
        for (int i = 0; i < nums.Length; i++)
            Console.WriteLine(p[i]);
    } // 고정 해제
}
```

### 문자열 고정

```csharp
unsafe
{
    string s = "Hello";
    fixed (char* ps = s)           // UTF-16 char*
    {
        for (int i = 0; i < s.Length; i++)
            Console.Write(ps[i]);
    }
}
```

> **핀 고정은 GC 효율 저하**(컴팩션 방해). **짧고 국소적으로** 사용하세요.

---

## 3) `stackalloc`—스택 메모리 직접 할당

스택에 **고정 크기 배열**을 즉시 할당합니다. 힙 할당/GC 부담이 없고, **스코프 종료 시 자동 해제**됩니다.

### 기본

```csharp
Span<int> buf = stackalloc int[5]; // 스택에 20바이트 (int 4B 기준)
for (int i = 0; i < buf.Length; i++) buf[i] = i;
foreach (var v in buf) Console.Write($"{v} "); // 0 1 2 3 4
```

### 바이트 버퍼

```csharp
Span<byte> bytes = stackalloc byte[16];
bytes.Clear();
bytes[0] = 0x42;
```

> `stackalloc` 크기는 **합리적인 상한**을 두세요(수 KB 내). 너무 크면 스택 오버플로 위험.

---

## 4) `Span<T>`—안전한 메모리 슬라이스

`Span<T>`는 **연속 메모리**(배열/포인터/스택 버퍼)를 **슬라이스**처럼 다루게 해줍니다.

```csharp
Span<byte> buf = stackalloc byte[8];
buf[0] = 1; buf[1] = 2; buf[2] = 3;

Span<byte> head = buf[..2];     // 1, 2
Span<byte> tail = buf[2..];     // 3, ...
```

### 주요 특징

- **경계 검사**로 오버런 방지
- **복사 없이** 슬라이스/부분 뷰
- `ref struct`이므로 **힙에 저장/박싱 불가**, **비동기/이터레이터 상태머신에서 사용 불가**

### `ReadOnlySpan<T>`

읽기 전용 뷰:

```csharp
ReadOnlySpan<char> ro = "Hello".AsSpan();
Console.WriteLine(ro[1]);  // 'e'
```

---

## 5) `ref struct`와 제약

`Span<T>` 자체가 `ref struct`입니다. `ref struct`는 **스택 전용** 타입으로 다음 제약이 있습니다.

| 제약 | 이유 |
|---|---|
| 힙에 저장 불가(클래스 필드/박싱 금지) | GC가 이동/수명 제어 불가 |
| `async`/`yield` 메서드에서 사용 금지 | 상태머신은 힙에 배치 |
| 인터페이스 구현/캐스팅 제한 | 박싱 필요 |

### 패턴

- `Span<T>`를 **메서드 인자/지역 변수**로만 다룬다.
- 길게 유지하지 말고 **즉시 처리 후 반환**.

---

## 6) 포인터 + `Span<T>` 브리지

비관리 주소를 **안전하게** 다루려면 포인터로부터 `Span<T>`를 구성합니다.

```csharp
using System.Runtime.InteropServices;

unsafe
{
    byte* p = (byte*)Marshal.AllocHGlobal(32);
    try
    {
        var span = new Span<byte>(p, 32);
        span.Fill(0xAA);
        // 안전한 범위 검사 혜택
    }
    finally
    {
        Marshal.FreeHGlobal((IntPtr)p);
    }
}
```

> 가능하면 **직접 포인터 연산**보다 **`Span<T>` 조작**을 우선하세요.

---

## 7) `MemoryMarshal` 기본기

`MemoryMarshal`은 `Span<T>`로 **저수준 변환**을 도와줍니다.

```csharp
using System.Runtime.InteropServices;

Span<int> ints = stackalloc int[] { 1, 2, 3, 4 };
Span<byte> bytes = MemoryMarshal.AsBytes(ints); // int 뷰 → byte 뷰

// 구조체 뷰 만들기
[StructLayout(LayoutKind.Sequential)]
struct Pixel { public byte R, G, B, A; }

Span<Pixel> px = MemoryMarshal.Cast<byte, Pixel>(bytes); // 길이/정렬 주의
```

- 구조체는 **blittable**이어야 안전(단순 POD).
- 필드 정렬/패딩은 ABI와 일치해야 함.

---

## 8) 문자열/인코딩 with `stackalloc` + `Span<byte>`

UTF-8로 무할당 인코딩(작은 문자열) 예:

```csharp
using System.Text;

ReadOnlySpan<char> src = "안녕";
int max = Encoding.UTF8.GetMaxByteCount(src.Length); // 상한
Span<byte> buf = max <= 128 ? stackalloc byte[128] : new byte[max]; // 소형은 스택, 대형은 힙
int written = Encoding.UTF8.GetBytes(src, buf);
Span<byte> payload = buf[..written];
// payload를 소켓/파일로 바로 쓰기
```

> **소형 버퍼는 `stackalloc`**, **대형은 힙/풀(ArrayPool)**로 폴백.

---

## 9) 실전 레시피

### 9.1 P/Invoke—버퍼를 포인터로 넘기기

```csharp
using System.Runtime.InteropServices;

class Native
{
    [DllImport("mylib", CallingConvention = CallingConvention.Cdecl)]
    public static extern int process_buffer(byte* src, int len, byte* dst, int cap);
}

public static bool TryProcess(ReadOnlySpan<byte> src, Span<byte> dst, out int written)
{
    written = 0;
    unsafe
    {
        fixed (byte* ps = src)
        fixed (byte* pd = dst)
        {
            int r = Native.process_buffer(ps, src.Length, pd, dst.Length);
            if (r < 0 || r > dst.Length) return false;
            written = r;
            return true;
        }
    }
}
```

- **핀 고정 범위를 최소화**하고, **Try-패턴**으로 실패 안전화.

### 9.2 바이너리 패킷 파서(무복사 슬라이싱)

```csharp
public static bool TryParseHeader(ReadOnlySpan<byte> data, out ushort magic, out int length)
{
    magic = 0; length = 0;
    if (data.Length < 6) return false;
    magic = (ushort)(data[0] | (data[1] << 8)); // LE
    length = data[2] | (data[3] << 8) | (data[4] << 16) | (data[5] << 24);
    return true;
}
```

- **복사 없이** 필요한 필드만 읽는다.
- 필요 시 `BinaryPrimitives`(endianness 도우미) 사용 가능.

### 9.3 `stackalloc` 임시 포맷 버퍼

```csharp
Span<char> tmp = stackalloc char[64];
if (DateTime.UtcNow.TryFormat(tmp, out int chars, "O")) // ISO 8601
{
    var slice = tmp[..chars];
    Console.WriteLine(slice.ToString());
}
```

---

## 10) 성능 직관과 수식

데이터 복사에 드는 시간은 대략

\[
T \approx \alpha + \beta \cdot (N \cdot S)
\]

- \(N\): 복사 횟수, \(S\): 평균 복사 크기
- \(\alpha\): 호출 오버헤드, \(\beta\): 메모리 대역폭 계수

**전략**
- 복사 대신 **슬라이스/참조** 전달
- **소형 버퍼는 스택**(stackalloc), **대형은 풀/스트림**
- **핀 고정 최소화**, 포인터 범위 최소화

---

## 11) 안전 수칙 & 안티패턴

### 반드시 지킬 것
- 포인터/핀 고정을 **짧고 명확하게** 유지
- 포인터 산술 시 **경계 검사**(길이/인덱스) 확실히
- `Span<T>`/`ReadOnlySpan<T>`를 **우선 사용**
- 구조체 레이아웃은 **`StructLayout`**로 네이티브와 일치

### 피할 것
- 긴 시간 핀 고정(대형 객체 핀 고정은 **LOH 단편화** 유발)
- `unsafe` 블록 남발(리뷰/테스트 어려움)
- `Span<T>`를 필드로 보관(불가). 대신 **즉시 처리**.

---

## 12) 미니 실습: 메모리 스캔(바이트 패턴 찾기)

```csharp
public static int IndexOf(ReadOnlySpan<byte> haystack, ReadOnlySpan<byte> needle)
{
    if (needle.IsEmpty) return 0;
    if (needle.Length > haystack.Length) return -1;

    int last = haystack.Length - needle.Length;
    for (int i = 0; i <= last; i++)
        if (haystack.Slice(i, needle.Length).SequenceEqual(needle))
            return i;
    return -1;
}

// 사용
ReadOnlySpan<byte> hs = stackalloc byte[] {1,2,3,4,2,3,5};
ReadOnlySpan<byte> nd = stackalloc byte[] {2,3};
Console.WriteLine(IndexOf(hs, nd)); // 1
```

- **무할당**, **범위 검사** 자동.
- 더 빠른 구현은 `SpanHelpers`/SIMD(System.Numerics) 고려.

---

## 13) 고급: `Buffer.MemoryCopy`와 포인터 블록 복사

아주 낮은 레벨의 **빠른 메모리 복사**:

```csharp
unsafe static void CopyBlock(byte* src, byte* dst, nuint count)
{
    Buffer.MemoryCopy(src, dst, count, count);
}
```

> 일반적으로는 **`Span<T>.CopyTo`** 또는 **`stream.CopyTo`**가 안전/간결.

---

## 14) 자주 하는 질문(FAQ)

**Q1. 언제 포인터를 써야 하나?**  
A. P/Invoke/드라이버/특수한 고성능 버퍼 조작처럼 **다른 대안(Span/Stream/ArrayPool)**으로 커버되지 않을 때.

**Q2. `Span<T>`를 클래스 필드로 둘 수 있나?**  
A. 불가(`ref struct`). 대신 **생성자 파라미터로 받아 즉시 처리**.

**Q3. 문자열 처리에서 `stackalloc`이 항상 빠른가?**  
A. **소형 문자열**에 한해 유리. 길거나 가변적이면 **힙/풀**을 사용하고, **할당 횟수**를 줄이는 게 더 중요.

**Q4. 핀 고정은 얼마나 짧아야 하나?**  
A. 네이티브 호출 전후 등 **필요한 최소 범위**만. 루프 전체를 고정하는 대신, **청크 단위**로 쪼개는 것도 방법.

---

## 15) 체크리스트

- [ ] `unsafe` 범위는 최소/명확?
- [ ] 배열/문자열 포인터 접근 시 **`fixed`**로 핀 고정했는가?
- [ ] `stackalloc` 크기는 합리적(수 KB)인가?
- [ ] 가능하면 **`Span<T>`/`ReadOnlySpan<T>`** 우선 사용했는가?
- [ ] 구조체/버퍼 레이아웃이 네이티브와 일치하는가?
- [ ] 예외/오류 경로에서 **누수/댕글링 포인터**가 없는가?

---

## 16) 요약

| 개념 | 핵심 |
|---|---|
| `unsafe`/포인터 | 최저 레벨 제어. 위험/성능 둘 다 큼 |
| `fixed` | GC 이동 방지. **짧게** 사용 |
| `stackalloc` | 스택 임시 버퍼. 소형·단명 데이터에 적합 |
| `Span<T>` | 무복사 슬라이스/경계 검사/안전성 |
| `ref struct` | 스택 전용. async/필드/박싱 금지 |
| 실전 원칙 | **Span 우선**, 포인터 최소화, 핀 고정 최소화, 복사 제거 |

---

## 17) 추가 예제 모음

### 17.1 `Span<T>.TryCopyTo`로 안전 복사

```csharp
Span<byte> dst = stackalloc byte[4];
ReadOnlySpan<byte> src = new byte[] {1,2,3,4,5}; // 길이 5
if (!src.TryCopyTo(dst))
{
    // 공간 부족 처리
}
```

### 17.2 바이트 → 정수 변환(엔디언 주의)

```csharp
using System.Buffers.Binary;

ReadOnlySpan<byte> d = new byte[] { 0x78, 0x56, 0x34, 0x12 };
int le = BinaryPrimitives.ReadInt32LittleEndian(d); // 0x12345678
```

### 17.3 포인터로 구조체 접근

```csharp
unsafe struct Header { public int Len; public int Type; }

unsafe
{
    byte* p = (byte*)stackalloc byte[8];
    Header* h = (Header*)p;
    h->Len = 1024;
    h->Type = 7;
}
```

> 포인터 캐스팅은 **정렬/패딩**을 확실히 알고 있을 때만.

---

## 18) 마무리

- 가능하면 **고수준 API(Span/Stream/UTF-8 인코더)**로 풀고, **정말 필요한 부분**에 한해 `unsafe`/포인터를 도입하세요.
- 성능은 수치로 판단하세요. 작은 최적화로도 GC 압력과 복사량을 크게 줄일 수 있습니다.
- 포인터를 쓰는 순간, **안전성/검증/테스트**의 책임은 전적으로 **우리의 코드**에게 있습니다.
