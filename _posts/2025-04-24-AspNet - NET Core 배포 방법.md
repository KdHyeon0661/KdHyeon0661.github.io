---
layout: post
title: AspNet - NET Core 배포 방법
date: 2025-04-24 19:20:23 +0900
category: AspNet
---
# ASP.NET Core 배포 방법 완전 정복 (IIS, Linux, Docker, Azure)

## 0. 배포 전략 큰 그림

- ASP.NET Core는 **자체 웹서버(Kestrel)** 를 포함한다. 엣지(Edge) 역활은 IIS/Nginx/Apache(리버스 프록시)가 담당.
- 배포 4대 축
  1) Windows 서버의 **IIS**  
  2) Linux 서버의 **Kestrel + Nginx**  
  3) **Docker** 컨테이너(→ Kubernetes 확장 가능)  
  4) **Azure App Service** (PaaS, 슬롯/모니터링/스케일 빵빵)
- 공통 운영 필수 요소: **환경 변수/비밀키**, **HTTPS/HSTS**, **헬스체크**(`/health`), **구조화 로깅**, **역프록시 헤더 처리**, **무중단 배포**.

---

## 1. 사전 공통: Publish 산출물과 런타임 모드

### 1.1 Self-contained vs Framework-dependent

| 구분 | 설명 | 장점 | 단점 | 사용 예 |
|---|---|---|---|---|
| Framework-dependent | 서버에 .NET Runtime 설치 필요 | 패키지 작음 | 서버 사전 준비 필요 | IIS, Linux 런타임 이미 구축된 환경 |
| Self-contained | 앱에 런타임 포함 | 서버 준비 최소화 | 산출물 큼 | 폐쇄망/런타임 없는 서버 |

```bash
# Framework-dependent
dotnet publish -c Release -o ./publish

# Self-contained (Linux x64 예시)
dotnet publish -c Release -r linux-x64 --self-contained true -p:PublishTrimmed=true -o ./publish
```

### 1.2 appsettings.{ENV}.json + 환경 변수 구성

- `ASPNETCORE_ENVIRONMENT` 로 Development/Staging/Production 분기
- 민감정보는 **환경 변수** 또는 **User Secrets(개발용)** / 비밀 저장소(Azure Key Vault 등)

```bash
# Linux 예
export ASPNETCORE_ENVIRONMENT=Production
export ConnectionStrings__Default="Server=db;Database=app;User Id=app;Password=***"
```

---

## 2. Windows IIS 배포 (Internet Information Services)

### 2.1 준비물

- **.NET Hosting Bundle** (ASP.NET Core Module 포함) 설치  
  → dotnet 공식 다운로드의 Hosting Bundle
- Windows Server + IIS 역할(Role)
- 방화벽/포트(80/443) 개방

### 2.2 Publish & 사이트 구성

```bash
dotnet publish -c Release -o C:\Sites\MyApp
```

IIS 관리자 → **사이트 추가**  
- 물리 경로: `C:\Sites\MyApp`  
- 호스트 이름/포트: 실제 운영 도메인/포트

### 2.3 web.config (in-process 권장)

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <add name="aspNetCore"
           path="*"
           verb="*"
           modules="AspNetCoreModuleV2"
           resourceType="Unspecified"/>
    </handlers>
    <aspNetCore processPath="dotnet"
                arguments="MyApp.dll"
                stdoutLogEnabled="true"
                stdoutLogFile=".\logs\stdout"
                hostingModel="inprocess">
      <environmentVariables>
        <environmentVariable name="ASPNETCORE_ENVIRONMENT" value="Production" />
        <environmentVariable name="ConnectionStrings__Default" value="Server=...;..." />
      </environmentVariables>
    </aspNetCore>
  </system.webServer>
