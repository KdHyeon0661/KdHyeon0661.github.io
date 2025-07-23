---
layout: post
title: AspNet - NET 플랫폼 개요와 발전
date: 2025-01-27 19:20:23 +0900
category: AspNet
---
# 🌐 .NET 플랫폼 개요와 발전

## ✅ .NET Framework vs .NET Core vs .NET 5/6/7/8

| 항목 | .NET Framework | .NET Core | .NET 5/6/7/8 (현행 .NET) |
|------|----------------|-----------|--------------------------|
| 최초 릴리스 | 2002 (.NET 1.0) | 2016 (.NET Core 1.0) | 2020 (.NET 5) 이후 매년 |
| 플랫폼 | Windows 전용 | Windows, macOS, Linux | 크로스 플랫폼 |
| 오픈소스 | ❌ 일부만 | ✅ 완전 오픈소스 | ✅ 완전 오픈소스 |
| 성능 | 무거움 | 빠름 | 최적화 |
| 유지보수 | 레거시 | 종료 예정 | 마이크로소프트의 현재 기준 |
| 대표 기술 | ASP.NET, WPF, WinForms | ASP.NET Core, CLI | 통합된 .NET 플랫폼 |

### 🔄 변화 요약

- `.NET Framework`는 Windows 전용으로, 레거시 시스템에서 여전히 많이 사용되지만 **더 이상 적극적인 기능 추가는 없음**.
- `.NET Core`는 경량 크로스 플랫폼 런타임으로 등장하여 `.NET Framework`를 대체.
- `.NET 5`부터는 이름을 **단순히 ".NET"**으로 통합, `Core`라는 명칭은 없어졌고, 현재는 `.NET 8`이 최신 LTS 버전.

---

# 🚀 ASP.NET Core의 특징과 장점

ASP.NET Core는 **웹 애플리케이션, API, 실시간 통신, 마이크로서비스 등 다양한 웹 기술을 지원하는 크로스 플랫폼 웹 프레임워크**입니다.

## 주요 특징

### ✅ 1. 크로스 플랫폼 (Windows, macOS, Linux)
- .NET Core 기반으로 모든 주요 OS에서 동작
- Linux 서버 + Docker와 궁합이 뛰어남

### ✅ 2. 성능 최적화
- Kestrel 웹 서버는 매우 빠르고 가볍다
- 비동기(Async/Await) 기반 처리로 대규모 트래픽 처리에 적합

### ✅ 3. 통합 웹 프레임워크
- 웹 사이트 (Razor Pages, MVC)
- API (REST API)
- 실시간 통신 (SignalR)
- 단일 프로젝트 안에 다양한 유형의 서비스 구현 가능

### ✅ 4. 강력한 DI (Dependency Injection) 내장
- DI가 프레임워크에 기본 탑재되어 있어 구조적인 코드 작성 가능

### ✅ 5. 모듈화 및 경량화
- 필요에 따라 구성 요소만 선택하여 경량화 가능
- 미들웨어 방식으로 요청 파이프라인 구성

### ✅ 6. 클라우드 친화적
- Azure와의 통합 최적화
- 구성 파일, 환경 변수, 비밀 저장소 등과 유연하게 연동

### ✅ 7. 오픈소스 및 커뮤니티 기반
- GitHub에서 개발 중
- 활발한 커뮤니티와 공식 문서

---

# 🧱 ASP.NET Core에서 제공하는 주요 구성 기술

## ✅ 1. 웹앱 (Razor Pages / MVC)

**설명**: HTML UI를 서버에서 생성해 클라이언트에 전송하는 구조

- **Razor Pages**: 페이지 단위의 UI 처리 (간단한 CRUD에 적합)
- **MVC (Model-View-Controller)**: 대규모 웹사이트나 구조화가 필요한 경우

**예시 사용처**: 회사 홈페이지, 블로그, 백오피스 관리 시스템 등

---

## ✅ 2. Web API

**설명**: JSON 기반의 RESTful API를 제공

- Controller 클래스에서 API 정의
- 클라이언트(앱, SPA, 외부 서비스 등)에서 데이터 소비

**예시 사용처**: 모바일 앱 백엔드, 프론트엔드(Vue/React) API 서버, 마이크로서비스

---

## ✅ 3. Blazor

**설명**: C#으로 프론트엔드까지 구현할 수 있는 Web UI 프레임워크

- **Blazor Server**: SignalR 기반으로 서버와 실시간 연결
- **Blazor WebAssembly**: C#을 브라우저에서 직접 실행 (SPA처럼 동작)

**장점**:
- JavaScript 없이 C#만으로 프론트 구현 가능
- 기존 .NET 기술 재사용

**예시 사용처**: 사내 포털, 복잡한 UI 앱, SPA

---

## ✅ 4. SignalR

**설명**: 실시간 양방향 통신을 위한 라이브러리

- WebSocket 기반으로 실시간 채팅, 알림, 대시보드에 사용
- 매우 빠르고, 클라이언트 SDK (JS, C#, Java 등) 다양

**예시 사용처**: 실시간 채팅, 실시간 주식 시세, 게임, IoT 대시보드

---

# 📌 요약

| 기술       | 주 목적         | 예시 |
|------------|----------------|------|
| Razor Pages | 단순 웹 UI      | 홈페이지, 관리 도구 |
| MVC        | 구조화된 웹앱    | 블로그, 쇼핑몰 |
| Web API    | 데이터 제공     | 앱 백엔드, 프론트용 API |
| Blazor     | C# 기반 UI 프론트 | SPA, 복잡한 UI 앱 |
| SignalR    | 실시간 통신     | 채팅, 대시보드 |