---
layout: post
title: JavaScript - Serverless와 JavaScript
date: 2025-06-01 23:20:23 +0900
category: JavaScript
---
# Serverless와 JavaScript: 프론트엔드 개발자를 위한 서버리스 입문

## Serverless 한 줄 정의와 오해 풀기

> **Serverless** = “서버 없음”이 아니라 “**서버 관리 부담 없음**”.
> 코드를 **함수 단위**로 배포하면, 실행 시점에만 **자동으로 인프라가 뜨고** 끝나면 내려갑니다(종량 과금).

### 동작 순서(이벤트 구동)

1. 이벤트 발생(HTTP 호출, Cron, 큐/스토리지/DB 트리거 등)
2. 플랫폼이 함수를 **컨테이너/아이솔레이트**에서 **자동 실행**
3. 실행 종료 → **자원 회수** → 사용량만 과금

---

## 왜 JavaScript와 궁합이 좋은가?

| 이유 | 상세 |
|---|---|
| Node.js 기본 지원 | 모든 메이저 서버리스에서 1급 시민 |
| 스택 일원화 | 프론트+백(함수) 동일 언어/패키지 생태계 |
| 빠른 실험 | 파일 하나로 API를 만들고 즉시 배포 |
| SDK/도구 | npm 생태계, TypeScript, 번들러, Lint, Test 생태계 풍부 |

---

## 플랫폼 스펙트럼: Regional Lambda vs Edge Functions

| 구분 | 예 | 런타임 | 콜드스타트 | 파일 접근 | 네트워킹 | 용도 |
|---|---|---|---|---|---|---|
| **리저널(Regional) FaaS** | AWS Lambda, GCF, Azure Functions | Node.js(컨테이너) | 수~수백 ms(메모리/언어/패키지 영향) | 임시 디스크 제한 | VPC 연동 가능 | DB 접근/무거운 작업 |
| **에지(Edge) Functions** | Cloudflare Workers, Vercel Edge | V8 Isolate/Deno | 극저지연(1~10ms) | 일반적으로 불가 | 제한적(표준 `fetch`) | 인증/리다이렉트/캐시/프록시 |

> **요령**: DB가 필요한 **상태풀 연산**은 보통 리저널, **인증/리라이팅/캐싱**은 에지에서 전처리 후 전달하는 **하이브리드** 구성이 흔합니다.

---

## 자주 쓰는 JS 서버리스 플랫폼 요약

| 플랫폼 | 진입 난이도 | 장점 | 유의점 |
|---|---|---|---|
| **AWS Lambda** (+ API Gateway) | 중 | 성숙/풍부한 통합, VPC/DB 연계 | 설정이 넓고 깊음, 콜드스타트/권한 관리 신경 |
| **Vercel Functions / Edge** | 하 | Next.js/프론트 통합, 배포 간단 | 장기실행/대용량 제약, 베타 기능 확인 |
| **Cloudflare Workers** | 하 | 초저지연 글로벌 엣지, KV/R2/D1 | Node 호환성 일부 다름, 패키지 제약 |
| **Firebase Functions** | 하 | Auth/RTDB/Firestore와 자연 통합 | 리전/요금/콜드스타트 이해 필요 |
| **Netlify Functions** | 하 | 정적+함수 통합, 쉬운 CI/CD | 고급 네트워킹/DB는 별도 설계 |

---

## 첫 코드 — “Hello”를 넘어서 실용 패턴으로

### AWS Lambda (Node.js 20, API Gateway Proxy)

/**`lambda/index.mjs`**/
```js
export const handler = async (event) => {
  // event.headers, event.queryStringParameters, event.body(JSON 문자열) 등
  return {
    statusCode: 200,
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ ok: true, message: 'Hello from Lambda' }),
  };
};
```

- **배포 자동화**: Serverless Framework, AWS CDK, SAM 중 하나를 권장
- **API Gateway**와 매핑 시 **CORS** 헤더 설정 필수

**SAM 로컬 실행(선택)**
```bash
sam init --runtime nodejs20.x
sam local start-api
```

---

### Vercel Functions (Node.js) — 파일 기반 라우팅

/**`api/sendMail.js`**/
```js
import nodemailer from 'nodemailer';

export default async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();
  const { name, email, message } = req.body || {};
  if (!name || !email || !message) return res.status(400).json({ error: 'invalid' });

  const transporter = nodemailer.createTransport({
    service: 'Gmail',
    auth: { user: process.env.GMAIL_ID, pass: process.env.GMAIL_PW },
  });

  await transporter.sendMail({
    from: email,
    to: 'admin@example.com',
    subject: `Contact from ${name}`,
    text: message,
  });

  res.status(200).json({ status: 'sent' });
}
```

