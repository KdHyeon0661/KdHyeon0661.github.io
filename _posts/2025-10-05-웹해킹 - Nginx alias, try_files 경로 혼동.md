---
layout: post
title: 웹해킹 - Nginx alias, try_files 경로 혼동
date: 2025-10-05 14:25:23 +0900
category: 웹해킹
---
# 🧱 8. Nginx `alias` / `try_files` 경로 혼동

## 핵심 요약 (Executive Summary)

- **문제**
  - `location /static/ { alias /srv/static/; }` 같은 **경로 매핑**에서
    `alias`의 **후행 슬래시(`/`) 유무**, `location` 매칭 유형(정확/접두/정규식)과의 조합,
    `try_files`의 적용 대상 **(URI vs 파일시스템 경로)** 혼동이 겹치면
    **웹루트 밖 파일 노출, 디렉터리 인덱싱, `.git`/비밀키 유출**로 이어질 수 있습니다.
- **방어 원칙**
  1) **후행 슬래시 일관성**: 접두 위치(location)가 `/path/`면 `alias`는 `/real/dir/`처럼 **슬래시 쌍으로**.
  2) **정규식 location + `alias` 지양**: 되도록 **접두(location)**+`alias` 조합만. 필요 시 **`rewrite` → `root`**로 단순화.
  3) **`try_files`로 최종 404 강제**: `try_files $uri =404;`(정적), 동적 위임 시 `try_files $uri @app;`.
  4) **`internal` + `X-Accel-Redirect`**: 보호파일/다운로드는 앱에서 경로를 직접 노출하지 말고 **내부 위치로 프록시**.
  5) **심볼릭 링크·디코딩·슬래시 정규화**: `disable_symlinks on;` 등으로 **경로 탈출 방지**.

---

# `root` vs `alias` — **개념 차이**

- `root` : 요청 URI **그대로** 뒤에 붙습니다.
  ```nginx
  location /static/ {
    root /srv;     # /static/a.png → /srv/static/a.png
  }
  ```
- `alias`: **location 접두부를 제거한 나머지**를 **alias 경로 뒤에 붙입니다.**
  ```nginx
  location /static/ {
    alias /srv/static/;  # /static/a.png → /srv/static/a.png
  }
  ```
  - **중요**: `location`이 **접두(`/static/`)**이면 `alias`는 **보통 뒤에 `/`**를 둬야 의도대로 동작합니다.
  - `alias` 뒤에 `/`가 없으면 `/srv/statica.png`처럼 **디렉터리/파일명이 붙어버리는** 사고가 납니다.

> **규칙**: **접두 location**(끝에 `/`) ↔ **alias 경로**(끝에 `/`) — **둘 다 슬래시!**

---

# 흔한 실수 패턴과 안전 대체

## 후행 슬래시 불일치

**❌ 취약/오동작 예**
```nginx
location /static/ {
  alias /srv/static;   # ← 슬래시 빠짐
}
# /static/a.png → /srv/statica.png (원치 않는 경로)

```

**✅ 안전 대체**
```nginx
location /static/ {
  alias /srv/static/;  # 접두/별칭 모두 '/'로 끝나게
  try_files $uri =404;
}
```

---

## 정규식 location에 `alias` 사용

정규식은 **전체 매칭 문자열**과 **캡처 사용**이 섞여 **매핑 규칙이 헷갈리기 쉽습니다.**
정규식 + `alias`는 **경로 조합 실수**가 잦으므로 **지양**합니다.

**❌ 권장하지 않음**
```nginx
location ~ ^/media/(.*)$ {
  alias /srv/media/$1;  # 정규식과 alias 결합 → 슬래시/디코딩 혼선 위험
  try_files $uri =404;  # $uri는 원래 URI 기준이라 혼란 가중
}
```

