---
layout: post
title: C# - 매개변수 한정자와 Nullable 관련 연산자
date: 2024-10-07 20:20:23 +0900
category: Csharp
---
# C# 매개변수 한정자와 Null 안전성 연산자 완벽 가이드

C# 코드를 작성하다 보면 메서드의 매개변수 선언부나 변수 할당 과정에서 낯선 키워드와 기호들을 자주 마주하게 됩니다. 매개변수가 메모리상에서 어떻게 전달되는지를 정밀하게 제어하는 `out`, `in`, `ref`, `params` 한정자와, 프로그램이 예기치 않게 종료되는 원인 순위권인 Null 참조 예외를 방어하는 `??`, `?.`, `!` 연산자는 실무 C# 프로그래밍의 핵심입니다.

이 글에서는 단순히 문법을 나열하는 것을 넘어, 메모리 관점에서의 내부 동작 원리와 실무에서 자주 사용되는 디자인 패턴까지 상세히 파헤쳐 보겠습니다.

## 매개변수 전달 한정자

기본적으로 C#에서 메서드에 인수를 전달할 때는 값에 의한 전달(Pass by Value) 방식을 사용합니다. 이는 원본 데이터의 복사본을 만들어 메서드 내부로 넘기는 방식입니다. 하지만 성능 최적화나 여러 개의 결과값을 반환해야 하는 특수한 상황에서는 복사본이 아닌 원본 메모리 주소를 직접 제어해야 할 필요가 생기며, 이때 매개변수 한정자를 사용합니다.

### out 한정자와 Try 패턴

`out` 키워드는 하나의 메서드가 여러 개의 결과값을 반환해야 할 때 가장 유용하게 사용됩니다. 일반적인 `return` 문은 단 하나의 값만 반환할 수 있기 때문입니다.

`out` 한정자의 가장 큰 특징은 호출하는 쪽에서는 변수를 초기화하지 않고 넘겨도 되지만, 메서드 내부에서는 해당 매개변수에 반드시 값을 할당(초기화)해야만 컴파일이 통과된다는 점입니다. 즉, 메서드가 이 변수를 출력 전용 파이프로 사용하겠다는 엄격한 계약을 컴파일러와 맺는 것입니다.

```csharp
string input = "123";

// 외부에서 미리 선언할 필요 없이 메서드 호출부에서 인라인 선언이 가능합니다.
if (int.TryParse(input, out int number))
{
    Console.WriteLine($"변환 성공: {number}");
}
else
{
    Console.WriteLine("변환 실패");
}

```

이러한 특성을 활용하여 C# 생태계에서는 Try 패턴이라는 설계 방식을 광범위하게 사용합니다. 메서드의 실제 반환값으로는 작업의 성공 여부를 나타내는 논리값(`bool`)을 넘기고, 실제 결과 데이터는 `out` 매개변수로 출력하는 방식입니다.

```csharp
// Dictionary에서의 Try 패턴 활용
if (dictionary.TryGetValue("key", out var value))
{
    Console.WriteLine(value);
}

// JsonDocument에서의 Try 패턴 활용
using JsonDocument doc = JsonDocument.Parse(json);
if (doc.RootElement.TryGetProperty("name", out var nameElement))
{
    Console.WriteLine(nameElement.GetString());
}

```

이 패턴은 변환이나 검색에 실패했을 때 무거운 예외(Exception)를 발생시키는 대신 부드럽게 분기 처리를 할 수 있게 해주어 애플리케이션의 성능 저하를 막아줍니다.

### ref 한정자와 참조 전달

`ref` 키워드는 인수를 메모리의 참조(Reference) 형태로 전달합니다. `out`과 비슷해 보이지만, `ref`는 양방향 통신을 목적으로 합니다. 호출하는 쪽에서 반드시 값을 초기화한 상태로 넘겨야 하며, 메서드 내부에서는 그 값을 읽을 수도 있고 새로운 값으로 덮어쓸 수도 있습니다.

이때 가장 주의해야 할 점은 값 형식(Value Type, 구조체 등)과 참조 형식(Reference Type, 클래스 등)을 `ref`로 넘길 때 발생하는 동작의 차이입니다. 아래의 텍스트 다이어그램을 통해 메모리 구조를 살펴보겠습니다.

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

값 형식(struct)을 `ref`로 넘기면 값의 복사본을 만드는 대신, 스택(Stack) 메모리 상의 원본 주소를 직접 가리키게 됩니다. 따라서 메서드 내부에서 값을 수정하면 원본 변수도 즉시 변경됩니다.

```csharp
int x = 10;
ModifyValue(ref x);
Console.WriteLine(x); // 출력 결과: 20

void ModifyValue(ref int a) => a = 20;

```

