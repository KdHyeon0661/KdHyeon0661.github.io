---
layout: post
title: AspNet - NET Core CI/CD 구성
date: 2025-04-29 19:20:23 +0900
category: AspNet
---
# ASP.NET Core CI/CD 구성 (with GitHub Actions)

## 1. CI/CD 핵심 개념(요약 복습)

| 구분 | 핵심 |
|---|---|
| CI | 코드 변경 시 자동으로 복원/빌드/정적분석/테스트/패키징을 수행 |
| CD | CI 산출물(아티팩트/이미지)을 표준화된 절차로 **안전하게** 배포 |
| 목표 | 자동화/일관성/가시성/빠른 피드백/안전한 롤백 |

---

## 2. GitHub Actions 기본 빌딩 블록

| 요소 | 설명 | 팁 |
|---|---|---|
| Workflow | `.github/workflows/*.yml`에 정의된 파이프라인 | 여러 파일로 **역할 분리**(CI vs CD) |
| Job | 독립 실행 단위(병렬/순차) | `needs:`로 의존성 지정 |
| Step | 실행 단계(액션 호출 or shell) | 공통 Step은 **Reusable Workflow/Composite Action**으로 추출 |
| Runner | 호스트(ubuntu/windows/mac/self-hosted) | 리눅스가 속도/비용 면에서 유리 |

---

## 3. 표준 리포지토리 구조와 브랜치 전략

```bash
MyApp/
├── src/
│   └── MyApp/                # ASP.NET Core 프로젝트
├── tests/
│   └── MyApp.Tests/          # 단위/통합 테스트
├── .github/
│   └── workflows/
│       ├── ci.yml            # PR/Push 시 빌드·테스트·검증
│       ├── cd-staging.yml    # 스테이징 자동 배포 (main merge)
│       └── cd-prod.yml       # 운영 수동 승인 배포(릴리스 태그)
```

- 브랜치: `feature/*` → PR → `develop` → `main`  
- 배포: `main` merge 시 **Staging**, 태그 `v*` 푸시 시 **Production**

---

## 4. 안전한 자격 증명: Secrets vs OIDC 연동

### 4.1 Secrets
- 위치: GitHub → Repository → Settings → Secrets and variables → Actions
- 예: `AZURE_WEBAPP_NAME`, `AZURE_WEBAPP_PUBLISH_PROFILE`, `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`

장점: 간편.  
단점: 장기 비밀 보관, **회전 주기/노출 위험** 관리 필요.

### 4.2 OIDC (Federated Credentials)
- GitHub Actions가 **클라우드에 신뢰 토큰을 교환**하여 **장기 키 없이** 로그인
- Azure 예시: **Entra ID**에 **Federated Credentials** 생성 → 워크플로에서 `azure/login@v2`로 로그인

{% raw %}
```yaml
- name: Azure login via OIDC
  uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id:  ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```
{% endraw %}

장점: 키 없는 보안, 중앙 관리.  
권장: 운영 계정은 **가능하면 OIDC**를 기본으로.

---

## 5. NuGet 캐시/속도 최적화 베스트 프랙티스

{% raw %}
```yaml
- name: Setup .NET
  uses: actions/setup-dotnet@v4
  with:
    dotnet-version: 8.0.x

- name: Cache NuGet
  uses: actions/cache@v4
  with:
    path: ~/.nuget/packages
    key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
    restore-keys: |
      ${{ runner.os }}-nuget-
```
{% endraw %}

- `hashFiles('**/*.csproj')`로 의존성 변경 시 자동 무효화  
- **checkout 깊이**: `actions/checkout@v3` with `fetch-depth: 0`은 버전/체인지로그 계산에 유용

---

## 6. CI 워크플로(확장형)

{% raw %}
```yaml
name: CI

on:
  pull_request:
    branches: [ main, develop ]
  push:
    branches: [ develop ]

jobs:
  build-test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
    - name: Checkout
      uses: actions/checkout@v3
      with: { fetch-depth: 0 }

    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with: { dotnet-version: 8.0.x }

    - name: Cache NuGet
      uses: actions/cache@v4
      with:
        path: ~/.nuget/packages
        key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
        restore-keys: ${{ runner.os }}-nuget-

    - name: Restore
      run: dotnet restore

    - name: Build
      run: dotnet build --configuration Release --no-restore /warnaserror

    - name: Lint (format check)
      run: dotnet format --verify-no-changes

    - name: Test with coverage
      run: |
        dotnet test --no-build --configuration Release \
          /p:CollectCoverage=true \
          /p:CoverletOutputFormat=cobertura \
          /p:CoverletOutput=TestResults/coverage.cobertura.xml

    - name: Publish Test Results
      uses: actions/upload-artifact@v4
      with:
        name: test-results
        path: "**/TestResults/*.trx"

    - name: Publish Coverage
      uses: actions/upload-artifact@v4
      with:
        name: coverage
        path: "**/TestResults/coverage.cobertura.xml"
```
{% endraw %}

