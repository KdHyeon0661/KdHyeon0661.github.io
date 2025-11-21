---
layout: post
title: JavaScript - 차세대 JavaScript 런타임
date: 2025-06-05 21:20:23 +0900
category: JavaScript
---
# 차세대 JavaScript 런타임: Bun과 Deno

## 빠른 비교 요약

| 항목 | Node.js | Deno | Bun |
|---|---|---|---|
| 엔진 | V8 | V8 | JavaScriptCore |
| 출시 | 2009 | 2020 | 2022 |
| TS 지원 | 도구로 별도 | 기본 내장 | 기본 내장 |
| 패키지 | npm | URL import + npm 지원(`npm:`) | npm 완전 호환(`bun install`) |
| 보안 | 개방형(기본 허용) | 샌드박스(기본 차단) | 개방형(기본 허용) |
| 내장 도구 | 적음 | fmt/test/lint/bench/compile | test/bundler/installer/runner |
| 속도(체감) | 안정 | 빠름 | 매우 빠름 |
| 강점 | 생태계/안정성 | 보안/표준 Web API | 속도/통합 DX |

---

## 런타임 기본 철학과 아키텍처

### Deno

- **보안 기본값**: 파일/네트워크/환경변수 접근은 `--allow-*` 플래그가 있어야 한다.
- **표준 Web API 우선**: `fetch`, `Request/Response`, `URLPattern`, `WebCrypto`, `Streams` 등 브라우저와 닮은 표면을 유지.
- **TS·포맷터·테스트·번들** 내장: 도구 종속성을 줄여 일관된 DX 제공.

### Bun

- **속도 지상주의**: JS 런타임 + 번들러 + 테스트 러너 + 패키지 매니저를 **하나**로 묶어 극단적 성능과 단순화를 추구.
- **npm 1급 지원**: 기존 Node 프로젝트를 거의 그대로 실행.
- **JSX/TS/JSON 내장 파싱**: 별도 전처리 없이 구동.

---

## “Hello, Runtime” — 가장 작은 서버

### Deno (권한 명시 + 표준 API)

```ts
// deno-hello.ts
Deno.serve(
  { port: 8080 },
  (_req) => new Response("Hello Deno", { headers: { "content-type": "text/plain" } })
);
```
실행:
```bash
deno run --allow-net=0.0.0.0:8080 deno-hello.ts
```

### Bun (초간단 + 초고속)

```ts
// bun-hello.ts
export default {
  port: 8080,
  fetch(_req: Request) {
    return new Response("Hello Bun", { headers: { "content-type": "text/plain" } });
  }
};
```
실행:
```bash
bun run bun-hello.ts
```

> 관찰 포인트
> - Deno는 “명시적 권한”이 없으면 네트워크 바인딩 자체가 거부된다.
> - Bun은 `fetch` 핸들러를 바로 노출하는 **서버 매니페스트 스타일**이 DX를 단순화한다.

---

## 파일/네트워크/환경변수 — 보안 모델과 권한

### Deno: 권한 플래그

```ts
// fs.ts
const data = await Deno.readTextFile("./secrets.txt");
console.log(data);
```
```bash
# 실패(권한 없음)

deno run fs.ts
# 성공(파일 읽기 허용)

deno run --allow-read=./secrets.txt fs.ts
```

기본 권한 스위치
- `--allow-read[=paths]`, `--allow-write`, `--allow-net[=hosts]`, `--allow-env[=vars]`, `--allow-run`, `--allow-ffi`

**권장**: 개발 중에도 최소 권한 원칙을 유지(실서버 잊지 않도록 습관화).

### Bun/Node: 기본 개방형

```ts
import fs from "node:fs/promises";
const t = await fs.readFile("./secrets.txt", "utf-8");
console.log(t);
```
- 추가 플래그 없음. **편하지만**, 운영에서 실수 위험은 높은 편 → **컨테이너/AppArmor/Seccomp** 등의 외곽 보안으로 보완.

---

## TypeScript — “설정 없이” 바로 쓰기

### Deno

