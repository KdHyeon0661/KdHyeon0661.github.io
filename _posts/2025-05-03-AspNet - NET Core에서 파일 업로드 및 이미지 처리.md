---
layout: post
title: AspNet - NET Core에서 파일 업로드 및 이미지 처리
date: 2025-05-03 20:20:23 +0900
category: AspNet
---
# ASP.NET Core에서 파일 업로드 및 이미지 처리

## 0. 요구사항 정리와 아키텍처 선택

| 상황 | 권장 아키텍처 |
|---|---|
| 소규모, 단일 인스턴스 | 로컬 디스크 저장 + In-Process 썸네일 |
| 다중 인스턴스, 수평 확장 | 프리사인(또는 SAS) 직업로드 → 오브젝트 스토리지(Blob/S3) + 백그라운드 가공 |
| 초대용량(수백 MB~GB) | 조각 업로드(Resumable) + 백엔드 메타데이터만 관리 |
| 규제/감사 필요 | 해시/원본 보존, 업로드 감사 로그, 승인 워크플로 |
| 민감 파일 | 암호화 저장(KMS/KeyVault), 기간 제한 서명 URL로만 배포 |

---

## 1. 기본: Razor Pages 단일 파일 업로드(보안 강화 버전)

### 1.1 폴더 구조
```
MyApp/
├── Pages/
│   ├── Upload.cshtml
│   └── Upload.cshtml.cs
├── Services/
│   ├── FileValidator.cs
│   └── ImagePipeline.cs
├── wwwroot/
│   └── uploads/
```

### 1.2 업로드 폼
```razor
@page
@model UploadModel
@{
    ViewData["Title"] = "파일 업로드";
}
<form method="post" enctype="multipart/form-data" asp-antiforgery="true">
    <input type="file" name="UploadFile" accept=".jpg,.jpeg,.png,.webp" />
    <button type="submit">Upload</button>
</form>

@if (!string.IsNullOrEmpty(Model.SavedFileName))
{
    <p>저장됨: @Model.SavedFileName</p>
    <img src="/uploads/@Model.SavedFileName" width="180" />
}
```

### 1.3 백엔드: 유효성 검사 + 안전 저장
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.AspNetCore.Http.Features;
using System.Security.Cryptography;

public class UploadModel : PageModel
{
    private readonly IWebHostEnvironment _env;
    private readonly FileValidator _validator;

    public UploadModel(IWebHostEnvironment env, FileValidator validator)
    {
        _env = env; _validator = validator;
    }

    [BindProperty] public IFormFile? UploadFile { get; set; }
    public string? SavedFileName { get; set; }

    public async Task<IActionResult> OnPostAsync()
    {
        if (UploadFile is null || UploadFile.Length == 0)
        {
            ModelState.AddModelError("", "파일을 선택하세요.");
            return Page();
        }

        if (!_validator.IsAllowedExtension(UploadFile.FileName, new[] { ".jpg", ".jpeg", ".png", ".webp" }))
        {
            ModelState.AddModelError("", "허용되지 않는 확장자입니다.");
            return Page();
        }

        if (UploadFile.Length > 10 * 1024 * 1024) // 10MB
        {
            ModelState.AddModelError("", "최대 10MB까지 업로드 가능합니다.");
            return Page();
        }

        // MIME(Reported) 검증
        if (!_validator.IsAllowedContentType(UploadFile.ContentType, new[] { "image/jpeg", "image/png", "image/webp" }))
        {
            ModelState.AddModelError("", "허용되지 않는 MIME 타입입니다.");
            return Page();
        }

        // 매직 넘버(파일 시그니처) 검증
        using var sigStream = UploadFile.OpenReadStream();
        if (!await _validator.IsValidSignatureAsync(sigStream))
        {
            ModelState.AddModelError("", "파일 시그니처가 유효하지 않습니다.");
            return Page();
        }
        sigStream.Position = 0;

        // 안전 파일명(경로 탐색 방지)
        var safeName = $"{Guid.NewGuid()}{Path.GetExtension(UploadFile.FileName).ToLowerInvariant()}";
        var uploadDir = Path.Combine(_env.WebRootPath, "uploads");
        Directory.CreateDirectory(uploadDir);
        var filePath = Path.Combine(uploadDir, safeName);

        // 저장(스트리밍)
        await using (var fs = new FileStream(filePath, FileMode.CreateNew, FileAccess.Write, FileShare.None, 64 * 1024, useAsync: true))
        {
            await sigStream.CopyToAsync(fs);
        }

        // (선택) 해시 기록/AV 스캔 훅
        // var hash = await _validator.ComputeSha256Async(filePath);
        // await _antivirus.ScanAsync(filePath); // 외부 AV 연동 훅

        SavedFileName = safeName;
        return Page();
    }
}
```

### 1.4 유효성 검사 유틸
```csharp
public class FileValidator
{
    private static readonly Dictionary<string, byte[][]> _signatures = new()
    {
        // JPEG: FF D8 FF
        [".jpg"] = new[] { new byte[] { 0xFF, 0xD8, 0xFF } },
        [".jpeg"] = new[] { new byte[] { 0xFF, 0xD8, 0xFF } },
        // PNG: 89 50 4E 47 0D 0A 1A 0A
        [".png"] = new[] { new byte[] { 0x89,0x50,0x4E,0x47,0x0D,0x0A,0x1A,0x0A } },
        // WEBP: "RIFF....WEBP"
        [".webp"] = new[] { new byte[] { 0x52,0x49,0x46,0x46 } }
    };