포인트:
- `dotnet format`으로 포맷 일관성
- `/warnaserror`로 빌드 경고를 실패로 승격(선택)
- 커버리지 산출물 업로드(이후 게이트/배지에 활용)

---

## 7. 아티팩트/버전/릴리스 노트 자동화

### 7.1 버전 자동화(태그 기반)
```yaml
- name: Get Version from Tag or Commit
  id: ver
  run: |
    if [[ "${GITHUB_REF}" == refs/tags/v* ]]; then
      echo "version=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
    else
      echo "version=0.0.0-${GITHUB_SHA:0:7}" >> $GITHUB_OUTPUT
    fi
```

{% raw %}`dotnet publish -p:Version=${{ steps.ver.outputs.version }}`{% endraw %}로 주입.

### 7.2 릴리스 노트
```yaml
- name: Create GitHub Release (on tag)
  if: startsWith(github.ref, 'refs/tags/v')
  uses: softprops/action-gh-release@v1
  with:
    generate_release_notes: true
    files: |
      publish/**
```

---

## 8. CD: Azure App Service(슬롯/승인/롤백)

### 8.1 Staging 자동 배포

{% raw %}
```yaml
name: CD-Staging

on:
  push:
    branches: [ main ]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: ${{ steps.deploy.outputs.webapp-url }}
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v4
      with: { dotnet-version: 8.0.x }

    - run: dotnet publish ./src/MyApp/MyApp.csproj -c Release -o ./publish

    - name: Azure login via OIDC
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id:  ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Deploy to staging slot
      id: deploy
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ secrets.AZURE_WEBAPP_NAME }}
        slot-name: 'staging'
        package: ./publish
```
{% endraw %}

### 8.2 운영 수동 승인(Environments 보호 규칙)
- GitHub → Settings → Environments → `production` → **Required reviewers** 지정

{% raw %}
```yaml
name: CD-Prod

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    environment:
      name: production  # 보호된 환경 → 승인 요구
    steps:
      # 빌드/로그인 생략
      - name: Swap staging->production
        uses: azure/cli@v2
        with:
          inlineScript: |
            az webapp deployment slot swap \
              --name ${{ secrets.AZURE_WEBAPP_NAME }} \
              --resource-group ${{ secrets.AZURE_RG }} \
              --slot staging --target-slot production
```
{% endraw %}

- 롤백: **역방향 슬롯 스왑** 또는 이전 릴리스 재배포

---

## 9. Docker 기반 배포(빌드 캐시/멀티플랫폼)

### 9.1 Dockerfile(멀티스테이지)
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish ./src/MyApp/MyApp.csproj -c Release -o /out

FROM base AS final
WORKDIR /app
COPY --from=build /out .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### 9.2 GH Actions: buildx + 캐시 + 푸시

{% raw %}
```yaml
- name: Set up QEMU
  uses: docker/setup-qemu-action@v3

- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Login to DockerHub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

- name: Build and push
  uses: docker/build-push-action@v6
  with:
    context: .
    file: ./Dockerfile
    push: true
    tags: myuser/myapp:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
    platforms: linux/amd64,linux/arm64
```
{% endraw %}

- 멀티플랫폼 이미지와 GH 캐시를 활용해 **속도/호환성** 극대화

---

## 10. Kubernetes 배포(옵션)

- 매니페스트/Helm 차트 관리, 이미지 태그로 롤링 업데이트
- 예: `kubectl set image deploy/myapp myapp=myuser/myapp:${{ steps.ver.outputs.version }}`

{% raw %}
```yaml
- name: Set Kube context
  uses: azure/k8s-set-context@v3
  with:
    method: kubeconfig
    kubeconfig: ${{ secrets.KUBECONFIG }}

- name: Rolling update
  run: kubectl set image deployment/myapp myapp=myuser/myapp:${{ steps.ver.outputs.version }}
```
{% endraw %}

