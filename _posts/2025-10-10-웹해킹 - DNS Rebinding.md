---
layout: post
title: 웹해킹 - DNS Rebinding
date: 2025-10-10 17:25:23 +0900
category: 웹해킹
---
# 21. DNS Rebinding
**— 개념 · 공격 흐름 · 안전 재현(막혀야 정상) · 방어 전략(Host 검증/인증/루프백 차단/프록시·브라우저 보조) · 스택별 코드 예제 · 인프라 구성 · 모니터링·테스트 체크리스트**

## 0) 한눈에 보기 (Executive Summary)

- **문제(정의)**  
  **DNS Rebinding**은 공격자가 **자신의 도메인**(예: `evil.tld`)을 처음엔 **공인 IP**로 응답하게 하고,  
  브라우저가 JS를 로드한 후 **같은 도메인**을 **사설/루프백 IP(예: `127.0.0.1`, `192.168.x.x`)** 로 재응답하도록 하여,  
  브라우저 **동일 출처 정책(SOP)**을 **도메인 기준으로 교란**해 **내부 서비스**에 접근·조작하도록 만드는 기법입니다.

- **핵심 조건**  
  1) 내부 서비스가 **인증 없이** 혹은 **약한 인증**으로 민감 동작 수행  
  2) 내부 서비스가 **`Host` 헤더 검증**을 하지 않음 (브라우저는 항상 `Host: evil.tld` 로 요청)  
  3) 내부 서비스가 **루프백/사설 IP**로 **외부 브라우저 트래픽**을 받아들임 (방화벽/바인딩 오구성)

- **핵심 방어**  
  - **Host/Origin 엄격 검증**: **허용 호스트 화이트리스트** 이외는 **즉시 400/421**  
  - **인증 강제**: 내부/관리 서비스는 **세션·mTLS·기기 바인딩** 등 **강한 인증** 필수  
  - **루프백/사설 네트워크 차단**: 프록시/방화벽/서비스 바인딩으로 **외부 접근 차단**  
  - **브라우저 노출 최소화**: 프레임 차단(`X-Frame-Options/COOP`), **민감 UI는 로컬 앱·VPN 전용**  
  - **프록시·DNS 보조**: `dnsmasq --stop-dns-rebind`, `unbound private-address`, RPZ, 기업 게이트웨이에서 **사설 주소 응답 차단**

---

# 1) 동작 원리(위협 모델)

1. 사용자가 `https://evil.tld` 방문 → 스크립트 로드.  
2. 공격자 DNS가 `evil.tld → 203.0.113.10`(공인) 으로 우선 응답.  
3. JS가 같은 도메인(`evil.tld`)에 AJAX/WebSocket 요청을 반복 → **두 번째 DNS 질의**가 발생.  
4. 공격자 DNS가 이번에는 **`evil.tld → 127.0.0.1` 또는 `192.168.0.1`** 로 응답(매우 짧은 TTL/즉시 재응답).  
5. 브라우저는 **도메인은 동일**하므로 SOP 허용, **IP는 내부**로 바뀐 상태에서 요청 전송.  
6. 내부 서비스(예: `http://127.0.0.1:2375` Docker API, `http://router/admin` 라우터 UI 등)가  
   **`Host: evil.tld`** 를 받아도 **검증 없이 처리**하면 조작/정보탈취 발생.

> **포인트**: 내부 서비스가 “**나에게 올 요청**은 **`Host`가 `localhost` 또는 `router.local`일 것**” 같은 가정을 깨뜨리는 것이 핵심입니다.  
> 브라우저는 항상 **도메인 문자열로 `Host` 헤더를 채우기** 때문에, 내부 서비스가 **`Host`를 검증**하지 않으면 그대로 속습니다.

---

# 2) 전형적 취약 구성(안티패턴)

- **인증 없는 내부 UI**: 라우터/프린터/개발용 콘솔이 **무인증** 또는 **약한 기본 비밀번호**.  
- **Host 검증 미구현**: `Host: *` 를 받아들이는 Dev 서버/레거시 서비스(예: 옛날 webpack dev server의 host check 비활성).  
- **0.0.0.0 바인딩 + 외부 포트 노출**: 내부 관리 서비스가 모든 인터페이스에 바인딩되어 외부 브라우저 접근 허용.  
- **프록시 오구성**: Nginx/Ingress가 **server_name** 미설정(디폴트 서버가 다 받아줌).  
- **기업 DNS/게이트웨이에서 리바인딩 보호 미사용**: 사설 IP 응답을 그대로 클라이언트에 전달.  
- **브라우저 보호 가정**: “브라우저가 IP 변조를 막아주겠지?”라는 추측. (브라우저 구현은 다양하며 **보조수단일 뿐**)