</configuration>
```

> `hostingModel="inprocess"`: IIS 워커 프로세스 내 호스팅으로 성능/일체감 우수.  
> `stdoutLogEnabled` 는 **문제 발생 시에만** 잠깐 켜고, 정상화 후 꺼둘 것.

### 2.4 HTTPS 바인딩

- IIS → 사이트 → 바인딩 → **https 추가**  
- 인증서(서버 인증) 선택, SNI 사용(다중도메인 시)

### 2.5 응용 프로그램 풀(App Pool) 설정 팁

- .NET CLR 버전: **무관**(in-process)
- 고급 설정:
  - **Idle Time-out**(분): 서버 자원/요구사항에 맞게
  - **Recycling**: 시간/요청수 기반 재시작 정책
- 아이덴티티: 애플리케이션 파일 접근 권한 필요한 경우 **권한 조정**

### 2.6 URL Rewrite & 리다이렉트(선택)

HTTP→HTTPS 강제, www 제거 등은 URL Rewrite 모듈로 처리.

```xml
<rewrite>
  <rules>
    <rule name="HTTPS Redirect" enabled="true" stopProcessing="true">
      <match url="(.*)" />
      <conditions>
        <add input="{HTTPS}" pattern="off" ignoreCase="true" />
      </conditions>
      <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" redirectType="Permanent" />
    </rule>
  </rules>
</rewrite>
```

### 2.7 로깅/핵심 트러블슈팅

- **Windows Event Viewer → Application**: ASP.NET Core Module 오류 확인
- `.\logs\stdout_*.log` (임시): 앱 초기화 실패 추적
- 방화벽/권한/경로/인증서/포트 충돌 점검
- `ANCM In-Process Handler Load Failure` → Hosting Bundle, VC++ 런타임, 파일 누락 확인

---

## 3. Linux + Nginx + Kestrel

### 3.1 .NET Runtime 설치(런타임형 배포 시)

```bash
wget https://dot.net/v1/dotnet-install.sh
chmod +x dotnet-install.sh
./dotnet-install.sh --channel 8.0
```

### 3.2 배포

```bash
dotnet publish -c Release -o /var/www/myapp
```

### 3.3 systemd 서비스로 데몬화

`/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My ASP.NET Core App
After=network.target

[Service]
WorkingDirectory=/var/www/myapp
ExecStart=/usr/bin/dotnet /var/www/myapp/MyApp.dll
Restart=always
RestartSec=5
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=ConnectionStrings__Default=Server=db;Database=app;User Id=app;Password=***

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
sudo systemctl status myapp
```

### 3.4 Nginx 리버스 프록시(HTTPS 권장)

```nginx
# /etc/nginx/sites-available/myapp.conf
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # 보안 헤더/제한(필요시 조정)
    add_header Strict-Transport-Security "max-age=15552000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options nosniff always;
    client_max_body_size 50m;

    location / {
        proxy_pass         http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection $connection_upgrade;
        proxy_set_header   Host $host;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/myapp.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 3.5 Let’s Encrypt 자동 발급/갱신

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com
sudo certbot renew --dry-run
```

### 3.6 ASP.NET Core에서 프록시 헤더 처리

```csharp
using Microsoft.AspNetCore.HttpOverrides;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// 프록시 인지용(원 IP/프로토콜)
var fwd = new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
};
fwd.KnownProxies.Add(System.Net.IPAddress.Parse("127.0.0.1"));
app.UseForwardedHeaders(fwd);

app.UseHttpsRedirection();
app.MapControllers();
app.Run();
```

> 필수: 그렇지 않으면 앱이 http로 인식하여 Secure 쿠키/리다이렉션 이상 동작.

---

## 4. Docker 배포(단독 컨테이너/Compose)

### 4.1 Dockerfile (멀티스테이지)

```dockerfile
# build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /out

# run
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /out .
ENV ASPNETCORE_URLS=http://0.0.0.0:8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

```bash
docker build -t myapp:latest .
docker run -d -p 8080:8080 --name myapp \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -e ConnectionStrings__Default="Server=db;..." \
  myapp:latest
```

> HTTPS는 컨테이너 외부 프록시(Nginx/Traefik, 클라우드 LB)에서 처리하는 패턴이 일반적.

