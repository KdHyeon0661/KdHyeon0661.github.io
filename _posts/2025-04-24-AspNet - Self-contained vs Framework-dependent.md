---
layout: post
title: AspNet - Self-contained vs Framework-dependent
date: 2025-04-24 20:20:23 +0900
category: AspNet
---
# Self-contained vs Framework-dependent 배포

## 핵심 요약(의사결정 트리)

- **서버/컨테이너**(런타임 표준화, 빠른 패치) → **Framework-dependent (FDD)** 권장
- **설치형/폐쇄망/런타임 없는 환경** → **Self-contained (SCD)** 권장
- **최소 배포 크기**가 목표 → FDD + Single-file(런타임 제외) 또는 SCD + Trim
- **콜드 스타트/성능 개선**이 목표 → ReadyToRun(R2R) 또는 AOT(nativeaot) 검토

---

## 개요 복습

| 배포 방식 | 설명 |
|---|---|
| **Framework-dependent (FDD)** | 서버에 .NET 런타임이 **이미 설치**되어 있고, 앱은 **런타임에 의존**하여 실행 |
| **Self-contained (SCD)** | 앱 패키지에 **런타임 포함**. 서버에 .NET 설치 불필요 |

실행 방식
- FDD: `dotnet MyApp.dll`
- SCD: `./MyApp` (Linux) / `MyApp.exe` (Windows)

---

## 장단점 비교(심화)

| 항목 | Framework-dependent | Self-contained |
|---|---|---|
| 런타임 설치 | 필요 | 불필요 |
| 패치/보안 업데이트 | **중앙 관리**(런타임만 갱신) | **앱 재배포** 필요 |
| 배포 크기 | 작음(수~수십 MB) | 큼(100MB~수백 MB) |
| 플랫폼 의존성(RID) | 낮음(런타임만 맞으면 OK) | **플랫폼별 빌드** 필요 |
| 콜드 스타트 | 보통 | 보통(옵션에 따라 증가/감소) |
| 디버깅/진단 | 단순 | 일부 옵션 사용 시 차이(단일파일/트림) |
| 운영 환경 | 서버/클라우드/컨테이너 | 설치형/폐쇄망/키오스크/POS |
| 라이선스/감사 | 서버 런타임 기준 | 앱 번들 기준(3rd party 포함 시 검토 필요) |

---

## 기본 빌드 예제

### Framework-dependent(FDD)

```bash
dotnet publish -c Release -o ./publish
# 또는 플랫폼을 지정하되, self-contained는 false

dotnet publish -c Release -r win-x64 --self-contained false -o ./publish
# 실행

dotnet ./publish/MyApp.dll
```

### Self-contained(SCD)

```bash
# Windows x64

dotnet publish -c Release -r win-x64 --self-contained true -o ./publish
# Linux x64

dotnet publish -c Release -r linux-x64 --self-contained true -o ./publish
# 실행

./publish/MyApp            # (Linux)
./publish/MyApp.exe        # (Windows)
```

---

## 고급 퍼블리시 옵션(공통 적용 가능)

### Single-file (단일 파일)

> 배포 편의성 향상(파일 하나). **FDD**도 가능하지만, 보통 **SCD + Single-file**을 활용.

```bash
dotnet publish -c Release -r linux-x64 --self-contained true \
  -p:PublishSingleFile=true -o ./publish
```

추가 옵션
- `-p:IncludeNativeLibrariesForSelfExtract=true`
  네이티브 라이브러리를 실행 시 임시 디렉터리로 **추출**(호환성↑)
- `-p:SelfContained=true -p:PublishSingleFile=true -p:PublishTrimmed=true`
  단일 파일 + 트리밍(크기↓). **리플렉션/동적 로딩 주의**.

런타임 추출 경로 제어(환경변수)
- `DOTNET_BUNDLE_EXTRACT_BASE_DIR=/tmp/dotnet_bundle`
- `DOTNET_BUNDLE_EXTRACT_TO_TEMP=1` (강제 임시 폴더)

### ReadyToRun (사전 JIT: R2R)

> **콜드 스타트 개선**(특히 Windows 서비스/함수형 앱). 크기 증가.

```bash
dotnet publish -c Release -r win-x64 --self-contained true \
  -p:PublishReadyToRun=true -o ./publish
```