---

# 3) “안전 재현”(막혀야 정상) 테스트 아이디어

> 공격 절차가 아니라 **차단이 잘 되는지 확인**하는 **무해한** 방법입니다.

- **Host 불일치 차단**: 내부 서비스에 `Host: evil.tld` 로 접근 시 **400/421/403** 이어야 합니다.
  ```bash
  # 내부 서비스(예: 로컬 서버 127.0.0.1:8080)로 '잘못된 Host' 전송
  curl -s -D- -H "Host: evil.tld" http://127.0.0.1:8080/ | head
  # 기대: 400 Bad Request 또는 421 Misdirected Request 또는 403 Forbidden
  ```

- **Loopback 차단 확인**: 프록시/방화벽이 **외부에서 들어오는 트래픽의 루프백 대역** 접근을 차단하는지 확인.
  ```bash
  # 게이트웨이/프록시에서 루프백/사설로의 프록시 요청이 거부되는지(로그/정책) 확인
  ```

- **프록시 ‘기본 서버’ 차단**: 설정되지 않은 호스트로 요청 시 **즉시 차단**(디폴트 444/400).
  ```bash
  # 아직 등록되지 않은 호스트로 접근
  curl -s -D- http://proxy.example.com -H "Host: not-allowed.example.com" | head
  # 기대: 444/400/421
  ```

---

# 4) 방어 전략(전층)

## 4.1 애플리케이션(내부 서비스) 레벨

### A) **Host 헤더 화이트리스트**
- **원칙**: **정확히** 허용된 호스트만 200, 나머지는 400/421.  
- 공격 페이지의 요청은 항상 `Host: evil.tld` 이므로 **즉시 차단**됩니다.

**Node/Express 예**
```js
// host-guard.js
const ALLOWED = new Set(["localhost", "127.0.0.1", "admin.internal", "router.local"]);

export function hostGuard(allowlist = ALLOWED) {
  return (req, res, next) => {
    // Express에서 신뢰 프록시 설정을 하지 않았다면 req.hostname/req.headers.host 둘 다 확인
    const host = String(req.headers.host || "").split(":")[0].toLowerCase();
    if (!allowlist.has(host)) {
      return res.status(421).send("Host not allowed");
    }
    next();
  };
}

// 사용
import express from "express";
const app = express();
app.disable("x-powered-by");
app.set("trust proxy", false); // 프록시가 없다면 꺼두기
app.use(hostGuard());
// ...
```

**Go(net/http) 예**
```go
func HostGuard(allowed map[string]struct{}) func(http.Handler) http.Handler {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      host := r.Host
      if i := strings.IndexByte(host, ':'); i >= 0 { host = host[:i] }
      if _, ok := allowed[strings.ToLower(host)]; !ok {
        w.WriteHeader(421); w.Write([]byte("Host not allowed")); return
      }
      next.ServeHTTP(w, r)
    })
  }
}
```

**Spring Boot (필터)**
```java
@Component
public class HostGuardFilter extends OncePerRequestFilter {
  private static final Set<String> ALLOWED = Set.of("localhost","127.0.0.1","admin.internal","router.local");

  @Override
  protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
      throws ServletException, IOException {
    String host = Optional.ofNullable(req.getHeader("Host")).orElse("").toLowerCase();
    host = host.contains(":") ? host.substring(0, host.indexOf(':')) : host;
    if (!ALLOWED.contains(host)) {
      res.setStatus(421); res.getWriter().write("Host not allowed"); return;
    }
    chain.doFilter(req, res);
  }
}
```

### B) **강한 인증(쿠키/세션·mTLS·기기 바인딩) + CSRF**
- 내부 UI라도 **로그인/세션** 필수.  
- 상태 변경은 **CSRF 토큰 + SameSite=Lax/Strict**. (DNS Rebinding은 **같은 도메인**이지만, 내부 서비스의 세션은 보통 **없어야** 안전합니다. **있다면** CSRF까지 고려.)

**예: 쿠키 보안 속성**
```http
Set-Cookie: sid=...; Path=/; Secure; HttpOnly; SameSite=Lax
```

