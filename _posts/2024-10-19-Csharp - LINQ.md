---
layout: post
title: C# - LINQ
date: 2024-10-19 19:20:23 +0900
category: Csharp
---
# C# LINQ: 데이터 처리의 예술, 기초부터 실전까지 완벽 정리

## LINQ의 본질과 철학

LINQ(Language Integrated Query)는 C#에 통합된 선언형 데이터 처리 도구입니다. 데이터 소스(컬렉션, 데이터베이스, XML 등)에 대한 질의를 SQL과 유사한 구문으로 표현할 수 있게 해주며, "무엇을" 원하는지에 집중하고 "어떻게"는 런타임에게 맡기는 선언형 프로그래밍 패러다임을 구현합니다.

```csharp
using System;
using System.Linq;
using System.Collections.Generic;

// LINQ의 두 가지 표현 방식
var numbers = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// 메서드 구문 (Method Syntax) - 현대적이고 체이닝이 용이
var methodSyntax = numbers
    .Where(n => n % 2 == 0)
    .Select(n => n * 2)
    .OrderByDescending(n => n);

// 쿼리 구문 (Query Syntax) - SQL과 유사한 가독성
var querySyntax = from n in numbers
                  where n % 2 == 0
                  select n * 2;
```

---

## 지연 실행(Lazy Evaluation): LINQ의 마법

LINQ의 가장 강력한 특징 중 하나는 지연 실행입니다. 쿼리를 정의하는 시점이 아니라 실제로 결과를 사용할 때(열거할 때) 연산이 수행됩니다.

```csharp
var query = numbers.Where(n => 
{
    Console.WriteLine($"필터링: {n}");
    return n > 5;
});

Console.WriteLine("쿼리 정의 완료"); // 아직 아무것도 출력되지 않음

// 여기서 실제 실행
foreach (var num in query)
{
    Console.WriteLine($"결과: {num}");
}

// 출력:
// 쿼리 정의 완료
// 필터링: 1
// 필터링: 2
// 필터링: 3
// ...
```

**즉시 실행이 필요한 경우**:
```csharp
var immediate = numbers.Where(n => n > 5).ToList();  // ToList(), ToArray(), Count() 등
var firstItem = numbers.First(n => n > 5);          // First(), Single(), Max() 등
```

---

## 핵심 연산자: 데이터 처리의 기본 도구들

### 필터링 - Where
조건에 맞는 요소만 선택합니다.

```csharp
var adults = people.Where(p => p.Age >= 18);
var longNames = people.Where(p => p.Name.Length > 10);

// 인덱스 활용
var items = list.Where((item, index) => index % 2 == 0); // 짝수 인덱스 요소만
```

### 변환 - Select
각 요소를 새로운 형태로 매핑합니다.

```csharp
var names = people.Select(p => p.Name);
var nameLengths = people.Select(p => p.Name.Length);
var summaries = people.Select(p => new { p.Name, IsAdult = p.Age >= 18 });

// 인덱스와 함께 변환
var indexed = items.Select((item, index) => new { Index = index, Value = item });
```

### 정렬 - OrderBy, ThenBy, OrderByDescending

```csharp
// 기본 정렬
var sortedByName = people.OrderBy(p => p.Name);

// 다중 정렬
var sorted = people
    .OrderBy(p => p.Department)
    .ThenBy(p => p.Name)
    .ThenByDescending(p => p.Salary);

// 커스텀 비교자
var caseInsensitive = strings.OrderBy(s => s, StringComparer.OrdinalIgnoreCase);
```

### 요소 접근 - First, Single, ElementAt

```csharp
// First: 첫 번째 요소 (없으면 예외)
var first = numbers.First();
var firstEven = numbers.First(n => n % 2 == 0);

// FirstOrDefault: 안전한 접근 (없으면 기본값)
var safeFirst = numbers.FirstOrDefault();
var safeEven = numbers.FirstOrDefault(n => n % 2 == 0);

// Single: 정확히 하나의 요소 (없거나 여러 개면 예외)
var onlyOne = numbers.Single(n => n == 42);

// ElementAt: 특정 위치의 요소
var third = numbers.ElementAt(2);
var safeThird = numbers.ElementAtOrDefault(2); // 범위 밖이면 기본값
```