### 4.2 docker-compose로 Nginx + 앱

`docker-compose.yml`:

```yaml
version: "3.9"
services:
  app:
    build: .
    container_name: app
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
    expose:
      - "8080"

  nginx:
    image: nginx:stable
    container_name: edge
    volumes:
      - ./deploy/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/letsencrypt/live/example.com:ro
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - app
```

`deploy/nginx.conf`:

```nginx
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl http2;
    server_name _;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
docker compose up -d --build
```

### 4.3 컨테이너 운영 팁

- 읽기 전용 루트, non-root 유저, 헬스체크 추가
- 로그는 **표준 출력**으로 수집(ELK/CloudWatch/Log Analytics 연계)
- 이미지 태그 전략: `:sha-<gitsha>` or SemVer + CI/CD

---

## 5. Azure App Service (PaaS)

### 5.1 장점/특징

- 운영 부담 최소화: 인프라 관리, OS 패치, LB, 인증서 바인딩, 스케일링, 슬롯 배포 제공
- CI/CD: GitHub Actions/DevOps와 **원클릭** 통합

### 5.2 기본 배포(Visual Studio/CLI)

```bash
dotnet publish -c Release -o ./publish
# Azure CLI zip 배포 예시
az webapp deploy --name <app-name> --resource-group <rg> --src-path ./publish
```

### 5.3 GitHub Actions 예시

`.github/workflows/azure.yml`:

```yaml
name: build-and-deploy
on:
  push:
    branches: [ "main" ]

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '8.x'
    - run: dotnet publish -c Release -o ./publish
    - name: Deploy
      uses: azure/webapps-deploy@v2
      with:
        app-name: 'myapp-prod'
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
        package: ./publish
```

### 5.4 슬롯(Slot) 무중단 배포

- **Staging → Production** 스왑
- 설정 스티키(슬롯 고정)로 비밀키/연결문자열 분리
- 장애시 **즉시 롤백(스왑 백)** 가능

### 5.5 App Service 구성

- **구성 → 애플리케이션 설정**: 환경 변수 주입  
- **TLS/SSL 설정**: 인증서 바인딩  
- **모니터링**: Application Insights, 로그 스트림

---

## 6. HTTPS/보안/성능 필수 체크리스트

- HTTPS 강제(리다이렉트) + HSTS(프로덕션만)
- TLS 1.2+ 제한, 약한 Cipher 제외
- 역프록시 뒤 `UseForwardedHeaders` 필수
- 업로드 크기/타임아웃 제한 (Nginx, Kestrel, App Service)
- 보안 헤더: CSP, X-Content-Type-Options, X-Frame-Options, Referrer-Policy
- 구조화 로깅(Serilog 등) + 요청 지표(성능/에러율)
- 헬스체크 `/health` → LB/Liveness/Readiness
- 비밀키/연결문자열: 환경 변수/Key Vault/Parameter Store
- GC/스레드풀/Socket 옵션(초고부하 환경) 튜닝은 **지표 기반**으로 점진 적용

---

## 7. 무중단/점진 배포(Blue-Green/Canary)

- **Nginx 업스트림 이원화**: `upstream app { server A; server B; }` 로 새 버전 가중치 조절
- Azure App Service **슬롯 스왑**: 표준 기능
- IIS **ARR/웹팜**: 로드밸런스 + 헬스체크
- 데이터 스키마 변화는 **backward-compatible** 하게(마이그레이션 → 코드 스위치 순)

---

## 8. 헬스체크/상태 페이지

### 8.1 .NET 내장 헬스체크

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("Default"), name: "db");

var app = builder.Build();
app.MapHealthChecks("/health"); // 200/503
app.Run();
```

- Nginx/Cloud LB가 `/health` 200일 때만 트래픽 라우팅

### 8.2 간단 상태 엔드포인트

```csharp
app.MapGet("/status", () =>
{
    return Results.Ok(new {
        uptime = Environment.TickCount64,
        env = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")
    });
});
```

---

## 9. 로그/모니터링 권장

- **Serilog** + Console/File/Seq/Elastic sink
- Nginx/IIS 액세스 로그와 상관관계 분석
- Azure: Application Insights, Log Analytics, Live Metrics
- 컨테이너: `docker logs`, OpenTelemetry, Elastic Stack, CloudWatch

예:

```csharp
using Serilog;

