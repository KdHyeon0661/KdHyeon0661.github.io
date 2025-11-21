---
layout: post
title: 웹해킹 - Directory Traversal / Path Traversal
date: 2025-09-24 19:25:23 +0900
category: 웹해킹
---
# Directory Traversal / Path Traversal

## 무엇이 Directory Traversal(=Path Traversal)인가?

**Directory Traversal(경로 조작)**은 사용자 입력에 포함된 경로 요소(`../`, `..\`, 절대경로, 이중 인코딩 등)를 악용해 **애플리케이션이 의도한 루트 밖의 파일**을 읽거나 쓰게 만드는 취약점입니다. 대표적으로 `/etc/passwd`, 앱의 `application.yml`, 데이터베이스 크리덴셜, 소스 코드, 키 파일 등이 노출됩니다. 또한 아카이브 해제(Zip)·업로드 처리기 등에서도 동일 원리로 **임의 위치 쓰기**가 발생합니다(Zip Slip).

> 결과적으로 공격자는 **민감 정보 열람 → 추가 침투(크리덴셜/키 재사용, RCE 체인)**로 이어갈 수 있습니다. 실무 가이드와 예제 랩은 PortSwigger/OWASP 문서를 참조하세요.

---

## 왜 발생하나? (취약 패턴 한눈에)

- **사용자 입력을 경로에 직접 삽입**: `?file=../../../../etc/passwd` 같은 값을 그대로 파일 API에 전달
- **인증 후에도 루트 밖 접근 검증 부재**: “로그다운로드”/“이미지 미리보기” 같은 기능
- **인코딩/OS 구분자 우회**: `%2e%2e%2f`, `%252e%252e%252f`, `..\`, 혼합 구분자(`..%2f..%5c`), 유니코드/Fullwidth `％2e` 등
- **절대경로/심볼릭 링크/단축 경로(Windows)** 우회: `C:\Windows\System32\drivers\etc\hosts`, `\\\\?\C:\...`, `\\server\share`, `file:////`
- **Zip 해제/아카이브 처리**: ZIP 항목 이름이 `../../app.war` 등인 경우 해제 시 상위 디렉터리로 탈출(Zip Slip)

---

## 공격 시나리오(예제 중심)

### “로그 보기” 기능에서 시스템 파일 유출

**❌ 취약(Express/Node.js)**
```javascript
// /view?name=app.log  → 그대로 파일을 읽음
import fs from "node:fs/promises";
import path from "node:path";
app.get("/view", async (req, res) => {
  const name = req.query.name; // 공격자: ../../../../etc/passwd
  const full = path.join("/var/app/logs", name); // join만으로는 불충분
  const txt = await fs.readFile(full, "utf8");    // 외부 파일까지 읽힘
  res.type("text/plain").send(txt);
});
```

**공격 예**
`/view?name=../../../../etc/passwd` → 운영체제 계정 목록이 노출.
`/view?name=../config/application.yml` → DB 패스워드/토큰 유출.
→ 이후 SSH/DB 접속·서버 내 평문 키 파일 검색 등 2차 침투로 확장.

---

### “이미지 미리보기”의 경로 조작

**❌ 취약(Flask)**
```python
from flask import Flask, request, send_file
app = Flask(__name__)

@app.get("/img")
def img():
    # 예: /img?file=avatars/1.png
    f = request.args.get("file","")
    return send_file(f)  # web root 기준이 아님 → 어디든 읽힘
```

**공격 예**
`/img?file=../../../../etc/ssh/sshd_config` → SSH 설정 유출.

---

### Zip Slip(아카이브 해제 시 Path Traversal)

**❌ 취약(Java)**
```java
try (ZipInputStream zis = new ZipInputStream(uploadedStream)) {
  ZipEntry e;
  Path destDir = Paths.get("/var/app/uploads");
  while ((e = zis.getNextEntry()) != null) {
    Path out = destDir.resolve(e.getName()); // normalize 없음
    Files.createDirectories(out.getParent());
    Files.copy(zis, out, StandardCopyOption.REPLACE_EXISTING); // ../../ 탈출 허용
  }
}
```

**공격 예**
ZIP 내부 항목 이름: `../../../../tomcat/webapps/ROOT/ROOT.jsp` → **웹쉘 쓰기**.
이 취약점은 “**Zip Slip**”으로 대중화된 아카이브 기반 경로조작입니다. 다수 프로젝트에 영향을 준 전례가 있으니 반드시 방어 패턴을 적용하세요.

---

## 방어 원칙(핵심 6가지)

1) **가능하면 파일명/경로를 직접 받지 말 것**: **ID 기반 매핑(화이트리스트)**
   - DB의 `files(id, stored_name, owner_id, base_dir)`에서 `id`로 조회 후 **서버가 보유한 안전한 경로**만 사용
