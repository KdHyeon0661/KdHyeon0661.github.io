---
layout: post
title: Avalonia - MVVM 로그인 및 인증 구조 설계
date: 2025-01-22 19:20:23 +0900
category: Avalonia
---
# ⚙️ Avalonia 설정 저장 및 불러오기 (JSON, SQLite 방식)

---

## 🧱 구성 개요

```
MyApp/
├── Config/
│   ├── AppSettings.cs
│   ├── ISettingsService.cs
│   └── JsonSettingsService.cs
├── Services/
│   └── SQLiteSettingsService.cs (선택)
└── settings.json
```

---

## 1️⃣ 설정 모델 정의

### 📄 Config/AppSettings.cs

```csharp
public class AppSettings
{
    public string Language { get; set; } = "ko";
    public string Theme { get; set; } = "Light"; // or "Dark"
    public string? AuthToken { get; set; }
    public bool AutoLogin { get; set; } = false;
}
```

---

## 2️⃣ 인터페이스 정의

### 📄 Config/ISettingsService.cs

```csharp
public interface ISettingsService
{
    Task<AppSettings> LoadAsync();
    Task SaveAsync(AppSettings settings);
}
```

---

## 3️⃣ JSON 저장 구현

### 📄 Config/JsonSettingsService.cs

```csharp
public class JsonSettingsService : ISettingsService
{
    private readonly string _path;

    public JsonSettingsService()
    {
        var folder = Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData);
        _path = Path.Combine(folder, "MyApp", "settings.json");
    }

    public async Task<AppSettings> LoadAsync()
    {
        if (!File.Exists(_path))
            return new AppSettings();

        var json = await File.ReadAllTextAsync(_path);
        return JsonSerializer.Deserialize<AppSettings>(json) ?? new AppSettings();
    }

    public async Task SaveAsync(AppSettings settings)
    {
        var dir = Path.GetDirectoryName(_path)!;
        Directory.CreateDirectory(dir);

        var json = JsonSerializer.Serialize(settings, new JsonSerializerOptions { WriteIndented = true });
        await File.WriteAllTextAsync(_path, json);
    }
}
```

---

## 4️⃣ DI 등록 (App.xaml.cs)

```csharp
private void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<ISettingsService, JsonSettingsService>();
    services.AddSingleton<AppState>();
}
```

---

## 5️⃣ 앱 시작 시 설정 불러오기

```csharp
public override async void OnFrameworkInitializationCompleted()
{
    var settingsService = Services.GetRequiredService<ISettingsService>();
    var settings = await settingsService.LoadAsync();

    var appState = Services.GetRequiredService<AppState>();
    appState.Language = settings.Language;
    appState.Theme = settings.Theme;
    appState.AuthToken = settings.AuthToken;

    if (settings.AutoLogin && !string.IsNullOrWhiteSpace(settings.AuthToken))
        ShowMainWindow(); // 자동 로그인
    else
        ShowLoginWindow();

    base.OnFrameworkInitializationCompleted();
}
```

---

## 6️⃣ 설정 변경 시 저장

예: 로그인 후 토큰 저장

```csharp
public async Task SaveTokenAsync(string token)
{
    var settingsService = Services.GetRequiredService<ISettingsService>();
    var settings = await settingsService.LoadAsync();
    settings.AuthToken = token;
    settings.AutoLogin = true;
    await settingsService.SaveAsync(settings);
}
```

---

## 7️⃣ (선택) SQLite 저장 방식

### 📄 SQLiteSettingsService.cs

```csharp
public class SQLiteSettingsService : ISettingsService
{
    private const string DbPath = "settings.db";

    public async Task<AppSettings> LoadAsync()
    {
        using var db = new LiteDatabase(DbPath);
        return db.GetCollection<AppSettings>().FindById(1) ?? new AppSettings();
    }

    public async Task SaveAsync(AppSettings settings)
    {
        using var db = new LiteDatabase(DbPath);
        var col = db.GetCollection<AppSettings>();
        settings.Id = 1;
        col.Upsert(settings);
    }
}
```

> 이 예시는 `LiteDB` 기반이며 `System.Data.SQLite` 또는 `Entity Framework Core` 기반도 사용 가능합니다.

---

## ✅ 저장 방식 선택 가이드

| 기준 | JSON | SQLite |
|------|------|--------|
| 단순 키/값 | 👍 적합 | 가능 |
| 구조적 데이터 | 😐 한계 | 👍 적합 |
| 파일 기반 | O | O |
| 읽기 속도 | 빠름 | 빠름 |
| 복잡한 설정 | 비추천 | 추천 |
| 동기화/검색 | 불가능 | 가능 |

---

## 🧪 테스트 및 확장 포인트

- 설정 저장 위치를 사용자 지정 가능하게 (`settingsPath`)
- 저장 실패 예외 처리 및 백업 기능 추가
- 변경 감지 이벤트 발행 (`OnSettingsChanged`)
- 버전 관리 및 마이그레이션

---

## 📘 결론

| 항목 | 구현 방법 |
|------|-----------|
| 설정 저장 | JSON 또는 SQLite로 구현 |
| DI 구조화 | `ISettingsService` 인터페이스로 추상화 |
| 설정 위치 | `%APPDATA%/MyApp/settings.json` |
| 상태 반영 | `AppState`에 로딩한 설정 적용 |
| UI 제어 | 테마, 언어, 자동 로그인 등에 연결 가능 |