**✅ 안전 대체 (정규식 → rewrite + root)**
```nginx
# 정규식으로 필요한 부분만 리라이트

location ~ ^/media/(.*)$ {
  rewrite ^/media/(.*)$ /$1 break;
  root /srv/media;      # → /srv/media/<$1>
  try_files $uri =404;
}

# 또는 접두 location로 단순화

location /media/ {
  alias /srv/media/;
  try_files $uri =404;
}
```

---

## `try_files` 오해 (URI vs 파일경로)

- `try_files`의 인자는 **파일시스템 경로로 해석**됩니다.
- **접두 location + alias**에서 `try_files $uri`는 **별칭이 적용된 파일 경로**로 확인됩니다(의도대로 쓰면 안전).
- **정규식 location**이나 **복잡한 rewrite** 후에는 해석이 혼동될 수 있으니 접두 방식 선호.

**권장 패턴**
```nginx
location /static/ {
  alias /srv/static/;
  # 1) 실제 파일만 제공하고 없으면 404
  try_files $uri =404;              # $uri는 /static/a → 별칭 후 /srv/static/a
  # 2) autoindex 끄기
  autoindex off;
  # 3) 범용 캐시 헤더는 정적 location에서만
  add_header Cache-Control "public, max-age=31536000, immutable";
}
```

---

## `alias`와 하위 위치 섞기

`alias`를 둔 위치 아래에 **추가 하위 location**을 겹치면 **매핑 혼선**이 생길 수 있습니다.
되도록 **한 location에서 끝내거나**, **명시적 재작성**으로 분기하세요.

**✅ 명시 분기**
```nginx
location /assets/ {
  alias /srv/assets/;
  try_files $uri =404;
}
location /assets/private/ { return 404; }  # 별칭 영역 특정 서브패스 차단
```

---

# **웹루트 밖 파일 노출**이 일어나는 경로

1) `alias` 후행 슬래시 누락 → 경로가 **붙어**버려 상위로 치우침
2) 정규식 캡처/디코딩 조합 → `..`, `%2e%2e/` 같은 **우회 문자열**이 의도치 않게 관통
3) 심볼릭 링크를 통해 **외부 디렉터리**로 빠져나감
4) `try_files`가 없거나 2차 처리로 **동적 백엔드**에 넘겨 **파일 내용을 출력**(다운로드)

> **방어 요점**: **매핑 단순화 + 슬래시 일관 + try_files =404 + 심볼릭 링크 제한**.

---

# **안전한 레시피 모음** (복붙 가능)

## 정적 파일 전용 매핑 (가장 흔한 케이스)

```nginx
location /static/ {
  alias /srv/static/;          # 둘 다 슬래시
  try_files $uri =404;         # 존재하지 않으면 404
  autoindex off;               # 디렉터리 목록 비활성
  types {                      # (선택) 확장자 미지정시 대비
    text/css css;
    application/javascript js;
  }
  add_header X-Content-Type-Options nosniff;
  add_header Cache-Control "public, max-age=31536000, immutable";
}
```

## — **`internal` + `X-Accel-Redirect`**

**Nginx**
```nginx
# 앱이 내부 경로로만 전달 가능 (직접 URL 접근 차단)

location /protected-download/ {
  internal;                    # 외부 접근 불가
  alias /srv/protected/;       # 실제 파일 경로
  try_files $uri =404;
}
```

**Node/Express (예)**
```js
// 사용자는 /download/:name 로 접근 → 앱은 접근 권한 검사 후 내부 리다이렉트
app.get("/download/:name", async (req, res) => {
  const user = req.user; // 권한 확인
  const name = req.params.name.replace(/[^a-z0-9_.-]/gi, ""); // 파일명 화이트리스트
  if (!(await canDownload(user, name))) return res.sendStatus(403);

  // 내부로만 유효한 URL로 위임
  res.set("X-Accel-Redirect", `/protected-download/${name}`);
  // (선택) 콘텐츠 타입/이름 지정
  res.set("Content-Type", "application/octet-stream");
  res.set("Content-Disposition", `attachment; filename="${name}"`);
  res.end();
});
```