### C) **메서드·콘텐츠 제한**
- 위험 동작은 **POST/PUT/DELETE + Origin 확인** + **JSON/전용 MIME** 강제 → 단순 GET 폼/이미지 요청으로는 수행 불가.  
- WebSocket은 **Origin 헤더 화이트리스트** 필수.

**Express WebSocket 핸드셰이크**
```js
wss.on('connection', (ws, req) => {
  const origin = req.headers.origin;
  if (!['https://admin.internal'].includes(origin)) {
    ws.close(1008, "Origin not allowed");
  }
});
```

## 4.2 프록시/웹서버 레벨

### A) **Nginx: server_name 엄격 + 디폴트 거부**
```nginx
# 기본(미매칭) 서버: 모든 예상치 못한 Host 차단
server {
  listen 80 default_server;
  return 444;  # 연결 즉시 종료
}

# 내부 관리 UI
server {
  listen 80;
  server_name admin.internal localhost 127.0.0.1;
  # 허용된 Host 외엔 접근 불가
  if ($host !~* ^(admin\.internal|localhost|127\.0\.0\.1)$) { return 421; }

  # 외부 클라이언트(공인 IP 대역)에서 들어오면 차단 (예: 사설·루프백만 허용)
  allow 127.0.0.0/8;
  allow 10.0.0.0/8;
  allow 172.16.0.0/12;
  allow 192.168.0.0/16;
  deny all;

  # 나머지 설정...
}
```

### B) **Apache**
```apache
<VirtualHost *:80>
  ServerName admin.internal
  <IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteCond %{HTTP_HOST} !^(admin\.internal|localhost|127\.0\.0\.1)$ [NC]
    RewriteRule ^ - [F]
  </IfModule>
  Require ip 127.0.0.0/8 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16
</VirtualHost>
```

### C) **Kubernetes Ingress(NGINX)**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: admin
  annotations:
    nginx.ingress.kubernetes.io/server-snippet: |
      if ($host !~* "^(admin\.internal)$") { return 421; }
spec:
  rules:
    - host: admin.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: admin-svc, port: { number: 80 } } }
```

> **팁**: 내부 관리 UI는 **ClusterIP + Ingress 없음** 또는 **VPN 전용**으로 두는 게 이상적입니다.

## 4.3 네트워크/OS 레벨

- **서비스 바인딩**: 관리 서비스는 **`127.0.0.1`(또는 내부망 IP)만** 바인딩.  
- **방화벽**: 루프백/사설 대역으로 들어오는 **외부 기원** 트래픽 **드롭**.  
- **Docker/K8s**: Docker API(2375) **비활성** 또는 **TLS**만, K8s 대시보드 **외부 노출 금지**.

**예: Linux `ss` 로 확인**
```bash
ss -ltnp | egrep 'LISTEN|:80|:2375'
# 0.0.0.0:2375 처럼 모든 인터페이스 바인딩이면 위험
```

**UFW 예**
```bash
ufw default deny incoming
ufw allow from 192.168.0.0/16 to any port 22 proto tcp
# 내부 관리 포트는 허용 IP 대역만
ufw allow from 10.0.0.0/8 to any port 8080 proto tcp
```

## 4.4 DNS/게이트웨이(보조 방어)

- **dnsmasq**: `--stop-dns-rebind` 활성 → 사설/루프백 주소를 **응답 차단**  
  ```
  # /etc/dnsmasq.d/rebind.conf
  stop-dns-rebind
  rebind-domain-ok=/your-corp.local/
  ```
- **unbound**: `private-address: 10.0.0.0/8` 등 설정으로 사설 주소 응답 차단  
- **RPZ**: 의심 도메인을 **정책 존**으로 리다이렉트/차단  
- **EDNS Client-Subnet** 무력화를 악용하는 경우도 있으니, 기업 DNS는 **권고 설정**을 따를 것

> DNS 차단은 **보조 수단**입니다. 환경에 따라 우회 가능하므로 **서버 측 Host 검증·인증**이 **핵심**입니다.

## 4.5 브라우저/표준(보조 수단)

- 일부 브라우저/네트워크는 **Private Network Access(PNA)** 등으로  
  **공용 출처 → 사설/루프백** 접근 시 **사전 점검/차단**을 도입했습니다.  
  **그러나 전역·영구 보장은 아님**: 제품/버전/플래그에 따라 달라질 수 있으므로 **서버 측 방어가 필수**입니다.

---

# 5) 특별 케이스

- **WebSocket 내부 게이트웨이**: `Origin` 헤더 **화이트리스트** 필수, 거부 시 **1008** 코드로 종료.  
- **프린터/IoT·라우터 UI**: 대개 **Host 검증/인증 미흡** → 펌웨어/설정에서 **원격 UI 비활성** 또는 **접속망 분리**.  
- **Dev 서버(프론트 빌드툴)**: `allowedHosts`(Vite/webpack dev) **필수 설정**, `disableHostCheck` 류 옵션 **금지**.

---

# 6) 로깅·탐지

- **지표**:  
  - **비허용 Host**로 들어온 요청 수 (예: `evil.tld`, 숫자 IP 형태 등)  
  - **사설 IP/루프백 인터페이스로 들어온 외부 요청**  
  - **404/421 비율**의 급증, 특정 경로에 대한 **이상 패턴**  
- **로그 필드**: `ts`, `remote_addr`, `server_addr`, `host`, `origin`, `ua`, `x-forwarded-for`, `decision(allowed/blocked)`.

---

# 7) 테스트 자동화(“막혀야 정상”)

**Nginx 기본서버 거부**
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://proxy -H "Host: random.evil"  # 444/400/421 기대
```

