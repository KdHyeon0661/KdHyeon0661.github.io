---
layout: post
title: C# - PInvoke, Marshal, NativeMemory
date: 2024-10-31 19:20:23 +0900
category: Csharp
---
# C# 네이티브 상호 운용성

C#과 .NET은 풍부한 기능을 제공하지만, 때로는 운영 체제 API, 하드웨어 제어, 고성능 C/C++ 라이브러리와 직접 통신해야 할 필요가 있습니다. P/Invoke(Platform Invocation Services)는 이러한 경계를 넘나들 수 있게 해주는 강력한 도구입니다.

---

## P/Invoke 기본

### DllImport 특성

네이티브 함수를 호출하려면 해당 함수의 서명을 C# 메서드로 선언하고 `DllImport` 특성을 붙입니다.

```csharp
[DllImport("user32.dll", CharSet = CharSet.Unicode)]
public static extern int MessageBox(IntPtr hWnd, string text, string caption, uint type);

// 사용
MessageBox(IntPtr.Zero, "안녕하세요", "인사", 0);
```

`DllImport` 특성은 네이티브 라이브러리 이름, 문자 인코딩, 호출 규약 등을 지정합니다.

### 호출 규약 (Calling Convention)

함수가 어떻게 호출되는지 정의합니다. 잘못된 규약을 사용하면 스택이 손상되어 충돌할 수 있습니다.

| 호출 규약 | 설명 | 사용 예 |
|-----------|------|---------|
| `CallingConvention.Winapi` | Windows API 기본 (대부분 `Stdcall`) | Windows 함수 |
| `CallingConvention.Cdecl` | C/C++ 기본, 가변 인자 함수에 필수 | C 표준 라이브러리 |
| `CallingConvention.StdCall` | Windows API 전통적 | `kernel32`, `user32` |

```csharp
[DllImport("msvcrt.dll", CallingConvention = CallingConvention.Cdecl)]
public static extern int printf(string format, __arglist);
```

---

## 문자열 마샬링

### 문자 인코딩

네이티브 함수는 ANSI, Unicode, UTF-8 등 다양한 인코딩을 사용합니다. `CharSet`으로 지정합니다.

```csharp
// ANSI (Windows 코드 페이지)
[DllImport("user32.dll", CharSet = CharSet.Ansi)]
public static extern int MessageBoxA(IntPtr hWnd, string text, string caption, uint type);

// Unicode (UTF-16)
[DllImport("user32.dll", CharSet = CharSet.Unicode)]
public static extern int MessageBoxW(IntPtr hWnd, string text, string caption, uint type);

// 플랫폼에 따라 자동 선택 (Windows: Unicode, Linux: ANSI)
[DllImport("user32.dll", CharSet = CharSet.Auto)]
public static extern int MessageBox(IntPtr hWnd, string text, string caption, uint type);
```

### StringBuilder를 이용한 출력 버퍼

네이티브 함수가 문자열을 버퍼에 쓸 때는 `StringBuilder`를 사용합니다.

```csharp
[DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
public static extern int GetCurrentDirectoryW(int nBufferLength, StringBuilder lpBuffer);

public string GetCurrentDirectory()
{
    StringBuilder sb = new StringBuilder(260);
    int length = GetCurrentDirectoryW(sb.Capacity, sb);
    if (length == 0)
        throw new System.ComponentModel.Win32Exception(Marshal.GetLastWin32Error());
    return sb.ToString(0, length);
}
```

### 네이티브 문자열 반환 처리

네이티브 측에서 할당한 문자열은 호출자가 해제해야 할 수 있습니다.

```csharp
[DllImport("mylib.dll", CharSet = CharSet.Ansi)]
public static extern IntPtr GetMessage();

[DllImport("mylib.dll", CharSet = CharSet.Ansi)]
public static extern void FreeMessage(IntPtr message);

public string GetAndFreeMessage()
{
    IntPtr ptr = GetMessage();
    try
    {
        return Marshal.PtrToStringAnsi(ptr);
    }
    finally
    {
        FreeMessage(ptr);
    }
}
```

---

## 구조체와 배열 마샬링

### 구조체 레이아웃

