---
layout: post
title: Avalonia - Avalonia 설정 저장 및 불러오기
date: 2025-02-03 19:20:23 +0900
category: Avalonia
---
# Avalonia 설정 저장 및 불러오기 (JSON, SQLite, 암호화, 마이그레이션)

## 0. 무엇을 만들 것인가

- 설정 모델 `AppSettings`(버전 포함) 정의
- **JSON 저장**(원자적 쓰기 + 백업)과 **SQLite 저장**(스키마/마이그레이션) 2가지 경로 제공
- 민감정보(토큰 등) **AES-256** 대칭 암호화
- 앱 시작 시 설정 로드 → `AppState`에 적용(테마/언어/로그인)
- UI에서 수정 → 즉시 저장(자동/수동 선택)
- 예외/경합/파일락/손상 대응
- 단위 테스트(임시 경로, 인메모리 DB)로 회귀 방지

---

## 1. 디렉터리 구조(확장판)

```
MyApp/
├── App.axaml / App.axaml.cs
├── Config/
│   ├── AppSettings.cs              // 모델(+ 버전/마이그레이션)
│   ├── ISettingsService.cs         // 저장소 추상화
│   ├── JsonSettingsService.cs      // JSON 저장소 구현
│   ├── SqliteSettingsService.cs    // SQLite 저장소 구현 (Microsoft.Data.Sqlite)
│   ├── ICryptoService.cs           // 민감정보 암호화 인터페이스
│   └── AesCryptoService.cs         // AES-256 GCM 등 구현
├── Services/
│   ├── AppState.cs                 // 전역 상태(테마/언어/토큰 등)
│   └── SettingsFacade.cs           // ViewModel이 쓰기 편한 파사드(자동 저장/검증/이벤트)
├── ViewModels/
│   ├── SettingsViewModel.cs        // UI 바인딩용
│   └── LoginViewModel.cs           // 예: 로그인 후 토큰 저장
├── Views/
│   └── SettingsView.axaml          // UI
├── Test/
│   └── SettingsTests.cs            // 단위 테스트 샘플
└── README.md
```

---

## 2. 저장 위치 설계(크로스플랫폼/포터블)

**원칙:** OS 권장 디렉터리 사용 + 포터블 모드(실행 파일 옆 저장) 옵션

```csharp
// Config/Paths.cs
public static class Paths
{
    public static string GetSettingsDirectory(string? appName = "MyApp")
    {
        // 1) 포터블 모드 우선
        var portable = Environment.GetEnvironmentVariable("MYAPP_PORTABLE");
        if (!string.IsNullOrEmpty(portable))
        {
            var exeDir = AppContext.BaseDirectory;
            return Path.Combine(exeDir, "data");
        }

        // 2) OS 권장 위치
        var platform = Environment.OSVersion.Platform;
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
        // Linux/Unix
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

## 3. 모델: 버전/민감정보/유효성

```csharp
// Config/AppSettings.cs
using System.ComponentModel.DataAnnotations;

public class AppSettings
{
    // 스키마/마이그레이션용
    public int SchemaVersion { get; set; } = 1;

    [Required, RegularExpression(@"^(en|ko|ja)$")]
    public string Language { get; set; } = "ko";

    [Required, RegularExpression(@"^(Light|Dark)$")]
    public string Theme { get; set; } = "Light";

    // 민감정보(암호화 저장): 실제 저장은 EncryptedAuthToken 필드에
    public string? AuthToken { get; set; }
    public bool AutoLogin { get; set; } = false;

    // 내부 저장용 암호문 필드(직렬화 대상)
    public string? EncryptedAuthToken { get; set; }

