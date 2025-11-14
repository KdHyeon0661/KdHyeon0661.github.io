---
layout: post
title: Avalonia - Native API 호출
date: 2025-04-07 21:20:23 +0900
category: Avalonia
---
# Avalonia에서 Native API 호출 심화 가이드

## 언제 Native API가 필요한가?

| 예시 | 사용 계층 | 설명 |
|------|-----------|------|
| Bluetooth 검색/연결 | Win32(BTH), BlueZ(DBus), CoreBluetooth | 스택이 OS별로 완전히 다름 |
| Serial(COM/tty) | .NET `SerialPort`(관리 래퍼) + OS | 장치 이름 규칙/권한/보안 고려 |
| 파일 탐색/권한 | Win32(공유 대화상자), macOS 권한, Linux 포털 | Avalonia `StorageProvider`로 대체 가능하나 네이티브 제어가 필요할 수 있음 |
| 오디오/진동 | WASAPI/CoreAudio/ALSA/FFI | 앱/장치 특화 기능 노출 |
| 센서/USB/HID | Win32 SetupAPI, libudev, IOKit | 일반적으로 C/C++ 라이브러리 래핑 필요 |

핵심 아이디어는 **“플랫폼 의존 코드를 인터페이스로 격리”**하고, **OS별 구현체**를 주입하여 Avalonia UI와 **느슨하게 결합**하는 것입니다.

---

## 기본: P/Invoke(Platform Invocation)로 C API 호출

### Win32 예시: `MessageBoxW`

```csharp
using System;
using System.Runtime.InteropServices;

internal static class Win32
{
    [DllImport("user32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
    public static extern int MessageBoxW(IntPtr hWnd, string text, string caption, uint type);
}

public static class Demo
{
    public static void ShowNativeBox()
    {
        Win32.MessageBoxW(IntPtr.Zero, "Hello from Win32!", "Title", 0 /* MB_OK */);
    }
}
```

- `CharSet.Unicode`를 지정해 **W API**(UTF-16)를 호출합니다.
- Win32에서 오류 진단 시 `Marshal.GetLastWin32Error()`를 사용합니다.

### POSIX 예시: `getpid()` (Linux/macOS)

```csharp
using System;
using System.Runtime.InteropServices;

internal static class LibC
{
    [DllImport("libc", EntryPoint = "getpid")]
    public static extern int GetPid();
}

Console.WriteLine($"PID: {LibC.GetPid()}");
```

- macOS와 Linux 모두 `libc`에서 동작합니다.
- 많은 API에서 포인터/구조체 마샬링이 필요합니다.

### 구조체/문자열/버퍼 마샬링 패턴

```csharp
[StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
public struct MyNativeInfo
{
    public int Size;
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 260)]
    public string Path;
}
```

- 고정 길이 문자열, 배열, 포인터(`IntPtr`)를 조합합니다.
- 네이티브에서 **할당/해제** 책임이 어디에 있는지 확실히 합니다.

---

## 플랫폼 분기와 안전한 자원 해제

### 플랫폼 감지

```csharp
if (OperatingSystem.IsWindows()) { /* Win32 경로 */ }
else if (OperatingSystem.IsLinux()) { /* Linux 경로 */ }
else if (OperatingSystem.IsMacOS()) { /* macOS 경로 */ }
```

### `SafeHandle`로 핸들 메모리 안전화

```csharp
public sealed class SafeNativeHandle : SafeHandle
{
    public SafeNativeHandle() : base(IntPtr.Zero, ownsHandle: true) { }
    public override bool IsInvalid => handle == IntPtr.Zero;

    [DllImport("kernel32.dll")]
    private static extern bool CloseHandle(IntPtr hObject);

    protected override bool ReleaseHandle() => CloseHandle(handle);
}
```

- 네이티브 핸들(파일/디바이스/메모리)의 **일관된 해제**를 보장합니다.
- 예외/조기 반환에도 릭이 나지 않습니다.

---

## 직결 가능한 케이스: .NET `SerialPort` (Windows/Linux/macOS)

### 최소 예제

```csharp
using System;
using System.IO.Ports;
using System.Threading.Tasks;

public sealed class SerialEcho
{
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
        Console.WriteLine($"Device: {response}");

        // 장시간 읽기: Task.Run으로 백그라운드 처리
        await Task.Run(() =>
        {
            while (port.IsOpen)
            {
                try
                {
                    var line = port.ReadLine();
                    // UI 갱신 필요 시 Dispatcher 사용 (아래 7장)
                    Console.WriteLine(line);
                }
                catch (TimeoutException) { /* 무시 */ }
            }
        });
    }
}
```

