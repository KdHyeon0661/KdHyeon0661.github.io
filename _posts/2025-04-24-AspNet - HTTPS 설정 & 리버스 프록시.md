---
layout: post
title: AspNet - HTTPS 설정 & 리버스 프록시
date: 2025-04-24 21:20:23 +0900
category: AspNet
---
# ASP.NET Core HTTPS 설정 & 리버스 프록시(Nginx, Apache)

## 큰 그림: 왜 리버스 프록시인가?

- `Kestrel`은 고성능이지만 **엣지(Edge) 보안/암호화/로깅/로드밸런싱**은 프록시(Nginx/Apache)가 더 유리
- 공통 패턴
  브라우저 ⇄ **Nginx/Apache(443/TLS)** ⇄ **Kestrel(HTTP/5000)**
  인증서/HTTPS, 압축, 캐시, Rate limit, Web Application Firewall 등을 **프록시에서 담당**

---

## 개발 환경에서 HTTPS (로컬)

### launchSettings.json 확인

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

### 개발 인증서 신뢰

```bash
dotnet dev-certs https --trust
```

- 로컬 개발 시 브라우저 경고 최소화
- 운영과 로컬을 **의식적으로 분리**(운영 인증서는 Let’s Encrypt/유료 CA)

---

## Kestrel에 직접 HTTPS 바인딩(선택)

> 운영에서는 보통 프록시에서 TLS 종료(Termination)를 하지만, **직접 바인딩도 가능**.

### Program/Host 설정

```csharp
using System.Net;
using Microsoft.AspNetCore.Server.Kestrel.Https;

var builder = WebApplication.CreateBuilder(args);

builder.WebHost.ConfigureKestrel(options =>
{
    options.Listen(IPAddress.Any, 5001, listen =>
    {
        listen.UseHttps("certs/site.pfx", "pfx-password"); // 파일 기반
        // 또는 인증서 스토어/PEM 로드도 가능
    });
});

builder.Services.AddControllers();
var app = builder.Build();
app.MapControllers();
app.Run();
```

### appsettings.json로 선언적 구성

```json
{
  "Kestrel": {
    "Endpoints": {
      "Https": {
        "Url": "https://0.0.0.0:5001",
        "Certificate": {
          "Path": "certs/site.pfx",
          "Password": "pfx-password"
        }
      }
    },
    "Limits": {
      "MaxRequestBodySize": 104857600
    }
  }
}
```

```csharp
builder.WebHost.ConfigureKestrel((ctx, opt) =>
{
    opt.Configure(ctx.Configuration.GetSection("Kestrel"));
});
```

### PEM → PFX 변환

```bash
# fullchain.pem + privkey.pem → site.pfx

openssl pkcs12 -export -out site.pfx -inkey privkey.pem -in fullchain.pem -password pass:pfx-password
```

> Windows/IIS와 달리 Linux 배포에서는 PEM을 프록시(Nginx/Apache), PFX를 Kestrel이 선호하는 경우가 많다.

---

## HTTPS 기본 보강: 리다이렉션 + HSTS

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(180);
});
var app = builder.Build();

// 80→443 강제
app.UseHttpsRedirection();

// 운영에서만 HSTS
if (!app.Environment.IsDevelopment())
    app.UseHsts();

app.MapControllers();
app.Run();
```

> HSTS는 **HTTPS 강제 캐시**이므로 실서버 도메인에서만 활성화.

---

## 리버스 프록시: Nginx 구성 (Ubuntu 기준)

### 설치

```bash
sudo apt update
sudo apt install -y nginx
```

### .NET 앱 배포/실행

```bash
dotnet publish -c Release -o /var/www/myapp
cd /var/www/myapp
dotnet MyApp.dll
```

> 실제로는 **systemd**로 서비스화하여 관리.

### Nginx 서버 블록(HTTP→HTTPS 리디렉트 + TLS + 프록시)

```nginx
# /etc/nginx/sites-available/myapp.conf

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;                # HTTP/2 활성화
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # 보안 강화 예시
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    # 운영 환경에 맞게 현대적인 cipher만 허용(가이드 참조)
    # add_header Content-Security-Policy "default-src 'self'";
    add_header Strict-Transport-Security "max-age=15552000; includeSubDomains; preload" always;

    # 클라이언트 업로드 크기 제한(필요 시)
    client_max_body_size 50m;

    location / {
        proxy_pass         http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;      # WebSocket
        proxy_set_header   Connection $connection_upgrade;
        proxy_set_header   Host $host;                 # 호스트 보존
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }
}
```

연결:

```bash
sudo ln -s /etc/nginx/sites-available/myapp.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Let’s Encrypt 발급/자동설정

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com
# 자동 인증서 설정, 80→443 리디렉션 구성

```

### 자동 갱신 확인

```bash
sudo systemctl status certbot.timer
sudo certbot renew --dry-run
```

---

## Apache 리버스 프록시

### 모듈 활성화

```bash
sudo a2enmod proxy proxy_http ssl headers proxy_wstunnel http2
sudo systemctl restart apache2
```

### VirtualHost

```apache
<VirtualHost *:80>
    ServerName example.com
    Redirect permanent / https://example.com/
