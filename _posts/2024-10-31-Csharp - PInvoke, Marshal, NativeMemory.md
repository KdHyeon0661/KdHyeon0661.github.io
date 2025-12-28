---
layout: post
title: C# - PInvoke, Marshal, NativeMemory
date: 2024-10-31 19:20:23 +0900
category: Csharp
---
# C# 네이티브 상호 운용성: P/Invoke, 마샬링, 그리고 안전한 경계 넘나들기

## 네이티브 상호 운용성의 세계로 들어서기

C#과 .NET 생태계는 풍부한 기능을 제공하지만, 때로는 운영 체제 API, 하드웨어 제어, 고성능 C/C++ 라이브러리와 직접 통신해야 할 필요가 있습니다. P/Invoke(Platform Invocation Services)는 이러한 경계를 넘나들 수 있게 해주는 강력한 도구입니다. 하지만 이 힘에는 책임이 따르며, 잘못 사용하면 메모리 누수, 크래시, 보안 취약점을 초래할 수 있습니다.

```csharp
// 가장 기본적인 P/Invoke 예제
[DllImport("user32.dll", CharSet = CharSet.Unicode)]
public static extern int MessageBox(IntPtr hWnd, string text, string caption, uint type);

// 사용
MessageBox(IntPtr.Zero, "안녕하세요!", "인사", 0);
```

## P/Invoke의 기본 구성 요소 이해하기

### DllImport 특성: 네이티브 세계와의 계약

```csharp
[AttributeUsage(AttributeTargets.Method)]
public class EnhancedDllImport : Attribute
{
    public string LibraryName { get; }
    public CharSet CharSet { get; set; } = CharSet.Auto;
    public CallingConvention CallingConvention { get; set; } = CallingConvention.Winapi;
    public bool SetLastError { get; set; }
    public bool ExactSpelling { get; set; }
    public bool PreserveSig { get; set; } = true;
    
    public EnhancedDllImport(string libraryName)
    {
        LibraryName = libraryName;
    }
}

// 실제 사용 예
[DllImport("kernel32.dll", 
    CharSet = CharSet.Unicode,
    SetLastError = true,
    ExactSpelling = true,
    CallingConvention = CallingConvention.StdCall)]
public static extern IntPtr GetModuleHandleW(string lpModuleName);
```

### 호출 규약: 함수가 어떻게 호출되는지 정의하기

```csharp
public class CallingConventionExamples
{
    // Windows API의 표준 호출 규약
    [DllImport("kernel32.dll", CallingConvention = CallingConvention.StdCall)]
    public static extern bool Beep(uint dwFreq, uint dwDuration);
    
    // C 라이브러리의 표준 호출 규약
    [DllImport("msvcrt.dll", CallingConvention = CallingConvention.Cdecl)]
    public static extern int printf(string format, __arglist);
    
    // 가변 인자 함수는 반드시 Cdecl이어야 함
    [DllImport("msvcrt.dll", CallingConvention = CallingConvention.Cdecl)]
    public static extern int sprintf(StringBuilder buffer, string format, __arglist);
    
    // COM 구성 요소와의 상호 운용
    [DllImport("ole32.dll", CallingConvention = CallingConvention.StdCall)]
    public static extern int CoInitialize(IntPtr pvReserved);
}
```

**호출 규약 중요성**: 올바른 호출 규약을 사용하지 않으면 스택이 손상되어 애플리케이션이 예측 불가능하게 충돌할 수 있습니다.

## 문자열 마샬링: 문화적 차이를 극복하기

### 문자 인코딩의 다양한 세계

