---
layout: post
title: AspNet - .NET CLI 기본 명령어 정리
date: 2025-01-27 19:20:23 +0900
category: AspNet
---
# .NET CLI 기본 명령어 정리

## 0. 시작 전에 — 환경 점검과 버전 고정

### 0.1 SDK/런타임 점검
```bash
dotnet --info           # 설치된 SDK/런타임, OS/RID, 경로 확인
dotnet --list-sdks      # 설치된 SDK 버전 목록
dotnet --list-runtimes  # 설치된 런타임 목록
```

### 0.2 팀/CI용 SDK 버전 고정(global.json)
```bash
# 루트에서 원하는 SDK 버전 지정
dotnet new globaljson --sdk-version 8.0.403 --force
```
- 여러 SDK가 설치된 환경에서 **일관된 빌드**를 보장합니다.
- CI(예: GitHub Actions)에서도 동일 버전으로 정렬됩니다.

---

## 1. 프로젝트 생성 — `dotnet new`

### 1.1 기본 문법
```bash
dotnet new [템플릿] -n [프로젝트명] [-o 출력폴더] [옵션...]
```

자주 쓰는 템플릿과 용도:

| 템플릿        | 설명                                 |
|---------------|--------------------------------------|
| `console`     | 콘솔 앱                              |
| `web`         | 빈 웹앱(호스트/미들웨어만)          |
| `webapp`      | Razor Pages 웹앱                     |
| `mvc`         | MVC 웹앱                             |
| `api`         | Web API                              |
| `blazorserver`| Blazor Server                        |
| `classlib`    | 클래스 라이브러리                    |
| `xunit`       | xUnit 테스트                         |
| `sln`         | 솔루션 파일 생성                     |

템플릿 찾기/설명 보기:
```bash
dotnet new list                      # 설치된 템플릿 나열
dotnet new webapp --help             # 템플릿별 옵션 도움말
```

### 1.2 예시
```bash
dotnet new webapp -n MyWebApp
dotnet new api     -n MyApi
dotnet new classlib -n MyLib
```

### 1.3 고급 옵션
```bash
dotnet new console -n ToolingDemo --framework net8.0
dotnet new api -n MyApi --no-https
dotnet new webapp -n MyWebApp --auth Individual   # Identity 포함(로컬 DB)
```

---

## 2. 애플리케이션 실행 — `dotnet run`

### 2.1 기본
```bash
cd MyWebApp
dotnet run
```
- `launchSettings.json` 설정이 있으면 해당 URL/포트로 기동합니다.
- 기본적으로 개발 환경에서 `http://localhost:5000`, `https://localhost:5001`가 사용될 수 있습니다.

### 2.2 구성/포트/환경 지정
```bash
dotnet run --configuration Debug
ASPNETCORE_URLS=http://localhost:8080 dotnet run
ASPNETCORE_ENVIRONMENT=Development dotnet run
```

### 2.3 변경 감지 실행(개발 편의)
```bash
dotnet watch run
```
- 파일 변경 시 자동 빌드/재시작합니다.

---

## 3. 빌드 — `dotnet build`

### 3.1 기본
```bash
dotnet build
```
- 산출물: `bin/Debug/net8.0/` 또는 `bin/Release/net8.0/`

### 3.2 구성/경고/병렬/정확성 옵션
```bash
dotnet build -c Release
dotnet build -warnaserror
dotnet build -maxcpucount
```

### 3.3 멀티타기팅 프로젝트
`.csproj`에서
```xml
<TargetFrameworks>net8.0;net7.0</TargetFrameworks>
```
빌드:
```bash
dotnet build -c Release
```

---

## 4. 패키지 복원 — `dotnet restore`

### 4.1 기본
```bash
dotnet restore
```
- 최신 SDK에서는 `build`/`run` 시 자동 복원되므로 명시 호출은 생략 가능.

### 4.2 소스/잠금/구성
```bash
dotnet restore --source https://api.nuget.org/v3/index.json
dotnet restore --locked-mode     # packages.lock.json 기반 고정 복원
```
NuGet 소스 구성(`NuGet.Config`)을 저장소에 포함하면, 팀/CI 환경이 일관됩니다.

---

## 5. 솔루션/프로젝트 관리 — `dotnet sln`, `dotnet new sln`, `dotnet add reference`

### 5.1 솔루션 생성·추가
```bash
dotnet new sln -n MySolution
dotnet sln MySolution.sln add MyWebApp/MyWebApp.csproj
dotnet sln MySolution.sln add MyApi/MyApi.csproj
dotnet sln MySolution.sln add MyLib/MyLib.csproj
```

### 5.2 프로젝트 참조
```bash
# WebApp에서 Lib 참조 추가
dotnet add MyWebApp/MyWebApp.csproj reference MyLib/MyLib.csproj
```

### 5.3 제거
```bash
dotnet sln MySolution.sln remove MyApi/MyApi.csproj
```

---

## 6. 테스트 — `dotnet test`

### 6.1 기본
```bash
dotnet new xunit -n MyLib.Tests
dotnet test
```

