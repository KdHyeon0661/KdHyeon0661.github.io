---
layout: post
title: 웹해킹 - File Upload 취약점
date: 2025-09-24 18:25:23 +0900
category: 웹해킹
---
# File Upload 취약점 — 웹쉘 업로드 사례와 보안 조치 완전 가이드

## 0. 개요: 왜 업로드가 위험한가

사용자 업로드는 “**신뢰할 수 없는 바이너리**가 **서버·브라우저·보조 처리기(이미지 라이브러리, AV, 변환기)**로 흘러드는 통로”입니다. 업로드 취약점은 크게 다음으로 나뉩니다.

- **서버 실행형**: 업로드 파일이 서버에서 **직접 실행**(예: `shell.php`, `shell.jsp`)되거나 설정 취약으로 **이중 확장자**까지 실행됨. 또한 경로 조작으로 파일 덮어쓰기/임의 위치 저장 가능.
- **클라이언트 공격형**: 업로드한 **SVG/HTML/PDF**에 스크립트가 내장되어 **XSS/콘텐츠 하이재킹** 유발, 브라우저의 MIME 스니핑까지 겹치면 확대.
- **파일 처리기 RCE형**: 업로드 파일을 처리하는 **이미지 변환기/파서**의 취약점 악용(예: **ImageTragick**, ImageMagick RCE).
- **용량·형식 기반 DoS**: **ZIP 폭탄**, 초대형 파일, 잘못된 포맷으로 리사이저/파서를 마비.

업로드 방어는 하나로 끝나지 않습니다. **확장자·MIME·시그니처·콘텐츠 검증**을 겹겹이 적용하고, **저장·서빙 구조**를 분리하며, **헤더·서버 설정·클라우드 정책**까지 아우르는 **다층 방어(Defense-in-Depth)**가 필수입니다.

---

## 1. 대표 공격 경로 디테일

### 1.1 웹쉘 업로드(서버 실행)

- 이중 확장자: `file.jpg.php`, `file.php.jpg` 등으로 **서버 파서/설정 빈틈**을 이용해 실행. 일부 환경/설정에서 실제로 실행될 수 있음.
- 구 버전 취약점: **널 바이트**(`.php%00.jpg`), **NTFS ADS**(`.asax:.jpg`, `::$DATA`) 등 **우회 케이스** 다수.

> **핵심**: “확장자만 보지 말고” 웹 서버가 **어떤 경로에서 무엇을 실행**하는지, 그리고 **저장 위치가 웹 루트인지**를 점검하세요.

### 1.2 클라이언트 공격(콘텐츠 하이재킹/XSS)

- **SVG**는 단순 이미지가 아니라 **스크립트/외부참조** 가능한 **활성 포맷**입니다. 많은 서비스/메일 클라이언트가 최근 SVG 취급을 강화했습니다. 업로드 SVG는 **표시하지 않거나** 라스터화/샌드박스로 처리하세요.
- **MIME 스니핑** 방지 헤더(`X-Content-Type-Options: nosniff`)가 없으면 브라우저가 콘텐츠를 **추정 렌더링**해 위험이 커집니다.

### 1.3 파일 처리기/변환기 RCE

- **ImageMagick** 계열 처리(리사이즈, 썸네일) 중 **ImageTragick(CVE-2016-3714)** 같은 **RCE** 사례가 있었습니다. 외부 “delegate” 호출 경로, 정책 파일 설정이 중요합니다.

### 1.4 업로드 로직의 논리 결함

- 이름/경로를 그대로 사용 → **경로 조작/덮어쓰기**
- 사이즈/개수 제한 없음 → **스토리지 고갈**
- 퍼블릭 버킷/폴더에 저장 → **무차별 배포/악성 호스팅**

---

## 2. 취약 코드 → 안전 코드 (언어별 레시피)

### 2.1 Node.js(Express + Multer) — 이미지 업로드

**❌ 취약 예시**: 확장자/`content-type`만 믿고, 웹 루트에 저장
```javascript
import express from "express";
import multer from "multer";
const app = express();

// 웹 루트에 바로 저장(취약), 파일명 유지(취약)
const upload = multer({ dest: "public/uploads" });

app.post("/upload", upload.single("file"), (req, res) => {
  // 여기서 아무 검증 없이 그대로 사용 → webroot에서 직접 접근 가능
  res.send(`/uploads/${req.file.originalname}`); // 원본 이름 노출(취약)
});
```