```csharp
public class StringMarshalingExamples
{
    // ANSI (현재 시스템 코드 페이지)
    [DllImport("user32.dll", CharSet = CharSet.Ansi)]
    public static extern int MessageBoxA(IntPtr hWnd, string text, string caption, uint type);
    
    // Unicode (UTF-16, Windows의 기본)
    [DllImport("user32.dll", CharSet = CharSet.Unicode)]
    public static extern int MessageBoxW(IntPtr hWnd, string text, string caption, uint type);
    
    // 플랫폼 자동 선택 (Windows에서는 Unicode, Linux에서는 ANSI)
    [DllImport("user32.dll", CharSet = CharSet.Auto)]
    public static extern int MessageBox(IntPtr hWnd, string text, string caption, uint type);
    
    // UTF-8 직접 처리 (.NET 5+)
    [DllImport("mylib.dll")]
    public static extern IntPtr GetUtf8String();
    
    public string GetStringFromNativeUtf8()
    {
        IntPtr nativeString = GetUtf8String();
        try
        {
            return Marshal.PtrToStringUTF8(nativeString);
        }
        finally
        {
            // 네이티브 측에서 메모리 해제 필요시
            // FreeNativeString(nativeString);
        }
    }
    
    // StringBuilder를 사용한 출력 버퍼
    [DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
    public static extern int GetCurrentDirectoryW(int nBufferLength, StringBuilder lpBuffer);
    
    public string GetCurrentDirectorySafe()
    {
        // Windows API의 MAX_PATH는 260이지만, 길게 준비
        StringBuilder buffer = new StringBuilder(32767);
        int length = GetCurrentDirectoryW(buffer.Capacity, buffer);
        
        if (length == 0)
        {
            int error = Marshal.GetLastWin32Error();
            throw new System.ComponentModel.Win32Exception(error);
        }
        
        return buffer.ToString(0, length);
    }
}
```

### 문자열 소유권과 메모리 관리

```csharp
public class StringOwnershipPatterns
{
    // 패턴 1: 네이티브가 할당, 호출자가 해제
    [DllImport("mylib.dll", CharSet = CharSet.Ansi)]
    public static extern IntPtr CreateMessageAnsi();
    
    [DllImport("mylib.dll", CharSet = CharSet.Ansi)]
    public static extern void FreeMessageAnsi(IntPtr message);
    
    // 패턴 2: 네이티브가 할당, 호출자가 해제 (Unicode)
    [DllImport("mylib.dll", CharSet = CharSet.Unicode)]
    public static extern IntPtr CreateMessageUnicode();
    
    [DllImport("mylib.dll", CharSet = CharSet.Unicode)]
    public static extern void FreeMessageUnicode(IntPtr message);
    
    // 패턴 3: 정적 버퍼 (네이티브가 소유)
    [DllImport("mylib.dll", CharSet = CharSet.Ansi)]
    public static extern IntPtr GetErrorMessage(int errorCode);
    
    // 패턴 4: 호출자가 버퍼 제공
    [DllImport("mylib.dll", CharSet = CharSet.Unicode)]
    public static extern int FormatMessageW(
        uint dwFlags,
        IntPtr lpSource,
        uint dwMessageId,
        uint dwLanguageId,
        StringBuilder lpBuffer,
        int nSize,
        IntPtr arguments);
    
    public string GetLastErrorDescription()
    {
        const int FORMAT_MESSAGE_FROM_SYSTEM = 0x00001000;
        const int FORMAT_MESSAGE_IGNORE_INSERTS = 0x00000200;
        
        StringBuilder buffer = new StringBuilder(1024);
        int length = FormatMessageW(
            FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS,
            IntPtr.Zero,
            (uint)Marshal.GetLastWin32Error(),
            0, // 언어 ID (0 = 시스템 기본)
            buffer,
            buffer.Capacity,
            IntPtr.Zero);
        
        if (length == 0)
        {
            return "Unknown error";
        }
        
        // Windows 메시지 끝의 줄바꿈 제거
        string result = buffer.ToString(0, length);
        return result.TrimEnd('\r', '\n', ' ', '.');
    }
}
```

## 구조체와 배열 마샬링: 데이터 구조 맞추기

### 구조체 레이아웃 정확히 맞추기

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1, CharSet = CharSet.Unicode)]
public struct Win32FindData
{
    public uint dwFileAttributes;
    public System.Runtime.InteropServices.ComTypes.FILETIME ftCreationTime;
    public System.Runtime.InteropServices.ComTypes.FILETIME ftLastAccessTime;
    public System.Runtime.InteropServices.ComTypes.FILETIME ftLastWriteTime;
    public uint nFileSizeHigh;
    public uint nFileSizeLow;
    public uint dwReserved0;
    public uint dwReserved1;
    
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 260)]
    public string cFileName;
    
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 14)]
    public string cAlternateFileName;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct DeviceInfo
{
    public int DeviceId;
    public DeviceType Type;
    
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 16)]
    public byte[] SerialNumber;
    
    public uint Flags;
    public long Timestamp;
}

