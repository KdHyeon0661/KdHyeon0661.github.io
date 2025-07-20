---
layout: post
title: C# - PInvoke, Marshal, NativeMemory
date: 2024-11-01 19:20:23 +0900
category: Csharp
---
# C# Span<T>와 Memory<T> 심화: 고성능 메모리 조작 기법

.NET에서는 고성능 메모리 작업을 위해 `Span<T>`, `Memory<T>`, `stackalloc` 등의 도구를 제공합니다.  
이들은 힙 할당을 줄이고, **슬라이스 기반의 안전한 연산**, **버퍼 처리**, **네이티브 메모리 접근**에 유용합니다.

---

## 🔷 1. Span<T>란?

- `Span<T>`는 배열, 포인터, 스택 메모리 등 **연속된 메모리 블록을 안전하게 슬라이스**할 수 있는 구조체입니다.
- `ref struct`이므로 **스택에만 존재**, GC 힙에는 못 올림

```csharp
Span<int> numbers = stackalloc int[5];
for (int i = 0; i < numbers.Length; i++)
    numbers[i] = i;

Span<int> slice = numbers.Slice(2, 2); // 2, 3
```

---

## 🔷 2. ReadOnlySpan<T>

- 읽기 전용 슬라이스
- `string`, `byte[]`, `stackalloc` 등에서 **읽기 전용 뷰**를 제공할 때 사용

```csharp
ReadOnlySpan<char> span = "Hello, World".AsSpan(7, 5);
Console.WriteLine(span.ToString()); // World
```

> ✅ Span을 인자로 넘길 때는 가능한 한 `ReadOnlySpan<T>`로 선언하는 것이 좋습니다.

---

## 🔷 3. Memory<T> – 힙 기반 슬라이스

- `Span<T>`는 스택 전용이라 **비동기 메서드나 필드 저장 불가**
- `Memory<T>`는 힙에서도 사용 가능 → 비동기나 람다에서 자유롭게 사용

```csharp
Memory<byte> buffer = new byte[10];
var slice = buffer.Slice(3, 4);
```

- `.Span` 프로퍼티를 통해 내부 `Span<T>` 참조 가능

---

## 🔷 4. stackalloc – 스택 메모리 직접 할당

```csharp
Span<byte> buffer = stackalloc byte[8];
```

- GC에 의한 힙 할당이 아니라, **스택에 직접 메모리 할당**
- 메모리 해제 필요 없음 (스코프 벗어나면 자동 해제)
- 대용량 데이터에는 부적합 (스택 오버플로우 가능)

---

## 🔷 5. Span으로 문자열 처리

```csharp
string input = "apple,banana,kiwi";
ReadOnlySpan<char> span = input.AsSpan();

int idx = span.IndexOf(',');
var first = span.Slice(0, idx);
var second = span.Slice(idx + 1);

Console.WriteLine(first.ToString());  // apple
Console.WriteLine(second.ToString()); // banana,kiwi
```

- **Substring을 대체하는 고성능 문자열 파싱**
- `.AsSpan()`은 메모리 복사를 하지 않음 → 빠름!

---

## 🔷 6. MemoryMarshal – Span과 포인터 변환 도구

```csharp
Span<byte> span = stackalloc byte[4];
span[0] = 0x01;
span[1] = 0x02;

ref ushort value = ref MemoryMarshal.GetReference(span);
```

- `MemoryMarshal.Cast`, `GetReference`, `AsBytes` 등 제공
- 바이너리 구조, 포인터 해석, 구조체 ↔ 바이트 변환 등에 활용

> 🧠 위험할 수 있으므로 구조체 레이아웃과 크기를 정확히 알아야 함

---

## 🔷 7. Span vs Memory 차이 요약

| 특징 | Span<T> | Memory<T> |
|------|---------|-----------|
| 힙 저장 가능 여부 | ❌ (스택 전용) | ✅ (힙에 저장 가능) |
| 비동기/람다 사용 | ❌ | ✅ |
| ref struct 여부 | ✅ | ❌ |
| 사용 예시 | 고속 파싱, 버퍼 처리 | 비동기 처리 시 안전한 메모리 전달 |

---

## ✅ 요약 정리

| 도구 | 용도 |
|------|------|
| `Span<T>` | 스택 메모리, 고속 슬라이스 |
| `ReadOnlySpan<T>` | 읽기 전용 뷰 제공 |
| `stackalloc` | GC 없는 스택 메모리 |
| `Memory<T>` | 힙에서도 사용 가능한 Span |
| `MemoryMarshal` | Span ↔ 바이트/포인터 변환 |