---
layout: post
title: C# - LINQ
date: 2024-10-19 19:20:23 +0900
category: Csharp
---
# C# LINQ 기초부터 실전까지 (Where, Select, GroupBy, Any, All, etc.)

LINQ(Language Integrated Query)는 C#에서 컬렉션, 배열, 데이터베이스 등을 SQL처럼 쉽게 다룰 수 있는 기능입니다.  
`System.Linq` 네임스페이스를 통해 대부분의 컬렉션에서 LINQ 메서드를 사용할 수 있습니다.

---

## 🔷 1. LINQ 사용을 위한 준비

```csharp
using System.Linq;
```

대부분의 LINQ 메서드는 `List<T>`, `Array`, `IEnumerable<T>` 등에 사용할 수 있습니다.

---

## 🔷 2. 기본 문법 (Method 문법 vs Query 문법)

```csharp
// Method 문법 (람다 기반)
var result = list.Where(x => x > 10).Select(x => x * 2);

// Query 문법 (SQL 스타일)
var result = from x in list
             where x > 10
             select x * 2;
```

> 실제 개발에서는 **메서드 문법이 더 많이 사용**됩니다.

---

## 🔷 3. 자주 쓰이는 LINQ 메서드

### ✅ Where – 필터링

```csharp
var filtered = list.Where(x => x % 2 == 0);
```

### ✅ Select – 변형 (매핑)

```csharp
var doubled = list.Select(x => x * 2);
```

### ✅ OrderBy / OrderByDescending – 정렬

```csharp
var sorted = list.OrderBy(x => x.Length);
```

### ✅ First / FirstOrDefault

```csharp
var first = list.First();              // 요소 없으면 예외
var safe = list.FirstOrDefault();      // 없으면 null 또는 기본값
```

### ✅ Any / All

```csharp
bool hasNegative = list.Any(x => x < 0);
bool allPositive = list.All(x => x > 0);
```

### ✅ Count / Sum / Average / Max / Min

```csharp
int count = list.Count();
int sum = list.Sum();
double avg = list.Average();
```

---

## 🔷 4. GroupBy – 그룹화

```csharp
var groups = people.GroupBy(p => p.Age);

foreach (var group in groups)
{
    Console.WriteLine($"나이: {group.Key}");
    foreach (var person in group)
        Console.WriteLine($" - {person.Name}");
}
```

---

## 🔷 5. ToList / ToArray – 결과 materialize

```csharp
var evenList = list.Where(x => x % 2 == 0).ToList();
```

LINQ는 **지연 실행(Lazy Evaluation)**이기 때문에, `.ToList()`나 `.ToArray()`로 **즉시 평가**해야 결과를 고정할 수 있습니다.

---

## 🔷 6. Distinct / Contains / Except

```csharp
var unique = list.Distinct();
bool hasValue = list.Contains(3);
var diff = list.Except(otherList); // 차집합
```

---

## 🔷 7. 복잡한 예시 – 조건 필터 + 정렬 + 변형

```csharp
var top3Names = people
    .Where(p => p.Age > 20)
    .OrderByDescending(p => p.Score)
    .Select(p => p.Name)
    .Take(3)
    .ToList();
```

---

## ✅ 요약 정리

| 메서드 | 설명 |
|--------|------|
| `Where` | 조건 필터링 |
| `Select` | 값 변형 (map) |
| `OrderBy` / `ThenBy` | 정렬 |
| `GroupBy` | 그룹핑 |
| `First`, `FirstOrDefault` | 첫 요소 |
| `Any`, `All` | 조건 만족 여부 |
| `Count`, `Sum`, `Average` | 수치 집계 |
| `Distinct`, `Except`, `Union` | 집합 연산 |
| `ToList`, `ToArray` | 결과 확정 (materialize) |