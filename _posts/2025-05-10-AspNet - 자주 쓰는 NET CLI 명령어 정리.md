---
layout: post
title: AspNet - 자주 쓰는 .NET CLI 명령어 정리
date: 2025-05-10 20:20:23 +0900
category: AspNet
---
# 자주 쓰는 .NET CLI 명령어 정리

## 0) CLI를 쓰는 이유 — “IDE 없이 끝까지 간다”

- 로컬 개발: `dotnet new/run/watch/test`
- 패키지/참조 관리: `dotnet add package/reference`
- 솔루션/프로젝트 오케스트레이션: `dotnet sln`, `dotnet restore/build`
- 배포 산출: `dotnet publish`(Self-contained/Single-file/Trim/AOT/ReadyToRun)
- 자동화/CI: 파이프라인에서 스크립트 그대로 재사용
- 개발 유틸: `user-secrets`, `dev-certs`, `ef`, `format`, `tool`, `workload`

---

## 1) 핵심 명령어 — 상시 쓰는 핫키 모음

| 명령 | 핵심 역할 | 실전 옵션/메모 |
|---|---|---|
| `dotnet new` | 템플릿으로 새 프로젝트/파일 생성 | `--list`, `--install`, `--update-apply`, `-n`, `-o`, `--auth`, `--framework` |
| `dotnet run` | 빌드 후 실행 | `--project`, `--launch-profile`, `--no-build` |
| `dotnet build` | 컴파일만 | `-c Release`, `-p:WarnAsError=true`, `--no-restore` |
| `dotnet publish` | 배포 산출물 출력 | `-c Release -o ./publish -r linux-x64 --self-contained true --p:PublishSingleFile=true --p:PublishTrimmed=true` |
| `dotnet restore` | NuGet 복원 | `--no-cache`, `--force-evaluate`, `--interactive` |
| `dotnet clean` | bin/obj 삭제 | CI 전후, 빌드 캐시 초기화 |
| `dotnet test` | 테스트 실행 | `--filter`, `--logger trx`, `--collect:"XPlat Code Coverage"` |
| `dotnet watch` | 변경 감지 자동 실행 | `watch run`, `watch test`, Hot Reload |
| `dotnet --info` | SDK/런타임/OS 정보 | 다중 SDK 충돌 진단 |
| `dotnet sln` | 솔루션 관리 | `add/remove/list` |
| `dotnet add` | 참조/패키지 추가 | `dotnet add <csproj> package`, `dotnet add reference` |
| `dotnet pack` | NuGet 패키지 만들기 | `-c Release -o ./nupkgs` |
| `dotnet nuget` | 소스/푸시 | `nuget add source`, `nuget push` |
| `dotnet tool` | 전역/로컬 도구 | `tool install -g`, `tool restore` |
| `dotnet workload` | 워크로드 | `install wasm-tools maui android ios` |
| `dotnet format` | 코드 포맷/분석 | 스타일/애널라이저 자동 고침 |

---

## 2) `dotnet new` — 템플릿 파헤치기

### 2.1 자주 쓰는 생성

```bash
dotnet new webapp -n MyWebApp              # Razor Pages
dotnet new mvc -n MyMvcApp                  # MVC
dotnet new webapi -n MyApi                  # API
dotnet new classlib -n CoreLib              # Class Library
dotnet new xunit -n MyApp.Tests             # xUnit 테스트
```

### 2.2 옵션 패턴

```bash
dotnet new mvc -n Shop.Web \
  --framework net8.0 \
  --auth Individual \
  -o src/Shop.Web
```

- `--auth Individual`: ASP.NET Identity 포함
- `--framework`: `net8.0`, `net9.0` 등
- `--list`: 설치된 템플릿 확인
- 커스텀 템플릿 설치/업데이트:
  ```bash
  dotnet new --install My.Templates::2.0.0
  dotnet new --update-apply
  ```

---

## 3) 솔루션/참조/패키지 — 멀티 프로젝트 구조 빠르게

