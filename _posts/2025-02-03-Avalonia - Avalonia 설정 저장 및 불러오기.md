---
layout: post
title: Avalonia - Avalonia 설정 저장 및 불러오기
date: 2025-02-03 19:20:23 +0900
category: Avalonia
---
# Avalonia 설정 저장 및 불러오기 (JSON, SQLite, 암호화, 마이그레이션)

Avalonia 애플리케이션에서 사용자 설정(테마, 언어, 자동 로그인 여부 등)을 저장하고 불러오는 기능은 필수적입니다. 이 글에서는 JSON 파일과 SQLite 데이터베이스 두 가지 방식을 모두 다루고, 민감 정보(토큰 등)는 암호화하여 저장하며, 스키마 버전 관리와 마이그레이션까지 고려한 실전 구조를 설명합니다. 초중급 개발자를 기준으로, 직접 코드를 따라 하며 이해할 수 있도록 구성했습니다.

---

## 목표

- 설정 모델(`AppSettings`)에 버전 정보를 포함하여 확장성 확보
- JSON 저장소(원자적 쓰기 + 백업)와 SQLite 저장소(트랜잭션 + 스키마 관리)를 모두 구현
- AES-256-GCM으로 민감 정보 암호화
- 앱 시작 시 설정을 불러와 전역 상태(`AppState`)에 반영하고, UI 변경 시 즉시 저장
- 예외(파일 손상, 동시 접근 등)에 대한 대비 및 단위 테스트

---

## 디렉터리 구조 (확장판)

```
MyApp/
├── App.axaml / App.axaml.cs
├── Config/
│   ├── AppSettings.cs              // 설정 모델 (버전, 유효성 검사)
│   ├── ISettingsService.cs         // 저장소 추상화 인터페이스
│   ├── JsonSettingsService.cs      // JSON 파일 저장 구현
│   ├── SqliteSettingsService.cs    // SQLite 저장 구현
│   ├── ICryptoService.cs           // 암호화 인터페이스
│   └── AesCryptoService.cs         // AES-256-GCM 구현
├── Services/
│   ├── AppState.cs                 // 전역 상태 (반응형)
│   ├── SettingsFacade.cs           // ViewModel에서 쓰기 편한 파사드
│   └── Paths.cs                    // OS별 저장 경로 결정
├── ViewModels/
│   ├── SettingsViewModel.cs
│   └── LoginViewModel.cs
├── Views/
│   └── SettingsView.axaml
└── Tests/
    └── SettingsTests.cs
```

**설명**:  
- `Config` 폴더에는 설정 관련 모델, 인터페이스, 구현체를 모았습니다.  
- `Services`에는 전역 상태와 파사드를 두어 ViewModel이 저장소의 종류를 알지 못하게 했습니다.  
- `Paths.cs`는 OS별 권장 디렉터리를 반환하는 유틸리티입니다.

---

## 저장 위치 설계 (크로스 플랫폼)

사용자 설정은 OS가 권장하는 위치에 저장하는 것이 일반적입니다. 여기에 포터블 모드(실행 파일 옆에 저장)도 지원할 수 있도록 합니다.

```csharp
// Config/Paths.cs
public static class Paths
{
    public static string GetSettingsDirectory(string? appName = "MyApp")
    {
        // 1) 포터블 모드: 환경 변수로 지정
        var portable = Environment.GetEnvironmentVariable("MYAPP_PORTABLE");
        if (!string.IsNullOrEmpty(portable))
            return Path.Combine(AppContext.BaseDirectory, "data");

        // 2) OS 권장 위치
        if (OperatingSystem.IsWindows())
        {
            var appData = Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData);
            return Path.Combine(appData, appName!);
        }
        if (OperatingSystem.IsMacOS())
        {
            var home = Environment.GetFolderPath(Environment.SpecialFolder.Personal);
            return Path.Combine(home, "Library", "Application Support", appName!);
        }
        // Linux / Unix
        var config = Environment.GetEnvironmentVariable("XDG_CONFIG_HOME")
                     ?? Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.Personal), ".config");
        return Path.Combine(config, appName!);
    }

    public static string GetJsonPath() => Path.Combine(GetSettingsDirectory(), "settings.json");
    public static string GetJsonBackupPath() => Path.Combine(GetSettingsDirectory(), "settings.bak.json");
    public static string GetSqlitePath() => Path.Combine(GetSettingsDirectory(), "settings.sqlite");
}
```

