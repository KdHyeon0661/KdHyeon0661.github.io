---
layout: post
title: C# - LINQ
date: 2024-10-19 19:20:23 +0900
category: Csharp
---
# C# LINQ 기초부터 실전까지 (Where, Select, GroupBy, Any, All, etc.)

## 준비 — 네임스페이스와 컬렉션

```csharp
using System;
using System.Linq;
using System.Collections.Generic;
```

- 대부분의 연산은 `IEnumerable<T>` 확장 메서드(= **LINQ to Objects**).
- 데이터 원천이 DB(예: EF Core)면 `IQueryable<T>`로 **식 트리**가 SQL로 번역됩니다(주의사항은 §14).

---

## 문법 두 가지: Method vs Query

```csharp
var result1 = list.Where(x => x > 10).Select(x => x * 2);        // Method 문법
var result2 = from x in list where x > 10 select x * 2;           // Query 문법
```

- 현업에서는 **Method 문법**을 더 많이 사용(체이닝·도구 친화적).
- Query 문법만 가능한 주요 구문: **join**, **group … by**, **into**, **let** 등(대부분 Method로도 구현 가능).

---

## 지연 실행(Deferred Execution) & 즉시 평가(Materialization)

- `Where/Select/GroupBy/OrderBy` 등은 **열거할 때 실행**됩니다.
- 즉시 결과가 필요하면 **`ToList()`/`ToArray()`/`ToDictionary()`** 등으로 materialize.

```csharp
var q = list.Where(x => x % 2 == 0);     // 아직 실행 X
var even = q.ToList();                   // 여기서 실행(평가)
```

> **주의**: 동일 쿼리를 여러 번 순회하면 **매번 재실행**됩니다(§12의 “다중 열거” 참고). 필요한 시점에 **한 번만 materialize**하세요.

---

## 가장 많이 쓰는 연산자

### Where — 필터링

```csharp
var evens = list.Where(x => x % 2 == 0);
```

### Select — 변환(매핑)

```csharp
var doubled = list.Select(x => x * 2);
var withIndex = list.Select((x, i) => new { Index = i, Value = x });
```

### OrderBy / ThenBy — 정렬

```csharp
var sorted = people.OrderBy(p => p.Age).ThenBy(p => p.Name);
var desc   = people.OrderByDescending(p => p.Score);
```
- 커스텀 비교자:
```csharp
var ciSorted = strings.OrderBy(s => s, StringComparer.OrdinalIgnoreCase);
```

### First / FirstOrDefault / Single / SingleOrDefault

```csharp
var first   = list.First();           // 없으면 예외
var safe    = list.FirstOrDefault();  // 없으면 기본값(레퍼런스형은 null)

var one     = list.Single(x => x == 7);          // 조건 만족 **딱 하나** 아니면 예외
var oneSafe = list.SingleOrDefault(x => x == 7); // 0개면 기본값, 2개 이상이면 예외
```

### Any / All — 존재/전부 검사

```csharp
bool hasNegative = list.Any(x => x < 0);
bool allPositive = list.All(x => x > 0);
```

### Count / Sum / Average / Max / Min

```csharp
int    count = list.Count();
int    sum   = ints.Sum();
double avg   = numbers.Average();
var    max   = people.Max(p => p.Score);
```

---

## GroupBy — 그룹화와 집계

```csharp
var groups = people.GroupBy(p => p.Age);
foreach (var g in groups)
{
    Console.WriteLine($"나이: {g.Key}, 인원: {g.Count()}");
    foreach (var p in g) Console.WriteLine($" - {p.Name}");
}
```

### 그룹에 대한 투영

```csharp
var stats = people
    .GroupBy(p => p.Department)
    .Select(g => new {
        Dept   = g.Key,
        Count  = g.Count(),
        Avg    = g.Average(p => p.Salary),
        Top    = g.MaxBy(p => p.Salary)   // .NET 6+: MaxBy/MinBy
    })
    .OrderByDescending(x => x.Avg);
```

### ToLookup — 즉시 색인화(멀티맵)

```csharp
var lookup = people.ToLookup(p => p.Age); // 나이→사람들
foreach (var p in lookup[30]) Console.WriteLine(p.Name);
```
- `GroupBy`는 **지연**, `ToLookup`은 **즉시**. 반복 조회가 많으면 `ToLookup`이 편리.

---

## 집합 연산: Distinct / Union / Intersect / Except

```csharp
var unique      = list.Distinct();
var union       = list1.Union(list2);
var intersect   = list1.Intersect(list2);
var except      = list1.Except(list2);
```

