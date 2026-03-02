---
layout: post
title: C# - LINQ
date: 2024-10-19 19:20:23 +0900
category: Csharp
---
# C# LINQ: 데이터 처리의 예술, 기초부터 실전까지 완벽 정리

LINQ(Language Integrated Query)는 C# 3.0과 .NET Framework 3.5에 도입된 혁신적인 기능으로, 데이터 질의를 언어 수준에 통합한 기술입니다. 과거에는 데이터베이스(SQL), XML(XPath), 메모리 내 컬렉션(foreach) 등 데이터를 담고 있는 소스의 형태에 따라 각기 다른 방식의 접근법을 배워야 했습니다. LINQ는 이러한 이질성을 제거하고 모든 데이터 소스에 대해 **통일된 선언적 질의 패턴**을 제공합니다.

이 글에서는 LINQ의 기본 철학부터 시작하여 내부에서 작동하는 지연 실행의 원리, 필수 연산자들의 깊이 있는 활용법, 그리고 실무에서 마주하는 복잡한 데이터 가공 시나리오까지 자세히 확장하여 정리해 보겠습니다.

## LINQ의 철학과 본질

### 명령형에서 선언형 프로그래밍으로의 진화

LINQ의 가장 큰 패러다임 전환은 코드를 작성하는 사고방식을 **'어떻게(How)'**에서 **'무엇을(What)'**으로 바꾼다는 점입니다.

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// [명령형 방식] 제어 흐름(루프, 조건문)을 일일이 지시합니다.
List<int> imperativeResult = new List<int>();
foreach (var n in numbers)
{
    if (n % 2 == 0)
    {
        imperativeResult.Add(n * 2);
    }
}
imperativeResult.Sort((a, b) => b.CompareTo(a));

// [선언형 방식 - LINQ] 원하는 결과의 '조건'과 '형태'만 명시합니다.
var declarativeResult = numbers
    .Where(n => n % 2 == 0)
    .Select(n => n * 2)
    .OrderByDescending(n => n)
    .ToList();

```

명령형 코드는 요구사항이 복잡해질수록 `if`와 `for` 문이 깊게 중첩되어 가독성이 심각하게 떨어집니다. 반면 선언형인 LINQ 코드는 영어 문장을 읽듯 자연스럽게 의도가 파악되며, 사이드 이펙트(Side Effect)를 줄여 유지보수성을 극대화합니다.

### LINQ의 두 가지 표현 구문

LINQ는 개발자의 선호도에 따라 두 가지 구문을 제공하며, 컴파일 단계에서 쿼리 구문은 내부적으로 모두 메서드 구문으로 완벽히 동일하게 변환됩니다.

* **메서드 구문 (Method Syntax)**: 람다식(Lambda Expression)을 활용하여 확장 메서드를 체이닝하는 방식입니다. 현대 C# 개발에서 가장 주력으로 사용되며, 쿼리 구문보다 지원하는 메서드(`Max`, `Min`, `Count` 등)가 훨씬 많습니다.
* **쿼리 구문 (Query Syntax)**: SQL과 매우 유사한 형태로 작성합니다. 다중 `from` 절(Cross Join)이나 복잡한 `join`, `group by` 연산을 작성할 때 메서드 구문보다 가독성이 뛰어난 경우가 많습니다.

## 지연 실행 (Deferred Execution)의 메커니즘

LINQ를 다룰 때 반드시 이해해야 하는 핵심 동작 원리가 바로 **지연 실행**입니다.

`Where`, `Select`, `OrderBy`와 같은 LINQ 쿼리를 변수에 할당하는 순간에는 메모리 상에서 실제 데이터 순회나 계산이 **단 하나도 일어나지 않습니다.** 쿼리는 단지 "데이터를 어떻게 처리할 것인지"에 대한 계획서(상태 머신)일 뿐입니다. 이 계획서는 `foreach`로 순회하거나, `ToList()`, `Count()`와 같은 구체화(Materialization) 메서드를 호출하는 바로 그 시점에 비로소 실행됩니다.

### 반복자 패턴과 yield return

이러한 지연 실행은 C#의 `yield return` 키워드를 활용한 반복자(Iterator) 패턴으로 구현되어 있습니다.

```csharp
// LINQ Where의 내부 구현을 단순화한 모습
public static IEnumerable<T> Where<T>(this IEnumerable<T> source, Func<T, bool> predicate)
{
    foreach (var item in source)
    {
        // 쿼리가 실행될 때, 조건을 만족하는 데이터 하나를 찾는 즉시 반환(yield)하고 대기합니다.
        if (predicate(item))
        {
            yield return item; 
        }
    }
}