- 카나리/블루그린: **서비스 셀렉터** 또는 **Ingress 라우팅**으로 점진 전환

---

## 11. IIS/Windows 서버 배포(옵션)

- 방법: Web Deploy(MSDeploy) / WinRM / FTP
- 간단 FTP 예시:

{% raw %}
```yaml
- name: Deploy via FTP
  uses: SamKirkland/FTP-Deploy-Action@v4
  with:
    server: ${{ secrets.FTP_SERVER }}
    username: ${{ secrets.FTP_USER }}
    password: ${{ secrets.FTP_PASS }}
    local-dir: ./publish
    server-dir: /site/wwwroot
```
{% endraw %}

---

## 12. 데이터베이스 마이그레이션(EF Core)

- 스테이징/운영 배포 전후에 **마이그레이션 명령** 실행
- 무중단을 위해 **백필드/점진 스키마** 전략 적용

{% raw %}
```yaml
- name: EF Core migration (App Service Exec)
  uses: azure/cli@v2
  with:
    inlineScript: |
      az webapp ssh --name ${{ secrets.AZURE_WEBAPP_NAME }} \
                    --resource-group ${{ secrets.AZURE_RG }} \
                    --command "dotnet MyApp.dll --migrate"
```
{% endraw %}

또는 별도 **DB 마이그레이션 작업 컨테이너**를 만들어 배포 훅에서 실행.

---

## 13. 환경 변수/설정 주입

- `appsettings.{ENV}.json` + **환경 변수** 오버라이드
- GH Actions의 **Environment Variables / Secrets**로 안전하게 주입

{% raw %}
```yaml
env:
  ASPNETCORE_ENVIRONMENT: "Production"
  ConnectionStrings__DefaultConnection: ${{ secrets.PROD_DB }}
```
{% endraw %}

---

## 14. 정적 분석/보안/라이선스/SBOM

- `dotnet analyzers`, `dotnet format`, `SonarCloud`, `CodeQL`(GitHub 제공)
- SBOM/서명:
```yaml
- name: Generate SBOM
  run: dotnet build-server shutdown && dotnet msbuild /t:GenerateSbom

- name: CodeQL Init
  uses: github/codeql-action/init@v3
  with: { languages: 'csharp' }

- name: CodeQL Analyze
  uses: github/codeql-action/analyze@v3
```

---

## 15. 모노레포/조건부 실행/재사용 워크플로

### 조건부 실행(특정 경로 변경 시에만)
```yaml
on:
  push:
    branches: [ main ]
    paths:
      - 'src/MyApp/**'
      - '.github/workflows/**'
```

### 재사용 워크플로 호출
```yaml
jobs:
  call-shared:
    uses: org/repo/.github/workflows/shared-ci.yml@main
    with:
      dotnet-version: '8.0.x'
```

---

## 16. 실패 자동 롤백/배포 잠금/동시성

- **동시성**: 같은 브랜치의 배포 충돌 방지
```yaml
concurrency:
  group: prod-deploy
  cancel-in-progress: true
```

- 배포 잠금/승인: Environments 보호 규칙
- 롤백: **이전 릴리스/이미지 태그**로 재배포, **슬롯 스왑 되돌리기**

---

## 17. 셀프 호스티드 러너(옵션)

- 장점: 사내 네트워크/비공개 자원 접근, 속도/캐시 향상
- 주의: **보안 격리**, 최소 권한 PAT/OIDC, 러너 자동 패치

---

## 18. 통합 예: CI → Staging → 승인 → Prod

아래는 **CI**, **Staging CD**, **Prod CD**를 분리한 현실적인 구성 예다.

### 18.1 ci.yml (PR/Develop)
```yaml
name: CI
on:
  pull_request: { branches: [ main, develop ] }
  push: { branches: [ develop ] }
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      # checkout/setup/cache/restore/build/format/test/coverage 업로드 (6장 참조)
```

### 18.2 cd-staging.yml (main merge → 자동 배포)
```yaml
name: CD-Staging
on:
  push: { branches: [ main] }
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      # publish → OIDC 로그인 → App Service staging 배포
```

