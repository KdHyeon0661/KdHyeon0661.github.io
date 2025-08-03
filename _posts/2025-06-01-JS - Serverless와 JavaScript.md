---
layout: post
title: JavaScript - Serverless와 JavaScript
date: 2025-06-01 23:20:23 +0900
category: JavaScript
---
# ☁️ Serverless와 JavaScript: 프론트엔드 개발자를 위한 서버리스 입문

---

## 📌 Serverless란?

> **Serverless(서버리스)**는 개발자가 직접 서버를 구축하거나 관리하지 않고, **클라우드 제공자가 실행 환경을 대신 관리해주는 컴퓨팅 모델**입니다.

- 서버가 없다는 뜻이 아니라 “**서버 관리를 신경 쓰지 않아도 되는**” 환경을 의미
- 대표 플랫폼: **AWS Lambda, Google Cloud Functions, Azure Functions, Vercel, Netlify, Cloudflare Workers**

---

## 🔧 Serverless의 작동 방식

1. 사용자가 API 호출, 이벤트 발생 (HTTP, Cron, DB 트리거 등)
2. 클라우드 서비스가 함수를 **자동 실행**
3. 실행이 끝나면 **자원 반환** (종량 과금)

---

## ✅ JavaScript와 Serverless의 찰떡궁합

| 이유 | 설명 |
|------|------|
| ✅ Node.js 기반 | 대부분의 Serverless 플랫폼은 Node.js 지원 (JavaScript 사용 가능) |
| ✅ 빠른 초기 진입 | `npm` 환경 그대로 활용 가능 |
| ✅ 프론트와 백의 연결 | 프론트엔드 개발자가 백엔드 API도 직접 구현 가능 |
| ✅ 번들 최소화 | JS 함수 단위로 구성하므로 경량화에 유리 |

---

## 💡 주요 Serverless 플랫폼에서의 JS 사용 예

---

### 1️⃣ AWS Lambda + Node.js

```js
exports.handler = async (event) => {
  return {
    statusCode: 200,
    body: JSON.stringify({ message: "Hello from Lambda!" }),
  };
};
```

- `handler` 함수가 엔트리포인트
- AWS API Gateway와 연동하여 HTTP API 구축 가능
- `Serverless Framework`, `AWS CDK` 등으로 배포 자동화 가능

---

### 2️⃣ Vercel Functions (Next.js 포함)

파일 기반 라우팅으로 함수 작성

```js
// api/hello.js
export default function handler(req, res) {
  res.status(200).json({ message: "Hello from Vercel!" });
}
```

- `api/` 디렉토리 하위에 JS 함수 파일 작성
- 자동 배포 + CDN 연동
- 매우 빠른 응답 속도 (에지 네트워크 기반)

---

### 3️⃣ Firebase Functions

```js
const functions = require("firebase-functions");

exports.helloWorld = functions.https.onRequest((req, res) => {
  res.send("Hello from Firebase!");
});
```

- 실시간 데이터베이스, 인증 등과 쉽게 연동
- Firebase Hosting과 자연스럽게 연결 가능

---

### 4️⃣ Cloudflare Workers (에지 컴퓨팅)

```js
export default {
  async fetch(request) {
    return new Response("Hello from the edge!", {
      headers: { "content-type": "text/plain" },
    });
  },
};
```

- 빠른 에지 실행
- Deno 기반 (Node.js와 약간 다름)
- 1ms Cold Start, 글로벌 배포

---

## 🎯 언제 Serverless를 선택해야 할까?

### ✅ 적합한 경우

- 프론트엔드 중심 프로젝트에서 간단한 백엔드 필요
- 특정 이벤트 기반 처리 (ex: 이미지 리사이징, 이메일 전송)
- 트래픽이 불규칙한 서비스
- 초기 인프라 비용을 아끼고 싶은 경우

### ❌ 부적합한 경우

- 항상 실행되어야 하는 백엔드 서버
- 초저지연 실시간 처리 (게임, 고빈도 거래 등)
- 대용량 파일 처리 (용량/시간 제한 존재)

---

## 🧪 실습 예시: Contact Form 메일 전송 API

```js
// Vercel - api/sendMail.js
import nodemailer from 'nodemailer';

export default async function handler(req, res) {
  const { name, email, message } = req.body;

  const transporter = nodemailer.createTransport({
    service: 'Gmail',
    auth: {
      user: process.env.GMAIL_ID,
      pass: process.env.GMAIL_PW,
    },
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

- 간단한 `POST` 요청으로 메일 전송 가능
- 프론트엔드와 백엔드가 동일 프로젝트 내에서 동작

---

## 💸 비용 구조

| 항목 | 설명 |
|------|------|
| 요청 수 | ex: AWS Lambda는 월 100만 요청까지 무료 |
| 실행 시간 | 1ms 단위 과금 |
| 메모리 크기 | 128MB ~ 1024MB 설정 가능 |

👉 **짧고 가벼운 작업이 많을수록 이득**

---

## ⚖️ 장점 vs 단점

### ✅ 장점

- 서버 유지보수 X
- 탄력적 확장 (트래픽 증가 대응)
- 빠른 배포/롤백
- 초기 비용 절감
- JavaScript 친화적

### ❌ 단점

- Cold Start 이슈 (특히 AWS)
- 디버깅/로컬 테스트 어려움
- 실행 시간/메모리 제한
- 상태 없는 구조 (DB/세션 관리 별도)

---

## 🔗 참고 도구 및 플랫폼

| 플랫폼 | 특징 |
|--------|------|
| [AWS Lambda](https://aws.amazon.com/lambda/) | 가장 성숙한 서버리스 환경 |
| [Vercel Functions](https://vercel.com/docs/functions) | Next.js 기반, 프론트엔드와 통합 |
| [Cloudflare Workers](https://developers.cloudflare.com/workers/) | 초고속 에지 컴퓨팅 |
| [Firebase Functions](https://firebase.google.com/docs/functions) | Firebase 생태계와 통합 |
| [Netlify Functions](https://docs.netlify.com/functions/overview/) | JAMStack + CI/CD 통합 |
| [Deno Deploy](https://deno.com/deploy) | TypeScript 우선, Deno 런타임 |

---

## ✅ 마무리 요약

| 항목 | 설명 |
|------|------|
| Serverless란? | 서버 없이 함수 단위로 클라우드에서 실행 |
| 왜 JS와 궁합이 좋을까? | Node.js 기반, npm 생태계, 빠른 개발 |
| 어디에 쓰일까? | 폼 처리, 이미지 변환, 이메일 전송, API 서버 |
| 주의할 점 | 실행 제한, Cold Start, 상태 관리 |

---

## 📚 더 읽을거리

- [Serverless Handbook (free)](https://serverlesshandbook.dev/)
- [Awesome Serverless](https://github.com/anaibol/awesome-serverless)
- [Vercel Serverless Functions Docs](https://vercel.com/docs/functions)
- [AWS Serverless Examples](https://github.com/aws-samples)