---

## 설정 모델 (AppSettings)

설정 모델에는 스키마 버전, 일반 필드, 암호화 저장용 필드를 포함합니다. `DataAnnotations`를 사용해 유효성 검사를 할 수 있습니다.

```csharp
// Config/AppSettings.cs
using System.ComponentModel.DataAnnotations;

public class AppSettings
{
    public int SchemaVersion { get; set; } = 1;

    [Required, RegularExpression(@"^(ko|en|ja)$")]
    public string Language { get; set; } = "ko";

    [Required, RegularExpression(@"^(Light|Dark)$")]
    public string Theme { get; set; } = "Light";

    public bool AutoLogin { get; set; } = false;

    // 실제 저장되는 암호화된 토큰
    public string? EncryptedAuthToken { get; set; }

    // 사용자 편의를 위한 평문 토큰 (암호화/복호화 후 사용)
    [JsonIgnore]
    public string? AuthToken { get; set; }

    public bool UseHardwareAcceleration { get; set; } = true;
    public int WindowWidth { get; set; } = 1280;
    public int WindowHeight { get; set; } = 800;
}
```

**참고**: `AuthToken`은 `JsonIgnore` 속성을 붙여 직렬화하지 않도록 하고, 대신 `EncryptedAuthToken`에 암호화된 값을 저장합니다.

---

## 암호화 서비스 (AES-256-GCM)

민감 정보는 AES-GCM 알고리즘으로 암호화합니다. 키는 외부에서 안전하게 관리해야 하지만, 여기서는 예시로 고정 키를 사용합니다.

```csharp
// Config/ICryptoService.cs
public interface ICryptoService
{
    string Encrypt(string plain);
    string Decrypt(string cipher);
}
```

```csharp
// Config/AesCryptoService.cs
using System.Security.Cryptography;
using System.Text;

public sealed class AesCryptoService : ICryptoService
{
    private readonly byte[] _key;

    public AesCryptoService(byte[] key) => _key = key;

    public string Encrypt(string plain)
    {
        if (string.IsNullOrEmpty(plain)) return plain;

        using var aes = new AesGcm(_key);
        var nonce = RandomNumberGenerator.GetBytes(12);
        var plainBytes = Encoding.UTF8.GetBytes(plain);
        var cipher = new byte[plainBytes.Length];
        var tag = new byte[16];

        aes.Encrypt(nonce, plainBytes, cipher, tag);
        var payload = Convert.ToBase64String(nonce.Concat(cipher).Concat(tag).ToArray());
        return payload;
    }

    public string Decrypt(string cipherText)
    {
        if (string.IsNullOrEmpty(cipherText)) return cipherText;

        var raw = Convert.FromBase64String(cipherText);
        var nonce = raw[..12];
        var tag = raw[^16..];
        var cipher = raw[12..^16];

        using var aes = new AesGcm(_key);
        var plain = new byte[cipher.Length];
        aes.Decrypt(nonce, cipher, tag, plain);
        return Encoding.UTF8.GetString(plain);
    }
}
```

---

## 저장소 인터페이스

`ISettingsService`는 설정을 로드하고 저장하는 표준 인터페이스입니다. 구현체는 JSON 또는 SQLite 방식으로 제공합니다.

```csharp
// Config/ISettingsService.cs
public interface ISettingsService
{
    Task<AppSettings> LoadAsync(CancellationToken ct = default);
    Task SaveAsync(AppSettings settings, CancellationToken ct = default);
}
```

---

## JSON 저장소 구현 (원자적 쓰기 + 백업)

JSON 파일을 안전하게 다루려면 **임시 파일 → 덮어쓰기** 방식으로 원자적 쓰기를 보장하고, 저장 전에 백업 파일을 만듭니다.

