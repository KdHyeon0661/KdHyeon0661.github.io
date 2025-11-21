---
layout: post
title: HTML - 문자셋(character set)
date: 2025-02-23 19:20:23 +0900
category: HTML
---
# 문자셋(Character Set)이란?

## 개념 정리 — 문자셋·인코딩·유니코드·코드포인트

- **문자셋(Character Set)**: 표현 가능한 **문자의 집합**. 예: 유니코드(전 세계 문자), ASCII(128자), EUC-KR(한글 중심).
- **인코딩(Encoding)**: 문자(코드포인트)를 **바이트 시퀀스**로 변환하는 규칙. 예: UTF-8, UTF-16, EUC-KR.
- **유니코드(Unicode)**: 문자에 **고유 번호(코드포인트)**를 부여하는 국제 표준. 예: "가" = U+AC00.
- **UTF-8**: 유니코드의 대표 인코딩. **가변 길이(1~4바이트)**, ASCII와 하위 호환.

> 핵심: **동일한 유니코드 문자라도** 인코딩(UTF-8/UTF-16/…​)이 다르면 **바이트 표현**이 달라집니다.

---

## 역사와 현재 — 왜 UTF-8인가?

- **ASCII(1960s)**: 7비트, 128자(영문/숫자/기호). 국제화 불가.
- **ISO-8859-1(Latin-1)**: 서유럽 문자 확장. 한글/한자 미지원.
- **EUC-KR**: 한국어 환경에서 오랫동안 사용. 범용성/이모지 미흡.
- **UTF-8(현 표준)**: 전 세계 문자, 이모지, 기호, 확장 평면까지 표현 가능. 웹/모바일/클라우드 전반의 실질 표준.

> 오늘날 **웹/서버/DB/이메일/파일** 전반에서 **UTF-8(가능하면 utf8mb4 계열)**로 통일이 최선입니다.

---

## 주의

### 권장 메타 태그

```html
<!-- 반드시 <head> 최상단 근처 -->
<meta charset="UTF-8">
<title>문자셋 예제</title>
```

- **위치 중요**: `<head>` 초반에 배치해야 브라우저가 소스 파싱 초기에 올바른 인코딩을 적용.
- HTML4 구식 표기:
```html
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
```
> 유지보수성/명확성 측면에서 **HTML5 방식**을 권장.

### 혼합 인코딩의 함정

- HTML 파일은 `UTF-8`인데, **서버 헤더**가 `charset=EUC-KR`로 응답 → 브라우저는 **HTTP 헤더**를 우선하기 때문에 깨짐.
- 반대로 서버는 UTF-8, **문서 메타가 EUC-KR** → 일부 브라우저/도구에서 혼선.

**정답**: **HTTP 헤더, HTML 메타, 템플릿 인코딩**을 **모두 UTF-8**로 일치.

---

## HTTP·서버 사이드 — 헤더/프레임워크별 설정

### HTTP 응답 헤더

```http
Content-Type: text/html; charset=UTF-8
```
- HTML뿐 아니라 JSON/JS/CSS에도 적절한 MIME 타입과 인코딩을 지정:
```http
Content-Type: application/json; charset=UTF-8
Content-Type: text/css; charset=UTF-8
Content-Type: application/javascript; charset=UTF-8
```

### Node.js(Express)

```js
const express = require('express');
const app = express();

// 1) 정적: 기본은 OK지만, 커스텀 헤더 추가 가능
app.use((req, res, next) => {
  res.set('Content-Type', 'text/html; charset=UTF-8'); // 라우트별로 유형 조정
  next();
});

// 2) JSON 응답: 자동 UTF-8. 단, 헤더 명시 습관화
app.get('/api', (req, res) => {
  res.set('Content-Type', 'application/json; charset=UTF-8');
  res.json({ message: '안녕하세요 😊' });
});

app.listen(3000);
```

### Python Flask

```python
from flask import Flask, Response, jsonify
app = Flask(__name__)

@app.get("/")
def index():
    html = "<!doctype html><meta charset='utf-8'><h1>안녕하세요 😊</h1>"
    return Response(html, headers={"Content-Type": "text/html; charset=utf-8"})

@app.get("/api")
def api():
    resp = jsonify({"message": "한글/이모지 OK"})
    resp.headers["Content-Type"] = "application/json; charset=utf-8"
    return resp
```

### Django