### 존재 여부 확인 - Any, All, Contains

```csharp
bool hasAdults = people.Any(p => p.Age >= 18);         // 조건 만족하는 요소가 하나라도 있는지
bool allAdults = people.All(p => p.Age >= 18);         // 모든 요소가 조건을 만족하는지
bool hasJohn = people.Any(p => p.Name == "John");      // 특정 값이 존재하는지
bool containsFive = numbers.Contains(5);               // 값 비교
bool containsPerson = people.Contains(specificPerson); // 참조 비교
```

### 집계 - Count, Sum, Average, Min, Max, Aggregate

```csharp
int totalPeople = people.Count();
int adultsCount = people.Count(p => p.Age >= 18);
decimal totalSalary = people.Sum(p => p.Salary);
decimal averageAge = people.Average(p => p.Age);
decimal maxSalary = people.Max(p => p.Salary);
string longestName = people.MaxBy(p => p.Name.Length)?.Name; // .NET 6+

// 커스텀 집계
var product = numbers.Aggregate((acc, n) => acc * n); // 1*2*3*4*...
var withSeed = numbers.Aggregate(10, (acc, n) => acc + n); // 초기값 10부터 시작
```

---

## 그룹화와 집계: 데이터 분석의 핵심

### GroupBy - 데이터 카테고리화

```csharp
// 기본 그룹화
var groupByAge = people.GroupBy(p => p.Age);

foreach (var group in groupByAge)
{
    Console.WriteLine($"나이 {group.Key}세:");
    foreach (var person in group)
    {
        Console.WriteLine($"  - {person.Name}");
    }
    Console.WriteLine($"  총 {group.Count()}명");
}

// 그룹별 집계
var departmentStats = employees
    .GroupBy(e => e.Department)
    .Select(g => new
    {
        Department = g.Key,
        EmployeeCount = g.Count(),
        AverageSalary = g.Average(e => e.Salary),
        MaxSalary = g.Max(e => e.Salary),
        MinSalary = g.Min(e => e.Salary),
        Employees = g.OrderBy(e => e.Name).ToList()
    })
    .OrderByDescending(d => d.AverageSalary);

// 복합 키 그룹화
var multiGroup = orders.GroupBy(o => new { o.Year, o.Month });
```

### ToLookup - 즉시 생성된 그룹 사전

```csharp
var lookup = products.ToLookup(p => p.CategoryId);

// 빠른 조회
var electronics = lookup[1]; // 카테고리 ID가 1인 모든 제품
foreach (var product in electronics)
{
    Console.WriteLine(product.Name);
}
```

`GroupBy`는 지연 실행되지만, `ToLookup`은 즉시 실행되어 메모리에 모든 그룹을 생성합니다. 빈번한 조회가 필요한 경우 유용합니다.

---

## 데이터 결합: 여러 소스 연결하기

### 내부 조인 (Inner Join)

```csharp
// 메서드 구문
var innerJoin = orders.Join(customers,
    order => order.CustomerId,
    customer => customer.Id,
    (order, customer) => new 
    { 
        OrderId = order.Id, 
        CustomerName = customer.Name,
        Amount = order.Total 
    });

// 쿼리 구문
var queryJoin = from order in orders
                join customer in customers on order.CustomerId equals customer.Id
                select new 
                { 
                    order.Id, 
                    customer.Name, 
                    order.Total 
                };
```

### 그룹 조인 (Group Join) - 일대다 관계

```csharp
var groupJoin = from customer in customers
                join order in orders on customer.Id equals order.CustomerId into customerOrders
                select new
                {
                    Customer = customer.Name,
                    OrderCount = customerOrders.Count(),
                    TotalSpent = customerOrders.Sum(o => o.Total),
                    Orders = customerOrders
                };
```

### 왼쪽 외부 조인 (Left Outer Join)

```csharp
var leftJoin = from customer in customers
               join order in orders on customer.Id equals order.CustomerId into customerOrders
               from order in customerOrders.DefaultIfEmpty()
               select new
               {
                   Customer = customer.Name,
                   OrderId = order?.Id,
                   Amount = order?.Total ?? 0
               };
```

### 교차 조인 (Cross Join)

```csharp
var crossJoin = from color in colors
                from size in sizes
                select new { Color = color, Size = size };
```

---

