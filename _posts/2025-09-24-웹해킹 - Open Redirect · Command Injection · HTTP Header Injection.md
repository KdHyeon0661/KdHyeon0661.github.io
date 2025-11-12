---
layout: post
title: 웹해킹 - Open Redirect · Command Injection · HTTP Header Injection
date: 2025-09-24 21:25:23 +0900
category: 웹해킹
---
# Open Redirect · Command Injection · HTTP Header Injection

## 0. 큰 그림 요약

- **Open Redirect**: 공격자가 사용자를 **신뢰 도메인 → 악성 도메인**으로 자동 이동시키도록 만드는 취약점. 피싱·세션 탈취 유도·OAuth 토큰 탈취·SSRF 체인 등에 쓰임.  
- **Command Injection**: 애플리케이션이 **OS 명령**을 구성할 때 사용자 입력이 명령의 일부가 되어 **임의 명령 실행**으로 이어지는 취약점. 파일 처리기/이미지 변환기/백업 스크립트 등에서 빈번.  
- **HTTP Header Injection(응답 스플리팅/CRLF)**: 사용자 제어 데이터가 **HTTP 헤더**로 들어가 **헤더 변조**, **캐시 포이즈닝**, **Open Redirect 유발**, **추가 응답 주입** 등을 야기. `Content-Disposition`, `Location`, `Set-Cookie` 등에 특히 주의.

> 방어 공통 원리: **(1) 입력을 신뢰하지 말 것**, **(2) 컨텍스트에 맞게 안전한 API 사용**(문자열 결합 금지), **(3) 화이트리스트·정규화·재검증**, **(4) 네트워크/런타임 하드닝**(권한 최소화/egress 통제/컨테이너 격리), **(5) 로깅·탐지·자동화 테스트**.

---

# 1. Open Redirect

### 1.1 개념과 위협
- **정의**: 서버가 `?next=...` 같은 **사용자 제공 URL**을 검증 없이 `302 Location` 등으로 리디렉트.  
- **영향**:  
  - **피싱 강화**: 합법 도메인 링크 클릭 → 자동으로 악성 사이트 이동.  
  - **OAuth/Federation 체인**: redirect_uri 검증 빈틈으로 **토큰 탈취**.  
  - **SSRF 체인**: 오픈 리다이렉터를 **중간 단계**로 써서 내부로 유도.  
  - **캐시 포이즈닝**과 결합하면 **대량 피해**.

---

### 1.2 취약 예제 → 안전 예제로 고치기

#### (A) Node.js (Express)

**❌ 취약**
```javascript
// /go?next=https://evil.example
app.get("/go", (req, res) => {
  const next = req.query.next || "/";
  res.redirect(next); // 검증 없음 → 오픈 리다이렉트
});
```

**✅ 안전 1 — “상대경로만” 허용(화이트리스트)**
```javascript
import { URL } from "node:url";

app.get("/go", (req, res) => {
  const next = (req.query.next ?? "").toString();

  // 1) 절대 URL 금지(스킴/호스트 포함)  2) 스킴 상대 //evil.com 금지
  if (!next.startsWith("/") || next.startsWith("//")) return res.redirect("/");

  // 3) 내부 라우트 허용 목록(옵션)
  const ALLOW = new Set(["/dashboard", "/profile", "/help"]);
  const dest = ALLOW.has(next) ? next : "/";

  return res.redirect(dest);
});
```

**✅ 안전 2 — “특정 도메인만” 허용**
```javascript
const ALLOW_HOSTS = new Set(["app.example.com", "support.example.com"]);
app.get("/go", (req, res) => {
  const raw = req.query.next?.toString() || "/";
  try {
    const u = new URL(raw, "https://app.example.com"); // 기준(base) 제공
    // 스킴/호스트 화이트리스트
    if (u.protocol !== "https:" || !ALLOW_HOSTS.has(u.hostname)) throw new Error();
    return res.redirect(u.href);
  } catch {
    return res.redirect("/");
  }
});
```

#### (B) Python (Flask)

**❌ 취약**
```python
@app.get("/go")
def go():
    nxt = request.args.get("next", "/")
    return redirect(nxt)  # 절대 URL, //, javascript: 우회 가능
```