// 명시적 레이아웃 (C 공용체 모방)
[StructLayout(LayoutKind.Explicit)]
public struct NetworkAddress
{
    [FieldOffset(0)]
    public uint IPv4Address;
    
    [FieldOffset(0)]
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 4)]
    public byte[] IPv4Bytes;
    
    [FieldOffset(0)]
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 16)]
    public byte[] IPv6Bytes;
}
```

### 배열과 버퍼 마샬링

```csharp
public class ArrayMarshalingExamples
{
    // 1. 입력 배열 (읽기 전용)
    [DllImport("mylib.dll", CallingConvention = CallingConvention.Cdecl)]
    public static extern double CalculateAverage(
        [MarshalAs(UnmanagedType.LPArray)] double[] values,
        int count);
    
    // 2. 출력 배열 (쓰기 전용)
    [DllImport("mylib.dll", CallingConvention = CallingConvention.Cdecl)]
    public static extern int GenerateRandomNumbers(
        [MarshalAs(UnmanagedType.LPArray), Out] int[] buffer,
        int count);
    
    // 3. 입출력 배열 (양방향)
    [DllImport("mylib.dll", CallingConvention = CallingConvention.Cdecl)]
    public static extern void TransformArray(
        [MarshalAs(UnmanagedType.LPArray), In, Out] float[] data,
        int length);
    
    // 4. 다차원 배열
    [DllImport("mylib.dll", CallingConvention = CallingConvention.Cdecl)]
    public static extern void ProcessMatrix(
        [MarshalAs(UnmanagedType.LPArray)] double[,] matrix,
        int rows,
        int columns);
    
    // 5. unsafe 포인터 방식 (고성능)
    [DllImport("mylib.dll", CallingConvention = CallingConvention.Cdecl)]
    public static extern unsafe void ProcessBuffer(
        byte* buffer,
        int length);
    
    public void ProcessDataUnsafe(byte[] data)
    {
        unsafe
        {
            fixed (byte* ptr = data)
            {
                ProcessBuffer(ptr, data.Length);
            }
        }
    }
}
```

## SafeHandle: 리소스 누수의 최후 방어선

```csharp
public abstract class SafeNativeHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    protected SafeNativeHandle() : base(true) { }
    protected SafeNativeHandle(IntPtr handle, bool ownsHandle) : base(ownsHandle)
    {
        SetHandle(handle);
    }
    
    [DllImport("kernel32.dll", SetLastError = true)]
    protected static extern bool CloseHandle(IntPtr hObject);
}

public sealed class SafeFileMappingHandle : SafeNativeHandle
{
    private SafeFileMappingHandle() : base() { }
    private SafeFileMappingHandle(IntPtr handle) : base(handle, true) { }
    
    [DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
    public static extern SafeFileMappingHandle CreateFileMappingW(
        SafeFileHandle hFile,
        IntPtr lpFileMappingAttributes,
        uint flProtect,
        uint dwMaximumSizeHigh,
        uint dwMaximumSizeLow,
        string lpName);
    
    protected override bool ReleaseHandle()
    {
        return CloseHandle(handle);
    }
    
    public static SafeFileMappingHandle CreateFromFile(
        string filePath,
        FileAccess access,
        FileShare share,
        uint size)
    {
        using (var fileHandle = CreateFileHandle(filePath, access, share))
        {
            uint protect = access.HasFlag(FileAccess.Write) 
                ? 0x04 /*PAGE_READWRITE*/ 
                : 0x02 /*PAGE_READONLY*/;
            
            return CreateFileMappingW(
                fileHandle,
                IntPtr.Zero,
                protect,
                0, // size high
                size,
                null);
        }
    }
    
    private static SafeFileHandle CreateFileHandle(
        string path,
        FileAccess access,
        FileShare share)
    {
        return File.OpenHandle(
            path,
            FileMode.OpenOrCreate,
            access,
            share,
            FileOptions.None);
    }
}

// CriticalHandle: 더 엄격한 수명 관리 (Dispose에서 예외 없음)
public sealed class CriticalMutexHandle : CriticalHandleZeroOrMinusOneIsInvalid
{
    public CriticalMutexHandle() : base() { }
    