**Flask (예)**
```python
@app.get("/download/<name>")
def download(name):
  if not can_download(current_user, name): abort(403)
  resp = Response()
  resp.headers["X-Accel-Redirect"] = f"/protected-download/{secure_filename(name)}"
  resp.headers["Content-Type"] = "application/octet-stream"
  return resp
```

> **효과**: 앱은 **경로를 공개하지 않고**, Nginx가 **내부 별칭 경로**에서만 파일을 읽습니다.

## 동적 앱과 혼용 (`try_files $uri @app;`)

```nginx
# 정적: 있으면 파일, 없으면 앱으로

location / {
  try_files $uri @app;
}

location @app {
  proxy_pass http://app_backend;
  # 앱 라우팅/보안은 여기서
}
```

> 이때 **정적 파일 디렉터리**는 `root` 또는 **접두+alias**로 **명확히 분리**하세요.
> (정적과 동적 처리가 뒤섞이면 파일 노출/오동작 위험이 올라갑니다.)

---

# **경로 정규화** & **심볼릭 링크** 방어

```nginx
# 심볼릭 링크 제한: 소유자 기준/전면 차단

disable_symlinks on;             # 또는 'if_not_owner' (버전에 따라)
# 이중 디코딩/슬래시 혼선 방지

merge_slashes on;
absolute_redirect on;
port_in_redirect off;            # (필요시)
# 숨김/민감 파일 차단(예: .git, env, key)

location ~ /\.(git|svn|hg)|(^|/)\.env {
  deny all;
}
```

> `disable_symlinks` 는 **별칭 디렉터리 밖**으로 빠져나가는 **심링크 경유 노출**을 줄입니다.

---

# **테스트 시나리오(스테이징만)** — “항상 404여야 하는 것들”

> 아래는 **유효성 검사** 아이디어입니다(운영/타사 금지).
> 의도는 **노출이 없음을 증명**하는 것 — 성공 기준은 **404/403/415/405** 등 **거절**입니다.

```bash
BASE=https://staging.example.com

# 상위로 나가기 시도(정규화/디코딩)

curl -si "$BASE/static/..%2f..%2fetc/passwd" | head -n1
curl -si "$BASE/static/../../../../etc/hosts" | head -n1

# .git/.env 같은 숨김 파일

curl -si "$BASE/static/.git/config" | head -n1
curl -si "$BASE/.env" | head -n1

# 슬래시 변형/이중 디코딩

curl -si "$BASE/static//a.css" | head -n1
curl -si "$BASE/static/%2f/a.css" | head -n1

# 디렉터리 인덱스 금지 확인

curl -si "$BASE/static/" | grep -i "autoindex" -n || true  # 인덱스 페이지가 나오면 취약
```

**기대 결과**: 모두 **404(또는 403)**. “파일 내용”이 응답되면 즉시 설정 수정.

---

# `try_files` 고급 팁

- **정적 제공**:
  - `try_files $uri $uri/ =404;` → 파일 또는 **디렉터리 인덱스**가 의도라면 `index` 지시어를 명시.
  - 디렉터리 인덱싱을 **원치 않으면** `=404`로 단락.
- **여러 경로 후보**:
  ```nginx
  location /assets/ {
    alias /srv/assets/;
    # 해시/버전된 파일이 없으면 원본으로 폴백, 그래도 없으면 404
    try_files $uri /assets/origin$uri =404;
  }
  ```
- **앱 위임**:
  - `try_files $uri @router;` : 정적 없을 때만 앱으로 넘김 → **파일 유출 확률 감소**.

---

# 우선순위** 주의

- `=`(정확일치) > `^~`(접두 고정) > 정규식(`~`, `~*`) > 일반 접두.
- 정규식이 뒤에 선언되어도 **우선순위 규칙**에 따라 매칭됩니다.
- **권장**: 정적은 `^~ /static/` 같은 **접두 고정**으로 일찍 매칭시켜 **앱으로 새지 않게**.

