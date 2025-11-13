---
layout: post
title: C# - PInvoke, Marshal, NativeMemory
date: 2024-10-31 19:20:23 +0900
category: Csharp
---
# C# P/Invoke, Marshal, NativeMemory

## 목차

1. P/Invoke 개요와 호출 규약(Calling Convention)
2. 문자열/문자 인코딩 마샬링(ANSI/Unicode/UTF-8, StringBuilder)
3. 구조체/배열 마샬링(Sequential/Explicit, Pack/Align, ByValArray)
4. 포인터/버퍼와 Span<byte> 브리지
5. 예외/에러 처리(SetLastError, GetLastWin32Error)
6. 핸들 수명: `SafeHandle`/`CriticalHandle` vs `IntPtr`
7. `Marshal` 필수 API 총정리와 활용 패턴
8. .NET 6+ `NativeMemory`—저수준 메모리 API
9. 고급: 콜백/Reverse P/Invoke, 함수 포인터(delegate*/UnmanagedCallersOnly)
10. DLL 로딩 전략(NativeLibrary, DllImportSearchPath)
11. 성능/안정성 체크리스트, 안티패턴
12. 종합 예제(Win32: MessageBox, 파일핸들; C DLL과 구조체/배열 교환)

---

## 1. P/Invoke 개요와 호출 규약

### 1.1 P/Invoke란?

