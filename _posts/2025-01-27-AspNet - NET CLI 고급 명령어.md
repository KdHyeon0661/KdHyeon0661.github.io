---
layout: post
title: AspNet - .NET CLI 고급 명령어 정리
date: 2025-01-27 19:20:23 +0900
category: AspNet
---
# .NET CLI 고급 명령어 정리

## 0. 준비: 버전 고정·환경 점검·개발 인증서

### 0.1 SDK/런타임 버전 확인
```bash
dotnet --info
dotnet --list-sdks
dotnet --list-runtimes
```

### 0.2 팀/CI를 위한 SDK 고정(global.json)
```bash
# 루트에 SDK 버전 고정
dotnet new globaljson --sdk-version 8.0.403 --force
```

### 0.3 개발용 HTTPS 인증서(ASP.NET Core)
```bash
dotnet dev-certs https --check
dotnet dev-certs https --trust   # OS 신뢰 저장소에 신뢰(사용자 확인 필요)
```

---

## 1. 템플릿 관리 — `dotnet new` 계열

사용자 제공 항목(목록/설치/제거)을 유지하고, 추가로 **검색/업데이트/템플릿 팩 구조/커스텀 템플릿 제작**까지 확장합니다.

### 1.1 템플릿 검색·목록
```bash
dotnet new search blazor
dotnet new --list
```
- `search`는 온라인 카탈로그에서 검색합니다.
- `--list`는 로컬에 설치된 템플릿을 나열합니다.

### 1.2 템플릿 설치/제거
```bash
# 버전 와일드카드 혹은 특정 버전
dotnet new --install Microsoft.AspNetCore.SpaTemplates::*
dotnet new --install "MyTemplate::1.0.0"

# 제거
dotnet new --uninstall MyTemplate
```

### 1.3 템플릿 팩(.nupkg/.zip) 구조 한눈에 보기
```
template-pack.nupkg
└── content/
    └── .template.config/
        ├── template.json      # 템플릿 메타데이터
        └── icon.png           # 템플릿 아이콘(선택)
    └── 프로젝트파일들...
```
- `template.json`에서 **이름/단축명(shortName)/기본 값/symbols/선택적 기능** 등을 선언합니다.

### 1.4 커스텀 템플릿 제작(요지)
```bash
# 1) 베이스 프로젝트 생성
dotnet new webapi -n MyCompany.Api.Template

# 2) .template.config 디렉터리와 template.json 추가
mkdir -p MyCompany.Api.Template/.template.config
# template.json은 필수 스키마를 준수

# 3) 템플릿 팩으로 패키징(NuGet)
dotnet new classlib -n DummyPack   # 임시 csproj를 팩 컨테이너로 활용
# DummyPack.csproj <Content>에 템플릿 폴더 포함 후
dotnet pack -c Release
# 생성된 .nupkg 배포 -> dotnet new --install <nupkg 경로 또는 피드>
```

---

## 2. 프로젝트 참조·구성 — `dotnet add reference`, `dotnet sln`

기존 내용에 **제거/상대경로 주의/솔루션 구조 팁**을 추가합니다.

### 2.1 참조 추가/제거
```bash
dotnet add MyApp/MyApp.csproj reference ../MyLibrary/MyLibrary.csproj
dotnet remove MyApp/MyApp.csproj reference ../MyLibrary/MyLibrary.csproj
```
- `.csproj`에 `<ProjectReference>`를 추가/제거합니다.
- **상대 경로**가 맞는지, **대소문자**(리눅스) 주의.

### 2.2 솔루션 관리
```bash
dotnet new sln -n MySolution
dotnet sln MySolution.sln add src/Web/Web.csproj src/Api/Api.csproj src/Lib/Lib.csproj
dotnet sln MySolution.sln remove src/Api/Api.csproj
```
- **권장 구조**: `src/`와 `tests/` 분리, 루트에 솔루션 파일 배치.
- 솔루션에 추가했더라도 **프로젝트 참조**는 별도로 추가해야 합니다.

---

## 3. 프로젝트 검사/분석 — `dotnet list`

사용자 제공 항목(패키지/참조) + **취약점·업데이트·트랜지티브 표시** 등 고급 옵션.

### 3.1 패키지 상태
```bash
dotnet list package
dotnet list package --outdated                   # 최신 버전 확인
dotnet list package --vulnerable                 # 알려진 취약점 보고
dotnet list package --include-transitive         # 전이 의존성 포함
```

### 3.2 참조 목록
```bash
dotnet list reference
```

---

