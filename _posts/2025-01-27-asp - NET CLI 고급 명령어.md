---
layout: post
title: Asp - .NET CLI 고급 명령어 정리
date: 2025-01-27 19:20:23 +0900
category: asp
---
# 🛠️ .NET CLI 고급 명령어 정리

.NET CLI는 단순한 프로젝트 생성/실행을 넘어서 **템플릿 관리, 도구 설치, 패키지 관리, 트러블슈팅 등 다양한 고급 기능**을 제공합니다.

---

## 🔧 1. 템플릿 관리

### ✅ `dotnet new --list`

설치된 템플릿 목록을 확인할 수 있습니다.

```bash
dotnet new --list
```

### ✅ `dotnet new --install`

사용자 정의 템플릿이나 외부 템플릿을 설치합니다.  
예: GitHub에서 제공되는 프로젝트 템플릿

```bash
dotnet new --install Microsoft.AspNetCore.SpaTemplates::*
dotnet new --install "MyTemplate::1.0.0"
```

### ✅ `dotnet new --uninstall`

설치한 템플릿 제거

```bash
dotnet new --uninstall MyTemplate
```

---

## 🔗 2. 프로젝트 참조 관리

### ✅ `dotnet add reference`

다른 프로젝트를 참조로 추가 (멀티 프로젝트 솔루션에서 사용)

```bash
dotnet add MyApp/MyApp.csproj reference ../MyLibrary/MyLibrary.csproj
```

- `ProjectReference`를 `.csproj`에 추가하는 효과

---

## 🔍 3. 프로젝트 검사 및 분석

### ✅ `dotnet list`

#### 📦 패키지 목록 확인

```bash
dotnet list package
```

- 설치된 NuGet 패키지를 확인
- `--outdated` 옵션으로 최신 버전 확인 가능

```bash
dotnet list package --outdated
```

#### 🔗 참조 확인

```bash
dotnet list reference
```

---

## 🧰 4. 전역 도구(Global Tools)

.NET에서는 CLI 기반의 유틸리티를 전역 도구 형태로 설치할 수 있습니다.

### ✅ `dotnet tool install`

전역 도구 설치

```bash
dotnet tool install -g dotnet-ef
```

- `-g` : 글로벌 설치 (사용자 전역)

### ✅ `dotnet tool update`

전역 도구 업데이트

```bash
dotnet tool update -g dotnet-ef
```

### ✅ `dotnet tool uninstall`

전역 도구 삭제

```bash
dotnet tool uninstall -g dotnet-ef
```

### ✅ `dotnet tool list`

설치된 도구 확인

```bash
dotnet tool list -g
```

---

## 🏗️ 5. 런타임 및 SDK 정보

### ✅ `dotnet --info`

현재 설치된 .NET SDK와 런타임 정보를 확인할 수 있습니다.

```bash
dotnet --info
```

### ✅ `dotnet --list-sdks`

설치된 SDK 목록 확인

```bash
dotnet --list-sdks
```

### ✅ `dotnet --list-runtimes`

설치된 런타임 목록 확인

```bash
dotnet --list-runtimes
```

---

## 🐛 6. 디버깅/트러블슈팅

### ✅ `dotnet --diagnostics`

실행 가능한 진단 툴 리스트 확인 (ex. `dotnet-trace`, `dotnet-dump`)

```bash
dotnet help diagnostics
```

### ✅ `dotnet build --verbosity:diag`

빌드 시 상세 로그 출력

```bash
dotnet build --verbosity:diagnostic
```

옵션:
- `quiet` (최소 출력)
- `minimal`
- `normal`
- `detailed`
- `diagnostic` (가장 상세)

---

## 🧱 7. Runtime 실행 및 dll 실행

### ✅ DLL 직접 실행

`publish` 또는 `build` 결과물 실행

```bash
dotnet ./bin/Debug/net8.0/MyApp.dll
```

→ 단독 실행 애플리케이션이 아닌 경우에 유용

---

## ✨ 8. 기타 유용한 명령어

| 명령어 | 설명 |
|--------|------|
| `dotnet migrate` | 구버전 프로젝트(.NET Framework) 마이그레이션 |
| `dotnet pack` | NuGet 패키지(.nupkg) 생성 |
| `dotnet nuget push` | NuGet 서버에 패키지 배포 |
| `dotnet workload install` | MAUI, WASM 등 추가 워크로드 설치 |
| `dotnet watch` | 코드 변경 시 자동 빌드 및 실행 (개발에 유용) |

```bash
dotnet watch run
```

---

# 📝 정리: 실무에 유용한 고급 명령어 모음

| 목적 | 명령어 예시 |
|------|-------------|
| 템플릿 설치 | `dotnet new --install` |
| 프로젝트 참조 추가 | `dotnet add reference` |
| 패키지 상태 확인 | `dotnet list package --outdated` |
| 전역 도구 관리 | `dotnet tool install -g [도구명]` |
| 런타임 정보 | `dotnet --list-sdks` / `--info` |
| 자동 재시작 | `dotnet watch run` |
| 상세 로그 | `dotnet build --verbosity:diagnostic` |