### 6.2 선택 실행/필터/로그
```bash
dotnet test --filter "Category=Fast"
dotnet test --logger "trx;LogFileName=test.trx"
```

### 6.3 변경 감지 테스트
```bash
dotnet watch test
```

---

## 7. 정리/패키지/의존성 — `dotnet clean`, `dotnet add package`, `dotnet remove package`, `dotnet list package`

### 7.1 빌드 결과물 정리
```bash
dotnet clean
```

### 7.2 NuGet 패키지 추가/제거/목록
```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet remove package Microsoft.EntityFrameworkCore.Sqlite
dotnet list package
dotnet list package --outdated
```

### 7.3 사설 피드나 사내 캐시 활용
`NuGet.Config`에 사설 레지스트리 추가 후:
```bash
dotnet restore --source "PrivateFeed" --source "nuget.org"
```

---

## 8. 배포 — `dotnet publish`

### 8.1 기본
```bash
dotnet publish -c Release -o ./publish
```
- 산출물: 애플리케이션 DLL, 런타임종속 파일 등

### 8.2 Self-contained/Single-file/Trimming/AOT/ReadyToRun
운영 체제별 **RID(Runtime Identifier)** 지정:
```bash
# Linux x64 Self-contained
dotnet publish -c Release -r linux-x64 --self-contained true -o publish/linux-x64
```

단일 파일:
```bash
dotnet publish -c Release -r win-x64 -p:PublishSingleFile=true -o publish/win-x64
```

트리밍(링커 최적화):
```bash
dotnet publish -c Release -r linux-x64 -p:PublishTrimmed=true -o publish/trim-linux-x64
```

ReadyToRun(빠른 시작 시간):
```bash
dotnet publish -c Release -p:PublishReadyToRun=true -o publish/r2r
```

Native AOT(선택 가능한 프로젝트 유형):
```bash
dotnet publish -c Release -r win-x64 -p:PublishAot=true -o publish/aot
```

> 주의: Trimming/AOT 사용 시 리플렉션 사용 코드가 제거될 수 있으므로 **동적 액세스 주석**이나 **트리밍 경고**를 해결해야 합니다.

---

## 9. 실전 예시 시나리오

### 9.1 최소 웹앱 생성부터 배포까지
```bash
dotnet new webapp -n MySite
cd MySite

# 개발 실행
dotnet run

# 테스트(필요 시)
dotnet new xunit -n MySite.Tests
dotnet test

# 배포용
dotnet publish -c Release -o ./publish
```

### 9.2 단일 솔루션 + Web + API + Lib + Test
```bash
mkdir Monorepo && cd Monorepo
dotnet new sln -n Monorepo

dotnet new webapp -n Web
dotnet new api -n Api
dotnet new classlib -n Domain
dotnet new xunit -n Domain.Tests

dotnet sln Monorepo.sln add Web/Web.csproj Api/Api.csproj Domain/Domain.csproj Domain.Tests/Domain.Tests.csproj
dotnet add Web/Web.csproj reference Domain/Domain.csproj
dotnet add Api/Api.csproj reference Domain/Domain.csproj

dotnet build -c Release
dotnet test
```

### 9.3 다중 대상 프레임워크/교차 배포
`.csproj` 설정:
```xml
<PropertyGroup>
  <TargetFrameworks>net8.0;net7.0</TargetFrameworks>
</PropertyGroup>
```
배포:
```bash
dotnet publish -c Release -r linux-x64 --self-contained true -o out/linux
dotnet publish -c Release -r win-x64   --self-contained true -o out/win
```

---

## 10. 구성/비밀/환경 — appsettings, User Secrets, 환경 변수

### 10.1 환경별 설정
```
appsettings.json
appsettings.Development.json
appsettings.Production.json
```

### 10.2 개발 비밀(User Secrets)
```bash
cd MyWebApp
dotnet user-secrets init
dotnet user-secrets set "Jwt:Key" "local-dev-secret"
```

### 10.3 환경 변수 주입
```bash
ASPNETCORE_ENVIRONMENT=Production dotnet run
ConnectionStrings__Default="Server=...;" dotnet run
```

---

## 11. 도구(툴) — `dotnet tool`

### 11.1 전역/로컬 설치
```bash
dotnet tool install -g dotnet-ef
dotnet tool install --local dotnet-ef
```

### 11.2 사용
```bash
dotnet ef --help
dotnet ef migrations add Init
dotnet ef database update
```

### 11.3 프로젝트 내부 `.config/dotnet-tools.json`
- 로컬 도구를 **프로젝트와 함께 버전 고정**하여 팀/CI 일관성 확보.

---

## 12. 빌드/테스트/배포 자동화 — CI 기본 스케치

### 12.1 GitHub Actions 예
```yaml
name: ci
on: [push, pull_request]
jobs:
  build-test-publish:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '8.0.x'
    - run: dotnet restore
    - run: dotnet build -c Release --no-restore
    - run: dotnet test --no-build --logger "trx;LogFileName=test.trx"
    - run: dotnet publish -c Release -o out
    - uses: actions/upload-artifact@v4
      with:
        name: drop
        path: out
```