반면 참조 형식(class)을 `ref`로 넘긴다는 것은 힙(Heap) 메모리를 가리키고 있는 포인터 자체의 주소를 넘기는 이중 포인터 개념과 같습니다. 따라서 메서드 내부에서 `new` 키워드로 아예 새로운 객체를 할당해버리면, 함수 밖의 원본 참조 변수도 새로운 객체를 가리키게 됩니다.

```csharp
class Person { public string Name = ""; }

Person p = new Person { Name = "John" };
ModifyReference(ref p);
Console.WriteLine(p.Name); // 출력 결과: "Jane"

void ModifyReference(ref Person person) 
{
    // 원본 참조가 가리키는 대상 자체를 완전히 새로운 객체로 교체합니다.
    person = new Person { Name = "Jane" }; 
}

```

### in 한정자와 읽기 전용 참조의 함정

`in` 키워드는 인수를 참조로 전달하지만, 메서드 내부에서 그 값을 절대 수정할 수 없도록 강제하는 읽기 전용 참조 기능입니다. 구조체(struct)와 같이 크기가 큰 값 형식을 메서드에 전달할 때 발생하는 깊은 복사(Deep Copy) 비용을 줄여 성능을 끌어올리기 위해 사용됩니다.

```csharp
public struct LargeStruct
{
    public long A, B, C, D; // 32바이트 크기의 구조체
}

public void Print(in LargeStruct data)
{
    // data.A = 10; // 컴파일 오류 발생! 읽기 전용입니다.
    Console.WriteLine(data.A);
}

```

하지만 `in` 매개변수에는 방어적 복사(Defensive Copy)라는 치명적인 성능 함정이 숨어 있습니다. 만약 전달된 구조체가 내부 상태를 변경할 수 있는 일반 구조체라면, 컴파일러는 메서드 내부에서 해당 구조체의 속성이나 메서드를 호출할 때 원본이 훼손될 가능성이 있다고 판단합니다.

그 결과, 컴파일러는 사용자가 모르게 구조체의 임시 복사본을 만들어 그 복사본 위에서 메서드를 실행해 버립니다. 성능 최적화를 위해 도입한 `in` 키워드가 오히려 불필요한 메모리 복사를 유발하는 것입니다.

```csharp
public struct Point
{
    public int X { get; set; }
    public void SetX(int value) => X = value;
}

public void Print(in Point p)
{
    // 원본을 보호하기 위해 보이지 않는 임시 복사본이 생성됩니다.
    p.SetX(10); 
    // 결과적으로 원본 구조체의 상태는 전혀 변하지 않습니다.
}

```

이러한 방어적 복사 문제를 원천 차단하려면, `in` 키워드로 전달할 구조체는 반드시 `readonly struct`로 선언하여 불변성(Immutability)을 보장하는 것이 C# 프로그래밍의 모범 사례입니다.

### params 한정자와 가변 인수 할당 오버헤드

`params` 키워드는 하나의 메서드가 여러 개의 인수를 개수에 제한 없이 유연하게 받을 수 있도록 해줍니다. 메서드 선언부의 가장 마지막 매개변수 자리에만 올 수 있으며, 배열 형태로 선언해야 합니다.

```csharp
public static int Sum(params int[] numbers)
{
    int total = 0;
    foreach (var num in numbers)
        total += num;
    return total;
}

Console.WriteLine(Sum(1, 2, 3));       // 쉼표로 구분하여 전달 가능
Console.WriteLine(Sum(new int[] { 10, 20 })); // 만들어진 배열 자체를 전달 가능

```

코드를 작성할 때는 쉼표로 나열하여 편리하게 호출하지만, 내부적으로 컴파일러는 이를 배열 인스턴스 생성 코드로 치환합니다. 즉, `Sum(1, 2, 3)`은 `Sum(new int[] { 1, 2, 3 })`으로 변환되어 실행됩니다.

이는 심각한 성능 오버헤드를 유발할 수 있습니다. 만약 반복문 내에서 `params` 메서드를 호출하고 반복 횟수를 

$$N$$

이라고 할 때, 매 반복마다 새로운 배열 객체가 힙(Heap) 영역에 생성되므로 

$$O(N)$$

에 비례하는 메모리 할당과 가비지 컬렉션(GC) 부담이 발생합니다. 성능이 중요한 로직에서는 배열을 미리 생성해두고 전달하는 방식으로 메모리 할당을 최소화해야 합니다.

## Nullable 및 Null 안전성 연산자

C#의 창시자들을 비롯해 수많은 개발자를 괴롭혀온 Null 참조 예외를 방어하기 위해, C#은 버전을 거듭하며 강력하고 우아한 Null 처리 연산자들을 도입해 왔습니다.

### 값 형식의 Null 허용 (Nullable 구조체)

