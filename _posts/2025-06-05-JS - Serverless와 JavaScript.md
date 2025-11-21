---
layout: post
title: JavaScript - Serverless와 JavaScript
date: 2025-06-05 20:20:23 +0900
category: JavaScript
---
# Serverless와 JavaScript: 서버 없이도 백엔드를 구축하는 방법

## 핵심 요약

- **Serverless = 코드(함수) + 이벤트(트리거) + 관리형 인프라**
- **JavaScript/TypeScript**는 대부분의 서버리스 플랫폼에서 1급 시민
- 작은 기능부터 시작해 **메일 전송·이미지 처리·웹훅·OAuth 콜백·Cron 작업·GraphQL/REST API** 까지 빠르게 구성
- 주의점: **Cold Start, 상태 없음, 실행 시간/메모리 제한, DB 연결 관리, 디버깅/관측**
- 해결책: **Edge 실행, 번들 최적화, 캐시, 큐·스케줄러, 서버리스 친화 DB, 관측(로그/트레이싱)**

---

## Serverless란 무엇인가

> **Serverless(서버리스)**는 서버의 준비·패치·확장·장애복구를 클라우드가 맡고, 개발자는 **핵심 로직만 함수로 작성**하는 실행 모델이다.

- 실행 단위: **FaaS(Function as a Service)** — Lambda/Functions/Workers
- 이벤트 소스: HTTP 요청, 메시지 큐, 스토리지 업로드, 스케줄(Cron), DB 변경, 웹훅 등
- 과금: 호출 수, 실행 시간(ms), 메모리(GB-s) 기반의 **종량제**

수식으로 간단히 보면(개념적):
$$
\text{월비용} \approx (\text{호출수}) \times (\text{평균 실행시간}) \times (\text{메모리 GB}) \times \text{단가}
$$

---

## JS와 Serverless가 잘 맞는 이유

| 이유 | 설명 |
|---|---|
| Node.js 이벤트 루프 | 이벤트 기반·IO 바운드에 최적 |
| npm 생태계 | 인증, 메일, 이미지, DB 등 라이브러리 풍부 |
| FE↔BE 통합 | 동일 언어로 **API + UI** 한 프로젝트에서 관리 |
| Edge 친화 | Cloudflare/Next Edge 같은 **ESM/웹표준 런타임**와 궁합 |

---

## 대표 플랫폼과 최소 예제

### AWS Lambda (Node.js 18/20)

```js
// handler.js
exports.handler = async (event) => {
  // event.queryStringParameters, event.body 등
  return {
    statusCode: 200,
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ ok: true, at: new Date().toISOString() }),
  };
};
```

- **API Gateway**와 연결해 HTTP API 제공
- **S3/DynamoDB/SNS/SQS** 등과 네이티브 통합
- IaC: **Serverless Framework, AWS SAM, CDK**로 배포 자동화

---

### Vercel Serverless/Edge Functions

```js
// api/hello.js  (Node runtime)
export default function handler(req, res) {
  res.status(200).json({ message: 'Hello from Vercel' });
}
```

```js
// api/edge-hello.js  (Edge runtime)
export const config = { runtime: 'edge' };
export default async function handler(req) {
  return new Response(JSON.stringify({ edge: true }), {
    headers: { 'content-type': 'application/json' },
  });
}
```

- Next.js의 **API Routes** 및 **Route Handlers**와 동일 철학
- **Edge**는 콜드스타트 미미, 글로벌 POP로 저지연

---

### Cloudflare Workers (에지 런타임, Deno 계열)

```js
export default {
  async fetch(request, env, ctx) {
    return new Response('Hello from the edge!', {
      headers: { 'content-type': 'text/plain' },
    });
  },
};
```

- 전 세계 POP에서 **초저지연** 실행
- 저장소: **KV, Durable Objects, D1(SQLite on edge), R2(객체 스토리지)**

---

### Firebase Cloud Functions (Node)

```js
const functions = require('firebase-functions');

exports.helloWorld = functions.https.onRequest((req, res) => {
  res.send('Hello from Firebase!');
});
```

