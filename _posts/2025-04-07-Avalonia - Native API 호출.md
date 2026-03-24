---
layout: post
title: Avalonia - Native API 호출
date: 2025-04-07 21:20:23 +0900
category: Avalonia
---
# Avalonia에서 Native API 호출

Avalonia는 데스크톱 플랫폼에서 네이티브 API를 직접 호출해야 할 때가 있다. 예를 들어 블루투스, 시리얼 통신, 특정 하드웨어 제어 등은 운영체제 수준의 기능을 사용해야 한다. 이 글에서는 **P/Invoke**를 기본으로 하되, 운영체제별 차이를 추상화하고 Avalonia UI와 깔끔하게 통합하는 방법을 소개한다.

## 왜 네이티브 API가 필요한가

| 시나리오 | 설명 |
|----------|------|
| 하드웨어 제어 | 블루투스, USB, HID, 센서, 프린터 등 |
| 고급 시스템 기능 | 전원 관리, 디스플레이 설정, 알림, 파일 다이얼로그 |
| 기존 C/C++ 라이브러리 활용 | OpenCV, FFmpeg 등 전문 라이브러리 |
| 플랫폼별 고유 기능 | macOS의 Keychain, Windows의 레지스트리 |

## 기본: P/Invoke로 C API 호출

가장 단순한 예로 Windows 메시지 박스를 호출해본다.

```csharp
using System.Runtime.InteropServices;

internal static class Win32
{
    [DllImport("user32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
    public static extern int MessageBoxW(IntPtr hWnd, string text, string caption, uint type);
}

// 사용 예
Win32.MessageBoxW(IntPtr.Zero, "Hello from Win32!", "제목", 0);
```

- `CharSet.Unicode`로 UTF-16 문자열을 전달한다.
- `SetLastError = true`로 오류 정보를 `Marshal.GetLastWin32Error()`로 확인할 수 있다.

Linux/macOS에서 프로세스 ID를 얻는 예:

```csharp
[DllImport("libc", EntryPoint = "getpid")]
public static extern int GetPid();
```

## 구조체와 버퍼 마샬링

네이티브 API가 구조체나 버퍼를 사용할 때는 `StructLayout`과 `MarshalAs`를 이용한다.

```csharp
[StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
public struct MyNativeInfo
{
    public int Size;
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 260)]
    public string Path;
}
```

## 플랫폼 분기와 자원 해제

### 운영체제 감지

```csharp
if (OperatingSystem.IsWindows()) { /* Win32 */ }
else if (OperatingSystem.IsLinux()) { /* Linux */ }
else if (OperatingSystem.IsMacOS()) { /* macOS */ }
```

### `SafeHandle`로 핸들 안전하게 관리

네이티브 핸들(파일, 디바이스)을 `SafeHandle`로 감싸면 리소스 누수를 방지할 수 있다.

```csharp
public sealed class SafeNativeHandle : SafeHandle
{
    public SafeNativeHandle() : base(IntPtr.Zero, true) { }
    public override bool IsInvalid => handle == IntPtr.Zero;

    [DllImport("kernel32.dll")]
    private static extern bool CloseHandle(IntPtr hObject);

    protected override bool ReleaseHandle() => CloseHandle(handle);
}
```

## 시리얼 포트 (직렬 통신) 예제

.NET은 `System.IO.Ports.SerialPort`를 제공하므로 별도 P/Invoke 없이 크로스 플랫폼으로 사용할 수 있다.

```csharp
using System.IO.Ports;

public async Task RunAsync(string portName = "COM3", int baud = 9600)
{
    using var port = new SerialPort(portName, baud)
    {
        ReadTimeout = 2000,
        WriteTimeout = 2000,
        NewLine = "\r\n"
    };
    port.Open();

    port.WriteLine("HELLO");
    string response = port.ReadLine();
    Console.WriteLine(response);
}
```

- Windows: `COM3`, Linux: `/dev/ttyUSB0`, macOS: `/dev/tty.usbserial-*` 등 포트 이름이 다르다.
- 권한: Linux에서는 사용자를 `dialout` 그룹에 추가해야 한다.

## 블루투스: 플랫폼별 스택과 추상화

블루투스는 OS별로 API가 완전히 다르기 때문에 **인터페이스 + 플랫폼별 구현** 패턴이 필수적이다.

### 공통 인터페이스 정의

```csharp
public interface IBluetoothService
{
    IAsyncEnumerable<BluetoothDeviceInfo> ScanAsync(CancellationToken ct);
    Task ConnectAsync(string deviceId, CancellationToken ct);
    Task<byte[]> ReadAsync(Guid service, Guid characteristic, CancellationToken ct);
    Task WriteAsync(Guid service, Guid characteristic, byte[] data, CancellationToken ct);
}

public record BluetoothDeviceInfo(string Name, string Id);
```

### 플랫폼별 구현

- **Windows**: `Windows.Devices.Bluetooth` (UWP/WinRT) 또는 `InTheHand.Bluetooth` 패키지 활용
- **Linux**: `Tmds.DBus`로 BlueZ DBus API 호출
- **macOS**: CoreBluetooth (Objective-C) 바인딩 필요, `InTheHand.Bluetooth`가 지원 가능

의존성 주입으로 등록한다.

```csharp
if (OperatingSystem.IsWindows())
    services.AddSingleton<IBluetoothService, WindowsBleService>();
else if (OperatingSystem.IsLinux())
    services.AddSingleton<IBluetoothService, BluezBleService>();
else if (OperatingSystem.IsMacOS())
    services.AddSingleton<IBluetoothService, MacCoreBluetoothService>();
```