2) **경로 표준화(canonicalization)** 후 **루트 디렉터리 내부인지 검사**
   - `resolve/normalize/realpath` → **루트 시작 여부**(`startsWith`, `is_relative_to`) 확인
3) **허용 목록**: 경로 문자/확장자/최대 길이 제한(영숫자, `_-.` 중심)
4) **심볼릭 링크/하드 링크/절대경로 거부**(+ Windows UNC/디바이스 경로)
5) **아카이브 해제**: 항목 이름을 **normalize + startsWith 검사**, 압축 내 심볼릭 링크 거부
6) **다운로드 전용 헤더 & MIME 스니핑 방지**: `Content-Disposition: attachment`, `X-Content-Type-Options: nosniff`로 브라우저 렌더링 억제(콘텐츠 하이재킹 연계 차단)

> **정리**: “**ID→서버가 아는 경로**”가 최선입니다. 불가피하게 경로를 받는다면 **정규화→루트 내부 검증**을 필수로 적용하세요.

---

## 취약 → 안전: 언어/프레임워크별 레시피

### — 안전한 파일 다운로드

**✅ 권장 1: ID 매핑**
```javascript
// DB: files(id, stored_name, base_dir)
app.get("/files/:id", async (req, res) => {
  const row = await db.get("SELECT stored_name, base_dir FROM files WHERE id=?", [req.params.id]);
  if (!row) return res.sendStatus(404);
  const base = path.resolve(row.base_dir);
  const full = path.resolve(base, row.stored_name); // 서버가 지정한 안전 이름
  if (!full.startsWith(base + path.sep)) return res.sendStatus(403);
  res.setHeader("X-Content-Type-Options","nosniff"); // 렌더링 억제
  res.setHeader("Content-Disposition", `attachment; filename="${path.basename(full)}"`);
  return res.sendFile(full);
});
```

**✅ 권장 2: 경로 입력을 받는 경우(정규화·루트검사)**
```javascript
const SAFE_ROOT = path.resolve("/var/app/reports");

function safeJoin(root, unsafePath) {
  const resolved = path.resolve(root, unsafePath); // normalize + absolute
  if (!resolved.startsWith(root + path.sep)) throw new Error("Traversal");
  return resolved;
}

app.get("/report", async (req, res) => {
  const name = req.query.name || "";               // 예: 2025-09-logs.txt
  if (!/^[\w.\-]{1,64}$/.test(name)) return res.sendStatus(400); // 문자 제한
  const file = safeJoin(SAFE_ROOT, name);
  res.setHeader("X-Content-Type-Options","nosniff");
  res.setHeader("Content-Disposition", `attachment; filename="${name}"`);
  res.sendFile(file);
});
```

---

### Python(Flask)

**✅ 안전 조합: `pathlib.resolve()` + `is_relative_to`**
```python
from pathlib import Path
from flask import Flask, request, send_file, abort

app = Flask(__name__)
SAFE_ROOT = Path("/var/app/logs").resolve()

def safe_path(name: str) -> Path:
    p = (SAFE_ROOT / name).resolve()
    # Python 3.9+: is_relative_to
    try:
        p.relative_to(SAFE_ROOT)
    except ValueError:
        abort(403)
    return p

@app.get("/logs")
def logs():
    name = request.args.get("name","")
    if not name or not name.isascii() or not all(c.isalnum() or c in "._-" for c in name):
        abort(400)
    return send_file(safe_path(name), as_attachment=True, download_name=name)
```