    [DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
    public static extern CriticalMutexHandle CreateMutexW(
        IntPtr lpMutexAttributes,
        bool bInitialOwner,
        string lpName);
    
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern bool ReleaseMutex(IntPtr hMutex);
    
    protected override bool ReleaseHandle()
    {
        return ReleaseMutex(handle);
    }
}
```

## Span<T>와 Memory<T>: 현대적 메모리 관리

```csharp
public class SpanInteropExamples
{
    // Span<T>를 사용한 안전한 버퍼 처리
    public static unsafe bool ProcessWithSpan(byte[] input, byte[] output)
    {
        if (output.Length < input.Length)
            return false;
        
        fixed (byte* pInput = input)
        fixed (byte* pOutput = output)
        {
            Span<byte> inputSpan = new Span<byte>(pInput, input.Length);
            Span<byte> outputSpan = new Span<byte>(pOutput, output.Length);
            
            // 네이티브 함수 호출
            return NativeProcessBuffer(pInput, input.Length, pOutput, output.Length);
        }
    }
    
    [DllImport("mylib.dll", CallingConvention = CallingConvention.Cdecl)]
    private static extern unsafe bool NativeProcessBuffer(
        byte* input, int inputLength,
        byte* output, int outputLength);
    
    // Memory<T>와 IMemoryOwner를 사용한 풀링
    public static async Task ProcessWithMemoryPoolAsync(
        Stream inputStream,
        Stream outputStream,
        CancellationToken cancellationToken = default)
    {
        using var inputOwner = MemoryPool<byte>.Shared.Rent(8192);
        using var outputOwner = MemoryPool<byte>.Shared.Rent(8192);
        
        Memory<byte> inputBuffer = inputOwner.Memory;
        Memory<byte> outputBuffer = outputOwner.Memory;
        
        int bytesRead;
        while ((bytesRead = await inputStream.ReadAsync(
            inputBuffer, cancellationToken)) > 0)
        {
            unsafe
            {
                fixed (byte* pInput = inputBuffer.Span)
                fixed (byte* pOutput = outputBuffer.Span)
                {
                    bool success = NativeTransform(
                        pInput, bytesRead,
                        pOutput, outputBuffer.Length);
                    
                    if (!success)
                        throw new InvalidOperationException("Transform failed");
                }
            }
            
            await outputStream.WriteAsync(
                outputBuffer[..bytesRead], cancellationToken);
        }
    }
    
    [DllImport("transform.dll", CallingConvention = CallingConvention.Cdecl)]
    private static extern unsafe bool NativeTransform(
        byte* input, int inputLength,
        byte* output, int outputCapacity);
}
```

## NativeMemory: .NET 6+의 저수준 메모리 API

```csharp
public class NativeMemoryExamples
{
    // 기본 메모리 할당/해제
    public static unsafe void BasicAllocation()
    {
        // 1KB 메모리 할당
        void* memory = NativeMemory.Alloc(1024);
        
        try
        {
            // 메모리 초기화
            NativeMemory.Fill(memory, 1024, 0x00);
            
            // Span으로 래핑
            Span<byte> buffer = new Span<byte>(memory, 1024);
            
            // 작업 수행
            buffer[0] = 0xFF;
            buffer[^1] = 0xAA;
        }
        finally
        {
            // 메모리 해제
            NativeMemory.Free(memory);
        }
    }
    
    // 정렬된 메모리 할당
    public static unsafe void AlignedAllocation()
    {
        const int alignment = 64; // AVX-512 정렬
        const int size = 1024;
        
        void* memory = NativeMemory.AlignedAlloc(size, alignment);
        
        try
        {
            // 정렬 확인
            bool isAligned = ((nint)memory % alignment) == 0;
            Console.WriteLine($"Memory is {(isAligned ? "aligned" : "not aligned")}");
            
            // SIMD 작업 등 정렬이 중요한 작업 수행
            ProcessAlignedMemory(memory, size);
        }
        finally
        {
            NativeMemory.AlignedFree(memory);
        }
    }
    