- **로컬 개발**: `vercel dev` (Next.js 프로젝트면 바로 동작)
- **환경변수**: Vercel Project Settings → Environment Variables

---

### Cloudflare Workers (에지, 빠른 프록시/인증)

/**`src/index.js`**/
```js
export default {
  async fetch(req, env, ctx) {
    const url = new URL(req.url);
    if (url.pathname === '/ping') {
      return new Response(JSON.stringify({ pong: true }), { headers: { 'content-type': 'application/json' }});
    }
    // 백엔드 API 프록시/캐시
    return fetch('https://api.example.com' + url.pathname, req);
  },
};
```

- **로컬**: `wrangler dev`
- **KV/R2/D1**와의 통합으로 세션리스 데이터 작업 최적

---

## 서버리스 “필수” 아키텍처 패턴

### 입력 검증(Validation) — *Zod 등 스키마 기반*

```js
import { z } from 'zod';

const schema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  message: z.string().min(1).max(2000),
});

export function parseJsonSafe(body) {
  try { return JSON.parse(body || '{}'); }
  catch { return null; }
}

// Lambda/Vercel 공용 핸들러 패턴
export async function handle(eventOrReq) {
  const payload = isLambda(eventOrReq)
    ? parseJsonSafe(eventOrReq.body)
    : eventOrReq.body;
  const result = schema.safeParse(payload);
  if (!result.success) return badRequest(result.error.format());
  return ok({ received: true });
}
```

### 오류 처리 & 표준 응답 포맷

```js
const ok = (data) => ({ statusCode: 200, body: JSON.stringify({ ok: true, data }) });
const badRequest = (err) => ({ statusCode: 400, body: JSON.stringify({ ok: false, error: err }) });
const fail = (e) => ({ statusCode: 500, body: JSON.stringify({ ok: false, error: 'internal' }) });
```

### 인증(JWT/세션) — 에지 선필터 → 리저널 백엔드

- **Edge**에서 쿠키/헤더 검사, 빠른 401/302 처리
- 통과 시 **백엔드 함수**로 전달하여 DB 등 무거운 연산 수행

### 파일 업로드 — **Presigned URL**

- 서버리스 함수에서 **S3/R2** 업로드 **사전 서명 URL** 발급 → 프론트가 직접 업로드(서버 부하↓)

**AWS S3 presign 예**
```js
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
const s3 = new S3Client({ region: process.env.AWS_REGION });

export async function createUploadUrl(key, contentType) {
  const cmd = new PutObjectCommand({ Bucket: process.env.BUCKET, Key: key, ContentType: contentType });
  return getSignedUrl(s3, cmd, { expiresIn: 60 }); // 60초 유효
}
```

### 비동기 파이프라인 — 큐/이벤트/크론

- **EventBridge(Cron)/Cloud Scheduler/Vercel Cron**으로 주기 작업
- **SQS/Cloud Tasks/Queues**로 비동기 처리, 재시도/보관/지연 지원

### DB 연결 베스트 프랙티스

- 서버리스는 **컨테이너가 재사용**될 수 있어 연결을 **모듈 스코프에 캐시**
- RDBMS는 커넥션 폭증 위험 → **RDS Proxy/pgBouncer** 또는 **서버리스 친화 DB(PlanetScale/Neon/DynamoDB/Firestore)** 고려

```js
// 모듈 스코프 캐시(의사코드)
let client;
export function getDb() {
  if (!client) client = newDbClient(process.env.DSN);
  return client;
}
```

### 관측성(Observability)

- **로그**: CloudWatch/Workers Logs/Vercel Logs
- **추적/분산 트레이싱**: AWS X-Ray, OpenTelemetry(OTel)
- **지표**: 함수 호출/오류/지연 시간, 큐 길이, 재시도 횟수

---

## 성능‧콜드스타트 최소화

| 요령 | 설명 |
|---|---|
| 번들 최소화 | ESM/Tree-shaking, 불필요 패키지 제거, `aws-sdk v3` 부분 import |
| Top-level 비용 ↓ | 거대한 초기화(ORM, 대형 JSON)를 핸들러 안에서 **게으른 초기화** |
| 메모리 튜닝 | Lambda는 **메모리↑ = CPU↑ → 콜드스타트↓/처리↑** |
| Provisioned Concurrency | 트래픽 피크 대비 사전 웜업(비용 발생) |
| 에지 활용 | 인증/리다이렉트/캐싱을 에지로 옮겨 백단 부담 ↓ |

---

## 비용 구조 이해(간단 모델)

