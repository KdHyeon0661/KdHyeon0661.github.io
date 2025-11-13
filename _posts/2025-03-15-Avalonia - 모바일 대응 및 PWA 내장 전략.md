---
layout: post
title: Avalonia - 모바일 대응 및 PWA 내장 전략
date: 2025-03-15 20:20:23 +0900
category: Avalonia
---
# Avalonia의 모바일 대응 및 PWA 내장 전략

## 1. 지원 현황 요약과 프로젝트 전략

| 플랫폼 | 현재 상태 | 실전 권장 전략 |
|---|---|---|
| Windows/macOS/Linux | 안정적 | 순수 Avalonia로 구현/배포 |
| Android/iOS | 실험적(Avalonia.Mobile) | 멀티타겟팅 + 조건부 컴파일로 “시도” 수준. 입력/제스처/성능 검증 필수 |
| Web(PWA) | 정식 미지원 | 데스크톱 앱에 **WebView** 내장, 로컬 웹앱(PWA) 바인딩으로 “하이브리드” 접근 |

핵심 구조 아이디어:
- **UI/도메인 코어**(ViewModel, Services, Domain)는 `net8.0` 클래스 라이브러리에 집약.
- 데스크톱/모바일/웹뷰 호스트는 **얇은 플랫폼 프로젝트**로 분리, `#if` 전처리로 기능 차등화.
- 웹 기능은 **내장 WebView**(데스크톱/모바일 모두) 위에서 공통 JS 번들 사용 → “한 벌의 웹 UI”를 각 플랫폼에서 재생.

---

## 2. 솔루션 구조(멀티 타겟) 예시

```
MySuite/
├─ src/
│  ├─ MyApp.Core/                    # ViewModel/Domain/Services (net8.0)
│  ├─ MyApp.Desktop/                 # Avalonia 데스크톱 호스트 (win/mac/linux)
│  ├─ MyApp.Mobile/                  # Android/iOS 호스트(실험적)
│  └─ MyApp.WebHost/                 # WebView+내장 PWA 리더(데스크톱 우선)
└─ web/
   ├─ index.html
   ├─ manifest.webmanifest
   └─ sw.js                          # Service Worker(오프라인 캐시)
```

### 2.1 공통 코드 예시 (ViewModel)

```csharp
// MyApp.Core/ViewModels/DashboardViewModel.cs
public class DashboardViewModel : ReactiveUI.ReactiveObject
{
    private string _title = "대시보드";
    public string Title
    {
        get => _title;
        set => this.RaiseAndSetIfChanged(ref _title, value);
    }

    public ObservableCollection<double> Series { get; } = new();

    public void Tick(double v) => Series.Add(v);
}
```

---

## 3. 데스크톱: WebView 하이브리드(내장 웹앱) 설계

> 정식 Web(PWA) 실행은 아직 이르므로, **데스크톱 앱 안에 WebView**를 넣고 로컬에 번들링한 웹앱을 **파일 또는 커스텀 스킴**으로 로드하는 접근이 가장 실용적이다.

### 3.1 패키지 설치

```bash
dotnet add package Avalonia.WebView.Desktop
```

> 구현체에 따라 WebView2(Windows)·CEF(크로스) 등이 사용된다. 프로젝트에 맞는 변형 패키지를 선택한다.

### 3.2 XAML: WebView 자리 배치

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

    <wv:WebView Grid.Row="1"
                x:Name="Web"
                Source="{Binding CurrentUri}"/>
  </Grid>
</UserControl>
```

### 3.3 Code-behind: 로컬 파일/커스텀 스킴 로드

```csharp
// MyApp.WebHost/ViewModels/WebShellViewModel.cs
public class WebShellViewModel : ReactiveUI.ReactiveObject
{
    private Uri _currentUri = new("file:///" + Path.GetFullPath("web/index.html"));
    public Uri CurrentUri
    {
        get => _currentUri;
        set => this.RaiseAndSetIfChanged(ref _currentUri, value);
    }

    private string _status = "Ready";
    public string Status
    {
        get => _status;
        set => this.RaiseAndSetIfChanged(ref _status, value);
    }

    public ReactiveCommand<Unit, Unit> ReloadCommand { get; }
    public ReactiveCommand<Unit, Unit> HomeCommand { get; }