    public bool IsAllowedExtension(string fileName, string[] whitelist)
    {
        var ext = Path.GetExtension(fileName).ToLowerInvariant();
        return whitelist.Contains(ext);
    }

    public bool IsAllowedContentType(string contentType, string[] whitelist)
        => whitelist.Contains(contentType);

    public async Task<bool> IsValidSignatureAsync(Stream s)
    {
        var header = new byte[12];
        var read = await s.ReadAsync(header);
        if (read < 3) return false;

        // 가장 짧은 서명 기준으로 확인
        foreach (var kv in _signatures)
        {
            foreach (var sig in kv.Value)
            {
                if (header.Length >= sig.Length && header.AsSpan(0, sig.Length).SequenceEqual(sig))
                {
                    // WEBP는 추가 확인: "WEBP" 존재 여부(12바이트 내)
                    if (kv.Key == ".webp")
                    {
                        var ascii = System.Text.Encoding.ASCII.GetString(header);
                        if (!ascii.Contains("WEBP")) return false;
                    }
                    return true;
                }
            }
        }
        return false;
    }

    public async Task<string> ComputeSha256Async(string path)
    {
        await using var fs = File.OpenRead(path);
        var hash = await SHA256.HashDataAsync(fs);
        return Convert.ToHexString(hash);
    }
}
```

---

## 2. 다중 파일 업로드(대용량 안전 처리)

```razor
@page
@model MultiUploadModel
<form method="post" enctype="multipart/form-data">
    <input type="file" name="UploadFiles" multiple accept=".jpg,.jpeg,.png,.webp" />
    <button type="submit">Upload</button>
</form>
<ul>
@foreach (var f in Model.Saved)
{
    <li>@f</li>
}
</ul>
```

```csharp
public class MultiUploadModel : PageModel
{
    private readonly IWebHostEnvironment _env; private readonly FileValidator _validator;
    public MultiUploadModel(IWebHostEnvironment env, FileValidator validator) { _env = env; _validator = validator; }

    [BindProperty] public List<IFormFile> UploadFiles { get; set; } = new();
    public List<string> Saved { get; set; } = new();

    public async Task<IActionResult> OnPostAsync()
    {
        var uploadDir = Path.Combine(_env.WebRootPath, "uploads");
        Directory.CreateDirectory(uploadDir);

        foreach (var file in UploadFiles.Where(f => f.Length > 0))
        {
            if (!_validator.IsAllowedExtension(file.FileName, new[] { ".jpg", ".jpeg", ".png", ".webp" })) continue;
            if (!_validator.IsAllowedContentType(file.ContentType, new[] { "image/jpeg", "image/png", "image/webp" })) continue;

            using var sig = file.OpenReadStream();
            if (!await _validator.IsValidSignatureAsync(sig)) continue;
            sig.Position = 0;

            var safe = $"{Guid.NewGuid()}{Path.GetExtension(file.FileName).ToLowerInvariant()}";
            var path = Path.Combine(uploadDir, safe);
            await using var fs = new FileStream(path, FileMode.CreateNew, FileAccess.Write, FileShare.None, 64 * 1024, true);
            await sig.CopyToAsync(fs);
            Saved.Add(safe);
        }
        return Page();
    }
}
```

---

## 3. 업로드 제한 및 서버 보호

### 3.1 글로벌 제한(Program.cs)
```csharp
using Microsoft.AspNetCore.Http.Features;