- `settings.py` 템플릿/파일 저장은 UTF-8로.
- 미들웨어/뷰에서 `Content-Type` 확인:
```python
from django.http import JsonResponse, HttpResponse

def hello(request):
    resp = HttpResponse("<meta charset='utf-8'>안녕하세요")
    resp["Content-Type"] = "text/html; charset=utf-8"
    return resp

def api(request):
    resp = JsonResponse({"ok": True})
    resp["Content-Type"] = "application/json; charset=utf-8"
    return resp
```

### PHP

```php
<?php
header('Content-Type: text/html; charset=UTF-8');
echo "<!doctype html><meta charset='utf-8'><h1>안녕하세요 😊</h1>";
```

### Nginx

```nginx
http {
  charset utf-8;                   # 기본 charset
  types {
    text/html   html htm shtml;
    text/css    css;
    application/javascript js;
    application/json json;
  }
}
```

### Apache

```apache
AddDefaultCharset UTF-8
AddType 'text/html; charset=UTF-8' .html
AddType 'application/json; charset=UTF-8' .json
```

> **한 줄 요약**: **서버 헤더 + 템플릿 파일 인코딩 + HTML 메타**를 **모두 UTF-8**로 맞춰야 혼선이 없습니다.

---

## DB 설정 — MySQL·PostgreSQL·ORM, 그리고 이모지

### MySQL/MariaDB — `utf8mb4`가 정답

- MySQL의 `utf8`은 **최대 3바이트**로, 일부 이모지(4바이트) 저장 실패.
- **반드시 `utf8mb4`와 적절한 정렬(예: `utf8mb4_0900_ai_ci`)**을 사용.

```sql
-- 서버/DB/테이블/컬럼 모두 점검
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';

-- DB 생성 시
CREATE DATABASE appdb
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

-- 테이블
CREATE TABLE posts (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  title VARCHAR(200),
  body TEXT,
  author VARCHAR(100)
) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;

-- 연결 세션
SET NAMES utf8mb4;
```

**인덱싱 주의**: MySQL InnoDB는 인덱스 키 길이 제한이 있습니다.
`VARCHAR(191)`는 utf8mb4(4바이트) 기준 **191×4=764바이트**로 안전(옛 호환).
최신 MySQL에서는 큰 키 길이도 가능하지만, **프리픽스 인덱스** 고려 권장.

### PostgreSQL

- 데이터베이스 자체가 유니코드 친화적. 기본적으로 **UTF-8 권장**.
```sql
CREATE DATABASE appdb WITH ENCODING 'UTF8' LC_COLLATE='ko_KR.utf8' LC_CTYPE='ko_KR.utf8' TEMPLATE=template0;
```

### ORM(Django/SQLAlchemy/TypeORM 등)

- **커넥션 파라미터**에서 `charset=utf8mb4` 명시.
- 마이그레이션 시 스키마에 **문자셋/콜레이션**이 반영되는지 확인.

---

## 이모지/조합문자/정규화 — 보이는 것과 길이가 다르다

### 가시 문자 ≠ 코드포인트 수 ≠ 바이트 수

- “👩‍💻”(여성 기술자 이모지)은 **여러 코드포인트의 조합(ZWJ, 변형 선택자 등)** 일 수 있습니다.
- 문자열 길이 제한/자르기/색인/정렬 시 **문자 경계(grapheme cluster)** 를 고려해야 합니다.

#### 길이 개념(요약)

- **문자 수(사용자가 보는 글자 수)**: grapheme cluster
- **코드포인트 수**: 유니코드 포인트 개수
- **바이트 수**: 인코딩(UTF-8 등) 적용 후의 실제 저장 크기

수학적으로, UTF-8 바이트 수는 각 코드포인트의 범위에 따라 달라집니다:
$$
\text{UTF-8 bytes} =
\begin{cases}
1 & \text{if } U+0000 \le c \le U+007F \\
2 & \text{if } U+0080 \le c \le U+07FF \\
3 & \text{if } U+0800 \le c \le U+FFFF \\
4 & \text{if } U+10000 \le c \le U+10FFFF
\end{cases}
$$

### 정규화(Normalization)