### 18.3 cd-prod.yml (태그 → 승인 후 운영)
```yaml
name: CD-Prod
on:
  push:
    tags: [ 'v*.*.*' ]
jobs:
  prod:
    runs-on: ubuntu-latest
    environment:
      name: production # 보호된 환경(승인 필요)
    steps:
      # 슬록 스왑 or Helm upgrade or 쿠버네티스 롤링 업데이트
```

---

## 19. 문제 해결 체크리스트

| 증상 | 원인/해결 |
|---|---|
| NuGet restore 매우 느림 | 캐시 키/restore-keys 점검, 사설 피드 인증 |
| 빌드/테스트 시간 과다 | `dotnet test --filter`, 병렬 테스트, 슬라이싱 |
| App Service 배포 실패 | 권한/리소스 그룹/슬롯 이름/런타임 스택 확인 |
| 출력 경로 오류 | `dotnet publish -o` 경로, `package` 입력 경로 확인 |
| 환경변수 미적용 | `env`/`secrets` 스코프, App Service 구성, `ASPNETCORE_ENVIRONMENT` 확인 |
| Docker 이미지 커짐 | 멀티스테이지/트리밍/디펜던시 정리, `--no-cache` 주의 |
| 커버리지 0% | 테스트 프로젝트 `CollectCoverage` 설정/경로/Framework 대상 확인 |

---

## 20. 결론 요약

| 항목 | 권장사항 |
|---|---|
| 보안 | 가능한 한 **OIDC**로 클라우드 로그인, Secrets 최소화 |
| 속도 | **NuGet 캐시**·빌드 병렬화·테스트 필터링 |
| 품질 | 포맷/정적분석/커버리지 게이트 |
| 배포 | **Staging 자동** + **Production 승인/슬롯 스왑/롤백** |
| 컨테이너 | buildx + 캐시 + 멀티플랫폼, 쿠버네티스면 카나리/블루그린 |
| 운영 | 릴리스/버전/릴리스 노트/SBOM/CodeQL로 가시성·규범 강화 |

---

## 부록 A) 단일 파일 예시(ci-cd.yml, Azure Web App 배포)

{% raw %}
```yaml
name: ASP.NET Core CI/CD

on:
  push: { branches: [ main ] }
  pull_request: { branches: [ main ] }

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v4
      with: { dotnet-version: 8.0.x }
    - uses: actions/cache@v4
      with:
        path: ~/.nuget/packages
        key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
        restore-keys: ${{ runner.os }}-nuget-
    - run: dotnet restore
    - run: dotnet build -c Release --no-restore
    - run: dotnet test -c Release --no-build /p:CollectCoverage=true /p:CoverletOutput=TestResults/coverage.cobertura.xml /p:CoverletOutputFormat=cobertura
    - uses: actions/upload-artifact@v4
      with: { name: coverage, path: "**/TestResults/coverage.cobertura.xml" }
    - run: dotnet publish ./src/MyApp/MyApp.csproj -c Release -o ./publish
    - name: Azure login via OIDC
      if: github.ref == 'refs/heads/main'
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id:  ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    - name: Deploy to Azure Web App (staging)
      if: github.ref == 'refs/heads/main'
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ secrets.AZURE_WEBAPP_NAME }}
        slot-name: 'staging'
        package: ./publish
```
{% endraw %}

---

## 부록 B) Docker 빌드 후 쿠버네티스 롤링 업데이트 예시

{% raw %}
```yaml
name: CI-Docker-K8s

on:
  push:
    tags: [ 'v*.*.*' ]

jobs:
  build-push:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.ver.outputs.version }}
    steps:
    - uses: actions/checkout@v3
    - name: Extract version
      id: ver
      run: echo "version=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
    - uses: docker/setup-buildx-action@v3
    - uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    - uses: docker/build-push-action@v6
      with:
        context: .
        file: ./Dockerfile
        push: true
        tags: myuser/myapp:${{ steps.ver.outputs.version }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy:
    runs-on: ubuntu-latest
    needs: build-push
    environment: production
    steps:
    - name: Set Kube context
      uses: azure/k8s-set-context@v3
      with:
        method: kubeconfig
        kubeconfig: ${{ secrets.KUBECONFIG }}
    - name: Rolling update
      run: |
        kubectl set image deployment/myapp myapp=myuser/myapp:${{ needs.build-push.outputs.version }}
        kubectl rollout status deployment/myapp --timeout=120s
```
{% endraw %}