    // 사용자 기타 설정(예시)
    public bool UseHardwareAcceleration { get; set; } = true;
    public int WindowWidth { get; set; } = 1280;
    public int WindowHeight { get; set; } = 800;
}
```

---

## 4. 암호화 서비스: AES-256-GCM

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
    // 키 관리: 실제 프로덕션에서는 OS 보호 API/DPAPI, KMS, .NET user-secrets 등을 고려
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

> 키는 예제로 `byte[32]` 고정. 실제로는 사용자별/장치별 안전한 저장소를 사용해야 한다.

---

## 5. 저장소 인터페이스

```csharp
// Config/ISettingsService.cs
public interface ISettingsService
{
    Task<AppSettings> LoadAsync(CancellationToken ct = default);
    Task SaveAsync(AppSettings settings, CancellationToken ct = default);
}
```

---

## 6. JSON 저장 구현(원자적 쓰기 + 백업/롤백)

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
            // 백업이 있다면 복구 시도
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

            // 복호화 → 메모리 상 AuthToken 채움
            if (!string.IsNullOrWhiteSpace(loaded.EncryptedAuthToken))
                loaded.AuthToken = _crypto.Decrypt(loaded.EncryptedAuthToken);

            // 스키마 마이그레이션
            MigrateIfNeeded(loaded);

            return loaded;
        }
        catch
        {
            // 손상 시 백업 복구 시도
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

        // 민감정보 암호화하여 저장 필드에 쓰기
        s.EncryptedAuthToken = string.IsNullOrWhiteSpace(s.AuthToken) ? null : _crypto.Encrypt(s.AuthToken);

        // 원자적 쓰기: 임시 파일 → Move
        var temp = Path.GetTempFileName();
        try
        {
            await using (var fs = File.Open(temp, FileMode.Create, FileAccess.Write, FileShare.None))
            {
                await JsonSerializer.SerializeAsync(fs, s, _opt, ct);
            }

            // 백업 갱신
            if (File.Exists(path))
                File.Copy(path, backup, overwrite: true);

            // 교체(가능하면 ReplaceFile 유사 동작)
            if (File.Exists(path)) File.Delete(path);
            File.Move(temp, path);
        }
        catch
        {
            // temp 삭제
            try { if (File.Exists(temp)) File.Delete(temp); } catch { /* ignore */ }
            throw;
        }
    }

    private static void MigrateIfNeeded(AppSettings s)
    {
        // 예: 스키마 1 → 2 업그레이드(샘플)
        // if (s.SchemaVersion < 2) { s.NewField = ...; s.SchemaVersion = 2; }
    }
}
```

**핵심 포인트**
- 임시파일 → Move 로 **원자적 쓰기** 보장(부분 쓰기/전원장애 대비)
- 저장 전 암호화, 로드 후 복호화
- 손상 시 **백업 복구** 경로
- `SchemaVersion` 기반 마이그레이션 훅

---

## 7. SQLite 저장 구현(스키마/마이그레이션)

**라이브러리:** `Microsoft.Data.Sqlite` (System.Data.SQLite 대체 가능)

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

        // 단일 레코드 테이블 가정(키=1)
        var cmd = conn.CreateCommand();
        cmd.CommandText = """
            SELECT SchemaVersion, Language, Theme, EncryptedAuthToken, AutoLogin, UseHardwareAcceleration, WindowWidth, WindowHeight
            FROM AppSettings
            WHERE Id = 1
        """;

        using var reader = await cmd.ExecuteReaderAsync(ct);

        if (!await reader.ReadAsync(ct))
            return new AppSettings(); // 기본값

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

        MigrateIfNeeded(conn, s, ct); // 필요 시 DB 업데이트
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
            INSERT INTO AppSettings(Id, SchemaVersion, Language, Theme, EncryptedAuthToken, AutoLogin, UseHardwareAcceleration, WindowWidth, WindowHeight)
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

        // 최초 레코드 없으면 시드
        var exists = conn.CreateCommand();
        exists.CommandText = "SELECT COUNT(*) FROM AppSettings WHERE Id=1";
        var count = (long)(await exists.ExecuteScalarAsync(ct) ?? 0L);
        if (count == 0)
        {
            var seed = conn.CreateCommand();
            seed.CommandText = """
                INSERT INTO AppSettings(Id, SchemaVersion, Language, Theme, EncryptedAuthToken, AutoLogin, UseHardwareAcceleration, WindowWidth, WindowHeight)
                VALUES(1, 1, 'ko', 'Light', NULL, 0, 1, 1280, 800)
            """;
            await seed.ExecuteNonQueryAsync(ct);
        }
    }

    private static void MigrateIfNeeded(SqliteConnection conn, AppSettings s, CancellationToken ct)
    {
        // 예: SchemaVersion < 2 → 열 추가/데이터 변환 등
        // if (s.SchemaVersion < 2) { ...; s.SchemaVersion = 2; Save... }
    }
}
```

**핵심 포인트**
- 단일 레코드 스키마(간단/안정)
- `ON CONFLICT(Id) DO UPDATE`로 UPSERT
- 마이그레이션 훅(`MigrateIfNeeded`) 제공
- 트랜잭션으로 **부분 저장 방지**

---

## 8. 전역 상태(AppState)와 반응형 연결

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

---

## 9. 파사드(SettingsFacade): 검증/동기화/자동 저장

```csharp
// Services/SettingsFacade.cs
using System.ComponentModel.DataAnnotations;

