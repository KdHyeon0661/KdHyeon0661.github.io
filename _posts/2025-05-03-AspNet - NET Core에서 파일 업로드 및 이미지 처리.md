---
layout: post
title: AspNet - NET Core에서 파일 업로드 및 이미지 처리
date: 2025-05-03 20:20:23 +0900
category: AspNet
---
# 📂 ASP.NET Core에서 파일 업로드 및 이미지 처리

---

## ✅ 1. 개요

ASP.NET Core에서는 **폼 기반** 또는 **AJAX 기반**으로  
파일(이미지, 문서 등)을 업로드할 수 있어요.

### 주요 처리 항목:

| 항목 | 설명 |
|------|------|
| 파일 수신 | `IFormFile` 객체를 통해 서버 수신 |
| 저장 | 디스크, DB, 클라우드 저장소 |
| 보안 | 파일 확장자, 크기, 경로 검증 |
| 가공 | 이미지 리사이징, 썸네일, 워터마크 |
| 반환 | 이미지 미리보기, 다운로드 링크 제공 |

---

## 🧾 2. Razor Page 예제 (단일 파일 업로드)

### 📁 프로젝트 구조 예시

```
MyApp/
├── Pages/
│   └── Upload.cshtml
│   └── Upload.cshtml.cs
├── wwwroot/
│   └── uploads/
```

### 📄 Upload.cshtml

```razor
<form method="post" enctype="multipart/form-data">
    <input type="file" name="UploadFile" />
    <button type="submit">Upload</button>
</form>
```

### 📄 Upload.cshtml.cs (백엔드)

```csharp
public class UploadModel : PageModel
{
    [BindProperty]
    public IFormFile UploadFile { get; set; }

    public async Task<IActionResult> OnPostAsync()
    {
        if (UploadFile != null && UploadFile.Length > 0)
        {
            var fileName = Path.GetFileName(UploadFile.FileName);
            var filePath = Path.Combine("wwwroot/uploads", fileName);

            using (var stream = new FileStream(filePath, FileMode.Create))
            {
                await UploadFile.CopyToAsync(stream);
            }

            return RedirectToPage("Success");
        }

        return Page();
    }
}
```

---

## 🔒 3. 보안 고려사항

| 항목 | 설명 |
|------|------|
| 확장자 제한 | `.jpg`, `.png` 등 허용된 확장자만 |
| 크기 제한 | 최대 업로드 크기 제한 (`[RequestSizeLimit]`) |
| 저장 경로 검증 | `Path.Combine`으로 경로 탐색 공격 방지 |
| Content-Type 검증 | `image/jpeg` 등 MIME 확인 |
| 파일명 변조 방지 | GUID 또는 Hash로 이름 변경 추천

### 🔐 예시: 파일명 무작위화

```csharp
var safeFileName = $"{Guid.NewGuid()}{Path.GetExtension(UploadFile.FileName)}";
```

---

## 🌠 4. 이미지 리사이징 (SixLabors.ImageSharp 활용)

### 🔹 NuGet 설치

```bash
dotnet add package SixLabors.ImageSharp
```

### 🔹 이미지 가공 예시

```csharp
using SixLabors.ImageSharp;
using SixLabors.ImageSharp.Processing;

public async Task SaveResizedImage(IFormFile file)
{
    using var image = Image.Load(file.OpenReadStream());
    image.Mutate(x => x.Resize(300, 300));

    var savePath = Path.Combine("wwwroot/uploads", Guid.NewGuid() + ".jpg");
    await image.SaveAsync(savePath);
}
```

> ✔ `Resize`, `Crop`, `Grayscale`, `Watermark` 등 다양한 가공 가능

---

## 📤 5. 다중 파일 업로드

```razor
<form method="post" enctype="multipart/form-data">
    <input type="file" name="UploadFiles" multiple />
    <button type="submit">Upload</button>
</form>
```

```csharp
[BindProperty]
public List<IFormFile> UploadFiles { get; set; }

public async Task<IActionResult> OnPostAsync()
{
    foreach (var file in UploadFiles)
    {
        if (file.Length > 0)
        {
            var path = Path.Combine("wwwroot/uploads", Guid.NewGuid() + Path.GetExtension(file.FileName));
            using var stream = new FileStream(path, FileMode.Create);
            await file.CopyToAsync(stream);
        }
    }

    return RedirectToPage("Success");
}
```

---

## 🖼 6. 업로드된 이미지 미리보기

```razor
<img src="/uploads/@fileName" width="150" />
```

- 업로드한 이미지는 `wwwroot/uploads`에 저장되므로 직접 접근 가능
- 보안이 필요하다면 **서버를 통해 반환하도록 제한**하는 것도 좋음

---

## 📁 7. 디렉터리 자동 생성

업로드 디렉터리가 없을 경우 자동으로 만들어줍니다.

```csharp
var dir = Path.Combine("wwwroot", "uploads");
if (!Directory.Exists(dir))
    Directory.CreateDirectory(dir);
```

---

## ☁️ 8. 클라우드 업로드 옵션

| 스토리지 | 연동 방법 |
|----------|------------|
| Azure Blob Storage | `Azure.Storage.Blobs` NuGet |
| Amazon S3 | `AWSSDK.S3` |
| Firebase Storage | REST API 또는 SDK |
| Google Cloud Storage | Google.Cloud.Storage.V1 |

> ⚠ 파일 크기가 크거나 서버 스케일이 중요한 경우 클라우드 저장이 유리

---

## 📜 9. 파일 업로드 크기 제한 설정

### ✅ Program.cs (ASP.NET Core 6+)

```csharp
builder.Services.Configure<FormOptions>(options =>
{
    options.MultipartBodyLengthLimit = 10485760; // 10MB
});
```

또는 Razor Page/Controller 단에서:

```csharp
[RequestSizeLimit(10485760)] // 10MB
public IActionResult Upload() { ... }
```

---

## 🔍 10. 실무 팁 요약

| 항목 | 팁 |
|------|-----|
| 파일 이름 | GUID + 확장자 권장 |
| 저장 경로 | `wwwroot/uploads/YYYY/MM/` 형태 추천 |
| 정적 미리보기 | 이미지만 직접 접근 허용 or 컨트롤러 기반 제공 |
| 부하 분산 | CloudFront, Azure CDN 등 정적 파일 CDN 구성 |
| 비동기 처리 | 이미지 리사이징은 Background Queue 추천 |

---

## ✅ 요약

| 주제 | 요약 |
|------|------|
| 파일 수신 | `IFormFile`, `List<IFormFile>` |
| 저장 방식 | 로컬 or 클라우드 |
| 보안 | 크기/확장자 제한, 경로 검증 필수 |
| 이미지 가공 | ImageSharp 사용 추천 |
| 정적 파일 접근 | wwwroot 하위 저장, 혹은 제한된 액세스 제공 |
