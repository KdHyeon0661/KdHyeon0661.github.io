---
layout: post
title: Avalonia - ViewBag, ViewData, TempData
date: 2025-03-22 19:20:23 +0900
category: Avalonia
---
# 🧰 ASP.NET Core에서 ViewBag, ViewData, TempData 완전 정리

---

## ✅ 개요

ASP.NET Core에서는 **Controller → View 또는 View ↔ View 간 데이터 전달**을 위해 다음 객체들을 제공합니다:

| 이름 | 유형 | 목적 | 지속 범위 |
|------|------|------|-------------|
| `ViewData` | `Dictionary` | Controller → View | 한 요청(Request) 내 |
| `ViewBag` | `dynamic` (ViewData Wrapper) | Controller → View | 한 요청(Request) 내 |
| `TempData` | `Dictionary` | 다음 요청까지 | Redirect 등에서 유지 |

---

## 🟦 1. ViewData

- `string` 키와 `object` 값을 가지는 **Dictionary**
- `ViewBag`과 달리 명시적 형 변환 필요

### ✅ 사용 예시

```csharp
// Controller
ViewData["Title"] = "제품 목록";
ViewData["Count"] = 10;
```

```html
<!-- View -->
<h2>@ViewData["Title"]</h2>
<p>총 개수: @(int)ViewData["Count"]</p>
```

---

## 🟣 2. ViewBag

- `ViewData`의 **dynamic wrapper**
- 컴파일 시점에는 타입 체크 불가

### ✅ 사용 예시

```csharp
// Controller
ViewBag.Title = "제품 목록";
ViewBag.Count = 10;
```

```html
<!-- View -->
<h2>@ViewBag.Title</h2>
<p>총 개수: @ViewBag.Count</p>
```

---

## 🟡 3. TempData

- **다음 요청까지 유지되는 데이터 저장소**
- `RedirectToAction` 같은 경우에 유용
- `Session` 기반

### ✅ 사용 예시

```csharp
// Controller: 데이터 저장
TempData["Message"] = "저장 완료되었습니다.";
return RedirectToAction("Index");

// 다음 요청에서 읽기
public IActionResult Index()
{
    ViewBag.Message = TempData["Message"];
    return View();
}
```

```html
<!-- View -->
@if (ViewBag.Message != null)
{
    <div>@ViewBag.Message</div>
}
```

### 🔄 TempData.Keep() / TempData.Peek()

| 메서드 | 설명 |
|--------|------|
| `Keep("key")` | 읽은 후에도 값을 보존 |
| `Peek("key")` | 값을 읽되 삭제하지 않음 |

---

## 📊 비교 정리

| 항목 | ViewData | ViewBag | TempData |
|------|----------|---------|----------|
| 타입 | Dictionary | dynamic | Dictionary |
| 대상 | View만 | View만 | 다음 요청까지 |
| 유지 범위 | 현재 요청 | 현재 요청 | 다음 요청까지 유지 |
| 형변환 필요 | ✅ Yes | ❌ No | ✅ Yes |
| 사용 예 | `ViewData["Title"]` | `ViewBag.Title` | `TempData["Message"]` |
| 내부 저장소 | `ViewDataDictionary` | `ViewData`의 wrapper | `ITempDataDictionary` |

---

## 💡 언제 어떤 걸 써야 할까?

| 상황 | 추천 |
|------|------|
| View에 간단한 값 전달 | ✅ `ViewBag` / `ViewData` |
| ViewModel이 복잡해 `Model`로 넘기기 어려운 소규모 값 | ✅ `ViewBag` |
| Redirect 후 메시지 표시 (`Flash Message`) | ✅ `TempData` |
| 타입 안정성과 유지보수 우선 | ✅ ViewModel 사용이 가장 바람직 |

---

## ⚠️ 주의 사항

- `ViewBag`, `ViewData`는 **TempData와 다르게 리다이렉션 시 소멸**
- `TempData`는 **세션 기반이므로 쿠키 설정 필요**
- `ViewBag`은 런타임 에러를 초래할 수 있으므로 **간단한 UI에만 사용**
- **복잡한 구조 데이터 전달 시 → ViewModel이 항상 최우선**

---

## ✅ 예제 통합

```csharp
public IActionResult Index()
{
    ViewBag.Title = "제품 목록";
    ViewData["Count"] = 5;
    TempData["Notice"] = "데이터가 성공적으로 처리되었습니다.";
    return View();
}
```

```html
<!-- Index.cshtml -->
<h2>@ViewBag.Title</h2>
<p>총 개수: @ViewData["Count"]</p>

@if (TempData["Notice"] != null)
{
    <div class="alert">@TempData["Notice"]</div>
}
```

---

## 🔚 결론

| ViewModel 우선 | `ViewBag`, `ViewData`, `TempData`는 보조적인 용도 |
|----------------|----------------------------------------------------|
| `ViewModel`은 타입 안정성, 테스트 편의성 모두 우수 |
| `ViewBag`/`ViewData`는 단순 메시지, 작은 정보 전달에 적합 |
| `TempData`는 페이지 이동 간 상태 유지에 유용 |

---

## 🔜 추천 다음 주제

- ✅ `ViewModel`과 `DTO` 패턴 적용
- ✅ `Session`, `Cookie`와 TempData 비교
- ✅ `Partial View`, `Layout` 간 데이터 전달 방식
- ✅ `Flash Message` 템플릿 통합 방법

---

`ViewBag`, `ViewData`, `TempData`는 MVC에서 자주 쓰이는 데이터 전달 도구이지만,  
**유지보수성과 안정성을 위해 ViewModel 기반 개발이 더 좋다**는 점을 기억하자!