- **Auth, Firestore, Storage** 등 Firebase 전체와 자연스러운 통합
- 에뮬레이터로 **로컬 개발** 강력

---

## 실전 시나리오별 레퍼런스 구현

### Contact Form → 이메일 전송 (Vercel)

```js
// api/sendEmail.js
import nodemailer from 'nodemailer';

export default async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();
  const { name, email, message } = req.body ?? {};
  if (!name || !email || !message) return res.status(400).json({ error: 'invalid' });

  const transporter = nodemailer.createTransport({
    service: 'Gmail',
    auth: { user: process.env.GMAIL_USER, pass: process.env.GMAIL_PASS },
  });

  await transporter.sendMail({
    from: email,
    to: 'admin@example.com',
    subject: `Contact: ${name}`,
    text: message,
  });

  res.status(200).json({ ok: true });
}
```

- 프론트에서 `fetch('/api/sendEmail', { method: 'POST', body: JSON.stringify(...) })`
- **.env** 로 민감정보 분리, Vercel **Project → Settings → Environment Variables**에 등록

---

### 이미지 업로드 → 썸네일 생성 (S3 + Lambda)

1) 사용자가 **S3 버킷**에 업로드
2) 버킷 **ObjectCreated** 이벤트 → **Lambda** 트리거
3) Lambda에서 Sharp로 썸네일 생성 → **S3에 저장**

```js
// index.mjs (Node.js 20 + ESM)
import sharp from 'sharp';
import { S3Client, GetObjectCommand, PutObjectCommand } from '@aws-sdk/client-s3';

const s3 = new S3Client({});
export const handler = async (event) => {
  for (const record of event.Records) {
    const Bucket = record.s3.bucket.name;
    const Key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));
    const Body = await s3.send(new GetObjectCommand({ Bucket, Key }))
      .then(r => r.Body.transformToByteArray());

    const thumb = await sharp(Body).resize({ width: 480 }).webp({ quality: 85 }).toBuffer();
    const outKey = Key.replace(/(\.[a-z]+)$/i, '_thumb.webp');

    await s3.send(new PutObjectCommand({
      Bucket, Key: outKey, Body: thumb, ContentType: 'image/webp'
    }));
  }
  return { ok: true };
};
```

- 이미지 처리 라이브러리는 **Layer/ESBuild 번들** 최적화 필요
- 대용량 처리 시 **SQS** 버퍼링, **Concurrency** 제한 고려

---

### + 서명 검증 (Vercel Edge)

```js
// app/api/stripe/route.js (Next 13+ Route Handler, Edge)
export const runtime = 'edge';
export async function POST(req) {
  const payload = await req.text();
  const signature = req.headers.get('stripe-signature');
  // Stripe SDK 대신 경량 검증 로직/라이브러리 사용(Edge 호환)
  const ok = verifyStripeSignature(payload, signature, process.env.STRIPE_WEBHOOK_SECRET);
  if (!ok) return new Response('sig error', { status: 400 });

  // 이벤트 파싱 및 처리
  return new Response(JSON.stringify({ received: true }), {
    headers: { 'content-type': 'application/json' },
  });
}
```

- Webhook은 **idempotency** 고려(중복 도착 대비)
- 검증 실패는 즉시 4xx 반환, 성공 후 빠르게 ACK → 백그라운드 Queue로 처리 분리 권장

---

### Cron 스케줄러(Cloudflare)

```js
// wrangler.toml
name = "cron-demo"
main = "src/index.js"
compatibility_date = "2025-01-01"

[triggers]
crons = ["0 9 * * *"]  # 매일 09:00 UTC

// src/index.js
export default {
  async scheduled(event, env, ctx) {
    // 리포트 생성·메일 발송·캐시 재빌드 등
    await env.MY_KV.put('last-run', new Date().toISOString());
  }
};
```

- 플랫폼 제공 Cron(Cloudflare/Vercel/Scheduler/CloudWatch)로 **주기 작업** 자동화
- 결과 저장은 KV/DB에 로그 남기고, 실패는 재시도·알림 연계