- `-p:PublishReadyToRunComposite=true` (일부 시나리오에 콜드 스타트 추가 개선)
- R2R은 **JIT 일부 대체**. 런타임/CPU에 따라 성능 편차 존재.

### AOT(nativeaot, .NET 8+)

> 네이티브 바이너리로 **초저지연/초경량 런타임**. 제약 많음(리플렉션/동적로딩 제한), 호환성 검토 필수.

```bash
dotnet publish -c Release -r linux-x64 -p:PublishAot=true -o ./publish
```

- 서버보단 **CLI/에이전트/함수형 워크로드**에 유리
- 크기 작고 시작 빠름. 대신 기능 제약/빌드시간↑

### Trim (링커로 미사용 코드 제거)

```bash
dotnet publish -c Release -r linux-x64 --self-contained true \
  -p:PublishTrimmed=true -o ./publish
```

- **반드시 테스트 필요**: 리플렉션/다이내믹 호출/소스제너레이터 사용 시 `TrimmerRootDescriptor.xml` 또는 `DynamicDependency` 특성으로 **보존 규칙** 지정 필요
- 이점: 크기 절감, 보안 표면 축소. 리스크: 런타임 누락 오류.

### Globalization/ICU 축소

```bash
# 문화권/지역화 기능 최소화 → 크기↓

dotnet publish -c Release -r linux-x64 --self-contained true \
  -p:InvariantGlobalization=true -o ./publish
```

- 단, 문화권별 대소문자/정렬/포맷이 중요한 앱은 **비권장**.
- Linux에서 ICU 데이터 패키지를 별도 관리 가능.

---

## csproj 구성 템플릿(프로파일/환경별)

### 다중 프로파일(조건부 PropertyGroup)

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <!-- 기본(FDD) -->
  <PropertyGroup Condition="'$(Configuration)'=='Release' AND '$(PublishProfile)'==''">
    <SelfContained>false</SelfContained>
    <PublishSingleFile>false</PublishSingleFile>
  </PropertyGroup>

  <!-- SCD + R2R + Single-file (Windows) -->
  <PropertyGroup Condition="'$(PublishProfile)'=='scd-win'">
    <RuntimeIdentifier>win-x64</RuntimeIdentifier>
    <SelfContained>true</SelfContained>
    <PublishSingleFile>true</PublishSingleFile>
    <PublishReadyToRun>true</PublishReadyToRun>
    <IncludeNativeLibrariesForSelfExtract>true</IncludeNativeLibrariesForSelfExtract>
  </PropertyGroup>

  <!-- SCD + Trim + Single-file (Linux) -->
  <PropertyGroup Condition="'$(PublishProfile)'=='scd-linux-trim'">
    <RuntimeIdentifier>linux-x64</RuntimeIdentifier>
    <SelfContained>true</SelfContained>
    <PublishSingleFile>true</PublishSingleFile>
    <PublishTrimmed>true</PublishTrimmed>
    <InvariantGlobalization>true</InvariantGlobalization>
  </PropertyGroup>
</Project>
```

사용 예:

```bash
# 기본(FDD)

dotnet publish -c Release -o ./publish

# SCD Windows

dotnet publish -c Release -p:PublishProfile=scd-win -o ./publish

# SCD Linux Trim

dotnet publish -c Release -p:PublishProfile=scd-linux-trim -o ./publish
```

---

## 심화

- SCD는 **RID를 명시**해야 한다(`win-x64`, `linux-x64`, `osx-arm64`, `linux-arm`, `linux-musl-x64` 등).
- **musl**(Alpine Linux) 기반은 별도 RID(`linux-musl-x64`) 필요.
- 다중 플랫폼 대상으로 **CI 매트릭스** 구성:

```yaml
strategy:
  matrix:
    rid: [ 'win-x64', 'linux-x64', 'osx-arm64' ]
steps:
  - run: dotnet publish -c Release -r ${{ matrix.rid }} --self-contained true -o ./publish/${{ matrix.rid }}