**✅ 안전**
```python
from urllib.parse import urlparse, urljoin
ALLOW_HOSTS = {"app.example.com", "support.example.com"}

def is_safe_url(target: str) -> bool:
    base = urlparse("https://app.example.com")
    test = urlparse(urljoin(base.geturl(), target))
    return (
        test.scheme == "https" and
        test.netloc in ALLOW_HOSTS and
        test.path.startswith("/")
    )

@app.get("/go")
def go():
    nxt = request.args.get("next", "/")
    if not is_safe_url(nxt): nxt = "/"
    return redirect(nxt)
```

#### (C) Spring Boot

**❌ 취약**
```java
@GetMapping("/go")
public String go(@RequestParam String next, HttpServletResponse res){
  return "redirect:" + next; // 문자열 결합 리다이렉트
}
```

**✅ 안전**
```java
private static final Set<String> ALLOW = Set.of("/home","/settings","/help");

@GetMapping("/go")
public String go(@RequestParam(required=false) String next){
  if (next == null || !next.startsWith("/") || next.startsWith("//")) return "redirect:/";
  if (!ALLOW.contains(next)) return "redirect:/";
  return "redirect:" + next;
}
```

---

### 1.3 테스트 & 방어 팁
- **차단해야 할 입력**:  
  - 절대 URL: `https://attacker.tld/x`  
  - 스킴 상대: `//attacker.tld/x`  
  - 데이터/자바스크립트 스킴: `javascript:alert(1)`, `data:text/html,...`  
  - 중간 오픈 리다이렉트 체인: `/redirect?next=https://evil`  
- **정책**:  
  - 가능하면 **리디렉트 목적지 자체를 입력받지 않기**(고정 라우트로 설계).  
  - 필요 시 **짧은 키 → 목적지 매핑**을 서버가 관리.  
  - OAuth는 **정확 일치(Exact Match)** 기반으로 등록, **와일드카드** 지양.

---

# 2. Command Injection

### 2.1 개념과 위협
- **정의**: 애플리케이션이 `tar`, `convert`, `ffmpeg`, `grep` 같은 **시스템 명령**을 **문자열로 조립**할 때 사용자 입력이 **옵션/연산자/쉘 메타문자**로 해석되어 **임의 명령 실행**으로 이어짐.  
- **일반 표면**: 썸네일 생성, 압축/백업, 바이러스 스캔 래퍼, PDF/오디오 변환, 관리자 콘솔에서 제공하는 “진단 명령” 등.  
- **영향**: 원격 코드 실행(RCE), 파일 유출/변조, 크리덴셜 탈취, lateral movement.

> **핵심 철칙**: **“문자열로 쉘 명령을 만들지 말고”** 가능하면 **라이브러리/네이티브 API**를 사용하거나, **명령 호출 시 argv 배열**을 사용해 **shell을 완전히 비활성화**하라.

---

### 2.2 취약 예제 → 안전 예제로 고치기

#### (A) Node.js

**❌ 취약**
```javascript
import { exec } from "node:child_process";
// 사용자가 업로드한 파일을 리사이즈 (취약: 문자열 보간)
app.post("/resize", (req, res) => {
  const { src, size } = req.body; // 예: src="x.png; echo H", size="100x100"
  exec(`convert ${src} -resize ${size} out.png`, (err) => {
    if (err) return res.status(500).send("fail");
    res.send("ok");
  });
});
```

**✅ 안전**
```javascript
import { spawn } from "node:child_process";
import path from "node:path";
function isValidSize(s){ return /^[1-9][0-9]{0,3}x[1-9][0-9]{0,3}$/.test(s); }

app.post("/resize", (req, res) => {
  const { src, size } = req.body;
  // 1) 파일은 서버가 지정한 디렉터리 내 UUID로만 접근(경로 화이트리스트)
  const SAFE_DIR = "/var/app/uploads";
  const safeSrc = path.join(SAFE_DIR, path.basename(src || ""));
  if (!isValidSize(size)) return res.status(400).send("bad size");

  // 2) shell=false (spawn 기본) + argv 배열
  const p = spawn("convert", [safeSrc, "-resize", size, "out.png"], { stdio: "ignore" });
  p.on("exit", (code) => code === 0 ? res.send("ok") : res.status(500).send("fail"));
});
```