builder.Services.Configure<FormOptions>(o =>
{
    o.MultipartBodyLengthLimit = 10L * 1024 * 1024; // 10MB
    o.MultipartHeadersLengthLimit = 64 * 1024;
});
```

### 3.2 엔드포인트별 제한
```csharp
[RequestSizeLimit(10_000_000)]
public IActionResult Upload() => View();
```

### 3.3 레이트리미팅(.NET 8)
```csharp
builder.Services.AddRateLimiter(_ => _.AddTokenBucketLimiter("upload", options =>
{
    options.TokenLimit = 10; options.QueueLimit = 0;
    options.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
    options.TokensPerPeriod = 10;
}));

app.UseRateLimiter();
app.MapPost("/upload", async context => { /* ... */ }).RequireRateLimiting("upload");
```

---

## 4. 이미지 처리 파이프라인(ImageSharp)

### 4.1 리사이즈/포맷/품질/EXIF 제거
```csharp
using SixLabors.ImageSharp;
using SixLabors.ImageSharp.Processing;
using SixLabors.ImageSharp.Formats.Jpeg;
using SixLabors.ImageSharp.Metadata.Profiles.Exif;

public class ImagePipeline
{
    public async Task ProcessAsync(Stream input, string outputPath, int maxW, int maxH, int quality = 80)
    {
        using var image = await Image.LoadAsync(input);
        // EXIF 제거(민감 메타정보 삭제)
        image.Metadata.ExifProfile = null;

        // 비율 유지 리사이즈(긴 변 기준)
        var ratio = Math.Min((double)maxW / image.Width, (double)maxH / image.Height);
        if (ratio < 1.0)
        {
            var w = (int)(image.Width * ratio);
            var h = (int)(image.Height * ratio);
            image.Mutate(x => x.Resize(new Size(w, h)));
        }

        var encoder = new JpegEncoder { Quality = quality };
        await image.SaveAsJpegAsync(outputPath, encoder);
    }

    public async Task GenerateThumbnailsAsync(string originalPath, string outDir)
    {
        Directory.CreateDirectory(outDir);
        var sizes = new[] { 128, 256, 512 };
        foreach (var s in sizes)
        {
            await using var fs = File.OpenRead(originalPath);
            using var img = await Image.LoadAsync(fs);
            img.Metadata.ExifProfile = null;
            img.Mutate(x => x.Resize(new ResizeOptions {
                Size = new Size(s, s),
                Mode = ResizeMode.Max
            }));
            var thumbPath = Path.Combine(outDir, $"{Path.GetFileNameWithoutExtension(originalPath)}_{s}.jpg");
            await img.SaveAsJpegAsync(thumbPath, new JpegEncoder { Quality = 80 });
        }
    }
}
```

### 4.2 워터마크/크롭 예시
```csharp
image.Mutate(x => x
    .Crop(new Rectangle(10, 10, 400, 400))
    .DrawText("MySite", yourFont, Color.White, new PointF(10, image.Height - 30)));
```

---

## 5. AJAX 업로드(Minimal API)와 진척도 표시

### 5.1 클라이언트
```html
<input type="file" id="fileInput" />
<progress id="pg" value="0" max="100"></progress>
<script>
const input = document.getElementById('fileInput');
const pg = document.getElementById('pg');