```csharp
// Config/JsonSettingsService.cs
using System.Text.Json;

public sealed class JsonSettingsService : ISettingsService
{
    private readonly ICryptoService _crypto;
    private readonly JsonSerializerOptions _opt = new()
    {
        WriteIndented = true,
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase
    };

    public JsonSettingsService(ICryptoService crypto) => _crypto = crypto;

    public async Task<AppSettings> LoadAsync(CancellationToken ct = default)
    {
        var path = Paths.GetJsonPath();
        var backup = Paths.GetJsonBackupPath();

        Directory.CreateDirectory(Path.GetDirectoryName(path)!);

        if (!File.Exists(path))
        {
            if (File.Exists(backup))
                File.Copy(backup, path, overwrite: true);
            else
                return new AppSettings();
        }

        try
        {
            await using var fs = File.Open(path, FileMode.Open, FileAccess.Read, FileShare.Read);
            var loaded = await JsonSerializer.DeserializeAsync<AppSettings>(fs, _opt, ct)
                         ?? new AppSettings();

            if (!string.IsNullOrWhiteSpace(loaded.EncryptedAuthToken))
                loaded.AuthToken = _crypto.Decrypt(loaded.EncryptedAuthToken);

            MigrateIfNeeded(loaded);
            return loaded;
        }
        catch
        {
            // 파일 손상 시 백업 복구 시도
            if (File.Exists(backup))
            {
                File.Copy(backup, path, true);
                return await LoadAsync(ct);
            }
            return new AppSettings();
        }
    }

    public async Task SaveAsync(AppSettings s, CancellationToken ct = default)
    {
        var path = Paths.GetJsonPath();
        var backup = Paths.GetJsonBackupPath();

        Directory.CreateDirectory(Path.GetDirectoryName(path)!);

        s.EncryptedAuthToken = string.IsNullOrWhiteSpace(s.AuthToken) ? null : _crypto.Encrypt(s.AuthToken);

        var temp = Path.GetTempFileName();
        try
        {
            await using (var fs = File.Open(temp, FileMode.Create, FileAccess.Write, FileShare.None))
            {
                await JsonSerializer.SerializeAsync(fs, s, _opt, ct);
            }

            if (File.Exists(path))
                File.Copy(path, backup, overwrite: true);

            if (File.Exists(path)) File.Delete(path);
            File.Move(temp, path);
        }
        catch
        {
            try { if (File.Exists(temp)) File.Delete(temp); } catch { /* ignore */ }
            throw;
        }
    }

    private static void MigrateIfNeeded(AppSettings s)
    {
        // 예: SchemaVersion 1 → 2로 업그레이드
        if (s.SchemaVersion < 2)
        {
            // s.NewField = ...;
            s.SchemaVersion = 2;
        }
    }
}
```

**핵심 포인트**  
- 임시 파일에 저장 후 `File.Move`로 원자적 교체  
- 저장 전 백업 생성, 로드 시 백업 복구 가능  
- 암호화/복호화는 저장/로드 시점에 적용  

---

## SQLite 저장소 구현 (스키마 + 마이그레이션)

SQLite를 사용하려면 `Microsoft.Data.Sqlite` 패키지를 추가합니다.

```bash
dotnet add package Microsoft.Data.Sqlite
```

스키마는 단일 레코드 테이블로 관리하며, 트랜잭션으로 일관성을 보장합니다.