    // 재할당
    public static unsafe void* ReallocateBuffer(void* oldBuffer, nuint oldSize, nuint newSize)
    {
        if (newSize <= oldSize)
            return oldBuffer;
        
        void* newBuffer = NativeMemory.Realloc(oldBuffer, newSize);
        
        if (newBuffer != oldBuffer)
        {
            // 새로운 메모리의 추가 부분 초기화
            byte* newBytes = (byte*)newBuffer;
            NativeMemory.Fill(newBytes + oldSize, newSize - oldSize, 0x00);
        }
        
        return newBuffer;
    }
    
    // 대용량 데이터 처리를 위한 메모리 풀
    public class NativeMemoryPool : IDisposable
    {
        private readonly ConcurrentQueue<(void* ptr, nuint size)> _pool = new();
        private readonly nuint _chunkSize;
        private readonly int _maxChunks;
        
        public NativeMemoryPool(nuint chunkSize, int maxChunks)
        {
            _chunkSize = chunkSize;
            _maxChunks = maxChunks;
        }
        
        public unsafe Span<byte> Rent()
        {
            if (_pool.TryDequeue(out var item))
            {
                return new Span<byte>(item.ptr, (int)item.size);
            }
            
            if (_pool.Count >= _maxChunks)
                throw new InvalidOperationException("Pool exhausted");
            
            void* memory = NativeMemory.Alloc(_chunkSize);
            return new Span<byte>(memory, (int)_chunkSize);
        }
        
        public unsafe void Return(Span<byte> buffer)
        {
            fixed (byte* ptr = buffer)
            {
                _pool.Enqueue((ptr, (nuint)buffer.Length));
            }
        }
        
        public unsafe void Dispose()
        {
            while (_pool.TryDequeue(out var item))
            {
                NativeMemory.Free(item.ptr);
            }
        }
    }
}
```

## 역방향 P/Invoke와 콜백

```csharp
public class ReversePInvokeExamples
{
    // 전통적인 델리게이트 방식
    [UnmanagedFunctionPointer(CallingConvention.Cdecl)]
    public delegate void ProgressCallback(int percent, [MarshalAs(UnmanagedType.LPStr)] string message);
    
    [DllImport("processing.dll", CallingConvention = CallingConvention.Cdecl)]
    public static extern int LongOperation(
        IntPtr data,
        int dataSize,
        ProgressCallback callback);
    
    public void PerformOperationWithCallback(byte[] data)
    {
        ProgressCallback callback = (percent, message) =>
        {
            Console.WriteLine($"[{percent}%] {message}");
        };
        
        // 델리게이트를 GC에 의해 수집되지 않도록 유지
        GCHandle callbackHandle = GCHandle.Alloc(callback);
        
        try
        {
            unsafe
            {
                fixed (byte* pData = data)
                {
                    int result = LongOperation(
                        (IntPtr)pData,
                        data.Length,
                        callback);
                        
                    if (result != 0)
                        throw new InvalidOperationException($"Operation failed: {result}");
                }
            }
        }
        finally
        {
            callbackHandle.Free();
        }
    }
    
    // 현대적 접근: UnmanagedCallersOnly (.NET 5+)
    [UnmanagedCallersOnly(CallConvs = new[] { typeof(CallConvCdecl) })]
    public static void NativeProgressCallback(int percent, byte* messageUtf8)
    {
        try
        {
            string message = Marshal.PtrToStringUTF8((IntPtr)messageUtf8);
            Console.WriteLine($"[{percent}%] {message}");
        }
        catch (Exception ex)
        {
            Console.Error.WriteLine($"Error in callback: {ex.Message}");
        }
    }
    
    // 함수 포인터 얻기 (.NET 9+)
    public static unsafe delegate* unmanaged[Cdecl]<int, byte*, void> GetCallbackPointer()
    {
        return &NativeProgressCallback;
    }
}
```

## 동적 라이브러리 로딩 전략

```csharp
public class DynamicLibraryLoader
{
    private readonly Dictionary<string, nint> _loadedLibraries = new();
    
