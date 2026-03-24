---
layout: post
title: C# - 매개변수 한정자와 Nullable 관련 연산자
date: 2024-10-07 20:20:23 +0900
category: Csharp
---
# C# 매개변수 한정자와 Null 안전성 연산자

C#을 다루다 보면 메서드 매개변수에 붙는 `out`, `ref`, `in`, `params`와 변수 할당 과정에서 등장하는 `??`, `?.`, `!` 같은 기호들을 자주 마주하게 됩니다. 이들은 단순한 문법적 편의를 넘어, 메모리 관리 방식, 성능 최적화, 예외 방어 등 C#의 깊은 동작 원리와 직결되어 있습니다.

---

## 매개변수 전달 한정자

C# 메서드에 인수를 전달할 때 기본은 **값에 의한 전달(Pass by Value)** 입니다. 즉, 원본 데이터의 복사본이 메서드 내부로 전달됩니다. 하지만 여러 값을 반환하거나, 복사 비용이 큰 데이터를 효율적으로 처리해야 할 때는 원본 메모리 주소를 직접 제어해야 하며, 이때 매개변수 한정자를 사용합니다.

### out 한정자와 Try 패턴

`out` 키워드는 메서드가 하나 이상의 결과값을 반환해야 할 때 사용합니다. `return` 문은 단일 값만 반환할 수 있기 때문입니다.

`out`의 핵심 규칙은 다음과 같습니다.
- 호출하는 쪽에서는 변수를 초기화하지 않고 전달할 수 있습니다.
- 메서드 내부에서는 해당 매개변수에 반드시 값을 할당해야 합니다. 할당하지 않으면 컴파일 오류가 발생합니다.

```csharp
string input = "123";
if (int.TryParse(input, out int number))
{
    Console.WriteLine($"변환 성공: {number}");
}
else
{
    Console.WriteLine("변환 실패");
}
```

이러한 특성을 활용한 **Try 패턴**은 C# 생태계 전반에서 널리 사용됩니다. 메서드의 반환값으로는 성공 여부(`bool`)를 반환하고, 실제 결과는 `out` 매개변수로 출력합니다.

```csharp
// Dictionary에서의 Try 패턴
if (dictionary.TryGetValue("key", out var value))
{
    Console.WriteLine(value);
}

// JsonDocument에서의 Try 패턴
using JsonDocument doc = JsonDocument.Parse(json);
if (doc.RootElement.TryGetProperty("name", out var nameElement))
{
    Console.WriteLine(nameElement.GetString());
}
```

이 패턴은 변환이나 검색에 실패했을 때 예외를 발생시키는 대신 조건 분기로 처리할 수 있게 해주어, 성능 저하를 방지하고 코드의 흐름을 명확하게 합니다.

### ref 한정자와 참조 전달

`ref` 키워드는 인수를 참조(Reference) 형태로 전달합니다. `out`과 달리 양방향 통신이 가능합니다. 호출하는 쪽에서 반드시 변수를 초기화한 상태로 넘겨야 하며, 메서드 내부에서는 값을 읽고 쓸 수 있습니다.

값 형식(struct)과 참조 형식(class)을 `ref`로 전달할 때 메모리 동작 방식이 다릅니다. 아래 다이어그램을 통해 살펴보겠습니다.

```text
[값 형식(struct)의 ref 전달]
Stack 메모리
[ 변수 x (값: 10) ] <--- ref a (x의 메모리 주소를 직접 참조)

[참조 형식(class)의 ref 전달]
Stack 메모리               Heap 메모리
[ 변수 p (참조 주소) ] ---> [ Person 객체 ("John") ]
         ^
         | ref person (p 변수 자체의 메모리 주소를 참조)
```

값 형식을 `ref`로 전달하면 스택(Stack) 상의 원본 주소를 직접 가리키므로, 메서드 내부에서 수정하면 원본 변수도 즉시 변경됩니다.

```csharp
int x = 10;
ModifyValue(ref x);
Console.WriteLine(x); // 20

void ModifyValue(ref int a) => a = 20;
```

참조 형식을 `ref`로 전달하면, 힙(Heap) 객체를 가리키는 참조 변수 자체의 주소를 전달하는 **이중 포인터** 개념이 됩니다. 따라서 메서드 내부에서 `new`로 새 객체를 할당하면, 호출부의 원본 참조도 새로운 객체를 가리키게 됩니다.

```csharp
class Person { public string Name = ""; }

Person p = new Person { Name = "John" };
ModifyReference(ref p);
Console.WriteLine(p.Name); // "Jane"

void ModifyReference(ref Person person) 
{
    person = new Person { Name = "Jane" }; 
}
```

### in 한정자와 방어적 복사

`in` 키워드는 인수를 읽기 전용 참조로 전달합니다. 주로 크기가 큰 구조체(struct)를 메서드에 전달할 때 값 복사 비용을 줄이기 위해 사용됩니다.

```csharp
public struct LargeStruct
{
    public long A, B, C, D; // 32바이트
}

public void Print(in LargeStruct data)
{
    // data.A = 10; // 컴파일 오류: 읽기 전용
    Console.WriteLine(data.A);
}
```

그러나 `in` 매개변수에는 **방어적 복사(Defensive Copy)** 라는 성능 함정이 있습니다. 만약 구조체가 내부 상태를 변경할 수 있는 일반 구조체라면, 컴파일러는 원본이 훼손될 가능성을 고려하여 메서드 내부에서 속성이나 메서드를 호출할 때 임시 복사본을 생성합니다.