**포인트**
- `resolve()`로 **심볼릭 링크/`..` 제거**된 **실경로**를 얻은 뒤, **루트 상대성**을 확인.
- 파일명은 **영숫자·`._-`**만 허용, 길이 제한.

---

### — Java NIO `normalize/resolve`

**✅ 안전(디렉터리 고정 + 정규화 + startsWith)**
```java
Path ROOT = Paths.get("/var/app/docs").toRealPath(); // canonical

Path safeResolve(String userInput) throws IOException {
  // 입력은 파일명 기준으로만 허용(문자 제한)
  if (!userInput.matches("[A-Za-z0-9._-]{1,64}")) throw new SecurityException("Bad name");
  Path p = ROOT.resolve(userInput).normalize();
  if (!p.toRealPath().startsWith(ROOT)) throw new SecurityException("Traversal");
  return p;
}

@GetMapping("/doc")
public ResponseEntity<Resource> doc(@RequestParam String name) throws Exception {
  Path file = safeResolve(name);
  HttpHeaders h = new HttpHeaders();
  h.add("X-Content-Type-Options", "nosniff");
  h.add(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + file.getFileName() + "\"");
  return ResponseEntity.ok().headers(h).body(new FileSystemResource(file));
}
```
- Java NIO는 `Paths.get(...).resolve(...).normalize()` 조합으로 **`..` 제거 + 절대화**를 제공하므로 이를 활용하세요. (실무 팁/모범 사례는 Java NIO 보안 글 참고)

---

### PHP — `realpath` + 프리픽스 체크

**✅ 안전**
```php
<?php
$ROOT = realpath("/var/app/media");  // canonical
$name = $_GET["name"] ?? "";
if (!preg_match('/^[A-Za-z0-9._-]{1,64}$/', $name)) { http_response_code(400); exit; }

$full = realpath($ROOT . DIRECTORY_SEPARATOR . $name);
if ($full === false || strpos($full, $ROOT . DIRECTORY_SEPARATOR) !== 0) {
  http_response_code(403); exit;
}

header("X-Content-Type-Options: nosniff");
header("Content-Disposition: attachment; filename=\"" . basename($full) . "\"");
readfile($full);
```

---

## 안전 해법

**✅ Java 예시**
```java
Path dest = Paths.get("/var/app/uploads").toRealPath(); // canonical
try (ZipInputStream zis = new ZipInputStream(uploadedStream)) {
  ZipEntry e;
  while ((e = zis.getNextEntry()) != null) {
    String name = e.getName();
    if (name.contains("..") || name.startsWith("/") || name.startsWith("\\")) {
      throw new SecurityException("Traversal in entry: " + name);
    }
    Path target = dest.resolve(name).normalize();
    if (!target.startsWith(dest)) throw new SecurityException("Outside dest");
    if (e.isDirectory()) { Files.createDirectories(target); continue; }
    Files.createDirectories(target.getParent());
    Files.copy(zis, target, StandardCopyOption.REPLACE_EXISTING);
  }
}
```
- 핵심: **항목 이름 검사 + `resolve().normalize()` + `startsWith(dest)`**. 이 문제는 2018년 Snyk가 **Zip Slip**으로 정리해 다수 프로젝트에 경고한 바 있습니다.
- Android/백엔드 가이드도 **동일한 원리**를 권장합니다.

---

## 운영 환경 방어(서버/프록시/스토리지)

- **다운로드 전용 헤더**: 모든 다운로드 응답에 `Content-Disposition: attachment` + `X-Content-Type-Options: nosniff` → 브라우저 **렌더링 시도 억제**(특히 PDF/HTML/SVG).
- **웹서버 루트와 데이터 분리**: 공개 경로에서 **직접 파일시스템을 노출**하지 말 것(프록시를 통해 **ID→서버 내부 파일** 매핑)
- **디렉터리 리스팅 금지**, **심볼릭 링크 추적 금지**(옵션 확인)
- **권한/컨테이너/볼륨**: 읽기 전용 마운트, 민감 파일 별도 볼륨, **루트 밖 배치**
- **로깅/탐지**: `../`, `%2e%2e`, 혼합 구분자, 절대경로 패턴 요청을 중앙 로깅·알림