    public WebShellViewModel()
    {
        ReloadCommand = ReactiveCommand.Create(() => CurrentUri = new Uri(CurrentUri.ToString()));
        HomeCommand   = ReactiveCommand.Create(() =>
            CurrentUri = new Uri("file:///" + Path.GetFullPath("web/index.html")));
    }
}
```

> 배포 시 `web/` 폴더를 Publish 출력물과 함께 배치하거나, **임베디드 리소스**로 넣고 런타임에 임시 폴더로 복사해 로드한다.

---

## 4. JS ↔ .NET 브리지(양방향 메시징)

WebView 기반 하이브리드는 **로그인/결제/지도** 등 “웹으로 구현한 화면”과 “Avalonia 네이티브 화면”을 연결해야 한다.
대표적인 패턴은 **postMessage/MessageReceived** 이벤트 기반 브리지다.

### 4.1 JS 측(웹 번들)

```html
<!-- web/index.html 일부 -->
<script>
  // 앱으로 메시지 보내기
  function sendToHost(payload) {
    // 구현체별 메시지 API가 다를 수 있음: window.chrome.webview.postMessage, or externalHost, or custom
    if (window.chrome && window.chrome.webview) {
      window.chrome.webview.postMessage(JSON.stringify(payload));
    } else if (window.external && window.external.sendMessage) {
      window.external.sendMessage(JSON.stringify(payload));
    } else {
      console.warn('Host messaging API not available.');
    }
  }

  // 예: 버튼 클릭 → 호스트에 "refresh-data" 명령 전송
  function requestRefresh() {
    sendToHost({ type: 'refresh-data' });
  }

  // 호스트에서 JS로 푸시(예: 초기 상태 주입)
  window.addEventListener('message', (ev) => {
    const msg = ev.data;
    console.log('host->web', msg);
    // TODO: 그래프/상태 갱신
  });
</script>
```

### 4.2 .NET 측(메시지 수신/응답)

구현체마다 이벤트 명이 다르므로 “어댑터” 클래스로 추상화한다.

```csharp
public interface IWebBridge
{
    IObservable<string> Messages { get; }
    void PostJson(object payload);
}

public class WebViewBridge : IWebBridge
{
    private readonly Subject<string> _messages = new();
    public IObservable<string> Messages => _messages;

    private readonly object _webViewControl;

    public WebViewBridge(object webViewControl /* WebView 구체 타입 */)
    {
        _webViewControl = webViewControl;

        // TODO: 실제 구현체 이벤트 연결
        // 예시: webView.CoreWebView2.WebMessageReceived += (s,e) => _messages.OnNext(e.TryGetAsString());
    }

    public void PostJson(object payload)
    {
        var json = JsonSerializer.Serialize(payload);
        // TODO: JS로 메시지 push (구현체 API 호출)
        // 예: webView.CoreWebView2.PostWebMessageAsString(json);
    }
}
```

ViewModel에 결합:

```csharp
public class WebShellViewModel : ReactiveUI.ReactiveObject
{
    private readonly IWebBridge _bridge;
    public WebShellViewModel(IWebBridge bridge)
    {
        _bridge = bridge;
        _bridge.Messages.Subscribe(OnWebMessage);
    }

    private void OnWebMessage(string raw)
    {
        try
        {
            var msg = JsonSerializer.Deserialize<Dictionary<string, object>>(raw);
            if (msg is null) return;

            if ((string?)msg["type"] == "refresh-data")
            {
                // 예: 코어 VM에서 데이터 갱신 → 결과를 웹으로 push
                var payload = new { type = "chart-data", series = new[] { 1, 2, 3, 5, 8 } };
                _bridge.PostJson(payload);
            }
        }
        catch { /* 로깅 */ }
    }
}
```

> 실전에서는 **일관된 프로토콜(예: `{type, id, payload}`)**로 정의하고, 요청/응답/알림을 구분해 라우팅한다.

---

## 5. 반응형 레이아웃/입력/스케일링(모바일 대비)

모바일/임베디드 타겟을 고려하면 **크기/밀도/입력**을 재검토해야 한다.

### 5.1 화면 크기에 따른 레이아웃 전환

```xml
<!-- RootView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui">
  <UserControl.Styles>
    <Style>
      <Style.Triggers>
        <!-- 폭이 720 미만이면 모바일 레이아웃 -->
        <DataTrigger Binding="{Binding $parent[Window].Bounds.Width}" Value="720">
          <!-- DataTrigger가 직접 비교를 못하므로 아래처럼 변형: -->
        </DataTrigger>
      </Style.Triggers>
    </Style>
  </UserControl.Styles>

  <!-- 간단 대안: 두 컨테이너를 두고 Visible 스위치 -->
  <Grid>
    <local:DesktopLayout IsVisible="{Binding IsDesktop}"/>
    <local:MobileLayout  IsVisible="{Binding IsMobile}"/>
  </Grid>