**Host 검증**
```bash
# 내부 서비스에 허용되지 않은 Host로 접근 시 차단
curl -s -D- http://127.0.0.1:8080/ -H "Host: evil.tld" | head  # 4xx/421
```

**WebSocket Origin 검증**
```bash
# 올바르지 않은 Origin으로 핸드셰이크 → 연결 거절(1008)
```

**CI 파이프라인(예)**
- e2e 스펙: “허용 Host 외 접근은 4xx”, “사설 대역으로의 외부 기원 요청은 프록시에서 거부”.

---

# 8) 종합 체크리스트

- [ ] **Host 화이트리스트**(미매칭 400/421), 프록시 기본서버 444  
- [ ] **강한 인증**(내부 UI는 로그인/세션·mTLS·VPN 전용)  
- [ ] **CSRF·Origin 확인**(상태 변경 요청)  
- [ ] **루프백/사설 바인딩**(0.0.0.0 금지) + 방화벽 IP 제한  
- [ ] **WebSocket Origin** 화이트리스트  
- [ ] **Dev 서버 `allowedHosts` 설정**, 임시/테스트 인스턴스 노출 금지  
- [ ] **dnsmasq/unbound** 등에서 **rebind 보호** 활성  
- [ ] **브라우저 보조(PNA 등)**는 참고 수준—**서버 방어가 주체**  
- [ ] **로그/경보**: 비허용 Host/Origin, 사설 대역 접근 시도  
- [ ] **정기 점검**: 포트 바인딩/Ingress/라우터 UI/IoT 디바이스 노출 상태

---

# 9) 부록: 방화벽/루프백 차단 예시

**Nginx에서 사설망 원천만 허용**(내부 UI가 외부로부터 접근될 수 있는 위치에 있을 경우):
```nginx
# RFC1918 + 루프백만 허용
allow 127.0.0.0/8;
allow 10.0.0.0/8;
allow 172.16.0.0/12;
allow 192.168.0.0/16;
deny all;
```

**iptables(개념 예시)**:
```bash
# 외부 인터페이스로 들어온 패킷이 목적지 127.0.0.0/8 이면 드롭 (비대칭 환경 주의)
iptables -A INPUT -i eth0 -d 127.0.0.0/8 -j DROP
# 목적지가 사설망인데 외부 인터페이스에서 들어오면 드롭
iptables -A INPUT -i eth0 -d 10.0.0.0/8 -j DROP
iptables -A INPUT -i eth0 -d 172.16.0.0/12 -j DROP
iptables -A INPUT -i eth0 -d 192.168.0.0/16 -j DROP
```

> 실제 방화벽 정책은 **라우팅/프록시 토폴로지**에 맞춰 설계해야 합니다.

---

## 맺음말

**DNS Rebinding**은 “**도메인은 그대로, IP만 바꿔치기**”로 **SOP 가정을 붕괴**시키는 전형적인 우회 기법입니다.  
**Host 헤더 화이트리스트**, **강한 인증**, **루프백/사설망 차단**, **프록시 기본서버 거부**를 표준으로 삼으면  
환경 전반에서 이 문제를 **구조적으로 제거**할 수 있습니다.