```ts
// types.ts
export function greet(name: string) {
  return `Hello, ${name}`;
}
```
```bash
deno run types.ts
```
- `tsconfig` 없이도 TS 인식. 필요 시 `deno.json`에서 옵션 지정.

`deno.json` 예:
```json
{
  "tasks": {
    "dev": "deno run --watch --allow-net src/main.ts",
    "test": "deno test --allow-all",
    "fmt": "deno fmt",
    "lint": "deno lint"
  },
  "compilerOptions": {
    "strict": true
  }
}
```

### Bun

```ts
// index.ts
const sum = (a: number, b: number) => a + b;
console.log(sum(1, 2));
```
```bash
bun run index.ts
```
- 별도 설정 없이 TS/JSX 작동.
- 타입체크는 `tsc --noEmit`로 별도 돌리는 것을 권장(속도와 역할 분리).

---

## 패키지·의존성 — URL import vs npm

### Deno의 URL import + npm

```ts
// std 모듈(버전 고정)
import { join } from "https://deno.land/std@0.224.0/path/join.ts";

// npm 패키지
import _ from "npm:lodash@4.17.21";
console.log(_.chunk([1,2,3,4], 2));
```
- `npm:` 프리픽스로 npm 생태계를 활용 가능.
- `import_map.json`으로 별칭/잠금 관리 가능.

`import_map.json`:
```json
{
  "imports": {
    "@std/": "https://deno.land/std@0.224.0/"
  }
}
```
실행:
```bash
deno run --allow-net --import-map=import_map.json app.ts
```

### Bun의 npm 완전 호환

```bash
bun init
bun install axios
```
```ts
import axios from "axios";
const { data } = await axios.get("https://httpbin.org/get");
console.log(data);
```
- **설치 속도**와 **실행 속도** 모두 빠름.
- `node_modules`를 표준적으로 사용하므로 Node와 호환성 높음.

---

## HTTP 서버 — 라우팅/미들웨어/스트림

### Deno — 표준 `serve`와 스트림

```ts
// deno-server.ts
import { serveFile } from "https://deno.land/std@0.224.0/http/file_server.ts";

Deno.serve(async (req) => {
  const url = new URL(req.url);
  if (url.pathname === "/") {
    return new Response("root");
  }
  if (url.pathname === "/file") {
    return await serveFile(req, "./public/readme.txt");
  }
  if (url.pathname === "/stream") {
    const stream = new ReadableStream({
      start(controller) {
        controller.enqueue(new TextEncoder().encode("chunk1\n"));
        setTimeout(() => controller.enqueue(new TextEncoder().encode("chunk2\n")), 100);
        setTimeout(() => controller.close(), 200);
      }
    });
    return new Response(stream);
  }
  return new Response("Not found", { status: 404 });
});
```
```bash
deno run --allow-net --allow-read deno-server.ts
```

### Bun — 라우터 없이도 간결

```ts
// bun-server.ts
import { file } from "bun";

export default {
  port: 3000,
  async fetch(req: Request) {
    const { pathname } = new URL(req.url);
    if (pathname === "/") return new Response("root");
    if (pathname === "/file") return new Response(await file("./public/readme.txt").text());
    if (pathname === "/stream") {
      const stream = new ReadableStream({
        start(c) {
          c.enqueue(new TextEncoder().encode("chunk1\n"));
          setTimeout(() => c.enqueue(new TextEncoder().encode("chunk2\n")), 100);
          setTimeout(() => c.close(), 200);
        }
      });
      return new Response(stream);
    }
    return new Response("Not Found", { status: 404 });
  }
};
```
```bash
bun run bun-server.ts
```

---

## 데이터·스토리지 — Deno KV / Bun SQLite

### Deno KV (서버 없는 키-값 저장)

```ts
// deno-kv.ts
const kv = await Deno.openKv(); // 로컬: ./kv.sqlite 생성

const userKey = ["user", "u_123"];
await kv.set(userKey, { name: "Kim", age: 29 });

const res = await kv.get(userKey);
console.log(res.value); // { name: "Kim", age: 29 }
```
실행:
```bash
deno run --allow-read --allow-write deno-kv.ts
```
- 트랜잭션/Watch API/TTL 등 지원(간단한 서버리스 상태 보관에 유용).