- Windows: `COM3`, Linux: `/dev/ttyUSB0` 또는 `/dev/ttyACM0`, macOS: `/dev/tty.*`.
- Linux/macOS에선 **권한**(`dialout` 그룹, udev 규칙)을 주의합니다.

### Avalonia MVVM 연동

- `ISerialService` 인터페이스로 감싸고 ViewModel에서 주입받아 UI와 분리합니다.
- 읽기 루프는 **취소 토큰**으로 종료 제어합니다.

---

## Bluetooth — 운영체제별 스택과 전략

Bluetooth는 **플랫폼별 스택이 완전히 다르므로** P/Invoke만으로 통일하기 어렵습니다. 대표 3가지:

- **Windows**: Win32 Bthprops / Device Enumeration / UWP BLE API(Desktop에서 호출은 제약)
- **Linux**: **BlueZ**(DBus); GATT/Adapter/Device는 DBus 인터페이스로 제어
- **macOS**: **CoreBluetooth**(Objective-C API); P/Invoke만으로는 번거롭고 **바인딩 라이브러리**가 현실적

### 크로스플랫폼 래퍼 사용(권장)

- **InTheHand.Bluetooth(32feet)**, 일부 시나리오에서 BLE 스캔/연결을 공통 API로 다룰 수 있습니다.
- 또는 Xamarin/MAUI 바인딩처럼 **플랫폼별 래퍼**를 만든 뒤 **공통 인터페이스**로 감쌉니다.

#### 예) InTheHand.Bluetooth 사용 스케치

```csharp
// Install-Package InTheHand.BluetoothLE
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using InTheHand.Bluetooth;

public interface IBluetoothService
{
    IAsyncEnumerable<BluetoothDeviceInfo> ScanAsync(CancellationToken ct);
}

public record BluetoothDeviceInfo(string Name, string Address);

public sealed class CrossPlatformBleService : IBluetoothService
{
    public async IAsyncEnumerable<BluetoothDeviceInfo> ScanAsync(
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct)
    {
        await foreach (var dev in Bluetooth.ScanForDevices(ct))
        {
            yield return new BluetoothDeviceInfo(dev.Name, dev.Id);
        }
    }
}
```

> 실제 BLE 특성 읽기/쓰기, 서비스/특성 UUID 브라우징까지 확장 가능합니다.

### Windows 네이티브 직접 접근(개요)

- 전통 Win32 BTH API(P/Invoke) or Windows.Devices.Bluetooth(BLE) 사용
  데스크톱에서 UWP/WinRT API 호출은 **권한/매니페스트 구성**이 필요.
- 실무에선 InTheHand 같은 래퍼가 초기 생산성을 크게 올립니다.

### Linux(BlueZ) — DBus 접근

- `org.bluez.Adapter1`, `org.bluez.Device1`, `org.bluez.GattCharacteristic1`를 DBus로 호출
- .NET에선 `Tmds.DBus` 같은 라이브러리 사용
- 장치 디스커버리/페어링/연결/특성 읽기/쓰기 = 모두 DBus 호출

```csharp
// Pseudo-code with Tmds.DBus
public interface IBluezAdapter
{
    Task StartDiscoveryAsync();
    Task StopDiscoveryAsync();
}
// 실제 구현은 DBus 프록시로 생성
```

### macOS(CoreBluetooth)

- Objective-C 런타임과 상호 운용 필요(바인딩 라이브러리 생성)
- P/Invoke로 직접 접근은 복잡(콜백 기반, 런루프, 스레딩)
- 크로스플랫폼 래퍼 또는 기존 바인딩 활용을 권장

---

## Avalonia와 네이티브 API의 **구조화**: 인터페이스 + DI + OS별 구현

### 공통 계약

```csharp
public interface IBluetoothService
{
    IAsyncEnumerable<BluetoothDeviceInfo> ScanAsync(CancellationToken ct);
    Task ConnectAsync(string id, CancellationToken ct);
    Task<byte[]> ReadAsync(Guid service, Guid characteristic, CancellationToken ct);
    Task WriteAsync(Guid service, Guid characteristic, byte[] data, CancellationToken ct);
}
```

### OS별 구현 등록

```csharp
// Program.cs or App bootstrap
if (OperatingSystem.IsWindows())
    services.AddSingleton<IBluetoothService, WindowsBleService>();
else if (OperatingSystem.IsLinux())
    services.AddSingleton<IBluetoothService, BluezBleService>();
else if (OperatingSystem.IsMacOS())
    services.AddSingleton<IBluetoothService, MacCoreBluetoothService>();
```