```

### 지연 실행이 주는 압도적 이점

1. **단일 순회 파이프라인**: 데이터를 필터링(`Where`)하고 변환(`Select`)할 때, 각각의 단계마다 새로운 배열(임시 메모리)을 만들지 않습니다. 데이터 한 건이 파이프라인의 처음부터 끝까지 통과한 뒤 다음 데이터가 처리되므로 메모리 효율이 극대화됩니다.
2. **무한 시퀀스 제어**: 끊임없이 생성되는 데이터 스트림이나 대용량 로그 파일을 읽을 때, 메모리 폭발 없이 필요한 만큼만 잘라서(`Take`) 처리할 수 있습니다.
3. **쿼리 최적화**: 데이터베이스 쿼리(`IQueryable`)의 경우, 실행 직전까지 여러 조건을 조합하여 가장 효율적인 단일 SQL 문장을 만들어 낼 수 있습니다.

## LINQ 핵심 연산자 심층 분석

단순한 데이터 검색을 넘어, 실무에서 자주 사용되는 필수 연산자들의 쓰임새를 자세히 알아보겠습니다.

### 1. 필터링 (Where, OfType)

```csharp
object[] mixedData = { 1, "Hello", 2, "World", 3.14 };

// OfType<T>를 사용하면 특정 타입만 안전하게 캐스팅하여 필터링합니다.
var stringsOnly = mixedData.OfType<string>(); // ["Hello", "World"]

```

### 2. 투영과 평탄화 (Select, SelectMany)

`Select`는 1:1 변환을 수행하지만, 요소 자체가 컬렉션을 품고 있는 중첩 구조일 때 이를 하나의 1차원 평면으로 펼쳐야(Flattening) 할 때가 있습니다. 이때 `SelectMany`를 사용합니다.

```text
[SelectMany 평탄화 원리]
School 1 -> [Class A, Class B]
School 2 -> [Class C]
SelectMany -> [Class A, Class B, Class C]

```

```csharp
public record Student(string Name, List<string> Subjects);

var students = new List<Student>
{
    new Student("Alice", new List<string> { "Math", "Physics" }),
    new Student("Bob", new List<string> { "Art", "History" })
};

// [오류 패턴] Select를 쓰면 List<List<string>> 형태의 2차원 컬렉션이 반환됩니다.
var subjectsList = students.Select(s => s.Subjects); 

// [올바른 패턴] SelectMany를 쓰면 모든 과목이 평탄화된 단일 List<string>이 됩니다.
var allSubjects = students.SelectMany(s => s.Subjects).Distinct(); 
// 결과: ["Math", "Physics", "Art", "History"]

```

### 3. 집합과 조인 (Join, GroupJoin)

두 개의 다른 데이터 소스를 특정 키(Key)를 기준으로 병합합니다. SQL의 INNER JOIN과 LEFT OUTER JOIN을 생각하면 이해하기 쉽습니다.

```csharp
var customers = new[] { 
    new { Id = 1, Name = "Kim" }, 
    new { Id = 2, Name = "Lee" } 
};
var orders = new[] { 
    new { OrderId = 101, CustId = 1, Product = "Laptop" },
    new { OrderId = 102, CustId = 1, Product = "Mouse" }
};

// Join: 고객 ID와 주문의 고객 ID가 일치하는 데이터만 연결 (Inner Join)
var innerJoinQuery = customers.Join(
    orders,
    c => c.Id,          // 기준 컬렉션(customers)의 키
    o => o.CustId,      // 대상 컬렉션(orders)의 키
    (c, o) => new { c.Name, o.Product } // 결과 생성
);

// GroupJoin: 한 명의 고객이 여러 주문을 가질 때, '고객 1 : 주문 N'의 그룹 형태로 묶어줍니다. (Left Join의 기초)
var groupJoinQuery = customers.GroupJoin(
    orders,
    c => c.Id,
    o => o.CustId,
    (customer, customerOrders) => new 
    { 
        CustomerName = customer.Name, 
        OrderCount = customerOrders.Count() 
    }
);