---

### Edge 캐시 프록시(API 캐싱, Cloudflare)

```js
export default {
  async fetch(req, env) {
    const url = new URL(req.url);
    if (url.pathname.startsWith('/news')) {
      const cacheKey = new Request(req.url, req);
      const cache = caches.default;
      let res = await cache.match(cacheKey);
      if (!res) {
        res = await fetch('https://api.example.com/news');
        res = new Response(res.body, res); // 복제
        res.headers.set('cache-control', 'public, s-maxage=60'); // 1분 CDN 캐시
        ctx.waitUntil(cache.put(cacheKey, res.clone()));
      }
      return res;
    }
    return fetch(req);
  }
};
```

- API 프록시 + 캐시 제어로 **백엔드 부하 감소**와 **응답 시간 개선**
- stale-while-revalidate 패턴 활용 시 UX 향상

---

## 데이터베이스·세션·상태 관리

서버리스는 **무상태**. 상태는 외부로 분리한다.

- KV/캐시: **Cloudflare KV**, Redis(SaaS), Upstash
- RDB: Serverless 친화형(커넥션 풀 포함) — **PlanetScale(MySQL)**, **Neon(Postgres)**, **Aurora Serverless v2**, **Supabase(Postgres)**
- 세션: **JWT(쿠키 HttpOnly + SameSite)** 또는 **비밀 키로 서명한 토큰**. 서버 저장형이면 Redis/DB에 세션 저장
- Connection 관리: Node 풀링은 함수 재사용이 약해 비용 큼 → **HTTP 기반 드라이버** or **서버리스 프록시** 사용

예: PlanetScale + Prisma
```bash
npx prisma init
# DATABASE_URL=...

```

```ts
// db.ts
import { PrismaClient } from '@prisma/client';
export const prisma = globalThis.prisma ?? new PrismaClient();
if (process.env.NODE_ENV !== 'production') globalThis.prisma = prisma;
```

> Lambda 재사용 컨테이너에서 커넥션 누수 방지. Neon(HTTP) 드라이버 등 **서버리스 전용 커넥터** 권장.

---

## 인증·권한 (AuthN/AuthZ)

- OAuth: GitHub/Google 등 **OAuth Redirect**와 **Callback**을 서버리스 함수로 처리
- JWT: 만료시간, audience, issuer 검증. **키 회전(JWKs)** 캐시
- 세션 고정 방지, CSRF(쿠키 사용 시 **Double Submit Token** 또는 SameSite=strict)
- RLS(Row Level Security): Supabase/Postgres로 세밀 권한

Next.js + Auth.js 간단 예시
```ts
// app/api/auth/[...nextauth]/route.ts
export { GET, POST } from 'next-auth';
```

구현 디테일은 Provider/Adapter 설정에 따라 상이.

---

## 성능 최적화·콜드스타트 대응

- Edge 런타임(Cloudflare, Vercel Edge) 사용 → **콜드스타트 최소화**
- 번들 최적화: **ESBuild/SWC**, tree-shaking, **최소 의존성**, 동적 import
- 핫경로에 **네이티브 API** 우선(Fetch, Web Crypto). Node 전용 패키지는 Edge에서 불가
- **Keep it short**: 함수 한 번 실행에 작업 분할, 큰 잡은 **큐(Queue)** 로 넘기기
- 캐시: CDN/Edge 캐시, **Cache-Control** 헤더 설계

---

## 보안·비밀관리·컴플라이언스

- Secrets: 플랫폼의 **환경변수/시크릿 매니저** 사용(AWS Secrets Manager, Vercel Env, Cloudflare Secrets)
- 입력 검증: Zod/Yup 등 스키마 검증
- 아웃바운드: egress 제한·고정 IP 필요 시 NAT/게이트웨이 구성
- CSP/Trusted Types: XSS 억제, HTML 인젝션 엄금
- 서명 검증(Webhook): **HMAC, 시그니처 타임스탬프** 확인, 재생공격 방지

---

## 로컬 개발·테스트·관측

### 로컬/스테이징