#### (B) Python

**❌ 취약**
```python
import os
# 진단 페이지에서 핑 테스트(취약)
def ping(host):
    return os.popen(f"ping -c 1 {host}").read()
```

**✅ 안전**
```python
import subprocess, shlex
def safe_ping(host: str):
    # 1) 호스트 형식 제한(간단 예: FQDN/IPv4만)
    import re
    if not re.fullmatch(r"[A-Za-z0-9.-]{1,253}", host): raise ValueError("bad host")
    # 2) shell=False + argv 배열
    out = subprocess.run(["ping", "-c", "1", host], capture_output=True, text=True, timeout=3, check=False)
    return out.stdout if out.returncode == 0 else ""
```

#### (C) PHP

**❌ 취약**
```php
<?php
// /zip?path=/var/app/data  → "tar -czf out.tgz /var/app/data"
$path = $_GET["path"];
echo shell_exec("tar -czf out.tgz " . $path); // 메타문자 주입 가능
```

**✅ 안전(라이브러리 우선)**
```php
<?php
// 1) OS 명령 대신 ZipArchive 등 라이브러리를 사용
$path = $_GET["path"] ?? "";
if (!preg_match('/^[A-Za-z0-9._-]{1,64}$/', $path)) { http_response_code(400); exit; }
$src = "/var/app/data/" . $path;
// ZipArchive를 이용해 안전하게 압축(샌드박스 디렉터리 내부만)
$zip = new ZipArchive();
$zip->open("/var/app/out.zip", ZipArchive::CREATE | ZipArchive::OVERWRITE);
$zip->addFile($src, basename($src));
$zip->close();
echo "ok";
```

#### (D) Java

**❌ 취약**
```java
String cmd = "ffmpeg -i " + input + " -b:v " + bitrate + " out.mp4";
Runtime.getRuntime().exec(cmd); // 문자열 결합
```

**✅ 안전**
```java
List<String> cmd = List.of("ffmpeg", "-i", input, "-b:v", bitrate, "out.mp4");
ProcessBuilder pb = new ProcessBuilder(cmd);
pb.redirectErrorStream(true);
Process p = pb.start(); // 쉘 미사용
```

---

### 2.3 방어 체크리스트(코드 + 운영)
- **코드 레벨**  
  - [ ] **shell 사용 금지**: `exec()`/`system()`/`os.popen()`/백틱 등 피하기.  
  - [ ] **argv 배열 호출**(Node `spawn`, Python `subprocess.run([...], shell=False)`, Java `ProcessBuilder`).  
  - [ ] **입력 화이트리스트**: 예상 형식(숫자 범위/크기/파일명/호스트/옵션)만 통과.  
  - [ ] **OS 명령 대신 라이브러리** 사용(예: 이미지/압축/네트워크 작업).  
  - [ ] **타임아웃/리소스 제한**: 무한 실행·대용량 처리를 제한(cgroup/ulimit).  
  - [ ] **작업 디렉터리/권한 샌드박스**: 전용 유저, chroot/컨테이너, 읽기 전용 마운트, 최소 권한.
- **운영/아키텍처**  
  - [ ] **egress 통제**(프록시/방화벽)로 임의 외부 접속 제한.  
  - [ ] **감사 로깅**: 명령 실행 경로/인자/실패율 모니터링(비정상 스파이크 경보).  
  - [ ] **서드파티 툴 최신화**(ffmpeg/ImageMagick 등 RCE 이슈 대응).

---

# 3. HTTP Header Injection (CRLF / Response Splitting / Host Header)

### 3.1 개념과 위협
- **정의**: HTTP 응답 헤더에 사용자 입력이 들어가 `\r\n`(CRLF) 삽입 → **새 헤더** 또는 **두 번째 응답** 주입(**응답 스플리팅**).  
- **영향**:  
  - **캐시 포이즈닝**(프록시/중간 캐시 오염)  
  - **Open Redirect 유발**(`Location` 조작)  
  - **쿠키/콘텐츠 유형 조작**(XSS/다운로드 강제)  
  - **호스트 헤더 주입**과 결합 시 **링크/비밀번호 재설정 URL 오염**, SSRF·내부 호출 교란.

