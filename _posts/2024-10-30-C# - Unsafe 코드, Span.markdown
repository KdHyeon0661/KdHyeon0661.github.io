---
layout: post
title: C# - Unsafe 코드, Span
date: 2024-10-30 19:20:23 +0900
category: C#
---
# C# 포인터와 Unsafe 코드, Span<T> 기초 정리

C#은 기본적으로 **안전한 메모리 관리**를 제공하지만, 때때로 포인터와 메모리 주소를 직접 다뤄야 할 경우가 있습니다.  
이 글에서는 C#에서 `unsafe`, 포인터, `fixed`, `stackalloc`, `Span<T>` 등 **네이티브 수준의 메모리 접근 방식**을 다룹니다.

---

## 🔷 1. `unsafe` 키워드란?

- C#의 포인터 연산을 허용하는 블록
- **컴파일 옵션 `/unsafe` 또는 `Allow unsafe code` 필요**

```csharp
unsafe
{
    int x = 10;
    int* p = &x;
    Console.WriteLine(*p); // 10
}
```

- `*p`: 포인터 역참조
- `&x`: 변수의 주소

> 기본적으로 안전하지 않기 때문에 일반 애플리케이션에서는 사용을 제한합니다.

---

## 🔷 2. fixed 문 – GC로부터 메모리 고정

C#의 가비지 컬렉터는 객체를 이동시킬 수 있기 때문에, **포인터로 접근할 땐 주소가 고정되어야** 합니다.

```csharp
unsafe
{
    int[] nums = { 1, 2, 3 };
    fixed (int* p = nums)
    {
        Console.WriteLine(p[0]); // 1
    }
}
```

- `fixed` 블록은 해당 변수의 메모리를 **GC가 이동하지 못하도록 고정**

---

## 🔷 3. stackalloc – 스택 메모리 직접 할당

스택에 메모리를 직접 할당하여 힙 할당보다 빠르게 처리합니다.

```csharp
Span<int> buffer = stackalloc int[5];
for (int i = 0; i < buffer.Length; i++)
    buffer[i] = i;

foreach (var n in buffer)
    Console.WriteLine(n); // 0 1 2 3 4
```

- `stackalloc`은 힙이 아닌 **스택 영역**에 데이터를 할당
- `Span<T>`와 함께 사용할 수 있음

---

## 🔷 4. Span<T> – 안전한 메모리 슬라이스

```csharp
Span<byte> bytes = stackalloc byte[4];
bytes[0] = 0x10;
bytes[1] = 0x20;

Span<byte> slice = bytes.Slice(1, 2); // 0x20, 0x00
```

- `Span<T>`는 배열, 포인터, 스택 메모리 등 **모든 연속 메모리 블록을 슬라이스처럼 다룸**
- 성능은 높지만 **`ref struct`로 힙에 저장할 수 없음**
- 메서드 인자로 넘길 수 있고, 복사 비용 없음

---

## 🔷 5. ref struct와 Span의 제약

```csharp
ref struct MyBuffer
{
    public Span<int> Data;
}
```

- `Span<T>`는 **힙에 존재할 수 없음 → 클래스 필드로 가질 수 없음**
- 따라서 `ref struct`는 스택에만 존재하며, GC에 의해 관리되지 않음
- 비동기 메서드나 람다 캡처 등에서도 사용할 수 없음

---

## ✅ 요약 정리

| 키워드 / 타입 | 설명 |
|---------------|------|
| `unsafe` | 포인터 사용 허용 |
| `*`, `&` | 포인터 연산자 |
| `fixed` | GC 이동 방지 (메모리 고정) |
| `stackalloc` | 스택 메모리 직접 할당 |
| `Span<T>` | 안전한 슬라이스 기반 메모리 접근 |
| `ref struct` | Span을 포함하는 스택 전용 구조체 |

---

## 🧠 언제 사용하나요?

- **Native API 호출 시 (P/Invoke)** 구조체를 포인터로 넘길 때
- **성능 최적화**가 필요한 코드 (ex. 대량 배열 연산)
- **GC 힙을 피하고 싶을 때**
- **Span<T>를 이용한 문자열 처리, 버퍼 조작** 등에