</VirtualHost>

<VirtualHost *:443>
    ServerName example.com

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem
    Protocols h2 http/1.1

    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"

    ProxyPreserveHost On
    ProxyPass        /  http://127.0.0.1:5000/
    ProxyPassReverse /  http://127.0.0.1:5000/

    # WebSocket (예: /ws)
    ProxyPassMatch "^/ws/(.*)$"  "ws://127.0.0.1:5000/ws/$1"
    ProxyPassReverse "/ws/"      "ws://127.0.0.1:5000/ws/"
</VirtualHost>
```

---

## 프록시 뒤 ASP.NET Core 필수: Forwarded Headers

프록시(공인 IP) 뒤에 숨은 앱은 실제 클라이언트 정보(원 IP/프로토콜)를 프록시 헤더로 받는다.

```csharp
using Microsoft.AspNetCore.HttpOverrides;

var app = builder.Build();

// KnownProxies/Networks에 프록시 주소를 등록(보안상 중요)
var options = new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto,
    RequireHeaderSymmetry = false
};
// 예: Nginx가 같은 호스트에 있다면 127.0.0.1
options.KnownProxies.Add(System.Net.IPAddress.Parse("127.0.0.1"));

app.UseForwardedHeaders(options);
app.UseHttpsRedirection();
app.MapControllers();
app.Run();
```

> 왜 필요한가?
> - `Request.Scheme`가 https인지 정확히 인지 → URL 생성/리다이렉트/쿠키 Secure 등에 영향
> - 클라이언트 원격 IP를 로깅·제한에 정확히 활용

---

## WebSocket, gRPC, HTTP/2/3

### WebSocket

- Nginx: `proxy_set_header Upgrade` & `Connection $connection_upgrade`
- Apache: `mod_proxy_wstunnel` 사용
- ASP.NET Core: `app.UseWebSockets()` 또는 SignalR 사용 시 자동 지원

### gRPC

- gRPC는 **HTTP/2** 필요
- Nginx는 `grpc_pass` 지시어 사용

```nginx
server {
    listen 443 ssl http2;
    server_name grpc.example.com;

    ssl_certificate     /etc/letsencrypt/live/grpc.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/grpc.example.com/privkey.pem;

    location / {
        grpc_pass grpc://127.0.0.1:5001; # Kestrel이 h2에서 수신
    }
}
```

- Kestrel 쪽도 `HttpProtocols.Http2`로 리슨하거나 gRPC 템플릿 사용

### HTTP/3(QUIC, 선택)

- 최신 Nginx/Cloud 프록시가 지원
- 운영 환경 제약/방화벽 고려. 우선 HTTP/2 안정화 → HTTP/3 점진 도입

---

## 보안 헤더/강화 체크리스트

- HSTS: `Strict-Transport-Security`
- CSP(Content-Security-Policy): XSS 방어
- Referrer-Policy, X-Content-Type-Options, X-Frame-Options, Permissions-Policy
- TLS 최소버전: TLS1.2 이상
- SSL Labs로 등급 확인
- 서버 서명/버전 숨김, 디렉터리 인덱싱 금지
- Nginx `client_max_body_size`, Apache `LimitRequestBody` 등 **업로드 제한** 명시

---

## systemd 서비스로 .NET 앱 데몬화

`/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My ASP.NET Core App
After=network.target

[Service]
WorkingDirectory=/var/www/myapp
ExecStart=/usr/bin/dotnet /var/www/myapp/MyApp.dll
Restart=always
# 예: 환경변수로 연결문자열/키 주입

Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=ConnectionStrings__Default=Server=db;Database=app;User Id=app;Password=***
Environment=Urls=http://0.0.0.0:5000

# 보안 경량화(선택)

NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

활성화:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
sudo systemctl status myapp
```

---

## 방화벽/포트

- 프록시는 80/443, 앱은 5000/5001(내부)
- UFW 예시:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

---

## Docker/Compose로 배포(선택)

### Dockerfile(.NET)

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
ENV ASPNETCORE_URLS=http://0.0.0.0:5000
EXPOSE 5000
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### docker-compose.yml (Nginx + App)

```yaml
version: "3.9"
services:
  app:
    build: .
    container_name: myapp
    restart: unless-stopped

  nginx:
    image: nginx:stable
    container_name: edge
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/letsencrypt/live/example.com:ro
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - app
```

`nginx.conf`는 앞의 프록시 예시와 동일하되 `proxy_pass http://app:5000;` 처럼 **서비스명**으로 연결.

---

## 운영 체크리스트

| 항목 | 점검 |
|---|---|
| 80 → 443 리디렉션 | 적용 |
| TLS1.2/1.3 | 적용 |
| HSTS | 운영 도메인에서만 |
| CSP 등 보안 헤더 | 적용 |
| `UseForwardedHeaders` + KnownProxies | 적용 |
| 업로드/타임아웃 | 명시 |
| 로그/모니터링 | reverse proxy access/error 로그 + 앱 구조화 로깅(Serilog 등) |
| certbot 자동갱신 | 타이머 확인 |
| Blue-Green/무중단 배포 | 프록시 업스트림 교대 또는 헬스 체크 |