---

## 테스트 페이로드(개발/QA 전용)

- 단순: `../secret.txt`, `..\secret.txt`
- 이중/혼합 인코딩: `%2e%2e%2f`, `%252e%252e%252f`, `..%2f..%5c`
- 절대경로: `/etc/passwd`, `C:\Windows\win.ini`, `\\server\share\file`
- 특수: `....//`, `.. . /`(공백/점 혼합), `%c0%ae%c0%ae/`(유니코드 변종)
- Zip Slip: ZIP 항목 이름에 `../../outside.txt`
→ **기대 결과**: 모두 **차단** 또는 **루트 내부로 정규화 후 거절**.

PortSwigger 아카데미의 랩(우회 기법, 필터 회피)을 통해 자동화 테스트에 바꾸기 좋습니다.

---

## 실제 사고로 이어지는 흐름(케이스 스터디 요약)

1) **로그/이미지 다운로드 기능**에 Traversal → **서버 설정/소스/크리덴셜** 유출
2) 유출된 **DB/클라우드 키**로 2차 접근 → **데이터 대량 유출**
3) 업로드/Zip 해제 기능의 Zip Slip로 **임의 파일 쓰기 → RCE/웹셸**
→ Path Traversal은 “정보 유출”에서 끝나지 않고 **RCE까지 확장**될 수 있습니다.

---

## 체크리스트(요약)

- [ ] **파일 ID 매핑** 사용(가능하면 경로 입력 금지)
- [ ] **`resolve/normalize/realpath`** 후 **루트 내부 여부** 확인
- [ ] **허용 문자/길이** 제한(영숫자·`._-`)
- [ ] **절대경로/UNC/디바이스/링크** 거부
- [ ] **아카이브 해제**: `normalize+startsWith`, 링크/절대경로 금지
- [ ] **다운로드 헤더**: `attachment` + `nosniff`
- [ ] **로깅/탐지**: `..`, 인코딩 변종, 혼합 구분자
- [ ] **문서화/리뷰**: 파일 접근/저장/해제 경로 **다이어그램화** 후 리뷰
- [ ] **정기 스캔**: 정적/동적 분석(도로 `../` 탐지), 핫스팟 취약점 점검

---

## 부록 — 프레임워크별 주의 포인트

- **Express/Node**: `path.join`만으로는 불충분 → `path.resolve(root, user)` 후 **prefix 검사**
- **Flask/Django**: `send_file`/`send_from_directory` 사용 시 **디렉터리/파일명을 코드가 결정**(사용자 입력은 **식별자**로만)
- **Spring**: `Resource`/`ResourceUtils`로 직접 경로를 받지 말 것, `Paths`/`normalize`/`toRealPath` 조합으로 루트 내부만 허용(문자 제한 포함). Java NIO 보안 팁 참고.
- **PHP**: `realpath` + 프리픽스 검사, open_basedir 같은 설정 보조(완전한 대책이 아님)

---

## 참고/근거

- **OWASP**: Path Traversal 개요(예/우회/방어), Cheat Sheet Series.
- **PortSwigger Web Security Academy**: Path Traversal 개념·랩·후속 공격.
- **Zip Slip**: Snyk 리서치/깃허브 PoC, Android/일반 가이드.
- **MDN**: `X-Content-Type-Options: nosniff`·MIME 검증(브라우저 스니핑 억제).
- **Java NIO 모범 사례**(경로 정규화/검사).

---

### 한 줄 요약

> **경로를 받지 말고(가능하면 ID만), 받아야 한다면 “정규화→루트 내부인지 확인→허용 문자 제한”**을 적용하세요.
> Zip 해제·다운로드·로그보기·이미지서빙 등 **파일 접근 기능**은 모두 Path Traversal의 후보입니다. 다층 방어와 테스트 페이로드로 **막혀야 정상**입니다.