## 4. NuGet 패키지 관리 — `dotnet add/remove/list package`, `dotnet nuget *`, `dotnet pack`

사용자 제공 내용에 **사설 피드, 캐시, 서명, 배포**까지 심화합니다.

### 4.1 패키지 추가/제거/목록
```bash
dotnet add package Newtonsoft.Json --version 13.0.3
dotnet remove package Newtonsoft.Json
dotnet list package --outdated
```

### 4.2 사설 피드/소스 관리
```bash
# 소스 추가/업데이트/제거
dotnet nuget add source "https://nuget.mycompany.com/v3/index.json" -n MyFeed
dotnet nuget update source MyFeed --username USER --password TOKEN --store-password-in-clear-text
dotnet nuget remove source MyFeed

# 인증 확인(피드 접근 실패 시)
dotnet nuget list source
```

### 4.3 캐시 정리(빌드 이슈 시 유용)
```bash
dotnet nuget locals all --list
dotnet nuget locals all --clear
```

### 4.4 패키지 생성/배포
`.csproj`에 메타데이터:
```xml
<PropertyGroup>
  <PackageId>MyCompany.MyLib</PackageId>
  <Version>1.2.0</Version>
  <Authors>MyCompany</Authors>
  <RepositoryUrl>https://github.com/myorg/myrepo</RepositoryUrl>
  <Description>Reusable library</Description>
  <PackageLicenseExpression>MIT</PackageLicenseExpression>
  <IncludeSymbols>true</IncludeSymbols>
  <SymbolPackageFormat>snupkg</SymbolPackageFormat>
</PropertyGroup>
```

생성/배포:
```bash
dotnet pack -c Release -o ./nupkg
dotnet nuget push ./nupkg/MyCompany.MyLib.1.2.0.nupkg \
  --api-key <TOKEN> --source https://api.nuget.org/v3/index.json
```
- GitHub Packages 예:
```bash
dotnet nuget push ./nupkg/*.nupkg \
  --api-key <GITHUB_PAT> \
  --source https://nuget.pkg.github.com/<OWNER>/index.json
```

---

## 5. 전역/로컬 도구(Global/Local Tools) — `dotnet tool *`

사용자 제공 항목에 **로컬 매니페스트/복원/실행/버전 고정**을 확장합니다.

### 5.1 전역 도구
```bash
dotnet tool install -g dotnet-ef
dotnet tool update  -g dotnet-ef
dotnet tool uninstall -g dotnet-ef
dotnet tool list -g
```

### 5.2 로컬 도구(프로젝트별 버전 고정)
```bash
# 로컬 매니페스트 생성(레포에 커밋)
dotnet new tool-manifest
dotnet tool install dotnet-ef
dotnet tool install dotnet-format
dotnet tool list

# CI/팀 동기화
dotnet tool restore
```
실행:
```bash
dotnet tool run dotnet-ef migrations add Init
dotnet tool run dotnet-format
```

---

## 6. 런타임/SDK 정보·업데이트 — `--info`, `--list-*`, `sdk check`, `workload *`

사용자 제공 내용에 **SDK 업데이트 확인, 워크로드 관리**를 확장합니다.

### 6.1 SDK 업데이트 확인
```bash
dotnet sdk check
```

### 6.2 워크로드(예: MAUI, WASM, Android, iOS)
```bash
dotnet workload search maui
dotnet workload install maui
dotnet workload list
dotnet workload update
dotnet workload repair
```
- 팀/CI에서도 동일 워크로드를 맞추기 위해 `workload restore`를 활용:
```bash
dotnet workload restore
```

---

## 7. 디버깅/트러블슈팅 — 로그·진단 도구·메모리/CPU 수집

사용자 제공 항목(상세 로그)을 보강하고, **진단 도구군**을 실제 사용 예와 함께 확장합니다.

### 7.1 빌드/실행 로그 상세
```bash
dotnet build --verbosity:diagnostic
dotnet publish --verbosity:detailed
```
- `quiet|minimal|normal|detailed|diagnostic` 단계가 있습니다.

### 7.2 진단 도구군(별도 설치 필요)
- `dotnet-trace`   : 이벤트 추적(Perf/CPU 샘플링)
- `dotnet-dump`    : 프로세스 덤프 생성/분석
- `dotnet-gcdump`  : GC 힙 통계 덤프
- `dotnet-counters`: 성능 카운터 실시간 수집
- `dotnet-monitor` : 컨테이너/서버에서 진단 엔드포인트 제공