- 커스텀 동치(예: 대소문자 무시 문자열)
```csharp
var ci = StringComparer.OrdinalIgnoreCase;
var uniqNames = names.Distinct(ci);
```

- **DistinctBy/UnionBy/IntersectBy/ExceptBy** (.NET 6+)
```csharp
var uniqByEmail = users.DistinctBy(u => u.Email);
```

---

## 투영 확장: SelectMany — 평탄화(flatten)

```csharp
var words = new[] { "hi", "there" };
var chars = words.SelectMany(w => w.ToCharArray()); // 'h','i','t','h','e','r','e'
```

- 일대다 관계(주문→주문항목) 평탄화에 적합.
```csharp
var allItems = orders.SelectMany(o => o.Items);
```

---

## 조인(Join/GroupJoin)과 Left Join 패턴

### 내부 조인(Inner Join)

```csharp
var q =
    from o in orders
    join c in customers on o.CustomerId equals c.Id
    select new { o.Id, CustomerName = c.Name, o.Total };
```
또는 Method:
```csharp
var q = orders.Join(customers,
    o => o.CustomerId,
    c => c.Id,
    (o, c) => new { o.Id, CustomerName = c.Name, o.Total });
```

### 그룹 조인(GroupJoin) — 1:N

```csharp
var q =
    from c in customers
    join o in orders on c.Id equals o.CustomerId into grp
    select new { c.Name, OrderCount = grp.Count(), Orders = grp };
```

### Left Join (없어도 포함)

```csharp
var left =
    from c in customers
    join o in orders on c.Id equals o.CustomerId into grp
    from o in grp.DefaultIfEmpty()             // 없으면 null
    select new { c.Name, OrderId = o?.Id, Total = o?.Total ?? 0m };
```

---

## 페이징/슬라이딩/청크/Zip

```csharp
var page = list.Skip((pageIndex - 1) * pageSize).Take(pageSize);
var window = list.Skip(start).Take(length);

var chunks = list.Chunk(3); // .NET 6+: [a,b,c],[d,e,f]...
foreach (var ch in chunks) Console.WriteLine($"[{string.Join(",", ch)}]");

var zipped = xs.Zip(ys, (x,y) => (x,y)); // 두 시퀀스 병렬 결합
```

---

## 사전/그룹 구조로 바로 만들기

```csharp
var dic = people.ToDictionary(p => p.Id);                 // 키 중복시 예외
var safeDic = people
    .GroupBy(p => p.Id)                                   // 충돌 처리
    .ToDictionary(g => g.Key, g => g.Last());

var byDept = people.GroupBy(p => p.Department)
                   .ToDictionary(g => g.Key, g => g.ToList());
```

---

## 고급 집계: Aggregate / Scan(누적) / MaxBy-MinBy

```csharp
var factorial = Enumerable.Range(1, 5).Aggregate((acc, x) => acc * x); // 120

// 누적 합 시퀀스(간단 구현)
IEnumerable<int> Scan(IEnumerable<int> src)
{
    int acc = 0;
    foreach (var x in src) { acc += x; yield return acc; }
}
```

- .NET 6+: `MaxBy(selector)`, `MinBy(selector)`로 객체의 최대/최소 결정자 선택.

---

## Query 문법 확장 — let/into

```csharp
var q =
    from p in people
    let key = p.Name.ToUpperInvariant()
    where key.StartsWith("A")
    select new { p.Name, Key = key };
```

- `into`로 중간 결과 이름을 바꿔 체인 계속:
```csharp
var q =
    from n in names
    select n.ToUpper() into upper
    where upper.Length > 3
    select upper;
```

---

## 지연 실행 함정과 “다중 열거” 이슈

```csharp
var q = ExpensiveSource().Where(x => x > 0);

// 두 번 순회하면 비싼 작업도 두 번 수행될 수 있음
var a = q.Count();
var b = q.Sum();
```

**대응**: 한 번만 실행하고 결과 재사용
```csharp
var materialized = q.ToList();
var a = materialized.Count;
var b = materialized.Sum();
```

> 또한, `foreach` 중 원본 컬렉션을 수정하면 예외가 날 수 있습니다(컬렉션 종류별 상이).

---

## 클로저(Closure) 캡처 함정

```csharp
var actions = new List<Action>();
for (int i = 0; i < 3; i++)
    actions.Add(() => Console.WriteLine(i));

// 출력: 3,3,3 (캡처된 변수 i가 같은 참조)
```

**대응**: 루프 변수 복사
```csharp
for (int i = 0; i < 3; i++)
{
    int captured = i;
    actions.Add(() => Console.WriteLine(captured)); // 0,1,2
}
```

LINQ 람다에서도 동일 규칙이 적용됩니다.