## 집합 연산: 데이터 비교 및 통합

```csharp
var list1 = new[] { 1, 2, 3, 4, 5 };
var list2 = new[] { 4, 5, 6, 7, 8 };

var distinct = list1.Distinct();               // 중복 제거: 1,2,3,4,5
var union = list1.Union(list2);                // 합집합: 1,2,3,4,5,6,7,8
var intersect = list1.Intersect(list2);        // 교집합: 4,5
var except = list1.Except(list2);              // 차집합: 1,2,3
var concat = list1.Concat(list2);              // 연결: 1,2,3,4,5,4,5,6,7,8

// .NET 6+의 *By 변형
var uniqueByName = people.DistinctBy(p => p.Name);
var unionByEmail = list1.UnionBy(list2, x => x.Email);
```

---

## 고급 변환 및 시퀀스 조작

### SelectMany - 평탄화(Flattening)

```csharp
// 중첩 컬렉션 평탄화
var departments = new[]
{
    new Department("개발팀", new[] { "Alice", "Bob", "Charlie" }),
    new Department("디자인팀", new[] { "David", "Eve" })
};

var allEmployees = departments.SelectMany(d => d.Employees);
// 결과: "Alice", "Bob", "Charlie", "David", "Eve"

// 다중 컬렉션 조합
var combinations = from x in list1
                   from y in list2
                   select new { X = x, Y = y };
```

### Zip - 두 시퀀스 병합

```csharp
var names = new[] { "Alice", "Bob", "Charlie" };
var ages = new[] { 25, 30, 35 };

var people = names.Zip(ages, (name, age) => new Person(name, age));
// 결과: Alice(25), Bob(30), Charlie(35)

// 세 개 이상의 시퀀스
var tripleZip = list1.Zip(list2, list3, (a, b, c) => (a, b, c));
```

### Chunk - 대용량 데이터 청킹 (.NET 6+)

```csharp
var largeList = Enumerable.Range(1, 1000);
var chunks = largeList.Chunk(100); // 각 청크는 최대 100개 요소

foreach (var chunk in chunks)
{
    ProcessChunk(chunk); // 대용량 데이터를 조각별로 처리
}
```

### Window - 슬라이딩 윈도우 (사용자 구현)

```csharp
public static IEnumerable<IEnumerable<T>> Window<T>(IEnumerable<T> source, int windowSize)
{
    var queue = new Queue<T>();
    
    foreach (var item in source)
    {
        queue.Enqueue(item);
        if (queue.Count == windowSize)
        {
            yield return queue.ToArray();
            queue.Dequeue();
        }
    }
}

// 이동 평균 계산
var prices = new[] { 100, 102, 101, 105, 103, 106 };
var movingAverages = Window(prices, 3)
    .Select(window => window.Average());
```

---

## 성능 최적화와 실전 패턴

### 1. 조기 필터링
```csharp
// 나쁜 예: 모든 데이터 변환 후 필터링
var bad = data.Select(x => ExpensiveTransform(x))
              .Where(x => x > 100);

// 좋은 예: 먼저 필터링하여 불필요한 변환 방지
var good = data.Where(x => x > 100)
               .Select(x => ExpensiveTransform(x));
```

### 2. 필요한 데이터만 선택
```csharp
// 필요한 열만 선택하여 메모리 사용 최적화
var efficient = orders
    .Where(o => o.Date.Year == 2024)
    .Select(o => new { o.Id, o.Total, o.CustomerName })
    .ToList();
```

### 3. 중복 계산 방지
```csharp
// 나쁜 예: 동일 쿼리 반복 실행
if (orders.Any(o => o.Total > 1000))
{
    var bigOrders = orders.Where(o => o.Total > 1000).ToList();
    // 쿼리가 두 번 실행됨
}

// 좋은 예: 한 번 실행하고 재사용
var bigOrders = orders.Where(o => o.Total > 1000).ToList();
if (bigOrders.Any())
{
    ProcessBigOrders(bigOrders);
}
```

### 4. 인덱스 활용 최적화
```csharp
// IList<T>나 배열인 경우 인덱스 접근이 효율적
var list = items as IList<T> ?? items.ToList();
for (int i = 0; i < list.Count; i++)
{
    // 인덱스 접근
}
```

