---
layout: post
title: AspNet - Secret Manager
date: 2025-04-10 21:20:23 +0900
category: AspNet
---
# 🔐 ASP.NET Core 시크릿 매니저(Secret Manager) 사용법 완전 정리

---

## ✅ 1. Secret Manager란?

- **개발 환경**에서 `appsettings.json` 파일에 **민감 정보를 직접 작성하지 않도록** 도와주는 도구
- **로컬 사용자 계정에 암호화된 형태로 저장**되며, Git 등 버전 관리에 노출되지 않음
- 운영 환경에서는 `환경 변수` 또는 `Azure Key Vault` 사용을 권장

> ❗ Git에 올리지 않고도 비밀 설정을 안전하게 개발 중 사용할 수 있게 해줍니다.

---

## 📦 2. 사용 전 준비

- .NET Core SDK가 설치되어 있어야 함
- ASP.NET Core 프로젝트는 `.csproj`에 UserSecrets 설정이 있어야 함

---

## 🧪 3. 프로젝트에 시크릿 매니저 활성화

```bash
dotnet user-secrets init
```

실행 결과:
- `.csproj` 파일에 다음 항목이 자동 추가됨

```xml
<UserSecretsId>your-project-guid</UserSecretsId>
```

---

## 🗝️ 4. 비밀 값 저장하기

```bash
dotnet user-secrets set "JwtSettings:SecretKey" "MySuperSecretKey123"
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=.;Database=AppDb;Trusted_Connection=True;"
```

> `:`을 사용하여 JSON 구조처럼 계층화 가능

---

## 🧾 5. 저장된 비밀 값 확인

```bash
dotnet user-secrets list
```

출력 예:
```bash
JwtSettings:SecretKey = MySuperSecretKey123
ConnectionStrings:DefaultConnection = Server=.;Database=AppDb;Trusted_Connection=True;
```

---

## 🧹 6. 비밀 값 삭제

- 개별 삭제:
```bash
dotnet user-secrets remove "JwtSettings:SecretKey"
```

- 전체 삭제:
```bash
dotnet user-secrets clear
```

---

## 🧑‍💻 7. 코드에서 비밀 값 사용

```csharp
var builder = WebApplication.CreateBuilder(args);

// 자동으로 user-secrets 포함됨 (환경이 Development일 때)
var secret = builder.Configuration["JwtSettings:SecretKey"];
Console.WriteLine($"비밀 키: {secret}");
```

> `appsettings.json`과 동일한 방식으로 `builder.Configuration["..."]`에서 사용 가능

---

## 🧭 8. 저장 위치 (로컬 경로)

- 실제 저장 위치는 OS마다 다릅니다:

| OS | 경로 |
|----|------|
| Windows | `%APPDATA%\Microsoft\UserSecrets\{UserSecretsId}` |
| Linux/macOS | `~/.microsoft/usersecrets/{UserSecretsId}` |

> ⚠️ 개인 사용자 디렉터리에 저장되므로, 다른 사용자에게는 공유되지 않음

---

## 🔐 9. 왜 필요한가?

| 방법 | 보안성 | Git 노출 위험 | 용도 |
|------|--------|----------------|------|
| appsettings.json | ❌ 민감 정보 노출 | 높음 | 설정값 저장 |
| 환경 변수 | ✅ 안전 | 낮음 | 운영 환경 |
| Secret Manager | ✅ 안전 | 없음 | **개발 환경용** |

> Secret Manager는 **운영이 아닌 개발용** 보안 저장소입니다.

---

## ⚠️ 10. 보안 주의사항

- Secret Manager는 **개발용 도구**이며 **운영 환경**에는 절대 사용하지 말 것
- 운영 환경에는 **환경 변수**, **Azure Key Vault**, **AWS Parameter Store** 등을 사용해야 함
- `dotnet user-secrets`는 **CLI 전용**이며 프로젝트별로 저장됨

---

## ✅ 요약

| 항목 | 내용 |
|------|------|
| 사용 목적 | 개발 시 민감 정보 보안 유지 |
| 저장 위치 | 로컬 사용자 프로필 (Git 미포함) |
| 명령어 | `dotnet user-secrets set/get/remove/list` |
| 구성 방식 | `UserSecretsId`로 프로젝트와 연결 |
| 사용 시점 | `Development` 환경일 때만 자동 로드 |