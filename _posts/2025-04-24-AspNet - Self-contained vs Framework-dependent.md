---
layout: post
title: AspNet - Self-contained vs Framework-dependent
date: 2025-04-24 20:20:23 +0900
category: AspNet
---
# 🧳 Self-contained vs Framework-dependent 배포 자세히 알아보기

---

## ✅ 1. 개요

ASP.NET Core 앱을 배포할 때는 **런타임 포함 여부**에 따라 2가지 방식 중 하나를 선택할 수 있어요.

| 배포 방식 | 설명 |
|-----------|------|
| **Framework-dependent** | 서버에 .NET Runtime이 설치되어 있어야 함 |
| **Self-contained** | 앱 실행에 필요한 .NET 런타임도 함께 포함되어 배포됨 |

---

## 🧩 2. 비교 요약

| 항목 | Framework-dependent | Self-contained |
|------|---------------------|----------------|
| .NET 설치 필요 | 서버에 있어야 함 | 필요 없음 |
| 파일 크기 | 작음 (수 MB) | 큼 (수백 MB) |
| 실행 파일 | `dotnet MyApp.dll` | `MyApp.exe` or `./MyApp` |
| OS/CPU 종속성 | 낮음 | 높음 (플랫폼별 빌드 필요) |
| 배포 편의성 | 빠름 | 이식성 높음 |
| 업데이트 방식 | 런타임 업데이트로 관리 | 재배포 필요 |

---

## 🚀 3. 빌드 예제

### ▶ Framework-dependent

```bash
dotnet publish -c Release -o ./publish
```

또는 명시적 옵션:

```bash
dotnet publish -c Release -r win-x64 --self-contained false
```

- 결과물: `.dll` 파일 존재, 실행 시 `dotnet` 명령 필요

```bash
dotnet MyApp.dll
```

---

### ▶ Self-contained

```bash
dotnet publish -c Release -r win-x64 --self-contained true
```

- 결과물: `.exe` or 리눅스의 경우 실행 파일(`MyApp`)
- 모든 실행 파일, 런타임 포함

```bash
./MyApp
```

---

## 🧠 4. 실무에서의 사용 기준

| 상황 | 권장 방식 |
|------|-----------|
| 클라우드, 컨테이너 환경 | Framework-dependent (런타임 미리 설치) |
| 설치형 제품, 데스크탑 배포 | Self-contained (런타임 포함) |
| 사내 Windows 서버 배포 | Framework-dependent + .NET Hosting Bundle |
| Kiosk, POS, 임베디드 장비 | Self-contained (자급자족 실행) |

---

## 🧮 5. 크기 차이 예시

| 타입 | 파일 수 | 대략적 크기 |
|------|---------|-------------|
| Framework-dependent | 10~20개 | 20~40MB |
| Self-contained (win-x64) | 150+개 | 100~250MB 이상 |

> 특히 **Windows GUI 앱**이나 **Blazor WebAssembly 앱**의 경우  
> Self-contained 크기가 **400MB 이상** 될 수 있음

---

## 🛠️ 6. 런타임 식별자(RID)

Self-contained 배포 시에는 **타겟 플랫폼**을 명시해야 해요.

| 플랫폼 | RID |
|--------|-----|
| Windows 64bit | `win-x64` |
| Linux 64bit | `linux-x64` |
| macOS (Intel) | `osx-x64` |
| macOS (ARM) | `osx-arm64` |
| Raspberry Pi | `linux-arm` |

```bash
dotnet publish -c Release -r linux-x64 --self-contained true
```

---

## 🔒 7. 보안 및 유지보수 관점

| 관점 | Framework-dependent | Self-contained |
|------|---------------------|----------------|
| 보안 패치 적용 | 서버의 런타임만 업데이트하면 됨 | 앱 재배포 필요 |
| 취약점 대응 속도 | 빠름 | 느림 (직접 재배포해야 함) |
| 관리 효율 | 중앙 집중적 | 앱별 따로 관리 필요 |

---

## 📦 8. .csproj에 기본 설정 추가 예시

```xml
<PropertyGroup>
  <RuntimeIdentifier>win-x64</RuntimeIdentifier>
  <SelfContained>true</SelfContained>
</PropertyGroup>
```

이렇게 설정하면 `dotnet publish` 할 때 자동으로 해당 방식으로 배포됨.

---

## ✅ 9. 요약

| 항목 | Framework-dependent | Self-contained |
|------|---------------------|----------------|
| 런타임 포함 여부 | ❌ | ✅ |
| 파일 크기 | 작음 | 큼 |
| 실행 방식 | `dotnet app.dll` | `./app` 또는 `app.exe` |
| 환경 독립성 | 낮음 | 높음 |
| 보안 패치 유지 | 자동 | 수동 (재배포) |
| 실무 활용 | 서버/클라우드 | 독립 실행 환경, 배포 설치용 |