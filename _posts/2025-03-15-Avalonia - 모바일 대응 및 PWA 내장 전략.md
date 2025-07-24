---
layout: post
title: Avalonia - 모바일 대응 및 PWA 내장 전략
date: 2025-03-15 19:20:23 +0900
category: Avalonia
---
# 📱 Avalonia의 모바일 대응 및 PWA 내장 전략 (실험적)

---

## 🎯 핵심 요약

| 영역 | 설명 |
|------|------|
| 📱 모바일 플랫폼 대응 | Android, iOS (아직은 실험적) |
| 🌐 PWA/웹 실행 | WebView 내장 or Avalonia.Web (장기 방향) |
| 🧪 개발 생산성 개선 | Hot reload, Cross-platform 공유 UI 등

---

## 1️⃣ Avalonia의 플랫폼 대응 현황

| 플랫폼 | 지원 수준 |
|--------|------------|
| 🖥️ Windows/macOS/Linux | ✅ 안정적 |
| 📱 Android, iOS         | ⚠️ 실험적 (Avalonia.Mobile) |
| 🌐 Web (PWA)            | ⚠️ 미지원 (WebView 우회 가능) |
| 🧩 Embedded (RaspberryPi 등) | ⚠️ 지원 중 (Limited OpenGL) |

---

## 2️⃣ Avalonia.Mobile (Android/iOS) 실험적 대응

### 🔬 개요

`Avalonia.Mobile`은 Android/iOS 지원을 위한 브랜치로, 아직 정식 패키지에 포함되어 있지 않습니다.  
2024년 기준으로 다음과 같은 상태입니다:

| 항목 | 설명 |
|------|------|
| 🏗️ .NET MAUI + Avalonia 혼용 가능성 | UI 재사용 실험 중 |
| 📦 GitHub 수동 빌드 필요 | nuget으로 설치 불가 |
| ❌ 아직 Production-Ready 아님 | View/입력 이슈 존재 |

---

### 🛠️ 시도 방법 (참고용)

```bash
git clone https://github.com/AvaloniaUI/Avalonia
cd Avalonia/samples/Mobile
dotnet build
```

→ Android/iOS용 `.apk`/`.app` 생성 테스트 가능  
(단, 시뮬레이터 설정 필요 + 아직 기기 최적화 부족)

---

## 3️⃣ WebView + Web 내장 방식

### ✅ 사용 이유

| 상황 | 해결책 |
|------|--------|
| 🌐 웹 페이지 표시 필요 | `WebView` 사용 |
| 🧱 일부 기능 웹 기반 구현 | Vue/React 내장 |
| 📦 로그인/결제 등 웹 처리 | 외부 링크/내장 브라우저 가능 |

---

### 📦 Avalonia.WebView 설치

```bash
dotnet add package Avalonia.WebView.Desktop
```

### 📄 MainView.axaml 예시

```xml
<avares:WebView Source="https://example.com" />
```

> CEFSharp 기반 → Chromium 엔진

---

### ✅ Hybrid 앱 구성 예

- Avalonia로 전체 UI 구성
- 특정 페이지 영역에 WebView 삽입
- 필요 시 JavaScript ↔ C# 상호작용 (JSBridge)

> 예: 로그인은 WebView로 처리, 메인 앱은 Avalonia로

---

## 4️⃣ 개발 생산성 향상을 위한 전략

### ⚡ 1. Hot Reload 지원

```bash
dotnet watch
```

- XAML/C# 수정 시 자동 반영
- `Avalonia.ReactiveUI` 조합 시 효과적

### 🧩 2. MVVM 기반 모듈화

- ViewModel, Service, Repository 분리
- View → Platform에 따라만 구현 변경

### 🔄 3. 공유 UI 컴포넌트화

- 커스텀 컨트롤 (e.g. `MyForm`, `MyTable`)
- 플랫폼 종속성 최소화

---

## 5️⃣ 향후 Avalonia의 웹 대응 방향

| 접근 | 설명 |
|------|------|
| 🔧 WebAssembly (실험 중) | Blazor/MAUI와 유사한 WASM 가능성 |
| 🧪 Avalonia.Web | 아직 계획 단계 수준 |
| 🌐 웹 내장 WebView 우회 전략 | 현재는 가장 실용적 |

---

## 📚 플랫폼별 요약

| 플랫폼 | 지원 | 대응 방법 |
|--------|------|-----------|
| Windows/macOS/Linux | ✅ | 기본 Avalonia 지원 |
| Android/iOS         | ⚠️ | Avalonia.Mobile + 실험 빌드 |
| Web(PWA)            | ❌ | WebView or 외부 연동 |
| Embedded            | ⚠️ | RaspberryPi(OpenGL) 조합 가능 |

---

## 📌 생산성 개선 도구 정리

| 도구 | 설명 |
|------|------|
| 🔄 dotnet watch | 코드/스타일 핫리로드 |
| 🧪 AvaloniaPreviewer | VS/Rider에서 실시간 미리보기 |
| 🧱 Git Submodules | 공통 UI 모듈화 및 공유 |
| 🛠️ ReactiveUI | MVVM 생산성 향상, 유효성 검사 용이 |
| 📦 Avalonia.FuncUI | Elm-like UI 선언 (실험적 대안)

---

## ✅ 결론

| 항목 | 정리 |
|------|------|
| 📱 모바일 | 아직 실험적이지만 `Avalonia.Mobile`로 일부 시도 가능 |
| 🌐 PWA | 정식 지원은 없음 → WebView가 현재 대안 |
| 🧪 생산성 | `dotnet watch`, MVVM 모듈화, HotReload 적극 활용 |
| 🧩 장기 전략 | WebAssembly 등 미래 확장 가능성 대비 필요