---

### 3.2 취약 예제 → 안전 예제로 고치기

#### (A) Location 헤더에 사용자 입력

**❌ 취약 (Node.js)**
```javascript
// /jump?to=/home
app.get("/jump", (req, res) => {
  const to = String(req.query.to || "/");
  // 공격자 입력: "/home\r\nX-Evil: 1" → 새 헤더 삽입 시도
  res.set("Location", to);
  res.status(302).end();
});
```

**✅ 안전**
```javascript
function sanitizeHeaderValue(v){
  if (/[\r\n]/.test(v)) throw new Error("CRLF");
  return v;
}
app.get("/jump", (req, res) => {
  let to = (req.query.to ?? "/").toString();
  // Open Redirect 방어(상대경로만)
  if (!to.startsWith("/") || to.startsWith("//")) to = "/";
  res.set("Location", sanitizeHeaderValue(to));
  res.status(302).end();
});
```
> **참고**: 최신 런타임/프레임워크는 헤더에 `\r\n`이 들어가면 예외를 던지는 경우가 많지만, **프록시/레거시/서드파티**가 끼면 여전히 위험. **항상** CR/LF 금지 + 리디렉트 방어를 병행.

#### (B) Content-Disposition filename 인젝션

**❌ 취약**
```javascript
app.get("/download", (req, res) => {
  const name = req.query.name || "file.txt";
  // name="x.txt\"\r\nX-Injected: 1\r\n\r\n<body>evil"
  res.set("Content-Disposition", `attachment; filename="${name}"`);
  res.sendFile(`/var/data/${name}`);
});
```

**✅ 안전 (RFC 5987 인코딩)**
```javascript
import path from "node:path";
function safeFilename(name) {
  // 허용 문자만, 길이 제한
  if (!/^[A-Za-z0-9._-]{1,64}$/.test(name)) return "file.txt";
  return name;
}
function rfc5987Encode(val){
  // 간단 버전(실무에선 검증된 유틸 사용 권장)
  return encodeURIComponent(val).replace(/'/g, "%27").replace(/\*/g, "%2A");
}
app.get("/download", (req, res) => {
  const raw = (req.query.name ?? "file.txt").toString();
  const name = safeFilename(raw);
  res.set("X-Content-Type-Options", "nosniff");
  res.set("Content-Disposition",
    `attachment; filename="${name}"; filename*=UTF-8''${rfc5987Encode(name)}`);
  res.sendFile(path.join("/var/data", name));
});
```

#### (C) Host Header 주입(절대 URL 생성)

**❌ 취약 (비밀번호 재설정 링크)**
```python
# ex) https://app.example/reset?token=...
# 프록시 뒤에서 Host 헤더를 그대로 신뢰
def make_reset_url(req, token):
    return f"{req.scheme}://{req.headers['Host']}/reset?token={token}"
```

**✅ 안전**
```python
ALLOWED_HOSTS = {"app.example", "app.example.com"}  # 프레임워크 설정으로 관리 권장
def make_reset_url(req, token):
    host = req.headers.get("Host","")
    if host not in ALLOWED_HOSTS: host = "app.example.com"
    return f"https://{host}/reset?token={token}"
```
- **프록시 환경**에서는 **프레임워크/리버스 프록시 설정**(예: Django `ALLOWED_HOSTS`, Express `trust proxy`, Nginx `proxy_set_header Host`)을 정확히 구성. 외부에서 임의 `Host`/`X-Forwarded-Host`를 전달해도 **허용 목록** 외는 거부/정규화.

---