```

---

## 보안/패치/유지보수 전략

| 관점 | FDD | SCD |
|---|---|---|
| 보안 패치 | 서버 런타임만 갱신 → **즉시 반영** | 새 이미지/패키지 재배포 |
| 운영 통제 | 중앙집중(런타임) | 앱 버전 단위 관리 |
| 사고시 롤백 | 런타임 롤백 가능 | 해당 앱만 롤백(버전별 유지 필요) |

권장:
- **서버/컨테이너 환경**: FDD + 정기 런타임 패치 파이프라인
- **설치형 제품**: SCD + 자동 업데이트 채널(인앱 업데이트, 배포자 도구)

---

## 성능/시작 시간 최적화 가이드

- **R2R**: 콜드스타트 개선, 용량 증가 감수
- **AOT**: 극저지연/초경량, 호환성 제약
- **Single-file**: 파일 I/O 간소화, 추출 모드에선 최초 실행 지연 가능
- **Tiered JIT/PGO**: 기본 활성화(.NET 8), **실제 트래픽 기반**으로 최적화 진전
- **Server GC**: 서버 워크로드에서 이점, 컨테이너 메모리 제한 고려

---

## 컨테이너에서의 선택

- 보통 **FDD** (런타임 포함 base 이미지 사용)
  예: `mcr.microsoft.com/dotnet/aspnet:8.0`
- SCD는 **런타임 미포함** 간단 base 이미지로 크기↓ 가능
  단, libc/ICU 등 OS 라이브러리 의존성 고려
- 멀티스테이지 Dockerfile로 크기/보안 최소화

```dockerfile
# build

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /out

# run (FDD)

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /out .
ENV ASPNETCORE_URLS=http://0.0.0.0:8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

SCD 컨테이너(런타임 미포함 베이스, 필요 라이브러리 확인):

```dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends libicu-dev ca-certificates \
  && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY ./publish .
ENV ASPNETCORE_URLS=http://0.0.0.0:8080
EXPOSE 8080
ENTRYPOINT ["./MyApp"]  # SCD 단일 바이너리
```

---

## 진단/로깅/크래시덤프 차이

- Single-file + 추출 모드: 일시 디렉터리에 어셈블리 전개 → **파일 경로/심볼 로드** 방식 유의
- Trim/AOT: 리플렉션 기반 라이브러리/로거 템플릿이 **제거**될 수 있음 → 루트 보존 규칙 정의
- 크래시덤프/프로파일러: SCD 네이티브/경량 환경에서 **도구 호환성** 확인

---

## 스크립트 예시(운영 배포 파이프라인)

### Linux 배포 스크립트(SCD + R2R + 단일파일)

```bash
#!/usr/bin/env bash

set -euo pipefail

RID="linux-x64"
OUT="./publish/${RID}"

dotnet restore
dotnet publish -c Release -r $RID --self-contained true \
  -p:PublishSingleFile=true \
  -p:PublishReadyToRun=true \
  -p:IncludeNativeLibrariesForSelfExtract=true \
  -o $OUT

# 서빙 경로로 교체(원자적 교체를 위해 심볼릭 링크/버전 폴더 사용 권장)

sudo systemctl stop myapp || true
rsync -av --delete "$OUT/" /var/www/myapp/
sudo systemctl start myapp
sudo systemctl status myapp --no-pager
```

### PowerShell(Windows, FDD + IIS)

```powershell
$Out = ".\publish"
dotnet publish -c Release -o $Out
# IIS 사이트 경로에 복사

Copy-Item "$Out\*" "C:\Sites\MyApp\" -Recurse -Force
iisreset
```

---

## 흔한 오류와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| 실행 시 라이브러리 로드 실패 | SCD에서 OS 라이브러리 누락 | OS 의존 패키지 설치(예: `libicu`), `IncludeNativeLibrariesForSelfExtract` 검토 |
| 단일파일 처음 실행 느림 | 추출 모드 | 환경변수로 추출 위치 고정, 또는 추출 없는 구성(호환성 확인) |
| Trim 후 런타임 오류 | 리플렉션/다이내믹 제거 | `TrimmerRootDescriptor.xml`/`DynamicDependency`로 보존 |
| R2R 성능 역전 | CPU/런타임/옵션 불일치 | 실제 워크로드 프로파일링, R2R on/off A/B 테스트 |
| AOT 빌드 실패 | 지원 외 기능/리플렉션 | AOT 가이드에 맞춘 코드/소스제너레이터 도입 |

---

## 실무 선택 가이드(상황별)