```bash
dotnet new sln -n Shop
dotnet new webapi -n Shop.Api -o src/Shop.Api
dotnet new classlib -n Shop.Core -o src/Shop.Core

dotnet sln Shop.sln add src/Shop.Api/Shop.Api.csproj
dotnet sln Shop.sln add src/Shop.Core/Shop.Core.csproj

dotnet add src/Shop.Api/Shop.Api.csproj reference src/Shop.Core/Shop.Core.csproj
dotnet add src/Shop.Api/Shop.Api.csproj package FluentValidation.AspNetCore
dotnet remove src/Shop.Api/Shop.Api.csproj package Bogus
```

- 빠른 구조화: `src/`, `tests/` 폴더로 구분해 CI 가독성↑

---

## 4) 실행/빌드 — 속도·재현성·진단 옵션

```bash
dotnet restore --force-evaluate --locked-mode   # lock파일 기반 재현 복원
dotnet build -c Release --no-restore \
  -p:Deterministic=true -p:ContinuousIntegrationBuild=true
dotnet run --project src/Shop.Api/Shop.Api.csproj
```

- `--no-restore`: 복원 단계를 분리해 캐시 활용
- CI에서 `-p:ContinuousIntegrationBuild=true`로 **소스 링크/심볼** 품질 개선

---

## 5) 테스트 — 필터/로그/커버리지

```bash
dotnet test tests/Shop.Api.Tests/Shop.Api.Tests.csproj \
  --filter "Category=Unit&FullyQualifiedName~Order" \
  --logger "trx;LogFileName=test.trx" \
  --collect:"XPlat Code Coverage" \
  -c Release --no-build
```

- **필터 예시**
  - `--filter TestCategory=Integration`
  - `--filter Name~ShouldReturn400`
- 커버리지 리포트(coverlet.collector 필요) + 변환(ReportGenerator 도구 조합)
- 느린 테스트 추적: `--blame-hang --blame-hang-timeout 5m`

**watch 테스트**

```bash
dotnet watch test --project tests/Shop.Api.Tests
```

---

## 6) `dotnet watch` — Hot Reload & 프론트감성 반복

```bash
dotnet watch run --project src/Shop.Web
```

- 변경 즉시 재시작, Razor/JS/CSS도 감지
- 환경 변수로 URL/포트 고정:
  - PowerShell: `$env:ASPNETCORE_URLS="http://localhost:5005"; dotnet watch run`
  - Bash: `ASPNETCORE_URLS=http://0.0.0.0:5005 dotnet watch run`

---

## 7) 배포: `dotnet publish` — 모드/플랫폼/최적화 매트릭스

### 7.1 기본형

```bash
dotnet publish -c Release -o ./publish
```

### 7.2 플랫폼/런타임 식별자(RID)

```bash
# 프레임워크 의존(서버에 .NET 런타임 필요)
dotnet publish -c Release -r linux-x64 --self-contained false -o ./out/linux

# Self-contained(런타임 포함)
dotnet publish -c Release -r win-x64 --self-contained true -o ./out/win
```

| 대표 RID | 설명 |
|---|---|
| `win-x64`, `win-arm64` | Windows |
| `linux-x64`, `linux-arm64` | Linux |
| `osx-x64`, `osx-arm64` | macOS |

### 7.3 단일 파일/트리밍/네이티브 프리뷰

```bash
dotnet publish -c Release -r linux-x64 --self-contained true \
  -p:PublishSingleFile=true \
  -p:PublishTrimmed=true \
  -p:TrimMode=partial \
  -o ./publish/linux

# 빠른 시작(ReadyToRun)
dotnet publish -c Release -r win-x64 \
  -p:PublishReadyToRun=true -o ./publish/win
```

> **주의**: Trim은 리플렉션/다이내믹 로드 코드에서 누락 위험.  
> 문제 영역에 `DynamicDependency`/`TrimmerRootDescriptor`/`ILLink` 설정으로 보정.

### 7.4 AOT(네이티브) 시나리오 스케치

```bash
dotnet publish -c Release -r linux-x64 \
  -p:PublishAot=true -o ./publish/aot
```

- 서버 사이드에서는 적용 범위를 신중히(빌드시간↑, 디버깅 제약)