설치/예시:
```bash
dotnet tool install -g dotnet-trace
dotnet tool install -g dotnet-dump
dotnet tool install -g dotnet-counters

# 실행 중 프로세스 식별(예: pid=1234)
dotnet-trace collect --process-id 1234 --providers Microsoft-Windows-DotNETRuntime:0x1c000080:5
dotnet-counters monitor System.Runtime --process-id 1234
dotnet-dump collect --process-id 1234
dotnet-dump analyze dump_1234.dmp
```

### 7.3 빌드 서버/캐시 문제 해결
```bash
dotnet build-server shutdown
dotnet nuget locals all --clear
```

---

## 8. 실행 — DLL 직접 실행, 구성/환경 변수

사용자 제공(직접 실행)을 유지하고, **환경 변수/URL/구성 오버라이드**를 추가합니다.

### 8.1 DLL 직접 실행
```bash
dotnet ./bin/Debug/net8.0/MyApp.dll
```

### 8.2 URL/환경 오버라이드(ASP.NET Core)
```bash
ASPNETCORE_URLS=http://+:8080 dotnet ./publish/MyWebApp.dll
ASPNETCORE_ENVIRONMENT=Production dotnet run
ConnectionStrings__Default="Server=..." dotnet run
```

---

## 9. 코드 변경 자동 반영 — `dotnet watch`

사용자 제공(기본)을 확장해 **Hot Reload/테스트**를 추가합니다.

```bash
# 웹앱 실행 + 변경 시 재시작/핫리로드
dotnet watch run

# 테스트 변경 감지
dotnet watch test
```
- 일부 시나리오에서 Hot Reload가 제한될 수 있으므로, **프로젝트 형식/언어 버전**을 확인하십시오.

---

## 10. 배포·성능 빌드 — publish 옵션 모음

고급 빌드 프로필을 빠르게 참고할 수 있는 모음입니다.

```bash
# Self-contained (런타임 포함)
dotnet publish -c Release -r linux-x64 --self-contained true -o out/linux

# 단일 파일
dotnet publish -c Release -r win-x64 -p:PublishSingleFile=true -o out/win

# 트리밍(사용 않는 IL 제거)
dotnet publish -c Release -r linux-x64 -p:PublishTrimmed=true -o out/trim

# ReadyToRun(빠른 시작)
dotnet publish -c Release -p:PublishReadyToRun=true -o out/r2r

# Native AOT(지원 대상 한정)
dotnet publish -c Release -r win-x64 -p:PublishAot=true -o out/aot
```
주의:
- 트리밍/AOT 시 **리플렉션/다이나믹 로드** 코드는 별도 보존 설정이 필요합니다.
- 플랫폼별 **RID**가 정확해야 합니다.

---

## 11. 고급 시나리오 예제

### 11.1 모노레포: Web + API + Lib + Tools(로컬 툴) 일관 빌드
```bash
mkdir -p mono/src/{Web,Api,Lib} mono/tests/Lib.Tests
cd mono

dotnet new sln -n Mono

dotnet new webapp -n Web -o src/Web
dotnet new api     -n Api -o src/Api
dotnet new classlib -n Lib -o src/Lib
dotnet new xunit -n Lib.Tests -o tests/Lib.Tests

dotnet sln Mono.sln add src/Web/Web.csproj src/Api/Api.csproj src/Lib/Lib.csproj tests/Lib.Tests/Lib.Tests.csproj

dotnet add src/Web/Web.csproj reference src/Lib/Lib.csproj
dotnet add src/Api/Api.csproj reference src/Lib/Lib.csproj
dotnet add tests/Lib.Tests/Lib.Tests.csproj reference src/Lib/Lib.csproj

# 로컬 도구 매니페스트(포맷터/EF/분석기 등)
dotnet new tool-manifest
dotnet tool install dotnet-ef
dotnet tool install dotnet-format
dotnet tool restore

# 빌드/테스트/포맷
dotnet build -c Release
dotnet test --logger "trx;LogFileName=test.trx"
dotnet tool run dotnet-format --verify-no-changes
```

### 11.2 커스텀 템플릿으로 팀 표준화
```bash
# 공통 레이어/디렉터리/CI 구성 포함 템플릿 제작 후
dotnet new --install ./MyCompany.TemplatePack.1.0.0.nupkg

# 팀 표준으로 새 프로젝트 생성
dotnet new mycompany-webapi -n Orders.Api --auth None --framework net8.0
```
- 팀 공통 규칙(분기명, 네이밍, 패키지, 애널라이저)을 **템플릿 수준**에서 강제할 수 있습니다.

