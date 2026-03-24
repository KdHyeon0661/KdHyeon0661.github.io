---
layout: post
title: Avalonia - 모바일 대응 및 PWA 내장 전략
date: 2025-03-15 20:20:23 +0900
category: Avalonia
---
# Avalonia의 모바일 대응 및 PWA 내장 전략

Avalonia는 데스크톱(Windows, macOS, Linux)에서 안정적인 크로스 플랫폼 UI 프레임워크다. 하지만 모바일(Android, iOS)과 웹(PWA) 지원은 아직 성숙 단계가 아니다. 이 글에서는 **지금 당장 실무에 적용 가능한 전략**으로, 데스크톱을 중심으로 하면서도 **코어 재사용**을 통해 모바일과 웹을 부분적으로 활용하는 방법을 소개한다.

## 지원 현황과 전략 개요

| 플랫폼 | 공식 지원 상태 | 현실적인 전략 |
|--------|----------------|----------------|
| Windows / macOS / Linux | 안정적 | 순수 Avalonia로 개발 및 배포 |
| Android / iOS | 실험적 (Avalonia.Mobile) | 소규모 검증용으로 사용, 입력/제스처/성능 테스트 필수 |
| Web (PWA) | 정식 지원 없음 | 데스크톱 앱 내에 **WebView**를 내장하고, 로컬에 번들링한 PWA를 로드하는 하이브리드 방식 |

핵심 아이디어는 **비즈니스 로직과 ViewModel을 공통 라이브러리(`MyApp.Core`)**에 모으고, 각 플랫폼용 프로젝트는 얇게 유지하는 것이다. 웹 기능은 내장 WebView 위에서 실행되는 **하나의 웹 UI 번들**로 통합해, 데스크톱과 모바일에서 같은 웹 화면을 재사용한다.

## 프로젝트 구조 예시

```
MySuite/
├── src/
│   ├── MyApp.Core/                  # 공통 ViewModel, 서비스, 모델 (net8.0)
│   ├── MyApp.Desktop/               # 데스크톱 호스트 (Avalonia)
│   ├── MyApp.Mobile/                # Android/iOS 호스트 (실험적)
│   └── MyApp.WebHost/               # WebView를 포함한 데스크톱 호스트 (PWA 내장)
└── web/
    ├── index.html
    ├── manifest.webmanifest
    ├── sw.js                        # Service Worker
    ├── styles.css
    └── app.js
```

## 공통 코어(Core) 설계

`MyApp.Core`는 UI 프레임워크에 의존하지 않는 순수 .NET 라이브러리다. ViewModel과 서비스 인터페이스, 모델을 이곳에 배치한다.

```csharp
// MyApp.Core/ViewModels/DashboardViewModel.cs
public class DashboardViewModel : ReactiveObject
{
    private string _title = "대시보드";
    public string Title
    {
        get => _title;
        set => this.RaiseAndSetIfChanged(ref _title, value);
    }

    public ObservableCollection<double> Series { get; } = new();

    public void Tick(double value) => Series.Add(value);
}
```

## 데스크톱 앱에 WebView 내장하기

Avalonia 공식 WebView 패키지를 사용해 데스크톱 앱 안에 브라우저를 띄운다. 로컬에 위치한 PWA 번들(`web/` 폴더)을 `file://` 스킴으로 로드한다.

### 패키지 추가

```bash
dotnet add package Avalonia.WebView.Desktop
```

### XAML: WebView 배치

```xml
<!-- MyApp.WebHost/Views/WebShellView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:wv="clr-namespace:Avalonia.WebView;assembly=Avalonia.WebView">
  <Grid RowDefinitions="Auto,*">
    <StackPanel Orientation="Horizontal" Spacing="8" Margin="8">
      <Button Content="새로고침" Command="{Binding ReloadCommand}"/>
      <Button Content="홈" Command="{Binding HomeCommand}"/>
      <TextBlock Text="{Binding Status}" Margin="8,0,0,0"/>
    </StackPanel>
    <wv:WebView Grid.Row="1" x:Name="Web" Source="{Binding CurrentUri}"/>
  </Grid>
</UserControl>
```