### 5. 문자열 비교 최적화
```csharp
// 대소문자 무시 비교 시 StringComparer 사용
var distinctNames = names.Distinct(StringComparer.OrdinalIgnoreCase);
var sorted = names.OrderBy(n => n, StringComparer.CurrentCulture);
```

---

## 실전 예제: 복잡한 비즈니스 로직 구현

### 예제 1: 전자상거래 주문 분석
```csharp
public class OrderAnalysis
{
    public void AnalyzeOrders(IEnumerable<Order> orders, DateTime startDate, DateTime endDate)
    {
        var analysis = orders
            .Where(o => o.OrderDate >= startDate && o.OrderDate <= endDate)
            .GroupBy(o => o.CustomerId)
            .Select(g => new CustomerSummary
            {
                CustomerId = g.Key,
                OrderCount = g.Count(),
                TotalSpent = g.Sum(o => o.TotalAmount),
                AverageOrderValue = g.Average(o => o.TotalAmount),
                FavoriteCategory = g.SelectMany(o => o.Items)
                                  .GroupBy(i => i.Category)
                                  .OrderByDescending(grp => grp.Sum(i => i.Quantity))
                                  .FirstOrDefault()?.Key,
                LastOrderDate = g.Max(o => o.OrderDate)
            })
            .OrderByDescending(c => c.TotalSpent)
            .ToList();
        
        // 추가 분석
        var topCustomers = analysis.Take(10);
        var highValueCustomers = analysis.Where(c => c.TotalSpent > 10000);
        var inactiveCustomers = analysis.Where(c => 
            (DateTime.Now - c.LastOrderDate).TotalDays > 90);
    }
}
```

### 예제 2: 로그 데이터 실시간 모니터링
```csharp
public class LogMonitor
{
    public IEnumerable<Alert> AnalyzeLogs(IEnumerable<LogEntry> logs, TimeSpan window)
    {
        var now = DateTime.UtcNow;
        var recentLogs = logs.Where(l => l.Timestamp > now - window);
        
        // 에러율 계산
        var errorRateByService = recentLogs
            .GroupBy(l => l.Service)
            .Select(g => new
            {
                Service = g.Key,
                TotalRequests = g.Count(),
                ErrorCount = g.Count(l => l.Level == LogLevel.Error),
                ErrorRate = (double)g.Count(l => l.Level == LogLevel.Error) / g.Count()
            })
            .Where(x => x.ErrorRate > 0.01) // 1% 이상 에러율
            .OrderByDescending(x => x.ErrorRate);
        
        // 응답 시간 이상 탐지
        var slowEndpoints = recentLogs
            .Where(l => l.Duration.HasValue)
            .GroupBy(l => l.Endpoint)
            .Select(g => new
            {
                Endpoint = g.Key,
                AvgDuration = g.Average(l => l.Duration.Value),
                P95Duration = g
                    .OrderBy(l => l.Duration)
                    .ElementAt((int)(g.Count() * 0.95))
                    .Duration
            })
            .Where(x => x.P95Duration > TimeSpan.FromSeconds(2));
        
        // 이상 패턴 감지
        var errorSpikes = recentLogs
            .Where(l => l.Level == LogLevel.Error)
            .GroupBy(l => new 
            { 
                l.Service, 
                Minute = l.Timestamp.RoundToMinute() 
            })
            .Where(g => g.Count() > 10) // 분당 10개 이상 에러
            .Select(g => new Alert
            {
                Type = AlertType.ErrorSpike,
                Service = g.Key.Service,
                Timestamp = g.Key.Minute,
                Count = g.Count()
            });
        
        return errorSpikes;
    }
}
```

---

## 주의사항과 함정 피하기

### 1. 다중 열거 문제
```csharp
var query = expensiveSource.Where(x => ExpensiveFilter(x));

// ❌ 나쁜 예: 쿼리가 두 번 실행됨
var count = query.Count();
var sum = query.Sum();

// ✅ 좋은 예: 한 번만 실행
var materialized = query.ToList();
var count = materialized.Count;
var sum = materialized.Sum();
```

