---
layout: post
title: 웹해킹 - Source Map / 디버그 아티팩트 노출
date: 2025-10-10 20:25:23 +0900
category: 웹해킹
---
# 24. Source Map / 디버그 아티팩트 노출
**— 개념 · 실제 노출 시 피해 · 안전한 빌드/배포 전략 · 프레임워크별 설정(Next/Vite/Webpack/Angular/CRA/SvelteKit/Nuxt/Rollup) · 서버/엣지 차단(Nginx/Apache/CloudFront/Worker) · CI 검사 파이프라인 · “막혀야 정상” 테스트 · 사고 시 대응**

> ⚠️ 목적  
> 프로덕션에서 **소스맵(.map)**·**디버그 산출물**이 노출되면 소스 코드·주석·내부 경로·빌드 환경이 외부에 드러나고, 취약 로직 리버스가 **매우 쉬워**집니다.  
> 이 문서는 “**노출을 원천 차단**”하고 “**사내 오류 추적 도구와만 안전하게 연동**”하는 실무 지침을 제공합니다.

---

## 0. 한눈에 보기 (Executive Summary)

- **문제**  
  - `.map`(JS/CSS source map)·디버그 파일(예: webpack stats, stacktrace 심볼, 자바스크립트 인라인 소스맵, 모바일 dSYM/mapping.txt)이 **공개 경로**에 있으면,  
    - 난독화 우회(원본 함수/변수명 복원),  
    - 주석/비공개 경로/토큰처럼 **코드에 박힌 민감 정보** 노출,  
    - 내부 API 경로/권한 우회 아이디어 파악 → 악용까지 이어질 수 있습니다.