- 유니코드에는 **동일한 가시 결과**를 낳는 **서로 다른 조합**이 존재.
- 예: `é`는 **단일 코드포인트(U+00E9)** 혹은 **`e`(U+0065)+결합 악센트(U+0301)`**.
- **NFC**(권장), NFD, NFKC, NFKD 등 **정규화** 개념 필요.
- 파일명/검색/중복 검사 전 **NFC 정규화**로 일관성 확보.

### 서버 사이드 예(정규화)

- Node.js:
```js
const s = 'e\u0301'; // 'e' + 결합 악센트
const nfc = s.normalize('NFC');
```
- Python:
```python
import unicodedata
s = 'e\u0301'
nfc = unicodedata.normalize('NFC', s)
```

---

## — UTF-8 with BOM의 함정

- UTF-8은 바이트 순서가 고정이라 **BOM이 불필요**.
- 일부 편집기는 `UTF-8 with BOM`으로 저장 → **서버 사이드 파서/CLI 스크립트/JSON 파서**가 **선행 바이트를 내용으로 오인**.
- **권장**: **UTF-8(BOM 없음)** 으로 저장.

---

## 폼/API/CSV/이메일 — 입출력 경로별 주의점

### HTML 폼/서버

- HTML 기본은 UTF-8. 서버 측 **요청 바디 파서의 문자셋** 확인.
- 파일 업로드(멀티파트)에서 **파일명 인코딩(RFC 5987/2231)** 이 이슈가 될 수 있음.
- URL 쿼리스트링/경로는 **퍼센트 인코딩**. 서버 라우팅에서 **디코딩 시점/중복 디코딩** 주의.

### JSON API

- 명시적으로:
```http
Content-Type: application/json; charset=UTF-8
```
- 일부 클라이언트/프록시가 **잘못 추정**하는 경우 있음 → 항상 헤더로 명시.

### CSV

- Excel은 지역 설정에 따라 **CP949/EUC-KR/Shift_JIS** 가 섞일 수 있음.
- **UTF-8 with BOM**을 요구하는 환경도 존재(구버전 Excel).
  가능하면 **명시**하고, 사내 표준을 문서화.
- 샘플 헤더/메타 파일로 **인코딩 사전 합의**.

### 이메일(MIME)

- 헤더/본문 모두 인코딩 명시 필요.
```http
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: quoted-printable
```
- 제목/이름 필드는 RFC 2047 인코딩:
```
Subject: =?UTF-8?B?7JWI64WV7ZWY?=
```
- HTML 이메일 본문에도 `<meta charset="UTF-8">` + 헤더 일치.

---

## 브라우저/도구에서의 확인

- **개발자도구 → Network**: `Content-Type` 헤더의 `charset` 확인.
- **View Source**: `<meta charset="...">` 위치/값 확인.
- **curl/wget/httpie** 로 원문 응답 검사:
```bash
curl -i https://example.com | sed -n '1,20p'
```
- **file/iconv/uchardet** 등으로 파일 인코딩 추정:
```bash
file -bi index.html
iconv -f EUC-KR -t UTF-8 old.html > new.html
```

---

## 트러블슈팅 — 흔한 증상과 원인·해결

### "���" 또는 물음표/깨짐

- 원인: **서버 헤더/HTML 메타/파일 저장 인코딩 불일치**
- 해결: **세 곳 모두 UTF-8로 통일**. 에디터 저장 형식 확인. BOM 제거.

### DB에 저장 시 깨짐/에러

- 원인: DB/커넥션/테이블/컬럼 인코딩 불일치, MySQL `utf8` 사용.
- 해결: **utf8mb4**로 전환, `SET NAMES utf8mb4`, 마이그레이션 시 스키마/인덱스 재점검.

### 이모지 저장 실패

- 원인: MySQL `utf8` 3바이트 제한.
- 해결: **utf8mb4** + 적절 콜레이션.

### 문자열 자르기 시 이모지 반쪽(깨짐)

- 원인: 바이트/코드포인트 단위 자르기.
- 해결: **grapheme cluster 단위** 자르기(라이브러리 사용).

### 혼합 인코딩 문서

- 원인: 일부 포함 파일/템플릿이 EUC-KR 등.
- 해결: 빌드 파이프라인에서 **정적 검사** + **iconv** 일괄 변환.

---

## 마이그레이션 가이드 — EUC-KR/CP949 → UTF-8

1. **소스/템플릿/정적 파일**: 에디터 일괄 변환(백업 필수).
   `iconv -f CP949 -t UTF-8 old.html > new.html`
2. **DB**:
   - MySQL: `ALTER DATABASE ... CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;`
   - 각 테이블/컬럼 변경 + 데이터 백업/복원.
3. **서버/리버스 프록시**: `Content-Type` 헤더 점검.
4. **이메일/CSV 파이프라인**: 발송/임포트 도구의 인코딩 옵션 통일.
5. **테스트**: 한글/중국어/아랍어/이모지/조합문자 케이스로 스냅샷 테스트.

---

## 실전 예제 모음

### HTML 스켈레톤(권장)

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>문자셋 예제</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body>
  <h1>안녕하세요 😊</h1>
  <p>이 문서는 UTF-8로 인코딩되었습니다.</p>
</body>
</html>
```