Log.Logger = new LoggerConfiguration()
  .Enrich.FromLogContext()
  .WriteTo.Console()
  .WriteTo.File("logs/app-.log", rollingInterval: RollingInterval.Day)
  .CreateLogger();

builder.Host.UseSerilog();
```

---

## 10. 트러블슈팅 모음

| 증상 | 원인 | 대처 |
|---|---|---|
| 프록시 뒤에서 https 인식 실패 | ForwardedHeaders 미적용 | `app.UseForwardedHeaders` + KnownProxies |
| 413 Request Entity Too Large | 프록시 업로드 제한 | Nginx `client_max_body_size`, IIS `maxAllowedContentLength` |
| gRPC 502/연결 실패 | HTTP/2 미구성 | Nginx `listen 443 ssl http2;` + `grpc_pass` |
| 인증서 만료 경고 | certbot 자동 갱신 실패 | `certbot renew --dry-run`, 타이머 확인 |
| IIS 500.30 | 앱 시작 실패 | Event Viewer, stdout 로그, Hosting Bundle 재확인 |
| CPU 100%/GC 문제 | 스레드/할당 과다 | 프로파일링, GC Server/Workstation, `MinThread` 조정(지표 기반) |

---

## 11. 보너스: Windows 서비스(Worker) 배포

웹이 아닌 백그라운드 서비스:

```csharp
using Microsoft.Extensions.Hosting.WindowsServices;

var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddHostedService<Worker>();

if (WindowsServiceHelpers.IsWindowsService())
{
    builder.Services.AddWindowsService(options => options.ServiceName = "MyWorker");
}

var host = builder.Build();
host.Run();
```

설치/관리: `sc create`, NSSM 또는 PowerShell `New-Service` 활용.

---

## 12. 최종 요약: 상황별 추천

| 상황 | 추천 |
|---|---|
| 사내 Windows 인프라, AD 연동, 레거시 연계 | **IIS** |
| 리눅스 VM 직접 운영, 세세한 튜닝/비용최적화 | **Kestrel + Nginx** |
| 일관성/재현성/멀티환경/멀티클라우드 | **Docker** (→ K8s 확장) |
| 빠른 PaaS, 무중단/모니터링/스케일 내장 | **Azure App Service** |

---

## 13. 실습 지향 QuickStart 레시피

### 13.1 최소 API + 헬스 + HTTPS 리다이렉트 + 프록시 헤더

```csharp
using Microsoft.AspNetCore.HttpOverrides;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddHealthChecks();
var app = builder.Build();

var fwd = new ForwardedHeadersOptions {
  ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
};
fwd.KnownProxies.Add(System.Net.IPAddress.Parse("127.0.0.1"));
app.UseForwardedHeaders(fwd);

app.UseHttpsRedirection();

app.MapGet("/", () => "OK");
app.MapHealthChecks("/health");
app.Run();
```

### 13.2 Nginx(프로덕션) 최소 설정

```nginx
server { listen 80; server_name example.com; return 301 https://$host$request_uri; }
server {
    listen 443 ssl http2; server_name example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

# 맺음말

- ASP.NET Core는 **어떤 플랫폼에도 잘 배포**된다.  
- 핵심은 **엣지 보안/HTTPS/프록시 헤더/로그/헬스/무중단** 같은 **운영 습관**이다.  
- 본 가이드의 레시피로 **IIS, Linux+Nginx, Docker, Azure** 어디서든 **실전 배포**하라.  
- 이후 단계: **Key Vault/Parameter Store**, **CDN/캐시**, **WAF/Rate Limit**, **Kubernetes 자동 복구/오토스케일**로 성숙도를 끌어올리자.