### Bun + SQLite

```ts
// bun-sqlite.ts
import { Database } from "bun:sqlite";

const db = new Database("app.db");
db.run("CREATE TABLE IF NOT EXISTS users (id TEXT PRIMARY KEY, name TEXT)");

const stmt = db.prepare("INSERT INTO users (id, name) VALUES (?, ?)");
stmt.run("u_1", "Kim");

const rows = db.query("SELECT * FROM users").all();
console.log(rows);
```
```bash
bun run bun-sqlite.ts
```
- **내장 모듈**처럼 빠르고 간단. 소형 API/CLI에 적합.

---

## 테스팅·벤치마크·리니터

### Deno test/bench/lint/fmt

```ts
// calc.ts
export const add = (a: number, b: number) => a + b;
```
```ts
// calc_test.ts
import { assertEquals } from "https://deno.land/std@0.224.0/assert/mod.ts";
import { add } from "./calc.ts";

Deno.test("add", () => {
  assertEquals(add(1, 2), 3);
});
```
```bash
deno test
deno bench
deno fmt
deno lint
```

### Bun test

```ts
// math.test.ts
import { expect, test } from "bun:test";

const add = (a: number, b: number) => a + b;

test("add works", () => {
  expect(add(1, 2)).toBe(3);
});
```
```bash
bun test
```

---

## 번들/배포 — 단일 바이너리, Edge, Docker

### Deno compile — 단일 실행 파일

```ts
// cli.ts
console.log("CLI runs!");
```
```bash
deno compile --allow-read --output=cli cli.ts
./cli
```
- 운영 환경에 런타임 설치 없이 배포 가능.

### Edge 친화: 표준 Fetch 핸들러

- Deno/Bun 모두 `Request -> Response` 인터페이스를 사용 → **Cloudflare Workers**, **Vercel Edge** 등에 이식 쉬움(약간의 어댑터만).

### Docker 최소 이미지 예시

**Deno**
```Dockerfile
# deno.Dockerfile

FROM denoland/deno:alpine-1.45.5
WORKDIR /app
COPY . .
RUN deno cache src/main.ts
CMD ["run", "--allow-net", "src/main.ts"]
```

**Bun**
```Dockerfile
# bun.Dockerfile

FROM oven/bun:1.1.20-alpine
WORKDIR /app
COPY package.json bun.lockb ./
RUN bun install --ci
COPY . .
CMD ["bun", "run", "src/main.ts"]
```

---

## 성능 최적화 체크리스트

- **I/O를 스트림화**: 파일/네트워크 응답을 한 번에 메모리에 올리지 말 것.
- **JSON 파싱/직렬화 비용** 줄이기: 필요한 필드만, 가능하면 **NDJSON**/chunk 사용.
- **로깅 수준 조절**: 동기 파일 로깅 대신 비동기/버퍼.
- **Bun**: 라우팅/미들웨어 겹겹이 쌓기보다 **순수 fetch 핸들러**와 가까운 구조 유지.
- **Deno**: 권한/모듈 캐시 전략 확정, `deno.json`에 `tasks`로 반복 작업 단축.
- **프로파일링**: Deno `--inspect`/Bun `--inspect` 계열 디버깅 플래그로 CPU/Heap 분석.

---

## 마이그레이션 가이드

### Node → Deno (점진)

1) **표준 Web API** 사용 비율 높이기(특히 `fetch`, `URL`, `TextEncoder/Decoder`).
2) 모듈 경로를 상대/절대 URL로 정리.
3) npm 패키지는 `npm:` 접두로 옮기고 동작 확인.
4) 파일/네트워크 접근은 권한 스위치를 명시.
5) 스크립트화:
```json
// deno.json
{
  "tasks": {
    "dev": "deno run --watch --allow-net src/app.ts",
    "start": "deno run --allow-net src/app.ts"
  }
}
```