input.addEventListener('change', async () => {
  const file = input.files[0];
  const form = new FormData();
  form.append('file', file);
  const xhr = new XMLHttpRequest();
  xhr.open('POST', '/api/upload');
  xhr.upload.onprogress = e => {
    if (e.lengthComputable) pg.value = e.loaded / e.total * 100;
  };
  xhr.onload = () => alert('done');
  xhr.send(form);
});
</script>
```

### 5.2 서버(Minimal API)
```csharp
app.MapPost("/api/upload", async (HttpRequest req, IWebHostEnvironment env, FileValidator validator) =>
{
    var form = await req.ReadFormAsync();
    var file = form.Files["file"];
    if (file is null || file.Length == 0) return Results.BadRequest("file required");

    if (!validator.IsAllowedExtension(file.FileName, new[] { ".jpg",".jpeg",".png",".webp" })) return Results.BadRequest("ext");
    if (!validator.IsAllowedContentType(file.ContentType, new[] { "image/jpeg","image/png","image/webp" })) return Results.BadRequest("type");

    using var s = file.OpenReadStream();
    if (!await validator.IsValidSignatureAsync(s)) return Results.BadRequest("sig");
    s.Position = 0;

    var name = $"{Guid.NewGuid()}{Path.GetExtension(file.FileName)}";
    var path = Path.Combine(env.WebRootPath, "uploads", name);
    Directory.CreateDirectory(Path.GetDirectoryName(path)!);

    await using var fs = new FileStream(path, FileMode.CreateNew, FileAccess.Write, FileShare.None, 64*1024, true);
    await s.CopyToAsync(fs);

    return Results.Ok(new { file = name, url = $"/uploads/{name}" });
});
```

---

## 6. 대용량 스트리밍(BodyReader)으로 메모리 사용 줄이기

```csharp
app.MapPost("/api/upload/stream", async (HttpContext ctx, IWebHostEnvironment env) =>
{
    var boundary = MultipartRequestHelper.GetBoundary(MediaTypeHeaderValue.Parse(ctx.Request.ContentType), 70);
    var reader = new MultipartReader(boundary, ctx.Request.Body);
    MultipartSection? section;

    var uploadDir = Path.Combine(env.WebRootPath, "uploads");
    Directory.CreateDirectory(uploadDir);

    while ((section = await reader.ReadNextSectionAsync()) != null)
    {
        var hasFileContentDisposition = ContentDispositionHeaderValue.TryParse(section.ContentDisposition, out var cd)
            && MultipartRequestHelper.HasFileContentDisposition(cd!);

        if (hasFileContentDisposition)
        {
            var ext = Path.GetExtension(cd!.FileName.Value ?? "").ToLowerInvariant();
            var safe = $"{Guid.NewGuid()}{ext}";
            var path = Path.Combine(uploadDir, safe);
            await using var target = File.Create(path);
            await section.Body.CopyToAsync(target); // 청크 단위로 바로 디스크에
        }
    }
    return Results.Ok();
});
```

> `MultipartRequestHelper`는 표준 샘플 패턴으로 구현(경계 추출/검사). 이 방식은 서버 메모리에 파일 전체를 올리지 않고 곧장 디스크로 스트리밍한다.

---

## 7. 조각(Resumable) 업로드(간단 버전)

1) 클라이언트가 업로드 세션 생성 요청 → 서버가 `uploadId` 반환  
2) 각 조각(chunkIndex, totalChunks)로 전송 → 서버는 임시 디렉터리에 저장  
3) 마지막 조각 도착 시 서버가 머지하고 무결성 해시 검증 → 완료

### 7.1 세션 생성
```csharp
app.MapPost("/api/chunk/init", () =>
{
    var uploadId = Guid.NewGuid().ToString("N");
    return Results.Ok(new { uploadId });
});
```

### 7.2 조각 업로드
```csharp
app.MapPost("/api/chunk/{uploadId}/{index:int}/{total:int}", async (string uploadId, int index, int total, HttpRequest req, IWebHostEnvironment env) =>
{
    var tmpDir = Path.Combine(env.ContentRootPath, "temp", uploadId);
    Directory.CreateDirectory(tmpDir);
    var chunkPath = Path.Combine(tmpDir, $"{index:D6}.part");

    await using var fs = new FileStream(chunkPath, FileMode.Create, FileAccess.Write, FileShare.None, 64*1024, true);
    await req.Body.CopyToAsync(fs);

    if (Directory.EnumerateFiles(tmpDir, "*.part").Count() == total)
    {
        // 머지
        var finalName = $"{uploadId}.bin";
        var finalPath = Path.Combine(env.WebRootPath, "uploads", finalName);
        Directory.CreateDirectory(Path.GetDirectoryName(finalPath)!);

        await using var outFs = new FileStream(finalPath, FileMode.Create, FileAccess.Write, FileShare.None, 256*1024, true);
        for (int i = 0; i < total; i++)
        {
            var part = Path.Combine(tmpDir, $"{i:D6}.part");
            await using var pfs = File.OpenRead(part);
            await pfs.CopyToAsync(outFs);
        }
        Directory.Delete(tmpDir, true);
        return Results.Ok(new { file = finalName });
    }
    return Results.Accepted();
});
```

> 운영에서는 체크섬(각 조각/전체), 타임아웃 정리 작업, 업로드 중단 재개 로직을 덧붙인다.

---

## 8. 다운로드 보호 및 직접 접근 차단

### 8.1 공개 디렉터리 대신 컨트롤러를 통한 제공
```csharp
[ApiController]
[Route("files")]
public class FilesController : ControllerBase
{
    private readonly IWebHostEnvironment _env;
    public FilesController(IWebHostEnvironment env) => _env = env;