### JSON API 응답(명시적 헤더)

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8

{"message":"한글/Emoji OK 🚀"}
```

### MySQL 테이블 생성(utf8mb4)

```sql
CREATE TABLE messages (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  text VARCHAR(500) NOT NULL
) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

### Python CSV(UTF-8 BOM 필요 환경 대응)

```python
import csv

rows = [["이름", "메모"], ["홍길동", "이모지 😊"]]
with open("out.csv", "w", newline="", encoding="utf-8-sig") as f:
    writer = csv.writer(f)
    writer.writerows(rows)
# utf-8-sig: Excel 일부 환경 호환을 위해 BOM 첨부

```

### 이메일(MIME 헤더 + 본문)

```text
Content-Type: text/html; charset=UTF-8
Content-Transfer-Encoding: quoted-printable
Subject: =?UTF-8?B?7JWI64WV7ZWYIOyImO2DgA==?=

<!doctype html>
<meta charset="utf-8">
<p>안녕하세요 😊</p>
```

---

## 품질 체크리스트(배포 전)

- [ ] HTML `<meta charset="UTF-8">`가 `<head>` 초반에 있는가?
- [ ] 모든 HTTP 응답에 올바른 `Content-Type; charset=UTF-8`가 설정되는가?
- [ ] 서버 템플릿/정적 파일이 **UTF-8(무BOM)** 으로 저장되어 있는가?
- [ ] DB(특히 MySQL)가 **utf8mb4**이며, 커넥션/테이블/컬럼/인덱스가 일치하는가?
- [ ] 이모지/조합문자/다언어 텍스트가 **입력→저장→조회→렌더링** 전 과정에서 유지되는가?
- [ ] 문자열 자르기/유효성 검사에서 **grapheme cluster** 단위를 고려했는가?
- [ ] CSV/이메일/파일 경로/파일명에서 **인코딩 합의**와 테스트를 마쳤는가?
- [ ] 정규화(NFC) 전략을 수립/적용했는가(검색·중복 검사·파일명)?

---

## FAQ — 실무에서 자주 묻는 것

**Q1. MySQL `utf8`인데 가끔 이모지가 깨져요.**
A. `utf8`은 3바이트까지만. **utf8mb4**로 전환하세요.

**Q2. 페이지는 UTF-8인데 왜 소스에선 깨져 보이나요?**
A. 에디터의 보기 인코딩이 달라서일 수 있습니다. 파일 인코딩 자체를 UTF-8로 저장.

**Q3. Excel에서 CSV 한글이 깨져요.**
A. 대상 환경에서 **UTF-8 with BOM(utf-8-sig)** 을 요구할 수 있습니다. 사내 표준 문서화.

**Q4. 길이 제한 100자의 칼럼에 이모지 문자열을 자를 때 안전하게?**
A. 바이트/코드포인트 단위가 아닌 **grapheme cluster** 단위로 자르는 라이브러리를 사용.

**Q5. 왜 어떤 서버는 메타보다 HTTP 헤더를 우선하나요?**
A. 표준적으로 브라우저는 **HTTP 헤더의 charset**을 우선합니다. 항상 헤더/메타/파일을 일치시키세요.

---

## 결론

- 문자셋은 **단순 설정**이 아니라, **시스템 전체의 합의**입니다.
- **UTF-8(무BOM)** 을 기본으로, DB는 **utf8mb4**, 헤더/메타/파일/툴 전부를 일치시킵니다.
- 이모지/조합문자/정규화/CSV/이메일 등 **엣지 케이스**까지 대비하면,
  국제화 시대의 텍스트 문제를 **사전에 차단**하고 유지보수를 크게 줄일 수 있습니다.