### ViewModel: 로컬 파일 경로 바인딩

```csharp
// MyApp.WebHost/ViewModels/WebShellViewModel.cs
public class WebShellViewModel : ReactiveObject
{
    private Uri _currentUri = new Uri("file:///" + Path.GetFullPath("web/index.html"));
    public Uri CurrentUri
    {
        get => _currentUri;
        set => this.RaiseAndSetIfChanged(ref _currentUri, value);
    }

    public ReactiveCommand<Unit, Unit> ReloadCommand { get; }
    public ReactiveCommand<Unit, Unit> HomeCommand { get; }

    public WebShellViewModel()
    {
        ReloadCommand = ReactiveCommand.Create(() =>
            CurrentUri = new Uri(CurrentUri.ToString())); // 간단히 같은 Uri 재할당
        HomeCommand = ReactiveCommand.Create(() =>
            CurrentUri = new Uri("file:///" + Path.GetFullPath("web/index.html")));
    }
}
```

> 배포 시 `web/` 폴더를 실행 파일과 함께 복사하거나, **임베디드 리소스**로 포함해 최초 실행 시 사용자 디렉터리에 풀어내는 방식을 쓸 수 있다.

## JS ↔ .NET 양방향 통신 (메시지 브리지)

웹 화면과 네이티브 코드가 서로 데이터를 주고받으려면 **메시지 브리지**가 필요하다. WebView2(Windows)나 CEF 등 엔진마다 API가 다르므로, 인터페이스로 추상화한다.

### 인터페이스 정의

```csharp
// MyApp.Core/Services/IWebBridge.cs
public interface IWebBridge
{
    IObservable<string> Messages { get; }
    void PostJson(object payload);
}
```

### WebViewBridge 구현 (Avalonia.WebView 기준)

```csharp
// MyApp.WebHost/Services/WebViewBridge.cs
public class WebViewBridge : IWebBridge
{
    private readonly Subject<string> _messages = new();
    public IObservable<string> Messages => _messages;

    private readonly WebView _webView;

    public WebViewBridge(WebView webView)
    {
        _webView = webView;
        // WebView에서 제공하는 메시지 이벤트 구독
        _webView.WebView?.WebMessageReceived += (s, e) =>
            _messages.OnNext(e.Message);
    }

    public void PostJson(object payload)
    {
        var json = JsonSerializer.Serialize(payload);
        _webView.WebView?.PostWebMessageAsString(json);
    }
}
```

> 실제 WebView 컨트롤에 접근하려면 XAML에서 `x:Name`을 지정하고 코드 비하인드에서 연결해야 한다. 이 예시는 개념 전달용이다.

### ViewModel에서 브리지 사용

```csharp
public class HybridDashboardViewModel : ReactiveObject
{
    private readonly IWebBridge _bridge;
    private readonly DashboardViewModel _core;

    public HybridDashboardViewModel(IWebBridge bridge, DashboardViewModel core)
    {
        _bridge = bridge;
        _core = core;

        _bridge.Messages.Subscribe(OnWebMessage);
        StartFeeder(); // 가상 데이터 피드
    }

    private void OnWebMessage(string raw)
    {
        var msg = JsonSerializer.Deserialize<Dictionary<string, object>>(raw);
        if (msg?["type"] as string == "ack")
        {
            // 웹에서 확인 응답 받음
        }
    }

    private void StartFeeder()
    {
        var rnd = new Random();
        Observable.Interval(TimeSpan.FromSeconds(1))
                  .Subscribe(_ =>
                  {
                      var val = rnd.NextDouble() * 100;
                      _core.Tick(val);
                      _bridge.PostJson(new { type = "chart-data", value = val });
                  });
    }
}
```