## Avalonia MVVM과 네이티브 API 통합

### UI 스레드 주의

네이티브 콜백은 별도 스레드에서 발생하므로 UI 업데이트는 반드시 `Dispatcher.UIThread`를 사용해야 한다.

```csharp
using Avalonia.Threading;

void OnNativeDataReceived(string data)
{
    Dispatcher.UIThread.Post(() =>
    {
        // ViewModel의 ObservableCollection 등 갱신
        Logs.Add(data);
    });
}
```

### 인터페이스 기반 서비스 + ViewModel

아래는 시리얼 포트를 예로 든 ViewModel이다.

```csharp
public interface ISerialService : IAsyncDisposable
{
    Task OpenAsync(string port, int baud, CancellationToken ct);
    Task CloseAsync();
    Task WriteLineAsync(string line, CancellationToken ct);
    IAsyncEnumerable<string> ReadLinesAsync(CancellationToken ct);
}

public sealed class SerialViewModel : ReactiveObject
{
    private readonly ISerialService _serial;
    private readonly ObservableCollection<string> _logs = new();
    public ReadOnlyObservableCollection<string> Logs { get; }

    public string Port { get; set; } = "COM3";
    public int Baud { get; set; } = 9600;

    public ReactiveCommand<Unit, Unit> OpenCommand { get; }
    public ReactiveCommand<Unit, Unit> CloseCommand { get; }
    public ReactiveCommand<string, Unit> SendCommand { get; }

    public SerialViewModel(ISerialService serial)
    {
        _serial = serial;
        Logs = new(_logs);

        OpenCommand = ReactiveCommand.CreateFromTask(OpenAsync);
        CloseCommand = ReactiveCommand.CreateFromTask(() => _serial.CloseAsync());
        SendCommand = ReactiveCommand.CreateFromTask<string>(SendAsync);
    }

    private async Task OpenAsync()
    {
        var cts = new CancellationTokenSource();
        await _serial.OpenAsync(Port, Baud, cts.Token);

        _ = Task.Run(async () =>
        {
            await foreach (var line in _serial.ReadLinesAsync(cts.Token))
            {
                Dispatcher.UIThread.Post(() => _logs.Add(line));
            }
        });
    }

    private async Task SendAsync(string line)
    {
        await _serial.WriteLineAsync(line, CancellationToken.None);
    }
}
```

## 네이티브 라이브러리 배포 (Self-contained)

자체 포함(self-contained) 배포 시 네이티브 DLL/so/dylib를 올바른 경로에 두어야 한다.

프로젝트 구조 예:

```
MyApp/
  runtimes/
    win-x64/native/myapi.dll
    linux-x64/native/libmyapi.so
    osx-x64/native/libmyapi.dylib
```

.csproj에 포함:

```xml
<ItemGroup>
  <Content Include="runtimes\win-x64\native\myapi.dll" PackagePath="runtimes\win-x64\native" />
  <Content Include="runtimes\linux-x64\native\libmyapi.so" PackagePath="runtimes\linux-x64\native" />
  <Content Include="runtimes\osx-x64\native\libmyapi.dylib" PackagePath="runtimes\osx-x64\native" />
</ItemGroup>
```

## 권한 및 보안

| 운영체제 | 주의점 |
|----------|--------|
| Windows | 일부 WinRT API는 앱 매니페스트에 능력(Capability) 선언 필요. UAC 권한 상승 |
| Linux | 디바이스 파일 접근 권한 (예: `dialout` 그룹), udev 규칙 |
| macOS | Info.plist에 권한 설명 문자열 추가 (예: `NSBluetoothAlwaysUsageDescription`), 앱 서명 및 공증(notarization) |

## 테스트 전략

- **인터페이스 기반**: 실제 하드웨어 없이 Mock/Fake 서비스를 만들어 ViewModel을 단위 테스트한다.
- **통합 테스트**: 실제 장치 또는 시뮬레이터가 있는 환경에서 실행한다.
- **리소스 누수 검사**: `SafeHandle`과 `IDisposable` 구현이 올바른지 확인한다.

## 디버깅 팁

- `DllNotFoundException` → 파일 존재, RID 일치, 경로 확인
- `EntryPointNotFoundException` → 함수 이름 철자, 호출 규약, 문자셋 점검
- 구조체 마샬링 오류 → `LayoutKind`, `Size`, `Pack` 속성 확인

## 수식 예: 직렬 버퍼 계산

보드레이트 \(B\)에서 초당 전송 가능한 바이트 수는 (시작/정지 비트 포함) 대략:

$$
\text{bytes\_per\_sec} \approx \frac{B}{10}
$$

예를 들어 \(B=115200\) 이면 초당 약 11520 바이트, 1024바이트 청크를 안전하게 보내려면 간격을 약 89ms 이상 유지하는 것이 좋다.

## 결론

- 간단한 네이티브 호출은 P/Invoke로 충분하지만 복잡한 기능(블루투스, 센서)은 플랫폼별 구현을 인터페이스로 추상화해야 한다.
- Avalonia UI와의 통합은 DI, ReactiveUI, Dispatcher를 활용해 MVVM 패턴을 유지한다.
- 배포 시 네이티브 라이브러리를 RID별로 포함하고, 권한 문제를 사전에 점검한다.
- 테스트 용이성을 위해 인터페이스 기반 설계를 기본으로 한다.

이러한 구조를 따르면 Avalonia 앱에서 네이티브 기능을 안정적으로 사용하면서도 코드 유지보수성과 테스트성을 확보할 수 있다.