public sealed class SettingsFacade
{
    private readonly ISettingsService _repo;
    private readonly AppState _state;

    public SettingsFacade(ISettingsService repo, AppState state)
    {
        _repo = repo; _state = state;
    }

    public async Task<AppSettings> LoadIntoStateAsync(CancellationToken ct = default)
    {
        var s = await _repo.LoadAsync(ct);
        ApplyToState(s);
        return s;
    }

    public async Task SaveFromStateAsync(CancellationToken ct = default)
    {
        var s = FromState();
        Validate(s);
        await _repo.SaveAsync(s, ct);
    }

    public void ApplyToState(AppSettings s)
    {
        _state.Language = s.Language;
        _state.Theme = s.Theme;
        _state.AuthToken = s.AuthToken;
    }

    public AppSettings FromState() => new()
    {
        Language = _state.Language,
        Theme = _state.Theme,
        AuthToken = _state.AuthToken
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

> ViewModel은 저장소가 JSON인지 SQLite인지 신경 쓰지 않고 `SettingsFacade`를 호출한다.

---

## 10. App 초기화/DI 등록

```csharp
// App.axaml.cs (요지)
using Microsoft.Extensions.DependencyInjection;

private void ConfigureServices(IServiceCollection services)
{
    // 32바이트 키(예제): 반드시 안전한 저장소/프로비저닝으로 대체
    var key = Enumerable.Repeat((byte)0x11, 32).ToArray();

    services.AddSingleton<ICryptoService>(_ => new AesCryptoService(key));

    // 둘 중 하나를 선택 등록(런타임 스위치도 가능)
    services.AddSingleton<ISettingsService, JsonSettingsService>();
    // services.AddSingleton<ISettingsService, SqliteSettingsService>();

    services.AddSingleton<AppState>();
    services.AddSingleton<SettingsFacade>();

    // ViewModel 등...
}
```

앱 시작 시 로드:

```csharp
public override async void OnFrameworkInitializationCompleted()
{
    var sp = Services;
    var facade = sp.GetRequiredService<SettingsFacade>();

    var loaded = await facade.LoadIntoStateAsync();

    // 테마/언어 초기화(I18N/Theme 매니저와 결합 가능)
    // 예: ThemeManager.Apply(loaded.Theme); LocalizationManager.SetCulture(loaded.Language);

    // 자동 로그인 분기
    if (loaded.AutoLogin && !string.IsNullOrWhiteSpace(loaded.AuthToken))
        ShowMainWindow();
    else
        ShowLoginWindow();

    base.OnFrameworkInitializationCompleted();
}
```

---

## 11. Settings UI: ViewModel & View

```csharp
// ViewModels/SettingsViewModel.cs
using ReactiveUI;
using System.Reactive;
using System.Threading.Tasks;

public sealed class SettingsViewModel : ReactiveObject
{
    private readonly SettingsFacade _settings;
    private readonly AppState _state;

    public SettingsViewModel(SettingsFacade settings, AppState state)
    {
        _settings = settings; _state = state;

        SaveCommand = ReactiveCommand.CreateFromTask(SaveAsync);
        ReloadCommand = ReactiveCommand.CreateFromTask(ReloadAsync);
    }

    // 바인딩은 AppState에 그대로 연결(간단)
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
    public bool AutoLogin { get; set; } // JSON/SQLite 모델과 연결하려면 저장 시 반영

    public ReactiveCommand<Unit, Unit> SaveCommand { get; }
    public ReactiveCommand<Unit, Unit> ReloadCommand { get; }

    private async Task SaveAsync()
    {
        // AutoLogin 반영 예시
        var s = _settings.FromState();
        s.AutoLogin = AutoLogin;
        await _settings.SaveFromStateAsync();
    }

    private async Task ReloadAsync()
    {
        await _settings.LoadIntoStateAsync();
        this.RaisePropertyChanged(nameof(Language));
        this.RaisePropertyChanged(nameof(Theme));
    }
}
```

```xml
<!-- Views/SettingsView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:vm="clr-namespace:MyApp.ViewModels"
             x:Class="MyApp.Views.SettingsView">
  <UserControl.DataContext>
    <vm:SettingsViewModel />
  </UserControl.DataContext>

  <StackPanel Margin="20" Spacing="10">
    <TextBlock Text="설정" FontSize="20"/>

    <StackPanel Orientation="Horizontal" Spacing="8">
      <TextBlock Text="언어" Width="80"/>
      <ComboBox SelectedItem="{Binding Language}">
        <ComboBoxItem Content="ko"/>
        <ComboBoxItem Content="en"/>
        <ComboBoxItem Content="ja"/>
      </ComboBox>
    </StackPanel>

    <StackPanel Orientation="Horizontal" Spacing="8">
      <TextBlock Text="테마" Width="80"/>
      <ComboBox SelectedItem="{Binding Theme}">
        <ComboBoxItem Content="Light"/>
        <ComboBoxItem Content="Dark"/>
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

## 12. 로그인 후 토큰 저장(민감정보 암호화)

```csharp
// ViewModels/LoginViewModel.cs (요지)
public sealed class LoginViewModel : ReactiveObject
{
    private readonly SettingsFacade _facade;
    private readonly AppState _state;

    public LoginViewModel(SettingsFacade facade, AppState state)
    {
        _facade = facade; _state = state;

        LoginCommand = ReactiveCommand.CreateFromTask(LoginAsync);
    }

    public string Username { get; set; } = "";
    public string Password { get; set; } = "";

    public ReactiveCommand<Unit, bool> LoginCommand { get; }

    private async Task<bool> LoginAsync()
    {
        // 실제 인증 로직
        var ok = Username == "admin" && Password == "1234";
        if (!ok) return false;

        // 예제 토큰
        _state.AuthToken = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";
        // 필요 시 AutoLogin도 온
        var s = _facade.FromState();
        s.AutoLogin = true;
        await _facade.SaveFromStateAsync();
        return true;
    }
}
```

---

## 13. 실패/경합/잠금/손상 대응

- **JSON**
  - 임시 파일 → Move로 **원자성**
  - 쓰기 전에 **백업** 생성, 로드시 손상 시 **백업 복구**
  - 파일 잠금은 OS별로 보수적 공유 모드 사용
- **SQLite**
  - 트랜잭션 필수
  - PRAGMA 설정(필요 시 `journal_mode=WAL`)로 동시 접근 안정성 개선
  - 백업은 파일 복사 또는 `VACUUM INTO` 활용 가능

---

## 14. 성능/최적화

- 설정은 일반적으로 **작고 드문 쓰기** → JSON이 간단/빠름
- 복잡한 구조/조회/부분 업데이트/버전 다중 관리 → SQLite가 유리
- 대량 I/O 시 배치 적용(메모리 캐시 → 주기 저장)

---

## 15. 단위 테스트(요지)

```csharp
// Test/SettingsTests.cs
using Xunit;
using FluentAssertions;

public sealed class SettingsTests
{
    [Fact]
    public async Task Json_SaveLoad_Roundtrip()
    {
        // 임시 경로(포터블 모드)
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

        var loaded = await repo.LoadAsync();

        loaded.Language.Should().Be("en");
        loaded.Theme.Should().Be("Dark");
        loaded.AuthToken.Should().Be("secret");
    }

    [Fact]
    public async Task Sqlite_SaveLoad_Roundtrip()
    {
        Environment.SetEnvironmentVariable("MYAPP_PORTABLE", "1");

        var key = Enumerable.Repeat((byte)0x33, 32).ToArray();
        var crypto = new AesCryptoService(key);
        var repo = new SqliteSettingsService(crypto);

        var s = await repo.LoadAsync();
        s.Language = "ja";
        s.Theme = "Dark";
        s.AuthToken = "tok";
        await repo.SaveAsync(s);

        var s2 = await repo.LoadAsync();
        s2.Language.Should().Be("ja");
        s2.Theme.Should().Be("Dark");
        s2.AuthToken.Should().Be("tok");
    }
}
```

---

## 16. 검증/에러 메시지(UI와 결합)

- 저장 전 `DataAnnotations`로 **유효성 검사**
- UI에서는 `ReactiveUI.Validation` 또는 `INotifyDataErrorInfo`로 바인딩
- 실패 시 사용자에게 토스트/다이얼로그로 안내

---

## 17. 테마/언어/i18n과의 결합(요점)

- `AppState.Theme` 변경 → ThemeManager(리소스 교체) 호출
- `AppState.Language` 변경 → LocalizationManager.SetCulture → ViewModels `RaisePropertyChanged`
- 설정 저장 시 이 두 값은 **즉시 반영**되어 다음 실행에도 동일

---

## 18. 저장 방식 선택 가이드(확장판)

| 기준 | JSON | SQLite |
|------|------|--------|
| 설정 규모가 작고 단순 | 매우 적합 | 과한 선택일 수 있음 |
| 다계층/복잡/조회/필터 | 한계 | 적합 |
| 원자성/백업/복구 | 임시파일+백업으로 해결 | 트랜잭션으로 자연스러움 |
| 마이그레이션 | 코드 변환/키 이동 | DDL/데이터 변환 |
| 외부 수정/툴링 | 손쉬움(에디터) | SQL 도구 필요 |
| 성능 | 매우 빠름 | 충분히 빠름 |

---

## 19. 체크리스트(운영 관점)

- [ ] 저장 경로 존재/권한 확인
- [ ] JSON 원자적 쓰기 + 백업/복구
- [ ] SQLite 트랜잭션/압축/VACUUM 전략
- [ ] 민감정보 암호화 강제(`AuthToken` 등)
- [ ] 스키마 버전 및 마이그레이션 코드 유지
- [ ] 예외 로깅(Serilog 등)
- [ ] 단위/통합 테스트(CI)
- [ ] 포터블 모드/리소스 경로 오버라이드 옵션
- [ ] 다국어/테마와의 즉시 반영

---

## 20. 결론

- **인터페이스(`ISettingsService`)로 저장소를 추상화**하면 JSON ↔ SQLite 전환이 쉽다.
- **원자적 쓰기/백업/트랜잭션/암호화/마이그레이션**은 실제 배포에서 반드시 필요하다.
- `AppState`(+ Theme/Localization 매니저)와 결합해 **UI/경험 전반을 일관되게 유지**할 수 있다.
- 단위 테스트로 회귀를 막고, CI에서 **GUI 없는 빠른 검증**이 가능하다.

---

## 부록 A) DI 예시(App.axaml.cs)

```csharp
private void ConfigureServices(IServiceCollection services)
{
    var key = Enumerable.Repeat((byte)0x11, 32).ToArray();
    services.AddSingleton<ICryptoService>(_ => new AesCryptoService(key));

    // 스위치로 저장 엔진 선택
    var engine = Environment.GetEnvironmentVariable("MYAPP_SETTINGS_ENGINE") ?? "json";
    if (engine.Equals("sqlite", StringComparison.OrdinalIgnoreCase))
        services.AddSingleton<ISettingsService, SqliteSettingsService>();
    else
        services.AddSingleton<ISettingsService, JsonSettingsService>();

    services.AddSingleton<AppState>();
    services.AddSingleton<SettingsFacade>();

    services.AddTransient<SettingsViewModel>();
    services.AddTransient<LoginViewModel>();
}
```

---

## 부록 B) 수학적 표기(버전 정책의 간단한 모델)

스키마 버전의 업데이트를 **그래프**로 생각하면, 각 노드는 버전, 간선은 마이그레이션 함수이다:

$$
\mathcal{V} = \{1,2,\ldots\},\quad
\mathcal{M}_{i\to j} : S_i \to S_j,\quad i<j
$$

**불변식**: 모든 런타임 시점에 로드된 설정 \( s \in S_i \) 에 대해 유효한 경로 \( i \to \cdots \to N \) 가 존재해야 하며, 최종형 \( S_N \) 은 코드가 기대하는 현재 스키마다.