    public TDelegate LoadFunction<TDelegate>(string libraryPath, string functionName)
        where TDelegate : Delegate
    {
        if (!_loadedLibraries.TryGetValue(libraryPath, out nint libraryHandle))
        {
            libraryHandle = NativeLibrary.Load(libraryPath);
            _loadedLibraries[libraryPath] = libraryHandle;
        }
        
        nint functionPtr = NativeLibrary.GetExport(libraryHandle, functionName);
        
        if (functionPtr == IntPtr.Zero)
            throw new EntryPointNotFoundException($"Function {functionName} not found in {libraryPath}");
        
        return Marshal.GetDelegateForFunctionPointer<TDelegate>(functionPtr);
    }
    
    public void UnloadLibrary(string libraryPath)
    {
        if (_loadedLibraries.TryGetValue(libraryPath, out nint handle))
        {
            NativeLibrary.Free(handle);
            _loadedLibraries.Remove(libraryPath);
        }
    }
    
    public void Dispose()
    {
        foreach (var handle in _loadedLibraries.Values)
        {
            NativeLibrary.Free(handle);
        }
        _loadedLibraries.Clear();
    }
    
    // 플랫폼별 라이브러리 이름 해결
    public static string GetPlatformLibraryName(string baseName)
    {
        if (RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
            return $"{baseName}.dll";
        else if (RuntimeInformation.IsOSPlatform(OSPlatform.Linux))
            return $"lib{baseName}.so";
        else if (RuntimeInformation.IsOSPlatform(OSPlatform.OSX))
            return $"lib{baseName}.dylib";
        else
            throw new PlatformNotSupportedException();
    }
    
    // 라이브러리 검색 경로 전략
    public static nint LoadWithSearchPaths(string libraryName, string[] searchPaths)
    {
        foreach (var path in searchPaths)
        {
            string fullPath = Path.Combine(path, libraryName);
            if (File.Exists(fullPath))
            {
                return NativeLibrary.Load(fullPath);
            }
        }
        
        // 기본 검색
        return NativeLibrary.Load(libraryName);
    }
}
```

## 실전 패턴: 안전한 네이티브 상호 운용성

### 패턴 1: 리소스 관리 래퍼

```csharp
public sealed class NativeLibraryWrapper : IDisposable
{
    private readonly nint _handle;
    private bool _disposed;
    
    public NativeLibraryWrapper(string libraryPath)
    {
        _handle = NativeLibrary.Load(libraryPath);
    }
    
    public TDelegate GetFunction<TDelegate>(string functionName)
        where TDelegate : Delegate
    {
        if (_disposed)
            throw new ObjectDisposedException(nameof(NativeLibraryWrapper));
        
        nint functionPtr = NativeLibrary.GetExport(_handle, functionName);
        return Marshal.GetDelegateForFunctionPointer<TDelegate>(functionPtr);
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            NativeLibrary.Free(_handle);
            _disposed = true;
        }
        GC.SuppressFinalize(this);
    }
    
    ~NativeLibraryWrapper()
    {
        Dispose();
    }
}
```

### 패턴 2: 예외 변환

```csharp
public static class NativeExceptionHelper
{
    public static void ThrowLastWin32Error()
    {
        int errorCode = Marshal.GetLastWin32Error();
        throw new Win32Exception(errorCode);
    }
    
    public static void ThrowIfFailed(bool success)
    {
        if (!success)
            ThrowLastWin32Error();
    }
    
    public static void ThrowIfFailed(int result)
    {
        if (result == 0) // 많은 Windows API는 0이 실패
            ThrowLastWin32Error();
    }
    
    public static T ThrowIfNull<T>(T? value) where T : class
    {
        if (value == null)
            ThrowLastWin32Error();
        return value;
    }
    
    public static IntPtr ThrowIfInvalid(IntPtr handle)
    {
        if (handle == IntPtr.Zero || handle == new IntPtr(-1))
            ThrowLastWin32Error();
        return handle;
    }
}
```

### 패턴 3: 성능 최적화를 위한 풀링

```csharp
public class NativeBufferPool : IDisposable
{
    private readonly ConcurrentStack<IntPtr> _pool = new();
    private readonly int _bufferSize;
    private readonly int _maxBuffers;
    