**✅ 개선 예시**: 다층 검증 + 안전 저장 + 안전 서빙
```javascript
import express from "express";
import multer from "multer";
import { randomUUID } from "crypto";
import { promises as fs } from "fs";
import path from "path";
import { fileTypeFromBuffer } from "file-type"; // magic 검출(libmagic 대안)
import createError from "http-errors";

const app = express();

// 1) 웹 루트 *밖* 임시 저장소
const upload = multer({ storage: multer.memoryStorage(), limits: { fileSize: 5 * 1024 * 1024 } });

// 2) 허용 목록(비즈니스 필수 최소)
const ALLOW_MIME = new Set(["image/png", "image/jpeg", "image/webp"]);
const ALLOW_EXT = new Set([".png", ".jpg", ".jpeg", ".webp"]);

// 3) 안전 저장 디렉터리(웹 루트 밖)
const SAFE_DIR = "/var/appdata/uploads"; // nginx/CloudFront에서 별도로 *다운로드 전용* 라우팅

app.post("/upload", upload.single("file"), async (req, res, next) => {
  if (!req.file) return next(createError(400, "no file"));

  // (a) 파일 시그니처 탐지(헤더 기반) – content-type 스푸핑 방지
  const sig = await fileTypeFromBuffer(req.file.buffer);
  if (!sig || !ALLOW_MIME.has(sig.mime)) return next(createError(415, "unsupported type"));

  // (b) 원본 파일명 사용 금지 → 무작위 이름 + 표준 확장자 강제
  const ext = sig.ext === "jpeg" ? ".jpg" : `.${sig.ext}`;
  if (!ALLOW_EXT.has(ext)) return next(createError(415, "bad extension"));
  const safeName = `${randomUUID()}${ext}`;

  // (c) 서버측 재인코딩(선택): 이미지 처리 라이브러리로 무해화(메타/스크립트 제거)
  // 예: sharp(req.file.buffer).toFormat('png').toFile(path.join(SAFE_DIR, safeName))

  // (d) 파일 기록(퍼미션 보수적)
  await fs.writeFile(path.join(SAFE_DIR, safeName), req.file.buffer, { mode: 0o600 });

  // (e) 응답은 "식별자 → 파일" 매핑만 제공, 직접 경로 노출 금지
  res.json({ id: safeName });
});
```
- **요점**: (1) **시그니처 검증**(magic)으로 `Content-Type` 스푸핑 방지, (2) **무작위 파일명**으로 경로·충돌·특수문자 위험 제거, (3) **웹 루트 밖 저장**으로 실행/직접 접근 차단.

> **참고**: libmagic/`file` 기반의 **시그니처 검증**은 확장자/헤더 우회를 어느 정도 막지만, **단독으로는 충분하지 않음**—반드시 **허용 목록**·**사이즈 제한**·**재인코딩/샌드박스**와 **조합**하세요.

### 2.2 PHP — `move_uploaded_file` 안전 패턴

**❌ 취약 예시**
```php
<?php
$target = __DIR__ . "/uploads/" . $_FILES["f"]["name"]; // 원본 파일명(취약)
move_uploaded_file($_FILES["f"]["tmp_name"], $target);  // webroot에 저장(취약)
echo "/uploads/" . $_FILES["f"]["name"];
```

**✅ 개선 예시**
```php
<?php
// 1) webroot 밖 저장
$SAFE_DIR = "/var/appdata/uploads";
if (!is_dir($SAFE_DIR)) mkdir($SAFE_DIR, 0700, true);

// 2) 파일 크기 제한
if ($_FILES["f"]["size"] > 5 * 1024 * 1024) http_response_code(413);

// 3) libmagic 기반 MIME 확인(finfo)
$finfo = new finfo(FILEINFO_MIME_TYPE);
$mime = $finfo->file($_FILES["f"]["tmp_name"]);
$allow = ["image/png" => "png", "image/jpeg" => "jpg", "image/webp" => "webp"];
if (!isset($allow[$mime])) http_response_code(415);

// 4) 안전 파일명 생성
$safe = bin2hex(random_bytes(16)) . "." . $allow[$mime];

// 5) 저장(권한 보수적)
$dest = $SAFE_DIR . "/" . $safe;
if (!move_uploaded_file($_FILES["f"]["tmp_name"], $dest)) http_response_code(500);

// 6) 응답은 내부 식별자만
echo json_encode(["id" => $safe]);
```
- **팁**: PHP-FPM/Apache 실행과 **물리적으로 분리된 디렉터리**를 쓰고, **.htaccess로 실행 차단**을 병행하세요(아래 서버 설정 참조).