- Avalonia에서는 **DI 컨테이너(예: Microsoft.Extensions.DependencyInjection)**를 사용해 ViewModel과 느슨한 결합을 유지합니다.

---

## 네이티브 라이브러리 번들링/배포 (RID별 native assets)

**Self-contained** 배포에서 네이티브 의존성(DLL/.so/.dylib)을 함께 내보냅니다.

### 프로젝트 구조

```
MyApp/
  runtimes/
    win-x64/native/myapi.dll
    linux-x64/native/libmyapi.so
    osx-x64/native/libmyapi.dylib
```

### .csproj 설정

```xml
<ItemGroup>
  <NativeLibrary Include="runtimes/win-x64/native/myapi.dll" />
  <NativeLibrary Include="runtimes/linux-x64/native/libmyapi.so" />
  <NativeLibrary Include="runtimes/osx-x64/native/libmyapi.dylib" />
</ItemGroup>
```

- 또는 `None Include` + `CopyToOutputDirectory`로 간단 구성 가능하나, **RID 폴더 규칙**을 따르면 NuGet 배포에도 유리합니다.

### 로딩 이슈 트러블슈팅

- `DllNotFoundException` 발생 시 **파일 존재/경로/RID 일치** 확인
- Linux에서 `ldd`/`ldconfig`로 동적 링크 종속성 체크
- macOS는 **codesign/notarization** 주의

---

## UI 스레드와 네이티브 콜백 — Dispatcher 사용

네이티브 콜백/백그라운드 스레드에서 UI를 건드리면 크래시합니다.

```csharp
using Avalonia.Threading;

void OnDeviceLineArrived(string line)
{
    Dispatcher.UIThread.Post(() =>
    {
        // ViewModel ObservableCollection 갱신
        Logs.Add(line);
    });
}
```

- ReactiveUI 사용 시에도 **UI 컬렉션/속성 갱신은 UI 스레드**에서 처리합니다.

---

## 권한/보안/샌드박스

| OS | 고려 사항 |
|----|-----------|
| Windows | 일부 WinRT API(BLE 등)는 **앱 매니페스트/능력(Capability)** 필요. UAC/드라이버 서명/드라이버 설치 포함 |
| Linux | 디바이스 파일 권한(예: `/dev/tty*`), 그룹(`dialout`) 추가, udev 규칙 |
| macOS | **Info.plist**에 목적 문자열(예: `NSBluetoothAlwaysUsageDescription`) 추가, App Sandbox, Notarization, Hardened Runtime |

- 개인정보/민감 데이터(장치 번호, 사용자 식별자)는 **로그/업로드** 전에 마스킹/동의 절차를 거칩니다.

---

## 파일/폴더 대화 상자 — 네이티브 vs Avalonia

Avalonia는 `IStorageProvider`로 크로스플랫폼 파일 대화상자를 제공합니다(권장):

```csharp
var storage = TopLevel.GetTopLevel(this)!.StorageProvider;
var files = await storage.OpenFilePickerAsync(new FilePickerOpenOptions
{
    Title = "장치 설정 불러오기",
    AllowMultiple = false,
    FileTypeFilter = new[] { FilePickerFileTypes.Json }
});
```

**진짜 네이티브 대화상자**의 특수 옵션(보안 스코프, macOS Security Scoped Bookmark 등)이 필요하면 P/Invoke/바인딩이 들어가지만, 대부분은 `StorageProvider`로 충분합니다.

---

## 고급 상호운용: C/C++ 라이브러리 래핑

### C API Wrapper를 만들고 P/Invoke

- 기존 C++ 라이브러리를 **C API로 래핑**(extern "C") → `DllImport`로 안전 호출
- 예: OpenCV/FFI/전용 드라이버 SDK를 래핑 DLL로 노출

```c
// myapi.h
#ifdef __cplusplus

extern "C" {
#endif

int myapi_init();
int myapi_do_work(const char* in, char* out, int outLen);
void myapi_free(void* p);
#ifdef __cplusplus

}
#endif

```

```csharp
internal static class MyApi
{
    [DllImport("myapi", CallingConvention = CallingConvention.Cdecl)]
    public static extern int myapi_init();

    [DllImport("myapi", CallingConvention = CallingConvention.Cdecl)]
    public static extern int myapi_do_work(string input,
        StringBuilder output, int outLen);

    [DllImport("myapi", CallingConvention = CallingConvention.Cdecl)]
    public static extern void myapi_free(IntPtr p);
}
```

### C++/CLI는 Windows 전용

- 강력하지만 **Windows 빌드 전용**이며 크로스플랫폼 목표에서는 지양.

