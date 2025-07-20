---
layout: post
title: C# - PInvoke, Marshal, NativeMemory
date: 2024-10-31 19:20:23 +0900
category: Csharp
---
# C# P/Invoke, Marshal, NativeMemory 완전 정리

C#은 **.NET 런타임 위에서 동작**하지만, 경우에 따라 C/C++로 작성된 DLL이나 Win32 API를 직접 호출해야 하는 상황이 있습니다.  
이때 사용하는 기술이 바로 **P/Invoke (Platform Invocation)**와 **Marshal**, 그리고 **NativeMemory**입니다.

---

## 🔷 1. P/Invoke란?

- C# 코드에서 C/C++로 작성된 **네이티브 DLL 함수**를 직접 호출할 수 있게 해주는 기술
- `DllImport` 특성을 사용하여 외부 DLL 함수 선언

### ✅ 예시: Win32 MessageBox 호출

```csharp
using System.Runtime.InteropServices;

public class NativeMethods
{
    [DllImport("user32.dll", CharSet = CharSet.Unicode)]
    public static extern int MessageBox(IntPtr hWnd, string text, string caption, uint type);
}

// 사용
NativeMethods.MessageBox(IntPtr.Zero, "Hello", "Title", 0);
```

---

## 🔷 2. 구조체 마샬링 – C와 구조체 맞추기

```csharp
[StructLayout(LayoutKind.Sequential)]
struct MyStruct
{
    public int id;
    public float value;
}
```

- `LayoutKind.Sequential`: 메모리 배치를 C와 동일하게 순서대로
- `StructLayout`은 반드시 필요 (C의 `struct`과 정확히 일치시켜야 함)

---

## 🔷 3. 문자열 마샬링 – `char*`, `wchar_t*` 다루기

### ✅ 문자열 전달

```csharp
[DllImport("mylib.dll", CharSet = CharSet.Ansi)]
public static extern void PrintMessage(string msg);
```

### ✅ 문자열 포인터 반환 (Marshal.PtrToString*)

```csharp
[DllImport("mylib.dll")]
public static extern IntPtr GetMessage();

string msg = Marshal.PtrToStringAnsi(GetMessage());
```

> `PtrToStringAnsi`, `PtrToStringUni`, `PtrToStringUTF8` 등 필요에 따라 선택

---

## 🔷 4. 배열 마샬링

### ✅ C 함수: `void Process(int* arr, int size);`

```csharp
[DllImport("mylib.dll")]
public static extern void Process([In, Out] int[] arr, int size);
```

- `[In, Out]`: 호출 시 배열이 참조로 전달되며, 수정된 결과를 가져올 수 있음

---

## 🔷 5. GCHandle – 객체를 네이티브에 안전하게 전달

```csharp
GCHandle handle = GCHandle.Alloc(myObject, GCHandleType.Pinned);
IntPtr ptr = GCHandle.ToIntPtr(handle);

// 네이티브 전달 후...

handle.Free(); // 꼭 해제할 것
```

- 객체를 고정(pinned)하면 GC가 이동시키지 않음
- C에 포인터로 안전하게 넘길 수 있음

---

## 🔷 6. NativeMemory – .NET 6+ 저수준 메모리 API

### ✅ 메모리 직접 할당 / 해제

```csharp
nint ptr = NativeMemory.Alloc((nuint)100); // 100바이트
NativeMemory.Free(ptr);
```

- `nint`, `nuint`: 플랫폼에 맞는 포인터형
- C의 `malloc` / `free`를 .NET에서 직접 제어 가능

---

## 🔷 7. Marshal 클래스 주요 메서드

| 메서드 | 설명 |
|--------|------|
| `Marshal.AllocHGlobal` | 힙 메모리 할당 (C의 malloc과 유사) |
| `Marshal.FreeHGlobal` | 할당 메모리 해제 |
| `Marshal.StructureToPtr` | 구조체 → 포인터 복사 |
| `Marshal.PtrToStructure<T>()` | 포인터 → 구조체 복사 |
| `Marshal.Copy` | 배열 ↔ 포인터 데이터 복사 |

---

## ✅ 요약 정리

| 기술 | 설명 |
|------|------|
| `DllImport` | 외부 DLL 호출 |
| `StructLayout` | 구조체 정렬 지정 |
| `Marshal` | 네이티브 ↔ 관리형 간 데이터 변환 |
| `GCHandle` | 객체 고정 및 포인터 변환 |
| `NativeMemory` | 포인터 수준의 메모리 할당 |
| `PtrToStringX` | 문자열 포인터 → C# 문자열 변환 |

---

## 🔐 주의사항

- P/Invoke는 **런타임 오류 발생 가능성**이 높음
- 호출 대상 함수와 시그니처가 정확히 일치해야 함
- 구조체 정렬/문자열 인코딩 등도 정확히 맞춰야 함