### 2.3 Spring Boot(Java) — `MultipartFile` + Tika/libmagic 검증

```java
@PostMapping("/upload")
public ResponseEntity<?> upload(@RequestParam("f") MultipartFile f) throws Exception {
  if (f.getSize() > 5_000_000L) return ResponseEntity.status(413).build();
  // Apache Tika or libmagic wrapper로 시그니처 검증
  String mime = new Tika().detect(f.getBytes());
  if (!List.of("image/png","image/jpeg","application/pdf").contains(mime))
    return ResponseEntity.status(415).build();

  String ext = switch (mime) {
    case "image/png" -> ".png";
    case "image/jpeg" -> ".jpg";
    case "application/pdf" -> ".pdf";
    default -> "";
  };
  Path dir = Path.of("/var/appdata/uploads");
  Files.createDirectories(dir);
  Path dest = dir.resolve(UUID.randomUUID() + ext);
  Files.write(dest, f.getBytes(), StandardOpenOption.CREATE_NEW);
  // 권한 보수적
  Files.setPosixFilePermissions(dest, Set.of(OWNER_READ, OWNER_WRITE));
  return ResponseEntity.ok(Map.of("id", dest.getFileName().toString()));
}
```
- **베스트 프랙티스**는 **확장자·MIME·시그니처 3중 검증**, **랜덤 이름**, **웹 루트 외부 저장**입니다.

---

## 3. 서버/리버스 프록시 설정으로 “실행”을 구조적으로 차단

### 3.1 Nginx — 업로드 경로에서 스크립트 실행 금지 & 다운로드 전용 헤더
```nginx
# 업로드는 별도 가상경로나 별도 도메인/서브도메인 권장
location ^~ /uploads/ {
    # MIME 스니핑 방지 + 브라우저 표시 억제
    add_header X-Content-Type-Options nosniff;
    add_header Content-Disposition "attachment";
    default_type application/octet-stream;
    types { }  # 콘텐츠 타입 매핑 제거 → 기본값만 사용
}

# PHP 실행 금지(업로드 경로 내)
location ~* ^/uploads/.*\.(php|phtml|phar)$ {
    return 403;
}
```
- **`X-Content-Type-Options: nosniff`**로 브라우저의 MIME 추정을 막고, **`Content-Disposition: attachment`**로 업로드 파일을 **브라우저 내 렌더 대신 다운로드**하게 만들어 XSS/콘텐츠 하이재킹 위험을 낮춥니다.

### 3.2 Apache(.htaccess/가상호스트) — 실행 금지
```apache
# /uploads/ 이하에서 PHP 등 스크립트 직접 실행 금지
<Directory "/var/www/site/uploads">
  RemoveHandler .php .phtml .phar
  <FilesMatch "\.(php|phtml|phar)$">
    Require all denied
  </FilesMatch>
  Header set X-Content-Type-Options "nosniff"
  Header set Content-Disposition "attachment"
</Directory>
```
- **주의**: `.htaccess`는 “**직접 요청 실행**”만 막습니다. 다른 위치의 스크립트가 `include`로 실행시키는 건 별도 문제이므로, **웹 루트 외 저장**이 더 안전합니다.

---

## 4. 클라우드(S3/CloudFront)로 안전하게 서빙하기(권장 아키텍처)

- **S3는 기본적으로 비공개**(Block Public Access)로 두고, **CloudFront(OAC)**를 통해서만 접근 허용. 이렇게 하면 **직접 URL 노출**과 과도한 퍼블릭 액세스를 줄입니다.
- CloudFront **응답 헤더 정책**으로 `X-Content-Type-Options: nosniff`, `Content-Disposition: attachment`를 고정 설정하여 **렌더링 억제**. (S3에서는 객체 메타로 `Content-Disposition`도 지정 가능)

---

## 5. “콘텐츠” 자체를 무해화(이미지/문서)