| 상황 | 권장 |
|---|---|
| **Kubernetes/컨테이너** 다수 서비스 | **FDD**, 표준 런타임 이미지 + 중앙 패치 |
| **폐쇄망 납품/설치형** | **SCD**, 자동 업데이트 채널 포함 |
| **초저지연 CLI/에이전트** | **AOT** 우선 검토(호환성 체크) |
| **대규모 배포, 빠른 롤백** | FDD(런타임 롤백/패치) 또는 App Service 슬롯 |
| **아주 작은 배포 이미지 필요** | SCD + Trim + InvariantGlobalization + Single-file (테스트 필수) |

---

## 예제: 모든 조합을 비교하는 매트릭스 빌드

GitHub Actions 매트릭스:

```yaml
name: publish-matrix
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        rid: [ 'win-x64', 'linux-x64' ]
        mode: [ 'fdd', 'scd' ]
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-dotnet@v4
      with: { dotnet-version: '8.x' }
    - run: dotnet restore
    - name: Publish
      run: |
        if [ "${{ matrix.mode }}" = "fdd" ]; then
          dotnet publish -c Release -r ${{ matrix.rid }} --self-contained false -o ./publish/${{ matrix.rid }}-fdd
        else
          dotnet publish -c Release -r ${{ matrix.rid }} --self-contained true \
            -p:PublishSingleFile=true -p:PublishReadyToRun=true \
            -o ./publish/${{ matrix.rid }}-scd
        fi
    - uses: actions/upload-artifact@v4
      with:
        name: publish-${{ matrix.rid }}-${{ matrix.mode }}
        path: ./publish/**
```

---

## 보너스: `runtimeconfig.json`/RollForward

FDD에서 런타임 상향 호환(롤포워드) 정책은 **호환범위**를 제어:

- `DOTNET_ROLL_FORWARD` 또는 `*.runtimeconfig.json` 내 `rollForward`
  예: `LatestMajor`, `Minor`, `Feature`, `Disable` 등
- 서버에 설치된 런타임과의 매칭 정책에 따라 **실행 가능/불가** 결정

---

## 결론

- **운영 표준화/패치 민첩성**이 중요하면 **Framework-dependent**.
- **런타임 없는 환경/설치형 배포**가 필요하면 **Self-contained**.
- 성능/크기 목표에 따라 **Single-file/ReadyToRun/Trim/AOT**를 **선택적으로 조합**하라.
- 무엇보다 **프로파일링/AB 테스트**로 실제 워크로드에 맞춘 결정을 반복하라.

---

## 샘플 `TrimmerRootDescriptor.xml` (Trim 안전 보존)

```xml
<linker>
  <assembly fullname="MyApp">
    <type fullname="MyApp.SomeTypeUsedViaReflection" />
    <method signature="System.String ToString()" />
  </assembly>
</linker>
```

`csproj`에 포함:

```xml
<ItemGroup>
  <TrimmerRootDescriptor Include="TrimmerRootDescriptor.xml" />
</ItemGroup>
```

---

## 최소 예제 csproj (옵션 실험판)

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>

    <!-- 실험: SCD + Single-file + R2R -->
    <SelfContained>true</SelfContained>
    <RuntimeIdentifier>linux-x64</RuntimeIdentifier>
    <PublishSingleFile>true</PublishSingleFile>
    <PublishReadyToRun>true</PublishReadyToRun>

    <!-- 크기 최적화 옵션 -->
    <!-- <PublishTrimmed>true</PublishTrimmed> -->
    <!-- <InvariantGlobalization>true</InvariantGlobalization> -->
  </PropertyGroup>
</Project>
```

---

## 수치/비율에 대한 간단한 수학적 감각

배포 크기 최적화는 보통 **가산적 절감**이 아닌 **곱셈적(비율) 절감**으로 체감된다.
예를 들어, 초기 크기 \( S_0 \) 에 대해 Trim으로 \( \alpha \)(0<α<1) 만큼 절감,
단일파일 패킹으로 \( \beta \) 만큼 추가 절감이라면, 최종 크기 \( S_f \)는

$$
S_f \approx S_0 \times \alpha \times \beta
$$

- 각 최적화의 **상호작용**(리소스/네이티브 추출 여부)에 따라 실제 값은 달라진다.
- 결론: **개별 절감률의 곱**이므로 작은 최적화도 **누적**되면 유의미하다.
