---
layout: post
title: AspNet - HTTPS 설정 & 리버스 프록시
date: 2025-04-24 21:20:23 +0900
category: AspNet
---
# 🔐 ASP.NET Core HTTPS 설정 & 리버스 프록시 (Nginx, Apache) 구성하기

---

## ✅ 1. HTTPS란?

**HTTPS (HyperText Transfer Protocol Secure)**  
→ HTTP + SSL/TLS를 결합한 **암호화된 통신 방식**

- 사용자 정보 보호
- 데이터 무결성 보장
- 신뢰성 인증 (도메인 + 인증서 기반)

> ASP.NET Core는 기본적으로 `Kestrel` 서버에서 HTTPS 지원을 내장하고 있음

---

## 📦 2. ASP.NET Core HTTPS 설정

### 🔹 개발 환경에서 (localhost)

`.csproj` 프로젝트 생성 시 자동 생성된 `launchSettings.json` 참고:

```json
"profiles": {
  "MyApp": {
    "commandName": "Project",
    "dotnetRunMessages": true,
    "launchBrowser": true,
    "applicationUrl": "https://localhost:5001;http://localhost:5000",
    "environmentVariables": {
      "ASPNETCORE_ENVIRONMENT": "Development"
    }
  }
}
```

### 🔹 인증서 생성 (개발용)

```bash
dotnet dev-certs https --trust
```

- 로컬 머신에 개발용 인증서가 설치됨
- 브라우저에 `신뢰된 인증서`로 등록됨

---

## 🏗 3. Kestrel 직접 HTTPS 설정 (생산용)

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.ConfigureKestrel(serverOptions =>
            {
                serverOptions.Listen(IPAddress.Any, 5001, listenOptions =>
                {
                    listenOptions.UseHttps("cert.pfx", "password");
                });
            });
            webBuilder.UseStartup<Startup>();
        });
```

> 🔐 인증서는 `.pfx` 형식의 **SSL 인증서** 파일 필요  
> 유료 인증서 or Let's Encrypt 무료 인증서 사용 가능

---

## 🌐 4. 리버스 프록시란?

리버스 프록시는 **클라이언트 요청을 중간에서 받아 백엔드 서버로 전달하는 서버** 역할

| 구성도 |
|--------|
| 브라우저 → [Nginx/Apache] → Kestrel (.NET 앱) |

- Kestrel은 고성능이지만 직접 HTTPS 관리에 불리
- 프록시가 인증서/보안/부하 분산 담당

---

## 🧰 5. Nginx + Kestrel + HTTPS 설정 (Ubuntu 기준)

### ▶ 1) Nginx 설치

```bash
sudo apt update
sudo apt install nginx
```

---

### ▶ 2) Kestrel 앱 실행

```bash
dotnet publish -c Release -o /var/www/myapp
cd /var/www/myapp
dotnet MyApp.dll
```

또는 systemd 등록해서 백그라운드 실행

---

### ▶ 3) Nginx 설정 (리버스 프록시 + HTTPS)

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

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

### ▶ 4) Let’s Encrypt SSL 인증서 발급

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d example.com
```

> 자동으로 인증서 설정 + HTTPS 리디렉션까지 설정됨

---

## 🧰 6. Apache + Kestrel 구성

1. Apache에서 `mod_proxy`, `mod_ssl` 활성화
2. 다음과 같이 `VirtualHost` 구성:

```apache
<VirtualHost *:443>
    ServerName example.com

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem

    ProxyPreserveHost On
    ProxyPass / http://localhost:5000/
    ProxyPassReverse / http://localhost:5000/
</VirtualHost>
```

---

## 📋 7. appsettings.json에서 Kestrel HTTPS 설정 (선택적)

```json
{
  "Kestrel": {
    "Endpoints": {
      "Https": {
        "Url": "https://0.0.0.0:5001",
        "Certificate": {
          "Path": "certs/cert.pfx",
          "Password": "1234"
        }
      }
    }
  }
}
```

Program.cs에 설정:

```csharp
builder.WebHost.ConfigureKestrel((context, options) =>
{
    options.Configure(context.Configuration.GetSection("Kestrel"));
});
```

---

## ✅ 8. 실무 주의사항

| 항목 | 설명 |
|------|------|
| 인증서 갱신 | Let's Encrypt는 90일마다 자동 갱신 필요 |
| HSTS 적용 | `Strict-Transport-Security` 헤더로 HTTPS 강제 |
| 포트 충돌 | Kestrel과 Nginx 포트 겹치지 않도록 주의 |
| 보안 테스트 | [SSL Labs](https://www.ssllabs.com/ssltest/) 등으로 보안 점수 확인 |
| 리버스 프록시 설정 | 반드시 `UseForwardedHeaders()` 적용 필요 |

---

## 🛡️ 9. UseForwardedHeaders 적용 (ASP.NET Core 내부에서)

리버스 프록시 뒤에 있을 경우, 클라이언트 IP 등 정보를 정확히 받기 위해 아래 추가:

```csharp
app.UseForwardedHeaders(new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
});
```

---

## ✅ 요약

| 항목 | 내용 |
|------|------|
| HTTPS 사용 이유 | 데이터 암호화, 무결성 보장, 신뢰성 확보 |
| 개발용 인증서 | `dotnet dev-certs https --trust` |
| 운영 환경 | 인증서(pfx or pem) 필수 |
| 리버스 프록시 | Nginx/Apache가 HTTPS 처리, Kestrel은 HTTP |
| 자동 HTTPS | Let's Encrypt + Certbot + Nginx 연동 |
| 내부 설정 보완 | `UseForwardedHeaders`, Kestrel 수동 구성 등 |

---

## 🔜 다음 추천 주제

- ✅ Let's Encrypt 인증서 자동 갱신 스크립트 구성
- ✅ Kestrel + Nginx + Docker 연동 배포 구성
- ✅ HTTPS 보안 설정 고급 옵션 (TLS 버전 제한, HSTS, CSP)

---

HTTPS는 **웹의 기본 보안 요소**이며,  
Nginx나 Apache 같은 프록시 서버를 통해 **유연하고 안전한 배포 구성이 가능**해요.  
보안 요구사항이 높을수록 프록시와 HTTPS 구성을 반드시 도입해야 합니다.