```

### 4. 집계의 유연함 (Aggregate)

`Sum`, `Max`, `Average` 등의 내장 집계 함수로 해결할 수 없는 복잡한 누적 연산이 필요할 때 사용하는 가장 근본적인 축약(Fold) 함수입니다.

```csharp
var words = new[] { "C#", "is", "awesome" };

// 초기값 "Result:"에서 시작하여, 각 단어를 차례대로 덧붙이는 누적 연산을 수행합니다.
string sentence = words.Aggregate("Result:", (current, next) => current + " " + next);
// 출력: "Result: C# is awesome"

```

## IEnumerable<T> vs IQueryable<T> 의 명확한 구분

LINQ를 다룰 때 발생하는 성능 문제의 80%는 이 두 인터페이스의 차이를 정확히 이해하지 못한 데서 비롯됩니다.

### IEnumerable<T> (LINQ to Objects)

* **작동 위치**: 애플리케이션의 메모리 위
* **동작 방식**: C#의 델리게이트(Func, Action)를 직접 실행합니다.
* **특징**: 데이터를 데이터베이스나 외부 시스템에서 모조리 메모리로 퍼올린 뒤에 필터링을 수행합니다. 데이터가 수백만 건이라면 엄청난 메모리 폭발이 발생합니다.

### IQueryable<T> (LINQ to Entities / SQL)

* **작동 위치**: 데이터베이스 엔진(SQL Server, MySQL 등) 내부
* **동작 방식**: 람다식을 코드로 실행하지 않고, **표현식 트리(Expression Tree)**라는 데이터 구조로 분석합니다. ORM(Entity Framework) 제공자가 이 트리를 뜯어보고 최적화된 진짜 `SQL` 쿼리문으로 번역합니다.
* **특징**: `Where`, `OrderBy` 조건이 모두 SQL의 `WHERE`, `ORDER BY` 절로 변환되어 DB 서버에서 실행되므로, 필요한 데이터 몇 건만 메모리로 깔끔하게 가져옵니다.

**[주의 사항]**
`IQueryable` 체이닝 중간에 `AsEnumerable()`, `ToList()`, `ToArray()`를 호출하는 순간, 쿼리의 번역은 거기서 중단되고 지금까지의 조건으로만 SQL을 날려 데이터를 메모리로 가져옵니다. 이후의 `Where`나 `Select`는 무거운 메모리 객체 연산으로 전락하므로, 데이터베이스에 넘길 조건은 반드시 구체화 호출 이전에 모두 작성해야 합니다.

## LINQ 실전 설계와 성능 최적화 패턴

### 1. 다중 열거 (Multiple Enumeration) 방지

지연 실행의 특성상, 구체화되지 않은 쿼리를 여러 번 사용하면 그때마다 처음부터 끝까지 연산이 다시 수행됩니다. 무거운 연산이 포함된 쿼리라면 치명적인 성능 저하를 부릅니다.

```csharp
var expensiveQuery = database.Orders.Where(o => ExpensiveCalculation(o));

// [나쁜 예] 아래 코드는 무거운 쿼리를 데이터베이스에 3번이나 별도로 날립니다.
if (expensiveQuery.Any()) 
{
    var count = expensiveQuery.Count();
    foreach (var item in expensiveQuery) { /* ... */ }
}

// [좋은 예] ToList()를 호출하여 결과를 메모리에 한 번 캐싱(Materialize)하고 재사용합니다.
var cachedResults = expensiveQuery.ToList();

if (cachedResults.Any())
{
    var count = cachedResults.Count;
    foreach (var item in cachedResults) { /* ... */ }
}

```

### 2. 조기 필터링 (Early Filtering)

데이터 파이프라인에서 무거운 변환(`Select`)이나 정렬(`OrderBy`)을 수행하기 전에, 데이터를 최소한으로 걸러내는(`Where`) 것이 성능 최적화의 첫걸음입니다.

```csharp
// [나쁜 예] 10만 건의 데이터를 모두 무거운 변환 함수에 통과시킨 뒤, 10건만 추려냅니다.
var bad = data.Select(x => ExpensiveTransform(x)).Where(x => x.Score > 90);