    public NativeBufferPool(int bufferSize, int maxBuffers = 100)
    {
        _bufferSize = bufferSize;
        _maxBuffers = maxBuffers;
    }
    
    public IntPtr Rent()
    {
        if (_pool.TryPop(out IntPtr buffer))
        {
            return buffer;
        }
        
        if (_pool.Count < _maxBuffers)
        {
            return Marshal.AllocHGlobal(_bufferSize);
        }
        
        throw new InvalidOperationException("Buffer pool exhausted");
    }
    
    public void Return(IntPtr buffer)
    {
        if (_pool.Count < _maxBuffers)
        {
            // 버퍼 재사용 전 초기화
            unsafe
            {
                byte* ptr = (byte*)buffer;
                for (int i = 0; i < _bufferSize; i++)
                {
                    ptr[i] = 0;
                }
            }
            _pool.Push(buffer);
        }
        else
        {
            Marshal.FreeHGlobal(buffer);
        }
    }
    
    public void Dispose()
    {
        while (_pool.TryPop(out IntPtr buffer))
        {
            Marshal.FreeHGlobal(buffer);
        }
    }
}
```

## 결론: 네이티브 상호 운용성의 미래와 현재

C#의 네이티브 상호 운용성 기능은 지난 20년 동안 꾸준히 발전해왔습니다. 초기의 간단한 P/Invoke에서 시작하여 현재는 안전성, 성능, 유연성 모두를 고려한 다양한 기능들을 제공하고 있습니다.

### 핵심 원칙 정리

1. **안전성 최우선**
   - `SafeHandle`은 네이티브 리소스 관리의 표준입니다.
   - 리소스 누수는 메모리 누수보다 더 심각한 문제를 일으킬 수 있습니다.
   - `using` 문과 `IDisposable` 패턴을 적극 활용하세요.

2. **정확성은 생명**
   - 호출 규약, 구조체 레이아웃, 문자 인코딩은 반드시 정확해야 합니다.
   - 테스트는 여러 플랫폼에서 수행하세요.
   - 문서화는 자세하게, 특히 경계 조건과 오류 처리를 명시하세요.

3. **성능과 안전성의 균형**
   - `Span<T>`와 `Memory<T>`는 안전성과 성능을 모두 잡을 수 있는 현대적 도구입니다.
   - 필요한 경우에만 `unsafe` 코드를 사용하세요.
   - 프로파일링으로 실제 성능 이점을 확인하세요.

4. **플랫폼 차이 존중**
   - Windows, Linux, macOS는 각기 다른 ABI와 관습을 가지고 있습니다.
   - `RuntimeInformation`을 사용하여 플랫폼별 코드 경로를 관리하세요.
   - 크로스 플랫폼 라이브러리 이름 변환을 자동화하세요.

### 미래를 향한 발전

.NET 7과 8에서는 네이티브 상호 운용성이 더욱 발전하고 있습니다:

- **AOT 컴파일**: 정적 링킹과 더 나은 성능
- **소스 생성기**: 컴파일 타임에 P/Invoke 코드 생성
- **더 나은 오류 메시지**: 디버깅 용이성 향상
- **SIMD 지원**: 벡터화 연산과의 통합

### 최종 조언

네이티브 상호 운용성은 강력한 도구이지만, 신중하게 사용해야 합니다. 가능하면 순수 .NET 솔루션을 먼저 고려하세요. 정말 필요한 경우에만 네이티브 코드로의 경계를 넘나들고, 그때는 이 가이드의 원칙을 따르세요.

기억하세요: 가장 우아한 코드는 네이티브 코드와의 경계를 완전히 숨기는 추상화 계층을 제공하는 코드입니다. 사용자는 여러분의 라이브러리가 내부적으로 네이티브 코드를 호출하는지 여부를 알 필요가 없으며, 알려고도 하지 않을 것입니다. 그저 안정적으로 동작하는 고성능 API를 원할 뿐입니다.

이것이 바로 .NET의 진정한 힘입니다: 복잡한 네이티브 상호 운용성의 세부 사항을 숨기면서도, 필요할 때는 그 모든 힘에 접근할 수 있는 능력. 이 균형을 잘 유지하는 것이 전문 C# 개발자의 자질입니다.