</UserControl>
```

```csharp
// RootViewModel.cs
public class RootViewModel : ReactiveUI.ReactiveObject
{
    private double _width;
    public double Width
    {
        get => _width;
        set => this.RaiseAndSetIfChanged(ref _width, value);
    }

    public bool IsMobile => Width < 720;
    public bool IsDesktop => !IsMobile;
}
```

윈도우 크기 변경 시 Width 바인딩:

```csharp
// RootView.axaml.cs
this.AttachedToVisualTree += (_,__) =>
{
    var win = this.VisualRoot as Window;
    if (win is null) return;
    win.GetObservable(Visual.BoundsProperty)
       .Select(b => b.Width)
       .Subscribe(w => (DataContext as RootViewModel)!.Width = w);
};
```

### 5.2 입력(터치/제스처)

- 리스트 아이템의 **HitTarget**을 넓힌다(버튼 최소 44px).
- 스와이프/두 손가락 확대 등은 **제스처 라이브러리**나 **PointerPressed/Released/Pinch 계산**으로 구현.
- ScrollViewer 속성: 터치관성/오버스크롤을 검토.

### 5.3 DPI/리소스 스케일

- **벡터(Geometry/IconFont)** 사용 권장.
- 래스터 이미지는 `1x/2x/3x` 분기(테마 리소스 키를 이용해 런타임 선택).
- 폰트 크기/Spacing은 **상대값(ThemeResource)**로 묶어 스케일 조정.

---

## 6. 모바일(실험) 타겟팅: 빌드와 조건부 컴파일

> Avalonia.Mobile 흐름은 실험적이다. 빌드 파이프라인을 시도하고, UI/입력 시나리오를 작은 화면에서 검증하는 수준으로 접근한다.

### 6.1 프로젝트 파일(예시)

```xml
<!-- MyApp.Mobile.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFrameworks>net8.0-android;net8.0-ios</TargetFrameworks>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\MyApp.Core\MyApp.Core.csproj" />
  </ItemGroup>

  <PropertyGroup Condition="'$(TargetFramework)'=='net8.0-android'">
    <DefineConstants>$(DefineConstants);ANDROID</DefineConstants>
  </PropertyGroup>
  <PropertyGroup Condition="'$(TargetFramework)'=='net8.0-ios'">
    <DefineConstants>$(DefineConstants);IOS</DefineConstants>
  </PropertyGroup>
</Project>
```

### 6.2 조건부 컴파일

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

> 모바일에서는 네이티브 권한(카메라/파일)과 라이프사이클(백그라운드/포그라운드)을 고려해야 한다. 실험 단계에서는 **WebView 기반 화면**을 우선 띄워 UX를 검증하는 것이 현실적이다.

---

## 7. 내장 PWA: 오프라인 웹앱 번들 + 캐시

> “앱 안에 웹앱”을 **로컬로 포함**하고, **Service Worker**로 오프라인 캐시를 활성화하면 네트워크 없이도 WebView UI가 동작한다.

### 7.1 manifest.webmanifest

```json
{
  "name": "MyApp Embedded PWA",
  "short_name": "MyApp",
  "display": "standalone",
  "start_url": "./index.html",
  "icons": [
    { "src": "./icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "./icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ],
  "background_color": "#121212",
  "theme_color": "#121212"
}
```

### 7.2 Service Worker(sw.js)

```js
const CACHE_NAME = 'myapp-cache-v1';
const ASSETS = [
  './index.html',
  './manifest.webmanifest',
  './styles.css',
  './app.js',
  './icons/icon-192.png',
  './icons/icon-512.png'
];

self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE_NAME).then(c => c.addAll(ASSETS)));
});

self.addEventListener('activate', e => {
  e.waitUntil(
    caches.keys().then(keys =>
      Promise.all(keys.filter(k => k !== CACHE_NAME).map(k => caches.delete(k))))
  );
});

self.addEventListener('fetch', e => {
  e.respondWith(
    caches.match(e.request).then(r => r || fetch(e.request))
  );
});
```

### 7.3 index.html에 등록

```html
<link rel="manifest" href="manifest.webmanifest">
<script>
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('./sw.js');
  }