---

## IEnumerable vs IQueryable — 번역 가능한 식만!

- `IEnumerable<T>`: **메모리 내 컬렉션**에 대한 **즉시/지연 계산**(C# 코드 그대로 실행).
- `IQueryable<T>`: **식 트리**를 DB 등 프로바이더가 **쿼리로 번역**(예: EF Core → SQL).

**중요**:
- `IQueryable`에서 **번역 불가**한 .NET 메서드를 쓰면 **런타임 예외** 혹은 **클라이언트 평가**(성능 저하) 위험.
- DB 쿼리는 하나로 합치고, 결과를 가져온 뒤에야 **메모리 내 연산**을 수행:
```csharp
// EF Core 예시
var rows = await db.People
    .Where(p => p.Age > 20)
    .OrderBy(p => p.Name)
    .Select(p => new { p.Name, p.Age })
    .ToListAsync(); // 여기까지는 SQL로 번역

var grouped = rows.GroupBy(r => r.Age / 10).ToList(); // 메모리 내 후처리
```

---

## 성능 팁

1) **필요한 열만 Select** (투영 최소화)
2) **필터 먼저 → 정렬/집계** 순(데이터 축소 후 비싼 연산)
3) **중복 열거 방지**: 재사용 시 `ToList()`
4) 문자열 비교는 `StringComparison`/`StringComparer` 명시
5) 핫패스에서 `GroupBy`/`OrderBy` 남발 금지(정렬/해시 비용 큼)
6) 매우 큰 배열/Span 기반 처리 필요 시: LINQ 대신 루프/`Span<T>` 검토
7) **예상 크기** 알면 `ToDictionary`/`List(capacity)`로 미리 용량 지정

간단한 비용 직관:
$$
\text{총비용} \approx \sum \text{연산자별 입력크기} \times \text{연산비용}
$$
필터로 **입력 크기를 먼저 줄이는 전략**이 효과적입니다.

---

## PLINQ(병렬 LINQ) — CPU 바운드 시 가속

```csharp
using System.Linq;

var parallel =
    Enumerable.Range(1, 1_000_000)
    .AsParallel()                 // PLINQ 시작
    .WithDegreeOfParallelism( Environment.ProcessorCount )
    .Where(IsPrime)
    .Select(x => x * x)
    .ToArray();
```

- I/O 바운드에는 효과 낮음(비동기/파이프라인 고려).
- 순서가 중요하면 `.AsOrdered()`, 성능은 다소 손해.

---

## 실전 예제 1 — 상위 N 카테고리 매출(Left Join 포함)

요구:
1) 모든 카테고리 이름을 표기(주문 없는 카테고리도 0으로).
2) 매출 상위 3개를 출력.

```csharp
public record Category(int Id, string Name);
public record Product(int Id, int CategoryId, string Name, decimal Price);
public record OrderLine(int ProductId, int Qty);

var categories = new[]
{
    new Category(1,"Book"), new Category(2,"Toy"), new Category(3,"Food")
};
var products = new[]
{
    new Product(1,1,"C# in Depth", 40m),
    new Product(2,1,"LINQ Pocket", 25m),
    new Product(3,2,"Blocks", 15m)
};
var lines = new[]
{
    new OrderLine(1,2), // 2 * 40
    new OrderLine(3,5), // 5 * 15
};

var salesPerCategory =
    from c in categories
    join p in products on c.Id equals p.CategoryId into gp
    from p in gp.DefaultIfEmpty()
    join l in lines    on p?.Id equals l.ProductId into gl
    from l in gl.DefaultIfEmpty()
    group new { p, l } by c into g
    select new
    {
        Category = g.Key.Name,
        Total = g.Sum(x => (x.p?.Price ?? 0m) * (x.l?.Qty ?? 0))
    };

var top3 = salesPerCategory
    .OrderByDescending(x => x.Total)
    .Take(3)
    .ToList();

foreach (var x in top3)
    Console.WriteLine($"{x.Category}: {x.Total:C}");
```

---

## 실전 예제 2 — 로그 분석: 상태코드별 상위 URL, 이동 평균