// [좋은 예] 점수가 90점 넘는 10건을 먼저 걸러낸 뒤, 그 10건에 대해서만 변환을 수행합니다.
var good = data.Where(x => x.Score > 90).Select(x => ExpensiveTransform(x));

```

### 3. 클로저(Closure) 변수 캡처의 함정

루프 안에서 LINQ 람다식을 만들 때 변수를 잘못 캡처하면 의도치 않은 동작이 발생합니다. (이 문제는 C# 5.0 이전의 `foreach`나 현재의 `for` 문에서 자주 발생합니다.)

```csharp
var actions = new List<Action>();

// [나쁜 예] 루프가 모두 끝난 뒤의 i 값(10)만을 모든 액션이 참조하게 됩니다.
for (int i = 0; i < 10; i++)
{
    actions.Add(() => Console.WriteLine(i)); 
}

// [좋은 예] 루프 내부에서 로컬 변수를 선언하여 각각의 고유한 값을 캡처하도록 합니다.
for (int i = 0; i < 10; i++)
{
    int captured = i; 
    actions.Add(() => Console.WriteLine(captured));
}

```

## 실전 시나리오: 복잡한 비즈니스 로직의 해결

기초 문법을 넘어, 실무에서 마주치는 복잡한 데이터 분석 요구사항을 LINQ로 어떻게 우아하게 해결하는지 보여주는 실전 예제입니다.

### 실전 예제 1: 전자상거래 주문 통계 분석

*요구사항: 최근 30일 내에 발생한 주문들을 고객별로 그룹화하고, 총 구매액이 1,000,000원 이상인 'VIP 고객' 상위 3명의 이름과 평균 구매액을 추출하라.*

```csharp
public record Order(string CustomerName, decimal Amount, DateTime OrderDate);

public List<dynamic> GetTopVipCustomers(List<Order> orders)
{
    var thirtyDaysAgo = DateTime.UtcNow.AddDays(-30);

    var vipStats = orders
        // 1. 기간 필터링 (조기 필터링)
        .Where(o => o.OrderDate >= thirtyDaysAgo)
        // 2. 고객 이름으로 그룹화 (Key: 이름, Value: 해당 고객의 주문들)
        .GroupBy(o => o.CustomerName)
        // 3. 그룹별로 익명 객체 투영 (총액, 평균액 계산)
        .Select(g => new 
        {
            CustomerName = g.Key,
            TotalSpent = g.Sum(o => o.Amount),
            AverageOrderValue = g.Average(o => o.Amount)
        })
        // 4. VIP 조건(100만 원 이상) 필터링
        .Where(x => x.TotalSpent >= 1000000m)
        // 5. 총 구매액 기준 내림차순 정렬
        .OrderByDescending(x => x.TotalSpent)
        // 6. 상위 3명만 추출
        .Take(3)
        // 7. 쿼리 실행 및 리스트 반환
        .ToList<dynamic>();
        
    return vipStats;
}

```

이처럼 7단계에 걸친 복잡한 로직이 단 하나의 자연스러운 체이닝 파이프라인으로 구축됩니다. 만약 이를 `foreach`와 중첩된 `Dictionary`, `if` 문으로 작성했다면 코드는 최소 3배 이상 길어지고 유지보수는 악몽이 되었을 것입니다.

## 마무리: 데이터 처리 패러다임의 완성

LINQ는 단순한 컬렉션 제어 라이브러리가 아닙니다. C# 언어에 함수형 프로그래밍(Functional Programming)의 불변성과 고차 함수 개념을 우아하게 이식하여, 개발자의 사고방식을 데이터 '처리'에서 데이터 '질의'로 격상시킨 혁명적인 도구입니다.

`Where`, `Select`, `GroupBy` 등 기초적인 연산자의 조합법을 익히는 것부터 시작하여, 지연 실행(Deferred Execution) 파이프라인이 낭비하는 메모리 없이 어떻게 데이터를 흘려보내는지 내부를 통찰해야 합니다. 나아가 `IEnumerable`과 `IQueryable`의 차이를 완벽히 이해하여 데이터베이스의 자원을 지켜내는 수준에 도달한다면, 여러분의 C# 코드는 더할 나위 없이 간결하고 강력해질 것입니다.