```csharp
// Config/SqliteSettingsService.cs
using Microsoft.Data.Sqlite;

public sealed class SqliteSettingsService : ISettingsService
{
    private readonly ICryptoService _crypto;

    public SqliteSettingsService(ICryptoService crypto) => _crypto = crypto;

    public async Task<AppSettings> LoadAsync(CancellationToken ct = default)
    {
        var path = Paths.GetSqlitePath();
        Directory.CreateDirectory(Path.GetDirectoryName(path)!);

        using var conn = new SqliteConnection($"Data Source={path}");
        await conn.OpenAsync(ct);
        await EnsureSchemaAsync(conn, ct);

        var cmd = conn.CreateCommand();
        cmd.CommandText = """
            SELECT SchemaVersion, Language, Theme, EncryptedAuthToken, AutoLogin,
                   UseHardwareAcceleration, WindowWidth, WindowHeight
            FROM AppSettings WHERE Id = 1
        """;

        using var reader = await cmd.ExecuteReaderAsync(ct);
        if (!await reader.ReadAsync(ct))
            return new AppSettings();

        var s = new AppSettings
        {
            SchemaVersion = reader.GetInt32(0),
            Language = reader.GetString(1),
            Theme = reader.GetString(2),
            EncryptedAuthToken = reader.IsDBNull(3) ? null : reader.GetString(3),
            AutoLogin = reader.GetBoolean(4),
            UseHardwareAcceleration = reader.GetBoolean(5),
            WindowWidth = reader.GetInt32(6),
            WindowHeight = reader.GetInt32(7)
        };

        if (!string.IsNullOrWhiteSpace(s.EncryptedAuthToken))
            s.AuthToken = _crypto.Decrypt(s.EncryptedAuthToken);

        MigrateIfNeeded(conn, s, ct);
        return s;
    }

    public async Task SaveAsync(AppSettings s, CancellationToken ct = default)
    {
        var path = Paths.GetSqlitePath();
        Directory.CreateDirectory(Path.GetDirectoryName(path)!);

        using var conn = new SqliteConnection($"Data Source={path}");
        await conn.OpenAsync(ct);
        await EnsureSchemaAsync(conn, ct);

        s.EncryptedAuthToken = string.IsNullOrWhiteSpace(s.AuthToken) ? null : _crypto.Encrypt(s.AuthToken);

        using var tx = await conn.BeginTransactionAsync(ct);
        var cmd = conn.CreateCommand();
        cmd.Transaction = tx;

        cmd.CommandText = """
            INSERT INTO AppSettings(Id, SchemaVersion, Language, Theme, EncryptedAuthToken, AutoLogin,
                                    UseHardwareAcceleration, WindowWidth, WindowHeight)
            VALUES(1, $sv, $lang, $theme, $token, $auto, $hwa, $w, $h)
            ON CONFLICT(Id) DO UPDATE SET
                SchemaVersion=$sv, Language=$lang, Theme=$theme, EncryptedAuthToken=$token,
                AutoLogin=$auto, UseHardwareAcceleration=$hwa, WindowWidth=$w, WindowHeight=$h
        """;

        cmd.Parameters.AddWithValue("$sv", s.SchemaVersion);
        cmd.Parameters.AddWithValue("$lang", s.Language);
        cmd.Parameters.AddWithValue("$theme", s.Theme);
        cmd.Parameters.AddWithValue("$token", (object?)s.EncryptedAuthToken ?? DBNull.Value);
        cmd.Parameters.AddWithValue("$auto", s.AutoLogin);
        cmd.Parameters.AddWithValue("$hwa", s.UseHardwareAcceleration);
        cmd.Parameters.AddWithValue("$w", s.WindowWidth);
        cmd.Parameters.AddWithValue("$h", s.WindowHeight);

        await cmd.ExecuteNonQueryAsync(ct);
        await tx.CommitAsync(ct);
    }

    private static async Task EnsureSchemaAsync(SqliteConnection conn, CancellationToken ct)
    {
        var sql = """
            CREATE TABLE IF NOT EXISTS AppSettings(
                Id INTEGER PRIMARY KEY CHECK (Id = 1),
                SchemaVersion INTEGER NOT NULL,
                Language TEXT NOT NULL,
                Theme TEXT NOT NULL,
                EncryptedAuthToken TEXT NULL,
                AutoLogin INTEGER NOT NULL,
                UseHardwareAcceleration INTEGER NOT NULL,
                WindowWidth INTEGER NOT NULL,
                WindowHeight INTEGER NOT NULL
            )
        """;
        var cmd = conn.CreateCommand();
        cmd.CommandText = sql;
        await cmd.ExecuteNonQueryAsync(ct);

        var existsCmd = conn.CreateCommand();
        existsCmd.CommandText = "SELECT COUNT(*) FROM AppSettings WHERE Id = 1";
        var count = (long)(await existsCmd.ExecuteScalarAsync(ct) ?? 0L);
        if (count == 0)
        {
            var seedCmd = conn.CreateCommand();
            seedCmd.CommandText = """
                INSERT INTO AppSettings(Id, SchemaVersion, Language, Theme, EncryptedAuthToken,
                                        AutoLogin, UseHardwareAcceleration, WindowWidth, WindowHeight)
                VALUES(1, 1, 'ko', 'Light', NULL, 0, 1, 1280, 800)
            """;
            await seedCmd.ExecuteNonQueryAsync(ct);
        }
    }

    private static void MigrateIfNeeded(SqliteConnection conn, AppSettings s, CancellationToken ct)
    {
        // 예: SchemaVersion 1 → 2 마이그레이션
        if (s.SchemaVersion < 2)
        {
            // ALTER TABLE 등 실행
            s.SchemaVersion = 2;
            // 저장은 SaveAsync에서 다시 호출되므로 여기서는 생략
        }
    }
}
```