### 3.3 방어 체크리스트
- [ ] 모든 **헤더 값에서 CR(`\r`), LF(`\n`) 금지**(전역 미들웨어).  
- [ ] `Location`/`Content-Disposition`/`Content-Type`/`Set-Cookie` 등 **핵심 헤더**는 **안전 빌더/유틸** 사용.  
- [ ] **Open Redirect 방어**와 **헤더 인젝션 방어**를 함께 적용(동일 입력이 양쪽에 쓰이는 경우 많음).  
- [ ] **Host/Forwarded 헤더 정규화**: 프록시 신뢰 경계 명확화, **허용 호스트**만 절대 URL 생성에 사용.  
- [ ] **캐시 헤더**(Vary/Cache-Control) 오용 금지, 캐시 앞단에서 **응답 스플리팅** 탐지/차단.

---

# 4. 공통 “안전 설계” 패턴

### 4.1 입력 처리 3원칙
1) **화이트리스트**: 예상되는 값만 통과(도메인/경로/파일명/숫자 범위 등).  
2) **정규화 후 비교**: URL/경로/호스트는 **정규화(canonicalization)** 후 검증.  
3) **컨텍스트별 API**: URL은 **파서/빌더**, 헤더는 **유효성 있는 setter**, 명령은 **argv**.

### 4.2 운영/아키텍처 하드닝
- **네트워크 분리/egress 통제**: SSRF·오픈 리다이렉트 체인 영향 축소.  
- **컨테이너/샌드박스**: 명령 실행/파일 접근 범위 최소화, 읽기 전용 마운트, `noexec` 마운트.  
- **권한 최소화**: 애플리케이션/유틸리티 사용자 권한 축소.  
- **로깅/탐지**: `\r\n` 포함 요청, 비정상 리다이렉트, 명령 실패율/시간 초과 스파이크 알림.

---

# 5. 프레임워크별 힌트(요약)

- **Express/Node**:  
  - `res.redirect()` 목적지는 반드시 **상대 경로 또는 화이트리스트 도메인**만 허용.  
  - 헤더는 프레임워크가 일부 검증하더라도 **추가 CRLF 검증**을 권장.  
  - 외부 명령 대신 Node 표준 라이브러리(예: `fs`, `zlib`, Sharp 등) 사용.

- **Flask/Django**:  
  - Django `ALLOWED_HOSTS`/`SECURE_PROXY_SSL_HEADER` 설정.  
  - `redirect()`에 전달하는 URL은 **검증 후** 사용.  
  - `send_file` 헤더(`Content-Disposition`)는 안전 유틸 사용.

- **Spring**:  
  - `redirect:` 사용 시 **상대경로 제한**, 절대 URL 금지.  
  - 비밀번호 리셋/링크 생성 시 **허용 호스트** 기반.  
  - 외부 명령은 `ProcessBuilder` + 검증 + 라이브러리 대체.

- **PHP**:  
  - `header()`에 사용자 입력을 전달하지 않기(파일명도 검증/인코딩).  
  - OS 명령 대신 확장/라이브러리 사용(ZipArchive, GD, Imagick 등).  
  - 리디렉트는 내부 경로만 허용.

---

# 6. “스모크 테스트” 예시(개발/QA 전용)

### 6.1 Open Redirect
- 입력:  
  - `?next=https://evil.tld/p` (절대 URL) → **/ 로 리디렉트**  
  - `?next=//evil.tld` (스킴 상대) → **/ 로 리디렉트**  
  - `?next=/profile` (허용 목록) → **/profile 로 리디렉트**

### 6.2 Command Injection
- 입력:  
  - 파일명에 공백/세미콜론/`&` 포함 → **거부**  
  - `size="100x100"`은 OK, `size="100x100;..."` → **거부**  
- 실행: **타임아웃**과 **리소스 제한**이 동작하는지 확인.

### 6.3 Header Injection
- 입력:  
  - `name="x\r\nX-Test: 1"` → **400 또는 필드 정규화로 제거**  
  - `to="/home\r\n\r\n<html>"` → **리디렉트 실패/차단**

---

# 7. “실전 시나리오” — 세 공격이 이어지는 체인