---

## 13. 디버깅/성능/분석 — watch, 로그, 트리밍 경고

### 13.1 변경 감지
```bash
dotnet watch run
dotnet watch test
```

### 13.2 빌드 경고와 실패 처리
```bash
dotnet build -warnaserror
```

### 13.3 트리밍 경고 점검
```bash
dotnet publish -c Release -p:PublishTrimmed=true -p:TrimmerDefaultAction=link
```
- 경고를 하나씩 해결하여 배포 크기/시작 시간을 최적화합니다.

---

## 14. 흔한 함정과 체크리스트

1. SDK 불일치
   - CI/로컬 간 SDK 버전 차이로 빌드 실패
   - 해결: `global.json`으로 SDK 고정
2. Self-contained/Single-file로 배포 시 파일 접근
   - 임베디드 리소스/리플렉션 문제
   - 해결: 필요 파일은 외부 출력, 리플렉션 대상은 트리밍 예외 처리
3. 다중 프로젝트 참조 누락
   - 슬루션에만 추가하고 프로젝트 참조를 빼먹는 실수
   - 해결: `dotnet add <A>.csproj reference <B>.csproj`
4. 테스트 필터
   - CI에서 특정 범주만 실행하려다 테스트 누락
   - 해결: `--filter` 사용 시 문서화/리뷰 프로세스
5. 환경 변수/시크릿 관리
   - 로컬 비밀키를 코드/레포에 커밋
   - 해결: User Secrets, CI 시크릿, Key Vault 등 사용

---

## 15. 확장 요약표 — 자주 쓰는 명령과 옵션

| 작업 | 명령 |
|------|------|
| 템플릿 나열 | `dotnet new list` |
| 프로젝트 생성 | `dotnet new webapp -n MyWebApp` |
| 실행 | `dotnet run` / `dotnet watch run` |
| 빌드 | `dotnet build -c Release` |
| 복원 | `dotnet restore --locked-mode` |
| 테스트 | `dotnet test --filter "Category=Fast"` |
| 정리 | `dotnet clean` |
| 패키지 추가/제거 | `dotnet add package ...` / `dotnet remove package ...` |
| 프로젝트 참조 | `dotnet add A.csproj reference B.csproj` |
| 솔루션 | `dotnet new sln -n MySolution` / `dotnet sln add` |
| 배포 | `dotnet publish -c Release -o ./publish` |
| Self-contained | `dotnet publish -r linux-x64 --self-contained true` |
| Single-file | `dotnet publish -p:PublishSingleFile=true` |
| 트리밍 | `dotnet publish -p:PublishTrimmed=true` |
| ReadyToRun | `dotnet publish -p:PublishReadyToRun=true` |
| Native AOT | `dotnet publish -p:PublishAot=true` |
| EF 도구 | `dotnet tool install --local dotnet-ef` |

---

## 16. 실전 흐름 정리(확장판)

```bash
# 1. 솔루션/프로젝트 생성
dotnet new sln -n MySolution
dotnet new webapp -n MySite
dotnet new classlib -n MySite.Domain
dotnet new xunit -n MySite.Tests

# 2. 솔루션/참조 구성
dotnet sln MySolution.sln add MySite/MySite.csproj MySite.Domain/MySite.Domain.csproj MySite.Tests/MySite.Tests.csproj
dotnet add MySite/MySite.csproj reference MySite.Domain/MySite.Domain.csproj
dotnet add MySite.Tests/MySite.Tests.csproj reference MySite.Domain/MySite.Domain.csproj

# 3. 의존성 추가
dotnet add MySite/MySite.csproj package Microsoft.EntityFrameworkCore.Sqlite
dotnet add MySite/MySite.csproj package Swashbuckle.AspNetCore

# 4. 개발 실행/테스트
dotnet watch run --project MySite/MySite.csproj
dotnet test

# 5. 릴리스 빌드/배포
dotnet build -c Release
dotnet publish MySite/MySite.csproj -c Release -o ./publish

# 6. 운영 타입별 빌드(예: Linux x64)
dotnet publish MySite/MySite.csproj -c Release -r linux-x64 --self-contained true -o ./publish/linux
```

---

## 17. 결론

- `.NET CLI`는 **편집기 독립**적으로 프로젝트 생성부터 빌드/테스트/배포까지를 **크로스 플랫폼**으로 일관되게 수행할 수 있게 합니다.
- 본 글은 기존 요약에 **옵션·시나리오·자동화·배포 전략**을 더해 **현업에서 바로 쓰는 레벨**로 확장했습니다.
- 팀/CI 환경에서는 **SDK 버전 고정(global.json)**, **NuGet 복원 일관성**, **Self-contained/Single-file/Trimming 옵션 영향**을 특히 주의하십시오.
- 작은 흐름은 `dotnet new → run/watch → test → publish`로, 복합 솔루션은 `sln/add/reference → build/test → publish`로 정형화하면 운영이 쉬워집니다.