### 5.1 이미지
- 업로드 이미지는 **재인코딩(라스터화)**해서 **EXIF/메타**와 숨겨진 페이로드를 제거.
- ImageMagick 사용 시 **정책 파일**로 위험 delegate 차단(과거 **ImageTragick**) 및 최신 버전 유지.

### 5.2 문서(PDF/Office) — **CDR(Content Disarm & Reconstruction)**
- AV는 **서명 기반**이라 우회가 가능. **CDR**은 “**실행 가능한 부분 제거 → 안전한 규격으로 재조립**” 접근으로 **제로데이 위험**을 낮춥니다(문서형 업로드에 특히 유용).

---

## 6. 악성 여부 스캔(AV) 파이프라인

- **ClamAV** 같은 서버측 AV로 업로드 직후 **동기/비동기 스캔**을 수행하고 **격리/차단** 플로우를 둡니다. (예: `clamdscan`/REST)
- 규모가 크면 **큐 기반(업로드 → 임시저장 → 스캔 → 정식저장)**으로 구성. 상용/오픈소스 모두 실무 적용사례 다수.

---

## 7. 특별 위험 포맷과 주의사항

- **SVG**: 스크립트·외부참조 가능 → **라스터화**(PNG 등)·**서버 렌더러 사용** 또는 **전면 비허용**. 최근 메일 클라이언트도 **Inline SVG 표시 제한** 추세.
- **PDF**: JavaScript/임베디드 컨텐츠 가능 → **CDR** 또는 **첨부 다운로드 강제**.
- **아카이브(ZIP)**: **ZIP 폭탄/경로탈출**(상대경로/심볼릭 링크) 검사 후 개별 항목 검증.

---

## 8. 파일명·경로·권한 하드닝 체크리스트

- **파일명**
  - **랜덤/UUID**로 교체, **허용 문자 제한**(영숫자·`-`·`_`·`.` 정도), **길이 제한**.
  - **선행 마침표**·복수 마침표(`..`)·예약명(CON/NUL 등)·트레일링 공백/점 제거.
- **경로**
  - 웹 루트 **외부** 저장. 다운로드는 **ID→파일** 매핑 핸들러로 제공.
- **권한**
  - 파일: `0600/0640`, 디렉터리: `0700/0750` 등 **최소 권한**. 실행 비트 제거.
  - 업로드 디렉터리는 **실행 핸들러 비활성**(Nginx/Apache 설정).

---

## 9. “웹쉘 업로드” 시나리오와 방어(교육용)

### 9.1 상황
- 취약 애플리케이션이 `public/uploads`에 업로드를 저장하고, **이중 확장자**를 막지 않음.
- 공격자는 `avatar.jpg.php`를 업로드(내용은 단순 `<?php echo "test"; ?>` 같은 **비해로운 데모**).

**결과**: `/uploads/avatar.jpg.php` 요청 시 **PHP 실행**. (일부 설정에서 발생 가능)
**방어 요점**: **웹 루트 외부 저장** + **확장자·시그니처 검증** + **업로드 경로 실행 금지** + **다운로드 전용 헤더**.

---

## 10. 프런트엔드/브라우저 레벨의 안전장치

- 업로드 성공 후 **미리보기**는 가능하면 **서버 라스터화 결과**(PNG)만 사용.
- 사용자 업로드 파일을 페이지에 **inline 렌더링 금지**(특히 SVG/PDF). **항상 다운로드(attachment)**로 제공. **MIME 스니핑 금지** 헤더 필수.

---

## 11. 테스트 벤치(보안/QA용) — “막혀야 정상”

1) **이중 확장자**: `test.jpg.php` 업로드 → 저장/접근이 **거부**되어야 함.
2) **널 바이트**: `file.php%00.jpg` → **거부** 또는 **정상화**되어야 함.
3) **시그니처 불일치**: `Content-Type: image/jpeg`지만 실제는 실행/스크립트 → **시그니처 검증 불일치로 차단**.
4) **SVG 활성 콘텐츠**: 업로드 후 **inline 미표시**, 다운로드만 되거나 라스터화된 결과만 노출.
5) **대용량/ZIP 폭탄**: **사이즈/타임아웃**에서 차단, 백엔드 경고/로그 생성.
6) **처리기 RCE 회피**: ImageMagick 정책 활성/버전 최신, 악성 샘플 변환 시 **차단**.