    [HttpGet("{id}")]
    public IActionResult Get(string id)
    {
        // 권한 검증 로직
        var path = Path.Combine(_env.ContentRootPath, "protected", id);
        if (!System.IO.File.Exists(path)) return NotFound();
        var contentType = "application/octet-stream";
        return PhysicalFile(path, contentType, enableRangeProcessing: true);
    }
}
```

### 8.2 서명 URL(짧은 TTL) 발급 패턴
- 다운로드 URL에 `token=HMAC(fileId, expires)` 추가
- 미들웨어에서 토큰/만료 검증 후 통과

```csharp
public static string Sign(string input, string secret)
{
    using var h = new System.Security.Cryptography.HMACSHA256(System.Text.Encoding.UTF8.GetBytes(secret));
    var sig = h.ComputeHash(System.Text.Encoding.UTF8.GetBytes(input));
    return Convert.ToHexString(sig);
}
```

---

## 9. 클라우드 스토리지(Blob/S3) 연동

### 9.1 Azure Blob 기본 업로드
```csharp
// dotnet add package Azure.Storage.Blobs
var blob = new BlobClient(connectionString, "uploads", fileName);
await using var stream = file.OpenReadStream();
await blob.UploadAsync(stream, overwrite: false);
```

### 9.2 SAS로 브라우저 직업로드(백엔드 부하 제로)
- 서버: **쓰기 권한이 담긴 SAS URL**을 짧게 발급
- 프런트: 그 URL에 바로 PUT 업로드

```csharp
// SAS 발급(개략)
var blobUri = new Uri($"https://{account}.blob.core.windows.net/uploads/{fileName}");
var sas = new BlobSasBuilder(BlobSasPermissions.Write, DateTimeOffset.UtcNow.AddMinutes(5)) { BlobContainerName="uploads", BlobName=fileName };
var sig = sas.ToSasQueryParameters(new StorageSharedKeyCredential(account, key));
var signedUrl = new UriBuilder(blobUri) { Query = sig.ToString() }.Uri;
return Results.Ok(new { url = signedUrl.ToString() });
```

### 9.3 AWS S3 Presigned URL
```csharp
// dotnet add package AWSSDK.S3
var request = new GetPreSignedUrlRequest
{
    BucketName = "my-bucket",
    Key = fileName,
    Verb = HttpVerb.PUT,
    Expires = DateTime.UtcNow.AddMinutes(5),
    ContentType = "image/jpeg"
};
var url = s3Client.GetPreSignedURL(request);
```

> 대형 파일은 **직업로드 + 이벤트(BlobTrigger/S3 Event)** 로 썸네일 파이프라인을 서버리스로 수행하면 확장성이 매우 좋다.

---

## 10. 백그라운드 작업으로 썸네일/워터마크 처리

```csharp
public class ImageJobQueue
{
    private readonly Channel<string> _queue = Channel.CreateUnbounded<string>();
    public ValueTask EnqueueAsync(string path) => _queue.Writer.WriteAsync(path);
    public IAsyncEnumerable<string> DequeueAsync(CancellationToken ct) => _queue.Reader.ReadAllAsync(ct);
}

public class ImageWorker : BackgroundService
{
    private readonly ImageJobQueue _q; private readonly ImagePipeline _pipe; private readonly ILogger<ImageWorker> _log;
    private readonly IWebHostEnvironment _env;

    public ImageWorker(ImageJobQueue q, ImagePipeline pipe, ILogger<ImageWorker> log, IWebHostEnvironment env)
    { _q = q; _pipe = pipe; _log = log; _env = env; }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var path in _q.DequeueAsync(stoppingToken))
        {
            try
            {
                var thumbs = Path.Combine(_env.WebRootPath, "uploads", "thumbs");
                await _pipe.GenerateThumbnailsAsync(path, thumbs);
            }
            catch (Exception ex) { _log.LogError(ex, "Thumbnail failed: {Path}", path); }
        }
    }
}
```

등록:
```csharp
builder.Services.AddSingleton<ImageJobQueue>();
builder.Services.AddHostedService<ImageWorker>();
builder.Services.AddSingleton<ImagePipeline>();
```

업로드 완료 시 큐잉:
```csharp
await _queue.EnqueueAsync(filePath);
```

---

## 11. CORS, CSRF, 헤더 보안

- 폼 업로드는 CSRF 보호(기본 `@Html.AntiForgeryToken()` / Razor Pages는 자동)
- AJAX 업로드는 `RequestVerificationToken` 헤더로 CSRF 토큰 전달
- 외부 출처에서 업로드 시 CORS 허용 도메인 명시
- `Content-Security-Policy`로 업로드 페이지의 스크립트/이미지 출처 제한
- 정적 디렉터리에는 `X-Content-Type-Options: nosniff` 권장

---

## 12. 로깅/감사/관찰

- 업로드 결과: 파일명, 원본명, 크기, 해시, 사용자ID, IP, User-Agent
- 처리 파이프라인: 단계별 소요시간(MiniProfiler), 실패 사유
- 저장 계층별 오류: 디스크/네트워크/권한 예외 로깅
- 용량/쿼터: 사용자별 총 용량, 파일 수 제한

예시(성공 로그):
```csharp
_logger.LogInformation("Upload ok {UserId} {File} {Size} {Sha256}",
    userId, savedName, file.Length, await _validator.ComputeSha256Async(filePath));
