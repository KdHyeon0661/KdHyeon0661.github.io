---
layout: post
title: AspNet - Blazor소개 및 비교
date: 2025-04-29 21:20:23 +0900
category: AspNet
---
# 🧩 Blazor 소개 및 기존 기술들과의 비교

---

## ✅ 1. Blazor란?

**Blazor**는 Microsoft에서 만든 **C# 기반 웹 UI 프레임워크**예요.  
JavaScript 없이 **C#과 Razor 문법으로 브라우저 UI를 구성**할 수 있게 해줍니다.

### 📌 이름 유래
- **BLA**(Browser) + **ZOR**(Razor)의 합성어

---

## 🛠 2. Blazor의 주요 특징

| 항목 | 설명 |
|------|------|
| ✅ C# 기반 | 전체 UI를 C#으로 작성 가능 |
| ✅ Razor 문법 사용 | ASP.NET Razor 문법 그대로 |
| ✅ SPA 지원 | React/Vue처럼 Single Page App 구조 |
| ✅ WebAssembly / SignalR | 클라이언트 실행 방식에 따라 선택 가능 |
| ✅ 컴포넌트 기반 | 재사용 가능한 UI 단위로 구성 |

---

## 🔀 3. Blazor 종류 (실행 방식에 따른 구분)

| Blazor 유형 | 실행 방식 | 특징 |
|-------------|-----------|------|
| **Blazor Server** | 서버에서 렌더링, SignalR로 UI 업데이트 | 초기 로딩 빠름, 서버 의존 |
| **Blazor WebAssembly (WASM)** | 브라우저에서 .NET DLL + WebAssembly 실행 | 완전 클라이언트, 독립성 높음 |
| **Blazor Hybrid** | .NET MAUI에서 웹뷰 형태로 UI 표시 | 데스크탑/모바일 앱 가능 |
| **Blazor United** (미래형) | 서버 + 클라이언트 혼합 | 최적화된 통합 접근 (ASP.NET Core 8+)

---

## 🧪 4. Blazor Server vs WASM 비교

| 항목 | Blazor Server | Blazor WASM |
|------|---------------|-------------|
| 실행 위치 | 서버 | 브라우저 (WebAssembly) |
| 초기 로딩 | 빠름 | 느림 (DLL 다운로드) |
| 상호작용 | SignalR | 직접 실행 |
| .NET API 사용 | 완전 가능 | 제한적 (JS interop 필요) |
| 오프라인 사용 | 불가 | 가능 |
| 실시간 통신 | 매우 뛰어남 | SignalR 별도 구현 필요 |
| 배포 크기 | 작음 | 큼 (~2~6MB) |
| 디버깅 편의성 | 좋음 | 개선 중 (.NET 8+)

---

## 📄 5. 기본 예제 (Blazor Server)

### 🔹 Razor Component (Counter.razor)

```razor
<h3>Counter</h3>

<p>Current count: @currentCount</p>

<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount = 0;

    private void IncrementCount()
    {
        currentCount++;
    }
}
```

---

## 🌐 6. Blazor를 선택하는 이유

| 장점 | 설명 |
|------|------|
| ✅ C# 풀스택 개발 | 백엔드 + 프론트엔드를 모두 C#으로 |
| ✅ JavaScript 최소화 | JS 의존도 낮음 (Interop 가능) |
| ✅ 컴포넌트 기반 | 재사용성/유지보수성 높음 |
| ✅ ASP.NET Core와 자연스러운 통합 | 인증/라우팅/MVC/SignalR 등 쉽게 연계 |

---

## ⚠️ 7. Blazor의 단점

| 항목 | 설명 |
|------|------|
| WASM의 초기 로딩 속도 | DLL 다운로드로 인해 느릴 수 있음 |
| JS 생태계 연동 한계 | NPM 기반 생태계 사용 어려움 |
| 모바일 친화성 부족 (Server) | 느린 반응성 |
| 커뮤니티/레퍼런스 부족 | React/Vue 대비 낮은 자료 수 |

---

## 🔁 8. 기존 프론트엔드 프레임워크와의 비교

| 항목 | React/Vue/Angular | Blazor |
|------|--------------------|--------|
| 언어 | JavaScript/TypeScript | C# |
| SPA 지원 | ✅ | ✅ |
| 구성 요소 | Component 기반 | Razor Component 기반 |
| 상태 관리 | Redux/Vuex 등 | 자체 상태 or Fluxor 등 |
| 빌드 도구 | Webpack/Vite 등 | .NET SDK |
| 브라우저 지원 | 광범위 | WebAssembly 지원 필요 |
| 러닝 커브 | 프론트 경험자에게 유리 | C# 개발자에게 유리 |

---

## 🔒 9. 인증/보안 관련

Blazor는 ASP.NET Core의 인증 시스템을 그대로 사용 가능하며:

- Identity 기반 로그인/가입 가능
- JWT / Cookie / OAuth 적용 용이
- SignalR과 연동해 실시간 인증 처리 가능

---

## 📦 10. 프로젝트 구조 (Blazor Server 예시)

```bash
MyBlazorApp/
├── Pages/         👉 라우팅되는 Razor 페이지
├── Shared/        👉 컴포넌트들
├── _Imports.razor 👉 공통 using 정의
├── App.razor      👉 라우팅 시스템 구성
├── MainLayout.razor
└── Program.cs     👉 앱 설정 진입점
```

---

## ✅ 11. 요약

| 항목 | 설명 |
|------|------|
| Blazor 핵심 | C# 기반 SPA 웹 UI 프레임워크 |
| 주요 방식 | Server, WASM, Hybrid, United |
| 장점 | C# 일관성, 컴포넌트 기반, ASP.NET 통합 |
| 단점 | WASM 성능, JS 한계, 비교적 작은 생태계 |
| 추천 대상 | C# 풀스택 개발자, ASP.NET Core 기반 팀

---

## 🔜 추천 다음 주제

- ✅ Blazor WebAssembly 프로젝트 실습
- ✅ Blazor SignalR 채팅 구현
- ✅ Blazor 상태 관리 (Fluxor, CascadingParameter 등)
- ✅ Blazor + Identity 기반 인증 구성

---

Blazor는 **C# 개발자가 프론트엔드까지 주도할 수 있는 미래형 플랫폼**이에요.  
JS 생태계에 비해 아직은 작지만, 지속적인 발전 중이며  
**.NET 8~9 시대의 핵심 웹 기술**로 자리잡고 있어요!