---

## 샘플: 공통 인터페이스 + OS별 구현 + Avalonia UI

### 계약

```csharp
public interface ISerialService : IAsyncDisposable
{
    Task OpenAsync(string port, int baud, CancellationToken ct);
    Task CloseAsync();
    Task WriteLineAsync(string line, CancellationToken ct);
    IAsyncEnumerable<string> ReadLinesAsync(CancellationToken ct);
}
```

### Windows/Linux 공통 구현(SerialPort)

```csharp
using System.IO.Ports;
using System.Threading.Channels;

public sealed class SerialService : ISerialService
{
    private SerialPort? _port;
    private Channel<string>? _ch;

    public async Task OpenAsync(string port, int baud, CancellationToken ct)
    {
        _port = new SerialPort(port, baud) { NewLine = "\r\n" };
        _port.Open();

        _ch = Channel.CreateUnbounded<string>();
        _ = Task.Run(() =>
        {
            while (!ct.IsCancellationRequested && _port!.IsOpen)
            {
                try
                {
                    var line = _port.ReadLine();
                    _ch.Writer.TryWrite(line);
                }
                catch { /* timeout 또는 종료 */ }
            }
            _ch.Writer.TryComplete();
        }, ct);

        await Task.CompletedTask;
    }

    public Task CloseAsync()
    {
        _port?.Close();
        return Task.CompletedTask;
    }

    public Task WriteLineAsync(string line, CancellationToken ct)
    {
        _port!.WriteLine(line);
        return Task.CompletedTask;
    }

    public async IAsyncEnumerable<string> ReadLinesAsync(
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct)
    {
        if (_ch is null) yield break;
        while (await _ch.Reader.WaitToReadAsync(ct))
            while (_ch.Reader.TryRead(out var line))
                yield return line;
    }

    public ValueTask DisposeAsync()
    {
        _port?.Dispose();
        return ValueTask.CompletedTask;
    }
}
```

### ViewModel (Avalonia + ReactiveUI)

```csharp
public sealed class SerialViewModel : ReactiveObject
{
    private readonly ISerialService _serial;
    private readonly ObservableCollection<string> _logs = new();
    public ReadOnlyObservableCollection<string> Logs { get; }

    public string Port { get => _port; set => this.RaiseAndSetIfChanged(ref _port, value); }
    public int Baud { get => _baud; set => this.RaiseAndSetIfChanged(ref _baud, value); }

    private string _port = "COM3";
    private int _baud = 9600;

    public ReactiveCommand<Unit, Unit> OpenCmd { get; }
    public ReactiveCommand<Unit, Unit> CloseCmd { get; }
    public ReactiveCommand<string, Unit> SendCmd { get; }

    public SerialViewModel(ISerialService serial)
    {
        _serial = serial;
        Logs = new ReadOnlyObservableCollection<string>(_logs);

        OpenCmd = ReactiveCommand.CreateFromTask(async () =>
        {
            var cts = new CancellationTokenSource();
            await _serial.OpenAsync(Port, Baud, cts.Token);

            _ = Task.Run(async () =>
            {
                await foreach (var line in _serial.ReadLinesAsync(cts.Token))
                {
                    // UI 스레드로 디스패치
                    Avalonia.Threading.Dispatcher.UIThread.Post(() => _logs.Add(line));
                }
            });
        });

        CloseCmd = ReactiveCommand.CreateFromTask(() => _serial.CloseAsync());
        SendCmd  = ReactiveCommand.CreateFromTask<string>(line => _serial.WriteLineAsync(line, CancellationToken.None));
    }
}
```

### View (XAML 스케치)

```xml
<StackPanel Spacing="8">
  <StackPanel Orientation="Horizontal" Spacing="8">
    <TextBox Width="120" Text="{Binding Port}"/>
    <NumericUpDown Width="100" Value="{Binding Baud}"/>
    <Button Content="Open" Command="{Binding OpenCmd}"/>
    <Button Content="Close" Command="{Binding CloseCmd}"/>
  </StackPanel>

  <ItemsControl Items="{Binding Logs}" />

  <StackPanel Orientation="Horizontal" Spacing="8">
    <TextBox x:Name="SendBox" Width="300"/>
    <Button Content="Send"
            Command="{Binding SendCmd}"
            CommandParameter="{Binding #SendBox.Text}"/>
  </StackPanel>
</StackPanel>
```

---

## 단위 테스트/모킹/시뮬레이터

- **인터페이스 기반**이므로 I/O 없는 **Fake/Mock** 구현으로 ViewModel 테스트가 쉽습니다.