---

## 실전 예제 모음

### 최소 ASP.NET Core 앱(HTTPS 리다이렉트, 프록시 헤더, 헬스 체크)

```csharp
using Microsoft.AspNetCore.HttpOverrides;

var builder = WebApplication.CreateBuilder(args);

// 헬스체크
builder.Services.AddHealthChecks();
// HSTS
builder.Services.AddHsts(o =>
{
    o.IncludeSubDomains = true;
    o.MaxAge = TimeSpan.FromDays(180);
    o.Preload = true;
});

var app = builder.Build();

// 프록시 헤더
var fwd = new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
};
fwd.KnownProxies.Add(System.Net.IPAddress.Parse("127.0.0.1"));
app.UseForwardedHeaders(fwd);

if (!app.Environment.IsDevelopment())
    app.UseHsts();

app.UseHttpsRedirection();

app.MapGet("/", (HttpContext ctx) => Results.Ok(new
{
    proto = ctx.Request.Scheme,
    ip = ctx.Connection.RemoteIpAddress?.ToString()
}));

app.MapHealthChecks("/health");
app.Run();
```

### Nginx 업스트림 헬스체크(간단)

- Nginx OSS에는 **능동적 헬스체크**가 제한적. 대신 시스템 레벨 헬스/로드밸런서 사용을 권장
- 최소한 `/health` 엔드포인트로 LB 수준에서 체크하도록 구성

---

## 자주 겪는 문제와 해법

| 문제 | 원인 | 해결 |
|---|---|---|
| 앱이 http로 인식되어 쿠키 Secure/리다이렉트 오동작 | 프록시 헤더 미적용 | `UseForwardedHeaders` + KnownProxies 지정 |
| 413 Request Entity Too Large | 프록시 업로드 제한 | Nginx `client_max_body_size`, Apache `LimitRequestBody` 조정 |
| WebSocket 연결 실패 | Upgrade 헤더/Proxy 설정 누락 | Nginx `Upgrade/Connection`, Apache `mod_proxy_wstunnel` 설정 |
| gRPC 502/프록시 오류 | HTTP/2 미설정 | `listen 443 ssl http2;` + `grpc_pass` + Kestrel h2 |
| 인증서 만료 | certbot 자동 갱신 실패 | `certbot renew --dry-run` 점검, timer 활성화 |
| HTTP/2가 안 켜짐 | Nginx/Apache 빌드/모듈/ALPN 미지원 | 최신 버전/모듈 확인, OpenSSL/ALPN 지원 체크 |

---

## 마무리 요약

- 개발: `dotnet dev-certs https --trust`, 로컬 HTTPS
- 운영: **리버스 프록시에서 TLS 종료**, Kestrel은 HTTP로 단순화
- 필수: `UseForwardedHeaders`로 원 IP/프로토콜 복원
- 보안: HSTS, TLS1.2+, 보안 헤더, 업로드/타임아웃 제한
- 자동화: certbot 갱신, systemd, Docker/Compose
- 고급: WebSocket/SignalR, gRPC(HTTP/2), HTTP/3 점진 도입

---

## 부록: 각종 스니펫 모음

### A) Nginx 보안 헤더 예

```nginx
add_header X-Content-Type-Options nosniff always;
add_header X-Frame-Options DENY always;
add_header Referrer-Policy no-referrer-when-downgrade always;
add_header Permissions-Policy "geolocation=(), microphone=()" always;
# CSP는 앱 상황에 맞게 조정 필요
# add_header Content-Security-Policy "default-src 'self'; script-src 'self'; object-src 'none'" always;

```

### B) Apache 보안 헤더 예

```apache
<IfModule mod_headers.c>
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "DENY"
Header always set Referrer-Policy "no-referrer-when-downgrade"
Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains; preload"
</IfModule>
```

### C) ASP.NET Core 보안 헤더(간단)

```csharp
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers["X-Content-Type-Options"] = "nosniff";
    ctx.Response.Headers["X-Frame-Options"] = "DENY";
    await next();
});
```

---

## 다음 학습 경로 제안

- Let’s Encrypt **자동 갱신 실패 대응** 패턴(systemd timer/log 분석)
- Nginx **Zero-downtime 배포** 패턴(Blue-Green, `proxy_next_upstream`)
- **Rate limiting / WAF**(Nginx `limit_req`, ModSecurity)와 ASP.NET Core 미들웨어 조합
- gRPC-Web, HTTP/3, QUIC 기반 성능/지연 최적화

---

HTTPS 및 리버스 프록시는 “보안+확장성+운영성”의 기본 골격이다.
위 가이드를 바탕으로 **프록시에서 엣지를 단단하게** 만들고, **Kestrel은 앱 로직에 집중**시키는 구조로 운영 품질을 올리자.