1) 공격자는 **오픈 리다이렉트**를 이용해 합법 도메인 링크를 클릭한 사용자들을 **피싱 페이지**로 유도.  
2) 피싱 페이지에서 쿠키/OTP를 가로채거나, 브라우저를 대상 앱의 **취약 업로드/변환 엔드포인트**로 몰아 **Command Injection**을 트리거.  
3) 결과 응답을 **HTTP Header Injection**으로 캐시 포이즈닝하여 **대량 사용자에게 악성 스크립트/리디렉트**를 전파.  
→ 실제로는 XSS/SSRF 등 다른 취약점과 얽히며, **다층 방어** 없는 단일 대책은 쉽게 우회됩니다.

---

# 8. 보안 점검 체크리스트(요약)

- **Open Redirect**  
  - [ ] 리디렉트 목적지 입력 **금지** 또는 **화이트리스트**  
  - [ ] 절대 URL/스킴 상대/`javascript:`/`data:` **거부**  
  - [ ] OAuth redirect_uri **정확 일치**만 허용

- **Command Injection**  
  - [ ] **shell 호출 금지**, **argv 배열**로만 실행  
  - [ ] 라이브러리로 대체(압축/이미지/네트워크 등)  
  - [ ] 입력 형식·범위 검증, 타임아웃·리소스 제한, 샌드박스·권한 최소화

- **HTTP Header Injection**  
  - [ ] 헤더 값에서 **CR/LF 금지**  
  - [ ] `Content-Disposition`/`Location`/`Set-Cookie` 안전 유틸 사용  
  - [ ] Host/Forwarded 헤더 **정규화·허용 목록**, 프록시 신뢰 경계 명확화

- **운영/거버넌스**  
  - [ ] 코드 리뷰 규칙: 문자열로 명령/헤더/URL을 **조립 금지**  
  - [ ] SAST/DAST 규칙 활성(오픈 리다이렉트/CRLF/명령 실행 API 탐지)  
  - [ ] 보안 헤더(HSTS, X-Content-Type-Options, CSP)와 캐시 정책 점검  
  - [ ] 로그/알림: 비정상 리디렉트/CRLF/명령 실패율/장시간 실행 감시

---

# 9. 부록 — 미니 유틸/스니펫 모음

### 9.1 “상대 경로만” 허용 유틸 (Node)
```javascript
export function onlyInternalPath(input, fallback="/"){
  if (typeof input !== "string") return fallback;
  if (!input.startsWith("/") || input.startsWith("//")) return fallback;
  // 경로 정규화(중복 슬래시/.. 제거)
  const cleaned = input.replace(/\/{2,}/g,"/").replace(/\/\.(?=\/|$)/g,"").replace(/\/[^/]+\/\.\.(?=\/|$)/g,"/");
  return cleaned || fallback;
}
```

### 9.2 헤더 값 검증(공용)
```javascript
export function headerSafe(v){
  if (typeof v !== "string") throw new Error("type");
  if (v.includes("\r") || v.includes("\n")) throw new Error("CRLF");
  return v;
}
```

### 9.3 Python — 파일명/경로 화이트리스트
```python
import re, pathlib
SAFE_RE = re.compile(r"^[A-Za-z0-9._-]{1,64}$")
def safe_filename(name: str) -> str:
  return name if SAFE_RE.fullmatch(name or "") else "file.txt"
def safe_join(root: pathlib.Path, name: str) -> pathlib.Path:
  p = (root / safe_filename(name)).resolve()
  if not str(p).startswith(str(root)): raise ValueError("Traversal")
  return p
```

### 9.4 Spring — 절대 URL 거부 헬퍼
```java
public static String safeRedirect(String next){
  if (next == null || !next.startsWith("/") || next.startsWith("//")) return "/";
  Set<String> allow = Set.of("/home","/me","/help");
  return allow.contains(next) ? next : "/";
}
```

---

## 맺음말
세 취약점은 **겉으로 단순해 보이지만**, 실무에서는 **프록시·클라우드·서드파티 도구**와 얽혀 사고로 번지기 쉽습니다.  
가장 안전한 지름길은 **문자열 결합 금지**, **화이트리스트 중심 설계**, **표준 라이브러리/검증된 유틸 사용**, **네트워크/런타임 하드닝**입니다.  
그리고 무엇보다, **자동화 테스트**로 “막혀야 정상”인 케이스를 꾸준히 검증하세요.