---

## 12. 운영 가이드(필수 설정 요약)

- **서버/리버스 프록시**: 업로드 경로 **실행 금지** + `nosniff` + `Content-Disposition: attachment`.
- **저장소**: **웹 루트 밖** 또는 **S3+CloudFront(OAC)**로 **사설 저장·프록시 서빙**.
- **검증**: 확장자 **허용 목록**, `Content-Type`은 **보조 지표**, **시그니처(libmagic/Tika)** 필수, 필요 시 **재인코딩/샌드박스**.
- **스캔/무해화**: **AV(ClamAV)** + **CDR**(문서) 병행.
- **로그/알림**: 업로드 실패/차단/스캔 결과/다운로드 트래픽 **집중 모니터링**.

---

## 13. 부록 — 실전 스니펫 모음

### 13.1 Express 다운로드 핸들러(안전 헤더 포함)
```javascript
app.get("/files/:id", async (req, res, next) => {
  const id = req.params.id.replace(/[^a-zA-Z0-9._-]/g, "");
  const file = path.join(SAFE_DIR, id);
  try {
    await fs.access(file);
    res.setHeader("X-Content-Type-Options", "nosniff");
    res.setHeader("Content-Disposition", `attachment; filename="${id}"`);
    res.sendFile(file);
  } catch (e) {
    next(); // 404
  }
});
```

### 13.2 Nginx: 업로드 전용 서브도메인(정적 다운로드 프록시)
```nginx
server {
  listen 443 ssl;
  server_name files.example.com;

  # 업로드 파일은 오직 다운로드(첨부)만
  location / {
    add_header X-Content-Type-Options nosniff;
    add_header Content-Disposition "attachment";
    proxy_pass http://app-internal/download/;
  }

  # 어떠한 스크립트도 실행 금지
  location ~* \.(php|phtml|phar|jsp|asp|aspx|cgi|pl|py)$ { return 403; }
}
```

### 13.3 S3/CloudFront(개요)
- S3 버킷: **Block Public Access ON**, 객체는 **사설**.
- CloudFront: **OAC** 설정으로만 접근, **응답 헤더 정책**에 `nosniff`/`attachment` 추가.

---

## 14. 자주 하는 실수 ↔ 교정

- **클라이언트 검증만**으로 충분하다고 생각 → 프록시/도구로 헤더·본문은 쉽게 조작됩니다. **서버측**이 최종 심판.
- **`Content-Type`만** 믿음 → 스푸핑 가능. **시그니처 검사**와 **허용 목록**을 결합하세요.
- **업로드를 웹 루트에 저장** → 설정 실수 하나면 바로 실행 경로. **외부 저장 + 실행 금지**가 기본.
- **SVG/문서 inline 렌더** → XSS/활성 콘텐츠 위험. **다운로드 강제** 또는 **CDR/라스터화**.
- **이미지 변환기 무신경 운영** → 과거 **ImageTragick** 같은 공급망형 RCE 전례. 정책·업데이트 필수.

---

## 참고 문헌 / 근거 링크

- **OWASP File Upload Cheat Sheet** — 다층 방어 원칙(확장자·시그니처·보관·권한·CDR/AV).
- **OWASP Unrestricted File Upload** — 이중 확장자/널바이트/ADS 등 우회 & 헤더 권장(`Content-Disposition`, `nosniff`).
- **MDN `X-Content-Type-Options: nosniff`** — MIME 스니핑 방지.
- **ImageTragick** (CVE-2016-3714) — 이미지 처리 RCE 사례.
- **libmagic** — 시그니처 검사 도구/라이브러리.
- **PortSwigger Web Security Academy** — 업로드 취약점 개요/테스트.
- **AWS S3/CloudFront OAC & Block Public Access** — 사설 버킷 + 프록시 서빙.

---

### 한 줄 요약
> 업로드 보안의 정석은 **“수용 최소화(허용 목록)” + “검증 중첩(확장자·MIME·시그니처·콘텐츠)” + “저장/서빙 분리(웹루트 외·프록시)” + “실행 차단(서버 설정)” + “무해화/스캔(CDR/AV)”** 입니다.
> 이 다층 방어가 **웹쉘 업로드·콘텐츠 하이재킹·변환기 RCE**까지 포괄적으로 낮춥니다.
