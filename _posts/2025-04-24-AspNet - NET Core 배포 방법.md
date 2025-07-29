---
layout: post
title: AspNet - NET Core 배포 방법
date: 2025-04-24 19:20:23 +0900
category: AspNet
---
# 🚀 ASP.NET Core 배포 방법 완전 정복 (IIS, Linux, Docker, Azure)

---

## 📌 개요

ASP.NET Core는 **운영체제 독립** 실행이 가능하므로  
다양한 환경에 유연하게 배포할 수 있음.

| 환경 | 설명 |
|------|------|
| **IIS** | Windows 서버 내장 웹서버 |
| **Linux + Nginx** | Kestrel 앞단에 Nginx 리버스 프록시 |
| **Docker** | 컨테이너화하여 플랫폼 독립 배포 |
| **Azure** | MS 클라우드에 원클릭 배포 지원 |

---

## 🪟 1. IIS 배포 (Windows Server)

### ✅ 환경

- Windows Server
- IIS (Internet Information Services)
- .NET Hosting Bundle 설치 필요

### 📦 준비

1. **.NET Hosting Bundle 설치**  
   [공식 다운로드](https://dotnet.microsoft.com/en-us/download/dotnet)에서 Hosting Bundle 설치

2. **ASP.NET Core 앱 Publish**

```bash
dotnet publish -c Release -o ./publish
```

3. **IIS 설정**

- IIS 관리자 실행 → 사이트 추가
- 물리 경로: `publish` 폴더 경로
- 포트 설정 (예: 5000)

4. **웹.config 생성**

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModuleV2" resourceType="Unspecified"/>
    </handlers>
    <aspNetCore processPath="dotnet" arguments="MyApp.dll" stdoutLogEnabled="true" stdoutLogFile=".\logs\stdout" hostingModel="inprocess"/>
  </system.webServer>
</configuration>
```

> 🛠 `processPath`는 `MyApp.dll` 기준으로 맞춰줘야 함

---

## 🐧 2. Linux + Nginx + Kestrel 배포

### ✅ 환경

- Ubuntu 등 Linux 서버
- .NET SDK or Runtime 설치
- Kestrel 실행 → Nginx 리버스 프록시 설정

### 📦 준비

1. **서버에 .NET Runtime 설치**

```bash
wget https://dot.net/v1/dotnet-install.sh
chmod +x dotnet-install.sh
./dotnet-install.sh --channel 8.0
```

2. **앱 배포**

```bash
dotnet publish -c Release -o /var/www/myapp
```

3. **서비스 등록 (systemd)**

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=ASP.NET Core App

[Service]
WorkingDirectory=/var/www/myapp
ExecStart=/usr/bin/dotnet MyApp.dll
Restart=always
RestartSec=10
SyslogIdentifier=myapp
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reexec
sudo systemctl enable myapp
sudo systemctl start myapp
```

4. **Nginx 설정**

```nginx
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass         http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection keep-alive;
        proxy_set_header   Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## 🐳 3. Docker 배포

### ✅ 장점

- 환경 간 일관성
- CI/CD 연동 쉬움
- 클라우드, 쿠버네티스 호환성 탁월

### 📄 `Dockerfile` 예제

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### 📦 빌드 및 실행

```bash
docker build -t myapp .
docker run -d -p 8080:80 --name myapp myapp
```

> 필요시 `docker-compose.yml`로 여러 컨테이너 구성 가능 (DB 포함)

---

## ☁️ 4. Azure App Service 배포

### ✅ 장점

- 무료 티어 있음
- GitHub Actions 등 CI/CD 통합 쉬움
- 무중단 배포/슬롯 전환 지원

### 📦 배포 방법

#### 📌 (1) Visual Studio로 배포

1. 솔루션 → 마우스 우클릭 → **"Publish"**
2. **Azure App Service 선택**
3. 자동으로 빌드 + 업로드 + 배포 완료

#### 📌 (2) CLI 배포 (zip 기반)

```bash
dotnet publish -c Release -o ./publish
az webapp deploy --name <앱이름> --resource-group <리소스그룹> --src-path ./publish
```

#### 📌 (3) GitHub Actions 자동 배포

`.github/workflows/azure.yml` 구성:

```yaml
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '8.x'
    - name: Build and Publish
      run: dotnet publish -c Release -o ./publish
    - name: Deploy to Azure WebApp
      uses: azure/webapps-deploy@v2
      with:
        app-name: '<앱이름>'
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
        package: ./publish
```

---

## 📊 배포 방식 비교

| 항목 | IIS | Linux+Nginx | Docker | Azure |
|------|-----|-------------|--------|-------|
| OS 의존성 | Windows 전용 | 리눅스 전용 | 없음 | 없음 |
| 배포 편의성 | 중 | 복잡 | 쉬움 | 매우 쉬움 |
| 확장성 | 낮음 | 중간 | 높음 | 매우 높음 |
| 자동화 | 수동 많음 | systemd 사용 | CI/CD 강력 | CI/CD 통합 |
| 운영/모니터링 | Event Viewer | 로그 직접 수집 | 도커 로그 | Azure Portal |

---

## ✅ 요약

| 목적 | 추천 배포 방식 |
|------|----------------|
| Windows 서버 내 사내 서비스 | IIS |
| 리눅스 서버 직접 운영 | Kestrel + Nginx |
| CI/CD, 클라우드 확장 | Docker |
| 빠르고 쉬운 클라우드 서비스 | Azure App Service |