**핵심 포인트**  
- `ON CONFLICT` 구문으로 UPSERT 구현  
- 트랜잭션으로 부분 저장 방지  
- 스키마 생성과 시드 데이터를 `EnsureSchemaAsync`에서 처리  

---

## 전역 상태와 파사드

`AppState`는 설정의 값을 반응형으로 보관합니다. `SettingsFacade`는 저장소와 `AppState`를 연결하며, ViewModel이 저장소의 종류를 몰라도 되게 합니다.

```csharp
// Services/AppState.cs
using ReactiveUI;

public sealed class AppState : ReactiveObject
{
    private string _language = "ko";
    private string _theme = "Light";
    private string? _authToken;

    public string Language
    {
        get => _language;
        set => this.RaiseAndSetIfChanged(ref _language, value);
    }

    public string Theme
    {
        get => _theme;
        set => this.RaiseAndSetIfChanged(ref _theme, value);
    }

    public string? AuthToken
    {
        get => _authToken;
        set => this.RaiseAndSetIfChanged(ref _authToken, value);
    }
}
```

```csharp
// Services/SettingsFacade.cs
using System.ComponentModel.DataAnnotations;

public sealed class SettingsFacade
{
    private readonly ISettingsService _repo;
    private readonly AppState _state;

    public SettingsFacade(ISettingsService repo, AppState state)
    {
        _repo = repo;
        _state = state;
    }

    public async Task LoadIntoStateAsync(CancellationToken ct = default)
    {
        var s = await _repo.LoadAsync(ct);
        ApplyToState(s);
    }

    public async Task SaveFromStateAsync(CancellationToken ct = default)
    {
        var s = FromState();
        Validate(s);
        await _repo.SaveAsync(s, ct);
    }

    private void ApplyToState(AppSettings s)
    {
        _state.Language = s.Language;
        _state.Theme = s.Theme;
        _state.AuthToken = s.AuthToken;
    }

    private AppSettings FromState() => new()
    {
        Language = _state.Language,
        Theme = _state.Theme,
        AuthToken = _state.AuthToken,
        // 기타 필드는 필요 시 채움
    };

    private static void Validate(AppSettings s)
    {
        var results = new List<ValidationResult>();
        var ctx = new ValidationContext(s);
        if (!Validator.TryValidateObject(s, ctx, results, true))
            throw new ValidationException(string.Join("; ", results.Select(r => r.ErrorMessage)));
    }
}
```

---

## DI 구성 및 앱 초기화

`App.axaml.cs`에서 DI 컨테이너를 구성하고, 앱 시작 시 설정을 로드합니다.