- 서버리스 비용은 주로 **요청 수**, **실행 시간(ms)**, **메모리(MB)**에 의해 결정.
- 개략식(설명용):
$$
\text{Cost} \approx \sum_i \left( \text{Requests}_i \cdot c_r + \text{Duration}_i \cdot \text{Memory}_i \cdot c_{rm} \right)
$$
- 즉, **짧고 가벼운 함수**를 많이 쓰는 워크로드에 유리.
- 장시간 지속/고정 트래픽이면 컨테이너/서버가 유리할 수 있음.

---

## 보안 체크리스트

- **원천 비밀관리**: AWS SSM/Secrets Manager, Vercel/Workers Secrets
- **최소 권한 IAM**: S3 접근시 **특정 버킷/프리픽스**만 허용
- **입력 검증/Zod**: 요청 파라미터/바디 검사
- **CORS**: `Access-Control-Allow-Origin` 등 정확 설정
- **Idempotency**: 결제/주문 등은 **Idempotency-Key**로 중복 처리 방지
- **감사 로그**: 민감 이벤트(권한 변경, 주문 상태) 구조화 로그 남기기

---

## 로컬 개발/디버깅

| 플랫폼 | 로컬 도구 |
|---|---|
| Lambda | **SAM CLI**, **LocalStack**, **serverless-offline** |
| Vercel | `vercel dev` (Next + Functions 통합) |
| Cloudflare | **Wrangler + Miniflare** |
| Firebase | **Emulator Suite**(Auth/Firestore/Functions/Hosting) |

> **계약 테스트**: 스텁/모킹만 믿지 말고, 간단한 **스테이징**에 주기적 자동 테스트도.

---

## 실전 예제 모음

### 연락 폼 메일 전송(Production-Ready)

// **Vercel Functions + 입력 검증 + 속도 제한 + 로깅**

/**`api/contact.js`**/
```js
import { z } from 'zod';

const schema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  message: z.string().min(1).max(2000),
});

let lastHit = 0;

export default async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();
  // 간단 속도 제한(예시)
  const now = Date.now();
  if (now - lastHit < 1000) return res.status(429).json({ error: 'Too Many Requests' });
  lastHit = now;

  const parsed = schema.safeParse(req.body);
  if (!parsed.success) return res.status(400).json({ error: parsed.error.flatten() });

  try {
    // 실제 이메일 전송: Resend/SES/SendGrid 등 권장 (Nodemailer는 SMTP 환경 의존성 有)
    console.log('[CONTACT]', parsed.data); // 구조화 로그
    return res.status(200).json({ ok: true });
  } catch (e) {
    console.error('[CONTACT_ERR]', e);
    return res.status(500).json({ ok: false });
  }
}
```

### 이미지 업로드(프리사인 + 리사이징 파이프)

// **Lambda**: S3 presign 발급 → **프론트**가 직접 PUT 업로드 → **S3 이벤트**로 리사이즈 Lambda 실행

/**`lambda/presign.mjs`**/
```js
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
const s3 = new S3Client({ region: process.env.AWS_REGION });

export const handler = async (event) => {
  const { key, contentType } = JSON.parse(event.body || '{}');
  if (!key || !contentType) return { statusCode: 400, body: 'invalid' };
  const cmd = new PutObjectCommand({ Bucket: process.env.BUCKET, Key: key, ContentType: contentType });
  const url = await getSignedUrl(s3, cmd, { expiresIn: 60 });
  return { statusCode: 200, body: JSON.stringify({ url }) };
};
```

> S3 `ObjectCreated` 트리거로 리사이즈 Lambda → 썸네일 버킷에 저장.
> 에지(CloudFront Functions/Workers)에서 이미지 URL 라우팅/캐시.

### 크론 잡(스케줄) 예 — Cloudflare Workers

/**`src/cron.js`**/
```js
export default {
  async scheduled(event, env, ctx) {
    // 매 시각 실행되는 유지보수 태스크
    const resp = await fetch(env.API_ENDPOINT + '/cleanup', { method: 'POST' });
    console.log('cleanup status', resp.status);
  }
};
```

`wrangler.toml`:
```toml
[triggers]
crons = ["0 * * * *"] # 매시 0분
```

### 에지 인증 프록시 — Cloudflare → 리저널 API

/**`src/auth-proxy.js`**/
```js
export default {
  async fetch(req, env) {
    const url = new URL(req.url);
    const token = req.headers.get('Authorization')?.replace('Bearer ', '');
    if (!token) return new Response('Unauthorized', { status: 401 });

    // 경량 검증(JWT 헤더/만료 등) 후, 백엔드로 전달
    const resp = await fetch(env.REGIONAL_API + url.pathname, {
      method: req.method, headers: req.headers, body: req.body
    });
    return resp;
  }
}
```

---

## 테스트/품질 — TDD/계약/부하

