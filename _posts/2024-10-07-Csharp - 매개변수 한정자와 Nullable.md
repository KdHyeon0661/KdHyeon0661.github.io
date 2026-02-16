---
layout: post
title: C# - 매개변수 한정자와 Nullable 관련 연산자
date: 2024-10-07 20:20:23 +0900
category: Csharp
---
# 매개변수 한정자(out, in, ref, params)와 Nullable 관련 연산자

C#을 사용하다 보면 자주 마주치지만 헷갈리기 쉬운 기능들이 있습니다. 특히 메서드의 매개변수를 다루는 `out`, `in`, `ref`, `params` 한정자와 Nullable 값을 안전하게 처리하는 `??`, `?.`, `!` 연산자는 실무에서 매우 유용하게 쓰입니다. 이번 포스트에서는 각각의 개념과 사용법, 고급 팁까지 예제 코드와 함께 살펴보겠습니다.

---

## 1. 매개변수 관련 키워드

### 1.1 out 한정자

`out` 키워드는 메서드가 여러 개의 값을 반환해야 할 때 주로 사용됩니다. `ref`와 비슷하지만 결정적인 차이가 있습니다.

**특징**
- 메서드 내에서 **반드시 초기화(할당)**해야 합니다.
- 호출하는 쪽에서는 변수를 초기화하지 않고 전달할 수 있습니다. (C# 7.0부터는 메서드 호출 시 바로 변수 선언 가능)

**예제**
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

**ref와의 차이점**
- `ref`는 메서드에 전달하기 전에 **반드시 초기화**되어 있어야 하며, 메서드 내에서 값을 읽고 쓸 수 있습니다.
- `out`은 전달하기 전에 초기화가 필요 없고, 메서드 내에서 **반드시 값을 할당**해야 합니다. 즉, 메서드가 값을 '출력'하는 용도로만 사용됩니다.

#### 🔍 Try-pattern 일반화
`out`은 단순히 `TryParse`에서만 사용되지 않습니다. **Try-pattern**은 작업 성공 여부를 `bool`로 반환하고 실제 결과는 `out` 매개변수로 전달하는 디자인 패턴입니다.

```csharp
// Dictionary.TryGetValue
if (dictionary.TryGetValue("key", out var value))
{
    Console.WriteLine(value);
}

// List.Find (out은 없지만 유사한 패턴)
var list = new List<int> { 1, 2, 3 };
var found = list.Find(x => x > 2); // 없으면 0 (기본값) 반환

// JsonElement.TryGetProperty
using JsonDocument doc = JsonDocument.Parse(json);
if (doc.RootElement.TryGetProperty("name", out var nameElement))
{
    Console.WriteLine(nameElement.GetString());
}
```

이 패턴의 장점은 예외를 발생시키지 않아 성능이 좋고, 코드 흐름이 명확해진다는 점입니다.

---

### 1.2 ref 한정자

`ref` 키워드는 인수를 **참조로 전달**할 때 사용합니다. 메서드 내에서 변수의 값을 읽고 쓸 수 있으며, 호출하는 쪽에서는 **반드시 초기화된 변수**를 전달해야 합니다.

**값 형식 vs 참조 형식에서의 동작 차이 (면접 단골 질문!)**

- **값 형식(struct)**: `ref`를 사용하면 변수의 **원본 위치**를 참조하므로 메서드 내에서 값을 변경하면 호출한 곳의 변수도 변경됩니다.
  ```csharp
  int x = 10;
  Modify(ref x);
  Console.WriteLine(x); // 20

  void Modify(ref int a) => a = 20;
  ```

- **참조 형식(class)**: 참조 형식 자체를 `ref`로 전달하면 **참조를 가리키는 포인터**가 전달됩니다. 따라서 메서드 내에서 새로운 객체를 할당하면 호출한 곳의 참조도 바뀝니다.
  ```csharp
  class Person { public string Name = ""; }
  Person p = new Person { Name = "John" };
  Modify(ref p);
  Console.WriteLine(p.Name); // "Jane"

  void Modify(ref Person person) => person = new Person { Name = "Jane" };
  ```

**ref 반환(ref return)**  
메서드가 참조를 반환하여 원본 데이터를 직접 조작할 수 있습니다.
```csharp
private int[] numbers = { 1, 2, 3 };
public ref int GetFirst() => ref numbers[0];

ref int first = ref GetFirst();
first = 10;
Console.WriteLine(numbers[0]); // 10
```

**ref 지역 변수(ref local)**  
지역 변수를 다른 변수의 별칭으로 사용할 수 있습니다.
```csharp
int x = 5;
ref int y = ref x; // y는 x의 별칭
y = 10;
Console.WriteLine(x); // 10
```

---

### 1.3 in 한정자

`in` 키워드는 읽기 전용으로 인수를 **참조로 전달**할 때 사용합니다. 주로 큰 구조체를 메서드에 전달할 때 값 복사 비용을 줄이기 위해 사용됩니다.

**특징**
- 인수는 참조로 전달되지만 메서드 내에서 **수정할 수 없습니다** (읽기 전용).
- 값 형식의 복사를 방지하여 성능을 향상시킵니다.
- `in` 매개변수는 선언 시 `readonly` 특성을 가집니다.

**예제**
```csharp
public struct LargeStruct
{
    public long A, B, C, D;
}

public void Print(in LargeStruct data)
{
    // data.A = 10; // 컴파일 오류! 읽기 전용
    Console.WriteLine(data.A);
}

// 호출
LargeStruct large = new LargeStruct();
Print(in large); // in 생략 가능 (컴파일러가 참조로 전달)
```

#### ⚠️ 방어적 복사(Defensive Copy)의 함정
`in` 매개변수로 전달된 구조체가 **읽기 전용이 아닌 경우**, 구조체의 속성이나 인스턴스 메서드를 호출하면 컴파일러가 **방어적 복사**를 수행합니다.

```csharp
public struct Point
{
    public int X { get; set; }
    public void SetX(int value) => X = value;
}

public void Print(in Point p)
{
    // p.X = 10; // 직접 할당은 컴파일 오류
    p.SetX(10); // 방어적 복사 발생! (원본은 변경되지 않음)
}
```

방어적 복사는 임시 복사본을 만들어 메서드를 호출하므로 성능 저하와 함께 의도치 않은 동작을 유발할 수 있습니다. 따라서 `in`은 **readonly struct**와 함께 사용하는 것이 바람직합니다.

```csharp
public readonly struct ReadOnlyPoint
{
    public int X { get; }
    public ReadOnlyPoint(int x) => X = x;
}
```

---

### 1.4 params 한정자

`params` 키워드는 메서드가 **가변 개수의 인수**를 받을 수 있도록 해줍니다.

**특징**
- 매개변수 배열 앞에 `params`를 붙입니다.
- 메서드 호출 시 인수를 쉼표로 구분하여 나열하거나, 배열 자체를 전달할 수 있습니다.
- `params`는 메서드 선언에서 **마지막 매개변수**여야 하며, 하나만 사용 가능합니다.

**예제**
```csharp
public static int Sum(params int[] numbers)
{
    int total = 0;
    foreach (var num in numbers)
        total += num;
    return total;
}

// 호출
Console.WriteLine(Sum(1, 2, 3));       // 출력: 6
Console.WriteLine(Sum(1, 2, 3, 4, 5)); // 출력: 15
Console.WriteLine(Sum(new int[] { 10, 20 })); // 배열도 가능
```

#### 🔧 내부 동작: 배열 생성
`params`를 사용하면 컴파일러는 **자동으로 배열을 생성**합니다. 예를 들어 `Sum(1, 2, 3)`은 컴파일 시 `Sum(new int[] { 1, 2, 3 })`으로 변환됩니다. 따라서 **메서드 호출마다 배열이 힙에 할당**되므로 성능에 민감한 코드에서는 주의해야 합니다.

이미 배열을 가지고 있다면 직접 전달하여 추가 할당을 피할 수 있습니다.
```csharp
int[] numbers = GetNumbers();
Sum(numbers); // 새로운 배열 생성 없음
```

---

## 2. Nullable 관련 연산자

C#에서는 값 형식에도 `null`을 허용하는 **Nullable<T>** 타입을 제공하며, 참조 형식에 대한 null 안전성 기능도 발전해왔습니다.

### 2.1 int? (Nullable<T>)

`int?`는 `Nullable<int>`의 축약형입니다. 값 형식(구조체)에 `null`을 할당할 수 있게 해줍니다.

```csharp
int? nullableInt = null;
if (nullableInt.HasValue)
{
    Console.WriteLine(nullableInt.Value);
}
else
{
    Console.WriteLine("null입니다.");
}
```

---

### 2.2 ?? (null 병합 연산자)

왼쪽 피연산자가 `null`이면 오른쪽 피연산자를 반환하고, `null`이 아니면 왼쪽 값을 반환합니다.

```csharp
string? name = null;
string result = name ?? "이름 없음";
Console.WriteLine(result); // 출력: 이름 없음

int? count = null;
int actualCount = count ?? 0; // null이면 0 반환
```

---

### 2.3 ??= (null 병합 할당 연산자) (C# 8.0+)

왼쪽 피연산자가 `null`이면 오른쪽 값을 할당합니다. 실무에서 자주 사용됩니다.

```csharp
List<int>? list = null;
list ??= new List<int>(); // list가 null이므로 새 인스턴스 할당
list.Add(1);
```

---

### 2.4 ?. (null 조건부 연산자)

객체의 멤버에 접근하기 전에 객체가 `null`인지 검사하여, `null`이면 `null`을 반환하고 그렇지 않으면 멤버에 접근합니다.

```csharp
string? name = null;
Console.WriteLine(name?.Length); // 출력: (아무것도 안 나옴, null 반환)

// null 병합 연산자와 함께 자주 사용
int length = name?.Length ?? 0;
Console.WriteLine(length); // 출력: 0
```

---

### 2.5 ! (null 무시 연산자)

Nullable 참조 타입 컨텍스트에서 컴파일러에게 **"이 값은 절대 null이 아니다"**라고 단언(assert)할 때 사용합니다. 주의해서 사용하지 않으면 런타임에 `NullReferenceException`이 발생할 수 있습니다.

```csharp
#nullable enable
string? maybeNull = GetSomeString();
string notNull = maybeNull!; // 내가 확실히 null이 아니라고 보장할 때 사용
Console.WriteLine(notNull.Length);
```

**주의사항**
- `!` 연산자는 컴파일러의 경고를 억제할 뿐, 실제로 null을 체크하지는 않습니다.
- 정말로 null이 아님이 확실한 경우에만 사용해야 합니다.

---

### 2.6 Nullable 참조 타입 (C# 8.0+)

C# 8.0부터 도입된 Nullable 참조 타입(NRT)은 **참조 타입 변수에 null 허용 여부를 명시적으로 표시**하여 컴파일 타임에 null 관련 버그를 줄이기 위한 기능입니다.

`#nullable enable` 지시문을 사용하면 해당 범위에서 null 검사가 활성화됩니다.

```csharp
#nullable enable
string nonNullable = null; // 경고 CS8600: Null 리터럴을 null 리터럴로 변환 중
string? nullable = null;   // OK
```

**왜 생겼을까요?**  
기존에는 참조 타입이 항상 null을 허용했기 때문에, 개발자의 의도(절대 null이 아님)를 코드로 표현할 수 없었습니다. NRT는 이러한 의도를 명시하여 API 계약을 명확히 하고, 호출하는 쪽에 경고를 제공합니다.

---

### 2.7 null 패턴 매칭 (is null)

객체가 null인지 검사할 때 `==` 대신 `is null`을 사용하면 연산자 오버로딩의 영향을 받지 않고 안전하게 비교할 수 있습니다.

```csharp
if (obj is null)
{
    Console.WriteLine("obj is null");
}
```

---

## 마무리

이상으로 C#에서 자주 사용되는 매개변수 한정자(`out`, `ref`, `in`, `params`)와 Nullable 관련 연산자(`int?`, `??`, `??=`, `?.`, `!`)에 대해 알아보았습니다. 단순한 문법을 넘어 내부 동작 방식(방어적 복사, 배열 생성)과 실제 활용 패턴(Try-pattern)까지 이해하면, 면접이나 실무에서 한 단계 더 깊이 있는 논의가 가능해집니다.

각각의 특징을 정확히 이해하고 적재적소에 사용하면 코드의 가독성과 안전성, 성능까지 챙길 수 있습니다. 특히 nullable 관련 기능들은 null 참조 예외를 방지하는 강력한 도구이니 적극 활용해보시기 바랍니다.