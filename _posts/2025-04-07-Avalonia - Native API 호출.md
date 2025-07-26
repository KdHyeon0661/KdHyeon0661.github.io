---
layout: post
title: Avalonia - Native API 호출
date: 2025-04-07 21:20:23 +0900
category: Avalonia
---
# 🧩 Avalonia에서 Native API 호출  
## (예: Bluetooth, Serial 통신, OS 기능 호출)

---

## 🎯 언제 필요한가?

| 예시 | 설명 |
|------|------|
| 🔵 Bluetooth 연결 | OS 레벨 블루투스 API 사용 필요 |
| 🔌 Serial (COM) 통신 | Win32 API or `/dev/tty*` 제어 |
| 🗂️ 파일 탐색기 / 권한 요청 | 네이티브 파일 선택기, macOS 보안 권한 |
| 🔈 오디오 제어, 진동 | OS API나 드라이버 연동 필요 |

> Avalonia는 순수 C# 크로스 플랫폼이므로 OS 고유 기능을 사용하려면 **Native API 연동**이 필요합니다.

---

## 1️⃣ 기본 방식: P/Invoke (Platform Invocation Services)

.NET에서는 Windows/Linux/macOS의 네이티브 함수를 직접 호출할 수 있는 `DllImport` 구문을 제공합니다.

### ✔️ Windows 예시: Win32 API 호출 (예: MessageBox)

```csharp
using System.Runtime.InteropServices;

public class NativeWin32
{
    [DllImport("user32.dll", CharSet = CharSet.Unicode)]
    public static extern int MessageBox(IntPtr hWnd, string text, string caption, uint type);
}
```

```csharp
NativeWin32.MessageBox(IntPtr.Zero, "Hello from native!", "Title", 0);
```

---

## 2️⃣ Linux/macOS 연동: C 라이브러리 호출

```csharp
[DllImport("libc")]
private static extern int getpid();

Console.WriteLine($"Current PID: {getpid()}");
```

- `libc`: 리눅스 및 macOS에서 사용되는 표준 C 라이브러리

> 📌 대부분은 `IntPtr`, `struct`, `MarshalAs`, `Span<T>` 등으로 데이터 형변환이 필요

---

## 3️⃣ 플랫폼 구분 처리

```csharp
if (OperatingSystem.IsWindows())
{
    NativeWin32.MessageBox(IntPtr.Zero, "윈도우입니다", "Info", 0);
}
else if (OperatingSystem.IsLinux())
{
    Console.WriteLine("리눅스에서 실행 중");
}
```

---

## 4️⃣ 실제 시나리오: 시리얼 포트 연동

### 🔧 방법 1: .NET 표준 `SerialPort` 사용

```csharp
using System.IO.Ports;

var port = new SerialPort("COM3", 9600);
port.Open();
port.WriteLine("Hello");
string response = port.ReadLine();
port.Close();
```

- Windows, Linux (`/dev/ttyUSB0`) 모두 지원
- Avalonia UI + SerialPort 연동으로 장치 제어 가능

---

## 5️⃣ 실전: Bluetooth API 호출 (Windows)

```csharp
[DllImport("bthprops.cpl", CharSet = CharSet.Unicode)]
public static extern int BluetoothIsConnectable(IntPtr hRadio, out bool isConnectable);
```

> 대부분 Bluetooth, HID, USB 등은 Native API 또는 외부 C++ 라이브러리 래핑이 필요합니다.

---

## 6️⃣ Avalonia.Platform 기반 구조화

Avalonia는 자체적으로 **플랫폼 추상화를 위한 인터페이스 구조**를 갖추고 있습니다.

### 예: IClipboard

```csharp
var clipboard = Avalonia.Application.Current.Clipboard;
await clipboard.SetTextAsync("클립보드에 복사됨");
```

→ 내부적으로 플랫폼별 Clipboard 구현을 `IAvaloniaPlatform`으로 분기함

---

### 직접 사용자 구현도 가능

```csharp
public interface IBluetoothService
{
    Task<IEnumerable<BluetoothDevice>> GetPairedDevicesAsync();
}
```

- Windows: Win32 호출로 구현
- Linux: `bluez` DBus 호출로 구현
- macOS: CoreBluetooth 연동

이후 Avalonia DI 시스템에 등록:

```csharp
services.AddSingleton<IBluetoothService, WindowsBluetoothService>();
```

> → 프로젝트 구동 시 OS별 적절한 구현 선택

---

## 7️⃣ Native 라이브러리 번들링

| OS | 예시 |
|----|------|
| Windows | `.dll` |
| Linux | `.so` |
| macOS | `.dylib` |

출시 시 `.csproj`에 포함:

```xml
<ItemGroup>
  <None Include="native/windows/myapi.dll">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>
</ItemGroup>
```

---

## 8️⃣ C++ 라이브러리 Wrapping (.NET interop)

- 경우에 따라서는 기존 C/C++ 라이브러리를 직접 호출해야 함
- C++/CLI 또는 C API wrapper → DLL → P/Invoke

---

## 9️⃣ 주의 사항

| 항목 | 설명 |
|------|------|
| ⚠️ Cross-platform 문제 | 플랫폼별 구현이 필요하므로 추상화 구조 권장 |
| ⚠️ UI Thread 접근 | 네이티브 호출 결과가 UI 변경 시 Dispatcher 사용 필요 |
| 🛡 보안 | Win/mac에서 일부 네이티브 호출은 권한 필요 (Bluetooth 등) |

---

## ✅ 정리

| 기술 | 사용 예 | Avalonia 연동 |
|------|---------|----------------|
| `P/Invoke` | Win32, libc, 사용자 DLL | Native 호출 |
| `SerialPort` | COM, USB | 장치 연동 |
| `Avalonia.Platform` | 플랫폼별 서비스 추상화 | 인터페이스/DI |
| 외부 라이브러리 | OpenCV, ffmpeg 등 | 번들링 + 호출 |
| `Dispatcher.UIThread` | UI 변경 시 필요 | 중요! |

---

## 📦 예제: BluetoothService 추상화

```csharp
public interface IBluetoothService
{
    Task<List<string>> GetDevicesAsync();
}

public class WindowsBluetoothService : IBluetoothService
{
    public Task<List<string>> GetDevicesAsync()
    {
        // P/Invoke 호출 + Marshal 처리
    }
}
```

```csharp
services.AddSingleton<IBluetoothService, WindowsBluetoothService>();
```

> Avalonia에서는 추상화 계층을 잘 구성하면 OS 종속 코드를 격리할 수 있음

---

## 📚 참고 자료

- [.NET P/Invoke 공식 문서](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/)
- [InTheHand.Net.Bluetooth 라이브러리](https://github.com/inthehand/32feet) – 크로스 플랫폼 Bluetooth 지원
- [Avalonia.Platform](https://github.com/AvaloniaUI/Avalonia/tree/master/src/Avalonia.Base/Platform)

---

## 🔚 결론

Avalonia에서 네이티브 기능을 호출하려면:

- ✅ 단순 기능: `P/Invoke`로 호출
- ✅ 고급 기능: 외부 라이브러리 또는 DBus 연동
- ✅ 구조화: `Avalonia.Platform` 또는 DI 기반 추상화
- ✅ 주의: UI 연동 시 Dispatcher 사용