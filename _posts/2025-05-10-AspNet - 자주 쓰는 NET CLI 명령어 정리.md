---
layout: post
title: AspNet - 자주 쓰는 .NET CLI 명령어 정리
date: 2025-05-10 20:20:23 +0900
category: AspNet
---
# 🛠 자주 쓰는 .NET CLI 명령어 정리

ASP.NET Core를 포함한 .NET 프로젝트는 **터미널 기반 CLI 도구**인 `dotnet`을 통해 대부분의 작업이 가능합니다.  
Visual Studio 없이도 프로젝트 생성, 실행, 빌드, 테스트, 배포까지 모두 CLI로 처리할 수 있어요.

---

## ✅ 기본 명령어 요약

| 명령어 | 기능 |
|--------|------|
| `dotnet new` | 새 프로젝트/파일 생성 |
| `dotnet run` | 앱 실행 (빌드 + 실행) |
| `dotnet build` | 코드 빌드 (컴파일) |
| `dotnet publish` | 배포용 코드 출력 |
| `dotnet restore` | NuGet 패키지 복원 |
| `dotnet clean` | 출력 파일 삭제 |
| `dotnet test` | 유닛 테스트 실행 |
| `dotnet watch` | 파일 변경 시 자동 재실행 |
| `dotnet --info` | SDK 및 환경 정보 확인 |

---

## 📦 `dotnet new` - 새 프로젝트/파일 생성

### 기본 구조

```bash
dotnet new [템플릿] [옵션]
```

### 예시

```bash
dotnet new webapp -n MyWebApp
dotnet new mvc -n MyMvcApp
dotnet new razor -n MyRazorApp
dotnet new webapi -n MyApi
```

### 주요 옵션

| 옵션 | 설명 |
|------|------|
| `-n` | 프로젝트 이름 지정 |
| `-o` | 생성 위치 지정 |
| `--auth` | 인증 방식 (`Individual`, `None`) |
| `--framework` | 대상 .NET 버전 (`net8.0` 등) |

### 사용 가능한 템플릿 보기

```bash
dotnet new --list
```

---

## 🚀 `dotnet run` - 앱 실행

- 빌드 후 자동 실행
- ASP.NET Core 앱 실행에 자주 사용

```bash
dotnet run
```

### 특정 프로젝트 지정 실행

```bash
dotnet run --project ./MyApp/MyApp.csproj
```

---

## 🧱 `dotnet build` - 컴파일만 수행

- 실행은 안 하고, 컴파일만 수행
- 주로 CI에서 사전 빌드 검증용

```bash
dotnet build
```

---

## 📤 `dotnet publish` - 배포용 출력

- 실제 서버에 배포할 **실행파일, DLL, 웹 정적 파일**을 출력

```bash
dotnet publish -c Release -o ./publish
```

### 주요 옵션

| 옵션 | 설명 |
|------|------|
| `-c` | 구성 모드 (`Debug`, `Release`) |
| `-o` | 출력 경로 지정 |
| `--self-contained` | 자체 실행 파일 포함 여부 |
| `--runtime` | 배포 대상 OS/플랫폼 (`linux-x64`, `win-x64` 등) |

---

## 🔁 `dotnet restore` - NuGet 복원

- `.csproj`에 정의된 패키지를 다운로드
- 일반적으로 `build`, `run` 시 자동 실행되므로 명시적으로 쓸 일은 적음

```bash
dotnet restore
```

---

## 🧹 `dotnet clean` - 빌드 아웃풋 정리

- `bin/`, `obj/` 폴더 제거
- 깨끗한 상태로 다시 빌드하고 싶을 때 사용

```bash
dotnet clean
```

---

## 🧪 `dotnet test` - 테스트 실행

- `xUnit`, `NUnit`, `MSTest` 프로젝트의 테스트 자동 실행

```bash
dotnet test
```

### 특정 프로젝트에서 테스트

```bash
dotnet test ./MyProject.Tests/MyProject.Tests.csproj
```

---

## 👀 `dotnet watch` - 변경 감지 자동 실행

- 코드 수정 시 자동으로 빌드 및 실행
- 프론트엔드 개발자들이 선호하는 기능

```bash
dotnet watch run
```

- `.cshtml`, `.cs`, `.js` 등 변경 시 자동 반영

---

## 🔎 `dotnet --info` - 환경 정보 출력

- 설치된 SDK, 런타임, OS 정보 출력

```bash
dotnet --info
```

> 문제 해결이나 다중 SDK 환경에서 유용

---

## 📜 기타 유용한 명령

### 사용 가능한 SDK 목록 보기

```bash
dotnet --list-sdks
```

### 사용 가능한 런타임 목록 보기

```bash
dotnet --list-runtimes
```

---

## 🧩 실무 팁

| 목적 | 명령어 |
|------|--------|
| 새 Razor Pages 앱 | `dotnet new razor -n MySite` |
| Web API 템플릿 생성 | `dotnet new webapi -n ApiApp` |
| 배포 빌드 | `dotnet publish -c Release -o dist` |
| 실시간 디버깅 | `dotnet watch run` |
| 테스트 자동화 | `dotnet test --logger:trx` |
| 로컬 인증 포함 앱 | `dotnet new mvc --auth Individual` |

---

## ✅ 요약

| 명령어 | 기능 |
|--------|------|
| `dotnet new` | 새 프로젝트/템플릿 생성 |
| `dotnet run` | 앱 실행 |
| `dotnet build` | 컴파일 |
| `dotnet publish` | 배포용 패키징 |
| `dotnet test` | 테스트 자동 실행 |
| `dotnet watch` | 실시간 변경 감지 실행 |
| `dotnet clean` | 정리 |
| `dotnet restore` | NuGet 복원 |