```csharp
// App.axaml.cs (일부)
using Microsoft.Extensions.DependencyInjection;

public partial class App : Application
{
    public static ServiceProvider Services { get; private set; } = default!;

    public override void OnFrameworkInitializationCompleted()
    {
        var services = new ServiceCollection();
        ConfigureServices(services);
        Services = services.BuildServiceProvider();

        var facade = Services.GetRequiredService<SettingsFacade>();
        // 설정 로드 (비동기)
        _ = facade.LoadIntoStateAsync();

        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            var mainVm = Services.GetRequiredService<MainViewModel>();
            desktop.MainWindow = new MainWindow { DataContext = mainVm };
        }

        base.OnFrameworkInitializationCompleted();
    }

    private void ConfigureServices(IServiceCollection services)
    {
        // 예제 키 (실제로는 안전한 저장소에서 로드)
        var key = Enumerable.Repeat((byte)0x11, 32).ToArray();
        services.AddSingleton<ICryptoService>(_ => new AesCryptoService(key));

        // 저장소 선택: 환경 변수로 JSON 또는 SQLite 결정
        var engine = Environment.GetEnvironmentVariable("MYAPP_SETTINGS_ENGINE") ?? "json";
        if (engine.Equals("sqlite", StringComparison.OrdinalIgnoreCase))
            services.AddSingleton<ISettingsService, SqliteSettingsService>();
        else
            services.AddSingleton<ISettingsService, JsonSettingsService>();

        services.AddSingleton<AppState>();
        services.AddSingleton<SettingsFacade>();
        services.AddTransient<SettingsViewModel>();
        // 기타 ViewModel 등록
    }
}
```

---

## 설정 UI (SettingsViewModel & View)

설정 화면은 `AppState`를 직접 바인딩하고, 저장/불러오기는 `SettingsFacade`를 통해 처리합니다.

```csharp
// ViewModels/SettingsViewModel.cs
using ReactiveUI;
using System.Reactive;
using System.Threading.Tasks;

public sealed class SettingsViewModel : ReactiveObject
{
    private readonly SettingsFacade _facade;
    private readonly AppState _state;

    public SettingsViewModel(SettingsFacade facade, AppState state)
    {
        _facade = facade;
        _state = state;

        SaveCommand = ReactiveCommand.CreateFromTask(SaveAsync);
        ReloadCommand = ReactiveCommand.CreateFromTask(ReloadAsync);
    }

    public string Language
    {
        get => _state.Language;
        set => _state.Language = value;
    }

    public string Theme
    {
        get => _state.Theme;
        set => _state.Theme = value;
    }

    public bool AutoLogin { get; set; } // 실제 저장 시 반영

    public ReactiveCommand<Unit, Unit> SaveCommand { get; }
    public ReactiveCommand<Unit, Unit> ReloadCommand { get; }

    private async Task SaveAsync()
    {
        // AutoLogin을 포함하려면 FromState() 확장 필요
        await _facade.SaveFromStateAsync();
    }

    private async Task ReloadAsync()
    {
        await _facade.LoadIntoStateAsync();
        this.RaisePropertyChanged(nameof(Language));
        this.RaisePropertyChanged(nameof(Theme));
    }
}
```

```xml
<!-- Views/SettingsView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyApp.Views.SettingsView">
    <StackPanel Margin="20" Spacing="10">
        <TextBlock Text="설정" FontSize="20"/>

        <StackPanel Orientation="Horizontal" Spacing="8">
            <TextBlock Text="언어" Width="80"/>
            <ComboBox SelectedItem="{Binding Language}">
                <ComboBoxItem>ko</ComboBoxItem>
                <ComboBoxItem>en</ComboBoxItem>
                <ComboBoxItem>ja</ComboBoxItem>
            </ComboBox>
        </StackPanel>

        <StackPanel Orientation="Horizontal" Spacing="8">
            <TextBlock Text="테마" Width="80"/>
            <ComboBox SelectedItem="{Binding Theme}">
                <ComboBoxItem>Light</ComboBoxItem>
                <ComboBoxItem>Dark</ComboBoxItem>
            </ComboBox>
        </StackPanel>

        <CheckBox Content="자동 로그인" IsChecked="{Binding AutoLogin}"/>

        <StackPanel Orientation="Horizontal" Spacing="8">
            <Button Content="저장" Command="{Binding SaveCommand}"/>
            <Button Content="다시 불러오기" Command="{Binding ReloadCommand}"/>
        </StackPanel>
    </StackPanel>
</UserControl>
```

---

## 로그인 후 토큰 저장

로그인 성공 시 `AppState.AuthToken`에 토큰을 설정하고, `SettingsFacade.SaveFromStateAsync()`를 호출하여 암호화 저장합니다.