### 7.5 MSBuild 속성 한번에

```bash
dotnet publish -c Release -r linux-x64 \
  -p:SelfContained=false \
  -p:PublishSingleFile=true \
  -p:PublishTrimmed=true \
  -p:GenerateDocumentationFile=true \
  -o ./publish
```

---

## 8) NuGet — 패키징/푸시/소스

```bash
# 패키지 만들기
dotnet pack src/Shop.Core/Shop.Core.csproj -c Release -o ./nupkgs

# 소스 추가(Private feed)
dotnet nuget add source "https://nuget.example.com/v3/index.json" -n PrivateFeed \
  --username USER --password TOKEN --store-password-in-clear-text

# 패키지 푸시
dotnet nuget push ./nupkgs/Shop.Core.1.2.3.nupkg \
  --api-key $NUGET_API_KEY --source "https://api.nuget.org/v3/index.json"
```

---

## 9) 도구/워크로드 — 개발 경험 확장

### 9.1 전역/로컬 도구

```bash
dotnet tool install -g dotnet-ef
dotnet tool update -g dotnet-ef

# 로컬(매니페스트)
dotnet new tool-manifest
dotnet tool install dotnet-ef
dotnet tool restore
```

### 9.2 워크로드

```bash
dotnet workload install wasm-tools
dotnet workload list
```

- Blazor WASM 최적화/MAUI 등 대상별 SDK를 on-demand로 설치

---

## 10) EF Core CLI(도구 조합 예시)

```bash
# 디자인 패키지(프로젝트에)
dotnet add src/Shop.Api/Shop.Api.csproj package Microsoft.EntityFrameworkCore.Design
dotnet add src/Shop.Api/Shop.Api.csproj package Npgsql.EntityFrameworkCore.PostgreSQL

# 마이그레이션/업데이트
dotnet ef migrations add Init --project src/Shop.Api --startup-project src/Shop.Api
dotnet ef database update --project src/Shop.Api --startup-project src/Shop.Api
```

- 컨텍스트/스타트업 분리 시 `--project`/`--startup-project`로 경로 지정

---

## 11) 개발 편의 — 시크릿/개발인증서/포맷/애널라이저

```bash
# User Secrets
dotnet user-secrets init --project src/Shop.Api
dotnet user-secrets set "Jwt:Key" "LOCAL-SECRET" --project src/Shop.Api

# 개발 HTTPS 인증서
dotnet dev-certs https --trust

# 코드 포맷/분석
dotnet format
dotnet format analyzers
```

---

## 12) 환경 고정 — `global.json`와 SDK 매칭

```json
{
  "sdk": {
    "version": "8.0.401",
    "rollForward": "disable"
  }
}
```

- 팀/CI에서 SDK 불일치로 인한 “빌드만 다르게”를 차단

---

## 13) 고급 빌드/구성 — 공통 속성/경고·품질 강제

### 13.1 `Directory.Build.props` (솔루션 루트)

```xml
<Project>
  <PropertyGroup>
    <LangVersion>latest</LangVersion>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <Nullable>enable</Nullable>
    <Deterministic>true</Deterministic>
    <ContinuousIntegrationBuild>true</ContinuousIntegrationBuild>
    <DisableImplicitNamespaceImports>false</DisableImplicitNamespaceImports>
  </PropertyGroup>
</Project>
```

### 13.2 `Directory.Build.targets` (패키징/아티팩트 규약 가능)

```xml
<Project>
  <Target Name="AfterBuild">
    <Message Text="Built $(MSBuildProjectName) => $(TargetDir)" Importance="high" />
  </Target>
</Project>
```

---

## 14) 성능/용량 기본기 — 빠른 스타트, 작은 배포

- **ReadyToRun**: 초기 JIT 부담 완화(파일↑)
- **SingleFile**: 파일/배포 간소화
- **Trim**: 크기↓(리플렉션 영역 예외 처리 필수)
- **압축**: `-p:EnableCompressionInSingleFile=true`
- **리소스 제외**: `-p:IncludeAllContentForSelfExtract=false`
- **R2R+SingleFile** 병행 시 테스트로 체감 확인(서버 스펙/워크로드에 따라 편차)