```csharp
public struct Point
{
    public int X { get; set; }
    public void SetX(int value) => X = value;
}

public void Print(in Point p)
{
    // 원본을 보호하기 위해 보이지 않는 복사본이 생성됨
    p.SetX(10); 
    // 원본 Point는 변경되지 않음
}
```

이 문제를 피하려면 `in`으로 전달할 구조체는 반드시 `readonly struct`로 선언하여 불변성을 보장하는 것이 모범 사례입니다.

### params 한정자와 배열 할당

`params` 키워드는 가변 개수의 인수를 받을 수 있게 합니다. 메서드 선언의 마지막 매개변수에만 사용할 수 있으며, 배열 형태로 선언합니다.

```csharp
public static int Sum(params int[] numbers)
{
    int total = 0;
    foreach (var num in numbers) total += num;
    return total;
}

Console.WriteLine(Sum(1, 2, 3));       // 쉼표로 구분
Console.WriteLine(Sum(new int[] { 10, 20 })); // 배열 직접 전달
```

편리함 뒤에는 성능 비용이 존재합니다. 컴파일러는 `Sum(1, 2, 3)` 호출을 `Sum(new int[] { 1, 2, 3 })`로 변환합니다. 반복문 내에서 사용하면 매 호출마다 새로운 배열이 힙에 할당됩니다. 반복 횟수를 $$N$$이라 할 때, 메모리 할당 횟수도 $$O(N)$$이 되어 가비지 컬렉션 부담이 커집니다. 성능이 중요한 로직에서는 배열을 미리 생성해 재사용하는 것이 좋습니다.

---

## Nullable 및 Null 안전성 연산자

Null 참조 예외(NullReferenceException)는 C# 프로그램에서 가장 흔히 발생하는 런타임 오류 중 하나입니다. C#은 이를 방지하기 위해 다양한 연산자와 타입 시스템을 제공합니다.

### 값 형식의 Null 허용

값 형식(`int`, `bool`, `struct` 등)은 기본적으로 `null`을 가질 수 없습니다. 데이터베이스나 JSON 등에서 "값 없음"을 표현해야 할 때는 `Nullable<T>` 구조체를 사용합니다. 약식 표기로 타입 뒤에 물음표(`?`)를 붙입니다.

```csharp
int? nullableInt = null;

if (nullableInt.HasValue)
{
    Console.WriteLine(nullableInt.Value);
}
```

### 널 병합 연산자 (??)와 할당 연산자 (??=)

`??` 연산자는 좌항이 `null`이 아니면 좌항을, `null`이면 우항을 반환합니다.

```text
[널 병합 연산자 A ?? B 논리 흐름]
             (A의 값 평가)
                  |
        +---------+---------+
        |                   |
    Null이 아님          Null임
        |                   |
    (A를 반환)          (B를 반환)
```

```csharp
string? name = null;
string result = name ?? "이름 없음"; // "이름 없음"
```

`??=` 연산자는 좌항이 `null`일 때만 우항을 할당합니다. 지연 초기화 패턴에서 유용합니다.

```csharp
List<int>? list = null;
list ??= new List<int>(); // list가 null이었으므로 새 객체 할당
list.Add(1);
```

### 널 조건부 연산자 (?.)

객체의 멤버에 접근할 때, 객체가 `null`이면 예외 대신 `null`을 반환하고 평가를 중단합니다.

```csharp
string? name = null;
Console.WriteLine(name?.Length); // null 출력 (예외 발생 안 함)

int length = name?.Length ?? 0; // 널 병합과 결합: 0
```

### 널 억제 연산자 (!)와 Nullable 참조 타입

C# 8.0부터 도입된 **Nullable 참조 타입(NRT)** 기능은 참조 타입 변수에 대해 null 가능성을 컴파일러가 분석하도록 합니다. 프로젝트 파일이나 `#nullable` 지시문으로 활성화합니다.

```csharp
#nullable enable

string nonNullable = null; // 경고: null을 할당할 수 없음
string? nullable = null;   // 정상
```

이 환경에서, 개발자가 특정 변수가 절대 `null`이 아니라고 확신할 때 `!` 연산자를 사용해 컴파일러 경고를 억제합니다.

```csharp
string? maybeNull = GetSomeString();
string notNull = maybeNull!; // 단언: null이 아님
Console.WriteLine(notNull.Length);
```

주의할 점은 `!`는 컴파일러 경고만 제거할 뿐, 런타임에서 실제로 `null`이 들어오면 여전히 예외가 발생한다는 것입니다. 따라서 신중하게 사용해야 합니다.

---

## 마무리

C#의 매개변수 한정자와 Null 안전성 연산자는 단순한 문법적 편의를 넘어, 메모리 효율성, 성능 최적화, 코드 안정성에 직접적인 영향을 미치는 중요한 도구입니다.

- `out`은 여러 값을 반환할 때, `ref`는 양방향 참조 전달이 필요할 때, `in`은 읽기 전용 참조로 값 형식의 복사 비용을 줄일 때 활용합니다.
- `params`는 가변 인수를 편리하게 처리하지만, 성능이 중요한 부분에서는 배열 재사용을 고려해야 합니다.
- `??`, `?.`, `??=`를 조합하면 null 체크를 간결하게 표현할 수 있으며, Nullable 참조 타입과 `!` 연산자를 통해 컴파일 타임에 더욱 안전한 코드를 작성할 수 있습니다.