- Firebase: **Emulators** (Auth/Functions/Firestore/Storage)
- Cloudflare: **wrangler dev** (로컬/리모트 미러)
- AWS: **SAM CLI / Localstack** (가급적 실제 스테이징으로 최종 확인)
- Vercel: **vercel dev** 로 로컬 라우팅

### 단위/통합 테스트

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';
export default defineConfig({ test: { environment: 'node' } });
```

```ts
// api/__tests__/sendEmail.test.ts
import { describe, it, expect, vi } from 'vitest';
import handler from '../sendEmail';

vi.mock('nodemailer', () => ({
  default: { createTransport: () => ({ sendMail: vi.fn().mockResolvedValue({}) }) }
}));

it('should reject bad body', async () => {
  const req = { method: 'POST', body: {} };
  const res = mockRes();
  await handler(req, res);
  expect(res.status).toHaveBeenCalledWith(400);
});
```

### 모니터링/로깅/트레이싱

- 플랫폼 로그 + 구조화 로그(JSON)
- 분산 트레이싱(OpenTelemetry) + 벤더(Sentry, Datadog)
- 지표: 호출 수, 에러율, p95 응답시간, 콜드스타트 비율

---

## 배포 자동화(IaC/CI)

### Serverless Framework (AWS)

```yaml
# serverless.yml

service: mailer
provider:
  name: aws
  runtime: nodejs20.x
functions:
  send:
    handler: handler.send
    events:
      - httpApi:
          path: /send
          method: post
```

```bash
npx serverless deploy
```

### Cloudflare

```bash
npm i -D wrangler
npx wrangler init
npx wrangler deploy
```

### Vercel

```bash
npm i -g vercel
vercel
vercel --prod
```

- CI: GitHub Actions에서 **lint/test/build/deploy** 자동화
- 환경 변수는 **각 환경(Dev/Preview/Prod)** 별로 분리

---

## 비용 모델링과 최적화

개념적 비용:
$$
\text{Cost} = \sum_i \left( N_i \times T_i \times M_i \times P \right) + \text{네트워크/게이트웨이/DB}
$$

- \(N_i\): 호출 수, \(T_i\): 평균 실행시간(초), \(M_i\): 메모리 GB, \(P\): GB-sec 단가
- 최적화 포인트
  - 실행시간 ↓: **I/O 캐시, 데이터 지역성**
  - 메모리 ↓: 적정 메모리(오버프로비저닝 지양)
  - 호출 수 ↓: **Edge 캐시**, **배치 처리**, Webhook 통합 감축
  - 게이트웨이 비용: API Gateway 대신 **함수 URL/프록시** 고려(트래픽 패턴에 따라)

---

## 안티패턴과 회피법

| 안티패턴 | 문제 | 해결책 |
|---|---|---|
| 거대한 “몬올리식” 함수 | 콜드스타트 증가, 코드 복잡 | 도메인별로 작은 함수로 분할, 큐/워크플로우 |
| RDB 커넥션 남발 | 커넥션 고갈, 지연 | 서버리스 친화 DB(HTTP 드라이버, 프록시), 연결 재사용 |
| 큰 파일 동기 처리 | 타임아웃 | presigned URL, 비동기 파이프라인(S3→Event→Lambda) |
| 동기 외부 API 다단 호출 | 지연·비용 상승 | 캐시, 배치, 병렬·비동기 플로우, Circuit Breaker |
| 비밀키 하드코딩 | 유출·컴플라이언스 | 환경 변수/시크릿 매니저 |
| 무관측 | 장애 원인 불명 | 로그·트레이싱·지표 필수화 |

---

## 아키텍처 패턴(캔드 레시피)

### 백엔드 for 프런트엔드(BFF)

- Next.js Route Handlers(Vercel) 또는 Workers에서 **외부 API 합치기 + 캐시**
- 장점: 브라우저에 키/비밀 노출 방지, 응답 스키마 통일

### 이벤트 드리븐

- S3 업로드 → 이벤트 → 썸네일/인덱싱 → DB 갱신
- 느슨한 결합, 장애 격리

### CQRS + 캐시

- 쓰기 경로는 DB/큐, 읽기는 Edge 캐시/KV로 초저지연 제공
- ISR/리밸리데이션(Next)로 자연스럽게 반영

---

## 프런트 개발자를 위한 빠른 선택 가이드

| 상황 | 권장 스택 |
|---|---|
| 포트폴리오/랜딩 + 폼/메일 | Next.js on Vercel + API Route(메일) + KV |
| 대시보드/CRUD + 인증 | Next.js + Auth.js + PlanetScale/Neon + Prisma |
| 이미지/미디어 파이프라인 | S3/R2 + 이벤트 Lambda/Workers + 큐 |
| 글로벌 초저지연 API | Cloudflare Workers + KV/D1 + 캐시 |
| Firebase 친화 앱 | Firebase Hosting + Functions + Firestore/Auth |

---

## 실전 프로젝트 뼈대(Next.js + Vercel + Prisma)

```
my-app/
├─ app/
│  ├─ api/
│  │  ├─ contact/route.ts        # POST → send email
│  │  └─ posts/route.ts          # CRUD → DB
│  └─ page.tsx                   # UI
├─ prisma/
│  └─ schema.prisma
├─ src/
│  └─ db.ts                      # Prisma client (serverless safe)
├─ .env                          # DB_URL, MAIL creds (local)
├─ package.json
└─ vercel.json                   # ISR, regions, routes(optional)
```

```ts
// app/api/posts/route.ts
import { prisma } from '@/src/db';