```

---

## 13. 테스트(xUnit + Integration)

### 13.1 단위: 검증기
```csharp
public class FileValidatorTests
{
    [Fact]
    public async Task Should_Reject_Invalid_Signature()
    {
        var v = new FileValidator();
        using var ms = new MemoryStream(new byte[] { 0x00, 0x01, 0x02 });
        var ok = await v.IsValidSignatureAsync(ms);
        Assert.False(ok);
    }
}
```

### 13.2 통합: 업로드 엔드포인트
```csharp
public class UploadIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    public UploadIntegrationTests(WebApplicationFactory<Program> f) => _client = f.CreateClient();

    [Fact]
    public async Task Upload_Image_Should_Return_Url()
    {
        using var form = new MultipartFormDataContent();
        await using var fs = File.OpenRead("testdata/valid.jpg");
        var file = new StreamContent(fs);
        file.Headers.ContentType = new System.Net.Http.Headers.MediaTypeHeaderValue("image/jpeg");
        form.Add(file, "file", "x.jpg");
        var rsp = await _client.PostAsync("/api/upload", form);
        rsp.EnsureSuccessStatusCode();
        var json = await rsp.Content.ReadAsStringAsync();
        Assert.Contains("/uploads/", json);
    }
}
```

---

## 14. 운영 체크리스트

- 확장자/MIME/시그니처 **3중 검증**
- 파일명 랜덤화, 경로 탐색 방지, 전용 디렉터리(권한 최소)
- 크기/헤더 제한, 레이트리밋, 요청 타임아웃
- 이미지 메타 제거(EXIF), 썸네일/워터마크 **백그라운드 처리**
- 공개 경로 최소화(민감 파일은 컨트롤러를 통해서만), 서명 URL
- 오브젝트 스토리지 직업로드(SAS/Presigned), 백엔드 부하 분리
- 로그/감사/알림, 디스크/버킷 사용량 모니터링
- 조각 업로드는 재시도/정리 작업/무결성 검증 포함

---

## 15. 구성 스니펫 모음

### 15.1 업로드 제한 응답을 ProblemDetails로
```csharp
app.Use(async (ctx, next) =>
{
    try { await next(); }
    catch (BadHttpRequestException ex) when (ex.Message.Contains("Request body too large"))
    {
        ctx.Response.StatusCode = StatusCodes.Status413PayloadTooLarge;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = 413, Title = "Payload too large", Detail = "파일 크기 제한을 초과했습니다."
        });
    }
});
```

### 15.2 wwwroot/uploads/{yyyy}/{MM}/ 경로
```csharp
var now = DateTime.UtcNow;
var dir = Path.Combine(_env.WebRootPath, "uploads", now.ToString("yyyy"), now.ToString("MM"));
Directory.CreateDirectory(dir);
var safeName = $"{Guid.NewGuid()}{Path.GetExtension(file.FileName).ToLowerInvariant()}";
var path = Path.Combine(dir, safeName);
```

---

## 결론

- **핵심은 안전성**(검증·격리·최소권한)과 **확장성**(직업로드·백그라운드 파이프라인)이다.  
- 작은 서비스는 로컬 저장 + 동기 처리로 시작해도 된다. 트래픽과 파일 크기가 커질수록 **직업로드(SAS/Presigned)**, **조각 업로드**, **서버리스 썸네일**로 진화시키자.  
- 항상 **측정/로그/알림**을 통해 병목과 실패를 가시화하고, 규제 환경이라면 **감사 추적(해시/메타/사용자/시점)** 을 남겨야 한다.