### 2. 캡처 변수 문제
```csharp
var actions = new List<Action>();

// ❌ 모든 액션이 마지막 i 값(10)을 사용
for (int i = 0; i < 10; i++)
{
    actions.Add(() => Console.WriteLine(i));
}

// ✅ 각 액션이 자신의 i 값을 캡처
for (int i = 0; i < 10; i++)
{
    int captured = i; // 로컬 변수로 캡처
    actions.Add(() => Console.WriteLine(captured));
}
```

### 3. 무한 시퀀스 주의
```csharp
// 무한 시퀀스 생성
var infinite = GenerateInfiniteSequence();

// ❌ ToList() 호출 시 무한 루프
// var list = infinite.ToList();

// ✅ Take로 제한
var first100 = infinite.Take(100).ToList();
```

### 4. IQueryable vs IEnumerable
```csharp
// Entity Framework 예제
public IActionResult GetProducts(int categoryId)
{
    // IQueryable: 데이터베이스에서 필터링
    var query = _context.Products
        .Where(p => p.CategoryId == categoryId)
        .OrderBy(p => p.Price);
    
    // 여기서까지는 쿼리만 구성, 아직 실행 안 됨
    
    var result = query.ToList(); // 여기서 SQL 실행
    
    // ❌ 클라이언트 측 평가 (모든 데이터를 메모리로 로드)
    var bad = _context.Products
        .AsEnumerable() // 주의: 모든 데이터를 메모리로!
        .Where(p => SomeCSharpMethod(p.Name)) // SQL로 변환 불가
        .ToList();
    
    return Ok(result);
}
```

---

## 결론: LINQ 마스터하기

LINQ는 단순한 데이터 질의 도구를 넘어 C# 프로그래밍 패러다임 자체를 바꾼 혁신적인 기능입니다. 효과적으로 사용하기 위한 핵심 원칙을 정리해 보겠습니다:

### 1. 선언적 사고로의 전환
LINQ의 진정한 힘은 "어떻게"가 아니라 "무엇을"에 집중하는 데 있습니다. 데이터 처리 로직을 간결하고 표현력 있게 작성할 수 있으며, 이는 코드의 가독성과 유지보수성을 크게 향상시킵니다.

### 2. 지연 실행의 이해와 활용
지연 실행은 LINQ의 성능 최적화 핵심 메커니즘입니다. 필요한 시점에만 데이터를 처리하고, 불필요한 계산을 피하며, 무한 시퀀스 같은 고급 패턴을 가능하게 합니다. 그러나 다중 열거 같은 함정을 피하기 위해 언제 materialize해야 하는지 이해하는 것이 중요합니다.

### 3. 조합의 예술
LINQ의 아름다움은 작고 단순한 연산자들을 조합하여 복잡한 데이터 처리 파이프라인을 구축할 수 있다는 점입니다. Where, Select, GroupBy, Join 같은 기본 블록들을 이해하고, 이를 창의적으로 조합하는 능력이 LINQ 마스터의 핵심 기술입니다.

### 4. 성능과 가독성의 균형
LINQ는 가독성을 희생하지 않으면서도 성능을 최적화할 수 있는 다양한 기법을 제공합니다: 조기 필터링, 필요한 데이터만 선택, 적절한 시점에 materialize, IQueryable을 통한 데이터베이스 최적화 등.

### 5. 도메인 지식과의 결합
가장 효과적인 LINQ 쿼리는 데이터 구조와 비즈니스 도메인을 깊이 이해한 상태에서 작성됩니다. 데이터의 특성, 관계, 접근 패턴을 이해할수록 더 효율적이고 정확한 쿼리를 작성할 수 있습니다.

LINQ를 마스터하는 과정은 단순히 구문을 배우는 것을 넘어, 데이터 중심 사고방식을 개발하는 과정입니다. 처음에는 익숙하지 않을 수 있지만, 일단 내재화되면 데이터 처리 작업이 즐거운 창의적 활동으로 변모할 것입니다. 복잡한 비즈니스 로직도 몇 줄의 LINQ 쿼리로 우아하게 표현하는 경험은 C# 개발자에게 주어진 특권이자 즐거움입니다.

데이터는 현대 소프트웨어의 핵심 자산입니다. LINQ는 이 자산을 다루는 가장 강력하고 표현력 있는 도구로서, 잘 마스터한다면 더 깔끔하고 효율적이며 유지보수하기 좋은 코드를 작성하는 데 크게 기여할 것입니다.