```csharp
public sealed class FakeSerialService : ISerialService
{
    private readonly Channel<string> _ch = Channel.CreateUnbounded<string>();
    public Task OpenAsync(string p, int b, CancellationToken ct) => Task.CompletedTask;
    public Task CloseAsync() => Task.CompletedTask;
    public Task WriteLineAsync(string line, CancellationToken ct)
    {
        _ch.Writer.TryWrite($"ECHO:{line}");
        return Task.CompletedTask;
    }
    public async IAsyncEnumerable<string> ReadLinesAsync(
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct)
    {
        while (await _ch.Reader.WaitToReadAsync(ct))
            while (_ch.Reader.TryRead(out var s))
                yield return s;
    }
    public ValueTask DisposeAsync() => ValueTask.CompletedTask;
}
```

- Bluetooth도 **스캐너/장치 시뮬레이터**를 만들어 테스트 시간을 줄입니다.

---

## 성능/안정성/디버깅 팁

- **장시간 연결**: CancellationToken/타임아웃/재연결 루프를 설계합니다.
- **리소스 릭**: SafeHandle/IDisposable/DisposeAsync로 확실한 해제.
- **로깅**: `Serilog`로 네이티브 호출 전/후를 로깅하고 에러 코드를 기록.
- **권한 실패/장치 없음**: 사용자 친화적 메시지 + 가이드(그룹 추가, 권한 안내).
- **동시성**: 장치 I/O는 단일 전용 Task로 직렬화, UI 갱신은 Dispatcher로 제한.

---

## 보안/개인정보

- BLE 주소/장치 이름/시리얼 로그에 **개인정보/식별자**가 포함될 수 있습니다. 업로드 전 **마스킹** 및 **동의** 절차를 거칩니다.
- macOS 권한 문자열(예: Bluetooth) 누락 시 앱이 조용히 실패할 수 있습니다. 배포 파이프라인에서 **정적 검사**로 방지합니다.

---

## 문제해결 체크리스트

1. **`DllNotFoundException`**: RID/native 경로/파일 권한 확인
2. **`EntryPointNotFoundException`**: 함수 이름/콜링 컨벤션/문자셋 점검
3. **포인터 크래시**: 구조체 레이아웃/Pack/ByRef/ByVal 문자열 확인
4. **UI 프리징**: 백그라운드에서 I/O, UI는 Dispatcher로
5. **권한 거부(ENOENT/AccessDenied)**: udev, 그룹, Plist, 매니페스트 점검
6. **BLE 연결 불가**: 어댑터 전원, 페어링/보안 수준, 서비스/특성 권한 확인

---

## 미니 프로젝트 템플릿 구조

```
src/
  MyApp/                           # Avalonia UI (Views, ViewModels)
  MyApp.Core/                      # 계약/DTO/도메인 (IBluetoothService, ISerialService)
  MyApp.Platform.Windows/          # Win32/WinRT 구현
  MyApp.Platform.Linux/            # BlueZ(DBus) 구현
  MyApp.Platform.Mac/              # CoreBluetooth 바인딩/래퍼
  MyApp.Native/                    # C API Wrapper (필요 시)
  MyApp.Tests/                     # ViewModel/서비스 단위 테스트 (Fake)
```

- UI는 Core 인터페이스만 알도록 분리 → **테스트성 + 유지보수성** 상승.

---

## 수식이 필요한 경우(버퍼 스루풋/타이밍 계산 예)

예: 직렬 포트에서 버퍼 초과를 막기 위한 추정(보드레이트 \(B\), 바이트/초 \(\approx B/10\)).

$$
\text{bytes\_per\_sec} \approx \frac{B}{10}, \quad
\text{safe\_interval} \approx \frac{\text{chunk\_size}}{\text{bytes\_per\_sec}}
$$

예를 들어 \(B=115200\), chunk 1024 bytes라면 초당 \(\approx 11520\,\text{B/s}\), 안전 간격 \(\approx 89\,\text{ms}\).

---

## 결론

- **간단한 네이티브 호출**은 P/Invoke로 충분하지만, **Bluetooth/센서/멀티미디어**처럼 복잡한 영역은 **플랫폼별 스택을 이해**하고 **인터페이스로 추상화**하여 OS별 구현을 주입하는 전략이 필수입니다.
- 배포는 **RID별 native 자산**을 포함하고, 권한/보안을 프로젝트 초기부터 설계해야 합니다.
- Avalonia UI와의 접점은 **Dispatcher(UI 스레드)**, **DI**, **Reactive 패턴**으로 깨끗하게 유지하세요.