- **단위 테스트**: Zod 스키마/핸들러 로직 순수 함수화 → Jest/Vitest로 검증
- **계약 테스트**: 클라이언트와 API 응답 스키마 고정 → Pact/Schema snapshot
- **부하 테스트**: k6/Artillery로 버스트/콜드스타트/스로틀 동작 확인

---

## 번들/빌드 — 작은 것이 빠르다

- **ESM 우선** + **Tree-shaking** 가능한 빌드(Vite/Rollup/esbuild)
- **서버 전용 코드 분리**: 프론트 공유 코드와 섞이지 않게 경로/에일리어스 구분
- **거대 패키지 분해**: 날짜/로다시 등 **부분 import**
- **AWS SDK v3**: 필요한 클라이언트만 가져오기

```js
// 나쁨: import AWS from 'aws-sdk'
// 좋음:
import { S3Client } from '@aws-sdk/client-s3';
```

---

## 운영 자동화(CI/CD)

- **Git Push → 미리보기/프로덕션**: Vercel/Netlify는 기본 탑재
- **AWS**: GitHub Actions → CDK/SAM 배포, 환경별 스택(Dev/Prod) 분리
- **비밀 주입**: OpenID Connect(OIDC)로 클라우드 자격증명 단기 발급(키 저장 X)

---

## 언제 Serverless가 정답이 아닐까?

- **항시 연결/저지연**: 초저지연 WebSocket 대규모 상시 연결(게임/트레이딩)
- **대용량 스트리밍/배치**: 길고 무거운 작업(영상 인코딩) — **Batch/ECS/배포형 서버** 고려
- **대규모 24/7 API**: 요청이 상시 폭주해 함수 단가가 누적되면 컨테이너/쿠버네티스가 유리

---

## 전체 비교표(요약)

| 항목 | 장점 | 단점 | 우수 사례 |
|---|---|---|---|
| Serverless (Regional) | 인프라 자동, 확장 용이, DB 접근 | 콜드스타트, 구성 복잡 | 결제/업무 API, 백오피스, 웹훅 |
| Edge Functions | 초저지연, 글로벌 배포, 캐시/프록시 | Node 호환성 제한, 파일/소켓 제약 | 인증/리디렉션, A/B 테스트, 헤더 변조 |
| 전통 서버/컨테이너 | 장기 실행, 자유도 높음 | 운영비/복잡도, 오토스케일 튜닝 | 실시간/대용량/장시간 잡 |

---

## 빠른 시작 “템플릿” 3종

### A) **Vercel Node 함수 + Zod + CORS 헤더**

```js
// api/hello.js
import { z } from 'zod';
const q = z.object({ name: z.string().default('world') });

export default function handler(req, res) {
  res.setHeader('Access-Control-Allow-Origin', '*');
  const parsed = q.safeParse(req.query);
  if (!parsed.success) return res.status(400).json({ error: 'bad query' });
  res.status(200).json({ message: `Hello, ${parsed.data.name}` });
}
```

### B) **Lambda + API Gateway + 에러 안전 래퍼**

```js
const wrap = (fn) => async (event) => {
  try { return await fn(event); }
  catch (e) { console.error(e); return { statusCode: 500, body: 'internal' }; }
};

export const handler = wrap(async (event) => {
  const name = (event.queryStringParameters || {}).name || 'world';
  return { statusCode: 200, body: JSON.stringify({ message: `Hello, ${name}` }) };
});
```

### C) **Cloudflare Worker 에지 캐시 프록시**

```js
export default {
  async fetch(req, env) {
    const url = new URL(req.url);
    const cacheKey = new Request(url.toString(), req);
    const cache = caches.default;

    let res = await cache.match(cacheKey);
    if (!res) {
      res = await fetch('https://news.ycombinator.com');
      res = new Response(res.body, res);
      res.headers.append('Cache-Control', 'max-age=60');
      ctx.waitUntil(cache.put(cacheKey, res.clone()));
    }
    return res;
  }
};
```

---

## 결론

- JS/TS 개발자에게 서버리스는 **API·웹훅·파일 업로드·정기 작업**을 빠르게 만들 수 있는 **가장 짧은 경로**입니다.
- **입력 검증/오류/관측/보안/비용**의 다섯 축만 지키면, 작은 함수들의 조합으로 **유지보수 가능한 백엔드**를 운영할 수 있습니다.
- 에지와 리저널을 적절히 섞어 **지연·비용·복잡도** 균형을 맞추세요.

---

## 참고 링크(정리)

- AWS Lambda: https://aws.amazon.com/lambda/
- Vercel Functions: https://vercel.com/docs/functions
- Cloudflare Workers: https://developers.cloudflare.com/workers/
- Firebase Functions: https://firebase.google.com/docs/functions
- Netlify Functions: https://docs.netlify.com/functions/overview/
- Serverless Patterns: https://serverlessland.com/patterns