---

## 15) 트러블슈팅 TOP 10

1. **패키지 복원 실패**: `dotnet nuget locals all --clear`, 프록시/인증 확인, `--interactive`
2. **SDK 충돌**: `dotnet --list-sdks`, `global.json` 고정
3. **런타임 불일치**: 프레임워크 의존 배포에서 서버 런타임 버전 확인
4. **Trim 후 런타임 예외**: 예외 스택에 리플렉션/동적로딩 흔적 → 보존 특성 추가
5. **Single-file 성능 저하**: 압축 off/리소스 분리/ReadyToRun 조합 실측
6. **테스트가 CI에서만 실패**: `--blame-hang`, 병렬 off(`-p:ParallelizeTestCollections=false`)
7. **HTTPS 개발 인증서 경고**: `dotnet dev-certs https --trust` 다시 실행
8. **Watch 미감지**: 파일시스템 워처 한계(도커 볼륨/네트워크FS) → 폴링 옵션 고려
9. **EF 명령 컨텍스트 못 찾음**: `--startup-project` 정확히 지정
10. **리눅스 실행 권한**: `chmod +x ./publish/MyApp`(Self-contained/SingleFile)

---

## 16) 실전 레시피 — “3줄로 끝내는” 상황별 스니펫

### 16.1 API 템플릿+패키지+실행
```bash
dotnet new webapi -n Api && cd Api
dotnet add package Swashbuckle.AspNetCore
dotnet run
```

### 16.2 멀티 OS 배포 산출
```bash
dotnet publish -c Release -r linux-x64 --self-contained true -o out/linux
dotnet publish -c Release -r win-x64   --self-contained true -o out/win
```

### 16.3 최소 배포(단일파일+트림)
```bash
dotnet publish -c Release -r linux-x64 --self-contained true \
  -p:PublishSingleFile=true -p:PublishTrimmed=true -o out/min
```

### 16.4 테스트 + 커버리지 + 리포트
```bash
dotnet test --collect:"XPlat Code Coverage" --logger:"trx;LogFileName=test.trx"
# (ReportGenerator 도구로 변환)
```

### 16.5 코드 일괄 포맷
```bash
dotnet tool install -g dotnet-format
dotnet format
```

---

## 17) 빠른 치트시트

```bash
# 템플릿
dotnet new --list
dotnet new mvc -n Web --framework net8.0 --auth Individual

# 솔루션/참조/패키지
dotnet new sln -n App
dotnet sln App.sln add src/Web/Web.csproj
dotnet add src/Web/Web.csproj package Serilog.AspNetCore
dotnet add src/Web/Web.csproj reference src/Core/Core.csproj

# 빌드/실행/감시
dotnet restore
dotnet build -c Release --no-restore
dotnet run --project src/Web/Web.csproj
dotnet watch run --project src/Web/Web.csproj

# 테스트
dotnet test --filter "Category=Unit" --logger trx

# 배포
dotnet publish -c Release -o ./publish
dotnet publish -c Release -r linux-x64 --self-contained true \
  -p:PublishSingleFile=true -p:PublishTrimmed=true -o ./publish/linux

# 도구/시크릿/인증서
dotnet tool install -g dotnet-ef
dotnet user-secrets init --project src/Web
dotnet user-secrets set "Jwt:Key" "LOCAL" --project src/Web
dotnet dev-certs https --trust
```

---

## 18) 결론

- **CLI는 표준화된 자동화의 언어**다. 한 번 정립한 스크립트는 로컬·CI·서버 어디서나 동일하게 동작한다.
- **빌드/배포 최적화(Trim/SingleFile/ReadyToRun/AOT)**는 효과와 리스크를 함께 가진다.  
  실측과 단계적 적용으로 **안전한 성능 향상**을 가져가라.
- 템플릿/도구/워크로드/시크릿/인증서 명령까지 익히면, **IDE 없이도 풀스택 사이클을 닫을 수 있다**.