### JS 측: 메시지 송수신

```html
<script>
  // 호스트로 메시지 보내기 (엔진에 따라 API 이름이 다를 수 있음)
  function sendToHost(payload) {
    if (window.chrome?.webview) {
      window.chrome.webview.postMessage(JSON.stringify(payload));
    } else if (window.external?.sendMessage) {
      window.external.sendMessage(JSON.stringify(payload));
    } else {
      console.warn('Host messaging API not available');
    }
  }

  // 호스트에서 메시지 수신
  window.addEventListener('message', ev => {
    const msg = ev.data;
    if (msg.type === 'chart-data') {
      updateChart(msg.value);
      sendToHost({ type: 'ack' });
    }
  });
</script>
```

## 반응형 레이아웃과 모바일 대응

모바일 환경에서는 화면 크기, 터치 입력, DPI 변화를 고려해야 한다.

### 화면 크기에 따른 분기

ViewModel에 창 너비를 바인딩하고, 너비에 따라 `IsMobile` 플래그를 계산한다.

```csharp
public class RootViewModel : ReactiveObject
{
    private double _windowWidth;
    public double WindowWidth
    {
        get => _windowWidth;
        set => this.RaiseAndSetIfChanged(ref _windowWidth, value);
    }

    public bool IsMobile => WindowWidth < 720;
    public bool IsDesktop => !IsMobile;
}
```

View에서 너비를 추적해 ViewModel에 전달한다.

```csharp
// RootView.axaml.cs
this.AttachedToVisualTree += (_, _) =>
{
    var win = this.VisualRoot as Window;
    if (win == null) return;
    win.GetObservable(Window.BoundsProperty)
        .Select(b => b.Width)
        .Subscribe(w => (DataContext as RootViewModel)!.WindowWidth = w);
};
```

XAML에서는 `IsVisible`로 두 가지 레이아웃을 전환한다.

```xml
<Grid>
    <local:DesktopLayout IsVisible="{Binding IsDesktop}"/>
    <local:MobileLayout  IsVisible="{Binding IsMobile}"/>
</Grid>
```

### 터치 대응

- 버튼과 탭 영역은 최소 44×44pt 크기를 권장한다.
- 스크롤 뷰어에 `AllowOverscroll` 같은 속성을 검토한다.
- 제스처(스와이프, 핀치)는 `Pointer` 이벤트를 직접 계산하거나 라이브러리를 사용한다.

### DPI 및 이미지 스케일

- 아이콘은 폰트 아이콘(예: Material Icons)이나 벡터 그래픽을 사용한다.
- 래스터 이미지는 1x, 2x, 3x 해상도를 분기해 제공한다.

## 모바일 호스트 (Android/iOS)

Avalonia.Mobile는 아직 실험 단계이므로, 전체 기능을 구현하기보다는 **WebView를 띄워 웹 UI로 대체**하는 접근이 현실적이다. 네이티브 권한(카메라, 파일)이 필요하다면 해당 부분만 네이티브 코드로 처리한다.

### Android 호스트 예시 (프로젝트 파일)

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0-android</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Avalonia" Version="11.0.0" />
    <PackageReference Include="Avalonia.Android" Version="11.0.0" />
    <ProjectReference Include="..\MyApp.Core\MyApp.Core.csproj" />
  </ItemGroup>
</Project>
```

### 조건부 컴파일

플랫폼별로 다른 코드를 작성하려면 `#if` 전처리기를 사용한다.

```csharp
public static class Platform
{
#if ANDROID
    public static string Name => "Android";
#elif IOS
    public static string Name => "iOS";
#else
    public static string Name => "Desktop";
#endif
}
```

## 내장 PWA: 오프라인 캐시와 Service Worker

WebView로 로컬 HTML을 로드할 때 **Service Worker**를 활용하면 오프라인에서도 웹 UI가 동작한다. `web/` 폴더에 `manifest.webmanifest`와 `sw.js`를 추가한다.