</script>
```

> 이 번들을 앱 리소스로 포함하고 **file:///…** 또는 **http://app.local/** 같은 커스텀 스킴으로 로드하면 인터넷 없이도 WebView UI가 동작한다.

---

## 8. 인증/보안 고려(하이브리드 기준)

- OAuth/OIDC는 **WebView**로 로그인 페이지를 띄우고, **리디렉션 URI**를 커스텀 스킴으로 돌려받아 **토큰 추출**.
- 토큰은 **보안 저장소**(Windows DPAPI, macOS Keychain, Android Keystore 등)에 저장. 데스크톱은 가능한 **OS 보안 저장소** 사용.
- JS↔.NET 브리지는 **화이트리스트 메시지 타입**만 허용하고, **서명된 요청/Nonce**로 재생 공격 방지.

---

## 9. 성능 팁

- WebView는 **한 화면에서만** 유지하고 내부 라우팅으로 페이지를 바꾼다(매번 생성/파괴 금지).
- 대용량 데이터는 **스트리밍/가상화**(리스트/테이블).
- 그래픽은 가능하면 **Skia 기반 Avalonia 네이티브**를 사용하고, 웹 파트는 **CSS 애니메이션 최소화**.

---

## 10. 빌드·배포·업데이트

- 데스크톱: `dotnet publish -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true`
- 웹 번들은 `web/` 폴더 채로 포함(또는 임베디드 리소스 → 최초 실행 시 사용자 캐시 폴더에 풀기).
- 자동 업데이트는 데스크톱 파트에서 처리(Squirrel, Zip-SelfUpdate 등). 웹 파트는 **sw.js 버전**을 바꿔 **오프라인 캐시 갱신**.

---

## 11. 통합 예제: 네이티브 + WebView + JS 브리지 + PWA

### 11.1 ViewModel: 차트 데이터 송수신

```csharp
public class HybridDashboardViewModel : ReactiveUI.ReactiveObject
{
    private readonly IWebBridge _bridge;
    private readonly DashboardViewModel _core;

    public HybridDashboardViewModel(IWebBridge bridge, DashboardViewModel core)
    {
        _bridge = bridge;
        _core = core;

        _bridge.Messages.Subscribe(OnWebMessage);
        StartFeeder();
    }

    private void OnWebMessage(string raw)
    {
        var m = JsonSerializer.Deserialize<Dictionary<string, object>>(raw);
        if (m?["type"] as string == "ack")
        {
            // 웹에서 ack 수신
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

### 11.2 JS 측: 차트 갱신

```html
<script>
  const points = [];
  window.addEventListener('message', ev => {
    const msg = ev.data;
    if (msg.type === 'chart-data') {
      points.push(msg.value);
      renderChart(points); // 캔버스/차트 라이브러리 갱신
      sendToHost({ type: 'ack' });
    }
  });
</script>
```

---

## 12. 리스크와 완화책

| 리스크 | 완화 전략 |
|---|---|
| 모바일 렌더링/입력 이슈 | 작은 단위부터 시도(MVP 화면 1~2개). 제스처/키보드 정책 정리 |
| WebView 엔진 차이 | WebKit/Chromium 간 CSS/JS 차이 최소화. 공통 기능만 사용 |
| 보안 | 토큰/비밀키는 OS 보안 저장소. 메시징 화이트리스트/서명/Nonce |
| 오프라인 캐시 동기화 | sw.js 버전 증가 → 강제 캐시 무효화. 앱 업데이트와 연동 |
| 유지보수 복잡도 | 코어/호스트 분리, 명확한 인터페이스(IWebBridge, IService) 유지 |

---

## 13. 체크리스트

- 공통 코드 80% 이상을 **MyApp.Core**에 유지
- 데스크톱/모바일 호스트는 **최소 기능**만(창/권한/파일/브라우저 호스팅)
- WebView 브리지는 **추상화 인터페이스**로 감싸기
- 반응형/입력/스케일을 **초기에** 확정(폰트/버튼 최소 크기 등)
- 오프라인 웹앱(PWA) 캐시 전략 **문서화**

---

## 결론

- **지금 당장 실전 배포** 기준으로는 데스크톱이 가장 안정적이다.
- **모바일은 실험적**이므로 작은 범위로 검증하며 **하이브리드(WebView)** 전략을 병행한다.
- PWA는 **내장 WebView + 로컬 번들 + Service Worker**로 충분히 “앱 같은 웹” UX를 제공할 수 있다.
- 장기적으로는 Avalonia의 웹/모바일 지원이 성숙 시, **코어 재사용**을 극대화해 자연스레 이행한다.