```csharp
public record Log(DateTime Ts, string Url, int Status, int Ms);

var logs = new List<Log> {
    new(DateTime.Parse("2025-11-10T10:00:00"), "/a", 200, 120),
    new(DateTime.Parse("2025-11-10T10:00:01"), "/a", 200, 80),
    new(DateTime.Parse("2025-11-10T10:00:02"), "/b", 500, 10),
    new(DateTime.Parse("2025-11-10T10:00:03"), "/a", 200, 130),
    new(DateTime.Parse("2025-11-10T10:00:04"), "/b", 500, 11),
};

// 상태코드별: URL 상위 1개(요청 수 기준)
var topPerStatus =
    logs.GroupBy(l => l.Status)
        .Select(g => new {
            Status = g.Key,
            TopUrl = g.GroupBy(l => l.Url)
                      .OrderByDescending(gg => gg.Count())
                      .Select(gg => new { Url = gg.Key, Count = gg.Count() })
                      .First()
        });

foreach (var x in topPerStatus)
    Console.WriteLine($"{x.Status} → {x.TopUrl.Url}({x.TopUrl.Count})");

// 이동 평균(3개 창) — 간단 구현
var byTs =
    logs.OrderBy(l => l.Ts).Select(l => l.Ms).ToList();

var window3 = byTs
    .Select((v,i) => i >= 2 ? byTs.Skip(i-2).Take(3).Average() : (double?)null)
    .ToList();

Console.WriteLine($"윈도우 평균: {string.Join(", ", window3.Select(x => x?.ToString("F1") ?? "-"))}");
```

---

## 고급 트릭 모음

- **DefaultIfEmpty**: 빈 시퀀스에 기본값을 하나 주입(Left Join 구현에 핵심).
- **Prepend/Append**: 시퀀스의 앞/뒤에 원소 하나 추가.
- **Range/Repeat**:
```csharp
var nums = Enumerable.Range(1, 5);          // 1..5
var zeros = Enumerable.Repeat(0, 3);        // 0,0,0
```
- **Chunk/Windowing**: 고정 크기 청크는 `.Chunk(n)`(6+), 가변 창은 직접 구현.
- **OfType<T>**: 시퀀스에서 특정 타입만 필터.
- **Cast<T>**: 모든 요소를 지정 타입으로 캐스팅(실패 시 예외).

---

## 테스트 가능성과 예측 가능성

- 쿼리의 **순서 보장**: `OrderBy`를 써서 명시. 해시 기반 집합/사전은 순서가 의미 없음.
- **시계/랜덤**과 섞지 말고, 필요 시 외부에서 값을 주입(재현성↑).

---

## 체크리스트

**설계**
- [ ] 데이터량 큰가? 먼저 **Where**로 줄이고 그다음 정렬/그룹
- [ ] 여러 번 쓰는 쿼리인가? **한 번만 materialize**
- [ ] 조인 키/동치/정렬 규칙 명확?
- [ ] 사전/색인 필요 시 **ToDictionary/ToLookup**
- [ ] EF Core면 **번역 가능성**(메서드, Regex, 사용자 함수) 검토

**성능**
- [ ] `Select`로 필요한 열만
- [ ] `StringComparer`/`StringComparison` 지정
- [ ] 핫패스에서 `GroupBy`/`OrderBy` 남발 자제
- [ ] PLINQ는 CPU 바운드에서만

**정확성**
- [ ] `First` vs `Single` 목적 구분
- [ ] `DefaultIfEmpty`(Left Join) 이해
- [ ] 클로저 변수 캡처 주의

---

## 요약 표

| 범주 | 핵심 메서드 | 비고 |
|---|---|---|
| 필터/투영 | `Where`, `Select`, `SelectMany` | 인덱스 오버로드, 평탄화 |
| 정렬 | `OrderBy`, `ThenBy`, `Reverse` | Comparer/Case-insensitive |
| 요소 | `First(OrDefault)`, `Single(OrDefault)`, `ElementAt` | 의미/예외 차이 주의 |
| 집계 | `Count`, `Sum`, `Average`, `Max`, `Min`, `Aggregate`, `MaxBy/MinBy` | 누적/커스텀 집계 |
| 그룹/색인 | `GroupBy`, `ToLookup` | 즉시 vs 지연 |
| 조인 | `Join`, `GroupJoin`, `DefaultIfEmpty` | Left Join 패턴 |
| 집합 | `Distinct`, `Union`, `Intersect`, `Except`, `*By` | 커스텀 동치 |
| 변환 | `ToList`, `ToArray`, `ToDictionary` | materialize |
| 기타 | `Skip/Take`, `Chunk`, `Zip`, `Range/Repeat` | 페이징/청크/병렬 결합 |

---

## 마무리

LINQ는 “**작은 연산자의 조합**으로 **큰 질의**를 읽기 쉽게 표현”하는 도구입니다.
핵심은 **지연 실행을 이해**하고, **한 번만 평가**하며, **필요한 최소 데이터만** 다루는 것.
DB와 연동 시에는 **번역 가능성**을 항상 염두에 두고, 복잡한 후처리는 **메모리로 가져온 뒤** 수행하세요.