`StructLayout`을 사용하여 네이티브 구조체와 정확히 맞춥니다.

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct DeviceInfo
{
    public int DeviceId;
    public uint Flags;
    public long Timestamp;
    
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 16)]
    public byte[] SerialNumber;
}
```

- `LayoutKind.Sequential`: 필드 순서대로 메모리 배치
- `Pack`: 정렬 크기 (1, 2, 4, 8 등)
- `LayoutKind.Explicit`: 공용체(union) 구현 시 사용

### 배열 전달

```csharp
// 입력 배열
[DllImport("mylib.dll")]
public static extern double CalculateAverage(
    [MarshalAs(UnmanagedType.LPArray)] double[] values,
    int count);

// 출력 배열
[DllImport("mylib.dll")]
public static extern void GenerateRandomNumbers(
    [Out] int[] buffer, int count);
```

### unsafe 포인터를 통한 고성능 접근

대량 데이터 처리 시 `unsafe`와 `fixed`를 사용할 수 있습니다.

```csharp
[DllImport("mylib.dll", CallingConvention = CallingConvention.Cdecl)]
public static extern unsafe void ProcessBuffer(byte* buffer, int length);

public unsafe void Process(byte[] data)
{
    fixed (byte* ptr = data)
    {
        ProcessBuffer(ptr, data.Length);
    }
}
```

---

## SafeHandle: 리소스 관리의 표준

네이티브 리소스(파일 핸들, 뮤텍스, 메모리 등)는 반드시 적절히 해제해야 합니다. `SafeHandle`은 이를 안전하게 관리하는 방법입니다.

```csharp
public class SafeFileHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    public SafeFileHandle() : base(true) { }

    [DllImport("kernel32.dll", SetLastError = true)]
    private static extern bool CloseHandle(IntPtr hObject);

    protected override bool ReleaseHandle()
    {
        return CloseHandle(handle);
    }
}

[DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
public static extern SafeFileHandle CreateFileW(
    string lpFileName,
    uint dwDesiredAccess,
    uint dwShareMode,
    IntPtr lpSecurityAttributes,
    uint dwCreationDisposition,
    uint dwFlagsAndAttributes,
    IntPtr hTemplateFile);

// 사용
using (SafeFileHandle handle = CreateFileW("test.txt", ...))
{
    // 작업
}
```

`SafeHandle`을 상속받으면 `Dispose` 시 자동으로 핸들이 닫히므로 리소스 누수를 방지할 수 있습니다.

---

## Span<T>와 Memory<T>: 현대적 메모리 접근

.NET Core 2.1부터 도입된 `Span<T>`는 네이티브 메모리를 안전하게 다룰 수 있는 방법입니다.

```csharp
[DllImport("mylib.dll", CallingConvention = CallingConvention.Cdecl)]
public static extern unsafe bool ProcessSpan(byte* buffer, int length);

public unsafe bool ProcessWithSpan(byte[] data)
{
    fixed (byte* ptr = data)
    {
        Span<byte> span = new Span<byte>(ptr, data.Length);
        // span으로 안전하게 접근 가능
        return ProcessSpan(ptr, data.Length);
    }
}
```

또한 `Memory<T>`와 `MemoryPool<T>`를 사용하면 대용량 데이터 풀링을 효율적으로 할 수 있습니다.

---

## NativeMemory: .NET 6+의 저수준 메모리 API

.NET 6부터 `NativeMemory` 클래스를 통해 비관리 메모리를 직접 할당/해제할 수 있습니다.

```csharp
unsafe void AllocateExample()
{
    void* memory = NativeMemory.Alloc(1024);
    try
    {
        NativeMemory.Fill(memory, 1024, 0x00);
        Span<byte> span = new Span<byte>(memory, 1024);
        // 작업
    }
    finally
    {
        NativeMemory.Free(memory);
    }
}
```

정렬된 메모리가 필요하면 `NativeMemory.AlignedAlloc`을 사용합니다.

---

## 역방향 P/Invoke: 콜백

네이티브 코드가 C# 코드를 호출해야 할 때는 콜백을 사용합니다. `UnmanagedFunctionPointer`로 델리게이트를 선언하고 네이티브에 전달합니다.

```csharp
[UnmanagedFunctionPointer(CallingConvention.Cdecl)]
public delegate void ProgressCallback(int percent, [MarshalAs(UnmanagedType.LPStr)] string message);

[DllImport("processing.dll", CallingConvention = CallingConvention.Cdecl)]
public static extern int LongOperation(IntPtr data, int dataSize, ProgressCallback callback);

public void PerformOperation(byte[] data)
{
    ProgressCallback callback = (percent, msg) =>
    {
        Console.WriteLine($"{percent}%: {msg}");
    };

    // 델리게이트가 GC에 수집되지 않도록 유지
    GCHandle handle = GCHandle.Alloc(callback);
    try
    {
        unsafe
        {
            fixed (byte* p = data)
            {
                LongOperation((IntPtr)p, data.Length, callback);
            }
        }
    }
    finally
    {
        handle.Free();
    }
}
```

.NET 5+에서는 `UnmanagedCallersOnly`를 사용하여 더 안전하게 콜백을 정의할 수 있습니다.

---

## 동적 라이브러리 로딩

`NativeLibrary` 클래스를 사용하면 런타임에 라이브러리를 로드하고 함수를 가져올 수 있습니다.

```csharp
public class DynamicLoader
{
    private nint _handle;

    public void LoadLibrary(string path)
    {
        _handle = NativeLibrary.Load(path);
    }

    public TDelegate GetFunction<TDelegate>(string name) where TDelegate : Delegate
    {
        nint ptr = NativeLibrary.GetExport(_handle, name);
        return Marshal.GetDelegateForFunctionPointer<TDelegate>(ptr);
    }

    public void Unload()
    {
        if (_handle != IntPtr.Zero)
        {
            NativeLibrary.Free(_handle);
            _handle = IntPtr.Zero;
        }
    }
}
```

플랫폼별 라이브러리 이름은 `RuntimeInformation`으로 결정할 수 있습니다.

```csharp
string libName = RuntimeInformation.IsOSPlatform(OSPlatform.Windows) ? "mylib.dll" :
                 RuntimeInformation.IsOSPlatform(OSPlatform.Linux) ? "libmylib.so" :
                 "libmylib.dylib";
```

---

## 실무 패턴: 안전한 래퍼 만들기

### 리소스 래퍼 클래스

```csharp
public sealed class NativeResource : IDisposable
{
    private readonly nint _handle;
    private bool _disposed;

    public NativeResource()
    {
        _handle = CreateNativeResource();
        if (_handle == IntPtr.Zero)
            throw new InvalidOperationException("Failed to create native resource");
    }

    [DllImport("mylib.dll")]
    private static extern nint CreateNativeResource();

    [DllImport("mylib.dll")]
    private static extern void DestroyNativeResource(nint handle);

    public void DoSomething()
    {
        if (_disposed) throw new ObjectDisposedException(nameof(NativeResource));
        NativeMethod(_handle);
    }

    [DllImport("mylib.dll")]
    private static extern void NativeMethod(nint handle);

    public void Dispose()
    {
        if (!_disposed)
        {
            DestroyNativeResource(_handle);
            _disposed = true;
        }
        GC.SuppressFinalize(this);
    }

    ~NativeResource() => Dispose();
}
```

### 예외 변환

네이티브 오류를 C# 예외로 변환합니다.

```csharp
[DllImport("mylib.dll", SetLastError = true)]
private static extern bool NativeOperation();

public void SafeOperation()
{
    if (!NativeOperation())
    {
        int error = Marshal.GetLastWin32Error();
        throw new InvalidOperationException($"Native operation failed: {error}");
    }
}
```

---

## 결론

C#의 네이티브 상호 운용성은 강력하지만 책임이 따릅니다. 다음 원칙을 기억하세요.

- **SafeHandle**을 사용해 네이티브 리소스를 안전하게 관리합니다.
- **문자열 인코딩과 호출 규약**을 정확히 맞춥니다.
- **구조체 레이아웃**을 명시적으로 지정합니다.
- 가능하면 **Span<T>** 와 **Memory<T>** 같은 현대적 추상화를 사용해 안전성과 성능을 모두 챙깁니다.
- **테스트**는 여러 플랫폼에서 수행합니다.

네이티브 코드와의 경계를 넘나드는 것은 복잡성을 증가시키지만, 잘 설계된 추상화 계층은 이러한 복잡성을 숨기고 안정적인 API를 제공합니다. 적절한 경우에만 사용하고, 항상 안전성을 최우선으로 하세요.