- **핵심 방어 원칙**
  1) **프로덕션 소스맵은 공개하지 않는다.** (필요 시 **hidden-source-map**/**nosources-source-map** + **비공개 업로드**)
  2) **배포 아티팩트에 `sourceMappingURL` 주석이 남지 않게** 빌드/후처리한다.
  3) **서버/엣지에서 .map 접근을 차단**하거나 **별도 사설 버킷/도메인**에 격리한다.
  4) **CI에서 정적 검사**(빌드 산출물에 `.map`/`sourceMappingURL`/디버그 파일이 있으면 **실패**).
  5) **오류 추적(Sentry/Rollbar 등)**에는 **비공개 업로드**만 수행(공개 오리진에 소스맵을 두지 않음).

---

## 1. 소스맵이 뭐고 왜 위험한가?

- **Source Map**: 번들·난독화된 코드 ↔ 원본 파일/라인으로 **매핑**하는 JSON.  
  브라우저 디버거는 이 파일을 가지고 **개발자 경험**을 좋게 합니다.  
- **프로덕션 위험**
  - `.map`에는 **원본 경로/파일명/변수명/주석**이 포함됩니다.  
  - 번들 옆에 `.map`이 **공개**되어 있고, 또는 JS/CSS 끝에  
    ```js
    //# sourceMappingURL=app.abc123.js.map
    ```
    같은 **주석이 남아** 있으면 누구나 다운받을 수 있습니다.
  - 결과: 취약 지점 찾아내기 + 리버스 비용 급감 → XSS/로직우회/키 노출(잘못된 클라이언트 내 임베드) 위험 증가.

> 비밀은 **원래 클라이언트에 실으면 안 되지만**, 현실적으로 주석·힌트·내부 경로 등이 남아 **공격 난이도를 크게 낮춥니다.**

---

## 2. 프로덕션 전략 설계 (권장 아키텍처)

### A) 사용자 브라우저에는 **소스맵을 제공하지 않기**
- JS/CSS 산출물 끝에서 **`sourceMappingURL` 주석 제거**.
- `.map` 파일은 **생성하되**(오류 추적에 필요), **배포(origin/CDN)에 올리지 않음**.  
  또는 **별도 사설 버킷/도메인**에 보관하여 **일반 웹에서 비공개**.

### B) 오류 추적 도구에는 **사설 업로드**
- Sentry/Rollbar/New Relic 등에 **소스맵 업로드** → 런타임 에러 스택을 **내부에서만** 디맵핑.
- 프로덕션 오리진엔 `.map`이 없어도, 우리만 **스택 프레임 복원** 가능.

### C) 서버/엣지에서 **.map 접근 차단**
- Nginx/Apache/CloudFront/Workers에서 `*.map`(및 디버그 파일) **deny**.
- 가급적 **산출물에서 제거** + **원천적으로 미업로드**가 더 안전.

### D) CI에서 **정적 검사로 “실패해야 정상”**
- 빌드 후 산출 디렉토리에서 **`.map` 파일 0개**여야 하거나(공개 배포 기준),  
  최소 **`sourceMappingURL` 문자열이 JS/CSS에 존재하면 실패**.

---

## 3. 프레임워크/번들러별 설정

### 3.1 Webpack
- 프로덕션 추천: `devtool: 'hidden-source-map'` 또는 `false`
  - `hidden-source-map`: 소스맵 생성 O, **번들에 주석 미삽입**(URL 힌트 없음).  
    → `.map`을 **공개 업로드 하지 말고**, **오류 추적 도구로만 업로드**.
  - `nosources-source-map`: 매핑은 있으나 **소스 콘텐츠 미포함**.  
    → 그래도 공개 업로드는 지양. 내부 업로드 용도로만.
  - `false`: 소스맵 생성 안 함(가장 안전하지만, 에러 역추적 불편 → APM/Sentry 업로드 안 쓰는 경우).

```js
// webpack.prod.js
module.exports = {
  mode: 'production',
  devtool: 'hidden-source-map', // 또는 false
  // ...
};
```

**Sentry 업로드 예 (sentry-webpack-plugin)**  
> 공개 오리진에 `.map`을 두지 않고, 빌드시 Sentry에 업로드.
```js
const SentryWebpackPlugin = require("@sentry/webpack-plugin");

module.exports = {
  // ...
  plugins: [
    new SentryWebpackPlugin({
      org: "your-org",
      project: "your-project",
      authToken: process.env.SENTRY_AUTH_TOKEN,
      // public 경로엔 업로드하지 않음
      include: "./dist",
      urlPrefix: "~/", // 배포 경로에 맞춰 조정
      release: process.env.RELEASE, // 릴리즈 태그 일치
      debug: false
    })
  ]
}
```

### 3.2 Vite / SvelteKit / Vue / React(비-CRA)
- `build.sourcemap`: `false` 또는 `'hidden'` 권장  
  - `false`: 생성 안 함.  
  - `'hidden'`: 생성하되 **배포 번들에 주석 미삽입**.
```ts
// vite.config.ts
import { defineConfig } from 'vite';
export default defineConfig({
  build: {
    sourcemap: 'hidden', // 또는 false
    // rollupOptions: { output: { ... } }
  }
});
```
- Sentry 업로드는 `vite-plugin-sentry` 등 사용(지도: 내부 업로드).

### 3.3 Rollup
```js
// rollup.config.mjs
export default {
  // ...
  output: {
    sourcemap: true,       // 생성하되
    sourcemapExcludeSources: false // 필요에 따라
  }
}
// 👉 번들 결과물의 마지막 주석을 strip(아래 후처리)하거나
//    .map은 사설 저장소로만 업로드
```
- **후처리**로 `sourceMappingURL` 주석 제거(아래 §4.2 코드).

### 3.4 Next.js
- `next.config.js`  
  - `productionBrowserSourceMaps: false` (기본 false) 유지.  
  - 소스맵이 필요하면 **@sentry/nextjs**로 **서버 측 업로드**만 수행.
```js
// next.config.js
const { withSentryConfig } = require('@sentry/nextjs');
module.exports = withSentryConfig(
  {
    reactStrictMode: true,
    productionBrowserSourceMaps: false, // 공개 금지
  },
  { silent: true }
);
```

### 3.5 CRA(Create React App)
- `.env.production`에 다음 설정:
```
GENERATE_SOURCEMAP=false
```
- 필요 시 `react-app-rewired`나 빌드 후 **주석 제거 스크립트**로 보강.

### 3.6 Angular
- `angular.json` 프로덕션 빌드:
```json
"configurations": {
  "production": {
    "optimization": true,
    "sourceMap": false,              // 또는 { "scripts": false, "styles": false, "hidden": true }
    "namedChunks": false,
    "buildOptimizer": true
  }
}
```
- Angular는 `hidden` 지원(버전에 따라): **생성하되 외부 노출 X** → 내부 업로드만.

### 3.7 Nuxt
- `nuxt.config.ts`
```ts
export default defineNuxtConfig({
  sourcemap: false, // 또는 { server: true, client: false }
  // client false를 권장(브라우저 노출 방지)
})
```

---

## 4. 빌드 후 **후처리(Strip/검사)**

### 4.1 CI에서 **.map 파일 차단**
```bash
# 빌드 산출물에 .map이 있으면 실패
if find dist -type f -name "*.map" | grep -q .; then
  echo "❌ .map files detected in dist (public). Failing build."; exit 1;
fi
```

### 4.2 `sourceMappingURL` 주석 제거(Inline/External 모두)
```js
// tools/strip-sourcemap-comment.js
import fs from 'node:fs';
import path from 'node:path';

const ROOT = process.argv[2] || 'dist';

const strip = (code) =>
  code
    // JS: //# sourceMappingURL=...
    .replace(/\/\/# sourceMappingURL=.*$/gm, '')
    // CSS: /*# sourceMappingURL=... */
    .replace(/\/\*# sourceMappingURL=.*?\*\//gs, '')
    // Inline data URL 케이스도 제거
    .replace(/sourceMappingURL=data:application\/json[^"\n\r]*/g, '');

function walk(dir) {
  for (const f of fs.readdirSync(dir)) {
    const p = path.join(dir, f);
    const st = fs.statSync(p);
    if (st.isDirectory()) walk(p);
    else if (/\.(js|css)$/i.test(p)) {
      const before = fs.readFileSync(p, 'utf8');
      const after = strip(before);
      if (before !== after) {
        fs.writeFileSync(p, after);
        console.log(`Stripped sourceMappingURL: ${p}`);
      }
    }
  }
}
walk(ROOT);
```
CI 단계:
```bash
node tools/strip-sourcemap-comment.js dist
```

### 4.3 “디버그 산출물” 블랙리스트 검사
```bash
# 흔한 디버그/내부 파일 차단
grep -R --include="*.js" -n "webpack:///" dist && { echo "❌ webpack debug path leaked"; exit 1; }
grep -R -E --include="*.{js,css,html}" -n "(sourceMappingURL|__webpack_require__|webpackJsonp)" dist > /tmp/hints.txt
# 발견되면 리뷰/정책에 따라 실패 처리
```

---

## 5. 서버/엣지에서 접근 차단

### 5.1 Nginx
```nginx
# 1. 디폴트: 모든 .map 접근 거부
location ~* \.map$ { return 403; }

# 2. 자산 경로 한정 (예: /assets/만)
location ^~ /assets/ {
  location ~* \.map$ { return 403; }
  try_files $uri =404;
}

# 3. 기타 내부 파일 흔적도 차단
location ~* /\.(git|env|htaccess|bash|log) { deny all; }
```

### 5.2 Apache
```apache
<FilesMatch "\.map$">
  Require all denied
</FilesMatch>
<FilesMatch "^(\.git|\.env|.*\.log)$">
  Require all denied
</FilesMatch>
```

### 5.3 CloudFront(권장: **아예 업로드하지 않기**)
- **Behavior**: `Path pattern: *.map` → **Origin Request Policy**로 **Signed URL만 허용** 혹은 **오리진 미연결(403)**.  
- **대안**: 소스맵은 **별도 사설 버킷**에 저장(공용 오리진에 없음).

### 5.4 Cloudflare Workers (간단 차단)
```js
export default {
  async fetch(req) {
    const url = new URL(req.url);
    if (url.pathname.endsWith('.map')) {
      return new Response('Forbidden', { status: 403 });
    }
    return fetch(req);
  }
}
```

---

## 6. 오류 추적 도구 연동(공개 노출 없이)

### 6.1 Sentry(예)
- 전략: 프로덕션 **번들에는 주석 없음**(hidden) + `.map`은 **Sentry로 업로드**.
- GitHub Actions 예:
```yaml
name: build
on: [push]
jobs:
  web:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npm run build
        env:
          RELEASE: ${{ github.sha }}
      - name: Upload source maps to Sentry
        run: |
          npx sentry-cli releases new $RELEASE
          npx sentry-cli releases files $RELEASE upload-sourcemaps ./dist --url-prefix "~/"
          npx sentry-cli releases finalize $RELEASE
        env:
          SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
          SENTRY_ORG: your-org
          SENTRY_PROJECT: your-project
      - name: Enforce no .map in dist (public)
        run: |
          if find dist -type f -name "*.map" | grep -q .; then
            echo "❌ .map detected in public dist"; exit 1; fi
          node tools/strip-sourcemap-comment.js dist
```

### 6.2 Rollbar/New Relic 등
- 유사하게 **CI에서 업로드** → **공개 오리진에는 업로드하지 않음**.

---

## 7. 모바일/백엔드 디버그 아티팩트도 고려

- **Android**: ProGuard/R8 `mapping.txt` → **공개 배포 금지**, Crashlytics/사내 서버에만 업로드.
- **iOS**: dSYM(심볼) → **사설 업로드**, 공개 저장 금지.
- **.NET**: PDB → 서버에만, 클라이언트 배포 금지.
- **Go/Java**: 디버그 심볼/DWARF/stacktrace map 등은 **내부 보관**.

> 웹 팀과 모바일/백엔드 팀 **공통 정책**으로 “디버그/심볼은 **공개 아티팩트 금지**”를 문서화하세요.

---

## 8. “막혀야 정상” 테스트 시나리오

1) **.map 직접 접근 차단**
```bash
curl -i https://static.example.com/assets/app.abc123.js.map
# 기대: 403/404/444 (절대 200 아님)
```
2) **번들에 주석 제거 확인**
```bash
curl -s https://static.example.com/assets/app.abc123.js | grep -q sourceMappingURL && echo "❌ 주석 존재" && exit 1
```
3) **인라인 소스맵 제거 확인**
```bash
grep -R "sourceMappingURL=data:application/json" dist && { echo "❌ inline map 발견"; exit 1; }
```
4) **오류 추적 역매핑 검증**
- 스테이징에서 의도적인 에러 발생 → Sentry에서 **원본 라인/파일명 복원** 확인(대외 공개 없이).

---

## 9. 사고 발생 시(이미 노출됨)

1) **캐시 무효화/파일 제거**: CDN·오리진에서 `.map`·디버그 파일 즉시 삭제/무효화.
2) **키/토큰 교체**: 리포지터리·코드 내 주석/변수에 민감값이 있었으면 **전면 회수/로테이션**.
3) **번들 재배포**: `sourceMappingURL` 제거, 빌드 파이프라인 수정.
4) **접근 로그 조사**: `.map` 접근 기록/다운로드 횟수 파악, 후속 공격 징후 모니터링.

---

## 10. 운영 체크리스트

- [ ] 프로덕션 **소스맵 비공개**(기본) / 필요 시 **hidden + 내부 업로드**  
- [ ] 번들에 **`sourceMappingURL` 주석 없음**  
- [ ] 오리진/CDN에 **`.map` 미업로드**(또는 엣지 차단)  
- [ ] **CI 검사**: `.map`/`sourceMappingURL`/디버그 산출물 발견 시 실패  
- [ ] 오류 추적 도구에는 **사설 업로드**만(릴리즈 태그 일치)  
- [ ] 서버/엣지 차단(Nginx/Apache/CloudFront/Workers) 설정  
- [ ] 모바일(dSYM/mapping.txt)/백엔드(PDB/심볼) **공개 금지**  
- [ ] 사고 대응 플레이북(키 회수/캐시 퍼지/로그 조사) 준비

---

## 11. 작게 시작하는 베이스라인(복붙 레시피)

1) **번들러**  
   - Webpack: `devtool: 'hidden-source-map'`  
   - Vite: `build.sourcemap = 'hidden'`  
   - Next: `productionBrowserSourceMaps = false`
2) **CI**  
   - 빌드 후 `strip-sourcemap-comment.js` 실행  
   - `find dist -name "*.map"` → 발견 시 실패
3) **서버**  
   - Nginx/Apache에서 `*.map` 403  
   - CloudFront/Worker로 403 보조
4) **오류 추적**  
   - Sentry/등에 **내부 업로드**  
   - 릴리즈 태그 일치(번들 ↔ 소스맵)

---

## 12. 예제: Express 정적 호스팅에서 .map 차단

```js
import express from 'express';
import path from 'node:path';

const app = express();
const DIST = path.join(process.cwd(), 'dist');

// 정적 서빙
app.use((req, res, next) => {
  if (req.url.endsWith('.map')) return res.status(403).send('Forbidden');
  next();
});
app.use(express.static(DIST, { immutable: true, maxAge: '1y', extensions: ['html'] }));

app.listen(8080, () => console.log('listening on :8080'));
```

---

## 13. 부록: 팀 내 가이드라인(1페이지 버전)

- 프로덕션 배포물에는 **.map/디버그 아티팩트 금지**  
- 예외(내부 역매핑 필요) 시: **hidden-source-map + 내부 업로드**, 공개 오리진 업로드 금지  
- 번들 끝 **`sourceMappingURL` 제거** 정책(빌드 후 스크립트)  
- 서버/엣지 **`.map` 차단**(Nginx/Apache/CloudFront/Worker)  
- CI에서 **정적 검사**(발견 시 실패)  
- 사고 시 **퍼지 → 키 로테이션 → 재배포 → 로그 조사**

---

### 마무리
소스맵/디버그 아티팩트는 개발 편의에 유용하지만, **프로덕션에서는 공격자 편의**가 되기 쉽습니다.  
**공개 비활성 + 내부 업로드 + 서버/엣지 차단 + CI 검사** 4단으로 방어하면 재발을 구조적으로 막을 수 있습니다.