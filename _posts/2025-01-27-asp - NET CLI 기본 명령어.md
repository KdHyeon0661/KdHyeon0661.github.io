---
layout: post
title: Asp - .NET CLI 기본 명령어
date: 2025-01-14 19:20:23 +0900
category: asp
---
# ⚙️ .NET CLI 기본 명령어 정리

`.NET CLI(Command Line Interface)`는 Visual Studio 없이도 터미널에서 프로젝트를 생성하고 실행하며, 빌드 및 패키징까지 가능한 도구입니다.

모든 명령어는 `dotnet`으로 시작하며, **크로스 플랫폼**(Windows, Linux, macOS)에서 동일하게 동작합니다.

---

## 🧱 1. 프로젝트 생성

### ✅ `dotnet new`

새로운 프로젝트나 파일을 생성하는 명령어입니다.

```bash
dotnet new [템플릿] -n [프로젝트명]
```

| 템플릿        | 설명                           |
|---------------|--------------------------------|
| `console`     | 콘솔 애플리케이션              |
| `web`         | ASP.NET Core 빈 웹앱 (MVC 아님) |
| `webapp`      | Razor Pages 기반 웹앱          |
| `mvc`         | MVC 웹앱                       |
| `api`         | Web API 프로젝트               |
| `blazorserver`| Blazor Server 앱               |
| `classlib`    | 클래스 라이브러리              |
| `xunit`       | xUnit 테스트 프로젝트          |

**예시**:
```bash
dotnet new webapp -n MyWebApp
```

---

## 🚀 2. 애플리케이션 실행

### ✅ `dotnet run`

프로젝트를 **빌드하고 즉시 실행**합니다. 주로 개발 중 사용됩니다.

```bash
dotnet run
```

- 자동으로 `Program.cs`를 찾아 실행
- 포트는 `launchSettings.json`에 따라 결정됨

**예시**:
```bash
cd MyWebApp
dotnet run
```

→ `http://localhost:5000`, `https://localhost:5001`로 접속 가능

---

## 🔨 3. 빌드

### ✅ `dotnet build`

코드를 컴파일하여 **실행 가능한 결과물(DLL 등)을 생성**합니다. 실행은 하지 않음.

```bash
dotnet build
```

- `bin/Debug/net8.0/` 또는 `Release/` 폴더에 출력

---

## 📦 4. 패키지 복원

### ✅ `dotnet restore`

`*.csproj`에 정의된 **NuGet 패키지를 다운로드**하고, 프로젝트에 필요한 의존성을 설정합니다.

```bash
dotnet restore
```

※ 최신 SDK에서는 `dotnet build`나 `run` 시 자동 복원되므로 명시적 호출은 거의 필요 없음.

---

## 📁 5. 솔루션 관리

### ✅ `dotnet new sln`

솔루션 파일 (`.sln`)을 생성합니다.

```bash
dotnet new sln -n MySolution
```

### ✅ `dotnet sln add`

솔루션에 프로젝트를 추가합니다.

```bash
dotnet sln MySolution.sln add MyWebApp/MyWebApp.csproj
```

---

## 🧪 6. 테스트

### ✅ `dotnet test`

`.csproj`에 정의된 **단위 테스트 프로젝트를 실행**합니다.

```bash
dotnet test
```

- xUnit, NUnit, MSTest 등 지원

---

## 🛠️ 7. 프로젝트/패키지 관리

### ✅ `dotnet clean`

빌드 결과물 (`bin/`, `obj/`)을 정리합니다.

```bash
dotnet clean
```

### ✅ `dotnet add package`

NuGet 패키지를 프로젝트에 추가합니다.

```bash
dotnet add package Microsoft.EntityFrameworkCore
```

---

## 📤 8. 배포용 빌드

### ✅ `dotnet publish`

프로젝트를 실행 가능한 상태로 **패키징 및 배포용으로 출력**합니다.

```bash
dotnet publish -c Release -o ./publish
```

옵션:
- `-c Release` : Release 모드 빌드
- `-o` : 출력 폴더 지정

→ `./publish` 폴더에 DLL, 실행 파일, 설정 등이 포함되어 생성됨

---

# 📝 요약 명령어 표

| 명령어 | 설명 |
|--------|------|
| `dotnet new` | 새 프로젝트 또는 파일 생성 |
| `dotnet run` | 애플리케이션 실행 |
| `dotnet build` | 컴파일하여 실행 파일 생성 |
| `dotnet restore` | 패키지 복원 |
| `dotnet test` | 테스트 실행 |
| `dotnet clean` | 빌드 파일 정리 |
| `dotnet publish` | 배포용 파일 출력 |
| `dotnet sln` | 솔루션 파일 관리 |
| `dotnet add package` | NuGet 패키지 추가 |

---

# ✅ 실전 예시 흐름

```bash
dotnet new webapp -n MySite      # 프로젝트 생성
cd MySite
dotnet run                       # 실행
dotnet build                     # 빌드
dotnet publish -c Release -o ./publish  # 배포 파일 생성
```