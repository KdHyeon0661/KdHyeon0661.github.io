---
layout: post
title: JavaScript - Serverless와 JavaScript
date: 2025-06-05 20:20:23 +0900
category: JavaScript
---
# ☁️ Serverless와 JavaScript: 서버 없이도 백엔드를 구축하는 방법

---

## 📌 Serverless란?

> **Serverless(서버리스)**는 서버를 직접 구축하거나 운영하지 않고, **클라우드 제공자가 서버의 실행, 확장, 유지보수를 대신 수행하는 구조**입니다.

- 서버는 존재하지만, 사용자는 **"코드만"** 작성하면 됩니다.
- 실행 단위는 주로 함수(Function)이며, 대표적으로 **FaaS (Function as a Service)** 형태로 제공됩니다.

---

## 🚀 JavaScript와 Serverless: 궁합이 좋은 이유

| 이유 | 설명 |
|------|------|
| ✅ Node.js 지원 | 대부분의 클라우드 플랫폼이 Node.js를 기본으로 지원 (JavaScript로 작성 가능) |
| ✅ npm 에코시스템 | 수많은 라이브러리와 도구를 그대로 사용 가능 |
| ✅ 프론트+백 통합 | 프론트엔드 개발자가 백엔드 API도 빠르게 구현 |
| ✅ 클라우드 친화적 | JS는 이벤트 기반 처리에 적합하여 서버리스와 잘 맞음 |

---

## 💡 대표 Serverless 플랫폼과 JavaScript 사용 예시

---

### 1️⃣ AWS Lambda (Node.js 기반)

```js
exports.handler = async (event) => {
  return {
    statusCode: 200,
    body: JSON.stringify({ message: "Hello from Lambda!" }),
  };
};
```

- `Node.js 18`까지 공식 지원
- `API Gateway`를 통해 HTTP 요청 처리 가능
- `S3`, `DynamoDB`, `SNS` 등 AWS 서비스와 연동

---

### 2️⃣ Vercel Functions

```js
// api/hello.js
export default function handler(req, res) {
  res.status(200).json({ message: 'Hello from Vercel!' });
}
```

- `Next.js` 기반 프레임워크의 서버리스 API 라우트
- `Edge Functions`로 초고속 응답 지원
- 배포 자동화 + 글로벌 CDN 포함

---

### 3️⃣ Firebase Cloud Functions

```js
const functions = require('firebase-functions');

exports.helloWorld = functions.https.onRequest((req, res) => {
  res.send("Hello from Firebase!");
});
```

- Firebase 인증, DB, Storage와 자연스럽게 연동
- TypeScript 기반 지원도 매우 뛰어남

---

### 4️⃣ Cloudflare Workers (JavaScript or TypeScript)

```js
export default {
  async fetch(request) {
    return new Response("Hello from the edge!", {
      headers: { "content-type": "text/plain" },
    });
  },
};
```

- 전 세계에 분산된 **에지 네트워크**에서 실행
- Cold Start가 거의 없음
- Deno 기반 실행 환경

---

## 🧪 실전 예제: 연락처 폼 이메일 전송 API (Vercel 기준)

```js
// api/sendEmail.js
import nodemailer from 'nodemailer';

export default async function handler(req, res) {
  const { name, email, message } = req.body;

  const transporter = nodemailer.createTransport({
    service: 'Gmail',
    auth: {
      user: process.env.GMAIL_USER,
      pass: process.env.GMAIL_PASS,
    },
  });

  await transporter.sendMail({
    from: email,
    to: 'admin@example.com',
    subject: `New message from ${name}`,
    text: message,
  });

  res.status(200).json({ success: true });
}
```

- 프론트엔드에서 호출 시 백엔드 없이도 메일 전송
- `.env`로 환경 변수 관리

---

## ⚖️ Serverless의 장단점

### ✅ 장점

| 항목 | 설명 |
|------|------|
| 💡 운영 부담 없음 | 인프라 설정이나 유지보수 불필요 |
| 📈 자동 확장 | 트래픽 급증에도 자동 대응 |
| 💰 종량 과금 | 사용한 만큼만 비용 부과 |
| ⚙️ 이벤트 기반 처리 | 비동기 작업에 적합 (예: DB 변경 시 자동 실행) |

### ❌ 단점

| 항목 | 설명 |
|------|------|
| 🕒 Cold Start | 일정 시간 동안 사용 없으면 첫 실행이 느릴 수 있음 |
| 🔒 상태 없음 | 세션 저장 어려움 → 외부 DB나 캐시 필요 |
| ⏱️ 실행 제한 | 보통 10~15초 내에 작업 완료해야 함 |
| 🧪 디버깅 | 로컬에서 디버깅이 까다로운 경우 많음 |

---

## 🔧 Serverless 활용 예시

| 용도 | 예시 |
|------|------|
| 이메일 전송 | Contact Form 처리 |
| 이미지 변환 | 썸네일 생성, 압축 |
| 주기 작업 | 크론 기반 스케줄링 (ex. 리포트 전송) |
| 인증 처리 | JWT 발급, OAuth 콜백 처리 |
| API Proxy | 외부 API 호출 및 캐싱 |

---

## 📦 JavaScript로 Serverless 개발 시 필수 도구

| 도구 | 설명 |
|------|------|
| `Serverless Framework` | AWS Lambda 등 배포 자동화 도구 |
| `Vercel CLI` | Vercel Functions 배포 |
| `Firebase CLI` | Functions 배포 및 관리 |
| `esbuild / SWC` | 빠른 번들링 (Cold Start 줄이기) |
| `dotenv` | 환경 변수 관리 |

---

## 🧠 Serverless와 프론트엔드 개발

### ✅ 프론트 개발자가 직접 할 수 있는 백엔드:

- REST API 제작
- 데이터 저장 (Firebase, Supabase, PlanetScale 등과 연계)
- 인증 처리 (OAuth, JWT)
- 메일 전송
- 이미지 처리

### 🧱 통합 개발 경험

- Next.js + API Route → 서버리스 + SSR + 프론트가 하나로 통합
- JAMStack 환경에서도 필수 도구

---

## 💸 비용 구조 예시 (AWS 기준)

| 항목 | 무료 | 단가 |
|------|------|------|
| 호출 수 | 월 100만 건 | 이후 1M당 약 $0.20 |
| 실행 시간 | 월 400,000 GB-sec | 이후 1GB-sec당 $0.00001667 |
| 트래픽 | 개별 서비스 과금 (API Gateway 등) |

---

## ✅ 마무리 요약

| 항목 | 내용 |
|------|------|
| Serverless란? | 서버를 직접 운영하지 않고 코드만 작성해 클라우드에서 실행 |
| JS와의 궁합 | Node.js 기반 환경이 많아 JavaScript 친화적 |
| 플랫폼 | AWS Lambda, Vercel, Firebase, Cloudflare 등 |
| 장점 | 빠른 개발, 비용 절감, 자동 확장 |
| 주의사항 | Cold Start, 상태 관리 필요, 디버깅 복잡 |

---

## 📚 참고 자료

- [AWS Lambda 공식 문서](https://docs.aws.amazon.com/lambda/)
- [Vercel Functions](https://vercel.com/docs/functions)
- [Cloudflare Workers](https://developers.cloudflare.com/workers/)
- [Firebase Functions](https://firebase.google.com/docs/functions)
- [Serverless Framework](https://www.serverless.com/)