```csharp
// ViewModels/LoginViewModel.cs (일부)
private async Task<bool> LoginAsync()
{
    // 실제 인증 로직
    var ok = await _authService.LoginAsync(Username, Password);
    if (!ok) return false;

    _state.AuthToken = _authService.GetToken();
    var s = _facade.FromState();
    s.AutoLogin = true; // 필요 시
    await _facade.SaveFromStateAsync();
    return true;
}
```

---

## 단위 테스트

테스트에서는 임시 경로(포터블 모드)를 사용하여 실제 파일 시스템에 영향을 주지 않도록 합니다.

```csharp
// Tests/SettingsTests.cs
using Xunit;
using FluentAssertions;

public sealed class SettingsTests
{
    [Fact]
    public async Task Json_Roundtrip_Works()
    {
        Environment.SetEnvironmentVariable("MYAPP_PORTABLE", "1");
        var key = Enumerable.Repeat((byte)0x22, 32).ToArray();
        var crypto = new AesCryptoService(key);
        var repo = new JsonSettingsService(crypto);
        var state = new AppState();
        var facade = new SettingsFacade(repo, state);

        state.Language = "en";
        state.Theme = "Dark";
        state.AuthToken = "secret";

        await facade.SaveFromStateAsync();

        state.Language = "ko";
        state.Theme = "Light";
        state.AuthToken = null;

        await facade.LoadIntoStateAsync();

        state.Language.Should().Be("en");
        state.Theme.Should().Be("Dark");
        state.AuthToken.Should().Be("secret");
    }

    [Fact]
    public async Task Sqlite_Roundtrip_Works()
    {
        Environment.SetEnvironmentVariable("MYAPP_PORTABLE", "1");
        var key = Enumerable.Repeat((byte)0x33, 32).ToArray();
        var crypto = new AesCryptoService(key);
        var repo = new SqliteSettingsService(crypto);

        var s = new AppSettings { Language = "ja", Theme = "Dark", AuthToken = "tok" };
        await repo.SaveAsync(s);

        var loaded = await repo.LoadAsync();
        loaded.Language.Should().Be("ja");
        loaded.Theme.Should().Be("Dark");
        loaded.AuthToken.Should().Be("tok");
    }
}
```

---

## JSON vs SQLite 선택 기준

| 기준 | JSON | SQLite |
|------|------|--------|
| 설정 규모 | 작고 단순할 때 적합 | 복잡한 구조나 다수 설정에 적합 |
| 원자성 | 임시 파일+백업으로 구현 가능 | 트랜잭션으로 자연스럽게 보장 |
| 마이그레이션 | 코드 수준에서 직접 처리 | DDL + 코드 병행 |
| 외부 편집 | 메모장으로 쉽게 확인 가능 | SQL 도구 필요 |
| 성능 | 매우 빠름 | 충분히 빠름 |
| 권장 사용 | 간단한 앱, 포터블 | 여러 설정 테이블, 로그 기록 등 |

---

## 체크리스트 (운영 관점)

- [ ] 저장 경로에 대한 읽기/쓰기 권한 확인
- [ ] JSON 원자적 쓰기 + 백업 복구 테스트
- [ ] SQLite 트랜잭션과 `WAL` 모드 검토
- [ ] 민감 정보 암호화 여부 확인 (`AuthToken` 등)
- [ ] 스키마 버전 관리 및 마이그레이션 코드 유지
- [ ] 예외 발생 시 로깅 (Serilog 등)
- [ ] 단위 테스트 CI 연동
- [ ] 포터블 모드 환경 변수 지원
- [ ] 테마, 언어 변경 시 앱 전반에 즉시 반영되도록 연동

---

## 결론

이 글에서는 Avalonia 앱에서 설정을 안전하게 저장하고 불러오는 전체적인 구조를 다루었습니다. 인터페이스 기반의 저장소 추상화 덕분에 JSON과 SQLite를 쉽게 교체할 수 있고, 암호화와 마이그레이션까지 포함하여 실전에 바로 적용할 수 있는 코드를 제공했습니다. `AppState`와 `SettingsFacade`를 활용하면 ViewModel에서 저장소의 구현을 몰라도 되며, 단위 테스트를 통해 안정성을 확보할 수 있습니다. 이 구조를 기반으로 자신의 앱에 맞게 확장해 보시기 바랍니다.