### Node → Bun (빠른 이식)

1) `bun install`로 의존성 설치(속도 향상 체감).
2) `bun run index.ts`로 TS/JSX 즉시 실행.
3) 테스트는 `bun test`로 교체(가능하면).
4) 성능 민감 구간에서 **즉시 개선** 체감(서버 스타트업, 번들링, 설치 속도).

---

## 실전 예제 프로젝트 — 간단 API + 캐시 + 테스트

### Deno 버전

```ts
// src/app.ts
const cache = new Map<string, string>();

Deno.serve(async (req) => {
  const url = new URL(req.url);

  if (url.pathname === "/") {
    return new Response("OK");
  }

  if (url.pathname === "/echo" && req.method === "POST") {
    const body = await req.text();
    return new Response(body, { headers: { "content-type": "text/plain" } });
  }

  if (url.pathname === "/cache") {
    const key = url.searchParams.get("key") ?? "default";
    if (cache.has(key)) return new Response(cache.get(key));
    const data = `value:${crypto.randomUUID()}`;
    cache.set(key, data);
    return new Response(data);
  }

  return new Response("Not Found", { status: 404 });
});
```
```bash
deno run --allow-net --watch src/app.ts
```

테스트:
```ts
// src/app_test.ts
import { assertEquals } from "https://deno.land/std@0.224.0/assert/mod.ts";

Deno.test("echo", async () => {
  const res = await fetch("http://localhost:8000/echo", { method: "POST", body: "hi" });
  assertEquals(await res.text(), "hi");
});
```
```bash
deno test --allow-net
```

### Bun 버전

```ts
// src/app.ts
const cache = new Map<string, string>();

export default {
  port: 8000,
  async fetch(req: Request) {
    const url = new URL(req.url);

    if (url.pathname === "/") return new Response("OK");

    if (url.pathname === "/echo" && req.method === "POST") {
      return new Response(await req.text(), { headers: { "content-type": "text/plain" } });
    }

    if (url.pathname === "/cache") {
      const key = url.searchParams.get("key") ?? "default";
      if (cache.has(key)) return new Response(cache.get(key));
      const data = `value:${crypto.randomUUID()}`;
      cache.set(key, data);
      return new Response(data);
    }
    return new Response("Not Found", { status: 404 });
  }
};
```
```bash
bun run src/app.ts
```

테스트:
```ts
// src/app.test.ts
import { expect, test } from "bun:test";

test("echo", async () => {
  const res = await fetch("http://localhost:8000/echo", { method: "POST", body: "hi" });
  expect(await res.text()).toBe("hi");
});
```
```bash
bun test
```

---

## 운영·관측: 로깅/헬스체크/그레이스풀 셧다운

### 공통 패턴 (Fetch 핸들러 기반)

```ts
// health.ts
export function healthHandler() {
  return new Response(JSON.stringify({ ok: true, ts: Date.now() }), {
    headers: { "content-type": "application/json" }
  });
}
// 서버 라우팅에서 /health 로 매핑
```

### Deno의 종료 시그널 처리

```ts
// graceful.ts
const abort = new AbortController();
Deno.addSignalListener("SIGTERM", () => abort.abort());
Deno.serve({ signal: abort.signal }, (_req) => new Response("OK"));
```

### Bun의 종료 훅(간단)

```ts
// bun-graceful.ts
let shuttingDown = false;
process.on("SIGTERM", () => (shuttingDown = true));

export default {
  port: 8080,
  fetch(_req: Request) {
    if (shuttingDown) return new Response("shutting down", { status: 503 });
    return new Response("ok");
  }
};
```

---

## 자주 맞닥뜨리는 함정과 대처