export async function GET() {
  const posts = await prisma.post.findMany({ orderBy: { createdAt: 'desc' } });
  return Response.json(posts, { headers: { 'cache-control': 'public, s-maxage=30' } });
}

export async function POST(req: Request) {
  const body = await req.json();
  const created = await prisma.post.create({ data: { title: body.title } });
  return Response.json(created, { status: 201 });
}
```

---

## 체크리스트(런칭 전)

- [ ] 모든 엔드포인트 스키마 검증(Zod)
- [ ] 비밀/환경변수 분리, 프리뷰/프로덕션 분리
- [ ] 캐시 정책(Cache-Control/SWR/ISR) 정의
- [ ] DB 커넥션/풀링 확인(서버리스 대응)
- [ ] 로깅·알림·에러 추적 연결
- [ ] 성능 측정: p95 응답, 콜드스타트 비율, 비용 가늠
- [ ] **재시도·아이템포턴시** 설계(결제/웹훅 등)

---

## 비용 감각 잡기(예시 계산)

가정: 월 200만 호출, 평균 80ms, 256MB(=0.25GB), GB-sec 단가 \(P\)

$$
\text{GB-sec} = N \times T \times M = 2{,}000{,}000 \times 0.08 \times 0.25 = 40{,}000
$$

$$
\text{계산비용} = 40{,}000 \times P \quad(+\ \text{게이트웨이/네트워크/DB 비용})
$$

- **캐시로 호출수를 50% 절감**하면 비용도 거의 절반 감소
- Edge에 올려 **응답·콜드스타트** 동시 개선 가능

---

## 결론

Serverless는 프런트엔드 개발자가 **백엔드의 80%**를 즉시 구현·운영하게 해주는 강력한 레버리지다.
남은 20%(고성능 장기 잡, 장시간 스트리밍, 대규모 배치)는 **이벤트/큐/엣지/워크플로우**와 조합하거나, 필요 시 **컨테이너/일반 서버**와 혼용하자.
핵심은 **작게 시작 → 계측 → 캐시/큐/스케일 아웃 → 비용과 성능의 균형**이다.

---

## 참고 자료

- AWS Lambda: https://docs.aws.amazon.com/lambda/
- Vercel Functions: https://vercel.com/docs/functions
- Cloudflare Workers: https://developers.cloudflare.com/workers/
- Firebase Functions: https://firebase.google.com/docs/functions
- Serverless Framework: https://www.serverless.com/
- Next.js(Edge/Route Handlers/ISR): https://nextjs.org/docs
- OpenTelemetry: https://opentelemetry.io/