### manifest.webmanifest

```json
{
  "name": "MyApp Embedded PWA",
  "short_name": "MyApp",
  "display": "standalone",
  "start_url": "./index.html",
  "icons": [
    { "src": "./icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "./icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

### sw.js (Service Worker)

```js
const CACHE_NAME = 'myapp-cache-v1';
const ASSETS = [
  './index.html',
  './manifest.webmanifest',
  './styles.css',
  './app.js'
];

self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE_NAME).then(c => c.addAll(ASSETS)));
});

self.addEventListener('activate', e => {
  e.waitUntil(
    caches.keys().then(keys =>
      Promise.all(keys.filter(k => k !== CACHE_NAME).map(k => caches.delete(k)))
    )
  );
});

self.addEventListener('fetch', e => {
  e.respondWith(caches.match(e.request).then(r => r || fetch(e.request)));
});
```

### index.html에 등록

```html
<link rel="manifest" href="manifest.webmanifest">
<script>
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('./sw.js');
  }
</script>
```

이제 WebView가 `file://.../index.html`을 로드하면 Service Worker가 자산을 캐시하고, 인터넷 연결 없이도 UI를 표시할 수 있다.

## 보안 고려사항 (하이브리드)

- OAuth 로그인은 WebView로 진행하고, 커스텀 URI 스킴(`myapp://`)으로 토큰을 받는다.
- 토큰은 **OS 보안 저장소**에 저장한다 (Windows DPAPI, macOS Keychain, Android Keystore).
- JS ↔ .NET 메시지는 화이트리스트 기반 타입만 허용하고, 필요시 Nonce나 서명을 추가한다.

## 배포와 업데이트

- 데스크톱: `dotnet publish -c Release -r win-x64 --self-contained true`
- 웹 번들(`web/` 폴더)을 실행 파일과 함께 배포하거나, 리소스로 포함한다.
- 자동 업데이트는 데스크톱 파트(Squirrel, ClickOnce 등)로 처리하고, 웹 파트는 `sw.js` 버전을 바꿔 캐시를 갱신한다.

## 리스크와 대응 전략

| 리스크 | 대응 |
|--------|------|
| 모바일 렌더링/입력 불안정 | WebView 위주로 UI를 구성하고, 네이티브 화면은 최소화 |
| WebView 엔진 차이 | 공통 HTML/CSS 기능만 사용, 브라우저 호환성 테스트 |
| 오프라인 캐시 동기화 | 앱 버전 변경 시 Service Worker 버전 증가 및 캐시 삭제 |
| 유지보수 복잡성 | 코어 라이브러리에 대부분 로직을 두고, 플랫폼별 코드는 최소화 |

## 결론

Avalonia는 데스크톱 애플리케이션 개발에 매우 적합하지만, 모바일과 웹 지원은 아직 성숙하지 않다. 따라서 **실용적인 접근법**으로는:

1. **공통 비즈니스 로직**을 순수 .NET 라이브러리로 분리한다.
2. 데스크톱 앱에는 **WebView**를 내장해 웹 UI(PWA)를 표시하고, JS ↔ .NET 브리지로 상호작용한다.
3. 모바일에서는 같은 WebView 기반 UI를 재사용하고, 필수적인 네이티브 기능만 조건부로 추가한다.
4. Service Worker로 오프라인 캐시를 구성해 웹 UI가 네트워크 없이도 동작하게 한다.

이 전략을 따르면 **코드 재사용성을 극대화**하면서도, 현 시점에서 모바일과 웹을 포함한 멀티 플랫폼 대응이 가능하다. 향후 Avalonia가 모바일과 웹을 정식 지원하게 되면, 핵심 코어를 그대로 두고 호스트 프로젝트만 교체하는 식으로 자연스럽게 전환할 수 있다.