| 이슈 | Deno | Bun | 대처 |
|---|---|---|---|
| CJS/ESM 혼용 | URL import 권장, `npm:` 브리지 | 대부분 호환 | ESM 일관성 유지, 필요시 번들러/tsconfig 정리 |
| 권한 문제 | 기본 차단으로 오류 빈번 | 해당 없음 | Deno 플래그 명시, `deno.json`에 task로 고정 |
| 네이티브 애드온 | 제한적 | 일부 패키지 이슈 | Web API 대안 모색, WASM/REST 프록시 |
| 디버깅 | `--inspect` | `--inspect` | DevTools로 프로파일링, 소스맵 확인 |
| 프로덕션 안정성 | 성숙 | 성숙 중 | Docker 이미지 고정, 헬스체크/오토리커버리 |

---

## 선택 가이드 (의사결정 트리)

- **기존 Node 기반, npm 패키지 의존 강함** → **Bun**으로 속도 개선, 유지비 최소.
- **보안 정책 강함/서버리스·에지·표준 API 선호** → **Deno**.
- **성능 한계 타파·CI 시간 단축·빠른 부트스트랩** → **Bun** 우선 검토.
- **장기 유지/팀 합의 필요** → PoC 두 개(동일 기능), **로드 테스트 + 운영성 평가**로 선택.

---

## CI/CD 스니펫

**Deno (GitHub Actions)**
```yml
name: deno-ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: denoland/setup-deno@v1
        with: { deno-version: v1.x }
      - run: deno fmt --check
      - run: deno lint
      - run: deno test --allow-all
```

**Bun (GitHub Actions)**
```yml
name: bun-ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
        with: { bun-version: latest }
      - run: bun install --ci
      - run: bun test
```

---

## 성능 미세 팁 모음

- 응답 헤더 `content-type` 정확히 지정(파이프라인 최적화).
- 대용량 응답은 `ReadableStream`으로 **청크 전송**.
- JSON은 **`Response.json()`** (Deno/Bun 지원 시) 사용해 직렬화 비용 절약.
- CPU 바운드 작업은 **Worker/Thread/WASM** 고려.
- 캐시 전략: 인메모리 Map + TTL → Redis(원격) → KV/SQLite 순으로 승격.

---

## 결론

- **Bun**: “빠른 실행/설치/번들/테스트”가 **하나의 도구**로 응집된 런타임. **기존 Node 생태계를 거의 그대로** 품고 **속도 이점**을 준다.
- **Deno**: **보안 기본값**과 **표준 Web API**로 **현대적이고 안전한 서버/스크립트**를 만든다. **TS·도구 내장**이 운영 복잡도를 낮춘다.
- **선택은 목적에 따라**: 보안·표준·서버리스 지향이면 Deno, 속도·DX·npm 호환이 중요하면 Bun. 대규모 운영은 **PoC→벤치→관측**으로 검증하자.

---

## 로컬 벤치 예제(간단)

```bash
# Deno

deno run --allow-net deno-hello.ts &
wrk -t4 -c200 -d15s http://127.0.0.1:8080
kill %1

# Bun

bun run bun-hello.ts &
wrk -t4 -c200 -d15s http://127.0.0.1:8080
kill %1
```
> 수치는 환경마다 달라진다. 반드시 **자신의 워크로드**로 측정할 것.

---

## 문제 해결 Q&A

- **Q. Deno에서 npm 패키지가 동작 안 해요.**
  A. 우선 `npm:` 접두로 가져오는지 확인. Node 전용 API(`fs-extra`의 CJS-only 등)는 호환이 제한될 수 있다. 대안 모듈 또는 URL std 모듈 사용 고려.

- **Q. Bun에서 일부 네이티브 애드온이 깨져요.**
  A. 최신 Bun에서 점차 개선 중. 임시로 Node로 해당 부분만 분리하거나, WASM 대체/원격 서비스로 분리.

- **Q. 타입체크와 실행을 분리하고 싶어요.**
  A. Bun: `tsc --noEmit` 별도 스텝. Deno: `deno check` 또는 `deno test`로 타입 오류 탐지.

---

## 참고 링크

- Deno: https://deno.land
- Bun: https://bun.sh
- Deno 표준 라이브러리: https://deno.land/std
- Bun SQLite: https://bun.sh/docs/api/sqlite
- WHATWG Streams/Fetch: https://developer.mozilla.org/en-US/docs/Web/API