### 11.3 사설 NuGet + 잠금 기반 복원
```bash
dotnet nuget add source "https://nuget.mycorp.com/v3/index.json" -n mycorp
dotnet restore --use-lock-file          # packages.lock.json 생성
dotnet restore --locked-mode            # 잠금 파일 기반 복원(버전 고정)
```

---

## 12. 유지보수·운영 체크리스트

1. SDK/워크로드 일관성  
   - `global.json`, `dotnet sdk check`, `dotnet workload list/restore`
2. 패키지/보안  
   - `dotnet list package --outdated/--vulnerable`, 주기적 업데이트
3. 캐시/빌드 서버  
   - `dotnet build-server shutdown`, `dotnet nuget locals all --clear`
4. 서명/라이선스/메타데이터  
   - `dotnet pack` 메타데이터 정합, 사내 규정 준수
5. 컨테이너/서버 운영 시 진단  
   - `dotnet-counters`, `dotnet-trace`, `dotnet-dump`, `dotnet-monitor`
6. 트리밍/AOT 영향  
   - 리플렉션 대상에 `DynamicDependency`/`DynamicallyAccessedMembers` 주석 적용

---

## 13. 명령어 요약(확장판)

| 목적 | 명령 |
|------|------|
| 템플릿 검색/설치/제거 | `dotnet new search`, `dotnet new --install`, `dotnet new --uninstall` |
| 프로젝트 참조 관리 | `dotnet add reference`, `dotnet list reference` |
| 패키지 점검 | `dotnet list package --outdated/--vulnerable` |
| 사설 피드/캐시 | `dotnet nuget add/update/remove source`, `dotnet nuget locals all --clear` |
| 도구(전역/로컬) | `dotnet tool install -g`, `dotnet new tool-manifest`, `dotnet tool restore` |
| SDK/워크로드 | `dotnet --list-sdks`, `dotnet sdk check`, `dotnet workload install/list/update` |
| 상세 로그 | `dotnet build --verbosity:diagnostic` |
| 진단 도구 | `dotnet-trace`, `dotnet-dump`, `dotnet-counters`, `dotnet-gcdump` |
| 자동 재실행 | `dotnet watch run`, `dotnet watch test` |
| 배포 빌드 | `dotnet publish`(+ `--self-contained`, `-p:PublishSingleFile`, `-p:PublishTrimmed`, `-p:PublishReadyToRun`, `-p:PublishAot`) |
| 빌드 서버 종료 | `dotnet build-server shutdown` |

---

## 14. 실전 흐름(종합)

```bash
# 1) 템플릿/워크로드 준비
dotnet new --list
dotnet workload list
dotnet sdk check

# 2) 솔루션/프로젝트 생성
dotnet new sln -n Suite
dotnet new webapp -n Web
dotnet new api -n Api
dotnet new classlib -n Core
dotnet sln Suite.sln add Web/Web.csproj Api/Api.csproj Core/Core.csproj
dotnet add Web/Web.csproj reference Core/Core.csproj
dotnet add Api/Api.csproj reference Core/Core.csproj

# 3) 패키지/도구/비밀
dotnet add Api/Api.csproj package Swashbuckle.AspNetCore
dotnet new tool-manifest
dotnet tool install dotnet-ef
dotnet tool restore
dotnet user-secrets init

# 4) 빌드/테스트/분석
dotnet restore
dotnet build -c Release --no-restore
dotnet test --logger "trx;LogFileName=test.trx"
dotnet list package --outdated

# 5) 배포 최적화
dotnet publish Web/Web.csproj -c Release -o out/web
dotnet publish Api/Api.csproj -c Release -r linux-x64 --self-contained true -p:PublishSingleFile=true -o out/api

# 6) 문제 발생 시
dotnet build --verbosity:diagnostic
dotnet build-server shutdown
dotnet nuget locals all --clear
```

---

# 결론

- 본 가이드는 기존 정리의 핵심을 유지하면서 **템플릿 제작/관리**, **사설 NuGet/잠금 복원**, **전역/로컬 도구 운용**, **워크로드 관리**, **진단 도구 활용**, **배포 최적화**까지 **현업 실무 관점**으로 확장했습니다.  
- 팀/CI 안정성을 위해 **SDK/워크로드/패키지 버전 고정**, **로컬 도구 매니페스트**, **잠금 기반 복원**을 적극 사용하십시오.  
- 성능·용량·시작 시간 요구가 있을 때는 `PublishSingleFile`, `PublishTrimmed`, `ReadyToRun`, `AOT`를 요구사항에 맞춰 조합하고, 진단 도구로 병목을 계측해 근거 기반으로 최적화하십시오.