- 관리 코드(C#)에서 **네이티브 함수**를 **직접 호출**하는 메커니즘
- 함수 원형과 ABI를 **정확히 일치**시켜야 함(이름, 호출 규약, 인코딩, 매개변수/반환 타입)

```csharp
using System;
using System.Runtime.InteropServices;

public static class NativeMethods
{
    [DllImport("user32.dll", CharSet = CharSet.Unicode, SetLastError = true,
               CallingConvention = CallingConvention.Winapi)]
    public static extern int MessageBox(IntPtr hWnd, string text, string caption, uint type);
}

NativeMethods.MessageBox(IntPtr.Zero, "Hello", "Title", 0);
```

### 1.2 호출 규약(CallingConvention)

| 규약 | 설명 | 비고 |
|---|---|---|
| `Cdecl` | 호출자(caller)가 스택 정리 | 일반 C 라이브러리(가변 인자 함수 포함) |
| `StdCall` | 피호출자(callee)가 스택 정리 | Win32 API 전통 |
| `ThisCall` | C++ 인스턴스 메서드 | 주로 네이티브 C++ 멤버 |
| `FastCall` | 레지스터 우선 | 일부 컴파일러/플랫폼 |
| `Winapi` | OS가 기본 규약 선택 | Windows에서 보통 StdCall |

규약 불일치 → **스택 손상/크래시**. C 헤더/문서로 **반드시 확인**.

---

## 2. 문자열/문자 인코딩 마샬링

### 2.1 CharSet

- `CharSet.Ansi` : `char*`를 ANSI(현재 코드페이지)로 마샬
- `CharSet.Unicode` : `wchar_t*`(UTF-16)로 마샬(Windows 기본 권장)
- `CharSet.Auto` : 플랫폼 기본(Win=Unicode)
- .NET 5+ : **UTF-8** 전용 API 지원(아래 참고)

### 2.2 문자열 인수/반환

```csharp
// in string (ANSI)
[DllImport("mylib.dll", CharSet = CharSet.Ansi)]
public static extern void print_message(string msg);

// in string (UTF-16)
[DllImport("mylib.dll", CharSet = CharSet.Unicode)]
public static extern void wprint_message(string msg);

// out char* (소유권: 네이티브)
[DllImport("mylib.dll", CharSet = CharSet.Ansi)]
public static extern IntPtr get_message_ansi();

string s = Marshal.PtrToStringAnsi(get_message_ansi());

// out wchar_t* (UTF-16)
[DllImport("mylib.dll", CharSet = CharSet.Unicode)]
public static extern IntPtr get_message_w();

string ws = Marshal.PtrToStringUni(get_message_w());
```

### 2.3 UTF-8 도우미

```csharp
[DllImport("mylib.dll")]
static extern IntPtr get_message_utf8(); // char* (UTF-8)

string utf8 = Marshal.PtrToStringUTF8(get_message_utf8());
```

### 2.4 `StringBuilder`—출력 버퍼(Caller-allocated)

네이티브가 **호출자가 제공한 버퍼**에 쓰는 패턴:

```csharp
[DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
static extern int GetModuleFileNameW(IntPtr hModule, System.Text.StringBuilder lpFilename, int nSize);

var sb = new System.Text.StringBuilder(260);
int len = GetModuleFileNameW(IntPtr.Zero, sb, sb.Capacity);
string path = sb.ToString();
```

> 길이/버퍼 크기 엄수. 반환 값으로 **오류** / **잘림** 감지.

---

## 3. 구조체/배열 마샬링

### 3.1 레이아웃과 패킹

```csharp
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct MyStruct
{
    public int id;
    public float value;
}
```

- `Sequential` : 필드 순서대로 배치
- `Explicit` + `FieldOffset` : 수동 오프셋 지정(비트필드/유니온)
- `Pack` : 정렬 단위(기본 8). C 측 `#pragma pack`과 맞추기

### 3.2 구조체 내 고정 배열(ByValArray/SizeConst)

C:
```c
typedef struct {
    int len;
    unsigned char data[16];
} Packet;
```

C#:
```csharp
[StructLayout(LayoutKind.Sequential)]
struct Packet
{
    public int len;
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 16)]
    public byte[] data;
}
```

> *주의*: `ByValArray`는 **구조체 자체 크기**를 늘린다. C와 **정확히 일치**해야 함.

### 3.3 배열 파라미터

```csharp
// C: void Process(int* arr, int size);
[DllImport("mylib.dll")]
public static extern void Process([In, Out] int[] arr, int size);
```

- `[In]` : 읽기용
- `[Out]` : 쓰기 결과 반환
- `[In, Out]` : 양방향

성능/고급: **고정/포인터** + `Span<T>` 활용(§4 참조)

---

## 4. 포인터/버퍼와 `Span<byte>` 브리지

네이티브 버퍼 ↔ 관리 코드 간 **무복사/안전한 경계 검사**를 위해 `Span<T>`를 적극 사용합니다.

```csharp
unsafe
{
    IntPtr p = Marshal.AllocHGlobal(1024);
    try
    {
        var span = new Span<byte>((void*)p, 1024);
        span.Clear();
        span[0] = 0x7F;
    }
    finally { Marshal.FreeHGlobal(p); }
}
```

배열(관리 메모리)을 네이티브에 넘길 때 **핀 고정**:

```csharp
unsafe
{
    byte[] buf = new byte[4096];
    fixed (byte* p = buf) // GC 이동 방지
    {
        // p를 네이티브에 전달
    } // 해제
}
```

---

## 5. 에러/예외 처리—`SetLastError`, `GetLastWin32Error`

```csharp
[DllImport("kernel32.dll", SetLastError = true)]
static extern bool CloseHandle(IntPtr hObject);

if (!CloseHandle(h))
{
    int err = Marshal.GetLastWin32Error();
    // err 로깅/예외 변환
}
```

- P/Invoke 선언에 `SetLastError=true` 필요
- Win32 API와 함께 쓰는 표준 패턴
- POSIX는 `errno` 직접 다루는 별도 패턴 필요

---

## 6. 핸들 수명: `SafeHandle` 권장

리소스 누수/이중 해제를 방지하려면 **`IntPtr` 대신 `SafeHandle`**을 쓰세요.

```csharp
using System;
using System.Runtime.InteropServices;

sealed class SafeFileHandleEx : SafeHandle
{
    public SafeFileHandleEx() : base(IntPtr.Zero, ownsHandle: true) { }
    public override bool IsInvalid => handle == IntPtr.Zero || handle == new IntPtr(-1);

    [DllImport("kernel32.dll", SetLastError = true)]
    static extern bool CloseHandle(IntPtr h);

    protected override bool ReleaseHandle() => CloseHandle(handle);
}

// 예시: CreateFile → SafeHandle 반환
[DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
static extern SafeFileHandleEx CreateFileW(
    string lpFileName, uint dwDesiredAccess, uint dwShareMode,
    IntPtr lpSecurityAttributes, uint dwCreationDisposition,
    uint dwFlagsAndAttributes, IntPtr hTemplateFile);
```

- `using`/`Dispose`로 자동 해제
- 예외 경로에서도 **누수 최소화**

---

## 7. `Marshal` 필수 API 정리

| API | 용도 | 예시 |
|---|---|---|
| `AllocHGlobal`/`FreeHGlobal` | 힙 블록 할당/해제 | C의 `malloc/free` 유사 |
| `StructureToPtr` | 구조체 → 포인터 | 네이티브 버퍼에 복사 |
| `PtrToStructure<T>` | 포인터 → 구조체 | 역마샬링 |
| `Copy` | 배열↔포인터 복사 | `Marshal.Copy(src, 0, dstPtr, len)` |
| `Read/WriteByte/Int32` | 원시 읽기/쓰기 | 포인터 오프셋 접근 |
| `StringToHGlobalUni/Ansi` | string → 네이티브 복사 | 종료 후 `FreeHGlobal` |
| `PtrToStringUni/Ansi/UTF8` | 네이티브 문자열 → C# | 소유권 정책 주의 |
| `GetLastWin32Error` | Win32 오류코드 | `SetLastError=true` 필요 |

### 예: 구조체 → 네이티브 버퍼

```csharp
[StructLayout(LayoutKind.Sequential)]
struct Header { public int len; public int type; }

IntPtr p = Marshal.AllocHGlobal(Marshal.SizeOf<Header>());
try
{
    var h = new Header { len = 128, type = 7 };
    Marshal.StructureToPtr(h, p, fDeleteOld: false);
    // p 전달
}
finally { Marshal.FreeHGlobal(p); }
```

---

## 8. .NET 6+ `NativeMemory`—더 간결한 저수준 API

```csharp
using System.Runtime.InteropServices;

nint p = NativeMemory.Alloc(100);      // 100바이트
NativeMemory.Fill((void*)p, 100, 0x00); // memset
NativeMemory.Free(p);
```

- `nint`/`nuint` : 포인터 크기 정수
- `AllocZeroed`, `Realloc`, `Copy`, `Move`, `Clear` 등 제공
- Marshaling 없이 **순수 바이트 처리**에 최적

### `Span<T>` 브리지

```csharp
nint p = NativeMemory.Alloc(256);
try
{
    Span<byte> buf = new((void*)p, 256);
    buf[0] = 0x42;
}
finally { NativeMemory.Free(p); }
```

---

## 9. 고급: 콜백/Reverse P/Invoke, 함수 포인터

### 9.1 콜백 델리게이트 마샬링

C:
```c
typedef void (*on_event_t)(int code);

__declspec(dllexport) void register_callback(on_event_t cb);
```

C#:
```csharp
[UnmanagedFunctionPointer(CallingConvention.Cdecl)]
public delegate void OnEvent(int code);

[DllImport("mylib", CallingConvention = CallingConvention.Cdecl)]
static extern void register_callback(OnEvent cb);

// 사용: 델리게이트 수명 유지(가비지 수집 방지)
OnEvent _callback = code => Console.WriteLine($"code={code}");
register_callback(_callback);
```

> 델리게이트가 **GC에 수거되면** 네이티브가 콜백 호출 시 크래시. **루트 보관 필수**.

### 9.2 함수 포인터(.NET 5+) + Reverse P/Invoke

```csharp
using System.Runtime.InteropServices;

[UnmanagedCallersOnly(CallConvs = new[] { typeof(CallConvCdecl) })]
public static void OnNative(int code) { /* ... */ }

// 함수 포인터 얻기
static unsafe delegate* unmanaged[Cdecl]<int, void> GetPtr()
{
    return (delegate* unmanaged[Cdecl]<int, void>)
        &OnNative;
}
```

C 측에 **함수 주소** 전달 → 오버헤드↓, 안전성↑(마샬링 제거). 단, **제약**과 **AOT/플랫폼** 이슈 확인.

---

## 10. DLL 로딩 전략

### 10.1 정적 바인딩—`DllImport`

간결하지만, 로딩/탐색 경로는 OS 기본 규칙에 따름.

- Windows: Side-by-side, PATH, 앱 폴더 등
- `.NET 5+`: `DefaultDllImportSearchPaths`로 제어 가능

```csharp
[DllImport("mylib")]
[DefaultDllImportSearchPaths(DllImportSearchPath.AssemblyDirectory)]
static extern int foo();
```

### 10.2 동적 로딩—`NativeLibrary`

```csharp
using System.Runtime.InteropServices;

nint h = NativeLibrary.Load("mylib");
try
{
    nint sym = NativeLibrary.GetExport(h, "add");
    var add = Marshal.GetDelegateForFunctionPointer<AddFn>(sym);
    int r = add(3, 4);
}
finally { NativeLibrary.Free(h); }

[UnmanagedFunctionPointer(CallingConvention.Cdecl)]
delegate int AddFn(int a, int b);
```

---

## 11. 성능/안정성 체크리스트 & 안티패턴

### 체크리스트
- [ ] C 헤더/문서 기준으로 **시그니처 1:1 대응**(정확한 타입·규약·인코딩)
- [ ] 구조체 레이아웃/패킹/정렬 **일치**
- [ ] 문자열/버퍼 길이/널 종료 **계약 준수**
- [ ] `SetLastError=true` + `GetLastWin32Error()`로 오류 처리
- [ ] 핸들은 `SafeHandle`로 수명 안전화, `Dispose` 보장
- [ ] 델리게이트 콜백 **루트 유지**(GC 수거 방지)
- [ ] 무분별한 핀 고정/LOH 핀닝 지양(단편화·GC 저하)
- [ ] **필요한 곳만** `unsafe` 사용 → `Span<T>` 우선

### 안티패턴
- `IntPtr` 핸들을 임의 해제/이중 해제
- `throw;` 대신 `throw ex;` 재던지기(스택 추적 손실)
- `CharSet.Ansi` 남발(국제화 문제), 예상 못한 로케일 의존
- 구조체에 **관리형 필드** 포함(예: string) → blittable 아님, 비효율/불안정

---

## 12. 종합 예제

### 12.1 Win32: MessageBox (기본기)

```csharp
[DllImport("user32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
static extern int MessageBoxW(IntPtr hWnd, string text, string caption, uint type);

_ = MessageBoxW(IntPtr.Zero, "안녕하세요", "제목", 0);
```

### 12.2 파일 핸들 얻기 + `SafeHandle`

```csharp
using System.Runtime.InteropServices;
using Microsoft.Win32.SafeHandles;

[DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
static extern SafeFileHandle CreateFileW(
    string name, uint access, uint share, IntPtr sec, uint disp, uint flags, IntPtr tmpl);

const uint GENERIC_READ = 0x80000000;
const uint OPEN_EXISTING = 3;

using SafeFileHandle h = CreateFileW(@"C:\Windows\notepad.exe", GENERIC_READ, 0, IntPtr.Zero, OPEN_EXISTING, 0, IntPtr.Zero);
if (h.IsInvalid)
{
    int err = Marshal.GetLastWin32Error();
    throw new System.ComponentModel.Win32Exception(err);
}
```

### 12.3 C DLL과 구조체/배열 교환

**C (mylib.c):**
```c
#include <stdint.h>
#ifdef _WIN32
#define API __declspec(dllexport)
#else
#define API
#endif

#pragma pack(push, 1)
typedef struct {
    int32_t id;
    float value;
    uint8_t data[16];
} record_t;
#pragma pack(pop)

API int process(record_t* rec, int count) {
    if (!rec || count <= 0) return -1;
    for (int i = 0; i < count; ++i) {
        rec[i].value *= 2.0f;
        rec[i].data[0] = (uint8_t)(rec[i].id & 0xFF);
    }
    return count;
}
```

**C# (Interop):**
```csharp
using System;
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct Record
{
    public int id;
    public float value;

    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 16)]
    public byte[] data;
}

static class Native
{
    [DllImport("mylib", CallingConvention = CallingConvention.Cdecl)]
    public static extern int process([In, Out] Record[] rec, int count);
}

var arr = new Record[2];
arr[0] = new Record { id = 1, value = 3.5f, data = new byte[16] };
arr[1] = new Record { id = 2, value = 1.25f, data = new byte[16] };

int n = Native.process(arr, arr.Length);
if (n < 0) throw new InvalidOperationException("native failed");

Console.WriteLine($"{arr[0].value}, {arr[0].data[0]}"); // 7.0, 1
Console.WriteLine($"{arr[1].value}, {arr[1].data[0]}"); // 2.5, 2
```

### 12.4 버퍼 직접 넘기기—핀 고정 + Span

C:
```c
API int transform(const uint8_t* in, int len, uint8_t* out, int cap) {
    if (!in || !out || len > cap) return -1;
    for (int i = 0; i < len; ++i) out[i] = (uint8_t)(in[i] ^ 0xAA);
    return len;
}
```

C#:
```csharp
using System.Runtime.InteropServices;

static class N
{
    [DllImport("mylib", CallingConvention = CallingConvention.Cdecl)]
    public static extern int transform(byte* input, int len, byte* output, int cap);
}

public static bool TryTransform(ReadOnlySpan<byte> src, Span<byte> dst, out int written)
{
    written = 0;
    if (dst.Length < src.Length) return false;
    unsafe
    {
        fixed (byte* ps = src)
        fixed (byte* pd = dst)
        {
            int r = N.transform(ps, src.Length, pd, dst.Length);
            if (r < 0) return false;
            written = r;
            return true;
        }
    }
}
```

---

## (보너스) 비용 모델 직관

네이티브 호출은 일반 메서드 호출보다 **상수 오버헤드**가 큽니다. 대략:

\[
T_{\text{call}} \approx \alpha_{\text{interop}} + \beta \cdot N_{\text{marshal}}
\]

- \(\alpha_{\text{interop}}\): 호출/컨텍스트 전환 비용(수십~수백 ns+)
- \(N_{\text{marshal}}\): 마샬링되는 필드/버퍼 수
- 대용량 버퍼는 **빈도↓, 단위 크기↑**로 묶어 보내고, 반복 호출을 줄이는 게 유리

---

## 결론

- **정확한 ABI 일치**(규약/레이아웃/인코딩)가 생명
- 핸들은 **SafeHandle**, 버퍼는 **Span/핀 고정 최소화**
- 문자열은 **UTF-16(Windows 기본)**, 필요 시 **UTF-8** 수동 변환
- 저수준 메모리는 **NativeMemory**로 간결/고성능 처리
- 콜백은 **루트 보관**(GC 방지) + 필요 시 `UnmanagedCallersOnly`/함수 포인터