C#에서 정수(`int`)나 실수(`double`), 논리값(`bool`)과 같은 구조체 기반의 값 형식은 기본적으로 메모리에 항상 기본값을 가지므로 `null`을 할당할 수 없습니다. 하지만 데이터베이스의 컬럼 값이나 JSON 파싱 결과처럼 값이 없음을 명시적으로 표현해야 할 때 `Nullable<T>` 구조체를 사용합니다.

```csharp
// Nullable<int>의 축약형 기호로 물음표(?)를 사용합니다.
int? nullableInt = null;

if (nullableInt.HasValue)
{
    Console.WriteLine(nullableInt.Value);
}

```

### 널 병합 연산자 (??) 및 할당 연산자 (??=)

널 병합 연산자 `??`는 코드를 간결하게 만들어주는 일등 공신입니다. 좌항의 값이 `null`인지 평가하여, `null`이 아니라면 좌항의 값을 그대로 반환하고, 만약 좌항이 `null`이라면 우항의 대체값을 반환합니다.

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
// name이 null이므로 우항인 "이름 없음"이 result에 할당됩니다.
string result = name ?? "이름 없음"; 

```

`??=` 연산자는 여기서 한 걸음 더 나아가, 좌항의 변수가 `null`일 때만 우항의 새로운 인스턴스를 생성하여 할당합니다. 지연 초기화(Lazy Initialization) 패턴을 구현할 때 매우 유용합니다.

```csharp
List<int>? list = null;

// list 변수가 null이므로, 새로운 List 객체를 메모리에 할당하여 대입합니다.
list ??= new List<int>(); 
list.Add(1);

```

### 널 조건부 연산자 (?.)

객체의 내부에 있는 속성이나 메서드에 접근할 때, 해당 객체가 `null`인 상태에서 마침표(`.`)를 찍으면 즉시 치명적인 예외가 발생합니다. `?.` 연산자는 점검 단계를 하나 추가하여, 객체가 `null`이면 예외를 터뜨리지 않고 조용히 `null`을 반환하며 평가를 중단합니다.

```csharp
string? name = null;
// name이 null이므로 Length 속성에 접근하지 않고 즉시 null을 반환합니다.
Console.WriteLine(name?.Length); 

// 널 병합 연산자와 결합하여, null일 경우 안전하게 기본값 0을 부여하는 패턴입니다.
int length = name?.Length ?? 0;

```

### 널 억제 연산자 (!)와 Nullable 참조 타입

최신 C#에서는 참조 형식(class, string 등)의 변수에도 암묵적인 `null` 할당을 막고, 컴파일 타임에 엄격하게 Null 안전성을 검사하는 Nullable 참조 타입(NRT, Nullable Reference Types) 기능이 도입되었습니다.

코드 상단에 `#nullable enable`을 선언하면, 기존에 당연하게 썼던 참조 타입 변수들이 기본적으로 절대 `null`이 될 수 없음을 의미하게 되며, 명시적으로 물음표(`?`)를 붙여야만 `null`을 허용합니다.

```csharp
#nullable enable

// 경고 발생: 일반 string 타입은 이제 null을 허용하지 않는 것으로 간주합니다.
string nonNullable = null; 

// 정상: 명시적으로 ?를 붙여 null 가능성을 컴파일러에게 알립니다.
string? nullable = null;   

```

이 엄격한 분석 환경 속에서, 개발자가 시스템 흐름상 해당 변수가 절대 `null`일 리가 없다고 확신하는 경우가 있습니다. 이때 변수명 뒤에 느낌표(`!`)를 붙여 사용하는 널 억제 연산자(Null-forgiving Operator)를 사용합니다.

```csharp
#nullable enable
string? maybeNull = GetSomeString();

// 컴파일러에게 "이 변수는 분석과 달리 절대 null이 아니니 경고를 무시해라"라고 단언합니다.
string notNull = maybeNull!; 
Console.WriteLine(notNull.Length);

```

단, `!` 연산자는 컴파일러의 경고를 강제로 끄는 역할만 할 뿐, 런타임 환경에서 물리적인 방어막을 쳐주는 것은 아닙니다. 만약 단언과 다르게 실제 실행 시점에 변수에 `null`이 들어있다면 예외가 발생하므로 매우 신중하게 사용해야 합니다.

## 마무리

C#이 제공하는 매개변수 한정자와 Null 안전성 연산자들은 단순히 타이핑을 줄여주는 문법적 편의성을 넘어서는 중요한 기능들입니다. 이들은 런타임 성능 최적화, 불필요한 메모리 할당 방지, 그리고 가장 예측하기 힘든 Null 참조 예외로부터 애플리케이션의 안정성을 지켜내는 강력한 구조적 도구입니다.

`in` 키워드의 방어적 복사 원리를 이해하고 불변 구조체를 설계하며, `?.`와 `??=`를 조합해 간결하고 안전한 코드 파이프라인을 구축하는 습관을 들이신다면 한 차원 더 견고한 C# 애플리케이션을 완성하실 수 있을 것입니다.