```nginx
location ^~ /static/ {
  alias /srv/static/;
  try_files $uri =404;
}
location / { try_files $uri @app; }
location @app { proxy_pass http://app; }
```

---

# — **화이트리스트만**

도메인에 따라 정적 루트를 바꾸는 경우 **사용자 입력을 경로에 끼우지 마세요.**
**도메인→디렉터리**는 **사전 매핑**으로만.

```nginx
# 예: a.example.com → /srv/tenants/a/, b.example.com → /srv/tenants/b/

map $host $tenant_root {
  default        /srv/tenants/default/;
  a.example.com  /srv/tenants/a/;
  b.example.com  /srv/tenants/b/;
}

server {
  listen 443 ssl http2;
  root $tenant_root;                 # root만으로 충분하면 alias 불필요
  location ^~ /static/ {
    alias $tenant_root/static/;      # (주의) 변수 alias는 Nginx 버전에 따라 제한적
    try_files $uri =404;
  }
}
```

> **팁**: 변수 확장과 `alias`의 조합은 버전·빌드에 따라 제약이 있습니다.
> 가능하면 **서버 블록 분리**(vhost) 또는 **root만**으로 설계하세요.

---

# 운영 하드닝 추가 항목

- `autoindex off;`(전역/정적 위치)
- `client_max_body_size` 제한 (업로드/경로 프로빙 남용 억제)
- `limit_except`로 허용 메서드 최소화 (정적은 `GET`, `HEAD`만)
- 에러 페이지 커스터마이징으로 **파일 경로 노출** 방지
- 업스트림에 **원본 경로**를 넘길 때(`X-Original-URI`) **로그로만** 남기고 **응답에 반영하지 않기**

---

# **CI/검수 자동화 아이디어**

- **Nginx 정적 검증**:
  ```bash
  nginx -t || exit 1
  nginx -T > /tmp/nginx.full.conf   # 전체 렌더 확인
  # 1) alias 후행 슬래시 점검
  grep -nE "location\s+/[^\s]*/\s*\{[^}]*alias\s+/[^;{}]*[^/];" -n /tmp/nginx.full.conf \
    && echo "::warning ::alias without trailing slash in prefixed location"
  # 2) 정규식 location에서 alias 사용 경고
  grep -nE "location\s+~\*?\s+.*\{[^}]*alias\s+" -n /tmp/nginx.full.conf \
    && echo "::warning ::alias used in regex location — review!"
  ```
- **E2E 스모크(스테이징)**: §6의 `curl` 시나리오 자동 실행 → **404/403** 기대.

---

# 체크리스트 (현장용)

- [ ] 접두 location(`/path/`) ↔ `alias /real/dir/;` **슬래시 일치**
- [ ] 정규식 location + `alias` **지양**, 필요시 `rewrite + root`로 단순화
- [ ] 모든 정적 위치에 `try_files $uri =404;`
- [ ] 민감/보호 경로는 `internal` + `X-Accel-Redirect`
- [ ] `.git`, `.env`, 키/백업 등 **숨김 파일 전면 차단**
- [ ] `disable_symlinks on;` 등 심링크 탈출 방지
- [ ] `autoindex off;`, 허용 메서드 최소화
- [ ] 위치 우선순위: 정적은 `^~`로 고정, 동적은 `@app`로 위임
- [ ] 스테이징에서 상위 이동/이중 디코딩/슬래시 변형 **모두 404** 확인
- [ ] CI에서 **alias 슬래시/정규식 경고** 룰 적용

---

## 맺음말

Nginx의 `alias`/`try_files`는 **강력하지만 미묘**합니다.
**슬래시 일관성**, **정규식 지양**, **`try_files`로 단락**, **내부 전송(`internal`/`X-Accel-Redirect`)** 네 가지 원칙만 지켜도
“**웹루트 밖 파일 노출**” 같은 치명적인 사고를 **구조적으로 방지**할 수 있습니다.
